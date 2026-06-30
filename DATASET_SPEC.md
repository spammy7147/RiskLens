# Dataset Specification

## 1. Purpose

이 문서는 RiskLens의 기준 데이터셋을 정의한다.

초기 목표는 실시간 공공 API 연동이 아니라, 공공데이터를 수동으로 확인하고 정적 CSV/JSON 기준 데이터셋으로 정리해 재현 가능한 분석 흐름을 만드는 것이다.

데이터셋 스펙은 다음 질문에 답해야 한다.

- 어떤 원천 데이터를 사용하는가?
- 원천 데이터에서 어떤 컬럼을 확보할 수 있는가?
- 내부 기준 데이터셋의 필드와 어떻게 매핑되는가?
- 금액, 기간, 비율의 단위는 무엇인가?
- 결측값과 매칭 실패는 어떻게 처리하는가?
- 결과 리포트에 어떤 출처와 기준연도를 표시하는가?

## 2. Dataset Principles

- 원천 데이터는 그대로 계산에 사용하지 않는다.
- 원천 데이터는 먼저 프로파일링하고 내부 기준 데이터셋으로 변환한다.
- 공공 의료비 통계는 실제 개인 본인부담금과 다를 수 있음을 항상 표시한다.
- 비급여, 간병비, 소득공백은 공공 의료비 데이터에 직접 포함되지 않는 항목으로 보고 별도 가정 데이터셋으로 분리한다.
- 모든 기준 데이터에는 `source_name`, `source_year`, `baseline_data_version`을 남긴다.

## 3. Dataset List

| 우선순위 | 데이터셋 | 상태 | 목적 |
| --- | --- | --- | --- |
| 1 | `medical_cost_baseline` | MVP 필수 | 질병군별 의료비 기준값 |
| 2 | `illness_scenario_assumption` | MVP 필수 | 소득공백, 비급여, 간병비 가정 |
| 3 | `cancer_context_baseline` | MVP 보조 | 암 발생률/생존율 공공통계 맥락 |
| 4 | `household_finance_baseline` | 후속 | 소득/자산/부채/지출 비교 기준 |
| 5 | `mortality_baseline` | 후속 | 사망 리스크 공공통계 맥락 |
| 6 | `employment_baseline` | 후속 | 소득중단 리스크 공공통계 맥락 |

MVP에서는 1, 2를 먼저 고정하고, 3은 점수 보정용으로 작게 사용한다.

## 4. medical_cost_baseline

### 4.1 Purpose

질병군, 연령대, 성별 기준의 의료비 통계를 내부 계산 모델에서 사용할 수 있는 형태로 정리한다.

이 데이터셋은 중대질병 분석에서 `adjusted_medical_cost`의 기준값이 된다.

### 4.2 Candidate Sources

| 우선순위 | 후보 출처 | 사용 목적 | 검증 상태 |
| --- | --- | --- | --- |
| 1 | 건강보험심사평가원 보건의료빅데이터개방시스템 - 질병 소분류(3단 상병) 통계 | 질병코드 단위의 환자수, 요양급여비용총액, 성별/연령대별 기준값 후보 | 공식 페이지에서 기간, 성별/연령 구간, 환자수, 요양급여비용총액 정의 확인 |
| 2 | 건강보험심사평가원 보건의료빅데이터개방시스템 - 국민관심질병통계 | 암 등 주요 질병군을 빠르게 탐색하는 후보 | 공식 페이지에서 입원/외래, 성별/연령 구간, 파일 제공 여부 확인 |
| 3 | 국가암정보센터 - 통계로 보는 암 | 암 발생률, 암종별 발생, 연령군별 암발생률, 생존율, 사망률, 유병률 | 공식 페이지에서 통계 메뉴와 암 발생률 데이터 확인 |
| 4 | 국민건강보험공단 건강보험 통계 | 의료 이용량, 진료비 통계 보조 후보 | 실제 컬럼 확인 필요 |
| 5 | 공공데이터포털 보건의료 데이터 | 파일/API 후보 탐색 | 실제 컬럼 확인 필요 |

### 4.2.1 Source Verification Notes

확인일: 2026-06-25

