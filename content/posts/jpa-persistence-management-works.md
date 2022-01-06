---
title: 영속성관리 - 내부 동작 방식
date: 2021-12-29 12:00:00
featuredImage: /images/JPA/jpa.png
categories:
  - JPA
tags:
  - JPA
  - Hibernate
---
## 1. 영속성 컨텍스트

![image](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2048.png)

- 웹 어플리케이션이 구동하는 시점에 `EntityManagerFactory` 를 생성하여 가지고 있으며, 사용자의 요청이 있을 때 EntityManager를 생성하여 커넥션 풀(`Connection Pool`)을 사용해서 DB를 핸들링 하게 된다.

<br>

<br>



### 영속성 컨텍스트란?

- 엔티티를 영구 저장하는 환경

- EntityManager.persist(entity);

  → DB에 저장한다기 보다는 영속성 컨텍스트를 통해 엔티티를 영속화 한다는 의미

  - 영속성 컨텍스트에 엔티티를 저장한다는 말

- 영속성 컨텍스트는 논리적인 개념

- 눈에 보이지 않는다.

- 엔티티 매니저를 통해서 영속성 컨텍스트에 접근

  - J2SE 환경

![image](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2049.png)



<br>

<br>



### 엔티티의 생명 주기

![image](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2050.png)

- 비영속(new / transient): 영속성 컨텍스트와 전혀 관계가 없는 **새로운** 상태
- 영속(managed): 영속성 컨텍스트에 **관리**되는 상태
- 준영속(detached): 영속성 컨텍스트에 저장되었다가 **분리**된 상태
- 삭제(removed): **삭제**된 상태



<br>

<br>



### 비영속

: JPA에 관계없이 객체 생성만 된 상태

![image](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2051.png)

```java
// 객체를 생성한 상태(비영속)
Member member = new Member();
member.setId("member1");
member.setUsername("회원1");
```



<br>

<br>



### 영속

![image](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2052.png)

```java
// 객체를 생성한 상태(비영속)
Member member = new Member();
member.setId("member1");
member.setUsername("회원1");

EntityManager em = emf.createEntityManager();
em.getTransaction().begin();

// 객체를 저장한 상태(영속)
em.persist(member);
```



<br>

<br>



### 준영속, 삭제

```java
// 회원 엔티티를 영속성 컨텍스트에서 분리, 준영속 상태
em.detach(member);

// 객체를 삭제한 상태(삭제)
em.remove(member);
```

- 준영속: 영속성 컨텍스트에서 삭제하는 것
- 삭제는 실제로 DB에서 해당 **ROW를 삭제**하는 것





<br>

<br>

<br>



## 2. 영속성 컨텍스트의 이점

### 1. 1차 캐시

```java
Member member = new Member();
member.setId("member1");
member.setUsername("회원1");

//1차 캐시에 저장됨
em.persist(member);

//1차 캐시에서 조회
Member findMember = em.find(Member.class, "member1");
```

- 조회 시 영속 컨텍스트안에서 1차 캐시를 조회 후 해당 엔티티가 있을 경우 캐시를 조회 해 온다.
- 조회 시 영속 컨텍스트 안에서 1차 캐시를 조회 후 해당 엔티티가 없을 경우 데이터베이스에서 조회해 온다.
- 데이터베이스 트랜잭션 내부에서 만들고 종료되기 때문에 하나의 비즈니스 로직이 종료될 경우 1차캐시는 다 사라지기 때문에 큰 도움이 되지 않는다. (비즈니스 로직이 복잡해 질 수록 효과는 커진다.)



<br>

<br>



### 2. 동일성(identity) 보장

```java
Member findMember = em.find(Member.class, 100L);
Member findMember2 = em.find(Member.class, 100L);

System.out.println(findMember == findMember2); // true
```

: 1차 캐시로 반복 가능한 읽기(REPEATABLE READ) 등급의 트랜잭션 격리 수준을 데이터베이스가 아닌 애플리케이션 차원에서 제공

> 참고: 트랜잭션의 격리수준이 궁금하면 트랜잭션이 보장해야 하는 ACID를 기억하자.



<br>

<br>



### 3. 엔티티 등록 - 트랜잭션을 지원하는 쓰지지연

```java
Member member = new Member(150L, "A");
Member member2 = new Member(160L, "B");

em.persist(member);
em.persist(member2); // 엔티티 등록은 아직 수행되지 않는다. (쓰지 지연)

System.out.println("=========");

tx.commit(); // 에티티 등록은 이 떄 동시에 수행된다.
```



<br>



memberA와 memberrB는 둘 다 쓰기지연 SQL저장소에 저장되어있고 실제 DB에 적용은 안된 상태

![image](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2053.png)



<br>



commit() 시점에 쓰기지연 SQL에 저장된 쿼리들을 다 실행시켜서 DB에 적용한다.

![image](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2054.png)



**쓰기 지연을 사용하는 이유**

-> 버퍼링 기능이 제공

