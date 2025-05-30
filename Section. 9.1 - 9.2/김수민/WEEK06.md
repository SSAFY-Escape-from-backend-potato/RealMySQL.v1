# Chpater 9. 옵티마이저와 힌트

여행을 할 때 최소의 비용이 드는 방법을 알아보고 여행 경로를 정하는 것처럼 MySQL에서도 쿼리를 최적으로 실행하기 위해 각 **테이블의 데이터가 어떤 분포**로 저장돼 있는지 통계 정보를 참조하며, 그러한 기본 데이터를 비교해 **최적의 실행 계획**을 수립하는 작업이 필요하다.

MySQL 서버를 포함한 대부분의 DBMS에서는 옵티마이저가 이러한 기능을 담당한다.

MySQL에서는 `EXPLAIN` 명령여로 쿼리 실행 계획을 확인할 수 있다.

## 9.1 개요

DBMS에서 옵티마이저는 가장 복잡하고 실행 계획을 이해하는 것 또한 상당히 어려운 부분이다. 하지만 실행 계획을 이해할 수 있어야만 실행 계획의 불합리한 부분을 찾아내고 더 최적화된 방법으로 실행 계획을 수립하도록 유도할 수 있다.

### 9.1.1 쿼리 실행 절차

MySQL 서버는 SQL 문장 그 자체가 아니라 **SQL 파스 트리**를 이용해 쿼리를 실행한다.

MySQL 서버에서 쿼리가 실행되는 과정은 크게 세 단계로 나눌 수 있다.

1. **SQL Parsing (→ Parse Tree 도출)**

   : 사용자로부터 요청된 SQL 문장을 잘게 쪼개서 MySQL 서버가 이해할 수 있는 수준으로 분리 (**parse tree**)한다.

   MySQL 서버의 ‘SQL 파서’ 모듈로 처리하며 SQL 문법 오류가 걸러지는 단계이다.

2. **최적화 및 실행 계획 수립 (→ 실행 계획 도출)**

   : SQL의 파싱 정보 (parse tree)를 확인하면서 어떤 테이블부터 읽고 어떤 인덱스를 이용해 테이블을 읽을지 선택한다.

   불필요한 조건 제거, 복합한 연산 단순화, 다중 테이블 조인 시 테이블 읽는 순서, 사용할 인덱스 결정, 레코드 가공 여부 결정 등등

3. 두 번째 단계에서 결정된 테이블의 읽기 순서나 선택된 인덱스를 이용해 스토리지 엔진을 통해 **데이터를 가져온다.**

### 9.**1.2 옵티마이저의 종류**

- **비용 기반 최적화 (Cost-based optimizer, CBO)**
  대부분의 DBMS가 선택하고 있다.
  쿼리를 처리하기 위한 여러 방법들을 만들고 각 단위 작업의 비용 (부하) 정보와 대상 테이블의 예측된 통계 정보를 이용해 실행 계획별 비용을 산출한다. **산출된 실행 방법들 중 최소 비용의 처리 방식을 선택**한다.
- **규칙 기반 최적화 (Rule-based optimizer, RBO)**
  대상 테이블의 레코드 건수나 선택도 등을 고려하지 않고 옵티마이저에 **내장된 우선순위에 따라 실행 계획을 수립**한다. 그렇기 때문에 같은 쿼리에 대해서는 거의 항상 같은 실행 방법이 생성된다.

## 9.2 기본 데이터 처리

DBMS들은 데이터 정렬, 그루핑 등의 기본 데이터 가공 기능을 기본으로 가지고 있지만 과정은 천차만별이다. MySQL은 이런 기본적인 가공을 위해 다음의 알고리즘들을 사용한다.

### 9.2.1 풀 테이블 스캔과 풀 인덱스 스캔

풀 테이블 스캔은 인덱스를 사용하지 않고 테이블의 데이터를 처음부터 끝까지 읽어서 요청된 작업을 처리하는 방식을 의미한다.

MySQL 옵티마이저는 다음과 같은 조건이 일치할 때 주로 풀 테이블 스캔을 선택한다.

- **테이블의 레코드 건수가 너무 작아서** 인덱스를 통해 읽는 것보다 풀 테이블 스캔을 하는 편이 더 빠른 경우 (일반적으로 테이블이 페이지 1개로 구성된 경우)
- WHERE절이나 ON 절에 **인덱스를 이용할 수 있는 적절한 조건이 없는 경우**
- 인덱스 레인지 스캔을 사용할 수 있는 쿼리라고 하더라도 **옵티마이저가 판단한 조건 일치 레코드 건수가 너무 많은 경우** (인덱스의 B-Tree를 샘플링해서 조사한 통계 정보 기준)

