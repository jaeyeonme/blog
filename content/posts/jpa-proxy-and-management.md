---
title: 프록시와 연관관계 관리
date: 2021-12-29 13:40:00
featuredImage: /images/JPA/jpa.png
categories:
  - JPA
tags:
  - JPA
  - Hibernate
---
## 1. 프록시

---

> 프록시란?
> 

테이블을 조회해서 객체를 가져올 때 연관관계 객체는 안가져 오고 싶으면 어떻게 해야 할까?

<br>

- em.find() : 데이터베이스를 통해서 실제 엔티티 객체 조회

- em.getReference(): 데이터베이스 조회를 미루는 가짜(프록시) 엔티티 객체 조회
  
    ```java
    Member member = em.getReference(Member.class, 1L);
    System.out.println("member = " + member.getClass()); // HibernateProxy 객체
    ```
    
    getReference() 메서드를 사용하면 진짜 객체가 아닌 하이버네이트 내부 로직으로 프록시 엔티티 객체 반환
    
    내부 구조는 틀은 같지만 내용이 비어있다.

    {{<image src="https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2022.png" caption="프록시 객체를 반환한다." >}}


<br>

<br>



**특징**

![image](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2023.png)

- 실제 클래스를상속받아서 만들어짐

- 실제 클래스와 겉 모양이 같다.

- 사용하는 입장에서는 진짜 객체인지 구분 필요가 없다. (이론적으로)

- 프록시 객체는 실제 객체의 참조(target)를 보관한다.

- 프록시 객체를 호출(getName())하면 프록시 객체는 실제 객체의 메소드 호출

- 프록시는 처음 사용할 때 한 번만 초기화

- 프록시 객체를 초기화 할 때 프록시 객체가 실제 엔티티로 바뀌는 것은 아님, 초기화되면 프록시 객체를 통해
  실제 엔티티에 접근 가능

- 프록시 객체는 원본 엔티티를 상속받음, 따라서 타입 체크시 주의해야함( == 비교 실패, 대신 instance of 사용)

    ```java
    m1.getClass() == m2.getClass() //false
    m1 instanceof Member
    m2 instanceof Member
    ```

<br>

- 영속성 컨텍스트에 찾는 엔티티가 이미 있으면 em.getReference()를 호출해도 실제 엔티티 반환

    ```java
    Member m1 = em.find(Member.class, member1.getId());
    System.out.println("m1 = "+ m1.getClass());//Member
    
    Member reference = em.getReference(Member.class, member1.getId());
    System.out.println("reference = " reference.getClass()); //Member
    
    m1 == reference //true
    ```

    이미 Member를 1차 캐시에도 올라와 있는데, 프록시를 반환할 필요가 없다.

<br>

- 반대로 getReference()로 프록시객체를 가지고 있으면 실제로 find()를 했을때도 프록시 객체를 반환함

- 영속성 컨텍스트의 도움을 받을 수 없는 준영속 상태일 때, 프록시를 초기화하면 문제 발생

    ```java
    Member refMember = em.getReference(Member.class, member.getId());
    System.out.println("refMember = "+ refMember.getClass());//Proxy
    
    em.detach(refMember);
    //em.clear
    
    refMember.getUsername(); //org.hibernate.LazyInitializationException
    ```




<br>
<br>


**프록시 객체의 초기화**

```java
Member member = em.getRefernce(Member.class, "id1");//(1)
member.getName();//(2)
```

![image](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2024.png)

코드 1라인에서 getReference()를 호출하면 프록시객체를 가져온 다음, getName()을 호출하면

JPA가 영속성 컨텍스트에 초기화 요청을 한다.

영속성 컨텍스트에서는 실제 DB를 조회해서 가져온 다음 실제 Entity에 값을 넣어 생성한 다음 프록시 객체는

실제 엔티티를 연결해서 실제 엔티티를 반환한다.

그 이후에는 이미 초기화되어있는 프록시객체 여기에 해당 엔티티를 반환한다.