```java
<property name="hibernate.jdbc.batch_size" value="10"/>
```

-> 10개씩 쌓일때마다 적용하게 하는 기능

-> 쿼리를 여러번 날리지 않고 최적화가 가능하다.


<br>
<br>



### 4. 엔티티 수정 - 변경감지(Dirty Checking)

```java
transaction.begin();

//엔티티 조회
Member findMember = em.find(Member.class, "memberA");

//영속 엔티티 데이터 수정
findMember.setUsername("hi");
findMember.setAge(10);

transaction.commit();
```

![image](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2055.png)

→ 1차 캐시안에는 @Id, Entity , 스냅샷 이 있다. 여기서 스냅샷 은 최초로 영속성 컨텍스트(1차캐시)에 들어오는순간 스냅샷을 찍어서 저장해둔다.

→ JPA는 트랜잭션이 커밋(commit)되는 순간 엔티티와 스냅샷을 모두 비교한다.

→ 변경된 것이 있을 경우 쓰기지연 SQL 저장소 에 업데이트 쿼리를 저장하고 수행하게 된다.



<br>

<br>



### 5. 엔티티 삭제


```java
//삭제 대상 엔티티 조회
Member member = em.find(Member.class, "memberA");
em.remove(member);//엔티티 삭제
```





<br>

<br>

<br>





## 3. 플러시(flush)

: 영속성 컨텍스트의 변경을 데이터베이스에 반영(sync)

- 플러시 발생

  1. 변경 감지
  2. 수정된 엔티티 쓰기지연 SQL 저장소에 등록
  3. 쓰기 지연 SQL저장소의 쿼리를 데이터베이스에 전송(등록, 수정, 삭제 쿼리)

- 영속성 컨텍스트를 플러시 하는 방법

  1. em.flush() : 직접 호출

  2. 트랜잭션 커밋 : 플러시 자동 호출

  3. JPQL 쿼리 실행 : 플러시 자동 호출

     - JPQL 쿼리 실행 시 자동으로 호출되는 이유

     ```java
     em.persist(memberA);
     em.persist(memberB);
     em.persist(memberC);
     
     //중간에 JPQL 실행
     query = em.createQuery("select m from Member m ", Member.class)
     List<Member> members = query.getResultList();
     ```

     - JPQL쿼리를 실행하는 시점에서 위에 영속화 컨텍스트에 등록한 member들이 조회가 안되는 경우를 막기 위해 JPA에서는 JPQL 쿼리를 수행하기전에 flush를 실행해서 DB와 영속성 컨텍스트간에 동기화를 해준다.



<br>

<br>



### Flush에 대한 오해

flush라는 단어의 뜻 때문에 1차 캐시가 비워진다고 오해할 수 있다. 하지만 flush해도 1차 캐시는 그대로다.1차 캐시를 flush한다는 뜻이 아니고, 쓰기 지연 SQL 저장소를 flush한다고 생각하면 편하다.

flush는 영속성 컨텍스트의 변경내용을 데이터베이스에 동기화하는 일이며, **영속성 컨텍스트는 트랜잭션이라는 작업 단위와 함께 유지된다고 설계**하는 게 좋다. 그러면 데이터 동기화에 대한 걱정이 없어진다.



<br>

<br>

<br>





## 4. 준영속 상태

엔티티를 새로 생성해서 persist하거나, find를 통해 DB에서 조회하는 경우 1차 캐시에 엔티티가 저장된다. 이를 영속 상태라 부른다.

이때 영속(managed) 상태의 엔티티가 영속성 컨텍스트에서 분리(datached)된 상태를 준영속 상태라 부르며, 영속성 컨텍스트가 제공하는 기능을 사용할 수 없다.

- `em.detach(entity) `: 특정 엔티티만 준영속 상태로 전환

- `em.clear()` :  영속성 컨텍스트를 완전히 초기화
- `em.close()` :  영속성 컨텍스트를 종료



<br>

<br>



**Ex1)**

```java

 // 영속
Member member = em.find(Member.class, 150L);
member.setName("AAAAA");

// 준영속
em.detach(member);
System.out.println("==========================");
tx.commit();
```

```java
Hibernate: 
    select
        member0_.id as id1_0_0_,
        member0_.name as name2_0_0_ 
    from
        Member member0_ 
    where
        member0_.id=?
==========================
```



<br>

<br>



**Ex2)**

```java
// 영속
Member member = em.find(Member.class, 150L);
member.setName("AAAAA");

em.clear();

Member member2 = em.find(Member.class, 150L);
System.out.println("==========================");
tx.commit();
```

```java
Hibernate: 
    select
        member0_.id as id1_0_0_,
        member0_.name as name2_0_0_ 
    from
        Member member0_ 
    where
        member0_.id=?
Hibernate: 
    select
        member0_.id as id1_0_0_,
        member0_.name as name2_0_0_ 
    from
        Member member0_ 
    where
        member0_.id=?
==========================
```

