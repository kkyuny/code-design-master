### ch3. 객체를 풍부하게 표현하기
## ✔ Lombok을 잘 사용해야 객체 디자인을 망치치 않는다.
- @Data는 지양하자
    - @Setter가 무분별하게 남용된다.
  ```java
  @Column(name = "email, nullable = false, updatable = false)
  private String email;
  // 위와 같이 변경이 되어선 안되는 코드에 Setter 설정은 적절하지 않다.
  ```
    - 양방향 관계 설정 시 `ToString 양방향 순환참조` 문제
        - 순환참조가 일어나는 부분에 `@Exclude`를 사용하여 해결할 수 있다.
    - `@EqualsAndHashCode`의 무분별한 남용
        - 사용하지 않는 모든 필드에 대해서 적용이 되기 때문에 필요한 필드에만 설정해서 사용하는 것이 효율적이다.
    - 롬복의 설정은 `lombok.config`의 공통 처리를 통해 제한
        - 롬복 config에 어긋나는 롬복 어노테이션은 빌드 시 관리할 수 있다.
        ```
        lombok.setter.flagUsage=ERROR
        -> @Setter 사용시 빌드 실패
        lombok.toString.flagUsage=WARNING
        -> @ToString 사용시 경고 출력
        ``` 
    - 클래스 상단의 `@Builder`는 지양하자.
        - 객체가 빌더를 통해 생성 시 필드의 기본 값은 null로 생성된다.
        - 필수 값 등이 누락될 수 있기 때문에 안정적인 방식이 아니다.
        - 생성자에 `@Builder`를 사용하여 필수 값을 입력받는 방식을 사용하는 것이 안정적이다.
        - 필수 값에 대한 검증은 생성자 내에서 `Assert.hasText(필드, 메세지)` 등을 사용해서 처리한다.        
        - 최종적으로 객체가 생성되는 시점에 필수 값과 선택 값이 구분되는 등 객체를 도메인 요구사항에 맞게 만드는 것이 중요하다.
          
### ch4. Exception 처리 방법
## ✔ 오류 코드보다 예외를 사용하자.
- 로직으로 예외 처리 가능하다면 `try-catch`는 지양한다.
- `try-catch`를 사용한다면 예외를 구체적으로 작성하는 것이 좋다.
## ✔ Checked Exception VS UnChecked Exception
- Checked Exception: `반드시 예외 처리해야함` / `Rollback 안됨` / `IOException, SQLException 등`
- UnChecked Exception: `예외 처리하지 않아도됨` / `Rollback 진행` / `NullPointerException, IllegalArgumentException 등`
    > 체크드 익셉션은 롤백이 되지 않기 때문에 다른 곳으로 예외를 전파하는건 바람직하지 않다.    
    > 예외를 언체크드 익셉션으로 처리할 수 있다면 언체크드 익셉션으로 처리하는 것이 좋다.
    > 체크드 익셉션은 롤백이 되지않고 언체크드 익셉션은 롤백이 되는 이유는
    > 체크드 익셉션은 예측이 가능한 예외이기 때문에 비즈니스적으로 처리가 가능하기 때문에 롤백을 지원하고
    > 언체크드 익셉션은 예측이 불가능한 예외이기 때문에 롤백을 지원한다.

