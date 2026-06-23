# Initial Setup Guide

## 1. Purpose

이 문서는 두 명이 Life Risk Simulator 프로젝트를 함께 시작할 때, 초기 작업 범위와 역할이 모호해지지 않도록 정리한 협업 기준 문서이다.

현재 프로젝트는 아직 구현 단계가 아니라, 도메인/데이터/계산 모델을 먼저 확정하는 단계이다.

초기 목표:

```text
공공통계 기반 정밀 재무 리스크 분석기의
도메인 모델, 데이터 기준값, 계산식, 테스트 기준을 먼저 고정한다.
```

## 2. Document Map

프로젝트 문서는 다음 순서로 읽는다.

| 순서 | 문서 | 역할 |
| --- | --- | --- |
| 1 | README.md | 프로젝트의 문제 정의, 철학, MVP 범위 |
| 2 | PROJECT_DIRECTION.md | 프로젝트 방향성, 분석 철학, 리스크 모델 개요 |
| 3 | DATA_SOURCE_RISK_MODEL_MAPPING.md | 공공데이터, 분석 지표, 리스크 모델 연결 |
| 4 | SYSTEM_ARCHITECTURE.md | Spring 서버와 Python 모델링 서버의 책임 분리 |
| 5 | INITIAL_SETUP.md | 두 명이 실제로 어떻게 시작할지 정리 |

각 문서의 기준 역할:

- README.md는 외부에 보여줄 프로젝트 소개 문서이다.
- PROJECT_DIRECTION.md는 프로젝트의 기획/도메인 방향 문서이다.
- DATA_SOURCE_RISK_MODEL_MAPPING.md는 데이터 분석 설계 문서이다.
- SYSTEM_ARCHITECTURE.md는 기술 구조와 서버 간 책임 분리 문서이다.
- INITIAL_SETUP.md는 팀 작업 기준 문서이다.

주의:

```text
README.md는 너무 복잡하게 만들지 않는다.
상세한 설계 내용은 별도 문서에 둔다.
```

## 3. Current Agreed Direction

현재까지 합의된 핵심 방향은 다음과 같다.

```text
프로젝트 유형:
공공통계 기반 정밀 재무 리스크 분석기

핵심 관점:
보험 상품 추천이 아니라 개인 재무 리스크 분석

기술 구조:
Spring + View 웹 애플리케이션
+ Python Modeling Server

분석 방식:
시나리오 기반 재무 충격 계산
+ 공공통계 기반 위험 맥락 보정

초기 리스크:
사망 리스크
중대질병 리스크
소득중단 리스크

공공데이터 역할:
기준선, 시나리오 근거, 점수 보정
```

## 4. What This Project Is Not

초기 설계와 구현에서 다음 방향으로 흐르지 않도록 주의한다.

- 보험 상품 추천 서비스가 아니다.
- 보험 가입 필요 여부를 단정하지 않는다.
- 개인의 질병/사망/실직 확률을 예측한다고 표현하지 않는다.
- 보험사 언더라이팅 모델을 만들지 않는다.
- 머신러닝을 초기 MVP에 넣지 않는다.
- 외부 공공데이터 API 실시간 연동을 1차 구현 목표로 두지 않는다.

권장 표현:

```text
이 시나리오에서 현재 대응 가능 금액이 부족합니다.
소득중단 시 유동성 압박이 큽니다.
중대질병 발생 시 의료비보다 소득공백 영향이 크게 나타납니다.
```

피해야 할 표현:

```text
이 보험에 가입해야 합니다.
당신은 암에 걸릴 가능성이 높습니다.
당신은 위험한 사람입니다.
```

## 5. Initial Work Order

초기 작업은 다음 순서로 진행한다.

### Step 1. 기준 데이터셋 정의

먼저 실제 API 연동이 아니라, 정적 CSV/JSON 기준 데이터셋을 설계한다.

초기 데이터셋 후보:

- mortality_baseline
- cancer_baseline
- household_finance_baseline
- employment_baseline
- medical_cost_baseline
- illness_scenario_assumption

이 단계의 산출물:

