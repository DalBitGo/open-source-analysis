# 사용 패턴 & 실전 가이드

## 📋 개요

예제 코드 12개를 분석하여 추출한 **실전 사용 패턴**과 **일반적인 워크플로우**를 정리합니다.
이 문서는 "구현을 이해했는데, 실제로 어떻게 쓰지?"라는 질문에 답합니다.

---

## 🎓 난이도별 사용 패턴

### Level 1: 기본 쿼리 (3줄)

```python
from claude_agent_sdk import query

async for message in query("What is 2 + 2?"):
    print(message)
```

**사용 사례**: 단발성 질문, 빠른 프로토타이핑

**예제**: `quick_start.py:19`

---

### Level 2: 옵션 추가 (5줄)

```python
from claude_agent_sdk import query, ClaudeAgentOptions

options = ClaudeAgentOptions(
    system_prompt="You are a helpful assistant.",
    max_turns=1
)

async for message in query("Explain Python", options=options):
    print(message)
```

**사용 사례**: 시스템 프롬프트 커스터마이징, 비용 제한

**예제**: `quick_start.py:27`, `system_prompt.py`

---

### Level 3: 도구 사용 (10줄)

```python
from claude_agent_sdk import query, ClaudeAgentOptions, AssistantMessage, TextBlock

options = ClaudeAgentOptions(
    allowed_tools=["Read", "Write"],
    system_prompt="You are a file assistant."
)

async for message in query("Create hello.txt with 'Hello!'", options=options):
    if isinstance(message, AssistantMessage):
        for block in message.content:
            if isinstance(block, TextBlock):
                print(f"Claude: {block.text}")
```

**사용 사례**: 파일 작업, Bash 명령 실행

**예제**: `quick_start.py:46`

---

### Level 4: 양방향 대화 (20줄)

```python
from claude_agent_sdk import ClaudeSDKClient

async with ClaudeSDKClient() as client:
    # 첫 번째 질문
    await client.query("What's the capital of France?")
    async for msg in client.receive_response():
        print(msg)

    # 후속 질문 (컨텍스트 유지)
    await client.query("What's the population?")
    async for msg in client.receive_response():
        print(msg)
```

**사용 사례**: 멀티턴 대화, 컨텍스트 유지 필요

**예제**: `streaming_mode.py:74`

---

### Level 5: SDK MCP 도구 (40줄)

```python
from claude_agent_sdk import tool, create_sdk_mcp_server, ClaudeAgentOptions, query

@tool("add", "Add two numbers", {"a": float, "b": float})
async def add_numbers(args):
    result = args["a"] + args["b"]
    return {"content": [{"type": "text", "text": f"Result: {result}"}]}

server = create_sdk_mcp_server("calculator", tools=[add_numbers])

options = ClaudeAgentOptions(
    mcp_servers={"calculator": server},
    allowed_tools=["mcp__calculator__add"]
)

async for msg in query("What is 5 + 3?", options=options):
    print(msg)
```

**사용 사례**: 커스텀 비즈니스 로직, 애플리케이션 상태 접근

**예제**: `mcp_calculator.py`

---

### Level 6: 훅 시스템 (60줄)

```python
from claude_agent_sdk import ClaudeSDKClient, ClaudeAgentOptions, HookMatcher

async def check_bash(input_data, tool_use_id, context):
    if "rm -rf" in input_data["tool_input"]["command"]:
        return {
            "hookSpecificOutput": {
                "hookEventName": "PreToolUse",
                "permissionDecision": "deny",
                "permissionDecisionReason": "Dangerous command"
            }
        }
    return {}

options = ClaudeAgentOptions(
    allowed_tools=["Bash"],
    hooks={"PreToolUse": [HookMatcher(matcher="Bash", hooks=[check_bash])]}
)

async with ClaudeSDKClient(options=options) as client:
    await client.query("Run: rm -rf /")
    async for msg in client.receive_response():
        print(msg)
```

**사용 사례**: 보안 정책, 감사 로깅, 입력 검증

**예제**: `hooks.py:84`

---

### Level 7: 권한 콜백 (동적 제어)

```python
from claude_agent_sdk import (
    ClaudeSDKClient, ClaudeAgentOptions,
    PermissionResultAllow, PermissionResultDeny
)

async def permission_callback(tool_name, input_data, context):
    # 동적 권한 검증
    if tool_name == "Write" and "/etc" in input_data.get("file_path", ""):
        return PermissionResultDeny(message="Cannot write to /etc")

    # 입력 수정
    if tool_name == "Bash":
        safe_input = input_data.copy()
        safe_input["timeout"] = 30  # 타임아웃 강제
        return PermissionResultAllow(updated_input=safe_input)

    return PermissionResultAllow()

options = ClaudeAgentOptions(can_use_tool=permission_callback)

async with ClaudeSDKClient(options=options) as client:
    await client.query("Write to /etc/passwd")
    async for msg in client.receive_response():
        print(msg)
```

