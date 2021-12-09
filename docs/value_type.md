# 값 타입


## 1. JPA 데이터 타입 분류
- 엔티티 타입
    - @Entity로 정의하는 객체다.
    - 데이터가 변경되어도 식별자로 추적이 가능하다.
- 값 타입
    - int, Integer, String처럼 단순히 값으로 사용하는 Java 기본 타입이나 객체다.
    - 식별자가 없고 값만 있으므로 변경시 추적이 불가능하다. (숫자 100을 200으로 변경한다.)

    
## 2. 기본값 타입
- 기본값 타입은 String, int 처럼 자바가 제공하는 기본 데이터 타입이다.
- 식별자 값도 없고 생명주기도 엔티티에 의존한다.
    - 따라서 엔티티 인스턴스를 제거하면, 기본값 타입의 값들도 제거된다.
- 값 타입은 공유하면 안된다.
- int, double같은 Java의 기본 타입은 항상 값을 복사하기 때문에 절대 공유되지 않는다.
- Integer같은 래퍼 클래스나 String 같은 특수한 클래스는 공유 가능하지만 불변객체이다.


## 3. 임베디드 타입(복합 값 타입)
#### Member.java
```java
@Entity
public class Member {
    @Id
    @GeneratedValue
    private Long id;
    
    private String name;
    
    @Embedded
    Period workPeriod;
    
    @Embedded
    Address homeAddress;
}
```

#### Period.java
```java
@Embeddable
public class Period {

    private LocalDateTime startDate;
    private LocalDateTime endDate;
}
```

#### Address
```java
@Embeddable
public class Period {

    private String city;
    private String street;
    private String zipcode;
}
```

- 기본 생성자가 필수다.
- 임베디드 타입은 재사용이 가능하며 Period 및 Address는 각각 고유의 비즈니스 로직을 가질 수 있다.
- 임베디드 타입을 포함한 모든 값 타입은, 값 타입을 소유한 엔티티에 생명주기를 의존한다.
- 임베디드 타입은 엔티티의 값일 뿐이다.
- 임베디드 타입의 사용과 관계없이 매핑되는 테이블은 같다.
    - Period 및 Address 관련 데이터는 별도의 테이블을 생성해서 저장하는 것이 아니라, Member 테이블에 전부 저장되어 있다.
- 객체와 테이블을 아주 세밀하게 매핑하는 것이 가능하다.
    - 잘 설계한 ORM 애플리케이션은 매핑한 테이블의 수보다 클래스의 수가 더 많다.
- 임베디드 타입이 null 이면 매핑한 컬럼 값은 모두 null이 된다.    

### 3.1 속성 재정의
```java
@Entity
public class Member {
    @Id
    @GeneratedValue
    private Long id;
    
    private String name;
    
    @Embedded
    Address homeAddress;

    @Embedded
    @AttributeOverrides({@AttributeOverride(name = "city", column = @Column(name = "COMPANY_CITY")),
            @AttributeOverride(name = "street", column = @Column(name = "COMPANY_STREET")),
            @AttributeOverride(name = "zipcode", column = @Column(name = "COMPANY_ZIPCODE"))
    })
    private Address anotherAddress;
}
```
- 한 엔티티에서 여러 개의 같은 값 타입을 사용하면 컬럼명이 중복된다.
- @AttributeOverride를 통해 매핑 정보를 재정의할 수 있다.


## 4. 값 타입과 불변 객체
#### Application.java
```java
member1.setHomeAddress(new Address("OldCity"));
Address address = member1.getHomeAddress;

address.setCity("NewCity");
member2.setHomeAddress(address);
```

- 임베디드 타입같은 값 타입을 여러 엔티티에서 공유하면 위험하다.
- 위 코드의 의도는 member2의 주소만 NewCity로 변경되길 기대했지만, member1의 주소도 NewCity로 변경되어 버린다.
    - member1과 member2가 같은 address 인스턴스를 참조하기 때문이다.
    - 영속성 컨텍스트는 member1과 member2 둘다 city 속성이 변경된 것으로 판단해서 각각 UPDATE 쿼리를 내보낸다.

