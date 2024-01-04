클래스 로더는 런타임 중에 클래스를 JVM에 동적으로 로드하는 역할을 담당합니다. 클래스 로더를 이용함으로써 JVM은 Java 애플리케이션을 실행하기 위해 기본 파일이나 파일 시스템에 대해 알 필요가 없습니다.

Java 클래스는 한 번에 메모리에 로드되지 않으며 애플리케이션이 필요할 때만 로드됩니다. 이 때, JRE가 Java ClassLoader를 호출하고 ClassLoader가 클래스를 메모리에 동적으로 로드합니다.


## 작동 방식
---
클래스 로더의 작동은 3 단계에 걸쳐 이루어집니다.

- Loading
- Linking
- Initialization

## 1. Loading
클래스 로더는 `.class` 파일을 읽고 해당 바이너리 데이터를 생성하여 method 영역에 저장합니다. 각 `.class` 파일은 다음과 같은 정보를 저장합니다.

- 로드된 클래스와 부모 클래스의 정규화된 이름
- 클래스 파일이 class, interface, enum 과 연관이 있는지 여부
- 제어자, 변수 그리고 메서드에 대한 정보 등

클래스가 JVM에 처음 로드되면 위의 정보들을 토대로 `Class` 타입의 객체가 생성됩니다. 해당 객체는 heap 영역에 저장되게 되며 프로그래머는 이를 이용하여 클래스 수준의 정보를 얻을 수 있습니다.

## 2. Linking
다음 세 단계의 작업을 수행합니다.

### Verification
`.class` 파일의 정확성을 보장합니다. 해당 파일이 올바른 형식인지, 유효한 컴파일러에 의해 생성되었는지 여부를 확인합니다.
확인 중에 오류가 발생하면 `VerifyError`가 발생합니다.

### Preparation
JVM은 클래스 정적 변수를 위한 메모리를 할당하고 메모리를 기본값으로 초기화합니다.

### Resolution
심볼릭 참조를 직접 참조로 대체합니다. 

## 3. Initialization
이 단계에서 모든 정적 변수에 코드와 정적 블록에서 정의된 값이 할당됩니다.


## 클래스 로더 종류
---
모든 클래스가 하나의 `ClassLoader`에 의해 로드되지는 않습니다. 클래스의 유형과 경로에 따라 특정 클래스를 로드하는 `ClassLoader`가 결정됩니다.

다음은 실제 실제 클래스들이 어떤 클래스 로더에 의해 로드되는지 알아보기 위한 예시입니다. 이를 토대로 클래스 로더의 종류에 대해 알아보겠습니다.

```java
import java.sql.DriverManager;

public class HelloWorld {
    public static void main(String[] args) {
        System.out.println(String.class.getClassLoader());
		System.out.println(DriverManager.class.getClassLoader());
        System.out.println(MyClass.class.getClassLoader());
    }
}
```
```java
null
jdk.internal.loader.ClassLoaders$PlatformClassLoader@404b9385
jdk.internal.loader.ClassLoaders$AppClassLoader@63947c6b
```

### 부트스트랩 클래스 로더
Java 클래스는 `java.lang.ClassLoader` 객체에 의해 로드됩니다. 그리고 해당 클래스 로더를 로드하기 위한 클래스 로더가 부트스트랩 클래스 로더입니다. 이는 JDK 내부 클래스를 로드하는 역할을 담당하며 다른 모든 클래스 로더 객체의 부모 역할을 수행합니다.

부트스트랩 클래스 로더는 코어 JVM의 일부로써 네이티브 코드로 작성됩니다. 예시에서도 볼수 있듯이 이는 클래스가 아니기 때문에 콘솔에도 `null`로 출력된 것을 확인할 수 있습니다.

### 플랫폼 클래스 로더
플랫폼 클래스 로더는 플랫폼 클래스의 로드를 담당합니다. 플랫폼 클래스는 Java SE platform API와 그 구현 클래스 그리고 JDK 관련 런타임 클래스를 포함합니다.

### 시스템 클래스 로더
애플리케이션 클래스 로더라고도 불립니다. 일반적으로 애플리케이션 클래스 경로, 모듈 경로 및 JDK 관련 도구에서 클래스를 JVM에 로드하는 작업을 처리합니다.


## 작동 원칙
---
클래스 로더는 다음 세 가지의 작동 원칙을 가지고 있습니다.
### 1. 위임 모델(Delegation Model)
클래스로더는 클래스를 찾으라는 요청이 있을 때 해당 요청을 부모 클래스로더에 위임합니다.

시스템 클래스로더는 해당 클래스의 로딩을 플랫폼 클래스로더에 위임하고, 플랫폼 클래스로더는 다시 부트스트랩 클래스로더에 요청을 위임합니다.

부트스트랩/플랫폼 클래스로더가 클래스 로딩에 실패한 경우에 시스템 클래스로더는 클래스 직접를 로드하려고 시도하고 실패한다면 `ClassNotFoundException`을 던집니다.

### 2. 고유 속성(Uniqueness Property)
클래스가 고유하고 반복되지 않도록 보장합니다.

부모 클래스로더가 로드한 클래스를 자식 클래스로더에 의해 로드되지 않도록 합니다. 부모 클래스로더가 찾을 수 없는 경우에만 자식 클래스로더가 클래스를 찾기 위한 시도를 합니다.

### 3. 가시성 원칙(Visibility Principle)
부모 클래스로더가 로드한 클래스는 자식 클래스로더에 표시됩니다. 그러나 자식 클래스로더가 로드한 클래스는 부모 클래스로더에 표시되지 않습니다.

예를 들어 시스템 클래스로더를 통해 `MyClass.class`를 로드했다면 플랫폼 클래스로더와 부트스트랩 클래스로더는 이를 볼 수 없습니다. 해당 클래스로더로 `MyClass.class`를 로드하려고 하면 `ClassNotFoundException` 예외가 발생할 것입니다.


## java.lang.ClassLoader
---
클래스 로더는 Java 런타임 환경의 일부입니다. JVM이 클래스를 요청하면 클래스 로더는 정규화된 클래스 이름(binary name)을 사용하여 런타임에 클래스 정의를 로드하려고 시도합니다.

`ClassLoader.loadClass()` 메서드는 런타임에 클래스 정의를 로드하는 책임을 가집니다. 이 메서드는 정규화된 이름을 기준으로 클래스를 로드하려고 시도합니다. 클래스가 로드되지 않았으면 부모 `classLoader`에게 요청을 위임합니다.

만일 부모 클래스 로더도 클래스를 찾지 못하면 `ClassLoader.findClass()`를 호출하여 자식 클래스가 정의한 방법에 의해 클래스를 찾기 위한 시도를 합니다.

결국 로드 할 수 없는 경우 `ClassNotFoundException`을 던집니다.

![[ClassLoader.loadClass().png]]

다음은 실제 `ClassLoader`의 코드입니다.
### loadClass()
```java
...
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // 첫 번째, 클래스가 이미 로드되었는지 확인한다.
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
	                    //부모 클래스 로더에게 클래스 로딩을 위임한다.
                        c = parent.loadClass(name, false); 
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // 부모 클래스 로더는 클래스를 찾지 못한 경우
                    // ClassNotFoundException을 던진다.
                }

                if (c == null) {  
                    // 부모 클래스 로더가 클래스를 찾지 못한 경우 findClass를 호출하여
                    // 클래스가 정의한 방법에 의해 클래스를 찾는다.
                    long t1 = System.nanoTime();
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
...
```




