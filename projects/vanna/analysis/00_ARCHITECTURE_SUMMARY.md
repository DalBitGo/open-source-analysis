# Vanna 전체 아키텍처 요약

## 📚 프로젝트 개요
- **프로젝트**: Vanna
- **저장소**: https://github.com/vanna-ai/vanna
- **목적**: **자연어를 SQL로 변환하는 RAG 기반 Text-to-SQL 프레임워크**
- **특징**: Vector Store와 LLM을 자유롭게 조합 가능한 플러그인 아키텍처

---

## 🎯 핵심 문제와 해결

### 문제 1: Text-to-SQL의 근본적 한계
**Before**: Fine-tuning으로 SQL 생성
- ❌ 데이터베이스 스키마 변경 시 재학습 필요
- ❌ 학습 비용 높음 (시간 + 비용)
- ❌ 새로운 테이블 추가 시 즉시 반영 불가능

**After**: RAG (Retrieval-Augmented Generation)
- ✅ 스키마 변경 즉시 반영 (Vector Store에 추가만)
- ✅ Fine-tuning 불필요
- ✅ Few-shot learning (과거 SQL 예시 활용)

**핵심 아이디어**:
```
질문: "지난달 매출 상위 10개 상품은?"
  ↓
1. Vector Search: 유사한 과거 질문 + SQL 검색
2. Retrieval: 관련 테이블 스키마 (DDL) 검색
3. Augmentation: LLM Prompt 구성 (Few-shot + DDL)
4. Generation: LLM이 SQL 생성
  ↓
SELECT * FROM products ORDER BY revenue DESC LIMIT 10
```

---

### 문제 2: Vector Store와 LLM 종속성
**Before**: 특정 Vector Store + 특정 LLM에 강결합
```python
class VannaOpenAI:
    def __init__(self):
        self.chroma = ChromaDB()
        self.llm = OpenAI()
```
- ❌ 다른 Vector Store 사용 불가능
- ❌ 다른 LLM 사용 불가능
- ❌ 새 구현 추가 시 전체 코드 수정

**After**: Mixin 패턴
```python
# Base class with abstract methods
class VannaBase(ABC):
    @abstractmethod
    def add_question_sql(self, question, sql): pass  # Vector Store

    @abstractmethod
    def submit_prompt(self, prompt): pass  # LLM

# 사용자가 조합
class MyVanna(ChromaDB_VectorStore, OpenAI_Chat):
    pass

# 또는
class MyVanna(Pinecone_VectorStore, Anthropic_Chat):
    pass
```
- ✅ Vector Store와 LLM 독립적
- ✅ 새 구현 추가 시 base.py 수정 불필요
- ✅ 사용자가 원하는 조합 선택

---

### 문제 3: 데이터베이스 독립성
**Before**: DB마다 별도 클래스
```python
class PostgresVanna: ...
class MySQLVanna: ...
class SnowflakeVanna: ...
# 11개 DB → 11개 클래스 → 코드 중복
```

**After**: Dynamic Function Binding (Closure 패턴)
```python
def connect_to_postgres(self, host, dbname, user, password):
    def connect_to_db():
        return psycopg2.connect(host=host, dbname=dbname, ...)

    def run_sql_postgres(sql: str) -> pd.DataFrame:
        conn = connect_to_db()  # Closure!
        cs = conn.cursor()
        cs.execute(sql)
        return pd.DataFrame(...)

    # 런타임에 함수 바인딩
    self.run_sql = run_sql_postgres
```
- ✅ 11개 DB 지원 (PostgreSQL, MySQL, Snowflake, BigQuery 등)
- ✅ Connection 정보 Closure로 캡슐화
- ✅ 코드 중복 최소화

---

## 🏛️ 전체 계층 구조

