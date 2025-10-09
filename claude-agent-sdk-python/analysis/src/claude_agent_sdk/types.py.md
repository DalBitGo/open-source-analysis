# types.py

**원본 경로**: `source/src/claude_agent_sdk/types.py`
**역할**: SDK의 모든 타입 정의 및 데이터 구조 제공
**라인 수**: 450줄
**의존성**: `typing_extensions`, `mcp.server` (TYPE_CHECKING)

---

## 📊 구조 개요

- **클래스**: 13개 (Dataclass 10개, TypedDict 9개)
- **타입 별칭**: 7개 (Literal, Union)
- **함수/메서드**: 1개 (`PermissionUpdate.to_dict()`)
- **의존성**: 최소 (기반 타입 모듈만)

---

## 🔍 상세 분석

### 1. Permission System Types

#### 1.1 `PermissionMode`
**라인**: 15

```python
PermissionMode = Literal["default", "acceptEdits", "plan", "bypassPermissions"]
```

**역할**: 도구 실행 권한 모드 정의

**값**:
- `default`: CLI가 위험한 도구에 대해 프롬프트 표시
- `acceptEdits`: 파일 편집 자동 승인
- `plan`: 계획 모드 (실행 전 검토)
- `bypassPermissions`: 모든 도구 허용 (주의 필요)

**설계 평가**:
- ✅ **Simplicity**: Literal 타입으로 컴파일 타임 검증
- ✅ **Evolvability**: 새 모드 추가 시 타입 체크로 호환성 보장
- ✅ **Operability**: 명확한 의미의 값

---

#### 1.2 `PermissionUpdate`
**라인**: 56-108

```python
@dataclass
class PermissionUpdate:
    type: Literal["addRules", "replaceRules", "removeRules",
                  "setMode", "addDirectories", "removeDirectories"]
    rules: list[PermissionRuleValue] | None = None
    behavior: PermissionBehavior | None = None
    mode: PermissionMode | None = None
    directories: list[str] | None = None
    destination: PermissionUpdateDestination | None = None

    def to_dict(self) -> dict[str, Any]:
        """Convert to TypeScript control protocol format."""
        # ...
```

**책임**:
- 권한 업데이트 명령 표현
- Python ↔ TypeScript 형식 변환

**`to_dict()` 메서드**:
```python
def to_dict(self) -> dict[str, Any]:
    result: dict[str, Any] = {"type": self.type}

    if self.type in ["addRules", "replaceRules", "removeRules"]:
        if self.rules is not None:
            result["rules"] = [
                {"toolName": rule.tool_name, "ruleContent": rule.rule_content}
                for rule in self.rules
            ]
        if self.behavior is not None:
            result["behavior"] = self.behavior

    elif self.type == "setMode":
        if self.mode is not None:
            result["mode"] = self.mode

    # ...
    return result
```

**설계 패턴**:
- **Type-based Dispatch**: `type` 필드로 다른 필드 결정
- **Adapter**: Python dataclass ↔ TypeScript JSON

**설계 평가**:
- ✅ **Operability**: 명시적 변환 로직
- ⚠️ **Simplicity**: 6가지 타입 변형, 복잡도 높음
- ✅ **Evolvability**: 새로운 `type` 추가 용이

---

#### 1.3 `PermissionResult`
**라인**: 122-140

```python
@dataclass
class PermissionResultAllow:
    behavior: Literal["allow"] = "allow"
    updated_input: dict[str, Any] | None = None
    updated_permissions: list[PermissionUpdate] | None = None

@dataclass
class PermissionResultDeny:
    behavior: Literal["deny"] = "deny"
    message: str = ""
    interrupt: bool = False

PermissionResult = PermissionResultAllow | PermissionResultDeny
```

**설계 패턴**:
- **Tagged Union**: `behavior` 필드로 타입 구분
- **Result Type**: 성공/실패를 타입으로 표현

**사용 예시**:
```python
async def can_use_tool(...) -> PermissionResult:
    if is_safe:
        return PermissionResultAllow(
            updated_permissions=[PermissionUpdate(...)]
        )
    else:
        return PermissionResultDeny(
            message="Tool not allowed",
            interrupt=True
        )
```