일반적으로 테이블 전체 크기는 인덱스보다 훨씬 크기 때문에 테이블을 처음부터 끝까지 읽는 작업은 상당히 많은 디스크 읽기가 필요하다. 그래서 대부분 DBMS 는 풀 테이블 스캔을 실행할 때 한꺼번에 **여러 개의 블록이나 페이지를 읽어오는 기능**을 내장하고 있다.

풀 테이블 스캔을 실행할 때 MyISAM 스토리지 엔진에서는 페이지를 하나씩 읽어오지만 InnoDB 스토리지 엔진은 특정 테이블의 연속된 데이터 페이지가 읽히면 백그라운드 스레드애 의해 **Read ahead\*** 작업이 자동으로 시작된다.

.\* **Read ahead**: 어떤 영역의 데이터가 앞으로 **필요해질 것을 예측**하여 요청이 오기 전 미리 디스크에서 읽어 InnoDB 버퍼 풀에 가져다 두는 것. 앞으로 필요할 페이지를 예측 → 미리 디스크에서 읽음 → InnoDB Buffer Pool에 적재 → 디스크 I/O 감소 → 성능 향상.

+) 버퍼 풀: 디스크 페이지를 메모리에 캐싱하는 InnoDB의 메모리 공간.

### 9.2.2 병렬 처리

여기서의 병렬 처리는 **하나의 쿼리를** 여러 스레드가 작업을 나누어 **동시에 처리**하는 것을 의미한다.

여러 스레드가 동시에 각각의 쿼리를 처리하는 것은 MySQL 서버가 처음 만들어질 때부터 가능했으나 MySQL 8.0부터 하나의 쿼리에 대한 병렬처리가 제한적으로 가능해졌다.

현재는 아무런 WHERE 조건 없이 단순히 테이블의 전체 건수를 가져오는 쿼리만 병렬로 처리할 수 있다.

병렬 처리용 스레드 개수가 늘어날 수록 쿼리 처리에 걸리는 시간은 줄어든다. 하지만 **스레드 개수를 아무리 늘리더라도 서버의 CPU 코어 개수를 넘어서는 경우에는 오히려 성능이 저하될 수 있다.**

### 9.2.3 ORDER BY 처리 (Using filesort)

레코드 1~2건을 가져오는 쿼리를 제외하면 대부분의 SELECT 쿼리에서 정렬은 필수적으로 사용된다.

- **정렬 처리 방법 두 가지**
  |             | 장점                                                                                                         | 단점                                                                                 |
  | ----------- | ------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------ |
  | 인덱스 이용 | - INSERT, UPDATE, DELETE 쿼리가 실행될 때 이미 인덱스가 정렬돼 있어 순서대로 읽기만 하면 되므로 매우 빠르다. | - INSERT, UPDATE, DELETE 작업 시 부가적인 인덱스 추가/삭제 작업이 필요하므로 느리다. |
  - 인덱스를 위한 추가적인 디스크 공간이 필요하다.
  - 인덱스 개수가 늘어날 수록 InnoDB 버퍼 풀을 위한 메모리가 많이 필요하다. |
    | Filesort 이용 | - 인덱스를 생성하지 않아도 되므로 인덱스 사용 시의 단점이 없다.
  - 정렬해야 할 레코드가 많지 않으면 메모리에서 Filesort가 처리되므로 충분히 빠르다. | - 정렬 작업이 쿼리 실행 시 처리되므로 레코드 대상 건수가 많아질수록 쿼리 응답 속도가 느리다. |

MySQL 서버에서 인덱스를 이용하지 않고 별도의 정렬 처리를 수행했는지는 실행 계획의 `Extra` 칼럼에 **“Using filesort”**메시지가 표시되는지 여부로 알 수 있다.