```
┌──────────────────────────────────────────────────────────┐
│  User Application                                         │
│  vn.ask("지난달 매출 상위 10개 상품은?")                  │
└─────────────────────┬────────────────────────────────────┘
                      │
┌─────────────────────┴────────────────────────────────────┐
│  High-Level API (base.py Part 4)                          │
│  - ask(): SQL 생성 → 실행 → 학습 → 시각화               │
│  - train(): Vector Store에 DDL/SQL 추가                  │
│  - generate_plotly_code(): 차트 자동 생성               │
└─────────────────────┬────────────────────────────────────┘
                      │
┌─────────────────────┴────────────────────────────────────┐
│  RAG Pipeline (base.py Part 1)                            │
│  1. Retrieval: get_similar_question_sql(question)         │
│                get_related_ddl(question)                   │
│  2. Augmentation: get_sql_prompt(question, examples, ddl) │
│  3. Generation: submit_prompt(prompt) → SQL               │
└────────┬──────────────────────┬────────────────────────────┘
         │                      │
         ▼                      ▼
┌──────────────────┐  ┌──────────────────┐
│ Vector Store     │  │ LLM              │
│ (Mixin)          │  │ (Mixin)          │
│                  │  │                  │
│ ChromaDB:        │  │ OpenAI:          │
│ - add_ddl()      │  │ - submit_prompt()│
│ - add_question_  │  │ - system_message │
│   sql()          │  │ - user_message   │
│ - get_similar_   │  │                  │
│   question_sql() │  │                  │
└────────┬─────────┘  └─────────┬────────┘
         │                      │
         ▼                      ▼
┌──────────────────┐  ┌──────────────────┐
│ ChromaDB         │  │ OpenAI API       │
│ (3 Collections)  │  │ (gpt-3.5/4)      │
│ - sql            │  │                  │
│ - ddl            │  │                  │
│ - documentation  │  │                  │
└──────────────────┘  └──────────────────┘

┌─────────────────────────────────────────────────────────┐
│  Database Abstraction (base.py Part 3)                   │
│  - connect_to_postgres() → self.run_sql 바인딩          │
│  - connect_to_snowflake() → self.run_sql 바인딩         │
│  - ... (11개 DB)                                        │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
                ┌────────────────┐
                │ Database       │
                │ (PostgreSQL,   │
                │  Snowflake,    │
                │  BigQuery...)  │
                └────────────────┘
```

---

## 📂 주요 모듈 상세

### 🔹 **RAG Pipeline** (base.py Part 1)

**핵심 메서드**: `generate_sql()`

**흐름**:
```python
def generate_sql(self, question: str):
    # 1. Retrieval (Vector Search)
    question_sql_list = self.get_similar_question_sql(question)  # 과거 유사 SQL
    ddl_list = self.get_related_ddl(question)                    # 관련 테이블 스키마
    doc_list = self.get_related_documentation(question)          # 비즈니스 컨텍스트

    # 2. Augmentation (Prompt 구성)
    prompt = self.get_sql_prompt(
        question=question,
        question_sql_list=question_sql_list,  # Few-shot examples
        ddl_list=ddl_list,                    # 테이블 정의
        doc_list=doc_list                     # 추가 설명
    )

    # 3. Generation (LLM 호출)
    llm_response = self.submit_prompt(prompt)  # Abstract method

    # 4. Extraction
    sql = self.extract_sql(llm_response)
    return sql
```

**Two-Step SQL Generation** (정확도 향상):
```python
# 문제: LLM이 데이터의 실제 값을 모름
# 질문: "John Smith의 주문 내역"
# DB: 실제로는 "john.smith@email.com"로 저장됨

# Step 1: Intermediate SQL (데이터 확인용)
intermediate_sql = "SELECT DISTINCT email FROM users WHERE name LIKE '%John%Smith%'"
df = self.run_sql(intermediate_sql)
# 결과: john.smith@email.com

# Step 2: Final SQL (실제 데이터 활용)
prompt_with_data = self.add_data_to_prompt(prompt, df)
final_sql = "SELECT * FROM orders WHERE user_email = 'john.smith@email.com'"
```

---

### 🔹 **Mixin Pattern** (base.py Part 2)

**문제**: Vector Store와 LLM을 독립적으로 구현하고 싶다

**해결**: Abstract Base Class + Mixin

