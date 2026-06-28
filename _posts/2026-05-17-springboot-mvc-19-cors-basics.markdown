---
layout: post
title: "SpringBoot 입문 19: CORS 기본과 전역 설정(WebMvcConfigurer)"
date: 2026-05-17 03:10:00 +0900
category: SpringBoot
permalink: /springboot/mvc-cors-basics
description: "Spring Boot에서 CORS가 왜 필요한지, 전역 설정과 특정 경로 허용 방법, 프론트엔드 연동 시 자주 만나는 문제를 쉽게 설명합니다."
image: "/assets/img/og/springboot-series-cover.svg"
---


# SpringBoot 입문 19: CORS 기본과 전역 설정(WebMvcConfigurer)

> 이 글에서는 브라우저의 CORS 오류가 왜 나는지와 Spring MVC에서 전역으로 풀어주는 방식을 설명합니다.
>
> 이전 글: [SpringBoot 입문 18: Swagger(OpenAPI) 문서 자동화(springdoc-openapi)](/springboot/mvc-openapi-swagger-springdoc)
> 다음 글: [SpringBoot 입문 20: Spring Security 아주 기초(인증/인가 감 잡기)](/springboot/mvc-security-basic)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 17: Actuator로 헬스체크/운영 지표 열기(주의점 포함)](/springboot/mvc-actuator-health-metrics)
> - [SpringBoot 입문 21: 파일 업로드/다운로드 기본(Multipart)](/springboot/mvc-file-upload-download)


프론트(예: React)와 백엔드(SpringBoot)를 붙이면 가장 흔히 만나는 에러가 CORS입니다.

목표:

- CORS가 뭔지 한 문장으로 설명한다
- Spring MVC에서 전역 CORS 설정을 적용한다
- "됐는데도 안 되는" 흔한 케이스를 안다

## 1) CORS가 뭐냐

브라우저가 "다른 출처(origin)"로 요청할 때, 서버가 허용해줘야 하는 보안 정책입니다.

## 2) 전역 설정(WebMvcConfigurer)

```java
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.CorsRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class WebCorsConfig implements WebMvcConfigurer {
  @Override
  public void addCorsMappings(CorsRegistry registry) {
    registry.addMapping("/**")
        .allowedOrigins("http://localhost:3000")
        .allowedMethods("GET", "POST", "PUT", "DELETE")
        .allowCredentials(true);
  }
}
```

`addCorsMappings`는 Spring Framework의 표준 확장 포인트입니다. citeturn0search3

## 3) 그래도 안 된다면(가장 흔한 원인)

- Spring Security를 쓰는 경우: Security 쪽 CORS 설정이 별도로 필요할 수 있음
- allowedOrigins에 `*`를 쓰면서 credentials를 true로 둔 경우(브라우저 정책)

## 정리

- CORS는 브라우저 정책
- MVC에서는 WebMvcConfigurer로 전역 설정을 줄 수 있다

<!-- codex-springboot-detail:start -->

## 여기서 한 단계 더 이해해보겠습니다

백엔드 API를 만들고 프론트엔드에서 호출하려는데 브라우저가 막아버리는 상황을 처음 만나면 꽤 당황스럽습니다. 이때 대부분의 원인은 CORS 정책입니다. 서버는 정상인데 브라우저 보안 정책 때문에 호출이 차단되는 것이므로, 단순히 API 코드만 보아서는 원인을 놓치기 쉽습니다.

로컬에서 `http://localhost:3000` 프론트엔드와 `http://localhost:8080` 백엔드를 함께 개발할 때 CORS는 거의 반드시 마주치게 됩니다. 그래서 이 주제는 단순 설정 한 줄이 아니라, 브라우저가 어떤 요청을 왜 막는지 이해하는 학습이라고 생각하시면 좋습니다.

이 주제에서 특히 중요하게 보셔야 하는 부분은 origin, method, header, credentials 개념과 `WebMvcConfigurer` 혹은 보안 설정에서 허용 범위를 다루는 방식 입니다. 처음 읽으실 때는 모든 문장을 외우려고 하기보다, 코드가 어느 계층에 놓이고 어떤 순서로 실행되는지를 천천히 따라가보시면 좋습니다. Spring Boot 학습은 각각의 기술을 따로 외우는 공부라기보다, 여러 조각이 실제 요청 흐름 안에서 어떻게 맞물리는지 이해하는 과정에 가깝습니다.

## 예시 코드로 흐름을 잡아보겠습니다

```java
package com.example.demo.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.CorsRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class CorsConfig implements WebMvcConfigurer {

  @Override
  public void addCorsMappings(CorsRegistry registry) {
    registry.addMapping("/api/**")
        .allowedOrigins("http://localhost:3000")
        .allowedMethods("GET", "POST", "PUT", "DELETE")
        .allowedHeaders("*")
        .allowCredentials(true);
  }
}
```

위 예시는 CORS을 가장 작게 체험할 수 있는 흐름으로 구성했습니다. 처음에는 코드 한 줄 한 줄의 문법보다, 선언 위치와 실행 시점을 중심으로 읽어보시는 편이 훨씬 이해가 잘 되실 것입니다. 예를 들어 설정 클래스인지, Controller인지, Service인지에 따라 책임이 달라지고, 그 책임이 곧 Spring Boot 구조를 읽는 기준이 됩니다.

