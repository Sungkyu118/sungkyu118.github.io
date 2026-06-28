---
layout: post
title: "SpringBoot 입문 15: 예외 처리 고급(에러코드, NotFound, 공통 응답 포맷)"
date: 2026-05-17 02:30:00 +0900
category: SpringBoot
permalink: /springboot/mvc-exception-handling-advanced
description: "Spring Boot 예외 처리를 고도화하면서 에러 코드, NotFound 예외, 공통 응답 포맷을 어떻게 맞추는지 실전 API 기준으로 설명합니다."
image: "/assets/img/og/springboot-series-cover.svg"
---


# SpringBoot 입문 15: 예외 처리 고급(에러코드, NotFound, 공통 응답 포맷)

> 이 글에서는 예외 처리 정책을 한 단계 올려 공통 에러 응답과 비즈니스 예외 코드를 맞추는 방법을 다룹니다.
>
> 이전 글: [SpringBoot 입문 14: 통합 테스트(@SpringBootTest)로 전체 흐름 확인하기](/springboot/mvc-integration-test-springboottest)
> 다음 글: [SpringBoot 입문 16: 로깅 기본(SLF4J)과 logback 설정](/springboot/mvc-logging-logback)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 13: MockMvc로 Controller 테스트하기(요청/응답 검증)](/springboot/mvc-web-test-mockmvc)
> - [SpringBoot 입문 17: Actuator로 헬스체크/운영 지표 열기(주의점 포함)](/springboot/mvc-actuator-health-metrics)


API가 커질수록 중요한 건 "실패 응답의 일관성"입니다. 프론트/클라이언트가 실패를 처리할 수 있어야, UX가 망가지지 않습니다.

목표:

- 커스텀 예외를 정의하고
- 에러코드를 부여하고
- 공통 에러 응답 포맷으로 내려준다

## 1) 에러 응답 DTO

```java
import java.util.List;

public record ApiError(
    String code,
    String message,
    List<FieldError> errors
) {
  public record FieldError(String field, String reason) {}
}
```

## 2) 커스텀 예외

```java
public class ApiException extends RuntimeException {
  private final String code;

  public ApiException(String code, String message) {
    super(message);
    this.code = code;
  }

  public String getCode() { return code; }
}
```

404 예:

```java
public class NotFoundException extends ApiException {
  public NotFoundException(String message) {
    super("NOT_FOUND", message);
  }
}
```

## 3) @RestControllerAdvice로 공통 처리

```java
import java.util.List;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.*;

@RestControllerAdvice
public class GlobalExceptionHandler {

  @ResponseStatus(HttpStatus.BAD_REQUEST)
  @ExceptionHandler(MethodArgumentNotValidException.class)
  public ApiError handleValidation(MethodArgumentNotValidException ex) {
    List<ApiError.FieldError> errors =
        ex.getBindingResult().getFieldErrors().stream()
            .map(e -> new ApiError.FieldError(e.getField(), e.getDefaultMessage()))
            .toList();
    return new ApiError("VALIDATION_ERROR", "Invalid request", errors);
  }

  @ResponseStatus(HttpStatus.NOT_FOUND)
  @ExceptionHandler(NotFoundException.class)
  public ApiError handleNotFound(NotFoundException ex) {
    return new ApiError(ex.getCode(), ex.getMessage(), List.of());
  }

  @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
  @ExceptionHandler(Exception.class)
  public ApiError handleUnknown(Exception ex) {
    return new ApiError("INTERNAL_ERROR", "Unexpected error", List.of());
  }
}
```

## 4) 실전 팁

- 에러코드는 프론트가 케이스를 분기할 수 있게 "안정적"이어야 합니다.
- 500은 가능한 한 "원인을 숨기고" 내부 로깅/모니터링으로 추적합니다.

## 정리

- 실패 응답을 표준화하면 팀이 편해진다
- 예외는 코드/상태/메시지를 분리해서 관리하는 게 유지보수에 유리하다

