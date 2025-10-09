# Internal Implementation Analysis

## 📋 개요

`_internal/` 디렉토리는 SDK의 실제 구현 로직을 담고 있으며, Public API의 단순함 뒤에 숨겨진 복잡성을 처리합니다.

---

## 🔧 핵심 컴포넌트

### 1. InternalClient (`_internal/client.py`)

**역할**: Public API(`query()`, `ClaudeSDKClient`)를 실제 구현으로 연결하는 **Orchestrator**

#### 주요 책임

```python
class InternalClient:
    async def process_query(
        prompt: str | AsyncIterable[dict],
        options: ClaudeAgentOptions,
        transport: Transport | None = None
    ) -> AsyncIterator[Message]:
```

**구현 로직:**

1. **옵션 검증 및 변환**
   - `can_use_tool` 콜백 사용 시 스트리밍 모드 강제
   - `can_use_tool`과 `permission_prompt_tool_name` 상호 배타적 검증
   - 자동으로 `permission_prompt_tool_name="stdio"` 설정 (제어 프로토콜용)

2. **Transport 관리**
   - 외부 Transport 제공 시 사용
   - 없으면 `SubprocessCLITransport` 자동 생성
   - `await transport.connect()` 호출

3. **SDK MCP 서버 추출**
   ```python
   # mcp_servers에서 type="sdk"인 서버만 필터링
   sdk_mcp_servers = {}
   for name, config in options.mcp_servers.items():
       if config.get("type") == "sdk":
           sdk_mcp_servers[name] = config["instance"]
   ```

4. **Query 객체 생성 및 실행**
   ```python
   query = Query(
       transport=transport,
       is_streaming_mode=not isinstance(prompt, str),
       can_use_tool=options.can_use_tool,
       hooks=self._convert_hooks_to_internal_format(options.hooks),
       sdk_mcp_servers=sdk_mcp_servers
   )

   await query.start()
   if is_streaming: await query.initialize()

   # 스트리밍 입력은 백그라운드 태스크로
   if isinstance(prompt, AsyncIterable):
       query._tg.start_soon(query.stream_input, prompt)

   # 메시지 파싱 및 yield
   async for data in query.receive_messages():
       yield parse_message(data)
   ```

#### 설계 특징

- **검증 레이어**: 옵션 조합의 유효성 검증 (TypeScript SDK 로직과 동일)
- **자동 구성**: 개발자가 명시하지 않은 설정 자동 추가
- **리소스 관리**: try/finally로 cleanup 보장

---

### 2. Query (`_internal/query.py`)

**역할**: Transport 위에서 **양방향 제어 프로토콜** 처리

#### 핵심 기능

##### A. 제어 프로토콜 라우팅

```python
async def _read_messages(self):
    async for message in self.transport.read_messages():
        msg_type = message.get("type")

        if msg_type == "control_response":
            # SDK → CLI로 보낸 요청의 응답
            self._handle_control_response(message)

        elif msg_type == "control_request":
            # CLI → SDK로 보낸 요청
            self._tg.start_soon(self._handle_control_request, message)

        else:
            # 일반 SDK 메시지 (사용자에게 전달)
            await self._message_send.send(message)
```

**메시지 흐름:**
```
┌─────────────────────────────────────────┐
│  Python SDK (Query)                     │
│                                         │
│  ┌──────────────┐   ┌──────────────┐   │
│  │ SDK → CLI    │   │ CLI → SDK    │   │
│  │ (outgoing)   │   │ (incoming)   │   │
│  │              │   │              │   │
│  │ - initialize │   │ - can_use_   │   │
│  │ - interrupt  │   │   tool       │   │
│  │ - set_model  │   │ - hook_      │   │
│  │              │   │   callback   │   │
│  │              │   │ - mcp_       │   │
│  │              │   │   message    │   │
│  └──────────────┘   └──────────────┘   │
└─────────────────────────────────────────┘
           ↕ JSON Lines
┌─────────────────────────────────────────┐
│  Claude Code CLI (Node.js)              │
└─────────────────────────────────────────┘
```

##### B. 훅(Hook) 시스템

**초기화 시 훅 등록:**

