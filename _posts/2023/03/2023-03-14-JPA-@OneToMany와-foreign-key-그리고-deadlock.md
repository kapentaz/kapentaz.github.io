---
title: "JPA @OneToMany와 foreign key 그리고 deadlock"
last_modified_at: 2023-03-05T20:09:00+00:00
header:
  show_overlay_excerpt: false
  overlay_image: /assets/images/post/2023/03/2023-03-14-title.jpg
  og_image: /assets/images/post/2023/03/2023-03-14-title.jpg
  overlay_filter: 0.6
  caption: "Unsplash의Georg Bommeli"
tags:
  - JPA
  - MySQL
category: #카테고리
  - JPA
  - MySQL
toc: false
toc_label: "Table Of Contents"
toc_icon: "fal fa-list-alt"
toc_sticky: true
classes: wide
comments: true
---

@OneToMany 관계와 foreign key를 사용하면서 Deallock이 발생할 수 있는 경우를 확인해 보고 해결 방법도 확인해 보겠습니다.

먼저 상품과 옵션의 테이블을 만들어보겠습니다. 상품과 옵션이 있고 이 둘은 1:N 관계로 옵션에는 상품번호를 foreign key로 설정합니다.

```sql
create table product
(
    product_no   int          not null auto_increment comment '상품번호',
    option_count int          not null default 0 comment '옵션수',
    primary key (`product_no`)
) engine = InnoDB
  default charset = utf8mb4 comment '상품';
```
```sql
create table product_option
(
    option_no  int          not null auto_increment comment '옵션번호',
    product_no int          not null comment '상품번호',
    primary key (`option_no`),
    constraint fk_product_option foreign key (product_no) references product (product_no)
) engine = InnoDB
  default charset = utf8mb4 comment '옵션';
```

이제 위에서 생성한 테이블 기준으로 Entity도 만듭니다. @OneToMany, @ManyToOne으로 양방향 관계를 설정하겠습니다. 

```java
@Entity
@Table(name = "product")
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private int productNo;

    @Column(name = "option_count")
    private int optionCount;

    @OneToMany(mappedBy = "product", cascade = CascadeType.PERSIST)
    private List<ProductOption> options = new ArrayList<>();

    public void addOption(ProductOption option) {
        this.optionCount = this.optionCount + 1;
        this.options.add(option);
    }

    public void increase(int optionCount) {
        this.optionCount = this.optionCount + optionCount;
    }
}
```
```java
@Entity
@Table(name = "product_option")
public class ProductOption {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private int optionNo;

    @ManyToOne(optional = false)
    @JoinColumn(name = "product_no")
    private Product product;

    public static ProductOption createBy(Product product) {
        ProductOption option = new ProductOption();
        option.product = product;
        return option;
    }
}
```

위에서 만든 Entity를 기존 상품을 조회해서 새로운 `ProductOption`을 추가하고 `Product`에는 옵션 수를 증가시키는 메소드를 하나 만들어 보겠습니다.
미리 등록해둔 상품번호 1에 해당하는 데이터를 조회하고 addOption()을 통해서 옵션을 등록하는 간단한 메서드입니다.

```java
@Transactional
public void addOption() {
    Product product = productRepository.findById(1).orElseThrow();
    product.addOption(ProductOption.createBy(product));
}
```

