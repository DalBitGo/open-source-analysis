# conn.py - Kafka 연결 관리

> **분석 일자**: 2025-10-29
> **파일**: `aiokafka/conn.py` (918 lines)
> **방법론**: 문제 해결 중심 분석 (파일별)

---

## 📋 파일 개요

### 파일 정보
- **경로**: `aiokafka/conn.py`
- **줄 수**: 918 lines
- **주요 클래스**: `AIOKafkaConnection`, 5개 SASL Authenticator

### 핵심 역할

이 파일은 **Kafka 브로커와의 개별 연결**을 관리합니다:
- TCP 연결 생성 및 유지
- SASL 인증 (5가지 방식)
- 비동기 요청/응답 처리
- Idle 연결 관리
- API 버전 협상

**누가 사용하는가?**
- `AIOKafkaClient` (client.py)가 Connection Pool로 관리
- Producer/Consumer는 간접적으로 사용

---

## 해결하는 핵심 문제들

이 파일은 **5가지 주요 문제**를 해결합니다. 각각을 "문제 → 고민 → 해결" 관점으로 분석합니다.

---

## 1. TCP 연결 생성과 비동기 I/O

### 문제

Kafka 브로커와 통신하려면 **TCP 소켓 연결**이 필요합니다:

1. **연결 생성 단계가 복잡**
   - TCP 3-way handshake
   - SSL/TLS handshake (보안 모드)
   - SASL 인증 (인증 필요 시)

2. **비동기 환경 (asyncio)**
   - 연결 생성 중 다른 작업 차단하면 안 됨
   - 읽기/쓰기가 모두 비동기여야 함

3. **연결 실패 처리**
   - 네트워크 오류, 타임아웃, DNS 실패 등
   - 사용자에게 명확한 에러 전달

### 고민했던 선택지

#### 선택지 1: 동기 소켓 (socket 모듈)

```python
# 간단하지만 블로킹
sock = socket.socket()
sock.connect((host, port))
sock.send(data)
```

**장점**: 간단, 익숙함
**단점**: 블로킹 → asyncio 환경에서 전체 이벤트 루프 차단
**왜 안 됨**: Kafka 클라이언트는 고성능 비동기 라이브러리 목표

#### 선택지 2: asyncio.open_connection()

```python
# asyncio 기본 API
reader, writer = await asyncio.open_connection(host, port)
await writer.write(data)
response = await reader.read(1024)
```

**장점**: 비동기, 간단
**단점**:
- StreamReader/Writer가 제공하는 기능이 제한적
- SSL, SASL 같은 고급 기능 통합 어려움
- 연결 상태 추적, 타임아웃 관리 직접 구현 필요

**왜 선택하지 않았는가**: 커스터마이징 필요성이 큼

#### 선택지 3 (최종): asyncio 저수준 API + 커스텀 Protocol

```python
# loop.create_connection() + 커스텀 프로토콜
reader = asyncio.StreamReader(limit=READER_LIMIT)
protocol = AIOKafkaProtocol(closed_fut, reader, loop=loop)

transport, _ = await loop.create_connection(
    lambda: protocol,
    host, port,
    ssl=ssl_context
)

writer = asyncio.StreamWriter(transport, protocol, reader, loop)
```

**장점**:
- ✅ 완전한 비동기
- ✅ SSL, SASL 통합 가능
- ✅ 연결 종료 감지 (connection_lost 콜백)
- ✅ 버퍼 크기 제어 (READER_LIMIT = 64KB)

**단점**:
- ❌ 코드 복잡도 증가
- ❌ asyncio 내부 API 이해 필요

**왜 선택했는가**:
- Kafka는 대용량 메시지 처리 → 버퍼 크기 제어 필수
- 연결 상태 추적 → Protocol의 connection_lost() 활용
- SSL/SASL 통합이 간단

### 최종 해결책

#### 구조

```python
# aiokafka/conn.py

class AIOKafkaProtocol(asyncio.StreamReaderProtocol):
    """연결 종료 감지를 위한 커스텀 프로토콜"""

    def __init__(self, closed_fut, *args, **kw):
        self._closed_fut = closed_fut
        super().__init__(*args, **kw)

    def connection_lost(self, exc):
        """Transport 종료 시 자동 호출"""
        super().connection_lost(exc)
        if not self._closed_fut.cancelled():
            self._closed_fut.set_result(None)


class AIOKafkaConnection:
    async def connect(self):
        """연결 생성"""
        loop = self._loop
        self._closed_fut = create_future()

        # SSL 컨텍스트 결정
        if self._security_protocol in ["PLAINTEXT", "SASL_PLAINTEXT"]:
            ssl = None
        else:
            ssl = self._ssl_context

        # 저수준 API로 연결 생성
        reader = asyncio.StreamReader(limit=READER_LIMIT, loop=loop)
        protocol = AIOKafkaProtocol(self._closed_fut, reader, loop=loop)

        async with async_timeout.timeout(self._request_timeout):
            transport, _ = await loop.create_connection(
                lambda: protocol,
                self.host, self.port,
                ssl=ssl
            )

        writer = asyncio.StreamWriter(transport, protocol, reader, loop)
        self._reader, self._writer, self._protocol = reader, writer, protocol

        # 백그라운드 읽기 Task 시작
        self._read_task = self._create_reader_task()

        # Idle 타이머 시작
        if self._max_idle_ms is not None:
            self._idle_handle = loop.call_soon(
                self._idle_check,
                weakref.ref(self)
            )

        # SASL 인증 (필요 시)
        if self._security_protocol in ["SASL_SSL", "SASL_PLAINTEXT"]:
            await self._do_sasl_handshake()

        return reader, writer
```

#### 핵심 아이디어

**1. 버퍼 크기 제어**

```python
READER_LIMIT = 2**16  # 64KB

reader = asyncio.StreamReader(limit=READER_LIMIT)
```

**왜 64KB?**
- Kafka 메시지는 KB~MB 크기 가능
- 기본 버퍼(8KB)로는 부족
- 너무 크면 메모리 낭비
- 64KB = 적절한 균형점

**2. 연결 종료 감지**

```python
class AIOKafkaProtocol(asyncio.StreamReaderProtocol):
    def connection_lost(self, exc):
        # 자동으로 호출됨 (TCP FIN/RST 수신 시)
        self._closed_fut.set_result(None)
```

**왜 필요한가?**
- 브로커가 갑자기 종료될 수 있음
- 네트워크 장애 발생 가능
- 조기 감지 → 재연결 빠름

**3. 타임아웃 설정**

```python
async with async_timeout.timeout(self._request_timeout):
    transport, _ = await loop.create_connection(...)
```

**트레이드오프**:
- 짧은 timeout: 실패 빠르게 감지, 불안정한 네트워크에서 문제
- 긴 timeout: 안정적이지만 장애 감지 느림
- 기본값 40초: 대부분의 환경에서 적절

---

## 2. SASL 인증 - 5가지 방식 지원

### 문제

Kafka는 **여러 인증 방식**을 지원합니다:

1. **환경마다 요구사항이 다름**
   - 개발: 인증 없음 (PLAINTEXT)
   - 스테이징: PLAIN (간단한 username/password)
   - 프로덕션: Kerberos (GSSAPI) 또는 OAuth

2. **각 인증 방식의 프로토콜이 다름**
   - PLAIN: 한 번의 username/password 전송
   - GSSAPI: 다단계 challenge-response
   - SCRAM: SHA-256/512 해시 기반
   - OAUTHBEARER: JWT 토큰 기반

3. **하위 호환성 유지**
   - Kafka 0.9: 다른 프로토콜 (GSSAPI만)
   - Kafka 0.10+: SaslHandshakeRequest
   - Kafka 1.0+: SaslAuthenticateRequest

### 고민했던 선택지

#### 선택지 1: 하나의 인증 방식만 지원

