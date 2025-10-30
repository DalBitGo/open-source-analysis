# aiokafka 전체 아키텍처 요약

## 📚 프로젝트 개요
- **프로젝트**: aiokafka
- **저장소**: https://github.com/aio-libs/aiokafka
- **목적**: **순수 Python asyncio 기반 Kafka 클라이언트**
- **특징**: librdkafka 없이 Kafka Wire Protocol을 직접 구현

---

## 🏛️ 전체 계층 구조

```
┌──────────────────────────────────────────────────────────┐
│  User Application                                         │
│  (Producer/Consumer API 사용)                             │
└─────────────────────┬────────────────────────────────────┘
                      │
┌─────────────────────┴────────────────────────────────────┐
│  Application Layer (Producer & Consumer)                  │
│  ┌─────────────────────────────┬──────────────────────┐  │
│  │  Producer                   │  Consumer            │  │
│  │  - AIOKafkaProducer         │  - AIOKafkaConsumer  │  │
│  │  - MessageAccumulator       │  - Fetcher           │  │
│  │  - Sender                   │  - GroupCoordinator  │  │
│  │  - TransactionManager       │  - SubscriptionState │  │
│  └─────────────────────────────┴──────────────────────┘  │
└─────────────────────┬────────────────────────────────────┘
                      │
┌─────────────────────┴────────────────────────────────────┐
│  Client Layer (Cluster Management)                        │
│  - AIOKafkaClient (연결 풀, 메타데이터 동기화)            │
│  - ClusterMetadata (브로커, 토픽, 파티션 정보 저장)       │
└─────────────────────┬────────────────────────────────────┘
                      │
┌─────────────────────┴────────────────────────────────────┐
│  Protocol Layer (Kafka Wire Protocol)                     │
│  - types.py (Int32, String, Array, Schema 등)            │
│  - struct.py (Request/Response 베이스)                   │
│  - metadata.py, produce.py, fetch.py 등 (46개 API)       │
└─────────────────────┬────────────────────────────────────┘
                      │
┌─────────────────────┴────────────────────────────────────┐
│  Network Layer (TCP/TLS + SASL)                          │
│  - AIOKafkaConnection (asyncio StreamReader/Writer)      │
│  - SASL 인증 (PLAIN, GSSAPI, SCRAM, OAUTHBEARER)        │
│  - Correlation ID 매칭, 타임아웃 관리                     │
└──────────────────────────────────────────────────────────┘
```

---

## 📂 주요 모듈 상세

### 🔹 **Network Layer** (conn.py)
**역할**: Kafka 브로커와의 TCP 연결 관리

**핵심 클래스**:
- `AIOKafkaConnection`: 단일 브로커 연결
  - asyncio StreamReader/Writer 사용
  - Correlation ID로 요청/응답 매칭
  - SASL 인증 (PLAIN, GSSAPI, SCRAM-SHA-256/512, OAUTHBEARER)
  - API 버전 협상
  - 유휴 연결 타임아웃

**특징**:
- ✅ Big-Endian 바이트 순서
- ✅ Weakref로 메모리 릭 방지
- ✅ CloseReason으로 종료 원인 추적
- ✅ Future 기반 비동기 응답 처리

---

### 🔹 **Client Layer** (client.py, cluster.py)

#### **AIOKafkaClient** (client.py)
**역할**: 클러스터 연결 풀 및 메타데이터 관리

**핵심 기능**:
- 연결 풀 관리: `_conns = {(node_id, group): AIOKafkaConnection}`
- 백그라운드 메타데이터 동기화 (`_md_synchronizer` 태스크)
- API 버전 자동 감지 (`check_version()`)
- Coordinator 조회 (Group/Transaction)

**메타데이터 갱신**:
```
주기적 (metadata_max_age_ms: 5분)
    or
강제 (토픽 추가, 연결 실패 시)
    ↓
MetadataRequest 전송
    ↓
cluster.update_metadata(response)
```

#### **ClusterMetadata** (cluster.py)
**역할**: 메타데이터 저장소 (IO 없음)

