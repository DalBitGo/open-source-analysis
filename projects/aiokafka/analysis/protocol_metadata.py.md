# protocol/metadata.py - Metadata API 프로토콜

## 📋 파일 개요
- **경로**: `aiokafka/protocol/metadata.py`
- **라인 수**: 272줄
- **주요 역할**: Kafka Metadata API (API_KEY=3) 요청/응답 정의

## 🎯 핵심 목적
클러스터 메타데이터(브로커, 토픽, 파티션)를 조회하는 **Metadata API**의 **요청/응답 프로토콜** 정의 (v0 ~ v5, 6개 버전)

---

## 🏗️ 프로토콜 구조

### **MetadataRequest** - 메타데이터 요청

#### **v0** - 기본 버전
```python
class MetadataRequest_v0(Request):
    API_KEY = 3
    API_VERSION = 0
    RESPONSE_TYPE = MetadataResponse_v0
    SCHEMA = Schema(
        ("topics", Array(String("utf-8")))  # 조회할 토픽 목록
    )
```

**필드**:
- `topics`: 조회할 토픽 목록
  - `[]` (빈 배열): **모든 토픽** 반환

**사용 예시**:
```python
# 특정 토픽 조회
request = MetadataRequest[0](topics=['test', 'production'])

# 모든 토픽 조회
request = MetadataRequest[0](topics=[])
```

#### **v1** - Null Array 지원
```python
class MetadataRequest_v1(Request):
    # topics:
    #   -1 (Null Array): 모든 토픽 반환
    #   [] (빈 배열): 토픽 없음

    API_KEY = 3
    API_VERSION = 1
    RESPONSE_TYPE = MetadataResponse_v1
    SCHEMA = MetadataRequest_v0.SCHEMA  # 동일
```

**v0 vs v1 차이**:
| 버전 | `topics=[]` | `topics=None` |
|------|-------------|---------------|
| v0 | 모든 토픽 | 불가능 |
| v1 | 토픽 없음 | 모든 토픽 (Null Array) |

#### **v4** - Auto Topic Creation 옵션
```python
class MetadataRequest_v4(Request):
    API_KEY = 3
    API_VERSION = 4
    RESPONSE_TYPE = MetadataResponse_v4
    SCHEMA = Schema(
        ("topics", Array(String("utf-8"))),
        ("allow_auto_topic_creation", Boolean)  # 추가
    )
```

**신규 필드**:
- `allow_auto_topic_creation`: 토픽 없으면 자동 생성 여부

---

### **MetadataResponse** - 메타데이터 응답

#### **v0** - 기본 버전
```python
class MetadataResponse_v0(Response):
    API_KEY = 3
    API_VERSION = 0
    SCHEMA = Schema(
        # 브로커 목록
        ("brokers", Array(
            ("node_id", Int32),       # 브로커 ID
            ("host", String("utf-8")), # 호스트명
            ("port", Int32)            # 포트
        )),

        # 토픽 목록
        ("topics", Array(
            ("error_code", Int16),      # 토픽 레벨 에러
            ("topic", String("utf-8")), # 토픽명
            # 파티션 목록
            ("partitions", Array(
                ("error_code", Int16),  # 파티션 레벨 에러
                ("partition", Int32),   # 파티션 ID
                ("leader", Int32),      # 리더 브로커 ID
                ("replicas", Array(Int32)),  # 복제본 브로커 ID 목록
                ("isr", Array(Int32))   # In-Sync Replicas
            ))
        ))
    )
```

**응답 구조**:
```
MetadataResponse
├── brokers[]
│   ├── node_id
│   ├── host
│   └── port
└── topics[]
    ├── error_code
    ├── topic
    └── partitions[]
        ├── error_code
        ├── partition
        ├── leader
        ├── replicas[]
        └── isr[]
```

