# SDK MCP 서버 & 훅 시스템 심층 분석

## 📋 개요

이 문서는 claude-agent-sdk-python의 두 가지 핵심 확장 메커니즘을 분석합니다:
1. **SDK MCP 서버**: In-process 도구 제공
2. **훅 시스템**: 에이전트 루프 제어

---

## 🔧 SDK MCP 서버

### 개념

**MCP (Model Context Protocol)**: Anthropic이 정의한 도구 제공 표준 프로토콜

**두 가지 MCP 서버 타입:**

| 타입 | 실행 위치 | 통신 방식 | 사용 사례 |
|------|----------|-----------|----------|
| **External MCP** | 별도 프로세스 | stdio (IPC) | 범용 도구, 시스템 통합 |
| **SDK MCP** | Python 프로세스 내부 | 함수 호출 | 애플리케이션 상태 접근, 커스텀 로직 |

### 아키텍처

```
┌─────────────────────────────────────────────────────┐
│ Python Application                                  │
│                                                     │
│  ┌──────────────────────────────────────┐          │
│  │ @tool("greet", ...)                  │          │
│  │ async def greet(args):               │          │
│  │     return {"content": [...]}        │          │
│  └──────────────────────────────────────┘          │
│              ↓                                      │
│  ┌──────────────────────────────────────┐          │
│  │ create_sdk_mcp_server(               │          │
│  │   name="my_server",                  │          │
│  │   tools=[greet]                      │          │
│  │ )                                    │          │
│  └──────────────────────────────────────┘          │
│              ↓ Returns McpSdkServerConfig          │
│  ┌──────────────────────────────────────┐          │
│  │ ClaudeAgentOptions(                  │          │
│  │   mcp_servers={                      │          │
│  │     "my_server": server_config       │          │
│  │   }                                  │          │
│  │ )                                    │          │
│  └──────────────────────────────────────┘          │
│              ↓                                      │
│  ┌──────────────────────────────────────┐          │
│  │ InternalClient.process_query()       │          │
│  │   - Extract sdk_mcp_servers          │          │
│  │   - Pass to Query                    │          │
│  └──────────────────────────────────────┘          │
│              ↓                                      │
│  ┌──────────────────────────────────────┐          │
│  │ Query._handle_sdk_mcp_request()      │          │
│  │   - Route JSONRPC messages           │          │
│  │   - Call MCP Server handlers         │          │
│  │   - Return results                   │          │
│  └──────────────────────────────────────┘          │
│              ↑ JSONRPC in/out                      │
└──────────────┼─────────────────────────────────────┘
               │
         ┌─────┴──────┐
         │   Claude   │
         │  Code CLI  │
         └────────────┘
```

### 구현 세부사항

#### 1. `@tool` 데코레이터

```python
@tool(
    name="add",
    description="Add two numbers",
    input_schema={"a": float, "b": float}
)
async def add_numbers(args):
    result = args["a"] + args["b"]
    return {
        "content": [{"type": "text", "text": f"Result: {result}"}]
    }
```

**내부 동작:**

```python
def tool(name, description, input_schema):
    def decorator(handler):
        return SdkMcpTool(
            name=name,
            description=description,
            input_schema=input_schema,
            handler=handler
        )
    return decorator
```

**반환 타입**: `SdkMcpTool[T]` (Generic 타입으로 입력 스키마 타입 추적)

#### 2. `create_sdk_mcp_server()`

**함수 시그니처:**

```python
def create_sdk_mcp_server(
    name: str,
    version: str = "1.0.0",
    tools: list[SdkMcpTool] | None = None
) -> McpSdkServerConfig
```

**구현 단계:**

##### A. MCP Server 인스턴스 생성

```python
from mcp.server import Server

server = Server(name, version=version)
```

##### B. 도구 맵 구축

```python
tool_map = {tool_def.name: tool_def for tool_def in tools}
```

##### C. `list_tools` 핸들러 등록