**설계 평가**:
- ✅✅ **Simplicity**: 명확한 의미, 타입 안전
- ✅ **Operability**: `interrupt` 플래그로 세밀한 제어
- ✅ **Evolvability**: Union 타입으로 확장 가능

---

### 2. Hook System Types

#### 2.1 `HookEvent`
**라인**: 150-157

```python
HookEvent = (
    Literal["PreToolUse"]
    | Literal["PostToolUse"]
    | Literal["UserPromptSubmit"]
    | Literal["Stop"]
    | Literal["SubagentStop"]
    | Literal["PreCompact"]
)
```

**역할**: 지원되는 Hook 이벤트 타입

**주석**:
```python
# Python SDK는 SessionStart, SessionEnd, Notification 미지원
```

**설계 평가**:
- ✅ **Simplicity**: 명확한 열거형
- ⚠️ **Operability**: 미지원 이유가 코드에만 존재 (문서화 필요)
- ✅ **Evolvability**: 새 이벤트 추가 쉬움

---

#### 2.2 `HookJSONOutput`
**라인**: 163-172

```python
class HookJSONOutput(TypedDict):
    decision: NotRequired[Literal["block"]]
    systemMessage: NotRequired[str]
    hookSpecificOutput: NotRequired[Any]
```

**특징**:
- **TypedDict**: 런타임 검증 없이 타입 힌트만 제공
- **NotRequired**: 모든 필드 선택적 (Python 3.11+)

**설계 평가**:
- ✅ **Simplicity**: 최소 구조
- ⚠️ **Operability**: `hookSpecificOutput: Any`는 타입 안전성 없음
- ✅ **Evolvability**: 새 필드 추가 용이

---

#### 2.3 `HookMatcher`
**라인**: 193-206

```python
@dataclass
class HookMatcher:
    matcher: str | None = None
    hooks: list[HookCallback] = field(default_factory=list)
```

**역할**: Hook 이벤트와 콜백 매칭

**예시**:
```python
HookMatcher(
    matcher="Bash",  # 도구명 패턴
    hooks=[check_bash_command]  # 콜백 함수들
)

HookMatcher(
    matcher="Write|Edit",  # 여러 도구 매칭
    hooks=[validate_file_operation]
)
```

**설계 평가**:
- ✅ **Simplicity**: 간단한 구조
- ✅ **Evolvability**: 정규식 패턴 지원 가능

---

### 3. MCP Server Types

#### 3.1 `McpServerConfig` Union
**라인**: 209-244

```python
class McpStdioServerConfig(TypedDict):
    type: NotRequired[Literal["stdio"]]
    command: str
    args: NotRequired[list[str]]
    env: NotRequired[dict[str, str]]

class McpSSEServerConfig(TypedDict):
    type: Literal["sse"]
    url: str
    headers: NotRequired[dict[str, str]]

class McpHttpServerConfig(TypedDict):
    type: Literal["http"]
    url: str
    headers: NotRequired[dict[str, str]]

class McpSdkServerConfig(TypedDict):
    type: Literal["sdk"]
    name: str
    instance: "McpServer"

McpServerConfig = (
    McpStdioServerConfig
    | McpSSEServerConfig
    | McpHttpServerConfig
    | McpSdkServerConfig
)
```

**설계 패턴**:
- **Tagged Union**: `type` 필드로 구분
- **Discriminated Union**: TypeScript 스타일

**사용 예시**:
```python
# External MCP (Subprocess)
stdio_server: McpServerConfig = {
    "type": "stdio",
    "command": "python",
    "args": ["-m", "mcp_server"],
    "env": {"API_KEY": "..."}
}

# SDK MCP (In-process)
sdk_server: McpServerConfig = {
    "type": "sdk",
    "name": "my-tools",
    "instance": server_instance
}
```

**설계 평가**:
- ✅✅ **Evolvability**: 4가지 전송 방식 지원, 확장 가능
- ✅ **Simplicity**: 타입으로 구분 명확
- ✅ **Operability**: 각 타입별 필수 필드 명시

---

### 4. Content Block Types

#### 4.1 Block Types
**라인**: 247-281

