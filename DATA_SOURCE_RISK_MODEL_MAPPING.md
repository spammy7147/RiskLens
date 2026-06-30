# Data Source - Metric - Risk Model Mapping

## 1. Purpose

이 문서는 RiskLens에서 사용할 공공데이터가 어떤 분석 지표로 변환되고, 각 리스크 모델에 어떻게 반영되는지 정리한다.

세부 데이터셋 필드와 단위는 `DATASET_SPEC.md`에서 관리한다. 계산식과 점수화 기준은 `CALCULATION_SPEC.md`에서 관리한다.

이 문서의 역할은 다음 질문에 답하는 것이다.

- 어떤 공공데이터 후보가 있는가?
- 해당 데이터에서 어떤 분석 지표를 만들 수 있는가?
- 그 지표는 어떤 리스크 분석에 쓰이는가?
- 최종 리포트에서는 어떻게 설명해야 하는가?
- 사용하면 안 되는 해석은 무엇인가?

## 2. Analysis Principles

공공데이터는 개인의 미래를 단정적으로 예측하는 용도로 사용하지 않는다.

피해야 할 표현:

```text
당신은 암에 걸릴 확률이 높습니다.
당신은 사망할 가능성이 높습니다.
당신은 실직할 가능성이 높습니다.
```

사용할 표현:

```text
공공 통계상 동일 연령대/성별/가구유형의 위험 맥락을 참고했을 때,
이 사용자는 위험 발생 시 재무 충격, 유동성 부족, 회복 기간 측면에서 취약합니다.
```

핵심 원칙:

- 발생 가능성보다 발생 시 재무 충격과 대응 가능성을 중심에 둔다.
- 공공 발생률은 최종 점수의 중심이 아니라 보정값으로 사용한다.
- 평균값만 보지 않고 가능하면 중위값, 분위값, 연령대별 기준값을 우선한다.
- 보험 상품명이 아니라 리스크 대응 가능 금액으로 해석한다.

## 3. Overall Mapping Flow

```text
Public Data
  -> Public Baseline Metrics
  -> Personal Finance Metrics
  -> Scenario Simulation
  -> Risk Sub Scores
  -> Final Risk Score
  -> Explainable Report
```

| Layer | 역할 | 예시 |
| --- | --- | --- |
| Public Data | 공공 통계 원천 데이터 | 사망률, 암 발생률, 실업률, 진료비 |
| Public Baseline Metrics | 비교/계산 기준값 생성 | 연령별 암 발생률, 질병별 평균 진료비 |
| Personal Finance Metrics | 개인 재무 구조 계산 | 유동자산, 필수지출, 월 저축가능액 |
| Scenario Simulation | 위험 발생 시 재무 충격 계산 | 중대질병 치료기간 현금흐름 |
| Risk Sub Scores | 하위 점수 산출 | 재무충격, 부족률, 유동성, 회복난이도 |
| Final Risk Score | 최종 점수와 등급 | 0~100점, 낮음/보통/높음/매우 높음 |
| Explainable Report | 결과 해석 | 왜 이 리스크가 취약한지 설명 |

## 4. Source Mapping Summary

| 출처 | 주요 데이터 | 생성 지표 | 사용 리스크 | 모델 반영 |
| --- | --- | --- | --- | --- |
| KOSIS 국가통계포털 | 사망률, 사망원인, 기대수명 | Mortality Context | 사망 | Public Risk Context Score |
| KOSIS 국가통계포털 | 가계금융복지조사, 가계동향조사 | Income/Asset/Debt Baseline | 전체 | Financial Impact, Liquidity Stress |
| 국가암정보센터 | 암 발생률, 생존율, 유병률 | Cancer Context | 중대질병 | Public Risk Context, Recovery Context |
| 건강보험심사평가원 | 질병 소분류(3단 상병) 통계, 국민관심질병통계 | Medical Cost Baseline | 중대질병 | Financial Impact, Coverage Gap |
| 국민건강보험공단 | 건강보험 통계, 의료이용 통계 | Medical Utilization Baseline | 중대질병 | Medical Cost, Treatment Duration |
| 고용노동부/KOSIS | 고용률, 실업률, 고용형태 | Employment Context | 소득중단 | Public Risk Context, Scenario Period |
| 공공데이터포털 | 보건의료/고용 보조 데이터 | Supplementary Baseline | 전체 | 보조 지표 후보 |

## 5. Risk-Specific Mapping

### 5.1 Death Risk

핵심 질문:

```text
본인의 소득이 사라졌을 때 남은 가족이 필요한 돈은 얼마인가?
```