```python
@server.list_tools()
async def list_tools() -> list[Tool]:
    tool_list = []
    for tool_def in tools:
        # input_schema를 JSON Schema로 변환
        if isinstance(tool_def.input_schema, dict):
            if "type" in tool_def.input_schema:
                # 이미 JSON Schema 형식
                schema = tool_def.input_schema
            else:
                # {name: type} 형식 → JSON Schema 변환
                properties = {}
                for param_name, param_type in tool_def.input_schema.items():
                    if param_type is str: properties[param_name] = {"type": "string"}
                    elif param_type is int: properties[param_name] = {"type": "integer"}
                    elif param_type is float: properties[param_name] = {"type": "number"}
                    elif param_type is bool: properties[param_name] = {"type": "boolean"}

                schema = {
                    "type": "object",
                    "properties": properties,
                    "required": list(properties.keys())
                }
        else:
            # TypedDict 등 → 기본 스키마
            schema = {"type": "object", "properties": {}}

        tool_list.append(Tool(
            name=tool_def.name,
            description=tool_def.description,
            inputSchema=schema
        ))
    return tool_list
```

**주요 로직**: Python 타입 힌트 → JSON Schema 자동 변환

##### D. `call_tool` 핸들러 등록

```python
@server.call_tool()
async def call_tool(name: str, arguments: dict[str, Any]) -> Any:
    if name not in tool_map:
        raise ValueError(f"Tool '{name}' not found")

    tool_def = tool_map[name]
    result = await tool_def.handler(arguments)

    # 사용자 함수 반환값 → MCP 형식 변환
    content = []
    if "content" in result:
        for item in result["content"]:
            if item["type"] == "text":
                content.append(TextContent(type="text", text=item["text"]))

    return content  # MCP SDK가 CallToolResult로 래핑
```

##### E. 서버 구성 반환

```python
return McpSdkServerConfig(
    type="sdk",
    name=name,
    instance=server  # 실제 Server 인스턴스
)
```

#### 3. Query의 MCP 브리지

**호출 흐름:**

```
Claude Code CLI
  ↓ (JSONRPC over control_request)
Query._handle_control_request()
  ↓ (subtype == "mcp_message")
Query._handle_sdk_mcp_request(server_name, message)
  ↓
MCP Server.request_handlers[method]()
  ↓
사용자 정의 handler 함수
  ↓
결과를 JSONRPC 형식으로 변환
  ↓ (control_response)
Claude Code CLI
```

**라우팅 코드:**

```python
async def _handle_sdk_mcp_request(self, server_name: str, message: dict):
    server = self.sdk_mcp_servers[server_name]
    method = message["method"]
    params = message.get("params", {})

    if method == "initialize":
        # 하드코딩된 초기화 응답
        return {
            "jsonrpc": "2.0",
            "id": message["id"],
            "result": {
                "protocolVersion": "2024-11-05",
                "capabilities": {"tools": {}},
                "serverInfo": {"name": server.name, "version": server.version}
            }
        }

    elif method == "tools/list":
        # MCP SDK 핸들러 호출
        request = ListToolsRequest(method=method)
        handler = server.request_handlers[ListToolsRequest]
        result = await handler(request)

        # 결과 변환
        tools_data = [
            {
                "name": tool.name,
                "description": tool.description,
                "inputSchema": tool.inputSchema.model_dump() if hasattr(...) else tool.inputSchema
            }
            for tool in result.root.tools
        ]
        return {"jsonrpc": "2.0", "id": message["id"], "result": {"tools": tools_data}}

    elif method == "tools/call":
        # 도구 실행
        request = CallToolRequest(
            method=method,
            params=CallToolRequestParams(
                name=params["name"],
                arguments=params.get("arguments", {})
            )
        )
        handler = server.request_handlers[CallToolRequest]
        result = await handler(request)

        # 결과 변환 (TextContent | ImageContent)
        content = []
        for item in result.root.content:
            if hasattr(item, "text"):
                content.append({"type": "text", "text": item.text})
            elif hasattr(item, "data") and hasattr(item, "mimeType"):
                content.append({"type": "image", "data": item.data, "mimeType": item.mimeType})

        return {"jsonrpc": "2.0", "id": message["id"], "result": {"content": content}}

    elif method == "notifications/initialized":
        # 초기화 완료 알림 - 단순 ACK
        return {"jsonrpc": "2.0", "result": {}}

    else:
        return {"jsonrpc": "2.0", "error": {"code": -32601, "message": f"Method '{method}' not found"}}
```