<!-- codex-springboot-detail:start -->

## 여기서 한 단계 더 이해해보겠습니다

예외 처리는 단순히 try-catch를 많이 쓰는 문제가 아니라, 클라이언트가 이해할 수 있는 방식으로 실패를 일관되게 전달하는 문제입니다. 프로젝트가 커질수록 "이 상황은 404인지 400인지", "에러 코드는 어떤 이름으로 내려줄지", "로그에는 무엇을 남길지"가 정말 중요해집니다.

처음에는 예외가 발생하면 기본 에러 페이지나 기본 JSON이 내려와도 괜찮아 보일 수 있지만, 프론트엔드와 함께 작업하거나 모바일 앱과 연결되기 시작하면 에러 응답 형식을 표준화하지 않은 대가가 빠르게 커집니다. 그래서 이 주제는 초반에 확실히 잡아두시는 편이 좋습니다.

이 주제에서 특히 중요하게 보셔야 하는 부분은 커스텀 예외, 공통 에러 DTO, `@RestControllerAdvice`를 통해 실패 응답을 한 구조로 묶는 것 입니다. 처음 읽으실 때는 모든 문장을 외우려고 하기보다, 코드가 어느 계층에 놓이고 어떤 순서로 실행되는지를 천천히 따라가보시면 좋습니다. Spring Boot 학습은 각각의 기술을 따로 외우는 공부라기보다, 여러 조각이 실제 요청 흐름 안에서 어떻게 맞물리는지 이해하는 과정에 가깝습니다.

## 예시 코드로 흐름을 잡아보겠습니다

```java
public record ErrorResponse(
    String code,
    String message
) {
}
```
```java
@Getter
public class NotFoundException extends RuntimeException {
  private final String code;

  public NotFoundException(String code, String message) {
    super(message);
    this.code = code;
  }
}
```
```java
@RestControllerAdvice
public class GlobalExceptionHandler {

  @ExceptionHandler(NotFoundException.class)
  public ResponseEntity<ErrorResponse> handleNotFound(NotFoundException ex) {
    return ResponseEntity.status(HttpStatus.NOT_FOUND)
        .body(new ErrorResponse(ex.getCode(), ex.getMessage()));
  }
}
```

위 예시는 예외 처리 고급을 가장 작게 체험할 수 있는 흐름으로 구성했습니다. 처음에는 코드 한 줄 한 줄의 문법보다, 선언 위치와 실행 시점을 중심으로 읽어보시는 편이 훨씬 이해가 잘 되실 것입니다. 예를 들어 설정 클래스인지, Controller인지, Service인지에 따라 책임이 달라지고, 그 책임이 곧 Spring Boot 구조를 읽는 기준이 됩니다.

또 하나 중요한 점은 '지금 당장 돌아가는 코드'와 '나중에도 유지보수하기 쉬운 코드'는 조금 다를 수 있다는 사실입니다. 그래서 예제를 실행해보신 뒤에는 변수명, 메서드명, 경로, 설정 값을 직접 한두 번 바꿔보시면서 어떤 부분이 고정 규칙이고 어떤 부분이 프로젝트 상황에 따라 달라지는지 구분해보시면 정말 큰 도움이 됩니다.

## 자주 하는 실수와 에러도 같이 보겠습니다

- 모든 예외를 500으로만 내려주면 클라이언트가 입력 오류와 서버 장애를 구분할 수 없어 협업이 어려워집니다.
- 예외 메시지에 내부 SQL 정보나 클래스 이름을 그대로 노출하면 보안상 좋지 않을 수 있습니다.
- Controller마다 서로 다른 에러 포맷을 내려주기 시작하면 프론트엔드에서 분기 처리가 폭발적으로 늘어날 수 있습니다.

