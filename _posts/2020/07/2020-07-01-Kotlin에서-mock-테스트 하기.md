---
title: "Kotlin에서 mock 테스트 하기"
last_modified_at: 2020-07-01T17:22+00:00
header:
  show_overlay_excerpt: false
  overlay_image: /assets/images/post/2020/07/2020-07-01-title.jpg
  og_image: /assets/images/post/2020/07/2020-07-01-title.jpg
  overlay_filter: 0.6
  caption: "Photo by Angèle Kamp on Unsplash"
  
tags:
  - Test
  - Kotlin
category: #카테고리
  - Test
toc: false
toc_label: "Table Of Contents"
toc_icon: "fal fa-list-alt"
toc_sticky: true
classes: wide
comments: true
---

예전에는 테스트 코드를 작성하려고 해도 스스로도 익숙하지 않았고 테스트 코드 작성에 대해 호의적이지 않은 사람이 있거나 일정 압박으로 테스트 코드를 작성하다가도 중간에 포기한 적이 많았습니다.
 
지금 진행하는 프로젝트는 kotlin으로 개발하고 있으며 구성원 모두가 테스트 코드의 필요성을 느끼고 있어서 일정 압박 속에서도 열심히 테스트 코드를 작성하고 있습니다. 이번에는 중간에 포기하지 않고 오픈한 이후에도 계속해서 잘 관리할 것으로 기대됩니다.

## MockK

