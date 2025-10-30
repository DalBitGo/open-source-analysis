# base.py (Part 3) - 데이터베이스 연결 패턴

> **파일**: `src/vanna/base/base.py` (line 761-1682)
> **역할**: 11개 데이터베이스 연결을 위한 헬퍼 메서드 + 동적 함수 바인딩 패턴
> **주요 메서드**: `connect_to_*()` 시리즈, `run_sql()`

---

## 📋 파일 개요

### 기본 정보
- **경로**: `src/vanna/base/base.py`
- **분석 범위**: line 761-1682 (약 920 lines)
- **주요 클래스**: `VannaBase` (DB 연결 부분)
- **핵심 역할**: 다양한 데이터베이스에 연결하고 SQL 실행 함수를 동적으로 설정

### 핵심 역할 (한 문장)
**"11개 데이터베이스마다 연결 로직을 제공하되, 코어 로직(`ask`, `generate_sql`)은 DB 독립적으로 유지하기 위해 `run_sql` 함수를 동적으로 바인딩"**

### 지원하는 데이터베이스
1. **Snowflake** (line 761-842)
2. **SQLite** (line 843-880)
3. **Postgres** (line 881-1006)
4. **MySQL** (line 1008-1095)
5. **ClickHouse** (line 1096-1174)
6. **Oracle** (line 1175-1265)
7. **BigQuery** (line 1266-1353)
8. **DuckDB** (line 1354-1402)
9. **MSSQL** (line 1403-1449)
10. **Presto** (line 1450-1566)
11. **Hive** (line 1567-1663)

---

## 🔍 해결하는 핵심 문제들

### 문제 1: DB 독립성 - 11개 DB 지원하되 코어 로직은 변경 없이

**문제**
- Vanna는 Postgres, MySQL, Snowflake, BigQuery 등 11개 DB 지원
- 각 DB마다 연결 방법, SQL 실행 방법이 다름
- 코어 로직(`ask`, `generate_sql`)을 DB마다 수정할 수 없음

**문제가 없었다면?**
- DB 하나만 지원 → 사용자 제한
- 또는 코어 로직에 if-else 지옥:
  ```python
  def run_sql(self, sql):
      if self.db_type == 'postgres':
          return postgres_execute(sql)
      elif self.db_type == 'mysql':
          return mysql_execute(sql)
      # ... 11개 분기
  ```

**고민했던 선택지**

**선택지 1: Strategy 패턴 - 별도 Executor 클래스**
```python
class SQLExecutor(ABC):
    @abstractmethod
    def execute(self, sql): pass

class PostgresExecutor(SQLExecutor):
    def execute(self, sql):
        return pd.read_sql(sql, postgres_conn)

class VannaBase:
    def __init__(self, executor: SQLExecutor):
        self.executor = executor

    def run_sql(self, sql):
        return self.executor.execute(sql)
```
- ✅ 장점: 깔끔한 분리, 확장 용이
- ❌ 단점:
  - 사용자가 Executor 객체 생성해야 함
  - 추가 레이어 복잡도
- 왜 선택 안 함: 간단한 헬퍼 메서드로도 충분

**선택지 2 (최종): 동적 함수 바인딩 - `self.run_sql`에 함수 할당**
```python
class VannaBase:
    def __init__(self):
        self.run_sql_is_set = False  # 플래그

    def connect_to_postgres(self, host, dbname, user, password, port):
        import psycopg2
        conn = psycopg2.connect(host=host, dbname=dbname, user=user, password=password, port=port)

        # 내부 함수 정의
        def run_sql_postgres(sql: str) -> pd.DataFrame:
            cs = conn.cursor()
            cs.execute(sql)
            results = cs.fetchall()
            df = pd.DataFrame(results, columns=[desc[0] for desc in cs.description])
            return df

        # self.run_sql에 함수 할당!
        self.run_sql = run_sql_postgres
        self.run_sql_is_set = True
        self.dialect = "PostgreSQL"

    def connect_to_mysql(self, host, dbname, user, password, port):
        import pymysql
        conn = pymysql.connect(host=host, database=dbname, user=user, password=password, port=port)

        def run_sql_mysql(sql: str) -> pd.DataFrame:
            conn.ping(reconnect=True)  # MySQL 특유의 재연결
            cs = conn.cursor()
            cs.execute(sql)
            results = cs.fetchall()
            df = pd.DataFrame(results, columns=[desc[0] for desc in cs.description])
            return df

        # 동일하게 할당
        self.run_sql = run_sql_mysql
        self.run_sql_is_set = True

    # 코어 로직은 DB 몰라도 됨!
    def ask(self, question):
        sql = self.generate_sql(question)

        if self.run_sql_is_set:
            df = self.run_sql(sql)  # 어떤 DB든 동일한 호출!
            return sql, df
        else:
            raise Exception("Connect to a database first")
```
- ✅ 장점:
  - 코어 로직 완전 독립 (`self.run_sql(sql)` 만 호출)
  - 사용자 경험 간단 (`vn.connect_to_postgres(...)` 한 줄)
  - DB별 최적화 가능 (MySQL은 `ping`, Postgres는 reconnect 로직)
