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
- Checked Exception: 반드시 예외 처리해야함 / Rollback 안됨 / IOException, SQLException
- UnChecked Exception: 예외 처리하지 않아도됨 / Rollback 진행 / NullPointerException, IllegalArgumentException
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
    - 스프링은 @ControllerAdvice을 통해 일관성 있게 처리할 수 있다.
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
- 비즈니스 로직을 수행하는 코드에서 예외가 발생하는 경우 최상위 Business Exception을 생성(exception 패키지 별도 생성)하여 처리하면 통일감 있는 예외처리를 할 수 있다.
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

