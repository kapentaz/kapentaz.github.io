---
title: "@ManyToOne의 N+1 문제 원인 및 해결"
last_modified_at: 2020-05-10T02465:00+00:00
header:
  show_overlay_excerpt: false
  overlay_image: /assets/images/post/2020/05/2020-05-10-title.jpg
  og_image: /assets/images/post/2020/05/2020-05-10-title.jpg
  overlay_filter: 0.6
  caption: "Photo by K. Mitch Hodge on Unsplash"
  
tags:
  - JPA
category: #카테고리
  - JPA
  - Hibernate
toc: false
toc_label: "Table Of Contents"
toc_icon: "fal fa-list-alt"
toc_sticky: true
classes: wide
comments: true
---


# @ManyToOne의 N+1 문제 원인 및 해결

Hibernate에서 @ManyToOne의 FetchType을 LAZY로 설정해도 추가 쿼리가 실행되는 이유와 N+1 현상이 발생하는 과정을 확인해보겠습니다.

직원 `Employee`와 회사 `Company`가 있고 이 둘 사이는 N:1 관계이기 때문에 `Employee`에 @ManyToOne을 설정해 보겠습니다.  참고로 DB Table 기준으로 본다면 employee가 company_no 컬럼을 가지고 있기 때문에 이 관계의 주인이라고 할 수 있습니다. 
```kotlin
@Entity
@Table(name = "company")
data class Company(
    @Id
    @Column(name = "company_no")
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val companyNo: Long = 0
) {
    @Column(name = "name")
    var name: String = ""
}
```
```kotlin
@Entity
@Table(name = "employee")
data class Employee(
    @Id
    @Column(name = "employee_no")
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val employeeNo: Long = 0,

    @ManyToOne
    @JoinColumn(name = "company_no")
    val company: Company
) {
    @Column(name = "name")
    var name: String = ""
}
```
그리고 테스트를 위해 필요한 기본 데이터를 insert 하겠습니다.
```sql
insert into company(name) values ('대박회사');  
insert into employee(name, company_no) VALUES ('김직원', 1);  
insert into employee(name, company_no) VALUES ('이직원', 1);  
insert into company(name) values ('우량회사');  
insert into employee(name, company_no) VALUES ('박직원', 2);  
insert into employee(name, company_no) VALUES ('전직원', 2);
```

## FetchType.EAGER

@ManyToOne의 기본 FetchType은 EAGER이기 때문에 관계에 필요한 데이터를 바로 조회 합니다. `Employee`를 Direct fetching 조회하면 Hibernate는 left  join으로 쿼리를 생성해서 조회합니다. 

```kotlin
entityManager.find(Employee::class.java, 1L)
```
```sql
select
    ...
from
    employee employee0_ 
left outer join
    company company1_ 
        on employee0_.company_no=company1_.company_no 
where
    employee0_.employee_no=?
```
inner join으로 변경하고 싶다면 @ManyToOne의 optional을 false로 변경하면 됩니다. 기본값은 true 입니다.
```kotlin
@ManyToOne(optional = false)
@JoinColumn(name = "company_no")
val company: Company
```
다시 실행하면 inner join으로 변경된 것을 확인할 수 있습니다.
```sql
select
    ...
from
    employee employee0_ 
inner join
    company company1_ 
        on employee0_.company_no=company1_.company_no 
where
    employee0_.employee_no=?
```

### N+1
앞서 하나의 `Employee`를 조회해봤고 이번에는 JQPL을 이용해서 전체 `Employee`를 조회해 보겠습니다. 
```kotlin
entityManager.createQuery(
    """
    select e
    from Employee e
""".trimIndent(), Employee::class.java
).resultList
```

