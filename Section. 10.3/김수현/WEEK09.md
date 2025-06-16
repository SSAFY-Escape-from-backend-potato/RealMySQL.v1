# Real MySQL 10.3

# 10.3 실행 계획 분석

- Intro
  - Explain(옵션 X) 실행 시 쿼리 문장 특성에 따라 표 형태로 1줄 이사의 결과 표시
  - 표의 각 라인(레코드)은 쿼리 문장에서 사용된 테이블(서브쿼리로 임시 테이블을 생성한 경우 그 임시 테이블까지 포함)
  - 실행 순서는 위에서 아래로 순서대로 표시(위 쪽 출력 결과이면 쿼리 바깥 부분 혹은 먼저 접근한 테이블, 아래 쪽 출력 결과 쿼리 안쪽 부분 혹은 나중에 접근한 테이블)
- id 칼럼

  ```sql
  mysql>  EXPLAIN
  				SELECT e.emp_no, e.first_name, s.from_date, s.salary
  				FROM employees e, salaries s
  				WHERE e.emp_no = s.emp_no LIMIT 10;

  -|----|-------------|-------|-------|--------------|--------------------|--------|------------|
   | id | select_type | table | type  | key          | ref                | rows   | Extra      |
  -|----|-------------|-------|-------|--------------|--------------------|--------|------------|
   | 1  | SIMPLE      | e     | index | ix_firstname | NULL               | 300252 | Using index|
   | 1  | SIMPLE      | s     | ref   | PRIMARY      | employees.e.emp_no | 10     | NULL       |
  -|----|-------------|-------|-------|--------------|--------------------|--------|------------|

  -- 조인되는 테이블 갯수만큼 실행 레코드 출력되지만 같은 id 값이 부여된다.
  -- 여러 테이블 조인되는 경우 id 값을 증가하지 않고 같은 id 값 부여
  ```

  ```sql
  mysql>  EXPLAIN
  				SELECT
  				( (SELECT COUNT(*) FROM employees e) + (SELECT COUNT(*) FROM departments d) ) AS total_count

  -|----|-------------|-------|-------|--------------|--------------------|--------|----------------|
   | id | select_type | table | type  | key          | ref                | rows   | Extra          |
  -|----|-------------|-------|-------|--------------|--------------------|--------|----------------|
   | 1  | PRIMARY     | NULL  | NULL  | NULL         | NULL               | NULL   | No tables used |
   | 3  | SUBQUERY    | d     | index | ux_deptname  | NULL               | 9      | Using index    |
   | 2  | SUBQUERY    | e     | index | ix_hiredate  | NULL               | 300252 | Using index    |      |
  -|----|-------------|-------|-------|--------------|--------------------|--------|----------------|

  -- 다음 쿼리의 실행 계획에서는 쿼리 문장 3개의 SELECT로 구성되어 있어 실행 계획의 레코드가 각기 다른 id를 가진다.
  -- 실행 계획의 id 칼럼이 테이블 접근 순서 의미 X
  ```

  ```sql
  mysql>  EXPLAIN FORMAT=TREE
  				SELECT * FROM dept_emp de
  				WHERE de.emp_no = ( SELECT e.emp_no
  														FROM employees e
  														WHERE e.first_name='Georgi'
  															AND e.last_name='Facello' LIMIT 1);

  -|----|-------------|-------|-------|-------------------|--------|----------------|
   | id | select_type | table | type  | key               | rows   | Extra          |
  -|----|-------------|-------|-------|-------------------|--------|----------------|
   | 1  | PRIMARY     | de    | ref   | ix_empno_fromdate | 1      | Using where    |
   | 2  | SUBQUERY    | e     | ref   | ix_hiredate       | 253    | Using index    |
  -|----|-------------|-------|-------|-------------------|--------|----------------|

  -- 실행 계획의 id 칼럼이 테이블 접근 순서 의미 X
  -- 위 쿼리 실행 순서는 employees 테이블 머저 읽고 그 결과로 dept_emp 테이블 읽는 순서
  ```

