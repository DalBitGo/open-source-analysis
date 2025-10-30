# Protocol 계층 - 전체 개요

## 📋 개요
- **경로**: `aiokafka/protocol/`
- **총 파일 수**: 15개 (.py)
- **총 라인 수**: ~4,500줄
- **주요 역할**: Kafka Wire Protocol 구현 (요청/응답 직렬화/역직렬화)

## 🎯 핵심 목적
**순수 Python으로 Kafka Wire Protocol을 구현**하여, 네트워크 바이트 스트림을 구조화된 Request/Response 객체로 변환하는 **프로토콜 계층**

---

## 🏗️ 전체 구조

```
protocol/
│
├── 📦 타입 시스템 (Type System)
│   ├── abstract.py (19줄)    - AbstractType 인터페이스
│   └── types.py (420줄)      - Int8, Int16, String, Array 등 기본 타입
│
├── 🔧 베이스 클래스 (Base Classes)
│   ├── struct.py (56줄)      - Request, Response 베이스
│   └── api.py (149줄)        - API_KEY 정의 및 버전 관리
│
├── 📨 핵심 프로토콜 (Core Protocols)
│   ├── metadata.py (272줄)   - Metadata 요청/응답 (클러스터 정보)
│   ├── produce.py (297줄)    - Produce 요청/응답 (메시지 전송)
│   └── fetch.py (520줄)      - Fetch 요청/응답 (메시지 수신)
│
├── 🧩 Consumer 프로토콜
│   ├── coordination.py (44줄) - FindCoordinator 요청/응답
│   ├── group.py (275줄)      - JoinGroup, SyncGroup, Heartbeat, LeaveGroup
│   ├── commit.py (312줄)     - OffsetCommit, OffsetFetch, GroupCoordinator
│   └── offset.py (246줄)     - ListOffsets (파티션 오프셋 조회)
│
├── 🔐 Transaction 프로토콜
│   └── transaction.py (150줄) - InitProducerId, AddPartitionsToTxn 등
│
├── ⚙️ Admin 프로토콜
│   └── admin.py (1423줄)     - CreateTopics, DeleteTopics, DescribeConfigs 등 30+ APIs
│
└── 📜 레거시/유틸
    └── message.py (324줄)    - MessageSet (v0/v1 메시지 포맷, 현재는 record 사용)
```

---

## 📚 파일별 상세 역할

### 🔹 타입 시스템

#### **abstract.py** (19줄)
```python
class AbstractType(Generic[T]):
    @abstractmethod
    def encode(cls, value: T) -> bytes: ...

    @abstractmethod
    def decode(cls, data: BytesIO) -> T: ...
```
- **역할**: 모든 프로토콜 타입의 추상 인터페이스
- **메서드**: `encode()`, `decode()`, `repr()`

#### **types.py** (420줄) ⭐
- **역할**: Kafka Wire Protocol의 기본 타입 구현
- **타입 종류**:
  ```
  Int8, Int16, Int32, Int64
  UInt32
  Boolean
  String, NullableString, CompactString
  Bytes, NullableBytes, CompactBytes
  Array, CompactArray
  Schema (복합 타입)
  TaggedFields (Kafka 2.4+ Flexible API)
  ```
- **특징**: Big-endian 인코딩, varint (compact), tagged fields 지원

---

### 🔹 베이스 클래스

#### **struct.py** (56줄) ⭐
```python
class Struct:
    SCHEMA = Schema(...)  # 서브클래스에서 정의

    def encode(self) -> bytes:
        return self.SCHEMA.encode(...)

    @classmethod
    def decode(cls, data: BytesIO):
        return cls(*cls.SCHEMA.decode(data))
```
- **역할**: Request/Response의 베이스 클래스
- **핵심 메서드**:
  - `encode()`: 객체 → 바이트
  - `decode()`: 바이트 → 객체
  - `build_request_header()`: 요청 헤더 생성

#### **api.py** (149줄) ⭐
```python
class RequestHeader:
    SCHEMA = Schema(
        ('api_key', Int16),
        ('api_version', Int16),
        ('correlation_id', Int32),
        ('client_id', NullableString('utf-8'))
    )

class Request:
    API_KEY = None        # 서브클래스에서 정의 (예: 3 for Metadata)
    API_VERSION = None    # 서브클래스에서 정의 (예: 0, 1, 2)
    RESPONSE_TYPE = None  # 대응하는 Response 클래스
    SCHEMA = Schema(...)  # 요청 필드 스키마
```
- **역할**: API 키, 버전, 헤더 정의
- **API_KEYS**: 46개 API 키 매핑 (Produce=0, Fetch=1, Metadata=3, ...)

