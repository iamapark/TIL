# 🚀 Spring Boot 3.x 고급 기능 상세 학습 계획서

## 📋 학습 개요

### 목표
- Spring Boot 3.x의 최신 고급 기능 완전 마스터
- 실무에서 바로 적용 가능한 실전 능력 습득
- 고성능, 클라우드 네이티브 애플리케이션 개발 역량 확보

### 학습 기간
- **총 기간**: 12주 (3개월)
- **주간 학습 시간**: 15-20시간
- **실습 프로젝트**: 4개 (주요 기능별)

### 전제 조건
- Java 17+ 기본 지식
- Spring Framework 기본 이해
- Maven/Gradle 빌드 도구 경험
- Docker 기본 사용법

---

## 📚 Week 1-2: Spring Boot 3.x 기초 및 Java 21 통합

### 학습 목표
- Spring Boot 3.x 전체 아키텍처 이해
- Java 21의 새로운 기능과 Spring Boot 통합
- 개발 환경 설정 및 기본 프로젝트 구성

### 세부 학습 내용

#### 1.1 Spring Boot 3.x 개요 (3일)
- **Spring Boot 3.2/3.3/3.4/3.5 주요 변경사항**
  - Java 17 최소 요구사항
  - Jakarta EE 9+ 마이그레이션
  - Spring Framework 6.x 통합
  
- **새로운 의존성 관리**
  ```xml
  <parent>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-parent</artifactId>
      <version>3.4.3</version>
  </parent>
  ```

- **마이그레이션 가이드 학습**
  - javax.* → jakarta.* 패키지 변경
  - 호환성 체크리스트
  - 기존 프로젝트 업그레이드 전략

#### 1.2 Java 21 통합 (4일)
- **Records 활용**
  ```java
  public record UserDto(String name, String email, Integer age) {}
  
  @RestController
  public class UserController {
      @PostMapping("/users")
      public ResponseEntity<UserDto> createUser(@RequestBody UserDto user) {
          // Record를 사용한 간결한 데이터 전송
          return ResponseEntity.ok(user);
      }
  }
  ```

- **Sealed Classes 적용**
  ```java
  public sealed interface PaymentMethod 
      permits CreditCard, DebitCard, DigitalWallet {
  }
  
  public record CreditCard(String number) implements PaymentMethod {}
  public record DebitCard(String number) implements PaymentMethod {}
  public record DigitalWallet(String email) implements PaymentMethod {}
  ```

- **Switch Expression 패턴 매칭**
  ```java
  public String processPayment(PaymentMethod payment) {
      return switch (payment) {
          case CreditCard(var number) -> "Processing credit card: " + number;
          case DebitCard(var number) -> "Processing debit card: " + number;
          case DigitalWallet(var email) -> "Processing wallet: " + email;
      };
  }
  ```

#### 1.3 개발 환경 설정 (3일)
- **IDE 설정** (IntelliJ IDEA/VS Code)
  - Java 21 SDK 설정
  - Spring Boot 3.x 플러그인 설치
  - 코드 스타일 및 포맷팅 구성

- **빌드 도구 설정**
  ```xml
  <!-- Maven -->
  <properties>
      <java.version>21</java.version>
      <spring-boot.version>3.4.3</spring-boot.version>
  </properties>
  
  <plugin>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-maven-plugin</artifactId>
      <configuration>
          <image>
              <builder>paketobuildpacks/builder-jammy-java-tiny</builder>
          </image>
      </configuration>
  </plugin>
  ```

### 실습 프로젝트 1: 기본 API 서버
- **목표**: Spring Boot 3.x + Java 21 기본 CRUD API 구현
- **기능**: 
  - Records를 사용한 DTO 설계
  - RESTful API 엔드포인트
  - 기본 예외 처리
- **완성 기한**: 2주차 말

### 평가 기준
- [ ] Spring Boot 3.x 프로젝트 성공적 생성
- [ ] Java 21 Record/Sealed Classes 올바른 사용
- [ ] 기본 REST API 동작 확인
- [ ] 코드 품질 및 구조화

---

## 🧵 Week 3-4: Virtual Threads (Project Loom) 마스터

### 학습 목표
- Virtual Threads 개념 및 동작 원리 완전 이해
- Spring Boot에서 Virtual Threads 효과적 활용
- 기존 Thread Pool과 성능 비교 분석