| 출처 | 공식 URL | 확인한 내용 | 모델링 판단 |
| --- | --- | --- | --- |
| HIRA 질병 소분류(3단 상병) 통계 | https://opendata.hira.or.kr/op/opc/olap3thDsInfoTab1.do | 한국표준질병ㆍ사인분류 소분류 기준으로 조회 가능, 입원/외래별, 성별/연령 5세구간별, 성별/연령 10세구간별 뷰 제공, 연간/월별 자료 제공 | `medical_cost_baseline`의 1순위 원천 후보 |
| HIRA 질병 소분류(3단 상병) 통계 | https://opendata.hira.or.kr/op/opc/olap3thDsInfoTab1.do | 요양급여비용총액은 전액 본인부담과 비급여를 제외한 금액이며, 보험자 부담금과 환자 부담금을 합한 금액으로 설명됨 | 실제 개인 부담액이 아니므로 `base_medical_cost`로만 사용하고 비급여/간병비는 가정으로 분리 |
| HIRA 질병 소분류(3단 상병) 통계 | https://opendata.hira.or.kr/op/opc/olap3thDsInfoTab1.do | 환자수는 범주 내 동일인의 중복을 제거하지만 다른 범주와 합산 시 중복 가능성이 있음 | 질병군 합산 시 중복 위험을 `quality_flags`에 기록 |
| HIRA 국민관심질병통계 | https://opendata.hira.or.kr/op/opc/olapMfrnIntrsIlnsInfoTab1.do | 사회적 이슈 또는 정보 제공 요구가 많은 질병 통계, 입원/외래별, 성별/연령 5세구간별, 성별/연령 10세구간별, 파일 제공 확인 | 암 관련 샘플 탐색용 후보 |
| 국가암정보센터 암 발생률 | https://www.cancer.go.kr/lay1/S1T639C640/contents.do | 암 발생자수, 조발생률, 연령표준화발생률, 성별 자료 확인 | `cancer_context_baseline`의 발생률 맥락 후보 |
| 국가암정보센터 통계 메뉴 | https://www.cancer.go.kr/lay1/S1T639C640/contents.do | 암종별 발생 현황, 연령군별 암발생률, 생존율, 5년 상대생존율, 사망률, 유병률 메뉴 확인 | `public_risk_context_score` 보조 지표 후보 |

현재까지의 판단:

```text
medical_cost_baseline의 1차 원천은 HIRA 질병 소분류(3단 상병) 통계로 둔다.
국민관심질병통계는 암 등 대표 질병 탐색과 교차 확인에 사용한다.
국가암정보센터는 의료비가 아니라 암 발생/생존/유병 맥락 점수에 사용한다.
```

### 4.2.2 Source-to-Field Mapping Status

| 내부 필드 | HIRA 질병 소분류 기준 매핑 | 상태 | 비고 |
| --- | --- | --- | --- |
| `disease_group` | 내부 분류로 생성 | DERIVED | C00-C97이면 `CANCER`처럼 매핑 |
| `disease_code` | KCD 소분류 코드 | VERIFIED | 파일 다운로드 후 실제 컬럼명 확인 필요 |
| `disease_name` | 질병명 | VERIFIED | 파일 다운로드 후 실제 컬럼명 확인 필요 |
| `age_band` | 성별/연령 5세 또는 10세 구간별 뷰 | VERIFIED | MVP는 10세 구간 우선 |
| `gender` | 성별/연령 구간별 뷰 | VERIFIED | 원천값을 `MALE`, `FEMALE`, `ALL`로 정규화 |
| `patient_count` | 환자수 | VERIFIED | 합산 시 중복 가능성 주의 |
| `total_medical_cost` | 요양급여비용총액 | VERIFIED | 비급여 제외, 실제 본인부담금 아님 |
| `avg_cost_per_patient` | `total_medical_cost / patient_count` | DERIVED | 내부 파생 지표 |
| `inpatient_cost` | 입원/외래별 뷰에서 입원 금액 | NEEDS_FILE_CHECK | 파일 컬럼 구조 확인 필요 |
| `outpatient_cost` | 입원/외래별 뷰에서 외래 금액 | NEEDS_FILE_CHECK | 파일 컬럼 구조 확인 필요 |
| `inpatient_cost_ratio` | `inpatient_cost / total_medical_cost` | DERIVED | 입원/외래 금액 확보 시 계산 |
| `avg_hospital_days` | 미확인 | UNKNOWN | 별도 진료일수/입원일수 컬럼 확인 필요 |
| `avg_visit_days` | 미확인 | UNKNOWN | 별도 내원일수 컬럼 확인 필요 |
| `source_name` | 내부 입력 | DERIVED | `HIRA_3RD_DISEASE_STATISTICS` |
| `source_year` | 심사년도 | VERIFIED | 연간 기준 우선 |
| `baseline_data_version` | 내부 버전 | DERIVED | 예: `hira-3rd-disease-2024-v1` |
| `quality_flags` | 내부 생성 | DERIVED | 중복/미확인/단위 변환 이슈 기록 |

상태 정의:

