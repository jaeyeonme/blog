---
title: QueryDSL 중급 문법
date: 2021-12-31 13:00:00
featuredImage: /images/JPA/querydsl.png
categories:
  - QueryDSL
tags:
  - QueryDSL
---
## 프로젝션과 결과 반환 - 기본

프로젝션 : select 대상 지정

<br>

**프로젝션 대상이 하나**

```java
List<String> result = queryFactory
	      .select(member.username)
	      .from(member)
	      .fetch();
```

- 프로젝션 대상이 하나면 타입을 명확하게 지정할 수 있음(`List<String>`)
- 프로젝션 대상이 둘 이상이면 튜플이나 DTO로 조회

<br>

<br>

### 튜플 조회

**프로젝션 대상이 둘 이상일 때 사용**

`com.querydsl.core.Tuple`

```java
List<Tuple> result = queryFactory
        .select(member.username, member.age)
        .from(member)
        .fetch();

for (Tuple tuple : result) {
    String username = tuple.get(member.username);
    Integer age = tuple.get(member.age);
    System.out.println("username = " + username);
    System.out.println("age = " + age);
}
```

- `Tuple` 은 리포지토리 계층안에서 쓰는 정도는 괜찮지만 그 밖의 계층에서는 DTO로 변환해서 쓰는 것이 좋다.
→ `Tuple` 도 결국 Querydsl에 종속적이기 떄문

<br>

<br>

<br>

## 프로젝션과 결과 반환 - DTO 조회

### 순수 JPA에서 DTO 조회

위에서 select 절에서 대상을 지정(프로젝션)해서 가져오고 싶을때 그 대상이 둘 이상이면 Tuple로 가져왔었다. <br>
하지만, 그 외 계층으로 가져갈 때는 DTO로 가져가는게 좋다.  우선 순수 JPA를 이용해서 DTO로 조회하는 코드를 작성해보자.

<br>
<br>

**MemberDto**

```java
@Data
@NoArgsConstructor
public class MemberDto {

    private String username;
    private int age;

    public MemberDto(String username, int age) {
        this.username = username;
        this.age = age;
    }
}
```

<br>
<br>

**순수 JPA에서 DTO 조회 코드**

```java
@Test
public void findDtoByJPQL() {
    List<MemberDto> result = em.createQuery("select new study.querydsl.dto.MemberDto(m.username, m.age) from Member m", MemberDto.class)
            .getResultList();

    for (MemberDto memberDto : result) {
        System.out.println("memberDto = " + memberDto);
    }
}
```

- 순수 JPA에서 DTO를 조회할 때는 new 명령어를 사용해야 함
- DTO의 package이름을 다 적어줘야해서 지저분함
- 생성자 방식만 지원함

<br>

<br>

### QueryDSL 빈 생성 (Bean Population)

결과를 DTO반환할 떄 사용하며 3가지 방법이 존재한다.

- 프로퍼티 접근
- 필드 직접 접근
- 생성자 사용

<br>

<br>

**프로퍼티 접근 - Setter**

```java
@Test
public void findDtoBySetter() {
    List<MemberDto> result = queryFactory
            .select(Projections.bean(MemberDto.class,
                    member.username,
                    member.age))
            .from(member)
            .fetch();

    for (MemberDto memberDto : result) {
        System.out.println("memberDto = " + memberDto);
    }
}
```

- Projections.bean(주입대상클래스, 프로퍼티1, 프로퍼티2...)방식으로 프로퍼티에 값을 주입

<br>

<br>

**필드 직접 접근**

```java
@Test
public void findDtoByField() {
    List<MemberDto> result = queryFactory
            .select(Projections.fields(MemberDto.class,
                    member.username,
                    member.age))
            .from(member)
            .fetch();

    for (MemberDto memberDto : result) {
        System.out.println("memberDto = " + memberDto);
    }
}
```

- Getter, Setter가 없어도 필드에 바로 값을 주입
- Projections.fields(주입대상클래스, 필드1, 필드2...)

<br>

<br>

필드 직접 접근 - **별칭이 다를 때**