| 구분 | 내용 |
| --- | --- |
| 개인 입력값 | 나이, 성별, 배우자 여부, 자녀 수, 부양가족 수, 월 소득, 월 필수지출, 부채, 사망 보장금액 |
| 공공데이터 | KOSIS 연령/성별 사망률, 사망원인, 기대수명, 가계지출 기준 |
| 생성 지표 | Mortality Context, Dependency Impact, Household Expense Baseline |
| 주요 계산값 | 가족 생활비 필요액, 자녀 교육비, 부채 정리 금액, 장례/정리 비용 |
| 점수 반영 | 재무충격, 준비부족, 가족영향, 회복난이도, 공공위험맥락 |
| 리포트 표현 | 사망 시나리오에서 부양가족 생활비와 잔여 부채를 감당하기 위한 준비금이 부족한지 분석 |

주의:

- 사망률은 개인 사망 가능성 예측이 아니다.
- 부양가족이 없는 사용자는 가족 생계보다 부채 정리, 장례비, 재무 정리 비용 중심으로 분석한다.

### 5.2 Critical Illness Risk

핵심 질문:

```text
질병으로 인해 치료비와 소득 공백이 발생하면 얼마가 부족한가?
```

| 구분 | 내용 |
| --- | --- |
| 개인 입력값 | 나이, 성별, 월 소득, 월 필수지출, 월 부채상환액, 유동자산, 질병 보장금액 |
| 공공데이터 | 국가암정보센터 암 통계, 심평원/건보 의료비 통계 |
| 생성 지표 | Cancer Context, Medical Cost Baseline, Treatment Duration Proxy |
| 주요 계산값 | 치료기간 현금유출, 활용 가능 자금, 부족금액, 회복개월 수 |
| 점수 반영 | 재무충격, 준비부족, 유동성압박, 회복난이도, 공공위험맥락 |
| 리포트 표현 | 중대질병 시나리오에서 치료비, 생활비, 부채상환, 간병비를 감당할 수 있는지 분석 |

주의:

- 암 발생률만으로 위험을 판단하지 않는다.
- 건강보험 진료비 통계와 실제 본인부담액은 다를 수 있다.
- 비급여, 간병비, 소득공백은 별도 가정으로 분리한다.

### 5.3 Income Interruption Risk

핵심 질문:

```text
소득이 끊겼을 때 몇 개월 버틸 수 있고, 목표 기간 대비 얼마가 부족한가?
```

| 구분 | 내용 |
| --- | --- |
| 개인 입력값 | 나이, 성별, 직업, 산업군, 고용형태, 월 소득, 배우자 소득, 월 필수지출, 월 부채상환액, 유동자산 |
| 공공데이터 | KOSIS 실업률/고용률, 고용노동부 고용통계, 실업급여 관련 통계 |
| 생성 지표 | Employment Context, Reemployment Period Assumption, Income Interruption Baseline |
| 주요 계산값 | 생존 가능 개월 수, 목표 생존 개월 수, 부족 개월 수, 부족금액 |
| 점수 반영 | 재무충격, 준비부족, 유동성압박, 회복난이도, 공공위험맥락 |
| 리포트 표현 | 소득중단 시 현재 비상금과 유동자산으로 버틸 수 있는 기간을 분석 |

주의:

- 실업률은 개인 실직 확률이 아니다.
- 고용 통계는 소득중단 시나리오 기간과 점수 보정에만 사용한다.

## 6. Metric Catalog

| 지표 | 의미 | 사용 리스크 | 상세 계산 위치 |
| --- | --- | --- | --- |
| Financial Impact Ratio | 손실이 연소득 대비 얼마나 큰지 | 전체 | `CALCULATION_SPEC.md` |
| Coverage Gap | 부족한 준비금 | 전체 | `CALCULATION_SPEC.md` |
| Coverage Ratio | 현재 대응 가능 비율 | 전체 | `CALCULATION_SPEC.md` |
| Liquidity Survival Months | 유동자산으로 버틸 수 있는 기간 | 중대질병, 소득중단 | `CALCULATION_SPEC.md` |
| Recovery Months | 부족금액 회복에 필요한 기간 | 전체 | `CALCULATION_SPEC.md` |
| Medical Cost Baseline | 질병별 의료비 기준값 | 중대질병 | `DATASET_SPEC.md` |
| Cancer Context | 암 발생률/생존율 공공통계 맥락 | 중대질병 | `DATASET_SPEC.md` |
| Mortality Context | 사망률 공공통계 맥락 | 사망 | 후속 dataset spec |
| Employment Context | 고용/실업 공공통계 맥락 | 소득중단 | 후속 dataset spec |

