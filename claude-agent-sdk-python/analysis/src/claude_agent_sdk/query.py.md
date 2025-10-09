# query.py

**원본 경로**: `source/src/claude_agent_sdk/query.py`
**역할**: 단방향 쿼리를 위한 `query()` 함수 제공
**라인 수**: 127줄
**의존성**: `InternalClient`, `Transport`, `types`

---

## 📊 구조 개요

- **함수**: 1개 (`query()`)
- **클래스**: 0개
- **복잡도**: 낮음 (단순한 Facade)
- **패턴**: Facade, Iterator

---

## 🔍 상세 분석

### 함수: `query()`

**라인**: 12-127

```python
async def query(
    *,
    prompt: str | AsyncIterable[dict[str, Any]],
    options: ClaudeAgentOptions | None = None,
    transport: Transport | None = None,
) -> AsyncIterator[Message]:
```

---

## 📋 파라미터 분석

### 1. `prompt`
**타입**: `str | AsyncIterable[dict[str, Any]]`

**두 가지 모드**:

#### 모드 1: 단순 문자열 (One-shot)
```python
async for msg in query(prompt="What is 2+2?"):
    print(msg)
```

**특징**:
- Fire-and-forget
- 단일 질문-응답
- 가장 간단한 사용

---

#### 모드 2: 스트리밍 (Continuous)
```python
async def prompts():
    yield {"type": "user", "message": {"role": "user", "content": "Hello"}}
    yield {"type": "user", "message": {"role": "user", "content": "How are you?"}}

async for msg in query(prompt=prompts()):
    print(msg)
```

**특징**:
- 여러 메시지 전송 가능
- 하지만 여전히 **단방향**
- 응답 기반 추가 질문 불가

**중요한 차이**:
```python
# query() - 단방향
async def prompts():
    yield msg1
    yield msg2  # msg1 응답 전에 미리 정의됨

# ClaudeSDKClient - 양방향
await client.query("Hello")
response = await client.receive_response()
if "error" in response:
    await client.query("Try again")  # 응답 기반 결정
```

---

### 2. `options`
**타입**: `ClaudeAgentOptions | None`
**기본값**: `None` → `ClaudeAgentOptions()`

**예시**:
```python
async for msg in query(
    prompt="Create a file",
    options=ClaudeAgentOptions(
        allowed_tools=["Write"],
        permission_mode="acceptEdits",
        cwd="/tmp"
    )
):
    print(msg)
```

---

### 3. `transport`
**타입**: `Transport | None`
**기본값**: `None` → `SubprocessCLITransport`

**사용 사례**: 커스텀 Transport 구현

```python
class MyTransport(Transport):
    async def connect(self): ...
    async def write(self, data: str): ...
    async def read(self) -> AsyncIterator[str]: ...

async for msg in query(
    prompt="Hello",
    transport=MyTransport()
):
    print(msg)
```

**설계 패턴**: **Dependency Injection**

---

## 🔄 함수 내부 구조

### 1. 기본값 설정
**라인**: 116-117

```python
if options is None:
    options = ClaudeAgentOptions()
```

**설계 평가**:
- ✅ **Simplicity**: 기본값으로 쉬운 시작
- ✅ **Convention over Configuration**

---

### 2. 환경변수 설정
**라인**: 119

```python
os.environ["CLAUDE_CODE_ENTRYPOINT"] = "sdk-py"
```

**목적**: 원격 측정(Telemetry)

**값**:
- `"sdk-py"`: `query()` 함수 사용
- `"sdk-py-client"`: `ClaudeSDKClient` 사용 (client.py:67)

**설계 평가**:
- ✅ **Operability**: 사용 패턴 추적 가능
- ⚠️ **Side Effect**: 전역 상태 변경

---

### 3. InternalClient에 위임
**라인**: 121-126

```python
client = InternalClient()

async for message in client.process_query(
    prompt=prompt, options=options, transport=transport
):
    yield message
```

