# § 10 실행 계획

# § 10.1 통계 정보

MySQL 서버는 5.7 버전까지 테이블과 인덱스에 대한 개괄적인 정보를 가지고 실행 계획을 수립했다.
테이블 칼럼의 값들이 실제 어떻게 분포되어 있는지에 대한 정보가 없기 때문에 실행 계획의 정확도가 떨어지는 경우가 많았다.

MySQL 8.0 버전부터는 인덱스 되지 않은 칼럼들에 대해서도 데이터 분포도를 수직해서 저장하는 히스토그램(Histogram) 정보가 도입되었다.

## § 10.1.1 테이블 및 인덱스 통계 정보

통계 정보가 정확하지 않다면 전혀 엉뚱한 방향으로 쿼리를 실행할 수 있다.

### § 10.1.1.1 MySQL 서버의 통계 정보

MySQL 5.6 버전부터는 InnoDB 스토리지 엔진을 사용하는 테이블에 대한 통계 정보를 영구적으로 관리할 수 있게 되었다. (기존에는 메모리에만 저장되어 휘발성이었음)

5.6 버전 부터는 각 테이블의 통계 정보를 mysql 데이터베이스의 `innodb_index_stats` 테이블과 `innodb_table_stats` 테이블로 관리할 수 있게 되었다.

```sql
mysql> CREATE TABLE tab_test (fd1 INT, fd2 VARCHAR(20), PRIMARY KEY(fd1))
        ENGINE=InnoDB
        STATS_PERSISTENT={ DEFAULT | 0 | 1 }
```

테이블을 생성할 때 `STATS_PERSISTENT` 옵션으로 테이블 단위로 영구적인 통계 정보를 보관할지 말지 선택할 수 있다.

다음과 같은 상황에서 자동으로 테이블 통계 정보가 갱신됨:

1. 테이블이 새로 열릴 때
2. 레코드 수가 많이 바뀔 때
3. 테이블 전체 레코드 중 1/16 이상이 INSERT, UPDATE, DELETE로 변경될 경우
4. ANALYZE TABLE 명령 실행 시
5. SHOW TABLE STATUS 또는 SHOW INDEX FROM 명령 실행 시
6. InnoDB 모니터링 기능이 활성화될 때
7. innodb_stats_on_metadata = ON 상태에서 SHOW TABLE STATUS가 실행될 때

이렇게 자주 통계 정보가 갱신되면 대규모 테이블에서는 통계 수집에 시간이 걸려 쿼리 성능에 영향을 줄 수 있고, 의도하지 않게 비효율적인 실행 계획으로 바뀔 수 있으므로

`innodb_stats_auto_recalc` 의 값을 off로 설정하여 자동 갱신을 막을 수 있다.

## § 10.1.2 히스토그램

8.0 버전부터 칼럼의 데이터 분포도를 참조할 수 있게 되었다.

### § 10.1.2.1 히스토그램 정보 수집 및 삭제

히스토그램 정보는 칼럼 단위로 관리되는데,

```sql
ANALYZE TABLE ... UPDATE HISTOGRAM
```

명령어를 실행하여 수동으로 수집 및 관리된다.

```sql
mysql> ANALYZE TABLE employees.employees
    -> UPDATE HISTOGRAM ON gender, hire_date;

mysql> SELECT *
    -> FROM COLUMN_STATISTICS
    -> WHERE SCHEMA_NAME='employees'
    ->   AND TABLE_NAME='employees'\G
```

```
*************************** 1. row ***************************
SCHEMA_NAME: employees
TABLE_NAME: employees
COLUMN_NAME: gender
HISTOGRAM: {
  "buckets": [
    [1, 0.5998529796789721],
    [2, 1.0]
  ],
  "data-type": "enum",
  "null-values": 0.0,
  "collation-id": 45,
  "last-updated": "2020-08-03 03:47:45.739242",
  "sampling-rate": 0.3477368727393573,
  "histogram-type": "singleton",
  "number-of-buckets-specified": 100
}

*************************** 2. row ***************************
SCHEMA_NAME: employees
TABLE_NAME: employees
COLUMN_NAME: hire_date
HISTOGRAM: {
  "buckets": [
    ["1985-02-01", "1985-02-08", 0.009838277646869273, 28],
    ["1985-03-01", "1985-03-28", 0.02015909973873832, 28],
    ["1985-03-29", "1985-04-26", 0.03015935508730267, 29],
    ["1985-04-27", "1985-05-25", 0.0399975822759954, 28],
    ...
    ["1997-01-24", "1997-06-25", 0.9700118824643023, 153]
  ]
}
```