#### Address.java
```java
@Embeddable
public class Address {

    private String city;
    // 기본 생성자는 필수다.
    protected Address() {}
    // 생성자로 초기값을 설정한다.
    public Address(String city) {
        this.city = city;
    }
    // Getter는 노출한다.
    public String getCity() {
        return city;
    }
    // Setter는 만들지 않는다.
}
```

- 이런 부작용을 막기 위해 값 타입을 위와 같이 **불변 객체**로 설계한다.
- 불변 객체란 생성 시점 이후 절대 값을 변경할 수 없는 객체이다.
- Integer, String은 자바가 제공하는 대표적인 불변 객체다.


## 5. 값 타입의 비교
- 값 타입은 인스턴스가 달라도 그 안에 값이 같으면 같은 것으로 인식한다.
- 동일성 비교: 인스턴스의 참조 값을 비교, `==` 사용
- 동등성 비교: 인스턴스의 값을 비교, `equals()` 사용
- 값 타입 비교시 `eqauls()`와 `hashcode()`를 모든 필드의 값을 비교하도록 재정의해주어야 한다.


## 6.값 타입 컬렉션
#### Member.java
```java
@Entity
public class Member {
    @Id
    @GeneratedValue
    private Long id;
    
    @Embedded
    private Address homeAddress;
    
    @ElementCollection
    @CollectionTable(name = "FAVORITE_FOODS", 
                     joinColumns = @JoinColimn(name = "MEMBER_ID"))
    @Column(name="FOOD_NAME")
    private Set<String> favoriteFoods = new HashSet<String>();

    @ElementCollection
    @CollectionTable(name = "ADDRESS", 
                     joinColumns = @JoinColumn(name = "member_id"))
    private List<Address> addressHistory = new ArrayList<Address>();
    //...
}
```
- 값 타입을 하나 이상 저장하려면 컬렉션에 보관하고 @ElementCollection, @CollectionTable 어노테이션을 사용하면 된다.
- DB는 컬렉션을 컬럼에 저장할 수 없기 때문에별도의 테이블이 필요하다.
- 값 타입 컬렉션은 영속성 전이(Cascade) + 고아 객체 제거 기능을 필수로 가진다.
    - 생명 주기가 엔티티에 의존한다.
    - 별도로 값 컬렉션을 persist()하지 않아도 상위 엔티티를 영속화할 때 함께 영속화된다.
    - 값 컬렉션에 변화가 생겨도 자동으로 수정된다.
- 값 타입 컬렉션도 기본적으로 지연 로딩 전략을 사용한다.

### 6.1 값 타입 컬렉션의 제약 사항
- 값 타입 컬렉션에 보관된 값의 값이 변경되면 DB에 있는 원본 데이터를 찾기 어렵다.
- 이런 문제로 JPA 구현체들은 값 타입 컬렉션에 변경 사항이 발생하면, 값 타입 컬렉션이 매핑된 테이블의 연관된 모든 데이터를 삭제하고, 현재 값 타입 컬렉션 객체이 있는 모든 값을 DB에 다시 저장한다.
- 또한 값 타입 컬렉션을 매핑하는 테이블은 모든 컬럼을 묶어서 기본키를 구성해야 한다.
    - DB 기본 키 제약 조건으로 인해 컬럼에 null을 입력할 수 없고, 같은 값을 중복해서 저장할 수 없는 제약도 있다.

### 6.2 대안
#### Member.java
```java
@Entity
public class Member {
    //...
    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
    @JoinColumn(name = "MEMBER_ID")
    private List<AddressEntity> addressHistory = new ArrayList<>();
    //...
}
```

#### AddressEntity.java
```java
@Entity
public class AddressEntity {

    @Id
    @GeneratedValue
    private Long id;
    private Address address;
}
```

- 실무에서는 값 타입 컬렉션이 매핑된 테이블에 데이터가 많다면 일대다 관계를 고려한다.
- 위와 같이 일대다 관계 + 영속성 전이(Cascade) + 고아 객체 제거(ORPHAN REMOVE) 기능을 적용하면 값 타입 컬렉션처럼 사용할 수 있다.
