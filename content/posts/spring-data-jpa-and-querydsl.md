---
title: Spring Data JPA와 QueryDSL
date: 2022-01-01 09:00:00
featuredImage: /images/JPA/querydsl.png
categories:
  - QueryDSL
tags:
  - QueryDSL
---
## 스프링 데이터 JPA 리포지토리로 변경

<br>

### 스프링 데이터 JPA - MemberRepository 생성

```java
package study.querydsl.repository;

import org.springframework.data.jpa.repository.JpaRepository;
import study.querydsl.entity.Member;

import java.util.List;

public interface MemberRepository extends JpaRepository<Member, Long> {

    // select m from m where m.username = ?;
    List<Member> findByUsername(String username);
}
```

<br>

### 스프링 데이터 JPA 테스트

```java
@Test
public void basicTest() throws Exception {
    Member member = new Member("member1", 10);
    memberRepository.save(member);

    Member findMember = memberRepository.findById(member.getId()).get();
    assertThat(findMember).isEqualTo(member);

    List<Member> result1 = memberRepository.findAll();
    assertThat(result1).containsExactly(member);

    List<Member> result2 = memberRepository.findByUsername("member1");
    assertThat(result2).containsExactly(member);

}
```

- QueryDSL 전용 기능인 회원 search를 작성할 수없다. → 사용자 정의 리포지토리 필요

<br>

<br>

<br>

## 사용자 정의 리포지토리

**사용자 정의 리포지토리 사용법**

1. 사용자 정의 인터페이스 작성
2. 사용자 정의 인터페이스 구현
3. 스프링 데이터 리포지토리에 사용자 정의 인터페이스 상속

<br>

<br>

### 사용자 정의 리포지토리 구성

![picture](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2047.png)

<br>

<br>

### 1. 사용자 정의 인터페이스 작성

```java
public interface MemberRepositoryCustom {
    List<Member> findMemberCustom();
}
```

<br>

<br>

### 2. 사용자 정의 인터페이스 구현

```java
public class MemberRepositoryImpl implements MemberRepositoryCustom{

    @Override
    public List<Member> findMemberCustom() {
        return null;
    }

    private final JPAQueryFactory queryFactory;

    public MemberRepositoryImpl(JPAQueryFactory queryFactory) {
        this.queryFactory = queryFactory;
    }

    @Override
    public List<MemberTeamDto> search(MemberSearchCondition condition) {
        return queryFactory
                .select(new QMemberTeamDto(
                        member.id.as("memberId"),
                        member.username,
                        member.age,
                        team.id.as("teamId"),
                        team.name.as("teamName")
                ))
                .from(member)
                .leftJoin(member.team, team)
                .where(
                        usernameEq(condition.getUsername()),
                        teamNameEq(condition.getTeamName()),
                        ageGoe(condition.getAgeGoe()),
                        ageLoe(condition.getAgeLoe())
                )
                .fetch();
    }

    private BooleanExpression usernameEq(String username) {
        return isEmpty(username)?null:member.username.eq(username);
    }
    private BooleanExpression teamNameEq(String teamName) {
        return hasText(teamName) ? team.name.eq(teamName):null;
    }

    private BooleanExpression ageGoe(Integer ageGoe) {
        return ageGoe!= null ? member.age.goe(ageGoe):null;
    }

    private BooleanExpression ageLoe(Integer ageLoe) {
        return ageLoe!= null ? member.age.loe(ageLoe):null;
    }
}
```

<br>

<br>

### 3. 스프링 데이터 리포지토리에 사용자 정의 인터페이스 상속

```java
public interface MemberRepository extends JpaRepository<Member, Long>, MemberRepositoryCustom {

    // select m from Member m where m.username = ?;
    List<Member> findByUsername(String username);
}
```

<br>

<br>

### **사용자 정의 리포지토리 테스트 코드 작성**

