# vanna/__init__.py 분석

## 파일 개요

**경로**: `src/vanna/__init__.py`
**라인 수**: 399줄
**목적**: 패키지 엔트리 포인트 + API 마이그레이션 + Vanna.AI Cloud 인증

이 파일은 일반적인 `__init__.py`와 다릅니다. **대부분의 함수가 deprecated**이며, 새로운 API로 마이그레이션을 유도합니다. 살아있는 코드는 `get_api_key()`와 `TrainingPlan` 클래스뿐입니다. 이 파일을 통해 대규모 API 마이그레이션 전략을 배울 수 있습니다.

---

## 핵심 문제 해결

### 1. 레거시 API 마이그레이션 전략

#### 문제
Vanna는 초기에 global 함수 방식으로 설계되었습니다. 하지만 Mixin 패턴으로 재설계하면서 API가 변경되었습니다. 기존 사용자 코드를 어떻게 처리할까?

**레거시 API (Deprecated)**:
```python
import vanna as vn

vn.set_api_key("...")
vn.set_model("my-model")
vn.add_ddl("CREATE TABLE users (...)")
sql = vn.generate_sql("Show me all users")
```

**새로운 API**:
```python
from vanna.remote import VannaDefault

vn = VannaDefault(model="my-model", api_key="...")
vn.add_ddl("CREATE TABLE users (...)")
sql = vn.generate_sql("Show me all users")
```

#### 고민한 대안들

**Option A: 조용히 deprecated (Warning만)**
```python
def set_api_key(key: str) -> None:
    warnings.warn("set_api_key is deprecated. Use VannaDefault instead.", DeprecationWarning)
    # 하지만 여전히 동작
    global api_key
    api_key = key
```
- 장점: 기존 코드 동작 유지
- 단점: 사용자가 무시할 가능성, 기술 부채 누적

**Option B: 즉시 에러 (Breaking Change)** ✅
```python
def error_deprecation():
    raise Exception("""
Please switch to the following method for initializing Vanna:

from vanna.remote import VannaDefault

api_key = # Your API key from https://vanna.ai/account/profile
vanna_model_name = # Your model name from https://vanna.ai/account/profile

vn = VannaDefault(model=vanna_model_name, api_key=api_key)
""")

def set_api_key(key: str) -> None:
    error_deprecation()

def generate_sql(question: str) -> str:
    error_deprecation()

# 모든 레거시 함수가 동일
```
- 장점: 명확한 마이그레이션 강제, 빠른 기술 부채 제거
- 단점: 기존 코드 즉시 중단

**Option C: 버전별 점진적 deprecation**
```python
# v0.1.x: 정상 동작
# v0.2.x: Warning 추가
# v0.3.x: Error 발생
```
- 장점: 사용자에게 충분한 시간
- 단점: 마이그레이션 기간 길어짐

#### 최종 선택: Option B

**선택 이유**:
1. **명확성**: 사용자가 즉시 알 수 있음
2. **가이드**: 에러 메시지에 마이그레이션 방법 포함
3. **클린 코드베이스**: 레거시 코드 유지 불필요

**구현 코드** (__init__.py:46-56):
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

def add_sql(question: str, sql: str, tag: Union[str, None] = "Manually Trained") -> bool:
    error_deprecation()

def generate_sql(question: str) -> str:
    error_deprecation()

# ... 30개 이상의 함수가 동일
```

**Deprecated 함수 목록**:
- `set_api_key()`, `set_model()`
- `add_sql()`, `add_ddl()`, `add_documentation()`
- `generate_sql()`, `ask()`
- `connect_to_snowflake()`, `connect_to_postgres()`, ...
- `train()`, `get_training_plan_postgres()`, ...

**마이그레이션 예시**:
```python
# OLD (에러 발생)
import vanna as vn
vn.set_api_key("...")
vn.set_model("my-model")