- **소트 버퍼 (Sort buffer)**
  **정렬 시 사용되는** 별도의 **메모리 공간** (`sort_buffer_size`로 설정).
  쿼리 실행이 완료되면 즉시 시스템으로 반납된다.
  if) 정렬해야 할 레코드 건수가 소트 버퍼로 할당된 공간보다 크다면?
  정렬할 레코드를 여러 조각으로 나누어 처리하고 이 과정에서 임시 저장을 위해 디스크를 사용한다.
  메모리의 소트 버퍼에서 정렬을 수행하고, 그 결과를 임시로 디스크에 기록한다. 다음 레코드를 가져와 다시 정렬 후 디스크에 임시저장한다. 이처럼 각 버퍼 크기만큼 정렬된 레코드를 다시 병합하면서 정렬을 수행해야 한다. 이 병합 작업을 **멀티 머지(Multi-merge)**라고 하고 수행된 머지 횟수는 `Sort_merge_passes` 라는 상태변수에 누적 집계된다. (`SHOW STATUS VARIABLES`)
  Q1. Sort buffer를 크게 설정하면 디스크를 사용하지 않으니 속도가 빨라질까?
  A1.
  (X). 벤치마크 결과 큰 차이 없음. 리눅스 계열 OS에서는 너무 큰 sort_buffer_size를 사용하면 큰 메모리 공간 할당으로 성능이 훨씬 떨어질 수도 있다.
  소트 버퍼를 너무 크게 설정하면 서버의 메모리가 부족해져 OS의 OOM-Killer가 여유 메모리 확보를 위해 프로세스를 강제종료시킬 수 있다. 빠른 성능을 얻을 수는 없지만 디스크IO는 줄일 수 있다.
  경험상 일반적인 트랜잭션 처리용 소트 버퍼 크기는 56KB~1MB 미만이 적절함
  소트 버처는 **세션 메모리 영역\***에 해당함. → 여러 클라이언트가 공유할 수 없음.

.\* MySQL의 메모리는 글로벌 메모리 영역과 세션 (로컬) 메모리 영역으로 나눌 수 있다. 4.1.3 메모리 할당 및 사용 구조 참조.

- **정렬 알고리즘**
  레코드 정렬 시 소트 버에 담는 데이터 범위에 따라 다음 2가지 정렬모드로 나눌 수 있다. (공식 명칭은 아님)
  - **Single-pass 정렬 방식**
    소트 버퍼에 정렬 기준 칼럼을 포함해 SELECT 대상이 되는 칼럼 전부를 담아서 정렬을 수행하는 방식이다. 정렬이 완료되면 정렬 버퍼의 내용을 그대로 클라이언트로 넘겨준다.
  - **Two-pass 정렬 방식**
    정렬 대상 칼럼과 PK 값만 소트 버퍼에 담아서 정렬을 수행하고 정렬된 순서대로 다시 PK로 테이블을 읽어 SELECT할 칼럼을 가져오는 방식이다.
    싱글 패스 정렬 방식이 도입되기 이전부터 사용되던 방식이다.
  **투 패스 방식은 테이블을 두 번 읽어야 하기 때문에 상당히 비효율적이지만 싱글 패스 방식은 이런 비효율이 없다. 하지만 싱글 패스 정렬 방식은 더 많은 소트버퍼 공간이 필요하다.**
  **싱글 패스 방식**은 **정렬 대상 레코드의 크기나 건수가 작은 경우 더 빠른 성능**을 보이며, **투 패스 방식**은 **정렬 대상 레코드의 크기나 건수가 많은 경우 효율적**이다.
  최신 버전에서는 일반적으로 싱글 패스 방식을 주로 사용하지만 다음과 같은 경우 싱글 패스 방식을 사용하지 못하고 투 패스 정렬 방식을 사용한다.
  - 레코드의 크기가 `max_length_for_sort_data` 시스템 변수 설정 값보다 클 때
  - BLOB이나 TEXT 타입의 칼럼이 SELECT 대상에 포함될 때
