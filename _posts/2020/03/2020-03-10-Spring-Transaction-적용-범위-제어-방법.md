---
title: "Spring Transaction 적용 범위 제어 방법"
last_modified_at: 2020-03-10T23:09:00+00:00
header:
  show_overlay_excerpt: false
  overlay_image: /assets/images/post/2020/03/2020-03-10-hand-shake.jpg
  og_image: /assets/images/post/2020/03/2020-03-10-hand-shake.jpg
  overlay_filter: 0.6
  caption: "Photo by Cytonn Photography on Unsplash"
  
tags:
  - Spring
  - Transaction
category: #카테고리
  - Spring
toc: false
toc_label: "Table Of Contents"
toc_icon: "fal fa-list-alt"
toc_sticky: true
classes: wide
comments: true
---



Spring을 사용하면 트랜잭션을 설정하기 위해 보통은 `@Transactional` 애노테이션을 많이 사용합니다. `@Transactional`은 클래스나 메서드에 지정해서 사용하게 됩니다. 그러다 보니 트랜잭션 적용 범위를 제어하기가 힘들 수 있는데요. 이럴 경우에 사용할 수 있는 방법에 대해서 확인해보겠습니다.

간단한 예제를 통해서 확인해보겠습니다. 아래와 같은 `update()` 메서드가 있습니다. 상품을 조회하고 업데이트 이후에 API 호출을 통해서 email 전송을 합니다. 

> 단순 조회 후 업데이트하기 때문에 트랜잭션이 필요한 건 아니지만 필요한 경우라고 가정하겠습니다.

```java
@Transactional
public void update(long productNo, String name, int price) {
    Optional<Product> product = productRepository.findById(productNo);
    product.ifPresent(p -> {
        p.setName(name);
        p.setPrice(price);
        productRepository.save(p);
    });
    // email 발송
    restTemplate.postForEntity("...", null, Void.class);
}
```
만약 상품 업데이트는 email 발송 성공/실패 여부와 관계없이 commit 되어야 한다면 email 발송하는 부분을 `@Transactional` 범위 밖으로 분리를 시켜키는 것이 좋습니다.  email 발송을 호출하는 동안에 DB 커넥션을 유지할 필요가 없기 때문입니다. 상품을 DB 업데이트하는 부분과 email 발송하는 부분을 분리해보겠습니다.

```java
public void update(long productNo, String name, int price) {  
    updateProduct(productNo, name, price);  
    sendEmail();  
}  
  
@Transactional
public void updateProduct(long productNo, String name, int price) {
    Optional<Product> product = productRepository.findById(productNo);
    product.ifPresent(p -> {
        p.setName(name);
        p.setPrice(price);
        productRepository.save(p);
    });
} 
  
private void sendEmail() {  
    restTemplate.postForEntity("...", null, Void.class);  
}
```
위 코드에는 문제가 있습니다. `update()` 메서드에서 `updateProduct()` 메서드를 호출하는 과정에 proxy 객체를 이용하지 않고 내부에서 호출하기 때문입니다. `@Transactional` 애노테이션이 제대로 동작하려면 internal 메서드 호출 방식을 사용하면 안 됩니다. 이를 해결할 수 있는 몇 가지 방법을 소개하겠습니다.

## Self Injection

자기 자신을 Spring Bean으로 Injection 하는 방법입니다. 이렇게 주입받는 Bean은 생성자 주입 방식으로는 처리가 불가능하기 때문에 `@Autowired`로 적용했습니다.
```java
@Service
@RequiredArgsConstructor
public class TransactionSample {
    private final RestTemplate restTemplate;
    private final ProductRepository productRepository;
    @Autowired
    private TransactionSample self;

    public void update(long productNo, String name, int price) {
        self.updateProduct(productNo, name, price);
        self.sendEmail();
    }
    ...
}
```
자신을 다시 Bean으로 주입받는 과정으로 인해 생성자 주입 방식을 사용할 수 없고 `update()` 처리를 하기 위해서 별도의 메서드 2개를 추가해야 하는 부분이 조금 아쉽습니다.

## TransactionTemplate 

programmatic transaction 설정을 위해서 Spring에서 제공하는 클래스입니다.  thread-safe 하기 때문에 Spring Bean으로 정의해서 사용할 수 있습니다. 
```java
@Bean  
public TransactionTemplate transactionTemplate(PlatformTransactionManager manager) {  
    return new TransactionTemplate(manager);  
}
```
약간 불편할 수 있는 부분은 `TransactionDefinition`을 다른 설정으로 바꾸기 싶다면 추가로 Bean을 정의해야 합니다. 아니면 사용하는 곳에서 직접 객체를 생성하면 됩니다.
```java
@Service
public class TransactionSample {
    private final RestTemplate restTemplate;
    private final ProductRepository productRepository;
    private final TransactionTemplate transactionTemplate;

    public TransactionSample(RestTemplate restTemplate, 
                             ProductRepository productRepository, 
                             PlatformTransactionManager manager) {
        this.restTemplate = restTemplate;
        this.productRepository = productRepository;
        this.transactionTemplate = new TransactionTemplate(manager,
                new DefaultTransactionDefinition(TransactionDefinition.PROPAGATION_REQUIRED));
    }

    public void update(long productNo, String name, int price) {
        transactionTemplate.executeWithoutResult(status -> {
            Optional<Product> product = productRepository.findById(productNo);
            product.ifPresent(p -> {
                p.setName(name);
                p.setPrice(price);
                productRepository.save(p);
            });
        });
        // email 발송
        restTemplate.postForEntity("...", null, Void.class);
    }
}
```

## @Transactional을 이용한 클래스 만들기

`@Transactional` 애노테이션을 이용해서 별도 TransactionHandler 클래스를 만드는 방법입니다. 익숙한 `@Transactional` 애노테이션을 그대로 활용해서 메서드 내에서 원하는 영역에만 트랜잭션을 적용할 수 있습니다.
```java
@Component
public class TransactionHandler {
    @Transactional(propagation = Propagation.REQUIRED)
    public void runInTransaction(Action action) {
        action.act();
    }

    public interface Action {
        void act();
    }
}
```
```java
public void updateProduct(long productNo, String name, int price) {
    transactionHandler.runInTransaction(() -> {
        Optional<Product> product = productRepository.findById(productNo);
        product.ifPresent(p -> {
            p.setName(name);
            p.setPrice(price);
            productRepository.save(p);
        });
    });
    // email 발송
    restTemplate.postForEntity("...", null, Void.class);
}
```
`@Transactional` 애노테이션이 제공해 주던 기능을 그대로 활용할 수 있습니다. return 값이 필요한 경우와 속성이 다른 경우 대해서 메서드를 추가해서 사용하면 됩니다.

## 결론
다른 고민 없이 심플하게 처리하는 방법은 self injection이고 Spring에서 제공하는 programmatic transaction 방법을 이용하려면 `TransactionTemplate`을 이용하면 됩니다. `@Transacional` 애노테이션을 이용해서 별도 클래스를 만들어서 사용할 수도 있습니다. 자신의 환경과 선호하는 방법에 맞춰서 선택해서 사용하면 됩니다.

끝.

## Reference
- [Spring @Transaction method call by the method within the same class, does not work?](https://stackoverflow.com/questions/3423972/spring-transaction-method-call-by-the-method-within-the-same-class-does-not-wo)
- [Using the TransactionTemplate](https://docs.spring.io/spring/docs/4.2.x/spring-framework-reference/html/transaction.html#tx-prog-template)