# chromadb_vector.py - ChromaDB Vector Store 구현

> **파일**: `src/vanna/chromadb/chromadb_vector.py` (257 lines)
> **역할**: ChromaDB를 사용한 Vector Store 구현 - VannaBase 추상 메서드 구체화
> **주요 클래스**: `ChromaDB_VectorStore`

---

## 📋 파일 개요

### 기본 정보
- **경로**: `src/vanna/chromadb/chromadb_vector.py`
- **줄 수**: 257 lines
- **주요 클래스**: `ChromaDB_VectorStore(VannaBase)`
- **핵심 역할**: RAG를 위한 Vector 저장/검색 구현 (ChromaDB 사용)

### 핵심 역할 (한 문장)
**"VannaBase의 9개 추상 메서드를 ChromaDB API로 구현 - 3개 Collection(sql/ddl/doc)에 벡터 저장하고 유사도 검색"**

### 누가 사용하는가?
- **Vanna 사용자**: `class MyVanna(ChromaDB_VectorStore, OpenAI_Chat)` 형태로 Mixin
- **VannaBase**: `generate_sql()` 내부에서 `get_similar_question_sql()` 호출
- **로컬 개발자**: 파일 기반 Vector DB (no server)

---

## 🔍 해결하는 핵심 문제들

### 문제 1: 3가지 타입 데이터 분리 저장

**문제**
- RAG에 필요한 데이터 3가지:
  1. Question-SQL 쌍 (과거 예시)
  2. DDL (스키마)
  3. Documentation (비즈니스 용어)
- 하나의 Collection에 섞어 저장? 분리 저장?
- 검색 시 타입별로 다른 개수 반환하려면?

**문제가 없었다면?**
```python
# 하나의 Collection에 모두 저장
collection.add(documents=[
    "CREATE TABLE ...",  # DDL
    '{"question": "...", "sql": "..."}',  # SQL
    "VIP = sales > 10000"  # Doc
])

# 검색 시 타입 구분 어려움
results = collection.query("top customers", n_results=10)
# → DDL, SQL, Doc 섞여서 나옴
```

**고민했던 선택지**

**선택지 1: 하나의 Collection + 메타데이터 필터**
```python
collection.add(
    documents=["CREATE TABLE ..."],
    metadatas=[{"type": "ddl"}]
)

# 검색 시 필터
results = collection.query(
    query_texts=["..."],
    where={"type": "ddl"}
)
```
- ✅ 장점: Collection 하나만 관리
- ❌ 단점:
  - 검색 시 필터링 비용
  - 타입별 n_results 설정 어려움
  - 같은 인덱스에서 검색 → 타입별 최적화 불가
- 왜 안 됨: 타입별 검색 개수 조절 필요

**선택지 2 (최종): 3개 Collection 분리**
```python
class ChromaDB_VectorStore(VannaBase):
    def __init__(self, config=None):
        # 3개 Collection 생성
        self.sql_collection = self.chroma_client.get_or_create_collection(
            name="sql",
            embedding_function=self.embedding_function
        )
        self.ddl_collection = self.chroma_client.get_or_create_collection(
            name="ddl",
            embedding_function=self.embedding_function
        )
        self.documentation_collection = self.chroma_client.get_or_create_collection(
            name="documentation",
            embedding_function=self.embedding_function
        )

        # 타입별 검색 개수 설정
        self.n_results_sql = config.get("n_results_sql", 10)
        self.n_results_ddl = config.get("n_results_ddl", 10)
        self.n_results_documentation = config.get("n_results_documentation", 10)

    def get_similar_question_sql(self, question: str) -> list:
        return self.sql_collection.query(
            query_texts=[question],
            n_results=self.n_results_sql  # SQL은 10개
        )

    def get_related_ddl(self, question: str) -> list:
        return self.ddl_collection.query(
            query_texts=[question],
            n_results=self.n_results_ddl  # DDL은 3개만
        )
```
- ✅ 장점:
  - 타입별 독립 검색 (속도 빠름)
  - 타입별 n_results 다르게 설정 가능
  - Collection별로 다른 임베딩 함수 사용 가능 (확장성)
- ⚠️ 단점: Collection 3개 관리
- 왜 선택: 검색 성능 + 유연성

**최종 해결책**

