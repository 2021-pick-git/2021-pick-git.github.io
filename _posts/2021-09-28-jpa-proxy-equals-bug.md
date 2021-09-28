---
layout: post
title: JPA 프록시 관련 버그 경험기
date: 2021-09-28 13:32:20 +0300
description: 프로젝트를 진행하면서 JPA 프록시 관련 버그를 경험하고 해결한 과정을 공유합니다.
img: jpa.png
tags: [백엔드, JPA]
---

# INTRO
- JPA 에서는 데이터베이스에서 연관객체 탐색을 효율적으로 하기 위해서 지연로딩 전략을 사용한다. 
- 지연로딩의 핵심은 연관관계에 있는 Entity가 실제로 사용되기 이전까지 DB에 실제로 참조하지 않고 프록시 객체로 대체하는 것이다. 
- JPA의 프록시 객체는 유용하지만 내부 동작방식에 대해서 제대로 알고있지 않으면 찾기 어려운 버그를 만날 수도 있다. 
- 다음은 JPA proxy 관련해서 프로젝트 진행시 만난 버그에 대한 내용이다. 

<br>
<br>

# 문제 상황 

### Entity 구조
- **참고**: 설명과 관련된 부분만 남기고 다른 로직 및 어노테이션은 대부분 생략했다. 

1. `Post` - 게시물 엔티티
    ```java
    @Entity
    public class Post {

        @Id
        @GeneratedValue(strategy = GenerationType.IDENTITY)
        private Long id;

        @ManyToOne(fetch = FetchType.LAZY)
        @JoinColumn(name = "user_id")
        private User user;

        @Embedded
        private Images images;

        @Embedded
        private PostContent content;

        @Embedded
        private Likes likes; //설명과 관련된 프로퍼티!! 

        @Embedded
        private Comments comments;

        @Embedded
        private PostTags postTags;

        private String githubRepoUrl;

        protected Post() {
        }

        //...부생성자 생략 

        //...불필요한 비지니스 로직 생략 
        //...getter 생략
    }
    ```
    <br>

2. `Likes`와 `Like` - Post 엔티티 하위의 Embedded 게시물 Like collection 포장객체
    - **참고**: 설명하고자 하는 부분과 깊게 연관된 핵심 Entity는 아니지만  상황 설명을 위해 간단히 프로퍼티만 소개한다. 
    ```java
    @Embeddable // Post 엔티티 안에 Embedded 되어 있음 
    public class Likes {

        @OneToMany(
            mappedBy = "post",
            fetch = FetchType.LAZY,
            cascade = CascadeType.PERSIST,
            orphanRemoval = true
        )
        private List<Like> likes;

        public Likes() {
            this(new ArrayList<>());
        }

        public Likes(List<Like> likes) {
            this.likes = likes;
        }

        //...getter 및 불필요한 비지니스 로직 생략
    }

    @Entity
    public class Like {

        @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
        private Long id;

        @ManyToOne(fetch = FetchType.LAZY)
        @JoinColumn(name = "post_id")
        private Post post;

        @ManyToOne(fetch = FetchType.LAZY)
        @JoinColumn(name = "user_id")
        private User user;

        protected Like() {
        }

        //...getter 및 불필요한 비지니스 로직 생략
    }    
    ```
    <br>

3. `User` - 어플리케이션 사용자 (게시물 좋아요, 유저간 팔로우 팔로잉 등의 행위를 함)
    ```java
    @Entity
    public class User {

        @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
        private Long id;

        @Embedded
        private Followers followers;

        @Embedded
        private Followings followings;

        //...일부 프로퍼티 생략 

        protected User() {
        }

        //...부 생성자 생략 

        public Boolean isFollowing(User targetUser) { //문제를 발생시킨 핵심 메소드!!! 
            if (this.equals(targetUser)) {
                return null;
            }

            return this.followings.isFollowing(targetUser);
        }

        //...getter 및 불필요한 비지니스 로직 생략 

        @Override
        public boolean equals(Object o) { //User는 Entity 이므로 Id로 동일성 및 동등성 확인 
            if (this == o) {
                return true;
            }

            if (o == null || getClass() != o.getClass()) { //중요한 포인트!!!! 
                return false;
            }

            User user = (User) o;

            return id != null ? id.equals(user.getId()) : user.getId() == null;
        }

        @Override
        public int hashCode() {
            return Objects.hash(getId());
        }
    }
    ```
    <br>

