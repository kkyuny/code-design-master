### ch2. 객체를 풍부하게 표현하기
## 1. Lombok을 잘 사용해야 객체 디자인을 망치치 않는다.
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
        - config에 어긋나는 동작들은 빌드 시 fail되어 관리할 수 있다.
    - 클래스 상단의 @Builder는 지양하자.
        -     
