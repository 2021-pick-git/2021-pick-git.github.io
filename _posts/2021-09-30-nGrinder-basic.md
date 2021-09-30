---
layout: post
title: nGrinder 설치 방법 및 파일 업로드 테스트 예제
date: 2021-09-30 13:32:20 +0300
description: nGrinder를 통해 스트레스 테스트를 진행해봅시다.
img: ngrinder.png
tags: [백엔드, nGrinder, Test]
---

## 1. nGrinder

안녕하세요, 케빈입니다. nGrinder는 Naver가 개발한 오픈 소스 프로젝트로서, 어플리케이션의 성능을 측정할 수 있는 스트레스 테스트 플랫폼을 제공합니다. Grinder 오픈 소스를 기반으로 하며 Jython으로 개발되었는데요. nGrinder를 통해 서비스의 가용성 및 임계점 등을 확인할 수 있습니다.

nGrinder 이외에도 JMeter, K6, Gatling 등 다양한 성능 측정 도구가 있습니다. 이 중 nGrinder를 선택한 이유는 groovy 스크립트 기반의 테스트 작성 및 강력한 시각화 기능 제공 등의 장점 때문이었습니다. 아무래도 Gradle 및 Jenkins를 통해 groovy를 자주 다루었던 만큼 가장 친숙했습니다. 또한 공식 문서 내용이 풍부하고 관련 포럼이 활성화되어 있다는 점 또한 한 몫했습니다.

<br>

## 2. Architecture

nGrinder를 구성하는 요소는 크게 Controller와 Agent 및 Target 등 세 가지가 존재합니다.

* Controller
  * 전반적인 작업이 Controller를 통해 수행됩니다.
  * 스트레스 테스트를 위한 웹 인터페이스를 제공합니다.
  * 테스트 프로세스를 체계화하며, 테스트 결과를 수집해 통계를 보여줍니다.
  * Controller를 통해 사용자는 테스트 수행을 위한 스크립트를 생성 및 수정할 수 있습니다.
* Agent
  * Controller의 명령을 받아 작업을 수행합니다.
  * Agent 모드시 프로세스 및 스레드를 실행시켜 Target 머신에 부하를 발생시킵니다.
  * Monitor 모드시 대상 시스템의 CPU 및 Memory 등을 모니터링합니다.
* Target
  * 테스트 대상이 되는 머신입니다.

### 2.1. 설치 방법

