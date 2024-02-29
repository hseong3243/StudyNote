#### MVC 패턴이란
---
MVC 패턴은 기존 비즈니스 로직과 뷰 로직이 모두 결합된 것에서 벗어나 모델, 뷰, 컨트롤러로 역할을 나누는 패턴입니다.

##### 컨트롤러(Controller)
HTTP 요청을 받아서 파라미터를 검증하고 비즈니스 로직을 호출합니다. 뷰에 전달할 결과 데이터를 조회해서 모델에 담습니다. 
##### 모델(Model)
뷰에 출력할 데이터를 담아둡니다. 뷰가 필요한 데이터를 모두 모델에 담아서 전달해주기 때문에 뷰는 비즈니스 로직이나 데이터 접근을 몰라도 되고 화면을 렌더링하는 일에만 집중할 수 있습니다.
##### 뷰(View)
모델에 담겨있는 데이터를 사용해서 화면을 그리는 일에 집중합니다.

#### 프론트 컨트롤러(Front Controller)
---
기존의 컨트롤러는 요청 하나마다 각각의 서블릿을 매핑한 것에 반해 프론트 컨트롤러 패턴은 하나의 서블릿으로 모든 클라이언트의 요청을 받습니다. 사용자의 요청이 들어오면 요청에 맞는 컨트롤러를 찾아서 호출합니다. 덕분에 코드 중복이 줄어들고 공통 처리가 가능합니다.

스프링 웹 MVC의 핵심이 프론트 컨트롤러 패턴이며 `DispatcherServlet`이 프론트 컨트롤러의 역할을 합니다.

#### 스프링 MVC 구조
---
![[스프링 MVC 구조.png]]
<출처: 김영한님의 스프링 MVC 2편>

스프링 MVC는 동작 순서는 다음과 같습니다.

1. 핸들러 조회: 핸들러 매핑을 통해 요청 URL에 매핑된 핸들러를 조회합니다.
2. 핸들러 어댑터 조회: 핸들러를 실행할 수 있는 핸들러 어댑터를 조회합니다.
3. 핸들러 어댑터 실행: 핸들러 어댑터를 실행합니다.
4. 핸들러 실행: 핸들러 어댑터가 실제 핸들러를 실행합니다.
5. ModelAndView 반환: 핸들러 어댑터는 핸들러가 반환하는 정보를 `ModelAndView`로 변환해서 반환합니다.
6. viewResolver 호출: 뷰 논리 이름을 물리 이름으로 바꾸기 위한 뷰 리졸버를 찾고 실행합니다.
7. View 반환: 뷰 리졸버는 뷰의 논리 이름을 물리 이름으로 바꾸고 렌더링을 위한 뷰 객체를 반환합니다.
8. 뷰 렌더링: 뷰를 통해서 뷰를 렌더링합니다.

##### 핸들러 매핑(HandlerMapping)
요청 URL에 매핑된 핸들러를 조회합니다.
```java
RequestMappingHandlerMapping  // 애노테이션 기반 컨트롤러에서 사용합니다.
BeanNameUrlHandlerMapping  // 스프링 빈의 이름으로 핸들러를 찾습니다.
```

##### 핸들러 어댑터(HandlerAdapter)
핸들러를 실행할 수 있는 핸들러 어댑터를 조회하고 실행하면서 핸들러 정보를 넘겨줍니다.
```java
RequestMappingHandlerAdpater  // 애노테이션 기반 컨트롤러에서 사용합니다.
HttpRequestHandlerAdapter  // HttpRequestHandler를 처리합니다.
```

##### 뷰 리졸버(ViewResolver)
핸들러 어댑터로부터 논리 뷰 이름을 획득합니다. 논리 뷰 이름을 가지고 `ViewResolver`를 순서대로 호출합니다.
```java
BeanNameViewResolver  // 빈 이름으로 뷰를 찾아서 반환합니다.
ThymeleafViewResolver  // 타임리프를 처리할 수 있는 뷰를 찾아서 반환합니다.
```

#### 어노테이션 기반 컨트롤러
---
스프링은 어노테이션을 활용하여 유연하고 실용적인 컨트롤러를 제공합니다. 핸들머 매핑과 핸들러 어댑터 중 가장 우선순위가 높은 것이 `@RequestMapping`을 사용하는 핸들러를 찾는 핸들러 매핑과 핸들러 어댑터입니다.

