> #### 지금부터 아래의 글에는 모두 @BeaforEach 를 사용하여 다음의 값을 넣어 줄 것입니다.
**member** : member1 / member2 / member3 / member4
**team** : teamA / teamB가 생성될 것입니다.


* Member와 Team은 N:1 연관관계
```java
  @BeforeEach
    public void before() {
        jpaQueryFactory = new JPAQueryFactory(em);
        Team teamA = new Team("teamA");
        Team teamB = new Team("teamB");
        em.persist(teamA);
        em.persist(teamB);
        Member member1 = new Member("member1", 10, teamA);
        Member member2 = new Member("member2", 20, teamA);
        Member member3 = new Member("member3", 30, teamB);
        Member member4 = new Member("member4", 40, teamB);
        em.persist(member1);
        em.persist(member2);
        em.persist(member3);
        em.persist(member4);
    }
```

## QueryDsl 기본 조인
```java
join(조인 대상, 별칭으로 사용할 Q타입)
```
<span style='background-color: #fff5b1'>join의 기본 문법은 join 될 대상, 그리고 알리아스로 사용할 Q타입의 별칭을 매개변수에 넣어줍니다.</span>
#### join example 1

* 예시를 들어서 팀의 이름과 각 팀의 평균연령을 구하는 코드를 QueryDsl로 구현해보겠습니다.
```java
    @Test
    @DisplayName("join해서 집합함수 사용하기")
    void group() throws Exception {

       List<Tuple> result = jpaQueryFactory
               .select(team.name, member.age.avg())
               .from(member)
               .join(member.team, team)
               .groupBy(team.name)
               .fetch();

       Tuple teamA = result.get(0);
       Tuple teamB = result.get(1);
       assertThat(teamA.get(team.name)).isEqualTo("teamA");
       assertThat(teamA.get(member.age.avg())).isEqualTo(15);
       assertThat(teamB.get(team.name)).isEqualTo("teamB");
       assertThat(teamB.get(member.age.avg())).isEqualTo(35);

    }
```
> #### 코드설명
1. select()안에 **team.name**과 **member.age.avg()** 를 사용해주어서 각각 team의 이름과 member회원의 나이의 평균을 구할 것 입니다.
2. join() 문법을 사용하여 member 테이블과 team테이블을 join시킬 것 입니다. join(조인 대상, 별칭으로 사용할 Q타입)
3. groupBy() 문법을 사용하여 team의 이름 별로 결과를 팀의 이름별로 집계할 것입니다.
4. **fetch() 문법**은  **리스트로 조회하고, 데이터가 없으면 빈 리스트 반환**시키는 문법입니다.

* 코드 실행 시 쿼리문
```sql
 select
        team1_.name as col_0_0_,
        avg(cast(member0_.age as double)) as col_1_0_ 
    from
        member member0_ 
    inner join
        team team1_ 
            on member0_.team_id=team1_.team_id 
    group by
        team1_.name
```
#### join example 2
* 다음 문제는 teamA에 속한 member의 결과를 얻어오는 QueryDsl 문법입니다.

```java
    @Test
    @DisplayName("join test")
    void join() throws Exception {

        List<Member> result = jpaQueryFactory.selectFrom(member)
                .join(member.team, team)
                .where(team.name.eq("teamA"))
                .fetch();

        assertThat(result)
                .extracting("username")
                .containsExactly("member1", "member2");

    }
```
> #### 코드설명
1.select, from은 같은 경우 selectFrom으로 합쳐서 사용할 수 있습니다. 필드를 몇개씩이나 조회할 때는 사용 X
2.join() 문법을 사용하여 member 테이블과 team테이블을 join시킬 것 입니다. join(조인 대상, 별칭으로 사용할 Q타입)
3.where() QueryDsl문법을 사용하여 eq() : equals(==)의 문법을 사용하여 teamA에 속사는 member를 조회합니다.

* 코드 실행 시 쿼리문
```sql
 select
        member0_.member_id as member_i1_0_,
        member0_.age as age2_0_,
        member0_.team_id as team_id4_0_,
        member0_.username as username3_0_ 
    from
        member member0_ 
    inner join
        team team1_ 
            on member0_.team_id=team1_.team_id 
    where
        team1_.name=?
```

## QueryDsl theta 조인
* <span style='background-color: #fff5b1'>연관 관계가 없는 필드로 조인하는 것</span>
* 세타 조인은 <span style='background-color: #fff5b1'>from절에 테이블을 나열</span>한다. (연관관계로의 조인이 아니기에) / 일반조인은 객체의 참조값 형식으로 join
* 조인하고자 하는 테이블의 모든 행을 결합하여, 두 테이블의 행 갯수를 곱한 만큼의 결과를 반환한다. (Cartesian Product: 곱집합)
* 실제 sql은 **cross join**이 발생한다. (디비가 성능 최적화를 진행해줍니다)
* 외부 조인은 불가능 하나, JPQL의 on절을 통해 외부 조인이 가능하다.