```python
# 초기화: 3개 Collection 생성
self.documentation_collection = self.chroma_client.get_or_create_collection(
    name="documentation",
    embedding_function=self.embedding_function,
    metadata=collection_metadata
)
self.ddl_collection = self.chroma_client.get_or_create_collection(
    name="ddl",
    embedding_function=self.embedding_function,
    metadata=collection_metadata
)
self.sql_collection = self.chroma_client.get_or_create_collection(
    name="sql",
    embedding_function=self.embedding_function,
    metadata=collection_metadata
)

# 추가: Collection 자동 선택
def add_question_sql(self, question: str, sql: str) -> str:
    question_sql_json = json.dumps({"question": question, "sql": sql})
    id = deterministic_uuid(question_sql_json) + "-sql"
    self.sql_collection.add(...)  # SQL Collection에 추가
    return id

def add_ddl(self, ddl: str) -> str:
    id = deterministic_uuid(ddl) + "-ddl"
    self.ddl_collection.add(...)  # DDL Collection에 추가
    return id

# 검색: Collection별로
def get_similar_question_sql(self, question: str) -> list:
    return self.sql_collection.query(
        query_texts=[question],
        n_results=self.n_results_sql
    )

def get_related_ddl(self, question: str) -> list:
    return self.ddl_collection.query(
        query_texts=[question],
        n_results=self.n_results_ddl
    )
```

**핵심 아이디어**
1. **Separation of Concerns**: 타입별 독립 저장소
2. **Type-specific Configuration**: 타입마다 다른 n_results
3. **ID Suffix**: `-sql`, `-ddl`, `-doc` 로 타입 구분

**트레이드오프**
- 얻은 것: 검색 성능, 타입별 설정, 확장성
- 희생한 것: Collection 관리 복잡도 (3개)

---

### 문제 2: 중복 데이터 방지 - Deterministic UUID

**문제**
- 같은 DDL을 여러 번 `train(ddl=...)` 호출하면?
- UUID 랜덤 생성 시 중복 저장됨
- Vector DB 용량 낭비, 검색 결과 중복

**문제가 없었다면?**
```python
import uuid

def add_ddl(self, ddl: str) -> str:
    id = str(uuid.uuid4())  # 랜덤 UUID
    self.ddl_collection.add(documents=ddl, ids=id)
    return id

# 같은 DDL 2번 호출
vn.train(ddl="CREATE TABLE users (...)")
vn.train(ddl="CREATE TABLE users (...)")
# → 2개의 다른 ID로 저장됨! 중복!
```

**고민했던 선택지**

**선택지 1: 저장 전에 검색해서 중복 체크**
```python
def add_ddl(self, ddl: str) -> str:
    # 1. 같은 내용 있는지 검색
    results = self.ddl_collection.query(query_texts=[ddl], n_results=1)
    if results and results["documents"][0] == ddl:
        return results["ids"][0]  # 이미 있으면 ID 반환

    # 2. 없으면 추가
    id = str(uuid.uuid4())
    self.ddl_collection.add(documents=ddl, ids=id)
    return id
```
- ✅ 장점: 중복 방지
- ❌ 단점:
  - 매번 검색 (느림)
  - Semantic search는 정확한 매칭 보장 안 함
- 왜 안 됨: 성능 문제

**선택지 2 (최종): Deterministic UUID (내용 기반 ID)**
```python
from ..utils import deterministic_uuid

def add_ddl(self, ddl: str) -> str:
    # 내용의 해시로 UUID 생성
    id = deterministic_uuid(ddl) + "-ddl"
    self.ddl_collection.add(
        documents=ddl,
        embeddings=self.generate_embedding(ddl),
        ids=id
    )
    return id

# utils.py
import hashlib
import uuid

def deterministic_uuid(content: str) -> str:
    """내용의 SHA256 해시를 UUID로 변환"""
    hash_object = hashlib.sha256(content.encode())
    hash_hex = hash_object.hexdigest()
    # 해시의 앞 32자를 UUID 형식으로
    return str(uuid.UUID(hash_hex[:32]))

# 사용 예시
vn.train(ddl="CREATE TABLE users (...)")
# → ID: "abc123...xyz-ddl"

vn.train(ddl="CREATE TABLE users (...)")  # 같은 DDL
# → ID: "abc123...xyz-ddl" (동일!)
# ChromaDB가 같은 ID면 update (덮어쓰기)
```
- ✅ 장점:
  - 검색 불필요 (빠름)
  - 같은 내용 = 같은 ID (중복 자동 방지)
  - ChromaDB의 upsert 동작 활용
