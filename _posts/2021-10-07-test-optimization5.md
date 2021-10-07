---
layout: post
title: 테스트코드 최적화 여행기 (5)
date: 2021-10-07 16:34:00 +0900
description: Pickgit의 테스트코드 최적화 방법을 소개합니다.
img: testoptima.png
tags: [백엔드, test, 최적화]
---

안녕하세요 깃들다팀의 손너잘입니다.

드디어 마지막 글이네요! 저번 글에서는 인수테스트의 조회용 테스트와 그외 테스트를 분리하여 테스트 속도의 최적화가 가능함을 확인해 봤습니다.

이번글에서는 실제로 테스트에 이를 적용해 보도록 하겠습니다.

### 다중 데이터 소스와 DB 선택 전략

다중 데이터소스를 사용하는 자세한 방법에 대해서는 [링크]를 참고하여 주세요!

![image](https://user-images.githubusercontent.com/33603557/136339400-401b643e-7010-476b-8de0-6b3ee8ffe6a2.png)

이전 글에서 저는 위와같은 테스트 환경을 구상했습니다. 이를 구현하기 위해서 2개의 h2를 띄우도록 하였습니다.

![image](https://user-images.githubusercontent.com/33603557/136339426-3c20189a-77c4-4003-8d8b-bf2c260d0f8a.png)

위와 같은 방식으로 read, wirte DataSouce bean을 생성하였습니다(이제보니 write db라는 말이 이상하네요.. 위 그림의 CUD DB가 write db라고 생각해 주시면 감사하겠습니다 🙂)

그리고 상황에 따라 특정 DataSource를 사용하도록 하기 위해서 `LazyConnectionDataSourceProxy` 을 활용했습니다.

![image](https://user-images.githubusercontent.com/33603557/136339469-c9b501c1-5e98-405f-ad7b-5aa32e702526.png)

`LazyConnectionDataSourceProxy` 은 JDBC Connection을 가지고 오는 시점까지 DataSource의 사용을 지연시킵니다. 

그렇다면, JDBC Connection을 가지고 오는 순간 어느 DataSource를 사용할것인지에 대한 선택방법만 정의해 주면 됩니다.

이는 `AbstractRoutingDataSource` 를 상속받은 클래스를 이용해 구성할 수 있습니다.

![image](https://user-images.githubusercontent.com/33603557/136339492-3cac13b4-bae8-4bec-95cf-4957e6e1fde1.png)

위와 같은 방식이죠, determinCurrentLookupKey에 DataSource 선택 전략을 작성해 주시면 되는데요, 저는 DataSourceSelecor라는 객체를 만들어서 이를 정의했습니다.

![image](https://user-images.githubusercontent.com/33603557/136339527-7779aa1f-fe63-4d39-bbdb-77c745a68b73.png)

toWrite()를 호출하면 selected필드를 write로, toRead()를 호출하면 selected 필드를 read로 변경하도록 하였고 getSelected()를 통해 상태를 가지고 올 수 있도록 하였습니다.

> 문자열을 return하는데 어떻게 DataSource가 선택되는지 궁금하실텐데 이에 대한 자세한 내용은 처음 알려드린 링크를 참조해 주세요!
> 

이때, DataSourceSelector가 Component로 지정되어 있기 때문에 다른 객체에서 이 객체를 DI받아 자신이 사용할 DataSource를 선택할 수 있게 됩니다.

![image](https://user-images.githubusercontent.com/33603557/136339558-c338fa3a-d433-4797-a59c-1847ab6fd41c.png)

저같은 경우는 AcceptanceTest 추상화 클래스에 DataSourceSelector를 DI받고 위임 메서드를 만들어 인수 테스트 클래스에서 toRead(), toWrite()를 지정할 수 있게 하였습니다. 이를 통해 조회 관련 테스트의 경우 toRead(); 메서드를 호출하여 read DB를 사용할 수 있도록 하였습니다.

또한 DB는 WriteDB가 기본으로 선택되도록 하였습니다.

![image](https://user-images.githubusercontent.com/33603557/136339572-f655fb44-2b87-4349-9696-e939d1fea0f6.png)

![image](https://user-images.githubusercontent.com/33603557/136339587-1c1c3a6d-d4f6-432f-81ea-3d9a7599b2e9.png)

DataBase clear로직은 dataSourceSelector가 Write 상태일때만 동작하게 하였으며, 모든 인수 테스트의 `@AfterEach` 에 toWrite()를 실행하도록 하여 각 테스트가 종료될 떄 마다 WriteDB를 사용하도록 다시 셋팅시켰습니다. 이렇게 한 이유는, 테스트의 실행 순서는 일정하지 않기 때문에 어느 시점에는 조회 관련 테스트가 돌아가고 어느 시점에서는 업데이트 관련 테스트가 돌아갈 수 있기 때문입니다. 따라서 조회 테스트시에만 readDB를 사용할것이다! 라고 명시시키고 다른 테스트에서는 기본적으로 WriteDB를 이용하도록 한 것 입니다.

### Table 셋팅

이제 조회 관련 테스트와 그 외 테스트가 서로 다른 DB를 사용할 수 있게 되었습니다.

여기서 문제가 발생하는데요, 테스트를 사용할때는 테이블을 자동으로 생성하기 위해 보통 Hibernate가 제공해주는 ddl-auto기능을 많이 사용하게 됩니다. 문제는 이 기능을 사용하게 되면 제가 기본으로 잡아놓은 WriteDB에만 테이블이 생성되고 ReadDB에는 테이블이 생성되지 않아 조회 관련 테스트가 다 실패하게 됩니다.

테이블이 자동 생성되는 시점은 `EntityManagerFactory` 가 build되는 시점입니다. `EntityManagerFactory` 를 빌드할때 ddl-auto 관련 속성을 명시하면 내부에서 Dialect가 쿼리를 만들어 DataSource를 통해 테이블을 생성하게 됩니다.

이때, DataSource에 제가 기본으로 설정한 DataSource가 WriteDB였고, EntityManagerFacotry 내부에서 Table을 정의하는 부분을 제가 컨트롤 할 수 없는것에 문제가 있었습니다. 때문에 ReadDB에 테이블을 정의할 방법을  생각하게 되었습니다.

![image](https://user-images.githubusercontent.com/33603557/136339613-08b7de75-295e-4d71-ab8d-6778f51a8269.png)

그래서 결국 만든것이 SchemaGenerator라는 녀석입니다. 당연히 Hibernate 구현체 종속적으로 구현되어 있습니다. 간단히 말하면, 프로젝트의 Entity들을 전부 파싱해서 DDL을 만들어서 String으로 반환시켜주는 역할을 합니다. *이 소스에 대한 설명은 [링크]를 참조해 주세요!*

이를 통해 DDL을 만들어줄 수 있었기 때문에 더이상 ddl-auto를 사용하지 않고 수동을 테이블을 정의할 수 있게 되었습니다. 그래서 과감하게 application-test.yml에서 ddl 자동생성 부분을 none으로 변경해 주었습니다.

![image](https://user-images.githubusercontent.com/33603557/136339630-861d34bd-3e45-48fa-8271-77d313a68564.png)

이제 테이블을 셋팅 시켜보겠습니다.

![image](https://user-images.githubusercontent.com/33603557/136339646-8fed96fd-674b-4d53-853d-17a65106f0c9.png)

제일 첫번째 글에서 table이름들을 추출했던 부분입니다(`DatabaseCleaner` 객체를 기억 하시나요 ㅎ). 이 컴포넌트는 `InitializeingBean` 를 상속하고 있기 때문에 ApplicationContext가 처음 초기화되는 시점에 `afterPropertiesSet()` 을 실행시키게 됩니다. 여기서 각 DB에 table을 create시켜줄 것 입니다. 이떄, `ddl-auto: create` 와 동일한 효과를 주기 위해서 처음에는 drop table을 진행하고 그다음 create table을 진행하도록 하였습니다. read db는 컨텍스트가 변경되더라도 상태를 유지할 수 있게 하도록 drop을 시키지 않았습니다. toRead()와 toWrite()가 무슨 함순지는 알고 계시니 어떤식으로 로직이 돌아가는지 이해 하실거라 생각합니다!

readDB쪽에 `SQLSyntaxErrorException` 을 잡아놓은 이유는, 사실 저의 귀차니즘에 가깝지만... db에 테이블이 이미 존재하면 create()과정을 진행하지 않도록 하는 로직을 짜야하는데... 귀찮아서 이미 존재하는 테이블을 create하면 에러난다는 점을 이용해서.....(ㅈㅅ)

어찌되었든 이런식으로 두 데이터베이스에 모두 테이블을 셋팅시킬 수 있었습니다!!

# 결과!!!

드디어 대망의 테스트 속도 결과입니다!!! (두구두구두구두구)

![Animation](https://user-images.githubusercontent.com/33603557/136339751-6bad80b5-97e0-40d8-aaf2-c5e26b38ad7b.gif)

![Animation2](https://user-images.githubusercontent.com/33603557/136339791-24dc350e-0583-4edf-b380-5080876c7044.gif)

비교군은, DirtiesContext를 제거한 테스트와 비교하였습니다. DirtiesContext가 존재하는 경우는 너무 느려서 제외했습니다... 

뭐... 말이 필요할까요.. 엄청난 차이가 발생합니다..!

![image](https://user-images.githubusercontent.com/33603557/136339831-90707cf5-002a-445a-8e5c-a17f669e6eb1.png)

![image](https://user-images.githubusercontent.com/33603557/136339851-bb17cad8-5319-4d25-8f08-ebcfd70187ce.png)

*테스트 코드의 개수가 다른 이유는 각 기법을 적용하는 사이에 테스트코드의 정리가 일어나서입니다ㅜ* 

지표상으로는 7초의 차이가 발생했지만, GIF를 보시면 알수 있듯 실제로는 더 많은 속도차이가 발생합니다!

### 단점

그러면 이 3번째 방법은 무조건적으로 좋을까요? 아닙니다. 이 방법을 적용하는데에는 몇가지 단점이 존재합니다. 

- 조회 관련 테스트를 작성하는 개발자가 DB의 상태를 알고 있어야 한다
    - 조회 관련 테스트의 경우는 미리 정의된 데이터들을 이용하여 테스트를 진행합니다. 따라서 이전 글에서 그린 그림과 같이 Fixture들이 어떤 식으로 정의되어 있는지 이해한 상태에서 코드를 작성해야하는 어려움이 있습니다.(하지만 적절한 문서화를 통해 해소할 수 있겠죠?)
- 새로운 도메인이 추가되었을 때 조회 테스트를 위한 환경을 조심히 구성해야 한다.
    - 새로운 요구사항이 생기고, 그에 따라 새로운 도메인이 추가되거나 삭제될 경우 다른 조회테스트에 영향이 갈지 안갈지를 잘 생각해서 테스트 환경을 새로 구성해야 합니다.
    

그렇다면 이 방법을 적용하는건 실질적으로 불가능 할까요? 저는 아니라고 생각합니다.

요구사항이 많이 일어나고 변화가 많은 프로젝트라면 적용하기 힘들겠지만 리팩토링을 진행해야하는 레거시 프로젝트나 작은 규모의 프로젝트에서 적용되면 큰 효과를 얻을 수 있을 것이라 생각됩니다. 빠른 테스트는 리팩토링에 다가가는 첫 걸음 이니까요🙂

이를 통해 깃-들다의 테스트 코드 최적화 여행기를 모두 마무리 하였습니다. 조금이라도 얻어가는게 있었으면 하는 마음이네요.. ㅎㅎ 긴 글 읽어주셔서 감사합니다. 깃들다팀의 손너잘이었습니다.
