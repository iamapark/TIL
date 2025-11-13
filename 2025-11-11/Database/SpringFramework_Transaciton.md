# 스프링 트랜잭션 AOP 동작 흐름

## 이 주제를 공부하는 이유
  - 매일메일의 공부 주제: `스프링 트랜잭션 AOP 동작 흐름에 대해서 설명해주세요.`

## 배운 내용

### Part 1: 데이터베이스 트랜잭션 기본 개념

#### 트랜잭션이란?

한문장 정의: 

```
트랜잭션은 데이터베이스에서 여러 작업을 하나의 논리적 단위로 묶어서 "전부 성공" 또는 "전부 실패"를 보장하는 메커니즘입니다.
```

#### 💡 트랜잭션이 필요한 이유
실생활 예시: 은행 계좌 이체

```
내 계좌: 10,000원
친구 계좌: 5,000원

내가 친구에게 3,000원 이체하는 과정:
1. 내 계좌에서 3,000원 추감 -> 7,000원
2. 친구 계좌에 3,000원 추가 -> 8,000원
```

만약 트랜잭션이 없다면? 😱
```
1. 내 계좌 -3,000원 성공 ✅
2. [갑자기 서버 다운! 🔥]
3. 친구 계좌 +3,000원 실패 ❌

결과:
 - 내 계좌: 7,000원 (3,000원 사라짐!)
 - 친구 계좌: 5,000원 (받지 못함!)
 - 3,000원이 증발!
```

트랜잭션이 있다면: ✅
```
트랜잭션 시작
├─ 1. 내 계좌 -3,000원
├─ 2. 친구 계좌 +3,000원
└─ [서버 다운 발생!]
   → 트랜잭션 롤백
   → 모든 작업 취소
   → 원래 상태로 복구

결과:
  - 내 계좌: 10,000원 (원상복구)
  - 친구 계좌: 5,000원 (원상복구)
  - 돈은 안전!
```

#### 🔑 트랜잭션의 4가지 특성 (ACID)


| 특성 | 영어 | 의미 | 예시 |
|--|--|--|--|
| 원자성 | Atomicity | 전부 성공 or 전부 실패 | 이체 중 실패 시 모든 작업 취소 |
| 일관성 | Consistency | 규칙을 항상 지킴 | 총 금액은 항상 일정 (돈이 생기거나 사라지지 않음) |
| 격리성 | Isolation | 동시 실행되어도 서로 영향 없음 | A와 B가 동시에 이체해도 충돌 없음 |
| 지속성 | Durability | 완료된 결과는 영구 저장 | 커밋 후 서버 다운되어도 결과 유지 |

#### 🔄 트랜잭션 생명 주기
```
1. BEGIN (시작)
     ↓
2. 여러 SQL 실행
   ├─ INSERT
   ├─ UPDATE
   ├─ DELETE
     ↓
3. 성공 여부 판단
   ├─ 성공 → COMMIT (확정) ✅
   └─ 실패 → ROLLBACK (취소) ❌
```

#### Java 코드로 보는 트랜잭션 (JDBC 예시):
```java
Connection conn = dataSource.getConnection();

try {
  // 1. 트랜잭션 시작
  conn.setAutoCommit(false);

  // 2. 비즈니스 로직 실행
  stmt.executeUpdate("UPDATE account SET balance = balance - 3000 WHERE id = 1");
  stmt.executeUpdate("UPDATE account set balance = balance + 3000 WHERE id = 2");

  // 3. 성공 시 커밋
  conn.commit();
} catch (Exception e) {
  // 4. 실패 시 롤백
  conn.rollback();
} finally {
  conn.setAutoCommit(true);
  conn.close();
}
```

#### ⚠️ 트랜잭션 없이 코드를 작성하면?
```java
// ❌ 나쁜 예: 트랜잭션 없음
public void transfer(Long fromId, Long toId, int amount) {
    // 각 작업이 독립적으로 자동 커밋
    accountRepository.decrease(fromId, amount); // Auto-commit ✅
    
    // 여기서 예외 발생하면?
    throw new RuntimeException("오류!");
    
    accountRepository.increase(toId, amount);   // 실행 안됨! ❌
    // 첫 번째 작업은 이미 커밋되어 롤백 불가능!
}
```

#### 📊 트랜잭션의 문제점
순수 JDBC 트랜잭션 코드의 문제:
  1. 코드 중복: 모든 메서드에 try-catch-finally
  2. 비즈니스 로직 흐려짐: 트랜잭션 코드 때문에 핵심 로직 파악 어려움
  3. 실수 가능성: commit/rollback 빼먹기 쉬움
  4. 테스트 어려움: 트랜잭션 코드까지 테스트해야 함

```java
// 😰 트랜잭션 코드가 비즈니스 로직을 압도
public void businessLogic() {
    Connection conn = null;
    try {
        conn = dataSource.getConnection();
        conn.setAutoCommit(false);
        
        // 실제 비즈니스 로직은 단 2줄
        orderService.createOrder();
        paymentService.processPayment();
        
        conn.commit();
    } catch (Exception e) {
        if (conn != null) {
            try {
                conn.rollback();
            } catch (SQLException ex) {
                // 롤백도 실패할 수 있음!
            }
        }
    } finally {
        if (conn != null) {
            try {
                conn.close();
            } catch (SQLException e) {
                // close도 예외 처리!
            }
        }
    }
}
```

### Part 2: 프록시 패턴 이해

#### 프록시 패턴이란?
** 한 문장 정의:**
프록시는 **실제 객체를 대신하는 대리인**으로, 실제 객체 앞에서 **추가 작업을 수행** 할 수 있는 디자인패턴 입니다.

#### 왜 프록시가 필요한가?

**실생활 예시: 비서(Proxy) vs 사장님(Real Object)**
```
고객이 사장님께 미팅 요청
         ↓
비서가 먼저 받음 (Proxy)
         ↓
   ├─ 일정 확인
   ├─ 회의실 예약
   ├─ 자료 준비
   └─ 사장님께 전달 (Real Object)
        ↓
     미팅 진행
        ↓
    비서가 마무리 (Proxy)
    ├─ 회의록 작성
    └─ 후속 조치
```
핵심: 사장님(실제 객체)은 본업에만 집중하고, 부가적인 작업은 비서(프록시)가 처리!

#### 🔑 프록시 패턴 구조
```java
// 1. 인터페이스 (공통 계약)
interface Service {
  void execute();
}

// 2. 실제 객체 (Real Object)
class RealService implements Service {
  @Override
  public void execute() {
    System.out.println("핵심 비즈니스 로직 실행!");
  }
}

// 3. 프록시 객체 (Proxy)
class ServiceProxy implements Service {
  private RealService realService;

  public ServiceProxy(RealService realService) {
    this.realService = realService;
  }

  @Override
  public void execute() {
    // 실제 메소드 호출 "전"에 추가 작업
    System.out.println("[프록시] 사전 처리: 로깅, 보안 체크...");
    
    // 실제 객체의 메서드 호출
    this.realService.execute();

    // 실제 메소드 호출 "후"에 추가 작업
    System.out.println("[프록시] 사후 처리: 리소스 정리...");
  }
}

// 4. 클라이언트 사용
Service service = new ServiceProxy(new RealService());
service.execute();
```

