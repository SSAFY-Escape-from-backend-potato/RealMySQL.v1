# § 5.1 트랜잭션

트랜잭션은 꼭 여러 개의 변경 작업을 수행하는 쿼리가 조합됐을 때만 의미 있는 개념은 아니다.

```sql
mysql> CREATE TABLE tab_myisam ( fdpk INT NOT NULL, PRIMARY KEY (fdpk) ) ENGINE=MYISAM;
mysql> INSERT INTO tab_myisam (fdpk) VALUES (3);

mysql> CREATE TABLE tab_innodb ( fdpk INT NOT NULL, PRIMARY KEY (fdpk) ) ENGINE=INNODB;
mysql> INSERT INTO tab_innodb (fdpk) VALUES (3);

```

```sql
-- // AUTO-COMMIT 활성화
mysql> SET autocommit=ON;

mysql> INSERT INTO tab_myisam (fdpk) VALUES (1),(2),(3);
mysql> INSERT INTO tab_innodb (fdpk) VALUES (1),(2),(3);

```

```sql
mysql> INSERT INTO tab_myisam (fdpk) VALUES (1),(2),(3);
ERROR 1062 (23000): Duplicate entry '3' for key 'PRIMARY'

mysql> INSERT INTO tab_innodb (fdpk) VALUES (1),(2),(3);
ERROR 1062 (23000): Duplicate entry '3' for key 'PRIMARY'

mysql> SELECT * FROM tab_myisam;
+------+
| fdpk |
+------+
|    1 |
|    2 |
|    3 |
+------+

mysql> SELECT * FROM tab_innodb;
+------+
| fdpk |
+------+
|    3 |
+------+

```

MyISAM 스토리지 엔진은 트랜잭션을 지원하지 않기 때문에 ‘3’을 저장하는 순간에 중복 키 오류가 발생해도 이미 INSERT 된 ‘1’과 ‘2’를 그대로 두고 쿼리 실행을 종료한다.

이러한 부분 업데이트 현상은 테이블 데이터의 정합성을 맞추는데 상당히 어려운 문제를 만들어 낸다.

## 주의 사항

트랜잭션의 범위를 최소화해야 함

ex.

```sql
1. 처리 시작
 ⇒ 데이터베이스 커넥션 생성
 ⇒ 트랜잭션 시작

2. 사용자의 로그인 여부 확인

3. 사용자의 글쓰기 내용의 오류 여부 확인

4. 첨부로 업로드된 파일 확인 및 저장

5. 사용자의 입력 내용을 DBMS에 저장

6. 첨부 파일 정보를 DBMS에 저장

7. 저장된 내용 또는 기타 정보를 DBMS에서 조회

8. 게시물 등록에 대한 알림 메일 발송

9. 알림 메일 발송 이력을 DBMS에 저장
 <= 트랜잭션 종료 (COMMIT)
 <= 데이터베이스 커넥션 반납

10. 처리 완료

```

- 실제로 DBMS에 데이터를 저장하는 작업은 5번부터 시작하므로, 2, 3, 4 의 절차가 아무리 빨리 처리되어도 트랜잭션에 포함시킬 필요가 없다.
데이터베이스 커넥션의 수는 제한적이기 때문에 단위 프로그램이 커넥션을 소유하는 시간이 길어질 수록 사용 가능한 여유 커넥션의 수는 줄어든다.
- 8처럼 네트워크를 통해 원격 서버와 통신하는 등의 작업은 반드시 트랜잭션 내에서 제거
프로그램이 실행되는 동안 메일 서버와 통신하지 못하는 상황이 발생한다면 웹 서버, DBMS 서버까지 위험해진다.
- 7번이 단순 조회라면 트랜잭션에서 제외해도 될 것 같다.

개선 ver.

```sql
1) 처리 시작

2) 사용자의 로그인 여부 확인

3) 사용자의 글쓰기 내용의 오류 발생 여부 확인

4) 첨부로 업로드된 파일 확인 및 저장  
 ⇒ 데이터베이스 커넥션 생성 (또는 커넥션 풀에서 가져오기)  
 ⇒ 트랜잭션 시작

5) 사용자의 입력 내용을 DBMS에 저장

6) 첨부 파일 정보를 DBMS에 저장  
 ⇒ 트랜잭션 종료 (COMMIT)

7) 저장된 내용 또는 기타 정보를 DBMS에서 조회

8) 게시물 등록에 대한 알림 메일 발송

9) 알림 메일 발송 이력을 DBMS에 저장  
 <= 트랜잭션 종료 (COMMIT)  
 <= 데이터베이스 커넥션 종료 (또는 커넥션 풀에 반납)

10) 처리 완료

```

# § 5.2 MySQL 엔진의 잠금

## 5.2.1 글로벌 락

MySQL에서 제공하는 잠금 중 가장 범위가 넓음.

한 세션에서 글로벌 락을 획득하면, 다른 세션에서 SELECT 외에 대부분의 DDL, DML 문장이 락이 해제될 때 까지 대기 상태로 남는다.

대상 테이블이나 데이터베이스가 달라도 영향을 받는다. (MySQL 서버의 모든 변경 작업 stop)

## 5.2.2 테이블 락

개별 테이블 단위로 설정되는 잠금

애플리케이션에서 사용할 필요가 거의 없음.

MyISAM 이나 MEMEORY 스토리지 엔진 사용 시 테이블에 데이터를 변경하는 쿼리가 실행될 때 발생.

