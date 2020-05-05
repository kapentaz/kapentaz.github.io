---
title: "@ManyToOne의 N+1 문제 원인 및 해결"
last_modified_at: 2020-05-05T17:55:00+00:00
header:
  show_overlay_excerpt: false
  overlay_image: /assets/images/post/2020/05/2020-05-05-title.jpg
  og_image: /assets/images/post/2020/05/2020-05-05-title.jpg
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
```

## Direct fetching
### left join을 inner join으로 변경
`Employee`를 Direct fetching 조회하면 Hibernate는 left  join으로 쿼리를 생성합니다. @ManyToOne의 default FetchType은 EAGER입니다.

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
먄약 inner join으로 변경하고 싶다면 @ManyToOne의 optional을 false로 변경하면 됩니다. 기본값은 true 입니다.
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

### FetchType.LAZY
company 정보가 필요 없는 상황에서도 매번 조회하는 것은 비효율적입니다. 필요에 따라 조회할 수 있도록 FetchType을 LAZY로 변경하고 다시 조회해보겠습니다.
```kotlin
@ManyToOne(optional = false, fetch = FetchType.LAZY)  
@JoinColumn(name = "company_no")  
val company: Company
```

```kotlin
entityManager.find(Employee::class.java, 1L)
```

employee 조회 쿼리만 실행되는 것을 확인할 수 있습니다.
```sql
select
    ...
from
    employee employee0_ 
where
    employee0_.employee_no=?
```

## Entity query fetching

실제 JPA를 사용하면 Direct fetching보다는 Entity query fetching을 해야 하는 경우가 많습니다. @ManyToOne을 다시 기본값으로 변경하고 조회해보겠습니다.

```kotlin
entityManager.createQuery(
    """
    select e
    from Employee e
    where e.employeeNo = :employeeNo
""".trimIndent(), Employee::class.java
).setParameter("employeeNo", 1L).singleResult
```
실행된 쿼리를 확인해보면 select가 2번 실행된 것을 확인할 수 있습니다.
```sql
select
    ...
    
from
    employee employee0_ 
where
    employee0_.employee_no=?

select
    ...
from
    company company0_ 
where
    company0_.company_no=?
```
왜 쿼리가 2번 실행될 까요? 그 이유는 HQL로 조회를 하게 되면 **mapping 관계에서 설정한 FetchType이 동작하지 않습니다.** 

employee를 먼저 조회한 후에 **관계 설정된 company 정보를 얻기 위해 추가 쿼리가 실행되는 것**입니다. Hibernate는 Collection 관계가 아니라면 정보를 얻기 위해 무조건 추가 쿼리를 실행합니다. 다만 employee의 company_no가 null 일 경우에는 조회할 데이터 자체가 없으므로 추가 쿼리 실행을 하지 않습니다.

### N + 1
위 예제 같이 한 employee가 아닌 여러 employee를 조회하고 그 company 정보가 여러개일 경우 그 수만큼 추가 select 쿼리가 실행됩니다. 즉 N+1 문제가 발생하게 됩니다.  이 문제는 위에서 말한대로 `@ManyToOne`관계에서 FetchType.LAZY으로 설정 한다고 해도 달라지는건 없습니다.  

해결 방법으로는 fetch join 또는 entity graphs가 있는데 fetch join에 관해서 확인해보겠습니다. 

```kotlin
entityManager.createQuery(
    """
    select e
    from Employee e
    join fetch e.company
""".trimIndent(), Employee::class.java
).resultList
```
join fetch를 직접 지정해서 join 쿼리가 실행되도록 할 수 있습니다. left outer join으로 처리하고 싶다면 `join fetch` 대신 `left join fetch`로 변경하면 됩니다.

#### Criteria

Criteria를 이용하는 경우에도 마찬가지로  mapping 관계에서 설정한 FetchType이 동작하지 않습니다. 이를 해결하기 위해서는 root.fetch() 메서드를 이용해서 join 쿼리를 생성해야 합니다.

```kotlin
val builder = entityManager.criteriaBuilder  
val criteria: CriteriaQuery<Employee> = builder.createQuery(Employee::class.java)  
val root: Root<Employee> = criteria.from(Employee::class.java)  

root.fetch<Employee, Company>(Employee::company.name, JoinType.INNER)  
  
val select: CriteriaQuery<Employee> = criteria.select(root)  
entityManager.createQuery(select).resultList
```

```sql
select
    ...
from
    employee employee0_ 
inner join
    company company1_ 
        on employee0_.company_no=company1_.company_no
```

끝.

## Reference
- [Fetching](https://docs.jboss.org/hibernate/orm/5.4/userguide/html_single/Hibernate_User_Guide.html#fetching)
- [Hibernate is making extra SQL statement with @ManyToOne and @Lazy fetching object](https://stackoverflow.com/questions/59849508/hibernate-is-making-extra-sql-statement-with-manytoone-and-lazy-fetching-objec)
- [N+1 query problem with @ManyToOne Association](https://discourse.hibernate.org/t/n-1-query-problem-with-manytoone-association/1293)