**저장 데이터**:
```python
_brokers = {node_id: BrokerMetadata}
_partitions = {topic: {partition: PartitionMetadata}}
_broker_partitions = {node_id: {TopicPartition}}
_coordinators = {node_id: BrokerMetadata}
```

**특징**:
- ✅ 순수 데이터 구조 (네트워크 없음)
- ✅ threading.Lock으로 스레드 안전
- ✅ Observer 패턴 (리스너 콜백)

---

### 🔹 **Protocol Layer** (protocol/)

#### **타입 시스템** (types.py)
**역할**: Kafka Wire Protocol 기본 타입 구현

**지원 타입**:
```
기본: Int8, Int16, Int32, UInt32, Int64, Boolean, Float64
가변: String, Bytes, Array, Schema
VarInt: UnsignedVarInt32, VarInt32, VarInt64
Compact: CompactString, CompactBytes, CompactArray, TaggedFields
```

**인코딩 예시**:
```python
String("test") → b'\x00\x04test'
Array([1, 2, 3]) → b'\x00\x00\x00\x03\x00\x00\x00\x01\x00\x00\x00\x02\x00\x00\x00\x03'
```

#### **Request/Response** (struct.py, api.py)
**역할**: 모든 프로토콜의 베이스 클래스

**구조**:
```python
class MetadataRequest_v1(Request):
    API_KEY = 3
    API_VERSION = 1
    RESPONSE_TYPE = MetadataResponse_v1
    SCHEMA = Schema(
        ('topics', Array(String('utf-8')))
    )
```

**메시지 형식**:
```
[4-byte size][RequestHeader][RequestBody]

RequestHeader:
  - api_key (Int16)
  - api_version (Int16)
  - correlation_id (Int32)
  - client_id (String)
```

#### **46개 API 프로토콜**
- **Core**: Metadata, Produce, Fetch, ListOffsets
- **Consumer**: FindCoordinator, JoinGroup, SyncGroup, Heartbeat, OffsetCommit, OffsetFetch
- **Transaction**: InitProducerId, AddPartitionsToTxn, EndTxn
- **Admin**: CreateTopics, DeleteTopics, DescribeConfigs, AlterConfigs 등

---

### 🔹 **Producer Layer** (producer/)

#### **AIOKafkaProducer** (producer.py)
**역할**: 메시지 전송 API

**핵심 흐름**:
```
await producer.send("test", value="hello", key="user1")
    ↓
1. _serialize(key, value) → bytes
2. _partition(key_bytes) → partition (hash 기반)
3. _message_accumulator.add_message() → 배치 버퍼
    ↓
Sender (백그라운드 태스크)
    ↓
ProduceRequest 전송 → ProduceResponse
    ↓
Future.set_result(RecordMetadata)
```

**핵심 컴포넌트**:
- **MessageAccumulator**: 파티션별 배치 버퍼, 압축
- **Sender**: 백그라운드 전송 태스크, 재시도
- **TransactionManager**: 트랜잭션 및 Idempotence

**주요 설정**:
| 설정 | 설명 | 기본값 |
|------|------|--------|
| `acks` | 응답 대기 수준 (0, 1, 'all') | 1 |
| `compression_type` | 압축 (gzip, snappy, lz4, zstd) | None |
| `max_batch_size` | 배치 최대 크기 | 16KB |
| `linger_ms` | 배치 대기 시간 | 0 (즉시) |
| `enable_idempotence` | 중복 방지 | False |

---

### 🔹 **Consumer Layer** (consumer/)

#### **AIOKafkaConsumer** (consumer.py)
**역할**: 메시지 수신 API

**핵심 흐름**:
```
await consumer.subscribe(['test'])
    ↓
GroupCoordinator
    ↓
1. FindCoordinator (Coordinator 찾기)
2. JoinGroup (그룹 가입, Leader 선출)
3. SyncGroup (파티션 할당)
4. Heartbeat (백그라운드)
    ↓
msg = await consumer.getone()
    ↓
Fetcher
    ↓
1. FetchRequest 전송 (Prefetch)
2. FetchResponse 파싱 (RecordBatch)
3. 역직렬화
    ↓
return ConsumerRecord
```

