# client.py - Kafka 클라이언트 (Connection Pool & 메타데이터 관리)

> **분석 일자**: 2025-10-29
> **파일**: `aiokafka/client.py` (701 lines)
> **방법론**: 문제 해결 중심 분석 (파일별)

---

## 📋 파일 개요

### 파일 정보
- **경로**: `aiokafka/client.py`
- **줄 수**: 701 lines
- **주요 클래스**: `AIOKafkaClient`

### 핵심 역할

이 파일은 **Kafka 클라이언트의 중심**입니다:
- Connection Pool 관리 (여러 브로커와의 연결)
- 메타데이터 동기화 (토픽, 파티션, 브로커 정보)
- Bootstrap 프로세스 (초기 연결)
- API 버전 협상

**누가 사용하는가?**
- `AIOKafkaProducer`와 `AIOKafkaConsumer`의 기반
- 사용자는 직접 사용하지 않음 (내부 컴포넌트)

---

## 해결하는 핵심 문제들

이 파일은 **4가지 주요 문제**를 해결합니다. 각각을 "문제 → 고민 → 해결" 관점으로 분석합니다.

---

## 1. Connection Pool 관리 - 여러 브로커와 동시 통신

### 문제

Kafka 클러스터는 **여러 브로커**로 구성됩니다:

1. **각 브로커마다 연결 필요**
   - 파티션마다 리더 브로커가 다름
   - Producer: 메시지를 각 파티션의 리더에게 전송
   - Consumer: 여러 파티션에서 동시에 fetch

2. **연결 재사용 필요**
   - 매번 새 연결 생성 시 → 느림 (TCP handshake)
   - 하지만 브로커가 여럿 → 하나의 연결로 부족

3. **동시성 제어**
   - 여러 Task가 동시에 같은 브로커 연결 생성 시도 → 중복 생성
   - 연결이 끊어졌는지 체크 필요

### 고민했던 선택지

#### 선택지 1: 브로커마다 항상 새 연결

```python
async def send(broker_id, request):
    conn = await create_conn(broker.host, broker.port)
    response = await conn.send(request)
    conn.close()
    return response
```

**장점**: 간단
**단점**: 성능 낮음 (매번 TCP handshake)
**왜 안 됨**: 고성능 요구사항 불만족

#### 선택지 2: 전역 Connection Pool (고정 크기)

```python
# 예: 10개 연결을 미리 생성
pool = [create_conn(...) for _ in range(10)]

async def send(broker_id, request):
    conn = pool[hash(broker_id) % 10]
    return await conn.send(request)
```

**장점**: 재사용, 빠름
**단점**:
- 브로커가 적으면 낭비 (10개 연결 but 브로커 3개)
- 브로커가 많으면 부족 (브로커 100개 but 연결 10개)
- 브로커 추가/제거 시 재조정 어려움

**왜 안 됨**: 유연성 부족

#### 선택지 3 (최종): 딕셔너리 기반 Lazy Pool

```python
self._conns = {}  # {(node_id, group): AIOKafkaConnection}

async def _get_conn(self, node_id, group=DEFAULT):
    conn_id = (node_id, group)

    # 1. 기존 연결 확인
    if conn_id in self._conns:
        if self._conns[conn_id].connected():
            return self._conns[conn_id]  # 재사용!
        del self._conns[conn_id]  # 끊어진 연결 제거

    # 2. 새 연결 생성 (Lock으로 중복 방지)
    async with self._get_conn_lock:
        if conn_id in self._conns:  # Double-check
            return self._conns[conn_id]

        broker = self.cluster.broker_metadata(node_id)
        self._conns[conn_id] = await create_conn(
            broker.host, broker.port,
            ...
        )

    return self._conns[conn_id]
```

**장점**:
- ✅ **Lazy 생성**: 필요한 연결만 생성 (리소스 절약)
- ✅ **자동 확장**: 브로커 추가 시 자동으로 연결 추가
- ✅ **재사용**: 연결을 딕셔너리에 캐싱
- ✅ **그룹 분리**: DEFAULT vs COORDINATION 연결 분리

**단점**:
- ❌ Lock 필요 (동시성 제어 복잡도)
- ❌ 연결 상태 관리 필요

**왜 선택했는가**: 유연성 + 성능의 균형

### 최종 해결책

#### 구조

```python
class ConnectionGroup(IntEnum):
    DEFAULT = 0       # 일반 메시지 전송
    COORDINATION = 1  # Consumer group 코디네이터

class AIOKafkaClient:
    def __init__(self, ...):
        self._conns = {}  # {(node_id, group): Connection}
        self._get_conn_lock = asyncio.Lock()

    async def _get_conn(self, node_id, *, group=ConnectionGroup.DEFAULT):
        """연결 가져오기 or 생성"""
        conn_id = (node_id, group)

        # 1. 기존 연결 확인
        if conn_id in self._conns:
            conn = self._conns[conn_id]
            if not conn.connected():
                del self._conns[conn_id]  # 끊어진 연결 제거
            else:
                return conn  # 재사용!

        # 2. 브로커 정보 조회
        if group == ConnectionGroup.DEFAULT:
            broker = self.cluster.broker_metadata(node_id)
            if broker is None:
                raise StaleMetadata(f"Broker {node_id} not in metadata")
        else:
            broker = self.cluster.coordinator_metadata(node_id)

        # 3. Lock으로 동시 생성 방지
        async with self._get_conn_lock:
            if conn_id in self._conns:  # Double-check
                return self._conns[conn_id]

            # 4. 새 연결 생성
            self._conns[conn_id] = await create_conn(
                broker.host, broker.port,
                client_id=self._client_id,
                on_close=self._on_connection_closed,  # 콜백 등록
                ...
            )

        return self._conns[conn_id]

    def _on_connection_closed(self, conn, reason):
        """연결 종료 시 콜백"""
        if reason in [CloseReason.CONNECTION_BROKEN, CloseReason.CONNECTION_TIMEOUT]:
            # 메타데이터 stale 가능성 → 강제 업데이트
            self.force_metadata_update()

    async def send(self, node_id, request, *, group=DEFAULT):
        """요청 전송 (사용자 API)"""
        if not await self.ready(node_id, group=group):
            raise NodeNotReadyError(...)

        conn = self._conns[(node_id, group)]
        return await conn.send(request)
```

#### 핵심 아이디어

**1. 딕셔너리 키: (node_id, group)**

```python
self._conns = {
    (0, DEFAULT): <Connection to broker 0>,
    (1, DEFAULT): <Connection to broker 1>,
    (1, COORDINATION): <Connection to broker 1 coordinator>,
}
```

**왜 그룹을 분리하나?**
- DEFAULT: 일반 메시지 전송, fetch
- COORDINATION: Consumer group 관리, offset commit

같은 브로커라도 **용도가 다르면 별도 연결** 사용
→ 일반 트래픽과 코디네이터 트래픽 격리

**2. Lazy 생성**

