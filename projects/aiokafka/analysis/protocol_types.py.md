# protocol/types.py - Kafka Wire Protocol 타입 시스템

## 📋 파일 개요
- **경로**: `aiokafka/protocol/types.py`
- **라인 수**: 421줄
- **주요 역할**: Kafka Wire Protocol의 기본 데이터 타입 구현 (인코딩/디코딩)

## 🎯 핵심 목적
**Kafka Wire Protocol**의 **모든 기본 타입**을 Python으로 구현하여, Python 객체 ↔ 바이트 스트림 변환을 담당하는 **타입 시스템의 핵심**

---

## 🏗️ 타입 계층 구조

```
AbstractType (abstract.py)
    ↓
┌────────────────────────────────────────┐
│  기본 정수 타입 (Fixed-length)          │
│  - Int8, Int16, Int32, UInt32, Int64   │
│  - Float64, Boolean                    │
└────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────┐
│  가변 길이 타입 (Variable-length)       │
│  - String (2-byte length prefix)       │
│  - Bytes (4-byte length prefix)        │
└────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────┐
│  복합 타입 (Composite)                  │
│  - Schema (Named fields)               │
│  - Array (Repeated elements)           │
└────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────┐
│  VarInt 타입 (Compact Encoding)        │
│  - UnsignedVarInt32, VarInt32          │
│  - VarInt64                            │
└────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────┐
│  Compact 타입 (Kafka 2.4+)             │
│  - CompactString, CompactBytes         │
│  - CompactArray                        │
│  - TaggedFields                        │
└────────────────────────────────────────┘
```

---

## 📦 타입별 상세 분석

### 🔹 기본 정수 타입 (Fixed-length Integers)

#### **Int8** - 8-bit signed integer
```python
class Int8(AbstractType[int]):
    _pack = struct.Struct(">b").pack      # big-endian signed char
    _unpack = struct.Struct(">b").unpack

    @classmethod
    def encode(cls, value: int) -> bytes:
        return _pack(cls._pack, value)    # int → bytes

    @classmethod
    def decode(cls, data: BytesIO) -> int:
        return _unpack(cls._unpack, data.read(1))  # bytes → int
```
- **형식**: `>b` (big-endian, signed byte)
- **범위**: -128 ~ 127
- **크기**: 1 byte
- **예시**: `5` → `b'\x05'`, `-1` → `b'\xff'`

#### **Int16** - 16-bit signed integer
```python
class Int16(AbstractType[int]):
    _pack = struct.Struct(">h").pack      # big-endian short
    _unpack = struct.Struct(">h").unpack

    @classmethod
    def encode(cls, value: int) -> bytes:
        return _pack(cls._pack, value)

    @classmethod
    def decode(cls, data: BytesIO) -> int:
        return _unpack(cls._unpack, data.read(2))
```
- **형식**: `>h` (big-endian, short)
- **범위**: -32768 ~ 32767
- **크기**: 2 bytes
- **예시**: `300` → `b'\x01\x2c'`

#### **Int32** - 32-bit signed integer
```python
class Int32(AbstractType[int]):
    _pack = struct.Struct(">i").pack      # big-endian int
    _unpack = struct.Struct(">i").unpack
    # ...
```
- **형식**: `>i` (big-endian, int)
- **크기**: 4 bytes
- **사용처**: 파티션 ID, 오프셋, 길이 필드 등

#### **UInt32** - 32-bit unsigned integer
```python
class UInt32(AbstractType[int]):
    _pack = struct.Struct(">I").pack      # big-endian unsigned int
    _unpack = struct.Struct(">I").unpack
```
- **형식**: `>I` (unsigned)
- **범위**: 0 ~ 4294967295
- **사용처**: CRC32 체크섬

#### **Int64** - 64-bit signed integer
```python
class Int64(AbstractType[int]):
    _pack = struct.Struct(">q").pack      # big-endian long long
    _unpack = struct.Struct(">q").unpack
```
- **크기**: 8 bytes
- **사용처**: 오프셋, 타임스탬프

#### **Float64** - 64-bit float
```python
class Float64(AbstractType[float]):
    _pack = struct.Struct(">d").pack
    _unpack = struct.Struct(">d").unpack
```
- **형식**: `>d` (IEEE 754 double)
- **크기**: 8 bytes

#### **Boolean**
```python
class Boolean(AbstractType[bool]):
    _pack = struct.Struct(">?").pack
    _unpack = struct.Struct(">?").unpack
```
- **크기**: 1 byte
- **값**: `True` → `b'\x01'`, `False` → `b'\x00'`

