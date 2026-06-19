---
layout: post
title: "SpringBoot 입문 2: 첫 Controller 만들기 + Debug로 흐름 이해하기"
date: 2026-05-17 00:20:00 +0900
category: SpringBoot
permalink: /springboot/mvc-hello-controller-debug
description: "Spring Boot에서 첫 Controller를 만들고 /hello 요청을 처리하는 흐름을 브라우저 호출과 IntelliJ Debug 예제로 함께 확인합니다."
image: "/assets/img/favicons/android-chrome-512x512.png"
---

# SpringBoot 입문 2: 첫 Controller 만들기 + Debug로 흐름 이해하기

> 이 글에서는 첫 번째 GET API를 만들고 요청이 Controller까지 도달하는 흐름을 눈으로 확인합니다.
>
> 이전 글: [SpringBoot 입문 1: IntelliJ + Java + Gradle로 프로젝트 생성부터 실행까지](/springboot/mvc-setup-intellij)
> 다음 글: [SpringBoot 입문 3: 요청 받기(@RequestBody) + Validation + 예외 처리](/springboot/mvc-request-validation-exception)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 6: REST 기본(@RequestParam/@PathVariable)과 HTTP 상태코드](/springboot/mvc-rest-basics-params-status)
> - [SpringBoot 입문 16: 로깅 기본(SLF4J)과 logback 설정](/springboot/mvc-logging-logback)

이 글에서는 Spring MVC의 가장 기본인 Controller를 만들고, 브라우저로 호출한 뒤, IntelliJ Debug로 내부 흐름을 확인합니다.

## 1) Hello API 만들기

`src/main/java/...` 아래에 컨트롤러를 만들고, 문자열을 반환해봅니다.

```java
package com.sungkyu.demo;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloController {

  @GetMapping("/hello")
  public String hello() {
    return "hello spring";
  }
}
```

`@RestController`는 "응답을 그대로 body로 내려준다"는 의미입니다(템플릿 렌더링이 아니라).

## 2) 실행 후 브라우저로 확인

- 서버 실행
- 브라우저에서 `http://localhost:8080/hello`

정상 응답:

```
hello spring
```

## 3) JSON 응답으로 바꿔보기(DTO)

문자열 대신 JSON을 내려보면 MVC 감이 더 빨리 옵니다.

```java
package com.sungkyu.demo;

public record HelloResponse(String message) {}
```

```java
@GetMapping("/hello-json")
public HelloResponse helloJson() {
  return new HelloResponse("hello json");
}
```

이제 `GET /hello-json`은 JSON으로 응답합니다.

## 4) Debug로 한 줄씩 따라가기

디버그를 하면 "요청이 컨트롤러로 어떻게 들어오는지" 감이 잡힙니다.

1. `helloJson()` 메서드에 브레이크포인트 찍기
2. IntelliJ에서 Debug 실행(벌레 아이콘)
3. 브라우저에서 `/hello-json` 호출

예시 이미지:

![IntelliJ Debug](/assets/img/intellij/spring_debug_breakpoint.svg)

## 5) 자주 하는 실수

### (1) 클래스가 스캔되지 않음

컨트롤러를 메인 클래스 패키지 바깥에 만들면 컴포넌트 스캔에서 빠질 수 있어요.

규칙:

- `@SpringBootApplication`이 있는 패키지 하위에 컨트롤러를 두는 게 가장 안전합니다.

### (2) 404가 뜸

- URL 오타
- 서버가 다른 포트로 떴음
- 컨트롤러가 스캔되지 않음

## 6) 다음 단계 예고

다음 글에서는 초보자들이 가장 많이 쓰는 조합을 한 번에 잡습니다.

- 입력값 받기(@RequestParam, @PathVariable, @RequestBody)
- Validation으로 검증하기
- 예외 처리(@RestControllerAdvice)