### ch5. API Server Error 처리
## ✔ 통일된 Error Response를 갖자.
- ErrorResponse 객체
```java
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
// Map<Key, Value> -> 정확한 Error Response 형태가 런타임시 결정되기 때문에 Map은 지양
public class ErrorResponse { 
    private String message;
    private int status;
    private List<FieldError> errors;
    private String code;
    ...

    @Getter
    @NoArgsConstructor(access = AccessLevel.PROTECTED)
    public static class FieldError {
        private String field;
        private String value;
        private String reason;
        ...
    }
}
```
- ErrorCode
```java
@Getter
@RequiredArgsConstructor
public enum ErrorCode {
    // Common
    INVALID_INPUT_VALUE(400, "C001", "Invalid Input Value"),
    METHOD_NOT_ALLOWED(405, "C002", "Invalid Input Value"),
    INTERNAL_SERVER_ERROR(500, "C003", "Server Error"),
    ...

    private final int status;
    private final String code; // HTTP Status Code
    private final String message;
    
    ErrorCode(final int status, final String code, final String message) {
        this.status = status;
        this.message = message;
        this.code = code;
    }
}
```
## ✔ @ControllerAdvice를 활용한 일관된 예외 핸들링
- 컨트롤러 예외처리
    - 컨트롤러에서 모든 요청에 대한 값 검증을 진행하고 서비스 레이어를 호출해야한다.
    - 컨트롤러의 중요 책임 중 하나는 요청 값에 대한 검증이다.
    - 스프링은 `@ControllerAdvice`을 통해 일관성 있게 처리할 수 있다.
    ```java
    @ControllerAdvice
    @Slf4j
    public class GlobalExceptionHandler {
        // 모든 예외를 잡을 수 있는 핸들러
        @ExceptionHandler(Exception.class)
        protected ResponseEntity<ErrorResponse> handleException(Exception e){
            log.error("handlerException", e);
            final ErrorResponse response = ErrorResponse.of(ErrorCode.INTERNAL_SERVER_ERROR);
    
            return new ResponseEntity<>(response, HttpStatus.valueOf(ErrorCode.INTERNAL_SERVER_ERROR.getStatus()));
        }
    
        // 구체적인 예외를 잡을 수 있는 핸들러
        @ExceptionHandler(IllegalArgumentException.class
                        ,IllegalStateException.class) // 배열 형태로도 받을 수 있다.
        protected ResponseEntity<ErrorResponse> handleIllegalException(RuntimeException e){
            log.error("IllegalHandlerException", e);
            final ErrorResponse response = ErrorResponse.of(ErrorCode.INVALID_INPUT_VALUE);
    
            return new ResponseEntity<>(response, HttpStatus.valueOf(ErrorCode.INVALID_INPUT_VALUE.getStatus()));
        }
    }
    ```
## ✔ 계층화를 통한 Business Exception 처리 방법
- 비즈니스 로직을 수행하는 코드에서 예외가 발생하는 경우 최상위 `Business Exception`을 생성(exception 패키지 별도 생성)하여 처리하면 통일감 있는 예외처리를 할 수 있다.
    - 글로벌 익셉션 핸들러
    ```java
    @ExceptionHandler(BusinessException.class)
    protected ResponseEntity<ErrorResponse> handleBusinessException(final BusinessException e) {
        log.error("handleEntityNotFoundException", e);
        final ErrorCode errorCode = e.getErrorCode();
        final ErrorResponse response = ErrorResponse.of(errorCode);
        return new ResponseEntity<>(response, HttpStatus.valueOf(errorCode.getStatus()));
    }
    ```
    - 비즈니스 익셉션
    ```java
    public class BusinessException extends RuntimeException {
        private final ErrorCode errorCode; // final 변수이기 때문에 반드시 초기화가 필요
    
        public BusinessException(ErrorCode errorCode) {
            super(errorCode.getMessage());
            this.errorCode = errorCode;
        }
    }
    ```
