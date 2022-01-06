---
title: 값 타입
date: 2021-12-29 14:00:00
featuredImage: /images/JPA/jpa.png
categories:
  - JPA
tags:
  - JPA
  - Hibernate
---
## 1. 기본값 타입

---

**엔티티 타입**
- @Entity로 정의하는 객체

- 데이터가 변해도 식별자로 지속해서 추적 가능
  
    ⇒ 엔티티 내부의 모든 값들을 바꿔도 식별자만 유지되면 추적이 가능하다는 의미
    
- Ex: 회원 엔티티의 키나 나이 값을 변경해도 식별자로 인식 가능

<br>

**값 타입**
- int, integer, String처럼 단순히 값으로 사용하는 자바 기본 타입이나 객체
- 식별자가 없고 값만 있으므로 변경시 **추적 불가**
- Ex: 숫자 100을 200으로 변경하면 완전히 다른 값으로 대체

<br>

**값 타입 분류**
1. 기본값 타입
    - 자바 기본 타입(int, double)
    - 래퍼 클래스(Integer, Long)
    - String
2. 임베디드 타입(enbedded type, 복합 값 타입)
    - *Ex: 우편번호, 좌표같은 복합 값을 Position클래스로 만들어 쓰려고 하는 것을 임베디드 타입*
3. 컬렉션 값 타입(collection value type)
    - *Java collection(Array, Map, Set)에 값을 넣을수 있는 것을 컬렉션 값 타입이라 한다.*

<br>

**값 타입 : 기본값 타입**
- Ex: String name, int age
- 생명주기를 엔티티에 의존
    - *회원을 삭제하면 이름, 나이 필드도 함께 삭제*
- 값 타입은 공유하면 안된다.
    - *Side Effect → 회원 이름 변경시 다른 회원의 이름도 함께 변경되면 안된다.*

<br>

- **참고: 자바의 기본 타입은 절대 공유가 되지 않는다.**
  - int, double 같은 기본 타입(primitive type)은 절대 공유되지 않는다.
  - 기본 타입은 항상 값을 복사함
  - Integer같은 래퍼 클래스나 String 같은 특수한 클래스는 공유 가능한 객체이지만 변경할 수 없다.

<br>

<br>

<br>

## 2. 임베디드 타입

---

**개요**
- 새로운 값 타입을 직접 정의할 수 있다.
- JPA는 임베디드 타입(embedded type)이라 한다.
- 주로 기본 값 타입을 모아서 만들어서 복합 값 타입이라고도 함
- **int, String과 같은 값 타입**

<br>

<br>

**임베디드 타입 사용법**
- @Embeddable : 값 타입을 정의하는 곳에 표시
- @Embedded : 값 타입을 사용하는 곳에 표시
- 기본 생성자 필수

<br>

<br>

**Example** <br>
: 예제를 통해 알아보는게 이해하기 쉽다.

**회원 엔티티는 이름, 근무 시작일, 근무 종료일, 주소 도시, 주소 번지, 주소 우편번호를 가진다.**
  - *city, street, zipcode는 주소로 합칠 수 있을 것 같다.*
  - *근무 시작일, 근무 종료일은 근무시간으로 합칠 수 있을 것 같다.*

    ![picture](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2027.png)

<br>
<br>

**회원 엔티티의 몇몇 값을 주소, 근무시간으로 합치면 어떻게 될까.**

![picture](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2028.png)

![picture](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2029.png)



<br>

**코드를 통해 좀 더 자세히 알아보자.**

```java
@Embeddable                    //값 타입이 정의되는 곳에 @Embeddable 사용
public class Period {
    private LocalDateTime startDate;
    private LocalDateTime endDate;
		public Period() { }
}

@Embeddable                   //값 타입이 정의되는 곳에 @Embeddable 사용
public class Address {
    private String city;
    private String street;
    private String zipcode;
	public Address() { }
}

@Entity
public class Member {
    @Id @GeneratedValue
    @Column(name = "MEMBER_ID")
    private Long id;
    @Column(name = "USERNAME")
    private String name;
    /*
    //임베디드 타입을 사용하지 않으면 주석 내의 기존 형태로 값 타입으로 선언해줘야 한다.
    //Period
    private LocalDateTime startDate;
    private LocalDateTime endDate;
    //Address
    private String city;
    private String street;
    private String zipcode;
    */

    @Embedded //값 타입이 사용되는 곳에 @Embedded 사용
    private Period workPeriod;

    @Embedded //값 타입이 사용되는 곳에 @Embedded 사용
    private Address homeAddress;
}
```