#### 🎨 프록시의 핵심 포인트
```
클라이언트
    ↓
   Proxy (대리인)
    ├─ 사전 처리
    ├─ Real Object 호출
    └─ 사후 처리
```
중요한 특징:
  1. 같은 인터페이스: 프록시와 실체 객체가 같은 인터페이스 구현
  2. 투명성: 클라이언트는 프록시인지 실제 객체인지 모름
  3. 제어 가능: 실제 객체 호출 전/후에 로직 추가 가능

#### 프록시 패턴 사용 사례
| 종류 | 목적 | 예시 |
|-|-|-|
|보호 프록시|접근 제어|권한 체크 후 실제 객체 호출|
|가상 프록시|지연 로딩|실제 필요할 때만 객체 생성|
|로깅 프록시|로그 기록|메서드 호출 전후 로깅|
|트랜잭션 프록시|트랜잭션 관리|⭐ 여기서 배울 것!|

#### 🎯 스프링 트랜잭션과의 연결
스프링이 프록시 트랜잭션을 관리하는 방식:
```java
// 개발자가 작성한 서비스 (Real Object)
@Service
class OrderService {
  @Transactional // 이 어노테이션이 마법
  public void createOrder() {
    // 비즈니스 로직만 집중
    orderRepository.save(order);
    paymentRepository.save(payment);
  }
}

// 스프링이 자동으로 생성하는 프록시
class OrderServiceProxy extends OrderService {
  private TransactionManager txManager;

  @Override
  public void createOrder() {
    // 1. 트랜잭션 시작
    txManager.begin();

    try {
      // 2. 실제 메서드 호출
      super.createOrder();

      // 3. 트랜잭션 커밋
      txManager.commit();
    } catch (Exception e) {
      // 4. 예외 시 롤백
      txManager.rollback();
      throw e;
    }
  }
}
```
개발자가 `@Transactional` 어노테이션만 붙이면, 스프링이 자동으로 프록시를 만들어서 트랜잭션 처리를 대신해줍니다!

### Part 3: AOP 개념 복습

#### AOP란?
한 문장 정의: `AOP(Aspect-Oriented Programming)는 공통 관심사를 핵심 비즈니스 로직에서 분리`하여 코드 중복을 제거하는 프로그래밍 패러다임입니다.

#### 왜 AOP가 필요한가?
문제 상황: 모든 메서드에 로깅과 트랜잭션 추가
```java
// ❌ AOP 없이 - 코드 중복 지옥
public void method1() {
    log.info("method1 시작");           // 로깅 (공통)
    transactionManager.begin();         // 트랜잭션 (공통)
    try {
        // 핵심 비즈니스 로직
        businessLogic1();
        transactionManager.commit();    // 트랜잭션 (공통)
        log.info("method1 종료");       // 로깅 (공통)
    } catch (Exception e) {
        transactionManager.rollback();  // 트랜잭션 (공통)
        log.error("method1 실패", e);   // 로깅 (공통)
    }
}

public void method2() {
    log.info("method2 시작");           // 똑같은 코드 반복!
    transactionManager.begin();
    try {
        businessLogic2();
        transactionManager.commit();
        log.info("method2 종료");
    } catch (Exception e) {
        transactionManager.rollback();
        log.error("method2 실패", e);
    }
}

// method3, method4... 계속 반복! 😱
```

AOP로 해결:
```java
// ✅ AOP 사용 - 깔끔한 비즈니스 로직
@Transactional  // 트랜잭션은 AOP가 처리
@Logging        // 로깅도 AOP가 처리
public void method1() {
    // 핵심 비즈니스 로직만!
    businessLogic1();
}

@Transactional
@Logging
public void method2() {
    businessLogic2();
}
```

#### AOP 핵심 용어
| 용어 | 의미 | 스프링 트랜잭션 예시 |
|-|-|-|
|**Aspect**|공통 관심사 모듈|트랜잭션 관리|
|**Join Point**|적용 가능한 지점|메서드 실행 시점|
|**Advice**|실제 부가 기능 코드|begin/commit/rollback|
|**Pointcut**|Advice를 적용할 위치|`@Transactional`이 붙은 메서드|
|**Weaving**|Aspect를 적용하는 과정|프록시 생성|

#### AOP 동작 방식
```
개발자가 작성한 코드:
┌──────────────────┐
│  @Transactional  │
│  public void     │
│  createOrder() { │
│    비즈니스 로직   │
│  }               │
└──────────────────┘
         ↓
   AOP가 자동으로 변환
         ↓
실제 실행되는 코드:
┌──────────────────────────┐
│ 트랜잭션 시작              │ ← Advice (Before)
│   ↓                      │
│ 비즈니스 로직 실행          │ ← Join Point
│   ↓                      │
│ 성공 시 커밋 / 실패 시 롤백 │ ← Advice (After)
└──────────────────────────┘
```

#### 프록시 패턴 + AOP = 스프링 트랜잭션
3단계 연결:
  1. **트랜잭션 관리 = 공통 관심사** (Aspect)
    - 모든 서비스 메서드가 필요함
    - 항상 같은 패턴: begin -> 로직 -> commit/rollback
  2. **프록시로 구현** (Weaving)
    - 실제 객체를 감싸는 프록시 생성
    - 프록시가 트랜잭션 처리
  3. **@Transactional = 적용 위치 지정** (Pointcut)
    - 어떤 메서드에 트랜잭션을 적용할지 표시

**시각적으로 보면:**
```
@Transactional이 붙은 메서드
        ↓
    AOP가 감지
        ↓
   프록시 생성
        ↓
프록시가 트랜잭션 관리
   ├─ begin()
   ├─ 실제 메서드 호출
   └─ commit() / rollback()
```

#### 스프링 AOP의 특징
1. 런타임 프록시 기반
```java
// 컴파일 타임에는 원본 코드 그대로
@Transactional
public void createOrder() {
    orderRepository.save(order);
}

// 런타임에 프록시가 생성되어 실행
// 개발자는 프록시를 직접 작성할 필요 없음!
```

2. 메서드 실행 시점만 지원
```java
// ✅ 지원: 메서드 호출 전후
@Transactional
public void method() { }

// ❌ 미지원: 필드 접근, 생성자 호출 등
```

3. 스프링 빈에만 적용
```java
// ✅ 적용됨: 스프링이 관리하는 빈
@Service
class OrderService {
    @Transactional
    public void createOrder() { }
}

// ❌ 적용 안됨: new로 생성한 객체
OrderService service = new OrderService();
service.createOrder(); // 트랜잭션 없음!
```

#### AOP 없이 vs AOP 사용
코드 비교:
```java
// ❌ AOP 없이 (100줄)
public class OrderService {
    public void createOrder() {
        // 트랜잭션 코드: 15줄
        Connection conn = null;
        try {
            conn = dataSource.getConnection();
            conn.setAutoCommit(false);
            
            // 비즈니스 로직: 3줄
            orderRepository.save(order);
            paymentRepository.save(payment);
            emailService.send(email);
            
            conn.commit();
        } catch (Exception e) {
            if (conn != null) conn.rollback();
            throw e;
        } finally {
            if (conn != null) conn.close();
        }
    }
    
    // 다른 메서드 5개도 똑같이 반복...
}

// ✅ AOP 사용 (10줄)
@Service
public class OrderService {
    
    @Transactional
    public void createOrder() {
        // 비즈니스 로직만: 3줄
        orderRepository.save(order);
        paymentRepository.save(payment);
        emailService.send(email);
    }
    
    // 다른 메서드들도 깔끔!
}
```
**효과:**
  - 코드 중복 제거: **90%** 감소
  - 가독성 향상: **핵심 로직**만 보임
  - 유지보수 개선: **트랜잭션 정책 변경 시 한 곳만 수정**