```python
@dataclass
class TextBlock:
    text: str

@dataclass
class ThinkingBlock:
    thinking: str
    signature: str

@dataclass
class ToolUseBlock:
    id: str
    name: str
    input: dict[str, Any]

@dataclass
class ToolResultBlock:
    tool_use_id: str
    content: str | list[dict[str, Any]] | None = None
    is_error: bool | None = None

ContentBlock = TextBlock | ThinkingBlock | ToolUseBlock | ToolResultBlock
```

**설계 특징**:
- **Minimal Dataclass**: 필수 필드만
- **Union Type**: 다형성

**블록 타입별 역할**:

| 타입 | 역할 | 예시 |
|------|------|------|
| `TextBlock` | 텍스트 응답 | "The answer is 42" |
| `ThinkingBlock` | 사고 과정 (Extended Thinking) | 내부 추론 |
| `ToolUseBlock` | 도구 호출 요청 | `{"id": "...", "name": "Bash", "input": {...}}` |
| `ToolResultBlock` | 도구 실행 결과 | `{"tool_use_id": "...", "content": "..."}` |

**설계 평가**:
- ✅ **Simplicity**: 각 블록이 단순하고 명확
- ✅ **Evolvability**: 새 블록 타입 추가 용이
- ✅ **Operability**: `is_error` 플래그로 에러 처리

---

### 5. Message Types

#### 5.1 Message Dataclasses
**라인**: 284-335

```python
@dataclass
class UserMessage:
    content: str | list[ContentBlock]
    parent_tool_use_id: str | None = None

@dataclass
class AssistantMessage:
    content: list[ContentBlock]
    model: str
    parent_tool_use_id: str | None = None

@dataclass
class SystemMessage:
    subtype: str
    data: dict[str, Any]

@dataclass
class ResultMessage:
    subtype: str
    duration_ms: int
    duration_api_ms: int
    is_error: bool
    num_turns: int
    session_id: str
    total_cost_usd: float | None = None
    usage: dict[str, Any] | None = None
    result: str | None = None

@dataclass
class StreamEvent:
    uuid: str
    session_id: str
    event: dict[str, Any]  # Raw Anthropic API stream event
    parent_tool_use_id: str | None = None

Message = UserMessage | AssistantMessage | SystemMessage | ResultMessage | StreamEvent
```

**메시지 타입별 역할**:

| 타입 | 방향 | 역할 |
|------|------|------|
| `UserMessage` | User → Claude | 사용자 입력 |
| `AssistantMessage` | Claude → User | AI 응답 |
| `SystemMessage` | System → User | 시스템 메시지 (메타데이터) |
| `ResultMessage` | System → User | 세션 종료, 비용/사용량 정보 |
| `StreamEvent` | Claude → User | 실시간 스트리밍 이벤트 |

**`parent_tool_use_id` 필드**:
- 도구 호출 컨텍스트 추적
- 서브에이전트 지원

**`ResultMessage` 필드 분석**:
```python
duration_ms: int           # 총 실행 시간
duration_api_ms: int       # API 호출 시간
num_turns: int             # 대화 턴 수
total_cost_usd: float      # 총 비용 (USD)
usage: dict[str, Any]      # 토큰 사용량
```

**설계 평가**:
- ✅ **Operability**: `ResultMessage`에 메트릭 포함
- ✅ **Simplicity**: 각 메시지 타입 명확
- ✅ **Evolvability**: Union 타입으로 확장 가능

---

### 6. Options & Configuration

#### 6.1 `ClaudeAgentOptions`
**라인**: 338-382

```python
@dataclass
class ClaudeAgentOptions:
    # Tool configuration
    allowed_tools: list[str] = field(default_factory=list)
    disallowed_tools: list[str] = field(default_factory=list)

    # System prompt
    system_prompt: str | SystemPromptPreset | None = None

    # MCP servers
    mcp_servers: dict[str, McpServerConfig] | str | Path = field(default_factory=dict)

    # Permission settings
    permission_mode: PermissionMode | None = None
    permission_prompt_tool_name: str | None = None
    can_use_tool: CanUseTool | None = None

    # Session management
    continue_conversation: bool = False
    resume: str | None = None
    fork_session: bool = False

    # Model settings
    model: str | None = None
    max_turns: int | None = None

    # Working directory
    cwd: str | Path | None = None
    add_dirs: list[str | Path] = field(default_factory=list)

    # Settings
    settings: str | None = None
    setting_sources: list[SettingSource] | None = None

    # Hooks
    hooks: dict[HookEvent, list[HookMatcher]] | None = None

    # Agents
    agents: dict[str, AgentDefinition] | None = None
    user: str | None = None

    # Advanced
    extra_args: dict[str, str | None] = field(default_factory=dict)
    max_buffer_size: int | None = None
    debug_stderr: Any = sys.stderr  # Deprecated
    stderr: Callable[[str], None] | None = None
    include_partial_messages: bool = False
```

