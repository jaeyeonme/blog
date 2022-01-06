---
title: QueryDSL 기본 문법
date: 2021-12-31 12:00:00
featuredImage: /images/JPA/querydsl.png
categories:
  - QueryDSL
tags:
  - QueryDSL
---
## JPQL VS QueryDSL

### QueryDSL vs JPQL

```java
@Test
public void startJQPL() {
    // member1을 찾아라
    String qlString =
            "select m from Member m " +
            "where m.username = :username";

    Member findMember = em.createQuery(qlString, Member.class)
            .setParameter("username", "member1")
            .getSingleResult();

    assertThat(findMember.getUsername()).isEqualTo("member1");
}

@Test
public void startQuerydsl() {
    // member1을 찾아라.
    JPAQueryFactory queryFactory = new JPAQueryFactory(em);
    QMember m = new QMember("m");

    Member findMember = queryFactory
            .select(m)
            .from(m)
            .where(m.username.eq("member1")) // 파라미터 바인딩 처리
            .fetchOne();

    assertThat(findMember.getUsername()).isEqualTo("member1");
}

```

- `EntityManager` 로 `JPAQueryFactory` 생성
- Querydsl은 JPQL 빌더
- `JPQL` : 문자(실행 시점 오류), Querydsl : 코드(컴파일 시점 오류)
- `JPQL` : 파라미터 바인딩 직접, Querydsl : 파라미터 바인딩 자동 처리

<br>

<br>

### JPAQueryFactory를 필드로

```java
@SpringBootTest
@Transactional
public class QuerydslBasicTest {

    @Autowired EntityManager em;
    JPAQueryFactory queryFactory;

    @BeforeEach
    public void before() {
        queryFactory = new JPAQueryFactory(em);
        // ...
		}

		@Test
    public void startQuerydsl() {
        // member1을 찾아라.
        QMember m = new QMember("m");

        Member findMember = queryFactory
                .select(m)
                .from(m)
                .where(m.username.eq("member1")) // 파라미터 바인딩 처리
                .fetchOne();

        assertThat(findMember.getUsername()).isEqualTo("member1");
    }
}
```

- JPAQueryFactory를 필드로 제공하면 동시성 문제는 어떻게 될까? 동시성 문제는 JPAQueryFactory를 생성할 때 <br> 제공하는 EntityManager(em)에 달려있다. 스프링 프레임워크는 여러 쓰레드에서 동시에 같은 <br> EntityManager에 접근해도, 트랜잭션 마다 별도의 영속성 컨텍스트를 제공하기 때문에, <br> 동시성 문제는 걱정하지 않아도 된다.

<br>

<br>

<br>

## 기본 Q-Type 활용

### Q클래스 인스턴스를 사용하는 2가지 방법

```java
QMember qMember = new QMember("m"); //별칭 직접 지정 
QMember qMember = QMember.member;   //기본 인스턴스 사용
```

<br>

<br>

### 기본 인스턴스를 static import와 함께 사용

```java
@Test
public void startQuerydsl() {
    Member findMember = queryFactory
            .select(QMember.member)
            .from(QMember.member)
            .where(QMember.member.username.eq("member1")) // 파라미터 바인딩 처리
            .fetchOne();

    assertThat(findMember.getUsername()).isEqualTo("member1");
}
```

<br>

다음 설정을 추가하면 실행되는 JPQL을 볼 수 있다.

```java
spring.jpa.properties.hibernate.use_sql_comments: true
```

<br>

**참고**: 같은 테이블을 조인해야 하는 경우가 아니면 기본 인스턴스를 사용하자

<br>

<br>

<br>

## 검색 조건 쿼리

### 기본 검색 쿼리

```java
@Test
public void search() {
    Member findMember = queryFactory
            .selectFrom(member)
            .where(member.username.eq("member1")
                    .and(member.age.eq(10)))
            .fetchOne();

    assertThat(findMember.getUsername()).isEqualTo("member1");
}
```

- 검색 조건은 `.and()` , `.or()` 를 메서드 체인으로 연결할 수 있다.
- `select` 와 `from` 은 `selectFrom` 으로 합칠 수 있다.

<br>

<br>

### JPQL이 제공하는 모든 검색 조건 제공

