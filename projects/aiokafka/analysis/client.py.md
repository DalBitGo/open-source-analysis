# client.py - Kafka 클러스터 클라이언트 관리

## 📋 파일 개요
- **경로**: `aiokafka/client.py`
- **라인 수**: 702줄
- **주요 역할**: Kafka 클러스터 메타데이터 관리, 연결 풀, API 버전 협상을 담당하는 고수준 클라이언트

## 🎯 핵심 목적
`conn.py`의 저수준 연결 위에 구축되어, **클러스터 메타데이터 자동 동기화**, **노드별 연결 풀 관리**, **API 버전 자동 감지**를 제공하는 **클라이언트 계층의 핵심 모듈**

---

## 🏗️ 주요 클래스 및 구조

### 1. **ConnectionGroup (IntEnum)**
```python
class ConnectionGroup(IntEnum):
    DEFAULT = 0       # 일반 브로커 연결
    COORDINATION = 1  # Group/Transaction Coordinator 연결
```
- **목적**: 연결을 역할별로 구분하여 관리
- **사용처**:
  - `DEFAULT`: Producer, Consumer의 일반 요청
  - `COORDINATION`: Consumer Group, Transaction 관리

---

### 2. **CoordinationType (IntEnum)**
```python
class CoordinationType(IntEnum):
    GROUP = 0        # Consumer Group Coordinator
    TRANSACTION = 1  # Transaction Coordinator
```
- **목적**: Coordinator 조회 시 타입 지정
- **Kafka 버전**: 0.11+ 에서 TRANSACTION 지원

---

### 3. **AIOKafkaClient** ⭐ (메인 클래스)

#### 초기화 파라미터
```python
def __init__(
    self,
    *,
    bootstrap_servers="localhost",         # 초기 연결 서버 (쉼표 구분 문자열 또는 리스트)
    client_id="aiokafka-{version}",       # 클라이언트 식별자
    metadata_max_age_ms=300000,           # 메타데이터 최대 유효 기간 (5분)
    request_timeout_ms=40000,             # 요청 타임아웃 (40초)
    retry_backoff_ms=100,                 # 재시도 백오프 시간
    api_version="auto",                   # API 버전 ("auto" 또는 튜플)
    security_protocol="PLAINTEXT",        # PLAINTEXT, SSL, SASL_PLAINTEXT, SASL_SSL
    connections_max_idle_ms=540000,       # 유휴 연결 타임아웃 (9분)
    ...
)
```

#### 핵심 속성
| 속성 | 타입 | 설명 |
|------|------|------|
| `cluster` | `ClusterMetadata` | 클러스터 메타데이터 (브로커, 토픽, 파티션 정보) |
| `_conns` | `dict` | 연결 풀: `{(node_id, group): AIOKafkaConnection}` |
| `_topics` | `set` | 추적 중인 토픽 목록 |
| `_sync_task` | `asyncio.Task` | 백그라운드 메타데이터 동기화 태스크 |
| `_md_update_fut` | `Future` | 진행 중인 메타데이터 업데이트 Future |
| `_md_update_waiter` | `Future` | 메타데이터 동기화 트리거용 |
| `_api_version` | `tuple` or `"auto"` | 브로커 API 버전 |

---

## 🔄 주요 메서드 및 실행 흐름

### 1. **`bootstrap()` - 클러스터 초기화**
```python
async def bootstrap(self):
    # 1. 부트스트랩 서버 순회하며 연결 시도
    for host, port, _ in self.hosts:
        bootstrap_conn = await create_conn(
            host, port,
            client_id=self._client_id,
            version_hint=version_hint,
            ...
        )

        # 2. 메타데이터 요청
        metadata = await bootstrap_conn.send(MetadataRequest[0]([]))
        self.cluster.update_metadata(metadata)

        # 3. 브로커 정보 없으면 부트스트랩 연결 유지
        if not len(self.cluster.brokers()):
            self._conns[("bootstrap", ConnectionGroup.DEFAULT)] = bootstrap_conn
        else:
            bootstrap_conn.close()
        break
    else:
        raise KafkaConnectionError("Unable to bootstrap")

    # 4. API 버전 자동 감지
    if self._api_version == "auto":
        self._api_version = await self.check_version()

    # 5. 백그라운드 메타데이터 동기화 시작
    if self._sync_task is None:
        self._sync_task = create_task(self._md_synchronizer())
```

