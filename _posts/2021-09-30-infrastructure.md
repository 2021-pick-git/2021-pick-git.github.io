---
layout: post
title: Pick-Git의 Infrasturcture
date: 2021-09-30 13:32:20 +0300
description: Pick-Git 서비스를 지탱하는 Infrasturcture를 소개합니다.
img: infra.png
tags: [백엔드, Infrasturcture]
---

## 1. 들어가며

안녕하세요, 케빈입니다. Pick-Git 서비스 개발 초기에는 정말 몇 개 안되는 AWS EC2만을 사용했었는데요. 지속적으로 서비스를 개선 및 운영하면서 다양한 요구 사항을 직면하게 되었으며, 소기의 목적을 달성하기 위해 수 많은 AWS EC2를 생성해 사용하게 되었습니다. 그 결과 Pick-Git의 Infrasturcture가 다소 복잡해졌고, 프론트엔드뿐만 아니라 백엔드 팀원조차 자신이 맡은 영역이 아니면 제대로 이해하기 힘든 문제가 발생했습니다.

해당 문제를 해결하기 위해 Infrasturcture에 대한 팀내 WiKi 자료를 제작하게 되었는데요. 기술 블로그를 통해 독자님들에게도 Pick-Git 서비스를 지탱하는 Infrasturcture를 함께 공유하고자 합니다.

<br>

## 2. Overall

