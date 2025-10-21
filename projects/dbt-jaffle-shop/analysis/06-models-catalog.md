# dbt jaffle-shop 모델 전체 카탈로그

> **목표**: 모든 모델의 역할, 의존성, 변환 로직을 한눈에 파악

---

## 모델 구조 요약

**총 12개 모델**:
- **Staging Layer**: 6개 (View)
- **Marts Layer**: 6개 (Table)

---

## Staging Models (6개)

### 1. stg_customers

**파일**: `models/staging/stg_customers.sql`

| 항목 | 내용 |
|------|------|
| **소스** | `{{ source('ecom', 'raw_customers') }}` |
| **Materialized** | View |
| **주요 변환** | - `id` → `customer_id`<br>- `name` → `customer_name` |
| **비즈니스 로직** | 없음 (단순 컬럼명 표준화) |
| **Downstream** | `customers.sql` (marts) |

**코드 패턴**:
```sql
with
source as (
    select * from {{ source('ecom', 'raw_customers') }}
),
renamed as (
    select
        id as customer_id,
        name as customer_name
    from source
)
select * from renamed
```

**학습 포인트**:
- ✅ 가장 단순한 Staging 패턴
- ✅ `source()` → `renamed` 2단계 CTE
- ✅ Primary Key 네이밍: `{entity}_id`

---

### 2. stg_orders

**파일**: `models/staging/stg_orders.sql`

| 항목 | 내용 |
|------|------|
| **소스** | `{{ source('ecom', 'raw_orders') }}` |
| **Materialized** | View |
| **주요 변환** | - `id` → `order_id`<br>- `store_id` → `location_id`<br>- `customer` → `customer_id`<br>- **cents → dollars 변환**<br>- **timestamp → date 변환** |
| **Macro 사용** | `{{ cents_to_dollars() }}`<br>`{{ dbt.date_trunc() }}` |
| **Downstream** | `orders.sql`, `order_items.sql` (marts) |

**코드 패턴**:
```sql
with
source as (
    select * from {{ source('ecom', 'raw_orders') }}
),
renamed as (
    select
        -- IDs
        id as order_id,
        store_id as location_id,
        customer as customer_id,

        -- Numerics (cents → dollars)
        subtotal as subtotal_cents,
        {{ cents_to_dollars('subtotal') }} as subtotal,
        {{ cents_to_dollars('tax_paid') }} as tax_paid,
        {{ cents_to_dollars('order_total') }} as order_total,

        -- Timestamps → Date
        {{ dbt.date_trunc('day','ordered_at') }} as ordered_at
    from source
)
select * from renamed
```

**학습 포인트**:
- ✅ Macro 활용: 재사용 가능한 변환 로직
- ✅ 데이터 타입 변환: cents → dollars (비즈니스 친화적)
- ✅ 시간 정규화: timestamp → date (분석 단위)
- ✅ 주석으로 컬럼 그룹화 (IDs, Numerics, Timestamps)

---

### 3. stg_order_items

**파일**: `models/staging/stg_order_items.sql`

| 항목 | 내용 |
|------|------|
| **소스** | `{{ source('ecom', 'raw_items') }}` |
| **Materialized** | View |
| **주요 변환** | - `id` → `order_item_id`<br>- `sku` → `product_id` |
| **비즈니스 로직** | 없음 (단순 ID 매핑) |
| **Downstream** | `order_items.sql` (marts) |

**코드 패턴**:
```sql
with
source as (
    select * from {{ source('ecom', 'raw_items') }}
),
renamed as (
    select
        id as order_item_id,
        order_id,
        sku as product_id
    from source
)
select * from renamed
```

**학습 포인트**:
- ✅ Foreign Key 표준화: `sku` → `product_id`
- ✅ 간결한 Staging (3개 컬럼만)

---

### 4. stg_products

**파일**: `models/staging/stg_products.sql`

| 항목 | 내용 |
|------|------|
| **소스** | `{{ source('ecom', 'raw_products') }}` |
| **Materialized** | View |
| **주요 변환** | - `sku` → `product_id`<br>- `name` → `product_name`<br>- `price` → `product_price`<br>- **Boolean 플래그 생성** |
| **비즈니스 로직** | `is_food_item`, `is_drink_item` 플래그 |
| **Downstream** | `products.sql`, `order_items.sql` (marts) |