```python
# 미리 생성 X
# 필요할 때만 생성
conn = await self._get_conn(broker_id)
```

**이점**:
- 브로커 3개만 사용 → 연결 3개만 생성
- 브로커 100개 중 10개만 사용 → 연결 10개만 생성

**3. Double-check locking**

```python
async with self._get_conn_lock:
    if conn_id in self._conns:  # 다시 확인!
        return self._conns[conn_id]
    # 생성...
```

**왜 필요?**
```
Task A: conn 없음 → Lock 대기
Task B: conn 없음 → Lock 대기
Task A: Lock 획득 → 생성 → 저장
Task B: Lock 획득 → (다시 확인 안 하면) 또 생성!
```

**4. 연결 종료 콜백**

```python
on_close=self._on_connection_closed
```

**왜?**
- 연결이 끊어지면 메타데이터가 오래됐을 가능성
- 브로커 장애, 네트워크 분리 등
- 조기 감지 → 메타데이터 갱신

---

## 2. 메타데이터 동기화 - 백그라운드 Task

### 문제

Kafka 클러스터는 **동적**입니다:

1. **변경사항 발생**
   - 새 토픽 생성
   - 파티션 추가
   - 리더 변경 (failover)
   - 브로커 추가/제거

2. **오래된 메타데이터 사용 시 문제**
   - 존재하지 않는 브로커에 요청 → 실패
   - 리더가 바뀌었는데 옛 리더에게 전송 → NOT_LEADER_FOR_PARTITION
   - 새 파티션을 몰라서 사용하지 못함

3. **언제 갱신할까?**
   - 매 요청마다? → 오버헤드
   - 에러 발생 시에만? → 느린 감지
   - 주기적으로? → 적절한 주기는?

### 고민했던 선택지

#### 선택지 1: 매 요청마다 메타데이터 조회

```python
async def send(broker_id, request):
    metadata = await fetch_metadata()  # 매번!
    broker = metadata.leader_for_partition(...)
    conn = await get_conn(broker.id)
    return await conn.send(request)
```

**장점**: 항상 최신 정보
**단점**: 메타데이터 요청이 너무 많음 (성능 저하)
**왜 안 됨**: 오버헤드

#### 선택지 2: 에러 발생 시에만 갱신

```python
async def send(broker_id, request):
    try:
        return await conn.send(request)
    except NOT_LEADER_ERROR:
        await refresh_metadata()  # 에러 후 갱신
        # 재시도...
```

**장점**: 오버헤드 적음
**단점**:
- 첫 요청은 실패함 (사용자 경험 나쁨)
- 브로커 장애 시 감지 느림

**왜 안 됨**: Reactive (반응적) → 문제 발생 후 대처

#### 선택지 3 (최종): 백그라운드 주기적 갱신 + 필요 시 강제 갱신

```python
# 백그라운드 Task (5분마다 자동 갱신)
async def _md_synchronizer(self):
    while True:
        await asyncio.wait([self._md_update_waiter], timeout=300)  # 5분
        await self._metadata_update(...)
        self._md_update_waiter = create_future()

# 필요 시 강제 갱신 (에러 발생 시 등)
def force_metadata_update(self):
    if not self._md_update_waiter.done():
        self._md_update_waiter.set_result(None)  # Wake up!
```

**장점**:
- ✅ **Proactive**: 문제 발생 전에 갱신
- ✅ **주기적**: 최대 5분 간격으로 자동 갱신
- ✅ **즉시 갱신**: 필요 시 즉시 갱신 가능

**단점**:
- ❌ 백그라운드 Task 관리 복잡도

**왜 선택했는가**: Proactive + Reactive 조합

### 최종 해결책

#### 구조

```python
class AIOKafkaClient:
    def __init__(self, ..., metadata_max_age_ms=300000):  # 5분
        self._metadata_max_age_ms = metadata_max_age_ms
        self._sync_task = None
        self._md_update_waiter = create_future()
        self._md_update_fut = None
        self._topics = set()

    async def bootstrap(self):
        """초기 연결 및 메타데이터 로드"""
        # ...bootstrap 로직...

        # 백그라운드 동기화 Task 시작
        if self._sync_task is None:
            self._sync_task = create_task(self._md_synchronizer())

    async def _md_synchronizer(self):
        """메타데이터 동기화 루프 (백그라운드)"""
        while True:
            # 1. Timeout (5분) 또는 Wake up 대기
            await asyncio.wait(
                [self._md_update_waiter],
                timeout=self._metadata_max_age_ms / 1000
            )

            # 2. 메타데이터 갱신
            topics = self._topics
            if self._md_update_fut is None:
                self._md_update_fut = create_future()

            ret = await self._metadata_update(self.cluster, topics)

            # 3. 토픽 리스트 변경 시 즉시 재갱신
            if topics != self._topics:
                continue  # 다시 루프

            # 4. Waiter 리셋
            self._md_update_waiter = create_future()

            # 5. 대기 중인 요청들에게 알림
            self._md_update_fut.set_result(ret)
            self._md_update_fut = None

    async def _metadata_update(self, cluster_metadata, topics):
        """실제 메타데이터 조회"""
        # MetadataRequest 구성
        version_id = 0 if self.api_version < (0, 10) else 1
        if version_id == 1 and not topics:
            topics = None  # 전체 토픽 조회
        metadata_request = MetadataRequest[version_id](topics)

        # 랜덤 브로커 선택
        nodeids = [b.nodeId for b in self.cluster.brokers()]
        random.shuffle(nodeids)

        # 응답 받을 때까지 시도
        for node_id in nodeids:
            conn = await self._get_conn(node_id)
            if conn is None:
                continue

            try:
                metadata = await conn.send(metadata_request)
            except (KafkaError, asyncio.TimeoutError):
                continue

            if not metadata.brokers:
                return False

            # 클러스터 메타데이터 업데이트
            cluster_metadata.update_metadata(metadata)
            return True

        # 모든 브로커 실패
        cluster_metadata.failed_update(None)
        return False

    def force_metadata_update(self):
        """즉시 갱신 (에러 발생 시 등)"""
        if self._md_update_fut is None:
            # Wake up _md_synchronizer
            if not self._md_update_waiter.done():
                self._md_update_waiter.set_result(None)
            self._md_update_fut = self._loop.create_future()

        # 백그라운드에서 갱신 완료 대기
        return asyncio.shield(self._md_update_fut)

    def add_topic(self, topic):
        """토픽 추가 → 메타데이터 갱신"""
        if topic in self._topics:
            res = create_future()
            res.set_result(True)
        else:
            res = self.force_metadata_update()  # 즉시 갱신

        self._topics.add(topic)
        return res
```

#### 핵심 아이디어

**1. Wake-up 패턴**

```python
# 기다리기
await asyncio.wait([self._md_update_waiter], timeout=300)

# 깨우기 (다른 곳에서)
if not self._md_update_waiter.done():
    self._md_update_waiter.set_result(None)
```

