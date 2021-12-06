# 엔티티 매핑


## 1. @Entity
- `@Entity`가 붙은 클래스는 JPA가 관리하며 엔티티라 부른다.
- 일반적으로 클래스의 이름을 기본값으로 사용한다.
- @Entity 적용시 주의사항
    - final 클래스, enum, 인터페이스, inner 클래스에는 선언할 수 없다.
    - JPA는 내부적으로 엔티티 클래스를 리플렉션을 통해 생성하기 때문에 기본 생성자가 필수다.
    - 저장할 필드는 final 불변으로 선언할 수 없다.


## 2. @Table
- `@Table`은 엔티티와 매핑할 테이블을 지정한다.
- 생략하면 매핑한 엔티티 이름을 테이블 이름으로 기본값으로 사용한다.


## 3. DB 스키마 자동 생성
```xml
<!-- persistence.xml -->
<property name="hibernate.hbm2ddl.auto" value="create"/>
```
- `persistence.xml`에 속성을 추가하여 DB 스키마를 자동 생성할 수 있다.
- DB 방언을 활용해서 DB에 맞는 DDL(Date Definition Language)을 어플리케이션 실행시점에 자동 생성하여 실행한다.
  
### 3.1 DB 스키마 자동 생성 옵션
- create : 기존 테이블을 삭제하고 새로 생성한다. (DROP + CREATE)
- create-drop : create 속성에 추가로 애플리케이션을 종료할 때 생성한 DDL을 제거한다. (DROP + CREATE + DROP)
- update : DB 테이블과 엔티티 매핑정보를 비교해서 변경 사항만 수정한다.
- validate : DB 테이블과 엔티티 매핑정보를 비교해서 차이가 있으면 경고를 남기고 애플리케이션을 실행하지 않는다.
- none : 자동 생성 기능을 사용하지 않는다.


## 4. DDL 생성 기능
- DDL 생성 기능은 DDL을 자동 생성할 때만 사용되고 JPA 실행 로직에는 영향을 끼치지 않는다.
  ```java
  @Column(name = "NAME", nullable = false, length = 10)
  private String username;
  ```


## 5. 기본키 매핑
- 기본키를 직접 할당할 때는 `@Id`만 사용하면 되고, 자동 생성 전략을 사용하려면 `@Id`에 `@GeneratedValue`를 추가하고 원하는 키 생성 전략을 선택하면 된다.
- 키 생성 전략을 사용하려면 아래와 같은 속성을 추가해야 한다.
  ```xml
  <!-- persistence.xml -->
  <property name="hibernate.id.new_generator_mappings" value="true"/>
  ```
  
### 5.1 IDENTITY 전략
```java
@Entity
public class Member {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
}
```
- 기본키 생성을 데이터 베이스에 위임하는 전략이다.
- 주로 MySQL, postgreSQL, SQL server, DB2에서 사용한다.
- 엔티티가 영속 상태가 되려면 식별자가 반드시 필요하다. 그런데 IDENTITY 전략은 엔티티를 DB에 저장해야 식별자를 구할 수 있으므로, em.persist()를 호출하는 즉시 INSERT SQL이 DB에 전달된다.
- 따라서 IDENTITY 전략은 트랙잭션을 지원하는 쓰기 지연이 동작하지 않는다.

