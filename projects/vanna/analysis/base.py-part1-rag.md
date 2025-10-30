# base.py (Part 1) - 핵심 RAG 워크플로우

> **파일**: `src/vanna/base/base.py` (line 93-382)
> **역할**: RAG(Retrieval-Augmented Generation) 기반 SQL 생성 핵심 로직
> **주요 메서드**: `generate_sql()`, `extract_sql()`, `generate_followup_questions()`

---

## 📋 파일 개요

### 기본 정보
- **경로**: `src/vanna/base/base.py`
- **분석 범위**: line 93-382 (약 290 lines)
- **주요 클래스**: `VannaBase` (일부)
- **핵심 역할**: 자연어 질문을 SQL로 변환하는 RAG 워크플로우 구현

### 핵심 역할 (한 문장)
**"사용자 질문 + 벡터 DB에서 가져온 컨텍스트를 조합해 LLM에게 SQL을 생성시키고, 응답에서 SQL을 추출하는 RAG 파이프라인"**

### 누가 사용하는가?
- **외부 사용자**: `vn.generate_sql("top 10 customers")`로 SQL 생성
- **내부 메서드**: `ask()` 메서드가 이 워크플로우를 실행
- **확장 개발자**: 다른 RAG 패턴을 구현할 때 참고

---

## 🔍 해결하는 핵심 문제들

### 문제 1: RAG 기반 SQL 생성 - 컨텍스트 조합

**문제**
- LLM만으로는 데이터베이스 스키마를 모름
- 사용자 질문만 던지면 엉뚱한 SQL 생성
- 어떻게 "관련 있는" 정보만 LLM에게 제공할까?

**문제가 없었다면?**
- 매번 전체 스키마를 프롬프트에 넣어야 함 → 토큰 낭비
- 또는 사용자가 직접 DDL을 프롬프트에 포함해야 함 → UX 나쁨

**고민했던 선택지**

**선택지 1: Fine-tuning**
```python
# LLM을 데이터베이스 스키마로 fine-tune
model = finetune(base_model, schema_data)
sql = model.generate(question)
```
- ✅ 장점: 프롬프트에 스키마 불필요
- ❌ 단점:
  - 스키마 변경 시 재훈련 필요
  - 비용 높음 (수천~수만 달러)
  - 시간 오래 걸림
- 왜 안 됨: 스키마가 자주 바뀌는 환경에서 비현실적

**선택지 2: 전체 스키마를 프롬프트에 포함**
```python
def generate_sql(question):
    all_ddl = get_all_database_schema()  # 전체!
    prompt = f"{all_ddl}\n\nQuestion: {question}"
    return llm.generate(prompt)
```
- ✅ 장점: 간단함
- ❌ 단점:
  - 큰 DB는 수만 줄 DDL → 토큰 한계 초과
  - 관련 없는 테이블도 포함 → LLM 혼란
  - 비용 증가 (토큰 많이 사용)
- 왜 안 됨: 실무 DB는 보통 100+ 테이블

**선택지 3 (최종): RAG - 관련 컨텍스트만 검색**
```python
def generate_sql(question):
    # 1. Vector DB에서 관련 정보만 검색
    similar_sqls = get_similar_question_sql(question)
    related_ddl = get_related_ddl(question)
    related_docs = get_related_documentation(question)

    # 2. 검색된 정보 + 질문으로 프롬프트 조립
    prompt = build_prompt(question, similar_sqls, related_ddl, related_docs)

    # 3. LLM 실행
    return llm.generate(prompt)
```
- ✅ 장점:
  - 토큰 효율적 (필요한 정보만)
  - 스키마 변경 시 재훈련 불필요 (벡터만 추가)
  - 비용 저렴
- ⚠️ 단점:
  - 검색 품질에 의존 (관련 정보를 못 찾으면 실패)
  - 구현 복잡도 증가
- 왜 선택: 실무 환경에 가장 적합 (유연성 + 비용 효율)

**최종 해결책**

```python
def generate_sql(self, question: str, allow_llm_to_see_data=False, **kwargs) -> str:
    """RAG 워크플로우"""
    # 1️⃣ Retrieval: 벡터 DB에서 관련 컨텍스트 검색
    question_sql_list = self.get_similar_question_sql(question, **kwargs)
    ddl_list = self.get_related_ddl(question, **kwargs)
    doc_list = self.get_related_documentation(question, **kwargs)

    # 2️⃣ Augmentation: 프롬프트 조립
    prompt = self.get_sql_prompt(
        initial_prompt=initial_prompt,
        question=question,
        question_sql_list=question_sql_list,
        ddl_list=ddl_list,
        doc_list=doc_list,
        **kwargs,
    )

    # 3️⃣ Generation: LLM 실행
    llm_response = self.submit_prompt(prompt, **kwargs)

    # 4️⃣ (선택) Intermediate SQL 실행
    if 'intermediate_sql' in llm_response:
        # LLM이 데이터 확인 필요하다고 판단 → 중간 SQL 실행
        intermediate_sql = self.extract_sql(llm_response)
        df = self.run_sql(intermediate_sql)
        # 결과를 컨텍스트에 추가하고 재시도
        prompt = self.get_sql_prompt(..., doc_list + [df.to_markdown()])
        llm_response = self.submit_prompt(prompt)

    return self.extract_sql(llm_response)
```

