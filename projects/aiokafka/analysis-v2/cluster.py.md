# cluster.py - 클러스터 메타데이터 관리

> **분석 일자**: 2025-10-29
> **파일**: `aiokafka/cluster.py` (397 lines)
> **방법론**: 문제 해결 중심 분석 (파일별)

---

## 📋 파일 개요

### 파일 정보
- **경로**: `aiokafka/cluster.py`
- **줄 수**: 397 lines
- **주요 클래스**: `ClusterMetadata`

### 핵심 역할

이 파일은 **Kafka 클러스터 정보를 저장**합니다:
- 브로커 목록 및 상태
- 토픽과 파티션 매핑
- 파티션 리더 정보
- Consumer group 코디네이터

**누가 사용하는가?**
- `AIOKafkaClient`가 메타데이터 저장소로 사용
- **No I/O**: 네트워크 요청 없음, 순수 데이터 구조

---

## 해결하는 핵심 문제들

이 파일은 **3가지 주요 문제**를 해결합니다.

---

## 1. In-Memory 메타데이터 저장 - 빠른 조회

### 문제

Kafka 메타데이터는 **자주 조회**됩니다:

1. **매 요청마다 필요**
   - 메시지 전송: "topic-A partition-0의 리더는?" → 브로커 2
   - Fetch: "topic-B partition-1의 리더는?" → 브로커 1
   - 초당 수천~수만 번 조회

2. **조회 성능이 중요**
   - 메타데이터 조회가 느리면 → 전체 성능 저하
   - O(1) 조회 필요

3. **여러 형태로 조회**
   - "토픽 X의 모든 파티션은?"
   - "파티션 Y의 리더는?"
   - "브로커 Z가 리더인 파티션들은?"

### 고민했던 선택지

#### 선택지 1: 매번 client에 요청

```python
def leader_for_partition(topic, partition):
    # 매번 client에 조회
    return client.get_leader(topic, partition)
```

**장점**: 구현 간단
**단점**: client 의존, I/O 발생 가능
**왜 안 됨**: 성능 저하

#### 선택지 2: 리스트로 저장

```python
brokers = [BrokerMetadata(...), ...]
partitions = [PartitionMetadata(...), ...]

def find_leader(topic, partition):
    for p in partitions:
        if p.topic == topic and p.partition == partition:
            return p.leader  # O(n)
```

**장점**: 간단
**단점**: O(n) 조회 → 느림
**왜 안 됨**: 성능 요구사항 불만족

#### 선택지 3 (최종): 딕셔너리 기반 인덱스

```python
_brokers = {}  # {node_id: BrokerMetadata}
_partitions = {}  # {topic: {partition: PartitionMetadata}}
_broker_partitions = {}  # {node_id: {TopicPartition...}}

def leader_for_partition(topic, partition):
    return _partitions[topic][partition].leader  # O(1)!
```

**장점**:
- ✅ **O(1) 조회**: 딕셔너리 lookup
- ✅ **여러 인덱스**: 다양한 조회 패턴 지원
- ✅ **메모리 효율**: 참조 공유

**단점**:
- ❌ 복잡도: 여러 딕셔너리 동기화 필요

**왜 선택했는가**: 성능이 가장 중요

### 최종 해결책

#### 구조

```python
class ClusterMetadata:
    def __init__(self, **configs):
        # 브로커 정보
        self._brokers = {}  # {node_id: BrokerMetadata}
        self._bootstrap_brokers = {}  # 초기 연결용
        self._coordinator_brokers = {}  # 코디네이터용

        # 토픽/파티션 정보
        self._partitions = {}  # {topic: {partition_id: PartitionMetadata}}

        # 역 인덱스: 브로커 → 파티션들
        self._broker_partitions = collections.defaultdict(set)
        # {node_id: {TopicPartition(topic, partition), ...}}

        # Consumer group 코디네이터
        self._groups = {}  # {group_name: node_id}
        self._coordinators = {}  # 코디네이터 상세 정보

        # 메타
        self._last_refresh_ms = 0
        self._need_update = True
        self._lock = threading.Lock()  # Thread-safe!

        self.unauthorized_topics = set()
        self.internal_topics = set()  # __consumer_offsets 등
        self.controller = None  # 컨트롤러 브로커

    def broker_metadata(self, broker_id):
        """브로커 정보 조회 - O(1)"""
        return (
            self._brokers.get(broker_id)
            or self._bootstrap_brokers.get(broker_id)
            or self._coordinator_brokers.get(broker_id)
        )

    def partitions_for_topic(self, topic):
        """토픽의 모든 파티션 - O(1)"""
        if topic not in self._partitions:
            return None
        return set(self._partitions[topic].keys())

    def available_partitions_for_topic(self, topic):
        """사용 가능한 파티션 (리더가 있는) - O(p)"""
        if topic not in self._partitions:
            return None
        return {
            partition
            for partition, metadata in self._partitions[topic].items()
            if metadata.leader != -1  # 리더 있음
        }

    def leader_for_partition(self, partition):
        """파티션 리더 조회 - O(1)"""
        if partition.topic not in self._partitions:
            return None
        partitions = self._partitions[partition.topic]
        if partition.partition not in partitions:
            return None
        return partitions[partition.partition].leader

    def partitions_for_broker(self, broker_id):
        """브로커가 리더인 파티션들 - O(1)"""
        return self._broker_partitions.get(broker_id)
```

#### 핵심 아이디어

**1. 중첩 딕셔너리 구조**

```python
_partitions = {
    "my-topic": {
        0: PartitionMetadata(leader=1, replicas=[1, 2], isr=[1, 2]),
        1: PartitionMetadata(leader=2, replicas=[2, 3], isr=[2, 3]),
        2: PartitionMetadata(leader=3, replicas=[3, 1], isr=[3, 1]),
    },
    "other-topic": {
        0: PartitionMetadata(leader=1, ...),
    }
}

# 조회
leader = _partitions["my-topic"][0].leader  # O(1)
```

