---
title: 객체지향 쿼리 언어 - 중급 문법
date: 2021-12-30 10:20:00
featuredImage: /images/JPA/jpa.png
categories:
  - JPA
tags:
  - JPA
  - Hibernate
---
## 1. 경로 표현식

---

### .(점)을 찍어 객체 그래프를 탐색하는 것

```java
select m.username -> 상태 필드 
	from Member m
	join m.team t -> 단일 값 연관 필드
	join m.orders o -> 컬렉션 값 연관 필드 
where t.name = '팀A'
```

<br>

**경로 표현식 용어 정리**

- 상태 필드(state field): 단순히 값을 저장하기 위한 필드(ex: m.usernmae)
- 연관 필드(association field): 연관관계를 위한 필드
    - 단일 값 연관 필드:
    ```java
    @ManyToOne, @OneToOne: 대상이 엔티티(ex: m.team)
    ```
    - 컬렉션 값 연관 필드:
    ```java
        @OneToMany, @ManyToMany, 대상이 컬렉션(ex m.orders)
    ```



<br>

<br>

**경로 표현식 특징**

- 상태 필드(state field): 경로 탐색의 끝, 탐색x
- 단일 값 연관 경로: 묵시적 내부 조인(inner join)발생, 탐색O
  
    ```java
    select m.team.name from Member m; //team에서 경로탐색이 더 가능하다(name)
    ```
    
- 컬렉션 값 연관 경로: 묵시적 내부 조인 발생, 탐색X
    - FROM 절에서 명시적 조인을 통해 별칭을 얻으면 별칭을 통해 탐색 가능
      
        ```java
        select m.username from Team t join t.members m;
        ```
        



<br>

<br>

<br>



**실무에서는 명시적 조인을 사용하자.**

**조인은 SQL 튜닝에 중요 포인트**

**묵시적 조인은 조인이 일어나는 상황을 한눈에 파악하기 어려움**



<br>

<br>

<br>



## 2. JPQL - 페치 조인 (fetchjoin)

---

- SQL 조인 종류는 아니고 JPA에서 제공하는 기능이다.
- JPQL에서 성능 최적화를 위해 제공하는 기능.
- 연관된 엔티티나 컬렉션을 SQL 한 번에 함께 조회하는 기능
- join fetch 명령어 사용
- 페치 조인 :: = [LEFT [OUTER] | INNER ] JOIN FETCH 조인 경로



<br>

**엔티티 페치 조인**

- 회원을 조회하면서 연관된 팀도 함께 조회(SQL 한 번에)
  
    ```java
    //JPQL
    select m from Member m join fetch m.team
    
    //SQL
    select m.* t.* from Member m inner join Team t on m.team_id = t.id;
    ```
    

<br>

- 도식화

    ![image](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2037.png)

    

    팀이 있는 회원을 조회하고 싶을 때 fetch join을 사용하면 내부적으로 `inner join`을 사용한다.

    팀이 없는 회원은 누락된다.




<br>

<br>

**페치 조인 사용 코드**

- 기존 연관관계 조회 로직의 문제
  
    ```java
    String jpql = "select m from Member m";
    List<Member> members = em.createQuery(jpql, Member.class) 
                                .getResultList();

    for (Member member : members) {
        System.out.println("username = " + member.getUsername() + ", " +
                        "teamName = " + member.getTeam().name());
    
    //회원1, 팀A(SQL)
    //회원2, 팀A(1차 캐시)
    //회원3, 팀B(SQL)

    //회원 100명 -> N + 1
    }
    ```
    
    최초 jpql을 통해 Member를 조회해 올때 Team의 정보는 Proxy객체로 가지고 있다. (실제론 없다는 의미) <br> 그렇기에 실제로 getTeam().getName()을 통해 팀의 정보를 조회하려고 할 때 SQL을 수행한다. <br> 주석 내용대로 한번 가져온 Team의 정보는 1차 캐시에 올라가 있기 때문에 더 조회할 필요는 없지만, <br> 회원을 N명 조회하게 되었을때 최대 N + 1 번 Team 조회 쿼리가 수행 될 수 있다.

<br>