```python
# PLAIN만 구현
async def authenticate(self):
    auth_str = f"\0{username}\0{password}"
    await self._writer.write(auth_str.encode())
```

**장점**: 간단
**단점**: 엔터프라이즈 환경 지원 불가 (Kerberos 필수)
**왜 안 됨**: Kafka 공식 클라이언트는 모든 SASL 방식 지원 필요

#### 선택지 2: if-else로 분기

```python
async def authenticate(self):
    if mechanism == "PLAIN":
        # PLAIN 로직
    elif mechanism == "GSSAPI":
        # GSSAPI 로직
    elif mechanism == "SCRAM-SHA-256":
        # SCRAM 로직
    # ...
```

**장점**: 간단, 한 곳에 모든 로직
**단점**:
- 메서드가 수백 줄로 증가
- 테스트 어려움
- 새 방식 추가 시 기존 코드 수정

**왜 안 됨**: 단일 책임 원칙 위배, 유지보수 어려움

#### 선택지 3 (최종): 플러그인 아키텍처

```python
# 추상 기본 클래스
class BaseSaslAuthenticator:
    async def step(self, auth_bytes):
        """인증 단계 수행. None 반환 시 완료"""
        raise NotImplementedError

# 각 방식별 구현체
class SaslPlainAuthenticator(BaseSaslAuthenticator): ...
class SaslGSSAPIAuthenticator(BaseSaslAuthenticator): ...
class ScramAuthenticator(BaseSaslAuthenticator): ...
class OAuthAuthenticator(BaseSaslAuthenticator): ...

# 사용
if self._sasl_mechanism == "GSSAPI":
    authenticator = self.authenticator_gssapi()
elif self._sasl_mechanism.startswith("SCRAM"):
    authenticator = self.authenticator_scram()
# ...

while True:
    res = await authenticator.step(auth_bytes)
    if res is None:
        break
    # ...
```

**장점**:
- ✅ 각 방식이 독립적인 클래스
- ✅ 새 방식 추가 쉬움 (기존 코드 변경 없음)
- ✅ 테스트 용이 (각 Authenticator 독립 테스트)
- ✅ 확장 가능 (사용자 정의 인증 방식 가능)

**단점**:
- ❌ 클래스 개수 증가 (5개 → 코드 분산)
- ❌ 추상화 오버헤드

**왜 선택했는가**:
- 엔터프라이즈 환경 지원 필수
- 새 인증 방식 계속 추가될 가능성
- 각 방식이 복잡 → 분리 필수

### 최종 해결책

#### 구조

```python
# 추상 기본 클래스
class BaseSaslAuthenticator:
    """모든 SASL 인증의 기본 클래스"""

    def __init__(self, loop=None):
        self._loop = loop or get_running_loop()

    async def step(self, auth_bytes):
        """
        인증 단계 수행

        Returns:
            None: 인증 완료
            (payload, expect_response): 다음 단계 계속
        """
        raise NotImplementedError


# PLAIN 인증 (가장 간단)
class SaslPlainAuthenticator(BaseSaslAuthenticator):
    """username/password를 한 번에 전송"""

    def __init__(self, *, sasl_plain_username, sasl_plain_password, **kw):
        super().__init__(**kw)
        self._username = sasl_plain_username
        self._password = sasl_plain_password

    async def step(self, auth_bytes):
        if auth_bytes is None:
            # 첫 단계: credentials 전송
            msg = f"\0{self._username}\0{self._password}".encode()
            return (msg, True)
        # 두 번째 단계: 완료
        return None


# GSSAPI (Kerberos) - 복잡함
class SaslGSSAPIAuthenticator(BaseSaslAuthenticator):
    """Kerberos 기반 인증 (다단계)"""

    def __init__(self, *, principal, **kw):
        super().__init__(**kw)
        self._principal = principal
        self._client = None  # gssapi.SecurityContext

    async def step(self, auth_bytes):
        if self._client is None:
            # 첫 단계: SecurityContext 생성
            service = gssapi.Name(
                self._principal,
                name_type=gssapi.NameType.hostbased_service
            )
            self._client = gssapi.SecurityContext(
                name=service,
                usage="initiate"
            )

        # Challenge-response 루프
        token = self._client.step(auth_bytes)

        if self._client.complete:
            return None  # 인증 완료
        else:
            return (token, True)  # 다음 단계 계속


# SCRAM (SHA-256/512)
class ScramAuthenticator(BaseSaslAuthenticator):
    """HMAC 기반 Challenge-Response"""

    def __init__(self, *, sasl_plain_username, sasl_plain_password,
                 sasl_mechanism, **kw):
        super().__init__(**kw)
        self._username = sasl_plain_username
        self._password = sasl_plain_password
        # SHA-256 or SHA-512
        self._hashfunc = (hashlib.sha256 if "256" in sasl_mechanism
                          else hashlib.sha512)
        self._nonce = str(uuid.uuid4()).replace("-", "")

    async def step(self, auth_bytes):
        # 3단계 프로토콜 구현
        # 1. Client-first message
        # 2. Server challenge 처리
        # 3. Client proof 전송
        # (복잡한 HMAC 계산 생략)
        ...


# OAuth Bearer
class OAuthAuthenticator(BaseSaslAuthenticator):
    """JWT 토큰 기반 인증"""

    def __init__(self, *, sasl_oauth_token_provider, **kw):
        super().__init__(**kw)
        self._token_provider = sasl_oauth_token_provider

    async def step(self, auth_bytes):
        if auth_bytes is None:
            # 토큰 획득
            token = await self._token_provider.token()
            msg = f"auth=Bearer {token}\x01\x01".encode()
            return (msg, True)
        return None
```

#### 사용 (AIOKafkaConnection에서)

```python
async def _do_sasl_handshake(self):
    # 1. Handshake (Kafka 0.10+)
    sasl_handshake = SaslHandShakeRequest(self._sasl_mechanism)
    response = await self.send(sasl_handshake)

    # 브로커가 지원하는지 확인
    if self._sasl_mechanism not in response.enabled_mechanisms:
        raise UnsupportedSaslMechanismError(...)

    # 2. Authenticator 선택
    if self._sasl_mechanism == "GSSAPI":
        authenticator = self.authenticator_gssapi()
    elif self._sasl_mechanism.startswith("SCRAM"):
        authenticator = self.authenticator_scram()
    elif self._sasl_mechanism == "OAUTHBEARER":
        authenticator = self.authenticator_oauth()
    else:
        authenticator = self.authenticator_plain()

    # 3. 인증 루프 (다단계 지원)
    auth_bytes = None
    while True:
        res = await authenticator.step(auth_bytes)
        if res is None:
            break  # 인증 완료
        payload, expect_response = res

        # 요청 전송
        req = SaslAuthenticateRequest(payload)
        resp = await self.send(req)

        # 에러 체크
        if resp.error_code != 0:
            raise AuthenticationFailedError(...)

        auth_bytes = resp.sasl_auth_bytes

    log.info("Authenticated via %s", self._sasl_mechanism)
```

#### 핵심 아이디어

**1. 플러그인 패턴**

새 인증 방식 추가 시:
```python
# 새 파일: my_auth.py
class MyCustomAuthenticator(BaseSaslAuthenticator):
    async def step(self, auth_bytes):
        # 커스텀 로직
        ...

# 사용
authenticator = MyCustomAuthenticator()
```

기존 코드 변경 없음!

**2. step() 인터페이스**

```python
async def step(self, auth_bytes):
    """
    Returns:
        None: 인증 완료
        (payload, expect_response): 계속
    """
```

**왜 이렇게?**
- 간단한 인증 (PLAIN): 1~2번 호출로 완료
- 복잡한 인증 (GSSAPI): 여러 번 challenge-response
- 동일한 인터페이스로 모든 방식 처리

**3. 하위 호환성**

