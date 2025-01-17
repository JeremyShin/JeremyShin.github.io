---
layout: post
title: 데이터 파이프라인 핵심 가이드 정리(1-3장) 
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


## 01. 데이터 파이프라인 소개 
* 데이터의 진정한 가치는 데이터가 정제되어 소비자에게 전달된 후의 잠재력에 있고, 가치 사슬의 각 단계를 통해 데이터 전달을 위해 효율적인 파이프라인이 필요하다.
* 이 책에서는 데이터 파이프라인이 무엇인지 이야기하고 데이터 생태계에 어떻게 적용하는지 소개한다.
	* 일괄처리 vs 스트리밍 데이터 수집, 직접 구축 vs 제품 구매 등 파이프라인 구현 시 일반적인 고려사항을 다룬다.

### 데이터파이프라인이란?
* 다양한 소스에서 새로운 가치를 얻을 수 있는 대상으로 데이터를 옮기고 변환하는 일련의 과정
* 과정의 복잡성은 원본 데이터의 크기와 상태, 구조 및 분석 프로젝트의 요구사항에 따라 달라진다. 
* 실제 파이프라인은 추출, 가공, 유효성 검사를 포함한 여러 단계로 구성되며, 때로는 머신러닝 모델을 학습/실행하기도 한다. 


### 누가 파이프라인을 구축할까
* 조직에서 파악해야 할 데이터 소스가 폭발적으로 증가하고 있고 동시에 통찰력을 제공하는 데이터에 대한 수요도 높아지고 있다.
* 데이터 엔지니어는 분석 생태계를 뒷바침하는 파이프라인을 구축하고 유지관리하는 데 전문적인 역량을 갖추고 있다.
* 데이터 과학자, 분석가와 협력하여 요구사항을 확장 가능한 프로덕션 상태로 전환하는 데 도움을 준다.
* 데이터 유효성, 적시성을 보장해야 하고 이를 위해 테스트, 경고 및 비상 계획을 수립한다. 
* 아래는 데이터 엔지니어가 보유하고 있는 공통 기술에 대하여 설명한다. 

#### SQL과 데이터 웨어하우징 기초 
* 숙련된 데이터 엔지니어는 고성능의 SQL 작성 방법을 알고 데이터 웨어하우징 및 데이터 모델링의 기본 사항을 이해한다.

#### 파이썬 / 자바
* 현재는 파이썬 / 자바가 데이터 엔지니어링에서 우위를 점하고 있지만 고(Go)와 같은 신예도 등장하고 있다.

#### 분산컴퓨팅
* 큰 데이터를 신속하게 처리하기 위해 분산 컴퓨팅 플랫폼을 활용한다. 
* 분산 컴퓨팅 대표적인 예는 하둡 에코 시스템, 아차피 스파크 등이다.
* 데이터 엔지니어는 이런 프레임워크를 언제 어떻게 활용하는지 알아야 한다.

#### 기본 시스템 관리
* 리눅스 명령줄 사용에 능숙해야 하며 응용 프로그램 로그 분석, 크론 작업 예약, 방화벽, 보안 설정의 문제를 해결해야 한다. 
* 클라우드(AWS, Azure, Google Cloud 등) 환경에서도 파이프라인을 구축 배포를 진행한다.

### 왜 데이터 파이프라인을 구축할까?
* 단일 대시보드 또는 단일 지표가 여러 소스 시스템에서 생성되는 데이터에서 파생되는 경우는 아주 흔하다.
* 원본 데이터는 정리, 정형화, 정규화, 결합, 집계, 마스킹, 보안을 위해 정제된다. 

---
TIP - 데이터 파이프라인은 적절한 데이터가 제공되도록 보장하여 분석 조직이 가장 잘하는 통찰력 제공에 집중할 수 있게한다. (오래된 데이터 처리와 다양한 정보 출처, 데이터 준비 등)

