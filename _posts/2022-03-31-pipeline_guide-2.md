---
layout: post
title: 데이터 파이프라인 핵심 가이드 정리(4장) 
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


## 04. 데이터 수집: 데이터 추출
* ELT 패턴은 데이터 분석, 데이터 사이언스, 데이터 제품을 위한 이상적인 설계다. 
* ELT 패턴의 처음 두 단계인 데이터 추출과 로드를 모두 **데이터 수집**이라고 이야기 한다.
* 개발 환경과 인프라를 설정하는 방법에 대해 설명하고 다양한 소스 시스템에서 데이터를 추출하는 방법을 설명한다.

### 파이썬 환경 설정
* 코드 예제를 실행하려면 파이썬 3버전을 실행하는 가상 시스템이 필요하다. 몇개의 라이브러리를 설치해야 한다.
* 이 장에 사용된 라이브러리를 설치하기 전에 설치할 가상 환경을 만드는 것이 가장 좋다.
* virtualenv는 다양한 프로젝트 및 애플리케이션의 파이썬 라이브러리를 관리하는데 유용하다.
* 아래는 가상환경을 생성하는 코드이다.
    $ python -m venv venv
* 이제 가상환경을 활성화 해보자
    $ source env/bin/activate
* pip을 사용하여 코드 예제에 사용되는 라이브러리를 설치하자. 설치할 첫 번째 라이브러리는 configparser이다.
     (env) $ pip install configparser
* 파이썬 스크립트와 동일한 디렉터리에 pipeline.conf 파일을 만든다. 일단 파일을 비워두자.
    (env) $touch pipeline.conf 
---
TIP - Git Repo에 구성 파일을 추가하지 말 것!
* 자격증명 및 연결 정보는 구성 파일에 저장되므로 Git 레포지토리에 추가하면 안 된다.
  (S3 버킷, 소스시스템, DW 엑세스할 수 있는 안전한 시스템에만 저장한다.)
* Git 레포지토리에서 .conf 파일을 제외할 수 있도록 .gitignore 파일에 *.conf 내용을 추가하자.

### 클라우드 파일 스토리지 설정
* 이 장의 각 예제는 S3 버킷을 사용한다.
* S3 버킷에 대한 적절한 액세스 제어 설정은 사용중인 DW에 따라 달라진다.
* 일반적으로 AWS IAM 역할을 사용하는 것이 가장 좋다.
* 버킷을 만들 때는 비공개로 유지하는 등의 기본 설정을 사용하는 것이 좋다.
* 파이프라인 일반적인 패턴을 소개한다. 각 추출 예제는 지정된 소스시스템에서 데이터를 추출하고 S3 버킷에 저장하고, 5장에서 DW로 로드시에는 해당 데이터를 S3 버킷에서 대상으로 로드한다.
* Azure의 Azure Storage, GCP의 Google Cloud Storage(GCS)가 AWS S3와 비슷한 공용 클라우드에 해당한다.
* 예제는 로컬, 온프레미스 스토리지를 사용하도록 각 예제를 수정할 수 있다.
* boto3를 설치하고 IAM을 설정한다. (책 33쪽 참고)
* 마지막으로 pipeline.conf 파일 안에 aws_boto_credentials 정보를 추가하자 
	* access_key, secret_key, bucket_name, account_id

### MySQL 데이터베이스에서 데이터 추출
* MySQL 추출은 두 가지 방법으로 수행 가능
	* SQL 활용 전체 또는 증분 추출
	* 이진 로그(binlog) 복제
* SQL 활용 데이터 추출은 간단하지만 자주 변경되는 대규모 데이터세트에는 확장성이 떨어진다.
* SQL 활용 전체 추출과 증분 추출도 트레이드오프가 존재한다.(다음 섹션에서 논의하자)
* 이진 로그 복제는 구현이 더 복잡하지만 원본 테이블의 변경되는 데이터 볼륨이 크거나 MySQL 소스에서 데이터를 더 자주 수집해야 하는 경우에 더 적합하다.
---
NOTE- 이진 로그 복제는 **스트리밍 데이터 수집**을 수행하는 하나의 경로이기도 하다.