- ⚠️ 단점:
  - Python에서 흔하지 않은 패턴 (함수를 메서드로 할당)
  - IDE 자동완성 제한 (`run_sql` 시그니처가 동적)
- 왜 선택: Python의 동적 특성 활용, 사용자 경험 최우선

**최종 해결책**

```python
class VannaBase:
    def __init__(self, config=None):
        self.run_sql_is_set = False  # 초기값

    # 기본 run_sql (에러 메시지)
    def run_sql(self, sql: str, **kwargs) -> pd.DataFrame:
        raise Exception(
            "You need to connect to a database first by running vn.connect_to_snowflake(), "
            "vn.connect_to_postgres(), similar function, or manually set vn.run_sql"
        )

    # Postgres 연결 예시
    def connect_to_postgres(self, host=None, dbname=None, user=None, password=None, port=None):
        import psycopg2

        # 환경변수 fallback
        host = host or os.getenv("HOST")
        dbname = dbname or os.getenv("DATABASE")
        user = user or os.getenv("PG_USER")
        password = password or os.getenv("PASSWORD")
        port = port or os.getenv("PORT")

        # 검증
        if not all([host, dbname, user, password, port]):
            raise ImproperlyConfigured("Missing Postgres credentials")

        # 연결
        def connect_to_db():
            return psycopg2.connect(host=host, dbname=dbname, user=user, password=password, port=port)

        conn = connect_to_db()

        # SQL 실행 함수 정의
        def run_sql_postgres(sql: str) -> pd.DataFrame:
            try:
                conn_local = connect_to_db()
                cs = conn_local.cursor()
                cs.execute(sql)
                results = cs.fetchall()
                df = pd.DataFrame(results, columns=[desc[0] for desc in cs.description])
                return df
            except psycopg2.InterfaceError:
                # 재연결 시도
                conn_local = connect_to_db()
                cs = conn_local.cursor()
                cs.execute(sql)
                results = cs.fetchall()
                df = pd.DataFrame(results, columns=[desc[0] for desc in cs.description])
                return df
            except psycopg2.Error as e:
                conn.rollback()
                raise ValidationError(e)

        # 동적 바인딩!
        self.run_sql = run_sql_postgres
        self.run_sql_is_set = True
        self.dialect = "PostgreSQL"
```

**핵심 아이디어**
1. **First-class Functions**: Python에서 함수는 객체 → 메서드에 할당 가능
2. **Closure**: 내부 함수가 외부 변수(`conn`, `host` 등) 캡처
3. **Flag Pattern**: `run_sql_is_set`로 연결 여부 확인

**트레이드오프**
- 얻은 것: 코어 로직 독립성, 사용자 경험 간단
- 희생한 것: 정적 타입 힌트 제한, IDE 지원 약함

---

### 문제 2: 환경변수 vs 명시적 파라미터 - 보안과 편의성

**문제**
- DB 인증 정보를 코드에 하드코딩하면 보안 위험
- 환경변수만 사용하면 불편 (매번 export 필요)
- 어떻게 둘 다 지원할까?

**문제가 없었다면?**
- 하드코딩 → Git에 비밀번호 노출 위험
- 환경변수만 → 테스트/개발 시 불편

**고민했던 선택지**

**선택지 1: 파라미터만 받기**
```python
def connect_to_postgres(self, host, dbname, user, password, port):
    # 환경변수 지원 없음
    conn = psycopg2.connect(...)
```
- ✅ 장점: 명확함
- ❌ 단점: 프로덕션에서 하드코딩 유혹
- 왜 안 됨: 보안 위험

**선택지 2: 환경변수만 읽기**
```python
def connect_to_postgres(self):
    host = os.getenv("POSTGRES_HOST")
    # 파라미터 지원 없음
```
- ✅ 장점: 보안
- ❌ 단점: 개발 시 불편, 유연성 낮음
- 왜 안 됨: UX 나쁨

**선택지 3 (최종): 파라미터 우선, 환경변수 Fallback**
```python
def connect_to_postgres(
    self,
    host: str = None,
    dbname: str = None,
    user: str = None,
    password: str = None,
    port: int = None
):
    # 1. 파라미터 우선
    # 2. 환경변수 fallback
    if not host:
        host = os.getenv("HOST")

    if not dbname:
        dbname = os.getenv("DATABASE")

    if not user:
        user = os.getenv("PG_USER")

    if not password:
        password = os.getenv("PASSWORD")

    if not port:
        port = os.getenv("PORT")

    # 3. 검증
    if not host:
        raise ImproperlyConfigured("Please set your postgres host")

    # ... 연결
```
- ✅ 장점:
  - 개발: 파라미터 직접 전달 (빠름)
  - 프로덕션: 환경변수 사용 (안전)
  - 유연성 (둘 다 지원)
- ⚠️ 단점: 코드 길어짐 (boilerplate)
- 왜 선택: 보안 + UX 균형

