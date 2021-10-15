---
layout: post
title: 태그 검색 성능 테스트 경험기_1
date: 2021-10-15 13:32:20 +0300
description: 태그 검색 API에 대한 성능 테스트를 진행하고 병목 지점을 파악해 개선한 경험기 입니다.  
img: performance-test.jpeg
tags: [백엔드, Database, 성능테스트]
---

## 💡 Intro

- 진행 중인 [프로젝트](https://github.com/woowacourse-teams/2021-pick-git)에서 구현한 웹 어플리케이션이 어느 정도의 부하를 견딜 수 있는지에 대한 성능테스트를 진행했다. 
- 프로젝트는 개발자를 타켓으로 한 깃헙 레포지토리를 연동한 게시물을 업로드하여 개발자들이 자신의 작업을 공유하고 다른 이들의 프로젝트를 캐줄얼하게 엿볼 수 있는 SNS형 웹 어플리케이션이다. 
- 사용자는 각 게시물에 관련된 태그를 남길 수 있고 해당 태그를 기반으로 검색하여 관련 게시물을 찾아볼 수 있다. (비로그인/로그인 모두 가능)
- 태그를 통해서 관련 게시물을 검색하는 성능테스트를 진행, 병목 지점을 분석하고 개선하는 과정을 따라가보자. 

<br>
<br>

## 🌩 사전 작업

### 테스트 더미 데이터 입력 
- 테스트를 진행하기 위해서는 실제 운영환경과 최대한 유사한 환경에서 테스트하는 것이 중요하다. 
- 운영환경과 유사한 환경이라고 하면 크게 1) 인프라 구조 2) 데이터 두 가지가 있다. 
- 먼저 대량의 더미 데이터를 입력하도록 한다. (팀원 케빈이 수고해주었다 !! 👍)

```
 * 테스트 데이터 : 게시물 100만 / 유저 20만
 *                 태그 10만 (1개당 게시물 10개)
 *                 댓글 100만 (게시물당 1개)
 *
 * 테스트 용이성을 위해 유저 1명 이름은 tester로 명명해 저장
 * 테스트 용이성을 위해 태그 3개 이름은 java, javascript, spring로 명명해 저장
```

### MariaDB 쿼리 캐시 끄기
- 왜 쿼리 캐시를 껐을까?
  - 실제 어플리케이션에서는 query cache 설정이 켜져있음에도 불구하고 cache 설정을 끈 이유는 실제 환경에서는 많은 유저들이 여러 태그를 검색하여 매번 다양한 쿼리가 실행되지만 테스트 환경에서는 3개의 태그를 랜덤으로 실행하기 때문에 캐시 적중률이 실제 환경보다 높다. 따라서 db 쿼리캐시를 꺼서 최대한 실제 환경과 맞춰주도록 한다. 
  - 참고로 MySQL 8.0 부터는 쿼리 캐시 기능이 꺼져있다고 한다. 
  - 또한 여전히 os 측에서 하는 memory 캐시 영향이 있지만 제어하기 어려운 부분이므로 우선 넘어가도록 한다. 

> 쿼리 캐시 확인

<p align="center"><img width="85%" src="https://user-images.githubusercontent.com/63405904/137438052-8604eb7d-a892-4a2b-9ca2-064ef832743b.png"></p>

<p align="center"><img width="85%" src="https://user-images.githubusercontent.com/63405904/137438572-2c20b02c-9cfb-4556-a9b8-ae362209423b.png"></p>

- MariaDB config 파일에서 cache size를 0으로 설정한다. 
- 이후 `sudo service mysqld restart` 로 DB를 재구동시킨다. 

> 변경 후 적용 확인

<p align="center"><img width="85%" src="https://user-images.githubusercontent.com/63405904/137438718-ba0973ee-3037-4729-b1cb-ef2d2fe43354.png"></p>


### MariaDB slow query 로그 설정하기 
- 오래 걸리는 쿼리에 대한 로그를 남겨 특정 쿼리로 인한 병목이 있는지 확인할 수 있도록 설정한다. 
- 단위는 1초 이상 걸리는 쿼리에 대한 로그를 남기는 것으로 했다. 

> Slow query 적용 중인지 확인
 
<p align="center"><img width="85%" src="https://user-images.githubusercontent.com/63405904/137438832-c3b085d0-8143-4974-8045-829455f6fbe7.png"></p>

