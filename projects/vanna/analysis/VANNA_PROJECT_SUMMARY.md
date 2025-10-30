# Vanna Project - 전체 요약

## 프로젝트 개요

**Vanna**는 자연어를 SQL로 변환하는 Text-to-SQL 프레임워크입니다. RAG (Retrieval-Augmented Generation)를 활용하여 데이터베이스 스키마를 학습하고, 사용자 질문에 맞는 SQL을 생성합니다.

**핵심 특징**:
- 🔌 **플러그인 아키텍처**: Vector Store와 LLM을 자유롭게 조합
- 🎯 **RAG 기반**: Fine-tuning 없이도 높은 정확도
- 🗄️ **11개 DB 지원**: PostgreSQL, MySQL, Snowflake, BigQuery 등
- 📊 **시각화 자동 생성**: Plotly 차트 코드 자동 생성
- ☁️ **Cloud + Local**: Vanna.AI Cloud 또는 Self-hosted

---

## 아키텍처 전체 구조

```
┌─────────────────────────────────────────────────────────────────┐
│                        User Application                          │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             │ ask("지난달 매출 상위 10개 상품은?")
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                   VannaBase (base/base.py)                       │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  High-Level API: ask(), train(), generate_sql()          │  │
│  └───────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  RAG Pipeline: Retrieval → Augmentation → Generation     │  │
│  └───────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Database Abstraction: 11 DB connections (dynamic bind)  │  │
│  └───────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Abstract Methods (구현 필요)                              │  │
│  │  - submit_prompt()        (LLM 구현)                      │  │
│  │  - add_question_sql()     (Vector Store 구현)            │  │
│  │  - add_ddl()              (Vector Store 구현)            │  │
│  │  - get_similar_question_sql() (Vector Store 구현)        │  │
│  └───────────────────────────────────────────────────────────┘  │
└────────────────────┬──────────────────────┬─────────────────────┘
                     │                      │
         ┌───────────┴──────────┐    ┌─────┴────────────────┐
         ▼                      ▼    ▼                      ▼
┌──────────────────┐  ┌──────────────────┐ ┌──────────────────┐
│ ChromaDB_Vector  │  │   OpenAI_Chat    │ │  Other LLMs      │
│ Store            │  │                  │ │ (Anthropic, etc) │
│ (chromadb/       │  │ (openai/         │ │                  │
│  chromadb_       │  │  openai_chat.py) │ │                  │
│  vector.py)      │  │                  │ │                  │
│                  │  │                  │ │                  │
│ - 3 collections  │  │ - Auto model     │ │                  │
│ - Deterministic  │  │   selection      │ │                  │
│   UUID           │  │ - Azure support  │ │                  │
│ - Persistent/    │  │ - Token counting │ │                  │
│   In-memory      │  │                  │ │                  │
└──────────────────┘  └──────────────────┘ └──────────────────┘
         │                      │                      │
         │                      │                      │
         ▼                      ▼                      ▼
┌──────────────────┐  ┌──────────────────────────────────────┐
│   ChromaDB       │  │         OpenAI API                   │
│   (Local/Remote) │  │         (gpt-3.5-turbo, gpt-4)      │
└──────────────────┘  └──────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                   사용자 코드 (Mixin 패턴)                        │
│                                                                  │
│  class MyVanna(ChromaDB_VectorStore, OpenAI_Chat):              │
│      def __init__(self, config=None):                           │
│          ChromaDB_VectorStore.__init__(self, config)            │
│          OpenAI_Chat.__init__(self, config)                     │
│                                                                  │
│  vn = MyVanna(config={"api_key": "sk-...", "model": "gpt-4"})  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 핵심 설계 결정

### 1. Mixin 패턴 (가장 중요!)

**문제**: Vector Store와 LLM을 독립적으로 조합하고 싶다

**해결**: Abstract Base Class + Mixin 패턴

```python
# base.py
class VannaBase(ABC):
    @abstractmethod
    def submit_prompt(self, prompt) -> str:
        """LLM 구현 필요"""
        pass

    @abstractmethod
    def add_question_sql(self, question: str, sql: str) -> str:
        """Vector Store 구현 필요"""
        pass

# 사용자 코드
class MyVanna(ChromaDB_VectorStore, OpenAI_Chat):
    pass  # 두 Mixin이 VannaBase를 완성