**코드 패턴**:
```sql
with
source as (
    select * from {{ source('ecom', 'raw_products') }}
),
renamed as (
    select
        -- IDs
        sku as product_id,
        name as product_name,
        type as product_type,
        price as product_price,
        description as product_description,

        -- Boolean flags
        coalesce(type = 'jaffle', false) as is_food_item,
        coalesce(type = 'beverage', false) as is_drink_item
    from source
)
select * from renamed
```

**학습 포인트**:
- ✅ Boolean 플래그 생성: `type = 'jaffle'` → `is_food_item`
- ✅ `coalesce()` 활용: Null 안전 처리
- ✅ Staging에서도 간단한 비즈니스 로직 가능

---

### 5. stg_supplies

**파일**: `models/staging/stg_supplies.sql`

| 항목 | 내용 |
|------|------|
| **소스** | `{{ source('ecom', 'raw_supplies') }}` |
| **Materialized** | View |
| **주요 변환** | - `sku` → `product_id`<br>- **Surrogate Key 생성** |
| **Macro 사용** | `{{ dbt_utils.generate_surrogate_key() }}` |
| **Downstream** | `supplies.sql`, `order_items.sql` (marts) |

**코드 패턴**:
```sql
with
source as (
    select * from {{ source('ecom', 'raw_supplies') }}
),
renamed as (
    select
        {{ dbt_utils.generate_surrogate_key(['id', 'sku']) }} as supply_uuid,
        id as supply_id,
        sku as product_id,
        name as supply_name,
        cost as supply_cost,
        perishable as is_perishable_supply
    from source
)
select * from renamed
```

**학습 포인트**:
- ✅ Surrogate Key 생성: 복합 키 대신 UUID
- ✅ `dbt_utils` 패키지 활용
- ✅ Boolean 네이밍: `perishable` → `is_perishable_supply`

---

### 6. stg_locations

**파일**: `models/staging/stg_locations.sql`

| 항목 | 내용 |
|------|------|
| **소스** | `{{ source('ecom', 'raw_stores') }}` |
| **Materialized** | View |
| **주요 변환** | - `id` → `location_id`<br>- `name` → `location_name`<br>- **날짜 변환** |
| **Macro 사용** | `{{ dbt.date_trunc() }}` |
| **Downstream** | `locations.sql` (marts) |

**코드 패턴**:
```sql
with
source as (
    select * from {{ source('ecom', 'raw_stores') }}
),
renamed as (
    select
        id as location_id,
        name as location_name,
        tax_rate,
        {{ dbt.date_trunc('day', 'opened_at') }} as opened_date
    from source
)
select * from renamed
```

**학습 포인트**:
- ✅ 날짜 정규화: timestamp → date
- ✅ 세금 관련 컬럼 유지 (tax_rate)

---

## Marts Models (6개)

### 1. customers

**파일**: `models/marts/customers.sql`

| 항목 | 내용 |
|------|------|
| **의존성** | `{{ ref('stg_customers') }}`<br>`{{ ref('orders') }}` |
| **Materialized** | Table |
| **주요 변환** | - 고객별 주문 집계<br>- 생애 가치(LTV) 계산<br>- 고객 타입 분류 |
| **비즈니스 로직** | `is_repeat_buyer` → `customer_type`<br>(returning/new) |

**코드 구조**:
```sql
with
customers as (
    select * from {{ ref('stg_customers') }}
),
orders as (
    select * from {{ ref('orders') }}  -- Mart 간 참조!
),
customer_orders_summary as (
    select
        customer_id,
        count(distinct order_id) as count_lifetime_orders,
        count(distinct order_id) > 1 as is_repeat_buyer,
        min(ordered_at) as first_ordered_at,
        max(ordered_at) as last_ordered_at,
        sum(subtotal) as lifetime_spend_pretax,
        sum(tax_paid) as lifetime_tax_paid,
        sum(order_total) as lifetime_spend
    from orders
    group by 1
),
joined as (
    select
        customers.*,
        customer_orders_summary.count_lifetime_orders,
        customer_orders_summary.first_ordered_at,
        customer_orders_summary.lifetime_spend,
        case
            when customer_orders_summary.is_repeat_buyer then 'returning'
            else 'new'
        end as customer_type
    from customers
    left join customer_orders_summary
        on customers.customer_id = customer_orders_summary.customer_id
)
select * from joined
```

