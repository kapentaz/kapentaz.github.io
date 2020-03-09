---
title: "JPA Batch Insert with MySQL"
last_modified_at: 2019-11-14T22:01:00+00:00
header:
  show_overlay_excerpt: false
  overlay_image: /assets/images/post/2019/2019-11-14-light_in_dark_sky.jpg
  og_image: /assets/images/post/2019/2019-11-14-light_in_dark_sky.jpg
  overlay_filter: 0.6
  caption: "Photo Credit: [Brady](https://kapentaz.github.io)"
tags:
  - JPA
  - Bulk
  - MySQL  
  - JDBC  
category: #카테고리
  - JPA
toc: false                          # 목차(table of contents) 사용여부
toc_label: "Table Of Contents"      # 목차 Label 변경
toc_icon: "fal fa-list-alt"         # https://fontawesome.com 에서 찾으면됨
toc_sticky: true                    # 스크롤 내릴때 같이 내려가는 목차
classes: wide                       # 기본 본문 넓이가 작다. wide 추천
comments: true                      # 댓글 시스템 사용여부
---

JPA에서 Batch Insert가 되지 않아서 그 이유를 확인한 과정을 공유합니다. Spring, Kotlin, MySQL 환경기준으로 작성했습니다.

## JDBC Batch

JPA Batch Insert를 확인하기 전에 먼저 JDBC에서는 어떻게 처리해야 하는지 먼저 살펴보겠습니다.

대량의 데이터를 DB에 insert 할 때면 JDBC 배치를 사용해서 SQL 쿼리를 일괄 처리하는 것이 효율적일 수 있습니다. 간단하게 10,000건을 등록하는 예제 코드를 만들어 봤습니다.
```kotlin
fun batchInsert() {  
  val sql = """  
      insert into product( 
        name, price, create_date 
      ) values ( 
        ?, ?, now() 
      ) 
    """.trimIndent()  
  
  dataSource.connection.use { conn ->  
    val ps = conn.prepareStatement(sql)  
    for (i in 1..10_000) {  
      ps.setString(1, "상품명")  
      ps.setInt(2, 5_000)  
      ps.addBatch()  
    }  
    ps.executeBatch()  
  }  
}
```
> 테스트를 목적으로 단순하게 작성 했으며 고정된 값으로 설정 했습니다.

실행 후 로그를 확인하면   batch 모드로 잘 실행된 것을 확인할 수 있습니다.

```
Name:dataSource, Connection:2, Time:16885, Success:True
Type:Prepared, Batch:True, QuerySize:1, BatchSize:10000
Query:["insert into product(
  name, price, create_date
) values (
  ?, ?, now()
)"]
Params:[(상품명,5000)...생략]
```
그런데 아무리 10,000건 이라고 하지만, 실행시간이 16초라니 시간이 너무 오래 걸린 것 같습니다. 이상해서 MySQL 실행 로그를 확인해 보니 역시 이상합니다. multi value 방식으로 등록하는것이 아니라 insert 를 하나하나 실행한 형태입니다.
```sql
insert into product(
  name, price, create_date
) values (  
  '상품명', 5000, now()  
)  
insert into product(  
  name, price, create_date  
) values (  
  '상품명', 5000, now()  
)
...
```
왜 그럴까요?
### rewriteBatchedStatements
이 옵션 때문입니다. 이 옵션은 말 그대로 batch 형태의 SQL로 재작성 하는 것입니다. 기본값이 false 이기 때문에 multi value 형태로 실행하기 위해서는 변경이 필요합니다. JDBC URL에  `rewriteBatchedStatements=true` 옵션을 추가하고 다시 실행해보겠습니다.

```
Name:dataSource, Connection:2, Time:223, Success:True
Type:Prepared, Batch:True, QuerySize:1, BatchSize:10000
Query:["insert into product(
  name, price, create_date
) values (
  ?, ?, now()
)"]
Params:[(상품명,5000)...생략]
```

실행결과를 보면 0.2초가 소요 됐습니다. 굳!

코드 실행후 로그만 믿고 있다간 발등 찍힐 수 있으니 주의가 필요할 것 같습니다.

{% include ad_content.html %}

## JPA Batch Insert

