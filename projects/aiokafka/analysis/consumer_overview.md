# Consumer 계층 - 메시지 수신

## 📋 개요
- **경로**: `aiokafka/consumer/`
- **주요 파일**: `consumer.py`, `fetcher.py`, `group_coordinator.py`, `subscription_state.py`
- **총 라인 수**: ~4,646줄
- **주요 역할**: Kafka에서 메시지 수신 (Consumer Group, Offset 관리, 리밸런싱)

## 🎯 핵심 목적
**Consumer Group 기반 메시지 수신**을 제공하며, **파티션 할당**, **오프셋 커밋**, **리밸런싱**, **Fetch 최적화**를 지원하는 **고수준 Consumer API**

---

## 🏗️ Consumer 아키텍처

```
AIOKafkaConsumer (사용자 API)
    ↓ subscribe(topics) / assign(partitions)
    │
    ├── GroupCoordinator
    │   ├── JoinGroup (Consumer Group 가입)
    │   ├── SyncGroup (파티션 할당)
    │   ├── Heartbeat (백그라운드)
    │   └── OffsetCommit (오프셋 커밋)
    │
    ├── SubscriptionState
    │   ├── 할당된 파티션 추적
    │   ├── Position (현재 오프셋)
    │   └── Committed (커밋된 오프셋)
    │
    └── Fetcher
        ├── FetchRequest 생성
        ├── RecordBatch 파싱
        └── 역직렬화
            ↓
        getone() / getmany() → 메시지 반환
```

---

## 📦 주요 컴포넌트

### 1. **AIOKafkaConsumer** (consumer.py)
- **역할**: 사용자 대면 API
- **핵심 메서드**: `subscribe()`, `getone()`, `getmany()`, `commit()`

### 2. **GroupCoordinator** (group_coordinator.py)
- **역할**: Consumer Group 프로토콜 관리
- **기능**: JoinGroup, SyncGroup, Heartbeat, LeaveGroup, OffsetCommit

### 3. **Fetcher** (fetcher.py)
- **역할**: 메시지 fetch 및 파싱
- **기능**: FetchRequest, RecordBatch 디코딩, Prefetch 버퍼

### 4. **SubscriptionState** (subscription_state.py)
- **역할**: 구독 및 파티션 할당 상태 추적
- **기능**: 파티션별 오프셋 관리 (position, committed, highwater)

---

## 🔄 AIOKafkaConsumer 주요 메서드

### **`__init__()`** - 초기화
```python
consumer = AIOKafkaConsumer(
    'topic1', 'topic2',  # 구독할 토픽
    bootstrap_servers='localhost:9092',
    group_id='my-group',                 # Consumer Group ID
    client_id='my-consumer',
    key_deserializer=lambda k: k.decode('utf-8'),
    value_deserializer=lambda v: json.loads(v.decode('utf-8')),
    auto_offset_reset='earliest',        # 'earliest', 'latest', 'none'
    enable_auto_commit=True,              # 자동 오프셋 커밋
    auto_commit_interval_ms=5000,        # 5초마다 자동 커밋
    max_poll_records=500,                # 한 번에 가져올 최대 메시지 수
    session_timeout_ms=10000,            # Consumer Group 세션 타임아웃
    ...
)
```

**핵심 파라미터**:
| 파라미터 | 설명 | 기본값 |
|----------|------|--------|
| `group_id` | Consumer Group ID | None (독립 Consumer) |
| `auto_offset_reset` | 오프셋 없을 때 동작 | 'latest' |
| `enable_auto_commit` | 자동 오프셋 커밋 여부 | True |
| `max_poll_records` | 한 번에 fetch할 최대 레코드 수 | 500 |
| `session_timeout_ms` | 세션 타임아웃 | 10000 (10초) |
| `heartbeat_interval_ms` | 하트비트 간격 | 3000 (3초) |

---

### **`start()`** - Consumer 시작
```python
async def start(self):
    # 1. Kafka 클라이언트 초기화
    await self.client.bootstrap()

    # 2. GroupCoordinator 시작 (group_id 있을 때)
    if self._group_id is not None:
        await self._coordinator.ensure_coordinator_known()

    # 3. Fetcher 시작
    await self._fetcher.start()

    # 4. 구독 또는 할당
    if self._subscription:
        await self.subscribe(self._subscription.topics)
    elif self._assignment:
        self.assign(self._assignment)
```

---

