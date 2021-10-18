---
layout: post
title: 서비스 성능 개선 - MySQL Optimizer 실행 계획 분석을 기반으로
date: 2021-10-18 13:00::38 +0900
description: 실행 계획 분석을 통해 서비스의 성능을 개선해보자.
img: performance.jpg
tags: [백엔드, MySQL, ElasticSearch]
---

## 1. 들어가며

안녕하세요, 케빈입니다. 현재 운영 중인 Pick-Git 서비스는 Spring Data JPA를 활용 중입니다. 개발 초기에는 서비스를 빠르게 개발하는 것이 급선무다 보니, 쿼리의 성능을 크게 고려하지 않았습니다. 기껏해야 트랜잭션 내부에서 실행되는 쿼리 로그를 보고 N + 1 문제가 있는지 등을 체크하는 정도에 불과했습니다. N + 1 문제는 Fetch Join 및 Batch Fetch Size 옵션 설정을 통해 어느정도 해소할 수 있었는데요.

어느 정도 필수적인 서비스 기능들이 완성되고부터는 서비스의 안정적인 운영에 관심이 많아졌습니다. nGrinder 및 K6 등을 통해 부하 테스트를 진행하면서 서비스의 트래픽 가용성을 확인할 수 있었는데요. DB Scale-Up이 불가능한 상황이라 DB Replication을 통한 Scale-Out을 통해 성능을 대폭 개선할 수 있었습니다. 그러나 여전히 트래픽 부하가 심해지면 다음과 같은 문제들이 여전히 존재했습니다.

* DB 서버의 CPU Utilization이 100%에 육박합니다.
* TPS가 급격하게 하락합니다.
* HikariCP Deadlock이 종종 발생합니다.

SNS 서비스 특성상 조회 트랜잭션이 빈번하게 발생하기 때문에, 일부 조회 요청은 Redis를 도입해 성능을 향상시킬 수 있겠지만 이 또한 미봉책에 불과합니다. 근본적으로 쿼리 성능을 향상시키는 것이 중요합니다. Optimizer 실행 게획 분석을 기반으로 쿼리 튜닝을 진행함으로써 서비스 성능을 개선한 사례를 공유하고자 합니다.

<br>

## 2. 사전 작업

운영 환경과 최대한 동일하게 구성한 테스트 환경을 구축하고, 테스트용 DB에 대량의 더미 데이터를 삽입했습니다. 테스트 데이터 셋의 구성은 다음과 같습니다.

* 게시물 100만건
* 댓글 100만건
* 유저 20만건
* 태그 10만건
* WAS 로드 밸런싱 및 DB Replication 적용

이후 팀원 코다가 다음 항목들에 대해 MariaDB 옵션을 설정했습니다.

