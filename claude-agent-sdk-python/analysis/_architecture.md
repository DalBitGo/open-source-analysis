# claude-agent-sdk-python Architecture Deep Dive

**분석 대상**: claude-agent-sdk-python v0.1.1
**분석일**: 2025-01-09
**분석 관점**: Operability, Simplicity, Evolvability

---

## 📐 전체 아키텍처 개요

### 계층적 구조 (Layered Architecture)

```
┌─────────────────────────────────────────────────────────────┐
│                    Public API Layer                         │
│  ┌────────────┐  ┌──────────────────┐  ┌────────────────┐  │
│  │  query()   │  │ ClaudeSDKClient  │  │ create_sdk_    │  │
│  │            │  │                  │  │ mcp_server()   │  │
│  └────────────┘  └──────────────────┘  └────────────────┘  │
│       ↓                  ↓                      ↓           │
└───────┼──────────────────┼──────────────────────┼───────────┘
        │                  │                      │
┌───────┼──────────────────┼──────────────────────┼───────────┐
│       │      Internal Implementation Layer      │           │
│       ↓                  ↓                      ↓           │
│  ┌─────────────────────────────────────────────────────┐   │
│  │         InternalClient / Query                      │   │
│  │  - Control protocol handling                        │   │
│  │  - Hook management                                  │   │
│  │  - Permission callbacks                             │   │
│  │  - MCP SDK server routing                           │   │
│  └─────────────────────────────────────────────────────┘   │
│       ↓                                                     │
│  ┌─────────────────────────────────────────────────────┐   │
│  │         MessageParser                               │   │
│  │  - JSON Lines parsing                               │   │
│  │  - Type conversion                                  │   │
│  └─────────────────────────────────────────────────────┘   │
└───────────────────────────┼─────────────────────────────────┘
                            ↓
┌───────────────────────────┼─────────────────────────────────┐
│          Transport Layer (Abstraction)                      │
│  ┌─────────────────────────────────────────────────────┐   │
│  │         Transport (Abstract Protocol)               │   │
│  │  - connect(), write(), read()                       │   │
│  └─────────────────────────────────────────────────────┘   │
│                            ↓                                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │      SubprocessCLITransport (Implementation)        │   │
│  │  - Process management                               │   │
│  │  - Stdin/Stdout/Stderr handling                     │   │
│  │  - CLI command building                             │   │
│  └─────────────────────────────────────────────────────┘   │
└───────────────────────────┼─────────────────────────────────┘
                            ↓
┌───────────────────────────┼─────────────────────────────────┐
│                   External Process                          │
│  ┌─────────────────────────────────────────────────────┐   │
│  │         Claude Code CLI (Node.js)                   │   │
│  │  - LLM API calls                                    │   │
│  │  - Tool execution                                   │   │
│  │  - MCP server management                            │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## 🧩 핵심 컴포넌트 분석

### 1. Public API Layer

#### 1.1 `query()` 함수
**파일**: `src/claude_agent_sdk/query.py`

**책임**:
- 단방향, 상태 비저장 쿼리 인터페이스
- InternalClient로 요청 위임

**특징**:
```python
async def query(
    prompt: str | AsyncIterable[dict[str, Any]],
    options: ClaudeAgentOptions | None = None,
    transport: Transport | None = None,
) -> AsyncIterator[Message]:
```

**설계 패턴**:
- **Facade**: 복잡한 내부 구현 숨김
- **Iterator**: `AsyncIterator[Message]` 스트리밍
- **Dependency Injection**: `transport` 파라미터로 구현 교체 가능

**평가**:
- ✅ Simplicity: 최소 3줄 코드로 사용 가능
- ✅ Evolvability: Transport 추상화로 확장 가능
- ⚠️ Operability: 에러 처리는 호출자 책임

---

#### 1.2 `ClaudeSDKClient` 클래스
**파일**: `src/claude_agent_sdk/client.py`

**책임**:
- 양방향, 상태 유지 대화 관리
- Async context manager 지원
- 인터럽트, 권한 모드 변경 등 제어 기능

**핵심 메서드**:
```python
class ClaudeSDKClient:
    async def connect(prompt: str | AsyncIterable | None) -> None
    async def query(prompt: str | AsyncIterable, session_id: str) -> None
    async def receive_messages() -> AsyncIterator[Message]
    async def receive_response() -> AsyncIterator[Message]
    async def interrupt() -> None
    async def set_permission_mode(mode: str) -> None
    async def set_model(model: str | None) -> None
    async def disconnect() -> None