## 7. Critical Illness Data Focus

MVP에서 가장 중요한 데이터 분석 포인트는 질병별 의료비다.

```text
질병 발생률
+ 질병별 평균 진료비
+ 치료/입원 기간
+ 소득공백 기간
+ 비급여/간병비 보정
= 중대질병 재무 충격
```

초기 질병군 후보:

| 질병군 | 선택 이유 | 주요 데이터 |
| --- | --- | --- |
| 암 | 발생률, 생존율, 유병률 등 공공 통계가 비교적 잘 정리되어 있고 중대질병 대표성이 높음 | 암 발생률, 암종별 발생, 5년 생존율, 암 진료비 |
| 심혈관질환 | 갑작스러운 치료비와 소득공백, 수술/입원 가능성이 큼 | 질병별 진료비, 입원일수, 수술/검사 통계 |
| 뇌혈관질환 | 치료 이후 장기 회복, 간병, 소득상실 가능성이 큼 | 질병별 진료비, 입원일수, 재활/장기치료 통계 |

세부 데이터셋 구조는 `DATASET_SPEC.md`의 `medical_cost_baseline`, `illness_scenario_assumption`, `cancer_context_baseline`에서 관리한다.

현재 1차 원천 판단:

| 우선순위 | 원천 | 판단 |
| --- | --- | --- |
| 1 | HIRA 질병 소분류(3단 상병) 통계 | 질병코드, 성별/연령 구간, 환자수, 요양급여비용총액 기반의 `medical_cost_baseline` 후보 |
| 2 | HIRA 국민관심질병통계 | 암 등 대표 질병군 탐색과 교차 확인 후보 |
| 3 | 국가암정보센터 암 통계 | 의료비가 아니라 발생률, 생존율, 유병률 기반의 위험 맥락 보조 후보 |

주의:

```text
HIRA의 요양급여비용총액은 비급여를 제외한 건강보험 진료비 기준이므로
실제 개인 부담액으로 직접 해석하지 않는다.
```

## 8. Report Language Mapping

| 분석 결과 | 사용자에게 보여줄 설명 |
| --- | --- |
| 부족금액 높음 | 해당 시나리오에서 현재 확인된 대응 가능 금액보다 필요한 금액이 큽니다. |
| 대응률 낮음 | 예상 필요금액 중 준비된 금액의 비율이 낮습니다. |
| 생존 가능 개월 수 낮음 | 소득이 중단될 경우 현재 유동자산으로 버틸 수 있는 기간이 짧습니다. |
| 회복기간 김 | 부족금액을 현재 저축 가능액으로 회복하는 데 오랜 시간이 필요합니다. |
| 부채상환부담 높음 | 월 소득 중 부채상환에 사용되는 비율이 높아 현금흐름 압박이 큽니다. |
| 공공위험맥락 높음 | 동일 연령대/성별/고용형태의 공공 통계를 참고해 위험 맥락을 보정했습니다. |

피해야 할 표현:

```text
보험 가입이 필요합니다.
암보험이 부족합니다.
정기보험을 추천합니다.
당신은 위험한 사람입니다.
```

사용할 표현:

```text
이 시나리오에서는 현재 대응 가능 금액이 부족합니다.
소득중단 시나리오에서 유동성 압박이 크게 나타납니다.
중대질병 시나리오에서 치료비보다 소득공백의 영향이 더 크게 나타납니다.
사망 시나리오에서 부양가족의 생활비 의존도가 주요 취약 요인입니다.
```

## 9. Data Collection Priority

초기에는 모든 데이터를 API로 자동 연동하지 않는다.

| Phase | 목표 | 산출물 |
| --- | --- | --- |
| 1 | 수동 수집 기반 정적 데이터셋 | 샘플 CSV/JSON, 컬럼 매핑표 |
| 2 | Python 프로파일링 | 결측/단위/파생 지표 확인 |
| 3 | 내부 기준 데이터셋 생성 | `medical_cost_baseline` |
| 4 | API 연동 후보 검토 | KOSIS/OpenAPI/공공데이터포털 후보 |
| 5 | 데이터 버전 관리 | source, year, collected_at, version |

가장 먼저 할 일:

```text
질병별 의료비 통계 후보에서 암 관련 샘플 5~10행을 골라
medical_cost_baseline 구조에 맞춰 매핑 가능 여부를 확인한다.
```
