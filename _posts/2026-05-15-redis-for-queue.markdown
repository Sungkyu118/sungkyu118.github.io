---
layout: post
title: "Redis for 큐"
date: 2026-05-15 02:30:00 +0900
category: Redis
permalink: /redis/queue
---

# Redis for 큐

백그라운드 작업, 비동기 처리, 이벤트 전달이 필요할 때 Redis를 큐처럼 사용하는 경우가 많습니다. 다만 "큐"라는 말 안에는 여러 선택지가 있고, 신뢰성 요구 수준에 따라 답이 달라집니다.

이 글은 다음을 목표로 합니다.

1. Redis를 큐로 쓰는 이유(트레이드오프)
2. List 기반 큐 vs Streams 기반 큐 차이
3. WebFlux에서 어떤 선택이 현실적인지

## 1) 언제/왜 쓰나 (트레이드오프)

### Redis 큐를 쓰는 이유

- 가볍고 빠름(별도 MQ 없이 시작 가능)
- 간단한 비동기 작업 분리(이메일 발송, 알림 등)

### 단점/주의점

- "정확히 한 번(exactly-once)" 같은 강한 보장은 어려움
- 신뢰성이 정말 중요하면 Kafka/RabbitMQ 같은 MQ를 고려해야 함

## 2) List 기반 큐(간단)

전통적으로 `LPUSH/RPOP` 또는 `LPUSH/BRPOP` 조합을 씁니다.

장점:
- 구현이 단순

단점:
- 소비자 그룹/재처리/추적 같은 기능이 빈약

## 3) Streams 기반 큐(추천 방향)

Redis Streams는 소비자 그룹, 메시지 ID, pending(처리중) 개념이 있어 "작업 큐"에 더 적합한 경우가 많습니다.

개념적으로는 이런 흐름:

- 생산: `XADD stream ...`
- 소비: `XREADGROUP GROUP ...`
- ack: `XACK`

## 4) WebFlux 관점에서 선택 기준

- "간단히 비동기 분리"가 목적이면 List로 시작 가능
- "재처리/유실 방지/소비자 그룹"이 필요하면 Streams로 가는 게 낫다

특히 운영 환경에서 작업 유실이 문제가 될 수 있다면, Streams를 먼저 검토하는 편이 안전합니다.

## 정리

- Redis로 큐를 만들 수 있지만, 요구 신뢰성에 따라 선택이 달라진다
- List는 단순, Streams는 작업 큐 기능이 더 풍부
- WebFlux에서는 논블로킹 소비/처리 구조를 설계하는 게 핵심이다

