# base.py (Part 2) - 추상 메서드 & Prompt 생성

> **파일**: `src/vanna/base/base.py` (line 383-693)
> **역할**: Vector Store/LLM 인터페이스 정의 및 토큰 관리형 Prompt 조립
> **주요 메서드**: 추상 메서드 9개, `get_sql_prompt()`, `add_*_to_prompt()` 유틸

---

## 📋 파일 개요

### 기본 정보
- **경로**: `src/vanna/base/base.py`
- **분석 범위**: line 383-693 (약 310 lines)
- **주요 클래스**: `VannaBase` (추상 메서드 부분)
- **핵심 역할**: 확장 가능한 아키텍처를 위한 인터페이스 정의 + 토큰 제한 내 프롬프트 조립

### 핵심 역할 (한 문장)
**"Vector DB와 LLM을 교체 가능하게 만드는 추상 인터페이스 + 14K 토큰 제한 내에서 최대한 많은 컨텍스트를 프롬프트에 넣는 조립 로직"**

### 누가 사용하는가?
- **프레임워크 확장자**: ChromaDB, Pinecone, OpenAI, Anthropic 등 구현체 개발
- **내부 메서드**: `generate_sql()`이 이 인터페이스를 통해 Vector/LLM 접근
- **일반 사용자**: 직접 호출 안 함 (내부 사용)

---

## 🔍 해결하는 핵심 문제들

### 문제 1: Vector Store 독립성 - 다양한 구현체 지원

**문제**
- Vanna를 ChromaDB, Pinecone, FAISS, Qdrant, Weaviate 등 다양한 Vector DB와 연동하려면?
- 각 Vector DB마다 API가 다름
- 코어 로직(`generate_sql`)을 Vector DB 변경 때마다 수정할 수 없음

**문제가 없었다면?**
- Vector DB 하나에만 종속 → 사용자 선택 제한
- 또는 코어 로직에 if-else 지옥:
  ```python
  if vector_db == 'chromadb':
      results = chromadb_search(...)
  elif vector_db == 'pinecone':
      results = pinecone_search(...)
  # ... 10개 이상의 분기
  ```

**고민했던 선택지**

**선택지 1: Adapter 패턴 - 별도 Adapter 클래스**
```python
class VectorDBAdapter(ABC):
    @abstractmethod
    def search(self, query): pass

class ChromaDBAdapter(VectorDBAdapter):
    def search(self, query):
        return chromadb_client.query(...)

class VannaBase:
    def __init__(self, adapter: VectorDBAdapter):
        self.adapter = adapter

    def generate_sql(self, question):
        results = self.adapter.search(question)
```
- ✅ 장점: 코어 로직과 분리, 확장 용이
- ❌ 단점:
  - 사용자가 Adapter 객체 생성해야 함
  - 추가 레이어 (복잡도 증가)
- 왜 선택 안 함: Python의 다중 상속이 더 간단

**선택지 2 (최종): Mixin 패턴 - 다중 상속**
```python
class VannaBase(ABC):
    @abstractmethod
    def get_similar_question_sql(self, question): pass

    @abstractmethod
    def add_question_sql(self, question, sql): pass

# 사용자가 직접 조합
from vanna.chromadb import ChromaDB_VectorStore
from vanna.openai import OpenAI_Chat

class MyVanna(ChromaDB_VectorStore, OpenAI_Chat):
    def __init__(self, config):
        ChromaDB_VectorStore.__init__(self, config)
        OpenAI_Chat.__init__(self, config)

vn = MyVanna(config={...})
```
- ✅ 장점:
  - 사용자가 자유롭게 조합 (ChromaDB + OpenAI / Pinecone + Anthropic)
  - 코어 로직은 추상 메서드만 호출 → Vector DB 독립적
  - Pythonic (다중 상속 활용)
- ⚠️ 단점:
  - MRO(Method Resolution Order) 이해 필요
  - `__init__` 충돌 가능성 (각 클래스에서 super() 사용 필요)
- 왜 선택: Python 생태계에 자연스러움, 최소한의 코드

**최종 해결책**

```python
class VannaBase(ABC):
    # ----------------- 추상 메서드: Vector Store 인터페이스 -----------------

    @abstractmethod
    def generate_embedding(self, data: str, **kwargs) -> List[float]:
        """텍스트 → 벡터 변환"""
        pass

    @abstractmethod
    def get_similar_question_sql(self, question: str, **kwargs) -> list:
        """질문과 유사한 과거 SQL 검색"""
        pass

    @abstractmethod
    def get_related_ddl(self, question: str, **kwargs) -> list:
        """질문과 관련된 DDL 검색"""
        pass

    @abstractmethod
    def get_related_documentation(self, question: str, **kwargs) -> list:
        """질문과 관련된 문서 검색"""
        pass

    @abstractmethod
    def add_question_sql(self, question: str, sql: str, **kwargs) -> str:
        """Question-SQL 쌍 추가"""
        pass

    @abstractmethod
    def add_ddl(self, ddl: str, **kwargs) -> str:
        """DDL 추가"""
        pass

    @abstractmethod
    def add_documentation(self, documentation: str, **kwargs) -> str:
        """문서 추가"""
        pass

    @abstractmethod
    def get_training_data(self, **kwargs) -> pd.DataFrame:
        """모든 학습 데이터 조회"""
        pass

    @abstractmethod
    def remove_training_data(self, id: str, **kwargs) -> bool:
        """학습 데이터 삭제"""
        pass

# ChromaDB 구현 예시 (src/vanna/chromadb/chromadb_vector.py)
class ChromaDB_VectorStore(VannaBase):
    def __init__(self, config):
        import chromadb
        self.chroma_client = chromadb.Client()
        self.collection = self.chroma_client.create_collection("vanna")

    def generate_embedding(self, data: str) -> List[float]:
        # ChromaDB의 임베딩 사용
        return self.collection._embedding_function([data])[0]

    def get_similar_question_sql(self, question: str, n=5):
        results = self.collection.query(
            query_texts=[question],
            n_results=n,
            where={"type": "question_sql"}
        )
        return results['documents']

    # ... 나머지 메서드 구현

# Pinecone 구현 예시 (src/vanna/pinecone/pinecone_vector.py)
class Pinecone_VectorStore(VannaBase):
    def __init__(self, config):
        import pinecone
        pinecone.init(api_key=config['api_key'])
        self.index = pinecone.Index("vanna")

    def generate_embedding(self, data: str) -> List[float]:
        # OpenAI 임베딩 사용
        import openai
        response = openai.Embedding.create(input=data, model="text-embedding-ada-002")
        return response['data'][0]['embedding']

    def get_similar_question_sql(self, question: str, n=5):
        embedding = self.generate_embedding(question)
        results = self.index.query(embedding, top_k=n, filter={"type": "question_sql"})
        return [r['metadata']['sql'] for r in results['matches']]

    # ... 나머지 메서드 구현
```