```

**설계 패턴**:
- **Context Manager**: `async with` 지원으로 자원 관리
- **Adapter**: Internal Query를 사용자 친화적 API로 변환
- **Streaming**: `AsyncIterator`로 실시간 응답

**라이프사이클**:
```
__aenter__()
    ↓
connect() → Transport.connect() → Query.start() → Query.initialize()
    ↓
query() / receive_messages() / interrupt()
    ↓
__aexit__()
    ↓
disconnect() → Query.close()
```

**평가**:
- ✅ Operability: 명확한 라이프사이클, 컨텍스트 매니저
- ✅ Simplicity: 직관적인 메서드명
- ⚠️ Evolvability: `_query`에 대한 강한 의존성

---

#### 1.3 MCP SDK Server 시스템
**파일**: `src/claude_agent_sdk/__init__.py`

**구성 요소**:
```python
@tool(name, description, input_schema)  # Decorator
def create_sdk_mcp_server(name, version, tools)  # Factory
class SdkMcpTool  # Tool definition
```

**아키텍처**:
```
User Function (Python)
    ↓ @tool decorator
SdkMcpTool[T]
    ↓ create_sdk_mcp_server()
MCP Server (In-Process)
    ↓ mcp_servers config
Query (SDK MCP routing)
    ↓ JSON-RPC
Claude Code CLI
```

**혁신적인 설계**:
1. **In-Process 실행**: 별도 프로세스 없음
2. **타입 안전성**: Generic `SdkMcpTool[T]`
3. **간단한 API**: Decorator 패턴
4. **MCP 표준 준수**: `mcp` 라이브러리 활용

**코드 예시** (from `__init__.py:124-277`):
```python
# 1. Tool 정의
@tool("add", "Add numbers", {"a": float, "b": float})
async def add(args):
    return {"content": [{"type": "text", "text": f"{args['a'] + args['b']}"}]}

# 2. MCP Server 생성
server = create_sdk_mcp_server("calc", tools=[add])

# 3. SDK에 등록
options = ClaudeAgentOptions(
    mcp_servers={"calc": server},
    allowed_tools=["mcp__calc__add"]
)
```

**내부 구현** (JSON Schema 자동 변환):
```python
# types.py → JSON Schema
{"a": float, "b": float}
    ↓
{
    "type": "object",
    "properties": {
        "a": {"type": "number"},
        "b": {"type": "number"}
    },
    "required": ["a", "b"]
}
```

**평가**:
- ✅✅ Simplicity: Decorator 패턴으로 극도로 단순
- ✅✅ Evolvability: MCP 표준 기반, 확장 가능
- ✅ Operability: In-process라 디버깅 쉬움

---

### 2. Internal Implementation Layer

#### 2.1 `InternalClient` / `Query`
**파일**: `src/claude_agent_sdk/_internal/client.py`, `_internal/query.py`

**책임**:
- **Control Protocol 처리**: CLI와 SDK 간 양방향 통신
- **Hook 관리**: PreToolUse, PostToolUse 등
- **Permission Callback**: `can_use_tool` 실행
- **SDK MCP 라우팅**: In-process MCP 서버 호출

**Control Protocol 흐름**:
```
CLI → SDK Control Request (JSON-RPC)
    ↓
