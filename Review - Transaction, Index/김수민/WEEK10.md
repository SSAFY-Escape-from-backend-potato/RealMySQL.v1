# Chapter 5. 트랜잭션과 잠금

### Q. 트랜잭션의 단위는 어떻게 설정하는 것이 좋을까요?

A. DBMS의 커넥션과 마찬가지로 꼭 필요한 코드 부분에 최소 단위로 적용하는 것이 좋다.

하나의 트랜잭션에 해당하는 코드 범위가 넓을 수록 도중에 오류가 발생하면 롤백 처리를 해야 하는 범위 또한 넓어진다. 따라서 네트워크를 통해 원격 서버와 통신하는 작업 등은 최대한 DBMS의 트랜잭션 내에서 제거하는 것이 좋으며, 반드시 하나의 트랜잭션으로 묶여야 하는 작업과 트랜잭션이 필요하지 않은 작업을 구분하여 별도의 트랜잭션으로 분리하는 것이 좋다.

일반적으로 DB 커넥션은 개수가 제한적이어서 각 단위 프로그램이 커넥션을 소유하는 시간이 길어질수록 하용 가능한 여유 거넥션의 개수는 줄어들게 된다. (p. 159)

### Q. employees 테이블에는 first_name 칼럼에 대한 인덱스만 존재합니다. 다음 결과를 보고 UPDATE문 실행 시 InnoDB에서 lock을 걸어야 하는 레코드 수에 대해 설명해주세요.

```sql
SELECT COUNT(*)
FROM employees
WHERE first_name='Gorgi';

=> 932

SELECT COUNT(*)
FROM employees
WHERE first_name='Gorgi' AND last_name='Klassen';

=> 1

UPDATE employees SET salary='5000000'
WHERE first_name='Gorgi' AND last_name='Klassen';
```

A. 932건의 레코드에 lock을 걸게 된다.

InnoDB는 레코드를 잠그는 것이 아니라 인덱스를 잠그는 방식으로 처리된다. last_name 칼럼은 인덱스가 없기 때문에 first_name=’Gorgi’인 레코드 932건을 모두 감궈야 한다.

적절한 인덱스가 준비되어 있지 않다면 각 클라이언트 간의 동시성이 상당히 떨어질 수 있으므로 인덱스를 적절히 설계하는 것이 중요하다.

만약 테이블에 인덱스가 하나도 없다면, 테이블을 풀 스캔 하면서 UPDATE 작업을 하게되고 이 과적에서 테이블에 있는 모든 레코드를 잠그게 된다. (p. 171)

### Q. shared lock과 exclusive lock에 대해 설명하고 각각이 필요한 상황을 얘기해주세요.

A.

| 구분           | Shared Lock                     | Exclusive Lock             |
| -------------- | ------------------------------- | -------------------------- |
| 사용 목적      | 읽기 보호                       | 쓰기 보호                  |
| 다른 읽기 허용 | O                               | X                          |
| 다른 쓰기 허용 | X                               | X                          |
| 충돌 여부      | X-Lock과 충돌                   | S-Lock, X-Lock과 모두 충돌 |
| 사용 예시      | `SELECT ... LOCK IN SHARE MODE` | `UPDATE`, `DELETE` 등      |

- 공유 락 사용 예시
  - 금융 시스템에서 고객 계좌 정보를 읽는 여러 트랜잭션이 있을 때, 동시에 조회는 가능하지만, 읽기 도중 UPDATE는 허용되어서는 안된다.
- 배타 락 사용 예시
  - 쇼핑몰에서 재고 수량을 차감할 때 서로 다른 트랜잭션이 동시에 수정할 수 없어야 한다.

### Q. dirty read, non-repeatable read, phantom read에 대해 설명하고 각각이 발생 가능한 격리수준을 이야기해주세요.

A.

위 세가지 이상현상은 모두 transaction의 isolation속성에 위배된다.

(여러 트랜잭션이 동시에 일어나더라도 각각의 트랜잭션은 다른 트랜잭션의 영향을 받지 않고 독립된 실행과 같은 결과를 보장해야 한다.)

