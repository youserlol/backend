# 🧠 Multi-Table Hybrid Search API v3.0

FastAPI · Claude 4.5 Sonnet · Qdrant · PostgreSQL 기반 **지능형 하이브리드 검색 & 분석 엔진**

이 프로젝트는 **자연어 질의 → SQL 정형 필터링 + Vector 비정형 의미 검색 → Reranking → Insight 생성** 구조로 이루어진 고급 검색 엔진입니다.

사용자의 질문을 LLM이 해석하고, Qdrant와 PostgreSQL을 조합한 하이브리드 검색으로 최적의 패널 데이터를 찾아냅니다.

---

# 🚀 Features (주요 기능)

- **자연어 → SQL + Semantic JSON 자동 파싱**
- **PostgreSQL + Qdrant 벡터 DB 하이브리드 검색**
- **Field-aware Semantic Routing** (질문과 가장 관련된 컬럼 자동 매칭)
- **Candidate Reranking 알고리즘**
- **Negative Filtering** (부정 응답 제거)
- **통계 차트 + 인사이트 요약 생성**

---

# 🏗️ System Architecture

아키텍처는 **Layered Architecture + Repository Pattern** 기반입니다.

```Javascript
graph TD
    User[Client / Frontend] -->|REST API Request| Main[Controller (main.py)]

    subgraph "Application Layer"
        Main --> Service[Service Orchestrator (services.py)]
    end

    subgraph "Logic Layer"
        Service --> LLM[Query Parser (llm.py)]
        Service --> Router[Semantic Router (semantic_router.py)]
        Service --> Search[Search Engine (search.py)]
        Service --> Insight[Analyst (insights.py)]
    end

    subgraph "Data Access Layer (Repository)"
        Search --> Repo[Repository (repository.py)]
        Insight --> Repo
    end

    subgraph "Infrastructure"
        Repo -->|SQL Filter| PG[(PostgreSQL: User Meta)]
        Repo -->|Vector Search| Qdrant[(Qdrant: Survey Data)]
    end

```

---

# 📂 Folder & Module Structure

## 1. Core Logic

### `main.py` — **Controller**

- FastAPI 엔드포인트 정의
- 요청 수신 → `services.py` 호출 → 응답 반환
- 비동기 API 처리 담당

### `services.py` — **Orchestrator**

- 검색 플로우 전체를 조율
    - LLM 분석
    - Routing
    - Hybrid Search
    - Insight 생성
- 모듈 간 트랜잭션 흐름 제어

### `search.py` — **Hybrid Search Engine (핵심)**

- SQL 후보군 존재 여부에 따라 Reranking 전략 선택
- PostgreSQL + Qdrant 하이브리드 검색 실행
- Python Fuzzy Match + Negative Filtering 수행

### `llm.py` — **Query Parser**

- Claude 3.5 Sonnet으로 자연어 → 구조화된 JSON 변환
    - `Demographic Filters` (SQL 용)
    - `Semantic Conditions` (Vector 용)

### `semantic_router.py` — **Field Routing**

- 질의와 가장 관련된 DB 컬럼을 벡터 기반으로 자동 매핑
- 예: “OTT” → `ott_count`, `most_used_app`

---

## 2. Data Layer

### `repository.py`

- **DB 접근 로직 분리**
- `PanelRepository` → PostgreSQL 조회
- `VectorRepository` → Qdrant 벡터 검색

### `db.py`

- PostgreSQL Connection Pool 관리
- Qdrant Client 싱글톤 인스턴스 유지

---

## 3. Support / Config

### `mapping_rules.py`

- 자연어 → 표준화된 비즈니스 룰 매핑
    - “MZ세대” → “20~30대”
    - “고소득” → “월 500 이상”
- 부정 표현 정규식 패턴(“안 본다”, “관심 없음”)

### `insights.py`

- 검색 결과 기반 차트 생성 (Bar/Pie)
- 통계 정리 및 한 줄 요약 인사이트 생성

### `settings.py`

- AWS Secrets Manager → 환경 변수 안전 로딩

---

# 🔄 Search Workflow (검색 플로우)

사용자 예시 질의:

> “서울 사는 30대 중 OTT를 즐겨 보는 사람 찾아줘”
> 

### **1. Query Understanding (llm.py)**

입력 질의 → SQL 필터 + 의미 조건 JSON 생성

```json
{
  "sql_filters": { "region_major": "서울", "age_range": [30, 39] },
  "semantic": "OTT를 즐겨 보는"
}

```

---

### **2. Semantic Routing (semantic_router.py)**

- “OTT” → `ott_count`, `most_used_app` 컬럼과 연관도 높다고 판단
- 해당 필드 중심으로 벡터 검색 스코어 계산

---

### **3. Hybrid Search + Reranking (search.py)**

### Case A — SQL 후보군 있음

1. PostgreSQL에서 후보 패널 ID 추출
2. **해당 ID만 대상으로** Qdrant Vector Reranking
3. 부정 응답 제거
4. Python Fuzzy Matching으로 미세 정합

### Case B — SQL 후보 없음

- 전체 Qdrant 컬렉션 Vector Search 실행

---

### **4. Aggregation (repository.py)**

`asyncio.gather`로 패널 메타 + 설문 벡터 정보 병렬 조회

---

### **5. Insight Generation (insights.py)**

- 통계 차트 생성
- 패턴 분석 → 한 줄 인사이트 생성 후 반환

---

# 💾 Data Schema

### **PostgreSQL — welcome_meta2**

정형 메타데이터 저장

- `panel_id`, `gender`, `birth_year`, `region_major`, `income_monthly`

### **Qdrant — qpoll_vectors_v2**

설문 벡터 저장

- `panel_id`, `question`, `sentence`, `vector`

### **Qdrant — welcome_subjective_vectors**

주관식 선호 데이터

- 취미, 가치관, 라이프스타일 텍스트 벡터

---

# 🛠️ Tech Stack

| 분야 | 기술 |
| --- | --- |
| Language | Python 3.11 |
| Backend | FastAPI, Uvicorn |
| Database | PostgreSQL(psycopg2), Qdrant |
| AI/ML | Claude 3.5 Sonnet, HuggingFace Embeddings |
| Architecture | Layered Architecture, Repository Pattern |
| Infra | Docker, AWS Secrets Manager |
| Orchestration | asyncio 기반 병렬 처리 |


## 의존성 설치
pip install -r requirements.txt

---

## 🛠️ 실행 방법

#### 1. 가상환경 활성화
- .\venv\Scripts\activate

#### 2. API 서버 실행
- uvicorn main:app --reload
- uvicorn main:app --reload --log-config log_config.json