- **정렬 처리 방법**

  쿼리에 ORDER BY가 사용되면 반드시 다음 3가지 처리 방법 중 하나로 정렬이 처리된다. 일반적으로 아래로 갈 수록 처리 속도는 떨어진다.

  | 정렬 처리 방법                                  | 실행 계획의 Extra 칼럼 내용       |
  | ----------------------------------------------- | --------------------------------- |
  | 인덱스를 사용한 정렬                            | 별도 표기 X                       |
  | 조인에서 드라이빙 테이블만 정렬                 | “Using filesort”                  |
  | 조인에서 조인 결과를 임시 테이블로 저장 후 정렬 | “Using temporary; Using filesort” |

  - **인덱스를 이용한 정렬**
    인덱스를 이용해 정렬이 처리되는 경우 실제 **인덱스의 값이 정렬돼 있기 때문에 인덱스의 순서대로 읽기만 하면 된다.** 별도의 정렬을 위한 추가 작업은 수행하지 않는다.
    인덱스를 이용한 정렬을 사용하려면 반드시
    1. ORDER BY에 명시된 칼럼이 **제일 먼저 읽는 테이블에 속하고**, (조인이 사용된 경우 드라이빙 테이블)
    2. **ORDER BY의 순서대로 생성된 인덱스**가 있어야 한다.
    3. 또한, WHERE절에 첫 번째로 읽는 테이블의 칼럼에 대한 조건이 있다면 그 조건과 ORDER BY는 같은 인덱스를 사용할 수 있어야 한다.
    B-Tree 계열의 인덱스 가 아닌 해시 인덱스나 전문 검색 인덱스 등에서는 인덱스를 이용한 정렬을 사용할 수 없다.
    ```sql
    인덱스로 정렬이 처리될 때는 ORDER BY가 명시된다고 해서 작업량이 더 늘어나지 않는다. ORDER BY를 명시하면 성능상의 손해가 없음은 물론이고 실행 계획 변경 시에도 기대한 결과 순서대로 데이터를 가져올 수 있으므로 어플리케이션 버그를 방지할 수 있다..
    ```
  - **조인의 드라이빙 테이블만 정렬**
    조인이 수행되면 결과 레코드의 건 수와 각 레코드의 크기도 커지기 때문에 **조인 실행 전 첫 번째 테이블의 레코드를 먼저 정렬한 다음 조인을 실행**하는 방법을 대안으로 생각해볼 수 있다.
    이 방법으로 정렬이 처리되려면 조인에서 첫 번째로 읽히는 테이블 (드라이빙 테이블)의 칼럼만으로 ORDER BY절을 작성해야 한다.
  - **임시 테이블을 이용한 정렬**

    2개 이상의 테이블을 조인한 결과를 정렬해야 한다면 임시 테이블이 필요할 수도 있다. 앞의 ‘조인의 드라이빙 테이블만 정렬’하는 방식 외의 조인 정렬 쿼리에서는 **항상 조인 결과를 임시 테이블에 저장하고 그 결과를 다시 정렬**하는 과정을 거친다.

    정렬의 3가지 방법 중 정렬해야 할 레코드 건수가 가장 많기 때문에 가장 느린 정렬 방법이다.

    ex) 드라이빙 테이블이 아닌 드리븐 테이블 칼럼이 ORDER BY에 포함된 경우

  - **정렬 처리 방법의 성능 비교**
    LIMIT는 테이블이나 처리 결과의 일부만 가져오기 때문에 MySQL 서버가 처리해야 할 작업량을 줄이는 역할을 한다.
    그러나 ORDER BY나 GROUP BY 같은 작업은 **WHERE 조건을 만족하는 레코드를 모두 가져와서 정렬/그루핑 작업을 실행해야만 LIMIT으로 건 수를 제한**할 수 있다. 그렇기 때문에 LIMIT처럼 결과 건수를 제한하는 조건은 네트워크로 전송되는 레코드의 건수를 줄일 수는 있지만 MySQL 서버가 해야 하는 작업량에는 그다지 변화가 없다.
    - **스트리밍 방식**
      서버쪽에서 처리할 데이터가 얼마인지에 관계 없이 **조건에 일치하는 레코드가 검색될 때마다 바로바로 클라이언드로 전송해주는 방식**이다.
      클라이언트는 MySQL 서버가 일치하는 레코드를 찾는 즉시 전달받기 때문에 동시에 데이터의 가공 작업을 시작할 수 있다. 쿼리가 얼마나 많은 레코드를 조회하느냐에 관계 없이 **빠른 응답시간**을 보장해준다.
      웹 서비스 같은 OLTP\* 환경에서는 쿼리 요청으로부터 첫 번째 레코드를 전달받게 되기까지의 응답 시간이 중요하다.
      스트리밍 방식으로 처리되는 쿼리에서 LIMIT 조건은 쿼리의 전체 실행시간을 상당히 줄여줄 수 있다.
      .\* OLTP: Online Transaction Processing, 네트워크 상 온라인 사용자들의 Database 요청에 대한 일괄 트랜잭션을 처리하는 것을 의미한다.
    - **버퍼링 방식**

          ORDER BY, GROUP BY 등에서는 쿼리의 결과를 스트리밍하지 못한다.

          MySQL 서버에서는 모든 레코드를 검색하고 정렬 작업을 하는 동안 클라이언트는 아무것도 하지 않고 기다려야 하기 때문에 응답 속도가 느려진다.

          ```sql
          스트리밍 처리는 어떤 클라이언트 도구를 사용하는지에 따라 방식이 달라질 수 있다. MySQL 서버가 스트리밍 방식으로 데이터를 전송하더라도 JDBC는 MySQL 서버로부터 받는 레코드를 내부 버퍼에 담아두고 마지막 레코드까지 모두 전달 받으면 그때서야 데이터를 클라이언트 어플리케이션에 반환한다.
          JDBC의 기본 동작은 버퍼링 방식이며 대량의 데이터를 가져와야 할 때는 스트리밍 방식으로 변경할 수 있다.
          ```

      9.2.3.3절 ‘정렬 처리 방법’의 3가지 처리 방법 중 **인덱스를 사용한 정렬방식만이 스트리밍 형태의 처리**이다.
    인덱스를 사용하지 못하고 별도의 Filesort 작업이 필요한 쿼리에서 LIMIT는 도움이 되긴 하지만 기대만큼 쿼리가 아주 빨라지지는 않는다. MySQL 서버에서 상위 N건만 정렬이 채워지만 정렬을 멈추고 결과를 반환한다. 하지만 MySQL 서버는 정렬을 위해 Quick Sort와 Heap Sort를 사용한다.

