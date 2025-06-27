# Real My SQL 질문 리스트

1. REPEATABLE READ 격리 수준 작동 방식에 대해서 설명해주고 Phantom Read가 발생하는 원인에 대해서 설명해 주세요

   Answer : 모든 InnoDB 트랜잭션은 고유한 트랜잭션 번호가 존재합니다. 언두 영역에 백업된 모든 레코드는 변경 발생시킨 트랜잭션 번호가 존재합니다. REPEATABLE READ의 경우 MVCC(잠금 없는 일관된 읽기) 보장하기 위해 현재 트랜잭션보다 이후에 커밋된 데이터는 조회 시 필터링합니다. 이로써 일관된 읽기(Consistent Read) 가 구현됩니다. 하지만 WHERE 조건에 해당하지 않던 새로운 레코드가 다른 트랜잭션에 의해 INSERT되는 경우, REPEATABLE READ에서는 기본적으로 이를 차단하지 않기 때문에 Phantom Read(팬텀 리드) 가 발생할 수 있습니다.

2. InnoDB의 독특한 구조로 인해 REPEATABLE READ에서 Phantom Read가 발생하지 않는 이유에 대해서 설명해주세요.

   Answer : InnoDB는 이러한 팬텀 현상을 방지하기 위해 Next-Key Lock이라는 레코드 락(Record Lock) + 갭 락(Gap Lock) 결합 방식을 내부적으로 활용해 범위에 대해 잠금을 걸고, Phantom Read를 방지합니다.

   ```sql
   -- 트랜잭션 A 시작
   START TRANSACTION;

   -- 조건에 맞는 사용자 조회
   SELECT * FROM users WHERE age BETWEEN 20 AND 30;

   -- InnoDB는 age BETWEEN 20 AND 30 범위 내의 레코드들에 대해 레코드 락을 걸고,
   -- 이 범위의 양쪽 경계 영역에도 Gap Lock을 함께 겁니다.
   -- 결과적으로, age가 20보다 크고 30보다 작은 모든 영역이 잠김 상태가 되어, 다른 트랜잭션이 이 범위 내에 레코드를 새로 삽입하는 것을 차단합니다.

   -- 트랜잭션 B에서 시도
   START TRANSACTION;

   INSERT INTO users (id, name, age) VALUES (1001, 'fan', 25);
   -- → 트랜잭션 A가 커밋되기 전까지 대기 (또는 오류)

   -- 즉, 트랜잭션 B는 age=25인 레코드를 INSERT하려 해도 트랜잭션 A가 해당 범위에 Next-Key Lock을 설정했기 때문에, 트랜잭션 A가 끝날 때까지 기다리거나, 교착 상태로 롤백됩니다.
   ```

3. 인덱스 스킵 스캔에 대해서 설명해주시고 어떠한 상황에서 인덱스 스캡 스캔이 문제가 발생하는지에 대해서 기술해주세요

   Answer : 인덱스 스킵 스캔(Index Skip Scan)은 복합 인덱스의 선두 컬럼에 조건이 없더라도 후행 컬럼의 조건만으로 인덱스를 사용할 수 있도록 하는 기능입니다. MySQL 8.0부터 InnoDB에서 지원되며, 옵티마이저가 조건절을 만족하기 위해 선두 컬럼의 가능한 값을 추정한 후 반복적으로 후행 컬럼 조건을 탐색합니다.

   ```sql
   mysql> ALTER TABLE employees ADD INDEX ix_gender_birthdate (gender, birth_date);

   mysql> SELECT * FROM employees WHERE birth_date >= '1965-02-01'
   mysql> SELECT * FROM employees WHERE gender='M' AND birth_date >= '1965-02-01'
   ```

   기존에는 두 번째 쿼리는 선두 컬럼인 gender 조건이 없어 인덱스를 사용할 수 없었지만, 인덱스 스킵 스캔이 적용되면 gender 컬럼의 모든 가능한 값(M, F)을 순차적으로 스캔하며 birth_date 조건을 만족하는 데이터를 후행 컬럼 기준으로 검색하게 됩니다.
   그러나 선두 컬럼의 유니크 값(카디널리티)이 너무 많을 경우 인덱스 스킵 스캔은 선두 컬럼별로 후행 컬럼을 반복 검색하기 때문에 오히려 Full Index Scan보다 느려질 수 있습니다. 예를 들어 region_code, birth_date가 결합된 인덱스 (region_code, birth_date)에서 region_code가 수천 개인 경우 birth_date >= '2000-01-01' 조건으로 스킵 스캔을 사용할 경우 수천 번 반복 조회가 발생할 수 있습니다.

   또한 인덱스 스킵 스캔은 인덱스에 포함된 컬럼만으로 쿼리가 해결될 수 있을 때, 즉 커버링 인덱스(covering index)일 때 가장 효과적입니다. 만약 SELECT 절에 인덱스에 포함되지 않은 다른 컬럼이 있다면, 인덱스를 통해 레코드 위치를 찾은 후 테이블 접근(secondary lookup)이 필요하므로 성능상의 이점이 줄어들 수 있습니다.

4. 인덱스에서 B+Tree 구조 대신 트리 구조나 Hash 구조를 사용하지 않는 이유는 무엇이라 생각하시나요?

   Answer : B+Tree가 인덱스 구조로 많이 사용되는 이유는 범위 검색과 특정 키 값으로 정렬됨에 있습니다. Hash는 자료구조 특성상 Range Scan이 불가능하기 때문에 기본 인덱스로 채택하기에는 어려움이 있습니다. 트리 구조는 깊이가 많이 깊어져 Disk I/O가 많이 발생할 수 있습니다. 일반적인 트리 구조의 경우 Depth가 깊어져서 Disk I/O가 많이 발생하여 성능상으로 B+Tree가 유리합니다.

5. 인덱스 레인지 스캔과 인덱스 풀 스캔의 차이점에 대해서 기술해주세요
   Answer : 인덱스 레인지 스캔의 경우 인덱스의 범위가 결정이 되었을 때 사용되는 방식입니다. 인덱스의 첫 번째 시점으로부터 시작 지점을 설정한 이후에 끝까지 탐색을 진행하지 않고 리프노드의 레코드 순서대로 읽다 반환 지점에서 종료를 하는 것이 특징입니다. 인덱스 풀 스캔의 경우 명시된 컬럼만으로 조건을 처리할 수 없는 경우 사용됩니다. 인덱스의 리프 노드를 연결하는 링크 리스트를 따라서 처음부터 끝까지 스캔하는 방식입니다. (cf 인덱스 풀 스캔의 경우 인덱스를 사용하지 못한다 또는 인덱스를 효율적으로 사용하지 못한다는 표현을 사용합니다.)