```python
# Kafka 0.9: GSSAPI만, handshake 없음
if self._version_hint and self._version_hint < (0, 10):
    handshake_klass = None
else:
    # Kafka 0.10+: handshake 필요
    handshake_klass = SaslHandShakeRequest
```

---

## 3. 비동기 요청/응답 처리 - Correlation ID

### 문제

Kafka 프로토콜은 **TCP 스트림**을 사용합니다:

1. **여러 요청을 동시에 보냄**
   - Producer: 여러 파티션에 동시 전송
   - Consumer: 메타데이터 + fetch 동시 요청
   - 비동기 환경: 응답 순서 보장 안 됨

2. **응답 매칭 문제**
   ```
   요청: [A, B, C]
   응답: [B, A, C]  # 순서 다름!
   ```
   어떤 응답이 어떤 요청에 대한 것인지?

3. **에러 처리**
   - 요청 전송 후 연결 끊김
   - 타임아웃
   - 응답 손실

### 고민했던 선택지

#### 선택지 1: 동기식 요청/응답

```python
# 한 번에 하나씩
async def send(self, request):
    await self._writer.write(request)
    response = await self._reader.read()
    return response
```

**장점**: 간단, 매칭 불필요
**단점**: 성능 낮음 (동시 요청 불가)
**왜 안 됨**: Kafka는 고성능 요구

#### 선택지 2: Queue로 순서 보장

```python
# 요청 순서대로 응답 대기
self._requests = asyncio.Queue()

async def send(self, request):
    fut = asyncio.Future()
    await self._requests.put(fut)
    await self._writer.write(request)
    return await fut

async def _read_loop(self):
    while True:
        response = await self._reader.read()
        fut = await self._requests.get()  # 순서대로!
        fut.set_result(response)
```

**장점**: 구현 간단
**단점**: 응답이 순서대로 와야 함 (Kafka는 보장 안 함)
**왜 안 됨**: Kafka 프로토콜과 맞지 않음

#### 선택지 3 (최종): Correlation ID

```python
# 각 요청에 고유 ID 부여
self._correlation_id = 0
self._requests = collections.deque()  # [(id, req, fut), ...]

def send(self, request):
    # 고유 ID 생성
    correlation_id = self._next_correlation_id()

    # 요청에 ID 삽입
    header = request.build_request_header(
        correlation_id=correlation_id,
        client_id=self._client_id
    )
    message = header.encode() + request.encode()

    # 전송
    self._writer.write(message)

    # Future 저장 (매칭용)
    fut = self._loop.create_future()
    self._requests.append((correlation_id, request, fut))

    return fut  # 나중에 resolve됨

async def _read_loop(self):
    while True:
        # 응답 읽기
        response = await self._reader.read()
        correlation_id = response.header.correlation_id

        # 매칭 (deque에서 찾기)
        for i, (req_id, req, fut) in enumerate(self._requests):
            if req_id == correlation_id:
                fut.set_result(response)
                del self._requests[i]
                break
```

**장점**:
- ✅ 응답 순서 무관
- ✅ 동시 요청 가능
- ✅ Kafka 프로토콜 표준

**단점**:
- ❌ 구현 복잡도
- ❌ Correlation ID 관리 필요

**왜 선택했는가**: Kafka 프로토콜이 이미 이 방식 사용

### 최종 해결책

#### 구조

```python
class AIOKafkaConnection:
    def __init__(self, ...):
        self._correlation_id = 0
        self._requests = collections.deque()  # FIFO queue

    def send(self, request, expect_response=True):
        """요청 전송"""
        # 1. Correlation ID 생성
        correlation_id = self._next_correlation_id()

        # 2. 요청 인코딩 (헤더 + 바디)
        header = request.build_request_header(
            correlation_id=correlation_id,
            client_id=self._client_id
        )
        message = header.encode() + request.encode()
        size = struct.pack(">i", len(message))

        # 3. 전송
        self._writer.write(size + message)

        # 4. 응답 불필요한 경우 (Produce with acks=0)
        if not expect_response:
            return self._writer.drain()

        # 5. Future 생성 및 저장
        fut = self._loop.create_future()
        self._requests.append(
            (correlation_id, request, fut)
        )

        # 6. Timeout 설정
        return wait_for(fut, self._request_timeout)

    def _next_correlation_id(self):
        """순환 증가 (0 ~ 2^31-1)"""
        self._correlation_id = (self._correlation_id + 1) % 2**31
        return self._correlation_id

    # 백그라운드에서 실행
    @staticmethod
    async def _read(self_ref):
        """응답 읽기 루프"""
        self = self_ref()
        reader = self._reader

        while True:
            # 1. 응답 크기 읽기 (4 bytes)
            resp = await reader.readexactly(4)
            (size,) = struct.unpack(">i", resp)

            # 2. 응답 바디 읽기
            resp = await reader.readexactly(size)

            # 3. 처리
            self = self_ref()
            if self is not None:
                self._handle_frame(resp)

    def _handle_frame(self, resp):
        """응답 처리"""
        # 1. 첫 번째 요청 가져오기 (FIFO)
        correlation_id, request, fut = self._requests[0]

        # 2. 응답 파싱
        resp = io.BytesIO(resp)
        response_header = request.parse_response_header(resp)

        # 3. Correlation ID 검증
        if response_header.correlation_id != correlation_id:
            # 순서 틀림! 연결 종료
            error = CorrelationIdError(
                f"Expected {correlation_id}, got {response_header.correlation_id}"
            )
            fut.set_exception(error)
            self.close(reason=CloseReason.OUT_OF_SYNC)
            return

        # 4. 응답 디코딩
        resp_type = request.RESPONSE_TYPE
        response = resp_type.decode(resp)

        # 5. Future resolve
        fut.set_result(response)

        # 6. Idle timer 갱신
        self._last_action = time.monotonic()

        # 7. 요청 제거 (FIFO)
        self._requests.popleft()
```

#### 핵심 아이디어

**1. Correlation ID = 순환 카운터**

```python
self._correlation_id = (self._correlation_id + 1) % 2**31

# 0, 1, 2, ..., 2147483646, 0, 1, ...
```

**왜 순환?**
- 무한 증가 시 overflow
- 2^31 = 21억: 현실적으로 충돌 없음

**2. deque = FIFO 큐**

```python
self._requests = collections.deque()

# 추가: O(1)
self._requests.append((id, req, fut))

# 제거: O(1)
self._requests.popleft()
```

**왜 deque?**
- 응답은 대부분 요청 순서대로 도착
- 첫 번째만 확인: O(1)
- list보다 빠름

**3. 순서 검증**

```python
if response_header.correlation_id != correlation_id:
    # 순서 틀림 → 연결 손상
    self.close(reason=CloseReason.OUT_OF_SYNC)
```

**왜 필요한가?**
- 네트워크 오류로 패킷 손실 가능
- 브로커 버그 가능성
- 조기 감지 → 재연결

**4. weakref로 메모리 누수 방지**

```python
# _read는 백그라운드 Task
@staticmethod
async def _read(self_ref):  # weakref!
    self = self_ref()
    if self is None:
        return  # 이미 해제됨
```

**문제**: Task가 강한 참조 → 순환 참조
```
Connection → Task → Connection (누수!)
```

**해결**: weakref 사용
```
Connection → Task → weakref(Connection) (OK!)
```

---

## 4. Idle 연결 관리

### 문제

TCP 연결을 무한정 유지하면:

1. **리소스 낭비**
   - 소켓 파일 디스크립터
   - 메모리 (버퍼)
   - 브로커 리소스

2. **방화벽 타임아웃**
   - 많은 방화벽이 idle 연결 자동 종료 (10~30분)
   - 종료된 연결 사용 시 → 에러

3. **좀비 연결**
   - 브로커는 종료했지만 클라이언트는 모름
   - 다음 요청 시에야 실패 감지

### 고민했던 선택지

#### 선택지 1: 연결 계속 유지

```python
# 아무것도 안 함
conn = await create_conn()
# 계속 사용
```

