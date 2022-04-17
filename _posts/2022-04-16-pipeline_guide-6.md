---
layout: post
title: 데이터 파이프라인 핵심 가이드 정리(8장) 
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


## 08. 파이프라인의 데이터 검증
* 데이터 자체 품질과 유효성 보장을 위해 데이터 검증에 투자해야 한다.
* 테스트되지 않은 데이터는 분석에 사용하기에 안전하지 않다고 가정하는 것이 좋다.
* 데이터 검증의 원칙을 설명한다.

### 일찍 그리고 자주 검증할 것
* 파이프라인 끝에서 데이터 품질 문제를 찾아 처음부터 다시 추적해야 하는 것은 최악의 시나리오다.
* 파이프라인 각 단계에서 데이터를 검증하면 현재 단계에서 근본 원인을 찾을 수 있다.

#### 소스 시스템 데이터 품질
* 데이터 수집 중 잘못된 데이터가 DW로 유입될 가능성이 높다. 아래의 경우를 살펴보자
	* 잘못된 데이터는 소스 시스템의 작동 자체에는 영향을 미치지 않을 수 있다.
	* 레코드의 연결이 끊어져도 소스 시스템이 정상적으로 작동하는 경우
	* 발견되지 않았거나 수정되지 않은 버그가 있는 경우

* 데이터 엔지니어는 DW에 로드된 데이터가 소스와 완벽히 일치하더라도 수집 중인 데이터에 품질 문제가 없다고 가정해서는 안된다.

#### 데이터 수집 위험
* 데이터 수집 프로세스 자체가 품질 문제를 야기하는 경우가 있다.
	* 수집 중 추출 또는 로드 단계에서 시스템 중단 또는 시간 초과
	* 증분 수집의 논리적 오류
	* 추출된 파일의 구문 분석(parsing) 문제

#### 데이터 분석가 검증 활성화
* DW에 로드된 데이터와 데이터 모델로 변환덴 데이터를 검증할 때 일반적으로 데이터 분석가가 자체 데이터 검증에 필요한 부분들을 잘 갖추고 있다. 각 데이터 모델 뿐 아니라 원본 데이터의 비즈니스 컨텍스트를 이해하는 사람들이다.
* 데이터 검증을 정의하고 실행하는 데 필요한 툴을 제공하는 것은 엔지니어에게 달려있다.

### 간단한 검증 프레임워크
* 파이썬으로 작성되고 SQL 기반 데이터 검증을 실행하도록 설계된 데이터 검증 프레임워크를 정의한다.

#### 유효성 검사기 프레임워크 코드
* 아래의 예제는 유효성 검사기에 대한 코드를 보여준다.
<script src="https://gist.github.com/JeremyShin/16e4bf572112563b1a171c83b7b41c4d.js"></script>
* 뒤에서 이 프레임워크가 실행되도록 설계된 데이터 검증 테스트 구조와 cli 및 airflow dag에서 테스트를 실행하는 방법을 설명한다.
#### 검증 테스트의 구조
* 이 프레임워크의 검증 테스트는 세 가지로 구성된다.
	* 단일 숫자 값을 생성하는 스크립트를 실행하는 SQL 파일
	* 단일 숫자 값을 생성하는 스크립트를 실행하는 두 번째 SQL 파일
	* SQL 스크립트에서 반환된 두 값을 비교하는데 사용되는 '비교 연산자'

#### 검증 테스트 실행
* 이전 섹션의 데이터 검증 테스트 예제를 사용하여 테스트를 실행할 수 있다.

		$ python validator.py order_count.sql order_full_count.sql equals
* 명령줄에 표시되지 않는 경우 종료 상태 코드다. 0이면 성공 -1일경우 실패다. 
* 다음 절에서 에어플로우 DAG에서 이 작업을 수행하는 방법을 보여준다. 실패시 슬랙 메세지, 이메일을 보내는 것과 같은 작업을 하는 것도 고려할 수 있다.

#### 에어플로우 DAG에서의 사용
* BashOperator를 활용한다.
* 이 예제의 경우 어떤 이유로 테이블1과 테이블2가 동일한지 확인하고 그렇지 않은 경우 작업에 실패하고 DAG에서 다운스트림 작업의 추가 실행을 중지한다고 가정한다.
* 아래의 작업을 elt_pipeline_sample.py DAG 정의에 추가한다.

		check_order_rowcount_task = BashOperator(
			task_id='check_order_rowcount',
			bash_command='set -e; python validator.py order_count.sql order_full_count.sql equals',
			dag=dag,
			)

