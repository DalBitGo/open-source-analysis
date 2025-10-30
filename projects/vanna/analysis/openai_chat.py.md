# vanna/openai/openai_chat.py 분석

## 파일 개요

**경로**: `src/vanna/openai/openai_chat.py`
**라인 수**: 128줄
**목적**: OpenAI API를 사용한 LLM 인터페이스 구현

VannaBase의 `submit_prompt()` 추상 메서드를 구현하여 OpenAI GPT 모델과의 통신을 담당합니다. 간단해 보이지만 Client 초기화, 모델 자동 선택, Azure/OpenAI 호환성, Deprecation 처리 등 다양한 문제를 해결합니다.

---

## 핵심 문제 해결

### 1. LLM 독립성 문제

#### 문제
base.py는 특정 LLM에 종속되지 않아야 합니다. 다양한 LLM(OpenAI, Anthropic, Local LLM)을 지원하려면?

#### 고민한 대안들

**Option A: LLM별 if-else 분기**
```python
class VannaBase:
    def submit_prompt(self, prompt):
        if self.llm_type == "openai":
            # OpenAI API 호출
        elif self.llm_type == "anthropic":
            # Anthropic API 호출
        # 새로운 LLM 추가 시마다 수정 필요
```
- 장점: 구현 간단
- 단점: OCP 위반, 확장성 낮음

**Option B: 추상 메서드 + Mixin 패턴** ✅
```python
# base.py
class VannaBase(ABC):
    @abstractmethod
    def submit_prompt(self, prompt, **kwargs) -> str:
        pass

# openai_chat.py
class OpenAI_Chat(VannaBase):
    def submit_prompt(self, prompt, **kwargs) -> str:
        response = self.client.chat.completions.create(...)
        return response.choices[0].message.content

# 사용자 코드
class MyVanna(ChromaDB_VectorStore, OpenAI_Chat):
    pass  # Mixin으로 조합
```
- 장점: LLM 독립성, 새 LLM 추가 시 base.py 수정 불필요
- 단점: Mixin 패턴 이해 필요

#### 최종 선택: Option B

**선택 이유**:
- 새로운 LLM 추가 시 base.py 수정 불필요
- 사용자가 Custom LLM 구현 가능
- Vector Store와 독립적으로 조합 가능

**트레이드오프**:
- Mixin 패턴의 복잡도 증가
- 초기 학습 곡선

---

### 2. Client 초기화 호환성

#### 문제
OpenAI SDK v1.0.0 마이그레이션으로 API가 변경되었습니다. 기존 사용자 코드를 어떻게 처리할까?

**변경 사항**:
```python
# Old (deprecated)
openai.api_key = "sk-xxx"
openai.api_type = "azure"
openai.api_base = "https://..."

# New (v1.0.0+)
client = OpenAI(api_key="sk-xxx")
```

#### 고민한 대안들

**Option A: 자동 마이그레이션**
```python
if "api_type" in config:
    # 자동으로 새 방식으로 변환
    self.client = OpenAI(
        api_key=config["api_key"],
        base_url=config["api_base"]
    )
```
- 장점: 기존 코드 동작
- 단점: 숨겨진 동작, 디버깅 어려움

**Option B: 명시적 에러 + 마이그레이션 가이드** ✅
```python
if "api_type" in config:
    raise Exception(
        "Passing api_type is now deprecated. "
        "Please pass an OpenAI client instead."
    )
```
- 장점: 명확한 의도, 사용자가 의식적으로 마이그레이션
- 단점: 기존 코드 즉시 중단

#### 최종 선택: Option B

**선택 이유**:
1. **명확성**: 사용자가 deprecated API 사용 중임을 명확히 인지
2. **문서화**: OpenAI SDK 공식 마이그레이션 가이드 참조 가능
3. **유지보수성**: 미래에 old API 제거 용이

**구현 코드** (openai_chat.py:18-31):
```python
if "api_type" in config:
    raise Exception(
        "Passing api_type is now deprecated. Please pass an OpenAI client instead."
    )

if "api_base" in config:
    raise Exception(
        "Passing api_base is now deprecated. Please pass an OpenAI client instead."
    )
```

**3가지 초기화 방법**:
```python
# 1. Client 주입 (권장: 테스트 용이)
client = OpenAI(api_key="sk-xxx", timeout=30.0)
vn = MyVanna(client=client, config={...})

# 2. Config로 api_key 전달
vn = MyVanna(config={"api_key": "sk-xxx"})

# 3. 환경 변수 (가장 간단)
# export OPENAI_API_KEY="sk-xxx"
vn = MyVanna()  # 자동으로 os.getenv() 사용
```