---

### 🔹 핵심 프로토콜

#### **metadata.py** (272줄)
```python
MetadataRequest_v0 = Struct(
    ('topics', Array(String('utf-8')))
)

MetadataRequest_v1 = Struct(
    ('topics', Array(String('utf-8'), nullable=True))  # null = all topics
)

MetadataResponse_v0 = Struct(
    ('brokers', Array(...)),
    ('topics', Array(...))
)

# 버전별 클래스 생성
MetadataRequest = [MetadataRequest_v0, MetadataRequest_v1, ...]
```
- **역할**: 클러스터 메타데이터 조회 (브로커, 토픽, 파티션)
- **버전**: v0 ~ v11 (12개 버전)
- **사용처**: `client.py`의 `bootstrap()`, `_metadata_update()`

#### **produce.py** (297줄)
```python
ProduceRequest_v0 = Struct(
    ('required_acks', Int16),
    ('timeout', Int32),
    ('topics', Array(
        ('topic', String('utf-8')),
        ('partitions', Array(
            ('partition', Int32),
            ('messages', Bytes)
        ))
    ))
)

ProduceResponse_v0 = Struct(
    ('topics', Array(
        ('topic', String('utf-8')),
        ('partitions', Array(
            ('partition', Int32),
            ('error_code', Int16),
            ('offset', Int64)
        ))
    ))
)
```
- **역할**: 메시지 전송 요청/응답
- **버전**: v0 ~ v9 (10개 버전)
- **핵심 필드**: `required_acks`, `timeout`, `messages`

#### **fetch.py** (520줄)
```python
FetchRequest_v0 = Struct(
    ('replica_id', Int32),
    ('max_wait_time', Int32),
    ('min_bytes', Int32),
    ('topics', Array(
        ('topic', String('utf-8')),
        ('partitions', Array(
            ('partition', Int32),
            ('fetch_offset', Int64),
            ('max_bytes', Int32)
        ))
    ))
)

FetchResponse_v0 = Struct(
    ('topics', Array(
        ('topic', String('utf-8')),
        ('partitions', Array(
            ('partition', Int32),
            ('error_code', Int16),
            ('highwater_offset', Int64),
            ('messages', Bytes)
        ))
    ))
)
```
- **역할**: 메시지 fetch 요청/응답
- **버전**: v0 ~ v13 (14개 버전)
- **핵심 필드**: `max_wait_time`, `min_bytes`, `fetch_offset`

---

### 🔹 Consumer 프로토콜

#### **coordination.py** (44줄)
```python
FindCoordinatorRequest_v0 = Struct(
    ('coordinator_key', String('utf-8'))  # group_id or transactional_id
)

FindCoordinatorRequest_v1 = Struct(
    ('coordinator_key', String('utf-8')),
    ('coordinator_type', Int8)  # 0=GROUP, 1=TRANSACTION
)
```
- **역할**: Group/Transaction Coordinator 조회
- **버전**: v0, v1, v2

#### **group.py** (275줄)
```python
JoinGroupRequest_v0 = Struct(
    ('group_id', String('utf-8')),
    ('session_timeout', Int32),
    ('member_id', String('utf-8')),
    ('protocol_type', String('utf-8')),  # "consumer"
    ('group_protocols', Array(...))
)

SyncGroupRequest_v0 = Struct(
    ('group_id', String('utf-8')),
    ('generation_id', Int32),
    ('member_id', String('utf-8')),
    ('group_assignment', Array(...))
)

HeartbeatRequest_v0 = Struct(
    ('group_id', String('utf-8')),
    ('generation_id', Int32),
    ('member_id', String('utf-8'))
)

LeaveGroupRequest_v0 = Struct(
    ('group_id', String('utf-8')),
    ('member_id', String('utf-8'))
)
```
- **역할**: Consumer Group 관리 (가입, 동기화, 하트비트, 탈퇴)
- **프로토콜**: JoinGroup, SyncGroup, Heartbeat, LeaveGroup, DescribeGroups, ListGroups

#### **commit.py** (312줄)
```python
OffsetCommitRequest_v0 = Struct(
    ('group_id', String('utf-8')),
    ('topics', Array(
        ('topic', String('utf-8')),
        ('partitions', Array(
            ('partition', Int32),
            ('offset', Int64),
            ('metadata', NullableString('utf-8'))
        ))
    ))
)

OffsetFetchRequest_v0 = Struct(
    ('group_id', String('utf-8')),
    ('topics', Array(
        ('topic', String('utf-8')),
        ('partitions', Array(Int32))
    ))
)

GroupCoordinatorRequest_v0 = Struct(
    ('group_id', String('utf-8'))
)
```
- **역할**: 오프셋 커밋/조회, Group Coordinator 조회 (구형)
- **버전**: v0 ~ v8 (다양한 버전)