이런 실수들은 대부분 개념을 몰라서라기보다, 실행 순서나 설정 위치를 한 번에 다 보지 못해서 생깁니다. 그래서 에러가 나면 조급하게 코드를 많이 바꾸기보다, 로그와 요청 흐름을 기준으로 '어디까지는 정상이고 어디서부터 어긋났는지'를 차분히 확인해보시는 편이 좋습니다.

## 실무에서는 이렇게 적용하시면 좋습니다

- 비즈니스 예외는 직접 이름을 붙여 명확하게 표현하시고, 상태코드와 에러 코드를 함께 설계해보시는 것이 좋습니다.
- 클라이언트에 보여줄 메시지와 서버 로그에 남길 상세 원인은 분리해서 생각하시는 편이 안전합니다.
- 에러 응답 구조를 먼저 정해두면 이후 Validation, 인증, DB 예외를 붙일 때도 기준이 흔들리지 않습니다.

처음부터 완벽한 구조를 만들려고 하실 필요는 없습니다. 다만 작은 예제라도 책임 분리, 예외 처리, 설정 관리, 로그 확인 같은 기본 습관을 함께 가져가시면 나중에 프로젝트가 커졌을 때 훨씬 덜 흔들리게 됩니다. 특히 Spring Boot는 편의 기능이 많아서 '왜 이렇게 동작하는지'를 놓치기 쉬우므로, 동작 원리를 설명할 수 있을 정도로 천천히 정리해두시는 것이 좋습니다.


<!-- codex-springboot-inline-links:start -->

지금 읽고 계신 주제가 아직 조금 추상적으로 느껴지신다면 [SpringBoot 입문 3: 요청 받기(@RequestBody) + Validation + 예외 처리](/springboot/mvc-request-validation-exception), [SpringBoot 입문 18: Swagger(OpenAPI) 문서 자동화(springdoc-openapi)](/springboot/mvc-openapi-swagger-springdoc), [SpringBoot 입문 31: 공통 응답 포맷과 예외 코드 표준화](/springboot/mvc-global-response-format) 글도 함께 읽어보시면 좋습니다. 앞단의 준비 과정과 다음 단계의 확장 흐름이 자연스럽게 이어져서, 지금 배우는 내용이 프로젝트 안에서 어디에 연결되는지 더 분명하게 감을 잡으실 수 있습니다.

<!-- codex-springboot-inline-links:end -->
## 마무리 정리

SpringBoot 입문 15: 예외 처리 고급(에러코드, NotFound, 공통 응답 포맷) 글을 공부하실 때는 한 번 읽고 끝내기보다, 직접 실행해보고 일부러 에러도 만들어보시고, 설정이나 코드 값을 조금씩 바꿔보시는 방식으로 접근해보시길 권합니다. 그렇게 하시면 단순 요약 글보다 훨씬 오래 기억에 남고, 다음 주제인 데이터 접근, 보안, 테스트, 운영 자동화로 넘어갈 때도 이해가 자연스럽게 이어집니다. 지금 단계에서 가장 중요한 것은 완벽함보다 흐름을 잡는 것이니, 천천히 따라오셔도 충분합니다.

<!-- codex-springboot-detail:end -->

<!-- codex-springboot-links:start -->

## 이어서 읽어보시면 좋습니다

- [SpringBoot 입문 3: 요청 받기(@RequestBody) + Validation + 예외 처리](/springboot/mvc-request-validation-exception)
- [SpringBoot 입문 18: Swagger(OpenAPI) 문서 자동화(springdoc-openapi)](/springboot/mvc-openapi-swagger-springdoc)
- [SpringBoot 입문 31: 공통 응답 포맷과 예외 코드 표준화](/springboot/mvc-global-response-format)

지금 글과 연결되는 흐름으로 골라두었으니, 바로 다음 실습이나 비교 학습을 이어가실 때 참고해보시면 좋겠습니다.

<!-- codex-springboot-links:end -->
