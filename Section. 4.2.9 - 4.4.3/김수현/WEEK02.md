# Real MySQL 4.2.9 ~ 4.4.3

# 4.2 InnoDB 스토리지 엔진 아키텍처

- 언두 로그
    - 정의 : 트랜잭션과 격리 수준을 보장하기 위해 DML로 변경되기 이전 버전의 백업 데이터
        - 트랜잭션 보장 : 트랜잭션 롤백 시 데이터 복구
        - 격리 수준 보장 : 특정 커넥션에서 데이터 변경 도중에 다른 커넥션에서 데이터 조회 시 트랜잭션 격리 수준에 맞게 언두 로그에 백업해둔 데이터를 읽어서 반환
            1. Read UnCommited : 언두 로그 거의 사용하지 않음 → Dirty Read 발생 가능 [ex) 트랜잭션 A가 UPDATE users SET balance=1000 where id =1; 실행 후 커밋하지 않은 상태에서, 트랜잭션 B가 SELECT balance FROM users WHERE id =  1; 하면 1000이 보일 수 있음.]. 즉 언두 로그 활용하지 않고 커밋되지 않은 변경된 데이터 직접 읽음
            2. Read Committed : SELECT 실행 시 현 트랜잭션에서 변경된 데이터가 있다면 언두 로그에서 마지막 커밋된 데이터 조회해 반환. 즉, 트랜잭션 격리 수준에 따라 커밋되지 않은 데이터는 읽을 수 없음 → 언두 로그에서 최신 커밋된 데이터 버전을 찾아 반환 [ex) 트랜잭션 A가 UPDATE users SET balance=1000 WHERE id =1; 실행 후 커밋하지 않으면 트랜잭션 B가 SELECT balance FROM users WHERE id =  1; 실행 시 언두 로그에서 마지막 커밋된 값(예: 500) 반환. 트랜잭션 A가 커밋하고 다시 트랜잭션 B가 조회 시 1000이 보임
            3. Repeatable Read : 트랜잭션이 시작될 때 언두 로그를 기반으로 해당 트랜잭션만을 위한 스냅샷 버전 유지. 이후 SELECT 실행 시 언두 로그에서 트랜잭션 시작 시점의 데이터 반환. 다른 트랜잭션에서 UPDATE 혹은 DELETE 수행하더라도 현 트랜잭션 변경 전 데이터(스냅샷 데이터)를 계속 읽음 [ex) 트랜잭션 A가 시작됨 → 트랜잭션 B가 UPDATE users SET balance=1000 WHERE id =1; 실행 후 커밋 → 트랜잭션 A가 SELECT balance FROM users WHERE id =  1; 실행 but 여전히 트랜잭션 A 시작 시점의 데이터 값 반환 → 트랜잭션 A가 종료 후 다시 조회하면 최신 값(1000)을 볼 수 있음.
            4. Serializable : SELECT 시에도 테이블 또는 행 단위의 SHARED LOCK을 걸어, 다른 트랜잭션이 데이터를 변경할 수 없도록 함. 즉 언두 로그보다 락을 활용해 동시성 제한하는 것이 특징 [ex) 트랜잭션 A가 SELECT balance FROM users WHERE id =  1; 실행하면 SHARED LOCK 적용 → 트랜잭션 B가 UPDATE users SET balance=1000 WHERE id =1; 실행하면 A의 SELECT가 완료될 때까지 대기 → 트랜잭션 A가 종료되면 트랜잭션 B 실행
    - MySQL 5.5 이전 : 증가한 언두 로그 공간은 줄어들지 않는다. ex) 100GB 크기의 테이블 DELETE 시 테이블은 삭제되지만 언두 로그의 공간 사용량이 늘어나 100GB가 된다. & 트랜잭션 시작하고 완료하지 않은 상태로 하루 정도 방치 시 InnoDB 스토리지 엔진은 언두 로그를 계속 보존한다. 따라서 InnoDB 스토리지 엔진의 언두 로그는 하루치 데이터 변경 계속 저장하고 언두 로그 저장 공간은 계속 증가
    - MySQL 5.5 이후 : 언두 로그 공간 문제점 완전 해결되었고 특히 MySQL 8.0에서는 언두 로그를 돌아가면서 순차적으로 사용해 디스크 공간 줄인다. 그러나 SHOW ENGINE INNODB STATUS(모든 버전에서 사용가능) 명령어 혹은 SELECT count FROM information_schema.innodb_metrics WHERE SUBSYSTEM =’transaction’ AND NAME=’trx_rseg_history_len’(8.0버전 명령어)를 통해서 지속적으로 모니터링해야된다.
    - 언두 테이블스페이스 : 언두 로그가 저장되는 공간(추가적인 정리 필요)
        - MySQL 5.6 이전 : 언두 로그가 모두 시스템 테이블스페이스(ibdata.ibd)에 저장
        - MySQL 5.6 : innodb_undo_tablespaces 도입되어 2보다 큰 값으로 설정 시 InnoDB 스토리지 엔진은 더이상 언두 로그를 시스템 테이블스페이스에 저장하지 않는다. But 0으로 설정 시 MySQL 5.6이전과 동일하게 시스템 테이블스페이스에 언두 로그가 저장된다.
        - MySQL 8.0 : innodb_undo_tablespaces 시스템 변수가 Deprecated & 언두 로그는 항상 시스템 테이블스페이스 외부 별도 로그 파일로 기록