#### **offset.py** (246줄)
```python
ListOffsetsRequest_v0 = Struct(
    ('replica_id', Int32),
    ('topics', Array(
        ('topic', String('utf-8')),
        ('partitions', Array(
            ('partition', Int32),
            ('timestamp', Int64),  # -1=latest, -2=earliest
            ('max_offsets', Int32)
        ))
    ))
)
```
- **역할**: 특정 타임스탬프의 오프셋 조회
- **사용처**: Consumer의 `seek_to_beginning()`, `seek_to_end()`

---

### 🔹 Transaction 프로토콜

#### **transaction.py** (150줄)
```python
InitProducerIdRequest_v0 = Struct(
    ('transactional_id', NullableString('utf-8')),
    ('transaction_timeout_ms', Int32)
)

AddPartitionsToTxnRequest_v0 = Struct(
    ('transactional_id', String('utf-8')),
    ('producer_id', Int64),
    ('producer_epoch', Int16),
    ('topics', Array(...))
)

AddOffsetsToTxnRequest_v0 = Struct(...)
EndTxnRequest_v0 = Struct(...)
```
- **역할**: 트랜잭션 관리 (Kafka 0.11+)
- **프로토콜**: InitProducerId, AddPartitionsToTxn, AddOffsetsToTxn, EndTxn, TxnOffsetCommit

---

### 🔹 Admin 프로토콜

#### **admin.py** (1423줄) 🔥
```python
# 30+ Admin APIs:
CreateTopicsRequest, CreateTopicsResponse
DeleteTopicsRequest, DeleteTopicsResponse
CreatePartitionsRequest, CreatePartitionsResponse
DescribeConfigsRequest, DescribeConfigsResponse
AlterConfigsRequest, AlterConfigsResponse
DescribeAclsRequest, DescribeAclsResponse
CreateAclsRequest, CreateAclsResponse
DeleteAclsRequest, DeleteAclsResponse
DescribeClientQuotasRequest, ...
ApiVersionRequest, ApiVersionResponse
SaslHandshakeRequest, SaslHandshakeResponse
SaslAuthenticateRequest, SaslAuthenticateResponse
...
```
- **역할**: 관리 작업 (토픽 생성/삭제, 설정 변경, ACL, SASL 등)
- **특징**: 가장 많은 API 포함

---

### 🔹 레거시/유틸

#### **message.py** (324줄)
```python
class MessageSet:
    # Kafka v0/v1 메시지 포맷 (현재는 RecordBatch 사용)
    HEADER_STRUCT = struct.Struct('>q')  # offset
    MESSAGE_STRUCT = struct.Struct('>i')  # size
```
- **역할**: 구형 메시지 포맷 (Kafka 0.10 이전)
- **현재**: `aiokafka/record/` 모듈 사용 (v2 RecordBatch)

---

## 🔄 프로토콜 버전 관리

### 버전별 클래스 생성 패턴
```python
# metadata.py 예시
MetadataRequest_v0 = Struct(...)
MetadataRequest_v1 = Struct(...)
MetadataRequest_v2 = Struct(...)

# 리스트로 관리
MetadataRequest = [
    MetadataRequest_v0,
    MetadataRequest_v1,
    MetadataRequest_v2,
    ...
]

# 사용 시
request = MetadataRequest[1](topics=['test'])  # v1 사용
```

### 버전 협상
```python
# conn.py의 VersionInfo
version_info.pick_best([
    MetadataRequest_v11,
    MetadataRequest_v10,
    ...
    MetadataRequest_v0,
])
# → 브로커가 지원하는 가장 높은 버전 선택
```

---

## 🎨 Kafka Wire Protocol 기본 개념

### 메시지 구조
```
[4-byte size][Request Header][Request Body]

Request Header:
  - api_key (Int16)
  - api_version (Int16)
  - correlation_id (Int32)
  - client_id (NullableString)

Response:
  - correlation_id (Int32)
  - [Response Body]
```

