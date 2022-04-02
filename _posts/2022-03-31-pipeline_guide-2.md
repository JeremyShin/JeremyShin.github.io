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