- 페치 조인을 통한 해결

    ```java
    String jpql = "select m from Member m join fetch m.team";
    List<Member> members = em.createQuery(jpql, Member.class) 
                                .getResultList();

    for (Member member : members) {
        //페치 조인으로 회원과 팀을 함께 조회해서 지연 로딩X
        System.out.println("username = " + member.getUsername() + ", " +
                            "teamName = " + member.getTeam().name());
    }
    ```

    페치조인은 조회 당시에 실제 엔티티가 담긴다. 그렇기 때문에 지연로딩 없이 바로 사용이 가능하다.




<br>

<br>



### 실무에서 자주 사용된다.

<br>
<br>

**컬렉션 페치 조인**

- 일대다 관계, 컬렉션 페치 조인
  
```java
//JPQL
select t from Team t join fetch t.members where t.name = '팀A';

//SQL
select t.*, m.* from team t, inner join member m on t.id = m.team_id
    where t.name = '팀A';
```

이를 수행하면 Team은 하나지만 Member가 1개 이상일 수 있다.

    
![image](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2038.png)
    
    
    
팀A는 1개이지만 그에 해당하는 멤버는 회원1과 회원2로 두개이기 때문에, 조회 결과는 위 표처럼 2개의 row가 된다.



<br>

![image](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2039.png)

팀은 하나이기에 같은 주소값을 가진 결과가 두개가 나오고 팀A의 입장에서는 회원1, 회원2를 가진다.




<br>

<br>

**컬렉션 페치 조인 사용 코드**

```java
String jpql = "select t from Team t join fetch t.members where t.name = '팀A'"
List<Team> teams = em.createQuery(jpql, Team.class).getResultList();
for (Team team : teams) {
    System.out.println("teamname = " + team.getName() + ", team = " + team);
    for (Member member : team.getMembers()) {
        //페치 조인으로 팀과 회원을 함께 조회해서 지연 로딩 발생 안함
        System.out.println("->username = " + member.getUsername()+ ", member = " + member);
    }
}
```

<br>

<br>

**페치 조인과 DISTINCT**

<br>
<br>

### previous: Team을 fetch join 해서 가져왔을 경우 사이즈는 어떻게 나올 것인가

```java
String query = "select t from Team t";
String query2 = "select t from Team t join fetch t.members";

List<Team> result = em.createQuery(query, Team.class).getResultList();
List<Team> result2 = em.createQuery(query2, Team .class).getResultList();

System.out.println("result size::"+ result.size()); //2
System.out.println("result2 size::"+ result2.size());//3
```

- 일대다(1:N) 관계에서는 join fetch 결과가 뻥튀기 될 수 있다.
  
    ![image](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2040.png)
    
    
    
  팀A는 하나지만 ROW는 2개로 뻥튀기 되었다
  
  <br>
  
  <br>
  
- SQL의 DISTINCT는 중복된 결과를 제거하는 명령

- JPQL의 DISTINCT 2가지 기능 제공
    1. SQL에 DISTINCT를 추가
    2. 애플리케이션에서 엔티티 중복 제거

```java
select distinct t from Team t join fetch t.members where t.name = '팀A';
```

위 코드를 실행하면 SQL에 DISTINCT를 추가하지만 데이터가 다르므로 SQL결과에서 중복제거 실패

![image](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2041.png)

ID(PK)도 다르고 NAME도 다르기 때문에 중복제거 실패

<br>

- 쿼리만으로는 중복제거가 안되기 때문에 JPA 추가적으로 DISTINCT가 추가로 애플리케이션에서 중복 제거를 시도한다.
- 같은 식별자를 가진 Team 엔티티 제거
  
    ![image](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2042.png)
    



<br>

<br>

### 반대로 다대일(N:1)은 뻥튀기 되지 않는다.

<br>

**페치 조인과 일반 조인의 차이**

- 일반 조인 실행시 연관된 엔티티를 함께 조회하지 않음.


```java
//JPQL
select from Team t join t.members m where t.name = '팀A';

//SQL
select t.* from Team t inner join member m on t.id = m.team_id
    where t.name = '팀A';
```


