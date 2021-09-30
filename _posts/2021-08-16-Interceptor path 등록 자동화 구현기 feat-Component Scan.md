---
layout: post
title: Interceptor path 등록 자동화 구현기 feat. Component Scan
date: 2021-08-16 11:20:00 +0300
description: 인터셉터 path 자동화 등록 라이브러리를 구현하며 얻을 지식을 공유합니다.
img: frontend-performance.png
tags: [백엔드, interceptor, 자동화]
---

예제 소스: https://github.com/bperhaps/ComponentScan

안녕하세요. 깃-들다 팀의 손너잘입니다.

이번 글에서는 저희 팀의 인증, 인가 로직 리팩토링과 관련된 이야기를 해보고자 합니다.

현재 저희 팀은 API에 접근하는 유저의 인증 및 인가 로직을 구현하기 위해 Interceptor를 약간 커스텀 하여 사용하고 있습니다.

커스텀을 진행한 이유는 간단한데요, 동일한 URL에 대해서 각각의 HttpMethod마다 서로 다른 인증로직을 부여하기 위해서 입니다.

예를 들면 https://api.pick-git.com/api/post 와 같은 URL에 대하여 POST 의 경우는 사용자의 인증을 진행하고, 적절한 인가 과정을 거쳐 로직을 실행해야 하지만 GET 의 경우는 로그인 하지 않은 사용자의 경우에도 게시물을 불러올 수 있도록 해야합니다.

 

일반적으로 인증 로직은 interceptor에 위치하게 되는데, 스프링에서 제공하는 기본 interceptor register의 경우 url path를 이용하여 interceptor에 등록, 제외할 수는 있지만 동일한 URL에 대한 METHOD 분기는 지원하고 있지 않습니다.

깃-들다 팀은 이를 해결하기 위해 데코레이터 패턴을 활용한 멋진 방법을 사용했습니다. 하지만 이를 이용하다 보니 몇가지 불편함이 생겼습니다.

