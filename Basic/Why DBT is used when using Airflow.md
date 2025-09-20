### Why DBT is used when using Airflow
Airflow를 사용하여 ELT 파이프라인 개발할 때 DBT를 사용하지 않는 경우 아래와 같은 문제(불편함)가 있음.
1. SQL 관리 어려움
- Opoerator 내 SQL을 직접 넣기 때문에 관리가 어렵고 가독성이 떨어짐
- 파일로 SQL 코드를 관리하는 경우에도 관리가 어려움
  
2. 리니지 관리 어려움
- DAG 내 여러 Operaotr(Python, SQL, Senssor,..)를 사용하기 때문에 Task가 많아져 복잡해짐
  
3. 테스트 및 품질 관리 어려움
- Airflow 내에서 Data Quality 기능 제공x
- dbt는 내장 테스트 혹은 테스트 패키지 등을 사용하여 쉽게 검증 가능
  
=> DBT를 사용하여 Airflow DAG를 구성하는 경우 위 문제 해소 가능