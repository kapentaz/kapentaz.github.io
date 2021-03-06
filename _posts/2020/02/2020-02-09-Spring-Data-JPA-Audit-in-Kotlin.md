---
title: "Spring Data JPA Audit in Kotlin"
last_modified_at: 2020-02-09T10:16:00+00:00
header:
  show_overlay_excerpt: false
  overlay_image: /assets/images/post/2020/02/2020-02-09-wood_and_river.jpg
  og_image: /assets/images/posts/2020/02/2020-02-09-wood_and_river.jpg
  overlay_filter: 0.6
  caption: "Photo Credit: [Brady](https://kapentaz.github.io)"
tags:
  - JPA
  - Spring
  - Audit
  - Kotlin
category: #카테고리
  - JPA
  - Kotlin
toc: false
toc_label: "Table Of Contents"
toc_icon: "fal fa-list-alt"
toc_sticky: true
classes: wide
comments: true
---


Kotlin으로 Entity를 만들 때 등록일/수정일 정의를 어떻게 하는 게 좋을지 고민한 내용입니다.

> Spring Data JPA, Kotlin, MySQL 환경 입니다.

먼저 아래와 같이 테이블을 하나 만들겠습니다.
```sql
create table `ticket`
(
    `ticket_no` bigint auto_increment not null comment '티켓번호',
    `create_at` datetime              not null comment '등록일시',
    `update_at` datetime              not null comment '수정일시',
    PRIMARY KEY (`ticket_no`)
) ENGINE = InnoDB
  default CHARSET = utf8 comment '티켓';
```

## Spring Data JPA Audit
위 테이블을 Entity로 만들어보겠습니다. entity를 저장/수정할 때 생성일시/수정일시를 직접 설정하고 싶지 않기 때문에 Spring Data JPA Audit을 사용했습니다. 자세한 사용법은 [여기](https://docs.spring.io/spring-data/jpa/docs/1.7.0.DATAJPA-580-SNAPSHOT/reference/html/auditing.html)를 참고하세요.
```kotlin
@Entity
@Table(name = "ticket")
@EntityListeners(AuditingEntityListener::class)
class Ticket  {
  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  val ticketNo: Long = 0

  @CreatedDate
  var createAt: LocalDateTime = LocalDateTime.now()

  @LastModifiedDate
  var updateAt: LocalDateTime = LocalDateTime.now()
}
```
property의 초기값 지정이 필요해서 `LocalDateTime.now()`로 설정 했습니다. 하지만 Audit 설정으로 실제 DB에 insert/update 할 때 날짜 객체를 또 생성할 테니 객체를 2번 생성하는 꼴입니다. 초기값 설정을 하고 싶지 않다면 `lateinit` 키워드를 추가해서 초기값 생성과정을 제거할 수 있습니다.

```kotlin
@CreatedDate  
lateinit var createAt: LocalDateTime 
  
@LastModifiedDate  
lateinit var updateAt: LocalDateTime
```
이제 초기값 생성 문제는 사라졌지만,  `Ticket` 객체를 생성하고 Audit 처리가 되기 전에 createAt, updateAt에 접근하게 되면 runtime 오류가 발생하게 됩니다. 

```kotlin
kotlin.UninitializedPropertyAccessException: lateinit property createAt has not been initialized
```
실제 이렇게 호출하는 코드를 작성하지 않는다고 하더라도 호출할 수 있는 가능성과 호출 시 오류가 발생한다는 것은 잠재적 문제라고 할수 있을 것 같습니다.

{% include ad_content.html %}

그리고 createAt, updateAt을 `var` 로 정의했기 때문에 setter를 통해서 다른 날짜로 변경할 수도 있으니 `val`로 변경하는게 좋을 것 같습니다. `val`로 변경하게 되면 `lateinit`을 사용할 수 없으니 다시 초기값 설정이 필요하게 됩니다. setter를 없애기를 위해서는 초기값 설정을 꼭 해야할 것 같습니다. 

> CurrentDateTimeProvider#getNow 참고

Audit을 통해서 `LocalDateTime` 객체를 생성할 테니 중복 생성하지 않고 미리 상수로 정의되어 있는 `LocalDateTime.MIN`로 지정합니다.

```kotlin
@CreatedDate  
val createAt: LocalDateTime = LocalDateTime.MIN 
  
@LastModifiedDate  
val updateAt: LocalDateTime = LocalDateTime.MIN
```
이 상태로 실행하면 runtime 오류가 발생합니다. 이유는 `val` proeprty로 설정하게 되면 실제 Java Class로 바뀌면서 final 키워드가 붙게 되어 
Audit을 통해서 값을 변경할 수 없기 때문에 오류가 발생합니다.
```
java.lang.UnsupportedOperationException: No accessor to set property @org.springframework.data.annotation.CreatedDate()private final java.time.LocalDateTime .....
```

> MappingMetadataAuditableBeanWrapper
> #setDateProperty 참고

마지막으로 선택한 방법은 `var` 로 정의를 하고 setter를 외부에 공개하지 않는 것입니다.

```kotlin
@Entity
@Table(name = "ticket")
@EntityListeners(AuditingEntityListener::class)
class Ticket {
  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  val ticketNo: Long = 0

  @CreatedDate
  var createAt: LocalDateTime = LocalDateTime.MIN
    private set

  @LastModifiedDate
  var updateAt: LocalDateTime = LocalDateTime.MIN
    private set
}
```

`LocalDateTime.MIN`을 초기값으로 설정해서 불필요한 기본값 객체 생성을 방지하고 혹시라도 Audit이 실행되기 전에 해당 property에 접근하더라도 runtime 오류가 발생하지 않습니다.
`val`이 아닌 `var`로 정의했기 때문에 Audit을 통해서 값 설정이 가능하고 마지막으로 `private set`을 통해서 외부에서 값을 변경할 수 있는 가능성을 제거했습니다.


끝.

## Reference
- [Auditing](https://docs.spring.io/spring-data/jpa/docs/1.7.0.DATAJPA-580-SNAPSHOT/reference/html/auditing.html)
- [Properties and Fields](https://kotlinlang.org/docs/reference/properties.html)

