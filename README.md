# RiskLens

RiskLens는 보험 상품 추천 서비스가 아니라, 개인의 생애 재무 리스크를 분석하는 서비스입니다.

사용자의 소득, 지출, 자산, 부채, 가족 구성 정보를 바탕으로 사망, 중대질병, 소득중단 같은 위험이 발생했을 때 예상 재무 충격과 부족금액을 계산합니다.

핵심 관점은 단순합니다.

```text
보험을 추천하지 않는다.
위험 발생 시 감당 가능한지 계산한다.
```

## 1. What This Project Does

RiskLens는 다음 질문에 답하는 것을 목표로 합니다.

- 내 가장 큰 경제적 위험은 무엇인가?
- 위험이 발생하면 필요한 금액은 얼마인가?
- 현재 유동자산과 보장금액으로 감당 가능한가?
- 부족금액은 얼마인가?
- 회복하려면 얼마나 걸리는가?
- 어떤 공공데이터와 가정을 근거로 계산했는가?

## 2. What This Project Is Not

- 보험 상품 추천 서비스가 아닙니다.
- 보험 가입 필요 여부를 단정하지 않습니다.
- 개인의 질병, 사망, 실직 확률을 예측하지 않습니다.
- 보험사 언더라이팅 모델을 만들지 않습니다.
- 초기 MVP에서는 머신러닝과 외부 API 실시간 연동을 하지 않습니다.

## 3. Current MVP Direction

초기 MVP는 전체 리스크를 얕게 모두 구현하기보다, **중대질병 발생 시 재무 충격 분석**을 첫 번째 수직 슬라이스로 완성합니다.

```text
공공 의료비 데이터
-> 기준 데이터셋
-> Python 계산 모델
-> Spring Boot 저장/연동
-> Vue 결과 리포트
```

첫 번째 분석은 다음 값을 계산합니다.

- 치료기간 총 현금 유출
- 활용 가능 자금
- 부족금액
- 대응률
- 회복개월 수
- 설명 가능한 리스크 점수

## 4. Tech Stack

| 영역 | 기술 | 역할 |
| --- | --- | --- |
| Frontend | Vue | 입력 화면, 결과 리포트, 분석 이력 |
| Backend | Spring Boot LTS | 요청 처리, 저장, 정제, Python 서버 호출 |
| Persistence | JPA, QueryDSL | 분석 결과, 기준 데이터, 이력 조회 |
| Modeling | Python Server | 계산식, 점수화, 데이터 분석 로직 |

## 5. Documentation Flow

처음 참여한 사람은 아래 순서로 읽습니다.

| 순서 | 문서 | 읽는 목적 |
| --- | --- | --- |
| 1 | [README.md](README.md) | 프로젝트 한눈에 보기 |
| 2 | [PROJECT_DIRECTION.md](PROJECT_DIRECTION.md) | 도메인 철학과 분석 방향 이해 |
| 3 | [RISK_ANALYSIS_VERTICAL_SLICE_DESIGN.md](RISK_ANALYSIS_VERTICAL_SLICE_DESIGN.md) | 첫 MVP 범위와 학습/구현 흐름 이해 |
| 4 | [DATA_SOURCE_RISK_MODEL_MAPPING.md](DATA_SOURCE_RISK_MODEL_MAPPING.md) | 공공데이터가 어떤 지표와 리스크에 연결되는지 이해 |
| 5 | [DATASET_SPEC.md](DATASET_SPEC.md) | 실제 기준 데이터셋 필드와 품질 기준 확인 |
| 6 | [CALCULATION_SPEC.md](CALCULATION_SPEC.md) | Python이 구현할 계산식과 점수 기준 확인 |
| 7 | [SYSTEM_ARCHITECTURE.md](SYSTEM_ARCHITECTURE.md) | Vue, Spring Boot, Python 책임 분리 확인 |
| 8 | [INITIAL_SETUP.md](INITIAL_SETUP.md) | 두 명의 역할 분담과 다음 작업 확인 |

문서 역할은 다음처럼 나눕니다.

- `PROJECT_DIRECTION.md`: 왜 이 프로젝트를 만들고 어떤 원칙을 지키는가
- `RISK_ANALYSIS_VERTICAL_SLICE_DESIGN.md`: 첫 번째로 무엇을 끝까지 만들 것인가
- `DATA_SOURCE_RISK_MODEL_MAPPING.md`: 어떤 공공데이터가 어떤 분석 지표로 이어지는가
- `DATASET_SPEC.md`: 내부 기준 데이터셋의 필드, 단위, 출처, 품질 기준은 무엇인가
- `CALCULATION_SPEC.md`: 부족금액, 회복개월, 점수는 어떻게 계산하는가
- `SYSTEM_ARCHITECTURE.md`: Vue, Spring Boot, Python이 각각 무엇을 책임지는가
- `INITIAL_SETUP.md`: 두 명이 어떤 순서로 작업할 것인가

## 6. Next Milestone

다음 마일스톤은 구현량이 아니라 **계산 가능한 분석 기준 고정**입니다.

완료 기준:

- 실제 공공데이터 후보의 컬럼을 확인한다.
- `medical_cost_baseline` 샘플 5~10행을 만든다.
- `illness_scenario_assumption` 샘플을 만든다.
- `CALCULATION_SPEC.md` 기준으로 Python 순수 함수 테스트 케이스를 정의한다.
- Spring-Python API 계약 초안을 작성한다.

## 7. Success Criteria

프로젝트의 성공은 보험 가입을 유도하는 것이 아닙니다.

사용자가 결과를 보고 다음 질문에 답할 수 있다면 성공입니다.

- 어떤 위험에서 재무적으로 가장 취약한가?
- 현재 자산과 보장으로 감당 가능한가?
- 부족금액은 얼마인가?
- 어떤 데이터와 가정이 결과에 사용되었는가?
- 어떤 위험부터 준비해야 하는가?