**사용 예시**:
```python
response = MetadataResponse_v0.decode(data)

# 브로커 정보
for broker in response.brokers:
    print(f"Broker {broker[0]}: {broker[1]}:{broker[2]}")
    # Broker 0: broker1:9092

# 토픽 정보
for topic in response.topics:
    error_code, topic_name, partitions = topic
    for partition in partitions:
        _, partition_id, leader, replicas, isr = partition
        print(f"{topic_name}-{partition_id}: leader={leader}, replicas={replicas}")
```

#### **v1** - Rack 및 Controller 추가
```python
class MetadataResponse_v1(Response):
    SCHEMA = Schema(
        ("brokers", Array(
            ("node_id", Int32),
            ("host", String("utf-8")),
            ("port", Int32),
            ("rack", String("utf-8"))  # 추가: Rack Awareness
        )),
        ("controller_id", Int32),      # 추가: 컨트롤러 브로커 ID
        ("topics", Array(
            ("error_code", Int16),
            ("topic", String("utf-8")),
            ("is_internal", Boolean),  # 추가: 내부 토픽 여부
            ("partitions", Array(...))
        ))
    )
```

**신규 필드**:
- `rack`: 브로커 Rack ID (Rack Awareness 지원)
- `controller_id`: 클러스터 컨트롤러 브로커 ID
- `is_internal`: 내부 토픽 여부 (`__consumer_offsets` 등)

#### **v2** - Cluster ID 추가
```python
class MetadataResponse_v2(Response):
    SCHEMA = Schema(
        ("brokers", Array(...)),
        ("cluster_id", String("utf-8")),  # 추가
        ("controller_id", Int32),
        ("topics", Array(...))
    )
```

**신규 필드**:
- `cluster_id`: Kafka 클러스터 고유 ID

#### **v3** - Throttle Time 추가
```python
class MetadataResponse_v3(Response):
    SCHEMA = Schema(
        ("throttle_time_ms", Int32),  # 추가: 쿼터 제한 시간
        ("brokers", Array(...)),
        ("cluster_id", String("utf-8")),
        ("controller_id", Int32),
        ("topics", Array(...))
    )
```

**신규 필드**:
- `throttle_time_ms`: 쿼터 제한으로 인한 지연 시간 (ms)

#### **v5** - Offline Replicas 추가
```python
class MetadataResponse_v5(Response):
    SCHEMA = Schema(
        ("throttle_time_ms", Int32),
        ("brokers", Array(...)),
        ("cluster_id", String("utf-8")),
        ("controller_id", Int32),
        ("topics", Array(
            ("error_code", Int16),
            ("topic", String("utf-8")),
            ("is_internal", Boolean),
            ("partitions", Array(
                ("error_code", Int16),
                ("partition", Int32),
                ("leader", Int32),
                ("replicas", Array(Int32)),
                ("isr", Array(Int32)),
                ("offline_replicas", Array(Int32))  # 추가
            ))
        ))
    )
```

**신규 필드**:
- `offline_replicas`: 오프라인 상태인 복제본 브로커 ID 목록

---

## 📊 버전별 진화 요약

| 버전 | 주요 변경 사항 |
|------|---------------|
| v0 | 기본 메타데이터 (brokers, topics, partitions) |
| v1 | Rack ID, Controller ID, is_internal 추가 |
| v2 | Cluster ID 추가 |
| v3 | Throttle Time 추가 |
| v4 | (Request) allow_auto_topic_creation 추가 |
| v5 | Offline Replicas 추가 |

---

## 🔄 사용 흐름

### 1. **client.py에서 메타데이터 조회**
```python
# client.py - bootstrap()
if self._api_version == "auto" or self._api_version < (0, 10):
    metadata_request = MetadataRequest[0]([])  # v0, 모든 토픽
else:
    metadata_request = MetadataRequest[1]([])  # v1, 모든 토픽

# 요청 전송
metadata = await bootstrap_conn.send(metadata_request)

# 응답 처리
self.cluster.update_metadata(metadata)
```