![img](https://blog.kakaocdn.net/dn/bfZt7w/btrb8g580TI/SOT82SpSbaZR6hlJBpqKzK/img.png)

위와 같이 매번 interceptor configuration을 일일이 적용시켜줘야 한다는 것이었습니다.

이를 깜빡하고 테스트코드를 작성하다가 테스트가 터지는 날에는 몇시간씩 디버거를 붙잡으면서 머릿속에 물음표를 띄우는 경우도 있었죠..

그러다 문득 인터셉터를 등록하는 과정을 어노테이션 기반으로 리팩토링 하는게 어떨까? 하는 생각이 들었습니다.

그렇게 모든 Interceptor 등록 로직을 어노테이션 기반으로 마이그레이션에 성공했고, 이번 글에서는 이에 대한 지식을 공유하고자 합니다.

 

제가 했던 기본적인 구상은 다음과 같습니다.

생성해야 하는 어노테이션

- @ForLoginAndGuestUser
- @ForGuest

위 어노테이션을 새로 생성하는 Controller Method에 적용하면, 자동으로 Custom Interceptor에 등록을 시켜주는 형태인 것이죠..

즉, 최종적 목표는 아래와 같은 로직을 완성하는 것 입니다.

```
@ForOnlyLoginUser
@PostMapping("/posts/{postId}/comments")
public ResponseEntity<CommentResponse> addComment()
    ...
}

@ForLoginAndGuestUser
@getMapping("/posts/{postId}/comments")
public ResponseEntity<CommentResponse> addComment()
    ...
}
```

기존 방식처럼 intercepter관련 로직을 신경쓰는 것 보다 훨씬 깔끔하고, 팀원들이 비즈니스 로직에 더 집중할 수 있겠죠?

### 기능 요구 사항

구현하고자 하는 기능의 요구사항은 다음과 같았습니다.

- 어노테이션이 붙은 Method로 부터 URI 파싱.

- 어노테이션이 붙은 Method로 부터 HttpMethod 파싱.

- 파싱한 데이터를 기반으로 특정한 기능 수행 (인증 로직 Interceptor에 파싱한 URI와 HttpMethod를 등록한다).

 

하지만, 이번 글에서는, 예제의 간소화를 위해 요구사항을 아래와 같이 변경하겠습니다.

- 어노테이션이 붙은 Method로 부터 URI 파싱.

- 어노테이션이 붙은 Method로 부터 HttpMethod 파싱.

- 파싱한 데이터를 기반으로 특정한 기능 수행 (특정 URI로 Get 요청 시 어노테이션이 붙은 Method들로 부터 파싱한 URI와 매칭되는 HttpMethod들을 반환한다).

## Annotation

가장 먼저 해야할 것은 Anntation을 생성 하는 것입니다.

```
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface ForLoginUser {
}

---------------------------------------

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface ForGuest {
}
```

간단하게 Method에만 적용 가능하고, RUNTIME에 동작하도록 작성했습니다

## 데이터 파싱 (URI and HttpMethod)

가장 먼저 어노테이션 기반으로 필요한 정보를 파싱해 보도록 하겠습니다.

파싱할 정보는 URI , HttpMethod 정도가 되겠네요. 현재 저희는 Spring boot Framwork를 사용하며 어노테이션 기반으로 코드를 작성하기 때문에 이를 기반으로 작성하겠습니다.

 

**Controller 클래스 추출 [**https://github.com/bperhaps/ComponentScan/tree/extract_controller]

가장 먼저 해야하는 것은, 수 많은 클래스들 중에 Controller 클래스만을 파싱하는 것 입니다. 먼저, 모든 컨트롤러들이 공통으로 가지고 있는것은 무엇일까요? 바로 @Controller, @RestController 입니다. 따라서 이 어노테이션을 가지고 있는 Class들을 파싱함으로써 Controller 클래스를 파싱할 수 있습니다.

해당 클래스에 특정 어노테이션이 있는지 확인하는 방법은 간단합니다. 리플렉션에서 제공하는 isAnnotationPresent() 메서드를 이용하여 간단히 해당 타입에 특정 어노테이션이 있는지 확인할 수 있습니다.

```
@Controller
public class ScanController1 {}

@RestController
public class ScanController1 {}

public class ScanController1 {}
```

테스트를 위해 위와같이 3개의 Controller 클래스를 생성하였습니다. 또한 아래와 같이 프로덕션 코드를 작성하여 Controller클래스를 분류해 보았습니다.



![img](https://blog.kakaocdn.net/dn/bRdkzx/btrci9RXZTZ/qiBrDVKxhqHrsw6cpf0bCk/img.png)



또한 아래와 같이 테스트 코드를 작성하여 제대로 동작하는지 확인하였습니다.



![img](https://blog.kakaocdn.net/dn/MQLR0/btrclI7Epb0/HQ4MMTYN9viNghr0nG7Ls1/img.png)



컨트롤러 어노테이션이 붙은 클래스인 ScanController1, ScanController2가 잘 추출됐는지 확인하는 테스트가 잘 돌아가는것을 확인할 수 있습니다.

 

**메소드 추출 [**https://github.com/bperhaps/ComponentScan/tree/extract_method]

컨트롤러를 추출했으니, 다음으로 할 것은 특정한 어노테이션이 붙은 메서드를 추출하는 것 입니다. 이번예제의 경우 @ForLoginUser , @ForLoginAndGuest 가 붙은 메서드를 추출하는게 목표가 되겠네요.

먼저 컨트롤러에 예시로 사용할 메서드를 생성하도록 하겠습니다. (자세한 소스는 github를 참조해 주세요)

```
@Controller
public class ScanController1 {

    @ForLoginAndGuest
    @GetMapping("/getTestGuest")
    public void getTest() {

    }

    @ForLoginUser
    @PostMapping("/postTstLogin")
    public void postTest() {

    }

    @PutMapping("/putTestNop")
    public void putTest() {

    }
}
```

메서드 추출 프로덕션은 다음과 같습니다



![img](https://blog.kakaocdn.net/dn/bxfBgS/btrclJyIAZR/Cp6nKICSHnhAS78TMxL0U0/img.png)



테스트 코드도 작성해 줍니다.



![img](https://blog.kakaocdn.net/dn/LTTgm/btrcebv1pf6/6kTs1Dr54No3RyzIDWCkqK/img.png)



소스를 한번 읽어보면, 각 컨트롤러들의 typetoken으로부터 메서드 타입들을 불러옵니다(getMethods(), getDeclaredMethod를 사용하지 않은 이유는, 어쩌피 스프링의 Mapping 어노테이션이 public method에서만 동작하기 때문에 굳이 private method까지 불러오는 수고를 할 필요가 없기 때문입니다). 그리고 각 메서드에 ForLoginAndguest, ForLoginUser 어노테이션이 있는지 확인하고 파싱합니다. 제가 예제로 만들어 놓은 클래스에서는 파싱되는 메서드의 갯수가 5개이기 때문에 전체 파싱되는 메서드의 개수가 5개인지 확인하는 assertion을 추가하였습니다(github 코드를 참조 하랍니다).

 

**HttpMethod, URI 파싱 [**https://github.com/bperhaps/ComponentScan/tree/URI_parse]

다음으로 진행해야 할 것은, mapping annotation들로부터 uri를 파싱하는 것 입니다. 저희가 만들어야 할 요구사항이기 때문이죠😉

그 전에, 저희 팀에서는 RequestMapping의 경우는 전역으로 공통되는 부분을 추려내기 위해 Controller쪽에서만 사용하고 있고, 메소드단위에서는 GetMapping, PostMapping.... 등을 사용하고 있습니다. 따라서 이러한 컨벤션을 지킨다는 가정하에 나머지 구현을 진행하도록 하겠습니다.

먼저 Spring에서 URI은 RequestMapping에 정의된 URI와 Method단위에 매핑된 Mapping 어노테이션이 합쳐져서 만들어집니다.

```
@RequestMapping("/controller2")
@RestController
public class ScanController2 {

    @ForLoginUser
    @GetMapping("/getTestLogin")
    public void getTest() {

    }
}
```

 

따라서 위와같이 있다면, /controller/getTestLogin 과 같이 URI가 생성되어야 하죠.

기본적으로 위와 같은 Mapping 어노테이션들에 적힌 URI는 values()라는 메서드에 정의되어 있습니다.



![img](https://blog.kakaocdn.net/dn/bWJt9k/btrcebiuDUS/TFnCgofxK23OBfmhADkla0/img.png)



어노테이션으로 부터 값을 불러오는 방식은 간단합니다. 해당 메서드(value())를 호출하기만 하면 되는데요, 위에서 메서드와 컨트롤러 파싱까지 성공했으니 파싱한 컨트롤러와 메서드에서 Mapping 어노테이션을 추출하여 value를 추출해 보도록 합시다. 먼저, 각 메서드가 어떤 Mapping Annotation을 가지고 있는지 확인해야하고, 해당 어노테이션으로 부터 value를 추출해야 합니다.



![img](https://blog.kakaocdn.net/dn/xCirM/btrcea4W1IL/uJi8JEDKJUkzPiKtCkeWTk/img.png)



일단 간단하게 enum을 이용해서 각 HttpMethod들을 컨트롤 할 수 있도록 하였습니다.

typeToken은 method에 정의된 annotation과의 타입비교를 위해 있고, httpMethod는 추후에 해당 method가 어떤 HttpMethod에 매핑되어 있나 반환해주기 위해 , 마지막 values는 어노테이션으로 부터 value를 추출하기 위해 존재합니다. 지금은 이해가 잘 안가실 수도 있으니 프로덕션을 통해서 한번 살펴봅시다.



![img](https://blog.kakaocdn.net/dn/nBxxB/btrclIfvf0f/H62p2fIY1pcbTaMCEPavtk/img.png)



먼저, method에 있는 Mapping어노테이션을 파싱하여 value로 정의된 URI를 파싱하는 로직입니다.

간단히 method를 인자로 받고, value를 추출하는 로직으로 작성했습니다. 또한 매칭되는것이 없다면 빈 문자열이 들어있는 리스트를 반환하도록 했습니다. 이때 Function<> values가 왜 존재하는지 궁금해 하실수도 있는데, 어노테이션은 기본적으로 상속이 불가하기 때문에 다형성을 이용하지 못합니다. 따라서 모든 Mapping annotation이 동일하게 value() 메서드를 가지고 있더라도 Class<? extends Annotation> 타입을 통해 value() 메서드를 호출시키는 게 불가합니다. 따라서 위와같이 약간은 정적으로 코드르 작성해 주셔야 합니다(혹시 리플랙션으로 value를 추출시킬 수 있는지 시도해 보았지만 Annotation이 인스턴스가 객체가 아니기 때문에 invoke가 불가능 한것 같더군요).

이젠, 마찬가지로 Controller쪽에 RequestMapping이 있다면 값을 추출하는 로직도 작성하도록 하겠습니다.



![img](https://blog.kakaocdn.net/dn/c8PWiQ/btrcb8ffl70/q6hNMXPezipXtrhAGggAdK/img.png)



간단하군요! 모든 준비과정은 끝났습니다. 지금까지 만든 객체들을 조합하여 URI를 파싱하는 클래스를 작성하도록 하겠습니다.



![img](https://blog.kakaocdn.net/dn/pKX9Y/btrcb9ZxVLs/4vre8KQYnTo6QuvtcqMKS0/img.png)



소스가 조금 길어서 메인 로직만 추출해 왔습니다. 자세한 소스는 깃허브를 참조해 주세요!

컨트롤러로 부터 URI를 추출하고, 메소드로부터 URI를 추출하고, 둘을 조합하여 최종 URI를 만들어내는 로직입니다.

테스트를 한번 작성해 봅시다.



![img](https://blog.kakaocdn.net/dn/bWo52V/btrb9RSovjc/Kn3FQGBM4KY0zUagRTr67k/img.png)



멋지게 통과하는군요! 이로서 URI 파싱로직이 완성되었습니다.

 

**httpMethod 파싱 [**https://github.com/bperhaps/ComponentScan/tree/extract_httpMethod**]**

요구사항의 마지막, HttpMethod를 파싱해 보도록 하겠습니다. 모든 준비물은 완성되어있으니 또 다시 조합하면 끝입니다.

우리는 위에서 URI를 파싱로직을 완성시켰는데요, 해당 URI에 매핑된 HttpMethod 또한 가지고 오기 위해서는 두 데이터를 담을 자료구조가 필요합니다.



![img](https://blog.kakaocdn.net/dn/bd4Y72/btrcb7UZmRy/k0KpNB4IhBBEst4U396Ksk/img.png)



간단하게 자바 bean규약에 의거하여 생성해 주었습니다.

위에서 우리가 HttpMethods enum을 생성할 때 필드로 HttpMethod를 넣어주었습니다. 바로 지금 이 기능을 위해 생성해둔 것 인데요, Method를 인자로 받고 매핑된 HttpMethod를 찾아 반환하는 로직을 구현해 보겠습니다.



![img](https://blog.kakaocdn.net/dn/c95ufY/btrci86BAkG/SCJsSGKeL5OBBC6zMAd1dk/img.png)



enum 내부에 구현하였습니다. 간단하죠? 이젠 위에서 만들었던 URIScanner가 기존처럼 URI만을 반환하는것이 아닌, Method까지 함께 반환하도록 로직을 변경해 보겠습니다.

 



![img](https://blog.kakaocdn.net/dn/luAKn/btrb7bdhle5/Mv0PpKxgM4ELsFa4KDeZy0/img.png)



이 부분 또한 메인로직만 들고왔습니다 기존 URI String만 반환했던것과는 다르게 위에서 만든 data structure를 반환하는것을 볼 수 있습니다.



![img](https://blog.kakaocdn.net/dn/mvIcM/btrb7zEUr6I/qjFHbIiKLEi7OsP6EPoS21/img.png)



테스트 코드또한 위와같이 변경하였고 정상적으로 동작하는것을 볼 수 있습니다.

이로써 우리가 원하는 URI와 HttpMethod 파싱 로직을 완성하였습니다!! (와~~)

하지만, 이 기능만 가지고는 뭔가를 하기가 애매합니다. 우리가 원하는 것은 프로그램이 로딩되면, 자동으로 모든 class들을 파싱한 뒤 파싱한 데이터를 가지고 특정 행동을 하는것 이기 때문입니다.

### Component Scan

필요한 정보들은 Reflection을 사용하여 간단하게 추출 가능한데, 문제는 ClassLoader의 동작 특성상 모든 클래스들의 Type Token을 불러올 수 없습니다(기본적으로 필요한 클래스를 동적으로 불러오는 방식을 사용하니까요).

따라서 Class.forName() 을 이용하여 클래스의 canonical path를 통해 Type Token을 가지고 와야 하는데요. 이를 위해서 Spring의 Component Scan과 비슷한 역할을 하는 모듈을 구성해야 합니다.

 

**Reflections[**https://github.com/bperhaps/ComponentScan/tree/using_reflections**]**

가장 간단한 방법은 Reflections 라이브러리[https://github.com/ronmamo/reflections]를 사용하는 것 입니다. 이미 충분히 오랜 기간 검증받고 많은 곳에서 사용하는 라이브러리임만큼 마음놓고 사용할 수 있는데요. 한번 Reflections를 이용해서 구현해 보도록 하겠습니다.



![img](https://blog.kakaocdn.net/dn/dPxGcb/btrb7bRQB3H/8xvSwtTauY1shGfwtWPjo1/img.png)



먼저, Reflections 라이브러리에 대한 의존성을 만들어줍니다.



![img](https://blog.kakaocdn.net/dn/cckefI/btrb7y0jwXc/lnyTFFyKIrOP0tEzq8srmk/img.png)



그런 뒤 reflections를 이용하여 모든 Class들에 대한 typeToken을 받아올 수 있도록 합니다. basePackage는 파싱할 package의 최상위 경로를 넣어주면 됩니다.

테스트 코드를 작성해 보겠습니다.



![img](https://blog.kakaocdn.net/dn/uYUTl/btrb8hD3nll/pICSWBkGVWL1ydKnaTn7fK/img.png)



테스트에는 처음에 만들었던, controller들이 있는 패키지를 기준으로 하였습니다.

정상적으로 3개의 컨트롤러를 파싱하는것을 볼 수 있습니다.

그러면 이 기능을 이용하여 기능을 구현해 보도록 하겠습니다.

요구사항은 아래와 같이 구현할 것 입니다.

1. GET /getUriAndMethods 를 수행하는 컨트롤러 메서드를 생성한다.
2. ArugumentResolver를 통해서 스프링부트 부팅 시 초기화 한 어노테이션이 붙은 controller method들의 URI와 HttpMethod들을 반환받는다.
3. 이를 반환 시킨다.

먼저 ArgumentResolver를 생성하겠습니다.



![img](https://blog.kakaocdn.net/dn/dfXwMa/btrclJlaJZi/7DUwxen0lJwkgHvJfRvNS1/img.png)



생성자로 초기화 시점에 파싱한 데이터를 받아올 수 있도록 하고, 이를 반환하도록 구성하였습니다.

다음은 WebMvcConfiguration입니다.



![img](https://blog.kakaocdn.net/dn/cpKc2R/btrb7zLKldT/CJEp5dt0GbWghXhj6vGh71/img.png)



리졸버를 등록해 주었고, getUriAndMethods() 를 통해 원하는 데이터를 파싱하도록 하였습니다.

마지막으로 Controller 부분입니다.



![img](https://blog.kakaocdn.net/dn/bq920x/btrb8hxjhdV/yO15nYKHt9jAf1LuHohkQ1/img.png)



마지막으로 테스트를 통해 정상적으로 동작하는지 확인해 보도록 하겠습니다.



![img](https://blog.kakaocdn.net/dn/J61sh/btrb7zLKlm3/Lpmb8sWSFcq4JVV18vBshK/img.png)



정상적으로 동작하는것을 볼 수 있습니다.

이를통해 우리는 전체 클래스를 스캔하고, 어노테이션을 통해 필요한 데이터만 추출한 뒤 이를 사용하는 방법에 대하여 알아봤습니다. 위 예제를 응용하면 여러가지 행위들을 할 수 있겠죠?

하지만 이렇게 끝낸다면 재미가 없습니다. 이번에는 Reflections를 사용하기 않고 전체 클래스를 파싱하는 방법에 대해 알아보겠습니다.

 

**Class parsing without Reflections[**https://github.com/bperhaps/ComponentScan/tree/using_raw**]**

위에서 우리는 클래스로더의 동작방식때문에 모든 클래스들을 typeToken을 런타임에서 가지고 오는것이 불가능 하다고 했습니다(이때 런타임에서 불가능하다는 것은, getAllClasses같은 메서드를 통해 메모리상에 있는 모든 클래스들을 가지고 오는것을 의미합니다). 따라서 Class.forName() 을 이용해야 한다고 했는데요. 전체 클래스들의 canonical path를 어떻게 가지고 올 수 있을까요?

방법은, 파일시스템을 이용하면 됩니다. 자바 프로젝트는 package경로와 파일시스템의 폴더구조가 동일합니다. 따라서 이를 이용하여 파일시스템을 탐색하면, 각 클래스들의 package canonical path를 구할 수 있습니다.

가장 먼저 해야할 일은 ClassL1oader가 참조하는 class파일들의 위치를 찾는 일입니다. 이 부분에서 저는 참 많은 애를 먹었는데요. 운영체제, 빌드 방식마다 소스들이 다르게 동작했기 때문입니다. 이번 글에서는 그 중에서 제가 성공한 방식을 설명드리겠습니다.

 

가장 먼저 알아내야 할 정보는 classPath입니다. ClassLoader가 참조하는 classPath를 알아내면, 해당 위치의 파일시스템을 통해 전체 class를 순회할 수 있습니다.



![img](https://blog.kakaocdn.net/dn/Y4sEf/btrb8hKOc2Q/LJVy20iNOKVWXR00Gt2P91/img.png)



방법은 위와 같습니다. 현재 동작중인 클래스의 class파일이 위치하는 곳을 찾아서 classPath를 유추할 수 있습니다. (이 외에도 ClassLoader.getResource() 등의 방법을 통해 얻을 수 있지만, 위와같이 유추하는 이유는 뒤에서 설명하겠습니다).

위와같이 소스를 작성하면 아래와 같은 출력을 볼 수 있습니다.



![img](https://blog.kakaocdn.net/dn/bI6q32/btrci8rYWtY/R3z9CGoSBQaf4TO89Ws9ak/img.png)



build/classes/java/main 으로 classPath를 찾은것을 확인할 수 있습니다. 위 코드가 프로덕션에 위치하기때문에 이러한 경로가 도출됩니다.

하지만 스캔의 범위를 test폴더까지 늘려야 할 필요가 있을수도 있습니다. 이를 위해 저는 아래와 같은 로직을 추가적으로 넣어줬습니다.



![img](https://blog.kakaocdn.net/dn/sosHQ/btrb8hjHMo4/1oPPGKYldabqx3SOcLw6Sk/img.png)



파싱한 경로의 마지막 부분을 하나 제거하여 하나 상위 폴더를 basePath로 만들어냈습니다. classes폴더 주고 상 java 폴더 내부에 main과 test(windows의 경우 production과 test)가 위치함으로 이렇게 하여 test와 main까지 탐색 범위를 늘릴 수 있습니다.



![img](https://blog.kakaocdn.net/dn/dLAGjh/btrb7zdRbVv/srTpbnMpiY0dIZO7KJRF9K/img.png)



메인 로직입니다. 마지막에는 추출한 경로를 package 형식에 맞게 만들어 주는 로직입니다. 천천히 읽어보면 이해가 될겁니다. 추출한 basePath부터 시작하여 파일시스템 탐색을 진행합니다. 파일 순회에는 nio에서 제공하는 walkFileTree를 이용하였습니다.



![img](https://blog.kakaocdn.net/dn/nTi3O/btrb9P78lyU/ZnKaTl73MgXg4PJpys8haK/img.png)



위 소스는 walkFileTree를 이용할때 필요한 FileVisitor의 구현체 소스입니다.

파일에 접근하였을 때, 파일이 class파일인지, 또한 제가 정의한 baseRoot를 포함 하는지 확인한 후 모두 부합하다면 내부 컬렉션에 전체 경로를 넣어줍니다. 그리고, getAllCanonicalPaths() 메서드에서 이 컬렉션을 반환받게 됩니다.

이제 테스트를 한번 돌려보겠습니다.



![img](https://blog.kakaocdn.net/dn/c3bYTl/btrb8fMY1dZ/HA92qUclAoUjZxnwhSvQVk/img.png)



테스트를 위에 테스트 폴더 내부에 위와같이 패키지를 하나 만들었습니다.



![img](https://blog.kakaocdn.net/dn/JP1N3/btrcb9FdLIz/KA7C7lvLLxxqi1WfAQFZfk/img.png)



테스트코드는 위와 같이 작성하였고, 정상적으로 동작하는것을 볼 수 있습니다.

마지막으로 이전에 Reflections를 이용하여 작성한 로직을 지금까지 만든 객체로 교체해보겠습니다.



![img](https://blog.kakaocdn.net/dn/c1haWb/btrb7z50Jiz/sKTYTNDPxjbDSRWiVD2Fp1/img.png)



ClassesScanner 로직을 위와같이 변경하였고, 이와 함께 변하는 다른 부분도 모두 변경시켰습니다. 이제 한번 이전에 작성한 인수테스트를 돌려보겠습니다.



![img](https://blog.kakaocdn.net/dn/X3Thb/btrcebv1ph4/mk3FdC1uUd1udVILuCGyQK/img.png)



정상적으로 동작하는걸 볼 수 있습니다.

 

**성공?**

자, 여기까지 왔으면 과연 성공일까요? 아닙니다. 막상 프로젝트를 빌드하여 jar파일로 추출한 뒤 동작시키면 제대로 동작하지 않는것을 알 수 있습니다.



![img](https://blog.kakaocdn.net/dn/cQoYDH/btrcb9yrlSZ/rIpkJ2XahbI0dZdOKuRBrK/img.png)



모든 테스트를 통과하고 빌드도 성공하지만



![img](https://blog.kakaocdn.net/dn/JlcfA/btrcb9LZysI/WYiQ1lK3w6c1JdwNIu12k1/img.png)



jar 파일을 실행시키면 터집니다!

 

왜 이런일이 발생할까요? 바로, 우리가 canonical path를 추출하는데 있어 파일시스템을 사용했기 때문입니다. 우리가 ide를 이용하여 빌드를 진행했다면, 당연히 build/classes에 클래스파일이 있습니다. 하지만 jar파일을 어떤가요? jar라는 압축파일 내부에 class파일이 존재합니다. 따라서 파일 순회가 불가능한 문제점이 발생합니다.

이 문제를 해결해 봅시다.

위에서 제가 classPath를 찾기위해서 source파일의 위치를 확인하는 방식을 사용했었습니다. 바로 지금 이 문제를 해결하기 위함이었습니다.

먼저, 실행중인 jar파일의 위치를 찾는게 중요합니다. 인터넷에 있는 수 많은 방법을 시도했지만 운영체제 등 환경의 영향을 많이 받았습니다. 그 중, 모든 환경에서 동작하던 방식이 위 방식이라 채택했습니다.

방법은 다음과 같습니다. jar파일의 위치를 찾았다면, 임시 폴더에 jar폴더를 풉니다. 그리고 그 폴더의 경로에서 class들을 파싱하면 위 로직을 동일하게 사용할 수 있습니다.

일단, jar파일에서 실행했을 때 경로가 어떻게 보이는지 확인해봅시다.



![img](https://blog.kakaocdn.net/dn/v6tFE/btrcgPe9dui/bqXXeypyafYZiveJpHNRP1/img.png)



이전 소스와 달라진 점이 있다면, 마지막이 getPath()가 아닌 toString()으로 변경되었습니다. 운영체제마다 다른데, getPath()를 사용하면 path가 null로 나옵니다.



![img](https://blog.kakaocdn.net/dn/l9wDe/btrcebv1piS/dUjBURGqO0WuipKk7XEvt0/img.png)



보이시는것 처럼 경로가 위와같이 도출됩니다. 위 형식은 모든 운영체제에서 동일합니다. 따라서 이에 맞게 경로를 파싱해주면 됩니다.



![img](https://blog.kakaocdn.net/dn/daISTS/btrcjai5vdH/p4S7E1ZKWQgYqzoHRB7ar0/img.png)



파싱 로직은 위와 같습니다. jar파일인 경우 경로의 시작이 jar로 시작하기 때문에 위와같이 분기를 주어 처리하였습니다. 변경 후에도 모든 테스트가 정상적으로 통과 하는군요. 이제 jar파일일 때 압축을 해제시키고 이를 basePath로 사용하는 로직을 작성하겠습니다.



![img](https://blog.kakaocdn.net/dn/kO81d/btrb7cb7k7f/Wk4XYiv9YkcTcHZ38r5kXk/img.png)



CanonicalPathScanner 의 로직이 이렇게 변경되었습니다! 간단하지요? 임시폴더에 zip 해제하는 소스는 인터넷에 많으니 설명 생략하도록 하겠습니다. 이제 한번 잘 돌아가는지 jar빌드 후 확인해 보겠습니다.



![img](https://blog.kakaocdn.net/dn/bhIqxt/btrcebQkSiR/XVbElbZTUUPxraYKDfcJ8k/img.png)



정상적으로 동작합니다!!

 

이로서 모든 구현이 완료되었습니다.

깃들다 팀에서는 위와같이 모든 구현을 진행하고, interceptor 자동등록을 구현했습니다. 여러분도 이 지식을 활용하여 여러 기능들을 만들어보길 바랍니다 감사합니다🙂