**주요 문제점**: Python MCP SDK에 `Transport` 추상화 없음 → 수동 라우팅 필수

### 사용 예제

#### 기본 사용

```python
from claude_agent_sdk import query, tool, create_sdk_mcp_server, ClaudeAgentOptions

@tool("greet", "Greet a user", {"name": str})
async def greet(args):
    return {"content": [{"type": "text", "text": f"Hello, {args['name']}!"}]}

server = create_sdk_mcp_server("greeter", tools=[greet])

options = ClaudeAgentOptions(
    mcp_servers={"greeter": server},
    allowed_tools=["greet"]
)

async for message in query("Greet Alice", options=options):
    print(message)
```

#### 애플리케이션 상태 접근

```python
class DataStore:
    def __init__(self):
        self.items = []

store = DataStore()

@tool("add_item", "Add item", {"item": str})
async def add_item(args):
    store.items.append(args["item"])  # 직접 접근!
    return {"content": [{"type": "text", "text": f"Added: {args['item']}"}]}

@tool("list_items", "List all items", {})
async def list_items(args):
    items_str = ", ".join(store.items)
    return {"content": [{"type": "text", "text": f"Items: {items_str}"}]}

server = create_sdk_mcp_server("store", tools=[add_item, list_items])
```

#### 에러 처리

```python
@tool("divide", "Divide numbers", {"a": float, "b": float})
async def divide(args):
    if args["b"] == 0:
        return {
            "content": [{"type": "text", "text": "Error: Division by zero"}],
            "is_error": True
        }
    result = args["a"] / args["b"]
    return {"content": [{"type": "text", "text": f"Result: {result}"}]}
```

### External vs SDK MCP 비교

```python
options = ClaudeAgentOptions(
    mcp_servers={
        # External MCP Server (별도 프로세스)
        "filesystem": {
            "type": "stdio",
            "command": "npx",
            "args": ["-y", "@modelcontextprotocol/server-filesystem", "/tmp"]
        },

        # SDK MCP Server (in-process)
        "custom": create_sdk_mcp_server("custom", tools=[...])
    }
)
```

**성능 비교:**

| 지표 | External MCP | SDK MCP |
|------|--------------|---------|
| 프로세스 생성 | 필요 (수백 ms) | 불필요 |
| IPC 오버헤드 | stdio (수 ms/호출) | 함수 호출 (μs) |
| 상태 공유 | 불가 (별도 메모리) | 가능 (같은 프로세스) |
| 배포 | 별도 실행 파일 필요 | 단일 Python 앱 |

---

## 🎣 훅(Hook) 시스템

### 개념

**훅**: Claude 에이전트 루프의 특정 시점에 실행되는 콜백 함수

**지원하는 이벤트:**

| 이벤트 | 발생 시점 | 용도 |
|--------|----------|------|
| `SessionStart` | 세션 시작 시 | 초기 컨텍스트 추가 |
| `UserPromptSubmit` | 사용자 입력 직전 | 프롬프트 수정, 컨텍스트 주입 |
| `PreToolUse` | 도구 실행 직전 | 도구 권한 검증, 입력 수정 |
| `PostToolUse` | 도구 실행 직후 | 결과 검증, 후처리 |
| `PrePromptCaching` | 프롬프트 캐싱 직전 | 캐시 정책 제어 |

### 아키텍처

