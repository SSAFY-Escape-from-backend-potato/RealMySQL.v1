## 10.3

- 실행계획은 TREE 혹은 JSON으로 볼 수 있음
- 실행 계획에서 위쪽에 출력된 결과일수록(id 칼럼의 값이 작을수록 쿼리의 바깥(Outer) 부분이거나 먼저 접근한 테이블이고, 아래쪽에 출력된 결과일수록(id 칼럼의 값이 클수록 쿼리의 안쪽(Inner)부분 또는 나중에 접근한 테이블에 해당함

## 10.3.1 id 칼럼

 id 칼럼은 단위 SELECT 쿼리별로 부여되는 식별자 값

 id 칼럼이 테이블의 접근 순서를 의미하지는 않는다

 EXPLAIN FORMAT=TREE 명령으로 확인해보면 순서를 더 정확히 알 수 있다.

## 10.3.2 select_type 칼럼

SELECT 쿼리가 어떤 타입의 쿼리인지 표시되는 칼럼

SIMPLE: UNION이나 서브쿼리를 사용하지 않는 단순한 SELECT 쿼리. 하나만 존재하며, 일반적으로 제일 바깥 SELECT 쿼리

PRIMARY: SELECT 쿼리의 실행 계획에서 가장 바깥쪽(Outer)에 있는 단위 쿼리. 하나만 존재함

UNION: UNION으로 결합하는 단위 SELECT 쿼리, 첫 번째를 제외한 두 번째 이후부터는 UNION이고 첫 번쨰는 DERIVED(결과를 저장하는 임시 테이블)

DEPENDENT UNION:  내부 쿼리가 외부의 값을 참조해서 처리될 때 select_type에 DEPENDENT 키워드가 표시된다.

UNION RESULT: UNION 결과를 담아두는 테이.  UNION ALL을 사용하면 MySQL 서버는 임시 테이블에 버퍼링하지 않기 때문에 생성안됨

SUBQUERY: FROM 절에 사용된 서브쿼리는 select_type DERIVED로 표시되고, 그 밖의 위치에서 사용된 서브쿼리는 전부 SUBQUERY라고 표시된다. 종류로는 아래와 같다.

- 중첩된 쿼리(Nested Query): SELECT되는 칼럼에 사용된 서브쿼리를 네스티드 쿼리라고 한다.
- 서브쿼리(Subquery): WHERE 절에 사용된 경우에는 일반적으로 그냥 서브쿼리라고 한다.
- 파생 테이블(Derived Table): FROM 절에 사용된 서브쿼리를 MySQL에서는 파생 테이블이라고 하며, 일반적으로 RDBMS에서는 인라인 뷰(Inline View) 또는 서브 셀렉트(Sub Select)라고 부른다.
- 스칼라 서브쿼리(Scalar Subquery): 하나의 값만(칼럼이 단 하나인 레코드 1건만) 반환하는 쿼리
- 로우 서브쿼리(Row Subquery): 칼럼의 개수와 관계없이 하나의 레코드만 반환하는 쿼리

DEPENDENT SUBQUERY: 서브쿼리가 바깥쪽(Outer) SELECT 쿼리에서 정의된 칼럼을 사용하는 경우

DERIVED: SELECT 쿼리의 실행 결과로 메모리나 디스크에 임시 테이블을 생성하는 것을 의미함 (+DERIVED인 것이 있는지 확인 후, 서브쿼리를 조인으로 해결할 수 있는 경우라면 서브쿼리보다는 조인을 사용할 것을 강력히 권장)

DEPENDENT DERIVED: 해당 테이블이 래터럴 조인으로 사용된 것

UNCACHEABLE SUBQUERY: SUBQUERY중 서브쿼리에 포함된 요소에 의해 캐시 자체가 불가능 할 때

UNCACHEABLE UNION: UNION중 요소에 의해 캐시 자체가 불가능 할 때

MATERIALIZED: DERIVED와 비슷하게 쿼리의 내용을 임시 테이블로 생성한다

## 10.3.3 table 칼럼

MySQL 서버의 실행 계획은 단위 SELECT 쿼리 기준이 아니라 테이블 기준으로 표시된다.

별도의 테이블을 사용하지 않는 SELECT 쿼리인 경우에는 table 칼럼에 NULL이 표시

파생된 테이블은 derived로 표시

## 10.3.4 partitions 칼럼

접근 한 파티션을 보여줌.

파티션이 여러 개인 테이블에서 불필요한 파티션을 빼고 쿼리를 수행하기 위해 접근해야 할 것으로 판단되는테이블만 골라내는 과정을 파티션 프루닝(Partition pruning)이라고 한다.

이를 위해 실행 계획을 통해서 어느 파티션을 읽는지 확인할 수 있어야 한다. 

## 10.3.5 type 칼럼

테이블의 레코드를 어떤 방식으로 읽었는지를 나타낸다. 여기서 방식이라 함은 인덱스를 사용해 레코드를 읽었는지, 아니면 테이블을 처음부터 끝까지 읽는 풀 테이블 스캔으로 레코드를 읽었는지 등을 의미

system: 레코드가 1건만 존재하는 테이블 또는 한 건도 존재하지 않는 테이블을 참조하는 형태의 접근 방법

const: 쿼리가 프라이머리 키나 유니크 키 칼럼을 이용하는 WHERE 조건절을
가지고 있으며, 반드시 1건을 반환하는 쿼리의 처리 방식. UNIQUE INDEX SCAN이라고도 표현

eq_ref: 여러 테이블이 조인되는 쿼리의 실행 계획에서만 표시된다. 조인에서 처음 읽은 테이블의 칼럼값을, 그다음 읽어야 할 테이블의 프라이머리 키나 유니크 키 칼럼의 검색 조건에 사용할 때를 가리켜 eq_ref라고 한다. 이때 두 번째 이후에 읽는 테이블의 type 칼럼에 eq_ref가 표시된다. 두 번째 이후에 읽는 테이블에서 반드시 1건만 존재한다는 보장이 있어야 사용할 수 있는 접근 방법이다.
ref: 조인의 순서와 인덱스의 종류에 관계없이 동등(Equal) 조건으로 검색(1건의 레코드만 반환된다는 보장이 없어도 됨)

fulltext: full-text Search 인덱스를 사용해 레코드를 읽는 접근 방법

ref_or_null:  ref 접근 방법과 같은데, NULL 비교가 추가된 형태다 (IS NULL)

unique_subquery: 서브쿼리에서 중복되지 않는유니크한 값만 반환할 때 이 접근 방법을 사용한다.

index_subquery: IN (subquery) 형태의 조건에서 subquery의 반환 값에 중복된 값이 있을 수 있지만 인덱스를 이용해 중복된 값을 제거할 수 있음

range: "<, >, IS NULL, BETWEEN, IN, LIKE" 등의 연산자를 이용해 인덱스를 검색할 때 사용된다

index_merge: 2개 이상의 인덱스를 이용해 각각의 검색 결과를 만들어낸 후, 그 결과를 병합해서 처리하는 방식이다

index: 인덱스를 처음부터 끝까지 읽는 인덱스 풀 스캔을 의미

ALL: 인덱스를 사용하지 않고, 테이블을 처음부터 끝까지 읽어서 레코드를 가져오는 풀 테이블 스캔

## 10.3.6 possible_keys 칼럼

옵티마이저가 최적의 실행 계획을 만들기 위해 후보로 선정했던 접근 방법에서 사용되는 인덱스의 목록

## 10.3.7 key 칼럼

- possible_keys 칼럼의 인덱스가 사용 후보였던 반면, key 칼럼에 표시되는 인덱스는 최종 선택된 실행
계획에서 사용하는 인덱스를 의미
- key 칼럼에 의도했던 인덱스가 표시되는지 확인하는 것이 중요

## 10.3.8 key_len 칼럼

- 인덱스의 각 레코드에서 몇 바이트까지 사용했는지 알려주는 값이다.
- 다중 칼럼 인덱스뿐 아니라 단일 칼럼으로 만들어진 인덱스에서도 같은 지표를 제공

## 10.3.9 ref 칼럼

- 접근 방법이 ref면 참조 조건(Equal 비교 조건)으로 어떤 값이 제공됐는지 보여준다.
- 상숫값을 지정했다면 ref 칼럼의 값은 const로 표시되고, 다른 테이블의 칼럼값이면 그 테이블명과 칼럼명이 표시
- ref 칼럼의 값이 func라고 표시될 때가 있다. 이는 "Function"의 줄임말로 참조용으로 사용되는 값을 그대로 사용한 것이 아니라 콜레이션 변환이나 값 자체의 연산을 거쳐서 참조됐다는 것을 의미

## 10.3.10 rows 칼럼

- 실행 계획의 효율성 판단을 위해 예측했던 레코드 건수를 보여준다.
- 각 스토리지 엔진별로 가지고 있는 통계 정보를 참조해 MySQL 옵티마이저가 산출해 낸 예상값이라서 정확하지는 않다.
- 얼마나 많은 레코드를 읽고 체크해야 하는지를 의미

## 10.3.11 filtered 칼럼

- 필터링되고 남은 레코드의 비율을 의미

## 10.3.12 Extra 칼럼

실행 계획에서 성능에 관련된 중요한 내용이 Extra 칼럼에 자주 표시된다.

const row not found: const 접근 방법으로 테이블을 읽었지만 실제로 해당 테이블에 레코드가 1건도 존재하지 않음

**Deleting all rows**: DELETE 문에서 WHERE 절이 없을 때 테이블 전체를 단 한 번의 핸들러 호출로 삭제하는 최적화

**Distinct**: DISTINCT 처리를 위해 조인 결과에서 중복 제거 작업이 수행됨

**FirstMatch**: 세미 조인 최적화 전략. 조인 대상 테이블에서 첫 번째 일치하는 레코드만 확인 후 검색 중단

**Full scan on NULL key**: IN 조건의 좌변 값이 NULL일 때, 서브쿼리 대상 테이블에 대해 풀 테이블 스캔이 발생할 수 있음

**Impossible HAVING**: HAVING 조건이 절대 참이 될 수 없는 경우 (예: NOT NULL 필드에 IS NULL 조건)

**LooseScan**: 세미 조인 최적화 전략. 인덱스를 듬성듬성 스캔하여 효율적으로 처리

**No matching min/max row**: MIN()/MAX() 조건에 일치하는 레코드가 없어서 결과가 NULL로 반환됨

**no matching row in const table**: const 접근 테이블에서 조건을 만족하는 레코드가 없음

**No matching rows after partition pruning**: 파티션 테이블에서 조건을 만족하는 파티션 자체가 없음

**No tables used**: FROM 절이 없거나 DUAL만 사용된 쿼리로, 테이블 접근이 없음

**Not exists**: 아우터 조인을 활용한 안티 조인 최적화. 조인 조건이 불일치하는 경우만 처리

**Plan isn't ready yet**: 실행 중인 커넥션에 대해 EXPLAIN을 수행했을 때 아직 실행 계획이 완전히 준비되지 않음

**Recursive**: WITH RECURSIVE를 사용한 재귀 CTE 쿼리에서 나타남

**Rematerialize**: LATERAL JOIN 시 선행 테이블의 각 레코드마다 서브쿼리를 실행해 임시 테이블 생성

**Select tables optimized away**: 인덱스를 활용한 MIN(), MAX(), COUNT(*) 쿼리에서 테이블 접근 자체를 생략

**Start temporary, End temporary**: Duplicate Weed-out 세미 조인 최적화. 중복 제거를 위해 임시 테이블 사용

**unique row not found**: OUTER JOIN 시 조인 대상 테이블에 유일한 레코드가 존재하지 않음

**Using filesort**: ORDER BY 정렬을 위해 인덱스를 사용하지 못하고 메모리 정렬 버퍼를 사용함

**Using index (커버링 인덱스)**: 필요한 칼럼이 모두 인덱스에 포함되어 있어 데이터 파일을 읽지 않고 처리

**Using index condition**: Index Condition Pushdown 최적화. 인덱스 조건을 더 깊이 활용함

**Using index for group-by**: 인덱스를 활용해 정렬 없이 GROUP BY를 수행하는 최적화

**타이트 인덱스 스캔(인덱스 스캔)을 통한 GROUP BY 처리**: 인덱스 전체를 순차 스캔하여 GROUP BY 수행

**루스 인덱스 스캔을 통한 GROUP BY 처리**: 필요한 레코드만 듬성듬성 스캔하는 최적화된 GROUP BY 처리 방식

**Using index for skip scan**: 인덱스의 선행 컬럼 조건 없이 후행 컬럼 조건만으로 스캔하는 최적화

**Using join buffer (Block Nested Loop)**: 인덱스가 없을 때 블록 네스티드 루프 방식으로 조인 처리

**Using join buffer (Batched Key Access)**: MRR 기반의 BKA 조인. 인덱스 키를 한꺼번에 전달해 조인 성능 향상

**Using join buffer (hash join)**: 드리븐 테이블에 해시 버퍼를 사용한 조인 방식

**Using MRR**: 여러 인덱스 키를 스토리지 엔진에 일괄 전달하여 디스크 접근을 최소화하는 최적화

**Using sort_union(...)**: OR 조건이 많고 결과 건수가 많을 때, 인덱스 결과를 정렬 후 병합

**Using union(...)**: OR 조건에서 두 인덱스의 결과를 병합하여 처리

**Using intersect(...)**: AND 조건에서 두 인덱스의 결과를 교집합으로 병합

**Using temporary**: GROUP BY, DISTINCT 등의 중간 결과를 저장하기 위해 임시 테이블을 사용

**Using where**: MySQL 엔진에서 조건 필터링 작업을 수행한 경우 (스토리지 엔진 레벨에서 못 거른 조건)

**Zero limit**: LIMIT 0 쿼리. 결과값이 아닌 메타데이터만 조회할 때 사용됨