### 스프링 트랜잭션 핵심
```
현재 위치:
✅ 1. 기초 개념 다지기
-> 2. 스프링 트랜잭션 핵심 ⭐️
   3. 전체 동작 흐름
```

### Part 1: Declarative Transaction Management (선언적 트랜잭션 관리)
🎯 선언적 vs 프로그래밍 방식

스프링은 두 가지 트랜잭션 관리 방식을 제공합니다:
|방식|설명|코드 스타일|
|-|-|-|
|프로그래밍 방식|코드로 직접 트랜잭션 제어|`transactionManager.begin()`, `transactionManager.commit()`|
|선언적 방식|어노테이션으로 트랜잭션 선언|`@Transactional`|

💡 프로그래밍 방식의 문제점
```java
// ❌ 프로그래밍 방식 - 코드가 복잡함
@Service
public class OrderService {
  private final TransactionTemplate transactionTemplate;

  public void createOrder(Order order) {
    transactionTemplate.execute(status -> {
      try {
        // 비지니스 로직
        orderRepository.save(order);
        paymentRepository.save(payment);
        return null;
      } catch (Exception e) {
        status.setRollbackOnly();
        throw e;
      }
    });
  }
}
```
문제점:
  - 🔴 트랜잭션 코드와 비즈니스 로직이 섞임
  - 🔴 모든 메서드마다 반복 코드
  - 🔴 테스트하기 어려움
  - 🔴 가독성 저하

🌟 선언적 방식의 장점
```java
// ✅ 선언적 방식 - 깔끔!
@Service
public class OrderService {
  @Transactional
  public void createOrder(Order order) {
    // 비즈니스 로직만!!
  }
}
```
**장점:**
  - ✅ 비즈니스 로직과 트랜잭션 분리
  - ✅ 코드 중복 제거
  - ✅ 테스트 용이
  - ✅ 가독성 향상
  - ✅ 유지보수 쉬움

#### 🔑 선언적 트랜잭션의 핵심 구성 요소
```
Declarative Transaction Management
           │
           │--- 1. @Transactional (선언)
           │    "이 메서드는 트랜잭션이 필요해!"
           │
           │--- 2. AOP Proxy (실행)
           │    프록시가 트랜잭션 로직을 대신 실행
           │
           │--- 3. Transaction Manager (관리)
           │    실제 트랜잭션을 시작/커밋/롤백
           │
           │--- 4. Transaction Synchronization Manager (동기화)
                트랜잭션 정보를 ThreadLocal에 저장
```

#### 📊 동작 흐름 미리보기
```
1. 클라이언트가 @Transactional 메서드 호출
          ↓
2. AOP 프록시가 인터셉트
          ↓
3. Transaction Manager에게 "트랜잭션 시작" 요청
          ↓
4. Transaction Manager가 DB Connection 획득
          ↓
5. Transaction Synchronization Manager에 저장 (ThreadLocal)
          ↓
6. 실제 메서드 실행
          ↓
7. 성공 시: Transaction Manager가 commit
   실패 시: Transaction Manager가 rollback
          ↓
8. Connection 반환
```

### Part 2: @Transactional 동작 원리
🎯 @Transactional 이란?
```java
@Target({ElementType.TYPE, Element.METHOD})
@Retention(RententionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Transactional {
  // 트랜잭션 속성들...
}
```
한 문장 정의: `@Transactional`은 **이 메서드를 트랜잭션 안에서 실행하라**고 스프링에게 알려주는 마커(marker) 어노테이션입니다.

#### 🔑 @Transactional의 주요 속성
```java
@Service
public class OrderService {
  @Transactional(
    propagation   = Propagation.REQUIRED,   // 전파 속성
    isolation     = Isolation.DEFAULT,      // 격리 수준
    timeout       = 30,                     // 타임아웃 (초))
    readOnly      = false,                  // 읽기 전용 여부
    rollbackFor   = Exception.class,        // 롤백 대상 예외
    noRollbackFor = RuntimeException.class  // 롤백하지 않을 예외
  )
  public void createOrder(Order order) {
    orderRepository.save(order);
  }
}
```

##### 1️⃣ Propagation (전파 속성) - 가장 중요! ⭐️
문제 상황:
```java
@Transactional
public void methodA() {
  // 트랜잭션 A 시작

  methodB(); // methodB도 @Transactional인데?

  // 트랜잭션을 새로 만들까? 기존 것을 사용할까?
}

@Transactional
public void methodB() {
  // ???
}
```

전파 속성 종류:
|속성|의미|동작|
|-|-|-|
|REQUIRED(기본값)|트랜잭션 필요|기존 있으면 참여, 없으면 새로 생성|
|REQUIRES_NEW|항상 새 트랜잭션|기존 있어도 중단하고 새로 생성|
|SUPPORTS|트랜잭션 지원|있으면 참여, 없어도 실행|
|MADATORY|트랜잭션 필수|기존 있으면 참여, 없으면 예외|
|NOT_SUPPORTED|트랜잭션 불필요|있으면 중단, 트랜잭션 없이 실행|
|NEVER|트랜잭션 금지|있으면 예외|
|NESTED|중첩 트랜잭션|기존 트랜잭션 내 중첩|

가장 많이 쓰이는 REQUIRED 예시:
```java
@Service
public class OrderService {
  @Autowired
  private PaymentService paymentService;

  @Transactional // REQUIRED (기본값)
  public void createOrder(Order order) {
    // 트랜잭션 A 시작
    orderRepository.save(order);

    // paymentService.pay()도 @Transactional(REQUIRED)
    // 트랜잭션 A에 참여! (새로 만들지 않음)
    paymentService.pay(order.getId());

    // 둘 다 트랜잭션 A 안에서 실행됨
    // 하나라도 실패하면 둘 다 롤백!
  }
}

@Service
public class PaymentService {
  @Transactional // REQUIRED
  public void pay(Long orderId) {
    paymentRepository.save(payment);
  }
}
```

REQUIRES_NEW 예시:
```java
@Service
public class OrderService {
  @Autowired
  private LogService logService;

  @Transactional // REQUIRED
  public void createOrder(Order order) {
    // 트랜잭션 A
    orderRepository.save(order);

    // REQUIRES_NEW -> 트랜잭션 B 새로 생성
    logService.log("주문 생성");

    // 주문이 실패해서 롤백되어도 로그는 별도 트랜잭션이라 커밋됨!
    throw new RuntimeException("주문 실패");
  }
}

@Service
public class LogService {
  @Transactional(propagation = Propagation.REQUIRES_NEW)
  public void log(String message) {
    // 트랜잭션 B (독립적)
    logRepository.save(log);
    // 여기는 항상 커밋됨!
  }
}
```