해당 쿼리 수행시 일반 조인은 연관된 엔티티를 먼저 조회하지 않기 때문에 프록시 객체 반환.

실제로 해당 엔티티를 사용할 때 실제 값을 조회한다.

반면, 페치 조인은 조회시 연관관계도 같이 조회(즉시 로딩)

페치 조인은 객체 그래프를 SQL 한 번에 조회하는 개념이다.
    



<br>

<br>

**페치 조인의 특징과 한계**

- 페치 조인 대상에는 별칭을 줄 수 없다

    ```java
    String query = "select t from Team t join fetch t.members as m"
    //as m 이라는 별칭(alias)는 fetch join에서 사용할 수 없다.
    ```

    하이버네이트는 가능하지만, 가급적 사용을 하지 않는게 좋다

    → ex: 팀을 조회하는 상황에서 멤버가 5명인데 3명만 조회한 경우 3명만 따로 조작하는 것은 몹시 위험. 
    
    <br>

    ```java
    String query = "select t from Team t join fetch t.members as m where m.age > 10"
    ```

    - 기본적으로 JPA에서 설계사상은 객체 그래프를 탐색한다는 것은 연관된 엔티티를 모두 <br> 가져온다는 것을 가정하고 만들어졌다.
    - fetch join에 별칭을 붙히고 where절을 더해 필터해서 결과를 가져오게 되면 모든걸 <br> 가져온 결과와 비교하여 다른 갯수에 대해서 정합성을 보장하지 않는다.

<br>

- 둘 이상의 컬렉션은 페치 조인 할 수 없다.

    ```java
    tring query = "select t from Team t join fetch t.members, t.orders"
    //불가능 fetch join에서 컬렉션은 1개만 사용하자.
    ```


<br>

