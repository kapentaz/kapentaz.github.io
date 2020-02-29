---
title: "Map에서 key와 value 바꾸기"
last_modified_at: 2020-02-20T22:13:00+09:00
header:
  show_overlay_excerpt: false
  overlay_image: /assets/images/post/2017/2017-09-03-snow_tree.jpg
  og_image: /assets/images/post/2017/2017-09-03-snow_tree.jpg
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


> 이전 블로그에서 작성한 글을 옮겨 왔습니다.

`Map`에서 key와 value의 위치를 서로 변경해야 할 일이 생길 수 있습니다. 어떻게 처리해야 할지 살펴보겠습니다. 기본적으로 `Map`은 중복된 key 값을 가질 수 없지만 value는 중복일 수 있습니다. 그래서 서로 위치를 변경하게 되면 value는 `List`로 처리해야 합니다. 

예를 들어서 확인해보겠습니다. 먼저 `Map`의 value에서 중복을 제거한 리스트를 이용해서 `Map.Entry`의 value가 일치하는 `Map.Entry`의 key를 List로 만들면 됩니다.
```java
public static void main(String[] args) {
    Map<String, String> map = new HashMap<>();
    map.put("order1", "apple");
    map.put("order2", "strawberry");
    map.put("order3", "apple");

    List<String> fruits = map.values().stream()
            .distinct()
            .collect(toList());

    Map<String, List<String>> collect = fruits.stream()
            .collect(toMap(Function.identity(), fruit -> map.entrySet().stream()
                    .filter(entry -> Objects.equals(fruit, entry.getValue()))
                    .map(Map.Entry::getKey)
                    .collect(toList())));
}
```
결과를 출력하면 아래와 같이 나옵니다.
```
apple=[order3, order1]
strawberry=[order2]
```

만약 value를 List로 처리하기 싫다면 `Collector.joining()` 를 이용하거나 직접 reduce() 처리를 하면 됩니다.
```java
Map<String, String> collect = fruits.stream()  
        .collect(toMap(Function.identity(), fruit -> map.entrySet().stream()  
                .filter(entry -> Objects.equals(fruit, entry.getValue()))  
                .map(Map.Entry::getKey)  
                .reduce("", (s1, s2) -> s1 + "," + s2)));
```

신경 써야 할 부분은 key는 중복될 수 없지만 value는 중복될 수 있으니 이 부분만 신경 쓰면 어떤 형태라도 key와 value를 바꾸는 것은 어렵지 않게 처리할 수 있습니다. 

끝.
