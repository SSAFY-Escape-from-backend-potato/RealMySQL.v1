# § 10.3 실행 계획 분석

MySQL 8.0 버전 부터는 EXPLAIN 의 결과로 출력되는 실행 계획의 포맷을 기존 테이블 포맷, JSON, TREE 형태로 선택할 수 있다.

```sql
EXPLAIN
SELECT *
FROM employees e
INNER JOIN salaries s ON s.emp_no = e.emp_no
WHERE first_name = 'ABC';
```

| id  | select_type | table | partitions | type | possible_keys        | key          | key_len | ref                | rows | filtered | Extra |
| --- | ----------- | ----- | ---------- | ---- | -------------------- | ------------ | ------- | ------------------ | ---- | -------- | ----- |
| 1   | SIMPLE      | e     | NULL       | ref  | PRIMARY,ix_firstname | ix_firstname | 58      | const              | 1    | 100.00   | NULL  |
| 1   | SIMPLE      | s     | NULL       | ref  | PRIMARY              | PRIMARY      | 4       | employees.e.emp_no | 10   | 100.00   | NULL  |

아무 옵션 없이 EXPLAIN 명령어를 실행하면 표 형태로 된 1줄 이상의 결과가 표시된다.  
표의 각 라인(레코드)은 쿼리 문장에서 사용된 테이블의 개수만큼 출력된다.  
출력된 실행 계획에서 위쪽에 출력된 결과일수록(id 칼럼의 값이 작을 수록) 쿼리의 Outer 부분이거나 먼저 접근한 테이블이다.

## § 10.3.1 id 칼럼

하나의 SELECT 문장은 1개 이상의 하위(SUB) SELECT 문장을 포함할 수 있다.

```sql
SELECT ...
FROM (SELECT ... FROM tb_test1) tb1, tb_test2 tb2
WHERE tb1.id = tb2.id;
```

=> 위 쿼리 문장에 있는 각 SELECT를 아래와 같이 분리해서 생각해볼 수 있다.  
SELECT 키워드 단위로 구분한 것을 '단위 쿼리'라고 표현하겠다.

```sql
mysql> SELECT ... FROM tb_test1;

mysql> SELECT ... FROM tb1, tb_test2 tb2 WHERE tb1.id = tb2.id;
```

실행 계획에서 가장 왼쪽에 표시되는 id 칼럼은 단위 SELECT 쿼리 별로 부여되는 식별자 값이다.

다음 예제에서 SELECT는 하나인데, 여러 개의 테이블이 조인되는 경우에는 id 값이 증가하지 않고 같은 id 값이 부여된다.

```sql
EXPLAIN
SELECT e.emp_no, e.first_name, s.from_date, s.salary
FROM employees e, salaries s
WHERE e.emp_no = s.emp_no
LIMIT 10;
```

| id  | select_type | table | type  | key          | ref                | rows   | Extra       |
| --- | ----------- | ----- | ----- | ------------ | ------------------ | ------ | ----------- |
| 1   | SIMPLE      | e     | index | ix_firstname | NULL               | 300252 | Using index |
| 1   | SIMPLE      | s     | ref   | PRIMARY      | employees.e.emp_no | 10     | NULL        |

<br>

---

<br>

반대로 다음 쿼리의실행 계획에서는 쿼리 문장이 3개의 단위 SELECT 쿼리로 구성되어 있으므로 각 레코드가 각기 다른 id 값을 지닌 것을 확인할 수 있다.

```sql
EXPLAIN
SELECT
  ((SELECT COUNT(*) FROM employees) + (SELECT COUNT(*) FROM departments)) AS total_count;
```

| id  | select_type | table       | type  | key         | ref  | rows   | Extra          |
| --- | ----------- | ----------- | ----- | ----------- | ---- | ------ | -------------- |
| 1   | PRIMARY     | NULL        | NULL  | NULL        | NULL | NULL   | No tables used |
| 3   | SUBQUERY    | departments | index | ux_deptname | NULL | 9      | Using index    |
| 2   | SUBQUERY    | employees   | index | ix_hiredate | NULL | 300252 | Using index    |

<br>

---

<br>

주의해야 할 것은 실행 계획의 id 칼럼이 테이블의 접근 순서를 의미하지는 않는다는 것이다.

다음 쿼리의 실행 계획을 살펴보면 dept_emp 테이블의 id 값은 1이고, employees 테이블의 id 값은 2로 표시됐다.

```sql
EXPLAIN FORMAT=TREE
SELECT * FROM dept_emp de
WHERE de.emp_no = (
  SELECT e.emp_no
  FROM employees e
  WHERE e.first_name = 'Georgi'
    AND e.last_name = 'Facello'
  LIMIT 1
);
```

| id  | select_type | table | type | key               | rows | Extra       |
| --- | ----------- | ----- | ---- | ----------------- | ---- | ----------- |
| 1   | PRIMARY     | de    | ref  | ix_empno_fromdate | 1    | Using where |
| 2   | SUBQUERY    | e     | ref  | ix_firstname      | 253  | Using where |

실제 이 쿼리의 실행 순서는 employees 테이블을 먼저 읽고, 그 결과를 이용해 dept_emp 테이블을 읽는 순서로 실행된 것이다.

## § 10.3.2 select_type 칼럼

각 단위 SELECT 쿼리가 어떤 타입의 쿼리인지 표시되는 칼럼이다.

select_type 칼럼에 표시될 수 있는 값은 다음과 같다.

- SIMPLE
- PRIMARY
- UNION
- DEPENDENT UNION
- UNION RESULT
- SUBQUERY
- DEPENDENT SUBQUERY
- DERIVED
- DEPENDENT DERIVED
- UNCACHEABLE SUBQUERY
- UNCACHEABLE UNION
- MATERIALIZED

### § 10.3.2.1 SIMPLE

UNION이나 서브쿼리를 사용하지 않는 단순한 SELECT 쿼리인 경우 해당 쿼리 문장의 select_type은 SIMPLE로 표시된다.(조인이 포함된 경우도 마찬가지)

쿼리 문장이 아무리 복잡하더라도 실행 계획에서 select_type이 SIMPLE인 단위 쿼리는 하나만 존재한다.  
일반적으로 제일 바깥 SELECT 쿼리가 SIMPLE로 표시된다.

### § 10.3.2.2 PRIMARY

UNION이나 서브쿼리를 가지는 SELECT 쿼리의 실행 계획에서 가장 바깥쪽에 있는 단위 쿼리는 select_type이 PRIMARY로 표시된다.  
SIMPLE과 마찬가지로 select_type이 PRIMARY인 단위 SELECT 쿼리는 하나만 존재한다.

### § 10.3.2.3 UNION

UNION으로 결합하는 단위 SELECT 쿼리 가운데 첫 번째를 제외한 두 번째 이후 단위 SELECT 쿼리의 select_type은 UNION으로 표시된다.  
UNION의 첫 번째 단위 SELECT는 UNION되는 쿼리 결과들을 모아서 저장하는 임시 테이블이므로 select_type이 DERIVED로 표시된다.

```sql
EXPLAIN
SELECT * FROM (
    (SELECT emp_no FROM employees e1 LIMIT 10)
    UNION ALL
    (SELECT emp_no FROM employees e2 LIMIT 10)
    UNION ALL
    (SELECT emp_no FROM employees e3 LIMIT 10)
) tb;
```

| id  | select_type | table      | type  | key         | ref  | rows   | Extra       |
| --- | ----------- | ---------- | ----- | ----------- | ---- | ------ | ----------- |
| 1   | PRIMARY     | <derived2> | ALL   | NULL        | NULL | 30     | NULL        |
| 2   | DERIVED     | e1         | index | ix_hiredate | NULL | 300252 | Using index |
| 3   | UNION       | e2         | index | ix_hiredate | NULL | 300252 | Using index |
| 4   | UNION       | e3         | index | ix_hiredate | NULL | 300252 | Using index |

### § 10.3.2.4 DEPENDENT UNION

DEPENDENT UNION 또한 UNION 이나 UNION ALL로 집합을 결합하는 쿼리에서 표시된다.  
DEPENDENT는 UNION이나 UNION ALL로 결합된 단위 쿼리가 외부 쿼리에 의해 영향을 받는 것을 의미한다.

```sql
EXPLAIN
SELECT *
FROM employees e1
WHERE e1.emp_no IN (
    SELECT e2.emp_no FROM employees e2 WHERE e2.first_name = 'Matt'
    UNION
    SELECT e3.emp_no FROM employees e3 WHERE e3.last_name = 'Matt'
);
```

| id   | select_type        | table      | type   | key     | ref  | rows   | Extra           |
| ---- | ------------------ | ---------- | ------ | ------- | ---- | ------ | --------------- |
| 1    | PRIMARY            | e1         | ALL    | NULL    | NULL | 300252 | Using where     |
| 2    | DEPENDENT SUBQUERY | e2         | eq_ref | PRIMARY | func | 1      | Using where     |
| 3    | DEPENDENT UNION    | e3         | eq_ref | PRIMARY | func | 1      | Using where     |
| NULL | UNION RESULT       | <union2,3> | ALL    | NULL    | NULL | NULL   | Using temporary |

`e1.emp_no`가 SELECT 문들의 결과에 포함되어 있는지 확인하는 쿼리  
내부 SELECT 들은 각각 employees 테이블을 다시 조회하지만, 외부 테이블인 e1의 칼럼을 사용하는 조건은 없음.

MySQL 옵티마이저는 `WHERE e1.emp_no IN (...)` 에서 내부 SELECT가 외부 쿼리의 결과에 따라 반복적으로 실행될 수 있다고 판단할 결우 DEPENDENT 로 분류함.

IN (...) 서브쿼리 자체가 반복 평가 가능성이 있으므로 최적화되지 않는 이상 DEPENDENT로 표시됨.

⚠️ 주의할 점  
DEPENDENT UNION은 쿼리 성능 저하의 원인이 될 수 있다..

UNION 내부 서브쿼리가 외부 쿼리에 종속적이라는 의미이므로, 매 행마다 내부 쿼리를 수행됨..

### § 10.3.2.5 UNION RESULT

UNION RESULT는 UNION 결과를 담아두는 테이블을 의미한다.

MySQL 8.0 이전 버전에서는 UNION ALL이나 UNION(UNION DISTINCT) 쿼리는 UNION의 결과를 임시 테이블로 생성했는데, 8.0 부터는 UNION ALL의 경우 임시 테이블을 생성하지 않도록 기능이 개선되었다.

실행 계획 상에서 이 임시 테이블을 가리키는 라인의 select_type이 UNION RESULT 이다.

