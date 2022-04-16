---
layout: post
title: 데이터 파이프라인 핵심 가이드 정리(7장) 
---

#  데이터 파이프라인 핵심 가이드

01. 데이터 파이프라인 소개
02. 최신 데이터 인프라
03. 일반적인 데이터 파이프라인 패턴
04. 데이터 수집: 데이터 추출
05. 데이터 수집: 데이터 로드
06. 데이터 변환하기
07. 파이프라인 오케스트레이션
08. 파이프라인 데이터 검증
09. 파이프라인 유지 관리 모범 사례
10.  파이프라인 성능 측정 및 모니터링 


## 07. 파이프라인 오케스트레이션
* 이전 장까지 파이프라인 빌딩 블록(수집, 변환 등)에 대해 설명했다. 
* 오케스트레이션은 파이프라인 각 단계가 올바른 순서로 실행되고 종속성이 적절하게 관리되게 한다.
* 이 장에서는 대부분 아파치 에어플로우(apache airflow) 예제에 집중한다.

### 방향성 비순환 그래프(DAG)
* 파이프라인 단계는 항상 한 방향(directed)으로 흐른다. (실행경로 보장)
* 파이프라인 그래프는 이전에 완료된 작업을 재실행하지 못한다(순환적)

### 아파치 에어플로우 설정 및 개요
* 에어플로우는 처음 출시된 6년 동안 가장 인기있는 워크플록 관리 플랫폼 중 하나가 되었다.
	* 웹 인터페이스
	* 고급 명령줄 유틸리티
	* 높은 수준의 사용자 정의 기능
	* 파이썬 빌드 -> 모든 언어 또는 플랫폼에서 실행
* 모든 종류의 종속 작업을 오케스트레이션 가능하다.

		NOTE - 이 장의 코드 샘플 및 개요는 에어플로우 1.x를 참조한다.

#### 설치 및 구성
* 설치는 매우 간단하다. pip을 사용하여 설치하자. 공식 에어플로우 빠른 시작 가이드를 참조한다. 
* http://localhost:8080 으로 실행 가능하다.

#### 에어플로우 데이터베이스
* 에어플로우는 DB를 사용하여 각 작업 및 DAG의 실행 기록 및 에어플로우 구성과 관련된 모든 메타데이터를 저장한다.
* 기본은 SQLite이지만 대규모 요구사항은 MySQL, Postgres를 사용하자.
* 구성 방법은 중략한다.

#### 웹 서버 및 UI
* airflow 홈페이지에는 DAG 목록이 표시된다. 
* 페이지에는 DAG에 대한 여러 링크와 정보가 있다.
	* 소스 파일이 있는 경로, 태그, 설명 등을 포함한 DAG 속성을 여는 링크
	* DAG 활성화/일시중지하는 토글
	* DAG 세부정보 페이지
	* DAG이 실행되는 일정(일시중지 아닐경우)
	* DAG 소유자
	* 최근 작업
	* DAG 마지막 실행 타임스탬프
	* 이전 DAG 실행 요약
	* DAG 구성 및 정보에 대한 링크 세트

* Code와 Trigger는 자세하게 살펴보자.
	* 코드 탭을 클릭하면 코드를 볼 수 있다. 작업간 정의, 종속성, DAG 구성을 확인하자.
	* 트리거 DAG 버튼은 DAG을 실행한다. 
* DAG 관리 이외에도 유용한 UI 기능을 살펴보자.
	* Data Profiling - Ad Hoc Query, Charts, Known Event 옵션 표시 (에어플로우 DB의 정보를 쿼리할 수도 있다.)
	* 찾아보기(Browse)에서 DAG 및 기타 로그 파일 실행 기록을 찾을 수 있으며 관리(Admin)에서 다양한 구성 설정을 찾을 수 있다.
	* 더 자세한 내용은 도큐먼트의 고급 구성 옵션을 찾아보자.


#### 스케줄러 
* airflow scheduler를 실행할 때 시작한 서비스다. 
* 실행 중일 때 스케줄러는 DAG / 작업을 모니터링하고 실행되도록 예약되었거나 종속성이 충족된 모든 작업을 실행한다. (DAG 작업인 경우)
* airflow.cfg 파일의 [core] 섹션에서 설정한다.
#### 실행기(Executors)
* 에어플로우가 지원하는 다양한 유형의 실행기가 있다.
* 기본적인 것은 SequentialExecutor가 사용되고 airflow.cfg 파일에서 실행기 유형을 변경할 수 있다.
	* SequentialExecutor는 한 번에 하나의 작업만 실행되므로 프로덕션 사용에는 적합하지 않다.
* 어떤 규모로든 에어플로우 사용은 CeleryExecutor, DaskExecutor, KubernetesExecutor와 같은 실행기를 사용하자.

#### 연산자(Operators)
* DAG에서 각 노드는 하나의 작업이다. 각 작업은 연산자를 구현한다.
* 연산자는 스크립트, 명령, 기타 작업을 실행한다.
* 연산자 종류
	* BashOperator
	* PythonOperator
	* SimpleHttpOperator
	* EmailOperator
	* SlackAPIOperator
	* MySqlOperator, PostgresOperator, SQL 명령 실행 위한 DB 연산자
	* Sensor

