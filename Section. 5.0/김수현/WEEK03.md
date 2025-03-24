# Real MySQL 5 : 트랜잭션과 잠금

# **Introduction**

- **잠금** : 동시성 제어 목적 ⇒ 여러 커넥션에서 동일 자원에 대허서 요청할 경우 하나의 시점에 하나의 커넥션만 허용
- **트랜잭션** : 데이터 정합성 보장 목적 ⇒ 논리적인 작업 셋이 완벽하게 처리되어 모두 처리하거나 하나라도 문제가 생기면 전체에 대해서 원상태로 복구하며 Partial Update 방지

# 5.1 트랜잭션

- Intro
  - MyISAM, Memory : 비트랜잭션 스토리지 엔진 ⇒ InnoDB에 비해 빠른 속도를 보인다.
  - InnoDB : 트잭션 지원 스토리지 엔진 ⇒ 데이터 정합성 측면에서 우수한 측면을 보인다.
- 5.1.1 MySQL에서 트랜잭션

  - MyISAM와 InnoDB에서 같은 쿼리문 실행 시 차이 비교

    ```jsx
    mysql > CREATE TABLE tab_myisam (fdpk INT NOT NULL, PRIMARY KEY (fdpk) ) ENGINE=MYISAM;
    mysql > INSERT INTO tab_myisam (fdpk) VALUES (3);

    mysql > CREATE TABLE tab_innodb (fdpk INT NOT NULL, PRIMARY KEY (fdpk) ) ENGINE=INNODB;
    mysql > INSERT INTO tab_innodb (fdpk) VALUES (3);

    mysql > INSERT INTO tab_myisam (fdpk) VALUES (1), (2), (3);
    ERROR 1062 (23000): Duplicate entry '3' for key 'PRIMARY'

    mysql > INSERT INTO tab_innodb (fdpk) VALUES (1), (2), (3);
    ERROR 1062 (23000): Duplicate entry '3' for key 'PRIMARY'

    mysql > SELECT * FROM tab_myisam;
    -|--------|-
    |   fdpk   |
    -|--------|-
    |    1    |
    |    2    |
    |    3    |
    -|--------|-

    mysql > SELECT * FROM tab_innodb;
    -|--------|-
    |   fdpk   |
    -|--------|-
    |    3    |
    -|--------|-
    ```

  - InnoDB : 원자성을 보장받지못해 Error가 발생해도 전체 요청에 대한 롤백을 처리하지 못하고 1,2까지 insert 이후에 3 insert 시에 중복 문제 발생 → 1, 2 insert에 대해 롤백하지 못해 이러한 결과가 발생(부분 업데이트)
  - InnoDB : 트랜잭션에 의해 원자성이 보장받아 1, 2, 3 insert 중에 3에 Error 발생 → 전체 요청에 대해서 롤백 처리

- 5.1.2 주의사항
  - 트랜잭션 적용 범위 : DBMS의 커넥션과 동일하게 최소의 코드에만 적용
    1. DBMS의 Insert, Update 하는 부분
    2. 단순 조회(Select)에는 트랜잭션 일반적으로 걸지 않는다.
    3. 메일 전송이나 FTP 파일 전송 작업 또는 네트워크 통한 원격 서버와 통신하는 것은 DBMS의 트랙잭션에서 제거하는 것이 좋다 → 메일 서버와 통신할 수 없는 상황이 발생하면 웹 서버뿐 아니라 DBMS 서버까지 위험해지는 상황 발생
    - 예시 상황
      1. 사용자가 웹 애플리케이션에서 회원 가입 진행
      2. DB에 Insert와 동시에 회원 가입 확인 메일 전송
      3. 두 작업을 하나의 트랜잭션으로 묶어서 처리한다고 가정
    - 문제 발생 시나리오
      - DB Insert는 정상적으로 완료
      - 그런데 메일 서버가 다운되어 메일 전송 실패
      - 트랜잭션이 걸려 있어 메일 전송 실패로 인해 전체 트랙잭션 롤백 → DB 저장된 회원 정보도 취소(rollback)
      - 트랜잭션이 장시간 메일 서버의 응답 기다리는 동안 DB 커넥션도 계속 점유된 상태
      - 이것이 반복되면 DB 커넥션 풀 부족 → 다른 요청 처리 불가 → 웹 서버 마비, DBMS 서버도 부하 발생
    - 정리
      - 메일 서버나 네트워크와 통신하는 작업은 불확실성이 높기 때문에, 이를 DB 트랜잭션 내에 포함하면 DBMS 리소스가 낭비되고 시스템 전체에 연쇄적인 장애를 초래할 수 있음.
      - 따라서 메일 전송은 DB 트랜잭션이 끝난 후 별도로 처리하거나, 비동기 처리 방식으로 구현하는 것이 안전합니다.