```java
member.username.eq("member1") // username = 'member1' 
member.username.ne("member1") //username != 'member1' 
member.username.eq("member1").not() // username != 'member1'

member.username.isNotNull()   //이름이 is not null

member.age.in(10, 20)         // age in (10,20) 
member.age.notIn(10, 20)      // age not in (10, 20) 
member.age.between(10,30)     // between 10, 30

member.age.goe(30)            // age >= 30 
member.age.gt(30)             // age > 30 
member.age.loe(30)            // age <= 30 
member.age.lt(30)             // age < 30

member.username.like("member%")       // like 검색 
member.username.contains("member")    // like ‘%member%’ 검색 
member.username.startsWith("member")  // like ‘member%’ 검색 
```

<br>

### AND 조건을 파라미터로 처리

```java
@Test
public void searchAndParam() {
    Member findMember = queryFactory
            .selectFrom(member)
            .where(
                    member.username.eq("member1"),
                    member.age.eq(10))
            .fetchOne();

    assertThat(findMember.getUsername()).isEqualTo("member1");
}
```

- `where()` 에 파라미터로 조건검색을 추가하면 `AND` 조건이 추가됨
- 이 경우 `null` 값은 무시 → 메서드 추출을 활용해서 동적 쿼리를 깔끔하게 만들 수 있음 → 뒤에서 설명

<br>

<br>

<br>

## 결과 조회

- `fetch()` : 리스트 조회, 데이터 없으면 빈 리스트 반환
- `fetchOne()` : 단 건 조회
    - 결과가 없으면 : `null`
    - 결과가 둘 이상이면 : `com.querydsl.core.NonUniqueResultException`
- `fetchFirst()` : `limit(1).fetchOne()`
- `fetchResults()` : 페이징 정보 포함, total count 쿼리 추가 실행
- `fetchCount()` : count 쿼리로 변경해서 count 수 조회

```java
// List
List<Member> fetch = queryFactory
        .selectFrom(member)
        .fetch();

// 단 건
Member fetchOne = queryFactory
        .selectFrom(QMember.member)
        .fetchOne();

// 처음 한 건 조회
Member fetchFirst = queryFactory
        .selectFrom(QMember.member)
        .fetchFirst();

// 페이징에서 사용
QueryResults<Member> results = queryFactory
        .selectFrom(member)
        .fetchResults();

results.getTotal();
List<Member> content = results.getResults();

// count 쿼리로 변경
long total = queryFactory
        .selectFrom(member)
        .fetchCount();
```

<br>

<br>

<br>

## 정렬

```java
/**
 * 회원 정렬 순서
 * 1. 회원 나이 내림차순 (desc)
 * 2. 회원 이름 오름차순 (asc)
 * 단 2에서 회원 이름이 없으면 마지막에 출력 (nulls last)
 */
@Test
public void sort() {
    em.persist(new Member(null, 100));
    em.persist(new Member("member5", 100));
    em.persist(new Member("member6", 100));

    List<Member> result = queryFactory
            .selectFrom(member)
            .where(member.age.eq(100))
            .orderBy(member.age.desc(), member.username.asc().nullsLast())
            .fetch();

    Member member5 = result.get(0);
    Member member6 = result.get(1);
    Member memberNull = result.get(2);
    assertThat(member5.getUsername()).isEqualTo("member5");
    assertThat(member6.getUsername()).isEqualTo("member6");
    assertThat(memberNull.getUsername()).isNull();

}
```

- `desc()` , `asc()` : 일반 정렬
- `nullsLast()` , `nullsFirst()` : null 데이터 순서 부여

<br>

<br>

<br>

## 페이징

### 조회 건수 제한

```java
@Test
public void paging1() {
    List<Member> result = queryFactory
            .selectFrom(member)
            .orderBy(member.username.desc())
            .offset(1)  // 0부터 시작(zero index)
            .limit(2)   // 최대 2건 조회
            .fetch();

    assertThat(result.size()).isEqualTo(2);
}
```

- 페이징할 때는 orderBy를 넣어서 정렬을 해줘야 잘 동작한다. <br> offset(1).limit(2)는 index(0)을 생략하고 두 개를 선택한다는 뜻
→ `[0][1][2][3]`

<br>

<br>

### 전체 조회 수가 필요하면?

```java
@Test
public void paging2() {
    QueryResults<Member> queryResults = queryFactory
            .selectFrom(member)
            .orderBy(member.username.desc())
            .offset(1)
            .limit(2)
            .fetchResults();

    assertThat(queryResults.getTotal()).isEqualTo(4);
    assertThat(queryResults.getLimit()).isEqualTo(2);
    assertThat(queryResults.getOffset()).isEqualTo(1);
    assertThat(queryResults.getResults().size()).isEqualTo(2);

}
```