**왜 중첩?**
- 토픽별로 그룹핑 → 토픽 삭제 시 간단
- 파티션 ID가 키 → O(1) lookup

**2. 역 인덱스 (Reverse Index)**

```python
_broker_partitions = {
    1: {TopicPartition("my-topic", 0), TopicPartition("other-topic", 0)},
    2: {TopicPartition("my-topic", 1)},
    3: {TopicPartition("my-topic", 2)},
}
```

**왜 필요?**
- "브로커 1이 죽으면 어떤 파티션이 영향받나?" 빠르게 파악
- Producer 부하 분산: 각 브로커의 파티션 수 확인

**3. 메타데이터 객체**

```python
@dataclass
class BrokerMetadata:
    nodeId: int
    host: str
    port: int
    rack: str | None

@dataclass
class PartitionMetadata:
    topic: str
    partition: int
    leader: int  # node_id
    replicas: list[int]
    isr: list[int]  # In-Sync Replicas
    error: int
```

불변 객체 → 안전한 공유

---

## 2. Thread-Safe 업데이트 - 동시성 제어

### 문제

메타데이터는 **여러 스레드/Task**가 접근합니다:

1. **읽기 (여러 곳에서)**
   - Producer Task: leader 조회
   - Consumer Task: partition 조회
   - Metadata sync Task: 업데이트

2. **쓰기 (백그라운드)**
   - Metadata sync Task가 주기적으로 업데이트
   - Update 중간에 읽기 발생 가능 → Inconsistent state

3. **Python의 GIL 한계**
   - GIL은 instruction 레벨 atomic만 보장
   - Dict 업데이트 도중 읽기 → 깨진 데이터 가능

### 고민했던 선택지

#### 선택지 1: Lock 없이 (GIL 신뢰)

```python
# GIL에만 의존
def update_metadata(self, metadata):
    self._partitions = new_partitions  # Atomic?
```

**장점**: 간단, Lock 오버헤드 없음
**단점**:
- Dict 업데이트는 atomic 아님
- Partial update 시 inconsistent
**왜 안 됨**: 안전하지 않음

#### 선택지 2: asyncio.Lock

```python
# asyncio Lock
self._lock = asyncio.Lock()

async def update_metadata(self, metadata):
    async with self._lock:
        # 업데이트...
```

**장점**: asyncio Task 간 동기화
**단점**:
- asyncio 전용 (threading 지원 안 함)
- 이 클래스는 I/O 없음 → async 불필요

**왜 안 됨**: 설계 방침과 맞지 않음

#### 선택지 3 (최종): threading.Lock

```python
self._lock = threading.Lock()

def update_metadata(self, metadata):
    with self._lock:
        # 원자적으로 업데이트
        ...
```

**장점**:
- ✅ Thread-safe
- ✅ asyncio와도 호환
- ✅ 간단

**단점**:
- ❌ Lock 오버헤드 (미미함)

**왜 선택했는가**: "안전 > 성능" (메타데이터 조회는 충분히 빠름)

### 최종 해결책

```python
class ClusterMetadata:
    def __init__(self):
        self._lock = threading.Lock()
        self._future = None
        self._need_update = True

    def update_metadata(self, metadata):
        """MetadataResponse로 업데이트 (Thread-safe)"""
        if not metadata.brokers:
            log.warning("No broker metadata -- ignoring")
            self.failed_update(Errors.MetadataEmptyBrokerList())
            return

        # 1. 새 데이터 구조 준비 (Lock 밖에서)
        _new_brokers = {}
        for broker in metadata.brokers:
            node_id, host, port, rack = broker
            _new_brokers[node_id] = BrokerMetadata(node_id, host, port, rack)

        _new_partitions = {}
        _new_broker_partitions = collections.defaultdict(set)

        for topic_data in metadata.topics:
            topic, partitions = topic_data
            _new_partitions[topic] = {}

            for partition_data in partitions:
                partition_id, leader, replicas, isr = partition_data
                _new_partitions[topic][partition_id] = PartitionMetadata(...)

                # 역 인덱스 구축
                if leader != -1:
                    _new_broker_partitions[leader].add(
                        TopicPartition(topic, partition_id)
                    )

        # 2. Lock 안에서 한 번에 교체 (Atomic)
        with self._lock:
            self._brokers = _new_brokers
            self._partitions = _new_partitions
            self._broker_partitions = _new_broker_partitions
            self.controller = _new_controller

            self._last_refresh_ms = time.time() * 1000
            self._last_successful_refresh_ms = self._last_refresh_ms
            self._need_update = False

            # Future 완료 알림
            if self._future:
                f = self._future
                self._future = None
                f.set_result(self)  # Lock 밖에서 호출!

    def request_update(self):
        """업데이트 요청 - Thread-safe"""
        with self._lock:
            self._need_update = True
            if not self._future or self._future.is_done:
                self._future = Future()  # concurrent.futures.Future
            return self._future
```

#### 핵심 아이디어

**1. Prepare-then-Swap 패턴**

```python
# Lock 밖: 준비 (느린 작업)
new_data = prepare_new_data(metadata)

# Lock 안: 교체 (빠른 작업)
with self._lock:
    self._data = new_data
```

**왜?**
- Lock 시간 최소화
- Lock 안에서는 포인터 교체만 → 매우 빠름
- 읽기 대기 시간 감소

**2. threading.Lock (not asyncio.Lock)**

**이유**:
- 이 클래스는 I/O 없음 (pure data structure)
- sync 메서드만 제공
- asyncio Event Loop 의존 없음

**호환성**:
```python
# asyncio에서도 사용 가능
loop.run_in_executor(None, cluster.update_metadata, metadata)

# 또는 그냥 호출 (Lock이 짧으면 OK)
cluster.update_metadata(metadata)
```

