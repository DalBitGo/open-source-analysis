# client.py

**원본 경로**: `source/src/claude_agent_sdk/client.py`
**역할**: 양방향 대화를 위한 ClaudeSDKClient 클래스 제공
**라인 수**: 336줄
**의존성**: `Transport`, `types`, `_errors`, `_internal.query`, `_internal.message_parser`

---

## 📊 구조 개요

- **클래스**: 1개 (`ClaudeSDKClient`)
- **Public 메서드**: 10개
- **Private 메서드**: 1개
- **Magic 메서드**: 2개 (`__aenter__`, `__aexit__`)
- **복잡도**: 중간 (상태 관리, 양방향 통신)

---

## 🔍 상세 분석

### 클래스: `ClaudeSDKClient`

**라인**: 14-336

#### 책임 (Responsibilities)
1. **양방향 대화 관리**: 사용자 ↔ Claude 실시간 통신
2. **세션 라이프사이클**: connect → query/receive → disconnect
3. **Control Protocol 위임**: Internal Query에 제어 명령 전달
4. **스트리밍 관리**: AsyncIterator로 메시지 스트리밍
5. **Context Manager**: `async with` 자원 관리

---

## 📋 메서드 상세 분석

### 1. `__init__()`
**라인**: 55-68

```python
def __init__(
    self,
    options: ClaudeAgentOptions | None = None,
    transport: Transport | None = None,
):
    if options is None:
        options = ClaudeAgentOptions()
    self.options = options
    self._custom_transport = transport
    self._transport: Transport | None = None
    self._query: Any | None = None
    os.environ["CLAUDE_CODE_ENTRYPOINT"] = "sdk-py-client"
```

**책임**:
- 옵션 초기화 (기본값 제공)
- Transport 저장 (DI 지원)
- 환경변수 설정 (원격 측정용)

**상태 변수**:
- `options`: 설정 저장
- `_custom_transport`: 사용자 제공 Transport
- `_transport`: 실제 사용 중인 Transport
- `_query`: Internal Query 인스턴스

**설계 평가**:
- ✅ **Simplicity**: 기본값으로 쉬운 시작
- ✅ **Evolvability**: `transport` DI로 확장 가능
- ⚠️ **Operability**: `_query: Any` - 타입 안전성 부족

---

### 2. `_convert_hooks_to_internal_format()`
**라인**: 69-83

```python
def _convert_hooks_to_internal_format(
    self, hooks: dict[HookEvent, list[HookMatcher]]
) -> dict[str, list[dict[str, Any]]]:
    """Convert HookMatcher format to internal Query format."""
    internal_hooks: dict[str, list[dict[str, Any]]] = {}
    for event, matchers in hooks.items():
        internal_hooks[event] = []
        for matcher in matchers:
            internal_matcher = {
                "matcher": matcher.matcher if hasattr(matcher, "matcher") else None,
                "hooks": matcher.hooks if hasattr(matcher, "hooks") else [],
            }
            internal_hooks[event].append(internal_matcher)
    return internal_hooks
```

**책임**: Public API 타입 → Internal 타입 변환

**변환**:
```python
# Input (Public API)
{
    "PreToolUse": [
        HookMatcher(matcher="Bash", hooks=[callback1, callback2])
    ]
}

# Output (Internal)
{
    "PreToolUse": [
        {
            "matcher": "Bash",
            "hooks": [callback1, callback2]
        }
    ]
}
```

**설계 패턴**: **Adapter Pattern**

**설계 평가**:
- ✅ **Simplicity**: 단순한 변환 로직
- ⚠️ **Operability**: `hasattr()` 사용 - Duck typing (런타임 검증)
- ✅ **Evolvability**: Public/Internal 타입 독립적

---

### 3. `connect()`
**라인**: 85-159

```python
async def connect(
    self, prompt: str | AsyncIterable[dict[str, Any]] | None = None
) -> None:
    """Connect to Claude with a prompt or message stream."""
```