```java
@Test
public void findUserDtoByField() throws Exception {
    QMember memberSub = new QMember("memberSub");
    List<UserDto> result = queryFactory
            .select(Projections.fields(UserDto.class,
                    member.username.as("name"),
                    ExpressionUtils.as(JPAExpressions
                            .select(memberSub.age.max())
                            .from(memberSub), "age")
            ))
            .from(member)
            .fetch();

    for (UserDto memberDto : result) {
        System.out.println("memberDto = " + memberDto);
    }
}
```

- 필드나 프로퍼티명 접근 생성방식에서 이름이 다를 때 해결방안을 as(`name`)을 이용해서 해결한다.
- `ExpressionUtils.as(source,alias)` : 필드나 서브 쿼리에 별칭 적용
- `username.as("memberName")` : 필드에 별칭 적용

<br>

<br>

**생성자 적용**

```java
@Test
public void findDtoByConstructor() {
    List<MemberDto> result = queryFactory
            .select(Projections.constructor(MemberDto.class,
                    member.username,
                    member.age))
            .from(member)
            .fetch();

    for (MemberDto memberDto : result) {
        System.out.println("memberDto = " + memberDto);
    }
}
```



<br>

<br>

<br>

## 프로젝션과 결과 반환 - @QueryProjection

프로젝션결과를 반환할 DTO에 `@QueryProjection` 을 생성자에 붙혀준 뒤 <br> `gradle` → `tasks` → `other` → `compileQuerydsl` 을 통해 DTO도 Q타입으로 생성해 준다.

<br>

<br>

### 생성자 + @QueryProjection

```java
@Data
@NoArgsConstructor
public class MemberDto {

    private String username;
    private int age;

    @QueryProjection
    public MemberDto(String username, int age) {
        this.username = username;
        this.age = age;
    }
}
```

- `./gradlew compileQuerydsl`
- `QMemberDto` 생성 확인

<br>

<br>

### @QueryProjection 활용

```java
@Test
public void findDtoByQueryProjection() throws Exception{
    List<MemberDto> result = queryFactory
            .select(new QMemberDto(member.username, member.age))
            .from(member)
            .fetch();

    for (MemberDto memberDto : result) {
        System.out.println("memberDto = " + memberDto);
    }
}
```

이 방법은 컴파일러로 타입을 체크할 수 있으므로 가장 안전한 방법이다. <br> 다만 DTO에 QueryDSL 어노테이션을 유지해야 하는 점과 DTO까지 Q 파일을 생성해야 하는 단점이 있다.

<br>

<br>

**distinct 사용**

```java
List<String> result = queryFactory
            .select(member.username).distinct()
            .from(member)
            .fetch();
```

<br>

**참고**: distinct는 JPQL의 distinct와 같다.


<br>

<br>

<br>

## 동적 쿼리 - BooleanBuilder 사용

예전 순수 JPA에서 동적 쿼리를 해결하는 방법으로 3가지가 있다고 했다.

1. 문자열 조합 → 조건에 따라 문자열을 결합하면 query문을 만들고 parameter를 세팅해주는 방법
2. JPA Criteria → JPA 표준 스펙에서 제공하는 기능.
3. QueryDSL → 오픈소스를 통해 제공되는 기능으로 쿼리 구현을 method로 한다.

<br>

<br>

여기서 QueryDSL을 사용해서 동적 쿼리를 해결하는 방식은 자세히 들어가면 두 가지 방식이 있다.

1. BooleanBuilder
2. Where 다중 파라미터 사용

<br>

<br>

### BooleanBuilder 사용

```java
@Test
public void dynamicQuery_BooleanBuilder() {
    String usernameParam = "member1";
    Integer ageParam = 10;

    List<Member> result = searchMember1(usernameParam, ageParam);
    assertThat(result.size()).isEqualTo(1);
}

private List<Member> searchMember1(String usernameCond, Integer ageCond) {

    BooleanBuilder builder = new BooleanBuilder();
    if (usernameCond != null) {
        builder.and(member.username.eq(usernameCond));
    }

    if (ageCond != null) {
        builder.and(member.age.eq(ageCond));
    }

    return queryFactory
                .selectFrom(member)
                .where(builder)
                .fetch();
}
```