* 다음엔 동일 파일에 있는 DAG 종속성 순서를 아래 코드로 재정의한다.

		extract_orders_task >> load_orders_task
		extract_customers_task >> load_customers_task
		load_orders_task >> check_order_rowcount_task
		check_order_rowcount_task >> revenue_model_task
		load_customers_task >> revenue_model_task

* check_order_rowcount_task가 실행되면 작업 정의에 따라 다음 Bash 명령어가 실행된다.

		set -e; python valicator.py order_count.sql order_full_count.sql equals

* set -e; 키워드를 사용하면 Bash가 0이 아닌 종료 상태 코드로 정의된 오류에 대해 스크립트 실행을 중지하도록 지시한다. 이 경우 에어플로우 작업이 실패하고 다운스트림 작업이 실행되지 않는다.
* 데이터 검증 테스트가 실패할 때 DAG의 추가 실행을 항상 중지할 필요는 없다. 아래의 섹션에서 이에 대해 논의해보자.


#### 파이프라인을 중단해야 할 때와 경고하고 계속해야 할 때
* 오류가 일어나서는 안되는 중요한 데이터의 경우 DAG을 중지하는 것이 올바른 접근법이다. 일반적으로 오래된 데이터가 잘못된 데이터보다 더 좋다! 
* 검증 테스트의 실패가 덜 심각하고 오히려 더 많은 정보를 제공하는 경우도 있다. 
* 오류를 발생시키고 파이프라인을 중지할지 또는 슬랙 채널에 경고를 보낼지에 대한 결정은 비즈니스 상황과 데이터 사용 사례를 기반으로 이루어져야한다. 그러기 위해서는 분석가, 엔지니어 모두 파이프라인에 데이터 검증 검사에 참여할 수 있는 권한이 있어야 한다.

#### 프레임워크의 확장
* 오픈소스나 상용 옵션을 고려하지 않고 이 프레임워크를 시작점으로 사용하기로 결정한 경우 고려할 몇 가지 개선 사항이 있다.
* 검증 프레임워크에서 필요한 사항은 테스트가 실패할 때 슬랙 채널이나 이메일로 알림을 보내는 것이다. 
* 슬랙 채널에 대한 수신 웹 훅을 만들어보자. 
<script src="https://gist.github.com/JeremyShin/c275e19686fa844dbf3412e2928c31d9.js"></script>
* 개선할 여지가 많지만 현재로서는 작업을 완료하기에 충분하다. 또한 테스트가 실패했다고 해서 반드시 파이프라인이 정지되어야 하는 것은 아니다. 

* 아래의 예제는 심각도를 나타내는 validator.py의 업데이트된 던더메서드의 main 블록을 보여준다. 심각도 수준이 halt(중지)인 스크립트가 실행되면 테스트 실패 시 종료 코드는 -1이 된다. 

<script src="https://gist.github.com/JeremyShin/422988e0ec42925de2abc3c60b5c9049.js"></script>
* 이 프레임워크를 확장하는 방법은 많지만 그 중 두 가지를 소개한다.
	* 애플리케이션을 통한 예외 처리
	* validator.py 단일 검사기의 실행으로 여러 테스트를 실행할 수 있는 기능

### 검증 테스트 예제
* 위의 예제들을 validator.py 코드에 추가했다고 가정하면 아래와 같은 명령줄에서 테스트를 실행할 수 있다.

		python validator.py order_count.sql order_full_count.sql equals warn
* 이 섹션에서는 파이프라인에서 데이터를 검증하는데 유용한 샘플 테스트를 정의하겠다.

#### 수집 후 중복된 레코드
* 중복 테스트는 가장 간단하고 일반적인 테스트다. 고려할 사항은 중복을 정의할 항목이다. 단일 ID 혹은 두 번째 열까지 확인할지 결정할 수 있다.

		WITH order_dups AS 
		(
			SELECT OrderId, COUNT(*)
			FROM Orders
			GROUP BY OrderId
			HAVING COUNT (*) > 1
		)
		
		SELECT COUNT(*)
		FROM order_dups;

* order_dup_zero.sql 파일을 만든다.

		SELECT 0

* 다음을 이용해 테스트를 수행한다.

		python validator.py order_dup.sql order_dup_zero.sql equals warn