```python
async def initialize(self):
    hooks_config = {}
    for event, matchers in self.hooks.items():
        for matcher in matchers:
            callback_ids = []
            for callback in matcher["hooks"]:
                callback_id = f"hook_{self.next_callback_id}"
                self.hook_callbacks[callback_id] = callback  # 콜백 저장
                callback_ids.append(callback_id)

            hooks_config[event].append({
                "matcher": matcher["matcher"],
                "hookCallbackIds": callback_ids
            })

    # CLI에 훅 구성 전송
    await self._send_control_request({
        "subtype": "initialize",
        "hooks": hooks_config
    })
```

**훅 실행 (CLI → SDK):**

```python
elif subtype == "hook_callback":
    callback_id = request_data["callback_id"]
    callback = self.hook_callbacks[callback_id]

    response = await callback(
        request_data["input"],
        request_data["tool_use_id"],
        {"signal": None}
    )

    # 응답을 CLI로 전송
    await self.transport.write(json.dumps(response))
```

##### C. 도구 권한 콜백

```python
async def _handle_control_request(self, request):
    if subtype == "can_use_tool":
        context = ToolPermissionContext(
            signal=None,
            suggestions=request_data.get("permission_suggestions", [])
        )

        result = await self.can_use_tool(
            request_data["tool_name"],
            request_data["input"],
            context
        )

        # PermissionResultAllow | PermissionResultDeny
        if isinstance(result, PermissionResultAllow):
            response = {"behavior": "allow"}
            if result.updated_input: response["updatedInput"] = result.updated_input
            if result.updated_permissions: response["updatedPermissions"] = [...]
        else:
            response = {"behavior": "deny", "message": result.message}
```

##### D. SDK MCP 서버 브리지

**문제**: Python MCP SDK는 TypeScript와 달리 `Transport` 추상화가 없음

**해결책**: JSONRPC 메서드를 수동 라우팅

```python
async def _handle_sdk_mcp_request(self, server_name: str, message: dict):
    server = self.sdk_mcp_servers[server_name]
    method = message["method"]

    if method == "initialize":
        return {
            "jsonrpc": "2.0",
            "result": {
                "protocolVersion": "2024-11-05",
                "capabilities": {"tools": {}},
                "serverInfo": {"name": server.name, "version": server.version}
            }
        }

    elif method == "tools/list":
        handler = server.request_handlers[ListToolsRequest]
        result = await handler(ListToolsRequest(method=method))
        return {"jsonrpc": "2.0", "result": {"tools": [...]}}

    elif method == "tools/call":
        handler = server.request_handlers[CallToolRequest]
        result = await handler(CallToolRequest(...))
        return {"jsonrpc": "2.0", "result": {"content": [...]}}
```

**제약**: TypeScript는 `server.connect(transport)`로 자동 처리, Python은 수동 라우팅 필요

#### 동시성 관리

```python
# anyio TaskGroup 사용
self._tg = anyio.create_task_group()
await self._tg.__aenter__()

# 백그라운드 태스크
self._tg.start_soon(self._read_messages)
self._tg.start_soon(self.stream_input, prompt)

# 요청-응답 동기화
self.pending_control_responses[request_id] = anyio.Event()
await event.wait()  # 타임아웃 60초
```

---

### 3. Message Parser (`_internal/message_parser.py`)

**역할**: 원시 JSON을 타입 안전한 Python 객체로 변환

#### 파싱 로직

```python
def parse_message(data: dict[str, Any]) -> Message:
    match data["type"]:
        case "user":
            # 단순 문자열 또는 ContentBlock 리스트
            content = []
            for block in data["message"]["content"]:
                match block["type"]:
                    case "text": content.append(TextBlock(text=block["text"]))
                    case "tool_use": content.append(ToolUseBlock(...))
                    case "tool_result": content.append(ToolResultBlock(...))
            return UserMessage(content=content, ...)

        case "assistant":
            # TextBlock | ThinkingBlock | ToolUseBlock | ToolResultBlock
            content = []
            for block in data["message"]["content"]:
                match block["type"]:
                    case "text": content.append(TextBlock(...))
                    case "thinking": content.append(ThinkingBlock(...))
                    case "tool_use": content.append(ToolUseBlock(...))
            return AssistantMessage(content=content, model=data["model"], ...)

        case "system":
            return SystemMessage(subtype=data["subtype"], data=data)

        case "result":
            return ResultMessage(
                duration_ms=data["duration_ms"],
                usage=data["usage"],
                total_cost_usd=data["total_cost_usd"],
                ...
            )

        case "stream_event":
            return StreamEvent(uuid=data["uuid"], event=data["event"], ...)
```

