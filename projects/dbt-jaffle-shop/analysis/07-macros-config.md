# dbt jaffle-shop Macros & 설정 분석

> **목표**: Macro 구현과 프로젝트 설정을 상세 분석

---

## Macros (2개)

### 1. cents_to_dollars

**파일**: `macros/cents_to_dollars.sql`

#### 전체 코드
```sql
{# A basic example for a project-wide macro to cast a column uniformly #}

{% macro cents_to_dollars(column_name) -%}
    {{ return(adapter.dispatch('cents_to_dollars')(column_name)) }}
{%- endmacro %}

{% macro default__cents_to_dollars(column_name) -%}
    ({{ column_name }} / 100)::numeric(16, 2)
{%- endmacro %}

{% macro postgres__cents_to_dollars(column_name) -%}
    ({{ column_name }}::numeric(16, 2) / 100)
{%- endmacro %}

{% macro bigquery__cents_to_dollars(column_name) %}
    round(cast(({{ column_name }} / 100) as numeric), 2)
{% endmacro %}

{% macro fabric__cents_to_dollars(column_name) %}
    cast({{ column_name }} / 100 as numeric(16,2))
{% endmacro %}
```

---

#### 구조 분석

**1단계: 메인 Macro (Dispatcher)**
```sql
{% macro cents_to_dollars(column_name) -%}
    {{ return(adapter.dispatch('cents_to_dollars')(column_name)) }}
{%- endmacro %}
```

**역할**:
- `adapter.dispatch()`: 현재 DB 어댑터에 맞는 Macro 자동 선택
- 예시:
  - PostgreSQL → `postgres__cents_to_dollars()` 호출
  - BigQuery → `bigquery__cents_to_dollars()` 호출
  - 기타 → `default__cents_to_dollars()` 호출

---

**2단계: DB별 구현**

| DB | Macro | 변환 로직 | 차이점 |
|----|-------|----------|--------|
| **Default** | `default__cents_to_dollars` | `(column / 100)::numeric(16, 2)` | 나누기 먼저 → 캐스트 |
| **PostgreSQL** | `postgres__cents_to_dollars` | `(column::numeric(16, 2) / 100)` | 캐스트 먼저 → 나누기 |
| **BigQuery** | `bigquery__cents_to_dollars` | `round(cast((column / 100) as numeric), 2)` | 명시적 반올림 |
| **Fabric** | `fabric__cents_to_dollars` | `cast(column / 100 as numeric(16,2))` | Default와 유사 |

---

#### 핵심 학습 포인트

**1. Cross-Database 호환성**
```sql
-- ✅ 하나의 Macro로 4개 DB 지원
{{ cents_to_dollars('subtotal') }}

-- ❌ DB마다 수동 분기하지 않음
{% if target.type == 'postgres' %}
    (subtotal::numeric / 100)
{% elif target.type == 'bigquery' %}
    round(cast(subtotal / 100 as numeric), 2)
{% endif %}
```

