---
layout: post
title: Log4j2 및 Logback의 Async Logging 성능 테스트 비교
date: 2021-09-29 13:32:20 +0300
description: 각 Logging Framework가 제공하는 비동기 로깅의 성능을 비교해봅시다.
img: log4j2.png
tags: [백엔드, Logging]
---

## 1. 들어가며

안녕하세요, 케빈입니다. Java 및 Spring 진영에서 사용할 수 있는 Logging Framework는 Log4j와 Logback 및 Log4j2 등 다양합니다. Pick-Git을 개발하면서 팀원들과 어떤 Logging Framework를 선택해야할지 고믾이 많았습니다. Logging Framework 종류별 특징을 학습하는 것에서 그치지 않고, 직접 Logging Framework들의 비동기 로깅(Async Logging) 성능을 테스트해보고 결과를 비교했습니다. 최종적으로 Log4j2를 선택하게 되었는데요. 의사 결정 과정에 대해 공유하고자 합니다.

<br>

## 2. Logging Framework 종류별 특징 및 관련 용어 정리

Logging Framework 종류별 특징 및 관련 용어 등을 먼저 정리해보고자 합니다. 이미 잘 알고 계시는 독자님들이라면 다음 절로 넘어가셔도 됩니다. 이번 절의 내용은 제가 예전에 정리한 [Logging 및 Spring Boot Logback 설정](https://xlffm3.github.io/spring%20&%20spring%20boot/logback-setting/#2-logging-%EA%B4%80%EB%A0%A8-%EC%9A%A9%EC%96%B4-%EC%A0%95%EB%A6%AC) 글에서 발췌했습니다.

초기의 Spring은 JCL(Jakarta Commons Logging)을 사용하여 로깅을 구현했습니다. JCL을 사용하면 기본적인 인터페이스인 Log와 Log 객체 생성을 담당하는 LogFactory만 구현하면 로깅 구현체 교체가 자유롭습니다. JCL은 GC가 제대로 작동하지 않는 등의 단점이 존재했는데요. 이 때 도입된 것이 SLF4J입니다.

SLF4J는 Java 진영의 다양한 로깅 프레임워크들에 대한 공용 인터페이스(Facade)입니다. 공용 인터페이스를 통해 특정 로깅 프레임워크에서 다른 로깅 프레임워크로 쉽게 전환할 수 있습니다. JCL이 지닌 문제 해결을 위해, 클래스 로더 대신에 컴파일 시점에 구현체를 선택하도록 변경되었습니다.

다음은 각 Logging Framework 종류별 특징입니다.

* Log4j : Apache의 Java 기반의 로깅 프레임워크입니다.
  * 가장 오래된 로깅 프레임워크며, 현재는 개발이 중단되었습니다.
* Logback : Log4j 이후에 출시된 로깅 프레임워크로, Spring Boot가 기본적으로 사용합니다.
  * Log4j보다 향상된 성능 및 필터링 옵션을 제공합니다.
  * 로그 레벨 변경 등에 대해 서버를 재시작할 필요 없이 자동 리로딩을 지원합니다.
* Log4j2 : 파일뿐만 아니라 HTTP, DB, Kafka에 로그를 남길 수 있으며 비동기적인 로거를 지원합니다.
  * 로깅 성능이 중요시될 때 Log4j2 사용을 고려합니다.
  * Log4j2 사용을 위해서는 spring-boot-starter-logging 모듈을 exclude하고 spring-boot-starter-logging-log4j2 의존성을 주입해야 합니다.
  * Logback과 달리 멀티 쓰레드 환경에서 비동기 로거(Async Logger)의 경우 Log4j 1.x 및 Logback보다 성능이 우수합니다.

<br>

## 3. Async Logging

> Test.java

```java
public void log() {
    for (int i = 0; i < 10; i++) {
        foo.bar();
        log.info("foo invoked");
    }
}
```

별도의 설정없이 로그 메시지를 남기면 동기적인 방식으로 파일 쓰기 작업이 진행됩니다. 즉, 로깅 메서드를 호출하는 시점(로그 발생)에 매번 파일 쓰기 작업이 수행된다. 파일 쓰기와 같은 I/O는 인메모리 작업이 아니라서 상대적으로 딜레이 및 부하가 큽니다.

반면 비동기 로깅을 사용하면 **로그 발생**과 **로그 쓰기**를 분리시킵니다. 로깅 메서드가 호출되면(로그 발생), Thread A는 인메모리 Queue 혹은 Buffer에 로그 정보를 집어넣습니다. Thread B는 Queue 혹은 Buffer에 쌓인 로그 데이터를 꺼내서 파일 쓰기만 수행합니다. 로깅 프레임워크마다 비동기 로깅 세부 구현이 다소 다르기 때문에 성능 차이가 발생할 수 있습니다.

### 3.1. 장점

로깅 메서드를 호출하는 시점(로그 발생)에 파일 I/O 작업이 바로 수행되지 않기 때문에, 어플리케이션의 성능이 빨라집니다.

### 3.2. 단점

Queue 혹은 Buffer의 용량이 80%를 도달하면, WARN 레벨 미만의 로그는 큐에서 제거되는 등 유실의 위험이 존재합니다. 또한 Queue 혹은 Buffer에 로그가 쌓인 상태에서 프로세스가 종료되면, 해당 로그는 기록되지 않습니다. 물론 옵션 조정을 통해 로깅 성능을 일부 타협하는 대신 로그 유실 가능성을 낮출 수 있습니다.

* Queue 혹은 Buffer 크기를 조절할 수 있습니다.
* Queue 혹은 Buffer가 꽉찬 경우 로그를 유실시킬지, 블락킹 상태에서 기다릴지 등을 선택할 수 있습니다.

또한 비동기 로깅은 기본적으로 Method Name 및 Line Number를 출력하지 않습니다. 이 또한 옵션 설정을 통해 출력을 가능하게 만들 수 있으나, 5 ~ 20배 정도 속도가 저하됩니다.

### 3.3. Sync & Async

> Test.js

```javascript
function a() {
    let result = b();
    console.log(result);
    console.log("a is done");
}

function b() {
    return 10;
}

/*
실행 결과
10
a is done
*/
```

Synchronous(동기)란 작업을 요청한 후 작업의 결과가 나올 때까지 기다린 후 처리하는 것을 의미합니다. A 함수가 B 함수를 호출했을 때, A 함수가 B 함수의 수행 결과 및 종료를 신경쓰는 경우를 예로 들 수 있습니다. 일반적인 경우 Blocking과 동일한 의미로 사용될 수 있습니다.

> Test.js

```javascript
function a() {
    fetch(url, options)
      .then(response => console.log("response arrives"))
      .catch(error => console.log("error thrown"));
    console.log("a is done");
}

/*
실행 결과
a is done
response arrives
*/
```

반면 Asynchronous(비동기)란 두 주체가 서로의 시작/종료 시간과는 관계 없이 별도의 수행 시작/종료시간을 가지고 있는 것을 의미합니다. A 함수가 B 함수를 호출했을 때, 호출된 함수의 수행 결과 및 종료를 호출된 함수 혼자 직접 신경 쓰고 처리하는 경우를 예로 들 수 있습니다. 대게 결과를 돌려주었을 때 순서와 결과(처리)에 관심이 있는지 아닌지로 판단할 수 있습니다.

<br>

## 4. Log4j2 Async Logger Performance

![image](https://user-images.githubusercontent.com/56240505/134123064-177446c8-864f-4528-aad6-a8644c0a092f.png)

[Log4j2 Performance](https://logging.apache.org/log4j/2.x/performance.html) 공식 문서에 기재된 여러 벤치마크 결과 지표에 따르면, Log4j2 **AsyncLogger**를 사용한 비동기 로깅의 성능이 가장 우수합니다. Logback 및 Log4j2 모두 AsyncAppender라고 하는 전통적인 비동기 로깅 기능을 지원하는데, 이는 ArrayBlockingQueue 를 사용합니다.

반면 비교적 나중에 공개된 Log4j2의 **AsyncLogger**는 LMAX Disruptor라는 Lock-Free Inter-Thread Communication 라이브러리를 사용함으로써, 더 높은 처리량 및 낮은 지연 시간을 달성했습니다.

실제로도 Log4j2의 AsyncLogger를 활용한 비동기 로깅이 Log4j2의 AsyncAppender 및 Logback의 AsyncAppender를 활용한 비동기 로깅보다 우수할까요? 개인적인 호기심이 생겨 테스트를 진행해보았습니다.

<br>

## 5. Log4j2 Async Logging 성능 테스트

### 5.1. 테스트 환경

> build.gradle

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-log4j2'
    implementation 'com.lmax:disruptor:3.4.2'

    modules {
        module("org.springframework.boot:spring-boot-starter-logging") {
            replacedBy("org.springframework.boot:spring-boot-starter-log4j2", "Use Log4j2 instead of Logback")
        }
    }

    //기타 Spring 관련 의존성 생략
}
```

Log4j2 및 Disruptor 의존성을 추가해주고, Spring Boot가 기본적으로 제공하는 ``spring-boot-starter-logging`` 모듈은 제외해줍니다.

> LogController.java

```java
@RequiredArgsConstructor
@RestController
public class LogController {

    private final LogService logService;

    @GetMapping("/log")
    public String log() {
        logService.log();
        return "log";
    }
}
```

> LogService.java

```java
@Slf4j
@Transactional(readOnly = true)
@Service
public class LogService {

    public void log() {
        for (int i = 0; i < 10_000; i++) {
            log.info("info log");
        }
    }
}
```

**최대한 Async Logging의 성능을 저하시키는 방향으로 Logger를 설정한 다음, nGrinder를 통해 테스트를 진행했습니다.** 10분동안 VUser 1000명이 ``GET /log``로 요청을 보냅니다.

### 5.2. Logback Async Logging (with AsyncAppender)

> logback-spring.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration debug="false">
  <include resource="org/springframework/boot/logging/logback/base.xml"/>
  <property name="home" value="logs/"/>

  <appender name="info-logger" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>${home}info.log</file>
    <append>false</append>
    <immediateFlush>false</immediateFlush>
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
      <fileNamePattern>${home}info-%d{yyyyMMdd}-%i.log</fileNamePattern>
      <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
        <maxFileSize>15MB</maxFileSize>
      </timeBasedFileNamingAndTriggeringPolicy>
      <maxHistory>30</maxHistory>
    </rollingPolicy>
    <encoder>
      <charset>utf8</charset>
      <Pattern>
        %d{yyyy-MM-dd HH:mm:ss.SSS} %-5level %t %class{36}.%M L:%L %n      > %m%n
      </Pattern>
    </encoder>
  </appender>

  <appender name="file-async" class="ch.qos.logback.classic.AsyncAppender">
    <appender-ref ref="info-logger"/>
    <includeCallerData>true</includeCallerData>
    <queueSize>1024</queueSize>
    <neverBlock>false</neverBlock>
    <discardingThreshold>0</discardingThreshold>
  </appender>

  <logger name="com.example.logging" level="INFO" additivity="false">
    <appender-ref ref="file-async"/>
  </logger>
</configuration>
```

먼저 Logback AsyncAppender를 사용하는 비동기 로깅을 테스트를 진행했습니다. 추후 테스트할 Log4j2의 AsyncAppender와 최대한 비슷하게 동작하도록 옵션들을 설정했습니다.

* RollingFileAppender
  * append : true(기본값)면 로그 출력이 기존 파일에 추가되며, false면 기존 파일의 내용이 모두 삭제되고 출력이 쌓입니다.
  * immediateFlush : true(기본값)면 로그 메세지들이 전혀 버퍼되지 않습니다.
* AsyncAppender
  * includeCallerData : 비동기 로깅에서도 Method Name 및 Line Number 등 위치 정보를 출력하게 해주는 옵션입니다.
  * queueSize : 기본값은 256지만 Log4j2의 기본값과 동일하게 1024로 설정합니다.
  * neverBlock : false(기본값)면 로그 발생시 Queue에 넣을 공간이 없으면 빈 공간이 생길 때 까지 블락킹 상태로 기다리며, 로그를 유실하지 않습니다.
  * discardingThreshold : Queue의 남은 용량이 **{해당 설정값 n}%** 이하가 되면, WARN 미만 로그가 유실되기 시작합니다.
    * 기본 값은 20이며, Queue에 남은 용량이 20% 이하가 되면 로그 유실이 발생합니다.
    * 0으로 세팅하면 Queue에 쌓인 로그를 드랍하지 않습니다.

![image](https://user-images.githubusercontent.com/56240505/134135579-664a9a11-309a-4168-92b7-62f0bd787d56.png)

TPS는 평균 4.0, 최대 6.0입니다.

![image](https://user-images.githubusercontent.com/56240505/134135724-0579f377-291a-4583-80f6-da7a566c9dd8.png)

MTT는 125,214입니다.

![image](https://user-images.githubusercontent.com/56240505/134135830-21423404-b447-4a4d-8254-3c4db55cdf2c.png)

총 수행된 테스트는 3,916개인데, 이 중 에러는 1,608개로 에러율이 41%였습니다. 후술할 Log4j2의 동기적인 로깅 방식보다 성능이 나쁜 점을 볼 때, 아무래도 AsyncAppender를 너무 나쁘게 설정한 것 같습니다. 😅

### 5.3. Log4j2 Sync Logging

> log4j2.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration>
  <Properties>
    <Property name="log-path">logs/log</Property>
    <Property name="log-pattern">%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level %t %class{36}.%M L:%L %n >
      %m%n
    </Property>
    <Property name="file-pattern">%d{yyyy-MM-dd}-%i.log.gz</Property>
  </Properties>

  <Appenders>
    <RollingFile name="FileAppender" fileName="${log-path}.log"
      filePattern="${log-path}-${file-pattern}" append="false">
      <PatternLayout pattern="${log-pattern}"/>
      <Policies>
        <SizeBasedTriggeringPolicy size="15MB"/>
        <TimeBasedTriggeringPolicy modulate="true" interval="1"/>
      </Policies>
      <DefaultRolloverStrategy>
        <Delete basePath="logs/log" maxDepth="1">
          <IfLastModified age="30d"/>
        </Delete>
      </DefaultRolloverStrategy>
    </RollingFile>
  </Appenders>

  <Loggers>
    <Logger name="com.example.logging" level="info" additivity="false">
      <AppenderRef ref="FileAppender"/>
    </Logger>
  </Loggers>
</Configuration>
```

이번에는 Log4j2를 사용하되 비동기 로깅이 아닌 동기적인 로깅 방식으로 테스트를 진행했습니다.

![image](https://user-images.githubusercontent.com/56240505/134126197-0f43123e-b6b2-4e02-bd9e-40913f0cb6d8.png)

TPS는 평균 5.6, 최대 8.0입니다.

![image](https://user-images.githubusercontent.com/56240505/134126348-626818d8-946f-423f-b102-e157b5c82e86.png)

MTT는 126,803입니다.

![image](https://user-images.githubusercontent.com/56240505/134126450-cc714e0e-824a-4c0b-8790-7caba31756f3.png)

총 수행된 테스트는 3,891개인데, 이 중 에러는 684개로 에러율이 17.6%였습니다.

> Log

```
org.springframework.transaction.CannotCreateTransactionException: Could not open JPA EntityManager for transaction; nested exception is org.hibernate.exception.JDBCConnectionException: Unable to acquire JDBC Connection, Could not open JPA EntityManager for transaction; nested exception is org.hibernate.exception.JDBCConnectionException: Unable to acquire JDBC Connection
```

하나의 트랜잭션이 커넥션을 점유하면서 1만번 로그 파일 쓰기 I/O 작업을 진행하는데요. 동기적인 처리로 인해 트랜잭션 수행 시간이 길어지다 보니, DB 커넥션을 30초 이상 얻지 못해 예외를 던지는 스레드가 많이 발생했습니다.

### 5.4. Log4j2 Async Logging (with AsyncAppender)

> log4j2.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration>
  <Properties>
    <Property name="log-path">logs/log</Property>
    <Property name="log-pattern">%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level %t %class{36}.%M L:%L %n >
      %m%n
    </Property>
    <Property name="file-pattern">%d{yyyy-MM-dd}-%i.log.gz</Property>
  </Properties>

  <Appenders>
    <RollingFile name="FileAppender" fileName="${log-path}.log"
      filePattern="${log-path}-${file-pattern}" immediateFlush="false" append="false">
      <PatternLayout pattern="${log-pattern}"/>
      <Policies>
        <SizeBasedTriggeringPolicy size="15MB"/>
        <TimeBasedTriggeringPolicy modulate="true" interval="1"/>
      </Policies>
      <DefaultRolloverStrategy>
        <Delete basePath="logs/log" maxDepth="1">
          <IfLastModified age="30d"/>
        </Delete>
      </DefaultRolloverStrategy>
    </RollingFile>

    <Async name="asyncAppender" includeLocation="true">
      <AppenderRef ref="FileAppender"/>
    </Async>
  </Appenders>

  <Loggers>
    <Logger name="com.example.logging" level="info" additivity="false">
      <AppenderRef ref="asyncAppender"/>
    </Logger>
  </Loggers>
</Configuration>
```

Log4j2의 AsyncAppender를 활용한 비동기 로깅을 테스트했습니다.

* RollingFile
  * append 및 immediateFlush 옵션은 Logback과 동일합니다.
* AsyncAppender
  * includeLocation : Logback의 includeCallerData 옵션과 동일합니다.
    * 기본값은 false이고, 사용하게 될 경우 속도가 5 ~ 20배 정도 느려집니다.
  * bufferSize : 기본값은 1024입니다.
  * blocking : true(기본값)면 로그 발생시 Queue에 넣을 공간이 없으면 빈 공간이 생길 때 까지 블락킹 상태로 기다리며, 로그를 유실하지 않습니다.
    * false면 로그 발생시 Queue가 꽉 찼다면 로그를 유실하고, Error Appender에 이벤트를 기록합니다.

![image](https://user-images.githubusercontent.com/56240505/135132454-449e53bd-17f1-4b72-b50a-11f7958c7e42.png)

TPS는 평균 7.0, 최대 9.0입니다.

![image](https://user-images.githubusercontent.com/56240505/135132564-27cd235e-8037-4cec-8397-ec38e0d294a6.png)

MTT는 112,305입니다.

![image](https://user-images.githubusercontent.com/56240505/135132641-ddce4e11-3196-4a33-bf04-d4eba3fcc990.png)

총 수행된 테스트는 4,506개인데, 이 중 에러는 488개로 에러율이 10.8%였습니다. 확실히 Logback의 AsyncAppender 및 Log4j2의 Sync Logging보다 성능이 좋습니다.

### 5.5. Log4j2 Async Logging (with AsyncLogger)

> log4j2.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration>
  <Properties>
    <Property name="log-path">logs/log</Property>
    <Property name="log-pattern">%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level %t %class{36}.%M L:%L %n >
      %m%n
    </Property>
    <Property name="file-pattern">%d{yyyy-MM-dd}-%i.log.gz</Property>
  </Properties>

  <Appenders>
    <RollingFile name="FileAppender" fileName="${log-path}.log"
      filePattern="${log-path}-${file-pattern}" immediateFlush="false" append="false">
      <PatternLayout pattern="${log-pattern}"/>
      <Policies>
        <SizeBasedTriggeringPolicy size="15MB"/>
        <TimeBasedTriggeringPolicy modulate="true" interval="1"/>
      </Policies>
      <DefaultRolloverStrategy>
        <Delete basePath="logs/log" maxDepth="1">
          <IfLastModified age="30d"/>
        </Delete>
      </DefaultRolloverStrategy>
    </RollingFile>
  </Appenders>

  <Loggers>
    <AsyncLogger name="com.example.logging" level="info" additivity="false" includeLocation="true">
      <AppenderRef ref="FileAppender"/>
    </AsyncLogger>
  </Loggers>
</Configuration>
```

마지막으로 Log4j2의 AsyncLogger를 활용한 비동기 로깅을 테스트했습니다.

![image](https://user-images.githubusercontent.com/56240505/134131438-0bdad77e-207a-481d-998b-bc46e4688ebd.png)

TPS는 평균 16.3, 최대 19.5입니다.

![image](https://user-images.githubusercontent.com/56240505/134131614-db96e3e3-426c-480a-8993-9256cd413b0f.png)

MTT는 58,116입니다.

![image](https://user-images.githubusercontent.com/56240505/134131729-f8201697-495e-46d6-80e4-6d79ee4c43c5.png)

총 수행된 테스트는 9,351개인데, 이 중 에러는 32개로 에러율이 0.3%였습니다.

![image](https://user-images.githubusercontent.com/56240505/134138570-bb58ae2c-1100-47ab-9b85-711554384dc2.png)

성능이 저하될 수 있는 옵션을 끄고 테스트를 진행한 결과, TPS는 평균 68.2 및 최대 77로 매우 높게 향상되는 것을 확인할 수 있었습니다.

<br>

## 6. 마치며

Logback의 AsyncAppender와 Log4j2의 AsyncAppender 및 Log4j2의 AsyncLogger 등의 비동기 로깅 성능 테스트를 비교 분석해보았습니다. 공식 문서의 벤치 마크 지표를 보고 호기심이 생겨 시작한 실험이었는데, 의의보다는 한계가 더 큰 것 같습니다.

테스트를 위해 각 Logging Framework들이 최대한 비슷하게 동작하도록 여러 옵션들을 설정했는데요. 결과적으로는 AsyncLogger의 성능이 가장 우수하다는 결론을 도출할 수 있었지만, 옵션 설정과 같은 여러 테스트 통제 환경이 비동기 로깅 성능들을 객관적으로 비교하는데 충분했는지 의문입니다. Logging Framework에서 제공하는 비동기 로깅 기능에 대한 이해도가 부족한만큼, 통제하지 못한 변인들도 많다고 생각됩니다. 제 스스로도 확실하지 않아 테스트를 신뢰할 수 없다는 점이 아쉽습니다. 😭

그럼에도 불구하고 한 가지 경향성을 확인할 수 있었습니다. 로그가 유실되지 않도록 설정하더라도, 트랜잭션에서 로그 발생이 잦다면 비동기 로깅을 채택하는 것이 TPS 및 에러율 등 성능 측면에서 동기적인 로깅 방식보다 유리하다는 점이었습니다. 비동기 로깅의 경우, 로그 파일 쓰기 I/O 작업이 즉시 진행되지 않아 트랜잭션 수행 시간이 짧아지면서 스레드간 DB 커넥션 회전이 원할해졌습니다. 따라서 기존의 동기적인 로깅 방식에서 발생하던 Hikari CP Dead Lock 에러가 많이 줄어든 것을 확인할 수 있었습니다.

비동기 로깅 방식을 채택함으로써 로그 유실로 인한 신뢰성 문제 등 관리 포인트가 늘어나지만, 성능이 중요한 어플리케이션이라면 충분히 고려해볼만한 기능이라고 생각됩니다. Pick-Git 팀은 Log4j2가 Logback보다 비동기 성능이 우수하며 다양한 Appender(Kafka, DB, HTTP)를 제공하는 만큼 확장성이 좋다고 판단해, Log4j2를 사용하기로 최종 결정했습니다.

<br>

---

## Reference

* [Log4j2 공식 문서](https://logging.apache.org/log4j/2.x/manual/async.html)
* [Logback 공식 문서](http://logback.qos.ch/manual/appenders.html)
* [Logback AsyncAppender](https://kwonnam.pe.kr/wiki/java/logback/asyncappender)