<br>



**프록시 확인**
- 프록시 인스턴스의 초기화 여부 확인
: PersistenceUnitUtil.isLoaded(Object entity) → entityManagerFactory.getPersistenceUnitUtil().isLoaded(object)
- 프록시 클래스 확인 방법
: entity.getClass().getname() 출력(..javasist.. or HibernateProxy...)
- 프록시 강제 초기화
: org.hibernate.Hibernate.initialize(entity);
- 참고: JPA 표준은 강제 초기화 없음
강제호출 : method.getName();



<br>

<br>

<br>



## 2. 즉시로딩과 지연로딩

---

### 지연 로딩

Member를 조회할 때 Team(연관관계)도 함께 조회해야 할까?
  : 단순히 member 정보만사용하는 비즈니스 로직

**지연 로딩 LAZY을 사용해서 프록시로 조회**

<br>

**`fetch = FetchType.LAZY`**

```java
/*Member*/
@Entity
public class Member {

    ...
    @ManyToOne(fetch = FetchType.LAZY) //지연로딩 사용
    @JoinColumn(name="TEAM_ID")
    private Team team;
    ...
}

...
Member m = em.find(Member.class, member1.getId());  // Member 객체 반환
System.out.println("m = "+ m.getTeam().getClass()); // Team$HibernateProxy객체 반환
m.getTeam().getName()                               // team을 실제로 사용하는 시점에서 db조회 엔티티 반환
...
```

연관관계에 있는 다른 엔티티를 사용하는 빈도수가 낮을 경우 지연로딩을 사용해 불필요한 엔티티 조회를 막을 수 있다.

<br>

<br>



### 즉시 로딩

❓Member와 Team을 같이 쓰는 빈도가 높을 경우에는 어떻게 해야 할까?



<br>

<br>



**즉시 로딩 EAGER를 사용해서 함꼐 조회
fetch = FetchType.EAGER**

```java
/*Member*/
@Entity
public class Member{
    ...
    @ManyToOne(fetch = FetchType.EAGER) //즉시로딩 사용
    @JoinColumn(name="TEAM_ID")
    private Team team;
    ...
}

...
Member m = em.find(Member.class, member1.getId());  // Member 객체 반환
System.out.println("m = "+ m.getTeam().getClass()); // Team 객체 반환
...
```

Member를 가져오는 시점에서 연관관계에 있는 Team까지 바로 가져오는 것을 즉시 로딩이라고 한다.



<br>

<br>



### 프록시와 즉시로딩 주의

- 가급적으로 지연 로딩만 사용(특히 실무에서)
- 즉시 로딩을 적용하면 예상하지 못한 SQL이 발생 <br>
**ex**: 하나의 엔티티에 연관된 엔티티가 다수라면 find() 한 번 수행시 수십 수백개의 테이블을 한번에...
- 즉시 로딩은 JPQL에서 N+1 문제를 일으킨다. <br>
**ex:**

```java
/*Member*/
@Entity
public class Member{
    ...
    @ManyToOne(fetch = FetchType.EAGER) //즉시로딩 사용
    @JoinColumn(name="TEAM_ID")
    private Team team;
    ...
}

...
List<Member> members = em.createQuery("select m from Member m", Member.class)
				.getResultList();
//SQL: select * from Member
//SQL: select * from Team where TEAM_ID = xxx
...
```

위 JPQL을 그대로 쿼리로 번역하게 되면 Member를 가져오기 위한 쿼리 수행 이후 바로 Member 내부의 <br> Team을 가져오기 위한 쿼리를 다시 수행하게 된다 → N+1(1개의 쿼리를 날리면 +N개의 쿼리가 추가수행된다)

*만약, Team과 같은 연관관계가 더 있다면?*

- @ManyToOne, @OneToOne은 기본이 즉시 로딩으로 되어 있다.→ **직접 전부 LAZY로 설정**
- @OneToMany, @ManyToMany는 기본이 지연 로딩



