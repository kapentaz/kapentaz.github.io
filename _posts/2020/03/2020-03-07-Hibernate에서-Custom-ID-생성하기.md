---
title: "Hibernate에서 Custom ID 생성하기"
last_modified_at: 2020-03-07T13:38:00+00:00
header:
  show_overlay_excerpt: false
  overlay_image: /assets/images/post/2020/03/2020-03-07-head.jpg
  og_image: /assets/images/post/2020/03/2020-03-07-head.jpg
  overlay_filter: 0.6
  caption: "Photo by Carson Arias on Unsplash"
  
tags:
  - JPA
  - IdentifierGenerator
category: #카테고리
  - JPA
toc: false
toc_label: "Table Of Contents"
toc_icon: "fal fa-list-alt"
toc_sticky: true
classes: wide
comments: true
---



보통 테이블의 키값은 DB에서 제공하는 sequence나  auto increment 같은 증가 값 형태를 많이 사용하는데요. 특정 방식으로 키값을 관리해야 하는 경우도 있습니다. 예를 들면 uuid 형태나 주문번호같이 일정 패턴으로 생성하는 값을 키값으로 사용하는 경우가 있습니다. 

## UUID로 ID 생성

> uuid는 hibernate에서 기본적으로 제공하고 있습니다.  기본적으로  제공하는 다른 종류의 키는 DefaultIdentifierGeneratorFactory에서 확인할 수 있습니다.

기본적으로 제공하는 uuid를 이용해서 처리하는 방식을 먼저 확인해보겠습니다. 아래와 같이 entity 저장을 하면 길이 36의 문자 uuid 값으로 잘 생성되는 것을 확인할 수 있습니다.
```kotlin
@Entity
@Table(name = """"order"""")
class Order{

  @Id  
  @GeneratedValue(generator = "uuid")  
  @GenericGenerator(name = "uuid", strategy = "uuid2")  
  @Column(name = "order_no")  
  val orderNo: String = ""

  @Column(name = "order_amt")
  var orderAmt: BigDecimal = BigDecimal.ZERO
    private set

  companion object {
    fun create(orderAmt: BigDecimal): Order {
      return Order().apply {
        this.orderAmt = orderAmt
      }
    }
  }
}
```

## Custom ID 생성

custom ID를 생성하기 위해서 `IdentifierGenerator` 인터페이스를 이용해서  `generate()` 메서드를 구현합니다.  session를 이용해서 쿼리를 실행하는 형태로도 만들 수 있지만, 현재 시간 정보와 6자리 랜덤 문자를 결합한 형태의 주문번호를 사용한다고 가정하고 만들겠습니다.
```java
class OrderNoGenerator: IdentifierGenerator {
  override fun generate(session: SharedSessionContractImplementor,
                        entity: Any): Serializable {
    return System.currentTimeMillis().toString() +
        (1..999999).random().toString().padStart(6, '0')
  }
}
```

주문번호 생성을 위해 만든 `OrderNoGenerator`를 @Id 애노테이션이 있는 곳에 설정합니다. 
```java
@Id  
@GeneratedValue(generator = "orderNo")  
@GenericGenerator(name = "orderNo",   
 strategy = "com.order.entity.OrderNoGenerator")  
@Column(name = "order_no")  
val orderNo: String = ""
```

테스트 코드를 실행하면 '1583553989221246952'  형태로 값이 생성되는 것을 확인할 수 있습니다. ID가 아닌 곳에 값을 설정하고 싶다면 지난 포스트 [Hibernate에서 Custom Value 생성하기](https://kapentaz.github.io/jpa/Hibernate%EC%97%90%EC%84%9C-Custom-Value-%EC%83%9D%EC%84%B1%ED%95%98%EA%B8%B0/)를 참고하면 됩니다.

끝.



### Reference
- [https://docs.jboss.org/hibernate/orm/5.4/userguide/html_single/Hibernate_User_Guide.html#identifiers-generators-GenericGenerator](https://docs.jboss.org/hibernate/orm/5.4/userguide/html_single/Hibernate_User_Guide.html#identifiers-generators-GenericGenerator)