- select_type 컬럼

  - SIMPLE : UNION이나 서브쿼리 사용하지 않는 단순 SELECT 경우 표시, 일반적으로 제일 바깥 SELECT의 select_type이 SIMPLE로 표시
  - PRIMARY : UNION이나 서브쿼리 가지는 SELECT 쿼리의 실행 계획에서 가장 바깥쪽 단위 쿼리는 select_type이 PRIMARY로 표시, SIMPLE과 마찬가지로 select_type이 PRIMARY 단위 SELECT 쿼리는 하나만 표시, 쿼리 제일 바깥쪽 있는 SELECT 단위 쿼리가 PRIMARY로 표시
  - UNION : UNION 결핮하는 단위 SELECT 가운데 첫 번째 제외한 두 번째 이후 단위 SELECT 쿼리의 select_type은 UNION으로 표시, UNION 첫 번째 단위 SELECT는 select_type이 UNION이 아니라 UNION 되는 쿼리 결과 모아서 저장하는 임시 테이블(DERIVED)이 select_type으로 표시
    ```sql
    mysql>  EXPLAIN
    				SELECT * FROM (
    					( SELECT emp_no FROM employees e1 LIMIT 10 ) UNION ALL
    					( SELECT emp_no FROM employees e2 LIMIT 10 ) UNION ALL
    					( SELECT emp_no FROM employees e3 LIMIT 10 ) ) tb;

    -|----|-------------|------------|-------|-------------|------|--------|----------------|
     | id | select_type | table      | type  | key         | ref  | rows   | Extra          |
    -|----|-------------|------------|-------|-------------|------|--------|----------------|
     | 1  | PRIMARY     | <derived2> | ALL   | NULL        | NULL | 1      | NULL           |
     | 2  | DERIVED     | e1         | index | ix_hiredate | NULL | 300252 | Using index    |
     | 3  | UNION       | e2         | index | ix_hiredate | NULL | 300252 | Using index    |
     | 4  | UNION       | e3         | index | ix_hiredate | NULL | 300252 | Using index    |
    -|----|-------------|------------|-------|-------------|------|--------|----------------|

    ```
  - DEPENDENT UNION : UNION select_type과 같이 UNION이나 UNION ALL로 집합을 결합하는 쿼리에서 표시 그리고 DEPENDENT는 UNION이나 UNION ALL로 결합된 단위 쿼리가 위부 쿼리에 의해 영향 받는 것을 의미
    ```sql
    mysql>  EXPLAIN
    				SELECT *
    				FROM employee e1 WHERE e1.emp_no IN (
    					SELECT e2.emp_no FROM employees e2 WHERE e2.first_name='MATT'
    					UNION
    					SELECT e3.emp_no FROM employees e3 WHERE e3.first_name='MATT'

    -|----|--------------------|-------------|------- |-------------|------|--------|-----------------|
     | id | select_type        | table       | type   | key         | ref  | rows   | Extra           |
    -|----|--------------------|-------------|--------|-------------|------|--------|-----------------|
     | 1  | PRIMARY            | e1          | ALL    | NULL        | NULL | 300252 | Using where     |
     | 2  | DEPENDENT SUBQUERY | e2          | eq_ref | ix_hiredate | func | 1      | Using where     |
     | 3  | DEPENDENT UNION    | e3          | eq_ref | ix_hiredate | func | 1      | Using where     |
     |NULL| UNION RESULT       | <union2, 3> | ALL    | ix_hiredate | NULL | NULL   | Using temporary |
    -|----|--------------------|-------------|--------|-------------|------|--------|-----------------|

    ```
  - UNION RESULT : UNION 결과 담아두는 테이블, MySQL 8.0뷰토 UNION ALL 경우 임시 테이블 사용 X but 여전히 UNION은 여전히 임시 테이블에 결과 버퍼링 → 실행 계획 상 임시 테이블을 가르키는 라인의 select_type이 UNION RESULT
    ```sql
    mysql>  EXPLAIN
    				SELECT emp_no FROM salaries WHERE salary > 10000
    				UNION DISTINCT
    				SELECT emp_no FROM dept_emp WHERE from_date > '2001-01-01';

    -|----|--------------------|-------------|------- |-------------|--------|--------------------------|
     | id | select_type        | table       | type   | key         | rows   | Extra                    |
    -|----|--------------------|-------------|--------|-------------|--------|--------------------------|
     | 1  | PRIMARY            | salaries    | range  | ix_salary   | 191348 | Using where; Using index |
     | 2  | UNION              | dept_emp    | range  | ix_fromdate | 5325   | Using where; Using index |
     |NULL| UNION RESULT       | <union1, 2> | ALL    | NULL        | NULL   | Using temporary          |
    -|----|--------------------|-------------|--------|-------------|--------|--------------------------|

    -- UNION RESULT 라인의 table 칼럼은 <union1,2) 표시되는데 id가 1과 2인 단위 쿼리의 조회 결과 UNION한 것을 의미

    mysql>  EXPLAIN
    				SELECT emp_no FROM salaries WHERE salary > 10000
    				UNION ALL
    				SELECT emp_no FROM dept_emp WHERE from_date > '2001-01-01';

    -|----|--------------------|-------------|------- |-------------|--------|--------------------------|
     | id | select_type        | table       | type   | key         | rows   | Extra                    |
    -|----|--------------------|-------------|--------|-------------|--------|--------------------------|
     | 1  | PRIMARY            | salaries    | range  | ix_salary   | 191348 | Using where; Using index |
     | 2  | UNION              | dept_emp    | range  | ix_fromdate | 5325   | Using where; Using index |
    -|----|--------------------|-------------|--------|-------------|--------|--------------------------|

    -- UNION ALL로 변경 시 UNION RESULT 라인 없어짐 -> 임시 테이블을 버퍼링하지 않기 때문
    ```
  - SUBQUERY : FROM절 이외에서 사용되는 서브쿼리만 의미
    ```sql
    mysql>  EXPLAIN
    				SELECT e.first_name,
    							(SELECT COUNT(*)
    							 FROM dept_emp de, dept_manager dm
    							 WHERE dm.dept_no = dm.dept_no) AS cnt
    				FROM employees e WHERE e.emp_no=10001;

    -|----|--------------------|-------------|------- |-------------|--------|--------------------------|
     | id | select_type        | table       | type   | key         | rows   | Extra                    |
    -|----|--------------------|-------------|--------|-------------|--------|--------------------------|
     | 1  | PRIMARY            | e           | const  | PRIMARY     | 1      | NULL                     |
     | 2  | SUBQUERY           | dm          | index  | PRIMARY     | 24     | Using index              |
     | 2  | SUBQUERY           | de          | ref    | PRIMARY     | 41392  | Using index              |
    -|----|--------------------|-------------|--------|-------------|--------|--------------------------|

    -- FROM 절에서 사용된 서브쿼리는 select_type이 DERIVED로 표시되고 그 밖의 위치에서 사용된 서브쿼리는 모두 SUBQUERY로 표시
    ```
  - DEPENDENT SUBQUERY : 서브쿼리가 바깥쪽(Outer) SELECT 쿼리에서 정의된 칼럼 사용하는 경우, select_type이 DEPENDENT SUBQUERY로 표시
    ```sql
    mysql>  EXPLAIN
    				SELECT e.first_name,
    							(SELECT COUNT(*)
    							 FROM dept_emp de, dept_manager dm
    							 WHERE dm.dept_no = dm.dept_no AND de.emp_no=e.emp_no) AS cnt
    				FROM employees e WHERE e.first_name='MATT';

    -|----|--------------------|-------------|------- |-------------------|--------|--------------------------|
     | id | select_type        | table       | type   | key               | rows   | Extra                    |
    -|----|--------------------|-------------|--------|-------------------|--------|--------------------------|
     | 1  | PRIMARY            | e           | ref    | ix_firstname      | 233    | Using index              |
     | 2  | DEPENDENT SUBQUERY | de          | ref    | ix_empno_fromdate | 1      | Using index              |
     | 2  | DEPENDENT SUBQUERY | dm          | ref    | PRIMARY           | 2      | Using index              |
    -|----|--------------------|-------------|--------|-------------------|--------|--------------------------|

    -- 안쪽(Inner)의 서브쿼리 결과가 바깥쪽(Outer) SELECT 쿼리의 칼럼에 의존적이기에 DEPENDENT 키워드 붙음
    -- DEPENDENT UNION과 마찬가지로 DEPENDENT SUBQUERY 또한 외부 쿼리 먼저 수행 후 내부 쿼리(서브 쿼리) 실행되야 하므로 (DEPENDENT 키워드 없는) 일반 서브쿼리보다 처리 속도 느릴 때가 많다
    ```
  - DERIVED : 단위 SELECT 쿼리의 실행 결과로 메모리나 디스크에 임시 테이블 생성하는 것을 의미, select_type이 DERIVED인 경우 생성되는 임시 테이블을 파생 테이블
    ```sql
    mysql>  EXPLAIN
    				SELECT *
    				FROM ( SELECT de.emp_no FROM dept_emp de GROUP BY de.emp_no ) tb,
    					employees e
    				WHERE e.emp_no = tb.emp_no;

    -|----|--------------------|-------------|------- |-------------------|--------|--------------------------|
     | id | select_type        | table       | type   | key               | rows   | Extra                    |
    -|----|--------------------|-------------|--------|-------------------|--------|--------------------------|
     | 1  | PRIMARY            | <derived2>  | ALL    | NULL              | 331143 | NULL                     |
     | 2  | PRIMARY            | e           | eq_ref | PRIMARY           | 1      | NULL                     |
     | 2  | DERIVED            | de          | index  | ix_empno_fromdate | 331143 | Using index              |
    -|----|--------------------|-------------|--------|-------------------|--------|--------------------------|

    -- FROM 절의 서브쿼리를 임시 테이블로 만들어서 처리
    ```
  - DEPENDENT DERIVED : 레터럴 조인(LATERAL JOIN) 기능이 추가되면서 FROM절의 서브쿼리에서도 외부 컬럼을 참조 가능
    ```sql
    mysql>  EXPLAIN
    				SELECT *
    				FROM employees e
    				LEFT JOIN LATERAL
    					( SELECT *
    						FROM salaries s
    						WHERE s.emp_no=e.emp_no
    						ORDER BY s.from_date DESC LIMIT 2 ) AS s2 ON s2.emp_no=e.emp_no

    -|----|--------------------|-------------|------- |-------------------|----------------------------|
     | id | select_type        | table       | type   | key               | Extra                      |
    -|----|--------------------|-------------|--------|-------------------|----------------------------|
     | 1  | PRIMARY            | e           | ALL    | NULL              | Rematerialize (<derived2>) |
     | 1  | PRIMARY            | <derived2>  | ref    | <auto_key0>       | NULL                       |
     | 2  | DEPENDENT DERIVED  | s           | ref    | PRIMARY           | Using filesort             |
    -|----|--------------------|-------------|--------|-------------------|----------------------------|

    -- DEPENDENT DERIVED 키워드는 해당 테이블이 레터럴 조인으로 사용된 것을 의미
    ```
  - UNCACHEABLE SUBQUERY : 조건이 똑같은 서브쿼리 실행 시 다시 실행하지 않고 이전의 실행 결과 그대로 사용할 수 있게 서브쿼리의 결과를 내부적인 캐시 공간에 저장(여기서 서브쿼리 캐시는 쿼리 캐시나 파생 테이블과 전혀 무관)
    - SUBQUERY : 바깥쪽(OUTER)의 영향을 받지 않으므로 처음 한 번만 실행해 그 결과를 캐시하고 필요 시 캐시된 결과 이용
    - DEPENDENT SUBQUERY는 의존하는 바깥쪽(OUTER) 쿼리의 칼럼 값 단위를 캐시하고 사용
    - 캐시 사용하지 못하는 요소
      1. 사용자 변수가 서브쿼리에 사용된 경우
      2. NOT-DETERMINISTIC 속성 스토이드 루틴이 서브쿼리 내에 사용된 경우
      3. UUID()나 RAND()와 같이 결괏값 호출할 때마다 달라지는 함수가 서브쿼리에 사용된 경우
    ```sql
    mysql>  EXPLAIN
    				SELECT *
    				FROM employees e WHERE e.emp_no = (
    					SELECT @status FROM dept_emp de WHERE de.dept_no='d005');

    -|----|----------------------|-------------|------- |-------------------|-----------|----------------------------|
     | id | select_type          | table       | type   | key               | rows      | Extra                      |
    -|----|----------------------|-------------|--------|-------------------|-----------|----------------------------|
     | 1  | PRIMARY              | e           | ALL    | NULL              | 300252    | Using where                |
     | 2  | UNCACHEABLE SUBQUERY | de          | ref    | PRIMARY           | 165571    | Using index                |
    -|----|----------------------|-------------|--------|-------------------|-----------|----------------------------|

    -- 사용자 변수(@status)가 사용된 쿼리 예제, 이 경우 WHERE 절에 사용된 단위 쿼리의 select_type이 UNCACHEABLE SUBQUERY로 표시되는 것을 확인
    ```
  - UNCACHEABLE UNION : UNION과 UNCACHEABLE 두 개 키워드 속성이 혼합된 select_type 의미
  - MATERIALIZED : 주로 FROM 절이나 IN(subquery) 형태의 쿼리에 사용된 서브쿼리의 최적화를 위해 사용된다.
    ```sql
    mysql>  EXPLAIN
    				SELECT *
    				FROM employees e
    				WHERE e.emp_no IN ( SELECT emp_no FROM salaries WHERE salary BETWEEN 100 AND 1000 );

    -|----|----------------------|-------------|------- |-------------------|-----------|----------------------------|
     | id | select_type          | table       | type   | key               | rows      | Extra                      |
    -|----|----------------------|-------------|--------|-------------------|-----------|----------------------------|
     | 1  | SIMPLE               | <subquery2> | ALL    | NULL              | NULL      | NULL                       |
     | 1  | SIMPLE               | e           | eq_ref | PRIMARY           | 1         | NULL                       |
     | 2  | MATERIALIZED         | salaries    | range  | ix_salary         | 1         | Using where; USING index   |
    -|----|----------------------|-------------|--------|-------------------|-----------|----------------------------|

    -- e다음 쿼리는 급여가 100보다 크거나 같고 1000보다 작거나 같은 직원 정보 모두 가져옴
    -- 서브쿼리 내용을 임시 테이블로 구체화(MATERIALIZED)한 후, 임시 테이블과 employees 테이블 조인하는 형태로 최적화
    ```