이번 글에서는 Docker를 활용해 EC2에 설치할 계획입니다. 자세한 내용은 [DockerHub](https://hub.docker.com/r/ngrinder/controller/)를 참조하길 바랍니다. 원한다면 Docker 없이 [Controller 및 Agent를 WAR 파일로 다운받아 실행할 수 있습니다.](https://github.com/naver/ngrinder/wiki/Installation-Guide)

> Shell

```bash
$ docker pull ngrinder/controller
$ docker run -d -v ~/ngrinder-controller:/opt/ngrinder-controller --name controller \
-p 80:80 -p 16001:16001 -p 12000-12009:12000-12009 ngrinder/controller
```

EC2에 Controller 컨테이너를 실행시키며, 필요한 포트 포워딩 및 볼륨 마운팅을 지정합니다.

* 80번 포트는 Controller의 기본 Web UI 포트입니다.
* 9010 - 9019 포트를 통해 Agent들이 Controller 클러스터로 연결됩니다.
* 12000 - 12029 포트는 테스트 실행 및 종료 등 컨트롤러 명령어와 에이전트별 테스트 실행 통계를 초별로 수집하는 포트입니다.
  * 컨트롤러는 해당 포트를 통해 스트레스 테스트를 할당합니다.
* 16001 포트를 통해 테스트를 하지 않는 유휴 상태의 Agent가 Controller에게 테스트 가능 메시지를 전달합니다.

> Shell

```bash
$ docker pull ngrinder/agent
```

공식 문서는 Controller 컨테이너가 동작 중인 머신과 Agent 컨테이너를 구동하지 말것을 강력하게 권고합니다. 여러 Agent들이 동작하다 보면, 부하를 발생시키는 머신의 자원을 모두 소모할 수 있기 때문입니다.

> Shell

```bash
$ docker run -d --name agent --link controller:controller ngrinder/agent
```

그럼에도 불구하고 편의상 Controller 컨테이너가 동작 중인 EC2에 Agent를 구동해야 한다면 위 명령어를 입력하면 됩니다. 여러 Agent 컨테이너를 띄워도 됩니다.

> Shell

```bash
$ docker run -v ~/ngrinder-agent:/opt/ngrinder-agent -d ngrinder/agent {controller-ec2-ip}:{controller-ec2-web-port}
```

별도의 EC2에서 Agent를 동작시킨다면 위처럼 Controller EC2 IP 및 Controller EC2 Web Port를 함께 인자로 넣어 컨테이너를 실행합니다. 이 때, 볼륨 마운팅 옵션은 생략해도 좋습니다.

![image](https://user-images.githubusercontent.com/56240505/133430489-c5c18ee3-6309-4f57-b9d2-e8795b5b279e.png)

``http://{controller-ec2-ip}``로 nGrinder Controller 웹 페이지로 접속해봅니다. 저는 설명글 편의상 localhost에 띄웠습니다.

``ID : admin, Password : admin``으로 로그인합니다.

![image](https://user-images.githubusercontent.com/56240505/133431769-c8e41d4a-0eb4-4966-8e88-7337e2df6570.png)

``계정 정보 - 에이전트 관리`` 탭에 지정한 Agent들이 Controller에 잘 연결되었는지 확인합니다. Agent가 없다면 구동한 Agent들이 Controller에 붙지 못한 것이니 설정을 재점검해야합니다.

<br>

## 3. Quick Start

![스크린샷 2021-09-15 오후 9 20 25](https://user-images.githubusercontent.com/56240505/133432384-3f02aec8-1d79-4389-9e78-2dc871d74378.png)

이제 간단하게 nGrinder를 실행해보겠습니다. 메인 페이지의 검색란에 부하 테스트를 진행할 URL을 입력하고 ``테스트 시작`` 버튼을 누릅시다.

![스크린샷 2021-09-15 오후 9 23 53](https://user-images.githubusercontent.com/56240505/133433024-59d2bd96-f3f3-4597-b916-4fc534271fb5.png)

여러 메뉴들에 대해 간략하게 정리했습니다.

* ``테스트명`` : 추후 해당 테스트 기록을 식별하는데 사용합니다.
* ``Agent`` : 해당 테스트에 사용할 Agent 개수를 선택할 수 있습니다.
* ``VUser`` : Agent 별 VUser 개수를 선택할 수 있습니다.
  * 사용할 수 있는 최대 VUser의 총합 개수는 ``Agent 개수 * Agent 별 VUser 개수``입니다.
* 해당 부하 테스트를 수행할 기간 및 실행 회수 등을 세밀하게 조정할 수 있습니다.
* ``저장 후 시작 버튼``을 통해 테스트를 실행합니다.
  * 테스트를 바로 시작할 수 있으며, 특정 시간에 예약을 걸어둘 수 있습니다.

![image](https://user-images.githubusercontent.com/56240505/133433651-bd6b5bbd-78b4-41ba-827b-500a71f8f16c.png)

Quick Start로 테스트를 시작하면 nGrinder가 ``target-url/`` 디렉토리 경로에 ``TestRunner.groovy`` 테스트 스크립트를 기본적으로 생성해줍니다.

원한다면 nGrinder에 작성해둔 다른 테스트 스크립트를 선택해서 사용할 수 있는데요. 우측의 ``R HEAD``를 눌러서 스크립트를 편집할 수 있습니다.

![image](https://user-images.githubusercontent.com/56240505/133434172-81d18a84-5199-4b18-bf03-c2bf5d820873.png)

테스트 스크립트는 groovy 문법을 사용하고 있어서 크게 어렵지 않습니다. 테스트할 API URL, Request Header, Request Body, Response Assertion, Logging 등을 스크립트에서 설정할 수 있습니다.

![image](https://user-images.githubusercontent.com/56240505/133441067-66d8877f-b0f9-4828-9e2f-849dc74a8aaa.png)

테스트를 시작하면 실시간 TPS 그래프가 화면에 시각화되며, 테스트가 종료되면 요약 결과가 집계됩니다. 아울러 좌측 하단에서 테스트 스크립트 로그 다운로드가 가능합니다.

![image](https://user-images.githubusercontent.com/56240505/133442499-9e417c45-c27c-4cde-bf7e-df0f3e855b62.png)

상세 보고서를 클릭하면 TPS 이외의 지표들을 확인할 수 있으며, CSV 다운로드를 제공합니다.

> TestRunner.groovy

```java
@Test
public void test() {
  HTTPResponse response = request.GET("https://google.com", params)

  if (response.statusCode == 301 || response.statusCode == 302) {
    grinder.logger.warn("Warning. The response may not be correct. The response code was {}.", response.statusCode)
  } else {
    assertThat(response.statusCode, is(200))
  }
}
```

샘플 테스트 스크립트의 Test 메서드에는 응답 결과에 대한 Assertion 검증이 있는 것을 확인할 수 있습니다.

![image](https://user-images.githubusercontent.com/56240505/133441864-38d9a77e-83c7-48a8-ad25-80239a215791.png)

테스트 수행시 Assertion 검증이 실패하면 ``에러``로 잡히게 됩니다. 통상적으로 너무 많은 에러가 발생하는 경우 테스트가 중단됩니다.

### 3.1. 주의사항

![image](https://user-images.githubusercontent.com/56240505/133446468-7994932d-d4d1-4fc4-ac57-40f48753aa09.png)

Quick Start가 생성해준 ``TestRunner.groovy`` 파일에 테스트 스크립트를 작성(Overwrite)하는 경우, **날려먹기 정말 정말 쉽습니다.**

Quick Start로 ``https://test.com/api/comments`` URL에 대해 테스트를 시작했다고 가정해보겠습니다. nGrinder는 ``test.com/api/comments/`` 디렉토리에 ``TestRunner.groovy`` 스크립트를 생성합니다. 해당 스크립트 파일에 테스트 수행 코드를 작성하고 저장하더라도, 이후 메인 페이지에서 실수로 ``https://test.com/api/comments`` 동일 URL에 대해 Quick Start 테스트 시작 버튼을 누르면 어떻게 될까요?

nGrinder가 제공하는 기본 ``TestRunner.groovy`` 파일이 개발자가 커스터마이징한 ``TestRunner.groovy`` 파일을 덮어써버립니다! Quick Start는 무조건 ``target-url/`` 디렉토리에 ``TestRunner.groovy`` 파일을 새로 생성하기 때문입니다.

또한 TestRunner라는 스크립트 파일 이름이 의미하는 바가 모호하기 때문에, 테스트 스크립트를 관리하기 복잡해집니다.

![image](https://user-images.githubusercontent.com/56240505/133440066-749b22f9-c74d-453c-a072-4968d953d43a.png)

따라서 웹 상단의 ``스크립트`` 메뉴에서 직접 테스트 스크립트를 작성하는 것이 낫습니다. 테스트 스크립트의 이름을 적절하게 지어주면 나중에 테스트에서 원하는 스크립트를 찾기 수월하합니다.

아울러 부하 테스트 또한 웹 상단의 ``성능 테스트``에서 직접 생성합시다.

<br>

## 4. 파일 업로드 테스트 예제

> PostRequest.java

```java
public class PostRequest {

    private List<MultipartFile> images;
    private String githubRepoUrl;
    private List<String> tags;
    private String content;

    private PostRequest() {
    }

    //생성자 및 Getter 생략
}
```

> PostController.java

```java
@Slf4j
@RestController
public class PostController {

    @PostMapping(value = "/api/abc")
    public ResponseEntity<Void> write(PostRequest request) {
        log.warn("{} {} {}", request.getImages(), request.getTags(), request.getContent());
        if (request.getImages() == null) {
            throw new IllegalStateException();
        }
        return ResponseEntity.ok().build();
    }
}
```

MultipartFile 요청을 처리할 수 있는 컨트롤러를 작성하고 Spring 어플리케이션을 실행해봅시다.

![image](https://user-images.githubusercontent.com/56240505/133446714-a8278e90-d5df-45d1-adb3-c5286b7b219b.png)

nGrinder가 업로드 테스트에 사용할 테스트용 정적 파일을 업로드하는 단계입니다. 테스트 스크립트와 동일한 위치에 ``resources`` 디렉토리를 생성하고, 해당 폴더 내부에 업로드에 사용할 파일을 업로드해둡시다.

> PostWrite.groovy

```java
import static net.grinder.script.Grinder.grinder
import static org.junit.Assert.*
import static org.hamcrest.Matchers.*
import net.grinder.script.GTest
import net.grinder.script.Grinder
import net.grinder.scriptengine.groovy.junit.GrinderRunner
import net.grinder.scriptengine.groovy.junit.annotation.BeforeProcess
import net.grinder.scriptengine.groovy.junit.annotation.BeforeThread
// import static net.grinder.util.GrinderUtils.* // You can use this if you're using nGrinder after 3.2.3
import org.junit.Before
import org.junit.BeforeClass
import org.junit.Test
import org.junit.runner.RunWith

import org.ngrinder.http.HTTPRequest
import org.ngrinder.http.HTTPRequestControl
import org.ngrinder.http.HTTPResponse
import HTTPClient.Codecs
import HTTPClient.NVPair
import org.ngrinder.http.cookie.Cookie
import org.ngrinder.http.cookie.CookieManager

/**
* A simple example using the HTTP plugin that shows the retrieval of a single page via HTTP.
*
* This script is automatically generated by ngrinder.
*
* @author admin
*/
@RunWith(GrinderRunner)
class PostWrite {

	public static GTest test
	public static HTTPRequest request
	public static NVPair[] headers = []
	public static List<Cookie> cookies = []

	@BeforeProcess
	public static void beforeProcess() {
		HTTPRequestControl.setConnectionTimeout(300000)
		test = new GTest(1, "32.232.168.41")
		request = new HTTPRequest()
		grinder.logger.info("before process.")
	}

	@BeforeThread
	public void beforeThread() {
		test.record(this, "test")
		grinder.statistics.delayReports = true
		grinder.logger.info("before thread.")
	}

	@Before
	public void before() {
		headers = [new NVPair("Authorization", "Bearer abc.def.xyz"), new NVPair("Content-Type", "multipart/form-data")]
		request.setHeaders(headers)
		CookieManager.addCookies(cookies)
		grinder.logger.info("before. init headers and cookies")
	}

	@Test
	public void test() {		
		NVPair param1 = new NVPair("githubRepoUrl", "https://github.com/test");
		NVPair param2 = new NVPair("content", "content");
		NVPair param3 = new NVPair("tags", "typescript");
		NVPair param4 = new NVPair("tags", "html");

		NVPair[] params = [param1, param2, param3, param4];
		NVPair[] files = [new NVPair("images", "./resources/test.png"), new NVPair("images", "./resources/test.png")];

		def data = Codecs.mpFormDataEncode(params, files, headers)
		HTTPResponse response = request.POST("http://32.232.168.41:8080/api/abc", data);

		if (response.statusCode == 301 || response.statusCode == 302) {
			grinder.logger.warn("Warning. The response may not be correct. The response code was {}.", response.statusCode)
		} else {
			assertThat(response.statusCode, is(200))
		}
	}
}
```

이제 테스트 스크립트를 작성할 차례입니다. 스크립트를 완성하면 스크립트 편집기 우측 상단의 ``검증`` 버튼을 눌러 API 서버와 잘 통신되는지 확인해봅시다. 아래는 코드에 대한 간략한 설명입니다.

* 헤더에 Authorization 및 Content-Type를 넣었습니다.
* ``/api/abc`` API 스펙대로 Request Body에 들어갈 ``multipart/form-data`` 형식의 데이터(Key-Value)를 정의합니다.
  * 업로드할 MultipartFile의 경우, Value는 업로드할 파일의 상대 경로가 됩니다.
* ``Codecs.mpFormDataEncode`` 메서드는 Key-Value 쌍의 데이터 및 파일을 ``multipart/form-data`` 인코딩을 통해 바이트 배열로 인코딩합니다.
  * 자세한 내용은 [Docs](http://grinder.sourceforge.net/g3/script-javadoc/HTTPClient/Codecs.html)를 참고하시길 바랍니다.

### 4.1. 삽질의 시작

![image](https://user-images.githubusercontent.com/56240505/133449208-e41b41d8-ddfd-4e6b-a411-0cba791eeb45.png)
![image](https://user-images.githubusercontent.com/56240505/133448867-7b0d583c-910d-408d-a958-3f6288facb9e.png)

그러나 테스트 스크립트를 검증해보면 예상했던 200 응답이 아닌 500 응답을 받았습니다. Multipart Servlet Request를 Resolve할 수 없다는 로그가 발생했는데요. ``multipart boundary``가 표기되지 않아서 생기는 에러라고 합니다.

Spring 어플리케이션의 Access Log를 보니 Authorization 및 Content-Type 헤더는 정상적으로 잘 적용된 것을 확인할 수 있었습니다. 또한 Request Body에 바이너리 데이터가 잘 들어와있다는 것을 추측할 수 있었는데요. Content-Length가 과하게 크며, Groovy 테스트 스크립트에서 ``println(data)``를 호출했을 때 바이너리 배열이 출력됬기 때문입니다.

> PostWrite.groovy

```java
@Before
public void before() {
  headers = [
    new NVPair("Authorization", "Bearer abc.def.xyz"),
    new NVPair("Content-Type", "multipart/form-data; boundary=-----1234")
  ]
  request.setHeaders(headers)
  CookieManager.addCookies(cookies)
  grinder.logger.info("before. init headers and cookies")
}
```

이번에는 [관련 글](https://stackoverflow.com/questions/54940472/no-multipart-boundary-was-found)을 참고하여, Content-Type을 정의할 때 boundary 값을 추가했습니다.

![image](https://user-images.githubusercontent.com/56240505/133450950-5fb1d1a3-2911-44d4-b126-54219375b02d.png)

검증을 해본 결과, 이번에는 Multipart Servlet Request를 Resolve했으나 정작 ``PostRequest``에 데이터가 전부 null로 바인딩되는 문제가 발생했습니다.

Request Body에 바이너리 데이터는 확실히 들어있으니, 아무래도 요청 본문의 Form-Data들이 제가 지정한 Boundary로 잘 구분되지 않아 생긴 에러로 추측했습니다.

![image](https://user-images.githubusercontent.com/56240505/133452279-22f83661-97c5-483e-b70a-b43b1979e1c6.png)

``Codecs.mpFormDataEncode`` 메서드 [Docs](http://grinder.sourceforge.net/g3/script-javadoc/HTTPClient/Codecs.html)를 확인해본 결과, 세 번째 파라미터에 대한 설명이 제대로 나와있습니다.

세 번째 파라미터에 배열을 대입하면, 배열의 0번째 원소를 ``Key: "Content-Type", Value: "multipart/form-data; boundary={random_string}"``으로 변환합니다.

> PostWrite.groovy

```java
grinder.logger.debug("Before headers[0]: ${headers[0]}, headers[1]: ${headers[1]}")
def data = Codecs.mpFormDataEncode(params, files, headers)
grinder.logger.debug("After headers[0]: ${headers[0]}, headers[1]: ${headers[1]}")
```

> Log

```
2021-09-15 23:29:36,925 DEBUG Before headers[0]: HTTPClient.NVPair[name=Authorization,value=Bearer abc.def.xyz], headers[1]: HTTPClient.NVPair[name=Content-Type,value=multipart/form-data; boundary=-----1234]
2021-09-15 23:29:36,960 DEBUG After headers[0]: HTTPClient.NVPair[name=Content-Type,value=multipart/form-data; boundary=--------ieoau._._+2_8_GoodLuck8.3-ds0d0J0S0Kl234324jfLdsjfdAuaoei-----], headers[1]: HTTPClient.NVPair[name=Content-Type,value=multipart/form-data; boundary=-----1234]
```

해당 메서드 호출 전후로 로그를 찍어본 결과, headers 배열의 첫 번째 원소가 Authorization에서 Content-Type으로 변경된 것을 확인할 수 있었습니다.

![image](https://user-images.githubusercontent.com/56240505/133450950-5fb1d1a3-2911-44d4-b126-54219375b02d.png)

그런데 검증을 다시 해봐도, Spring Controller에는 변경된 Headers가 아닌 예전 Headers가 담긴 요청이 날아가고 있었습니다. 😭

### 4.2. 삽질의 끝

> PostWrite.groovy

```java
grinder.logger.debug("Before headers[0]: ${headers[0]}, headers[1]: ${headers[1]}")
def data = Codecs.mpFormDataEncode(params, files, headers)
request.setHeaders(headers);
grinder.logger.debug("After headers[0]: ${headers[0]}, headers[1]: ${headers[1]}")
HTTPResponse response = request.POST("http://localhost:8081/api/abc", data);
```

``Codecs.mpFormDataEncode`` 메서드 수행 이후 ``request.setHeaders``를 한 번더 호출해주니 문제가 해결되었습니다. @Before 메서드에서 ``request.setHeaders()``를 호출하기 때문에 당연히 ``request`` 객체는 ``headers``를 참조하기 있을 것이라 예단했었는데요. 참조 변수 ``headers`` 내부 값이 변경되면 ``request`` 또한 변경된 ``headers``를 바라볼 것이라고 생각했기 때문입니다.

> HttpRequest.java

```java
public void setHeaders(NVPair[] nvPairHeaders) {
  setHeaders(convert(nvPairHeaders, BasicHeader::new));
}
```

> PairListConvertUtils.java

```java
public static <R> List<R> convert(NVPair[] pairs, BiFunction<String, String, R> converter) {
  Function<NVPair, R> pairMapper = pair -> converter.apply(pair.getName(), pair.getValue());
  return Arrays.stream(pairs)
    .map(pairMapper)
    .collect(toList());
}
```

실제 메서드 코드를 까보니 ``setHeaders()`` 메서드는 내부적으로 주어진 Map 혹은 NVPair 배열의 값을 방어적으로 복사해서 사용하고 있었습니다. 😣

### 4.3. 스크립트 최종본

> PostWrite.groovy

```java
import static net.grinder.script.Grinder.grinder
import static org.junit.Assert.*
import static org.hamcrest.Matchers.*
import net.grinder.script.GTest
import net.grinder.script.Grinder
import net.grinder.scriptengine.groovy.junit.GrinderRunner
import net.grinder.scriptengine.groovy.junit.annotation.BeforeProcess
import net.grinder.scriptengine.groovy.junit.annotation.BeforeThread
// import static net.grinder.util.GrinderUtils.* // You can use this if you're using nGrinder after 3.2.3
import org.junit.Before
import org.junit.BeforeClass
import org.junit.Test
import org.junit.runner.RunWith

import org.ngrinder.http.HTTPRequest
import org.ngrinder.http.HTTPRequestControl
import org.ngrinder.http.HTTPResponse
import HTTPClient.Codecs
import HTTPClient.NVPair
import org.ngrinder.http.cookie.Cookie
import org.ngrinder.http.cookie.CookieManager

/**
* A simple example using the HTTP plugin that shows the retrieval of a single page via HTTP.
*
* This script is automatically generated by ngrinder.
*
* @author admin
*/
@RunWith(GrinderRunner)
class test {

	public static GTest test
	public static HTTPRequest request
	public static NVPair[] headers = []
	public static List<Cookie> cookies = []

	@BeforeProcess
	public static void beforeProcess() {
		HTTPRequestControl.setConnectionTimeout(300000)
		test = new GTest(1, "32.232.168.41")
		request = new HTTPRequest()
		grinder.logger.info("before process.")
	}

	@BeforeThread
	public void beforeThread() {
		test.record(this, "test")
		grinder.statistics.delayReports = true
		grinder.logger.info("before thread.")
	}

	@Before
	public void before() {
		headers = [new NVPair("", ""), new NVPair("Authorization", "Bearer abc.def.xyz")]
		request.setHeaders(headers)
		CookieManager.addCookies(cookies)
		grinder.logger.info("before. init headers and cookies")
	}

	@Test
	public void test() {		
		NVPair param1 = new NVPair("githubRepoUrl", "https://github.com/test");
		NVPair param2 = new NVPair("content", "content");
		NVPair param3 = new NVPair("tags", "typescript");
		NVPair param4 = new NVPair("tags", "html");

		NVPair[] params = [param1, param2, param3, param4];
		NVPair[] files = [new NVPair("images", "./resources/test.png"), new NVPair("images", "./resources/test.png")];

		def data = Codecs.mpFormDataEncode(params, files, headers)
		request.setHeaders(headers);
		HTTPResponse response = request.POST("http://32.232.168.41:8080/api/abc", data);

		if (response.statusCode == 301 || response.statusCode == 302) {
			grinder.logger.warn("Warning. The response may not be correct. The response code was {}.", response.statusCode)
		} else {
			assertThat(response.statusCode, is(200))
		}
	}
}
```

아무튼 인코딩 메서드를 호출하면 주어진 Headers 배열의 0번 인덱스의 값이 변경된다는 것을 알았습니다. Authorization 헤더를 Headers 배열의 0번 인덱스에 지정해두면 인코딩 메서드 호출 이후 해당 헤더가 누락될 수 있으니, 1번 인덱스에 저장해두고 0번 인덱스는 빈 값을 넣어줍니다. 이후 검증해보면 MultipartFile이 Spring Controller의 DTO에 잘 바인딩되는 것을 확인할 수 있습니다.

<br>

---

## Reference

* [nGrinder - WiKi](https://github.com/naver/ngrinder/wiki)
* [20190322 nGrinder file upload(multipart/form-data) 기능 확인](https://gist.github.com/ihoneymon/a83b22a42795b349c389a883a7bbf356)
