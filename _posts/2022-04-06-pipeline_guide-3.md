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


## 05. 데이터 수집: 데이터 로드
* 이제 데이터를 로드할 차례다. 로드 방법은 추출 산출물이 어떤 모습일지에 따라 다르다. 
* 이 섹션에서는 테이블 각 열에 해당하는 값을 사용하여 추출된 데이터를 csv 파일로 로드하는 방법과 CDC 형식의 산출물을 추출하는 방법을 설명한다.

### Amazon Redshift 웨어하우스를 대상으로 구성
* IAM 역할이 없는 경우 생성하자. (redshift 생성시 과금 주의!!)
* [생성은 AWS 설명서를 참고](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create.html)
* 생성한 IAM 역할은 Redshift 클러스터와 연결한다.
* [연결은 AWS 설명서를 참고](https://docs.aws.amazon.com/redshift/latest/mgmt/copy-unload-iam-role.html)
* pipeline.conf 파일에 섹션을 추가하자.

		[aws_creds]
		database = my_warehouse
		username = pipeline_user
		password = my_password
		host = my_example.1234.us-east-1.redshift.amazonaws.com
		port = 5439
		iam_role = RedshiftLoadRole

* Redshift 자격증명 모범 사례

		프로덕션 환경에서는 IAM 인증을 사용하여 보다 강력한 보안 전략을 고려하는 것이 좋다. [(IAM 인증 가이드 참고)](https://docs.aws.amazon.com/redshift/latest/mgmt/generating-user-credentials.html)
		로컬 pipeline.conf 파일보다는 DB 자격증명 및 보안을 위한 password를 더 안전하게 저장하는 것이 좋다. 
		[Vault](https://www.vaultproject.io/)와 같은 것이 있다.
	 

### Redshift 웨어하우스에 데이터 로드
* S3에서 redshift로 데이터를 로드하는 가장 효율적인 방법은 COPY 명령어를 사용하는 것이다. 

		COPY my_schma.my_table (column_1, column_2, ..)
		FROM 's3://bucket-name/file.csv'
		iam_role 'arn:aws:iam::222role/RedshiftLoadRole';

* 위의 명령이 실행되면 file.csv 내용이 Redshift에 추가된다.
* 파이썬 스크립트에서 COPY 명령을 구현하기 위해 Boto3 라이브러리를 사용할 수도 있다.

		(env) $ pip install psycopg2

* copy_to_redshift.py 파일을 만들자.
<script src="https://gist.github.com/JeremyShin/09a0c60e979de26ba0df724637a50dd1.js"></script>

* 3장의 order_extract.csv 파일로 추출된 데이터를 로드한다.

		CREATE TABLE	public.Orders (
			OrderId int,
			OrderStatus varchar(30),
			LastUpdated timestamp
			);

* 아래의 스크립트를 실행하자.

		(env) $ python copy_to_redshift.py

#### 증분 및 전체 로드

* 데이터가 전부 추출된 경우 로딩 스크립트에 추가할 사항이 하나 있다. COPY 작업을 실행 전에 Redshift에서 대상 테이블을 잘라야 한다.(TRUNCATE 사용).

* copy_to_redshift.py 파일에서 아래의 코드를 추가한다.

		...
		bucket_name = parser.get("aws_boto_credentials", "bucket_name")
		
		# 대상 테이블을 truncate
		sql = "TRUNCATE public.Orders;"
		cur = rs_conn.cursor()
		cur.execute(sql)
		
		cur.close()
		...
		

* 데이터가 증분 추출되었다면 대상 테이블을 자르면 안된다. (생각할 문제가 많다!)
* 마지막 업데이트된 시간을 나타내는 타임스탬프에 의존하여 데이터를 구분한다.
* 아래와 같은 경우가 있다.

|OrederId| OrderStatus  | LastUpdated |
|1|Backordered| 2020-06-01 12:00:00 |
|1|Shipped  | 2020-06-09 12:00:25 |

* 기록 보관의 관점에서 볼 때 두 데이터를 모두 보는게 이상적이다. 
* 데이터 수집의 목표는 데이터 추출 및 로드에 중점을 두는 것이다. 데이터로 무엇을 할지는 데이터 변환 단계에서 할 일이다.

#### CDC 로그에서 추출한 데이터 로드
* DED의 경우 위의 경우 + 삭제된 레코드에도 액세스 가능하다.

		insert|1|Backordered|2020-06-01 12:00:00
		update|1|Shipped|2020-06-09 12:00:25
		delete|1|Shipped|2020-06-10 09:0512

* 전체 추출에서는 레코드가 완전히 사라졌을 것
* 증분 추출에서는 삭제(delete)를 가져오지 않았을 것
* CDC에서는 삭제 이벤트가 선택되어 CSV파일에 포함됨

### Snowflake 웨어하우스를 대상으로 구성
* snowflake 인스턴스에서 S3 버킷 액세스 구성에 대한 3가지 옵션이 있다.
	* snowflake 스토리지 통합 구성
	* AWS IAM 구성
	* AWS IAM 사용자 구성

* 단계별 자유도와 제약이 있으니 확인하자

* 구성 마지막 단계는 외부 단계(external stage)를 만들게 되는데, snowflake가 액세스 가능하도록 외부 저장 위치를 가리키는 객체다.

* 아래의 포멧으로 유사한 파일 형식을 사용할 수 있다.

		CREATE or REPLACE FILE FORMAT pipe_csv_format 
		TYPE = 'csv'
		FIELD_DELIMITER = "|";

* snowflake 설명서 마지막에서 버킷에 대한 단계를 생성시 구문은 다음과 같다.

		USE SCHEMA my_db.my_schema;
		
		CREATE STAGE my_s3_stage
			storage_integration = s3_int
			url = 's3://pipeline-bucket/'
			file_format = pipe_csv_format;

* 마지막으로 snowflake 로그인 자격증명이 있는 pipeline.conf 파일에 섹션을 추가한다.

		[snowflake_creds]
		username = snowflake_user
		password = snowflake_password
		account_name = snowflake_acct1.us-east-2.aws


### Snowflake 데이터 웨어하우스에 로드

* 추출된 파일에서 데이터를 로드하는 구문을 설명한다.
* Snowflake 데이터 로드 매커니즘은 COPY INTO 명령이다. 자세한 사항은 [Snowflake 설명서](https://docs.snowflake.com/en/sql-reference/sql/copy-into-table.html)를 참조하자. 

```sql
	COPY INTO destination_table
	FROM @my_s3_stage/extract_file.csv;
```

* 여러 파일을 테이블에 한 번에 로드하는 것도 가능하다.

```sql
	COPY INTO destination_table
	FROM @my_s3_stage
	pattern='.*extract*.csv';
```

* 이제 COPY INTO 명령이 작동하는 방식을 알았으니 파이프라인의 로드를 자동화하기 위한 파이썬 스크립트를 작성해보자.

```sh
	(env) $ pip install snowflake-connector-python
```

<script src="https://gist.github.com/JeremyShin/bff62a798d8b0e9704b8e66c13a1b4dd.js"></script>


### 파일스토리지를 데이터 레이크로 사용

* S3 버킷(또는 다른 클라우드 스토리지)에 정형화 또는 반정형화된 형태로 저장된 데이터 저장소를 데이터 레이크라고한다.
* 최근 몇년동안 SQL에 익숙한 사용자가 데이터 레이크에 쿼리할 수 있게 도와주는 도구가 등장했다.
* 데이터 과학 또는 머신러닝 프로젝트 탐색 단계에서 담당자는 데이터의 모양(shape)이 무엇인지 모를수있다.
* 원본 형태로 데이터에 접근하면 DW의 테이블에 로드하는 것이 합리적인지 여부를 결정할 수 있고 쿼리 최적화를 할 수 있다.
* 데이터 레이크와 DW는 경쟁이 아닌 보완적인 솔루션이다.

### 오픈 소스 프레임워크
* [Singer](https://www.singer.io/) - 파이썬으로 작성, tap 사용 후 데이터 추출 및 JSON 형태로 대상으로 스트리밍한다.
* Singer는 문서화가 잘 되어있으며 슬랙, 깃허브 커뮤니티가 있다.

### 상업적 대안
* 일정 조정 및 오케스트레이션 기능이 기본적으로 제공된다. (추가 비용이 든다)
* 인기 있는 도구는 Stitch와 Fivetran이다.

#### 비용
* Stitch와 Fivetran은 모두 볼륨 기반 가격 모델을 가지고 있다. 금액은 데이터 양에 따라 결정된다.

#### 공급업체 존속
* 공급업체에 투자하면 향후 도구/제품으로 마이그레이션 시 엄청난 양의 작업에 직면하게 된다.

#### 커스터마이징에 필요한 코딩
* 미리 빌드된 커넥터가 없는 경우 코드를 직접 작성해야 한다.
* Singer - 탭에 작성
* Fivetran - AWS lambda, Azure Function, Google Cloud Functions를 활용한다.
* 데이터 소스가 많은 경우에 Singer, Fivetran에 추가 비용을 부담한다.

#### 보안 및 개인 정보 보호
* 두 제품 모두 기술적으로는 소스 시스템과 Loading 대상에 모두 엑세스 할 수 있다. 위험 허용 범위, 규정 요구 사항, 잠재적인 책임, 데이터 프로세스를 검토하고 승인하는 오버헤드로 활용을 꺼릴 수 있다.
* 사용자 지정 도구와 사용 도구를 혼합하여 선택하는 경우 로깅, 경고 및 종속성 관리와 같은 항목을 표준화하는 방법을 고려해야 한다.