- dirty read
  : commit되지 않은 레코드 값을 읽어 일어날 수 있는 이상현상
  ![image.png](<images\image(1).png>)
- non-repeatable read (fuzzy read)
  : 하나의 트랜잭션에서 같은 데이터를 두 번 읽었는데 값이 달라지는 이상현상
  ![image.png](<images\image(2).png>)
- phantom read
  : 하나의 트랜잭션에서 같은 조건으로 데이터를 두 번 읽었는데 없던 데이터가 생기는 이상현상
  ![image.png](<images\image(3).png>)

| Isolation Level  | Dirty Read | Non-Repeatable Read | Phantom Read |
| ---------------- | ---------- | ------------------- | ------------ |
| Read Uncommitted | O          | O                   | O            |
| Read Committed   | X          | O                   | O            |
| Repeatable Read  | X          | X                   | O            |
| Serializable     | X          | X                   | X            |

[transaction isolation level 설명! - 쉬운코드](https://youtu.be/bLLarZTrebU?si=k38W8m-N6ixkAql-)

→ 표준 isolation level의 확장된 관점과 주요 RDBMS에서 제공하는 isolation level에 대한 설명이 있어서 한 번 보면 좋을 것 같아요.

# Chapter 8. 인덱스

### Q. first_name에만 인덱스가 존재할 때, 다음 두 쿼리 실행 시 성능차이가 있는지 이야기하고 차이가 있다면 그 이유를 설명해주세요.

```sql
SELECT * FROM employees ORDER BY first_name ASC LIMIT 10;

SELECT * FROM employees ORDER BY first_name DESC LIMIT 10;
```

A. 역방향 스캔에서 약간의 오버헤드가 추가로 발생한다.

MySQL 서버의 InnoDB 스토리지 엔진에서 forward scan과 backward scan은 B-Tree의 리프 노드인 페이지 (블록) 간의 양방향 연결 리스트를 통해 뒤쪽부터 읽는지 앞쪽부터 읽는지의 차이가 있다.

하지만 InnoDB 리프 페이지 내부의 레코드는 단방향 링크로만 연결되어 있으며 InnoDB 페이지에서 데이터는 물리적으로 순서대로 저장되지 않는다. 또한, InnoDB의 page lock은 왼쪽→오른쪽 순차 접근을 전제로 설계되었다.

이 경우 역순 정렬 인덱스를 추가하여 성능을 개선할 수 있다. (p. 246)

+) 추가
아래 네 가지 내부 요인이 합쳐져 보통 10 ~ 20 % 정도의 성능 열세가 보고됩니다.