Query.handle_control_request()
    ↓ (분기)
    ├─ can_use_tool → Permission callback 실행
    ├─ hook_callback → Hook 함수 실행
    ├─ mcp_message → SDK MCP 서버 호출
    └─ initialize → 초기화 응답
    ↓
SDK Control Response (JSON-RPC)
    ↓
CLI
```

**코드 구조** (추정, 파일 미열람):
```python
class Query:
    async def handle_control_request(request: SDKControlRequest):
        if request["request"]["subtype"] == "can_use_tool":
            result = await self._can_use_tool(...)
            return {"subtype": "success", "response": result}
        elif request["request"]["subtype"] == "hook_callback":
            result = await self._execute_hook(...)
            return {"subtype": "success", "response": result}
        # ...
```

**평가**:
- ✅ Operability: 명확한 프로토콜, 에러 처리
- ✅ Simplicity: Public API와 완전히 분리
- ⚠️ Evolvability: Control protocol 변경 시 호환성 문제 가능

---

#### 2.2 `MessageParser`
**파일**: `src/claude_agent_sdk/_internal/message_parser.py`

**책임**:
- JSON Lines 파싱
- Dict → Dataclass 변환 (`UserMessage`, `AssistantMessage` 등)

**구조** (추정):
```python
def parse_message(data: dict[str, Any]) -> Message:
    msg_type = data.get("type")
    if msg_type == "user":
        return UserMessage(...)
    elif msg_type == "assistant":
        return AssistantMessage(...)
    elif msg_type == "system":
        return SystemMessage(...)
    # ...
```

**ContentBlock 파싱**:
```python
content_blocks = []
for block in data["content"]:
    if block["type"] == "text":
        content_blocks.append(TextBlock(text=block["text"]))
    elif block["type"] == "tool_use":
        content_blocks.append(ToolUseBlock(...))
```

**평가**:
- ✅ Simplicity: 단순한 변환 로직
- ⚠️ Operability: JSON 스키마 검증 부재 (추정)
- ✅ Evolvability: 새로운 메시지 타입 추가 용이

---

### 3. Transport Layer

#### 3.1 `Transport` (Abstract)
**파일**: `src/claude_agent_sdk/_internal/transport/__init__.py`

**인터페이스** (추정):
```python
class Transport(Protocol):
    async def connect() -> None
    async def write(data: str) -> None
    async def read() -> AsyncIterator[str]
    async def close() -> None
```

**설계 패턴**:
- **Abstract Interface**: 구현체 교체 가능
- **Dependency Inversion**: 상위 레이어가 Transport에 의존

**평가**:
- ✅✅ Evolvability: 구현체 교체 용이 (예: WebSocket, gRPC)
- ✅ Simplicity: 최소 인터페이스
- ✅ Operability: 명확한 책임

---

#### 3.2 `SubprocessCLITransport`
**파일**: `src/claude_agent_sdk/_internal/transport/subprocess_cli.py`

**책임**:
- Claude Code CLI 프로세스 관리
- Stdin/Stdout/Stderr 스트림 핸들링
- CLI 명령어 빌드
- 프로세스 라이프사이클 관리

**핵심 메서드**:
```python
class SubprocessCLITransport(Transport):
    def _find_cli() -> str  # CLI 바이너리 찾기
    def _build_command() -> list[str]  # 명령어 생성
    async def connect() -> None  # 프로세스 시작
    async def write(data: str) -> None  # Stdin 쓰기
    async def read() -> AsyncIterator[str]  # Stdout 읽기
    async def _handle_stderr() -> None  # Stderr 처리
