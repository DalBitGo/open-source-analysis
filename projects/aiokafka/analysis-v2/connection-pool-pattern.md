# Connection Pool 패턴 - aiokafka

> **분석 일자**: 2025-10-29
> **파일**: `aiokafka/client.py` (line 162, 419-482)
> **방법론**: 문제 해결 중심 분석

---

## 해결하려는 문제

### 문제 상황

Kafka 클라이언트가 브로커와 통신하려면 **TCP 연결**이 필요합니다. 하지만:

1. **TCP 연결 생성 비용이 크다**
   - 3-way handshake (수 ms)
   - SASL 인증 (추가 수십~수백 ms)
   - SSL/TLS handshake (보안 모드 시)

2. **자주 통신해야 한다**
   - Producer: 초당 수백~수천 개 메시지 전송
   - Consumer: 지속적으로 메시지 폴링
   - 매번 새 연결 생성 시 → 성능 저하

3. **여러 브로커와 통신**
   - Kafka는 여러 브로커로 구성 (클러스터)
   - 각 브로커마다 별도 연결 필요
   - 파티션 리더가 다른 브로커에 있을 수 있음

4. **동시 요청 필요**
   - 비동기 환경 (asyncio)
   - 여러 요청을 동시에 보내야 함 (성능 최적화)

### 누구를 위한 해결책인가?

- **개발자**: 복잡한 연결 관리를 신경 쓰지 않고 `send()` 호출만 하면 됨
- **시스템**: 연결 재사용으로 CPU/메모리 절약, 처리량 증가
- **Kafka 브로커**: 불필요한 연결 생성/종료 부하 감소

---

## 고민했던 선택지

### 선택지 1: 매번 새 연결 생성

```python
# 간단한 방법
async def send(node_id, request):
    conn = await create_connection(broker.host, broker.port)
    await conn.send(request)
    await conn.close()
```

**장점:**
- 코드가 매우 간단
- 연결 관리 불필요
- 메모리 사용 최소

**단점:**
- 성능이 매우 느림 (연결 생성 비용)
- 초당 1000개 메시지 전송 시 → 1000번 연결 생성
- SASL 인증 반복 → 추가 지연

**왜 선택하지 않았는가?**
→ Kafka는 **고성능 메시징**을 목표로 하므로, 이 방법은 성능 요구사항을 만족하지 못함

---

### 선택지 2: 하나의 연결 재사용

```python
# 연결 하나만 유지
_conn = await create_connection(broker.host, broker.port)

async def send(node_id, request):
    await _conn.send(request)
```

**장점:**
- 빠름 (연결 재사용)
- 간단함 (하나의 연결만 관리)

**단점:**
- **동시 요청 불가능**
  - 하나의 연결은 순차적으로만 처리
  - 비동기 환경에서 병목 발생
- **여러 브로커 지원 불가**
  - Kafka는 여러 브로커로 구성
  - 브로커마다 다른 연결 필요

**왜 선택하지 않았는가?**
→ Kafka 클러스터는 **여러 브로커**로 구성되고, **동시 요청**이 필수

---

### 선택지 3 (최종 선택): Connection Pool

```python
# 브로커별로 연결 관리
self._conns = {}  # {(node_id, group): AIOKafkaConnection}

async def _get_conn(self, node_id, group=DEFAULT):
    conn_id = (node_id, group)

    # 1. 기존 연결 재사용
    if conn_id in self._conns:
        conn = self._conns[conn_id]
        if conn.connected():
            return conn  # 재사용!
        else:
            del self._conns[conn_id]  # 끊어진 연결 제거

    # 2. 새 연결 생성
    async with self._get_conn_lock:  # 동시 생성 방지
        if conn_id not in self._conns:
            self._conns[conn_id] = await create_conn(...)

    return self._conns[conn_id]
```

**장점:**
- ✅ **빠름**: 연결 재사용 → 생성 비용 절약
- ✅ **동시 요청 가능**: 여러 연결을 풀에 유지
- ✅ **브로커별 관리**: 각 브로커마다 독립적인 연결
- ✅ **자동 관리**: 끊어진 연결 감지 및 재생성

**단점:**
- ❌ **복잡함**: Lock, 딕셔너리 관리, 상태 추적
- ❌ **메모리 사용**: 여러 연결을 메모리에 유지
- ❌ **리소스 관리**: idle 연결 정리 필요

**왜 선택했는가?**
→ Kafka의 고성능, 다중 브로커, 비동기 요구사항을 모두 만족하는 **유일한 방법**

---

## 최종 해결책

### 구조