##### @Controller
스프링 MVC에서 어노테이션 기반 컨트롤러로 인식하며 해당 컨트롤러를 스프링 빈으로 등록합니다. 
##### @RequestMapping
요청 정보를 매핑합니다. 해당 URL이 호출되면 해당하는 메서드가 호출됩니다.
##### ModelAndView
모델과 뷰 정보를 담아서 반환하기 위한 객체입니다.

스프링 부트 3.0 이전에는 클래스 레벨에 `@RequestMapping`이 있어도 컨트롤러로 인식하였으나 3.0 이후부터는 클래스 레벨에 항상 `@Controller`가 있어야 스프링 컨트롤러로 인식됩니다. 

##### @RequestParam
HTTP 요청 파라미터 정보를 받기 위한 어노테이션입니다.
```java
@GetMapping("/hello")
public String hello(@RequestParam("number") Long number) {}
```
##### @PathVariable
URL 경로 변수를 받기 위한 어노테이션입니다. `@RequestMapping`은 URL 경로를 템플릿화 할 수 있으며 `@PathVariable`을 사용하여 매칭되는 부분을 편리하게 조회할 수 있습니다. `@PathVariable`의 이름과 파라미터 이름이 같으면 생략 가능합니다.
```java
@GetMapping("/members/{memberId}")
public String getMember(@PathVariable("memberId") Long memberId) {}
```
##### @RequestHeader
HTTP 헤더를 조회하기 위한 어노테이션입니다.
```java
@GetMapping("/members/{memberId}")
public String getMember(
	@RequestHeader MultiValueMap<String, String> headers,  // 모든 헤더를 조회
	@RequestHeader("Authorization") String authorization)  // Authorization 헤더를 조회
```
##### @CookieValue
쿠키를 조회합니다.
```java
@GetMapping("/refresh")
public String refreshAccessToken(
@CookieValue(value = "refreshToken") String refreshToken) {}
```
##### @ModelAttribute
해당 어노테이션을 사용하면 HTTP 요청 파라미터를 객체에 집어넣을 수 있습니다. 객체의 프로퍼티(stter)를 호출해서 파라미터의 값을 바인딩합니다.
```java
@GetMapping("/hello")
public String hello(@ModelAttribute HelloRequest request) {}
```
##### @RequestBody
해당 어노테이션을 사용하면 HTTP 메시지 바디 정보를 조회할 수 있습니다. `@ModelAttribute`는 생략 가능하나 `@RequestBody`는 생략이 불가합니다.
```java
@PostMapping("/hello")
public String hi(@RequestBody HiRequest request) {}
```

`@RequestBody`와 `@ResonseBody`는 HTTP 메시지 컨버터에 의해서 JSON을 객체로, 객체를 JSON으로 변환해줍니다.

#### HTTP 메시지 컨버터
---
![[HTTP 메시지 컨버터.png]]<출처: 김영한님의 스프링 MVC 1편>

스프링 MVC는 HTTP 요청에 `@RequestBody`, 응답에 `@ResponseBody`인 경우 HTTP 메시지 컨버터를 적용합니다.

HTTP 메시지 컨버터는 `RequestMappingHandlerAdapter`에 의해서 적용됩니다.
![[RequestMappingHandlerAdapter.png]]
<출처: 김영한님의 스프링 MVC 1편>

##### ArgumentResolver
핸들러 어댑터는 `ArgumentResolver`를 호출하여 핸들러가 필요로 하는 파라미터 값을 생성합니다. 파라미터 값이 모두 준비되면 핸들러를 호출하며 파라미터 값을 넘겨줍니다.

`@RequestBody`, `@ResponseBody`를 처리하는 `ArgumentResolver`가 HTTP 메시지 컨버터를 사용하여 필요한 객체를 생성합니다.
![[HTTP 메시지 컨버터2.png]]<출처: 김영한님의 스프링 MVC 1편>

#### Bean Validation
---
Bean Validation은 JSR-380이라는 기술 표준입니다. 대표적인 구현체가 하이버네이트 Validator입니다. starter 의존성을 이용하여 Bean Validation을 추가하면 스프링 부트는 `LocalValidatorFactoryBean`을 글로벌 Validator로 등록합니다.

