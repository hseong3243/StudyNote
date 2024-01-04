실행 엔진은 `.class`를 실행합니다. 실행 엔진은 세 부분으로 분류할 수 있습니다.

- Interpreter
- JIT Compiler
- Garbage Collector

## Interpreter
바이트 코드를 한 줄 씩 읽어 기계어로 변환합니다. 메서드를 호출할 때마다 매번 기계어로 변환하여 속도가 느리다는 단점을 가지고 있습니다.

## JIT(Just-In-Time) compiler
실행 엔진이 반복되는 메서드 호출을 발견할 경우 JIT 컴파일러가 전체 바이트코드를 네이티브 코드로 컴파일합니다. 이를 이용하여 Java 애플리케이션의 성능을 향상시킬 수 있습니다.
