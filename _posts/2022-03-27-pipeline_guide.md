
#  데이터 파이프라인 핵심 가이드

1. 데이터 파이프라인 소개
2. 최신 데이터 인프라
3. 일반적인 데이터 파이프라인 패턴
4. 데이터 수집: 데이터 추출
5. 데이터 수집: 데이터 로드
6. 데이터 변환하기
7. 파이프라인 오케스트레이션
8. 파이프라인 데이터 검증
9. 파이프라인 유지 관리 모범 사례
10.파이프라인 성능 측정 및 모니터링 


## 1. 데이터 파이프라인 소개 
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

