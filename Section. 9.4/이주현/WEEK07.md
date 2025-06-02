# § 9.4 쿼리 힌트

MySQL 옵티마이저가 항상 최선의 실행 계획을 선택하지 않을 수 있기 때문에, 개발자가 쿼리 힌트를 이용해 실행 계획에 직접 개입할 수 있다.

## § 9.4.1 인덱스 힌트

### § 9.4.1.1 `STRAIGHT_JOIN`

- 조인 순서를 옵티마이저에게 맡기지 않고, FROM 절에 기술된 순서대로 조인을 수행하게 한다.
- 주로 드라이빙 테이블을 명시적으로 지정하고 싶을 때 사용.

```sql
SELECT * FROM A STRAIGHT_JOIN B ON A.id = B.id;
```

---

### § 9.4.1.2 `USE INDEX` / `FORCE INDEX` / `IGNORE INDEX`

- USE INDEX: 옵티마이저에게 특정 인덱스를 선호하도록 유도
- FORCE INDEX: 지정된 인덱스를 강제 사용
- IGNORE INDEX: 특정 인덱스를 무시

```sql
SELECT * FROM emp USE INDEX (idx_emp_no) WHERE emp_no = 1001;
SELECT * FROM emp FORCE INDEX (idx_emp_no) WHERE emp_no = 1001;
SELECT * FROM emp IGNORE INDEX (idx_emp_no) WHERE emp_no = 1001;
```

---

### § 9.4.1.3 `SQL_CALC_FOUND_ROWS`

- LIMIT을 사용해 일부만 조회하더라도, 조건에 일치하는 전체 결과 수를 함께 구함
- 쿼리 성능 저하 가능성 있음

```sql
SELECT SQL_CALC_FOUND_ROWS * FROM emp WHERE gender = 'M' LIMIT 10;
SELECT FOUND_ROWS();
```

---

## § 9.4.2 옵티마이저 힌트

### § 9.4.2.1 옵티마이저 힌트 종류

- MySQL 5.7부터 지원되는 방식
- `/*+ 힌트 */` 형식으로 쿼리 내에 삽입
- 인덱스 힌트보다 우선순위가 높음

```sql
SELECT /*+ NO_MERGE(emp) */ * FROM emp WHERE ...;
```

---

### § 9.4.2.2 `MAX_EXECUTION_TIME`

- 해당 쿼리의 최대 실행 시간을 밀리초 단위로 제한
- 초과 시 쿼리 강제 종료

```sql
SELECT /*+ MAX_EXECUTION_TIME(1000) */ * FROM emp WHERE ...;
```

---

### § 9.4.2.3 `SET_VAR`

- 쿼리 단위로 세션 변수를 설정할 수 있음

```sql
SELECT /*+ SET_VAR(sort_buffer_size = 262144) */ * FROM emp ORDER BY name;
```

---

### § 9.4.2.4 `SEMIJOIN` & `NO_SEMIJOIN`

- `IN`, `EXISTS` 등 반정합(semijoin) 처리를 사용 여부를 지정

```sql
SELECT /*+ SEMIJOIN(@subq1) */ * FROM emp WHERE emp_no IN (SELECT emp_no FROM dept);
```

---

### § 9.4.2.5 `SUBQUERY`

- 서브쿼리를 서브쿼리로 유지할지, 인라인 뷰나 조인으로 풀지 여부 결정

```sql
SELECT /*+ SUBQUERY(@subq1) */ * FROM emp WHERE ...;
```

---

### § 9.4.2.6 `BNL` / `NO_BNL` / `HASHJOIN` / `NO_HASHJOIN`

- BNL: Block Nested Loop 조인 사용
- HASHJOIN: 해시 조인 사용
- 버전 및 설정에 따라 일부 기능은 사용 불가할 수 있음

```sql
SELECT /*+ BNL(emp) */ * FROM emp JOIN dept ON emp.dept_no = dept.dept_no;
```

---

### § 9.4.2.7 `JOIN_FIXED_ORDER` / `JOIN_ORDER` / `JOIN_PREFIX` / `JOIN_SUFFIX`

- 조인 순서를 지정하거나 제한함

```sql
SELECT /*+ JOIN_ORDER(t1, t2, t3) */ ...;
SELECT /*+ JOIN_PREFIX(t1, t2) */ ...;
```

---

### § 9.4.2.8 `MERGE` / `NO_MERGE`

- 뷰나 파생 테이블을 병합할지 여부를 지정

```sql
SELECT /*+ NO_MERGE(v_emp) */ * FROM v_emp WHERE emp_no = 1001;
```

---

### § 9.4.2.9 `INDEX_MERGE` / `NO_INDEX_MERGE`

- 복수 인덱스를 병합해서 사용하는 기능의 사용 여부 지정

```sql
SELECT /*+ NO_INDEX_MERGE(emp) */ * FROM emp WHERE col1 = 1 OR col2 = 2;
```

---

### § 9.4.2.10 `NO_ICP`

- Index Condition Pushdown 기능을 비활성화
- 인덱스 조건을 테이블 스토리지 엔진으로 넘기지 않음

```sql
SELECT /*+ NO_ICP(emp) */ * FROM emp WHERE ...;
```

---

### § 9.4.2.11 `SKIP_SCAN` / `NO_SKIP_SCAN`

- SKIP SCAN은 다중 컬럼 인덱스의 첫 번째 컬럼에 조건이 없더라도,
  두 번째 이후 컬럼 조건을 사용해서 인덱스를 이용할 수 있도록 하는 기능이다.
- 옵티마이저가 판단하여 자동 적용되지만, 명시적으로 제어할 수 있다.

| 힌트                  | 설명                               |
| --------------------- | ---------------------------------- |
| `SKIP_SCAN(table)`    | 인덱스 스킵 스캔을 사용하도록 유도 |
| `NO_SKIP_SCAN(table)` | 스킵 스캔을 사용하지 않도록 제한   |

```sql
SELECT /*+ SKIP_SCAN(emp) */ * FROM emp WHERE last_name = 'Kim';
SELECT /*+ NO_SKIP_SCAN(emp) */ * FROM emp WHERE last_name = 'Kim';
```

SKIP SCAN은 B-Tree 인덱스를 다차원 인덱스처럼 사용하는 기술로, 항상 성능이 좋은 것은 아님. 드라이빙 범위가 너무 넓어져 느려질 수 있다.

---

### § 9.4.2.12 `INDEX` / `NO_INDEX`

- 특정 쿼리에서 사용할 인덱스를 명시하거나, 특정 인덱스는 사용하지 않도록 지정할 수 있다.
- 인덱스 힌트(`USE INDEX`, `IGNORE INDEX`)와 유사하지만, 옵티마이저 힌트 문법으로 표현된다는 점에서 다르다.
- 인덱스 힌트와 옵티마이저 힌트가 함께 존재하면 옵티마이저 힌트가 **우선 적용**된다.

```sql
SELECT /*+ INDEX(emp idx_emp_no) */ * FROM emp WHERE emp_no = 1001;
SELECT /*+ NO_INDEX(emp idx_emp_name) */ * FROM emp WHERE emp_name = 'Lee';
```

실수로 잘못된 인덱스를 지정하면 실행 계획에서 무시되며, 경고 로그에 표시된다.
