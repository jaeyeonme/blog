---
title: 다양한 연관관계 매핑
date: 2021-12-29 13:00:00
featuredImage: /images/JPA/jpa.png
categories:
  - JPA
tags:
  - JPA
  - Hibernate
---
## 1. 연관관계 매핑시 고려사항 3가지

---

- 다중성
  - 다대일 : `@ManyToOne`
  - 일대다 : `@OneToMany`
  - 일대일 : `@OneToOne`
  - 다대다 : `@ManyToMany` <br>
    **→ 다대다는 실무에서 사용하면 안된다.**
  - 단방향, 양방향
    - 테이블
      - 외래 키 하나로 양쪽 조인 가능
      - 방향이라는 개념이 없음
      - 양쪽이 서로 참조하면 양방향
- 연관관계의 주인
  - 테이블은 외래 키 하나로 두 데이블의 연관관계를 찾음
  - 객체 야방향 관계는 A → B, B → A 처럼 참조가 2군데
  - 연관관계의 주인: 외래 키를 관리하는 참조
  - 주인의 반대편 : 외래 키에 영향을 주지않고 단순 조회(참조)만 가능



<br>

<br>

<br>

## 2. 다대일 [N:1]

---

### 다대일(`N:1`) 단방향

- ERD
  
    ![image](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2016.png)
    
    다대일(`N:1`)단방향
    
    - 가장 많이 사용하는 연관관계
    - 다대일의 반대는 일대다

<br>

### 다대일 양방향

- ERD
  
    ![image](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2011.png)
    
    - 외래 키가 있는 쪽이 연관관계의 주인
    - 양쪽을 서로 참조하도록 개발
    - 연관관계가 주인이 아닌 쪽은 단순 조회만 가능하기에 필드만 추가해주면 된다.
    
    ```java
    @OneToMany
    private List<Member> members = new ArrayList<>();
    ```
    



<br>

<br>

<br>

## 3. 일대다 [1:N]

---

### 일(One)이 연관관계의 주인이다.

→ 권장하는 방법은 아니다. 실무에서도 거의 사용되지 않음

- ERD
  
    ![image](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2012.png)
    
    - 테이블 일대다 관계는 항상 다(N) 쪽에 외래 키가 있다.
    - 객체와 테이블의 차이 떄문에 반대편 테이블의 외래키를 관리하는 특이한 구조
    



<br>

<br>

### 권장하지 않는 이유

1. 테이블에서는 항상 다(N) 쪽에 외래키가 있기 때문에 패러다임충돌이 있다.

2. `@JoinColumn` 을 꼭 사용해야 한다. 그렇지 않으면 조인 테이블 방식을 사용한다. (중간에 테이블을 하나 추가함)

3. 실무에서는 테이블이 수십개 이상 운영이 되는데, 관리 및 트레이싱이 어렵다.
   
    **→ Ex) 일대다(`1:N`)에서 저장(save)이 될 때 양쪽 객체를 저장한 뒤 update query를 통해 외래키 설정(3번이나 수행)**



<br>

> 결론: 기본은 다대일(`N:1`)로 구현하다 필오에 의해 양방향 다대일(`N:1`) 관계를 수립하도록 하자.
> 



<br>

<br>



### 일대다(`1:N`)양방향

- ERD
  
    ![image](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2013.png)
    
    - 이런 매핑은 공식적으로는 존재하지 않는다.
    
    - `@JoinColumn(insertable=false, updatable=false)`
      
        ```java
        /* 팀(Team) */
        public class Team{
                ...
        @OneToMany
        @JoinColumn(name="TEAM_ID")
        private List<Member> members = new ArrayList<>();
        		...
        }
        
        /* 멤버(Member) */
        public class Member{
        		...
        @ManyToOne
        @JoinColumn(name="TEAM_ID", insertable=false, updatable=false)
        private Team team;
        		...
        }
        ```
        
        **→ Member ENtity의 team field가 읽기전용 field가 됐다.**



<br>

- 읽기 전용 필드를 사용해서 양방향 처럼 사용하는 방법
- 다대일 양방향을 사용하자.



<br>

<br>

<br>



## 4. 일대일(1:1)

---

![image](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2014.png)

- 주 테이블이나 대상 테이블 중에 외래 키 선택 가능
- 외래키에 데이터베이스 유니크 제약조건 추가
- 다대일 연관관계와 동일하게 외래키가 있는곳이 연관관계의 주인
- 연관관계의 주인이 아닌 곳에 mappedBy를 넣어준다.
- 정리
    1. 주 테이블에 외래 키
        - 주 객체가 대상 객체의 참조를 가지는 것처럼 주 테이블에 외래 키를 두고 대상 테이블을 찾음
        - 객체지향 개발자 선호
        - JPA 매핑 정리
        - 장점 : 주 테이블만 조회해도 대상 테이블에 데이터가 있는지 확인 가능
        - 단점 : 값이 없으면 외래 키에 null 허용
    2. 대상 테이블의 외래 키
        - 대상 테이블에 외래 키가 존재
        - 전통적인 데이터베이스 개발자 선호
        - 장점 : 주 테이블과 대상 테이블을 일대일에서 일대다 관계로 변경할 때 테이블 구조 유지
        - 단점 : 주 테이블에는 외래 키가 없기 때문에 대상 테이블이 있는지 없는지 알 수 없기 때문에 즉시로딩이 무조건 된다.
        



<br>

<br>

<br>



## 5. 다대다 [N:M]

---

- 실무에서는 거으 ㅣ사용하지도 않고 추천하지도 않는 연관관계
- 객체는 컬렉션을 사용해서 객체 2개로 다대다 관계 가능
- @ManyToMany 사용
- @JoinTable로 연결 테이블 지정



<br>

<br>



### 다대다 매핑의 한계

- 편리해 보이지만 실무에서 사용안함
- 연겵 테이블이 단순히 연결만 하고 끝나지 않음
- 주문시간, 수량 같은 데이터가 들어올 수 있음. 그런데 못 들어옴
- 중간테이블에 추가적인 데이터를 넣을 수 없다는 한계점 존재.
- 중간테이블이 숨겨져 있기 떄문에 의도치 않은 쿼리가 생성될 수 있음



<br>

<br>



### 다대다 한계 극복

- 연결 테이블용 엔티티 추가(연결 테이블을 엔티티로 승격)
    - Ex: Order와 Item 사이에 ORderItem 연결 테이블을 엔티티로 추가
- @ManyToMany → @OneToMany, @ManyToOne
- 아래의 Order 테이블과 다르게 MEMBER_ID, PRODUCT_ID를 복합키로 PK 선언도 가능하지만, <br>
새로운 프라이머리 키를 선언해서 사용하는게 조금 더 선호됨

![image](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2015.png)




