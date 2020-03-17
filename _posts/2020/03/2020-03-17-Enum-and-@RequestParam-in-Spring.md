---
title: "Enum and @RequestParam in Spring"
last_modified_at: 2020-03-17T20:04:00+00:00
header:
  show_overlay_excerpt: false
  overlay_image: /assets/images/post/2020/03/2020-03-17-apples.jpg
  og_image: /assets/images/post/2020/03/2020-03-17-apples.jpg
  overlay_filter: 0.6
  caption: "Photo by twinsfisch on Unsplash"
  
tags:
  - Java
  - Spring
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


Spring에서 문자나 숫자로 전달된 Request Parameter를 Enum 타입으로 처리하는 방법과 custom conveter를 만드려면 어떻게 해야 하는지 확인해보겠습니다. 

## StringToEnumConverterFactory

데이터를 검색할 때 조건을 구분하기 위한 `SearchType` 이라는 Enum 클래스가 있습니다. "제목", "생성일" 그리고 "이름" 이렇게 3가지 종류의 검색 타입을 구분할 수 있도록 만들었습니다.
```java
public enum SearchType {
    TITLE,
    CREATED_AT,
    NAME
}
```
그리고 이 `SearchType` 과 `keyword`라는 파라미터로 검색어를 입력 받는 Controller를 만들었습니다. 
```java
@Slf4j
@RestController
public class EnumController {

    @GetMapping(value = "/search")
    public ResponseEntity<Void> search(@RequestParam(value = "searchType") String searchType,
                                       @RequestParam(value = "keyword") String keyword) {
        log.info("searchType: {}, keyword: {}", SearchType.valueOf(searchType), keyword);
        return ResponseEntity.ok().build();
    }
}
```
**`/search?searchType=NAME&keyword=이름`** 이렇게 호출 하면 전달한 파라미터 그대로 로그가 생기는걸 확인할 수 있습니다.
```
searchType: NAME, keyword: 이름
```
search() 메서드를 보면 searchType 파라미터를  `String` 타입으로 정의를 했습니다. 그리고 로그를 남기기 전에 `SearchType` 객체로 변경하는 작업을 직접 (`SearchType.valueOf(searchType)`) 처리 했습니다. 

파라미터로 전달받은 String을 Enum 객체로 매번 이렇게 바꾼다는 것은 매우 귀찮고 비효율적입니다. 다행히도 Spring에서는  `StringToEnumConverterFactory` 를 제공하고 있습니다. Spring 에서는 기본으로 사용할 수 있도록 설정해주고 있습니다. `@RequestParam` 으로 받는 파라미터 타입만 변경해 주면 됩니다.

```java
@GetMapping(value = "/search")
public ResponseEntity<Void> search(@RequestParam(value = "searchType") SearchType searchType,
                                   @RequestParam(value = "keyword") String keyword) {
    log.info("searchType: {}, keyword: {}", searchType, keyword);
    return ResponseEntity.ok().build();
}
```

## Custom ConverterFactory

Request Parameter를 Enum 클래스로 하고 싶은데 Enum의 이름이 아니라 별도로 정의한  code 값으로 Parameter를 전달받고 싶다면 Custom ConverterFactory를 직접 만들어서 사용할 수 있습니다.

먼저 `SearchType` 클래스에서 code 값을 가질 수 있도록 `Constant`라는 Interface를 하나 정의해서 구현할 수 있도록 하겠습니다.

> 직접 생성한 모든 Enum 클래스는 내부적으로 `java.lang.Enum`을 상속받고 있는 구조이기 때문에 다른 클래스를 상속할 수 없고 interface는 구현할 수 있습니다.
 
```java
public interface Constant {  
    String getCode();  
}

@Getter
@RequiredArgsConstructor
public enum SearchType implements Constant {
    TITLE("SEARCH001"),
    CREATED_AT("SEARCH002"),
    NAME("SEARCH003");
    private final String code;
}
```
그리고 `Constant` Interface 구현한 Enum을 대상으로 String을 Enum 객체로 변환할 수 있는 ConverterFactory를 만들겠습니다.
```java
public class CodeToEnumConverterFactory implements ConverterFactory<String, Enum<? extends Constant>> {

    @Override
    public <T extends Enum<? extends Constant>> Converter<String, T> getConverter(Class<T> targetType) {
        return new StringToEnumsConverter<>(targetType);
    }

    private static final class StringToEnumsConverter<T extends Enum<? extends Constant>> implements Converter<String, T> {

        private final Class<T> enumType;
        private final boolean constantEnum;

        public StringToEnumsConverter(Class<T> enumType) {
            this.enumType = enumType;
            this.constantEnum = Arrays.stream(enumType.getInterfaces()).anyMatch(i -> i == Constant.class);
        }

        @Override
        public T convert(String source) {
            if (source.isEmpty()) {
                return null;
            }

            T[] constants = enumType.getEnumConstants();
            for (T c : constants) {
                if (constantEnum) {
                    if (((Constant) c).getCode().equals(source.trim())) {
                        return c;
                    }
                } else {
                    if (c.name().equals(source.trim())) {
                        return c;
                    }
                }
            }
            return null;
        }
    }
}
```
> `Constant` Interface  구현 여부에 따라 분리해서 모든 Enum을 처리할 수 있는 형태로 만들었습니다. 변환 가능한 Enum을 찾지 못할 경우 null이나 Exception 처리는 본인의 환경에 맞춰서 처리하면 됩니다.

새로 만든 `CodeToEnumConverterFactory` 를 Spring에 등록합니다.
```java
@Configuration
public class WebConfig extends WebMvcConfigurationSupport {

    @Override
    protected void addFormatters(FormatterRegistry registry) {
        registry.addConverterFactory(new CodeToEnumConverterFactory());
    }
}
```
이제 **`/search?searchType=SEARCH001&keyword=이름`**  이렇게 다시 호출하면 새로 정의한 code로 호출하면 기존처럼 로그가 생성되는걸 확인할 수 있습니다.
```
searchType: TITLE, keyword: 이름
```

끝.