또 하나 중요한 점은 '지금 당장 돌아가는 코드'와 '나중에도 유지보수하기 쉬운 코드'는 조금 다를 수 있다는 사실입니다. 그래서 예제를 실행해보신 뒤에는 변수명, 메서드명, 경로, 설정 값을 직접 한두 번 바꿔보시면서 어떤 부분이 고정 규칙이고 어떤 부분이 프로젝트 상황에 따라 달라지는지 구분해보시면 정말 큰 도움이 됩니다.

## 자주 하는 실수와 에러도 같이 보겠습니다

- `allowedOrigins("*")`와 `allowCredentials(true)`를 함께 쓰려 하면 스펙 제약 때문에 기대대로 동작하지 않을 수 있습니다.
- 실제 요청은 GET인데 그 전에 OPTIONS preflight가 먼저 가는 구조를 모르고 있으면 원인 추적이 어려워질 수 있습니다.
- Security 설정과 MVC 설정이 따로 있을 때 둘 중 하나만 바꾸고 끝내면 여전히 차단되는 경우가 있습니다.

이런 실수들은 대부분 개념을 몰라서라기보다, 실행 순서나 설정 위치를 한 번에 다 보지 못해서 생깁니다. 그래서 에러가 나면 조급하게 코드를 많이 바꾸기보다, 로그와 요청 흐름을 기준으로 '어디까지는 정상이고 어디서부터 어긋났는지'를 차분히 확인해보시는 편이 좋습니다.

## 실무에서는 이렇게 적용하시면 좋습니다

- 처음에는 가능한 한 좁은 origin만 허용하고, 필요할 때만 범위를 넓혀보시는 편이 안전합니다.
- 브라우저 개발자 도구의 Network 탭에서 preflight와 본 요청을 같이 보시면 이해가 훨씬 빨라집니다.
- 프론트 개발 서버 주소가 바뀌면 CORS 설정도 함께 확인하시는 습관을 들이시면 좋습니다.

처음부터 완벽한 구조를 만들려고 하실 필요는 없습니다. 다만 작은 예제라도 책임 분리, 예외 처리, 설정 관리, 로그 확인 같은 기본 습관을 함께 가져가시면 나중에 프로젝트가 커졌을 때 훨씬 덜 흔들리게 됩니다. 특히 Spring Boot는 편의 기능이 많아서 '왜 이렇게 동작하는지'를 놓치기 쉬우므로, 동작 원리를 설명할 수 있을 정도로 천천히 정리해두시는 것이 좋습니다.


<!-- codex-springboot-inline-links:start -->

지금 읽고 계신 주제가 아직 조금 추상적으로 느껴지신다면 [SpringBoot 입문 20: Spring Security 아주 기초(인증/인가 감 잡기)](/springboot/mvc-security-basic), [SpringBoot 입문 18: Swagger(OpenAPI) 문서 자동화(springdoc-openapi)](/springboot/mvc-openapi-swagger-springdoc), [SpringBoot 입문 30: JWT 인증 기본 흐름](/springboot/mvc-jwt-auth-basics) 글도 함께 읽어보시면 좋습니다. 앞단의 준비 과정과 다음 단계의 확장 흐름이 자연스럽게 이어져서, 지금 배우는 내용이 프로젝트 안에서 어디에 연결되는지 더 분명하게 감을 잡으실 수 있습니다.

<!-- codex-springboot-inline-links:end -->
## 마무리 정리

SpringBoot 입문 19: CORS 기본과 전역 설정(WebMvcConfigurer) 글을 공부하실 때는 한 번 읽고 끝내기보다, 직접 실행해보고 일부러 에러도 만들어보시고, 설정이나 코드 값을 조금씩 바꿔보시는 방식으로 접근해보시길 권합니다. 그렇게 하시면 단순 요약 글보다 훨씬 오래 기억에 남고, 다음 주제인 데이터 접근, 보안, 테스트, 운영 자동화로 넘어갈 때도 이해가 자연스럽게 이어집니다. 지금 단계에서 가장 중요한 것은 완벽함보다 흐름을 잡는 것이니, 천천히 따라오셔도 충분합니다.

<!-- codex-springboot-detail:end -->

<!-- codex-springboot-links:start -->

## 이어서 읽어보시면 좋습니다

- [SpringBoot 입문 20: Spring Security 아주 기초(인증/인가 감 잡기)](/springboot/mvc-security-basic)
- [SpringBoot 입문 18: Swagger(OpenAPI) 문서 자동화(springdoc-openapi)](/springboot/mvc-openapi-swagger-springdoc)
- [SpringBoot 입문 30: JWT 인증 기본 흐름](/springboot/mvc-jwt-auth-basics)

지금 글과 연결되는 흐름으로 골라두었으니, 바로 다음 실습이나 비교 학습을 이어가실 때 참고해보시면 좋겠습니다.

<!-- codex-springboot-links:end -->