UNION RESULT 는 실제 쿼리에서 단위 쿼리가 아니기 때문에 별도의 id 값은 부여되지 않는다.

```sql
EXPLAIN
SELECT emp_no FROM salaries WHERE salary > 100000
UNION DISTINCT
SELECT emp_no FROM dept_emp WHERE from_date > '2001-01-01';
```

| id   | select_type  | table      | type  | key         | rows   | Extra                    |
| ---- | ------------ | ---------- | ----- | ----------- | ------ | ------------------------ |
| 1    | PRIMARY      | salaries   | range | ix_salary   | 191348 | Using where; Using index |
| 2    | UNION        | dept_emp   | range | ix_fromdate | 5325   | Using where; Using index |
| NULL | UNION RESULT | <union1,2> | ALL   | NULL        | NULL   | Using temporary          |

=> UNION DISTINCT를 UNION ALL로 변경하면 임시 테이블을 생성하지 않으므로 UNION RESULT 라인이 없어진 것을 확인할 수 있다.

```sql
EXPLAIN
SELECT emp_no FROM salaries WHERE salary > 100000
UNION ALL
SELECT emp_no FROM dept_emp WHERE from_date > '2001-01-01';
```

| id  | select_type | table    | type  | key         | rows   | Extra                    |
| --- | ----------- | -------- | ----- | ----------- | ------ | ------------------------ |
| 1   | PRIMARY     | salaries | range | ix_salary   | 191348 | Using where; Using index |
| 2   | UNION       | dept_emp | range | ix_fromdate | 5325   | Using where; Using index |

### § 10.3.2.6 SUBQUERY

select_type의 SUBQUERY는 FROM 절 이외에서 사용되는 서브쿼리만을 의미한다.

```sql
EXPLAIN
SELECT e.first_name,
       (SELECT COUNT(*)
        FROM dept_emp de, dept_manager dm
        WHERE dm.dept_no = de.dept_no) AS cnt
FROM employees e
WHERE e.emp_no = 10001;
```

| id  | select_type | table | type  | key     | rows  | Extra       |
| --- | ----------- | ----- | ----- | ------- | ----- | ----------- |
| 1   | PRIMARY     | e     | const | PRIMARY | 1     | NULL        |
| 2   | SUBQUERY    | dm    | index | PRIMARY | 24    | Using index |
| 2   | SUBQUERY    | de    | ref   | PRIMARY | 41392 | Using index |

FROM 절에 사용된 서브쿼리는 select_type이 DERIVED로 표시되고, 그 밖의 위치에서 사용된 서브쿼리는 전부 SUBQUERY라고 표시된다.

> ### 🔎 참고
>
> 서브쿼리는 사용하는 위치에 따라 각각 다른 이름을 지니고 있다.

- **중첩된 쿼리(Nested Query)**:  
  `SELECT`되는 칼럼에 사용된 서브쿼리를 **네스티드 쿼리**라고 한다.

- **서브쿼리(Subquery)**:  
  `WHERE` 절에 사용된 경우에는 일반적으로 그냥 **서브쿼리**라고 한다.

- **파생 테이블(Derived Table)**:  
  `FROM` 절에 사용된 서브쿼리를 MySQL에서는 **파생 테이블**이라고 하며,  
  일반적으로 RDBMS에서는 **인라인 뷰(Inline View)** 또는 **서브 셀렉트(Sub Select)** 라고 부른다.

---

> 또한 서브쿼리가 반환하는 값의 특성에 따라 다음과 같이 구분하기도 한다:

- **스칼라 서브쿼리(Scalar Subquery)**:  
  하나의 값만 반환함. 단 하나의 레코드, 단 하나의 칼럼을 반환하는 쿼리

- **로우 서브쿼리(Row Subquery)**:  
  칼럼의 개수와 관계없이 **하나의 레코드만** 반환하는 쿼리

### § 10.3.2.7 DEPENDENT SUBQUERY

서브쿼리가 바깥쪽 SELECT 쿼리에서 정의된 칼럼을 사용하는 경우,
select_type에 DEPENDENT SUBQUERY라고 표시된다.

```sql
EXPLAIN
SELECT e.first_name,
       (
         SELECT COUNT(*)
         FROM dept_emp de, dept_manager dm
         WHERE dm.dept_no = de.dept_no
           AND de.emp_no = e.emp_no
       ) AS cnt
FROM employees e
WHERE e.first_name = 'Matt';
```

| id  | select_type        | table | type | key               | rows | Extra       |
| --- | ------------------ | ----- | ---- | ----------------- | ---- | ----------- |
| 1   | PRIMARY            | e     | ref  | ix_firstname      | 233  | Using index |
| 2   | DEPENDENT SUBQUERY | de    | ref  | ix_empno_fromdate | 1    | Using index |
| 2   | DEPENDENT SUBQUERY | dm    | ref  | PRIMARY           | 2    | Using index |

| 항목                        | 설명                                                                                                          |
| --------------------------- | ------------------------------------------------------------------------------------------------------------- |
| `PRIMARY`                   | 외부 쿼리: `employees` 테이블에서 `first_name = 'Matt'` 조건으로 `ix_firstname` 인덱스 활용. 예상 233건 스캔. |
| `DEPENDENT SUBQUERY` - `de` | 내부 서브쿼리에서 `e.emp_no`를 참조하므로 외부 쿼리에 **의존적**. 인덱스를 통해 `dept_emp` 스캔.              |
| `DEPENDENT SUBQUERY` - `dm` | `dept_manager` 테이블도 `de.dept_no`에 따라 조인되므로 역시 의존적. PRIMARY 키 활용.                          |
| `Using index`               | 모두 커버링 인덱스를 사용하여 테이블 자체의 row를 읽지 않아도 됨 (성능 최적화됨).                             |

- employees 테이블에서 first_name이 'Matt'인 직원들을 찾고,

- 해당 직원의 emp_no를 기준으로 dept_emp와 dept_manager를 조인하여,

- 그 직원이 속한 부서가 매니저가 있는 부서인지 확인하고 개수를 세는 서브쿼리를 실행함.

외부 쿼리가 먼저 수행된 후 내부 쿼리(서브쿼리)가 실행돼야 하므로 DEPENDENT가 없는 일반 서브쿼리보다는 처리 속도가 느릴 때가 많다.

### § 10.3.2.8 DERIVED

MySQL 5.5 버전까지는 서브쿼리가 FROM 절에 사용된 경우 항상 select_type이 DERIVED 인 실행 계획을 만든다.

하지만 MySQL 5.6 부터는 옵티마이저 옵션(optimizer_switch 시스템 변수)에 따라 FROM 절의 서브쿼리를 외부 쿼리와 통합하는 형태의 최적화가 수행되기도 한다.

DERIVED는 단위 SELECT 쿼리의 실행 결과로 메모리나 디스크에 임시 테이블(파생 테이블)이 생성됨을 의미한다.

또한 MySQL 5.5 버전까지는 파생 테이블에 인덱스가 전혀 없으므로 다른 테이블과 조인할 때 성능 상 불리할 때가 많았다.

```sql
EXPLAIN
SELECT *
FROM (SELECT de.emp_no FROM dept_emp de GROUP BY de.emp_no) tb,
     employees e
WHERE e.emp_no = tb.emp_no;
```

| id  | select_type | table      | type   | key               | rows   | Extra       |
| --- | ----------- | ---------- | ------ | ----------------- | ------ | ----------- |
| 1   | PRIMARY     | <derived2> | ALL    | NULL              | 331143 | NULL        |
| 1   | PRIMARY     | e          | eq_ref | PRIMARY           | 1      | NULL        |
| 2   | DERIVED     | de         | index  | ix_empno_fromdate | 331143 | Using index |

#### 실행 계획 설명

| 항목                     | 의미                                                                                  |
| ------------------------ | ------------------------------------------------------------------------------------- |
| `<derived2>`             | `FROM` 절 서브쿼리(파생 테이블)로 생성된 임시 테이블을 의미한다.                      |
| `DERIVED`                | 해당 서브쿼리를 실행해 임시 테이블로 만드는 단계임을 나타낸다.                        |
| `ALL`, `eq_ref`, `index` | 각각 테이블 전체 스캔, 인덱스를 통한 정확한 조인, 인덱스를 통한 순차 탐색을 의미한다. |
| `Using index`            | `de` 테이블은 인덱스만으로 처리되는 **커버링 인덱스**를 사용한다.                     |

⚠️ 파생 테이블에 대한 최적화가 부족한 버전의 MySQL 서버를 사용 중인 경우  
DERIVED 형태의 실행 계획을 조인으로 해결할 수 있게 쿼리를 바꿔주는 것이 좋다.

> ### 📌 참고
>
> 쿼리를 튜닝하기 위해 실행 계획을 확인할 때 가장 먼저 `select_type` 칼럼의 값이 `DERIVED`인 것이 있는지 확인해야 한다.  
> 서브쿼리를 조인으로 해결할 수 있는 경우라면 서브쿼리보다는 조인을 사용할 것을 강력히 권장한다.
>
> 실제로 개발 시점에 쿼리를 작성할 때 많은 개발자들이 기능을 조금씩 단계적으로 추가해 가면서 쿼리를 개발한다.  
> 이러한 개발 과정은 필요한 쿼리를 개발하기는 쉽지만 대부분 쿼리가 조인이 아니라 서브쿼리 형태로 작성된다는 단점이 있다.
>
> 물론 이런 절차로 개발하는 것이 생산성은 높지만 쿼리 성능은 떨어진다.  
> 쿼리를 서브쿼리 형태로 작성하는 것이 편하다면 반드시 마지막에도 서브쿼리를 조인으로 풀어서 고쳐 쓰는 습관을 들여야 한다.
>
> 그러면 어느 순간 서브쿼리로 작성하는 단계 없이 바로 조인으로 복잡한 쿼리를 개발할 수 있을 것이다.

### § 10.3.2.9 DEPENDENT DERIVED

MySQL 8.0 부터는 래터럴 조인(LATERAL JOIN) 기능이 추가되면서 FROM 절의 서브쿼리에서도 외부 컬럼을 참조할 수 있게 되었다.

employees 테이블의 레코드 1건 당 salaries 테이블의 레코드를 최근 순서대로 최대 2건까지만 가져와서 조인을 수행한다.

```sql
SELECT *
FROM employees e
LEFT JOIN LATERAL (
    SELECT *
    FROM salaries s
    WHERE s.emp_no = e.emp_no
    ORDER BY s.from_date DESC
    LIMIT 2
) AS s2 ON s2.emp_no = e.emp_no;
```