수집된 히스토그램 정보는 시스템 딕셔녀리에 저장되고,  
MySQL 서버가 시작될 때 `information_schema` 데이터 베이스의 `column_statistics` 테이블에 로드된다.

히스토그램의 종류

- Singleton: 칼럼값 개별로 레코드 건수를 관리. Value-Based 또는 도수 분포라고 불림
- Equi-Height: 칼럼값의 범위를 균등한 개수로 구분해서 관리 Height Balanced 히스토그램이라고도 불림.

![](/Section.%2010.1%20-%2010.2/이주현/images/10.1.png)
![](/Section.%2010.1%20-%2010.2/이주현/images/10.2.png)

생성된 히스토그램은

```sql
mysql> ANALYZE TABLE employees.employees
       DROP HISTOGRAM ON gender, hire_date;
```

로 삭제할 수 있다.  
히스토그램의 삭제는 테이블의 데이터 참조가 아닌 딕셔너리 내용 삭제로 이뤄지기 때문에 다른 쿼리의 성능에 영향을 주지 않고 즉시 완료된다.

히스토그램을 삭제하지 않고 MySQL 옵티마이저가 히스토그램을 사용하지 않게 하려면
`optimizer_switch` 시스템 변수의 값을 변경하면 된다.

```sql
mysql> SET GLOBAL optimizer_switch='condition_fanout_filter = off';

-- // 현재 커넥션에서 실행되는 쿼리만 히스토그램을 사용하지 않게 설정
mysql> SET SESSION optimizer_switch='condition_fanout_filter=off';

-- // 현재 쿼리만 히스토그램을 사용하지 않게 설정
mysql> SELECT /*+ SET_VAR(optimizer_switch='condition_fanout_filter=off') */ *
       FROM ...

```

### § 10.1.2.2 히스토그램의 용도

히스토그램이 도입되기 이전 MySQL 서버가 가지고 있던 통계 정보는 카디널리티 정도였다.  
예를 들어, 테이블의 레코드가 1000건이고 어떤 칼럼의 유니크한 값 개수가 100개였다면 MySQL 서버는 이 칼럼에 대해 특정 값 동등 비교를 하면 대략 10개의 레코드가 일치할 것으로 예측한다.

하지만 실제 응용 프로그램의 데이터는 항상 균등한 분포를 가지지 않는다.

히스토그램은 특정 칼럼이 가지는 모든 값에 대한 분포 정보를 갖지는 않지만, 각 범위(버킷) 별로 레코드 건수와 유니크한 값의 개수 정보를 갖기 때문에 훨씬 정확한 예측을 할 수 있다.

```sql
mysql> EXPLAIN
       SELECT *
       FROM employees
       WHERE first_name='Zita'
       AND birth_date BETWEEN '1950-01-01' AND '1960-01-01';
```

```
+----+-------------+-----------+------+---------------+------+----------+
| id | select_type | table     | type | key           | rows | filtered |
+----+-------------+-----------+------+---------------+------+----------+
|  1 | SIMPLE      | employees | ref  | ix_firstname  | 224  | 11.11    |
+----+-------------+-----------+------+---------------+------+----------+
```

```sql
mysql> ANALYZE TABLE employees
         UPDATE histogram ON first_name, birth_date;

mysql> EXPLAIN
SELECT *
  FROM employees
 WHERE first_name='Zita'
   AND birth_date BETWEEN '1950-01-01' AND '1960-01-01';
```

```
+----+-------------+-----------+------+---------------+------+----------+
| id | select_type | table     | type | key           | rows | filtered |
+----+-------------+-----------+------+---------------+------+----------+
|  1 | SIMPLE      | employees | ref  | ix_firstname  | 224  | 60.82    |
+----+-------------+-----------+------+---------------+------+----------+
```

```sql
mysql> SELECT
    ->   SUM(CASE WHEN birth_date between '1950-01-01' and '1960-01-01' THEN 1 ELSE 0 END)
    ->   / COUNT(*) AS ratio
    -> FROM employees WHERE first_name='Zita';
```

```
+--------+
| ratio  |
+--------+
| 0.6384 |
+--------+
```

실제 비율: 63.84%, 약 143명  
히스토그램을 사용한 실행 계획이 실제 값과 더 근접함

조인의 드라이빙 테이블을 결정할 때도 히스토그램을 활용할 수 있다.

### § 10.1.2.3 히스토그램과 인덱스

MySQL 서버에서는 쿼리의 실행 계획을 수립할 때 사용 가능한 인덱스들로부터 조건절에 일치하는 레코드 건수를 대략 파악하고 최종적으로 가장 나은 실행 계획을 선택한다