**핵심 아이디어**
1. **Semantic Search**: 질문과 유사한 과거 SQL, DDL, 문서를 벡터 검색
2. **Dynamic Prompting**: 필요한 정보만 동적으로 프롬프트에 추가
3. **Two-step Generation**: 필요 시 중간 SQL로 데이터 확인 후 최종 SQL 생성

**트레이드오프**
- 얻은 것: 유연성, 토큰 효율, 비용 절감
- 희생한 것: 벡터 DB 의존성, 검색 품질이 결과에 영향

---

### 문제 2: LLM 응답에서 SQL 추출 - 형식 다양성

**문제**
- LLM은 SQL을 다양한 형식으로 반환
  - Markdown 코드 블록: ` ```sql\nSELECT...\n``` `
  - 설명 + SQL: `Here's the query:\nSELECT...`
  - 순수 SQL: `SELECT...`
- 어떻게 일관되게 SQL만 추출할까?

**문제가 없었다면?**
- LLM 응답 전체를 SQL로 실행 → 에러
- 사용자가 수동으로 복사/붙여넣기 → UX 나쁨

**고민했던 선택지**

**선택지 1: LLM에게 특정 형식 강제**
```python
prompt = "Return ONLY SQL, no explanation. Format: SELECT..."
```
- ✅ 장점: 추출 로직 불필요
- ❌ 단점:
  - LLM이 항상 지시를 따르지는 않음 (특히 GPT-3.5)
  - 다른 LLM 사용 시 형식 달라짐
- 왜 안 됨: LLM은 비결정적 (non-deterministic)

**선택지 2: JSON 형식 강제**
```python
prompt = "Return JSON: {\"sql\": \"SELECT...\"}"
response = json.loads(llm_response)
sql = response["sql"]
```
- ✅ 장점: 파싱 안정적
- ❌ 단점:
  - LLM이 JSON 형식 안 지킬 수도
  - JSON escape 문제 (SQL 안의 따옴표)
  - 모든 LLM이 JSON 잘 생성하는 건 아님
- 왜 선택 안 함: 오히려 더 불안정할 수 있음

**선택지 3 (최종): 여러 패턴으로 Regex 추출**
```python
def extract_sql(self, llm_response: str) -> str:
    # 1순위: CREATE TABLE ... AS SELECT
    sqls = re.findall(r"\bCREATE\s+TABLE\b.*?\bAS\b.*?;", llm_response, re.DOTALL | re.IGNORECASE)
    if sqls:
        return sqls[-1]

    # 2순위: WITH (CTE)
    sqls = re.findall(r"\bWITH\b .*?;", llm_response, re.DOTALL | re.IGNORECASE)
    if sqls:
        return sqls[-1]

    # 3순위: SELECT ... ;
    sqls = re.findall(r"\bSELECT\b .*?;", llm_response, re.DOTALL | re.IGNORECASE)
    if sqls:
        return sqls[-1]

    # 4순위: ```sql ... ```
    sqls = re.findall(r"```sql\s*\n(.*?)```", llm_response, re.DOTALL | re.IGNORECASE)
    if sqls:
        return sqls[-1].strip()

    # 5순위: ``` ... ``` (언어 지정 없음)
    sqls = re.findall(r"```(.*?)```", llm_response, re.DOTALL | re.IGNORECASE)
    if sqls:
        return sqls[-1].strip()

    # 실패: 응답 전체 반환
    return llm_response
```
- ✅ 장점:
  - 대부분의 LLM 응답 형식 커버
  - 우선순위로 정확도 향상
  - Fallback으로 안정성 확보
- ⚠️ 단점:
  - Regex 패턴 유지보수 필요
  - 새로운 LLM 형식 등장 시 추가 필요
- 왜 선택: 현실적으로 가장 안정적

**핵심 아이디어**
1. **우선순위 기반 매칭**: 정확한 패턴부터 시도
2. **Fallback 전략**: 모든 패턴 실패 시 전체 반환 (사용자가 판단)
3. **마지막 매치 선택**: `sqls[-1]` → LLM이 수정한 최종 버전

**트레이드오프**
- 얻은 것: 다양한 LLM 응답 처리, 안정성
- 희생한 것: 완벽한 추출 보장 못 함 (regex 한계)

---

### 문제 3: Intermediate SQL - 데이터 Introspection

