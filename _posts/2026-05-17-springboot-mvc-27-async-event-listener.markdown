---
layout: post
title: "SpringBoot 입문 27: 이벤트 기반 처리와 @Async"
date: 2026-05-17 22:20:00 +0900
category: SpringBoot
permalink: /springboot/mvc-async-event-listener
---

# SpringBoot 입문 27: 이벤트 기반 처리와 @Async

회원가입, 결제 완료 같은 흐름에서 부가 작업은 비동기로 분리하는 편이 좋다.

## 핵심 포인트
- `ApplicationEventPublisher`로 도메인 이벤트 발행
- `@EventListener` + `@Async`로 비동기 후처리
- 메인 트랜잭션과 후처리를 분리해 응답 시간 단축

## 정리
핵심 로직과 부가 로직을 분리하면 코드 가독성과 장애 대응력이 모두 좋아진다.