**핵심 컴포넌트**:
- **GroupCoordinator**: Consumer Group 프로토콜 관리
- **Fetcher**: FetchRequest, Prefetch 버퍼, 역직렬화
- **SubscriptionState**: 파티션별 오프셋 추적

**주요 설정**:
| 설정 | 설명 | 기본값 |
|------|------|--------|
| `group_id` | Consumer Group ID | None |
| `auto_offset_reset` | 오프셋 없을 때 ('earliest', 'latest') | 'latest' |
| `enable_auto_commit` | 자동 커밋 여부 | True |
| `max_poll_records` | 한 번 fetch 최대 레코드 | 500 |

---

## 🔄 주요 실행 흐름

### **Producer 메시지 전송**
```
┌─────────────────┐
│  User Code      │
└────────┬────────┘
         │ await producer.send("test", value="hello")
         ↓
┌─────────────────────────────────────┐
│  AIOKafkaProducer                   │
│  1. Serialize (key, value → bytes) │
│  2. Partition (hash(key) % 3)      │
│  3. Add to MessageAccumulator      │
└────────┬────────────────────────────┘
         │ (버퍼에 추가, Future 반환)
         ↓
┌─────────────────────────────────────┐
│  Sender (백그라운드)                 │
│  1. 배치 수집 (linger_ms 대기)      │
│  2. 압축 (gzip)                     │
│  3. ProduceRequest 생성             │
└────────┬────────────────────────────┘
         │
         ↓
┌─────────────────────────────────────┐
│  AIOKafkaClient                     │
│  - node_id 선택 (partition leader)  │
│  - conn.send(ProduceRequest)       │
└────────┬────────────────────────────┘
         │
         ↓
┌─────────────────────────────────────┐
│  AIOKafkaConnection                 │
│  - header + body 인코딩             │
│  - TCP 전송                         │
│  - 응답 대기 (correlation_id)       │
└────────┬────────────────────────────┘
         │
         ↓
┌─────────────────────────────────────┐
│  ProduceResponse                    │
│  - partition, offset, error_code    │
│  - Future.set_result(metadata)     │
└─────────────────────────────────────┘
         │
         ↓
┌─────────────────┐
│  User Code      │
│  metadata = await fut  │
│  → RecordMetadata(offset=123) │
└─────────────────┘
```

### **Consumer 메시지 수신**
```
┌─────────────────┐
│  User Code      │
└────────┬────────┘
         │ await consumer.subscribe(['test'])
         ↓
┌─────────────────────────────────────┐
│  GroupCoordinator                   │
│  1. FindCoordinator                │
│  2. JoinGroup (Leader 선출)        │
│  3. SyncGroup (파티션 할당)        │
│  4. Heartbeat 시작 (백그라운드)    │
└────────┬────────────────────────────┘
         │ (파티션 할당 완료)
         ↓
┌─────────────────┐
│  User Code      │
└────────┬────────┘
         │ msg = await consumer.getone()
         ↓
┌─────────────────────────────────────┐
│  Fetcher                            │
│  1. Prefetch 버퍼 확인              │
│  2. 버퍼 없으면 FetchRequest 전송   │
└────────┬────────────────────────────┘
         │
         ↓
┌─────────────────────────────────────┐
│  AIOKafkaClient                     │
│  - conn.send(FetchRequest)         │
└────────┬────────────────────────────┘
         │
         ↓
┌─────────────────────────────────────┐
│  FetchResponse                      │
│  - RecordBatch 파싱                 │
│  - 압축 해제                        │
│  - 역직렬화                         │
│  - Prefetch 버퍼에 저장             │
└────────┬────────────────────────────┘
         │
         ↓
┌─────────────────┐
│  User Code      │
│  msg = ConsumerRecord(...)  │
│  → msg.value = "hello"      │
└─────────────────┘
```