**핵심 로직**:
1. `bootstrap_servers` 중 하나와 연결 성공할 때까지 시도
2. 초기 메타데이터 가져와서 `cluster` 업데이트
3. 브로커 목록 없으면 부트스트랩 연결을 임시로 보관
4. `api_version="auto"` 시 자동 버전 감지
5. 백그라운드 메타데이터 동기화 태스크 시작

---

### 2. **`_md_synchronizer()` - 메타데이터 자동 동기화**
```python
async def _md_synchronizer(self):
    while True:
        # 1. metadata_max_age_ms 주기로 대기 (또는 강제 업데이트 요청 시)
        await asyncio.wait(
            [self._md_update_waiter],
            timeout=self._metadata_max_age_ms / 1000,
        )

        # 2. 메타데이터 업데이트
        topics = self._topics
        if self._md_update_fut is None:
            self._md_update_fut = create_future()

        ret = await self._metadata_update(self.cluster, topics)

        # 3. 토픽 목록 변경 시 즉시 재시도
        if topics != self._topics:
            continue

        # 4. 대기 중인 Future 완료
        self._md_update_waiter = create_future()
        self._md_update_fut.set_result(ret)
        self._md_update_fut = None
```

**핵심 패턴**:
- **주기적 업데이트**: `metadata_max_age_ms` (기본 5분) 주기
- **강제 업데이트**: `force_metadata_update()` 호출 시 `_md_update_waiter` 완료
- **토픽 변경 감지**: 업데이트 중 토픽 목록 변경 시 즉시 재시도
- **Future 기반 대기**: 다른 코드에서 `_md_update_fut`를 await하여 완료 대기

---

### 3. **`_metadata_update()` - 메타데이터 갱신**
```python
async def _metadata_update(self, cluster_metadata, topics):
    # 1. API 버전에 따른 요청 생성
    version_id = 0 if self.api_version < (0, 10) else 1
    if version_id == 1 and not topics:
        topics = None  # v1에서 None = 전체 토픽
    metadata_request = MetadataRequest[version_id](topics)

    # 2. 랜덤 노드 선택 (부트스트랩 포함)
    nodeids = [b.nodeId for b in self.cluster.brokers()]
    if ("bootstrap", ConnectionGroup.DEFAULT) in self._conns:
        nodeids.append("bootstrap")
    random.shuffle(nodeids)

    # 3. 성공할 때까지 노드 순회
    for node_id in nodeids:
        conn = await self._get_conn(node_id)
        if conn is None:
            continue

        try:
            metadata = await conn.send(metadata_request)
        except (KafkaError, asyncio.TimeoutError) as err:
            log.warning("Unable to request metadata: %r", err)
            continue

        # 4. 메타데이터 업데이트
        if not metadata.brokers:
            return False
        cluster_metadata.update_metadata(metadata)

        # 5. 부트스트랩 연결 정리
        if ("bootstrap", ConnectionGroup.DEFAULT) in self._conns and len(self.cluster.brokers()):
            conn = self._conns.pop(("bootstrap", ConnectionGroup.DEFAULT))
            conn.close()
        break
    else:
        cluster_metadata.failed_update(None)
        return False
    return True
```

**특징**:
- **랜덤 노드 선택**: 부하 분산
- **재시도 로직**: 실패 시 다음 노드로
- **부트스트랩 정리**: 정상 클러스터 정보 획득 후 부트스트랩 연결 종료

