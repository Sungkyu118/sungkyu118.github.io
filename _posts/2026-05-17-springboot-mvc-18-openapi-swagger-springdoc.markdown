---
layout: post
title: "SpringBoot 입문 18: Swagger(OpenAPI) 문서 자동화(springdoc-openapi)"
date: 2026-05-17 03:00:00 +0900
category: SpringBoot
permalink: /springboot/mvc-openapi-swagger-springdoc
---

# SpringBoot 입문 18: Swagger(OpenAPI) 문서 자동화(springdoc-openapi)

API를 만들면 다음 문제가 바로 생깁니다.

- "이 API는 어떤 요청을 받고 어떤 응답을 주지?"

문서를 손으로 관리하면 금방 코드와 어긋납니다. Spring Boot 3.x에서는 `springdoc-openapi`를 많이 씁니다. citeturn0search1turn0search9

## 1) 의존성 추가(Gradle)

Spring MVC(= `spring-boot-starter-web`)이면 아래를 추가합니다.

```gradle
dependencies {
  implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.4.0'
}
```

Spring Boot 3.x에서 `springdoc-openapi-starter-webmvc-ui`가 일반적인 선택입니다. citeturn0search2turn0search1

## 2) 실행 후 접속

기본 URL:

- `http://localhost:8080/swagger-ui.html`
- 또는 `http://localhost:8080/swagger-ui/index.html`

## 3) 실전 팁

- 운영(prod)에서는 swagger UI를 막는 팀도 많습니다.
- local/dev에서만 켜고, prod에서는 비활성화하는 전략이 흔합니다(Profiles로 분기).

## 정리

- 문서는 자동화하는 게 장기적으로 이득
- springdoc으로 OpenAPI/Swagger UI를 쉽게 띄울 수 있다