- ⚠️ 단점: 내용 1글자만 달라도 다른 ID
- 왜 선택: 성능 + 자동 중복 제거

**최종 해결책**

```python
def add_question_sql(self, question: str, sql: str) -> str:
    # JSON으로 serialize (순서 보장)
    question_sql_json = json.dumps(
        {"question": question, "sql": sql},
        ensure_ascii=False  # 한글 등 유니코드 그대로
    )

    # Deterministic UUID + 타입 suffix
    id = deterministic_uuid(question_sql_json) + "-sql"

    self.sql_collection.add(
        documents=question_sql_json,
        embeddings=self.generate_embedding(question_sql_json),
        ids=id
    )
    return id

def add_ddl(self, ddl: str) -> str:
    id = deterministic_uuid(ddl) + "-ddl"
    self.ddl_collection.add(
        documents=ddl,
        embeddings=self.generate_embedding(ddl),
        ids=id
    )
    return id
```

**핵심 아이디어**
1. **Content-addressable Storage**: 내용 = 주소
2. **Hash-based ID**: SHA256 → UUID
3. **Type Suffix**: `-sql`, `-ddl`, `-doc` 로 타입별 구분 (같은 내용이라도 타입 다르면 다른 ID)

**트레이드오프**
- 얻은 것: 자동 중복 제거, 빠른 추가
- 희생한 것: 내용 약간 변경도 새 ID (업데이트 어려움)

---

### 문제 3: Persistent vs In-memory - 개발/프로덕션 모드

**문제**
- 개발: 빠른 테스트 필요 (in-memory)
- 프로덕션: 데이터 영속성 필요 (persistent)
- 어떻게 둘 다 지원?

**문제가 없었다면?**
```python
# Persistent만 지원
client = chromadb.PersistentClient(path="./chromadb")
# → 테스트 시 파일 생성됨, 느림
```

**고민했던 선택지**

**선택지 1: 별도 클래스**
```python
class ChromaDB_Persistent(VannaBase):
    def __init__(self):
        self.client = chromadb.PersistentClient(...)

class ChromaDB_InMemory(VannaBase):
    def __init__(self):
        self.client = chromadb.EphemeralClient(...)
```
- ✅ 명확함
- ❌ 코드 중복
- 왜 안 됨: 하나로 충분

**선택지 2 (최종): Config 파라미터로 선택**
```python
def __init__(self, config=None):
    if config is None:
        config = {}

    path = config.get("path", ".")
    curr_client = config.get("client", "persistent")  # 기본값 persistent

    if curr_client == "persistent":
        self.chroma_client = chromadb.PersistentClient(
            path=path,
            settings=Settings(anonymized_telemetry=False)
        )
    elif curr_client == "in-memory":
        self.chroma_client = chromadb.EphemeralClient(
            settings=Settings(anonymized_telemetry=False)
        )
    elif isinstance(curr_client, chromadb.api.client.Client):
        # 고급: 직접 생성한 client 전달 가능
        self.chroma_client = curr_client
    else:
        raise ValueError(f"Unsupported client: {curr_client}")

# 사용 예시
# Persistent (기본)
vn = MyVanna(config={"path": "./my_chromadb"})

# In-memory (테스트)
vn = MyVanna(config={"client": "in-memory"})

# 고급: 커스텀 client
custom_client = chromadb.HttpClient(host="remote-server")
vn = MyVanna(config={"client": custom_client})
```
- ✅ 장점:
  - 하나의 클래스로 3가지 모드
  - 고급 사용자는 커스텀 client 전달 가능
- ⚠️ 단점: Config 파라미터 이해 필요
- 왜 선택: 유연성

**핵심 아이디어**
1. **Factory Pattern**: Config에 따라 다른 client 생성
2. **Dependency Injection**: 고급 사용자는 직접 주입 가능
3. **Smart Defaults**: 기본값은 persistent (프로덕션 우선)

**트레이드오프**
- 얻은 것: 유연성, 테스트 용이성
- 희생한 것: Config 복잡도

---

### 문제 4: 검색 결과 파싱 - ChromaDB 응답 형식

**문제**
- ChromaDB query 결과 형식이 복잡함:
  ```python
  {
      "ids": [["id1", "id2"]],
      "documents": [[{"question": "...", "sql": "..."}]],
      "distances": [[0.1, 0.2]]
  }
  ```