이제 JPA Batch Insert에 대해서 알아 보겠습니다. 위에서 사용했던  Product 테이블 기준으로 Entity 클래스를 하나 만듭니다.
```kotlin
@Entity
@Table(name = "product")
data class Product(

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  @Column(name = "product_no")
  var productNo: Int = 0,

  @Column(name = "name")
  var name: String,

  @Column(name = "price")
  var price: Int = 0,

  @Column(name = "create_date")
  var createDate: LocalDateTime
)
```
그리고 간단하게 10,000건을 등록하는 테스트 코드를 작성해서 실행합니다.
```kotlin
fun batchInsert() {
  val products = IntRange(1, 10_000).map {
    Product(name = "상품명", price = 5_000, createDate = LocalDateTime.now())
  }
  productRepository.saveAll(products) 
}
```
결과는 20초 이상의 시간과 함께 batch 모드로 실행되지 않은 것을 확인할 수 있습니다.
```
718281727 nanoseconds spent preparing 10000 JDBC statements;
20366087525 nanoseconds spent executing 10000 JDBC statements;
0 nanoseconds spent executing 0 JDBC batches;
```
확인해보니 기본적으로 JPA Batch Insert가 비활성화되어 있기 때문에 옵션을 변경해야 한다고 합니다. Spring 설정 정보에 아래 내용 추가합니다
```properties
spring.jpa.properties.hibernate.order_inserts=true
spring.jpa.properties.hibernate.order_updates=true
spring.jpa.properties.hibernate.jdbc.batch_size=50
```
다시 테스트 코드를 실행해보면 잘 될 줄 알았는데 결과는 역시나 안됩니다.
```
704668858 nanoseconds spent preparing 10000 JDBC statements;
20382439004 nanoseconds spent executing 10000 JDBC statements;
0 nanoseconds spent executing 0 JDBC batches;
```
왜 그럴까요? 좀 더 확인 해봐야겠습니다.