```java
/**
 * 세타 조인(연관관계가 없는 필드로 조인)
 * 회원의 이름이 팀 이름과 같은 회원 조회
 */
@Test
    @DisplayName("theta 조인")
    public void theta_join() throws Exception {
        em.persist(new Member("teamA"));
        em.persist(new Member("teamB"));

        List<Member> result = jpaQueryFactory
                .select(member)
                .from(member, team)
                .where(member.username.eq(team.name))
                .fetch();

        assertThat(result)
                .extracting("username")
                .containsExactly("teamA", "teamB");
    }
}
```
* 쿼리문을 보면 cross join이 발생 🔽
```sql
 select
        member0_.member_id as member_i1_0_,
        member0_.age as age2_0_,
        member0_.team_id as team_id4_0_,
        member0_.username as username3_0_ 
    from
        member member0_ 
        cross join 
        team team1_ 
    where
        member0_.username=team1_.name
```

> ## 내부조인과 외부조인의 차이점
**내부 조인**에서 두 테이블에 공통되지 않는 속성이 있으면 **아무 것도 반환하지 않습니다.** 반면 **외부 조인**의 경우 속성이 비어 있으면 **NULL을 반환**합니다.
![](https://velog.velcdn.com/images/guns95/post/d21b8fae-347f-4c10-9fed-132f06ccb7ee/image.png)


### On절을 사용한 외부조인

>LEFT OUTER JOIN : 조인문의 왼쪽에 있는 테이블의 **모든 결과를 가져 온 후** 오른쪽 테이블의 데이터를 매칭하고, **매칭되는 데이터가 없는 경우 NULL**로 표시합니다.

* 팀 이름이 teamA인 팀만 조회. 회원은 모두 조회(회원과 팀을 조인)

```java
@Test
    @DisplayName("on절 조인인")
   void join_on_filtering() throws Exception {

        List<Tuple> result = jpaQueryFactory.select(member, team)
                .from(member)
                .leftJoin(member.team, team)
                .on(team.name.eq("teamA"))
                .fetch();

        for (Tuple tuple : result) {
            System.out.println("tuple = " + tuple);
        }
    }
```

> member(왼쪽 테이블)의 값은 모두 가져오면서 team엔티티와 조인하여서 team객체에서 "teamA"인 팀만 조회하는 QueryDsl 문법. 매칭되는 데이터가 없는 경우에는 null로 가져옵니다.-> 외부조인의 결과 

* 실행된 쿼리문🔽
* 맨 마지막에 and(team.name = 값)의 조건이 추가 되있는 것을 볼 수 있습니다.
```sql
   select
        member0_.member_id as member_i1_0_0_,
        team1_.team_id as team_id1_1_1_,
        member0_.age as age2_0_0_,
        member0_.team_id as team_id4_0_0_,
        member0_.username as username3_0_0_,
        team1_.name as name2_1_1_ 
    from
        member member0_ 
    left outer join
        team team1_ 
            on member0_.team_id=team1_.team_id 
            and (
                team1_.name=?
            )
```
* 추출된 데이터 값
* LEFT OUTER JOIN의 속성으로 해당 조건에 맞지 않으면 Null값을 반환 합니다.
```java
tuple = [Member(id=3, username=member1, age=10), Team(id=1, name=teamA, members=[Member(id=3, username=member1, age=10), Member(id=4, username=member2, age=20)])]
tuple = [Member(id=4, username=member2, age=20), Team(id=1, name=teamA, members=[Member(id=3, username=member1, age=10), Member(id=4, username=member2, age=20)])]
tuple = [Member(id=5, username=member3, age=30), null]
tuple = [Member(id=6, username=member4, age=40), null]
```

외부조인과 내부조인을 확실하게 비교하고 둘의 차이점을 알기위해서 다음은 위의 조건을 똑같이 하되 내부조인으로 QueryDsl을 진행해보겠습니다.

```java
 @Test
    @DisplayName("내부조인과 외부조인의 차이")
    void innerJoin_and_outerJoin() throws Exception {

        List<Tuple> result = jpaQueryFactory.select(member, team)
                .from(member)
                .join(member.team, team) // <- 내부 조인 구간 
                .on(team.name.eq("teamA"))
                .fetch();

        for (Tuple tuple : result) {
            System.out.println("tuple = " + tuple);
        }

    }
```
내부 조인과 외부 조인의 차이점을 아까 설명한 글이 있습니다.
다시 한번 말하지만, **내부 조인**에서 두 테이블에 공통되지 않는 속성이 있으면 **아무 것도 반환하지 않습니다.** 반면 **외부 조인**의 경우 속성이 비어 있으면 **NULL을 반환**이였습니다. 방금 실행한 코드는 내부 조인이므로 다음과 같은 데이터 조회가 이루어집니다.🔽
```java
tuple = [Member(id=3, username=member1, age=10), Team(id=1, name=teamA, members=[Member(id=3, username=member1, age=10), Member(id=4, username=member2, age=20)])]
tuple = [Member(id=4, username=member2, age=20), Team(id=1, name=teamA, members=[Member(id=3, username=member1, age=10), Member(id=4, username=member2, age=20)])]
```
> on 절을 활용해 조인 대상을 필터링 할 때, 외부조인이 아니라 **내부조인(inner join)을 사용**하면, 
where 절에서 필터링 하는 것과 기능이 동일하다. 따라서 on 절을 활용한 조인 대상 필터링을 사용할 때, 
**내부조인 이면 익숙한 where 절로 해결**하고, 정말 **외부조인이 필요한 경우에만 이 기능(on절)을 사용**하자.