| 상태 | 의미 |
| --- | --- |
| `VERIFIED` | 공식 페이지에서 해당 기준 또는 개념 확인 |
| `NEEDS_FILE_CHECK` | 공식 페이지에서 뷰는 확인했지만 실제 다운로드 파일 컬럼 확인 필요 |
| `DERIVED` | 원천 컬럼이 아니라 내부에서 생성 |
| `UNKNOWN` | 현재 공식 페이지 확인만으로는 확보 여부 불명확 |

### 4.3 Internal Fields

| 필드 | 타입 | 단위 | 필수 | 설명 |
| --- | --- | --- | --- | --- |
| `disease_group` | string | enum | Y | `CANCER`, `CARDIOVASCULAR`, `CEREBROVASCULAR` |
| `disease_code` | string | KCD/code | N | 원천 데이터의 질병 코드 |
| `disease_name` | string | text | Y | 질병명 |
| `age_band` | string | age range | Y | `30-39`, `40-49`, `ALL` 등 |
| `gender` | string | enum | Y | `MALE`, `FEMALE`, `ALL`, `UNKNOWN` |
| `patient_count` | number | 명 | Y | 환자 수 또는 진료 인원 |
| `total_medical_cost` | number | KRW | Y | 총진료비 또는 원천 금액 |
| `avg_cost_per_patient` | number | KRW/person | Y | 1인당 평균 진료비 |
| `inpatient_cost` | number | KRW | N | 입원 진료비 |
| `outpatient_cost` | number | KRW | N | 외래 진료비 |
| `inpatient_cost_ratio` | number | ratio | N | 입원 진료비 비중 |
| `avg_hospital_days` | number | days | N | 평균 입원일수 |
| `avg_visit_days` | number | days | N | 평균 내원일수 |
| `source_name` | string | text | Y | 데이터 출처 |
| `source_year` | number | year | Y | 기준연도 |
| `baseline_data_version` | string | version | Y | 기준 데이터 버전 |
| `quality_flags` | string | csv text | N | `MISSING_INPATIENT_COST` 등 |

### 4.4 Derived Metrics

```text
avg_cost_per_patient =
  total_medical_cost / patient_count

inpatient_cost_ratio =
  inpatient_cost / total_medical_cost
```

계산 규칙:

- `patient_count <= 0`이면 `avg_cost_per_patient`를 계산하지 않는다.
- `total_medical_cost <= 0`이면 해당 row는 모델 입력에서 제외한다.
- `inpatient_cost` 또는 `outpatient_cost`가 없으면 비율 계산을 하지 않고 `quality_flags`에 기록한다.
- 원천 금액 단위가 천원/백만원이면 내부 데이터셋에서는 반드시 원 단위 KRW로 변환한다.

### 4.5 Sample Rows

아래 값은 구조 예시이며 실제 통계값이 아니다.

| disease_group | disease_code | disease_name | age_band | gender | patient_count | total_medical_cost | avg_cost_per_patient | source_name | source_year | baseline_data_version |
| --- | --- | --- | --- | --- | ---: | ---: | ---: | --- | ---: | --- |
| CANCER | C00-C97 | Malignant neoplasms | 40-49 | ALL | 10000 | 180000000000 | 18000000 | SAMPLE | 2025 | baseline-2026-001 |
| CARDIOVASCULAR | I20-I25 | Ischemic heart diseases | 50-59 | ALL | 8000 | 96000000000 | 12000000 | SAMPLE | 2025 | baseline-2026-001 |
| CEREBROVASCULAR | I60-I69 | Cerebrovascular diseases | 60-69 | ALL | 6000 | 90000000000 | 15000000 | SAMPLE | 2025 | baseline-2026-001 |

### 4.6 First Manual Collection Target

첫 수동 수집은 아래 조건으로 제한한다.

| 항목 | 기준 |
| --- | --- |
| 원천 | HIRA 질병 소분류(3단 상병) 통계 |
| 기간 | 최신 연간 심사년도 1개 |
| 질병군 | 암 |
| 질병코드 | C00-C97 범위 중 주요 암 5~10개 |
| 연령대 | 10세 구간 우선 |
| 성별 | `ALL`이 가능하면 우선, 없으면 남/여 분리 |
| 필수 지표 | 질병코드, 질병명, 환자수, 요양급여비용총액 |
| 보조 지표 | 입원/외래 구분 금액, 입원일수, 내원일수 |

수동 정리 시 파일명:

```text
data/samples/medical_cost_baseline_sample.csv
```

초기에는 실제 파일을 만들기 전에 원천 컬럼명과 내부 필드 매핑표를 먼저 확정한다.

## 5. illness_scenario_assumption

### 5.1 Purpose

공공 의료비 통계에 직접 포함되지 않는 재무 충격 요소를 명시적으로 분리한다.

이 데이터셋은 정답이 아니라 분석 가정이다. 리포트에는 반드시 사용한 가정을 노출한다.