vn = MyVanna()
```

**장점**:
- Vector Store와 LLM을 자유롭게 조합
- 새로운 구현 추가 시 base.py 수정 불필요
- 단일 책임 원칙 (SRP)

**조합 예시**:
- ChromaDB + OpenAI
- Pinecone + Anthropic Claude
- Qdrant + Local LLM
- Custom Vector Store + Custom LLM

**관련 파일**:
- `base.py` (2118줄) - Abstract methods 정의
- `chromadb_vector.py` (257줄) - Vector Store 구현
- `openai_chat.py` (128줄) - LLM 구현

---

### 2. RAG vs Fine-tuning

**문제**: 데이터베이스 스키마를 LLM에게 어떻게 학습시킬까?

**Option A: Fine-tuning**
- 장점: 높은 정확도
- 단점: 비용, 시간, 스키마 변경 시 재학습 필요

**Option B: RAG (Retrieval-Augmented Generation)** ✅
- 장점: 즉시 업데이트, 낮은 비용
- 단점: 프롬프트 길이 제한

**Vanna의 선택**: RAG

**구현** (base.py:93-267):
```python
def generate_sql(self, question: str):
    # 1. Retrieval (Vector Search)
    question_sql_list = self.get_similar_question_sql(question)
    ddl_list = self.get_related_ddl(question)
    doc_list = self.get_related_documentation(question)

    # 2. Augmentation (Prompt 구성)
    prompt = self.get_sql_prompt(
        question=question,
        question_sql_list=question_sql_list,
        ddl_list=ddl_list,
        doc_list=doc_list
    )

    # 3. Generation (LLM 호출)
    llm_response = self.submit_prompt(prompt)

    # 4. Extraction
    sql = self.extract_sql(llm_response)
    return sql
```

**Vector Store 구조** (chromadb_vector.py):
```python
# 3개의 Collection으로 분리
self.sql_collection = chroma_client.get_or_create_collection("sql")
self.ddl_collection = chroma_client.get_or_create_collection("ddl")
self.documentation_collection = chroma_client.get_or_create_collection("documentation")
```

**검색 로직**:
- 질문과 유사한 과거 SQL 검색 (Few-shot learning)
- 관련 테이블 스키마 검색 (DDL)
- 비즈니스 컨텍스트 검색 (Documentation)

**관련 파일**:
- `base.py` - Part 1: RAG workflow (lines 93-382)
- `chromadb_vector.py` - Vector Store 구현

---

### 3. Dynamic Function Binding (Database 독립성)

**문제**: 11개의 서로 다른 DB API를 어떻게 통합할까?

**Option A: Adapter 패턴 (각 DB마다 클래스)**
```python
class PostgresAdapter:
    def run_sql(self, sql): ...

class MySQLAdapter:
    def run_sql(self, sql): ...
```
- 장점: 명확한 분리
- 단점: 클래스 폭발, 코드 중복

**Option B: Dynamic Function Binding** ✅
```python
def connect_to_postgres(self, host, dbname, user, password, port):
    def connect_to_db():
        return psycopg2.connect(host=host, dbname=dbname, ...)

    def run_sql_postgres(sql: str) -> pd.DataFrame:
        conn = connect_to_db()
        cs = conn.cursor()
        cs.execute(sql)
        results = cs.fetchall()
        columns = [desc[0] for desc in cs.description]
        return pd.DataFrame(results, columns=columns)

    # 런타임에 함수를 인스턴스 메서드로 바인딩!
    self.run_sql = run_sql_postgres
    self.run_sql_is_set = True
```
- 장점: 코드 간결, Closure로 connection 캡슐화
- 단점: 런타임 바인딩 (IDE 자동완성 어려움)

**지원 DB**:
- PostgreSQL, MySQL, SQLite, Snowflake, BigQuery
- DuckDB, ClickHouse, Oracle, Hive, Presto, Databricks

**관련 파일**:
- `base.py` - Part 3: Database (lines 761-1682)

---

### 4. Two-Step SQL Generation (정확도 향상)

**문제**: LLM이 데이터의 실제 값을 모르면 정확한 SQL을 생성하기 어렵다

**예시**:
```
질문: "John Smith의 주문 내역을 보여줘"
문제: DB에는 "john.smith@email.com"로 저장되어 있음 (이름이 아닌 이메일)
```

**해결**: Intermediate SQL

```python
def generate_sql(self, question: str, allow_llm_to_see_data=False):
    # 1차: LLM이 Intermediate SQL 생성
    llm_response = self.submit_prompt(prompt)

    if self.is_sql_valid(llm_response) and 'intermediate_sql' in llm_response:
        # 2. Intermediate SQL 실행
        intermediate_sql = self.extract_sql(llm_response)
        df = self.run_sql(intermediate_sql)

        # 3. 데이터를 보고 Final SQL 생성
        prompt_with_data = self.add_data_to_prompt(prompt, df)
        llm_response = self.submit_prompt(prompt_with_data)

    return self.extract_sql(llm_response)