### 어떻게 데이터 파이프라인을 구축할까?
* 이 책을 통해 파이프라인 구축을 위한 가장 인기 있는 솔루션, 프레임워크를 살펴보고 조직의 요구사항과 제약 조건에 따라 어떤 제품을 사용할지 결정하는 방법을 알아보자.
* 이 책의 모든 코드는 파이썬 / SQL로 작성되었다
* 데이터 엔지니어는 파이프라인을 구축하고 이를 안정적이고 안전하게 제시간에 제공하고 처리하는 인프라를 지원해야 한다. 이를 위해 파이프라인 모니터링, 유지 관리 및 확장에 대비해야 한다.

## 02. 최신 데이터 인프라
* 파이프라인 구축을 위한 제품, 설계를 결정하기 전에 최신 데이터 스택을 구성하는 요소를 이해할 필요가 있다.

### 데이터 소스의 다양성
* 대부분 조직은 수십개 이상의 데이터 소스가 있고, 이를 통해 분석 작업을 수행한다. 
* 최신 데이터 인프라의 핵심 구성 요소 5가지
	* 클라우드 데이터 웨어하우스와 데이터 레이크
	* 데이터 수집 도구
	* 워크플로 오케스트레이션 플렛폼
	* 모델링 도구 및 프레임워크
	* 데이터 소스의 다양성

### 소스 시스템 소유권
* 분석 팀은 일반적으로 조직이 구축하고 소유한 소스 시스템과 타사 도구 및 공급업체에서 데이터를 수집한다.
* 소스 시스템이 위치하는 곳이 어디인지 이해하는 것은 중요하다. 
	* 타사 데이터 소스에 액세스하기 위해서는 제한이 있을 수 있고, 엑세스 가능한 데이터와 세부 수준까지 사용자 환경에 맞추기는 더욱 어렵다.
* 시스템 구축시에 데이터 수집을 고려하여 설계해야 한다. 
	* 데이터 수집이 시스템에 부하를 주는 경우
	* 데이터를 점진적(incremental)으로 로드할 수 있는지
	* 제한적인 리소스를 얼마나 확보할 수 있는지 등을 고려해야 한다

### 수집 인터페이스 및 데이터 구조
* 데이터 엔지니어가 수집을 구축할 때 가장 먼저 알아볼 것은 **소스 데이터를 얻는 방법과 형식**이다. 
* 일반적인 데이터 인터페이스를 살펴보자
	* 애플리케이션 뒤에 있는 DB(Postgres, MySQL ..)
	* 시스템 상단의 추상화 계층 (REST API ..)
	* 스트림 처리 플랫폼(KAFKA .. )
	* 공유 네트워크 파일 시스템 또는 클라우드 스토리지 버킷(NFS, log, csv ..)
	* 데이터 웨어하우스 또는 데이터 레이크
	* HDFS, HBase 의 데이터
* 일반적인 데이터 구조도 알아보자
	* REST API의 JSON
	* MySQL
	* MySQL 내의 JSON
	* 반 정형화 된 로그 데이터
	* CSV, 고정 폭 형식(FWF) 및 기타 플랫 파일 형식
	* 플랫 파일의 JSON
	* Kafka의 스트림 출력
* 각 인터페이스와 데이터 구조는 각각의 도전 과제와 기회를 동시에 가지고 있다.
* 파이프라인에서 누락되거나 불완전한 데이터를 처리하는 방법은 상황에 따라 달라지며 데이터 유연성이 증가할수록 점점 더 많이 필요하다.

### 데이터 사이즈
* 크고 작은 데이터 세트를 함께 수집하고 모델링하는 것이 일반적이다. 파이프라인 각 단계를 설계시 데이터 사이즈를 고려해야 하지만 사이즈 크기가 가치의 유무를 결정하지 않는다.
* 데이터 파이프라인에 관련해서는 데이터를 크기보다는 스펙트럼 측면에서 생각하는 것이 좋다.

