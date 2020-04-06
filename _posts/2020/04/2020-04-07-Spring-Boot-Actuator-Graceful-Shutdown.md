---
title: "Spring Boot Actuator Graceful Shutdown"
last_modified_at: 2020-04-07T19:29:00+00:00
header:
  show_overlay_excerpt: false
  overlay_image: /assets/images/post/2020/04/2020-04-07-title.jpg
  og_image: /assets/images/post/2020/04/2020-04-07-title.jpg
  overlay_filter: 0.6
  caption: "Photo by Alec Favale on Unsplash"
  
tags:
  - spring
category: #카테고리
  - Spring
toc: false
toc_label: "Table Of Contents"
toc_icon: "fal fa-list-alt"
toc_sticky: true
classes: wide
comments: true
---


Spring Boot 환경에서 application을 shutdown 하는 방법 중 대표적인 것이 actuator의 shutdown endpoint 기능을 이용하는 것입니다. 이 endpoint는 예상과 달리 처리 중인 요청이 있더라도 그냥 shutdown 처리를 합니다. 정말로 그런지 확인해보겠습니다.

## graceful 하지 않는 shutdown
먼저 actuator shutdown endpoint를 사용하려면 설정을 통해서 활성화시켜야 합니다.

```yml
# actuator
management:
  endpoint:
    shutdown:
      enabled: true
  endpoints:
    web:
      exposure:
        include: "*"
```

> shutdown endpoint가 외부로 공개되면 누구나 실행할 수 있기 때문에 권한 설정이나 내부에서만 실행 가능한 구조로 만들어야 합니다

위와 같이 `application.yml`에 설정을 추가하고 

shutdown endpoint를 호출하면 "Shutting down, bye..."라는 응답과 함께 application도 종료된 것을 확인할 수 있습니다. 

```http
POST http://localhost:8888/actuator/shutdown

HTTP/1.1 200 
Content-Type: application/vnd.spring-boot.actuator.v3+json
Transfer-Encoding: chunked
Date: Sun, 05 Apr 2020 01:52:44 GMT
Keep-Alive: timeout=60
Connection: keep-alive

{
  "message": "Shutting down, bye..."
}
```
그렇다면 요청받은 내용을 좀 오랫동안 처리 중인 스레드가 있을 경우에 `/actuator/shutdown`을 호출하면 어떻게 될까요?

먼저 아래와 같이 Controller를 만들고
```java
@Slf4j
@RestController
public class ShutdownRestController {

    @GetMapping(value = "/long-process")
    public void process(@RequestParam(value = "sec") int sec) {
        for (; sec > 0; sec--) {
            log.info("{}", sec);
            try {
                Thread.sleep(1000);
            } catch (InterruptedException ignored) {
            }
        }
    }
}
```

30초간 대기하도록 호출한 다음에

```http
GET http://localhost:8080/long-process?sec=30
```
shutdown을 호출하면 
```
POST http://localhost:8888/actuator/shutdown
```

좀 전과 동일하게 "Shutting down, bye..." 메시지와 함께 application이 즉시(0.5초 후에) 종료됩니다. 
```json
{
  "message": "Shutting down, bye..."
}
```

예상한 기대 동작은 `/actuator/shutdown`을 호출한 이후에 새로운 요청은 더 이상 처리하지 않지만 내부적으로 처리 중인 작업은 완료하고 종료될 것이라 생각했지만 실제로는 그렇지 않습니다.

[Marcos Barbero's Blog](https://blog.marcosbarbero.com/graceful-shutdown-spring-boot-apps/) 블로그 내용을 확인해 보면 tomcat 환경에서 graceful shutdown 하는 방법에 대해서 소개하고 있습니다. 이 블로그에 나와 있는 내용대로 `GracefulShutdown` 클래스를 만들고 설정한 후 `/actuator/shutdown`을 호출하면 지정한 시간만큼 대기한 후에 application이 종료되는 것을 확인할 수 있습니다.


## Spring Reactive 환경에서 Shutdown
위에서 살펴본 블로그 예제는 Tomcat 환경을 기준으로 설명되어 있습니다. 

Spring Reactive 환경이라면 기본으로 Netty를 서버로 사용합니다. Netty일 경우에도 `/actuator/shutdown`를 호출하면 graceful shutdown 동작을 하지 않습니다. 해결 방법이 있을 것 같아 관련해서 구글 검색해봐도 자료가 많지도 않고 그나마 있는 것도 기대처럼 동작하지 않았습니다.

## Spring Boot 2.3.0 Graceful shutdown

Netty 환경에 대해서 직접 구현하기에는 이해하기 부족했기 때문에 좀 더 확인한 결과 Spring Boot 2.3.0에서 graceful shutdown을 지원한다고 합니다.

2020-04-07 기준으로 아직 M4 버전입니다. M3 버전의 [Release Notes](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-2.3.0-M3-Release-Notes)를 보면 Graceful shutdown 기능이 추가된 것을 확인할 수 있습니다.


`server.shutdown.grace-period` 옵션을 통해서 대기 시간을 설정할 수 있습니다. 
```yaml
server:  
  port: 8080  
  shutdown:  
    grace-period: 60000		# 최대 60초 대기
```

위와 같이 설정하고 아래와 같이 webClient로 40초 이상 소용되는 작업을 비동기 호출 하고 `/actuator/shutdown`을 호출하면 해당 작업이 완료된 이후에 서버가 종료되는 것을 확인할 수 있습니다.
```java
@Configuration
public class ShutdownRouter {

    @Bean
    public RouterFunction<ServerResponse> composedRoutes() {
        return route(GET("/long-process"), request -> {
            // 40초 소요되는 작업 호출
            Mono<String> result = webClient.get()
                    .uri("http://localhost:8888/long-process?sec=40")
                    .retrieve()
                    .bodyToMono(String.class);
            return ok().body(result, String.class);
        });
    }
}
```

보통 API에서 요청을 처리할 때는 길어도 수 초내에 처리를 끝나서 문제가 없겠지만 예외적으로 길게 처리하더라도 gracful shutdown 최대 대기 시간을 넘지 않도록 하는 게 좋을 것 같습니다.

끝.



## Reference
- [Shutdown a Spring Boot Application](https://www.baeldung.com/spring-boot-shutdown)
- [Graceful Shutdown of a Spring Boot Application](https://www.baeldung.com/spring-boot-graceful-shutdown)
- [Graceful Shutdown Spring Boot Applications](https://blog.marcosbarbero.com/graceful-shutdown-spring-boot-apps/)
- [Spring Boot Actuator Web API Documentation](https://docs.spring.io/spring-boot/docs/2.2.5.RELEASE/actuator-api//html/)
- [Spring Boot 2.3.0 M3 Release Notes](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-2.3.0-M3-Release-Notes)