```

**흐름**:
```
질문: "John Smith의 주문 내역"
↓
LLM: Intermediate SQL 생성
  SELECT DISTINCT email FROM users WHERE name LIKE '%John%Smith%'
↓
실행 결과: john.smith@email.com
↓
LLM: Final SQL 생성 (실제 데이터 활용)
  SELECT * FROM orders WHERE user_email = 'john.smith@email.com'
```

**관련 파일**:
- `base.py` - Part 1: RAG workflow (lines 131-267)

---

### 5. Token Budget 관리

**문제**: GPT-3.5-turbo의 컨텍스트는 4K (16K 모델: 16K). 수십 개의 테이블 DDL을 모두 넣으면?

**해결**: Dynamic Token Budget

```python
def add_ddl_to_prompt(
    self,
    initial_prompt: str,
    ddl_list: list,
    max_tokens: int = 14000
) -> str:
    if len(ddl_list) > 0:
        initial_prompt += "\n\nYou may use the following DDL statements as a reference:\n\n"

    for ddl in ddl_list:
        # 현재 프롬프트 토큰 수
        current_tokens = self.str_to_approx_token_count(initial_prompt)

        # DDL 추가 시 토큰 수
        ddl_tokens = self.str_to_approx_token_count(ddl)

        # Budget 초과 여부 체크
        if current_tokens + ddl_tokens < max_tokens:
            initial_prompt += f"{ddl}\n\n"
        else:
            # Budget 초과 → 더 이상 추가 안 함
            break

    return initial_prompt
```

**우선순위**:
1. System prompt (항상 포함)
2. Few-shot examples (가장 중요)
3. DDL (가능한 만큼)
4. Documentation (가능한 만큼)

**토큰 계산**:
```python
def str_to_approx_token_count(self, string: str) -> int:
    return len(string) / 4  # 근사치: 4 chars ≈ 1 token
```

**관련 파일**:
- `base.py` - Part 2: Abstraction (lines 606-693)
- `openai_chat.py` - 자동 모델 선택 (3500 토큰 기준)

---

### 6. Training Plan (대량 학습)

**문제**: 수백 개의 테이블을 모두 학습하면 비용과 시간이 많이 든다

**해결**: Preview → Modify → Execute 패턴

```python
# 1. Plan 생성 (DB 메타데이터 자동 스캔)
plan = vn.get_training_plan_postgres(
    filter_schemas=["public"],
    use_historical_queries=True
)

# 2. Preview
print(plan.get_summary())
# Train on DDL: public users
# Train on DDL: public orders
# Train on DDL: public temp_table  ← 제거하고 싶음

# 3. Modify
plan.remove_item("Train on DDL: public temp_table")

# 4. Execute
vn.train(plan=plan)
```

**TrainingPlan 구조** (__init__.py:171-250):
```python
@dataclass
class TrainingPlanItem:
    item_type: str      # "sql", "ddl", "is"
    item_group: str     # Schema
    item_name: str      # Table name
    item_value: str     # Actual DDL

class TrainingPlan:
    _plan: List[TrainingPlanItem]

    def get_summary(self) -> List[str]:
        return [str(item) for item in self._plan]

    def remove_item(self, item: str):
        # 불필요한 항목 제거
        for plan_item in self._plan:
            if str(plan_item) == item:
                self._plan.remove(plan_item)
```

**장점**:
- 자동화 (DB 스캔)
- 투명성 (무엇을 학습할지 미리 확인)
- 제어 (불필요한 항목 제거)

**관련 파일**:
- `__init__.py` - TrainingPlan 클래스 정의
- `base.py` - Part 4: High-level API (train 메서드)

---

### 7. API 마이그레이션 전략

**문제**: Vanna v0.x는 global 함수 방식, v1.0은 인스턴스 방식. 기존 사용자 코드를 어떻게 처리?

**해결**: Breaking Change + 명확한 에러 메시지

```python
def error_deprecation():
    raise Exception("""
Please switch to the following method for initializing Vanna:

from vanna.remote import VannaDefault

api_key = # Your API key from https://vanna.ai/account/profile
vanna_model_name = # Your model name from https://vanna.ai/account/profile

vn = VannaDefault(model=vanna_model_name, api_key=api_key)
""")

# 모든 레거시 함수
def set_api_key(key: str) -> None:
    error_deprecation()

def generate_sql(question: str) -> str:
    error_deprecation()