| 핵심 원인                     | 상세 메커니즘                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| ----------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 페이지 내부 링크 구조         | InnoDB 리프 페이지의 레코드는 단방향(singly) 링크로만 연결됩니다. • 정방향 스캔은 `next_record` 포인터 하나만 따라가면 O(1). • 역방향 스캔은 “현재 레코드의 바로 이전”을 모르기 때문에 페이지 첫 슬롯부터 다시 훑어 O(n/2) 탐색 → CPU 캐시 miss 증가.([dba.stackexchange.com](https://dba.stackexchange.com/questions/199551/why-scanning-an-index-backwards-is-slower?utm_source=chatgpt.com))                                                                                    |
| 프리페치(read-ahead) 알고리즘 | InnoDB의 선형·랜덤 read-ahead는 ‘왼쪽→오른쪽’ 순차 I/O를 전제로 얼마 이상 연속 페이지가 접근되면 다음 extents를 선읽기 합니다.([dev.mysql.com](https://dev.mysql.com/doc/refman/8.1/en/innodb-performance-read_ahead.html?utm_source=chatgpt.com), [dba.stackexchange.com](https://dba.stackexchange.com/questions/311766/why-linear-read-ahead-prefetch-may-improve-performance?utm_source=chatgpt.com)) 역방향은 이 패턴을 충족하지 못해 미리읽기 트리거율↓ → 저장장치 I/O 호출↑ |
| 버퍼 풀 LRU 관리              | 순차(정방향) 스캔은 mid-point insertion 휴리스틱과 맞물려 “머지않아 재사용될” 페이지를 young 리스트에 두지만, 역방향 패턴은 이를 빗겨가 page eviction 비율이 더 높아질 수 있음                                                                                                                                                                                                                                                                                                     |
| 경합(latch) 및 삽입 패턴      | PK·인덱스 키가 단조 증가(예: `AUTO_INCREMENT`)면 가장 오른쪽 리프 페이지에 INSERT가 집중됩니다. DESC 스캔은 바로 그 페이지부터 읽기 때문에 쓰기가 잦은 페이지와 같은 latch를 빈번히 경쟁 → 지연 증가.                                                                                                                                                                                                                                                                              |

| 구문                                                                              | 보정/추가 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| --------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **“InnoDB 페이지는 힙처럼 사용되기 때문에 물리적으로 순서대로 저장되지 않는다.”** | 레코드는 삽입 시 **페이지 내 자유 공간(free list)** 아무 곳에나 배치될 수 있어 **물리 주소는 정렬돼 있지 않다**. ❗그러나 **“정렬된 논리 순서를 빠르게 찾기”** 위해 페이지 하단부에 **_Page Directory_**(슬롯 배열)이 유지된다. • 디렉터리 슬롯은 4–8개 레코드마다 하나씩 “이 레코드의 오프셋”을 담아 두어, • (1) forward scan, (2) 역방향 탐색, (3) 이진 검색 모두를 가속한다. ([blog.jcole.us](https://blog.jcole.us/2013/01/14/efficiently-traversing-innodb-btrees-with-the-page-directory/?utm_source=chatgpt.com)) |
| **역방향 스캔 오버헤드**                                                          | 디렉터리가 있더라도 역방향 스캔은 ▶ 슬롯 이진 탐색 → 슬롯 내부 선형 탐색을 반복하므로, 정방향보다 **CPU·버퍼 풀 miss·프리페치 효율**이 소폭 떨어진다(실측 10 ~ 20 %).                                                                                                                                                                                                                                                                                                                                                    |
| **버퍼 풀·I/O 관점**                                                              | InnoDB의 **Read-Ahead** 로직과 **buffer-pool LRU(중간 지점 삽입)** 휴리스틱은 “왼쪽→오른쪽 순차 접근”을 전제로 설계되었다. 내림차순 스캔은 이 패턴을 덜 충족해 **예측 miss·eviction 비율**이 소폭 증가한다. ([dev.mysql.com](https://dev.mysql.com/doc/refman/8.2/en/innodb-performance-midpoint_insertion.html?utm_source=chatgpt.com))                                                                                                                                                                                 |

### Q. PK가 명시적으로 선언되지 않은 경우 InnoDB 테이블이 클러스터링 테이블을 구성하는 과정을 설명해주세요.

A.

1. PK가 있으면 PK를 클러스터링 키로 선택한다.
2. NOT NULL 옵션의 UNIQUE INDEX 중 첫 번째 인덱스를 클러스터링 키로 선택한다.
3. 유니크한 값을 가지도록 6 바이트짜리 RowID 칼럼 내부적으로 추가한 후 클러스터링 키로 선택한다.

이때, 적절한 클러스터링 키 후보를 찾지 못하는 경우 3번의 경우가 발생하며 이렇게 자동으로 추가된 일련번호 칼럼은 사용자에게 노출되지 않고 쿼리 문장에 명시적으로 사용할 수 없다.

즉, PK나 UNIQUE 인덱스가 없는 InnoDB 테이블에서는 아무 의미 없는 숫자로 클러스터링이 수행되며 사용자는 아무런 이점을 얻을 수 없다. 클러스터링 인덱스는 테이블당 하나만 생성 가능하므로 가능하면 PK를 명시적으로 생성하는 것이 좋다. (p. 273)
