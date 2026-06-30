# Calculation Specification

## 1. Purpose

이 문서는 RiskLens의 계산 기준을 정의한다.

초기 범위는 **중대질병 발생 시 재무 충격 분석**이다.

계산식은 Python 모델링 서버의 단일 책임으로 둔다. Spring Boot와 Vue는 동일 계산식을 중복 구현하지 않는다.

## 2. Modeling Principles

- 개인의 질병 발생 확률을 예측하지 않는다.
- 보험 상품 가입 필요성을 판정하지 않는다.
- 결과는 시나리오 기반 재무 충격과 현금 부족을 설명한다.
- 점수는 원화 금액과 부족금액을 보조하는 설명 지표다.
- 의료비, 생활비, 소득공백, 간병비를 분리해서 보여준다.
- 소득손실과 생활비를 이중 계산하지 않는다.

## 3. Inputs

### 3.1 User Profile

| 입력 | 단위 | 설명 |
| --- | --- | --- |
| `age` | years | 사용자 나이 |
| `gender` | enum | `MALE`, `FEMALE`, `UNKNOWN` |

### 3.2 Finance

| 입력 | 단위 | 설명 |
| --- | --- | --- |
| `monthly_income` | KRW/month | 월 세후소득 |
| `monthly_essential_expense` | KRW/month | 필수생활비 |
| `monthly_total_expense` | KRW/month | 총 월지출 |
| `monthly_debt_payment` | KRW/month | 월 부채상환액 |
| `liquid_assets` | KRW | 즉시 사용 가능한 유동자산 |
| `illness_protection_amount` | KRW | 질병 발생 시 활용 가능한 보장/대응 금액 |

### 3.3 Scenario

| 입력 | 단위 | 설명 |
| --- | --- | --- |
| `disease_group` | enum | `CANCER`, `CARDIOVASCULAR`, `CEREBROVASCULAR` |
| `severity_level` | enum | `MILD`, `MODERATE`, `SEVERE` |

### 3.4 Baseline Data

| 입력 | 출처 |
| --- | --- |
| `avg_cost_per_patient` | `medical_cost_baseline` |
| `income_loss_months` | `illness_scenario_assumption` |
| `maintained_income_ratio` | `illness_scenario_assumption` |
| `non_covered_cost_multiplier` | `illness_scenario_assumption` |
| `caregiver_cost_monthly` | `illness_scenario_assumption` |
| `living_expense_multiplier` | `illness_scenario_assumption` |

## 4. Core Calculation

### 4.1 Medical Cost

```text
base_medical_cost =
  avg_cost_per_patient

adjusted_medical_cost =
  base_medical_cost * non_covered_cost_multiplier
```

해석:

- `base_medical_cost`는 공공 의료비 통계의 1인당 평균 진료비다.
- `adjusted_medical_cost`는 비급여, 기타 추가 비용 가능성을 반영한 분석용 보정값이다.
- 보정계수는 정답이 아니라 가정이며 리포트에 표시한다.

### 4.2 Treatment Period Cash Outflow

```text
essential_expense_during_treatment =
  monthly_essential_expense
  * living_expense_multiplier
  * income_loss_months

debt_payment_during_treatment =
  monthly_debt_payment
  * income_loss_months

caregiver_cost =
  caregiver_cost_monthly
  * income_loss_months

treatment_cash_outflow =
  adjusted_medical_cost
  + essential_expense_during_treatment
  + debt_payment_during_treatment
  + caregiver_cost
```

해석:

- 치료기간에도 생활비와 부채상환은 계속 발생한다고 본다.
- 생활비가 치료기간 중 증가할 수 있으므로 `living_expense_multiplier`를 둔다.
- 간병비는 의료비 통계와 분리한다.

### 4.3 Treatment Period Cash Inflow

```text
maintained_monthly_income =
  monthly_income * maintained_income_ratio

maintained_income_during_treatment =
  maintained_monthly_income * income_loss_months

available_amount =
  liquid_assets
  + illness_protection_amount
  + maintained_income_during_treatment
```

해석:

- 치료기간 중 일부 소득이 유지될 수 있다.
- 유동자산과 질병 보장자산은 치료기간에 활용 가능한 자금으로 본다.
- 소득을 비용으로 더하지 않고, 유지 가능한 현금유입으로 반영한다.

### 4.4 Gap and Coverage

```text
coverage_gap =
  max(treatment_cash_outflow - available_amount, 0)

coverage_ratio =
  available_amount / treatment_cash_outflow
```

예외:

- `treatment_cash_outflow <= 0`이면 계산 불가 상태로 처리한다.
- `coverage_ratio`는 1을 초과할 수 있지만 UI에서는 100% 이상 대응 가능으로 표현한다.

### 4.5 Recovery Months

```text
monthly_saving_capacity =
  monthly_income
  - monthly_total_expense
  - monthly_debt_payment

recovery_months =
  coverage_gap / monthly_saving_capacity
```

예외:

- `coverage_gap == 0`이면 `recovery_months = 0`
- `monthly_saving_capacity <= 0`이고 `coverage_gap > 0`이면 `recovery_status = NOT_RECOVERABLE_WITH_CURRENT_CASHFLOW`

주의:

- `monthly_total_expense`에 부채상환액이 이미 포함된 입력 구조라면 중복 차감하지 않는다.
- MVP 입력에서는 `monthly_total_expense`와 `monthly_debt_payment`의 포함 관계를 API 계약에서 명확히 해야 한다.

## 5. Output Metrics