# ... 30개 이상의 함수가 동일
```

**Deprecation 레벨**:
1. 문서에만 명시 (약함)
2. Warning 로그 (중간)
3. DeprecationWarning (강함)
4. **즉시 Error** ✅ (매우 강함) ← Vanna의 선택

**효과**:
- 사용자가 즉시 마이그레이션
- 기술 부채 빠르게 제거
- 코드베이스 정리

**관련 파일**:
- `__init__.py` - 모든 레거시 함수가 deprecated

---

## 파일별 역할 요약

| 파일 | 라인 수 | 역할 | 핵심 패턴 |
|------|---------|------|-----------|
| `base.py` | 2118 | 핵심 프레임워크 | RAG, Mixin, Dynamic Binding, Pipeline |
| `chromadb_vector.py` | 257 | Vector Store 구현 | 3-collection 분리, Deterministic UUID |
| `openai_chat.py` | 128 | LLM 구현 | Client 주입, 자동 모델 선택, Azure 호환 |
| `__init__.py` | 399 | 엔트리 포인트 | API 마이그레이션, OTP 인증, TrainingPlan |

### base.py (핵심 프레임워크)

**구조**:
- Part 1: RAG Workflow (lines 93-382)
  - `generate_sql()`, `extract_sql()`, Intermediate SQL
- Part 2: Abstraction (lines 383-693)
  - Abstract methods, Prompt generation, Token budget
- Part 3: Database (lines 761-1682)
  - 11개 DB 연결, Dynamic function binding, Closure
- Part 4: High-level API (lines 1683-2118)
  - `ask()`, `train()`, Training Plan, Visualization

**핵심 메서드**:
```python
def ask(question, print_results=True, auto_train=True, visualize=True):
    """All-in-one API: SQL 생성 → 실행 → 학습 → 시각화"""
    sql = self.generate_sql(question)
    df = self.run_sql(sql)
    if auto_train:
        self.add_question_sql(question, sql)
    if visualize:
        fig = self.get_plotly_figure(...)
    return sql, df, fig
```

### chromadb_vector.py (Vector Store)

**핵심 설계**:
- 3개 Collection 분리 (SQL/DDL/Documentation)
- Deterministic UUID (중복 방지)
- Persistent vs In-memory 모드

```python
def add_ddl(self, ddl: str) -> str:
    id = deterministic_uuid(ddl) + "-ddl"
    self.ddl_collection.add(documents=ddl, ids=id)
    return id
```

### openai_chat.py (LLM)

**핵심 설계**:
- Client 주입 (테스트 용이)
- 자동 모델 선택 (3500 토큰 기준)
- Azure OpenAI 호환

```python
def submit_prompt(self, prompt, **kwargs) -> str:
    num_tokens = sum(len(m["content"]) / 4 for m in prompt)

    # 자동 모델 선택
    if num_tokens > 3500:
        model = "gpt-3.5-turbo-16k"
    else:
        model = "gpt-3.5-turbo"

    response = self.client.chat.completions.create(model=model, messages=prompt)
    return response.choices[0].message.content
```

### __init__.py (엔트리 포인트)

**핵심 설계**:
- OTP 인증 (Vanna.AI Cloud)
- TrainingPlan 클래스
- 30개 이상 레거시 함수 deprecated

```python
def get_api_key(email: str, otp_code=None) -> str:
    # 1. 환경 변수 체크
    if os.environ.get("VANNA_API_KEY"):
        return os.environ["VANNA_API_KEY"]

    # 2. OTP 전송
    __unauthenticated_rpc_call("send_otp", [UserEmail(email=email)])

    # 3. OTP 입력
    otp_code = input("Check your email for the code and enter it here: ")

    # 4. API Key 발급
    key = __unauthenticated_rpc_call("verify_otp", [UserOTP(email=email, otp=otp_code)])
    return key["key"]
```

---

## 실전 활용 가이드

### 1. 기본 사용 (Local)

```python
from vanna.chromadb import ChromaDB_VectorStore
from vanna.openai import OpenAI_Chat

# Mixin으로 조합
class MyVanna(ChromaDB_VectorStore, OpenAI_Chat):
    def __init__(self, config=None):
        ChromaDB_VectorStore.__init__(self, config=config)
        OpenAI_Chat.__init__(self, config=config)

# 초기화
vn = MyVanna(config={
    "api_key": "sk-...",  # OpenAI API Key
    "model": "gpt-3.5-turbo"
})

# DB 연결
vn.connect_to_postgres(
    host="localhost",
    dbname="mydb",
    user="user",
    password="pass"
)

