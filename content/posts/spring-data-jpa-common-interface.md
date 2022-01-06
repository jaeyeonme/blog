---
title: 공통 인터페이스 기능
date: 2021-12-31 09:00:00
featuredImage: /images/JPA/spring-data-jpa.png
categories:
  - Spring Data JPA
tags:
  - Spring Data JPA
  - Hibernate
---

## 1. 순수 JPA기반 리포지토리 만들기

- 기본 CRUD
  - 저장
  - 변경 → 변경감지 사용
  - 삭제
  - 전체 조회
  - 단건 조회
  - 카운트

> 참고: JPA에서 수정은 변경감지 기능을 사용하면 된다.
트랜잭션 안에서 엔티티를 조회한 다음에 데이터를 변경하면, 트랜잭션 종료 시점에 변경감지 기능이 작동해서 변경된 엔티티를 감지하고 UPDATE SQL을 실행한다.

<br>

**순수 JPA 기반 리포지토리**

```java
@Repository
public class MemberJpaRepository {

    @PersistenceContext
    private EntityManager em;

    // Create (저장)
    public Member save(Member member) {
        em.persist(member);
        return member;
    }

    // Delete (삭제)
    public void delete(Member member) {
        em.remove(member);
    }

    // Read (조회)
    public List<Member> findAll() {
        // JPQL
        return em.createQuery("select m from Member m", Member.class)
                    .getResultList();
    }

    public Optional<Member> findById(Long id) {
        Member member = em.find(Member.class, id);
        return Optional.ofNullable(member);
    }

    public long count() {
        return em.createQuery("select count(m) from Member m", Long.class)
                    .getSingleResult();
    }

    public Member find(Long id) {
        return em.find(Member.class, id);
    }
}
```

- 회원 리포지토리와 거의 동일하다.



<br>

<br>

<br>



## 2. 공통 인터페이스 설정

### javaConfig 설정 - 스프링 부트 사용시 생략 가능

```java
@Configuration 
@EnableJpaRepositories(basePackages = "jpabook.jpashop.repository") 
public class AppConfig {}
```

- 스프링 부트 사용시 `@SpringBootApplication` 위치를 지정(해당 패키지와 하위 패키지 인식)
- 만약 위치가 달라지면 `@EnableJpaRepositories` 필요



<br>

<br>



### 스프링 데이터 JPA가 구현 클래스 대신 생성

![image](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2045.png)

- `org.springframework.data.repository.Repository` 를 구현한 클래스는 스캔 대상
    - MemberRepository 인터페이스가 동작한 이유
    - 실제 출력해보기(Proxy)
    - memberRepository.getClass() → class com.sun.proxy.$ProxyXXX
- `@Repository` 애노테이션 생략 가능
    - 컴포넌트 스캔 스프링 데이터 JPA가 자동으로 처리
    - JPA 예외를 스프링 예외로 변환하는 과정도 자동으로 처리
    



<br>

<br>

<br>



## 3. 공통 인터페이스 적용

순수 JPA로 구현한 `MemberJpaRepository` 대신에 스프링 데이터 JPA가 제공하는 공통 인터페이스 사용

### 스프링 데이터 JPA 기반 Repository

```java
public interface MemberRepository extends JpaRepository<Member, Long> {
}
```

- Generic
    - `T` : 엔티티 타입
    - `ID` : 식별자 타입(PK)

<br>

<br>

<br>



## 4. 공통 인터페이스 분석

- JpaRepository 인터페이스 : 공통 CURD 제공
- 제네릭은 <엔티티 타입, 식별자 타입> 설정

`JpaRepository` 공통 기능 인터페이스

```java
public interface JpaRepository<T, ID extends Serializable> extends PagingAndSortingRepository<T, ID> { 
	    ...
}
```

<br>

`JpaRepository` 를 사용하는 인터페이스

```java
public interface MemberRepository extends JpaRepository<Member, Long> { 
}
```

<br>

<br>



### 공통 인터페이스 구성

![image](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2046.png)

<br>

**주의**

- `T findOne(ID)` → `Optional<T> findById(ID)` 변경

<br>

**제네릭 타입**

- `T` : 엔티티
- `ID` : 엔티티의 식별자 타입
- `S` : 엔티티와 그 자식 타입

<br>

**주요 메서드**

- `save(S)` : 새로운 엔티티는 저장하고 이미 있는 엔티티는 병합한다.
- `delete(T)` : 엔티티 하나르 삭제한다. 내부에서 `EntityManager.remove()` 호출
- `findById(ID)` : 엔티티 하나를 조회한다. 내부에서 `EntityManager.find()` 호출
- `getOne(ID)` : 엔티티를 프록시로 조회한다. 내부에서 `EntityManager.getReference()` 호출
- `findAll(...)` : 모든 엔티티를 조회한다. 정렬(`Sort`)이나 페이징(`Pageable`) 조건을 파라미터로 제공할 수 있다.

<br>

**참고: `JpaRepository` 는 대부분의 공통 메서드를 제공한다.**