```python
# base.py
class VannaBase(ABC):
    # Vector Store 인터페이스
    @abstractmethod
    def add_question_sql(self, question: str, sql: str) -> str:
        pass

    @abstractmethod
    def get_similar_question_sql(self, question: str) -> List[Tuple[str, str]]:
        pass

    # LLM 인터페이스
    @abstractmethod
    def submit_prompt(self, prompt: List[dict]) -> str:
        pass

    @abstractmethod
    def system_message(self, message: str) -> dict:
        pass

# chromadb_vector.py
class ChromaDB_VectorStore(VannaBase):
    def add_question_sql(self, question, sql):
        id = deterministic_uuid(json.dumps({"question": question, "sql": sql})) + "-sql"
        self.sql_collection.add(documents=json.dumps(...), ids=id)

    def get_similar_question_sql(self, question):
        results = self.sql_collection.query(query_texts=[question], n_results=10)
        return [(q, s) for q, s in results]

# openai_chat.py
class OpenAI_Chat(VannaBase):
    def submit_prompt(self, prompt):
        response = self.client.chat.completions.create(
            model="gpt-3.5-turbo",
            messages=prompt
        )
        return response.choices[0].message.content

    def system_message(self, message):
        return {"role": "system", "content": message}

# 사용자 코드 (Mixin 조합!)
class MyVanna(ChromaDB_VectorStore, OpenAI_Chat):
    def __init__(self, config=None):
        ChromaDB_VectorStore.__init__(self, config=config)
        OpenAI_Chat.__init__(self, config=config)

vn = MyVanna(config={"api_key": "sk-..."})
```

**장점**:
- ✅ Vector Store와 LLM 완전히 독립
- ✅ 새 구현 추가 시 base.py 수정 불필요
- ✅ 사용자가 원하는 조합 선택 (ChromaDB + OpenAI, Pinecone + Claude 등)

---

### 🔹 **Token Budget 관리** (base.py Part 2)

**문제**: GPT-3.5-turbo 컨텍스트 4K (16K 모델: 16K). 수십 개 테이블 DDL을 모두 넣으면?

**해결**: Dynamic Token Budget
```python
def add_ddl_to_prompt(self, initial_prompt, ddl_list, max_tokens=14000):
    for ddl in ddl_list:
        current = self.str_to_approx_token_count(initial_prompt)
        ddl_tokens = self.str_to_approx_token_count(ddl)

        # Budget 초과 여부 체크
        if current + ddl_tokens < max_tokens:
            initial_prompt += f"{ddl}\n\n"
        else:
            break  # 더 이상 추가 안 함

    return initial_prompt
```

**우선순위**:
1. System prompt (항상 포함)
2. Few-shot examples (가장 중요)
3. DDL (가능한 만큼)
4. Documentation (여유 있으면)

---

### 🔹 **Database Abstraction** (base.py Part 3)

**Dynamic Function Binding**:
```python
def connect_to_postgres(self, host, dbname, user, password, port):
    # Closure로 connection 정보 캡슐화
    def connect_to_db():
        return psycopg2.connect(
            host=host,
            dbname=dbname,
            user=user,
            password=password,
            port=port
        )

    def run_sql_postgres(sql: str) -> pd.DataFrame:
        conn = connect_to_db()  # Closure!
        cs = conn.cursor()
        cs.execute(sql)
        results = cs.fetchall()
        columns = [desc[0] for desc in cs.description]
        return pd.DataFrame(results, columns=columns)

    # 런타임에 함수 바인딩!
    self.run_sql = run_sql_postgres
    self.run_sql_is_set = True
```

**지원 DB** (11개):
- PostgreSQL, MySQL, SQLite, Snowflake, BigQuery
- DuckDB, ClickHouse, Oracle, Hive, Presto, Databricks

---

### 🔹 **High-Level API** (base.py Part 4)

**Pipeline 패턴**: `ask()` 메서드가 6단계 통합
```python
def ask(self, question, print_results=True, auto_train=True, visualize=True):
    # 1. Generate SQL
    sql = self.generate_sql(question)

    # 2. Execute
    df = self.run_sql(sql)

    # 3. Auto-train (성공한 SQL 자동 학습)
    if len(df) > 0 and auto_train:
        self.add_question_sql(question, sql)

    # 4. Generate Plotly code
    if visualize:
        plotly_code = self.generate_plotly_code(
            question=question,
            sql=sql,
            df=df
        )

        # 5. Execute Plotly code (with graceful degradation)
        try:
            fig = self.get_plotly_figure(plotly_code, df)
        except Exception as e:
            fig = None  # 차트 실패해도 데이터는 반환

    # 6. Return all
    return sql, df, fig
```

