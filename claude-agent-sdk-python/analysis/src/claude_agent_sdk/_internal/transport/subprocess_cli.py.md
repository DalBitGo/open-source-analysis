# _internal/transport/subprocess_cli.py

**원본 경로**: `source/src/claude_agent_sdk/_internal/transport/subprocess_cli.py`
**역할**: Claude Code CLI와의 서브프로세스 통신 구현
**라인 수**: 498줄
**의존성**: `anyio`, `_errors`, `types`, `Transport`

---

## 📊 구조 개요

- **클래스**: 1개 (`SubprocessCLITransport`)
- **메서드**: 10개 (Public 5개, Private 5개)
- **복잡도**: 높음 (프로세스 관리, 스트림 핸들링, 에러 처리)
- **핵심 역할**: Python ↔ Node.js CLI 브릿지

---

## 🔍 상세 분석

### 클래스: `SubprocessCLITransport`

**라인**: 33-498

**상속**: `Transport` (Protocol/Abstract)

---

## 📋 클래스 변수 및 상수

### 상수
**라인**: 29-30

```python
_DEFAULT_MAX_BUFFER_SIZE = 1024 * 1024  # 1MB buffer limit
MINIMUM_CLAUDE_CODE_VERSION = "2.0.0"
```

**설계 평가**:
- ✅ **Operability**: 버퍼 오버플로우 방지
- ✅ **Evolvability**: 버전 호환성 관리

---

### 인스턴스 변수
**라인**: 36-58

```python
def __init__(self, prompt, options, cli_path=None):
    self._prompt = prompt
    self._is_streaming = not isinstance(prompt, str)
    self._options = options
    self._cli_path = str(cli_path) if cli_path else self._find_cli()
    self._cwd = str(options.cwd) if options.cwd else None

    # Process & Streams
    self._process: Process | None = None
    self._stdout_stream: TextReceiveStream | None = None
    self._stdin_stream: TextSendStream | None = None
    self._stderr_stream: TextReceiveStream | None = None
    self._stderr_task_group: anyio.abc.TaskGroup | None = None

    # State
    self._ready = False
    self._exit_error: Exception | None = None
    self._max_buffer_size = options.max_buffer_size or _DEFAULT_MAX_BUFFER_SIZE
```

**상태 관리**:
- Process lifecycle
- Stream handles
- Error tracking

---

## 🔧 메서드 상세 분석

### 1. `_find_cli()`
**라인**: 60-84

```python
def _find_cli(self) -> str:
    """Find Claude Code CLI binary."""
    if cli := shutil.which("claude"):
        return cli

    locations = [
        Path.home() / ".npm-global/bin/claude",
        Path("/usr/local/bin/claude"),
        Path.home() / ".local/bin/claude",
        Path.home() / "node_modules/.bin/claude",
        Path.home() / ".yarn/bin/claude",
    ]

    for path in locations:
        if path.exists() and path.is_file():
            return str(path)

    raise CLINotFoundError(
        "Claude Code not found. Install with:\n"
        "  npm install -g @anthropic-ai/claude-code\n"
        "\nIf already installed locally, try:\n"
        '  export PATH="$HOME/node_modules/.bin:$PATH"\n'
        "\nOr specify the path when creating transport:\n"
        "  SubprocessCLITransport(..., cli_path='/path/to/claude')"
    )
```

**검색 순서**:
1. `shutil.which("claude")` - PATH 환경변수
2. `~/.npm-global/bin/claude` - npm global
3. `/usr/local/bin/claude` - 시스템 전역
4. `~/.local/bin/claude` - 사용자 로컬
5. `~/node_modules/.bin/claude` - 프로젝트 로컬
6. `~/.yarn/bin/claude` - yarn 전역

**설계 평가**:
- ✅✅ **Operability**: 상세한 에러 메시지 (설치 방법 포함)
- ✅ **Simplicity**: 일반적인 위치 모두 검색
- ✅ **Evolvability**: `cli_path` 오버라이드 가능