**사용 사례**: 실시간 권한 검증, 파라미터 주입

**예제**: `tool_permission_callback.py`

---

## 🔄 일반적인 워크플로우

### 워크플로우 1: 간단한 질문 (query)

```
사용자 질문
    ↓
query("질문", options)
    ↓
메시지 스트림 순회
    ↓
AssistantMessage 출력
    ↓
완료
```

**코드 예시:**
```python
async for message in query("What is Python?"):
    if isinstance(message, AssistantMessage):
        for block in message.content:
            if isinstance(block, TextBlock):
                print(block.text)
```

---

### 워크플로우 2: 멀티턴 대화 (ClaudeSDKClient)

```
ClaudeSDKClient 생성
    ↓
query() 전송
    ↓
receive_response() 순회
    ↓
다음 query() 전송 (컨텍스트 유지)
    ↓
receive_response() 순회
    ↓
...
    ↓
close (async with 자동)
```

**코드 예시:**
```python
async with ClaudeSDKClient() as client:
    await client.query("Tell me about Python")
    async for msg in client.receive_response():
        display(msg)

    await client.query("What's its history?")  # 이전 대화 기억
    async for msg in client.receive_response():
        display(msg)
```

**예제**: `streaming_mode.py:74`

---

### 워크플로우 3: SDK MCP 도구 제공

```
1. @tool 데코레이터로 함수 정의
    ↓
2. create_sdk_mcp_server(tools=[...])
    ↓
3. ClaudeAgentOptions(mcp_servers={...}, allowed_tools=[...])
    ↓
4. query() 또는 ClaudeSDKClient
    ↓
5. Claude가 도구 호출
    ↓
6. Python 함수 실행 (in-process)
    ↓
7. 결과를 Claude에게 반환
```

**코드 예시:**
```python
# 1. 도구 정의
@tool("get_weather", "Get weather", {"city": str})
async def get_weather(args):
    # 실제 API 호출
    weather = await fetch_weather_api(args["city"])
    return {"content": [{"type": "text", "text": weather}]}

# 2. 서버 생성
server = create_sdk_mcp_server("weather", tools=[get_weather])

# 3. 옵션 설정
options = ClaudeAgentOptions(
    mcp_servers={"weather": server},
    allowed_tools=["mcp__weather__get_weather"]
)

# 4. 사용
async for msg in query("What's the weather in Seoul?", options=options):
    print(msg)
```

**예제**: `mcp_calculator.py`

---

### 워크플로우 4: 훅으로 동작 제어

```
1. 훅 함수 정의
    ↓
2. ClaudeAgentOptions(hooks={event: [HookMatcher(...)]})
    ↓
3. ClaudeSDKClient 생성 (스트리밍 필수)
    ↓
4. initialize() 시 훅 등록 (자동)
    ↓
5. Claude가 도구 사용 시도
    ↓
6. CLI → SDK 훅 콜백 요청
    ↓
7. Python 훅 함수 실행
    ↓
8. 결과에 따라 허용/차단/수정
```

**코드 예시:**
```python
# 1. 훅 정의
async def audit_hook(input_data, tool_use_id, context):
    logger.info(f"Tool used: {input_data['tool_name']}")
    return {}  # 간섭 없이 통과

# 2. 등록
options = ClaudeAgentOptions(
    hooks={"PreToolUse": [HookMatcher(matcher=None, hooks=[audit_hook])]}
)

# 3. 사용
async with ClaudeSDKClient(options=options) as client:
    await client.query("Do something")
    async for msg in client.receive_response():
        print(msg)
```

**예제**: `hooks.py`

---

## 🎨 메시지 처리 패턴

### 패턴 1: 간단한 텍스트만 출력

```python
async for message in query("Hello"):
    if isinstance(message, AssistantMessage):
        for block in message.content:
            if isinstance(block, TextBlock):
                print(block.text)
```

**사용 사례**: 간단한 CLI 도구

---

### 패턴 2: 도구 사용 감지

```python
async for message in query("Create a file"):
    if isinstance(message, AssistantMessage):
        for block in message.content:
            if isinstance(block, ToolUseBlock):
                print(f"Using tool: {block.name}")
            elif isinstance(block, TextBlock):
                print(f"Claude: {block.text}")
```

**사용 사례**: 도구 실행 추적, UI 업데이트

**예제**: `streaming_mode.py:233`

---

### 패턴 3: 사고 과정 출력

```python
from claude_agent_sdk import ThinkingBlock

async for message in query("Complex problem"):
    if isinstance(message, AssistantMessage):
        for block in message.content:
            if isinstance(block, ThinkingBlock):
                print(f"[Thinking] {block.thinking}")
            elif isinstance(block, TextBlock):
                print(f"Claude: {block.text}")
```

