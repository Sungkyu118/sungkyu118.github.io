---
layout: post
title: "SpringBoot 입문 26: Redis 캐시 기본 적용"
date: 2026-05-17 22:20:00 +0900
category: SpringBoot
permalink: /springboot/mvc-redis-cache-basics
---

# SpringBoot 입문 26: Redis 캐시 기본 적용

조회가 많은 API는 캐시를 적용하면 응답 속도를 크게 개선할 수 있다.

## 핵심 포인트
- `@Cacheable`로 읽기 캐시 적용
- `@CacheEvict`로 데이터 변경 시 캐시 무효화
- TTL 설정으로 오래된 데이터 자동 정리

## 정리
캐시는 성능 향상 도구이지만, 데이터 일관성 기준을 먼저 정하고 적용해야 운영 안정성이 높아진다.