# NEW
from vanna.remote import VannaDefault
vn = VannaDefault(model="my-model", api_key="...")
```

---

### 2. OTP 인증 시스템

#### 문제
Vanna.AI Cloud API 사용을 위한 인증이 필요합니다. 비밀번호 저장 없이 안전하게 인증하려면?

#### 고민한 대안들

**Option A: Email + Password**
```python
vn.login(email="user@example.com", password="...")
```
- 장점: 전통적 방식
- 단점: 비밀번호 저장 필요, 보안 위험

**Option B: OAuth (Google, GitHub)**
```python
vn.login_with_google()  # Browser redirect
```
- 장점: 안전, 편리
- 단점: 브라우저 필요 (CLI에서 불편)

**Option C: OTP (One-Time Password)** ✅
```python
api_key = vn.get_api_key(email="user@example.com")
# Email로 OTP 전송
# 터미널에서 OTP 입력
# API Key 발급
```
- 장점: 비밀번호 저장 불필요, CLI 친화적, 보안
- 단점: 매번 OTP 입력 (하지만 API Key 캐싱으로 해결)

#### 최종 선택: Option C

**선택 이유**:
1. **보안**: 비밀번호 저장 불필요
2. **CLI 친화적**: Browser redirect 없음
3. **간편**: 이메일만 있으면 시작 가능

**구현 코드** (__init__.py:75-130):
```python
def get_api_key(email: str, otp_code: Union[str, None] = None) -> str:
    # 1. 환경 변수 체크 (재인증 불필요)
    vanna_api_key = os.environ.get("VANNA_API_KEY", None)
    if vanna_api_key is not None:
        return vanna_api_key

    # 2. Email 검증
    if email == "my-email@example.com":
        raise ValidationError(
            "Please replace 'my-email@example.com' with your email address."
        )

    # 3. OTP 전송 (otp_code 없으면)
    if otp_code is None:
        params = [UserEmail(email=email)]
        d = __unauthenticated_rpc_call(method="send_otp", params=params)

        if "result" not in d:
            raise OTPCodeError("Error sending OTP code.")

        status = Status(**d["result"])
        if not status.success:
            raise OTPCodeError(f"Error sending OTP code: {status.message}")

        # 4. 사용자 입력 대기
        otp_code = input("Check your email for the code and enter it here: ")

    # 5. OTP 검증
    params = [UserOTP(email=email, otp=otp_code)]
    d = __unauthenticated_rpc_call(method="verify_otp", params=params)

    if "result" not in d:
        raise OTPCodeError("Error verifying OTP code.")

    key = ApiKey(**d["result"])
    if key is None:
        raise OTPCodeError("Error verifying OTP code.")

    return key.key
```

**인증 흐름**:
```
사용자                    Vanna.AI Server
  |                             |
  |-- send_otp(email) ---------->|
  |<-- OTP to Email -------------|
  |                             |
[Terminal: OTP 입력]
  |                             |
  |-- verify_otp(email, otp) -->|
  |<-- API Key -----------------|
  |                             |
[환경 변수에 저장]
```

**사용 예시**:
```python
# 첫 실행 (OTP 입력 필요)
api_key = vn.get_api_key(email="user@example.com")
# Check your email for the code and enter it here: 123456

# 환경 변수 저장
import os
os.environ["VANNA_API_KEY"] = api_key

# 이후 실행 (재인증 불필요)
vn = VannaDefault(api_key=api_key, model="my-model")
```

**프로덕션 사용**:
```bash
# .env 파일
VANNA_API_KEY=vn_abc123...

# 코드에서
from dotenv import load_dotenv
load_dotenv()

api_key = vn.get_api_key(email="user@example.com")  # 자동으로 env에서 로드
vn = VannaDefault(api_key=api_key, model="my-model")
```

---

### 3. TrainingPlan 설계

#### 문제
데이터베이스에 수백 개의 테이블, 수천 개의 컬럼이 있습니다. 모든 것을 자동으로 학습하면 비용과 시간이 많이 듭니다. 사용자가 preview하고 선택적으로 학습하려면?

#### 고민한 대안들

**Option A: 전체 자동 학습 (No Preview)**
```python
vn.train_all_tables()  # 모든 테이블 DDL 자동 학습
```
- 장점: 간단
- 단점: 불필요한 테이블 학습 → 비용 증가, 정확도 저하

**Option B: 수동 선택**
```python
vn.train(ddl="CREATE TABLE users (...)")
vn.train(ddl="CREATE TABLE orders (...)")
# 100개 테이블 → 100줄 코드
```
- 장점: 완전한 제어
- 단점: 반복적, 지루함

**Option C: TrainingPlan (Preview + Modify + Execute)** ✅
```python
# 1. Plan 생성 (DB 메타데이터 스캔)
plan = vn.get_training_plan_postgres()

# 2. Preview
print(plan.get_summary())
# Train on DDL: public users
# Train on DDL: public orders
# Train on DDL: internal audit_logs  ← 제거하고 싶음

# 3. 불필요한 항목 제거
plan.remove_item("Train on DDL: internal audit_logs")