- Slow 쿼리 설정 적용 하기
    - slow_query_log = 1 부터 long_query_time 까지 적용
    - 적용 후 `sudo service mysqld restart` 로 재시작
    <p align="center"><img width="85%" src="https://user-images.githubusercontent.com/63405904/137438852-19161fd9-ac7b-4948-bdb3-0df7991bbf6f.png"></p>


> Slow query 설정 후 확인

<p align="center"><img width="85%" src="https://user-images.githubusercontent.com/63405904/137439074-ff7ed9e9-0ff8-454b-b471-1634454ca3bf.png"></p>

- 제대로 적용되었는지 확인하기 위해서 5초 이상 걸리는 쿼리를 실행하고 로그파일 경로의 `mariadb-slow.log`에 대항 쿼리에 대한 로그가 남았는지 확인해보자. 
  
  <p align="center"><img width="85%" src="https://user-images.githubusercontent.com/63405904/137439218-aa74256f-24e3-40c8-9351-03f83fda8a6b.png"></p>

  <p align="center"><img width="85%" src="https://user-images.githubusercontent.com/63405904/137439229-266c8c68-1626-43b6-a481-c99b245c6b20.png"></p>

  <p align="center"><img width="85%" src="https://user-images.githubusercontent.com/63405904/137439238-f496e3bf-41fc-4c2a-afdc-5008d843c634.png"></p>

<br>
<br>

## 🌩 테스트 진행하기 

### 테스트 환경 

테스트를 위해 구축한 테스트 환경은 다음과 같다. 
- WAS 2대가 각각 AWS EC2 Medium 사양으로 실행중이다. 
- AWS EC2 Medium 사양으로 Reverse Proxy가 있으며 Load balancer 역할을 하면 ssl 적용이 되어 있다. 
- 데이터 베이스는 AWS EC2 Medium에 MariaDB로 3대가 연결되어 있다. 
  - Master DB 1개, Slave DB 2대로 replication이 적용되어 있다. 
  