# 5.2 MySQL 엔진 잠금

- Intro
  - MySQL 엔진 레벨 잠금 : 모든 스토리지에 영향 미침
  - 스토리지 엔진 레벨 잠금 : 특정 스토리지에만 영향 미침 → 스토리지 엔진 간 영향 X
- 5.2.1 글로벌 락

  - 명령어 : FLUSH TABLES WITH READ LOCK
  - 범위 : MySQL 잠금 가운데 가장 큰 범위 → MySQL 서버 전체, 작업 대상 테이블이나 데이터베이스가 다르더라도 동일하게 영향
  - 주의 사항 : 글로벌 락은 MySQL 서버 내 모든 테이블 닫고 잠금을 건다.만약 장시간 SELECT 쿼리가 실행되는 상황에서 글로벌 락을 걸면 SELECT 완료될 때까지 장시간 대기 → 최악의 경우 SELCET와 글로벌 락으로 인해 뒤에 UPDATE, DELETE 쿼리가 오랜 시간 동안 실행되지 못하고 기다릴 수 있다. → 가급적 사용하지마라
  - InnoDB 상용화되면서 트랜잭션 지원하기 때문에 모든 데이터 변경 작업 멈출 필요 없다. & MySQL 8.0부터 더 가벼운 글로벌 락 도입
  - 추가적으로 작성

- 5.2.2 테이블 락

  - 정의 : 개별 테이블 단위로 설정되는 잠금, 명시적 혹은 묵시적으로 특정 테이블 락 획득할 수 있다.
  - 명시적 : “LOCK TABLES table_name [ READ | WRITE]”, “UNLOCK TABLES”
  - 묵시적
    - MyISAM, MEMORY : 변경되는 테이블에 잠금 설정 → 데이터 변경 → 잠금 해제, 즉 쿼리가 실행되는 동안 자동으로 획득했다가 완료 후 자동 해제
    - InnoDB : DML 쿼리에서는 자동적으로 테이블 잠금 X, DDL 쿼리에서는 자동적으로 테이블 잠금

- 5.2.3 네임드 락

  - 정의 : GET_LOCK(’특정 문자열’, [할당 시간]) 함수를 이용해 임의의 문자열에 대해 잠금 설정
  - 대상 : 테이블, 레코드, AUTO_INCREMENT 같은 데이터베이스 객체가 아닌 사용자가 지정한 문자열

    ```jsx
    mysql > SELECT GET_LOCK('mylock', 2);
    // "mylock"이라는 문자열에 대해 잠금 획득
    // 이미 잠금 사용 중이면 2초 동안만 대기 (2초 이후 자동 잠금 해제)

    mysql > SELECT IS_FREE_LOCK('mylock');
    // "mylock"이라는 문자열에 대해 잠금 설정되어 있는지 확인

    mysql > SELECT RELEASE_LOCK('mylock');
    // "mylock"이라는 문자열에 대해 잠금 반납(해제)

    //3개 함수 모두 정상적으로 락 획득하거나 해제한 경우 1, 아닌 경우 NULL이나 0 반환
    ```

  - 사례 : 배치 프로그램 등 한 번에 많은 레코드 변경하는 쿼리에서 발생하는 데드락이 일어나는 상황에서 네임드 락 걸고 쿼리 실행 시 동일 데이터 변경, 참조 프로그램끼리 분류해 해결 가능
  - MySQL 8.0 이후부터 네임드 락 중첩 사용 가능

    ```jsx
    mysql > SELECT GET_LOCK('mylock1', 10);
    // mylock1에 대해 작업 실행
    mysql > SELECT GET_LOCK('mylock2', 10);
    // mylock2에 대해 작업 실행

    mysql > SELECT RELEASE_LOCK('mylock2');
    mysql > SELECT RELEASE_LOCK('mylock1');

    mysql > SELECT RELEASE_ALL_LOCKS();
    // mylock2, mylock1 동시에 모두 해제
    ```

