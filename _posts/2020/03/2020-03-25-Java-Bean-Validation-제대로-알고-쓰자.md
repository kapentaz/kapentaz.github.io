---
title: "Java Bean Validation 제대로 알고 쓰자"
last_modified_at: 2020-03-25T00:45:00+00:00
header:
  show_overlay_excerpt: false
  overlay_image: /assets/images/post/2020/03/2020-03-25-title.jpg
  og_image: /assets/images/post/2020/03/2020-03-25-title.jpg
  overlay_filter: 0.6
  caption: "Photo by Hans Ripa on Unsplash"
  
tags:
  - Java
  - Validation
category: #카테고리
  - Java
toc: false
toc_label: "Table Of Contents"
toc_icon: "fal fa-list-alt"
toc_sticky: true
classes: wide
comments: true
---


개발하면서 제일 중요하게 생각하는 것 중에 하나가 validation입니다. 개발하고 운영하다 보면 클라이언트로부터 입력받은 값의 오류로 발생하는 장애가 꽤 많습니다. 잘못된 값을 전달받아 즉시 오류가 발생하면 그나마 다행입니다. 오류 없이 그대로 데이터가 저장이나 수정되고 그 데이터로 다른 작업을 진행하면서 오류가 발생하기도 합니다. 그런 경우에는 어디서부터 잘못되었는지 원인 파악도 힘들어집니다.