---

## 🎯 핵심 특징

### **1. 순수 Python 구현**
- ✅ librdkafka 없음 → C 의존성 제거
- ✅ Kafka Wire Protocol 완전 구현
- ✅ 타입 시스템 (Int32, String, Array 등)
- ✅ 46개 API 프로토콜 지원

### **2. asyncio 네이티브**
- ✅ async/await 문법
- ✅ asyncio StreamReader/Writer
- ✅ Future 기반 응답 처리
- ✅ 백그라운드 태스크 (Sender, Heartbeat, Metadata)

### **3. 프로덕션 기능**
- ✅ **압축**: gzip, snappy, lz4, zstd
- ✅ **배치 처리**: MessageAccumulator
- ✅ **재시도**: Sender 자동 재시도
- ✅ **Idempotence**: 중복 방지 (Sequence)
- ✅ **트랜잭션**: Exactly-Once Semantics
- ✅ **Consumer Group**: 파티션 자동 할당, 리밸런싱
- ✅ **오프셋 관리**: 자동/수동 커밋
- ✅ **SASL 인증**: PLAIN, GSSAPI, SCRAM, OAUTHBEARER
- ✅ **SSL/TLS**: 암호화 통신

### **4. 성능 최적화**
- ✅ **Prefetch**: Consumer가 미리 메시지 가져옴
- ✅ **배치 전송**: 여러 메시지를 하나의 요청으로
- ✅ **압축**: 네트워크 대역폭 절약
- ✅ **연결 풀**: 브로커별 연결 재사용
- ✅ **Back-pressure**: 버퍼 제한으로 메모리 보호

---

## 📖 학습 포인트

### **Kafka 프로토콜 이해**
1. **Wire Protocol**: Big-Endian, VarInt, Compact 타입
2. **API 버전 관리**: MetadataRequest v0 ~ v5
3. **Correlation ID**: 요청/응답 매칭
4. **Consumer Group**: JoinGroup, SyncGroup, Heartbeat
5. **트랜잭션**: InitProducerId, AddPartitionsToTxn

### **asyncio 패턴**
1. **비동기 I/O**: StreamReader/Writer
2. **Future**: 비동기 응답 대기
3. **백그라운드 태스크**: create_task()
4. **Weakref**: 순환 참조 방지
5. **Context Manager**: async with

### **아키텍처 설계**
1. **계층 분리**: Network → Protocol → Client → Application
2. **Observer 패턴**: 메타데이터 리스너
3. **Template Method**: Struct 베이스 클래스
4. **Strategy**: Partitioner, Serializer
5. **배치 처리**: MessageAccumulator

---

## 📊 분석 문서 목록

### **Core 계층**
1. `conn.py.md` - TCP 연결 & SASL 인증
2. `client.py.md` - 연결 풀 & 메타데이터 동기화
3. `cluster.py.md` - 메타데이터 저장소

### **Protocol 계층**
4. `protocol_overview.md` - 전체 개요 (46개 API)
5. `protocol_types.py.md` - 타입 시스템
6. `protocol_struct_api.md` - Request/Response 베이스
7. `protocol_metadata.py.md` - Metadata API 예시

### **Producer 계층**
8. `producer_overview.md` - Producer API & 메시지 전송

### **Consumer 계층**
9. `consumer_overview.md` - Consumer API & 메시지 수신

### **전체 요약**
10. **`00_ARCHITECTURE_SUMMARY.md`** (이 문서)

---

## 🎓 결론

**aiokafka**는:
1. ✅ **순수 Python**으로 Kafka를 완전히 구현한 프로젝트
2. ✅ **asyncio 네이티브**로 고성능 비동기 I/O 제공
3. ✅ **프로덕션 기능** 완비 (트랜잭션, Idempotence, Consumer Group)
4. ✅ **교육적 가치**: Kafka 프로토콜 및 asyncio 패턴 학습

→ **Python asyncio + Kafka**를 배우려면 **필수로 분석해야 할 프로젝트**!