**문제**
- 사용자 질문: "Show me customers named John"
- LLM이 모르는 것: `customers` 테이블의 `name` 컬럼에 정확히 어떤 값이 있는지
- DDL만 있으면 `name VARCHAR(100)`는 알지만, 실제 데이터는 몰라서 WHERE 조건을 틀리게 생성
  - 예: `WHERE name = 'John'` (실제 데이터: `'john smith'`, `'JOHN DOE'`)

**문제가 없었다면?**
- LLM이 추측으로 SQL 생성 → 결과 0건
- 사용자가 데이터 확인 후 다시 질문 → 비효율

**고민했던 선택지**

**선택지 1: 항상 샘플 데이터를 프롬프트에 포함**
```python
ddl = "CREATE TABLE customers ..."
sample_data = "Sample rows: ['john smith', 'jane doe', ...]"
prompt = f"{ddl}\n{sample_data}\n\nQuestion: {question}"
```
- ✅ 장점: LLM이 실제 데이터 형식 파악
- ❌ 단점:
  - 모든 테이블 샘플 포함 → 토큰 폭발
  - 민감한 데이터 LLM에 노출 위험
  - 항상 필요한 건 아님 (간단한 질문은 불필요)
- 왜 안 됨: 비용 + 보안

**선택지 2: 사용자에게 에러 메시지 반환**
```python
if result.empty:
    return "No results. Please refine your question."
```
- ✅ 장점: 간단
- ❌ 단점: 사용자가 여러 번 시도해야 함
- 왜 안 됨: UX 나쁨

**선택지 3 (최종): LLM이 필요 시 중간 SQL 요청**
```python
# LLM에게 가이드 제공
prompt += """
2. If you need to know specific values in a column,
   generate an intermediate SQL query to find distinct values.
   Prepend with comment: intermediate_sql
"""

# LLM 응답 예시:
# "-- intermediate_sql\nSELECT DISTINCT name FROM customers WHERE name LIKE '%john%'"

if 'intermediate_sql' in llm_response:
    if not allow_llm_to_see_data:
        return "LLM needs to see data. Set allow_llm_to_see_data=True"

    # 중간 SQL 실행
    intermediate_sql = self.extract_sql(llm_response)
    df = self.run_sql(intermediate_sql)

    # 결과를 컨텍스트에 추가하고 재생성
    doc_list.append(f"Results of {intermediate_sql}:\n{df.to_markdown()}")
    prompt = self.get_sql_prompt(..., doc_list=doc_list)
    llm_response = self.submit_prompt(prompt)
```
- ✅ 장점:
  - 필요할 때만 실행 (토큰 효율)
  - LLM이 판단 (자동화)
  - 사용자 컨트롤 (`allow_llm_to_see_data` 플래그)
- ⚠️ 단점:
  - LLM 호출 2번 (비용 증가)
  - 중간 SQL이 틀리면 최종 SQL도 틀림
- 왜 선택: 정확도 >> 비용 (RAG 핵심)

**핵심 아이디어**
1. **LLM Self-awareness**: LLM이 정보 부족을 인지
2. **Two-step Reasoning**: 데이터 확인 → 최종 SQL 생성
3. **보안 제어**: `allow_llm_to_see_data` 플래그로 사용자 제어

**트레이드오프**
- 얻은 것: 정확도 향상, 자동화
- 희생한 것: LLM 호출 2배, 응답 시간 증가

---

### 문제 4: 후속 질문 생성 - 탐색적 분석 지원

**문제**
- 사용자가 "Top 10 customers" 질문 후 막막함
- 다음에 무엇을 물어볼 수 있을까?
- BI 도구처럼 "드릴다운" 기능 제공 가능할까?

**문제가 없었다면?**
- 사용자가 직접 생각 → 시간 소요
- 분석 깊이 제한 (다음 질문을 못 떠올림)

**고민했던 선택지**

**선택지 1: 고정된 템플릿 질문**
```python
templates = [
    "Show me details of {entity}",
    "What is the trend over time?",
    "Group by {column}"
]
```
- ✅ 장점: 빠름, 비용 없음
- ❌ 단점: 맥락 무시, 범용적이지 않음
- 왜 안 됨: 질문/결과에 따라 후속 질문이 달라야 함

