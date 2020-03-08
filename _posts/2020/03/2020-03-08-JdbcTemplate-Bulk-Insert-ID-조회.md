---
title: "JdbcTemplate Bulk Insert ID 조회"
last_modified_at: 2020-03-08T19:17:00+00:00
header:
  show_overlay_excerpt: false
  overlay_image: /assets/images/post/2020/03/2020-03-08-bulk.jpg
  og_image: /assets/images/post/2020/03/2020-03-08-bulk.jpg
  overlay_filter: 0.6
  caption: "Photo by Mildly Useful on Unsplash"
  
tags:
  - JDBC
  - Bulk
  - Spring
category: #카테고리
  - JDBC
toc: false
toc_label: "Table Of Contents"
toc_icon: "fal fa-list-alt"
toc_sticky: true
classes: wide
comments: true
---



이전에 작성했던 [JPA Batch Insert with MySQL](https://kapentaz.github.io/jpa/JPA-Batch-Insert-with-MySQL/) 글에서 JDBC Batch와 JdbcTemplate을 이용한 bulk insert 처리하는 방법에 대해 잠깐 얘기한 적이 있었습니다. 이번에는 batch insert 이후에 생성된 데이터의 키값을 가져오는 방법에 대해서 확인해보겠습니다.

> Kotlin, Spring, MySQL 환경입니다. 

## JDBC bulk Insert 키 조회
JDBC에서 bulk insert 후 키값을 조회하려면 PrepareStatement를 생성할 때 파라미터 `Statement.RETURN_GENERATED_KEYS` 을 추가하면 `ps.executeBatch()` 실행한 이후에 `getGeneratedKeys()` 메서드를 통해서 insert 한 키값을 확인할 수 있습니다.
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
    val ps = conn.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS)

    for (i in 1..10) {
      ps.setString(1, "상품명")
      ps.setInt(2, 5_000)
      ps.addBatch()
    }
    ps.executeBatch()

    val keys = ps.generatedKeys.use {
      generateSequence {
        if (it.next()) it.getLong(1) else null
      }.toList()
    }
  }
}
```
참고로 JDBC 스펙에서는 `getGeneratedKeys()`와 `executeBatch()`를 같이 사용하는 것에 대해 필수 구현 사항이 아니기 때문에 특정 DB나 구 버전에서는 지원하지 않을 수 있다고 합니다. 그래서 환경에 따라서 이 방법으로 키값을 조회 못할 수 있습니다.

{% include ad_content.html %}

## JdbcTemplate bulk Insert 키 조회

Spring JdbcTemplate은 bulk insert 한 키값을 조회할 수 있는 메서드를 제공하지 않습니다. 위에서 언급한 것처럼 클라이언트 환경에 따라 달라질 수 있기 때문입니다. 다른 방법으로는  `select last_insert_id()` 쿼리를 실행해서 마지막에 입력한 키값을 조회하는 것입니다.
```kotlin
fun batchInsert(products: List<Product>): List<Long> {
  val sql = """
        insert into product(
          name, price, create_date
        ) values (
          ?, ?, now()
        )
      """.trimIndent()

  return products.chunked(50).map {
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

    jdbcTemplate.queryForObject("select last_insert_id()", Long::class.java)?.let { productNo ->
      products.mapIndexed { index, _ -> productNo + index }
    } ?: emptyList()
  }.flatten()
}
```
`last_insert_id()`로 조회를 하면 bulk insert 한 첫 번째 row의 키값을 조회하기 때문에 등록한 수만큼 인덱스를 하나씩 더하면 전체 키값을 얻을 수 있습니다.

위 예제에는 문제가 하나 있는데  `last_insert_id()` 값은 DB 커넥션이 바뀌기 되면 전혀 다른 결과 값을 가져오게 됩니다. 이를 방지하기 위해서는  insert 문과 `last_insert_id()`을 같은 DB 커넥션을 사용할 수 있도록 Transaction으로 묶음 처리를 해야 합니다.

```kotlin
fun batchInsert(products: List<Product>): List<Long> {
  val sql = """
        insert into product(
          name, price, create_date
        ) values (
          ?, ?, now()
        )
      """.trimIndent()

  return products.chunked(50).map {
    transactionTemplate.execute { _ ->
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

      jdbcTemplate.queryForObject("select last_insert_id()", Long::class.java)?.let { productNo ->
        products.mapIndexed { index, _ -> productNo + index }
      }
    }
  }.filterNotNull().flatten()
}
```
`TransactionTemplate` 을 이용해서 트랜잭션을 설정했습니다. 이제는 DB 커넥션이 달라서 생기는 문제는 사라지게 됐습니다. Transaction을 batch insert 전체 또는 chunk 단위로 할지는 각 상황에 맞게 설정해서 사용하면 될 것 같습니다.

끝.

## Reference
- [Is there anyway to get the generated keys when using Spring JDBC batchUpdate?](https://stackoverflow.com/questions/2423815/is-there-anyway-to-get-the-generated-keys-when-using-spring-jdbc-batchupdate)
- [get Identity from sql batch insert via jdbctemplate.batchupdate](https://stackoverflow.com/questions/25300278/get-identity-from-sql-batch-insert-via-jdbctemplate-batchupdate)
- [spring jdbctemplate returns 0 for last_insert_id()](https://stackoverflow.com/questions/12236948/spring-jdbctemplate-returns-0-for-last-insert-id)
- [does last_insert_id() returns correct result when writes from multiple sessions?](https://dba.stackexchange.com/questions/31346/does-last-insert-id-returns-correct-result-when-writes-from-multiple-sessions)