**핵심 아이디어**
1. **Interface Segregation**: Vector Store 관련 메서드만 추상화
2. **Mixin Pattern**: 사용자가 Vector + LLM 조합 선택
3. **Consistent API**: 모든 구현체가 동일한 인터페이스

**트레이드오프**
- 얻은 것: 확장성, 사용자 선택권, 코어 로직 독립성
- 희생한 것: 다중 상속 복잡도, 구현체마다 9개 메서드 구현 필요

---

### 문제 2: LLM 독립성 - 다양한 LLM 지원

**문제**
- OpenAI, Anthropic, Google Gemini, Ollama 등 다양한 LLM 지원
- 각 LLM마다 메시지 형식이 다름:
  - OpenAI: `{"role": "system", "content": "..."}`
  - Anthropic: `{"role": "user", "content": "..."}`
  - Ollama: 평문 문자열

**문제가 없었다면?**
- LLM 하나에만 종속
- 또는 코어 로직에 LLM별 분기

**고민했던 선택지**

**선택지 1: 통일된 메시지 형식 강제**
```python
# 내부 형식 고정
message = {"role": "system", "content": "..."}

# 각 LLM에서 변환
class AnthropicChat:
    def submit_prompt(self, messages):
        # Anthropic 형식으로 변환
        converted = convert_to_anthropic_format(messages)
        return anthropic_api.call(converted)
```
- ✅ 장점: 코어 로직 단순
- ❌ 단점: 변환 로직 복잡, 각 LLM 특수 기능 활용 어려움
- 왜 선택 안 함: LLM마다 프롬프트 최적화 방법이 다름

**선택지 2 (최종): 추상 메서드 - 각 LLM이 자유롭게 구현**
```python
class VannaBase(ABC):
    @abstractmethod
    def system_message(self, message: str) -> any:
        """시스템 메시지 생성 (LLM 형식에 맞게)"""
        pass

    @abstractmethod
    def user_message(self, message: str) -> any:
        """사용자 메시지 생성"""
        pass

    @abstractmethod
    def assistant_message(self, message: str) -> any:
        """어시스턴트 메시지 생성 (few-shot 예시용)"""
        pass

    @abstractmethod
    def submit_prompt(self, prompt, **kwargs) -> str:
        """LLM에게 프롬프트 전송 & 응답 받기"""
        pass

# OpenAI 구현
class OpenAI_Chat(VannaBase):
    def system_message(self, message: str):
        return {"role": "system", "content": message}

    def user_message(self, message: str):
        return {"role": "user", "content": message}

    def assistant_message(self, message: str):
        return {"role": "assistant", "content": message}

    def submit_prompt(self, prompt: list, **kwargs):
        import openai
        response = openai.ChatCompletion.create(
            model="gpt-4",
            messages=prompt,  # [{"role": "...", "content": "..."}, ...]
            **kwargs
        )
        return response['choices'][0]['message']['content']

# Anthropic 구현
class Anthropic_Chat(VannaBase):
    def system_message(self, message: str):
        # Anthropic은 system을 별도로
        return {"type": "system", "content": message}

    def user_message(self, message: str):
        return {"role": "user", "content": message}

    def assistant_message(self, message: str):
        return {"role": "assistant", "content": message}

    def submit_prompt(self, prompt: list, **kwargs):
        import anthropic
        # System과 messages 분리
        system = [m['content'] for m in prompt if m.get('type') == 'system']
        messages = [m for m in prompt if m.get('role')]

        response = anthropic.Anthropic().messages.create(
            model="claude-3-opus",
            system=system[0] if system else "",
            messages=messages,
            **kwargs
        )
        return response.content[0].text
```
- ✅ 장점:
  - 각 LLM의 최적 형식 사용
  - LLM 특수 기능 활용 가능
  - 코어 로직은 메시지 형식 몰라도 됨
- ⚠️ 단점: 4개 메서드 구현 필요
- 왜 선택: 각 LLM의 강점 활용

**핵심 아이디어**
1. **Factory Pattern**: `*_message()` 메서드가 LLM별 메시지 객체 생성
2. **Abstraction**: 코어 로직은 메시지 타입(`any`) 몰라도 됨
3. **Flexibility**: 각 LLM이 최적 형식 선택

**트레이드오프**
- 얻은 것: LLM 독립성, 최적화 가능
- 희생한 것: 구현체마다 4개 메서드 구현

---

### 문제 3: 토큰 제한 관리 - 14K 토큰 내 프롬프트 조립