이렇게 중요한 작업 중의 하나인 validation을 위해 Java에서는 [Jakarta Bean Validation](https://beanvalidation.org/)라는 Specification을 이용할 수 있습니다. Annotation을 이용해서 Validation을 처리할 수 있는 방법인데요. 이 Specification을 구현한 모듈이 [Hibernate Validator](https://docs.jboss.org/hibernate/stable/validator/reference/en-US/html_single/#validator-gettingstarted-createproject)입니다.  

## Dependency

Hibernate Validator를 사용하기 위해 필요한 dependency 정보입니다. 
```groovy
implementation('org.hibernate.validator:hibernate-validator:6.1.2.Final')  
implementation('org.glassfish:jakarta.el:3.0.3')
```

Jakarta Bean Validation은 Hibernate Validator에 포함되어 있어서 별도로 지정하지 않아도 됩니다.

`jakarta.el`은 validation 결과 메시지를 표현하기 위해서 사용하는데 JAVA EE 컨터이너를 사용하고 있다면 이미 추가가 되어 있을 것이고 그렇지 않다면 dependency를 추가해야 합니다.


참고로 Jakarta Bean Validation만 추가하면 Hibernate Validator 같은 모듈을 추가하라고 에러가 발생합니다.
```
javax.validation.NoProviderFoundException
Unable to create a Configuration, because no Bean Validation provider could be found. 
Add a provider like Hibernate Validator (RI) to your classpath.
```

EL 모듈을 추가하지 않아도 오류가 발생합니다.
```
javax.validation.ValidationException
HV000183: Unable to initialize 'javax.el.ExpressionFactory'. 
Check that you have the EL dependencies on the classpath, or use ParameterMessageInterpolator instead
```

{% include ad_content.html %}

## Validation

Validation 처리를 위해서 직접 코드로 할 수도 있겠지만 Jakarta Bean Validation을 이용하면 기본 제공 Annotation만 이용하더라도 심플하게 처리할 수 있습니다. 참고로 Hibernate Validator에서도 몇 가지 Annotation을 [**추가로 제공**](https://docs.jboss.org/hibernate/stable/validator/reference/en-US/html_single/#validator-defineconstraints-hv-constraints)합니다.

### Field 적용
Annotation을 이용하는 방법에 대해서 살펴보겠습니다. `Product`라는 클래스가 있고 객체에서 가질 수 있는 값에 대한 검증 정보를 아래와 같이 설정할 수 있습니다.

```java
@Getter
@RequiredArgsConstructor
public class Product {
    @Positive
    private final int productNo;

    @Size(min = 4, max = 100)
    private final String name;

    @Min(0)
    @Max(99_999_999)
    private final int price;

    @DecimalMin(value = "0.0")
    @DecimalMax(value = "100.0")
    private final BigDecimal discount;

    @PastOrPresent
    private final LocalDateTime saleStartAt;

    @Future
    private final LocalDateTime saleEndAt;

    @Past
    private final LocalDate dateOfManufacture;
}
```
Annotation 이름만 봐도 어떤 역할을 할지 충분히 예측이 가능합니다. 예제에서 사용하지 않은 Annotation과 자세한 설명은 [**Built-in Constraint definitions**](https://beanvalidation.org/2.0/spec/#builtinconstraints)에서 확인할 수 있습니다.

이제 위에서 정의한 `Product` 객체를 생성해서 Validation을 실행해보겠습니다.
```java
public static void main(String[] args) {
    // Product 객체 생성
    Product product = new Product(0, "화장품", 13000, BigDecimal.valueOf(5.5),
            LocalDateTime.now().plusDays(5),
            LocalDateTime.now().plusDays(10),
            LocalDate.now().minusMonths(3));

    // Validator 생성
    ValidatorFactory validatorFactory = Validation.buildDefaultValidatorFactory();
    Validator validator = validatorFactory.getValidator();

    // validation 및 출력
    Set<ConstraintViolation<Product>> validate = validator.validate(product);
    validate.forEach(System.out::println);
    
    // 예외로 바로 처리할 수도 있습니다.
    // throw new ConstraintViolationException(validate);
}
```
```
ConstraintViolationImpl{interpolatedMessage='size must be between 4 and 100크기가 4에서 100 사이여야 합니다', propertyPath=', propertyPath=, rootBeanClass=class com.validation.ch01.Product, messageTemplate='{javax.validation.constraints..message}'}
ConstraintViolationImpl{interpolatedMessage='과거 또는 현재의 날짜여야 합니다', propertyPath=productNosaleStartAt, rootBeanClass=class com.validation.ch01.Product, messageTemplate='{javax.validation.constraints.Positive.message}'}
ConstraintViolationImpl{interpolatedMessage='0보다 커야 합니다', propertyPath=productNo, rootBeanClass=class com.validation.ch01.Product, messageTemplate='{javax.validation.constraints.PastOrPresentositive.message}'}
```

Factory로 Validator 생성하고 검증 대상을 `validate()` 메서드로 실행하면 그 결과를  얻을 수 있습니다. Validation을 통과하지 못했다고 해서 바로 Exception이 발생하지는 않습니다. 

### Getter 적용

Annotation을 field에 적용할 수 있지만 getter에도 적용할 수 있습니다. boolean의 경우 `is`나 `has`로 시작하는 경우에 해당합니다. 

>  DefaultGetterPropertySelectionStrategy 클래스를 보면 "get", "is", "has"로 시작하는 메서드인지 확인하는 코드가 있습니다.

```java
@Getter
@RequiredArgsConstructor
public class Product {
  
    ....
		
    /**
     * 종료일이 시작일보다 미래인지 확인
     */
    @AssertTrue
    public boolean isValidSalePeriod() {
        return saleEndAt.isAfter(saleStartAt);
    }

    @Size(max = 2)
    public String getName() {
        return this.name;
    }
}
```
```
ConstraintViolationImpl{interpolatedMessage='must be true', propertyPath=validSalePeriod, rootBeanClass=class com.validation.ch01.Product, messageTemplate='{javax.validation.constraints.AssertTrue.message}'}
```
눈치 빠른 분들은 아시겠지만 `isValidSalePeriod()` 메서드를 보면 필드값을 리턴하는 게 아니라 `saleEndAt`이 `saleStartAt` 보다 미래인지 확인하는 용도로 사용했습니다. 이 방법을 이용하면 Cross Field Validation 처리를 할 수 있습니다.

위 실행 결과 메시지를 보면  `must be true`라고 되어 있는데 좀 더 상세하게 메시지를 표현하고 싶다면 직접 Message를 정의해서 처리할 수 있습니다. Message 처리와 관련해서는 조금 뒤에 자세히 알아보겠습니다. 

### Nested

아래와 같은 Option  클래스가 있고 
```java
@Getter
@RequiredArgsConstructor
public class Option {
    @Length(min = 1, max = 100)
    private final String optionName;
    @URL
    private final String optionImageUrl;
}
```

`Product`에서 이 옵션 정보를 가지고 있을 때 `Option`에서 Validation 애노테이션을 적용했다고 해서 그냥 적용되지는 않습니다. nested로 구성되어 있을 경우에는 `@Valid` 애노테이션을 설정해야 제대로 동작을 합니다.

```java
@Getter
@RequiredArgsConstructor
public class Product {

    ...
	
    @Valid
    private final List<Option> options;
}
```

{% include ad_content.html %}

### Parameter

Method Parameter를 대상으로 Validation을 실행할 수도 있습니다.  `Method` Reflection을 이용하는 방법인데 특정 파라미터의 Validation 결과를 확인할 수 있습니다. 참고로 실제 메서드를 실행하는 것은 아닙니다. Reflection 정보로 파라미터가 유효한지 확인만 합니다.

```java
public class Calculator {
    @Min(100000)
    public int calculate(@Min(1_000) int price,
                         @Max(3_000) int discountPrice) {
        return price - discountPrice;
    }  
}
```
```java
public static void main(String[] args) throws NoSuchMethodException {
    // ExecutableValidator 생성
    ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
    ExecutableValidator validator = factory.getValidator().forExecutables();

    // Calculator 클래스의 calculate(..) 메서드 정보 생성
    Calculator calculator = new Calculator();
    Method method = Calculator.class.getMethod("calculate", int.class, int.class);
  
    // 메서드 파라미터 validation. calculate(..) 메서드를 실행하지는 않음
    Set<ConstraintViolation<Calculator>> violations = validator.validateParameters(
            calculator, method, new Object[] { 900, 4_000 });

    violations.forEach(System.out::println);
}
```
```
ConstraintViolationImpl{interpolatedMessage='must be less than or equal to 3000', propertyPath=calculate.discountPrice, rootBeanClass=class com.validation.ch02.Calculator, messageTemplate='{javax.validation.constraints.Max.message}'}
ConstraintViolationImpl{interpolatedMessage='must be greater than or equal to 1000', propertyPath=calculate.price, rootBeanClass=class com.validation.ch02.Calculator, messageTemplate='{javax.validation.constraints.Min.message}'}
```

#### Dynamic Proxy

실무에서는 이런 식으로 사용하기는 힘들 것입니다. Dynamic Proxy를 이용하면 좀 더 편하게 이용할 수 있습니다. Dynamic Proxy를 적용한 예제를 만들어 보겠습니다.  `Calculator` 인터페이스를 만들고 그 구현체로 `CalculatorImpl` 만들겠습니다.
```java
public interface Calculator {
    int calculate(@Min(1_000) int price,
                  @Max(3_000) int discountPrice);
}

public class CalculatorImpl implements Calculator {  
    public int calculate(int price, int discountPrice) {  
        return price - discountPrice;  
  }  
}
```
그리고 Dynamic Proxy를 위해 `ValidationInvocationHandler` 클래스도 만들겠습니다. 메서드 파라미터 검증이 실패할 경우 `ConstraintViolationException`이 발생하도록 하겠습니다.
```java
public class ValidationInvocationHandler implements InvocationHandler {

    private final ExecutableValidator validator;
    private final Object target;

    public ValidationInvocationHandler(ExecutableValidator validator, Object target) {
        this.validator = validator;
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      	// 실행할 메서드 전에 파라미터 validation 실행
        Set<ConstraintViolation<Object>> violations = validator.validateParameters(proxy, method, args);
        if (!violations.isEmpty()) {
            throw new ConstraintViolationException(violations);
        }
      
      	// 실제 실행할 메서드
        return method.invoke(target, args);
    }
}
```
Proxy 객체를 통해서 `calculate()` 메서드를 실행하면 
```java
public static void main(String[] args) {
    ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
    ExecutableValidator validator = factory.getValidator().forExecutables();

    // Proxy 객체 생성
    Calculator calculatorProxy = (Calculator) Proxy.newProxyInstance(
            CalculatorImpl.class.getClassLoader(),
            new Class[]{Calculator.class},
            new ValidationInvocationHandler(validator, new CalculatorImpl()));

    // proxy를 통해서 파라미터 검증 후 calculate(..) 메서드 실행
    calculatorProxy.calculate(1000, 4000);
}
```
예상한대로 오류가 발생합니다.
```
Exception in thread "main" javax.validation.ConstraintViolationException: calculate.discountPrice: must be less than or equal to 3000
```

### Return

`validateReturnValue()` 메서드를 이용하면 return value도 validation 할 수 있지만 이것도 실제 메서드를 실행하는 것은 아닙니다.

```java
Set<ConstraintViolation<Calculator>> violations = validator.validateReturnValue(calculator, method, 100);
```

### Constructor

`ExecutableValidator`의 `validateConstructorReturnValue()`와 `validateConstructorParameters()` 메서드를 이용하면 생성자에서도 동일하가 파라미터와 return 값을 Validation 할 수 있습니다.

{% include ad_content.html %}

## Message 

Validation 실행 결과를 출력한 내용을 보면  `ConstraintViolationImpl`에서 4가지 주요 항목을 확인할 수 있습니다. **propertyPath**, **rootBeanClass**, **messageTemplate** 그리고 **interpolatedMessage** 

**propertyPath**는 Violation이 발생한 위치, **rootBeanClass**은 대상 클래스입니다.

**messageTemplate**은 오류 메시지 템플릿 key를 의미합니다. 일반적으로는 Annotation 클래스의 fullname과 message 붙여서 나타냅니다. 예를 들면 아래와 같은 구조입니다.

```
"{javax.validation.constraints.Positive.message}"
"{javax.validation.constraints.Min.message}"
"{javax.validation.constraints.Future.message}"
```
Hibernate Validator 모듈에서는 기본 템플릿 key에 대한 Message Property를 제공합니다. Locale 별로 다양한 Property를 제공하고 있습니다. 지원하는 property는 모듈 내부를 살펴보면 확인할 수 있습니다. 물론 한국어는 제공하고 있습니다.

Validator 객체를 생성하기 전에 Locale 설정을 변경하면 언어별 메시지를 확인할 수 있습니다. 

> 지금 사용하고 있는 컴퓨터 기본 설정이 영어로 되어 있어서 한국어로 보려고 Locale 변경을 했습니다.

```java
Locale.setDefault(KOREA);  
  
ValidatorFactory validatorFactory = Validation.buildDefaultValidatorFactory();  
Validator validator = validatorFactory.getValidator();  
  
Set<ConstraintViolation<Product>> validate = validator.validate(product);  
validate.forEach(System.out::println);
```

```
ConstraintViolationImpl{interpolatedMessage='올바른 URL이어야 합니다', propertyPath=options[0].imageUrl, rootBeanClass=class com.validation.ch01.Product, messageTemplate='{org.hibernate.validator.constraints.URL.message}'}
ConstraintViolationImpl{interpolatedMessage='0보다 커야 합니다', propertyPath=productNo, rootBeanClass=class com.validation.ch01.Product, messageTemplate='{javax.validation.constraints.Positive.message}'}
ConstraintViolationImpl{interpolatedMessage='크기가 4에서 100 사이여야 합니다', propertyPath=name, rootBeanClass=class com.validation.ch01.Product, messageTemplate='{javax.validation.constraints.Size.message}'}
ConstraintViolationImpl{interpolatedMessage='과거 또는 현재의 날짜여야 합니다', propertyPath=saleStartAt, rootBeanClass=class com.validation.ch01.Product, messageTemplate='{javax.validation.constraints.PastOrPresent.message}'}
```

**interpolatedMessage**은 messageTemplate으로 가져온 메시지에서 Expression Llanguage를 이용해서 치환된 메시지를 나타냅니다. 

예를 들어 `@Size(min = 4, max = 100)`라고 설정할 경우 아래 message에서 `min`과 `max`가 치환된 결과 메시지가 interpolatedMessage 내용이 됩입니다.
```
size must be between {min} and {max}
```
`@DecimalMax`의 경우를 보면 condition에 따라 메시지 출력을 다르게 할 수도 있습니다.

```
must be less than ${inclusive == true ? 'or equal to ' : ''}{value}
```

### Custom Message

#### message 속성 재정의
Validation Annotation에 있는 message는 재정의 할 수 있습니다. 

message에 노출할 메시지를 직접 작성할 수 있습니다. 이 방법은 소스코드에서 바로 메시지 내용을 알 수 있는 것이 장점이지만, 다국어 처리가 어렵고 메시지를 수정하기 위해서는 Java 코드를 수정해야합니다.
```java
@AssertTrue(message = "시작일은 종료일보다 과거로 설정해야 합니다.")  
public boolean isValidSalePeriod() {  
    return saleEndAt.isAfter(saleStartAt);  
}
```
 Custom MessgaeTemplate을 지정하는 방법입니다. 정의한 코드로 Message Property 파일에 등록하면 해당 메시지로 결과를 반환합니다. 다국어 지원도 가능하고 메시지 변경시 Java 코드를 수정할 필요가 없습니다.
```java
@AssertTrue(message = "{com.sample.Product.AssertTrue.message}")  
public boolean isValidSalePeriod() {  
    return saleEndAt.isAfter(saleStartAt);  
}
```
```properties
com.sample.Product.AssertTrue.message=시작일은 종료일보다 과거로 설정해야 합니다.
```

#### Message Property 재정의
기본으로 저장되어 있는 메시지 말고 새롭게 정의하고 싶다면 classpath에 `ValidationMessages.properties` 이름으로 property를 추가하면 됩니다. 새로 추가한 항목만 재정의해서 사용할 수 있습니다. locale 별로 메시지를 다르게 하기 위해서는 locale 정보를 파일명(예를 들어 `ValidationMessages_ko.properties` )에 포함해서 만들면 됩니다. 

"상품번호"라고 이름을 입력해서 좀 더 이해하기 쉽게 message를 만들었습니다.

```properties
javax.validation.constraints.Positive.message=상품번호는 0보다 큰수를 입력하세요.
```

Hibernate Validator는 Message Property 처리를 위해 모듈 내에 포함되어 있는`org.hibernate.validator.ValidationMessages` 파일명과 Application Classpath에서 `ValidationMessages` 이름의 Message Property를 찾습니다. 사용자가 정의한 `ValidationMessages`에 messageTemplate이 존재하면 그걸 먼저 사용하고 없다면 기본으로 정의한 Message Property를 사용합니다.

{% include ad_content.html %}

## Custom Validator

기본적으로 제공하는 Annotation으로는 충분하지 않을 수 있습니다. 필요에 따라 직접 만들 수도 있습니다. 객체와 메서드 기준으로 확인을 했으니 클래스에 적용할 Custom Validator를 만들어 보겠습니다.

Custom Constraint Annotation을 만들 때는 **message**, **groups** 그리고 **payload** 3개는 꼭 정의해야 합니다. message는 앞서 살펴본대로 메시지 관리를 위해 사용됩니다.
```java
@Documented
@Constraint(validatedBy = ProductConstraintValidator.class)
@Target( { ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
public @interface ProductConstraint {
    String message() default "{com.validation.ch03.ProductConstraint.message}";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```
**groups은** 상황별 validation 제어를 위해 사용할 수 있습니다. 예를 들어 Insert 할 때와 Update할 때 Validation을 구분해서  실행 하고 싶을 때 사용할 수 있습니다.  

```java
// groups 구분을 위해 정의한 interface
public interface Insert{}  
public interface Update{}
```
```java
@DecimalMin(value = "0.0", groups = Insert.class)  
@DecimalMax(value = "100.0", groups = Update.class)  
private final BigDecimal discount;
```

**payload**는 심각도를 나타낸다고 하는데 사실 이렇게까지 디테일하게 나눠서 처리할 일은 없었기에 실제 사용해본적은 없습니다.

이제 위에서 만든 `ProductConstraint` 애노테이션을 처리할 Validator를 만들겠습니다. `ConstraintValidator` 인터페이스를 구현해야 합니다. `ProductConstraint`  애노테이션이 `Product` 클래스에 적용된 경우에 실행하게 됩니다. 

```java
public class ProductConstraintValidator 
        implements ConstraintValidator<ProductConstraint, Product> {

    @Override
    public void initialize(ProductConstraint annotation) {
      	// annotation에 있는 정보를 멤버변수로 저장해서 isValid()에서 사용할 수 있습니다.
    }

    @Override
    public boolean isValid(Product product, ConstraintValidatorContext context) {
        return product.getSaleEndAt().isAfter(product.getSaleStartAt());
    }
}
```

`isValid()` 메서드에서 파라미터를 Product로 받을 수 있기 때문에 Cross Field Validation 처리를 할 수 있습니다.

만약 Cross Field Validation가 목적이라면 이렇게 별도의 애노테이션과 Validator 클래스를 만드는 것이 꺼려질 수도 있습니다. 그렇다면 앞서 살펴본 `@AssertTrue` 애노테이션을 사용하는 게 효과적일 수도 있습니다. 

끝.


## Reference
- [Jakarta Bean Validation](https://beanvalidation.org/)
- [Hibernate Validator 6.1.2.Final](https://docs.jboss.org/hibernate/stable/validator/reference/en-US/html_single/#validator-gettingstarted-createproject)
- [JavaBean Validation - Validation messages](https://www.logicbig.com/tutorials/java-ee-tutorial/bean-validation/validation-messages.html)
- [Method Constraints with Bean Validation 2.0](https://www.baeldung.com/javax-validation-method-constraints)
- [Naming conventions for java methods that return boolean(No question mark)](https://stackoverflow.com/questions/3874350/naming-conventions-for-java-methods-that-return-booleanno-question-mark)
