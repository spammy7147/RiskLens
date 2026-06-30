# Risk Analysis Vertical Slice Design

## 1. Purpose

이 문서는 RiskLens의 첫 번째 구현 단위를 정의한다.

첫 번째 수직 슬라이스는 **중대질병 발생 시 재무 충격 분석**이다.

목표는 빠르게 많은 기능을 만드는 것이 아니라, 공공데이터와 개인 재무 정보를 결합해 다음 질문에 답할 수 있는 분석 흐름을 만드는 것이다.

```text
중대질병이 발생했을 때
사용자는 치료비, 생활비, 부채상환, 간병비를 감당할 수 있는가?
부족하다면 얼마가 부족하고, 회복하려면 얼마나 걸리는가?
```

이 분석은 보험 상품 추천이 아니다. 질병 발생 시나리오에서 발생 가능한 재무 충격과 대응 가능성을 설명하는 모델이다.

## 2. Why Critical Illness First

중대질병 리스크를 첫 번째 수직 슬라이스로 선택한 이유는 다음과 같다.

- 질병별 의료비 공공통계를 활용할 수 있다.
- 치료비, 소득공백, 생활비, 부채상환, 간병비를 함께 분석할 수 있다.
- Python 데이터 분석 학습 흐름을 만들기 좋다.
- Spring, Python, Vue의 책임 분리가 명확하다.
- 포트폴리오에서 "데이터를 어떻게 재무 리스크로 변환했는가"를 보여주기 좋다.

중대질병 분석의 핵심 메시지는 다음과 같다.

```text
우리는 개인의 질병 발생 확률을 예측하지 않는다.
공공 의료비 통계와 개인 현금흐름을 결합해
중대질병 발생 시 감당해야 할 재무 충격을 추정한다.
```

## 3. Scope

### 3.1 Included

- 질병군별 기준 의료비 데이터셋 정의
- 중대질병 시나리오 가정 데이터셋 정의
- 개인 재무 입력값 정의
- 필요자금, 활용 가능 자금, 부족금액 계산
- 대응률, 회복개월 수 계산
- 설명 가능한 0~100점 리스크 점수 산출
- Spring-Python API 계약 초안
- Vue 리포트에서 보여줄 주요 항목 정의

### 3.2 Excluded

- 개인별 질병 발생 확률 예측
- 보험 상품 추천
- 보험사 API 연동
- 머신러닝 모델
- 실시간 공공데이터 API 연동
- 의료적 진단 또는 치료 조언

## 4. End-to-End Flow

```text
공공 의료비 원천 데이터
-> 데이터 프로파일링
-> medical_cost_baseline 생성
-> illness_scenario_assumption 정의
-> 사용자 재무 입력 수집
-> Python 계산 모델 실행
-> Spring 분석 결과 저장
-> Vue 리포트 표시
```

## 5. First Data Questions

구현 전에 반드시 확인해야 할 데이터 질문이다.

1. 질병별 총진료비와 환자 수를 함께 얻을 수 있는가?
2. 연령대와 성별 기준으로 의료비를 나눌 수 있는가?
3. 입원/외래 진료비 또는 입원일수 정보를 얻을 수 있는가?
4. 제공되는 금액이 총진료비인지, 급여비인지, 본인부담금인지 구분 가능한가?
5. 비급여, 간병비, 소득공백은 공공데이터에 없으므로 어떤 가정으로 분리할 것인가?
6. 데이터 기준연도와 수집일을 결과에 어떻게 남길 것인가?

데이터 컬럼은 확정된 것이 아니라 검증 대상이다. 실제 확보 가능한 컬럼은 `DATASET_SPEC.md`에서 관리한다.

## 6. Core Datasets

첫 번째 수직 슬라이스는 두 개의 기준 데이터셋으로 시작한다.

| 데이터셋 | 역할 |
| --- | --- |
| `medical_cost_baseline` | 질병군, 연령대, 성별 기준 의료비 통계 |
| `illness_scenario_assumption` | 의료비 통계에 없는 소득공백, 비급여, 간병비 가정 |

원천 데이터는 그대로 모델에 넣지 않는다. 먼저 프로파일링하고, 서비스 계산에 맞는 기준 데이터셋으로 변환한다.

## 7. Calculation Model Summary

중대질병 분석은 현금흐름 관점으로 계산한다.

```text
treatment_cash_outflow =
  adjusted_medical_cost
  + essential_expense_during_treatment
  + debt_payment_during_treatment
  + caregiver_cost

treatment_cash_inflow =
  maintained_income_during_treatment
  + liquid_assets
  + illness_protection_amount

coverage_gap =
  max(treatment_cash_outflow - treatment_cash_inflow, 0)
```