# 4. 실행
vn.train(plan=plan)
```
- 장점: 자동 스캔 + 수동 필터링
- 단점: 추가 구현 복잡도

#### 최종 선택: Option C

**선택 이유**:
1. **자동화**: DB 메타데이터 자동 스캔
2. **제어**: 사용자가 불필요한 항목 제거 가능
3. **투명성**: 학습 전에 무엇을 학습할지 명확히 알 수 있음

**구현 코드** (__init__.py:171-250):
```python
@dataclass
class TrainingPlanItem:
    item_type: str      # "sql", "ddl", "is"
    item_group: str     # Schema 또는 Database
    item_name: str      # Table 또는 Query 이름
    item_value: str     # 실제 DDL 또는 SQL 내용

    def __str__(self):
        if self.item_type == self.ITEM_TYPE_SQL:
            return f"Train on SQL: {self.item_group} {self.item_name}"
        elif self.item_type == self.ITEM_TYPE_DDL:
            return f"Train on DDL: {self.item_group} {self.item_name}"
        elif self.item_type == self.ITEM_TYPE_IS:
            return f"Train on Information Schema: {self.item_group} {self.item_name}"

    ITEM_TYPE_SQL = "sql"
    ITEM_TYPE_DDL = "ddl"
    ITEM_TYPE_IS = "is"


class TrainingPlan:
    """
    A class representing a training plan. You can see what's in it, and remove items from it that you don't want trained.
    """

    _plan: List[TrainingPlanItem]

    def __init__(self, plan: List[TrainingPlanItem]):
        self._plan = plan

    def __str__(self):
        return "\n".join(self.get_summary())

    def get_summary(self) -> List[str]:
        """Get a summary of the training plan."""
        return [f"{item}" for item in self._plan]

    def remove_item(self, item: str):
        """Remove an item from the training plan."""
        for plan_item in self._plan:
            if str(plan_item) == item:
                self._plan.remove(plan_item)
                break
```

**TrainingPlanItem 타입**:
- `ITEM_TYPE_DDL`: 테이블 스키마 (CREATE TABLE ...)
- `ITEM_TYPE_SQL`: 과거 쿼리 (historical queries)
- `ITEM_TYPE_IS`: Information Schema 메타데이터

**사용 예시**:
```python
from vanna.remote import VannaDefault

vn = VannaDefault(model="my-model", api_key="...")

# 1. Postgres DB 연결
vn.connect_to_postgres(
    host="localhost",
    dbname="mydb",
    user="user",
    password="pass"
)

# 2. Training Plan 생성
plan = vn.get_training_plan_postgres(
    filter_schemas=["public"],  # Only public schema
    include_information_schema=False,
    use_historical_queries=True  # pg_stat_statements에서 과거 쿼리 가져오기
)

# 3. Preview
for item in plan.get_summary():
    print(item)

# Output:
# Train on DDL: public users
# Train on DDL: public orders
# Train on DDL: public products
# Train on SQL: SELECT * FROM users WHERE ...
# Train on SQL: SELECT COUNT(*) FROM orders WHERE ...

# 4. 불필요한 항목 제거
plan.remove_item("Train on DDL: public temp_table")

# 5. 실행
vn.train(plan=plan)
```

**구현 위치**:
- `__init__.py`에는 **인터페이스만** 정의
- 실제 구현은 `base.py`의 `get_training_plan_generic()` 등
- 하지만 이 파일에서도 deprecated! (VannaDefault에서 구현)

---

### 4. Unauthenticated RPC 패턴

#### 문제
`get_api_key()` 함수는 인증 전에 호출됩니다. 어떻게 인증 없이 Vanna.AI API를 호출할까?

#### 고민한 대안들

**Option A: Public API 엔드포인트 (인증 불필요)**
```python
response = requests.post("https://vanna.ai/public/send_otp", json={...})
```
- 장점: 간단
- 단점: 여러 엔드포인트 관리 필요

**Option B: Unauthenticated RPC** ✅
```python
def __unauthenticated_rpc_call(method, params):
    data = {"method": method, "params": [dataclass_to_dict(obj) for obj in params]}
    response = requests.post(_unauthenticated_endpoint, json=data)
    return response.json()

