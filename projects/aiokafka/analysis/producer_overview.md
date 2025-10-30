# Producer 계층 - 메시지 전송

## 📋 개요
- **경로**: `aiokafka/producer/`
- **주요 파일**: `producer.py`, `sender.py`, `message_accumulator.py`, `transaction_manager.py`
- **총 라인 수**: ~2,370줄
- **주요 역할**: Kafka로 메시지 전송 (배치 처리, 압축, 재시도)

## 🎯 핵심 목적
**비동기 메시지 전송**을 제공하며, **배치 처리**, **압축**, **파티셔닝**, **재시도**, **트랜잭션**을 지원하는 **고수준 Producer API**

---

## 🏗️ Producer 아키텍처

```
AIOKafkaProducer (사용자 API)
    ↓ send(topic, value, key)
    │
    ├── Serialization (key/value → bytes)
    ├── Partitioning (파티션 선택)
    └── MessageAccumulator (배치 버퍼)
            ↓
         Sender (백그라운드 태스크)
            ↓
         client.send(ProduceRequest)
            ↓
         conn.send() → Kafka 브로커
```

---

## 📦 주요 컴포넌트

### 1. **AIOKafkaProducer** (producer.py)
- **역할**: 사용자 대면 API
- **핵심 메서드**: `send()`, `flush()`, `start()`, `stop()`

### 2. **MessageAccumulator** (message_accumulator.py)
- **역할**: 파티션별 메시지 버퍼 관리
- **기능**: 배치 생성, 압축, 메모리 관리

### 3. **Sender** (sender.py)
- **역할**: 백그라운드에서 배치 전송
- **기능**: 재시도, 네트워크 I/O, 응답 처리

### 4. **TransactionManager** (transaction_manager.py)
- **역할**: 트랜잭션 및 Idempotence 관리
- **기능**: Producer ID 할당, Sequence 관리

---

## 🔄 AIOKafkaProducer 주요 메서드

### **`__init__()`** - 초기화
```python
producer = AIOKafkaProducer(
    bootstrap_servers='localhost:9092',
    client_id='my-producer',
    key_serializer=lambda k: k.encode('utf-8'),
    value_serializer=lambda v: json.dumps(v).encode('utf-8'),
    acks='all',                    # 0, 1, 'all'
    compression_type='gzip',       # None, 'gzip', 'snappy', 'lz4', 'zstd'
    max_batch_size=16384,          # 16KB
    linger_ms=10,                  # 배치 대기 시간
    max_request_size=1048576,      # 1MB
    enable_idempotence=True,       # 중복 방지
    ...
)
```

**핵심 파라미터**:
| 파라미터 | 설명 | 기본값 |
|----------|------|--------|
| `acks` | 응답 대기 수준 (0, 1, 'all') | 1 (idempotence 시 'all') |
| `compression_type` | 압축 타입 | None |
| `max_batch_size` | 배치 최대 크기 | 16384 (16KB) |
| `linger_ms` | 배치 대기 시간 | 0 (즉시 전송) |
| `enable_idempotence` | 중복 방지 | False |
| `transactional_id` | 트랜잭션 ID | None |

**acks 설정**:
- `0`: 응답 대기 안 함 (빠름, 손실 가능)
- `1`: 리더만 응답 대기 (중간)
- `'all'`: 모든 ISR 응답 대기 (느림, 안전)

---

### **`start()`** - Producer 시작
```python
async def start(self):
    # 1. Kafka 클러스터 연결
    await self.client.bootstrap()

    # 2. 압축 타입 버전 체크
    if self._compression_type == 'lz4':
        assert self.client.api_version >= (0, 8, 2)
    elif self._compression_type == 'zstd':
        assert self.client.api_version >= (2, 1, 0)

    # 3. Sender 태스크 시작
    await self._sender.start()

    # 4. API 버전에 따른 설정
    self._message_accumulator.set_api_version(self.client.api_version)
    self._producer_magic = 0 if self.client.api_version < (0, 10) else 1
```

**역할**:
1. 클라이언트 초기화 및 메타데이터 조회
2. 버전 호환성 체크
3. Sender 백그라운드 태스크 시작
4. 메시지 포맷 버전 설정

---