---

### 2. `_build_command()`
**라인**: 86-201

**역할**: CLI 명령어 및 인자 빌드

**핵심 구조**:
```python
def _build_command(self) -> list[str]:
    cmd = [self._cli_path, "--output-format", "stream-json", "--verbose"]

    # System prompt
    if self._options.system_prompt:
        cmd.extend(["--system-prompt", self._options.system_prompt])

    # Tools
    if self._options.allowed_tools:
        cmd.extend(["--allowedTools", ",".join(self._options.allowed_tools)])

    # MCP servers (SDK vs External 분리)
    if self._options.mcp_servers:
        if isinstance(self._options.mcp_servers, dict):
            servers_for_cli = {}
            for name, config in self._options.mcp_servers.items():
                if config.get("type") == "sdk":
                    # SDK 서버: instance 필드 제거
                    sdk_config = {k: v for k, v in config.items() if k != "instance"}
                    servers_for_cli[name] = sdk_config
                else:
                    # External 서버: 그대로
                    servers_for_cli[name] = config

            if servers_for_cli:
                cmd.extend(["--mcp-config", json.dumps({"mcpServers": servers_for_cli})])

    # Input mode
    if self._is_streaming:
        cmd.extend(["--input-format", "stream-json"])
    else:
        cmd.extend(["--print", "--", str(self._prompt)])

    return cmd
```

**SDK MCP 서버 처리** (라인 138-162):
```python
# SDK 서버 → instance 제거
{"type": "sdk", "name": "calc", "instance": <Server>}
    ↓
{"type": "sdk", "name": "calc"}  # CLI에 전달

# 이유: Python MCP Server instance는 CLI에 전달 불가
# SDK가 Control Protocol로 직접 처리
```

**설계 평가**:
- ✅ **Operability**: 모든 옵션을 CLI 플래그로 변환
- ✅ **Evolvability**: `extra_args`로 미래 플래그 지원
- ✅ **Simplicity**: 명확한 if-else 구조

---

### 3. `connect()`
**라인**: 203-276

**역할**: 서브프로세스 시작 및 스트림 설정

**핵심 단계**:

#### 3.1 환경변수 설정
**라인**: 212-221

```python
process_env = {
    **os.environ,                           # 시스템 환경변수
    **self._options.env,                    # 사용자 제공 환경변수
    "CLAUDE_CODE_ENTRYPOINT": "sdk-py",     # 텔레메트리
    "CLAUDE_AGENT_SDK_VERSION": __version__, # SDK 버전
}

if self._cwd:
    process_env["PWD"] = self._cwd
```

**우선순위**: 시스템 < 사용자 < SDK

---

#### 3.2 Stderr 처리 결정
**라인**: 223-230

```python
should_pipe_stderr = (
    self._options.stderr is not None
    or "debug-to-stderr" in self._options.extra_args
)

stderr_dest = PIPE if should_pipe_stderr else None
```

**조건**:
- `stderr` 콜백 제공 → Pipe
- `debug-to-stderr` 플래그 → Pipe
- 둘 다 없음 → `/dev/null` (무시)

---

#### 3.3 프로세스 생성
**라인**: 232-240

```python
self._process = await anyio.open_process(
    cmd,
    stdin=PIPE,
    stdout=PIPE,
    stderr=stderr_dest,
    cwd=self._cwd,
    env=process_env,
    user=self._options.user,  # Unix: 다른 사용자로 실행 가능
)
```

**anyio.open_process**:
- asyncio + trio 양쪽 지원
- 비동기 스트림 제공

---

#### 3.4 스트림 설정
**라인**: 242-259

