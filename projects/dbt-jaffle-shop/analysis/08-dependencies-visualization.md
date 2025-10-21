# dbt jaffle-shop 의존성 그래프 분석

> **목표**: 모델 간 의존성을 시각화하고 실행 순서 파악

---

## 전체 의존성 그래프 (DAG)

```
┌─────────────────────────────────────────────────────────────┐
│                      Seeds (Raw Data)                        │
│  raw_customers  raw_orders  raw_items  raw_products          │
│  raw_supplies   raw_stores                                   │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                    Staging Layer (Views)                     │
│                                                              │
│  ┌─────────────┐  ┌──────────┐  ┌──────────────┐           │
│  │stg_customers│  │stg_orders│  │stg_order_items│          │
│  └─────────────┘  └──────────┘  └──────────────┘           │
│                                                              │
│  ┌───────────┐  ┌────────────┐  ┌──────────────┐           │
│  │stg_products│ │stg_supplies│  │stg_locations │           │
│  └───────────┘  └────────────┘  └──────────────┘           │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                     Marts Layer (Tables)                     │
│                                                              │
│            ┌──────────────┐                                 │
│            │ order_items  │◄─────┐                          │
│            └──────────────┘      │                          │
│                   ↓               │                          │
│            ┌──────────────┐      │                          │
│            │   orders     │◄─────┼─────┐                    │
│            └──────────────┘      │     │                    │
│                   ↓               │     │                    │
│            ┌──────────────┐      │     │                    │
│            │  customers   │      │     │                    │
│            └──────────────┘      │     │                    │
│                                  │     │                    │
│  ┌─────────┐  ┌─────────┐  ┌────────┐│                     │
│  │products │  │supplies │  │locations││                     │
│  └─────────┘  └─────────┘  └─────────┘│                     │
│                                        │                     │
│            ┌─────────────────────────┐ │                     │
│            │metricflow_time_spine    │ │                     │
│            │   (독립 - 패키지 사용)   │ │                     │
│            └─────────────────────────┘ │                     │
└────────────────────────────────────────┼─────────────────────┘
                                         │
                           (모두 Staging Layer에 의존)
```

---

## 의존성 상세 분석

### Staging Layer → Marts Layer

| Marts 모델 | 의존하는 Staging 모델 |
|------------|----------------------|
| **customers** | `stg_customers`, (`orders` ← Mart 참조!) |
| **orders** | `stg_orders`, (`order_items` ← Mart 참조!) |
| **order_items** | `stg_order_items`, `stg_orders`, `stg_products`, `stg_supplies` |
| **products** | `stg_products` |
| **supplies** | `stg_supplies` |
| **locations** | `stg_locations` |
| **metricflow_time_spine** | 없음 (dbt_date 패키지) |

---

### Mart 간 의존성 체인

```
order_items (기본)
    ↓ (참조)
orders (order_items 사용)
    ↓ (참조)
customers (orders 사용)
```

**코드 근거**:

**order_items.sql**:
```sql
-- Staging만 참조
select * from {{ ref('stg_order_items') }}
select * from {{ ref('stg_orders') }}
select * from {{ ref('stg_products') }}
select * from {{ ref('stg_supplies') }}
```

**orders.sql**:
```sql
-- ✅ Mart 참조!
select * from {{ ref('order_items') }}
```

**customers.sql**:
```sql
-- ✅ Mart 참조!
select * from {{ ref('orders') }}
```

---

## 실행 순서 (dbt run)

### Phase 1: Staging Layer (병렬 실행 가능)

```bash
[Run 1] 병렬 6개
├─ stg_customers
├─ stg_orders
├─ stg_order_items
├─ stg_products
├─ stg_supplies
└─ stg_locations
```

**이유**: 모두 Seeds에만 의존 → 상호 독립

**실행 시간** (예상):
- 각 모델: 0.1초
- 병렬 실행: 0.1초 (6개 동시)

---