- 이중 리스트 (batch query 지원)
- documents가 JSON 문자열일 수도, 객체일 수도
- 어떻게 일관되게 파싱?

**고민했던 선택지**

**선택지 1: 각 메서드에서 파싱**
```python
def get_similar_question_sql(self, question):
    results = self.sql_collection.query(...)
    if results and "documents" in results:
        docs = results["documents"]
        if len(docs) == 1 and isinstance(docs[0], list):
            return [json.loads(d) for d in docs[0]]
    return []

# 다른 메서드도 동일 로직 반복 ❌
```
- ❌ 코드 중복

**선택지 2 (최종): Static helper 메서드**
```python
@staticmethod
def _extract_documents(query_results) -> list:
    """ChromaDB query 결과에서 documents 추출"""
    if query_results is None:
        return []

    if "documents" in query_results:
        documents = query_results["documents"]

        # 이중 리스트 언래핑
        if len(documents) == 1 and isinstance(documents[0], list):
            try:
                # JSON 문자열이면 파싱
                documents = [json.loads(doc) for doc in documents[0]]
            except Exception:
                # 일반 문자열이면 그대로
                return documents[0]

        return documents

    return []

# 사용
def get_similar_question_sql(self, question: str) -> list:
    return ChromaDB_VectorStore._extract_documents(
        self.sql_collection.query(
            query_texts=[question],
            n_results=self.n_results_sql
        )
    )

def get_related_ddl(self, question: str) -> list:
    return ChromaDB_VectorStore._extract_documents(
        self.ddl_collection.query(
            query_texts=[question],
            n_results=self.n_results_ddl
        )
    )
```
- ✅ 장점: 코드 재사용, 일관된 파싱
- ⚠️ 단점: Static 메서드 (상속 시 override 어려움)
- 왜 선택: DRY 원칙

