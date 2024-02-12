그동안 프로젝트를 진행하고 테스트 코드를 작성하면서 협력하는 객체들은 주로 목킹(mocking)을 했습니다. 실제 구현체 대신 목 객체를 사용하면서 개인적으로 느끼는 바가 있었습니다. 이번 글에서는 테스트 더블 4가지와 느낀 점에 대해 한 번 정리해보려 합니다. 

## 테스트 더블(Test Double)
---
테스트 더블(테스트 대역)은 프로덕션 코드에서 실제 상호작용하는 객체 대신 테스트 코드에서 사용하는 가짜 객체들을 뜻합니다. 영화의 스턴트맨처럼 테스트 코드에서 실제 객체 대신 활약하는 스턴트맨의 역할을 수행합니다.
### 목(Mock)
목은 객체가 기대한 대로 상호작용하는지 확인하기 위한 객체입니다. 자바에서는 `Mockito`를 이용하면 쉽게 목 객체를 생성할 수 있습니다. 
```java
@ExtendWith(MockitoExtension.class)
class AuthServiceTest {

	@InjectMocks
	private Authservice authService;

	@Mock
	private MemberRepository memberRepository;
}
```

목을 이용한 테스트에서는 주로 행위 검증을 수행할 수 있습니다. 다음과 같이 `verify()` 또는 BDD 스타일의 `then()`을 이용하면 메서드의 호출이 정상적으로 이루어졌는지 검증할 수 있습니다.
```java
@Test
@DisplayName("성공: 회원이 저장됨")
void createMember() {
	//given
	CreateMemberCommand command = new CreateMemberCommand("email@email.com", "asdf123!");

	//when
	authService.createMember(command);

	//then
	then(memberRepository).should().save(any());
}
```

### 스텁(Stub)
스텁은 단순하게 구현되어 미리 준비된 답변만을 반환하는 객체입니다. `save()` 메서드를 여러번 호출하며 다양한 `member`를 인수로 전달한다고 해도 `findAll()`이 반환하는 리스트에는 처음에 스터빙(stubbing) 한 `stubMember` 하나만 들어있는 리스트를 반환합니다.

직접 스텁 객체를 구현해줄 수도 있지만 `Mockito`가 제공하는 기능을 이용하여 테스트에서 사용하는 메서드에 대해서만 스터빙해줄 수도 있습니다. `when()` 또는 BDD 스타일의 `given()` 중 선호하는 스타일에 따라 다음과 같이 작성할 수 있습니다.
```java
@Test  
@DisplayName("성공: 액세스 토큰, 리프레시 토큰 반환")  
void login() {  
    //given  
    LoginCommand loginCommand = new LoginCommand("email@email.com", "asdf123!"); 
    Member member = new Member(  
        "email@email.com",  
        passwordEncoder.encode("asdf123!"),  
        "nickname",  
        MemberRole.ROLE_USER);
    given(memberRepository.findByEmail(any())).willReturn(Optional.of(member));
    //when(memberRepository.findByEmail(any())).thenReturn(Optional.of(member)); 
      
    //when  
    LoginResponse response = authService.login(loginCommand);  
  
    //then  
    assertThat(response.accessToken()).isNotBlank();  
    assertThat(response.refreshToken()).isNotBlank();  
}
```

테스트의 목적에 따라서 행위 검증을 수행할 수도 있고 `junit` 또는 `assertj`를 이용해 상태 검증을 수행할 수도 있습니다.

### 스파이(Spy)
스파이는 얼마나 많은 호출이 이루어졌는가를 기록하는 스텁입니다. 예를 들면 알림 서비스의 알림 발신 메서드의 호출이 몇 번이나 이루어졌는지 기록하는데 사용할 수 있습니다.
```java
public class NotificationServiceStub implements NotificationService {

	private int count = 0;

	@Override
	public void send(NotificationCommand command) {
		count++;
	}

	public int getSendCount() {
		return count;
	}
}
```

### 페이크(Fake)
페이크 객체는 실제 동작하는 구현을 가지고 있지만 프로덕션에는 적합하지 않은 객체입니다. `HashMap`을 이용한 메모리 저장소가 예가 될 수 있습니다.
```java
public final class FakeMemberRepository implements MemberRepository {  
  
    private List<Member> memory = new ArrayList<>();  
  
    public StubMemberRepository(Member... members) {  
        memory.addAll(Arrays.stream(members).toList());  
    }  
  
    @Override  
    public Optional<Member> findByEmail(String email) {  
        return memory.stream().filter(member -> member.getEmail().equals(email)).findFirst();  
    }  
  
    @Override  
    public Optional<Member> findByNickname(String nickname) {  
        return memory.stream().filter(member -> member.getNickname().equals(nickname)).findFirst();  
    }  
  
    @Override  
    public Long save(Member member) {  
	    memory.add(member);  
        return memory.size();
    }  
}
```

단, JPA를 사용하는 경우 JpaRepository를 상속하는 인터페이스로는 오버라이딩 해야 할 메서드만 한 무더기이기에 위와 같은 페이크 또는 스텁 객체를 구현하는데 어려움이 따릅니다.
![[JpaReposiotry 확장.png]]