##### 2️⃣ Isolation (격리 수준)
**문제: 동시성 이슈**
```
Transaction A          Transaction B
   │                       │
   ├─ 계좌 조회 (10,000원)  │
   │                       ├─ 계좌 조회 (10,000원)
   ├─ 5,000원 출금          │
   ├─ 커밋 (5,000원)        │
   │                       ├─ 3,000원 출금 
   │                       └─ 커밋 (???)

최종 잔액: 2,000원? 7,000원? 🤔
```

격리 수준:
|수준|설명|발생 가능한 문제|
|-|-|-|
|DEFAULT|DB 기본값 사용|DB마다 다름|
|READ_UNCOMMITTED|커밋 안 된 것도 읽기|Dirty Read|
|READ_COMMITTED|커밋된 것만 읽기|Non-Repetable Read|
|REPETABLE_READ|같은 데이터 반복 조회|Phantom Read|
|SERIALIZABLE|완전 격리|성능 저하|

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public void transfer(Long from, Long to, int amount) {
  // 다른 트랜잭션이 커밋한 데이터만 읽음
}
```

##### 3️⃣ readOnly
읽기 전용 최적화:
```java
@Transactional(readOnly = true)
public List<Order> findOrders() {
  // 읽기만 함
  return orderRepository.findAll();
}
```
효과:
  - ✅ JPA: flush를 하지 않아 성능 향상
  - ✅ JDBC: DB가 읽기 전용으로 최적화
  - ✅ 실수로 데이터 변경 방지

##### 4️⃣ rollbackFor / noRollbackFor
기본 롤백 규칙:
  - ✅ RuntimeException (Unchecked): 롤백
  - ❌ Exception (Checked): 커밋
```java
// 기본 동작
@Transactional
public void method1() {
    // RuntimeException → 롤백 ✅
    throw new RuntimeException("롤백됨");
}

@Transactional
public void method2() throws Exception {
    // Checked Exception → 커밋 ❌
    throw new Exception("커밋됨?!");
}

// 커스텀 설정
@Transactional(rollbackFor = Exception.class)
public void method3() throws Exception {
    // 이제 Checked Exception도 롤백! ✅
    throw new Exception("롤백됨");
}
```

#### 🚨 @Transactional 주의 사항
주의 1: 프록시 방식의 한계
```java
@Service
public class OrderService {
  @Transactional
  public void createOrder() {
    // 트랜잭션 적용 ✅

    updateStock(); // 같은 클래스 내부 호출
  }

  @Transactional
  public void updateStock() {
    // ❌ 트랜잭션 적용 안됨!
    // 이유: 프록시를 거치지 않고 직접 호출
  }
}
```
**왜 안될까?**
```
클라이언트
    ↓
[프록시]
    ├─ 트랜잭션 시작
    ├─ createOrder() 호출 ← 프록시를 거침 ✅
    │    ↓
    │  this.updateStock() ← 직접 호출 (프록시 우회) ❌
    └─ 트랜잭션 커밋
```

주의 2: public method만 가능
```java
@Service
public class OrderService {
    
    @Transactional
    public void method1() {
        // ✅ 트랜잭션 적용
    }
    
    @Transactional
    protected void method2() {
        // ❌ 적용 안됨 (Spring AOP는 public만)
    }
    
    @Transactional
    private void method3() {
        // ❌ 적용 안됨
    }
}
```

주의 3: 스프링 빈만 가능
```java
// ❌ 트랜잭션 적용 안됨
OrderService service = new OrderService();
service.createOrder();

// ✅ 트랜잭션 적용됨
@Autowired
private OrderService service; // 스프링 빈
service.createOrder();
```

### Part 3: Transaction Manager 역할
🎯 Transaction Manager란?
> Transaction Manager는 트랜잭션을 실제로 시작, 커밋, 롤백하는 핵심 컴포넌트입니다.

#### 🔑 PlatformTransactionManager 인터페이스
스프링의 모든 Transaction Manager는 이 인터페이스를 구현합니다.

```java
public interface PlatformTransactionManager {
    
    // 트랜잭션 시작
    TransactionStatus getTransaction(TransactionDefinition definition) 
        throws TransactionException;
    
    // 트랜잭션 커밋
    void commit(TransactionStatus status) 
        throws TransactionException;
    
    // 트랜잭션 롤백
    void rollback(TransactionStatus status) 
        throws TransactionException;
}
```
**핵심 메서드 3개:**
  1. `getTransaction()`: 트랜잭션 시작 또는 기존 트랜잭션 참여
  2. `commit()`: 트랜잭션 커밋
  3. `rollback()`: 트랜잭션 롤백

#### 📊 Transaction Manager 구현체
```
PlatformTransactionManager (인터페이스)
    │
    ├─── DataSourceTransactionManager
    │    └─ JDBC, MyBatis 사용 시
    │
    ├─── JpaTransactionManager
    │    └─ JPA, Hibernate 사용 시
    │
    ├─── HibernateTransactionManager
    │    └─ Hibernate만 사용 시
    │
    ├─── JtaTransactionManager
    │    └─ 분산 트랜잭션 (JTA) 사용 시
    │
    └─── WebLogicJtaTransactionManager
        └─ WebLogic 서버에서 JTA 사용 시
```

#### 🌟 DataSourceTransactionManager
```java
@Configuration
public class AppConfig {
  @Bean
  public DataSource dataSource() {
    // DB Connection Pool 설정
    return new HikariDataSource();
  }

  @Bean
  public PlatformTransactionManager transactionManager() {
    return  new DataSourceTransactionManager(dataSource());
  }
}
```
동작 방식:
```java
// 내부 동작 (간소화)
public class DataSourceTransactionManager implements PlatformTransactionManager {
  private DataSource dataSource;

  @Override
  public TransactionStatus getTransaction(TransactionDefinition def) {
    // 1. DataSource에서 Connection 획득
    Connection conn = dataSource.getConnection();

    // 2. 자동 커밋 크기
    conn.setAutoCommit(false);

    // 3. 격리 수준 설정
    conn.setTransactionIsolation(def.getIsolationLevel());

    // 4. Connection을 ThreadLocal에 저장
    TransactionSynchronizationManager.bindResource(dataSource, conn);

    return new DefaultTransactionStatus(conn, ...);
  }

  @Override
  public void commit(TransactionStatus status) {
    Connection conn = status.getConnection();
    conn.commit(); // JDBC commit
    conn.close();
  }

  @Override
  public void rollback(TransactionStatus status) {
    Connection conn = status.getConnection();
    conn.rollback(); // JDBC rollback
    conn.close();
  }
}
```

#### 🌟 JpaTransactionManager
JPA, Hibernate와 함께 사용:
```java
@Configuration
@EnableJpaRepositories
public class JpaConfig {
  @Bean
  public LocalContainerEntityManagerFactoryBean entityManagerFactory() {
    // JPA EntityManager 설정
    return new LocalContainerEntityManagerFactoryBean();
  }

  @Bean
  public PlatformTransactionManager transactionManager() {
    return new JpaTransactionManager(
      entityManagerFactory().getObject()
    );
  }
}
```

동작 방식:
```java
// 내부 동작 (간소화)
public class JpaTransactionManager implements PlatformTransactionManager {
  private EntityManagerFactory emf;

