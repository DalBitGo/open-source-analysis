# 테스트 분석: 엣지 케이스 & 베스트 프랙티스

## 📋 개요

테스트 코드는 "어떻게 사용해야 하는가"를 보여주는 **실행 가능한 문서**입니다.
이 분석에서는 E2E 테스트와 유닛 테스트를 통해 발견한 중요한 사용 패턴과 주의사항을 정리합니다.

---

## 🧪 테스트 구조

### 테스트 분류

```
tests/
├── e2e-tests/                       # 7개 E2E 테스트 (실제 Claude API 호출)
│   ├── test_sdk_mcp_tools.py       # SDK MCP 서버 테스트
│   ├── test_tool_permissions.py    # 권한 콜백 테스트
│   ├── test_dynamic_control.py     # 동적 제어 (모델 변경, 인터럽트)
│   ├── test_agents_and_settings.py # 에이전트, 설정 소스
│   ├── test_include_partial_messages.py
│   ├── test_stderr_callback.py
│   └── conftest.py
└── tests/                           # 3개 유닛 테스트 (Mock 사용)
    ├── test_tool_callbacks.py       # 콜백 로직 단위 테스트
    ├── test_subprocess_buffering.py
    └── test_changelog.py

**마커:**
- `@pytest.mark.e2e`: 실제 API 호출 (비용 발생, 느림)
- `@pytest.mark.asyncio`: 비동기 테스트
```

---

## 🎯 핵심 발견 사항

### 1. **SDK MCP 도구 네이밍 규칙** ⚠️

```python
# ❌ 틀린 사용법
server = create_sdk_mcp_server("myserver", tools=[echo_tool])
options = ClaudeAgentOptions(
    mcp_servers={"myserver": server},
    allowed_tools=["echo"]  # ❌ 동작 안 함!
)

# ✅ 올바른 사용법
options = ClaudeAgentOptions(
    mcp_servers={"myserver": server},
    allowed_tools=["mcp__myserver__echo"]  # 형식: mcp__{서버명}__{도구명}
)
```

**발견 위치**: `test_sdk_mcp_tools.py:39`

**이유**: CLI가 MCP 도구를 `mcp__{server_name}__{tool_name}` 형식으로 네이밍

---

### 2. **SDK MCP 도구는 기본적으로 차단됨** 🔒

```python
# Test case: test_sdk_mcp_without_permissions (line 139)
server = create_sdk_mcp_server("noperm", tools=[echo_tool])
options = ClaudeAgentOptions(
    mcp_servers={"noperm": server}
    # ❌ allowed_tools 없음
)

# 결과: echo_tool이 실행되지 않음!
assert "echo" not in executions  # 통과
```

**교훈**: SDK MCP 서버를 추가해도 `allowed_tools`에 명시하지 않으면 사용 불가

---

### 3. **권한 콜백 사용 시 스트리밍 모드 필수** ⚠️

```python
# ✅ 올바른 사용법
async with ClaudeSDKClient(options=options) as client:
    await client.query("...")  # 스트리밍 모드
    async for msg in client.receive_response():
        pass

# ❌ 단방향 모드는 can_use_tool과 호환 불가
async for msg in query("...", options=ClaudeAgentOptions(can_use_tool=callback)):
    pass  # ValueError 발생!
```

**발견 위치**: `test_tool_permissions.py:30`, `_internal/client.py:54`

**이유**: 콜백 실행을 위해 양방향 제어 프로토콜 필요

---

### 4. **`disallowed_tools`는 `allowed_tools`보다 우선** 🚫

```python
# Test case: test_sdk_mcp_permission_enforcement (line 54)
options = ClaudeAgentOptions(
    mcp_servers={"test": server},
    disallowed_tools=["mcp__test__echo"],  # 차단
    allowed_tools=["mcp__test__greet"]      # 허용
)

# Claude가 echo와 greet 모두 사용 시도
# 결과:
assert "echo" not in executions  # echo는 차단됨
assert "greet" in executions     # greet은 실행됨
```