### IDENTITY
검색  [결과](https://docs.jboss.org/hibernate/orm/5.4/userguide/html_single/Hibernate_User_Guide.html#batch-session-batch)에 따르면 GenerationType의 **IDENTITY**  타입 식별자를 사용할 경우 Hibernate가 JDBC 배치를 비활성화 시켜 버린다고 합니다. IDENTITY 방식이라면 안된다는 것이죠. 이유가 뭘까요?

간단합니다. entity를 persist 하려면 @Id로 지정한 필드에 값이 필요한데 IDENTITY(auto_increment) 타입은 실제 DB에 insert를 해야만 값을 얻을 수 있기 때문에 batch 처리가 되지 않는 것입니다. 
> 미리 @Id값을 설정할 수 있다면 잘 동작할 것입니다

GenerationType은 총 3가지가 있습니다.

* **IDENTITY**
* **SEQUENCE**
* **TABLE**

IDENTITY는 안된다고 하고, SEQUENCE는 MySQL에 없으니 확인해 볼 수 있는 건 TABLE뿐입니다.

{% include ad_content.html %}

### Table
별도 키 생성 테이블을 만들어서 사용하는 방법입니다. 시퀀스 방식을 테이블로 표현했다고 할 수 있습니다.
`spring.jpa.generate-ddl=true`  설정과 `GenerationType.TABLE`로 지정하면 JPA가 시퀀스 용도의 테이블 hibernate_sequences  을 자동으로 생성해주지만, `TableGenerator` 애노테이션을 통해서 직접 테이블을 설정할 수도 있습니다. 

>  운영 환경에서는 자동 생성보다는 sequence 테이블을 미리 생성하는 것이 좋을 것 같습니다.

```kotlin
@Id  
@GeneratedValue(strategy = GenerationType.TABLE, generator = "sequence_generator")  
@TableGenerator(name = "sequence_generator", table = "product_sequence",  
  pkColumnName = "sequence_name", pkColumnValue = "product_no", 
  initialValue = 1, allocationSize=50)  
@Column(name = "product_no")  
var productNo: Int = 0
```
`TableGenerator` 애노테이션의 속성 설명입니다.

* name: generator 이름
* table: 시퀀스 테이블 이름
* pkColumnName: 시퀀스 테이블의 pk 컬럼명
* pkColumnValue: 시퀀스 테이블의 pk 값
* initialValue: 초기값
* allocationSize: 증가값

이제 위에서 실행했던 10,000건 insert 코드를 다시 실행해보면, 정상적으로 batch 모드로 실행된 것을 확인할 수 있습니다. 

```
Name:dataSource, Connection:3, Time:754, Success:True
Type:Prepared, Batch:True, QuerySize:1, BatchSize:10000
Query:["insert into product (create_date, name, price, product_no) values (?, ?, ?, ?)"]
Params:[(2019-10-25 23:22:22.820885,상품명,5000,103)...생략]
```

#### AllocationSize

batch 모드로 잘 실행은 되었지만,  로그를 확인해보면 sequence를 가져오기 위한 쿼리가 추가로 실행되었는데, 해당 쿼리에 대해서 좀 더 알아보겠습니다. 

```sql
> select tbl.next_val from product_sequence tbl where tbl.sequence_name=? for update

> update product_sequence set next_val=?  where next_val=? and sequence_name=?
```

product_sequence 시퀀스 테이블에 다른 트랜잭션에서도 접근할 수 있기 때문에 **row-lock**을 위해 for update 문을 이용해서 조회하고, 설정한 allocationSize 값만큼 next_val을 증가시킵니다.

시퀀스 생성은 saveAll을 통해서 insert 하려는 list size만큼을 미리 생성하기 때문에 list size 보다 지나치게 작은 사이즈로 설정하는 것은 성능상 문제가 될 수 있습니다. 만약 1로 설정하고, batch insert 하려는 size가 10,000건이라면 총 10,000번의 시퀀스 생성을 시도하게 됩니다. 이렇게 비효율적으로 동작하지 않도록 적절한 사이즈를 설정할 필요가 있습니다.

> allocationSize의 기본값은 50입니다.

#### Performance

만약 product 테이블의 sequence를 위한 row-lock을 saveAll() 메소드가 끝날 때까지 유지하게 되면, saveAll()이 종료될 때까지 다른 트랜잭션에서 product 테이블 시퀀스 정보를 얻을 수 없는 문제가 있기 때문에   **sequence 생성은 별도 트랜잭션에서 처리하고, 바로 commit 하게 됩니다.** 

sequence를 얻기 위해 별도 트랜잭션에서 처리한다는 것은 sequence를 생성하는 동안은 추가 connection이 필요하다는 걸 의미합니다. 그렇기 때문에 connection pool이 부족한 애플리케이션이라면 connection을 얻기 위해 대기 시간이 더 발생할 수 있습니다.  (connection pool 사이즈를 1로 변경하고 테스트를 해보면 connection을 얻지 못해서 실패하는 것을 확인할 수 있습니다.)

row-lock으로 인해 동시성이 떨어지고, connection이 추가로 필요한 부분 등과 같은 이유로 TABLE 생성 방식을 권장하지 않는 글([Why you should never use the TABLE identifier generator with JPA and Hibernate](https://vladmihalcea.com/why-you-should-never-use-the-table-identifier-generator-with-jpa-and-hibernate/))도 있습니다. 그래도 각자 개발하는 환경과 요구사항이 다르기 때문에 본인 환경에 맞춰서 사용하면 좋을것 같습니다.

###  대안
Table 방식을 이용하기는 어렵고 batch insert가 필요한 상황이라면 MyBatis 또는 JOOQ등을 사용해서 해결할 수 있습니다. 만약 batch insert를 위해서만 해당 라이브러리 추가나 설정이 부담스럽다면 Spring JDBC이용해서 처리할 수도 있습니다.
```kotlin
fun batchInsert(products: List<Product>) {
  products.chunked(50).forEach {
    jdbcTemplate.batchUpdate(sql, object : BatchPreparedStatementSetter {
      override fun setValues(ps: PreparedStatement, i: Int) {
        val product = it[i]
        ps.setString(1, product.name)
        ps.setInt(2, product.price)
      }

      override fun getBatchSize(): Int {
        return it.size
      }
    })
  }
}
```

## 결론

MySQL에서 auto_increment로 테이블 키값을 관리하고, **IDENTITY 타입으로 키를 생성한다면, JPA Batch Insert는 불가능합니다.** Batch 처리가 꼭 필요하다면 GenerationType 타입을 Table으로 변경하거나, Spring JDBC, MyBatis 또는 JOOQ 등을 사용해야 할 것입니다.

끝.

## 참고

- [Performant Batch Inserts Using JDBC](https://dzone.com/articles/performant-batch-inserts-using-jdbc)
- [Session batching](https://docs.jboss.org/hibernate/orm/5.4/userguide/html_single/Hibernate_User_Guide.html#batch-session-batch)
- [Why you should never use the TABLE identifier generator with JPA and Hibernate](https://vladmihalcea.com/why-you-should-never-use-the-table-identifier-generator-with-jpa-and-hibernate/)
- [JPA - 기본 키 매핑 전략(@Id)](https://coding-start.tistory.com/m/71)
- [The best way to do batch processing with JPA and Hibernate](https://vladmihalcea.com/the-best-way-to-do-batch-processing-with-jpa-and-hibernate/)

 