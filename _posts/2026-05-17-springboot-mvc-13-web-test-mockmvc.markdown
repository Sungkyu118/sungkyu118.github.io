---
layout: post
title: "SpringBoot 입문 13: MockMvc로 Controller 테스트하기(요청/응답 검증)"
date: 2026-05-17 02:10:00 +0900
category: SpringBoot
permalink: /springboot/mvc-web-test-mockmvc
description: "Spring Boot Controller를 MockMvc로 테스트하는 방법, 요청과 응답 검증 포인트, 웹 레이어 테스트를 시작할 때 필요한 기본 구성을 설명합니다."
image: "/assets/img/og/springboot-series-cover.svg"
---


# SpringBoot 입문 13: MockMvc로 Controller 테스트하기(요청/응답 검증)

> 이 글에서는 Controller 레이어를 MockMvc로 검증하면서 요청과 응답 테스트 감각을 익힙니다.
>
> 이전 글: [SpringBoot 입문 12: 단위 테스트(JUnit5 + Mockito)로 Service 검증하기](/springboot/mvc-unit-test-junit-mockito)
> 다음 글: [SpringBoot 입문 14: 통합 테스트(@SpringBootTest)로 전체 흐름 확인하기](/springboot/mvc-integration-test-springboottest)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 11: 트랜잭션(@Transactional) 기본과 흔한 오해](/springboot/mvc-transaction-basics)
> - [SpringBoot 입문 15: 예외 처리 고급(에러코드, NotFound, 공통 응답 포맷)](/springboot/mvc-exception-handling-advanced)


Controller 테스트는 "HTTP 요청/응답"을 코드로 고정하는 작업입니다. MockMvc는 Spring MVC에서 가장 흔한 선택이에요.

목표:

- `GET`/`POST` 요청 테스트
- JSON 응답 검증
- Validation 실패 케이스 검증

## 1) 기본 세팅

```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@AutoConfigureMockMvc
class HelloControllerTest {

  @Autowired MockMvc mvc;

  @Test
  void hello_returns_text() throws Exception {
    mvc.perform(get("/hello"))
        .andExpect(status().isOk())
        .andExpect(content().string("hello spring"));
  }
}
```

## 2) JSON 응답 검증

`/hello-json`이 `{"message":"hello json"}`를 준다고 할 때:

```java
@Test
void hello_json_returns_message() throws Exception {
  mvc.perform(get("/hello-json"))
      .andExpect(status().isOk())
      .andExpect(jsonPath("$.message").value("hello json"));
}
```

## 3) Validation 실패 케이스

`POST /users`에서 name이 비어있으면 400이 나와야 합니다.

```java
@Test
void create_user_validation_fails() throws Exception {
  mvc.perform(post("/users")
          .contentType("application/json")
          .content("{\"name\":\"\"}"))
      .andExpect(status().isBadRequest());
}
```

## 정리

- MockMvc는 MVC에서 컨트롤러 요청/응답을 고정하는 표준 도구
- 정상/실패 케이스를 모두 테스트하면 API가 흔들리지 않는다

<!-- codex-springboot-detail:start -->

## 여기서 한 단계 더 이해해보겠습니다

Controller 테스트는 브라우저를 직접 눌러보는 대신, 요청과 응답이 기대대로 오가는지 자동으로 검증할 수 있게 해줍니다. 특히 상태코드, JSON 구조, Validation 실패 응답처럼 사람이 매번 눈으로 확인하기 귀찮은 부분을 안정적으로 잡아낼 수 있어서, 웹 계층 품질을 유지하는 데 큰 도움이 됩니다.

MockMvc는 톰캣을 실제로 띄우지 않아도 Controller를 호출한 것처럼 테스트할 수 있게 해줍니다. 그래서 단위 테스트보다 조금 더 웹에 가깝고, 통합 테스트보다 가볍다는 중간 지점에 있습니다. 입문 단계에서는 이 위치를 이해하시면 테스트 선택 기준이 훨씬 명확해집니다.

이 주제에서 특히 중요하게 보셔야 하는 부분은 HTTP 요청을 코드로 만들고, 상태코드와 JSON 응답 구조를 `jsonPath`로 검증하는 방식 입니다. 처음 읽으실 때는 모든 문장을 외우려고 하기보다, 코드가 어느 계층에 놓이고 어떤 순서로 실행되는지를 천천히 따라가보시면 좋습니다. Spring Boot 학습은 각각의 기술을 따로 외우는 공부라기보다, 여러 조각이 실제 요청 흐름 안에서 어떻게 맞물리는지 이해하는 과정에 가깝습니다.

## 예시 코드로 흐름을 잡아보겠습니다

```java
@WebMvcTest(UserController.class)
class UserControllerTest {

  @Autowired
  private MockMvc mockMvc;

  @MockBean
  private UserService userService;

  @Test
  void create_shouldReturn201() throws Exception {
    given(userService.create("sungkyu"))
        .willReturn(new UserResponse(1L, "sungkyu"));

    mockMvc.perform(post("/users")
            .contentType(MediaType.APPLICATION_JSON)
            .content("{\"name\":\"sungkyu\"}"))
        .andExpect(status().isCreated())
        .andExpect(jsonPath("$.id").value(1L))
        .andExpect(jsonPath("$.name").value("sungkyu"));
  }
}
```