**핵심 아이디어**
1. **Progressive Fallback**: 파라미터 → 환경변수 → 에러
2. **Fail-fast Validation**: 모든 필수 값 확인 후 에러
3. **Clear Error Messages**: 무엇이 빠졌는지 명확히

**트레이드오프**
- 얻은 것: 보안 + 개발 편의성
- 희생한 것: Boilerplate 코드

---

### 문제 3: 연결 재사용 vs 재연결 - 안정성과 성능

**문제**
- DB 연결은 비쌈 (TCP handshake, 인증)
- 하지만 연결은 끊어질 수 있음 (timeout, network issue)
- 매번 새 연결? 연결 재사용? 끊어지면 재연결?

**문제가 없었다면?**
- 매번 새 연결 → 느림
- 연결 재사용만 → 끊어지면 에러

**고민했던 선택지**

**선택지 1: 매번 새 연결 (Stateless)**
```python
def run_sql_postgres(sql):
    conn = psycopg2.connect(...)  # 매번 연결
    cs = conn.cursor()
    cs.execute(sql)
    conn.close()
    return df
```
- ✅ 장점: 연결 문제 없음 (항상 fresh)
- ❌ 단점: 느림 (매번 handshake)
- 왜 안 됨: 성능 중요

**선택지 2: 연결 재사용 (Stateful)**
```python
conn = psycopg2.connect(...)  # 한 번만

def run_sql_postgres(sql):
    cs = conn.cursor()  # 재사용
    cs.execute(sql)
    return df
```
- ✅ 장점: 빠름
- ❌ 단점: 연결 끊어지면 에러
- 왜 안 됨: 안정성 낮음

**선택지 3 (최종): Postgres - 재연결 시도, MySQL - ping**
```python
# Postgres: 재연결 시도
def connect_to_postgres(self, ...):
    def connect_to_db():
        return psycopg2.connect(...)

    def run_sql_postgres(sql: str):
        try:
            conn = connect_to_db()
            cs = conn.cursor()
            cs.execute(sql)
            return df
        except psycopg2.InterfaceError:
            # 재연결 시도
            conn = connect_to_db()
            cs = conn.cursor()
            cs.execute(sql)
            return df

    self.run_sql = run_sql_postgres

# MySQL: ping으로 연결 확인
def connect_to_mysql(self, ...):
    conn = pymysql.connect(...)

    def run_sql_mysql(sql: str):
        conn.ping(reconnect=True)  # MySQL 특유 기능
        cs = conn.cursor()
        cs.execute(sql)
        return df

    self.run_sql = run_sql_mysql
```
- ✅ 장점:
  - 대부분: 빠름 (재사용)
  - 연결 끊김: 자동 재연결
  - DB별 최적화 (MySQL은 ping)
- ⚠️ 단점: 재연결 시 약간 느림
- 왜 선택: 안정성 + 성능 균형

**핵심 아이디어**
1. **Retry on Error**: `InterfaceError` 잡고 재시도
2. **DB-specific Optimization**: MySQL의 `ping(reconnect=True)`
3. **Factory Function**: `connect_to_db()` 클로저로 재사용

**트레이드오프**
- 얻은 것: 안정성 (자동 재연결)
- 희생한 것: 에러 시 약간 느림

---

### 문제 4: BigQuery - 인증 복잡도

**문제**
- BigQuery는 인증이 복잡함:
  1. Google Colab: 자동 인증
  2. 로컬: JSON 키 파일
  3. GCE/GKE: 자동 인증 (Application Default Credentials)
- 어떻게 모든 경우를 지원할까?

**고민했던 선택지**

**선택지 1: JSON 키만 지원**
```python
def connect_to_bigquery(self, cred_file_path):
    credentials = service_account.Credentials.from_service_account_file(cred_file_path)
    client = bigquery.Client(credentials=credentials)
```
- ✅ 장점: 간단
- ❌ 단점: Colab/GCE에서 불편
- 왜 안 됨: 유연성 낮음

**선택지 2 (최종): 여러 인증 방법 시도**
```python
def connect_to_bigquery(self, cred_file_path=None, project_id=None):
    import sys
    from google.cloud import bigquery
    from google.oauth2 import service_account

    # 1. Google Colab 체크
    if "google.colab" in sys.modules:
        from google.colab import auth
        auth.authenticate_user()  # 자동 인증
        return bigquery.Client(project=project_id)

    # 2. ADC (Application Default Credentials) 시도
    if not cred_file_path:
        try:
            conn = bigquery.Client(project=project_id)
            return conn  # 성공
        except:
            print("Could not find implicit credentials")

    # 3. JSON 키 파일 사용
    if cred_file_path:
        validate_config_path(cred_file_path)  # 파일 존재 확인
        with open(cred_file_path, "r") as f:
            credentials = service_account.Credentials.from_service_account_info(
                json.loads(f.read()),
                scopes=["https://www.googleapis.com/auth/cloud-platform"]
            )
        return bigquery.Client(project=project_id, credentials=credentials)

    raise ImproperlyConfigured("Could not connect to BigQuery")
```
- ✅ 장점: 모든 환경 지원
- ⚠️ 단점: 복잡함
- 왜 선택: BigQuery 특성상 필요