### 세부 학습 내용

#### 3.1 Virtual Threads 이론 (3일)
- **기본 개념 학습**
  - Platform Threads vs Virtual Threads
  - Carrier Threads와 Virtual Thread 관계
  - Pinning 현상과 해결 방법

- **성능 특성 이해**
  ```java
  // Traditional Thread Pool
  @Bean
  public TaskExecutor taskExecutor() {
      ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
      executor.setCorePoolSize(10);
      executor.setMaxPoolSize(100);
      return executor;
  }
  
  // Virtual Threads
  @Bean
  public TaskExecutor virtualTaskExecutor() {
      return TaskExecutor.class.cast(
          Executors.newVirtualThreadPerTaskExecutor()
      );
  }
  ```

#### 3.2 Spring Boot Virtual Threads 설정 (3일)
- **자동 구성 활성화**
  ```properties
  # application.properties
  spring.threads.virtual.enabled=true
  ```

- **Tomcat Virtual Threads 설정**
  ```java
  @Bean
  public TomcatProtocolHandlerCustomizer<?> protocolHandlerVirtualThreadExecutorCustomizer() {
      return protocolHandler -> {
          protocolHandler.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
      };
  }
  ```

- **@Async 메서드와 Virtual Threads**
  ```java
  @Service
  public class EmailService {
      
      @Async
      public CompletableFuture<Void> sendEmailAsync(String to, String subject) {
          // I/O 집약적 작업 - Virtual Thread에서 실행
          emailClient.send(to, subject);
          return CompletableFuture.completedFuture(null);
      }
  }
  ```

#### 3.3 성능 최적화 및 모니터링 (4일)
- **JFR (Java Flight Recorder) 모니터링**
  ```bash
  java -XX:+FlightRecorder 
       -XX:StartFlightRecording=duration=60s,filename=virtual-threads.jfr 
       -jar app.jar
  ```

- **Virtual Threads 성능 측정**
  ```java
  @RestController
  public class PerformanceTestController {
      
      @GetMapping("/stress-test")
      public ResponseEntity<String> stressTest() {
          var startTime = System.currentTimeMillis();
          
          // 1000개의 동시 요청 시뮬레이션
          var futures = IntStream.range(0, 1000)
              .mapToObj(i -> CompletableFuture.supplyAsync(() -> {
                  return simulateIoOperation();
              }))
              .toList();
              
          CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
          
          var endTime = System.currentTimeMillis();
          return ResponseEntity.ok("Completed in: " + (endTime - startTime) + "ms");
      }
  }
  ```

- **모니터링 및 디버깅 도구**
  - JConsole Virtual Thread 모니터링
  - VisualVM Thread 분석
  - Custom Metrics 구현

### 실습 프로젝트 2: 고성능 동시성 API 서버
- **목표**: Virtual Threads를 활용한 고성능 API 서버 구현
- **기능**:
  - 수천 개의 동시 요청 처리
  - 외부 API 호출 최적화
  - 성능 모니터링 대시보드
- **성능 목표**: 
  - 1만 동시 요청 처리
  - 평균 응답 시간 100ms 이하
- **완성 기한**: 4주차 말

### 평가 기준
- [ ] Virtual Threads 올바른 설정 및 활성화
- [ ] 기존 Thread Pool 대비 성능 향상 확인
- [ ] 대용량 동시 요청 처리 성공
- [ ] 모니터링 지표 수집 및 분석

---

## 🏗️ Week 5-6: Native Image & GraalVM 최적화

### 학습 목표
- GraalVM Native Image 컴파일 프로세스 완전 이해
- Spring Boot AOT (Ahead-of-Time) 처리 마스터
- 프로덕션 환경 Native Image 배포 경험

### 세부 학습 내용

#### 5.1 GraalVM Native Image 기초 (3일)
- **GraalVM 설치 및 설정**
  ```bash
  # SDKMAN을 통한 설치
  sdk install java 22.3.r21-nik
  sdk use java 22.3.r21-nik
  
  # Native Image 설치 확인
  native-image --version
  ```