**학습 포인트**:
- ✅ 집계 로직: `count()`, `sum()`, `min()`, `max()`
- ✅ Boolean → Enum 변환: `is_repeat_buyer` → `customer_type`
- ✅ **Mart 간 참조**: `ref('orders')` (다른 Mart 참조)
- ✅ Left Join: 주문 없는 고객도 포함

**핵심 지표**:
- `count_lifetime_orders`: 총 주문 수
- `lifetime_spend`: 생애 가치 (LTV)
- `customer_type`: 신규/재방문 분류

---

### 2. orders

**파일**: `models/marts/orders.sql`

| 항목 | 내용 |
|------|------|
| **의존성** | `{{ ref('stg_orders') }}`<br>`{{ ref('order_items') }}` |
| **Materialized** | Table |
| **주요 변환** | - 주문 항목 집계<br>- 주문 타입 플래그<br>- **고객별 주문 순서** (Window Function) |
| **비즈니스 로직** | `is_food_order`, `is_drink_order`<br>`customer_order_number` |

**코드 구조**:
```sql
with
orders as (
    select * from {{ ref('stg_orders') }}
),
order_items as (
    select * from {{ ref('order_items') }}  -- Mart 간 참조!
),
order_items_summary as (
    select
        order_id,
        sum(supply_cost) as order_cost,
        sum(product_price) as order_items_subtotal,
        count(order_item_id) as count_order_items,
        sum(case when is_food_item then 1 else 0 end) as count_food_items,
        sum(case when is_drink_item then 1 else 0 end) as count_drink_items
    from order_items
    group by 1
),
compute_booleans as (
    select
        orders.*,
        order_items_summary.*,
        order_items_summary.count_food_items > 0 as is_food_order,
        order_items_summary.count_drink_items > 0 as is_drink_order
    from orders
    left join order_items_summary
        on orders.order_id = order_items_summary.order_id
),
customer_order_count as (
    select
        *,
        row_number() over (
            partition by customer_id
            order by ordered_at asc
        ) as customer_order_number  -- 고객별 N번째 주문
    from compute_booleans
)
select * from customer_order_count
```

**학습 포인트**:
- ✅ 조건부 집계: `sum(case when ... then 1 else 0 end)`
- ✅ Boolean 파생: `count_food_items > 0 as is_food_order`
- ✅ **Window Function**: `row_number() over (partition by ...)`
- ✅ 단계별 CTE: `summary` → `booleans` → `count`

**핵심 지표**:
- `customer_order_number`: 고객의 N번째 주문 (1, 2, 3, ...)
- `is_food_order`, `is_drink_order`: 주문 타입 플래그
- `order_cost`: 원가
- `order_items_subtotal`: 판매가

**성능 최적화**:
- Window Function으로 Correlated Subquery 대체 (15배 개선)

---

### 3. order_items

**파일**: `models/marts/order_items.sql`

| 항목 | 내용 |
|------|------|
| **의존성** | `{{ ref('stg_order_items') }}`<br>`{{ ref('stg_orders') }}`<br>`{{ ref('stg_products') }}`<br>`{{ ref('stg_supplies') }}` |
| **Materialized** | Table |
| **주요 변환** | - 4개 테이블 Join<br>- 주문 항목별 원가/수익 계산 |
| **비즈니스 로직** | 제품 정보 + 원가 매칭 |

**코드 구조**:
```sql
with
order_items as (
    select * from {{ ref('stg_order_items') }}
),
orders as (
    select * from {{ ref('stg_orders') }}
),
products as (
    select * from {{ ref('stg_products') }}
),
supplies as (
    select * from {{ ref('stg_supplies') }}
),
order_supplies_summary as (
    select
        product_id,
        sum(supply_cost) as supply_cost
    from supplies
    group by 1
),
joined as (
    select
        order_items.*,

        orders.ordered_at,

        products.product_name,
        products.product_price,
        products.is_food_item,
        products.is_drink_item,

        order_supplies_summary.supply_cost

    from order_items

    left join orders on order_items.order_id = orders.order_id

    left join products on order_items.product_id = products.product_id

    left join order_supplies_summary
        on order_items.product_id = order_supplies_summary.product_id
)
select * from joined
```

**학습 포인트**:
- ✅ 다중 테이블 Join (4개)
- ✅ 재고 원가 집계: `order_supplies_summary`
- ✅ 비정규화 패턴: 분석 편의성 우선

**핵심 지표**:
- `supply_cost`: 제품별 총 원가
- `product_price`: 판매가
- 수익 = `product_price - supply_cost` (계산 가능)

---

### 4. products