**장점**: 간단
**단점**: 위 문제들 발생
**왜 안 됨**: 장기 실행 프로세스에서 문제

#### 선택지 2: 주기적으로 ping

```python
# 30초마다 Heartbeat
asyncio.create_task(self._heartbeat_loop())

async def _heartbeat_loop(self):
    while True:
        await asyncio.sleep(30)
        await self.send(HeartbeatRequest())
```

**장점**: 연결 활성 유지
**단점**:
- 불필요한 네트워크 트래픽
- 브로커 부하
- 실제로 사용 안 하는 연결도 유지

**왜 안 됨**: 오버헤드

#### 선택지 3 (최종): Idle timeout + 자동 재연결

```python
# 일정 시간 사용 안 하면 종료
if idle_for >= timeout:
    self.close(reason=CloseReason.IDLE_DROP)

# 다음 사용 시 자동 재연결 (client.py에서)
conn = await client._get_conn(node_id)  # 자동 생성
```

**장점**:
- ✅ 리소스 절약
- ✅ 네트워크 트래픽 없음
- ✅ 필요할 때만 재연결

**단점**:
- ❌ 재연결 비용 (첫 요청 느림)

**왜 선택했는가**: 대부분의 연결은 버스트 트래픽 → idle이 길다

### 최종 해결책

#### 구조

```python
class AIOKafkaConnection:
    def __init__(self, ..., max_idle_ms=540000):  # 9분
        self._max_idle_ms = max_idle_ms
        self._last_action = time.monotonic()
        self._idle_handle = None

    async def connect(self):
        # ...연결 생성...

        # Idle checker 시작
        if self._max_idle_ms is not None:
            self._idle_handle = self._loop.call_soon(
                self._idle_check,
                weakref.ref(self)  # 메모리 누수 방지
            )

    @staticmethod
    def _idle_check(self_ref):
        """주기적으로 호출됨 (weakref 사용)"""
        self = self_ref()
        if self is None:
            return  # 이미 해제됨

        # 1. Idle 시간 계산
        idle_for = time.monotonic() - self._last_action
        timeout = self._max_idle_ms / 1000

        # 2. Pending 요청이 있으면 idle 아님
        if (idle_for >= timeout) and not self._requests:
            # Idle timeout → 연결 종료
            self.close(CloseReason.IDLE_DROP)
        else:
            # 3. 다시 스케줄링
            if self._requests:
                # Pending 요청 있으면 timeout만큼 대기
                wake_up_in = timeout
            else:
                # 남은 시간만 대기
                wake_up_in = timeout - idle_for

            self._idle_handle = self._loop.call_later(
                wake_up_in,
                self._idle_check,
                self_ref
            )

    def _handle_frame(self, resp):
        """응답 처리 시 마다 idle timer 갱신"""
        # ...응답 처리...

        # Idle timer 갱신
        self._last_action = time.monotonic()
```

#### 핵심 아이디어

**1. Lazy 스케줄링**

```python
if self._requests:
    wake_up_in = timeout  # 전체 대기
else:
    wake_up_in = timeout - idle_for  # 남은 시간만
```

**왜?**
- Pending 요청 있으면: 응답 올 때까지 기다림
- 없으면: 정확한 timeout 계산

**2. weakref로 메모리 누수 방지**

```python
self._idle_handle = self._loop.call_later(
    wake_up_in,
    self._idle_check,
    weakref.ref(self)  # 중요!
)
```

**문제**: call_later가 강한 참조 유지
```
Connection → idle_check → Connection (순환!)
```

**해결**: weakref
```
Connection → idle_check → weakref(Connection) (OK!)
```

Connection이 해제되면 idle_check도 자동 정리됨

**3. 기본값 540초 (9분)**

```python
connections_max_idle_ms=540000  # 9분
```

**왜 9분?**
- Kafka 브로커 기본값: `connections.max.idle.ms=600000` (10분)
- 클라이언트가 먼저 종료 → 깨끗한 종료
- 너무 짧으면: 재연결 빈번
- 너무 길면: 리소스 낭비

---

## 5. API Version Negotiation

### 문제

Kafka는 **버전마다 프로토콜이 다릅니다**:

1. **클라이언트와 브로커 버전 불일치**
   - 클라이언트: 최신 (Kafka 3.0)
   - 브로커: 구버전 (Kafka 0.10)
   - 최신 API 사용 시 → 에러

2. **각 API마다 여러 버전**
   - ProduceRequest: v0 ~ v9
   - FetchRequest: v0 ~ v13
   - 어떤 버전 사용할지?

3. **하위 호환성 유지**
   - 구버전 브로커도 지원해야 함

### 고민했던 선택지

#### 선택지 1: 고정 버전 사용

```python
# 항상 최신 버전 사용
request = ProduceRequest[9](...)
```

**장점**: 간단
**단점**: 구버전 브로커에서 실패
**왜 안 됨**: 하위 호환성 필수

#### 선택지 2: 사용자가 수동 설정

```python
# 사용자가 직접 지정
client = AIOKafkaClient(api_version=(0, 10, 0))
```

**장점**: 사용자가 제어
**단점**:
- 사용자가 버전 알아야 함 (불편)
- 실수 가능성

**왜 안 됨**: UX 나쁨

#### 선택지 3 (최종): 자동 협상

```python
# 1. 브로커에 버전 질의
response = await conn.send(ApiVersionRequest[0]())

# 2. 지원 버전 저장
versions = {}
for api_key, min_ver, max_ver in response.api_versions:
    versions[api_key] = (min_ver, max_ver)

# 3. 요청 시 자동 선택
def pick_best(request_versions):
    # 브로커가 지원하는 가장 높은 버전 선택
    for req_klass in reversed(request_versions):
        if min_ver <= req_klass.API_VERSION <= max_ver:
            return req_klass
```

**장점**:
- ✅ 자동화 (사용자 개입 불필요)
- ✅ 최적 버전 선택
- ✅ 하위 호환성

**단점**:
- ❌ 초기 연결 시 추가 왕복 (1 RTT)

**왜 선택했는가**: UX > 약간의 성능 손실

### 최종 해결책

#### 구조

```python
class VersionInfo:
    """브로커 지원 버전 저장"""

    def __init__(self, versions):
        # {api_key: (min_version, max_version)}
        self._versions = versions

    def pick_best(self, request_versions):
        """가장 적합한 버전 선택"""
        api_key = request_versions[0].API_KEY

        if api_key not in self._versions:
            # 브로커 정보 없으면 최소 버전 사용
            return request_versions[0]

        min_version, max_version = self._versions[api_key]

        # 역순으로 (높은 버전부터) 탐색
        for req_klass in reversed(request_versions):
            if min_version <= req_klass.API_VERSION <= max_version:
                return req_klass  # 매칭!

        # 매칭 실패
        raise KafkaError(
            f"No compatible version for API {api_key}"
        )


class AIOKafkaConnection:
    def __init__(self, ..., version_hint=None):
        self._version_hint = version_hint
        self._version_info = VersionInfo({})  # 초기엔 비어있음

    async def connect(self):
        # ...연결 생성...

        # Version lookup (Kafka 0.10+)
        if self._version_hint and self._version_hint >= (0, 10):
            await self._do_version_lookup()

    async def _do_version_lookup(self):
        """브로커에 지원 버전 질의"""
        # ApiVersionRequest는 v0만 있음 (안정적)
        version_req = ApiVersionRequest[0]()
        response = await self.send(version_req)

        # 버전 테이블 구성
        versions = {}
        for api_key, min_version, max_version in response.api_versions:
            versions[api_key] = (min_version, max_version)

        self._version_info = VersionInfo(versions)


# 사용 예시 (client.py에서)
class AIOKafkaClient:
    async def send(self, node_id, request):
        conn = await self._get_conn(node_id)

        # Version negotiation이 이미 완료됨
        # 자동으로 최적 버전 선택됨!
        return await conn.send(request)
```

