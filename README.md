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
- 오류 코드보다 예외를 사용하자.
    - 로직으로 예외 처리 가능하다면 `try-catch`는 지양한다.
    - `try-catch`를 사용한다면 예외를 구체적으로 작성하는 것이 좋다.
- Checked Exception VS UnChecked Exception
  - Checked Exception: 반드시 예외 처리해야함 / Rollback 안됨 / IOException, SQLException
  - UnChecked Exception: 예외 처리하지 않아도됨 / Rollback 진행 / NullPointerException, IllegalArgumentException
    > 체크드 익셉션은 롤백이 되지 않기 때문에 다른 곳으로 예외를 전파하는건 바람직하지 않다.    
    > 예외를 언체크드 익셉션으로 처리할 수 있다면 언체크드 익셉션으로 처리하는 것이 좋다.