### 데이터 클렌징 작업과 유효성 검사
* 소스 데이터 품질도 중요하다. 지저분한 데이터의 특성을 보면 아래와 같다.
	* 중복되거나 모호한 레코드
	* 고립된 레코드
	* 불완전하거나 누락된 레코드
	* 텍스트 인코딩 오류
	* 일치하지 않는 형식
	* 레이블이 잘못되었거나 레이블이 지정되지 않은 데이터
* 데이터 클렌징과 유효성을 보장해주는 마법의 주문은 없다. 하지만 현재 데이터 생태계는 주요 특성과 접근 방식이 있다. 이 책에서 다뤄보자

#### 최악을 가정하고 최상을 기대하라
* 깨끗한 출력을 위해 데이터를 식별하고 정리하는 파이프라인을 구축하자

#### 가장 적합한 시스템에서 데이터를 정리하고 검증하라
* ETL, ELT 등의 방법이 있다. 데이터 클렌징과 검증 프로세스를 서두르지 말고 올바른 작업에 올바른 도구를 사용하자.

#### 자주 검증하라
* 파이프라인이 끝날 때 데이터를 검증하면 어디에서 문제가 발생헀는지 파악하기 어렵다. 단계 단계마다 검증해보자.

### 클라우드 데이터 웨어하우스 및 데이터 레이크
* 분석 및 데이터 웨어하우징 환경은 클라우드 환경(AWS, GCP, Azure)의 등장에 따라 급격하게 변화했다.
	* 클라우드 등장으로 구축 및 배포가 쉬워짐. 더이상 iT 대규모 초기 비용에 대한 예산을 기다릴 필요가 없음 
	* 클라우드 공급업체에서 유지보수를 진행
	* 지속적인 클라우드 내 스토리지 비용 감소
	* 확장성이 뛰어난 열 기반 DB(Redshift, Snowflake, Big query) 등장
* 이런 변화는 DW를 변화시키고 데이터 레이크 개념을 도입했다. 두 가지 개념을 간략히 정의해보자.
	* 데이터 웨어하우스
		* 분석 활동 지원을 위해 서로 다른 시스템의 데이터가 모델링되어 저장되는 데이터베이스
	* 데이터 레이크
		* 다양한 데이터 유형뿐만 아니라 대량의 데이터가 포함될 가능성이 높다. 
		* 표준 DB 처럼 정형화 된 데이터를 쿼리하는데 최적화되어 있지는 않다. 

### 데이터 수집 도구
* 한 시스템에서 다른 시스템으로 데이터를 수집할 필요성은 거의 모든 파이프라인에서 공통적이다.
* 이 책에서는 아래와 같은 일반적인 도구 및 프레임워크에 대해서 설명한다
	* Singer
	* Stitch
	* Fivetran
* 이런 툴이 보급되어 있음에도 불구하고 일부 팀은 아래와 같은 이유로 데이터 수집을 자체 프레임워크를 직접 개발하기도 한다.
	* 비용
	* 직접 구축을 선호하는 분위기
	* 접적, 보안 위험에 대한 우려
* 상용 제품의 가치를 어디에 두느냐에 따라 상용제품의 설계/적용의 범위가 달라진다.

### 데이터 변환 및 모델링 도구
* 데이터 모델링과 데이터 변환이라는 용어는 종종 같은 의미로 사용되는 경우가 많다. 각각의 용어를 살펴보자.
* 데이터 변환
	* ETL 또는 ELT 프로세스에서 T (Transformation)에 해당하는 광범위한 용어다. 
	* 변환은 지정된 타임스탬프를 변경하는 간단한 작업일 수도 있고 비즈니스 로직을 통해 집계/필터링 된 원본 열을 바탕으로 새 지표를 생성하는 복잡한 작업일 수 있다.