| id  | select_type       | table      | type | key         | Extra                      |
| --- | ----------------- | ---------- | ---- | ----------- | -------------------------- |
| 1   | PRIMARY           | e          | ALL  | NULL        | Rematerialize ((derived2)) |
| 1   | PRIMARY           | <derived2> | ref  | {auto_key0} | NULL                       |
| 2   | DEPENDENT DERIVED | s          | ref  | PRIMARY     | Using filesort             |

### 📘 설명

- `LATERAL` 조인을 사용하면 **서브쿼리 내부에서 외부 쿼리의 칼럼(`e.emp_no`)을 참조**할 수 있다.
- MySQL에서는 반드시 `LATERAL` 키워드를 사용해야 외부 컬럼 참조가 허용된다.  
  → 만약 `LATERAL`을 생략하고 외부 컬럼을 참조하면 오류 발생.
- 내부 서브쿼리에서 `ORDER BY ... LIMIT 2`와 같이 **행 순서를 제어**하는 기능을 사용할 수 있으므로  
  **N건 중 상위 몇 건 추출**과 같은 동작을 외부 테이블 기준으로 반복 수행 가능하다.

#### 실행 계획 해석

| 항목                         | 의미                                                             |
| ---------------------------- | ---------------------------------------------------------------- |
| `Rematerialize ((derived2))` | 파생 테이블을 **행마다 다시 생성**함 (비용 있음)                 |
| `DEPENDENT DERIVED`          | `s` 서브쿼리가 외부 테이블(`e`)의 값에 따라 **매번 다시 실행됨** |
| `Using filesort`             | `ORDER BY` 처리를 위해 정렬 수행됨 (인덱스만으로 정렬 불가)      |

---

### § 10.3.2.10 UNCACHEABLE SUBQUERY

하나의 쿼리 문장에 서브쿼리가 하나만 있더라도 실제 그 서브쿼리가 한 번만 실행되는 것은 아니다.

하지만 조건이 똑같은 서브쿼리가 실행될 때는 다시 실행하지 않고 이전의 실행 결과를 그대로 사용할 수 있게 서브쿼리의 결과를 내부적인 캐시 공간에 담아둘 수 있다.

지금 언급하는 서브쿼리 캐시는 쿼리 캐시나 파생 테이블(DERIVED)와는 전혀 무관한 기능이다.

SUBQUERY 는 바깥쪽의 영향을 받지 않으므로 처음 한 번만 실행해서 그 결과를 캐시하고 필요할 때 캐시된 결과를 이용한다.

![](/Section.%2010.3/이주현/images/10.4.png)

하지만 DEPENDENT SUBQUERY는 그림 10.4 처럼 캐시는 되지만, 딱 한 번만 캐시되는 것이 아니라 외부 쿼리 값 단위로 캐시가 만들어진다.

```sql
SELECT @status FROM dept_emp de ...
```

사용자 변수가 서브 쿼리에 사용되어 MySQL이 캐시를 하지 않았다.

| id  | select_type          | table | type | key     | rows   | Extra       |
| --- | -------------------- | ----- | ---- | ------- | ------ | ----------- |
| 1   | PRIMARY              | e     | ALL  | NULL    | 300252 | Using where |
| 2   | UNCACHEABLE SUBQUERY | de    | ref  | PRIMARY | 165571 | Using index |

#### ✅ 언제 UNCACHEABLE SUBQUERY가 되는가?

1. 사용자 변수가 서브쿼리에 사용된 경우

2. NOT DETERMINISTIC 함수가 서브쿼리 내에 있는 경우
   (→ 같은 입력이어도 실행할 때마다 결과가 달라지는 함수)

3. RAND(), UUID() 같이 호출할 때마다 결과가 달라지는 함수가 서브쿼리에 포함된 경우

### § 10.3.2.11 UNCACHEABLE UNION

### § 10.3.2.12 MATERIALIZED

5.6 버전부터 도입된 select_type으로, 주로 FROM 절이나 IN(subquery) 형태의 쿼리에 사용된 서브쿼리의 최적화를 위해 사용된다.

```sql
EXPLAIN
SELECT *
FROM employees e
WHERE e.emp_no IN (
    SELECT emp_no
    FROM salaries
    WHERE salary BETWEEN 100 AND 1000
);
```

급여가 100보다 크거나 같고 1000보다 적거나 같은 직원들의 정보를 모두 가져오는 쿼리이다.

5.6버전까지는 employees 테이블을 읽어서 employees 테이블의 레코드마다 salaries 테이블을 읽는 서브쿼리가 실행되는 형태로 처리되었지만, 5.7 부터는 서브쿼리의 내용을 임시 테이블로 구체화(Materialization)한 후, 임시 테이블과 employees 테이블을 조인하는 형태로 최적화되어 처리된다.

| id  | select_type  | table       | type   | key       | rows | Extra                    |
| --- | ------------ | ----------- | ------ | --------- | ---- | ------------------------ |
| 1   | SIMPLE       | <subquery2> | ALL    | NULL      | NULL | NULL                     |
| 1   | SIMPLE       | e           | eq_ref | PRIMARY   | 1    | NULL                     |
| 2   | MATERIALIZED | salaries    | range  | ix_salary | 1    | Using where; Using index |

## § 10.3.3 table 칼럼

MySQL 서버의 실행 계획은 단위 SELECT 쿼리 기준이 아니라 테이블 기준으로 표시된다.

```sql
mysql> EXPLAIN SELECT NOW();

mysql> EXPLAIN SELECT NOW() FROM DUAL;
```

"DUAL" 이라는 테이블은 없는데, 오류는 발생하지 않는다.
"FROM DUAL" 부분을 제거하고, 첫 번째 쿼리와 동일하게 변형해서 처리한다.

| id  | select_type | table | key  | key_len | Extra          |
| --- | ----------- | ----- | ---- | ------- | -------------- |
| 1   | SIMPLE      | NULL  | NULL | NULL    | No tables used |

<br>

---

<br>

table 칼럼에 <derived N>, <union M, N> 처럼 <>로 둘러싸인 부분은 임시 테이블을 의미한다.

```sql
SELECT *
FROM (
  SELECT de.emp_no FROM dept_emp de GROUP BY de.emp_no
) tb,
employees e
WHERE e.emp_no = tb.emp_no;
```

| id  | select_type | table      | type   | key               | rows   | Extra       |
| --- | ----------- | ---------- | ------ | ----------------- | ------ | ----------- |
| 1   | PRIMARY     | <derived2> | ALL    | NULL              | 331143 | NULL        |
| 1   | PRIMARY     | e          | eq_ref | PRIMARY           | 1      | NULL        |
| 2   | DERIVED     | de         | index  | ix_empno_fromdate | 331143 | Using index |

![](/Section.%2010.3/이주현/images/10.5.png)

### 실행 계획 분석 요점

1. **첫 번째 라인**

   - 테이블 이름이 `<derived2>`로 되어 있음
   - `id = 1`이지만, `id = 2`가 **먼저 실행**되어 파생 테이블이 준비되어야 함
   - → 이 라인은 **파생 테이블이 준비된 후** 사용된다는 의미

2. **두 번째 라인 (`id = 2`)**

   - `select_type`이 `DERIVED`
   - 즉, 실제로 `dept_emp` 테이블을 읽고 `GROUP BY`하여 **파생 테이블을 생성**하는 단계

3. **세 번째 라인 (`id = 1`, `table = e`)**

   - `employees` 테이블을 `PRIMARY KEY`로 `tb.emp_no`와 **조인**함
   - `eq_ref`이므로 매우 빠른 키 조인 방식

4. **정리된 실행 순서**
   - `id = 2` (파생 테이블 생성)
   - `id = 1` 중 `<derived2>` (파생 테이블 사용)
   - `id = 1` 중 `employees` (조인 실행)

## § 10.3.4 partitions 칼럼

8.0 부터 EXPLAIN 명령으로 파티션 관련 실행 계획까지 모두 확인할 수 있게 되었다.

employess_2 테이블은 hire_date 기준으로 5년 단위로 나누어진 파티션을 갖도록 생성

```sql
-- 파티셔닝 테이블 생성
CREATE TABLE employees_2 (
    emp_no      INT NOT NULL,
    birth_date  DATE NOT NULL,
    first_name  VARCHAR(14) NOT NULL,
    last_name   VARCHAR(16) NOT NULL,
    gender      ENUM('M','F') NOT NULL,
    hire_date   DATE NOT NULL,
    PRIMARY KEY (emp_no, hire_date)
)
PARTITION BY RANGE COLUMNS(hire_date) (
    PARTITION p1986_1990 VALUES LESS THAN ('1990-01-01'),
    PARTITION p1991_1995 VALUES LESS THAN ('1996-01-01'),
    PARTITION p1996_2000 VALUES LESS THAN ('2001-01-01'),
    PARTITION p2001_2005 VALUES LESS THAN ('2006-01-01')
);

-- 데이터 이관
INSERT INTO employees_2
SELECT * FROM employees;
```

```sql
EXPLAIN
SELECT *
FROM employees_2
WHERE hire_date BETWEEN '1999-11-15' AND '2000-01-15';
```

1999.11.15 ~ 2000.01.15 사이인 레코드 검색

| id  | select_type | table       | partitions             | type | rows  |
| --- | ----------- | ----------- | ---------------------- | ---- | ----- |
| 1   | SIMPLE      | employees_2 | p1996_2000, p2001_2005 | ALL  | 21743 |

테이블 생성 구문에서 파티션 목록을 보면, 이 쿼리에서 조회하려는 데이터는 p1996_2000과 p2001_2005 파티션에 저장돼 있음을 알 수 있는데, 실제로 옵티마이저는 쿼리의 hire_date 칼럼 조건을 보고, 나머지 파티션에 대해선느 어떻게 접근할지, 데이터 분포가 어떠한지 등의 분석을 실행하지 않는다.

이처럼 파티션이 여러 개인 테이블에서 불필요한 파티션을 빼고 쿼리를 위해 접근해야 할 것으로 판단되는 테이블만 골라내는 과정을 **파티션 프루닝(Partition prunning)** 이라고 한다.

실행계획에서 type이 ALL로 표시되는데, 이것은 employees_2 테이블을 풀 테이블 스캔 한다는 의미가 아니다.

MySQL을 포함한 대부분의 RDBMS에서 지원하는 파티션은 물리적으로 개별 테이블처럼 별도의 공간을 갖기 때문에 p1996_2000 파티션과 p2001_2005 파티션만 풀 스캔 한다는 뜻이다.

## § 10.3.5 type 칼럼

쿼리의 실행 계획에서 type 이후 칼럼은 MySQL 서버가 각 테이블의 레코드를 어떤 방식으로 읽었는지를 나타낸다.(인덱스 사용/풀 테이블 스캔 등)