```python
# aiokafka/client.py

class AIOKafkaClient:
    def __init__(self, ..., connections_max_idle_ms=540000):
        self._conns = {}  # Connection Pool
        self._get_conn_lock = asyncio.Lock()
        self._connections_max_idle_ms = connections_max_idle_ms

    async def _get_conn(self, node_id, *, group=DEFAULT):
        """연결 가져오기 or 생성"""
        conn_id = (node_id, group)

        # 1. 기존 연결 확인
        if conn_id in self._conns:
            if self._conns[conn_id].connected():
                return self._conns[conn_id]  # 재사용
            del self._conns[conn_id]  # 끊어진 연결 제거

        # 2. 새 연결 생성 (Lock으로 동시 생성 방지)
        async with self._get_conn_lock:
            if conn_id in self._conns:  # Double-check
                return self._conns[conn_id]

            broker = self.cluster.broker_metadata(node_id)
            self._conns[conn_id] = await create_conn(
                broker.host, broker.port,
                max_idle_ms=self._connections_max_idle_ms,
                on_close=self._on_connection_closed,  # 콜백 등록
                ...
            )

        return self._conns[conn_id]

    async def send(self, node_id, request):
        """사용자가 호출하는 메서드"""
        if not await self.ready(node_id):  # 연결 준비 확인
            raise NodeNotReadyError(...)

        conn = self._conns[(node_id, group)]
        return await conn.send(request)  # 실제 전송
```

### 핵심 아이디어

#### 1. 딕셔너리 기반 Pool

```python
self._conns = {
    (0, DEFAULT): <Connection to broker 0>,
    (1, DEFAULT): <Connection to broker 1>,
    (2, COORDINATION): <Connection to coordinator>,
}
```

**왜 딕셔너리인가?**
- 키: `(node_id, group)` → 브로커별 + 그룹별 연결 관리
- O(1) 조회 성능
- Python의 기본 자료구조 (추가 의존성 없음)

**왜 튜플 키인가?**
- `node_id`: 브로커 식별자 (0, 1, 2, ...)
- `group`: 연결 용도 (DEFAULT vs COORDINATION)
  - DEFAULT: 일반 메시지 전송
  - COORDINATION: Consumer group 코디네이터

#### 2. Lock으로 동시 생성 방지

```python
async with self._get_conn_lock:
    if conn_id not in self._conns:  # Double-check pattern
        self._conns[conn_id] = await create_conn(...)
```

**문제**: 동시에 여러 코루틴이 같은 연결 생성 시도
```python
# 나쁜 예: Lock 없이
Task A: conn not in _conns → create_conn() 시작
Task B: conn not in _conns → create_conn() 시작  # 중복!
```

**해결**: Lock + Double-check
- Lock: 한 번에 하나의 Task만 진입
- Double-check: Lock 획득 후 다시 확인 (먼저 들어온 Task가 이미 생성했을 수 있음)

#### 3. Idle 연결 관리

```python
# conn.py
class AIOKafkaConnection:
    def __init__(self, ..., max_idle_ms=540000):  # 9분
        self._max_idle_ms = max_idle_ms
        self._last_action = time.monotonic()

    @staticmethod
    def _idle_check(conn_ref):
        """주기적으로 idle 시간 확인 (weakref 사용)"""
        conn = conn_ref()
        if conn is None:
            return  # 이미 해제됨

        if (time.monotonic() - conn._last_action) * 1000 > conn._max_idle_ms:
            conn.close(reason=CloseReason.IDLE_DROP)
        else:
            # 다시 스케줄링
            conn._loop.call_later(30, AIOKafkaConnection._idle_check, conn_ref)
```

**왜 필요한가?**
- 사용하지 않는 연결을 계속 유지 → 메모리/소켓 낭비
- 브로커도 idle 연결 유지 비용 발생

**트레이드오프:**
- 너무 짧은 timeout: 자주 재생성 (성능 저하)
- 너무 긴 timeout: 리소스 낭비
- 기본값 540초(9분): Kafka 브로커 기본값과 일치

---

### 실제 코드 예시

#### 사용자 관점 (간단함)

```python
from aiokafka import AIOKafkaProducer

producer = AIOKafkaProducer(bootstrap_servers='localhost:9092')
await producer.start()

# 내부적으로 Connection Pool 사용
await producer.send('my-topic', b'message 1')  # 연결 생성
await producer.send('my-topic', b'message 2')  # 연결 재사용!
await producer.send('my-topic', b'message 3')  # 연결 재사용!

# 사용자는 연결 관리 신경 안 써도 됨
```

#### 내부 동작 (복잡함)

```python
# producer.send() 내부 흐름

1. producer.send('my-topic', b'msg')
   ↓
2. client.send(leader_node_id, ProduceRequest)
   ↓
3. conn = await client._get_conn(leader_node_id)
   ├─ _conns에 있으면 → 재사용
   └─ 없으면 → create_conn() → _conns에 저장
   ↓
4. await conn.send(request)
   ↓
5. 응답 반환
```

---

## 트레이드오프

### 얻은 것 ✅