**Training Plan** (Preview-Modify-Execute 패턴):
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
# Train on DDL: public temp_table

# 3. Modify
plan.remove_item("Train on DDL: public temp_table")

# 4. Execute
vn.train(plan=plan)
```

---

### 🔹 **ChromaDB Vector Store** (chromadb_vector.py)

**3-Collection 분리**:
```python
self.sql_collection = chroma_client.get_or_create_collection("sql")
self.ddl_collection = chroma_client.get_or_create_collection("ddl")
self.documentation_collection = chroma_client.get_or_create_collection("documentation")
```

**이유**:
- SQL: 질문-SQL 쌍 (Few-shot learning)
- DDL: 테이블 스키마 (CREATE TABLE ...)
- Documentation: 비즈니스 컨텍스트 ("revenue는 VAT 제외")

**Deterministic UUID** (자동 중복 제거):
```python
def add_ddl(self, ddl: str) -> str:
    id = deterministic_uuid(ddl) + "-ddl"
    self.ddl_collection.add(documents=ddl, ids=id)
    return id

# 같은 DDL 추가 시 → 같은 UUID → ChromaDB가 자동 업데이트
```

---

### 🔹 **OpenAI LLM** (openai_chat.py)

**자동 모델 선택** (비용 최적화):
```python
def submit_prompt(self, prompt):
    # 토큰 수 근사치
    num_tokens = sum(len(m["content"]) / 4 for m in prompt)

    # 3500 토큰 기준으로 모델 선택
    if num_tokens > 3500:
        model = "gpt-3.5-turbo-16k"  # 비싼 모델
    else:
        model = "gpt-3.5-turbo"      # 저렴한 모델

    response = self.client.chat.completions.create(
        model=model,
        messages=prompt,
        temperature=self.temperature
    )
    return response.choices[0].message.content
```

**비용 비교**:
- gpt-3.5-turbo: $0.001 / 1K tokens
- gpt-3.5-turbo-16k: $0.002 / 1K tokens
- gpt-4: $0.03 / 1K tokens (30배!)

**Azure OpenAI 호환**:
```python
# kwargs 우선순위: kwargs > config > 자동 선택
if kwargs.get("model"):
    model = kwargs["model"]
elif kwargs.get("engine"):  # Azure OpenAI
    engine = kwargs["engine"]
elif self.config.get("model"):
    model = self.config["model"]
else:
    # 자동 선택
```

---

### 🔹 **API 마이그레이션** (__init__.py)

**문제**: v0.x는 global 함수, v1.0은 인스턴스. 기존 사용자 코드 처리?

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

# 모든 레거시 함수 (30개 이상)
def set_api_key(key: str) -> None:
    error_deprecation()

def generate_sql(question: str) -> str:
    error_deprecation()
```

**효과**:
- 사용자가 즉시 마이그레이션
- 기술 부채 빠르게 제거
- 코드베이스 정리

**OTP 인증** (Vanna.AI Cloud):
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

## 🎨 핵심 설계 패턴

### 1. **Mixin 패턴** (구조적 독립성)
**목적**: Vector Store와 LLM을 독립적으로 조합

**구현**:
```python
class VannaBase(ABC):
    @abstractmethod
    def abstract_method(self): pass

class FeatureA(VannaBase):
    def method_a(self): ...

class FeatureB(VannaBase):
    def method_b(self): ...

class Combined(FeatureA, FeatureB):
    pass  # 조합!
```

**적용**:
- ChromaDB + OpenAI
- Pinecone + Anthropic
- Custom Vector Store + Local LLM

---

### 2. **RAG Pipeline** (정확도 향상)
**목적**: Fine-tuning 없이 높은 정확도

**흐름**:
```
질문 → Retrieval (Vector Search) → Augmentation (Prompt) → Generation (LLM) → SQL
```

**핵심**:
- Few-shot learning (과거 SQL 예시)
- 관련 DDL만 포함 (Token budget)
- Two-step generation (데이터 확인 → 정확한 SQL)

