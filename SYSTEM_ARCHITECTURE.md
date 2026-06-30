# System Architecture

## 1. Architecture Decision

RiskLens는 다음 구조로 진행한다.

```text
Vue Frontend
  -> Spring Boot Backend
  -> Python Modeling Server
  -> Spring Boot Backend
  -> Vue Frontend
```

기본 웹 화면은 Vue로 구성한다.

백엔드는 Spring Boot LTS 버전을 기준으로 구성하며, JPA와 QueryDSL을 사용한다.

리스크 분석 모델링, 통계 기반 계산, 점수 산출은 Python 서버에서 담당한다.

Spring Boot 서버는 외부 공공 API 또는 수집 데이터를 저장, 정제, 관리하고 Python 모델링 서버에 필요한 입력 형태로 전달한다. Python 서버는 분석 결과를 계산한 뒤 Spring Boot 서버로 반환하고, Spring Boot 서버는 결과를 Vue 화면에 전달한다.

Spring Boot의 정확한 버전은 프로젝트 생성 시점에 공식 지원 중인 안정 버전을 확인해 고정한다.

## 2. Responsibility Split

| 영역 | 담당 시스템 | 책임 |
| --- | --- | --- |
| 웹 화면 | Vue Frontend | 사용자 입력, 결과 조회, 분석 이력 화면 |
| 사용자 요청 처리 | Spring Boot Backend | 요청 검증, 인증/세션, 분석 요청 생성 |
| 데이터 저장 | Spring Boot Backend + JPA | 사용자 입력, 분석 결과, 공공 기준 데이터 저장 |
| 동적 조회 | Spring Boot Backend + QueryDSL | 분석 이력, 기준 데이터, 조건 검색 |
| 외부 API 연동 | Spring Boot Backend | KOSIS, 공공데이터, 보건의료 데이터 수집 |
| 데이터 정제 | Spring Boot Backend | 외부 데이터를 내부 기준 데이터셋 형태로 정규화 |
| 모델 입력 생성 | Spring Boot Backend | Python 서버에 보낼 분석 요청 DTO 구성 |
| 리스크 계산 | Python Modeling Server | 시나리오 계산, 점수 모델, 통계 기반 분석 |
| 결과 해석 데이터 생성 | Python Modeling Server | 부족금액, 대응률, 생존개월, 회복기간, 점수 |
| 결과 저장 | Spring Boot Backend + JPA | Python 분석 결과 저장 |
| 결과 표시 | Vue Frontend | 사용자 리포트 화면 렌더링 |

## 3. Why This Split

이 구조를 선택하는 이유는 다음과 같다.

### 3.1 Vue Frontend

Vue는 사용자 입력 화면과 결과 리포트를 동적으로 구성하기에 적합하다.

Vue가 담당하기 좋은 것:

- 단계형 입력 폼
- 분석 결과 리포트 화면
- 리스크 점수/순위 시각화
- 사용자 입력 검증의 1차 피드백
- Spring Boot API 호출

### 3.2 Spring Boot Backend

Spring Boot는 웹 API, 데이터 저장, 외부 API 연동, 사용자 요청 처리에 적합하다.

Spring Boot가 담당하기 좋은 것:

- 사용자 입력 요청 처리
- 분석 이력 저장
- 공공데이터 수집 스케줄링
- DB 트랜잭션 관리
- JPA 기반 영속성 관리
- QueryDSL 기반 동적 조회
- 외부 API 호출
- 정제된 기준 데이터 관리
- Python 서버 호출
- Vue에 결과 API 제공

### 3.3 Python Modeling Server

Python은 데이터 분석, 통계 처리, 모델링 로직에 적합하다.

Python이 담당하기 좋은 것:

- 공공통계 기반 기준값 조회
- 질병별 의료비 기반 계산
- 시나리오별 손실 계산
- 점수 정규화
- 가중치 기반 최종 점수 산출
- 민감도 분석
- 향후 pandas, numpy, scikit-learn 기반 확장

## 4. Data Flow

### 4.1 Public Data Collection Flow