- 5.2.4 메타데이터 락
  - 정의 : 데이터베이스 객체(테이블, 뷰…)의 이름이나 구조 변경하는 경우 획득하는 잠금, 명시적 획득 불가
  - 묵시적 획득 : RENAME TABLE 과 같은 테이블 이름 변경 등과 같이 자동으로 획득, RENAME의 경우 원본 이름과 변경 이름 모두 한꺼번에 잠금 설정
    ```jsx
    mysql > RENAME TABLE rank TO rank_backup, rank_new TO rank;
    ```
    - 위의 같이 두 개의 RENAME 한꺼번에 실행 시에 ‘Table not found rank’ 같은 상황 발생하지 않고 적용하는 것이 가능
    ```jsx
    mysql > RENAME TABLE rank TO rank_backup;
    mysql > RENAME TABLE rank_new TO rank;
    ```
    - 위의 같이 두 개의 쿼리문으로 나누어서 실행하는 경우 rank 테이블이 존재하지 않는 순간이 생겨 ‘Table not found rank’ 같은 상황 발생

# 5.3 InnoDB 스토리지 엔진 잠금

- Intro
  - InnoDB 레코드 기반 잠금으로 MyISAM보다 뛰어난 동시성 처리 보임
  - 스토리지 엔진 레벨 잠금 : 특정 스토리지에만 영향 미침 → 스토리지 엔진 간 영향 X
