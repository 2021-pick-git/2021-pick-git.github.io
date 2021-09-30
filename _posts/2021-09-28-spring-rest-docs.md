---
layout: post
title: Spring REST Docs 적용 (Gradle 7)
date: 2021-09-28 13:32:20 +0300
description: API를 문서화해봅시다.
img: restdocs.png
tags: [백엔드, Spring]
---

## 1. Spring REST Docs

안녕하세요, 케빈입니다. Pick-Git 서비스 개발 초기에는 프론트엔드 팀원들과 API 스펙 협의 후 Notion에 작성해서 공유했었는데요. 해당 방법에는 여러 문제점이 존재했습니다. 수기로 작성하다 보니 인적 실수로 인해 오기재 하는 경우가 많았습니다. 또한 개발 과정에서 API 스펙 변경이 매우 빈번하게 발생하는데, 이를 미기재하는 실수로 인해 일정에 차질이 종종 생기곤 했습니다.

팀원 모두 조금 더 규격화된 API 문서화의 도입의 필요성을 느꼈습니다. 우리 팀이 선택한 도구는 Spring Rest Docs입니다. Spring REST Docs는 RESTful 서비스의 문서화를 도와주는 도구인데요. 문서 작성 도구로 기본적으로 Asciidoctor를 사용하고, 이를 통해 최종적으로 HTML을 생성합니다. 심지어 Markdown을 사용하도록 설정할 수 있습니다.

전반적인 흐름은 다음과 같습니다.

* Spring MVC Test, Spring WebFlux WebTestClient, RestAssured 등으로 테스트 코드를 작성한다.
* 해당 테스트 코드를 통해 Snippet이 자동 생성된다.
* 사용자가 미리 작성해둔 템플릿 문서와 Snippet을 결합해 최종적인 API 문서를 만들어낸다.

### 1.1. Swagger 대비 장점

비슷한 대안으로 Swagger가 있습니다. 이 또한 쉽게 API를 문서화할 수 있는데요. 특히 Swagger는 API 동작을 테스트하는데 특화되었습니다. 그러나 Swagger는 프로덕션 코드에 애너테이션을 부착하는 등 코드 침투라는 단점이 존재합니다.

반면 Spring REST Docs는 프로덕션 코드에 별다른 영향을 주지 않습니다. 아울러 테스트를 기반으로 생성되며, Snippet이 올바르지 않을 경우 테스트가 실패합니다. 테스트를 강제하며, 테스트가 검증되면 작성되는 문서 또한 신뢰할 수 있다는 장점이 존재합니다.

<br>

## 2. 설정

> log

```bash
Some problems were found with the configuration of task ':asciidoctor' (type 'AsciidoctorTask').
  - In plugin 'org.asciidoctor.convert' type 'org.asciidoctor.gradle.AsciidoctorTask' method 'asGemPath()' should not be annotated with: @Optional, @InputDirectory.
```

설정을 위해 공식 문서에 기술된 방법을 참고했으나 위 에러에 계속 직면했습니다.

> build.gradle

```groovy
plugins {
    id 'org.asciidoctor.jvm.convert' version '3.3.2'
}
```

Gradle 7 부터는 ``org.asciidoctor.convert``가 아닌 ``asciidoctor.jvm.convert``를 사용한다고 합니다.

> build.gradle

```groovy
dependencies {
    testImplementation 'org.springframework.restdocs:spring-restdocs-mockmvc'
}
```

REST Docs를 RestAssured 혹은 WebTestClient로 작성하고 싶다면 ``spring-restdocs-webtestclient`` 혹은 ``spring-restdocs-restassured``로 교체하면 됩니다.

> build.gradle

```groovy
ext {
    snippetsDir = file('build/generated-snippets')
}

test {
    outputs.dir snippetsDir
    useJUnitPlatform()
}

asciidoctor {
    inputs.dir snippetsDir
    dependsOn test
}
```

Gradle 문법에 익숙하지 않지만 알아두면 좋은 개념들을 정리했습니다.

* ``dependsOn`` : 특정 Task가 의존하는 Task를 명시합니다.
  * 즉, ``asciidoctor`` Task는 의존하는 ``test`` Task 이후에 수행됩니다.
* ``finalizedBy`` : 특정 Task의 후행 Task를 명시합니다.
  * ``asciidoctor`` Task에 ``finalizedBy copy``라고 명시되어 있으면, ``asciidoctor`` Task 이후 copy Task가 실행됩니다.

> build.gradle

```groovy
ext {
    snippetsDir = file('build/generated-snippets')
}
```