* 이 섹션은 MySQL을 소스로 데이터 추출해야 하는 독자와 관련이 있다. 이 책은 로컬에 MySQL을 만들거나 RDS 활용을 추천한다고 한다.
* Orders 테이블을 만들고 데이터를 추가해보자.
<script src="https://gist.github.com/JeremyShin/4abf5edbd11e73033c698ff6b8f81a63.js"></script>

### 전체 또는 증분 MySQL 테이블 추출
* **전체 추출**에서는 추출 작업을 실행할 때마다 테이블의 모든 레코드가 추출된다.
* 쉬운 방법이지만 대용량 테이블의 경우 실행 시간이 오래 걸릴 수 있다.
* **증분 추출**에서는 추출 작업의 마지막 실행 이후 변경되거나 추가된 원본 테이블의 레코드만 추출된다.
* 마지막 추출의 타임스탬프는 DW의 추출 작업 로그 테이블에 저장하거나 DW의 대상 테이블에서 마지막 업데이트 열의 최대 타임스탬프를 쿼리하여 검색할 수 있다.
* 가상 Orders 테이블을 예로 들면 원본 MySQL DB에서 실행되는 SQL 쿼리는 아래와 같다.
<script src="https://gist.github.com/JeremyShin/5a304f359c6bda3cf8201427cd9557af.js"></script>

NOTE- 변경할 수 없는 데이터가 표함된 테이블의 경우에는 LastUpdated 열 대신 레코드가 생성된 시간에 대한 타임스탬프를 사용할 수 있다.

* {{last_extraction_run}} 변수는 추출 작업의 최근 실행 시간을 나타내는 타임스탬프다. 일반적으로 DW 대상 테이블에서 쿼리된다. 이 경우 DW =에서 SQL을 사용하여 {{ last_extraction_run }}에 사용된 결괏값을 확인한다.
<script src="https://gist.github.com/JeremyShin/28a190044efd8740caf9f18add629989.js"></script>

TIP - 마지막으로 업데이트된 날짜 캐시
Orders 테이블이 큰 경우 추출 작업을 빠르게 수행할 수 있도록 로그 테이블에 마지막으로 업데이트된 레코드의 값을 저장할 수 있다.대상 테이블에 MAX(LastUpdated) 값을 DW에 저장해야 한다. 
* 증분 추출이 최적의 성능에 이상적임 하지만 단점과 이유가 있으니 살펴보자
	* 삭제된 행은 캡처되지 않음
	* 원본 테이블에서 마지막 업데이트 시간에 대해 신뢰 가능해야 함
* MySQL DB에서 전체 및 증분 추출은 모두 DB에서 실행되지만 파이썬 스크립트에 의해 트리거되는 SQL 쿼리를 사용하여 구현 가능하다
* 관련 설정은 책을 참고하자
* MySQL 관련 DB 연결 초기화 코드 공유 및 csv 파일을 s3 버킷에 업로드하는 코드를 확인해보자

<script src="https://gist.github.com/JeremyShin/cdf8f648dbca8c0ebc06228c5a06202a.js"></script>

* 스크립트가 실행되면 Orders 테이블의 전체 내용이 DW / 데이터 스토어에 로드되기를 기다리는 S3 버킷의 CSV 파일에 포함된다.
* 데이터를 증분 추출하려면 스크립트를 수정해야 한다. extract_mysql_full.py 파일을 복사하여 extract_mysql_incremental.py 사본에서 시작하는 것을 추천한다
* Redshift 클러스터와 상호작용하려면 psycopg2 라이브러리를 설치한다. (pip install psycopy2)
* 다음은 Redshipt 클러스터에 연결하고 쿼리하여 Orders 테이블에서 MAX(LastUpdated) 값을 가져오는 코드다.
<script src="https://gist.github.com/JeremyShin/45c97f0163a4d186f231248c700ee214.js"></script>

