## Staging: Preparing our atomic building blocks
### 개요
스테이징 계층은 우리가 여정을 시작하는 지점입니다.  
이곳이 프로젝트의 기반이 되며, 이후 더 복잡하고 유용한 모델을 구축하는 데 사용할 개별 구성 요소들을 프로젝트에 가져오는 곳입니다.  
  
이 가이드에서는 비유를 사용합니다: 원자(atoms), 분자(molecules), 복잡한 출력물(proteins 또는 세포) 와 같이 모듈화하여 생각하기.  
원시(source) 시스템의 데이터가 “에너지와 쿼크의 수프(soup)”라면, 스테이징 계층은 이 재료를 정제해 원자 단위로 만드는 단계입니다.  
----
### staging: File and folders
아래는 models/staging 디렉터리 구조의 예시입니다.  
```
models/staging
├── jaffle_shop
│   ├── _jaffle_shop__docs.md
│   ├── _jaffle_shop__models.yml
│   ├── _jaffle_shop__sources.yml
│   ├── base
│   │   ├── base_jaffle_shop__customers.sql
│   │   └── base_jaffle_shop__deleted_customers.sql
│   ├── stg_jaffle_shop__customers.sql
│   └── stg_jaffle_shop__orders.sql
└── stripe
    ├── _stripe__models.yml
    ├── _stripe__sources.yml
    └── stg_stripe__payments.sql
```  
** 폴더 구조 원칙 **
- 소스 시스템별 하위 디렉터리로 구분
    - 내부 트랜잭션 DB, Stripe API, 이벤트 수집 시스템 등 각 소스 시스템을 기준으로 나눕니다.
- 로더(loader)별 구분 지양
    - 예: Fivetran, Stitch 등 방식별로 나누는 방식은 확장성이 떨어짐
- 비즈니스 그룹별로 나누는 것도 지양
    - 예: ‘마케팅’, ‘재무’ 등으로 나누면 너무 일찍 분리되어 정의 충돌이 생길 수 있음
- 폴더 구조는 코드베이스를 탐색하고 이해하는 핵심 인터페이스이며, DAG (의존성 그래프)과 데이터 모델 흐름을 반영해야 함
- dbt의 선택자(selector) 문법과도 연관되어 폴더 구조가 기능적으로도 활용됨
    - 예: dbt build --select staging.stripe+처럼 특정 소스 관련 모델만 실행 가능  
** 파일 네이밍 원칙 **
- stg_[source]__[entity]s.sql 형식 권장
    - 예: stg_stripe__payments.sql(이중 언더스코어는 source 와 entity 구분을 명확히 하기 위함)
- 엔터티명은 복수형 사용
    - 예: orders, customers
- 파일 이름은 고유해야 하며, 모델 이름과 일치해야 함
- 단순히 stg_[entity].sql처럼 쓰는 방식은 프로젝트가 커질수록 충돌 위험 있음
----
### Stagging: Models
스테이징 모델의 일반 패턴 예시는 다음과 같습니다
```
-- stg_stripe__payments.sql

with

source as (
    select * from {{ source('stripe','payment') }}
),

renamed as (
    select
        id as payment_id,
        orderid as order_id,
        paymentmethod as payment_method,
        case
            when payment_method in ('stripe', 'paypal', 'credit_card', 'gift_card') then 'credit'
            else 'cash'
        end as payment_type,
        status,
        amount as amount_cents,
        amount / 100.0 as amount,
        case
            when status = 'successful' then true
            else false
        end as is_completed_payment,
        date_trunc('day', created) as created_date,
        created::timestamp_ltz as created_at
    from source
)

select * from renamed
```
이 예시에서 볼 수 있는 주요 패턴
- 변환 유형
    - ✅ 이름 바꾸기 (renaming)
    - ✅ 형 변환 (type casting)
    - ✅ 기본 연산 (예: 센트를 달러로 변환)
    - ✅ 카테고리화 (조건문으로 범주화)

- 지양되는 변환
    - ❌ 조인 (joins) — 스테이징 단계에서는 각 소스 테이블을 독립된 원자 단위로 준비
    - ❌ 집계 (aggregations) — 그룹화를 하면 원시 상세 데이터가 사라질 위험 있음
- Materialization 방식
    - 일반적으로 스테이징 모델은 view 로 materialize
    - 다운스트림 모델이 최신 데이터를 바로 참조 가능
    - 웨어하우스 공간 낭비 방지
- source 매크로 사용
    - 스테이징 계층에서만 source 매크로 사용
    - 스테이징 모델은 소스 테이블과 1:1 관계 유지
** DRY 원칙 적용 **
- 반복적인 변환 로직은 가능한 한 상류 계층 (staging) 으로 올려 중복을 제거
- 예: 금액 단위 변환, 형 변환 등은 downstream에서 하지 않고 스테이징에서 처리
----
### Staging: Other considerations
- Base 모델
    - 조인이 필요할 경우 스테이징 내 하위 디렉터리에서 base 모델을 두어 처리(예: 삭제 테이블(join) 연결, 여러 소스 병합 등)
- 동형 소스 병합 (Unioning multiple identical sources)
    - 여러 지역의 전자상거래 플랫폼 등 동일 스키마 테이블이 여러 개 있을 경우, 먼저 병합한 후 이후 로직 수행
- 자동 생성 (Codegen)
    - 스테이징 모델은 반복되는 패턴이 많기 때문에 boilerplate 생성을 자동화하는 패키지 사용 권장
- Utilities 폴더
    - models/utilities 디렉터리는 스테이징 폴더 밖에 있지만, 공통적으로 사용되는 도구성 모델(예:date spine)을 보관
- 개발 흐름 vs DAG 순서
    - 이 가이드는 DAG 흐름 순서대로 설명하지만, 실제 개발은 반드시 그런 순서로 하지 않음
    - 일반적으로: 스프레드시트로 설계 → SQL 논리 작성 → 필요한 스테이징 모델 생성 → mart 모델 구성 → 리팩터링
    - 로직을 상류 계층으로 끌어올려 각 계층을 깔끔하게 유지하고 테스트 적용 가능 영역을 넓힘
----
### 요약
- 스테이징 계층의 목적은 원시(source) 데이터를 정제해 downstream에서 활용 가능한 원자 단위 구성 요소로 만드는 것
- 폴더 구조는 소스 시스템별 하위 디렉터리 중심이며, 비즈니스 그룹별 구분은 피하는 것이 좋음
- 파일 명명 규칙: stg_[source]__[entity]s.sql 같이 명확하고 일관성 있게
- 모델 구조: 이름 변경, 형 변환, 기본 연산, 카테고리화 등을 포함하되 조인이나 집계는 피함
- Materialization: 스테이징 계층은 보통 view 로 처리
- 기타 고려사항: 조인이 필요한 경우 base 모델 사용, 동형 소스 병합, 코드 자동 생성 활용, 유틸리티 모델 분리, 개발 흐름과 DAG 흐름의 차이 인식
  
### 참고
- https://docs.getdbt.com/best-practices/how-we-structure/2-staging#staging-models