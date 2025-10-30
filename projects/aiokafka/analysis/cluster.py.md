# cluster.py - Kafka 클러스터 메타데이터 관리

## 📋 파일 개요
- **경로**: `aiokafka/cluster.py`
- **라인 수**: 398줄
- **주요 역할**: Kafka 클러스터 메타데이터 저장 및 조회 (브로커, 토픽, 파티션, coordinator)

## 🎯 핵심 목적
**순수 데이터 구조**로서 Kafka 클러스터의 메타데이터를 관리하며, **IO 없이** MetadataResponse를 파싱하여 내부 상태를 업데이트하고 리스너에게 변경 사항을 알리는 **메타데이터 저장소**

---

## 🏗️ 주요 클래스 및 구조

### **ClusterMetadata** (단일 핵심 클래스)

#### 초기화 파라미터
```python
DEFAULT_CONFIG = {
    "retry_backoff_ms": 100,
    "metadata_max_age_ms": 300000,  # 5분
    "bootstrap_servers": [],
}

def __init__(self, **configs):
    # 설정 병합
```

#### 핵심 속성
```python
# 브로커 정보
self._brokers = {}              # node_id -> BrokerMetadata
self._bootstrap_brokers = {}    # "bootstrap-{i}" -> BrokerMetadata (초기 연결용)
self._coordinator_brokers = {}  # "coordinator-{id}" -> BrokerMetadata (Group/Txn)

# 토픽/파티션 정보
self._partitions = {}           # topic -> {partition -> PartitionMetadata}
self._broker_partitions = defaultdict(set)  # node_id -> {TopicPartition...}

# Coordinator 정보
self._groups = {}               # group_name -> node_id
self._coordinators = {}         # node_id -> BrokerMetadata (새 방식, Kafka 0.11+)
self._coordinator_by_key = {}   # (type, key) -> node_id

# 메타데이터 상태
self._last_refresh_ms = 0
self._last_successful_refresh_ms = 0
self._need_update = True
self._future = None             # 업데이트 대기용 Future

# 리스너 및 락
self._listeners = set()         # 메타데이터 변경 시 호출할 콜백
self._lock = threading.Lock()   # 스레드 안전성

# 특수 토픽
self.unauthorized_topics = set()  # 접근 권한 없는 토픽
self.internal_topics = set()      # 내부 토픽 (__consumer_offsets 등)
self.controller = None            # 컨트롤러 브로커 (Metadata v1+)
self.need_all_topic_metadata = False
```

---

## 🔄 주요 메서드 및 실행 흐름

### 1. **`update_metadata()` - 메타데이터 업데이트** ⭐
```python
def update_metadata(self, metadata):
    """MetadataResponse를 받아서 내부 상태 업데이트"""

    # 1. 브로커 정보 파싱
    _new_brokers = {}
    for broker in metadata.brokers:
        if metadata.API_VERSION == 0:
            node_id, host, port = broker
            rack = None
        else:
            node_id, host, port, rack = broker
        _new_brokers[node_id] = BrokerMetadata(node_id, host, port, rack)

    # 2. 컨트롤러 정보 (Metadata v1+)
    if metadata.API_VERSION > 0:
        _new_controller = _new_brokers.get(metadata.controller_id)

    # 3. 토픽/파티션 정보 파싱
    _new_partitions = {}
    _new_broker_partitions = defaultdict(set)
    _new_unauthorized_topics = set()
    _new_internal_topics = set()

    for topic_data in metadata.topics:
        if metadata.API_VERSION == 0:
            error_code, topic, partitions = topic_data
            is_internal = False
        else:
            error_code, topic, is_internal, partitions = topic_data

        if is_internal:
            _new_internal_topics.add(topic)

        error_type = Errors.for_code(error_code)
        if error_type is Errors.NoError:
            _new_partitions[topic] = {}
            for p_error, partition, leader, replicas, isr, *_ in partitions:
                _new_partitions[topic][partition] = PartitionMetadata(
                    topic=topic,
                    partition=partition,
                    leader=leader,
                    replicas=replicas,
                    isr=isr,
                    error=p_error,
                )
                # 리더 브로커에 파티션 매핑 추가
                if leader != -1:
                    _new_broker_partitions[leader].add(TopicPartition(topic, partition))

        # 토픽 에러 처리
        elif error_type is Errors.TopicAuthorizationFailedError:
            _new_unauthorized_topics.add(topic)
        # 기타 에러 로깅...

    # 4. 락 걸고 상태 업데이트
    with self._lock:
        self._brokers = _new_brokers
        self.controller = _new_controller
        self._partitions = _new_partitions
        self._broker_partitions = _new_broker_partitions
        self.unauthorized_topics = _new_unauthorized_topics
        self.internal_topics = _new_internal_topics

        f = None
        if self._future:
            f = self._future
        self._future = None
        self._need_update = False

    # 5. 타임스탬프 업데이트
    now = time.time() * 1000
    self._last_refresh_ms = now
    self._last_successful_refresh_ms = now

    # 6. 대기 중인 Future 완료
    if f:
        f.set_result(self)

    # 7. 리스너 호출
    for listener in self._listeners:
        listener(self)
```