---

### 4. **`force_metadata_update()` - 메타데이터 강제 갱신**
```python
def force_metadata_update(self):
    if self._md_update_fut is None:
        # Wake up the `_md_synchronizer` task
        if not self._md_update_waiter.done():
            self._md_update_waiter.set_result(None)
        self._md_update_fut = self._loop.create_future()

    # Metadata will be updated in background by synchronizer
    return asyncio.shield(self._md_update_fut)
```

**패턴**:
1. `_md_synchronizer`를 깨움 (`_md_update_waiter` 완료)
2. 완료 대기용 Future 생성
3. `asyncio.shield()`로 취소 방지하며 반환

**사용처**:
- 토픽 추가 시 (`add_topic()`)
- 연결 실패 시 (`_on_connection_closed()`)
- 파티션 정보 필요 시 (`_wait_on_metadata()`)

---

### 5. **`_get_conn()` - 연결 획득/생성**
```python
async def _get_conn(self, node_id, *, group=ConnectionGroup.DEFAULT, no_hint=False):
    conn_id = (node_id, group)

    # 1. 기존 연결 재사용
    if conn_id in self._conns:
        conn = self._conns[conn_id]
        if not conn.connected():
            del self._conns[conn_id]
        else:
            return conn

    # 2. 브로커 메타데이터 조회
    if group == ConnectionGroup.DEFAULT:
        broker = self.cluster.broker_metadata(node_id)
        if broker is None:
            raise StaleMetadata(f"Broker id {node_id} not in current metadata")
    else:
        broker = self.cluster.coordinator_metadata(node_id)

    # 3. 락 걸고 연결 생성 (중복 방지)
    async with self._get_conn_lock:
        if conn_id in self._conns:
            return self._conns[conn_id]

        version_hint = self._api_version if self._api_version != "auto" and not no_hint else None

        self._conns[conn_id] = await create_conn(
            broker.host, broker.port,
            client_id=self._client_id,
            on_close=self._on_connection_closed,
            version_hint=version_hint,
            ...
        )
    return self._conns[conn_id]
```

**핵심 로직**:
- **연결 재사용**: 동일 `(node_id, group)`에 대한 연결 풀링
- **락 보호**: 동시 요청 시 중복 연결 방지
- **콜백 등록**: `on_close=self._on_connection_closed`로 재연결 트리거
- **버전 힌트**: `no_hint=True`는 버전 감지 시에만 사용

---

### 6. **`send()` - 요청 전송**
```python
async def send(self, node_id, request, *, group=ConnectionGroup.DEFAULT):
    # 1. 연결 확인
    if not (await self.ready(node_id, group=group)):
        raise NodeNotReadyError(f"node {node_id} not ready")

    # 2. ProduceRequest (acks=0) 특수 처리
    expect_response = True
    if isinstance(request, tuple(ProduceRequest)) and request.required_acks == 0:
        expect_response = False

    # 3. 연결을 통해 요청 전송
    try:
        result = await self._conns[(node_id, group)].send(request, expect_response)
    except asyncio.TimeoutError as exc:
        # 4. 타임아웃 시 연결 종료 (재생성 유도)
        self._conns[(node_id, group)].close(reason=CloseReason.CONNECTION_TIMEOUT)
        raise RequestTimedOutError() from exc
    return result
```

**특징**:
- **ProduceRequest (acks=0)**: 응답 대기 안 함 (fire-and-forget)
- **타임아웃 처리**: 연결 종료 → 다음 요청 시 재생성

---

