# protocol/struct.py & api.py - Request/Response 기반 클래스

## 📋 파일 개요
- **경로**: `aiokafka/protocol/struct.py`, `aiokafka/protocol/api.py`
- **라인 수**: 56줄 + 150줄 = 206줄
- **주요 역할**: Request/Response의 베이스 클래스 및 헤더 정의

## 🎯 핵심 목적
모든 Kafka 프로토콜 Request/Response의 **공통 베이스 클래스**를 제공하여, **Schema 기반 인코딩/디코딩**, **헤더 생성**, **API 메타데이터 관리**를 추상화

---

## 🏗️ 주요 클래스 구조

### **struct.py - Struct 베이스**

#### **Struct** 클래스
```python
class Struct:
    SCHEMA: ClassVar = Schema()  # 서브클래스에서 재정의

    def __init__(self, *args: Any, **kwargs: Any):
        # 1. 위치 인자 방식
        if len(args) == len(self.SCHEMA.fields):
            for i, name in enumerate(self.SCHEMA.names):
                self.__dict__[name] = args[i]

        # 2. 키워드 인자 방식
        elif len(args) == 0:
            for name in self.SCHEMA.names:
                self.__dict__[name] = kwargs.pop(name, None)
            if kwargs:
                raise ValueError("Unknown keywords")
        else:
            raise ValueError("Args must be empty or mirror schema")

    def encode(self) -> bytes:
        # SCHEMA를 사용해 인코딩
        return self.SCHEMA.encode([self.__dict__[name] for name in self.SCHEMA.names])

    @classmethod
    def decode(cls, data: BytesIO | bytes) -> Self:
        # SCHEMA를 사용해 디코딩
        if isinstance(data, bytes):
            data = BytesIO(data)
        return cls(*[field.decode(data) for field in cls.SCHEMA.fields])

    def get_item(self, name: str) -> Any:
        if name not in self.SCHEMA.names:
            raise KeyError(f"{name} is not in the schema")
        return self.__dict__[name]

    def __repr__(self) -> str:
        key_vals = []
        for name, field in zip(self.SCHEMA.names, self.SCHEMA.fields):
            key_vals.append(f"{name}={field.repr(self.__dict__[name])}")
        return self.__class__.__name__ + "(" + ", ".join(key_vals) + ")"

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Struct):
            return NotImplemented
        if self.SCHEMA != other.SCHEMA:
            return False
        for attr in self.SCHEMA.names:
            if self.__dict__[attr] != other.__dict__[attr]:
                return False
        return True
```

**핵심 기능**:
1. **Schema 기반 초기화**: SCHEMA 필드명에 맞춰 인스턴스 속성 생성
2. **자동 인코딩/디코딩**: SCHEMA를 위임하여 바이트 변환
3. **두 가지 생성 방식**:
   - 위치 인자: `Struct(value1, value2, ...)`
   - 키워드 인자: `Struct(field1=value1, field2=value2)`
4. **타입 안전 디코딩**: 클래스 메서드로 바이트 → 객체 생성
5. **디버깅 지원**: `__repr__`로 읽기 쉬운 표현

---

### **api.py - Request/Response 헤더**

#### **RequestHeader_v0** - 표준 요청 헤더
```python
class RequestHeader_v0(Struct):
    SCHEMA = Schema(
        ("api_key", Int16),          # API 식별자 (0=Produce, 1=Fetch, ...)
        ("api_version", Int16),      # API 버전 (0, 1, 2, ...)
        ("correlation_id", Int32),   # 요청/응답 매칭용 ID
        ("client_id", String("utf-8"))  # 클라이언트 식별자
    )

    def __init__(
        self, request: Request, correlation_id: int = 0, client_id: str = "aiokafka"
    ):
        super().__init__(
            request.API_KEY, request.API_VERSION, correlation_id, client_id
        )
```

**필드 설명**:
| 필드 | 타입 | 설명 | 예시 |
|------|------|------|------|
| `api_key` | Int16 | API 종류 | 3 (Metadata) |
| `api_version` | Int16 | 사용할 API 버전 | 1 |
| `correlation_id` | Int32 | 요청 고유 ID | 123 |
| `client_id` | String | 클라이언트 식별 | "aiokafka-1.0" |