### 타입 인코딩
| 타입 | 크기 | 예시 |
|------|------|------|
| Int8 | 1 byte | `0x01` |
| Int16 | 2 bytes | `0x00 0x03` (big-endian) |
| Int32 | 4 bytes | `0x00 0x00 0x00 0x0A` |
| String | 2-byte length + UTF-8 | `0x00 0x04 test` |
| Array | 4-byte length + elements | `0x00 0x00 0x00 0x02 [elem1][elem2]` |

### Compact 타입 (Kafka 2.4+)
```python
# Varint 인코딩 (가변 길이)
CompactString  # length: varint, null = 0
CompactArray   # length: varint
TaggedFields   # key-value pairs with tags
```

---

## 🔗 다른 계층과의 관계

```
┌─────────────────────────────────────┐
│  Producer / Consumer / Admin        │
│  (비즈니스 로직)                      │
└────────────┬────────────────────────┘
             │
             ↓
┌─────────────────────────────────────┐
│  client.py                           │
│  (연결 관리, 메타데이터)               │
└────────────┬────────────────────────┘
             │
             ↓ send(request)
┌─────────────────────────────────────┐
│  conn.py                             │
│  (TCP 연결, 요청/응답 매칭)            │
└────────────┬────────────────────────┘
             │
             ↓ encode() / decode()
┌─────────────────────────────────────┐
│  protocol/ (이 계층)                  │
│  (Wire Protocol 직렬화/역직렬화)       │
└─────────────────────────────────────┘
             │
             ↓
    [네트워크 바이트 스트림]
```

### 사용 예시
```python
# client.py
from aiokafka.protocol.metadata import MetadataRequest

# 요청 객체 생성
request = MetadataRequest[1](topics=['test'])

# conn.py에서 인코딩 후 전송
header = request.build_request_header(correlation_id=1, client_id="aiokafka")
message = header.encode() + request.encode()
# → b'\x00\x00\x00\x1a\x00\x03\x00\x01...'

# 응답 수신 후 디코딩
response = MetadataRequest[1].RESPONSE_TYPE.decode(BytesIO(resp_bytes))
# → MetadataResponse(brokers=[...], topics=[...])
```

---

## ⚙️ 핵심 설계 패턴

### 1. **Schema 기반 선언적 정의**
```python
# types.py
Schema(
    ('field1', Int32),
    ('field2', String('utf-8')),
    ('field3', Array(Int16))
)
```
- **장점**: 코드 간결, 버전 변경 시 스키마만 수정

### 2. **버전별 클래스 배열**
```python
MetadataRequest = [
    MetadataRequest_v0,
    MetadataRequest_v1,
    ...
]
# 인덱스 = API 버전
```
- **장점**: 버전 선택 간단 (`MetadataRequest[version]`)

### 3. **추상 타입 인터페이스**
```python
class AbstractType:
    encode(value) -> bytes
    decode(data) -> value
```
- **장점**: 모든 타입이 동일 인터페이스, 재귀적 인코딩 가능

### 4. **Struct 상속**
```python
class MetadataRequest_v1(Struct):
    API_KEY = 3
    API_VERSION = 1
    RESPONSE_TYPE = MetadataResponse_v1
    SCHEMA = Schema(...)
```
- **장점**: 메타데이터와 스키마 분리

---

## 🔑 핵심 특징 요약

| 특징 | 설명 |
|------|------|
| **순수 Python** | librdkafka 없이 Wire Protocol 직접 구현 |
| **버전 지원** | Kafka 0.8.2 ~ 최신 (각 API 10+ 버전) |
| **타입 안전** | Schema 기반 인코딩/디코딩 |
| **Compact 지원** | Kafka 2.4+ Flexible API (varint, tagged fields) |
| **확장 가능** | 새 API 추가 시 파일만 생성하면 됨 |

---

## 📖 다음 심층 분석 파일

1. **types.py** - 타입 시스템 구현
2. **struct.py** - Request/Response 베이스
3. **api.py** - API 키 및 헤더
4. **metadata.py** - 실제 프로토콜 예시

---

## 🎓 결과적으로 이 계층은

**Kafka Wire Protocol의 순수 Python 구현체**로서:
1. ✅ **46개 API** 지원 (Produce, Fetch, Metadata, Admin 등)
2. ✅ **다중 버전 관리** (각 API당 평균 5-10개 버전)
3. ✅ **선언적 스키마** (Schema 기반 간결한 정의)
4. ✅ **타입 안전성** (AbstractType 인터페이스로 일관된 인코딩)
5. ✅ **Compact API 지원** (Kafka 2.4+ varint, tagged fields)

→ `conn.py`가 바이트를 주고받는다면, `protocol/`은 **바이트 ↔ 객체** 변환을 담당