**핵심 로직**:
1. **MetadataResponse 파싱**: API 버전에 따라 다른 필드 처리
2. **원자적 업데이트**: Lock 내에서 모든 상태 동시 변경
3. **Future 완료**: `request_update()`로 대기 중인 코드 깨우기
4. **리스너 알림**: 메타데이터 변경 시 콜백 호출

---

### 2. **`failed_update()` - 실패 처리**
```python
def failed_update(self, exception):
    f = None
    with self._lock:
        if self._future:
            f = self._future
            self._future = None
    if f:
        f.set_exception(exception)
    self._last_refresh_ms = time.time() * 1000
```
- **용도**: 메타데이터 요청 실패 시 대기 중인 Future에 예외 전파
- **호출처**: `client.py`의 `_metadata_update()`

---

### 3. **브로커 조회 메서드**

#### `brokers()` - 전체 브로커 목록
```python
def brokers(self):
    return set(self._brokers.values()) or set(self._bootstrap_brokers.values())
```
- **Fallback**: 정상 메타데이터 없으면 부트스트랩 브로커 반환

#### `broker_metadata()` - 특정 브로커 조회
```python
def broker_metadata(self, broker_id):
    return (
        self._brokers.get(broker_id)
        or self._bootstrap_brokers.get(broker_id)
        or self._coordinator_brokers.get(broker_id)
    )
```
- **우선순위**: 일반 브로커 → 부트스트랩 → Coordinator

#### `coordinator_metadata()` - Coordinator 조회 (신규)
```python
def coordinator_metadata(self, node_id):
    return self._coordinators.get(node_id)
```
- **용도**: Kafka 0.11+ Transaction/Group Coordinator

---

### 4. **토픽/파티션 조회 메서드**

#### `partitions_for_topic()` - 전체 파티션
```python
def partitions_for_topic(self, topic: str) -> set[int] | None:
    if topic not in self._partitions:
        return None
    return set(self._partitions[topic].keys())
```
- **반환**: `{0, 1, 2, ...}` (파티션 ID 집합) 또는 `None`

#### `available_partitions_for_topic()` - 사용 가능 파티션
```python
def available_partitions_for_topic(self, topic):
    if topic not in self._partitions:
        return None
    return {
        partition
        for partition, metadata in self._partitions[topic].items()
        if metadata.leader != -1
    }
```
- **조건**: `leader != -1` (리더 선출 완료)
- **용도**: Producer가 메시지 보낼 수 있는 파티션 확인

#### `leader_for_partition()` - 파티션 리더
```python
def leader_for_partition(self, partition):
    """Return node_id of leader, -1 if unavailable, None if unknown."""
    if partition.topic not in self._partitions:
        return None
    partitions = self._partitions[partition.topic]
    if partition.partition not in partitions:
        return None
    return partitions[partition.partition].leader
```
- **반환값**:
  - `node_id` (int): 정상
  - `-1`: 리더 없음 (선출 중)
  - `None`: 메타데이터 없음

#### `partitions_for_broker()` - 브로커의 파티션 목록
```python
def partitions_for_broker(self, broker_id):
    return self._broker_partitions.get(broker_id)
```
- **용도**: 특정 브로커가 리더인 파티션 목록
- **반환**: `{TopicPartition(topic='test', partition=0), ...}`

