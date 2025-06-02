# Real MySQL 9.4

# 9.4 쿼리 힌트

- Intro
  - 쿼리 힌트 : MySQL 서버의 부족한 실행 계획 즉 비즈니스 로직을 이해하는데 용이하기 위한 수단
  - 종류
    - 인덱스 힌트
    - 옵티마이저 힌트
- 인덱스 힌트

  - STRAIGHT_JOIN, USE_INDEX 등 존재 → 옵티마이저 힌트 도입되지 전의 기능
  - SELECT, UPDATE 명령에서만 사용
  - 가능하면 옵티마이저 힌트 써라
  - STRAIGHT_JOIN : 옵티마이저 힌트인 동시에 조인 키워드

    - SELECT, UPDATE, DELETE 쿼리에서 여러 개 테이블 조인 시 조인 순서 고정

      ```sql
      mysql>  EXPLAIN
      				SELECT *
      				FROM employees e, dept_emp de, departments d
      				WHERE e.emp_no=de.emp_no AND d.dept_no=de.dept_no;

      |-----|-------------|--------|--------|---------------|--------|---------------|
      |  id | select_type | table  | type   | key           | rows   | Extra         |
      |-----|-------------|--------|--------|---------------|--------|---------------|
      |  1  | SIMPLE      | d      | index  | ux_deptname   |   9    | Using index   |
      |  1  | SIMPLE      | de     | ref    | PRIMARY       | 41392  | NULL          |
      |  1  | SIMPLE      | e      | eq_ref | PRIMARY       |   1    | NULL          |
      |-----|-------------|--------|--------|---------------|--------|---------------|
      ```

    - 일반적으로 조인 위해서 칼럼들의 인덱스 여부로 조인 순서 결정, 조인 컬럼 인덱스에 문제 없을 시(WHERE 조건 있는 경우 WHERE 조건 만족하는) 레코드가 적은 테이블을 드라이빙으로 선택 → 여기는 departments가 레코드 수가 적어서 드라이빙으로 선정

      ```sql
      mysql>  SELECT STRAIGHT_JOIN
      					e.first_name, e.last_name, d.dept_name
      				FROM employees e, dept_emp de, departments d
      				WHERE e.emp_no=de.emp_no AND d.dept_no=de.dept_no;

      mysql>  SELECT /*! STRAIGHT_JOIN */
      					e.first_name, e.last_name, d.dept_name
      				FROM employees e, dept_emp de, departments d
      				WHERE e.emp_no=de.emp_no AND d.dept_no=de.dept_no;

      |-----|-------------|--------|--------|-------------------|--------|---------------|
      |  id | select_type | table  | type   | key               | rows   | Extra         |
      |-----|-------------|--------|--------|-------------------|--------|---------------|
      |  1  | SIMPLE      | e      | ALL    | NULL              | 300473 | NULL          |
      |  1  | SIMPLE      | de     | ref    | ix_empno_fromdate |   1    | Using index   |
      |  1  | SIMPLE      | d      | eq_ref | PRIMARY           |   1    | NULL          |
      |-----|-------------|--------|--------|-------------------|--------|---------------|
      ```

    - STRAIGHT_JOIN 통해 FROM 절에 명시도니 테이블 순서대로 조인 유도
    - STRAIGHT_JOIN 힌트로 조인 순서 조정하기 좋은 상황

      1. 임시 테이블(인라인 뷰 또는 파생 테이블)과 일반 테이블 조인 : 일반적으로 임시 테이블을 드라이빙 테이블로 선정하는게 좋다. 일반 테이블의 조인 컬럼에 인덱스가 없는 경우는 레코드 건수가 작은 쪽을 드라이빙을 선정 → 옵티마이저가 실행 계획 제대로 수립하지 못해 심각한 성능 저하 있는 경우 힌트 사용
      2. 임시 테이블끼리 조인 : 임시테이블은 항상 인덱스가 없기 때문에 크기가 작은 테이블을 드라이빙 테이블로 선택하는게 좋다.
      3. 일반 테이블끼리 조인 : 양쪽 모두 조인 컬럼에 인덱스 있거나 모두 없는 경우 레코드 건수가 적은 테이블을 드라이빙으로 선택, 그 이외에 조인 컬럼에 인덱스가 없는 테이블 드라이빙 테이블

         cf) 레코드 건수 : 테이블 전체 레코드 수가 아니라 인덱스 사용할 수 있는 WHERE 조건까지 포함해 그 조건 만족하는 레코드 건수

    - STRAIGHT_JOIN 과 비슷한 역할 : JOIN_FIXED_ORDER, JOIN_ORDER, JOIN_PREFIX, JOIN_SUFFIX
      - JOIN_FIXED_ORDER는 STRAIGHT_JOIN과 동일한 효과 (사용 시 FROM 절 모든 테이블에 대해서 조인 순서가 결정되는 효과)
      - JOIN_ORDER, JOIN_PREFIX, JOIN_SUFFIX 나머지 3개는 일부 테이블 조인 순서에 대해서만 제안하는 힌트

  - USE INDEX / FORCE INDEX / IGNORE INDEX

    - 3 ~ 4개 이상 컬럼 포함하는 비슷한 인덱스 여러 개 존재하는 경우 옵티마이저가 적절한 결정 내리지 못함 즉 실수함 → 강제로 특정 인덱스 사용하도록 힌트 추가
      1. USE INDEX : 가장 자주 사용되는 인덱스 힌트, MySQL 옵티마이저에게 특정 테이블 인덱스 사용하도록 권장하는 힌트(인덱스 힌트 주어지면 대부분 옵티마이저가 채택하지만 항상 그런 것은 또 아님)
      2. FORCE INDEX : USE INDEX보다 옵티마이저에게 미치는 영향이 더 강함
      3. IGNORE INDEX : 특정 인덱스 사용하지 못하게 하는 용도 → 옵티마이저가 풀 테이블 스캔을 사용하도록 유도하기 위해 사용
    - 위의 3종류 인덱스 힌트 모두 용도 명시 가능, 특별히 인덱스에 용도 명시되지 않으면(사용 가능한 경우) 주어진 인덱스를 3가지 용도로 사용

      1. USE INDEX FOR JOIN : JOIN은 테이블 간 조인뿐 아니라 레코드 검색하기 위한 용도까지 포함
      2. USE INDEX FOR ORDER BY : 명시된 인덱스를 ORDER BY 용도로만 사용하게 제한
      3. USE INDEX FOR GROUP BY : 명시된 인덱스를 GROUP BY 용도로만 사용하게 제한

      ```sql
      mysql>  SELECT * FROM employees WHERE emp_no = 10001;
      mysql>  SELECT * FROM employees FORCE INDEX(primary) WHERE emp_no = 10001;
      mysql>  SELECT * FROM employees USE INDEX(primary) WHERE emp_no = 10001;

      mysql>  SELECT * FROM employees IGNORE INDEX(primary) WHERE emp_no = 10001;
      mysql>  SELECT * FROM employees FORCE INDEX(ix_firstname) WHERE emp_no = 10001;

      -- 첫 번째부터 세 번째 쿼리는 모두 primary 키로 동일한 실행 계획으로 쿼리 실행
      -- 기본적을 인덱스 주어지지 않아도 emp_no = 10001 조건으로 프라이머리 키 사용하는게 최적이라는 것은 옵티마이저도 인식
      -- 네 번쨰는 프라이머리 키 인덱스 사용 못하게 힌트 사용 -> 풀 테이블 스캔 유도
      -- 다섯 번째 또한 프라이머리 키 인덱스가 아닌 전혀 관계 없는 인덱스 사용하도록 유도했더니 풀 테이블 스캔 실행
      ```

    - 전문 검색 (Full Text Search) 인덱스 있는 경우 다른 일반 보조 인덱스(B-Tree 인덱스) 사용할 수 있는 상황이여도 전문 검색 인덱스 선택하는 경우 많다.(전문 검색 인덱스와 같은 인덱스는 선택 시 가중치를 둔다)

  - SQL_CALC_FOUND_ROWS

    - 개요 : MySQL은 LIMIT 사용하는 경우 조건 만족 레코드가 LIMIT보다 많아도 명시된 LIMIT만큼 찾으면 검색 작업 멈춘다.
    - 특징 : SQL_CALC_FOUND_ROWS는 LIMIT 만족한만큼 찾아도 끝까지 검색 → 최종적으로 LIMIT만큼만 반환

      ```sql
      mysql>  SELECT SQL_CALC_FOUND_ROWS * FROM employees LIMIT 5;

      |--------|-------------|-------------|-------------|---------|-------------|
      | emp_no | birth_date  | first_name  | last_name   | gender  | hire_date   |
      |--------|-------------|-------------|-------------|---------|-------------|
      | 10001  | 1953-09-02  | Georgi      | Facello     | M       | 1986-06-26  |
      | 10002  | 1964-06-02  | Bezalei     | Simmel      | F       | 1985-11-02  |
      |--------|-------------|-------------|-------------|---------|-------------|

      mysql>  SELECT FOUND_ROWS() AS total_record_count;
      |--------------------|
      | total_record_count |
      |--------------------|
      |   300024           |
      |--------------------|
      ```

      ```sql
      mysql>  SELECT SQL_CALC_FOUND_ROWS * FROM employees WHERE first_name='Georgi' LIMIT 0, 20;
      mysql>  SELECT FOUND_ROWS() AS total_record_count;

      -- 한 번의 쿼리 실행으로 필요 정보 2가지 모두 가져오는 것으로 보이지만 FOUND_ROWS() 함수 실행 위해 또 한 번의 쿼리 필요 즉 쿼리 2번 실행
      -- first_name='Georgi 조건 처리 위해 employees의 ix_firstname 인덱스 레인지 스캔으로 값을 읽는다. 만족하는 레코드는 촏 253건이다. LIMIT 조건으로 처음 20건만 가져오지만 SQL_CALC_FOUND_ROWS 힌트 떄문에 조건 만족하는거 전부 읽어야된다.
      -- ix_firstname 인덱스 통해 실제 데이터 레코드 찾아가는 작업 253번 실행 -> 랜덤 I/O가 253번 발생

      mysql>  SELECT COUNT(*) FROM employees WHERE first_name='Georgi';
      mysql>  SELECT * FROM employees WHERE first_name='Georgi' LIMIT 0, 20;

      -- 이 방식도 쿼리 2번 실행
      -- 첫 번째 쿼리 실행 시 ix_firstname으로 인덱스 레인지 스캔, 커버링 인덱스 떄문에 랜덤 I/O 발생하지 않는다. 실제 레코드 데이터 필요한게 아닌 건수만 가져오면 되기 떄문
      -- ix_firstname으로 레인지 스캔으로 접근한 후 실제로 데이터 레코드 읽어야되므로 랜덤 I/O 발생 -> 그러나 LIMIT 제한으로 250번이 아닌 20번만 실행
      ```

    - SQL_CALC_FOUND_ROWS 사용하는 경우가 느린 것은 쉽게 알 수 있다. → 힌트 사용해도 정확한 레코드 건수를 가져올 수 없다. → SQL_CALC_FOUND_ROWS 는 성능 향상을 위한 힌트가 아닌 개발자의 편의를 위해 만들어진 힌트이기 때문이다.

