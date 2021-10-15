---
layout: post
title: 테스트코드 최적화 여행기 (2)
date: 2021-10-07 16:13:00 +0900
description: Pickgit의 테스트코드 최적화 방법을 소개합니다.
img: testoptima.png
tags: [백엔드, test, 최적화]
---


안녕하세요 깃들다팀의 손너잘 입니다.

테스트 최적화의 두번째 글을 작성하게 되었습니다.

이전 글에서 작성하였듯, 첫번째로 할 것은 DirtiesContext의 제거입니다.

많은 프로젝트에서 매번 테스트의 환경을 초기화 시키기 위해 DirtiesContext를 사용합니다. 하지만 DirtiesContext는 스프링 테스트가 매 테스트마다 Application Context를 다시 Load하도록 합니다. 이는 스프링 테스트가 제공해주는 Application Context의 캐싱의 이점을 전혀 가져가지 못합니다.

깃들다팀 또한 처음 프로젝트를 시작했을 떄 테스트의 환경을 초기화해주기 위해(정확히는 인메모리 DB초기화를 위함이지만요) DirtiesContext를 사용했습니다.

![image](https://user-images.githubusercontent.com/33603557/136336806-82c8a41b-b7a2-449b-9111-303977a1401f.png)

심지어 `BEFORE_EACH_TEST_METHOD` 를 사용하였죠... 프로젝트 초반에는 큰 문제가 없었으나 프로젝트가 점점 커질수록 테스트코드가 많아지고 문제가 발생했습니다.

![image](https://user-images.githubusercontent.com/33603557/136336820-6d34a488-6631-40a2-b1f4-3a430a4ccfd0.png)

무려 테스트를 한번 돌리는데 3분이 넘는 시간이 소요되는 문제였습니다.. 수치상으로는 3분이 찍혔지만 중간중간에 Application Context가 재시작 되는 시간이 누산되지 않아서 실제 시간은 거의 4분~5분이 소요됐습니다.

### DirtiesContext 제거

이를 해결하기 위해, 가장 먼저 DirtiesContext를 제거하여 Application Context를 재활용 할 수 있도록 하였습니다.

![image](https://user-images.githubusercontent.com/33603557/136336840-15cb0c86-4502-4dc9-8fe2-8249b7871175.png)

모든 AcceptanceTest의 DirtiesContext를 제거하고 공통으로 사용하는 어노테이션들을 추상클래스에 몰아넣어 테스트 클래스들이 이를 상속받도록 하였습니다. 하지만 여기서 문제가 발생합니다. Application Context가 공유되다보니 DB의 초기화작업이 필요하게 된 것 입니다.

참 많은 고민을 했습니다. `@Sql` 을 사용해 볼까?도 생각 해봤고, ApplicationTest 추상 클래스에 `@BeforeEach`에 제거 쿼리를 작성하여 초기화 할까도 생각했지만 이러한 방법에는 여러가지 단점이 있었습니다.

- 테이블의 외례키 제약조건에 따라서 테이블의 삭제 순서를 정해줘야 한다
- Entity의 변경사항을 자동으로 반영하지 못하고 변경이 발생할떄 마다 삭제 쿼리를 매번 수정해줘야한다.

이러한 고민을 하던 중 우아한테크코스의 `브라운` 코치님에게 힌트를 얻어 아래와 같은 클래스를 작성했습니다.

```java
@Component
@ActiveProfiles("test")
public class DatabaseCleaner implements InitializingBean {

    @PersistenceContext
    private EntityManager entityManager;

    private List<String> tableNames;

    @Override
    public void afterPropertiesSet() {
        entityManager.unwrap(Session.class)
                .doWork(this::extractTableNames);
    }

    private void extractTableNames(Connection conn) throws SQLException {
        List<String> tableNames = new ArrayList<>();

        ResultSet tables = conn
                .getMetaData()
                .getTables(conn.getCatalog(), null, "%", new String[]{"TABLE"});

        while(tables.next()) {
            tableNames.add(tables.getString("table_name"));
        }

        this.tableNames = tableNames;
    }

    public void execute() {
        entityManager.unwrap(Session.class)
                .doWork(this::cleanUpDatabase);
    }

    private void cleanUpDatabase(Connection conn) throws SQLException {
        Statement statement = conn.createStatement();
        statement.executeUpdate("SET REFERENTIAL_INTEGRITY FALSE");

        for (String tableName : tableNames) {
            statement.executeUpdate("TRUNCATE TABLE " + tableName);
            statement.executeUpdate("ALTER TABLE " + tableName + " ALTER COLUMN id RESTART WITH 1");
        }

        statement.executeUpdate("SET REFERENTIAL_INTEGRITY TRUE");
    }
}
```

소스를 설명하자면 다음과 같습니다.

- `@Component`와 `InitializingBean`을 통해 Application Context가 이 컴포넌트를 최초 로딩할 때 `afterPropertiesSet()` 의 소스를 수행하게 합니다.
- EntityManager를 받아와서 db의 테이블 정보를 받아오는 쿼리를 통해 table 이름을 저장합니다.

사실 살펴보면 별거 없습니다. 여기서 재미있는점은 `SET REFERENTIAL_INTEGRITY FALSE` 를 통해 무결성 제약조건을 잠시 OFF 시켜서 테이블의 제거 순서를 신경쓰지 않아도 되게한다는점 입니다.

그러면 이 컴포넌트를 어떻게 이용할 수 있을까요?

![image](https://user-images.githubusercontent.com/33603557/136336882-f6eeca59-d29b-4b4b-b01e-e44ade694749.png)

저같은 경우는 위에서 만든 AcceptanceTest 추상 클래스에 `DatabaseCleaner` 클래스를 DI받아서 `@AfterEach` 에서 db clear작업을 진행해 주도록 하였습니다.

그러면 결과를 한번 볼까요?

![image](https://user-images.githubusercontent.com/33603557/136336898-cc32ad2a-a923-4494-a879-669aba4e7c82.png)

3분이 넘게 걸리던 시간이 17~18초 정도로 줄어들었습니다! 엄청난 성과입니다. 깃들다 팀의 경우 인수테스트만 100개가 넘어가는데 이를 모두 새로운 ApplicationContext를 띄워서 테스트 하려고 하니 엄청난 시간이 걸렸던 것 입니다.

이번글에서는 DirtiesContext의 제거와 DB초기화에 대한 이야기를 나눠봤습니다.

다음 글에서는 테스트 환경을 통한 속도 향상 방법에 대해 작성해 보도록 하겠습니다.

읽어주셔서 감사합니다 🙂
