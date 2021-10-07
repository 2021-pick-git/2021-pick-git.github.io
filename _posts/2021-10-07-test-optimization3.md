---
layout: post
title: 테스트코드 최적화 여행기 (3)
date: 2021-10-07 16:20:00 +0900
description: Pickgit의 테스트코드 최적화 방법을 소개합니다.
img: testoptima.png
tags: [백엔드, test, 최적화]
---

우리는 처음에 Context를 재사용 하기 위해서 Dirties Context를 제거했습니다. 그리고 그를 통해 비약적인 테스트 속도 개선을 이뤄냈습니다. 그래서 저는 이제 모든 테스트가 동일한 Context를 돌려 사용한다고 생각했고, 안심하고 있었습니다. 그런데 전체 테스트를 진행했을때 이상한점을 찾았습니다. 테스트가 잘 진행되다가 중간중간에 한번씩 뚝,뚝, 걸리는 부분이 있는겁니다. 

그래서 그 뚝 뚝 걸리는 테스트를 확인해 봤더니 왠걸... 새로운 Context를 Load하고 있었습니다. 왜 이런일이 발생하는가 소스를 비교하고 실험해본 결과, 스프링이 테스트에서 컨텍스트를 관리하는 방법과 관련이 있었습니다.

스프링은 기본적으로 테스트를 진행할 때 Context를 캐싱하여 여러 테스트에서 돌려 사용할 수 있도록 합니다(토비의 스프링3.1 vol.1). DirtiesContext를 제거한것도 이것과 관련이 있죠. 이때, 캐싱의 조건(?)를 생각해봐야합니다. 스프링 테스트은 설정단위로 Application Context을 캐싱합니다. 예시를 통해서 확인해보겠습니다.