- table 칼럼

  - MySQL 서버 실행 계획은 단위 SELECT 쿼리 기준이 아니라 테이블 기준으로 표시
    ```sql
    mysql>  EXPLAIN SELECT NOW();
    mysql>  EXPLAIN SELECT NOW() FROM DUAL;

    -- 오라클의 경우 FROM이 없으면 오류 발생하지만 MySQL의 경우 그렇지 않다.

    -|----|----------------------|-------------|-------------------|-----------|-----------------|
     | id | select_type          | table       | key               | key_len   | Extra           |
    -|----|----------------------|-------------|-------------------|-----------|-----------------|
     | 1  | SIMPLE               | NULL        | NULL              | NULL      | No tables used  |
    -|----|----------------------|-------------|-------------------|-----------|-----------------|
    ```
  - table 칼럼에 <>로 둘러쌍니 이름이 명시된 경우 많은데 이 테이블은 임시 테이블 의미, <>안에 항상 표시되는 숫자는 단위 SELECT 쿼리의 id 값을 지칭
    ```sql
    mysql>  EXPLAIN
    				SELECT *
    				FROM employees e
    				WHERE e.emp_no IN ( SELECT emp_no FROM salaries WHERE salary BETWEEN 100 AND 1000 );

    -|----|----------------------|-------------|------- |-------------------|-----------|----------------------------|
     | id | select_type          | table       | type   | key               | rows      | Extra                      |
    -|----|----------------------|-------------|--------|-------------------|-----------|----------------------------|
     | 1  | PRIMARY              | <derived2>  | ALL    | NULL              | 331143    | NULL                       |
     | 1  | PRIMARY              | e           | eq_ref | PRIMARY           | 1         | NULL                       |
     | 2  | DERIVED              | de          | index  | ix_empno_fromdate | 331143    | Using index                |
    -|----|----------------------|-------------|--------|-------------------|-----------|----------------------------|

    -- <derived2>의 경우 단위 SELECT 쿼리의 id 값이 2인 실행 계획으로부터 만들어진 파생 테이블
    -- 단위 SELECT 쿼리의 id 값이 2인 실행 계획(실행 계획의 최하위 라인)은 dept_emp 테이블로부터 SELECT된 결과가 저장된 파생 테이블이라는 점
    -- 1. 첫 번째 라인의 테이블이 <derived2>라는 것으로 보아 이 라인보다 id값이 2인 라인이 먼저 실행되고 그 결과가 파생 테이블로 준비되야 한다는 것을 알 수 있다.
    -- 2. 세 번째 라인(id값이 2인 라인)을 보면 select_type 컬럼의 값이 DERIVED로 표시돼 있다. 즉 이 라인은 table 컬럼에 표시된 dept_emp 테이블을 읽어 파생 테이블을 생성
    -- 3. 세 번째 라인의 분석이 끝났으므로 다시 실행 계획의 첫 번째 라인으로 돌아감
    -- 4. 첫 번째 라인과 두 번째 라인은 같은 id를 갖고 있어 2개 테이블이 조인되는 쿼리, 그런데 <derived2> 테이블이 e 테이블 보다 먼저 표시되어 있기에 <derived2>가 드라이빙 테이블, e 테이블이 드리븐 테이블이 된다.
    ```

- partitions 컬럼

  ```sql
  mysql>  CREATE TABLE employees_2 (
  						emp_no int NOT NULL,
  						birth_date DATE NOT NULL,
  						first_name VARCHAR(4) NOT NULL,
  						last_name VARCHAR(4) NOT NULL,
  						gender ENUM('M', 'F') NOT NULL,
  						hire_date DATE NOT NULL,
  						PRIMARY KEY (emp_no, hire_date)
  				) PARTITION BY RANGE COLUMNS(hire_date)
  				( PARTITION p1986_1990 VALUES LESS THAN ('1990-01-01'),
  					PARTITION p1991_1995 VALUES LESS THAN ('1995-01-01'),
  					PARTITION p1996_2000 VALUES LESS THAN ('2000-01-01'),
  					PARTITION p2001_2005 VALUES LESS THAN ('2006-01-01'));

  mysql>  INSERT INTO employees_2 SELECT * FROM employees;

  mysql>  EXPLAIN
  				SELECT *
  				FROM employees_2
  				WHERE hire_date BETWEEN '1999-11-15' AND '2000-01-15';

  -- 쿼리 조회 데이터가 p1996_2000과 p2001_2005 파티션에 저장되어 있다
  -- 실제 옵티마이저는 쿼리의 hire_date 컬럼 조건 보고 필요 데이터는 p1996_2000과 p2001_2005 파티션에 있다는 것을 안다
  -- 실행 계획에서도 나머지 파티션에 대해서 분석 X
  -- 파티션 여러 개인 테이블에서 불필요한 파티션 빼고 쿼리 수행하기 위해 접근해야 할 것으로 판단되는 테이블만 골라내는 과정 = 파티션 프루닝(Partition pruning)

  -|----|----------------------|-------------|--------------------------|---------|-----------|
   | id | select_type          | table       | partitions               | type    | rows      |
  -|----|----------------------|-------------|--------------------------|---------|-----------|
   | 1  | SIMPLE               | employees_2 | p1996_2000, p2001_2005   | ALL     | 21743     |
  -|----|----------------------|-------------|--------------------------|---------|-----------|

  -- 실행 계획을 통해 p1996_2000, p2001_2005 2개의 ㅌ파티션만 접근했다는 것을 파악
  -- type 컬럼이 ALL이다 -> 풀 테이블 스캔으로 쿼리 처리
  -- 어떻게 풀 테이블 스캔으로 테이블의 일부를 읽는가> -> 대부분 RDBMS에서 지원하는 파티션은 물리적으로 개별 테이블처럼 별도의 저장 공간 -> p1996_2000, p2001_2005 2개의 파티션을 풀 스캔
  ```