- **정렬 관련 상태 변수**
  MySQL 서버는 처리하는 주요 작업에 대해서 해당 작업의 실행 횟수를 상태 변수로 저장한다.
  정렬과 관련해서도 지금까지 몇 건의 레코드를 정렬처리했는지, 소트 버퍼 간의 병합 작업 (멀티 머지)이 몇 번이나발생했는지 등을 확인할 수 있다.
    <aside>
    
    FLUSH STATSU;
    
    SHOW STATUS LIKE ‘Sort%’;
    
    </aside>


### 9.2.4 GROUP BY 처리

GROUP BY는 ORDER BY와 마찬가지로 쿼리가 스트리밍된 처리를 할 수 없게 하며, GROUT BY에 사용된 조건은 인덱스를 사용해서 처리될 수 없다.

GROUP BY 작업은 인덱스를 사용하는 경우와 그렇지 못한 경우로 나누어 볼 수 있다. 인덱스를 사용하는 경우 인덱스를 차례로 읽는 인덱스 스캔 방법과 인덱스를 건너뛰면서 읽는 루스 인덱스 스캔으로 나뉜다.

인덱스를 사용하지 못하는 쿼리에서 GROUP BY 작업은 임시 테이블을 사용한다.

- **인덱스 스캔을 이용하는 GROUP BY (타이트 인덱스 스캔)**

  ORDER BY와 마찬가지로 **조인의 드라이빙 테이블에 속한 칼럼만 이용해 그루핑**할 때 GROUP BY 칼럼으로 이미 인덱스가 있다면 그 인덱스를 차례대로 읽으면서 그루핑 작업을 수행하고 그 결과로 조인을 처리한다.

  인덱스를 통해 처리되는 쿼리는 이미 정렬된 인덱스를 읽는 것이므로 쿼리 실행 시점에 추가적인 정렬 작업이나 내부 임시 테이블은 필요하지 않다. GROUP BY가 인덱스를 사용해 처리된다 하더라도 그룹 합수 등의 그룹값을 처리해야 해서 임시 테이블이 필요할 때도 있다.

  이러한 그루핑 방식을 사용하는 쿼리 실행 계획에서는 Extra 칼럼에 별도로 GROUP BY 관련 코멘트 `Using index for group-by` 나 임시 테이블 사용 또는 정렬 관련 코멘트 `Using temporary, Using filesort` 가 표시되지 않는다.