| 출력 | 단위 | 설명 |
| --- | --- | --- |
| `treatment_cash_outflow` | KRW | 치료기간 총 현금 유출 |
| `available_amount` | KRW | 활용 가능 자금 |
| `coverage_gap` | KRW | 부족금액 |
| `coverage_ratio` | ratio | 대응률 |
| `recovery_months` | months/null | 부족금액 회복 예상 기간 |
| `recovery_status` | enum | 회복기간 계산 상태 |
| `score` | 0~100 | 최종 리스크 점수 |
| `grade` | enum | `LOW`, `MODERATE`, `HIGH`, `VERY_HIGH` |

## 6. Score Model

점수는 아래 하위 점수의 가중합으로 계산한다.

```text
critical_illness_score =
  financial_impact_score * 0.30
  + coverage_gap_score * 0.25
  + liquidity_stress_score * 0.20
  + recovery_difficulty_score * 0.15
  + public_risk_context_score * 0.10
```

## 7. Sub Score Normalization

### 7.1 Financial Impact Score

```text
financial_impact_ratio =
  treatment_cash_outflow / annual_income
```

| ratio | score |
| ---: | ---: |
| `<= 0.2` | 20 |
| `<= 0.5` | 40 |
| `<= 1.0` | 60 |
| `<= 2.0` | 80 |
| `> 2.0` | 100 |

### 7.2 Coverage Gap Score

```text
coverage_gap_ratio =
  coverage_gap / treatment_cash_outflow
```

| ratio | score |
| ---: | ---: |
| `0` | 0 |
| `<= 0.25` | 25 |
| `<= 0.5` | 50 |
| `<= 0.75` | 75 |
| `> 0.75` | 100 |

### 7.3 Liquidity Stress Score

```text
survival_months =
  liquid_assets / (monthly_essential_expense + monthly_debt_payment)
```

| survival_months | score |
| ---: | ---: |
| `>= 12` | 20 |
| `>= 6` | 40 |
| `>= 3` | 60 |
| `>= 1` | 80 |
| `< 1` | 100 |

### 7.4 Recovery Difficulty Score

| recovery_months | score |
| ---: | ---: |
| `0` | 0 |
| `<= 12` | 30 |
| `<= 24` | 50 |
| `<= 60` | 80 |
| `> 60` or not recoverable | 100 |

### 7.5 Public Risk Context Score

초기에는 공공통계 분위 기반 점수로 단순화한다.

| context percentile | score |
| ---: | ---: |
| `<= 20` | 20 |
| `<= 40` | 40 |
| `<= 60` | 60 |
| `<= 80` | 80 |
| `> 80` | 100 |

주의:

- 이 점수는 개인의 발병 확률이 아니다.
- 동일 연령대/성별 기준 공공통계 맥락을 반영하는 보조 점수다.

## 8. Grade

| final score | grade | 표시 의미 |
| ---: | --- | --- |
| `0~29` | `LOW` | 현재 입력 기준으로 재무 충격이 낮음 |
| `30~59` | `MODERATE` | 일부 보완 필요 |
| `60~79` | `HIGH` | 부족금액 또는 회복 부담이 큼 |
| `80~100` | `VERY_HIGH` | 재무 충격과 유동성 압박이 매우 큼 |

등급은 보험 필요성 판단이 아니라 시나리오 분석 결과의 요약이다.

## 9. Edge Cases

| 케이스 | 처리 |
| --- | --- |
| `monthly_income <= 0` | 소득 기반 점수 일부 계산 불가, 리포트에 표시 |
| `treatment_cash_outflow <= 0` | 분석 실패 |
| `liquid_assets < 0` | 입력 검증 실패 |
| `monthly_saving_capacity <= 0` | 회복기간 산정 불가 |
| 기준 데이터 매칭 실패 | fallback baseline 또는 `BASELINE_NOT_FOUND` |
| `coverage_gap = 0` | 회복기간 0개월, 부족금액 없음 |
| 점수 계산 일부 실패 | 해당 하위 점수 null, 최종 점수 산정 불가 또는 fallback |

## 10. Example

입력:

```text
monthly_income = 4,000,000
monthly_essential_expense = 2,200,000
monthly_total_expense = 3,000,000
monthly_debt_payment = 500,000
liquid_assets = 8,000,000
illness_protection_amount = 5,000,000
avg_cost_per_patient = 18,000,000
income_loss_months = 6
maintained_income_ratio = 0.5
non_covered_cost_multiplier = 1.3
caregiver_cost_monthly = 1,000,000
living_expense_multiplier = 1.0
```

계산:

```text
adjusted_medical_cost = 23,400,000
essential_expense_during_treatment = 13,200,000
debt_payment_during_treatment = 3,000,000
caregiver_cost = 6,000,000
treatment_cash_outflow = 45,600,000

maintained_income_during_treatment = 12,000,000
available_amount = 25,000,000

coverage_gap = 20,600,000
coverage_ratio = 0.55
```

이 결과의 해석:

```text
중대질병 시나리오에서 총 현금 유출은 45,600,000원으로 추정됩니다.
치료기간 중 유지 가능 소득, 유동자산, 질병 관련 대응금액을 반영하면
부족금액은 20,600,000원입니다.
```

## 11. Versioning

분석 결과에는 반드시 다음 버전을 포함한다.

| 버전 | 설명 |
| --- | --- |
| `model_version` | 계산식 버전 |
| `baseline_data_version` | 기준 의료비 데이터 버전 |
| `assumption_version` | 시나리오 가정 버전 |
| `score_version` | 점수 정규화 기준 버전 |

버전이 없으면 동일 입력으로 같은 결과를 재현하기 어렵다.