---

### 5. **Coordinator 관리**

#### `add_coordinator()` - Coordinator 추가 (신규)
```python
def add_coordinator(self, node_id, host, port, rack=None, *, purpose):
    """Keep track of coordinators separately by purpose"""
    # 1. 같은 목적의 기존 coordinator 제거
    if purpose in self._coordinator_by_key:
        old_id = self._coordinator_by_key.pop(purpose)
        del self._coordinators[old_id]

    # 2. 새 coordinator 등록
    self._coordinators[node_id] = BrokerMetadata(node_id, host, port, rack)
    self._coordinator_by_key[purpose] = node_id
```
- **purpose**: `(CoordinationType, key)` 튜플
  - 예: `(CoordinationType.GROUP, "my-consumer-group")`
  - 예: `(CoordinationType.TRANSACTION, "my-transactional-id")`
- **특징**: 같은 목적의 coordinator는 하나만 유지 (리밸런싱 대응)

#### `add_group_coordinator()` - Group Coordinator 추가 (구형)
```python
def add_group_coordinator(self, group, response):
    """Update with GroupCoordinatorResponse (deprecated)"""
    error_type = Errors.for_code(response.error_code)
    if error_type is not Errors.NoError:
        self._groups[group] = -1
        return None

    # Coordinator 전용 node_id 생성
    node_id = f"coordinator-{response.coordinator_id}"
    coordinator = BrokerMetadata(node_id, response.host, response.port, None)

    self._coordinator_brokers[node_id] = coordinator
    self._groups[group] = node_id
    return node_id
```
- **node_id 형식**: `"coordinator-{id}"` (일반 브로커와 구분)
- **용도**: Consumer Group 전용 연결 사용

#### `coordinator_for_group()` - Group Coordinator 조회
```python
def coordinator_for_group(self, group):
    return self._groups.get(group)
```

---

### 6. **메타데이터 상태 관리**

#### `request_update()` - 업데이트 요청
```python
def request_update(self):
    """Flags metadata for update, return Future"""
    with self._lock:
        self._need_update = True
        if not self._future or self._future.is_done:
            self._future = Future()
        return self._future
```
- **플래그**: `_need_update = True` (TTL 만료 표시)
- **Future 반환**: 호출자가 업데이트 완료 대기 가능

**사용처**:
```python
# client.py
fut = cluster.request_update()
await self._metadata_update(cluster, topics)
await fut  # 완료 대기
```

---

### 7. **리스너 패턴**

#### `add_listener()` / `remove_listener()`
```python
def add_listener(self, listener):
    self._listeners.add(listener)

def remove_listener(self, listener):
    self._listeners.remove(listener)
```

**사용 예**:
```python
def on_metadata_change(cluster):
    print(f"Cluster updated: {cluster}")

cluster.add_listener(on_metadata_change)
# update_metadata() 호출 시 자동으로 on_metadata_change() 실행
```

**실제 사용처**:
- Consumer: 파티션 할당 변경 감지
- Producer: 리더 변경 감지

---

### 8. **부트스트랩 관리**

#### `_generate_bootstrap_brokers()` - 부트스트랩 생성
```python
def _generate_bootstrap_brokers(self):
    bootstrap_hosts = collect_hosts(self.config["bootstrap_servers"])
    brokers = {}
    for i, (host, port, _) in enumerate(bootstrap_hosts):
        node_id = f"bootstrap-{i}"
        brokers[node_id] = BrokerMetadata(node_id, host, port, None)
    return brokers
```
- **node_id 형식**: `"bootstrap-0"`, `"bootstrap-1"`, ...
- **용도**: 초기 연결 전까지 사용

#### `is_bootstrap()` - 부트스트랩 여부 확인
```python
def is_bootstrap(self, node_id):
    return node_id in self._bootstrap_brokers
```

---

### 9. **유틸리티 메서드**

#### `topics()` - 전체 토픽 목록
```python
def topics(self, exclude_internal_topics=True):
    topics = set(self._partitions.keys())
    if exclude_internal_topics:
        return topics - self.internal_topics
    else:
        return topics
```
- **내부 토픽**: `__consumer_offsets`, `__transaction_state` 등