InnoDB는 일반적인 DML에서 레코드/범위 락을 사용하지만 AUTO_INCREMENT, 명시적 LOCK TABLES, 메타데이터 변경 같은 상황에서는 테이블 락이 발생할 수 있다.

## 5.2.3 네임드 락

임의의 문자열에 대한 잠금

자주 사용되지 않음

애플리케이션‑레벨 상호 배제(크론 잡 동시 실행 방지 등)에 사용됨.

## 5.2.4 메타데이터 락

데이터베이스 객체(테이블이나 뷰 등)의 이름이나 구조를 변경하는 경우 획득하는 잠금.

# § 5.3 InnoDB 스토리지 엔진의 잠금

## 잠금의 종류

### 레코드 락

다른 DBMS의 레코드 락과 동일한 역할.

차이점은 레코드 자체가 아니라 인덱스의 레코드를 잠근다.

### 갭 락

레코드와 바로 인접한 레코드 사이의 간격을 잠근다.

레코드와 레코드 사이에 새로운 레코드가 INSERT 되는 것을 막기 위함.

단독으로 사용되기 보다 넥스트 키 락의 일부로 자주 사용된다.

### 넥스트 키 락

- **구성** : 레코드 락 + 바로 앞 **갭 락**  
- **언제 걸리나?**  
  - `SELECT … FOR UPDATE` / `FOR SHARE`(8.0.22 이전 `LOCK IN SHARE MODE`)  
  - `UPDATE`·`DELETE`와 같은 **락킹 읽기**  
  - 단, 검색 조건이 “유니크 인덱스 = 값”**이면 갭 락을 생략하고 **레코드 락만** 획득.
- 격리 수준 : 비‑락킹 `SELECT`를 제외하면 모든 격리 수준에서 동작한다.
  
#### 예제

```sql
-- 테이블 구조
CREATE TABLE user (
  id INT PRIMARY KEY,
  name VARCHAR(20)
) ENGINE=InnoDB;

-- 데이터
INSERT INTO user VALUES (10, 'A'), (20, 'B'), (30, 'C');

```

```sql
-- 현재 테이블 상태
+----+------+
| id | name |
+----+------+
| 10 | A    |
| 20 | B    |
| 30 | C    |
+----+------+

```

1. 유니크 검색
```sql
-- 트랜잭션 1
START TRANSACTION;
SELECT * FROM user WHERE id = 20 FOR UPDATE;

```

획득 락 → 레코드 락(id = 20)
허용 행위 → 다른 세션은 id = 15·25 삽입 모두 가능

2. 범위 검색
```sql
SELECT * FROM user WHERE id BETWEEN 10 AND 30 FOR UPDATE;

```

획득 락 → 레코드 락(10·20·30) + 갭 락 (‑∞,10], (10,20], (20,30]
허용 행위 → 다른 세션은 id = 35 삽입 가능, 15·25 삽입은 대기/차단
   

### 자동 증가 락

AUTO_INCREMENT 칼럼이 사용된 테이블에 동시에 여러 레코드가 INSERT 되는 경우

중복되지 않고 저장된 순서대로 증가하는 일련번호를 부여하기 위해 사용하는 테이블 수준의 잠금

AUTO_INCREMENT 락은 테이블에 하나만 존재하기 때문에 두 개의 INSERT 쿼리가 실행되면 하나의 쿼리는 락을 기다려야 함.(AUTO_INCREMENT 칼럼에 명시적으로 값을 주어도 락이 필요함)

5.1 이상의 버전부터는 innodb_autoinc_lock_mode 라는 시스템 변수를 이용하여

자동 증가 락 대신 훨씬 가볍고 빠른 래치(뮤텍스)를 이용하도록 설정 가능

## 인덱스와 잠금

# § 5.4 MySQL 의 격리 수준

저번 스터디 때 정리해서

간략하게 표로 작성

| 격리 수준 | Dirty Read | Non-Repeatable Read | Phantom Read | 해결 방식 (MySQL/InnoDB 기준) |
| --- | --- | --- | --- | --- |
| Read Uncommitted | ❌ 발생함 | ❌ 발생함 | ❌ 발생함 | 아무런 차단 안 함. 모든 트랜잭션 변화가 즉시 보임 |
| Read Committed | ✅ 차단 | ❌ 발생함 | ❌ 발생함 | MVCC로 커밋된 데이터만 보여줌 |
| Repeatable Read | ✅ 차단 | ✅ 차단 | ❌ 발생함 | MVCC로 같은 스냅샷 유지. 팬텀은 갭 락 없으면 발생 |
| Repeatable Read<br>+ Next-Key Lock | ✅ 차단 | ✅ 차단 | (단순 SELECT 기준) 발생 가능<br/>(락킹 읽기) 차단 | InnoDB Repeatable Read는 락킹 읽기(SELECT … FOR UPDATE/LOCK IN SHARE MODE) 에 대해 넥스트 키 락으로 팬텀을 차단한다. 단순 SELECT에서는 팬텀 발생 가능. |
| Serializable | ✅ 차단 | ✅ 차단 | ✅ 차단 | 모든 SELECT에 범위 또는 레코드 락 부여 |
- Dirty Read: 다른 트랜잭션이 커밋하지 않은 데이터가 보이는 현상
- Non-Repeatable Read: 같은 쿼리를 두 번 했을 때 값이 달라지는 현상(다른 트랜잭션이UPDATE/DELETE)
- Phantom Read: 같은 조건의 쿼리를 반복했을 때 처음에 없던 레코드가 생기는 현상(다른 트랜잭션이 INSERT)