**교훈**: 블랙리스트(disallowed) + 화이트리스트(allowed) 조합 가능

---

### 5. **권한 콜백으로 입력 수정 가능** 🔧

```python
# Test case: test_permission_callback_input_modification (line 132)
async def modify_callback(tool_name, input_data, context):
    modified_input = input_data.copy()
    modified_input["safe_mode"] = True  # 안전 플래그 추가
    return PermissionResultAllow(updated_input=modified_input)

# CLI는 수정된 입력으로 도구 실행
# 원래: {"file_path": "/etc/passwd"}
# 수정: {"file_path": "/etc/passwd", "safe_mode": true}
```

**발견 위치**: `test_tool_callbacks.py:132-168`

**사용 사례**:
- 위험한 경로 sanitization
- 기본 파라미터 주입
- 로깅 컨텍스트 추가

---

### 6. **동적 모델 변경 가능** 🔄

```python
# Test case: test_set_model (line 44)
async with ClaudeSDKClient(options=options) as client:
    # 기본 모델로 시작
    await client.query("What is 1+1?")
    async for msg in client.receive_response(): pass

    # Haiku로 변경
    await client.set_model("claude-3-5-haiku-20241022")
    await client.query("What is 2+2?")
    async for msg in client.receive_response(): pass

    # 기본 모델로 복귀
    await client.set_model(None)
    await client.query("What is 3+3?")
    async for msg in client.receive_response(): pass
```

**발견 위치**: `test_dynamic_control.py:44-73`

**사용 사례**:
- 간단한 작업은 Haiku (저렴)
- 복잡한 작업은 Sonnet (강력)

---

### 7. **설정 소스 기본값: 빈 배열** 📂

```python
# Test case: test_setting_sources_default (line 52)
options = ClaudeAgentOptions(
    cwd=project_dir
    # setting_sources 지정 안 함
)

# 결과: 프로젝트 로컬 설정이 로드되지 않음!
# outputStyle == "default" (로컬 설정 무시)
```

**vs**

```python
# setting_sources 명시
options = ClaudeAgentOptions(
    cwd=project_dir,
    setting_sources=["user", "project", "local"]
)

# 결과: 로컬 설정이 로드됨
# outputStyle == "local-test-style"
```

**발견 위치**: `test_agents_and_settings.py:52-88`

**교훈**: 프로젝트 설정을 사용하려면 명시적으로 `setting_sources` 지정 필요

---

### 8. **콜백 예외는 에러 응답으로 변환** 🛡️

```python
# Test case: test_callback_exception_handling (line 171)
async def error_callback(tool_name, input_data, context):
    raise ValueError("Callback error")

# CLI에게 전송되는 응답:
{
    "type": "control_response",
    "response": {
        "subtype": "error",
        "request_id": "...",
        "error": "Callback error"
    }
}
```

**발견 위치**: `test_tool_callbacks.py:171-204`

**교훈**: 콜백 내 예외는 자동으로 catch되어 CLI에 전달됨 (프로세스 크래시 방지)

---

### 9. **에이전트 정의는 초기화 시 사용 가능** 🤖

```python
# Test case: test_agent_definition (line 20)
options = ClaudeAgentOptions(
    agents={
        "test-agent": AgentDefinition(
            description="A test agent",
            prompt="You are a test agent. Always respond with...",
            tools=["Read"],
            model="sonnet"
        )
    }
)

async with ClaudeSDKClient(options=options) as client:
    await client.query("...")

    async for message in client.receive_response():
        if isinstance(message, SystemMessage) and message.subtype == "init":
            agents = message.data.get("agents", [])
            assert "test-agent" in agents  # 확인 가능
```

**발견 위치**: `test_agents_and_settings.py:20-47`

**SystemMessage의 init 서브타입**: 세션 초기화 정보 포함

---

### 10. **인터럽트는 "베스트 에포트"** ⏸️