- 5.3.1 InnoDB 스토리지 엔진의 잠금
  - 레코드 락
    - 정의 : 특정 레코드 자체를 잠금
    - InnoDB 스토리지 엔진 레코드 락은 레코드 자체가 아니라 인덱스의 레코드 잠금, 인덱스가 하나도 없는 테이블이라도 내부적으로 자동 생성된 클러스터 인덱스는 존재 → 이를 통해서 레코드 락을 건다.
  - 갭(GAP) 락
    - 정의 : 레코드 - 레코드 사이의 간격 잠금
    - 사용법 : 레코드와 레코드 사이의 간격에 새로운 레코드가 생성(INSERT)되는 것을 제어한다.
  - 넥스트 키 락
    - 정의 : 레코드 락 + 넥스트 락 → 즉, **레코드 + 그 이전 갭**을 잠금.
    - STATEMENT 포맷의 바이너리 로그 사용하는 MySQL 서버에서는 REPEATABLE READ 격리 수준을 사용
    - 예시 : 테이블에 10, 20, 30이라는 인덱스 값이 있을 때, SELECT \* FROM table WHERE value = 20 FOR UPDATE; 실행 시 넥스트 키 락은 [10~20] 구간을 잠금.
    - 목적 및 역할
      - **팬텀 리드 방지**
        1. 트랜잭션 중간에 **다른 트랜잭션이 레코드 사이에 새로운 레코드를 삽입하는 것 방지**. ex) value BETWEEN 10 AND 30 조회 후, 다른 트랜잭션이 value=25; 삽입 불가.
        2. **동시성 제어 :** 동시에 삽입·조회·갱신하는 작업에서 **데이터 일관성 유지**.
  - 자동 증가 락
    - 정의 : AUTO_INCREMENT 칼럼이 사용된 테이블에 동시에 여러 레코드 INSERT 시 저장되는 각 레코드가 중복되지 않고 저장한 순서대로 일련번호 증가를 위함
    - INSERT, REPLACE 등 같이 새로운 레코드 저장하는 쿼리에서만 작동, UPDATE나 DELETE 등 쿼리에서 걸리지 않는다.
    - 자동 증가 락을 명시적으로 획득, 해제 방법 X → 자동 증가 락은 아주 짧은 시간 동안 걸리고 해제되는 잠금이기에
    - MySQL 5.1 이후 innodb_autoinc_lock_mode
      - 0 (전통 모드)
        - 모든 INSERT 문에서 자동 증가 락 사용
        - MySQL 5.0과 동일 방식
        - 가장 보수적, 동시성 낮음
      - **1** (연속 모드, 기본값)
        - MySQL이 **INSERT 건수를 사전에 정확히 파악할 수 있으면 락 대신 래치(뮤텍스)** 사용
        - 정확한 건수 파악 불가능 시 자동 증가 락 사용
          ex) INSERT … SELECT ….
      - 2 (간편 모드)
        - **모든 상황에서 자동 증가 락 미사용**, 항상 래치로 처리
        - **가장 높은 동시성**, AUTO_INCREMENT 순서 보장 안 됨
      - 각 모드의 특징 및 예시
        | 상황 | 0 모드 | 1 모드 | 2 모드 |
        | ---------------------------------- | ------------ | ------------ | --------- |
        | `INSERT INTO t VALUES (NULL, 'A')` | 자동 증가 락 | 래치 사용 | 래치 사용 |
        | `INSERT INTO t SELECT ...` | 자동 증가 락 | 자동 증가 락 | 래치 사용 |
        | 동시성 수준 | 낮음 | 중간 | 높음 |
        | 순차적 번호 보장 | 보장 | 대부분 보장 | 미보장 |
      - 래치(뮤텍스)
        - 정의
          1. **공유 자원에 대한 짧은 시간의 잠금**.
          2. 주로 **메모리 내 자원(버퍼, 캐시, 인덱스 페이지 등)** 보호용.
          3. **락보다 더 가볍고 빠르게 작동** → 아주 짧은 작업을 보호할 때 사용.
        - 특징
          1. 락(Lock)보다 **작업 단위가 작고, 유지 시간 짧음**.
          2. **트랜잭션 레벨에서 작동하는 락과 달리**, 래치는 내부 처리 중 발생. ex)**InnoDB에서 버퍼 풀 페이지에 접근할 때 래치 사용**.
- 5.3.2 인덱스와 잠금
  - 예시
    - employees 테이블에는 first_name 칼럼만 멤버로 담긴 ix_firstname이라는 인덱스가 있다.
    - employees 테이블에서 first_name = ‘Georgi’인 사원은 전체 253명 있으며 first_name = ‘Georgi’이고 last_name=’Klassen’인 사원은 1명만 존재한다.
    - UPDATE employees SET hire_date = NOW() WHERE first_name = ‘Georgi’ AND last_name=’Klassen’ 실행 시 last_name에 대한 인덱스가 없기에 ix_firstname 인덱스에 의해 253건의 레코드 전부 잠긴다. → UPDATE 중 253건에 대해서 다른 요청이 기다려야하는 상황
    - 만약 인덱스가 존재하지 않는다면 TABLE FULL SCAN에 의해 전체 테이블 자체가 잠긴다.
