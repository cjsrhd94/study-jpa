# 영속성 관리


## 1. 엔티티 매니저 팩토리와 엔티티 매니저
```java
EntityManagerFactory emf = Persistence.createEntityManagerFactory("name");
EntityManager em = emf.createEntityManager();
```

- JPA는 요청이 들어올 때 마다 엔티티 매니저 팩토리에서 엔티티 매니저를 생성한다.
- 엔티티 매니저 팩토리는 여러 스레드가 동시에 접근해도 안전하므로, 서로 다른 스레드 간에 공유해도 된다.
- 반면 엔티티 매니저는 여러 스레드가 동시에 접근하면 동시성 문제가 발생하기 때문에 스레드 간에 절대 공유하면 안된다.
- 엔티티 매니저는 데이터 베이스 연결이 필요한 시점에 DB 커넥션을 얻는다.


## 2. 영속성 컨텍스트
- 영속성 컨텍스트는 "엔티티를 영구 저장하는 환경"이라는 뜻이다.
- `em.persist()` 메서드가 실행되는 시점에는 실제 DB에 저장하는 것이 아니라, 영속성 컨텍스트에 저장하여 엔티티를 영속화 한다.
- 영속성 컨텍스트는 논리적인 개념으로, 엔티티 매니저를 통해 영속성 컨텍스트에 접근할 수 있다.


## 3. 엔티티의 생명주기
- 비영속(new/transient)
    - 순수한 객체 상태로, 영속성 컨텍스트나 DB와는 전혀 관련이 없는 상태.
    
- 영속(managed)
    - 엔티티가 영속성 컨텍스트에 의해 관리 되는 상태.
    - em.persist() 이외에 em.find()나 JPQL을 사용해서 조회한 엔티티도 영속 상태이다.
    
- 준영속(detached)
    - 영속상태의 엔티티를 영속성 컨텍스트가 관리하지 않는 상태.
    
- 삭제(removed)
    - 엔티티를 영속성 컨텍스트와 DB에서 삭제하는 것.
    

## 4. 영속성 컨텍스트의 특징
- 영속 상태는 식별자 값`(@Id)`이 반드시 있어야 한다. 영속성 컨텍스트가 엔티티를 식별자 값으로 구분하기 때문이다.
- JPA는 트랜잭션을 커밋하는 순간 영속성 컨텍스트에 새로 저장된 엔티티를 DB에 반영을 하는데 이를 플러시(flush)라고 한다.
- 영속성 컨텍스트가 엔티티를 관리하면 다섯 가지 장점이 있다. (1차 캐시, 동일성 보장, 트랙잭션을 지원하는 쓰기 지연, 변경 감지, 지연 로딩)

### 4.1 1차 캐시
- 1차 캐시는 영속성 컨텍스트 내부에 있는 캐시를 뜻한다.
- 영속성 컨텍스트에 엔티티를 저장할 때, 1차 캐시에 map으로 저장된다.
- key에는 `@Id로 선언한 필드 값`이, value에는 `해당 엔티티 자체`가 저장된다.

- `em.find()` 메서드가 실행되면, 우선 엔티티 매니저 내부 1차 캐시에서 엔티티를 조회한다.
- 1차 캐시에 찾는 엔티티가 없을 시, DB를 조회하여 엔티티를 생성하고, 1차 캐시에 저장한 후 영속상태의 엔티티를 반환한다.

### 4.2 동일성 보장
- `em.find()`를 반복하여 호출해도 영속성 컨텍스트는 1차 캐시에 있는 같은 엔티티 인스턴스를 반환한다. 따라서 동일성이 보장된다.
- JPA는 1차 캐시로 반복 가능한 읽기(REPEATABLE READ) 등급의 트랜잭션 격리 수준을 DB가 아닌 애플리케이션 차원에서 제공한다.

