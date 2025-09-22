## Data Quality
- 2가지 방식의 데스트가 있음
    1. data_tests(tests) : 테이블 생성 후 데이터 검증
        - Generic test : 테스트가 포맷이 정해져있음, 모든 테이블에 쓰이는 테스트(ex. 특정 컬럼 not null, unique 확인), YAML 파일을 통해 검증
        - Singular test : 특정 테이블에만 쓰이는 테스트, 쿼리를 통해 검즘
    2. unit_tests : 테이블 생성 전 쿼리 로직 검증

### 1. Generic Test
- models 폴더 > YAML 파일에 정의함
- schema.yaml
  - 테스트의 name은 unique해야됨.
```
models:
  - name: price_daily_merged
    description: price merged data
    columns:
      - name : ticker
        tests:
          - not_null:
              name: my_not_null
              config:
                serverity: warn
          - unique
          - accepted_values
              values: ['APPL', 'MSFT', 'GOOGLE]
          - relationship

  - name: kospi200_consitituents_daily
  - name: price_of_kospi200_daily
```
```
dbt test
```

### 2. Singular Test
- SQL Based Test
- tests 폴더에서 진행
- dbt_project.yml의 test-paths 확인
- price_data_high_is_highest.sql
```
select
    *
from {{ ref("price_daily_merged") }}
where
    high < low
    or high < close
    or high < open
```