- `if` 문을 통해 usernameCond.ageCond를 builder.and() 메서드를 통해 조건을 넣어주고 있다.

<br>

<br>

### 동적 쿼리 - Where 다중 파라미터 사용

```java
@Test
public void dynamicQuery_whereParam() throws Exception{
    String usernameParam = null;
    Integer ageParam = null;

    List<Member> result = searchMember2(usernameParam, ageParam);
    assertThat(result.size()).isEqualTo(1);
}

private List<Member> searchMember2(String usernameCond, Integer ageCond) {
    return queryFactory
            .selectFrom(member)
            .where(usernameEq(usernameCond), ageEq(ageCond))
            .fetch();
}

private BooleanExpression usernameEq(String usernameCond) {
    return usernameCond != null ? member.username.eq(usernameCond):null;
}

private BooleanExpression ageEq(Integer ageCond) {
    return ageCond != null ? member.age.eq(ageCond):null;
}
```

- `where` 조건에 `null` 값은 무시된다.
- `usernameEq()` 과 같은 메서드는 다른 쿼리에서 재활용 할 수도 있다.
- 쿼리 자체의 가독성이 높아진다.

<br>

<br>

### 메서드들을 조합해서 사용 할 수도 있다.

```java
private BooleanExpression allEq(String usernameCond, Integer ageCond) {
    return usernameEq(usernameCond).and(ageEq(ageCond));
}
```

- `method chaining` 이 있다.
- `null` 체크는 주의해서 처리해야 함

<br>

<br>

<br>

## 수정, 삭제 벌크 연산
### 쿼리 한번으로 대량 데이터 수정

```java
@Test
public void bulkUpdate() throws Exception{
    //member1(10), memberr2(20) -> 비회원
    //member3(30), member4(40) -> 그대로
    long count = queryFactory
            .update(member)
            .set(member.username, "비회원")
            .where(member.age.lt(28))
            .execute();

    em.flush();
    em.clear();

    List<Member> result = queryFactory
            .selectFrom(member)
            .fetch();
    for (Member member1 : result) {
        System.out.println("member1 = " + member1);
    }
}
```

<br>

<br>

### 기존 숫자에 1 더하기

```java
long count = queryFactory
	      .update(member)
	      .set(member.username, "비회원")
	      .where(member.age.lt(28))
	      .execute();
```

- 곱하기를 하려면 `member.age.multiply(x)` 를 사용.

<br>

<br>

### 쿼리 한번으로 대량 데이터 삭제

```java
long count = queryFactory
        .delete(member)
        .where(member.age.gt(18))
        .execute();
```

<br>

**주의** <br> JPQL 배치와 마찬가지로, 영속성 컨텍스트에 있는 엔티티를 무시하고 실행되기 때문에 <br>배치 쿼리를 실행하고 나면 영속성 컨텍스트를 초기화 하는 것이 안전하다.

<br>

<br>

<br>

## SQL function 호출하기

SQL function은 JPA와 같이 Dialect에 등록된 내용만 호출할 수 있다.

<br>

### Replace function 사용해보기

```java
@Test
public void sqlFunction() throws Exception{
    List<String> result = queryFactory
            .select(
                    Expressions.stringTemplate("function('replace', {0},{1},{2})",
                            member.username, "member", "M"))
            .from(member)
            .fetch();
    for (String s : result) {
        System.out.println("s = " + s);
    }
}
/* 실행 결과 */
/*
    M1
    M2
    M3
    M4
*/
```

- MemberX → MX가 된 걸 확인할 수 있다.

<br>

<br>

### lower function 사용해보기

```java
@Test
public void sqlFunction2() throws Exception{
    List<String> result = queryFactory
            .select(member.username)
            .from(member)
            .where(member.username.eq(
                Expressions.stringTemplate("function('lower', {0})", member.username)))
            .fetch();

    for (String s : result) {
        System.out.println("s = " + s);
    }
}
/* 실행 결과 */
/*
    member1
    member2
    member3
    member4
*/
```

- lower같은 ansi 표준 함수들은 querydsl이 대부분 내장하고 있기에 `.lower()` 로 처리해도 된다.
  
    → `.where(member.username.eq(member.username.lower()))`