**왜 이렇게?**
- 주기적 갱신: timeout (5분)
- 즉시 갱신: set_result() → 바로 깨어남
- 하나의 루프로 두 가지 케이스 처리

**2. asyncio.shield()**

```python
return asyncio.shield(self._md_update_fut)
```

**왜?**
- 여러 Task가 동시에 갱신 요청
- 실제로는 한 번만 갱신
- 취소되어도 갱신은 계속 진행

**3. 랜덤 브로커 선택**

```python
nodeids = [b.nodeId for b in self.cluster.brokers()]
random.shuffle(nodeids)
```

**왜 랜덤?**
- 부하 분산 (항상 첫 번째 브로커에만 요청 X)
- 한 브로커 장애 시에도 다른 브로커 시도

**4. 토픽 리스트 변경 감지**

```python
topics = self._topics
# ...갱신...
if topics != self._topics:
    continue  # 즉시 재갱신
```

**왜?**
- 갱신 중에 새 토픽 추가될 수 있음
- 오래된 정보로 갱신되는 것 방지

---

## 3. Bootstrap 프로세스 - 초기 연결

### 문제

클라이언트는 **전체 클러스터 정보**를 모릅니다:

1. **처음에는 아무것도 모름**
   - 브로커 IP 목록만 있음 (bootstrap_servers)
   - 어떤 토픽이 있는지 모름
   - 어느 브로커가 살아있는지 모름

2. **Bootstrap servers 중 일부는 죽어있을 수 있음**
   - 예: `bootstrap_servers=['broker1:9092', 'broker2:9092', 'broker3:9092']`
   - broker1 죽음, broker2, 3 살아있음
   - 어떤 서버가 살아있는지 모름

3. **초기 메타데이터 로드 필요**
   - 메타데이터를 한 번만 가져오면 전체 클러스터 정보 획득
   - 하지만 어느 브로커에서 가져올까?

### 고민했던 선택지

#### 선택지 1: 첫 번째 서버만 시도

```python
host, port = bootstrap_servers[0]
conn = await create_conn(host, port)
metadata = await conn.send(MetadataRequest())
```

**장점**: 간단
**단점**: 첫 서버 죽으면 실패
**왜 안 됨**: 가용성 낮음

#### 선택지 2: 모든 서버에 동시 연결

```python
tasks = [create_conn(host, port) for host, port in bootstrap_servers]
results = await asyncio.gather(*tasks, return_exceptions=True)
conn = [r for r in results if not isinstance(r, Exception)][0]
```

**장점**: 빠름
**단점**: 리소스 낭비 (불필요한 연결 생성)
**왜 안 됨**: 오버헤드

#### 선택지 3 (최종): 순차적 시도 + 첫 성공 사용

```python
for host, port in bootstrap_servers:
    try:
        conn = await create_conn(host, port)
        metadata = await conn.send(MetadataRequest())
        break  # 성공!
    except:
        continue  # 다음 서버 시도

if not metadata:
    raise KafkaConnectionError("Bootstrap failed")
```

**장점**:
- ✅ 한 서버만 살아있어도 OK
- ✅ 리소스 절약 (필요한 만큼만 연결)
- ✅ 순서 보장 (선호 서버 먼저)

**단점**:
- ❌ 순차적 → 느릴 수 있음

**왜 선택했는가**: 단순함 + 충분한 성능

### 최종 해결책

```python
async def bootstrap(self):
    """초기 클러스터 메타데이터 로드"""
    # MetadataRequest 버전 선택
    if self._api_version == "auto" or self._api_version < (0, 10):
        metadata_request = MetadataRequest[0]([])
    else:
        metadata_request = MetadataRequest[1]([])

    # Bootstrap servers 순차 시도
    for host, port, _ in self.hosts:
        log.debug("Attempting to bootstrap via %s:%s", host, port)

        # 1. 연결 시도
        try:
            bootstrap_conn = await create_conn(
                host, port,
                client_id=self._client_id,
                ...
            )
        except (OSError, asyncio.TimeoutError) as err:
            log.error("Unable to connect to %s:%s: %s", host, port, err)
            continue  # 다음 서버

        # 2. 메타데이터 요청
        try:
            metadata = await bootstrap_conn.send(metadata_request)
        except (KafkaError, asyncio.TimeoutError) as err:
            log.warning("Unable to get metadata from %s:%s", host, port)
            bootstrap_conn.close()
            continue

        # 3. 성공! 메타데이터 업데이트
        self.cluster.update_metadata(metadata)

        # 4. Bootstrap 연결 처리
        if not len(self.cluster.brokers()):
            # 토픽 없는 새 클러스터 → bootstrap 연결 유지
            bootstrap_id = ("bootstrap", ConnectionGroup.DEFAULT)
            self._conns[bootstrap_id] = bootstrap_conn
        else:
            # 정상 클러스터 → 연결 종료 (나중에 재연결)
            bootstrap_conn.close()

        log.debug("Received cluster metadata: %s", self.cluster)
        break
    else:
        # 모든 서버 실패
        raise KafkaConnectionError(f"Unable to bootstrap from {self.hosts}")

    # 5. API 버전 협상 (auto 모드)
    if self._api_version == "auto":
        self._api_version = await self.check_version()

    # 6. 백그라운드 동기화 시작
    if self._sync_task is None:
        self._sync_task = create_task(self._md_synchronizer())
```

#### 핵심 아이디어

**1. Bootstrap 연결 특별 처리**

```python
if not len(self.cluster.brokers()):
    # 빈 클러스터 → 연결 유지
    self._conns[("bootstrap", DEFAULT)] = bootstrap_conn
else:
    # 정상 클러스터 → 종료
    bootstrap_conn.close()
```

**왜?**
- 새 클러스터 (토픽 없음) → broker 메타데이터도 없음
- Bootstrap 연결 유지 → 나중에 메타데이터 재조회
- 토픽 생성 후 → 정상 브로커 정보 획득 → bootstrap 연결 제거

**2. for-else 구문**

```python
for host, port in servers:
    try:
        # 성공 시 break
        break
    except:
        continue
else:
    # break 안 됨 = 모두 실패
    raise Error()
```

Python의 `for-else`: break 안 되면 else 실행

---

## 4. API 버전 체크 - 브로커 버전 확인

### 문제

브로커 버전을 **자동으로** 감지해야 합니다:

1. **사용자는 버전을 모름**
   - 클러스터 관리자만 브로커 버전 알 수 있음
   - 사용자가 매번 설정하기 번거로움

2. **버전마다 기능 다름**
   - Kafka 0.9: 기본 기능
   - Kafka 0.10+: Timestamp, SASL 개선
   - Kafka 1.0+: Exactly-once semantics
   - Kafka 2.0+: ZooKeeper 독립

3. **잘못된 버전 사용 시 에러**
   - 최신 API → 구버전 브로커 = UnsupportedVersionError
   - 구버전 API → 최신 브로커 = 기능 활용 못함

### 고민했던 선택지

