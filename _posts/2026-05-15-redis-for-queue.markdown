---
layout: post
title: "Redis for 큐: List로 시작하고 Streams를 고민하는 기준"
date: 2026-05-15 02:30:00 +0900
category: Redis
permalink: /redis/queue
description: "Redis List와 Streams를 기준으로 큐를 설계할 때의 선택 기준과 운영 포인트를 설명합니다."
image:
  path: "/assets/img/og/redis-series-cover.svg"
  alt: "Redis 시리즈 공통 대표 이미지"
---

# Redis for 큐: List로 시작하고 Streams를 고민하는 기준

> Redis List와 Streams를 기준으로 큐를 설계할 때의 선택 기준과 운영 포인트를 설명합니다.
>
> 이전 글: [Redis 레이트 리밋: Lua로 원자성 보장하기](/redis/rate-limit-lua)
> 다음 글: [Redis Streams 실전: 작업 큐로 쓰는 법](/redis/streams-queue)
> 함께 보면 좋은 글:
> - [Redis Streams 실전: 작업 큐로 쓰는 법](/redis/streams-queue)
> - [Redis Pub/Sub vs Streams: 이벤트 전달을 어디에 써야 할까](/redis/pubsub-vs-streams)

큐는 요청을 바로 처리하지 않고 뒤로 미루기 위한 구조입니다. 사용자가 버튼을 눌렀을 때 화면 응답은 빨리 주고, 메일 발송이나 알림 처리, 집계 같은 일은 백그라운드에서 처리하는 식입니다.

Redis는 큐 용도로도 자주 사용됩니다. 다만 "큐처럼 쓸 수 있다"와 "운영 가능한 큐다"는 다릅니다. 단순 작업이면 List로도 충분하지만, 재처리와 consumer 관리가 중요해지면 Streams가 더 맞을 수 있습니다.

List와 Set의 기본 차이가 아직 낯설다면 [Redis 자료구조 2: List와 Set](/redis/ds-lists-sets)을 먼저 보면 큐 구조를 이해하기가 더 쉽습니다.

## 큐를 쓰는 이유

큐를 쓰는 가장 큰 이유는 요청-응답 흐름을 가볍게 만들기 위해서입니다.

예를 들어 회원가입 후 아래 작업을 모두 동기 처리한다고 생각해보겠습니다.

- 사용자 저장
- 환영 메일 발송
- 관리자 알림
- 통계 이벤트 적재
- 추천 데이터 초기화

이 모든 작업을 한 요청 안에서 처리하면 응답이 느려지고, 메일 서버가 느린 것만으로 회원가입 API가 실패할 수 있습니다. 큐를 쓰면 핵심 요청과 후처리를 분리할 수 있습니다.

## List 기반 큐

가장 단순한 구조는 List입니다.

```bash
LPUSH queue:email "{\"userId\":42,\"type\":\"welcome\"}"
RPOP queue:email
```

생산자는 `LPUSH`로 작업을 넣고, 소비자는 `RPOP`으로 꺼냅니다. 구조가 단순하고 이해하기 쉽습니다.

하지만 단순한 만큼 한계도 분명합니다.

## List 큐의 주의점

### 1) 작업을 꺼낸 뒤 consumer가 죽으면 어떻게 할까

`RPOP`으로 작업을 꺼낸 순간 Redis에서는 사라집니다. 그런데 처리 중 서버가 죽으면 그 작업은 유실될 수 있습니다.

이 문제를 막으려면 processing queue를 따로 두거나, Streams 같은 구조를 고려해야 합니다.

### 2) 재시도 정책이 없다

메일 발송 실패, 외부 API 실패, 일시적 네트워크 오류는 흔합니다. 큐를 운영하려면 재시도 횟수, 지연 재시도, 실패 보관소 같은 정책이 필요합니다.

### 3) 처리 순서와 병렬성 기준이 필요하다

무조건 순서대로 처리해야 하는 작업인지, 여러 consumer가 병렬 처리해도 되는지에 따라 구조가 달라집니다.

## Streams가 필요한 순간

Redis Streams는 List보다 큐에 가까운 기능을 제공합니다. 메시지 ID, consumer group, pending entry 관리가 가능합니다.

Streams를 고려할 만한 상황:

