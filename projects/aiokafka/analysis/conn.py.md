# conn.py - Kafka TCP 연결 관리

## 📋 파일 개요
- **경로**: `aiokafka/conn.py`
- **라인 수**: 919줄
- **주요 역할**: Kafka 브로커와의 비동기 TCP 연결 생성, 관리, SASL 인증 처리

## 🎯 핵심 목적
순수 Python asyncio 기반으로 Kafka 브로커와의 저수준 네트워크 연결을 관리하며, 프로토콜 버전 협상, SASL 인증, 요청/응답 매칭을 담당하는 **네트워크 계층의 핵심 모듈**

---

## 🏗️ 주요 클래스 및 구조

### 1. **CloseReason (IntEnum)**
```python
class CloseReason(IntEnum):
    CONNECTION_BROKEN = 0
    CONNECTION_TIMEOUT = 1
    OUT_OF_SYNC = 2
    IDLE_DROP = 3
    SHUTDOWN = 4
    AUTH_FAILURE = 5
```
- **목적**: 연결 종료 원인을 구분하여 디버깅 및 재연결 로직에 활용
- **특징**: IntEnum으로 명확한 종료 사유 추적 가능

---

### 2. **VersionInfo**
```python
class VersionInfo:
    def pick_best(self, request_versions):
        # API 버전 협상 로직
```
- **목적**: Kafka 브로커가 지원하는 API 버전과 클라이언트 요청을 매칭
- **주요 메서드**:
  - `pick_best()`: 브로커 지원 버전 범위 내에서 가장 높은 클라이언트 버전 선택
- **사용 시점**: `ApiVersionRequest` 응답 후 `_version_info`에 저장됨

---

### 3. **AIOKafkaConnection** ⭐ (메인 클래스)

#### 초기화 파라미터
```python
def __init__(
    self,
    host, port,
    client_id="aiokafka",
    request_timeout_ms=40000,
    api_version=(0, 8, 2),
    ssl_context=None,
    security_protocol="PLAINTEXT",  # PLAINTEXT, SSL, SASL_PLAINTEXT, SASL_SSL
    max_idle_ms=None,
    sasl_mechanism=None,  # PLAIN, GSSAPI, SCRAM-SHA-256/512, OAUTHBEARER
    ...
)
```

#### 핵심 속성
| 속성 | 타입 | 설명 |
|------|------|------|
| `_reader` | `asyncio.StreamReader` | 비동기 읽기 스트림 |
| `_writer` | `asyncio.StreamWriter` | 비동기 쓰기 스트림 |
| `_requests` | `deque` | `(correlation_id, request, future)` 튜플 큐 |
| `_correlation_id` | `int` | 요청/응답 매칭용 ID (0 ~ 2^31-1 순환) |
| `_version_info` | `VersionInfo` | 브로커 API 버전 정보 |
| `_read_task` | `asyncio.Task` | 백그라운드 응답 읽기 태스크 |
| `_idle_handle` | `asyncio.TimerHandle` | 유휴 연결 타임아웃 체커 |

#### 주요 메서드

**1) `connect()` - 연결 수립**
```python
async def connect(self):
    # 1. TCP 연결 생성 (SSL 옵션)
    transport, _ = await loop.create_connection(
        lambda: protocol, self.host, self.port, ssl=ssl
    )

    # 2. 백그라운드 read 태스크 시작
    self._read_task = self._create_reader_task()

    # 3. API 버전 협상 (Kafka 0.10+)
    if self._version_hint and self._version_hint >= (0, 10):
        await self._do_version_lookup()

    # 4. SASL 인증 (필요 시)
    if self._security_protocol in ["SASL_SSL", "SASL_PLAINTEXT"]:
        await self._do_sasl_handshake()
```
- **특징**:
  - `async_timeout.timeout()`으로 연결 타임아웃 보장
  - 버전 협상은 0.10 이상에서만 수행
  - 예외 발생 시 자동으로 `close()` 호출

