---
layout: post
title: "SpringBoot 입문 15: 예외 처리 고급(에러코드, NotFound, 공통 응답 포맷)"
date: 2026-05-17 02:30:00 +0900
category: SpringBoot
permalink: /springboot/mvc-exception-handling-advanced
---

# SpringBoot 입문 15: 예외 처리 고급(에러코드, NotFound, 공통 응답 포맷)

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