#### 선택지 1: 사용자가 수동 설정

```python
client = AIOKafkaClient(api_version=(2, 0, 0))
```

**장점**: 간단
**단점**: 사용자가 버전 알아야 함 (UX 나쁨)
**왜 안 됨**: 자동화 필요

#### 선택지 2: 요청 보내고 에러로 판단

```python
try:
    await conn.send(NewRequest())
    version = "new"
except UnsupportedVersionError:
    version = "old"
```

**장점**: 자동
**단점**: 시행착오 방식 (느림, 에러 발생)
**왜 안 됨**: 우아하지 않음

#### 선택지 3 (최종): ApiVersionRequest로 자동 감지

```python
# Kafka 0.10+에서 지원
response = await conn.send(ApiVersionRequest())

# 각 API의 지원 버전 확인
for api_key, min_ver, max_ver in response.api_versions:
    ...

# 가장 높은 버전 추론
version = infer_version_from_apis(response)
```

**장점**:
- ✅ 정확함 (직접 조회)
- ✅ 한 번만 조회
- ✅ 모든 API 버전 정보 획득

**단점**:
- ❌ Kafka 0.9는 지원 안 함 (ApiVersionRequest 없음)

**왜 선택했는가**: Kafka 0.10+ 대부분 사용

### 최종 해결책

```python
async def check_version(self, node_id=None):
    """브로커 버전 자동 감지"""
    if node_id is None:
        # 랜덤 브로커 선택
        node_id = self.get_random_node()

    # 1. ApiVersionRequest 전송
    from aiokafka.protocol.admin import ApiVersionRequest
    request = ApiVersionRequest[0]()

    try:
        response = await self.send(node_id, request)
    except Errors.KafkaError as err:
        # ApiVersionRequest 지원 안 함 (Kafka 0.9)
        raise UnrecognizedBrokerVersion() from err

    # 2. 응답 검증
    error_type = Errors.for_code(response.error_code)
    if error_type is not Errors.NoError:
        raise error_type()

    # 3. API 지원 버전 확인
    self._check_api_version_response(response)

    # 4. 브로커 버전 추론
    # FetchRequest v3 지원 → Kafka 0.10.1+
    # ProduceRequest v3 지원 → Kafka 0.11+
    # 등등...

    if any(api.api_key == 1 and api.max_version >= 4
           for api in response.api_versions):
        return (0, 11, 0)  # Kafka 0.11+
    elif any(api.api_key == 1 and api.max_version >= 3
             for api in response.api_versions):
        return (0, 10, 1)  # Kafka 0.10.1+
    else:
        return (0, 10, 0)  # Kafka 0.10.0
```

#### 핵심 아이디어

**특정 API 버전으로 Kafka 버전 추론**

```
FetchRequest (api_key=1):
  - v0: Kafka 0.8
  - v1-2: Kafka 0.9
  - v3: Kafka 0.10.1
  - v4+: Kafka 0.11+

ProduceRequest (api_key=0):
  - v0-1: Kafka 0.8-0.9
  - v2: Kafka 0.10
  - v3+: Kafka 0.11+ (idempotence)
```

각 버전의 "시그니처" API 확인 → Kafka 버전 추론

---

## 전체 초기화 흐름

```
User: producer = AIOKafkaProducer(...)
      await producer.start()
  ↓
Client: await client.bootstrap()
  ├─ 1. Bootstrap servers 순차 시도
  │    ├─ create_conn(broker1) → 실패
  │    └─ create_conn(broker2) → 성공!
  ├─ 2. MetadataRequest 전송
  ├─ 3. cluster.update_metadata(response)
  ├─ 4. API 버전 자동 감지 (api_version="auto")
  │    └─ ApiVersionRequest → Kafka 0.11 감지
  └─ 5. 백그라운드 동기화 Task 시작
       └─ _md_synchronizer() 루프 시작 (5분마다 갱신)

이후 사용:
  User: await producer.send('topic', b'msg')
    ↓
  Client: await client.send(broker_id, request)
    ├─ 1. _get_conn(broker_id) → Connection Pool에서 가져오기
    ├─ 2. conn.send(request)
    └─ 3. 응답 반환
```

---

## 주요 클래스/메서드 참고

### AIOKafkaClient

| 메서드 | 역할 | 반환 |
|--------|------|------|
| `bootstrap()` | 초기 연결 및 메타데이터 로드 | `None` |
| `_get_conn(node_id, group)` | Connection Pool에서 연결 가져오기/생성 | `AIOKafkaConnection` |
| `send(node_id, request)` | 요청 전송 | `Future[Response]` |
| `force_metadata_update()` | 즉시 메타데이터 갱신 | `Future` |
| `add_topic(topic)` | 토픽 추가 및 메타데이터 갱신 | `Future` |
| `check_version()` | API 버전 자동 감지 | `(major, minor, patch)` |
| `_md_synchronizer()` | 백그라운드 메타데이터 동기화 루프 | `None` |

### 설정

| 설정 | 기본값 | 설명 |
|------|--------|------|
| `bootstrap_servers` | "localhost" | 초기 브로커 목록 |
| `metadata_max_age_ms` | 300000 | 메타데이터 갱신 주기 (5분) |
| `request_timeout_ms` | 40000 | 요청 타임아웃 (40초) |
| `connections_max_idle_ms` | 540000 | Idle 연결 타임아웃 (9분) |
| `api_version` | "auto" | API 버전 (auto 또는 튜플) |

---

## 배운 점

### 1. Lazy Resource Management

**Connection Pool을 Lazy하게 관리:**

```python
# 미리 생성 X
_conns = {}

# 필요할 때만 생성
if conn_id not in _conns:
    _conns[conn_id] = await create_conn(...)
```

**언제 사용?**
- 리소스가 비쌀 때 (TCP 연결, DB 연결)
- 사용 패턴을 예측할 수 없을 때
- 최대 개수가 동적일 때

### 2. Wake-up 패턴 (Future + timeout)

```python
# 대기
await asyncio.wait([waiter], timeout=300)

# 깨우기
waiter.set_result(None)
```

**용도**:
- 주기적 작업 + 즉시 실행 기능
- 백그라운드 Task 제어
- Graceful shutdown

### 3. Double-check Locking (asyncio)

```python
if resource not in pool:  # Check 1
    async with lock:
        if resource not in pool:  # Check 2 (Double-check!)
            pool[resource] = await create()
```

**왜 필요?**
- 여러 Task가 동시에 Lock 대기
- Lock 획득 전에 다른 Task가 이미 생성했을 수 있음

### 4. asyncio.shield()

```python
fut = asyncio.shield(background_task)
```

**의미**: Task 취소되어도 background_task는 계속 실행

**언제 사용?**
- 취소해도 완료해야 하는 작업 (DB commit, 파일 쓰기)
- 여러 호출자가 공유하는 작업

### 5. for-else 구문

```python
for item in items:
    if condition(item):
        break
else:
    # break 안 됨 = 조건 만족하는 item 없음
    handle_failure()
```

