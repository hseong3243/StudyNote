## 개요
---
프로젝트를 진행 초기에 반복적인 테스트 실행을 위해서 해결해야 하는 문제점이 두가지 있다고 판단하였습니다.

- 첫째, 애플리케이션 컨텍스트 초기화 횟수를 줄여 테스트 시간을 개선할 것
- 둘째, 문서화 코드 중복을 줄일 것

저는 해당 문제를 해결하기 위해 테스트 코드를 최적화한 테스트 컨벤션을 제시하였습니다. 프로젝트 종료 후 테스트 수행 시간을 체크하였을 때 약 30% 정도 테스트 시간이 감소한 것을 확인할 수 있었습니다. 이번 게시글에서는 상기한 문제의 해결 방법에 대해서 다루겠습니다.

## 컨텍스트 캐싱
---
스프링 부트는 통합 테스트를 지원하기 위해 다양한 어노테이션을 지원합니다. `@WebMvcTest`는 표현 계층 통합 테스트를 지원하기 위한 어노테이션으로  Spring MVC를 구성하는 빈들을 스캔하여 어플리케이션 컨텍스트를 구성합니다.

스프링은 반복적인 테스트를 위한 강력한 기능인 컨텍스트 캐싱을 제공합니다. configuration 파라미터의 조합을 key로 하여 어플리케이션 컨텍스트를 캐시합니다. 이후 동일한 configuration을 사용한다면 후속 테스트에서는 캐시된 컨텍스트를 재사용하여 테스트 시간을 상당히 줄일 수 있습니다.

문제는 configuration 파라미터의 조합이 조금이라도 달라지거나 `@MockBean`, `@SpyBean`의 사용여부에 따라서 캐시된 컨텍스트를 재사용하지 못하는 경우가 빈번하게 발생한다는 점입니다.

그렇기 때문에 `@WebMvcTest(UserController.class)`와 같이 단일 컨트롤러만 테스트하거나 개별 테스트 클래스마다 `@MockBean UserService userService`와 같이 목 객체를 사용하는 경우 컨텍스트 캐싱을 제대로 활용하지 못하는 상황이 발생할 수 있었습니다. 이러한 문제를 조기에 방지하기 위해서는 표현 계층의 통합 테스트 configuration을 동일하게 구성해야합니다.

이 문제를 해결하기 위한 방법은 추상 클래스 한 곳에 통합 테스트를 위한 설정을 모으고 서비스 객체들은 미리 `@MockBean`으로 선언하는 것입니다.

```java
@WebMvcTest  
public abstract class BaseControllerTest {  
  
    @Autowired  
    protected ObjectMapper objectMapper;  
  
    @Autowired  
    protected MockMvc mockMvc;  

    @MockBean  
    protected UserService userService;  
  
    ...  
  
    @MockBean  
    protected NotificationService notificationService;  
  
    protected static final String AUTHORIZATION = "Authorization";  
  
    protected String accessToken;  
  
    @BeforeEach  
    void authenticationSetUp() {  
        accessToken = AuthFixture.accessToken();  
    }  
}
```

이제 컨트롤러 테스트 작성 시 `BaseControllerTest`를 상속하면 됩니다. 최초 1회 스프링 MVC를 구성하는 빈들이 초기화된 후  `BaseControllerTest`를 상속한 모든 테스트 클래스는 캐시된 컨텍스트를 사용하여 이전보다 빠른 속도로 테스트를 수행하게 됩니다. 또한, `ObjectMapper`와 `MockMvc`등 항상 같이 사용되는 객체들도 추상 클래스에 선언되어 있기 때문에 테스트 작성에만 집중할 수 있는 장점도 있습니다.

![[스크린샷 2023-10-04 오후 6.32.55.png]]

![[스크린샷 2023-10-04 오후 6.31.53.png]]

다음은 프로젝트 종료 후 intellij profiler를 이용하여 테스트 수행 시간이 얼마나 개선되었는지 측정한 결과입니다. 비교를 위하여 도커를 띄우지 않았기 때문에 테스트 컨테이너를 사용하는 테스트 3개는 실패합니다.

![[스크린샷 2023-10-04 오후 5.11.53.png]]
![[스크린샷 2023-10-04 오후 6.20.22.png]]

테스트 수행 시간 자체는 큰 차이가 나지 않으나 컨텍스트가 초기화되는 시간을 포함하면 18초에서 12초로 개선된 것을 확인할 수 있습니다. 또한, 테스트 컨텍스트의 경우에는 기존 17개에서 5개로 줄어들었습니다.

## 반복되는 문서화 코드
---
프로젝트를 진행하면서 저희 팀은 문서화 도구로 RestDocs를 선택하였습니다. RestDocs는 문서화를 위해 표현 계층 테스트를 강제하고 다른 문서화 도구인 Swagger와 달리 컨트롤러로부터 문서화 코드를 분리할 수 있다는 장점이 주요한 이유였습니다.

표현 계층 테스트를 작성하다보니 가장 귀찮았던 부분은 `print()`와 `prettyPrint()` 였습니다. 

