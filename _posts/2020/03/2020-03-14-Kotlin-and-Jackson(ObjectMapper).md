---
title: "Kotlin and Jackson(ObjectMapper)"
last_modified_at: 2020-03-14T19:34:00+00:00
header:
  show_overlay_excerpt: false
  overlay_image: /assets/images/post/2020/03/2020-03-14-books.jpg
  og_image: /assets/images/post/2020/03/2020-03-14-books.jpg
  overlay_filter: 0.6
  caption: "Photo by Patrick Tomasso on Unsplash"
  
tags:
  - ObjectMapper
  - Jackson
category: #카테고리
  - Kotlin
  - Json
toc: false
toc_label: "Table Of Contents"
toc_icon: "fal fa-list-alt"
toc_sticky: true
classes: wide
comments: true
---


kotlin 환경에서 Jackson 라이브러리를 사용할 때 객체 변환이 생각대로 잘 안될 수 있습니다. 어떤 경우가 있는지 어떻게 해결할 수 있는지 확인해 보겠습니다.

## 너무 거추장스러운 Annotation
먼저 아래  json을 deserialize 해보겠습니다.

```json
{
  "ticketNo": 189,
  "name": "test",
  "discount": 5.5,
  "couponNos": [
    1,
    2
  ],
  "extraInfo": {
    "address": "서울시",
    "device": "MOBILE"
  }
}
```
`SampleRequest` 이름의 data class를 하나 만듭니다. 불변 상태를 위해 생성자에서 필요한 값을 설정하는 구조입니다.
```kotlin
data class SampleRequest(
    val ticketNo: Int,
    val name: String,
    val discount: BigDecimal,
    val couponNos: List<Int>,
    val extraInfo: Map<String, Any>
)
```
위 내용으로 Jackson의 ObjectMapper를 이용해서 실행해보면...
```kotlin
objectMapper.readValue<SampleRequest>(json, SampleRequest::class.java)
```
오류가 발생합니다. 
```
com.fasterxml.jackson.databind.exc.InvalidDefinitionException: Cannot construct instance of `com.study.jpa.batch.json.SampleRequest` (no Creators, like default construct, exist): cannot deserialize from Object value (no delegate- or property-based Creator)
```
**(no Creators, like default construct, exist)**라는 메시지로 보아 아무래도 생성자 방식으로 deserialize 하려고 하는데 잘 안되는 것 같습니다. 확인해 보니깐`@JsonCreator`와 `@JsonProperty`를 이용하면 가능할 것 같습니다. `SampleRequest`를 조금 변경해 보겠습니다. 

```kotlin
data class SampleRequest @JsonCreator constructor(
    @JsonProperty("ticketNo") val ticketNo: Int,
    @JsonProperty("name") val name: String,
    @JsonProperty("discount") val discount: BigDecimal,
    @JsonProperty("couponNos") val couponNos: List<Int>,
    @JsonProperty("extraInfo") val extraInfo: Map<String, Any>
)
```
`@JsonCreator`는 class에서 사용 못 하기 때문에 생성자에 적용해야 합니다. 그러다 보니 생성자 관련 코드가 추가되어야 하다 보니 클래스가 약간 지저분 해지는 것 같습니다. kotlin의 간결함이 많이 사라지는 것 같네요. 그래도 이렇게 변경하고 실행하면 이상 없이 잘 만들어지는 걸 확인할 수 있습니다. 

> property 이름이 json이랑 같아도 @JsonProperty를 사용해야 합니다. 그렇지 않으면 오류가 발생합니다. `Argument #0 has no property name, is not Injectable: can not use as Creator`


## KotlinModule 

data class와 불변인 구조를 포기하고 싶지는 않은데 이것을 유지하려면 너무 많은 annotation을 사용해야 하는 것 같습니다. 이 방법밖에 없다면 정말 사용하기 싫을 텐데요. 다행히도 `jackson-module-kotlin` 모듈에서 `KotlinModule` 클래스를 통해서 문제를 해결할 수 있습니다. 

> implementation("com.fasterxml.jackson.module:jackson-module-kotlin")

`ObjectMapper` 객체를 생성해서 모듈을 직접 지정할 수도 있고 Jackson에서 제공하는 확장 함수로 쉽게 사용할 수도 있습니다.

```kotlin
val mapper1 = ObjectMapper().registerModule(KotlinModule())  
val mapper2 = jacksonObjectMapper()  
val mapper3 = ObjectMapper().registerKotlinModule()
```
이제 annotation을 다 걷어내고 다시 실행 해보겠습니다.