---

### 3. **Dynamic Function Binding** (DB 독립성)
**목적**: 11개 DB를 단일 인터페이스로

**구현**:
```python
def connect_to_db():
    conn = psycopg2.connect(...)  # Connection 정보 캡슐화
    return conn

def run_sql(sql):
    conn = connect_to_db()  # Closure!
    ...

self.run_sql = run_sql  # 런타임 바인딩
```

**장점**:
- Connection pool 캡슐화
- DB별 코드 중복 최소화
- 사용자는 connection 관리 불필요

---

### 4. **Token Budget 관리** (비용 최적화)
**목적**: LLM 컨텍스트 길이 제한 대응

**구현**:
```python
for item in items:
    if current_tokens + item_tokens < max_tokens:
        prompt += item
    else:
        break
```

**우선순위**:
1. System prompt (필수)
2. Few-shot examples (정확도에 가장 중요)
3. DDL (관련 테이블만)
4. Documentation (여유 있으면)

---

### 5. **Pipeline 패턴** (사용 편의성)
**목적**: 복잡한 워크플로우를 단일 API로

**구현**:
```python
def ask(question):
    sql = self.generate_sql(question)         # Step 1
    df = self.run_sql(sql)                    # Step 2
    if auto_train: self.add_question_sql()    # Step 3
    if visualize: fig = self.generate_chart() # Step 4
    return sql, df, fig
```

**효과**:
- 6단계 → 1줄
- 사용자는 복잡도 몰라도 됨

---

### 6. **Preview-Modify-Execute** (사용자 제어)
**목적**: 자동화 + 사용자 제어 균형

**구현**:
```python
plan = create_plan()        # 1. Preview
plan.remove_item("...")     # 2. Modify
execute(plan)               # 3. Execute
```

**적용**:
- Training Plan (수백 개 테이블 학습)
- Bulk operations (확인 후 실행)

---

## 📊 성능 특성

### Token Budget 영향
- **Small context** (< 2K tokens): gpt-3.5-turbo
- **Medium context** (2K-4K): gpt-3.5-turbo
- **Large context** (4K-16K): gpt-3.5-turbo-16k (자동 선택)

### 비용 최적화
- 자동 모델 선택: 30-50% 비용 절감
- Token budget 관리: 불필요한 DDL 제외
- 캐싱 (Redis): 20-40% 캐시 히트

### 정확도
- RAG (Few-shot): 기본 정확도 70-80%
- Two-step SQL: 정확도 85-90%로 향상
- Domain별 모델 분리: 90-95%

---

## 🔧 확장 포인트

### 1. Custom Vector Store 구현
```python
class MyVectorStore(VannaBase):
    def add_question_sql(self, question, sql):
        # Pinecone, Qdrant, Weaviate 등
        pass

    def get_similar_question_sql(self, question):
        pass
```

### 2. Custom LLM 구현
```python
class MyLLM(VannaBase):
    def submit_prompt(self, prompt):
        # Anthropic, Cohere, Local LLM 등
        pass
```

### 3. Custom Database 연결
```python
def connect_to_custom_db(self, connection_string):
    def run_sql_custom(sql):
        # 새로운 DB 연결
        pass

    self.run_sql = run_sql_custom
```

---

## 📝 요약

| 측면 | 내용 |
|------|------|
| **핵심 목적** | 자연어 → SQL (RAG 기반) |
| **주요 문제** | Fine-tuning 비용, Vector Store/LLM 종속성, DB 독립성 |
| **해결 방법** | RAG, Mixin 패턴, Dynamic Binding, Token Budget |
| **핵심 패턴** | Mixin, RAG Pipeline, Closure, Pipeline, Preview-Modify-Execute |
| **확장성** | Vector Store, LLM, Database 모두 교체 가능 |
| **성능** | Token budget 관리 + 자동 모델 선택 |
| **비용 최적화** | 30-50% (자동 모델 선택 + 캐싱) |
| **정확도** | 70-95% (RAG + Two-step + Domain 분리) |

**한 줄 요약**: Mixin 패턴과 RAG를 활용하여 Vector Store와 LLM을 자유롭게 조합하는 Text-to-SQL 프레임워크입니다.