---

### 🔹 가변 길이 타입 (Variable-length Types)

#### **String** - Nullable UTF-8 문자열
```python
class String:
    def __init__(self, encoding: str = "utf-8"):
        self.encoding = encoding

    def encode(self, value: str | None) -> bytes:
        if value is None:
            return Int16.encode(-1)  # null = -1
        encoded_value = str(value).encode(self.encoding)
        return Int16.encode(len(encoded_value)) + encoded_value

    def decode(self, data: BytesIO) -> str | None:
        length = Int16.decode(data)
        if length < 0:
            return None  # -1 → null
        value = data.read(length)
        if len(value) != length:
            raise ValueError("Buffer underrun decoding string")
        return value.decode(self.encoding)
```

**인코딩 형식**:
```
[2-byte length][UTF-8 bytes]

예시:
"test" → b'\x00\x04test'
null   → b'\xff\xff'
""     → b'\x00\x00'
```

**특징**:
- **null 가능**: length = -1
- **최대 길이**: 32767 bytes (Int16 max)
- **인코딩**: UTF-8 기본 (변경 가능)

#### **Bytes** - Nullable 바이트 배열
```python
class Bytes(AbstractType[bytes | None]):
    @classmethod
    def encode(cls, value: bytes | None) -> bytes:
        if value is None:
            return Int32.encode(-1)
        else:
            return Int32.encode(len(value)) + value

    @classmethod
    def decode(cls, data: BytesIO) -> bytes | None:
        length = Int32.decode(data)
        if length < 0:
            return None
        value = data.read(length)
        if len(value) != length:
            raise ValueError("Buffer underrun decoding Bytes")
        return value

    @classmethod
    def repr(cls, value: bytes | None) -> str:
        # 100바이트 이상이면 축약
        return repr(
            value[:100] + b"..." if value and len(value) > 100 else value
        )
```

**인코딩 형식**:
```
[4-byte length][raw bytes]

예시:
b'\x01\x02\x03' → b'\x00\x00\x00\x03\x01\x02\x03'
null            → b'\xff\xff\xff\xff'
```

**String vs Bytes**:
| 속성 | String | Bytes |
|------|--------|-------|
| 길이 prefix | Int16 (2 bytes) | Int32 (4 bytes) |
| 최대 크기 | 32KB | 2GB |
| 인코딩 | UTF-8 | raw bytes |
| 용도 | 토픽명, group_id | 메시지 payload, RecordBatch |

---

### 🔹 복합 타입 (Composite Types)

#### **Schema** - 구조화된 필드
```python
class Schema:
    names: tuple[str, ...]
    fields: tuple[ValueT, ...]

    def __init__(self, *fields: tuple[str, ValueT]):
        if fields:
            self.names, self.fields = zip(*fields, strict=False)
        else:
            self.names, self.fields = (), ()

    def encode(self, item: Sequence[Any]) -> bytes:
        if len(item) != len(self.fields):
            raise ValueError("Item field count does not match Schema")
        return b"".join(field.encode(item[i]) for i, field in enumerate(self.fields))

    def decode(self, data: BytesIO) -> tuple[Any, ...]:
        return tuple(field.decode(data) for field in self.fields)

    def repr(self, value: Any) -> str:
        key_vals = []
        for i in range(len(self)):
            try:
                field_val = getattr(value, self.names[i])  # 객체 속성
            except AttributeError:
                field_val = value[i]  # 튜플 인덱스
            key_vals.append(f"{self.names[i]}={self.fields[i].repr(field_val)}")
        return "(" + ", ".join(key_vals) + ")"
```

**사용 예시**:
```python
# 정의
header_schema = Schema(
    ('api_key', Int16),
    ('api_version', Int16),
    ('correlation_id', Int32),
    ('client_id', String('utf-8'))
)

# 인코딩
data = (3, 1, 123, "aiokafka")
encoded = header_schema.encode(data)
# → b'\x00\x03\x00\x01\x00\x00\x00\x7b\x00\x08aiokafka'

# 디코딩
decoded = header_schema.decode(BytesIO(encoded))
# → (3, 1, 123, "aiokafka")
```

**특징**:
- **Named fields**: 필드명 추적
- **순차 인코딩**: 필드 순서대로 직렬화 (구분자 없음)
- **repr**: 디버깅용 읽기 쉬운 표현