Python 특유의 구문 → 모든 반복 실패 처리에 유용

### 6. Bootstrap 연결의 특별 처리

```python
if not cluster.brokers():
    # 빈 클러스터 → 연결 유지
    _conns["bootstrap"] = conn
else:
    # 정상 → 연결 종료
    conn.close()
```

**왜?**
- 초기 상태 vs 정상 상태의 차이 인식
- 단계별로 다른 전략 사용

### 7. Proactive + Reactive 조합

**Proactive**: 주기적 갱신 (5분마다)
**Reactive**: 에러 발생 시 즉시 갱신

```python
# Proactive
await asyncio.wait([waiter], timeout=300)
await refresh()

# Reactive
except NOT_LEADER_ERROR:
    force_metadata_update()
```

**이점**: 대부분은 Proactive로 처리, 예외 상황만 Reactive

### 8. 비슷한 상황에 적용

| 패턴 | 적용 가능한 곳 |
|------|----------------|
| Lazy Connection Pool | 데이터베이스, gRPC, Redis |
| 백그라운드 동기화 | 캐시 갱신, 설정 리로드, 헬스체크 |
| Wake-up 패턴 | 스케줄러, Job queue |
| Double-check locking | Singleton, 리소스 초기화 |
| Bootstrap 로직 | 분산 시스템 초기화 |

---

## 실전 적용 가이드

이 섹션은 **실제 프로젝트에 패턴을 적용**할 때 도움이 됩니다.

### 가이드 1: Lazy Connection Pool 구현하기

**상황**: 여러 서버와 통신하는 클라이언트 구현 (DB, gRPC, Redis 등)

#### Step 1: 요구사항 정의

```markdown
질문:
- [ ] 서버 개수가 동적으로 변하는가?
- [ ] 연결이 끊어질 수 있는가?
- [ ] 동시에 여러 요청을 보내는가?
- [ ] 연결 생성 비용이 큰가?
```

#### Step 2: Pool 설계

```python
from collections import defaultdict
import asyncio

class ConnectionPool:
    def __init__(self, create_connection_fn):
        self._conns = {}  # {node_id: Connection}
        self._lock = asyncio.Lock()
        self._create_connection = create_connection_fn

    async def get_connection(self, node_id):
        """연결 가져오기 or 생성 (Lazy)"""
        # 1. 기존 연결 확인
        if node_id in self._conns:
            conn = self._conns[node_id]
            if conn.is_connected():
                return conn  # 재사용!
            else:
                del self._conns[node_id]  # 끊어진 연결 제거

        # 2. Lock으로 동시 생성 방지
        async with self._lock:
            # Double-check pattern
            if node_id in self._conns:
                return self._conns[node_id]

            # 3. 새 연결 생성
            conn = await self._create_connection(node_id)
            self._conns[node_id] = conn
            return conn

    async def close_all(self):
        """모든 연결 종료"""
        async with self._lock:
            conns = list(self._conns.values())
            self._conns.clear()

        await asyncio.gather(
            *[conn.close() for conn in conns],
            return_exceptions=True
        )

# 사용 예시
async def create_db_connection(node_id):
    host, port = get_server_info(node_id)
    return await asyncpg.connect(host=host, port=port)

pool = ConnectionPool(create_db_connection)
conn = await pool.get_connection(server_id)  # Lazy 생성
result = await conn.query("SELECT * FROM users")
```

#### Step 3: 의사결정 체크리스트

**Pool 크기 제한 필요?**
- YES → `Semaphore` 추가
- NO → 무제한 (리소스 주의)

```python
class LimitedConnectionPool(ConnectionPool):
    def __init__(self, create_connection_fn, max_size=100):
        super().__init__(create_connection_fn)
        self._semaphore = asyncio.Semaphore(max_size)

    async def get_connection(self, node_id):
        await self._semaphore.acquire()
        try:
            return await super().get_connection(node_id)
        except:
            self._semaphore.release()
            raise
```

**Idle timeout 필요?**
- YES → 각 연결에 idle check 추가
- NO → 메모리 사용 지속

---

### 가이드 2: 백그라운드 동기화 Task 구현하기

**상황**: 주기적으로 갱신해야 하는 데이터 (설정, 메타데이터, 캐시 등)

#### Step 1: Wake-up 패턴 이해하기

```python
class BackgroundSync:
    def __init__(self, refresh_interval=300):  # 5분
        self._interval = refresh_interval
        self._waiter = None
        self._sync_task = None
        self._data = None

    async def start(self):
        """백그라운드 Task 시작"""
        if self._sync_task is None:
            self._sync_task = asyncio.create_task(self._sync_loop())

    async def _sync_loop(self):
        """주기적 동기화 루프"""
        while True:
            # Wake-up 패턴: timeout 또는 즉시 깨우기 대기
            self._waiter = asyncio.Future()

            try:
                await asyncio.wait_for(
                    self._waiter,
                    timeout=self._interval
                )
            except asyncio.TimeoutError:
                pass  # 정상 timeout

            # 데이터 갱신
            try:
                self._data = await self._fetch_data()
                print(f"Data refreshed: {self._data}")
            except Exception as e:
                print(f"Refresh failed: {e}")

    async def _fetch_data(self):
        """실제 데이터 조회 (오버라이드 필요)"""
        raise NotImplementedError

    def force_refresh(self):
        """즉시 갱신 (Wake up!)"""
        if self._waiter and not self._waiter.done():
            self._waiter.set_result(None)

    async def stop(self):
        """백그라운드 Task 종료"""
        if self._sync_task:
            self._sync_task.cancel()
            try:
                await self._sync_task
            except asyncio.CancelledError:
                pass

# 사용 예시
class ConfigSync(BackgroundSync):
    async def _fetch_data(self):
        # 실제 설정 조회
        return await fetch_config_from_server()

sync = ConfigSync(refresh_interval=60)  # 1분마다
await sync.start()

# 나중에 즉시 갱신 필요 시
sync.force_refresh()
```

#### Step 2: Proactive + Reactive 조합

```python
class SmartSync(BackgroundSync):
    async def get_data(self):
        """데이터 조회 (캐시 반환)"""
        if self._data is None:
            # 아직 초기화 안 됨 → 즉시 갱신
            await self._fetch_and_update()
        return self._data

    async def _fetch_and_update(self):
        """갱신 및 캐시 업데이트"""
        self._data = await self._fetch_data()

    async def on_error(self, error):
        """에러 발생 시 → Reactive 갱신"""
        if isinstance(error, StaleDataError):
            self.force_refresh()

# 사용
sync = SmartSync()
await sync.start()

try:
    result = await use_data(sync.get_data())
except StaleDataError as e:
    await sync.on_error(e)  # Reactive
```

---

### 가이드 3: Bootstrap 프로세스 구현하기

**상황**: 분산 시스템 초기화 (클러스터, 서비스 디스커버리)

#### Step 1: 순차 시도 패턴

