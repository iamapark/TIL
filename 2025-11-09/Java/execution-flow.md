# Java Class Loader

## 이 주제를 공부하는 이유
- Youtube에 'java class loader'를 검색해서 이 영상을 찾음
: https://www.youtube.com/watch?v=BeMi8K0AFAc
- 이 영상을 보면서 이 공부 노트를 적음. class loader에 국한된 내용이 아니고 JVM 전반에 관한 내용

## 배운 내용

### Class Loader: Loading
#### Jvm executes class from the following sources:
  - Java Runtime (platform classes)
  - Application classpath
  - Auto generated on-the-fly(Proxy, Reflection accessors, invoke dynamic implemenatation)
  - Provided by the application itself

#### Every class is loaded by a class loader:
  - Platform classes are loaded by the bootstrap class loader
  - Classes from application classpath are loaded by the system class loader (AppClassLoader <= jdk.internal.loader.ClassLoaders$AppClassLoader)
  - Application classes may create user-defined class loaders

#### JVM Startup
  - main class ia loaded by system class loader (from application classpath or modulepath): Provokes loading of a part of platform classes like `j.i.Object` and its dependencies
  - public static void amin (String[] args) of the main class is executed

#### Class loading process (creation of a class)
  - Class file parsing: Class format is checked on correctness
  - Create of a runtime representation of the class in a special JVM memory area: runtime constant pool in Method Area aka `Meta Space` aka `Permanent Generation`
  - Loading of a superclass and superinterfaces

#### Class loader 계층 구조
```java
public class ClassLoaderDemo {
  public static void main(String[] args) {
    // String 클래스 (Java 핵심 클래스)
    ClassLoader stringLoader = String.class.getClassLoader();
    System.out.println("String: " + stringLoader);
    // → null (Bootstrap이 로드, Java에서 null로 표현)

    // ArrayList 클래스 (Java 표준 라이브러리)
    ClassLoader listLoader = ArrayList.class.getClassLoader();
    System.out.println("ArrayList: " + listLoader);
    // → null (Bootstrap이 로드)

    // 현재 클래스 (사용자 애플리케이션)
    ClassLoader appLoader = ClassLoaderDemo.class.getClassLoader();
    System.out.println("ClassLoaderDemo: " + appLoader);
    // → jdk.internal.loader.ClassLoaders$AppClassLoader

    // 부모 확인
    System.out.println("Parent of App: " + appLoader.getParent());
    // → jdk.internal.loader.ClassLoaders$PlatformClassLoader

    System.out.println("Parent of Platform: " + appLoader.getParent().getParent());
    // → null (Bootstrap)
  }
}
```
  - String.class.getClassLoader()가 null로 표현되는 이유
    - Java 핵심 표준 라이브러리(ex: `java.lang.*`, `java.util.*`, `java.io.*` 등)는 Bootstrap Classloader가 로드함
    - Bootstrap ClassLoader는 Java 클래스가 아님. Bootstrap ClassLoader는 JVM 자체의 일부이며 네이티브 플랫폼 언어(C 코드)로 작성되어 있어 바이트코드가 아니므로 Java 코드에서 표현할 수 없음
    - JVM 설계자들이 `null`을 Bootstrap의 표현으로 결정
  - ClassLoaderDemo.class.getClassLoader()
    - ClassLoaderDemo는 사용자가 작성한 코드
    - Application ClassLoader가 로드
    - Java 9+에서는 `jdk.internal.loader.ClassLoaders$AppClassLoader`
  - appLoader.getParent()
    - Application ClassLoader의 **부모**는 Platform ClassLoader
  - appLoader.getParent().getParent()
    - Platform ClassLoader의 부모는 Bootstrap
    - Bootstrap은 `null`로 표현됨
  

#### 전체 계층 시각화
```
              null
(Bootstrap ClassLoader - 네이티브 코드)
                ↑
                │ .getParent()
                │
PlatformClassLoader@5cbc508c
(Platform ClassLoader - Java 클래스)
                ↑
                │ .getParent()
                │
AppClassLoader@30946e09
(Application ClassLoader - Java 클래스)
                ↑
                │ .getClassLoader()
                │
      ClassLoaderDemo.class
      (사용자 클래스)
```