**선택지 2 (최종): LLM이 결과를 보고 생성**
```python
def generate_followup_questions(
    self, question: str, sql: str, df: pd.DataFrame, n_questions: int = 5
) -> list:
    message_log = [
        self.system_message(
            f"User asked: '{question}'\n"
            f"SQL: {sql}\n"
            f"Results (first 25 rows):\n{df.head(25).to_markdown()}"
        ),
        self.user_message(
            f"Generate {n_questions} followup questions based on this data.\n"
            "Requirements:\n"
            "- Each question should be answerable with SQL\n"
            "- Questions should dig deeper into the data\n"
            "- No explanations, just questions (one per line)"
        ),
    ]

    llm_response = self.submit_prompt(message_log)
    # "1. What are their email addresses?\n2. ..." → 리스트로 변환
    numbers_removed = re.sub(r"^\d+\.\s*", "", llm_response, flags=re.MULTILINE)
    return numbers_removed.split("\n")
```
- ✅ 장점:
  - 맥락 인식 (질문 + SQL + 결과)
  - 데이터 기반 (실제 결과 보고 제안)
  - 유연함 (템플릿 불필요)
- ⚠️ 단점:
  - LLM 호출 비용
  - 항상 좋은 질문 생성하는 건 아님
- 왜 선택: 탐색적 분석 경험 향상

**핵심 아이디어**
1. **Context-aware**: 결과 데이터를 LLM에게 보여줌
2. **Actionable**: 버튼으로 바로 실행 가능한 질문 생성
3. **Limited preview**: `df.head(25)` → 토큰 절약

**트레이드오프**
- 얻은 것: 탐색 경험, 사용자 가이드
- 희생한 것: 추가 LLM 호출 비용

---

## ⭐ 실전 적용 가이드

### 가이드 1: RAG SQL 생성기 구현하기

**상황**: 사내 데이터베이스에 자연어 쿼리 인터페이스 추가

#### Step 1: 요구사항 정의
- [ ] 어떤 LLM 사용? (OpenAI / Anthropic / 로컬 모델)
- [ ] 어떤 Vector DB 사용? (ChromaDB / Pinecone / FAISS)
- [ ] 데이터베이스 종류? (Postgres / MySQL / BigQuery)
- [ ] 보안: LLM에게 실제 데이터 보여줄 수 있는가?

#### Step 2: 기본 구현

```python
from vanna.openai import OpenAI_Chat
from vanna.chromadb import ChromaDB_VectorStore

class MyVannaSQLGenerator(ChromaDB_VectorStore, OpenAI_Chat):
    def __init__(self, config=None):
        ChromaDB_VectorStore.__init__(self, config=config)
        OpenAI_Chat.__init__(self, config=config)

# 초기화
vn = MyVannaSQLGenerator(config={
    'api_key': 'sk-...',
    'model': 'gpt-4',
    'max_tokens': 14000  # 프롬프트 크기 제한
})

# DB 연결
vn.connect_to_postgres(
    host='localhost',
    dbname='mydb',
    user='myuser',
    password='***'
)
```

#### Step 3: 학습 데이터 추가

```python
# 1. DDL 추가 (테이블 스키마)
vn.train(ddl="""
CREATE TABLE customers (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100),
    total_sales DECIMAL(10,2)
)
""")

# 2. 예시 SQL 추가 (과거 쿼리)
vn.train(
    question="Who are the top 10 customers?",
    sql="SELECT name, total_sales FROM customers ORDER BY total_sales DESC LIMIT 10"
)

# 3. 비즈니스 용어 추가
vn.train(documentation="""
'VIP customers' means customers with total_sales > 10000
""")
```

#### Step 4: SQL 생성

```python
# 기본 사용
sql = vn.generate_sql("Show me VIP customers")
print(sql)
# → SELECT name, email FROM customers WHERE total_sales > 10000

# 중간 SQL 허용 (데이터 확인 필요 시)
sql = vn.generate_sql(
    "Show customers named John",
    allow_llm_to_see_data=True  # ⚠️ 보안 고려!
)
```

#### Step 5: 의사결정 체크리스트

| 상황 | 선택 |
|------|------|
| 작은 DB (< 20 테이블) | 모든 DDL 벡터 DB에 추가 |
| 큰 DB (100+ 테이블) | 자주 쓰는 테이블만 우선 추가 |
| 민감한 데이터 | `allow_llm_to_see_data=False` (기본값) |
| 정확도 중요 | Intermediate SQL 허용 |
| 비용 중요 | Intermediate SQL 비활성화 |

---

### 가이드 2: Intermediate SQL 패턴 구현

**상황**: LLM이 데이터 확인 후 정확한 SQL 생성

#### Step 1: Prompt에 가이드 추가

```python
class MyVanna(VannaBase):
    def get_sql_prompt(self, question, ...):
        prompt = super().get_sql_prompt(question, ...)

        # 추가 가이드
        prompt += """
        ===Advanced: Intermediate SQL
        If you need to check actual data values:
        1. Generate a SELECT DISTINCT query first
        2. Add comment: -- intermediate_sql
        3. I will run it and give you results
        4. Then generate the final SQL
        """
        return prompt
```

#### Step 2: 응답 처리