#### 서블릿 필터
---
필터는 서블릿 실행 전에 실행됩니다. 특정 URL 패턴을 적용할 수 있습니다. 필터는 체인으로 구성되며 자유롭게 추가할 수 있습니다.
```
HTTP 요청 -> WAS -> 필터1 -> 필터2 -> 서블릿 -> 컨트롤러
```

#### 스프링 인터셉터
---
인터셉터는 스프링 MVC가 제공하는 기술로 서블릿 필터와 동일하게 공통 관심사를 해결할 수 있는 기능입니다. 인터셉터는 서블릿 실행 후에 실행됩니다.
```
HTTP 요청 -> was -> 필터 체인 -> 서블릿 -> 인터셉터 -> 컨트롤러
```

커스텀 인터셉터를 구현하기 위해서는 `HandlerInterceptor` 인터페이스를 구현하면 됩니다.
```java
@Slf4j  
@RequiredArgsConstructor  
public class JwtAuthenticationInterceptor implements HandlerInterceptor {  
  
    private static final String HEADER = "Authorization";  
    private static final String BEARER = "Bearer ";  
  
    private final JwtAuthenticationProvider jwtAuthenticationProvider;  
    private final AuthenticationContext authenticationContext;  
  
    @Override  
    public boolean preHandle(  
        HttpServletRequest request,  
        HttpServletResponse response,  
        Object handler) {  
        log.debug("[Auth] JWT 인증 인터셉터 시작");  
        String bearerAccessToken = request.getHeader(HEADER);  
        if(Objects.nonNull(bearerAccessToken)) {  
            log.debug("[Auth] JWT 인증 프로세스 시작");  
            String accessToken = removeBearer(bearerAccessToken);  
            JwtAuthentication authentication = jwtAuthenticationProvider.authenticate(accessToken);  
            authenticationContext.setAuthentication(authentication);  
            log.debug("[Auth] JWT 인증 프로세스 종료. 사용자 인증됨. {}", authentication);  
        }  
        log.debug("[Auth] Jwt 인증 인터셉터 종료");  
        return true;  
    }  
  
    private String removeBearer(String bearerAccessToken) {  
        if(!bearerAccessToken.contains(BEARER)) {  
            throw new ShoutLinkException("올바르지 않은 액세스 토큰 형식입니다.", ErrorCode.INVALID_ACCESS_TOKEN);  
        }  
        return bearerAccessToken.replace(BEARER, "");  
    }  
  
    @Override  
    public void afterCompletion(  
        HttpServletRequest request,  
        HttpServletResponse response,  
        Object handler,  
        Exception ex) {  
        authenticationContext.releaseContext();  
    }  
}
```

- `preHandle`: 핸들러 어댑터 호출 전에 호출됩니다. 반환 값이 true이면 다음으로 진행하고 false이면 진행하지 않습니다.
- `postHandle`: 핸들러 어댑터 호출 후에 호출됩니다.
- `afterCompletion`: 뷰가 렌더링 된 이후에 호출됩니다. 예외와 무관하게 항상 호출됩니다.

생성한 인터셉터는 `WebMvcConfigurer`의 `addInterceptors()`를 사용해서 등록할 수 있습니다.
```java
@Configuration  
@RequiredArgsConstructor  
public class WebConfig implements WebMvcConfigurer {  
  
    private final JwtAuthenticationProvider jwtAuthenticationProvider;  
    private final AuthenticationContext authenticationContext;  
  
    @Override  
    public void addInterceptors(InterceptorRegistry registry) {  
        registry.addInterceptor(  
                new JwtAuthenticationInterceptor(jwtAuthenticationProvider, authenticationContext))  
            .order(1)  
            .addPathPatterns("/api/**");  
    }

}
```

#### ArgumentResolver
---
커스텀 `ArgumentResovler`를 만들기 위해서는 어노테이션를 만들고 `HandleMethodArgumentResolver` 인터페이스를 구현하면 됩니다.
```java
@Target(ElementType.PARAMETER)  // 파라미터에만 적용합니다.
@Retention(RetentionPolicy.RUNTIME)  // 리플렉션 등을 활용할 수 있도록 런타임까지 어노테이션 정보가 남습니다.
public @interface LoginUser {  
  
}
```