이 때 조건절에 일치하는 레코드 건수를 예측하기 위해 옵티마이저는 실제 인덱스의 B-Tree를 샘플링 해서 살펴본다. 이 작업을 "인덱스 다이브(Index Dive)"라고 한다.

MySQL 서버에서는 인덱스된 칼럼을 검색 조건으로 사용하는 경우 그 칼럼의 히스토그램은 사용하지 않고 실제 인덱스 다이브를 통해 직접 수집한 정보를 활용한다.  
이는 실제 검색 조건의 대상 값에 대한 샘플링을 실행하는 것이므로 항상 히스토그램보다 정확한 결과를 기대할 수 있기 때문이다.

## § 10.1.3 코스트 모델

MySQL 서버가 쿼리를 처리하려면 다음과 같은 작업을 필요로 한다.

- 디스크로부터 데이터 페이지 읽기
- 메모리(InnoDB 버퍼 풀)로부터 데이터 페이지 읽기
- 인덱스 키 비교
- 레코드 평가
- 메모리 임시 테이블 작업
- 디스크 임시 테이블 작업

전체 쿼리의비용을 계산하는 데 필요한 단위 작업들의 비용을 코스트 모델(Cost Model)이라고 한다.

MySQL 5.7 이전 버전까지는 이런 작업들의 비용을 소스코드에 상수화 해서 사용했다.

5.7 버전부터 각 단위 작업의 비용을 DBMS 관리자가 조정할 수 있게 되었으나 인덱스가 없는 칼럼의 데이터 분포나 메모리에 상주 중인 페이지의 비율 등 비용 계산과 연관된 정보가 부족했다.

8.0 버전부터 칼럼의 히스토그램과 인덱스 별 메모리에 적재된 페이지 비율이 관리되고 실행 계획에 사용되기 시작했다.

MySQL 서버 8.0의 코스트 모델은 다음 2개 테이블에 저장돼 있는 설정 값을 사용한다.
두 테이블 모두 `mysql` 데이터베이스에 존재한다.

- `server_cost`: 인덱스를 찾고 레코드를 비교하고 임시 테이블 처리에 대한 비용 관리
- `engine_cost`: 레코드를 가진데이터 페이지를 가져오는 데 필요한 비용 관리

| **구분**    | **cost_name**                | **default_value** | **설명**                         |
| ----------- | ---------------------------- | ----------------- | -------------------------------- |
| engine_cost | io_block_read_cost           | 1.00              | 디스크 데이터 페이지 읽기        |
|             | memory_block_read_cost       | 0.25              | 메모리 데이터 페이지 읽기        |
|             | disk_temptable_create_cost   | 20.00             | 디스크 임시 테이블 생성          |
|             | disk_temptable_row_cost      | 0.50              | 디스크 임시 테이블의 레코드 읽기 |
| server_cost | key_compare_cost             | 0.05              | 인덱스 키 비교                   |
|             | memory_temptable_create_cost | 1.00              | 메모리 임시 테이블 생성          |
|             | memory_temptable_row_cost    | 0.10              | 메모리 임시 테이블의 레코드 읽기 |
|             | row_evaluate_cost            | 0.10              | 레코드 비교                      |

MySQL 서버에서 각 실행 계획의 계산된 비용(Cost)은 다음과 같이 확인할 수 있다.

```sql
mysql> EXPLAIN FORMAT=TREE
    -> SELECT *
    -> FROM employees WHERE first_name='Matt' \G
```

```
*************************** 1. row ***************************
EXPLAIN: -> Index lookup on employees using ix_firstname (first_name='Matt')
           (cost=256.10 rows=233)
```

<br>

---

<br>

코스트 모델에서 중요한 것은 각 단위 작업에 설정되는 비용 값이 커지면 어떤 실행 계획들이 고비용으로 바뀌고 어떤 실행 계획들이 저비용으로 바뀌는지를 파악하는 것이다.

대표적으로 각 단위 작업의 비용이 변경되며 예상할 수 있는 결과들은 다음과 같다.

- `key_compare_cost` 비용을 높이면 MySQL 서버 옵티마이저가 가능하면 정렬을 수행하지 않는 방향의 실행 계획을 선택할 가능성이 높아진다.

- `row_evaluate_cost` 비용을 높이면 풀 스캔을 실행하는 쿼리들의 비용이 높아지고, MySQL 서버 옵티마이저는 가능하면 인덱스를 기반으로 하는 실행 계획을 선택할 가능성이 높아진다.

- `disk_temptable_create_cost`와 `disk_temptable_row_cost` 비용을 높이면 MySQL 옵티마이저는 디스크에 임시 테이블을 만들지 않는 방향의 실행 계획을 선택할 가능성이 높아진다.