### **`subscribe()`** - 토픽 구독 (Consumer Group)
```python
async def subscribe(self, topics=(), pattern=None, listener=None):
    """Consumer Group 기반 토픽 구독 (파티션 자동 할당)"""
    if isinstance(topics, str):
        topics = [topics]

    self._subscription.subscribe(topics=topics, pattern=pattern, listener=listener)

    # GroupCoordinator를 통해 파티션 할당
    if self._group_id:
        await self._coordinator.ensure_active_group()
```

**사용 예시**:
```python
# 특정 토픽 구독
await consumer.subscribe(['topic1', 'topic2'])

# 패턴 구독 (정규식)
await consumer.subscribe(pattern='^test.*')
```

**vs assign()**:
- `subscribe()`: Consumer Group 사용, 파티션 **자동 할당** (리밸런싱)
- `assign()`: 독립 Consumer, 파티션 **수동 할당** (리밸런싱 없음)

---

### **`assign()`** - 파티션 수동 할당
```python
def assign(self, partitions):
    """특정 파티션 수동 할당 (독립 Consumer)"""
    self._subscription.assign_from_user(partitions)
    self._client.set_topics([tp.topic for tp in partitions])
```

**사용 예시**:
```python
from aiokafka.structs import TopicPartition

# 특정 파티션만 할당
consumer.assign([
    TopicPartition('test', 0),
    TopicPartition('test', 2)
])
```

---

### **`getone()`** - 메시지 하나 수신 ⭐
```python
async def getone(self, *partitions):
    """메시지 하나 가져오기 (대기)"""
    assert self.assignment(), "No partitions assigned"

    # Prefetch 버퍼에서 가져오기
    msg = self._fetcher.next_record(partitions)
    if msg is None:
        # 버퍼 비었으면 fetch
        await self._fetcher.fetched_records(partitions, timeout=0)
        msg = self._fetcher.next_record(partitions)

    if msg:
        return msg

    # 대기
    while True:
        await self._fetcher.fetched_records(partitions)
        msg = self._fetcher.next_record(partitions)
        if msg:
            return msg
```

**반환값**: `ConsumerRecord`
```python
msg = await consumer.getone()
print(f"Topic: {msg.topic}")
print(f"Partition: {msg.partition}")
print(f"Offset: {msg.offset}")
print(f"Key: {msg.key}")
print(f"Value: {msg.value}")
print(f"Timestamp: {msg.timestamp}")
print(f"Headers: {msg.headers}")
```

---

### **`getmany()`** - 메시지 여러 개 수신
```python
async def getmany(self, *partitions, timeout_ms=0, max_records=None):
    """여러 메시지 가져오기 (배치)"""
    max_records = max_records or self._max_poll_records
    records = {}

    # Prefetch 버퍼에서 가져오기
    records = self._fetcher.fetched_records(
        partitions, timeout=timeout_ms / 1000, max_records=max_records
    )

    return records
```

**반환값**: `{TopicPartition: [ConsumerRecord, ...]}`
```python
records = await consumer.getmany(timeout_ms=1000, max_records=100)

for tp, messages in records.items():
    for msg in messages:
        print(f"{tp}: {msg.value}")
```

**getone() vs getmany()**:
| 메서드 | 반환 | 용도 |
|--------|------|------|
| `getone()` | 단일 `ConsumerRecord` | 실시간 처리 |
| `getmany()` | `dict[TopicPartition, list]` | 배치 처리 (효율적) |

---

### **`commit()`** - 오프셋 커밋
```python
async def commit(self, offsets=None):
    """현재 오프셋 커밋"""
    if self._group_id is None:
        raise IllegalOperation("Cannot commit without group_id")

    if offsets is None:
        # 현재 position 커밋
        offsets = self._subscription.all_consumed_offsets()

    await self._coordinator.commit_offsets(offsets)
```

**수동 커밋 예시**:
```python
consumer = AIOKafkaConsumer(
    'test',
    group_id='my-group',
    enable_auto_commit=False  # 자동 커밋 비활성화
)

async for msg in consumer:
    process(msg)
    await consumer.commit()  # 처리 후 커밋
```

---

### **`seek()`** - 오프셋 이동
```python
def seek(self, partition, offset):
    """특정 오프셋으로 이동"""
    self._subscription.seek(partition, offset)
```

**사용 예시**:
```python
tp = TopicPartition('test', 0)

# 특정 오프셋으로 이동
consumer.seek(tp, 100)

# 처음부터
consumer.seek_to_beginning(tp)

# 최신부터
consumer.seek_to_end(tp)

# 타임스탬프 기반
offsets = await consumer.offsets_for_times({tp: timestamp_ms})
consumer.seek(tp, offsets[tp].offset)
```

---

## 🔄 Consumer Group 프로토콜

