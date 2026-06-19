---
layout: post
title: "SpringBoot 입문 30: JWT 인증 기본 흐름"
date: 2026-05-17 22:20:00 +0900
category: SpringBoot
permalink: /springboot/mvc-jwt-auth-basics
description: "Spring Boot에서 JWT 인증이 동작하는 기본 흐름, Access Token과 Refresh Token 역할, 토큰 검증 시 자주 헷갈리는 포인트를 정리합니다."
image: "/assets/img/og/springboot-series-cover.svg"
---

# SpringBoot 입문 30: JWT 인증 기본 흐름

> 이 글에서는 세션 대신 JWT를 쓸 때 로그인, 토큰 발급, 검증까지 어떤 순서로 흐르는지 이해합니다.
>
> 이전 글: [SpringBoot 입문 29: QueryDSL로 동적 조회 만들기](/springboot/mvc-querydsl-intro)
> 다음 글: [SpringBoot 입문 31: 공통 응답 포맷과 예외 코드 표준화](/springboot/mvc-global-response-format)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 28: 스케줄러로 배치 작업 시작하기](/springboot/mvc-scheduler-batch-basics)
> - [SpringBoot 입문 32: Testcontainers로 MySQL 통합 테스트](/springboot/mvc-testcontainers-mysql)


세션 대신 토큰 기반 인증을 도입할 때 가장 먼저 이해해야 하는 흐름을 정리한다.

## 핵심 포인트
- 로그인 성공 시 Access Token 발급
- 요청 필터에서 토큰 검증 후 인증 객체 구성
- 만료, 재발급(Refresh Token) 정책 분리

## 정리
JWT는 상태 없는 인증에 유리하지만, 토큰 보관 위치와 만료 정책을 안전하게 설계해야 한다.