- 5.3.3 레코드 수준의 잠금 확인 및 해제

  - MySQL 5.1이전 : 레코드 잠금에 대한 메타 정보(딕셔너리 테이블) 제공하지 않았다.
  - MySQL 5.1 ~ MySQL 8.0 이전 : information_schema DB에 INNODB_TRX라는 테이블과 INNODB_LOCKS, INNODB_LOCK_WAITS라는 테이블로 확인 가능
  - MySQL 8.0이후 : information_schema 정보들이 조금씩 deprecated되고 performance_schema의 data_locks와 data_lock_waits 테이블로 대체
  - 예시
    | 커넥션1 | 커넥션2 | 커넥션3 |
    | ----------------------------------------- | ---------------- | ---------------- |
    | BEGINS; | | |
    | UPDATE employees |
    | SET birth_date = NOW() WHERE empno=10001; | | |
    | | UPDATE employees |
    | SET birth_date = NOW() WHERE empno=10001; | |
    | | | UPDATE employees |
    | SET birth_date = NOW() WHERE empno=10001; |

    ```jsx
    mysql > SHOW PROCESSLIST;

    -|----|------|----------|---------------------------------------------------------------|
     | Id | Time | State    |                                                               |
    -|----|------|----------|---------------------------------------------------------------|
     | 17 |  607 |          | NULL                                                          |
     | 18 |   22 | updating | UPDATE employees SET birth_date = NOW() WHERE empno=10001;    |
     | 19 |   21 | updating | UPDATE employees SET birth_date = NOW() WHERE empno=10001;    |
    -|----|------|----------|---------------------------------------------------------------|
    ```

    - 17번 스레드가 Update가 완료되었으나 COMMIT 실행하지 않았으므로 업데이트한 레코드의 잠금을 그대로 가지고 있다. → 18, 19번 스레드는 잠금 대기로 인해 Update 명령어를 실행 중
    - 18번 스레드는 17번 스레드를 기다리고 19번 스레드는 17, 18번 스레드를 기다리고 있는 상태 → 17번 스레드가 가지고 있는 잠금 해제하고 18번 스레드가 잠금 획득 이후 UPDATE 완료한 후에 잠금 풀어야 19번이 비로소 실행 가능
    - KILL 17 번 명령어로 17번이 만약 잠금을 너무 오래동안 가지고 있다면 강제 종료 명령으로 나머지 UPDATE 수행하게 하면서 잠금 경합을 해결

# 5.4 MySQL의 격리 수준

- Intro
  - MySQL의 격리 수준은 크게 ‘READ UNCOMMITED’, ‘READ COMMITED’, ‘REPEATABLE READ’, ‘SERIALIZABLE’ 4가지 존재
  - 뒤로 갈 수록 격리(고립)정도 높아지고 동시 처리 성능은 떨어진다.
    | | DIRTY READ | NON-REPEATABLE READ | PHANTOM READ |
    | --------------- | ---------- | ------------------- | ------------------- |
    | READ UNCOMMITED | 발생 | 발생 | 발생 |
    | READ COMMITED | 없음 | 발생 | 발생 |
    | REPEATABLE READ | 없음 | 없음 | 발생(InnoDB는 없음) |
    | SERIALIZABLE | 없음 | 없음 | 없음 |
  - Oracle 기본값 : READ COMMITED / MySQL 기본값 : REPEATABLE READ
  - InnoDB의 REPEATABLE READ인 경우 PHANTOM READ가 발생하지 않는 이유 : 넥스트 키 락(Next-Key Lock)
    - 넥스트 키 락(Next-Key Lock) : 레코드 + 앞의 간격(갭)을 잠금해서 새로운 레코드 삽입 차단
    - REPEATABLE READ + 넥스트 키 락(Next-Key Lock) : 다른 트랜잭션의 INSERT 막음 → 팬텀 리드 방지
- 5.4.1 READ UNCOMMITED
  - 정의 : 각 트랜잭션의 변경 내용이 COMMIT이나 ROLLBACK 여부에 관계없이 다른 트랜잭션에 조회된다.
  - 문제점 : 알 수 없는 문제로 인해 해당 트랜잭션에서 Insert한 것을 Rollback해도 다른 트랜잭션에서 Insert된 것이 정상적이라고 생각해 계속 처리하는 문제가 발생
  - Dirty Read : 위의 상황과 같이 어떤 트랜잭션에서 처리한 작업이 완료되지 않았는데도 다른 트랜잭션에 조회할 수 있는 현상 → 데이터가 나타났다가 사라졌다 하는 현상이 초래해 사용자와 개발자를 혼란스럽게 함
  - READ UNCOMMITED는 RDBMS 표준에서 트랜잭션의 격리 수준으로 인정하지 않을 정도로 정합성에서 문제가 많다.