### 2. **cluster.py에서 메타데이터 파싱**
```python
# cluster.py - update_metadata()
def update_metadata(self, metadata):
    # 브로커 파싱
    for broker in metadata.brokers:
        if metadata.API_VERSION == 0:
            node_id, host, port = broker
            rack = None
        else:
            node_id, host, port, rack = broker
        self._brokers[node_id] = BrokerMetadata(node_id, host, port, rack)

    # 토픽 파싱
    for topic_data in metadata.topics:
        if metadata.API_VERSION == 0:
            error_code, topic, partitions = topic_data
            is_internal = False
        else:
            error_code, topic, is_internal, partitions = topic_data

        for p_error, partition, leader, replicas, isr, *_ in partitions:
            self._partitions[topic][partition] = PartitionMetadata(
                topic, partition, leader, replicas, isr, p_error
            )
```

---

## 📦 버전 리스트 export

```python
# 파일 끝부분
MetadataRequest = [
    MetadataRequest_v0,
    MetadataRequest_v1,
    MetadataRequest_v2,
    MetadataRequest_v3,
    MetadataRequest_v4,
    MetadataRequest_v5,
]

MetadataResponse = [
    MetadataResponse_v0,
    MetadataResponse_v1,
    MetadataResponse_v2,
    MetadataResponse_v3,
    MetadataResponse_v4,
    MetadataResponse_v5,
]
```

**사용**:
```python
from aiokafka.protocol.metadata import MetadataRequest, MetadataResponse

# 버전 선택
request = MetadataRequest[1](topics=None)  # v1, 모든 토픽
# request = MetadataRequest[4](topics=['test'], allow_auto_topic_creation=True)
```

---

## 🎨 Schema 정의 패턴

### 중첩 Array
```python
# 3-level 구조
Schema(
    ("topics", Array(              # Level 1: 토픽 배열
        ("topic", String("utf-8")),
        ("partitions", Array(      # Level 2: 파티션 배열
            ("partition", Int32),
            ("replicas", Array(Int32))  # Level 3: replica ID 배열
        ))
    ))
)
```

### Schema 재사용
```python
class MetadataRequest_v1(Request):
    SCHEMA = MetadataRequest_v0.SCHEMA  # 동일 스키마 재사용

class MetadataResponse_v4(Response):
    SCHEMA = MetadataResponse_v3.SCHEMA  # 동일 스키마 재사용
```

### 점진적 확장
```python
# v2: v1 + cluster_id
Schema(
    ("brokers", ...),
    ("cluster_id", String("utf-8")),  # 추가
    ("controller_id", Int32),
    ("topics", ...)
)

# v3: v2 + throttle_time_ms
Schema(
    ("throttle_time_ms", Int32),  # 추가
    ("brokers", ...),
    ("cluster_id", String("utf-8")),
    ...
)
```

---

## 🔗 다른 모듈과의 관계

### 사용처
```
metadata.py
    ↓ import
client.py (bootstrap, _metadata_update)
    ↓
cluster.py (update_metadata)
    ↓
producer.py, consumer.py (메타데이터 참조)
```

### 의존성
```
metadata.py
    ↓ import
api.py (Request, Response)
    ↓ import
types.py (Schema, Array, Int32, String, ...)
```

---

## 🎓 결과적으로 이 파일은

**Kafka Metadata API의 프로토콜 정의**로서:
1. ✅ **6개 버전** 지원 (v0 ~ v5)
2. ✅ **점진적 진화** (Rack, Controller, Cluster ID, Throttle 추가)
3. ✅ **Schema 기반** 간결한 정의
4. ✅ **중첩 구조** (brokers → topics → partitions → replicas)
5. ✅ **Auto Topic Creation** 옵션 (v4+)
6. ✅ **Offline Replicas** 추적 (v5+)

→ `client.py`가 클러스터 정보를 얻기 위해 사용하는 **핵심 프로토콜**
