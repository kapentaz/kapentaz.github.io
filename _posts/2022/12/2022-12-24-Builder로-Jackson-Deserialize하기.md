---
title: "Builder로 Jackson Deserialize하기"
last_modified_at: 2022-12-24T13:09:00+00:00
header:
  show_overlay_excerpt: false
  overlay_image: /assets/images/post/2022/12/2022-12-24-title.jpg
  og_image: /assets/images/post/2022/12/2022-12-24-title.jpg
  overlay_filter: 0.6
  caption: "Photo by Ralph (Ravi) Kayden on Unsplash"
  
tags:
  - Jackson
  - Builder
category: #카테고리
  - Java
toc: false
toc_label: "Table Of Contents"
toc_icon: "fal fa-list-alt"
toc_sticky: true
classes: wide
comments: true
---

개인적으로는 DTO 클래스가 필요할 때 불변 클래스로 만드는 것을 선호합니다.  
여러 장점이 있지만 그 중에서도 객체 값이 중간에 변경될 일이 없기 때문에 
다른 걱정이 없이 사용할 수 있는 장점이 특히 매력적인 것 같습니다.

불변 클래스로 만든 DTO를 Jackson에서 사용할 때 어떻게 해야할 지 확인해보겠습니다.

먼저 `PersonDto`라는 클래스를 만들어보겠습니다.
```java
@Getter
@ToString
@EqualsAndHashCode
public class PersonDto {
    private final String name;
    private final int age;

    @Builder
    private PersonDto(String name, int age) {
        this.name = name;
        this.age = age;
    }
}
```
`name`와 `age` 이렇게 2개의 final 필드가 있고, private 생성자도 하나 만들었습니다.
private으로 만든 이유는 @Builder 애노테이션으로 생성된 Builder 클래스로만 객체를 생성하도록 하기 위해서입니다.

## 불변 클래스로 Json 처리하기

이제 `PersonDto`를 json으로 serialize/deserialize하는 간단한 테스트코드를 만들어서 실행하겠습니다.
```java
@Test
void testJson() throws JsonProcessingException {
    PersonDto personDto = PersonDto.builder()
            .name("Tom")
            .age(18)
            .build();

    String json = objectMapper.writeValueAsString(personDto);
    log.debug("personDto Json:: {}", json);
    PersonDto dto = objectMapper.readValue(json, PersonDto.class);
    log.debug("personDto:: {}", dto);

    Assertions.assertEquals(personDto, dto);
}
```
테스트코드를 실행 해보면 아래 메시지와 함께 실패하게 됩니다.


```
Cannot construct instance of `com.example.person.PersonDto` (no Creators, like default constructor, exist): cannot deserialize from Object value (no delegate- or property-based Creator)
 at [Source: (String)"{"name":"Tom","age":18}"; line: 1, column: 2]
```
상세 내용을 보면 objectMapper의 readValue를 처리하는 곳에서 에러가 발생하는 것을 확인할 수 있습니다.
```java
PersonDto dto = objectMapper.readValue(json, PersonDto.class);
```
오류 메시지 내용을 보니 deserialize하기 위해서 기본 생성자를 추가하면 해결이 될 것 같습니다.
하지만, 기본 생성자를 만들게 되면 객체에 값을 생성하기 위한 setter 같은 메소드가 존재해야 하기 때문에 불변을 유지할 수 없게 됩니다.

불변은 포기할 수 없습니다. 다른 해결방법을 확인해 보겠습니다.


## 방법1. @JsonCreator와 @JsonProperty 사용하기
 
`PersonDto`생성자에 @JsonCreator와 @JsonProperty를 적용하면 위에서 만든 테스트 클래스가 성공합니다.
이 방법은 필드 이름을 문자열로 추가해야 하고, 필드가 변경될 때 잘 챙겨야 한다는 단점이 있습니다.

```java
@Getter
@ToString
@EqualsAndHashCode
public class PersonDto {
    private final String name;
    private final int age;

    @Builder
    private @JsonCreator PersonDto(
            @JsonProperty("name") String name,
            @JsonProperty("age") int age) {
        this.name = name;
        this.age = age;
    }
}
```
![성공1](https://raw.githubusercontent.com/kapentaz/kapentaz.github.io/master/assets/images/post/2022/12/2022-12-24-solution1.png)

## 방법2. @JsonDeserialize와 @JsonPOJOBuilder 사용하기

@JsonPOJOBuilder는 json과 필드 이름이 다른 경우에 사용하는데 @JsonDeserialize과 함께 사용하면 Builder로 deserialize할 수 있습니다.
방법1 보다는 좋지만, deserialize를 직접 정의해야 하는게 조금은 번거롭습니다.

```java
@Getter
@ToString
@EqualsAndHashCode
@JsonDeserialize(builder = PersonDto.PersonDtoBuilder.class)
public class PersonDto {
    private final String name;
    private final int age;

    @Builder
    private PersonDto(String name, int age) {
        this.name = name;
        this.age = age;
    }

    @JsonPOJOBuilder(withPrefix = "")
    public static class PersonDtoBuilder {
    }
}
```

## 방법3. @Jacksonized 사용하기

@Jacksonized는 Lombok **v1.18.14**에 추가되었습니다. [실험적](https://projectlombok.org/features/experimental/) 기능이긴 하지만,
@Jacksonized을 생성자에 사용하면 다른 방법보다 코드가 훨씬 간결해집니다.

@Jacksonized는 @Builder 추가 기능으로 deserialize시 빌더로 처리될 수 있도록 합니다.
사실 디컴파일 해보면 결국 방법2 처럼 @JsonDeserialize와 @JsonPOJOBuilder가 추가 되어 있는 것을 확인할 수 있습니다. 

```java
@Getter
@ToString
@EqualsAndHashCode
public class PersonDto {
    private final String name;
    private final int age;

    @Builder
    @Jacksonized
    private PersonDto(String name, int age) {
        this.name = name;
        this.age = age;
    }
}
```

## 결론
불변 클래스로 Json deserialize하는 방법을 살펴봤습니다. 
기본 생성자를 추가해서 불변을 포기하지 말고, 본인 환경에 맞는 방법을 찾아서 사용하면 좋을 것 같습니다.

끝.

## Reference
- [@Jacksonized](https://projectlombok.org/features/experimental/Jacksonized)
- [@JsonPOJOBuilder](https://www.baeldung.com/jackson-advanced-annotations)


