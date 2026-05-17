---
layout: post
title: "SpringBoot 입문 25: JPA N+1 문제와 fetch join"
date: 2026-05-17 22:20:00 +0900
category: SpringBoot
permalink: /springboot/mvc-jpa-n-plus-one
---

# SpringBoot 입문 25: JPA N+1 문제와 fetch join

JPA를 사용할 때 가장 자주 만나는 성능 이슈가 N+1 문제다.

## 핵심 포인트
- 연관 엔티티를 지연 로딩할 때 반복 조회 발생
- `fetch join`으로 필요한 연관 데이터를 한 번에 조회
- 무분별한 즉시 로딩보다 쿼리 의도를 분명히 하는 것이 중요

## 정리
엔티티 설계만큼 조회 쿼리 전략이 중요하며, 로그를 보고 실제 쿼리 횟수를 확인하는 습관이 필요하다.
