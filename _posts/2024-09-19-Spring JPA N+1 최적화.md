---
title: "Spring JPA N+1 최적화"
date: 2024-09-19 23:00:00 +09:00
categories: [스프링,JPA]
tags:
  [
    스프링,
    JPA,
    N+1,
    Fetch Join,
    지연로딩
  ]
---

<br>

> Spring data JPA를 사용하면 ORM이 쿼리를 작성해 주기 때문에 편리하게 Java 코드만 작성하면 된다. 하지만, 데이터베이스 테이블과 자바 객체 사이에서 연관관계 매핑을 진행하여 조회할 때 우리가 흔히 사용하는 FetchType.Lazy 옵션은 조회시 N+1개의 쿼리로 진행하는 것을 확인할 수 있다. 이처럼 우리가 원하던 결과가 아닌 N+1개의 쿼리를 어떻게 해결할 수 있는지 학습해보았다. 

<br>


테스트 코드의 대부분은 querydsl을 사용하여 테스트하였습니다.


## 1. FetchType.Lazy로 설정하고 조회시 발생하는 문제

<br>

보통 우리는 아래와 같이 Entity 연관관계 매핑에 fetch = FetchType.Lazy를 사용한다.

```Java
@Entity
@Getter
@NoArgsConstructor
@ToString(of = {"id", "age", "team"})
public class Member {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "id")
    private Long id;

    private String username;
    private int age;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "team_id")
    private Team team;


    public Member(String username, int age, Team team) {
        this.username = username;
        this.age = age;
        this.team = team;
    }
}
```

Member - Team이 다대일 관계로 매핑되어있을 때, 특정 로직에서 member를 전체조회하며 team의 정보까지 같이 받아가야하는 로직에 대해서 생각해 보자.

