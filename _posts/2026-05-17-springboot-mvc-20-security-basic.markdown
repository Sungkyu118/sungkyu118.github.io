---
layout: post
title: "SpringBoot 입문 20: Spring Security 아주 기초(인증/인가 감 잡기)"
date: 2026-05-17 03:20:00 +0900
category: SpringBoot
permalink: /springboot/mvc-security-basic
---

# SpringBoot 입문 20: Spring Security 아주 기초(인증/인가 감 잡기)

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