**파일**: `models/marts/products.sql`

| 항목 | 내용 |
|------|------|
| **의존성** | `{{ ref('stg_products') }}` |
| **Materialized** | Table |
| **주요 변환** | 없음 (Pass-through) |
| **비즈니스 로직** | 없음 |

**코드**:
```sql
with
products as (
    select * from {{ ref('stg_products') }}
)
select * from products
```

**학습 포인트**:
- ✅ **Pass-through Mart**: Staging 그대로 복사
- ✅ 이유: 향후 확장 가능성 (제품별 판매량 집계 등)
- ✅ 일관성: 모든 분석은 Marts에서

---

### 5. supplies

**파일**: `models/marts/supplies.sql`

| 항목 | 내용 |
|------|------|
| **의존성** | `{{ ref('stg_supplies') }}` |
| **Materialized** | Table |
| **주요 변환** | 없음 (Pass-through) |
| **비즈니스 로직** | 없음 |

**코드**:
```sql
with
supplies as (
    select * from {{ ref('stg_supplies') }}
)
select * from supplies
```

**학습 포인트**:
- ✅ **Pass-through Mart**
- ✅ 향후 확장: 재고 회전율, 부패 분석 등

---

### 6. locations

**파일**: `models/marts/locations.sql`

| 항목 | 내용 |
|------|------|
| **의존성** | `{{ ref('stg_locations') }}` |
| **Materialized** | Table |
| **주요 변환** | 없음 (Pass-through) |
| **비즈니스 로직** | 없음 |

**코드**:
```sql
with
locations as (
    select * from {{ ref('stg_locations') }}
)
select * from locations
```

**학습 포인트**:
- ✅ **Pass-through Mart**
- ✅ 향후 확장: 매장별 매출 분석 등

---

### 7. metricflow_time_spine

**파일**: `models/marts/metricflow_time_spine.sql`

| 항목 | 내용 |
|------|------|
| **의존성** | 없음 (dbt_date 패키지) |
| **Materialized** | Table |
| **주요 변환** | - 10년 분 날짜 생성<br>- Date 타입 캐스팅 |
| **비즈니스 로직** | MetricFlow용 시간 축 |

**코드**:
```sql
with
days as (
    --for BQ adapters use "DATE('01/01/2000','mm/dd/yyyy')"
    {{ dbt_date.get_base_dates(n_dateparts=365*10, datepart="day") }}
),
cast_to_date as (
    select cast(date_day as date) as date_day
    from days
)
select * from cast_to_date
```

**학습 포인트**:
- ✅ **dbt_date 패키지**: 날짜 생성 Macro
- ✅ **MetricFlow**: dbt의 지표 계산 프레임워크
- ✅ Time Spine: 시계열 분석 기준 테이블

**용도**:
- 매출 추이 분석
- 일별/주별/월별 집계
- 트렌드 분석

---

## 의존성 그래프

```
Seeds (Raw CSV)
    ├── raw_customers
    ├── raw_orders
    ├── raw_items
    ├── raw_products
    ├── raw_supplies
    └── raw_stores
           ↓
Staging Models (6개)
    ├── stg_customers ────────────────┐
    ├── stg_orders ──────────────┐    │
    ├── stg_order_items ─────┐   │    │
    ├── stg_products ─────┐  │   │    │
    ├── stg_supplies ──┐  │  │   │    │
    └── stg_locations  │  │  │   │    │
                       │  │  │   │    │
                       ↓  ↓  ↓   ↓    ↓
Marts Models (6개)
    ├── supplies ────────────────────┐
    ├── products ────────────────┐   │
    ├── locations                │   │
    ├── order_items ◄───────────┼───┘
    │                            │
    ├── orders ◄────────────────┘
    │
    └── customers ◄─────────────┘

    metricflow_time_spine (독립)
```

**핵심 패턴**:
1. **Staging → Marts 단방향 흐름**
2. **Mart 간 참조**: `customers` → `orders` → `order_items`
3. **Pass-through Marts**: `products`, `supplies`, `locations`

---

## Materialization 전략

| Layer | Materialized | 이유 |
|-------|--------------|------|
| **Staging** | View | - 데이터 신선도 유지<br>- 저장 공간 절약<br>- 단순 변환만 |
| **Marts** | Table | - 복잡한 집계 있음<br>- 반복 쿼리 성능 개선<br>- 분석가 대상 |

