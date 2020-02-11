---
title: "Collectors.toMap은 NPE 주의가 필요하다"
last_modified_at: 2018-08-15T21:26:28-05:00
header:
  show_overlay_excerpt: false
  overlay_image: /assets/images/post/2018/blue_sky.jpg
  og_image: /assets/images/posts/2018/blue_sky.jpg
  overlay_filter: 0.6
  caption: "Photo Credit: [Brady](https://kapentaz.github.io)"
tags:
  - Java
category: #카테고리
  - Java
toc: false
toc_label: "Table Of Contents"
toc_icon: "fal fa-list-alt"
toc_sticky: true
classes: wide
comments: true
---



Java8에서 `Collectors.toMap()`을 사용할 경우 NPE 발생에 대한 주의가 필요합니다. 개발 및 테스트 환경에서는 이상이 없었는데 운영에서 오류가 발생한 경험에 대한 내용입니다.

아래와 같이 리스트 형태의 Item을 Map 형식으로 변경 하려고 할 경우 `toMap()`에서 NPE가 발생합니다.

```java
List<Item> items = Arrays.asList(
		new Item("Apple", 1800),
		new Item("Banana", 900),
		new Item("Grape", 3100),
		new Item("Mango", null));

Map<String, Integer> itemMap = items.stream()
		.collect(Collectors.toMap(Item::getName, Item::getPrice));	// NPE
```

`Collectors.toMap()`의 내용을 살펴보면 Java8에서 추가된 Map 인터페이스의 merge() 메소드를 사용하는데 HashMap의 merge() 에서는 value가 null인 경우를 하용하지 않습니다. HashMap 자체는 key와 value 모두 null을 허용하는데, 왜 merge()에서는 NPE를 발생시킬까요? 그 이유는 Map의 key와 value 중에서 value가 merge() 대상이인데, value가 null이면 merge를 할 수가 없기 때문에 NPE가 발생하게 됩니다.

이 문제를 해결하기 위해서는 stream의 filter를 통해서 value로 지정될 값이 null인 경우를 대상에서 제외하는 방법과

```java
Map<String, Integer> itemMap = items.stream()
		.filter(i -> Objects.nonNull(i.getPrice()))
		.collect(Collectors.toMap(Item::getName, Item::getPrice));
```

Collector를 직접 지정하는 방법이 있습니다.

```java
Map<String, Integer> itemMap = items.stream()
		.collect(HashMap::new, (m1, m2) -> m1.put(m2.getName(), m2.getPrice()), HashMap::putAll);
```

추가로 만약에 아래와 같이 "Apple"이라는 동일한 key가 존재할 경우에는 `IllegalStateException(Duplicate key)`이 발생하게 됩니다.

```java
List<Item> items = Arrays.asList(
		new Item("Apple", 1800),
		new Item("Banana", 900),
		new Item("Grape", 3100),
		new Item("Apple", 110));

Map<String, Integer> itemMap = items.stream()
		.collect(Collectors.toMap(Item::getName, Item::getPrice));	// 예외발생
```

 이것은 `BinaryOperator` 타입의 `mergeFunction` 을 지정하는 방법으로 해결할 수 있습니다. key가 중복될 경우 어떻게 처리할지를 전달하는 것이죠.

value가 Integer 타입이므로 sum()을 mergeFunction으로 지정했습니다. String 타입이라면 String::concat, List 타입이라면 List::addAll로 하거나 직접 구현할 수도 있다.

```java
Map<String, Integer> itemMap = items.stream()
				.collect(Collectors.toMap(Item::getName, Item::getPrice, Integer::sum));
```


## 결론
`Collectors.toMap()`을 사용할 때는 NPE가 발생할 수 있으므로 사용하기 전에 객체 내용중  value로 지정될 필드가 null일 가능성이 존재하는지 꼭 확인할 필요가 있습니다.
