---
title: 연관관계 매핑 기초
date: 2021-12-29 12:50:00
featuredImage: /images/JPA/jpa.png
categories:
  - JPA
tags:
  - JPA
  - Hibernate
---
## 1. 연관관계의 필요성

---

: 객체지향 설계의 목표는 자율적인 객체들의 협력 공동체를 만드는 것이다.

<br>

<br>

<br>



## 2. 객체를 테이블에 맞춰 데이터 중심으로 모델링하면, 협력 관계를 만들 수 없다.

### 2-1. 객체를 테이블에 맞추어 모델링(연관관계가 없는 객체)

![image](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2010.png)



<br>

- 객체를 테이블에 맞추어 모델링 (참조 대신에 외래 키를 그대로 사용)

    ```java
    /* 회원(Member) 엔티티*/
    @Entity
    public class Member {
        
        @Id @GeneratedValue
        private Long id;
        
        @Column(name = "USERNAME")
        private String name;
        
        @Column(name = "TEAM_ID")
        private Long teamId;
    }
    
    /* 팀(Team) 엔티티 */
    @Entity
    public class Team{
        @Id @GeneratedValue
        private Long id;
        private String name;
    }
    ```

    - 객체를 위와 같이 테이블에 맞춰 모델링을 할 경우 생기는 문제는 무엇일까?

<br>

- 객체를 테이블에 맞춰 모델링 했을 경우 DB에 저장&조회하는 로직

    ```java
    /* 팀과 멤버를 저장하는 로직 */
    Team team = new Team();
    team.setName("teamA");
    em.persist(team);
    
    Member member = new Member();
    member.setName("mamber1");
    member.setTeamId(team.getId());
    em.persist(member);
    ```

    - 외래키 식별자를 직접다루고 있는데, 이럴 경우 문제는? → 조회할 때 역시 해당 외래키를 가지고 조인 쿼리를 직접 짜야 한다.
    - **Q. `member1` 이 소속된 팀 정보를 조회하려면 어떻게 해야하는가?**

    <br>

    ```java
    Member findMember = em.find(Member.class, member.getId());
    Long findTeamId = findMember.getTeamId();
    Team findTeam = em.find(Team.class, findTeamId);
    ```

    - 매번 member를 우선 조회한 뒤 외래키를 뽑아 그것으로 팀의 정보를 조회해야 한다.
    → 협력관계를 만들 수 없다.

<br>
<br>

```java
Hibernate: 
    
    create table Member (
       MEMBER_ID bigint not null,
        TEAM_ID bigint,
        USERNAME varchar(255),
        primary key (MEMBER_ID)
    )
Hibernate: 
    
    create table Team (
       TEAM_ID bigint not null,
        name varchar(255),
        primary key (TEAM_ID)
    )
```

> 결론: 외래키를 직접 관리하는 테이블에 맞춘 객체 모델링은 객체간의 협력관계를 만들 수 없고, <br> 객체가 참조를 통해 연관객체를 찾는 다는 사상을 적용할 수 없다. <br>
이 말은 객체지향 프로그래밍의 패러다임을 정면으로 반박하는 것.
> 

> 객체 지향 모델링 (객체 연관관계 사용)



<br>

<br>



![image](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2021.png)



<br>

- 객체 지향적으로 엔티티 설계(객체 연관관계 사용)
  
    ```java
    /* 회원(Member) 엔티티*/
    @Entity
    public class Member {
    
        @Id @GeneratedValue
        private Long id;
    
        @Column(name = "USERNAME")
        private String name;
        
        @ManyToOne
        @JoinColumn(name = "team_id")
        private Team team;
    		
        ... getter, setter
    }
    ```
    
    - `@ManyToOne` `@JoinColumn` 을 통해 멤버(Member)에서 팀(Team)을 참조하도록 헀다.

<br>
<br>

![image](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2022.png)