* 데이터 모델링
	* 보다 구체적인 데이터 변환 유형이다. 데이터 모델은 데이터 분석을 위해 데이터를 이해하고 최적화된 형식으로 정형화하고 정의한다. 
	* 데이터 모델은 일반적으로 데이터 웨어하우스에서 하나 이상의 테이블로 표시된다. 
* 최신 데이터 인프라에는 데이터 변환 관련된 여러 가지 방법론과 도구가 있다. 
	* 데이터 수집 도구에는 변환 기능을 제공하지만 제한된 기능으로서 제공한다.
	* 복잡한 데이터 변환 및 데이터 모델링을 위해서는 dbt(9장 참고)와 같이 특별히 설계된 도구와 프레임워크를 찾는 것이 바람직하다. 
	* 또한 데이터 엔지니어와 분석가에게 익숙한 언어(SQL, Python 등)으로 작성 가능하다.
* SQL에서 데이터 모델 구축을 지원하는 변환 프로그램을 작성하는 것이 바람직하다. 
	* 다수의 사용자가 참여 가능하고 개발 프로세스에 처음부터 끝까지 관여할 수 있다. 

### 워크플로 오케스트레이션 플랫폼
* 조직의 데이터 파이프라인 복잡성과 수가 증가하면 데이터 인프라에 데이터 워크플로 오케스트레이션 플렛폼을 도입해야 한다. 
* apache airflow, Fuigi, aws glue와 같은 플랫폼은 좀 더 일반적인 용도로 설계되어 다양한 데이터 파이프라인에 사용된다.
* Kubeflow Pipeline과 같은 플랫폼들은 구체적인 사용 사례와 플랫폼을 위해 설계되었다. 
	* (Kubeflow Pipeline의 경우 Docker 컨테이너에 구축된 머신러닝 워크플로)

### 방향성 비순환 그래프
* 파이프라인 단계는 항상 **방향성**을 가진다. - 모든 종속 작업이 완료되어야만 그 다음 작업이 실행된다. 
* 파이프라인 그래프는 **비순환** 그래프여야 한다. - 작업은 돌아갈 수 없기 때문에 다음으로만 갈 수 있다.
* DAG은 작업들의 집합을 나타내며 실제 수행하는 내용은 다른 시스템에 있다. 예를 들어보자.
	1. 관계형 DB에서 쿼리하고 결과를 CSV 파일에 저장하는 SQL 스크립트를 실행
	2. 앞서 저장된 CSV 파일을 로드하여 형태를 변경, 정렬하여 새로운 CSV로 저장
	3. 두 번째 작업에서 생성된 CSV를 Snowflake DW로 로드하기 위해 SQL에서 COPY 명령어 실행

### 데이터 인프라 커스터마이징
* 제약 조건(비용, 엔지니어링 리소스, 보안 및 법적 리스크 허용 범위)과 그에 따른 트레이드 오프를 이해해야 한다.
* 이 책에서 이런 내용을 언급하고 제품 또는 도구 선택시 중요한 의사 결정 사항을 설명한다.

## 03. 일반적인 데이터 파이프라인 패턴
* 파이프라인은 각자 다른 목표와 제약 조건을 갖는다. 
  (배치 or 실시간 여부나 시각화 or 머신러닝 모델 입력값으로 사용할지 등..)
* 이 장에서는 다양한 사용 사례로 확장 가능한 몇 가지 공통 패턴을 소개한다. 후속 장은 이를 기반으로 파이프라인을 구현해본다. 

#### ETL과 ELT
* 대표적인 데이터 파이프라인 패턴이다. 
* 두 패턴 모두 DW에 데이터를 공급하고 분석가나 보고 도구가 이를 유용하게 쓸 수 있게 하는 데이터 처리 접근 방식이다.
* 추출(extract) : 다양한 소스에서 데이터를 수집하는 단계
* 로드(load) : 원본 데이터(ELT) 혹은 변환이 완료된 데이터(ETL)를 최종 대상(DW, 데이터레이크 등)으로 가져오는 단계
* 변환(transfrom) : 분석가, 시각화 도구, 파이프라인이 제공하는 사용 사례에 각 소스 시스템의 원본 데이터를 결합하고 형식을 지정하는 단계