**핵심 로직**:

#### 3.1 Empty Stream 생성
**라인**: 93-99

```python
async def _empty_stream() -> AsyncIterator[dict[str, Any]]:
    # Never yields, but indicates that this function is an iterator and
    # keeps the connection open.
    return
    yield {}  # type: ignore[unreachable]

actual_prompt = _empty_stream() if prompt is None else prompt
```

**목적**: `prompt=None`일 때 연결만 열고 대기

**트릭**:
- `return` 후 `yield` → 영원히 실행 안 되지만 타입은 `AsyncIterator`
- 연결은 유지, 메시지는 나중에 `query()`로 전송

---

#### 3.2 Permission 설정 검증
**라인**: 103-122

```python
if self.options.can_use_tool:
    # canUseTool callback requires streaming mode
    if isinstance(prompt, str):
        raise ValueError(
            "can_use_tool callback requires streaming mode. "
            "Please provide prompt as an AsyncIterable instead of a string."
        )

    # canUseTool and permission_prompt_tool_name are mutually exclusive
    if self.options.permission_prompt_tool_name:
        raise ValueError(
            "can_use_tool callback cannot be used with permission_prompt_tool_name. "
            "Please use one or the other."
        )

    # Automatically set permission_prompt_tool_name to "stdio" for control protocol
    options = replace(self.options, permission_prompt_tool_name="stdio")
else:
    options = self.options
```

**검증 규칙**:
1. `can_use_tool` 사용 시 → 스트리밍 모드 필수
2. `can_use_tool`과 `permission_prompt_tool_name` 상호 배타적
3. `can_use_tool` 사용 시 → 자동으로 `permission_prompt_tool_name="stdio"` 설정

**설계 평가**:
- ✅✅ **Operability**: 명확한 에러 메시지
- ✅ **Simplicity**: 자동 설정으로 사용자 편의
- ✅ **Evolvability**: 검증 로직 중앙화

---

#### 3.3 Transport 생성 및 연결
**라인**: 124-132

```python
# Use provided custom transport or create subprocess transport
if self._custom_transport:
    self._transport = self._custom_transport
else:
    self._transport = SubprocessCLITransport(
        prompt=actual_prompt,
        options=options,
    )
await self._transport.connect()
```

**설계 패턴**: **Dependency Injection + Factory**

---

#### 3.4 SDK MCP 서버 추출
**라인**: 134-140

```python
sdk_mcp_servers = {}
if self.options.mcp_servers and isinstance(self.options.mcp_servers, dict):
    for name, config in self.options.mcp_servers.items():
        if isinstance(config, dict) and config.get("type") == "sdk":
            sdk_mcp_servers[name] = config["instance"]
```

**목적**: In-process MCP 서버 인스턴스 분리

**흐름**:
```
ClaudeAgentOptions.mcp_servers
    ↓ Filter
sdk_mcp_servers = {"calc": <MCP Server instance>}
    ↓ Pass to
Query(sdk_mcp_servers=...)
    ↓ Route
MCP message → In-process server.call_tool()
```

---

#### 3.5 Query 생성 및 초기화
**라인**: 142-158

```python
self._query = Query(
    transport=self._transport,
    is_streaming_mode=True,  # ClaudeSDKClient always uses streaming mode
    can_use_tool=self.options.can_use_tool,
    hooks=self._convert_hooks_to_internal_format(self.options.hooks)
    if self.options.hooks
    else None,
    sdk_mcp_servers=sdk_mcp_servers,
)

# Start reading messages and initialize
await self._query.start()
await self._query.initialize()

# If we have an initial prompt stream, start streaming it
if prompt is not None and isinstance(prompt, AsyncIterable) and self._query._tg:
    self._query._tg.start_soon(self._query.stream_input, prompt)
```