테스트 툴은 K6로 진행한다. 
- AWS EC2 Medium 에 K6 테스트 서버를 구축했다. 
- 왜 K6일까?
  - 사실 팀 차원에서 하는 테스트 툴은 [Ngrinder](https://naver.github.io/ngrinder/#:~:text=nGrinder%20is%20a%20platform%20for,inconveniences%20and%20providing%20integrated%20environments) 이다. 
  - 하지만 AWS 권한 제한으로 인해 controller와 agent를 별도의 EC2로 분리하지 못했다. (그것 때문인지는 모르겠지만 간혹 랜덤하게 K6와 동일한 테스트를 돌렸을 때 결과가 매우 다르게 나올때도 있었다...) ngrinder는 반드시 분리하도록 권장하기 때문에 혹시 모를 영향을 최소화 하기 위해서 나는 K6에서 진행하였다. 
  - 또한 K6는 문서가 굉장히 깔끔하게 잘 되어 있어 스크립트를 짜거나 테스트 설정을 하는 것이 입문자에게 편하다는 장점이 있었다. 

### 테스트 스크립트 및 설정 
- K6는 자바스크립트로 스크립트를 짠다. 
- 부하 테스트는약 10분간 148 명의 vuser로 진행했다. 
  - 본래 30분 이상을 하기를 권장하지만 시간 관계상 10분만 진행하고 빠르게 결과를 분석하기로 했다. 
- 스크립트 
  - 3개의 태그를 랜덤으로 골라 get 요청을 보낸다. 
  - 응답코드가 200 인지 확인한다. 

    ```javascript
    import http from 'k6/http';
    import { URL } from 'https://jslib.k6.io/url/1.0.0/index.js';
    import { check } from 'k6';
    import { randomIntBetween } from "https://jslib.k6.io/k6-utils/1.1.0/index.js";

    export let options = {
    vus: 148,
    duration: '600s',
    };

    export default function () {

    var tags = ['java', 'javascript', 'spring'];

    const url = new URL('https://test-pick-git.o-r.kr/api/posts');

    const index = randomIntBetween(0, 2);

    url.searchParams.append('type', 'tags');
    url.searchParams.append('keyword', tags[index]);
    url.searchParams.append('page', 0);
    url.searchParams.append('limit', 10);

    const res = http.get(url.toString());

    check(res, {
        'is status 200': (r) => r.status === 200,
    });
    }
    ```

- 성능 테스트를 진행하면서 서버의 상태를 관리하기 위해 각각 WAS 2대, DB 2대에 대한 상태를 출력하고 모니터링 했다. 
- `vmstat 1 -Sm` 와 `top` 명령어를 통해 프로세스의 상태, CPU 상태, 스왑 발생 여부, load average 등을 확인했다.  

<br>
<br>

## 🌩 테스트 진행하기 

### 첫번째 테스트 - WAS 오류 
- WAS 및 DB 모니터링: 왼쪽 위 부터 시계 방향으로 `was1` → `was2` → `slaveDB2` → `slaveDB1`

    <p align="center"><img width="85%" src="https://user-images.githubusercontent.com/63405904/137442374-21e53603-b1d6-4505-9290-9bc65dfc1d38.png"></p>

    <p align="center"><img width="85%" src="https://user-images.githubusercontent.com/63405904/137442700-d1806f69-4ac2-4a15-981a-a42d0555f67b.png"></p>

  - WAS2에 대한 CPU idle 비율이 100% 이므로 해당 WAS가 동작하지 않은 것을 알아내었다. 확인해보니 어플리케이션이 종료되어 있었다. 테스트 진행시간이 5분정도 경과되었을 때 was2에 어플리케이션을 띄웠고 테스트는 그대로 계속 진행했다. 

- 테스트 결과 
  
    <p align="center"><img width="85%" src="https://user-images.githubusercontent.com/63405904/137442922-2b3148e4-d243-480e-b442-3ea3a5168eb4.png"></p>

- DB의 경우 OS메모리 캐싱이 되므로 DISK I/O는 발생하지 않았다. 
- 다만 비효율적인 쿼리에 의해 CPU 과부하가 걸리는 것을 확인할 수 있었다.
    - 맨 왼쪽 칼럼 **r**(실행 대기 프로세스 수) 수치가 10 정도로 매우 높다.
    - 본래 r은 CPU 코어 갯수여야 서버가 잘 돌아가고 있다고 판단한다. (현재 ec2 CPU 코어 개수 2개)
- 요청 당 실행 시간(http_req_duration) **13.33 초**로 매우 긴 시간이 소요되기에 개선해야 할 점이 명확히 보였다. 

<br>

### 두번째 테스트 
- WAS 및 DB 모니터링: 왼쪽 위 부터 시계 방향으로 `was1` → `was2` → `slaveDB2` → `slaveDB1`

    <p align="center"><img width="85%" src="https://user-images.githubusercontent.com/63405904/137443197-e4a921d2-f0b5-45b8-a8f1-84ce23bf5c94.png"></p>

    <p align="center"><img width="85%" src="https://user-images.githubusercontent.com/63405904/137443204-86fd6f1b-1f46-4613-bb31-00dc4091bb68.png"></p>

  - 앞 테스트와 동일하게 WAS의 CPU나 I/O 상황은 대체적으로 양호하고 DB 서버에 CPU 과부하가 걸리는 것을 확인할 수 있다. 

- 테스트 결과 
  
    <p align="center"><img width="85%" src="https://user-images.githubusercontent.com/63405904/137443363-66ab8e2e-33a6-4cd5-9e59-72354e93c3fc.png"></p>

- WAS가 2대였음에도 불구하고 error rate이 줄어든 것 밖에 나아진 부분은 없었다. 
    - 요청 실행 시간이나 테스트 갯수 tps 등의 수치가 위와 동일했다. 
- 이것을 통해 알 수 있는 것은 WAS의 성능이 아니라 DB에 의한 성능저하라는 것이다. 

<br>

- 더 명확하게 알아보기 위해 slow query 로그를 확인해 보았다. 
  - 로그를 확인해보니 태그를 검색하고 검색 결과인 게시물을 조회하는 쿼리가 1.5 초 정도 소요되는 것을 확인할 수 있었다. 
  
    <p align="center"><img width="85%" src="https://user-images.githubusercontent.com/63405904/137443674-1307b8ca-3cfa-41c8-ba80-c0e1adf39e12.png"></p>


> 다음 포스트에서 병목이 생기는 DB 쿼리를 진단하고 개선한 후 결과에 대해서 다룬다. 

> 백엔드 코다입니다 🙌