```text
External Public API / CSV
  -> Spring Boot Backend
  -> Raw Public Data Storage
  -> Normalized Baseline Dataset
  -> Database
```

설명:

- 외부 API 또는 CSV에서 공공 데이터를 수집한다.
- Spring Boot가 원천 데이터를 저장한다.
- Spring Boot가 프로젝트 내부 기준 데이터셋 형태로 정제한다.
- 정제된 데이터는 Python 모델링 서버가 사용할 수 있는 구조로 관리한다.

초기에는 외부 API 실시간 연동보다 정적 CSV/JSON 기반 데이터셋을 먼저 사용한다.

### 4.2 Risk Analysis Flow

```text
User Input
  -> Vue Frontend
  -> Spring Boot Backend
  -> Validate and Save Input
  -> Build Modeling Request
  -> Python Modeling Server
  -> Calculate Risk Results
  -> Spring Boot Backend
  -> Save Analysis Result
  -> Vue Frontend
  -> Render Report
```

설명:

- 사용자는 Vue 화면에서 정보를 입력한다.
- Spring Boot는 입력값을 검증하고 저장한다.
- Spring Boot는 기준 데이터와 사용자 입력을 조합해 Python 서버 요청을 만든다.
- Python 서버는 리스크 계산과 점수화를 수행한다.
- Spring Boot는 결과를 저장하고 Vue에 응답한다.
- Vue는 결과 리포트를 화면에 표시한다.

## 5. Boundary Between Vue, Spring Boot, and Python

각 시스템의 경계를 명확히 한다.

### Vue가 담당하지 않는 것

Vue는 화면 상태와 사용자 인터랙션을 담당하며, 도메인 계산과 데이터 영속성 책임을 갖지 않는다.

Vue에서 하지 않을 것:

- 리스크 계산
- 점수 산출
- 공공데이터 정제
- 분석 결과 영속화
- 외부 공공 API 직접 호출

### Spring Boot가 판단하지 않는 것

Spring Boot는 최종 리스크 점수나 모델 계산을 직접 수행하지 않는다.

Spring Boot에서 하지 않을 것:

- 리스크 점수 산출
- 질병별 손실 모델링
- 시나리오별 부족금액 계산
- 점수 정규화
- 리스크 순위 결정

### Python이 담당하지 않는 것

Python 서버는 웹 애플리케이션과 영속성 책임을 갖지 않는다.

Python에서 하지 않을 것:

- 사용자 화면 렌더링
- 사용자 입력 저장
- 분석 이력 저장
- 외부 API 원천 데이터 수집의 주 책임
- 로그인/세션/권한 처리

## 6. API Contract Direction

Vue와 Spring Boot, Spring Boot와 Python은 HTTP API로 통신한다.

```text
Vue -> Spring Boot: 사용자 입력/결과 조회 API
Spring Boot -> Python: 모델링 요청 API
```

초기 엔드포인트 후보:

```text
Vue -> Spring Boot:
POST /api/analyses
GET /api/analyses/{analysisId}

Spring Boot -> Python:
POST /risk-analysis
```

Spring -> Python 요청 예시 구조:

```json
{
  "analysisId": "analysis-001",
  "userProfile": {},
  "household": {},
  "incomeProfile": {},
  "expenseProfile": {},
  "assetProfile": {},
  "debtProfile": {},
  "protectionProfile": {},
  "baselineData": {
    "mortality": {},
    "cancer": {},
    "employment": {},
    "householdFinance": {},
    "medicalCost": {}
  },
  "assumptions": {}
}
```

Python -> Spring 응답 예시 구조:

```json
{
  "analysisId": "analysis-001",
  "riskResults": [
    {
      "riskType": "INCOME_INTERRUPTION",
      "score": 82,
      "grade": "HIGH",
      "requiredAmount": 24000000,
      "availableAmount": 8000000,
      "coverageGap": 16000000,
      "coverageRatio": 0.33,
      "liquiditySurvivalMonths": 2.1,
      "recoveryMonths": 18,
      "mainFactors": [
        "LOW_LIQUID_ASSET",
        "HIGH_FIXED_EXPENSE",
        "DEBT_PAYMENT_STRESS"
      ],
      "explanations": []
    }
  ],
  "ranking": [
    "INCOME_INTERRUPTION",
    "CRITICAL_ILLNESS",
    "DEATH"
  ],
  "modelVersion": "risk-model-v1",
  "assumptionVersion": "assumption-2026-001"
}
```