**핵심 아이디어**
1. **Environment Detection**: `sys.modules` 체크
2. **Fallback Chain**: Colab → ADC → JSON
3. **Security**: `validate_config_path()`로 파일 검증

**트레이드오프**
- 얻은 것: 모든 환경 지원
- 희생한 것: 코드 복잡도

---

### 문제 5: SQL Dialect 차이 - 세미콜론, 예약어 등

**문제**
- Oracle/Presto: 세미콜론 있으면 에러
- Postgres/MySQL: 세미콜론 있어도 OK
- LLM이 세미콜론 포함해서 SQL 생성 가능

**고민했던 선택지**

**선택지 1: LLM에게 dialect 명시**
```python
prompt = f"Generate {self.dialect} SQL without semicolons"
```
- ✅ 장점: LLM이 dialect에 맞게 생성
- ❌ 단점: LLM이 완벽하지 않음
- 왜 안 됨: 여전히 세미콜론 나올 수 있음

**선택지 2 (최종): SQL 실행 전에 후처리**
```python
# Oracle
def run_sql_oracle(sql: str):
    sql = sql.rstrip()
    if sql.endswith(';'):
        sql = sql[:-1]  # 세미콜론 제거
    cs.execute(sql)

# Presto
def run_sql_presto(sql: str):
    sql = sql.rstrip()
    if sql.endswith(';'):
        sql = sql[:-1]
    cs.execute(sql)
```
- ✅ 장점: 확실함
- ⚠️ 단점: DB별로 추가 로직
- 왜 선택: LLM보다 확실

**핵심 아이디어**
1. **Defensive Programming**: SQL 실행 전 정규화
2. **DB-specific Handling**: 필요한 DB만 후처리
3. **Set Dialect**: `self.dialect = "Oracle"` → LLM 힌트

**트레이드오프**
- 얻은 것: 안정성
- 희생한 것: DB별 특수 코드

---

## ⭐ 실전 적용 가이드

### 가이드 1: 새로운 DB 연결 추가하기

**상황**: Vanna가 지원하지 않는 DB 추가 (예: Redshift)

#### Step 1: 요구사항 정의
- [ ] Python DB 드라이버 확인 (예: `psycopg2` for Redshift)
- [ ] 연결 파라미터 (host, port, database, user, password)
- [ ] 환경변수 이름 정의
- [ ] SQL dialect 특성 (세미콜론, 예약어 등)

#### Step 2: 기본 구현

```python
def connect_to_redshift(
    self,
    host: str = None,
    dbname: str = None,
    user: str = None,
    password: str = None,
    port: int = None,
    **kwargs
):
    """
    Connect to Amazon Redshift (Postgres-compatible)
    """
    try:
        import psycopg2
    except ImportError:
        raise DependencyError(
            "You need to install psycopg2: pip install psycopg2-binary"
        )

    # 환경변수 fallback
    if not host:
        host = os.getenv("REDSHIFT_HOST")
    if not dbname:
        dbname = os.getenv("REDSHIFT_DATABASE")
    if not user:
        user = os.getenv("REDSHIFT_USER")
    if not password:
        password = os.getenv("REDSHIFT_PASSWORD")
    if not port:
        port = os.getenv("REDSHIFT_PORT", 5439)  # Redshift 기본 포트

    # 검증
    if not all([host, dbname, user, password, port]):
        raise ImproperlyConfigured("Missing Redshift credentials")

    # 연결 팩토리
    def connect_to_db():
        return psycopg2.connect(
            host=host,
            dbname=dbname,
            user=user,
            password=password,
            port=port,
            **kwargs
        )

    # 초기 연결 테스트
    conn = connect_to_db()

    # SQL 실행 함수
    def run_sql_redshift(sql: str) -> pd.DataFrame:
        try:
            conn_local = connect_to_db()
            cs = conn_local.cursor()
            cs.execute(sql)
            results = cs.fetchall()
            df = pd.DataFrame(results, columns=[desc[0] for desc in cs.description])
            return df
        except psycopg2.InterfaceError:
            # 재연결 시도
            conn_local = connect_to_db()
            cs = conn_local.cursor()
            cs.execute(sql)
            results = cs.fetchall()
            df = pd.DataFrame(results, columns=[desc[0] for desc in cs.description])
            return df
        except psycopg2.Error as e:
            raise ValidationError(f"Redshift error: {e}")

    # 동적 바인딩
    self.run_sql = run_sql_redshift
    self.run_sql_is_set = True
    self.dialect = "Amazon Redshift"

    print(f"✅ Connected to Redshift: {dbname}@{host}")
```

#### Step 3: 사용 예시

```python
from vanna.base import VannaBase
from vanna.chromadb import ChromaDB_VectorStore
from vanna.openai import OpenAI_Chat

class MyVanna(ChromaDB_VectorStore, OpenAI_Chat, VannaBase):
    pass

vn = MyVanna(config={'api_key': 'sk-...', 'model': 'gpt-4'})

# Redshift 연결
vn.connect_to_redshift(
    host="my-cluster.redshift.amazonaws.com",
    dbname="mydb",
    user="myuser",
    password="***",
    port=5439
)

# 또는 환경변수 사용
# export REDSHIFT_HOST=...
# vn.connect_to_redshift()

sql = vn.generate_sql("Show top 10 customers")
df = vn.run_sql(sql)
```