**바이트 예시**:
```
MetadataRequest v1, correlation_id=1, client_id="test"
→ b'\x00\x03\x00\x01\x00\x00\x00\x01\x00\x04test'
   └─ api_key=3
         └─ api_version=1
               └─ correlation_id=1
                     └─ client_id="test"
```

#### **RequestHeader_v1** - Flexible API 헤더 (Kafka 2.4+)
```python
class RequestHeader_v1(Struct):
    SCHEMA = Schema(
        ("api_key", Int16),
        ("api_version", Int16),
        ("correlation_id", Int32),
        ("client_id", String("utf-8")),
        ("tags", TaggedFields),  # 추가: 확장 필드
    )

    def __init__(
        self,
        request: Request,
        correlation_id: int = 0,
        client_id: str = "aiokafka",
        tags: dict[int, bytes] | None = None,
    ):
        super().__init__(
            request.API_KEY, request.API_VERSION, correlation_id, client_id, tags or {}
        )
```

**v0 vs v1 차이**:
- **v1**: `TaggedFields` 추가 (향후 호환성)
- **사용 시점**: Request의 `FLEXIBLE_VERSION = True`일 때

#### **ResponseHeader_v0** - 표준 응답 헤더
```python
class ResponseHeader_v0(Struct):
    SCHEMA = Schema(
        ("correlation_id", Int32),  # 요청 ID 그대로 반환
    )
```

**특징**:
- **단순**: correlation_id만 포함
- **용도**: 요청과 응답 매칭

#### **ResponseHeader_v1** - Flexible API 응답 헤더
```python
class ResponseHeader_v1(Struct):
    SCHEMA = Schema(
        ("correlation_id", Int32),
        ("tags", TaggedFields),  # 확장 필드
    )
```

---

### **Request** - 추상 요청 클래스
```python
class Request(Struct, metaclass=abc.ABCMeta):
    FLEXIBLE_VERSION: ClassVar[bool] = False

    @property
    @abc.abstractmethod
    def API_KEY(self) -> int:
        """API 식별자 (0=Produce, 1=Fetch, 3=Metadata, ...)"""

    @property
    @abc.abstractmethod
    def API_VERSION(self) -> int:
        """API 버전 (0, 1, 2, ...)"""

    @property
    @abc.abstractmethod
    def RESPONSE_TYPE(self) -> type[Response]:
        """대응하는 Response 클래스"""

    @property
    @abc.abstractmethod
    def SCHEMA(self) -> Schema:
        """요청 필드 스키마"""

    def expect_response(self) -> bool:
        """응답 대기 여부 (ProduceRequest acks=0 제외하고 대부분 True)"""
        return True

    def build_request_header(
        self, correlation_id: int, client_id: str
    ) -> RequestHeader_v0 | RequestHeader_v1:
        """요청 헤더 생성"""
        if self.FLEXIBLE_VERSION:
            return RequestHeader_v1(self, correlation_id, client_id)
        return RequestHeader_v0(self, correlation_id, client_id)

    def parse_response_header(
        self, read_buffer: BytesIO | bytes
    ) -> ResponseHeader_v0 | ResponseHeader_v1:
        """응답 헤더 파싱"""
        if self.FLEXIBLE_VERSION:
            return ResponseHeader_v1.decode(read_buffer)
        return ResponseHeader_v0.decode(read_buffer)

    def to_object(self) -> dict[str, Any]:
        """디버깅용: Struct → dict 변환"""
        return _to_object(self.SCHEMA, self)
```

**핵심 메서드**:
1. **`build_request_header()`**: 헤더 생성 (conn.py에서 사용)
2. **`parse_response_header()`**: 응답 헤더 파싱 (conn.py에서 사용)
3. **`to_object()`**: 디버깅/로깅용 dict 변환

---

### **Response** - 추상 응답 클래스
```python
class Response(Struct, metaclass=abc.ABCMeta):
    @property
    @abc.abstractmethod
    def API_KEY(self) -> int:
        """API 식별자"""

    @property
    @abc.abstractmethod
    def API_VERSION(self) -> int:
        """API 버전"""

    @property
    @abc.abstractmethod
    def SCHEMA(self) -> Schema:
        """응답 필드 스키마"""

    def to_object(self) -> dict[str, Any]:
        """디버깅용: Struct → dict 변환"""
        return _to_object(self.SCHEMA, self)
```

---

## 🔄 사용 흐름