```java
@RequiredArgsConstructor  
public class LoginUserArgumentResolver implements HandlerMethodArgumentResolver {  
  
    private final AuthenticationContext authenticationContext;  
  
    @Override  
    public boolean supportsParameter(MethodParameter parameter) {  
        boolean hasParameterAnnotation = parameter.hasParameterAnnotation(LoginUser.class);  
        boolean hasLongParameterType = parameter.getParameterType().isAssignableFrom(Long.class);  
        return hasParameterAnnotation && hasLongParameterType;  
    }  
  
    @Override  
    public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,  
        NativeWebRequest webRequest, WebDataBinderFactory binderFactory) {  
        Authentication authentication = authenticationContext.getAuthentication();  
        checkAuthenticated(authentication);  
        return authentication.getPrincipal();  
    }  
  
    private void checkAuthenticated(Authentication authentication) {  
        if(Objects.isNull(authentication)) {  
            throw new ShoutLinkException("인증되지 않은 사용자 요청입니다.", ErrorCode.UNAUTHENTICATED);  
        }  
    }  
}
```

- `supportsParameter()`: 어노테이션을 가진 파라미터이면서 지정된 타입이면 해당 리졸버가 사용됩니다.
- `resolveArgument()`: 컨트롤러 호출 직전에 호출 되어서 필요한 파라미터 정보를 생성해줍니다. 

인터셉터와 동일하게 `WebMvcConfigurer`에 리졸버를 추가해야 합니다.
```java
@Configuration  
@RequiredArgsConstructor  
public class WebConfig implements WebMvcConfigurer {  
  
    private final JwtAuthenticationProvider jwtAuthenticationProvider;  
    private final AuthenticationContext authenticationContext;  
  
    @Override  
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {  
        resolvers.add(new LoginUserArgumentResolver(authenticationContext));  
        resolvers.add(new NullableUserArgumentResolver(authenticationContext));  
    }
}
```

#### API 예외 처리
---
##### @ExceptionHandler
해당 어노테이션을 사용하면 해당 컨트롤러에서 발생한 예외와 그 자식 예외까지 모두 잡아서 공통 처리할 수 있습니다.

스프링의 우선순위는 항상 자세한 것이 우선권을 가집니다. 따라서 자식 예외를 처리하는 예외 핸들러가 존재하면 해당 핸들러를 우선합니다.
##### @ControllerAdvice
해당 어노테이션을 사용하면 모든 컨트롤러에서 발생하는 예외에 대한 공통 처리 지점을 만들 수 있습니다.
```java
@RestControllerAdvice
public class ExControllerAdvice {

	@ExceptionHandler(NoSuchElementException.class)
	public ResponseEntity<ErrorResponse> noSuchExHandle(NoSuchElementException e) {
		return new ErrorResponse("존재하지 않는 요소입니다.");
	}
}
```

`@ControllerAdvice`는 특정 컨트롤러를 지정할 수도 있고 패키지를 지정할 수도 있습니다.

#### 스프링 타입 컨버터
---
스프링은 쿼리 스트링, HTTP 메시지 바디, YML 정보 등 다양한 문자에 대해서 적절한 타입 변환을 수행해줍니다. 인터셉터와 아규먼트 리졸버와 동일하게 컨버터 또한 확장가능한 인터페이스를 제공합니다.
```java
@FunctionalInterface  
public interface Converter<S, T> {  
  
    T convert(S source);  
  
    default <U> Converter<S, U> andThen(Converter<? super T, ? extends U> after) {  
       Assert.notNull(after, "'after' Converter must not be null");  
       return (S s) -> {  
          T initialResult = convert(s);  
          return (initialResult != null ? after.convert(initialResult) : null);  
       };  
    }  
  
}
```

컨버터의 등록은 `WebMvcConfigurer`의 `addFormatters()`를 이용하여 등록할 수 있습니다.

#### 파일 업로드
---
HTTP는 문자와 바이너리를 동시에 전송하기 위한 `multipart/form-data`라는 전송 방식을 제공합니다. 스프링은 `MultipartFile`이라는 인터페이스를 통해 멀티파트 파일을 편리하게 이용할 수 있습니다.

## 참고
---
김영한님의 스프링 MVC