#### **Array** - 반복 요소 배열
```python
class Array:
    array_of: ValueT

    def __init__(
        self,
        array_of_0: ValueT | tuple[str, ValueT],
        *array_of: tuple[str, ValueT],
    ):
        if array_of:
            # Schema 형식 (여러 필드)
            self.array_of = Schema(array_of_0, *array_of)
        else:
            # 단일 타입
            self.array_of = array_of_0

    def encode(self, items: Sequence[Any] | None) -> bytes:
        if items is None:
            return Int32.encode(-1)  # null
        encoded_items = (self.array_of.encode(item) for item in items)
        return b"".join(
            (Int32.encode(len(items)), *encoded_items),
        )

    def decode(self, data: BytesIO) -> list[Any] | None:
        length = Int32.decode(data)
        if length == -1:
            return None
        return [self.array_of.decode(data) for _ in range(length)]

    def repr(self, list_of_items: Sequence[Any] | None) -> str:
        if list_of_items is None:
            return "NULL"
        return "[" + ", ".join(self.array_of.repr(item) for item in list_of_items) + "]"
```

**인코딩 형식**:
```
[4-byte length][element 0][element 1]...[element N-1]

예시:
Array(Int16)([1, 2, 3])
→ b'\x00\x00\x00\x03\x00\x01\x00\x02\x00\x03'
   └─ length=3    └─ 1  └─ 2  └─ 3
```

**사용 패턴**:

**패턴 1: 단일 타입 배열**
```python
Array(String('utf-8'))  # Array of strings
Array(Int32)            # Array of integers
```

**패턴 2: Schema 배열 (인라인 정의)**
```python
Array(
    ('partition', Int32),
    ('offset', Int64),
    ('metadata', String('utf-8'))
)
# → Array of {partition, offset, metadata} 구조체
```

---

### 🔹 VarInt 타입 (Compact Encoding)

#### **UnsignedVarInt32** - 부호 없는 가변 길이 정수
```python
class UnsignedVarInt32(AbstractType[int]):
    @classmethod
    def encode(cls, value: int) -> bytes:
        value &= 0xFFFFFFFF
        ret = b""
        while (value & 0xFFFFFF80) != 0:
            b = (value & 0x7F) | 0x80  # MSB = 1 (계속)
            ret += struct.pack("B", b)
            value >>= 7
        ret += struct.pack("B", value)  # MSB = 0 (종료)
        return ret

    @classmethod
    def decode(cls, data: BytesIO) -> int:
        value, i = 0, 0
        while True:
            (b,) = struct.unpack("B", data.read(1))
            if not (b & 0x80):  # MSB = 0 → 마지막 바이트
                break
            value |= (b & 0x7F) << i
            i += 7
            if i > 28:
                raise ValueError(f"Invalid value {value}")
        value |= b << i
        return value
```

**VarInt 인코딩 방식**:
```
- 7비트씩 사용, MSB는 continuation bit
- MSB=1: 다음 바이트 있음
- MSB=0: 마지막 바이트

예시:
0     → b'\x00'
127   → b'\x7f'
128   → b'\x80\x01'  (0b10000000 0b00000001)
16383 → b'\xff\x7f'
16384 → b'\x80\x80\x01'
```

**장점**:
- 작은 숫자는 1-2 바이트 (Int32는 항상 4 바이트)
- Kafka 2.4+ Flexible API에서 사용

#### **VarInt32** - 부호 있는 가변 길이 정수
```python
class VarInt32(AbstractType[int]):
    @classmethod
    def encode(cls, value: int) -> bytes:
        value &= 0xFFFFFFFF
        # ZigZag 인코딩: 부호를 LSB로 이동
        return UnsignedVarInt32.encode((value << 1) ^ (value >> 31))

    @classmethod
    def decode(cls, data: BytesIO) -> int:
        value = UnsignedVarInt32.decode(data)
        # ZigZag 디코딩
        return (value >> 1) ^ -(value & 1)
```

**ZigZag 인코딩**:
```
부호 있는 정수를 효율적으로 인코딩
0  → 0
-1 → 1
1  → 2
-2 → 3
2  → 4

변환 공식:
  (n << 1) ^ (n >> 31)  # 인코딩
  (n >> 1) ^ -(n & 1)   # 디코딩
```

#### **VarInt64** - 64-bit 가변 길이 정수
```python
class VarInt64(AbstractType[int]):
    # UnsignedVarInt32와 유사하지만 64비트
    # ZigZag 인코딩 사용
```