#### 핵심 아이디어

**1. Lazy 초기화**

```python
# 초기: 비어있음
self._version_info = VersionInfo({})

# 첫 연결 시: 채움
await self._do_version_lookup()
```

**왜?**
- 모든 브로커가 ApiVersionRequest 지원하는 건 아님 (Kafka 0.9 이하)
- version_hint로 스킵 가능

**2. 역순 탐색 (높은 버전 우선)**

```python
for req_klass in reversed(request_versions):
    # ProduceRequest[9], [8], [7], ...
    if min_ver <= req_klass.API_VERSION <= max_ver:
        return req_klass  # 첫 매칭 = 가장 높은 버전
```

**왜?**
- 새 버전 = 더 많은 기능, 더 나은 성능
- 예: ProduceRequest v8부터 idempotence 지원

**3. Fallback to v0**

```python
if api_key not in self._versions:
    return request_versions[0]  # 최소 버전
```

**왜?**
- Version lookup 실패 시 (Kafka 0.9)
- 안전하게 v0 사용 (모든 브로커 지원)

---

## 전체 연결 흐름

### 연결 생성 시퀀스

```
User: await producer.start()
  ↓
Client: await self._get_conn(broker_id)
  ↓
Connection: await conn.connect()
  ├─ 1. TCP 연결 생성 (loop.create_connection)
  │    └─ StreamReader/Writer 준비
  ├─ 2. 백그라운드 읽기 Task 시작 (_read)
  ├─ 3. Idle checker 시작 (_idle_check)
  ├─ 4. API Version lookup (Kafka 0.10+)
  │    └─ await self._do_version_lookup()
  └─ 5. SASL 인증 (필요 시)
       └─ await self._do_sasl_handshake()
            ├─ Handshake request
            ├─ Authenticator 선택 (PLAIN/GSSAPI/SCRAM/OAUTH)
            └─ 인증 루프 (다단계 challenge-response)
```

### 요청/응답 플로우

```
User: await producer.send('topic', b'msg')
  ↓
Producer: await client.send(broker_id, ProduceRequest)
  ↓
Connection: fut = conn.send(request)
  ├─ 1. Correlation ID 생성 (0, 1, 2, ...)
  ├─ 2. 요청 인코딩 (header + body)
  ├─ 3. TCP 전송 (writer.write)
  ├─ 4. Future + ID 저장 (deque에 추가)
  └─ 5. Future 반환 (나중에 resolve됨)

백그라운드 읽기 Task:
  ├─ 1. 응답 크기 읽기 (4 bytes)
  ├─ 2. 응답 바디 읽기 (size bytes)
  ├─ 3. _handle_frame(resp)
  │    ├─ Correlation ID 검증
  │    ├─ 응답 디코딩
  │    ├─ Future.set_result(response)
  │    ├─ Idle timer 갱신
  │    └─ Deque에서 제거
  └─ 4. 다음 응답 대기 (while True)
```

### 연결 종료

```
브로커 종료 or 네트워크 오류:
  ↓
Protocol.connection_lost(exc)
  ↓
_closed_fut.set_result(None)
  ↓
_read Task 종료
  ↓
Connection.close()
  ├─ 1. Writer/Reader 정리
  ├─ 2. Pending 요청 모두 실패 처리
  ├─ 3. Callback 호출 (client._on_connection_closed)
  └─ 4. Idle handle 취소

Idle timeout:
  ↓
_idle_check() detects timeout
  ↓
Connection.close(reason=IDLE_DROP)
  └─ (동일한 정리 과정)
```

---

## 주요 클래스/메서드 참고

### AIOKafkaConnection

| 메서드 | 역할 | 반환 |
|--------|------|------|
| `connect()` | 연결 생성, SASL 인증 | `(reader, writer)` |
| `send(request)` | 요청 전송, Future 반환 | `Future[Response]` |
| `close(reason)` | 연결 종료, 정리 | `Future` |
| `connected()` | 연결 상태 확인 | `bool` |
| `_do_version_lookup()` | API 버전 협상 | `None` |
| `_do_sasl_handshake()` | SASL 인증 수행 | `None` |
| `_idle_check()` | Idle timeout 체크 | `None` |
| `_handle_frame()` | 응답 처리 | `None` |

### BaseSaslAuthenticator

| Authenticator | 방식 | 복잡도 |
|---------------|------|--------|
| `SaslPlainAuthenticator` | username/password | 간단 (1 step) |
| `SaslGSSAPIAuthenticator` | Kerberos | 복잡 (다단계) |
| `ScramAuthenticator` | HMAC SHA-256/512 | 중간 (3 steps) |
| `OAuthAuthenticator` | JWT 토큰 | 중간 (1 step) |

### 설정

| 설정 | 기본값 | 설명 |
|------|--------|------|
| `request_timeout_ms` | 40000 | 요청 타임아웃 (40초) |
| `max_idle_ms` | 540000 | Idle 연결 타임아웃 (9분) |
| `api_version` | "auto" | API 버전 (자동 협상) |
| `security_protocol` | "PLAINTEXT" | 보안 프로토콜 |
| `sasl_mechanism` | None | SASL 인증 방식 |

---

## 배운 점

### 1. 저수준 asyncio API의 활용

**고수준 API (open_connection)의 한계:**
```python
# 간단하지만 제한적
reader, writer = await asyncio.open_connection(host, port)
```

**저수준 API로 커스터마이징:**
```python
# 완전한 제어
reader = asyncio.StreamReader(limit=CUSTOM_LIMIT)
protocol = CustomProtocol(closed_fut, reader)
transport, _ = await loop.create_connection(
    lambda: protocol, host, port, ssl=ssl
)
```

**언제 저수준 API를 사용할까?**
- ✅ 버퍼 크기 제어 필요
- ✅ 연결 종료 감지 필요 (connection_lost)
- ✅ SSL, 인증 등 복잡한 설정

### 2. 플러그인 아키텍처로 확장성 확보

**문제**: 여러 인증 방식 지원
**해결**: 추상 기본 클래스 + 구현체들

```python
class BaseSaslAuthenticator:
    async def step(self, auth_bytes):
        raise NotImplementedError

# 각 방식별 구현
class SaslPlainAuthenticator(BaseSaslAuthenticator): ...
class SaslGSSAPIAuthenticator(BaseSaslAuthenticator): ...
```

**이점**:
- 새 방식 추가 시 기존 코드 변경 없음
- 각 방식 독립 테스트
- 사용자 정의 인증 가능

**적용 가능한 경우**:
- 여러 전략/알고리즘 지원 (Strategy 패턴)
- 플러그인 시스템
- 다양한 백엔드 지원 (DB, 캐시, 스토리지)

### 3. Correlation ID로 비동기 매칭

**문제**: 응답 순서가 요청과 다를 수 있음
**해결**: 각 요청에 고유 ID 부여

```python
# 전송
correlation_id = self._next_correlation_id()
self._requests.append((correlation_id, request, fut))

# 수신
if response.correlation_id == correlation_id:
    fut.set_result(response)
```

**적용 가능한 경우**:
- HTTP/2, gRPC (stream ID)
- 데이터베이스 드라이버 (query ID)
- RPC 프레임워크

### 4. weakref로 순환 참조 방지

**문제**: 백그라운드 Task가 객체를 참조 → 순환 참조
```python
# 나쁜 예
async def _read(self):  # 강한 참조
    while True:
        ...
```

**해결**: weakref 사용
```python
# 좋은 예
@staticmethod
async def _read(self_ref):  # weakref
    self = self_ref()
    if self is None:
        return  # 이미 해제됨
```

**언제 사용?**
- 백그라운드 Task/Timer
- 콜백 함수
- 캐시 구현

### 5. Idle timeout은 트레이드오프

**짧은 timeout:**
- ✅ 리소스 빨리 정리
- ❌ 재연결 빈번 (성능 저하)