```java
Team team = new Team();
team.setName("teamA");
em.persist(team);

Member member = new Member();
member.setName("mamber1");
member.setTeam(team);
em.persist(member);

em.flush();
em.clear();

Member findMember = em.find(Member.class, member.getId());
Team findTeam = member.getTeam();
```





<br>

<br>

<br>



## 3. 양방향 연관관계와 연관관계 주인1 - 기본

---

![image](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2023.png)

- 단방향에서 양방향이 된다는 것의 의미는 양측에서 서로 참조할 수 있다는 것이다.
  기존 단방향에서는 <br> `Member` 에서 getTeam()을 통해 `Team` 엔티티를 참조할 수 있지만 `Team` 에서는 `Member` 를 참조할 수 없었다. <br>
  하지만 테이블 연관관계에서는 외래키를 가지고 양측에서 서로를 참조할 수 있다.

- `Team` 객체에 members라는 List를 추가해서 양방향 연관관계를 만들어준다.
  
    ```java
    @Entity
    public class Team{
        ...
        @OneToMany(mappedBy = "team")
        private List<Member> members = new ArrayList<>();
        ...
    }
    ```
    

<br>

- 추가한 `members` 를 확인하는 코드를 작성해보자.

    ```java
    Member findMember = em.find(Member.class, member.getId());
    List<Member> members = findMember.getTeam().getMembers();
    for (Member member1 : members) {
        System.out.println("member1.getName() = " + member1.getName()); // member1.getName() = mamber1
    }
    ```

    - 이제 반대방향으로도 객체 그래프 탐색이 가능해졌다.


<br>

<br>

### 연관관계의 주인과 mappedBy

- mappedBy = 연관관계의 개념에 대해 이해를 어렵게 만드는 주범!
- 객체와 테이블간 연관관계를 맺는 차이를 이해해야 한다.

<br>

<br>

### 객체와 테이블이 관계를 맺는 차이란?

- 객체 연관관계 = 2개
    - 회원 → 팀 연관관계 1개(단방향)
    - 팀 → 회원 연관관계 1개(단방향)
- 테이블 연관관계 = 1개
    - 회원 ←→ 팀의 연관관계 1개(양방향)

<br>

<br>

### 결국 양방향 관계란

- 객체의 양방향 관계는 사실 양방향 관계가 아니라 서로 다른 단방향 관계 2개다.
- 객체를 양방향으로 참조하려면 단방향 연관관계를 2개 만들어야 한다.
    - `A -> B(a.getB())`
    - `B -> A(b.getA())`
- 테이블에서 양방향으로 참조하려면 외래 키 하나로 연관관계를 가진다. (양쪽으로 조인할 수 있다.)

```sql
SELECT * FROM MEMBER M JOIN TEAM T ON M.TEAM_ID = T.TEAM_ID

SELECT * FROM TEAM T JOIN MEMBER M ON T.TEAM_ID = M.TEAM_ID
```



<br>

<br>

### 둘 중 하나를 외래 키로 관리해야 한다.

![image](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2024.png)

- 두 객체에서 서로가 서로를 참조하는 값을 만들어 놓은 상황
    - 여기서 뭘로 DB에서 외래키를 만들어 관리해야 하는가?
      
        → 연관관계의 주인(Owner)를 정해야 한다.

<br>

<br>

### 연관관계의 주인(Owner)

**→ 양방향 매핑 규칙**

- 객체의 두 관계중 하나를 연관관계의 주인으로 지정
- **연관관계의 주인만이 외래 키를 관리(등록, 수정)**
- **주인이 아닌쪽은 읽기만 가능**
- 주인은 mappedBy 속성을 사용하지 않는다.
- 주인이 아니면 mappedBy 속성으로 주인을 지정한다.

<br>

<br>

### 누구를 주인으로 하는가?

- 외래 키가 있는 곳을 주인으로 정하라.
- 여기서는 Member.team이 연관관계의 주인

