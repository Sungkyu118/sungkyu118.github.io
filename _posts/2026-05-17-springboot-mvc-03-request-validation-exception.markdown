---
layout: post
title: "SpringBoot 입문 3: 요청 받기(@RequestBody) + Validation + 예외 처리"
date: 2026-05-17 00:30:00 +0900
category: SpringBoot
permalink: /springboot/mvc-request-validation-exception
---

# SpringBoot 입문 3: 요청 받기(@RequestBody) + Validation + 예외 처리

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

## 다음 글 아이디어

여기까지 하면 "API 개발을 시작할 수 있는 최소 체력"이 생깁니다.

다음으로는 보통 이 순서로 갑니다.

- DB 연결(JPA + H2/PostgreSQL)
- 서비스 계층 분리(Service/Repository)
- 테스트(JUnit + MockMvc)