type 칼럼에 표시될 수 있는 값

- system
- const
- eq_ref
- ref
- fulltext
- ref_or_null
- unique_subquery
- index_subquery
- range
- index_merge
- index
- ALL

ALL을 제외한 나머지는 모두 인덱스를 사용하는 접근 방법이다.
ALL은 풀 테이블 스캔

하나의 단위 SELECT 쿼리는 위의 접근 방법 중에서 단 하나만 사용할수 있다.  
또한 index_merge를 제외한 나머지 접근 방법은 하나의 인덱스만 사용한다.

### § 10.3.5.1 system

레코드가 1건만 존재하는 테이블 또는 한 건도 존재하지 않는 테이블을 참조하는 형태의 접근 방법을 나타냄.

### § 10.3.5.2 const

테이블의 레코드 건수와 관계없이 쿼리가 프라이머리 키나 유니크 키 칼럼을 이용하는 WHERE 조건절을 가지고 있으며, 반드시 1건을 반환하는 쿼리의 처리 방식

다른 DBMS에서는 이를 유니크 인덱스 스캔(UNIQUE INDEX SCAN) 이라고 표현한다.

```sql
EXPLAIN
SELECT * FROM employees WHERE emp_no = 10001;
```

| id  | select_type | table     | type  | key     | key_len |
| --- | ----------- | --------- | ----- | ------- | ------- |
| 1   | SIMPLE      | employees | const | PRIMARY | 4       |

<br>

---

<br>

프라이머리 키의 일부만 조건으로 사용할 경우 const가 아닌 ref로 표시된다.

```sql
EXPLAIN
SELECT * FROM dept_emp WHERE dept_no = 'd005';
```

| id  | select_type | table    | type | key     | rows   |
| --- | ----------- | -------- | ---- | ------- | ------ |
| 1   | SIMPLE      | dept_emp | ref  | PRIMARY | 165571 |

<br>

---

<br>

프라이머리 키나 유니크 인덱스의 모든 칼럼을 동등 조건으로 WHERE 절에 명시하면 const를 사용한다.

```sql
EXPLAIN
SELECT *
FROM dept_emp
WHERE dept_no = 'd005' AND emp_no = 10001;
```

| id  | select_type | table    | type  | key     | key_len | rows |
| --- | ----------- | -------- | ----- | ------- | ------- | ---- |
| 1   | SIMPLE      | dept_emp | const | PRIMARY | 20      | 1    |

#### 참고

##### 🔎 실행 계획에서 `const`가 나타나는 경우

| 키포인트           | 설명                                                                                                           |
| ------------------ | -------------------------------------------------------------------------------------------------------------- |
| **`type = const`** | 옵티마이저가 **단 한 번만 읽고 상수(스칼라 값)로 치환**할 수 있다고 판단한 테이블(또는 서브쿼리)에 부여되는 값 |
| **상수 치환 과정** | ① 서브쿼리 실행 → ② 결과를 상수값으로 고정 → ③ 바깥 쿼리에 해당 상수를 전달                                    |
| **장점**           | 이후 단계에서 테이블 접근이 아예 사라지므로 I/O 비용·조인 비용이 크게 줄어듦                                   |

---

###### 예시 1 — 원본 쿼리

```sql
EXPLAIN
SELECT COUNT(*)
FROM employees e1
WHERE first_name = (
    SELECT first_name      -- ◀ emp_no = 100001인 직원의 first_name
    FROM employees e2
    WHERE emp_no = 100001
);
```

| id  | select_type | table | type  | key          | key_len | rows | 의미                     |
| --- | ----------- | ----- | ----- | ------------ | ------- | ---- | ------------------------ |
| 1   | PRIMARY     | e1    | ref   | ix_firstname | 58      | 248  | `first_name` 인덱스 범위 |
| 2   | SUBQUERY    | e2    | const | PRIMARY      | 4       | 1    | **상수 서브쿼리**        |

- **2번 라인**은 `PRIMARY KEY`로 정확히 한 행만 조회 → `const`
- 옵티마이저가 먼저 실행해 값을 ‘상수’로 만든 뒤 바깥 쿼리에 전달

---

###### 예시 2 — 실제 실행 시점에 변환되는 형태

```sql
SELECT COUNT(*)
FROM employees e1
WHERE first_name = 'Jasminko';  -- 100001번 사원의 first_name
```

> 서브쿼리 결과가 **'Jasminko'** 로 고정됐으므로 이후 단계에서는  
> `employees` 테이블 한 곳만 읽으며, `type = const` 접근 비용이 발생하지 않음.

---

##### ✅ 요약

- `type` 컬럼이 `const`이면 **“이 테이블(또는 서브쿼리)은 한 번만 읽고 상수로 대체된다”** 는 뜻
- 주로 **프라이머리 키·유니크 키**로 **단일 행**을 찾는 서브쿼리에서 발생
- 쿼리 성능 향상 ↔ 불필요한 조인/스캔 제거 → **실행 계획 최적화의 중요한 신호**다.

### § 10.3.5.3 eq_ref

여러 테이블이 조인되는 쿼리의 실행 계획에서만 표시됨.  
조인에서 처음 읽은 테이블의 칼럼 값을 그 다음 읽어야 할 테이블의 프라이머리 키나 유니크 키 칼럼의 검색 조건에 사용할 때에 eq_ref가 표시된다..

두 번째 이후에 읽는 테이블의 type 칼럼에 eq_ref가 표시된다.  
또한 두 번째 이후에 읽히는 테이블을유니크 키로 검색할 때 그 유니크 인덱스는 NOT NULL 이어야 하며, 다중 칼럼으로 만들어진 프라이머리 키나 유니크 인덱스라면 인덱스의 모든 칼럼이 비교 조건에 사용 돼야만 eq_ref 접근 방법이 사용될 수 있다.

즉, 조인에서 두 번째 이후에 읽는 테이블에서 반드시 1건만 존재한다는 보장이 있어야 사용할 수 있다.

```sql
EXPLAIN
SELECT *
FROM dept_emp de, employees e
WHERE e.emp_no = de.emp_no AND de.dept_no = 'd005';
```

| id  | select_type | table | type   | key     | key_len | rows   |
| --- | ----------- | ----- | ------ | ------- | ------- | ------ |
| 1   | SIMPLE      | de    | ref    | PRIMARY | 16      | 165571 |
| 1   | SIMPLE      | e     | eq_ref | PRIMARY | 4       | 1      |

1. dept_emp 테이블 먼저 조회  
   복합 기본키 (emp_no, dept_no)의 일부만 사용 -> ref
2. employees 테이블  
   emp_no는 employees의 기본 키 -> eq_ref 타입  
   조인 대상이 명확하므로 rows = 1

### § 10.3.5.4 ref

eq_ref와는 달리 조인의 순서와 관계 없이 사용되며, 프라이머리 키나 유니크 키 등의 제약 조건도 없다.

인덱스의 종류와 관계 없이 동등 조건으로 검색할 때는 ref 접근 방법이 사용된다.

ref 타입은 반환되는 레코드가 반드시 1건이라는 보장이 없으므로 const, eq_ref 보다는 빠르지 않다. 하지만 동등 조건으로만 비교되므로 매우 빠른 레코드 조회 방법 중 하나이다.

```sql
EXPLAIN
SELECT * FROM dept_emp WHERE dept_no = 'd005';
```

| id  | select_type | table    | type | key     | key_len | ref   |
| --- | ----------- | -------- | ---- | ------- | ------- | ----- |
| 1   | SIMPLE      | dept_emp | ref  | PRIMARY | 16      | const |

---

#### 🧠 실행 계획 해석

- `dept_emp` 테이블의 기본 키는 `(emp_no, dept_no)`로 구성된 **복합 프라이머리 키**이다.
- 하지만 `WHERE dept_no = 'd005'` 조건은 **복합 키의 일부**만 사용하고 있음.
- 이 경우:
  - `const` 접근 방식은 사용되지 않음 (인덱스 전체가 주어져야 const 사용 가능)
  - 대신 `ref` 방식으로 처리됨 → 인덱스를 통해 **일치하는 여러 행을 탐색**

##### 📌 주의

- `ref = const`는 테이블에 전달된 **입력값이 상수값**이라는 뜻이며,
- 접근 방식 자체가 `const`는 아니라는 점에 유의해야 함.

### § 10.3.5.5 fulltext

MySQL 서버의 전문 검색(Full-text Search) 인덱스를 사용해 레코드를 읽는 접근 방법을 의미한다.

MySQL 서버에서 전문 검색 조건은 우선 순위가 상당히 높다.  
쿼리에서 전문 인덱스를 사용하는 조건과 그 이외의 일반 인덱스를 사용하는 조건을 함께 사용하면,  
일반 인덱스의 접근 방법이 const, eq_ref, ref가 아니라면 MySQL은 전문 인덱스를 사용하는 조건을 선택해서 처리한다.
<br>

---

<br>  
전문 검색용 인덱스가 있는 테이블 생성

```sql
CREATE TABLE employee_name (
    emp_no     INT NOT NULL,
    first_name VARCHAR(14) NOT NULL,
    last_name  VARCHAR(16) NOT NULL,
    PRIMARY KEY (emp_no),
    FULLTEXT KEY fx_name (first_name, last_name)
        WITH PARSER ngram
) ENGINE = InnoDB;
```

```sql
EXPLAIN
SELECT *
FROM employee_name
WHERE emp_no = 10001
  AND emp_no BETWEEN 10001 AND 10005
  AND MATCH(first_name, last_name)
      AGAINST('Facello' IN BOOLEAN MODE);
```

위 쿼리는 3개의 조건을 가지고 있다.

1. employee_name 테이블의 프라이머리 키를 1건만 조회하는 const 타입
2. range 타입
3. 전문 검색 조건

<br>

1. 실행 계획을 보면 =>

   | id  | select_type | table         | type  | key     | key_len |
   | --- | ----------- | ------------- | ----- | ------- | ------- |
   | 1   | SIMPLE      | employee_name | const | PRIMARY | 4       |

   최종적으로 MySQL은 const 타입의 조건을 선택했다.

2. const 조건을 제외하면?

   ```sql
   EXPLAIN
   SELECT *
   FROM employee_name
   WHERE emp_no BETWEEN 10001 AND 10005
   AND MATCH(first_name, last_name)
       AGAINST('Facello' IN BOOLEAN MODE);
   ```

   | id  | select_type | table         | type     | key     | Extra                             |
   | --- | ----------- | ------------- | -------- | ------- | --------------------------------- |
   | 1   | SIMPLE      | employee_name | fulltext | fx_name | Using where; Ft_hints: no_ranking |

   range 대신 전문 검색 조건을 선택한다.  
   하지만 range 접근 방법이 더 빨리 처리되는 경우도 많으므로 조건별로 성능을 확인해 보는 편이 좋다.

