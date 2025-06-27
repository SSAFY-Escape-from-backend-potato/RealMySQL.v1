## 질문 1. MySQL에서 B-Tree 인덱스와 Hash 인덱스의 차이점은 무엇인가요?

답변:

B-Tree 인덱스는 범위 검색과 정렬에 유리한 반면, Hash 인덱스는 동등 비교(=)에만 최적화되어 있습니다.
MySQL의 대표 엔진인 InnoDB는 기본적으로 B+Tree 기반 인덱스를 사용합니다.
Hash 인덱스는 MEMORY 엔진에서 사용되며, 인덱스 컬럼을 해시 함수로 변환하여 값을 찾기 때문에 매우 빠르지만, 범위 조건, LIKE, ORDER BY, IN 등에는 사용할 수 없습니다.
	•	B-Tree: WHERE age BETWEEN 20 AND 30, ORDER BY name
	•	Hash: WHERE id = 123

<br>

## 질문 2. 인덱스가 있어도 쿼리 성능이 느릴 수 있는 경우는 어떤 상황인가요?

답변:

다음과 같은 경우 인덱스를 사용하더라도 성능이 떨어질 수 있습니다:
	1.	인덱스 컬럼에 함수 적용: WHERE DATE(created_at) = '2024-06-01' → 인덱스 사용 불가
	2.	형 변환: WHERE phone_number = 01012341234 (숫자 → 문자열)
	3.	선두 컬럼 불일치: (col1, col2) 복합 인덱스인데 col2만 검색
	4.	인덱스 선택성 낮음: ‘gender’처럼 값이 소수개일 경우 인덱스보다 전체 테이블 스캔이 더 빠를 수 있음
	5.	InnoDB에서는 covering index가 아닐 경우 테이블 row 접근(Bookmark Lookup)이 추가로 발생

따라서 인덱스를 쓰더라도 쿼리 튜닝과 구조 이해가 병행되어야 합니다.

<br>

## 질문 3. InnoDB에서 클러스터형 인덱스란 무엇이며, 어떤 장점과 단점이 있나요?

답변:

InnoDB는 기본 키를 기준으로 정렬된 B+Tree 구조인 클러스터형 인덱스(Clustered Index)를 사용합니다.
테이블의 데이터 자체가 인덱스의 leaf 노드에 저장되기 때문에, 보조 인덱스(Secondary Index)는 실제 데이터 주소가 아니라 클러스터 인덱스의 키를 저장합니다.
	•	장점:
	•	기본 키 순으로 데이터가 정렬되어 있어 범위 검색이 빠름
	•	기본 키 기반 검색 시 추가 접근 없이 데이터 조회 가능
	•	단점:
	•	기본 키가 너무 길면 보조 인덱스가 비대해져 성능 저하
	•	기본 키 변경 시 전체 데이터 재정렬 필요 → 비용 큼

<br>

## 질문 4. 트랜잭션에서 격리 수준(Isolation Level)에 따라 발생할 수 있는 문제점은 무엇인가요?

답변:

트랜잭션의 격리 수준에 따라 다음과 같은 문제가 발생할 수 있습니다:

| 격리 수준          | 허용되는 현상             | 설명                                      |
|-------------------|---------------------------|-------------------------------------------|
| READ UNCOMMITTED  | Dirty Read                | 커밋되지 않은 변경을 읽음                |
| READ COMMITTED    | Non-repeatable Read       | 동일 쿼리라도 읽는 값이 달라짐           |
| REPEATABLE READ   | Phantom Read *(InnoDB는 방지)* | 범위 쿼리에서 결과 행 수가 달라짐     |
| SERIALIZABLE      | 없음                      | 가장 강력한 격리 수준, 동시성 낮음       |

InnoDB는 기본적으로 REPEATABLE READ이며, MVCC를 통해 Phantom Read도 방지합니다.
하지만 SELECT ... FOR UPDATE 같은 명시적 잠금이 없으면 동시성 문제가 발생할 수 있습니다.

<br>

## 질문 5. InnoDB 트랜잭션이 ACID를 만족하기 위해 사용하는 메커니즘은 어떤 것이 있나요?

답변:

InnoDB는 ACID 보장을 위해 다음 메커니즘을 사용합니다:  

	1.	Atomicity (원자성):  
	    Undo 로그를 사용하여 트랜잭션 실패 시 롤백 가능  

	2.	Consistency (일관성):
	    FK 제약, 트랜잭션 내 무결성 체크 등을 통해 보장  

	3.	Isolation (격리성):  
	    트랜잭션 격리 수준 + MVCC로 구현
        
	4.	Durability (지속성):  
	    Redo 로그 + 디스크 플러시(fsync)로 커밋된 변경을 디스크에 반영

InnoDB는 Doublewrite Buffer, Redo Log, Undo Log, Flush 정책 등을 조합하여 ACID를 보장합니다.

