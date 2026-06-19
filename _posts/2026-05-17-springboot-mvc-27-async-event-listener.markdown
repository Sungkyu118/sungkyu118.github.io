---
layout: post
title: "SpringBoot 입문 27: 이벤트 기반 처리와 @Async"
date: 2026-05-17 22:20:00 +0900
category: SpringBoot
permalink: /springboot/mvc-async-event-listener
description: "Spring Boot에서 이벤트 기반 처리와 @Async를 함께 사용할 때의 흐름, 비동기 처리 장점과 주의할 점을 입문자 눈높이로 설명합니다."
image: "/assets/img/og/springboot-series-cover.svg"
---

# SpringBoot 입문 27: 이벤트 기반 처리와 @Async

> 이 글에서는 요청 처리와 분리하고 싶은 작업을 이벤트와 비동기로 넘기는 기본 패턴을 익힙니다.
>
> 이전 글: [SpringBoot 입문 26: Redis 캐시 기본 적용](/springboot/mvc-redis-cache-basics)
> 다음 글: [SpringBoot 입문 28: 스케줄러로 배치 작업 시작하기](/springboot/mvc-scheduler-batch-basics)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 25: JPA N+1 문제와 fetch join](/springboot/mvc-jpa-n-plus-one)
> - [SpringBoot 입문 29: QueryDSL로 동적 조회 만들기](/springboot/mvc-querydsl-intro)


회원가입, 결제 완료 같은 흐름에서 부가 작업은 비동기로 분리하는 편이 좋다.

## 핵심 포인트
- `ApplicationEventPublisher`로 도메인 이벤트 발행
- `@EventListener` + `@Async`로 비동기 후처리
- 메인 트랜잭션과 후처리를 분리해 응답 시간 단축

## 정리
핵심 로직과 부가 로직을 분리하면 코드 가독성과 장애 대응력이 모두 좋아진다.