# 학습
vn.train(ddl="CREATE TABLE users (id INT, name TEXT, email TEXT)")
vn.train(
    question="전체 사용자 수는?",
    sql="SELECT COUNT(*) FROM users"
)

# SQL 생성
sql = vn.generate_sql("이메일이 gmail인 사용자는 몇 명?")
print(sql)
# SELECT COUNT(*) FROM users WHERE email LIKE '%@gmail.com'

# 실행 + 시각화
sql, df, fig = vn.ask("월별 가입자 수 추이를 차트로 보여줘")
fig.show()  # Plotly 차트
```

### 2. Cloud 사용 (Vanna.AI)

```python
from vanna.remote import VannaDefault
import vanna as vn

# 1. API Key 발급 (첫 실행만)
api_key = vn.get_api_key(email="user@example.com")
# Check your email for the code and enter it here: 123456

# 2. Vanna 초기화
vanna_instance = VannaDefault(
    model="my-sales-model",
    api_key=api_key
)

# 3. 이후 동일
vanna_instance.connect_to_postgres(...)
vanna_instance.train(...)
sql = vanna_instance.generate_sql("...")
```

### 3. Training Plan으로 대량 학습

```python
# 1. Training Plan 생성
plan = vn.get_training_plan_postgres(
    filter_schemas=["public"],
    use_historical_queries=True  # pg_stat_statements에서 과거 쿼리 가져오기
)

# 2. Preview
for item in plan.get_summary():
    print(item)

# Output:
# Train on DDL: public users
# Train on DDL: public orders
# Train on DDL: public temp_table
# Train on SQL: SELECT * FROM users WHERE ...

# 3. 필터링
plan.remove_item("Train on DDL: public temp_table")

# 4. 실행
vn.train(plan=plan)
```

### 4. Custom LLM 구현

```python
from vanna.base import VannaBase
from vanna.chromadb import ChromaDB_VectorStore

class Anthropic_Chat(VannaBase):
    def __init__(self, config=None):
        VannaBase.__init__(self, config=config)
        from anthropic import Anthropic
        self.client = Anthropic(api_key=config["api_key"])

    def submit_prompt(self, prompt, **kwargs) -> str:
        # Claude API 형식으로 변환
        system = next(m["content"] for m in prompt if m["role"] == "system")
        messages = [m for m in prompt if m["role"] != "system"]

        response = self.client.messages.create(
            model="claude-3-sonnet-20240229",
            system=system,
            messages=messages,
            max_tokens=1024
        )
        return response.content[0].text

    def system_message(self, message: str) -> dict:
        return {"role": "system", "content": message}

    def user_message(self, message: str) -> dict:
        return {"role": "user", "content": message}

    def assistant_message(self, message: str) -> dict:
        return {"role": "assistant", "content": message}

# 사용
class MyVanna(ChromaDB_VectorStore, Anthropic_Chat):
    def __init__(self, config=None):
        ChromaDB_VectorStore.__init__(self, config=config)
        Anthropic_Chat.__init__(self, config=config)

vn = MyVanna(config={"api_key": "sk-ant-..."})
```

### 5. Production 설정

```python
from vanna.chromadb import ChromaDB_VectorStore
from vanna.openai import OpenAI_Chat
from openai import OpenAI
import os

class MyVanna(ChromaDB_VectorStore, OpenAI_Chat):
    def __init__(self, config=None):
        ChromaDB_VectorStore.__init__(self, config=config)
        OpenAI_Chat.__init__(self, config=config)

# Client 주입 (타임아웃, 리트라이 설정)
client = OpenAI(
    api_key=os.getenv("OPENAI_API_KEY"),
    timeout=30.0,
    max_retries=3
)

vn = MyVanna(
    client=client,
    config={
        "model": "gpt-3.5-turbo",
        "temperature": 0.0,  # 결정적 출력
        # ChromaDB persistent mode
        "path": "./chroma_db",
        "client_settings": {
            "anonymized_telemetry": False
        }
    }
)

# DB 연결 (Connection Pool 사용)
from sqlalchemy import create_engine
engine = create_engine(
    f"postgresql://{user}:{password}@{host}:{port}/{dbname}",
    pool_size=10,
    max_overflow=20
)

vn.run_sql = lambda sql: pd.read_sql(sql, engine)

# 에러 핸들링
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=4, max=60)
)
def generate_sql_with_retry(question):
    return vn.generate_sql(question)

# 사용
try:
    sql = generate_sql_with_retry("지난달 매출은?")
except Exception as e:
    logging.error(f"Failed to generate SQL: {e}")
    # Fallback 처리