<br>

**주의: count 쿼리가 실행되니 성능상 주의!**

<br>

**참고** <br> 실무에서 페이징 쿼리를 작성할 때, 데이터를 조회하는 쿼리는 여러 테이블을 조인해야 하지만, <br> count 쿼리는 조인이 필요 없는 경우도 있다. 그런데 이렇게 자동화된 count 쿼리는 원본 쿼리와 같이 모두 조인을 <br> 해버리기 때문에 성능이 안나올 수 있다. <br> count 쿼리에 조인이 필요없는 성능 최적화가 필요하다면, count 전용 쿼리를 별도로 작성해야 한다.


<br>

<br>

<br>

## 집합

```java
/**
 * JPQL
 * select
 *    COUNT(m),    // 회원수
 *    SUM(m.age),  // 나이 합
 *    AVG(m.age),  // 평균 나이
 *    MAX(m.age),  // 최대 나이
 *    MIN(m.age)   // 최소 나이
 * from Member m
 */
@Test
public void aggregation() {
    List<Tuple> result = queryFactory
            .select(
                    member.count(),
                    member.age.sum(),
                    member.age.avg(),
                    member.age.max(),
                    member.age.min()
            )
            .from(member)
            .fetch();

    Tuple tuple = result.get(0);
    assertThat(tuple.get(member.count())).isEqualTo(4);
    assertThat(tuple.get(member.age.sum())).isEqualTo(100);
    assertThat(tuple.get(member.age.avg())).isEqualTo(25);
    assertThat(tuple.get(member.age.max())).isEqualTo(40);
    assertThat(tuple.get(member.age.min())).isEqualTo(10);

}
```

- JPQL이 제공하는 모든 집합 합수를 제공한다.
- tuple은 프로젝션과 결과반환에서 설명한다.

<br>

<br>

### GroupBy 사용

```java
/**
 * 팀의 이름과 각 팀의 평균 연령을 구해라.
 */
@Test
public void group() throws Exception {
    List<Tuple> result = queryFactory
            .select(team.name, member.age.avg())
            .from(member)
            .join(member.team, team)
            .groupBy(team.name)
            .fetch();

    Tuple teamA = result.get(0);
    Tuple teamB = result.get(1);

    assertThat(teamA.get(team.name)).isEqualTo("teamA");
    assertThat(teamA.get(member.age.avg())).isEqualTo(15);  // (10 + 20) / 2
    assertThat(teamB.get(team.name)).isEqualTo("teamB");
    assertThat(teamB.get(member.age.avg())).isEqualTo(35);  // (30 + 40) / 2
    
}
```

`groupBy` , 그룹화된 결과를 제한하려면 `having`

<br>

<br>

### groupBy(), having() 예시

```java
.groupBy(item.price)
.having(item.price.gt(1000))
```

<br>

<br>

<br>

## 조인 - 기본 조인

### 기본 조인

조인의 기본 문법은 첫 번째 파라미터에 조인 대상을 지정하고, <br> 두 번째 파라미터에 별칭(alias)으로 사용할 Q 타입을 지정하면 된다.

```java
join(조인 대상, 별칭으로 사용할 Q타입)
```

<br>

**기본 조인**

```java
/**
 * 팀 A에 소속된 모든 회원
 */
@Test
public void join() {
    List<Member> result = queryFactory
            .selectFrom(member)
            .join(member.team, team)
            .where(team.name.eq("teamA"))
            .fetch();

    assertThat(result)
            .extracting("username")
            .containsExactly("member1", "member2");
}
```

- `join()`, `innerJoin()` : 내부 조인(inner join)
- `leftJoin()` : left 외부 조인(left outer join)
- `rightJoin()` : right 외부 조인(right outer join)
- JPQL의 `on` 과 성능 최적화를 위한 `fetch` 조인 제공 → 다음 on 절에서 설명

<br>
<br>

### 세타 조인

연관관계가 없는 필드로 조인

```java
/**
 * 세타 조인
 * 회원의 이름이 팀 이름과 같은 회원 조회
 */
@Test
public void theta_join() {
    em.persist(new Member("teamA"));
    em.persist(new Member("teamB"));
    em.persist(new Member("teamC"));

    List<Member> result = queryFactory
            .select(member)
            .from(member, team)
            .where(member.username.eq(team.name))
            .fetch();

    assertThat(result)
            .extracting("username")
            .containsExactly("teamA", "teamB");
}
```