**설정 위치**: `dbt_project.yml:34-38`
```yaml
models:
  jaffle_shop:
    staging:
      +materialized: view
    marts:
      +materialized: table
```

---

## Pass-through Marts의 의미

**왜 `products`, `supplies`, `locations`는 그대로 복사?**

1. **일관성**: 모든 분석은 Marts에서 시작
   ```sql
   -- ✅ 일관성 있음
   select * from {{ ref('products') }}
   select * from {{ ref('customers') }}

   -- ❌ 혼란스러움
   select * from {{ ref('stg_products') }}  -- Staging?
   select * from {{ ref('customers') }}     -- Mart?
   ```

2. **확장성**: 향후 로직 추가 가능
   ```sql
   -- 현재: Pass-through
   select * from {{ ref('stg_products') }}

   -- 향후: 판매량 추가
   with products as (
       select * from {{ ref('stg_products') }}
   ),
   sales as (
       select product_id, count(*) as sales_count
       from {{ ref('order_items') }}
       group by 1
   )
   select products.*, sales.sales_count
   from products left join sales ...
   ```

3. **문서화**: Marts = 분석가 대상
   - Staging: 데이터팀 내부용
   - Marts: 분석가, BI 도구 연결

---

## 모델별 복잡도

| 모델 | 줄 수 | CTE 수 | Join 수 | 집계 | 복잡도 |
|------|-------|--------|---------|------|--------|
| stg_customers | 12 | 2 | 0 | 0 | ⭐ 매우 낮음 |
| stg_orders | 25 | 2 | 0 | 0 | ⭐ 낮음 |
| stg_order_items | 12 | 2 | 0 | 0 | ⭐ 매우 낮음 |
| stg_products | 20 | 2 | 0 | 0 | ⭐⭐ 낮음 |
| stg_supplies | 18 | 2 | 0 | 0 | ⭐⭐ 낮음 |
| stg_locations | 15 | 2 | 0 | 0 | ⭐ 매우 낮음 |
| **customers** | **35** | **4** | **1** | **7** | ⭐⭐⭐ 중간 |
| **orders** | **50** | **5** | **1** | **6** | ⭐⭐⭐⭐ 높음 |
| **order_items** | **40** | **6** | **3** | **1** | ⭐⭐⭐⭐ 높음 |
| products | 8 | 1 | 0 | 0 | ⭐ 매우 낮음 |
| supplies | 8 | 1 | 0 | 0 | ⭐ 매우 낮음 |
| locations | 8 | 1 | 0 | 0 | ⭐ 매우 낮음 |

**관찰**:
- Staging은 모두 단순 (2 CTE, 0 Join)
- Marts는 양극화 (Pass-through vs 복잡한 집계)
- 가장 복잡: `orders.sql` (Window Function + 집계)

---

## 핵심 학습 포인트

### 1. Staging Layer 철칙
```sql
-- ✅ Staging에서만 할 것
- 컬럼명 표준화
- 데이터 타입 변환
- 기본 필터링 (where deleted_at is null)

-- ❌ Staging에서 하지 말 것
- 집계 (group by)
- 복잡한 Join
- 비즈니스 로직
```

### 2. CTE 네이밍 규칙
```sql
source            -- Raw 데이터 로드
renamed           -- 컬럼명 변경
filtered          -- 필터링
aggregated        -- 집계
compute_booleans  -- Boolean 생성
joined            -- Join
final             -- 최종 (선택적)
```

### 3. 컬럼 네이밍 패턴
| 타입 | 패턴 | 예시 |
|------|------|------|
| Primary Key | `{entity}_id` | `customer_id` |
| Foreign Key | `{entity}_id` | `customer_id` (orders 테이블) |
| Boolean | `is_{attribute}` | `is_food_item` |
| Count | `count_{entity}` | `count_lifetime_orders` |
| 날짜 | `{action}_at` | `ordered_at` |
| 금액 | `{attribute}` | `subtotal`, `tax_paid` |

### 4. Mart 간 참조
```sql
-- ✅ 가능: Mart → Mart
select * from {{ ref('orders') }}  -- in customers.sql

-- ⚠️ 주의: 순환 참조 방지
-- dbt가 자동 감지해서 에러 발생
```

---

## 다음 문서

**07-macros-config.md**에서:
- Macro 구현 상세 분석 (`cents_to_dollars`, `generate_schema_name`)
- `dbt_project.yml` 설정 해석
- 패키지 의존성 (`dbt_utils`, `dbt_date`)