## ✔ 효율적인 Validaion
- Custom Validation 어노테이션 만들기
    - 컨트롤러에서 직접 값 검증을 진행할 수 있지만 `검증 어노테이션`을 만들어 처리하면 중복되는 코드를 줄일 수 있다.
      ```java
      // 컨트롤러
      @PostMapping
      public Member create(@RequestBody @Valid final SignUpRequest dto) {
          ...
      }

      // dto
      @Getter
      @NoArgsConstructor(access = AccessLevel.PROTECTED)
      public class SignUpRequest {          
          @Email
          @EmailUnique
          private String email;

        public SignUpRequest(String email) {
            this.email = email;
        }
      }
      ```
      > 컨트롤러에 요청이 들어왔을 때 이메일 형식검사를 진행한다.
    - 커스컴 어노테이션 만들기
      ```java
      @Documented
      @Constraint(validatedBy = EmailDuplicationValidator.class)
      @Target({ElementType.METHOD, ElementType.FIELD})
      @Retention(RetentionPolicy.RUNTIME)
      public @interface EmailUnique {
          String message() default "Email is Duplication";
    
          Class<?>[] groups() default {};
    
          Class<? extends Payload>[] payload() default {};
      }
      ```
      - **@Constraint**: 어떤 Validator 클래스와 연결되는지를 지정함(검증 로직)
      - **@Target**: 어노테이션을 어디에 붙일 수 있는지 (예: 필드, 메서드 등)
      - **@Retention(RetentionPolicy.RUNTIME)**: 런타임에도 유지되어야 검증 가능    
      - **message / groups / payload**: Bean Validation 표준 규약 필수 요소
      - **@interface**: 자바에서 어노테이션을 정의할 때 쓰는 문법
      - **@인터페이스인 이유**: 어노테이션은 필드나 메서드 위에 붙여서 메타데이터를 알려주는 도구이기 때문에 정적 구조(=interface)로 정의
    - 어노테이션 검증 로직
      ```java
      @Component
      @RequiredArgsConstructor
      public class EmailDuplicationValidator implements ConstraintValidator<EmailUnique, String> {
          private final MemberRepository memberRepository;

          @Override
          public void initialize(EmailUnique emailUnique) {

          }

          @Override
          public boolean isValid(String email, ConstraintValidatorContext cxt) {
              boolean isExistEmail = memberRepository.existsByEmail(email);

              if (isExistEmail) {
                  cxt.disableDefaultConstraintViolation();
                  cxt.buildConstraintViolationWithTemplate(
                          MessageFormat.format("Email {0} already exists!", email))
                      .addConstraintViolation();
              }
              return !isExistEmail;
          }
      }
      ```
      - **implements ConstraintValidator<EmailUnique, String>**: `@EmailUnique`가 붙은 `String 타입` 필드에 대해 유효성 검사하는 클래스라는 의미
        > dto에 `@EmailUnique` private `String` email; 선언으로 해당 로직을 처리할 수 있음.
        > dto 자체에 벨리데이션이 필요한 경우 클래스에 커스텀 어노테이션을 추가하면 된다.
        > 여러 필드를 검사해야 할 땐 어노테이션을 새로 추가하는 것이 일반적인 방법이다.
        > @Constraint()에 대응하는 클래스로 검증로직이 false를 반환하면 예외를 throw한다.
        > 컨트롤러 어드바이스에서 해당 익셉션을 @ExceptionHandler(Exception.class)로 받아 처리할 수 있다.

### ch6. 좋은 객체 디자인
## ✔ 객체의 적절한 크기를 찾는 여정
- 책임은 명확해야한다.
    - 서비스 메서드나 클래스는 **무엇을 하는지, 왜 존재하는지**가 명확해야 한다.
    - 메서드나 클래스에 의도와 규칙이 드러나도록 설계하면 책임이 명확해지고 혼동이 사라질 수 있다.
- 책임이 명확하면 대체가 가능하다.
    ```java
    Member updateName(Long id, String name); // 대체성이 없고 책임이 명확하지 않다.
    void changePassword(PasswordChangeRequest dto); // 비밀번호를 바꾼다라는 책임이 명확하다.
    ```
- updateName는 왜 책임이 명확하지 않고 changePassword는 책임이 명확하다는 것일까?
    - updateName은 `값을 바꾸는 API` 정도로 인식되고, 책임이나 절차를 기대하지 않는다.
    - changePassword는 도메인 특성상, 보안과 인증이 자동으로 연상되어 `관습화된 책임/규칙(예: 본인 인증, 정책 준수, 이력 관리 등)`이 따라붙는다.
    - 따라서, 특정 메서드를 보고 **그 행위에 어떤 책임과 절차가 포함되어야 하는지를 연상할 수 있는가의 여부**가, 책임의 명확성을 좌우한다.
    - `책임이 명확하다`란?
        - 단순히 **무엇을 한다**는 수준을 넘어서, 그 행위에 포함된 맥락, 규칙, 절차, 연관 기능들까지 구체적으로 도출 가능해야 한다는 의미이다.
    - 즉, `changePassword`는 그 자체로 책임의 경계와 맥락을 유추하기 쉬운 책임이 명확한 반면, `updateName`은 행위가 단순하다고 해석되기 때문에 책임이 명확하지 않다.