- 체인지 버퍼
    - 개요 : RDBMS에서 레코드 INSERT, UPDATE 시 데이터 파일 + 인덱스를 업데이트 작업 필요 → 인덱스 업데이트는 랜덤하게 디스크 읽는 작업, 많은 자원 소모
    - 정의 : 인덱스 페이지를 디스크로부터 읽어와 즉시 실행시키지 않고 임시 공간에 저장하고 사용자에게 결과 반환하는 형태로 성능 향상 → 이때 사용되는 임시 공간 = 체인지 버퍼
    - 체인지 머지 스레드 : 체인지 버퍼에 저장된 인덱스 레코드 조각은 백그라운드 스레드에 인해 병합되는 스레드

- 리두 로그 및 로그 버퍼
    - 리두 로그
        - 정의 : ACID 중 D와 연관되어 있어 MySQL 서버가 비정상적으로 종료되었을 때 데이터 파일에 기록되지 못한 데이터를 잃지 않게 해주는 안전장치
        - 개요 : 거의 모든 DBMS는 데이터 쓰기보다 읽기 성능 고려한 자료 구조 가지고 있기에 쓰기 작업에서 디스크 랜덤 엑세스 필요 즉 변경 데이터를 데이터 파일에 기록하는데 큰 비용 발생
        - 목적 : 쓰기 비용이 낮은 자료구조를 갖고 있어 비정상 종료 시 리두 로그 내용을 이용해 복구
        - MySQL 비정상 종료 후 InnoDB 스토리지 엔진의 비일관된 데이터 종류
            1. 커밋됐지만 데이터 파일에 기록되지 않은 데이터 : 리두 로그에 저장된 데이터를 데이터 파일에 복사해 복구
            2. 롤백됐지만 데이터 파일에 이미 기록된 데이터 : 리두 로그로 해결X, 변경되기 전 데이터 가진 언두 로그의 내용 가져와 데이터 파일에 복사, 리두 로그 역할은 커밋, 롤백, 트랜잭션 실행 중간 상태 중 어떠한 것인지 확인
        - innodb_flush_log_at_trx_commit : 리두 로그가 어느 주기로 디스크에 동기화될지 결정
            - 설정값 0 : 1초에 한 번씩 리두 로그를 디스크로 기록하고 동기화
            - 설정값 1 : 매번 트랜잭션 커밋 시 디스크로 기록되고 동기화
            - 설정값 2 : 매번 트랜잭션 커밋 시 디스크로 기록되지만 동기화는 1초에 한 번씩

              ⇒ 설정값 0, 2가 반드시 동기화 작업이 1초 간격으로 실행되는 것은 아니다. DDL로 인한 스키마 변경시 리두 로그가 디스크로 동기화되기에 1초보다 작은 간격으로 동기화 작업이 수행된다.

        - 리두 로그 파일 크기 : innodb_log_file_size 시스템 변수는 리두 로그 파일의 크기, innodb_log_file_in_group 시스템 변수는 리두 로그 파일의 개수를 결정 →전체 리두 로그 파일 크기 = innodb_log_file_size * innodb_log_file_in_group
        - 리두 로그 아카이빙
            - 정의 : 리두 로그가 저장 공간 초과하거나 특정 정책으로 인해 오래된 로그 보관할 필요 있을 떄 백업 및 보존하는 과정 → 데이터 복원, PITR 수행 가능
            - 종류
                1. 자동 아카이빙 : DBMS의 정책에 따라 자동으로 리두 로그를 보관
                2. 수동 아카이빙 : 관리자가 특정 시점에 리두 로그를 백업 및 이동 → 데이터 백업과 함께 수행되며 특정 트랜잭션 기록 장기 보관 목적으로 사용
            - 사용법
                1. innodb_redo_log_archive_dirs 시스템 변수로 아카이빙된 리두 로그가 저장될 디렉토리 설정
                2. innodb_redo_log_archive_start 사용자 정의 함수UDF를 실행(1개 또는 2개 파라미터 존재, 첫번째 파라미터 = 아카이빙할 디렉토리, 두번째 파라미터 = 서브디렉토리(선택사항))
                3.  innodb_redo_log_archive_stop UDF로 리두 로그 아카이빙 종료 → 삭제는 사용자가 수동으로
            - 아카이빙 세션 비정상 종료 시 아카이빙을 멈추고 아카이빙된 리두 로그도 전부 삭제
        - 리두 로그 활성화 및 비활성화
            - MySQL 8.0 이전 : 수동으로 리두 로그 비활성화 X
            - MySQL 8.0 이후 : 데이터 복구하거나 대용량 데이터 한 번에 적재하는 경우 리두 로그 비활성화해 데이터 적재 시간 단축 가능

              cf) MySQL 서버에서 트랜잭션 커밋 시 데이터 파일은 즉시 디스코로 동기화되지 않지만 리두 로그는 항상 디스크로 기록 → 이것 또한 innodb_flush_log_at_trx_commit 설정이 0, 2인 경우 즉시 디스코로 동기화되지 않을 수 있으나중요한 데이터 경우 1로 설정해 즉시 디스크로 동기화시키는 것을 권장


