---
layout: post
title: "Redis for 랭킹"
date: 2026-05-15 02:40:00 +0900
category: Redis
permalink: /redis/ranking
---

# Redis for 랭킹

랭킹/리더보드는 Redis의 Sorted Set(ZSET)이 대표적인 해결책입니다. "점수 기반 정렬 + 상위 N개 조회"를 매우 빠르게 할 수 있어요.

이 글은 다음을 목표로 합니다.

1. 랭킹에 Redis를 쓰는 이유(트레이드오프)
2. ZSET의 핵심 명령과 모델링
3. 운영에서 자주 묻는 것(기간 랭킹/동점 처리/메모리)

## 1) 언제/왜 쓰나 (트레이드오프)

### 쓰면 좋은 경우

- 상위 N명/상위 N개를 빠르게 보여줘야 함
- 점수 업데이트가 자주 일어남(좋아요, 포인트, 조회수 등)

### 단점/주의점

- 점수 업데이트가 엄청 많으면 키/샤딩/메모리 고려 필요
- "기간 랭킹"은 키를 나눠서 관리해야 함(일/주/월)

## 2) ZSET 기본 모델

예: 일간 랭킹 키

- 키: `rank:daily:2026-05-15`
- 멤버: userId
- 스코어: 점수

대표 명령:

- 점수 증가: `ZINCRBY key 1 member`
- 상위 조회(내림차순): `ZREVRANGE key 0 9 WITHSCORES`
- 내 순위: `ZREVRANK key member`

## 3) 기간 랭킹 설계

일/주/월 키를 분리하고 TTL을 걸어두면 운영이 쉬워집니다.

- `rank:daily:YYYY-MM-DD` (TTL 8일)
- `rank:weekly:YYYY-WW` (TTL 8주)
- `rank:monthly:YYYY-MM` (TTL 13개월)

## 4) 동점 처리

ZSET은 기본적으로 score로 정렬하고, score가 같으면 member의 사전순으로 정렬됩니다. 동점 정책이 중요하면 아래를 고려합니다.

- score를 "점수 * 큰 수 + 보조값"으로 합성(주의: 범위/정밀도)
- 별도 필드로 tie-breaker를 둔 후 앱에서 후처리

## 정리

- 랭킹은 ZSET이 정석이다
- 기간 랭킹은 키를 분리하고 TTL을 걸어 운영한다
- 동점 정책/메모리/업데이트 빈도를 초반에 결정하면 덜 꼬인다