### 5.2 SEQUENCE 전략
```java
@Entity
@SequenceGenerator(name = "sequence-generator",     
                    sequenceName = "MEMBER_SEQ",    // DB에 등록되어 있는 시퀀스 이름
                    initialValue = 1,               // DDL 생성시 처음 시작하는 수를 지정
                    allocationSize = 1)             // 시퀀스 한 번 호출에 증가하는 수 (최적화에 사용)
public class Member {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "sequence-generator")
    private Long id;
}
```
- 유일 값을 순서대로 생성하는 특별한 DB 오브젝트를 사용하여 기본키를 생성하는 전략이다.
- 주로  postgreSQL, DB2, H2에서 사용한다.
- SEQUENCE 전략은 한번 쓰기 작업을 수행할 때 SELECT 쿼리 한 번, INSERT 쿼리 한 번, 총 두번의 쿼리를 날리는 문제가 발생한다.
- 이는 IDENTITY 전략과 다르게 `em.persist()`를 호출할 때 먼저 DB 시퀀스를 사용해서 식별자를 조회하기 때문에 발생하는 문제이다.
- 이런 문제를 해결하기 위해 `allocationSize` 속성을 통해 미리 해당 수만큼 시퀀스를 어플리케이션의 메모리에 저장하고, `em.persist()`를 호출할 때마다 메모리에 적재된 시퀀스를 통해 PK를 얻는 방법을 사용한다.
- ex) 시퀀스 초기 값이 1, 기본값이 50이라면 미리 1~50에 해당하는 시퀀스를 어플리케이션의 메모리에 저장한다. 50까지 다 사용하면 51~100에 해당하는 시퀀스를 요청한다.

### 5.3 TABLE 전략
```java
@Entity
@TableGenerator(name = "table-generator", 
                table = "MY_SEQUENCES",         // 키 생성 테이블 명
                pkColumnValue = "MEMBER_SEQ",   // 시퀀스 컬럼 명
                allocationSize = 1)             // 시퀀스 한 번 호출에 증가하는 수 (최적화에 사용)
public class Member {

    @Id
    @GeneratedValue(strategy = GenerationType.TABLE, generator = "table-generator")
    private Long id;
}
```
- 키 생성 전용 테이블을 하나 만들고 여기에 이름과 값으로 사용할 컬럼을 만들어 DB 시퀀스를 흉내내는 전략이다.
- 테이블을 사용하기 때문에 모든 DB에 적용할 수 있다.
- 영속성 컨텍스트가 PK를 알아야할 때마다 시퀀스 테이블로부터 PK를 조회하고, next_val 컬럼에 UPDATE 쿼리를 보내서 값을 증가시킨다.
- SEQUENCE 전략보다 DB와 한번 더 통신하기 때문에, `allocationSize` 속성을 통한 최적화가 필요하다.

### 5.4 권장하는 식별자 선택 전략
- 기본키는 세가지 조건을 만족해야한다.
    1. null 값을 허용하지 않는다 
    2. 유일해야 한다
    3. 변해선 안 된다.
- 위 모든 사항을 만족시키기 위해서는 대리 키를 사용해야 한다. 
- ex) Long형 + 대체키 + 키 생성전략 형태

## 6. 필드와 컬럼 매핑
```java
@Entity
public class Member {

    @Id
    private Long id;
    
    @Column(name = "name", nullable = false, length = 20)
    private String userName;
    
    @Enumerated(EnumType.STRING)
    private Role role;
    
    private LocalDateTime createTime;

    @Lob
    private String description;
    
    @Transient
    private Long except;
}
```
- @Column: 객체 필드를 테이블 컬럼에 매핑한다. name, nullable을 주로 사용한다.
- @Enumerated : Enum 타입을 매핑할 때 사용한다.
    - EnumType.ORDINAL : enum 순서를 DB에 저장한다. 변경에 취약하기 때문에 사용하지 않는다.
    - EnumType.STRING : enum 이름을 DB에 저장한다.
- @Temporal : 날짜 타입을 매핑할 때 사용한다.
    - TemporalType.DATE : 날짜, DB date 타입과 매핑
    - TemporalType.TIME : 시간, DB time 타입과 매핑
    - TemporalType.TIMESTAMP : 날짜와 시간, DB timestamp 타입과 매핑
- @Lob : 매핑하는 필드 타입이 문자면 CLOB으로 매핑하고, 나머지는 BLOB으로 매핑한다.
- @Transient : 이 필드는 매핑하지 않으며, 객체에 임시로 어떤 값을 보관하고 싶을 때 사용한다.

---
### Reference
- [자바 ORM 표준 JPA 프로그래밍](https://www.inflearn.com/course/ORM-JPA-Basic)