- type 컬럼

  - type 이후 컬럼은 MySQL 서버가 각 테이블 레코드를 어떤 방식으로 읽었는가 → 인텍스 사용했는지, 아니면 풀 테이블 스캔을 했는가와 같은 것을 의미
  - MySQL 매뉴얼에서 type 컬럼을 “조인 타입”으로 소개 또한 MySQL에서 하나의 테이블로부터 레코드 읽는 작업도 조인처럼 처리 → SELECT 쿼리 테이블 갯우에 관계없이 실행 계획의 type 컬럼ㅇ르 “조인 타입”으로 명시
  - 하나의 단위 SELECT 쿼리는 12개의 접근법 중 하나만 사용 가능
    1. system : 레코드가 1건만 존재하는 테이블 또는 한 건도 존재하지 않는 테이블 참조하는 형태의 접근법 → InnoDB 엔진 사용하는 테이블에서 나타지 않고 MyISAM이나 MEMORY 테이블에서만 사용되는 접근법

       ```sql
       mysql>  CREATE TABLE tb_dual (fd1 int NOT NULL) ENGINE=MyISAM;
       mysql>  INSERT INTO tb_dual VALUES (1);

       mysql>  EXPLAIN SELECT * FROM tb_dual;

       -|----|----------------------|-------------|---------|-----------|-----------|
        | id | select_type          | table       | type    | rows      | Extra     |
       -|----|----------------------|-------------|---------|-----------|-----------|
        | 1  | SIMPLE               | tb_dual     | system  | 1         | NULL      |
       -|----|----------------------|-------------|---------|-----------|-----------|

       mysql>  CREATE TABLE tb_dual (fd1 int NOT NULL) ENGINE=InnoDB;
       mysql>  INSERT INTO tb_dual VALUES (1);

       mysql>  EXPLAIN SELECT * FROM tb_dual;

       -|----|----------------------|-------------|---------|-----------|-----------|
        | id | select_type          | table       | type    | rows      | Extra     |
       -|----|----------------------|-------------|---------|-----------|-----------|
        | 1  | SIMPLE               | tb_dual     | ALL     | 1         | NULL      |
       -|----|----------------------|-------------|---------|-----------|-----------|

       ```

    2. const : 레코드 건수와 관계없이 쿼리가 프라이머리 키나 유니크 키 컬럼을 이용하는 WHERE 조건절을 가지고 반드시 1건을 반환하는 쿼리의 처리 방식 → 다른 DBMS에서는 이를 유니크 인덱스 스캔(UNIQUE INDEX SCAN)이라고 표현

       ```sql
       mysql>  EXPLAIN
       				SELECT * FROM employees WHERE emp_no=10001;

       -|----|----------------------|-------------|---------|-----------|-----------|
        | id | select_type          | table       | type    | key       | key_len   |
       -|----|----------------------|-------------|---------|-----------|-----------|
        | 1  | SIMPLE               | employees   | const   | PRIMARY   | 4         |
       -|----|----------------------|-------------|---------|-----------|-----------|

       mysql>  EXPLAIN
       				SELECT * FROM dept_emp WHERE dept_no='d005';

       -|----|----------------------|-------------|---------|-----------|-----------|
        | id | select_type          | table       | type    | key       | rows      |
       -|----|----------------------|-------------|---------|-----------|-----------|
        | 1  | SIMPLE               | employees   | ref     | PRIMARY   | 165571    |
       -|----|----------------------|-------------|---------|-----------|-----------|

       -- 다중 컬럼으로 구성된 프라이머리 키나 유니크 키 중에서 인덱스의 일부 컬럼만 조건으로 사용 시 const 타입 접근법 사용X
       -- 실제 레코드가 1건만 저장되어있다고 확실할 수 없기에 -> 이 경우 type 컬럼에 ref로 표시

       mysql>  EXPLAIN
       				SELECT * FROM dept_emp WHERE dept_no='d005' AND emp_no=10001;

       -|----|----------------------|-------------|---------|-----------|-----------|-----------|
        | id | select_type          | table       | type    | key       | key_len   | rows      |
       -|----|----------------------|-------------|---------|-----------|-----------|-----------|
        | 1  | SIMPLE               | employees   | const   | PRIMARY   | 20        | 1         |
       -|----|----------------------|-------------|---------|-----------|-----------|-----------|

       -- 그러나 프라이머리 키나 유니크 인덱스의 모든 컬럼을 동등 조건으로 WHERE절에 명시할 경우 const 접근법 사용

       ```

    3. eq_ref : 여러 테이블이 조인되는 쿼리의 실행 계획에서만 표시, 조인에서 처음 읽은 테이블의 컬럼값을 그 다음 읽어야 할 테이블의 프라이머리 키나 유니크 키 컬럼의 검색 조건에 사용할 때를 가르켜 eq_ref라고 한다 → 이때 두번째 이휴 읽히는 테이블 유니크 키 검색 시 그 유니크 인덱스는 NOT NULL & 다중 컬럼으로 만들어진 프라이머리 키나 유니크 인덱스 경우 인덱스의 모든 컬럼이 WHERE절에 명시 되어야 eq_ref 접근법 사용 가능

       ```sql
       mysql>  EXPLAIN
       				SELECT * FROM dept_emp de, employees e
       				WHERE e.emp_no=de.emp_no AND de.dept_no='d005';

       -|----|----------------------|-------------|---------|-----------|-----------|-----------|
        | id | select_type          | table       | type    | key       | key_len   | rows      |
       -|----|----------------------|-------------|---------|-----------|-----------|-----------|
        | 1  | SIMPLE               | de          | ref     | PRIMARY   | 16        | 165571    |
        | 1  | SIMPLE               | e           | eq_ref  | PRIMARY   | 4         | 1         |
       -|----|----------------------|-------------|---------|-----------|-----------|-----------|

       -- c첫 번째 라인과 두 번째 라인의 id 값이 1로 같으므로 두개의 테이블이 조인으로 실행
       -- dept_emp 테이블이 실행 계획의 위쪽에 있으므로 dept_emp 테이블 먼저 일고 e.emp_no=de.emp_no 조건 이용해 employees 테이블 검색
       -- employees 테이블의 emp_no는 프라이머리 키라서 실행 계획의 두 번째 라인은 type 컬럼이 eq_ref로 표시
       ```

    4. ref : eq_ref와 달리 조인 순서와 관계없이 사용되며 프라이머리 키나 유니크 등의 제약 조건도 없다. 인덱스 종류와 관계없이 동등 조건으로 검색 시 ref 접근법 사용 → 반환 레코드가 반드시 1건이라는 보장이 없으므로 const나 eq_ref보다 빠르지 않다 But 동등 조건으로만 비교되므로 매우 빠른 레코드 조회 방법 중 하나

       ```sql
       mysql>  EXPLAIN
       				SELECT * FROM dept_emp WHERE dept_no='d005';

       -|----|----------------------|-------------|---------|-----------|-----------|---------|
        | id | select_type          | table       | type    | key       | key_len   | ref     |
       -|----|----------------------|-------------|---------|-----------|-----------|---------|
        | 1  | SIMPLE               | dept_emp    | ref     | PRIMARY   | 16        | const   |
       -|----|----------------------|-------------|---------|-----------|-----------|---------|

       -- v프라이머리 키 구성하는 컬럼(dept_no, emp_no) 중에서 일부만 동등 조건으로 WHERE 절에 명시 -> 일치하는게 1개라는 보장없다.
       -- 따라서 const가 아닌 ref 접근법 사용 실행 계획의 ref 컬럼값에 const 명시 -> 이는 const 접근법이 아니라 ref 접근법에서 값 비교에 사용된 입력값이 상수('d005')였음을 의미

       ```

    5. fulltext : 전문 검색(Full-text Search) 인덱스 사용해 레코드 읽는 접근법 의미 → 전문 검색 인덱스는 통계 정보가 관리 되지 않으며 전혀 다른 SQL 문법 사용해야한다.(MATCH AGAINST 구문을 사용해 실행 + 테이블에 전문 인덱스 없으면 쿼리 오류 발생하고 중지)

       ```sql
       mysql>  CREATE TABLE employee_name (
       						emp_no int NOT NULL,
       						first_name VARCHAR(4) NOT NULL,
       						last_name VARCHAR(4) NOT NULL,
       						PRIMARY KEY (emp_no)
       						FULLTEXT KEY fx_name (first_name, last_name) WITH PARSER ngram
       				) ENGINE=InnoDB;

       mysql>  EXPLAIN
       				SELECT *
       				FROM employee_name
       				WHERE emp_no=10001
       				AND emp_no BETWEEN 10001 AND 10005
       				AND MATCH(first_name, last_name) AGAINST('Facello' IN BOOLEAN MODE);

       -|----|----------------------|---------------|---------|-----------|-----------|
        | id | select_type          | table         | type    | key       | key_len   |
       -|----|----------------------|---------------|---------|-----------|-----------|
        | 1  | SIMPLE               | employee_name | const   | PRIMARY   | 4         |
       -|----|----------------------|---------------|---------|-----------|-----------|

       -- 첫 번째 조건은 employee_name 테이블의 프라이머리 키를 1건만 조회하는 const 타입
       -- 두 번째 조건은 밑에서 설명할 range 타입 조건
       -- 마지막 조건은 전문 검색 조건
       -- 옵테마이저가 최종적으로 선택한 것은 const 접근법 -> IF 첫번째 조건이 없으면 어떤 조건을 택할까?

       mysql>  EXPLAIN
       				SELECT *
       				FROM employee_name
       				WHERE emp_no BETWEEN 10001 AND 10005
       				AND MATCH(first_name, last_name) AGAINST('Facello' IN BOOLEAN MODE);

       -|----|----------------------|---------------|----------|-----------|-----------|-----------------------------------|
        | id | select_type          | table         | type     | key       | key_len   | Extra                             |
       -|----|----------------------|---------------|----------|-----------|-----------|-----------------------------------|
        | 1  | SIMPLE               | employee_name | fulltext | fx_name   | 4         | Using where; Ft_hints: no_ranking |
       -|----|----------------------|---------------|----------|-----------|-----------|-----------------------------------|

       -- 세 번째 조건 선택 -> 쿼리에서 전문 건색 조건 사용 시 MySQL 서버는 주저 없이 fulltext 접근법 사용
       -- 그러나 지금까지 fulltext보다 일반 인덱스 사용하는 range 접근법이 더 빠른 경우가 더 많았다. -> 조건별로 성능 확인해라
       ```

    6. ref_or_null : ref 접근법과 같은데 NULL 비교가 추가된 형태 → ref 방식 또는 NULL 비교(IS NULL) 접근법 의미

       ```sql
       mysql>  EXPLAIN
       				SELECT * FROM titles
       				WHERE to_date='1985-03-01' OR to_date IS NULL;

       -|----|----------------------|-------------|-------------|-----------|-----------|---------|------|
        | id | select_type          | table       | type        | key       | key_len   | ref     | rows |
       -|----|----------------------|-------------|-------------|-----------|-----------|---------|------|
        | 1  | SIMPLE               | titles      | ref_or_null | ix_todate | 4         | const   | 2    |
       -|----|----------------------|-------------|-------------|-----------|-----------|---------|------|
       ```

    7. unique_subquery : WHERE 조건절에서 사용하는 IN 형태의 쿼리 위한 접근법 → 서브쿼리에서 중복되지 않는 유니크한 값만을 반환할 때 해당 접근법 사용

       ```sql
       mysql>  EXPLAIN
       				SELECT * FROM departments
       				WHERE dept_no IN ( SELECT dept_no FROM dept_emp WHERE emp_no=10001);

       -|----|----------------------|-------------|-----------------|-------------|-----------|
        | id | select_type          | table       | type            | key         | key_len   |
       -|----|----------------------|-------------|-----------------|-------------|-----------|
        | 1  | PRIMARY              | departments | index           | ux_deptname | 162       |
        | 1  | DEPENDENT SUBQUERY   | dept_emp    | unique_subquery | PRIMARY     | 20        |
       -|----|----------------------|-------------|-----------------|-------------|-----------|
       ```

    8. index_subquery : IN 특성상 괄호 안에 있는 값 목록에서 중복된 값이 먼저 제거 → 그러나 업무 특성상 IN에서 서브쿼리가 중복된 값을 반환할 수도 있다. 이때 서브쿼리의 중복된 값을 인덱스 통해 제거 가능 → index_subquery 접근법 사용
       1. unique_subquery : 서브쿼리 반환 값에는 중복 없으므로 중복 제거 필요 X
       2. index_subquery : 서브쿼리 반환 값에 중복된 값 있을 수 있지만 인덱스 이용해 중복 값 제거
    9. range : 흔히 아는 인덱스 레이니 스캔 형태 접근법, range는 인덱스를 하나의 값이 아니라 범위로 검색하는 경우 의미(<, >, IS NULL, BETWEEN, IN, LIKE)

       ```sql
       mysql>  EXPLAIN
       				SELECT *
       				FROM employees
       				WHERE emp_no BETWEEN 10002 AND 10004;

       -|----|----------------------|-------------|-----------------|-------------|-----------|--------|
        | id | select_type          | table       | type            | key         | key_len   | rows   |
       -|----|----------------------|-------------|-----------------|-------------|-----------|--------|
        | 1  | SIMPLE               | employees   | range           | PRIMARY     | 4         | 3      |
       -|----|----------------------|-------------|-----------------|-------------|-----------|--------|
       ```

    10. index_merge : 2개 이상의 인덱스 사용해 각각 검색 결과 만든 후 병합해 처리하는 방식 → 그렇게 효율적으로 작동하지 않는다

        1. 여러 인덱스 읽어야 하므로 range보다 일반적으로 효율성 떨어짐
        2. 전문 검색 인덱스 사용하는 쿼리에서 index_merge 적용X
        3. index_merge 접근법 처리된 결과는 항상 2개 이상의 집합이 되므로 두 집합의 교집합이나 합집합 또는 중복 제거 같은 부가적인 작업 필요

        ```sql
        mysql>  EXPLAIN
        				SELECT *
        				FROM employees
        				WHERE emp_no BETWEEN 10001 AND 11000 OR first_name='Smith';

        -|----|----------------------|-----------------------|-----------|-------------------------------------------------|
         | id | select_type          | key                   | key_len   | Extra                                           |
        -|----|----------------------|-----------------------|-----------|-------------------------------------------------|
         | 1  | index_merge          | PRIMARY, ix_firstname | 4, 58     | Using union(PRIMARY, ix_firstname); Using where |
        -|----|----------------------|-----------------------|-----------|-------------------------------------------------|

        -- 위의 쿼리는 범위 조건은 프라이머리 키로 이름 검색 조건은 ix_firstname 인덱스를 이용해 조회 후 두 결과를 병합
        ```

    11. index : 인덱스를 처음부터 끝까지 읽는 인덱스 풀 스캔 → range와 같이 인덱스의 필요 부분만 읽는 것을 의미하지 않는다. → 밑의 조건 중 (1번+2번) 혹은 (1번+3번) 조건 충족하는 쿼리에서 사용

        1. range나 const, ref 같은 접근법으로 인덱스 사용하지 못하는 경우
        2. 인덱스 포함된 컬럼만으로 처리할 수 있는 쿼리인 경우(즉, 데이터 파일 읽지 않아도 되는 경우)
        3. 인덱스 이용해 정렬이나 그루핑 가능한 경우(즉, 별도의 정렬 작업 피할 수 있는 경우)

        ```sql
        mysql>  EXPLAIN
        				SELECT * FROM departments ORDER BY dept_name DESC LIMIT 10;

        -|----|----------------------|-------------|-----------------|-------------|-----------|--------|
         | id | select_type          | table       | type            | key         | key_len   | rows   |
        -|----|----------------------|-------------|-----------------|-------------|-----------|--------|
         | 1  | SIMPLE               | departments | index           | ux_deptname | 162       | 9      |
        -|----|----------------------|-------------|-----------------|-------------|-----------|--------|

        -- 해당 쿼리는 WHERE 조건 없어 range, const, ref 사용 못한다
        -- 그러나 정렬하려는 컬럼은 인덱스가 있으므로 별도의 정렬 처리 피하고 index 접근법 사용
        ```

    12. ALL : 풀 테이블 스캔 의미하는 접근법 → 테이블 처음부터 끝까지 읽은 후 불필요한 레코드 제거 후 반환(지금까지 접근법으로 처리할 수 없을 때 사용하는 최후의 보루 즉 가장 비효율적인 방법)

