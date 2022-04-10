
---
layout: post
title: 데이터 파이프라인 핵심 가이드 정리(6장) 
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


## 06. 데이터 변환하기
* 데이터 변환에는 비문맥적 데이터 조작과 비즈니스 컨텍스트 및 논리를 염두에 둔 데이터 모델링이 포함될 수 있다.

### 비문맥적 변환
* EtLT의 소문자 t는 아래와 같은 비문맥 데이터 변환을 나타낸다.
	* 테이블의 중복 레코드 제거
	* URL 매개변수를 개별 구성요소로 구문 분석
	* ...
* 아래는 ETL에 비해 EtLT가 적절한 경우를 설명한다.

#### 테이블에서 레코드 중복 제거
* DW에 수집된 데이터 테이블에 중복 레코드가 존재할 수 있다. 이유는 아래와 같이 여러 가지다.
	* 증분 데이터 수집에서 수집 시간이 겹쳐서 이미 수집된 레코드를 다시 가져오는 경우
	* 원본 시스템에 중복 레코드가 실수로 생성된 경우
	* 나중에 채워진(backfilled) 데이터가 로드된 데이터와 겹치는 경우
* 중복레코드를 확인하고 제거하는 것은 SQL이 가장 좋다.

```SQL
	CREATE TABLE Orders (
		orderId int,
		orderStatus varchar(30),
		LastUpdated timestamp
	);
	
	INSERT INTO Orders VALUES(1, "Backordered", "2020-06-01");
	INSERT INTO Orders VALUES(1, "Shipped", "2020-06-09");
	INSERT INTO Orders VALUES(2, "Shipped", "2020-07-11");
	INSERT INTO Orders VALUES(1, "Shipped", "2020-06-09");
	INSERT INTO Orders VALUES(3, "Shipped", "2020-07-12");
```

* 중복 레코드 확인은 group by, having을 활용하자.
```SQL
	SELECT orderId, 
			orderStatus,
			LastUpdated,
			COUNT(*) AS dup_count
	FROM Orders
	GROUP BY orderId, orderStatus, LastUpdated
	HAVING COUNT(*) > 1;
```

* 중복을 찾았으니 아래의 2가지 방법으로 제거해보자
	* 쿼리 시퀀스 사용 - 이 방법은 작업중에 테이블이 비어있게된다.
		```SQL
		CREATE TABLE distinct_orders AS 
			SELECT DISTINCT OrderId,
							OrdersStatus,
							LastUpdated
			FROM ORDERS;
		
		TRUNCATE TABLE Orders;
		INSERT INTO Orders SELECT * FROM distinct_orders;
		DROP TABLE distinct_orders;
		```
	* 윈도우 기능 사용 
```SQL
	SELECT OrderId,
			OrderStatus,
			LastUpdated,
			ROW_NUMBER() OVER(PARTITION BY OrderId,
								OrderStatus,
								LastUpdated)
			AS dup_count
	FROM Orders;
```

#### URL 파싱
* URL에는 변환단계에서 구분분석 및 DB 테이블에 컬럼을 구분해 적재할 구성요소가 많다.
* 아래의 의미있는 구성 요소 6가지를 살펴보자.
	* 도메인 : www.mydomain.com
	* URL 경로 : /page-home
	* utm_content 매개변수 값 
	* utm_medium 매개변수 값
	* utm_source 매개변수 값
	* utm_campaign 매개변수 값
* UTM 매개변수 : 마케팅 및 광고 캠페인을 추적하는데 사용되는 URL 매개변수이며, 대부분 플렛폼과 조직에 공통적이다.
* URL 구문 분석은 python, sql로 모두 가능하다.
	* python 스크립트를  확인해보자.
	<script src="https://gist.github.com/JeremyShin/b33ae58fccf1e5cf1a6497d590d7bb27.js"></script>



### 언제 변환할 것인가, 수집 중 혹은 수집 후?
* EtLT를 고려해야하는 몇 가지 이유를 보자.
	* SQL 이외의 언어를 사용하여 수행하는 것이 가장 쉽다.
	* 변환은 데이터 품질 문제를 해결한다.
* 비즈니스 로직과 관련된 변화의 경우 데이터 수집과 별도로 변환을 분리하는 것이 좋다. 이런 유형의 변환을 데이터 모델링이라고 한다.

