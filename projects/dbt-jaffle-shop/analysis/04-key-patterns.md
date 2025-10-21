# dbt jaffle-shop 핵심 패턴 추출

> **목표**: 재사용 가능한 패턴을 추출해서 내 프로젝트에 바로 적용

---

## 패턴 1: CTE 체인 (SQL 모듈화)

### 문제
- 복잡한 SQL은 읽기도 수정하기도 어려움
- 디버깅할 때 중간 결과 확인 불가능

### 패턴
```sql
with

-- Step 1: 데이터 로드 (명확한 이름)
source_data as (
    select * from {{ ref('stg_xxx') }}
),

-- Step 2: 필터링/정제
filtered as (
    select *
    from source_data
    where is_valid = true
),

-- Step 3: 집계
aggregated as (
    select
        group_key,
        count(*) as total_count,
        sum(amount) as total_amount
    from filtered
    group by 1
),

-- Step 4: 최종 변환
final as (
    select
        *,
        total_amount / total_count as avg_amount
    from aggregated
)

select * from final
```

### 핵심 규칙

1. **CTE 이름 = 동사형**
   - ✅ `filtered`, `aggregated`, `joined`, `ranked`
   - ❌ `temp1`, `t2`, `data`

2. **한 CTE당 한 가지 일**
   - ✅ `filtered`: 필터링만
   - ❌ `filtered_and_aggregated`: 2가지 일

3. **마지막은 항상 `final`**
   ```sql
   select * from final
   ```

### 언제 사용?

| 상황 | 사용 여부 |
|------|----------|
| SELECT 3줄 이하 | ❌ 불필요 |
| SELECT 10-50줄 | ✅ 권장 |
| SELECT 100줄+ | ✅ 필수! |
| Join 2개 이상 | ✅ 필수! |
| 집계 + Join | ✅ 필수! |

### 실전 적용

**Before**:
```sql
-- 150줄 단일 쿼리
SELECT
    c.id,
    SUM(...),
    AVG(...),
    ...
FROM customers c
JOIN orders o ON ...
WHERE ...
GROUP BY ...
```

**After**:
```sql
with
customers as (select * from ...),      -- 10줄
orders as (select * from ...),         -- 15줄
customer_metrics as (                  -- 20줄
    select ... from orders group by ...
),
final as (                             -- 10줄
    select ... from customers join customer_metrics ...
)
select * from final
```

---

## 패턴 2: Staging 표준화 레이어

### 문제
- Raw 데이터는 소스마다 형식 다름
- 컬럼명 불일치
- 데이터 타입 불일치

### 패턴

**stg_{entity}.sql**:
```sql
with

source as (
    select * from {{ source('raw', 'table_name') }}
),

renamed as (
    select
        -- IDs: {entity}_id 형식
        id as customer_id,
        external_id as external_customer_id,

        -- 텍스트: {entity}_{attribute}
        name as customer_name,
        email as customer_email,

        -- Boolean: is_{attribute}
        active as is_active,
        verified as is_verified,

        -- 숫자: 단위 명시
        balance_cents,
        {{ cents_to_dollars('balance_cents') }} as balance,

        -- 날짜: {action}_at
        created_at,
        {{ dbt.date_trunc('day', 'created_at') }} as created_date

    from source
)

select * from renamed
```

### 네이밍 컨벤션

| 타입 | 패턴 | 예시 |
|------|------|------|
| Primary Key | `{entity}_id` | `customer_id`, `order_id` |
| Foreign Key | `{entity}_id` | `customer_id` (orders 테이블에서) |
| Boolean | `is_{attribute}` | `is_active`, `is_deleted` |
| Count | `count_{entity}` | `count_orders` |
| 날짜 | `{action}_at` | `created_at`, `updated_at` |
| 금액 | `{attribute}` | `subtotal`, `tax_paid` |

### 언제 사용?

✅ **항상 사용!**
- Raw 데이터를 직접 Marts에서 쓰지 말 것
- Staging은 "정제 계층"

### 실전 적용

**내 프로젝트 규칙**:
```yaml
# dbt_project.yml
models:
  my_project:
    staging:
      +materialized: view        # Staging은 View
      +schema: staging
    marts:
      +materialized: table       # Marts는 Table
      +schema: analytics
```

---

## 패턴 3: ref() 의존성 관리

### 문제
- SQL 파일 실행 순서 관리 복잡
- 순환 참조 감지 못함
- 병렬 실행 불가능

### 패턴

**절대 하지 말 것**:
```sql
-- ❌ 테이블 직접 참조
SELECT * FROM analytics.customers

-- ❌ 하드코딩
SELECT * FROM dev_schema.orders
```

**항상 이렇게**:
```sql
-- ✅ ref() 사용
SELECT * FROM {{ ref('customers') }}
SELECT * FROM {{ ref('stg_orders') }}
```

### dbt가 자동으로 하는 일

```
1. 모든 ref() 스캔
2. DAG 생성
3. 실행 순서 결정
4. 병렬 가능한 것 병렬 실행
5. 순환 참조 감지
```

### 실행 순서 예시

```
# dbt run 실행 시:

[1] 병렬 실행 (의존성 없음)
    - stg_customers
    - stg_orders
    - stg_products

[2] orders 실행 (stg_orders 완료 후)

[3] 병렬 실행
    - customers (stg_customers + orders 완료 후)
    - order_items (orders 완료 후)
```