주의할 점:

- 월소득 전체를 비용처럼 더하지 않는다.
- 생활비와 소득손실을 이중 계산하지 않는다.
- 의료비 통계와 비급여/간병비/소득공백 가정은 분리한다.
- 부족금액 계산은 "보험 필요금액"이 아니라 "시나리오상 현금 부족액"이다.

상세 계산식과 점수화 기준은 `CALCULATION_SPEC.md`에서 관리한다.

## 8. Score Model Summary

점수는 결론이 아니라 설명을 돕는 보조 지표다.

최종 점수는 아래 하위 점수로 구성한다.

| 하위 점수 | 의미 |
| --- | --- |
| Financial Impact Score | 연소득 대비 총 재무 충격 |
| Coverage Gap Score | 필요자금 중 부족한 비율 |
| Liquidity Stress Score | 유동자산으로 버틸 수 있는 기간 |
| Recovery Difficulty Score | 부족금액 회복 난이도 |
| Public Risk Context Score | 공공통계 기반 질병 위험 맥락 |

리포트에서는 점수보다 원화 금액, 부족금액, 대응률, 회복기간을 먼저 보여준다.

## 9. Team Work Split

### Person A. Data Analysis / Python Lead

- 공공 의료비 데이터 후보 조사
- 원천 컬럼 확인
- `DATASET_SPEC.md` 작성
- `medical_cost_baseline` 샘플 정리
- Python 순수 계산 함수 설계
- pytest 계산 테스트 작성

### Person B. Spring / Integration Lead

- Spring 도메인 구조 설계
- 기준 데이터 저장 방식 설계
- Spring-Python API 계약 설계
- 분석 요청/결과 저장 구조 설계
- Vue 조회 API 설계
- 리포트 응답 구조 설계

역할은 소유권이지 독점이 아니다. 데이터셋, 계산식, API DTO는 반드시 교차 리뷰한다.

## 10. Python Learning Path

Python 서버를 바로 만들지 않는다. 분석 학습 순서는 다음과 같다.

```text
pandas로 CSV 읽기
-> 컬럼/결측/단위 확인
-> groupby와 파생 지표 생성
-> 기준 데이터셋 저장
-> 순수 함수로 계산식 구현
-> pytest로 계산 검증
-> FastAPI endpoint로 감싸기
```

노트북은 탐색용이고, 서비스 계산 로직은 Python 모듈의 순수 함수로 옮긴다.

## 11. API Contract Direction

Spring Boot는 사용자 입력과 기준 데이터 버전을 관리하고, Python 서버에 분석 요청을 보낸다.

요청에는 최소한 다음 정보가 포함되어야 한다.

- `analysisId`
- `riskType`
- 사용자 나이/성별
- 월소득, 필수지출, 총지출, 부채상환액
- 유동자산, 질병 관련 보장금액
- 질병군, 중증도
- `baselineDataVersion`
- `assumptionVersion`

응답에는 최소한 다음 정보가 포함되어야 한다.

- 필요자금
- 활용 가능 자금
- 부족금액
- 대응률
- 회복개월 수
- 하위 점수
- 최종 점수와 등급
- 사용한 데이터 버전
- 사용한 가정
- 모델 버전

상세 JSON 구조는 이후 `API_CONTRACT.md`에서 확정한다.

## 12. Success Criteria

첫 번째 수직 슬라이스는 아래 기준을 만족하면 완료로 본다.

- `medical_cost_baseline` 샘플 데이터셋이 있다.
- `illness_scenario_assumption` 샘플 데이터셋이 있다.
- 데이터셋 필드, 단위, 출처, 기준연도가 문서화되어 있다.
- Python 순수 함수가 필요자금, 부족금액, 회복기간, 점수를 계산한다.
- pytest가 정상 케이스와 경계 케이스를 검증한다.
- Python 응답에 `modelVersion`, `baselineDataVersion`, `assumptionVersion`이 포함된다.
- Spring Boot가 분석 요청과 결과를 저장할 수 있다.
- Vue 리포트가 부족금액, 대응률, 회복기간, 사용 가정을 보여준다.
- 결과 문구가 보험 추천처럼 보이지 않는다.

## 13. Next Documents

다음 문서를 순서대로 작성한다.

1. `DATASET_SPEC.md`
2. `CALCULATION_SPEC.md`
3. `API_CONTRACT.md`
4. `TEST_CASES.md`
5. `MODELING_SERVER_DESIGN.md`
6. `BACKEND_DESIGN.md`
7. `FRONTEND_REPORT_DESIGN.md`

현재 단계의 다음 작업은 `DATASET_SPEC.md`와 `CALCULATION_SPEC.md` 초안을 고정하는 것이다.