- 대체성이 없다면 굳이 인터페이스로 둘 필요가 없다.
    - 단일 구현만 존재하고 확장 가능성이 낮다면 인터페이스는 오버엔지니어링.
- 서비스는 `행위 중심`으로 설계해야 한다.
    - MemberService -> Member All... 모든 기능이 혼합되어 있다.
    - MemberSignService -> 회원가입에 대한 책임만 명확히 분리됨.
      
## ✔ 객체는 협력 관계를 유지해야 한다.
- 객체는 **혼자 모든 일을 처리하지 않는다.**
- 각 객체는 **자신의 책임만 수행**하고, **필요한 작업은 다른 객체와 협력**하여 완성한다.
- 즉, 객체는 **역할을 분리**하고 **유기적인 협력 구조**를 갖는 것이 이상적이다.
- ✅ 왜 중요한가?

| 이유           | 설명                                                                 |
|----------------|----------------------------------------------------------------------|
| 역할 분리       | 각 객체가 책임을 잘게 나누어 유연하고 변경에 강한 구조가 됨             |
| 응집도 ↑, 결합도 ↓ | 협력을 통해 느슨하게 연결되어 재사용성과 유지보수성이 향상됨             |
| 확장성 확보     | 새로운 기능이 필요한 경우 객체 하나만 추가하거나 교체하면 됨             |

- ✅ 예시
    - ❌ 나쁜 설계: 하나의 객체가 모든 책임을 가짐
    ```java
    class OrderService {
        void createOrder() {
            // 결제 처리
            // 재고 차감
            // 할인 계산
            // 알림 발송
        }
    }
    ```
    - ✅ 좋은 설계: 객체 간 협력 구조
    ```java
    class OrderService {
        private PaymentProcessor paymentProcessor;
        private InventoryService inventoryService;
    
        void createOrder() {
            paymentProcessor.process();
            inventoryService.reduceStock();
        }
    }
    ```
    - `OrderService`: 조율자 역할
    - `PaymentProcessor`, `InventoryService`: 각자의 책임 수행
    - 객체 간 협력 구조로 변경에 강한 설계

## ✔ 묻지말고 시켜라
### 🧭 핵심 원칙
- 객체는 **복종**하는 존재가 아니라, **자율적인 협력자**여야 한다.
- 객체 간에는 **정보를 묻기보다**, **책임을 시켜야** 한다.

### 📌 왜 중요한가?
- 객체의 내부 상태를 외부에서 묻는 방식은 유연하지 못하다.
- 다양한 조건이나 상태가 생기면 **중복 코드**, **조건 분기 증가**, **유지보수성 저하** 발생.
- 반면 객체에게 책임을 부여하면 **캡슐화 유지**, **응집도 향상**, **책임 중심 설계 가능**.

### 🔍 안티패턴 예시: 조건을 묻고 판단
```java
public void apply(final long couponId) {
    final CouponLegacy coupon = getCoupon(couponId);

    if (LocalDate.now().isAfter(coupon.getExpirationDate())) {
        throw new IllegalStateException("사용 기간이 만료된 쿠폰입니다.");
    }
    if (coupon.isUsed()) {
        throw new IllegalStateException("이미 사용한 쿠폰입니다.");
    }
}
```
- 외부에서 쿠폰 상태를 일일이 묻고 있다.
- 상태 기준 조건 로직이 중복되거나, 조건 변경 시 유지보수가 어려움.
  
