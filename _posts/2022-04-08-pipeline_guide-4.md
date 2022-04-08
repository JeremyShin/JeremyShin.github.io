---
layout: post
title: 데이터 파이프라인 핵심 가이드 정리(5장) 
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
	
	INSERT INTO Orders VALUES(1, "Backordered', '2020-06-01');
	INSERT INTO Orders VALUES(1, "Shipped', '2020-06-09');
	INSERT INTO Orders VALUES(2, "Shipped', '2020-07-11');
	INSERT INTO Orders VALUES(1, "Shipped', '2020-06-09');
	INSERT INTO Orders VALUES(3, "Shipped', '2020-07-12');
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
	* 쿼리 시퀀스 사용
	* 

#### URL 파싱

### 언제 변환할 것인가, 수집 중 혹은 수집 후?

### 데이터 모델링 기초

#### 주요 데이터 모델링 용어

#### 완전히 새로 고침 된 데이터 모델링

#### 완전히 새로 고침 된 데이터의 차원을 천천히 변경

#### 증분 수집된 데이터 모델링

#### 추가 전용(Append-only) 데이터 모델링

#### 변경 캡처 데이터 모델링