#### 시각화: ClassLoader 로딩 순서
```
[JVM 시작 시점]

네이티브 코드 (C/C++)
┌──────────────────────────────┐
│  Bootstrap ClassLoader       │
│  (JVM에 컴파일되어 내장됨)     │
└──────────────────────────────┘
         │
         │ (로드)
         ↓
┌──────────────────────────────┐
│  java.lang.Object            │
│  java.lang.Class             │
│  java.lang.ClassLoader ⭐    │ ← 여기!
│  java.lang.String            │
│  java.util.*                 │
└──────────────────────────────┘
         │
         │ (인스턴스 생성)
         ↓
┌──────────────────────────────┐
│  PlatformClassLoader 객체    │
│  extends ClassLoader         │
└──────────────────────────────┘
         │
         │ (인스턴스 생성)
         ↓
┌──────────────────────────────┐
│  AppClassLoader 객체         │
│  extends ClassLoader         │
└──────────────────────────────┘
         │
         │ (사용자 클래스 로드)
         ↓
┌──────────────────────────────┐
│  YourMainClass               │
└──────────────────────────────┘
```

#### Web application(ex: Tomcat) class loading 예시
```
Tomcat의 ClassLoader 계층:

Bootstrap
    ↓
Platform
    ↓
Application (Tomcat 자체)
    ↓
Common ClassLoader
    ↓
WebApp ClassLoader 1 (app1.war)
    ↓
WebApp ClassLoader 2 (app2.war)
```

### Class Loader: Linking
  - Java bytecode verification
  - Preparation
  - Resolution of symbolic refrences 

### Class Loader: Initialization
Call of a class static initializer

Class Initizliation
  - Happends on first use:
    - new
    - static field access
    - static method call
  - Provokes initizliation of a super class and super interfaces with default methods

### Execution engine

#### Java bytecode execution
JVM may execute bytecode via:
  - Interpretation
  - Translation into native code, that will run directly on CPU

#### Compilers
  - Dynamic (Just-In-Time -- JIT): Translation into native code happens at application runtime
  - Static (Ahead-Of-Time -- AOT): Translation happens before prgoram execution

#### Dynamic Compilers (JIT)
  - Work concurrently with program execution
  - Compile hot code only
  - Hot code is determined be means of profiling
  - Profiling information is used for optimal optimizations (speculative)

#### Static Compilers (AOT)
  - are not limited in resources for optimizations
  - compile every method of a program using the most aggressive optimizations
  - No overheads at run-time (fast startup)

### Meta information subsystem

#### Reflection
  - allows access to classes, fields, methods via name (by string literal) from a Java program
  - is implemented in the JVM via access to Meta space
  - Key feature of Java for many popular frameworks and JVM-based programming languages implementations (Groovy, Clofure, Ruby, etc.)

## 📌 질문 1: 전체 흐름의 큰 그림
```
질문:
"자바 프로그램이 작성된 후 실행되기까지의 전체 과정을 큰 그림으로 설명해주세요. 주요 단계가 무엇이고, 각 단계에서 무슨 일이 일어나나요?"

설명 팁:

  - .java 파일을 작성한 후부터 시작해주세요
  - 각 단계의 이름과 역할을 설명
  - 전문 용어를 사용하시되, 간단히 풀어서 설명
  - 왜 이런 여러 단계가 필요한지 포함하면 더 좋습니다!
```

### 설명:

1. Compile
  - `.java` 파일이 작성되고 `javac` 명령어로 컴파일 됩니다. 이 컴파일 단계를 거쳐 `.class` 파일이 만들어집니다.
  - 이 `.class` 파일을 bytecode라고 합니다. 이 bytecode는 CPU가 직접 실행할 수 없으며 JVM에 의해서만 실행될 수 있습니다.
  - OS와 실제 소스코드(java) 사이에 JVM 이라는 단계를 둬서 한 번 작성된 코드가 어떤 실행 환경(OS)이든지 정상적으로 실행될 수 있게 할 수 있는 것이 바로 JVM 언어(ex: Java, Groovy, Kotlin)의 장점입니다.