4. `Followings`와 `Follow`
    - `Followings` - 해당 `User`의 팔로워리스트를 저장하는 포장객체 (`Followers`도 동일한 형태로 되어 있다.)
    - `Follow` - Followers, Followings 리스트에 담겨 있는 VO 엔티티로 source, target 유저간의 팔로우 관계를 나타내는 엔티티
    ```java
    @Embeddable
    public class Followings {

        @OneToMany(
            mappedBy = "source",
            fetch = FetchType.LAZY,
            cascade = CascadeType.PERSIST,
            orphanRemoval = true
        )
        private List<Follow> followings;

        protected Followings() {
        }

        //...생성자 생략 

        public Boolean isFollowing(User targetUser) { //문제를 발생시킨 핵심 메소드!!! 
            return followings.stream()
                .anyMatch(follow -> follow.isFollowing(targetUser));
        }

        //...일부 비지니스 로직 생략
    }

    @Entity
    @Table(
        uniqueConstraints = {
            @UniqueConstraint(columnNames = {"source_id", "target_id"})
        }
    )
    public class Follow {

        @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
        private Long id;

        @ManyToOne(fetch = FetchType.LAZY)
        @JoinColumn(name = "source_id")
        private User source;

        @ManyToOne(fetch = FetchType.LAZY)
        @JoinColumn(name = "target_id")
        private User target;

        protected Follow() {
        }

        //...생성자 및 유효성 검사 로직 생략 

        public boolean isFollowing(User targetUser) { //문제를 발생시킨 핵심 메소드!!! 
            return this.target.equals(targetUser);
        }

        //...getter 생략 
        //equals & hashcode는 VO로 취급되어 필드가 같은지 확인 (즉, 유저가 같은 유저인지 확인)
    }
    ```
    <br>

5. `PostRepository` 게시물 좋아요 리스트 조회 쿼리
    ```java
    @Query("select distinct p from Post p left join fetch p.likes where p.id = :postId")
    Optional<Post> findPostWithLikeUsers(@Param("postId") Long postId); // 포스트를 조회할 때 좋아요 리스트를 fetch join 해서 즉시로딩 한다. 
    ```
    <br>

### 버그 발생
- 현재 흐름은 다음과 같다.  
    1. `Post`를 좋아요 한 유저 리스트를 반환하려함. (`Post`내부의 `Likes`를 반환)
    2. 좋아요 한 유저 리스트를 조회할 때, 조회하는 source 유저가 팔로잉 하고 있는 target 유저는 팔로잉 중이라고 나타냄.
    <br>
    <p align="center"><img width=220 src="https://user-images.githubusercontent.com/63405904/130729021-67475c69-7b74-46bf-ac55-2902ddded2f9.png"></p>
    
    3. source 유저가 target user를 following 하고 있는 여부를 `User`의 `isFollowing` 메소드를 통해서 확인함.
        - 이때 source와 target이 같은 경우(자기 자신인 경우) - `null` 반환
        - source가 target을 팔로잉 중인 경우 - `true` 반환
        - source가 target을 필로우 하지 않는 경우 - `false` 반환 
    

<br>

- **문제는, 로그인 한 source 유저와 즉시로딩 해 가져온 좋아요 리스트의 User 간의 팔로잉 여부가 모두 `false`로 출력이 된 것이다.** 

<br>
<br>

# 발생 원인 

- `PostRepository`에서 `findPostWithLikeUsers()`를 사용해 포스트와 좋아요 리스트를 즉시로딩(`@OneToMany` 관계) 할 때 `Like` 엔티티 내부의 `User`는 즉시로딩 하지 않으므로 proxy 객체이다. 
- 좋아요한 target 유저 리스트를 가져와서 로그인 유저인 source 유저의 `isFollowing()` 메소드로 두 유저간의 팔로잉 여부를 확인한다. 
    ```java
    // User.java
    public Boolean isFollowing(User targetUser) {
        if (this.equals(targetUser)) { //자기 자신일 경우 null 반환
            return null;
        }

        return this.followings.isFollowing(targetUser);
    }

    //Followings.java
    public Boolean isFollowing(User targetUser) {
        return followings.stream()
            .anyMatch(follow -> follow.isFollowing(targetUser));
    }

    //Follow.java
    public boolean isFollowing(User targetUser) { 
        return this.target.equals(targetUser); //User.java 의 equals & hashCode를 사용 
    }
    ```