### 7. **`check_version()` - API 버전 자동 감지**
```python
async def check_version(self, node_id=None):
    test_cases = [
        ((0, 10), ApiVersionRequest_v0()),      # Kafka 0.10+
        ((0, 9), ListGroupsRequest_v0()),       # Kafka 0.9
        ((0, 8, 2), GroupCoordinatorRequest_v0(...)),
        ((0, 8, 1), OffsetFetchRequest_v0(...)),
        ((0, 8, 0), MetadataRequest_v0([])),
    ]

    conn = await self._get_conn(node_id, no_hint=True)
    for version, request in test_cases:
        try:
            if not conn.connected():
                await conn.connect()

            # 1. 테스트 요청 전송 (타임아웃 0.1초)
            task = create_task(conn.send(request))
            await asyncio.wait([task], timeout=0.1)

            # 2. MetadataRequest로 연결 상태 검증
            with contextlib.suppress(KafkaError):
                await conn.send(MetadataRequest_v0([]))

            response = await task
        except KafkaError:
            continue
        else:
            if isinstance(request, ApiVersionRequest_v0):
                # 3. ApiVersionResponse로 정밀 버전 판별
                return self._check_api_version_response(response)
            return version

    raise UnrecognizedBrokerVersion()
```

**프로토콜**:
- **순차 프로빙**: 높은 버전부터 시도
- **연결 재사용 불가**: 실패 시 연결 깨짐 → 재연결 후 재시도
- **정밀 버전 감지**: `ApiVersionResponse`로 세밀한 버전 판별 (2.6, 2.5, ...)

**`_check_api_version_response()`**:
```python
def _check_api_version_response(self, response):
    test_cases = [
        ((2, 6, 0), DescribeClientQuotasRequest_v0),
        ((2, 5, 0), DescribeAclsRequest_v2),
        ((2, 4, 0), ProduceRequest[8]),
        ...
        ((0, 10, 1), MetadataRequest[2]),
    ]

    max_versions = {
        api_key: max_version
        for api_key, _, max_version in response.api_versions
    }

    for broker_version, struct in test_cases:
        if max_versions.get(struct.API_KEY, -1) >= struct.API_VERSION:
            return broker_version

    return (0, 10, 0)  # 최소 0.10
```
- **로직**: 브로커가 지원하는 API 버전으로 Kafka 버전 역추정

---

### 8. **`_wait_on_metadata()` - 토픽 메타데이터 대기**
```python
async def _wait_on_metadata(self, topic):
    # 1. 이미 메타데이터 있으면 즉시 반환
    partitions = self.cluster.partitions_for_topic(topic)
    if partitions is not None:
        return partitions

    # 2. 토픽 추적 목록에 추가
    self.add_topic(topic)

    # 3. 타임아웃까지 재시도
    t0 = time.monotonic()
    while True:
        await self.force_metadata_update()
        partitions = self.cluster.partitions_for_topic(topic)
        if partitions is not None:
            return partitions

        if (time.monotonic() - t0) > (self._request_timeout_ms / 1000):
            raise UnknownTopicOrPartitionError()

        if topic in self.cluster.unauthorized_topics:
            raise Errors.TopicAuthorizationFailedError(topic)

        await asyncio.sleep(self._retry_backoff)
```

**사용처**: Producer, Consumer가 토픽 파티션 정보 필요 시 호출

---

### 9. **`coordinator_lookup()` - Coordinator 조회**
```python
async def coordinator_lookup(self, coordinator_type, coordinator_key):
    node_id = self.get_random_node()

    # 1. API 버전에 따른 요청 생성
    if self.api_version > (0, 11):
        request = FindCoordinatorRequest[1](coordinator_key, coordinator_type)
    else:
        # Group coordination only (Transaction 미지원)
        assert coordinator_type == CoordinationType.GROUP
        request = FindCoordinatorRequest[0](coordinator_key)

    # 2. Coordinator 정보 요청
    resp = await self.send(node_id, request)
    error_type = Errors.for_code(resp.error_code)
    if error_type is not Errors.NoError:
        raise error_type()

    # 3. 클러스터에 Coordinator 정보 추가
    self.cluster.add_coordinator(
        resp.coordinator_id,
        resp.host,
        resp.port,
        rack=None,
        purpose=(coordinator_type, coordinator_key),
    )
    return resp.coordinator_id
```

