---
layout: post
title: "SpringBoot 입문 3: 요청 받기(@RequestBody) + Validation + 예외 처리"
date: 2026-05-17 00:30:00 +0900
category: SpringBoot
permalink: /springboot/mvc-request-validation-exception
description: "Spring Boot에서 @RequestBody와 Validation을 함께 사용하는 방법, 검증 실패 시 자주 만나는 에러, 예외 응답을 정리하는 흐름을 예제로 설명합니다."
image: "/assets/img/og/springboot-series-cover.svg"
---

# SpringBoot 입문 3: 요청 받기(@RequestBody) + Validation + 예외 처리

> 이 글에서는 요청 DTO를 받고 검증한 뒤, 실패를 일관된 에러 응답으로 바꾸는 기본 패턴을 익힙니다.
>
> 이전 글: [SpringBoot 입문 2: 첫 Controller 만들기 + Debug로 흐름 이해하기](/springboot/mvc-hello-controller-debug)
> 다음 글: [SpringBoot 입문 6: REST 기본(@RequestParam/@PathVariable)과 HTTP 상태코드](/springboot/mvc-rest-basics-params-status)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 15: 예외 처리 고급(에러코드, NotFound, 공통 응답 포맷)](/springboot/mvc-exception-handling-advanced)
> - [SpringBoot 입문 31: 공통 응답 포맷과 예외 코드 표준화](/springboot/mvc-global-response-format)

실전 API는 "요청을 받고 → 검증하고 → 실패는 일관되게 에러로 내려주기"가 기본입니다. 이 글은 이 흐름을 초보자도 그대로 복사해서 쓸 수 있게 만드는 게 목표예요.

## 1) 의존성 추가(Validation)

Gradle에 Validation 스타터를 추가합니다.

`build.gradle`:

```gradle
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-web'
  implementation 'org.springframework.boot:spring-boot-starter-validation'
}
```

Gradle sync가 끝나면 진행하세요.

## 2) 요청 DTO 만들기

```java
package com.sungkyu.demo;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;

public record CreateUserRequest(
    @NotBlank(message = "name is required")
    @Size(max = 20, message = "name must be <= 20")
    String name
) {}
```

## 3) Controller에서 @RequestBody 받기

```java
package com.sungkyu.demo;

import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

import jakarta.validation.Valid;

@RestController
public class UserController {

  @PostMapping("/users")
  public HelloResponse createUser(@Valid @RequestBody CreateUserRequest req) {
    return new HelloResponse("created: " + req.name());
  }
}
```

## 4) 테스트 호출(curl)

정상 요청:

```bash
curl -X POST http://localhost:8080/users ^
  -H "Content-Type: application/json" ^
  -d "{\"name\":\"sungkyu\"}"
```

검증 실패 요청(name 비움):

```bash
curl -X POST http://localhost:8080/users ^
  -H "Content-Type: application/json" ^
  -d "{\"name\":\"\"}"
```

이때 Spring 기본 에러 응답이 내려오는데, 초보자 입장에서는 이 포맷이 "프로젝트마다 다르게" 느껴질 수 있어요. 그래서 아래처럼 에러 포맷을 통일합니다.

## 5) 예외 포맷 통일(@RestControllerAdvice)

에러 응답 DTO:

```java
package com.sungkyu.demo;

import java.util.List;

public record ErrorResponse(
    String code,
    String message,
    List<FieldError> errors
) {
  public record FieldError(String field, String reason) {}
}
```

예외 처리:

```java
package com.sungkyu.demo;

import java.util.List;

import org.springframework.http.HttpStatus;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.bind.annotation.RestControllerAdvice;

@RestControllerAdvice
public class ApiExceptionHandler {

  @ResponseStatus(HttpStatus.BAD_REQUEST)
  @ExceptionHandler(MethodArgumentNotValidException.class)
  public ErrorResponse handleValidation(MethodArgumentNotValidException ex) {
    List<ErrorResponse.FieldError> errors =
        ex.getBindingResult().getFieldErrors().stream()
            .map(e -> new ErrorResponse.FieldError(e.getField(), e.getDefaultMessage()))
            .toList();

    return new ErrorResponse("VALIDATION_ERROR", "Invalid request", errors);
  }
}
```

이제 검증 실패 시 에러 포맷이 안정적으로 유지됩니다.

## 6) 자주 막히는 포인트

- `jakarta.validation` 패키지 import가 안 됨: validation dependency가 없는 상태일 가능성이 큼
- record가 안 됨: Java 버전이 낮거나, 프로젝트 SDK가 17이 아닌 상태일 수 있음


<!-- codex-springboot-inline-links:start -->

지금 읽고 계신 주제가 아직 조금 추상적으로 느껴지신다면 [SpringBoot 입문 2: 첫 Controller 만들기 + Debug로 흐름 이해하기](/springboot/mvc-hello-controller-debug), [SpringBoot 입문 6: REST 기본(@RequestParam/@PathVariable)과 HTTP 상태코드](/springboot/mvc-rest-basics-params-status), [SpringBoot 입문 15: 예외 처리 고급(에러코드, NotFound, 공통 응답 포맷)](/springboot/mvc-exception-handling-advanced) 글도 함께 읽어보시면 좋습니다. 앞단의 준비 과정과 다음 단계의 확장 흐름이 자연스럽게 이어져서, 지금 배우는 내용이 프로젝트 안에서 어디에 연결되는지 더 분명하게 감을 잡으실 수 있습니다.

<!-- codex-springboot-inline-links:end -->
## 다음 글 아이디어

여기까지 하면 "API 개발을 시작할 수 있는 최소 체력"이 생깁니다.

다음으로는 보통 이 순서로 갑니다.

- DB 연결(JPA + H2/PostgreSQL)
- 서비스 계층 분리(Service/Repository)
- 테스트(JUnit + MockMvc)

<!-- codex-springboot-links:start -->

## 이어서 읽어보시면 좋습니다

- [SpringBoot 입문 2: 첫 Controller 만들기 + Debug로 흐름 이해하기](/springboot/mvc-hello-controller-debug)
- [SpringBoot 입문 6: REST 기본(@RequestParam/@PathVariable)과 HTTP 상태코드](/springboot/mvc-rest-basics-params-status)
- [SpringBoot 입문 15: 예외 처리 고급(에러코드, NotFound, 공통 응답 포맷)](/springboot/mvc-exception-handling-advanced)

지금 글과 연결되는 흐름으로 골라두었으니, 바로 다음 실습이나 비교 학습을 이어가실 때 참고해보시면 좋겠습니다.

<!-- codex-springboot-links:end -->
