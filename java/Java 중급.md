#### 스트림과 람다
---
스트림과 람다는 자바 8부터 새롭게 추가된 API입니다.
##### 스트림(stream)
스트림은 컬렉션을 처리하고 조작하는데 효율적인 방법을 제공합니다. 주요 이점은 다음과 같습니다.

- 루프와 조건문을 통해 동작을 구현할 필요 없이 선언형으로 동작의 수행을 구현할 수 있습니다.
- 여러 연산을 파이프라인으로 연결해서 복잡한 데이터 처리를 수행해도 높은 가독성과 명확성을 유지할 수 있습니다.
- 병렬 처리를 지원하기 때문에 성능을 향상시킬 수 있습니다.

##### 람다(lambda)
람다 표현식은 메서드로 전달할 수 있는 익명 함수를 단순화한 것입니다.
람다 표현식은 다음과 같이 람다 파라미터, 화살표, 람다 바디로 구성되어 있습니다.
```java
Comparator<Integer> compare = (a, b) -> a - b;  //오름차순 정렬
```

##### 함수형 인터페이스(functional interface)
함수형 인터페이스는 오직 하나의 추상 메서드만 가지는 인터페이스를 말합니다. 람다 표현식을 사용하면 함수형 인터페이스를 간결하게 표한할 수 있습니다. 함수형 인터페이스는 `Predicate`, `Comparator`, `Consumer`, `Function` 등 다양한 종류가 있습니다.

#### 리플렉션
---
리플렉션을 이용하면 클래스나 메서드의 메타정보를 동적으로 획득하고 코드도 동적으로 호출할 수 있습니다.
리플렉션은 런타임에 동작하기때문에 컴파일 시점에 오류를 잡을 수 없기 때문에 신중하게 사용해야 합니다.

#### JDK 동적 프록시
---
JDK 동적 프록시는 인터페이스를 기반으로 프록시를 동적으로 만들어주는 기술입니다. 프록시를 적용할 로직을 `InvocationHandler` 인터페이스를 사용하여 구현하면 됩니다.
```java
package java.lang.reflect;

public interface InvocationHandler {

   public Object invoke(Object proxy, Method method, Object[] args)
	  throws Throwable;

}
```

#### CGLIB
---
CGLIB는 바이트코드를 조작해서 동적으로 클래스를 생성하는 기술을 제공하는 라이브러리입니다. 이를 사용하면 인터페이스 없이 구체 클래스만 가지고도 동적 프록시를 만들어낼 수 있습니다. CGLIB는 프록시를 적용할 로직을 `MethodInterceptor` 인터페이스를 사용해서 구현하면 됩니다.
```java
package org.springframework.cglib.proxy;

public interface MethodInterceptor extends Callback {
  Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable;
}
```

CGLIB는 자식 클래스를 동적으로 생성하기 때문에 부모 클래스의 생성자와 `final` 키워드 사용에 주의해야 합니다.