```python
# Stdout (항상 필요)
if self._process.stdout:
    self._stdout_stream = TextReceiveStream(self._process.stdout)

# Stderr (조건부)
if should_pipe_stderr and self._process.stderr:
    self._stderr_stream = TextReceiveStream(self._process.stderr)
    # 백그라운드 태스크 시작
    self._stderr_task_group = anyio.create_task_group()
    await self._stderr_task_group.__aenter__()
    self._stderr_task_group.start_soon(self._handle_stderr)

# Stdin (모드별 처리)
if self._is_streaming and self._process.stdin:
    self._stdin_stream = TextSendStream(self._process.stdin)
elif not self._is_streaming and self._process.stdin:
    # String 모드: stdin 즉시 닫기 (CLI가 --print 옵션으로 처리)
    await self._process.stdin.aclose()
```

**설계 패턴**: **Asynchronous Pipeline**

---

#### 3.5 에러 처리
**라인**: 262-276

```python
except FileNotFoundError as e:
    # 작업 디렉토리 vs CLI 바이너리 구분
    if self._cwd and not Path(self._cwd).exists():
        error = CLIConnectionError(f"Working directory does not exist: {self._cwd}")
    else:
        error = CLINotFoundError(f"Claude Code not found at: {self._cli_path}")
    self._exit_error = error
    raise error from e

except Exception as e:
    error = CLIConnectionError(f"Failed to start Claude Code: {e}")
    self._exit_error = error
    raise error from e
```

**설계 평가**:
- ✅✅ **Operability**: 구체적인 에러 타입, 원인 체인 (`from e`)
- ✅ **Debuggability**: `_exit_error` 저장

---

### 4. `_handle_stderr()`
**라인**: 278-304

```python
async def _handle_stderr(self) -> None:
    """Handle stderr stream - read and invoke callbacks."""
    if not self._stderr_stream:
        return

    try:
        async for line in self._stderr_stream:
            line_str = line.rstrip()
            if not line_str:
                continue

            # 우선순위 1: stderr 콜백
            if self._options.stderr:
                self._options.stderr(line_str)

            # 우선순위 2: debug_stderr (하위 호환)
            elif (
                "debug-to-stderr" in self._options.extra_args
                and self._options.debug_stderr
            ):
                self._options.debug_stderr.write(line_str + "\n")
                if hasattr(self._options.debug_stderr, "flush"):
                    self._options.debug_stderr.flush()

    except anyio.ClosedResourceError:
        pass  # 정상 종료
    except Exception:
        pass  # Stderr 에러는 무시 (메인 프로세스에 영향 없음)
```

**설계 특징**:
- **백그라운드 태스크**: `start_soon()`로 비동기 실행
- **에러 격리**: Stderr 에러가 메인 로직 방해 안 함
- **하위 호환성**: `debug_stderr` 지원 유지

**설계 평가**:
- ✅ **Operability**: 콜백으로 유연한 처리
- ✅ **Robustness**: 에러 무시로 안정성 확보
- ✅ **Evolvability**: 콜백 → 커스텀 처리 가능

---

### 5. `close()`
**라인**: 306-348

```python
async def close(self) -> None:
    """Close the transport and clean up resources."""
    self._ready = False

    if not self._process:
        return

    # 1. Stderr task group 종료
    if self._stderr_task_group:
        with suppress(Exception):
            self._stderr_task_group.cancel_scope.cancel()
            await self._stderr_task_group.__aexit__(None, None, None)
        self._stderr_task_group = None

    # 2. 모든 스트림 닫기
    if self._stdin_stream:
        with suppress(Exception):
            await self._stdin_stream.aclose()
        self._stdin_stream = None

    if self._stderr_stream:
        with suppress(Exception):
            await self._stderr_stream.aclose()
        self._stderr_stream = None

    if self._process.stdin:
        with suppress(Exception):
            await self._process.stdin.aclose()

    # 3. 프로세스 종료
    if self._process.returncode is None:
        with suppress(ProcessLookupError):
            self._process.terminate()
            with suppress(Exception):
                await self._process.wait()

    # 4. 모든 상태 초기화
    self._process = None
    self._stdout_stream = None
    self._stdin_stream = None
    self._stderr_stream = None
    self._exit_error = None
```

**정리 순서**:
1. Stderr 태스크 취소
2. 모든 스트림 닫기
3. 프로세스 종료
4. 상태 초기화