### 4.3 트랜잭션을 지원하는 쓰기 지연
- 트랜잭션 내부에서 `em.persist()`가 일어날 때, 엔티티들을 1차 캐시에 저장하고, INSERT 쿼리를 생성하여 내부 쿼리 저장소에 쌓아 놓는다.
- 그리고 트랜잭션을 커밋하는 시점에 플러시를 통해 영속성 컨텍스트의 변경내용을 DB에 동기화 시킨뒤  실제 DB 트랙잭션을 커밋한다.
- JDBC 일괄 처리 옵션으로 INSERT 쿼리를 일정 크기 만큼 모아서 한번에 처리할 수 있다.

  ```xml
  <!-- persistence.xml -->
  <property name="hibernate.jdbc.batch_size" value=10/>
  ```

### 4.4 변경 감지
- 영속 상태인 엔티티의 변경사항을 DB에 자동으로 반영하는 기능을 변경 감지라 한다.
- JPA는 엔티티를 영속성 컨텍스트에 보관할 때, 최초 상태를 복사해서 저장해두는데 이것을 스냅샷이라 한다.
- 플러시 시점에 스냅샷과 엔티티를 비교해서 변경된 엔티티를 찾아 UPDATE 쿼리를 생성하고 쓰기 지연 SQL 저장소에 보낸다.
- 따라서 영속상태의 엔티티를 가져와서 값만 바꾸면 수정된다.


## 5. 플러시
- 플러시는 영속성 컨텍스트의 변경 내용을 DB에 반영한다.
- 플러시를 실행하면 아래와 같은 일이 일어난다.
    1. 변경 감지가 동작해서 영속성 컨텍스트에 있는 모든 엔티티를 스냅샷과 비교해서 수정된 엔티티를 찾는다.
    2. 수정된 엔티티는 수정 쿼리를 만들어 쓰기 지연 SQL 저장소에 등록한다.
    3. 쓰기 지연 SQL 저장소의 쿼리를 DB에 전송한다. (등록, 수정, 삭제 쿼리)

### 5.1 영속성 컨텍스트를 플러시 하는 방법
- `em.flush()`로 직접호출
- 트랜잭션 커밋 시 플러시 자동 호출
    - 트랜잭션만 커밋하면 어떤 데이터도 DB에 반영되지 않는다.
    - JPA는 트랜잭션을 커밋할 때 플러시를 자동으로 호출하여 영속성 컨텐스트의 변경내용을 DB에 반영한다.
- JPQL 쿼리 실행시 플러시 자동 호출
    - 영속화한 상태의 객체에 대해 JPQL로 SELECT 쿼리를 날리면 DB에 저장되어 있는 값이 결과가 조회되지 않는다.
    - JPA는 이런 문제를 예방하기 위해 JPQL을 실행할 때도 플러시를 자동으로 호출한다.
    
### 5.2 플러시 모드 옵션
- FlushModeType.AUTO : 커밋이나 쿼리를 실행할 때 플러시. (기본값)
- FlushModeType.COMMIT: 커밋 할때만 플러시.


## 6. 준영속 상태
- 영속 상태의 엔티티가 영속성 컨텍스트에서 분리된 상태로, 준영속 상태의 엔티티는 영속성 컨텍스트가 제공하는 기능을 사용할 수 없다.

### 6.1 준영속 상태로 전환하는 방법
- em.detach() : 특정 엔티티를 준영속 상태로 만든다.
- em.clear() : 해당 영속성 컨텍스트의 모든 엔티티를 준영속 상태로 만든다.
- em.close() : 영속성 컨텍스트를 종료하면 해당 영속성 컨텍스트가 관리하던 영속 상태의 엔티티가 모두 준영속 상태가 된다.

### 6.2 준영속 상태의 특징
- 거의 비영속 상태에 가깝다.
- 한 번 영속 상태였으므로 반드시 식별자 값을 가지고 있다.
- 지연 로딩을 할 수 없다.

---
### Reference
- [자바 ORM 표준 JPA 프로그래밍](https://www.inflearn.com/course/ORM-JPA-Basic)