**긴 timeout:**
- ✅ 재연결 드묾
- ❌ 리소스 낭비

**해결**: 설정으로 제어
```python
connections_max_idle_ms=540000  # 사용자가 조정 가능
```

**기본값 선택 기준**:
- 브로커 기본값보다 짧게 (클라이언트가 먼저 종료)
- 일반적인 사용 패턴 고려 (버스트 트래픽)

### 6. API Version Negotiation의 중요성

**문제**: 다양한 버전 지원
**해결**: 자동 협상

```python
# 브로커에 질의
response = await conn.send(ApiVersionRequest())

# 자동 선택
best_version = version_info.pick_best([v0, v1, v2, ...])
```

**이점**:
- 사용자 편의성 (버전 몰라도 됨)
- 최적 버전 자동 선택
- 하위 호환성

**적용 가능**:
- 프로토콜 버전 관리 (HTTP, gRPC)
- Feature negotiation
- Capability discovery

### 7. 에러 처리의 세밀함

이 파일은 **6가지 종료 이유**를 구분합니다:

```python
class CloseReason(IntEnum):
    CONNECTION_BROKEN = 0    # 네트워크 오류
    CONNECTION_TIMEOUT = 1   # 타임아웃
    OUT_OF_SYNC = 2         # Correlation ID 불일치
    IDLE_DROP = 3           # Idle timeout
    SHUTDOWN = 4            # 정상 종료
    AUTH_FAILURE = 5        # 인증 실패
```

**왜 중요한가?**
- 디버깅: 어떤 이유로 종료됐는지 명확
- 재시도 로직: 이유에 따라 다르게 처리
- 모니터링: 메트릭 수집 (AUTH_FAILURE 비율 등)

### 8. 비슷한 상황에 적용

이 파일의 패턴들을 적용할 수 있는 경우:

| 패턴 | 적용 가능한 곳 |
|------|----------------|
| 저수준 asyncio API | 네트워크 라이브러리, 커스텀 프로토콜 |
| 플러그인 아키텍처 | 인증, 압축, 직렬화 방식 지원 |
| Correlation ID | HTTP/2, gRPC, 데이터베이스 드라이버 |
| weakref | 백그라운드 작업, 캐시 |
| Idle timeout | DB 연결 풀, HTTP keep-alive |
| Version negotiation | API 서버, 프로토콜 구현 |
| 세밀한 에러 처리 | 프로덕션 시스템, 모니터링 |

---

## 실전 적용 가이드

이 섹션은 **실제 프로젝트에 패턴을 적용**할 때 도움이 됩니다.

### 가이드 1: 비동기 TCP 연결 라이브러리 만들기

**상황**: gRPC, Redis, 또는 커스텀 프로토콜 클라이언트 구현

#### Step 1: 요구사항 정의

```markdown
질문:
- [ ] 버퍼 크기 제어가 필요한가? (대용량 메시지)
- [ ] 연결 종료 감지가 중요한가? (빠른 재연결)
- [ ] SSL/TLS 지원이 필요한가?
- [ ] 인증 방식이 여러 개인가?
```

#### Step 2: API 레벨 선택

| 요구사항 | 선택 | 이유 |
|----------|------|------|
| 간단한 클라이언트 | `asyncio.open_connection()` | 충분함 |
| 버퍼 크기 제어 필요 | 저수준 API (StreamReader) | 제어 가능 |
| 연결 종료 감지 필요 | 커스텀 Protocol | connection_lost() |

#### Step 3: 구현 템플릿

```python
import asyncio

class MyProtocol(asyncio.StreamReaderProtocol):
    """연결 종료 감지용"""
    def __init__(self, closed_fut, *args, **kw):
        self._closed_fut = closed_fut
        super().__init__(*args, **kw)

    def connection_lost(self, exc):
        super().connection_lost(exc)
        if not self._closed_fut.cancelled():
            self._closed_fut.set_result(exc)

class MyConnection:
    async def connect(self, host, port, *, buffer_size=65536, timeout=30):
        loop = asyncio.get_running_loop()
        self._closed_fut = loop.create_future()

        # 1. StreamReader 생성 (버퍼 크기 지정)
        reader = asyncio.StreamReader(limit=buffer_size, loop=loop)
        protocol = MyProtocol(self._closed_fut, reader, loop=loop)

        # 2. 연결 생성
        async with asyncio.timeout(timeout):
            transport, _ = await loop.create_connection(
                lambda: protocol,
                host, port,
                ssl=None  # 필요 시 SSLContext
            )

        # 3. StreamWriter 생성
        writer = asyncio.StreamWriter(transport, protocol, reader, loop)
        self._reader, self._writer = reader, writer

        # 4. 백그라운드 읽기 Task
        self._read_task = asyncio.create_task(self._read_loop())

        return self

    async def _read_loop(self):
        """백그라운드에서 응답 읽기"""
        while True:
            try:
                data = await self._reader.read(4096)
                if not data:
                    break
                self._handle_data(data)
            except Exception:
                break
```

#### Step 4: 의사결정 체크리스트

**버퍼 크기 결정:**
- 메시지 평균 크기 < 8KB → 기본값 (8KB) 사용
- 메시지 평균 크기 8-64KB → 64KB
- 메시지 평균 크기 > 64KB → 메시지 크기 × 2

**타임아웃 설정:**
- 로컬 네트워크 (LAN) → 5-10초
- 인터넷 → 30-60초
- 불안정한 네트워크 → 60-120초

---

### 가이드 2: 플러그인 인증 시스템 만들기

**상황**: 여러 인증 방식을 지원해야 하는 API 클라이언트

#### Step 1: 인증 방식 파악

```markdown
지원할 방식:
- [ ] API Key (간단)
- [ ] OAuth 2.0 (복잡)
- [ ] JWT (중간)
- [ ] Custom (커스텀)
```

#### Step 2: 추상 클래스 설계

```python
from abc import ABC, abstractmethod

class BaseAuthenticator(ABC):
    """모든 인증의 기본"""

    @abstractmethod
    async def authenticate(self, request):
        """
        요청에 인증 정보 추가

        Returns:
            modified_request: 인증 정보가 추가된 요청
        """
        pass

# 구현체들
class ApiKeyAuthenticator(BaseAuthenticator):
    def __init__(self, api_key):
        self._api_key = api_key

    async def authenticate(self, request):
        request.headers['Authorization'] = f'Bearer {self._api_key}'
        return request

class OAuthAuthenticator(BaseAuthenticator):
    def __init__(self, token_provider):
        self._token_provider = token_provider
        self._token = None
        self._token_expiry = 0

    async def authenticate(self, request):
        # 토큰 갱신 필요?
        if time.time() >= self._token_expiry:
            self._token = await self._token_provider.get_token()
            self._token_expiry = time.time() + 3600

        request.headers['Authorization'] = f'Bearer {self._token}'
        return request

# 사용
class MyClient:
    def __init__(self, authenticator: BaseAuthenticator):
        self._auth = authenticator

    async def request(self, endpoint, data):
        req = Request(endpoint, data)
        req = await self._auth.authenticate(req)  # 플러그인!
        return await self._send(req)
```

#### Step 3: 새 인증 방식 추가 (확장)

```python
# 기존 코드 수정 없이 추가!
class CustomAuthenticator(BaseAuthenticator):
    async def authenticate(self, request):
        # 커스텀 로직
        ...
        return request

# 사용
client = MyClient(CustomAuthenticator(...))
```

---

### 가이드 3: Correlation ID 패턴 구현

**상황**: 비동기 요청/응답에서 매칭 필요

#### Step 1: 요구사항 확인

```markdown
- [ ] 응답 순서가 요청과 다를 수 있는가? → YES면 필수
- [ ] 동시 요청이 여러 개인가? → YES면 필수
- [ ] 요청 타임아웃이 필요한가? → YES면 Future 사용
```

