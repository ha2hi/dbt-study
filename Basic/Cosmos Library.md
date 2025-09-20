## Cosmos Library 
### Astromer Cosmos 라이브러리 사용 이유??
예를들어 ELT 파이프라인을 Airflow에서 실행하는 경우 `모델1 run -> 모델1 test -> 모델2 run -> 모델2 test..`를 1개의 DAG로 구성해야됨.  
위와 같이 구성한 경우 모든 DBT Model의 RUN/TEST 조합을 수동으로 DAG에 나열해야 함.  
- 모델 수가 많아지는 경우 반복 작업이 늘고, 잘못 연결 시 의존성 문제 발생(휴먼 에러)
- Airflow DAG는 Task 단위의 Dependency만 표현 가능, DBT 데이터 모델간 의존성을 직접 반영하기 어려움 
이를 Astromer의 Cosmos lib를 사용하는 경우 해당 문제를 해결 가능함.  
  
### Astromer 분석
- Astromer의 dbt 관련 Operator들은 Airflow 내부 소스코드 구조와 동일하게 BaseOperator를 기본으로 상속 받음
- local operator 파일을 확인해보면 run, seed, test Class가 존재 이는 cli에서 dbt run, dbt test와 동일한 기능
- 따라서 local Operator만 이해하면 docker, kubernetes 이해도 수월할 것 임.
- 실제로 사용할 때는 개별 Operator를 사용하는 것 보다는 추상화된 Class인 DbtDag 혹은 DbtTeskGroup을 사용

### DbtTaskGroup
- DbtTaskGroup을 사용하기 위해서는 4가지 config가 반드시 필요
```
@dag(dag_id="dag_with_dbt", **dag_args)
def dbt_test():
    vars = {
        "end_dt" : "{{  data_interval_end.in_timezone('Asia/Seoul').strftime('%Y-%m-%d')  }}"
    }

    tg = DbtTaskGroup(
        project_config=ProjectConfig(
            manifest_path=os.path.join("/usr/local/dbt_repo", "target", "manifest.json"),
            project_name="my_first_dbt_project",
            dbt_vars = vars
        ),
        profile_config=ProfileConfig(
            profile_name="my_first_dbt_project",
            target_name="dev",
            profiles_yml_filepath=os.path.join("/usr/local/dbt_repo", "profiles", "profiles.yml"),
        ),
        render_config=RenderConfig(
            load_method=LoadMode.DBT_MANIFEST,
            test_behavior=TestBehavior.AFTER_EACH,
            select=[
            ],
        ),
        execution_config=ExecutionConfig(
            execution_mode=ExecutionMode.LOCAL,
            invocation_mode=InvocationMode.DBT_RUNNER,
            dbt_project_path="/usr/local/dbt_repo"
        ),
        operator_args={
            "install_deps": True
        },
    )

    from airflow.operator.empty import EmptyOperator
    empty_oper = EmptyOperator(
        task_id = "some_task",
    )

    a >> tg.tasks_map["model.<프로젝트명>.<모델명>"]
```
- project_config : 프로젝트 설정
    - manifest_path : DBT 정보인 manifest 파일 위치 지정
    - project_name : 프로젝트 이름
    - dbt_vars : Jinja 템플릿 변수 전달
- profile_config : Database 연결 정보
    - profile_name : dbt에서 사용할 profile 이름(profiles.yml의 key와 같음)
    - target_name : 사용할 실행 대상(ex. dev, prod)
    - profiles_yml_filepath : profiles YAML 파일을 DBT 프로젝트에 저장한걸 사용하는 경우 위치 지정
        - profile을 선택할 때 yml 파일을 사용하거나 Airflow의 Connection으로 profile을 생성하여 지정할 수 있음
- render_config : DBT 프로젝트를 Airflow의 DAG/TaskGroup으로 렌더링(=변환)할 때 동작 방식을 제어하는 설정
    - load_method : DBT 모델/리소스를 DAG로 변환 시 어떤 방식을 사용하지 지정
        - LoadMode.DBT_MANIFEST -> manifest.json 사용
        - LoadMode.DBT_LS -> dbt ls 명령어 실행
    - test_behavior : 모델->테스트->모델 옵션 혹은 모델->모델 옵션등 여러 가지 실행 옵션 선택 가능
        - TestBehavior.AFTER_EACH : 각 모델 실행 직후 해당 모델의 테스트 실행
        - TestBehavior.NONE : run만 실행되고 test 실행X
        - TestBehavior.AFTER_ALL : 모든 모델 실행이 끝난 뒤 한 번 에 테스트 실행
    - select : 어떤 모델을 실행할 것 인지 지정
        - dbt CLI의 --select와 동일
- execution_config : 실행 모드 선택
    - execution_mode : 어떤 환경에서 실행할지 결정
        - ExecutionMode.LOCAL → Airflow 워커 안에서 직접 실행
        - ExecutionMode.DOCKER → Docker 컨테이너에서 실행
        - ExecutionMode.KUBERNETES → KubernetesPodOperator로 실행
    - invocation_mode : 어떻게 실행할지
    - dbt_project_path : DBT 프로젝트 위치
- operator_args : 기존 Dag의 Arg 입력 가능