<br>

<br>



### N+1의 해결책

- 우선 전부 지연로딩으로 설정한다.
  그 다음 가져와야 하는 엔티티에 한해서 fetch join을 사용해서 가져온다.

```java
 List<Member> members = em.createQuery("select m from Member m join fetch m.team", Member.class)
 				.getResultList();
```

 이렇게 JPQL을 실행하면 `fetch join`을 통해 Team 도 가져왔기 때문에 문제가 없다.



<br>

<br>



### 지연 로딩 활용

- Member와 Team은 자주 함께 사용 → 즉시 로딩
- member와 Order는 가끔 사용 → 지연 로딩
- Order와 Product는 자주 함께 사용 → 즉시 로딩

![image](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2025.png)



<br>

<br>



### 지연 로딩 활용 - 실무

- 모든 연관관계에서 지연 로딩을 사용하자.

- 실무에서 즉시 로딩을 사용하지 마라.

- JPQL fetch 조인이나, 엔티티 그래프 기능을 사용해라.

- 즉시 로딩은 내가 의도하지 않은 쿼리가 수행된다.

  

<br>

<br>

<br>



## 3. 영속성 전이 (CASCAD)와 고아 객체

---

**특정 엔티티를 영속 상태로 만들 때 연관된 엔티티도 함께 영속 상태로 만들고 싶을 때 사용.** <br>
- **Ex**: 부모 엔티티를 저장할 때 자식 엔티티도 함께 저장.

![image](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2026.png)



<br>

<br>

**영속성 전이: 저장**

- 영속성 전이가 안되는 기본적인 엔티티 저장 방법
  
    ```java
    /*영속성 전이가 안되는 엔티티 저장 방빕*/
    ...
    @Entity
    public class Parent{
    ...
    @OneToMany(mappedBy = "parent")
    private List<Child> childList = new ArrayList<>();

    public void addChild(Child child){
        childList.add(child);
        child.setParent(this);
    }
    ...
    }
    
    @Entity
    public class Child{
        ...
        @ManyToOne
        @JoinColumn(name = "parent_id")
        private Parent parent;
        ...
    }
    
    ...
    Child child1 = new Child();
    Child child2 = new Child();
    Parent parent = new Parent();
    parent.addChild(child1);
    parent.addChild(child2);
    
    em.persist(parent);
    em.persist(child1);
    em.persist(child2); // persist를 3번이나 해야한다.
    ```
    
    위와 같이 persist를 세 번 호출해야 정상적으로 동작을 한다. 아주 아름답지 않은<?> 코드이다.
    
    persist 한 번으로 child 까지 같이 persist는 불가능한가?

<br>

<br>

- 영속정 전이(CASCADE)를 이용한 엔티티 저장 방법

    ```java
    /*영속성 전이가 안되는 엔티티 저장 방빕*/
    ...
    @Entity
    public class Parent{
    	...
    	@OneToMany(mappedBy = "parent", cascade=CascadeType.ALL)//영속성 전이 속성(CASCADE)사용
    	private List<Child> childList = new ArrayList<>();
    
    	public void addChild(Child child){
    		childList.add(child);
    		child.setParent(this);
    	}
    	...
    }
    
    @Entity
    public class Child{
    	...
    	@ManyToOne
    	@JoinColumn(name = "parent_id")
    	private Parent parent;
    	...
    }
    
    ...
    Child child1 = new Child();
    Child child2 = new Child();
    Parent parent = new Parent();
    parent.addChild(child1);
    parent.addChild(child2);
    
    em.persist(parent);// parent만 persist 해주니 child도 같이 persist된다.
    ```

    parent만 persist해주니 그에 관련된 childList들도 같이 persist가 된다.

<br>

<br>

**CASCADE의 종류**