```python
class Bootstrapper:
    def __init__(self, seed_servers):
        self.seed_servers = seed_servers  # [(host, port), ...]
        self.cluster_info = None

    async def bootstrap(self):
        """여러 서버 중 하나라도 성공하면 OK"""
        errors = []

        for host, port in self.seed_servers:
            try:
                # 1. 연결 시도
                conn = await asyncio.wait_for(
                    self._connect(host, port),
                    timeout=10
                )

                # 2. 초기 정보 조회
                self.cluster_info = await conn.get_cluster_info()

                # 3. 성공! (for-else의 break)
                print(f"Bootstrapped from {host}:{port}")
                return True

            except Exception as e:
                errors.append((host, port, e))
                continue  # 다음 서버 시도

        # for-else: 모든 서버 실패
        raise BootstrapError(f"All servers failed: {errors}")

    async def _connect(self, host, port):
        """연결 생성 (오버라이드 필요)"""
        raise NotImplementedError

# 사용
bootstrapper = Bootstrapper([
    ("server1.example.com", 9092),
    ("server2.example.com", 9092),
    ("server3.example.com", 9092),
])

try:
    await bootstrapper.bootstrap()
except BootstrapError as e:
    print(f"Bootstrap failed: {e}")
```

#### Step 2: 병렬 시도 (빠른 부팅)

```python
async def bootstrap_parallel(seed_servers, timeout=10):
    """모든 서버에 동시 연결 시도"""
    tasks = [
        asyncio.create_task(_try_bootstrap(host, port))
        for host, port in seed_servers
    ]

    done, pending = await asyncio.wait(
        tasks,
        timeout=timeout,
        return_when=asyncio.FIRST_COMPLETED  # 첫 성공만
    )

    # 나머지 취소
    for task in pending:
        task.cancel()

    # 성공한 결과 반환
    for task in done:
        try:
            return task.result()  # 첫 성공 결과
        except Exception:
            continue

    raise BootstrapError("All servers failed")

async def _try_bootstrap(host, port):
    conn = await connect(host, port)
    info = await conn.get_cluster_info()
    return info
```

**트레이드오프:**
- 순차: 리소스 절약, 느림
- 병렬: 빠름, 리소스 많이 사용

---

### 가이드 4: API 버전 자동 감지 구현하기

**상황**: 서버 버전에 따라 다른 API 사용 (HTTP, gRPC, DB)

#### Step 1: Capability Discovery

```python
class VersionNegotiator:
    def __init__(self, conn):
        self.conn = conn
        self.server_version = None
        self.api_versions = {}  # {api_name: (min, max)}

    async def negotiate(self):
        """서버 버전 자동 감지"""
        try:
            # 1. 버전 정보 요청
            response = await self.conn.send(GetVersionRequest())

            # 2. 지원 API 목록 파싱
            for api in response.supported_apis:
                self.api_versions[api.name] = (api.min_version, api.max_version)

            # 3. 버전 추론
            self.server_version = self._infer_version()

            return self.server_version

        except UnsupportedRequest:
            # 구버전 서버 (버전 정보 API 없음)
            return self._fallback_version()

    def _infer_version(self):
        """API 지원 여부로 버전 추론"""
        # 예: FetchAPI v4 지원 → Server 2.0+
        if "Fetch" in self.api_versions:
            min_v, max_v = self.api_versions["Fetch"]
            if max_v >= 4:
                return (2, 0)
            elif max_v >= 3:
                return (1, 5)

        return (1, 0)  # 기본값

    def _fallback_version(self):
        """버전 정보 API 없는 구버전"""
        return (0, 9)

    def pick_api_version(self, api_name, preferred_versions):
        """최적 API 버전 선택"""
        if api_name not in self.api_versions:
            return preferred_versions[0]  # 최소 버전

        min_v, max_v = self.api_versions[api_name]

        # 역순으로 탐색 (높은 버전 우선)
        for version in reversed(preferred_versions):
            if min_v <= version <= max_v:
                return version

        raise UnsupportedAPIError(f"{api_name} not supported")

# 사용
negotiator = VersionNegotiator(conn)
version = await negotiator.negotiate()

# API 버전 선택
fetch_version = negotiator.pick_api_version("Fetch", [1, 2, 3, 4])
request = FetchRequest[fetch_version](...)
```

#### Step 2: Feature Flags

```python
class FeatureDetector:
    def __init__(self, server_version):
        self.version = server_version

    def supports_transactions(self):
        return self.version >= (2, 0)

    def supports_compression(self):
        return self.version >= (1, 5)

    def supports_streaming(self):
        return self.version >= (2, 5)

# 사용
detector = FeatureDetector(server_version)

if detector.supports_transactions():
    await client.begin_transaction()
else:
    # 수동 롤백 로직
    pass
```

---

## 안티패턴과 흔한 실수

실제 구현 시 자주 발생하는 문제들과 해결책입니다.

### 실수 1: Pool에서 끊어진 연결 재사용

**❌ 나쁜 예:**
```python
async def get_connection(self, node_id):
    if node_id in self._conns:
        return self._conns[node_id]  # 끊어졌는지 확인 안 함!

    # 새 연결 생성...
```

**문제:**
- 연결이 끊어진 후에도 Pool에 남아있음
- 사용 시 `BrokenPipeError`, `ConnectionResetError` 발생

**✅ 좋은 예:**
```python
async def get_connection(self, node_id):
    if node_id in self._conns:
        conn = self._conns[node_id]

        # 연결 상태 확인!
        if not conn.is_connected():
            del self._conns[node_id]  # 제거
            await conn.close()  # 정리
        else:
            return conn  # 재사용

    # 새 연결 생성...
```

---

### 실수 2: 백그라운드 Task 정리 안 함

**❌ 나쁜 예:**
```python
class Client:
    async def start(self):
        # 백그라운드 Task 시작
        self._sync_task = asyncio.create_task(self._sync_loop())

    async def close(self):
        # Task 정리 안 함!
        pass
```

**문제:**
- Task가 계속 실행됨 (리소스 누수)
- `asyncio.CancelledError` 경고 발생 가능

**✅ 좋은 예:**
```python
class Client:
    async def start(self):
        self._sync_task = asyncio.create_task(self._sync_loop())
        self._closing = False

    async def close(self):
        # 1. 종료 플래그 설정
        self._closing = True

        # 2. Task 취소
        if self._sync_task:
            self._sync_task.cancel()

            try:
                await self._sync_task
            except asyncio.CancelledError:
                pass  # 정상 취소

    async def _sync_loop(self):
        while not self._closing:  # 종료 플래그 확인
            try:
                await self._do_sync()
                await asyncio.sleep(60)
            except asyncio.CancelledError:
                break  # 정상 종료
```

---

### 실수 3: Bootstrap 실패 시 부분 상태