- from 절에 여러 엔티티를 선택해서 세타 조인
- 외부 조인 불가능 → 다음에 설명한 조인 on을 사용하면 외부 조인가능

<br>

<br>

## 조인 - on절

- ON절을 활용한 조인(JPA 2.1부터 지원)
    1. 조인 대상 필터링
    2. 연관관계 없는 엔티티 외부 조인
    

<br>
<br>

### 1. 조인 대상 필터링

예) 회원과 팀을 조인하면서, 팀 이름이 teamA인 팀만 조인, 회원은 모두 조회

```java
/**
 * 예) 회원과 팀을 조인하면서, 팀 이름이 teamA인 팀만 조인, 회원은 모두 조회
 * JPQL: select m, t from Member m left join m.team t on t.name = 'teamA';
 *
 * @throws Exception
 */
@Test
public void join_on_filtering() throws Exception {
    List<Tuple> result = queryFactory
            .select(member, team)
            .from(member)
            .leftjoin(member.team, team).on(team.name.eq("teamA"))
            .fetch();
    for (Tuple tuple : result) {
        System.out.println("tuple = " + tuple);
    }
}
/* 실행 결과 */
/*
tuple = [Member(id=3, username=member1, age=10), Team(id=1, name=teamA)]
tuple = [Member(id=4, username=member2, age=20), Team(id=1, name=teamA)]
tuple = [Member(id=5, username=member3, age=30), null]
tuple = [Member(id=6, username=member4, age=40), null]
*/
```

- `on` 절을 사용하여 조인 대상 필터링 할 때 내부조인(inner Join)을 사용하면 where절에서 필터링 하는 것과 기능이 동일
  
    ```java
    .leftjoin(member.team, team).on(team.name.eq("teamA")) -> .join(member.team, team).on(team.name.eq("teamA"))
    ```
    

<br>

<br>

### 2. 연관관계 없는 엔티티 외부 조인

예) 회원의 이름과 팀의 이름이 같은 대상 **외부 조인**

```java
/**
 * 2. 연관관계 없는 엔티티 외부 조인
 * 예) 회원의 이름과 팀의 이름이 같은 대상 외부 조인
 * JPQL : SELECT m, t FROM Member m LEFT JOIN Team t on m.username = t.name
 * SQL  : SELECT m.*, t.* FROM Member m LEFT JOIN Team t ON m.username = t.username
 */
@Test
public void join_on_no_relation() {
    em.persist(new Member("teamA"));
    em.persist(new Member("teamB"));
    em.persist(new Member("teamC"));

    List<Tuple> result = queryFactory
            .select(member, team)
            .from(member)
            .leftJoin(team).on(member.username.eq(team.name))
            .fetch();

    for (Tuple tuple : result) {
        System.out.println("tuple = " + tuple);
    }
}
/* 실행 결과 */
/*
t=[Member(id=3, username=member1, age=10), null]
t=[Member(id=4, username=member2, age=20), null]
t=[Member(id=5, username=member3, age=30), null]
t=[Member(id=6, username=member4, age=40), null]
t=[Member(id=7, username=teamA, age=0), Team(id=1, name=teamA)]
t=[Member(id=8, username=teamB, age=0), Team(id=2, name=teamB)]
*/
```

- 하이버네이트 5.1 부터 `on` 을 사용해서 바로 서로 관계가 없는 필드로 외부 조인하는 기능이 추가되었다. <br> 물론 내부 조인도 가능하다.
- **주의!** 문법을 잘 봐야 한다. **leftJoin()** 부분에 일반 조인과 다르게 엔티티 하나만 들어간다.
  - 일반조인 : `leftJoin(member.Team, team)`
  - on 조인 : `from(member).leftJoin(team).on(xxx)`

<br>

<br>

<br>

## 조인 - 페치 조인

**페치 조인은 SQL 자체적으로 제공하는 기능은 아니고 JPA에서 SQL조인을 활용해서 연관된 엔티티를 <br> SQL 한 번에 조회해 가져오는 기능이다.<br> <br>
성능 최적화를 위해 엔티티 연관관계에서 모든 로딩전략을 `지연로딩 (fetch = fetchType.LAZA)`으로 설정하는데, <br> 페치조인을 사용하면,
getter 호출시마다 쿼리를 수행하는 것을 막을 수 있다.**

<br>
<br>

### Before :: 페치 조인 미적용

지연로딩으로 Member, Team SQL 쿼리 각각 실행

