---
layout: post
title: "SpringBoot 입문 30: JWT 인증 기본 흐름"
date: 2026-05-17 22:20:00 +0900
category: SpringBoot
permalink: /springboot/mvc-jwt-auth-basics
---

# SpringBoot 입문 30: JWT 인증 기본 흐름

세션 대신 토큰 기반 인증을 도입할 때 가장 먼저 이해해야 하는 흐름을 정리한다.

## 핵심 포인트
- 로그인 성공 시 Access Token 발급
- 요청 필터에서 토큰 검증 후 인증 객체 구성
- 만료, 재발급(Refresh Token) 정책 분리

## 정리
JWT는 상태 없는 인증에 유리하지만, 토큰 보관 위치와 만료 정책을 안전하게 설계해야 한다.