총 3개의 쿼리가 실행됩니다.  `Employee`를 조회하기 위한 쿼리와 `Employee`와 관계 있는 `Company`를 조회하기 위한 쿼리입니다.
```sql
select
    ...
from
    employee employee0_

select
    ... 
from
    company company0_ 
where
    company0_.company_no=?

select
    ... 
from
    company company0_ 
where
    company0_.company_no=?
```
`Employee`가 속한 `Company` 만큼 추가 쿼리가 실행되기 때문에 이와 같은 현상을 N+1이라고 합니다. 

이런 현상이 생기는 이유는 예제에서 JPQL로 `Employee`만 조회했고 조회한 이후에  @ManyToOne 관계를 확인하게 되는데 FetchType이 EAGER이기 때문에 뒤늦게   `Company` 를 가져오기 위해 추가 쿼리가 실행되는 것입니다.

## FetchType.LAZY

만약 FetchType을 LAZY로 변경하게 되면 이런 현상은 발생하지 않습니다.  `Company` 를 proxy 객체로 가지고 있고 필요한 시점에 쿼리를 호출하기 때문입니다. 

> 참고로 JPA에서 기본이 EAGER로 되어 있기 때문에 Hibernate에서도 스펙에 따르고 있지만 기본적으로 LAZY로 변경해서 사용하길 권장하고 있습니다.

FetchType.LAZY로 변경했을 때 주의점은 트랜잭션에 내에서 toString()이나 property 복사 같은 작업을 하면서 `Employee`의 `Company`에 접근하게 되면 추가 쿼리가 실행되면서 역시 N+1이 발생하게 됩니다. 

트랜잭션 밖에서 `Employee`의 `Company`에 접근하게 되면 `LazyInitializationException` 오류가 발생합니다.

### Fetch Join

1개 이상의 `Employee`를 조회할 때는 N+1 발생 가능성에 대해서 항상 인지하고 있어야합니다. @ManyToOne의 FetchType을 LAZY로 설정했지만 `Company`까지 필요한 경우에는 Fetch Join을 이용합니다.

```kotlin
entityManager.createQuery(
    """
    select e
    from Employee e
    join fetch e.company
""".trimIndent(), Employee::class.java
).resultList
```
`join fetch`으로 join 쿼리가 실행되도록 할 수 있습니다. left outer join으로 처리하고 싶다면 `join fetch` 대신 `left join fetch`로 변경하면 됩니다.

### Kotlin All-open plugin

kotlin은 기본이 final 클래스입니다. LAZY 동작이 정상적으로 되려면 Hibernate가 프록시 객체 생성할 수 있도록 Entity는 open 클래스 이어야 합니다. 직접 하나 하나 지정할 수도 있고 All-open 플러그인을 이용해서 한번에 처리할 수 있습니다. All-open에 대한 자세한 내용은 [여기](https://kotlinlang.org/docs/reference/compiler-plugins.html#all-open-compiler-plugin)를 참고하세요.

## 결론
@ManyToOne의 기본 FetchType은 EAGER이지만 특별한 경우가 아니라면 LAZY를 기본 세팅해서 사용하는게 좋습니다. LAZY로 사용하더라도 N+1이 발생할 수 있음을 인지하고 사용해야하며 필요한 경우에는 fetch join이나 설명은 안 했지만 entity graphs를 이용해서 한번에 데이터를 모두 조회하도록 합니다.

끝.


## Reference
- [Fetching](https://docs.jboss.org/hibernate/orm/5.4/userguide/html_single/Hibernate_User_Guide.html#fetching)
- [Hibernate is making extra SQL statement with @ManyToOne and @Lazy fetching object](https://stackoverflow.com/questions/59849508/hibernate-is-making-extra-sql-statement-with-manytoone-and-lazy-fetching-objec)
- [N+1 query problem with @ManyToOne Association](https://discourse.hibernate.org/t/n-1-query-problem-with-manytoone-association/1293)
- [JPA entity written in Kotlin fetches association while not expected (Java equivalent works as expected though)](https://youtrack.jetbrains.com/issue/KT-28525)