- 컬렉션을 페치 조인하면 페이징 API를 사용할 수 없다.

    - 일대일, 다대일 같은 단일 값 연관 필드들은 페치 조인해도 페이징 가능

    - 하이버네이트는 경고 로그를 남기고 메모리에서 페이징( 매우 위험)
      
        ```java
        String query = "select t from Team t join fetch t.members";
        List<Team> result = em.createQuery(query, Team.class)
        				    .setFirstResult(0)
        					.setMaxResults(1)
        					.getResultList();
        ```
        
    - 실행결과
      
        ![image](https://cdn.jsdelivr.net/gh/jaeyeonme/jaeyeonme.github.io@master/assets/img/JPA/Untitled%2043.png)
        
        경고 로그 출력 후 메모리에서 페이징 하는데 쿼리를 보면 limit offset이 없다.
        
        <br>
        
**그냥 쓰지 말자!**

<br>

<br>

### 해결 방안

- 일대다를 다대일로 방향을 전환하여 해결한다
  
    ```java
    String query = "select m from Member m join fetch m.team t";
    ```
    
- BatchSize()
  
    ```java
    public class Team{
    ...
    
    @BatchSize(size = 100)
    @OneToMany(mappedBy = "team")
    private List<Member> members;
    ...
    }
    
    String query = "select t from Team t";
    ```
    
    지연로딩 상태이지만, 조회할 때 members를 BatchSize의 size만큼 조회해 온다.
    
    BatchSize()는 글로벌 설정으로 할 수도 있다.
    
    ```java
    //persistence.xml
    <property name="hibernate.default_batch_fetch_size" value="100"/>
    ```
    

<br>

- 연관된 엔티티들을 SQL한 번으로 조회 - 성능 최적화
- 엔티티에 직접 적용하는 글로벌 로딩 전략보다 우선함
  ```java
  @OneToMany(fetch = FetchType.LAZY)//글로벌 로딩 전략
  ```
- 실무에서 글로벌 로딩 전략은 모두 지연 로딩
- 최적화가 필요한 곳은 페치 조인 적용

<br>

<br>

**페치 조인 - 정리**
- 모든 것을 페치 조인으로 해결할 수는 없다.
- 페치 조인은 객체 그래프를 유지할 때 사용하면 효과적이다.
- 여러 테이블을 조인해서 엔티티가 아닌 전혀 다른 결과를 내야 하면, 페치 조인 보다는 일반 조인을 사용하고 <br> 필요한 데이터들만 조회해서 DTO로 반환하는 것이 효과적



<br>

<br>

<br>



## 3. 다형성 쿼리

---

![image](https://cdn.jsdelivr.net/gh/jaeyeonme/jaeyeonme.github.io@master/assets/img/JPA/Untitled%2044.png)

<br>

<br>

**TYPE**

- 조회 대상을 특정 자식으로 한정
- ex: Item 중 Book, Movie를 조회해라
  
```java
//JPQL
select i from Item i where type(i) IN(Book, Movie)

//SQL
select i from Item i where i.DTYPE in('B', 'M');
```
    

<br>

<br>

**TREAT(JPA2.1)**

- 자바의 타입 캐스팅과 유사(형변환)
- 상속 구조에서 부모 타입을 특정 자식 타입으로 다룰 때 사용
- FROM, WHERE, SELECT(하이버네이트 지원)사용
- **ex:** 부모님 Item과 자식 Book이 있다.
  
    ```java
    //JPQL
    select from Item i where treat(i as Book).author = 'kim';
    
    //SQL[
    select i.* from Item i where i.DTYPE = 'B' and i.author = 'kim';
    ```
    



<br>

<br>

<br>

## 4. 엔티티 직접 사용

---

### 기본 키 값

- JPQL에서 엔티티를 직접 사용하면 SQL에서 해당 엔티티의 기본 키 값을 사용
  
    ```java
    //JPQL
    select count(m.id) from Member m //엔티티의 아이디를 사용
    select count(m) from Member m //엔티티를 직접 사용
    
    //SQL(JPQL 둘 다 같은 다음 SQL 실행)
    select count(m.id) as cnt from Member m
    Java
    ```
    
- 파라미터를 엔티티를 넘겨주거나 식별자를 넘겨주더라도 실행된 SQL은 같다.
  
    ```java
    /*엔티티를 파라미터로 전달*/
    String jpql = "select m from Member m where m = :member";
    List resultList = em.createQuery(jpql)
                            .setParameter("member", member)
                            .getResultList();
    
    /*식별자를 직접 전달*/
    String jpql = "select m from Member m where m.id = :memberId";
    List resultList = em.createQuery(jpql)
                            .setParameter("memberId", memberId)
                            .getResultList();
    ```

    위 두 JPQL의 실행 SQL은 아래와 같이 동일하다

    ```java
    select m.* from Member m where m.id = ?
    ```


<br>

<br>

### 외래 키 값

- 기본키와 로직은 동일하다. 엔티티 or 외래 키를 쓰면 실행 SQL은 동일하다
  
    ```java
    Team team = em.find(Team.class, 1L);
    
    String query = "select m from Member m where m.team = :team";
    List resultList = em.createQuery(query)
                            .setParameter("team", team)
                            .getResultList();
    
    String query = "select m from Member m where m.team.id = :teamId";
    List resultList = em.createQuery(query)
                            .setParameter("teamId", teamId)
                            .getResultList();
    ```
    
    실행된 SQL
    
    ```java
    select m.* from Member m where m.team_id = ?
    ```
    



<br>

<br>

<br>

## 5. Named 쿼리

---

- 미리 정의해서 이름을 부여해두고 사용하는 JPQL
- 정적 쿼리
- 어노테이션, XML에 정의
  
    ```java
    @Entity
    @NamedQuery(
        name="Member.findByUsername",
        query="select m from Member m where m.username = :username")
    public class Member {
    ...
    }
    
    ...
    
    List<Member> resultList = em.createNamedQuery("Member.findByUsername", Member.class)
                                    .setParameter("username", "회원1")
                                    .getResultList();
    ```
    
    - 애플리케이션 로딩 시점에 초기화 후 재사용
      
        → JPA는 결국 SQL로 parsing 되어 사용되는데 로딩 시점에 초기화가 된다면 parsing cost를 절약 가능.
        
    - 애플리케이션 로딩 시점에 쿼리를 검

<br>

<br>

**XML에 정의**

```java
//[META_INF?persistence.xml]
<persistence-unit name="jpabook">
		<mapping-file>META-INF/ormMember.xml</mapping-file>

//[META-INF/ormMember.xml]
<?xml version="1.0" encoding="UTF-8"?>
<entity-mappings xmlns="htt://xmlns.jcp.org/xml/ns/persistence/orm" version="2.1">
    <named-query name="Member.findByUsername">
            <query>
                    <![CDATA[ select m from Member m where m.username = :username]]
            </query>
    </named-query>
</entity-mappings>
```

사용법은 @NamedQuery의 query를 사용방법과 같다.

- NamedQuery와 XML에 정의돈 Query중 XML이 항상 우선권을 가진다. <br>
→ 애플리케이션 운영 환경에 따라 다른 XML를 배포할 수 있다.



<br>

<br>

### SpringData JPA를 사용하면 NamedQuery를 이미 사용.

```java
@Repository
public interface MemberRepository extends JpaRepository<Member, Long>{

@Query("select u from User u where u.username = ?1")
Member findByUsername(String username);
}
```

`@Repository annotation`이 등록된 인터페이스에서 사용되는 `@Query annotation`에 있는 <br> JPQL (or native)들이 NamedQuery로써 컴파일시에 등록되는 것이다.



<br>

<br>

<br>



## 6. 벌크 연산

---

- 일반적으로 우리가 알고 있는 SQL의 update or delete 문을 생각하면 된다.
  - ex: 재고가 10개 미만인 모든 상품의 가격을 10% 상승하려면?
- JPA의 dirty check로 실행하기 위해서는 너무 많은 SQL이 실행되야 한다.
    1. 재고가 10개 미만인 상품을 리스트 조회
    2. 상품 엔티티의 가격 10% 증가
    3. 트랜잭션 커밋 시점에 dirty checking.
- 변경된 데이터가 100건이면 100건의 UPDATE SQL 실행

<br>

<br>

**Example**

- 쿼리 한 번으로 여러 테이블 업데이트(엔티티)
- executeUpdate()의 결과는 영향받은 엔티티 수 반환
- UPDATE, DELETE 지원
- INSERT (insert into ... select, 하이버네이트 지원)
  
    ```java
    String query = "update Product p "+
                    "set p.price = p.price * 1.1 where p.stockAmount < :stockAmount";
    
    int resultCount = em.createQuery(qlString)
                            .setParameter("stockAmount", 10)
                            .executeUpdate();
    ```
    

<br>

<br>

**벌크 연산 주의**

- 벌크 연산은 영속성 컨텍스트를 무시하고 데이터베이스에 직접 쿼리
    - 벌크 연산을 먼저 실행
    - 벌크 연산 수행 후 영속성 컨텍스트 초기화
      
        → 엔티티 조회 후 벌크연산으로 엔티티 업데이트가 되버리면 DB의 엔티티와 영속성 컨텍스트의 엔티티가 <br> 서로 다른 값이 되게 된다.
        



<br>

<br>

<br>



### 엔티티 직접 사용 - 외래 키 값

---

```java
Team team = em.find(Team.class, 1L);

String qlString = “select m from Member m where **m.team = :team”**; 
List resultList = em.createQuery(qlString)
                        .setParameter("team", **team**) 
                        .getResultList();
```

```java
String qlString = “select m from Member m where **m.team.id = :teamId”**; 
List resultList = em.createQuery(qlString)
                        .setParameter("teamId", **teamId**) 
                        .getResultList();
```

<br>

<br>

**실행된 SQL**

```java
select m.* from Member m where **m.team_id=?**
```

```java
String query = "select m From Member m where m.team = :team";
List<Member> members = em.createQuery(query, Member.class)
        .setParameter("team", teamA)
        .getResultList();

for (Member member : members) {
    System.out.println("member = " + member);
}
```

```java
Hibernate: 
    /* select
        m 
    From
        Member m 
    where
        m.team = :team */ select
            member0_.id as id1_0_,
            member0_.age as age2_0_,
            member0_.TEAM_ID as team_id5_0_,
            member0_.type as type3_0_,
            member0_.username as username4_0_ 
        from
            Member member0_ 
        where
            member0_.TEAM_ID=?
member = Member{id=3, username='회원1', age=0}
member = Member{id=4, username='회원2', age=0}
```



<br>

<br>

<br>



## 5. JPQL - Named 쿼리

### Named 쿼리 - 정적 쿼리

---

- 미리 정의해서 이름을 부여해두고 사용하는 JPQL
- 정적 쿼리
- 어노테이션, XML에 정의
- 어플리케이션 로딩 시점에 초기화 후 재사용
- `어플리케이션 로딩 시점에 쿼리를 검증`



<br>

<br>

### Named 쿼리 - 어노테이션

---

```java
@Entity
@NamedQuery(
        name = "Member.findByUsername",
        query="select m from Member m where m.username = :username")
public class Member {
```

```java
List<Member> resultList = em.createNamedQuery("Member.findByUsername", Member.class)
        .setParameter("username", "회원1")
        .getResultList();

for (Member member : resultList) {
    System.out.println("member = " + member);
}
```

```java
Hibernate: 
    /* insert jpql.Member
        */ insert 
        into
            Member
            (age, TEAM_ID, type, username, id) 
        values
            (?, ?, ?, ?, ?)
Hibernate: 
    /* Member.findByUsername */ select
        member0_.id as id1_0_,
        member0_.age as age2_0_,
        member0_.TEAM_ID as team_id5_0_,
        member0_.type as type3_0_,
        member0_.username as username4_0_ 
    from
        Member member0_ 
    where
        member0_.username=?
member = Member{id=3, username='회원1', age=0}
```



<br>

<br>

### Named 쿼리 - XML에 정의

---

```xml
[META-INF/persistence.xml]
<persistence-unit name="jpabook" >
    <mapping-file>META-INF/ormMember.xml</mapping-file>
```

```xml
[META-INF/ormMember.xml]

<?xml version="1.0" encoding="UTF-8"?>
<entity-mappings xmlns="http://xmlns.jcp.org/xml/ns/persistence/orm" version="2.1">

    <named-query name="Member.findByUsername">
            <query><![CDATA[ 
                    select m from Member m 
                    where m.username = :username 
            ]]></query> 
    </named-query>

    <named-query name="Member.count">
            <query>select count(m) from Member m</query> 
    </named-query>

</entity-mappings>
```



<br>

<br>

### Named 쿼리 환경에 따른 설정

---

- XML이 항상 우선권을 가진다.
- 애플리케이션 운영 환경에 따라 다른 XML을 배포할 수 있다.



<br>

<br>

<br>



## 6. JPQL - 벌크 연산

### 벌크 연산

---

- 재고가 10개 미만인 모든 상품의 가격을 10% 상승하려면?
- JPA 변경 감지 기능으로 실행하려면 너무 많은 SQL 실행
    1. 재고가 10개 미만인 상품을 리스트로 조회한다.
    2. 상품 엔티티의 가격을 10% 증가한다.
    3. 트랜잭션 커밋 시점에 변경감지가 동작한다.
- 변경된 데이터가 100건이라면 100번의 UPDATE SQL 실행



<br>

<br>

### 벌크 연산 예제

---

- 쿼리 한 번으로 여러 테이블 로우 변경(엔티티)
- `executeUpdate()의 결과는 영향받은 엔티티 수 반환`
- `UPDATE, DELETE 지원`
- `INSERT(insert into .. select, 하이버네이트 지원)`

```java
// FLUSH 자동 호출
int resultCount = em.createQuery("update Member m set m.age = 20")
        .executeUpdate();

System.out.println("resultCount = " + resultCount);
```



<br>

<br>

### 벌크 연산 주의

---

- 벌크 연산은 영속성 컨텍스트를 무시하고 데이터베이스에 직접 쿼리
  - 벌크 연산을 먼저 실행
  - `벌크 연산 수행 후 영속성 컨텍스트 초기화`

    ```java
    // FLUSH 자동 호출
    int resultCount = em.createQuery("update Member m set m.age = 20")
            .executeUpdate();

    em.clear();

    Member findMember = em.find(Member.class, member1.getId());
    System.out.println("findMember = " + findMember.getAge());
    ```