**문제**
- LLM 컨텍스트 제한 (예: GPT-4 8K, 32K)
- Vanna 기본값: 14K 토큰 (`self.max_tokens = 14000`)
- DDL + Documentation + Question-SQL 모두 넣으면 초과 가능
- 어떻게 우선순위를 두고 제한 내에서 최대한 넣을까?

**문제가 없었다면?**
- 제한 초과 → API 에러
- 또는 앞부분 잘림 → 중요한 정보 누락

**고민했던 선택지**

**선택지 1: 모든 컨텍스트를 요약**
```python
ddl_summary = llm.summarize(all_ddl)
doc_summary = llm.summarize(all_docs)
prompt = f"{ddl_summary}\n{doc_summary}\n{question}"
```
- ✅ 장점: 항상 제한 내
- ❌ 단점:
  - 요약 시 정보 손실
  - 추가 LLM 호출 (비용 + 시간)
  - 요약이 잘못되면 SQL도 틀림
- 왜 안 됨: 정보 손실 > 토큰 절약

**선택지 2: 고정 개수만 포함**
```python
prompt = f"{ddl_list[:3]}\n{doc_list[:5]}\n{question}"
```
- ✅ 장점: 간단
- ❌ 단점: 토큰 낭비 또는 부족 (항목마다 크기 다름)
- 왜 안 됨: 비효율적

**선택지 3 (최종): 동적 토큰 계산 + 우선순위 추가**
```python
def add_ddl_to_prompt(
    self, initial_prompt: str, ddl_list: list[str], max_tokens: int = 14000
) -> str:
    if len(ddl_list) > 0:
        initial_prompt += "\n===Tables \n"

        for ddl in ddl_list:
            # 토큰 계산 (간단한 휴리스틱: 문자 수 / 4)
            current_tokens = self.str_to_approx_token_count(initial_prompt)
            ddl_tokens = self.str_to_approx_token_count(ddl)

            # 제한 내에 들어가면 추가
            if current_tokens + ddl_tokens < max_tokens:
                initial_prompt += f"{ddl}\n\n"
            else:
                # 더 이상 못 넣으면 중단
                break

    return initial_prompt

def str_to_approx_token_count(self, string: str) -> int:
    # 간단한 추정: 영어 4글자 ≈ 1 토큰
    return len(string) / 4
```
- ✅ 장점:
  - 토큰 효율적 (제한 직전까지 채움)
  - 우선순위 (Vector search 상위 결과부터)
  - 동적 (프롬프트 크기에 따라 조절)
- ⚠️ 단점:
  - 토큰 계산 정확하지 않음 (실제: tiktoken 사용해야)
  - 뒤쪽 항목은 잘림 가능성
- 왜 선택: 실용적 균형

**최종 해결책**

```python
def get_sql_prompt(
    self,
    initial_prompt: str,
    question: str,
    question_sql_list: list,
    ddl_list: list,
    doc_list: list,
    **kwargs,
):
    """토큰 제한 내에서 최대한 많은 컨텍스트 포함"""
    if initial_prompt is None:
        initial_prompt = f"You are a {self.dialect} expert. " + \
        "Please help to generate a SQL query to answer the question. "

    # 1. DDL 추가 (가장 중요)
    initial_prompt = self.add_ddl_to_prompt(
        initial_prompt, ddl_list, max_tokens=self.max_tokens
    )

    # 2. Static documentation 추가
    if self.static_documentation != "":
        doc_list.append(self.static_documentation)

    # 3. Documentation 추가
    initial_prompt = self.add_documentation_to_prompt(
        initial_prompt, doc_list, max_tokens=self.max_tokens
    )

    # 4. Response Guidelines 추가 (항상 포함)
    initial_prompt += (
        "===Response Guidelines \n"
        "1. If context is sufficient, generate valid SQL. \n"
        "2. If you need to check actual values, use intermediate_sql. \n"
        "3. If context is insufficient, explain why. \n"
        "4. Use most relevant tables. \n"
        f"6. Ensure SQL is {self.dialect}-compliant. \n"
    )

    # 5. Few-shot 예시 추가 (Question-SQL pairs)
    message_log = [self.system_message(initial_prompt)]
    for example in question_sql_list:
        if example and "question" in example and "sql" in example:
            message_log.append(self.user_message(example["question"]))
            message_log.append(self.assistant_message(example["sql"]))

    # 6. 실제 사용자 질문 추가
    message_log.append(self.user_message(question))

    return message_log
```

**핵심 아이디어**
1. **Progressive Addition**: DDL → Doc → SQL 순으로 추가 (우선순위)
2. **Token Budgeting**: 각 단계마다 남은 토큰 확인
3. **Early Stopping**: 제한 도달 시 중단
4. **Essential Last**: Response Guidelines는 항상 포함

**트레이드오프**
- 얻은 것: 토큰 효율, API 에러 방지
- 희생한 것: 정확한 토큰 계산 안 함 (tiktoken 사용하면 느림)

---

### 문제 4: Few-shot Learning - 예시 SQL로 LLM 가이드

**문제**
- LLM에게 "SQL 생성해줘"만 하면 스타일이 일관되지 않음
  - 어떤 때는 `SELECT *`, 어떤 때는 명시적 컬럼
  - 날짜 형식, JOIN 스타일 등 다름
- 어떻게 "우리 프로젝트 스타일"로 SQL 생성하게 할까?

**문제가 없었다면?**
- 생성된 SQL을 수동으로 수정
- 또는 프롬프트에 상세한 스타일 가이드 (토큰 낭비)

**고민했던 선택지**

**선택지 1: 상세한 스타일 가이드 프롬프트**
```python
prompt = """
Rules:
1. Always use explicit column names (no SELECT *)
2. Use LEFT JOIN instead of RIGHT JOIN
3. Date format: YYYY-MM-DD
4. Use snake_case for aliases
... (50개 규칙)
"""
```
- ✅ 장점: 명확함
- ❌ 단점:
  - 토큰 많이 사용
  - LLM이 모든 규칙 기억 못 함
  - 규칙 추가 시 프롬프트 수정