```python
def generate_sql_with_introspection(self, question):
    # 1차 시도
    response = self.submit_prompt(question)

    # Intermediate SQL 체크
    if '-- intermediate_sql' in response or 'intermediate_sql' in response:
        # 중간 SQL 추출
        intermediate = self.extract_sql(response)

        # 실행
        df = self.run_sql(intermediate)

        # 결과를 컨텍스트에 추가
        prompt = f"Previous results:\n{df.to_markdown()}\n\nNow generate final SQL for: {question}"

        # 2차 시도
        response = self.submit_prompt(prompt)

    return self.extract_sql(response)
```

#### Step 3: 사용 예시

```python
# 사용자 질문
question = "Show me customers in New York"

# LLM 1차 응답:
# -- intermediate_sql
# SELECT DISTINCT state FROM customers

# 시스템이 실행 → ['NY', 'New York', 'NEW YORK']

# LLM 2차 응답 (정확함!):
# SELECT * FROM customers WHERE state IN ('NY', 'New York', 'NEW YORK')
```

---

### 가이드 3: 커스텀 SQL 추출 로직

**상황**: 특정 LLM이 독특한 형식으로 SQL 반환

#### Step 1: 기본 패턴 확인

```python
# Vanna의 기본 추출 순서:
# 1. CREATE TABLE ... AS SELECT
# 2. WITH ... (CTE)
# 3. SELECT ... ;
# 4. ```sql ... ```
# 5. ``` ... ```
```

#### Step 2: 커스텀 로직 추가

```python
class MyVanna(VannaBase):
    def extract_sql(self, llm_response: str) -> str:
        # 1. 커스텀 패턴 먼저 시도
        # 예: Claude가 <sql>...</sql> 태그 사용
        import re
        sqls = re.findall(r'<sql>(.*?)</sql>', llm_response, re.DOTALL)
        if sqls:
            return sqls[-1].strip()

        # 2. 기본 로직으로 fallback
        return super().extract_sql(llm_response)
```

#### Step 3: 로깅 추가 (디버깅)

```python
def extract_sql(self, llm_response: str) -> str:
    print(f"[DEBUG] LLM Response:\n{llm_response}")

    sql = super().extract_sql(llm_response)

    print(f"[DEBUG] Extracted SQL:\n{sql}")
    return sql
```

---

## ⭐ 안티패턴과 흔한 실수

### 실수 1: 모든 테이블 DDL을 벡터 DB에 추가

**❌ 나쁜 예:**
```python
# 100개 테이블의 DDL을 모두 추가
for table in all_tables:
    vn.train(ddl=get_ddl(table))

# 결과: 검색 시 너무 많은 테이블이 반환 → LLM 혼란
```

**문제:**
- Vector search가 관련 없는 테이블도 반환
- 프롬프트가 불필요하게 길어짐
- LLM이 잘못된 테이블 선택 가능성 증가

**✅ 좋은 예:**
```python
# 1. 자주 쓰는 핵심 테이블만
core_tables = ['customers', 'orders', 'products']
for table in core_tables:
    vn.train(ddl=get_ddl(table))

# 2. 또는 테이블 간 관계 정보 추가
vn.train(documentation="""
customers.id = orders.customer_id (foreign key)
orders.product_id = products.id (foreign key)
""")

# 3. 필요 시 추가 학습
if user_asks_about_rare_table:
    vn.train(ddl=get_ddl(rare_table))
```

---

### 실수 2: `allow_llm_to_see_data=True`를 기본값으로 사용

**❌ 나쁜 예:**
```python
# 모든 쿼리에서 데이터 노출 허용
sql = vn.generate_sql(question, allow_llm_to_see_data=True)
```

**문제:**
- 민감한 데이터가 LLM (OpenAI 등)에 전송
- GDPR/컴플라이언스 위반 가능
- 불필요한 경우에도 중간 SQL 실행 → 비용 증가

**✅ 좋은 예:**
```python
# 1. 기본값은 False
sql = vn.generate_sql(question)  # allow_llm_to_see_data=False

# 2. 사용자에게 명시적으로 물어보기
if sql_failed or "insufficient context" in result:
    user_approval = input("Allow AI to see sample data? (y/n): ")
    if user_approval == 'y':
        sql = vn.generate_sql(question, allow_llm_to_see_data=True)

# 3. 또는 비민감 데이터만 허용
if table_is_public_data:
    sql = vn.generate_sql(question, allow_llm_to_see_data=True)
```

---

### 실수 3: SQL 추출 없이 LLM 응답 직접 실행

**❌ 나쁜 예:**
```python
llm_response = vn.submit_prompt(prompt)
# "Here's your query: SELECT * FROM users"

df = vn.run_sql(llm_response)  # ❌ 에러!
# SQLSyntaxError: "Here's your query:" is not valid SQL
```

**문제:**
- LLM이 설명과 함께 SQL 반환
- DB가 파싱 못 함

