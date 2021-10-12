---
layout: post
title: 나만 몰랐었던 fetch join에 별칭을 쓰지 않는 이유
date: 2021-10-06 13:32:20 +0300
description: Fetch join 시 별칭을 쓰지 않는 이유에 대해 공부한 글입니다. 추가로 저희 프로젝트에서 별칭을 쓴 경우와 비교해 봅니다. 
img: jpa.png
tags: [백엔드, Database, Jpa]
---

## Intro
- JPA의 `fetch join` 사용시 별칭을 쓰면 안되는 이유가 무엇인지 알아본다. 
- [프로젝트](https://github.com/woowacourse-teams/2021-pick-git)애서 fetch join 시 별칭 사용에 대해서 고민해본다. 

## fetch join 별칭은 왜 안될까 ?
- fetch join에서 별칭이 안되는 이유는 데이터의 일관성이 깨지기 때문이다.
- 예를 들어서 다음과 같은 코드는 fetch join 대상에 조건문이 들어가서 일관성이 깨진 경우이다.

    ```java
    @DisplayName("fetch join 대상에 조건문을 걸었을 때 데이터가 불일치하다.")
    @Test
    void findTeamWithSpecificNameMember() {
        // given

        // 데이터 삽입
        Team teamA = new Team("teamA");
    
        Member memberA1 = new Member("memberA1");
        Member memberA2 = new Member("memberA2");
        Member memberA3 = new Member("memberA3");

        List<Member> teamAMembers = List.of(memberA1, memberA2, memberA3);

        teamA.setMembers(teamAMembers);
        teamRepository.save(teamA);

        testEntityManager.flush();
        testEntityManager.clear();

        // 데이터 조회
        Team teamA = teamRepository.findById(1L).orElseThrow();
        int teamAMemberSize = teamA.getMembers().size();
        testEntityManager.clear();

        // when
        Team teamAWithMemberName = teamRepository.findTeamWithSomeMemberByName("memberA1");

        // then
        /* 본래 teamA에 3명의 멤버가 들어가있지만 fetch join 대상에 where문이 들어가면서 데이터 불일치가 일어났다.
        * collection 에는 관련 데이터가 모두 들어가있기를 기대하는데 그렇지 않다.
        * 따라서 fetch join 대상에 필터링 조건을 거는 것을 지양한다. 
        */
        assertThat(teamA.getName()).isEqualTo(teamAWithMemberName.getName());
        assertThat(teamAMemberSize).isNotEqualTo(teamAWithMemberName.getMembers().size());
    }
    ```

- TeamA에 대한 member collection 은 본래 3개이다. 그리고 fetch join을 하면 연관된 데이터가 모두 들어올 것이라고 가정한다. 
- 하지만 위와 같이 fetch join 대상에 별칭을 주어 where 필터링 조건을 사용하면 실제로 TeamA에 연관된 멤버는 3명이지만 `memberA1`만 연관 데이터로 들어온다. 
- DB의 상태에 대한 일관성이 깨진다. 

### 하지만 예외는 있다

- 일관성을 해치지지 않는 한에서 성능에 도움이 된다면 예외적으로 사용해도 된다. (아마도 하이버네이트가 별칭을 허용하는 이유...)
- 예를 들어 다음과 같은 쿼리는 일관성을 해치지 않는다. 

    ```sql
    select m from Member m join fetch m.team where t.name = :teamName
    ```

- 하지만 위의 쿼리가 left join fetch로 되면 일관성이 깨진다. (Team이 null이 아닌 Member에 대해서 null 값이 들어가기 때문이다.)
- 때문에 매우 조심스럽게 사용해야한다. 

## 우리 프로젝트에 있는 별칭은?!

- 깃들다 프로젝트에도 fetch join 대상에 별칭을 사용하는 부분이 있다. 다음 [포스트](http://tech.pick-git.com/jpa-proxy-equals-bug/)에 어떤 상황이었는지 배경 설명이 자세하게 되어있다. 

```java
@Query("select distinct p from Post p left join fetch p.likes.likes l left join fetch l.user where p.id = :postId")
Optional<Post> findPostWithLikeUsers(@Param("postId") Long postId);
```

- Post 안에는 해당 게시물을 좋아요한 유저들 정보를 담은 `Like` 리스트가 담겨있다.

  ```java
  @Entity
  public class Post {

    //....

     @OneToMany(
        mappedBy = "post",
        fetch = FetchType.LAZY,
        cascade = CascadeType.PERSIST,
        orphanRemoval = true
    )
    private List<Like> likes;

    //...
  }

  public class Like {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "post_id")
    private Post post;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id")
    private User user;

    //...
  }
  ```

- 위 쿼리를 살펴보면 
  - 별칭이 `p.likes.likes l`에 사용된다. 
  - where 조건문에는 fetch join 대상을 필터링 하지 않는다. 
  - 따라서 데이터 일관성을 헤치지 않는다.

- fetch join을 할 때 주의해야하는 부분은 collection을 여러개 fetch join 할 경우이다. 
- 위 같은 경우는 `post -> like` 관계는 OneToMany라서 한번까지 fetch join 할 수 있다. 
- `like -> user`는 ManyToOne 관계 이므로 추가 fetch join을 할 수 있었다. 

## 마무리 
- 처음에 버그를 마주하고 fetch join 대상에 별칭을 두는 것이 찝찝했지만 왜 안되는지 모르는 상태로 (나만) 넘어갔다.
- 검토해보니 fetch join 대상이 아니었으며 여러 collection을 fetch join 하는 상황도 아니었다. 
- 하지만 이런 예외적인 경우는 자세히 알아보고 주의해서 사용해야 할 것 같다. 또 왜인지 모르고 그냥 안쓰지는 말자. 

<br>
<br>

> 백엔드 코다입니다 🙌