---

### 3. 자동 모델 선택

#### 문제
프롬프트 길이에 따라 적절한 모델을 선택해야 합니다. 사용자가 직접 모델을 선택하면?

#### 고민한 대안들

**Option A: 항상 최대 컨텍스트 모델**
```python
model = "gpt-3.5-turbo-16k"  # 항상 16K
```
- 장점: 간단, 토큰 초과 없음
- 단점: 비용 낭비 (짧은 프롬프트에도 16K 사용)

**Option B: 토큰 수 기반 자동 선택** ✅
```python
num_tokens = sum(len(m["content"]) / 4 for m in prompt)
if num_tokens > 3500:
    model = "gpt-3.5-turbo-16k"
else:
    model = "gpt-3.5-turbo"
```
- 장점: 비용 최적화
- 단점: 토큰 계산 오차 (실제와 다를 수 있음)

**Option C: 사용자 지정 우선**
```python
# kwargs > config > 자동 선택
if kwargs.get("model"):
    model = kwargs["model"]
elif self.config.get("model"):
    model = self.config["model"]
else:
    # 자동 선택
```
- 장점: 유연성 최대
- 단점: 우선순위 로직 복잡

#### 최종 선택: Option C (B 포함)

**선택 이유**:
1. **비용 효율**: 짧은 프롬프트는 저렴한 모델 사용
2. **유연성**: 사용자가 명시적으로 지정 가능
3. **안전성**: 긴 프롬프트 자동 처리

**구현 코드** (openai_chat.py:66-120):
```python
def submit_prompt(self, prompt, **kwargs) -> str:
    # 토큰 수 근사치 계산 (4 chars ≈ 1 token)
    num_tokens = sum(len(m["content"]) / 4 for m in prompt)

    # 우선순위: kwargs > config > 자동 선택
    if kwargs.get("model"):
        model = kwargs["model"]
    elif kwargs.get("engine"):  # Azure OpenAI
        engine = kwargs["engine"]
    elif self.config and "model" in self.config:
        model = self.config["model"]
    else:
        # 자동 모델 선택
        model = "gpt-3.5-turbo-16k" if num_tokens > 3500 else "gpt-3.5-turbo"

    print(f"Using model {model} for {num_tokens} tokens (approx)")
    response = self.client.chat.completions.create(
        model=model,
        messages=prompt,
        temperature=self.temperature
    )
    return response.choices[0].message.content
```

**토큰 계산 정확도**:
- 근사치 공식: `chars / 4 ≈ tokens`
- 실제 토큰 계산은 tiktoken 사용 권장
- 하지만 근사치로도 모델 선택에는 충분

---

### 4. Azure vs OpenAI 호환성

#### 문제
Azure OpenAI는 `engine` 파라미터, 일반 OpenAI는 `model` 파라미터를 사용합니다.

**API 차이**:
```python
# OpenAI
client.chat.completions.create(model="gpt-3.5-turbo", ...)

# Azure OpenAI
client.chat.completions.create(engine="my-deployment", ...)
```

#### 고민한 대안들

**Option A: 별도 클래스**
```python
class OpenAI_Chat(VannaBase): ...
class AzureOpenAI_Chat(VannaBase): ...
```
- 장점: 명확한 분리
- 단점: 코드 중복, 유지보수 2배

**Option B: 통합 클래스 + 파라미터 분기** ✅
```python
if kwargs.get("model"):
    response = self.client.chat.completions.create(model=..., ...)
elif kwargs.get("engine"):
    response = self.client.chat.completions.create(engine=..., ...)
```
- 장점: 코드 통합, DRY 원칙
- 단점: if-else 중복 (각 분기마다 API 호출)

#### 최종 선택: Option B

**선택 이유**:
- 두 API가 거의 동일 (파라미터명만 차이)
- 코드 중복 최소화
- 사용자는 config만 변경하면 됨

**사용 예시**:
```python
# OpenAI
vn = MyVanna(config={"model": "gpt-4"})

# Azure OpenAI
client = AzureOpenAI(
    api_key="...",
    azure_endpoint="https://my-resource.openai.azure.com/"
)
vn = MyVanna(client=client, config={"engine": "my-gpt-4-deployment"})
```