**✅ 좋은 예:**
```python
llm_response = vn.submit_prompt(prompt)
sql = vn.extract_sql(llm_response)  # ✅ SQL만 추출
df = vn.run_sql(sql)
```

---

### 실수 4: 후속 질문에 전체 DataFrame 전달

**❌ 나쁜 예:**
```python
# 100만 행 결과
df = vn.run_sql(sql)

# LLM에게 전체 데이터 전달
followups = vn.generate_followup_questions(
    question, sql, df  # ❌ 토큰 폭발!
)
```

**문제:**
- 100만 행을 Markdown으로 변환 → 수백만 토큰
- API 토큰 제한 초과 또는 비용 폭발

**✅ 좋은 예:**
```python
# Vanna는 이미 head(25)만 사용
# src/vanna/base/base.py:326
def generate_followup_questions(self, question, sql, df, n=5):
    # ...
    f"Results (first 25 rows):\n{df.head(25).to_markdown()}"
    # ...

# 추가 최적화: 샘플링
if len(df) > 1000:
    df_sample = df.sample(25)
else:
    df_sample = df.head(25)

followups = vn.generate_followup_questions(question, sql, df_sample)
```

---

### 실수 5: 에러 발생 시 재시도 없음

**❌ 나쁜 예:**
```python
sql = vn.generate_sql(question)
df = vn.run_sql(sql)  # ❌ SQL 에러 시 중단
```

**문제:**
- LLM이 가끔 문법 오류 생성
- 사용자는 아무것도 못 봄

**✅ 좋은 예:**
```python
def generate_sql_with_retry(question, max_retries=2):
    for attempt in range(max_retries):
        sql = vn.generate_sql(question)

        try:
            df = vn.run_sql(sql)
            return sql, df
        except Exception as e:
            if attempt < max_retries - 1:
                # 에러를 LLM에게 알려주고 재생성
                question = f"Previous SQL failed: {e}\nPlease fix and regenerate: {question}"
            else:
                # 최종 실패
                return sql, f"Error: {e}"

sql, result = generate_sql_with_retry("Show top customers")
```

---

### 실수 6: 벡터 검색 결과를 검증 없이 사용

**❌ 나쁜 예:**
```python
def generate_sql(self, question):
    similar_sqls = self.get_similar_question_sql(question)
    # 검색 결과가 실제로 관련 있는지 확인 안 함
    prompt = build_prompt(question, similar_sqls)
    return self.submit_prompt(prompt)
```

**문제:**
- Vector search가 항상 완벽하지 않음
- 관련 없는 SQL이 프롬프트에 포함 → LLM 혼란

**✅ 좋은 예:**
```python
def generate_sql(self, question):
    similar_sqls = self.get_similar_question_sql(question, n=10)

    # 1. 유사도 임계값 필터링
    filtered = [s for s in similar_sqls if s.get('similarity', 0) > 0.7]

    # 2. 또는 LLM에게 관련성 판단 요청
    if len(filtered) > 5:
        prompt = f"Which of these SQLs are relevant to '{question}'?\n{filtered}"
        relevant_ids = self.submit_prompt(prompt)
        filtered = [s for s in filtered if s['id'] in relevant_ids]

    prompt = build_prompt(question, filtered[:3])  # 상위 3개만
    return self.submit_prompt(prompt)
```

---

### 실수 7: 중간 SQL이 무한 루프

**❌ 나쁜 예:**
```python
def generate_sql(self, question):
    response = self.submit_prompt(question)

    while 'intermediate_sql' in response:  # ❌ 무한 루프 가능!
        intermediate = self.extract_sql(response)
        df = self.run_sql(intermediate)
        response = self.submit_prompt(f"Results: {df}\nGenerate final SQL")
```

**문제:**
- LLM이 계속 intermediate_sql 생성 가능
- 비용 폭발 + 시간 초과

**✅ 좋은 예:**
```python
def generate_sql(self, question, max_intermediate_steps=1):
    response = self.submit_prompt(question)

    steps = 0
    while 'intermediate_sql' in response and steps < max_intermediate_steps:
        intermediate = self.extract_sql(response)
        df = self.run_sql(intermediate)
        response = self.submit_prompt(f"Results: {df}\nNow generate FINAL SQL (no more intermediate)")
        steps += 1

    if 'intermediate_sql' in response:
        # 여전히 intermediate → 경고
        return "Error: LLM couldn't generate final SQL"

    return self.extract_sql(response)
```

---

## ⭐ 스케일 고려사항

### 소규모 (< 20 테이블, < 100 쿼리/일)

**권장 사항:**
- ✅ 로컬 Vector DB (ChromaDB/FAISS)
- ✅ 모든 테이블 DDL 추가해도 OK
- ⚠️ Intermediate SQL 비활성화 (비용 절감)

