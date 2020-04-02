---
title: "Spring Boot Bean Validation 제대로 알고 쓰자"
last_modified_at: 2020-04-03T00:03:00+00:00
header:
  show_overlay_excerpt: false
  overlay_image: /assets/images/post/2020/04/2020-04-03-title.jpg
  og_image: /assets/images/post/2020/04/2020-04-03-title.jpg
  overlay_filter: 0.6
  caption: "Photo by Trần Toàn on Unsplash"
  
tags:
  - Java
  - Spring
  - Validation
category: #카테고리
  - Spring
toc: false
toc_label: "Table Of Contents"
toc_icon: "fal fa-list-alt"
toc_sticky: true
classes: wide
comments: true
---


지난번에 작성한 [**Java Bean Validation 제대로 알고 쓰자**](https://kapentaz.github.io/java/Java-Bean-Validation-%EC%A0%9C%EB%8C%80%EB%A1%9C-%EC%95%8C%EA%B3%A0-%EC%93%B0%EC%9E%90/#)에 이어서 Spring Boot 환경에서 Validation을 어떻게 사용할 수 있는지 확인해보겠습니다. Spring에서도 Hibernate Validator를 사용합니다. Java Bean Validation에 대해서 잘 모른다면 지난 글을 먼저 읽어보는 것을 추천합니다.

## Dependency
Spring Boot에서는 `spring-boot-starter-validation`를 추가하면 Validation 관련 필요한 라이브러리가 모두 추가됩니다.
```groovy
implementation("org.springframework.boot:spring-boot-starter-validation")  
implementation("org.springframework.boot:spring-boot-starter-web")
```

## AutoConfiguration
Spring Boot `ValidationAutoConfiguration` 클래스를 통해서 `LocalValidatorFactoryBean`와 `MethodValidationPostProcessor`를 자동으로 설정합니다.

`LocalValidatorFactoryBean`는 Spring에서 Validator를 사용하기 위해서 필요하고 `MethodValidationPostProcessor`는 메서드 파라미터 또는 리턴 값을 검증하기 위해서 사용됩니다.

AutoConfiguration으로 별다른 설정 없이 Spring Boot에서 바로 Validation을 사용할 수 있습니다.

## @RequestPram and @PathVariable

메서드 파라미터나 리턴 값을 검증하기 위해서는 클래스에 `@Validated`을 적용하면 `MethodValidationPostProcessor`에 의해서 Validation이 가능하도록 프록시 객체가 생성됩니다.

Spring 내부에서 생성된 프록시 객체를 통해서 `@RquestParam`과 `@PathVariable`에 바로 Validation 애노테이션을 적용할 수 있습니다. 

```java
@Validated
@RestController
public class ProductRestController {

	@GetMapping(value = "/products")
	public ResponseEntity<Void> search(
			@Min(1) @RequestParam(value = "page") int page,
			@Min(1) @Max(100) @RequestParam(value = "size") int size,
			@Range(min = 1, max = 10) @RequestParam(value = "keyword") String keyword) {
		// page는 1보다 크고 size는 1~100 사이
		// keyword는 글자수 1~10 사이
		return ResponseEntity.noContent().build();
	}

	@GetMapping(value = "/products/{productNo}")
	public ResponseEntity<Void> getProduct(
			@Min(1) @PathVariable("productNo") int productNo) {
		// productNo는 최소 1이상
		return ResponseEntity.noContent().build();
	}
}
```

`@Validated`만 클래스에 적용한다면 Controller뿐만 아니라 다른 Spring Bean 모두에서 Validation 애노테이션을 적용할 수 있습니다.

실제 메서드 Validation 처리가 이루어지는 코드는 `MethodValidationInterceptor`에서 확인할 수 있습니다.

{% include ad_content.html %}

## ConstraintViolationException

위와 같이 메서드 파라미터나 리턴 값에 문제가 있으면 `ConstraintViolationException` 오류가 발생합니다. Spring에서는 이 오류를 기본적으로 HTTP 500 에러로 처리하기 때문에 사용자 요청 오류인 HTTP 400으로 변경하고 싶다면  별도로 변경처리를 해야 합니다.

`@ControllerAdvice`를 이용해서 `ConstraintViolationException`이 발생할 경우에 변경처리한 예제입니다.

```java
@ControllerAdvice
public class CustomExceptionHandler extends ResponseEntityExceptionHandler {
	@ExceptionHandler(value = {ConstraintViolationException.class})
	protected ResponseEntity<Object> handleConstraintViolation(ConstraintViolationException e, WebRequest request) {
		return handleExceptionInternal(e, e.getMessage(), new HttpHeaders(), HttpStatus.BAD_REQUEST, request);
	}
}
```
위 예제에서는 에러 메시지를 response body로 바로 표현했지만 각자의 환경에 맞는 response body 구조는 정의해서 사용하는 것이 좋습니다.

## @RequestBody

`@RequestBody`로 전달받은 값에 대해서도 검증이 가능합니다. 

`@RequestBody` 애노테이션을  설정하면 `RequestResponseBodyMethodProcessor`를 통해서 메서드 파라미터가 바인딩 됩니다. 그리고 `@Validated`이나 `@Valid` 애노테이션을 같이 설정하면 Valiation도 함께 처리합니다. 만약 처리 결과 중에 오류가 있을 경우 `MethodArgumentNotValidException`이 발생합니다.


```java
@Slf4j
@RestController
@RequiredArgsConstructor
public class ProductRestController {

	@PostMapping(value = "/products")
	public ResponseEntity<Void> create(@Valid @RequestBody ProductRequest request) {
		log.info("productRequest: {}", request);
		return ResponseEntity.noContent().build();
	}
}
```

## MethodArgumentNotValidException

`@ModelAttribute` 나 `@RequestBody` 처리를 위해 데이터 바인딩 중에 Validation 오류가 있을 경우 발생하는 오류입니다.  이 오류는 `ConstraintViolationException`과 다르게 기본적으로 HTTP 400 오류로 처리합니다.  

`ConstraintViolationException`에서는 `BindingResult`  정보를 가지고 있어서 필요한 오류 정보와 메시지 코드를 확인할 수 있습니다.

```java
@Slf4j
@ControllerAdvice
public class CustomExceptionHandler extends ResponseEntityExceptionHandler {
	@Override
	protected ResponseEntity<Object> handleMethodArgumentNotValid(
			MethodArgumentNotValidException ex, HttpHeaders headers,
			HttpStatus status, WebRequest request) {

		List<ObjectError> allErrors = ex.getBindingResult().getAllErrors();
		for (ObjectError error : allErrors) {
			log.error("error: {}", error);
			// 코드 정보를 확인할 수 있다.
			log.error("error: {}", Arrays.toString(error.getCodes()));
		}
		return super.handleMethodArgumentNotValid(ex, headers, status, request);
	}
}
```
위에서 로그로 확인한 error 객체에서는 code 정보가 있는데 이것을 활용해서 오류 메시지를 관리할 수도 있습니다. 먼저 어떻게 code가 생성되는지 확인해보겠습니다.

## MessageCodesResolver

Spring에서는 오류 메시지 코드관리르 위해 `MessageCodesResolver` 구현체인 `DefaultMessageCodesResolver`를 기본으로 사용합니다. 

이 클래스를 이용해서 코드가 생성되는 것을 확인해보면  애노테이션과 클래스 그리고 검증 대상 field 값을 통해서 만들어지는 것을 알 수 있습니다.
```java
public static void main(String[] args) {
	DefaultMessageCodesResolver codesResolver = new DefaultMessageCodesResolver();
	//codesResolver.setMessageCodeFormatter(Format.POSTFIX_ERROR_CODE);
	String[] codes = codesResolver.resolveMessageCodes(
			"Min", "productRequest", "price", int.class);

	for (String code : codes) {
		System.out.println(code);
	}
}
```
위 코드를 실행하면 애노테이션의 이름이 제일 앞에 있는 구조로 생성됩니다.
```
Min.productRequest.price
Min.price
Min.int
Min
```
애노테이션 이름 위치를 변경하고 위 예제 코드에서 주석처리 되어 있는 `setMessageCodeFormatter()` 메서드 설정에 따라 위치를 변경할 수도 있습니다.
```
productRequest.price.Min
price.Min
int.Min
Min
```
Spring에서 자동으로 설정하는 `DefaultMessageCodesResolver`를 직접 바꾸려면 아래와 같이 변경하면 됩니다.
```java
@Configuration
public class WebConfiguration implements WebMvcConfigurer {
	...
	@Override
	public MessageCodesResolver getMessageCodesResolver() {
		DefaultMessageCodesResolver codesResolver = new DefaultMessageCodesResolver();
		codesResolver.setMessageCodeFormatter(Format.POSTFIX_ERROR_CODE);
		return codesResolver;
	}
}
```

{% include ad_content.html %}

## Message

위에서 생성한 코드를 messageTemplate key로 사용해서 처리할 수도 있습니다.

classpath에 `CustomMessages.properties`이름의 파일을 만들고 
```properties
productRequest.price.Range=상품가격은 {2} ~ {1} 사이만 입력할 수 있습니다.
productRequest.saleEndAt.Future=판매종료일은 과거로 설정할 수 없습니다.
```
MessageSource와 Validator를 Spring Bean으로 등록합니다.
```java
@Configuration
public class WebConfiguration implements WebMvcConfigurer {
	@Bean
	public MessageSource messageSource() {
		ReloadableResourceBundleMessageSource messageSource =
				new ReloadableResourceBundleMessageSource();
		messageSource.addBasenames("classpath:messages/CustomMessages");
		messageSource.setDefaultEncoding("UTF-8");
		return messageSource;
	}
	
	@Bean
	public LocalValidatorFactoryBean getValidator(MessageSource messageSource) {
		LocalValidatorFactoryBean bean = new LocalValidatorFactoryBean();
		bean.setValidationMessageSource(messageSource);
		return bean;
	}
	...
}
```
그리고 ExceptionHandler에서 code에 해당하는 메시지를 찾습니다.
```java
@Slf4j
@ControllerAdvice
@RequiredArgsConstructor
public class CustomExceptionHandler extends ResponseEntityExceptionHandler {

	private final MessageSource messageSource;

	@Override
	protected ResponseEntity<Object> handleMethodArgumentNotValid(
			MethodArgumentNotValidException ex, HttpHeaders headers,
			HttpStatus status, WebRequest request) {

		List<ObjectError> allErrors = ex.getBindingResult().getAllErrors();
		for (ObjectError error : allErrors) {
			String message = Arrays.stream(Objects.requireNonNull(error.getCodes()))
					.map(c -> {
						Object[] arguments = error.getArguments();
						Locale locale = LocaleContextHolder.getLocale();
						try {
							return messageSource.getMessage(c, arguments, locale);
						} catch (NoSuchMessageException e) {
							return null;
						}
					}).filter(Objects::nonNull)
					.findFirst()
					// 코드를 찾지 못할 경우 기본 메시지 사용.
					.orElse(error.getDefaultMessage());

			log.error("error message: {}", message);
		}
		return super.handleMethodArgumentNotValid(ex, headers, status, request);
	}
}
```
```
error message: 판매종료일은 과거로 설정할 수 없습니다.
error message: 상품가격은 0 ~ 99,999,999 사이만 입력할 수 있습니다.
```

위에서 작성한 properties 파일 내용을 보면 `상품가격은 {2} ~ {1} 사이만 입력할 수 있습니다.`  이렇게 argument가 index로 되어 있는 것을 확인할 수 있습니다.  순서는 왜 {2}, {1}로 했으며 지난번에 확인한 것한 메시지와 다르게 왜 argument 이름이 아니고 index로 해야 할까요?

먼저 index 순서가 {2}를 {1} 보다 먼저 한 이유는 Annotation의 속성의 이름 알파벳 순서로 처리되기 때문입니다. `@Range`에는 2가지 속성이 있는데 min과 max입니다. 이 이름 순서로 정렬되기 때문에 {1}은 속성 **max**, {2}는 속성 **min**입니다. {0}은 참고로 필드명입니다. 

두 번째로 왜 argument를 name이 아니고 index로 처리해야 할까요? name으로 처리하는 것은 Hibernate의 `AbstractMessageInterpolator`에서 처리를 하는데 위 예제 코드처럼 Spring에서 별도 메시지로 처리할 경우 이Hibernate의 MessageInterpolator를 사용하지 않기 때문에 결국 Java MessageFormat에 따라 index로 동작하게 됩니다.

만약  name 기준으로 처리하고 싶다면 Hibernate Validator 기본인 classpath의ValidationMessages.properties 파일에 메시지를 추가하고 
```properties
productRequest.price.Range=상품가격 ${validatedValue}은 {min} ~ {max} 범위에 포함되지 않습니다.
```
별도로 정의했던 Spring Bean messageSource 와 validator를 제거하고 Spring Boot 기본설정으로만 하면 아래와 같은 메시지를 결과를 받을 수 있습니다.
```java
@Range(min = 0, max = 99_999_999, message = "{productRequest.price.Range}")  
private final int price;
```
```
error message: 상품가격 -1은 0 ~ 99999999 범위에 포함되지 않습니다.
```

## @Valid와 @Validated 차이

두 애노테이션의 차이점은 알아보면 `@Valid`는 jakarta.validation-api에서 제공하는 애노테이션입니다. nested 객체나 메서드 파라미터 객체를 validation할  사용합니다. 하지만 groups 관련 설정이 없기 때문에 Spring에서 `@Validated` 애노테이션을 추가로 만들어서 사용합니다.

```java
// groups를 설정 불강
@Valid
private final ProductRequest request
```
```java
// groups를 설정 가능
@Validated(value = { Create.class, Update.class })
private final ProductRequest request
```
끝.


## Reference
- [Difference between @Valid and @Validated in Spring](https://stackoverflow.com/questions/36173332/difference-between-valid-and-validated-in-spring)
- [Error Handling for REST with Spring](https://www.baeldung.com/exception-handling-for-rest-with-spring)
- [Spring Validation Message Interpolation](https://www.baeldung.com/spring-validation-message-interpolation)
- [The orders of bean validation default parameters?](https://stackoverflow.com/questions/10461945/the-orders-of-bean-validation-default-parameters)
- [Java Localization – Formatting Messages](https://www.baeldung.com/java-localization-messages-formatting)
- [Spring Validation custom messages - field name](https://stackoverflow.com/questions/34249937/spring-validation-custom-messages-field-name)