**트레이드오프**:
- API 호출 코드 중복 (lines 71-120)
- 하지만 if-else로 model/engine만 분기하면 더 복잡해질 수 있음

---

### 5. Message 포맷 추상화

#### 문제
OpenAI API는 특정 메시지 포맷을 요구합니다:
```python
[
    {"role": "system", "content": "You are a helpful assistant."},
    {"role": "user", "content": "Hello"}
]
```

base.py에서 이 포맷을 직접 사용하면 LLM 종속성이 생깁니다.

#### 고민한 대안들

**Option A: base.py에서 직접 dict 생성**
```python
# base.py
prompt = [
    {"role": "system", "content": system_msg},
    {"role": "user", "content": user_msg}
]
```
- 장점: 간단
- 단점: LLM 종속성, 다른 LLM은 포맷이 다를 수 있음

**Option B: Helper 메서드로 추상화** ✅
```python
# openai_chat.py
def system_message(self, message: str) -> dict:
    return {"role": "system", "content": message}

# base.py (LLM 독립적)
prompt = [
    self.system_message(system_msg),
    self.user_message(user_msg)
]
```
- 장점: LLM 독립성, 포맷 변경 시 한 곳만 수정
- 단점: 간접 호출 (미미한 오버헤드)

#### 최종 선택: Option B

**구현 코드** (openai_chat.py:44-51):
```python
def system_message(self, message: str) -> any:
    return {"role": "system", "content": message}

def user_message(self, message: str) -> any:
    return {"role": "user", "content": message}

def assistant_message(self, message: str) -> any:
    return {"role": "assistant", "content": message}
```

**base.py에서 사용** (base.py:420-425):
```python
def get_sql_prompt(self, question, question_sql_list, ddl_list, doc_list):
    message_log = [self.system_message(initial_prompt)]

    for question, sql in question_sql_list:
        message_log.append(self.user_message(question))
        message_log.append(self.assistant_message(sql))

    message_log.append(self.user_message(question))
    return message_log
```

**다른 LLM 구현 예시**:
```python
class Anthropic_Chat(VannaBase):
    def system_message(self, message: str) -> dict:
        # Claude API는 system을 별도 파라미터로
        return {"type": "system", "text": message}

    def submit_prompt(self, prompt, **kwargs):
        # Claude API 형식으로 변환
        system = next(m["text"] for m in prompt if m["type"] == "system")
        messages = [m for m in prompt if m["type"] != "system"]

        response = self.client.messages.create(
            system=system,
            messages=messages,
            ...
        )
```

---

## 실전 활용 가이드

### 1. 기본 사용

```python
from vanna.openai import OpenAI_Chat
from vanna.chromadb import ChromaDB_VectorStore

class MyVanna(ChromaDB_VectorStore, OpenAI_Chat):
    def __init__(self, config=None):
        ChromaDB_VectorStore.__init__(self, config=config)
        OpenAI_Chat.__init__(self, config=config)

# 환경 변수 사용 (가장 간단)
import os
os.environ["OPENAI_API_KEY"] = "sk-xxx"
vn = MyVanna()

# SQL 생성
sql = vn.generate_sql("지난달 매출 상위 10개 상품은?")
```

### 2. Client 주입 (권장: 테스트 용이)

```python
from openai import OpenAI

# 타임아웃, 리트라이 설정
client = OpenAI(
    api_key="sk-xxx",
    timeout=30.0,
    max_retries=3
)

vn = MyVanna(client=client)

# 테스트 시 Mock Client 주입 가능
from unittest.mock import Mock
mock_client = Mock()
mock_client.chat.completions.create.return_value = Mock(
    choices=[Mock(message=Mock(content="SELECT * FROM products"))]
)
vn_test = MyVanna(client=mock_client)
```

### 3. 모델 명시 (GPT-4 사용)

```python
# Config으로 지정
vn = MyVanna(config={"model": "gpt-4", "temperature": 0.0})

# 또는 런타임에 지정
sql = vn.generate_sql("복잡한 쿼리", model="gpt-4")
```

### 4. Azure OpenAI 사용

```python
from openai import AzureOpenAI

client = AzureOpenAI(
    api_key="...",
    api_version="2024-02-01",
    azure_endpoint="https://my-resource.openai.azure.com/"
)

vn = MyVanna(
    client=client,
    config={"engine": "my-gpt-35-deployment"}
)
```

### 5. Temperature 조절

