---
layout: post
title: "SpringBoot 입문 20: Spring Security 아주 기초(인증/인가 감 잡기)"
date: 2026-05-17 03:20:00 +0900
category: SpringBoot
permalink: /springboot/mvc-security-basic
description: "Spring Security 입문자가 인증과 인가 개념을 처음 잡을 수 있도록 기본 흐름, 401과 403 차이, SecurityFilterChain 시작점을 쉽게 설명합니다."
image:
  path: "/assets/img/og/springboot-security-cover.svg"
  alt: "SpringBoot Security basics article cover"
---

# SpringBoot 입문 20: Spring Security 아주 기초(인증/인가 감 잡기)

> 이 글에서는 Spring Security를 처음 붙였을 때 왜 401과 403이 나오는지부터, 어떤 경로를 열고 막을지 결정하는 기초 흐름을 이해합니다.
>
> 이전 글: [SpringBoot 입문 19: CORS 기본과 전역 설정(WebMvcConfigurer)](/springboot/mvc-cors-basics)
> 다음 글: [SpringBoot 입문 21: 파일 업로드/다운로드 기본(Multipart)](/springboot/mvc-file-upload-download)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 30: JWT 인증 기본 흐름](/springboot/mvc-jwt-auth-basics)
> - [SpringBoot 입문 18: Swagger(OpenAPI) 문서 자동화(springdoc-openapi)](/springboot/mvc-openapi-swagger-springdoc)

Security는 초보자에게 어렵지만, "기본 흐름"만 잡아도 개발을 시작할 수 있습니다.

용어:

- 인증(Authentication): 너 누구야?
- 인가(Authorization): 너 이 기능 해도 돼?

## 1) 의존성 추가

```gradle
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-security'
}
```

추가하면 기본적으로 많은 엔드포인트가 보호되기 때문에, 처음엔 "왜 401/403이 뜨지?"를 경험하게 됩니다.

## 2) 가장 쉬운 시작: 특정 경로만 열기

초보자용으로는 `/hello`, `/swagger` 등만 허용하고 나머지는 막는 형태로 시작하면 감을 잡기 좋습니다.

## 3) 실전 팁

- 처음부터 JWT까지 들어가면 복잡도가 급증합니다.
- MVC 입문 시리즈에서는 "SecurityFilterChain 개념 + 허용/차단"만 우선 잡는 걸 추천합니다.

## 정리

- Security는 인증/인가를 담당한다
- 의존성 추가만으로도 기본 보호가 켜질 수 있다
- 처음엔 경로 기반으로 허용/차단부터 잡자