  @Override
  public TransactionStatus getTransaction(TransactionDefinition def) {
    // 1. EntityManager 생성
    EntityManager em = emf.createEntityManager();

    // 2. JPA 트랜잭션 시작
    em.getTransaction().begin();

    // 3. EntityManager를 ThreadLocal에 저장
    TransactionSynchronizationManager.bindResource(
      emf, em
    );

    return new DefaultTransactionStatus(em, ...);
  }

  @Override
  public void commit(Transaction status) {
    EntityManager em = status.getEntityManager();
    em.flush();       // 변경사항을 DB에 반영
    em.getTransaction().commit(); // JPA commit
    em.close();
  }

  @Override
  public void rollback(TransactionStatus status) {
    EntityManager em = status.getEntityManager();
    em.getTransaction().rollback(); // JPA rollback
    em.close();
  }
}
```

#### 📊 DataSource vs JPA Transaction Manager 비교
|항목|DataSourceTransactionManager|JpaTransactionManager|
|-|-|-|
|**사용 기술**|JDBC, MyBatis|JPA, Hibernate|
|**관리 대상**|Connection|EntityManager|
|**획득**|`dataSource.getConnection()`|`emf.createEntityManager()`|
|**시작**|`conn.setAutoCommit(false)`|`em.getTransaction().begin()`|
|**커밋**|`conn.commit()`|`em.getTransaction().commit()`|
|**롤백**|`conn.rollback()`|`em.getTransaction().rollback()`|
|**종료**|`conn.close()`|`em.close()`|

#### 🔃 Transaction Manager의 핵심 역할
```
Transaction Manager의 3대 책임:

1. 트랜잭션 리소스 관리
   ├─ Connection 또는 EntityManager 획득
   └─ ThreadLocal에 저장 (스레드 안전)

2. 트랜잭션 경계 설정
   ├─ 시작: begin / setAutoCommit(false)
   ├─ 종료: commit / rollback
   └─ 전파 속성 적용 (REQUIRED, REQUIRES_NEW 등)

3. 예외 변환
   └─ DB 예외를 스프링 예외로 변환
```

#### 🌟 Transaction Synchronization Manager
ThreadLocal 기반 트랜잭션 동기화:
```java
public abstract class TransactionSynchronizationManager {

  // ThreadLocal로 트랜잭션 리소스 저장
  private static final ThreadLocal<Map<Object, Object>> resources = 
    new NamedThreadLocal<>("Transactional resources");

  // 현재 스레드의 connection 저장
  public static void bindResource(Object key, Object value) {
    Map<Object, Object> map = resources.get();
    if (map == null) {
      map = new HashMap<>();
      resource.set(map);
    }
    map.put(key, value);
  }

  // 현재 스레드의 connection 가져오기
  public static Object getResource(Object key) {
    Map<Object, Object> map = resources.get();
    return map != null ? map.get(key) : null;
  }
}
```

왜 ThreadLocal을 사용할까?
```java
@Service
public class OrderService {
  @Autowired
  private OrderRepository orderRepository;
  @Autowired
  private PaymentRepository paymentRepository;
  
  @Transactional
  public void createOrder(Order order) {
      // 1. Transaction Manager가 Connection 획득
      // 2. ThreadLocal에 저장
      
      orderRepository.save(order);
      // → Repository가 ThreadLocal에서 같은 Connection 꺼내 씀
      
      paymentRepository.save(payment);
      // → 이것도 ThreadLocal에서 같은 Connection 꺼내 씀
      
      // 같은 Connection = 같은 트랜잭션!
  }
}
```
**효과:**
  - ✅ 같은 스레드 내 모든 Repository가 같은 Connection 사용
  - ✅ Connection을 파라미터로 전달할 필요 없음
  - 스레드 안전

### 트랜잭션 시작/커밋/롤백 전체 플로우

🎯 예제 시나리오
```java
@RestController
public class OrderController {
  @Autowired
  private OrderService orderService; // proxy 주입됨

  @PostMapping("/orders")
  public ResponseEntity<Order> createOrder(@RequestBody OrderRequest request) {
    orderService.createOrder(request.toOrder());
    return ResponseEntity.ok().build();
  }
}

@Service
public class OrderService {
  @Autowired
  private OrderRepository orderRepository;

  @Autowired
  private PaymentService paymentService;

  @Transactional
  public void createOrder(Order order) {
    // 1. 주문 저장
    orderRepository.save(order);

    // 2. 결제 처리 (다른 @Transactional 메서드 호출)
    paymentService.processPayment(order.getId());

    // 3. 재고 감소
    order.decreaseStock();
  }
}

@Service
public class PaymentService {
    @Autowired
    private PaymentRepository paymentRepository;
    
    @Transactional(propagation = Propagation.REQUIRED)
    public void processPayment(Long orderId) {
        Payment payment = new Payment(orderId);
        paymentRepository.save(payment);
    }
}
```

#### 전체 호출 흐름
```
[1] HTTP 요청
      ↓
[2] OrderController.createOrder()
      ↓
[3] OrderService(프록시).createOrder() <- 프록시 진입!
      ↓
[4] TransactionInterceptor.invoke()
      ↓
[5] TransactionAspectSupport.invokeWithinTransaction()
      ↓
[6] TransactionManager.getTransaction() <- 트랜잭션 시작
      ↓
[7] OrderService(원본).createOrder() <- 실제 메서드 실행
      ↓
[8] orderRepository.save()
      ↓
[9] PaymentService(프록시).processPayment() <- 중첩 호출
      ↓
[10] 기존 트랜잭션에 참여 (REQUIRED)
      ↓
[11] paymentRepository.save()
      ↓
[12] 메서드 완료
      ↓
[13] TransactionManager.commit() <- 트랜잭션 커밋
```

#### 단계 별 상세 분석
Phase 1: 프록시 진입
```java
// [3] 프록시 메서드 호출
orderService.createOrder(order);