* [Query Cache OFF](https://lynmp.com/en/article/ml811c9dc506)
* [Duration이 1초 이상 걸리는 Slow Query 로깅](https://zetawiki.com/wiki/MySQL_%EC%8A%AC%EB%A1%9C%EC%9A%B0_%EC%BF%BC%EB%A6%AC_%EB%A1%9C%EA%B7%B8_%EC%84%A4%EC%A0%95)

MySQL 8.0부터는 Query Cache 기능이 사라졌습니다. 실제 운영 DB의 경우 Query Cache 기능을 사용하고 있지만, 더 상세한 성능 측정을 위해 테스트 DB는 Query Cache 옵션을 끄도록 했습니다.

<br>

## 3. 홈 피드 조회

> PostRepository.java

```java
@Query("select p from Post p left join fetch p.user order by p.createdAt desc")
List<Post> findAllPosts(Pageable pageable);
```

홈 피드 조회 비회원 유저의 홈 피드 게시물 조회 JPQL 쿼리입니다. 팀원 코다가 [홈피드 조회 성능 테스트 경험기_2](https://2021-pick-git.github.io/project-pickgit-homefeed-performance-test-2/) 글에서 홈 피드 조회에 대한 쿼리 튜닝 경험을 이미 공유한 상태인데요. 이번 절에서는 팀원 코다가 정리한 자료를 짧게 요약해보겠습니다. (코다 감사!)

> SQL

```sql
select
    post0_.id as id1_8_0_,
    user1_.id as id1_14_1_,
    post0_.content as content2_8_0_,
    post0_.created_at as created_3_8_0_,
    post0_.github_repo_url as github_r4_8_0_,
    post0_.updated_at as updated_5_8_0_,
    post0_.user_id as user_id6_8_0_,
    user1_.description as descript2_14_1_,
    user1_.image as image3_14_1_,
    user1_.name as name4_14_1_,
    user1_.company as company5_14_1_,
    user1_.github_url as github_u6_14_1_,
    user1_.location as location7_14_1_,
    user1_.twitter as twitter8_14_1_,
    user1_.website as website9_14_1_
from
    post post0_
left outer join
    user user1_
        on post0_.user_id=user1_.id
order by
    post0_.created_at desc limit 0, 10;
```

![image](https://user-images.githubusercontent.com/63405904/137449059-6b18bcad-3794-41b9-b7e2-6bc42ee7a971.png)

옵티마이저 실행 계획을 확인한 결과, 게시물 100만건 Table Full Scan하고 Filesort를 진행하고 있었습니다. 게시물을 최신순으로 정렬하고 상위 10개를 가져오기 때문입니다.

![image](https://user-images.githubusercontent.com/63405904/137443674-1307b8ca-3cfa-41c8-ba80-c0e1adf39e12.png)

슬로우 쿼리 로그에는 쿼리 수행 시간이 1.5초에서 3초까지 소요되었습니다.

![image](https://user-images.githubusercontent.com/63405904/137443204-86fd6f1b-1f46-4613-bb31-00dc4091bb68.png)

부하 테스트를 진행할 때 ``vmstat``으로 WAS 및 DB 서버 상태를 검사한 결과입니다. 상단의 2개는 WAS이며 하단의 2개가 조회를 위한 DB Slave 서버입니다. DB 서버의 CPU Utilization(us) 및 실행 대기 프로세스의 수 (r)이 매우 높게 측정되는 것을 볼 수 있습니다.

DB에 병목이 심한데 Scale-Up을 진행할 수 없는 상황인만큼 쿼리 최적화가 시급합니다.

![image](https://user-images.githubusercontent.com/63405904/137449278-a84af99e-42d5-4ada-86aa-cc3c9c05a2fb.png)

팀원 코다가 ``created_at`` 컬럼에 인덱스를 걸고 다시 실행 계획을 확인한 결과, 이번에는 Table Full Scan없이 Index Full Scan을 통해 게시물을 조회합니다. Filesort도 사라졌으며 쿼리의 Duration 또한 3초에서 0.0094초로 대폭 하락했습니다.

![image](https://user-images.githubusercontent.com/56240505/137612612-1243258f-7df5-40c2-9989-f80b74ca1377.png)

과거 게시물이 35만건일 때 VUser 600명이 10분간 홈 피드를 조회하는 부하 테스트를 진행한 적이 있었는데요. 이 당시 TPS가 평균 30을 기록했습니다. 현재는 게시물이 100만건이니 인덱스를 걸지 않는다면 이보다 성능이 더 나쁘게 측정될 것입니다.

![image](https://user-images.githubusercontent.com/56240505/137612845-2f566f6f-0503-44f2-9bb4-c7224fbcaf18.png)

이번에는 게시물이 100만건일 때 VUser 500명이 10분간 홈 피드를 조회하는 부하 테스트를 진행했습니다. TPS가 평균 300대로 매우 높게 측정되는 것을 확인할 수 있었습니다.

![image](https://user-images.githubusercontent.com/63405904/137449561-613620d9-d87a-4433-a626-aca8ea74ebcf.png)

테스트 DB 서버의 대기 중인 프로세스의 수 및 CPU Utilization 또한 많이 개선되었습니다.

### 3.1. Pagination의 늪

성능이 개선되어서 팀원 모두들 흐뭇(?)했었는데, 팀원 코다가 다음과 같은 이슈를 제기했습니다.

> 조회 쿼리 실행할 때 Page 크기가 특정 범위를 초과하면 다시 Full Table Scan하네요?

![image](https://user-images.githubusercontent.com/56240505/137614141-0eeebdc1-be5f-4be2-a406-592a78d7f6d9.png)

일단 다시 MySQL Workbench를 실행해서 실행 계획을 확인해봅시다.

> SQL

```sql
select
    post0_.id as id1_8_0_,
    user1_.id as id1_14_1_,
    post0_.content as content2_8_0_,
    post0_.created_at as created_3_8_0_,
    post0_.github_repo_url as github_r4_8_0_,
    post0_.updated_at as updated_5_8_0_,
    post0_.user_id as user_id6_8_0_,
    user1_.description as descript2_14_1_,
    user1_.image as image3_14_1_,
    user1_.name as name4_14_1_,
    user1_.company as company5_14_1_,
    user1_.github_url as github_u6_14_1_,
    user1_.location as location7_14_1_,
    user1_.twitter as twitter8_14_1_,
    user1_.website as website9_14_1_
from
    post post0_
left outer join
    user user1_
        on post0_.user_id=user1_.id
order by
    post0_.created_at desc limit 5000, 10;
```

<img width="1097" alt="스크린샷 2021-10-17 오후 3 20 45" src="https://user-images.githubusercontent.com/56240505/137614295-82803936-59e1-4e7f-8916-5c020db562b2.png">

OFFSET에 해당하는 부분을 5000으로 높이니 인덱스를 타지 않고 Table Full Scan, Filesort를 사용하고 있습니다. 아울러 쿼리 수행 시간이 다시 나빠지기 시작했습니다.

![image](https://user-images.githubusercontent.com/56240505/137618326-21720224-c645-451e-9210-8306f41ac079.png)

OFFSET을 500000으로 높인 결과입니다. 즉, OFFSET 크기와 조회 쿼리 성능이 반비례함을 알 수 있습니다.

> TestRunner.groovy

```groovy
@Test
public void test() {
  long randomId = Math.abs(random.nextInt() % 99999) + 1
  HTTPResponse response = request.GET("https://{test-api-address}/api/posts?page=" + randomId + "&limit=10", params)

  if (response.statusCode == 301 || response.statusCode == 302) {
    grinder.logger.warn("Warning. The response may not be correct. The response code was {}.", response.statusCode)
  } else {
    assertThat(response.statusCode, is(200))
  }
}
```

기존의 테스트 스크립트는 ``page=10`` 쿼리 파라미터를 고정해서 사용했었습니다. 이번에는 높은 범위의 난수를 생성해서 API 요청을 보냈습니다.

![image](https://user-images.githubusercontent.com/56240505/137614492-da7e3bc4-4a9c-497c-b0bf-8fcc27cdf0ec.png)

테스트를 시작한지 1분쯤 되니 테스트가 중단되었습니다. 에러율이 50%를 초과했기 때문인데요. TPS가 극악에 달할 정도로 낮은 것을 확인했습니다.

![image](https://user-images.githubusercontent.com/56240505/137614376-5a509b7f-e31d-473d-b8c0-a53d66de5210.png)

에러 로그를 보니 조회 쿼리를 수행하는데 너무 많은 시간이 소요되어 대기 중이던 트랜잭션 워커 쓰레드가 Connection을 얻지 못해 예외를 발생시키고 있습니다. HikariCP Deadlock 문제가 발생하는 중입니다.

![image](https://user-images.githubusercontent.com/56240505/137614475-2086f17d-cf05-4724-811e-2497a0fde5c2.png)

기존에 ``page=10`` 일때는 인덱스를 적용하지 않아도 쿼리를 수행하는데 1초 ~ 3초 정도 소요되었습니다. 그러나 ``page=600,000`` 혹은 ``page=900,000``과 같은 같이 OFFSET이 높은 경우 수행 시간 10초 이상의 쿼리들이 빈번했습니다. 최대 18초가 소요된 쿼리도 보이네요.

### 3.2. 문제 원인

> SQL

```sql
select * from post order by id limit 150000, 10;
```

해당 쿼리는 150,010개의 행을 ID로 정렬한 다음 그 중 처음 10개 행을 반환하겠다는 의미입니다. 실질적으로 10개의 레코드가 필요하지만, 그 과정에서 150,000개에 달하는 개수의 레코드를 카운팅하게됩니다.

일반적으로 테이블은 레코드를 정렬해주는 인덱스가 있다면 Filesort를 사용하지 않습니다. 그런데 정렬된 데이터를 조회하는데 쿼리가 많게는 10초 이상이 걸린다는 것은 확실히 문제가 있어 보입니다.

![image](https://user-images.githubusercontent.com/56240505/137615067-4ad57c8a-0850-4cc2-98ca-deec02fc7e8c.png)

인덱스는 논리적인 개념에서 테이블의 일부분이지만 물리적으로는 테이블과 별도로 존재하는 개체입니다. 대표적으로 두 가지 정보를 가지고 있습니다.

* Index Key : 인덱스가 생성된 컬럼들입니다.
* Table Pointer : 특정 레코드가 나타내는 행을 고유하게 지칭하는 어떠한 값을 의미합니다.
  * InnoDB의 경우 PK 값입니다.

인덱스는 B-Tree 자료구조에 저장되며 이를 바탕으로 ref 및 range 검색을 통해 데이터를 빠르게 조회합니다. 그러나 인덱스 자체는 테이블의 모든 정보를 담고 있지 않습니다. 순서를 유지하며 실제 테이블 값을 찾기 위해서, 인덱스와 테이블이 연결되어야 합니다. 즉, DB 엔진은 행 포인터를 기반으로 각 인덱스 레코드에 상응하는 테이블 레코드를 찾아 인덱싱이 되지 않은 모든 데이터를 반환해야합니다.

![image](https://user-images.githubusercontent.com/56240505/137616242-4b25a0ca-b7bb-49a0-b4f2-20170de423d4.png)

테이블 레코드를 상응하는 인덱스 레코드와 연결하는 행위를 **Row Lookup**이라고 합니다. 위 사진에서 인덱스와 테이블을 연결하는 구불구불한 화살표들입니다.

인덱스 레코드와 테이블 레코드는 메모리 상 및 디스크 상 모두 서로 떨어져 개별적으로 존재합니다. Row Lookup은 Page나 Cache가 점점 더 적중하지 못하고 Miss하는 상황을 유발합니다. 또한 디스크는 순차 접근을 통해 탐색을 할 것이고 그 결과 많은 비용이 발생합니다. 위 사진에서 모든 연결들을 순회하는데 많은 시간이 소요됩니다.

### 3.3. 해결

그런데 LIMIT 및 OFFSET 등을 사용할 때 이러한 Row Lookup 작업이 정말 필요할까요? 만약 ``LIMIT 8, 2`` 쿼리라면 인덱스를 사용해 앞선 8개의 값은 무시하고 남은 2개만 반환하면 될 일입니다.

![image](https://user-images.githubusercontent.com/56240505/137617871-f9d217e8-9cbf-4c82-862a-751fd98c136c.png)

이러한 방법을 **Late Row Lookup**이라고 합니다. DB 엔진은 정말 다른 방법이 없을 때 Row Lookup을 진행합니다. 만약 인덱스 필드들을 기반으로 행을 필터링하는 방법이 존재한다면, 실제 MySQL Table을 Lookup하기 전 수행됩니다.

> SQL

```sql
select * from post where id > 150000 order by id limit 10;
```

이처럼 인덱스를 사용할 수 있는 WHERE 구문을 통해 먼저 데이터를 필터링하면 150,000개의 Row는 무시하고 LIMIT에 걸린 10개의 Row에 대해서만 Row Lookup이 발생합니다. 따라서 Pagination을 처리할 때 적절히 WHERE 조건을 사용해주면, 상기의 문제를 극복할 수 있습니다.

현재 운영 중인 서비스 중 비회원의 홈 피드 조회는 게시물을 최신순으로 보여줍니다. 즉, 날짜를 기반으로 내림차순 정렬하는데요. 따라서 프론트엔드 측에서 다음 페이지를 갱신할 때, 이전 페이지에서 가장 하단에 위치한 게시물의 날짜를 백엔드로 전달하고 이를 WHERE 조건으로 함께 사용합니다.

> SQL

```sql
select
    post0_.id as id1_8_0_,
    user1_.id as id1_14_1_,
    post0_.content as content2_8_0_,
    post0_.created_at as created_3_8_0_,
    post0_.github_repo_url as github_r4_8_0_,
    post0_.updated_at as updated_5_8_0_,
    post0_.user_id as user_id6_8_0_,
    user1_.description as descript2_14_1_,
    user1_.image as image3_14_1_,
    user1_.name as name4_14_1_,
    user1_.company as company5_14_1_,
    user1_.github_url as github_u6_14_1_,
    user1_.location as location7_14_1_,
    user1_.twitter as twitter8_14_1_,
    user1_.website as website9_14_1_
from
    post post0_
left outer join
    user user1_
        on post0_.user_id=user1_.id
where
    post0_.created_at < ${'foo-date'}
order by
    post0_.created_at desc limit 10;
```

![image](https://user-images.githubusercontent.com/56240505/137623039-de8a5e32-0a7f-4080-a309-1d0608f96fc8.png)

조회 쿼리 성능이 다시 향상되었으며, Full Table Scan 및 Filesort를 더이상 사용하지 않습니다.

그런데 날짜를 WHERE 조건에 사용하는 것이 다소 불편하다고 느꼈습니다. 대부분의 경우 ID PK 데이터 순서와 created_at 데이터 순서가 동일합니다. 따라서 프론트엔드 측에서 다음 페이지를 갱신할 때, 이전 페이지에서 가장 하단에 위치한 게시물의 날짜가 아닌 ID를 전달해서 쿼리에 사용할 수 있습니다.

> SQL

```sql
select
    post0_.id as id1_8_0_,
    user1_.id as id1_14_1_,
    post0_.content as content2_8_0_,
    post0_.created_at as created_3_8_0_,
    post0_.github_repo_url as github_r4_8_0_,
    post0_.updated_at as updated_5_8_0_,
    post0_.user_id as user_id6_8_0_,
    user1_.description as descript2_14_1_,
    user1_.image as image3_14_1_,
    user1_.name as name4_14_1_,
    user1_.company as company5_14_1_,
    user1_.github_url as github_u6_14_1_,
    user1_.location as location7_14_1_,
    user1_.twitter as twitter8_14_1_,
    user1_.website as website9_14_1_
from
    post post0_
left outer join
    user user1_
        on post0_.user_id=user1_.id
where
    post0_.id < ${'foo-post-id'}
order by
    post0_.created_at desc limit 10;
```

![image](https://user-images.githubusercontent.com/56240505/137623347-a8b76fc7-b1b9-4ef6-ac21-da907bbfea3a.png)

WHERE 조건을 사용하지 않던 때보다는 쿼리 성능이 향상 되었으나, WHERE 조건에 날짜를 사용하는 방식보다는 현저히 느립니다. WHERE 조건에 날짜를 사용하는 경우 INDEX Range Scan을 사용하지만, ID를 사용하니 INDEX Full Scan을 사용하고 있습니다. 현재 WHERE 조건과 ORDER 조건에서 사용 중인 인덱스가 다르기 때문입니다.

SQL 문장이 WHERE 절과 ORDER BY 절을 가진다면, WHERE 조건은 A 인덱스를 사용하고 ORDER BY는 B 인덱스를 사용하는 것은 불가능합니다. WHERE 절과 ORDER BY 절이 같이 사용된 쿼리 문장은 다음 중 한 가지 방법으로 인덱스를 사용합니다.

* WHERE 절과 ORDER 절 모두 동시에 같은 인덱스를 사용하기.
  * WHERE 절의 비교 조건에 사용하는 칼럼과 ORDER BY 절의 정렬 대상 칼럼이 모두 하나의 인덱스에 연속해서 포함되어 있을 때 이 방식으로 인덱스를 사용할 수 있습니다.
* WHERE 절의 인덱스만 사용하기.
* ORDER 절의 인덱스만 사용하기.

현재 Index Full Scan을 사용하지 않기 위해서는 WHERE + ORDER 컬럼을 모두 포함하는 ``INDEX (ID, CREATED_AT)``를 새로 만들거나 어느 한 쪽의 인덱스를 사용하도록 WHERE 절이나 ORDER 절의 컬럼을 수정해야합니다. 편의를 위해 ORDER 절도 ID를 사용하도록 수정해보겠습니다.

> SQL

```sql
select
    post0_.id as id1_8_0_,
    user1_.id as id1_14_1_,
    post0_.content as content2_8_0_,
    post0_.created_at as created_3_8_0_,
    post0_.github_repo_url as github_r4_8_0_,
    post0_.updated_at as updated_5_8_0_,
    post0_.user_id as user_id6_8_0_,
    user1_.description as descript2_14_1_,
    user1_.image as image3_14_1_,
    user1_.name as name4_14_1_,
    user1_.company as company5_14_1_,
    user1_.github_url as github_u6_14_1_,
    user1_.location as location7_14_1_,
    user1_.twitter as twitter8_14_1_,
    user1_.website as website9_14_1_
from
    post post0_
left outer join
    user user1_
        on post0_.user_id=user1_.id
where
    post0_.id < ${'foo-post-id'}
order by
    post0_.created_at desc limit 10;
```

![image](https://user-images.githubusercontent.com/56240505/137623472-d95c3feb-df9e-4d7a-93a1-8cc59c255ebc.png)

WHERE 절과 ORDER 절 모두 ID를 사용하니 다시금 Index Range Scan을 하며 성능이 향상되었습니다.

![image](https://user-images.githubusercontent.com/56240505/137632838-76626ce0-ef49-492b-96b8-c39f2e67d0be.png)

nGrinder를 통해 부하 테스트를 진행했습니다. 난수 발생을 통해  ``page=600000``과 같은 극단적인 요청을 지속적으로 발생시켰지만, 평균 TPS가 300대로 유지되는 등 Pagination 조회 성능이 크게 하락하지 않는 것을 확인할 수 있습니다. 😊😊

<br>

## 4. 유저 검색

> UserRepository.java

```java
@Query("select u from User u where u.basicProfile.name like %:username%")
List<User> searchByUsernameLike(@Param("username") String username, Pageable pageable);
```

> SQL

```sql
select
    user0_.id as id1_14_,
    user0_.description as descript2_14_,
    user0_.image as image3_14_,
    user0_.name as name4_14_,
    user0_.company as company5_14_,
    user0_.github_url as github_u6_14_,
    user0_.location as location7_14_,
    user0_.twitter as twitter8_14_,
    user0_.website as website9_14_
from
    user user0_
where
    user0_.name like ? limit ?
```

현재 사용 중인 유저 검색 방법은 여러 문제가 존재합니다.

1. 앞서 언급한 Pagination OFFSET 문제가 존재하기 때문에 WHERE에 적절한 조건이 추가되어야 합니다.
2. **유저를 이름으로 조회할 때 LIKE 연산자를 이용하는데, % 기호를 앞에 넣어 사용하면 유저 이름 인덱스를 사용하지 못합니다.**

![image](https://user-images.githubusercontent.com/56240505/137668813-bf956dc0-e2a6-4bd9-b891-4d7899c35dbe.png)

따라서 해당 쿼리를 실행하면 Table Full Scan이 발생합니다.

![image](https://user-images.githubusercontent.com/56240505/137668936-6fd70a7a-702a-4ec8-8a07-e1b245357824.png)

회원이 20만건일 때 VUser 500명이 10분간 회원 이름을 검색하는 부하 테스트를 진행했습니다. 평균 TPS는 38.9입니다.

![image](https://user-images.githubusercontent.com/56240505/137669129-08e4048f-7977-47ff-854d-10a25887944f.png)

테스트를 진행할 때 DB 서버의 상태를 측정해보니 대기 중인 프로세스의 수 및 CPU Utilization이 매우 높습니다. 현재 테스트는 심지어 이전 절처럼 Pagination OFFSET을 높게 준 것이 아니라 ``page=0``으로 매우 낮게 주었습니다. 즉, 1번 문제는 쉽게 처치할 수 있으나 궁극적인 원인은 2번 문제입니다. 팀 회의를 통해 도출한 대안은 다음과 같습니다.

1. 인덱스를 타기 위해 ``LIKE :username%``과 같이 앞에서 사용하는 % 기호를 삭제하기.
2. 쿼리 튜닝을 포기하고 검색 요청의 경우 Elastic Search를 통해 처리하기.

1번 방법은 클라이언트가 입력한 문자열로 시작하는 유저 이름만을 검색하기에 사용자 경험 측면에서 부적절하다고 판단했습니다. 따라서 2번 방법을 채택하기로 결정했습니다.

물론 Elastic Search라는 별도의 DB를 사용함으로써 고려해야할 관리 포인트가 많아진다는 단점이 존재합니다.

* RDB와의 데이터 정합성을 제대로 유지해야합니다.
* ES는 읽기 성능이 우수하지만 쓰기 성능이 나쁜 만큼, 업데이트 방식에 대한 추가적인 학습이 필요합니다.
  * 실시간으로 처리할 것인지 배치 방식으로 처리할 것인지 등.
* 고가용성을 위한 단일 장애점 극복을 위해 ES 클러스터링 및 샤딩 등 아키텍쳐에 대한 고려도 필요합니다.

그러나 지금으로서는 ES를 도입하는 것이 가장 최선이라 여겨졌습니다.

![image](https://user-images.githubusercontent.com/56240505/137674715-51fcf017-29dd-4f60-bf16-3004f37e1079.png)

ES를 적용한 다음 부하 테스트를 진행했습니다. RDB를 사용할 때와 테스트 전제 조건은 동일합니다. RDB의 경우 DB Replication을 적용해 2대의 Slave DB가 조회를 처리했음에도 불구하고 평균 TPS가 38.9였습니다.

그러나 ES의 경우 단 1대의 서버만으로도 평균 TPS가 82.2를 기록했습니다! 이는 RDB를 사용하는 기존의 방식보다 성능이 2배 이상을 상회합니다. 향후 ElasticSearch 또한 클러스터링 및 샤딩 등을 적용하면 더 높은 성능 및 고가용성을 얻을 수 있겠다고 느꼈습니다. 😊

<br>

## 5. 마치며

> 성능 개선과 클린코드는 끝을 바라보고 하는게 아니에요.

Optimizer 실행 계획 분석을 통해 팀원들과 의사결정을 진행하고, 쿼리 튜닝 및 별도의 DB 도입을 통해 서비스의 일부 기능들의 성능을 개선했습니다. 그럼에도 불구하고 아직 갈 길이 멀다고 느껴집니다. 여전히 개선이 필요한 쿼리들이 많기 때문입니다.

> SQL

```sql
select
    post0_.id as id1_8_0_,
    user1_.id as id1_14_1_,
    post0_.content as content2_8_0_,
    post0_.created_at as created_3_8_0_,
    post0_.github_repo_url as github_r4_8_0_,
    post0_.updated_at as updated_5_8_0_,
    post0_.user_id as user_id6_8_0_,
    user1_.description as descript2_14_1_,
    user1_.image as image3_14_1_,
    user1_.name as name4_14_1_,
    user1_.company as company5_14_1_,
    user1_.github_url as github_u6_14_1_,
    user1_.location as location7_14_1_,
    user1_.twitter as twitter8_14_1_,
    user1_.website as website9_14_1_
from
    post post0_
inner join
    user user1_
        on post0_.user_id=user1_.id
where
    user1_.id in (
        select
            user3_.id
        from
            follow follow2_
        inner join
            user user3_
                on follow2_.target_id=user3_.id
                and (
                    follow2_.source_id=?
                )
        )
        or user1_.id=?
order by
    post0_.created_at desc limit ?
```

로그인 회원의 경우, 홈 피드를 조회할 때 자기 자신의 게시물 및 팔로잉 중인 유저의 게시물만 조회하는 비즈니스 로직을 가지고 있습니다. 이 부분 또한 페이지네이션이 걸려 있습니다.

![image](https://user-images.githubusercontent.com/56240505/137677642-e3102376-af67-4357-9b24-d868e9036e60.png)

OR 절 때문에 인덱스를 타지 못해 Table Full Scan이 발생할 뿐만 아니라, 임시 테이블 생성 및 Filesort를 사용합니다. 그 결과 쿼리 수행 속도가 배우 느려집니다. OR 절만 삭제해도 인덱스를 사용해 성능이 대폭 향상되는데, 해당 부분을 어떻게 개선할지 팀 차원에서 지속적으로 논의 중입니다. 🧐

> SQL

```sql
select
    distinct post1_.id as id1_8_,
    post1_.content as content2_8_,
    post1_.created_at as created_3_8_,
    post1_.github_repo_url as github_r4_8_,
    post1_.updated_at as updated_5_8_,
    post1_.user_id as user_id6_8_
from
    post_tag posttag0_
inner join
    post post1_
        on posttag0_.post_id=post1_.id cross
join
    tag tag2_
where
    posttag0_.tag_id=tag2_.id
    and (
        tag2_.name in (
            ?
        )
    ) limit ?
```

게시물을 복수 개의 태그를 기반으로 조회하는 쿼리입니다. 게시물과 태그는 N:N 관계를 맺고 있으며, 태그 이름은 유니크 인덱스가 걸려있습니다.

![image](https://user-images.githubusercontent.com/56240505/137678232-50dce594-f0c3-42d0-9e6e-3d92e400c9f1.png)

Index Range Scan을 사용하고 커버링 인덱스가 적용되고 있습니다.

![image](https://user-images.githubusercontent.com/56240505/137678144-c62872b1-d0a8-4147-917c-f13a349b58cc.png)

평균 TPS는 300 이상으로 성능 또한 현재로서는 나름 준수한 편입니다. 그런데 현재 SQL 쿼리에서 Cross Join이 발생하고 있습니다. 성능에 시급한 이슈는 없지만 추후 이러한 부분을 제거할 수 있을지 등 또한 학습 차원에서 팀원들과 논의 중에 있습니다. (다들 쿼리 튜닝이 처음인지라...) 긴 글 읽어주셔서 감사합니다.

<br>

---

## References

* [홈피드 조회 성능 테스트 경험기_1](https://2021-pick-git.github.io/project-pickgit-homefeed-performance-test-2/)
* [홈피드 조회 성능 테스트 경험기_2](https://2021-pick-git.github.io/project-pickgit-homefeed-performance-test-2/)
* [MySQL ORDER BY / LIMIT performance: late row lookups](https://explainextended.com/2009/10/23/mysql-order-by-limit-performance-late-row-lookups/)
* [Real MySQL [7-8] 쿼리 작성 및 최적화 - WHERE, ORDER BY 인덱스 사용](https://weicomes.tistory.com/191)