```

---

## 스케일별 전략

### Small Scale (< 10 테이블, < 100 req/day)

**특징**:
- 개발 환경 또는 PoC
- Training Plan 불필요
- In-memory Vector Store 가능

**권장 설정**:
```python
vn = MyVanna(config={
    "api_key": "sk-...",
    "model": "gpt-3.5-turbo",
    # In-memory ChromaDB
    "client": chromadb.Client()
})

# 수동 학습 (10개 이하)
vn.train(ddl="CREATE TABLE users (...)")
vn.train(ddl="CREATE TABLE orders (...)")
```

**비용**: $10-50/month

---

### Medium Scale (10-100 테이블, 100-1K req/day)

**특징**:
- 프로덕션 초기
- Training Plan 필수
- Persistent Vector Store

**권장 설정**:
```python
# Persistent ChromaDB
vn = MyVanna(config={
    "api_key": "sk-...",
    "model": "gpt-3.5-turbo",
    "path": "./chroma_db"  # Persistent
})

# Training Plan으로 대량 학습
plan = vn.get_training_plan_postgres(filter_schemas=["public"])
plan.remove_item("Train on DDL: public temp_table")
vn.train(plan=plan)

# Rate Limit 대응
from tenacity import retry
@retry(stop=stop_after_attempt(3))
def generate_sql_with_retry(question):
    return vn.generate_sql(question)
```

**비용**: $50-500/month

**최적화**:
- 자동 모델 선택 활용 (gpt-3.5-turbo vs 16k)
- 캐싱 레이어 추가 (Redis)

---

### Large Scale (100+ 테이블, 1K+ req/day)

**특징**:
- 대규모 엔터프라이즈
- Domain별 모델 분리
- 비용 최적화 필수

**권장 설정**:

#### 1. Domain별 모델 분리
```python
# Sales 모델
vn_sales = MyVanna(config={..., "path": "./chroma_db_sales"})
plan_sales = vn_sales.get_training_plan_postgres(filter_schemas=["sales"])
vn_sales.train(plan=plan_sales)

# Analytics 모델
vn_analytics = MyVanna(config={..., "path": "./chroma_db_analytics"})
plan_analytics = vn_analytics.get_training_plan_postgres(filter_schemas=["analytics"])
vn_analytics.train(plan=plan_analytics)
```

**장점**:
- 도메인별 정확도 향상 (20-30%)
- 검색 속도 증가 (작은 Vector Store)
- 권한 분리

#### 2. Tier-based 모델 선택
```python
class SmartVanna(MyVanna):
    def generate_sql(self, question, **kwargs):
        complexity = self.estimate_complexity(question)

        if complexity == "simple":
            kwargs["model"] = "gpt-3.5-turbo"  # 저렴
        elif complexity == "complex":
            kwargs["model"] = "gpt-4"  # 비싸지만 정확

        return super().generate_sql(question, **kwargs)

    def estimate_complexity(self, question):
        # JOIN, 서브쿼리, 집계 함수 등으로 판단
        if any(keyword in question.lower() for keyword in ["join", "subquery", "aggregate"]):
            return "complex"
        return "simple"
```

**비용 절감**: 30-50%

#### 3. 캐싱 레이어
```python
import hashlib
import redis

redis_client = redis.Redis(host='localhost', port=6379)

def generate_sql_cached(question):
    cache_key = hashlib.md5(question.encode()).hexdigest()
    cached = redis_client.get(cache_key)

    if cached:
        return cached.decode()

    sql = vn.generate_sql(question)
    redis_client.setex(cache_key, 3600, sql)  # 1시간 캐시
    return sql
```

**캐시 Hit Rate**: 20-40% → 비용 20-40% 절감

#### 4. 모니터링 (Prometheus + Grafana)
```python
from prometheus_client import Counter, Histogram

token_counter = Counter('vanna_tokens_total', 'Total tokens used', ['model'])
latency_histogram = Histogram('vanna_latency_seconds', 'Request latency')

class MonitoredVanna(MyVanna):
    def submit_prompt(self, prompt, **kwargs):
        with latency_histogram.time():
            response = super().submit_prompt(prompt, **kwargs)

        num_tokens = sum(len(m["content"]) / 4 for m in prompt)
        model = kwargs.get("model", "gpt-3.5-turbo")
        token_counter.labels(model=model).inc(num_tokens)

        return response
```

**비용**: $500+/month

---

## 핵심 패턴 요약

### 1. Mixin 패턴 (구조적 패턴)

**목적**: 여러 기능을 독립적으로 조합

**구현**:
```python
class VannaBase(ABC):
    @abstractmethod
    def abstract_method(self): pass