**설계 패턴**: **RAII (Resource Acquisition Is Initialization)**

**설계 평가**:
- ✅✅ **Operability**: 완벽한 자원 정리
- ✅ **Robustness**: `suppress(Exception)` - 부분 실패 허용
- ✅ **Simplicity**: 명확한 정리 순서

---

### 6. `write()`
**라인**: 350-380 (추정)

```python
async def write(self, data: str) -> None:
    """Write data to stdin stream."""
    if not self._ready or not self._stdin_stream:
        raise CLIConnectionError("Transport not connected")

    try:
        await self._stdin_stream.send(data)
    except anyio.BrokenResourceError as e:
        raise CLIConnectionError("CLI process stdin closed") from e
```

**역할**: JSON Lines를 stdin에 쓰기

**설계 평가**:
- ✅ **Operability**: 상태 검증
- ✅ **Error Handling**: BrokenResourceError → CLIConnectionError

---

### 7. `read()`
**라인**: 382-430 (추정)

```python
async def read(self) -> AsyncIterator[str]:
    """Read lines from stdout stream."""
    if not self._ready or not self._stdout_stream:
        raise CLIConnectionError("Transport not connected")

    buffer_size = 0

    try:
        async for line in self._stdout_stream:
            buffer_size += len(line)

            # 버퍼 오버플로우 방지
            if buffer_size > self._max_buffer_size:
                raise CLIConnectionError(
                    f"Output buffer exceeded {self._max_buffer_size} bytes"
                )

            yield line.rstrip()

        # 프로세스 종료 후 에러 체크
        if self._process and self._process.returncode != 0:
            raise ProcessError(
                "CLI process exited with error",
                exit_code=self._process.returncode
            )

    except anyio.ClosedResourceError:
        if self._exit_error:
            raise self._exit_error
        raise CLIConnectionError("CLI process stdout closed")
```

**보호 메커니즘**:
1. **상태 검증**: `_ready`, `_stdout_stream` 체크
2. **버퍼 제한**: `_max_buffer_size` 초과 시 에러
3. **종료 코드 체크**: 비정상 종료 감지
4. **에러 전파**: `_exit_error` 저장 후 재발생

**설계 평가**:
- ✅✅ **Operability**: 다층 에러 처리
- ✅ **Robustness**: 버퍼 오버플로우 방지
- ✅ **Debuggability**: 종료 코드 포함한 에러

---

### 8. `_check_claude_version()`
**라인**: 추정 위치

```python
async def _check_claude_version(self) -> None:
    """Check if Claude Code version meets minimum requirement."""
    try:
        result = await anyio.run_process(
            [self._cli_path, "--version"],
            check=False
        )
        version_output = result.stdout.decode().strip()
        # Parse version and compare with MINIMUM_CLAUDE_CODE_VERSION
        # ...
    except Exception:
        # 버전 체크 실패는 경고만, 계속 진행
        pass
```

**설계 평가**:
- ✅ **Evolvability**: 호환성 보장
- ✅ **Robustness**: 버전 체크 실패해도 계속 진행

---

## 🔄 라이프사이클

```
[초기화]
    ↓ __init__()
[생성됨]
    ↓ connect()
    ├─ _find_cli()
    ├─ _build_command()
    ├─ _check_claude_version()
    ├─ anyio.open_process()
    ├─ TextReceiveStream (stdout, stderr)
    ├─ TextSendStream (stdin)
    └─ _handle_stderr() 백그라운드 시작
[연결됨]
    ↓ write() / read() 반복
[통신 중]
    ↓ close()
    ├─ cancel stderr task
    ├─ close all streams
    ├─ terminate process
    └─ clear state
[종료됨]
```

---

## 💡 설계 평가

### Operability (운영성) - ⭐⭐⭐⭐⭐

