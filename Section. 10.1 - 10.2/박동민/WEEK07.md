## 10.1

테이블과 인덱스 에 대한 개괄적인 정보만 참고하면 정확도가 떨어질 수도 있기 때문에 데이터 분포도 기준으로 히스토그램을 만들었다.

## 10.1,1

비용 기반 최적화 → 통계 정보 기반 ex) 데이터가 몇 개 있는지

MySQL: 비용 기반 최적화인데 휘발성도 강하고 정확도도 별로라서 아래와 같은 개선들을 함

## 10.1.1.1 서버통계정보

 스토리지 엔진을 사용하는 테이블에 대한 통계 정보를 영구적으로 관리.

기존에는 아래와 같은 경우 자동으로 통계 정보가 갱신 되었으나 의도치 않은 방식으로 쿼리를 읽게 되도록 변경될 수 있다는 문제점을 내포. 따라서 지금은 영구적으로 반영해버림(옵션으로 변경 가능)

- 테이블이 새로 오픈되는 경우
- 테이블의 레코드가 대량으로 변경되는 경우(테이블의 전체 레코드 중에서 1/16 정도의 UPDATE 또는 INSERT나
DELETE가 실행되는 경우)
- ANALYZE TABLE 명령이실행되는 경우
- SHOW TABLE STATUS 명령이나 SHOW INDEX FROM 명령이 실행되는경우
- InnoDB 모니터가 활성화되는 경우
- innodb_stats_on_metadata 시스템 설정이 ON인 상태에서 SHOW TABLE STATUS 명령이 실행되는 경우

## 10.1.2 히스토그램

기존에는 랜덤으로 페이지를 참조해서 실행계획 수립, 그러나 이제는 데이터 분포도인 히스토그램을 사용

## 10.1.2.1

히스토그램은 버킷(Bucket) 단위로 구분되어 레코드 건수나 칼럼값의 범위가 관리되는데, 싱글톤 히
스토그램은 칼럼이 가지는 값별로 버킷이 할당되며 높이 균형 히스토그램에서는 개수가 균등한 칼럼값
의 범위별로 하나의 버킷이 할당된다. 싱글톤 히스토그램은 각 버킷이 칼럼의 값과 발생 빈도의 비율의
2개 값을 가진다. 반면 높이 균형 히스토그램은 각 버킷이 범위 시작 값과 마지막 값, 그리고 발생 빈도
율과 각 버킷에 포함된 유니크한 값의 개수 등 4개의 값을 가진다.

변경사항이 생기면 실행계획에 변동이 생길 수 있음

## 10.1.2.2

통계 정보가 없으면 옵티마이저는 데이터 분포가 균등하다는 가정을 하기 떄문에 부정확함. 따라서 쿼리 성능 차이가 발생

## 10.1.2.3

히스토그램은 주로 인덱스되지 않은 칼럼에 대한 데이터 분포도를
참조하는 용도로 사용됨(인덱스는 항상 히스토그램보다 좋은 성능을 가지고 있기 때문)

인덱스 다이브 :옵티마이저는 실제 인덱스의 B-Tree를 샘플링해서 살펴본다. 

## 10.1.3

MySQL 서버가 쿼리를 처리하려면 다음과 같은 다양한 작업을 필요로 한다.

- 디스크로부터 데이터 페이지 읽기
- 메모리(InnoDB 버퍼 풀)로부터 데이터 페이지 읽기
- 인덱스 키 비교
- 레코드 평가
- 메모리 임시 테이블 작업
- 디스크 임시 테이블 작업

코스트 모델: MySQL 서버는 사용자의 쿼리에 대해 이러한 다양한 작업이 얼마나 필요한지 예측하고 전체 작업 비용을 계산한 결과를 바탕으로 최적의 실행 계획을 찾는다. 이렇게 전체 쿼리의 비용을 계산하는 데 필요한 단위 작업들의 비용. 

각 단위 작업에 설정되는 비용 값이 커지면 어떤 실행 계획들이 고비용으
로 바뀌고 어떤 실행 계획들이 저비용으로 바뀌는지를 파악하는 것이 중요

## 10.2

EXPLAIN

## 10.2.2

EXPLAIN ANALYZE: 소요된 시간 정보를 확인할 수 있다.  단계별로 소요된 시간 정보를 TREE 형태로 보여줌.

EXPLAIN ANALYZE 명령은 EXPLAIN 명령과 달리 실행 계획만 추출하는 것이 아니라 실제 쿼리를 실행하고 사용된 실행 계획과 소요된 시간을 보여주는 것이다. 그래서 쿼리의 실행 시간이 아주 많이 걸리는 쿼리라면 EXPLAIN ANALYZE 명령을 사용하면 쿼리가 완료돼야 실행 계획의 결과를 확인할 수 있다. 쿼리의 실행 계획이 아주 나쁜 경우라면 EXPLAIN 명령으로 먼저 실행 계획만 확인해서 어느 정도 튜닝한 후 EXPLAIN ANALYZE 명령을 실행하는 것이 좋다.