**설계 패턴**: **Facade Pattern**

**책임 분리**:
```
query() (Public API)
    ↓ Facade
InternalClient.process_query() (Internal)
    ↓ Uses
Transport, Query, MessageParser
```

---

## 📖 Docstring 분석

### 핵심 설명 (라인 18-44)

**강조하는 차이점**:

| 특징 | `query()` | `ClaudeSDKClient` |
|------|-----------|-------------------|
| **방향성** | Unidirectional | Bidirectional |
| **상태** | Stateless | Stateful |
| **복잡도** | Simple | Complex |
| **인터럽트** | ✗ | ✓ |
| **후속 메시지** | ✗ | ✓ |

**사용 시기 (When to use)**:

#### `query()` 사용
- ✅ Simple one-off questions
- ✅ Batch processing of independent prompts
- ✅ Code generation or analysis tasks
- ✅ Automated scripts and CI/CD pipelines
- ✅ When you know all inputs upfront

#### `ClaudeSDKClient` 사용
- ✅ Interactive conversations with follow-ups
- ✅ Chat applications or REPL-like interfaces
- ✅ When you need to send messages based on responses
- ✅ When you need interrupt capabilities
- ✅ Long-running sessions with state

---

### 예시 (라인 68-113)

#### 예시 1: 단순 쿼리
```python
async for message in query(prompt="What is the capital of France?"):
    print(message)
```

---

#### 예시 2: 옵션 포함
```python
async for message in query(
    prompt="Create a Python web server",
    options=ClaudeAgentOptions(
        system_prompt="You are an expert Python developer",
        cwd="/home/user/project"
    )
):
    print(message)
```

---

#### 예시 3: 스트리밍 모드
```python
async def prompts():
    yield {"type": "user", "message": {"role": "user", "content": "Hello"}}
    yield {"type": "user", "message": {"role": "user", "content": "How are you?"}}

# All prompts are sent, then all responses received
async for message in query(prompt=prompts()):
    print(message)
```

**주석 강조**:
```python
# All prompts are sent, then all responses received
```

**의미**: 여전히 단방향! 응답 기반 추가 질문 불가

---

#### 예시 4: 커스텀 Transport
```python
from claude_agent_sdk import query, Transport

class MyCustomTransport(Transport):
    # Implement custom transport logic
    pass

transport = MyCustomTransport()
async for message in query(
    prompt="Hello",
    transport=transport
):
    print(message)
```

---

## 💡 설계 평가

### Operability (운영성) - ⭐⭐⭐☆☆

**강점**:
1. ✅ **환경변수 설정**: 원격 측정 지원
2. ✅ **기본값 제공**: 에러 방지

**약점**:
1. ⚠️ **에러 처리 없음**: 모두 InternalClient에 위임
2. ⚠️ **전역 상태 변경**: `os.environ` 직접 수정
3. ⚠️ **로깅 없음**: 디버깅 어려움

**개선 제안**:
```python
import logging

logger = logging.getLogger(__name__)

async def query(...) -> AsyncIterator[Message]:
    logger.info("query_started", prompt_type=type(prompt).__name__)
    try:
        # ...
        yield message
    except Exception as e:
        logger.error("query_failed", error=str(e))
        raise
    finally:
        logger.info("query_completed")
```

---

### Simplicity (단순성) - ⭐⭐⭐⭐⭐

**강점**:
1. ✅✅ **최소주의**: 단 1개 함수
2. ✅✅ **명확한 책임**: Facade 역할만
3. ✅ **기본값**: `options=None` 처리
4. ✅ **타입 힌트**: 명확한 시그니처

**코드 라인 수**:
- 실제 로직: **5줄**
- Docstring: **100줄+**
- 비율: 문서 20배!

**평가**: 극도로 단순, 문서가 코드보다 중요

---

### Evolvability (발전성) - ⭐⭐⭐⭐⭐

