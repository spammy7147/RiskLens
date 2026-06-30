# Initial Setup Guide

## 1. Purpose

이 문서는 두 명이 RiskLens 프로젝트를 함께 시작할 때 사용하는 협업 기준 문서다.

현재 단계의 목표는 코드를 많이 작성하는 것이 아니라, **공공데이터 기반 중대질병 재무 리스크 분석을 계산 가능한 수준으로 고정하는 것**이다.

## 2. Current Agreed Direction

| 항목 | 결정 |
| --- | --- |
| 프로젝트 유형 | 공공데이터 기반 개인 재무 리스크 분석기 |
| 핵심 관점 | 보험 상품 추천이 아니라 개인 재무 리스크 분석 |
| 첫 MVP | 중대질병 발생 시 재무 충격 분석 |
| Frontend | Vue |
| Backend | Spring Boot LTS, JPA, QueryDSL |
| Modeling | Python Server |
| 데이터 전략 | 초기에는 수동 수집 정적 CSV/JSON, 이후 API 연동 |
| 계산 책임 | Python이 계산식과 점수화 담당 |
| 저장 책임 | Spring Boot가 입력, 기준 데이터, 결과 저장 담당 |

## 3. Documentation Map

처음 참여한 사람은 아래 순서로 읽는다.

| 순서 | 문서 | 역할 |
| --- | --- | --- |
| 1 | `README.md` | 프로젝트 한눈에 보기 |
| 2 | `PROJECT_DIRECTION.md` | 도메인 철학과 분석 원칙 |
| 3 | `RISK_ANALYSIS_VERTICAL_SLICE_DESIGN.md` | 첫 MVP 범위와 학습/구현 흐름 |
| 4 | `DATA_SOURCE_RISK_MODEL_MAPPING.md` | 공공데이터와 리스크 모델 연결 |
| 5 | `DATASET_SPEC.md` | 기준 데이터셋 필드, 단위, 품질 기준 |
| 6 | `CALCULATION_SPEC.md` | 중대질병 계산식과 점수 기준 |
| 7 | `SYSTEM_ARCHITECTURE.md` | Vue, Spring Boot, Python 책임 분리 |
| 8 | `INITIAL_SETUP.md` | 역할 분담과 다음 작업 |

문서가 겹칠 때는 아래 기준을 따른다.

| 주제 | 기준 문서 |
| --- | --- |
| 프로젝트 철학 | `PROJECT_DIRECTION.md` |
| 첫 MVP 범위 | `RISK_ANALYSIS_VERTICAL_SLICE_DESIGN.md` |
| 공공데이터 후보와 지표 매핑 | `DATA_SOURCE_RISK_MODEL_MAPPING.md` |
| 데이터셋 필드와 단위 | `DATASET_SPEC.md` |
| 계산식과 점수화 | `CALCULATION_SPEC.md` |
| 서버 책임 분리 | `SYSTEM_ARCHITECTURE.md` |
| 팀 작업 순서 | `INITIAL_SETUP.md` |

## 4. What This Project Must Not Become

- 보험 상품 추천 서비스
- 보험 가입 필요 여부 판정기
- 개인 질병/사망/실직 확률 예측기
- 보험사 언더라이팅 모델
- 데이터 근거 없는 임의 점수 계산기

권장 표현:

```text
이 시나리오에서 현재 대응 가능 금액이 부족합니다.
중대질병 발생 시 의료비보다 소득공백 영향이 크게 나타납니다.
공공 의료비 통계와 분석 가정을 분리해 계산했습니다.
```

피해야 할 표현:

```text
이 보험에 가입해야 합니다.
당신은 암에 걸릴 가능성이 높습니다.
당신은 위험한 사람입니다.
```

## 5. Team Role Split

역할은 소유권이지 독점이 아니다. 데이터셋, 계산식, API 계약은 반드시 서로 리뷰한다.

### Person A. Data Analysis / Python Lead

담당:

- 공공 의료비 데이터 후보 조사
- 원천 데이터 컬럼 확인
- `medical_cost_baseline` 샘플 정리
- `illness_scenario_assumption` 가정값 검토
- Python 순수 계산 함수 설계
- pytest 계산 테스트 작성

주로 보는 문서:

- `DATA_SOURCE_RISK_MODEL_MAPPING.md`
- `DATASET_SPEC.md`
- `CALCULATION_SPEC.md`
- `RISK_ANALYSIS_VERTICAL_SLICE_DESIGN.md`

### Person B. Spring / Integration Lead

담당:

- Spring 프로젝트 구조 설계
- 기준 데이터 저장 방식 설계
- 분석 요청/결과 저장 구조 설계
- Spring-Python API 계약 설계
- Vue에 제공할 결과 API 설계
- 분석 이력 조회 구조 설계

주로 보는 문서:

- `SYSTEM_ARCHITECTURE.md`
- `DATASET_SPEC.md`
- `CALCULATION_SPEC.md`
- 이후 작성할 `API_CONTRACT.md`
- 이후 작성할 `BACKEND_DESIGN.md`

## 6. Immediate Work Order

### Step 1. 실제 공공데이터 후보 확인

목표:

```text
질병별 의료비 통계에서
medical_cost_baseline에 매핑 가능한 컬럼이 실제로 있는지 확인한다.
```