- possible_keys 칼럼 : 옵티마이저가 최적의 실행 계획 만들기 위한 후보로 선정한 접근법에서 사용되는 인덱스 목록, 즉 ‘사용될 법했던 인덱스 목록’ → 실행 계획 확인 시 특별한 경우 제외하고 무시해도 된다.(나열되었다고 해서 해당 인덱스를 사용했다고 판단X)
- key 칼럼 : 최종 선택된 실행 계획에서 사용된 인덱스
- key_len 칼럼 : 실제 업무에서 사용된 테이블은 다중 칼럼으로 만들어진 인덱스가 더 많다. → 쿼리 처리하기 위해 다중 컬럼으로 구성된 인덱스에서 몇 개의 컬럼까지 사용했는지 명시(정확히 각 레코드에서 몇 바이트까지 사용되었는지)
  ```sql
  mysql>  EXPLAIN
  				SELECT * FROM dept_emp WHERE dept_no='d005';

  -|----|----------------------|-------------|-----------------|-------------|-----------|
   | id | select_type          | table       | type            | key         | key_len   |
  -|----|----------------------|-------------|-----------------|-------------|-----------|
   | 1  | SIMPLE               | dept_emp    | ref             | PRIMARY     | 16        |
  -|----|----------------------|-------------|-----------------|-------------|-----------|

  -- dept_no는 CHAR(4)이기 때문에 프라이머리 키에서 앞쪽 16바이트만 유효하게 사용

  mysql>  EXPLAIN
  				SELECT * FROM dept_emp WHERE dept_no='d005' AND emp_no=10001;

  -|----|----------------------|-------------|-----------------|-------------|-----------|
   | id | select_type          | table       | type            | key         | key_len   |
  -|----|----------------------|-------------|-----------------|-------------|-----------|
   | 1  | SIMPLE               | dept_emp    | const           | PRIMARY     | 20        |
  -|----|----------------------|-------------|-----------------|-------------|-----------|

  -- emp_no 칼럼 타입은 INTEGER이며 4버아투 차지
  -- 프라이머리 키의 dept_no뿐 아니라 emp_no까지 사용
  -- 따라서 key_len은 dept_no 칼럼 길이와 emp_no 칼럼 길이의 합인 20으로 표시

  mysql>  CREATE TABLE titles (
  						emp_no int NOT NULL,
  						title varchar(50) NOT NULL,
  						from_date date NOT NULL,
  						to_date date DEFAULT NULL,
  						PRIMARY KEY (emp_no, from_date, title),
  						KEY ix_todate(to_date)
  				);

  mysql>  EXPLAIN
  				SELECT * FROM titles WHERE to_date<='1985-10-10'

  -|----|----------------------|-------------|-----------------|-------------|-----------|
   | id | select_type          | table       | type            | key         | key_len   |
  -|----|----------------------|-------------|-----------------|-------------|-----------|
   | 1  | SIMPLE               | titles      | range           | ix_todate   | 4         |
  -|----|----------------------|-------------|-----------------|-------------|-----------|

  -- MySQL에서 Date는 3바이트인데 왜 key_len에 4가 출력되었을까?
  -- DATE 타입 사용하면서 NULL 저장할 수 있는 (NULLABLE) 칼럼이 정의 되어 있기에 1바이트 더 소모 -> (3+1)로 4바이트 소모
  ```
- ref 칼럼 : 참조 조건(Equal 비교 조건)으로 어떤 값이 제공됐는지 보여준다.
  - 상수값 지정 : ref 값은 const
  - 다른 테이블 컬럼값 : 그 테이블과 컬럼명
  - 가끔 func으로 표시 : ‘Function’의 약어로 콜레이션 변환이나 값 자체의 연산을 거쳐서 참조
    ```sql
    mysql>  EXPLAIN
    				SELECT *
    				FROM employees e, dept_emp de
    				WHERE e.emp_no = de.emp_no;

    -|----|----------------------|-------------|-----------------|-------------|---------------------|
     | id | select_type          | table       | type            | key         | ref                 |
    -|----|----------------------|-------------|-----------------|-------------|---------------------|
     | 1  | SIMPLE               | de          | ALL             | NULL        | NULL                |
     | 1  | SIMPLE               | e           | eq_ref          | PRIMARY     | employees.de.emp_no |
    -|----|----------------------|-------------|-----------------|-------------|---------------------|

    -- employees 테이블과 dept_emp 조인하는데 emp_no 컬럼의 값에 대해 아무 가공 일어나지 않음 -> ref에 조인 칼럼 대상 이름 그대로 표시

    mysql>  EXPLAIN
    				SELECT *
    				FROM employees e, dept_emp de
    				WHERE e.emp_no = (de.emp_no-1);

    -|----|----------------------|-------------|-----------------|-------------|------|
     | id | select_type          | table       | type            | key         | ref  |
    -|----|----------------------|-------------|-----------------|-------------|------|
     | 1  | SIMPLE               | de          | ALL             | NULL        | NULL |
     | 1  | SIMPLE               | e           | eq_ref          | PRIMARY     | func |
    -|----|----------------------|-------------|-----------------|-------------|------|

    -- dept_emp 테이블의 emp_no 값에서 1 뺀 값으로 employees 테이블과 조인 -> ref에 조인 칼럼 이름이 아닌 func라고 표시
    ```
- rows 칼럼 : 실행 계획의 효율성 판단을 위해 예측한 레코드 건수 보여준다. 각 스토리지 엔진별로 가진 통계 정보 참조해 MySQL 옵티마이저가 산출한 예상값이므로 정확하지 않다. 또한 레코드의 예측치가 아닌 쿼리 처리 위해 얼마나 많은 레코드 읽고 체크해야하는지 의미 → 실행 계획의 rows 컬럼에 출력되는 값과 실제 쿼리 반환 레코드 건수는 일치하지 않는 경우가 많다.
  ```sql
  mysql>  EXPLAIN
  				SELECT *
  				FROM dept_emp
  				WHERE from_date>='1985-01-01';

  -|----|----------------------|-------------|-----------------|-------------|---------|------------|
   | id | select_type          | table       | type            | key         | key_len | rows       |
  -|----|----------------------|-------------|-----------------|-------------|---------|------------|
   | 1  | SIMPLE               | dept_emp    | ALL             | NULL        | NULL    | 331143     |
  -|----|----------------------|-------------|-----------------|-------------|---------|------------|

  -- rows 컬럼을 보면 대략 331143건의 레코드 읽어야된다고 예측
  -- 전체 레코드 갯수가 331143이기에 인덱스를 이용하기보다 풀 테이블 스캔 수행

  mysql>  EXPLAIN
  				SELECT *
  				FROM dept_emp
  				WHERE from_date>='1985-01-01';

  -|----|----------------------|-------------|-----------------|-------------|---------|------------|
   | id | select_type          | table       | type            | key         | key_len | rows       |
  -|----|----------------------|-------------|-----------------|-------------|---------|------------|
   | 1  | SIMPLE               | dept_emp    | range           | ix_fromdate | 3       | 292        |
  -|----|----------------------|-------------|-----------------|-------------|---------|------------|

  -- rows 컬럼을 보면 대략 292건의 레코드 읽어야된다고 예측
  -- 인덱스를 이용해 range 접근법 사용
  ```