### § 10.3.5.6 ref_or_null

ref 접근 방법과 같은데, NULL 비교가 추가된 접근 방법이다.

```sql
EXPLAIN
SELECT *
FROM titles
WHERE to_date = '1985-03-01' OR to_date IS NULL;
```

| id  | select_type | table  | type        | key       | key_len | ref   | rows |
| --- | ----------- | ------ | ----------- | --------- | ------- | ----- | ---- |
| 1   | SIMPLE      | titles | ref_or_null | ix_todate | 4       | const | 2    |

### § 10.3.5.7 unique_subquery

WHERE 조건절에서 사용될 수 있는 IN(subquery) 형태의 쿼리를 위한 접근 방법이다.

서브쿼리에서 중복되지 않는 유니크한 값만 반환할 때 이 접근 방법을 사용한다.

```sql
EXPLAIN
SELECT *
FROM departments
WHERE dept_no IN (
  SELECT dept_no
  FROM dept_emp
  WHERE emp_no = 10001
);
```

| id  | select_type        | table       | type            | key         | key_len |
| --- | ------------------ | ----------- | --------------- | ----------- | ------- |
| 1   | PRIMARY            | departments | index           | ux_deptname | 162     |
| 2   | DEPENDENT SUBQUERY | dept_emp    | unique_subquery | PRIMARY     | 20      |

### § 10.3.5.8 index_subquery

IN 연산자의 특성 상 IN(subquery), IN(상수 나열) 형태의 조건은 괄호 안에 있는 값의 목록 중 중복된 값을 먼저 제거해야 한다.

unique_subquery 접근 방법은 In(subquery) 조건의 subquery가 중복된 값을 만들어내지 않는다는 보장이 있으므로 별도의 중복을 제거할 필요가 없었다.

하지만 IN(subquery)에서 subquery가 중복된 값을 반환할 경우, 이 중복된 값을 인덱스를 이용해서 제거할 수 있을 때 index_subquery 접근 방법이 사용된다.

### § 10.3.5.9 range

인덱스 레인지 스캔 형태의 접근 방법이다.

주로 <, >, IS NULL, BETWEEN, IN, LIKE 등의 연산자를 이용해 인덱스를 검색할 때 사용된다.

우선 순위가 낮은 편이지만, range 접근 방법도 상당히 빠르다.

```sql
EXPLAIN
SELECT *
FROM employees
WHERE emp_no BETWEEN 10002 AND 10004;
```

| id  | select_type | table     | type  | key     | key_len | rows |
| --- | ----------- | --------- | ----- | ------- | ------- | ---- |
| 1   | SIMPLE      | employees | range | PRIMARY | 4       | 3    |

### § 10.3.5.10 index_merge

2개 이상의 인덱스를 이용해 각각의 검색 결과를 만들어낸 후, 그 결과를 병합해서 처리하는 방식이다.

- 여러 인덱스를 읽어야 하므로 일반적으로 range 접근 방법보다 효율성이 떨어진다.
- 전문 검색 인덱스를 사용하는 쿼리에서는 index_merge가 적용되지 않는다.
- index_merge 접근 방법으로 처리된 결과는 항상 2개 이상의 집합이 되기 때문에 그 두 집합의 교집합이나 합집합, 또는 중복 제거와 같은 부가적인 작업이 더 필요하다.

```sql
EXPLAIN
SELECT *
FROM employees
WHERE emp_no BETWEEN 10001 AND 11000
   OR first_name = 'Smith';
```

| id  | type        | key                   | key_len | Extra                                           |
| --- | ----------- | --------------------- | ------- | ----------------------------------------------- |
| 1   | index_merge | PRIMARY, ix_firstname | 4, 58   | Using union(PRIMARY, ix_firstname); Using where |

두 개의 조건이 OR 연산자로 연결된 쿼리이다.  
그런데 두 개의 조건이 각각 다른 인덱스를 최적으로 사용할 수 있는 조건이다.

그래서 옵티마이저는 "emp_no BETWEEN 10001 AND 11000" 조건은 employees 테이블의 프라이머리 키,  
"first_name='Smith'" 조건은 ix_firstname 인덱스를 이용해 조회한 후  
두 결과를 병합하는 형태로 처리하는 실행했다.

### § 10.3.5.11 index

인덱스 풀 스캔을 의미한다.  
테이블 풀 스캔과 비교하는 레코드 건수는 같지만, 데이터 파일 보다는 인덱스의 크기가 작으므로 풀 테이블 스캔 보다는 빠르게 처리됨

- range나 const, ref 같은 접근 방법으로 인덱스를 사용하지 못하는 경우
- 인덱스에 포함된 칼럼만으로 처리할 수 있는 쿼리인 경우(데이터 파일을 읽지 않아도 되는 경우)
- 인덱스를 이용해 정렬이나 그루핑 작업이 가능한 경우

```sql
EXPLAIN
SELECT *
FROM departments
ORDER BY dept_name DESC
LIMIT 10;
```

| id  | select_type | table       | type  | key         | key_len | rows |
| --- | ----------- | ----------- | ----- | ----------- | ------- | ---- |
| 1   | SIMPLE      | departments | index | ux_deptname | 162     | 9    |

LIMIT 조건이 있으므로 매우 효율적이다.  
단순히 인덱스를 역순으로 읽어서 10개만 가져오면 되기 때문에 index 접근 방법이 매우 적합하다.

### § 10.3.5.12 ALL

가장 마지막에 선택되는가장 비효율적인 방법이다.

## § 10.3.6 possible_keys 칼럼

옵티마이저가 최적의 실행 계획을 만들기 위해 후보로 선정했던 접근 방법에서 사용되는 인덱스의 목록.

possible_key 칼럼에 인덱스 이름이 나열됐다고 해서 그 인덱스를 사용하는 것이 아니다.

## § 10.3.7 key 칼럼

최종 선택된 실행 계획에서 사용하는 인덱스

쿼리를 튜닝할 때 key 칼럼에 의도했던 인덱스가 표시되는지 확인하는 것이중요하다.

실행 계획의 type이 ALL 일 때와 같이 인덱스를 전혀 사용하지 못하면 NULL로 표시된다.

## § 10.3.8 key_len 칼럼

실제 사용하는 테이블은 다중 칼럼으로 만들어진 인덱스가 많다.

key_len 칼럼의 값은 쿼리를 처리하기 위해 다중 칼럼으로 구성된 인덱스의 각 레코드에서 몇 바이트까지 사용했는지를 알려준다.

```sql
EXPLAIN
SELECT *
FROM dept_emp
WHERE dept_no = 'd005';
```

| id  | select_type | table    | type | key     | key_len |
| --- | ----------- | -------- | ---- | ------- | ------- |
| 1   | SIMPLE      | dept_emp | ref  | PRIMARY | 16      |

(dept_no, emp_no)로 구성된 프라이머리 키 중에 dept_no만 비교에 사용했다는 뜻이다.  
MySQL 서버가 utf8mb4 문자 메모리 할당을 위해 4바이트,  
dept_no 타입이 CHAR(4) 여서 16바이트가 사용된 것

## § 10.3.9 ref 칼럼

접근 방법이 ref 라면 동등 비교 조건으로 어떤 값이 제공됐는지 표시된다.

상수값이 지정됐다면 const, 다른 테이블의 칼럼 값이면 그 테이블 명과 칼럼 명이 표시된다.  
값이 그대로 사용되지 않고 연산을 거쳐 참조되면 func 라고 표시된다.

```sql
EXPLAIN
SELECT *
FROM employees e, dept_emp de
WHERE e.emp_no = de.emp_no;
```

| id  | select_type | table | type   | key     | ref                 |
| --- | ----------- | ----- | ------ | ------- | ------------------- |
| 1   | SIMPLE      | de    | ALL    | NULL    | NULL                |
| 1   | SIMPLE      | e     | eq_ref | PRIMARY | employees.de.emp_no |

---

```sql
EXPLAIN
SELECT *
FROM employees e, dept_emp de
WHERE e.emp_no = (de.emp_no - 1);
```

| id  | select_type | table | type   | key     | ref  |
| --- | ----------- | ----- | ------ | ------- | ---- |
| 1   | SIMPLE      | de    | ALL    | NULL    | NULL |
| 1   | SIMPLE      | e     | eq_ref | PRIMARY | func |

## § 10.3.10 rows 칼럼

쿼리를 처리하기 위해 얼마나 많은 레코드를 읽고 체크해야 하는지를 나타냄.

반환하는 레코드의 예측치가 아니다  
그래서 실행 계획의 rows 칼럼에 출력되는 값과 실제 쿼리 결과 반환된 레코드 건수는 일치하지 않는 경우가 많다.

```sql
EXPLAIN
SELECT *
FROM dept_emp
WHERE from_date >= '1985-01-01';
```

| id  | select_type | table    | type | key  | key_len | rows   |
| --- | ----------- | -------- | ---- | ---- | ------- | ------ |
| 1   | SIMPLE      | dept_emp | ALL  | NULL | NULL    | 331143 |

> 🔸 옵티마이저는 `ix_fromdate` 인덱스를 사용할 수 있었음에도 불구하고,  
> 🔸 전체 레코드를 읽는 것이 비용적으로 더 낫다고 판단 → 인덱스 사용 안함

---

```sql
EXPLAIN
SELECT *
FROM dept_emp
WHERE from_date >= '2002-07-01';
```

| id  | select_type | table    | type  | key         | key_len | rows |
| --- | ----------- | -------- | ----- | ----------- | ------- | ---- |
| 1   | SIMPLE      | dept_emp | range | ix_fromdate | 3       | 292  |

> 🔸 `range` 스캔: 인덱스를 활용하여 from_date가 '2002-07-01' 이상인 행만 조회  
> 🔸 읽어야 할 예상 row 수는 약 292건으로 **상대적으로 적은 범위**  
> 🔸 이 경우는 **인덱스 스캔이 더 효율적**이라고 판단되어 사용됨

## § 10.3.11 filtered 칼럼

인덱스를 통해 읽은 레코드 중 실제로 조건을 만족하는 비율(%)을 나타냄

### 예시

```sql
SELECT *
FROM employees e
JOIN salaries s ON s.emp_no = e.emp_no
WHERE e.first_name = 'Matt'
  AND e.hire_date BETWEEN '1990-01-01' AND '1991-01-01'
  AND s.from_date BETWEEN '1990-01-01' AND '1991-01-01'
  AND s.salary BETWEEN 50000 AND 60000;
```