**강점**:
1. ✅ **Transport DI**: 구현체 교체 가능
2. ✅ **InternalClient 위임**: Public API 안정적
3. ✅ **타입 확장**: `prompt` 타입 추가 가능

**확장 시나리오**:

#### 시나리오 1: 새로운 Transport
```python
class WebSocketTransport(Transport):
    async def connect(self): ...
    async def write(self, data: str): ...
    async def read(self) -> AsyncIterator[str]: ...

# query() 코드 변경 없음!
async for msg in query(prompt="...", transport=WebSocketTransport()):
    print(msg)
```

#### 시나리오 2: InternalClient 개선
```python
# _internal/client.py 리팩터링
class InternalClient:
    async def process_query(self, ...):
        # 새로운 최적화, 캐싱, 재시도 로직 추가
        # ...

# query() 코드 변경 없음!
```

#### 시나리오 3: 새로운 prompt 타입
```python
# 미래
PromptType = str | AsyncIterable[dict] | StructuredPrompt

async def query(
    *,
    prompt: PromptType,  # ← 타입만 확장
    ...
):
    # InternalClient가 처리
```

---

## 🔧 설계 패턴

| 패턴 | 구현 | 목적 |
|------|------|------|
| **Facade** | `query() → InternalClient` | 복잡성 숨김 |
| **Iterator** | `AsyncIterator[Message]` | 스트리밍 |
| **Dependency Injection** | `transport` 파라미터 | 확장성 |
| **Default Parameter** | `options=None` | 편의성 |

---

## 🎯 주요 인사이트

### 1. **극도의 단순성**
```python
# 전체 로직
if options is None:
    options = ClaudeAgentOptions()

os.environ["CLAUDE_CODE_ENTRYPOINT"] = "sdk-py"

client = InternalClient()
async for message in client.process_query(...):
    yield message
```

**5줄로 끝!**

---

### 2. **Facade의 완벽한 예시**
```
User Code
    ↓
query() (5줄)
    ↓
InternalClient (수백 줄)
    ↓
Transport, Query, MessageParser (수천 줄)
```

**이점**:
- Public API 안정적
- 내부 리팩터링 자유
- 테스트 용이

---

### 3. **단방향 vs 양방향 명확히 구분**

Docstring에서 강조:

> **Unidirectional**: Send all messages upfront, receive all responses
>
> **Stateless**: Each query is independent, no conversation state

**철학**: 올바른 도구 선택 유도

---

### 4. **문서가 코드보다 중요**

- 코드: 5줄
- Docstring: 100줄+
- 예시: 4개

**의미**: API 사용법이 구현보다 중요

---

### 5. **환경변수로 Telemetry**

```python
os.environ["CLAUDE_CODE_ENTRYPOINT"] = "sdk-py"
```

**추정 사용**:
```
CLI → 환경변수 읽음 → 통계 전송
- query() 사용률
- ClaudeSDKClient 사용률
```

---

## 🔄 데이터 흐름

```
User Code
    ↓
query(prompt="Hello", options=...)
    ↓
options = options or ClaudeAgentOptions()
    ↓
os.environ["CLAUDE_CODE_ENTRYPOINT"] = "sdk-py"
    ↓
InternalClient()
    ↓
client.process_query(prompt, options, transport)
    ↓ (내부)
Transport.connect()
    ↓
Claude Code CLI spawn
    ↓
Stdin: prompt
    ↓
Stdout: JSON Lines
    ↓
MessageParser.parse_message()
    ↓
yield Message to user
```

---

## 🚀 개선 제안

### 1. 로깅 추가
```python
import logging

logger = logging.getLogger(__name__)

async def query(...) -> AsyncIterator[Message]:
    logger.debug(
        "query_started",
        prompt_type=type(prompt).__name__,
        has_options=options is not None,
        has_transport=transport is not None,
    )

    client = InternalClient()
    message_count = 0

    async for message in client.process_query(...):
        message_count += 1
        logger.debug(f"message_received", type=type(message).__name__)
        yield message

    logger.info("query_completed", message_count=message_count)
```