**2) `send()` - 요청 전송**
```python
def send(self, request, expect_response=True):
    correlation_id = self._next_correlation_id()
    header = request.build_request_header(correlation_id, client_id)
    message = header.encode() + request.encode()
    size = struct.pack(">i", len(message))

    self._writer.write(size + message)  # Kafka wire protocol: [size(4 bytes)][message]

    if not expect_response:
        return self._writer.drain()

    fut = self._loop.create_future()
    self._requests.append((correlation_id, request, fut))
    return wait_for(fut, self._request_timeout)
```
- **프로토콜**: `[4-byte size][correlation_id][request]` 형식
- **응답 대기**: Future를 큐에 추가 → `_read()` 태스크가 매칭하여 resolve
- **타임아웃**: `request_timeout_ms` 내에 응답 없으면 `asyncio.TimeoutError`

**3) `_read()` - 백그라운드 응답 수신** (staticmethod + weakref)
```python
@staticmethod
async def _read(self_ref):
    while True:
        # 1. 응답 크기 읽기 (4 bytes)
        resp = await reader.readexactly(4)
        (size,) = struct.unpack(">i", resp)

        # 2. 응답 본문 읽기
        resp = await reader.readexactly(size)

        # 3. 응답 처리
        self._handle_frame(resp)
```
- **weakref 사용 이유**: 순환 참조 방지로 메모리 릭 방지
- **에러 핸들링**: `_on_read_task_error()` 콜백에서 연결 종료 처리
- **블로킹**: `readexactly()`로 정확한 바이트 수 대기

**4) `_handle_frame()` - 응답 파싱 및 매칭**
```python
def _handle_frame(self, resp):
    correlation_id, request, fut = self._requests[0]

    if correlation_id is None:  # SASL 토큰
        fut.set_result(resp)
    else:
        response_header = request.parse_response_header(resp)

        # Correlation ID 검증
        if response_header.correlation_id != correlation_id:
            fut.set_exception(Errors.CorrelationIdError(...))
            self.close(reason=CloseReason.OUT_OF_SYNC)
            return

        response = request.RESPONSE_TYPE.decode(resp)
        fut.set_result(response)

    self._requests.popleft()
```
- **FIFO 큐**: 요청 순서대로 응답 매칭 (Kafka는 순서 보장)
- **검증**: Correlation ID 불일치 시 연결 종료
- **특수 케이스**: Kafka 0.8.2의 GroupCoordinatorResponse 버그 우회

**5) `close()` - 연결 종료**
```python
def close(self, reason=None, exc=None):
    self._writer.close()
    self._read_task.cancel()

    # 대기 중인 모든 요청에 예외 전파
    for _, _, fut in self._requests:
        if not fut.done():
            fut.set_exception(KafkaConnectionError(...))

    # 콜백 호출 (클라이언트에 알림)
    if self._on_close_cb:
        self._on_close_cb(self, reason)
```
- **정리 순서**: Writer 종료 → Read 태스크 취소 → 대기 Future 실패 처리 → 콜백
- **`__del__`**: 리소스 릭 방지용 경고 및 강제 종료

**6) `_idle_check()` - 유휴 연결 관리**
```python
@staticmethod
def _idle_check(self_ref):
    idle_for = time.monotonic() - self._last_action
    if (idle_for >= timeout) and not self._requests:
        self.close(CloseReason.IDLE_DROP)
    else:
        # 다음 체크 예약
        self._idle_handle = self._loop.call_later(wake_up_in, self._idle_check, self_ref)
```
- **트리거**: `max_idle_ms` 파라미터 설정 시 활성화
- **조건**: 요청이 없고 마지막 액션 후 일정 시간 경과
- **재귀 스케줄링**: `call_later()`로 타이머 체인

---

### 4. **SASL 인증 메커니즘**

#### 공통 베이스: `BaseSaslAuthenticator`
```python
class BaseSaslAuthenticator:
    def step(self, payload):
        return self._loop.run_in_executor(None, self._step, payload)

    def _step(self, payload):
        data = self._authenticator.send(payload)  # Generator 기반
        return data or None
```
- **패턴**: Generator를 Executor에서 실행 (블로킹 암호화 연산 대비)

#### 지원 메커니즘

**1) PLAIN** (`SaslPlainAuthenticator`)
```python
def authenticator_plain(self):
    data = "\0".join([username, username, password]).encode("utf-8")
    resp = yield data, True
    assert resp == b""
```
- **RFC-4616**: `\0username\0username\0password` 형식
- **보안 경고**: SASL_PLAINTEXT 사용 시 경고 로그

