## Marts: Business-defined entities
----
### 개요
이 계층은 우리가 지금까지 준비해 온 원자(staging 모델) 및 분자(intermediate 모델) 들을 하나로 결합해, 정체성과 목적을 가진 엔터티(Entity 또는 Concept) 단위의 테이블을 만드는 곳입니다.  
예: 주문(order), 고객(customer), 결제(payment), 클릭 이벤트(click event) 등 각 개념이 고유한 mart로 표현됩니다.  
각 행(row)은 해당 개념의 단일 인스턴스를 나타냅니다.  
전통적인 Kimball의 스타 스키마 방식과는 달리, 현대의 데이터 웨어하우징 방식에서는 저장(Storage)은 저렴하고 연산(Compute)이 비싸므로, 여러 관련 데이터를 denormalize(중복 포함)해서 한 테이블에 넣는 방식이 오히려 효율적일 수 있습니다.  
예컨대, customers mart 안에 orders 관련 컬럼을 가져다 쓰는 것이, 매번 조인하는 것보다 효율적일 수 있습니다.  
----
### 파일 및 폴더 구조
- Marts: File and folders
```
models/marts
├── finance
│   ├── _finance__models.yml
│   ├── orders.sql
│   └── payments.sql
└── marketing
    ├── _marketing__models.yml
    └── customers.sql
```
- 폴더 구성 원칙
    - 부서 또는 관심 분야(business domain) 기준으로 그룹화 (예: finance, marketing)
    - mart 수가 많지 않다면 하위 폴더를 과도하게 나누지 않아도 됨
    - 이 계층에서는 더 이상 원본(source)-conformed 데이터를 유지할 필요 없으므로, 비즈니스 개념 중심의 그룹화가 자연스러움
- 파일 명명규칙
    - 파일명은 핵심 엔터티 단어를 사용 (예: customers, orders)
    - 시간 기반 롤업을 포함하지 않은 순수 mart는 orders_per_day 같은 이름을 붙이지 않는 것이 일반적
    - 동일한 개념을 부서별로 따로 만드는 것은 보통 반패턴(anti-pattern)으로 간주됨 예: finance_orders vs marketing_orders — 가능하다면 단일 개념 orders를 공유하는 것이 좋음
    - 단, 부서별로 측정 방식이 완전히 다를 경우엔 예외가 있을 수 있으며, 그 경우엔 tax_revenue vs revenue 같은 명확한 구분을 두는 것이 더 낫다는 설명이 있습니다.  
  
----
### Marts: Models
- orders.sql
```
with
  orders as (
    select * from {{ ref('stg_jaffle_shop__orders') }}
  ),

  order_payments as (
    select * from {{ ref('int_payments_pivoted_to_orders') }}
  ),

  orders_and_order_payments_joined as (
    select
      orders.order_id,
      orders.customer_id,
      orders.order_date,
      coalesce(order_payments.total_amount, 0) as amount,
      coalesce(order_payments.gift_card_amount, 0) as gift_card_amount
    from orders
    left join order_payments
      on orders.order_id = order_payments.order_id
  )

select * from orders_and_order_payments_joined
```
- customers.sql
```
with
  customers as (
    select * from {{ ref('stg_jaffle_shop__customers') }}
  ),

  orders as (
    select * from {{ ref('orders') }}
  ),

  customer_orders as (
    select
      customer_id,
      min(order_date) as first_order_date,
      max(order_date) as most_recent_order_date,
      count(order_id) as number_of_orders,
      sum(amount) as lifetime_value
    from orders
    group by 1
  ),

  customers_and_customer_orders_joined as (
    select
      customers.customer_id,
      customers.first_name,
      customers.last_name,
      customer_orders.first_order_date,
      customer_orders.most_recent_order_date,
      coalesce(customer_orders.number_of_orders, 0) as number_of_orders,
      customer_orders.lifetime_value
    from customers
    left join customer_orders
      on customers.customer_id = customer_orders.customer_id
  )

select * from customers_and_customer_orders_joined
```
- Materialization (실제 테이블 생성 방식)
    - 이 계층에서는 테이블(table) 또는 점진적 증가 모델(incremental model) 로 materialize 하는 것이 권장됨
    - 일반적인 전략:
    1. 처음엔 뷰(view) 로 시작 (저장 공간 거의 안 쓰이면서 최신 값을 항상 반영)
    2. 뷰가 너무 느려지면 테이블로 변환
    3. 테이블마저 빌드 비용이 커지면 incremental model 로 전환
        - 너무 많은 mart 모델을 처음부터 incremental 로 만드는 것은 피해야 함
        - Granularity (그레인) 은 엔터티 기준으로 유지 — 예: orders mart는 여전히 order 단위 그레인
        - denormalization: 여러 개념의 컬럼을 끌어와 한 테이블을 넓게 설계하는 것이 권장됨
        - 한 mart 내에 너무 많은 조인 또는 개념 결합은 피해야 함 예: 여러 staging / intermediate 모델을 복잡하게 조합하면 가독성과 유지보수에 부담
        - 보통 4~5 개 개념 이상을 한 mart 안에서 합치는 건 복잡성 초과 신호 이럴 땐 중간 intermediate 모델을 더 추가해 구조를 계층화하는 것이 낫다