```

**CLI 명령어 빌드 예시** (from line 86-200):
```python
def _build_command(self) -> list[str]:
    cmd = [self._cli_path, "--output-format", "stream-json", "--verbose"]

    if self._options.system_prompt:
        cmd.extend(["--system-prompt", self._options.system_prompt])

    if self._options.allowed_tools:
        cmd.extend(["--allowedTools", ",".join(self._options.allowed_tools)])

    if self._options.mcp_servers:
        # SDK MCP: instance 필드 제거
        servers_for_cli = {
            name: {k: v for k, v in config.items() if k != "instance"}
            for name, config in self._options.mcp_servers.items()
            if config.get("type") == "sdk"
        }
        cmd.extend(["--mcp-config", json.dumps({"mcpServers": servers_for_cli})])

    if self._is_streaming:
        cmd.extend(["--input-format", "stream-json"])
    else:
        cmd.extend(["--print", "--", str(self._prompt)])

    return cmd
```

**프로세스 관리**:
```
_find_cli()
    ↓
_build_command()
    ↓
anyio.open_process(cmd, stdin=PIPE, stdout=PIPE, stderr=PIPE)
    ↓
TextReceiveStream(stdout) + TextSendStream(stdin)
    ↓
_handle_stderr() (별도 태스크)
    ↓
read() / write() 스트리밍
    ↓
close() → process.terminate()
```

**에러 처리**:
- `CLINotFoundError`: CLI 바이너리 없음
- `ProcessError`: 프로세스 종료 코드 != 0
- `CLIConnectionError`: 연결 실패
- `CLIJSONDecodeError`: JSON 파싱 실패

**평가**:
- ✅ Operability: 상세한 에러 타입, 프로세스 관리
- ✅ Simplicity: 명확한 책임 분리
- ✅ Evolvability: Transport 인터페이스 준수

---

## 🔄 데이터 흐름 분석

### 단방향 쿼리 (`query()`)

```
User Code
    ↓
query(prompt="Hello")
    ↓
InternalClient.process_query()
    ↓
SubprocessCLITransport.connect()
    ↓ spawn
Claude Code CLI Process
    ↓ stdin
"--print -- Hello"
    ↓ stdout (JSON Lines)
{"type": "assistant", "content": [...]}
    ↓
MessageParser.parse_message()
    ↓
AssistantMessage(content=[TextBlock("Hi!")])
    ↓
yield to user
```

---

### 양방향 대화 (`ClaudeSDKClient`)

```
User Code
    ↓
async with ClaudeSDKClient() as client:
    ↓
connect()
    ↓
SubprocessCLITransport.connect()
    ↓
Query.start() + Query.initialize()
    ↓ (Control Protocol)
CLI → {"type": "control_request", "request": {"subtype": "initialize"}}
    ↓
Query → {"type": "control_response", "response": {"subtype": "success"}}
    ↓
client.query("Hello")
    ↓
Transport.write({"type": "user", "message": "Hello"})
    ↓ stdin
CLI processes
    ↓ stdout
{"type": "assistant", ...}
    ↓
Query.receive_messages()
    ↓
MessageParser
    ↓
client.receive_response()
    ↓
yield AssistantMessage
```

---

### SDK MCP 도구 호출

```
User defines @tool
    ↓
create_sdk_mcp_server(tools=[...])
    ↓
MCP Server instance (in-process)
    ↓
ClaudeAgentOptions(mcp_servers={"srv": server})
    ↓
Query stores sdk_mcp_servers={...}
    ↓
CLI → {"type": "control_request", "request": {"subtype": "mcp_message"}}
    ↓
Query.handle_mcp_message()
    ↓
MCP Server.call_tool(name, args)
    ↓
User's @tool function executes
    ↓
Return result
    ↓
Query → {"type": "control_response", "response": {...}}
    ↓