**라이프사이클**:
1. `Query` 인스턴스 생성
2. `start()` → 메시지 읽기 태스크 시작
3. `initialize()` → Control protocol 초기화
4. `stream_input()` → 초기 프롬프트 스트리밍 (백그라운드)

**설계 평가**:
- ✅ **Operability**: 명확한 초기화 단계
- ⚠️ **Complexity**: `_query._tg` 접근 (캡슐화 위반)
- ✅ **Evolvability**: Query에 제어 위임

---

### 4. `receive_messages()`
**라인**: 160-168

```python
async def receive_messages(self) -> AsyncIterator[Message]:
    """Receive all messages from Claude."""
    if not self._query:
        raise CLIConnectionError("Not connected. Call connect() first.")

    from ._internal.message_parser import parse_message

    async for data in self._query.receive_messages():
        yield parse_message(data)
```

**책임**:
- Internal Query에서 raw dict 수신
- `parse_message()`로 typed Message 변환
- AsyncIterator로 스트리밍

**데이터 흐름**:
```
CLI (JSON)
    ↓
Transport.read() → str
    ↓
Query.receive_messages() → dict[str, Any]
    ↓
parse_message() → Message (AssistantMessage, UserMessage, ...)
    ↓
yield to user
```

**설계 패턴**: **Iterator + Adapter**

**설계 평가**:
- ✅ **Simplicity**: 간단한 변환 로직
- ✅ **Operability**: 연결 상태 검증
- ✅ **Evolvability**: parse_message는 확장 가능

---

### 5. `query()`
**라인**: 170-198

```python
async def query(
    self, prompt: str | AsyncIterable[dict[str, Any]], session_id: str = "default"
) -> None:
    """Send a new request in streaming mode."""
    if not self._query or not self._transport:
        raise CLIConnectionError("Not connected. Call connect() first.")

    # Handle string prompts
    if isinstance(prompt, str):
        message = {
            "type": "user",
            "message": {"role": "user", "content": prompt},
            "parent_tool_use_id": None,
            "session_id": session_id,
        }
        await self._transport.write(json.dumps(message) + "\n")
    else:
        # Handle AsyncIterable prompts - stream them
        async for msg in prompt:
            # Ensure session_id is set on each message
            if "session_id" not in msg:
                msg["session_id"] = session_id
            await self._transport.write(json.dumps(msg) + "\n")
```

**책임**:
1. 프롬프트 타입 처리 (str vs AsyncIterable)
2. JSON Lines 형식으로 직렬화
3. Transport에 쓰기

**메시지 형식**:
```json
{
  "type": "user",
  "message": {
    "role": "user",
    "content": "Hello Claude"
  },
  "parent_tool_use_id": null,
  "session_id": "default"
}
```

**설계 평가**:
- ✅ **Simplicity**: 간단한 JSON 변환
- ✅ **Operability**: session_id 자동 설정
- ⚠️ **Type Safety**: dict 구조 하드코딩 (타입 검증 없음)

---

### 6. `interrupt()`
**라인**: 200-204

```python
async def interrupt(self) -> None:
    """Send interrupt signal (only works with streaming mode)."""
    if not self._query:
        raise CLIConnectionError("Not connected. Call connect() first.")
    await self._query.interrupt()
```

**책임**: Query에 인터럽트 신호 전달

**사용 예시**:
```python
async with ClaudeSDKClient() as client:
    await client.query("Generate 1000 files")
    # User cancels
    await client.interrupt()
```

---

### 7. `set_permission_mode()`
**라인**: 206-228

```python
async def set_permission_mode(self, mode: str) -> None:
    """Change permission mode during conversation."""
    if not self._query:
        raise CLIConnectionError("Not connected. Call connect() first.")
    await self._query.set_permission_mode(mode)
```