### **`send()`** - 메시지 전송 ⭐
```python
async def send(
    self,
    topic,
    value=None,
    key=None,
    partition=None,
    timestamp_ms=None,
    headers=None
):
    """비동기 메시지 전송 (즉시 반환, Future로 결과 대기)"""

    # 1. 메타데이터 준비 대기
    await self.client._wait_on_metadata(topic)

    # 2. 트랜잭션 체크 (필요 시)
    if self._txn_manager and not self._txn_manager.is_in_transaction():
        raise IllegalOperation("Can't send messages while not in transaction")

    # 3. 직렬화
    key_bytes, value_bytes = self._serialize(topic, key, value)

    # 4. 파티션 선택
    partition = self._partition(
        topic, partition, key, value, key_bytes, value_bytes
    )

    # 5. MessageAccumulator에 추가 (배치 버퍼)
    tp = TopicPartition(topic, partition)
    fut = await self._message_accumulator.add_message(
        tp, key_bytes, value_bytes,
        timeout=self._request_timeout_ms / 1000,
        timestamp_ms=timestamp_ms,
        headers=headers
    )

    return fut  # Future: 나중에 await하여 결과 확인
```

**실행 흐름**:
```
send("test", value="hello", key="key1")
    ↓
1. _wait_on_metadata("test")       # 토픽 메타데이터 준비
    ↓
2. _serialize(key, value)          # bytes로 변환
    ↓
3. _partition(key_bytes, ...)      # 파티션 선택 (partitioner 사용)
    ↓
4. _message_accumulator.add_message()  # 배치 버퍼에 추가
    ↓
5. return Future                   # 응답 대기용 Future 반환
```

**Future 사용 예시**:
```python
# 비동기 전송 (버퍼에 추가만)
fut = await producer.send("test", value="hello")

# 나중에 결과 대기
metadata = await fut  # RecordMetadata(topic, partition, offset)
print(f"Sent to {metadata.topic}-{metadata.partition} offset {metadata.offset}")
```

---

### **`send_and_wait()`** - 동기식 전송
```python
async def send_and_wait(self, topic, value=None, key=None, ...):
    """메시지 전송 + 즉시 결과 대기"""
    future = await self.send(topic, value, key, ...)
    return await future
```

**차이점**:
```python
# send(): 빠름 (배치 버퍼에만 추가)
fut = await producer.send("test", value="hello")
# ... 다른 작업 ...
result = await fut  # 나중에 결과 확인

# send_and_wait(): 느림 (전송 완료까지 대기)
result = await producer.send_and_wait("test", value="hello")
```

---

### **`flush()`** - 버퍼 비우기
```python
async def flush(self):
    """대기 중인 모든 배치 전송 완료까지 대기"""
    await self._message_accumulator.flush()
```

**사용 시점**:
- Producer 종료 전
- 중요한 메시지 전송 후 확실한 전송 보장 필요 시

---

### **`stop()`** - Producer 종료
```python
async def stop(self):
    if self._closed:
        return
    self._closed = True

    # 1. MessageAccumulator 종료 + Sender 태스크 대기
    await asyncio.wait(
        [
            create_task(self._message_accumulator.close()),
            self._sender.sender_task
        ],
        return_when=asyncio.FIRST_COMPLETED
    )

    # 2. Sender 종료
    await self._sender.close()

    # 3. Client 종료
    await self.client.close()
```

**종료 순서**:
1. MessageAccumulator 닫기 (새 메시지 거부)
2. Sender 태스크 종료
3. 네트워크 연결 종료

---

### **파티셔닝 로직**

#### **`_partition()`** - 파티션 선택
```python
def _partition(self, topic, partition, key, value, serialized_key, serialized_value):
    # 1. partition 명시 시 사용
    if partition is not None:
        assert partition in self._metadata.partitions_for_topic(topic)
        return partition

    # 2. partitioner 사용 (key 기반 해싱)
    all_partitions = list(self._metadata.partitions_for_topic(topic))
    available = list(self._metadata.available_partitions_for_topic(topic))
    return self._partitioner(serialized_key, all_partitions, available)
```

**DefaultPartitioner 동작**:
```python
# key 있음: 동일 key → 동일 partition (murmur2 해시)
partitioner(key=b"user123", all=[0,1,2], available=[0,1,2])
→ hash(b"user123") % 3 = 1  # 항상 partition 1

# key 없음: available 중 랜덤 선택
partitioner(key=None, all=[0,1,2], available=[0,2])
→ random.choice([0, 2])  # partition 1은 리더 없음
```

---

### **직렬화**

#### **`_serialize()`** - Key/Value 직렬화
```python
def _serialize(self, topic, key, value):
    # key 직렬화
    if self._key_serializer is None:
        serialized_key = key  # 이미 bytes
    else:
        serialized_key = self._key_serializer(key)

    # value 직렬화
    if self._value_serializer is None:
        serialized_value = value
    else:
        serialized_value = self._value_serializer(value)

    # 크기 체크
    message_size = ...  # 계산
    if message_size > self._max_request_size:
        raise MessageSizeTooLargeError(...)

    return serialized_key, serialized_value
```