- 왜 안 됨: 비효율적

**선택지 2 (최종): Few-shot Learning - 예시로 학습**
```python
message_log = [
    system_message("You are a SQL expert."),

    # Few-shot 예시 1
    user_message("Show top 10 customers"),
    assistant_message("""
        SELECT
            c.customer_id,
            c.customer_name,
            SUM(o.total_amount) as total_sales
        FROM customers c
        LEFT JOIN orders o ON c.customer_id = o.customer_id
        GROUP BY c.customer_id, c.customer_name
        ORDER BY total_sales DESC
        LIMIT 10
    """),

    # Few-shot 예시 2
    user_message("Recent orders"),
    assistant_message("""
        SELECT
            order_id,
            order_date,
            customer_name
        FROM orders
        WHERE order_date >= CURRENT_DATE - INTERVAL '7 days'
        ORDER BY order_date DESC
    """),

    # 실제 질문
    user_message("Show me VIP customers")
]

# LLM은 예시의 스타일을 따라함:
# - 명시적 컬럼
# - LEFT JOIN
# - snake_case
# - 날짜 형식
```
- ✅ 장점:
  - LLM이 예시에서 패턴 학습
  - 규칙 명시 불필요 (암묵적 학습)
  - 새 스타일 추가는 예시만 추가
- ⚠️ 단점:
  - 예시가 많으면 토큰 많이 사용
  - 예시 품질이 중요
- 왜 선택: LLM의 강점 활용

**최종 해결책**

```python
# get_sql_prompt에서 Few-shot 예시 추가
message_log = [self.system_message(initial_prompt)]

# Vector search로 가져온 유사 Question-SQL을 Few-shot으로 사용
for example in question_sql_list:  # 보통 3-5개
    if example and "question" in example and "sql" in example:
        message_log.append(self.user_message(example["question"]))
        message_log.append(self.assistant_message(example["sql"]))

# 실제 질문
message_log.append(self.user_message(question))

# LLM에게 전송
llm_response = self.submit_prompt(message_log)
```

**핵심 아이디어**
1. **Retrieval-based Few-shot**: Vector search로 유사 SQL 자동 선택
2. **Context Learning**: LLM이 예시에서 스타일 + 로직 학습
3. **Dynamic Examples**: 질문마다 다른 예시 (관련 있는 것만)

**트레이드오프**
- 얻은 것: 일관된 SQL 스타일, 규칙 명시 불필요
- 희생한 것: Few-shot 예시 토큰 사용

---

## ⭐ 실전 적용 가이드

### 가이드 1: 커스텀 Vector Store 구현하기

**상황**: Vanna가 지원하지 않는 Vector DB 사용 (예: Elasticsearch)

#### Step 1: 요구사항 정의
- [ ] Vector DB API 확인 (검색, 추가, 삭제)
- [ ] Embedding 모델 선택 (OpenAI / HuggingFace / Custom)
- [ ] 메타데이터 구조 정의 (type, question, sql 등)

#### Step 2: 기본 구현

```python
from vanna.base import VannaBase
import pandas as pd
from typing import List

class Elasticsearch_VectorStore(VannaBase):
    def __init__(self, config=None):
        VannaBase.__init__(self, config=config)

        from elasticsearch import Elasticsearch
        self.es = Elasticsearch([config.get('host', 'localhost:9200')])
        self.index_name = config.get('index', 'vanna')

    # 1. Embedding 생성
    def generate_embedding(self, data: str, **kwargs) -> List[float]:
        import openai
        response = openai.Embedding.create(
            input=data,
            model="text-embedding-ada-002"
        )
        return response['data'][0]['embedding']

    # 2. 유사 Question-SQL 검색
    def get_similar_question_sql(self, question: str, **kwargs) -> list:
        embedding = self.generate_embedding(question)

        # Elasticsearch kNN search
        response = self.es.search(
            index=self.index_name,
            body={
                "knn": {
                    "field": "embedding",
                    "query_vector": embedding,
                    "k": 5,
                    "num_candidates": 100
                },
                "query": {
                    "term": {"type": "question_sql"}
                }
            }
        )

        return [
            {
                "question": hit['_source']['question'],
                "sql": hit['_source']['sql'],
                "similarity": hit['_score']
            }
            for hit in response['hits']['hits']
        ]

    # 3. 관련 DDL 검색
    def get_related_ddl(self, question: str, **kwargs) -> list:
        embedding = self.generate_embedding(question)

        response = self.es.search(
            index=self.index_name,
            body={
                "knn": {
                    "field": "embedding",
                    "query_vector": embedding,
                    "k": 3,
                    "num_candidates": 50
                },
                "query": {
                    "term": {"type": "ddl"}
                }
            }
        )

        return [hit['_source']['ddl'] for hit in response['hits']['hits']]

    # 4. 관련 문서 검색
    def get_related_documentation(self, question: str, **kwargs) -> list:
        embedding = self.generate_embedding(question)

        response = self.es.search(
            index=self.index_name,
            body={
                "knn": {
                    "field": "embedding",
                    "query_vector": embedding,
                    "k": 5,
                    "num_candidates": 50
                },
                "query": {
                    "term": {"type": "documentation"}
                }
            }
        )

        return [hit['_source']['doc'] for hit in response['hits']['hits']]

    # 5. Question-SQL 추가
    def add_question_sql(self, question: str, sql: str, **kwargs) -> str:
        embedding = self.generate_embedding(f"{question} {sql}")
        doc_id = kwargs.get('id', None)

        self.es.index(
            index=self.index_name,
            id=doc_id,
            body={
                "type": "question_sql",
                "question": question,
                "sql": sql,
                "embedding": embedding
            }
        )

        return doc_id

    # 6. DDL 추가
    def add_ddl(self, ddl: str, **kwargs) -> str:
        embedding = self.generate_embedding(ddl)
        doc_id = kwargs.get('id', None)

        self.es.index(
            index=self.index_name,
            id=doc_id,
            body={
                "type": "ddl",
                "ddl": ddl,
                "embedding": embedding
            }
        )

        return doc_id

    # 7. 문서 추가
    def add_documentation(self, documentation: str, **kwargs) -> str:
        embedding = self.generate_embedding(documentation)
        doc_id = kwargs.get('id', None)

        self.es.index(
            index=self.index_name,
            id=doc_id,
            body={
                "type": "documentation",
                "doc": documentation,
                "embedding": embedding
            }
        )

        return doc_id

    # 8. 학습 데이터 조회
    def get_training_data(self, **kwargs) -> pd.DataFrame:
        response = self.es.search(
            index=self.index_name,
            body={"query": {"match_all": {}}, "size": 1000}
        )

        data = []
        for hit in response['hits']['hits']:
            source = hit['_source']
            data.append({
                'id': hit['_id'],
                'type': source['type'],
                'content': source.get('question', source.get('ddl', source.get('doc')))
            })

        return pd.DataFrame(data)

    # 9. 데이터 삭제
    def remove_training_data(self, id: str, **kwargs) -> bool:
        try:
            self.es.delete(index=self.index_name, id=id)
            return True
        except:
            return False
```