```kotlin
data class SampleRequest(
    val ticketNo: Int,
    val name: String,
    val discount: BigDecimal,
    val couponNos: List<Int>,
    val extraInfo: Map<String, Any>
)

fun main() {
  val mapper = jacksonObjectMapper()
  val readValue = mapper.readValue<SampleRequest>(json, SampleRequest::class.java)
  println(readValue)
}
```
결과는 성공입니다.
```
SampleRequest(ticketNo=189, name=test, discount=5.5, couponNos=[1, 2], extraInfo={address=서울시, device=MOBILE})
```

## Not Null 타입 

`SampleRequest`  클래스를 보면 모두 not null 타입으로 정의를 했습니다. 만약 json value 중에서 null이 있으면 어떤 일이 생길까요?

먼저 json에서 ticketNo를 null로 바꾸고 실행해보면 오류는 발생하지 않고 0으로 설정됩니다. primitive 타입이라 null 일 수 없어서 그렇습니다. 

> primitive 타입의 null 문제는 DeserializationFeature.FAIL_ON_NULL_FOR_PRIMITIVES 설정을 통해서 제어할 수 있습니다.

 String 프로퍼티인 name 을 null로 바꾸고 실행하면 어떻게 될까요?
```json
{
    "name": null,
     ...
}
```
```
MissingKotlinParameterException: Instantiation of ... value failed for JSON property name due to missing (therefore NULL) value for creator parameter name which is a non-nullable typ
```
not null 타입인데 null 값을 세팅하려다 보니 오류가 발생합니다.  

하지만 json에서 `"name": null`이 아니라 아예 빼버리고 name property에 기본값을 설정하면 잘 동작합니다.  다시 말해서 `{}` => `val name: String = ""` 이렇게 하면 기본값으로 적용됩니다.

문제는 처리할 json에서 name 을 아예 전달 안 할지 null이라도 전달할지 알 수가 없기 때문에 이렇게만 처리하기에는 만족스럽지 못 합니다.

검색해서 찾아봤더니 비슷한 이슈가 이미 있었습니다. [Using Kotlin Default Parameter Values when JSON value is null and Kotlin parameter type is Non-Nullable](https://github.com/FasterXML/jackson-module-kotlin/issues/130) . 

jackson-module-kotlin 2.10.1 부터 `KotlinModule`에 `nullisSameAsDefault` 속성이 추가 되었다고 합니다. default가 false이기 때문에 true로 변경하면 json에서 name을 전달하지 않은 경우와 null인 경우 모두 기본값으로 적용이 됩니다.
```kotlin
ObjectMapper().registerModule(KotlinModule(nullisSameAsDefault = true))
```
> [jackson-module-kotlin:2.10.1]([https://github.com/FasterXML/jackson-module-kotlin/compare/jackson-module-kotlin-2.10.1...master](https://github.com/FasterXML/jackson-module-kotlin/compare/jackson-module-kotlin-2.10.1...master))에서 "Fixed issue #130 Default Parameter Values when JSON value is null (#259)" 을 확인할 수 있습니다.

nullisSameAsDefault 속성을 이용하면 나머지 다른 타입의 property도 모두 동일한 상황일 때 기본값으로 잘 동작합니다.

```kotlin
data class SampleRequest(
    val ticketNo: Int,
    val name: String = "",
    val discount: BigDecimal = BigDecimal.ZERO,
    val couponNos: List<Int> = emptyList(),
    val extraInfo: Map<String, Any> = emptyMap()
)
```

만약 json에 대한 NotEmpty 같은 validation이 필요하다면 deserialize 과정에 할 필요가 없습니다. `SampleRequest` 객체가 만들어지고 난 이후에 처리하면 됩니다. 이렇게 하면 property 타입을 nullable로 변경하지 않기 때문에 내부적으로 `SampleRequest` 객체를 사용하는 코드에서 좀 더 간결하고 안정적으로 사용할 수 있습니다

끝.

## Reference
- [Kotlin, jackson: cannot annotate @JsonCreator in primary constructor](https://stackoverflow.com/questions/51350426/kotlin-jackson-cannot-annotate-jsoncreator-in-primary-constructor)
- [Using Kotlin Default Parameter Values when JSON value is null and Kotlin parameter type is Non-Nullable](https://github.com/FasterXML/jackson-module-kotlin/issues/130)