### ✅ 개선된 설계: 쿠폰에 책임을 위임
``` java
public void apply(final long couponId) {
    final Coupon coupon = getCoupon(couponId);
    coupon.apply(); // 쿠폰 객체 내부에서 모든 유효성 처리
}
```
- 묻지 않고 시킨다.
- 쿠폰 객체가 스스로 `apply()` 책임을 수행.
- 비즈니스 규칙과 상태 관리를 도메인 객체 내부에 캡슐화.

### 정리
- 좋은 객체지향 설계란 객체에게 충분한 책임을 부여하고, 그 책임을 직접 수행하게 하는 방식이다.
- `묻지 말고 시켜라`는 객체 간 `자율성과 협력의 원칙`을 잘 보여주는 설계 철학이다.
- `객체는 협력 관계를 유지해야 한다.`파트와 연관지어 이해하면 좋을 것 같다.

### ch7. 시스템 내 강결합 문제 해결
## ApplicationEventPublisher를 이용한 시스템 내의 강결합 문제 해결
### 🔧 기존 문제점
```java
@Service
@RequiredArgsConstructor
public class MemberSignUpService {

    private final MemberRepository memberRepository;
    private final CouponIssueService couponIssueService;
    private final EmailSenderService emailSenderService;

    @Transactional
    public void signUp(final MemberSignUpRequest dto) {
        final Member member = memberRepository.save(dto.toEntity());
        emailSenderService.sendSignUpEmail(member); // 외부 호출
        couponIssueService.issueSignUpCoupon(member.getId()); // 예외 발생 시 전체 롤백
    }
}
```
- 🔴 문제 요약
    - 회원가입, 이메일 전송, 쿠폰 발급이 하나의 서비스에 모두 의존 → 강한 결합
    - 트랜잭션 내에서 외부 시스템 호출 → 예외 발생 시 롤백 불일치 위험
    (ex: 이메일은 전송됐지만 DB는 롤백)

### 🧩 해결 전략: 이벤트 기반 아키텍처 도입
### 1. 이벤트발행 구조로 리팩토링
- 회원가입 후 이벤트 발행 → 리스너가 이메일 전송 / 쿠폰 발급 담당      
```java
@Transactional
public void signUp(final MemberSignUpRequest dto) {
    final Member member = memberRepository.save(dto.toEntity());
    eventPublisher.publishEvent(new MemberSignedUpEvent(member));
}
```

### 2. 리스너 등록 (@TransactionalEventListener)
```java
@Component
@RequiredArgsConstructor
public class MemberEventHandler {

    private final EmailSenderService emailSenderService;

    @TransactionalEventListener
    public void memberSignedUpEventListener(MemberSignedUpEvent event) {
        emailSenderService.sendSignUpEmail(event.getMember());
    }
}
```
- `@TransactionalEventListener` 사용 시 **트랜잭션이 Commit된 후 실행됨**
- 트랜잭션 롤백 시 외부 호출도 실행되지 않음 -> **데이터 정합성 보장**

### 3. 비동기처리 (@Async + @EventListener)
```java
@Component
@RequiredArgsConstructor
public class OrderEventHandler {

    private final CartService cartService;

    @Async
    @EventListener
    public void orderCompletedEventListener(OrderCompletedEvent event) {
        cartService.deleteCart(event.getOrder());
    }
}
```
- `@Async` 사용 시 별도 쓰레드에서 실행 → 성능 최적화 / 트랜잭션 분리되어 비동기 처리


### ch8. 테스팅 방법 및 전략(JUnit5)
## ✅ 테스트 인스턴스 생성 전략
- JUnit5에서는 **각 테스트 메서드마다 새로운 인스턴스가 생성**됨
  - → 테스트 간 **상호 영향 방지**
- 클래스당 한번만 인스턴스 생성 방법
```java
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
```
  
## ✅ 테스트 메서드 실행 순서 지정
- 클래스 레벨에서 `@TestMethodOrder` 선언
```java
@TestMethodOrder(MethodOrder.OrderAnnotation.class)
```
- 각 테스트에 실행 순서 지정
```java
@Order(1)
@Test
void testFirst() {}

@Order(2)
@Test
void testSecond() {}
```