#### Step 2: 구현

```python
import collections
import asyncio

class RequestResponseClient:
    def __init__(self):
        self._correlation_id = 0
        self._pending = collections.deque()  # [(id, fut), ...]
        self._lock = asyncio.Lock()

    def _next_id(self):
        """순환 ID 생성"""
        self._correlation_id = (self._correlation_id + 1) % (2**31)
        return self._correlation_id

    async def send(self, request, timeout=30):
        """요청 전송"""
        # 1. ID 생성
        corr_id = self._next_id()

        # 2. 요청 인코딩 (ID 포함)
        encoded = self._encode(request, correlation_id=corr_id)

        # 3. Future 생성
        loop = asyncio.get_running_loop()
        fut = loop.create_future()

        # 4. Pending에 추가
        self._pending.append((corr_id, fut))

        # 5. 전송
        await self._writer.write(encoded)

        # 6. 응답 대기 (timeout)
        try:
            return await asyncio.wait_for(fut, timeout=timeout)
        except asyncio.TimeoutError:
            # Pending에서 제거
            self._pending = collections.deque(
                (id, f) for id, f in self._pending if id != corr_id
            )
            raise

    async def _read_loop(self):
        """응답 읽기 (백그라운드)"""
        while True:
            # 1. 응답 읽기
            response = await self._reader.read()
            response_id = response.correlation_id

            # 2. Pending에서 찾기 (FIFO)
            if not self._pending:
                continue

            expected_id, fut = self._pending[0]

            # 3. ID 검증
            if response_id != expected_id:
                # 순서 틀림! → 프로토콜 오류
                print(f"Expected {expected_id}, got {response_id}")
                fut.set_exception(ProtocolError("Out of sync"))
                return

            # 4. Future resolve
            fut.set_result(response)
            self._pending.popleft()
```

#### Step 4: 주의사항

**❌ 하지 말 것:**
```python
# ID 충돌 가능 (순환 안 함)
self._correlation_id += 1  # 오버플로우!
```

**✅ 해야 할 것:**
```python
# 순환 ID
self._correlation_id = (self._correlation_id + 1) % 2**31
```

---

### 가이드 4: Idle Timeout 구현

**상황**: 오래 사용하지 않은 연결 정리

#### Step 1: 설정 결정

| 트래픽 패턴 | Idle Timeout | 이유 |
|-------------|--------------|------|
| 지속적 (24/7) | 긴 시간 (10분+) | 재연결 비용 |
| 버스트 (간헐적) | 중간 (5분) | 균형 |
| 드묾 (하루 1번) | 짧음 (1분) | 리소스 절약 |

#### Step 2: 구현

```python
import time
import weakref

class IdleConnection:
    def __init__(self, max_idle_ms=300000):  # 5분
        self._max_idle_ms = max_idle_ms
        self._last_action = time.monotonic()
        self._idle_handle = None

    async def connect(self):
        # ...연결...

        # Idle checker 시작
        if self._max_idle_ms is not None:
            loop = asyncio.get_running_loop()
            self._idle_handle = loop.call_soon(
                self._idle_check,
                weakref.ref(self)  # 중요!
            )

    @staticmethod
    def _idle_check(self_ref):
        """주기적 체크 (static + weakref)"""
        self = self_ref()
        if self is None:
            return  # 이미 해제됨

        idle_time = time.monotonic() - self._last_action
        timeout = self._max_idle_ms / 1000

        if idle_time >= timeout:
            # Idle timeout → 종료
            self.close()
        else:
            # 다시 스케줄링
            wake_up = timeout - idle_time
            loop = asyncio.get_running_loop()
            self._idle_handle = loop.call_later(
                wake_up,
                IdleConnection._idle_check,
                self_ref
            )

    def _update_activity(self):
        """활동 시 호출"""
        self._last_action = time.monotonic()
```

#### Step 3: weakref 사용 이유

**❌ 문제 (강한 참조):**
```python
# 순환 참조!
loop.call_later(timeout, self._idle_check, self)
#                                          ^^^^
# loop → callback → self → loop
# → 메모리 누수!
```

**✅ 해결 (weakref):**
```python
loop.call_later(timeout, IdleConnection._idle_check, weakref.ref(self))
#                        ^^^^^^^^^^^^^^^ static      ^^^^^^^^^^^^^^^^ weak
# self 삭제되면 callback도 자동 정리됨
```

---

## 안티패턴과 흔한 실수

실제 구현 시 자주 발생하는 문제들과 해결책입니다.

### 실수 1: Lock 안에서 I/O

**❌ 나쁜 예:**
```python
async def get_connection(self, node_id):
    async with self._lock:
        if node_id not in self._conns:
            # Lock 안에서 네트워크 I/O!
            conn = await create_conn(host, port)  # 수백 ms
            self._conns[node_id] = conn
        return self._conns[node_id]
```

**문제:**
- Lock 시간 = 연결 생성 시간 (수백 ms)
- 다른 Task들 모두 대기 → 전체 성능 저하

**✅ 좋은 예:**
```python
async def get_connection(self, node_id):
    # 1. Lock 밖에서 확인
    async with self._lock:
        if node_id in self._conns:
            return self._conns[node_id]

    # 2. Lock 밖에서 I/O
    conn = await create_conn(host, port)  # 느린 작업

    # 3. Lock 안에서 빠른 작업만
    async with self._lock:
        if node_id not in self._conns:  # Double-check!
            self._conns[node_id] = conn
        return self._conns[node_id]
```

---

### 실수 2: Double-check 없이 생성

**❌ 나쁜 예:**
```python
async def get_connection(self, node_id):
    if node_id not in self._conns:
        async with self._lock:
            # Double-check 없음!
            self._conns[node_id] = await create_conn(...)
    return self._conns[node_id]
```

**문제:**
```
Task A: node_id 없음 → Lock 대기
Task B: node_id 없음 → Lock 대기
Task A: Lock 획득 → 생성 → 저장
Task B: Lock 획득 → 또 생성! (중복)
```

**✅ 좋은 예:**
```python
async def get_connection(self, node_id):
    if node_id not in self._conns:
        async with self._lock:
            if node_id not in self._conns:  # Double-check!
                self._conns[node_id] = await create_conn(...)
    return self._conns[node_id]
```

---

### 실수 3: weakref 없이 콜백 등록

**❌ 나쁜 예:**
```python
def start_idle_check(self):
    loop = asyncio.get_running_loop()
    loop.call_later(300, self._idle_check)  # 강한 참조!
    #                    ^^^^^^^^^^^^^^^^
    # loop → callback → self → ... → loop
    # 순환 참조!
```

**문제:**
- 객체 삭제해도 callback이 참조 유지
- 메모리 누수

**✅ 좋은 예:**
```python
def start_idle_check(self):
    loop = asyncio.get_running_loop()
    loop.call_later(
        300,
        self._idle_check_static,  # static 메서드
        weakref.ref(self)         # weak 참조
    )

@staticmethod
def _idle_check_static(self_ref):
    self = self_ref()
    if self is None:
        return  # 이미 삭제됨
    # ...체크...
```

---

### 실수 4: Correlation ID 오버플로우

**❌ 나쁜 예:**
```python
def next_id(self):
    self._id += 1
    return self._id  # 2^63-1 넘으면 오버플로우!
```

**문제:**
- 장기 실행 프로세스에서 오버플로우
- Python은 무한 int지만 프로토콜은 32bit or 64bit 제한

**✅ 좋은 예:**
```python
def next_id(self):
    self._id = (self._id + 1) % (2**31)  # 순환!
    return self._id
```

---

### 실수 5: 에러 무시

**❌ 나쁜 예:**
```python
async def connect(self):
    try:
        self._conn = await create_conn(...)
    except Exception:
        pass  # 에러 무시!
```

**문제:**
- 연결 실패해도 모름
- 나중에 사용 시 None 에러