**3. concurrent.futures.Future**

```python
from concurrent.futures import Future  # not asyncio.Future!

self._future = Future()

# 완료 알림
self._future.set_result(self)

# 대기
await asyncio.wrap_future(self._future)
```

**왜 concurrent.futures.Future?**
- threading과 asyncio 모두 호환
- asyncio.wrap_future()로 변환 가능

---

## 3. 토픽/파티션 매핑 - 복잡한 관계 관리

### 문제

Kafka의 토픽/파티션 구조는 **복잡**합니다:

1. **다층 구조**
   - 클러스터 → 토픽들 → 파티션들 → 리더 브로커
   - 각 파티션마다 리더, 복제본, ISR

2. **다양한 조회 패턴**
   - "토픽 X의 파티션들"
   - "파티션 Y의 리더"
   - "브로커 Z가 리더인 파티션들"
   - "사용 가능한 파티션들 (리더 있는)"

3. **동적 변경**
   - 파티션 추가
   - 리더 변경 (failover)
   - 브로커 추가/제거

### 고민했던 선택지

#### 선택지 1: 플랫 리스트

```python
partitions = [
    {"topic": "A", "partition": 0, "leader": 1},
    {"topic": "A", "partition": 1, "leader": 2},
    {"topic": "B", "partition": 0, "leader": 1},
]
```

**장점**: 간단
**단점**: 조회 O(n), 업데이트 어려움
**왜 안 됨**: 성능

#### 선택지 2: 그래프 구조

```python
class Topic:
    def __init__(self, name):
        self.name = name
        self.partitions = []

class Partition:
    def __init__(self, topic, id):
        self.topic = topic  # 참조
        self.id = id
        self.leader = None
```

**장점**: 관계 명확
**단점**: 복잡도 높음, GC 부담
**왜 안 됨**: 오버 엔지니어링

#### 선택지 3 (최종): 중첩 딕셔너리 + 역 인덱스

```python
# 주 저장소
_partitions = {topic: {partition: PartitionMetadata}}

# 역 인덱스
_broker_partitions = {broker_id: {TopicPartition, ...}}
```

**장점**:
- ✅ O(1) 조회
- ✅ 다양한 패턴 지원
- ✅ 간단한 구조

**단점**:
- ❌ 업데이트 시 일관성 유지 필요

**왜 선택했는가**: 성능 + 단순함

### 최종 해결책

#### 조회 패턴들

```python
# 1. 토픽의 모든 파티션
def partitions_for_topic(self, topic):
    """O(1)"""
    if topic not in self._partitions:
        return None
    return set(self._partitions[topic].keys())

# 2. 사용 가능한 파티션 (리더 있는)
def available_partitions_for_topic(self, topic):
    """O(p) - p는 파티션 개수"""
    if topic not in self._partitions:
        return None
    return {
        partition_id
        for partition_id, metadata in self._partitions[topic].items()
        if metadata.leader != -1  # 리더 있음
    }

# 3. 파티션의 리더
def leader_for_partition(self, partition):
    """O(1)"""
    if partition.topic not in self._partitions:
        return None
    return self._partitions[partition.topic][partition.partition].leader

# 4. 브로커가 리더인 파티션들
def partitions_for_broker(self, broker_id):
    """O(1)"""
    return self._broker_partitions.get(broker_id)

# 5. Consumer group 코디네이터
def coordinator_for_group(self, group):
    """O(1)"""
    return self._groups.get(group)
```

#### 업데이트 시 일관성 유지

```python
def update_metadata(self, metadata):
    # 주 저장소와 역 인덱스를 함께 구축
    _new_partitions = {}
    _new_broker_partitions = collections.defaultdict(set)

    for topic_data in metadata.topics:
        topic = topic_data.topic
        _new_partitions[topic] = {}

        for partition_data in topic_data.partitions:
            partition_id = partition_data.partition
            leader = partition_data.leader

            # 주 저장소
            _new_partitions[topic][partition_id] = PartitionMetadata(...)

            # 역 인덱스 (동시 구축!)
            if leader != -1:
                _new_broker_partitions[leader].add(
                    TopicPartition(topic, partition_id)
                )

    # 한 번에 교체
    with self._lock:
        self._partitions = _new_partitions
        self._broker_partitions = _new_broker_partitions
```

**핵심**: 주 저장소와 역 인덱스를 **동시에** 구축 → 항상 일관성 보장

---

## 메타데이터 업데이트 흐름

```
Client._md_synchronizer() (백그라운드)
  ↓
1. MetadataRequest 전송
  ↓
2. MetadataResponse 수신
  ↓
3. cluster.update_metadata(response)
   ├─ a. 새 딕셔너리 준비 (Lock 밖)
   │    ├─ _new_brokers 구축
   │    ├─ _new_partitions 구축
   │    └─ _new_broker_partitions 구축 (역 인덱스)
   ├─ b. Lock 획득
   ├─ c. 포인터 교체 (매우 빠름)
   │    ├─ self._brokers = _new_brokers
   │    ├─ self._partitions = _new_partitions
   │    └─ self._broker_partitions = _new_broker_partitions
   ├─ d. 타임스탬프 갱신
   ├─ e. Lock 해제
   └─ f. Future 완료 알림
  ↓
4. 대기 중인 요청들 재개
```

---

## 주요 메서드 참고

### ClusterMetadata

| 메서드 | 역할 | 복잡도 |
|--------|------|--------|
| `broker_metadata(id)` | 브로커 정보 조회 | O(1) |
| `partitions_for_topic(topic)` | 토픽의 모든 파티션 | O(1) |
| `available_partitions_for_topic(topic)` | 사용 가능한 파티션 | O(p) |
| `leader_for_partition(partition)` | 파티션 리더 | O(1) |
| `partitions_for_broker(broker_id)` | 브로커의 파티션들 | O(1) |
| `update_metadata(response)` | 메타데이터 업데이트 | O(n) |
| `request_update()` | 업데이트 요청 | O(1) |