# 사용
__unauthenticated_rpc_call("send_otp", [UserEmail(email="...")])
__unauthenticated_rpc_call("verify_otp", [UserOTP(email="...", otp="...")])
```
- 장점: 단일 엔드포인트, RPC 패턴
- 단점: RPC 구현 필요

#### 최종 선택: Option B

**선택 이유**:
- 단일 엔드포인트 (`_unauthenticated_endpoint`)
- 메서드 추가가 쉬움 (서버만 수정)
- JSON-RPC와 유사한 패턴

**구현 코드** (__init__.py:58-67):
```python
_unauthenticated_endpoint = "https://ask.vanna.ai/unauthenticated_rpc"

def __unauthenticated_rpc_call(method, params):
    headers = {"Content-Type": "application/json"}
    data = {
        "method": method,
        "params": [__dataclass_to_dict(obj) for obj in params]
    }

    response = requests.post(
        _unauthenticated_endpoint,
        headers=headers,
        data=json.dumps(data)
    )
    return response.json()

def __dataclass_to_dict(obj):
    return dataclasses.asdict(obj)
```

**RPC 메서드**:
- `send_otp`: Email로 OTP 전송
- `verify_otp`: OTP 검증 → API Key 발급

**데이터 타입** (types/__init__.py):
```python
@dataclass
class UserEmail:
    email: str

@dataclass
class UserOTP:
    email: str
    otp: str

@dataclass
class ApiKey:
    key: str

@dataclass
class Status:
    success: bool
    message: str
```

**요청 예시**:
```json
// send_otp
{
  "method": "send_otp",
  "params": [
    {"email": "user@example.com"}
  ]
}

// verify_otp
{
  "method": "verify_otp",
  "params": [
    {"email": "user@example.com", "otp": "123456"}
  ]
}
```

**응답 예시**:
```json
// send_otp 응답
{
  "result": {
    "success": true,
    "message": "OTP sent successfully"
  }
}

// verify_otp 응답
{
  "result": {
    "key": "vn_abc123..."
  }
}
```

---

### 5. Global 변수 vs 인스턴스

#### 문제
레거시 API는 global 변수를 사용했습니다:
```python
import vanna as vn
vn.api_key = "..."
vn.run_sql = lambda sql: pd.read_sql(sql, engine)
```

새 API는 인스턴스 기반입니다:
```python
vn = VannaDefault(api_key="...")
vn.run_sql = lambda sql: pd.read_sql(sql, engine)
```

둘 다 지원해야 할까?

#### 고민한 대안들

**Option A: Global만 지원 (레거시)**
```python
# __init__.py
api_key: Union[str, None] = None
run_sql: Union[Callable, None] = None
```
- 장점: 간단
- 단점: 여러 인스턴스 사용 불가

**Option B: 인스턴스만 지원 (새 API)**
```python
class VannaDefault:
    def __init__(self):
        self.api_key = None
        self.run_sql = None
```
- 장점: OOP, 여러 인스턴스 가능
- 단점: 레거시 코드 중단

**Option C: 둘 다 지원** ✅
```python
# __init__.py - Global 변수 (호환성)
api_key: Union[str, None] = None
run_sql: Union[Callable[[str], pd.DataFrame], None] = None
fig_as_img: bool = False

# 하지만 권장하지 않음 (deprecated)
```
- 장점: 호환성
- 단점: 혼란 가능성

#### 최종 선택: Option C (하지만 Deprecated)

**구현 코드** (__init__.py:24-40):
```python
api_key: Union[str, None] = None  # API key for Vanna.AI

fig_as_img: bool = False  # Whether or not to return Plotly figures as images

run_sql: Union[Callable[[str], pd.DataFrame], None] = None
"""
**Example**
```python
vn.run_sql = lambda sql: pd.read_sql(sql, engine)
```

Set the SQL to DataFrame function for Vanna.AI. This is used in the [`vn.ask(...)`][vanna.ask] function.
Instead of setting this directly you can also use [`vn.connect_to_snowflake(...)`][vanna.connect_to_snowflake] to set this.
"""

__org: Union[str, None] = None  # Organization name for Vanna.AI
```

**주의**: 모든 함수가 `error_deprecation()`을 호출하므로, 이 global 변수들은 실제로 사용되지 않습니다!

**새 API에서**:
```python
class VannaDefault:
    def __init__(self, model, api_key):
        self.api_key = api_key
        self.model = model
        # 인스턴스 변수 사용
```

---

## 실전 활용 가이드

### 1. Vanna.AI Cloud 사용 (OTP 인증)

```python
from vanna.remote import VannaDefault
import vanna as vn

# 1. API Key 발급 (첫 실행)
api_key = vn.get_api_key(email="user@example.com")
# Check your email for the code and enter it here: 123456

