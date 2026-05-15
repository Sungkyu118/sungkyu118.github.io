---
layout: post
title: "Redis 자료구조 3: Sorted Set(ZSET) (랭킹의 정석)"
date: 2026-05-15 00:10:00 +0900
category: Redis
permalink: /redis/ds-zset
---

# Redis 자료구조 3: Sorted Set(ZSET) (랭킹의 정석)

Sorted Set(ZSET)은 Redis를 Redis답게 만드는 자료구조 중 하나입니다. "점수(score)로 정렬된 집합"이라서 랭킹/리더보드에 정석이에요.

## 1) 기본 모델

- key: 랭킹 이름(기간/종류 포함)
- member: 유저/아이템 ID
- score: 점수(정렬 기준)

예:

- `rank:daily:2026-05-15`

## 2) 대표 명령

- 점수 증가: `ZINCRBY key 1 member`
- 상위 N개: `ZREVRANGE key 0 9 WITHSCORES`
- 내 순위: `ZREVRANK key member`

## 3) 기간 랭킹 운영

기간별로 키를 나누고 TTL로 청소하는 방식이 가장 단순합니다.

- daily/weekly/monthly 키 분리
- TTL로 자동 정리

## 4) 동점 처리

score가 같으면 member의 사전순으로 정렬됩니다. 동점 정책이 중요하면 별도 로직(보조 스코어/후처리)을 고려해야 합니다.

## 정리

- 랭킹은 ZSET이 정석
- 기간 랭킹은 키 분리 + TTL 운영이 깔끔
- 동점 정책을 초반에 정하면 운영이 편하다

