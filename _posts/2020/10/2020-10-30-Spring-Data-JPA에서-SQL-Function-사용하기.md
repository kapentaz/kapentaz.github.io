---
title: "Spring Data JPA에서 SQL Function 사용하기"
last_modified_at: 2020-08-22T17:22+00:00
header:
  show_overlay_excerpt: false
  overlay_image: /assets/images/post/2020/10/2020-10-30-title.jpg
  og_image: /assets/images/post/2020/10/2020-10-30-title.jpg
  overlay_filter: 0.6
  caption: "Photo by Luca Bravo on Unsplash"
  
tags:
  - JPA
category: #카테고리
  - JPA
toc: false
toc_label: "Table Of Contents"
toc_icon: "fal fa-list-alt"
toc_sticky: true
classes: wide
comments: true
---


JPA를 사용하면서 SQL function을 사용하고 싶은 경우가 생길 수가 있습니다. 예제를 통해서 필요한 상황과 해결 방법을 알아보겠습니다.

간단한 예제를 하나 만들어 보겠습니다. 게시물을 관리하는 테이블(article)이 있고 게시물에는 여러 tag를 설정할 수 있습니다.  태그는 별도의 활용 가치는 없고 단순 노출을 목적으로 한다면 간단하게 json 컬럼으로 여러 태그 정보를 관리할 수 있을 것입니다. 

```sql
create table article
(
    article_no int           not null auto_increment comment '일련번호',
    title      varchar(255)  not null comment '제목',
    contents   varchar(4000) not null comment '내용',
    tags       json          null comment '태그',
    created_at timestamp     not null default CURRENT_TIMESTAMP comment '등록일시',
    primary key (`article_no`),
    index article_idx (title, contents)
) engine = InnoDB default charset = utf8mb4 comment '게시물';
```
테이블 구조에 맞게 예제 데이터를 등록하고
```sql
insert into article(title, contents, tags, created_at) values
('제목1', '내용1', '["맛집", "가성비"]', now()),
('제목2', '내용2', '["힐링"]', now());
```

Entity를 만들어서 
```kotlin
@Entity
@Table(name = "article")
class Article {
  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  val articleNo: Int = 0

  var title: String = ""
    private set

  var contents: String = ""
    private set

  var tags: String? = null
    private set

  @CreationTimestamp
  val createdAt: LocalDateTime = LocalDateTime.MIN
}
```

간단하게 테스트 코드를 작성해서 실행하면 성공합니다.

```kotlin
@Test
fun testSelect() {
  val list = articleRepository.findAll()
  assert(list.size == 2)
}
```



## AttributeConverter

tags는 JSON Array 구조이기 때문에 entity에서 String 타입으로 관리하는 것보다 List<String>으로 정의하는 것이 더 효과적일 것입니다. AttributeConverter이용하면 이 문제를 해결할 수 있습니다.

tags가 DB에 저장될 때는 String으로, 객체로 만들 때는 List<String>으로 만들 수 있도록 AttributeConverter를 구현합니다.

```kotlin
class StringListConverter : AttributeConverter<List<String>?, String?> {
  private val mapper = ObjectMapper()

  override fun convertToDatabaseColumn(attribute: List<String>?): String? {
    return attribute?.let { mapper.writeValueAsString(it) }
  }

  override fun convertToEntityAttribute(dbData: String?): List<String>? {
    return mapper.readValue(dbData, jacksonTypeRef<List<String>>())
  }
}
```
String 타입으로 정의했던 tags도 List<String>으로 변경하고 @Convert을 설정합니다.

```kotlin
@Convert(converter = StringListConverter::class)
var tags: List<String>? = null
  private set
```

다시 테스트 코드를 실행해 보면 성공합니다.



## 타입 변경으로 인한 문제

아무 문제 없이 잘 사용하고 있던 어느 날, 게시물을 조회할 때 tag를 검색 조건에 포함하는 요구 사항이 발생합니다. tags는 AttributeConverter를 통해서 List<String> 타입으로 관리되고 있기 때문에 생각처럼 잘 동작할지 모르겠습니다.

몇 가지 케이스로 확인해보겠습니다.

### @Query

List<String> 타입으로 정의되어 있다 보니 JPQL에서 like 키워드로 검색할 수도 없습니다. 
IDE에서부터 미리 에러 표시가 노출됩니다. `Type mismatch: string type expected`

```kotlin
@Query("""
  select a
  from Article a 
  where a.tags like :tags
""")
fun findByTags(@Param("tags") tags: List<String>): List<Article>
```

### Specification

Specification도 equals, like 모두 동작 하지 않습니다.