**✅ 좋은 예:**
```python
async def connect(self):
    try:
        self._conn = await create_conn(...)
    except OSError as err:
        log.error("Connection failed: %s", err)
        raise ConnectionError(f"Cannot connect to {host}:{port}") from err
    except asyncio.TimeoutError:
        log.error("Connection timeout")
        raise ConnectionError("Connection timeout") from err
```

---

### 실수 6: asyncio.Lock vs threading.Lock 혼동

**❌ 나쁜 예:**
```python
class MyClass:
    def __init__(self):
        # 이벤트 루프 없이 asyncio.Lock 생성!
        self._lock = asyncio.Lock()  # 에러!
```

**문제:**
- asyncio.Lock은 running loop 필요
- `__init__`에서 생성 시 loop 없을 수 있음

**✅ 방법 1: Lazy 생성**
```python
@property
def _lock(self):
    if not hasattr(self, '_lock_instance'):
        self._lock_instance = asyncio.Lock()
    return self._lock_instance
```

**✅ 방법 2: threading.Lock (더 간단)**
```python
import threading

class MyClass:
    def __init__(self):
        self._lock = threading.Lock()  # OK!

    async def method(self):
        # asyncio에서도 사용 가능
        with self._lock:
            ...
```

---

### 실수 7: Future 취소 처리 안 함

**❌ 나쁜 예:**
```python
async def send(self, request):
    fut = asyncio.Future()
    self._pending.append(fut)

    await self._writer.write(request)

    return await fut  # 취소되면?
```

**문제:**
- Task 취소 시 Future가 pending에 남음
- 메모리 누수

**✅ 좋은 예:**
```python
async def send(self, request):
    fut = asyncio.Future()
    self._pending.append((id, fut))

    try:
        await self._writer.write(request)
        return await fut
    except asyncio.CancelledError:
        # 취소 시 pending에서 제거
        self._pending = [
            (i, f) for i, f in self._pending if f != fut
        ]
        raise
```

---

## 스케일 고려사항

규모별로 다른 전략이 필요합니다.

### 소규모 (연결 < 10, 요청 < 1k/s)

**권장 사항:**
- ✅ 간단한 구현도 OK
- ✅ Lock은 필요하지만 경합 적음
- ⚠️  매번 새 연결도 고려 가능

**구현 예시:**
```python
# 간단한 방식도 충분
class SimpleClient:
    async def send(self, request):
        # Connection Pool 없이
        async with asyncio.timeout(30):
            conn = await asyncio.open_connection(host, port)
            await conn.write(request)
            return await conn.read()
```

**모니터링:**
- 연결 실패율
- 평균 응답 시간

---

### 중규모 (연결 10-100, 요청 1k-10k/s)

**권장 사항:**
- ✅ Connection Pool 필수
- ✅ Lazy 생성
- ✅ Idle timeout 설정
- ⚠️  Lock 경합 시작됨 → Double-check 중요

**구현 예시:**
```python
class MediumClient:
    def __init__(self):
        self._conns = {}
        self._lock = asyncio.Lock()
        self._max_idle_ms = 300000  # 5분

    async def get_conn(self, node_id):
        # Lock 밖 확인
        if node_id in self._conns:
            conn = self._conns[node_id]
            if conn.connected():
                return conn

        # Lock 안 생성
        async with self._lock:
            if node_id not in self._conns:
                self._conns[node_id] = await create_conn(
                    ...,
                    max_idle_ms=self._max_idle_ms
                )
        return self._conns[node_id]
```

**모니터링:**
- Pool 크기 (연결 수)
- Lock 대기 시간
- Idle 연결 비율
- 재연결 빈도

---

### 대규모 (연결 100+, 요청 10k+/s)

**권장 사항:**
- ✅ Connection Pool 최대 크기 제한
- ✅ Health check 추가
- ✅ Circuit breaker 고려
- ✅ Metrics/Monitoring 필수
- ⚠️  Lock-free 알고리즘 고려

**추가 최적화:**

```python
class LargeScaleClient:
    def __init__(self, max_conns=1000):
        self._conns = {}
        self._lock = asyncio.Lock()
        self._max_conns = max_conns
        self._conn_semaphore = asyncio.Semaphore(max_conns)

        # Metrics
        self._metrics = {
            'total_requests': 0,
            'failed_requests': 0,
            'pool_size': 0,
        }

    async def get_conn(self, node_id):
        # Semaphore로 최대 연결 수 제한
        await self._conn_semaphore.acquire()

        try:
            # ...기존 로직...

            # Pool 크기 체크
            if len(self._conns) >= self._max_conns:
                # LRU 방식으로 오래된 연결 제거
                await self._evict_idle_connection()

            # ...
        finally:
            self._conn_semaphore.release()

    async def _evict_idle_connection(self):
        """가장 오래 사용하지 않은 연결 제거"""
        # LRU 알고리즘 구현
        ...
```

**모니터링 필수:**
```python
# Prometheus 스타일
- connection_pool_size: 현재 Pool 크기
- connection_pool_active: 사용 중인 연결
- connection_create_duration_seconds: 연결 생성 시간
- connection_failures_total: 연결 실패 횟수
- request_duration_seconds: 요청 처리 시간
```

**Circuit Breaker 추가:**
```python
from enum import Enum

class CircuitState(Enum):
    CLOSED = "closed"  # 정상
    OPEN = "open"      # 실패 많음 → 차단
    HALF_OPEN = "half_open"  # 회복 중

class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self._failures = 0
        self._threshold = failure_threshold
        self._timeout = timeout
        self._state = CircuitState.CLOSED
        self._last_failure_time = 0

    async def call(self, func, *args, **kwargs):
        if self._state == CircuitState.OPEN:
            # 차단 중
            if time.time() - self._last_failure_time > self._timeout:
                self._state = CircuitState.HALF_OPEN
            else:
                raise CircuitBreakerOpen("Too many failures")

        try:
            result = await func(*args, **kwargs)
            # 성공 → 회복
            if self._state == CircuitState.HALF_OPEN:
                self._state = CircuitState.CLOSED
                self._failures = 0
            return result
        except Exception as err:
            # 실패 → 카운트
            self._failures += 1
            self._last_failure_time = time.time()

            if self._failures >= self._threshold:
                self._state = CircuitState.OPEN

            raise
```

---

### 병목 지점과 해결책

| 병목 | 증상 | 해결책 |
|------|------|--------|
| Lock 경합 | Lock 대기 시간 증가 | Lock-free 알고리즘, Shard |
| 연결 생성 느림 | 첫 요청 지연 | Connection Pool 미리 워밍업 |
| 메모리 부족 | OOM | Pool 크기 제한, LRU eviction |
| CPU 100% | Context switching | 연결 수 제한, Rate limiting |

---

## 요약

| 문제 | 선택지 | 최종 해결 | 트레이드오프 |
|------|--------|-----------|--------------|
| TCP 연결 | 동기 vs open_connection vs 저수준 | 저수준 API | 복잡도 vs 제어력 |
| SASL 인증 | 하나만 vs if-else vs 플러그인 | 플러그인 | 코드 분산 vs 확장성 |
| 요청/응답 | 동기 vs Queue vs Correlation ID | Correlation ID | 구현 복잡도 vs 동시성 |
| Idle 연결 | 계속 유지 vs ping vs timeout | Timeout | 재연결 비용 vs 리소스 절약 |
| API 버전 | 고정 vs 수동 vs 자동 | 자동 협상 | 추가 RTT vs UX |

**핵심 메시지:**
- 간단함 < 성능 + 확장성 (고성능 라이브러리이므로)
- 복잡도는 추상화로 감춤 (사용자는 간단한 API만)
- 모든 선택은 트레이드오프 → 설정으로 제어 가능

---

**분석 완료일**: 2025-10-29
**방법론**: "문제 → 고민 → 해결" 중심 파일별 분석
**다음 분석**: client.py (Connection Pool 관리)