class FeatureA(VannaBase):
    def abstract_method(self): return "A"

class FeatureB(VannaBase):
    def another_method(self): return "B"

class Combined(FeatureA, FeatureB):
    pass  # 두 Mixin을 조합
```

**적용**:
- Vector Store + LLM 조합
- 새 구현 추가 시 base 수정 불필요

---

### 2. RAG Pipeline (아키텍처 패턴)

**목적**: Fine-tuning 없이 높은 정확도

**흐름**:
```
질문 → Retrieval (Vector Search) → Augmentation (Prompt 구성) → Generation (LLM) → SQL
```

**적용**:
- ChromaDB로 유사 SQL/DDL 검색
- Few-shot learning (과거 예시 활용)

---

### 3. Dynamic Function Binding (Closure 패턴)

**목적**: DB 독립성

**구현**:
```python
def connect_to_db():
    conn = psycopg2.connect(...)
    return conn

def run_sql(sql):
    conn = connect_to_db()  # Closure!
    ...

self.run_sql = run_sql  # 런타임 바인딩
```

**적용**:
- 11개 DB 지원
- Connection pool 캡슐화

---

### 4. Token Budget 관리 (제약 조건 패턴)

**목적**: LLM 컨텍스트 길이 제한 대응

**구현**:
```python
current_tokens = count_tokens(prompt)
for item in items:
    if current_tokens + count_tokens(item) < max_tokens:
        prompt += item
    else:
        break
```

**적용**:
- 14K 토큰 budget
- 우선순위 기반 추가

---

### 5. Template Method (행동 패턴)

**목적**: 공통 알고리즘 + 구현 위임

**구현**:
```python
class VannaBase:
    def generate_sql(self, question):
        # 1. Retrieval (구현 위임)
        examples = self.get_similar_question_sql(question)  # Abstract

        # 2. Augmentation (공통 로직)
        prompt = self.build_prompt(question, examples)

        # 3. Generation (구현 위임)
        response = self.submit_prompt(prompt)  # Abstract

        return self.extract_sql(response)
```

**적용**:
- `generate_sql()` 전체 흐름
- 각 단계는 abstract method로 위임

---

### 6. Pipeline 패턴 (통합 패턴)

**목적**: 복잡한 워크플로우 단일 API로 통합

**구현**:
```python
def ask(self, question):
    # 1. Generate SQL
    sql = self.generate_sql(question)

    # 2. Execute
    df = self.run_sql(sql)

    # 3. Auto-train (optional)
    if auto_train:
        self.add_question_sql(question, sql)

    # 4. Visualize (optional)
    if visualize:
        fig = self.generate_plotly_figure(question, df)

    return sql, df, fig
```

**적용**:
- `ask()` - 6단계 통합
- 사용자는 한 줄로 모든 기능 사용

---

### 7. Preview-Modify-Execute (사용자 제어 패턴)

**목적**: 자동화 + 사용자 제어

**구현**:
```python
# 1. Preview
plan = create_plan()
print(plan.summary())

# 2. Modify
plan.remove_item("unwanted")

# 3. Execute
execute(plan)
```

**적용**:
- TrainingPlan
- 대량 데이터 처리 전 확인 가능

---

## 프로젝트에서 배운 핵심 교훈

### 1. 추상화의 힘 (Mixin 패턴)

**교훈**: 기능을 독립적인 Mixin으로 분리하면 조합의 유연성이 극대화됩니다.

**적용**:
- Vector Store와 LLM을 완전히 독립적으로 구현
- 사용자가 원하는 조합 선택 가능
- 새 구현 추가 시 기존 코드 수정 불필요

**다른 프로젝트에 적용**:
```python
# 예: Web Framework
class RequestHandler(ABC): ...
class JSONSerializer(RequestHandler): ...
class XMLSerializer(RequestHandler): ...
class BasicAuth(RequestHandler): ...
class JWTAuth(RequestHandler): ...

# 조합
class MyAPI(JSONSerializer, JWTAuth):
    pass

class LegacyAPI(XMLSerializer, BasicAuth):
    pass
