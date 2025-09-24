## dbt 프로젝트 구조화 방법
### 왜 구조가 중요한가?
Analytics Engineering의 본질은 협업 팀들이 규모 있게 더 나은 의사결정을 할 수 있도록 돕는 것입니다.  
우리는 의사결정을 할 수 있는 역량이 한정(의사결정 결정 피로)되어 있으며, 사회적 동물로서 시스템과 패턴에 의존해 타인과의 협업을 최적화합니다.  
이런 특성 때문에, 협업 프로젝트에서는 누가 어디에 폴더를 만들고 어떻게 이름 짓는지와 같은 ** 사소한 결정들이 반복되지 않도록 일관성 있고 이해하기 쉬운 규칙을 정해두는 것이 중요합니다. **  
dbt 프로젝트에서는 여러 부서의 도메인 지식이 모여 기업 전체의 목표와 내러티브를 코드로 표현하는 협업 작업입니다.  
따라서 가능한 많은 사람들이 자신의 전문성을 긍정적으로 활용할 수 있도록 하고, 조직이 확장될 수 있도록 프로젝트가 접근 가능하고 유지보수 가능하도록 깊고 폭넓은 패턴을 마련하는 것이 중요합니다.  
스티브 잡스가 매일 같은 옷을 입어 의사결정 피로를 줄였듯이, 이 가이드는 프로젝트에 대한 "검은 터틀넥" + "뉴발란스"같은 일관된 복장을 제안합니다.  
여기서 복장이라 함은 의상 대신 파일, 폴더, 네이밍 컨벤션, 코드 패턴 등을 의미합니다.  
어떻게 명명하고, 그룹화하고, 쪼개고 합칠 것 인지 이 것이 바로 프로젝트 구조입니다.  
이 가이드는 출발점일 뿐입니다.  
조직마다 “자신만의 옷 스타일”(컨벤션)을 선택할 수 있지만, 그 선택을 명확히 문서화하고 팀원 모두가 접근할 수 있게 선언하고, 무엇보다 일관성을 유지하는 것이 중요합니다.  
그렇지만 모든 dbt 프로젝트에 적용되는 기본 원칙이 하나 있습니다.
바로 데이터를 source-conformed 상태에서 business-conformed 상태로 이동시키는 일관된 흐름을 만드는 것입니다.  
source-conformed 데이터는 외부 시스템으로부터 우리가 제어할 수 없는 형태로 주어지는 반면, business-conformed 데이터는 우리 조직이 정의하는 개념과 요구에 따라 재형성된 데이터입니다.  
어떤 패턴이나 규칙을 프로젝트에 도입하든, 이 변환 흐름을 유지하는 것이 변환 계층(transformation layer)의 본질이며, dbt가 이를 수행하는 도구라는 점은 변하지 않습니다.  
  
이 가이드는 원래 Claire Carroll의 동일한 제목의 애널리틱스 엔지니어링 게시물에서 영감을 받은 것이며, 시간이 흐르면서 일부 세부사항은 바뀌었지만 이 핵심 궤적은 유지되고 있습니다.  
새로운 도구가 시야를 확장할 때마다, 새로운 경험이 시각을 날카롭게 할 때마다, 다양한 관점이 더해질 때마다 이 가이드는 점진적으로 업데이트될 것입니다.
그러나 그 모든 변화는 결국 데이터가 source-conformed 에서 business-conformed 로 흐르는 과정을 지원하는 데 초점을 둡니다.  
----
### 학습 목표
이 가이드가 달성하고자 하는 주요 목표는 다음 세 가지입니다.  
1. 일반적인 dbt 프로젝트를 구조화하는 최신 권장 방식을 철저히 다룰 것
2. 이러한 권장을 포괄적인 예제로 설명할 것
3. 각 단계마다 왜 이러한 방식을 추천하는지를 설명하여, 조직의 고유한 요구에 맞게 언제 어떻게 벗어날지 스스로 판단할 수 있게 할 것  
  
이 가이드를 통해, dbt 프로젝트 구성 요소들이 어떻게 맞물리는지를 더 깊이 이해하고, 애널리틱스 엔지니어링의 목적과 원칙이 더 명확하고 직관적으로 느껴지도록 돕고자 합니다.  
의도적으로 구조를 설계하면, 데이터 흐름 뿐 아니라 코드베이스와 창출되는 아티팩트들(산출물)까지 스토리를 지닌 구성이 가능해집니다.  
이 패턴들과 그 기반이 되는 원칙 간의 연결성을 강화함으로써, 여러분은 조직의 필요에 맞는 문서를 작성하고 팀이 실제로 활용할 수 있는 구조를 만들 수 있게 될 것입니다.  
이 가이드는 ‘Jaffle Shop’이라는 가상의 회사를 예시로 삼아 설명합니다.  
이 회사는 두 가지 주요 데이터 소스를 가집니다. 
- jaffle_shop라는 관계형 트랜잭션 DB 복제본 (고객, 주문 등 핵심 엔터티 포함)
- 결제 처리용 Stripe 연동 데이터  
----
### 가이드 구조
1. models 폴더를 세가지 기본 계층에 대한 파일, 폴더 및 모델을 어떻게 구성했는지 살펴보겠습니다.
- staging : 소스 데이터로부터 초기 모듈형 구성 요소를 생성
- intermediate : 특정 목적을 갖는 변환 논리를 쌓아 가기
- marts : 조직이 중요하게 여기는 비즈니스 엔터티 완성
2. 프로젝트의 나머지 구조
- models 외에도 tests, seeds, analyses, macros 등 폴더와 YAML 파일 설정이 어떻게 유기적으로 작동하는지 명세
아래는 이 프로젝트의 전체 파일 트리 예시입니다  
```
jaffle_shop
├── README.md
├── analyses
├── seeds
│   └── employees.csv
├── dbt_project.yml
├── macros
│   └── cents_to_dollars.sql
├── models
│   ├── intermediate
│   │   └── finance
│   │       ├── _int_finance__models.yml
│   │       └── int_payments_pivoted_to_orders.sql
│   ├── marts
│   │   ├── finance
│   │   │   ├── _finance__models.yml
│   │   │   ├── orders.sql
│   │   │   └── payments.sql
│   │   └── marketing
│   │       ├── _marketing__models.yml
│   │       └── customers.sql
│   ├── staging
│   │   ├── jaffle_shop
│   │   │   ├── _jaffle_shop__docs.md
│   │   │   ├── _jaffle_shop__models.yml
│   │   │   ├── _jaffle_shop__sources.yml
│   │   │   ├── base
│   │   │   │   ├── base_jaffle_shop__customers.sql
│   │   │   │   └── base_jaffle_shop__deleted_customers.sql
│   │   │   ├── stg_jaffle_shop__customers.sql
│   │   │   └── stg_jaffle_shop__orders.sql
│   │   └── stripe
│   │       ├── _stripe__models.yml
│   │       ├── _stripe__sources.yml
│   │       └── stg_stripe__payments.sql
│   └── utilities
│       └── all_dates.sql
├── packages.yml
├── snapshots
└── tests
    └── assert_positive_value_for_total_amount.sql
```  
  
*결정피로 : 장기간 의사결정을 한 후 내리는 의사결정의 질이 저하되는 현상

## 참고
- https://docs.getdbt.com/best-practices/how-we-structure/1-guide-overview