<br>

**임베디드 타입의 장점**

- 재사용
    - Period나 Address는 다른 객체에서도 사용할 수 있어 재사용성을 높힌다.
- 높은 응집도
- Period.isWork()처럼 해당 값 타입만 사용하는 의미있는 메소드를 만들 수 있음

```java
private boolean isWork(){
	...
}
```

- 임베디드 타입을 포함한 모든 값 타입은, 값 타입을 소유한 엔티티에 생명주기를 의존한다.



<br>

<br>



### 임베디드 타입을 통해 객체를 분리하더라도 테이블은 하나만 매핑된다.

![picture](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2030.png)



<br>

**임베디드 타입과 테이블 매핑**

- 임베디드 타입은 엔티티의 값일 뿐이다.
- 임베디드 타입을 사용하기 전과 후에 매핑하는 테이블은 같다. (자료3 참고)
- 객체와 테이블을 아주 세밀하게(find-grained) 매핑하는 것이 가능하다.
- 잘 설계한 ORM 애플리케이션은 매핑한 테이블의 수 보다 클래스의 수가 더 많음



<br>

**임베디드 타입과 테이블 매핑**

![picture](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2031.png)

- Member Entity는 Address라는 임베디드 타입을 가질 수 있고, Address라는 임베디드 타입 역시 <br> zipcode라는 값 타입을 가질 수 있다.
- Member Entity는 PhoneNumber라는 임베디드 타입을 가질 수 있고,
  **PhoneNumber라는 임베디드 타입은 PhoneEntity라는 Entity를 가질 수 있다.
  임베디드 타입 클래스 안에서 Column도 사용 가능하다**
  
    ```java
    @Embeddable
    public class Address {
    private String city;
    private String street;
    
    @Column(name = "ZIPCODE") // 이것 역시 가능하다.
    private String zipcode;
    	
    private Member member; //가능하다.
    
    public Address() { }
    ```
  



<br>

> **@AttributeOverride : 속성 재정의**

<br>

<br>



###  Member안에 동일한 임베디드 타입이 있다면 어떻게 될까?

```java
@Entity
public class Member {
    @Id @GeneratedValue
    @Column(name = "MEMBER_ID")
    private Long id;
    @Column(name = "USERNAME")
    private String name;
    @Embedded
    private Period workPeriod;

    @Embedded
    private Address homeAddress;

    //workAddress라는 동일한 homeAddress와 동일한 타입이 추가된다면 어떻게 될까.
    @Embedded
    private Address workAddress;
    //error MappingException: Repeated column in mapping for entity
}
```



<br>

<br>



### @AttributeOverride 를 사용해서 컬러 명 속성을 재정의 해준다.

```java
@Embedded
@AttributeOverrides({
    @AttributeOverride(name="city", column = @Column(name="WORK_CITY")),
    @AttributeOverride(name="street", column = @Column(name="WORK_STREET")),
    @AttributeOverride(name="zipcode", column = @Column(name="WORK_STREET"))
})
private Address workAddress;
```

- 위와 같이 @AttributeOverrides를 이용하면 Address의 field명들이 재정의 되어 정상 동작한다.





<br>

<br>

<br>



## 3. 값 타입과 불변 객체

---

### : **값 타입은 복잡한 객체 세상을 조금이라도 단순화 하려고 만든 개념. 따라서 값 타입은 단순하고 안전하게 다룰 수 있어야 한다.**



<br>

**값 타입 공유 참조**
- 임베디드 타입 같은 값 타입을 여러 엔티티에서 공유하면 위험하다.



![picture](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2032.png)



<br>



**시나리오1 - member와 member2가 같은 address를 바라보고 있다.**

```java
Address address = new Address("city", "street", "10000");
Member member = new Member();
member.setUsername("member1");
member.setHomeAddress(address);
em.persist(member);

Member member2 = new Member();
member.setUsername("member2");
member.setHomeAddress(address);
em.persist(member);
```



<br>

**시나리오2 - 시나리오1에서 member가 주소지를 변경하고 싶어서 Address를 수정한다.**

```java
member.getHomeAddress().setCity("newCity");
```



<br>

**SideEffect 발생 - member2의 Address정보까지 바뀌어 버린다.**

![picture](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2033.png)



<br>

**값 타입 복사**