// 실제로는 이렇게 동작:
OrderService$$CGLIB$$12345678.createOrder(order) {
  // MethodInvocation 생성
  MethodInvocation invocation = createMethodInvocation(
    "createOrder",
    new Object[]{order}
  );

  // [4] TransactionInterceptor에게 위임
  return interceptor.invoke(invocation);
}
```

Phase 2: TransactionInterceptor 실행
```java
// [4] TransactionInterceptor.invoke()
public class TransactionInterceptor extends TransactionAspectSupport {
  @Override
  public Object invoke(MethodInvocation invocation) throws Throwable {
    // 1. 대상 클래스 정보
    Class<?> targetClass = invocation.getThis().getClass();

    // 2. 호출할 메서드
    Method method = invocation.getMethod();

    // [5] 트랜잭션 내에서 메서드 실행
    return invokeWithinTransaction(
      method,
      targetClass,
      invocation::proceed // 실제 메서드 호출
    );
  }
}
```

Phase 3: 트랜잭션 속성 파싱
```java
// [5] TransactionAspectSupport.invokeWithinTransaction()
protected Object invokeWithinTransaction(
  Method method,
  Class<?> targetClass,
  InvocationCallback invocation
) throws Throwable {
  // 1. @Transactional 속성 읽기
  TransactionAttribute txAttr = getTransactioAttributeSource()
    .getTransactionAttribute(method, targetClass);
  
  // txAttr 내용:
  // - propagation: REQUIRED
  // - isolation: DEFAULT
  // - timeout: -1 (무제한)
  // - readOnly: false
  // - rollbackFor: [RuntimeException.class]

  // 2. TransactionManager 찾기
  PlatformTransactionManager tm = determineTransactionManager(txAttr);

  // 3. 트랜잭션 내에서 실행
  return createTransactionIfNecessary(tm, txAttr, 메서드명)
    .execute(status -> {
      return invocation.proceedWithInvocation();
    });
}
```

Phase 4: 트랜잭션 시작
```java
// [6] TransactionManager.getTransaction()
public class DataSourceTransactionManager {
  @Override
  public TransactionStatus getTransaction(TransactionDefinition definition) {
    // 1. 기존 트랜잭션이 있는지 확인
    Object transaction = doGetTransaction();

    // 2. 기존 트랜잭션 확인
    if (isExistingTransaction(transaction)) {
      // 전파 속성에 따라 처리
      return handleExistingTransaction(definition, transaction);
    }

    // 3. 새 트랜잭션 시작 필요
    return startNewTransaction(definition, transaction);
  }

  private TransactionStatus startNewTransaction(
    TransactionDefinition definition,
    Object transaction
  ) {
    // 1. Connection 획득
    Connection conn = dataSource.getConnection();

    // 2. 자동 커밋 끄기
    conn.setAutoCommit(false);

    // 3. 격리 수준 설정
    if (definition.getIsolationLevel() != Isolation.DEFAULT) {
      conn.setTransactionIsolation(
        definition.getIsolationLevel().value()
      );
    }

    // 4. ThreadLocal에 Connection 저장
    TransactionSynchronizationManager.bindResource(
      dataSource,
      new ConnectionHolder(conn)
    );

    // 5. TransactionStatus 반환
    return new DefaultTransactionStatus(
      transaction,
      true,   // newTransaction = true
      conn,
      definition
    );
  }
}
```

**현재 상태:**
```
Thread: http-nio-8080-exec-1
    │
    ├─ ThreadLocal<Map<DataSource, ConnectionHolder>>
    │   └─ HikariDataSource → ConnectionHolder
    │                          └─ Connection (autoCommit=false) ✅
    │
    └─ TransactionStatus
        ├─ transaction: DataSourceTransaction
        ├─ newTransaction: true
        ├─ connection: HikariProxyConnection@123abc
        └─ completed: false
```

Phase 5: 실제 메서드 실행
```java
// [7] 원본 메서드 실행
orderService.createOrder(order);

public void createOrder(Order order) {
  // [8] orderRepository.save()
    orderRepository.save(order);
    // Repository 내부에서 Connection 가져오기:
    Connection conn = TransactionSynchronizationManager.getResource(dataSource);
    // 같은 Connection으로 SQL 실행
    PreparedStatement stmt = conn.prepareStatement(
      "INSERT INTO orders (id, amount) VALUES (?, ?)"
    );
    stmt.setLong(1, order.getId());
    stmt.setInt(2, order.getAmount());
    stmt.executeUpdate(); // 아직 커밋 안됨!
  
  // [9] PaymentService 호출
  paymentService.processPayment(order.getId());
}
```

Phase 6: 중첩 트랜잭션 처리 (REQUIRED)
```java
// [9] PaymentService(프록시).processPayment() 호출
// -> TransactionInterceptor 다시 실행

public TransactionStatus getTransaction(TransactionDefinition definition) {
  // 1. 기존 트랜잭션 확인
  ConnectionHolder holder = (ConnectionHoder)
    TransactionSynchronizationManager.getResource(dataSource);
  
  if (holder != null && holder.isTransactionActive()) {
    // 기존 트랜잭션 있음! ✅

    // 2. REQUIRED 속성 확인
    if (definition.getPropagationBehavior() == PROPAGATION_REQUIRED) {
      // 기존 트랜잭션에 참여!
      return new DefaultTransactionStatus(
        transaction,
        false,    newTransaction = false (새 트랜잭션 아님!)
        holder.getConnection,
        definition
      );
    }
  }
}
```
**중요 포인트:**
```
OrderService.createOrder() [트랜잭션 A]
          ↓
orderRepository.save()  [트랜잭션 A 사용]
          ↓
PaymentService.processPayment() [트랜잭션 A에 참여] ⭐️
          ↓
paymentRepository.save()  [트랜잭션 A 사용]
          ↓
모두 같은 Connection, 같은 트랜잭션!
하나라도 실패하면 전부 롤백!
```

Phase 7: 메서드 완료 커밋
```java
// [12] createOrder() 메서드 정상 완료
// -> TransactionInterceptor로 돌아옴

protected Object invokeWithinTransaction(...) {
  try {
    // 메서드 실행
    Object result = invocation.proceedWithInvocation();

    // [13] 성공 시 커밋
    commitTransactionAfterReturning(txInfo);

    return result;
  } catch (Throwable ex) {
    // 실패 시 롤백
    completeTransactionAfterThrowing(txInfo, ex);
    throw ex;
  }
}

// [13] 커밋 실행
private void commitTransactionAfterReturning(TransactionInfo txInfo) {
  if (txInfo.getTransactionStatus().inNewTransaction()) {
    // 이 메서드가 트랜잭션을 시작했다면 커밋
    txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
  }
  // 참여한 경우 (REQUIRED) -> 커밋하지 않음
}

// TransactionManager.commit()
public void commit(TransactionStatus status) {
  if (status.isCompleted()) {
    throw new IllegalTransactionStateException("이미 완료된 트랜잭션");
  }

  DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;

  // 롤백 마크가 있는지 확인
  if (defStatus.isLocalRollbackOnly()) {
    rollback(status);
    return;
  }

  // 실제 커밋
  processCommit(defStatus);
}

private void processCommit(DefaultTransactionStatus status) {
  try {
    // 1. Connection.commit() 호출
    Connection conn = status.getConnection();
    conn.commit(); ✅

    // 2. TransactionSynchronization 콜백 호출
    triggerAfterCommit();
  } finally {
    // 3. 리소스 정리
    cleanupAfterCompletion(status);
  }
}

private void cleanupAfterCompletion(TransactionStatus status) {
  // 1. ThreadLocal에서 Connection 제거
  TransactionSynchronizationManager.unbindResource(dataSource);

  // 2. Connection 반환 (커넥션 풀로)
  Connection conn = status.getConnection();
  DataSourceUtils.releaseConnection(conn, dataSource);
}
```
**최종 상태:**
```
Thread: http-nio-8080-exec-1
    │
    ├─ ThreadLocal<Map> → (비어있음, 정리됨) ✅
    │
    └─ DB 상태
        ├─ orders 테이블: INSERT 커밋됨 ✅
        └─ payments 테이블: INSERT 커밋됨 ✅
