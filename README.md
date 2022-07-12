# Study-JPA
인터넷 강의를 통해 Spring MVC, ORM(JPA) 학습
- JPA Query, Test Code를 통한 로직 검증

## 1. JPA 쿼리  
 - EntityManager를 통한 쿼리
 ```
private final EntityManager em;

public void save(Member member) {
    em.persist(member);
}

public List<Member> findByName(String name) {
    return em.createQuery("select m from Member m where m.name = :name", Member.class)
          .setParameter("name", name)
          .getResultList();
}
```

- Jpql을 통한 쿼리 ( 동적 쿼리 생성이 불편하다 )
```
String jpql = "select o from Order o join o.member m";
boolean isFirstCondition = true;

//주문 상태 검색
if (orderSearch.getOrderStatus() != null) {
    if (isFirstCondition) {
      jpql += " where";
      isFirstCondition = false;
      } else {
      jpql += " and";
    }
      jpql += " o.status = :status";
      ...
}
```

- Criteria를 통한 쿼리 ( 컴파일 단계에서 쿼리 오류 체크가 가능해 동적 쿼리를 안전하게 생성할 수 있다. )
```
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<Order> cq = cb.createQuery(Order.class);
Root<Order> o = cq.from(Order.class);
...
if (orderSearch.getOrderStatus() != null) {
    Predicate status = cb.equal(o.get("status"), orderSearch.getOrderStatus());
    criteria.add(status);
}

if (StringUtils.hasText(orderSearch.getMemberName())) {
    Predicate name = cb.like(m.get("name"), "%" + orderSearch.getMemberName() + "%");
    criteria.add(name);
}

...

cq.where(criteria.toArray(new Predicate[criteria.size()]));
TypedQuery<Order> query = em.createQuery(cq).setMaxResults(1000);
return query.getResultList();               
 ```

## 2. Test Code를 통한 로직 검증
```
@Test
public void 상품주문() throws Exception{
    //given
    Member member = createMember();
    Book book = createBook("시골 JPA", 10000, 10);

    int orderCount = 2;

    //when
    Long orderId = orderService.order(member.getId(), book.getId(), orderCount);
    
    //then
    Order getOrder = orderRepository.findOne(orderId);

    assertEquals("상품 주문시 상태는 ORDER", OrderStatus.ORDER, getOrder.getStatus());
    assertEquals("주문한 상품 종류 수가 정확해야 한다.", 1, getOrder.getOrderItems().size());
    assertEquals("주문 가격은 가격 * 수량이다.", 10000 * orderCount, getOrder.getTotalPrice());
    assertEquals("주문 수량만큼 재고가 줄어야 한다.", 8, book.getStockQuantity());
}

@Test(expected = NotEnoughStockException.class)
public void 상품주문_재고수량초과() throws Exception{
  //given
  Member member = createMember();
  Item item = createBook("시골 JPA", 10000, 10);

  int orderCount = 11;

  //when
  orderService.order(member.getId(), item.getId(), orderCount);

  //then
  fail("재고 수량 부족 예외가 발생해야한다.");
}
...
```