따라서 직접 구현체를 만들어 테스트에 이용하려는 경우에는 JpaRepository를 상속하는 인터페이스에 의존하는 대신 별도의 인터페이스를 추가하여 영속성 계층을 구성하는 방식으로 풀어나갈 수 있습니다.
![[Drawing 2024-01-25 23.02.17.excalidraw.png]]

## mock 객체를 이용한 스터빙의 문제점
---
테스트 더블을 이용한 테스트 방법에 대해 찾아다니다 보면 테스트 스타일에 따라 두 가지로 나뉜다는 이야기를 볼 수 있습니다. 
### Mockist
- 모키스트는 모든 협력 객체에 대해 mock을 사용합니다.
### Classical
- 고전적인 스타일은 가능하면 실제 객체를 사용합니다. 만일 실제 객체를 사용하기 어려운 경우에는 테스트 대역을 사용합니다.

저의 경우에는 Mockist라 할 수 있습니다. 그동안 프로젝트를 진행하면서 서비스 계층의 단위 테스트를 수행하는 경우 모든 협력 객체를 mocking하여 사용하곤 했습니다. 물론 상태 검증이 어려운 경우를 제외하곤 스터빙을 수행하여 상태 검증을 수행하기는 했지만 스터빙을 했다고 해서 목 객체가 스텁 객체로 변하지는 않습니다. 이러한 **Mockist 스타일의 가장 큰 문제점은 리팩토링 내성이 부족**하다는 것입니다.

처음에는 `given()`을 이용한 스터빙은 직접 스텁 객체를 만들지 않고 목 객체를 이용하여 간단하게 테스트를 구성할 수 있는 것처럼 보였습니다. 그리고 시간이 지나면서 하나, 둘 리팩토링을 수행하기 시작하면 곳곳에서 테스트가 실패하는 것을 확인할 수 있습니다.

테스트 코드에서 어떤 메서드가 호출되고, 어떤 값을 반환할지 명확하게 명시하기 때문에 테스트 대상이 되는 메서드의 코드가 조금이라도 변하면 관련된 모든 테스트가 실패할 가능성을 가지고 있습니다. 

우리가 작성한 테스트 코드는 프로덕션 코드의 **리팩토링 후에도 기능이 잘 작동한다는 것을 보장할 수 있어야 합니다.** 테스트의 대상이 되는 메서드의 파라미터 목록이나 반환 타입이 달라진다면 테스트 코드 역시 달라지는 것이 당연합니다. 하지만 `given()`으로 다른 객체와의 상호 작용을 세세하게 명시함으로 인해 **새로운 협력 객체가 추가**되거나, **스터빙 한 메서드의 시그니처가 변경**되면 순식간에 테스트는 실패하게 됩니다. 테스트 코드가 올바른 리팩토링을 수행했는지에 대한 지표가 되지 못합니다.

## 결론
---
선택할 수 있는 대안은 고전적인 스타일로 실제 구현체를 이용하여 테스트를 작성하는 것입니다. 물론 데이터베이스를 붙이거나 외부 API 호출을 수행할 수는 없습니다. 대신 적절하게 스텁, 페이크 객체를 구현하여 협력 객체를 대체할 수 있습니다.
```java
@ParameterizedTest  
@CsvSource({"asdf123!", "asdfasdf12341234!@#$"})  
@DisplayName("성공: 비밀번호가 영어 소문자, 특수문자, 숫자를 모두 포함하여 8자 이상, 20자 이하이다.")  
void createMember(String validPassword) {  
    //given (given()을 이용한 스터빙)
    CreateMemberCommand createMemberCommand  
        = new CreateMemberCommand("email@email.com", validPassword, "닉네임"); 
	given(memberRepository.findByEmail(any())).willReturn(MemberFixture.member());
	given(memberRepository.findByNickname(any())).willReturn(MemberFixture.member());
  
    //when  
    CreateMemberResponse response = authService.createMember(createMemberCommand);  
  
    //then  
    assertThat(response.memberId()).isNotNull();  
}
```

```java
@BeforeEach  
void setUp() {   
    memberRepository = new FakeMemberRepository(MemberFixture.member());  
    authService = new AuthService(memberRepository, passwordEncoder, jwtProvider);  
}

@ParameterizedTest  
@CsvSource({"asdf123!", "asdfasdf12341234!@#$"})  
@DisplayName("성공: 비밀번호가 영어 소문자, 특수문자, 숫자를 모두 포함하여 8자 이상, 20자 이하이다.")  
void createMember(String validPassword) {  
    //given (Fake 객체 사용)
    CreateMemberCommand createMemberCommand  
        = new CreateMemberCommand("email@email.com", validPassword, "닉네임"); 
  
    //when  
    CreateMemberResponse response = authService.createMember(createMemberCommand);  
  
    //then  
    assertThat(response.memberId()).isNotNull();  
}
```

몇 번인가 리팩토링을 시도하던 순간에 `given()`으로 인해 테스트 코드에 순식간에 빨간뿔이 들어오던 상황을 경험한 적이 있습니다. `Mockito`를 사용하는 대신 테스트용 구현체를 만드는데 좀 더 시간이 걸릴 수는 있겠지만 여러 개의 테스트 코드를 변경하는 대신 테스트용 구현체 하나만 관리하면 조금 더 자유로운 리팩토링을 수행할 수 있지 않을까 생각합니다.

## 참고
---
https://martinfowler.com/articles/mocksArentStubs.html