생성된 스니펫의 저장 위치를 명시합니다.

> build.gradle

```groovy
test {
    outputs.dir snippetsDir
    useJUnitPlatform()
}
```

테스트 Task의 아웃풋 디렉토리를 스니펫 저장 위치로 설정합니다.

> build.gradle

```groovy
asciidoctor {
    inputs.dir snippetsDir
    dependsOn test
}
```

스니펫 저장 위치를 인풋으로 지정합니다. ``asciidoctor`` Task는 테스트 Task 수행 이후에 수행됩니다. 따라서 문서가 생성되기 전에 테스트가 수행되도록 보장합니다. 물론 꼭 의존성을 위와 같이 설정하지 않고 Jenkins Pipeline에서 Task를 적당히 순서에 맞춰 수행해도 됩니다.

<br>

## 3. 테스트 코드 작성

> PostControllerTest.java

```java
// when, then
ResultActions resultActions = mockMvc.perform(get("/api/posts/{id}", "1")
    .accept(MediaType.APPLICATION_JSON_VALUE))
    .andExpect(status().isOk())
    .andExpect(content().string(expected));

verify(postService, times(1)).readById(1L);

// restDocs
resultActions.andDo(document("post-read-one",
    preprocessRequest(prettyPrint()),
    preprocessRequest(prettyPrint()),
    responseFields(fieldWithPath("id").description("게시물 id"),
        fieldWithPath("title").description("제목"),
        fieldWithPath("content").description("내용"),
        fieldWithPath("author").description("글쓴이"),
        fieldWithPath("viewCounts").description("조회수"),
        fieldWithPath("createdDate").description("작성일"),
        fieldWithPath("modifiedDate").description("수정일"),
        fieldWithPath("imageUrls").description("사진 링크")))
);
```

> MockMvcRestDocumentation

```java
public static RestDocumentationResultHandler document(String identifier,
			OperationRequestPreprocessor requestPreprocessor, OperationResponsePreprocessor responsePreprocessor,
			Snippet... snippets) {
         //...
      }
```

``andDo()``메서드에 ``document()``를 추가하여 스니펫을 생성합니다. 가장 첫 번째 인자는 스니펫의 Identifier입니다.

``preprocessRequest(prettyPrint())``를 넣어주면 요청과 응답의 내용을 형식화함으로써 가독성이 높아집니다. ``document()`` 메서드는 오버로딩되어 있어서 ``prettyPrint()``를 요청, 응답, 요청 및 응답 모두 등 총 3가지에 적용할 수 있습니다.

Snippet의 경우 요청 헤더, 응답 헤더, 응답 본문 데이터 필드 등 다양하게 작성할 수 있는데요. 사용 예제 및 공식 문서를 보면서 학습하면 이해가 빠를 것입니다.

> PostControllerTest.java

```java
// restDocs
resultActions.andDo(document("post-read-multiple",
    getDocumentRequest(),
    getDocumentResponse(),
    responseFields(fieldWithPath("simplePostResponses[].id").description("게시물 id"),
        fieldWithPath("simplePostResponses[].title").description("제목"),
        fieldWithPath("simplePostResponses[].content").description("내용"),
        fieldWithPath("simplePostResponses[].author").description("글쓴이"),
        fieldWithPath("simplePostResponses[].viewCounts").description("조회수"),
        fieldWithPath("simplePostResponses[].createdDate").description("작성일"),
        fieldWithPath("simplePostResponses[].modifiedDate").description("수정일"),
        fieldWithPath("startPage").description("시작 페이지"),
        fieldWithPath("endPage").description("끝 페이지"),
        fieldWithPath("prev").description("이전 페이지 여부"),
        fieldWithPath("next").description("다음 페이지 여부")))
);
```

List가 포함된 응답은 ``[]``를 통해 처리합니다. 만약 응답이 단일 리스트만을 포함하고 있다면 예제와 다르게 ``[].id``, ``[].title``과 같이 작성할 수 있습니다.

> PostControllerTest.java

```java
resultActions.andDo(document("post-write-login",
    getDocumentRequest(),
    getDocumentResponse(),
    requestHeaders(headerWithName(HttpHeaders.AUTHORIZATION).description("Bearer token")),
    requestPartBody("images"),
    responseHeaders(headerWithName(HttpHeaders.LOCATION).description("게시물 주소")))
);
```

헤더 또한 검증이 가능합니다.

<img width="337" alt="스크린샷 2021-09-01 오전 2 48 32" src="https://user-images.githubusercontent.com/56240505/131551597-ee2fdecf-5633-4420-8214-6dfc2a0bb034.png">