- 작업 유실을 줄이고 싶다.
- 여러 consumer가 나눠 처리해야 한다.
- 처리 실패 후 재처리가 필요하다.
- 누가 어디까지 처리했는지 추적하고 싶다.

자세한 내용은 [Redis Streams 실전: 작업 큐로 쓰는 법](/redis/streams-queue)에서 다룹니다.

## 큐 설계에서 가장 중요한 질문

큐를 만들 때는 명령어보다 요구사항을 먼저 봐야 합니다.

1. 작업은 유실되어도 되는가?
2. 중복 처리되어도 되는가?
3. 실패하면 몇 번 재시도할 것인가?
4. 처리 순서가 중요한가?
5. 처리 결과를 어디에 기록할 것인가?

이 질문에 답하지 않으면 큐는 금방 블랙박스가 됩니다. "넣었는데 처리됐는지 모르겠다"는 상태가 가장 위험합니다.

## 실무 예시: 메일 발송 큐

메일 발송은 큐에 잘 맞는 작업입니다. 사용자는 회원가입 결과를 빨리 받아야 하고, 메일 발송은 잠깐 늦어져도 됩니다.

```bash
LPUSH queue:mail '{"userId":42,"template":"welcome"}'
```

처리 쪽에서는 작업을 꺼내 메일을 보내고, 실패하면 retry queue나 dead letter queue로 보낼 수 있습니다.

실제 운영에서는 아래가 필요합니다.

- 처리 성공 로그
- 실패 사유 기록
- 최대 재시도 횟수
- 오래 쌓인 작업 알림

## 운영에서 자주 하는 실수

### 1) 큐 길이를 보지 않는다

큐가 계속 쌓이면 consumer가 처리 속도를 따라가지 못한다는 뜻입니다. `LLEN`이나 Streams lag를 봐야 합니다.

### 2) 실패 작업을 버린다

실패를 단순 로그로만 남기면 나중에 복구가 어렵습니다. 실패 큐나 별도 저장소를 두는 편이 좋습니다.

### 3) 중복 처리를 고려하지 않는다

큐 시스템은 장애 상황에서 중복 처리가 발생할 수 있습니다. 작업 자체를 idempotent하게 만드는 것이 중요합니다.

<!-- codex-category-inline-links:start -->

지금 읽고 계신 주제가 아직 조금 추상적으로 느껴지신다면 [Redis 자료구조 2: List와 Set, 순서와 중복을 다루는 법](/redis/ds-lists-sets), [Redis Pub/Sub vs Streams: 이벤트 전달을 어디에 써야 할까](/redis/pubsub-vs-streams), [Redis Streams 실전: 작업 큐로 쓰는 법](/redis/streams-queue) 글도 함께 읽어보시면 좋겠습니다. 같은 Redis 흐름 안에서 앞단의 배경과 다음 단계의 확장 포인트를 같이 보실 수 있어서, 지금 배우는 내용이 실제 프로젝트에서 어디에 연결되는지 훨씬 더 선명하게 이해하실 수 있습니다.

<!-- codex-category-inline-links:end -->
## 정리

Redis List는 간단한 큐를 빠르게 만들기에 좋습니다. 하지만 작업 유실, 재시도, consumer group, 실패 추적이 중요해지는 순간 Streams를 고민해야 합니다.

큐는 단순히 비동기로 미루는 도구가 아니라, 실패와 재처리까지 포함한 운영 구조입니다. 그래서 처음부터 "실패하면 어떻게 할 것인가"를 같이 설계해야 합니다.

<!-- codex-category-links:start -->

## 이어서 읽어보시면 좋습니다

- [Redis 자료구조 2: List와 Set, 순서와 중복을 다루는 법](/redis/ds-lists-sets)
- [Redis Pub/Sub vs Streams: 이벤트 전달을 어디에 써야 할까](/redis/pubsub-vs-streams)
- [Redis Streams 실전: 작업 큐로 쓰는 법](/redis/streams-queue)

지금 글과 바로 이어서 읽기 좋은 흐름으로 묶어두었으니, 개념을 비교해보시거나 다음 실습으로 넘어가실 때 차근차근 따라가보시면 좋겠습니다.

<!-- codex-category-links:end -->