#### Step 3: 사용 예시

```python
from my_vanna import Elasticsearch_VectorStore
from vanna.openai import OpenAI_Chat

class MyVanna(Elasticsearch_VectorStore, OpenAI_Chat):
    def __init__(self, config):
        Elasticsearch_VectorStore.__init__(self, config)
        OpenAI_Chat.__init__(self, config)

vn = MyVanna(config={
    'host': 'localhost:9200',
    'index': 'vanna',
    'api_key': 'sk-...',
    'model': 'gpt-4'
})

vn.train(ddl="CREATE TABLE customers (...)")
sql = vn.generate_sql("Show top customers")
```

---

### 가이드 2: 커스텀 LLM 구현하기

**상황**: Vanna가 지원하지 않는 LLM 사용 (예: Cohere)

#### Step 1: LLM API 확인
- [ ] 메시지 형식 확인
- [ ] API 키 필요 여부
- [ ] 응답 형식 확인

#### Step 2: 구현

```python
from vanna.base import VannaBase

class Cohere_Chat(VannaBase):
    def __init__(self, config=None):
        VannaBase.__init__(self, config=config)
        import cohere
        self.co = cohere.Client(config.get('api_key'))
        self.model = config.get('model', 'command')

    def system_message(self, message: str) -> dict:
        # Cohere는 system을 별도 처리
        return {"role": "SYSTEM", "message": message}

    def user_message(self, message: str) -> dict:
        return {"role": "USER", "message": message}

    def assistant_message(self, message: str) -> dict:
        return {"role": "CHATBOT", "message": message}

    def submit_prompt(self, prompt: list, **kwargs) -> str:
        # Cohere 형식으로 변환
        chat_history = []
        system_prompt = ""

        for msg in prompt:
            if msg['role'] == 'SYSTEM':
                system_prompt = msg['message']
            else:
                chat_history.append({
                    'role': msg['role'],
                    'message': msg['message']
                })

        # 마지막 메시지가 사용자 질문
        user_query = chat_history[-1]['message']
        chat_history = chat_history[:-1]  # 제거

        response = self.co.chat(
            message=user_query,
            chat_history=chat_history,
            preamble=system_prompt,
            model=self.model,
            **kwargs
        )

        return response.text
```

---

### 가이드 3: 토큰 최적화 - 정확한 토큰 계산

**상황**: `str_to_approx_token_count`가 부정확해서 토큰 초과

#### Step 1: tiktoken 설치

```bash
pip install tiktoken
```

#### Step 2: 정확한 토큰 계산 구현

```python
from vanna.base import VannaBase
import tiktoken

class OptimizedVanna(VannaBase):
    def __init__(self, config=None):
        super().__init__(config)
        # GPT-4 인코더
        self.encoding = tiktoken.encoding_for_model("gpt-4")

    def str_to_approx_token_count(self, string: str) -> int:
        # 정확한 토큰 계산
        return len(self.encoding.encode(string))

    def add_ddl_to_prompt(self, initial_prompt, ddl_list, max_tokens=14000):
        # 기존 로직 그대로 (토큰 계산만 정확해짐)
        if len(ddl_list) > 0:
            initial_prompt += "\n===Tables \n"

            for ddl in ddl_list:
                current = self.str_to_approx_token_count(initial_prompt)
                ddl_tokens = self.str_to_approx_token_count(ddl)

                # 여유 마진 (500 토큰)
                if current + ddl_tokens < max_tokens - 500:
                    initial_prompt += f"{ddl}\n\n"
                else:
                    print(f"[Token Limit] Skipped {len(ddl_list) - ddl_list.index(ddl)} DDLs")
                    break

        return initial_prompt
```

#### Step 3: 동적 max_tokens 설정

```python
class AdaptiveVanna(OptimizedVanna):
    def get_sql_prompt(self, question, question_sql_list, ddl_list, doc_list, **kwargs):
        # 질문 길이에 따라 max_tokens 조정
        question_tokens = self.str_to_approx_token_count(question)

        if question_tokens > 500:
            # 긴 질문 → 컨텍스트 줄이기
            adjusted_max = self.max_tokens - question_tokens - 2000
        else:
            adjusted_max = self.max_tokens - 2000  # 응답 공간 확보

        return super().get_sql_prompt(
            question=question,
            question_sql_list=question_sql_list[:2],  # 예시 줄이기
            ddl_list=ddl_list,
            doc_list=doc_list,
            **kwargs
        )
```