- **Spring Boot Native Image 빌드**
  ```xml
  <!-- Maven Native Profile -->
  <profiles>
      <profile>
          <id>native</id>
          <build>
              <plugins>
                  <plugin>
                      <groupId>org.graalvm.buildtools</groupId>
                      <artifactId>native-maven-plugin</artifactId>
                  </plugin>
              </plugins>
          </build>
      </profile>
  </profiles>
  ```

- **빌드 명령어 실습**
  ```bash
  # 직접 컴파일
  ./mvnw -Pnative native:compile
  
  # Docker 이미지 빌드
  ./mvnw spring-boot:build-image -Pnative
  ```

#### 5.2 AOT (Ahead-of-Time) 처리 심화 (4일)
- **Spring AOT 처리 이해**
  ```java
  @Configuration
  public class AotConfiguration {
      
      @Bean
      @RegisterReflectionForBinding(UserDto.class)
      public UserService userService() {
          return new UserService();
      }
  }
  ```

- **RuntimeHints API 활용**
  ```java
  @Component
  public class CustomRuntimeHintsRegistrar implements RuntimeHintsRegistrar {
      
      @Override
      public void registerHints(RuntimeHints hints, ClassLoader classLoader) {
          // 리플렉션 힌트 등록
          hints.reflection()
              .registerType(TypeReference.of(MyCustomClass.class))
              .withMembers(MemberCategory.INVOKE_DECLARED_METHODS);
              
          // 리소스 힌트 등록
          hints.resources()
              .registerPattern("static/**")
              .registerPattern("templates/**");
              
          // JNI 힌트 등록 (필요시)
          hints.jni()
              .registerType(TypeReference.of(NativeLibraryClass.class));
      }
  }
  ```

- **빌드 시간 최적화**
  ```properties
  # native-image 빌드 옵션
  spring.native.build-args=-H:+ReportExceptionStackTraces,-H:+PrintClassInitialization
  ```

#### 5.3 Native Image 문제 해결 및 최적화 (3일)
- **일반적인 문제들과 해결책**
  ```java
  // 동적 클래스 로딩 문제 해결
  @ImportRuntimeHints(MyRuntimeHints.class)
  @SpringBootApplication
  public class Application {
      public static void main(String[] args) {
          SpringApplication.run(Application.class, args);
      }
  }
  
  class MyRuntimeHints implements RuntimeHintsRegistrar {
      @Override
      public void registerHints(RuntimeHints hints, ClassLoader classLoader) {
          // 동적으로 로드되는 클래스들 명시적 등록
          hints.reflection().registerType(DynamicallyLoadedClass.class,
              MemberCategory.INVOKE_DECLARED_CONSTRUCTORS,
              MemberCategory.INVOKE_DECLARED_METHODS);
      }
  }
  ```

- **성능 벤치마크**
  ```bash
  # 시작 시간 측정
  time ./target/native-app
  
  # 메모리 사용량 측정
  /usr/bin/time -v ./target/native-app
  ```

- **크기 최적화**
  ```properties
  # Dead code elimination 활성화
  spring.native.remove-unused-autoconfig=true
  spring.native.remove-yaml-support=true
  spring.native.remove-jmx-support=true
  ```

### 실습 프로젝트 3: Native Cloud Function
- **목표**: GraalVM Native Image 기반 서버리스 함수 구현
- **기능**:
  - AWS Lambda / Google Cloud Function 호환
  - 빠른 콜드 스타트 (200ms 이하)
  - 최소 메모리 사용량 (64MB 이하)
- **성과 지표**:
  - 네이티브 이미지 크기: 100MB 이하
  - 시작 시간: 50ms 이하
  - 메모리 사용량: 32MB 이하
- **완성 기한**: 6주차 말

### 평가 기준
- [ ] GraalVM Native Image 성공적 컴파일
- [ ] AOT 힌트 올바른 구성
- [ ] 성능 목표 달성 (시작 시간, 메모리)
- [ ] 프로덕션 배포 가능한 이미지 생성

---

## 📊 Week 7-8: Observability & Monitoring 심화

### 학습 목표
- Spring Boot 3.x Observability 생태계 완전 이해
- Micrometer + OpenTelemetry 통합 구성
- 프로덕션 레벨 모니터링 시스템 구축

### 세부 학습 내용