**2) GSSAPI/Kerberos** (`SaslGSSAPIAuthenticator`)
```python
def authenticator_gssapi(self):
    client_ctx = gssapi.SecurityContext(name=cname, usage="initiate")
    while not client_ctx.complete:
        client_token = client_ctx.step(server_token)
        server_token = yield client_token, True
    # QOP 처리...
```
- **라이브러리**: `gssapi` (선택적 의존성)
- **프로토콜**: GSSAPI 컨텍스트 완료까지 다중 왕복

**3) SCRAM-SHA-256/512** (`ScramAuthenticator`)
```python
def authenticator_scram(self):
    client_first = self.first_message().encode("utf-8")  # n=user,r=nonce
    server_first = yield client_first, True
    self.process_server_first_message(server_first)  # Salt, iterations
    client_final = self.final_message().encode("utf-8")  # c=biws,r=nonce,p=proof
    server_final = yield client_final, True
    self.process_server_final_message(server_final)  # 서명 검증
```
- **RFC-5802**: Challenge-response 기반
- **보안**: PBKDF2로 salted password 생성 → HMAC으로 proof 계산

**4) OAUTHBEARER** (`OAuthAuthenticator`)
```python
async def step(self, payload):
    token = await self._sasl_oauth_token_provider.token()
    return (self._build_oauth_client_request(token, extensions).encode("utf-8"), True)
```
- **비동기**: 토큰 제공자가 async 메서드
- **확장**: 선택적 key-value 확장 지원

---

### 5. **유틸리티 함수**

**`create_conn()`** - 팩토리 함수
```python
async def create_conn(host, port, **kwargs):
    conn = AIOKafkaConnection(host, port, **kwargs)
    await conn.connect()
    return conn
```

**`get_ip_port_afi()`** - 호스트 파싱
```python
# 지원 형식:
# - "host"              → (host, 9092, AF_UNSPEC)
# - "host:9092"         → (host, 9092, AF_INET)
# - "[::1]"             → ("::1", 9092, AF_INET6)
# - "[::1]:9092"        → ("::1", 9092, AF_INET6)
```

**`collect_hosts()`** - 부트스트랩 서버 파싱
```python
collect_hosts("host1:9092,host2:9093", randomize=True)
# → [(host, port, afi), ...] (셔플됨)
```

---

## 🔗 다른 모듈과의 관계

### 의존성 (Imports)
```
conn.py
├── aiokafka.protocol (요청/응답 클래스)
│   ├── ApiVersionRequest
│   ├── SaslHandShakeRequest
│   └── SaslAuthenticateRequest
├── aiokafka.errors (예외)
├── aiokafka.abc (AbstractTokenProvider)
└── aiokafka.util (asyncio 헬퍼)
```

### 사용처
- `client.py`: `AIOKafkaConnection`을 래핑하여 재연결 로직 추가
- `producer.py`, `consumer.py`: `client.py`를 통해 간접 사용

---

## ⚙️ 핵심 설계 패턴

### 1. **Correlation ID 기반 요청/응답 매칭**
```
Client                          Broker
  |-- send(req1, corr_id=1) -->|
  |-- send(req2, corr_id=2) -->|
  |<-- recv(corr_id=1) --------|
  |<-- recv(corr_id=2) --------|
```
- **FIFO 보장**: Kafka는 요청 순서대로 응답
- **검증**: 불일치 시 `OUT_OF_SYNC`로 연결 종료

### 2. **Weakref로 순환 참조 방지**
```python
self_ref = weakref.ref(self)
read_task = create_task(self._read(self_ref))
```
- **문제**: 태스크가 self를 참조하면 GC 불가
- **해결**: weakref로 참조, 매 루프마다 유효성 체크

### 3. **Future 기반 비동기 응답**
```python
fut = self._loop.create_future()
self._requests.append((correlation_id, request, fut))
return wait_for(fut, timeout)  # 호출자는 await
```
- **장점**: 동시 다중 요청 가능
- **타임아웃**: `wait_for()`로 자동 취소