* last_update_warehouse에 저장된 값을 사용하여 MySQL DB에서 실행되는 추출 쿼리를 수정한다.
* 다음은 MySQL DB에서 SQL 쿼리를 실행해주는 업데이트된 코드다.
<script src="https://gist.github.com/JeremyShin/99a35d24531ad34c464cd313ce97873d.js"></script>

* 증분 추출을 위한 전체 extract_mysql_incremental.py 스크립트 (last_updated 값에 대해 redseift 클러스터를 사용)는 다음과 같다.
<script src="https://gist.github.com/JeremyShin/8804eb6b718705b009edc51f6285147c.js"></script>

i경고 - SQL 추출은 DB에 부담을 주며 프로덕션 쿼리 실행도 차단할 수 있다. 복제본 사용을 고려하라.

### MySQL 데이터의 이진 로그 복제

<<<<<<< HEAD
* 구현하기는 더 복잡하지만, 대용량 데이터 수집이 필요한 경우 MySQL 이진 로그를 사용하는 것이 더 효과적이다.
* 이진 로그는 CDC의 한 형태다. 
* MySQL 이진 로그는 DB에서 수행된 모든 작업에 대한 기록을 보관하는 로그다. 
	* 테이블 생성, 수정, insert, update, delete 작업도 기록된다.
	* 이진 로그의 원래 목적은 다른 MySQL 인스턴스로 데이터를 복제하기 위한 것이지만, DW 수집 용도로도 매력적이다.

TIP - 사전 구축된 프레임워크 사용 고려
이진 로그 복제 복잡성으로 인해 이런 방식으로 데이터 수집은 오픈소스나 상용 제품을 고려하는 것이 좋다.

* DW가 MySQL이 아닐 가능성이 높기 때문에 단순하게 내장된 MySQL 복제 기능을 사용할 수는 없다.
* 이진 로그를 사용하여 MySQL이 아닌 곳으로의 데이터 이관은 아래의 단계를 거친다.
	1. MySQL 서버에서 이진 로그를 활성화하고 구성한다.
	2. 초기 전체 테이블 추출을 실행하고 로드한다.
	3. 지속적으로 이진 로그를 추출한다.
	4. 추출된 이진 로그를 DW로 변환하여 로드한다.

> NOTE - 이진 로그를 수집 단계에서 사용하려면 먼저 목적지인 DW 테이블 소스인 MySQL DB 테이블의 현재 상태로 채워야 한다. 그 다음 이진 로그를 사용하여 후속 변경 사항을 수집한다. 그렇게 하려면 이진 로그 수집을 켜기 전에 소스 테이블에 LOCK을 설정 뒤 해당 테이블에 mysqldump를 실행 후 덤프 결과를 DW 테이블에 로드해야 한다.

* 이진 로그 활성화 / 구성은 MySQL 이진 로그 설명서를 참조하는 것이 가장 좋다. 
* 주요 구성 설정을 확인해보자.

> TIP - 소스 시스템 소주자와의 논의
> 이진 로그 구성 권한은 종종 시스템 관리자에게만 부여된다. 데이터 엔지니어는 변경사항이 다른 시스템에 영향을 미칠 수 있으므로 DB 소유자와 논의하고 함께 작업해야 한다.

* 이진 로그 구성과 관련하여 MySQL DB에서 선행되어야 하는 두 가지 주요 설정이 있다.
1. 이진 로깅이 활성화되어 있는지 확인한다. SQL 쿼리로 확인 가능하다. 
<script src="https://gist.github.com/JeremyShin/113a6b512ed7a942d02a379c1b388fff.js"></script>
그 이후에 로깅 형식이 적절하게 설정되어 있는지 확인한다. MySQL 최신 버전은 세 가지 형식이 지원된다. 

	* STATEMENT - 행 수집에 대해 SQL문 자체를 기록한다.
	* ROW - 행 자체의 데이터로 이진 로그 행에 표시된다.
	* MIXED - 이진 로그에 STATE 형식 레코드와 ROW 형식 레코드 모두 기록한다.

