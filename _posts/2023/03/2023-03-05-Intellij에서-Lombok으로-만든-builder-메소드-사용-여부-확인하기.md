---
title: "Intellij에서 Lombok으로 만든 builder 메소드 사용 여부 확인하기"
last_modified_at: 2023-03-05T20:09:00+00:00
header:
  show_overlay_excerpt: false
  overlay_image: /assets/images/post/2023/03/2023-03-05-title.jpg
  og_image: /assets/images/post/2023/03/2023-03-05-title.jpg
  overlay_filter: 0.6
  caption: "Unsplash의Alvaro Pinot"
tags:
  - Java
  - IntelliJ
category: #카테고리
  - Java
  - IntelliJ
toc: false
toc_label: "Table Of Contents"
toc_icon: "fal fa-list-alt"
toc_sticky: true
classes: wide
comments: true
---

Lombok은 많은 편의 기능을 제공하기 때문에 거의 대부분의 자바 개발자들이 사용하고 있을 것입니다.
오늘은 Lombok 중에서도 `@Builder` 애노테이션으로 생성된 builder 메소드 사용 여부를 확인 방법에 대해서 확인해 보겠습니다.

여기 가격과 할인가격 정보를 가지고 있는 `Product` 클래스가 있습니다.
```java
@Getter
@Builder
public class Product {
    private final int price;
    private final int discountPrice;
}
```

클래스에 `@Builder` 애노테이션을 설정하면 Lombok이 생성자와 Builder 클래스를 함께 만들어 줍니다.
하지만 이 방법은 예상치 못한 오류를 발생시킬 수 있습니다. 자세한 내용은 [여기](https://kwonnam.pe.kr/wiki/java/lombok/pitfall)서 확인할 수 있습니다.

링크에 있는 주의점을 참고하여 
아래와 같이 생성자를 직접 생성하고 생성자에 `@Builder` 애노테이션을 적용하도록 변경했습니다.
생성자를 직접 사용하여 발생할 수 있는 문제도 차단하기 위해 private으로 정의했습니다.
```java
@Getter
public class Product {
    private final int price;
    private final int discountPrice;

    @Builder
    private Product(int price, int discountPrice) {
        this.price = price;
        this.discountPrice = discountPrice;
    }
}
```

`Product` 클래스에 있는 `builder()` 메소드를 호출하면 정상적으로 잘 되는 것을 확인할 수 있습니다.
```java
public static void main(String[] args) {
    Product product = Product.builder()
            .price(10_000)
            .discountPrice(1_000)
            .build();
}
```

오늘 확인 하려고 하는 것은 `@Builder` 애노테이션으로 생성된 메소드를 이 프로젝트 어디에서 사용하고 있는지 
확인하려면 어떻게 해야 할까? 입니다. 우리는 Product 클래스의 생성자를 직접 사용하고 있는 것이 아니기 때문에
생성자에서 Intellij의 `Find Usages(⌥F7)`를 사용하더라도 실제 사용하고 있는 곳을 찾을 수가 없습니다.

사용하고 있는 곳을 찾을 수 없다면 코드를 제거하거나 리팩토링하기가 어려워지게 됩니다.
이걸 해결할 있는 방법을 확인해 보겠습니다.

## 방법1
`@Builder` 애노테이션의 `builderMethodName`나  `buildMethodName` 속성을 정의해서 이름으로 찾기를 할 수 있습니다.
하지만 Intellij의 `Find Usages(⌥F7)`를 사용해서 찾는게 아니고 문자열로 찾기 때문에 정확하게 사용하는 곳을 찾는 방법은 아닙니다.

```java
@Getter
public class Product {
    private final int price;
    private final int discountPrice;

    @Builder(builderMethodName = "customBuilder", buildMethodName = "customBuild")
    private Product(int price, int discountPrice) {
        this.price = price;
        this.discountPrice = discountPrice;
    }
}
```
```java
public static void main(String[] args) {
    Product product = Product.customBuilder()
            .price(10_000)
            .discountPrice(1_000)
            .customBuild();
}
```
## 방법2
Intellij Structure에서 보면 소스코드에서는 보이지 않는 Lombok으로 생성된 builder() 메소를 확인할 수 있습니다.
여기서 `Find Usages(⌥F7)`를 사용하면 실제 사용하고 있는 곳을 찾을 수 있습니다.
![Find Usages](https://raw.githubusercontent.com/kapentaz/kapentaz.github.io/master/assets/images/post/2023/03/2023-03-05-structure.png)

끝