---

### 가이드 2: Connection Pooling 추가하기

**상황**: 대규모 서비스에서 연결 재사용 최적화

#### Step 1: Pooling 라이브러리 선택
```bash
pip install psycopg2 sqlalchemy
```

#### Step 2: 구현

```python
def connect_to_postgres_with_pool(
    self,
    host: str,
    dbname: str,
    user: str,
    password: str,
    port: int = 5432,
    pool_size: int = 5,
    max_overflow: int = 10
):
    from sqlalchemy import create_engine
    from sqlalchemy.pool import QueuePool

    # SQLAlchemy 엔진 (Connection Pool 내장)
    connection_string = f"postgresql://{user}:{password}@{host}:{port}/{dbname}"
    engine = create_engine(
        connection_string,
        poolclass=QueuePool,
        pool_size=pool_size,  # 기본 연결 수
        max_overflow=max_overflow,  # 추가 가능 연결 수
        pool_pre_ping=True  # 연결 체크
    )

    def run_sql_postgres_pooled(sql: str) -> pd.DataFrame:
        with engine.begin() as conn:
            df = pd.read_sql_query(sql, conn)
            return df

    self.run_sql = run_sql_postgres_pooled
    self.run_sql_is_set = True
    self.dialect = "PostgreSQL"

    print(f"✅ Connected with pool (size={pool_size}, max_overflow={max_overflow})")
```

#### Step 3: 모니터링

```python
from sqlalchemy import event

@event.listens_for(engine, "connect")
def receive_connect(dbapi_conn, connection_record):
    print(f"[Pool] New connection created")

@event.listens_for(engine, "checkout")
def receive_checkout(dbapi_conn, connection_record, connection_proxy):
    print(f"[Pool] Connection checked out")

# Pool 상태 확인
def check_pool_status(self):
    pool = self.engine.pool
    print(f"Pool size: {pool.size()}")
    print(f"Checked out: {pool.checkedout()}")
    print(f"Overflow: {pool.overflow()}")
```

---

### 가이드 3: 보안 강화 - Secret Manager 통합

**상황**: 프로덕션에서 환경변수 대신 Secret Manager 사용

#### Step 1: AWS Secrets Manager 예시

```python
def connect_to_postgres_with_secrets(
    self,
    secret_name: str,
    region_name: str = "us-east-1"
):
    import boto3
    import json

    # Secrets Manager 클라이언트
    client = boto3.client('secretsmanager', region_name=region_name)

    try:
        response = client.get_secret_value(SecretId=secret_name)
        secret = json.loads(response['SecretString'])
    except Exception as e:
        raise ImproperlyConfigured(f"Could not retrieve secret: {e}")

    # Secret 구조 예시:
    # {
    #   "host": "db.example.com",
    #   "dbname": "mydb",
    #   "user": "myuser",
    #   "password": "***",
    #   "port": 5432
    # }

    # 기존 connect 메서드 재사용
    self.connect_to_postgres(
        host=secret['host'],
        dbname=secret['dbname'],
        user=secret['user'],
        password=secret['password'],
        port=secret['port']
    )

    print(f"✅ Connected using secret: {secret_name}")
```

#### Step 2: GCP Secret Manager

```python
def connect_to_bigquery_with_secrets(
    self,
    project_id: str,
    secret_name: str
):
    from google.cloud import secretmanager
    import json

    client = secretmanager.SecretManagerServiceClient()
    name = f"projects/{project_id}/secrets/{secret_name}/versions/latest"

    response = client.access_secret_version(request={"name": name})
    secret = json.loads(response.payload.data.decode('UTF-8'))

    # JSON 키 파일 내용이 secret에 저장되어 있음
    from google.oauth2 import service_account
    credentials = service_account.Credentials.from_service_account_info(secret)

    self.connect_to_bigquery(project_id=project_id, credentials=credentials)
```

---

### 가이드 4: 타임아웃 설정

**상황**: 긴 쿼리 방지

#### Step 1: 구현

```python
def connect_to_postgres_with_timeout(
    self,
    host: str,
    dbname: str,
    user: str,
    password: str,
    port: int = 5432,
    query_timeout: int = 30  # 초
):
    import psycopg2

    def connect_to_db():
        conn = psycopg2.connect(
            host=host, dbname=dbname, user=user, password=password, port=port
        )
        # Statement timeout 설정
        conn.cursor().execute(f"SET statement_timeout = {query_timeout * 1000}")  # ms
        return conn

    def run_sql_postgres(sql: str) -> pd.DataFrame:
        try:
            conn = connect_to_db()
            cs = conn.cursor()
            cs.execute(sql)
            results = cs.fetchall()
            df = pd.DataFrame(results, columns=[desc[0] for desc in cs.description])
            return df
        except psycopg2.errors.QueryCanceled:
            raise ValidationError(f"Query exceeded {query_timeout}s timeout")

    self.run_sql = run_sql_postgres
    self.run_sql_is_set = True
    self.dialect = "PostgreSQL"
```