```java
@Test
public void searchTest() {
    Team teamA = new Team("teamA");
    Team teamB = new Team("teamB");
    em.persist(teamA);
    em.persist(teamB);

    Member member1 = new Member("member1", 10, teamA);
    Member member2 = new Member("member2", 20, teamA);

    Member member3 = new Member("member3", 30, teamB);
    Member member4 = new Member("member4", 40, teamB);
    em.persist(member1);
    em.persist(member2);
    em.persist(member3);
    em.persist(member4);

    MemberSearchCondition condition = new MemberSearchCondition();
    condition.setAgeGoe(35);
    condition.setAgeLoe(40);
    condition.setTeamName("teamB");

    List<MemberTeamDto> result = memberRepository.search(condition);
    assertThat(result).extracting("username").containsExactly("member4");
}
```

<br>

<br>

<br>

## 스프링 데이터 페이징 활용 1 - Querydsl 페이징 연동

- 스프링 데이터의 Page, Pageable을 활용해보자.
- 전체 카운트를 한번에 조회하는 단순한 방법
- 데이터 내용과 전체 카운트를 별도로 조회하는 방법

<br>

<br>

### 사용자 정의 인터페이스에 페이징 2가지 추가

```java
public interface MemberRepositoryCustom {

    List<MemberTeamDto> search(MemberSearchCondition condition);
    Page<MemberTeamDto> searchPageSimple(MemberSearchCondition condition, Pageable pageable);
    Page<MemberTeamDto> searchPageComplex(MemberSearchCondition condition, Pageable pageable);
}
```

<br>

<br>

### 전체 카운트를 한번에 조회하는 단순한 방법 - searchPageSimple()

```java
@Override
public Page<MemberTeamDto> searchPageSimple(MemberSearchCondition condition, Pageable pageable) {
    QueryResults<MemberTeamDto> results = queryFactory
            .select(new QMemberTeamDto(
                    member.id.as("memberId"),
                    member.username,
                    member.age,
                    team.id.as("teamId"),
                    team.name.as("teamName")
            ))
            .from(member)
            .leftJoin(member.team, team)
            .where(
                    usernameEq(condition.getUsername()),
                    teamNameEq(condition.getTeamName()),
                    ageGoe(condition.getAgeGoe()),
                    ageLoe(condition.getAgeLoe())
            )
            .offset(pageable.getOffset()) // 몇번째부터 시작할 것인가
            .limit(pageable.getPageSize()) // 몇개를 가져올 것인가
            .fetchResults();

    List<MemberTeamDto> content = results.getResults();
    long totalCount = results.getTotal();
    return new PageImpl<>(content, pageable, totalCount);
}
```

- Querydsl이 제공하는 `fetchResults()` 를 사용하면 내용과 전체 카운트를 한번에 조회할 수 있다. (실제 쿼리는 2번 호출)
- `fetchResult()` 는 쿼리 카운트 쿼리 실행시 필요없는 `order by` 는 제거한다.

<br>

<br>

### 테스트 코드 작성

```java
@Test
public void searchPageSimple() throws Exception {
    Team teamA = new Team("teamA");
    Team teamB = new Team("teamB");
    em.persist(teamA);
    em.persist(teamB);

    Member member1 = new Member("member1", 10, teamA);
    Member member2 = new Member("member2", 20, teamA);

    Member member3 = new Member("member3", 30, teamB);
    Member member4 = new Member("member4", 40, teamB);
    em.persist(member1);
    em.persist(member2);
    em.persist(member3);
    em.persist(member4);

    MemberSearchCondition condition = new MemberSearchCondition();
    PageRequest pageRequest = PageRequest.of(0, 3);

    Page<MemberTeamDto> result = memberRepository.searchPageSimple(condition, pageRequest);
    assertThat(result.getSize()).isEqualTo(3);
    assertThat(result.getContent()).extracting("username").containsExactly("member1", "member2", "member3");
}
```

<br>

<br>

### 데이터 내용과 전체 카운트를 별도로 조회하는 방법 - **searchPageComplex()**