```

---

### 2. RAG vs Fine-tuning 선택

**교훈**: 빠르게 변하는 데이터는 RAG가 유리합니다.

**RAG 선택 조건**:
- ✅ 데이터가 자주 변경 (DB 스키마)
- ✅ 즉시 업데이트 필요
- ✅ 비용 민감

**Fine-tuning 선택 조건**:
- ✅ 데이터가 고정적 (도메인 지식)
- ✅ 초저지연 필요
- ✅ 오프라인 사용

**Vanna의 선택**: RAG (DB 스키마는 자주 변경됨)

---

### 3. Dynamic Function Binding의 트레이드오프

**장점**:
- 코드 간결 (11개 DB → 11개 함수만)
- Closure로 connection 캡슐화

**단점**:
- IDE 자동완성 어려움
- 런타임 에러 가능성

**교훈**: 코드 간결성 vs 타입 안정성 트레이드오프

**개선 방법**:
```python
# Type hint로 명시
def connect_to_postgres(self, ...):
    def run_sql_postgres(sql: str) -> pd.DataFrame: ...

    self.run_sql: Callable[[str], pd.DataFrame] = run_sql_postgres
    self.run_sql_is_set = True
```

---

### 4. Token Budget의 현실적 제약

**교훈**: LLM은 무한한 컨텍스트를 가지지 않습니다. 우선순위가 필요합니다.

**Vanna의 우선순위**:
1. System prompt (항상)
2. Few-shot examples (가장 중요)
3. DDL (가능한 만큼)
4. Documentation (여유 있으면)

**다른 프로젝트에 적용**:
- 문서 검색: 가장 관련 높은 청크만
- 코드 리뷰: 변경된 파일 우선
- 번역: 핵심 문장 우선

---

### 5. Breaking Change의 올바른 방법

**교훈**: Deprecation은 명확하고 과감하게.

**Vanna의 전략**:
- 즉시 에러 (Warning 아님)
- 명확한 마이그레이션 가이드
- 코드 예시 포함

**효과**:
- 사용자가 즉시 마이그레이션
- 기술 부채 빠르게 제거
- 코드베이스 정리

**다른 프로젝트에 적용**:
- API 버전 변경
- 설정 파일 형식 변경
- 데이터 모델 변경

---

### 6. Preview-Modify-Execute 패턴

**교훈**: 자동화와 사용자 제어의 균형.

**적용 사례**:
- Training Plan (DB 스캔)
- Bulk email (수신자 확인)
- File deletion (삭제 목록 확인)
- Database migration (변경 사항 확인)

**핵심 원칙**:
- 자동화 (스캔/생성)
- 투명성 (preview)
- 제어 (modify)
- 안전성 (execute 전 확인)

---

### 7. Closure로 State 캡슐화

**교훈**: Closure는 Connection pool 등 state 캡슐화에 유용합니다.

**Vanna의 사용**:
```python
def connect_to_postgres(self, host, dbname, user, password):
    def connect_to_db():
        return psycopg2.connect(host=host, dbname=dbname, ...)

    def run_sql(sql):
        conn = connect_to_db()  # Closure!
        ...

    self.run_sql = run_sql
```

**장점**:
- Connection 정보 캡슐화
- 사용자는 connection 관리 불필요
- 함수만 호출하면 됨

**다른 프로젝트에 적용**:
- API client (token 캡슐화)
- File handler (file path 캡슐화)
- Cache (cache key 캡슐화)

---

## 최종 요약

**Vanna는 무엇인가?**
- Text-to-SQL RAG 프레임워크
- Mixin 패턴으로 Vector Store + LLM 조합
- 11개 DB 지원, Plotly 차트 자동 생성

**핵심 설계 결정**:
1. **Mixin 패턴**: Vector Store와 LLM 독립
2. **RAG**: Fine-tuning 불필요, 즉시 업데이트
3. **Dynamic Binding**: 11개 DB 단일 인터페이스
4. **Token Budget**: 14K 제한 내 우선순위 관리
5. **Training Plan**: Preview-Modify-Execute
6. **Breaking Change**: 명확한 마이그레이션 강제

**프로젝트 구조**:
- `base.py` (2118줄) - 핵심 프레임워크
- `chromadb_vector.py` (257줄) - Vector Store
- `openai_chat.py` (128줄) - LLM
- `__init__.py` (399줄) - 엔트리 포인트

**배운 핵심 교훈**:
1. Mixin 패턴의 조합 유연성
2. RAG vs Fine-tuning 선택 기준
3. Token Budget의 현실적 제약
4. Breaking Change의 과감함
5. Preview-Modify-Execute 패턴
6. Closure로 State 캡슐화

**적용 가능한 도메인**:
- Text-to-Code (Python, JS, SQL 등)
- Document QA (RAG)
- API Client Framework (Mixin)
- Database Toolkit (Dynamic Binding)

**한 줄 요약**: Vanna는 Mixin 패턴과 RAG를 활용하여 Vector Store와 LLM을 자유롭게 조합하는 Text-to-SQL 프레임워크입니다.
