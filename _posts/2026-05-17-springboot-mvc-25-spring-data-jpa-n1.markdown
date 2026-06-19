---
layout: post
title: "SpringBoot 입문 25: JPA N+1 문제와 fetch join"
date: 2026-05-17 22:20:00 +0900
category: SpringBoot
permalink: /springboot/mvc-jpa-n-plus-one
description: "Spring Boot JPA에서 N+1 문제가 왜 생기는지, fetch join으로 어떻게 줄이는지, 조회 성능 문제를 처음 이해하는 관점에서 설명합니다."
image: "/assets/img/og/springboot-series-cover.svg"
---

# SpringBoot 입문 25: JPA N+1 문제와 fetch join

> 이 글에서는 JPA 조회 성능 문제의 대표 사례인 N+1이 왜 생기는지와 fetch join으로 어떻게 줄이는지 이해합니다.
>
> 이전 글: [SpringBoot 입문 24: Docker Compose로 앱+DB 함께 실행](/springboot/mvc-docker-compose-basics)
> 다음 글: [SpringBoot 입문 26: Redis 캐시 기본 적용](/springboot/mvc-redis-cache-basics)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 23: Docker로 실행하기(가장 단순한 Dockerfile)](/springboot/mvc-dockerize-basics)
> - [SpringBoot 입문 27: 이벤트 기반 처리와 @Async](/springboot/mvc-async-event-listener)


JPA를 사용할 때 가장 자주 만나는 성능 이슈가 N+1 문제다.

## 핵심 포인트
- 연관 엔티티를 지연 로딩할 때 반복 조회 발생
- `fetch join`으로 필요한 연관 데이터를 한 번에 조회
- 무분별한 즉시 로딩보다 쿼리 의도를 분명히 하는 것이 중요

## 정리
엔티티 설계만큼 조회 쿼리 전략이 중요하며, 로그를 보고 실제 쿼리 횟수를 확인하는 습관이 필요하다.
