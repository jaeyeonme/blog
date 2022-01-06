---
title: 객체지향 쿼리 언어 - 기본 문법
date: 2021-12-30 10:00:00
featuredImage: /images/JPA/jpa.png
categories:
  - JPA
tags:
  - JPA
  - Hibernate
---
## 1. 소개

---

**JPA는 다양한 쿼리 방법을 지원**
- JPQL
- JPA Criteria
- QueryDSL
- Native  SQL
- JDBC API 직접 사용, MyBatis, SpringJdbcTemplate 함께 사용



<br>



**JPQL 소개**
- 가장 단순한 조회 방법
  - EntityManager.find()
  - 객체 그래프 탐색 (a.getB().getC())
- 나이가 18살 이상인 회원을 모두 검색하고 싶다면?

<br>

**JPQL - 필요성**
- JPA를 사용하면 엔티티 객체를 중심으로 개발
- 문제는 검색 쿼리
- 검색을 할 때도 `테이블이 아닌 엔티티 객체를 대상으로 검색`
- 모든 DB 데이터를 객체로 변환해서 검색하는 것은 불가능
- 애플리케이션이 필요한 데이터만 DB에서 불러오려면 결국 검 색 조건이 포함된 SQL이 필요

<br>

**JPQL - 특징**
- JPA는 SQL을 추상화한 JPQL이라는 객체 지향 쿼리를 제공
  
    ```java
    String sql = "select m from Member.class as m where m.username like '%until%'";
    ```
    
    일반 SQL과 비슷하지만 엔티티를 대상으로 한다는 점이 다르다.
    
    - SQL을 추상화해서 특정 데이터베이스 SQL에 의존하지 않는다.
    - 동적 쿼리 생성이 쉽지 않다.
    
    <br>

**Criteria - 소개**
- 문자가 아닌 자바코드로 JPQL을 작성할 수 있음
  
```java
//Criteria 사용 준비
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<Member> query = cb.createQuery(Member.class);

//루트 클래스 (조회를 시작할 클래스)
Root<Member> m = query.from(Member.class);

//쿼리 생성 
CriteriaQuery<Member> cq =  query.select(m).where(cb.equal(m.get("username"), “kim”));
List<Member> resultList = em.createQuery(cq).getResultList()
```

  - JPQL 빌더 역할
  - JPA 공식 기능

❌실무에선 거의 사용되지 않는다.

- 쿼리를 동적으로 생성할 수는 있지만 구현이 너무 복잡하고 실용성이 없다.
- 문자가 아닌 자바코드로 JPQL을 작성할 수 있음
- JPQL 빌더 역할
- JPA 공식 기능
- 단점 : 너무 복잡하고 실용성이 없다.
- Criteria 대신에 `QueryDSL 사용 권장`



<br>

<br>

<br>



**QueryDSL 소개**

오픈소스 라이브러리

```java
// JPQL
// SELECT m from Member m WHERE m.age > 18
JPAFactoryQuery query = new JPAQueryFactory(em);
QMember m = QMember.member;

List<Member> list = 
        query.selectFrom(m)
                    .where(m.age.gt(18))
                    .orderBy(m.name.desc())
                    .fetch();
```

- 문자가 아닌 자바코드로 JPQL을 작성할 수 있음

- JPQL 빌더 역할

- 컴파일 시점에 문법 오류를 찾을 수 있음 

<br>
<br>

```java
...
List<Member> list = query.selectFrom(m) 
                        .wheree(m.age.gt(18)) //컴파일 시점에서 오류 검출 가능
                        .orderBy(m.name.desc())
                        .fetch()
```

- 동적쿼리 작성 편리함
- 단순하고 쉽다.
- 실무 사용 권장

<br>

<br>