---

## ⭐ 안티패턴과 흔한 실수

### 실수 1: 비밀번호 하드코딩

**❌ 나쁜 예:**
```python
vn.connect_to_postgres(
    host="prod-db.example.com",
    dbname="production",
    user="admin",
    password="P@ssw0rd123"  # ❌ Git에 커밋됨!
)
```

**문제:**
- Git 히스토리에 비밀번호 영구 저장
- 팀원 모두가 비밀번호 알게 됨
- 보안 감사 실패

**✅ 좋은 예:**
```python
# 방법 1: 환경변수
export POSTGRES_PASSWORD="P@ssw0rd123"
vn.connect_to_postgres(
    host="prod-db.example.com",
    dbname="production",
    user="admin"
)  # password는 환경변수에서 자동 읽음

# 방법 2: .env 파일 (gitignore 추가!)
from dotenv import load_dotenv
load_dotenv()
vn.connect_to_postgres(...)

# 방법 3: Secret Manager
vn.connect_to_postgres_with_secrets(secret_name="prod-db-creds")
```

---

### 실수 2: `run_sql` 덮어쓰기 전에 호출

**❌ 나쁜 예:**
```python
vn = MyVanna()

# ❌ DB 연결 전에 SQL 실행 시도
sql = vn.generate_sql("Show top customers")
df = vn.run_sql(sql)  # Exception: You need to connect to a database first
```

**문제:**
- `run_sql_is_set = False` 상태
- 기본 `run_sql` 메서드는 에러만 발생

**✅ 좋은 예:**
```python
vn = MyVanna()

# 1. 먼저 DB 연결
vn.connect_to_postgres(...)

# 2. SQL 생성 & 실행
sql = vn.generate_sql("Show top customers")
df = vn.run_sql(sql)

# 또는 ask() 사용 (내부에서 체크)
sql, df, fig = vn.ask("Show top customers")
```

---

### 실수 3: 연결 객체 닫기 (Closure 이해 안 함)

**❌ 나쁜 예:**
```python
def connect_to_postgres(self, ...):
    conn = psycopg2.connect(...)

    def run_sql_postgres(sql):
        cs = conn.cursor()
        cs.execute(sql)
        return df

    self.run_sql = run_sql_postgres

    conn.close()  # ❌ 클로저 안에서 conn 사용하는데 닫음!
```

**문제:**
- `run_sql` 호출 시 `conn`이 이미 닫혀있음
- `psycopg2.InterfaceError: connection already closed`

**✅ 좋은 예:**
```python
def connect_to_postgres(self, ...):
    conn = psycopg2.connect(...)

    def run_sql_postgres(sql):
        cs = conn.cursor()  # ✅ 클로저가 conn 유지
        cs.execute(sql)
        return df

    self.run_sql = run_sql_postgres
    # conn.close() 하지 않음!
```

---

### 실수 4: 모든 DB에 동일한 로직 (DB 특성 무시)

**❌ 나쁜 예:**
```python
# 모든 DB에 동일한 연결 로직
for db in ['postgres', 'mysql', 'oracle']:
    conn = connect(db)
    # ❌ MySQL은 ping 필요, Oracle은 세미콜론 제거 필요
```

**문제:**
- MySQL: 연결 끊김 체크 없음
- Oracle: 세미콜론 에러
- BigQuery: 인증 방법 다름

**✅ 좋은 예:**
```python
# MySQL: ping
def run_sql_mysql(sql):
    conn.ping(reconnect=True)  # MySQL 특수 처리
    cs.execute(sql)

# Oracle: 세미콜론 제거
def run_sql_oracle(sql):
    sql = sql.rstrip().rstrip(';')  # Oracle 특수 처리
    cs.execute(sql)

# BigQuery: ADC
if "google.colab" in sys.modules:
    auth.authenticate_user()  # BigQuery 특수 처리
```

---

### 실수 5: 환경변수 이름 충돌

**❌ 나쁜 예:**
```python
# Postgres
host = os.getenv("HOST")
user = os.getenv("USER")

# MySQL (동일 환경변수 사용)
host = os.getenv("HOST")  # ❌ Postgres와 충돌!
user = os.getenv("USER")
```

**문제:**
- 여러 DB 동시 사용 시 환경변수 충돌
- `USER`는 시스템 환경변수와도 충돌

**✅ 좋은 예:**
```python
# Postgres
host = os.getenv("POSTGRES_HOST")
user = os.getenv("POSTGRES_USER")

# MySQL
host = os.getenv("MYSQL_HOST")
user = os.getenv("MYSQL_USER")

# Vanna 코드에서는:
# Postgres: PG_USER
# MySQL: USER (기존 호환성)
# → 문서에 명시
```

---

### 실수 6: 에러 처리 없이 연결