### 4. **콜백 기반 연결 종료 알림**
```python
# 생성 시
conn = AIOKafkaConnection(..., on_close=my_callback)

# 종료 시
def close(self, reason):
    if self._on_close_cb:
        self._on_close_cb(self, reason)
```
- **용도**: 상위 레이어(client.py)에서 재연결 트리거

---

## 📊 실행 흐름 예시

### 정상 요청/응답
```
1. client.send(MetadataRequest)
   ↓
2. AIOKafkaConnection.send()
   - correlation_id = 1
   - self._requests.append((1, req, fut))
   - self._writer.write(b'\x00\x00\x00\x1a\x00\x00\x00\x01...')
   ↓
3. 백그라운드 _read() 태스크
   - resp = await reader.readexactly(4)  # size
   - resp = await reader.readexactly(size)  # body
   ↓
4. _handle_frame(resp)
   - correlation_id, request, fut = self._requests[0]
   - response = request.RESPONSE_TYPE.decode(resp)
   - fut.set_result(response)  ← await가 완료됨
   - self._requests.popleft()
```

### SASL 인증 흐름 (SCRAM-SHA-256)
```
1. connect()
   ↓
2. _do_sasl_handshake()
   ↓
3. SaslHandShakeRequest → 브로커가 SCRAM-SHA-256 지원 확인
   ↓
4. ScramAuthenticator.step(None)
   - client_first = "n=user,r=nonce"
   - yield (client_first, True)
   ↓
5. 브로커 응답: "r=nonce+server,s=salt,i=4096"
   ↓
6. ScramAuthenticator.step(server_first)
   - PBKDF2로 salted_password 계산
   - client_proof = XOR(client_key, client_signature)
   - client_final = "c=biws,r=nonce+server,p=proof"
   - yield (client_final, True)
   ↓
7. 브로커 응답: "v=server_signature"
   ↓
8. 서명 검증 → 인증 완료
```

---

## 🚨 에러 처리

### 예외 타입
| 예외 | 발생 상황 |
|------|-----------|
| `KafkaConnectionError` | 연결 실패, 송신 실패 |
| `CorrelationIdError` | 응답 ID 불일치 |
| `UnsupportedSaslMechanismError` | 브로커가 SASL 메커니즘 미지원 |
| `asyncio.TimeoutError` | `request_timeout_ms` 초과 |

### 연결 종료 처리
```python
def _on_read_task_error(cls, self_ref, read_task):
    try:
        read_task.result()
    except Exception as exc:
        if not isinstance(exc, (OSError, EOFError, ConnectionError)):
            log.exception("Unexpected exception")

        self = self_ref()
        if self is not None:
            self.close(reason=CloseReason.CONNECTION_BROKEN, exc=exc)
```
- **자동 종료**: 읽기 태스크 에러 → 연결 종료 → 콜백 호출

---

## 🔑 핵심 특징 요약

| 특징 | 설명 |
|------|------|
| **순수 asyncio** | librdkafka 없이 Python으로 TCP/TLS 구현 |
| **프로토콜 지원** | Kafka 0.8.2 ~ 최신 버전 (버전 협상) |
| **보안** | SSL/TLS, SASL (PLAIN, GSSAPI, SCRAM, OAuth) |
| **성능** | 백그라운드 읽기, 파이프라인 요청 |
| **안정성** | 타임아웃, 유휴 연결 정리, weakref로 메모리 릭 방지 |
| **디버깅** | CloseReason으로 종료 원인 추적 |

---

## 🎓 결과적으로 이 파일은

**Kafka 클라이언트의 네트워크 기반 레이어**로서:
1. ✅ Kafka 브로커와의 **순수 Python 비동기 TCP 연결** 제공
2. ✅ **API 버전 협상**으로 다양한 Kafka 버전 지원
3. ✅ **SASL/SSL 인증**으로 프로덕션 환경 보안 요구사항 충족
4. ✅ **Correlation ID 매칭**으로 요청/응답 무결성 보장
5. ✅ **Weakref 패턴**으로 메모리 안전성 확보
6. ✅ **콜백 기반 이벤트**로 상위 레이어와 느슨한 결합 유지

→ `client.py`, `producer.py`, `consumer.py`가 의존하는 **핵심 네트워크 추상화 계층**