**사용 예시**:
```python
import json

producer = AIOKafkaProducer(
    key_serializer=lambda k: k.encode('utf-8'),
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

await producer.send("test", key="user123", value={"name": "John"})
# → key=b"user123", value=b'{"name":"John"}'
```

---

## 🔐 트랜잭션 지원

### **트랜잭션 메서드**
```python
# 트랜잭션 시작
await producer.begin_transaction()

# 메시지 전송
await producer.send("topic1", value="msg1")
await producer.send("topic2", value="msg2")

# Consumer 오프셋 커밋 (Exactly-Once Semantics)
await producer.send_offsets_to_transaction(
    offsets={TopicPartition("input", 0): OffsetAndMetadata(100, "")},
    group_id="my-group"
)

# 트랜잭션 커밋
await producer.commit_transaction()

# 실패 시 롤백
# await producer.abort_transaction()
```

**Context Manager 사용**:
```python
async with producer.transaction():
    await producer.send("topic1", value="msg1")
    await producer.send("topic2", value="msg2")
    # 자동 커밋 (예외 시 자동 롤백)
```

---

## 📊 메시지 흐름

### 전체 흐름
```
User Code
    ↓
await producer.send("test", value="hello", key="user1")
    ↓
Producer._serialize()  # key, value → bytes
    ↓
Producer._partition()  # partition 선택 (hash(key))
    ↓
MessageAccumulator.add_message()  # 배치 버퍼에 추가
    ↓
[배치 버퍼]
    ├── test-0: [msg1, msg2, msg3]
    ├── test-1: [msg4, msg5]
    └── test-2: [msg6]
    ↓
Sender (백그라운드 태스크)
    ↓
ProduceRequest 생성 (배치 → RecordBatch)
    ↓
client.send(ProduceRequest)
    ↓
conn.send() → Kafka 브로커
    ↓
ProduceResponse 수신
    ↓
Future.set_result(RecordMetadata)
    ↓
await fut → RecordMetadata(topic, partition, offset)
```

---

## ⚙️ 핵심 설계 패턴

### 1. **비동기 배치 처리**
```python
# 메시지는 즉시 버퍼에 추가 (빠름)
fut1 = await producer.send("test", value="msg1")
fut2 = await producer.send("test", value="msg2")
fut3 = await producer.send("test", value="msg3")

# 백그라운드에서 배치로 전송
# [msg1, msg2, msg3] → 하나의 ProduceRequest

# 나중에 결과 확인
results = await asyncio.gather(fut1, fut2, fut3)
```

### 2. **Future 기반 응답 처리**
```python
# send()는 Future 반환 (블로킹 안 함)
fut = await producer.send("test", value="hello")
# → Future<RecordMetadata>

# 다른 작업 수행 가능
# ...

# 나중에 결과 대기
metadata = await fut
print(metadata.offset)
```

### 3. **백그라운드 Sender 태스크**
```python
# start() 시 Sender 태스크 시작
self._sender_task = create_task(self._sender_loop())

# Sender 루프
async def _sender_loop(self):
    while not self._closed:
        # 1. 배치 가져오기
        batch = await self._accumulator.get_batch()

        # 2. ProduceRequest 생성 및 전송
        response = await self.client.send(request)

        # 3. Future 완료
        for fut in batch.futures:
            fut.set_result(metadata)
```

---

## 🔑 핵심 특징 요약

| 특징 | 설명 |
|------|------|
| **비동기 I/O** | asyncio 기반 non-blocking 전송 |
| **배치 처리** | 여러 메시지를 하나의 요청으로 전송 (성능 향상) |
| **압축** | gzip, snappy, lz4, zstd 지원 |
| **파티셔닝** | key 기반 해싱 (동일 key → 동일 partition) |
| **재시도** | 실패 시 자동 재시도 (Sender 담당) |
| **Idempotence** | 중복 메시지 방지 (Sequence 관리) |
| **트랜잭션** | Exactly-Once Semantics 지원 |
| **Back-pressure** | 버퍼 가득 차면 send() 블로킹 |

---

## 🎓 결과적으로 Producer 계층은

**Kafka로 메시지를 전송하는 고수준 API**로서:
1. ✅ **비동기 전송** (Future 기반)
2. ✅ **배치 처리** (MessageAccumulator)
3. ✅ **압축** (gzip, snappy, lz4, zstd)
4. ✅ **파티셔닝** (key 기반 해싱)
5. ✅ **재시도** (Sender)
6. ✅ **Idempotence** (중복 방지)
7. ✅ **트랜잭션** (Exactly-Once)

→ 사용자는 `await producer.send()`만 호출하면 모든 복잡한 로직이 **자동 처리**됨
