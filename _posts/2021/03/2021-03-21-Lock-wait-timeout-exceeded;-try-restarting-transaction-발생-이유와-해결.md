---
title: "Lock wait timeout exceeded; try restarting transaction 발생 이유와 해결"
last_modified_at: 2021-03-21T15:00+00:00
header:
  show_overlay_excerpt: false
  overlay_image: /assets/images/post/2021/03/2021-03-21-title.jpg
  og_image: /assets/images/post/2021/03/2021-03-21-title.jpg
  overlay_filter: 0.6
  caption: "Photo by Chepe Nicoli on Unsplash"
  
tags:
  - MySQL
category: #카테고리
  - MySQL
toc: false
toc_label: "Table Of Contents"
toc_icon: "fal fa-list-alt"
toc_sticky: true
classes: wide
comments: true
---


# Lock wait timeout exceeded; try restarting transaction 발생 이유와 해결

지난밤에 발생한 오류를 확인하던 중 메시지 하나가 눈에 띄었습니다.

> org.springframework.dao.CannotAcquireLockException ... Lock wait timeout exceeded; try restarting transaction

Lock을 얻지 못해 타임아웃이 발생했다는 내용입니다.

오류 메시지가 발생한 곳을 확인해 보니 데이터를 하나 삭제하는 기능이었습니다. 데이터를 삭제하기 위해 Lock이 필요했을 텐데요. 어디선가 먼저 Lock을 가지고 있어서 계속해서 대기하다가 타임아웃이 발생한 것으로 보였습니다.

Lock을 잡고 있는 트랜잭션이 어떤 것인지 확인하기 위해 검색하다가 [MySQL Lock 상황 문제 해결](https://www.popit.kr/mysql-lock-%EC%83%81%ED%99%A9-%EB%AC%B8%EC%A0%9C-%ED%95%B4%EA%B2%B0/)이라는 제목의 글을 통해 MySQL에서 Lock 정보를 확인하는 방법에 대해서 알 수 있었습니다.

쿼리를 통해서 확인할 수 있다고 되어 있었지만 Lock을 잡고 있던 트랜잭션은 이미 종료 되었고 복잡한 애플리케이션 구조로 인해 어디에서 발생 쿼리 문제인지 바로 파악하기 어려웠습니다. 또 이 문제가 반복적으로 계속 발생하는게 아니라서 확인하기 쉽지 않았습니다.

## 재현 코드
Lock을 확인할 수 있는 방법을 알게 되었지만, 아직 직접 확인하지 못했으니 간단하게 재현 코드를 만들어서 확인 하겠습니다.

심플한 테이블을 하나 만들고 Lock을 가질 수 있도록 데이터도 미리 등록합니다.
```sql
create table product_history
(
    history_no int       not null auto_increment comment '일련번호',
    product_no int       not null comment '상품번호',
    created_at timestamp not null default CURRENT_TIMESTAMP comment '등록일시',
    primary key (`history_no`)
) engine = InnoDB
  default charset = utf8mb4 comment '상품히스토리';  
```
```sql
insert into product_history(product_no) values  
(1), (2), (3), (4), (5), (6), (7), (8), (9), (10);
```


하나의 트랜잭션에서는 Lock을 잡고 있고 다른 트랜잭션에서는 Lock을 얻기 위해 대기하고 있는 상황을 재현하기 위해 코드를 작성 했습니다.
```kotlin
@Service
class DemoService(
    private val jdbcTemplate: JdbcTemplate,
    private val transactionTemplate: TransactionTemplate,
) {

  fun deleteAll() {
    Executors.newSingleThreadExecutor {
      Executors.defaultThreadFactory().newThread(it).apply {
        isDaemon = false
      }
    }.run {
      submit {
        transactionTemplate.execute {
          jdbcTemplate.update("delete from product_history")
	      // 테이블 전체 삭제 후 트랜잭션을 계속 유지하기 위해 60초 대기
          TimeUnit.SECONDS.sleep(60)
        }
      }
      shutdown()
    }
  }
  
  fun delete() {
    transactionTemplate.execute {
	  // 특정 데이터 하나 삭제
      jdbcTemplate.update("delete from product_history where product_no = 1")
    }
  }
}
```
테이블 전체 데이터를 삭제하는 `deleteAll()`를 먼저 실행하고 트랜잭션이 바로 commit되지 않도록 60초 대기하도록 합니다. 이후에 `delete()`를 실행해서 `deleteAll()`이 완료될 때까지 대기하는 상황을 코드로 만들었습니다.

```kotlin
@SpringBootApplication
class DemoApplication

fun main(args: Array<String>) {
  val application = runApplication<DemoApplication>(*args)

  application.getBean(DemoService::class.java).run {
    deleteAll()
    TimeUnit.SECONDS.sleep(1)	// delete가 먼저 실행되는 경우를 방지하기 위해 1초 딜레이
    delete()
  }
}
```

이제 main 메서드를 실행하면 어떻게 될까요?

`delete()`가 실행되고 50초 후에 **Lock wait timeout exceeded; try restarting transaction** 라는 메시지와 함께 오류가 발생합니다. MySQL의 [innodb_lock_wait_timeout](https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_lock_wait_timeout) 기본 설정이 50초로 되어 있기 때문입니다.

## 증상 확인

문제의 원인을 재현을 통해서 확인해 봤습니다. 역시 롱 트랜잭션이 문제입니다.

위 코드 기준으로 오류가 발생하기 50초 전까지 `INNODB_LOCK_WAITS` 테이블을 통해서 Lock을 얻기 위해 대기 중인 트랜잭션을 확인할 수 있고 `INNODB_TRX` 테이블을 통해 Lock을 가지고 오랫동안 실행 중인 트랜잭션을 찾을 수 있습니다.

```sql
> select * from information_schema.INNODB_LOCKS;		-- 현재 Lock 정보
> select * from information_schema.INNODB_LOCK_WAITS;	-- Lock을 대기 정보
> select * from information_schema.INNODB_TRX;			-- 트랜잭션 상태
```

찾은 `trx_id`의 실행중인 쿼리를 확인하고 싶을 경우에는 아래 쿼리를 이용하면 어떤 작업 때문에 시간이 오래 소요되는지 확인할 수 있습니다.

```sql
select  
  esh.event_name,  
  esh.sql_text  
from information_schema.INNODB_TRX trx  
    inner join information_schema.INNODB_LOCK_WAITS lw  
        on trx.trx_id = lw.blocking_trx_id  
  inner join performance_schema.threads th  
        on th.processlist_id = trx.trx_mysql_thread_id  
  inner join performance_schema.events_statements_history esh  
        on esh.thread_id = th.thread_id  
where trx.trx_id = :trx_id  
order by esh.event_id;
```

만약 롱트랜잭션이 처리되기 기다리기 보다 강제로 종료 시켜야 한다면 `INNODB_TRX` 테이블에서 `trx_mysql_thread_id` 컬럼에 있는 값을 이용해서 직접 프로세스를 종료시킬 수 있습니다.

```sql
SELECT * FROM information_schema.processlist where id = 175;  -- 프로세스 확인
kill 175;	-- 강제종료
```

## 결론

배치에서 롱 트랜잭션이 발생하는 경우는 자주 있는데 처음에는 데이터가 작아서 문제가 되지 않다가 시간이 지나면서 이렇게 문제 되는 경우가 종종 있습니다. 배치의 경우 트랜잭션을 지나치게 길지 않게 적절한 chunk 사이즈로 처리하는 것이 좋습니다.

끝.