# 2. 환경 변수 저장 (선택)
import os
os.environ["VANNA_API_KEY"] = api_key

# 3. Vanna 초기화
vanna_instance = VannaDefault(
    model="my-sales-model",
    api_key=api_key
)

# 4. DB 연결
vanna_instance.connect_to_postgres(
    host="localhost",
    dbname="sales_db",
    user="user",
    password="pass"
)

# 5. 학습
vanna_instance.train(ddl="CREATE TABLE products (id INT, name TEXT, price FLOAT)")

# 6. SQL 생성
sql = vanna_instance.generate_sql("지난달 매출 상위 10개 상품은?")
```

### 2. TrainingPlan으로 대량 학습

```python
from vanna.remote import VannaDefault

vn = VannaDefault(model="my-model", api_key="...")

# 1. DB 연결
vn.connect_to_postgres(host="localhost", dbname="mydb", user="user", password="pass")

# 2. Training Plan 생성
plan = vn.get_training_plan_postgres(
    filter_schemas=["public", "analytics"],  # 특정 스키마만
    include_information_schema=False,
    use_historical_queries=True  # pg_stat_statements 활용
)

# 3. Preview (Jupyter Notebook에서 유용)
import pandas as pd
summary = plan.get_summary()
df = pd.DataFrame([
    {"item": item} for item in summary
])
print(df)

# 4. 필터링 (불필요한 테이블 제거)
for item in summary:
    if "temp_" in item or "audit_" in item:
        plan.remove_item(item)

# 5. 실행
vn.train(plan=plan)

# 6. 확인
training_data = vn.get_training_data()
print(f"Total training items: {len(training_data)}")
```

### 3. 환경 변수로 API Key 관리

```python
# .env 파일
VANNA_API_KEY=vn_abc123xyz...
VANNA_MODEL_NAME=my-sales-model

# 코드
from dotenv import load_dotenv
import os
from vanna.remote import VannaDefault

load_dotenv()

vn = VannaDefault(
    model=os.getenv("VANNA_MODEL_NAME"),
    api_key=os.getenv("VANNA_API_KEY")
)

# 또는 get_api_key()가 자동으로 env 체크
api_key = vn.get_api_key(email="user@example.com")  # VANNA_API_KEY 있으면 재인증 불필요
```

### 4. Local vs Cloud 선택

```python
# Cloud (Vanna.AI 서버에서 Vector Store + LLM)
from vanna.remote import VannaDefault
vn = VannaDefault(model="my-model", api_key="...")

# Local (내 서버에서 ChromaDB + OpenAI)
from vanna.chromadb import ChromaDB_VectorStore
from vanna.openai import OpenAI_Chat

class MyVanna(ChromaDB_VectorStore, OpenAI_Chat):
    def __init__(self, config=None):
        ChromaDB_VectorStore.__init__(self, config=config)
        OpenAI_Chat.__init__(self, config=config)

vn = MyVanna(config={
    "api_key": "sk-...",  # OpenAI API Key
    "model": "gpt-3.5-turbo"
})
```

### 5. 레거시 코드 마이그레이션

```python
# OLD (에러 발생)
import vanna as vn

vn.set_api_key("vn_abc123...")
vn.set_model("my-model")
vn.add_ddl("CREATE TABLE users (...)")
sql = vn.generate_sql("Show me all users")

# NEW (정상 동작)
from vanna.remote import VannaDefault

vn = VannaDefault(model="my-model", api_key="vn_abc123...")
vn.train(ddl="CREATE TABLE users (...)")
sql = vn.generate_sql("Show me all users")
```

**마이그레이션 체크리스트**:
- [ ] `import vanna as vn` → `from vanna.remote import VannaDefault`
- [ ] `vn.set_api_key()` + `vn.set_model()` → `VannaDefault(model=..., api_key=...)`
- [ ] `vn.add_ddl()` → `vn.train(ddl=...)`
- [ ] `vn.add_sql()` → `vn.train(question=..., sql=...)`
- [ ] `vn.connect_to_postgres()` → `vn.connect_to_postgres()`

---

## Anti-Patterns (하지 말아야 할 것들)

### ❌ 1. 레거시 API 사용

```python
# WRONG: Deprecated API
import vanna as vn
vn.set_api_key("...")
vn.generate_sql("...")  # Exception 발생!

