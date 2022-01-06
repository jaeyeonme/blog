---
title: 엔티티매핑
date: 2021-12-29 12:10:00
featuredImage: /images/JPA/jpa.png
categories:
  - JPA
tags:
  - JPA
  - Hibernate
---
## 엔티티 매핑 소개

---

- 객체와 테이블 매핑 : `@Entity` , `@Table`
- 필드와 컬럼 매핑 : `@Column`
- 기본 키 매핑 : `@Id`
- 연관관계 매핑 : `@ManyToOne` , `@JoinColumn`

<br>

<br>



### JPA로 데이터베이스 스키마를 자동 생성하는 방법

JPA로 어플리케이션 로딩 시점에 테이블을 생성할 수 있다. 일일이 스키마를 수정하기 힘든 **개발 환경에서 사용**하면 좋다. <br> 운영에서 쓰려면 절대 그대로 사용해서 안 되고 다듬어야 한다.

엔티티 매핑을 해놓으면, JPA가 어플리케이션 실행 시점에 DDL을 자동 생성하여 수행한다. <br> 객체 중심으로 개발해 놓기만 하면 되니 테이블을 건드릴 필요 없다. <br> 게다가 데이터베이스 방언(dialect)를 활용해 DB에 알맞는 DDL을 만든다.



<br>
<br>


**스프링 부트에선 spring.jpa.hibernate.ddl-auto 속성**으로 스키마 생성 전략을 설정할 수 있다.

1. create : 기존 테이블 삭제 후 다시 생성 (DROP -> CREATE)

2. create-drop : create + 종료 시점 drop (DROP -> CREATE -> DROP)

3. update : 추가된 사항만 반영, 이미 반영된 것을 지우는 작업은 불가능! (특히 운영DB에서 사용하면 안 됨)

4. validate : 엔티티와 테이블이 알맞게 매핑되었는지 검증

5. none : 사용하지 않음

로컬 서버에서 자유롭게 사용하고, 여러 명의 개발자가 사용하는 서버에선 사용하지 않는 게 좋다.<br> `create / create-drop`은 기존 테이블을 드랍하기 때문에 기존 데이터를 날려버리고, <br> update는 alter문을 실행하면서 테이블 lock을 걸기 때문에 조심해야 한다.

<br>

**실무에서는 `validate` 를 사용**



<br>

<br>

<br>



## 1. 객체와 테이블 매핑

### @Entity

---

- @Entity가 붙은 클래스는 JPA가 관리, 엔티티라 한다.
- JPA를 사용해서 테이블과 매핑할 클래스는 @Entity 필수
- 주의
  - `기본 생성자 필수` (파라미터가 없는 public 또는 protected 생성자)
  - final 클래스, enum, interface, inner 클래스 사용X
  - 저장할 필드에 final 사용X


<br>

<br>

### @Entity 속성 정리

---

- 속성 : name
  - JPA에서 사용할 엔티티 이름을 지정한다.
  - 기본값 : 클래스 이름을 그대로 사용 (예: Member)
  - 같은 클래스 이름이 없으면 가급적 기본값을 사용한다.

<br>

<br>

**주의할 점**

1. 기본 생성자 필수 (public or protected)
2. final 클래스 / inner 클래스 / enum / interface 에는 사용 불가능
3. DB 컬럼과 매핑될 필드에 final 사용 불가능



<br>

<br>

### @Table

---

- @Table은 엔티티와 매핑할 테이블 지정



| 속성                   | 기능                                | 기본값             |
| ---------------------- | ----------------------------------- | ------------------ |
| name                   | 매핑할 테이블 이름                  | 엔티티 이름을 사용 |
| catalog                | 데이터베이스 catalog 매핑           |                    |
| schema                 | 데이터베이스 schema 매핑            |                    |
| uniqueConstrints (DDL) | DDL 생성 시에 유니크 제약 조건 생성 |                    |



<br>

<br>

<br>



## 2. 데이터베이스 스키마 자동 생성

---

### 데이터베이스 스키마 자동 생성

- DDL을 어플리케이션 실행 시점에 자동 생성
- 테이블 중심 → 객체 중심
- 데이터베이스 방언을 활용해서 데이터베이스에 맞는 적절한 DDL 생성
- 이렇게 `생성된 DDL은 개발 장비에서만 사용`
- 생성된 DDL은 운영서버에서는 사용되지 않거나 적절히 다듬은 후 사용

<br>

<br>

### 데이터베이스 스키마 자동 생성 - 속성

1. hibernate.hbm2ddl.auto
    1. create : 기존 테이블 삭제 후 다시 생성 (DROP + CREATE)
    2. create-drop : create와 같으나 종료시점에 테이블 DROP
    3. update : 변경분만 반영 (운영DB에는 사용하면 안됨)
    4. validate : 엔티티와 테이블이 정상 매핑되었는지만 확인
    5. none : 사용하지 않음

<br>

<br>

### DDL 생성 기능

- 제약조건 추가 : 회원 이름 `필수` , 10자 초과 X
  - `@Column(nullable = false, length = 10)`
