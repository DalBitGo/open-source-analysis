# dbt-jaffle-shop: 분석 요약

## 1. 프로젝트의 핵심 문제 해결 방식

`dbt-jaffle-shop`은 **"분석을 위해 원본 데이터를 어떻게 신뢰할 수 있는 고품질 데이터 모델로 변환하고 관리해야 하는가?"** 라는 데이터 분석의 근본적인 문제를 해결하기 위한 모범 사례를 제시합니다. 이 프로젝트는 dbt(Data Build Tool)를 사용하여, 복잡하고 지저분할 수 있는 데이터 변환 과정을 **추적 가능하고, 테스트 가능하며, 재사용 가능한** SQL 코드로 관리하는 방법을 보여줍니다.

핵심 해결 방식은 **계층적 모델링(Layered Modeling)** 입니다.

1.  **Staging (1단계)**: 원본 데이터와 1:1로 대응되는 모델을 만들어, 컬럼명 변경, 타입 캐스팅 등 최소한의 정리만 수행합니다. 이는 원본 데이터의 변경이 전체 파이프라인에 미치는 영향을 최소화하는 **방화벽** 역할을 합니다.
2.  **Marts (2단계)**: 잘 정리된 Staging 모델들을 조합하고, 비즈니스 로직(예: 고객 유형 분류, 핵심 지표 계산)을 적용하여 최종 분석가가 사용하기 쉬운 **주제 중심적인 데이터 모델**을 만듭니다.

이러한 접근 방식은 복잡한 데이터 파이프라인을 여러 개의 단순하고 명확한 SQL 모델로 분해하여, 전체 시스템의 **유지보수성, 신뢰성, 협업 효율성**을 극대화합니다.

## 2. 생성된 분석 문서 목록

- **개요**: [01-overview.md](./01-overview.md)
- **아키텍처**: [02-architecture.md](./02-architecture.md)
- **실행 흐름 분석**: [03_execution_flow.md](./03_execution_flow.md)

- **파일 단위 심층 분석 (핵심 파일)**:
    - [`stg_customers.sql.md`](../original/models/staging/stg_customers.sql.md): Staging 모델의 기본 역할(컬럼명 변경) 분석
    - [`stg_orders.sql.md`](../original/models/staging/stg_orders.sql.md): Staging 모델의 심화 역할(단위 변환, 정규화) 및 매크로 활용 분석
    - [`customers.sql.md`](../original/models/marts/customers.sql.md): Marts 모델의 역할(Staging 모델 조합, 비즈니스 로직 적용, KPI 계산) 분석
    - [`cents_to_dollars.sql.md`](../original/macros/cents_to_dollars.sql.md): 매크로를 통한 코드 재사용 및 멀티-데이터베이스 지원 방식 분석

## 3. 가장 흥미로운 설계 결정 3가지

1.  **`ref()`와 `source()`를 통한 동적 의존성 관리**
    - SQL 코드에 테이블명을 하드코딩하는 대신, `{{ ref(...) }}`와 `{{ source(...) }}`를 사용하는 것은 dbt의 가장 강력한 기능입니다. `dbt run` 실행 시, dbt는 이 함수들을 해석하여 모델 간의 의존 관계(DAG)를 동적으로 구성하고 올바른 순서로 모델을 실행합니다. 이는 전체 파이프라인의 유연성과 안정성을 극적으로 향상시킵니다.

2.  **계층적 CTE를 통한 SQL 가독성 확보**
    - 모든 모델이 `with ... as (...)` 형태의 CTE(Common Table Expression)를 여러 단계로 사용하여 작성되었습니다. 각 CTE는 "소스 가져오기", "이름 변경하기", "집계하기", "조인하기" 등 명확한 단일 책임을 갖습니다. 이 덕분에 수백 줄의 복잡한 SQL 쿼리도 각 변환 단계를 쉽게 따라가며 이해할 수 있습니다. 이는 SQL 코드도 소프트웨어 코드처럼 읽기 쉽게 작성해야 한다는 원칙을 잘 보여줍니다.

3.  **`adapter.dispatch`를 활용한 크로스-데이터베이스 매크로**
    - `cents_to_dollars` 매크로에서 `adapter.dispatch`를 사용한 방식은 매우 영리합니다. 동일한 매크로 호출이 현재 dbt가 연결된 데이터 웨어하우스(PostgreSQL, BigQuery 등)에 따라 각기 다른 SQL 코드를 생성하게 만듭니다. 이로써 모델 개발자는 데이터베이스별 SQL 문법 차이를 걱정할 필요 없이 비즈니스 로직에만 집중할 수 있으며, 프로젝트의 이식성(portability)을 크게 높입니다.