#### 7.1 Micrometer Observation API (3일)
- **핵심 개념 학습**
  ```java
  @Service
  public class UserService {
      
      private final ObservationRegistry observationRegistry;
      
      public User findUser(String id) {
          return Observation.createNotStarted("user.find", observationRegistry)
              .contextualName("find-user-by-id")
              .tag("user.id", id)
              .observe(() -> {
                  // 실제 비즈니스 로직
                  return userRepository.findById(id);
              });
      }
  }
  ```

- **Custom Metrics 구현**
  ```java
  @Component
  public class CustomMetrics {
      
      private final Counter userCreationCounter;
      private final Timer userSearchTimer;
      private final Gauge activeUsersGauge;
      
      public CustomMetrics(MeterRegistry meterRegistry) {
          this.userCreationCounter = Counter.builder("users.created.total")
              .description("Total number of users created")
              .register(meterRegistry);
              
          this.userSearchTimer = Timer.builder("users.search.duration")
              .description("User search operation duration")
              .register(meterRegistry);
              
          this.activeUsersGauge = Gauge.builder("users.active.current")
              .description("Current number of active users")
              .register(meterRegistry, this, CustomMetrics::getActiveUserCount);
      }
  }
  ```

#### 7.2 OpenTelemetry 통합 (4일)
- **의존성 및 설정**
  ```xml
  <dependency>
      <groupId>io.micrometer</groupId>
      <artifactId>micrometer-tracing-bridge-otel</artifactId>
  </dependency>
  <dependency>
      <groupId>io.opentelemetry</groupId>
      <artifactId>opentelemetry-exporter-otlp</artifactId>
  </dependency>
  ```

  ```properties
  # OpenTelemetry 설정
  management.tracing.sampling.probability=1.0
  management.otlp.tracing.endpoint=http://localhost:4318/v1/traces
  management.otlp.metrics.endpoint=http://localhost:4318/v1/metrics
  management.otlp.logs.endpoint=http://localhost:4318/v1/logs
  ```

- **분산 추적 구현**
  ```java
  @RestController
  public class OrderController {
      
      @Autowired
      private PaymentService paymentService;
      
      @PostMapping("/orders")
      @NewSpan("order.creation")
      public ResponseEntity<Order> createOrder(@RequestBody OrderRequest request) {
          // 자동으로 트레이스 생성
          var order = orderService.createOrder(request);
          
          // 수동 스팬 생성
          Span.current().addEvent("order.validation.completed");
          
          // 하위 서비스 호출 (자동 추적)
          paymentService.processPayment(order.getPaymentInfo());
          
          return ResponseEntity.ok(order);
      }
  }
  ```

- **Baggage와 Context Propagation**
  ```java
  @Component
  public class RequestContextService {
      
      public void setUserContext(String userId) {
          // Baggage를 통한 컨텍스트 전파
          Baggage.current()
              .toBuilder()
              .put("user.id", userId)
              .build()
              .makeCurrent();
      }
      
      @EventListener
      public void handleUserAction(UserActionEvent event) {
          // 현재 컨텍스트에서 Baggage 읽기
          String userId = Baggage.current().get("user.id");
          // 로그에 사용자 정보 포함
          MDC.put("userId", userId);
      }
  }
  ```

#### 7.3 고급 모니터링 설정 (3일)
- **Actuator 엔드포인트 커스터마이징**
  ```java
  @Component
  public class CustomHealthIndicator implements HealthIndicator {
      
      @Override
      public Health health() {
          // 비즈니스 로직 기반 헬스 체크
          boolean databaseConnected = checkDatabaseConnection();
          boolean externalServiceAvailable = checkExternalService();
          
          if (databaseConnected && externalServiceAvailable) {
              return Health.up()
                  .withDetail("database", "Connected")
                  .withDetail("external-service", "Available")
                  .build();
          }
          
          return Health.down()
              .withDetail("database", databaseConnected ? "Connected" : "Down")
              .withDetail("external-service", externalServiceAvailable ? "Available" : "Down")
              .build();
      }
  }
  ```

- **OTLP 로깅 설정**
  ```xml
  <!-- logback-spring.xml -->
  <configuration>
      <appender name="OTLP" class="io.opentelemetry.instrumentation.logback.v1_0.OpenTelemetryAppender">
          <endpoint>http://localhost:4318/v1/logs</endpoint>
      </appender>
      
      <root level="INFO">
          <appender-ref ref="OTLP"/>
          <appender-ref ref="CONSOLE"/>
      </root>
  </configuration>
  ```