* 다음 SQL 쿼리를 실행하여 현재 이진 로그의 형식을 확인 가능하다.
<script src="https://gist.github.com/JeremyShin/52a3c2eb38a61f3c941156183d00aa13.js"></script>

* 이진 로그 형식과 기타 구성 설정은 MySQL DB 인스턴스 관련 my.cnf 파일에서 설정된다. 

		[mysqld]
		binlog_format = row
		...

* 이제 이진 로깅이 ROW 형식이 사용되도록 설정되었다.
* 이진 로그에서 가져올 ROW 형식 이벤트에는 세 가지 유형이 있다. - 심화 복제 전략을 사용할떄는 테이블 구조를 수정하는 이벤트를 추출하는 것도 중요하다. 
	* WRITE_ROWS_EVENT
	* UPDATE_ROWS_EVENT
	* DELETE_ROWS_EVENT
* 이진 로그에서 이벤트를 가져올 차례다. 오픈소스 파이썬 라이브러리를 활용해보자. 가장 인기 있는 프로젝트는 python-mysql-replication 프로젝트다.

		(env) $pip install mysql-replication

* DB에 연결하고 이진 로그를 읽어 보자. 

		NOTE - 기본 이진 로그 파일 이름과 경로는 my.conf 파일의 log_bin 변수에 설정된다. 
		자세한 내용은 BinLogStreamReader 클래스에 대한 설명서를 참조한다.

<script src="https://gist.github.com/JeremyShin/e64cb38d06ba53fa6153c2a845dbeedf.js"></script>

* 예제에서 인스턴스화된 BinLogStreamReader 개체에 주의할 점이 있다. 
1. resume_stream=True 설정과 log_pos 값은 지정된 지점에서 이진 로그를 읽기 시작한다.
2. BinLogStreamReader에게 deleteRowsEvent, WriteRowsEvent, UpdateRowsEvent만 읽도록 지시한다.
3. 스크립트를 모든 이벤트를 반복하고 사람이 읽을 수 있는 형식으로 출력한다.

* 출력을 DW로 쉽게 로드하기 위해서는 몇 가지 작업을 더 수행해야 한다.
	1. 데이터를 다른 형식으로 파싱하고 기록한다. csv 파일의 행에 각 이벤트를 기록한다.
	2. 추출 및 로드하려는 테이블당 하나의 파일을 작성한다.

* 첫 번째 변경사항을 해결하기 위해 .dump() 함수 대신 이벤트 속성 구문을 분석하여 csv 파일에 쓴다. 
* 두 번째 경우에는 orders_extract.csv 라는 파일에 Orders 테이블과 관련된 이벤트만 기록한다.
* 예제를 살펴보자
<script src="https://gist.github.com/JeremyShin/94f1616132e78fcd61d826e5acd1fc8f.js"></script>

* 실행 후 orders_extract.csv는 다음과 같이 표시된다.

		insert|1|Backordered|2020-06-01 12:00:00
		update|1|Shipped|2020-06-09 12:00:25

### PostgreSQL 데이터베이스에서 데이터 추출
* MySQL과 비슷하게 SQL 활용 / 복제 지원 기능을 선택하여 사용할 수 있다.
* 몇 가지 방법이 있지만, 이 장에서는 **Postgres WAL(Write-Ahead Log)**을 데이터스트림으로 변환하는 방법을 중점적으로 다룬다.
* 이 섹션의 코드 예제는 매우 간단하며 Postgres DB의 Orders라는 테이블을 참고한다.
* 샘플 테이블, 데이터를 만들어보자. 
<script src="https://gist.github.com/JeremyShin/a4e307393b394e3fc7b477d76d5cbe7b.js"></script>