**사용처**:
- Consumer: Group Coordinator 조회 (오프셋 커밋, 리밸런싱)
- Producer: Transaction Coordinator 조회 (트랜잭션)

---

### 10. **`_on_connection_closed()` - 연결 종료 콜백**
```python
def _on_connection_closed(self, conn, reason):
    # Connection failures imply stale metadata
    if reason in [CloseReason.CONNECTION_BROKEN, CloseReason.CONNECTION_TIMEOUT]:
        self.force_metadata_update()
```

**역할**: 연결 문제 발생 시 메타데이터 갱신 트리거 (리더 변경 가능성)

---

## 🔗 다른 모듈과의 관계

### 의존성 (Imports)
```
client.py
├── aiokafka.conn (create_conn, AIOKafkaConnection)
├── aiokafka.cluster (ClusterMetadata)
├── aiokafka.protocol (각종 Request/Response)
│   ├── MetadataRequest
│   ├── FindCoordinatorRequest
│   ├── ApiVersionRequest
│   └── ...
└── aiokafka.errors (예외)
```

### 사용처
- `producer.py`: `AIOKafkaClient`를 사용하여 메타데이터 및 연결 관리
- `consumer.py`: `AIOKafkaClient`를 사용하여 Consumer Group 관리
- `admin.py`: `AIOKafkaClient`를 래핑하여 관리 작업 수행

---

## ⚙️ 핵심 설계 패턴

### 1. **연결 풀 관리**
```python
_conns = {
    (node_id=0, group=DEFAULT): conn_0,
    (node_id=1, group=DEFAULT): conn_1,
    (node_id=0, group=COORDINATION): coord_conn_0,
    ("bootstrap", group=DEFAULT): bootstrap_conn,
}
```
- **키**: `(node_id, ConnectionGroup)` 튜플
- **재사용**: 동일 노드+그룹 요청 시 연결 재사용
- **자동 정리**: 연결 끊김 시 `_on_connection_closed()`로 제거

### 2. **백그라운드 메타데이터 동기화**
```
_md_synchronizer() (백그라운드 태스크)
    ↓
    주기적 or 강제 트리거
    ↓
    _metadata_update()
    ↓
    cluster.update_metadata()
    ↓
    _md_update_fut.set_result()  ← 대기 중인 코드 깨어남
```
- **트리거 방식**:
  - 주기적: `metadata_max_age_ms` 타이머
  - 강제: `force_metadata_update()` 호출 시 `_md_update_waiter` 완료

### 3. **API 버전 프로빙**
```
check_version()
    ↓
    ApiVersionRequest_v0 전송
    ↓ (실패 시)
    ListGroupsRequest_v0 전송
    ↓ (실패 시)
    ... (순차적 버전 다운그레이드)
```
- **장점**: 브로커 버전 몰라도 자동 감지
- **단점**: 초기 연결 시 약간의 오버헤드

### 4. **Future 기반 메타데이터 대기**
```python
# Producer에서
await client.force_metadata_update()  # 백그라운드에서 업데이트 대기

# Consumer에서
partitions = await client._wait_on_metadata(topic)  # 토픽 메타데이터 준비될 때까지
```

---

## 📊 실행 흐름 예시

### 초기화 및 부트스트랩
```
1. client = AIOKafkaClient(bootstrap_servers="broker1:9092,broker2:9092")
   ↓
2. await client.bootstrap()
   ↓
3. broker1:9092에 연결 → MetadataRequest 전송
   ↓
4. cluster.update_metadata(response)
   - brokers: [0, 1, 2]
   - topics: {topic1: [0, 1, 2], topic2: [0, 1]}
   ↓
5. check_version() → api_version = (2, 5, 0)
   ↓
6. _md_synchronizer() 백그라운드 태스크 시작
```