- `ALL` : 모두 적용(모든 곳에서 맞춰야 하면 해당 옵션)
- `PERSIST` : 영속(저장할 때만 사용할 것이라면 해당 옵션)
- `REMOVE` : 삭제
- `MERGE` : 병합
- `REFRESH` : REFRESH
- `DETACH` : DETACH



<br>

<br>



### 영속성 전이(CASCADE)는 언제 써야 할까?

: 전이 될 대상이 한 군데에서만 사용된다면 써도 된다.

하지만, 해당 엔티티(Child)가 특정 엔티티(Parent)에 종속되지 않고 여러군데서 사용된다면

사요하지 않는게 좋다.

- 라이프 사이클이 동일할 때
- 단일 소유자 관계일 때



<br>

<br>



**고아 객체**

: 고아 객체 제거: 부모 엔티티와 연관관계가 끊어진 자식 엔티티를 자동으로 삭제

- orphanRemoval = true
  
    ```java
    @Entity
    public class Parent {

        ...
        @OneToMany(mappedBy = "parent", cascade=CascadeType.ALL, orphanRemoval = true)
        private List<Child> childList = new ArrayList<>();

        public void addChild(Child child){
            childList.add(child);
            child.setParent(this);
        }
        ...
    }
    
    @Entity
    public class Child{

        ...
        @ManyToOne
        @JoinColumn(name = "parent_id")
        private Parent parent;
        ...
    }
    
    ...
    Child child1 = new Child();
    Child child2 = new Child();
    Parent parent = new Parent();
    parent.addChild(child1);
    parent.addChild(child2);
    
    em.persist(parent);// parent만 persist 해주니 child도 같이 persist된다.
    
    em.flush();
    em.clear();
    
    Parent findParent = em.find(Parent.class, parent.getId());
    findParent.getChildList().remove(0); // orphanRemoval 동작
    ```
    

<br>



**고아 객체 - 주의**

- 참조가 제거된 엔티티는 다른 곳에서 참조하지 않는 고아 객체로 보고 삭제하는 기능
- 참조하는 고이 하나일 때 사용해야 함
- 특정 엔티티가 개인 소유할 때 사용
- @OneToOne, @OneToMany만 가능
- 참고: 개념적으로 부모를 제거하면 자식은 고아가 된다. 따라서 고아 객체 제거 기능을 활성화 하면,
부모를 제거할 때 자식도 함께 제거된다. 마치 CAscadeType.REMOVE처럼 동작한다.
- Parent객체를 지우게 되면 Parent가 소유하고있는 ChildList에 속한엔티티들이 전부 같이 삭제된다.



<br>

<br>



**영속성 전이 + 고아 객체, 생명 주기
(CascadeType.ALL + orphanRemoval=true)**

- 스스로 생명주기를 관리하는 엔티티는 em.persist()로 영속화, em.remove()로 제거
- 두 옵션을 모두 활성화 하면 부모 엔티티를 통해서 자식의 생명주기 관리가 가능하다.
- 도메인 주고 설계(DDD)의 Aggregate Root 개념을 구현할 때 유용



<br>

<br>



**참고: Aggregate Root개념을 알기 위해 우선 Aggregate를 설명해야 한다.**

연관깊은 도메인들을 각각이 아닌 하나의 집합으로 다루는 것을 Aggregate라 한다.  <br> 즉 데이터 변경의 단위로 다루는 연관 객체의묶음.<br>Aggregate에는 루트(root)와 경계(boundary)가 있는데, 경계는 Aggregate에 무엇이 포함되고 포함되지 않는지를 정의한다. <br>
루트는 단 하나만 존재하며, Aggregate에 포함된 특정 엔티티를 가르킨다. <br>
경계안의 객체는 서로 참조가 가능하지만, 경계밖의 객체는 해당 Aggregate의 구성요소 가운데 루트만 참조할 수 있다. <br>

참고: DDD에서 루트 엔티티는 전역 식별성 Global Identity을 가진 엔티티라고 합니다. <br>
- **Ex)**  주문시스템의 Order Entity



<br>

<br>

<br>

