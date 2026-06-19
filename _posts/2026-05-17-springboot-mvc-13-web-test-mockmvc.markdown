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