**네이티브 SQL 소개**
- JPA가 제공하는 SQL을 직접 사용하는 기능
- JPQL로 해결할 수 없는 특정 데이터베이스에 의존적인 기능
    - ex: 오라클 CONNECT BY, 특정 DB만 사용한는 SQL 힌트
    
    ```java
    String sql ="SELECT ID, AGE, TEAM_ID, NAME FROM MEMBER WHERE NAME = ‘kim’";
    List<Member> resultList = em.createNativeQuery(sql, Member.class)
                                    .getResultList();
    ```
    

<br>

<br>

**JDBC 직접 사용, SpringJdbcTemplate 등**

- PA를 사용하면서 JDBC 커넥션을 직접 사용하거나, 스프링 JdbcTemplate, 마이바티스등 함꼐 사용가능

- 단, 영속성 컨텍스트를 적절한 시점에 강제로 플러시 필요
    - ex: JPA를 우회해서 SQL을 실행하기 직전 영속성 컨텍스트를 수동 플러시 해줘야 한다.
    
    ```java
    Member member = new Member();
    member.setUsername("catsbi");
    
    conn.createQuery("select * from Member where username = 'catsbi'");
    //결과 없음
    ```
    
    member는 Jdbc가 쿼리를 수행하는시점에서 영속성 컨텍스트에만 있고 db에 아직 저장되지 않았기 때문에 <br> 조회결과가 없다 그러므로 쿼리 수행 전 수동으로 플러시를 해줘야 한다.



<br>

<br>

<br>



## 2. 기본 문법과 쿼리 API

---

**JPQL 소개**
- JPQL은 객체지향 쿼리 언어다. 따라서 테이블을 대상으로 쿼리하는 것이 아니라 엔티티 객체를 대상으로 쿼리한다.
- JPQL은 SQL을 추상화해서 특정 데이터베이스 SQL에 의존하지 않는다.
  - **EX:** 조회 기능을 만들 때 특정 DB에 의존하는 SQL을 따로 안만들어도 된다.
- JPQL은 결국 SQL로 변환된다.

<br>

**객체/DB 모델**

- UML
  
    ![image](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2036.png)
    
    - **code**
    
        ```java
        @Entity
        public class Member {
            @Id @GeneratedValue
            private Long id;
            private String username;
            private int age;
        
            @ManyToOne
            @JoinColumn(name = "TEAM_ID")
            private Team team;
        
                //getter , setter...
        }
        
        @Entity
        public class Team {
            @Id @GeneratedValue
            private Long id;
            private String name;
        
            @OneToMany(mappedBy = "team")
            private List<Member> members = new ArrayList<Member>();
            public Long getId() {
                return id;
            }
        }
        
        @Entity
        @Table(name = "ORDERS")
        public class Order {
            @Id @GeneratedValue
            private Long id;
            private int orderAmount;
            @Embedded
            private Address address;
            @ManyToOne
            @JoinColumn(name = "PRODUCT_ID")
            private Product product;
        
        }
        
        @Entity
        public class Product {
            @Id @GeneratedValue
            private Long id;
            private String name;
            private int price;
            private int stockAmound;
        }
        
        @Embeddable
        public class Address {
            private String city;
            private String street;
            private String zipcode;
        }
        ```
    



<br>

<br>

**JPQL 문법**

기본 SQL문법과 동일

- select m from Member as m where m.age > 18 → 대소문자를 구분한다.
- 엔티티와 속성은 대소문자 구분을 한다.
- JPQL 키워드는 대소문자 구분을 하지 않는다(SELECT, FROM, where)
- 엔티티 이름 사용, 테이블 이름이 아님(Member)
- **별칭은 필수(alias)** (as는 생략 가능)

<br>

**집합과 정렬**

- count(m), sum(m.age), avg(m.age), max(m.age), min(m.age)
- group by, having



<br>



**TypeQuery, Query**

- TypeQuery : 반환 타입이 명확할 때 사용
  
    ```java
    TypedQuery<Member> query = em.createQuery("SELECT m FROM Meber m", Member.class);
    ```
    