### **리밸런싱 흐름**
```
1. Consumer 시작
    ↓
2. FindCoordinator (Group Coordinator 찾기)
    ↓
3. JoinGroup (그룹 가입)
    - Leader 선출
    - 모든 Consumer의 구독 정보 수집
    ↓
4. SyncGroup (파티션 할당)
    - Leader가 파티션 할당 계산
    - 각 Consumer에게 할당 전달
    ↓
5. Fetch & Heartbeat
    - 메시지 수신
    - 백그라운드 하트비트
    ↓
6. Rebalance 트리거 시
    - Consumer 추가/제거
    - 토픽 변경
    → 다시 JoinGroup으로
```

### **Heartbeat 메커니즘**
```python
# GroupCoordinator에서 백그라운드 태스크
async def _heartbeat_task(self):
    while True:
        await asyncio.sleep(self._heartbeat_interval_ms / 1000)

        try:
            await self._send_heartbeat_request()
        except Errors.RebalanceInProgressError:
            # 리밸런싱 필요
            await self._rejoin_group()
        except Errors.IllegalGenerationError:
            # Generation 변경됨
            await self._rejoin_group()
```

---

## 📊 메시지 수신 흐름

### 전체 흐름
```
User Code
    ↓
await consumer.getone()
    ↓
Fetcher.next_record()  # Prefetch 버퍼 확인
    ↓ (버퍼 비었으면)
Fetcher.fetched_records()
    ↓
FetchRequest 생성 및 전송
    ↓
client.send(FetchRequest)
    ↓
FetchResponse 수신
    ↓
RecordBatch 파싱 (압축 해제, 역직렬화)
    ↓
Prefetch 버퍼에 저장
    ↓
Fetcher.next_record() → ConsumerRecord 반환
    ↓
await msg → ConsumerRecord(topic, partition, offset, key, value)
```

### **Prefetch 최적화**
```python
# Fetcher는 백그라운드에서 미리 메시지 가져옴
# 사용자가 getone() 호출 시 즉시 반환 가능

# 예시:
# [버퍼] msg1, msg2, msg3, msg4, msg5 (이미 fetch됨)

await consumer.getone()  # → msg1 (즉시 반환)
await consumer.getone()  # → msg2 (즉시 반환)
await consumer.getone()  # → msg3 (즉시 반환)

# 버퍼 부족하면 백그라운드에서 다음 배치 fetch
```

---

## ⚙️ 핵심 설계 패턴

### 1. **비동기 Iterator**
```python
async for msg in consumer:
    print(msg.value)
    # 자동 커밋 (enable_auto_commit=True)
```

### 2. **Context Manager**
```python
async with AIOKafkaConsumer('test', group_id='my-group') as consumer:
    async for msg in consumer:
        process(msg)
# 자동으로 start() 및 stop() 호출
```

### 3. **Prefetch 버퍼**
```python
# FetchRequest로 여러 메시지 미리 가져옴
# getone() 호출 시 버퍼에서 즉시 반환 (레이턴시 감소)
```

### 4. **백그라운드 Heartbeat**
```python
# GroupCoordinator가 백그라운드에서 하트비트 전송
# 사용자는 신경 쓸 필요 없음
```

---

## 🔑 핵심 특징 요약

| 특징 | 설명 |
|------|------|
| **Consumer Group** | 파티션 자동 할당 및 리밸런싱 |
| **오프셋 관리** | 자동/수동 커밋, seek 지원 |
| **Prefetch** | 백그라운드에서 미리 메시지 가져오기 (성능) |
| **역직렬화** | key/value deserializer 지원 |
| **Heartbeat** | 백그라운드 하트비트 (자동) |
| **리밸런싱** | Consumer 추가/제거 시 자동 파티션 재할당 |
| **At-least-once** | 메시지 중복 가능, Exactly-once는 Consumer 직접 구현 |
| **Back-pressure** | Fetch 버퍼 제한으로 메모리 보호 |

---

## 🎓 결과적으로 Consumer 계층은

**Kafka에서 메시지를 수신하는 고수준 API**로서:
1. ✅ **Consumer Group** (파티션 자동 할당)
2. ✅ **오프셋 관리** (자동/수동 커밋)
3. ✅ **Prefetch** (성능 최적화)
4. ✅ **리밸런싱** (장애 대응)
5. ✅ **Heartbeat** (백그라운드 자동)
6. ✅ **역직렬화** (deserializer)
7. ✅ **Seek** (오프셋 제어)

→ 사용자는 `async for msg in consumer`만 하면 모든 복잡한 로직이 **자동 처리**됨