```python
# Test case: test_interrupt (line 78)
async with ClaudeSDKClient() as client:
    await client.query("Count from 1 to 100 slowly.")

    # 즉시 인터럽트 전송
    await client.interrupt()

    # 하지만 응답이 이미 완료되었을 수도 있음
    async for message in client.receive_response():
        pass  # 타이밍에 따라 전체 응답이 올 수도 있음
```

**발견 위치**: `test_dynamic_control.py:78-97`

**교훈**: `interrupt()`는 즉시 중단을 보장하지 않음 (레이스 컨디션)

---

## 🏗️ Mock Transport 패턴

유닛 테스트에서 사용하는 패턴:

```python
class MockTransport(Transport):
    def __init__(self):
        self.written_messages = []  # 전송된 메시지 기록
        self.messages_to_read = []  # 읽을 메시지 큐
        self._connected = False

    async def connect(self): self._connected = True
    async def close(self): self._connected = False

    async def write(self, data: str):
        self.written_messages.append(data)  # 검증용 저장

    def read_messages(self):
        async def _read():
            for msg in self.messages_to_read:
                yield msg
        return _read()

# 사용 예시
transport = MockTransport()
query = Query(transport=transport, is_streaming_mode=True, can_use_tool=callback)

# 요청 시뮬레이션
request = {"type": "control_request", "request_id": "test-1", ...}
await query._handle_control_request(request)

# 응답 검증
assert len(transport.written_messages) == 1
assert '"behavior": "allow"' in transport.written_messages[0]
```

**발견 위치**: `test_tool_callbacks.py:17-46`

**장점**:
- 실제 CLI 프로세스 불필요
- 빠른 테스트 실행
- 엣지 케이스 정밀 제어

---

## 📊 테스트 커버리지 분석

### E2E 테스트가 검증하는 것

| 기능 | 테스트 파일 | 검증 내용 |
|------|------------|----------|
| SDK MCP 도구 실행 | `test_sdk_mcp_tools.py` | 네이밍 규칙, 권한, 여러 도구 |
| 권한 콜백 | `test_tool_permissions.py` | 콜백 호출, allow/deny |
| 동적 제어 | `test_dynamic_control.py` | 모델 변경, 권한 모드, 인터럽트 |
| 에이전트 & 설정 | `test_agents_and_settings.py` | 에이전트 정의, 설정 소스 |

### 유닛 테스트가 검증하는 것

| 기능 | 테스트 파일 | 검증 내용 |
|------|------------|----------|
| 콜백 로직 | `test_tool_callbacks.py` | 입력 수정, 예외 처리, 훅 실행 |

### 빠진 테스트 (추론)

- ❓ SDK MCP 에러 처리 (도구 내 예외)
- ❓ 훅 체인 실행 (여러 훅 순차 호출)
- ❓ 대용량 스트리밍 (백프레셔 처리)
- ❓ 재연결 시나리오 (CLI 크래시 복구)

---

## 💡 베스트 프랙티스 (테스트에서 추출)

### 1. **도구 네이밍은 명확하게**

```python
# ✅ Good
server = create_sdk_mcp_server("calculator", tools=[add, subtract])
allowed_tools=["mcp__calculator__add", "mcp__calculator__subtract"]

# ❌ Bad
allowed_tools=["add", "subtract"]  # 동작 안 함
```

### 2. **권한 콜백은 항상 ClaudeSDKClient 사용**

```python
# ✅ Good
async with ClaudeSDKClient(options=ClaudeAgentOptions(can_use_tool=callback)) as client:
    await client.query("...")

# ❌ Bad
async for msg in query("...", options=ClaudeAgentOptions(can_use_tool=callback)):
    pass  # ValueError!
```

### 3. **콜백 내 예외는 자동 처리됨**

```python
# 안전함 - 예외가 발생해도 프로세스 크래시 안 함
async def my_callback(tool_name, input_data, context):
    if dangerous(input_data):
        raise ValueError("Dangerous input!")  # CLI에 에러 응답 전송
    return PermissionResultAllow()
```