#### 실행 계획 (조인 순서: employees → salaries)

| table | rows | filtered |
| ----- | ---- | -------- |
| e     | 233  | 16.03    |
| s     | 10   | 0.48     |

- `employees`: 233건 중 약 **16.03%**, 즉 **37건**이 조건 만족
- `salaries`: 10건 중 **0.48%**만 최종 조건 만족

---

### 예시(조인 순서를 변경한 경우)

```sql
SELECT /*+ JOIN_ORDER(s, e) */
FROM employees e, salaries s ...
```

#### 실행 계획 (조인 순서: salaries → employees)

| table | rows | filtered |
| ----- | ---- | -------- |
| s     | 3314 | 11.11    |
| e     | 1    | 5.00     |

- `salaries`: 3314건 중 약 **368건 (3314 \* 0.1111)** 조건 만족
- `employees`: 각 salary에 대해 1건씩 조회, 그중 5%만 조건 만족

=> filtered 값이 낮은 조건일수록 **실제 처리량이 줄어들고**, **빠른 성능을 유도**할 수 있음.

## § 10.3.12 Extra 칼럼

쿼리의 실행 계획에서 성능과 관련된 중요한 내용이 Extra 칼럼에 자주 표시된다.

### § 10.3.12.1 const row not found

const 접근 방법으로 테이블을 읽었지만 실제로 해당 테이블에 레코드가 1건도 없는 경우

### § 10.3.12.2 Deleting all rows

스토리지 엔진의 핸들러 차원에서 테이블의 모든 레코드를 삭제하는 기능을 제공하는 스토리지 엔진을 사용하는 경우

조건절이 없는 DELETE 문장의 실행 계획에서 표시됨

한 번의 핸들러 함수 호출로 빠르게 처리되는 경우

### § 10.3.12.3 Distinct

```sql
EXPLAIN
SELECT DISTINCT d.dept_no
FROM departments d, dept_emp de
WHERE de.dept_no = d.dept_no;
```

| id  | select_type | table | type  | key         | Extra                        |
| --- | ----------- | ----- | ----- | ----------- | ---------------------------- |
| 1   | SIMPLE      | d     | index | ux_deptname | Using index; Using temporary |
| 1   | SIMPLE      | de    | ref   | PRIMARY     | Using index; Distinct        |

departments 테이블과 dept_emp 테이블에 모두 존재하는 dept_no만 유니크하게 가져오기 위한 쿼리

![](/Section.%2010.3/이주현/images/10.6.png)

쿼리의 DISTINCT를 처리하기 위해 조인하지 않아도 되는 항목은 모두 무시하고 꼭 필요한 것만 조인, dept_emp 테이블에서는 꼭 필요한 레코드만 읽었다.

### § 10.3.12.4 FirstMatch

세미 조인의 여러 최적화 중 FirstMatch 전략이 사용될 경우

Extra 칼럼에 "FirstMatch(table_name)" 이 출력된다.

```sql
EXPLAIN SELECT *
FROM employees e
WHERE e.first_name = 'Matt'
  AND e.emp_no IN (
    SELECT t.emp_no
    FROM titles t
    WHERE t.from_date BETWEEN '1995-01-01' AND '1995-01-30'
  );
```

### 실행 계획

| id  | table | type | key          | rows | Extra                                   |
| --- | ----- | ---- | ------------ | ---- | --------------------------------------- |
| 1   | e     | ref  | ix_firstname | 233  | NULL                                    |
| 1   | t     | ref  | PRIMARY      | 1    | Using where; Using index; FirstMatch(e) |

table_name은 기준 테이블을 말하는데, 위 실행 계획의 경우 employees 테이블을 기준으로 titles 테이블에서 첫 번째로 일치하는 한 건만 검색한다는 것을 의미한다.

### § 10.3.12.5 Full scan on NULL key

"col1 IN (SELECT col2 FROM...)" 같은 쿼리에서 자주 발생할 수 있는데, col1의 값이 NULL이 되면 조건은 "NULL IN (SELECT col2 FROM...)" 같이 바뀐다.

SQL 표준에서 NULL은 "알 수 없는 값"으로 정의하기 때문에  
정의된 연산 순서에 따라 비교가 다르게 작동한다.

- 서브쿼리에 결과가 1건이라도 있으면 최종 결과는 'NULL'
- 서브쿼리에 결과가 없으면 최종 결과는 'FALSE'

이 비교과정에서 col1이 NULL 이면 서브쿼리에 사용된 테이블에 대해서 풀 테이블 스캔을 해야만 결과를 알아낼 수 있다.

```sql
EXPLAIN
SELECT d.dept_no,
       NULL IN (SELECT id.dept_name FROM departments d2)
FROM departments d1;
```

| id  | select_type | table | type           | Extra                                           |
| --- | ----------- | ----- | -------------- | ----------------------------------------------- |
| 1   | PRIMARY     | d1    | index          | Using index                                     |
| 2   | SUBQUERY    | d2    | index_subquery | Using where; Using index; Full scan on NULL key |

Extra 칼럼에 "Full scan on NULL key"는 col1이 NULL이 되면, 차선책으로 서브쿼리 테이블에 대해서 풀 테이블 스캔을 사용할 것이라는 것을 나타낸다.

col1이 NOT NULL로 정의된 칼럼이라면 이 방법은 사용되지 않는다.

### § 10.3.12.6 Impossible HAVING

쿼리에 사용된 HAVING 절의 조건을 만족하는 레코드가 없을 때 실행 계획의 Extra 칼럼에는 "Impossible HAVING" 키워드가 표시된다.

```sql
EXPLAIN
SELECT e.emp_no, COUNT(*) AS cnt
FROM employees e
WHERE e.emp_no = 10001
GROUP BY e.emp_no
HAVING e.emp_no IS NULL;
```

| id  | select_type | table | type | key  | Extra             |
| --- | ----------- | ----- | ---- | ---- | ----------------- |
| 1   | SIMPLE      | NULL  | NULL | NULL | Impossible HAVING |

### § 10.3.12.7 Impossible WHERE

WHERE 조건이 항상 FALSE인 경우

### § 10.3.12.8 LooseScan

세미 조인의 최적화 중 LooseScan 최적화 전략이 사용된 경우

```sql
EXPLAIN
SELECT *
FROM departments d
WHERE d.dept_no IN (
  SELECT de.dept_no
  FROM dept_emp de
);
```

| id  | table | type   | key     | rows   | Extra                  |
| --- | ----- | ------ | ------- | ------ | ---------------------- |
| 1   | de    | index  | PRIMARY | 331143 | Using index; LooseScan |
| 1   | d     | eq_ref | PRIMARY | 1      | NULL                   |

### § 10.3.12.9 No matching min/max row

WHERE 조건절을 만족하는 레코드가 한 건도 없는 경우: "Impossible WHERE..."

MIN()이나 MAX() 같은 집합 함수가 있는 쿼리의 조건절에 일치하는 레코드가 없는 경우: "No matching min/max row"

그리고 MIN() MAX()의 결과로 NULL 반환

### § 10.3.12.10 no matching row in const table

조인에 사용된 테이블에서 const 방법으로 접근할 때 일치하는 레코드가 없는 경우

```sql
EXPLAIN
SELECT *
FROM dept_emp de,
     (SELECT emp_no FROM employees WHERE emp_no = 0) tb1
WHERE tb1.emp_no = de.emp_no
  AND de.dept_no = 'd005';
```

| id  | select_type | table | type | key  | Extra                          |
| --- | ----------- | ----- | ---- | ---- | ------------------------------ |
| 1   | SIMPLE      | NULL  | NULL | NULL | no matching row in const table |

### § 10.3.12.11 No matching rows after partition pruning

파티셔님된 테이블에 대한 UPDATE DELETE 명령의 실행 계획에서 표시될 수 있다.

해당 파티션에 UPDATE 하거나 DELETE할 대상 레코드가 없는 경우우

### § 10.3.12.12 No tables used

FROM 절이 없는 쿼리 문장이나 없는 테이블을 사용하는 쿼리의 실행 계획에서 표시됨

### § 10.3.12.13 Not exists

A 테이블에는 존재하지만 B 테이블에는 없는 값을 조회해야 하는 경우 NOT IN(subquery) 나 NOT EXIST 를 주로 이용하는데, 이러한 형태의 조인을 안티-조인(Anti-JOIN) 이라고 한다.

레코드 건수가 많을 때는 아우터 조인을 이용하면 빠른 성능을 낼 수 있다.

```sql
EXPLAIN
SELECT *
FROM dept_emp de
LEFT JOIN departments d ON de.dept_no = d.dept_no
WHERE d.dept_no IS NULL;
```

| id  | select_type | table | type   | key     | Extra                   |
| --- | ----------- | ----- | ------ | ------- | ----------------------- |
| 1   | SIMPLE      | de    | ALL    | NULL    | NULL                    |
| 1   | SIMPLE      | d     | eq_ref | PRIMARY | Using where; Not exists |

"Not exists" 메시지는 옵티마이저가 dept_emp 테이블의 레코드를 이용해 departments 테이블을 조인할 때 departments 테이블의 레코드가 존재하는지 아닌지만 판단한다는 것을 의미한다.

### § 10.3.12.14 Plan isn't ready yet

8.0 버전에서는 다른 커넥션에서 실행 중인 쿼리의 실행 계획을 살펴볼 수 있다.

```sql
EXPLAIN FOR CONNECTION 8;
```

| id  | select_type | table     | type | rows   | Extra       |
| --- | ----------- | --------- | ---- | ------ | ----------- |
| 1   | SIMPLE      | employees | ALL  | 300473 | Using where |

이렇게 EXPLAIN FOR CONNECTION 명령을 실행했을 때 Extra 칼럼에 "Plan is not ready yet" 이라는 메시지가 표시되면 해당 커넥션에서 아직 쿼리의 실행 계획을 수립하지 못한 상태라는 것이다.

### § 10.3.12.15 Range checked for each record(index map: N)

조인 조건에 상수가 없고 양쪽 모두 변수일 때 발생

ex. `e2.emp_no >= e1.emp_no`

MySQL 옵티마이저는 어느 테이블을 먼저 읽을지, 어떤 인덱스를 사용할지를 결정할 수 없다. 이유는, 한 테이블의 레코드를 하나씩 읽을 때마다 상대편 테이블의 조건 기준 값이 계속 변하기 때문이다.

