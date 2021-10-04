---
layout: post
title: Gradle 프로젝트에 JaCoCo & SonarQube 적용
date: 2021-09-30 13:32:20 +0300
description: 소프트웨어는 변화가 잦은 만큼 개발 과정에서 예기치못한 부작용을 직면하기 쉽습니다.
img: sonarcube.png
tags: [백엔드, SonarQube, JaCoCo]
---

## 1. 들어가며

안녕하세요, 케빈입니다. 개발 관련 서적 및 강의에서 단연 강조하는 내용 중 하나가 테스트 코드 작성의 중요성이 아닐까 싶습니다. 테스트 코드를 꼼꼼하게 작성하면 어떤 장점이 있을까요?

* 개발자가 작성한 기능들이 의도한대로 정상 동작하는지 확인할 수 있습니다.
* 기능 검증 과정을 통해 미처 생각하지 못한 엣지 케이스를 사전에 포착 및 대응함으로써 서비스의 안정성을 높여줍니다.
* 기존 기능이 수정되거나 신규 기능이 추가되었을 때 발생할 수 있는 부작용(Side-Effect)를 쉽게 발견하고 제거합니다.
* TDD를 통해 가독성 및 재사용성이 높은 객체지향적인 코드를 작성하는데 의식적인 노력을 할 수 있습니다.

소프트웨어는 변화가 잦은 만큼 개발 과정에서 예기치못한 부작용을 직면하기 쉽습니다. 이 때문에 Pick-Git 팀 또한 꼼꼼하게 테스트 코드를 작성하고자 의식적으로 노력 중입니다. 위의 장점을 제대로 누리기 위해서는 로직 분기상 발생할 수 있는 모든 경우의 수에 대해 빠짐없이 테스트 코드를 작성해야 합니다. 그러나 개발자도 사람인지라 테스트 코드를 작성하는 과정에서, 일부 로직에 대한 테스트 코드 작성이 인적 실수로 누락되고는 합니다.

우리가 작성한 테스트 코드가 얼마나 꼼꼼하게 잘 작성되었는지 정량적으로 확인할 수 있는 지표가 있다면, 이런 문제를 초기에 해결함으로써 코드 품질을 높게 유지할 수 있을 것입니다. 오늘 글에서 알아볼 코드 커버리지 및 정적 코드 분석 도구가 바로 테스트 코드 작성에 대한 대표적인 정량 지표입니다.

<br>

## 2. Code Coverage

코드 커버리지는 소프트웨어의 테스트 케이스가 얼마나 충족되었는지를 나타내는 지표 중 하나입니다. 테스트를 진행했을 때 코드 자체가 얼마나 실행되었는지 **수치**를 의미합니다. 코드의 안정성을 보장해주는 지표라고 생각하면 쉽습니다. 코드 커버리지는 대게 **소스 코드를 기반으로 수행하는 테스트**를 통해 측정합니다.

테스트 코드는 모든 시나리오에 대해 작성되는 것이 베스트지만, 인적 실수로 인해 누락되기 싶습니다. 코드 커버리지 분석을 통해 테스트가 누락된 부분을 빠르게 파악함으로써 인적 실수를 예방할 수 있습니다. 코드 커버리지는 후술할 정적 코드 분석 도구 등과 함께 활용하는데요. 코드 커버리지 기준에 미달하면 코드를 Merge하지 못하게 방지함으로써, 코드 커버리지 및 프로덕트 코드 품질을 항상 일정 수준 이상으로 유지하는 것이 일반적입니다.

### 2.1. 기준

