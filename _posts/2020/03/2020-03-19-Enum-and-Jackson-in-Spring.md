---
title: "Enum and Jackson in Spring"
last_modified_at: 2020-03-19T00:30:00+00:00
header:
  show_overlay_excerpt: false
  overlay_image: /assets/images/post/2020/03/2020-03-19-title.jpg
  og_image: /assets/images/post/2020/03/2020-03-19-title.jpg
  overlay_filter: 0.6
  caption: "Photo by Michael Weidner on Unsplash"
  
tags:
  - Java
  - Jackson
category: #카테고리
  - Java
  - Spring
toc: false
toc_label: "Table Of Contents"
toc_icon: "fal fa-list-alt"
toc_sticky: true
classes: wide
comments: true
---


지난번에 작성한 [**Enum and @RequestParam in Spring**](https://kapentaz.github.io/java/spring/Enum-and-@RequestParam-in-Spring/)에 이어서 이번에는 JSON으로 전달된 Request Body에서 Enum 타입을 처리하려면 어떻게 해야 하는지 확인해보겠습니다. 

## EnumDeserializer

Spring의 ConverterFactory는 Parameter를 대상으로 실행하기 때문에 JSON을 객체로 변환하는 deserialize 역할은 Jackson 라이브러리를 이용합니다. 그래서 Jackson의 `ObjectMapper`에서 Enum 변환 처리를 해줘야 합니다. 

다행히 Jackson에서는 기본으로 `EnumDeserializer`가 있고 이미 설정되어 있어서 Enum의 name으로 변환하는 것은 따로 신경 쓸 부분이 없습니다. 잘 되는지 확인 한번 해보겠습니다.

아래와 같은 JSON이 있고 type은 `NOTICE`, `NORMAL` 2가지가 있고 값이 있어서 Enum 클래스를 정의해서 request 모델을 만들겠습니다.
```json
{  
  "type": "NOTICE",  
  "title": "제목",  
  "content": "내용"  
}
```
```java
public enum Type {
    NOTICE,
    NORMAL
}

@Getter
@ToString
public class ArticleCreateRequest {
    private final Type type;
    private final String title;
    private final String content;

    @JsonCreator
    public ArticleCreateRequest(@JsonProperty("type") Type type,
                                @JsonProperty("title") String title,
                                @JsonProperty("content") String content) {
        this.type = type;
        this.title = title;
        this.content = content;
    }
}
```
이제 Controller를 만들고
```java
@Slf4j
@RestController
public class ArticleController {

    @PostMapping(value = "/articles")
    public ResponseEntity<Void> create(@RequestBody ArticleCreateRequest request) {
        log.info("create: {}", request);
        return ResponseEntity.ok().build();
    }
}
```
HTTP Request 파일을 만들어서 호출해보면 status가 200으로 잘 호출됩니다.

> HTTP Request 파일 관련 내용은 지난번 작성한 [**Http Request File in Intellij**](https://kapentaz.github.io/intellij/Http-Request-File-in-Intellij) 에서 확인할 수 있습니다.

```http
POST http://localhost:8080/articles
Content-Type: application/json

{
  "type": "NOTICE",
  "title": "제목",
  "content": "내용"
}

{% raw %}
> {% client.test("status", function() {
  client.assert(response.status === 200, "OK");
});
{% endraw %}
%}
```

## Custom Enum 

Enum의 name이 아니라 별도로 정의한 code 값으로 처리하고 싶을 때는 `@JsonCreator`와 `@JsonValue`를 이용하는 2가지 방법이 있습니다.

### @JsonCreator 
`@JsonCreator`를 static method에 설정하는 방법입니다. Enum은 JVM 내에서 유일한 객체로 만들어지기 때문에 Enum의 생성자에 `@JsonCreator`을 설정한다고 해도  이미 객체가 만들어진 이후이기 때문에 제대로 동작하지 않습니다.
```java
@Getter
@RequiredArgsConstructor
public enum Type implements Constant {
    NOTICE("TYPE001"),
    NORMAL("TYPE002");
    private final String code;

    @JsonCreator
    public static Type codeOf(String code) {
        return Arrays.stream(Type.values())
                .filter(t -> t.code.equals(code))
                .findAny()
                .orElseThrow(() -> new IllegalArgumentException(code  + " is illegal argument."));
    }
}
```

### @JsonValue

`@JsonValue` 가 적용되어 있으면 그 값으로 serialize/deserialize 모두 처리합니다.  Enum이 구현할 Interface를 하나 만들고 구현해야 할 메서드에 `@JsonValue` 을 설정하면 Interface를 구현한 모든 Enum이 동일하게 동작하게 됩니다.
```java
public interface Constant {
    @JsonValue
    String getCode();
}

@Getter
@RequiredArgsConstructor
public enum Type implements Constant {
    NOTICE("TYPE001"),
    NORMAL("TYPE002");
    private final String code;
}
```

```java
@Slf4j  
@RestController  
public class ArticleController {  
  
    @PostMapping(value = "/articles")  
    public ResponseEntity<ArticleCreateRequest> create(@RequestBody ArticleCreateRequest request) {  
        return ResponseEntity.ok(request);  
  }  
}
```

```http
POST http://localhost:8080/articles

HTTP/1.1 200 
Content-Type: application/json
Transfer-Encoding: chunked
Date: Sun, 15 Mar 2020 06:24:36 GMT
Keep-Alive: timeout=60
Connection: keep-alive

{
  "type": "TYPE001",
  "title": "제목",
  "content": "내용"
}
```

> Enum으로 deserialize 하는 상세 과정은 `BasicDeserializerFactory` 클래스의 createEnumDeserializer() 메서드에서 확인할 수 있습니다.


끝.