- **루스 인덱스 스캔을 이용하는 GROUP BY**

  루스 인덱스 스캔 방식은 인덱스의 레코드를 건너뛰면서 필요한 부분만 읽어서 가져오는 것을 의미한다.

  옵티마이저가 루스 인덱스 스캔을 사용할 때는 실행 계획의 Extra 칼럼에 `Using index for group-by` 코멘트가 표시된다.

  10.3.12.24.2절 ‘루스 인덱스 스캔을 통한 BROUP BY 처리’ 내용 참조.

  MySQL의 루스 인덱스 스캔은 단일 테이블에 대해 수행되는 GROUP BY 처리에만 사용할 수 있다. 프리픽스 인덱스\*는 루스 인덱스 스캔을 사용할 수 없다.

  **인덱스 레인지 스캔에선는 유니크한 값의 수가 많을수록 성능이 향상되는 반면 루스 인덱스 스캔에서는 인덱스의 유니크한 값의 수가 적을수록 성능이 향상된다.**

  즉, 루스 인덱스 스캔은 분포도가 좋지 않은 인덱스일수록 더 빠른 결과를 만들어낸다. 루스 인덱스 스캔으로 처리되는 쿼리에서는 별도의 임시 테이블이 필요하지 않다.

  루스 인덱스 스캔이 사용될 수 있을지 없을지 판단하는 것은 WHERE절의 조건이나 ORDER BY 절이 인덱스를 사용할 수 있을지 없을지 판단하는 것보다 더 어렵다.

  .\* Prefix index: 칼럼 값의 앞쪽 일부만으로 생성된 인덱스

- **임시 테이블을 사용하는 GROUP BY**
  GROUP BY의 기준 칼럼이 드라이빙 테이블에 있든 드리븐 테이블에 있든 관계 없이 인덱스를 전혀 사용하지 못 하는 경우 이 방식으로 처리된다.
  Extra 칼럼에 `Using temporary` 메시지가 표시되게 된다.
  MySQL 8.0 이전까지는 GROUP BY가 사용된 쿼리는 그루핑 칼럼을 기준으로 묵시적인 정렬도 함께 수행됐으나, 8.0버전부터는 묵시적인 정렬은 더이상 실행되지 않는다.
  MySQL 8.0에서는 GROUP BY가 필요한 경우 내부적으로 GROUP BY절의 칼럼들로 구성된 유니크 인덱스를 가진 임시 테이블을 만들어서 중복 제거와 집합 함수 연산을 수행한다.

### **9.2.5 DISTINCT 처리**

DISTINCT는 MIN(), MAX(), COUNT()같은 **집계 함수\***와 함께 사용되는 경우와 집계 함수가 없는 경우 2가지로 구분해서 살펴볼 수 있다. 이 두 경우 DISTINCT가 영향을 미치는 범위가 달라진다.

**집합 함수\*\***와 같이 DISTINCT가 사용되는 쿼리의 실행 계획에서 DISTINCT 처리가 인덱스를 사용하지 못 할 경우 항상 임시 테이블이 필요하다. 하지만 **실행 계획의 Extra 칼럼에는 Using temporary 메시지가 출력되지 않는다.**

인덱스된 칼럼에 대해 DISTINCT 처리를 수행할 때는 인덱스를 풀 스캔하거나 레인지 스캔하면서 임시 테이블 없이 최적화된 처리를 수행할 수 있다.

**.\* 집계 함수(Aggregate Function):** 데이터 그룹을 하나의 값으로 요약하는 함수 (SUM, COUNT, AVG, MAX, MIN 등)

**.** 집합 함수(Set Function):** 보통 **집계 함수 중 DISTINCT를 포함해서 집합적인 의미로 처리\*\*할 때 주로 부르는 표현

- **SELECT DISTINCT**

  단순히 SELECT되는 레코드 중에서 유니크한 레코드만 가져오고자 하면 SELECT DISTINCT 형태의 쿼리 문장을 사용한다. 이 경우는 GROUP BY와 내부적으로 동일한 방식으로 처리된다.

  ```sql
  DISTINCT는 레코드를 유니크하게 SELECT하는 것이지, 특정 칼럼만 유니크하게 조회하는 것이 아니다.
  ```

- **집합 함수와 함께 사용된 DISTINCT**
  집합 함수가 없는 SELECT 쿼리에서 DISTINCT는 **조회하는 모든 칼럼의 조합이 유니크한 것들**만 가져온다. 하지만 집합 함수 내에서 사용된 DISTINCT는 그 **집합 함수의 인자로 전달된 칼럼값이 유니크한 것들**을 가져온다.

### 9.2.6 내부 임시 테이블 (Internal temporary table)활용

MySQL 엔진이 스토리지 엔진으로부터 받아온 레코드를 정렬하거나 그루핑 할 때는 내부적인 임시 테이블을 사용한다.