```kotlin
@Test
fun testSelect() {
  val list = articleRepository.findAll { root, _, cb ->
    cb.like(root.get<String>(Article::tags.name), "맛집")
  }
}
```

### QueryDSL

queryDSL 역시 eq, contains 모두 동작하지 않습니다

```kotlin
override fun getArticle(tags: List<String>): List<Article> {
  return from(article)
      .where(BooleanBuilder().apply {
        this.and(article.tags.contains(tags))
      })
      .fetch()
}
```

tags의 타입이 LIst<String>로 되어 있기 때문에 like 검색을 할 수가 없는 구조가 됩니다. 아무래도 조회 방법을 변경하는 것으로 해결하기는 어려울 것 같습니다. 그럼 타입을 그냥 List<String>에서 String으로 그냥 다시 변경해야 할까요?

## MetadataBuilderContributor

MySQL의 `JSON_CONTAINS`함수를 JPA에서 사용할 수 있으면 좋을 것 같아 검색을 했더니, 이 문제를 해결할 수 있는 [좋은 글](https://vladmihalcea.com/hibernate-sql-function-jpql-criteria-api-query/)을 찾았습니다. Hibernate 5.2.18부터 MetadataBuilderContributor라는 인터페이스를 사용할 수 있고 이 것을 구현하면 SQL Function을 추가할 수 있습니다.

>Dialect에 추가하는 방법도 있지만, Dialect를 수정하거나 버전을 변경할 경우 같이 챙겨서 수정해야 하기에 추천하지 않는다고 합니다.

```kotlin
class SqlFunctionsMetadataBuilderContributor : MetadataBuilderContributor {
  override fun contribute(metadataBuilder: MetadataBuilder) {
    metadataBuilder.applySqlFunction("JSON_CONTAINS",
        StandardSQLFunction("JSON_CONTAINS", StandardBasicTypes.BOOLEAN))
  }
}
```
새로 만든 SqlFunctionsMetadataBuilderContributor를 Spring 설정 파일에 추가합니다.
```properties
spring.jpa.properties.hibernate.metadata_builder_contributor=com.example.SqlFunctionsMetadataBuilderContributor
```

## 해결 방법

SqlFunctionsMetadataBuilderContributor 추가로 이제 `JSON_CONTAINS`를 사용할 수 있게 됐습니다. 앞서 살펴 봤던 케이스별로 적용방법을 확인해보겠습니다.

### @Query

tags 파라미터에 json array 형태의 문자열을 전달하는 방법입니다. 

```kotlin
@Query("""
  select a
  from Article a
  where function('JSON_CONTAINS', a.tags, :tags) = true
""")
fun getArticle(@Param("tags") tags: String): List<Article>
```

위 예제는 파라미터를 String으로 전달하고 있습니다.
아무래도 List<String>으로 전달하고 싶을 것입니다. 이럴 경우에는 SpEL를 이용해서 ObjectMapper Bean을 사용하는 방법도 있습니다. 

```kotlin
@Query("""
  select a
  from Article a
  where function('JSON_CONTAINS', a.tags, :#{@objectMapper.writeValueAsString(#tags)}) = true
""")
fun getArticle3(@Param("tags") tags: List<String>): List<Article>
```

### Specification

간단하게 테스트 코드를 만들어서 실행해보면 성공합니다.

```kotlin
@Test
fun testSelect() {
  val mapper = ObjectMapper()
  val list = articleRepository.findAll { root, _, cb ->
    cb.conjunction().apply {
      expressions.add(
          cb.equal(
              cb.function(
                  "JSON_CONTAINS",
                  Boolean::class.java,
                  root.get<List<String>>(Article::tags.name),
                  cb.literal(mapper.writeValueAsString(listOf("힐링"))),
              ), true
          )
      )
    }
  }
}
```

### QueryDSL

QueryDSL도 역시 잘 됩니다.

```kotlin
override fun getArticle(tags: List<String>): List<Article> {
  return from(article)
      .where(BooleanBuilder().apply {
        this.and(Expressions.booleanTemplate("JSON_CONTAINS({0}, {1}) = true",
            article.tags, mapper.writeValueAsString(tags)))
      })
      .fetch()
}
```

## 결론

SQL Function이 필요하다면 JPA에서 Native Query 없이도 사용할 수 있습니다.

## Reference

- [The best way to map a projection query to a DTO (Data Transfer Object) with JPA and Hibernate](https://vladmihalcea.com/the-best-way-to-map-a-projection-query-to-a-dto-with-jpa-and-hibernate/)
- [How to use mysql “json_contains” with JPA Specification](https://stackoverflow.com/questions/57904266/how-to-use-mysql-json-contains-with-jpa-specification)