```python
# 결정적 출력 (항상 동일한 SQL)
vn = MyVanna(config={"temperature": 0.0})

# 창의적 출력 (다양한 SQL)
vn = MyVanna(config={"temperature": 0.9})

# 기본값: 0.7 (균형)
```

### 6. Token 수 모니터링

```python
# submit_prompt는 자동으로 토큰 수 출력
vn.generate_sql("...")
# Output: Using model gpt-3.5-turbo for 850 tokens (approx)

# 또는 로깅 캡처
import logging
logging.basicConfig(level=logging.INFO)

# 프로덕션: stdout 캡처
import io
import sys
old_stdout = sys.stdout
sys.stdout = io.StringIO()
sql = vn.generate_sql("...")
token_log = sys.stdout.getvalue()
sys.stdout = old_stdout
print(f"Token usage: {token_log}")
```

---

## Anti-Patterns (하지 말아야 할 것들)

### ❌ 1. Config에 api_type 사용

```python
# WRONG: Deprecated API
vn = MyVanna(config={
    "api_type": "azure",  # Exception 발생!
    "api_base": "https://...",
    "api_key": "..."
})

# CORRECT: Client 주입
from openai import AzureOpenAI
client = AzureOpenAI(...)
vn = MyVanna(client=client)
```

**이유**: OpenAI SDK v1.0.0부터 deprecated. 명시적 에러로 마이그레이션 강제.

### ❌ 2. 하드코딩된 API Key

```python
# WRONG: 코드에 직접 작성
vn = MyVanna(config={"api_key": "sk-xxx123..."})

# CORRECT: 환경 변수 또는 Secret Manager
import os
api_key = os.getenv("OPENAI_API_KEY")
# 또는
from google.cloud import secretmanager
api_key = get_secret("openai-api-key")

vn = MyVanna(config={"api_key": api_key})
```

**이유**:
- Git에 실수로 커밋 방지
- 환경별 키 분리 (dev/prod)
- 보안 감사 추적

### ❌ 3. 토큰 수 무시

```python
# WRONG: 무제한 컨텍스트 가정
for doc in huge_doc_list:
    prompt += doc  # 100K+ tokens

sql = vn.generate_sql(question)  # 토큰 초과 에러!

# CORRECT: Token budget 관리 (base.py에서 제공)
ddl_list = vn.get_related_ddl(question)
prompt = vn.add_ddl_to_prompt(initial_prompt, ddl_list, max_tokens=14000)
```

**이유**:
- GPT-3.5-turbo: 4K context (16K 모델: 16K)
- 자동 모델 선택도 한계 있음
- base.py의 `add_ddl_to_prompt`가 자동으로 budget 관리

### ❌ 4. Temperature 극단값

```python
# WRONG: 항상 temperature=1.0
vn = MyVanna(config={"temperature": 1.0})
sql = vn.generate_sql("SELECT * FROM users WHERE id = {user_id}")
# Output: "SELECT * FROM customers WHERE user_identifier = ..."
# → 구문 오류 가능성 높음

# CORRECT: SQL 생성은 낮은 temperature
vn = MyVanna(config={"temperature": 0.0})  # 결정적 출력
```

**이유**:
- SQL은 정확성이 중요 (창의성 불필요)
- High temperature → 구문 오류 증가
- 권장: 0.0 ~ 0.3

### ❌ 5. 모델 선택 무시

```python
# WRONG: 항상 gpt-4 사용
vn = MyVanna(config={"model": "gpt-4"})
# 간단한 쿼리도 gpt-4 사용 → 비용 10배

# CORRECT: 자동 모델 선택 활용
vn = MyVanna()  # 기본값: 토큰 수 기반 선택

# 또는 복잡한 쿼리만 gpt-4
if is_complex_query(question):
    sql = vn.generate_sql(question, model="gpt-4")
else:
    sql = vn.generate_sql(question)  # gpt-3.5-turbo
```

**비용 비교** (2024년 기준):
- gpt-3.5-turbo: $0.001 / 1K tokens
- gpt-4: $0.03 / 1K tokens (30배)

### ❌ 6. 에러 처리 없음

```python
# WRONG: 예외 무시
sql = vn.generate_sql(question)  # API 에러 시 크래시

# CORRECT: 에러 처리
try:
    sql = vn.generate_sql(question)
except Exception as e:
    if "rate_limit" in str(e):
        time.sleep(60)
        sql = vn.generate_sql(question)
    elif "context_length_exceeded" in str(e):
        # Token budget 줄이기
        vn.config["max_tokens"] = 8000
        sql = vn.generate_sql(question)
    else:
        raise
```

