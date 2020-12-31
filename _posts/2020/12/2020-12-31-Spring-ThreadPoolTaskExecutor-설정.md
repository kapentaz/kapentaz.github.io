---
title: "Spring ThreadPoolTaskExecutor 설정"
last_modified_at: 2020-12-31T20:54+00:00
header:
  show_overlay_excerpt: false
  overlay_image: /assets/images/post/2020/12/2020-12-31-title.jpg
  og_image: /assets/images/post/2020/12/2020-12-31-title.jpg
  overlay_filter: 0.6
  caption: "Photo by Alfons Morales on Unsplash"
  
tags:
  - Spring
category: #카테고리
  - Spring
toc: false
toc_label: "Table Of Contents"
toc_icon: "fal fa-list-alt"
toc_sticky: true
classes: wide
comments: true
---



ThreadPoolTaskExecutor는 이름에서 알 수 있듯이 스레드 풀을 사용하는 Executor입니다. 상위 인터페이스를 확인해 보면 `java.util.concurrent.Executor`를 Spring에서 구현한 것을 확인할 수 있습니다. 이 스레드 풀을 사용할 때 설정에 몇 가지 주의점이 필요합니다. 한번 확인해보겠습니다.

## 스레드 설정
```java
@Bean("simpleTaskExecutor")
public TaskExecutor taskExecutor() {
    ThreadPoolTaskExecutor taskExecutor = new ThreadPoolTaskExecutor();
    taskExecutor.setCorePoolSize(5);	// 기본 스레드 수
    taskExecutor.setMaxPoolSize(10);	// 최대 스레드 수
    return taskExecutor;
}
```

core와 max 사이즈를 설정할 수 있습니다. 여기서 주의할 점이 있습니다. 최초 core 사이즈만큼 동작하다가 더 이상 처리할 수 없을 경우 max 사이즈만큼 스레드가 증가할 것이라고 예상할 수 있지만 **사실 그렇지 않습니다**.

내부적으로는 Integer.MAX_VALUE 사이즈의 LinkedBlockingQueue를 생성해서 core 사이즈만큼의 스레드에서 task를 처리할 수 없을 경우 queue에서 대기하게 됩니다. queue가 꽉 차게 되면 그때 max 사이즈만큼 스레드를 생성해서 처리하게 됩니다.

## Capacity
core 사이즈 보다 많은 요청이 발생할 경우 Integer.MAX_VALUE 사이즈만큼의 queue를 이용한다고 했는데 이게 너무 크다고 생각된다면 queueCapacity 사이즈를 변경할 수 있습니다.

```java
@Bean("simpleTaskExecutor")
public TaskExecutor taskExecutor() {
    ThreadPoolTaskExecutor taskExecutor = new ThreadPoolTaskExecutor();
    taskExecutor.setCorePoolSize(5);	// 기본 스레드 수
    taskExecutor.setMaxPoolSize(10);	// 최대 스레드 수
    taskExecutor.setQueueCapacity(100);	// Queue 사이즈
    return taskExecutor;
}
```

위와 같이 설정한다면 최초 5개의 스레드에서 처리하다가 처리 속도가 밀릴 경우 100개 사이즈 queue에서 대기하고 그 보다 많은 요청이 발생할 경우 최대 10개 스레드까지 생성해서 처리하게 됩니다.

## RejectedExecutionHandler

max 스레드까지 생성하고 queue까지 꽉 찬 상태에서 추가 요청이 오면 `RejectedExecutionException`예외가 발생합니다. 더 이상 처리할 수 없다는 오류인데요.  오류가 발생하는 걸 손놓고 지켜봐야만 하는 건 아닙니다. 우리에게는 몇 가지 선택권이 있습니다.

기본적으로 `RejectedExecutionHandler` 인터페이스를 구현한 몇가지 클래스가 제공됩니다.
- **AbortPolicy**
	- 기본 설정
	- RejectedExecutionException을 발생시킵니다.
- **DiscardOldestPolicy**
	- 오래된 작업을 skip 합니다. 
	- 모든 task가 무조건 처리되어야 할 필요가 없을 경우 사용합니다.
-  **DiscardPolicy**
	- 처리하려는 작업을 skip 합니다.
	- 역시 모든 task가 무조건 처리되어야 할 필요가 없을 경우 사용합니다.
- **CallerRunsPolicy**
	- shutdown 상태가 아니라면 ThreadPoolTaskExecutor에 **요청한 thread에서 직접 처리**합니다.

예외와 누락 없이 최대한 처리하려면 `CallerRunsPolicy`로 설정하는 것이 좋을 것 같습니다.
 
```java
@Bean("simpleTaskExecutor")
public TaskExecutor taskExecutor() {
    ThreadPoolTaskExecutor taskExecutor = new ThreadPoolTaskExecutor();
    taskExecutor.setCorePoolSize(5);	// 기본 스레드 수
    taskExecutor.setMaxPoolSize(10);	// 최대 스레드 수
    taskExecutor.setQueueCapacity(100);	// Queue 사이즈
    taskExecutor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
    return taskExecutor;
}
```

## Shutdown
이렇게 별도로 정의한 스레드 풀에서 열심히 작업이 이루어지고 있을 때 애플리케이션 종료를 요청하면 어떻게 될까요?  Spring Boot Actuator를 이용해서 종료를 시켜보면  호출 즉시 application이 **바로 종료**되는 것을 확인할 수 있습니다.
```http
POST http://localhost:8888/actuator/shutdown
```
이렇게 즉시 종료되면 아직 처리되지 못한 task는 유실되게 됩니다. 유실 없이 마지막까지 다 처리하고 종료되길 원한다면 설정을 추가해야 합니다.

```java
@Bean("simpleTaskExecutor")
public TaskExecutor taskExecutor() {
    ThreadPoolTaskExecutor taskExecutor = new ThreadPoolTaskExecutor();
    taskExecutor.setCorePoolSize(5);	// 기본 스레드 수
    taskExecutor.setMaxPoolSize(10);	// 최대 스레드 수
    taskExecutor.setQueueCapacity(100);	// Queue 사이즈
    taskExecutor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
    taskExecutor.setWaitForTasksToCompleteOnShutdown(true);
    return taskExecutor;
}
```
waitForTasksToCompleteOnShutdown 을 true로 하게 되면 queue에 남아 있는 모든 작업이 완료될 때까지 기다리게 됩니다. 

### Timeout
만약 모든 작업이 처리되길 기다리기 힘든 경우라면 최대 종료 대기 시간을 설정할 수 있습니다.
```java
@Bean("simpleTaskExecutor")
public TaskExecutor taskExecutor() {
    ThreadPoolTaskExecutor taskExecutor = new ThreadPoolTaskExecutor();
    taskExecutor.setCorePoolSize(5);	// 기본 스레드 수
    taskExecutor.setMaxPoolSize(10);	// 최대 스레드 수
    taskExecutor.setQueueCapacity(100);	// Queue 사이즈
    taskExecutor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
    taskExecutor.setWaitForTasksToCompleteOnShutdown(true);
    taskExecutor.setAwaitTerminationSeconds(60);	// shutdown 최대 60초 대기
    return taskExecutor;
}
```

## 결론
`ThreadPoolTaskExecutor` 설정을 제대로 알고 사용하지 않을 경우 예상과 다른 퍼포먼스, 오류 발생과  task 유실이 발생할 수 있으니 제대로 확인하고 사용할 필요가 있습니다.

끝. 