## Deadlock 발생 재현
데드락이 발생하는 상황을 확인하기 위해 2개의 스레드로 위에서 만든 addOption() 메서드를 실행하는 간단한 코드를 작성해보겠습니다.
```java
public static void main(String[] args) {
    ConfigurableApplicationContext context = SpringApplication.run(DeadlockApplication.class, args);
    ProductService productService = context.getBean(ProductService.class);
    ExecutorService executorService = Executors.newFixedThreadPool(2);

    while (true) {
        Future<?> submit1 = executorService.submit(productService::addOption);
        Future<?> submit2 = executorService.submit(productService::addOption);
        try {
            submit1.get();
            submit2.get();
        } catch (Exception e) {
            executorService.shutdown();
            throw new RuntimeException(e);
        }
    }
}
```
실행하면 결과는 어떻게 될까요? 제 경우에는 실행과 동시에 바로 Deadlock이 발생하는 것을 확인할 수 있었습니다.  
```
com.mysql.cj.jdbc.exceptions.MySQLTransactionRollbackException: Deadlock found when trying to get lock; try restarting transaction
	at com.mysql.cj.jdbc.exceptions.SQLError.createSQLException(SQLError.java:123) ~[mysql-connector-j-8.0.32.jar:8.0.32]
	at com.mysql.cj.jdbc.exceptions.SQLExceptionsMapping.translateException(SQLExceptionsMapping.java:122) ~[mysql-connector-j-8.0.32.jar:8.0.32]
	at com.mysql.cj.jdbc.ClientPreparedStatement.executeInternal(ClientPreparedStatement.java:916) ~[mysql-connector-j-8.0.32.jar:8.0.32]
	at com.mysql.cj.jdbc.ClientPreparedStatement.executeUpdateInternal(ClientPreparedStatement.java:1061) ~[mysql-connector-j-8.0.32.jar:8.0.32]
	at com.mysql.cj.jdbc.ClientPreparedStatement.executeUpdateInternal(ClientPreparedStatement.java:1009) ~[mysql-connector-j-8.0.32.jar:8.0.32]
	at com.mysql.cj.jdbc.ClientPreparedStatement.executeLargeUpdate(ClientPreparedStatement.java:1320) ~[mysql-connector-j-8.0.32.jar:8.0.32]
	at com.mysql.cj.jdbc.ClientPreparedStatement.executeUpdate(ClientPreparedStatement.java:994) ~[mysql-connector-j-8.0.32.jar:8.0.32]
```

## Deadlock이 발생한 이유

### Deadlock 발생 로그
"(1) TRANSACTION"의 "(1) WAITING FOR THIS LOCK TO BE GRANTED" 부분을 보면 product 테이블 레코드에 X락을 얻기 위해 대기중인 것을 확인할 수 있습니다.
"(2) TRANSACTION"의 "(2) "HOLDS THE LOCK(S)"를 보면 product 테이블 레코드에 S락을 걸고 있고 "(2) WAITING FOR THIS LOCK TO BE GRANTED"를 보면 product 테이블에 다시 X락을 얻기 위해 대기중인 것을 확인할 수 있습니다.
``` 
------------------------
LATEST DETECTED DEADLOCK
------------------------
2023-03-13 23:50:17 0x40e6eb2700
*** (1) TRANSACTION:
TRANSACTION 104362, ACTIVE 0 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 5 lock struct(s), heap size 1136, 2 row lock(s), undo log entries 1
MySQL thread id 271, OS thread handle 278617589504, query id 12991 172.21.0.1 root updating
update product set option_count=1 where product_no=1
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 45 page no 3 n bits 72 index PRIMARY of table `deadlock`.`product` trx id 104362 lock_mode X locks rec but not gap waiting
Record lock, heap no 2 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
 0: len 4; hex 80000001; asc     ;;
 1: len 6; hex 0000000197a9; asc       ;;
 2: len 7; hex a70000011b0110; asc        ;;
 3: len 4; hex 80000000; asc     ;;

*** (2) TRANSACTION:
TRANSACTION 104363, ACTIVE 0 sec starting index read
mysql tables in use 1, locked 1
5 lock struct(s), heap size 1136, 2 row lock(s), undo log entries 1
MySQL thread id 270, OS thread handle 278752077568, query id 12992 172.21.0.1 root updating
update product set option_count=1 where product_no=1
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 45 page no 3 n bits 72 index PRIMARY of table `deadlock`.`product` trx id 104363 lock mode S locks rec but not gap
Record lock, heap no 2 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
 0: len 4; hex 80000001; asc     ;;
 1: len 6; hex 0000000197a9; asc       ;;
 2: len 7; hex a70000011b0110; asc        ;;
 3: len 4; hex 80000000; asc     ;;

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 45 page no 3 n bits 72 index PRIMARY of table `deadlock`.`product` trx id 104363 lock_mode X locks rec but not gap waiting
Record lock, heap no 2 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
 0: len 4; hex 80000001; asc     ;;
 1: len 6; hex 0000000197a9; asc       ;;
 2: len 7; hex a70000011b0110; asc        ;;
 3: len 4; hex 80000000; asc     ;;

*** WE ROLL BACK TRANSACTION (2)
------------
TRANSACTIONS
------------
```
product 테이블에 update를 하기위해 X락을 가지려고 하는건 알겠는데 S락을 가지고 대기하고 있는 이유는 뭘까요?