#### 전체 또는 증분 Postgres 테이블 추출
* 앞 섹션 예제와 마찬가지로 이 예제는 소스 DB의 Orders 테이블에서 데이터를 추출하여 CSV 파일에 쓴 다음 S3 버킷에 업로드한다.
* MySQL 예제와의 유일한 차이점은 데이터를 추출하는데 사용할 파이썬 라이브러리이다.

		(env) $pip install pyscopg2

* pipeline.conf 파일에 새 섹션을 추가해야 한다.

		[postgres_config]
		host = myhost.com
		port = 5432
		username = my_username
		password = my_password
		database = db_name

* 아래는 전체 코드다.
<script src="https://gist.github.com/JeremyShin/321b3bf6d347762e5f53261b2add0a4f.js"></script>

#### Write-Ahead 로그를 사용한 데이터 복제
* MySQL 이진 로그와 같이 Postgres WAL은 추출을 위한 CDC 방법으로 사용가능하다. 
* 여기서는 Debezium 이라는 오픈 소스 분산 플랫폼을 사용하여 Postgres WAL의 콘텐츠를 S3버킷이나 DW로 스트리밍하는 것을 제안한다.
* 카프카 및 Debezium을 통한 스트리밍 데이터 수집에서 후술한다.

### MongoDB에서 데이터 추출
* 집합에서 몽고디비 도큐먼트 하위 집합을 추출하는 방법을 보여준다.

		(env) $pip install pymongo

* 코드에서 데이터를 추출하는 방법과 MongoDB Atlas로 클러스터를 무료로 생성하여 샘플을 실행하는 방법이 있다.
* MongoDB Atlas에서 호스팅하는 클러스터에 연결 시 dnspython 파이썬 라이브러리를 하나 더 설치해야 한다.
* pipeline.conf 파일에 새 색션을 추가한다.

		[mongo_config]
		hostname = my_host.com
		username = mongo_user
		password = mongo_password
		database = my_database
		collection = my_collection

* sample_mongodb.py 파일을 만들자.
<script src="https://gist.github.com/JeremyShin/f1aed625a0a04ac32925b4600e305e93.js"></script>

* 실행하면 3개의 문서가 MongoDB 컬렉션에 삽입된다. 

		(env) $ python sample_mongodb.py

<script src="https://gist.github.com/JeremyShin/bdec5684fc24bb4b9ecd417c459f953d.js"></script>

* 결과는 아래와 같다.

		1|2020-12-13 11:01:37.942000|signup
		2|2020-12-13 11:01:37.942000|pageview
		3|2020-12-13 11:01:37.942000|login

### REST API에서 데이터 추출
* REST API는 데이터를 추출하는 흔한 방법이다. 
* 내부 API를 수집하거나 외부 서비스/공급업체 (트위터 등)에서 제공하는 API를 수집해야 할 수도 있다.
* API에 관계없이 데이터 추출에는 공통 패턴이 있다. 
1. API 엔드포인트로 HTTP GET 요청을 보낸다.
2. JSON 형식일 가능성이 높은 응답을 수락한다.
3. 응답을 구문 분석하고 CSV 파일로 변환(평탄화)한다.

* 이 예제에서는 Open Notify라는 API에 연결한다. NASA의 데이터를 반환한다. 

		http://api.open-notify.org/iss-pass.json?lat=42.36&lon=71.05

* API를 쿼리하고 파이썬에서 응답을 처리하려면 requests 라이브러리를 설치한다. 아래 예제 코드를 보자.
<script src="https://gist.github.com/JeremyShin/aff0f5f405bc58a71a1fea1431742680.js"></script>

### 카프카 및 Debezium을 통한 스트리밍 데이터 수집