### 1. **프로토콜 정의 (metadata.py 예시)**
```python
from aiokafka.protocol.api import Request, Response
from aiokafka.protocol.types import Schema, Array, String

# Request 정의
class MetadataRequest_v1(Request):
    API_KEY = 3
    API_VERSION = 1
    RESPONSE_TYPE = MetadataResponse_v1
    SCHEMA = Schema(
        ('topics', Array(String('utf-8')))
    )

# Response 정의
class MetadataResponse_v1(Response):
    API_KEY = 3
    API_VERSION = 1
    SCHEMA = Schema(
        ('brokers', Array(...)),
        ('topics', Array(...))
    )

# 버전별 리스트
MetadataRequest = [MetadataRequest_v0, MetadataRequest_v1, ...]
```

### 2. **요청 생성 및 전송 (client.py)**
```python
# 1. 요청 객체 생성
request = MetadataRequest[1](topics=['test', 'production'])

# 2. 헤더 생성
header = request.build_request_header(
    correlation_id=123,
    client_id="aiokafka"
)

# 3. 인코딩
message = header.encode() + request.encode()
# → b'\x00\x03\x00\x01\x00\x00\x00\x7b\x00\x08aiokafka...'
#     └─ header                           └─ request body

# 4. 전송 (conn.py)
size = struct.pack(">i", len(message))
writer.write(size + message)
```

### 3. **응답 수신 및 파싱 (conn.py)**
```python
# 1. 응답 수신
resp_bytes = await reader.readexactly(size)

# 2. 헤더 파싱
resp_buffer = BytesIO(resp_bytes)
response_header = request.parse_response_header(resp_buffer)

# 3. correlation_id 검증
if response_header.correlation_id != correlation_id:
    raise CorrelationIdError(...)

# 4. 응답 본문 디코딩
response = request.RESPONSE_TYPE.decode(resp_buffer)
# → MetadataResponse_v1(brokers=[...], topics=[...])

# 5. 사용
for broker in response.brokers:
    print(broker.nodeId, broker.host, broker.port)
```

---

## 📦 to_object() 헬퍼 함수

```python
def _to_object(schema: Schema, data: Struct | dict[int, Any]) -> dict[str, Any]:
    """Struct를 JSON 직렬화 가능한 dict로 변환"""
    obj = {}
    for idx, (name, _type) in enumerate(zip(schema.names, schema.fields)):
        if isinstance(data, Struct):
            val = data.get_item(name)
        else:
            val = data[idx]

        # 재귀적 변환
        if isinstance(_type, Schema):
            obj[name] = _to_object(_type, val)
        elif isinstance(_type, Array):
            if isinstance(_type.array_of, Schema):
                obj[name] = [_to_object(_type.array_of, x) for x in val]
            else:
                obj[name] = val
        else:
            obj[name] = val

    return obj
```

**사용 예시**:
```python
request = MetadataRequest[1](topics=['test'])
print(request.to_object())
# → {'topics': ['test']}

response = MetadataResponse_v1(brokers=[...], topics=[...])
print(response.to_object())
# → {
#     'brokers': [
#       {'nodeId': 0, 'host': 'broker1', 'port': 9092},
#       ...
#     ],
#     'topics': [...]
#   }
```

**용도**:
- 로깅/디버깅
- JSON 직렬화 (테스트, 모니터링)

---

## 🎨 설계 패턴 분석

### 1. **Template Method 패턴**
```python
class Struct:
    SCHEMA = Schema()  # 서브클래스에서 재정의

    def encode(self):
        # 템플릿 메서드: SCHEMA 사용
        return self.SCHEMA.encode(...)

    @classmethod
    def decode(cls, data):
        # 템플릿 메서드: SCHEMA 사용
        return cls(*[field.decode(data) for field in cls.SCHEMA.fields])
```

**장점**:
- 서브클래스는 SCHEMA만 정의하면 인코딩/디코딩 자동

### 2. **Factory Method 패턴**
```python
class Request:
    @abstractmethod
    def RESPONSE_TYPE(self) -> type[Response]:
        """대응하는 Response 클래스"""

# 사용
request = MetadataRequest[1](...)
response = request.RESPONSE_TYPE.decode(data)  # 자동으로 올바른 Response 클래스 사용
```