### 데이터 모델링 기초
* 이 섹션에서는 ELT 페턴 변환 단계에서 비즈니스 컨텍스트가 고려한다. 

#### 주요 데이터 모델링 용어
* 이 섹션에서 데이터 모델 용어를 사용할때는 SW의  개별 SQL 테이블을 가리키는 것이다.
* 데이터 모델에서는 아래 두 가지 속성에 중점을 둔다.
	* 측정 - 측정하고 싶은 것
	* 속성 - 그룹화하려는 항목
* 세분성(granularity)은 데이터 모델에 저장된 세부 정보 수준을 말한다. - 시간마다 정보 제공해야 한다면 ..?!

#### 완전히 새로 고침 된 데이터 모델링
* 소스 데이터 저장소의 최신 상태가  포함된 테이블을 다루게 된다.
* 아래 질문을 답하기 위해 데이터 모델을 만들 수 있는지 생각해보자
	* 특정 월에  특정 국가에서 발생한 주문으로 인해 발생한 수익은 얼마인가?
	* 주어진 날에 얼마나 많은 주문이 접수 되었는가?
* 팩트와 차원(fact and dimention)

#### 완전히 새로 고침 된 데이터의 차원을 천천히 변경
* 데이터가 전체 새로고침되는 경우 종종 과거 변경 사항을 추적하기 위해 고급 데이터 모델링 개념이 사용된다.
* 완전 새로고침된 데이터를 사용하면 각 수집 사이에 전체 기록을 유지하고 변경사항을 스스로 추적해야 한다.
* 이를 SCD(slowly changing dimension)이라고 한다.
* 쿼리로 처리할 수 있다.(책 참조)


#### 증분 수집된 데이터 모델링
* 위의 질문 (특정 월에 .. , 주어진 날에 ..) 을 기반으로 생각해보자.
* 이 경우 데이터 모델 구축 전 테이블 레코드에 대한 변경 사항을 처리하는 방법을 결정해야 한다.
* 선택은 비즈니스 사례에 필요한 로직을 기반으로 해야하지만 구현은 약간 다르다.
* 특정 월에 특정 국가에서 이뤄진 주문에서 얼만큼의 수익이 발생했는지에 대한 질문에 답할때 영국에서 처리되었지만, 유저가 미국에 살고있을 경우도 있다.
* 주문 당시 고객이 살았던 국가에 주문을 할당하려면 로직 변경이 필요하다.(CTE common table expression)의 가장 최근 레코드를 찾는 대신 각 고객이 주문한 시간 또는 그 이전에 업데이트된 가장 최근 레코드를 찾는다.

#### 추가 전용(Append-only) 데이터 모델링
* append-only 데이터는 변경할 수 없는 데이터다.
* 이런 테이블의 각 레코드는 변경되지 않는 일종의 이벤트다. append-only 데이터 모델링은 새로 고친 데이터 모델링과 유사하다.
*  아래 질문에 답하도록 데이터 모델을 정의해보자.
	* 하루에 사이트의 각 url path에 대한 pageview는 몇번인가?
	* 각 국가의 고객이 매일 생성하는 pageview는 몇개인가?
* 데이터 모델의 세분성은 일단위다. 세 가지 속성이 필요하다.
	* 날짜
	* url path
	* 국가
* 요구되는 측정 항목은 아래와 같다.
	* pageview
* pageview를 설계하고 매일 모델을 업데이트하려면 두 가지 옵션이 있다.
	* pageview 모델을 완전히 새로고침한다.
	* 새 레코드만 증분으로 새로고침한다.

#### 변경 캡처 데이터 모델링
* 4장에서 CDC를 통해 수집된 데이터는 DW에 특정 방식으로 저장된다는 점을 상기해보자.
* 저장된 데이터를 모델링하는 방법은 데이터 모델이 답해야 하는 질문에 따라 다르다.
* CDC에서 수집한 데이터의 또 다른 일반적인 용도는 변경 자체를 이해하는 것이다.
* 이전 섹션과 마찬가지로 CDC를 통해 수집된 데이터가 완전히 새로 로드되지 않고 증분 로드되면 잠재적인 성능 향상을 얻을 수 있다.

