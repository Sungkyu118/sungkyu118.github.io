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

<!-- codex-springboot-detail:start -->

## 여기서 한 단계 더 이해해보겠습니다

로그는 개발 중 디버깅 도구이기도 하지만, 운영 중 장애를 파악하는 가장 중요한 단서이기도 합니다. 콘솔에 `System.out.println`을 찍는 수준에서 멈추지 않고, 로그 레벨과 포맷을 이해해두시면 실제 운영 상황에서 훨씬 빠르게 원인을 좁혀갈 수 있습니다.

특히 Spring Boot에서는 기본 로깅 체인이 이미 잘 갖춰져 있기 때문에, SLF4J 인터페이스와 logback 설정만 이해해도 대부분의 기본 작업을 처리하실 수 있습니다. 처음에는 "언제 info를 쓰고 언제 debug를 쓰는지"만 감이 와도 큰 도움이 됩니다.

이 주제에서 특히 중요하게 보셔야 하는 부분은 SLF4J 로거 사용법, 로그 레벨 구분, 그리고 logback 설정으로 출력 방식을 제어하는 흐름 입니다. 처음 읽으실 때는 모든 문장을 외우려고 하기보다, 코드가 어느 계층에 놓이고 어떤 순서로 실행되는지를 천천히 따라가보시면 좋습니다. Spring Boot 학습은 각각의 기술을 따로 외우는 공부라기보다, 여러 조각이 실제 요청 흐름 안에서 어떻게 맞물리는지 이해하는 과정에 가깝습니다.

## 예시 코드로 흐름을 잡아보겠습니다

```java
package com.example.demo.service;

import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

@Slf4j
@Service
public class UserLogService {

  public void createUser(String name) {
    log.info("create user request name={}", name);

    try {
      log.debug("validation start name={}", name);
      // business logic
      log.info("create user success name={}", name);
    } catch (Exception ex) {
      log.error("create user failed name={}", name, ex);
      throw ex;
    }
  }
}
```

위 예시는 로깅과 Logback을 가장 작게 체험할 수 있는 흐름으로 구성했습니다. 처음에는 코드 한 줄 한 줄의 문법보다, 선언 위치와 실행 시점을 중심으로 읽어보시는 편이 훨씬 이해가 잘 되실 것입니다. 예를 들어 설정 클래스인지, Controller인지, Service인지에 따라 책임이 달라지고, 그 책임이 곧 Spring Boot 구조를 읽는 기준이 됩니다.

또 하나 중요한 점은 '지금 당장 돌아가는 코드'와 '나중에도 유지보수하기 쉬운 코드'는 조금 다를 수 있다는 사실입니다. 그래서 예제를 실행해보신 뒤에는 변수명, 메서드명, 경로, 설정 값을 직접 한두 번 바꿔보시면서 어떤 부분이 고정 규칙이고 어떤 부분이 프로젝트 상황에 따라 달라지는지 구분해보시면 정말 큰 도움이 됩니다.

## 자주 하는 실수와 에러도 같이 보겠습니다

- 문자열 결합으로 로그를 만들면 debug 레벨이 꺼져 있어도 문자열이 미리 계산되어 성능상 불리할 수 있습니다.
- 비밀번호, 토큰, 주민번호 같은 민감 정보를 그대로 로그에 남기면 심각한 보안 문제가 될 수 있습니다.
- 예외가 났는데 `ex.getMessage()`만 남기고 스택 트레이스를 같이 넘기지 않으면 원인 추적이 어려워질 수 있습니다.

이런 실수들은 대부분 개념을 몰라서라기보다, 실행 순서나 설정 위치를 한 번에 다 보지 못해서 생깁니다. 그래서 에러가 나면 조급하게 코드를 많이 바꾸기보다, 로그와 요청 흐름을 기준으로 '어디까지는 정상이고 어디서부터 어긋났는지'를 차분히 확인해보시는 편이 좋습니다.

## 실무에서는 이렇게 적용하시면 좋습니다

- 정상 흐름의 큰 단계는 info, 더 세밀한 추적은 debug, 예외 상황은 error처럼 역할을 구분해보시는 것이 좋습니다.
- 로그는 많다고 좋은 것이 아니라, 나중에 문제를 찾을 수 있을 정도로 의미 있게 남는 것이 중요합니다.
- 실행 요청 단위로 requestId를 남기는 구조까지 가시면 운영 관찰성이 크게 좋아집니다.

처음부터 완벽한 구조를 만들려고 하실 필요는 없습니다. 다만 작은 예제라도 책임 분리, 예외 처리, 설정 관리, 로그 확인 같은 기본 습관을 함께 가져가시면 나중에 프로젝트가 커졌을 때 훨씬 덜 흔들리게 됩니다. 특히 Spring Boot는 편의 기능이 많아서 '왜 이렇게 동작하는지'를 놓치기 쉬우므로, 동작 원리를 설명할 수 있을 정도로 천천히 정리해두시는 것이 좋습니다.

## 마무리 정리

SpringBoot 입문 16: 로깅 기본(SLF4J)과 logback 설정 글을 공부하실 때는 한 번 읽고 끝내기보다, 직접 실행해보고 일부러 에러도 만들어보시고, 설정이나 코드 값을 조금씩 바꿔보시는 방식으로 접근해보시길 권합니다. 그렇게 하시면 단순 요약 글보다 훨씬 오래 기억에 남고, 다음 주제인 데이터 접근, 보안, 테스트, 운영 자동화로 넘어갈 때도 이해가 자연스럽게 이어집니다. 지금 단계에서 가장 중요한 것은 완벽함보다 흐름을 잡는 것이니, 천천히 따라오셔도 충분합니다.

<!-- codex-springboot-detail:end -->