**주요 에러**:
- `RateLimitError`: 리트라이 필요
- `ContextLengthExceeded`: Token budget 초과
- `AuthenticationError`: API Key 오류

---

## 스케일 고려사항

### Small Scale (< 100 req/day)

**특징**:
- 개발 환경 또는 PoC
- Rate limit 걱정 없음
- 비용 < $10/month

**권장 설정**:
```python
vn = MyVanna(config={
    "temperature": 0.0,  # 결정적 출력
    # model 지정 안 함 (자동 선택)
})

# 환경 변수로 API Key 관리
os.environ["OPENAI_API_KEY"] = "sk-xxx"
```

**모니터링**:
- 수동 확인으로 충분
- 터미널 출력으로 토큰 수 확인

---

### Medium Scale (100-1K req/day)

**특징**:
- 프로덕션 초기
- Rate limit 주의 필요
- 비용 $50-500/month

**권장 설정**:
```python
from openai import OpenAI

client = OpenAI(
    api_key=os.getenv("OPENAI_API_KEY"),
    timeout=30.0,  # 타임아웃 설정
    max_retries=2  # 리트라이
)

vn = MyVanna(
    client=client,
    config={
        "temperature": 0.0,
        "model": "gpt-3.5-turbo"  # 명시적 지정
    }
)
```

**Rate Limit 대응**:
```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=4, max=60)
)
def generate_sql_with_retry(question):
    return vn.generate_sql(question)
```

**비용 모니터링**:
```python
# 토큰 수 로깅
import logging
logging.basicConfig(
    filename="vanna_usage.log",
    level=logging.INFO,
    format="%(asctime)s - %(message)s"
)

# submit_prompt 출력 캡처
# 일일 집계 후 비용 계산
```

---

### Large Scale (1K+ req/day)

**특징**:
- 대규모 프로덕션
- 비용 최적화 필수
- 비용 $500+/month

**권장 설정**:

#### 1. Tier-based 모델 선택
```python
class SmartVanna(ChromaDB_VectorStore, OpenAI_Chat):
    def generate_sql(self, question, **kwargs):
        complexity = self.estimate_complexity(question)

        if complexity == "simple":
            kwargs["model"] = "gpt-3.5-turbo"
        elif complexity == "complex":
            kwargs["model"] = "gpt-4"

        return super().generate_sql(question, **kwargs)

    def estimate_complexity(self, question):
        # JOIN 개수, 서브쿼리, 집계 함수 등으로 판단
        if "join" in question.lower() or "aggregate" in question.lower():
            return "complex"
        return "simple"
```

#### 2. 캐싱 레이어 추가
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

**캐시 Hit Rate**:
- 일반적으로 20-40% 캐시 히트
- 비용 20-40% 절감

#### 3. Batch Processing
```python
# 동일 시간대 요청 배치 처리
from asyncio import gather

async def batch_generate_sql(questions):
    tasks = [vn.generate_sql_async(q) for q in questions]
    return await gather(*tasks)

# Rate limit 공유로 효율 증가
```

#### 4. 비용 모니터링 (Prometheus + Grafana)
```python
from prometheus_client import Counter, Histogram

token_counter = Counter('vanna_tokens_total', 'Total tokens used', ['model'])
latency_histogram = Histogram('vanna_latency_seconds', 'Request latency')

class MonitoredVanna(MyVanna):
    def submit_prompt(self, prompt, **kwargs):
        with latency_histogram.time():
            response = super().submit_prompt(prompt, **kwargs)

        # 토큰 수 계산
        num_tokens = sum(len(m["content"]) / 4 for m in prompt)
        model = kwargs.get("model", "gpt-3.5-turbo")
        token_counter.labels(model=model).inc(num_tokens)

        return response
```

**알림 설정**:
```yaml
# Prometheus alerting rule
groups:
  - name: vanna_cost
    rules:
      - alert: HighTokenUsage
        expr: rate(vanna_tokens_total[1h]) > 100000
        annotations:
          summary: "High token usage detected"
```

#### 5. Azure OpenAI로 마이그레이션

**이유**:
- 엔터프라이즈 SLA
- 더 높은 rate limit
- Region별 배포

```python
from openai import AzureOpenAI

client = AzureOpenAI(
    api_key=os.getenv("AZURE_OPENAI_KEY"),
    azure_endpoint=os.getenv("AZURE_OPENAI_ENDPOINT"),
    api_version="2024-02-01"
)

vn = MyVanna(
    client=client,
    config={"engine": "my-gpt-35-deployment"}
)
```