---

## 배운 점

### 1. In-Memory 인덱스 설계

**여러 조회 패턴 → 여러 인덱스**

```python
# 주 인덱스
_partitions = {topic: {partition: Metadata}}

# 역 인덱스
_broker_partitions = {broker: {TopicPartition}}
```

**적용**:
- 데이터베이스 인덱스 설계
- 캐시 레이어 설계
- 검색 엔진

### 2. Prepare-then-Swap 패턴

```python
# Lock 밖: 준비
new_data = prepare()

# Lock 안: 교체
with lock:
    self.data = new_data
```

**이점**:
- Lock 시간 최소화
- 읽기 스레드 대기 감소

**적용**:
- Configuration reload
- 캐시 갱신
- Hot-swap 패턴

### 3. threading.Lock vs asyncio.Lock

| 상황 | 선택 |
|------|------|
| I/O 없는 데이터 구조 | `threading.Lock` |
| asyncio 코루틴 | `asyncio.Lock` |
| 혼합 (thread + asyncio) | `threading.Lock` + executor |

**이유**: `threading.Lock`은 범용, `asyncio.Lock`은 asyncio 전용

### 4. concurrent.futures.Future

```python
from concurrent.futures import Future

# threading과 asyncio 모두 호환
fut = Future()

# asyncio에서 사용
await asyncio.wrap_future(fut)
```

**언제?**
- Thread-safe 통신
- threading ↔ asyncio 브리지

### 5. 역 인덱스의 중요성

**문제**: "브로커 X의 파티션들" 빠르게 조회
**해결**: 역 인덱스 유지

**트레이드오프**:
- 메모리 증가 vs 조회 속도
- 업데이트 복잡도 증가 vs 조회 단순화

### 6. 불변 객체 (namedtuple, dataclass)

```python
@dataclass(frozen=True)
class BrokerMetadata:
    nodeId: int
    host: str
    port: int
```

**이점**:
- Thread-safe (수정 불가)
- 안전한 공유
- Hashable

### 7. 비슷한 상황에 적용

| 패턴 | 적용 가능한 곳 |
|------|----------------|
| In-memory 인덱스 | 캐시, 검색 엔진, ORM |
| Prepare-then-Swap | Hot reload, 무중단 배포 |
| threading.Lock | 데이터 구조, 설정 관리 |
| 역 인덱스 | 관계형 데이터, 그래프 탐색 |

---

## 실전 적용 가이드

이 섹션은 **실제 프로젝트에 패턴을 적용**할 때 도움이 됩니다.

### 가이드 1: In-Memory 메타데이터 저장소 구현하기

**상황**: 자주 조회하는 데이터를 메모리에 캐싱 (설정, 사용자 정보, 라우팅 테이블 등)

#### Step 1: 요구사항 정의

```markdown
질문:
- [ ] 조회가 얼마나 자주 발생하는가? (초당 수천 번 이상 → O(1) 필수)
- [ ] 데이터 구조가 복잡한가? (중첩, 관계 있음)
- [ ] 여러 방식으로 조회하는가? (ID로, 이름으로, 그룹으로 등)
- [ ] 동시성 제어가 필요한가? (멀티스레드/Task)
```

#### Step 2: 인덱스 설계

```python
class MetadataStore:
    """메모리 메타데이터 저장소"""

    def __init__(self):
        # 주 인덱스 (중첩 딕셔너리)
        self._data = {}  # {category: {item_id: ItemData}}

        # 역 인덱스 (빠른 조회)
        self._by_name = {}  # {name: item_id}
        self._by_tag = {}  # {tag: {item_id, ...}}

        # 동시성 제어
        self._lock = threading.Lock()

    def add_item(self, category, item_id, name, tags, data):
        """아이템 추가 (Thread-safe)"""
        with self._lock:
            # 주 인덱스
            if category not in self._data:
                self._data[category] = {}
            self._data[category][item_id] = data

            # 역 인덱스 업데이트
            self._by_name[name] = item_id
            for tag in tags:
                if tag not in self._by_tag:
                    self._by_tag[tag] = set()
                self._by_tag[tag].add(item_id)

    def get_by_id(self, category, item_id):
        """ID로 조회 - O(1)"""
        with self._lock:
            return self._data.get(category, {}).get(item_id)

    def get_by_name(self, name):
        """이름으로 조회 - O(1)"""
        with self._lock:
            item_id = self._by_name.get(name)
            if item_id:
                # 주 인덱스에서 카테고리 찾기 (O(c), c=카테고리 수)
                for category, items in self._data.items():
                    if item_id in items:
                        return items[item_id]
            return None

    def get_by_tag(self, tag):
        """태그로 조회 - O(1)"""
        with self._lock:
            item_ids = self._by_tag.get(tag, set())
            result = []
            for category, items in self._data.items():
                for item_id in item_ids:
                    if item_id in items:
                        result.append(items[item_id])
            return result

# 사용 예시
store = MetadataStore()
store.add_item(
    category="users",
    item_id="u123",
    name="Alice",
    tags=["admin", "developer"],
    data={"email": "alice@example.com"}
)

# 다양한 조회
user = store.get_by_id("users", "u123")  # O(1)
user = store.get_by_name("Alice")  # O(1)
admins = store.get_by_tag("admin")  # O(1) + O(n)
```

#### Step 3: 인덱스 종류 선택

| 조회 패턴 | 인덱스 종류 | 복잡도 |
|----------|------------|--------|
| ID로 조회 | `{id: data}` | O(1) |
| 이름으로 조회 | `{name: id}` | O(1) |
| 그룹으로 조회 | `{group: {id, ...}}` | O(1) + O(n) |
| 범위 조회 | 정렬된 리스트 | O(log n) |