- 유니크 제약조건 추가
  - `@Table(uniqueConstraints = {@UniqueCOnstraint(name = "NAME_AGE_UNIQUE", columnNames = {"NAME", "AGE"})})`
- DDL 생성 기능은 DDL을 자동 생성할 때만 사용되고 JPA의 실행 로직에는 영향을 주지 않는다.

<br>

<br>

<br>

## 3. 필드와 컬럼 매핑

---

### 요구사항 추가

1. 회원은 일반 회원과 관리자로 구분해야 한다.
2. 회원 가입일과 수정일이 있어야 한다.
3. 회원을 설명할 수 있는 필드가 있어야 한다. 이 필드는 길이 제한이 없다.

```java
@Entity
public class Member {
    
    @Id @GeneratedValue
    @Column(name = "MEMBER_ID")
    private Long id;
    
    private String username;
    
    @Enumerated(EnumType.STRING)
    private RoleType roleType;
    
    @Temporal(TemporalType.TIMESTAMP)
    private Date cereatedDate;
    
    private LocalDate lastModifiedDate;
    
    @Lob
    private String description;

    @Transient
    private int tmp_number;
}
```

```java
Hibernate: 
    
    create table Member (
       id bigint not null,
        age integer,
        createdDate timestamp,
        description clob,
        lastModifiedDate timestamp,
        roleType varchar(255),
        name varchar(255),
        primary key (id)
    )
```

<br>

###  @Enumerated

- 자바 Enum 타입을 매핑할 때 사용

- ORIDINAL 타입을 사용하지 말것.
  
    → enum타입이 추가, 변경, 삭제 되어 순서가 달라질 경우 사이드이펙트가 생긴다.
    
- EnumType.ORIGINAL : ENUM 순서를 데이터베이스에 저장

- EnumType.STRING : ENUM 이름을 데이터베이스에 저장

<br>

### @Temporal

- 날짜 타입(Date, Calendar)을 매핑할 때 사용
- LocalDate(년일), LocalDateTime(년월일)을 사용할 때는 생략 가능 (최신 하이버네이트 지원)

<br>

###  @Lob

- 데이터베이스 BLOB, CLOB 타입과 매핑
- 매핑하는 필드 타입이 문자면 CLOB, 나머지는 BLOB 매핑
- CLOB : String, char[], java.sql.CLOB
- BLOB : bypte[], java.sql.BLOB

<br>

### @Transient

- 필드 매핑이 안되게 하는 어노테이션

<br>

### @Column

| 속성                      | 설명                                                         | 기본값                                  |
| ------------------------- | ------------------------------------------------------------ | --------------------------------------- |
| name                      | 필드와 매핑할 테이블의 칼럼 이름                             | 객체의 필드 이름                        |
| insertable, updatable     | 등록, 변경 가능 여부                                         | TRUE                                    |
| nullable (DDL)            | null 값의 허용 여부를 설정. false로 하면 DDL  생성 시에 not null 제약조건 붙음. |                                         |
| unique (DDL)              | @Table의 uniqueConstraints와 같지만 한 컬럼에 유니크 제약조건을 걸 때 사용 |                                         |
| column Defainitaion (DDL) | 데이터베이스 컬럼 정보를 직접 줄 수 있다.<br />ex) varchar(100) default 'EMPTY' | 필드의 자바 타입과 <br>방언 정보를 사용 |
| length (DDL)              | 문자 길이 제약조건, String 타입에만 사용                     | 255                                     |
| precision, scale (DDL)    | BigDecimal 타입에서 사용. <br>precision: 소수점을 포함한 전체 자릿수<br>scale: 소수의 자릿수 다<br>double, float 타입에는 적용되지 않는다. 아주 큰 숫자나 소수를 다뤄야 할 때만 사용 | precision=19,<br>scale=2                |







<br>

<br>

<br>





## 4. 기본 키 매핑

---

1. 직접 할당 : `@Id` 만 사용
2. 자동 생성 (`@GeneratedValue`)(stategy)
    1. IDENTITY
        1. 데이터베이스에 위임, MySQL
        2. 주로 MySQL, PostgreSQL, SQL Server, DB2에서 사용
        3. JPA는 보통 트랜잭션 커밋 시점에 INSERT SQL 수행
        4. AUTO_INCREMENT는 DB에 INSERT SQL을 실행 한 이후에 ID값을 알 수 있다.
            - 영속성 관리 시점에서 1차캐시에 @Id값을 알 수 없다는 말이 된다.
            - 그렇기에 이 케이스에서는 persist()수행 시 바로 insert Query가 수행된다.
            - 그렇기에 IDENTITY 케이스에서는 지연쓰기가 제한된다. <br> (하지만 크게 성능 하락이 있거나 하지는 않다.)
    2. SEQUENCE
        1. 데이터베이스 시퀀스 오브젝트 사용. ORACLE
        2. @SequenceGenerator 필요
           
            ```java
            @Entity
            @SequenceGenerator(
                    name = "MEMBER_SEQ_GENERATOR",
                    sequenceName = "MEMBER_SEQ",
                    initialValue = 1
            )
            public class Member2 {
            		@Id
                @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "MEMBER_SEQ_GENERATOR")
                @Column(name = "MEMBER_ID")
                private Long id;
            ...
            }
            ```
            
        3.  영속화(persist())시 시퀀스에서 next value를 가져와서 해당 값을 @Id에 가지고 1차 캐싱을 해준다.
        4. 지연 쓰기가 가능하다.