**구현 예시:**
```python
from vanna.chromadb import ChromaDB_VectorStore
from vanna.openai import OpenAI_Chat

class SmallVanna(ChromaDB_VectorStore, OpenAI_Chat):
    def __init__(self):
        ChromaDB_VectorStore.__init__(self, config={'path': './chromadb'})
        OpenAI_Chat.__init__(self, config={'model': 'gpt-3.5-turbo'})  # 저렴한 모델

vn = SmallVanna()

# 모든 테이블 추가 (간단함)
for table in get_all_tables():
    vn.train(ddl=get_ddl(table))
```

**모니터링:**
```python
# 기본 메트릭만
print(f"Total training data: {len(vn.get_training_data())}")
print(f"SQL generated: {sql}")
```

---

### 중규모 (50-100 테이블, 1000 쿼리/일)

**권장 사항:**
- ✅ 클라우드 Vector DB (Pinecone/Qdrant)
- ✅ 자주 쓰는 테이블만 우선 추가
- ✅ 캐싱 도입
- ✅ Intermediate SQL 선택적 허용

**구현 예시:**
```python
from vanna.pinecone import Pinecone_VectorStore
from vanna.openai import OpenAI_Chat
from functools import lru_cache

class MediumVanna(Pinecone_VectorStore, OpenAI_Chat):
    def __init__(self):
        Pinecone_VectorStore.__init__(self, config={
            'api_key': '...',
            'index_name': 'vanna-prod'
        })
        OpenAI_Chat.__init__(self, config={'model': 'gpt-4'})

    @lru_cache(maxsize=100)
    def generate_sql(self, question: str) -> str:
        # 같은 질문은 캐시
        return super().generate_sql(question)

vn = MediumVanna()

# 테이블 우선순위
priority_tables = ['customers', 'orders', 'products']
for table in priority_tables:
    vn.train(ddl=get_ddl(table))

# 나머지는 lazy loading
def on_table_needed(table_name):
    vn.train(ddl=get_ddl(table_name))
```

**모니터링:**
```python
import time

class MonitoredVanna(MediumVanna):
    def generate_sql(self, question):
        start = time.time()
        sql = super().generate_sql(question)
        duration = time.time() - start

        # 메트릭 기록
        log_metric('sql_generation_time', duration)
        log_metric('prompt_tokens', self.str_to_approx_token_count(sql))

        return sql
```

---

### 대규모 (1000+ 테이블, 10000+ 쿼리/일)

**권장 사항:**
- ✅ 분산 Vector DB (Milvus/Weaviate)
- ✅ 테이블 네임스페이스 분리 (부서별, 프로젝트별)
- ✅ 멀티 레벨 캐싱 (Redis + 로컬)
- ✅ Prompt 최적화 (토큰 제한 줄이기)
- ✅ A/B 테스트 (다양한 Prompt 전략)

**구현 예시:**
```python
from vanna.milvus import Milvus_VectorStore
from vanna.openai import OpenAI_Chat
import redis

class LargeVanna(Milvus_VectorStore, OpenAI_Chat):
    def __init__(self, namespace='default'):
        self.namespace = namespace
        self.redis = redis.Redis()

        Milvus_VectorStore.__init__(self, config={
            'host': 'milvus-cluster',
            'collection': f'vanna_{namespace}'  # 네임스페이스별 분리
        })
        OpenAI_Chat.__init__(self, config={
            'model': 'gpt-4-turbo',
            'max_tokens': 8000  # 토큰 제한 줄임
        })

    def generate_sql(self, question: str) -> str:
        # 1. Redis 캐시 확인
        cache_key = f"sql:{self.namespace}:{hash(question)}"
        cached = self.redis.get(cache_key)
        if cached:
            return cached.decode()

        # 2. 생성
        sql = super().generate_sql(question)

        # 3. 캐시 저장 (1시간)
        self.redis.setex(cache_key, 3600, sql)

        return sql

    def get_related_ddl(self, question: str, **kwargs):
        # Vector search 결과 제한 (토큰 절약)
        results = super().get_related_ddl(question, **kwargs)
        return results[:2]  # 최대 2개 테이블만

# 네임스페이스별 인스턴스
vn_sales = LargeVanna(namespace='sales')
vn_marketing = LargeVanna(namespace='marketing')
```

**모니터링 (Prometheus):**
```python
from prometheus_client import Counter, Histogram

sql_generation_counter = Counter('vanna_sql_generated_total', 'Total SQL generated')
sql_generation_duration = Histogram('vanna_sql_generation_seconds', 'SQL generation time')
sql_error_counter = Counter('vanna_sql_errors_total', 'SQL generation errors')

class MonitoredLargeVanna(LargeVanna):
    def generate_sql(self, question):
        with sql_generation_duration.time():
            try:
                sql = super().generate_sql(question)
                sql_generation_counter.inc()
                return sql
            except Exception as e:
                sql_error_counter.inc()
                raise
```