**트레이드오프:**
- 인덱스 많음 → 빠른 조회, 메모리 사용 증가
- 인덱스 적음 → 메모리 절약, 조회 느림

---

### 가이드 2: Prepare-then-Swap 패턴 구현하기

**상황**: 데이터 갱신 시 Lock 시간 최소화 (설정 리로드, 캐시 갱신 등)

#### Step 1: 문제 이해

**❌ 나쁜 방법 (Long Lock):**
```python
def update_config(self, new_config):
    with self._lock:  # Lock 시작
        # 느린 작업들...
        self._servers = self._parse_servers(new_config)  # 수백 ms
        self._routes = self._build_routes(new_config)    # 수백 ms
        self._cache = self._warm_cache(new_config)       # 초 단위
        # Lock 종료
```

**문제:**
- Lock 시간 = 전체 처리 시간 (초 단위)
- 읽기 Task들이 모두 대기 → 성능 저하

#### Step 2: Prepare-then-Swap

```python
def update_config(self, new_config):
    """설정 갱신 (Lock 시간 최소화)"""

    # 1. Prepare (Lock 밖에서 - 느린 작업)
    new_servers = self._parse_servers(new_config)  # 수백 ms
    new_routes = self._build_routes(new_config)    # 수백 ms
    new_cache = self._warm_cache(new_config)       # 초 단위

    # 2. Swap (Lock 안에서 - 빠른 작업)
    with self._lock:  # Lock 시작
        self._servers = new_servers
        self._routes = new_routes
        self._cache = new_cache
        self._version += 1  # 버전 증가
        # Lock 종료 (수 μs)

# 읽기는 언제든지
def get_server(self, server_id):
    """O(1) 조회 (Lock 짧음)"""
    with self._lock:
        return self._servers.get(server_id)
```

**이점:**
- Lock 시간: 초 단위 → μs 단위
- 읽기 Task 대기 시간 최소화

#### Step 3: Immutable 데이터 구조 사용

```python
from dataclasses import dataclass
from typing import List, Dict

@dataclass(frozen=True)  # Immutable!
class ServerConfig:
    id: str
    host: str
    port: int
    tags: List[str]

class ConfigStore:
    def __init__(self):
        self._config = None  # Immutable 객체
        self._lock = threading.Lock()

    def update(self, new_config_data):
        # 1. 새 immutable 객체 생성 (Lock 밖)
        new_config = ServerConfig(
            id=new_config_data['id'],
            host=new_config_data['host'],
            port=new_config_data['port'],
            tags=new_config_data.get('tags', [])
        )

        # 2. 포인터 교체 (Lock 안)
        with self._lock:
            self._config = new_config

    def get(self):
        """Thread-safe 읽기 (복사 불필요)"""
        with self._lock:
            return self._config  # Immutable → 안전한 공유
```

**왜 Immutable?**
- Lock 밖에서 읽어도 안전 (변경 불가)
- 복사 비용 없음 (참조만 공유)

---

### 가이드 3: 역 인덱스 구축하기

**상황**: 양방향 조회 필요 (사용자→그룹, 그룹→사용자)

#### Step 1: 문제 정의

**요구사항:**
- "사용자 X가 속한 그룹들은?" → O(1)
- "그룹 Y에 속한 사용자들은?" → O(1)

**Naive 방법 (느림):**
```python
# 사용자 → 그룹들
def get_user_groups(user_id):
    groups = []
    for group in all_groups:  # O(n)
        if user_id in group.members:
            groups.append(group)
    return groups
```

#### Step 2: 역 인덱스 구축

```python
class GroupManager:
    def __init__(self):
        # 주 인덱스
        self._groups = {}  # {group_id: Group}

        # 역 인덱스
        self._user_groups = {}  # {user_id: {group_id, ...}}

        self._lock = threading.Lock()

    def add_user_to_group(self, user_id, group_id):
        """사용자 → 그룹 추가"""
        with self._lock:
            # 주 인덱스 업데이트
            if group_id not in self._groups:
                self._groups[group_id] = Group(group_id)
            self._groups[group_id].members.add(user_id)

            # 역 인덱스 업데이트 (동시!)
            if user_id not in self._user_groups:
                self._user_groups[user_id] = set()
            self._user_groups[user_id].add(group_id)

    def get_user_groups(self, user_id):
        """O(1) 조회"""
        with self._lock:
            return self._user_groups.get(user_id, set())

    def get_group_members(self, group_id):
        """O(1) 조회"""
        with self._lock:
            group = self._groups.get(group_id)
            return group.members if group else set()

    def remove_user_from_group(self, user_id, group_id):
        """일관성 유지하며 제거"""
        with self._lock:
            # 주 인덱스에서 제거
            if group_id in self._groups:
                self._groups[group_id].members.discard(user_id)

            # 역 인덱스에서 제거 (동시!)
            if user_id in self._user_groups:
                self._user_groups[user_id].discard(group_id)
                if not self._user_groups[user_id]:
                    del self._user_groups[user_id]  # 빈 set 제거
```

#### Step 3: 일관성 유지

**핵심: 주 인덱스와 역 인덱스를 항상 함께 업데이트**

```python
def update_all(self, new_data):
    """전체 재구축"""
    # 1. 새 인덱스들 구축 (Lock 밖)
    new_groups = {}
    new_user_groups = {}

    for group_data in new_data:
        group_id = group_data['id']
        new_groups[group_id] = Group(group_id)

        for user_id in group_data['members']:
            new_groups[group_id].members.add(user_id)

            # 역 인덱스 (동시 구축!)
            if user_id not in new_user_groups:
                new_user_groups[user_id] = set()
            new_user_groups[user_id].add(group_id)

    # 2. 한 번에 교체 (Lock 안)
    with self._lock:
        self._groups = new_groups
        self._user_groups = new_user_groups
```

---

### 가이드 4: threading.Lock vs asyncio.Lock 선택하기

