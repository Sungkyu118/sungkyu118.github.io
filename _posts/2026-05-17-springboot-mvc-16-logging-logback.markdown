---
layout: post
title: "SpringBoot 입문 16: 로깅 기본(SLF4J)과 logback 설정"
date: 2026-05-17 02:40:00 +0900
category: SpringBoot
permalink: /springboot/mvc-logging-logback
description: "Spring Boot에서 SLF4J와 logback으로 로그를 남기는 기본 방법, 로그 레벨 설정, 민감정보 로그 주의사항까지 함께 정리합니다."
image: "/assets/img/og/springboot-series-cover.svg"
---

# SpringBoot 입문 16: 로깅 기본(SLF4J)과 logback 설정

> 이 글에서는 디버깅에 도움 되는 로그를 어떻게 남기고, 어떤 정보는 남기면 안 되는지 정리합니다.
>
> 이전 글: [SpringBoot 입문 15: 예외 처리 고급(에러코드, NotFound, 공통 응답 포맷)](/springboot/mvc-exception-handling-advanced)
> 다음 글: [SpringBoot 입문 17: Actuator로 헬스체크/운영 지표 열기(주의점 포함)](/springboot/mvc-actuator-health-metrics)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 14: 통합 테스트(@SpringBootTest)로 전체 흐름 확인하기](/springboot/mvc-integration-test-springboottest)
> - [SpringBoot 입문 18: Swagger(OpenAPI) 문서 자동화(springdoc-openapi)](/springboot/mvc-openapi-swagger-springdoc)


로그는 "나중에 디버깅을 쉽게 만드는 보험"입니다. 초보자에게는 로그를 어떻게 찍어야 하는지, 무엇을 찍으면 안 되는지(민감정보)가 중요합니다.

## 1) SLF4J로 로그 찍기

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class LogController {
  private static final Logger log = LoggerFactory.getLogger(LogController.class);

  @GetMapping("/log-test")
  public String test() {
    log.info("log-test called");
    return "ok";
  }
}
```

## 2) 로그 레벨

- TRACE/DEBUG: 개발용
- INFO: 운영 기본
- WARN: 비정상 상황(하지만 서비스는 동작)
- ERROR: 에러

## 3) logback 설정 파일

Spring Boot 기본은 logback입니다.

파일:

- `src/main/resources/logback-spring.xml`

초보자 단계에서는 "패턴과 레벨"만 조절해도 충분합니다.

## 4) 절대 로그에 남기면 안 되는 것

- 비밀번호
- 액세스 토큰/리프레시 토큰
- 주민번호/결제정보 등 민감정보

## 정리

- SLF4J로 로그 찍고, 레벨을 의도에 맞게 사용
- 운영에서는 민감정보를 절대 남기지 않는다