### 3. **Strategy 패턴 (Flexible vs Standard)**
```python
def build_request_header(self, ...):
    if self.FLEXIBLE_VERSION:
        return RequestHeader_v1(...)  # Flexible API 전략
    return RequestHeader_v0(...)       # Standard 전략
```

### 4. **Adapter 패턴 (Struct ↔ Schema)**
```python
# Struct는 Schema를 감싸서 객체지향 인터페이스 제공
class Struct:
    def __init__(self, *args):
        # args를 SCHEMA.names에 매핑
        for i, name in enumerate(self.SCHEMA.names):
            self.__dict__[name] = args[i]

    def encode(self):
        # 객체 → SCHEMA로 변환 → 인코딩
        return self.SCHEMA.encode([self.__dict__[name] for name in self.SCHEMA.names])
```

---

## 🔗 다른 모듈과의 관계

### 계층 구조
```
abstract.py (AbstractType)
    ↓
types.py (Int16, String, Schema, ...)
    ↓
struct.py (Struct)
    ↓
api.py (Request, Response, Headers)
    ↓
metadata.py, produce.py, fetch.py 등 (구체적인 프로토콜)
```

### 사용 예시
```python
# metadata.py
class MetadataRequest_v1(Request):  # api.py의 Request 상속
    SCHEMA = Schema(...)             # types.py의 Schema 사용

# conn.py
header = request.build_request_header(...)  # api.py 메서드
message = header.encode() + request.encode()  # struct.py 메서드
```

---

## 📊 실제 바이트 예시

### MetadataRequest v1 인코딩
```python
request = MetadataRequest[1](topics=['test'])
header = request.build_request_header(correlation_id=1, client_id="aiokafka")

# 헤더 인코딩
header_bytes = header.encode()
# b'\x00\x03'           # api_key=3 (Metadata)
# b'\x00\x01'           # api_version=1
# b'\x00\x00\x00\x01'   # correlation_id=1
# b'\x00\x08aiokafka'   # client_id="aiokafka"

# 요청 본문 인코딩
body_bytes = request.encode()
# b'\x00\x00\x00\x01'   # topics.length=1
# b'\x00\x04test'       # topics[0]="test"

# 전체 메시지
size = struct.pack(">i", len(header_bytes + body_bytes))
message = size + header_bytes + body_bytes
# b'\x00\x00\x00\x1f'   # size=31
# + header_bytes
# + body_bytes
```

### MetadataResponse v1 디코딩
```python
# 응답 수신
resp_bytes = b'\x00\x00\x00\x01...'  # correlation_id=1 + ...

# 헤더 파싱
resp_buffer = BytesIO(resp_bytes)
response_header = request.parse_response_header(resp_buffer)
# → ResponseHeader_v0(correlation_id=1)

# 본문 디코딩
response = MetadataResponse_v1.decode(resp_buffer)
# → MetadataResponse_v1(
#     brokers=[BrokerMetadata(0, 'broker1', 9092), ...],
#     topics=[TopicMetadata('test', [...]), ...]
#   )
```

---

## ⚙️ 핵심 특징 요약

| 특징 | 설명 |
|------|------|
| **Schema 기반** | SCHEMA만 정의하면 인코딩/디코딩 자동 |
| **타입 안전** | 추상 클래스로 필수 속성 강제 |
| **Flexible API** | FLEXIBLE_VERSION으로 v0/v1 헤더 자동 선택 |
| **디버깅 지원** | `__repr__`, `to_object()` 제공 |
| **헤더 자동화** | `build_request_header()` 메서드로 간편한 생성 |
| **응답 타입 연결** | `RESPONSE_TYPE`으로 Request ↔ Response 매핑 |

---

## 🎓 결과적으로 이 파일들은

**모든 Kafka 프로토콜의 공통 베이스**로서:
1. ✅ **Struct**: Schema 기반 자동 인코딩/디코딩
2. ✅ **Request/Response**: API 메타데이터 및 헤더 관리
3. ✅ **Headers**: 요청/응답 헤더 자동 생성 및 파싱
4. ✅ **Flexible API**: Kafka 2.4+ TaggedFields 지원
5. ✅ **디버깅**: `__repr__`, `to_object()` 제공
6. ✅ **타입 안전**: 추상 클래스로 필수 속성 강제

→ 모든 프로토콜(`metadata.py`, `produce.py` 등)이 상속하는 **공통 인프라**