- 데이터셋별 필드 정의
- 데이터 출처
- 기준 연도
- 수집 방식
- 결측값 처리 방식

### Step 2. 입력 모델 정의

사용자 입력을 Level 1, Level 2, Level 3로 나눈다.

초기 구현은 Level 1과 일부 Level 2만 사용한다.

이 단계의 산출물:

- UserProfile 입력 스키마
- Household 입력 스키마
- IncomeProfile 입력 스키마
- ExpenseProfile 입력 스키마
- AssetProfile 입력 스키마
- DebtProfile 입력 스키마
- ProtectionProfile 입력 스키마

### Step 3. 계산식 정의

코드 작성 전에 계산식을 먼저 문서로 확정한다.

초기 계산 대상:

- 사망 리스크 필요금액
- 중대질병 리스크 필요금액
- 소득중단 리스크 필요금액
- 부족금액
- 대응률
- 생존 가능 개월 수
- 회복 소요 개월 수
- 하위 점수
- 최종 리스크 점수

### Step 4. 테스트 케이스 정의

계산 로직을 구현하기 전에 테스트 케이스를 먼저 만든다.

필수 테스트 케이스:

- 유동자산이 0원인 경우
- 월 저축 가능액이 0 이하인 경우
- 총자산은 크지만 유동자산이 적은 경우
- 부양가족이 없는 경우
- 배우자 소득이 높은 경우
- 부채상환액이 과도한 경우
- 질병 보장금액은 있지만 소득공백 대응금이 부족한 경우
- 공공통계 기준값이 매칭되지 않는 경우

### Step 5. 백엔드 구조 설계

계산식과 테스트 케이스가 정리된 뒤 백엔드 구조를 설계한다.

초기 백엔드 목표:

- 입력값 검증
- 기준 데이터 조회
- 리스크별 계산
- 점수 산출
- 결과 리포트 생성
- 분석 이력 저장

단, 최종 리스크 계산과 모델링은 Python 서버가 담당하고 Spring은 요청 처리, 데이터 저장, Python 서버 호출, 결과 표시를 담당한다.

## 6. Suggested Role Split

두 명이 동시에 작업할 때는 다음처럼 나누는 것이 좋다.

### Person A. Domain & Calculation Owner

담당:

- 리스크별 계산식 정의
- 입력값 의미 정리
- 점수 산식 설계
- 테스트 케이스 설계
- 결과 리포트 문구 검토
- Python 모델링 서버의 계산 책임 범위 검토

주요 문서:

- PROJECT_DIRECTION.md
- DATA_SOURCE_RISK_MODEL_MAPPING.md
- 이후 작성할 CALCULATION_SPEC.md
- 이후 작성할 MODELING_SERVER_DESIGN.md

### Person B. Data & Backend Owner

담당:

- 공공데이터 출처 확인
- 기준 데이터셋 구조 설계
- 정적 CSV/JSON 샘플 생성
- 백엔드 모듈 구조 설계
- 데이터 조회/계산 파이프라인 설계
- Spring-Python API 계약 설계

주요 문서:

- DATA_SOURCE_RISK_MODEL_MAPPING.md
- SYSTEM_ARCHITECTURE.md
- 이후 작성할 DATASET_SPEC.md
- 이후 작성할 BACKEND_DESIGN.md
- 이후 작성할 API_CONTRACT.md

주의:

```text
역할은 소유권이지 독점이 아니다.
계산식과 데이터셋은 서로 강하게 연결되므로 PR/리뷰에서 반드시 교차 확인한다.
```

## 7. First Milestone

첫 번째 마일스톤은 코드를 많이 작성하는 것이 아니라, 계산 가능한 분석 모델을 고정하는 것이다.

Milestone 1 완료 기준:

- 핵심 리스크 3개가 확정되어 있다.
- 각 리스크의 입력값이 정의되어 있다.
- 각 리스크의 계산식 초안이 있다.
- 각 리스크의 필요한 공공 기준 데이터가 정의되어 있다.
- 질병별 의료비 기준 데이터셋 구조가 있다.
- 점수 모델의 하위 점수와 가중치가 정의되어 있다.
- 최소 테스트 케이스가 문서화되어 있다.