**사용 예시** (from docstring):
```python
async with ClaudeSDKClient() as client:
    # Start with default permissions
    await client.query("Help me analyze this codebase")

    # Review mode done, switch to auto-accept edits
    await client.set_permission_mode('acceptEdits')
    await client.query("Now implement the fix we discussed")
```

**설계 평가**:
- ✅ **Operability**: 동적 권한 변경 가능
- ✅ **Simplicity**: Query에 위임

---

### 8. `set_model()`
**라인**: 230-252

```python
async def set_model(self, model: str | None = None) -> None:
    """Change the AI model during conversation."""
```

**사용 예시**:
```python
await client.set_model('claude-sonnet-4-5')
```

---

### 9. `get_server_info()`
**라인**: 254-277

```python
async def get_server_info(self) -> dict[str, Any] | None:
    """Get server initialization info including available commands and output styles."""
    if not self._query:
        raise CLIConnectionError("Not connected. Call connect() first.")
    return getattr(self._query, "_initialization_result", None)
```

**반환 정보**:
- Available commands (slash commands)
- Output styles
- Server capabilities

**설계 평가**:
- ⚠️ **Encapsulation**: `getattr(self._query, "_initialization_result")` - 내부 변수 접근
- ✅ **Operability**: 서버 정보 노출로 디버깅 가능

---

### 10. `receive_response()`
**라인**: 279-318

```python
async def receive_response(self) -> AsyncIterator[Message]:
    """
    Receive messages from Claude until and including a ResultMessage.

    This async iterator yields all messages in sequence and automatically terminates
    after yielding a ResultMessage (which indicates the response is complete).
    """
    async for message in self.receive_messages():
        yield message
        if isinstance(message, ResultMessage):
            return
```

**차이점**: `receive_messages()` vs `receive_response()`

| 메서드 | 종료 조건 | 사용 사례 |
|--------|----------|----------|
| `receive_messages()` | 명시적 중단 필요 | 연속 대화, 무한 스트림 |
| `receive_response()` | `ResultMessage` 수신 시 자동 종료 | 단일 응답, 간단한 쿼리 |

**사용 예시**:
```python
async with ClaudeSDKClient() as client:
    await client.query("What's 2+2?")

    # Option 1: receive_response (자동 종료)
    async for msg in client.receive_response():
        print(msg)
        # ResultMessage 수신 후 자동 종료

    # Option 2: receive_messages (수동 제어)
    async for msg in client.receive_messages():
        print(msg)
        if should_stop:
            break
```

**설계 평가**:
- ✅✅ **Simplicity**: 일반적인 사용 사례 간소화
- ✅ **Evolvability**: `receive_messages()` 기반, 확장 가능

---

### 11. `disconnect()`
**라인**: 320-325

```python
async def disconnect(self) -> None:
    """Disconnect from Claude."""
    if self._query:
        await self._query.close()
        self._query = None
    self._transport = None
```

**책임**:
- Query 종료
- 상태 초기화

**설계 평가**:
- ✅ **Operability**: 명확한 정리 로직
- ⚠️ **Resource Management**: Transport 명시적 close 없음 (Query가 처리 추정)

---

### 12. Context Manager (`__aenter__` / `__aexit__`)
**라인**: 327-335

```python
async def __aenter__(self) -> "ClaudeSDKClient":
    """Enter async context - automatically connects with empty stream for interactive use."""
    await self.connect()
    return self

async def __aexit__(self, exc_type: Any, exc_val: Any, exc_tb: Any) -> bool:
    """Exit async context - always disconnects."""
    await self.disconnect()
    return False
```

**사용 패턴**:
```python
async with ClaudeSDKClient(options=...) as client:
    await client.query("Hello")
    async for msg in client.receive_response():
        print(msg)
# 자동으로 disconnect() 호출
```

**설계 패턴**: **Context Manager (Resource Acquisition Is Initialization)**

**설계 평가**:
- ✅✅ **Operability**: 자원 누수 방지
- ✅✅ **Simplicity**: Pythonic, 명확한 라이프사이클
- ✅ **Evolvability**: 표준 Python 패턴