- 옵티마이저 힌트

  - 종류(영향 범위 따라)
    1. 인덱스 : 특정 인덱스의 이름을 사용할 수 있는 옵티마이저 힌트
    2. 테이블 : 특정 테이블의 이름을 사용할 수 있는 옵티마이저 힌트
    3. 쿼리 블록 : 특정 쿼리 블록에 사용할 수 있는 옵티마이저 힌트(명시된 쿼리 블록에 대해서만 영향을 미침)
    4. 글로벌(쿼리 전체) : 전체 쿼리에 대해 영향을 미치는 힌트
  - SET_VAR
    - 개요 : MySQL 시스템 변수들도 쿼리 실행 계획에 영향을 미침
    - 용도 : 이렇게 MySQL 시스템 변수를 조정하여 쿼리 성능을 향상시키는 용도로 활용 가능(모든 시스템 변수를 조정할 수 있는 것은 아님)
  - SEMIJOIN & NO_SEMIJOIN

    - 개요 : SEMIJOIN(세미조인)은 **두 테이블 간의 조인 방식 중 하나로**, **조인된 테이블의 컬럼 값을 결과에 포함하지 않고, 존재 여부만 확인**하는 조인 → **불필요한 조인을 제거**할 수 있어서 **성능 최적화**에 유리

      ```sql
      SELECT a.*
      FROM employees a
      WHERE EXISTS (
          SELECT 1 FROM departments b WHERE b.id = a.dept_id
      );

      -- departments 테이블에 해당 부서가 존재하는지만 확인.
      -- 실제로 departments의 컬럼은 조회하지 않음.

      ```

    - 용도

      - SEMIJOIN (세미조인)

        - **두 테이블을 조인할 때, 상대 테이블의 컬럼은 결과에 포함하지 않고, 존재 여부만 판단하는 조인 방식**
        - MySQL에서는 IN, EXISTS 와 같은 **서브쿼리**를 실행할 때 **자동으로 세미조인 변환**을 시도함

          ```sql
          SELECT * FROM employees e
          WHERE EXISTS (
              SELECT 1 FROM departments d WHERE d.id = e.dept_id
          );

          -- 옵티마이저는 이 EXISTS 조건을 세미조인으로 바꾸어 실행할 수 있음
          ```

      - NO_SEMIJOIN
        - **세미조인 최적화를 강제로 비활성화**함
        - 옵티마이저가 IN, EXISTS 구문을 세미조인으로 변환하지 않도록 제어

    - 사용 목적
      | 항목 | 내용 |
      | --------------- | -------------------------------------------------------------------------------------------------------------------------------- |
      | 성능 최적화 | 세미조인은 일반 조인보다 적은 양의 데이터를 처리하므로 성능이 개선될 수 있음 |
      | 옵티마이저 제어 | 상황에 따라 세미조인이 오히려 비효율적일 수 있으므로, 힌트를 통해 옵티마이저의 선택을 제어 가능(서브쿼리 결과가 너무 작을 때 등) |
    - 예시

      ```sql
      SELECT /*+ SEMIJOIN(@subq1) */ *
      FROM employees e
      WHERE EXISTS (
          SELECT /*+ QB_NAME(subq1) */ 1 FROM departments d WHERE d.id = e.dept_id
      );

      SELECT /*+ NO_SEMIJOIN(@subq1) */ *
      FROM employees e
      WHERE EXISTS (
          SELECT /*+ QB_NAME(subq1) */ 1 FROM departments d WHERE d.id = e.dept_id
      );

      ```

  - JOIN_FIXED_ORDER & JOIN_ORDER & JOIN_PREFIX & JOIN_SUFFIX
    - 개요 : STRAIGHT_JOIN 힌트를 통해 조인 순서 결정했으나 FROM 절에 사용된 테이블 순서를 조인 순서에 맞춘다는 번거러움 존재 & STRAIGHT_JOIN은 한 번 사용 후 FROM 절에 명시된 모든 테이블 순서 결정되기 때문에 일부는 조인 순서를 강제하고 나머지는 옵티마이저에게 순서 결정하게 맞기는 것 불가능 → 이를 보완하는 목적
    - 종류
      1. JOIN_FIXED_ORDER : STRAIGHT_JOIN 힌트와 동일하게 FROM 절 테이블 순서대로 조인 실행하는 힌트
      2. JOIN_ORDER : FROM 절에 사용된 테이블 순서가 아니라 힌트에 명시된 테이블 순서대로 조인 실행하는 힌트
      3. JOIN_PREFIX : 조인에서 드라이빙 테이블만 강제하는 힌트
      4. JOIN_SUFFIX : 조인에서 드리븐 테이블(가장 마지막에 조인돼야 할 테이블들)만 강제하는 힌트
  - MERGE & NO_MERGE

    - 개요 : MySQL 옵티마이저가 내부 쿼리를 외부 쿼리와 병합하는게 나을 수 있고 때로 내부 임시 테이블 생성하는게 더 나을 수 있기에 명시적으로 제어하기 위한 힌트 → 서브쿼리를 메인 쿼리에 병합(MERGE)할지 말지를 옵티마이저가 판단 명시적 제어

      ```sql
      -- 병합을 강제
      SELECT /*+ MERGE(@subq1) */ ...
      FROM (
          SELECT /*+ QB_NAME(subq1) */ ...
      ) AS derived_table;

      -- 병합을 방지
      SELECT /*+ NO_MERGE(@subq1) */ ...
      FROM (
          SELECT /*+ QB_NAME(subq1) */ ...
      ) AS derived_table;
      ```

  - INDEX_MERGE & NO_INDEX_MERGE

    - 개요 : 하나의 인덱스만으로 검색 대상 범위 좁힐 수 없는 경우 여러 인덱스를 통해 검색한 결과를 교집합 또는 합집합으로 결과 반환 → 하나의 테이블에 대해 여러 인덱스 동시 사용하는 것을 인덱스 머지(Index Merge)

      ```sql
      -- 인덱스 병합 전략을 강제 활성화
      SELECT /*+ INDEX_MERGE(user age_idx city_idx) */ *
      FROM user
      WHERE age = 30 AND city = 'Seoul';

      -- 옵티마이저가 여러 인덱스를 병합하지 못하게 함
      SELECT /*+ NO_INDEX_MERGE(user) */ *
      FROM user
      WHERE age = 30 AND city = 'Seoul';
      ```

  - NO_ICP

    - What is ICP : **인덱스 스캔 중** 조건절(WHERE)을 **조기에 적용하여**, **디스크에서 레코드를 읽기 전에 필터링하는 최적화 기법**.
    - 개요 : ICP 최적화는 사용 가능하면 항상 성능 향상에 도움된다. 그러나 ICP 인해 여러 실행 계획 비용이 잘못된다면 결과적으로 잘못된 실행 계획 수립
    - ICP로 인해 실행 계획 비용 오판 되는 경우

      1. 조건절이 인덱스를 사용할 수 있으나, 매우 비선택적인(많은 row가 매칭되는) 조건일 경우
      2. 옵티마이저는 "조건을 인덱스에서 필터링하니 빠르겠지?"라고 **과대평가**
      3. 실제로는 **디스크에서 많은 레코드를 다시 가져와야 해서** 오히려 **Full Table Scan보다 느림**

      ```sql

      CREATE TABLE users (
          id INT PRIMARY KEY,
          gender CHAR(1),
          age INT,
          city VARCHAR(50),
          INDEX idx_gender_age (gender, age)
      );

      -- 성별은 'M', 'F' 뿐이라서 매우 비선택적 (매칭 건수 많음)
      -- age는 조건 없음 (range 없음)

      -- 쿼리
      EXPLAIN SELECT * FROM users WHERE gender = 'M';

      -- 옵티마이저 선택: idx_gender_age 인덱스 + ICP
      -- 예상: gender = 'M' 조건이 인덱스에서 걸러지므로 효율적일 것
      -- 실제: gender = 'M'인 row가 너무 많아 대량의 레코드 fetch 발생
      -- Full Table Scan이 더 나았을 수도 있음
      ```

  - SKIP_SCAN & NO_SKIP_SCAN
    - 개요 : 인덱스 스킵 스캔은 선행 칼럼에 대한 조건 없어도 옵티마이저가 해당 인덱스 사용할 수 있게 하는 최적화 기능 but 조건 누락된 선행 컬럼이 가지는 유니크 값이 많다면 성능이 떨어진다.
    - NO_SKIP_SCAN을 통해 유니크 값 갯수 제대로 분석못하거나 잘못도니 경로로 비효율적 인덱스 스킵 스캔 방