# CORRECT: 새 API
from vanna.remote import VannaDefault
vn = VannaDefault(model="...", api_key="...")
vn.generate_sql("...")
```

**이유**: 모든 레거시 함수가 `error_deprecation()` 호출 → 즉시 에러

### ❌ 2. API Key 하드코딩

```python
# WRONG: 코드에 직접
vn = VannaDefault(model="my-model", api_key="vn_abc123xyz...")

# CORRECT: 환경 변수
import os
vn = VannaDefault(
    model=os.getenv("VANNA_MODEL_NAME"),
    api_key=os.getenv("VANNA_API_KEY")
)
```

**이유**:
- Git에 실수로 커밋 방지
- 환경별 키 분리 (dev/staging/prod)

### ❌ 3. OTP를 스크립트에서 자동화

```python
# WRONG: OTP를 코드로 자동 입력 시도
import time
api_key = vn.get_api_key(email="user@example.com", otp_code="123456")  # 실패 가능

# CORRECT: 첫 실행은 수동, 이후 API Key 재사용
# 1. 첫 실행 (수동)
api_key = vn.get_api_key(email="user@example.com")
# [터미널에서 OTP 입력]

# 2. API Key 저장
with open(".vanna_key", "w") as f:
    f.write(api_key)

# 3. 이후 실행 (자동)
with open(".vanna_key") as f:
    api_key = f.read()
vn = VannaDefault(model="my-model", api_key=api_key)
```

**이유**: OTP는 일회용이므로 재사용 불가. API Key를 캐싱해야 함.

### ❌ 4. TrainingPlan 없이 전체 학습

```python
# WRONG: 모든 테이블 무작위 학습
for table in all_tables:
    ddl = get_ddl(table)
    vn.train(ddl=ddl)

# CORRECT: TrainingPlan으로 preview 후 선택
plan = vn.get_training_plan_postgres()
print(plan.get_summary())  # 무엇을 학습할지 확인

# 불필요한 테이블 제거
plan.remove_item("Train on DDL: temp_table")
vn.train(plan=plan)
```

**이유**:
- Temp 테이블, Audit 테이블 등 불필요한 데이터 학습 방지
- Vector Store 크기 감소 → 검색 속도 향상
- 비용 절감 (불필요한 임베딩 생성 방지)

### ❌ 5. Email 검증 무시

```python
# WRONG: Example email 그대로 사용
api_key = vn.get_api_key(email="my-email@example.com")
# ValidationError: Please replace 'my-email@example.com' with your email address.

# CORRECT: 실제 이메일 사용
api_key = vn.get_api_key(email="user@company.com")
```

**이유**: `get_api_key()`는 명시적으로 example email을 거부합니다 (__init__.py:96-99).

### ❌ 6. Global 변수 사용 (혼란)

```python
# WRONG: Global 변수와 인스턴스 혼용
import vanna as vn
from vanna.remote import VannaDefault

vn.api_key = "abc"  # Global 변수 (deprecated)
vanna_instance = VannaDefault(api_key="xyz")  # 인스턴스 변수

# 어느 것이 사용될까? → 혼란!

# CORRECT: 인스턴스만 사용
vn = VannaDefault(api_key="xyz")
```

**이유**: Global 변수는 deprecated이며, 여러 인스턴스 사용 시 충돌 가능.

---

## 스케일 고려사항

### Small Scale (< 10 테이블)

**특징**:
- 개발 환경 또는 PoC
- Training Plan 불필요 (수동 학습 충분)

**권장 설정**:
```python
vn = VannaDefault(model="my-model", api_key="...")

# 수동 학습
vn.train(ddl="CREATE TABLE users (...)")
vn.train(ddl="CREATE TABLE orders (...)")
vn.train(question="지난달 매출은?", sql="SELECT SUM(amount) FROM orders WHERE ...")
```

**모니터링**:
- 수동 확인으로 충분
- Training data 개수만 체크: `len(vn.get_training_data())`

---

### Medium Scale (10-100 테이블)

**특징**:
- 프로덕션 초기
- Training Plan 필수
- Schema 별로 분리 학습

**권장 설정**:
```python
vn = VannaDefault(model="my-model", api_key="...")

# 1. Training Plan 생성 (Schema별)
plan_public = vn.get_training_plan_postgres(
    filter_schemas=["public"],
    use_historical_queries=True
)

plan_analytics = vn.get_training_plan_postgres(
    filter_schemas=["analytics"],
    use_historical_queries=False  # 분석용 테이블은 historical query 제외
)

# 2. 필터링
for item in plan_public.get_summary():
    if "temp_" in item or "_old" in item:
        plan_public.remove_item(item)