Gradle 옵션에서 Test를 실행하면 다음과 같이 스니펫이 생성됩니다. 테스트 코드를 통해 생성된 스니펫과 **사용자가 미리 작성해둔 템플릿 문서**를 결합해 최종 API HTML 문서를 생성합니다.

사용자가 미리 작성해둘 파일은 Gradle 기준 ``src/docs/asciidoc``에 위치시킵니다. 이 때 파일 확장자는 ``.adoc``입니다.

> api.adoc

```adoc
ifndef::snippets[]
:snippets: ./build/generated-snippets
endif::[]

== User
=== 회원탈퇴 (비로그인)
==== Request
include::{snippets}/withdraw-not-login/http-request.adoc[]
==== Response
include::{snippets}/withdraw-not-login/http-response.adoc[]

=== 회원탈퇴 (로그인)
=== 게시물 단건 조회
==== Request
include::{snippets}/post-read-one/http-request.adoc[]
==== Response
include::{snippets}/post-read-one/http-response.adoc[]
==== Fields
include::{snippets}/post-read-one/response-fields.adoc[]
```

정적 블로그를 꾸미듯이 API 문서 또한 조금만 더 노력을 들이면 매우 아름답게 구성할 수 있습니다. 또한 각각의 스니펫 identifier 폴더 하위에는 첨부할 수 있는 다양한 adoc들이 존재하는데, 상황에 맞게 잘 선택하시면 됩니다.

> build.gradle

```groovy
bootJar {
    dependsOn asciidoctor
    copy {
        from "${asciidoctor.outputDir}"
        into 'BOOT-INF/classes/static/docs'
    }
}
```

해당 설정을 추가해주고, 이번에는 Gradle 옵션에서 bootJar를 실행해봅시다.

<img width="324" alt="스크린샷 2021-09-01 오전 2 59 33" src="https://user-images.githubusercontent.com/56240505/131552947-69d0a451-1da1-41fb-a690-30e16a6cbc42.png">

bootJar를 실행하면 ``src/docs/asciidoc``에 위치시킨 사용자가 정의한 adoc 파일과 테스트 코드에서 생성된 스니펫들이 조합된 최종 API 문서가 ``build/docds/asciidoc`` 디렉토리에 생성됩니다.

> build.gradle

```groovy
bootJar {
    dependsOn asciidoctor
    copy {
        from "${asciidoctor.outputDir}"
        into 'BOOT-INF/classes/static/docs'
    }
    finalizedBy 'copyDocument'
}

task copyDocument(type: Copy) {
    dependsOn bootJar
    from file("build/docs/asciidoc")
    into file("src/main/resources/static/docs")
}
```

bootJar Task를 통해 ``api.html``이 생성되면, 후행 Task로 copyDocument가 실행되도록 합시다. copyDocument가 수행되면 classpath(resources)로 API 문서가 복사됩니다.

![image](https://user-images.githubusercontent.com/56240505/131554209-8727e655-12b7-4e7e-b811-ab9507fd0416.png)

![image](https://user-images.githubusercontent.com/56240505/131554295-6907e473-0694-47ef-9572-e65eecae3a17.png)

bootJar로 실행하면 컴파일을 거치고 테스트 Task가 시작됩니다. 테스트 Task가 종료되면 의존 관계에 따라 asciidoctor, bootJar, copyDocument Task가 각각 실행됩니다.

![image](https://user-images.githubusercontent.com/56240505/131554549-9c6978bf-f20b-4e84-967f-72d43469deb9.png)

어플리케이션을 실행하면 잘 접속되는 것을 확인할 수 있습니다.

<br>

## 3. 기타

> build.gradle

```groovy
asciidoctor.doFirst {
    delete file('src/main/resources/static/docs')
}
```

만약 문서에 수정이 발생했음에도 반영되지 않는다면 다음 태스크를 추가하여 복사 이전에 기존에 있던 클래스패스의 API 문서를 제거하도록 합시다.

<br>

---

## Reference

* [Spring REST Docs](https://docs.spring.io/spring-restdocs/docs/current/reference/html5/)
* [[Spring] Spring rest docs 적용기(gradle 7.0.2)](https://velog.io/@max9106/Spring-Spring-rest-docs%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%9C-%EB%AC%B8%EC%84%9C%ED%99%94)
* [Configuring asciidoctor when using Spring Restdoc](https://stackoverflow.com/questions/68539790/configuring-asciidoctor-when-using-spring-restdoc)