---

### 2. Context Manager 지원
```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def query_context(
    prompt: str | AsyncIterable[dict[str, Any]],
    options: ClaudeAgentOptions | None = None,
    transport: Transport | None = None,
):
    """Context manager version of query() for resource management."""
    if options is None:
        options = ClaudeAgentOptions()

    os.environ["CLAUDE_CODE_ENTRYPOINT"] = "sdk-py-context"

    client = InternalClient()

    try:
        async for message in client.process_query(...):
            yield message
    finally:
        # Cleanup
        pass

# 사용
async with query_context(prompt="Hello") as messages:
    async for msg in messages:
        print(msg)
```

---

### 3. 타입 검증
```python
async def query(
    *,
    prompt: str | AsyncIterable[dict[str, Any]],
    options: ClaudeAgentOptions | None = None,
    transport: Transport | None = None,
) -> AsyncIterator[Message]:
    if options is None:
        options = ClaudeAgentOptions()

    # 타입 검증
    if not isinstance(prompt, (str, AsyncIterable)):
        raise TypeError(f"prompt must be str or AsyncIterable, got {type(prompt)}")

    # ...
```

---

### 4. 에러 래핑
```python
from ._errors import QueryError

async def query(...) -> AsyncIterator[Message]:
    try:
        # ...
        async for message in client.process_query(...):
            yield message
    except CLINotFoundError:
        raise  # Re-raise as-is
    except Exception as e:
        raise QueryError(f"Query failed: {e}") from e
```

---

## 📊 비교: `query()` vs `ClaudeSDKClient`

| 특징 | `query()` | `ClaudeSDKClient` |
|------|-----------|-------------------|
| **코드 라인** | 5줄 | 300+줄 |
| **복잡도** | ⭐ | ⭐⭐⭐⭐ |
| **사용 사례** | One-shot, Batch | Interactive, REPL |
| **상태** | Stateless | Stateful |
| **방향** | 단방향 | 양방향 |
| **인터럽트** | ✗ | ✓ |
| **세션 제어** | ✗ | ✓ (set_permission_mode, etc.) |
| **Context Manager** | ✗ | ✓ |
| **시작 쉬움** | ✅✅ | ✅ |
| **유연성** | ⭐⭐ | ⭐⭐⭐⭐⭐ |

**선택 기준**:
- 간단한 작업 → `query()`
- 대화형, 복잡한 제어 → `ClaudeSDKClient`

---

## 🎓 학습 포인트

### 1. Facade Pattern의 가치
- Public API: 극도로 단순
- Internal: 복잡성 모두 숨김
- 사용자 경험 ↑

### 2. Progressive Disclosure
```python
# 레벨 1: 최소
query(prompt="Hello")

# 레벨 2: 옵션
query(prompt="Hello", options=ClaudeAgentOptions(...))

# 레벨 3: 커스텀 Transport
query(prompt="Hello", transport=MyTransport())

# 레벨 4: 양방향 필요 → ClaudeSDKClient
```

### 3. 문서 중심 설계
- 코드 5줄, 문서 100줄
- 예시 4개
- **API는 사용자가 이해하는 게 중요**

### 4. 올바른 추상화 수준
- `query()`: 단순 작업
- `ClaudeSDKClient`: 복잡한 작업
- **하나의 도구로 모든 걸 하려 하지 않음**

---

## 📝 요약

`query()` 함수는:
- ✅ **극도로 단순** (5줄 코드)
- ✅ **명확한 목적** (단방향 쿼리)
- ✅ **완벽한 Facade** (복잡성 숨김)
- ✅ **확장 가능** (Transport DI)
- ✅ **문서 중심** (100줄 docstring)

**설계 철학**:
> "Make simple things simple, and complex things possible"

- Simple: `query(prompt="Hello")`
- Complex: `ClaudeSDKClient`

---

**분석 작성**: Claude Code
**분석 프레임워크**: Operability, Simplicity, Evolvability