**상황**: 동시성 제어가 필요한 데이터 구조

#### Step 1: 차이점 이해

| 특성 | threading.Lock | asyncio.Lock |
|------|----------------|--------------|
| 사용 환경 | 멀티스레드 | asyncio 코루틴 |
| 블로킹 | OS 레벨 | 이벤트 루프 |
| 사용법 | `with lock:` | `async with lock:` |
| 호환성 | 어디서나 | asyncio 전용 |
| 성능 | 빠름 (C 구현) | 느림 (Python) |

#### Step 2: 선택 기준

```python
# Case 1: I/O 없는 데이터 구조 → threading.Lock
class DataStore:
    def __init__(self):
        self._data = {}
        self._lock = threading.Lock()  # ✅ threading.Lock

    def set(self, key, value):
        """I/O 없음 (순수 메모리 작업)"""
        with self._lock:  # 동기 Lock
            self._data[key] = value

    def get(self, key):
        with self._lock:
            return self._data.get(key)

# asyncio에서도 사용 가능!
async def use_in_asyncio():
    store = DataStore()
    store.set("key", "value")  # OK! (Lock이 짧으면)
    value = store.get("key")
```

```python
# Case 2: I/O 있는 코루틴 → asyncio.Lock
class AsyncCache:
    def __init__(self):
        self._cache = {}
        self._lock = asyncio.Lock()  # ✅ asyncio.Lock

    async def get_or_fetch(self, key):
        """I/O 포함 (네트워크 요청)"""
        async with self._lock:  # 비동기 Lock
            if key in self._cache:
                return self._cache[key]

            # I/O 작업 (await 필요)
            value = await fetch_from_network(key)
            self._cache[key] = value
            return value
```

#### Step 3: 혼합 사용

```python
# threading.Lock + asyncio 함께 사용
class HybridStore:
    def __init__(self):
        self._data = {}
        self._lock = threading.Lock()  # threading.Lock

    def set_sync(self, key, value):
        """동기 메서드"""
        with self._lock:
            self._data[key] = value

    async def set_async(self, key, value):
        """비동기 메서드"""
        # threading.Lock을 asyncio에서 사용
        # 주의: Lock 안에서 await 불가!
        with self._lock:
            self._data[key] = value  # 빠른 작업만

    async def set_async_with_io(self, key):
        """I/O 포함 시 Lock 밖에서"""
        # 1. Lock 밖에서 I/O (느린 작업)
        value = await fetch_from_network(key)

        # 2. Lock 안에서 업데이트 (빠른 작업)
        with self._lock:
            self._data[key] = value
```

**가이드라인:**
- ✅ 데이터 구조만 (I/O 없음) → `threading.Lock`
- ✅ 코루틴 + I/O → `asyncio.Lock`
- ⚠️  `threading.Lock` 안에서 `await` 금지!

---

## 안티패턴과 흔한 실수

실제 구현 시 자주 발생하는 문제들과 해결책입니다.

### 실수 1: 일관성 없는 인덱스 업데이트

**❌ 나쁜 예:**
```python
def add_item(self, topic, partition, leader):
    # 주 인덱스만 업데이트
    self._partitions[topic][partition] = PartitionData(leader=leader)

    # 역 인덱스 업데이트 빠뜨림!
    # self._broker_partitions[leader].add((topic, partition))
```

**문제:**
- 주 인덱스: "topic-A partition-0 리더 = broker-1"
- 역 인덱스: broker-1의 파티션 목록에 없음
- **일관성 깨짐!**

**✅ 좋은 예:**
```python
def add_item(self, topic, partition, leader):
    with self._lock:
        # 주 인덱스
        self._partitions[topic][partition] = PartitionData(leader=leader)

        # 역 인덱스 (동시 업데이트!)
        if leader not in self._broker_partitions:
            self._broker_partitions[leader] = set()
        self._broker_partitions[leader].add((topic, partition))
```

---

### 실수 2: Lock 밖에서 인덱스 구축 후 교체 안 함

**❌ 나쁜 예:**
```python
def update_metadata(self, metadata):
    # Lock 밖에서 인덱스 구축
    for item in metadata:
        # 직접 self._data 수정 (Lock 없음!)
        self._data[item.id] = item  # Race condition!
```

**문제:**
- 여러 스레드가 동시에 `update_metadata()` 호출 시
- `_data` 딕셔너리가 손상될 수 있음

**✅ 좋은 예:**
```python
def update_metadata(self, metadata):
    # 1. 새 딕셔너리 구축 (Lock 밖)
    new_data = {}
    for item in metadata:
        new_data[item.id] = item

    # 2. 포인터 교체 (Lock 안)
    with self._lock:
        self._data = new_data
```

---

### 실수 3: Immutable 객체 수정 시도

**❌ 나쁜 예:**
```python
@dataclass(frozen=True)
class BrokerMetadata:
    id: int
    host: str
    port: int

# 사용
broker = store.get_broker(1)
broker.host = "new-host"  # FrozenInstanceError!
```

**문제:**
- `frozen=True` → 수정 불가
- 예외 발생

**✅ 방법 1: 새 객체 생성**
```python
old_broker = store.get_broker(1)
new_broker = dataclass.replace(
    old_broker,
    host="new-host"
)
store.update_broker(1, new_broker)
```

**✅ 방법 2: Mutable 객체 사용 (주의)**
```python
@dataclass  # frozen=False (기본값)
class BrokerMetadata:
    id: int
    host: str
    port: int

# Lock 필요!
with store._lock:
    broker = store.get_broker(1)
    broker.host = "new-host"
```

---

### 실수 4: Lock 없이 딕셔너리 읽기

**❌ 나쁜 예:**
```python
def get_broker(self, broker_id):
    # Lock 없이 읽기!
    return self._brokers.get(broker_id)
```

