---
layout: post
title: M1(ARM)에서 Embedded Redis를 사용하는 방법
date: 2021-10-20 16:21::48 +0900
description: M1(ARM)에서 Embedded Redis를 사용하는 방법을 알아봐요.
img: redis-on-m1.png
tags: [백엔드, Database, Embedded Redis, M1, ARM]
---

안녕하세요, 다니입니다. 🌻

<br/>

## 이슈 1

### 문제 상황

우아한테크코스 팀 프로젝트를 진행하며 Redis를 도입했다.<br/>
테스트 환경에서는 Embedded Redis가 동작하게 구성했는데, M1에서는 해당 DB가 실행되지 않는 현상이 발생했다.<br/>
백엔드 팀원 대부분이 M1을 사용하고 있어 이를 해결하는 게 우선시 됐다.<br/>

관련 내용을 찾아보다가

- Embedded Redis 라이브러리에서 mac_arm64용 바이너리가 준비되어 있지 않다.
- 또, 소스 코드에도 MAC_OS_X_arm64가 없다.

이와 같은 사실을 알게 됐다.<br/>

참고한 글에 따르면, Embedded Redis 라이브러리를 수정해서 사용할 수도 있다고 한다.<br/>
하지만, 공식 문서에서는 사용자가 직접 바이너리를 지정해서 이용하는 걸 추천한다고 설명했다.<br/>

그래서 `RedisServer` 객체를 활용하는 방식을 선택했다.<br/>

### 해결 방법

먼저 Redis 소스 코드를 다운받고 컴파일한다.<br/>
아래 과정을 거치면, `redis-6.2.5/src` 경로에 바이너리 파일들이 빌드되어 준비된다.<br/>

```bash
// 소스 코드 다운로드
% wget https://download.redis.io/releases/redis-6.2.5.tar.gz

// 압축 해제
% tar xzf redis-6.2.5.tar.gz 

// redis-6.2.5 디렉토리 이동
% cd redis-6.2.5

// 파일 링크 및 설치 등
% make
```

<br/>

다음으로 `redis-server` 바이너리 파일을 실행한다.<br/>

```bash
% src/redis-server

39720:C 20 Oct 2021 14:28:50.565 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
39720:C 20 Oct 2021 14:28:50.565 # Redis version=6.2.5, bits=64, commit=d9e6d08d, modified=0, pid=39720, just started
39720:C 20 Oct 2021 14:28:50.565 # Warning: no config file specified, using the default config. In order to specify a config file use src/redis-server /path/to/redis.conf
39720:M 20 Oct 2021 14:28:50.566 * monotonic clock: POSIX clock_gettime
                _._                                                  
           _.-``__ ''-._                                             
      _.-``    `.  `_.  ''-._           Redis 6.2.5 (d9e6d08d/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._                                  
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 39720
  `-._    `-._  `-./  _.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |           https://redis.io       
  `-._    `-._`-.__.-'_.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |                                  
  `-._    `-._`-.__.-'_.-'    _.-'                                   
      `-._    `-.__.-'    _.-'                                       
          `-._        _.-'                                           
              `-.__.-'                                               

39720:M 20 Oct 2021 14:28:50.569 # Server initialized
39720:M 20 Oct 2021 14:28:50.569 * Ready to accept connections 
```

<br/>

앞에서 `redis-server`가 잘 실행됐다면, 해당 파일의 이름을 변경하여 프로젝트에 추가한다.<br/>
이때, resources 디렉토리 하위에 복사해야 한다.<br/>

<p align="center">
    <img style="width: 490px; height: 180px" src="https://user-images.githubusercontent.com/50176238/138035778-731a34b3-995a-4244-9913-d657a5d5a7ac.png">
</p>

<br/>

마지막으로 `EmbeddedRedisConfiguration` 클래스에 아래 코드를 추가한다.<br/>
여기까지 하면, M1에서도 Embedded Redis가 잘 작동된다.<br/>

```java
@PostConstruct
public void redisServer() throws IOException, URISyntaxException {
    int redisPort = isRedisRunning() ? findAvailablePort() : port;

    if (isArmMac()) {
        redisServer = new RedisServer(
            Objects.requireNonNull(getRedisFileForArcMac()),
            redisPort
        );
    }
    if (!isArmMac()) {
        redisServer = new RedisServer(redisPort);
    }

    redisServer.start();
}

private boolean isArmMac() {
    return Objects.equals(System.getProperty("os.arch"), "aarch64") &&
        Objects.equals(System.getProperty("os.name"), "Mac OS X");
}

private File getRedisFileForArcMac() {
    try {
        return new ClassPathResource("binary/redis/redis-server-6.2.5-mac-arm64")
            .getFile();
    } catch (Exception e) {
        throw new EmbeddedRedisServerException();
    }
}
```

<br/>

## 이슈 2

### 문제 상황

그런데, 나는 위의 방법을 적용해도 테스트가 계속 실패했다.<br/>
어떤 부분을 놓쳤는지 원인을 분석했는데, JDK 버전이 호환되지 않아 터진 것이었다.<br/>
기존에는 default 버전을 쓰고 있었는데, `Azul Zulu Community(M1용)` 버전으로 바꿔야 했다.<br/>

### 해결 방법

해당 버전으로 변경하고 실행해보니까, 이번에는 정말로 문제 없이 잘 작동했다.<br/>
테스트 실행 속도가 이전보다 빨라진 건 덤이었다!<br/>

<p align="center">
    <img src="https://user-images.githubusercontent.com/50176238/137240384-f05b3448-dc53-4ee6-b044-972c7a19d94f.png">
</p>

<br/>

## References

- [ARM Mac (M1) 에서 embedded-redis 사용하기](https://velog.io/@hakjong/ARM-Mac-M1-%EC%97%90%EC%84%9C-embedded-redis-%EC%82%AC%EC%9A%A9)
- [Redis - Download](https://redis.io/download)
- [Apple Silicon M1 용 어플 호환성+JDK](https://cholchori.tistory.com/2167)