2. Class Loader (Loading)
  - `java` 명령어로 bytecode가 실행됩니다.
  - 가장 먼저 Loading 이라는 단계를 실행합니다.
  - 제일 먼저 native code로 작성된 Bootstrap ClassLoader가 실행됩니다. 이 Bootstrap ClassLoader는 JVM의 가장 기초적인 모듈들(ex: `java.lang.*`, `java.util.*` 등)을 로딩합니다.
  - Bootstrap ClassLoader는 또한 Platform ClassLoader를 로딩합니다. Platform ClassLoader는 Application ClassLoader를 로딩합니다.
  - 컴파일 결과인 bytecode는 Classpath에 포함되어 실행되어야 하고 이 Classpath에 포함된 bytecode가 Application ClassLoader에 의해 로딩됩니다.
  - Class Loader는 `Delegation Method`를 따릅니다.
    - Application -> Platform -> Bootstrap 순서로 클래스 로드를 시도합니다. 즉, 하위(자식) 로더는 부모 로더에게 클래스 로딩을 위임하는 것입니다.
    - 특정 클래스를 로드해야 하는 상황일 때, Boostrap ClassLoader가 모르면 Platform ClassLoader가 찾아보고 여기서도 찾을 수 없다면 Application ClassLoader가 시도합니다. 만약 여기서도 모른다면 `ClassNotfoundException`이 발생합니다.

3. Class Loader (Linking)
  - Verification: bytecode가 JVM 규약을 따르고 있는지 검증하는 단계입니다. 만약 검증을 통과하지 못하는 경우 `VerifyError`가 발생합니다.
  - Preparation: Static field에 초기값이 설정됩니다. integer의 경우 0, boolean의 경우 false 입니다.
  - Resolution: Symbolic 참조 경로가 실제 메모리 경로를 참조하도록 업데이트 됩니다. 그러나 이 과정은 Optional 합니다. 또한 Lazy resolution이 설정된 경우에는 Runtime 시점에 Resolution 과정이 발생할 수도 있습니다.

4. Class Loader (Initialization)
  - Static block이 실행됩니다.
  - Static field에 초기값이 할당됩니다.

5. Execution Engine
  - Interpreter
    - Interpreter에 의해 bytecode가 line by line 단위로 실행됩니다. 이 과정은 초기 실행은 빠를 수 있지만 전체 실행 속도는 느릴 수 있습니다.
    - Interpreter에 의해 실행되는 코드 정보는 Profiler에 의해 수집됩니다. 그리고 임계값을 넘어서는 수치로 실행되는 코드는 `hotspot` 코드로 판정되어 JIT Compiler에 의해 native code로 컴파일 됩니다.

  - JIT Compiler
    - Tiered Compilation
      - C1: Client Compiler. 빠른 컴파일 & 기본 최적화
      - C2: Server Compiler. 느린 컴파일 & 높은 최적화
    - Compiler의 종류에 따라서 임계값이 다르게 설정됩니다. C1 컴파일러는 Client compiler로서 임계값이 상대적으로 낮고 C2 컴파일러는 Server compiler로서 임계값이 상대적으로 높습니다. 이 임계값은 `XX:CompileThreshold` 설정을 통해 임의의 값으로 설정할 수 있습니다.
    - native code는 Interpreter가 아닌 CPU에 의해 바로 실행될 수 있는 코드로서 상대적으로 매우 빠른 실행 속도를 갖습니다.

## 📌 질문 2: Interpreter와 JIT Compiler의 협력
```
질문:
"Interpreter와 JIT Compiler는 실제로 어떻게 협력하나요? 구체적으로:
1) Profiler가 어떤 정보를 수집하나요?

단순히 "호출 횟수"만 세나요?
다른 정보도 수집하나요?

2) Hotspot이라고 판정되면 정확히 무슨 일이 일어나나요?

JIT 컴파일이 즉시 일어나나요?
컴파일되는 동안 코드는 계속 실행되나요?

3) Tiered Compilation에서 C1과 C2의 협력 과정은?

코드가 C1 → C2로 어떻게 넘어가나요?
C1으로 컴파일된 코드가 다시 C2로 재컴파일되나요?

4) 컴파일된 native code는 어디에 저장되나요?

메모리의 어느 영역에?
프로그램 종료 후에도 남아있나요?"
```