- 5.4.2 READ COMMITED

  - 정의 : 오라클에서 기본적으로 사용되는 격리 수준, 어떤 트랜잭션에서 데이터 변경 시 Commit 완료된 데이터만 다른 트랜잭션에서 조회 가능 → Dirty Read 발생 X
    | 조회 상황 | 데이터 출처 |
    | ---------------------------- | ---------------------------------------------------------------------------- |
    | 커밋되지 않은 데이터 조회 | 조회 불가능(Dirty Read 방지) |
    | 변경된 데이터 Commit 완료 전 | 현재 트랜잭션은 변경 데이터 사용, 다른 트랜잭션은 Undo 영역 백업 데이터 사용 |
    | Commit 완료된 데이터 조회 | Undo 영역 아닌 실제 테이블(데이터 파일)에서 조회 가능 |
  - 예시 흐름

    1. 트랜잭션 A

       ```jsx
       UPDATE accounts SET balance = 1000 WHERE id = 1;
       -- 아직 커밋 안함 → Undo 영역에 이전 balance 백업됨
       ```

    2. 트랜잭션 B

       ```jsx
       SELECT balance FROM accounts WHERE id = 1;
       -- Undo 영역에서 백업된 balance 읽음 (트랜잭션 A가 커밋 전이므로)
       ```

    3. 트랜잭션 A 커밋 후 : Undo 영역 아닌 실제 테이블에서 최신 값 읽음

       ```jsx
       트랜잭션 B

       SELECT balance FROM accounts WHERE id = 1;
       -- 실제 테이블에서 balance 읽음
       ```

  - NON-REPEATABLE READ

    - 예시

      1. 사용자 B

         ```jsx
         BEGIN
         SELECT * FROM employess WHERE first_name = 'Toto';
         -- 결과 없음
         ```

         | empno | first_name |
         | ----- | ---------- |
         | 1     | Kim        |
         | 2     | Park       |

      2. 사용자 A

         ```jsx
         BEGIN
         UPDATE SET first_name = 'Toto'
         COMMIT
         ```

         | empno | first_name |
         | ----- | ---------- |
         | 1     | Toto       |
         | 2     | Park       |

      3. 사용자 B

         ```jsx
         SELECT * FROM employess WHERE first_name = 'Toto';
         -- 1건 반환
         ```

         | empno | first_name |
         | ----- | ---------- |
         | 1     | Toto       |
         | 2     | Park       |

    - 정의 : 하나의 트랜잭션에서 **같은 쿼리를 두 번 실행했는데 결과가 다르게 나오는 현상**. 이는 다른 트랜잭션이 **중간에 데이터를 변경하고 커밋했기 때문**. → 읽기 일관성(Read Consistency)이 깨진 상태

