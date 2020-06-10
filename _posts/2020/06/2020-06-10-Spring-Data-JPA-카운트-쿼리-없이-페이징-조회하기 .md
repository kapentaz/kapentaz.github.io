---
title: "Spring Data JPA 카운트 쿼리 없이 페이징 조회하기"
last_modified_at: 2020-06-10T23:40+00:00
header:
  show_overlay_excerpt: false
  overlay_image: /assets/images/post/2020/06/2020-06-10-title.jpg
  og_image: /assets/images/post/2020/06/2020-06-10-title.jpg
  overlay_filter: 0.6
  caption: "Photo by Luis Quintero on Unsplash"
  
tags:
  - JPA
category: #카테고리
  - JPA
toc: false
toc_label: "Table Of Contents"
toc_icon: "fal fa-list-alt"
toc_sticky: true
classes: wide
comments: true
---



Spring Data JPA를 이용하면 쉽게 페이징 처리를 할 수 있습니다. `JpaRepository` 인터페이스를 상속받으면 Spring에서 제공하는 페이징 관련 기능을 이용할 수 있습니다.

## 페이징 처리를 위한 카운트 쿼리 실행

JpaRepository 인터페이스가 상속받은 PagingAndSortingRepository에 있는 findAll 메서드 이용하는 방법입니다. 이 메서드는 검색 조건 없이 전체를 대상으로 조회를 합니다.
```kotlin
val pageRequest = PageRequest.of(1, 10)  
val page: Page<Product> = productRepository.findAll(pageRequest)```
```
findAll의 리턴 타입이 Page<Product>이고 Page 객체에 카운트 정보를 만들기 위해 카운트 쿼리를 추가로 실행합니다.

### Query by Example
Entity를 이용해서 검색 조건으로 사용하는 Query by Example도  Page<Product>를 리턴하기 때문에 카운트 쿼리를 추가로 실행합니다.
```kotlin
val product = Product(name = "검색이름")  
val example = Example.of(product)  
val pageRequest = PageRequest.of(1, 10)  
val page: Page<Product> = productRepository.findAll(example, pageRequest)
```

### Specification
JpaSpecificationExecutor 인터페이스에서 제공하는 findAll  메서드를 이용하는 경우에도 Page<Product>를 리턴하기 때문에 카운트 쿼리를 추가로 실행합니다.

```kotlin
val spec = Specification<Product> { root, _, builder ->  
  builder.equal(root.get<String>(Product::name.name), "a")  
}  
val pageRequest = PageRequest.of(1, 10)  
val page: Page<Product> =productRepository.findAll(spec, pageRequest)
````

## 카운트 쿼리 없이 페이징 조회하기

앞서 살펴본 방법은 페이징 처리를 위해서 카운트 쿼리를 실행했습니다. 하지만 상황에 따라서는 이 카운트 쿼리가 불필요할 수 있습니다. 최초에 한 번만 카운트 조회하고 이후에는 원하는 페이지로 이동만 하는 경우가 대표적인 사례일 것입니다.

### Query Methods

Spring Data JPA Query Methods 기능을 이용해서 리턴 타입을 List로 하면 카운트 쿼리 없이 실행합니다.
```kotlin
fun findAllBy(pageable: Pageable): List<Product>
```

### @Query
@Query 애노테이션을 이용해서 리턴 타입을 List로 하면 역시 카운트 쿼리 없이 실행합니다.
```kotlin
@Query("select p from Product p")  
fun findAllPage(pageable: Pageable): List<Product>
```

### Specification
Specification을 이용해서 카운트 쿼리 없이 페이징 처리를 하려면 인터페이스에서 바로 해결할 수는 없고 직접 구현 코드를 작성해야 합니다. 

Repository 별로 직접 구현하는 것보다는 Customize the Base Repository 기능을 이용해서 공통으로 만들어서 사용하는 게 좋습니다.

```kotlin
@NoRepositoryBean  
interface BaseRepository<T, ID : Serializable?> : JpaRepository<T, ID>, JpaSpecificationExecutor<T> {  
  fun findByPage(spec: Specification<T>, pageable: Pageable): List<T>
}
```
```kotlin
class BaseRepositoryImpl<T, ID>(  
    private val entityInformation: JpaEntityInformation<T, *>?,  
 protected val em: EntityManager  
) : SimpleJpaRepository<T, ID>(entityInformation!!, em), BaseRepository<T, ID> {  
  
  override fun findByPage(spec: Specification<T>, pageable: Pageable): List<T> {
    val query = getQuery(spec, pageable)
    query.firstResult = pageable.offset.toInt()
    query.maxResults = pageable.pageSize
    return query.resultList
  }
}
```
findByPage라는 커스텀 메서드를 만들어서 specification을 이용해서 페이징 조회를 하지만 카운트 쿼리는 실행하지 않도록 처리했습니다.


```kotlin
val spec = Specification<Product> { root, _, builder ->  
  builder.equal(root.get<String>(Product::name.name), "상품이름")  
}  
val pageRequest = PageRequest.of(1, 10)  
val list: List<Product> = productRepository.findByPage(spec, pageRequest)
```

## 결론
성능이 중요하지 않다면 심플하게 Page 객체로 반환하는 구조도 괜찮을 수 있지만, 매번 카운트 쿼리가 필요한 상황이 아니며, 쿼리 한 번이라도 줄일 필요가 있다면 페이징 처리와 카운트 쿼리는 필요에 따라 실행할 수 있는 구조로 하는 게 좋을 것 같습니다.

끝.