emp_no가 1번부터 1억번까지 있다고 하자.
e1.emp_no=1인 경우에는 e2의 테이블 1억 건 전부 읽어야하고, e1.emp_no=1억 이라면 e2에서는 한 건의 레코드만 읽으면 된다.

이처럼 e1 테이블의 emp_no가 작을 때는 e2 테이블을 풀 테이블 스캔으로, emp_no가 클 때는 e2 테이블을 인덱스 레인지 스캔으로 접근하는 것이 효율적이다.

레코드마다 인덱스 레인지 스캔을 체크한다 -> "Range checked for each record"

![](/Section.%2010.3/이주현/images/10.7.png)

```sql
EXPLAIN
SELECT *
FROM employees e1, employees e2
WHERE e2.emp_no >= e1.emp_no;
```

| id  | select_type | table | type | Extra                                          |
| --- | ----------- | ----- | ---- | ---------------------------------------------- |
| 1   | SIMPLE      | e1    | ALL  | NULL                                           |
| 1   | SIMPLE      | e2    | ALL  | Range checked for each record (index map: 0x1) |

index map 1은 후보 인덱스 중 첫 번째 인덱스를 나타낸다.

e2 테이블의 첫 번째 인덱스를 사용할지, 아니면 풀 테이블 스캔을 할지 레코드마다 결정하겠다는 뜻이다.

### § 10.3.12.16 Recursive

8.0 부터 CTE(Common Table Expression)을 이용해 재귀 쿼리를 작성할 수 있게 되었다.

```sql
WITH RECURSIVE cte (n) AS (
  SELECT 1
  UNION ALL
  SELECT n + 1 FROM cte WHERE n < 5
)
SELECT * FROM cte;
```

| id  | select_type | table      | type | key  | Extra                  |
| --- | ----------- | ---------- | ---- | ---- | ---------------------- |
| 1   | PRIMARY     | <derived2> | ALL  | NULL | NULL                   |
| 2   | DERIVED     | NULL       | ALL  | NULL | No tables used         |
| 3   | UNION       | cte        | ALL  | NULL | Recursive; Using where |

1. `"n"`이라는 칼럼 하나를 가진 `cte`라는 이름의 **내부 임시 테이블**을 생성
2. `"n"` 칼럼의 값이 1부터 5까지 1씩 증가하게 해서 **레코드 5건**을 만들어 `cte` 임시 테이블에 저장

WITH 절 다음의 `SELECT` 문에서는 WITH 절에 의해 생성된 내부 임시 테이블을 `FROM cte`로 참조하며, `WHERE` 조건이 없으므로 **풀 스캔**을 수행하여 결과를 반환한다.

이처럼 **재귀 CTE(Common Table Expression)** 를 사용하는 경우 `EXPLAIN`의 `Extra` 칼럼에 `"Recursive"` 구문이 표시된다.

### § 10.3.12.17 Rematerialize

8.0 부터 래터럴 조인이 추가되었는데, 래터럴로 조인되는 테이블은 선행테이블의 레코드 별로 서브쿼리를 실행해서 그 결과를 임시 테이블에 저장한다.

이 과정을 'Rematerializing' 이라고 한다.

```sql
EXPLAIN
SELECT * FROM employees e
LEFT JOIN LATERAL (
  SELECT *
  FROM salaries s
  WHERE s.emp_no = e.emp_no
  ORDER BY s.from_date DESC
  LIMIT 2
) s2 ON s2.emp_no = e.emp_no
WHERE e.first_name = 'Matt';
```

| id  | select_type       | table        | type | key           | Extra                        |
| --- | ----------------- | ------------ | ---- | ------------- | ---------------------------- |
| 1   | PRIMARY           | e            | ref  | ix_firstname  | Rematerialize (`<derived2>`) |
| 1   | PRIMARY           | `<derived2>` | ref  | `{auto_key0}` | NULL                         |
| 2   | DEPENDENT DERIVED | s            | ref  | PRIMARY       | Using filesort               |

실행 계획에서는 employees 테이블의 레코드마다 salaries 테이블에서 emp_no가 일치하는 레코드 중에서 from_data 칼럼의 역순으로 2건만 가져와 'derived2' 임시 테이블로 저장했다.

그리고 employees 테이블과 'derived2' 테이블을 조인한다.

그런데 여기서 'derived2' 임시 테이블은 employees 테이블의 레코드마다 새로 내부 임시 테이블이 생성된다.

여기서 표시되는 `Rematerialize`는 `LATERAL` 서브쿼리 결과를 **반복해서 생성하는 비용이 크다**는 것을 의미

### § 10.3.12.18 Select tables optimized away

MIN() 또는 MAX() 만 SELECT 절에 사용되거나 GROUP BY로 MIN(), MAX()를 조회하는 쿼리가 인덱스를 오름차순 또는 내림차순으로 1건만 읽는 형태의 최적화가 적용되면,

Extra 칼럼에 'Select tables optimized away' 가 표시된다.

![](/Section.%2010.3/이주현/images/10.8.png)
![](/Section.%2010.3/이주현/images/10.9.png)

### § 10.3.12.19 Start temporary, End temporary

세미 조인 최적화 중에 Duplicate Weed-out 최적화 전략이 사용되면

'Start temporary'와 'End temporary' 문구를 표시하게된다.

```sql
EXPLAIN
SELECT * FROM employees e
WHERE e.emp_no IN (
  SELECT s.emp_no
  FROM salaries s
  WHERE s.salary > 150000
);
```

| id  | select_type | table | type   | key       | Extra                                     |
| --- | ----------- | ----- | ------ | --------- | ----------------------------------------- |
| 1   | SIMPLE      | s     | range  | ix_salary | Using where; Using index; Start temporary |
| 1   | SIMPLE      | e     | eq_ref | PRIMARY   | End temporary                             |

Duplicate Weed-out은 서브쿼리의 결과에 중복이 많을 경우 이를 제거하기 위해 임시 테이블을 사용하는 전략이다.

'Start temporary'는 조인이 내부 임시 테이블 사용을 시작함을 의미

'End temporary'는 그 임시 테이블 사용이 끝났음을 의미

### § 10.3.12.20 unique row not found

두 개의 테이블이 각각 유니크(프라이머리 키 포함) 칼럼으로 아우터 조인을 수행하는 쿼리에서 아우터 테이블에 일치하는 레코드가 존재하지 않을 때

```sql
-- 테스트 케이스를 위한 테스트용 테이블 생성
CREATE TABLE tb_test1 (fdpk INT, PRIMARY KEY(fdpk));
CREATE TABLE tb_test2 (fdpk INT, PRIMARY KEY(fdpk));

-- 생성된 테이블에 레코드
INSERT INTO tb_test1 VALUES (1),(2);
INSERT INTO tb_test2 VALUES (1);

EXPLAIN
SELECT t1.fdpk
FROM tb_test1 t1
    LEFT JOIN tb_test2 t2 ON t2.fdpk=t1.fdpk
WHERE t1.fdpk=2;
```

| id  | select_type | table | type  | key     | rows | Extra                |
| --- | ----------- | ----- | ----- | ------- | ---- | -------------------- |
| 1   | SIMPLE      | t1    | const | PRIMARY | 1    | Using index          |
| 1   | SIMPLE      | t2    | const | PRIMARY | 0    | unique row not found |

### § 10.3.12.21 Using filesort

ORDER BY를 처리하기 위해 인덱스를 이용할 수 있지만, 적절한 인덱스 사용이 어려울 경우 MySQL 서버가 조회된 레코드를 다시 한 번 정렬해야 한다.

ORDER BY 처리가 인덱스를 이용하지 못할 때 Using filesort 가 표시된다.

정렬용 메모리 버퍼에 조회된 레코드를 복사해 정렬을 수행

```sql
EXPLAIN
SELECT * FROM employees
ORDER BY last_name DESC;
```

| id  | select_type | table     | type | key  | rows   | filtered | Extra          |
| --- | ----------- | --------- | ---- | ---- | ------ | -------- | -------------- |
| 1   | SIMPLE      | employees | ALL  | NULL | 300363 | 100.00   | Using filesort |

### § 10.3.12.22 Using index(커버링 인덱스)

데이터 파일을 전혀 읽지 않고 인덱스만 읽어서 쿼리를 모두 처리할 수 있을 때

![](/Section.%2010.3/이주현/images/10.10.png)

employees 테이블에 5만건이 저장되어 있다고 하자.
아래 쿼리의 실행 계획은 풀 테이블 스캔일 것이다.

```sql
-- birth_date 포함: 인덱스 사용 안됨 (ALL)
EXPLAIN
SELECT first_name, birth_date
FROM employees
WHERE first_name BETWEEN 'Babette' AND 'Gad';
```

| id  | select_type | table     | type | key  | rows   | Extra       |
| --- | ----------- | --------- | ---- | ---- | ------ | ----------- |
| 1   | SIMPLE      | employees | ALL  | NULL | 300473 | Using where |

```sql
-- first_name만 조회: 인덱스 사용됨 (range)
EXPLAIN
SELECT first_name
FROM employees
WHERE first_name BETWEEN 'Babette' AND 'Gad';
```

| id  | select_type | table     | type  | key          | key_len | Extra                    |
| --- | ----------- | --------- | ----- | ------------ | ------- | ------------------------ |
| 1   | SIMPLE      | employees | range | ix_firstname | 58      | Using where; Using index |

인덱스가 걸려 있는 컬럼만 조회하도록 쿼리를 수정했더니 풀 테이블 스캔 대신 인덱스 레인지 스캔을 사용하는 것을 볼 수 있다.

두 번째 예제처럼 인덱스만으로 쿼리를 수행할 수 있을 때 'Using index' 라는 메시지가 출력된다.

레코드 건수에 따라 차이가 있을 수 있지만 쿼리를 커버링 인덱스로 처리할 수 있을 때와 아닐 때 성능 차이는 수십 배에서 수백 배까지 날 수 있다.

하지만 인덱스에 많은 칼럼을 추가하면 메모리 낭비가 심해지고 레코드를 저장하거나 변경하는 작업이 매우 느려져 위험한 상황이 될 수 있다.

### § 10.3.12.23 Using index condition

MySQL 옵티마이저가 인덱스 컨디션 푸시다운(Index condition pushdown) 최적화를 사용하면 'Using index condition' 메시지가 출력된다.

```sql
SELECT *
FROM employees
WHERE last_name = 'Acton'
  AND first_name LIKE '%sal';
```

| id  | select_type | table     | type | key                   | key_len | Extra                 |
| --- | ----------- | --------- | ---- | --------------------- | ------- | --------------------- |
| 1   | SIMPLE      | employees | ref  | ix_lastname_firstname | 66      | Using index condition |

### § 10.3.12.24 Using index for group-by