**강점**:
1. ✅✅ **완벽한 에러 처리**
   - `CLINotFoundError`: CLI 바이너리 없음
   - `CLIConnectionError`: 연결 실패, 작업 디렉토리 없음
   - `ProcessError`: 프로세스 비정상 종료
   - 각 에러마다 상세한 메시지

2. ✅✅ **자원 관리**
   - 모든 스트림 명시적 닫기
   - `suppress(Exception)`로 부분 실패 허용
   - RAII 패턴

3. ✅ **버퍼 오버플로우 방지**
   - `_max_buffer_size` (기본 1MB)
   - 실시간 체크

4. ✅ **Stderr 처리**
   - 콜백 시스템
   - 백그라운드 태스크
   - 하위 호환 (`debug_stderr`)

5. ✅ **환경변수 관리**
   - 시스템 < 사용자 < SDK 우선순위
   - 텔레메트리 (`CLAUDE_CODE_ENTRYPOINT`)

---

### Simplicity (단순성) - ⭐⭐⭐☆☆

**강점**:
1. ✅ **명확한 책임**: Transport 역할만
2. ✅ **단계별 초기화**: connect() 내부 명확한 단계

**약점**:
1. ⚠️ **복잡도 높음**: 498줄, 10개 메서드
2. ⚠️ **상태 변수 많음**: 8개 인스턴스 변수
3. ⚠️ **조건 분기**: MCP 서버 처리, Stderr 처리 등

**평가**: 불가피한 복잡도 (프로세스 관리 특성상)

---

### Evolvability (발전성) - ⭐⭐⭐⭐⭐

**강점**:
1. ✅ **Transport 인터페이스**: 구현체 교체 가능
2. ✅ **`extra_args`**: 미래 CLI 플래그 지원
3. ✅ **콜백 시스템**: `stderr` 커스터마이징
4. ✅ **버전 체크**: 호환성 관리
5. ✅ **환경변수 우선순위**: 사용자 오버라이드 가능

**확장 시나리오**:

#### 시나리오 1: WebSocket Transport
```python
class WebSocketTransport(Transport):
    async def connect(self):
        self._ws = await websockets.connect("wss://...")

    async def write(self, data: str):
        await self._ws.send(data)

    async def read(self) -> AsyncIterator[str]:
        async for msg in self._ws:
            yield msg
```

#### 시나리오 2: SSH Transport
```python
class SSHTransport(Transport):
    async def connect(self):
        self._ssh = await asyncssh.connect(host, ...)

    async def write(self, data: str):
        self._ssh.stdin.write(data)

    async def read(self) -> AsyncIterator[str]:
        async for line in self._ssh.stdout:
            yield line
```

---

## 🔧 설계 패턴

| 패턴 | 위치 | 목적 |
|------|------|------|
| **Adapter** | `SubprocessCLITransport` | Node.js CLI ↔ Python |
| **Pipeline** | `stdout/stderr/stdin` | 데이터 스트리밍 |
| **RAII** | `close()` | 자원 자동 정리 |
| **Error Handling Hierarchy** | `_errors.py` | 구체적 에러 타입 |
| **Builder** | `_build_command()` | 복잡한 명령어 생성 |
| **Callback** | `stderr` | 커스터마이징 포인트 |

---

## 🎯 주요 인사이트

### 1. **Python ↔ Node.js 브릿지**
```
Python SDK
    ↓ subprocess
Node.js CLI (Claude Code)
    ↓ HTTP
Anthropic API
```

**이점**:
- Python 사용자 친화적
- Node.js CLI 재사용
- 각 언어의 장점 활용

---

### 2. **SDK MCP 서버 분리 처리**
```python
# Python에서:
mcp_servers = {
    "calc": {"type": "sdk", "instance": <Server>}
}

# CLI에 전달:
{"calc": {"type": "sdk"}}  # instance 제거

# 이유:
# - Python MCP Server는 Python 프로세스 내에서만 실행 가능
# - CLI는 Node.js → Python Server 호출 불가
# - Control Protocol로 SDK가 직접 처리
```

---