CLI receives result
```

---

## 🎯 아키텍처 평가

### Operability (운영성) - ⭐⭐⭐⭐☆

**강점**:
1. **명확한 에러 계층**
   ```python
   ClaudeSDKError (Base)
   ├─ CLIConnectionError
   │  └─ CLINotFoundError
   ├─ ProcessError (exit_code, stderr 포함)
   ├─ CLIJSONDecodeError
   └─ MessageParseError
   ```

2. **프로세스 관리**
   - 자동 재시작 (추정)
   - Stderr 모니터링
   - 타임아웃 처리

3. **타입 안전성**
   - mypy strict 모드
   - Dataclass 기반 메시지
   - TypedDict로 런타임 검증 최소화

**개선점**:
- ⚠️ 로깅 시스템 부재 (명시적 logger 사용 제한적)
- ⚠️ 메트릭/모니터링 없음
- ⚠️ Retry 정책 불명확

---

### Simplicity (단순성) - ⭐⭐⭐⭐⭐

**강점**:
1. **레이어 분리**
   ```
   Public API (3개 핵심)
       ↓
   Internal (사용자 몰라도 됨)
       ↓
   Transport (교체 가능)
   ```

2. **Progressive Disclosure**
   - 레벨 1: `query("Hello")` (3줄)
   - 레벨 2: `ClaudeAgentOptions` (설정)
   - 레벨 3: `ClaudeSDKClient` (양방향)
   - 레벨 4: `@tool` (커스텀 도구)
   - 레벨 5: `hooks` (세밀한 제어)

3. **일관된 네이밍**
   - `ClaudeAgentOptions`, `ClaudeSDKClient`
   - `UserMessage`, `AssistantMessage`
   - `TextBlock`, `ToolUseBlock`

4. **Dataclass 활용**
   - 보일러플레이트 최소화
   - IDE 자동완성 지원

---

### Evolvability (발전성) - ⭐⭐⭐⭐⭐

**강점**:
1. **Transport 추상화**
   ```python
   # 현재: Subprocess CLI
   # 미래: WebSocket, gRPC, Direct API
   transport = CustomTransport()
   query(prompt="...", transport=transport)
   ```

2. **MCP 표준 활용**
   - External MCP + SDK MCP 혼용
   - 표준 프로토콜로 확장 무한

3. **Control Protocol**
   - 새로운 Hook 타입 추가 가능
   - Request/Response 확장 가능

4. **Public/Internal 분리**
   - `_internal/` 리팩터링 시 Public API 영향 없음

5. **TypedDict + Literal**
   ```python
   PermissionMode = Literal["default", "acceptEdits", "plan", ...]
   # 새로운 모드 추가 시 타입 체크
   ```

**설계 결정**:
- ✅ **Dependency Injection**: Transport, options
- ✅ **Interface Segregation**: Transport, Query 분리
- ✅ **Open/Closed**: MCP 서버 확장 가능

---

## 🔧 설계 패턴 정리

| 패턴 | 위치 | 목적 |
|------|------|------|
| **Facade** | `query()`, `ClaudeSDKClient` | 복잡성 숨김 |
| **Adapter** | `SubprocessCLITransport` | CLI ↔ Python 어댑터 |
| **Iterator** | `AsyncIterator[Message]` | 스트리밍 응답 |
| **Factory** | `create_sdk_mcp_server()` | MCP 서버 생성 |
| **Decorator** | `@tool` | 도구 정의 간소화 |
| **Context Manager** | `async with ClaudeSDKClient()` | 자원 관리 |
| **Protocol** | `Transport` | 인터페이스 추상화 |
| **Dependency Injection** | `transport` 파라미터 | 구현체 교체 |
| **Template Method** | `Query.handle_control_request()` | 공통 흐름, 확장 포인트 |

---

## 🚀 확장 시나리오

### 1. WebSocket Transport 추가
```python
class WebSocketTransport(Transport):
    async def connect(self):
        self._ws = await websockets.connect("wss://claude-api")

    async def write(self, data: str):
        await self._ws.send(data)

    async def read(self):
        async for msg in self._ws:
            yield msg

# 사용
transport = WebSocketTransport()
async for msg in query(prompt="...", transport=transport):
    print(msg)
```

### 2. 새로운 Hook 타입 추가
```python
# types.py
HookEvent = (
    ...
    | Literal["PreAPICall"]  # 새 Hook!
)

# Query에서 자동으로 처리됨
```

### 3. 커스텀 메시지 타입
```python
@dataclass
class DebugMessage:
    debug_info: dict[str, Any]