![picture](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2034.png)



<br> <br>



### 값(인스턴스)을 복사해서 사용한다.

```java
Address address = new Address("city", "street", "10000");
Address copyAddress = new Address(address.getCity(), 
                                address.getStreet(), 
                                address.getZipcode());
```



<br>

<br>



### 누군가 실수로 값 복사가 아닌 기존 값을 넣는다면 막을 수 있을까?

**객체 타입의 한계**
 - 항상 값을 복사해서 사용하면 공유참조로 인해 발생하는 부작용을 피할 수 있다.
 - 문제는 임베디드 타입처럼 직접 정의한 값 타입은 자바의 기본타입이 아닌 객체 타입이다.
 - 자바 기본 타입에 값을 대입하면 값을 복사한다.
 - **객체 타입은 참조 값을 직접 대입하는 것을 막을 방법이 없다.**
 - **객체의 공유 참조는 피할 수 없다.**

```java
Address address = new Address("city", "street", "10000");
Address copyAddress = new Address(address.getCity(), 
                                address.getStreet(), 
                                address.getZipcode());
...
member2.setHomAddress(member.getHomeAddress()); //막을 수 없다.
```

- 기본 타입(primitive type)은 `=`으로 값을 복사한다. <br>
하지만, 객체 타입에서 `=` 을 통한 태입은 참조를 전달한다. <br>
⇒ 인스턴스가 하나이기에 같이 변경된다.



<br>

<br>



**불변 객체**
- 객체 타입을 수정할 수 없도록 **부작용을 원천 차단한다.**
- 값 타입은 불변 객체(immutable object)로 설계해야 한다.
- 불변 객체 : 생성 시점 이후 절대 값을 변경할 수 없는 객체
- 생성자로만 값을 수정하고 수정자(Setter)를 만들지 않으면 된다.
- 참고: Integer, String은 자바가 제공하는 대표적인 불변 객체
- Example

```java
public class Address {
    private String city;
    private String street;
    private String zipcode;

    public Address() {
    }
    public Address(String city, String street, String zipcode){
        this.city =city;
        this.street =street;
        this.zipcode =zipcode;
    }
    public String getCity() {
        return city;
    }

    public String getStreet() {
        return street;
    }

    public String getZipcode() {
        return zipcode;
    }

    private void setCity(String city) {
        this.city = city;
    }

    private void setStreet(String street) {
        this.street = street;
    }

    private void setZipcode(String zipcode) {
        this.zipcode = zipcode;
    }
}
```

- 값을 변경해야 하는 경우에는 어떻게 하나요?
    - 새로 만들어 준다.(ex: city가 바뀌게 된다.)

```java
Address newAddress = new Address(newCity, address.getStreet(), address.getZipCode())
member.setHomeAddress(newAddress);
```



<br>

<br>

<br>



## 4. 값 타입의 비교

---

- 값 타입: 인스턴스가 달라도 그 안에 값이 같으면 같은 것으로 봐야 함
  
```java
// primitive type 비교
int a = 10; 
int b = 10;
System.out.println(a == b); //true

// 임베디드 타입(인스턴스) 비교
Address a = new Address("서울", "AAA", 1000);
Address b = new Address("서울", "AAA", 1000);
System.out.println(a == b); //false
```
    
어째서 임베디드 타입의 `==` 비교는 false가 뜨는것인가?
당연하다. 인스턴스가 다르니 다른객체이기 때문이다.

<br>
<br>

***그럼 어떻게 해야할까?***

- 동일설(identity)비교 : 인스턴스의 참조 값을 비교, == 사용

- 동등성(equivalence)비교 : 인스턴스의 값을 비교, equals() 사용

- 값 타입은 a.equals(b)를 사용해서 동등성 비교를 해야 한다.

- 값 타입의 equals()메소드를 적절하게 재정의 해준다.(주로 모든 필드 사용)

```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (o == null || getClass() != o.getClass()) return false;
    Address address = (Address) o;
    return Objects.equals(city, address.city) &&
            Objects.equals(street, address.street) &&
            Objects.equals(zipcode, address.zipcode);
}

@Override
public int hashCode() {
    return Objects.hash(city, street, zipcode);
}

...
...
...
// 임베디드 타입(인스턴스) 비교
Address a = new Address("서울", "AAA", 1000);
Address b = new Address("서울", "AAA", 1000);
System.out.println(a.equals(b));//true 이제 true가 된다.
```