- Mart 간 상호 참조
    - 때때로 한 mart (예: orders) 를 다른 mart (예: customers) 에서 참조하는 경우가 있음
    - 하지만 이런 참조는 신중하게 설계해야 하며, 순환 참조(circular dependency)가 생기지 않도록 해야 함
- 엔터티 기반 그레인 유지
    - mart는 특정 개념의 단일 인스턴스를 행 단위로 담아야 함
    - 예: orders mart 내에서 users 와 orders 를 날짜 단위로 묶어 user_orders_per_day 같은 것은 metrics 계층에 해당함  
----
### 기타 고려사항
- 디버깅을 위한 테이블 활용
    - 실전에서는 view / ephemeral 모델 위주로 쌓다가, 오류나 버그 추적이 어려울 경우 중간 모델들을 임시로 테이블로 materialize 해 보면, 오류가 어디서 일어나는지 더 분명하게 알 수 있음
- Semantic Layer 사용 시 설계 차이
    - 만약 dbt의 Semantic Layer 를 사용하는 경우, marts 설계에서 가능한 정규화(normalization) 방향을 선호
    - 반면 Semantic Layer 없이 사용한다면, 위에서 설명한 대로 denormalized 설계가 일반적
    - Semantic Layer와 marts 관련 구조/명명 규칙 변화는 “How we build our metrics” 문서의 “Refactoring an existing rollup” 부분 등에서 자세히 다룸  
----
### 요약
- Mart 계층은 비즈니스 개념(entity)을 중심으로, staging / intermediate 레벨의 조각들을 결합해 최종적으로 사용자가 접근할 수 있는 테이블을 만드는 단계
- 폴더 구조는 비즈니스 도메인(예: finance, marketing) 중심 그룹화
- 파일명은 엔터티 중심 (예: orders.sql, customers.sql)
- 모델 구조: 조인과 결합을 활용해 여러 개념의 정보를 담지만, 핵심 개념의 그레인은 유지
- Materialization 전략: view → table → incremental 순으로 점진적으로 복잡도를 증가
- 권장 패턴
    - 넓고 denormalized 한 테이블 설계
    - 너무 많은 조인/개념 결합은 피하고, 필요 시 intermediate 모델로 나누기
    - mart 간 참조는 가능하지만 순환 참조 주의
- 디버깅 고려: 오류 추적이 어렵다면 임시 테이블로 materialize
- *Semantic Layer 유무에 따라 설계 방식이 달라질 수 있음 — normalization vs denormalization 선택
  
*semantic Layer : 메타데이터의 한 종류, 데이터를 어떻게 해석할 것 인지이데 핸 설명 ex) 매출 = orders.amount의 합계, 주문수 = order_id 개수
### 참고
- https://docs.getdbt.com/best-practices/how-we-structure/4-marts