**필드 그룹핑**:

1. **도구 제어**: `allowed_tools`, `disallowed_tools`
2. **프롬프트**: `system_prompt`
3. **MCP**: `mcp_servers`
4. **권한**: `permission_mode`, `can_use_tool`
5. **세션**: `continue_conversation`, `resume`, `fork_session`
6. **모델**: `model`, `max_turns`
7. **환경**: `cwd`, `add_dirs`, `settings`
8. **확장**: `hooks`, `agents`
9. **고급**: `extra_args`, `max_buffer_size`

**설계 패턴**:
- **Builder Pattern** (암시적): 모든 필드 선택적, 기본값 제공
- **Configuration Object**: 단일 객체로 모든 설정 관리

**설계 평가**:
- ✅ **Simplicity**: 기본값으로 쉬운 시작
- ⚠️ **Simplicity**: 필드 수 많음 (20개+), 복잡도 높음
- ✅ **Evolvability**: `extra_args`로 미래 확장 가능
- ✅ **Operability**: `stderr` 콜백으로 디버깅

**개선 제안**:
```python
# 그룹핑으로 단순화
@dataclass
class ClaudeAgentOptions:
    tool_config: ToolConfig = field(default_factory=ToolConfig)
    permission_config: PermissionConfig = field(default_factory=PermissionConfig)
    session_config: SessionConfig = field(default_factory=SessionConfig)
    # ...
```

---

### 7. Control Protocol Types

#### 7.1 SDK Control Request/Response
**라인**: 384-450

```python
class SDKControlInterruptRequest(TypedDict):
    subtype: Literal["interrupt"]

class SDKControlPermissionRequest(TypedDict):
    subtype: Literal["can_use_tool"]
    tool_name: str
    input: dict[str, Any]
    permission_suggestions: list[Any] | None
    blocked_path: str | None

class SDKHookCallbackRequest(TypedDict):
    subtype: Literal["hook_callback"]
    callback_id: str
    input: Any
    tool_use_id: str | None

class SDKControlMcpMessageRequest(TypedDict):
    subtype: Literal["mcp_message"]
    server_name: str
    message: Any

# ... 총 6가지 Request 타입

class SDKControlRequest(TypedDict):
    type: Literal["control_request"]
    request_id: str
    request: SDKControlInterruptRequest | ...

class ControlResponse(TypedDict):
    subtype: Literal["success"]
    request_id: str
    response: dict[str, Any] | None

class ControlErrorResponse(TypedDict):
    subtype: Literal["error"]
    request_id: str
    error: str

class SDKControlResponse(TypedDict):
    type: Literal["control_response"]
    response: ControlResponse | ControlErrorResponse
```

**프로토콜 흐름**:
```
CLI → SDKControlRequest
    {
        "type": "control_request",
        "request_id": "req-123",
        "request": {
            "subtype": "can_use_tool",
            "tool_name": "Bash",
            "input": {"command": "ls"}
        }
    }
    ↓
SDK processes (Query.handle_control_request)
    ↓
SDK → SDKControlResponse
    {
        "type": "control_response",
        "response": {
            "subtype": "success",
            "request_id": "req-123",
            "response": {"behavior": "allow"}
        }
    }
```

**설계 평가**:
- ✅ **Operability**: Request ID로 추적 가능
- ✅ **Simplicity**: Tagged union으로 타입 안전
- ✅ **Evolvability**: 새 request 타입 추가 용이

---

## 💡 설계 평가

### Operability (운영성) - ⭐⭐⭐⭐☆

