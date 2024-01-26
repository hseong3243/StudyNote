## 테스트 더블
---
### 목(Mock)
목은 객체가 기대한 대로 상호작용하는지 확인하기 위한 객체입니다. 자바에서 주로 사용하는 `Mockito`를 이용하면 쉽게 목 객체를 생성할 수 있습니다. 
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

## mock 객체를 이용한 스터빙의 문제점
---
테스트 더블을 이용한 테스트 방법에 대해 찾아다니다 보면 테스트 스타일에 따라 두 가지로 나뉜다는 이야기를 볼 수 있습니다. 
### Mockist
- 모키스트는 모든 협력 객체에 대해 mock을 사용합니다.
### Classical
- 고전적인 스타일은 가능하면 실제 객체를 사용합니다. 만일 실제 객체를 사용하기 어려운 경우에는 테스트 대역을 사용합니다.

저의 경우에는 Mockist라 할 수 있습니다. 그동안 프로젝트를 진행하면서 서비스 계층의 단위 테스트를 수행하는 경우 모든 협력 객체를 mocking하여 사용하곤 했습니다. 물론 상태 검증이 어려운 경우를 제외하곤 스터빙을 수행하여 상태 검증을 수행하기는 했지만 여전히 대부분의 협력 객체는 목 객체로 구성되어 있었습니다.
### 부족한 리팩토링 내성
처음에는 `given()`을 이용한 스터빙은 직접 스텁 객체를 만들지 않고 목 객체를 이용하여 간단하게 테스트를 구성할 수 있는 것처럼 보였습니다. 그리고 시간이 지나면서 하나, 둘 리팩토링을 수행하기 시작하면 곳곳에서 테스트가 실패하는 것을 확인할 수 있습니다.

이는 Mockist 스타일의 가장 큰 문제점입니다. 테스트 코드에서 어떤 메서드가 호출되고, 어떤 값을 반환할지 명확하게 명시하기 때문에 테스트 대상이 되는 메서드의 코드가 조금이라도 변하면 관련된 모든 테스트가 실패할 가능성을 가지고 있습니다. 

우리가 작성한 테스트 코드는 프로덕션 코드의 **리팩토링 후에도 기능이 잘 작동한다는 것을 보장할 수 있어야 합니다.** 테스트의 대상이 되는 메서드의 파라미터 목록이나 반환 타입이 달라진다면 테스트 코드 역시 달라지는 것이 당연합니다. 하지만 `given()`으로 다른 객체와의 상호 작용을 세세하게 명시함으로 인해 새로운 협력 객체가 추가되거나, 기존에 호출하던 메서드의 시그니처가 약간이라도 변화하게 되면 순식간에 테스트는 실패하게 됩니다. 테스트 코드가 올바른 리팩토링을 수행했는지에 대한 지표가 되지 못합니다.

## 참고
---
https://martinfowler.com/articles/mocksArentStubs.html