- `memory_temptable_create_cost`와 `memory_temptable_row_cost` 비용을 높이면 MySQL 서버 옵티마이저는 메모리 임시 테이블을 만들지 않는 방향의 실행 계획을 선택할 가능성이 높아진다.

- `io_block_read_cost` 비용이 높아지면 MySQL 서버는 쿼리를 수행하는 동안 InnoDB 버퍼 풀에 데이터 페이지가 적재되지 않을 경우 디스크에서 읽어야 하므로 디스크 접근이 줄어드는 방향을 선택할 가능성이 높아진다.

- `memory_block_read_cost` 비용이 높아지면 MySQL 서버는 InnoDB 버퍼 풀에 적재된 데이터 페이지가 상대적으로 적다고 하더라도 그 인덱스를 사용할 가능성이 높아진다.

# § 10.2 실행 계획 확인

MySQL 서버의 실행 계획은 `DESC` 또는 `EXPLAIN` 명령으로 확인할 수 있다.  
MySQL 8.0 버전부터는 `EXPLAIN` 명령에 사용할 수 있는 새로운 옵션이 추가되었고, 실행 계획의 **출력 포맷**과 **실제 쿼리 실행 결과까지 확인**할 수 있는 기능도 구분해서 사용할 수 있다.

---

## § 10.2.1 실행 계획 출력 포맷

기존에는 `EXPLAIN EXTENDED`, `EXPLAIN PARTITIONS` 등 명령이 구분되었으나  
MySQL 8.0부터는 모든 내용이 통합되어 `PARTITIONS`와 `EXTENDED` 옵션은 제거되었고,  
`FORMAT` 옵션을 사용하여 실행 계획의 출력 방식(JSON, TREE) 을 선택할 수 있다.

#### 기본 테이블 포맷

```sql
mysql> EXPLAIN
       SELECT *
       FROM employees e
       INNER JOIN salaries s ON s.emp_no = e.emp_no
       WHERE first_name = 'ABC';
```

#### 트리 포맷

```sql
mysql> EXPLAIN FORMAT=TREE
       SELECT *
       FROM employees e
       INNER JOIN salaries s ON s.emp_no = e.emp_no
       WHERE first_name='ABC'\G
```

결과 예시:

```
EXPLAIN: -> Nested loop inner join (cost=2.40 rows=10)
   -> Index lookup on e using ix_firstname (cost=0.35 rows=1)
   -> Index lookup on s using PRIMARY (cost=2.05 rows=10)
```

#### JSON 포맷

```sql
mysql> EXPLAIN FORMAT=JSON
       SELECT *
       FROM employees e
       INNER JOIN salaries s ON s.emp_no = e.emp_no
       WHERE first_name='ABC'\G
```

JSON 결과는 `query_block`, `cost_info`, `nested_loop`, `used_key_parts` 등을 포함하며
MySQL 옵티마이저가 수립한 실행 계획의 구조적 흐름을 표현하는 데 효과적이다.

---

### 10.2.2 쿼리의 실행 시간 확인

MySQL 8.0.18부터 `EXPLAIN ANALYZE` 기능이 추가되어,
쿼리 실행 계획뿐만 아니라 각 단계별 소요 시간(actual time) 과 처리된 레코드 수(rows), 반복 횟수(loops) 를 확인할 수 있다.

```sql
mysql> EXPLAIN ANALYZE
       SELECT e.emp_no, AVG(s.salary)
       FROM employees e
       INNER JOIN salaries s ON s.emp_no = e.emp_no
       WHERE s.salary > 50000
       AND s.from_date >= DATE('1990-01-01')
       AND s.to_date <= DATE('1990-01-01')
       AND e.first_name = 'Matt'
       GROUP BY e.hire_date;
```

#### 실행 계획 단계 예시

```
A) Table scan on <temporary>
B) Aggregate using temporary table
C) Nested loop inner join
D) Index lookup on e using ix_firstname
E) Filter: salary 조건, 날짜 조건
F) Index lookup on s using PRIMARY
```

#### 실제 실행 순서

1. D) Index lookup on e using ix_firstname
2. F) Index lookup on s using PRIMARY
3. E) 필터
4. C) Nested loop join
5. B) 집계 (GROUP BY)
6. A) 임시 테이블 읽기

> 같은 레벨이면 위에 있는 라인 먼저 실행
> 다른 레벨이면 가장 안쪽에 위치한 라인부터 실행

---

#### 결과 해석 예시 (라인 F 기준)

- `actual time=0.007..0.009`: 평균 0.009ms 소요, 첫 레코드 찾는 데 0.007ms
- `rows=10`: 평균 10개 레코드 반환
- `loops=233`: 해당 인덱스 루프가 233회 반복됨