* MySQL 이진 로그, Postgres WALs와 같은 CDC 시스템을 통해 데이터 수집시 프레임워크 활용이 꼭 필요하다.
* Debezium은 여러 오픈 소스 서비스로 구성된 분산 시스템으로 일반적인 CDC 시스템에서 행수준 변경을 캡처한 후 다른 시스템에서 사용할 수 있는 이벤트로 스트리밍해주는 시스템이다.
* 설치에는 3가지 구성요소가 있다.
1. 아파치 주키퍼 - 분산 환경을 관리하고 각 서비스 구성을 처리
2. 아파치 카프카 - 분산 스트리밍 플랫폼
3. 아파치 카프카 커넥트 - 쉽게 스트리밍 할 수 있도록 카프카를 다른 시스템과 연결하는 도구.

* 카프카는 포틱별로 정리된 메세지를 교환한다. 하나의 시스템은 토픽에 publish 하고 하나 이상의 시스템은 consume하거나 subscribe 할 수 있다. Debezium은 이런 시스템을 함께 연결하고 일반적인 CDC 구현을 위한 커넥터를 포함한다.
* Debezium은 다양한 커넥터가 있다(MongoDB, MySQL, PostgresSQL, Microsoft SQL Server, Oracle, Db2, Cassandra, S3, Snowflake 등) 

		NOTE - Debezium 설명서를 잘 참고하자

=======
* 구현하기는 더 복잡하지만, 대용량 데이터 수집이 필요한 경우 변경 사항 복제를 위해 MySQL 이진 로그 내용을 사용하는 것이 더 효과적이다.

NOTE - 이진 로그는 CDC의 한 형태다. 대부분의 원본 데이터 저장소에는 사용할 수 있는 CDC 형식이 있다.

* MySQL 이진 로그는 DB에서 수행된 모든 작업에 대한 기록을 보관하는 로그다. (테이블 생성, 수정사항, insert, update, delete 작업도 기록된다)
* 이진 로그 원래 목적은 다른 MySQL 인스턴스로 데이터를 복제하기 위한 것이지만 DW로 데이터를 수집하기 위한 목적으로 사용하기에는 매우 매력적이다.

TIP- 사전 구축된 프레임워크 사용 고려
이진 로그 복제의 복잡성으로 인해 오픈소스 프레임워크 또는 상용 제품을 고려하는 것이 좋다. 

* MySQL에서 MySQL이 아닌 곳으로 데이터를 수집하기 위해 이진 로그를 사용하려면 여러 단계를 수행해야 한다.
1. MySQL 서버에서 이진 로그를 활성화하고 구성한다.
2. 초기 전체 테이블 추출을 실행하고 로드한다.
3. 지속적으로 이진 로그를 추출한다.
4. 추출된 이진 로그를 DW로 변환하여 로드한다.

NOTE - 3단계는 이진 로그 수집 단계에서 사용하려면 먼저 목적지인 DW의 테이블을 소스인 MySQL DB 테이블의 현재 상태로 채워야 한다.
그 다음 이진 로그를 사욯아여 후속 변경 사항을 수집해야 한다.
(이진 로그 수집을 켜기 전에 추출할 소스 테이블에 LOCK을 설정한 뒤 해당 테이블에 대해 mysqldump를 실행한 다음 mysqldump 결과를 DW 테이블에 로드해야 한다.

* 이진 로그를 활성화 및 구성하는 방법은 [최신 MySQL 이진 로그 설명서](https://dev.mysql.com/doc/refman/8.0/en/binary-log.html)를 참조하는 것이 가장 좋다. 

* 이진 로그 구성과 관련하여 MySQL DB에서 선행되어야 하는 두 가지 주요 설정이 있다.
1. 이진 로깅이 되어있는지 확인한다. (DB에서 SQL로 확인 가능)
<script src="https://gist.github.com/JeremyShin/113a6b512ed7a942d02a379c1b388fff.js"></script>

반환 상태가 ON으로 되어있으면 된다. OFF로 되어있는 경우 MySQL 설명서를 참조하여 해당 버전을 활성화해야 한다.

2. 이진 로깅 형식이 적절한지 확인한다. 3가지 형식이 지원된다.
	1. STATEMENT
	2. ROW
	3. MIXED
>>>>>>> 17af3cb0bf6cfcdf46c0535c9050a6ab02dd492c

