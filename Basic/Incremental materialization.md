### Incremental materialization
- 운영환경에서 DB는 대용량의 데이터이기 때문에 view, table삭제 후 생성 작업이 부담될 수 있음
- Incremetal은 증분 데이터만 추가 하는 방법
- 증분 데이터를 append 또는 delete&insert 등 전략을 취할 수 있음
    - 데이터베이스에 따라서 전략이 다름.
- 스키마가 변경이 된 경우 기존과 동일한게 full refresh(drop&create)할 수 있음
- is_incremental()이 True가 되는 경우
    - 모델의 테이블이 이미지 존재해야됨
    - materialized = 'incremetal'로 설장되어 있어야함
    - full-refresh의 Flag가 없어야 함
- unique key를 설정해야 키를 기준으로 delete&insert가 됨.
    - Database에서 merge를 지원해야됨.
- 만약 증분 모델 전체를 다시 빌드하기 위해서는 --full-refresh 플래그 추가 필요
    - dbt run --full-refresh --select my_incremetal_model+
    - 다운 스트림 모델도 Rebuild하는 경우 "+" 추가
- incremetal stategy는 DB 마다 다름
    - duckdb는 append와 delete+insert만 가능

### 방법1
- incremental_model.yml
```
# append전략을 취할 때는 unique_key 키 필요 없음
{{
    config(
        materialized='incremental',
        unique_key=['target_index_name', 'ticket', 'date_kst'],
        incremetal_strategy='delete_insert'
    )
}}
select
    *
from
    {{  sorucre("main", "source_table")  }}
where
{% if is_incremental() %}
    DATE_COLUMN >= (
        select max(DATE_COLUMN) from {{ this }}
    )
{% endif %}
```  

### 방법2
- 방법1의 단점
    - 모델이 없는 경우 is_incremental() 조건을 만족시키지 못하기 때문에 Source테이블을 Full Scan해야 되기 때문에 메모리 관점에서 좋지 못함.
    - 멱등성을 보장하지 못함 : 예를 들어 방법1에서 incremental 조건절이 ">"인 경우 2번 연속으로 실행되는 경우 변화가 없음 따라서 결과가 인풋으로 들어가기 때문에 예측이 어려워 장애를 초래할 위험이 있음

- incremental_model.yml
```
{{
    config(
        materialized='incremental',
        unique_key=['target_index_name', 'ticket', 'date_kst'],
        incremetal_strategy='delete_insert'
    )
}}
select
    *
from
    {{  sorucre("main", "source_table")  }}
where
    date_kst >= '{{ var("start_dt")  }}'
    and date_kst <= '{{ var("end_dt")  }}'
```

### 참고
- https://docs.getdbt.com/docs/build/incremental-models