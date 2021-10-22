---
layout: post
title: 하이버네이트 default-batch-fetch-size 가 안되는 현상 😢
date: 2021-10-22 13:32:20 +0300
description: 프로젝트를 진행하면서 JPA hiberbate.default_batch_fetch_size가 적용되지 않은 형상을 공유합니다.
img: jpa.png
tags: [백엔드, JPA]
---

## 🐾 목차
- [🐾 목차](#-목차)
- [💡 Intro](#-intro)
- [🌩 `hiberbate.default_batch_fetch_size`](#-hiberbatedefault_batch_fetch_size)
- [🌩 본 프로젝트 문제 상황](#-본-프로젝트-문제-상황)
  - [Entity 구조](#entity-구조)
  - [문제상황](#문제상황)
- [🌩 이상현상 들여다보기](#-이상현상-들여다보기)
  - [Entity 구조](#entity-구조-1)
  - [상황 재현](#상황-재현)
- [🌩 Help](#-help)

<br>
<br>

## 💡 Intro 

- JPA를 [프로젝트](https://github.com/woowacourse-teams/2021-pick-git)에서 사용하면서 연관 엔티티를 호출할 때 생기는 N+1을 해결한 경험이 있다. 이때 해결 방법으로 hibernate의 `default_batch_fetch_size`를 yml에 설정하여 해결했었다.
  - [참고링크](https://yjksw.github.io/jpa-query-bug/)
  - [해결부분](https://yjksw.github.io/jpa-query-bug/#%ED%95%B4%EA%B2%B0%EB%B0%A9%EB%B2%95-batchsize-%EC%A0%81%EC%9A%A9%ED%95%98%EA%B8%B0)
- 프로젝트를 전반적으로 체크하던 와중에 위 설정에 의한 in query가 실행되지 않고 여전히 N+1 문제가 발생하는 부분을 발견하였다. 
- 해당 현상을 공유하기 위해 글을 작성한다. (여전히 이유는 못 찾았다 😢)

<br>

## 🌩 `hiberbate.default_batch_fetch_size`

우선 간단하게 위 설정에 대해서 짚고 넘어가보자.

- 설정할 수 있는 방법은 두 가지 이다. 
  - `@BatchSize(size={sizeNum})` 어노테이션 활용 
      - 클래스, 메소드, 필드 레벨에서 사용할 수 있다. 
      - 해당 사이즈 만큼의 상위 엔티티 id가 in query로 나간다. 

  -  `spring.jpa.properties.hibernate.default_batch_fetch_size={batchSize}`를 application.properties에 지정
        -  전역적으로 적용이 되어서 상위 엔티티의 lazy loading된 하위 엔티티를 한꺼번에 in query로 로딩한다. 

<br>

- Hibernate javadocs 공식 문서에 다음과 같이 서술한다. 

    ```java
    Defines size for batch loading of collections or lazy entities. For example...
        @Entity
        @BatchSize(size=100)
        class Product {
            ...
        }
    
    will initialize up to 100 lazy Product entity proxies at a time.
            @OneToMany
            @BatchSize(size = 5) /
            Set getProducts() { ... };
    
    will initialize up to 5 lazy collections of products at a time
    ```

    - 즉, 속한 collection이나 lazy entities 들을 한꺼번에 batch로 로딩해준다. 
    - Batch로 로딩할 경우 하나의 쿼리로 연관 엔티티를 한꺼번에 가지고 올 수 있어서 성능이 향상된다. 

<br>

- 다음 [Hibernate Document](https://docs.jboss.org/hibernate/orm/4.3/manual/en-US/html/ch05.html)를 확인해보자.
  
    > @BatchSize specifies a "batch size" for fetching instances of this class by identifier. Not yet loaded instances are loaded batch-size at a time (default 1).

  - *not yet loaded instance*를 batch로 로딩할 수 있는 설정이라고 한다. 

<br>

## 🌩 본 프로젝트 문제 상황

### Entity 구조 

- 다소 복잡하지만 우리 프로젝트에서의 상황을 살펴보자. 
- 조금 이해하기 쉽게 프로젝트의 일부 엔티티 관계를 그림으로 표현해 보았다. 

    <p align="center"><img width="35%" src="https://user-images.githubusercontent.com/63405904/138268405-8ada5b7b-278a-4c6f-971c-9fddc4e5c44f.png"></p>

    - 사용자는 포트폴리오를 만들고 본인이 진행한 여러 프로젝트들을 포함시킬 수 있다.
    - 각 프로젝트마다 프로젝트를 나타내는 태그를 여러개 추가할 수 있다. 예를 들어 Java, Web 등등의 태그로 키워드를 나열할 수 있다.
    - 프로젝트와 태그는 다대다 관계이기 때문에 중간 테이블인 ProjectTag로 연결되어 있다. 
    - ProjectTag는 프로젝트 id와 태그 id를 가지고 있다. 

<br>

- 코드가 더 편한 사람들을 위해 Entity를 추가해본다. 편의를 위해 getter, 생성자, 다른 메소드와 관련 없는 필드들은 생략한다.

    1. Portfolio.java

        ```java
        @Entity
        public class Portfolio {

            @Id
            @GeneratedValue(strategy = GenerationType.IDENTITY)
            private Long id;

            // 다른 필드 생략 

            @OneToMany(
            mappedBy = "portfolio",
            fetch = FetchType.LAZY,
            cascade = CascadeType.PERSIST,
            orphanRemoval = true
            )
            private final List<Project> projects;
        ```

    2. Project.java

        ```java
        @Entity
        public class Project {

            @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
            private Long id;

            // 다른 필드 생략 

            @OneToMany(
                mappedBy = "project",
                fetch = FetchType.LAZY,
                cascade = CascadeType.PERSIST,
                orphanRemoval = true
            )
        
            private List<ProjectTag> tags;
        }
        ```

    3. ProjectTag.java

        ```java
        @Entity
        @Table(
            uniqueConstraints = {
                @UniqueConstraint(columnNames = {"tag_id", "project_id"})
            }
        )
        public class ProjectTag {

            @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
            private Long id;

            @ManyToOne(fetch = FetchType.LAZY)
            @JoinColumn(name = "tag_id")
            private Tag tag;

            @ManyToOne(fetch = FetchType.LAZY)
            @JoinColumn(name = "project_id")
            private Project project;
        }
        ```

    4. Tag.java

        ```java
        @Entity
        public class Tag {

            @Id
            @GeneratedValue(strategy = GenerationType.IDENTITY)
            private Long id;

            @Column(nullable = false, unique = true, length = MAX_TAG_LENGTH)
            private String name;
        }
        ```

<br>

### 문제상황

- 동일한 Assembler(같은 코드)로 응답 DTO를 만들때 포트폴리오를 조회할 때는 in 쿼리로 나가고, 포트폴리오를 업데이트 할 때는 n+1 쿼리가 나간다.
    ```java
    @Transactional(readOnly = true)
    public PortfolioResponseDto read(String username, UserDto userDto) {
        Optional<Portfolio> portfolio = portfolioRepository.findPortfolioByUsername(username);

        if (userDto.isGuest() && portfolio.isEmpty()) {
            throw new NoSuchPortfolioException();
        }

        return portfolioDtoAssembler.toPortfolioResponseDto(
            portfolio.orElseGet(() -> portfolioRepository.save(Portfolio.empty(getUser(username))))
        ); // default_batch_fetch_size로 인한 In 쿼리 수행 
    }

    @Transactional
    public PortfolioResponseDto update(PortfolioRequestDto portfolioRequestDto, UserDto userDto) {
        Portfolio portfolio = portfolioRepository.findById(portfolioRequestDto.getId())
            .orElseThrow(NoSuchPortfolioException::new);
        User user = getUser(userDto.getUsername());

        if (!portfolio.isOwnedBy(user)) {
            throw new UnauthorizedException();
        }
        portfolio.update(portfolioDtoAssembler.toPortfolio(portfolioRequestDto));

        entityManager.flush();

        return portfolioDtoAssembler.toPortfolioResponseDto(portfolio); //Tag를 lazy loading 할때 n+1 쿼리 발생
    }
    ```
  - [전체 코드보기](https://github.com/woowacourse-teams/2021-pick-git/blob/develop/backend/pick-git/src/main/java/com/woowacourse/pickgit/portfolio/application/PortfolioService.java)

<br>

- 쿼리 결과 

    ```java
    // read 메소드 실행 시 
    Hibernate: 
    select
        tag0_.id as id1_13_0_,
        tag0_.name as name2_13_0_ 
    from
        tag tag0_ 
    where
        tag0_.id in (
            ?, ?
        )

    // update 메소드 실행 시 
    Hibernate: 
    select
        tag0_.id as id1_13_0_,
        tag0_.name as name2_13_0_ 
    from
        tag tag0_ 
    where
        tag0_.id=?
    Hibernate: 
        select
            tag0_.id as id1_13_0_,
            tag0_.name as name2_13_0_ 
        from
            tag tag0_ 
        where
            tag0_.id=?
    ```

<br>

- 위 쿼리가 수행되는 `Portfolio -> PortfolioResponseDto` 로 변환시키는 assembler의 코드는 다음과 같다. 
  - 필드가 많아서 당황스럽겠지만 Project 부분만 보고 감만 잡으면 된다. (* 표시해둔 곳)
  - 간단히 말하면 get을 통해 lazy loading 하위 엔티티의 값을 가져온다. 
    
    ```java
    public PortfolioResponseDto toPortfolioResponseDto(Portfolio portfolio) {
        return new PortfolioResponseDto(
            portfolio.getId(),
            portfolio.getName(),
            portfolio.isProfileImageShown(),
            portfolio.getProfileImageUrl(),
            portfolio.getIntroduction(),
            portfolio.getCreatedAt(),
            portfolio.getUpdatedAt(),
            toContactResponsesDto(portfolio.getContacts()),
            toProjectResponsesDto(portfolio.getProjects()),  // *
            toSectionResponsesDto(portfolio.getSections())
        );
    }

    private List<ProjectResponseDto> toProjectResponsesDto(Projects projects) { // *
        return projects.getValues().stream()
            .map(this::toProjectResponseDto)
            .collect(toList());
    }

    private ProjectResponseDto toProjectResponseDto(Project project) { 
        List<TagResponseDto> tags = project.getTags().stream()
            .map(this::toTagResponseDto) // *
            .collect(toList());

        return new ProjectResponseDto(
            project.getId(),
            project.getName(),
            project.getStartDate(),
            project.getEndDate(),
            project.getType().getValue(),
            project.getImageUrl(),
            project.getContent(),
            tags
        );
    }

    private TagResponseDto toTagResponseDto(ProjectTag tag) { // *
        return new TagResponseDto(
            tag.getTagId(),
            tag.getTagName()
        );
    }
    ```

<br>

## 🌩 이상현상 들여다보기

- 프로젝트 코드로 들여다보기는 복잡하여 파악하기 어려움으로 동일한 상황을 간단한 테스트코드로 재현해보았다. 

### Entity 구조 

1. Member.java
    ```java
    @Entity
    public class Member {

        @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
        private Long id;

        private String name;

        @ManyToOne(fetch = FetchType.LAZY)
        @JoinColumn(name = "team_id")
        private Team team;

        // getter 및 생성자 생략
    } 
    ```

2. Team.java
    ```java
    @Entity
    public class Team {

        @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
        private Long id;

        private String name;

        // getter 및 생성자 생략
    }
    ```

<br>

### 상황 재현 

- 먼저 application.properties에 다음 설정을 해주었다. 
    ```properties
    spring.jpa.properties.hibernate.default_batch_fetch_size=10
    ```

- 다음은 In 쿼리가 정상동작하는 테스트코드다. 
    ```java
    @DisplayName("Member 리스트 조회 시 Team을 lazy loading 할 때 in 쿼리 Team이 한꺼번에 조회된다.")
    @Test
    void team_inquery_working() {

        // given
        Team teamA = new Team("TeamA");
        Team teamB = new Team("TeamB");
        teamRepository.save(teamA);
        teamRepository.save(teamB);

        testEntityManager.flush();
        testEntityManager.clear();

        Member member1 = new Member("member1");
        Member member2 = new Member("member2");
        member1.setTeam(teamA);
        member2.setTeam(teamB);

        memberRepository.save(member1);
        memberRepository.save(member2);

        testEntityManager.flush();
        testEntityManager.clear();

        // when
        List<Member> members = new ArrayList<>();
        members.add(memberRepository.findById(1L).get()); // Member 를 조회하는 쿼리가 생성된다.
        members.add(memberRepository.findById(2L).get());

        List<String> teamNames = members.stream()
            .map(member -> member.getTeam().getName())
            .collect(toList()); // Team 을 조회하는 쿼리가 in 쿼리로 수행된다.

        // then
        assertThat(teamNames).hasSize(2);
    }
    ```

    **[실행 Query]**
    ```
    Hibernate: 
    select
        team0_.id as id1_1_0_,
        team0_.name as name2_1_0_ 
    from
        team team0_ 
    where
        team0_.id in (
            ?, ?
        )
    ```

<br>

- 다음은 프로젝트 상황은 동일하게 재현한 in 쿼리가 수행되지 않는 테스트코드다. 
    ```java
    @DisplayName("Team을 initialize 할 때 in 쿼리가 수행되지 않는다.")
    @Test
    void team_inquery_notWorking() {

        EntityManager em = testEntityManager.getEntityManager();

        // given
        Team savedTeamA = teamRepository.save(new Team("TeamA"));
        Team savedTeamB = teamRepository.save(new Team("TeamB"));

        testEntityManager.flush();
        testEntityManager.clear();

        // when
        Member member1 = new Member("member1");
        Member member2 = new Member("member2");

        Team teamA = em.getReference(Team.class, savedTeamA.getId()); // Team은 프록시 객체다.
        Team teamB = em.getReference(Team.class, savedTeamB.getId());
        member1.setTeam(teamA);
        member2.setTeam(teamB);

        memberRepository.save(member1);
        memberRepository.save(member2);

        testEntityManager.flush();

        List<Member> members = new ArrayList<>();
        members.add(memberRepository.findById(1L).get()); // 영속성 컨텍스트에 있는 Member를 로딩한다.
        members.add(memberRepository.findById(2L).get());

        List<String> teamNames = members.stream()
            .map(member -> member.getTeam().getName()) // 각 멤버의 개수만큼 team을 select하는 쿼리를 실행한다.
            .collect(toList());

        // then
        assertThat(teamNames).hasSize(2);
    }
    ```

    **[실행 Query]**    
    ```
    Hibernate: 
        select
            team0_.id as id1_1_0_,
            team0_.name as name2_1_0_ 
        from
            team team0_ 
        where
            team0_.id=?
    
    Hibernate: 
        select
            team0_.id as id1_1_0_,
            team0_.name as name2_1_0_ 
        from
            team team0_ 
        where
            team0_.id=?
    ```

- `findAll()`로 전체 멤버를 조회할 수 있지만, 그렇다면 두번째 테스트코드의 경우 모든 멤버가 영속성 컨텍스트에 있음에도 불구하고 영속성 컨텍스트는 전체 데이터인지 알 수 없기 때문에 Member를 조회하는 쿼리를 날린다. 
- 영속성 컨텍스트에 이미 있는 엔티티를 가져온다는 것을 확인하기 위해 Member를 `findById()`로 가져왔다. 

<br>

## 🌩 Help

- `default_batch_fetch_size` 설정이 먹히지 않는 이유로 **해당 엔티티 (대상 엔티티 혹은 상위 엔티티)가 영속성 컨텍스트에서 관리되지 않을 경우**를 생각해볼 수 있다. 위 개념에서 다루었듯이 아직 초기화 되지 않은 collections 혹은 lazy products에 대해서 한꺼번에 로딩해주는 역할을 하기 때문이다. 만일 하위 엔티티가 영속성 컨텍스트에서 관리되고 있지 않다면 로딩할 프록시 또한 없을 것이고 상위 엔티티가 관리되고 있지 않다면 연관관계를 파악할 수 없으므로 in 쿼리에 인자로 보낼 id 값이 없을 것이다. 
- 위 테스트 코드를 보았을 때 첫번째와 두번째 상황을 요약해보자. 
    
    - 첫번째 테스트코드 - in 쿼리 동작
        - Member가 `findById`로 조회되고 Team은 프록시 객체이다. (Lazy loading) 
        - Member 리스트의 팀 목록을 조회할 때 Member의 Id가 in 쿼리로 들어간다. 

    - 두번째 테스트코드 - in 쿼리 동작 안함 
        - Team은 이미 존재한다.
        - 새로운 Member를 생성하고 Team을 em.getReference를 통해 Team의 프록시 객체를 Member의 Team을 지정한다. 이후 `save()`를 통해서 Member 엔티티를 저장하고 flush 하여 데이터베이스에 반영한다.  
        - `findAll()`를 통헤 멤버 List를 가져온다. 이때 영속성 컨텍스트에 있는 Member가 조회된다. 
        - 해당 Member의 Team은 `getReference()`로 조회된 프록시 객체이다. 

<br>

- 다음과 같은 이유로 두 테스트코드가 같은 상황이라고 생각한다. 
    - Member가 영속성 컨텍스트에 실제 엔티티로 관리되고 있다는 것. 
    - Member와 연관된 Team 엔티티가 모두 프록시 객체이며 영속성 컨텍스트에 있다는 것. 
    - 연관관계는 두 경우 모두 잘 매핑이 되어 있다는 것. 
      - 두번째 테스트코드의 마지막 `flush()` 이후 `clear()`를 통해 영속성 컨텍스를 한번 초기화 하면 in 쿼리가 정상동작한다. 
- 실제 조회된 Member 리스트의 내부를 디버깅해 들여다 보았다. 

    - In query가 정상 동작하는 members 

        <p align="center"><img width="75%" src="https://user-images.githubusercontent.com/63405904/138413600-1ec9f61a-5161-48a0-9919-fdbbcb1f88f7.png"></p>
    
    - In query가 동작하지 않는 members 

        <p align="center"><img width="75%" src="https://user-images.githubusercontent.com/63405904/138413853-021c8bab-ada8-49f5-9497-bac041da22ce.png"></p>

<br>

- 디버깅했을 때 두가지 상태가 모두 똑같지만 하나는 in 쿼리가 동작하고 하나는 동작하지 않는 이유를 결국 못 찾았다. 😢
- 우선은 현상만 기록하고 계속 알아볼 예정이다 !! 
- 혹시 아시는 분은 .. 연락주세요.. 깃헙이나 이메일, 댓글 아무거나 환영 !! 🎉

<br>
<br>

**[참고자료]**
- [apiref.com/hibernate5/BatchSize.html](apiref.com/hibernate5/BatchSize.html)
- [https://docs.jboss.org/hibernate/orm/4.3/manual/en-US/html/ch05.html](https://docs.jboss.org/hibernate/orm/4.3/manual/en-US/html/ch05.html)
- [https://wckhg89.tistory.com/10](https://wckhg89.tistory.com/10)


<br>
<br>

> 백엔드 코다입니다 🙌