```java
@Override
public Page<MemberTeamDto> searchPageComplex(MemberSearchCondition condition, Pageable pageable) {
    List<MemberTeamDto> content = queryFactory
            .select(new QMemberTeamDto(
                    member.id.as("memberId"),
                    member.username,
                    member.age,
                    team.id.as("teamId"),
                    team.name.as("teamName")
            ))
            .from(member)
            .leftJoin(member.team, team)
            .where(
                    usernameEq(condition.getUsername()),
                    teamNameEq(condition.getTeamName()),
                    ageGoe(condition.getAgeGoe()),
                    ageLoe(condition.getAgeLoe())
            )
            .offset(pageable.getOffset())
            .limit(pageable.getPageSize())
            .fetch();

    long total = queryFactory
            .select(member)
            .from(member)
            .leftJoin(member.team, team)
            .where(
                    usernameEq(condition.getUsername()),
                    teamNameEq(condition.getTeamName()),
                    ageGoe(condition.getAgeGoe()),
                    ageLoe(condition.getAgeLoe())
            )
            .fetchCount();

    return new PageImpl<>(content, pageable, total);
}
```

- `fetch()` 를 사용해서 content만 가져 온다.
- totalCount는 `fetchCount()` 를 이용해서 따로 가져온다.
- 이렇게 전체 카운트 조회하는 방법을 분리하면 성능 향상 효과를 기대할 수 있다.
  
    → 조인을 줄여도 되거나, 다른곳에 몰려있거나 하는 경우 별도로 작성해서 성능 향상을 기대할 수 있다.
    

<br>

<br>

<br>

## 스프링 데이터 페이징 활용 2 - CountQuery 최적화

### PageableExecutionUtils.getPage()로 최적화

```java
JPAQuery<Member> countQuery = queryFactory
        .select(member)
        .from(member)
        .leftJoin(member.team, team)
        .where(usernameEq(condition.getUsername()),
                teamNameEq(condition.getTeamName()),
                ageGoe(condition.getAgeGoe()),
                ageLoe(condition.getAgeLoe())
        );

return PageableExecutionUtils.getPage(content, pageable, countQuery::fetchCount);
//        return new PageImpl<>(content, pageable, total);
```

- 스프링 데이터 라이브러리가 제공
- count 쿼리가 생략 가능한 경우 생략해서 처리
    - 페이지 시작이면서 컨텐츠 사이즈가 페이지 사이즈보다 작을 때
    - 마지막 페이지 일 때 (offset + 컨텐츠 사이즈를 더해서 전체 사이즈 구함)
    

<br>

<br>

<br>

## 스프링 데이터 페이징 활용3 - 컨트롤러 개발

### Controller API작성

```java
@RestController
@RequiredArgsConstructor
public class MemberController {

    private final MemberJpaRepository memberJpaRepository;
    private final MemberRepository memberRepository;

    @GetMapping("/v1/members")
    public List<MemberTeamDto> searchMemberV1(MemberSearchCondition condition) {
        return memberJpaRepository.search(condition);
    }

    @GetMapping("/v2/members")
    public Page<MemberTeamDto> searchMemberV2(MemberSearchCondition condition, Pageable pageable) {
        return memberRepository.searchPageSimple(condition, pageable);
    }

    @GetMapping("/v3/members")
    public Page<MemberTeamDto> searchMemberV3(MemberSearchCondition condition, Pageable pageable) {
        return memberRepository.searchPageComplex(condition, pageable);
    }
}
```

- [`http://localhost:8080/v2/members?size=5&page=2`](http://localhost:8080/v2/members?size=5&page=2)

### **스프링 데이터 정렬(Sort)**

스프링 데이터 JPA는 자신의 정렬(Sort)을 Querydsl의 정렬(OrderSpecifier)로 편리하게 변경하는 기능을 제공한다.

<br>

<br>

**스프링 데이터 Sort를 Querydsl의 OrderSpecifier로 변환**

```java
JPAQuery<Member> query = queryFactory.selectFrom(member);
for (Sort.Order o : pageable.getSort()) {
    PathBuilder pathBuilder = new PathBuilder(member.getType(), member.getMetadata());
    query.orderBy(new OrderSpecifier(o.isAscending() ? Order.ASC : Order.DESC, pathBuilder.get(o.getProperty())));
}
List<Member> result = query.fetch();
```

<br>

**참고**: 정렬(`Sort`)은 조건이 조금만 복잡해져도 `Pageable` 의 `Sort` 기능을 사용하기 어렵다. <br> 루트 엔티티 범위를 넘어가는 동적 정렬 기능이 필요하면 스프링 데이터 페이징이 제공하는 `Sort` 를 사용하기 보다는 파라미터를 받아서 직접 처리하는 것을 권장한다.