- **아무리 디버깅을 해봐도 비교하는 source 유저와 target 유저의 식별자(Id)가 같음에도 불구하고 `Follow.java`의 `isFollowing()`에서 false가 반환 되었다.**
- 그 원인은 `User.java`에서 오버라이드한 equals hashcode에 있었다. 
    ```java
    @Override
    public boolean equals(Object o) {
        if (this == o) {
            return true;
        }

        if (o == null || getClass() != o.getClass()) { //(1) 중요한 포인트!!!! 
            return false;
        }

        User user = (User) o;

        return id != null ? id.equals(user.getId()) : user.getId() == null;
    }

    @Override
    public int hashCode() {
        return Objects.hash(getId());
    }
    ```
- 위 **(1)** 에서 `User`객체의 Id로 비교하기 이전에 두 객체가 같은 클래스인지 `o.getClass()`로 비교하고 있었다. 하지만 proxy 객체는 `getClass()` 로 비교하면 실제 entity와 같지 않기 때문에 `false`를 반환한다. 
- 따라서 프록시 객체와 실제 entity를 비교할때는 `instance of`를 사용해야한다. <br>[JPA Proxy 참고링크](https://prolog.techcourse.co.kr/posts/1334)

<br>
<br>

# 해결 방법

- 생각한 해결방법은 2가지 이다. 
1. `Post`와 `Like`의 `fetch join` 시 `Like`의 `User` 까지 모두 `fetch join`으로 즉시로딩
   
    - Post와 Like -> `@OneToMany` 관계
    - Like와 User -> `@ManyToOne` 관계
    - 위와 같은 연관관계는 두 번 fetch join 하여 Like의 User까지 즉시로딩 할 수 있다. 
        ```java
        @Query("select distinct p from Post p left join fetch p.likes.likes l left join fetch l.user where p.id = :postId")
        Optional<Post> findPostWithLikeUsers(@Param("postId") Long postId);
        ```
    - 위와 같이 `User`까지 즉시로딩 한다면, `User`가 더 이상 proxy 객체가 아니기 때문에 `getClass()`를 해도 문제가 발생하지 않는다. 
    - 하지만 지나치게 복잡한 연관관계를 즉시로딩 하는 것이며 JPQL에서 fetch join 시 별칭을 쓰는 것은 JPA 표준 스펙에 맞지 않기 때문에 추천하는 방법이 아니다. (Hibernate 구현상 가능하므로 할 수 있긴 하다.) <br>
    [JPA fetch join 시 별칭 참고링크](https://www.inflearn.com/questions/15876)

    <br>
    
2. `User`의 `equals()` 메소드의 `getClass()` 비교 부분을 `instance of` 로 수정 
    - `User.java`의 equals 메소드를 다음과 같이 수정하면 올바른 값을 반환한다. 
        ```java
        @Override
        public boolean equals(Object o) {
            if (this == o) {
                return true;
            }
            if (!(o instanceof User)) { // 이 부분!!  
                return false;
            }

            User user = (User) o;

            return id != null ? id.equals(user.getId()) : user.getId() == null;
        }
        ```

<br>
<br>

# 마무리
- JPA Proxy 객체에 대해서 학습했으나, 이론으로 알고 있던 부분을 직접 버그로 경험하며 학습할 수 있었다.
- 디버깅 시 User 객체의 주소값이 ID가 같을 경우 같은 해시값으로 찍혔기 때문에 원인을 알기 더 어려웠다. 
- 또한 디버깅 포인트를 override 하여 IDE에서 자동으로 추가한 `equals()`에 걸 생각을 하지 못한 것도 디버깅을 어렵게 했던 포인트였다. 
- 개인적으로 `equals()`를 수정하는 두번째 해결방법을 추천하지만, override 한 메소드를 수정하는 것이 다른 팀원에게 충분히 공유되지 않으면 또다른 버그 포인트가 될 수 있다고 생각한다. (당연하게 생각하여 자세히 들여다보지 않는 부분이므로)

```toc
```