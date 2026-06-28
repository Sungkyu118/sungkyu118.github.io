---
layout: post
title: "SpringBoot 입문 2: 첫 Controller 만들기 + Debug로 흐름 이해하기"
date: 2026-05-17 00:20:00 +0900
category: SpringBoot
permalink: /springboot/mvc-hello-controller-debug
description: "Spring Boot에서 첫 Controller를 만들고 /hello 요청을 처리하는 흐름을 브라우저 호출과 IntelliJ Debug 예제로 함께 확인합니다."
image: "/assets/img/og/springboot-series-cover.svg"
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

<!-- codex-springboot-detail:start -->

## 여기서 한 단계 더 이해해보겠습니다

Controller는 대부분의 Spring Boot 학습에서 가장 먼저 만나는 진입점입니다. 요청이 들어왔을 때 어떤 메서드가 선택되고, 파라미터가 어떤 값으로 바인딩되며, 리턴값이 최종 HTTP 응답으로 어떻게 바뀌는지 이해하면 이후 학습이 훨씬 쉬워집니다.

여기에 디버깅을 함께 익혀두시면 "왜 404가 나는지", "왜 값이 null인지", "왜 예상한 메서드가 호출되지 않는지"를 감으로 추측하지 않고 눈으로 확인할 수 있습니다. 특히 입문자분들은 브레이크포인트 한 번 잘 걸어보는 경험이 이후 문제 해결 능력을 크게 바꿔줍니다.

이 주제에서 특히 중요하게 보셔야 하는 부분은 요청이 Controller 메서드로 들어오고, 디버거에서 변수 값을 확인하고, 응답으로 나가는 흐름을 한 단계씩 따라가는 것 입니다. 처음 읽으실 때는 모든 문장을 외우려고 하기보다, 코드가 어느 계층에 놓이고 어떤 순서로 실행되는지를 천천히 따라가보시면 좋습니다. Spring Boot 학습은 각각의 기술을 따로 외우는 공부라기보다, 여러 조각이 실제 요청 흐름 안에서 어떻게 맞물리는지 이해하는 과정에 가깝습니다.

## 예시 코드로 흐름을 잡아보겠습니다

```java
package com.example.demo.web;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class DebugHelloController {

  @GetMapping("/hello/{name}")
  public String hello(@PathVariable String name) {
    String message = "안녕하세요, " + name + "님";
    return message;
  }
}
```

위 예시는 Controller와 디버깅을 가장 작게 체험할 수 있는 흐름으로 구성했습니다. 처음에는 코드 한 줄 한 줄의 문법보다, 선언 위치와 실행 시점을 중심으로 읽어보시는 편이 훨씬 이해가 잘 되실 것입니다. 예를 들어 설정 클래스인지, Controller인지, Service인지에 따라 책임이 달라지고, 그 책임이 곧 Spring Boot 구조를 읽는 기준이 됩니다.

또 하나 중요한 점은 '지금 당장 돌아가는 코드'와 '나중에도 유지보수하기 쉬운 코드'는 조금 다를 수 있다는 사실입니다. 그래서 예제를 실행해보신 뒤에는 변수명, 메서드명, 경로, 설정 값을 직접 한두 번 바꿔보시면서 어떤 부분이 고정 규칙이고 어떤 부분이 프로젝트 상황에 따라 달라지는지 구분해보시면 정말 큰 도움이 됩니다.

## 자주 하는 실수와 에러도 같이 보겠습니다

- 브레이크포인트가 회색으로 비활성화되어 있으면 디버그 모드가 아니라 일반 실행 모드로 켠 경우가 많습니다.
- URL은 `/hello/sungkyu`인데 메서드 매핑은 `/hello/{id}`처럼 다르게 적혀 있어도 동작은 되지만, 의미가 어긋나면 나중에 혼란이 커질 수 있습니다.
- 디버깅 중 애플리케이션을 강제로 멈추면 포트가 잠시 해제되지 않아 재실행 시 충돌이 날 수 있습니다.