사용자가 CREATE TAMPORARY TABLE로 만든 임시 테이블과 달리 내부 임시 테이블은 쿼리의 처리가 완료되면 자동으로 삭제된다. 또한 다른 세션이나 다른 쿼리에서는 볼 수 없으며 사용하는 것도 불가능하다.

```sql
MySQL 서버는 디스크의 임시 테이블 생성 시 파일 **오픈 즉시 파일 삭제**를 실행하여 MySQL 서버가 종료되거나 해당 쿼리가 종료되면 임시 테이블이 즉시 사라지도록 보장한다.
(파일이 오픈된 상태에서 삭제되면 OS는 그 파일을 즉시 삭제하지 않고, 파일을 참조하는 프로세스가 모두 없어지면 그때 자동으로 파일을 삭제한다.)
```

- **메모리 임시 테이블과 디스크 임시 테이블**

  MySQL 8.0 이전 버전까지는 원본 테이블의 스토리지 엔진과 관계 없이 임시 테이블이 메모리를 사용할 때는 MEMORY 스토리지 엔진을 사용하며, 디스크에 저장될 때는 MyISAM 스토리지 엔진을 이용했다.

  MySQL 8.0 버전부터 메모리는 TempTable이라는 스토리지 엔진을 사용하고, 디스크는 InnoDB 스토리지 엔진을 사용하도록 개선됐다.

  MEMORY 스토리지 엔진은 `varchar`, `varbinary` 같은 가변 길이 타입 지원 X, 가변타입은 최대 길이만큼 메모리 할당 → TempTable 스토리지 엔진

  MyISAM 스토리지 엔진은 트랜잭션 지원 X → InnoDB 스토리지 엔진

  임시 테이블의 크기가 1GB보다 커지는 경우 MySQL 서버는 메모리의 임시 테이블을 디스크로 기록하게 되는데, 이때 MySQL 서버는 다음의 2가지 디스크 저장 방식 중 하나를 선택한다.

  1. MMAP 파일로 디스크에 기록 → 오버헤드 적음
  2. InnoDB 테이블로 기록

  `temptable_use_mmap` 시스템 변수로 설정 가능. 디스크에 생성되는 임시 테이블은 `tmpdir` 시스템 변수에 정의된 디렉터리에 저장된다.

- **임시 테이블이 필요한 쿼리**

  다음과 같은 패턴의 쿼리는 MySQL 엔진에서 별도의 데이터 가공 작업을 필요로 하므로 대표적으로 이시 테이블을 생성하는 케이스다.

  - ORDER BY와 GROUP BY에 명시된 칼럼이 다른 쿼리
  - ORDER BY나 GROUP BY에 명시된 칼럼이 조인의 순서상 첫번째 테이블이 아닌 쿼리
  - DISTINCT와 ORDER BY가 동시에 쿼리에 존재하는 경우
  - DISTINCT가 인덱스로 처리되지 못하는 쿼리
  - UNION이나 UNION DISTINCT가 사용된 쿼리 (select_type 칼럼이 UNION RESULT인 경우)
  - 쿼리의 실행 계획에서 select_type이 DERIVED인 쿼리

  마지막 3개의 패턴은 쿼리 실행 계획에서 Using temporary가 표시되지 않고 임시 테이블이 사용되는 예이다.

- **임시 테이블이 디스크에 생성되는 경우**

  내부 임시 테이블은 기본적으로 메모리에 생성되지만 다음 조건을 만족하면 메모리 임시 테이블을 사용할 수 없다.

  - UNION이나 UNION ALL에서 SELECT되는 칼럼 중 길이가 512바이트 이상인 크기의 칼럼이 있는 경우
  - GROUP BY나 DISTINCT 칼럼에서 512바이트 이상인 크기의 칼럼이 있는 경우
  - 메모리 임시 테이블의 크기가 tmp_table_size 또는 max_heap_table_size 시스템 변수보다 크거나 temptable_max_ram 시스템 변수 값보다 큰 경우

- **임시 테이블 관련 상태 변수**
  실행 계획에서 Using temporary가 표시되면 임시 테이블을 사용했음을 알 수 있다. 하지만 임시 테이블이 메모리에서 처리됐는지 디스크에서 처리됐는지는 알 수 없으며 멸 개의 임시 테이블이 사용됐는지도 알 수 없다.
  임시 테이블이 디스크에 생성됐는지 메모리에 생성됐는지는 SHOW SESSION STATUS LIKE ‘Create_tmp%’;를 통해 확인할 수 있다.
