---
title: "MySQL Master-Slave 데이터의 등록시간 불일치 현상"
last_modified_at: 2019-06-12T15:49:13-05:00 # 마지막 수정일
header:
  overlay_image: /assets/images/post/2019/2019-06-12-mysql-replication.png
  og_image: /assets/images/post/2019/2019-06-12-mysql-replication.png
  overlay_filter: 0.6
  caption: "Photo Credit: [manhtuong.net](https://manhtuong.net)"
  
tags: # 태그(복수개 가능)
  - MySQL  
  - JPA
category: #카테고리
  - MySQL
toc: false                          # 목차(table of contents) 사용여부
toc_label: "Table Of Contents"      # 목차 Label 변경
toc_icon: "fal fa-list-alt"         # https://fontawesome.com 에서 찾으면됨
toc_sticky: true                    # 스크롤 내릴때 같이 내려가는 목차
classes: wide                       # 기본 본문 넓이가 작다. wide 추천
comments: true                      # 댓글 시스템 사용여부
---

> 직접 서비스를 운영하면서 겪은 MySQL 관련 문제를 공유합니다.

## MySQL Master-Slave 데이터의 등록시간 불일치 현상

제휴몰과 주문연동 테스트를 할 경우 대부분 운영환경에서 진행하고 있었습니다. 이렇게 테스트한 데이터는 운영 DB에 계속 남기 때문에, 테스트 데이터를 Master DB에서 delete 처리를 했습니다. 이후 Admin 페이지에서 확인한 결과, 분명히 삭제한 데이터가 계속해서 조회가 되고 있었습니다. "아니 어떻게?" 여러 가지 가능성을 점치다 한가지 떠오르는 것이 있었습니다. Read 용으로 사용하고 있는 Slave에 복제 지연이 발생하고 있는 게 아닐까? "이렇게 오랫동안?" 불가능하다고 생각했지만, Slave에 접속해서 데이터를 확인해보니 삭제됐어야 할 데이터가 멀쩡히 존재했습니다.

DBA에게 문의한 결과 Slave에서 실행된 delete 쿼리는 where 절의 모든 항목의 값이 일치하는지 확인 후 실행하는 구조인데, 등록일을 저장하는 DATETIME 칼럼의 데이터가 Master-Slave 간에 불일치하여 Slave에서는 정상적으로 delete 되지 않는 것으로 확인됐습니다. "어떻게 날짜가 다르게 등록되어 있을 수 있을까?"

우선 대상 테이블과 Entity를 확인해봤습니다. 등록 일시는 LocalDateTime 타입으로 정의되어 있었고, 동일한 타입으로 주문 일시가 있었습니다. 하지만, 주문 일시는 Master-Slave 간에 시간차 없이 제대로 저장되어 있었습니다. 다른 점은 주문 일시는 파라미터로 전달받아서 설정하는 정보이고, 등록일은 Spring Data Jpa의 Auditing 기능으로 처리하는 것이었습니다. 

```java
private LocalDateTime orderYmdt;	// 주문일시

@CreatedDate
private LocalDateTime registerYmdt;	// 등록일시
```

"무슨 차이가 있을까?" log4jdbc 통해서 확인한 insert 쿼리에도 특이점은 없었기 때문에 더욱 혼란스러웠습니다. 

```sql
INSERT INTO order (
    c_key, o_amt, o_key, o_url, order_ymdt, payment, register_ymdt
) VALUES (
    '90b15..c1xxx', 48990,'20180120001', 'http://...', '2018-01-20 14:12:53', 'PAYCO', '2018-10-12 15:55:36'
) 
```

결국 insert 하는 부분을 JPA에서 MyBatis로 바꾸고, 등록 일시 부분을 MySQL 함수 now()로 변경한 후 테스트를 진행했습니다. 확인 결과 Master-Slave 데이터가 오차 없이 동일했습니다. 그럼 이제 모두 MyBatis로 바꿔야 할까요? 근본적인 문제는 따로 있는 것 같아서 원인과 방법을 찾고 싶어졌습니다.



## MySQL 버그

