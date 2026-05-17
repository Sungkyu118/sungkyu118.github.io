---
layout: post
title: "SpringBoot 입문 31: 공통 응답 포맷과 예외 코드 표준화"
date: 2026-05-17 22:20:00 +0900
category: SpringBoot
permalink: /springboot/mvc-global-response-format
---

# SpringBoot 입문 31: 공통 응답 포맷과 예외 코드 표준화

클라이언트와 협업할 때 응답 구조를 통일하면 개발 속도가 빨라진다.

## 핵심 포인트
- 성공/실패 응답 스키마 공통화
- 비즈니스 예외 코드를 enum으로 관리
- `@RestControllerAdvice`로 예외 응답 일괄 처리

## 정리
API 응답 표준화는 기능 추가보다 먼저 챙기면 장기 유지보수 비용을 줄여준다.
