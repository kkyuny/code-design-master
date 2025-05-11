### ch3. 객체를 풍부하게 표현하기
## ✔ Lombok을 잘 사용해야 객체 디자인을 망치치 않는다.
- @Data는 지양하자
    - @Setter가 무분별하게 남용된다.
  ```
  @Column(name = "email, nullable = false, updatable = false)
  private String email;
  -> 위와 같이 변경이 되어선 안되는 코드에 Setter 설정은 적절하지 않다.
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

### ch5. API Server Error 처리
## ✔ 통일된 Error Response를 갖자.
- ErrorResponse 객체
```
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
```
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
    ```
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
    ```
    @ExceptionHandler(BusinessException.class)
    protected ResponseEntity<ErrorResponse> handleBusinessException(final BusinessException e) {
        log.error("handleEntityNotFoundException", e);
        final ErrorCode errorCode = e.getErrorCode();
        final ErrorResponse response = ErrorResponse.of(errorCode);
        return new ResponseEntity<>(response, HttpStatus.valueOf(errorCode.getStatus()));
    }
    ```
    - 비즈니스 익셉션
    ```
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
      ```
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
      ```
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
      - **@Retention**: 런타임에도 유지되어야 검증 가능    
      - **message / groups / payload**: Bean Validation 표준 규약 필수 요소
      - **@interface**: 자바에서 어노테이션을 정의할 때 쓰는 문법
      - **@인터페이스인 이유**: 어노테이션은 필드나 메서드 위에 붙여서 메타데이터를 알려주는 도구이기 때문에 정적 구조(=interface)로 정의
    - 어노테이션 검증 로직
      ```
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

### ch6. 좋은 객체 디자인
## ✔ 객체의 적절한 크기를 찾는 여정
- 책임은 명확해야한다.
    - 서비스 메서드나 클래스는 **무엇을 하는지, 왜 존재하는지**가 명확해야 한다.
- 책임이 명확하면 대체가 가능하다.
    ```
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
    ```
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