- filtered 컬럼
  ```sql
  mysql>  EXPLAIN
  				SELECT *
  				FROM employees e, salaries s
  				WHERE e.first_name='Matt'
  					AND e.hire_date BETWEEN '1990-01-01' AND '1991-01-01'
  					AND s.emp_no=e.emp_no
  					AND s.from_date BETWEEN '1990-01-01' AND '1991-01-01'
  					AND s.salary BETWEEN 50000 AND 60000;

  -|----|----------------------|-------------|-----------------|--------------|------|----------|
   | id | select_type          | table       | type            | key          | rows | filtered |
  -|----|----------------------|-------------|-----------------|--------------|------|----------|
   | 1  | SIMPLE               | e           | ref             | ix_firstname | 233  | 16.03    |
   | 1  | SIMPLE               | s           | ref             | PRIMARY      | 10   | 0.48     |
  -|----|----------------------|-------------|-----------------|--------------|------|----------|

  -- employees 테이블 인덱스 조건에 일치하는 레코드는 233건, 그 중 16.03만 인덱스 사용 못하는 BETWEEN 조건에 일치
  -- filtered 컬럼의 값은 필터링되고 남은 레코드의 비율 -> 조인 수행한 레코드 건수는 대략 37(233 * 0.16.03)

  mysql>  EXPLAIN
  				SELECT /*+ JOIN_ORDER(s, 3) */ *
  				FROM employees e, salaries s
  				WHERE e.first_name='Matt'
  					AND e.hire_date BETWEEN '1990-01-01' AND '1991-01-01'
  					AND s.emp_no=e.emp_no
  					AND s.from_date BETWEEN '1990-01-01' AND '1991-01-01'
  					AND s.salary BETWEEN 50000 AND 60000;

  -|----|----------------------|-------------|-----------------|--------------|------|----------|
   | id | select_type          | table       | type            | key          | rows | filtered |
  -|----|----------------------|-------------|-----------------|--------------|------|----------|
   | 1  | SIMPLE               | s           | range           | ix_salary    | 3314 | 11.11    |
   | 1  | SIMPLE               | e           | eq_ref          | PRIMARY      | 1    | 5.00     |
  -|----|----------------------|-------------|-----------------|--------------|------|----------|

  -- 조인 순서 거꾸로 하면 대략 salaries 테이블에서 368건 (3314*0.1111) 조건 일치해서 employees 테이블과 조인
  -- 조인 과정 줄이고 그 과정에서 읽어온 데이터 저장해둘 메모리 사용량 낮추기 위해 대상 건수가 적은 테이블 선행 테이블 선택할 가능성이 높다
  -- filterd 칼럼 표시된 값이 얼마나 정확히 예측될 수 있냐에 따라 조인 성능이 달라진다.
  ```