**❌ 나쁜 예:**
```python
def connect_to_postgres(self, ...):
    conn = psycopg2.connect(...)  # ❌ 에러 처리 없음

    def run_sql_postgres(sql):
        cs = conn.cursor()
        cs.execute(sql)  # ❌ 에러 처리 없음
```

**문제:**
- 연결 실패 시 스택 트레이스만 출력
- 사용자가 뭐가 잘못됐는지 모름

**✅ 좋은 예:**
```python
def connect_to_postgres(self, ...):
    try:
        conn = psycopg2.connect(...)
    except psycopg2.OperationalError as e:
        raise ImproperlyConfigured(
            f"Could not connect to Postgres at {host}:{port}\n"
            f"Error: {e}\n"
            f"Check: 1) Credentials 2) Network 3) Database running"
        )

    def run_sql_postgres(sql):
        try:
            cs = conn.cursor()
            cs.execute(sql)
        except psycopg2.Error as e:
            conn.rollback()
            raise ValidationError(f"SQL Error: {e}\nSQL: {sql}")
```

---

### 실수 7: 연결 테스트 없이 반환

**❌ 나쁜 예:**
```python
def connect_to_postgres(self, ...):
    conn = psycopg2.connect(...)
    # ❌ 연결 성공 확인 안 함

    def run_sql_postgres(sql):
        ...

    self.run_sql = run_sql_postgres
```

**문제:**
- 연결은 되었지만 권한 없을 수도
- 사용자가 나중에 SQL 실행 시 에러

**✅ 좋은 예:**
```python
def connect_to_postgres(self, ...):
    conn = psycopg2.connect(...)

    # 연결 테스트
    try:
        cs = conn.cursor()
        cs.execute("SELECT 1")
        print(f"✅ Connected to Postgres: {dbname}@{host}")
    except psycopg2.Error as e:
        raise ValidationError(f"Connection test failed: {e}")

    def run_sql_postgres(sql):
        ...

    self.run_sql = run_sql_postgres
```

---

## ⭐ 스케일 고려사항

### 소규모 (< 10 동시 쿼리)

**권장 사항:**
- ✅ 간단한 연결 재사용
- ✅ 환경변수로 인증
- ⚠️ Connection Pool 불필요

**구현 예시:**
```python
# 기본 Vanna 연결 그대로 사용
vn = MyVanna()
vn.connect_to_postgres(
    host="localhost",
    dbname="mydb",
    user="myuser",
    password=os.getenv("DB_PASSWORD")
)
```

**모니터링:**
```python
# 기본 로깅만
import logging
logging.basicConfig(level=logging.INFO)
```

---

### 중규모 (10-100 동시 쿼리)

**권장 사항:**
- ✅ Connection Pool 도입
- ✅ Query timeout 설정
- ✅ 재연결 로직 강화
- ✅ Secret Manager 사용

**구현 예시:**
```python
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

class PooledVanna(MyVanna):
    def connect_to_postgres_pooled(self, ...):
        engine = create_engine(
            connection_string,
            poolclass=QueuePool,
            pool_size=10,  # 기본 연결 10개
            max_overflow=20,  # 최대 30개까지
            pool_pre_ping=True,  # 연결 체크
            pool_recycle=3600  # 1시간마다 재생성
        )

        def run_sql_postgres(sql):
            with engine.begin() as conn:
                df = pd.read_sql_query(sql, conn, timeout=30)
                return df

        self.run_sql = run_sql_postgres

vn = PooledVanna()
vn.connect_to_postgres_pooled(...)
```

**모니터링:**
```python
import time

class MonitoredVanna(PooledVanna):
    def run_sql(self, sql):
        start = time.time()
        try:
            df = super().run_sql(sql)
            duration = time.time() - start
            print(f"[SQL] {duration:.3f}s, {len(df)} rows")
            return df
        except Exception as e:
            print(f"[SQL Error] {e}")
            raise
```

---

### 대규모 (100+ 동시 쿼리)

**권장 사항:**
- ✅ Read Replica 분산
- ✅ Query Queue + Worker Pool
- ✅ 메트릭 수집 (Prometheus)
- ✅ Circuit Breaker 패턴

**구현 예시:**
```python
from sqlalchemy import create_engine
from prometheus_client import Counter, Histogram
import random

sql_query_counter = Counter('vanna_sql_queries_total', 'Total SQL queries')
sql_query_duration = Histogram('vanna_sql_query_seconds', 'SQL query duration')
sql_error_counter = Counter('vanna_sql_errors_total', 'SQL errors')

class LargeScaleVanna(MyVanna):
    def __init__(self, config):
        super().__init__(config)
        # Read Replica 엔진들
        self.read_engines = [
            create_engine(f"postgresql://...@read-replica-{i}", poolclass=QueuePool)
            for i in range(3)  # 3개 replica
        ]
        self.write_engine = create_engine("postgresql://...@master", poolclass=QueuePool)

    def run_sql(self, sql):
        sql_query_counter.inc()

        # SELECT는 Replica, 나머지는 Master
        if sql.strip().upper().startswith('SELECT'):
            engine = random.choice(self.read_engines)  # Load balancing
        else:
            engine = self.write_engine

        with sql_query_duration.time():
            try:
                with engine.begin() as conn:
                    df = pd.read_sql_query(sql, conn, timeout=60)
                    return df
            except Exception as e:
                sql_error_counter.inc()
                raise
```

