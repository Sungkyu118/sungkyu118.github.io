---
layout: post
title: "SpringBoot 입문 14: 통합 테스트(@SpringBootTest)로 전체 흐름 확인하기"
date: 2026-05-17 02:20:00 +0900
category: SpringBoot
permalink: /springboot/mvc-integration-test-springboottest
description: "Spring Boot에서 @SpringBootTest로 전체 흐름을 검증하는 통합 테스트 기본, 단위 테스트와의 차이, 느려지는 이유와 사용 기준을 정리합니다."
image: "/assets/img/og/springboot-series-cover.svg"
---

# SpringBoot 입문 14: 통합 테스트(@SpringBootTest)로 전체 흐름 확인하기

> 이 글에서는 애플리케이션을 실제로 띄우는 통합 테스트가 언제 필요한지와 비용을 함께 이해합니다.
>
> 이전 글: [SpringBoot 입문 13: MockMvc로 Controller 테스트하기(요청/응답 검증)](/springboot/mvc-web-test-mockmvc)
> 다음 글: [SpringBoot 입문 15: 예외 처리 고급(에러코드, NotFound, 공통 응답 포맷)](/springboot/mvc-exception-handling-advanced)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 12: 단위 테스트(JUnit5 + Mockito)로 Service 검증하기](/springboot/mvc-unit-test-junit-mockito)
> - [SpringBoot 입문 16: 로깅 기본(SLF4J)과 logback 설정](/springboot/mvc-logging-logback)


단위 테스트/컨트롤러 테스트가 "부분"을 검증한다면, 통합 테스트는 "전체가 연결되는지" 확인합니다.

초보자에게 추천하는 최소 통합 테스트 목표:

- 앱 컨텍스트가 뜨는지
- 주요 엔드포인트가 200을 내는지
- DB가 붙는 흐름이 동작하는지

## 1) 가장 기본적인 컨텍스트 테스트

```java
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;

@SpringBootTest
class ContextLoadsTest {
  @Test
  void contextLoads() {}
}
```

이게 깨지면 "설정/빈 구성" 문제가 있다는 뜻입니다.

## 2) TestRestTemplate로 실제 HTTP 호출(선택)

초보자 단계에서는 MockMvc가 더 흔하지만, 실제 포트로 띄우고 호출하는 테스트도 가능합니다.

## 3) 통합 테스트가 느려지는 이유

- Spring 컨텍스트를 띄우는 비용
- DB/외부 의존 연결

그래서 테스트 전략은 보통:

- 단위 테스트 많이(빠름)
- 통합 테스트는 핵심 흐름만(느림)

## 정리

- `@SpringBootTest`는 전체 연결 확인용
- 느리기 때문에 핵심 케이스만 넣는 게 실전 전략이다