```java
@PersistenceUnit
EntityManagerFactory emf;

@Test
public void fetchJoinNo() throws Exception {
    em.flush();
    em.clear();

    Member member1 = queryFactory
            .selectFrom(member)
            .where(member.username.eq("member1"))
            .fetchOne();

    boolean loaded = emf.getPersistenceUnitUtil().isLoaded(member1.getTeam());
    assertThat(loaded).as("페치 조인 미적용").isFalse();
}

/* 수행 쿼리 */
/*
        select member1
        from Member member1
        where member1.username = 'member1'
*/
```

- Member Entity만 조회되었뿐 연관관계게 있는 Team Entity는 조회되지 않았다. <br>
→ 그렇기 때문에 `assertThat(loaded).as("페치 조인 미적용").isFalse();` 은 통과 된다.

<br>

<br>

### After :: 페치 조인 적용

즉시로딩으로 Member, Team SQL 쿼리 조인으로 한번에 조회

```java
@Test
public void fetchJoinUse() throws Exception {
    em.flush();
    em.clear();

    Member member1 = queryFactory
            .selectFrom(member)
            .join(member.team, team).fetchJoin()
            .where(member.username.eq("member1"))
            .fetchOne();

    boolean loaded = emf.getPersistenceUnitUtil().isLoaded(member1.getTeam());
    assertThat(loaded).as("페치 조인 적용").isTrue();
}
```

- `join(member.team, team)` 옆을보면 `.fetchJoin()` 을 호출하는 것을 볼 수 있다. <br>
이렇게 `fetchJoin()` 을 호출해주면 연관관계에 있는 Team Entity도 함께 조회하기 때문에 <br> `loaded` 가 true가 된 것을 확인할 수 있다.

<br>

<br>

## 서브 쿼리

`com.querydsl.jpa.JPAExpressions` 사용하여 서브쿼리 사용이 가능하다.

<br>
<br>

### 서브 쿼리 eq 사용

```java
/**
 * 나이가 가장 많은 회원 조회
 */
@Test
public void subQuery() {

    QMember memberSub = new QMember("memberSub");

    List<Member> result = queryFactory
            .selectFrom(member)
            .where(member.age.eq(
                    select(memberSub.age.max())
                            .from(memberSub)
            ))
            .fetch();

    assertThat(result).extracting("age")
            .containsExactly(40);
}
```

<br>

<br>

### 서브 쿼리 goe 사용

```java
/**
 * 나이가 평균 이상인 회원
 */
@Test
public void subQueryGoe() {

    QMember memberSub = new QMember("memberSub");

    List<Member> result = queryFactory
            .selectFrom(member)
            .where(member.age.goe(
                    select(memberSub.age.avg())
                            .from(memberSub)
            ))
            .fetch();

    assertThat(result).extracting("age")
            .containsExactly(30, 40);
}
```

<br>

<br>

### 서브쿼리 여러 건 처리 in 사용

```java
/**
 * 서브쿼리 여러 건 처리, in 사용
 */
@Test
public void subQueryIn() {

    QMember memberSub = new QMember("memberSub");

    List<Member> result = queryFactory
            .selectFrom(member)
            .where(member.age.in(
                    select(memberSub.age)
                            .from(memberSub)
                            .where(memberSub.age.gt(10))
            ))
            .fetch();

    assertThat(result).extracting("age")
            .containsExactly(20, 30, 40);
}
```

<br>

<br>

### select 절에 subquery

```java
@Test
public void selectSubquery() {

    QMember memberSub = new QMember("memberSub");

    List<Tuple> result = queryFactory
            .select(member.username,
                    select(memberSub.age.avg())
                            .from(memberSub))
            .from(member)
            .fetch();

    for (Tuple tuple : result) {
        System.out.println("tuple = " + tuple.get(member.username));
        System.out.println("age = " + tuple.get(JPAExpressions.select(memberSub.age.avg())
                                                    .from(memberSub)));
    }
}
```

<br>

<br>

### static import 활용

```java
import static com.querydsl.jpa.JPAExpressions.select;

List<Member> result = queryFactory
                        .selectFrom(member) 
                        .where(member.age.eq(select(memberSub.age.max()) 
                        .from(memberSub) )) 
                        .fetch();
```

<br>

**from 절의 서브쿼리 한계**

JPA JPQL 서브쿼리의 한계점으로 from 절의 서브쿼리(인라인 뷰)는 지원하지 않는다. 당연히 Querydsl 도 지원하지 않는다. 하이버네이트 구현체를 사용하면 select 절의 서브쿼리는 지원한다. Querydsl도 하이버네이트 구현체를 사용하면 select 절의 서브쿼리를 지원한다.