# 3. 실행
vn.train(plan=plan_public)
vn.train(plan=plan_analytics)
```

**모니터링**:
```python
# Training data 분석
training_data = vn.get_training_data()

# Type별 개수
ddl_count = len([d for d in training_data if d["training_data_type"] == "ddl"])
sql_count = len([d for d in training_data if d["training_data_type"] == "sql"])

print(f"DDL: {ddl_count}, SQL: {sql_count}")
```

---

### Large Scale (100+ 테이블)

**특징**:
- 대규모 엔터프라이즈
- 여러 모델 분리 (domain별)
- 자동화된 Training Plan 관리

**권장 설정**:

#### 1. Domain별 모델 분리
```python
# Sales 모델
vn_sales = VannaDefault(model="sales-model", api_key="...")
plan_sales = vn_sales.get_training_plan_postgres(filter_schemas=["sales"])
vn_sales.train(plan=plan_sales)

# Analytics 모델
vn_analytics = VannaDefault(model="analytics-model", api_key="...")
plan_analytics = vn_analytics.get_training_plan_postgres(filter_schemas=["analytics"])
vn_analytics.train(plan=plan_analytics)

# HR 모델
vn_hr = VannaDefault(model="hr-model", api_key="...")
plan_hr = vn_hr.get_training_plan_postgres(filter_schemas=["hr"])
vn_hr.train(plan=plan_hr)
```

**장점**:
- 도메인별 정확도 향상
- 검색 속도 증가 (작은 Vector Store)
- 권한 분리 (HR 데이터는 HR 모델만)

#### 2. Training Plan 자동 필터링
```python
def filter_training_plan(plan: vn.TrainingPlan, exclude_patterns: List[str]) -> vn.TrainingPlan:
    """불필요한 테이블 자동 제거"""
    for item in plan.get_summary():
        for pattern in exclude_patterns:
            if pattern in item.lower():
                plan.remove_item(item)
    return plan

plan = vn.get_training_plan_postgres()
plan = filter_training_plan(plan, exclude_patterns=[
    "temp_", "_old", "_backup", "audit_", "log_"
])
vn.train(plan=plan)
```

#### 3. 증분 학습 (Incremental Training)
```python
import schedule
import time

def incremental_training():
    """매일 새로운 테이블만 학습"""
    # 1. 현재 training data
    current_data = vn.get_training_data()
    current_tables = set(d["item_name"] for d in current_data if d["training_data_type"] == "ddl")

    # 2. 새로운 Training Plan
    plan = vn.get_training_plan_postgres()

    # 3. 새로운 테이블만 필터
    for item in plan.get_summary():
        table_name = item.split(":")[-1].strip()
        if table_name in current_tables:
            plan.remove_item(item)

    # 4. 학습
    if len(plan._plan) > 0:
        print(f"Training {len(plan._plan)} new items")
        vn.train(plan=plan)

# 매일 자정 실행
schedule.every().day.at("00:00").do(incremental_training)

while True:
    schedule.run_pending()
    time.sleep(60)
```

#### 4. 비용 모니터링
```python
# Vanna.AI Cloud는 API 호출 기반 과금
# Training 비용 추정
def estimate_training_cost(plan: vn.TrainingPlan):
    num_items = len(plan._plan)
    # 가정: 1 training item = 1 API call = $0.01
    estimated_cost = num_items * 0.01
    print(f"Estimated cost: ${estimated_cost:.2f} for {num_items} items")
    return estimated_cost

plan = vn.get_training_plan_postgres()
cost = estimate_training_cost(plan)

if cost > 100:
    print("Cost too high! Please filter the plan.")
else:
    vn.train(plan=plan)
```

#### 5. 모델 버전 관리
```python
import datetime

# 버전별 모델
timestamp = datetime.datetime.now().strftime("%Y%m%d")
model_name = f"sales-model-{timestamp}"

vn = VannaDefault(model=model_name, api_key="...")

# 학습
plan = vn.get_training_plan_postgres()
vn.train(plan=plan)

# 이전 버전과 비교 테스트
test_questions = [
    "지난달 매출은?",
    "상위 10개 상품은?",
    # ...
]

for question in test_questions:
    sql_new = vn.generate_sql(question)
    # Compare with previous model
    # ...