---

## 🔄 상태 다이어그램

```
[초기화]
    ↓ __init__()
[생성됨]
    ↓ connect() / __aenter__()
[연결됨]
    ↓ query(), receive_messages(), interrupt(), set_*()
[대화 중]
    ↓ disconnect() / __aexit__()
[종료됨]
```

**상태 전이 조건**:
- `connect()` 호출 전 → `CLIConnectionError`
- `disconnect()` 후 다시 사용 → `CLIConnectionError`

---

## 💡 설계 평가

### Operability (운영성) - ⭐⭐⭐⭐☆

**강점**:
1. ✅ **명확한 에러 메시지**:
   ```python
   "can_use_tool callback requires streaming mode. "
   "Please provide prompt as an AsyncIterable instead of a string."
   ```

2. ✅ **상태 검증**:
   ```python
   if not self._query:
       raise CLIConnectionError("Not connected. Call connect() first.")
   ```

3. ✅ **Context Manager**: 자원 자동 정리

4. ✅ **동적 제어**: `set_permission_mode()`, `set_model()`, `interrupt()`

**개선점**:
- ⚠️ `_query: Any` - 타입 안전성 부족
- ⚠️ `getattr(self._query, "_initialization_result")` - 캡슐화 위반
- ⚠️ Transport 명시적 close 없음

---

### Simplicity (단순성) - ⭐⭐⭐⭐⭐

**강점**:
1. ✅ **Progressive API**:
   ```python
   # 간단한 시작
   async with ClaudeSDKClient() as client:
       await client.query("Hello")
       async for msg in client.receive_response():
           print(msg)

   # 고급 기능
   client = ClaudeSDKClient(options=ClaudeAgentOptions(
       can_use_tool=my_callback,
       hooks={...}
   ))
   ```

2. ✅ **명확한 메서드명**:
   - `connect()`, `query()`, `receive_messages()`
   - `interrupt()`, `set_permission_mode()`

3. ✅ **기본값 제공**:
   ```python
   options = options or ClaudeAgentOptions()
   ```

4. ✅ **두 가지 수신 패턴**:
   - `receive_messages()`: 무한 스트림
   - `receive_response()`: 단일 응답

---

### Evolvability (발전성) - ⭐⭐⭐⭐⭐

**강점**:
1. ✅ **Dependency Injection**:
   ```python
   ClaudeSDKClient(transport=CustomTransport())
   ```

2. ✅ **Internal Query 위임**:
   - Public API는 안정적
   - Internal 구현 교체 가능

3. ✅ **Hook/Callback 확장성**:
   - `can_use_tool`, `hooks` 시스템
   - 새로운 Hook 타입 추가 용이

4. ✅ **Protocol 기반 설계**:
   ```python
   self._transport: Transport  # 구현체 교체 가능
   ```

---

## 🔧 설계 패턴

| 패턴 | 위치 | 목적 |
|------|------|------|
| **Facade** | `ClaudeSDKClient` | Internal Query 숨김 |
| **Adapter** | `_convert_hooks_to_internal_format()` | Public ↔ Internal 타입 변환 |
| **Context Manager** | `__aenter__` / `__aexit__` | 자원 관리 |
| **Dependency Injection** | `__init__(transport)` | 구현체 교체 |
| **Iterator** | `receive_messages()` | 스트리밍 |
| **Template Method** | `receive_response()` | 일반적 패턴 캡슐화 |
| **Delegation** | 대부분 메서드 | Query에 제어 위임 |

---

## 🎯 주요 인사이트

### 1. **Facade + Delegation 패턴**
```
ClaudeSDKClient (Simple, Public)
    ↓ delegates
Internal Query (Complex, Hidden)
    ↓ uses
Transport, MessageParser, Control Protocol
```