- Extra 칼럼 : 쿼리 실행 계획에서 성능과 관련된 중요 내용이 Extra 컬럼에 자주 표시된다. 주로 내부적인 처리 알고리즈멩 대해 깊이 있는 내용을 보여준다.
  1. const row not found : const 접근법으로 테이블 읽었으나 레코드가 1건도 존재하지 않은 경우에 표시
  2. Deleting all rows : 스토리지 엔진 핸들러 차원에서 테이블 모든 레코드 삭제하는 기능 제공하는 스토리지 엔진 테이블 경우 “Deletin all rows” 무구 표시, WHERE 조건절 없는 DELETE 문장 실행 계획에 자주 표시되며 모든 레코드 삭제하는 핸들러 기능(API) 한 번 호출하여 처리됬다는 것을 의미
  3. Distinct

     ```sql
     mysql>  EXPLAIN
     				SELECT DISTINCT d.dept_no
     				FROM departments d, dept_emp de WHERE d.dept_no = de.dept_no;

     -|----|----------------------|-------------|-----------------|--------------|------------------------------|
      | id | select_type          | table       | type            | key          | Extra                        |
     -|----|----------------------|-------------|-----------------|--------------|------------------------------|
      | 1  | SIMPLE               | d           | index           | ux_deptname  | Using index; Using temporary |
      | 1  | SIMPLE               | de          | ref             | PRIMARY      | Using index; Distinct        |
     -|----|----------------------|-------------|-----------------|--------------|------------------------------|

     -- 조인하지 않아도 되는 항목은 모두 무시하고 꼭 필요한 것만 조인
     ```

  4. FirstMatch : 드라이빙 테이블 기준으로 드리븐 테이블에서 첫 번째로 일치하는 한 건만 검색

     ```sql
     mysql>  EXPLAIN
     				SELECT *
     				FROM employees
     				WHERE e.first_name='Matt'
     					AND e.emp_no IN (
     						SELECT t.emp_no
     						FROM titles t
     						WHERE t.from_date BETWEEN '1995-01-01' AND '1995-01-30'
     				);

     -|----|----------------------|-------------|-----------------|-------------------------------------------------|
      | id | select_type          | table       | type            | Extra                                           |
     -|----|----------------------|-------------|-----------------|-------------------------------------------------|
      | 1  | PRIMARY              | d1          | index           | Using index                                     |
      | 2  | SUBQUERY             | d2          | index_subquery  | Using where; Using index; Full scan on NULL key |
     -|----|----------------------|-------------|-----------------|-------------------------------------------------|

     ```

  5. Full scan on NULL key : col1 IN ( SELECT col2 FROM…) 에서 col1이 NULL인 된 경우 조건이 NULL IN (SELECT col2 FROM …)으로 바뀜, SQL 표준에서 NULL은 알 수 없는 값 따라서 정의대로 연산 수행하기 위해 다음과 같이 비교

     1. 서브쿼리가 1건이라도 결과 레코드 가진다면 최종 비교 결과는 NULL → 서브쿼리 사용된 테이블에 대해 풀 테이블 스캔을 해야 결과 알 수 있다.
     2. 서비쿼리가 1건도 결과 레코드 가지지 않는다면 최종 비교 결과는 FALSE

     ```sql
     mysql>  EXPLAIN
     				SELECT d.dept_no,
     							NULL IN (SELECT id.dept_name FROM departments d2) FROM departments d1;

     -|----|-------------|-----------------|--------------|------|------------------------------------------|
      | id | table       | type            | key          | rows | Extra                                    |
     -|----|-------------|-----------------|--------------|------|------------------------------------------|
      | 1  | e           | ref             | ix_firstname | 233  | NULL                                     |
      | 2  | t           | ref             | PRIMARY      | 1    | Using where; Using index; FirstMatch(e)  |
     -|----|-------------|-----------------|--------------|------|------------------------------------------|

     mysql>  EXPLAIN
     				SELECT *
     				FROM tb_test1
     				WHERE col1 IS NOT NULL
     					AND col1 IN(SELECT col2 FROM tb_test2);
     ```

  6. Impossible HAVING : HAVING 절 조건 만족하는 레코드 없는 경우

     ```sql
     mysql>  EXPLAIN
     				SELECT e.emp_no, COUNT(*) AS cnt
     				FROM employees e
     				WHERE e.emp_no=10001
     				GROUP BY e.emp_no
     				HAVING e.emp_no IS NULL;

     -|----|----------------------|-------------|-----------------|--------------|------------------------------|
      | id | select_type          | table       | type            | key          | Extra                        |
     -|----|----------------------|-------------|-----------------|--------------|------------------------------|
      | 1  | SIMPLE               | NULL        | NULL            | NULL         | Impossible HAVING            |
     -|----|----------------------|-------------|-----------------|--------------|------------------------------|

     -- e.emp_no는 프라이머리 키이면서 NOT NULL이기 떄문에 e.emp_no IS NULL; 조건 만족하는 것이 없다.
     ```

  7. Impossible WHERE : WHERE 절이 항상 FALSE가 될 수밖에 없는 상황

     ```sql
     mysql>  EXPLAIN
     				SELECT *
     				FROM employees e
     				WHERE e.emp_no IS NULL;

     -|----|----------------------|-------------|-----------------|---------------------|
      | id | select_type          | table       | type            | Extra               |
     -|----|----------------------|-------------|-----------------|---------------------|
      | 1  | SIMPLE               | NULL        | NULL            | Impossible WHERE    |
     -|----|----------------------|-------------|-----------------|---------------------|

     -- e.emp_no는 프라이머리 키이면서 NOT NULL이기 떄문에 e.emp_no IS NULL; 조건 만족하는 것이 없다.
     -- 불가능한 WHERE 조건
     ```

  8. LooseScan : 세미 조인 최적화 중에 LooseScan 최적화 전략 사용 시

     ```sql
     mysql>  EXPLAIN
     				SELECT *
     				FROM departments d
     				WHERE d.dept_no IN (SELECT de.dept_no FROM dept_emp de);

     -|----|-------------|-----------------|--------------|--------|---------------------------|
      | id | table       | type            | key          | rows   | Extra                     |
     -|----|-------------|-----------------|--------------|--------|---------------------------|
      | 1  | de          | index           | PRIMARY      | 331143 | Using index; LooseScan    |
      | 2  | d           | eq_ref          | PRIMARY      | 1      | NULL                      |
     -|----|-------------|-----------------|--------------|--------|---------------------------|
     ```

  9. No matching min/max row : MIN()이나 MAX() 같은 집합 함수가 있는 쿼리 조건절에 일치하는 코드가 한 건도 없을 때 사용, MIN()이나 MAX() 결과로 NULL 반환

     ```sql
     mysql>  EXPLAIN
     				SELECT MIN(dept_no), MAX(dept_no)
     				FROM dept_emp WHERE dept_no='';

     -|----|----------------------|-------------|-------|----------|-------------------------------------------------|
      | id | select_type          | table       | type  | key      | Extra                                           |
     -|----|----------------------|-------------|-------|----------|-------------------------------------------------|
      | 1  | SIMPLE               | NULL        | NULL  | NULL     | No matching min/max row                         |
     -|----|----------------------|-------------|-------|----------|-------------------------------------------------|

     ```

  10. no matching row in const table : const 접근법 시 일치하는 레코드 없은 경우(이것도 Impossible WHERE와 같은 종류)

      ```sql
      mysql>  EXPLAIN
      				SELECT *
      				FROM dept_emp de,
      					(SELECT emp_no FROM employees WHERE emp_no=0) tb1
      				WHERE tb1.emp_no=de.emp_no AND de.dept_no='d005'

      -|----|----------------------|-------------|-------|----------|-------------------------------------------------|
       | id | select_type          | table       | type  | key      | Extra                                           |
      -|----|----------------------|-------------|-------|----------|-------------------------------------------------|
       | 1  | SIMPLE               | NULL        | NULL  | NULL     | no matching row in const table                  |
      -|----|----------------------|-------------|-------|----------|-------------------------------------------------|

      ```

  11. No matching rows after partition pruning : 파티션 된 테이블에 대한 UPDATE 또는 DELETE 명령의 실행 계획에서 표시될 수 있는데 해당 파티션에서 UPDATE하거나 DELETE할 대상 레코드가 없는 경우 표시된다.

      ```sql
      mysql>  CREATE TABLE employees_parted (
      						emp_no int NOT NULL,
      						birth_date DATE NOT NULL,
      						first_name VARCHAR(4) NOT NULL,
      						last_name VARCHAR(4) NOT NULL,
      						gender ENUM('M', 'F') NOT NULL,
      						hire_date DATE NOT NULL,
      						PRIMARY KEY (emp_no, hire_date)
      				) PARTITION BY RANGE COLUMNS(hire_date)
      				( PARTITION p1986_1990 VALUES LESS THAN ('1990-01-01'),
      					PARTITION p1991_1995 VALUES LESS THAN ('1995-01-01'),
      					PARTITION p1996_2000 VALUES LESS THAN ('2000-01-01'),
      					PARTITION p2001_2005 VALUES LESS THAN ('2006-01-01'));

      mysql>  INSET INTO employees_parted SELECT * FROM employees;

      mysql>  SELECT MAX(hire_date) FROM employees_parted
      -|------------------|
       | MAX(hire_date)   |
      -|------------------|
       | 2000-01-28       |
      -|------------------|

      mysql>  EXPLAIN DELETE FROM employees_parted WHERE hire_date >= '2020-01-01'

      -|----|----------------------|-------------|--------------------------|---------|------------------------------------------|
       | id | select_type          | table       | partitions               | type    | Extra                                    |
      -|----|----------------------|-------------|--------------------------|---------|------------------------------------------|
       | 1  | DELETE               | NULL        | NULL                     | NULL    | No matching rows after partition pruning |
      -|----|----------------------|-------------|--------------------------|---------|------------------------------------------|

      -- DELETE 문장의 실행 계획에서 partitions 컬럼이 비어 있다. 단순히 삭제할 레코드 없음 의미가 아니라 대상 파티션이 없다는 것이다.

      mysql>  EXPLAIN DELETE FROM employees_parted WHERE hire_date < '1990-01-01'

      -|----|----------------------|------------------|--------------------------|---------|------------------------------------------|
       | id | select_type          | table            | partitions               | type    | Extra                                    |
      -|----|----------------------|------------------|--------------------------|---------|------------------------------------------|
       | 1  | DELETE               | employees_parted | p1986_1990               | ALL     | Using where                              |
      -|----|----------------------|------------------|--------------------------|---------|------------------------------------------|

      -- 위의 예제는 삭제할 레코드는 없지만 대상 파티션이 존재해 partitions 칼럼이 비어 있지 않고 Extra 칼럼에도 해당 메시지 표시 X
      ```

  12. No tables used : From 절 없는 문장이나 From DUAL 형태 쿼리 실행 계획에서 발생

      ```sql
      mysql>  EXPLAIN SELECT 1;
      mysql>  EXPLAIN SELECT 1 FROM dual;

      -|----|----------------------|-------------|-------|----------|------------------|
       | id | select_type          | table       | type  | key      | Extra            |
      -|----|----------------------|-------------|-------|----------|------------------|
       | 1  | SIMPLE               | NULL        | NULL  | NULL     | No tables used   |
      -|----|----------------------|-------------|-------|----------|------------------|

      ```

  13. Not exists

      - 안티 조인 : A 테이블에는 존재하지만 B 테이블에는 존재하지 않는 값 조회해야하는 쿼리 사용 → NOT IN(서브쿼리) 혹은 NOT EXISTS 연산자 사용
      - 똑같은 처리를 아우터 조인으로 이용해서도 구현 가능 → 레코드 건수가 만을 때는 안티 조인보다 아우터 조인 이용하면 더 빠른 성능 낼 수 있다.

      ```sql
      mysql>  EXPLAIN
      				SELECT *
      				FROM dept_emp de
      					LEFT JOIN departments d ON de.dept_no=d.dept_no
      				WHERE d.dept_no IS NULL

      -|----|----------------------|-------------|--------|----------|-------------------------|
       | id | select_type          | table       | type   | key      | Extra                   |
      -|----|----------------------|-------------|--------|----------|-------------------------|
       | 1  | SIMPLE               | de          | ALL    | NULL     | NULL                    |
       | 1  | SIMPLE               | d           | eq_ref | PRIMARY  | Using where; Not exists |
      -|----|----------------------|-------------|--------|----------|-------------------------|

      ```

  14. Plan isn’y ready yet : 해당 커넥션에서 아직 쿼리의 실행 계획을 수립 못한 상태에서 Explain For Connection 명령이 실행된 것 의미 → 대상 커넥션의 쿼리 실행 계획 수립할 여유 시간 주고 다시 Explain For Connection 실행
  15. Range check for each record(index map:N) : 레코드마다 인덱스 레인지 스캔을 체크한다

      ```sql
      mysql>  EXPLAIN
      				SELECT *
      				FROM employees e1, employees e2
      				WHERE e2.emp_no >= e1.emp_no

      -|----|----------------------|-------------|--------|----------------------------------------------|
       | id | select_type          | table       | type   | Extra                                        |
      -|----|----------------------|-------------|--------|----------------------------------------------|
       | 1  | SIMPLE               | e1          | ALL    | NULL                                         |
       | 1  | SIMPLE               | e2          | ALL    | Range check for each record(index map : 0x1) |
      -|----|----------------------|-------------|--------|----------------------------------------------|

      -- 매 레코드 단위모다 1번 인덱스를 쓸지 풀 테이블 스캔을 할지 결정하면서 처리
      -- ALL값으로 표시되면 풀 테이블 스캔으로 처리된 것으로 해석
      -- Extra 칼럼에 Range check for each record(index map:N) 표시되며 동시에 type에는 ALL로 표시 -> 후보 인덱스가 별로 도움되지 않아 최종적으로 풀 스캔을 실행
      ```

  16. Recursive : CTE를 이용해 재귀 쿼리를 작성 시 사용

      ```sql
      mysql>  WITH RECURSIVE cte (n) AS
      				(
      					SELECT 1
      					UNION ALL
      					SELECT n+1 FROM cte WHERE n < 5
      				)
      				SELECT * FROM cte;

      -|----|----------------------|-------------|--------|----------|-------------------------|
       | id | select_type          | table       | type   | key      | Extra                   |
      -|----|----------------------|-------------|--------|----------|-------------------------|
       | 1  | PRIMARY              | <derived2>  | ALL    | NULL     | NULL                    |
       | 2  | DERIVED              | d           | NULL   | NULL     | No tables used          |
       | 3  | UNION                | cte         | ALL    | NULL     | Recursive; Using where  |
      -|----|----------------------|-------------|--------|----------|-------------------------|
      ```

  17. Rematerialize : LATERAL 조인되는 테이블은 선행 테이블의 레코드별로 서브 쿼리 실행해 그 결과를 임시 테이블에 저장하는 과정

      ```sql
      mysql>  EXPLAIN
      				SELECT * FROM employees e
      					LEFT JOIN LATERAL ( SELECT *
      															FROM salaries s
      															WHERE s.emp_no=e.emp_no
      															ORDER BY s.from_date DESC LIMIT 2 ) s2 ON s2.emp_no=e.emp_no
      				WHERE e.first_name='Matt';

      -|----|----------------------|-------------|--------|--------------|----------------------------|
       | id | select_type          | table       | type   | key          | Extra                      |
      -|----|----------------------|-------------|--------|--------------|----------------------------|
       | 1  | PRIMARY              | e           | ref    | ix_firstname | Rematerialize (<derived2>) |
       | 1  | PRIMARY              | <derived2>  | ref    | <auto_key0>  | NULL                       |
       | 3  | DEPENDENT DERIVED    | s           | ref    | PRIMARY      | Using filesort             |
      -|----|----------------------|-------------|--------|--------------|----------------------------|

      -- employeees 테이블 레코드마다 salaries 테이블에서 emp_no가 일치하는 레코드 중에서 from_date 칼럼의 역순으로 2건만 가져와 임시 테이블 'derived2'에 저장
      -- employees와 derived2 테이블 조인한다.
      -- 이때 derived2 임시 테이블은 employees 테이블의 레코드마다 새로운 임시 테이블 생성 -> 이렇게 매번 임시 테이블 생성 시 Extra 컬럼에 Rematerialize 표시
      ```

  18. Select tables optimized away : MIN()또는 MAX()만 SELECT 절에 사용되거나 GROUP BY로 MIN(), MAX() 조회하는 쿼리가 인덱스를 오름차순 또는 내림차순으로 1건만 읽는 형태의 최적화 적용 시 사용

      ```sql
      mysql>  EXPLAIN
      				SELECT MAX(emp_no), MIN(emp_no) FROM employees;

      mysql>  EXPLAIN
      				SELECT MAX(from_date), MIN(from_date) FROM salaries WHERE emp_no=10002;

      -|----|----------------------|-------------|--------|--------------|------------------------------|
       | id | select_type          | table       | type   | key          | Extra                        |
      -|----|----------------------|-------------|--------|--------------|------------------------------|
       | 1  | SIMPLE               | NULL        | NULL   | NULL         | Select tables optimized away |
      -|----|----------------------|-------------|--------|--------------|------------------------------|

      -- 첫 번째 쿼리는 employees 테이블 있는 emp_no 컬럼에 인덱스 생성되어 있어 "Select tables optimized away" 최적화 가능
      -- 두 번째 쿼리 경우 salaries 테이블에 (emp_no, from_date)로 인덱스 생성되어 있으므로 인덱스가 emp_no=10002 레코드 검색 후 결과 중 오름차순 또는 내림차순으로 하나만 조회하기에 이러한 최적화 가능
      ```

  19. Start temporary, End temporary : 세미 조인 최적화 중 Duplicate Weed-out 최적화 전략 사용 시 Extra 컬럼에 명시
  20. unique row now found : 두 테이블이 각각 유니크(프라이머리 키 포함) 컬럼으로 아우터 조인 수행 쿼리에서 아우터 테이블에 일치하는 레코드 존재하지 않을 시 Extra 컬럼에 명시
  21. Using filesort : 적절한 인덱스로 ORDER BY 처리 못하는 경우 서버는 조회된 레코드 다시 정렬 → 즉 ORDER BY를 인덱스를 사용하지 못한 경우에 Extra 컬럼에 명시되며 조회 레코드를 정렬용 메모리 버퍼에 복사 후 퀵 소트 또는 힙 소트 알고리즘으로 정렬
  22. Using index(커버릴 인덱스) : 데이터 파일 전혀 읽지 않고 인덱스만 읽어 쿼리 모두 처리 가능할 때 Extra 컬럼에 명시
  23. Using index condition : 옵티마이저가 인덱스 컨데션 푸시 다운 최적화 사용 시 Extra 컬럼에 명시
  24. Using index for group-by : Group By(고부하 작업) 처리에 인덱스를 이용할 때 Extra 컬럼에 명시, 이를 루스 인덱스 스캔이라고 부른다.
      1. 타이트 인덱스 스캔(인덱스 스캔)을 통한 Group By 처리 : AVG(), SUM() 등을 위해서는 인덱스로 GROUP BY 처리 가능해도 필요 레코드만 읽을 수 없다. GROUP BY를 위해 인덱스 사용하지만 루스 인덱스 스캔이라고 하지 않는다.
      2. 루스 인덱스 스캔을 통한 GROUP BY 처리 : 단일 컬럼으로 구성된 인덱스에서 그루핑 컬럼 말고 아무것도 조회하지 않는 쿼리에서 루스 인덱스 스캔 사용 가능 그리고 다중 컬럼으로 만들어지 인덱스에서 GROUP BY 절이 인덱스 사용함은 물론이고 MIN(), MAX() 등에서 ‘루스 인덱스 스캔’ 사용될 수있다. → 이때 인덱스를 필요 부분만 읽는다
  25. Using index for skip scan : 옵티마이저가 인덱스 스킵 스캔 최적화 사용 시 Extra 컬럼에 명시

      ```sql
      mysql>  ALTER TABLE employees
      					ADD INDEX ix_gender_birthdate (gender, birthdate);

      mysql>  EXPLAIN
      				SELECT gender, birthdate
      				FROM employees
      				WHERE birthdate >= '1965-02-01'

      -|----|-------------|--------|---------------------|----------------------------------------|
       | id | table       | type   | key                 | Extra                                  |
      -|----|-------------|--------|---------------------|----------------------------------------|
       | 1  | employees   | range  | ix_gender_birthdate | Using where; Using index for skip scan |
      -|----|-------------|--------|---------------------|----------------------------------------|

      -- 첫 번째 쿼리는 employees 테이블 있는 emp_no 컬럼에 인덱스 생성되어 있어 "Select tables optimized away" 최적화 가능
      -- 두 번째 쿼리 경우 salaries 테이블에 (emp_no, from_date)로 인덱스 생성되어 있으므로 인덱스가 emp_no=10002 레코드 검색 후 결과 중 오름차순 또는 내림차순으로 하나만 조회하기에 이러한 최적화 가능
      ```

  26. Using join buffer(Block Nested Loop), Using join buffer(Batched Key Access), Using join buffer(hash join) : 조인 시 드리븐 테이블에 적절한 인덱스 없으면 MySQL 서버는 Block Nested Loop Join 이나 Hash Join 사용 → MySQL 서버는 조인 버퍼 사용 → 즉 실행 계획에서 조인 버퍼가 사용되는 실행 계획의 Extra 컬럼에는 “Using join buffer” 명시

      ```sql
      mysql>  EXPLAIN
      				SELECT *
      				FROM dept_emp de, employees e
      				WHERE de.from_date>'2005-01-01' AND e.emp_no<10904;

      -|----|----------------------|-------------|--------|--------------|-------------------------------------------|
       | id | select_type          | table       | type   | key          | Extra                                     |
      -|----|----------------------|-------------|--------|--------------|-------------------------------------------|
       | 1  | SIMPLE               | de          | range  | ix_fromdate  | Using index condition                     |
       | 1  | SIMPLE               | e           | range  | PRIMARY      | Using where; Using join buffer(hash join) |
      -|----|----------------------|-------------|--------|--------------|-------------------------------------------|

      -- Extra 컬럼에 "Using join buffer" 문구 표시, 이는 쿼리가 조인을 수행하기 위해 조인 버퍼 사용
      -- 그 뒤에 "hash join" 문구는 조인 버퍼 활용해 해시 조인으로 처리
      ```

  27. Using MRR

      - MySQL 인젠은 실행 계획 수립 후 스토리지 엔진의 API 호출해 쿼리 처리
      - InnoDB 포함한 스토리지 엔진은 쿼리 실행의 전체 부분 알지 못해 최적화 한계 존재
      - 많은 레코드 읽는 과정이여도 스토리지 엔진은 MySQL 엔진 넘겨주는 키 값 기준으로 레코드 한건씩 읽어서 반환하는 방식으로 밖에 작동하지 못하는 한계 존재 → MySQL 서버에서 단점 보완을 위해 MRR(Mutli Range Read) 최적화 도입
      - 작동 원리 : MySQL 엔진은 여러 키 값을 한 번에 스토리지 엔진에 전달 후 키 값 정렬해 최소한 페이지 접근만으로 필요 레코드 읽을 수 있게 최적화

      ```sql
      mysql>  EXPLAIN
      				SELECT /*+ JOIN_ORDER(s, e) */ *
      				FROM employees e, salaries s
      				WHERE e.first_name='Matt'
      					AND e.hire_date BETWEEN '1990-01-01' AND '1991-01-01'
      					AND s.emp_no=e.emp_no
      					AND s.from_date BETWEEN '1990-01-01' AND '1991-01-01'
      					AND s.salary BETWEEN 50000 AND 60000;

      -|----|-------------|-----------------|--------------|-----------|------|----------------------------------|
       | id | table       | type            | key          | key_len   | rows | Extra                            |
      -|----|-------------|-----------------|--------------|-----------|------|----------------------------------|
       | 1  | s           | range           | ix_salary    | 4         | 3314 | Using index condition; Using MRR |
       | 1  | e           | eq_ref          | PRIMARY      | 4         | 1    | Using where                      |
      -|----|-------------|-----------------|--------------|-----------|------|----------------------------------|

      -- 위 실행 계획은 salaries 테이블 읽은 레코드로 employees 테이블 검색 위한 조인 키 값 모아서 MRR 엔진으로 전달
      -- MRR 엔진은 키 값 정렬 후 employees 테이블 최적화 방법으로 접근했음을 의미
      ```

  28. Using sort_union(…), Using union(…), Using intersect(…) : 쿼리가 index_merge 접근법으로 실행 경우 2개의 인덱스가 동시에 사용 가능 → 실행 계획 Extra 칼럼에 두 인덱스로부터 읽은 결과 어떻게 병합했는지 상세히 설명위해 3개 중 하나의 메시지 선택 출력
  29. Using intersect(…) : 각각 인덱스 사용할 수 있는 조건이 AND로 연결된 경우 각 처리 결과에서 교집합 추출하는 작업 수행
  30. Using union(…) : 각각 인덱스 사용할 수 있는 조건이 OR로 연결된 경우 각 처리 결과에서 합집합 추출하는 작업 수행
  31. Using sort_union(…) : Using union과 같은 방식으로 수행되지만 Using union으로 처리할 수 없는 경우( OR로 연결된 상대적으로 대량의 range 조건들) 이 방식으로 처리, 차이점은 Using sort_union(…)은 프라이머리 키만 먼저 읽어서 정렬 후 병합해 비로소 레코드 읽어서 반환
  32. Using temporary : 쿼리 처리하는 동안 중간 결과 담아 두기 위해 임시 테이블(Temporary table) 사용하는 경우 표시, 이때 사용된 임시테이블이 메모리에 생성되었는지 디스크에 생성되었는지는 실행 계획으로 알 수 없다.

      ```sql
      mysql>  EXPLAIN
      				SELECT *
      				FROM employees
      				GROUP BY gender
      				ORDER BY MIN(emp_no);

      -|----|----------------------|-------------|--------|---------------------------------|
       | id | select_type          | table       | type   | Extra                           |
      -|----|----------------------|-------------|--------|---------------------------------|
       | 1  | SIMPLE               | employees   | index  | Using temporary; Using filesort |
      -|----|----------------------|-------------|--------|---------------------------------|

      -- 위 쿼리는 GROUP BY 칼럼과 ORDER BY 칼럼이 다르기에 임시 테이블 필요
      -- 인덱스 사용 못하는 GROUP BY 쿼리는 실행 계획에 Using temporary 표시되는 대표적인 예
      ```

  33. Using where : MySQL 엔진 레이어에서 별도의 가공해서 필터리(여과) 작업을 처리한 경우에 Extra 컬럼에 명시
  34. Zero limit : MySQL 서버에서 데이터 값이 아닌 쿼리 결과값의 메타데이터만 필요한 경우 존재, 즉 쿼리의 결과가 몇 개의 컬럼 가지고 각 컬럼 타입이 무엇인지 등의 정보만 필요한 경우 존재 → 이 경우 쿼리의 마지막에 LIMIT 0 사용 → 옵티마이저는 사용자의 의도(메타 정보만 조회하는) 알아채고 테이블 레코드 전혀 읽지 않고 결과값의 메타 정보만 반환, 이 경우 Extra 컬럼에 Zero limit 명시