---

## ⭐ 안티패턴과 흔한 실수

### 실수 1: 추상 메서드 미구현

**❌ 나쁜 예:**
```python
class MyVectorStore(VannaBase):
    def add_question_sql(self, question, sql):
        # 구현함
        pass

    # ❌ 나머지 8개 메서드 미구현

vn = MyVectorStore()  # ❌ 에러!
# TypeError: Can't instantiate abstract class with abstract methods
```

**문제:**
- Python ABC는 모든 추상 메서드 구현 강제
- 하나라도 빠지면 인스턴스 생성 불가

**✅ 좋은 예:**
```python
class MyVectorStore(VannaBase):
    # 9개 메서드 모두 구현
    def generate_embedding(self, data): ...
    def get_similar_question_sql(self, question): ...
    def get_related_ddl(self, question): ...
    def get_related_documentation(self, question): ...
    def add_question_sql(self, question, sql): ...
    def add_ddl(self, ddl): ...
    def add_documentation(self, documentation): ...
    def get_training_data(self): ...
    def remove_training_data(self, id): ...

    # LLM 메서드도 (또는 다른 클래스와 Mixin)
    def system_message(self, msg): ...
    def user_message(self, msg): ...
    def assistant_message(self, msg): ...
    def submit_prompt(self, prompt): ...
```

---

### 실수 2: 다중 상속 시 `__init__` 충돌

**❌ 나쁜 예:**
```python
class ChromaDB_VectorStore(VannaBase):
    def __init__(self, config):
        # super() 안 함!
        self.chroma = init_chroma(config)

class OpenAI_Chat(VannaBase):
    def __init__(self, config):
        # super() 안 함!
        self.api_key = config['api_key']

class MyVanna(ChromaDB_VectorStore, OpenAI_Chat):
    def __init__(self, config):
        ChromaDB_VectorStore.__init__(self, config)
        OpenAI_Chat.__init__(self, config)  # ❌ VannaBase.__init__ 2번 호출!
```

**문제:**
- `VannaBase.__init__`이 두 번 호출됨
- 설정 중복, 잠재적 버그

**✅ 좋은 예:**
```python
class ChromaDB_VectorStore(VannaBase):
    def __init__(self, config=None):
        super().__init__(config)  # ✅ MRO 따라 호출
        self.chroma = init_chroma(config)

class OpenAI_Chat(VannaBase):
    def __init__(self, config=None):
        super().__init__(config)  # ✅
        self.api_key = config['api_key']

class MyVanna(ChromaDB_VectorStore, OpenAI_Chat):
    def __init__(self, config=None):
        super().__init__(config)  # ✅ 한 번만 호출됨

# MRO: MyVanna → ChromaDB_VectorStore → OpenAI_Chat → VannaBase
```

---

### 실수 3: 토큰 계산 없이 모든 컨텍스트 추가

**❌ 나쁜 예:**
```python
def get_sql_prompt(self, question, ddl_list, doc_list):
    prompt = "You are a SQL expert.\n\n"

    # 토큰 체크 없이 모두 추가
    for ddl in ddl_list:  # 100개 테이블!
        prompt += f"{ddl}\n\n"

    for doc in doc_list:  # 50개 문서!
        prompt += f"{doc}\n\n"

    return prompt  # ❌ 50K 토큰 초과!
```

**문제:**
- LLM API 토큰 제한 초과 에러
- 또는 앞부분 잘림 (truncation)

**✅ 좋은 예:**
```python
def get_sql_prompt(self, question, ddl_list, doc_list):
    max_tokens = 14000
    prompt = "You are a SQL expert.\n\n"

    # DDL 추가 (토큰 체크)
    prompt = self.add_ddl_to_prompt(prompt, ddl_list, max_tokens)

    # Doc 추가 (토큰 체크)
    prompt = self.add_documentation_to_prompt(prompt, doc_list, max_tokens)

    # 최종 확인
    final_tokens = self.str_to_approx_token_count(prompt)
    print(f"[Token Usage] {final_tokens}/{max_tokens}")

    return prompt
```

---

### 실수 4: 잘못된 메시지 순서 (Few-shot)

**❌ 나쁜 예:**
```python
message_log = [
    system_message("You are SQL expert"),

    # ❌ 순서 틀림!
    assistant_message("SELECT * FROM users"),
    user_message("Show all users"),

    user_message("Show top customers")
]
```

**문제:**
- LLM이 Few-shot 예시를 이해 못 함
- 응답 품질 저하

**✅ 좋은 예:**
```python
message_log = [
    system_message("You are SQL expert"),

    # ✅ 올바른 순서: user → assistant
    user_message("Show all users"),
    assistant_message("SELECT id, name, email FROM users"),

    user_message("Show active users"),
    assistant_message("SELECT * FROM users WHERE status = 'active'"),

    # 실제 질문
    user_message("Show top customers")
]
```

---

### 실수 5: Vector Search 결과를 필터링 없이 사용

**❌ 나쁜 예:**
```python
def get_sql_prompt(self, question, ...):
    # Vector search 결과 그대로 사용
    similar_sqls = self.get_similar_question_sql(question)  # 5개

    # ❌ 유사도 0.2인 것도 포함!
    for sql in similar_sqls:
        message_log.append(user_message(sql['question']))
        message_log.append(assistant_message(sql['sql']))
```

**문제:**
- 관련 없는 SQL이 Few-shot 예시로 → LLM 혼란
- 토큰 낭비