### 실습 프로젝트 4: 완전한 Observability 스택
- **목표**: 마이크로서비스 환경에서 완전한 관찰 가능성 구현
- **구성 요소**:
  - Spring Boot 애플리케이션 3개 (User, Order, Payment 서비스)
  - OpenTelemetry Collector
  - Jaeger (분산 추적)
  - Prometheus (메트릭)
  - Grafana (시각화)
  - ELK Stack (로깅)
- **구현 기능**:
  - 서비스 간 분산 추적
  - 비즈니스 메트릭 대시보드
  - 실시간 알림 시스템
- **완성 기한**: 8주차 말

### 평가 기준
- [ ] Micrometer Observation API 올바른 사용
- [ ] OpenTelemetry 완전한 통합
- [ ] 분산 추적 end-to-end 구현
- [ ] 프로덕션 레벨 모니터링 대시보드

---

## 🐳 Week 9-10: Docker Compose & Testcontainers 통합

### 학습 목표
- Spring Boot 3.1+ Docker Compose 지원 완전 활용
- Testcontainers 고급 패턴 및 최적화
- 로컬 개발 환경과 테스트 환경 통합

### 세부 학습 내용

#### 9.1 Docker Compose 통합 (3일)
- **spring-boot-docker-compose 설정**
  ```xml
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-docker-compose</artifactId>
      <scope>runtime</scope>
  </dependency>
  ```

- **compose.yaml 구성**
  ```yaml
  services:
    postgres:
      image: 'postgres:15'
      environment:
        POSTGRES_DB: myapp
        POSTGRES_USER: user
        POSTGRES_PASSWORD: password
      ports:
        - '5432:5432'
      healthcheck:
        test: ["CMD-SHELL", "pg_isready -U user -d myapp"]
        interval: 10s
        timeout: 5s
        retries: 5
        
    redis:
      image: 'redis:7'
      ports:
        - '6379:6379'
      healthcheck:
        test: ["CMD", "redis-cli", "ping"]
        interval: 10s
        timeout: 3s
        retries: 5
        
    elasticsearch:
      image: 'elasticsearch:8.11.0'
      environment:
        - discovery.type=single-node
        - xpack.security.enabled=false
      ports:
        - '9200:9200'
  ```

- **Service Connection 자동 구성**
  ```properties
  # 자동으로 설정되는 속성들
  spring.datasource.url=jdbc:postgresql://localhost:5432/myapp
  spring.datasource.username=user
  spring.datasource.password=password
  
  spring.data.redis.host=localhost
  spring.data.redis.port=6379
  
  spring.elasticsearch.uris=http://localhost:9200
  ```

#### 9.2 Testcontainers 고급 활용 (4일)
- **@ServiceConnection 사용**
  ```java
  @SpringBootTest
  @Testcontainers
  class IntegrationTest {
      
      @Container
      @ServiceConnection
      static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
          .withDatabaseName("testdb")
          .withUsername("test")
          .withPassword("test");
          
      @Container
      @ServiceConnection
      static RedisContainer redis = new RedisContainer("redis:7");
      
      @Test
      void contextLoads() {
          // 테스트 로직
      }
  }
  ```

- **로컬 개발 환경 설정**
  ```java
  @TestConfiguration(proxyBeanMethods = false)
  public class LocalDevelopmentConfiguration {
      
      @Bean
      @ServiceConnection
      PostgreSQLContainer<?> postgresContainer() {
          return new PostgreSQLContainer<>("postgres:15")
              .withDatabaseName("devdb")
              .withUsername("dev")
              .withPassword("dev")
              .withReuse(true); // 컨테이너 재사용
      }
      
      @Bean
      @ServiceConnection
      RedisContainer redisContainer() {
          return new RedisContainer("redis:7")
              .withReuse(true);
      }
  }
  ```

- **복합 Testcontainers 설정**
  ```java
  @Testcontainers
  class ComplexIntegrationTest {
      
      @Container
      static Network network = Network.newNetwork();
      
      @Container
      @ServiceConnection
      static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
          .withNetwork(network)
          .withNetworkAliases("postgres");
          
      @Container
      static GenericContainer<?> app = new GenericContainer<>("my-app:latest")
          .withNetwork(network)
          .withEnv("DATABASE_URL", "jdbc:postgresql://postgres:5432/testdb")
          .dependsOn(postgres);
  }
  ```