```

### 트랜잭션 롤백 시나리오
🔴 예외 발생 케이스
```java
@Transactional
public void createOrder(Order order) {
  orderRepository.save(order);

  // 예외 발생!
  if (order.getAmount() > 10000) {
    throw new RuntimeException("금액 초과");
  }

  paymentService.processPayment(order.getId());
  // 실행 안됨
}
```

롤백 플로우
```java
protected Object invokeWithinTransaction(...) {
  try {
    Object result = invocation.proceedWithInvocation(); // 예외 발생!
    commitTransactionAfterReturning(txInfo);
    return result;
  } catch (Throwable ex) {
    // [예외 캐치!]
    completeTransactionAfterThrowing(txInfo, ex);
    throw ex;
  }
}

private void completeTransactionAfterThrowing(
  TransactionInfo txiInfo, Throwable ex
) {
  // 1. 롤백해야 하는 예외인지 확인
  if (txInfo.transactionAttribute.rollbackOn(ex)) {
    // RuntimeException -> true (기본값)
    // Exception -> false (기본값)

    // 2. 롤백 실행
    txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
  } else {
    // 롤백하지 않고 커밋
    txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
  }
}

// TransactionManager.rollback()
public void rollback(TransactionStatus status) {
  DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
  processRollback(defStatus);
}

private void processRollback(DefaultTransactionStatus status) {
  try {
    // 1. Connection.rollback() 호출
    Connection conn = status.getConnection();
    conn.rollback(); 🔄

    // 2. TransactionSynchronization 콜백
    triggerAfterCompletion(STATUS_ROLLED_BACK);
  } finally {
    // 3. 리소스 정리
    cleanupAfterCompletion(status);
  }
}
```
**결과:**
```
DB 상태:
├─ orders 테이블: 변경사항 없음 (롤백됨) 🔄
└─ payments 테이블: 변경사항 없음 (실행조차 안됨)
```

### 🎯 핵심 컴포넌트 관계도
```
┌─────────────────────────────────────────────┐
│          Client (Controller)                │
└─────────────────┬───────────────────────────┘
                  │ 호출
                  ↓
┌─────────────────────────────────────────────┐
│         Proxy (CGLIB 생성)                   │
│  ┌─────────────────────────────────────┐    │
│  │   TransactionInterceptor            │    │
│  │   ├─ @Transactional 속성 읽기        │    │
│  │   └─ TransactionManager 호출        │    │
│  └─────────────────────────────────────┘    │
└─────────────────┬───────────────────────────┘
                  │
                  ↓
┌─────────────────────────────────────────────┐
│      PlatformTransactionManager             │
│  ├─ getTransaction() (트랜잭션 시작)          │
│  ├─ commit() (커밋)                          │
│  └─ rollback() (롤백)                        │
└─────────────────┬───────────────────────────┘
                  │
                  ↓
┌─────────────────────────────────────────────┐
│  TransactionSynchronizationManager          │
│  └─ ThreadLocal<Connection>                 │
└─────────────────┬───────────────────────────┘
                  │
                  ↓