**✅ 좋은 예:**
```python
def get_sql_prompt(self, question, ...):
    similar_sqls = self.get_similar_question_sql(question, n=10)

    # 1. 유사도 필터링
    filtered = [s for s in similar_sqls if s.get('similarity', 0) > 0.7]

    # 2. 상위 3개만
    top_3 = filtered[:3]

    # 3. Few-shot에 추가
    for sql in top_3:
        message_log.append(user_message(sql['question']))
        message_log.append(assistant_message(sql['sql']))
```

---

### 실수 6: `max_tokens`와 LLM output tokens 혼동

**❌ 나쁜 예:**
```python
self.max_tokens = 14000  # Vanna의 프롬프트 제한

# OpenAI 호출
openai.ChatCompletion.create(
    model="gpt-4",
    messages=prompt,
    max_tokens=14000  # ❌ 이건 LLM 응답 길이!
)
```

**문제:**
- Vanna `max_tokens`: 프롬프트 크기 제한
- LLM `max_tokens`: 응답 크기 제한
- 혼동하면 응답이 너무 길거나 잘림

**✅ 좋은 예:**
```python
# Vanna 프롬프트 제한
self.max_tokens = 14000

prompt = self.get_sql_prompt(...)  # 14K 이내로 조립

# OpenAI 호출 (응답 길이 제한)
openai.ChatCompletion.create(
    model="gpt-4",
    messages=prompt,
    max_tokens=500  # SQL은 보통 짧음
)
```

---

### 실수 7: Static documentation 중복 추가

**❌ 나쁜 예:**
```python
# 매번 static doc 추가
for i in range(10):
    doc_list.append(self.static_documentation)  # ❌ 중복!

prompt = self.add_documentation_to_prompt(initial_prompt, doc_list)
```

**문제:**
- 동일 문서가 10번 추가
- 토큰 낭비, LLM 혼란

**✅ 좋은 예:**
```python
# Vanna는 이미 처리함 (base.py:610-611)
if self.static_documentation != "":
    doc_list.append(self.static_documentation)  # 한 번만
```

---

## ⭐ 스케일 고려사항

### 소규모 (< 1000 documents)

**권장 사항:**
- ✅ 간단한 토큰 계산 (`len(string) / 4`)
- ✅ Few-shot 예시 5개까지 OK
- ⚠️ tiktoken 불필요 (오버헤드)

**구현 예시:**
```python
class SmallVanna(VannaBase):
    def __init__(self, config):
        super().__init__(config)
        self.max_tokens = 8000  # 작은 프롬프트

    def str_to_approx_token_count(self, string):
        return len(string) / 4  # 간단!
```

**모니터링:**
```python
# 기본 로깅만
print(f"Prompt tokens: ~{vn.str_to_approx_token_count(prompt)}")
```

---

### 중규모 (1000-10000 documents)

**권장 사항:**
- ✅ tiktoken으로 정확한 토큰 계산
- ✅ Few-shot 예시 3개로 제한
- ✅ Vector search 유사도 필터링 (> 0.7)

**구현 예시:**
```python
import tiktoken

class MediumVanna(VannaBase):
    def __init__(self, config):
        super().__init__(config)
        self.max_tokens = 14000
        self.encoding = tiktoken.encoding_for_model("gpt-4")

    def str_to_approx_token_count(self, string):
        return len(self.encoding.encode(string))

    def get_sql_prompt(self, question, question_sql_list, ddl_list, doc_list):
        # 유사도 필터링
        filtered_sqls = [
            s for s in question_sql_list
            if s.get('similarity', 0) > 0.7
        ][:3]  # 상위 3개

        return super().get_sql_prompt(
            question, filtered_sqls, ddl_list, doc_list
        )
```

**모니터링:**
```python
import time

class MonitoredVanna(MediumVanna):
    def get_sql_prompt(self, *args, **kwargs):
        start = time.time()
        prompt = super().get_sql_prompt(*args, **kwargs)
        duration = time.time() - start

        tokens = self.str_to_approx_token_count(str(prompt))
        print(f"[Prompt Build] {duration:.3f}s, {tokens} tokens")

        return prompt
```

---

### 대규모 (10000+ documents)

**권장 사항:**
- ✅ tiktoken + 캐싱
- ✅ Few-shot 예시 1-2개로 제한
- ✅ 토큰 예산 동적 조정
- ✅ Embedding 캐싱 (같은 텍스트 반복)

**구현 예시:**
```python
import tiktoken
from functools import lru_cache

class LargeVanna(VannaBase):
    def __init__(self, config):
        super().__init__(config)
        self.max_tokens = 12000  # 여유 확보
        self.encoding = tiktoken.encoding_for_model("gpt-4")

    @lru_cache(maxsize=1000)
    def str_to_approx_token_count(self, string):
        # 캐싱 (같은 DDL 반복 계산 방지)
        return len(self.encoding.encode(string))

    def get_sql_prompt(self, question, question_sql_list, ddl_list, doc_list):
        # 토큰 예산 동적 계산
        question_tokens = self.str_to_approx_token_count(question)
        available = self.max_tokens - question_tokens - 2000  # 응답 공간

        # DDL 우선 (가장 중요)
        ddl_budget = available * 0.6  # 60%
        doc_budget = available * 0.3  # 30%
        few_shot_budget = available * 0.1  # 10%

        # Few-shot 최소화 (1개만)
        filtered_sqls = question_sql_list[:1]

        # 동적 max_tokens
        prompt = self.get_sql_prompt_with_budget(
            question=question,
            question_sql_list=filtered_sqls,
            ddl_list=ddl_list,
            doc_list=doc_list,
            ddl_budget=ddl_budget,
            doc_budget=doc_budget
        )

        return prompt

    @lru_cache(maxsize=500)
    def generate_embedding(self, data: str):
        # Embedding 캐싱 (같은 텍스트 반복)
        return super().generate_embedding(data)
```

