---
layout: post
title: DB Replication을 구성한 이유
date: 2021-10-04 01:33:32 +0900
description: 깃-들다에서 DB Replication을 구성한 이유를 알아봐요.
img: db-replication.jpeg
tags: [백엔드, Database, Replication]
---

안녕하세요, 다니입니다. 🌻

<br/>

프로젝트를 진행하며 DB Replication을 적용하는 경험을 했습니다.<br/>
이번 글에서는 이를 활용한 이유를 간단하게 정리하려 합니다.<br/>

<br/>

## DB 기존 구조

현재는 사용자 수가 적어 WAS 1대, DB 1대로 인프라를 구축했습니다.<br/>
지금 당장은 서버에 문제가 생길 정도의 트래픽이 발생하지 않아 DB를 1대만 둬도 괜찮았습니다.<br/>

<p align="center">
    <img src="https://user-images.githubusercontent.com/50176238/135753219-abbfb51a-ec39-48df-aa2d-ab6222e16311.png">
</p>

한편, DB를 1대만 두면 단일 장애점이 될 수 있다는 우려가 있었습니다.<br/>
그래서 DB를 확장해야 하나 고민하던 찰나, 팀의 기대치만큼 서버가 트래픽을 처리할 수 있는지 스트레스 테스트를 진행하고 의사결정을 하기로 했습니다.

<br/>

### 성능 테스트

#### 목표값

- Throughput
    - 17 ~ 50회

#### 설정값

- DB 데이터
    - User - 20만 건
    - Post - 20만 건
    - Comment - 20만 건
- VUser
    - 읽기 연산 - 152명
    - 쓰기 연산 - 152명
- 방식
    - 읽기 연산과 쓰기 연산을 30분간 동시 수행

#### 읽기 연산

평균 TPS는 11.8, 최대 TPS는 22.5로 나왔습니다.<br/>

<p align="center">
    <img src="https://user-images.githubusercontent.com/50176238/135757029-8aae3eda-189f-48ed-95cc-376dcdfecc17.png">
</p>

#### 쓰기 연산

평균 TPS는 11.6, 최대 TPS는 22로 나왔습니다.<br/>

<p align="center">
    <img src="https://user-images.githubusercontent.com/50176238/135757067-a29c84d8-ba49-400d-b7c6-64e07130d2a2.png">
</p>

#### 결론

예상보다 TPS가 낮게 나타나는 걸 확인했습니다.<br/>
따라서 DB를 Scale up 또는 Scale out하여 성능을 높이기로 했는데, 프로젝트 상황을 고려하면 Scale up은 불가능해 Scale out을 하기로 결정했습니다.

<br/>

## DB 확장 방법

DB를 Scale out하는 방법은 Clustering과 Replication이 있습니다.<br/>
Sharding도 Scale out을 하는 게 아닌가? 라고 생각할 수 있는데, Sharding은 데이터 조회 성능을 향상시키기 위해 데이터를 분산 저장하는 기술입니다.<br/>
Clustering과 Replication은 각 서버가 동일한 데이터를 저장하고 있는 반면, Sharding은 각 서버가 서로 다른 데이터를 저장하고 있습니다.<br/>
따라서, 여기서는 Clustering과 Replication을 중심으로 비교하고 결론을 내리겠습니다.

<br/>

### Clustering

Clustering은 DB 서버를 2대 이상, DB 스토리지를 1대로 구성하는 형태입니다.<br/>

그림처럼 DB 서버 2대를 모두 Active 상태로 운영하면, DB 서버 1대가 죽더라도 DB 서버 1대는 살아있어 서비스는 정상적으로 할 수 있습니다.<br/>
또한 DB 서버를 여러 대로 두면, 트래픽을 분산하여 감당하게 되어 CPU와 Memory에 대한 부하가 적어지는 장점이 있습니다.<br/>

그러나, DB 서버들이 DB 스토리지를 공유하기 때문에 DB 스토리지에 병목이 생기는 단점이 있습니다.<br/>

<p align="center">
    <img src="https://user-images.githubusercontent.com/50176238/135759902-fa2a6b46-8423-468d-8e30-9d96aeb3419d.png">
</p>

<br/>

그래서 DB 서버 2대 중 1대를 Stand-by 상태로 두어 단점을 보완할 수 있습니다.<br/>

Stand-by 상태의 서버는 Active 상태의 서버에 문제가 생겼을 때, Fail Over를 통해 상호 전환되어 장애에 대응할 수 있습니다.<br/>
이와 같이 구성하면, DB 스토리지에 대한 병목 현상도 해결됩니다.<br/>

그렇지만, Fail Over가 이뤄지는 시간 동안은 영업 손실이 필연적으로 발생합니다.<br/>
또 DB 서버 비용은 이전과 동일하나, 가용률은 이전에 비해 대략 1/2로 줄어드는 단점도 있습니다.<br/>