- 어댑티브 해시 인덱스(AHI)
    - 정의 : InnoDB 스토리지 엔진에서 사용자가 자주 요청하는 데이터에 대해 자동으로 생성하는 인덱스, 사용자는 innodb_adaptive_hash_index 시스템 변수로 활성화, 비활성화 결정 가능
    - 목적 : B-Tree 검색 시간을 줄여주기 위함
    - 일반적인 B-Tree 인덱스 보다 빠른 이유
        1. B-Tree와 해시 인덱스를 조합하여 검색 속도 향상
            - B-Tree 인덱스는 O(logN) 시간복잡도를 가짐
            - 해시 인덱스는 키 값 기반으로 O(1)의 검색 속도 제공
            - 어댑티브 해시 인덱스는 자주 조회되는 키 값을 감지하여 자동으로 해시 인덱스를 생성하므로 검색 속도를 크게 향상.
        2. 메모리 내에서 검색 처리 (디스크 I/O 감소)
            - 어댑티브 해시 인덱스는 InnoDB 버퍼 풀(Buffer Pool) 내에서 동작.
            - B-Tree 인덱스는 특정 데이터가 메모리에 없으면 디스크에서 읽어야 하지만, 어댑티브 해시 인덱스는 디스크 I/O 없이 즉시 검색 가능.
        3. 직접적인 키 값 검색으로 인덱스 탐색 과정 단축
            - B-Tree는 루트 노드 → 내부 노드 → 리프 노드 순으로 탐색해야 하지만,
            - AHI는 해시 테이블을 이용해 한 번에 원하는 데이터를 검색할 수 있어 탐색 과정이 단축됨.

      cf) 해시 인덱스 : Key(인덱스 키 값)과 Value(데이터 페이지 주소)의 쌍으로 관리

    - 어뎁티브 해시 인덱스 적용하기 좋은 환경
        1. 디스크의 데이터가 InnoDB 버퍼 풀 크기가 비슷한 경우(디스크 읽기가 많지 않은 경우) : 어댑티브 해시 인덱스는 InnoDB 버퍼 풀에서 관리되므로 자주 참조되는 데이터가 메모리에 유지될 수 있는 경우 가장 효과적
        2. 동등 조건 검색(동등 비교와 IN 연산자)이 많은 경우 : 해시 인덱스는 Key, Value쌍으로 관리되므로 정확한 키 값 검색에 유리함
        3. 쿼리가 데이터 중 일부 데이터에만 집중된 경우 : 데이터 검색 범위 넓지 않고 특정 데이터 반복적 검색 시 InnoDB가 감지해 자동으로 해시 인덱스 생성하고 효과적으로 작동함
    - 어뎁티브 해시 인덱스 적용하기 좋은 않은 환경
        1. 디스크 읽기가 많은 경우
        2. 특정 패턴의 쿼리가 많은 경우(조인이나 LIKE 패턴 검색)
        3. 매우 큰 데이터 가진 테이블의 레코드 폭넓게 읽는 경우
    - 테이블 삭제 작업에 어뎁티브 해시 인덱스의 영향력
        - 테이블의 인덱스가 어뎁티브 해시 인덱스에 적재된 상황에서 테이블 삭제(DROP), 변경(ALTER) 요청 시 모든 데이터 페이지 내용을 어뎁티브 해시 인덱스에서 제거 → 상당히 많은 CPU 자원 사용, 즉 어댑티브 해시 인덱스의 도움 증가할수록 테이블 삭제 또는 변경 작업에 치명적
    - CPU 사용량에 따른 어댑티브 인덱스의 적절한 비율(히트율)
        - 히트율 확인 방법 : SHOW ENGINE INNODB STATUS 명령으로 InnoDB 상태 조회 후 hash searches/s와 non-hash searches/s 값을 확인 → 히트율 (%) = (hash searches/s) / ((hash searches/s) + (non-hash searches/s)) * 100
        - CPU 사용량과 AHI 히트율의 관계 : 어뎁티브 해시 인덱스는 인덱스를 활용해 검색 성능 향상을 돕지만 추가적인 메모리와 CPU 자원이 필요 → 히트율이 높다면 CPU 사용량 감소하고 전체적인 성능 향상 반면, 히트율이 낮다면 오버헤드가 발생해 CPU 사용량이 증가할 수 있다.
        - 어뎁티브 해시 인덱스 도입 및 유지 결정 시 고려 사항
            1. 히트율이 높고 CPU 사용량이 낮은 경우: AHI가 효과적으로 작동하고 있으므로 유지하는 것이 바람직합니다.
            2. 히트율이 낮고 CPU 사용량이 높은 경우: AHI로 인한 오버헤드가 발생할 수 있으므로 비활성화를 고려해야 합니다.
            3. 히트율은 높지만 CPU 사용량이 높은 경우: AHI로 인한 메모리 사용이 과도하여 CPU 부하가 증가할 수 있으므로, 메모리 사용량을 확인하고 필요에 따라 AHI를 조정해야 합니다.

