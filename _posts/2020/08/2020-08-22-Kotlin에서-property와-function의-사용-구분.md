---
title: "Kotlin에서 property와 function의 사용 구분"
last_modified_at: 2020-08-22T17:22+00:00
header:
  show_overlay_excerpt: false
  overlay_image: /assets/images/post/2020/08/2020-08-22-title.jpg
  og_image: /assets/images/post/2020/08/2020-08-22-title.jpg
  overlay_filter: 0.6
  caption: "Photo by Shahadat Rahman on Unsplash"
  
tags:
  - Kotlin
category: #카테고리
  - Kotlin
toc: false
toc_label: "Table Of Contents"
toc_icon: "fal fa-list-alt"
toc_sticky: true
classes: wide
comments: true
---

kotlin을 이용해서 개발하고 있지만 최근에서야 property와 function의 차이점을 알게 된 것 같습니다. 어떻게 구분해서 사용하는 것인가? 대한 고민 없이 그냥 사용했었는데요. 그 차이에 대해서 고민한 내용입니다.

## property
property는 `var`나 `val`를 통해서 표현합니다. 먼저 **Product** 클래스를 만들어서 *price*, *discount* 그리고 *stock*을 정의해보겠습니다.
```kotlin
class Product(
    val price: Int,
    val discount: Int,
    var stock: Int
)
```

그리고 **Product**에서 *price*, *discount*를 이용해서 판매가(salePrice)를 가져오는 function을 추가해보겠습니다. 

```kotlin
class Product(
    val price: Int,
    val discount: Int,
    var stock: Int
) {
  fun getSalePrice(): Int {
    return price - discount
  }
}
```
이제 **Product** 객체를 생성해서 정의한 property와 function을 호출해보겠습니다.

```kotlin
fun main() {
  val product = Product(price = 1000, discount = 200, stock = 10)
  product.discount
  product.price
  product.stock
  product.getSalePrice()
}
```

property인 *discount*, *price*, *stock*은 접근하는 것이고 function인 *getSalePrice()*은 호출하는 것이라는 차이점이 보입니다.  

*getSalePrice()*도 다른 property 처럼 사용할 수 있는 방법이 있습니다. *salePrice* property를 만들고 속성 접근자(getter)를 재정의 하는 방법입니다.

```kotlin
class Product(
    val price: Int,
    val discount: Int,
    var stock: Int
) {
  val salePrice: Int
    get() = price - discount
}	

fun main() {
  val product = Product(price = 1000, discount = 200, stock = 10)
  product.discount
  product.price
  product.stock
  product.salePrice
}
```

이제 salePrice도 호출 방식이 아니라 속성에 접근하는 구조로 변경됐습니다. 

그런데 salePrice의 속성 접근자를 이용했기 때문에 salePrice에 접근하려고 할 때마다 값을 계산하게 됩니다. *price*와 *discount*는 `val`이기 때문에 최초 **Product**가 생성된 이후에는 변경될 일이 없습니다.

변경될 일이 없는 값을 매번 계산하는 것은 비효율적이기 때문에 *salePrice*에 바로 값을 셋팅하고 이후에는 다시 계산하지 않도록 변경하는 것이 좋겠습니다.

```kotlin
class Product(
    val price: Int,
    val discount: Int,
    var stock: Int
) {
  val salePrice = price - discount
}
```

## function

이번에는 재고를 차감하는 메소드(subtract)와 stock이 0일 경우에 매진을 나타내는 메소드(isSoldOut)을 만들어 보겠습니다.

```kotlin
class Product(
    val price: Int,
    val discount: Int,
    var stock: Int
) {
  
  fun isSoldOut(): Boolean {
    return stock == 0
  }
  
  fun subtract() {
    stock--
  }
}
```

그리고 **Product** 객체를 만들어서 재고를 차감하고 매진여부를 확인하는 코드를 만들어 보겠습니다.

```kotlin
fun main() {
  val product = Product(price = 1000, discount = 200, stock = 1)
  product.subtract()
  product.isSoldOut()
}
```

정상적으로 동작하는 코드이지만 *isSoldOut()* 은 매진 여부를 나타내는 것인데 앞서 살펴 본 것 처럼 property로 접근하는 방법이 더 자연스러울 것 같습니다. 한번 변경해보겠습니다.

```kotlin
class Product(
    val price: Int,
    val discount: Int,
    var stock: Int
) {
  val isSoldOut: Boolean
    get() = stock == 0
  
  fun subtract() {
    stock--
  }
  
}

fun main() {
  val product = Product(price = 1000, discount = 200, stock = 1)
  product.subtract()
  product.isSoldOut
}
```

salePrice처럼 isSoldOut를 property 호출방식으로 변경하는게 더 자연스러운 것 같습니다.

## property와 function 구분

property와 function으로 정의하고 호출해 보니 둘의 차이점이 보입니다. property는 **Product** 의 상태 값을 의미하고 function은 **Product** 의 행위를 의미합니다. 앞서 살펴본 매진여부는 **Product** 의 매진상태를 의미하기 때문에  *isSoldOut()* 보다  *isSoldOut* 으로 변경하는 것이 더 자연스럽게 느껴진 것 같습니다.

그동안 Java를 사용하면서 getter/setter에 익숙해져서 상태 값 접근이 필요할 때 function으로 정의하는 경우가 있었는데 재고 차감 같은 어떠한 행위를 하는 것이 아닌 경우에는 function 보다는 property로 정의해서 사용하는 것이 더 바람직할 것 같습니다.

끝.




## Reference
- [Properties and Fields](https://kotlinlang.org/docs/reference/properties.html)
- [Functions](https://kotlinlang.org/docs/reference/functions.html)