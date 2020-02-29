---
title: "Kotlin val property and nullable columns in JPA"
last_modified_at: 2020-02-10T23:21:00+09:00
header:
  show_overlay_excerpt: false
  overlay_image: /assets/images/post/2020/02/2020-02-10-grass_leaf.jpg
  og_image: /assets/images/posts/2020/02/2020-02-10-grass_leaf.jpg
  overlay_filter: 0.6
  caption: "Photo Credit: [Brady](https://kapentaz.github.io)"
tags:
  - JPA
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


기존 테이블 기준으로 Kotlin Entity를 생성 하려다 보면 property type을 nullable 과 not null 중에 고민하게 되는 상황이 있습니다. 왜 이런 상황이 생기는지 어떻게 해결하는지 좋을지 확인해보겠습니다.

예전부터 사용하던 Ticket 테이블이 있다고 해보겠습니다. `status`는 상태를 나타내는 컬럼으로 `UNUSED`, `USE`, 그리고 `EXPIRATION` 이렇게 3가지 문자열 값을 가질 수 있습니다. 모든 status는 3가지 중에 한가지 값을 무조건 가져야 할 것으로 보이지만, DB column 타입이 null로 되어 있습니다. 그러다 보니 3가지 값 말고 null 일 수도 있는 가능성이 생겼습니다. 실제 데이터가 null인 `status`도 있겠죠.
```sql
create table `ticket`
(
    `ticket_no` bigint auto_increment not null comment '티켓번호',
    `price`     int                   not null comment '가격',
    `status`    varchar(10)           null comment '상태',
    PRIMARY KEY (`ticket_no`)
) ENGINE = InnoDB
  default CHARSET = utf8 comment '티켓';
```
nullable로 설정한 정확인 이유는 모르지만 추측해본다면.. 
1. 새롭게 컬럼을 추가하면서 그냥 null로 지정.
2. null을 아무것도 선택하지 않았다는 의미로 사용.
3. 테이블의 사이즈가 너무 커서 not null과 default 값 설정하기 어려움.

1번인 경우는 패스하고 보통은 2번 아니면 3번의 이유일 것 같은데요. 

2번은 null을 어떤 의미 있는 값으로 사용하기 보다는 column을 not null로 지정하고 `상태없음`이라는 값을 만들어서 기본값으로 지정하는 게 더 좋았을 것 같습니다. null이 가질 수 있는 의미가 정확하게 무엇인지 알기 어렵거나 시간이 지나면 그 의미가 변하거나 다양해질 가능성이 있기 때문입니다.

3번 같은 경우에는 테이블에 사이즈가 크고 엑세스가 자주 발생한다면 신규 컬럼을 추가하면서 기본값까지 추가하면 업데이트가 발생하게 되니 작업하기 부담스러웠을 수 있습니다. 서비스 점검을 통해서만 처리할 수 있는 상황이라면 현실적 타협 통해서 null로 지정한 케이스 일 것 같습니다.

{% include ad_content.html %}

## var or val

사실 어떤 이유라고 하더라도 도메인 로직상으로는 `status`가 nullable 할 수 없는 상황이라고 한다면 테이블은 당장 바꾸지 않더라도 Entity를 만들 때는 `status` 타입을 not null 로 하는 게 맞을 것 같습니다.

DB 중심적인 개발을 하지 않는다면 보통은 개발한 코드를 훨씬 자주 만나게 될 텐데 코드에서 nullable 타입으로 지정하게 되면 이후에 개발하는 사람에게 불필요한 코드나 오해를 불러일으킬 수 있습니다.

column 타입과 상관없이 도메인 로직에서는 null 일수가 없기 때문에 아래와 같이 Entity를 만들었습니다.
 
> 변경할 수 있는 값들은 setter를 노출하지 않고 내부 method를 만들어서 처리 합니다. status 도 Enum으로 지정하는 것이 더 좋지만 예제에서는 String으로 했습니다.

```kotlin
@Entity
@Table(name = "ticket")
class Ticket(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val ticketNo: Long = 0
) {
  var price: Int = 0
    private set

  var status: String = "UNUSED"
    private set
}
```
이렇게 만들면 이후 생성하는 Ticket은 status가 not null 타입이기 때문에 null로 DB에 저장할 수 가 없습니다. 기존 데이터를 조회하는 것은 오류 없이 잘 되는 것으로 보이지만 실제로는 문제가 있습니다.

`status`가 null인 경우에도 JPA조회를 하면 객체는 정상적으로 생성됩니다. `status`가 not null 타입이라 믿고 사용하다는 결국 NPE를 만날수 있습니다.

이유는 `@Id` 위치에 따라 영속성 객체를 만들 때 field 또는 proeprty에 값을 설정하게 되는데 `@get:Id` 가 아니기 때문에 필드 주입방식으로 처리합니다. Java로 보면 JPA에서 실제 필드값을 null로 셋팅을 하기 때문에 Kotlin의 not null 타입이 소용이 없는 것입니다.

## 포기할 수 없는 Not Null 타입
실제는 nullable 할 수 없는 값이기 때문에 Not Null 타입은 포기할 수 없습니다.  아래처럼 변경해보겠습니다.
```kotlin
@Entity
@Table(name = "ticket")
class Ticket(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val ticketNo: Long = 0
) {
  var price: Int = 0
    private set

  var status: String = "UNUSED"
    private set
    get() = field.takeIf { it != null } ?: "UNUSED"
}
```
getter를 호출할 때 field 값이 null인지 확인하고 null일 경우 "UNUSED"를 반환하게 했습니다. IDE에서 경고 메시지가 나오네요. `Condition 'it != null' is always 'true'` stauts는 null 타입이 아니기에 항상 true이다 라는 건데요. 하지만 위에서 말했듯이 null인 상태로 조회가 될 수 있습니다. 

이대로 처리한다면 신규 데이터를 등록할 때도 기존 데이터를 조회해서 사용할 때도 문제없이 처리할 수 있습니다.

그리고 경고 메시지와 불필요한 코드가 있어 마음이 불편할 수 있습니다. 기존 데이터를 서비스 운영에 부담 없는 수준에서 단계적으로 null을 기본값으로 업데이트 처리하고 이후에 해당 코드를 제거하는 게 좋을 것 같습니다.

## 결론
도메인 영역에서 nullable과 not null을 명확하게 구분해서 사용하자.

끝.


## Reference
- [Properties and Fields](https://kotlinlang.org/docs/reference/properties.html)