**2. DRY 원칙 (Don't Repeat Yourself)**
```sql
-- ✅ Macro 사용 (1곳에서 관리)
{{ cents_to_dollars('subtotal') }}
{{ cents_to_dollars('tax_paid') }}
{{ cents_to_dollars('order_total') }}

-- ❌ 중복 코드
(subtotal / 100)::numeric(16, 2)
(tax_paid / 100)::numeric(16, 2)
(order_total / 100)::numeric(16, 2)
```

**3. 데이터 타입 안전성**
- `numeric(16, 2)`: 정수 16자리, 소수 2자리
- 예시: `12345.67` (14자리 정수 + 소수 2자리)
- 금액 계산에 적합 (부동소수점 오차 없음)

---

#### 사용 예시

**Before (Raw)**:
```sql
-- raw_orders 테이블
subtotal: 1250  (cents)
tax_paid: 125   (cents)
```

**After (Staging)**:
```sql
-- stg_orders.sql
select
    subtotal as subtotal_cents,                    -- 1250
    {{ cents_to_dollars('subtotal') }} as subtotal -- 12.50
from source
```

**결과**:
| subtotal_cents | subtotal |
|----------------|----------|
| 1250 | 12.50 |
| 125 | 1.25 |

---

### 2. generate_schema_name

**파일**: `macros/generate_schema_name.sql`

#### 전체 코드
```sql
{% macro generate_schema_name(custom_schema_name, node) %}

    {% set default_schema = target.schema %}

    {# seeds go in a global `raw` schema #}
    {% if node.resource_type == 'seed' %}
        {{ custom_schema_name | trim }}

    {# non-specified schemas go to the default target schema #}
    {% elif custom_schema_name is none %}
        {{ default_schema }}


    {# specified custom schema names go to the schema name prepended with the the default schema name in prod (as this is an example project we want the schemas clearly labeled) #}
    {% elif target.name == 'prod' %}
        {{ default_schema }}_{{ custom_schema_name | trim }}

    {# specified custom schemas go to the default target schema for non-prod targets #}
    {% else %}
        {{ default_schema }}
    {% endif %}

{% endmacro %}
```

---

#### 구조 분석

**입력**:
- `custom_schema_name`: 모델에서 지정한 스키마 이름
- `node`: dbt 노드 객체 (resource_type 포함)

**출력**: 실제 DB 스키마 이름

---

#### 로직 흐름

```
1. node.resource_type == 'seed'?
   YES → raw (Seeds는 무조건 raw 스키마)
   NO  → 2번으로

2. custom_schema_name이 없음?
   YES → default_schema (예: dev_alice)
   NO  → 3번으로

3. target.name == 'prod'?
   YES → {default_schema}_{custom_schema} (예: prod_staging)
   NO  → default_schema (예: dev_alice)
```

---

#### 실전 예시

**설정**:
```yaml
# dbt_project.yml
models:
  jaffle_shop:
    staging:
      +schema: staging
    marts:
      +schema: analytics

seeds:
  jaffle_shop:
    +schema: raw
```

**결과**:

| 환경 | 리소스 타입 | custom_schema | 최종 스키마 |
|------|------------|---------------|------------|
| **Dev** | seed | raw | `raw` |
| **Dev** | staging | staging | `dev_alice` |
| **Dev** | marts | analytics | `dev_alice` |
| **Prod** | seed | raw | `raw` |
| **Prod** | staging | staging | `prod_staging` |
| **Prod** | marts | analytics | `prod_analytics` |

---

#### 핵심 학습 포인트

**1. 환경별 스키마 분리**
```
Dev (개발자별):
  - dev_alice
  - dev_bob

Prod (명확한 구분):
  - prod_staging
  - prod_analytics
  - raw
```

**2. Seeds는 전역 스키마**
```sql
-- ✅ 모든 환경에서 같은 스키마
raw.raw_customers  -- Dev도 raw, Prod도 raw
```

**이유**:
- Seeds는 공통 참조 데이터
- 환경별 복제 방지

**3. Dev 환경 단순화**
```
Dev: 모두 dev_alice
→ 스키마 1개로 관리 (개발 편의성)

Prod: prod_staging, prod_analytics 분리
→ 계층 명확화 (운영 안정성)
```

---

## 프로젝트 설정

### dbt_project.yml

**파일**: `dbt_project.yml`

#### 전체 내용
```yaml
config-version: 2

name: "jaffle_shop"
version: "3.0.0"
require-dbt-version: ">=1.5.0"

dbt-cloud:
  project-id: 275557 # Put your project id here

profile: default # Put your profile here

model-paths: ["models"]
analysis-paths: ["analyses"]
test-paths: ["data-tests"]
seed-paths: ["seeds"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]

target-path: "target"
clean-targets:
  - "target"
  - "dbt_packages"

vars:
  "dbt_date:time_zone": "America/Los_Angeles"

seeds:
  jaffle_shop:
    +schema: raw
    jaffle-data:
      +enabled: "{{ var('load_source_data', false) }}"

models:
  jaffle_shop:
    staging:
      +materialized: view
    marts:
      +materialized: table
```

---

#### 섹션별 분석

**1. 프로젝트 메타데이터**
```yaml
name: "jaffle_shop"
version: "3.0.0"
require-dbt-version: ">=1.5.0"
```

| 항목 | 값 | 의미 |
|------|-----|------|
| `name` | jaffle_shop | 프로젝트 고유 이름 |
| `version` | 3.0.0 | 프로젝트 버전 (Semantic Versioning) |
| `require-dbt-version` | >=1.5.0 | dbt 1.5.0 이상 필요 |

---

**2. 디렉토리 구조**
```yaml
model-paths: ["models"]
analysis-paths: ["analyses"]
test-paths: ["data-tests"]
seed-paths: ["seeds"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]
```

**표준 디렉토리 레이아웃**:
```
jaffle_shop/
├── models/          # SQL 모델
├── analyses/        # 일회성 분석 쿼리
├── data-tests/      # 커스텀 데이터 테스트
├── seeds/           # CSV 파일
├── macros/          # Jinja Macro
└── snapshots/       # SCD Type 2 스냅샷
```

---

**3. 전역 변수**
```yaml
vars:
  "dbt_date:time_zone": "America/Los_Angeles"
```

**용도**:
- `dbt_date` 패키지에서 시간대 설정
- `metricflow_time_spine.sql`에서 사용

**예시**:
```sql
-- 시간대 변환 시
{{ dbt_date.convert_timezone('UTC', var('dbt_date:time_zone'), 'created_at') }}
→ UTC → America/Los_Angeles 변환
```

---

**4. Seeds 설정**
```yaml
seeds:
  jaffle_shop:
    +schema: raw
    jaffle-data:
      +enabled: "{{ var('load_source_data', false) }}"
```

**해석**:

| 설정 | 값 | 의미 |
|------|-----|------|
| `+schema: raw` | raw | Seeds는 `raw` 스키마에 로드 |
| `+enabled: ...` | 조건부 활성화 | `load_source_data=true`일 때만 로드 |

**실행 예시**:
```bash
# 기본 (Seeds 로드 안 함)
dbt seed

# Seeds 로드
dbt seed --vars '{"load_source_data": true}'
```

---

**5. Models 설정** ⭐ 가장 중요
```yaml
models:
  jaffle_shop:
    staging:
      +materialized: view
    marts:
      +materialized: table
```

**Materialization 전략**:

| Layer | Materialized | SQL | 이유 |
|-------|--------------|-----|------|
| **Staging** | `view` | `CREATE VIEW ...` | - 데이터 신선도<br>- 저장 공간 절약<br>- 단순 변환만 |
| **Marts** | `table` | `CREATE TABLE ...` | - 복잡한 집계<br>- 쿼리 성능<br>- 분석가 대상 |

**예시**:
```sql
-- stg_customers.sql (View)
dbt run --select stg_customers
→ CREATE VIEW staging.stg_customers AS ...

-- customers.sql (Table)
dbt run --select customers
→ CREATE TABLE analytics.customers AS ...
```

---

#### Materialization 옵션

| 타입 | SQL | 용도 |
|------|-----|------|
| **view** | `CREATE VIEW` | Staging, 빠른 변환 |
| **table** | `CREATE TABLE` | Marts, 복잡한 집계 |
| **incremental** | `INSERT INTO` | 대용량, 증분 업데이트 |
| **ephemeral** | CTE | 중간 단계, 저장 안 함 |

**프로젝트별 선택 기준**:
- **< 100만 행**: view/table
- **> 100만 행**: incremental
- **중간 단계**: ephemeral

---

## 패키지 의존성

### packages.yml (추정)

이 프로젝트는 2개 패키지 사용:
1. **dbt_utils**: `generate_surrogate_key()` 등
2. **dbt_date**: `get_base_dates()` 등

**예상 설정**:
```yaml
packages:
  - package: dbt-labs/dbt_utils
    version: 1.0.0
  - package: calogica/dbt_date
    version: 0.7.0
```

**설치**:
```bash
dbt deps  # packages/ 디렉토리에 다운로드
```

---

## 핵심 학습 포인트

### 1. Macro 작성 패턴

**Cross-DB Macro 템플릿**:
```sql
{# 1. Dispatcher #}
{% macro my_macro(column_name) -%}
    {{ return(adapter.dispatch('my_macro')(column_name)) }}
{%- endmacro %}

{# 2. Default 구현 #}
{% macro default__my_macro(column_name) -%}
    -- 기본 로직
{%- endmacro %}

{# 3. DB별 오버라이드 (선택적) #}
{% macro postgres__my_macro(column_name) -%}
    -- PostgreSQL 특화 로직
{%- endmacro %}
```

---

### 2. 스키마 네이밍 전략

**옵션 A: 환경별 Prefix**
```
Dev:  dev_alice.customers
Prod: prod.customers
```

**옵션 B: 계층별 Suffix** (jaffle_shop 방식)
```
Dev:  dev_alice.customers
Prod: prod_staging.stg_customers
      prod_analytics.customers
```

**선택 기준**:
- 팀 크기 작음 → 옵션 A (단순)
- 팀 크기 큼 → 옵션 B (명확한 계층)

---

### 3. Materialization 결정 트리

```
1. 데이터 크기는?
   < 1M 행 → 2번
   > 1M 행 → incremental

2. 집계 있나?
   없음 → view
   있음 → 3번

3. 재사용 빈도는?
   낮음 (1-2회) → view
   높음 (10+회) → table
```

---

### 4. 실전 적용

**내 프로젝트에서 Macro 만들기**:

**예시 1: 이메일 정제**
```sql
{% macro clean_email(column_name) -%}
    lower(trim({{ column_name }}))
{%- endmacro %}
```

**예시 2: 날짜 차이 (일)**
```sql
{% macro days_between(start_date, end_date) -%}
    {{ return(adapter.dispatch('days_between')(start_date, end_date)) }}
{%- endmacro %}

{% macro default__days_between(start_date, end_date) -%}
    datediff('day', {{ start_date }}, {{ end_date }})
{%- endmacro %}

{% macro bigquery__days_between(start_date, end_date) -%}
    date_diff({{ end_date }}, {{ start_date }}, day)
{%- endmacro %}
```

---

## 다음 단계

**08-dependencies-visualization.md**에서:
- DAG(의존성 그래프) 시각화
- 실행 순서 상세 분석
- 병렬 실행 전략