┌─────────────────────────────────────────────┐
│         DataSource (Connection Pool)        │
│  └─ Connection (실제 DB 연결)                │
└─────────────────────────────────────────────┘
```

## 학습 후 설명

### Transaction 이란?
하나의 요청에서 여러 개의 데이터 작업을 수행해야 하는 경우, 이 여러 건의 데이터 작업을 논리적으로 하나로 묶어서 처리하는 것을 말합니다. 이 논리적으로 묶인 트랜잭션은 그래서 내부의 여러 데이터 작업 건이 "전부 성공" 하거나 "전부 실패" 해야 합니다.

예를 들어 A(100만원)와 B(200만원)가 있을 때, A가 B에게 20만원을 송금한다고 가정해봅시다.
  - 데이터 작업 1: A의 계좌에서 20만원을 차감 (결과: A 계좌 80만원)
  - 데이터 작업 2: B의 계좌에서 20만원을 증감 (결과: B 계좌 220만원)
그런데 만약 `데이터 작업 1`은 성공했는데 `데이터 작업 2`처리 과정에서 실패한다면 A 계좌는 80만원이지만 B 계좌는 여전히 200만원일 것입니다. 이러한 불일치를 제거하기 위해 이 2개의 데이터 처리 작업을 하나로 묶어서 관리하는 것을 트랜잭션 처리라고 하는 것입니다.

그래서 트랜잭션으로 이 2개 데이터 처리 작업이 묶여있는 상태에서는:
  - 데이터 작업 1 과정에서 실패: A 계좌에서 차감 작업 자체가 발생하지 않으므로 A, B 계좌 모두 정상 상태가 됩니다.
  - 데이터 작업 2 과정에서 실패: A 계좌에서는 차감되었으나 B 계좌의 증감 작업이 발생하지 않았습니다. 그러나 이 때, A 계좌 차감 작업 자체도 뒤로 돌려서(rollback) A, B 계좌 모두 이전 상태가 유지되게끔 합니다. 이 경우도 A, B 계좌 모두 정상 상태입니다.
  - 데이터 작업 1,2 모두 성공: 이 경우에는 기존과 동일하게 A 차감, B 증감 작업이 정상적으로 발생합니다.

따라서 트랜잭션 처리는 내부의 여러 데이터 처리 과정에서 실패가 발생하는 경우, 이전에 어떤 처리(Insert, Update, Delete)가 있었던간에 상관 없이 해당 작업 자체를 없었던 일로 되돌리는 것이 핵심입니다. 그래서 데이터 정합성을 지킬 수 있게 하는 것입니다.

### 트랜잭션 처리 과정
구체적인 RDBMS마다 용어의 차이는 있지만 일반적으로 트랜잭션 처리는 다음의 과정을 거칩니다.
  1. Begin: 이 때, Connection을 생성합니다 (또는 Connection Pool에서 가져옵니다)
  2. Business Logic: Insert/Update/Delete 등의 쿼리문을 통해 데이터가 생성/수정/삭제 처리 됩니다.
  3. Commit: 최종적으로 비즈니스 로직 실행이 성공하면 Commit을 통해 해당 처리된 데이터를 '영구적'으로 DB에 저장합니다.
  4. Rollback: 만약 비즈니스 로직 처리 과정에서 오류가 발생하면 Commit이 아닌 Rollback을 통해 트랜잭션으로 묶인 모든 데이터 처리 결과를 되돌립니다.
  5. Release: 최초에 생성했던 Connection을 Connection Pool에 반환합니다.

### Proxy 패턴이란?
원본 객체(Real Object)가 있고 이 원본 객체 전/후로 어떤 특수한 처리를 하고 싶을 때 Proxy 패턴을 사용할 수 있습니다.
예를 들어, 어떤 Class A의 B 라는 메소드가 있고 이 B 메소드가 호출될 때마다 호출 정보를 로깅하고 싶은 경우에 Proxy 패턴을 사용하여 Class A의 Proxy 객체를 만들어서 클라이언트에 전달하고 클라이언트는 이 Proxy 객체를 호출합니다. Proxy 객체 내부에서는 A 메소드가 호출되는 순간에 이 호출을 가로채서 의도한 로깅 작업을 수행하고 그 이후에 원본 객체(Class A)의 B 메소드를 직접 호출합니다.

이 패턴은 원본 객체에는 실제로 중요한 비지니스 로직만 구현하고 이 비즈니스 로직을 호출 전/후에 처리해야 할 다른 로직은 Proxy 내부에 구현합니다. 이를 통해 중요한 로직만 원본 객체에서 관리하고 그 이외의 로직은 분리하여 상대적으로 깔끔하고 관리하기 쉬운 코드를 작성할 수 있게 합니다. 또한 단순히 1회성에 그치는 전/후 코드가 아니라 반복적으로 실행되는 전/후 코드가 있는 경우에 이 Proxy 패턴이 빛을 발합니다.

이 Proxy 패턴은 우리가 지금 살펴보고 있는 트랜잭션 처리에 알맞은 디자인 패턴이라고 할 수 있습니다.
원본 클래스(비즈니스 로직)은 그대로 두고 트랜잭션 처리에 필요한 기타 다른 로직을 Proxy 클래스에 작성하게 하는 것입니다. 이를 통해 개발자는 본인이 구현해야 하는 Feature 코드를 원본 클래스에 작성하고 트랜잭션 관련 코드는 Transaction만 전문적으로 처리하는 Proxy 클래스에 일임합니다.

### Spring에서 트랜잭션을 어떻게 처리하는지?
Spring Framework에서는 Spring AOP를 활용하여 트랜잭션을 처리합니다.

Spring AOP는 프레임워크 차원에서 Proxy 패턴을 손쉽게 사용할 수 있게 해주는 기능입니다.
개발자가 `@Transactional` 어노테이션을 클래스나 메소드에 선언하면 해당 클래스, 해당 메소드를 트랜잭션으로 처리한다는 의미고 이것이 Spring AOP에 의해 관리됩니다. 

이제부터 구체적으로 Spring이 트랜잭션을 어떻게 처리하는지 살펴보겠습니다.

1. 최초에 Spring Application이 시작됨
  - Spring Framework은 @ComponentScan 설정을 통해 지정된 경로(패키지)의 모든 클래스를 순회하면서 스프링 컴포넌트로 선언된 클래스를 찾습니다. 스프링 컴포넌트(@Component) 클래스는 스프링에서 빈으로 관리합니다.
  - BeanPostProcessor에서 @Component 클래스를 스프링 빈으로 등록하는 과정에서 스프링에서는 `@Transactional` 어노테이션이 클래스 또는 클래스 내부에 선언되어 있는지 확인합니다. 만약 선언되어 있다면 해당 클래스를 Proxy로 감싸서 빈으로 등록합니다
2. 사용자 요청을 @Controller에서 받아서 @Transactional이 선언된 특정 메소드가 호출되는 시점
  - 이 때, 메소드가 곧바로 실행되는 것이 아니라 Spring AOP에 의해 Proxy 클래스가 대신 호출됩니다. 스프링에서는 기본적으로 CGLIB proxy 클래스를 사용합니다.
  - 이 Proxy 클래스는 target method를 Override 합니다. 그래서 클라이언트가 method를 호출할 때, 실제 객체(Real Object)의 target method가 아니라 Proxy 클래스의 Override 된 메소드가 호출됩니다. 그리고 이 메소드 내부에는 TransactionInterceptor가 선언되어 있어서 트랜잭션 관련 처리를 담당합니다.
  - TransactionInterceptor는 target method 호출 전에 일단 TransactionManager에게 트랜잭션 시작을 지시합니다.
  - TransactionManager는 connection 생성, autoCommit=false, TransactionSynchronizationManager에게 생성한 Connection을 저장하는 등의 처리를 수행합니다.
  - TransactionInterceptor는 TransactionManager가 트랜잭션 시작 처리를 완료하면 이제 target method를 호출합니다.
  - target method가 정상적으로 종료되면 TransactionInterceptor는 다시 TransactionManager에게 commit을 지시합니다.
    - TransactionManager는 DB Commit을 실행하고 Connection을 TransactionSyncrhonization에서 제거합니다. 그리고 Connection을 Pool에 다시 반환합니다.
  - 만약 target method가 비정상적으로 종료되면 TransactionInterceptor는 TransactionManager에게 rollback을 지시합니다.
    - TransactionManager는 DB Rollback을 실행하고 TransactionSyncrhonization에서 제거합니다. 그리고 Connection을 Pool에 다시 반환합니다.
3. 사용자 요청을 처리하는 과정에서 @Transactional 선언 메소드를 중첩으로 호출하는 경우
  - 예를 들어, Controller -> Class A(Proxy) -> Class B(Proxy) 순으로 호출된다고 가정해봅시다. Class A와 Class B의 메서드는 모두 @Transactional로 선언되어 있습니다.
  - 이때는 @Transactional 어노테이션이 선언될 때, 내부 attribute를 어떤 값으로 설정했는지가 중요합니다. 그리고 여러 attribute 중에서도 propagation(전파 속성)이 `REQUIRED` 라고 하겠습니다. REQUIRED는 기존 트랜잭션이 있으면 '참여'하고 없으면 새로 생성한다는 의미입니다.
  - Class A의 메소드가 호출될 때는 앞에서 설명한 것과 동일합니다. 그런데 Class B의 메서드가 호출될 때는 조금 다르게 동작합니다. Class B 역시 CGLIB Proxy로 처리가 되고 그 내부에는 TransactionInterceptor가 있습니다. TransactionInterceptor는 TransactionManager에게 트랜잭션 시작을 지시하는데, 이 때 TransactionManager에서 현재 실행 흐름, 그리고 @Transactional 어노테이션의 attribute 값을 참조하여 새 Connection을 만들지 아니면 기존 Connection을 재사용할지를 결정합니다. `REQUIRED` propagation 속성이 지정된 경우 TransactionManager는 기존 Connection을 TransactionSynchronizationManager로부터 꺼내와서 다시 재사용 하도록 합니다.
  - 그래서 만약 Class B의 실제 메서드 처리 과정에서 오류가 발생하는 경우에는 Class B 메서드 처리 과정에서 발생했던 모든 데이터 처리(Insert/Update/Delete) 실행이 롤백되고 이어서 Class A 메서드 처리 과정(Class B 메서드 이전에 실행되었던)에서 발생했던 모든 데이터 처리 실행도 롤백됩니다.
4. TransactionSynchronizationManager을 사용하는 이유?
  - TransactionSnchronizationManager는 ThreadLocal을 사용하여 Connection을 저장합니다. ThreadLocal은 각 스레드마다 독립적인 변수를 가질 수 있게 해주는 기능입니다.
  - 동시에 여러 요청이 들어와도 각 요청 별로 독립적인 Connection을 사용할 수 있게 해줍니다.
  - 실제로 Connection을 사용하는 곳은 Repository 입니다. Service layer에서 Repository 코드까지 Connection을 실제로 파라미터로 넘기지 않아도 되게끔 해줍니다.
  - 같은 트랜잭션 내의 모든 코드가 자동으로 동일한 Connection을 사용하게 해줍니다.


## 참고 자료
  - [Claude 학습 자료](https://claude.ai/share/18a6c1fb-da5b-4c4b-b75b-8a23d07d2856)
  - [쉬운 코드 - BJ.43-1 concurrency control 기초 이론: schedule과 serializability 설명! 트랜잭션 isolation 보장을 위한 이론](https://www.youtube.com/watch?v=DwRN24nWbEc)