**이점**:
- Public API 안정적
- Internal 리팩터링 자유
- 테스트 용이

---

### 2. **Progressive Disclosure**
```python
# 레벨 1: 기본 사용
async with ClaudeSDKClient() as client:
    await client.query("Hello")

# 레벨 2: 옵션 추가
client = ClaudeSDKClient(options=ClaudeAgentOptions(...))

# 레벨 3: 커스텀 Transport
client = ClaudeSDKClient(transport=MyTransport())

# 레벨 4: Hook/Callback
client = ClaudeSDKClient(options=ClaudeAgentOptions(
    can_use_tool=my_callback,
    hooks={...}
))
```

---

### 3. **자동 설정 (Convention over Configuration)**
```python
if self.options.can_use_tool:
    # 자동으로 permission_prompt_tool_name="stdio" 설정
    options = replace(self.options, permission_prompt_tool_name="stdio")
```

**이점**:
- 사용자 실수 방지
- 보일러플레이트 감소

---

### 4. **두 가지 수신 패턴**
- `receive_messages()`: Low-level, 완전 제어
- `receive_response()`: High-level, 간편 사용

**설계 원칙**: **Provide both low-level and high-level APIs**

---

### 5. **Empty Stream 트릭**
```python
async def _empty_stream() -> AsyncIterator[dict[str, Any]]:
    return
    yield {}  # 영원히 실행 안 됨
```

**목적**: 타입은 `AsyncIterator`지만 실제로는 아무것도 yield 안 함
**용도**: 연결만 열고 나중에 메시지 전송

---

## 🚀 개선 제안

### 1. 타입 안전성 강화
```python
# Before
self._query: Any | None = None

# After
from ._internal.query import Query
self._query: Query | None = None
```

### 2. Transport 명시적 close
```python
async def disconnect(self) -> None:
    if self._query:
        await self._query.close()
    if self._transport:
        await self._transport.close()  # ← 추가
    self._query = None
    self._transport = None
```

### 3. 상태 관리 명시화
```python
from enum import Enum, auto

class ClientState(Enum):
    CREATED = auto()
    CONNECTED = auto()
    DISCONNECTED = auto()

class ClaudeSDKClient:
    def __init__(self):
        self._state = ClientState.CREATED

    async def connect(self):
        if self._state != ClientState.CREATED:
            raise ValueError(f"Cannot connect from state {self._state}")
        # ...
        self._state = ClientState.CONNECTED
```

### 4. 메시지 직렬화 타입화
```python
# Before (dict 하드코딩)
message = {
    "type": "user",
    "message": {"role": "user", "content": prompt},
    ...
}

# After (TypedDict 사용)
class UserMessageProtocol(TypedDict):
    type: Literal["user"]
    message: dict[str, str]
    parent_tool_use_id: str | None
    session_id: str

message: UserMessageProtocol = {
    "type": "user",
    "message": {"role": "user", "content": prompt},
    ...
}
```

---

## 📊 의존성 관계

```
ClaudeSDKClient
├─ depends on → Transport (Abstract)
├─ depends on → ClaudeAgentOptions (types.py)
├─ depends on → Query (_internal/query.py)
├─ depends on → MessageParser (_internal/message_parser.py)
├─ depends on → SubprocessCLITransport (default)
└─ depends on → _errors (CLIConnectionError)
```

---

## 🎓 학습 포인트

### 1. Facade Pattern 실제 적용
- Public API 단순화
- Internal 복잡성 숨김
- 테스트 용이성

### 2. Context Manager Best Practice
- `async with` 지원
- 자원 자동 정리
- 예외 안전성

### 3. Progressive Disclosure
- 기본 사용 쉽게
- 고급 기능 점진적 노출

### 4. Delegation over Inheritance
- Query에 제어 위임
- 상속보다 조합 선호

---

**분석 작성**: Claude Code
**분석 프레임워크**: Operability, Simplicity, Evolvability