### Phase 2: Simple Marts (병렬 실행 가능)

```bash
[Run 2] 병렬 4개
├─ products          (stg_products만 의존)
├─ supplies          (stg_supplies만 의존)
├─ locations         (stg_locations만 의존)
└─ metricflow_time_spine (독립)
```

**이유**: Staging만 의존, 서로 독립

**실행 시간** (예상):
- 각 모델: 0.2초
- 병렬 실행: 0.2초

---

### Phase 3: order_items

```bash
[Run 3] 단일
└─ order_items       (4개 Staging 의존)
```

**이유**:
- `stg_order_items`, `stg_orders`, `stg_products`, `stg_supplies` 모두 필요
- Phase 1 완료 후 실행 가능

**실행 시간** (예상): 0.5초 (Join 3개)

---

### Phase 4: orders

```bash
[Run 4] 단일
└─ orders            (order_items 의존)
```

**이유**:
- `order_items` Mart 참조
- Phase 3 완료 후 실행

**실행 시간** (예상): 1.0초 (Window Function)

---

### Phase 5: customers

```bash
[Run 5] 단일
└─ customers         (orders 의존)
```

**이유**:
- `orders` Mart 참조
- Phase 4 완료 후 실행

**실행 시간** (예상): 0.8초 (집계)

---

### 전체 실행 시간

```
Phase 1: 0.1초 (Staging 6개 병렬)
Phase 2: 0.2초 (Simple Marts 4개 병렬)
Phase 3: 0.5초 (order_items)
Phase 4: 1.0초 (orders)
Phase 5: 0.8초 (customers)
─────────────────────────
총:      2.6초
```

**만약 순차 실행이었다면?**
```
Staging: 6 × 0.1 = 0.6초
Marts:   7 × 0.5 = 3.5초
─────────────────────────
총:      4.1초

병렬 실행 개선율: 37% 단축
```

---

## dbt 명령어별 실행

### 1. 전체 실행
```bash
dbt run
```

**실행 순서**:
1. Staging (6개 병렬)
2. Simple Marts (4개 병렬)
3. order_items
4. orders
5. customers

---

### 2. 특정 모델만
```bash
dbt run --select customers
```

**실행**:
- `customers.sql`만 실행
- **주의**: 의존성 테이블이 없으면 에러!

---

### 3. 의존성 포함 실행
```bash
dbt run --select +customers
```

**실행 순서** (`+` = upstream):
1. Staging (6개)
2. order_items
3. orders
4. customers

**의미**: `customers`와 그것이 의존하는 모든 것

---

### 4. Downstream 포함
```bash
dbt run --select order_items+
```

**실행 순서** (`+` = downstream):
1. order_items
2. orders
3. customers

**의미**: `order_items`와 그것을 쓰는 모든 것

---

### 5. 전체 체인
```bash
dbt run --select +customers+
```

**실행**: 전체 DAG (customers 중심)

---

### 6. 특정 디렉토리
```bash
dbt run --select staging.*
dbt run --select marts.*
```

**실행**:
- `staging.*`: Staging Layer만
- `marts.*`: Marts Layer만

---

## 의존성 시각화 (dbt docs)

### 생성
```bash
dbt docs generate
dbt docs serve
```

**결과**: `http://localhost:8080`에서 대화형 DAG 확인

---

### Lineage Graph (예상)

**customers 중심 그래프**:
```
stg_customers ───────────┐
                         ├──► customers
stg_orders ──► order_items ──► orders ─┘
             ↑         ↑
stg_products ┘         │
stg_supplies ──────────┘
```

**orders 중심 그래프**:
```
stg_order_items ─────┐
                     ├──► order_items ──► orders
stg_orders ──────────┤
                     │
stg_products ────────┤
                     │
stg_supplies ────────┘
```

---

## 순환 참조 감지

### dbt가 자동 감지하는 경우

