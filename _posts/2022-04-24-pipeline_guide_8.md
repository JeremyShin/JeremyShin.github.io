---
layout: post
title: 데이터 파이프라인 핵심 가이드 정리(10장) 
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


## 10. 파이프라인 성능 측정 및 모니터링
* 이 장에서는 데이터 수집 및 작업 성과 측정을 수행하기 위해 도움되는 사례를 공유한다.

### 중요 파이프라인 지표
* 파이프라인 전체에서 캡처해야 하는 데이터를 결저하기 전에 먼저 추적할 지표를 결정해야 한다.
* 지표 선택은 나와 이해관계자에게 중요한 것이 무엇인지 식별하는것에서 시작해야 한다. 예시를 보자.
	* 얼마나 많은 검증 테스트가 실행되고 얼마나 통과하는가?
	* 특정 DAG이 성공적으로 실행되는 빈도
	* week/month/year 단위의 파이프라인 런타임

### 데이터 웨어하우스 준비
* 파이프라인 성능을 모니터링하고 보고하기 위한 도구로 데이터 웨어하우스는 좋은 도구다. 파이프라인 각 단계에서 로그 데이터를 저장해보자!
* 8장에서 정의한 데이터 검증 프레임워크를 활용한다. (파이프라인 성능 측정에 필수적인 지표를 개발하는데 사용할 수 있다)

#### 데이터 인프라 스키마
* 먼저 에에프로우에서 DAG 실행 기록을 저장할 테이블이 필요하다. 
* 아래는 dag_run_history라는 테이블의 정의다. 데이터 수집 주에 데이터를 로드하는 스키마에 관계없이 DW에 생성되어야 한다.

		CREATE TABLE dag_run_history (
			id int,
			dag_id varchar(250),
			execution_date timestamp with time zone,
			state varchar(250),
			run_id varchar(250),
			external_trigger boolean,
			end_date timestamp with time zone,
			start_date timestamp with time zone
			);

* + 데이터 유효성에 대한 통찰력을 제공하는 것이 중요하다. 유혀성 검사 테스트 결과를 기록하는 테이블을 만들어보자.

		 CREATE TABLE validation_run_history (
			 script_1 varchar(255),
			 script_2 varchar(255),
			 comp_operator varchar(10),
			 test_result varchar(20),
			 test_run_at timestamp
			 );
* 이 장의 나머지 부분은 두 테이블에 로드된 데이터를 채우고 사용하는 논리를 구현한다.

### 성능 데이터 로깅 및 수집
* 첫 번째 테이블은 4, 5 장에서 배운 것처럼 데이터 수집 작업을 빌드하여 채워진다.
* 두 번째 테이블은 8장에서 처음 소개된 데이터 유효성 검사 응용 프로그램의 개선이 요구된다.

#### 에어플로우에서 DAG 실행 기록 수집
* 이 섹션에서는 54페이지의 PostgresSQL DB에서 데이터 추출에 정의된 모델을 따른다.
* 출력은 4장에서 설정한 s3 버킷에 업로드되는 dag_run_extract.csv 파일이다.
* pipeline.conf 파일에 섹션을 추가하자.

		[airflowdb_config]
		host = localhost
		port = 5432
		username = airflow
		password = pass1
		database = airflowdb

* 이 예제는 에어플로우 애플리케이션 데이터베이스에서 직접 DAG 실행 기록을 수집하고 있지만 이상적으로는 API 또는 다른 추상화 계층을 통해 수집한다. 에어플로우 2.0이 출시되며 확장적이고 안정적인 REST API를 사용할 수 있게 되었다. 

<script src="https://gist.github.com/JeremyShin/ffe8cbd662bc83bd48c870088d881b6e.js"></script>

* 추출이 완료되면 CSV 파일의 내용을 DW에 로드할 수 있다. DW 로드하는 코드는 중략한다.(책 204p 참고)


#### 데이터 유효성 검사기에 로깅 추가
* 8장에서 소개한 유효성 검사 테스트 결과 기록을 위한 log_result 함수를 validator.py 스크립트에 추가한다.

		def log_result(db_conn, script_1, script_2, comp_operator, result):
			m_query = """
				INSERT INTO validation_run_history(script_1, script_2, comp_operator, test_result, rest_run_at)
					VALUES(%s, %s, %s, %s, current_timestamp); """
			m_cursor = db_conn.cursor()
			m_cursor.execute(m_query(=, (script_1, script_2, comp_operator, result))
			db_conn.commit()
			m_cursor.close()
			db_conn.close()
			return

* 유효성 검사가 테스트될 때마다 validation_run_history 테이블에 기록된다.
* 아래의 예제는 테스트 데이터를 생성하기 위해 검증 테스트를 실행하는 코드다. (코드 설명은 중략한다.)

		TIP - 대규모 로깅
		대용량 로그 데이터를 생성하려는 경우 먼저 Splunk, SumoLogic 또는 ELK 스택과 같은 로그 분석 인프라로 
		라우팅하는 것을 고려해보는 것이 좋다. 로그 데이터가 이런 플랫폼으로 전송되면 나중에 데이터 웨어하우스에 
		대량으로 수집할 수 있다.

### 성능 데이터 변환
* 이제 파이프라인에서 주요 이벤트를 캡처하여 DW에 저장하므로 파이프라인 성능을 보고할 수 있다.
* 간단한 ETL 서비스를 만들어 실험해볼 수 있다.

#### DAG 성공률
* 데이터 세분성을 고려해야 한다. 여기서는 각 DAG 성공율을 일별로 측정하고 싶다고 가정한다.

#### 시간 경과에 따른 DAG 런타임 변경
* 시간이 지남에 따라 DAG 런차임 측정은 완료하는 데 시간이 더 오래 걸리는 DAG을 추적하는데 자주 사용되므로 DW의 데이터가 부실해질 위험이 있다. 
* 각 DAG의 일별 평균 실행 시간을 계산하는 테이블을 생성해보자.

#### 검증 테스트 볼륨 및 성공률
* 유효성 검사기 테스트이 결과를 일일 단위로 계산하고 저장하는 새로운 데이터 모델을 정의해보자.

### 성능 파이프라인 조정
* 위에서 설명했던 작업들을 기존 인프라를 사용하여 활용할 수도 있다.

#### DAG 성능
* 위 예제들의 파이프라인을 몇 가지 단계를 더 추가해보자.
	1. 에어플로우 데이터베이스에서 데이터를 추출한다.
	2. 에어플로우 추출에서 웨어하우스로 데이터를 로드한다.
	3. 에어플로우 히스토리를 변환한다.
	4. 데이터 유효성 검사 기록을 변환한다.

### 성능 투명성
* 프로덕션 파이프라인 및 데이터 검증 테스트 성능 측정 결과로 얻은 통찰력을 데이터 팀 및 이해 관계자와 공유하는 것은 매우 중요하다.
* 이해관계자와의 신뢰를 구축하고 팀에 대한 주인의식과 자부심을 형성할 수 있다.
* 아래는 이 장 전체에서 생성된 데이터와 통찰력을 활요하기 위한 몇 가지 팁이다.
	1. 시각화 도구 활용
	2. 요약된 지표를 정기적으로 공유
	3. 현재 가치 뿐만 아니라 추세를 관찰
	4. 트랜드에 대응