<p align="center">
    <img src="https://user-images.githubusercontent.com/50176238/135760156-bb900351-2411-49cb-b54f-187f0c6cbec3.png">
</p>

<br/>

추가적으로, Clustering은 DB 스토리지를 1대만 사용하기 때문에 DB 스토리지가 단일 장애점이 될 수 있습니다.<br/>
DB 스토리지에 문제가 발생하면, 데이터를 복구할 수 없는 치명타가 생깁니다.<br/>

<br/>

### Replication

Replication은 각 DB 서버가 각자 DB 스토리지를 갖고 있는 형태입니다.<br/>

사용자가 Master DB에 SELECT/INSERT/UPDATE/DELETE을 하면, Master DB는 Slave DB에 데이터를 복제합니다.<br/>
이렇게 하면, DB 스토리지 단일 장애점을 해소하여 데이터 손실을 방지할 수 있습니다.<br/>

하지만, Master DB만 일을 하고 Slave DB는 놀게 되는 상황이 발생합니다.<br/>

<p align="center">
    <img src="https://user-images.githubusercontent.com/50176238/135760449-60c7016f-c833-40bf-a631-1f7628b2bb40.png">
</p>

<br/>

그래서 Master DB에 INSERT/UPDATE/DELETE를 하고, Slave DB에 SELECT를 하는 방식으로 각각 DB에 트래픽을 분산할 수 있습니다.<br/>

<p align="center">
    <img src="https://user-images.githubusercontent.com/50176238/135760467-1297451d-1b45-467b-ba51-1754658137a1.png">
</p>

<br/>

### 결론

Clustering은 DB 스토리지를 1대로 구성한다는 부분에서 잘못하면 큰 문제로 이어질 것 같았습니다.<br/>
따라서 Replication을 선택하는 게 안전하고 올바르겠다는 생각이 들었고, Replication을 적용하기로 결정했습니다.<br/>

이때, Master DB 1대에 Slave DB는 몇 개로 둬야 할지 고민하게 됐습니다.<br/>
SNS 특성상 읽기 연산이 빈번하게 발생하기 때문에 Slave DB를 1대로 두는 것보다 2대 이상 두는 것이 나을 거라 판단했습니다.<br/>

인프라 비용을 생각하면 Slave DB를 무작정 많이 두는 건 옳지 않았습니다.<br/>
그래서 Slave DB를 우선 2대로 두고, 나중에 더 많은 트래픽이 발생하면 Slave DB의 개수를 늘리기로 했습니다.<br/>

<br/>

## DB Replication 이후 구조

앞에서 설명한 것처럼 Master DB 1대와 Slave DB 2대로 구성하게 됐습니다.<br/>

<p align="center">
    <img src="https://user-images.githubusercontent.com/50176238/135752960-f8a081d5-4ef7-4700-b56e-9de90cff3ed1.png">
</p>

### 성능 테스트

#### 목표값

- Throughput
    - 17 ~ 50회

#### 설정값

- DB 데이터
    - User - 20만 건
    - Post - 20만 건
    - Comment - 20만 건
- VUser
    - 읽기 연산 - 152명
    - 쓰기 연산 - 152명
- 방식
    - 읽기 연산과 쓰기 연산을 30분간 동시 수행

#### 읽기 연산

평균 TPS는 40.1, 최대 TPS는 62로 나왔습니다.<br/>

<p align="center">
    <img src="https://user-images.githubusercontent.com/50176238/135757796-65468741-a1ae-493a-a4b7-43612256d958.png">
</p>

#### 쓰기 연산

평균 TPS는 85.9, 최대 TPS는 121로 나왔습니다.<br/>

<p align="center">
    <img src="https://user-images.githubusercontent.com/50176238/135757806-4e5136a4-a783-4969-941a-9e7df6620fc9.png">
</p>

#### 결론

Replication을 설정하니 TPS가 기대치보다 높게 나타났습니다.<br/>
DB를 Scale out하는 게 맞는 걸까? 라는 의구심이 있었는데, 적절했다는 확신을 얻게 됐습니다.<br/>

한편 읽기 연산에서 TPS가 점진적으로 하락하는 양상을 보이는데, 이는 쓰기 작업이 누적되어 데이터 개수가 늘어나서 영향을 받은 것으로 추측됩니다.<br/>

<br/>

## 마무리

애플리케이션 개발에서 성능을 개선할 수 있는 지점은 다양하게 존재합니다.<br/>
이 경험에서는 DB 확장을 통해 성능을 조금 더 좋게 만들 수 있었습니다.<br/>

<br/>

## References

- [Clustering vs Replication vs Sharding](https://jordy-torvalds.tistory.com/94)
- [How To Setup MySQL DB Replication On Kubernetes](https://faun.pub/how-to-setup-mysql-db-replication-on-kubernetes-95292cb2d7f1) (썸네일)