이런 실수들은 대부분 개념을 몰라서라기보다, 실행 순서나 설정 위치를 한 번에 다 보지 못해서 생깁니다. 그래서 에러가 나면 조급하게 코드를 많이 바꾸기보다, 로그와 요청 흐름을 기준으로 '어디까지는 정상이고 어디서부터 어긋났는지'를 차분히 확인해보시는 편이 좋습니다.

## 실무에서는 이렇게 적용하시면 좋습니다

- 브레이크포인트는 return 직전에도 한 번 걸어보시고, 요청 파라미터가 실제로 어떤 값으로 들어오는지 꼭 확인해보시면 좋습니다.
- Step Over와 Step Into를 구분해서 써보시면 메서드 호출 흐름을 보는 감각이 빨리 생깁니다.
- 디버거 변수 창에서 값이 바뀌는 모습을 보는 경험은 Validation, Service, JPA 학습으로 이어질 때도 계속 도움이 됩니다.

처음부터 완벽한 구조를 만들려고 하실 필요는 없습니다. 다만 작은 예제라도 책임 분리, 예외 처리, 설정 관리, 로그 확인 같은 기본 습관을 함께 가져가시면 나중에 프로젝트가 커졌을 때 훨씬 덜 흔들리게 됩니다. 특히 Spring Boot는 편의 기능이 많아서 '왜 이렇게 동작하는지'를 놓치기 쉬우므로, 동작 원리를 설명할 수 있을 정도로 천천히 정리해두시는 것이 좋습니다.



<!-- codex-springboot-inline-links:start -->

지금 읽고 계신 주제가 아직 조금 추상적으로 느껴지신다면 [SpringBoot 입문 3: 요청 받기(@RequestBody) + Validation + 예외 처리](/springboot/mvc-request-validation-exception), [SpringBoot 입문 6: REST 기본(@RequestParam/@PathVariable)과 HTTP 상태코드](/springboot/mvc-rest-basics-params-status), [SpringBoot 입문 13: MockMvc로 Controller 테스트하기(요청/응답 검증)](/springboot/mvc-web-test-mockmvc) 글도 함께 읽어보시면 좋습니다. 앞단의 준비 과정과 다음 단계의 확장 흐름이 자연스럽게 이어져서, 지금 배우는 내용이 프로젝트 안에서 어디에 연결되는지 더 분명하게 감을 잡으실 수 있습니다.

<!-- codex-springboot-inline-links:end -->
## 마무리 정리
SpringBoot 입문 2: 첫 Controller 만들기 + Debug로 흐름 이해하기 글을 공부하실 때는 한 번 읽고 끝내기보다, 직접 실행해보고 일부러 에러도 만들어보시고, 설정이나 코드 값을 조금씩 바꿔보시는 방식으로 접근해보시길 권합니다. 그렇게 하시면 단순 요약 글보다 훨씬 오래 기억에 남고, 다음 주제인 데이터 접근, 보안, 테스트, 운영 자동화로 넘어갈 때도 이해가 자연스럽게 이어집니다. 지금 단계에서 가장 중요한 것은 완벽함보다 흐름을 잡는 것이니, 천천히 따라오셔도 충분합니다.

<!-- codex-springboot-detail:end -->

<!-- codex-springboot-links:start -->

## 이어서 읽어보시면 좋습니다

- [SpringBoot 입문 3: 요청 받기(@RequestBody) + Validation + 예외 처리](/springboot/mvc-request-validation-exception)
- [SpringBoot 입문 6: REST 기본(@RequestParam/@PathVariable)과 HTTP 상태코드](/springboot/mvc-rest-basics-params-status)
- [SpringBoot 입문 13: MockMvc로 Controller 테스트하기(요청/응답 검증)](/springboot/mvc-web-test-mockmvc)

지금 글과 연결되는 흐름으로 골라두었으니, 바로 다음 실습이나 비교 학습을 이어가실 때 참고해보시면 좋겠습니다.

<!-- codex-springboot-links:end -->