#### `with_partitions()` - 파티션 추가 복사본
```python
def with_partitions(self, partitions_to_add):
    """Returns a copy of metadata with partitions added"""
    new_metadata = ClusterMetadata(**self.config)
    new_metadata._brokers = copy.deepcopy(self._brokers)
    new_metadata._partitions = copy.deepcopy(self._partitions)
    # ...
    for partition in partitions_to_add:
        new_metadata._partitions[partition.topic][partition.partition] = partition
    return new_metadata
```
- **용도**: 테스트 또는 임시 메타데이터 생성

---

## 🔗 데이터 구조 예시

### 내부 상태 예시
```python
cluster = ClusterMetadata()

# MetadataResponse 처리 후:
cluster._brokers = {
    0: BrokerMetadata(nodeId=0, host="broker1", port=9092, rack="rack1"),
    1: BrokerMetadata(nodeId=1, host="broker2", port=9092, rack="rack2"),
    2: BrokerMetadata(nodeId=2, host="broker3", port=9092, rack="rack1"),
}

cluster._partitions = {
    "test-topic": {
        0: PartitionMetadata(topic="test-topic", partition=0, leader=0, replicas=[0, 1], isr=[0, 1]),
        1: PartitionMetadata(topic="test-topic", partition=1, leader=1, replicas=[1, 2], isr=[1, 2]),
        2: PartitionMetadata(topic="test-topic", partition=2, leader=2, replicas=[2, 0], isr=[2, 0]),
    },
    "__consumer_offsets": {
        0: PartitionMetadata(...),
        # ... 50개 파티션
    },
}

cluster._broker_partitions = {
    0: {TopicPartition("test-topic", 0), TopicPartition("other", 3)},
    1: {TopicPartition("test-topic", 1)},
    2: {TopicPartition("test-topic", 2), TopicPartition("other", 0)},
}

cluster.internal_topics = {"__consumer_offsets", "__transaction_state"}
cluster.unauthorized_topics = set()
cluster.controller = BrokerMetadata(nodeId=1, ...)
```

---

## 🔗 다른 모듈과의 관계

### 의존성 (Imports)
```
cluster.py
├── aiokafka.structs (BrokerMetadata, PartitionMetadata, TopicPartition)
├── aiokafka.errors (에러 코드 변환)
├── aiokafka.conn (collect_hosts)
└── threading (Lock)
```

### 사용처
- `client.py`: `ClusterMetadata` 인스턴스를 관리하고 `update_metadata()` 호출
- `producer.py`: 파티션 리더 조회 (`leader_for_partition()`)
- `consumer.py`: Coordinator 조회 (`coordinator_for_group()`)
- `fetcher.py`: 파티션 목록 조회 (`partitions_for_topic()`)

---

## ⚙️ 핵심 설계 패턴

### 1. **순수 데이터 구조 (No IO)**
```python
# ❌ 네트워크 요청 없음
# ✅ MetadataResponse를 받아서 상태만 업데이트
cluster.update_metadata(response)
```
- **장점**: 테스트 용이, 스레드 안전성 관리 단순

### 2. **스레드 안전성 (threading.Lock)**
```python
with self._lock:
    self._brokers = _new_brokers
    self._partitions = _new_partitions
    # 원자적 업데이트
```
- **이유**: asyncio 코드에서도 여러 태스크가 동시 접근 가능

### 3. **Future 기반 업데이트 대기**
```python
# 호출자
fut = cluster.request_update()
# ... 백그라운드에서 업데이트 ...
await fut  # 완료 대기

# 업데이트 완료 시
if f:
    f.set_result(self)
```

### 4. **Observer 패턴 (리스너)**
```
update_metadata()
    ↓
    상태 변경
    ↓
    for listener in self._listeners:
        listener(self)
```
- **용도**: 메타데이터 변경 시 자동 대응 (파티션 재할당 등)

### 5. **3-tier 브로커 관리**
```python
_brokers             # 일반 클러스터 브로커
_bootstrap_brokers   # 초기 연결용
_coordinator_brokers # Group/Transaction Coordinator
```
- **이유**: 각 역할별로 다른 라이프사이클