## ✅ 전처리 / 후처리 어노테이션
- @BeforeAll: 테스트 전체 시작 전 1회 / static 메서드 필요 (인스턴스 생성 전에 실행되므로 static 필요)
- @BeforeEach: 각 테스트 메서드 실행 전	
- @AfterEach: 각 테스트 메서드 실행 후	
- @AfterAll: 테스트 전체 종료 후 1회	/ static 메서드 필요 (실행 시점은 테스트 후지만, 어떤 인스턴스를 쓸지 정해져 있지 않기 때문)

## ✅ @ParameterizedTest (매개변수 기반 테스트)
- 주요 어노테이션
    - @ValueSource	기본 타입 값 배열 전달
    - @EnumSource	Enum 상수 전달
    - @CsvSource	CSV 형태 문자열로 다중 파라미터 전달
    - @MethodSource	메서드에서 파라미터 목록을 반환
- 하나의 테스트 메서드를 입력 된 값의 크기만큼 반복 실행할 수 있게 해주는 JUnit5 기능이다.
```java
@ParameterizedTest
@ValueSource(strings = {"apple", "banana"})
void testFruits(String fruit) {
    assertNotNull(fruit);
}
```

## AssertJ 사용 예시
```java
import static org.assertj.core.api.Assertions.*;

@Test
void testString() {
    String name = "chatgpt";
    assertThat(name)
        .isNotNull()
        .startsWith("chat")
        .endsWith("gpt")
        .hasSize(7);
}
```
> 체이닝 방법을 사용하여 테스트 할 수 있다.<br>
> 정확한 실패 메시지 제공과 풍부한 조건 메서드를 제공하여 테스트가 용이하다.

## ✅ 스프링의 다양한 테스트 종류
- 통합테스트
- 단위 테스트, JPA, MVC(Slice Test)
- 단위 테스트, Service Mock 테스트
- POJO, 도메인 테스트
    - 포조 테스트가 쉽고 간단하다. 스프링 빈과 별개로 테스트 할 수 있어서 속도도 좋다.
    - 도메인, 유틸 클래서, xxxService는 포조로 테스트를 해보자.
 
## API Test 개선
### 🔧 문제점
- 테스트 시 JSON 문자열을 직접 코드에 작성해야 하므로 관리가 불편함
- 테스트용 객체 생성을 위해 별도 DTO나 생성 코드 작성이 필요함
  - 이는 실제 서비스 코드에는 불필요한 테스트 전용 코드 증가로 이어짐

### ✅ 개선 방안
1. `.json` 파일을 `resources` 디렉토리 하위에 별도 저장
    - 예: `src/test/resources/json/member-request.json`
2. 테스트 코드에서 `readJson()` 등의 유틸 메서드를 통해 해당 JSON 파일을 로딩하여 사용
3. 테스트용 객체 생성을 방지하기 위해 해당 클래스 생성자에 다음 설정 적용:
    ```java
    @NoArgsConstructor(access = AccessLevel.PRIVATE)
    ```
    - 외부에서 객체 생성을 막아 테스트 코드용 불필요한 생성 차단
    - 테스트 코드에서는 ObjectMapper.readValue() 내부에서 리플렉션(reflection)을 통해 강제로 생성자에 접근
    - ❓리플렉션이란
        - 컴파일된 클래스에 대해 권한이 없이도 런타임에 구조를 분석하고 조작할 수 있게 하는 기능
    - 🎯주요기능
      - 클래스 이름으로 클래스 로딩
      - private 생성자/필드/메서드에도 접근 가능 (setAccessible(true))
      - 객체 없이도 클래스 정보 접근 가능
      - 동적으로 메서드 호출, 필드 값 읽기/쓰기 가능
        
### 💡 기대 효과
- 테스트 JSON 관리가 파일 단위로 분리되어 가독성과 재사용성 증가
- 테스트 목적의 불필요한 객체/코드 생성을 예방
- 전체 테스트 코드의 명확성과 유지보수성 향상