### 에어플로우 DAG 구축
* DB에서 데이터를 추출하여 DW에 로드한 다음 데이터를 데이터 모델로 변환해보자.
#### 간단한 DAG
* 먼저 DAG이 어떻게 정의되는지 살펴보자.
* 아래의 예제는 세 가지 작업이 있는 DAG이다. 이것을 DAG 정의 파일(definitin file)이라고 한다.
* 각 작업은 BashOperator로 정의되며 아래와 같은 작업을 실행한다.
	1. 텍스트 print
	2. 3초 대기
	3. 텍스트 print

<script src="https://gist.github.com/JeremyShin/b0bab05fd922ff713bfe5eb5504b742c.js"></script>

* 파이썬 스크립트에서 필요 모듈을 가져온다. 
* 다음으로 DAG이 정의되고 이름(simple_dag), 일정, 시작 날짜 등과 같은 일부 속성이 할당된다. 
* DAG에서 3가지 작업을 정의한다. BashOperator로 bash 명령어가 실행된다.
* 마지막 두 줄은 작업 간의 종속성을 정의한다. 
* DAG 실행을 위해서는 airflow.cfg 파일에서 DAG 위치를 설정한다.

		dags_folder = /User/myuser/airflow/dags

* 위의 파이썬 파일(simple_dag.py)을 dags_folder 위치에 배치한다.
* 웹의 UI에서 simple_dag 이라는 파일을 찾자. 나타나지 않는다면 UI를 새로 실행한다.
* DAG의 이름을 클릭하여 DAG의 토글을 ON으로 변경하자.
* DAG Runs에서 실행 상태를 확인 가능하다.

		TIP - DAG의 start_date를 하루 전으로 설정하여 활성화하자마자 
		실행이 예약되고 실행되게 했다.
		프로덕션 배포에서는 원하는 미래의 날짜로 하드 코딩하는 것이 더 
		합리적일 수 있다.
		

#### ELT 파이프라인 DAG
* 간단한 DAG을 만들어보았으니 ETL를 위한 DAG을 빌드해보자.
* 이 DAG은 5가지 가지 작업으로 구성된다.
* BashOperator를 사용하여 postgres DB 테이블에서 데이터 추출하고 결과를 CSV 파일로 S3 버켓에 보내는 두 개의 다른 파이썬 스크립트를 실행
* S3 버켓에서 DW로 데이터를 로드하는 해당 작업이 실행된다.
* BashOperator를 사용하여 CSV를 로드하는 로직이 포함된 파이썬 스크립트를 실행한다.

		TIP - 위 예제는 오케스트레이션과 실행하는 프로세스의 로직을 더 많이
		 분리하고 싶었다. 이 구성은 파이썬 라이브러리 호환 문제를 피할 수 
		 있다. 일반적으로 프로젝트(및 Git 저장소)를 분리하여 데이터 인프라
		  전반에 걸쳐 로직을 유지 관리하는 것이 더 쉽다.
		  오케스트레이션과 파이프라인 프로세스 로직도 다르지 않다.

<script src="https://gist.github.com/JeremyShin/be969afb2e4bc59f1f60e904fae46c6f.js"></script>

* 마지막 작업은 PostgresOperator를 사용하여 DW의 에어플로우와 동일한 시스템의 디렉터리에 저장된 sql 스크립트를 실행한다.
* SQL 스크립트 내용은 6장의 데이터 모델 변환과 비슷하다. 

<script src="https://gist.github.com/JeremyShin/95905284a906c4bf68b311f47cc0fa46.js"></script>

* DAG 활성화 전에 PostgresOperator에 사용할 연결을 설정해야 한다.
* 에어플로우 웹 UI에서 redshift_dw 연결을 정의하자.
* 아래의 단계를 따라보자.
	1. 웹 UI 상단 탐색 모음 -> 관리자(admin) -> 연결(connection)
	2. create 탭 클릭
	3. conn id를 redshift_dw로 설정한다.
	4. 연결 유형으로 Postgres를 선택한다.
	5. DB연결 정보를 설정한다.
	6. 저장한다.

* 이제 DAG 활성화 준비가 끝났으니 DAG을 실행해보자.


### 추가 파이프라인 작업

#### 경고 및 알림
* 알림을 보내는데는 여러가지 옵션이 있다. 
* 에어플로우 이메일을 보내기 전에 airflow.cfg의 [smtp] 섹션에 SMTP 서버에 대한 세부 정보를 제공해야 한다.
* 작업에서 EmailOperator를 사용하여 DAG의 어느 지점에서나 이메일을 보낼 수 있다. 

		email_task = EmailOperator(
			task_id=‘send_email’,
			to=“me@example.com”,
			subject=“Airflow Test Email”,
			html_content=‘some test content’,
		)

* 이외에도 슬랙, 마이크로소프트 팀즈 및 기타 플랫폼에 메세지를 보내기 위한 공식/커뮤니티 지원 운영자가 있다.