**핵심 아이디어**
1. **DRY (Don't Repeat Yourself)**: 공통 로직 추출
2. **Defensive Parsing**: 다양한 형식 처리
3. **Graceful Degradation**: 파싱 실패 시 원본 반환

**트레이드오프**
- 얻은 것: 코드 재사용, 일관성
- 희생한 것: Static (확장성 약간 감소)

---

### 문제 5: Collection 초기화 - Reset vs Delete

**문제**
- 사용자가 "학습 데이터 전부 삭제" 원할 때
- ChromaDB Collection 삭제 vs 데이터만 삭제?

**고민했던 선택지**

**선택지 1: Collection 내 데이터만 삭제**
```python
def remove_all_sql(self):
    all_ids = self.sql_collection.get()["ids"]
    self.sql_collection.delete(ids=all_ids)
```
- ✅ Collection 유지
- ❌ 대량 데이터 시 느림

**선택지 2 (최종): Collection 삭제 & 재생성**
```python
def remove_collection(self, collection_name: str) -> bool:
    """Collection을 empty 상태로 리셋"""
    if collection_name == "sql":
        # 1. 삭제
        self.chroma_client.delete_collection(name="sql")

        # 2. 재생성
        self.sql_collection = self.chroma_client.get_or_create_collection(
            name="sql",
            embedding_function=self.embedding_function
        )
        return True

    # ddl, documentation도 동일
    ...
```
- ✅ 빠름 (대량 데이터도)
- ⚠️ Collection 메타데이터도 초기화
- 왜 선택: 성능

**핵심 아이디어**
1. **Atomic Reset**: 삭제 + 재생성
2. **Type-specific**: Collection별로 리셋
3. **Reference Update**: `self.sql_collection` 참조 갱신

**트레이드오프**
- 얻은 것: 빠른 리셋
- 희생한 것: Collection 메타데이터 초기화

---

## ⭐ 실전 적용 가이드

### 가이드 1: ChromaDB 기본 사용

**상황**: 로컬에서 Vanna 시작

#### Step 1: 설치
```bash
pip install chromadb
```

#### Step 2: 기본 사용
```python
from vanna.chromadb import ChromaDB_VectorStore
from vanna.openai import OpenAI_Chat

class MyVanna(ChromaDB_VectorStore, OpenAI_Chat):
    def __init__(self, config=None):
        ChromaDB_VectorStore.__init__(self, config=config)
        OpenAI_Chat.__init__(self, config=config)

# Persistent (기본) - 데이터 영속화
vn = MyVanna(config={
    'api_key': 'sk-...',
    'model': 'gpt-4',
    'path': './my_vanna_db'  # 데이터 저장 위치
})

# 학습
vn.train(ddl="CREATE TABLE customers (...)")
vn.train(question="top customers", sql="SELECT * FROM ...")

# 사용
sql = vn.generate_sql("Show me VIP customers")
```

---

### 가이드 2: In-memory 모드 (테스트)

**상황**: 단위 테스트, 빠른 실험

#### Step 1: 구성
```python
import pytest

@pytest.fixture
def vn():
    """테스트용 in-memory Vanna"""
    return MyVanna(config={
        'api_key': 'sk-...',
        'client': 'in-memory'  # 파일 생성 안 함!
    })

def test_generate_sql(vn):
    # 테스트 데이터 추가
    vn.train(ddl="CREATE TABLE users (id INT, name TEXT)")
    vn.train(question="all users", sql="SELECT * FROM users")

    # 테스트
    sql = vn.generate_sql("show all users")
    assert "SELECT" in sql
    assert "users" in sql

    # 테스트 종료 시 자동 삭제 (in-memory)
```

---

### 가이드 3: 커스텀 Embedding 함수

**상황**: OpenAI 대신 HuggingFace 임베딩 사용

#### Step 1: Embedding 함수 정의
```python
from chromadb.utils import embedding_functions

# Sentence Transformers 사용
sentence_transformer_ef = embedding_functions.SentenceTransformerEmbeddingFunction(
    model_name="all-MiniLM-L6-v2"
)

vn = MyVanna(config={
    'api_key': 'sk-...',
    'embedding_function': sentence_transformer_ef,  # 커스텀 함수
    'path': './vanna_db'
})

# 사용은 동일
vn.train(ddl="...")
```

#### Step 2: 다국어 임베딩
```python
# 한국어 최적화 모델
multilingual_ef = embedding_functions.SentenceTransformerEmbeddingFunction(
    model_name="sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2"
)

vn = MyVanna(config={
    'embedding_function': multilingual_ef,
    'language': 'Korean'  # Vanna 응답 언어도 한국어
})

vn.train(ddl="CREATE TABLE 고객 (...)")
sql = vn.generate_sql("상위 10명의 고객을 보여줘")
```

---

### 가이드 4: 타입별 검색 개수 조정

**상황**: DDL은 많이, SQL 예시는 적게

#### Step 1: 설정
```python
vn = MyVanna(config={
    'api_key': 'sk-...',
    'n_results_sql': 3,  # SQL 예시 3개만
    'n_results_ddl': 10,  # DDL은 10개
    'n_results_documentation': 5  # 문서 5개
})

# 또는 기본값 설정 후 개별 override
vn = MyVanna(config={
    'n_results': 5,  # 기본 5개
    'n_results_ddl': 15  # DDL만 15개
})
```

#### Step 2: 런타임 조정
```python
# 설정 변경
vn.n_results_sql = 2  # Few-shot 예시 줄이기
vn.n_results_ddl = 20  # DDL 늘리기

sql = vn.generate_sql("...")
```

---

### 가이드 5: Collection 관리

**상황**: 학습 데이터 확인 및 삭제

#### Step 1: 전체 데이터 조회
```python
# 모든 학습 데이터 조회
df = vn.get_training_data()
print(df.head())
#    id                  question         content              training_data_type
# 0  abc-sql            top customers    SELECT * FROM ...    sql
# 1  def-ddl            None             CREATE TABLE ...     ddl
# 2  ghi-doc            None             VIP = sales > 1000   documentation

# 필터링
sql_data = df[df['training_data_type'] == 'sql']
print(f"Total SQL examples: {len(sql_data)}")
```

#### Step 2: 개별 데이터 삭제
```python
# ID로 삭제
vn.remove_training_data("abc123-sql")

# 조건부 삭제
df = vn.get_training_data()
old_sqls = df[df['training_data_type'] == 'sql'].head(10)['id']

for id in old_sqls:
    vn.remove_training_data(id)
```

#### Step 3: Collection 전체 리셋
```python
# SQL 예시 전부 삭제
vn.remove_collection("sql")

# DDL 전부 삭제
vn.remove_collection("ddl")

# Documentation 전부 삭제
vn.remove_collection("documentation")
```

---

### 가이드 6: Remote ChromaDB Server

**상황**: 팀 공유 Vector DB

#### Step 1: Server 실행
```bash
# Docker로 ChromaDB 서버 실행
docker run -p 8000:8000 chromadb/chroma

# 또는 Python
chroma run --host 0.0.0.0 --port 8000
```

#### Step 2: Client 연결
```python
import chromadb

# HTTP Client 생성
remote_client = chromadb.HttpClient(
    host="remote-server.com",
    port=8000
)

# Vanna에 전달
vn = MyVanna(config={
    'api_key': 'sk-...',
    'client': remote_client  # 커스텀 client!
})

# 사용은 동일
vn.train(ddl="...")
```

---

## ⭐ 안티패턴과 흔한 실수

### 실수 1: 같은 데이터 여러 번 추가 (UUID 이해 안 함)

**❌ 나쁜 예:**
```python
# 루프에서 같은 DDL 반복 추가
for i in range(10):
    vn.train(ddl="CREATE TABLE users (...)")
# → 10번 호출해도 ID 동일 → 1번만 저장됨 (의도와 다를 수 있음)
```

**문제:**
- Deterministic UUID → 같은 내용 = 같은 ID
- 사용자는 10번 추가했다고 생각하지만 실제로는 1번

**✅ 좋은 예:**
```python
# 1번만 호출
vn.train(ddl="CREATE TABLE users (...)")

# 또는 ID 직접 확인
id1 = vn.train(ddl="CREATE TABLE users (...)")
id2 = vn.train(ddl="CREATE TABLE users (...)")
print(f"Same ID? {id1 == id2}")  # True
```

---

### 실수 2: In-memory 모드로 프로덕션 배포

**❌ 나쁜 예:**
```python
# Production
vn = MyVanna(config={
    'client': 'in-memory'  # ❌ 서버 재시작 시 데이터 사라짐!
})

# 학습 데이터 추가
vn.train(ddl="...")  # 100개 추가

# 서버 재시작
# → 모든 학습 데이터 사라짐!
```

**문제:**
- In-memory는 메모리에만 저장
- 재시작/크래시 시 데이터 손실

**✅ 좋은 예:**
```python
# Production: Persistent
vn = MyVanna(config={
    'path': '/var/lib/vanna/chromadb'  # 영속 저장
})

# 또는 Remote Server
remote_client = chromadb.HttpClient(host="chromadb-server")
vn = MyVanna(config={'client': remote_client})
```

---

### 실수 3: Collection 이름 충돌

**❌ 나쁜 예:**
```python
# 여러 프로젝트가 같은 path 사용
vn_project_a = MyVanna(config={'path': './chromadb'})  # sql, ddl, doc collection
vn_project_b = MyVanna(config={'path': './chromadb'})  # ❌ 같은 collection!

vn_project_a.train(ddl="CREATE TABLE a (...)")
vn_project_b.train(ddl="CREATE TABLE b (...)")
# → 두 프로젝트 데이터가 섞임!
```

**문제:**
- Collection 이름이 하드코딩 ("sql", "ddl", "documentation")
- 같은 path 사용 시 데이터 충돌

**✅ 좋은 예:**
```python
# 방법 1: Path 분리
vn_project_a = MyVanna(config={'path': './chromadb_project_a'})
vn_project_b = MyVanna(config={'path': './chromadb_project_b'})

# 방법 2: Collection metadata로 구분
vn_project_a = MyVanna(config={
    'path': './chromadb',
    'collection_metadata': {'project': 'a'}
})

# 방법 3: 커스텀 collection 이름 (고급, 코드 수정 필요)
class CustomChromaDB(ChromaDB_VectorStore):
    def __init__(self, config, prefix=""):
        self.prefix = prefix
        super().__init__(config)

    def __init__(self, config):
        # ...
        self.sql_collection = self.chroma_client.get_or_create_collection(
            name=f"{self.prefix}_sql"
        )
```

---

### 실수 4: Embedding 함수 변경 시 기존 데이터 무시

**❌ 나쁜 예:**
```python
# 초기: OpenAI 임베딩
vn = MyVanna(config={'path': './chromadb'})
vn.train(ddl="...")  # OpenAI 임베딩으로 저장

# 나중에: HuggingFace 임베딩으로 변경
vn = MyVanna(config={
    'path': './chromadb',  # 같은 path!
    'embedding_function': sentence_transformer_ef  # ❌ 다른 함수!
})

# 검색 시 문제
results = vn.get_related_ddl("...")
# → 검색 안 됨! (쿼리는 HF 임베딩, 저장된 데이터는 OpenAI 임베딩)
```

**문제:**
- 임베딩 함수 변경 시 기존 데이터와 호환 안 됨
- Vector space가 다름

**✅ 좋은 예:**
```python
# 방법 1: 새로운 path 사용
vn = MyVanna(config={
    'path': './chromadb_v2',  # 새로운 경로
    'embedding_function': sentence_transformer_ef
})
# 데이터 다시 학습 필요

# 방법 2: 마이그레이션 스크립트
def migrate_embeddings(old_vn, new_vn):
    df = old_vn.get_training_data()

    for _, row in df.iterrows():
        if row['training_data_type'] == 'sql':
            data = json.loads(row['content'])
            new_vn.train(question=data['question'], sql=data['sql'])
        elif row['training_data_type'] == 'ddl':
            new_vn.train(ddl=row['content'])
        # ...
```

---

### 실수 5: `get_training_data()` 결과를 수정 후 저장 안 함

**❌ 나쁜 예:**
```python
# 데이터 조회
df = vn.get_training_data()

# 수정
df.loc[0, 'content'] = "SELECT * FROM customers_v2"  # ❌ DataFrame만 수정

# 저장 안 함
# → 실제 ChromaDB에는 반영 안 됨!
```

**문제:**
- `get_training_data()`는 스냅샷 반환
- DataFrame 수정해도 ChromaDB에 자동 반영 안 됨

**✅ 좋은 예:**
```python
# 데이터 조회
df = vn.get_training_data()

# 수정하려면: 삭제 후 재추가
old_id = df.loc[0, 'id']
new_content = "SELECT * FROM customers_v2"

vn.remove_training_data(old_id)
vn.train(sql=new_content)  # 질문 자동 생성
```

---

### 실수 6: Large Batch Insert 시 메모리 부족

**❌ 나쁜 예:**
```python
# 10000개 DDL을 한 번에
ddls = [get_ddl(table) for table in all_tables]  # 10000개

for ddl in ddls:
    vn.train(ddl=ddl)  # ❌ 메모리 폭발 (임베딩 생성)
```

**문제:**
- 임베딩 함수가 메모리 사용
- 대량 데이터 시 OOM

**✅ 좋은 예:**
```python
# 방법 1: Batch 크기 제한
batch_size = 100

for i in range(0, len(ddls), batch_size):
    batch = ddls[i:i+batch_size]
    for ddl in batch:
        vn.train(ddl=ddl)

    # 주기적으로 메모리 정리
    if i % 500 == 0:
        import gc
        gc.collect()
        print(f"Progress: {i}/{len(ddls)}")

# 방법 2: Training Plan 사용 (Vanna가 알아서 처리)
plan = vn.get_training_plan_snowflake()
vn.train(plan=plan)
```

---

### 실수 7: Collection 삭제 후 참조 사용

**❌ 나쁜 예:**
```python
# Collection 삭제
vn.remove_collection("sql")

# ❌ 하지만 vn.sql_collection 참조는 여전히 옛날 객체!
results = vn.sql_collection.query(...)  # 에러 또는 빈 결과
```

**문제:**
- `remove_collection()`은 `self.sql_collection` 재할당함
- 하지만 직접 참조 저장했으면 옛날 객체

**✅ 좋은 예:**
```python
# 항상 vn 통해서 접근
vn.remove_collection("sql")

# VannaBase 메서드 사용
results = vn.get_similar_question_sql("...")  # ✅ 새로운 collection 사용

# 또는 재초기화
vn = MyVanna(config={...})
```

---

## ⭐ 스케일 고려사항

### 소규모 (< 1000 documents)

**권장 사항:**
- ✅ Persistent 로컬 저장
- ✅ 기본 임베딩 함수 OK
- ✅ n_results 기본값 (10)

**구현 예시:**
```python
vn = MyVanna(config={
    'path': './vanna_db',
    'api_key': 'sk-...'
})
```

---

### 중규모 (1000-10000 documents)

**권장 사항:**
- ✅ 타입별 n_results 최적화
- ✅ 경량 임베딩 함수 고려 (Sentence Transformers)
- ✅ 검색 성능 모니터링

**구현 예시:**
```python
from chromadb.utils import embedding_functions

vn = MyVanna(config={
    'path': './vanna_db',
    'embedding_function': embedding_functions.SentenceTransformerEmbeddingFunction(
        model_name="all-MiniLM-L6-v2"  # 빠르고 가벼움
    ),
    'n_results_sql': 5,  # Few-shot 줄이기
    'n_results_ddl': 15,  # DDL은 넉넉히
    'n_results_documentation': 5
})
```

---

### 대규모 (10000+ documents)

**권장 사항:**
- ✅ Remote ChromaDB Server (분산)
- ✅ 검색 캐싱
- ✅ Collection 분할 (프로젝트별, 스키마별)
- ✅ 메트릭 수집

**구현 예시:**
```python
import chromadb
from functools import lru_cache

# Remote server
remote_client = chromadb.HttpClient(
    host="chromadb-cluster.internal",
    port=8000
)

class ScalableVanna(ChromaDB_VectorStore):
    @lru_cache(maxsize=500)
    def get_similar_question_sql(self, question: str) -> list:
        # 캐싱 (같은 질문 반복 시 빠름)
        return super().get_similar_question_sql(question)

vn = ScalableVanna(config={
    'client': remote_client,
    'n_results_sql': 3,  # 최소화
    'n_results_ddl': 10,
    'n_results_documentation': 3
})

# 검색 성능 모니터링
import time
from prometheus_client import Histogram

search_duration = Histogram('chromadb_search_seconds', 'Search duration')

class MonitoredVanna(ScalableVanna):
    def get_similar_question_sql(self, question: str) -> list:
        with search_duration.time():
            return super().get_similar_question_sql(question)
```

---

## 💡 배운 점

### 1. Collection 분리는 타입별 최적화
**핵심 개념**: SQL/DDL/Doc을 별도 Collection에 → 타입별 n_results 조절
**언제 사용?**: 다양한 타입 데이터를 Vector DB에 저장
**적용 가능한 곳**:
- 문서 검색 (제목/본문/메타데이터 분리)
- 멀티모달 검색 (텍스트/이미지 분리)

### 2. Deterministic UUID는 자동 중복 제거
**핵심 개념**: 내용의 해시 = ID → 같은 내용 = 같은 ID
**언제 사용?**: 중복 데이터 방지, 검색 없이 빠른 추가
**적용 가능한 곳**:
- 파일 저장 (Content-addressable storage)
- 캐시 키 생성

### 3. Config 파라미터로 다양한 모드 지원
**핵심 개념**: Persistent/In-memory/Remote를 config로 선택
**언제 사용?**: 개발/테스트/프로덕션 모드 분리
**적용 가능한 곳**:
- 데이터베이스 연결 (로컬/스테이징/프로덕션)
- 로깅 레벨 (디버그/정보/에러)

### 4. Static Helper로 공통 로직 추출
**핵심 개념**: `_extract_documents` Static 메서드로 파싱 로직 재사용
**언제 사용?**: 여러 메서드에서 동일한 후처리
**적용 가능한 곳**:
- JSON 파싱
- 응답 정규화

### 5. Collection 삭제 & 재생성이 빠른 리셋
**핵심 개념**: 데이터 하나씩 삭제보다 Collection 통째로 리셋
**언제 사용?**: 대량 데이터 삭제
**적용 가능한 곳**:
- 테스트 픽스처 초기화
- 캐시 flush

---

## 📊 요약

| 항목 | 내용 |
|------|------|
| **핵심 문제** | VannaBase 추상 메서드를 ChromaDB로 구현 |
| **핵심 패턴** | 3-Collection 분리 + Deterministic UUID + Config Factory |
| **주요 트레이드오프** | Collection 3개 관리 vs 타입별 최적화 |
| **핵심 기법** | 1) SQL/DDL/Doc Collection 분리<br>2) 내용 해시로 UUID 생성<br>3) Persistent/In-memory 모드<br>4) Static helper로 파싱 재사용 |
| **적용 시 주의** | 1) In-memory는 테스트만<br>2) Path 분리 (프로젝트별)<br>3) Embedding 함수 변경 시 재학습<br>4) Large batch는 분할 |
| **스케일 전략** | 소규모: 로컬 Persistent<br>중규모: 경량 임베딩<br>대규모: Remote Server + 캐싱 |
| **실무 적용** | 로컬 RAG 시스템, Serverless Vector DB |

---

**✅ chromadb_vector.py 분석 완료!**

다음: `openai/openai_chat.py` 분석
