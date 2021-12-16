
# 스프링 부트와 JPA 활용

### 기능 목록
- 회원 기능
  - 회원 등록
  - 회원 조회
- 상품 기능
  - 상품 등록
  - 상품 수정
  - 상품 조회
- 주문 기능
  - 상품 주문
  - 주문 내역 조회
  - 주문 취소
- 기타 요구사항
  - 상품은 재고 관리가 필요하다.
  - 상품의 종류는 도서, 음반, 영화가 있다.
  - 상품을 카테고리로 구분할 수 있다.
  - 상품 주문시 배송 정보를 입력할 수 있다.
  
### 회원 테이블 분석
![ERD](https://user-images.githubusercontent.com/62706198/146130131-8e78b579-537d-49c0-bfe0-d6e83704b5a5.PNG)


### 연관관계 매핑 분석

- 회원과 주문: 일대다 , 다대일의 양방향 관계다. 따라서 연관관계의 주인을 정해야 하는데, 
외래 키가 있는 주문을 연관관계의 주인으로 정하는 것이 좋다. 
그러므로 Order.member를 ORDERS.MEMBER_ID 외래 키와 매핑한다.

- 주문상품과 주문: 다대일 양방향 관계다. 외래 키가 주문상품에 있으므로 주문상품이 연관관계의 주인이다.
그러므로 OrderItem.order 를 ORDER_ITEM.ORDER_ID 외래 키와 매핑한다.
- 주문상품과 상품: 다대일 단방향 관계다. OrderItem.item 을 ORDER_ITEM.ITEM_ID 외래 키와 매핑한다.
- 주문과 배송: 일대일 양방향 관계다. Order.delivery 를 ORDERS.DELIVERY_ID 외래 키와 매핑한다.
- 카테고리와 상품: @ManyToMany 를 사용해서 매핑한다.(실무에서 @ManyToMany는 사용하지 말자. 
여기서는 다대다 관계를 예제로 보여주기 위해 추가했을 뿐이다)

### 엔티티 설계시 주의점
1. 엔티티에서는 가급적 Setter를 사용하지 말자
    - Setter가 모두 열려있다면 변경 포인트가 너무 많아서, 유지보수가 어렵다. 나중에 리펙토링으로 Setter를 제거하자
2. 모든 연관관계는 지연로딩으로 설정
    - 즉시로딩(EAGER)은 예측이 어렵고, 어떤 SQL이 실행될지 추측하기 어렵다. 특히 JPQL을 실행할 때 N+1 문제가 자주 발생한다.
    - 실무에서 모든 연관관계는 지연로딩(LAZY)으로 설정해야 한다.
    - 연관된 엔티티를 함께 DB에서 조회해야 하면, fetch join 또는 엔티티 그래프 기능을 사용한다.
    - @XtoOne(OneToOne, ManyToOne)관계는 기본이 즉시로딩이므로 직접 지연로딩으로 설정해야 한다.
3. 컬렉션은 필드에서 초기화 하자.
    - 컬렉션은 필드에서 바로 초기화하는 것이 안전하다.
    - null문제에서 안전하다.
    - 하이버네이트는 엔티티를 영속화 할 때, 컬랙션을 감싸서 하이버네이트가 제공하는 내장 컬렉션으로 변경한다. 
만약 getOrders() 처럼 임의의 메서드에서 컬력션을 잘못 생성하면 하이버네이트 내부 메커니즘에 문제가 발생할 수 있다. 따라서 필드레벨에서 생성하는 것이 가장 안전하고, 코드도 간결하다.

### 어플리케이션 아키텍쳐
![아키텍쳐](https://user-images.githubusercontent.com/62706198/146131284-2693b483-6cb8-450c-9891-5493681df786.PNG)
- 계층형 구조 사용
  - controler, web: 웹 계층
  - service: 비즈니스 로직, 트랜잭션 처리
  - repository: JPA를 직접 사용하는 계층, 엔티티 매니저 사용
  - domain: 엔티티가 모여 있는 계층, 모든 계층에서 사용
- 개발 순서: 서비스, 리포지토리 계층을 개발하고, 테스트 케이스를 작성해서 검증, 마지막에 웹 계층에 적용

### 어노테이션
- @Repository: 스프링 빈으로 등록, JPA 예외를 스프링 기반 예외로 예외 변환
- @PersistenceContext: 엔티티 메니저(EntityManager) 주입
- @PersistenceUnit: 엔티티 매니저 팩토리(EntityManagerFactory) 주입
- @Transactional: 트랜잭션, 영속성 컨텍스트
  - readonly=true: 데이터의 변경이 없는 읽기 전용 메서드에 사용, 영속성 컨텍스트를 플러시 하지 않으므로 약간의 성능 향상(읽기 전용에는 다 적용)
  - 데이터베이스 드라이버가 지원하면 DB에서 성능 향상
- @Autowired: 생성자 Injection 많이 사용, 생성자가 하나면 생략 가능
- @RunWith(SpringRunner.class): 스프링과 테스트 통합
- @SpringBootTest: 스프링 부트 띄우고 테스트(이게 없으면 @Autowired 실패)
- @Transactional: 반복 가능한 테스트 지원, 각각의 테스트를 실행할 때마다 트랜잭션을 시작하고 테스트가 끝나면 트랜잭션을 강제로 롤백

### 도메인 모델 패턴
- 주문 서비스의 주문과 주문 취소 메서드를 보면 비즈니스 로직 대부분이 엔티티에 있다. 서비스계층은
단순히 엔티티에 필요한 요청을 위임하는 역할을 한다. 이처럼 엔티티가 비즈니스 로직을 가지고 객체 지향의 특성을 적극 활용하는 것을 도메인 모델 패턴이라 한다.
- 반대로 엔티티에는 비즈니스 로직이 거의 없고 서비스 계층에서 대부분의 비즈니스 로직을 처리하는 것을 트랜잭션 스크립트 패턴이라 한다.