**Circuit Breaker:**
```python
from pybreaker import CircuitBreaker

class CircuitBreakerVanna(LargeScaleVanna):
    def __init__(self, config):
        super().__init__(config)
        self.breaker = CircuitBreaker(
            fail_max=5,  # 5번 실패 시 open
            timeout_duration=60  # 60초 후 재시도
        )

    def run_sql(self, sql):
        try:
            return self.breaker.call(super().run_sql, sql)
        except CircuitBreakerError:
            raise ValidationError("Database circuit breaker open (too many errors)")
```

**모니터링 (Prometheus):**
```yaml
# Prometheus alerts
- alert: HighSQLErrorRate
  expr: rate(vanna_sql_errors_total[5m]) > 0.1
  annotations:
    summary: "SQL error rate > 10%"

- alert: SlowQueries
  expr: histogram_quantile(0.95, vanna_sql_query_seconds) > 10
  annotations:
    summary: "P95 query time > 10s"

- alert: HighQueryVolume
  expr: rate(vanna_sql_queries_total[1m]) > 1000
  annotations:
    summary: "Query rate > 1000/min"
```

---

## 💡 배운 점

### 1. 동적 함수 바인딩은 강력한 패턴
**핵심 개념**: `self.method = function` → 런타임에 메서드 교체
**언제 사용?**: 다양한 구현체 지원 + 코어 로직 독립성
**적용 가능한 곳**:
- 플러그인 시스템 (로더가 함수 주입)
- Strategy 패턴 대안

### 2. Closure로 연결 상태 캡처
**핵심 개념**: 내부 함수가 외부 변수(`conn`) 캡처 → 상태 유지
**언제 사용?**: Factory 패턴, Stateful 동작
**적용 가능한 곳**:
- Connection Pool
- 캐시 함수

### 3. Fallback Chain으로 유연성 확보
**핵심 개념**: 파라미터 → 환경변수 → 기본값 → 에러
**언제 사용?**: 다양한 환경 (개발/프로덕션) 지원
**적용 가능한 곳**:
- 설정 로딩
- 인증 시스템

### 4. DB별 특수 처리는 불가피
**핵심 개념**: MySQL `ping`, Oracle 세미콜론, BigQuery 인증
**언제 사용?**: 다양한 백엔드 지원
**적용 가능한 곳**:
- 멀티 클라우드 (AWS/GCP/Azure)
- 멀티 DB (SQL/NoSQL)

### 5. 연결 재사용 vs 안정성 트레이드오프
**핵심 개념**: 재사용 (빠름) vs 재연결 (안정)
**언제 사용?**: 네트워크 불안정한 환경
**적용 가능한 곳**:
- HTTP 클라이언트 (Keep-Alive)
- gRPC Connection Pool

### 6. Secret Manager는 프로덕션 필수
**핵심 개념**: 환경변수 > 하드코딩, Secret Manager > 환경변수
**언제 사용?**: 프로덕션 배포
**적용 가능한 곳**:
- API 키 관리
- 데이터베이스 인증

### 7. Connection Pool은 성능의 핵심
**핵심 개념**: 연결 생성 비용 >> 재사용 이득
**언제 사용?**: 동시 쿼리 10개 이상
**적용 가능한 곳**:
- 웹 서버 (DB 연결)
- API Gateway (백엔드 연결)

### 8. 명확한 에러 메시지가 UX
**핵심 개념**: "Connection failed" < "Check 1) Credentials 2) Network 3) DB running"
**언제 사용?**: 사용자 대면 라이브러리
**적용 가능한 곳**:
- CLI 도구
- SDK

---

## 📊 요약

| 항목 | 내용 |
|------|------|
| **핵심 문제** | 11개 DB 지원 + 코어 로직 독립성 |
| **핵심 패턴** | 동적 함수 바인딩 + Closure + Fallback Chain |
| **주요 트레이드오프** | 유연성 & 단순성 vs 정적 타입 제한 |
| **핵심 기법** | 1) `self.run_sql = function` 동적 할당<br>2) Closure로 `conn` 캡처<br>3) 파라미터 → 환경변수 Fallback<br>4) DB별 특수 처리 (ping, 세미콜론) |
| **적용 시 주의** | 1) 비밀번호 하드코딩 금지<br>2) DB 연결 후 SQL 실행<br>3) Closure 이해 (conn 닫지 말기)<br>4) DB 특성 고려 |
| **스케일 전략** | 소규모: 기본 연결<br>중규모: Connection Pool<br>대규모: Read Replica + Circuit Breaker |
| **실무 적용** | DB 추상화 레이어, 멀티 DB 지원 서비스 |

---

## 🔗 다음 단계

- **Part 4**: 고수준 API (`ask()`, `train()`) 분석 - 사용자가 직접 호출하는 메인 API

---

**✅ Part 3 분석 완료!**
