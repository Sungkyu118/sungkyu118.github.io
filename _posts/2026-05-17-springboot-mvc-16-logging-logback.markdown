---
layout: post
title: "SpringBoot 입문 16: 로깅 기본(SLF4J)과 logback 설정"
date: 2026-05-17 02:40:00 +0900
category: SpringBoot
permalink: /springboot/mvc-logging-logback
---

# SpringBoot 입문 16: 로깅 기본(SLF4J)과 logback 설정

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