mocking 처리를 위해서 Java를 사용할 때는 [Mockito](https://site.mockito.org/)를 이용했었는데 Kotlin에서는 [MockK](https://mockk.io/)을 사용하고 있습니다. 공식 사이트에 가이드가 잘 나와 있기 때문에 여기에서는 기본 사용법만 확인해 보겠습니다.

## Mock 테스트

OrderService(주문)와 PaymentService(결제)가 있고 주문처리 중 결제를 호출하는데 이 코드를 테스트해보겠습니다.

```kotlin
interface PaymentService {  
  fun pay(): PayResult  
}
```

```kotlin
data class PayResult(  
    val codes: List<String>
)
```

```kotlin
class OrderService(  
    private val paymentService: PaymentService  
) {  
  fun order() {  
    paymentService.pay()  
  }  
}
```

주문 코드 테스트가 목적이기 때문에 PaymentService 호출 부분은 mock 객체로 생성하고 실행합니다.

```kotlin
@Test  
fun mockTest() {  
  // paymentService mock 객체 생성  
  val paymentService = mockk<PaymentService>()  
  // 생성된 mock을 이용해서 orderService 객체 생성  
  val orderService = OrderService(paymentService)  
  
  // order 실행  
  orderService.order()  
}
```
`order()`에서는 PaymentService mock 객체의 `pay()` 호출하게 되는데 mock 객체다 보니 어떤 결과를 반환 해야 할지 몰라서 오류가 발생하게 됩니다.
```
no answer found for: PaymentService(#1).pay()
io.mockk.MockKException: no answer found for: PaymentService(#1).pay()
```

이 에러는 `every {...}`를 통해서 `pay()`를 호출할 때 어떤 값을 반환할지 설정하면 해결할 수 있습니다. 
```kotlin
@Test  
fun mockTest() {  
  // paymentService mock 객체 생성  
  val paymentService = mockk<PaymentService>()  
  // 생성된 mock을 이용해서 orderService 객체 생성  
  val orderService = OrderService(paymentService)  
  
  // pay() 실행시 mock 객체 return  
  every { paymentService.pay() } returns mockk()  
  
  // order() 실행  
  orderService.order()  
  
  // pay() 메소드를 실행 했는지 호출(행위) 검증  
  verify { paymentService.pay() }  
}
```

## Relaxed mock 테스트

`every {...}`를 통해서 매번 mock 처리를 하는 것은 번거로울 수 있습니다. mock 대상이 많거나 특별히 확인할 내용이 없다면 더욱 그럴 것입니다. 이런 경우에 relaxed mock을 이용하는 게 좋습니다.

```kotlin
@Test  
fun mockTest() {  
  // paymentService relaxed mock 객체 생성  
  val paymentService = mockk<PaymentService>(relaxed = true)  
  // 생성된 mock을 이용해서 orderService 객체 생성  
  val orderService = OrderService(paymentService)  
  
  // order() 실행  
  orderService.order()  
  
  // pay() 메소드를 실행 했는지 호출(행위) 검증  
  verify { paymentService.pay() }
}
```
`every {...}` 코드가 없지만 오류도 발생하지 않고 `verify {...}`가 통과합니다. 

relaxed mock의 메소드를 호출하면 `0`, `false`, `""` 과 같은 기본값을 반환하고 참조 타입인 경우에는 다시 relaxed mock 객체를 반환합니다. 

## Stub
relaxed mock으로 모든 경우가 해결되지는 않습니다. `pay()` 호출로 payResult을 반환받고 그 결과 코드에는 무조건 "SUCCESS"가 있어야 하는 상황이라면 오류가 발생합니다.
```kotlin
class OrderService(  
    private val paymentService: PaymentService  
) {  
  fun order() {  
    val payResult = paymentService.pay()  
    payResult.codes.first { it == "SUCCESS"}  
  }  
}
```
"**SUCCESS**" 와 매칭되는 코드가 하나 이상 있어야 하지만 그렇지 않기 때문에 발생하는 오류입니다.
```
Collection contains no element matching the predicate.
java.util.NoSuchElementException: Collection contains no element matching the predicate.
```

이런 경우에는 stub으로 해결할 수 있습니다.

```kotlin
fun mockTest() {  
  // paymentService relaxed mock 객체 생성  
  val paymentService = mockk<PaymentService>(relaxed = true)  
  // 생성된 mock을 이용해서 orderService 객체 생성  
  val orderService = OrderService(paymentService)  

  // pay() 실행시 stub 객체 return    
  val payResult = PayResult(codes = listOf("SUCCESS"))
  every { paymentService.pay() } returns payResult  
  
  // order() 실행  
  orderService.order()  
  
  verify { paymentService.pay() }  
  assert(payResult.codes.first() == "SUCCESS")
}
```

mock과 stub의 다른점 중에 하나는 mock은 메서드(행위)에 대한 검증으로 `verify`를 사용하고 stub은 상태에 대한 검증으로 `assert`를 사용합니다.

## Spy 테스트

아래와 같은 Order 클래스의 `getAmountForPayment()` 메서드가 제대로 동작하는지 테스트해보겠습니다.
```kotlin
data class Order(
    val orderNo: Long,
    val orderAmount: Int,
    val deliveryFee: Int,
    val name: String,
    val address: String,
    val contact: String
) {
  fun getAmountForPayment(): Int {
    return orderAmount + deliveryFee
  }
}
```

Order를 mock 객체로 생성해서 테스트하면 성공적으로 실행됩니다.
```kotlin
@Test  
fun spyTest() {  
  val order = mockk<Order>()  
  
  every { order.getAmountForPayment() } returns mockk(relaxed = true)  
  
  order.getAmountForPayment()  
  
  verify { order.getAmountForPayment() }  
}
```
그런데 뭔가 이상합니다.  앞서 mock은 메서드(행위)를 테스트하기 위함이라고 했습니다. 메소드 실행 여부를 확인하는 것인데 `getAmountForPayment()`가 제대로 동작하는지 확인하는 용도로는 맞지 않은 것 같습니다.

그럼 이제 Order를 mock이 아니라 직접 생성해서 테스트해보겠습니다.
```kotlin
@Test
fun spyTest() {
  val order = Order(
      orderNo = 1,
      orderAmount = 19_800,
      deliveryFee = 2_500,
      name = "김주문",
      address = "서울특별시",
      contact = "010-0000-000"
  )

  val amountForPayment = order.getAmountForPayment()

  assert(19_800 + 2_500 == amountForPayment)
}
```
테스트 결과는 성공입니다. 

결과는 성공이지만, Order 객체를 직접 생성하면서 생성자 필수 값을 다 설정해야 하는 부분은 좀 불편한 것 같습니다. `getAmountForPayment()` 테스트에 필요하지 않은 값까지 설정해야 하기 때문입니다.

이런 경우에는 spy를 사용하는 게 좋습니다.

```kotlin
@Test
fun spyTest() {
  val order = mockk<Order> {
    every { orderAmount } returns 19_800
    every { deliveryFee } returns 2_500
  }

  val amountForPayment = spyk(order).getAmountForPayment()

  assert(19_800 + 2_500 == amountForPayment)
}
``` 
mock으로 생성한 Order 객체에 `getAmountForPayment()` 실행에 필요한 값만 설정하고 spy 객체로 변경하고 실행하면 원했던 대로 테스트 결과를 확인할 수 있습니다.

이처럼 spy는 mock과 실제 객체를 같이 사용하여 테스트하는 효과를 볼 수 있습니다.  

## Annotation

주문, 결제 mock 객체 생성을 직접 하지 않고 Annotation으로 할 수도 있습니다.

```kotlin
class OrderServiceTest {  
  
  @MockK  
  private lateinit var paymentService: PaymentService  
  
  @InjectMockKs  
  private lateinit var orderService: OrderService  
  
  @BeforeEach  
  fun setUp() {  
    MockKAnnotations.init(this, relaxed = true)  
  }  
  
  @Test  
  fun mockTest() {  
    // pay() 실행시 stub 객체 return    
    val payResult = PayResult(codes = listOf("SUCCESS"))
    every { paymentService.pay() } returns payResult  
  
    orderService.order()  
  
    verify { paymentService.pay() }  
  }  
}
```
junit5를 사용하고 있다면 MockKExtension를 이용해서 mock 초기 셋팅을 처리할 수 있습니다.
```kotlin
@ExtendWith(MockKExtension::class)  
class OrderServiceTest {  
  
  @MockK(relaxed = true)
  private lateinit var paymentService: PaymentService  
  
  @InjectMockKs  
  private lateinit var orderService: OrderService  
  
  @Test  
  fun mockTest() {  
    // pay() 실행시 stub 객체 return    
    val payResult = PayResult(codes = listOf("SUCCESS"))
    every { paymentService.pay() } returns payResult  
  
    orderService.order()  
  
    verify { paymentService.pay() }  
  }  
}
```

끝.

## Reference
- [https://mockk.io/](https://mockk.io/)