여러 방법을 찾던중 DBA를 통해서 MySQL 버그 리포트 [URL](https://bugs.mysql.com/bug.php?id=74550)를 전달 받았습니다. 우리는 server-side prepared statements를 사용하고 있는데, 이럴 경우 temporal 타입의 millisecond 부분이 Master에서는 반올림되지만, Slave에서는 절삭되는 문제가 있다고 합니다. 소스코드 변경없이 JDBC 설정에서 `useServerPrepStmts=false`로 변경하면 문제해결이 가능하다고 했습니다. 코드변경이 필요없는 방법이라 좋지만, 왠지 prepared statements를 제대로 활용하지 못하는 방법인 것 같아서 망설여 졌습니다.

우선 왜 반올림 처리를 하는지 궁금했습니다. DATETIME 타입은 초단위까지 나타낸다고 생각했는데 [MySQL 5.6부터는 DATETIME에 microsecond(6digits)까지 설정이 가능](https://dev.mysql.com/doc/refman/5.6/en/fractional-seconds.html)하고 설정하지 않을 경우 기본이 DATETIME(0)이라고 합니다. 우리는 기본값인 DATETIME(0)이였습니다. Spring Data Jpa의 Auditing 통해서 생성되는 데이터는 `LocalDateTime.now()` 이고 nanoseconds가 포함된 정보였습니다. 로그를 다시 확인해봐야겠습니다.

```
...
PreparedStatement.setString(6, "PAYCO")
PreparedStatement.setTimestamp(7, 2018-10-12 15:55:32.348032)
...
```

setTimestamp를 통해서 microsecond까지 전달되고 있었네요. log4jdbc에서 보기 좋게 남긴 insert 로그에는 초 단위까지만 남아 있어서 미처 생각하지 못했습니다.  

이 버그는 현재 [수정 커밋](https://github.com/codership/mysql-wsrep/commit/6b3d07f3343a1fe7039cfc5fb8b6da092ccde793)되어 있으며, [MySQL 5.7.18](https://dev.mysql.com/doc/relnotes/mysql/5.7/en/news-5-7-18.html) 부터 적용되어 있습니다. 안타깝게도 **저희는 그 아래 버전의 MySQL을 사용**하고 있습니다.



## JDBC properties의 useServerPrepStmts 그리고 cachePrepStmts

useServerPrepStmts 옵션은 어떤 기능일까요? MySQL은 4.1부터 server-side prepared statements를 사용할 수 있게 됐습니다. 그 이전에는 client-side prepared statements만 시용할 수 있었는데, 이 것은 [제가 알고있는 prepared statements와 조금 달랐습니다](https://stackoverflow.com/questions/32286518/whats-the-difference-between-cacheprepstmts-and-useserverprepstmts-in-mysql-jdb/32645365#32645365). 그냥 실행할 SQL에 value 까지 모두 채워진 text로 서버에 전송됩니다. 그리고 서버에서 파싱, 최적화 같은 과정을 매번 실행됩니다.

`useServerPrepStmts=true`로 설정하면 server-side prepared statements를 사용할 수 있습니다. SQL 문장을 한 번만 서버로 전송하고, 이후 실행되는 execution은 Binary Protocol을 이용하게 됩니다. 실제 placeholder value는 client에서 설정되어 서버로 전송합니다. 

```sql
ps=conn.prepareStatement("select ?")
ps.setInt(1, 42)
ps.executeQuery()
ps.setInt(1, 43)
ps.executeQuery()

254 Prepare    select ?
254 Execute    select 42
254 Execute    select 43
```

기본값이 true가 아닌 이유는 [초기 버그와 하위 호환성](https://dev.mysql.com/doc/relnotes/connector-j/5.1/en/news-5-0-5.html#connector-j-5-0-5-feature) 문제로 보입니다. 

또 다른 옵션인 cachePrepStmts는 어떤 기능일까요?  `cachePrepStmts=true`로 설정할 경우 MySQL JDBC Driver는 서버에 있는 prepared statements를 사용합니다. 사용할 수 있는게 없다면 실패를 하게 되고, prepare 과정을 다시 시작하게 됩니다. (기본값은 false고, useServerPrepStmts를 true로 설정했다면, 함께 true로 설정하는 것이 좋을 것 같습니다.) 

아래 쿼리를 통해서 현재 prepared statements 상황을 조회할 수 있습니다. [참고](http://kwonnam.pe.kr/wiki/database/mysql/jdbc) 

```sql
SELECT * FROM information_schema.global_status
WHERE variable_name IN ('Com_stmt_prepare', 'Com_stmt_execute', 'Prepared_stmt_count');
```

* **Com_stmt_prepare** : Connection.preparedStatement() 호출 횟수
* **Com_stmt_execute** : prepared statements로 쿼리가 실행된 횟수 
* **Prepared_stmt_count** : 서버에 저장돼 있는 prepared statements 갯수. 



## DateTime의 microseconds 지원

MySQL 5.5까지는 DATETIME 타입은 초단위 까지만 처리가 가능했지만, MySQL 5.6 부터는 microseconds(6digits)까지 설정이 가능합니다. 설정하지 않을 경우 기본값인 DATETIME(0)으로 처리 됩니다. (기본값이 6이 아니고, 0인 이유는 하위 버전 호환성을 위해서 라고 합니다.) 여기서 중요한 것은 초 이하의 시간정보는 자릿수에 맞게 반올림 처리가 됩니다. 이 부분은 TIME, TIMESTAMP도 동일합니다.

```sql
CREATE TABLE test(datetime3 datetime(3));
INSERT INTO test(datetime3) VALUES ('2018-10-15 23:59:58.5005');
select * from test;

> 2018-10-15 23:59:58.501
```

이러한 **반올림처리에 대한 오류나 경고는 따로 발생하지 않으며 SQL 표준**이라고 합니다. 기본값인 DATETIME(0)을 사용중이기 때문에 날짜정보를 **등록할 때 초단위까지만 전송하지 않을 경우** 예상치 못한 **반올림 처리로 1초의 시간차가 발생할 수 있습니다**. JDBC의 sendFractionalSeconds 프로퍼티 설정을  false(디폴트 true)로 설정해서 문제를 해결할 수도 있지만 전체적으로 fractional seconds 사용할 수 없기 때문에 주의해서 사용해야 할 것 같습니다.



## Spring Data JPA auditing의 DateTimeProvider 변경

원인은 파악된 것 같습니다. 우리의 편리한 Auditing 기능은 포기할 수 없습니다. MySQL DATETIME(0)에서 반올림 현상이 발생하지 않도록 수정합시다.  `@CreatedDate`, `@LastModifiedDate` 애노테이션이 있는 곳에 날짜 객체를 생성하는 역할은 `DateTimeProvider` 구현체가 처리합니다. 초 단위 이하를 절삭하는 새로운 구현체를 하나 만듭니다.

```java
@Component
public class LocalDateTimeProvider implements DateTimeProvider {

    @Override
    public Optional<TemporalAccessor> getNow() {
        return Optional.of(LocalDateTime.now().truncatedTo(ChronoUnit.SECONDS));
    }

}
```

디폴트로 `CurrentDateTimeProvider` 클래스를 사용하기 때문에 새로 구현한 클래스를 사용하도록 설정 정보도 함께 수정합니다.

```java
@EnableJpaAuditing(dateTimeProviderRef = "localDateTimeProvider")
```

 이후 등록/수정시 초 단위까지만 전송되는 것을 확인할 수 있습니다. (더 정확히는 0 millisecond 로 전송됩니다.)

끝!

## Reference
 - [Server side prepared statements leads to potential off-by-second timestamp on sl](https://bugs.mysql.com/bug.php?id=74550)
 - [11.3.6 Fractional Seconds in Time Values](https://dev.mysql.com/doc/refman/5.6/en/fractional-seconds.html)
 - [BUG#25187670: BACKPORT BUG#19894382 TO 5.6 AND 5.7.](https://github.com/codership/mysql-wsrep/commit/6b3d07f3343a1fe7039cfc5fb8b6da092ccde793)
 - [Changes in MySQL 5.7.18 (2017-04-10, General Availability)](https://dev.mysql.com/doc/relnotes/mysql/5.7/en/news-5-7-18.html)
 - [What's the difference between cachePrepStmts and useServerPrepStmts in MySQL JDBC Driver](https://stackoverflow.com/questions/32286518/whats-the-difference-between-cacheprepstmts-and-useserverprepstmts-in-mysql-jdb/32645365#32645365)
 - [Functionality Added or Changed](https://dev.mysql.com/doc/relnotes/connector-j/5.1/en/news-5-0-5.html#connector-j-5-0-5-feature)
 - [MySQL JDBC](http://kwonnam.pe.kr/wiki/database/mysql/jdbc)
