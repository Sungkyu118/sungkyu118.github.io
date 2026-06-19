---
layout: post
title: "Redis Pub/Sub vs Streams: 이벤트 전달을 어디에 써야 할까"
date: 2026-05-15 03:30:00 +0900
category: Redis
permalink: /redis/pubsub-vs-streams
description: "Redis Pub/Sub와 Streams의 차이, 메시지 유실 가능성, 재처리 요구사항에 따른 선택 기준을 정리합니다."
image:
  path: "/assets/img/og/redis-series-cover.svg"
  alt: "Redis 시리즈 공통 대표 이미지"
---

# Redis Pub/Sub vs Streams: 이벤트 전달을 어디에 써야 할까

> Redis Pub/Sub와 Streams의 차이, 메시지 유실 가능성, 재처리 요구사항에 따른 선택 기준을 정리합니다.
>
> 이전 글: [Redis Streams 실전: 작업 큐로 쓰는 법](/redis/streams-queue)
> 다음 글: [Redis for 랭킹: Sorted Set으로 실시간 순위 만들기](/redis/ranking)
> 함께 보면 좋은 글:
> - [Redis for 큐: List로 시작하고 Streams를 고민하는 기준](/redis/queue)
> - [Redis Streams 실전: 작업 큐로 쓰는 법](/redis/streams-queue)

Redis에는 이벤트를 전달하는 방식으로 Pub/Sub과 Streams가 있습니다. 둘 다 메시지를 전달할 수 있지만 성격은 꽤 다릅니다.

간단히 말하면 Pub/Sub은 "지금 듣고 있는 사람에게 바로 방송"하는 구조이고, Streams는 "메시지를 기록해두고 consumer가 읽어가는" 구조입니다.

이 차이를 모르고 Pub/Sub을 작업 큐처럼 쓰면 메시지 유실 문제를 만날 수 있고, 반대로 단순 실시간 알림에 Streams를 과하게 붙이면 구조가 필요 이상으로 무거워질 수 있습니다.

작업 큐 관점에서 Redis를 처음 보는 중이라면 [Redis for 큐](/redis/queue)를 먼저 보고, 그 다음 이 글에서 Pub/Sub과 Streams의 차이를 정리하면 흐름이 더 자연스럽습니다.

## Pub/Sub은 방송에 가깝다

Pub/Sub은 publisher가 channel에 메시지를 보내면, 그 순간 구독 중인 subscriber들이 메시지를 받는 구조입니다.

```bash
SUBSCRIBE notice
PUBLISH notice "deploy started"
```

장점은 단순하고 빠르다는 것입니다. 실시간 알림, 서버 간 가벼운 이벤트 전달, 관리자 화면 갱신 같은 곳에 잘 맞습니다.

하지만 중요한 한계가 있습니다. subscriber가 잠시 끊겨 있으면 그동안 발행된 메시지는 받을 수 없습니다. 메시지가 저장되지 않기 때문입니다.

## Pub/Sub이 잘 맞는 상황

Pub/Sub은 이런 상황에 잘 맞습니다.

- 놓쳐도 치명적이지 않은 실시간 알림
- 현재 접속 중인 사용자에게만 보내면 되는 이벤트
- 서버 간 가벼운 broadcast
- 운영 알림처럼 즉시성이 중요하지만 재처리까지는 필요 없는 메시지

예를 들어 관리자 페이지에서 "새 주문이 들어왔음"을 현재 접속 중인 관리자에게 알려주는 정도라면 Pub/Sub이 충분할 수 있습니다.

## Streams는 기록이 남는 이벤트 로그에 가깝다

Streams는 메시지를 stream에 쌓아두고 consumer가 읽어가는 구조입니다.

```bash
XADD stream:mail * userId 42 template welcome
XREAD COUNT 10 STREAMS stream:mail 0
```

Streams는 메시지가 Redis에 남기 때문에, consumer가 잠시 늦게 읽어도 다시 가져갈 수 있습니다. consumer group을 이용하면 여러 worker가 나눠 처리하는 구조도 만들 수 있습니다.

이 점 때문에 Streams는 작업 큐나 이벤트 처리 파이프라인에 더 잘 맞습니다.

## Streams가 잘 맞는 상황

Streams는 이런 상황에 적합합니다.

- 메시지 유실을 줄이고 싶다.
- 여러 worker가 나눠 처리해야 한다.
- 실패한 메시지를 다시 처리해야 한다.
- 어디까지 처리했는지 추적해야 한다.
- consumer group 단위로 작업 분배가 필요하다.

이 주제는 [Redis Streams 실전: 작업 큐로 쓰는 법](/redis/streams-queue)에서 더 구체적으로 이어집니다.

## 가장 중요한 선택 기준: 놓쳐도 되는가

Pub/Sub과 Streams를 고를 때 가장 먼저 물어볼 질문은 이것입니다.

**이 메시지를 놓쳐도 되는가?**

놓쳐도 된다면 Pub/Sub이 단순하고 좋습니다. 놓치면 안 된다면 Streams나 다른 메시지 큐를 검토해야 합니다.

예:

- 채팅방 현재 접속자에게 typing 이벤트 전달: Pub/Sub 가능
- 결제 완료 후 포인트 적립 작업: Streams 또는 별도 MQ 권장
- 관리자 화면 실시간 새로고침 알림: Pub/Sub 가능
- 메일 발송 작업 큐: Streams 쪽이 더 안전

## 운영 난이도 차이

Pub/Sub은 단순하지만 추적이 어렵습니다. 메시지가 저장되지 않으니 "누가 받았는가", "왜 못 받았는가"를 나중에 확인하기 어렵습니다.

Streams는 추적이 가능하지만 운영해야 할 개념이 늘어납니다. consumer group, pending 메시지, ack, 재처리 정책을 알아야 합니다.

그래서 단순함과 안정성 사이에서 요구사항을 보고 선택해야 합니다.

## 자주 하는 실수

### 1) Pub/Sub으로 중요한 작업을 처리한다

메일 발송, 결제 후처리, 포인트 적립처럼 놓치면 안 되는 작업을 Pub/Sub으로만 처리하면 위험합니다.

### 2) Streams를 쓰면서 ACK 처리를 빼먹는다

Streams는 메시지를 읽는 것만으로 끝나지 않습니다. consumer group을 쓴다면 처리 완료 후 ack를 해야 pending이 정리됩니다.

### 3) 재처리 정책 없이 큐를 만든다

실패한 메시지를 어떻게 다시 처리할지, 몇 번까지 재시도할지, 계속 실패하면 어디에 보관할지 정해야 합니다.

## 정리

Redis Pub/Sub은 지금 듣고 있는 대상에게 바로 전달하는 방송 구조입니다. 빠르고 단순하지만 메시지 유실을 감수해야 합니다.

Redis Streams는 메시지를 기록하고 consumer가 처리하는 구조입니다. 운영 개념은 더 많지만, 작업 큐와 재처리가 필요한 이벤트 처리에 더 적합합니다.

선택 기준은 간단합니다. 놓쳐도 되는 실시간 신호면 Pub/Sub, 놓치면 안 되는 작업이면 Streams를 먼저 검토하면 됩니다.