**문제:**
- 읽는 도중 다른 스레드가 `self._brokers` 교체 가능
- 예: `self._brokers = new_brokers` (Prepare-then-Swap)
- Python에서는 보통 안전하지만, **보장 안 됨**

**✅ 좋은 예:**
```python
def get_broker(self, broker_id):
    with self._lock:
        return self._brokers.get(broker_id)
```

**예외: Immutable + 포인터 교체**
```python
# 이 경우는 Lock 없이도 안전
def get_broker(self, broker_id):
    brokers = self._brokers  # 참조 복사 (atomic)
    return brokers.get(broker_id)

# 왜 안전?
# - self._brokers 교체는 포인터 변경 (atomic)
# - 옛 brokers 딕셔너리는 살아있음 (GC 나중에)
```

---

### 실수 5: Lock 안에서 느린 작업

**❌ 나쁜 예:**
```python
def update_from_server(self):
    with self._lock:  # Lock 시작
        # 네트워크 I/O (초 단위!)
        data = requests.get("http://server/metadata").json()

        # 파싱 (수백 ms)
        parsed = self._parse(data)

        # 업데이트
        self._data = parsed
        # Lock 종료
```

**문제:**
- Lock 시간 = I/O + 파싱 시간 (초 단위)
- 모든 읽기 Task 대기

**✅ 좋은 예:**
```python
async def update_from_server(self):
    # 1. Lock 밖에서 I/O
    data = await fetch_from_server()

    # 2. Lock 밖에서 파싱
    parsed = self._parse(data)

    # 3. Lock 안에서 교체만
    with self._lock:
        self._data = parsed
```

---

### 실수 6: 빈 컬렉션 vs None

**❌ 나쁜 예:**
```python
def partitions_for_topic(self, topic):
    if topic not in self._partitions:
        return None  # None 반환

# 사용
partitions = store.partitions_for_topic("my-topic")
for p in partitions:  # TypeError if None!
    ...
```

**문제:**
- 호출자가 None 체크 필요
- 까먹으면 TypeError

**✅ 좋은 예:**
```python
def partitions_for_topic(self, topic):
    if topic not in self._partitions:
        return set()  # 빈 set 반환

# 사용 (간단!)
for p in store.partitions_for_topic("my-topic"):
    ...  # None 체크 불필요
```

---

### 실수 7: defaultdict 남용

**❌ 나쁜 예:**
```python
from collections import defaultdict

self._partitions = defaultdict(dict)

# 문제
partitions = self._partitions["non-existent-topic"]
# → {} (빈 딕셔너리 자동 생성!)
# 의도: None 반환하고 싶음
```

**문제:**
- 존재하지 않는 키 접근 시 자동 생성
- 의도치 않은 데이터 추가

**✅ 방법 1: 일반 dict 사용**
```python
self._partitions = {}

partitions = self._partitions.get("non-existent-topic")
# → None (명시적)
```

**✅ 방법 2: defaultdict + 명시적 체크**
```python
self._partitions = defaultdict(dict)

def partitions_for_topic(self, topic):
    if topic in self._partitions:
        return self._partitions[topic]
    return None  # 명시적
```

---

## 스케일 고려사항

규모별로 다른 전략이 필요합니다.

### 소규모 (항목 < 1000, 조회 < 100 req/s)

**권장 사항:**
- ✅ 간단한 딕셔너리 저장
- ✅ threading.Lock 하나로 충분
- ✅ 역 인덱스 선택적 (필요한 것만)
- ⚠️  메모리 최적화 불필요

**구현 예시:**
```python
class SimpleMetadata:
    def __init__(self):
        self._data = {}  # 간단한 딕셔너리
        self._lock = threading.Lock()

    def set(self, key, value):
        with self._lock:
            self._data[key] = value

    def get(self, key):
        with self._lock:
            return self._data.get(key)

    def get_all(self):
        with self._lock:
            return list(self._data.values())  # 복사
```

**모니터링:**
- 메타데이터 크기
- 갱신 빈도

---

### 중규모 (항목 1k-100k, 조회 100-10k req/s)

**권장 사항:**
- ✅ 중첩 딕셔너리 + 역 인덱스
- ✅ Prepare-then-Swap 패턴
- ✅ Immutable 데이터 사용
- ✅ 메모리 프로파일링
- ⚠️  Lock 시간 측정

**구현 예시:**
```python
class MediumMetadata:
    def __init__(self):
        # 중첩 구조
        self._data = {}  # {category: {id: data}}

        # 역 인덱스들
        self._by_name = {}
        self._by_tag = defaultdict(set)

        # Lock
        self._lock = threading.Lock()

        # 메트릭
        self._metrics = {
            'total_items': 0,
            'update_count': 0,
            'lock_wait_time': 0,
        }

    def update_all(self, new_data):
        """Prepare-then-Swap"""
        start = time.time()

        # 1. Prepare (Lock 밖)
        new_main = {}
        new_by_name = {}
        new_by_tag = defaultdict(set)

        for category, items in new_data.items():
            new_main[category] = {}
            for item_id, item in items.items():
                new_main[category][item_id] = item
                new_by_name[item.name] = (category, item_id)
                for tag in item.tags:
                    new_by_tag[tag].add((category, item_id))

        # 2. Swap (Lock 안)
        with self._lock:
            self._data = new_main
            self._by_name = new_by_name
            self._by_tag = new_by_tag
            self._metrics['total_items'] = sum(
                len(items) for items in new_main.values()
            )
            self._metrics['update_count'] += 1

        # 메트릭 기록
        elapsed = time.time() - start
        self._metrics['lock_wait_time'] = elapsed
```

**모니터링:**
- Lock 획득 시간
- 메모리 사용량 (역 인덱스 포함)
- 갱신 소요 시간

---

### 대규모 (항목 100k+, 조회 10k+ req/s)

**권장 사항:**
- ✅ Lock-free 읽기 (가능하면)
- ✅ Copy-on-Write 패턴
- ✅ Sharding (카테고리별 Lock 분리)
- ✅ 메모리 압축 (필요 시)
- ✅ Lazy 인덱스 구축