- 5.4.3 REPEATABLE READ

  - 정의 : MySQL InnoDB 스토리지 엔진의 기본 격리 수준. 한 트랜잭션 내 같은 쿼리 반복 실행해도 항상 같은 결과 반환
  - 작동 방식 : MVCC 위해 언두 영역에 백업된 이전 데이터를 이용해 동일 트랜잭션 내에서 동일한 결과 반환
  - 모든 InnoDB 트랜잭션은 고유한 트랜잭션 번호(순차적 증가)를 가지며 언두 영역에 백업된 데이터는 트랜잭션 번호를 포함, InnoDB가 정해진 주기로 언두 영역 백업 데이터 삭제
  - 예시

    1. 사용자 B

       ```jsx
       BEGIN(TRX-ID : 10)
       SELECT * FROM employess WHERE empno= 1;
       -- 1건 반환(Kim)

       -- 이 시점에서 트랜잭션 10은 트랜잭션 6까지의 커밋된 데이터만 조회 가능.
       -- 이후 변경사항은 Undo 영역의 백업 데이터 기준으로 처리됨.
       ```

       - employess 테이블
         | TRX-ID | empno | first_name |
         | ------ | ----- | ---------- |
         | 6 | 1 | Kim |
         | 6 | 2 | Park |

    2. 사용자 A

       ```jsx
       BEGIN(TRX-ID : 12)
       UPDATE SET first_name = 'Toto'
       COMMIT(TRX-ID : 12)

       -- 레코드의 최신 TRX-ID는 12, 사용자 B 입장에선 Undo에 백업된 TRX-ID 6 기준 데이터 사용.
       ```

       - employess 테이블
         | TRX-ID | empno | first_name |
         | ------ | ----- | ---------- |
         | 6 | 1 | Toto |
         | 6 | 2 | Park |
       - 언두 로그
         | TRX-ID | empno | first_name |
         | ------ | ----- | ---------- |
         | 6 | 1 | Kim |

    3. 사용자 B

       ```jsx
       SELECT * FROM employess WHERE first_name = 'Toto';
       -- 1건 반환(Kim)

       -- 사용자 B는 TRX-ID 10으로 실행 중이며, TRX-ID 12 이후의 변경사항은 무시.
       -- Undo 영역의 TRX-ID 6에 따라 여전히 Kim을 조회.
       ```

       - employess 테이블
         | TRX-ID | empno | first_name |
         | ------ | ----- | ---------- |
         | 6 | 1 | Toto |
         | 6 | 2 | Park |
       - 언두 로그
         | TRX-ID | empno | first_name |
         | ------ | ----- | ---------- |
         | 6 | 1 | Kim |

  - 핵심 메커니즘
    1. 스냅샷 관리 : 트랜잭션 시작 시점에 볼 수 있는 TRX-ID 목록 저장
    2. Undo 데이터 사용 조건 : 레코드 TRX-ID → 트랜잭션 스냅샷 기준이면 Undo 데이터 사용
    3. Undo 데이터 보존 : **InnoDB는 주기적으로 정리**, 트랜잭션 종료/슬로우 쿼리 따라 유지
    4. 일관성 유지 방식 : Undo를 통한 백업 데이터 조회로 MVCC 유지
  - Phontom Read

    - 정의 : **트랜잭션 내에서 같은 조건의 SELECT 쿼리를 두 번 실행했을 때, 처음에는 없던 새로운 레코드가 두 번째 실행 시 나타나는 현상**.
    - 원인 : 다른 트랜젹션이 중간에 INSERT로 새로운 레코드 삽입 후 커밋했기 때문 → 기존 레코드의 **변경으로 인한 Non-Repeatable Read와 다르게**, **새로운 레코드의 삽입에 의한 읽기 불일치**가 Phontom Read.
    - 예시

      1. 사용자 B

         ```jsx
         BEGIN(TRX-ID : 10)
         SELECT * FROM employess WHERE empno >= 2;
         -- 1건 반환(Park)
         ```

         - employees 테이블
           | TRX-ID | empno | first_name |
           | ------ | ----- | ---------- |
           | 6 | 1 | Kim |
           | 6 | 2 | Park |

      2. 사용자 A

         ```jsx
         BEGIN(TRX-ID : 12)
         INSERT INTO employees (3, Lee)
         COMMIT(TRX-ID : 12)
         ```

         - employees 테이블
           | TRX-ID | empno | first_name |
           | ------ | ----- | ---------- |
           | 6 | 1 | Toto |
           | 6 | 2 | Park |
           | 12 | 3 | Lee |

      3. 사용자 B

         ```jsx
         SELECT * FROM employess WHERE empno >= 2;
         -- 2건 반환(Park, Lee)
         ```

         - employees 테이블
           | TRX-ID | empno | first_name |
           | ------ | ----- | ---------- |
           | 6 | 1 | Toto |
           | 6 | 2 | Park |
           | 12 | 3 | Lee |

  - 5.4.4 SERIALIZABLE
    - 정의 : 가장 단순한 격리 수준이면서 동시에 가장 엄격하다. → 동시 처리 능력이 다른 격리 수준보다 떨어진다.
    - 일반적으로 InnoDB에서 순수한 SELECT 명령어는 아무런 레코드 잠금도 설정하지 않고 실행된다.
    - 그러나 SERIALIZABLE 경우 일기 작업도 공유 잠금(읽기 잠금)을 획득해야하고 동시에 다른 트랜잭션은 그러한 레코드 변경 못함
    - 즉 한 트랜잭션에서 읽고 쓰는 레코드는 다른 트랜잭션에서 절대 접근 X → Phantom Read까지 방지
