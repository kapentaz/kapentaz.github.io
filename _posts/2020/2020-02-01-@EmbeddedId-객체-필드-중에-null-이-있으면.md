---
title: "@EmbeddedId 객체 필드 중에 null 이 있으면?"
last_modified_at: 2020-02-01T01:32:00-05:00
header:
  show_overlay_excerpt: false
  overlay_image: /assets/images/post/2020/2020-02-01-tree_and_guys.jpg
  og_image: /assets/images/posts/2020/2020-02-01-tree_and_guys.jpg
  overlay_filter: 0.6
  caption: "Photo Credit: [Brady](https://kapentaz.github.io)"
tags:
  - JPA
  - EmbeddedId
  - Embeddable
category: #카테고리
  - JPA
toc: false
toc_label: "Table Of Contents"
toc_icon: "fal fa-list-alt"
toc_sticky: true
classes: wide
comments: true
---


PK 컬럼이 not null인 것 처럼 JPA에서 @Id 애노테이션이 있는 필드도 not null 이어야 합니다. PK 컬럼이 여러개인 복합키인 경우에도 PK로 지정한 컬럼은 모두 not null 이어야 하는데요. JPA를 사용하면서 이부분에 대해 경험한 내용입니다.

> Kotlin, HIbernate, MySQL 환경으로 테스트했습니다.

아래와 같은 카테고리 테이블이 있다고 가정해보겠습니다. 계층적인 구조인 카테고리 정보를 하나의 row로 표현한 테이블입니다. `depth3_category_no`는 중복될 수 없는 값이기에 별도 자동 증가 컬럼 대신 PK로 지정되어 있습니다.

```sql
create table `category`
(
    `depth1_category_no`    int          not null comment 'depth1 카테고리번호',
    `depth1_category_name`  varchar(200) not null comment 'depth1 카테고리이름',
    `depth1_category_image` varchar(200) null comment 'depth1 카테고리이미지',
    `depth2_category_no`    int          not null comment 'depth2 카테고리번호',
    `depth2_category_name`  varchar(200) not null comment 'depth2 카테고리이름',
    `depth2_category_image` varchar(200) null comment 'depth2 카테고리이미지',
    `depth3_category_no`    int          not null comment 'depth3 카테고리번호',
    `depth3_category_name`  varchar(200) not null comment 'depth3 카테고리이름',
    `depth3_category_image` varchar(200) null comment 'depth3 카테고리이미지',
    `create_date`           datetime     not null comment '생성일시',
    PRIMARY KEY (`depth3_category_no`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8 comment '카테고리';
```
테이블 기준으로 샘플 데이터 4개를 insert합니다.
```sql
insert into category values (1, '패션', 'image1', 2, '여성의류', 'image2', 3, '바지', 'image3', now());
insert into category values (1, '패션', 'image1', 2, '여성의류', 'image2', 4, '티셔츠', 'image4', now());
insert into category values (1, '패션', null, 5, '남성의류', null, 6, '아우터', null, now());
insert into category values (1, '패션', null, 5, '남성의류', null, 7, '트레이닝복', null, now());
``` 
Entity 클래스도 같이 만듭니다.

```kotlin
@Entity
@Table(name = "category")
data class Category(
    @Column(name = "depth1_category_no")
    val depth1CategoryNo: Int = 0,

    @Column(name = "depth1_category_name")
    val depth1CategoryName: String,

    @Column(name = "depth1_category_image")
    val depth1CategoryImage: String? = null,

    @Column(name = "depth2_category_no")
    val depth2CategoryNo: Int = 0,

    @Column(name = "depth2_category_name")
    val depth2CategoryName: String,

    @Column(name = "depth2_category_image")
    val depth2CategoryImage: String? = null,

    @Id
    @Column(name = "depth3_category_no")
    val depth3CategoryNo: Int = 0,

    @Column(name = "depth3_category_name")
    val depth3CategoryName: String,

    @Column(name = "depth3_category_image")
    val depth3CategoryImage: String? = null,
    
    @Column(name = "create_date")
    val createDate: LocalDateTime = LocalDateTime.now()
)
 ```