**사용 사례**: 디버깅, 투명성 확보

---

### 패턴 4: 비용 추적

```python
total_cost = 0.0

async for message in query("Task"):
    if isinstance(message, ResultMessage):
        if message.total_cost_usd:
            total_cost += message.total_cost_usd
            print(f"Cost: ${message.total_cost_usd:.4f}")
```

**사용 사례**: 비용 모니터링, 예산 제한

**예제**: `quick_start.py:63`

---

### 패턴 5: 부분 메시지 포함

```python
options = ClaudeAgentOptions(include_partial_messages=True)

async for message in query("Long response", options=options):
    if isinstance(message, AssistantMessage):
        # 스트리밍 중에도 부분 응답 수신
        for block in message.content:
            if isinstance(block, TextBlock):
                print(block.text, end="", flush=True)
```

**사용 사례**: 실시간 UI 업데이트, 타이핑 효과

**예제**: `include_partial_messages.py`

---

## 🛠️ 고급 패턴

### 패턴 1: 애플리케이션 상태 공유

```python
# 애플리케이션 상태
class AppState:
    def __init__(self):
        self.db_connection = ...
        self.user_session = ...

app_state = AppState()

# SDK MCP 도구가 상태에 접근
@tool("query_db", "Query database", {"sql": str})
async def query_db(args):
    # 직접 상태 접근!
    results = await app_state.db_connection.execute(args["sql"])
    return {"content": [{"type": "text", "text": str(results)}]}

server = create_sdk_mcp_server("db", tools=[query_db])
```

**사용 사례**: 데이터베이스 연결, 세션 관리, 캐시 공유

---

### 패턴 2: 에러 처리가 있는 도구

```python
@tool("divide", "Divide numbers", {"a": float, "b": float})
async def divide(args):
    if args["b"] == 0:
        return {
            "content": [{"type": "text", "text": "Error: Division by zero"}],
            "is_error": True  # 에러 플래그
        }

    result = args["a"] / args["b"]
    return {"content": [{"type": "text", "text": f"Result: {result}"}]}
```

**사용 사례**: 에러를 Claude에게 전달하여 재시도 유도

**예제**: `mcp_calculator.py:52`

---

### 패턴 3: 로깅 훅

```python
import logging

async def logging_hook(input_data, tool_use_id, context):
    logging.info(f"Tool: {input_data['tool_name']}, Input: {input_data['tool_input']}")
    return {}  # 간섭 없음

options = ClaudeAgentOptions(
    hooks={"PreToolUse": [HookMatcher(matcher=None, hooks=[logging_hook])]}
)
```

**사용 사례**: 감사 로그, 디버깅

---

### 패턴 4: 입력 sanitization

```python
async def sanitize_file_path(input_data, tool_use_id, context):
    if input_data["tool_name"] == "Write":
        file_path = input_data["tool_input"].get("file_path", "")

        # 위험한 경로 차단
        if file_path.startswith("/etc") or file_path.startswith("/sys"):
            return {
                "hookSpecificOutput": {
                    "hookEventName": "PreToolUse",
                    "permissionDecision": "deny",
                    "permissionDecisionReason": "Cannot write to system directories"
                }
            }

    return {}
```

**사용 사례**: 보안 정책 적용

**예제**: `hooks.py:45`

---

### 패턴 5: 동적 모델 전환

```python
async with ClaudeSDKClient() as client:
    # 간단한 작업 - Haiku (저렴)
    await client.set_model("claude-3-5-haiku-20241022")
    await client.query("What is 2+2?")
    async for msg in client.receive_response(): print(msg)

    # 복잡한 작업 - Sonnet (강력)
    await client.set_model("claude-sonnet-4-20250514")
    await client.query("Solve this complex problem...")
    async for msg in client.receive_response(): print(msg)
```

**사용 사례**: 비용 최적화, 성능 조절

**예제**: `streaming_mode.py:369`

---

### 패턴 6: 에이전트 정의

```python
options = ClaudeAgentOptions(
    agents={
        "python-expert": AgentDefinition(
            description="Python programming expert",
            prompt="You are an expert Python developer. Always suggest type hints.",
            tools=["Read", "Write", "Bash"],
            model="sonnet"
        )
    }
)

# Claude가 /agent python-expert 명령으로 전환 가능
```

**사용 사례**: 특화된 에이전트, 역할 기반 동작

**예제**: `agents.py`

---

## 📝 코드 템플릿

### 템플릿 1: 기본 CLI 도구