![image](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2025.png)

- Q. Team에서 외래키를 관리하는 것은 불가능한가?
A. 가능은 하다. 하지만 Team에서 members를 수정하면 Team이 아닌 Member에 업데이트 쿼리가 날라가는 불일치 현상 발생.

> 결론: 외래키가 있는 곳을 주인(Owner)로 결정하라.



<br>

<br>

<br>

## 4. 양방향 연관관계와 연관관계의 주인1 - 주의점

---

### 양방향 매핑시 가장 많이 하는 실수

(연관관계 주인에 값을 입려하지 않음)

```java
Member member = new Member();
member.setName("mamber1");
em.persist(member);

Team team = new Team();
team.setName("teamA");
team.getMembers().add(member);
em.persist(team);

/* 실행 결과 */
```

- 실행결과
  
    ![image](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2026.png)
    
<br>

> 결론: 양방향 매핑시 연관관계의 주인에 값을 입력해야 한다.
순수한 객체 관계를 고려하면 양쪽 다 값을 입력해야 한다.
> 

<br>

```java
Team team = new Team();
team.setName("teamA");
em.persist(team);

Member member = new Member();
member.setName("mamber1");

team.getMembers().add(member);
member.setTeam(team); //연관관계의 주인에 값 입력

em.persist(member);
```

Q. 만약 여기서 `team.getMembers().add(member);` 을 넣지 않는다면 어떤 문제가 있을까?

A. DB에 반영하는데 문제는 생기지 않는다. 하지만, 영속화 컨텍스트의 1차 캐시에 저장된 team에서는 members에 해당 Member가 추가되지 않은 상태이다. 이런 상황에서 team.members를 사용하게 된다면 DB에서 조회하는게 아닌 1차 캐시에서 꺼내 사용하기 때문에 해당 member가 추가되지 않은 결과가 반환 될 것이고, 문제가 생기게 된다. 그렇기 때문에 **양쪽에 모두 값을 세팅해주는게 맞다.**

<br>

<br>

> TIP: 연관관계 편의 메서드를 생성하자.
> 

```java
public void changeTeam(Team team) {
    this.team = team;
    team.getMembers().add(this);
}
```

- 이제 Team을 세팅해주는 시점에서 해당 team에 Member도 같이 추가된다.
- (Optional) 연관관계 편의 메서드같은 옵셔널 메서드는 관례적으로 쓰이는 Getter, Setter가 아닌 사용자 정의 메서드명(임의)으로 정의해주는게 좋다.



<br>

<br>

### 양방향 매핑시 무한 루프를 조심하자.(순환 참조)

**→ Ex: toString(), lombok, JSON 생성 라이브러리**

```java
/* 회원(Member) 엔티티*/
@Entity
public class Member {
    ...
@Override
public String toString() {
    return "Member{" +
            "id=" + id +
            ", name='" + name + '\'' +
            ", team=" + team +
            '}';
}
    ...
}

/* 팀(Team) 엔티티 */
@Entity
public class Team{
...
@Override
public String toString() {
    return "Team{" +
            "id=" + id +
            ", name='" + name + '\'' +
            ", members=" + members +
            '}';
}
    ...
}
```

- 실행결과
  
    ![image](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2027.png)
    



<br>

<br>

### 양방향 매핑 정리

- 단방향 매핑만으로도 이미 연관관계 매핑은 완료
- 양방향 매핑은 반대 방향으로 조회(객체 그래프 탐색) 기능이 추가된 것 뿐
- JPQL에서 역방향으로 탐색할 일이 많음
- **단방향 매핑을 잘 하고 양방향은 필요할 떄 추가해도 됨
(테이블에 영향을 주지 않는다.)**
- 연관관계의 주인은 외래 키의 위치를 기준으로 정해야 함.
    - 비즈니스 로직을 기준으로 연관관계의 주인을 선택하면 안된다.
    
