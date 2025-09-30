### BigQuery and dbt
BigQuery는 서버리스 MPP(Massive Parallel Processing) 기반으로, 대규모 SQL 처리가 강점입니다. 여기에 dbt를 활용할 수 있습니다.
  
### GCP Cloud Shell
GCP에서 제공하는 Cloud Shell을 통해 간단하고 빠르게 dbt를 설치하고 테스트 할 수 있습니다.  
특히 Window를 사용하는 경우 Cloud Shell을 사용하는 것이 간편합니다.  
(Window 사용 시 Gcloud 인증, Git 설치, 그외 dbt 설정 과정 중 이슈 발생할 수 있음)  
  
### pip3 install dbt
```
pip3 install --user --upgrade dbt-bigquery
```  
만약 버전 충돌 혹은 그외 이슈가 발생하는 경우
```
pip3 uninstall dbt-core
pip3 install --user --upgrade dbt-bigquery
```  
  
### dbt Project init
```
~/.local/bin/dbt init first_project
```

### Source 생성
- BigQuery 테이블 생성
```
CREATE OR REPLACE TABLE newyork_taxi.tlc_green_trips_2022
PARTITION BY DATE(pickup_datetime)
CLUSTER BY pickup_location_id, dropoff_location_id
AS
SELECT *
FROM bigquery-public-data.new_york_taxi_trips.tlc_green_trips_2022;
```  
  
- Soruce 확인
```
dbt ls
```

### Dimesion & Fact Model 생성
```
dbt run -s dim_locations

dbt run -s fact_trips
```  
  
### Mart 생성
```
dbt run -s mart_location_partters
```  
- 업스트림 모두 실행
```
dbt run -s +mart_location_partters
```