#### 에러 처리

```python
# 타입 검증
if not isinstance(data, dict):
    raise MessageParseError(f"Expected dict, got {type(data)}")

# 필수 필드 검증
try:
    return AssistantMessage(content=..., model=data["message"]["model"])
except KeyError as e:
    raise MessageParseError(f"Missing field in assistant message: {e}", data)
```

#### 설계 특징

- **패턴 매칭**: Python 3.10+ match 구문 활용
- **타입 안전성**: TypedDict 기반 Message 타입
- **명확한 에러**: 원본 데이터 포함하여 디버깅 용이

---

### 4. Error Hierarchy (`_errors.py`)

```python
ClaudeSDKError (base)
├── CLIConnectionError
│   └── CLINotFoundError         # Claude Code CLI 미설치
├── ProcessError                 # 서브프로세스 실패 (exit code + stderr)
├── CLIJSONDecodeError          # JSON 파싱 실패 (원본 라인 포함)
└── MessageParseError           # 메시지 타입 변환 실패 (원본 데이터 포함)
```

#### 특징

- **컨텍스트 보존**: 각 에러가 원인 데이터 저장
- **계층적 catch**: `except ClaudeSDKError`로 모든 SDK 에러 처리 가능
- **진단 정보**: exit code, stderr, 원본 JSON 등

---

## 🔄 전체 실행 흐름

### 단방향 모드 (String Prompt)

```python
query("Hello Claude")

↓ (Public API)

InternalClient.process_query(prompt="Hello Claude", ...)
  1. Transport 생성 (SubprocessCLITransport)
  2. Query 생성 (is_streaming_mode=False)
  3. query.start() → _read_messages 백그라운드 태스크 시작
  4. query.receive_messages() → 메시지 yield
  5. parse_message(data) → Message 객체 변환

↓ (Transport Layer)

SubprocessCLITransport
  - CLI 프로세스 시작: claude-code agent --prompt "Hello Claude"
  - stdout에서 JSON Lines 읽기
  - stdin 즉시 닫기 (입력 없음)

↓ (CLI)

Claude Code CLI (Node.js)
  - Anthropic API 호출
  - 응답을 JSON Lines로 stdout 출력
  - 프로세스 종료
```

### 양방향 모드 (AsyncIterable Prompt)

```python
async with ClaudeSDKClient() as client:
    await client.query("Hello")
    async for msg in client.receive_response():
        print(msg)

↓

InternalClient.process_query(prompt=AsyncIterable, ...)
  1. Query 생성 (is_streaming_mode=True)
  2. query.start()
  3. query.initialize() → CLI와 핸드셰이크
     - 훅 등록
     - 지원 기능 협상
  4. query.stream_input(prompt) → 백그라운드로 입력 스트리밍
  5. query.receive_messages() → 출력 스트리밍

↓

양방향 통신:
  SDK → CLI: initialize, interrupt, set_model, 입력 메시지
  CLI → SDK: can_use_tool, hook_callback, mcp_message
  SDK ← CLI: user/assistant/system/result 메시지
```

---

## 💡 핵심 설계 결정

### 1. **제어 프로토콜의 분리**

**문제**: CLI가 SDK에 역으로 요청해야 하는 상황 (훅, 권한 확인, MCP 호출)

**해결책**:
- 일반 메시지 (type: user/assistant/system/result) → 사용자에게 전달
- 제어 메시지 (type: control_request/control_response) → Query가 내부 처리
- 메시지 스트림 분리 (`_message_send`/`_message_receive`)

### 2. **비동기 요청-응답 패턴**

```python
# 요청 전송
request_id = f"req_{counter}_{random_hex}"
self.pending_control_responses[request_id] = anyio.Event()
await self.transport.write(json.dumps(control_request))

# 응답 대기 (다른 코루틴)
async def _read_messages(self):
    if msg_type == "control_response":
        request_id = response["request_id"]
        self.pending_control_results[request_id] = response
        self.pending_control_responses[request_id].set()  # 이벤트 발화

# 원래 코루틴 재개
await event.wait()
result = self.pending_control_results.pop(request_id)
```

### 3. **SDK MCP 서버의 In-Process 실행**

**외부 MCP 서버**:
```
Python SDK → Transport → CLI → stdio → External MCP Server (subprocess)
```

**SDK MCP 서버**:
```
Python SDK → Query._handle_sdk_mcp_request → MCP Server (in-process)
```