![image](https://user-images.githubusercontent.com/56240505/135299698-534d23c5-d6b5-4a6a-b924-dd670f049271.png)

Pick-Git에서 사용 중인 EC2는 크게 4가지 그룹으로 분류됩니다.

* Product 환경 관련 EC2
* Develop 환경 관련 EC2
* Test 환경 관련 EC2
* 기타 EC2 (Jenkins, SonarCube, nGrinder 등)

서비스에서 사용 중인 EC2 서버에 대한 SSH(22번 포트) 접근은 Bastion 서버 내부에서만 접속 가능하도록 구성했습니다. 개발 팀원은 자신의 Local PC에서 SSH를 통해 서비스 EC2 서버로 직접 접근할 수 없습니다. Bastion EC2 서버로 SSH 접속한 다음, Bastion 서버 내부에서 서비스 EC2 서버로 SSH 접속이 가능합니다.

SSH(22번 포트)는 호스트들이 Public Network를 통해 통신할 때 보안적으로 안전하게 통신하기 위해 사용하는 프로토콜(공개키 & 비밀키 방식)입니다. 만약 SSH 관련 보안 결함이 발생하면 서비스에 심각한 문제를 야기할 수 있습니다.

SSH(22번 포트)로 접근할 수 있는 모든 EC2 서버에 대해 동일한 수준의 보안을 설정하려면, Auto-Scaling 등 확장성을 고려한 구성과 배치가 필요합니다. Infrasturcture 규모가 커지면 관리 포인트 또한 늘어나기 때문에, 일반적으로 보안 설정을 일정 부분 포기해야 합니다.

Bastion 서버로 보안을 집중 시키고, 새로 생성하는 EC2 서버를 Bastion 서버로 연결시키면 모든 서버가 동일한 수준의 보안을 손쉽게 가질 수 있습니다. 악성 루트킷 혹은 랜섬웨어 등으로 피해를 보더라도 Bastion Server만 재구성하면 되므로, 서비스에 끼치는 악영향을 최소화할 수 있습니다.

Bastion 서버에 대한 구성 방법은 [Bastion 서버 구성](https://xlffm3.github.io/devops/bastion/) 글을 참고하시길 바랍니다.

<br>

## 3. Product 환경

![image](https://user-images.githubusercontent.com/56240505/135310979-e8bcc484-0118-4bda-b9a6-ab51c3360ab0.png)

서비스 초기에는 웹 서버(Product Reverse Proxy)에 프론트엔드 정적 파일을 배포한 다음, 정적 파일 요청은 웹 서버가 처리하고 동적인 컨텐츠 요청은 뒷단의 WAS로 요청을 위임해 처리하도록 구성했었습니다.

그러나 AWS에서 제공하는 CDN 서비스인 CloudFront로도 웹 서버가 제공하는 캐싱 기능 등의 이점을 충분히 누릴 수 있다고 판단했습니다. 따라서 AWS S3에 프론트엔드 정적 파일을 배포한 다음, S3 정적 파일 읽기 작업을 수행하는 CloudFront 주소에 Pick-Git 서비스 실 도메인 주소(https://pick-git.com)를 연결했습니다.

Product Reverse Proxy 주소는 TLS 적용 및 Product API 서버 도메인 주소를 연결해둔 상태입니다. 이 외에도 Reverse Proxy는 Product WAS에 대한 로드 밸런싱 및 Blue/Green 배포 작업 등을 수행합니다.

서비스 초기에는 단일 DB 서버를 구성했으나, 단일 장애점을 없애고 가용성을 높이기 위해 DB Replication을 적용했습니다. DB Replication을 이번 글에서 전부 설명하긴 어려울 것 같은데요. 짧게 요약하자면 DB 스토리지를 물리적으로 다른 서버에 복제하는 것입니다. Write 작업은 Master Node가 담당하고 Read 작업은 Slave Node가 담당함으로써 DB 서버 부하를 분산시킬 수 있으며 실시간 DB 데이터 백업이 가능합니다. 또한 하나의 Slave Node에 장애가 발생하더라도 다른 Slave Node(다중화)를 통해 서비스가 중단되지 않고 계속 정상 운영됩니다.

이미지 업로드 요청 API의 경우 Product WAS가 S3로 이미지를 업로드합니다.

<br>

## 4. Test 환경

![image](https://user-images.githubusercontent.com/56240505/135311128-0cd0c00a-b0e3-492e-81df-e04b7cfa2425.png)

Test 환경은 백엔드 API와 시나리오에 대한 Load & Stress Test 수행 및 성능 진단을 목적으로 생성했습니다. 최대한 Product 환경과 유사하게 구성했으며, nGrinder 서버를 통해 테스트를 진행합니다.

테스트 대상 시스템의 성능을 측정하기 위해서 외부 시스템은 항상 기대한 결과만을 반환하는 환경이 필요합니다. 이 때, 어플리케이션에서 객체를 Mocking하거나 Dummy Controller를 사용하지 않았습니다.

* Http Connection Pool 미사용
* Connection Thread 미사용
* I/O 미발생

등의 문제가 존재하기 때문입니다. 성능 테스트에서 중요한 관점인 Thread 및 리소스 사용을 무시하게 되며, 테스트 시스템의 자원과 리소스를 함께 사용하는 등 테스트의 신뢰성이 떨어집니다.

테스트를 위한 요소는 테스트 대상 시스템에 절대로 영향을 미쳐서는 안 되기 때문에, 테스트 대상 시스템과 완벽히 분리된 Mock 서버를 만들어 배포한 다음 테스트를 진행했습니다.

nGrinder를 활용한 부하 및 스트레스 테스트에 대한 상세한 내용은 우아한형제들 기술블로그의 [결제 시스템 성능, 부하, 스트레스 테스트](https://techblog.woowahan.com/2572/) 글을 참고하시길 바랍니다.

<br>

## 5. Develop 환경

![image](https://user-images.githubusercontent.com/56240505/135313157-4c48b29d-11b6-43cf-b0e4-34b43583f820.png)

Develop 환경의 경우 주로 개발 과정에서 프론트엔드 및 백엔드 기능들이 유기적으로 잘 연동되는지 등을 확인하는 역할입니다. 웹 서버(Develop Reverse Proxy)에 프론트엔드 정적 파일을 배포합니다.

<br>

## 6. SonarCube

![image](https://user-images.githubusercontent.com/56240505/135315104-7d2dc8a4-ed2e-4ebc-a358-0b690f29906f.png)
![image](https://user-images.githubusercontent.com/56240505/135314750-4957c932-0343-4c28-a987-010515cf4fa4.png)

Pick-Git 백엔드 프로젝트는 SonarCube를 통해 프로젝트 코드에 대한 정적 분석을 진행하고 있습니다.

``Backend`` 라벨이 달린 Pull Request가 생성 및 업데이트될 때마다 WebHook이 Jenkins 서버로 전송됩니다. Jenkins 서버의 SonarCube 파이프라인이 이를 감지하고 작업을 시작합니다. Build & Test 작업을 진행하고, JaCoCo 라이브러리를 통해 코드 커버리지 분석 리포트를 생성합니다. 최종적으로 SonarCube 서버에 정적 분석을 요청합니다.

이러한 분석 과정을 진행하면서 성공 혹은 실패했을 때 Jenkins는 GitHub Open API를 바탕으로 해당 PR에 결과를 댓글로 알림합니다. 테스트 커버리지 기준 미충족 등 파이프라인이 실패한 경우 Pull Request Merge를 방지하고 있습니다.

<br>

## 7. CI/CD Pipeline

![image](https://user-images.githubusercontent.com/56240505/135314865-ed0d7176-baf3-4a5a-88ca-7359732668b6.png)
![image](https://user-images.githubusercontent.com/56240505/135314932-00405217-7c00-40cd-a174-8c33bdd24e4e.png)

Pull Request Merge가 발생하면 WebHook이 Jenkins 서버로 전송됩니다. 이 때, 어떤 브랜치로 코드가 병합됬으며 Pull Request의 라벨이 ``Backend``와 ``Frontend`` 중 어떤 것인지 등의 정보를 Jenkins가 분석합니다. 이후 조건에 부합하는 Pipeline을 찾아 작업을 수행하고 적합한 서버로 코드를 배포합니다.

백엔드의 경우 Build & Test 작업 및 Rest Docs 등의 작업을 수행한 다음 코드를 배포합니다.

<br>

## 8. 마치며

![image](https://user-images.githubusercontent.com/56240505/135315783-f42a9cd5-d2e8-46a8-af6a-cace24dfba31.png)
![image](https://user-images.githubusercontent.com/56240505/135315820-a708bd5c-7de0-4d62-9d54-645d3b475f7e.png)
![image](https://user-images.githubusercontent.com/56240505/135315841-34e130e1-bf1e-4bfc-8e8f-7c495242d766.png)
![image](https://user-images.githubusercontent.com/56240505/135318118-f3570032-9cf1-4305-a39e-fc6d0e5fbcc4.png)

이번 WiKi 자료를 제작하면서 커뮤니케이션의 중요성에 대해 다시금 생각해보는 계기가 되었습니다. 프론트엔드 팀원들과 기능 시연을 진행하던 도중 백엔드 이슈로 인해 시연이 중단되는 경우가 왕왕 있었는데요. 백엔드 팀원들은 나름대로(?) 문제 원인을 설명하고 디버깅에 착수하곤 했습니다.

그런데 문제 원인을 프론트엔드 팀원의 눈높이가 아닌 백엔드 입장에서 너무 간략하게 설명하고 넘어갔습니다. 😅 이후 회고 때, 프론트엔드 팀원이 **문제 원인을 제대로 이해하지 못하고 백엔드가 문제가 해결할 때까지 그저 대기해야만 하니 약간 답답하다**는 고충을 토로했습니다.

따라서 Infrasturcture 설명 자료를 제작하면서, 신규 기능 배포 과정에서 발생할 수 있는 백엔드 이슈들에 대한 FAQ를 정리해 프론트엔드 팀원들과 소통하는 시간을 가졌습니다. 항상 ``나만 아는 지식이 없도록 하자``는 팀 규칙을 준수하려고 노력했는데요. ``소통의 시작은 상대방과 눈높이를 맞추는 것에서 시작``한다는 기본적인 격언부터 충실히 실천해야겠다고 느낀 하루였습니다.

<br>

---

## Reference

* [결제 시스템 성능, 부하, 스트레스 테스트](https://techblog.woowahan.com/2572/)