### 실전 팁

**특정 모델만 실행**:
```bash
# 한 모델만
dbt run --select customers

# 한 모델 + upstream (의존성)
dbt run --select +customers

# 한 모델 + downstream (이걸 쓰는 것들)
dbt run --select customers+

# 전체
dbt run --select +customers+
```

---

## 패턴 4: 계층별 책임 분리

### Staging Layer

**책임**:
- ✅ 컬럼명 표준화
- ✅ 데이터 타입 캐스팅
- ✅ 기본 필터링 (`where deleted_at is null`)
- ✅ 1:1 소스 매핑 (1 source = 1 staging model)

**금지**:
- ❌ 집계 (`group by`)
- ❌ Join
- ❌ 복잡한 비즈니스 로직

**Example**:
```sql
-- ✅ Good
select
    id as order_id,
    status as order_status
from source
where deleted_at is null

-- ❌ Bad (집계는 Marts에서)
select
    customer_id,
    count(*) as total_orders
from source
group by 1
```

---

### Marts Layer

**책임**:
- ✅ 비즈니스 로직
- ✅ 집계
- ✅ Join
- ✅ Window Function
- ✅ 분석가가 쓸 수 있는 형태

**금지**:
- ❌ Raw 테이블 직접 참조
- ❌ 과도한 중첩 (Intermediate로 분리)

**Example**:
```sql
-- ✅ Good
with
orders as (select * from {{ ref('stg_orders') }}),
aggregated as (
    select
        customer_id,
        count(*) as total_orders,
        sum(total) as lifetime_value
    from orders
    group by 1
)
select * from aggregated

-- ❌ Bad (raw 직접 참조)
select * from raw.orders
```

---

### Intermediate Layer (선택적)

**언제 필요한가?**
- Marts가 100줄+ 넘어갈 때
- 같은 Join을 여러 Marts에서 재사용할 때
- 복잡한 Window Function 중간 단계

**Example**:
```sql
-- intermediate/customer_order_history.sql
with
orders as (select * from {{ ref('stg_orders') }}),
payments as (select * from {{ ref('stg_payments') }}),
joined as (
    select ...
    from orders
    join payments on ...
)
select * from joined

-- marts/customers.sql
-- ✅ 복잡한 Join 재사용
select * from {{ ref('customer_order_history') }}
```

---

## 패턴 5: Window Function 활용

### 문제
- "N번째 행", "순위", "누적합" 계산 복잡

### 패턴

**고객별 N번째 주문**:
```sql
select
    *,
    row_number() over (
        partition by customer_id
        order by ordered_at asc
    ) as customer_order_number
from orders
```

**누적 합계**:
```sql
select
    *,
    sum(order_total) over (
        partition by customer_id
        order by ordered_at asc
        rows between unbounded preceding and current row
    ) as cumulative_spend
from orders
```

**이동 평균 (최근 3개)**:
```sql
select
    *,
    avg(order_total) over (
        partition by customer_id
        order by ordered_at asc
        rows between 2 preceding and current row
    ) as moving_avg_3
from orders
```

### 실전 활용

| 비즈니스 질문 | Window Function |
|--------------|-----------------|
| 첫 구매 고객 | `where row_number() = 1` |
| 재구매 고객 | `where row_number() > 1` |
| Top 10 상품 | `where rank() <= 10` |
| 월별 누적 매출 | `sum(...) over (order by month)` |

---

## 패턴 6: Macro로 재사용

### 문제
- 같은 로직을 여러 모델에서 반복

### 패턴

**macros/cents_to_dollars.sql**:
```sql
{% macro cents_to_dollars(column_name) %}
    ({{ column_name }} / 100.0)::numeric(10,2)
{% endmacro %}
```

**사용**:
```sql
select
    {{ cents_to_dollars('subtotal_cents') }} as subtotal,
    {{ cents_to_dollars('tax_cents') }} as tax
from orders
```

### 자주 쓰는 Macro 패턴

**1. 타입 변환**:
```sql
{% macro safe_cast(column_name, data_type) %}
    cast({{ column_name }} as {{ data_type }})
{% endmacro %}
```

**2. Null 처리**:
```sql
{% macro default_value(column_name, default) %}
    coalesce({{ column_name }}, {{ default }})
{% endmacro %}
```

**3. 날짜 계산**:
```sql
{% macro days_between(start_date, end_date) %}
    datediff('day', {{ start_date }}, {{ end_date }})
{% endmacro %}
```

---

## 패턴 요약

| 패턴 | 해결하는 문제 | 사용 시점 |
|------|--------------|----------|
| **CTE 체인** | 복잡한 SQL | 50줄+ 쿼리 |
| **Staging** | 불일치 데이터 | 항상 |
| **ref()** | 의존성 관리 | 항상 |
| **계층 분리** | 책임 혼재 | 항상 |
| **Window Fn** | 순위/누적 | N번째, 순위 필요 시 |
| **Macro** | 로직 중복 | 3회 이상 반복 시 |

---

## 다음 문서

**05-apply-to-my-work.md**에서:
- 이 패턴들을 **내 프로젝트에 어떻게 적용**할지
- 구체적인 실행 계획
