---
layout: post
title: "Redis 자료구조 2: List와 Set (큐, 피드, 중복 제거)"
date: 2026-05-15 00:09:00 +0900
category: Redis
permalink: /redis/ds-lists-sets
---

# Redis 자료구조 2: List와 Set (큐, 피드, 중복 제거)

List/Set은 "순서"와 "중복"이라는 문제를 다루는 기본 도구입니다.

## 1) List: 순서가 있는 컬렉션

실무에서 흔한 용도:

- 간단한 작업 큐(단, 신뢰성 요구 낮을 때)
- 최근 목록(최근 본 상품 등)

대표 명령:

- `LPUSH key value`
- `RPOP key`
- `LRANGE key 0 9`

"최근 10개" 같은 기능은 List로 구현하기 쉽습니다.

## 2) Set: 중복 없는 집합

실무에서 흔한 용도:

- 좋아요한 사용자 목록
- 팔로우 관계(팔로잉/팔로워)
- 태그/카테고리 집합

대표 명령:

- `SADD key member`
- `SREM key member`
- `SISMEMBER key member`
- `SMEMBERS key`

Set의 강점:

- 중복이 자동으로 제거됨
- membership 체크가 빠름

## 3) List vs Set 선택 기준

- 순서가 중요하면 List
- 중복 제거/포함 여부가 중요하면 Set

## 4) 실전 팁

- "최근 목록"은 길이가 무한히 자라지 않게 자르기(`LTRIM`)를 같이 고려
- Set은 멤버 수가 많아지면 메모리 사용을 체크