### 메타데이터 강제 업데이트
```
1. Producer: client.force_metadata_update()
   ↓
2. _md_update_waiter.set_result(None)  ← _md_synchronizer 깨어남
   ↓
3. _md_synchronizer:
   - _md_update_fut = create_future()
   - await _metadata_update(cluster, topics)
   - _md_update_fut.set_result(True)
   ↓
4. Producer: await force_metadata_update()  ← 완료
```

### 연결 획득 및 요청
```
1. Producer: await client.send(node_id=1, ProduceRequest(...))
   ↓
2. client.ready(1) → _get_conn(1, group=DEFAULT)
   ↓
3. _conns[(1, DEFAULT)] 없음
   ↓
4. broker = cluster.broker_metadata(1) → (host="broker2", port=9092)
   ↓
5. async with _get_conn_lock:
       conn = await create_conn("broker2", 9092, on_close=_on_connection_closed)
       _conns[(1, DEFAULT)] = conn
   ↓
6. await conn.send(ProduceRequest(...))
```

### 연결 실패 처리
```
1. conn.send() → asyncio.TimeoutError
   ↓
2. client.send():
   - conn.close(reason=CONNECTION_TIMEOUT)
   ↓
3. _on_connection_closed(conn, CONNECTION_TIMEOUT)
   - force_metadata_update()  ← 리더 변경 가능성
   ↓
4. 다음 요청 시:
   - _get_conn(1) → 연결 재생성
```

---

## 🚨 에러 처리

### 예외 타입
| 예외 | 발생 상황 |
|------|-----------|
| `KafkaConnectionError` | 부트스트랩 실패, 메타데이터 업데이트 실패 |
| `NodeNotReadyError` | 연결 없는 노드에 요청 시도 |
| `RequestTimedOutError` | 요청 타임아웃 |
| `StaleMetadata` | 메타데이터에 없는 브로커 ID 참조 |
| `UnknownTopicOrPartitionError` | 토픽 메타데이터 타임아웃 |
| `UnrecognizedBrokerVersion` | API 버전 감지 실패 |

### 재시도 로직
- **메타데이터 업데이트**: 모든 노드 순회, 실패 시 `failed_update()` 기록
- **토픽 대기**: `request_timeout_ms` 내에서 `retry_backoff_ms` 간격으로 재시도
- **연결 생성**: 실패 시 `None` 반환, 상위 레이어에서 재시도 결정

---

## 🔑 핵심 특징 요약

| 특징 | 설명 |
|------|------|
| **부트스트랩** | 초기 서버 목록에서 하나라도 연결 성공하면 클러스터 전체 정보 획득 |
| **메타데이터 동기화** | 백그라운드 태스크로 자동 갱신 (주기적 + 이벤트 기반) |
| **연결 풀** | `(node_id, group)` 별로 연결 재사용 |
| **API 버전 자동 감지** | 프로빙으로 브로커 버전 자동 판별 |
| **Coordinator 관리** | Group/Transaction Coordinator 별도 연결 관리 |
| **장애 대응** | 연결 실패 시 자동으로 메타데이터 갱신 |

---

## 🎓 결과적으로 이 파일은

**Kafka 클러스터와의 고수준 통신 계층**으로서:
1. ✅ `conn.py` 위에서 **연결 풀** 및 **메타데이터 관리** 제공
2. ✅ **백그라운드 동기화**로 메타데이터 자동 갱신 (클러스터 변경 대응)
3. ✅ **API 버전 자동 감지**로 다양한 Kafka 버전 지원
4. ✅ **Future 기반 비동기 패턴**으로 효율적인 대기 메커니즘
5. ✅ **Coordinator 조회**로 Consumer Group, Transaction 기능 지원
6. ✅ **장애 복구 자동화**로 연결 문제 시 메타데이터 갱신 트리거

→ `Producer`, `Consumer`, `Admin`이 의존하는 **클러스터 추상화 레이어**