코드의 구조는 크게 구문, 조건, 결정으로 분류할 수 있습니다. 코드 커버리지는 이러한 코드 구조를 얼마나 커버했느냐에 따라 측정 기준이 달라집니다. 예제 및 상세한 설명이 궁금하다면 [코드 분석 도구 적용기 - 1편, 코드 커버리지(Code Coverage)가 뭔가요?](https://velog.io/@lxxjn0/%EC%BD%94%EB%93%9C-%EB%B6%84%EC%84%9D-%EB%8F%84%EA%B5%AC-%EC%A0%81%EC%9A%A9%EA%B8%B0-1%ED%8E%B8-%EC%BD%94%EB%93%9C-%EC%BB%A4%EB%B2%84%EB%A6%AC%EC%A7%80Code-Coverage%EA%B0%80-%EB%AD%94%EA%B0%80%EC%9A%94) 글을 참고 바랍니다.

* 구문(Statement)
  * 라인 커버리지로서, 코드 한 줄이 한 번이상 실행되면 충족됩니다.
  * 가장 많이 사용되는 기준이다.
* 조건(Condition)
  * 모든 조건식의 내부 조건이 true/false을 가지게 되면 충족된다.
  * 조건 커버리지를 기준으로 테스트를 진행할 경우, 구문 커버리지와 결정 커버리지를 만족하지 못하는 경우가 존재합니다.
* 결정(Decision)
  * 브랜치 커버리지로서, 모든 조건식이 true/false을 가지게 되면 충족됩니다.

### 2.2. 각 기준의 장단점

조건 및 결정 커버리지는 코드 실행에 대한 테스트보다는 로직의 시나리오에 대한 테스트입니다. 조건(true/false)에 대해 모두 만족하면 코드 커버리지를 만족한다고 봅니다. 즉, 조건문이 존재하지 않는 코드는 두 커버리지의 대상에서 제외되며 해당 코드를 아예 테스트하지 않습니다.

반면 라인 커버리지는 조건문의 true/false 시나리오에 대해 완벽히 테스트했다고 보장할 수 없지만, 조건문 내부의 코드가 실행되었을 때 문제가 없다는 것은 보장할 수 있습니다. 다시 말해 라인 커버리지를 만족하면 모든 시나리오를 테스트한다는 보장은 할 수 없지만, 어떤 코드가 실행되더라도 해당 코드는 문제가 없다는 보장은 할 수 있다는 의미입니다. 따라서 라인 커버리지를 많이 사용하는 편입니다.

<br>

## 3. JaCoCo

Java 어플리케이션의 코드 커버리지를 분석해주는 라이브러리입니다. 대안으로 Cobertura 및 Clover 등이 있으나 JaCoCo가 가장 레퍼런스가 많으며 사용 방법이 간단합니다.

* 코드 커버리지에 대한 결과를 HTML, CSV, XML 등의 리포트로 산출합니다.
* 설정한 커버리지 만족 여부를 확인할 수 있습니다.

<br>

## 4. Gradle 설정

> build.gradle

```groovy
plugins {
    id 'jacoco'
}

jacoco {
    toolVersion = '0.8.7'
}
```

기본적으로 HTML 리포트는 ``$buildDir/reports/jacoco/test`` 디렉토리에 생성됩니다.

> build.gradle

```groovy
test {
    useJUnitPlatform()
    finalizedBy jacocoTestReport
}

jacocoTestReport {
    dependsOn test
    reports {
        html.enabled true
        xml.enabled true
        csv.enabled true

        html.destination file("src/jacoco/jacoco.html")
        xml.destination file("src/jacoco/jacoco.xml")
    }
    finalizedBy 'jacocoTestCoverageVerification'
}

jacocoTestCoverageVerification {
    violationRules {
        rule {
            element = 'CLASS'
            enabled = true

            limit {
                counter = 'LINE'
                value = 'COVEREDRATIO'
                minimum = 0.60
            }

            limit {
                counter = 'METHOD'
                value = 'COVEREDRATIO'
                minimum = 0.60
            }
        }
    }
}
```

JaCoCo로 분석 리포트를 생성하는 Task 수행 전에 항상 테스트 Task 수행되도록 finalizedBy 및 dependsOn 명령어를 추가합니다.

``jacocoTestReport``는 코드 커버리지 결과를 원하는 형태의 리포트로 생성해주는 Task입니다. 분석 리포트 수행시 생성되는 파일 종류를 설정할 수 있으며, build 디렉토리에 생성되는 파일을 원하는 위치로 가져올 수 있습니다.

이후 수행되는 ``jacocoTestCoverageVerification``는 분석된 코드 커버리지가 미리 설정한 기준치에 충족하는지 여부를 확인합니다. 참고로 limit 구문에 들어갈 수 있는 Key 값과 Value 값은 다양하니 공식 문서를 참고하길 바랍니다.

![image](https://user-images.githubusercontent.com/56240505/131561797-7af304a9-178e-4e1b-aef8-cf7e584c43ef.png)

기능적으로 테스트들은 모두 통과하더라도, violationRules에 정한 기준치를 통과하지 못하면 결국 ``jacocoTestCoverageVerification`` Task가 실패해서 빌드에 실패합니다. 코드 커버리지 충족이 어렵다면 기준을 현실적으로 적당히 타협한 뒤 점진적으로 향상하는 방법을 고려합시다. 아울러 테스트 할 필요가 없거나 하기 어려운 파일들은 인위적으로 검사에서 제외할 수 있습니다.

> build.gradle

```groovy
jacocoTestReport {
    dependsOn test
    reports {
        html.enabled true
        xml.enabled true
        csv.enabled true

        html.destination file("src/jacoco/jacoco.html")
        xml.destination file("src/jacoco/jacoco.xml")
    }

    def Qdomains = []
    for (qPattern in '**/QA'..'**/QZ') {
        Qdomains.add(qPattern + '*')
    }

    afterEvaluate {
        classDirectories.setFrom(
                files(classDirectories.files.collect {
                    fileTree(dir: it, excludes: [
                            '**/BlogApplication*',
                            '**/*Request*',
                            '**/*Response*',
                            '**/*Dto*',
                            '**/*OAuthClient*',
                            '**/*Interceptor*',
                            '**/*Exception*',
                            '**/*Storage*',
                            '**/*BaseDate*',
                            '**/*PageController*'
                    ] + Qdomains)
                })
        )
    }
    finalizedBy 'jacocoTestCoverageVerification'
}

jacocoTestCoverageVerification {
    def Qdomains = []
    for (qPattern in '*.QA'..'*.QZ') {
        Qdomains.add(qPattern + '*')
    }

    violationRules {
        rule {
            element = 'CLASS'
            enabled = true

            limit {
                counter = 'LINE'
                value = 'COVEREDRATIO'
                minimum = 0.60
            }

            limit {
                counter = 'METHOD'
                value = 'COVEREDRATIO'
                minimum = 0.60
            }

            excludes = [
                    '**.*BlogApplication*',
                    '**.*Request*',
                    '**.*Response*',
                    '**.*Dto*',
                    '**.*OAuthClient*',
                    '**.*Interceptor*',
                    '**.*Exception*',
                    '**.*Storage*',
                    '**.*BaseDate*',
                    '**.*PageController*',
            ] + Qdomains
        }
    }
}
```

afterEvaluate 옵션은 분석 리포트 생성시 특정 파일들을 제외합니다. excludes 옵션은 코드 커버리지 조건 충족 확인시 특정 파일들을 제외합니다.

QueryDsl을 사용하는 경우 추가적으로 생성되는 QClass 파일들을 분석 리포트 생성 및 커버리지 조건 검사에서 제외시켜야 합니다.

![image](https://user-images.githubusercontent.com/56240505/131563842-56ec5f6d-788f-4f3d-9e2d-c5b8f6588987.png)

테스트를 수행하면 build 디렉토리에 분석 리포트가 잘 생성된 것을 확인할 수 있습니다.

![image](https://user-images.githubusercontent.com/56240505/131563575-ae6032b5-78aa-4a0a-9afc-693468175262.png)

또한 build 디렉토리에 생성된 파일들이 원하는 경로로 복사된 것을 확인할 수 있습니다.

<img width="1179" alt="스크린샷 2021-09-01 오전 4 42 58" src="https://user-images.githubusercontent.com/56240505/131565965-45d20924-0c61-4e93-b4f0-a5c128f3aa50.png">
<img width="1179" alt="스크린샷 2021-09-01 오전 4 43 25" src="https://user-images.githubusercontent.com/56240505/131565940-135298d5-2bc8-42af-bf69-d24088f85b34.png">

세부적으로 코드 커버리지를 확인할 수 있습니다.

### 4.1. Lombok 제외

> lombok.config

```config
lombok.addLombokGeneratedAnnotation = true
```

해당 파일을 gradlew가 위치한 프로젝트 루트 위치에 저장장합니다. 이는 @Builder, @Getter 등의 테스트를 제외시킵니다.

<br>

## 5. SonarQube

정적 프로그램 분석이란 실제 실행 없이 컴퓨터 소프트웨어를 분석하는 것을 의미합니다. 정적 코드 분석 도구는 개발된 코드에서 버그, 코드 스멜, 보안 취약점 등 코드 품질을 지속적으로 검사해주는 플랫폼입니다.

앞서 언급한 JaCoCo는 코드 커버리지만 체크할 뿐 그 외적인 코드 품질 요소를 검사하는 기능은 없습니다. 이를 보완하기 위해 정적 코드 분석 도구를 함께 사용합니다.

SonarQube는 대표적인 정적 코드 분석 도구입니다. PMD, FindBugs, CheckStyle 등 대안 도구가 존재하지만 레퍼런스가 상대적으로 부족합니다. 또한 SonarQube는 Github 및 Jenkins 등과 연동이 간편하며, 자동 정적 코드 분석을 쉽게 구성할 수 있습니다. 아울러 오픈 소스이며 플러그인 및 언어를 다양하게 지원합니다.

JaCoCo(코드 커버리지 분석 리포트) 및 SonarQube(정적 분석 도구)를 함께 사용하면 개발된 코드를 자동 및 지속적으로 검사하게 됩니다. 이는 코드 커버리지 및 코드 품질 목표를 달성하고 일정 수준 이상을 유지하는데 기여합니다.

<br>

## 6. EC2 SonarQube 컨테이너 실행

> Shell

```bash
$ sudo apt update && sudo apt install -y docker.io && sudo apt install -y docker-compose
$ sudo usermod -aG docker ubuntu
```

Docker를 설치하고 이후 ``docker-compose.yml`` 파일을 작성합니다.

> docker-compose.yml

```bash
version: '3'
services:
   SonarQube:
      image: sonarqube:7.7-community
      container_name: SonarQube-conatiner
      ports:
        - 8080:9000
      # environment:
		  #     - sonar.jdbc.url=jdbc:mysql://localhost:3306/sonarqube?useUnicode=true&characterEncoding=utf8&rewriteBatchedStatements=true&useConfigs=maxPerformance
		  #     - sonar.jdbc.username=sonar
		  #     - sonar.jdbc.password=sonar
	    # volumes:
      #   - /home/SonarQube:/opt/sonarqube
```

SonarQube 7.8 이상부터는 JDK11 및 PostgreSQL만 지원하기 때문에 7.7을 사용했습니다. SonarQube 분석 결과를 별도로 DB에 저장할 수 있는데요. 원한다면 해당 옵션 주석을 해제하고 DB를 구성하면 됩니다.

SonarQube는 기본적으로 9000 포트를 사용하는데, 위 예제의 경우 8080으로 접속하도록 설정했습니다.

> Shell

```bash
$ docker-compose up -d
```

컨테이너를 실행합니다.

<br>

## 7. SonarQube 설정

![스크린샷 2021-08-31 오후 12 09 51](https://user-images.githubusercontent.com/56240505/131566189-17e468af-cd99-454f-bde6-26df829d8dc8.png)

``http://{ec2-SonarQube-public-ip}:{port}``로 접속한 뒤 로그인합니다. 기본 계정은 아이디 및 비밀번호가 모두 admin입니다.

<img width="433" alt="스크린샷 2021-08-31 오후 2 24 48" src="https://user-images.githubusercontent.com/56240505/131567754-5ba54a93-aebb-44d4-a395-e653caf00616.png">

프로젝트를 생성합니다.

![image](https://user-images.githubusercontent.com/56240505/131566456-1cf5554c-9f53-434e-83e8-f816ae311041.png)

토큰을 생성하고 메모해둡니다.

<img width="757" alt="스크린샷 2021-09-01 오전 4 49 14" src="https://user-images.githubusercontent.com/56240505/131566750-3bf316c6-ff23-4217-87a4-76a79c4db1ba.png">

이후 친절하게 Gradle로 SonarQube Scanner를 실행하는 방법을 알려주는데요. 해당 방법은 Gradle 프로젝트에 SonarQube를 설정하는 방식이라 이번 글에서는 사용하지 않습니다.

<img width="606" alt="스크린샷 2021-09-01 오전 4 51 24" src="https://user-images.githubusercontent.com/56240505/131566952-404e7f03-f0bf-4e70-be70-ba521fc62201.png">
<img width="486" alt="스크린샷 2021-09-01 오전 4 52 11" src="https://user-images.githubusercontent.com/56240505/131567072-d92496a9-ff3d-4c2c-b93e-25b33a308f32.png">

생성한 프로젝트로 들어가서 Webhook을 위와 같이 설정해줍니다. Jenkins에서 SonarQube 정적 분석 요청을 보내면 처리 결과를 Jenkins로 보내기 위함입니다.

<br>

## 8. Jenkins 설정

아직 Jenkins를 구축하지 않았다면 [Jenkins Pipeline를 통한 CI/CD 구현](https://xlffm3.github.io/devops/jenkins-pipeline/) 글을 참고하여 Jenkins 서버를 구축합니다. 이후 다음 플러그인 3개를 설치합니다.

* SonarQube Scanner for Jenkins
* Sonar Quality Gates Plugin
* GitHub Pull Request Builder

<img width="441" alt="스크린샷 2021-08-31 오후 2 31 59" src="https://user-images.githubusercontent.com/56240505/131567980-b24eee21-bc83-4778-8a71-ded1552bf648.png">

``Jenkins 관리- Credentials 관리`` 탭에서 새로운 Credential을 추가합시다. 이전에 발급받은 SonarQube Token을 Secret text로 설정하면 됩니다.

<img width="527" alt="스크린샷 2021-08-31 오후 2 34 35" src="https://user-images.githubusercontent.com/56240505/131567986-54c350ba-4946-4a95-acc4-12e0fd94f6a5.png">

그 다음 환경 설정에서 SonarQube Server 이름과 URL 및 토큰을 지정해주면 됩니다.

![image](https://user-images.githubusercontent.com/56240505/131568653-bd849f70-dd53-4a34-b4c0-b414931a92f4.png)

동일한 환경 설정 탭에서 GitHub Pull Request Builder를 설정해줍시다. Credentials는 [GitHub Personal Access Token](https://github.com/settings/tokens)입니다. Admin list에 본인의 GitHub id를 기입해둡시다.

<img width="421" alt="스크린샷 2021-09-01 오전 4 59 18" src="https://user-images.githubusercontent.com/56240505/131567967-ecd36dba-0b81-4570-99b4-c40f31ea7709.png">

``Jenkins 관리- Global Tool Configuration``에서 위 사진과 같이 SonarQube Scanner를 설정합니다.

### 8.1. GitHub Pull Request Builder

코드 배포를 위해 사용하는 일반적인 Jenkins 파이프라인은 특정 브랜치에 코드가 푸시될 때, 빌드 및 테스트 등을 진행하는 구조입니다.

그러나 SonarQube 파이프라인의 경우 PR이 생성되고 Merge되기 전에 파이프라인이 동작합니다. SonarQube 파이프라인은 다음 항목들을 검사할 계획입니다.

* Build & Test 성공인가?
* JaCoCo가 설정한 코드 커버리지 기준을 충족하는가?
* SonarQube가 설정한 코드 스멜 기준을 충족하는가?

만약 하나라도 실패한다면 PR Merge를 금지시킵니다. 이를 통해 코드의 문제점을 신속하게 특정하고 점진적으로 코드 퀄리티를 향상 시킬 수 있습니다.

GitHub Pull Request Builder는 PR이 발생하면 스크립트를 실행하고, 빌드가 끝난 뒤에 결과를 리포팅 해주는 플러그인입니다. Jenkins가 PR에 결과 리포팅을 하기 위해 어드민의 Personal Access Token이 필요한 것입니다.

<br>

## 9. Pipeline 프로젝트 설정

<img width="625" alt="스크린샷 2021-09-01 오전 5 20 28" src="https://user-images.githubusercontent.com/56240505/131570473-bd28bfe4-e768-4fd3-9785-f9915a4b89bb.png">

먼저 SonarQube를 위한 Pipeline 프로젝트를 생성하고 위와 같이 설정해줍시다.

![image](https://user-images.githubusercontent.com/56240505/131570625-787d8c00-6e8a-4a0e-a338-efa396f50a01.png)

고급 버튼을 선택하여 Pipeline 스크립트 실행을 유발시키는 라벨과 그렇지 않은 라벨 및 화이트 리스트 등을 자유롭게 선택할 수 있습니다. 저는 backend 라벨을 붙인 PR에 한해서 정적 분석을 진행하고자 합니다.

<img width="609" alt="스크린샷 2021-09-01 오전 5 18 31" src="https://user-images.githubusercontent.com/56240505/131570242-394bbc69-13ad-45e7-9865-76529fde5920.png">

``Use github hooks for build triggering``을 체크한 뒤 저장했면, 정적 분석을 진행할 GitHub 프로젝트 레포지토리에 위와 같은 Webhook이 추가됩니다.

### 9.1. 참고

![image](https://user-images.githubusercontent.com/56240505/131571051-585c2813-d648-4962-bb2f-797091d5a80a.png)

SonarQube를 사용하는 방법에는 크게 3가지가 있습니다. 복잡한 파이프라인이 불필요하다면 Freestyle로도 충분합니다.

* SonarQube 분석에 필요한 정보(서버 주소, 토큰, 파일 위치 등)를 Gradle 프로젝트에 기술하고 Jenkins Pipeline에서는 간단히 ``./gradlew --info sonarqube``를 실행하는 방식.
* 위 사진 처럼 Jenkins Freestyle 프로젝트를 통해 간단히 SonarQube를 사용하는 방식.
* Gradle 프로젝트에 별다른 설정 없이 이번 글처럼 Jenkins Pipeline을 통해 SonarQube를 사용하는 방식.

<br>

## 10. Pipeline 스크립트 작성

> Pipeline Script

```groovy
pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                checkout([$class: 'GitSCM', branches: [[name: '${sha1}']], extensions: [[$class: 'SubmoduleOption', disableSubmodules: false, parentCredentials: true, recursiveSubmodules: true, reference: '서브모듈 레포 주소', trackingSubmodules: true]], userRemoteConfigs: [[credentialsId: '액세스토큰 지정값', refspec: '+refs/pull/*:refs/remotes/origin/pr/*', url: '프로젝트 레포 주소']]])
            }
        }

        stage('Build & Test') {
            steps {
                sh'''
                   cd backend/ && ./gradlew clean build || IS_FAIL=true
                   if [[ $IS_FAIL = "true" ]]; then
                      echo "Build Failure"
                      /* 원한다면 GitHub API PR에 댓글다는 기능을 여기에 작성하여 실패 사유 알람을 보낼 수 있다. */
                      exit 1
                   else
                      echo "Build Success"
                   fi
                '''
            }
        }

        stage('JaCoCo Report') {
            steps {
                sh'''
                    cd backend/ && ./gradlew jacocoTestReport || IS_FAIL=true
                    if [[ $IS_FAIL = "true" ]]; then
                        echo "JaCoCo Report Failure"
                        /* 원한다면 GitHub API PR에 댓글다는 기능을 여기에 작성하여 실패 사유 알람을 보낼 수 있다. */
                        exit 1
                    else
                        echo "JaCoCo Report Success"
                    fi
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('환경설정에서 지정한 소나큐브 서버 이름') {
                    sh'''
                        cd backend/
                        /var/jenkins_home/tools/hudson.plugins.sonar.SonarRunnerInstallation/지정한 소나큐브 스캐너 이름/bin/sonar-scanner \
                        -D sonar.host.url=소나큐브 서버 주소 \
                        -D sonar.login=소나큐브 토큰 \
                        -D sonar.projectKey=소나큐브 프로젝트 키 \
                        -D sonar.projectName=소나큐브 프로젝트 이름 \
                        -D sonar.projectVersion=0.0.1-SNAPSHOT \
                        -D sonar.sources=src/main/java \
                        -D sonar.tests=src/test \
                        -D sonar.java.binaries=build/classes \
                        -D sonar.java.libraries= \
                        -D sonar.java.libraries.empty=true \
                        -D sonar.sourceEncoding=UTF-8 \
                        -D sonar.java.coveragePlugin=jacoco \
                        -D sonar.coverage.jacoco.xmlReportPaths=src/jacoco/jacoco.xml \
                        -D sonar.test.inclusions=**/*Test.java \
                        -D sonar.exclusions=**/dto/**,**/*OAuthClient.java,**/S3*.java \
                    '''
                }
            }
        }

        stage('SonarQube') {
            steps {
                script {
                    def qg = waitForQualityGate();
                    if (qg.status != 'OK') {
                       sh'''
                          echo "SonarQube Quality Failure"
                          /* 원한다면 GitHub API PR에 댓글다는 기능을 여기에 작성하여 실패 사유 알람을 보낼 수 있다. */
                       '''
                       error "Pipeline aborted due to quality gate failure"
                    }
                    echo "All Passed!"
                    /* 원한다면 GitHub API PR에 댓글다는 기능을 여기에 작성하여 성공 사유 알람을 보낼 수 있다. */
                }
            }
        }
    }
}
```

Pipeline을 작성하는 방법은 [Jenkins Pipeline를 통한 CI/CD 구현](https://xlffm3.github.io/devops/jenkins-pipeline/) 글을 참고하길 바랍니다. Checkout의 경우 서브 모듈을 사용하고 있는 스크립트이기 때문에, 서브 모듈을 사용하지 않는다면 스크립트가 다를 수 있습니다.

> stage('Checkout')

```groovy
branches: [[name: '${sha1}']]
refspec: '+refs/pull/*:refs/remotes/origin/pr/*'
```

Checkout의 경우 위 두 부분을 주의해서 작성해야 합니다. ``${sha1}`` 는 PR을 발생시킨 브랜치 명을 의미합니다.

> stage('JaCoCo Report')

```groovy
stage('JaCoCo Report') {
    steps {
        sh'''
            cd backend/ && ./gradlew jacocoTestReport || IS_FAIL=true
            if [[ $IS_FAIL = "true" ]]; then
                echo "JaCoCo Report Failure"
                exit 1
            else
                echo "JaCoCo Report Success"
            fi
        '''
    }
}
```

JaCoCo 리포트를 생성하는 Task를 실행합니다. 사실 이번 글의 Gradle 예제에서는 JaCoCo 관련 Task가 Test Task와 의존 관계를 맺기에, 그 이전의 Stage인 Build & Test에서 ``./gradlew clean build``를 할 때 리포트가 생성됩니다.

따라서 해당 stage는 지워도 되는데요. Pick-Git 서비스의 경우 예제 Gradle 설정과 다르게 Test 및 JaCoCo 관련 Task들이 서로 의존 관계가 없는 상태라 추가해두었습니다.

> stage('SonarQube Analysis')

```groovy
stage('SonarQube Analysis') {
    steps {
        withSonarQubeEnv('환경설정에서 지정한 소나큐브 서버 이름') {
            sh'''
                cd backend/
                /var/jenkins_home/tools/hudson.plugins.sonar.SonarRunnerInstallation/지정한 소나큐브 스캐너 이름/bin/sonar-scanner \
                -D sonar.host.url=소나큐브 서버 주소 \
                -D sonar.login=소나큐브 토큰 \
                -D sonar.projectKey=소나큐브 프로젝트 키 \
                -D sonar.projectName=소나큐브 프로젝트 이름 \
                -D sonar.projectVersion=0.0.1-SNAPSHOT \
                -D sonar.sources=src/main/java \
                -D sonar.tests=src/test \
                -D sonar.java.binaries=build/classes \
                -D sonar.java.libraries= \
                -D sonar.java.libraries.empty=true \
                -D sonar.sourceEncoding=UTF-8 \
                -D sonar.java.coveragePlugin=jacoco \
                -D sonar.coverage.jacoco.xmlReportPaths=src/jacoco/jacoco.xml \
                -D sonar.test.inclusions=**/*Test.java \
                -D sonar.exclusions=**/dto/**,**/*OAuthClient.java,**/S3*.java \
            '''
        }
    }
}
```

``withSonarQubeEnv`` 블록을 사용해 연동할 SonarQube 서버를 선택할 수 있습니다. Jenkins global configuration에 저장된 연결 설정 정보를 자동으로 Scanner에 전달합니다.

Freestyle 프로젝트는 설정 값을 넘겨주면 알아서 Sonar Scanner를 실행하지만, Pipeline의 경우 직접 실행 스크립트를 작성해야 합니다. ``-D`` 를 통해 제공하는 파라미터는 적당히 본인의 상황에 맞게 잘 조절하면 됩니다.

> stage('SonarQube Analysis')

```groovy
stage('SonarQube analysis') {
  // requires SonarQube Scanner 2.8+
  def scannerHome = tool 'SonarQube Scanner 2.8';
  withSonarQubeEnv('My SonarQube Server') {
    sh "${scannerHome}/bin/sonar-scanner"
  }
}
```

코드가 너무 길다고 느껴지면 공식 문서를 참고해 축약하는 것을 시도해보시길 바랍니다.

> stage('SonarQube')

```groovy
stage('SonarQube') {
    steps {
        script {
            def qg = waitForQualityGate();
            if (qg.status != 'OK') {
               sh'''
                  echo "SonarQube Quality Failure"
               '''
               error "Pipeline aborted due to quality gate failure"
            }
            echo "All Passed!"
        }
    }
}
```

``waitForQualityGate``를 사용하면, SaonrQube 서버가 분석을 완료하고 보낸 응답 결과를 받아볼 수 있습니다. SonarQube 쪽의 프로젝트에서 반드시 이 글 초기 설정처럼 Jenkins 서버에 대한 ``<your Jenkins instance>/sonarqube-webhook`` Webhook을 지정해두어야 합니다.

<img width="1035" alt="스크린샷 2021-09-01 오전 6 25 25" src="https://user-images.githubusercontent.com/56240505/131577869-2f37d33b-276c-44a0-8c7d-133526815951.png">

SonarQube 서버가 정상적으로 분석을 처리하는 것을 확인할 수 있습니다.

### 10.1. 주의

> Log

```
/var/jenkins_home/tools/hudson.plugins.sonar.SonarRunnerInstallation/sonarQube-server/bin/sonar-scanner: not found
```

파이프라인 빌드할 때 위 예외가 발생하면, Jenkins EC2 내부의 볼륨 마운팅 디렉토리에 해당 경로가 존재하는지 확인합니다. 만약 없다면 신규 Pipeline 프로젝트 생성 후 다음과 같은 스크립트들을 빌드해보고, 해당 경로가 생겼는지 확인해봅시다.

> Pipeline Script

```groovy
stage("SonarQube analysis") {
        steps {
            script {
                def scannerHome = tool 'SonarQube Scanner';
                withSonarQubeEnv('지정한 소나 큐브 서버 이름') {
                    sh 'mvn clean package sonar:sonar'
                }
            }
        }
}
```

혹은

> Pipeline Script

```groovy
stage('SonarQube analysis') {
  withSonarQubeEnv('지정한 소나 큐브 서버 이름') {
    // requires SonarQube Scanner for Maven 3.2+
    sh 'mvn org.sonarsource.scanner.maven:sonar-maven-plugin:3.2:sonar'
  }
}
```

이래도 해결되지 않는다면 Sonar Scanner를 mvn으로 다운받는 방법들을 찾아 적용해보시길 바랍니다. ``/var/jenkins_home/tools/hudson.plugins.sonar.SonarRunnerInstallation/sonarQube-server/bin/sonar-scanner`` 디렉토리가 정상적으로 생성되면 기존의 SonarQube 스크립트가 잘 작동할 것입니다.

### 10.2. 기타

<img width="713" alt="스크린샷 2021-08-31 오후 3 01 50" src="https://user-images.githubusercontent.com/56240505/131578055-509cb132-2451-48c5-abef-af438d2fa3d7.png">

![image](https://user-images.githubusercontent.com/56240505/135467996-6c259683-127e-4c48-9120-766dfe788445.png)

파이프라인 구축에 큰 문제가 없다면, PR 생성시 Jenkins가 스크립트 빌드를 시작합니다. 이후 코드 커버리지 및 정적 분석 기준 충족 여하에 따라 해당 PR에 결과를 리포팅합니다.

<img width="900" alt="스크린샷 2021-09-01 오전 5 14 36" src="https://user-images.githubusercontent.com/56240505/131577650-cf45436a-5de9-4a3a-b3ae-7641045b9cab.png">

![image](https://user-images.githubusercontent.com/56240505/135468084-1aef9df1-15d5-44e3-a240-45d2dbb5a5e0.png)

정적 분석 Pipeline이 실패하는 경우 특정 브랜치로의 Merge를 금지하도록 브랜치 보호 규칙을 설정할 수 있습니다.

<br>

---

## Reference

* Special Thanks to [손너잘](https://github.com/bperhaps) & [다니](https://github.com/da-nyee)
* [코드 분석 도구 적용기 - 1편, 코드 커버리지(Code Coverage)가 뭔가요?](https://velog.io/@lxxjn0/%EC%BD%94%EB%93%9C-%EB%B6%84%EC%84%9D-%EB%8F%84%EA%B5%AC-%EC%A0%81%EC%9A%A9%EA%B8%B0-1%ED%8E%B8-%EC%BD%94%EB%93%9C-%EC%BB%A4%EB%B2%84%EB%A6%AC%EC%A7%80Code-Coverage%EA%B0%80-%EB%AD%94%EA%B0%80%EC%9A%94)
* [코드 분석 도구 적용기 - 2편, JaCoCo 적용하기](https://velog.io/@lxxjn0/%EC%BD%94%EB%93%9C-%EB%B6%84%EC%84%9D-%EB%8F%84%EA%B5%AC-%EC%A0%81%EC%9A%A9%EA%B8%B0-2%ED%8E%B8-JaCoCo-%EC%A0%81%EC%9A%A9%ED%95%98%EA%B8%B0)
* [Gradle 프로젝트에 JaCoCo 설정하기](https://woowabros.github.io/experience/2020/02/02/jacoco-config-on-gradle-project.html)
* [The JaCoCo Plugin - Gradle User Manual](https://docs.gradle.org/current/userguide/jacoco_plugin.html)
* [좌충우돌 jacoco 적용기](https://bottom-to-top.tistory.com/36)
* [SonarQube 정적분석 및 Jenkins CI/CD 통합](https://waspro.tistory.com/596)
* [SonarQube: MySQL, PostgreSQL 전환 및 버전 업](https://wickso.me/sonarqube/mysql-to-postgresql/)
* [Analyzing with SonarQube Scanner for Jenkins](https://sonarqubekr.atlassian.net/wiki/spaces/SCAN/pages/88768984/Analyzing+with+SonarQube+Scanner+for+Jenkins)
* [No tool named SonarQube Scanner 2.8 found error](https://stackoverflow.com/questions/52772631/no-tool-named-sonarqube-scanner-2-8-found-error)
* [Set a SonarQube webhook in Jenkinsfile](https://stackoverflow.com/questions/54428685/set-a-sonarqube-webhook-in-jenkinsfile)
* [빌드/테스트는 내가 해줄게. 너는 코딩에 집중해 (by GitHub Pull Request Builder)](https://taetaetae.github.io/2020/09/07/github-pullrequest-build/)