#### 데이터 유효성 검사
* 데이터 유효성 검사를 실행하기 위해 에어플로우 DAG에 작업을 추가하는 것이 좋다.
* 유효성 검사는 sql, 파이썬 스크립트에서 구현하거나 다른 외부 응용 프로그램을 호출하여 구현할 수 있다.

### 고급 오케스트레이션 구성
* 이번 섹션에서는 더 복잡한 파이프라인을 구축하거나 공유 종속성 또는 다른 일정으로 여러 파이프라인을 조정해야 할 때 직면할 수 있는 몇 가지 문제를 소개한다.

#### 결합된 파이프라인 작업 vs 결합되지 않은 파이프라인 작업
* 데이터가 지속적으로 DW로 흐르지만 데이터 변환은 30분마다 설정된 간격으로 실행되도록 예약한다. 데이터 수집의 특정 실행은 데이터를 데이터 모델로 변환하는 작업에 직접 종속되어 있지 않다. 이러한 상황은 결합되지 않은 작업으로 본다.
* 데이터 엔지니어는 파이프라인을 조정하는 방법에 대해 신중해야 한다. 파이프라인 전반에 걸쳐 일관되고 탄력적인 결정을 내려야 한다.
#### DAG을 분할해야 하는 경우
* 데이터 인프라에서 ELT, 유효성 검사, 경고 작업이 포함된 DAG은 때때로 매우 복잡해지기도 한다.
* 작업을 여러 DAG으로 분할해야 하는 시점과 단일 DAG에 유지해야 하는 시점을 결정하는 세 가지 요소를 보자
	* 작업을 다른 일정으로 실행해야 하는 경우 여러 DAG으로 나눈다.
	* 파이프라인이 진정으로 독립적인 경우 별도로 유지된다.
	* DAG이 너무 복잡해지면 논리적으로 분리될 수 있는지 여부를 결정한다.

#### 센서로 여러 DAG 조정
* 에어플로우 Sensor는 일부 외부 작업 또는 프로세스 상태를 확인하고 기준이 충족되면 DAG에서 다운스트림 종속성을 계속 실행하도록 설계되었다.
* 두 개의 서로 다른 에어플로우 DAG을 조정해야 하는 경우 ExternalTaskSensor를 사용하여 다른 DAG의 작업 상태 또는 DAG의 전체 상태를 확인할 수 있다.

<script src="https://gist.github.com/JeremyShin/84cf0db112729908879fa3556ac11024.js"></script>

* DAG이 활성화되면 dag_sensor 작업을 시작한다. 속성을 살펴보자
	* external_dag_id는 센서가 모니터링할 DAG의 ID이다. 이 경우 elt_pipeline_sample DAG이다.
	* external_dag_id 속성은 None으로 설정되는데 센서는 전체 elt_pipeline_sample DAG이 success까지 기다린다.
	* mode 속성이 일정 변경(reschedule)로 설정된다. 
	* timeout 파라미터는 ExternalTaskSensor가 시간 초과되기 전에 외부 종속성을 계속 확인하는 시간으로 설정된다.
* 기억할 사항은 DAG이 특정 일정에 따라 실행되므로 센서가 특정 DAG 실행을 확인해야 한다. 
* 두 DAG이 서로 다른 일정으로 실행되는 경우 센서가 확인해야 하는 elt_pipeline_sample의 실행을 지정하는 것이 가장 좋다. ExternalTasksensor의 execution_delta 또는 execution_date_fn 파라미터를 사용하여 이를 수행할 수 있다.
* execution_delta 매개변수를 사용하여 특정 실행을 볼 수 있다. 예를 들어 30분마다 예약된 DAG의 최근 실행을 보려면 다음과 같이 정의된 작업을 생성한다.

		sen1 = ExternalTaskSensor(
						task_id='dag_sensor',
						external_dag_id = 'elt_pipeline_sample',
						external_task_id = None,
						dag=dag,
						mode='reschedule',
						timeout=2500,
						execution_delta=timedelta (minutes=30))


### 관리형 에어플로우 옵션
* 프로덕션 레벨에서는 인스턴스를 최신 상태로 유지하고, 기본 리소스를 확장하는 것은 데이터 엔지니어가 모두 수행하기 어려울 때가 있다.
* 가장 잘 알려진 두 가지 관리형 옵션은 구글 클라우드의 Cloud Composer와 Astronomer다. 월별 요금이 (훨씬 더) 많이 발생하지만 관리 편의성이 높아진다.
* 상황에 따라 구매 결정을 고려할 수 있다.
	* 자체 호스팅을 도와줄 수 있는 시스템 운영 팀이 있는가?
	* 관리형 서비스에 지출할 예산이 있는가?
	* 얼마나 많은 DAG과 작업이 파이프라인에 있는가?
	* 보안 및 개인 정보 보호 요구 사항은 무엇인가?

### 기타 오케스트레이션 프레임워크
* 에어플로우 외에 Luigi 및 Dagster와 같은 훌륭한 오케스트레이션 프레임워크가 있다. 
* 머신러닝 파이프라인 오케스트레이션에 맞춰진 Kubeflow Pipelines도 ML 커뮤니티에서 인기 있다.
* dbt도 탁월한 옵션이다.