### 설명:

#### 1) Interpreter와 JIT Compiler는 실제로 어떻게 협력하나요?
Profiler가 수집하는 정보
  - 기본 카운터:
    - 메서드 진입 횟수 (Method Entry Count)
    - 루프 반복 횟수 (Back-edge Count) - 루프를 몇 번 도는지
  - 타입 정보:
    ```java
    Animal animal = ???;
    animal.speak();
    ```
    - "이 변수가 실제로는 90% Dog, 10% Cat이네!"
    - 이런 정보로 최적화 가능!
  - 브랜치 정보:
    ```java
    if (condition) {
      // 95% 여기로 옴
    } else {
      // 5%만 여기
    }
    ```
    - 어느 분기가 더 자주 실행되는지
  - 기타 런타임 정보
    - Null 체크 실패율
    - 예외 발생 빈도
    - 객체 할당 패턴

#### 2) Hotspot이라고 판정되면 정확히 무슨 일이 일어나나요?
  - Hotspot으로 판정되면 Compile 요청을 Queue에 넣습니다.
  - Interpreter는 실행을 멈추지 않고 계속 코드를 실행합니다.
    - 동시에 백그라운드에서 Compiler thread가 가동됩니다.
    - Compeiler thread는 컴파일을 수행합니다.
  - 컴파일이 완료된 이후에는 다음 호출부터는 Native Code가 실행됩니다.

#### 3) Tiered Compilation에서 C1과 C2의 협력 과정은?
  - 최초 Interpreter에 의해 코드가 실행되는 것을 `Level 0` 이라고 부르고 각 level 별 임계값을 넘어설 때마다 Level 1, Level 2, Level 3, 마지막 Level 4 단계로 진행
    - Level 0: Interprepter에 의해 코드 실행, Profiling 수집
    - Level 1: C1 컴파일러에 의해 컴파일된 코드가 실행, Profiling 수집하지 않음
    - Level 2: C1 컴파일러에 의해 컴파일된 코드가 실행, 최소한의 Profiling 정보만 수집
    - Level 3: C1 컴파일러에 의해 컴파일된 코드가 실행, 모든 Profiling 코드가 삽입되어 정보를 수집
    - Level 4: C2 컴파일러에 의해 컴파일된 코드가 실행
  - Level 1~3: 같은 C1 컴파일러에 의해 컴파일된 코드이지만 Profiling 코드가 어느 정도 삽입되어 있는지가 다름. 그래서 Level 1에 비해 Level 3 코드가 느림. 모든 Profiling 코드가 삽입되어 Profiling 정보를 수집하기 때문임
    - Level 1의 경우 Profiling 정보를 수집하지 않기 때문에 C2로 진행이 불가능
    - Level 2는 Deprecated 상태
  - Level 0에서 곧바로 Level 3를 거쳐 Level 4로 가거나 아니면 그냥 Level 3에서 멈추는 경우도 있음
  - 만약 필요다하면 Deoptimization 과정을 거칠 수도 있음

#### 4) 컴파일된 native code는 어디에 저장되나요?
  - C1, C2 컴파일러에 의해 컴파일된 Native Code는 JVM 메모리의 `Code Cache` 영역에 저장됨
  - Code Cache: 크기에 제한이 있으며 `-XX:ReservedCodeCacheSize=512m`와 같이 설정할 수 있음
  - 만약 Code Cache 영역이 가득 차면 더 이상의 JIT compile은 발생하지 못함
  - 프로그램이 종료하면 Code Cache 영역은 사라짐. 다만 AOT(Ahead Of Time) 컴파일 같은 기술의 경우는 미리 컴파일하여 native code를 저장해놓을 수 있음

## 참고 자료
  - https://www.youtube.com/watch?v=BeMi8K0AFAc
  - https://claude.ai/share/4fb18160-77b0-4264-a7eb-73391e92c697