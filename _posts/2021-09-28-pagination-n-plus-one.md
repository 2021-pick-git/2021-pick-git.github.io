---
layout: post
author: Kevin
title: JPA Pagination, 그리고 N + 1 문제
date: 2021-09-28 13:32:20 +0300
description: Pagination과 Fetch Join을 함께 사용할 때 발생할 수 있는 문제를 대처해봅시다.
img: nplusone.jpeg
tags: [백엔드, JPA]
---

## 1. 들어가며

게시판 기능을 제공하는 웹 어플리케이션에 접속하여 게시물 목록을 요청하는 경우를 상상해봅시다. DB에 저장되어 있는 게시물은 수백 만개에 육박할 수도 있습니다. 모든 게시물 목록을 조회해 화면에 렌더링하는 경우, 클라이언트가 브라우저 혹은 모바일 기기로 이를 한 눈에 보기 어려움을 겪을 공산이 큽니다. 또한 클라이언트가 보지도 않을 데이터까지 DB에서 조회하여 네트워크를 통해 전달하기 때문에, 서버의 리소스가 불필요하게 낭비됩니다.

![image](https://user-images.githubusercontent.com/56240505/126959267-f01b6829-19bf-4662-829b-651049cc55c2.png)

반면 위 사진처럼 한 페이지에서는 N개의 데이터만 보여주고, 다음 페이지로 이동하라는 클라이언트의 추가 요청이 있을 때 마다 다음 순번의 N개의 데이터를 보여준다면 UX 및 리소스 측면의 단점을 보완할 수 있습니다. 이 처럼 한 화면에 보여주는 데이터의 범위를 결정하는 일련의 방식을 페이지네이션 혹은 페이징이라고 합니다. Pick-Git은 위 사진과 같이 게시판 형태가 아닌 인피니티 스크롤을 기반으로 페이지네이션을 적용하고 있습니다. 그리고 이번 글에서는 Pick-Git을 개발하면서 당면했던 JPA Pagination 및 N + 1 문제 및 해결 방법을 공유하고자 합니다.

<br>

## 2. Pagination

일반적으로 DB 벤더에 따라 페이징을 처리하는 쿼리가 천차만별입니다. MySQL의 경우 LIMIT 및 OFFSET 구문 등을 사용함으로써 페이징을 처리할 수 있지만, Oracle은 그보다 더 복잡한 쿼리가 수반됩니다. 그러나 JPA를 사용한다면 별도의 쿼리 작성 없이 Pagination API를 사용하기만 하면 됩니다. JPA가 설정된 DB 벤더 방언에 맞게 페이징 쿼리를 자동으로 생성하기 때문입니다.

> PostRepository.java

```java
@Query("select p from Post p")
List<Post> findWithPagination(Pageable pageable);
```

> PostRepositoryTest.java

```java
@DisplayName("Pageable을 사용하여 페이징 처리한다.")
@Test
void pagination() {
    postRepository.findWithPagination(Pageable.ofSize(10));
    // postRepository.findWithPagination(PageRequest.of(0, 2)); 가능
}
```

> SQL

```sql
Hibernate:
    select
        post0_.id as id1_4_,
        post0_.content as content2_4_
    from
        post post0_ limit ?
```

Spring Data JPA의 경우 Pageable 구현체를 Repository 쿼리 메서드의 파라미터로 전달함으로써 쿼리에 페이징을 동적으로 추가할 수 있습니다. 개발자가 Pageable 인터페이스를 직접 구현하거나, 미리 준비되어 있는 정적 팩토리 메서드를 통해 간편하게 페이지네이션 범위를 지정할 수 있습니다. 현재 설정된 방언에 따라 JPQL 쿼리 및 페이지네이션 기능이 SQL 쿼리로 변환되는 것을 확인할 수 있습니다.

<br>

## 3. N + 1 및 Fetch Join

> Post.java

```java
@Entity
public class Post {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id")
    private User user;

    @OneToMany(mappedBy = "post", fetch = FetchType.LAZY)
    private List<Image> images = new ArrayList<>();

    @OneToMany(mappedBy = "post", fetch = FetchType.LAZY)
    private List<Like> likes;

    @OneToMany(mappedBy = "post", fetch = FetchType.LAZY)
    private List<Comment> comments;

    @OneToMany(mappedBy = "post", fetch = FetchType.LAZY)
    private List<PostTag> postTags;

    // ...
}
```

Pick-Git 프로젝트에서 사용하는 Post 엔티티를 간략하게 표현한 예제 코드입니다.


> PostService.java

```java
@Transactional(readOnly = true)
public List<PostResponseDto> homeFeed(HomeFeedRequestDto homeFeedRequestDto) {
    Pageable pageable = getPagination(homeFeedRequestDto);
    User requestUser = findUserByName(homeFeedRequestDto.getRequestUserName());
    List<Post> result = postRepository.findAllAssociatedPostsByUser(requestUser, pageable);
    return PostDtoAssembler.assembleFrom(requestUser, result);
}
```

복수건의 게시물 조회 요청의 경우, Pagination API를 바탕으로 게시물을 조회합니다.

> PostResponseDto.java

```java
public class PostResponseDto {

    private Long id;
    private List<String> imageUrls;
    private String authorName;
    private Integer likesCount;
    private List<String> tags;
    private List<CommentResponseDto> comments;
    // ...
}
```

이후 Post 엔티티 내부 필드(연관 엔티티)들을 ``getter``로 호출해 가공하여 응답 DTO를 작성합니다. 기능을 구현하고 보니 JPA가 Post 페이징 조회 쿼리 1건 이외에도 추가적으로 5건(Comment, Like, Image, User, Tag)의 조회 쿼리를 날리고 있었습니다. 즉, 한 개의 트랜잭션을 처리하는데 총 6번의 SQL 조회 쿼리가 발생하는 심각한 문제였습니다.

이러한 현상을 N + 1 이라고 합니다. 쿼리 1번으로 N건의 엔티티를 가져왔는데, 글로벌 지연 로딩 전략으로 인해 관련 컬럼을 얻기 위해 쿼리를 N번 추가로 수행하는 현상을 의미합니다. 테스트 코드를 통해 더 자세히 살펴보겠습니다.

> PostRepositoryTest.java

```java
@DisplayName("Pageable을 사용하여 페이징 처리한다.")
@Test
void pagination() {
    List<Post> posts = postRepository.findWithPagination(PageRequest.of(0, 3));

    for (Post post : posts) {
        System.out.println(post.getComments());
    }
}
```

> SQL

```sql
Hibernate:
    select
        post0_.id as id1_4_,
        post0_.content as content2_4_
    from
        post post0_ limit ?
Hibernate:
    select
        comments0_.post_id as post_id4_2_0_,
        comments0_.id as id1_2_0_,
        comments0_.id as id1_2_1_,
        comments0_.content as content2_2_1_,
        comments0_.like_id as like_id3_2_1_,
        comments0_.post_id as post_id4_2_1_
    from
        comment comments0_
    where
        comments0_.post_id=?
[com.learning.jpa.domain.Comment@3a1238cc]
Hibernate:
    select
        comments0_.post_id as post_id4_2_0_,
        comments0_.id as id1_2_0_,
        comments0_.id as id1_2_1_,
        comments0_.content as content2_2_1_,
        comments0_.like_id as like_id3_2_1_,
        comments0_.post_id as post_id4_2_1_
    from
        comment comments0_
    where
        comments0_.post_id=?
[com.learning.jpa.domain.Comment@ce5df3f]
Hibernate:
    select
        comments0_.post_id as post_id4_2_0_,
        comments0_.id as id1_2_0_,
        comments0_.id as id1_2_1_,
        comments0_.content as content2_2_1_,
        comments0_.like_id as like_id3_2_1_,
        comments0_.post_id as post_id4_2_1_
    from
        comment comments0_
    where
        comments0_.post_id=?
[com.learning.jpa.domain.Comment@5ab690ec]
```

페이징으로 조회한 N개의 Post 엔티티를 순회하며 Comment를 조회해봅시다. Post는 1:N 관계를 맺고 있는 Comment에 대해 글로벌 지연 로딩 전략을 채택하고 있습니다. 그 결과, 페이징(LIMIT)을 통해 조회한 Post 리스트를 순회하면서 ``getComments()``를 호출할 때마다 추가적으로 Comment 조회 쿼리가 발생합니다. 즉, 조회한 Post가 1000개라면 1000개의 추가 쿼리가 발생하는 전형적인 **N + 1 문제**가 나타납니다.

실제 Pick-Git에서 사용하는 Post 엔티티의 경우, 응답을 작성할 때 Comment 뿐만 아니라 Image, Tag, Like, User 등 여러 지연 로딩된 엔티티 테이블들을 추가적으로 참조합니다. 따라서 조회한 Post가 1000건이면 5000건의 추가 쿼리가 발생하게 됩니다.

이를 해결하기 위해 대게 Fetch Join을 적용하게 됩니다.

### 3.1. Fetch Join

Fetch Join이란 JPQL로 특정 엔티티를 조회할 때 연관된 엔티티 혹은 컬렉션을 즉시 로딩과 같이 한 번에 함께 조회하는 기능입니다.

> PostRepository.java

```java
@Query("select distinct p from Post p join fetch p.comments")
List<Post> findWithPagination(Pageable pageable);
```

> SQL

```sql
2021-07-26 22:34:32.764  WARN 14832 --- [    Test worker] o.h.h.internal.ast.QueryTranslatorImpl   : HHH000104: firstResult/maxResults specified with collection fetch; applying in memory!
Hibernate:
    select
        distinct post0_.id as id1_4_0_,
        comments1_.id as id1_2_1_,
        post0_.content as content2_4_0_,
        comments1_.content as content2_2_1_,
        comments1_.like_id as like_id3_2_1_,
        comments1_.post_id as post_id4_2_1_,
        comments1_.post_id as post_id4_2_0__,
        comments1_.id as id1_2_0__
    from
        post post0_
    inner join
        comment comments1_
            on post0_.id=comments1_.post_id
[com.learning.jpa.domain.Comment@72dc246c]
[com.learning.jpa.domain.Comment@6e76be45]
[com.learning.jpa.domain.Comment@37ef8e6b]
```

``Post 전체 조회 쿼리 1개 (1) + 각 Post별 Comment 조회 쿼리 3개 (N)`` 총 쿼리가 4개가 날아가던 것이 Fetch Join을 적용함으로써 1개로 줄어들었습니다. 성능이 최적화된 것으로 보이지만 다소 이상한 부분이 존재합니다.

먼저, 페이징할 때 사용하던 기존의 SQL LIMIT 구문이 등장하지 않았습니다. 또한 쿼리 결과를 전부 메모리에 적재한 뒤 어플리케이션 단에서 Pagination 작업을 수행한다는 경고 로그가 발생합니다. 왜 이러한 현상이 발생하는 것일까요?

Post 엔티티가 3개 있고, 각각의 Post 엔티티는 연관된 Comment가 7개 존재한다고 가정해봅시다. 1:N 관계를 Join하면 총 21(3 * 7)개의 DB Row가 조회됩니다. 데이터의 수가 변경되기 때문에 단순하게 LIMIT 구문을 사용하는 쿼리로 페이지네이션을 적용하기 어렵습니다. 따라서 조회한 결과를 모두 메모리로 가져와서 JPA가 페이지네이션 계산을 진행합니다.

**1:N 관계의 컬렉션을 Fetch Join하면서 동시에 Pagination API를 사용하면 OutOfMemoryError가 발생할 수 있기 때문에, 이 둘을 동시에 사용해서는 안 됩니다.**

> CommentTest.java

```java
List<Comment> comments = entityManager
    .createQuery("select c from Comment c join fetch c.post", Comment.class)
    .setFirstResult(0)
    .setMaxResults(10)
    .getResultList();
```

> SQL

```sql
Hibernate:
    select
        comment0_.id as id1_2_0_,
        post1_.id as id1_4_1_,
        comment0_.content as content2_2_0_,
        comment0_.post_id as post_id4_2_0_,
        post1_.content as content2_4_1_
    from
        comment comment0_
    inner join
        post post1_
            on comment0_.post_id=post1_.id limit ?
```

반대로 N:1 관계의 엔티티를 Fetch Join할 때는 문제없이 Pagination API를 적용할 수 있습니다. Comment와 Post는 N:1 관계이기 때문에 Join해도 조회되는 DB ROW의 수가 변경되지 않기 때문입니다. SQL 로그에도 LIMIT 구문이 잘 찍혀있고, 별도의 경고 로그가 발생하지 않습니다.

<br>

## 4. 해결 방안

Pagination API를 사용할 때 ``~ToOne`` 관계의 엔티티는 Fetch Join해도 괜찮지만, ``~ToMany`` 관계의 엔티티에 대해서는 다른 접근이 필요합니다.

> application.properties

```properties
spring.jpa.properties.hibernate.default_batch_fetch_size=1000
```

그 중 가장 쉬운 해결 방안으로는 Batch Size를 지정하는 것입니다. @BatchSize 애너테이션을 부착할 수도 있지만, 어플리케이션 전역 설정을 위해 properties에 값을 정의했습니다. 해당 옵션에 대한 설명은 후술하겠습니다.

> PostRepositoryTest.java

```java
List<Post> posts = entityManager.createQuery("select p from Post p", Post.class)
    .setFirstResult(0)
    .setMaxResults(10)
    .getResultList();
```

> SQL

```sql
Hibernate:
    select
        post0_.id as id1_4_,
        post0_.content as content2_4_
    from
        post post0_ limit ?
Hibernate:
    select
        comments0_.post_id as post_id4_2_1_,
        comments0_.id as id1_2_1_,
        comments0_.id as id1_2_0_,
        comments0_.content as content2_2_0_,
        comments0_.like_id as like_id3_2_0_,
        comments0_.post_id as post_id4_2_0_
    from
        comment comments0_
    where
        comments0_.post_id in (
            ?, ?, ?
        )
[com.learning.jpa.domain.Comment@6090a8fc]
[com.learning.jpa.domain.Comment@77d45b37]
[com.learning.jpa.domain.Comment@23447af]
```

Fetch Join 없이 Pagination API를 사용해보았습니다. 페이징(LIMIT)을 통해 POST 리스트를 조회하지만, Comment 관련 조회 쿼리가 기존과 다르게 나갑니다. 기존에는 반복문을 순회하면서 N개의 Post 엔티티에 대해 ``where comments0_.post_id=?``를 포함하는 Comment 조회 쿼리가 N번 발생했습니다. 그러나 Batch Size를 적용한 결과 ``where comments0_.post_id in (?, ?, ?)``를 포함하는 Comment 조회 쿼리 1개로 줄어들었습니다.

@BatchSize 혹은 ``spring.jpa.properties.hibernate.default_batch_fetch_size`` 옵션을 적용하면

1. X 타입 엔티티가 지연 로딩된 ``~ToMany`` 관계의 Y 타입 컬렉션을 최초 조회할 때
2. 이미 조회한 X 타입 엔티티(즉, 영속성 컨텍스트에서 관리되고 있는 엔티티)들의 ID들을 모아서
3. ``WHERE Y.X_ID IN (?, ?, ?...)`` 와 같은 SQL IN 구문에 담아 Y 타입 데이터 조회 쿼리를 날립니다.
4. X 타입 엔티티들이 필요로 하는 모든 Y 타입 데이터를 한 번에 조회합니다.

여기서 Batch Size 옵션에 할당되는 숫자는 **IN 구문에 넣을 부모 엔티티 Key(ID)의 최대 개수**를 의미합니다.

예를 들어 봅시다. 별다른 조치를 취하지 않은 상황에서, Post 1000개가 담긴 리스트를 순회하면서 Comment를 호출하는 코드는 지연 로딩으로 인해 1000개의 추가 쿼리가 발생합니다.

반면 Batch Size 옵션을 1000으로 지정해두면, 반복문을 순회하며 Comments를 최초로 조회하는 시점에

1. 영속성 컨텍스트에서 관리되고 있는 1000개의 Post 엔티티 ID가
2. Comments 조회 쿼리의 IN 구문 ``where comment.post_id in (?, ?, ?, ...)``에 포함되어 날아갑니다.
3. 단 하나의 Comment 조회 쿼리로 1000개의 Post 엔티티가 필요로 하는 모든 Comment 관련 데이터를 조회해옵니다.

보통 옵션값을 1,000 이상 주지 않습니다.

### 4.1. 결론

> PostRepository.java

```java
@Query("select p from Post p left join fetch p.user order by p.createdAt desc")
List<Post> findAllPosts(Pageable pageable);
```

따라서 Pick-Git은 Post를 Pagination API를 통해 조회할 때 다음과 같은 방법을 사용합니다.

* @ManyToOne 등 N:1 관계의 자식 엔티티에 대해서는 모두 Fetch Join을 적용함으로써 한방 쿼리를 수행한다.
* @OneToMany 등 1:N 관계의 자식 엔티티는 Fetch Join을 하지 않는다.
  * 대신 상기 선언한 ``hibernate.default_batch_fetch_size`` 적용을 통한 in 쿼리로 성능을 보장한다.

<br>

## 5. 마치며

Batch Size 옵션은 상술한 ``Pagination + Fetch Join 문제``뿐만 아니라, 다른 JPA의 한계를 극복하는데 도움이 됩니다.

가령, JPA는 엔티티를 조회할 때 2개 이상의 1:N 관계의 컬렉션을 Fetch Join하지 못하도록 되어있습니다. 조회되는 데이터가 너무 많아지는 카테시안 곱(Cartesian Product) 때문에, 2개 이상의 컬렉션을 Fetch Join하면 MultipleBagFetchException이 발생합니다. 오직 1개의 컬렉션만 Fetch Join이 가능합니다.

특정 엔티티가 보유하는 1:N 관계의 컬렉션 모두를 즉시 로딩처럼 조회할 수 없다는 것은, 이번 글의 Post 엔티티 예제처럼 N + 1 문제가 발생한다는 것을 시사합니다. 하지만 Batch Size 옵션을 통해 이러한 JPA Fetch Join 한계로 인해 발생하는 N + 1 문제를 어느 정도 해소할 수 있습니다.

<br>

---

## Reference

* [jpa fetch join](https://velog.io/@rainmaker007/jpa-fetch-join)
* [MultipleBagFetchException 발생시 해결 방법](https://jojoldu.tistory.com/457)
* [fetch join 시 paging 문제](https://www.inflearn.com/questions/14663)
* [fetch join 과 pagination 을 같이 쓸 때 [HHH000104: firstResult/maxResults specified with collection fetch; applying in memory]](https://javabom.tistory.com/104)
* [N+1 쿼리 문제](https://zetawiki.com/wiki/N%2B1_%EC%BF%BC%EB%A6%AC_%EB%AC%B8%EC%A0%9C)
* [이미지 출처](https://www.infragistics.com/help/aspnet/webdatagrid-paging)
* [티저 출처](https://medium.com/the-marcy-lab-school/what-is-the-n-1-problem-in-graphql-dd4921cb3c1a)