---
TIP - 추출과 로드의 분리
* 추출 및 로드 단계의 조합을 종종 데이터 수집이라고 부른다. 그러나 파이프라인 설계시 서로 다른 시스템과 인프라에서 ETL(혹은 ELT)해야 하는 복잡성이 있기 때문에 두 단계를 별개로 고려하는 것이 좋다.


#### ETL을 넘어선 ELT의 등장
* 과거에는 ETL이 거의 유일한 표준
	* 방대한 양의 원본 데이터를 로드, 변환하는데 필요한 자원(스토리지, 컴퓨팅 자원 등)이 부족함	
	* 행 기반 DB 사용
* 클라우드의 등장으로 ELT의 개념이 등장
	* + 열 기반 DB를 기반으로 로드 후 변환 가능 
	   (I/O 효율성, 데이터 압축, 병렬 노드에 데이터 및 쿼리를 분산하는 기능)
* 아래는 열 기반 스토리지에 대한 설명 ([원본링크](https://softwareengineeringdaily.com/2017/01/13/columnar-data-apache-arrow-and-parquet-with-julien-le-dem-and-jacques-nadeau/))
![열기반 스토리지](https://github.com/JeremyShin/jeremyshin.github.io/blob/master/images/column-oriented-database1.jpg?raw=true)

#### EtLT 하위 패턴
* EtLt 하위 패턴의 몇가지 예를 보자
	* 테이블에서 레코드 중복 제거
	* URL 파라미터를 개별 구성요소로 구문 분석
	* 민감한 데이터 마스킹 또는 난독화
* 위의 유형의 변환은 비즈니스 로직과 분리되거나 민감한 데이터를 마스킹하는 것과 같이 법적인 이유료 파이프라인 초기에 필요한 경우도 있다. 
* 대부분의 최신 데이터 웨어하우스는 가장 효율적인 방법으로 데이터를 로드한다. 
* 아래의 나머지 ELT 관련 패턴은 EtLT 하위 패턴도 포함되도록 가정할 수 있다. 

#### 데이터 분석을 위한 ELT
* ELT는 데이터 분석 파이프라인에 있어 가장 일반적이고 최적의 패턴이 되었다. 
* 이는 추출과 로드는 데이터 엔지니어가 담당하고 변환은 데이터 분석가가 담당하는 방법이다. 이유를 살펴보자
	* 열 기반 DB는 대용량 데이터를 처리하는데 적합 
	* 분석가는 일반적으로 SQL에 능통
	* 엔지니어는 추출/로드에 집중하고 분석가는 보고 및 분석 용도의 데이터를 변환 가능
	* ETL 패턴은 전체 파이프라인에 걸쳐 엔지니어가 필요하기 떄문에 명확한 분리가 불가능함
	* ELT 패턴은 추출 및 로드 프로세스를 구축 시 분석가가 수행 작업을 검증해야 하는 필요성을 줄여준다. 
	* 변환 단계를 뒤로 넘김으로서 분석가에게 더 많은 옵션과 유연성을 제공


TIP - ELT 등장으로 분석가는 엔지니어에 의해 차단되지 않고 데이터로부터 가치를 제공할 수 있는 자율성과 권한을 갖게됨. 이러한 이유로 분석 엔지니어와 같은 새로운 직함도 생겨남 

#### 데이터 과학을 위한 ELT
* 데이터 과학 팀을 위해 구축된 파이프라인은 DW에서 데이터 분석을 위해 구축된 파이프라인과 비슷하다.
* 그러나 데이터 과학자는 분석가와 다른 요구사항이 있다.
* 일반적으로 데이터 과학자는 데이터 분석가보다 세분화된 (때로는 원본) 데이터에 액세스해야 한다.
	* 데이터 분석가 : 지표 생성, 대시보드 강화하는 데이터 모델 구축
	* 데이터 과학자 : 데이터 탐색 및 예측 모델 구축
* 데이터 과학자는 추출-로드 중에 획득한 많은 데이터를 분기하여 사용할 가능성이 높다.

#### 데이터 제품 및 머신러닝을 위한 ELT
* 데이터 제품을 강화하는 몇 가지 일반적인 예는 다음과 같다.
	* 비디오 스트리밍 홈 화면을 구동하는 콘텐츠 추천 엔진
	* 전자상거래 웹사이트의 개인화된 검색 엔진
	* 사용자가 생성한 레스토랑 리뷰에 대한 감성 분석을 수행하는 애플리케이션
* 이러한 제품은 하나 이상의 ML 모델에 의해 구동될 가능성이 높다. 다양한 소스의 데이터가 필요하며 일정 수준의 변환이 필요하다. 

머신러닝 파이프라인의 단계
* 시작 부분은 ELT와 유사한 패턴을 따른다. 
* 차이점
	* 데이터 분석 : 분석은 변환 단계에서 데이터를 데이터 모델로 변환하는데 중점을 둠
	* 머신러닝 : DW, 데이터 레이크에 로드된 이후 머신러닝 모델을 빌드하고 업데이트 하는 것과 관련된 여러 단계가 있음
* 머신러닝 개발과 관련된 파이프라인 단계를 보자
	* 데이터 수집
		* 책 4, 5 장의 수집 프로세스를 따른다. 다만 머신러닝 파이프라인은 한 가지 추가로 고려할 사항이 있다.
		* 수집 데이터가 머신러닝 모델이 나중에 학습 또는 검증을 위해 참조 가능하도록 버전이 지정되는지 확인해야 한다.
		* 데이터세트 버전 관리를 위한 자세한 도구와 접근 방식은 아래를 참고하자
			* 책을 추천한다
			* 머신러닝 파이프라인 구축(Building Machhine Learning Pipelines), O'Reilly, 2020
			* 핸즈온 머신러닝 (한빛미디어, 2018)
			* 파이썬 라이브러리를 활용한 머신러닝 (한빛미디어, 2018)
	* 데이터 전처리
		* 머신러닝 개발에 사용 가능하도록 데이터를 정리하고 모델에 사용할 준비를 하는 단계
	* 모델 교육
		* 새 데이터를 수집/전처리 후 머신러닝 모델을 다시 학습한다.
	* 모델 배포
		* 가장 어려운 부분일 수 있다. 데이터세트 및 학습 모델의 버전 관리도 필요하다. 운영 수준에 도달하기 위해 관련자들끼리 조정할 일이 많다. 잘 설계된 파이프라인은 중요한 역할을 한다.

---
TIP - 파이프라인에서 데이터 검증은 필수적이며 파이프라인 전반에 걸쳐 이루어진다. 분석가용 파이프라인 검증은 데이터 수집(추출-로드), 모델링(변환) 후에 발생하고 머신러닝 파이프라인에서는 수집된 데이터의 유효성 검사도 중요하다

#### 파이프라인에 피드백 통합
* 좋은 머신러닝 파이프라인은 모델 개선을 위한 피드백 수집도 포함된다. 
* 피드백 관련 모든 정보를 DW로 다시 수집하고 학습데이터, 미래 모델, 실험 및 분석에 활용하고 향후 버전에 통합 할 수 있다.
* 수집된 데이터는 분석가가 다시 분석 가능하다. 분석가는 종종 모델 효율성을 측정하고 조직에 모델 주요 지표를 표시하기 위한 대시보드 구축 임무를 맡게 된다. 이해 관계자는 대시보드를 확인하여 다양한 모델이 비즈니스와 고객에게 얼마나 효과적인지 이해할 수 있다.