- Query : 반환 타입이 명확하지 않을 때 사용
  
    ```java
    Query query = em.createQuery("SELECT m.username, m.age FROM Meber m");
    ```
    



<br>

<br>

**결과 조회 API**

- `query.getResultList()` : 결과가 하나 이상일 때 리스트 반환
    - 결과가 없으면 빈 리스트 반환
- `uery.getSingleResult()` : 결과가 하나만 나오는 것 외의 모든 상황에서 에러가 나오기 때문에 사용에 주의가 필요
  - 결과가 없으면 : javax.persistence.NoResultException
  - 둘 이상이면 : javax.persistence.NonUniqueResultException



<br>

<br>



**파라미터 바인딩 - 이름 기준**
❓위치기반의 파라미터 바인딩도 있지만 추천하지 않는다.

```java
/*Usercase - 1*/
SELECT m FROM Member m where m.username = :username
query.setParameter("username" usernameParam);

/*Usecase - 2 위치 기반*/
SELECT m FROM Member m where m.username = ?1
query.setParameter(1, usernameParam);
```



<br>

<br>

<br>



## 3. 프로젝션(SELECT)

---

- SELECT 절에 조회할 대상을 지정하는 것
- 프로젝션 대상 : 엔티티, 임베디드 타입, 스칼라 타입(숫자, 문자등 기본 데이터 타입)
- SELECT m FROM Member m → 엔티티 프로젝션
- SELECT [m.team](http://m.team) FROM Member m → 엔티티 프로젝션
- SELECT m.address FROM Member m → 임베디드 타입 프로젝션
- SELECT m.username, m.age FROM Mebmer m → 스칼라 타입 프로젝션
- DISTINCT로 중복 제거



<br>

<br>



### 묵시적 조인 명시적 조인

**묵시적 조인**
  
```java
List<Team> result = em.createQuery("select m.team from Member m", Team.class)
//SQL: SELECT t.id, t.name FROM Member m inner join TEAM t on m.team_id = t.id
```

위 코드처럼 JPQL에는 JOIN문법이 없지만 자연스럽게 JOIN을 해서 Team Entity를 조회해 온다.
    

<br>

**명시적 조인**

```java
List<Team> result = em.createQuery("select t from Member m join m.team t", Team.class)
//SQL: SELECT t.id, t.name FROM Member m inner join TEAM t on m.team_id = t.id
```

실행되는 SQL은 동일하지만 명시적으로 JPQL에 적어줬기에 가독성이 높아지고 JOIN 쿼리가 날아가겠다고 예측이 가능하다.

- 묵시적 조인보다는 명시적 조인을 하는게 좋다.



<br>

<br>

### 임베디드 타입 프로젝션

```java
em.createQuery("select o.address from Order o", Address.class).getRresultList();
//SQL: SELECT o.city, o.street, o.zipcode FROM ORDERS o
```

임베디드 타입은 따로 조인을 해서 가져오지는 않는다.

But, from절에 Order가 아닌 Address를 적으면 에러가난다. 엔티티로부터 시작되야 한다.



<br>

<br>



### 스칼라 타입 프로젝션 / 여러 값 조회

⇒ 스칼라타입으로 조회를 할때 값을 어떻게 저장해서 사용해야 할까?

```java
em.createQuery("select distinct m.address, m.age from Member m").getRresultList();
```

1. Query 타입으로 조회
2. Object[] 타입으로 조회
3. new 명령어로 조회
    - 단순 값을 DTO로 바로 조회
        ```java
        SELECT new jpabook.jpql.UserDTO(m.username, m.age)from Memer m;
        ```
    - 패키지 명을 포함한 전체 클래스 명을 적어줘야 한다.
    - 순사와 타입이 일치하는 생성자 필요.



<br>

<br>

<br>



## 4. 페이징

---

- JPA는 페이징을 다음 두 API로 추상화
- setFirstResult(int startPosition): 조회 시작 위치(0부터 시작)
- setMaxResults(int maxResult): 조회할 데이터 수

```java
em.createQuery("select m from Member m order by m.age desc", Member.class)
    .setFirstResult(0)
    .setMaxResults(10)
    .getReulstList();
```

<br>
<br>

### 기존에는 Diarect별로 방언을 맞춰서 쿼리를 하나하나 구현해야 했는데, <br> 이제는 두 개의 함수로(setFirstResult() setMaxResults)로 해결할 수 있어 몹시 편리하다.



<br>

<br>

**Example : MySQL방언**

```java
SELECT
    M.ID AS ID,
    M.AGE AS AGE,
    M.TEAM_ID AS TEAM_ID,
    M.NAME AS NAME
FROM
    MEMBER M
ORDER BY
    M.NAME DESC LIMIT ?,?
```

<br>

**Example : Oracle 방언**

```java
SELECT *
FROM (SELECT ROW_.*, ROWNUM ROWNUM_
      FROM (SELECT M.ID      AS ID,
                   M.AGE     AS AGE,
                   M.TEAM_ID AS TEAM_ID,
                   M.NAME    AS NAME
            FROM MEMBER M
            ORDER BY M.NAME
           ) ROW_
      WHERE ROWNUM <= ?
     )
WHERE ROWNUM_ > ?
```



<br><br>

<br>



## 5. 조인

---

- 내부 조인

```java
SELECT m FROM Member m [INNER] JOIN m.team t
```

- 외부 조인

```java
SELECT m FROM Member m LEFT [OUTER] JOIN m.team t
```

- 세타 조인

```java
select count(m) from Member m, Team t where m.username = t.name
```



<br>

<br>



**조인 - ON절**

1. **조인 대상 필터링 (회원과 팀을 조인하면서, 팀 이름이 A인 팀만 조인)**
   - JPQL : SELECT m, t FROM Member m LEFT JOIN m.team t on [t.name](http://t.name/) = 'A'
   - SQL: SELECT m.*, t.* FROM Member m LEFT JOIN Team t ON m.TEAM_ID = [t.id](http://t.id/) and [t.name](http://t.name/) ='A'

2. **연관관계 없는 엔티티 외부 조인 (회원의 이름과 팀의 이름이 같은 대상 외부조인)**
   - JPQL: SELECT m,t FROM Member m LEFT JOIN Team t on m.username = [t.name](http://t.name/)
   - SQL: SELECT m.*, t.* FROM Member m LEFT JOIN Team t ON m.username = t.username



<br>

<br>

<br>



## 6. 서브쿼리

---

**Example_1 : 나이가 평균보다 많은 회원**e

```java
select m from Member m
where m.age > (select avg(m2.age) from Member m2);
```

**Example_2 : 한 건이라도 주문한 고객**

```java
select m from Member m
where (select count(o) from Order o where m=o.member)>0
```



<br>

<br>

**서브 쿼리 지원 함수**

- [NOT] EXISTS(subquery) : 서브쿼리에 결과가 존재하면 참
    - {ALL|ANY|SOME} (subquery)
    - ALL 모두 만족하면 참
    - ANY, SOME: 같은 의미, 조건을 하나라도 만족하면 참
- [NOT] IN(subquery): 서브쿼리의 결과 중 하나라도 같은 것이 있으면 참
- 예제
    - 팀A 소속인 회원
      
        ```java
        select m from Member m
        where exists (select t from m.team t where t.name = '팀A')
        ```
        
    - 전체 상품 각각의 재고보다 주문량이 많은 주문들
      
        ```java
        select o from Order o
        where o.orderAmount > ALL(select p.stockAmount from Product p)
        ```
        
    - 어떤 팀이든 티에 소속된 회원
      
        ```java
        select m from Member m
        where m.team = ANY(select t from Team t)
        ```
        



<br>

<br>

**JPA 서브 쿼리 한계**

- PA는 WHERE, HAVING절에서만 서브 쿼리 사용 가능
- SELECT 절도 가능(하이버네이트에서 지원)
- **FROM 절의 서브 쿼리는 현재 JPQL에서 불가능**
    - **조인으로 풀 수 있으면 풀어서 해결**



<br>

<br>

<br>



## 7. JPQL 타입표현과 기타식

---

- 문자: 'HELLO', 'SHE"S'
- 숫자: 10L(Long), 10D(Double), 10F(Float)
- Boolean: TRUE, FALSE
- ENUM: jpabook.MemberType.Admin(패키지명 포함)
  
    ```java
    select m.username, 'HELLO', true from Member m 
     where m.type = jpql.MemberType.ADMIN
    ```
    
- 엔티티 타입: TYPE(m)= Member(상속 관계에서 사용)
  
    ```java
    em.createQuery("select i from Item i where type(i) = Book", Item.class);
    ```
    



<br>

<br>

**JPQL 기타**

- SQL과 문법이 같은 식
- EXISTS, IN
- AND, OR, NOT
- =, >, ≥, <, ≤, <>
- BETWEEN, LIKE, IS NULL



<br>

<br>

<br>



## 8. 조건식 (CASE등등)

---

**조건식 - CASE식**

### 기본 CASE식

```java
select case when m.age <= 10 then '학생요금'
            when m.age >= 60 then '경로요금'
            else '일반요금'
    end
from Member m
```

<br>

<br>

### 단순 CASE 식

```java
select case t.name
        when '팀A' then '인센티브 110%'
        when '팀B' then '인센티브 120%'
        else '인센티브 105%'
    end
from Team t
```

- `COALESCE`: 하나씩 조회해서 null이 아니면 반환
  
    ```java
    select coalesce(m.username, '이름 없는 회원') from Member m;
    ```
    
<br>

- `NULLIF`: 두 값이 같으면 null 반환, 다르면 첫번째 값 반환
  
    ```java
    select NULLIF(m.username, '관리자') from Member ;
    ```
    



<br>

<br>

<br>

## 9. JPQL 함수

---

- CONCAT
- SUBSTRING
- TRIM
- LOWER, UPPER
- LENGTH
- LOCATE
- ABS, SQRT, MOD
- SIZE INDEX(JPA용도)

<br>

```java
//- CONCAT
select concat('a','b'); //ab

//- SUBSTRING: firstParam의 값을 secondParam위치부터 thirdParam갯수만큼 잘라서 반환
select substring('abcd', 2,3) // bc

//- TRIM
select trim(' lee han sol ')//lee han sol

//- LOWER, UPPER
select LOWER('hansolHI');//hansolhi
select UPPER('hansolHI');//HANSOLHI

//- LENGTH
select LENGTH('hansolHI'); // 6

//- LOCATE
select LOCATE('so', 'hansol');//4

//- ABS, SQRT, MOD
select ABS(-30);// 30
select SQRT(4);//2
select MOD(4,2);//0

//- SIZE, INDEX(JPA용도)
select SIZE(t.members) from Team t // 0
```

<br>

<br>



**사용자 정의 함수 호출**

- 하이버네이트는 사용전 방언에 추가해야 한다.
  - 사용하는 DB방언을 상속받고, 사용자 정의 함수를 등록한다. <br>
  실제 소스코드내부에 정의되있는 함수들을 참고해서 작성해주면 된다.

```java
//group_concat이라는 함수를 만들어서 등록한다고 가정한다.
public class MyPostgresDialect extends PostgreSQL94Dialect {
    public MyPostgresDialect() {
        registerFunction("group_concat", new StandardSQLFunction("group_concat", StandardBasicTypes.STRING));
    }

...
...
...

//설정파일 등록
<property name="hibernate.dialect" value="jpql.MyPostgresDialect"/>
```