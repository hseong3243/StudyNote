## JVM이란 무엇인가?
자바 가상 머신(Java virtual machine, JVM)은 컴퓨터 상에서 Java 바이트코드로 컴파일 된 프로그램을 실행할 수 있도록 해주는 가상 머신입니다. 개발자가 작성한 주요 메서드를 실제로 호출하는 역할을 담당합니다.

JVM의 존재 덕분에 우리가 윈도우에서 개발을 하든, 맥에서 개발을 하든 OS에 관계없이 어디서든 실행할 수 있는 애플리케이션을 개발할 수 있습니다. 이러한 특징으로 인해 Java 애플리케이션을 Write Once Use Anywhere라고도 합니다.

## JVM Architecture
JVM은 다음 세가지 구성 요소로 이루어져 있습니다.

- [[Class Loader]]
- [[Runtime data area]]
- [[Execution engine]]