정확한 DTO는 `CALCULATION_SPEC.md`, `API_CONTRACT.md`, `BACKEND_DESIGN.md` 작성 시 확정한다.

## 7. Storage Direction

Spring Boot 서버가 저장 책임을 갖는다.

저장 대상:

- 사용자 입력값
- 분석 요청 이력
- Python 서버 분석 결과
- 공공 원천 데이터
- 정제된 기준 데이터셋
- 데이터 출처와 기준 연도
- 모델 버전
- 가정 버전

중요:

```text
분석 결과는 반드시 modelVersion, assumptionVersion, baselineDataVersion과 함께 저장한다.
```

이유:

- 공공 통계 기준 연도가 바뀌면 결과가 달라질 수 있다.
- 점수 가중치가 바뀌면 과거 결과와 현재 결과를 구분해야 한다.
- 포트폴리오에서 데이터 분석 재현성을 보여줄 수 있다.

## 8. Initial Implementation Order

구현은 다음 순서로 진행한다.

```text
1. 정적 기준 데이터셋 정의
2. Python 모델링 서버의 순수 계산 함수 작성
3. Python 서버 API 래핑
4. Spring 입력/결과 DTO 정의
5. Spring에서 Python 서버 호출
6. 분석 결과 저장
7. Vue 입력/결과 화면 구현
8. 외부 API 수집 기능 추가
```

주의:

```text
외부 API 연동을 먼저 하지 않는다.
계산 모델이 고정되기 전에는 데이터 수집 자동화보다 정적 기준 데이터셋이 우선이다.
```

## 9. Risks and Design Cautions

### 9.1 DTO Drift

Spring과 Python의 요청/응답 구조가 다르게 변하면 통합이 깨진다.

대응:

- API 요청/응답 스키마를 문서화한다.
- 예시 JSON을 테스트 데이터로 관리한다.
- 변경 시 양쪽이 함께 리뷰한다.

### 9.2 Logic Duplication

Spring과 Python 양쪽에 같은 계산식이 생기면 결과가 불일치할 수 있다.

대응:

- 계산 로직은 Python에 둔다.
- Spring Boot는 요청 생성, 저장, Vue API 제공만 담당한다.

### 9.3 Data Version Ambiguity

공공데이터 기준 연도와 모델 버전이 없으면 결과 재현이 어렵다.

대응:

- 모든 기준 데이터에 sourceYear, collectedAt, sourceName을 둔다.
- 분석 결과에 modelVersion, assumptionVersion, baselineDataVersion을 저장한다.

### 9.4 Python Server Failure

Python 서버 장애 시 Spring 화면에서 분석을 완료할 수 없다.

대응:

- Spring Boot는 분석 상태를 REQUESTED, PROCESSING, COMPLETED, FAILED로 관리한다.
- 실패 사유를 저장한다.
- 재시도 가능 구조를 고려한다.

## 10. Related Design Documents

이 아키텍처와 연결되는 문서는 다음과 같다.

| 문서 | 상태 | 목적 |
| --- | --- | --- |
| DATASET_SPEC.md | 작성됨 | Spring Boot가 저장/정제할 기준 데이터셋 정의 |
| CALCULATION_SPEC.md | 작성됨 | Python이 수행할 계산식 정의 |
| API_CONTRACT.md | 다음 작성 | Vue-Spring Boot, Spring Boot-Python 요청/응답 DTO 정의 |
| BACKEND_DESIGN.md | 다음 작성 | Spring Boot 모듈, JPA 엔티티, QueryDSL 조회, 분석 이력 구조 정의 |
| MODELING_SERVER_DESIGN.md | 다음 작성 | Python 서버 구조와 계산 모듈 정의 |

다음 작성 순서:

```text
API_CONTRACT.md
-> BACKEND_DESIGN.md
-> MODELING_SERVER_DESIGN.md
```