#### 9.3 테스트 최적화 및 성능 (3일)
- **테스트 슬라이스와 Testcontainers**
  ```java
  @DataJpaTest
  @Testcontainers
  class UserRepositoryTest {
      
      @Container
      @ServiceConnection
      static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");
      
      @Autowired
      private TestEntityManager entityManager;
      
      @Autowired
      private UserRepository userRepository;
      
      @Test
      void findByEmail_ShouldReturnUser() {
          // JPA 관련 테스트만 실행
      }
  }
  ```

- **테스트 컨테이너 재사용**
  ```properties
  # testcontainers.properties
  testcontainers.reuse.enable=true
  ```

- **병렬 테스트 실행**
  ```java
  @Execution(ExecutionMode.CONCURRENT)
  @SpringBootTest
  class ParallelIntegrationTest {
      
      @Container
      @ServiceConnection
      static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
          .withDatabaseName("parallel_test_" + UUID.randomUUID());
  }
  ```

### 실습 미니 프로젝트들

#### 프로젝트 4-1: 마이크로서비스 로컬 개발 환경
- **목표**: Docker Compose를 활용한 완전한 로컬 개발 환경
- **구성**: API Gateway + 3개 마이크로서비스 + 공유 데이터베이스

#### 프로젝트 4-2: 통합 테스트 자동화
- **목표**: Testcontainers 기반 CI/CD 통합 테스트
- **특징**: GitHub Actions와 연동된 자동화된 테스트

### 평가 기준
- [ ] Docker Compose 완전한 통합
- [ ] Testcontainers 올바른 사용
- [ ] 로컬 개발 환경 자동화
- [ ] 테스트 성능 최적화

---

## 🎯 Week 11-12: 종합 프로젝트 & 성능 최적화

### 학습 목표
- 모든 학습 내용을 통합한 실전 프로젝트 완성
- 성능 최적화 및 프로덕션 준비
- 포트폴리오 완성 및 기술 문서화

### 최종 프로젝트: 고성능 E-commerce 백엔드

#### 프로젝트 요구사항
- **아키텍처**: 마이크로서비스 (4개 서비스)
  - User Service (사용자 관리)
  - Product Service (상품 관리)  
  - Order Service (주문 처리)
  - Payment Service (결제 처리)

- **기술 스택**:
  - Spring Boot 3.4.3 + Java 21
  - Virtual Threads 활성화
  - GraalVM Native Image 지원
  - Complete Observability (Metrics, Tracing, Logging)
  - Docker Compose 개발 환경
  - Testcontainers 테스트 자동화

#### 성능 목표
- **시작 시간**: 3초 이하 (JVM), 500ms 이하 (Native)
- **처리량**: 1000 TPS 이상
- **응답 시간**: 95%ile 200ms 이하
- **메모리 사용량**: 512MB 이하 (JVM), 128MB 이하 (Native)

#### 구현 기능

##### Week 11: 핵심 기능 구현
1. **User Service**
   ```java
   @RestController
   @RequestMapping("/api/users")
   public class UserController {
       
       @PostMapping
       @Timed("users.create")
       public ResponseEntity<UserDto> createUser(@RequestBody @Valid CreateUserRequest request) {
           return Observation.createNotStarted("user.create", observationRegistry)
               .contextualName("create-user")
               .observe(() -> {
                   var user = userService.createUser(request);
                   return ResponseEntity.ok(userMapper.toDto(user));
               });
       }
   }
   ```

2. **Product Service with Virtual Threads**
   ```java
   @Service
   public class ProductService {
       
       @Async
       @Retryable(value = {Exception.class}, maxAttempts = 3)
       public CompletableFuture<ProductDto> enrichProductData(Product product) {
           // Virtual Thread에서 실행되는 외부 API 호출
           return CompletableFuture.supplyAsync(() -> {
               var enrichedData = externalApiClient.getProductData(product.getId());
               return productMapper.toEnrichedDto(product, enrichedData);
           });
       }
   }
   ```