**❌ 나쁜 예:**
```python
async def bootstrap(self):
    # 1. 연결 생성
    self._bootstrap_conn = await create_conn(host, port)

    # 2. 메타데이터 조회 (실패 가능!)
    metadata = await self._bootstrap_conn.get_metadata()

    # 3. 상태 업데이트
    self._cluster.update(metadata)
```

**문제:**
- 메타데이터 조회 실패 시 → 연결은 생성됐지만 메타데이터 없음
- 부분적으로 초기화된 상태 → 예상치 못한 에러

**✅ 좋은 예:**
```python
async def bootstrap(self):
    bootstrap_conn = None  # 로컬 변수

    try:
        # 1. 연결 생성
        bootstrap_conn = await create_conn(host, port)

        # 2. 메타데이터 조회
        metadata = await bootstrap_conn.get_metadata()

        # 3. 모두 성공 → 상태 업데이트
        self._bootstrap_conn = bootstrap_conn
        self._cluster.update(metadata)
        self._initialized = True

    except Exception:
        # 실패 시 정리
        if bootstrap_conn:
            await bootstrap_conn.close()
        raise  # 재발생
```

---

### 실수 4: Metadata 갱신 중 읽기

**❌ 나쁜 예:**
```python
async def update_metadata(self, metadata):
    # 업데이트 도중 다른 Task가 읽기 가능!
    self._brokers = metadata.brokers
    await asyncio.sleep(0)  # Yield
    self._topics = metadata.topics  # Inconsistent!
```

**문제:**
- 업데이트 도중 다른 Task가 읽기 → 일관성 깨짐
- `_brokers`는 새 데이터, `_topics`는 옛 데이터

**✅ 좋은 예:**
```python
def update_metadata(self, metadata):
    # 1. 새 데이터 준비 (Lock 밖에서)
    new_brokers = metadata.brokers
    new_topics = metadata.topics

    # 2. Lock 안에서 한 번에 교체
    with self._lock:
        self._brokers = new_brokers
        self._topics = new_topics
        self._version += 1  # 버전 증가
```

---

### 실수 5: Force refresh 중복 호출

**❌ 나쁜 예:**
```python
def force_metadata_update(self):
    # 매번 새 Future 생성!
    self._update_fut = asyncio.Future()
    self._waiter.set_result(None)  # Wake up
    return self._update_fut
```

**문제:**
- 여러 Task가 동시에 호출 → 중복 갱신
- 각자 다른 Future 대기 → 일부만 완료 알림 받음

**✅ 좋은 예:**
```python
def force_metadata_update(self):
    # 이미 갱신 중이면 기존 Future 반환
    if self._update_fut is None or self._update_fut.done():
        self._update_fut = asyncio.Future()

        # Wake up (한 번만)
        if not self._waiter.done():
            self._waiter.set_result(None)

    # 모두 같은 Future 대기
    return asyncio.shield(self._update_fut)
```

---

### 실수 6: 순환 참조로 메모리 누수

**❌ 나쁜 예:**
```python
class Client:
    async def start(self):
        # 백그라운드 Task가 self 참조
        self._sync_task = asyncio.create_task(self._sync_loop())
        #                                      ^^^^^^^^^^^^^^^^
        # Task → self → Task (순환!)
```

**문제:**
- `Client` 삭제해도 `_sync_task`가 참조 유지
- 메모리 누수

**✅ 방법 1: Task 명시적 취소**
```python
async def close(self):
    if self._sync_task:
        self._sync_task.cancel()
        await self._sync_task  # 완전히 종료 대기
        self._sync_task = None  # 참조 제거
```

**✅ 방법 2: weakref 사용**
```python
import weakref

async def start(self):
    self_ref = weakref.ref(self)
    self._sync_task = asyncio.create_task(
        self._sync_loop_static(self_ref)
    )

@staticmethod
async def _sync_loop_static(self_ref):
    while True:
        self = self_ref()
        if self is None:
            break  # 이미 삭제됨

        await self._do_sync()
        await asyncio.sleep(60)
```

---

### 실수 7: asyncio.wait() 결과 처리 안 함

**❌ 나쁜 예:**
```python
await asyncio.wait([self._waiter], timeout=300)
# 결과 무시!
```

**문제:**
- `wait()`는 완료/미완료 Task 세트 반환
- 예외 발생해도 모름

**✅ 좋은 예:**
```python
done, pending = await asyncio.wait(
    [self._waiter],
    timeout=300
)

# 완료된 Task 결과 확인
for task in done:
    try:
        task.result()  # 예외 재발생
    except Exception as e:
        log.error(f"Waiter error: {e}")

# 미완료 Task 정리
for task in pending:
    task.cancel()
```

---

## 스케일 고려사항

규모별로 다른 전략이 필요합니다.

### 소규모 (연결 < 10, 서버 < 5)

**권장 사항:**
- ✅ 간단한 딕셔너리 Pool
- ✅ 주기적 메타데이터 갱신 (5분)
- ✅ 순차 Bootstrap (병렬 불필요)
- ⚠️  Connection Pool 없이도 고려 가능

**구현 예시:**
```python
# 간단한 클라이언트
class SimpleClient:
    def __init__(self, servers):
        self.servers = servers
        self._conns = {}  # 간단한 Pool

    async def send(self, server_id, request):
        if server_id not in self._conns:
            # Lazy 생성
            self._conns[server_id] = await self._connect(server_id)

        return await self._conns[server_id].send(request)

# 메타데이터 갱신도 간단하게
async def refresh_metadata_simple():
    while True:
        await asyncio.sleep(300)  # 5분
        try:
            metadata = await fetch_metadata()
            update_metadata(metadata)
        except Exception as e:
            log.warning(f"Metadata refresh failed: {e}")
```

**모니터링:**
- 연결 실패율
- 평균 응답 시간

---

### 중규모 (연결 10-100, 서버 5-50)

**권장 사항:**
- ✅ Lock + Double-check Pool
- ✅ Wake-up 패턴 (Proactive + Reactive)
- ✅ Health check 추가
- ✅ 연결별 메트릭 수집
- ⚠️  Pool 크기 제한 고려

**구현 예시:**
```python
class MediumScaleClient:
    def __init__(self, servers, max_conns_per_server=10):
        self._conns = {}
        self._lock = asyncio.Lock()
        self._max_conns = max_conns_per_server
        self._conn_semaphores = {}  # {server_id: Semaphore}

        # 메트릭
        self._metrics = {
            'total_requests': 0,
            'failed_requests': 0,
            'pool_size': 0,
        }

    async def get_connection(self, server_id):
        # Semaphore로 서버별 최대 연결 수 제한
        if server_id not in self._conn_semaphores:
            self._conn_semaphores[server_id] = asyncio.Semaphore(
                self._max_conns
            )

        await self._conn_semaphores[server_id].acquire()

        try:
            # Double-check lock
            async with self._lock:
                if server_id not in self._conns:
                    conn = await self._create_connection(server_id)
                    self._conns[server_id] = conn
                    self._metrics['pool_size'] += 1

            return self._conns[server_id]
        except:
            self._conn_semaphores[server_id].release()
            raise

    async def health_check(self):
        """주기적 헬스체크"""
        while True:
            await asyncio.sleep(30)  # 30초마다

            dead_servers = []
            for server_id, conn in list(self._conns.items()):
                if not conn.is_connected():
                    dead_servers.append(server_id)

            # 죽은 연결 제거
            async with self._lock:
                for server_id in dead_servers:
                    del self._conns[server_id]
                    self._metrics['pool_size'] -= 1
```

