---
title: "JPA Criteria로 관계 설정 없이 조인하기"
last_modified_at: 2020-06-02T23:22+00:00
header:
  show_overlay_excerpt: false
  overlay_image: /assets/images/post/2020/06/2020-06-02-title.jpg
  og_image: /assets/images/post/2020/06/2020-06-02-title.jpg
  overlay_filter: 0.6
  caption: "Photo by Irina Iriser on Unsplash"
  
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




JPA Criteria에서 Entity 관계를 설정한 경우에는 조인 쿼리를 통해서 Entity를 조회할 수 있습니다.  Order와 Member가 N:1로 설정되어 있을 때 JPA Criteria로 조회하는 방법을 확인해보겠습니다.
```kotlin
@Entity
@Table(name = "member")
data class Member(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "member_no")
    val memberNo: Int
) {
    @Column(name = "name")
    var name: String = ""
}
```

```kotlin
@Entity
@Table(name = "`order`")
data class Order(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "order_no")
    val orderNo: Int,

    @Column(name = "product_no")
    val productNo: Int,

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "member_no")
    val member: Member
)
```
```kotlin
val builder = entityManager.criteriaBuilder
val criteria: CriteriaQuery<Order> = builder.createQuery(Order::class.java)
val order: Root<Order> = criteria.from(Order::class.java)

// fetch join
order.fetch<Order, Member>(Order::member.name, JoinType.INNER)

val select = criteria.select(order)
val resultList = entityManager.createQuery(select).resultList
```

위 코드를 실행하면  Order와 Member를 영속성 객체로 한 번에 조회할 수 있습니다.

```sql
SELECT order0_.order_no   AS order_no1_3_0_, 
       member1_.member_no AS member_n1_2_1_, 
       order0_.member_no  AS member_n3_3_0_, 
       order0_.product_no AS product_2_3_0_, 
       member1_.name      AS name2_2_1_ 
FROM   ` order` order0_ 
       INNER JOIN member member1_ 
               ON order0_.member_no = member1_.member_no 
```

## 관계 설정하지 않은 경우

Order와 Member를 관계 설정하지 않아도 조인 쿼리를 만들 수 있습니다.
먼저 앞서 사용했던 Order Entity에서 Member와의 관계 설정을 제거하겠습니다.

```kotlin
@Entity
@Table(name = "`order`")
data class Order(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "order_no")
    val orderNo: Int,

    @Column(name = "product_no")
    val productNo: Int,

    @Column(name = "member_no")
    val memberNo: Int
)
```

그리고 아래 코드를 실행하면 조인 쿼리가 실행되는 것을 확인할 수 있습니다.

```kotlin
val builder = entityManager.criteriaBuilder
val criteria: CriteriaQuery<Order> = builder.createQuery(Order::class.java)

// order 와 member 조인
val member: Root<Member> = criteria.from(Member::class.java)
val order: Root<Order> = criteria.from(Order::class.java)

val select = criteria.select(order)

// Order.memberNo 와 Member.memberNo가 같은 경우
val predicate = builder.equal(
    member.get<Int>(Member::memberNo.name),
    order.get<Int>(Order::memberNo.name)
)
criteria.where(predicate)

val resultList = entityManager.createQuery(select).resultList
```

실제 실행된 SQL 쿼리를 확인해 보겠습니다.

```sql
SELECT order1_.order_no   AS order_no1_3_, 
       order1_.member_no  AS member_n2_3_, 
       order1_.product_no AS product_3_3_ 
FROM   member member0_ 
       CROSS JOIN ` order` order1_ 
WHERE  member0_.member_no = order1_.member_no 
```

INNER JOIN이나  OUTER JOIN이 아닌 CROSS JOIN으로 실행됩니다.  CROSS JOIN은 카티션 곱이라고도 하는데 양쪽 테이블의 모든을 행을 조인 시키기 때문에 의도한 것이 아니라면 필요한 데이터만 조인하기 위해서 조인 컬럼을 조건으로 추가해야 합니다.

```kotlin
val predicate = builder.equal(
    member.get<Int>(Member::memberNo.name),
    order.get<Int>(Order::memberNo.name)
)
criteria.where(predicate)
```

이렇게  관계 설정 없이 Order를 영속성 객체로 조회했지만, 아쉽게도 Member 까지는 영속성 객체로 조회할 수는 없습니다. 

Member를 영속성 객체로 조회할 수 없더라도 **Order를 조회하는 과정에 Member를 검색 조건으로 함께 사용해야 하는 경우**에는 유용할 수 있습니다.

그리고 영속성 객체는 아니더라도 `CriteriaBuilder`를 이용해서 array나 tuple을 이용해서 필요한 Member 컬럼을 Order와 함께 추가로 조회할 수는 있습니다. 관련 내용은 다음에 따로 정리하도록 하겠습니다.

끝.