GROUP BY 처리가 인덱스를 이용하면 레코드의 정렬이 필요하지 않고 인덱스의 필요한 부분만 읽으면 되기 때문에 상당히 효율적이고 빠르게 처리된다. (루스 인덱스 스캔)

#### § 10.3.12.24.1 타이트 인덱스 스캔(인덱스 스캔)을 통한 GROUP BY 처리

GROUP BY 처리에 인덱스를 이용할 수 있더라도 AVG(), SUM(), COUNT() 처럼 조회하려는 값이 모든 인덱스를다 읽어야 할 때는 필요한 레코드만 듬성듬성 읽을 수 없다.

이러한 쿼리의 실행계획에는 'Using index for group-by' 메시지가 출력되지 않는다.

```sql
EXPLAIN
SELECT first_name, COUNT(*) AS counter
FROM employees
GROUP BY first_name;
```

| id  | select_type | table     | type  | key          | key_len | Extra       |
| --- | ----------- | --------- | ----- | ------------ | ------- | ----------- |
| 1   | SIMPLE      | employees | index | ix_firstname | 58      | Using index |

#### § 10.3.12.24.2 루스 인덱스 스캔을 통한 GROUP BY 처리

단일 칼럼으로 구성된 인덱스에서는 그루핑 칼럼 말고는 아무것도 조회하지 않는 쿼리에서 루스 인덱스 스캔을 사용할 수 있다.

다중 칼럼으로 만들어진 인덱스에서는 GROUP BY 절이 인덱스를 사용하고, MIN()이나 MAX()같이 조회하는 값이 인덱스의 첫 번째 또는 마지막 레코드만 읽는 쿼리는 루스 인덱스 스캔이 사용될 수 있다.

```sql
EXPLAIN
SELECT emp_no,
       MIN(from_date) AS first_changed_date,
       MAX(from_date) AS last_changed_date
FROM salaries
GROUP BY emp_no;
```

| id  | select_type | table    | type  | key     | key_len | Extra                    |
| --- | ----------- | -------- | ----- | ------- | ------- | ------------------------ |
| 1   | SIMPLE      | salaries | range | PRIMARY | 4       | Using index for group-by |

#### ⚠️ WHERE 조건절 유무에 따른 GROUP BY 실행 계획 차이

##### ✅ WHERE 조건절이 **없는 경우**

- `WHERE` 절의 조건이 전혀 없는 쿼리는  
  `GROUP BY` 절의 컬럼과 `SELECT`로 가져오는 컬럼이 **"루스 인덱스 스캔(Lose Index Scan)"**을  
  사용할 수 있는 조건만 갖추면 된다.
- 그렇지 못한 쿼리는 **타이트 인덱스 스캔**(tight index scan)이나  
  별도의 정렬 과정을 통해 처리된다.

---

##### ✅ WHERE 조건절이 있지만 **검색을 위해 인덱스를 사용하지 못하는 경우**

- `GROUP BY` 절은 인덱스를 사용할 수 있지만,  
  `WHERE` 조건절이 인덱스를 사용하지 못할 때는 먼저 `GROUP BY`를 위해 **모든 레코드를 읽어야 한다.**
- 이 경우는 루스 인덱스 스캔을 이용할 수 없고,  
  **타이트 인덱스 스캔** 과정을 통해 `GROUP BY`가 처리된다.

---

##### ✅ WHERE 조건절이 있고, **검색을 위해 인덱스를 사용하는 경우**

- 하나의 인덱스가 `WHERE` 조건과 `GROUP BY` 처리를 **동시에 만족**할 경우,  
  **index_merge 없이도** 효율적인 실행 계획이 수립된다.
- 즉, `WHERE` 조건으로 걸러낼 수 있는 데이터가 많고,  
  `GROUP BY` 처리 컬럼이 동일한 인덱스에 포함되어 있다면  
  옵티마이저는 **정렬 없이 인덱스 레벨에서 GROUP BY 수행**이 가능하다.
- 이 경우에는 `Using index for group-by` 문구가 `Extra` 칼럼에 표시된다.

### § 10.3.12.25 Using index for skip scan

옵티마이저가 인덱스 스킵 스캔 최적화를 사용하는 경우

```sql
-- 인덱스 생성
ALTER TABLE employees
  ADD INDEX ix_gender_birthdate (gender, birth_date);

-- 실행 계획 확인
EXPLAIN
SELECT gender, birth_date
FROM employees
WHERE birth_date >= '1965-02-01';
```

| id  | table     | type  | key                 | Extra                                  |
| --- | --------- | ----- | ------------------- | -------------------------------------- |
| 1   | employees | range | ix_gender_birthdate | Using where; Using index for skip scan |

인덱스가 (gender, birth_date) 복합 인덱스인데, 조건은 birth_date에만 걸려있음

일반적으로 선두 칼럼인 gender가 없으면 인덱스를 사용할 수 없지만,  
8.0부터는 인덱스 스킵 스캔을 사용해 gender 값들을 내부적으로 루프 돌며 birth_date 조건을 평가함

### § 10.3.12.26 Using join buffer(Block Nested Loop), Using join buffer(Batched Key Access), Using join buffer(hash join)

일반적으로 빠른 쿼리 실행을 위해 조인되는 칼럼은 인덱스를 생성한다.

조인에서 뒤에 읽는 테이블의 칼럼에 인덱스가 필요한데, 옵티마이저는 조인되는 두 테이블에 있는 각 칼럼에서 인덱스를 조사하여 인덱스가 없는 테이블을 먼저 읽어 조인을 수행한다.

뒤에 읽는 테이블(드리븐 테이블)은 검색 위주로 사용되기 때문에 인덱스가 없으면 성능에 큰 영향을 미치기 때문이다.

드리븐 테이블에 검색을 위한 적절한 인덱스가 없다면 블록 네스티드 루프 조인이나 해시 조인을 사용한다.  
조인 버퍼가 사용되고, 조인 버퍼를 사용하는 실행 계획의 Extra 칼럼에는 'Using join buffer' 메시지가 표시된다.

```sql
EXPLAIN
SELECT *
FROM dept_emp de, employees e
WHERE de.from_date > '2005-01-01'
  AND e.emp_no < 10904;
```

| id  | select_type | table | type  | key         | Extra                                      |
| --- | ----------- | ----- | ----- | ----------- | ------------------------------------------ |
| 1   | SIMPLE      | de    | range | ix_fromdate | Using index condition                      |
| 1   | SIMPLE      | e     | range | PRIMARY     | Using where; Using join buffer (hash join) |

조인 조건이 없는 카테시안 조인은 항상 조인 버퍼를 사용한다.

### § 10.3.12.27 Using MRR

MRR(Multi Range Read): MySQL 엔진은 여러 개의 키 값을 한 번에 스토리지 엔진으로 전달하고, 스토리지 엔진은 넘겨 받은 키 값들을 정렬해서 최소한의 페이지 접근만으로 필요한 레코드를 읽을 수 있도록 하는 최적화

MRR이 도입되어 각 스토리지 엔진은 디스크 접근을 최소화 할 수 있게 되었다.

```sql
EXPLAIN
SELECT /*+ JOIN_ORDER(s, e) */ *
FROM employees e,
     salaries s
WHERE e.first_name='Matt'
  AND e.hire_date BETWEEN '1990-01-01' AND '1991-01-01'
  AND s.emp_no=e.emp_no
  AND s.from_date BETWEEN '1990-01-01' AND '1991-01-01'
  AND s.salary BETWEEN 50000 AND 60000;
```

| id  | table | type   | key       | key_len | rows | Extra                            |
| --- | ----- | ------ | --------- | ------- | ---- | -------------------------------- |
| 1   | s     | range  | ix_salary | 4       | 3314 | Using index condition; Using MRR |
| 1   | e     | eq_ref | PRIMARY   | 4       | 1    | Using where                      |

### § 10.3.12.28 Using sort_union(...), Using union(...), Using inntersect(...)

쿼리가 index_merge 접근 방법으로 실행되는 경우에는 2개 이상의 인덱스가 동시에 사용될 수 있다.

이때 Extra 칼럼에는 두 인덱스로부터 읽은 결과를 어떻게 병합했는지를 설명한다.

- **Using intersect(...)**

  - 각 인덱스를 사용할 수 있는 조건이 `AND`로 연결된 경우,
  - 각 처리 결과에서 **교집합**을 추출해내는 작업을 수행했다는 의미이다.

- **Using union(...)**

  - 각 인덱스를 사용할 수 있는 조건이 `OR`로 연결된 경우,
  - 각 처리 결과에서 **합집합**을 추출해내는 작업을 수행했다는 의미이다.

- **Using sort_union(...)**
  - `Using union`과 동일하게 합집합을 수행하지만,
  - `Using union`으로 처리할 수 없는 경우(`OR`로 연결된 상대적으로 **대량의 range 조건들**) 이 방식으로 처리된다.
  - `Using sort_union`과 `Using union`의 차이점:
    - `Using sort_union`은 **프라이머리 키만 먼저 읽어서 정렬하고 병합**한 후,
    - **필요한 레코드를 읽어서 반환**할 수 있다는 것이다.

### § 10.3.12.29 Using temporary

쿼리를 수행하는 동안 중간 결과를 담아 두기 위해 임시 테이블(Temporary table)을 사용한다.

### § 10.3.12.30 Using where

스토리지 엔진에서 처리한 결과를 MySQL 엔진 레이어에서 별도 가공을 해서 필터링을 한 경우에 'Using where' 메시지가 출력된다.

```sql
EXPLAIN
SELECT *
  FROM employees
WHERE emp_no BETWEEN 10001 AND 10100
  AND gender='F';
```

| id  | select_type | table     | type  | key     | rows | filtered | Extra       |
| --- | ----------- | --------- | ----- | ------- | ---- | -------- | ----------- |
| 1   | SIMPLE      | employees | range | PRIMARY | 100  | 50.00    | Using where |

---

- 이 쿼리에서 **작업 범위를 결정짓는 조건**은 `emp_no BETWEEN 10001 AND 10100`이다.  
  반면 `gender='F'` 조건은 **체크 조건**으로 작동한다.
- `emp_no` 조건만 만족하는 레코드는 **100건**이지만,  
  두 조건을 모두 만족하는 레코드는 **37건**뿐이다.
- 즉, **스토리지 엔진**은 100건을 읽어서 MySQL 엔진에 넘겼고,  
  MySQL 엔진은 **63건을 필터링해서 버렸다**는 의미이다.
- 여기서 `Using where`는 MySQL 레벨에서 조건 필터링을 수행했음을 의미한다.

### § 10.3.12.31 Zero limit