**잘못된 예**:
```sql
-- customers.sql
select * from {{ ref('orders') }}

-- orders.sql
select * from {{ ref('customers') }}  -- ❌ 순환!
```

**에러 메시지**:
```
Compilation Error
  Cycle detected in model dependencies:
  customers → orders → customers
```

**해결책**:
1. Intermediate Layer 추가
2. 로직 재설계

---

## 병렬 실행 최적화

### 현재 구조의 병렬성

| Phase | 모델 수 | 병렬 가능 | 이유 |
|-------|--------|----------|------|
| Phase 1 | 6 | ✅ 6개 | Staging - 상호 독립 |
| Phase 2 | 4 | ✅ 4개 | Simple Marts - 독립 |
| Phase 3 | 1 | ❌ 1개 | order_items - 체인 시작 |
| Phase 4 | 1 | ❌ 1개 | orders - 체인 중간 |
| Phase 5 | 1 | ❌ 1개 | customers - 체인 끝 |

**병렬 효율**: 10개 중 10개가 최대 병렬 (Phase 1-2에서)

---

### 만약 Mart 간 참조가 없었다면?

**가상 시나리오**:
```sql
-- customers.sql (직접 Staging 참조)
select * from {{ ref('stg_customers') }}
select * from {{ ref('stg_orders') }}  -- orders 대신
```

**병렬성**:
```
Phase 1: Staging 6개 병렬
Phase 2: Marts 7개 병렬  ← 개선!
```

**실행 시간**:
```
Phase 1: 0.1초
Phase 2: 1.0초 (가장 느린 모델 기준)
─────────────────
총:      1.1초 (기존 2.6초 대비 58% 단축)
```

**Trade-off**:
- ✅ 성능: 58% 빠름
- ❌ 중복: `orders` 로직이 여러 곳에 복사됨
- ❌ 유지보수: 로직 변경 시 3곳 수정

---

## 실전 의존성 설계 패턴

### 패턴 1: 체인형 (jaffle_shop 방식)

```
stg → mart1 → mart2 → mart3
```

**장점**:
- ✅ 중복 제거 (DRY)
- ✅ 로직 재사용

**단점**:
- ❌ 순차 실행
- ❌ 느림

**적용 시점**: 로직 재사용이 중요할 때

---

### 패턴 2: 병렬형

```
stg1 → mart1
stg2 → mart2
stg3 → mart3
```

**장점**:
- ✅ 병렬 실행
- ✅ 빠름

**단점**:
- ❌ 중복 가능
- ❌ 유지보수 어려움

**적용 시점**: 성능이 중요할 때 (대용량)

---

### 패턴 3: Intermediate Layer (권장)

```
stg → intermediate → mart1
                  → mart2
                  → mart3
```

**장점**:
- ✅ 중복 제거
- ✅ 병렬 실행 (mart1-3)

**단점**:
- ❌ 계층 1개 추가

**적용 시점**: 프로젝트 규모 중간 이상

---

## 핵심 학습 포인트

### 1. DAG 읽기
```sql
-- ref() 찾기 → 의존성 파악
select * from {{ ref('stg_orders') }}     -- upstream
select * from {{ ref('order_items') }}    -- upstream (Mart!)
```

### 2. 실행 순서 예측
```
1. Staging (의존성 0)
2. Marts (Staging만 의존)
3. Marts (Mart 의존)
```

### 3. 병렬 실행 전략
```bash
# 빠른 개발 (특정 모델만)
dbt run --select customers

# 안전한 배포 (의존성 포함)
dbt run --select +customers

# 영향 범위 확인 (downstream)
dbt run --select order_items+
```

### 4. 의존성 설계 원칙
```
- Staging: 독립적으로 (병렬 최대화)
- Marts: 재사용 vs 병렬 Trade-off
- Intermediate: 복잡도 증가 시 도입
```

---

## 다음 문서

**README.md 업데이트**:
- 전체 분석 요약
- 문서 간 네비게이션
- Level 2 분석 완료 선언