**구현 예시:**
```python
import threading
from typing import Dict, Any

class LargeScaleMetadata:
    def __init__(self, num_shards=16):
        # Shard별 Lock (경합 분산)
        self._shards = [
            {
                'data': {},
                'lock': threading.RLock(),
            }
            for _ in range(num_shards)
        ]
        self._num_shards = num_shards

        # 메트릭
        self._metrics = {
            'shard_sizes': [0] * num_shards,
            'lock_contentions': [0] * num_shards,
        }

    def _get_shard(self, key):
        """키로 Shard 선택"""
        shard_idx = hash(key) % self._num_shards
        return self._shards[shard_idx]

    def set(self, key, value):
        """Sharded Lock"""
        shard = self._get_shard(key)

        with shard['lock']:
            shard['data'][key] = value

    def get(self, key):
        """Lock-free 읽기 (가능하면)"""
        shard = self._get_shard(key)

        # Copy-on-Write로 갱신하면 Lock 없이도 안전
        data_snapshot = shard['data']  # 참조 복사 (atomic)
        return data_snapshot.get(key)

    def update_shard(self, shard_idx, new_data):
        """특정 Shard만 갱신"""
        shard = self._shards[shard_idx]

        # Prepare-then-Swap
        new_dict = dict(new_data)  # 복사

        with shard['lock']:
            shard['data'] = new_dict  # 포인터 교체
            self._metrics['shard_sizes'][shard_idx] = len(new_dict)

    def update_all(self, all_data: Dict[int, Dict[str, Any]]):
        """전체 Shard 병렬 갱신"""
        import concurrent.futures

        with concurrent.futures.ThreadPoolExecutor(max_workers=self._num_shards) as executor:
            futures = []
            for shard_idx, data in all_data.items():
                future = executor.submit(self.update_shard, shard_idx, data)
                futures.append(future)

            # 모두 완료 대기
            concurrent.futures.wait(futures)

# Copy-on-Write 패턴
class CopyOnWriteDict:
    def __init__(self):
        self._data = {}
        self._lock = threading.Lock()

    def set(self, key, value):
        """쓰기 시 복사"""
        with self._lock:
            # 전체 딕셔너리 복사
            new_data = self._data.copy()
            new_data[key] = value
            self._data = new_data  # 포인터 교체 (atomic)

    def get(self, key):
        """Lock-free 읽기"""
        # 포인터 복사 (atomic)
        data_snapshot = self._data
        return data_snapshot.get(key)
```

**고급 최적화:**

1. **Lazy 인덱스 구축**
```python
class LazyIndex:
    def __init__(self):
        self._data = {}
        self._index_cache = None
        self._index_dirty = True
        self._lock = threading.Lock()

    def add(self, key, value):
        with self._lock:
            self._data[key] = value
            self._index_dirty = True  # 인덱스 무효화

    def get_index(self):
        """필요할 때만 인덱스 구축"""
        with self._lock:
            if self._index_dirty:
                # 인덱스 재구축
                self._index_cache = self._build_index()
                self._index_dirty = False

            return self._index_cache

    def _build_index(self):
        # 느린 작업
        return {v: k for k, v in self._data.items()}
```

2. **메모리 압축 (대량 데이터)**
```python
import pickle
import zlib

class CompressedMetadata:
    def __init__(self):
        self._compressed = None
        self._cache = None

    def set(self, data):
        """압축 저장"""
        serialized = pickle.dumps(data)
        self._compressed = zlib.compress(serialized)
        self._cache = data  # 캐시

    def get(self):
        """압축 해제"""
        if self._cache is not None:
            return self._cache

        # 압축 해제 (느림)
        serialized = zlib.decompress(self._compressed)
        self._cache = pickle.loads(serialized)
        return self._cache
```

**모니터링 필수:**
```python
# Prometheus 스타일 메트릭
- metadata_items_total: 총 항목 수
- metadata_shard_sizes: Shard별 크기
- metadata_update_duration_seconds: 갱신 시간
- metadata_lock_wait_seconds: Lock 대기 시간
- metadata_memory_bytes: 메모리 사용량
```

---

### 병목 지점과 해결책

| 병목 | 증상 | 해결책 |
|------|------|--------|
| Lock 경합 | 읽기 느림 | Shard별 Lock (10-100 shards) |
| 메모리 부족 | OOM | Lazy 인덱스, 압축, 페이징 |
| 갱신 느림 | 전체 블록 | Partial update, 증분 갱신 |
| 역 인덱스 너무 많음 | 메모리 증가 | 필요한 것만, Lazy 구축 |
| Copy 비용 | 갱신 느림 | Copy-on-Write, Immutable |

---

## 요약

| 문제 | 선택지 | 최종 해결 | 트레이드오프 |
|------|--------|-----------|--------------|
| 메타데이터 저장 | 리스트 vs 딕셔너리 | 중첩 딕셔너리 + 역 인덱스 | 메모리 vs 성능 |
| 동시성 제어 | GIL vs asyncio.Lock vs threading.Lock | threading.Lock | 범용성 vs asyncio 최적화 |
| 토픽/파티션 매핑 | 플랫 vs 그래프 vs 딕셔너리 | 딕셔너리 + 역 인덱스 | 단순함 vs 다양한 조회 |

**핵심 메시지:**
- In-memory 데이터는 **여러 인덱스**로 빠른 조회
- **Prepare-then-Swap**으로 Lock 시간 최소화
- **threading.Lock**은 범용적 (asyncio에서도 사용 가능)
- No I/O = No async = 순수 데이터 구조

---

**분석 완료일**: 2025-10-29
**방법론**: "문제 → 고민 → 해결" 중심 파일별 분석
**관련 파일**: client.py (ClusterMetadata 사용자)