**모니터링 (Prometheus):**
```python
from prometheus_client import Histogram, Gauge

prompt_tokens_gauge = Gauge('vanna_prompt_tokens', 'Prompt size in tokens')
prompt_build_duration = Histogram('vanna_prompt_build_seconds', 'Prompt build time')

class MonitoredLargeVanna(LargeVanna):
    def get_sql_prompt(self, *args, **kwargs):
        with prompt_build_duration.time():
            prompt = super().get_sql_prompt(*args, **kwargs)

        tokens = self.str_to_approx_token_count(str(prompt))
        prompt_tokens_gauge.set(tokens)

        if tokens > self.max_tokens * 0.9:
            print(f"[WARNING] Prompt near limit: {tokens}/{self.max_tokens}")

        return prompt
```

**알림 설정:**
```yaml
# Prometheus alerts
- alert: PromptTooLarge
  expr: vanna_prompt_tokens > 13000
  annotations:
    summary: "Prompt size > 13K tokens (near 14K limit)"

- alert: SlowPromptBuild
  expr: histogram_quantile(0.95, vanna_prompt_build_seconds) > 1.0
  annotations:
    summary: "P95 prompt build time > 1s (tiktoken slow?)"
```

---

## 💡 배운 점

### 1. Mixin 패턴은 Python RAG 프레임워크의 강력한 도구
**핵심 개념**: 다중 상속으로 Vector DB + LLM 조합
**언제 사용?**: 사용자에게 구현체 선택권을 주고 싶을 때
**적용 가능한 곳**:
- ML 프레임워크 (모델 + 옵티마이저 + 스케줄러 조합)
- 웹 프레임워크 (인증 + 캐싱 + 로깅 Mixin)

### 2. 추상 클래스는 확장 가이드
**핵심 개념**: 추상 메서드로 "무엇을 구현해야 하는지" 명시
**언제 사용?**: 플러그인 아키텍처, 확장 가능한 프레임워크
**적용 가능한 곳**:
- 데이터베이스 드라이버 (JDBC 스타일)
- 파일 시스템 추상화 (S3, GCS, 로컬)

### 3. 토큰 관리는 LLM 비용의 핵심
**핵심 개념**: 프롬프트 크기 = 비용, 토큰 예산 관리 필수
**언제 사용?**: 대규모 RAG, LLM 서비스
**적용 가능한 곳**:
- 동적 프롬프트 조립 (우선순위 기반)
- 토큰 예산 분할 (DDL 60%, Doc 30%, Few-shot 10%)

### 4. Few-shot Learning은 규칙보다 강력
**핵심 개념**: 예시로 스타일 + 로직 학습
**언제 사용?**: 일관된 출력 형식 필요할 때
**적용 가능한 곳**:
- 코드 생성기 (스타일 가이드 불필요)
- 번역기 (tone/style 예시로 학습)

### 5. Retrieval-based Few-shot은 RAG의 핵심
**핵심 개념**: Vector search로 관련 예시만 선택
**언제 사용?**: 대규모 예시 DB
**적용 가능한 곳**:
- 고객 응대 챗봇 (유사 과거 대화 검색)
- 코드 완성 (유사 코드 스니펫 검색)

### 6. 정확한 토큰 계산 vs 속도 트레이드오프
**핵심 개념**: `len(string)/4` (빠름) vs tiktoken (정확)
**언제 사용?**: 소규모는 간단, 대규모는 tiktoken
**적용 가능한 곳**:
- 실시간 서비스 (캐싱 필수)
- 배치 작업 (정확도 우선)

### 7. `super()`는 다중 상속에서 필수
**핵심 개념**: MRO 따라 올바른 순서로 초기화
**언제 사용?**: Mixin 패턴 사용 시 항상
**적용 가능한 곳**:
- Django 클래스 기반 뷰
- Python 믹스인 라이브러리

### 8. 추상 메서드 개수는 구현 부담
**핵심 개념**: 9개 추상 메서드 = 확장자 부담
**언제 사용?**: 필수 메서드만 추상화, 옵션은 디폴트 구현
**적용 가능한 곳**:
- 플러그인 시스템 (최소 인터페이스)
- 템플릿 메서드 패턴

---

## 📊 요약

| 항목 | 내용 |
|------|------|
| **핵심 문제** | Vector DB/LLM 독립성 + 토큰 제한 관리 |
| **핵심 패턴** | Mixin (다중 상속) + 추상 클래스 + 동적 토큰 예산 |
| **주요 트레이드오프** | 확장성 & 유연성 vs 다중 상속 복잡도 & 구현 부담 |
| **핵심 기법** | 1) 9개 추상 메서드로 Vector Store 인터페이스<br>2) 4개 추상 메서드로 LLM 인터페이스<br>3) 토큰 계산하며 프롬프트 조립<br>4) Retrieval-based Few-shot |
| **적용 시 주의** | 1) 모든 추상 메서드 구현 필수<br>2) `super()` 사용 (MRO)<br>3) 토큰 계산 vs 속도 트레이드오프<br>4) Few-shot 유사도 필터링 |
| **스케일 전략** | 소규모: 간단한 토큰 계산<br>중규모: tiktoken + 필터링<br>대규모: 캐싱 + 동적 예산 + 최소 Few-shot |
| **실무 적용** | RAG 프레임워크, 플러그인 아키텍처, LLM 서비스 |

---

## 🔗 다음 단계

- **Part 3**: 데이터베이스 연결 패턴 분석 (Snowflake, Postgres, BigQuery 등)
- **Part 4**: 고수준 API (`ask()`, `train()`) 분석

---

**✅ Part 2 분석 완료!**