```python
#!/usr/bin/env python3
import anyio
from claude_agent_sdk import query, AssistantMessage, TextBlock

async def main():
    prompt = input("You: ")

    async for message in query(prompt):
        if isinstance(message, AssistantMessage):
            for block in message.content:
                if isinstance(block, TextBlock):
                    print(f"Claude: {block.text}")

if __name__ == "__main__":
    anyio.run(main)
```

---

### 템플릿 2: 대화형 봇

```python
#!/usr/bin/env python3
import anyio
from claude_agent_sdk import ClaudeSDKClient, AssistantMessage, TextBlock

async def main():
    async with ClaudeSDKClient() as client:
        while True:
            prompt = input("\nYou: ")
            if prompt.lower() in ["exit", "quit"]:
                break

            await client.query(prompt)

            async for msg in client.receive_response():
                if isinstance(msg, AssistantMessage):
                    for block in msg.content:
                        if isinstance(block, TextBlock):
                            print(f"Claude: {block.text}")

if __name__ == "__main__":
    anyio.run(main)
```

---

### 템플릿 3: 파일 어시스턴트

```python
#!/usr/bin/env python3
import anyio
from claude_agent_sdk import query, ClaudeAgentOptions, AssistantMessage, TextBlock

async def main():
    options = ClaudeAgentOptions(
        allowed_tools=["Read", "Write", "Glob"],
        system_prompt="You are a file management assistant."
    )

    prompt = input("File task: ")

    async for message in query(prompt, options=options):
        if isinstance(message, AssistantMessage):
            for block in message.content:
                if isinstance(block, TextBlock):
                    print(block.text)

if __name__ == "__main__":
    anyio.run(main)
```

---

### 템플릿 4: 커스텀 도구 서버

```python
#!/usr/bin/env python3
import anyio
from claude_agent_sdk import (
    query, ClaudeAgentOptions, tool, create_sdk_mcp_server,
    AssistantMessage, TextBlock
)

@tool("my_tool", "Description", {"param": str})
async def my_tool(args):
    # 비즈니스 로직
    result = process(args["param"])
    return {"content": [{"type": "text", "text": result}]}

async def main():
    server = create_sdk_mcp_server("myserver", tools=[my_tool])

    options = ClaudeAgentOptions(
        mcp_servers={"myserver": server},
        allowed_tools=["mcp__myserver__my_tool"]
    )

    async for msg in query("Use my tool", options=options):
        if isinstance(msg, AssistantMessage):
            for block in msg.content:
                if isinstance(block, TextBlock):
                    print(block.text)

if __name__ == "__main__":
    anyio.run(main)
```

---

## 🎯 사용 사례별 추천 패턴

| 사용 사례 | 추천 패턴 | 난이도 |
|-----------|----------|--------|
| 간단한 질문 | `query()` | Level 1 |
| 시스템 프롬프트 커스터마이징 | `ClaudeAgentOptions` | Level 2 |
| 파일/Bash 작업 | `allowed_tools` | Level 3 |
| 멀티턴 대화 | `ClaudeSDKClient` | Level 4 |
| 커스텀 비즈니스 로직 | SDK MCP 도구 | Level 5 |
| 보안/감사 | 훅 시스템 | Level 6 |
| 동적 권한 검증 | 권한 콜백 | Level 7 |

---

## 💡 실전 팁

### 1. **anyio vs asyncio**

```python
# ✅ 권장 (anyio - asyncio + trio 호환)
import anyio
anyio.run(main)

# ✅ 허용 (asyncio만 사용)
import asyncio
asyncio.run(main())
```

### 2. **메시지 필터링**

```python
# 공통 display 함수 정의
def display_message(msg):
    if isinstance(msg, AssistantMessage):
        for block in msg.content:
            if isinstance(block, TextBlock):
                print(f"Claude: {block.text}")
    elif isinstance(msg, ResultMessage):
        print(f"Cost: ${msg.total_cost_usd:.4f}")

# 재사용
async for msg in client.receive_response():
    display_message(msg)
```

### 3. **에러 처리**

```python
from claude_agent_sdk import CLIConnectionError

try:
    async for msg in query("Hello"):
        print(msg)
except CLIConnectionError as e:
    print(f"Claude Code not found: {e}")
except Exception as e:
    print(f"Error: {e}")
```

### 4. **타임아웃**

```python
import anyio

async with ClaudeSDKClient() as client:
    await client.query("Long task")

    with anyio.fail_after(30):  # 30초 타임아웃
        async for msg in client.receive_response():
            print(msg)
```

### 5. **컨텍스트 윈도우 관리**

```python
options = ClaudeAgentOptions(
    max_turns=5  # 최대 5턴으로 제한
)

# 긴 대화에서 컨텍스트 초과 방지
```

---

**작성**: Claude Code
**분석 대상**: 예제 코드 12개
**난이도**: Level 1 (3줄) ~ Level 7 (권한 콜백)
**템플릿**: 4가지 즉시 사용 가능한 코드