```
┌─────────────────────────────────────────────────────┐
│ Python Application                                  │
│                                                     │
│  async def check_bash(input, tool_use_id, context): │
│      if "rm -rf" in input["tool_input"]["command"]:│
│          return {"permissionDecision": "deny"}      │
│      return {}                                      │
│                                                     │
│  options = ClaudeAgentOptions(                     │
│      hooks={                                        │
│          "PreToolUse": [                           │
│              HookMatcher(                           │
│                  matcher="Bash",                    │
│                  hooks=[check_bash]                 │
│              )                                      │
│          ]                                          │
│      }                                              │
│  )                                                 │
└─────────────────────────────────────────────────────┘
               ↓ (hooks 전달)
┌─────────────────────────────────────────────────────┐
│ InternalClient.process_query()                      │
│   - _convert_hooks_to_internal_format()            │
│   - Query에 전달                                    │
└─────────────────────────────────────────────────────┘
               ↓
┌─────────────────────────────────────────────────────┐
│ Query.initialize()                                  │
│   1. 각 콜백에 ID 할당 (hook_0, hook_1, ...)       │
│   2. self.hook_callbacks[id] = callback            │
│   3. CLI에 hooks_config 전송                       │
│      {                                              │
│        "PreToolUse": [{                            │
│          "matcher": "Bash",                         │
│          "hookCallbackIds": ["hook_0"]             │
│        }]                                           │
│      }                                              │
└─────────────────────────────────────────────────────┘
               ↓ (JSONRPC initialize)
┌─────────────────────────────────────────────────────┐
│ Claude Code CLI                                     │
│   - 훅 구성 저장                                    │
│   - 에이전트 루프 시작                              │
└─────────────────────────────────────────────────────┘
               ↓ (PreToolUse 이벤트 발생)
┌─────────────────────────────────────────────────────┐
│ Claude Code CLI                                     │
│   - 매칭 훅 찾기 (matcher == "Bash")                │
│   - SDK에 hook_callback 요청 전송                  │
│      {                                              │
│        "subtype": "hook_callback",                  │
│        "callback_id": "hook_0",                     │
│        "input": {"tool_name": "Bash", ...}          │
│      }                                              │
└─────────────────────────────────────────────────────┘
               ↓ (control_request)
┌─────────────────────────────────────────────────────┐
│ Query._handle_control_request()                    │
│   1. callback_id로 함수 찾기                        │
│   2. callback = self.hook_callbacks["hook_0"]      │
│   3. result = await callback(input, tool_use_id,   │
│                               context)              │
│   4. CLI에 result 반환                             │
└─────────────────────────────────────────────────────┘
               ↓ (control_response)
┌─────────────────────────────────────────────────────┐
│ Claude Code CLI                                     │
│   - 훅 결과 처리                                    │
│   - permissionDecision == "deny" → 도구 차단       │
│   - 에이전트 루프 계속                              │
└─────────────────────────────────────────────────────┘
```

### 구현 세부사항

#### 1. 훅 정의

```python
from claude_agent_sdk.types import HookMatcher, HookContext, HookJSONOutput

async def my_hook(
    input_data: dict[str, Any],
    tool_use_id: str | None,
    context: HookContext
) -> HookJSONOutput:
    # input_data: 이벤트별 데이터 (tool_name, tool_input 등)
    # tool_use_id: 도구 호출 ID (PreToolUse/PostToolUse만)
    # context: signal (취소 토큰 - 현재 미구현)

    return {
        "hookSpecificOutput": {
            "hookEventName": "PreToolUse",
            "permissionDecision": "deny",  # "allow" | "deny"
            "permissionDecisionReason": "Blocked for security"
        }
    }
```

**반환 타입**: `HookJSONOutput` (TypedDict)

```python
class HookJSONOutput(TypedDict, total=False):
    hookSpecificOutput: dict[str, Any]  # 이벤트별 출력
    # 빈 dict {} 반환 시 → 기본 동작
```

#### 2. 훅 등록

```python
options = ClaudeAgentOptions(
    hooks={
        "PreToolUse": [
            HookMatcher(
                matcher="Bash",  # 도구 이름 또는 None (모든 도구)
                hooks=[check_bash, log_tool_use]  # 여러 콜백 가능
            ),
            HookMatcher(
                matcher="Read",
                hooks=[check_file_access]
            )
        ],
        "PostToolUse": [
            HookMatcher(
                matcher=None,  # 모든 도구
                hooks=[log_results]
            )
        ]
    }
)
```

#### 3. 내부 변환

```python
def _convert_hooks_to_internal_format(self, hooks):
    internal_hooks = {}
    for event, matchers in hooks.items():
        internal_hooks[event] = []
        for matcher in matchers:
            internal_hooks[event].append({
                "matcher": matcher.matcher,  # "Bash" | None
                "hooks": matcher.hooks        # [func1, func2]
            })
    return internal_hooks
```

#### 4. 초기화 시 등록