Milestone 1에서는 다음을 하지 않는다.

- 프론트엔드 화면 구현
- 사용자 인증 구현
- 보험 상품 추천 기능
- 외부 API 자동 연동
- 머신러닝 모델링
- 결제/보험사 연동

## 8. Required Next Documents

다음 문서를 순서대로 작성한다.

| 우선순위 | 문서 | 목적 |
| --- | --- | --- |
| 1 | DATASET_SPEC.md | 정적 기준 데이터셋의 필드, 출처, 예시값 정의 |
| 2 | CALCULATION_SPEC.md | 리스크별 계산식과 점수 산식 정의 |
| 3 | API_CONTRACT.md | Spring-Python 요청/응답 구조 정의 |
| 4 | TEST_CASES.md | 계산 로직 검증 케이스 정의 |
| 5 | BACKEND_DESIGN.md | Spring 도메인 모델과 모듈 구조 정의 |
| 6 | MODELING_SERVER_DESIGN.md | Python 모델링 서버 구조 정의 |

추천 순서:

```text
DATASET_SPEC.md
-> CALCULATION_SPEC.md
-> API_CONTRACT.md
-> TEST_CASES.md
-> BACKEND_DESIGN.md
-> MODELING_SERVER_DESIGN.md
-> 구현
```

## 9. Open Decisions

아직 확정되지 않은 결정 사항이다.

| 결정 항목 | 현재 상태 | 권장 방향 |
| --- | --- | --- |
| 기술 스택 | 확정 | Spring + View, Python Modeling Server |
| 데이터 저장소 | 미정 | 초기에는 정적 JSON/CSV, 이후 DB |
| API 연동 | 미정 | 초기에는 수동 수집 데이터, 이후 API |
| 중대질병 범위 | 암/심혈관/뇌혈관 후보 | 1차는 암 중심, 이후 확장 |
| 점수 정규화 방식 | 미정 | 구간 기반 0~100점으로 시작 |
| 리포트 등급 기준 | 미정 | 낮음/보통/높음/매우 높음 |
| 기준 통계 연도 | 미정 | 데이터 수집 시 명시 |
| 개인 입력 단계 | Level 1~3 방향만 있음 | 1차 구현 범위 확정 필요 |

확정된 기술 방향:

```text
웹 화면과 백엔드:
Spring + View

모델링:
Python Modeling Server

서버 간 통신:
HTTP API

데이터 저장/정제:
Spring Backend

리스크 계산/점수화:
Python Modeling Server
```

## 10. Definition of Ready

구현을 시작하기 전에 다음 조건을 만족해야 한다.

- 계산식에 필요한 입력값이 정의되어 있다.
- 입력값의 단위가 명확하다.
- 기준 데이터셋의 필드가 정의되어 있다.
- 공공데이터 출처와 기준 연도가 기록되어 있다.
- 결측값 처리 방식이 정해져 있다.
- 계산 결과의 단위가 명확하다.
- 최소 테스트 케이스가 있다.
- 사용자에게 보여줄 표현 원칙이 정리되어 있다.

## 11. Definition of Done

초기 계산 기능 하나가 완료되었다고 판단하려면 다음 조건을 만족해야 한다.

- 계산식이 문서와 일치한다.
- 정상 케이스 테스트가 있다.
- 경계값 테스트가 있다.
- 0 또는 음수 입력에 대한 처리가 있다.
- 기준 데이터가 없을 때의 fallback이 있다.
- 결과에 사용된 주요 가정이 리포트에 포함된다.
- 결과 문구가 보험 추천처럼 보이지 않는다.

## 12. Immediate Next Action

가장 먼저 할 일은 다음이다.

```text
DATASET_SPEC.md 작성
```

이 문서에서는 다음을 확정한다.

- 각 기준 데이터셋의 파일명
- 필드명
- 단위
- 예시값
- 출처
- 기준 연도
- 필수 여부
- 결측값 처리

그 다음 `CALCULATION_SPEC.md`에서 실제 계산식을 확정한다.