Entity로 만들고 보니 보니 depth1 ~ depth3 까지 필드 구조가 중복된는것이 눈에 띕니다.  이 부분을 Embeddable 객체로 만들고 Entity를 변경하겠습니다.
```kotlin
@Embeddable
data class CategoryInfo(
    val categoryNo: Int = 0,
    val categoryName: String,
    val categoryImage: String? = null
): Serializable
```
```kotlin
@Entity
@Table(name = "category")
data class Category(
    @Embedded
    @AttributeOverrides(
        AttributeOverride(name = "categoryNo", column = Column(name = "depth1_category_no")),
        AttributeOverride(name = "categoryName", column = Column(name = "depth1_category_name")),
        AttributeOverride(name = "categoryImage", column = Column(name = "depth1_category_image"))
    )
    val depth1CategoryInfo: CategoryInfo,

    @Embedded
    @AttributeOverrides(
        AttributeOverride(name = "categoryNo", column = Column(name = "depth2_category_no")),
        AttributeOverride(name = "categoryName", column = Column(name = "depth2_category_name")),
        AttributeOverride(name = "categoryImage", column = Column(name = "depth2_category_image"))
    )
    val depth2CategoryInfo: CategoryInfo,

    @EmbeddedId
    @AttributeOverrides(
        AttributeOverride(name = "categoryNo", column = Column(name = "depth3_category_no")),
        AttributeOverride(name = "categoryName", column = Column(name = "depth3_category_name")),
        AttributeOverride(name = "categoryImage", column = Column(name = "depth3_category_image"))
    )
    val depth3CategoryInfo: CategoryInfo,

    @Column(name = "create_date")
    val createDate: LocalDateTime = LocalDateTime.now()
)
```
코드 라인수는 비슷하지만 중복되는 정보를 CategoryInfo 로 관리할 수 있게 됐습니다.

Embeddable 형태로 변경하면서 `depth3_category_no` 에 지정된 @Id는 더 이상 사용수 없기 때문에 @EmbeddedId를 설정 하도록 변경했습니다. 

변경된 구조로 데이터가 잘 조회되는지 테스트 코드를 아래와 같이 실행하면 실패를 하게 됩니다. collection에 null element 가 포함되어 있기 때문입니다.

```kotlin
@Test  
fun test() {  
  val list = categoryRepository.findAll()  
  assert(list.size == 4)                  // 성공  
  assert(list.filterNotNull().size == 4)  // 실패  
}
```

Hibernate의 소스코드를 따라 가서 `org.hibernate.type.ComponentType#hydrate` 확인하면 이유를 알 수 있습니다. 아래는 해당 메소드의 내용 일부 입니다. key 값일 경우 필드가 하나라도 null 이라면 null로 리턴 합니다. 다시 말하면 `CategoryInfo` 를 리턴해야 하는데 null로 리턴해버리는 것이죠.

```java
for ( int i = 0; i < propertySpan; i++ ) {
	int length = propertyTypes[i].getColumnSpan( session.getFactory() );
	String[] range = ArrayHelper.slice( names, begin, length ); //cache this
	Object val = propertyTypes[i].hydrate( rs, range, session, owner );
	if ( val == null ) {
		if ( isKey ) {
			return null; //different nullability rules for pk/fk
		}
	}
	...
}
```
이렇게 처리되다 보니 @EmbeddedId로 지정한 객체의 필드가 null이게 되니 영속성 객체로 인식할 기준값이 없는 것이고, 그러다 보니 null 을 반환하게 되는 것입니다. 
이제 선택 가능하 방법은
1. 자동증가 형태의 PK 컬럼 추가.
2. depth1 ~ depth2만 `CategoryInfo`로 변경
3. `CategoryInfo`를 제거하고 사용하지 않음

1번이 가장 좋은 방법일 것 같지만 여건이 되지 않는다면 2번 또는 3번으로 선택해야 할 것 같습니다.

끝.

## Reference
- [If Any field value of @EmbeddedId is null, which problem will arise?](https://stackoverflow.com/questions/52110754/if-any-field-value-of-embeddedid-is-null-which-problem-will-arise)