**Rate Limit 비교**:
| Service | TPM (Tokens Per Minute) | RPM (Requests Per Minute) |
|---------|--------------------------|----------------------------|
| OpenAI Free | 40K | 200 |
| OpenAI Paid (Tier 1) | 60K | 500 |
| Azure OpenAI | 120K+ | Configurable |

---

## 핵심 학습

### 1. Mixin 패턴의 실전 적용

OpenAI_Chat은 VannaBase를 상속하지만, 독립적으로 사용되지 않습니다. 반드시 Vector Store와 함께 조합됩니다:

```python
# 이렇게 사용 안 함
vn = OpenAI_Chat()  # Vector Store 없음 → 불완전

# 이렇게 사용
class MyVanna(ChromaDB_VectorStore, OpenAI_Chat):
    pass
```

**핵심 원칙**:
- 각 Mixin은 하나의 책임만 (SRP)
- Mixin 간 의존성 없음
- Base class로 조합

### 2. Client 주입 패턴

테스트 용이성을 위해 Client를 주입받습니다:

```python
def __init__(self, client=None, config=None):
    if client is not None:
        self.client = client
        return
    # 아니면 자동 생성
```

**장점**:
- Mock Client로 테스트 가능
- Timeout, Retry 등 세밀한 제어
- 여러 Vanna 인스턴스가 Client 공유 가능

### 3. Deprecation 전략

기존 API를 제거하는 방법:
1. **즉시 에러**: 명확한 메시지로 마이그레이션 가이드
2. **문서화**: 새 API 사용법 명시
3. **타이밍**: LLM SDK 업데이트와 동시 (OpenAI v1.0.0)

```python
if "api_type" in config:
    raise Exception("Passing api_type is now deprecated. Please pass an OpenAI client instead.")
```

**효과**:
- 사용자가 즉시 마이그레이션
- 미래에 코드 제거 용이
- 기술 부채 방지

### 4. 자동 vs 수동의 균형

토큰 수에 따라 자동으로 모델을 선택하지만, 사용자가 명시적으로 지정할 수도 있습니다:

```python
# 자동 (편리함)
sql = vn.generate_sql(question)

# 수동 (제어)
sql = vn.generate_sql(question, model="gpt-4")
```

**설계 원칙**:
- **기본값은 자동** (80% 케이스 커버)
- **옵션으로 수동 제어** (20% 엣지 케이스)
- **우선순위 명확** (kwargs > config > default)

### 5. OpenAI API의 실전 Tip

#### Token 계산 정확도
```python
num_tokens = len(message["content"]) / 4  # 근사치
```

- 실제: tiktoken 라이브러리 사용
- 하지만 근사치로도 모델 선택에는 충분
- 정확도 ±10% 내외

#### Temperature 가이드
- **0.0**: 결정적 (SQL 생성)
- **0.3-0.5**: 약간의 변형 (차트 코드)
- **0.7**: 기본값 (균형)
- **0.9+**: 창의적 (문서 생성)

#### Model vs Engine
- **model**: OpenAI API
- **engine**: Azure OpenAI API
- 동일한 모델이지만 파라미터명만 다름

---

## 요약

| 측면 | 내용 |
|------|------|
| **핵심 역할** | VannaBase의 LLM 인터페이스를 OpenAI로 구현 |
| **주요 문제** | 1) LLM 독립성 2) Client 초기화 3) 자동 모델 선택 4) Azure 호환성 5) Message 포맷 |
| **해결 방법** | 추상 메서드 구현 + Client 주입 + 토큰 기반 자동 선택 |
| **핵심 패턴** | Mixin 패턴, Dependency Injection, Template Method |
| **확장성** | 다른 LLM(Anthropic, Local)도 동일 패턴으로 구현 가능 |
| **비용 최적화** | 토큰 수 기반 모델 선택으로 30-50% 비용 절감 |
| **프로덕션 고려** | Rate limit, 캐싱, 모니터링, Tier-based 선택 |
| **테스트 용이성** | Client 주입으로 Mock 테스트 가능 |
| **대표 코드** | `submit_prompt()` - 우선순위 체인 + 자동 모델 선택 |

**한 줄 요약**: OpenAI API를 추상화하여 LLM 독립적인 Text-to-SQL 시스템을 구현하는 Mixin 클래스입니다.
