* MySQL의 격리 수준 4개 말해보시오, 각각 설명해보시오

  READ UNCOMMITTED - 다른 트랜잭션이 커밋하지 않은 변경 내용도 읽을 수 있음, Dirty Read가 발생할 수 있음.
  
  READ COMMITTED - 다른 트랜잭션이 커밋한 데이터만 읽을 수 있음, Non-repeatable Read는 발생 가능

  REPEATABLE READ - 트랜잭션 내에서 같은 쿼리는 항상 동일한 결과를 반환, Phantom Read 발생가능
  
  SERIALIZABLE - 가장 높은 격리 수준. 모든 트랜잭션을 순차적으로 실행한 것처럼 동작함

* Auto increment lock 이란 무엇인가요?

  Auto increment는 자동으로 증가하는 값을 추출하기 위해 사용, 여러 레코드가 INSERT되는 경우도 있기에 이를 테이블 수준의 lcok으로 관리함

* 글로벌 락에 대해 설명해주세요

  MySQL에서 제공하는 잠금 가운데 가장 범위가 큰 락이다. 다른 세션에서 SELECT를 제외한 대부분의 DDL 문장이나 DML 문장을 실행하는 경우 글로벌 락이 해제될 때까지 대기 상태로 남는다. 글로벌 락이 영향을 미치는 범위는 MySQL 서버 전체이다

* 트랜잭션을 사용하는 이유는 무엇인가요?

  트랜잭션은 작업의 완전성을 보장하기 위해, 데이터의 정합성을 보장하기 위해

* Partial Update란 무엇인가요?

  작업의 일부만 반영되고 나머지는 실패한 상태로 남는 현상