**모니터링:**
- Pool 크기 변화
- Lock 대기 시간
- Health check 실패율
- 서버별 연결 수

---

### 대규모 (연결 100+, 서버 50+)

**권장 사항:**
- ✅ Shard별 Pool (Lock 경합 감소)
- ✅ Circuit Breaker 패턴
- ✅ Connection warming (미리 생성)
- ✅ LRU eviction (오래된 연결 제거)
- ✅ 상세 메트릭 + 알림
- ✅ Graceful degradation

**구현 예시:**
```python
from collections import OrderedDict

class LargeScaleClient:
    def __init__(self, servers, max_conns=1000, shards=10):
        # Shard별 Pool (Lock 경합 분산)
        self._shards = [
            {
                'conns': OrderedDict(),  # LRU용
                'lock': asyncio.Lock(),
            }
            for _ in range(shards)
        ]
        self._max_conns = max_conns
        self._max_per_shard = max_conns // shards

        # Circuit Breaker
        self._circuit_breakers = {}  # {server_id: CircuitBreaker}

        # 메트릭
        self._metrics = {
            'total_conns': 0,
            'active_conns': 0,
            'circuit_breaker_trips': 0,
        }

    def _get_shard(self, server_id):
        """서버 ID로 Shard 선택"""
        shard_idx = hash(server_id) % len(self._shards)
        return self._shards[shard_idx]

    async def get_connection(self, server_id):
        """LRU + Circuit Breaker"""
        shard = self._get_shard(server_id)

        # Circuit Breaker 확인
        breaker = self._get_circuit_breaker(server_id)
        if breaker.is_open():
            raise CircuitBreakerOpen(f"Server {server_id} circuit open")

        async with shard['lock']:
            # LRU: 최근 사용한 연결 앞으로
            if server_id in shard['conns']:
                conn = shard['conns'].pop(server_id)
                shard['conns'][server_id] = conn  # 맨 뒤로

                if conn.is_connected():
                    return conn
                else:
                    del shard['conns'][server_id]

            # Shard 크기 제한
            if len(shard['conns']) >= self._max_per_shard:
                # LRU eviction (맨 앞 = 가장 오래됨)
                old_id, old_conn = shard['conns'].popitem(last=False)
                await old_conn.close()

            # 새 연결 생성
            try:
                conn = await breaker.call(
                    self._create_connection, server_id
                )
                shard['conns'][server_id] = conn
                self._metrics['total_conns'] += 1
                return conn
            except Exception as e:
                self._metrics['circuit_breaker_trips'] += 1
                raise

    def _get_circuit_breaker(self, server_id):
        if server_id not in self._circuit_breakers:
            self._circuit_breakers[server_id] = CircuitBreaker(
                failure_threshold=5,
                timeout=60
            )
        return self._circuit_breakers[server_id]

    async def warmup(self, server_ids):
        """연결 미리 생성 (워밍업)"""
        tasks = [
            self.get_connection(server_id)
            for server_id in server_ids
        ]
        await asyncio.gather(*tasks, return_exceptions=True)

# Circuit Breaker 구현
class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self._failures = 0
        self._threshold = failure_threshold
        self._timeout = timeout
        self._state = "CLOSED"  # CLOSED, OPEN, HALF_OPEN
        self._last_failure = 0

    def is_open(self):
        if self._state == "OPEN":
            # Timeout 지나면 HALF_OPEN
            if time.time() - self._last_failure > self._timeout:
                self._state = "HALF_OPEN"
                return False
            return True
        return False

    async def call(self, func, *args, **kwargs):
        if self.is_open():
            raise CircuitBreakerOpen()

        try:
            result = await func(*args, **kwargs)

            # 성공 → CLOSED
            if self._state == "HALF_OPEN":
                self._state = "CLOSED"
                self._failures = 0

            return result
        except Exception as e:
            # 실패 카운트
            self._failures += 1
            self._last_failure = time.time()

            if self._failures >= self._threshold:
                self._state = "OPEN"

            raise
```

**모니터링 필수:**
```python
# Prometheus 스타일 메트릭
- client_connection_pool_size: Pool 크기
- client_connection_pool_active: 사용 중인 연결
- client_connection_create_duration_seconds: 연결 생성 시간
- client_connection_failures_total: 연결 실패 횟수
- client_circuit_breaker_state: Circuit breaker 상태
- client_request_duration_seconds: 요청 처리 시간
- client_metadata_refresh_duration_seconds: 메타데이터 갱신 시간
```

**알림 설정:**
- Pool 크기 > 80% → 경고
- Circuit breaker OPEN → 즉시 알림
- 메타데이터 갱신 실패 > 3회 → 경고

---

### 병목 지점과 해결책

| 병목 | 증상 | 해결책 |
|------|------|--------|
| Lock 경합 | get_connection() 느림 | Shard별 Pool (10-100 shards) |
| 메타데이터 갱신 느림 | 요청 지연 증가 | 비동기 갱신 (asyncio.shield) |
| Bootstrap 실패 | 시작 불가 | 재시도 로직 + 여러 seed 서버 |
| Pool 무한 증가 | OOM | LRU eviction + 최대 크기 제한 |
| Circuit breaker 과민 | 불필요한 차단 | Threshold 조정 (5→10) |

---

## 요약

| 문제 | 선택지 | 최종 해결 | 트레이드오프 |
|------|--------|-----------|--------------|
| Connection Pool | 항상 새 연결 vs 고정 Pool vs Lazy Pool | Lazy 딕셔너리 Pool | 복잡도 vs 유연성 |
| 메타데이터 동기화 | 매번 vs 에러 시 vs 주기적 | 백그라운드 + 즉시 갱신 | Task 관리 vs Proactive |
| Bootstrap | 첫 서버 vs 모두 vs 순차 시도 | 순차 시도 | 속도 vs 리소스 |
| API 버전 | 수동 설정 vs 시행착오 vs 직접 조회 | ApiVersionRequest | Kafka 0.9 미지원 vs 자동화 |

**핵심 메시지:**
- Connection Pool은 Lazy + 딕셔너리 = 유연성
- 메타데이터는 Proactive(주기적) + Reactive(즉시) 조합
- Bootstrap은 순차 시도 = 단순함 + 충분한 성능
- 모든 리소스는 "필요할 때" 생성 (Lazy)

---

**분석 완료일**: 2025-10-29
**방법론**: "문제 → 고민 → 해결" 중심 파일별 분석
**다음 분석**: cluster.py (메타데이터 저장 및 관리)