1. **성능 향상**
   - 연결 재사용으로 생성 비용 제거
   - 벤치마크: 10배 이상 빠름 (연결 생성 vs 재사용)

2. **동시성 확보**
   - 여러 브로커에 동시 요청 가능
   - 비동기 환경 최적화

3. **자동 관리**
   - 끊어진 연결 자동 감지 및 재생성
   - Idle 연결 자동 정리
   - 사용자는 신경 안 써도 됨

4. **확장성**
   - 브로커 추가 시 자동으로 새 연결 생성
   - 그룹별 연결 관리 (DEFAULT, COORDINATION)

### 포기한 것 ❌

1. **간단함**
   - 딕셔너리 관리
   - Lock 사용
   - 상태 추적 (connected, idle)
   - 콜백 등록 (`on_close`)

2. **메모리 사용**
   - 여러 연결을 메모리에 유지
   - 각 연결마다 reader/writer 버퍼
   - 브로커 10개 × 평균 100KB = ~1MB

3. **디버깅 복잡도**
   - 연결 상태 추적 어려움
   - Lock 경합 가능성
   - 타이밍 이슈 (race condition)

---

## 배운 점

### 1. Connection Pool은 "필수"가 아니라 "선택"

네트워크 라이브러리에서 항상 Connection Pool이 필요한 건 아닙니다:
- **간단한 CLI 도구**: 매번 새 연결 (간단함 우선)
- **고성능 서버**: Pool 사용 (성능 우선)
- **중간 규모**: 상황에 따라 선택

**결정 기준:**
- 연결 생성 빈도가 높은가?
- 성능이 중요한가?
- 복잡도를 감수할 가치가 있는가?

### 2. Lock + Double-check 패턴

비동기 환경에서 리소스 생성 시 필수:

```python
async with self._lock:
    if resource not in self._pool:  # Double-check!
        self._pool[resource] = await create_resource()
```

**왜 필요한가?**
- Lock 없으면: 중복 생성
- Double-check 없으면: Lock 대기 중 다른 Task가 이미 생성했을 수 있음

### 3. 설정으로 트레이드오프 제어

```python
connections_max_idle_ms=540000  # 사용자가 조절 가능
```

완벽한 기본값은 없습니다. 사용자가 상황에 맞게 선택:
- 연결이 많은 환경: 짧은 timeout (메모리 절약)
- 요청이 드문 환경: 긴 timeout (재생성 비용 절약)

### 4. weakref로 메모리 누수 방지

```python
# conn.py
self._idle_handle = loop.call_soon(
    self._idle_check,
    weakref.ref(self)  # Strong reference 방지
)
```

**문제**: 순환 참조 → 메모리 누수
```
Connection → idle_check callback → Connection (강한 참조)
```

**해결**: weakref 사용
```
Connection → idle_check callback → weakref(Connection) (약한 참조)
```

### 5. 비슷한 상황에 적용

이 패턴을 사용할 수 있는 경우:
- ✅ **데이터베이스 연결 풀** (PostgreSQL, MySQL)
- ✅ **HTTP 클라이언트** (aiohttp, httpx)
- ✅ **gRPC 채널 관리**
- ✅ **Redis 연결 풀**

사용하지 말아야 할 경우:
- ❌ 연결 생성이 빠른 경우 (Unix socket)
- ❌ 요청 빈도가 매우 낮은 경우 (하루 1번)
- ❌ 간단함이 더 중요한 경우 (프로토타입)

---

## 참고 자료

### 관련 파일
- `aiokafka/client.py:162` - `self._conns = {}` (Pool 초기화)
- `aiokafka/client.py:419-482` - `_get_conn()` (핵심 로직)
- `aiokafka/client.py:412-417` - `_on_connection_closed()` (콜백)
- `aiokafka/conn.py:130-268` - `AIOKafkaConnection` (개별 연결)

### 설정
- `connections_max_idle_ms`: Idle 연결 타임아웃 (기본 540초)

### 관련 개념
- **Connection Pooling**: 리소스 재사용 패턴
- **Double-check locking**: 동시성 제어 패턴
- **weakref**: 메모리 관리 기법

---

## 요약

| 항목 | 내용 |
|------|------|
| **문제** | TCP 연결 생성 비용이 크고, 여러 브로커와 동시 통신 필요 |
| **선택지** | 매번 새 연결 vs 하나의 연결 vs Pool |
| **최종 선택** | 딕셔너리 기반 Connection Pool |
| **핵심 아이디어** | `(node_id, group)` 키로 연결 관리 + Lock + Idle timeout |
| **얻은 것** | 성능, 동시성, 자동 관리, 확장성 |
| **포기한 것** | 간단함, 메모리, 디버깅 용이성 |
| **적용 가능** | DB, HTTP, gRPC 등 네트워크 통신 라이브러리 |

---

**분석 완료일**: 2025-10-29
**방법론**: "문제 → 고민 → 해결" 중심 분석