**강점**:
1. ✅ **타입 안전성**: Literal, Union으로 잘못된 값 방지
2. ✅ **메트릭 포함**: `ResultMessage`에 비용, 시간, 사용량
3. ✅ **에러 처리**: `is_error`, `ToolResultBlock.is_error`
4. ✅ **디버깅**: `stderr` 콜백, `debug_stderr`

**개선점**:
- ⚠️ `hookSpecificOutput: Any` - 타입 안전성 부족
- ⚠️ `SystemMessage.data: dict[str, Any]` - 구조 불명확

---

### Simplicity (단순성) - ⭐⭐⭐⭐☆

**강점**:
1. ✅ **Dataclass 활용**: 보일러플레이트 최소화
2. ✅ **명확한 네이밍**: `PermissionResultAllow`, `TextBlock`
3. ✅ **기본값 제공**: `field(default_factory=list)`

**개선점**:
- ⚠️ `ClaudeAgentOptions` 필드 수 많음 (20개+)
- ⚠️ `PermissionUpdate` 6가지 타입 변형 복잡

---

### Evolvability (발전성) - ⭐⭐⭐⭐⭐

**강점**:
1. ✅✅ **Union 타입**: 새로운 메시지/블록 타입 추가 용이
2. ✅✅ **Literal 열거형**: 새 모드/이벤트 추가 시 타입 체크
3. ✅ **Tagged Union**: `type` 필드로 확장 가능
4. ✅ **`extra_args`**: 미래 CLI 플래그 지원
5. ✅ **TypedDict**: 런타임 오버헤드 없이 확장

**예시**:
```python
# 새 블록 타입 추가
@dataclass
class ImageBlock:
    url: str
    alt_text: str

ContentBlock = TextBlock | ... | ImageBlock  # ← 여기만 수정

# 새 Hook 이벤트 추가
HookEvent = ... | Literal["PreAPICall"]  # ← 여기만 수정
```

---

## 🔧 설계 패턴

| 패턴 | 위치 | 목적 |
|------|------|------|
| **Tagged Union** | `McpServerConfig`, `PermissionUpdate` | 타입 구분 |
| **Discriminated Union** | `Message`, `ContentBlock` | 다형성 |
| **Result Type** | `PermissionResult` | 성공/실패 표현 |
| **Builder Pattern** | `ClaudeAgentOptions` | 유연한 설정 |
| **Adapter** | `PermissionUpdate.to_dict()` | Python ↔ TypeScript |

---

## 🎯 주요 인사이트

### 1. **타입 시스템 활용**
- `Literal`: 컴파일 타임 검증
- `Union`: 다형성
- `TypedDict`: 런타임 오버헤드 없음
- `Dataclass`: 보일러플레이트 최소화

### 2. **TypeScript 영향**
- Tagged Union
- Discriminated Union
- `NotRequired` (TS의 `Partial`)

### 3. **최소 의존성**
- 외부 의존성 없음 (TYPE_CHECKING 제외)
- 순수 Python 타입만 사용

### 4. **확장성 우선**
- Union 타입으로 새 타입 추가 용이
- `extra_args`로 미래 대비

---

## 🚀 개선 제안

### 1. Options 그룹핑
```python
@dataclass
class ToolConfig:
    allowed: list[str] = field(default_factory=list)
    disallowed: list[str] = field(default_factory=list)

@dataclass
class ClaudeAgentOptions:
    tools: ToolConfig = field(default_factory=ToolConfig)
    permissions: PermissionConfig = field(default_factory=PermissionConfig)
    # ...
```

### 2. 타입 안전성 강화
```python
# Before
hookSpecificOutput: NotRequired[Any]

# After
class PreToolUseOutput(TypedDict):
    permissionDecision: Literal["allow", "deny"]
    permissionDecisionReason: str

hookSpecificOutput: NotRequired[PreToolUseOutput | PostToolUseOutput | ...]
```

### 3. Validation
```python
@dataclass
class PermissionUpdate:
    def __post_init__(self):
        if self.type in ["addRules", ...] and not self.rules:
            raise ValueError(f"{self.type} requires rules")
```

---

**분석 작성**: Claude Code
**분석 프레임워크**: Operability, Simplicity, Evolvability
