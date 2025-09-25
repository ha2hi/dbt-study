## Intermediate: Purpose-built transformation steps
----
### Intermediate: 파일과 폴더 구성
원자(atomic) 데이터가 준비되면, 우리는 그것들을 더 복합적이고 연결된 형태(분자처럼)로 엮어 나갑니다. Intermediate 계층이 이 역할을 담당합니다. 이곳에서는 여러 staging 모델을 결합하거나 가공하여, 이후 mart 계층으로 가는 중간 결과들을 만듭니다.  
- models/intermediate
```
- 폴더 구분 방식
    - staging 계층에서는 소스 시스템별 폴더 구분을 하였지만, intermediate 계층에서는 비즈니스 관점 중심으로 폴더를 나눕니다. 즉, “finance”, “marketing” 등 비즈니스 도메인 기반 하위 디렉터리로 분리합니다.
- 파일 이름 규칙
    - 파일 이름에는 변환 동작(동사)을 포함시키는 것이 권장됩니다. 예를 들어 int_payments_pivoted_to_orders.sql 처럼 “payments” 엔터티가 “orders” 단위로 pivot 변환되었다는 의미가 드러나게 이름 짓는 식입니다.
    - 이 계층에서는 소스 vs 엔터티 구분을 위해 이중 언더스코어(__)를 사용하는 것을 반드시 고집하지 않아도 되며, 엔터티 + 동작이 자연스럽게 연결된 이름을 사용하는 게 낫다고 설명하고 있습니다.
    - 너무 이른 최적화는 피해야 합니다. 작은 프로젝트나 모델 수가 적을 경우, subdirectory를 나누지 않아도 괜찮습니다. 그러나 staging 계층만큼은 소스 시스템별 구분을 처음부터 해 두는 것이 좋습니다.
----
### Intermediate: Models
- 예시 모델: int_payments_pivoted_to_orders.sql
    - 이 모델은 payments 데이터를 order 단위로 집계 및 pivot 처리하는 목적을 가집니다. 내부 CTE 이름도 pivot_and_aggregate_payments_to_order_grain 처럼 동작이 직관적으로 드러나도록 명명되어 있습니다
```
-- int_payments_pivoted_to_orders.sql

{%- set payment_methods = ['bank_transfer','credit_card','coupon','gift_card'] -%}

with

payments as (

   select * from {{ ref('stg_stripe__payments') }}

),

pivot_and_aggregate_payments_to_order_grain as (

   select
      order_id,
      {% for payment_method in payment_methods -%}

         sum(
            case
               when payment_method = '{{ payment_method }}' and
                    status = 'success'
               then amount
               else 0
            end
         ) as {{ payment_method }}_amount,

      {%- endfor %}
      sum(case when status = 'success' then amount end) as total_amount

   from payments

   group by 1

)

select * from pivot_and_aggregate_payments_to_order_grain
```
- Intermediate 모델의 특성 및 제약:
    - ❌ 일반적으로 최종 사용자에게 노출되지 않아야 함 즉, dashboards나 앱이 직접 참조하는 스키마(프로덕션 스키마)에는 포함시키지 않는 것이 바람직합니다. 
    - ✅ ephemeral로 materialize 하는 것이 기본 시작점으로 권장됨
        - 이렇게 하면 웨어하우스에 불필요한 테이블이 늘지 않고, downstream 모델에 포함(interpolate)되어 실행됩니다. 
        - 또는 뷰(view) 로 materialize 해서 별도의 custom schema 또는 별도 권한 영역에 두는 방식도 많이 사용됩니다. 이 방식은 디버깅이나 개발 편의성을 높여 줍니다. 
- Intermediate 모델이 자주 활용되는 패턴들
    1. 구조 단순화 (Structural simplification)
    - 여러 엔터티(staging 또는 다른 intermediate 모델)를 적절한 그룹으로 묶어 mart 계층에서의 복잡한 조인을 줄입니다.
    2. Re-graining
    - 예를 들어 orders 테이블을 아이템 단위로 전개(fan out)해야 할 경우, intermediate 계층에서 이를 수행한 뒤 mart에서는 간결하게 참조할 수 있게 만듭니다.
    3. 복잡한 로직 격리 (Isolation of complexity)
    - 특정 복잡한 변환 부분을 별도 intermediate 모델로 분리하면, 디버깅이나 테스트가 쉬워지고 downstream 모델이 더 읽기 쉽고 단순해집니다.
- DAG 흐름과 모델 입력/출력 설계
    - Intermediate 계층이 확장되며, DAG(의존성 그래프)는 화살표 모양처럼 점점 넓어지는 형태를 띠게 됩니다 (출력은 적고 입력은 많음). 여러 엔터티에서 입력이 오지만, 한 모델에서 여러 출력을 내는 것은 피하는 것이 좋습니다.
    - 즉, 여러 화살표가 들어오는 것은 괜찮지만 여러 화살표가 나가는 것은 경계해야 함이 권고됩니다.
----
### 요약
- Intermediate 계층은 staging (원자 단위) 이후, 여러 엔터티를 결합하고 목적 기반 변환을 수행하는 중간 단계
- 폴더 구분은 비즈니스 도메인 중심 (finance, marketing 등), 파일명에는 동사 중심 네이밍 (예: int_X_pivoted_to_Y)
- Intermediate 모델은 일반적으로 ephemeral 또는 뷰(view) 로 materialize 하며, 최종 사용자에게 직접 노출되지 않는 것이 이상적
- 자주 쓰이는 패턴: 구조 단순화, 그레인 변경, 복잡한 로직 격리
- 설계 원칙: 여러 입력은 허용하되 여러 출력은 피하기 (단일 책임 중심)
- DAG 흐름은 점점 넓어지도록 설계 (포스트-staging 단계에서 개념들을 모아가는 형태)

### 참고
- https://docs.getdbt.com/best-practices/how-we-structure/3-intermediate