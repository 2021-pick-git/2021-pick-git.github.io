---
layout: post
title: nGrinder 스트레스 테스트를 통한 어플리케이션 성능 분석 및 개선 사례
date: 2021-09-30 13:32:20 +0300
description: Pick-Git 서비스는 어느 정도의 가용성을 보유하는가?
img: ngrinder.png
tags: [백엔드, nGrinder, Test]
---

## 1. 들어가며

안녕하세요, 케빈입니다. 부하 및 스트레스 테스트는 각 시스템의 응답 성능 및 한계치를 확인하는 수단입니다. 특히, 부하가 많아질 때 나타나는 증상 등은 성능을 개선할 수 있는 단초가 됩니다. nGrinder를 바탕으로 어플리케이션 서비스의 성능 임계점 및 확장성 등을 간편하게 진단할 수 있습니다. 이번 글에서는 실제 Pick-Git 어플리케이션 서비스의 성능을 nGrinder로 분석하고 개선한 사례를 공유하고자 합니다. 만약 nGrinder 구성 방법이 궁금하시다면 [nGrinder 설치 방법 및 파일 업로드 테스트 예제](https://xlffm3.github.io/devops/nGrinder/) 글을 참고하시길 바랍니다.

<br>

## 2. 테스트 계획

### 2.1. 전제 조건

처음 테스트 계획을 수립하는 만큼 미숙한 점이 많습니다. 이 점 양해부탁드립니다. 초기 테스트 전제 조건은 다음과 같이 구성했습니다.

* 1일 예상 사용자(DAU) : 10만명
* 1명당 1일 평균 접속 수 : 15회
* 1일 총 접속 횟수 : 150만회
* 1일 평균 RPS(Request Per Second) : 17회
* 1일 최대 RPS(Request Per Second) : 50회
* Latency : 50 ~ 100ms 이하
* Throughput : 17회~50회

초기 목표 RPS는 약 70회로 잡았으며, 부하 테스트에서 사용할 VUser는 [공식](https://xlffm3.github.io/devops/web-performance-and-stress-test/#53-vuser-%EB%8F%84%EC%B6%9C-%EA%B3%B5%EC%8B%9D)에 의거해 105 정도로 산출했습니다. 스트레스 테스트를 위한 VUser는 300 ~ 600 등 유동적으로 적용했습니다. 아울러 부하 및 스트레스 테스트 모두 30분 ~ 1시간을 진행합니다.

### 2.2. 시나리오 대상

접속 빈도가 높으며, 많은 DB 리소스를 조합해 결과를 보여주는 부분을 시나리오 대상으로 선정했습니다. 대표적으로 홈 피드(게시물) 조회 페이지 및 게시물 작성 기능이 있겠습니다.

### 2.3. 테스트 환경

운영 환경과 동일한 스펙의 Reverse Proxy, WAS, DB 서버 등을 구성했습니다. 테스트 성능은 DB에 많은 영향을 받기 때문에, User와 Post 및 Comment를 각각 20만건 저장해두었습니다. 미처 신경쓰지 못한 결과, 일부 테스트의 경우 누적된 쓰기 작업으로 인해 게시물이 35만건 저장된 상태에서 테스트가 진행되었습니다. 😅

또한 테스트 대상 시스템의 성능을 측정하기 위해서 외부 시스템은 항상 기대한 결과만을 반환하는 환경이 필요합니다. 이 때, 어플리케이션에서 객체를 Mocking하거나 Dummy Controller를 사용하지 않았습니다.

* Http Connection Pool 미사용
* Connection Thread 미사용
* I/O 미발생

등의 문제가 존재하기 때문입니다. 성능 테스트에서 중요한 관점인 Thread 및 리소스 사용을 무시하게 되며, 테스트 시스템의 자원과 리소스를 함께 사용하는 등 테스트의 신뢰성이 떨어집니다.

테스트를 위한 요소는 테스트 대상 시스템에 절대로 영향을 미쳐서는 안 되기 때문에, S3 이미지 업로드 및 GitHub API와 같은 외부 시스템은 테스트 대상 시스템과 완벽히 분리된 Mock 서버를 만들어 배포한 다음 테스트를 진행했습니다.

nGrinder를 활용한 부하 및 스트레스 테스트에 대한 상세한 내용은 우아한형제들 기술블로그의 [결제 시스템 성능, 부하, 스트레스 테스트](https://techblog.woowahan.com/2572/) 글을 참고하시길 바랍니다.

<br>

## 3. nGrinder 부하 테스트

> 홈 피드(게시물) 조회 기능을 테스트한다.

![image](https://user-images.githubusercontent.com/56240505/133959311-04008088-0304-48f2-81e6-9f9b2813565f.png)

VUser 105명으로, 데이터 셋이 적은 경우 TPS가 100 ~ 400으로 좋게 측정되었습니다.

![image](https://user-images.githubusercontent.com/56240505/133959234-815adebd-1920-4524-88b1-071db8270ed5.png)

반면 데이터 셋이 20만건인 경우, TPS가 평균 27로 좋지 못했습니다. 에러율 또한 7.1%에 달했습니다.

<br>

## 3. nGrinder 스트레스 테스트

### 3.1. 단일 DB 서버

Pick-Git의 초기 Infrastructure는 WAS 1대 및 DB 1대 등 단일 구조입니다. 이같은 환경에 대해 스트레스 테스트를 진행했습니다.

테스트에 사용되는 VUser는 총 300명입니다. VUser 150명이 게시물 작성 요청을 보내는 동안, VUser 150명은 홈 피드(게시물) 조회 요청을 보냅니다.

![image](https://user-images.githubusercontent.com/56240505/133959863-633bf591-9a84-474b-b497-700f902c1bdf.png)
![image](https://user-images.githubusercontent.com/56240505/134140694-d5a1fd80-30d4-480c-8761-1c6e286d8a79.png)

* 읽기
  * TPS : 평균 11.8, 최대 22.5
  * MTT : 12,854

![image](https://user-images.githubusercontent.com/56240505/133959921-531572c0-a6c7-46e1-9fe3-082ccb94f5c8.png)
![image](https://user-images.githubusercontent.com/56240505/134140952-b895934e-6aec-4acc-9239-5a50d4dfdf5d.png)

* 쓰기
  * TPS 평균 11.6, 최대 22
  * MTT : 13,067

### 3.2. DB Replication 적용

단일 DB 환경에 대한 스트레스 테스트 진행 후, DB Replication 도입의 필요성을 느꼈습니다. 단일 장애점을 없애고 가용성을 높일 수 있을 뿐만 아니라 성능 측면에서도 도움이 될 것이라 판단했습니다.

DB Replication은 DB 스토리지를 물리적으로 다른 서버에 복제하는 것입니다. 2대 이상의 DB 서버가 동일한 데이터를 담도록 실시간으로 동기화하는 기술인만큼 DB 데이터 백업이 가능합니다.

하나의 Slave Node에 장애가 발생하더라도 다른 Slave Node(다중화)를 통해 서비스가 중단되지 않고 계속 정상 운영됩니다. 특히 Master Node 서버에 장애가 발생해 서버가 중지되더라도, Slave Node 중 1개를 Master Node로 승격시켜 서비스 중단 없이 데이터를 빠르게 복구하는 자동 Failover 기능을 제공합니다.

Write 작업은 Master Node가 담당하고 Read 작업은 Slave Node가 담당함으로써 DB 서버 부하를 분산시킬 수 있습니다. 어플리케이션에서 DB 부하로 인한 병목 현상이 서비스 장애의 주 원인인데, 이를 개선하는데 도움이 됩니다.

이번에는 DB Replication(Master 1대 및 Slave 2대) 적용 후 스트레스 테스트를 진행했습니다.

![image](https://user-images.githubusercontent.com/56240505/133960411-6f618365-94a5-4024-ba9c-66e707c2da1f.png)
![image](https://user-images.githubusercontent.com/56240505/134141122-efc2b69c-32b3-47f3-90df-15a9fe4c76c5.png)

* 읽기
  * TPS : 평균 40.1, 최대 62
  * MTT : 3,789

제 불찰로 인해, 이전 쓰기 테스트로 인해 생성된 게시물을 삭제하지 않고 읽기 테스트를 진행했습니다. 지속적인 쓰기 작업으로 인해 게시물이 35만건으로 누적되는 상태라, TPS가 점진적으로 하락하는 모습을 보입니다.

이러한 점을 고려하더라도 읽기 성능이 기존보다 많이 개선된 것을 볼 수 있습니다. 만약 테스트 초기 게시물이 20만건이었다면 TPS 평균이 40을 훨씬 상회했을 것이라 추측합니다.

* 교훈
  * 처음에는 테스트를 돌릴 때마다, DB를 초기화하고 반복문을 돌며 20만건씩 데이터를 넣는 작업을 진행했습니다.
  * DB Dump를 떠두고 불필요한 반복문 순회 작업을 생략할 수 있다는 사실을 나중에 알았습니다. 😭😭
  * 최초로 DB에 데이터를 넣을 때만 반복문을 순회하며, 이후 DB를 초기화하면 미리 떠둔 DB Dump로 데이터를 넣습니다.

![image](https://user-images.githubusercontent.com/56240505/133960420-9a46bd2c-385d-4308-b194-4521135c239d.png)
![image](https://user-images.githubusercontent.com/56240505/134141277-77dca89a-3339-432d-9573-c718e3293a49.png)

* 쓰기
  * TPS : 평균 85.9, 최대 121
  * MMT : 1,774

확실히 DB Replication을 적용한 결과, 읽기 및 쓰기 성능 모두 드라마틱하게 상승한 것을 확인할 수 있었습니다!

### 3.3. 단일 WAS

DB Replication을 적용했으나 현재 WAS는 Scale-Out 없이 여전히 1대만 운용 중입니다. 해당 환경에 대해서도 스트레스 테스트를 진행했습니다. VUser 600명이 홈 피드(게시물) 조회 요청을 보내며, 저장된 게시물은 35만건입니다.

![image](https://user-images.githubusercontent.com/56240505/133962415-f0a2982b-46c1-4137-bcb4-95bd22fbc913.png)

* 읽기 (TPS 평균 31.4, 최대 35.5)

### 3.4. 로드 밸런싱 적용

이번에는 WAS를 1대 더 증설한 다음, Nginx Reverse Proxy로 Round Robin 방식의 로드 밸런싱을 적용했습니다.

![image](https://user-images.githubusercontent.com/56240505/133963095-20cff82d-ce8f-4cf5-adfd-b21a11a57c89.png)

* 읽기 (TPS 평균 31.3, 최대 40)

로드 밸런싱 적용 유무와는 상관없이 TPS 및 MMS는 큰 차이가 없었습니다. CloudWatch 분석 결과 단일 WAS 환경이나 로드 밸런싱 환경이나 Reverse Proxy EC2 및 WAS EC2의 CPU Utilization은 매우 낮은 수준이었습니다.

![image](https://user-images.githubusercontent.com/56240505/133962575-8dcda866-2b09-4fd3-943f-f507cdf8d6de.png)

반면 DB 서버 CPU Utilization은 매우 높았습니다. 조회 요청에 대한 DB 부하가 매우 심한 것으로 보입니다. 성능 향상을 위해 DB 서버 Scale-Out과 Scale-Up 및 Indexing 등을 활용한 쿼리 튜닝 등이 필요한 시점이라는 생각이 들었습니다.

<br>

## 4. 서비스의 한계

![image](https://user-images.githubusercontent.com/56240505/133959387-61b18900-fbaf-4b0c-845e-baf0d87678c1.png)

데이터를 100만건 넣고 VUser를 750으로 높여 테스트를 진행하자, TPS가 극적으로 하락했습니다. 또한 테스트 실패율이 50%를 초과했습니다. 확실히 데이터 셋이 많아질수록 성능이 매우 하락한다는 것을 확인할 수 있었습니다.

> Log

```java
o.h.engine.jdbc.spi.SqlExceptionHelper : hikari-pool-1 – Connection is not available, request timed out after 30000ms.
```

로그를 분석해보니 작업 쓰레드가 트랜잭션 처리를 위해 커넥션을 요청했으나, 30초동안 받지 못해 예외를 발생시키고 있었습니다. hikariCP를 적절하게 설정함으로써 해결할 수 있다는 글을 보았는데, 근본적으로는 DB 서버 Scale-Out과 Scale-Up 및 조회 쿼리 성능 튜닝 등이 답이 아닐까 추측했습니다.

게시물 조회를 처리하는 트랜잭션의 경우, 많은 테이블의 데이터를 조합해 응답을 작성합니다. 조회 쿼리에 Join이 포함되는 등 쿼리가 매우 복잡해집니다. N + 1 해결을 위해 Batch Fetch Size를 설정하긴 했으나, 완벽한 한방 쿼리가 아니기 때문에 추가적인 쿼리가 나갑니다.

CP 크기를 늘리더라도, 게시물 조회 트랜잭션 성능을 개선하지 못하는 이상 트랜잭션 스레드가 커넥션을 점유하는 시간은 여전히 길 것입니다. 즉, 늘린 CP 사이즈로도 감당하지 못할 트래픽이 몰린다면 동일한 HikariCP Dead Lock 문제가 또다시 발생할 것으로 추정됩니다.

추후 Pinpoint 등 다양한 도구를 도입해 더 섬세하게 지표를 분석하고 점진적으로 성능을 개선할 계획입니다. 😇

<br>

---

## Reference

* [HikariCP Dead lock에서 벗어나기 (이론편)](https://techblog.woowahan.com/2664/)
* [결제 시스템 성능, 부하, 스트레스 테스트](https://techblog.woowahan.com/2572/)