```python
async def initialize(self):
    hooks_config = {}
    for event, matchers in self.hooks.items():
        hooks_config[event] = []
        for matcher in matchers:
            callback_ids = []
            for callback in matcher["hooks"]:
                callback_id = f"hook_{self.next_callback_id}"
                self.next_callback_id += 1
                self.hook_callbacks[callback_id] = callback  # 저장
                callback_ids.append(callback_id)

            hooks_config[event].append({
                "matcher": matcher["matcher"],
                "hookCallbackIds": callback_ids
            })

    response = await self._send_control_request({
        "subtype": "initialize",
        "hooks": hooks_config
    })
    return response
```

#### 5. 실행 (CLI → SDK)

```python
async def _handle_control_request(self, request):
    if subtype == "hook_callback":
        callback_id = request_data["callback_id"]
        callback = self.hook_callbacks.get(callback_id)

        if not callback:
            raise Exception(f"No hook callback found for ID: {callback_id}")

        response_data = await callback(
            request_data["input"],
            request_data.get("tool_use_id"),
            {"signal": None}  # TODO: AbortSignal 지원
        )

        # 성공 응답 전송
        await self.transport.write(json.dumps({
            "type": "control_response",
            "response": {
                "subtype": "success",
                "request_id": request_id,
                "response": response_data
            }
        }))
```

### 사용 예제

#### PreToolUse: 도구 차단

```python
async def block_dangerous_commands(input_data, tool_use_id, context):
    if input_data["tool_name"] != "Bash":
        return {}

    command = input_data["tool_input"].get("command", "")
    dangerous_patterns = ["rm -rf /", "mkfs", "dd if=/dev/zero"]

    for pattern in dangerous_patterns:
        if pattern in command:
            return {
                "hookSpecificOutput": {
                    "hookEventName": "PreToolUse",
                    "permissionDecision": "deny",
                    "permissionDecisionReason": f"Dangerous command: {pattern}"
                }
            }

    return {}  # 허용
```

#### UserPromptSubmit: 컨텍스트 추가

```python
async def add_user_context(input_data, tool_use_id, context):
    user_preferences = load_user_preferences()

    return {
        "hookSpecificOutput": {
            "hookEventName": "UserPromptSubmit",
            "additionalContext": f"User preferences: {user_preferences}"
        }
    }
```

#### PostToolUse: 결과 검증

```python
async def validate_tool_result(input_data, tool_use_id, context):
    tool_name = input_data["tool_name"]
    result = input_data["tool_result"]

    if tool_name == "Read" and "password" in result["content"]:
        return {
            "hookSpecificOutput": {
                "hookEventName": "PostToolUse",
                "overrideResult": {
                    "content": "[REDACTED: Password detected]",
                    "is_error": False
                }
            }
        }

    return {}
```

#### SessionStart: 초기 설정

```python
async def session_start(input_data, tool_use_id, context):
    return {
        "hookSpecificOutput": {
            "hookEventName": "SessionStart",
            "additionalContext": "You are an expert Python developer. Always prefer type hints."
        }
    }

options = ClaudeAgentOptions(
    hooks={"SessionStart": [HookMatcher(matcher=None, hooks=[session_start])]}
)
```

### 훅 체인

**여러 훅이 등록된 경우:**

```python
async def log_hook(input_data, tool_use_id, context):
    logger.info(f"Tool called: {input_data['tool_name']}")
    return {}  # 다음 훅으로

async def validate_hook(input_data, tool_use_id, context):
    if invalid:
        return {"hookSpecificOutput": {"permissionDecision": "deny"}}
    return {}

options = ClaudeAgentOptions(
    hooks={
        "PreToolUse": [
            HookMatcher(matcher="Bash", hooks=[log_hook, validate_hook])
        ]
    }
)
```

**실행 순서**: `log_hook` → `validate_hook`

**조기 종료**: 첫 번째 훅이 `"deny"` 반환 시 나머지 실행 안 함

---

## 🎯 핵심 설계 패턴

### 1. **Decorator + Factory 패턴**

```python
# Decorator
@tool("name", "desc", schema)
def my_tool(args): ...

# Factory
create_sdk_mcp_server(tools=[my_tool])
```