위 예시는 MockMvc 테스트을 가장 작게 체험할 수 있는 흐름으로 구성했습니다. 처음에는 코드 한 줄 한 줄의 문법보다, 선언 위치와 실행 시점을 중심으로 읽어보시는 편이 훨씬 이해가 잘 되실 것입니다. 예를 들어 설정 클래스인지, Controller인지, Service인지에 따라 책임이 달라지고, 그 책임이 곧 Spring Boot 구조를 읽는 기준이 됩니다.

또 하나 중요한 점은 '지금 당장 돌아가는 코드'와 '나중에도 유지보수하기 쉬운 코드'는 조금 다를 수 있다는 사실입니다. 그래서 예제를 실행해보신 뒤에는 변수명, 메서드명, 경로, 설정 값을 직접 한두 번 바꿔보시면서 어떤 부분이 고정 규칙이고 어떤 부분이 프로젝트 상황에 따라 달라지는지 구분해보시면 정말 큰 도움이 됩니다.

## 자주 하는 실수와 에러도 같이 보겠습니다

- `@WebMvcTest`는 Service 빈을 전부 올리지 않기 때문에, 필요한 의존성을 `@MockBean`으로 넣지 않으면 컨텍스트 로딩 자체가 실패할 수 있습니다.
- JSON 요청인데 contentType을 빼먹으면 415 Unsupported Media Type 같은 예기치 않은 오류를 만날 수 있습니다.
- 응답 전체 문자열을 통째로 비교하면 포맷 변화에 너무 민감해져서 유지보수가 어려워질 수 있습니다.

이런 실수들은 대부분 개념을 몰라서라기보다, 실행 순서나 설정 위치를 한 번에 다 보지 못해서 생깁니다. 그래서 에러가 나면 조급하게 코드를 많이 바꾸기보다, 로그와 요청 흐름을 기준으로 '어디까지는 정상이고 어디서부터 어긋났는지'를 차분히 확인해보시는 편이 좋습니다.

## 실무에서는 이렇게 적용하시면 좋습니다

- 상태코드와 핵심 필드만 `jsonPath`로 검증하는 습관을 들이시면 테스트가 훨씬 안정적입니다.
- Validation 실패 케이스를 하나 넣어보시면 Controller 테스트의 가치가 바로 체감되실 것입니다.
- MockMvc는 웹 계층에 집중하는 도구이므로, 서비스 로직 검증은 별도의 단위 테스트로 분리해두시는 편이 좋습니다.

처음부터 완벽한 구조를 만들려고 하실 필요는 없습니다. 다만 작은 예제라도 책임 분리, 예외 처리, 설정 관리, 로그 확인 같은 기본 습관을 함께 가져가시면 나중에 프로젝트가 커졌을 때 훨씬 덜 흔들리게 됩니다. 특히 Spring Boot는 편의 기능이 많아서 '왜 이렇게 동작하는지'를 놓치기 쉬우므로, 동작 원리를 설명할 수 있을 정도로 천천히 정리해두시는 것이 좋습니다.


<!-- codex-springboot-inline-links:start -->

지금 읽고 계신 주제가 아직 조금 추상적으로 느껴지신다면 [SpringBoot 입문 2: 첫 Controller 만들기 + Debug로 흐름 이해하기](/springboot/mvc-hello-controller-debug), [SpringBoot 입문 3: 요청 받기(@RequestBody) + Validation + 예외 처리](/springboot/mvc-request-validation-exception), [SpringBoot 입문 14: 통합 테스트(@SpringBootTest)로 전체 흐름 확인하기](/springboot/mvc-integration-test-springboottest) 글도 함께 읽어보시면 좋습니다. 앞단의 준비 과정과 다음 단계의 확장 흐름이 자연스럽게 이어져서, 지금 배우는 내용이 프로젝트 안에서 어디에 연결되는지 더 분명하게 감을 잡으실 수 있습니다.

<!-- codex-springboot-inline-links:end -->
## 마무리 정리

SpringBoot 입문 13: MockMvc로 Controller 테스트하기(요청/응답 검증) 글을 공부하실 때는 한 번 읽고 끝내기보다, 직접 실행해보고 일부러 에러도 만들어보시고, 설정이나 코드 값을 조금씩 바꿔보시는 방식으로 접근해보시길 권합니다. 그렇게 하시면 단순 요약 글보다 훨씬 오래 기억에 남고, 다음 주제인 데이터 접근, 보안, 테스트, 운영 자동화로 넘어갈 때도 이해가 자연스럽게 이어집니다. 지금 단계에서 가장 중요한 것은 완벽함보다 흐름을 잡는 것이니, 천천히 따라오셔도 충분합니다.

<!-- codex-springboot-detail:end -->

<!-- codex-springboot-links:start -->

## 이어서 읽어보시면 좋습니다

- [SpringBoot 입문 2: 첫 Controller 만들기 + Debug로 흐름 이해하기](/springboot/mvc-hello-controller-debug)
- [SpringBoot 입문 3: 요청 받기(@RequestBody) + Validation + 예외 처리](/springboot/mvc-request-validation-exception)
- [SpringBoot 입문 14: 통합 테스트(@SpringBootTest)로 전체 흐름 확인하기](/springboot/mvc-integration-test-springboottest)

지금 글과 연결되는 흐름으로 골라두었으니, 바로 다음 실습이나 비교 학습을 이어가실 때 참고해보시면 좋겠습니다.

<!-- codex-springboot-links:end -->