1. Table
    1. 키 생성 용 테이블 사용, 모든 DB에서 사용
    2. 키 생성 전용 테이블을 하나 만들어서 데이터베이스 시퀀스를 흉내내는 전략
    3. 모든 데이터 베이스에서 사용할 수 있지만, 성능이 떨어진다.
    4. @TableGenerator 필요
       
        ```java
        @Entity
        @TableGenerator(
                name = "MEMBER_SEQ_GENERATOR",
                table = "MY_SEQUENCES",
                pkColumnName = "MEMBER_SEQ", allocationSize = 1)
        public class Member2 {
        
            @Id
            @GeneratedValue(strategy = GenerationType.TABLE, generator = "MEMBER_SEQ_GENERATOR")
            @Column(name = "MEMBER_ID")
            private Long id;
        ...
        }
        ```
        
    5. 속성
        - `name` : 식별자 생성기 이름 (필수)
        - `table` : 키 생성 테이블 명(MEMBER_SEQ_GENERAGOR)
        - `pkColumnName` : 시퀀스 컬럼명(MY_SEQUENCES)
        - `pkColumnName` : 시퀀스 값 컬럼명
        - `pkColumnValue` : 키로 사용할 값 이름
        - `initialValue` : 초기 값, 마지막으로 생성된 값이 기준이다.
        - `allocationSize` : 시퀀스 한 번 호출에 증가하는 수(성능 최적촤에 사용됨)
        - `catalog` , `schema` : 데이터베이스 catalog, schema 이름
        - `uniqueConstraints(DDL)` : 유니크 제약 조건을 지정할 수 있다.
    
2. AUTO : 방언에 따라 자동 지정, 기본값

1. 권장하는 식별자 전략
    1. 기본 키 제약 조건: `not null`, `unique`, `not update`
    2. 이 조건을 계속 만족하는 자연키는 찾기 힘들기 때문에 대리키(대체키)를 사용하자.
    3. 예를 들면 주민등록번호도 기본 키로 적절하지 않다. → 기본 키가 주민등록번호가 되면 연관매핑을 맺은 다른 테이블에서도 외래키로 주민번호를 사용하기에 여기저기에 개인정보가 퍼지게 된다.
    4. 권장 : Long형 + 대체키 + 키 생성전략 사용.(AUTO나 SequenceObject쓰자 + 회사내의 툴대로 사용)

<br>

> 고급: 기본키 생성 전략이 SEQUENCE인 경우

<br>

**Q. 시퀀스를 얻느라 한 번, 엔티티를 저장한다고 한 번... 잦은 네트워크 통신이 발생하는데 더 줄일순 없을까?**

**A. 최적화 방안(`allocationSize`) : 미리 인자값 갯수만큼 가져와서 사용한다.**

```java
@Entity
@SequenceGenerator(
        name = "MEMBER_SEQ_GENERATOR",
        sequenceName = "MEMBER_SEQ",
        initialValue = 1, allocationSize = 50
)
public class Member2 {
		@Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "MEMBER_SEQ_GENERATOR")
    @Column(name = "MEMBER_ID")
    private Long id;
...
}
```

- 미리 50개의 시퀀스를 가져와서 사용한다.

- 그렇다면 1~100까지 100개의 next value를 가져온 다음 커밋이 롤백된다면 시퀀스는 어떻게 되는가?
  
    → 구멍이 된다. 그리고 100까지 구멍으로 두는게 더 낫다.
    
    왜냐하면 시퀀스를 100까지 받았다고 하면 100번이라 로깅도 남기고 비즈니스 로직을 수행하게 되는데, <br> 문제가 생겨 롤백이 되었을때 시퀀스도 롤백되면 로그에는 다시 100이 남을 것이고 다음 비즈니스 로직에서 <br> 시퀀스를 받을 경우 다시 100번이 된다.
    이럴 경우 어디가 정상이고 어디가 비정상 로깅인지 확인이 어려워진다.



<br>

<br>



### 시퀀스 조회의 비용을 줄이는 옵션

---

한 트랜잭션 안에서 여러 개의 엔티티를 저장한다면, 시퀀스의 다음 값을 얻기 위해 네트워크를 계속해서 사용하는 <br> 비용이 걱정될 수 있다. 이 때 시퀀스 한 번 호출에 증가할 수를 지정하는, `allocationSize 프로퍼티를 사용`하면 된다.

시퀀스의 다음값 n개를 한 번에 땡겨오는 옵션이다. 한 번의 호출로 DB의 시퀀스 값을 n개만큼 증가시켜 놓는 것이다. <br> 그렇게 어플리케이션 메모리에서 n개의 값을 사용하다가 다 소진했다면, 다시 한 번 호출해서 n개를 또 할당받을 수 있다.

여러 WAS에서 이 옵션을 사용하더라도 문제 없이 동시성 이슈가 해결된다고 한다.