3. **Order Service with Saga Pattern**
   ```java
   @Component
   public class OrderSagaOrchestrator {
       
       @SagaOrchestrationStart
       public void processOrder(OrderCreatedEvent event) {
           sagaManager.choreography()
               .step("reserve-inventory")
                   .invokeParticipant("product-service")
                   .compensate("release-inventory")
               .step("process-payment")
                   .invokeParticipant("payment-service")
                   .compensate("refund-payment")
               .step("create-shipment")
                   .invokeParticipant("shipping-service")
                   .compensate("cancel-shipment")
               .execute(event);
       }
   }
   ```

##### Week 12: 최적화 및 마무리
1. **성능 프로파일링**
   ```bash
   # JFR 프로파일링
   java -XX:+FlightRecorder \
        -XX:StartFlightRecording=duration=300s,filename=perf-analysis.jfr \
        -jar app.jar
        
   # Native Image 빌드 최적화
   ./mvnw -Pnative native:compile \
       -Dspring-boot.build-image.builder=paketobuildpacks/builder-jammy-java-tiny
   ```

2. **부하 테스트**
   ```bash
   # K6 부하 테스트
   k6 run --vus 100 --duration 5m load-test.js
   ```

3. **모니터링 대시보드 완성**
   - Grafana 대시보드 구성
   - 알림 규칙 설정
   - SLI/SLO 정의

### 최종 평가 기준
- [ ] **기능 완성도** (30%)
  - 모든 API 엔드포인트 동작
  - 비즈니스 로직 완전 구현
  - 에러 처리 및 예외 상황 대응

- [ ] **성능 목표 달성** (25%)
  - TPS, 응답시간, 메모리 사용량 목표 달성
  - Virtual Threads 효과적 활용
  - Native Image 성능 최적화

- [ ] **코드 품질** (20%)
  - 클린 코드 원칙 준수
  - SOLID 설계 원칙 적용
  - 적절한 디자인 패턴 사용

- [ ] **테스트 커버리지** (15%)
  - 단위 테스트 80% 이상
  - 통합 테스트 완전 구현
  - Testcontainers 올바른 활용

- [ ] **문서화** (10%)
  - API 문서 (OpenAPI/Swagger)
  - 아키텍처 다이어그램
  - 성능 벤치마크 결과

---

## 📖 참고 자료 및 리소스

### 공식 문서
- [Spring Boot 3.x Reference](https://docs.spring.io/spring-boot/docs/current/reference/html/)
- [GraalVM Native Image Guide](https://www.graalvm.org/latest/reference-manual/native-image/)
- [OpenTelemetry Java Documentation](https://opentelemetry.io/docs/instrumentation/java/)
- [Testcontainers Documentation](https://testcontainers.com/)

### 필수 도서
- "Spring Boot: Up and Running" - Mark Heckler
- "Optimizing Java" - Benjamin J. Evans
- "Java Performance: In-Depth Advice" - Scott Oaks

### 온라인 강의
- Spring Academy - Spring Boot 3.x 과정
- Baeldung - Spring Boot 고급 기능
- InfoQ - Virtual Threads 심화 과정

### 유용한 도구
- **프로파일링**: JProfiler, YourKit, async-profiler
- **부하 테스트**: K6, JMeter, Gatling
- **모니터링**: Micrometer, OpenTelemetry, Jaeger
- **개발 도구**: IntelliJ IDEA Ultimate, VS Code

---

## 🎯 학습 성공 팁

### 실습 중심 학습
- 매일 최소 2시간 실습 코딩
- 작은 예제부터 시작해서 점진적 확장
- 공식 문서를 먼저 참조하는 습관

### 커뮤니티 활용
- Spring 공식 블로그 정기 구독
- Stack Overflow 질문 및 답변 참여
- GitHub 오픈소스 프로젝트 기여

### 성과 측정
- 주간 학습 일지 작성
- 실습 프로젝트 GitHub 공개
- 기술 블로그 포스팅 (월 2회)

### 지속적 개선
- 코드 리뷰 요청 및 피드백 반영
- 성능 벤치마크 정기 실행
- 최신 버전 업데이트 추적

---

*이 학습 계획서는 Spring Boot 3.x의 고급 기능을 체계적으로 마스터하기 위한 실전 중심의 로드맵입니다. 각 주차별 목표를 달성하며 점진적으로 전문성을 키워나가시기 바랍니다.*

**최종 업데이트**: 2025년 1월