---
title: "API 응답이 JSON Array인 경우 Object 형태로 변경하기"
last_modified_at: 2020-02-15T17:08:00-05:00
header:
  show_overlay_excerpt: false
  overlay_image: /assets/images/post/2020/2020-02-15-option.jpeg
  og_image: /assets/images/posts/2020/2020-02-15-option.jpeg
  overlay_filter: 0.6
  caption: "Photo Credit: [Brady](https://kapentaz.github.io)"
tags:
  - API
  - JSON
category: #카테고리
  - JSON
toc: false
toc_label: "Table Of Contents"
toc_icon: "fal fa-list-alt"
toc_sticky: true
classes: wide
comments: true
---



자바 클래스 `List` 형태의 응답을 `Map`으로 제공받고 싶은데 현실적으로 어려운 경우 할 수 있는 방법에 관한 내용입니다.

> Java, jackson 라이브러리를 사용했습니다.

어떤 **상품들**을 조회하는 API가 있습니다. path는 `/products`이고 응답은 아래와 같이 JSON Array로 제공합니다.
```json
[
  {
    "productNo": 1,
    "name": "상품명1",
    "price": 100
  },
  {
    "productNo": 2,
    "name": "상품명2",
    "price": 200
  }
]
```
클라이언트는 이 JSON 응답을 보통 아래와 같이 `List` 객체로 변환해서 사용할 것입니다.
```java
List<Product> list = objectMapper.readValue(jsonString, 
	new TypeReference<List<Product>>() {});
```
만약 각 element를 하나씩 찾아서 어떤 처리를 해야 할 경우에 `List` 는 특정 element를 찾기 위해서는 loop를 이용해서 하나씩 찾아야 하기 때문에 `Map`으로 변환해서 처리하게 효율적일 수 있습니다.

`Map`으로 변환해서 처리하게 되면 비즈니스 로직과 별 상관없는 코드가 생겨서 불편하게 보일 수 있습니다. 이러다 보면 애초에 응답을 아래와 같이 `Map` 형태로 제공해주면 좋지 않을까?라고 생각할 수 있습니다. 
```json
{
  "1": {
    "productNo": 1,
    "name": "상품명1",
    "price": 100
  },
  "2": {
    "productNo": 2,
    "name": "상품명2",
    "price": 200
  }
}
```
```java
Map<Integer, Product> map = objectMapper.readValue(jsonString,  
	new TypeReference<Map<Integer, Product>>() {});  
  
Product product1 = map.get(1);  
Product product2 = map.get(2);
```
하지만 사실 API를 제공하는 입장에서는 *상품들*을 요청한 것이고 Repository에 저장되어 있는 데이터를 List 형태로 조회하고 이를 JSON Array 형태의 응답으로 제공하는 것은 자연스러운 상황입니다. 클라이언트의 변경 요청을 쉽게 받아들이지 않을 수 있습니다. 그리고 이미 다른 곳에 제공하는 API라면 더욱 변경해서 제공하기 어려울 것입니다. 

이럴 경우에는 그냥 `@JsonCreator`와 `@JsonValue`를 이용해서 Response 모델을 만드는 걸 추천합니다. 
```java
public class ProductResponse {
    private final List<Product> list;

    @JsonCreator
    public ProductResponse(List<Product> list) {
        this.list = list;
    }

    @JsonValue
    public List<Product> getList() {
        return list;
    }

    public Map<Integer, Product> toMap() {
        return list.stream().collect(
                Collectors.toMap(Product::getProductNo, Function.identity()));
    }
}
```
위와 같이 클래스를 만들면 deserialize할 때는 `@JsonCreator`를 이용하고 serialize할 때는 `@JsonValue`를 사용하게 됩니다. `toMap()` 메서드도 추가해서 `List`를 `Map`으로 바꾸는 역할도 응답 클래스에서 처리할 수 있게 됩니다.

끝. 