```

---

## 핵심 학습

### 1. Breaking Change의 올바른 방법

Vanna는 대규모 API 변경을 **즉시 에러**로 처리했습니다:

```python
def error_deprecation():
    raise Exception("""
Please switch to the following method for initializing Vanna:

from vanna.remote import VannaDefault
vn = VannaDefault(model=vanna_model_name, api_key=api_key)
""")
```

**핵심 원칙**:
1. **명확한 에러 메시지**: 무엇이 문제이고 어떻게 해결하는지 명시
2. **마이그레이션 가이드**: 코드 예시 포함
3. **과감한 결정**: Warning이 아닌 Error (빠른 기술 부채 제거)

**효과**:
- 사용자가 즉시 마이그레이션
- 코드베이스 정리 (레거시 코드 유지 불필요)
- 미래 유지보수 비용 감소

### 2. OTP 인증의 UX 설계

CLI 환경에서 OTP 인증을 구현할 때:

```python
if otp_code is None:
    # 1. OTP 전송
    __unauthenticated_rpc_call("send_otp", [UserEmail(email=email)])
    # 2. 사용자 입력 대기
    otp_code = input("Check your email for the code and enter it here: ")

# 3. OTP 검증
__unauthenticated_rpc_call("verify_otp", [UserOTP(email=email, otp=otp_code)])
```

**핵심 원칙**:
1. **환경 변수 우선**: 재인증 불필요
2. **명확한 프롬프트**: "Check your email for the code..."
3. **프로그래매틱 지원**: `otp_code` 파라미터로 자동화 가능

### 3. TrainingPlan 패턴

대량 데이터 처리 시 "Preview → Modify → Execute" 패턴:

```python
# 1. Plan 생성 (자동 스캔)
plan = vn.get_training_plan_postgres()

# 2. Preview
print(plan.get_summary())

# 3. Modify
plan.remove_item("Train on DDL: temp_table")

# 4. Execute
vn.train(plan=plan)
```

**적용 사례**:
- Database migration (preview → modify → execute)
- Bulk email sending (preview recipients → remove → send)
- File deletion (preview → confirm → delete)

**핵심 원칙**:
- 자동화 (스캔)
- 투명성 (preview)
- 제어 (modify)
- 안전성 (execute 전에 확인)

### 4. Unauthenticated RPC 패턴

인증 전 API 호출:

```python
def __unauthenticated_rpc_call(method, params):
    data = {"method": method, "params": params}
    response = requests.post(_unauthenticated_endpoint, json=data)
    return response.json()
```

**장점**:
- 단일 엔드포인트
- 메서드 추가가 쉬움
- JSON-RPC와 유사

**적용 사례**:
- 회원가입 (인증 전)
- 비밀번호 재설정
- Public API

### 5. Deprecation 전략의 스펙트럼

| 수준 | 방법 | 효과 | 사용 시기 |
|------|------|------|-----------|
| 1 | 문서에만 명시 | 약함 | 사소한 변경 |
| 2 | Warning 로그 | 중간 | 점진적 변경 |
| 3 | DeprecationWarning | 강함 | 주요 변경 |
| 4 | 즉시 Error ✅ | 매우 강함 | Breaking change |

**Vanna의 선택**: Level 4 (즉시 Error)

**선택 기준**:
- API 전체 재설계 → Level 4
- 함수 시그니처 변경 → Level 3
- 파라미터명 변경 → Level 2
- 내부 구현 변경 → Level 1

---

## 요약

| 측면 | 내용 |
|------|------|
| **핵심 역할** | 패키지 엔트리 포인트 + API 마이그레이션 + Vanna.AI 인증 |
| **주요 문제** | 1) 레거시 API 마이그레이션 2) OTP 인증 3) TrainingPlan 설계 4) Unauthenticated RPC 5) Global vs Instance |
| **해결 방법** | 즉시 에러 + 명확한 가이드 + OTP 흐름 + Preview-Modify-Execute 패턴 |
| **핵심 패턴** | Breaking Change, OTP Auth, TrainingPlan, RPC, Deprecation |
| **살아있는 코드** | `get_api_key()`, `TrainingPlan` 클래스 |
| **Deprecated 코드** | 30개 이상의 함수 (모두 `error_deprecation()` 호출) |
| **학습 포인트** | Breaking Change의 올바른 방법, CLI 인증 UX, Preview-Modify-Execute 패턴 |
| **대표 코드** | `error_deprecation()` - 명확한 마이그레이션 가이드 포함 에러 |

**한 줄 요약**: 대규모 API 마이그레이션을 Breaking Change로 처리하며, OTP 인증과 TrainingPlan으로 Vanna.AI Cloud 사용성을 제공하는 엔트리 포인트입니다.