![image](https://user-images.githubusercontent.com/33603557/136337461-4adafeb8-fb80-452f-970e-05a6f1c4d394.png)

```java
@DataJpaTest
public class DataJpaTestTest {

    @Autowired
    PostRepository postRepository;

    @Test
    void name() {
    }
}
```

```java
@SpringBootTest
public class SprintBootTestTest {

    @Autowired
    PostRepository postRepository;

    @Test
    void name() {
    }
}
```

DataJpaTest를 4개, SpringBootTest를 4개 만들었습니다. 이를 실행시키면 총 몇개의 Application Context가 만들어질까요? 정답은 2개입니다. `@DataJpaTest` 는 DataJpaTest에 필요한 필수 Bean만 Context에 Load하고 SpringBootTest의 경우는 모든 Bean을 Load하게 됩니다. 즉, 둘의 Configuration 다르죠. 따라서 총 컨택스트는 2번 생성됩니다(동시에 2개가 생성되는게 아닙니다.)

![image](https://user-images.githubusercontent.com/33603557/136337489-ac32472c-39e3-4462-b220-8eaea3cfda9f.png)

![image](https://user-images.githubusercontent.com/33603557/136337501-595fe0cc-887e-4f33-a30b-3f008d405f59.png)

이 이야기를 왜 했는가 하니, 깃들다의 테스트코드들의 중간에 잠깐씩 멈추던게 바로 위 문제때문이었습니다. 어떤 Integration test는 DataJpaTest로 작성되어 있고, 또 다른 어떤 Integration test는 다른 Integration test에서는 사용하지 않는 Configuration을 Import받고있는 등 Configuration이 통일성이 없는 부분이 있던거죠. 이러한 문제는 AcceptanceTest에도 동일하게 발생했습니다.

처음에는  테스트 상단에 있는 어노테이션들을 일치시키면 동일한 환경이 구성되면서 이 문제가 해결될것이라고 생각했습니다. 그러나 이는 착각이었습니다. 위의 상황뿐만 아니라, `@MockBean` 을 사용해도 컨텍스트를 새로 만들고, 목킹된 객체의 개수, 종류에 따라서도 컨텍스트를 새로 만들어냅니다. 또한 `@import` 에 따라서도 컨텍스트를 새로 만듦니다. 지금 생각해보면 당연한건데 처음에는 진짜 어질어질했습니다.. 지금까지 이런걸 생각하지 않고 테스트 코드를 짰는데 얼마나 많은 성능적 손해를 본건지..

깃들다의 경우 OAuth를 통한 로그인을 구현하고 있기 때문에 로그인을 위해 `@MockBean`을 통해 로그인 관련 컴포넌트를 목킹시켜 사용하고 있었습니다. 따라서 위에서 말했던 모든 상황마다 새로운 Context를 만들어내면서 속도가 어마무시하게 느려졌습니다.

![image](https://user-images.githubusercontent.com/33603557/136337539-ef40d971-31d8-48fc-af39-db42422b50fb.png)

![image](https://user-images.githubusercontent.com/33603557/136337549-72187882-ec18-421a-a6dd-55ac0cdf32be.png)

![image](https://user-images.githubusercontent.com/33603557/136337568-d58e1692-aa7b-42b2-a1b9-bc58acbe8f3f.png)

*~~테스트를 느리게 만들었던 아이들 중 몇명...~~*

이 문제를 확인하고 테스트의 Configuration을 통일시켜주는 작업을 진행했습니다. 정말 사소한것 하나로도 새로운 Context가 생성되는걸 보면서 한숨이 푹푹 나왔습니다...

![image](https://user-images.githubusercontent.com/33603557/136337584-6796ee81-8340-4b02-9c50-c394842af0b1.png)

심지어 WebMvcTest의 경우에도 매 테스트 클래스마다 컨텍스트를 따로 다 만들어내는것을 확인했습니다. Mock되는 객체들이 서로 다르니 어쩔 수 없겠지요.. 일단 최대한 묶을 수 있는 테스트는 묶어서 Configuration을 최대한 맞춰주었습니다.

![image](https://user-images.githubusercontent.com/33603557/136337606-6f68cbb2-ce87-4f4c-8844-50f6956e2975.png)

WebMcvTest의 경우는 Controller Test에 필요한 모든 Mock과 Controller class 파일들을 abstract class에 몰아넣고 모든 ControllerTest(MockMvcTest)가 이를 상속받도록 했습니다. 어쩌피 컨트롤러딴 테스트이기 때문에 서로 각자의 Service에 관여할일이 없어 모든 Service를 목킹 하여도 괜찮다고 생각했기 때문입니다.

이러한 방식으로 AcceptanceTest, DataJpaTest, WebMvcTest, IntegrationTest를 각각 하나의 Context를 이용해서 테스트 가능하도록 리팩토링 해 주었습니다.

결론적으로, 이 방식을 적용했을때나 안했을때나 Intellij에 찍히는 테스트 시간을 큰 차이가 없습니다. 하지만 이는 Intellij가 중간에 새롭게 실행되는 Application Context Load시간을 테스트 시간에 누적하지 않아서 발생한 문제입니다. 실제로 적용해 보시면 속도차이가 많이 발생한다는 사실을 확인할 수 있을 것 입니다. 

> *이 글 시리즈의 제일 마지막에 GIF로 테스트 동작하는것을 녹화하여 올려놓았는데, 그것을 확인해보면 중간에 테스트가 잠깐씩 멈추는게 현저히 적은것을 확인할 수 있습니다(이 글에서 설명했듯,*  *AcceptanceTest, DataJpaTest, WebMvcTest, IntegrationTest마다 Application Context를 새로 Load하기 위해총 4번 멈춥니다). 그리고 테스트가 전체적으로 매끄럽게 돌아가고요. 그런 결과가 가능했던 이유가 이번 글에서 설명한 이 설정 덕분이었습니다. 어떤 차이인지 궁금하시다면 잠시 테스트최적화여행기 5편의 결과를 보고 오시면 좋을 것 같습니다* 🙂
> 

지금까지 저희 팀이 얼마나 많은Configuration을 가진 테스트를 진행한건지..  한번 테스트를 돌릴때 마다 얼마나 많은 Application Context가 생성된건지.. 제대로 신경써주지 못한 테스트코드에게 미안한 마음이 들었습니다.

마무리를 어떻게 지어야 할지 모르겠네요... 뭐, Spring 테스트의 Context 생성 조건을 잘 이해한다면 이런식으로도 테스트 시간을 줄일 수 있다는 사실을 알게 되어서 재미있었습니다.

그러면 다음에는 조회 관련 테스트를 분리하여 테스트 성능을 향상시키는 방법에 대해서 글을 작성해 보겠습니다. 읽어주셔서 감사합니다.