#### 수집 후의 예기치 않은 행 개수
* 이번 예제는 데이터가 매일 수집된다고 가정 후 가장 최근(어제) 로드된 Orders 테이블의 레코드 수가 우리가 생각한 범위 내에 있는지 확인한다.
* 표준편차 계산을 사용하여 어제의 행 수가 Orders 테이블의 전체 과거 기록을 기반으로 90% 신뢰 수준 내에 있는지 확인해 보자. 
* 통계에서 이것은 정규 분포 곡선의 양쪽 아래를 살펴보고 있기 때문에 양측 검증으로 간주된다.
* z-점수 계산기로 신뢰구간 90%인 양측 검정을 확인하면 1.645이다.
* order_yesterday_zscore.sql 파일을 만들어보자.

		WITH orders_by_day AS (
			SELECT 
					CAST(OrderDate AS DATE) AS order_date,
					COUNT(*) AS order_count
			FROM Orders
			GROUP BY CAST(OrderDate AS DATE)
		),
		order_count_zscoure AS (
			SELECT
					order_date,
					order_count,
					(order_count AVG(order_count) over()) / (STDDEV(order_count) OVER()) AS z_score
			FROM orders_by_day
		)
		SELECT ABS(z_score) AS. twosided_score
		FROM order_count_zscore
		WHERE order_date = CAST(current_timestamp AS DATE) - interval '1 day');

* 아래는 확인 값을 반환한다. zscore_90_twosided.sql

		SELECT 1.645;

* 테스트를 쉘에서 실행해보자.

		python validator.py order_yesterday_zscore.sql zscore_90_twosided.sql greater_equals warn



#### 지표 값 변동

* 앞의 예제는 수집 후 데이터 유효성을 확인했다. 이 예제는 파이프라인 변환 단계에서 데이터 모델링 후 문제를 확인한다.
* 변환 단계는 잘못된 로직을 포함하여 행을 복제하거나 삭제할 수 있는 여러 가지 문제가 발생할 수 있다. 
* 파이프라인 끝에서 구축된 데이터 모델에 대해 검증 검사를 실행하는 것이 항상 좋은 방법이다. 
* 다음 세 가지 사항을 확인할 수 있다.
	* 지표 값이 특정 상한 및 하한 범위 내에 있는지 확인
	* 데이터 모델에서 행 개수 증가(또는 감소) 확인
	* 특정 지표 값에 예상치 못한 변동이 있는지 확인
* 여기서는 지표 값 변동 호가인을 위한 마지막 예제 하나만 제공하고자 한다.
* return_yesterday_zscore.sql 파일을 만들어보자.

		WITH revenue_by_day AS (
			SELECT
					CAST(OrderDate AS DATE) AS order_date,
					SUM(ordertotal) AS total_revenue
			FROM Orders
			GROUP BY CAST(OrderDate AS DATE)
		),
		daily_revenue_zscore AS (
			SELECT 
					order_date,
					total_revenue,
					(total_revenue - AVG(total_revenue) over()) / stddev(total_revenue) over()) AS z_score
			FROM revenue_by_day
		)
		SELECT ABS(z_score) AS twosided_score
		FROM daily_revenue_zscore
		WHERE order_date = CAST(current_timestamp AS DATE) - interval '1 day';

* zscore_90_twosided.sql 

		SELECT 1.645;

* 테스트 실행은 아래와 같다.

		python validator.py revenue_yesterday_zscore.sql zscore_90_twosided.sql greater_equals warn

* 이 예제에서는 현재 날짜를 기준으로 전월의 총 수익을 확인한다.

		WITH revenue_by_day AS (
			SELECT 
					date_part('month', order_date) AS order_month,
					SUM(ordertotal) AS total_revenue
			FROM Orders 
			WHERE order_date > date_trunc('month', current_timestamp - interval '12 month')
				AND order_date < date_trunc('month', current_timestamp)
			GROUP BY date_part('month', order_date)
		),
		daily_revenue_zscore AS (
			SELECT 
					order_month,
					total_revenue,
					(total_revenue - AVG(total_revenue) over()) / (STDDEV(total_revenue) over()) AS z_score
			FROM revenue_by_day
		)
		SELECT ABS(z_score) AS twosided_score
		FROM daily_revenue_zscore
		WHERE order_month = date_part('month', date_trunc('month', current_timestamp - interval '1 months'));

* 데이터 모델 지표값에 대한 검증은 비즈니스 컨텍스트를 잘 알고있는 분석가에게 맡기는 것이 최선이다. 


### 상용 및 오픈 소스 데이터 검증 프레임워크
* 일부 데이터 수집 도구는 행 수 변경, 예기치 않은 값 등을 확인하는 기능이 포함되어 있다. dbt 같은 데이터 변환 프레임워크에는 기능적으로 데이터 유효성 검사 및 테스트가 포함된다. 
* 데이터 검증을 위한 오픈소스 프레임워크도 있다. 머신러닝 파이프라인을 구축하고 텐서플로를 사용하는 경우 텐서플로 데이터 검증을 고려할 수 있다. 일반적인 검증은 Yahoo의 Validator라는 옵션도 사용 가능하다.

