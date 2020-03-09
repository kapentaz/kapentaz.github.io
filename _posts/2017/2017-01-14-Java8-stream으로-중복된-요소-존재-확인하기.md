---
title: "Java8 stream으로 중복된 요소 존재 확인하기"
last_modified_at: 2020-03-03T01:00:00+00:00
header:
  show_overlay_excerpt: false
  overlay_image: /assets/images/post/2017/2017-01-14-duplicate.jpg
  og_image: /assets/images/post/2017/2017-01-14-duplicate.jpg
  overlay_filter: 0.6
  caption: "Photo by Obi Onyeador on Unsplash"
  
tags:
  - Java
  - Stream
category: #카테고리
  - Java
toc: false
toc_label: "Table Of Contents"
toc_icon: "fal fa-list-alt"
toc_sticky: true
classes: wide
comments: true
---



Java8 stream을 사용하여 중복된 요소가 있는지 확인하려면 `distinct()`를 호출한 후 `count()` 정보와 원본 리스트의 `size()`를 비교하여 알 수 있습니다. 

```java
public static void main(String[] args) {
    List<Integer> list = Arrays.asList(1, 2, 3, 4, 5, 1);

    boolean duplicated = list.stream()
            .distinct()
            .count() != list.size();

    System.out.println(duplicated);
}
```

리스트 요소의 특정한 필드 값을 기준으로 중복된 요소가 있는지 확인하려면 `map()`으로 메서드 레퍼런스를 전달하여 처리할 수 있습니다.

```java
@Getter
@RequiredArgsConstructor
public class Member {
    private final Long memberNo;
    private final String name;
}
```

```java
public static void main(String[] args) {
    List<Member> list = Arrays.asList(
            new Member(1L, "Tom"),
            new Member(2L, "Smith"),
            new Member(3L, "Tom"));

    boolean duplicated = list.stream()
            .map(Member::getName)
            .distinct()
            .count() != list.size();

    System.out.println(duplicated);
}
```

`count()` 정보와 원본 리스트의 `size()`를 비교를 하는 것은 list 전체를 대상으로 중복으로 제거한 후에 비교하는 방식이라서 대상이 많을 경우에는 비효율적입니다. 

다른 방법으로는 `HashSet`을 이용해서 처리할 수 있습니다. `add()`메서드를 통해서 중복된 요소가 존재할 경우 false를 리턴하기 때문에 중복이 존재할 경우 전체를 다 실행하지(정렬에 따라) 않고도 알 수 있습니다.  

```java
public static void main(String[] args) {
    List<Member> list = Arrays.asList(
            new Member(1L, "Tom"),
            new Member(2L, "Smith"),
            new Member(3L, "Tom"));

    boolean unique = list.stream()
            .map(Member::getName)
            .allMatch(new HashSet<>()::add);

    System.out.println(unique);
}
```

`allMatch()`에서 new를 통해서 생성한 `HashSet`이 매 호출마다 인스턴스를 생성할 거라고 생각될 수 있지만 실제로 컴파일된 결과를 보면 그렇지 않습니다. 한번 생성한 `HashSet`의 `add()` 메서드 레퍼런스를 전달하는 것을 확인할 수 있습니다.  

끝.