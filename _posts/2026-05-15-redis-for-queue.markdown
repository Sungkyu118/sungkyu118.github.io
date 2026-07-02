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

이 글은 Redis를 큐처럼 쓸 때의 출발점에 가깝기 때문에, 자료구조와 메시지 전달 방식까지 같이 보셔야 전체 판단이 쉬워집니다. [Redis 자료구조 2: List와 Set, 순서와 중복을 다루는 법](/redis/ds-lists-sets), [Redis Pub/Sub vs Streams: 이벤트 전달을 어디에 써야 할까](/redis/pubsub-vs-streams), [Redis Streams 실전: 작업 큐로 쓰는 법](/redis/streams-queue) 글을 이어서 보시면, "지금은 List로 충분한지", "이벤트 전달과 작업 큐를 같은 것으로 보면 왜 안 되는지", "언제 Streams로 넘어가야 하는지"가 훨씬 명확해집니다.

<!-- codex-category-inline-links:end -->
## 정리

Redis List는 간단한 큐를 빠르게 만들기에 좋습니다. 하지만 작업 유실, 재시도, consumer group, 실패 추적이 중요해지는 순간 Streams를 고민해야 합니다.

큐는 단순히 비동기로 미루는 도구가 아니라, 실패와 재처리까지 포함한 운영 구조입니다. 그래서 처음부터 "실패하면 어떻게 할 것인가"를 같이 설계해야 합니다.

<!-- codex-category-links:start -->

## 이어서 읽어보시면 좋습니다

- [Redis 자료구조 2: List와 Set, 순서와 중복을 다루는 법](/redis/ds-lists-sets)

  큐를 Redis로 이해하려면 먼저 List가 순서를 어떻게 다루는지, Set이 중복을 어떻게 바라보는지 알고 있어야 합니다. 이 자료구조 글을 함께 읽으면 큐 예제의 명령어가 단순 암기가 아니라 자료구조의 성질에서 자연스럽게 나온다는 점을 이해할 수 있습니다.

- [Redis Pub/Sub vs Streams: 이벤트 전달을 어디에 써야 할까](/redis/pubsub-vs-streams)

  큐와 이벤트 전달은 비슷해 보이지만 실패 처리와 메시지 보관 기준에서 완전히 다른 선택이 됩니다. Pub/Sub와 Streams 비교 글을 같이 읽으면 Redis로 비동기 처리를 만들 때 작업 큐, 실시간 알림, 이벤트 로그를 구분하는 기준을 더 안전하게 세울 수 있습니다.

- [Redis Streams 실전: 작업 큐로 쓰는 법](/redis/streams-queue)

  List 기반 큐를 이해했다면 Streams 큐 글은 자연스러운 다음 단계입니다. 소비자 그룹, pending 메시지, 재처리 같은 개념을 보면 실무에서 왜 단순 push/pop만으로는 부족해지는 순간이 오는지 더 구체적으로 느낄 수 있습니다.

위 글들은 지금 읽은 내용과 바로 이어지는 흐름으로 묶어두었습니다. 천천히 따라가시면 개념을 따로 외우는 느낌보다, Redis를 실제 서비스 요구사항에 맞게 고르고 운영하는 감각으로 자연스럽게 이어가실 수 있습니다.

<!-- codex-category-links:end -->