Message = ... | DebugMessage

# MessageParser에서 처리
def parse_message(data):
    if data["type"] == "debug":
        return DebugMessage(debug_info=data["info"])
```

---

## 💡 아키텍처 인사이트

### 1. **프로세스 간 통신 (IPC) 패턴**
- Python SDK ↔ Node.js CLI
- JSON Lines 프로토콜
- Bidirectional streaming (stdin/stdout)
- Control protocol (Request/Response)

### 2. **In-Process MCP 혁신**
- 기존: Python → Subprocess MCP Server → CLI
- 개선: Python (MCP Server in-process) → CLI
- 성능 향상, 디버깅 용이

### 3. **타입 안전성 우선 설계**
- Runtime 검증 최소화 (TypedDict)
- Compile-time 검증 (mypy strict)
- IDE 지원 극대화

### 4. **계층적 추상화**
```
Simplicity (Public API)
    ↓
Complexity (Internal)
    ↓
Flexibility (Transport)
```

### 5. **Progressive Disclosure 철학**
- 초보자: 3줄로 시작
- 중급자: Options로 세밀한 제어
- 고급자: Transport, Hook 커스터마이징

---

## 📊 의존성 분석

### External Dependencies
```python
anyio>=4.0.0           # 비동기 I/O 추상화 (asyncio + trio)
mcp>=0.1.0             # MCP 프로토콜 표준
typing_extensions      # 타입 힌트 (Python < 3.11)
```

### Internal Module Dependencies
```
types.py (독립적)
    ↓
_errors.py (독립적)
    ↓
__init__.py (MCP 서버)
    ↓ uses
types.py + mcp
    ↓
_internal/transport/subprocess_cli.py
    ↓ uses
_errors.py + types.py + anyio
    ↓
_internal/query.py + _internal/client.py
    ↓ uses
Transport + MessageParser
    ↓
query.py + client.py (Public API)
    ↓ uses
InternalClient + Transport
```

**특징**:
- ✅ 순환 의존성 없음
- ✅ 명확한 계층 구조
- ✅ `types.py`가 최하단 (의존성 없음)

---

## 🎓 학습 포인트

### 1. SDK 설계 원칙
- **간단한 시작, 복잡한 옵션**: `query()` vs `ClaudeSDKClient`
- **Public/Internal 분리**: `_internal/`로 구현 숨김
- **Transport 추상화**: 다양한 백엔드 지원

### 2. Python 비동기 패턴
- `anyio` 사용으로 asyncio + trio 동시 지원
- `AsyncIterator`로 스트리밍
- Context manager (`async with`)

### 3. 프로세스 간 통신
- JSON Lines 프로토콜
- Control protocol (Request/Response)
- Stderr 모니터링

### 4. 타입 시스템 활용
- Dataclass: 간결한 데이터 구조
- TypedDict: 딕셔너리 타입 검증
- Literal: 열거형 타입
- Generic: 타입 안전한 도구 시스템

---

## 🔮 미래 개선 방향

### 1. 로깅/모니터링 강화
```python
import structlog

logger = structlog.get_logger()

async def query(...):
    logger.info("query_started", prompt=prompt[:50])
    # ...
    logger.info("query_completed", duration=elapsed)
```

### 2. Retry 정책
```python
@dataclass
class ClaudeAgentOptions:
    retry_policy: RetryPolicy | None = None

@dataclass
class RetryPolicy:
    max_retries: int = 3
    backoff: float = 1.0
```

### 3. Metrics
```python
from prometheus_client import Counter, Histogram

query_counter = Counter("claude_queries_total")
query_duration = Histogram("claude_query_duration_seconds")
```

### 4. Connection Pooling
```python
class TransportPool:
    async def acquire() -> Transport
    async def release(transport: Transport)
```

---

**작성**: Claude Code
**분석 프레임워크**: Operability, Simplicity, Evolvability