### 원인 확인
먼저 실행된 쿼리를들을 확인해 보겠습니다.

`Product` Entity를 조회하기 위해 select 쿼리가 실행되었고, ProductOption이 insert되고 옵션 수량을 증가시키기 위해 `Product`를 update하는 쿼리가 실행된 것을 확인할 수 있습니다.
```java
[pool-1-thread-1] jdbc.sqltiming : select p1_0.product_no,p1_0.option_count from product p1_0 where p1_0.product_no=1
[pool-1-thread-2] jdbc.sqltiming : select p1_0.product_no,p1_0.option_count from product p1_0 where p1_0.product_no=1
[pool-1-thread-1] jdbc.sqltiming : insert into product_option (product_no) values (1)
[pool-1-thread-2] jdbc.sqltiming : insert into product_option (product_no) values (1)
[pool-1-thread-2] jdbc.sqltiming : update product set option_count=1 where product_no=1
[pool-1-thread-1] jdbc.sqltiming : update product set option_count=1 where product_no=1 
```
`product_option` 테이블에 insert와  `product` 테이블을 update 하는데 Deadlock이 왜 발생할까요?
그 이유는 최초 테이블 생성시 **설정한 foreign key**와 관련이 있습니다.

foreign key를 설정하게 되면 MySQL은 부모나 자식 관계에 있는 테이블에 S락을 설정하게 됩니다. foreign key 관계에 있는 데이터가
제대로 존재하는지 확인하기 위해서이죠. 자세한 설명은 [공식문서](https://dev.mysql.com/doc/mysql-reslimits-excerpt/5.7/en/ansi-diff-foreign-keys.html)에 나와 있습니다.
> In an SQL statement that inserts, deletes, or updates many rows, foreign key constraints (like unique constraints) are checked row-by-row. 
> When performing foreign key checks, InnoDB sets shared row-level locks on child or parent records that it must examine.


## 해결방법1(분산락)
레디스를 이용해서 분산락을 설정해서 메서드를 동시에 실행하지 못하도록 막는 방법이 있습니다.
하지만, 분산락을 적용해야 하는 대상 메서드가 서로 다르고 통합하기 어려운 구조라면 적용하기 어려울 수 있습니다.
그리고 사실 위 상황에 대한 근본적인 해결책은 아닙니다.

## 해결방법2(순서변경)
foreign key로 인해 S락 설정이 먼저 적용되는 부분을 X락이 먼저 적용되도록 순서를 변경하면 됩니다.
다시 말해 insert와 update의 순서를 변경하면 자연스럽게 문제를 해결할 수 있습니다. 

JPA에서 ID가 IDENTITY 방식인 경우 쓰기 지연이 방식이 안 되기 때문에 insert보다 updaterk 먼저 실행될 수 있도록 flush()를 하면 됩니다.  

update를 먼저하게 될 경우 X락이 설정되기 때문에 foreign key로 인해 S락을 설정하려면 X락이 끝나기를 기다려야 하기 때문에
Deadlock이 발생하지 않습니다.

하지만 순서가 중요하다는 것을 모두가 인지하고 주의해야 하기 때문에 관리적 어려운 문제가 생길 수 있습니다.

## 해결방법3(foreign key 제거)
foreign key 자체를 제거하는 방법이 있습니다. 여러 제약들로 인해 실무에서는 foreign key를 설정하면서 사용하는 경우는 
그렇게 많지는 않습니다. foreign key 설정과 관계 없이 애플리케이션에 관계에 대한 관리가 잘 되어 있도록 개발하는게 좋습니다.

## 참고 
- [FOREIGN KEY Constraint Differences](https://dev.mysql.com/doc/mysql-reslimits-excerpt/5.7/en/ansi-diff-foreign-keys.html)
- [foreign key 에 의한 부모, 자식 테이블의 잠금관계](https://m.blog.naver.com/parkjy76/220639066476)
- [MySQL - 12장. 쿼리 종류별 잠금](https://happyer16.tistory.com/entry/MySQL-12%EC%9E%A5-%EC%BF%BC%EB%A6%AC-%EC%A2%85%EB%A5%98%EB%B3%84-%EC%9E%A0%EA%B8%88)

끝




