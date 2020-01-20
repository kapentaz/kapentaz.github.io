---
title: "Http Request File in Intellij"
last_modified_at: 2020-01-20T23:41:00-05:00
excerpt: "뒤늦게 알게 된 Http Request File 기능! 좋습니다."
header:
  overlay_image: /assets/images/background/light_in_the_city.jpg
  og_image: /assets/images/background/light_in_the_city.jpg
  overlay_filter: 0.6
  caption: "Photo Credit: [Brady](https://kapentaz.github.io)"
tags:
  - IntelliJ
  - Test
  - API
category: #카테고리
  - IntelliJ
toc: false
toc_label: "Table Of Contents"
toc_icon: "fal fa-list-alt"
toc_sticky: true
classes: wide
comments: true
---


뒤늦게 알게된 Http Request File 기능이 좋아서 [메뉴얼](https://www.jetbrains.com/help/idea/http-client-in-product-code-editor.html) 참고해서 내용 정리립니다.

## Rest Client

개발을 진행하다 보면 API를 호출이 필요한 경우가 있습니다. 코드로 작성하기 전에 미리 잘 되는지 요청과 응답 정보를 확인할 필요가 있는데요. Intellij에서는 이럴 때 사용하기 좋은 Rest Client라는 기능이 있습니다.

> Tool > HTTP Client > Test Restful Web Service

![Rest Client](https://raw.githubusercontent.com/kapentaz/kapentaz.github.io/master/assets/images/post/2020/http_client.png)

request 정보를 설정하고 호출하면 응답 정보를 Intellij에서 바로 확인할 수 있습니다. method, host, path, header 등을 설정할 수도 있고 응답도 바로 확인할 수 있으니 개발할 때 꽤 유용하게 사용할 수 있습니다.

이 기능을 처음 알게 되었을 때 적극적으로 사용하려고 했지만 지금은 사용하지 않습니다. 그 이유는 API를 호출한 기록 보기가 어려웠고 API 호출 정보를 저장하고 불러오기도 불편했습니다. 그리고 새로운 request 정보를 등록하기 위해서는 너무 많은 커서 이동과 클릭이 필요했습니다.

개발할 때 도구 사용을 위해서 창 이동을 별로 좋아하지 않아서 Sourcetree, MySQL Workbench를 사용하지 않고 Intellij로 다 해결하는 저에게는 쉽지 않았지만 결국 포기하고 [Postman](https://www.getpostman.com/)을 사용합니다.

Postman은 히스토리 관리도 편했고, Collections을 통해서 미리 정의하고 호출하기도 편했습니다. 링크를 통해서 다른 사람과 호출 정보를 공유할 수도 있고 그 외에도 다양한 기능이 있습니다. 하지만 여전히 많은 마우스 이동과 클릭 등 불편한 점이 있습니다.

## Http Request File

IntelliJ IDEA 2017.3부터 http request file 기능이 생겼다고 합니다. 생성 방법은 간단합니다.  `.http` 확장자를 가진 파일을 만들면 됩니다. 프로젝트에 파일로 관리할 정도는 아니고 잠시 테스트할 목적이라면 scratch file 파일로 생성( `shift` + `command` + `n`)할 수도 있습니다.

API 호출 정보를 file로 관리할 수 있으니 git을 통해서 버전 관리와 공유가 편한 장점이 있습니다. 그리고 코드로 요청 정보를 만들기 때문에 editor 창에서 바로 작성할 수 있서 편리합니다. 요청 문법 구조는 다음과 같습니다.

```
Method Request-URI HTTP-Version
Header-field: Header-value

Request-Body
```

하나의 파일에 여러 요청 정보를 등록할 수 있으며 각 요청은 ###로 구분합니다.

### Live Templates

라이브 템플릿을 활용하면 더 빠르게 작성할 수 있습니다. 미리 등록된 template은 외에도 필요에 따라 추가할 수 있습니다.

> Preferences > Editor > Live Templates

![live template](https://raw.githubusercontent.com/kapentaz/kapentaz.github.io/master/assets/images/post/2020/http_request_file_live_template.gif)

| Template | Usage                                     |
| :------- | :---------------------------------------- |
| gtr      | GET Request                               |
| gtrp     | GET Request with query parameters         |
| ptr      | POST Request with simple body             |
| ptrp     | Post Request with parameter-like body     |
| mptr     | Post Request to submit a form             |
| fptr     | Post Request to submit a form with a file |

### External File

request body는 미리 정의되어 있는 외부 파일로 지정할 수 있습니다. 외부 파일 경로는 `./idea/httpRequests` 여기 아래에 위치하거나 파일의 fullpath를 지정해야 합니다. (같은 경로에 있는 파일은 경로를 못찾는건 같은데 이유는 아직 모르겠네요.)

```http
POST http://localhost:8080/products
Content-Type: application/json

< product.json
```

### Environment

API 호출 테스트를 하다 보면 Dev와 Real 같이 환경별로 호출해야 할 수 있습니다. 이럴 때 환경마다 http file을 만드는 건 비효율적이기 때문에 환경별 변수를 파일로 지정할 수 있습니다. `rest-client.env.json`라고 파일명을 만들고 변수를 지정하면 사용할 수 있습니다.

```json
{
  "local":{
    "host":"localhost"
  },
  "dev":{
    "host":"dev.sample.com"
  },
  "real":{
    "host":"sample.com"
  }
}
```

```http
GET http://{{host}}:8080/products/1
```

![http request file env](https://raw.githubusercontent.com/kapentaz/kapentaz.github.io/master/assets/images/post/2020/http_request_file_env.png)

### Variable and Test

#### Variable

하나 이상의 API를 호출할 경우에 다른 API를 호출한 response를 이용해서 다음 API request를 해야하는 경우가 있습니다. 이럴 때는 client 객체를 이용해서 변수로 저장했다가 사용할 수 있습니다.
```http
GET http://localhost:8080/products/1
Accept: application/json

{% raw %}
> {%  client.global.set("productName", response.body.productName); %}

###

POST http://localhost:8080/products
Content-Type: application/json

{"productName": "{{productName}}"}
{% endraw %}

###
```

#### Test
API  호출 결과에 대한  검증도 진행할 수 있습니다.  보통 status 확인으로 충분하지만, response 내용도 함께 검증할 수 있습니다.

```http
GET http://localhost:8080/products/1
Accept: application/json

{% raw %}
> {% client.test("test", function() {
  client.log(response.body);
  client.assert(response.status === 200, "OK");
});
%}
{% endraw %}
```

## 결론
API 호출과 확인을 파일로 작성하기 때문에 editor 창에서 바로 더 빠르고 편하게 만들 수 있습니다. git으로 버전 관리와 공유하기도 용이합니다. 다른 도구 사용을 위해 창 이동도 필요 없습니다. 개인적으로 굉장히 만족스럽습니다. 아직 사용해보지 않았다면, 사용해보시길 추천드립니다.

끝.

## Reference

- [HTTP client in IntelliJ IDEA code editor](https://www.jetbrains.com/help/idea/http-client-in-product-code-editor.html)
- [IntelliJ IDEA integrated HTTP Client](https://www.vojtechruzicka.com/intellij-idea-tips-tricks-testing-restful-web-services/)
- [Handling responses in the HTTP Client](https://blog.jetbrains.com/phpstorm/2018/04/handling-reponses-in-the-http-client/)
- [HTTP Client reference](https://www.jetbrains.com/help/idea/http-client-reference.html)
- [HTTP Response reference](https://www.jetbrains.com/help/idea/http-response-reference.html)