---

## 📊 실행 흐름 예시

### 초기 부트스트랩
```
1. cluster = ClusterMetadata(bootstrap_servers="broker1:9092,broker2:9092")
   ↓
2. _generate_bootstrap_brokers()
   - _bootstrap_brokers = {
       "bootstrap-0": BrokerMetadata("bootstrap-0", "broker1", 9092),
       "bootstrap-1": BrokerMetadata("bootstrap-1", "broker2", 9092),
     }
   ↓
3. brokers() 호출
   → set([BrokerMetadata("bootstrap-0", ...), BrokerMetadata("bootstrap-1", ...)])
```

### 메타데이터 업데이트
```
1. client.py: metadata = await conn.send(MetadataRequest(...))
   ↓
2. cluster.update_metadata(metadata)
   ↓
3. MetadataResponse 파싱:
   - brokers: [0, 1, 2]
   - topics: {
       "test": [(partition=0, leader=0), (partition=1, leader=1)],
     }
   ↓
4. with self._lock:
       self._brokers = {0: ..., 1: ..., 2: ...}
       self._partitions = {"test": {0: ..., 1: ...}}
       self._broker_partitions = {0: {TopicPartition("test", 0)}, ...}
   ↓
5. 리스너 호출:
   for listener in self._listeners:
       listener(self)  # Consumer가 파티션 재할당 시작
```

### Coordinator 조회
```
1. Consumer: node_id = await client.coordinator_lookup(CoordinationType.GROUP, "my-group")
   ↓
2. client.py:
   - resp = await self.send(FindCoordinatorRequest(...))
   - cluster.add_coordinator(resp.coordinator_id, resp.host, resp.port,
                              purpose=(CoordinationType.GROUP, "my-group"))
   ↓
3. cluster.py:
   - purpose = (CoordinationType.GROUP, "my-group")
   - 기존 coordinator 제거 (있다면)
   - _coordinators[node_id] = BrokerMetadata(...)
   - _coordinator_by_key[purpose] = node_id
   ↓
4. Consumer: coordinator = cluster.coordinator_metadata(node_id)
```

---

## 🚨 에러 처리

### 토픽 에러 처리
| 에러 코드 | 처리 |
|----------|------|
| `NoError` | 정상 파싱, 파티션 정보 저장 |
| `LeaderNotAvailableError` | 경고 로그 (auto-create 진행 중) |
| `UnknownTopicOrPartitionError` | 에러 로그 (토픽 없음) |
| `TopicAuthorizationFailedError` | `unauthorized_topics`에 추가 |
| `InvalidTopicError` | 에러 로그 (잘못된 토픽명) |

### 메타데이터 업데이트 실패
```python
cluster.failed_update(exception)
# → Future.set_exception(exception)
# → 대기 중인 코드에 예외 전파
```

---

## 🔑 핵심 특징 요약

| 특징 | 설명 |
|------|------|
| **순수 데이터 구조** | IO 없이 상태만 관리 |
| **스레드 안전** | threading.Lock으로 동시성 제어 |
| **Future 지원** | 비동기 업데이트 대기 가능 |
| **Observer 패턴** | 리스너로 변경 사항 자동 알림 |
| **다층 브로커 관리** | 일반/부트스트랩/Coordinator 구분 |
| **API 버전 대응** | MetadataResponse v0 ~ 최신 지원 |

---

## 🎓 결과적으로 이 파일은

**Kafka 클러스터 메타데이터의 중앙 저장소**로서:
1. ✅ **IO 없는 순수 데이터 구조**로 테스트 및 유지보수 용이
2. ✅ **스레드 안전성**으로 asyncio 환경에서도 안정적 동작
3. ✅ **Future 기반 대기**로 효율적인 메타데이터 갱신 흐름
4. ✅ **Observer 패턴**으로 메타데이터 변경 자동 전파
5. ✅ **다층 브로커 관리**로 부트스트랩/Coordinator/일반 브로커 역할 구분
6. ✅ **리더/파티션 조회**로 Producer/Consumer 라우팅 지원

→ `client.py`가 네트워크 계층이라면, `cluster.py`는 **메타데이터 모델 계층**
