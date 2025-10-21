# fastapi-best-practices: 아키텍처 분석

## 1. 제안된 구조 (디렉토리)

이 프로젝트는 코드가 아닌 문서이므로, `README.md`에서 제안하는 **도메인 기반(domain-driven)의 프로젝트 구조**가 바로 분석 대상 아키텍처입니다. 이는 여러 도메인을 가진 모놀리식 애플리케이션에 최적화된 구조입니다.

```
fastapi-project/
├── src/
│   ├── auth/               # "인증" 도메인
│   │   ├── router.py       # API 엔드포인트 (Controller)
│   │   ├── schemas.py      # Pydantic 모델 (DTO)
│   │   ├── models.py       # DB 모델 (ORM)
│   │   ├── service.py      # 비즈니스 로직
│   │   ├── dependencies.py # 의존성 주입 (DI)
│   │   └── ...
│   │
│   ├── posts/              # "게시물" 도메인
│   │   └── ... (auth와 동일한 구조)
│   │
│   ├── config.py           # 전역 설정
│   ├── database.py         # DB 연결
│   └── main.py             # FastAPI 앱 초기화
│
└── tests/                  # 테스트 코드 (src와 동일한 구조)
```

## 2. 구조의 목적성: 왜 이런 구조를 선택했는가?

이 아키텍처의 핵심 목표는 **높은 응집도(High Cohesion)와 낮은 결합도(Low Coupling)** 를 달성하여, 프로젝트가 커져도 복잡성을 제어하고 유지보수를 쉽게 만드는 것입니다.

1.  **도메인(기능) 중심의 모듈화 (`src/auth`, `src/posts`)**
    *   **설계**: 관련된 모든 코드(라우터, 스키마, 서비스, 모델)를 하나의 도메인 디렉토리 안에 함께 배치합니다. 예를 들어, '인증'과 관련된 모든 기능은 `src/auth` 안에서 찾을 수 있습니다.
    *   **목적**: 이는 **높은 응집도**를 만들어냅니다. 특정 기능을 수정하거나 새로운 기능을 추가할 때, 개발자는 여러 디렉토리를 헤맬 필요 없이 해당 도메인 디렉토리 내에서 대부분의 작업을 완료할 수 있습니다. 이는 코드의 위치를 예측하기 쉽게 만들고, 개발자 경험을 크게 향상시킵니다.
    *   **대안과의 비교**: 파일 유형별(`routers/`, `models/`)로 구성하는 방식은, 기능 하나를 수정하기 위해 여러 디렉토리를 오가야 하므로 코드의 파편화가 심해지고 응집도가 낮아집니다.

2.  **계층적 책임 분리 (router → service → model)**
    *   **`router.py` (API 계층)**: HTTP 요청을 받고, 응답을 반환하는 역할만 책임집니다. 요청 데이터를 파싱하고, 적절한 서비스 함수를 호출한 뒤, 그 결과를 HTTP 응답으로 변환합니다.
    *   **`service.py` (비즈니스 로직 계층)**: 실제 비즈니스 규칙과 로직을 수행합니다. 예를 들어, "게시물을 생성하기 전에 사용자가 글쓰기 권한이 있는지 확인한다"와 같은 로직이 여기에 위치합니다. 데이터베이스 모델이나 외부 API와 직접 상호작용합니다.
    *   **`models.py` (데이터 접근 계층)**: 데이터베이스 테이블의 구조를 정의합니다. `service.py`는 이 모델을 통해 DB에 데이터를 읽고 씁니다.
    *   **목적**: 각 계층이 명확한 단일 책임(Single Responsibility)을 갖게 하여 **낮은 결합도**를 유지합니다. 예를 들어, 데이터베이스 스키마가 변경되더라도 `models.py`와 `service.py`의 일부만 수정하면 되며, API 계층인 `router.py`는 영향을 받지 않을 수 있습니다.

3.  **데이터 형태의 명확한 분리 (`schemas.py` vs `models.py`)**
    *   **`schemas.py` (Pydantic 모델)**: API를 통해 외부와 주고받는 데이터의 형태(DTO, Data Transfer Object)를 정의합니다. 요청 본문(request body)의 유효성 검사나 응답 본문(response body)의 형태를 정의하는 데 사용됩니다.
    *   **`models.py` (SQLAlchemy/ORM 모델)**: 데이터베이스 테이블의 구조와 관계를 정의합니다.
    *   **목적**: API의 데이터 형태와 데이터베이스의 데이터 형태를 분리함으로써, 두 계층 사이의 결합을 끊습니다. 데이터베이스 스키마에 내부적으로만 사용하는 컬럼이 추가되더라도, API 응답에는 노출되지 않도록 제어할 수 있습니다. 이는 API의 안정성과 보안을 높여줍니다.

4.  **의존성 주입의 적극적 활용 (`dependencies.py`)**
    *   **설계**: "요청한 사용자가 이 게시물의 소유자인가?"와 같이 여러 엔드포인트에서 반복적으로 필요한 검증 로직을 별도의 의존성 함수로 분리하여 `dependencies.py`에 모아둡니다.
    *   **목적**: 반복적인 코드를 제거(DRY)하고, 검증 로직을 재사용 가능하게 만듭니다. 또한 라우터 함수의 본문은 순수한 비즈니스 로직에만 집중할 수 있게 하여 코드를 더 깔끔하게 유지합니다.

## 3. 아키텍처 다이어그램 (요청 처리 흐름)

이 구조에서 일반적인 HTTP 요청은 다음과 같은 흐름으로 처리됩니다.

```mermaid
graph TD
    A[Client] -- HTTP Request --> B(FastAPI)
    
    subgraph "API Layer (router.py)"
        B --> C{API Endpoint<br/>/posts/{post_id}}
        C -- Depends --> D[Dependency<br/>(dependencies.py)]
    end

    subgraph "Business Logic Layer (service.py)"
        D --> E{Service Function<br/>get_post_by_id(...)}
    end

    subgraph "Data Access Layer"
        E -- Uses --> F[ORM Model<br/>(models.py)]
        F <--> G[(Database)]
    end

    E -- Returns Data --> C
    C -- Pydantic Serialization<br/>(schemas.py) --> B
    B -- HTTP Response --> A

```

1.  클라이언트가 특정 엔드포인트(예: `/posts/123`)로 **요청**을 보냅니다.
2.  `router.py`의 해당 엔드포인트 함수가 요청을 받습니다.
3.  엔드포인트에 정의된 **의존성 함수**(`dependencies.py`)가 먼저 실행되어 요청의 유효성(예: 게시물 존재 여부, 사용자 권한)을 검증합니다.
4.  검증이 통과되면, `router.py`는 `service.py`에 정의된 **서비스 함수**를 호출하여 실제 비즈니스 로직을 위임합니다.
5.  `service.py`는 `models.py`에 정의된 **ORM 모델**을 사용하여 데이터베이스와 상호작용합니다.
6.  `service.py`가 처리 결과를 반환하면, `router.py`는 `schemas.py`에 정의된 **Pydantic 모델**을 사용하여 응답 데이터의 형태를 만들고 직렬화합니다.
7.  최종적으로 FastAPI를 통해 클라이언트에게 **HTTP 응답**이 전송됩니다.
