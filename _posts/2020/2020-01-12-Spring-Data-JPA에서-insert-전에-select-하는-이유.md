---
title: "Spring Data JPA에서 Insert 전에 Select 하는 이유"
last_modified_at: 2020-01-12T01:10:00-05:00
header:
  show_overlay_excerpt: false
  overlay_image: /assets/images/background/fried-potato.jpg
  og_image: /assets/images/background/fried-potato.jpg
  overlay_filter: 0.6
  caption: "Photo Credit: [Brady](https://kapentaz.github.io)"
tags:
  - JPA
  - Bulk
category: #카테고리
  - JPA
toc: false
toc_label: "Table Of Contents"
toc_icon: "fal fa-list-alt"
toc_sticky: true
classes: wide
comments: true
---



[이전 글]([https://kapentaz.github.io/jpa/JPA-Batch-Insert-with-MySQL/])에서 bulk insert 처리를 할 수 있는 방법 중에 하나가 @Id 값 알고 있는 경우라고 했었는데요. 실제로 잘 되는지 확인하는 과정에 발생한 문제 내용을 공유합니다.

## Select before Insert
어떤 상품(Product)과 상품상세(ProductDetail)가 있는데 이 둘은 관계 설정이 되어 있지 않습니다. 그리고 상품상세 PK는 상품의 PK와 동일하다고 가정해보겠습니다. 상품상세의 entity는 아래와 같습니다.

```kotlin
@Entity  
@Table(name = "product_detail")  
data class ProductDetail(  
  
    @Id  
    @Column(name = "product_no")  
    val productNo: Int = 0,  
  
    @Column(name = "image_url")  
    val imageUrl: String,  
  
    @Column(name = "content")  
    val content: String  
)
```
@Id 값은 상품에서 이미 생성한  productNo를 사용하기 때문에 별도 generate 설정이 없습니다.  상품상세를 등록하려는 시점에는 당연히 productNo를 알고 있을 것입니다. 그럼 이제 상품상세를 등록해보겠습니다.
```kotlin
fun save() {  
  ProductDetail(1, "http://test.xxx/sample.jpg", "상품상세").let {  
    productDetailRepository.save(it)  
  }    
}
```
실행 로그를 확인해보면 정상적으로 insert 쿼리가 실행된 것을 확인할 수 있습니다. 하지만 그전에 select 쿼리가 한 번 실행된 로그가 있습니다.
```
select
        productdet0_.product_no as product_1_2_0_,
        productdet0_.content as content2_2_0_,
        productdet0_.image_url as image_ur3_2_0_ 
    from
        product_detail productdet0_ 
    where
        productdet0_.product_no=?
....
....
insert 
    into
        product_detail
        (content, image_url, product_no) 
    values
        (?, ?, ?)
```
`save()`를 실행했는데 insert 전에 왜 select 쿼리가 실행되는 걸까요? 이런 상황이라면 만약 ProductDetail를 bulk insert 하려고 10,000개를 `saveAll()` 하게 되면 10,000번의 select 쿼리가 실행될 텐데요. 끔찍하네요. 원인을 찾아야겠습니다.

## Persist or Merge
먼저 `CrudRepository` 구현체인 `SimpleJpaRepository` 에서 `save()` 메서드를 어떻게 처리하고 있는 살펴보겠습니다. 코드가 보이는대로 읽어보면 새로운 entity 이면 `persist()` 메서드를 아니면 `merge()` 메서드를  실행하는 것을 알 수 있습니다.
```java
@Transactional
public <S extends T> S save(S entity) {

	if (entityInformation.isNew(entity)) {
		em.persist(entity);
		return entity;
	} else {
		return em.merge(entity);
	}
}
```
우리는 새로운 상품상세를 등록하는 거니깐 `persist()` 메서드가 실행되어야겠죠? 하지만 결과는 아닙니다. `merge()`가 실행됩니다. 이유는 @Id 필드에 이미 값이 존재하기 때문입니다. entity의 @Id 값이 세팅되어 있기 때문에 이미 DB에 존재하는 데이터로 간주하고  insert를 하지 않고 update할 항목이 있는지 확인하기 위해 select가 실행되는 것입니다. 결국은 @Id 값으로 select 한 결과가 없기 때문에 insert 쿼리를 실행하는것입니다.

## Solution
@Id 값을 이미 알고 있을 때 `merge()`가 아니라 `persis()`를 실행할 수 있는 방법은 `Persistable`  인터페이스를 구현하는 것입니다.  2가지 구현해야 하는 메소드가 존재하는데요. 상품상세에 적용해 보겠습니다.
```kotlin
@Entity
@Table(name = "product_detail")
data class ProductDetail(

    @Id
    @Column(name = "product_no")
    val productNo: Int = 0,

    @Column(name = "image_url")
    val imageUrl: String,

    @Column(name = "content")
    val content: String
): Persistable<Int> {
    @Transient
    private var isNew = true

    override fun getId(): Int {
        return productNo
    }

    override fun isNew(): Boolean {
        return isNew
    }

    @PrePersist
    @PostLoad
    fun markNotNew() {
        isNew = false
    }
}
```
새로운 entity 여부를 결정하는 프로퍼티 `isNew`가 있고 기본값은 true로 설정합니다. `ProductDetail`를 저장하기 위해 새로운 객체를 생성하게 되면 기본값인 true가 되겠죠. 그리고 @PrePersist, @PostLoad를 통해서 영속성 객체로 만들어질 때 false로 변경하여 DB에 저장하거나 불러온 시점 이후부터는 새로운 entity로 인식하지 않게 처리힙니다.

## 결론
사실 정확히 표현하자면 bulk insert 처리 자체에는 문제가 없습니다. 로그를 확인해보면 multi value 형태로 insert가 잘 실행됩니다. 다만 영속성 객체로 만들기 위해 하나 개별 select 쿼리가 추가 실행됩니다. 9이제 이런 현상이 왜 발생하는지 살펴보았으니 혹시 insert 전에 select가 발생한다면 잘 해결할 수 있을 것입니다.

끝.


## Reference

 - [Saving Entities](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#jpa.entity-persistence.saving-entites)