<br>

**from 절의 서브쿼리 해결방안**

1. 서브쿼리 → `JOIN`으로 변경한다. (가능한 상황도 있고, 불가능한 상황도 있다.)
2. 애플리케이션에서 쿼리를 2번 분리해서 실행한다.
3. `nativeSQL`을 사용한다.

<br>

<br>

<br>

## Case 문

**select, 조건절(where), order by에서 사용 가능**

<br>

### 단순한 조건

```java
@Test
public void basicCase() throws Exception {
    List<String> result = queryFactory
            .select(member.age
                    .when(10).then("열살")
                    .when(20).then("스무살")
                    .otherwise("기타")
            )
            .from(member)
            .fetch();
    for (String s : result) {
        System.out.println("s = " + s);
    }
}
/*실행결과*/
/*
열살
스무살
기타
기타
*/
```

- age가 10살이면 ‘열살’, 20이면 ‘스무살’ 그 밖에는 ‘기타’로 출력

<br>

<br>

### 복잡한 조건

```java
@Test
public void complexCase() {
    List<String> result = queryFactory
            .select(new CaseBuilder()
                    .when(member.age.between(0, 20)).then("0~20살")
                    .when(member.age.between(21, 30)).then("21~30살")
                    .otherwise("기타"))
            .from(member)
            .fetch();

    for (String s : result) {
        System.out.println("s = " + s);
    }
}
```

- CaseBuilder()를 통해서 동작
- basic한 case와는 다르게 `when` 절안에 조건이 들어간다.

<br>

<br>

### orderBy에서 Case문 함께 사용하기 예제

예를 들어서 다음과 같은 임의의 순서로 회원을 출력하고 싶다면?

1. 0 ~ 30살이 아닌 회원을 가장 먼저 출력
2. 0 ~ 20살 회원 출력
3. 21 ~ 30살 회원 출력

```java
/**
 * orderBy에서 Case문 함께 사용하기
 */
@Test
public void orderByCase() {
    NumberExpression<Integer> rankPath = new CaseBuilder()
            .when(member.age.between(0, 20)).then(2)
            .when(member.age.between(21, 30)).then(1)
            .otherwise(3);

    List<Tuple> result = queryFactory
            .select(member.username, member.age, rankPath)
            .from(member)
            .orderBy(rankPath.desc())
            .fetch();

    for (Tuple tuple : result) {
        String username = tuple.get(member.username);
        Integer age = tuple.get(member.age);
        Integer rank = tuple.get(rankPath);
        System.out.println("username = " + username + " age = " + age + " rank = " + rank);
    }
}
```

QueryDSL은 자바 코드로 작성하기 때문에 `rankPath` 처럼 복잡한 조건을 변수로 선언해서 <br> `select` 절, `orderBy` 절에서 함께 사용할 수 있다.

```java
username = member4 age = 40 rank = 3
username = member1 age = 10 rank = 2
username = member2 age = 20 rank = 2
username = member3 age = 30 rank = 1
```

<br>

<br>

<br>

## 상수, 문자 더하기

상수가 필요하면 `Expressions.constant(xxx)` 사용

```java
@Test
public void constant() {
    List<Tuple> result = queryFactory
            .select(member.username, Expressions.constant("A"))
            .from(member)
            .fetch();

    for (Tuple tuple : result) {
        System.out.println("tuple = " + tuple);
    }
}
```

<br>

**참고** <br> 위와 같이 최적화가 가능하면 SQL에 constant 값을 넘기지 않는다. <br> 상수를 더하는 것 처럼 최적화가 어려우면 SQL에 constant 값을 넘긴다.

<br>

<br>

### 문자 더하기 concat

```java
@Test
public void concat() {

    // {username}_{age}
    List<String> result = queryFactory
            .select(member.username.concat("_").concat(member.age.stringValue()))
            .from(member)
            .where(member.username.eq("member1"))
            .fetch();

    for (String s : result) {
        System.out.println("s = " + s);
    }
}
/*실행결과*/
/*
s = member1_10
*/
```

<br>

**참고** <br> `member.age.stringValue()` 부분이 중요한데, 문자가 아닌 다른 타입들은 `stringValue()` 로 문자로 변환할 수 있다. <br> 이 방법은 ENUM을 처리할 떄도 자주 사용한다.