**장점**:
- IPC 오버헤드 없음
- 애플리케이션 상태에 직접 접근
- 단일 프로세스 배포

**제약**:
- Python MCP SDK의 한계로 수동 라우팅 필요
- TypeScript는 `Transport` 추상화로 자동 처리

### 4. **훅 시스템의 양방향 통신**

**등록 (SDK → CLI)**:
```python
hooks_config = {
    "PreToolUse": [{
        "matcher": "Bash",
        "hookCallbackIds": ["hook_0", "hook_1"]
    }]
}
await query._send_control_request({"subtype": "initialize", "hooks": hooks_config})
```

**실행 (CLI → SDK)**:
```python
# CLI가 훅 트리거 시
control_request = {
    "type": "control_request",
    "request": {
        "subtype": "hook_callback",
        "callback_id": "hook_0",
        "input": {...}
    }
}
# SDK가 콜백 실행 후 응답
```

### 5. **타입 안전성 vs 런타임 유연성**

**타입 정의**:
```python
Message = UserMessage | AssistantMessage | SystemMessage | ResultMessage | StreamEvent
```

**파싱**:
```python
def parse_message(data: dict) -> Message:
    # 런타임에 dict → TypedDict 변환
    # mypy는 리턴 타입이 Message임을 보장
```

**장점**:
- 개발 시 IDE 자동완성
- 타입 체커 검증
- 런타임 오버헤드 없음 (TypedDict는 dict)

---

## 🎯 주요 발견

### 1. **계층적 책임 분리**

| 레이어 | 책임 |
|--------|------|
| `InternalClient` | 옵션 검증, 리소스 조립, 생명주기 관리 |
| `Query` | 제어 프로토콜, 훅/권한 라우팅, MCP 브리징 |
| `Transport` | 프로세스 관리, 저수준 I/O |
| `MessageParser` | 타입 변환, 에러 처리 |

### 2. **Python vs TypeScript 차이 대응**

**TypeScript 장점 활용 불가**:
- MCP Transport 추상화 → 수동 라우팅
- Decorator 기반 서버 → 함수 기반으로 적응

**Python 장점 활용**:
- `anyio` → asyncio + trio 동시 지원
- Pattern matching → 가독성 높은 메시지 파싱
- Type hints → mypy strict 모드

### 3. **확장 포인트**

- **커스텀 Transport**: `Transport` 인터페이스 구현
- **커스텀 에러 핸들링**: `ClaudeSDKError` 서브클래싱
- **메시지 변환**: `parse_message` 함수 래핑

### 4. **성능 최적화**

- 메시지 버퍼링: `max_buffer_size=100`
- 백그라운드 스트리밍: `_tg.start_soon`
- 타임아웃 설정: `anyio.fail_after(60.0)`

---

## 📊 복잡도 분석

| 파일 | LOC | 주요 복잡도 원인 |
|------|-----|------------------|
| `_internal/query.py` | ~530 | 제어 프로토콜 라우팅, MCP 브리징 |
| `_internal/client.py` | ~120 | 옵션 검증 및 변환 |
| `_internal/message_parser.py` | ~170 | 타입별 파싱 로직 |
| `_errors.py` | ~60 | 에러 계층 및 컨텍스트 |

**총 복잡도**: 약 880 LOC (Public API는 ~300 LOC)

**비율**: Internal 구현이 Public API의 약 3배 복잡

---

## 🔮 개선 가능 영역

### 1. **Python MCP SDK 개선 시**

현재:
```python
# 수동 라우팅
if method == "tools/list": ...
elif method == "tools/call": ...
```

이상적:
```python
# Transport 추상화 사용 (TypeScript처럼)
transport = InMemoryTransport(query._read, query._write)
await server.connect(transport)
```

### 2. **로깅 추가**

현재는 `logger.debug`만 일부 사용. 다음 추가 가능:
- 제어 요청/응답 로깅
- MCP 호출 추적
- 훅 실행 시간 측정

### 3. **Graceful Shutdown**

현재는 `_tg.cancel_scope.cancel()` → 즉시 취소

개선안:
- 진행 중인 제어 요청 대기
- 부분 응답 저장
- 재연결 지원

---

**작성**: Claude Code
**분석 범위**: `_internal/` 디렉토리 전체
**참조 파일**: client.py (122 LOC), query.py (529 LOC), message_parser.py (173 LOC), _errors.py (57 LOC)