- InnoDB와 MyISAM, MEMEORY 스토리지 엔진
    - InnoDB vs MyISAM : MySQL 8.0이후부터 모든 기능이 InnoDB 스토리지 엔진만으로 구현할 수 있게 되며서 MyISAM이 없어질 것으로 예상 → MySQL 5.1, MySQL 5.5 버전이면 MyISAM, MEMORY 스토리지의 성능상 장점 기대 가능하지만 MySQL 8.0 이후부터 InnoDB와의 비교가 무색할만큼 비교 대상이 아니다.
    - InnoDB
        - 특징
            1. MySQL 8.0이후 기본 스토리지
            2. 트랜잭션 지원(ACID 보장)
            3. 클러스터리된 PK 인덱스 사용
            4. 외래키 지원
        - 장점
            1. 트랜잭션(ROLLBACK, COMMIT) 지원
            2. 충돌 시 데이터 복구 가능(Undo / Redo 로그)
            3. 동시성 처리 우수(MVCC 사용)
        - 단점
            1. INSERT 성능이 상대적으로 낮음
            2. 테이블 크기 증가 시 성능 저하 가능
    - MyISAM
        - 특징
            1. 비트랜잭션 스토리지 엔진
            2. 전체 테이블 잠금(Row-level Locking) 미지원
            3. 기본적으로 테이블이 독립적인 파일로 저장
        - 장점
            1. INSERT, SELECT 속도 빠름(특히 조회 성능 뛰어남)
            2. 데이터가 가벼움
        - 단점
            1. 데이터 무결성 취약(트랜잭션, 외래키 미지원
            2. 충돌 발생 시 복구 어려움
            3. 동시 쓰기 작업 취약(전체 테이블 잠금)
    - MEMORY
        - 특징
            1. 모든 데이터 메모리 저장
            2. 해시 인덱스 기반
            3. 트랜잭션 미지원
        - 장점
            1. 읽기, 쓰기 속도 매우 빠름
            2. 임시 테이블 용도로 적합
        - 단점
            1. 서버 재시작 시 데이터 유실(모든 데이터 메모리에 저장)
            2. 데이터 크기가 메모리 용량에 의해 제한
    - 선택기준
        1. InnoDB : 트랜잭션 필요 시
        2. MyISAM : 읽기 성능이 중요하고 트랜잭션이 필요 없으면
        3. MEMEORY : 임시 테이블이나 빠른 조회 필요 시

# 4.3 MyISAM 스토리지 엔진 아키텍처

- 키 캐시
    - 특징 : InnoDB 버퍼 풀과 비슷한 역할 But MyISAM의 키 캐시는 인덱스만을 대상으로 작동하고인덱스 디스크 쓰기 작업에 대해서 부분적 버퍼링 역할
    - 키 캐시 히트율(Hit rate) = 100 - (Key_reads / Key_read_requests * 100)

      cf) 히트율을 99% 이상으로 권장