### 2. **Callback Registry**

```python
# 등록
self.hook_callbacks[callback_id] = callback

# 실행
callback = self.hook_callbacks[callback_id]
await callback(...)
```

### 3. **Type-Safe Configuration**

```python
# 타입 힌트로 구성 검증
options = ClaudeAgentOptions(
    hooks={
        "PreToolUse": [HookMatcher(matcher="Bash", hooks=[...])]
    }
)
```

### 4. **Protocol Bridge**

```python
# Python 객체 ↔ JSONRPC 메시지 변환
def python_to_jsonrpc(result): ...
def jsonrpc_to_python(message): ...
```

---

## 💡 고급 활용

### 1. 상태 기반 훅

```python
class ToolUsageTracker:
    def __init__(self):
        self.usage_count = {}

    async def rate_limit_hook(self, input_data, tool_use_id, context):
        tool_name = input_data["tool_name"]
        self.usage_count[tool_name] = self.usage_count.get(tool_name, 0) + 1

        if self.usage_count[tool_name] > 10:
            return {
                "hookSpecificOutput": {
                    "permissionDecision": "deny",
                    "permissionDecisionReason": "Rate limit exceeded"
                }
            }
        return {}

tracker = ToolUsageTracker()
options = ClaudeAgentOptions(
    hooks={"PreToolUse": [HookMatcher(matcher=None, hooks=[tracker.rate_limit_hook])]}
)
```

### 2. 도구 입력 수정

```python
async def sanitize_input(input_data, tool_use_id, context):
    if input_data["tool_name"] == "Bash":
        command = input_data["tool_input"]["command"]
        # 안전한 명령으로 수정
        safe_command = command.replace("rm", "ls")

        return {
            "hookSpecificOutput": {
                "updatedInput": {"command": safe_command}
            }
        }
    return {}
```

### 3. 조건부 권한 부여

```python
async def conditional_permission(input_data, tool_use_id, context):
    tool_name = input_data["tool_name"]
    suggestions = context.get("suggestions", [])

    if "allow_read_only" in suggestions and tool_name == "Write":
        return {"hookSpecificOutput": {"permissionDecision": "deny"}}

    return {}
```

---

## 📊 성능 고려사항

### SDK MCP 서버

| 요소 | 비용 | 최적화 |
|------|------|--------|
| 도구 등록 | O(1) | 초기화 시 한 번 |
| list_tools | O(n) tools | 캐싱 가능 |
| call_tool | O(1) lookup + handler 시간 | 비동기 실행 |

### 훅 시스템

| 요소 | 비용 | 최적화 |
|------|------|--------|
| 훅 등록 | O(n) hooks | 초기화 시 한 번 |
| 매칭 | O(1) (CLI가 처리) | N/A |
| 실행 | O(1) lookup + handler 시간 | 비동기 체인 |
| 왕복 지연 | ~1-10ms (in-process RPC) | 배치 처리 불가 (순차) |

---

## 🔮 개선 가능 영역

### 1. SDK MCP 서버

**현재 제약:**
- Python MCP SDK의 Transport 부재 → 수동 라우팅
- 지원 메서드 제한 (tools만, resources/prompts 없음)

**개선안:**
```python
# TypeScript처럼
transport = InMemoryTransport(...)
await server.connect(transport)
# 모든 MCP 기능 자동 지원
```

### 2. 훅 시스템

**현재 제약:**
- AbortSignal 미지원 (context.signal = None)
- 동기식 실행 (병렬 훅 불가)

**개선안:**
```python
# 병렬 훅 실행
results = await asyncio.gather(*[hook(...) for hook in hooks])

# 취소 지원
async with anyio.CancelScope() as scope:
    context.signal = scope
    await hook(...)
```

### 3. 디버깅

**추가하면 좋을 기능:**
- 훅 실행 추적 (시간, 입력/출력)
- MCP 호출 로깅
- 에러 스택 트레이스 개선

---

**작성**: Claude Code
**분석 범위**: SDK MCP (__init__.py), 훅 시스템 (_internal/query.py), 예제 (hooks.py)
**참조 파일**: __init__.py (124-278 LOC), query.py (187-289 LOC), hooks.py (185 LOC)