1차 확인 대상:

```text
건강보험심사평가원 보건의료빅데이터개방시스템
-> 의료통계정보
-> 질병/행위별 의료 통계
-> 질병 소분류(3단 상병) 통계
```

완료 기준:

- HIRA 질병 소분류(3단 상병) 통계 파일을 확인했다.
- 원천 컬럼 목록이 있다.
- 환자 수, 총진료비, 연령대, 성별, 입원/외래 여부를 확인했다.
- 금액 단위와 기준연도를 확인했다.
- 매핑 불가능한 필드를 표시했다.

### Step 2. 샘플 기준 데이터셋 작성

목표:

```text
암 관련 샘플 5~10행을 medical_cost_baseline 구조에 맞춰 정리한다.
```

완료 기준:

- `medical_cost_baseline` 샘플이 있다.
- `illness_scenario_assumption` 샘플이 있다.
- 모든 금액은 KRW 단위다.
- `source_name`, `source_year`, `baseline_data_version`, `assumption_version`이 있다.
- 샘플 값과 실제 통계값을 구분한다.

초기 샘플 기준:

| 항목 | 기준 |
| --- | --- |
| 원천 | HIRA 질병 소분류(3단 상병) 통계 |
| 기준연도 | 최신 연간 심사년도 1개 |
| 질병군 | 암 |
| 질병코드 | C00-C97 범위 중 주요 암 5~10개 |
| 연령대 | 10세 구간 우선 |
| 성별 | `ALL`이 가능하면 우선, 없으면 남/여 분리 |

### Step 3. 계산 테스트 케이스 정의

목표:

```text
CALCULATION_SPEC.md의 계산식이 테스트 가능한 입력/출력으로 바뀐다.
```

필수 케이스:

- 유동자산이 0원인 경우
- 질병 보장금액이 있는 경우
- 치료기간 중 소득 일부가 유지되는 경우
- 월 저축 가능액이 0 이하인 경우
- 기준 데이터가 매칭되지 않는 경우
- 부족금액이 0원인 경우
- 월 총지출과 부채상환액이 중복 입력될 위험이 있는 경우

### Step 4. API 계약 초안 작성

목표:

```text
Spring Boot가 Python 서버에 무엇을 보내고,
Python 서버가 어떤 결과를 돌려주는지 JSON으로 고정한다.
```

산출물:

- `API_CONTRACT.md`
- Spring -> Python request 예시
- Python -> Spring response 예시
- 오류 응답 예시
- `model_version`, `baseline_data_version`, `assumption_version`, `score_version` 포함 여부

### Step 5. 구현 시작

구현은 아래 순서로 시작한다.

```text
1. Python 순수 계산 함수
2. pytest 계산 테스트
3. FastAPI 래핑
4. Spring DTO와 Python 호출 adapter
5. 분석 요청/결과 저장
6. Vue 결과 리포트
```

## 7. Definition of Ready

구현을 시작하기 전에 다음 조건을 만족해야 한다.

- 기준 데이터셋 필드가 정의되어 있다.
- 실제 원천 데이터 컬럼과 내부 필드의 매핑 여부를 확인했다.
- 계산식에 필요한 입력값과 단위가 명확하다.
- 부족금액과 회복개월 수의 예외 처리가 정의되어 있다.
- 최소 테스트 케이스가 있다.
- 결과 문구가 보험 추천처럼 보이지 않는다.

## 8. Definition of Done

중대질병 분석 MVP가 완료되었다고 판단하려면 다음 조건을 만족해야 한다.

- 샘플 기준 데이터셋으로 분석이 가능하다.
- Python 계산 함수가 문서의 계산식과 일치한다.
- 정상 케이스와 경계 케이스 테스트가 있다.
- Python 응답에 모델/데이터/가정/점수 버전이 포함된다.
- Spring Boot가 분석 요청과 결과를 저장한다.
- Vue가 부족금액, 대응률, 회복기간, 사용 가정을 보여준다.
- 기준 데이터가 없을 때 fallback 또는 실패 상태가 명확하다.
- 리포트가 보험 상품 추천처럼 보이지 않는다.

## 9. Open Decisions

| 결정 항목 | 현재 상태 | 권장 방향 |
| --- | --- | --- |
| 중대질병 1차 범위 | 암/심혈관/뇌혈관 후보 | 첫 샘플은 암 중심 |
| 공공 의료비 원천 | 후보만 있음 | 실제 컬럼 확인 후 확정 |
| 데이터 저장소 | 미정 | 초기 샘플은 CSV, 이후 DB 저장 |
| API 방식 | 미정 | Spring -> Python HTTP API |
| 분석 처리 방식 | 미정 | MVP는 동기 호출 + 결과 저장, 이후 상태 기반 비동기 확장 |
| 점수 기준 | 초안 있음 | 테스트 케이스로 검증 후 조정 |
| Vue 리포트 구조 | 미정 | 점수보다 부족금액과 근거 먼저 표시 |

## 10. Next Action

가장 먼저 할 일은 하나다.

```text
DATASET_SPEC.md를 기준으로
HIRA 질병 소분류(3단 상병) 통계의 실제 다운로드 파일 컬럼을 확인하고
medical_cost_baseline의 NEEDS_FILE_CHECK, UNKNOWN 항목을 갱신한다.
```
