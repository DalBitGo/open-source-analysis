# confluent-kafka-python: 아키텍처 분석

## 1. 전체 구조 (디렉토리)

`confluent-kafka-python`의 디렉토리 구조는 **C언어 기반의 핵심 라이브러리(`librdkafka`)**와 이를 감싸는 **Python 인터페이스**를 명확히 분리하고, 사용자를 위한 **예제와 테스트**를 충실히 제공하는 형태로 설계되었습니다.

```
confluent-kafka-python/
├── src/confluent_kafka/  # Python 소스 코드
│   ├── __init__.py
│   ├── admin/            # AdminClient 관련 API
│   ├── cimpl/            # C 구현 래퍼 (CPython Implementation)
│   ├── error.py          # Kafka 에러/예외 정의
│   ├── experimental/     # 실험적 기능 (예: aio)
│   └── schema_registry/  # Schema Registry 클라이언트
│
├── examples/             # 다양한 사용 시나리오 예제 코드
├── tests/                # 단위/통합 테스트 코드
├── docs/                 # 문서
├── setup.py              # 빌드 및 설치 스크립트
└── ...
```

## 2. 구조의 목적성: 왜 이런 구조를 선택했는가?

이 프로젝트의 핵심 목표는 **"Python의 편의성 + C의 성능"** 입니다. 폴더 구조는 이 목표를 효율적으로 달성하기 위해 다음과 같이 설계되었습니다.

1.  **성능과 편의성의 분리 (`src/confluent_kafka/cimpl` vs `src/confluent_kafka`)**
    *   **`cimpl/`**: CPython 구현부를 의미합니다. 이곳에 `librdkafka`의 C 함수들을 직접 호출하고 Python 객체로 변환하는 저수준(low-level) 코드가 위치합니다. 성능이 중요한 모든 로직은 여기서 처리됩니다.
    *   **`confluent_kafka/`**: `cimpl`의 저수준 객체들을 사용자가 다루기 쉬운 고수준(high-level) Python 클래스(`Producer`, `Consumer` 등)로 감싸는 역할을 합니다. 사용자는 이 인터페이스만 보고 코드를 작성하므로, 복잡한 C언어 내부 구현을 알 필요가 없습니다.
    *   **목적**: 이 분리 구조 덕분에, 개발자들은 성능 최적화는 C 레벨에 집중하고, API의 유연성과 편의성은 Python 레벨에서 개선할 수 있습니다. 즉, **관심사를 명확히 분리**하여 유지보수성과 개발 효율성을 높입니다.

2.  **사용자 경험 중심의 설계 (`examples/` 와 `tests/`)**
    *   **`examples/`**: 단순한 `Producer`/`Consumer` 예제부터 `asyncio` 통합, `Schema Registry` 사용, 트랜잭션 API 예제까지 다양한 실전 시나리오를 제공합니다. 이는 라이브러리의 학습 곡선을 낮추고, 사용자가 복잡한 기능을 쉽게 도입할 수 있도록 돕습니다.
    *   **`tests/`**: 방대한 테스트 코드는 C 레벨의 저수준 기능과 Python 고수준 API의 동작을 모두 검증합니다. 이는 라이브러리의 **신뢰성**을 보장하며, 특히 프로덕션 환경에서 안정성이 중요한 Kafka 클라이언트에게 필수적인 요소입니다.

3.  **확장성을 고려한 모듈화 (`admin/`, `schema_registry/`, `experimental/`)**
    *   Kafka의 핵심 기능인 메시지 송수신 외에, 부가적인 기능들을 별도의 모듈로 분리했습니다.
    *   `admin/`: 토픽 생성/삭제 등 관리 기능
    *   `schema_registry/`: 스키마 관리 및 직렬화 기능
    *   `experimental/`: `AIOProducer`와 같은 새로운 기능을 안정적인 메인 코드와 분리하여 점진적으로 도입할 수 있게 합니다.
    *   **목적**: 새로운 기능이 추가되더라도 기존 코드에 미치는 영향을 최소화하고, 라이브러리의 **확장성과 유지보수성**을 높입니다.

## 3. 아키텍처 다이어그램 (Mermaid)

이 라이브러리의 핵심 아키텍처는 다음과 같이 표현할 수 있습니다.

```mermaid
graph TD
    subgraph "User Application (Python)"
        A[High-Level API<br/>Producer, Consumer, AdminClient] --> B{Python Wrapper<br/>(src/confluent_kafka/)}
    end

    subgraph "confluent-kafka-python Library"
        B --> C{CPython Glue Code<br/>(src/confluent_kafka/cimpl)}
        C --> D[librdkafka<br/>(High-performance C Library)]
    end

    subgraph "Apache Kafka Cluster"
        D <--> E[Kafka Brokers]
    end

    F[Schema Registry] <--> B
```

- **사용자 애플리케이션**: `Producer`, `Consumer` 등 사용하기 쉬운 Python 클래스를 호출합니다.
- **Python Wrapper**: 사용자의 호출을 받아 내부 CPython 모듈로 전달하는 역할을 합니다.
- **CPython Glue Code**: Python 객체와 C 구조체 간의 변환을 처리하고, `librdkafka`의 함수를 직접 호출합니다. **성능의 핵심**입니다.
- **librdkafka**: 실제 Kafka 브로커와 통신하는 모든 저수준 작업을 수행합니다.
- **Schema Registry**: (선택적으로) Python Wrapper 단에서 Schema Registry와 통신하여 스키마를 관리합니다.
