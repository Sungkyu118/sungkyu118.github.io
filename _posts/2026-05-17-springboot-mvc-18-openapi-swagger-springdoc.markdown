---
layout: post
title: "SpringBoot 입문 18: Swagger(OpenAPI) 문서 자동화(springdoc-openapi)"
date: 2026-05-17 03:00:00 +0900
category: SpringBoot
permalink: /springboot/mvc-openapi-swagger-springdoc
description: "Spring Boot 3.x 프로젝트에 springdoc-openapi를 붙여 Swagger UI를 열고 API 문서를 자동화하는 방법, 운영 환경에서 주의할 점까지 정리합니다."
image:
  path: "/assets/img/og/springboot-swagger-cover.svg"
  alt: "SpringBoot Swagger and OpenAPI article cover"
---

# SpringBoot 입문 18: Swagger(OpenAPI) 문서 자동화(springdoc-openapi)

> 이 글에서는 Spring Boot API 문서를 자동으로 열어주는 Swagger UI를 붙이고, 운영에서 어디까지 공개할지 판단하는 기준까지 다룹니다.
>
> 이전 글: [SpringBoot 입문 17: Actuator로 헬스체크/운영 지표 열기(주의점 포함)](/springboot/mvc-actuator-health-metrics)
> 다음 글: [SpringBoot 입문 19: CORS 기본과 전역 설정(WebMvcConfigurer)](/springboot/mvc-cors-basics)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 3: 요청 받기(@RequestBody) + Validation + 예외 처리](/springboot/mvc-request-validation-exception)
> - [SpringBoot 입문 31: 공통 응답 포맷과 예외 코드 표준화](/springboot/mvc-global-response-format)

API를 만들다 보면 다음 문제가 바로 생긴다.

- "이 API는 어떤 요청을 받고 어떤 응답을 주지?"

문서를 수동으로 관리하면 금방 코드와 어긋난다. Spring Boot 3.x에서는 `springdoc-openapi`를 많이 사용한다.

## 1) 의존성 추가(Gradle)

Spring MVC(= `spring-boot-starter-web`)라면 아래를 추가한다.

```gradle
dependencies {
  implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.4.0'
}
```

Spring Boot 3.x에서는 `springdoc-openapi-starter-webmvc-ui`가 일반적인 선택이다.

## 2) 실행 후 접속

기본 URL:

- `http://localhost:8080/swagger-ui.html`
- 또는 `http://localhost:8080/swagger-ui/index.html`

## 3) 실전 팁

- 운영(prod)에서는 Swagger UI를 막는 경우가 많다.
- local/dev에서만 열고, prod에서는 비활성화하는 전략이 안전하다(Profiles로 분리).

## 정리

- 문서를 자동화하면 개발 효율이 올라간다.
- springdoc으로 OpenAPI/Swagger UI를 쉽게 구성할 수 있다.
