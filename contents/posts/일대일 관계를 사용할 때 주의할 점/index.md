---
title: "일대일 관계를 사용할 때 주의할 점"
date: 2023-03-24 16:50:00
update: 2022-03-24
tags:
- JPA
- OneToOne
- 일대일 관계
- N+1
---

JPA에서는 일대일 연관관계 매핑을 `@OneToOne` 을 사용하여 맺을 수 있습니다. 간단한 예제로 확인해보겠습니다.

```java
@Entity
@Getter
@Setter
public class Member {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "id", nullable = false)
    private Long id;

    @OneToOne(mappedBy = "member", cascade = CascadeType.ALL, orphanRemoval = true, fetch = FetchType.LAZY)
    @JoinColumn(unique = true)
    private Locker locker;
}

@Entity
public class Locker {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "id", nullable = false)
    private Long id;

    @OneToOne(fetch = FetchType.LAZY)
    private Member member;
}
```

`Member` 와 `Locker` 를 일대일 관계를 맺고 연관관계 주인을 Locker가 가지도록 하였습니다.

## 연관관계 주인
객체의 양방향 관계는 서로 다른 단방향 관계 2개로 이루어져 있습니다.JPA는 어떤 객체를 기준으로 외래 키를 관리해야 할까요? 

> Member.locker.id? Locker.member.id?

여기서 연관관계 주인을 설정해주어야 합니다. 연관관계 주인이란 테이블의 외래 키를 관리하는 객체를 의미합니다. 연관관계 주인이 아닌 곳에서는 조회만 가능하게 됩니다.

연관관계 주인은 `mappedBy`를 사용하여 설정할 수 있습니다. 위 코드를 보면 `Member.Locker` 위에 `mappedBy = "member"`가 있는 것을 확인할 수 있습니다. 이 뜻은 `Locker의 "member" 변수가 연관관계 주인이라는 뜻으로 `Member.Locker` 에서는 외래키의 값에 대해 조회만 가능하다는 것 입니다.

```java
        // given
        Member member = save(new Member("name1")); // 영속화 및 저장

        // when
        Locker locker = save(new Locker()); // 영속화 및 저장
        // member.setLocker(locker); update query x
        // locker.setMember(member); update query o
```
실제 setter를 통해 객체를 할당하면 연관관계 주인인 필드를 가진 `Locker` 에서는 update 쿼리가 날라가고 `Member` 에서는 발생하지 않는 것을 확인할 수 있습니다.

### 일대일 관계에서는 스키마가 바뀝니다.
일대일 관계에서는 연관관계 주인에 따라 외래키의 위치가 바뀝니다. 위 같은 경우에는 `Locker` 가 외래키에 대한 연관관계 주인을 가지고 있기 때문에 `Locker` 테이블에 `account_id` 외래키를 가지고 있습니다.

## 연관관계 주인이 없는 Member 조회할 때 JPA는 어떻게 할까? (지연로딩)
```java
public class Member {
    @OneToOne(mappedBy = "member", cascade = CascadeType.ALL, orphanRemoval = true, fetch = FetchType.LAZY)
    @JoinColumn(unique = true)
    private Locker locker;
}
```
`fetch = FetchType.LAZY` 로 설정되어 있으니 지연로딩으로 조회할 줄 알았지만, 실제 조회했을 때 2번의 쿼리가 발생합니다.
```sql
select * from member;
select * from locker where member_id = ?;
```
왜냐하면 외래키가 필드에 존재하지 않기 때문입니다.
### 프록시 객체에 한계
JPA는 지연로딩을 위한 프록시를 할당해주어야 하는데 이 때 Null 값을 프록시로 만들지 못합니다. 외래키가 존재하지 않는 `Member` 에서는 null을 할당할 수 없으니 어쩔 수 없이 member를 조회하고 member.id를 사용하여 `Locker` 를 조회하게 됩니다.
```java
void findWithLock() {
    // given
    save(new Member("name1"));
    save(new Member("name2"));
    save(new Member("name3"));
    em.clear();

    // when
    List<Member> members = em.createQuery("select m from Member m", Member.class).getResultList();

    // then
    assertThat(members).hasSize(3);
}
```
```sql
select * from member;

// n+1 발생
select * from locker where member_id = ?;
select * from locker where member_id = ?;
select * from locker where member_id = ?;
```
3개의 Member를 조회하기 위해 3번의 쿼리가 추가로 발생하게 됩니다. 지연로딩으로 설정해도 즉시로딩처럼 실행되는 것을 확인할 수 있습니다.

## 일대일 관계에서 N+1이 발생하는 경우

### 연관관계 주인이 아닌 엔티티에서 조회할 때
앞에서 이야기했던 내용입니다. 단, 한 가지 경우에 예외가 있습니다.
- PK(id)로 조회하고 `FetchType.EAGER`로 설정한 경우

```java
@Entity
@Getter
@Setter
public class Member {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "id", nullable = false)
    private Long id;

    @OneToOne(mappedBy = "member", cascade = CascadeType.ALL, orphanRemoval = true, fetch = FetchType.EAGER)
    @JoinColumn(unique = true)
    private Locker locker;
}

---

Member member = memberRepository.findById(1L); // fetch join 사용
```
이 땐 JPA 내에서 최적화하여 fetch join을 사용할 수 있습니다. 이 외에 `findbyName` 같이 필드값을 기준으로 조회하게 될 경우
`FetchType.EAGER`로 설정해도 2번의 쿼리를 실행하게 됩니다.

> JPA 입장에서 id를 모르기 때문에 어쩔 수 없는 부분이라고 생각합니다.

### 자주 사용하는 엔티티를 연관관계 주인으로 설정
이 방법은 근본적인 해결이 아닙니다. 하지만, 일대일 관계에서는 위 같은 문제가 있기 때문에 성능 개선을 위해
애플리케이션에서 자주 사용하는 조회를 파악하고 연관관계 주인 즉 fk를 어느 테이블에 놓을지 고민해보는 것도 좋은 것 같습니다.

### EntityGraph
n+1이 발생하는 JPQL 에 `EntityGraph`를 사용합니다. 해당 조회 쿼리에서는 fetch join을 사용할 수 있게 됩니다.

```java
@NamedEntityGraphs({
        @NamedEntityGraph(
                name = "graph.Account.consultantInfo",
                attributeNodes = {@NamedAttributeNode(value = "consultantInfo"),@NamedAttributeNode(value = "organization")})
        })
public class Account extends BaseDomainWithId implements DomainWithMapper {
 // ...
}
```

```java
@Repository
public interface Account extends BaseRepository<Product> {

    @EntityGraph(attributePaths = {"consultantInfo"})
    List<Account> findAllByName(String name);
}
```

2가지 방법이 있습니다. 첫 번째 방법은 `Criteria`, `QueryDSL` 에서 조회 시 `setHint` 를 사용할 때 설정하고 사용하게 됩니다.
두 번째 방법은 Spring Data JPA에서 어노테이션을 붙여 fetch join을 사용할 수 있는 방법입니다.
그리고 `@NamedEntityGraphs`에 작성된 패치 전략을 `@EntityGraph`에 적용할 수 있습니다.

```java
@Repository
public interface Account extends BaseRepository<Product> {

    @EntityGraph(value = "graph.Account.consultantInfo")
    List<Account> findAllByName(String name);
}
```
