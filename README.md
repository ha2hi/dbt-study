# dbt-study
### dbt 소개
1. DBT?
- Data Build Tool("T" in ELT, ETL)
- Tool(Python lib) used in buuilding a data warehouse
- Framework(like a Django or Flask)
- Independent with Warehouse, orchestration soluion

2. DBT 핵심 개념
- SQL + yaml + jinja
- Resources : DBT의 기본 컴포넌트
  - sources : 외부의 원천 데이터
  - models : Sources로 부터 만든 테이블
  - tests
  - semantic_models
  - metrics
- Profiles
  - Data Warehouse onnection info
  - 환경(Prod, dev) 분리
- Compile + Run 구조
  - DBT에서 개발한 YAML, Jinja는 최종적으로 웨어하우스가 이해할 수 있는 SQL로 컴파일 되어야함