<br>

<br>

<br>



## 5. 값 타입 컬렉션

---

![picture](https://cdn.jsdelivr.net/gh/jaeyeonme/blog@main/static/images/JPA/Untitled%2035.png)

- 값 타입을 하나 이상 저장할 때 사용
- @ElementCollection, @CollectionTable 사용
- 데이터베이스는 컬렉션을 같은 테이블에 저장할 수 없다.
- 컬렉션을 저장하기 위한 별도의 테이블이 필요하다.

<br>

```java
@ElementCollection
@CollectionTable(name = "FAVORITE_FOOD", joinColumns = @JoinColumn(name="MEMBER_ID")
@Column(name = "FOOD_NAME")
private Set<String> favoriteFoods = new HashSet<>();

@ElementCollection
@CollectionTable(name = "addressHistory", joinColumns = @JoinColumn(name="MEMBER_ID")
@Column(name = "ADDRESS")
private List<Address> addressHistory = new ArrayList<>();
```

<br>
<br>

**값 타입 컬렉션 사용**

- 값 타입 조회 예제
  값 타입 컬렉션도 지연 로딩 사용 전략

```java
Member findMember = em.find(Member.class, 1L);
/*
SQL: select MEMBER_ID, street, zipcode, USERNAME from MEMBER 
            where MEMBER_ID = 1L;
*/
```




<br>
<br>

- 값 타입 수정 예제

```java
/*기본적인 임베디드 타입 변경*/
findMember.getHomeAddress().setCity("newCity");// 잘못된 수정

Address add = findMember.getHomeAddress();
findMember.setHomeAddress(new Address("newCity", add.getStreet(), add.getZipcode()));

/*값 타입 컬렉션 수정 예제 - 치킨을 김밥으로 변경*/
findMember.getFavoriteFoods().remove("치킨");
findMember.getFavoriteFoods().add("김밥");

findMember.getAddressHistory().remove(new Address("old1", "street", "10000"));
findMember.getAddressHistory().add(new Address("newCity", "street", "10000"));
```

값 타입 컬렉션은 생명주기를 소유객체에 의존한다.

- 여기서 값 타입 컬렉션 실행 쿼리를 보면 기존 데이터만 삭제하고 신규 데이터를 추가하는 것이 아닌 <br> 값 타입 컬렉션 데이터 전체가 갈아끼워진다.
- **참고:** 값 타입 컬렉션은 영송석 전이(CASCADE)와 고아 객체 제거 기능을 필수로 가진다고 볼 수 있다.



<br>

**값 타입 컬렉션의 제약사항**

- 값 타입은 엔티티와 다르게 식별자 개념이 없다.
- 값은 변경하면 추적이 어렵다.
- 값 타입 컬렉션에서 변경 사항이 발생하면, 주인 엔티티와 연관된 모든 데이터를 삭제하고 값 타입 컬렉션에 있는 <br> 현재 값 모두 다시 저장한다.
- 값 타입 컬렉션을 매핑하는 테이블은 모든 컬럼을 묶어서 기본키를 구성해야한다.
(null 입력x, 중복 저장x)
- **이러한 이유로 실무에선 사용 안하는 걸 추천한다.**
  - 실무에서는 상황에 따라 값 타입 컬렉션 대신 일대다 관계를 고려한다.
  - 일대다 관계를 위한 엔티티를 만들고, 여기에서 값 타입을 사용
  - 영속성 전이(CASCADE)+ 고아 객체 제거를 사용해서 값 타입 컬렉션처럼 사용

```java
@OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
@JoinColumn(name = "MEMBER_ID")
private List<AddressEntity> addressHistory = new ArrayList<>();
```




<br>

**값 타입 컬렉션 사용 시기**

: 정말 단순한 경우 <br>
**Ex**: 좋아하는 음식메뉴 다중 선태과 같이 심플한 자료들



<br>

<br>

### 정리

1. 엔티티 타입의 특징
    - 식별자가 있다.
    - 생명 주기 관리 (값 타입은 생명주기 관리를 주도적으로 할 수 없다.)
    - 공유
2. 값 타입의 특징
    - 식별자가 없다.
    - 생명 주기를 엔티티에 의존
    - 공유하지 않는 것이 안전(복사해서 사용)
    - 불변 객체로 만드는 것이 안전
    

<br>

### 식별자가 필요하고, 지속해서값을추적, 변경해야한다면 그것은값타입이아닌엔티티