---

### 🔹 Compact 타입 (Kafka 2.4+ Flexible API)

#### **CompactString**
```python
class CompactString(String):
    def encode(self, value: str | None) -> bytes:
        if value is None:
            return UnsignedVarInt32.encode(0)  # null = 0 (기존은 -1)
        encoded_value = str(value).encode(self.encoding)
        return UnsignedVarInt32.encode(len(encoded_value) + 1) + encoded_value

    def decode(self, data: BytesIO) -> str | None:
        length = UnsignedVarInt32.decode(data) - 1
        if length < 0:
            return None
        value = data.read(length)
        if len(value) != length:
            raise ValueError("Buffer underrun decoding string")
        return value.decode(self.encoding)
```

**기존 String과의 차이**:
| 속성 | String | CompactString |
|------|--------|---------------|
| 길이 인코딩 | Int16 (2 bytes 고정) | UnsignedVarInt32 (1-5 bytes) |
| null 표현 | -1 | 0 |
| 길이 오프셋 | length | length + 1 |

**예시**:
```python
# String
"test" → b'\x00\x04test'  (6 bytes)

# CompactString
"test" → b'\x05test'  (5 bytes, 0x05 = length+1)
```

#### **CompactBytes**
```python
class CompactBytes(AbstractType[bytes | None]):
    @classmethod
    def encode(cls, value: bytes | None) -> bytes:
        if value is None:
            return UnsignedVarInt32.encode(0)
        else:
            return UnsignedVarInt32.encode(len(value) + 1) + value

    @classmethod
    def decode(cls, data: BytesIO) -> bytes | None:
        length = UnsignedVarInt32.decode(data) - 1
        if length < 0:
            return None
        value = data.read(length)
        if len(value) != length:
            raise ValueError("Buffer underrun decoding Bytes")
        return value
```

#### **CompactArray**
```python
class CompactArray(Array):
    def encode(self, items: Sequence[Any] | None) -> bytes:
        if items is None:
            return UnsignedVarInt32.encode(0)
        encoded_items = (self.array_of.encode(item) for item in items)
        return b"".join(
            (UnsignedVarInt32.encode(len(items) + 1), *encoded_items),
        )

    def decode(self, data: BytesIO) -> list[Any] | None:
        length = UnsignedVarInt32.decode(data) - 1
        if length == -1:
            return None
        return [self.array_of.decode(data) for _ in range(length)]
```

**특징**:
- 길이를 VarInt로 인코딩 → 작은 배열 시 공간 절약
- `length + 1` 인코딩 (null = 0, 빈 배열 = 1)

#### **TaggedFields** - Flexible API 태그 필드
```python
class TaggedFields(AbstractType[dict[int, bytes]]):
    @classmethod
    def encode(cls, value: dict[int, bytes]) -> bytes:
        ret = UnsignedVarInt32.encode(len(value))
        for k, v in value.items():
            assert isinstance(v, bytes)
            assert isinstance(k, int) and k > 0
            ret += UnsignedVarInt32.encode(k)  # tag
            ret += UnsignedVarInt32.encode(len(v))  # size
            ret += v
        return ret

    @classmethod
    def decode(cls, data: BytesIO) -> dict[int, bytes]:
        num_fields = UnsignedVarInt32.decode(data)
        ret = {}
        if not num_fields:
            return ret
        prev_tag = -1
        for _ in range(num_fields):
            tag = UnsignedVarInt32.decode(data)
            if tag <= prev_tag:
                raise ValueError(f"Invalid or out-of-order tag {tag}")
            prev_tag = tag
            size = UnsignedVarInt32.decode(data)
            val = data.read(size)
            ret[tag] = val
        return ret
```

**인코딩 형식**:
```
[num_fields][tag_1][size_1][data_1][tag_2][size_2][data_2]...

예시:
{3: b'foo', 5: b'bar'}
→ b'\x02\x03\x03foo\x05\x03bar'
   └─ 2개 └─ tag=3, size=3 └─ tag=5, size=3
```

**특징**:
- **태그 순서 보장**: 태그는 오름차순
- **향후 호환성**: 새 필드 추가 시 기존 클라이언트에 영향 없음
- **Flexible API**: Kafka 2.4+ MetadataRequest v9+ 등

---

## 🔄 인코딩/디코딩 흐름

