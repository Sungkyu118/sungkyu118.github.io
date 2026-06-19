---
layout: post
title: "SpringBoot 입문 26: Redis 캐시 기본 적용"
date: 2026-05-17 22:20:00 +0900
category: SpringBoot
permalink: /springboot/mvc-redis-cache-basics
description: "Spring Boot에 Redis 캐시를 붙여 조회 성능을 개선하는 기본 흐름, @Cacheable 적용 지점과 캐시 도입 전 알아둘 주의사항을 정리합니다."
image: "/assets/img/og/springboot-series-cover.svg"
---

# SpringBoot 입문 26: Redis 캐시 기본 적용

> 이 글에서는 Redis 캐시를 처음 붙일 때 어떤 메서드에 캐시를 둘지와 무효화 포인트를 고민하는 기준을 다룹니다.
>
> 이전 글: [SpringBoot 입문 25: JPA N+1 문제와 fetch join](/springboot/mvc-jpa-n-plus-one)
> 다음 글: [SpringBoot 입문 27: 이벤트 기반 처리와 @Async](/springboot/mvc-async-event-listener)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 24: Docker Compose로 앱+DB 함께 실행](/springboot/mvc-docker-compose-basics)
> - [SpringBoot 입문 28: 스케줄러로 배치 작업 시작하기](/springboot/mvc-scheduler-batch-basics)


조회가 많은 API는 캐시를 적용하면 응답 속도를 크게 개선할 수 있다.

## 핵심 포인트
- `@Cacheable`로 읽기 캐시 적용
- `@CacheEvict`로 데이터 변경 시 캐시 무효화
- TTL 설정으로 오래된 데이터 자동 정리

## 정리
캐시는 성능 향상 도구이지만, 데이터 일관성 기준을 먼저 정하고 적용해야 운영 안정성이 높아진다.