**알림 설정:**
```yaml
# Prometheus alert rules
- alert: HighSQLErrorRate
  expr: rate(vanna_sql_errors_total[5m]) > 0.1
  annotations:
    summary: "SQL generation error rate > 10%"

- alert: SlowSQLGeneration
  expr: histogram_quantile(0.95, vanna_sql_generation_seconds) > 10
  annotations:
    summary: "P95 SQL generation time > 10s"
```

---

## 💡 배운 점

### 1. RAG는 Fine-tuning의 실용적 대안
**핵심 개념**: LLM + Vector DB로 도메인 지식 주입
**언제 사용?**: 지식이 자주 변경되거나, Fine-tuning 비용이 부담될 때
**적용 가능한 곳**:
- 사내 문서 검색 + Q&A 시스템
- 코드베이스 기반 코드 생성기
- 제품 카탈로그 기반 추천 시스템

### 2. Two-step Generation으로 정확도 향상
**핵심 개념**: LLM이 정보 부족 시 데이터 확인 후 재생성
**언제 사용?**: 정확도가 중요하고, 추가 비용 감당 가능할 때
**적용 가능한 곳**:
- 금융 리포트 생성 (숫자 정확도 필수)
- 법률 문서 분석 (근거 확인 필요)

### 3. Prompt Engineering은 반복적 개선
**핵심 개념**: 처음부터 완벽한 Prompt는 없음, 실험과 개선
**언제 사용?**: 항상!
**적용 가능한 곳**:
- A/B 테스트로 여러 Prompt 전략 비교
- 사용자 피드백 기반 Prompt 개선

### 4. Regex는 LLM 출력 파싱의 현실적 방법
**핵심 개념**: LLM 출력은 비결정적 → 여러 패턴으로 대응
**언제 사용?**: JSON 스키마 강제가 어려울 때
**적용 가능한 곳**:
- 코드 생성기 (여러 언어/형식)
- 구조화된 데이터 추출

### 5. 토큰 관리는 비용과 성능의 핵심
**핵심 개념**: 프롬프트 크기 = 비용 + 응답 시간
**언제 사용?**: 대규모 서비스, 예산 제한
**적용 가능한 곳**:
- Max token 제한 (14K)
- 샘플링 (df.head(25))
- 캐싱 (같은 질문 반복)

### 6. 보안과 UX는 Trade-off
**핵심 개념**: `allow_llm_to_see_data` - 정확도 vs 보안
**언제 사용?**: 사용자에게 선택권 부여
**적용 가능한 곳**:
- 민감도에 따라 다른 모드 제공
- 명시적 동의 받기

### 7. Vector Search 품질이 RAG 성공을 좌우
**핵심 개념**: 관련 없는 컨텍스트 → LLM 혼란
**언제 사용?**: RAG 구현 시 검색 품질 검증 필수
**적용 가능한 곳**:
- Embedding 모델 선택 중요
- 유사도 임계값 설정
- 검색 결과 reranking

### 8. Context-aware 후속 질문은 탐색 경험 향상
**핵심 개념**: 결과 데이터 기반 다음 액션 제안
**언제 사용?**: 탐색적 분석, BI 도구
**적용 가능한 곳**:
- 대시보드의 "추천 질문" 기능
- 챗봇의 "이렇게 물어보세요" 버튼

---

## 📊 요약

| 항목 | 내용 |
|------|------|
| **핵심 문제** | LLM이 DB 스키마를 모름 → RAG로 해결 |
| **핵심 패턴** | Retrieval (벡터 검색) + Augmentation (프롬프트 조립) + Generation (LLM) |
| **주요 트레이드오프** | 유연성 & 비용 효율 vs 벡터 검색 품질 의존 |
| **핵심 기법** | 1) Vector search로 관련 컨텍스트만 검색<br>2) Regex 다중 패턴으로 SQL 추출<br>3) Intermediate SQL로 정확도 향상<br>4) 토큰 관리로 비용 최적화 |
| **적용 시 주의** | 1) 보안: `allow_llm_to_see_data` 신중히<br>2) 벡터 검색 품질 검증<br>3) 토큰 제한 고려<br>4) 에러 처리 & 재시도 |
| **스케일 전략** | 소규모: 로컬 Vector DB, 모든 테이블<br>중규모: 클라우드 Vector DB, 캐싱<br>대규모: 분산 Vector DB, 네임스페이스, 멀티레벨 캐싱 |
| **실무 적용** | 사내 DB용 자연어 쿼리 인터페이스, BI 도구 대안 |

---

## 🔗 다음 단계

- **Part 2**: 추상 메서드 & Prompt 생성 로직 분석
- **Part 3**: 데이터베이스 연결 패턴 분석
- **Part 4**: 고수준 API (`ask()`, `train()`) 분석

---

**✅ Part 1 분석 완료!**