### 4. **설정 소스는 명시적으로**

```python
# ✅ 프로젝트 설정 사용
options = ClaudeAgentOptions(
    cwd=project_dir,
    setting_sources=["user", "project", "local"]
)

# ❌ 기본값은 빈 배열 (설정 없음)
options = ClaudeAgentOptions(cwd=project_dir)
```

### 5. **메시지 타입별 처리**

```python
async for message in client.receive_response():
    if isinstance(message, AssistantMessage):
        # 텍스트, 사고, 도구 사용
        for block in message.content:
            if isinstance(block, TextBlock):
                print(block.text)

    elif isinstance(message, SystemMessage):
        if message.subtype == "init":
            # 초기화 정보 (에이전트, 설정 등)
            agents = message.data.get("agents", [])

    elif isinstance(message, ResultMessage):
        # 완료 정보 (비용, 소요 시간 등)
        if message.total_cost_usd > 0:
            print(f"Cost: ${message.total_cost_usd:.4f}")
```

---

## 🔍 엣지 케이스

### 1. **빈 도구 리스트**

```python
# 허용됨 - 도구 없는 MCP 서버
server = create_sdk_mcp_server("empty", tools=[])
```

### 2. **None 모델 (기본 모델로 복귀)**

```python
await client.set_model("claude-3-5-haiku-20241022")  # Haiku 사용
await client.set_model(None)  # 기본 모델로 복귀
```

### 3. **입력 수정 시 updated_input**

```python
# 원본 입력을 변경하면 안 됨!
async def bad_callback(tool_name, input_data, context):
    input_data["safe_mode"] = True  # ❌ 원본 수정
    return PermissionResultAllow()

# 복사본 생성 후 반환
async def good_callback(tool_name, input_data, context):
    modified = input_data.copy()
    modified["safe_mode"] = True
    return PermissionResultAllow(updated_input=modified)  # ✅
```

---

## 📈 테스트에서 배운 아키텍처 인사이트

### 1. **설정 소스 기본값이 빈 배열인 이유**

보안상의 이유 - 의도하지 않은 프로젝트 설정 로드 방지

### 2. **SDK MCP 도구 네이밍에 서버명 포함 이유**

여러 MCP 서버에서 같은 이름의 도구 충돌 방지
- Server A: `add` + Server B: `add` → `mcp__A__add`, `mcp__B__add`

### 3. **콜백 예외 자동 처리 이유**

사용자 코드의 버그로 인한 전체 세션 크래시 방지
- 콜백 실패 → 해당 도구만 차단, 세션은 계속

### 4. **인터럽트가 베스트 에포트인 이유**

분산 시스템의 특성 - Python SDK와 Node.js CLI 간 타이밍
- 메시지가 이미 전송 중이면 취소 불가

---

## 🎯 테스트 작성 시 주의사항

### E2E 테스트

```python
@pytest.mark.e2e  # 필수 마커
@pytest.mark.asyncio
async def test_my_feature():
    # API 비용 발생 - 최소한의 쿼리로
    options = ClaudeAgentOptions(max_turns=1)  # 비용 절감

    async with ClaudeSDKClient(options=options) as client:
        await client.query("Simple question")

        async for msg in client.receive_response():
            # 검증 로직
            pass
```

### 유닛 테스트

```python
@pytest.mark.asyncio
async def test_callback_logic():
    transport = MockTransport()
    query = Query(transport=transport, is_streaming_mode=True, ...)

    # 직접 제어 요청 시뮬레이션
    request = {"type": "control_request", ...}
    await query._handle_control_request(request)

    # 응답 검증
    assert '"behavior": "allow"' in transport.written_messages[0]
```

---

**작성**: Claude Code
**분석 대상**: E2E 테스트 7개, 유닛 테스트 3개
**주요 발견**: 10가지 핵심 패턴 + 5가지 베스트 프랙티스