```java
@Test  
@DisplayName("성공")  
void findUser() throws Exception {  
    //given  
    ...
  
    //when  
    ResultActions resultActions = mockMvc.perform(get(...)
	    ...);
  
    //then  
    resultActions.andExpect(status().isOk())  
        .andDo(print())  
        .andDo(document("find-user",  
                preprocessRequest(prettyPrint()),  
                preprocessResponse(prettyPrint()),  
                requestHeaders(...),  
                responseFields(...)  
            )  
        );  
}
```

`print()`는 실행된 요청의 결과에 대한 세부사항을 출력하는 정적 메서드입니다. 다음과 같이 요청과 응답에 대한 결과를 보여주기에 애용하고 있습니다.

![[스크린샷 2023-10-04 오후 6.42.29.png]]

`prettyPrint()`은 RestDocs 문서화 시 요청과 응답의 내용을 이름 그대로 예쁘게 출력해주는 메서드입니다.  마찬가지로 애용하고 있습니다.

이 둘은 테스트 작성시 항상 추가하는 코드이기 때문에 가능하면 제거하고 싶었습니다. 우선 문서화와 관련된 코드를 제거하기 위해 다음과 같은 configuration을 작성합니다.

```java
@TestConfiguration  
public class RestDocsConfig {  
  
    @Bean  
    public RestDocumentationResultHandler write() {  
        return MockMvcRestDocumentation.document(  
            "{method-name}",  
            preprocessRequest(prettyPrint()),  
            preprocessResponse(prettyPrint())  
        );  
    }  
}
```

위의 설정은 요청과 응답의 문서화에 `prettyPrint()`를 항상 적용할 뿐만 아니라 케밥 케이스(-)가 적용된 메서드 이름으로 스니펫이 생성되게 됩니다.

이제 앞서 작성한 추상 클래스에 Import 해준뒤 다음과 같이 mockMvc를 구성해줍니다.

```java
@WebMvcTest  
@Import(RestDocsConfig.class)  
@ExtendWith(RestDocumentationExtension.class)  
public abstract class BaseControllerTest {  
  
    ...
	@Autowired  
	protected RestDocumentationResultHandler restDocs;
	...
  
    @BeforeEach  
    void mockMvcSetUp(  
        final WebApplicationContext context,  
        final RestDocumentationContextProvider provider) {  
        this.mockMvc = MockMvcBuilders.webAppContextSetup(context)  
            .alwaysDo(print())  
            .alwaysDo(restDocs)  
	        .apply(MockMvcRestDocumentation.documentationConfiguration(provider))  
            .build();  
    }  
}
```

`alwaysDo()`는 모든 응답에 항상 적용되어야 하는 global action을 정의하는 메서드입니다. 해당 메서드의 인자로 `print()`와 `RestDocumentationResultHandler` 인스턴스를 전달함으로써 더 이상 표현 계층 테스트 코드를 작성하며 `andDo(print())`와 `preprocessRequest(prettyPrint())`, `preprocessResponse(prettyPrint())`를 직접 작성할 필요 없이 항상 적용될 것입니다.

`apply()`는 `MockMvc` 설정을 자동화하는데 사용되는 메서드입니다. 인자로 문서화를 위한 자동화하기 위한 `MockMvcConfigurer`를 전달하여 `@AutoConfigureRestDocs` 어노테이션을 대체합니다.

이제 `BaseControllerTest`를 상속한 테스트 클래스는 `RestDocumentationResultHandler`의 인스턴스 변수인 `restDocs`를 사용하여 중복 코드를 작성할 필요 없이 문서화를 위한 핵심 코드만 작성하면 됩니다.

```java
@Test  
@DisplayName("성공")  
void findUser() throws Exception {  
    //given  
    ...
  
    //then  
    resultActions.andExpect(status().isOk())  
        .andDo(restDocs.document(  
                requestHeaders(...),  
                responseFields(...)  
            )  
        );  
}
```

그럼 자동으로 메서드 이름에 케밥 케이스가 적용되어 `find-user`라는 문서가 생성됩니다. 요청과 응답 문서에도 `prettyPrint()`가 적용되고 콘솔에도 요청, 응답에 대한 세부사항이 출력됩니다.

## 맺으며
---
본 게시글에서는 표현 계층 통합 테스트의 중복 코드 제거와 컨텍스트 캐싱에 대해 다루었습니다. 컨텍스트 재사용은 `@DataJpaTest`나 `@SpringBootTest`도 전체적인 맥락은 동일합니다. 테스트 코드간에 동일한 configuration을 사용하면 컨텍스트 초기화를 최소화할 수 있습니다.

현재 프로젝트는 종료되었으나 영속성 계층 테스트와 테스트 컨테이너를 사용한 테스트 코드가 있어 해당 부분 역시 컨텍스트를 재사용 할 수 있도록 리팩토링을 진행하고 있습니다. 아쉬운 점은 인수 테스트를 많이 작성하지 못하여 컨텍스트 캐싱의 강력함에 대해 체감할 기회가 적었습니다.

다음에 진행할 프로젝트에서는 좀 더 풍성한 테스트 코드 작성하고 듬직한 애플리케이션을 만들어보고 싶습니다. 그럼 최종 프로젝트를 진행하며 다시 돌아오도록 하겠습니다.

## 참고
---
https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#features.testing.spring-boot-applications

https://docs.spring.io/spring-framework/reference/testing/testcontext-framework/ctx-management/caching.html

