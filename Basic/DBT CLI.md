## 주요 dbt 명령어
### 프로젝트 초기화
- `dbt init`
```
dbt init <프로젝트명>
```
    - <프로젝트명> 폴더를 생성하고, 그안에 기본 프로젝트 구조를 만든다.
    - `dbt_project.yml`, `models/`, `test/`, `source/` 등 생성
    - DW와 연결을 위한 profiles.yml 파일이 생성된다.
    - ~/.dbt/profiles.yml 파일에 Connection 정보 입력
    ```
    my_first_prj:
    outputs:
        dev:
        type: duckdb
        path: ./dev.duckdb
        threads: 1
        schema: dbt_dev
        prod:
        type: duckdb
        path: ./prd.duckdb
        threads: 4
        schema: prod
    target: dev
    ```
### 기본 실행 명령어
```
- `dbt run`
    - 모델을 실행하여 새로운 모델(테이블/뷰)을 생성
    - `--select` or `-s` 옵션으로 특정 모델만 실행 가능
    - 모델명 앞 뒤에 `+`를 추가하여 선행 모델(업스트림) 혹은 후행 모델(다운스트림) 모델까지 실행 가능
    - dbt run 작업 시 1.tmp 테이블에 데이터 저장, 2.기존 테이블 backup 테이블로 rename, 3. tmp 테이블 본테이블로 rename, 4. backup 테이블 삭제 
    - models/ 폴더 사용
- `dbt test`
    - 데이터 품질 테스트 실행
    - `--select` or `-s` 옵션으로 특정 모델만 실행 가능
    - test/ 폴더 사용
- `dbt seed`
    - CSV파일을 업로드하여 DB에 테이블로 저장 가능
    - seed/ 폴더 사용
- `dbt source`
    - DB원천 데이터 등록
- `dbt snapsoht`
    - 스냅샷 정의를 실행하여 데이터 변경 이력을 관리
- `dbt build`
    - `run`과 `test`를 동시에 실행
  
### 개발 & 디버깅 명령어
- `dbt debug`
    - 연결, 설정 등 환경을 점검
    - profiles.yml, dbt_project.yml, git 설치 확인
    - Window에서 profiles.yml path 지정 시 UTF-8 에러 발생... 원인 확인 필요
` `dbt compile`
    - 모델을 Jinja로 렌덩리하여 SQL 파일을 생성
  
### 문서화 & 메타데이터
- `dbt docs generate`
    - 메타데이터(JSON) 파일 생성
- `dbt docs serve`
    - dbt 정석 웹서버 실행
    - 데이터 리니지 혹은 메타데이터 확인 가능