### 5.2 Internal Fields

| 필드 | 타입 | 단위 | 필수 | 설명 |
| --- | --- | --- | --- | --- |
| `disease_group` | string | enum | Y | 질병군 |
| `severity_level` | string | enum | Y | `MILD`, `MODERATE`, `SEVERE` |
| `income_loss_months` | number | months | Y | 소득 감소 또는 중단 기간 |
| `recovery_months` | number | months | Y | 회복 기간 가정 |
| `maintained_income_ratio` | number | ratio | Y | 치료기간 중 유지 가능 소득 비율 |
| `non_covered_cost_multiplier` | number | multiplier | Y | 비급여/추가비용 보정 계수 |
| `caregiver_cost_monthly` | number | KRW/month | Y | 월 간병/돌봄 비용 가정 |
| `living_expense_multiplier` | number | multiplier | Y | 치료기간 필수생활비 보정 계수 |
| `source_note` | string | text | Y | 가정 근거와 주의사항 |
| `assumption_version` | string | version | Y | 가정 버전 |

### 5.3 Initial Assumptions

아래 값은 MVP 시작용 가정이며 실제 근거 조사 후 조정한다.

| disease_group | severity_level | income_loss_months | recovery_months | maintained_income_ratio | non_covered_cost_multiplier | caregiver_cost_monthly | living_expense_multiplier |
| --- | --- | ---: | ---: | ---: | ---: | ---: | ---: |
| CANCER | MODERATE | 6 | 12 | 0.5 | 1.3 | 1000000 | 1.0 |
| CARDIOVASCULAR | MODERATE | 3 | 6 | 0.6 | 1.2 | 800000 | 1.0 |
| CEREBROVASCULAR | SEVERE | 9 | 18 | 0.3 | 1.4 | 1500000 | 1.1 |

### 5.4 Report Requirement

리포트에는 다음 의미의 문구를 포함한다.

```text
소득공백, 비급여, 간병비는 공공 진료비 통계에 직접 포함되지 않는 항목이므로
분석 가정값으로 분리해 계산했습니다.
```

## 6. cancer_context_baseline

### 6.1 Purpose

암 발생률, 생존율, 유병률 같은 공공통계는 개인의 암 발생 확률을 예측하는 데 사용하지 않는다.

MVP에서는 `public_risk_context_score`의 보조 맥락으로만 사용한다.

### 6.2 Candidate Fields

| 필드 | 설명 |
| --- | --- |
| `cancer_type` | 암종 |
| `age_band` | 연령군 |
| `gender` | 성별 |
| `incidence_rate` | 발생률 |
| `survival_rate_5y` | 5년 상대생존율 |
| `prevalence_count` | 유병자 수 |
| `source_name` | 출처 |
| `source_year` | 기준연도 |
| `baseline_data_version` | 데이터 버전 |

## 7. Data Quality Gates

| Gate | 통과 기준 | 실패 시 처리 |
| --- | --- | --- |
| Schema Gate | 필수 컬럼 존재 | 데이터셋 로딩 실패 |
| Unit Gate | 금액/기간/비율 단위 명시 | 데이터셋 사용 불가 |
| Key Gate | 질병군, 연령대, 성별 매칭 가능 | fallback 또는 실패 |
| Null Gate | 핵심 계산 컬럼 null 없음 | row 제외 또는 UNKNOWN 처리 |
| Division Gate | 나눗셈 분모가 0보다 큼 | 파생 지표 계산 제외 |
| Version Gate | 출처, 기준연도, 버전 존재 | 리포트 표시 불가 |

## 8. MVP Data Collection Assignment

첫 번째 실제 데이터 작업은 다음이다.

```text
질병별 의료비 통계 후보에서
암 관련 샘플 데이터를 5~10행만 골라
medical_cost_baseline 구조에 맞춰 수동 정리한다.
```

완료 기준:

- 원천 데이터 후보 URL 또는 파일명이 있다.
- 원천 컬럼 목록이 있다.
- 내부 필드와 매핑 가능한 컬럼이 표시되어 있다.
- 매핑 불가능한 필드가 표시되어 있다.
- 금액 단위, 기간 단위, 기준연도가 표시되어 있다.
- 샘플 row 5~10개가 있다.

현재 기준의 우선순위:

```text
1. HIRA 질병 소분류(3단 상병) 통계에서 암 관련 연간 파일을 다운로드한다.
2. 실제 컬럼명을 확인한다.
3. Source-to-Field Mapping Status 표의 NEEDS_FILE_CHECK와 UNKNOWN 항목을 갱신한다.
4. 암 관련 5~10행을 medical_cost_baseline_sample.csv 형태로 수동 정리한다.
```