- 운영체제의 캐시 및 버퍼
    - MyISAM 테이블 인덱스는 키 캐시 활용해 디스크 검색하지 않고 빠르게 검색 가능 지원
    - 디스크로부터 I/O 해결해주는 캐시나 버퍼링 기능 지원 X → 디스크 읽기, 쓰기는 OS의 디스크 읽기, 쓰기 작업으로 요청

      cf) OS의 캐시 읽기 : 남는 메모리 사용 따라서 전체 메모리를 애플리케이션 단에서 모두 사용하면 OS의 캐시 용도로 사용할 구간이 없어 캐시 읽기 수행 X → MyISAM 테이블의 데이터 캐시 못해 쿼리 처리 늦어짐

- 데이터 파일과 프라이머리 키(인덱스) 구조
    - InnoDB 사용하는 테이블 : PK로 클러스터링되어 저장
    - MyISAM 사용하는 테이블 : 클러스터링없이 데이터 파일이 힙(Heap) 공간처럼 활용 → PK 무관하게 INSERT 순으로 데이터 파일에 저장
    - ROWID : MyISAM 테이블에 저장되는 레코드의 물리적 주소값
        - 고정 길이 : 테이블 생성 시 MAX_ROWS 명시하는 경우로 최대로 가질 수 있는 레코드 한정, INSERT 순번 = ROWID
        - 가변 길이  : 테이블 생성 시 MAX_ROWS 명시하지 않는 경우, myisam_data_pointer_size 설정된 바이트 수만큼 ROWID 공간으로 사용

          cf) myisam_data_pointer_size : 기본값 7로 이 경우 2~7바이트까지 가변적인 ROWID 갖게됨, 첫번째 바이트는 ROWID 길이 저장 용도이고 나머지는 실제 ROWID를 저장하는 용도


# 4.4 MySQL 로그 파일

- 에러 로그 파일
    - 정의 : MySQL 실행 중 발생하는 에러나 경고 메시지 출력되는 로그 파일
    - 종류
        1. MySQL 시작 과정과 관련된 정보 및 에러 메시지
        2. MySQL 비정상 종료로 인한 InnoDB 트랜잭션 복구 메시지
        3. 쿼리 처리 중 발생하는 문제 관련 에러 메시지
        4. 비정상 종료된 커넥션 메시지
        5. InnoDB 모니터링 혹은 상태 조회 명령 결과 메시지
        6. MySQL 종료 메시지
- 제너럴 쿼리 로그 파일
    - 정의 : 시간 단위로 실행된 쿼리의 내용 기록하는 로그 파일, 슬로우 쿼리와 달리 쿼리 요청 시 바로 기록되기에 실행 중 에러 여부 고려않고 로그 파일에 기록
- 슬로우 쿼리 로그
    - 정의 : long_query_time 시스템 변수에 설정한 시간 이상의 시간이 소요된 쿼리가 모두 기록
    - 조건
        1. 반드시 쿼리가 정상적으로 실행 완료
        2. MySQL이 쿼리를 실행 한 후 실제 소요된 시간이 long_query_time 이상인 쿼리
    - log_output 옵션
        - TABLE : 제너럴 로그나 슬로우 쿼리 로그를 테이블(general_log와 slow_log)에 저장 → general_log와 slow_log 테이블은 CSV 스토리지 엔진 사용하기에 CSV 파일로 저장되는 것과 동일
        - FILE : 로그의 내용을 디스크의 파일로 저장
    - 슬로우 쿼리 내용
        - Time : 쿼리가 종료된 시점
        - User@Host : 쿼리를 실행한 사용자 계정
        - Query_time : 쿼리가 실행되는데 걸린 전체 시간
        - Rows_examined : 쿼리가 처리되기 위해 몇 건의 레코드에 접근했는지 의미

          cf) Rows_sent : 실제 몇 건의 처리 결과를 클라이언트로 보냈는지 의미

          ⇒ Rows_sent 표시된 레코드 건수가 상단히 적으면 튜닝해볼 가치 있다.