### 인코딩 예시: MetadataRequest
```python
# Schema 정의 (metadata.py)
MetadataRequest_v1 = Schema(
    ('topics', Array(String('utf-8')))
)

# 인코딩
topics = ["test", "production"]
encoded = MetadataRequest_v1.encode([topics])

# 단계별 변환:
# 1. Array.encode(["test", "production"])
#    → Int32.encode(2) + String.encode("test") + String.encode("production")
#
# 2. Int32.encode(2)
#    → b'\x00\x00\x00\x02'
#
# 3. String.encode("test")
#    → Int16.encode(4) + b'test'
#    → b'\x00\x04test'
#
# 4. String.encode("production")
#    → b'\x00\x0aproduction'
#
# 최종 결과:
# b'\x00\x00\x00\x02\x00\x04test\x00\x0aproduction'
```

### 디코딩 흐름
```python
data = BytesIO(b'\x00\x00\x00\x02\x00\x04test\x00\x0aproduction')

# 1. Array.decode()
length = Int32.decode(data)  # 2
items = []
for _ in range(2):
    items.append(String.decode(data))

# 2. 첫 번째 String.decode()
length = Int16.decode(data)  # 4
value = data.read(4)  # b'test'
items.append(value.decode('utf-8'))  # "test"

# 3. 두 번째 String.decode()
# ... "production"

# 최종: ["test", "production"]
```

---

## ⚙️ 핵심 설계 패턴

### 1. **AbstractType 인터페이스**
```python
class AbstractType:
    encode(value) -> bytes
    decode(data: BytesIO) -> value
    repr(value) -> str
```
- **일관성**: 모든 타입이 동일한 인터페이스
- **재귀 인코딩**: Schema/Array가 재귀적으로 하위 타입 호출

### 2. **Big-Endian 바이트 순서**
```python
struct.Struct(">i")  # > = big-endian
```
- **네트워크 표준**: 대부분의 네트워크 프로토콜과 일치
- **Kafka 호환**: Java의 ByteBuffer와 동일

### 3. **Nullable 인코딩**
```python
# String: length = -1 (Int16)
# Bytes: length = -1 (Int32)
# CompactString: length = 0 (VarInt)
# Array: length = -1 (Int32)
```

### 4. **Compact 타입 최적화**
```
기존 (String):    [Int16 length][data]  → 2 + N bytes
Compact:          [VarInt length+1][data] → 1-5 + N bytes (작은 문자열은 1 + N)

기존 배열 10개:   [Int32=10][...] → 4 bytes
Compact 배열 10개: [VarInt=11][...] → 1 byte
```

---

## 🔗 다른 모듈과의 관계

### 사용처
```
types.py
    ↓ 사용
struct.py (Request/Response 베이스)
    ↓ 사용
metadata.py, produce.py, fetch.py 등 (구체적인 프로토콜)
    ↓ 사용
conn.py (네트워크 전송)
```

### 사용 예시
```python
# metadata.py
from aiokafka.protocol.types import Schema, Array, String, Int32

MetadataRequest_v1 = Schema(
    ('topics', Array(String('utf-8')))
)

# 사용
request = MetadataRequest_v1.encode([["test"]])
# types.py의 Schema, Array, String이 재귀적으로 인코딩
```

---

## 🔑 핵심 특징 요약

| 특징 | 설명 |
|------|------|
| **Big-Endian** | 네트워크 바이트 순서 (Java 호환) |
| **Nullable** | String, Bytes, Array 모두 null 가능 |
| **VarInt** | 작은 숫자 효율적 인코딩 (1-5 bytes) |
| **Compact** | Kafka 2.4+ 공간 최적화 |
| **Tagged Fields** | 향후 호환성 보장 (Flexible API) |
| **재귀 구조** | Schema/Array가 중첩 가능 |
| **타입 안전** | AbstractType 인터페이스로 일관성 |

---

## 🎓 결과적으로 이 파일은

**Kafka Wire Protocol의 타입 시스템 구현체**로서:
1. ✅ **10+ 기본 타입** (Int8 ~ Int64, String, Bytes 등)
2. ✅ **복합 타입** (Schema, Array로 중첩 구조 표현)
3. ✅ **VarInt 인코딩** (작은 값 최적화)
4. ✅ **Compact 타입** (Kafka 2.4+ Flexible API)
5. ✅ **Tagged Fields** (향후 호환성)
6. ✅ **Big-Endian** (Java/네트워크 표준 호환)

→ 모든 프로토콜 정의(`metadata.py`, `produce.py` 등)의 **기반 타입 시스템**
