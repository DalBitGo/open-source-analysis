# dbt-jaffle-shop: 실행 흐름 분석

## 1. 핵심 실행 커맨드

`dbt-jaffle-shop` 프로젝트의 핵심 실행 커맨드는 다음과 같습니다.

1.  `dbt run`: 데이터 모델을 데이터 웨어하우스에 생성(빌드)합니다.
2.  `dbt test`: 생성된 데이터 모델의 품질과 정합성을 검증합니다.
3.  `dbt seed`: (선택적) `seeds` 디렉토리의 CSV 파일을 데이터 웨어하우스에 테이블로 로드합니다.

## 2. 커맨드별 실행 흐름 분석

### `dbt run` 실행 시나리오

`dbt run`은 dbt 프로젝트의 심장과 같은 명령어로, SQL로 작성된 모델들을 실제 데이터베이스의 뷰나 테이블로 만들어주는 역할을 합니다. 실행 과정은 다음과 같습니다.

- **입력(Input)**: `models/` 디렉토리의 모든 `.sql` 파일, `dbt_project.yml` 파일, `profiles.yml` (DB 연결정보)

- **처리(Process)**:
    1.  **프로젝트 파싱(Parsing)**: dbt는 먼저 `dbt_project.yml`을 읽어 프로젝트의 설정을 파악하고, `models/` 디렉토리 전체를 스캔하여 모든 SQL 파일을 인식합니다.

    2.  **의존성 그래프(DAG) 생성**: 각 SQL 파일 내의 `{{ ref(...) }}`와 `{{ source(...) }}` 함수를 분석하여 모델 간의 의존 관계를 파악합니다. 이 정보를 바탕으로 어떤 모델을 먼저 실행해야 할지 순서가 정해진 방향성 비순환 그래프(DAG, Directed Acyclic Graph)를 메모리에 생성합니다.
        > 예: `customers.sql`이 `ref('stg_customers')`를 포함하므로, dbt는 `stg_customers`를 먼저 실행해야 한다고 판단합니다.

    3.  **컴파일(Compilation)**: DAG의 순서에 따라 각 모델을 실행 가능한 순수 SQL로 변환(컴파일)합니다. 이 과정에서 `ref` 함수는 실제 데이터베이스의 테이블/뷰 이름(예: `dev.stg_customers`)으로 치환되고, `cents_to_dollars` 같은 매크로는 해당 데이터베이스에 맞는 SQL 구문(예: `(price / 100)::numeric(16, 2)`)으로 변환됩니다.

    4.  **SQL 실행(Execution)**: 컴파일된 SQL을 `profiles.yml`에 설정된 데이터 웨어하우스에 전송하여 실행합니다. `dbt_project.yml`의 설정에 따라 다음과 같이 다른 DDL(데이터 정의어)이 사용됩니다.
        - `models/staging/`의 모델: `CREATE VIEW dev.stg_customers AS (...)` 형태의 SQL이 실행됩니다.
        - `models/marts/`의 모델: `CREATE TABLE dev.customers AS (...)` 형태의 SQL이 실행됩니다.

- **출력(Output)**:
    - 데이터 웨어하우스에 `stg_customers` (뷰), `customers` (테이블) 등 각 모델에 해당하는 뷰와 테이블이 생성됩니다.
    - 터미널에는 각 모델의 실행 결과(성공/실패)가 로그로 출력됩니다.

### `dbt test` 실행 시나리오

`dbt test`는 데이터의 신뢰도를 보장하는 핵심적인 명령어로, 정의된 규칙을 데이터가 잘 지키고 있는지 검증합니다.

- **입력(Input)**: `models/**/*.yml` 파일에 정의된 테스트들 (예: `not_null`, `unique`), `data-tests/` 디렉토리의 커스텀 테스트 SQL 파일

- **처리(Process)**:
    1.  **테스트 수집**: dbt는 `models` 디렉토리의 모든 `.yml` 파일을 스캔하여 각 모델과 컬럼에 적용된 테스트들을 수집합니다.

    2.  **테스트 SQL 생성**: 각 테스트 항목을 **"실패 케이스를 찾는 SQL 쿼리"**로 자동 생성합니다.
        > 예: `customers` 모델의 `customer_id` 컬럼에 `unique` 테스트가 정의된 경우, dbt는 다음과 같은 SQL을 생성합니다:
        > `select customer_id from dev.customers where customer_id is not null group by customer_id having count(*) > 1`

        > 예: `not_null` 테스트의 경우, 다음과 같은 SQL을 생성합니다:
        > `select * from dev.customers where customer_id is null`

    3.  **SQL 실행 및 결과 확인**: 생성된 SQL 쿼리들을 데이터 웨어하우스에 실행합니다. 만약 쿼리 결과로 **하나 이상의 레코드(row)가 반환되면, 이는 규칙을 위반한 데이터가 존재한다는 의미이므로 해당 테스트는 '실패(Failure)'**로 간주됩니다. 결과가 0건이면 '성공(Success)'입니다.

- **출력(Output)**:
    - 터미널에 각 테스트의 실행 결과(PASS/FAIL)와 실패한 레코드의 수가 요약되어 출력됩니다.
    - 테스트 실패 시, 0이 아닌 종료 코드(exit code)를 반환하여 CI/CD 파이프라인에서 후속 작업을 중단시킬 수 있습니다.