### 3. **버퍼 오버플로우 방지**
```python
buffer_size = 0
async for line in self._stdout_stream:
    buffer_size += len(line)
    if buffer_size > self._max_buffer_size:
        raise CLIConnectionError(f"Buffer exceeded {self._max_buffer_size}")
```

**용도**: DoS 공격 방지, 메모리 보호

---

### 4. **Stderr 백그라운드 처리**
```python
# 메인 로직과 독립적으로 실행
self._stderr_task_group.start_soon(self._handle_stderr)

# 에러 무시
except Exception:
    pass  # Stderr 에러가 메인 로직 방해 안 함
```

**설계 원칙**: **Fault Isolation**

---

### 5. **환경변수 우선순위**
```python
process_env = {
    **os.environ,           # 우선순위 3
    **self._options.env,    # 우선순위 2
    "CLAUDE_CODE_ENTRYPOINT": "sdk-py",  # 우선순위 1 (SDK 고정)
}
```

**이유**:
- 시스템: 기본값
- 사용자: 커스터마이징
- SDK: 필수값 (오버라이드 불가)

---

## 🚀 개선 제안

### 1. 타임아웃 추가
```python
async def connect(self, timeout: float = 30.0) -> None:
    with anyio.fail_after(timeout):
        self._process = await anyio.open_process(...)
```

### 2. 프로세스 재시작 (Retry)
```python
class SubprocessCLITransport(Transport):
    def __init__(self, ..., max_retries: int = 3):
        self._max_retries = max_retries

    async def connect(self) -> None:
        for attempt in range(self._max_retries):
            try:
                # ...
                break
            except CLIConnectionError as e:
                if attempt == self._max_retries - 1:
                    raise
                await anyio.sleep(1 * (attempt + 1))  # Exponential backoff
```

### 3. 헬스 체크
```python
async def is_alive(self) -> bool:
    """Check if subprocess is still running."""
    if not self._process:
        return False
    return self._process.returncode is None

async def healthcheck(self) -> None:
    """Send ping to CLI and expect pong."""
    await self.write('{"type": "ping"}\n')
    # expect pong within timeout
```

### 4. 메트릭 수집
```python
@dataclass
class TransportMetrics:
    bytes_sent: int = 0
    bytes_received: int = 0
    messages_sent: int = 0
    messages_received: int = 0
    errors: int = 0

class SubprocessCLITransport:
    def __init__(self, ...):
        self._metrics = TransportMetrics()

    async def write(self, data: str):
        self._metrics.bytes_sent += len(data)
        self._metrics.messages_sent += 1
        # ...
```

---

## 📊 복잡도 분석

### 라인 수 분포
- `_build_command()`: ~115줄 (23%)
- `connect()`: ~73줄 (15%)
- `close()`: ~42줄 (8%)
- `read()`: ~48줄 (10%)
- 기타: ~220줄 (44%)

### 복잡도 원인
1. **CLI 옵션 변환**: `_build_command()` 복잡
2. **에러 처리**: 다양한 에러 케이스
3. **스트림 관리**: stdin/stdout/stderr 동시 처리
4. **자원 정리**: 순서 있는 정리 필요

---

## 🎓 학습 포인트

### 1. 서브프로세스 관리
- `anyio.open_process()` 사용
- stdin/stdout/stderr 비동기 스트림
- 프로세스 종료 코드 체크

### 2. 에러 처리 계층
```
CLINotFoundError (CLI 바이너리 없음)
    ↓
CLIConnectionError (연결 실패)
    ↓
ProcessError (비정상 종료)
```

### 3. RAII 패턴
- `connect()` → 자원 획득
- `close()` → 자원 해제
- `suppress(Exception)` → 부분 실패 허용

### 4. Fault Isolation
- Stderr 에러 → 메인 로직 영향 없음
- 버퍼 오버플로우 → 조기 종료
- 프로세스 종료 → 에러 전파

---

**분석 작성**: Claude Code
**분석 프레임워크**: Operability, Simplicity, Evolvability