먼저, Member 리스트를 조회하는 쿼리가 1개 발생한다.
![image1](https://github.com/user-attachments/assets/18eff38a-7b3d-44b0-8341-02870dbf5b40)

다음으로, 각 member 마다 member.getTeam()을 하게되면 지연로딩으로 설정하였기 떄문에 조회 시점에 쿼리가 발생하게 된다. 아래와 같이 2명의 member가 있고 각 member가 서로다른 team에 속해있다면 2개의 쿼리가 추가적으로 더 발생하게 된다.

![image2](https://github.com/user-attachments/assets/5bc77325-2fdd-4b73-9a21-3f86c4f31696)


이처럼, 여러 Entity를 한번에 조회하는 로직에서 지연로딩을 설정하였을 때 N+1개의 쿼리가 발생하는데 이것을 N+1 문제라고 한다.

그럼 지연로딩을 사용하지 않으면 되지 않을까? 

> 한 Entity당 1개 연관관계 매핑에만 즉시로딩을 사용할 수 있고 즉시로딩으로 설정하게 되면 모든 로직에서 필요없는 team이라는 정보를 항상 조회하기 때문에 불필요한 리소스를 계속 사용하게 되는 문제가 있다.
{: .prompt-danger }

그렇다면 어떻게 최적화를 할 수 있을까?

1. fetch join 사용하기 : 미리 member를 조회할 때 member와 관련된 team을 join시켜서 조회

2. dto로 조회하기 : 미리 서비스 로직에서 필요한 데이터를 담고있는 dto를 만들어 두고 조회시 JPQL이나 Querydsl을 이용하여 Entity를 조회하는 것이 아니라 dto로 바로 조회하는 방법


이 글에서는 fetch join을 사용해서 최적화하는 방법에 대해서 자세하게 공부하고자 한다. dto로 사용하는 방법은 1차 캐시를 거치지 않고 바로 조회하기 때문에 이점을 주의해서 사용하면 된다.

<Br>

## 2. Fetch join을 사용하여 최적화

<Br>

먼저 fetch join을 사용할 때에는 매핑이 어떻게 되어있는지가 가장 중요하다.

### 1. ManyToOne 연관관계 매핑 fetch join

<Br>

위 Entity 코드에서 Member - Team의 경우 ManyToOne의 다대일 관계이다. 다대일 매핑의 경우 조회시 fetch join을 사용하면 된다.

```Java
1. JPQL
@Query("SELECT m FROM Member m JOIN FETCH m.team")
List<Member> findMemberWithTeam()


2. Querydsl
JPAQueryFactory queryFactory = new JPAQueryFactory(em);

List<Member> memberList = queryFactory
            .select(member)
            .from(member)
            .join(member.team, team).fetchJoin()
            .fetch();
```
이렇게 사용하게 되면 Member를 조회할 때 지연로딩으로 설정하였더라도 member와 관련된 team에 대한 정보들도 한번에 조회하게 된다.

![image3](https://github.com/user-attachments/assets/ea8b2569-7d26-4d4c-9ee2-cb187d4dfd4f)

/* */ 사이는 jpql이고, 그 뒤에 오는 쿼리는 ORM이 작성한 쿼리이다.

fetch join을 사용하게 되면 실제 쿼리는 join을 이용하여 한번에 조회하게 된다.

> 그럼 한번에 여러 ManyToOne을 fetch join도 가능할까?
{: .prompt-warning }

새로운 Entity Board와 Board Type을 추가하여 테스트해보자.

Board - Member (다대일)<br>
Board - BoardType (다대일)

```Java
@Entity
@NoArgsConstructor
@Getter
public class Board {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "id")
    private Long id;

    private String title;
    private String content;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "member_id")
    private Member member;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "board_type")
    private BoardType boardType;
    
    public Board(String title, String content, Member member, BoardType boardType) {
        this.title = title;
        this.content = content;
        this.member = member;
        this.boardType = boardType;
    }
}
```


이 상황에서 게시글을 전체조회하며 Member의 이름과 BoardType의 이름을 함께 조회하는 로직의 코드를 작성하여 다수의 ManyToOne을 fetch join하는 상황을 테스트해보자.


테스트 코드는 아래와 같이 작성하여 테스트를 진행하였습니다.
```Java
@Test
void testNPlusProblemSolveCase2(){
      em.flush();
      em.clear();
      JPAQueryFactory queryFactory = new JPAQueryFactory(em);

      List<Board> boardList = queryFactory
              .select(board)
              .from(board)
              .join(board.member, member).fetchJoin()
              .join(board.boardType, boardType).fetchJoin()
              .fetch();

      System.out.println("------------------------------------------");
      for(Board b : boardList) {
          System.out.println(b.getBoardType().getName());
          System.out.println(b.getMember().getUsername());
      }
}
```

쿼리 로그를 살펴보면 아래와 같이 나온다.

![image](https://github.com/user-attachments/assets/93c81b54-288c-4807-b89f-954c05f41371)

역시 /* */ 사이의 쿼리는 JPQL이고 그 뒤에 나오는 쿼리가 ORM이 작성한 쿼리이다.

테스트코드로 확인해 보았을때 다수의 ManyToOne fetch join은 가능하다고 판단할 수 있다.

<br>

### 2. OneToMany 연관관계 Collection fetch join

<br>

일대다의 경우 쿼리는 동일하다.


```Java
1. JPQL
@Query("SELECT t FROM Team t JOIN FETCH t.members where t.name = '팀A' ")
List<Team> findTeamWithMembers()


2. Querydsl
JPAQueryFactory queryFactory = new JPAQueryFactory(em);

List<Team> teamList = queryFactory
            .select(team)
            .from(team)
            .join(team.members, member).fetchJoin()
            .fetch();
```

하지만 이렇게 작성한 코드로 실행을 하게되면 전혀 다른 결과가 나온다.
아래 그림을 살펴보자

![image](https://github.com/user-attachments/assets/72111c85-ab17-4b93-9e02-51feede7275a)

팀 A가 존재하고 회원1과 회원2가 팀A에 속할때 두 테이블을 조인시켜 팀A와 관련된 member만 뽑아낸다면 아래와 같은 결과가 나온다.

![image](https://github.com/user-attachments/assets/ed0038a0-bd80-4827-a450-755c1d226358)

즉, 동일한 팀 A가 팀 A를 가진 member의 수만큼 반복되어 나타난다.<br>
이렇게 조인된 결과를 JPA가 매핑하게되면 Team 객체가 2번 중복되어 리스트에 넣어진다. 우리는 하나의 요소만 team 리스트에 들어갈 것을 예상하지만 리스트에 2개가 들어가 있다. 

--> 조회 조건이 넓어질 수록 원하지 않게 중복되게 조회됨

따라서 JPQL의 distinct를 사용하여 애플리케이션 단계에서 리스트에 중복되게 담기는 것을 제거할 수 있다.

최근 hibernate 6부터는 중복제거가 자동으로 되기 때문에 정말 편리하다.


> 역시 여러 개의 일대다 매핑을 fetch join 할 수 있는가?
{: .prompt-warning }

답은 아니요다. 실제 구현해서 테스트 코드로 실행해 보면 아래와 같은 오류 메시지를 볼 수 있다.
![image](https://github.com/user-attachments/assets/0c766d90-4ed9-4e43-9b79-57c5385646c2)

MutipleBagFetchException 문제로 @OneToMany 관계를 단일 쿼리에서 동시에 페치할 수 없음을 확인할 수 있었다.

> Why? 왜 그럴까?
{: .prompt-warning }

여러개의 OneToMany 관계를 가지는 엔티티를 fetch할 때 Hibernate는 SQL 조인문을 사용하여 데이터를 가져온다.

그러나, 이 과정에서 SQL은 여러 OneToMany 관계에 대해 Cartesian Product를 생성하게 된다. (가능한 모든 조합을 생성)

![image](https://github.com/user-attachments/assets/28736d16-e0d7-46b2-b0a2-be52dd898634)

위 그림처럼 회원 1에 게시물이 2개있고 주문이 3개있는 데이터를 가져오는 경우

그림의 오른쪽 표처럼 총 2 * 3의 6개의 데이터로 뻥튀기 된다. 이를 Hibernate는 정합성을 유지하기 어렵다고 판단하여 금지한 것이다.

실제로 데이터를 가져올 때 어떻게 가져오는지 궁금하여 DTO로 테스트를 진행해보았다.

```Java
@BeforeEach
    public void before() {
        Team teamA = new Team("teamA");
        Team teamB = new Team("teamB");
        em.persist(teamA);
        em.persist(teamB);

        Member member1 = new Member("member1", 10, teamA);


        em.persist(member1);


        BoardType type1 = new BoardType("공지사항");
        BoardType type2 = new BoardType("일반");
        BoardType type3 = new BoardType("질문");

        em.persist(type1);
        em.persist(type2);
        em.persist(type3);

        Board board1 = new Board("제목1","내용1",member1,type1);
        Board board2 = new Board("제목2","내용2",member1,type2);
        Board board3 = new Board("제목3","내용3",member1,type3);

        em.persist(board1);
        em.persist(board2);
        em.persist(board3);

        Orders orders1 = new Orders("주문내역1",member1);
        Orders orders2 = new Orders("주문내역2",member1);
        Orders orders3 = new Orders("주문내역3",member1);

        em.persist(orders1);
        em.persist(orders2);
        em.persist(orders3);

    }

    /** 한번에 조회하기 -> 중복될 수 있음 */
    @Test
    void testNPlusProblemSolve(){
        em.flush();
        em.clear();
        JPAQueryFactory queryFactory = new JPAQueryFactory(em);

        List<GetMemberWithOrdersAndBoardsDto> memberList = queryFactory
                .select(Projections.constructor(GetMemberWithOrdersAndBoardsDto.class,
                        member.id,
                        member.username,
                        member.age,
                        board.title,
                        board.content,
                        orders.name))
                .from(member)
                .leftJoin(member.boards, board)   // board와 조인
                .leftJoin(member.orders, orders)   // order와 조인
                .fetch();

        for(GetMemberWithOrdersAndBoardsDto result : memberList){
            System.out.println(result);
        }
    }
```


한번에 하나의 DTO로 조회하게 될 경우 아래와 같이 3개의 게시물에 3개의 주문내역일 경우 9개의 결과가 나오게 된다. -> 데이터 중복 발생

![image](https://github.com/user-attachments/assets/e40dec3c-e934-4fd2-b5ce-2480a5a36150)


서비스 로직에서 만약 2개의 OneToMany 관계의 엔티티를 한번에 조회하게 될 경우 우리는 어떤 선택을 할 수 있을까?

```yml
hibernate:
  default_batch_fetch_size: 10
```

yml 파일에 위 옵션을 주어서 fetch join시 발생하는 N+1문제를 in 쿼리로 조회하게 바꿀 수 있다.

따라서 이와 같은 경우 OneToMany관계 개수만큼 쿼리가 증가하지만 N개의 요소만큼 증가하지 않아 성능이 좋아진다고 볼 수 있다.

<br>

```text
쿼리의 개수 변동 사항을 파악해 보면

1. Member 조회 쿼리 1개
2. member 마다 board 조회 쿼리 N개
3. member 마다 Order 조회 쿼리 N개

로 2N + 1개가 발생하는데 

default_batch_fetch_size: 10 옵션을 통해 

3개의 쿼리로 조회할 수 있다.
```

2n+1  --> 1 + m

n : 데이터 개수에 비례

m : OneToMany 개수에 비례

<br>

### 3. OneToMany  + ManyToOne Fetch Join

<br>

마지막으로 이 두개를 한번에 조회하는 경우 어떻게 될까? Many가 하나라 작동은 될것이라고 예상할 수 있다.

테스트 코드를 통해 시도해보았다.


```java
 @Test
   
    void testNPlusProblemSolve(){
        em.flush();
        em.clear();
        JPAQueryFactory queryFactory = new JPAQueryFactory(em);

        List<Member> memberList = queryFactory
                .select(member)
                .from(member)
                .join(member.team, team).fetchJoin()
                .join(member.boards, board).fetchJoin()
                .fetch();

        System.out.println("------------------------------------------");
        for(Member member1 : memberList) {
            System.out.println(member1.getBoards());
            System.out.println(member1.getTeam().getName());
        }
    }
```

```sql
select
            m1_0.id,
            m1_0.age,
            b1_0.member_id,
            b1_0.id,
            b1_0.board_type,
            b1_0.content,
            b1_0.title,
            t1_0.id,
            t1_0.name,
            m1_0.username 
        from
            member m1_0 
        join
            team t1_0 
                on t1_0.id=m1_0.team_id 
        join
            board b1_0 
                on m1_0.id=b1_0.member_id
```

그 결과 위와 같은 쿼리가 날라갔고 fetch join이 잘 작동함을 확인해 볼 수 있다.


<br>

## 3. 결론

<br>

Fetch 타입이 지연로딩일 경우 서비스 로직에서 특정 엔티티 리스트를 조회하고 그 엔티티와 연관된 엔티티를 개별로 조회할 필요가 있을 때 우리는 fetch 조인 사용을 고려하면 된다.

방법에는 fetch join 이외에도 dto조회, batch size 설정등이 있는데 해결 순서도를 대충 적어본다면

1. Fetch Join으로 성능을 최적화한다 -> 대부분의 N+1문제가 해결됨
2. Fetch Join으로 안된다면 DTO로 직접 조회를 사용한다.
3. OneToMany의 경우 default batch size 옵션을 사용하자
4. 최후의 방법은 JPA가 제공하는 JDBC Template을 사용해서 직접 sql을 사용하여 조회하는 것이다.

