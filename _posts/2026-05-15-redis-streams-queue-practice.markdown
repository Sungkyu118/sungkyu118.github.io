---
layout: post
title: "Redis Streams 실전: 작업 큐로 쓰는 법"
date: 2026-05-15 03:20:00 +0900
category: Redis
permalink: /redis/streams-queue
---

# Redis Streams 실전: 작업 큐로 쓰는 법

Redis Streams는 Redis 안에서 메시지를 쌓고 consumer가 읽어가는 자료구조입니다. 단순한 Pub/Sub과 달리 메시지가 stream에 남고, consumer group을 통해 여러 worker가 나눠 처리할 수 있습니다.

메일 발송, 알림 처리, 비동기 집계, 후처리 작업처럼 "요청과 분리해서 처리하되 유실되면 곤란한 작업"에 잘 맞습니다.

Pub/Sub과의 차이는 [Redis Pub/Sub vs Streams](/redis/pubsub-vs-streams)에서 먼저 보면 좋습니다.

## 기본 명령어

메시지를 추가할 때는 `XADD`를 사용합니다.

```bash
XADD stream:mail * userId 42 template welcome
```

읽을 때는 `XREAD`를 사용할 수 있습니다.

```bash
XREAD COUNT 10 STREAMS stream:mail 0
```

하지만 실무 큐에서는 consumer group을 쓰는 경우가 많습니다.

```bash
XGROUP CREATE stream:mail mail-workers $ MKSTREAM
XREADGROUP GROUP mail-workers worker-1 COUNT 10 STREAMS stream:mail >
XACK stream:mail mail-workers 1710000000000-0
```

## consumer group이 필요한 이유

consumer group은 여러 worker가 같은 stream을 나눠 처리할 수 있게 해줍니다. worker가 3개라면 각 worker가 서로 다른 메시지를 가져가 처리할 수 있습니다.

이 구조가 중요한 이유는 처리량을 늘릴 수 있고, 어떤 메시지가 처리 중인지 추적할 수 있기 때문입니다.

List 기반 큐보다 운영성이 필요한 경우에는 [Redis for 큐](/redis/queue)에서 이야기한 것처럼 Streams가 더 적합할 수 있습니다.

## ACK가 중요하다

Streams에서 consumer group을 쓸 때 메시지는 읽는 것만으로 끝나지 않습니다. 처리가 끝났다면 `XACK`를 호출해야 합니다.

ACK가 없으면 Redis는 해당 메시지를 pending 상태로 봅니다. worker가 죽었거나 처리에 실패했을 때 pending 메시지를 다시 가져와 재처리할 수 있습니다.

이 점이 Pub/Sub이나 단순 List 큐와 다른 큰 차이입니다.

## 실패와 재처리

작업 큐는 성공보다 실패 설계가 더 중요합니다. 외부 API가 잠깐 실패하거나, 메일 서버가 느리거나, worker가 중간에 죽는 일은 충분히 생깁니다.

정해야 할 것:

- 몇 번까지 재시도할 것인가?
- 재시도 간격은 어떻게 둘 것인가?
- 계속 실패한 메시지는 어디에 둘 것인가?
- 실패 사유는 어디에 기록할 것인가?

Streams는 pending 메시지를 추적할 수 있지만, 재시도 정책은 애플리케이션이 명확하게 설계해야 합니다.

## 메시지 크기와 보관 정책

Stream은 메시지를 계속 쌓을 수 있습니다. 하지만 무한히 쌓으면 메모리 문제가 됩니다. 그래서 trim 정책을 고려해야 합니다.

```bash
XADD stream:mail MAXLEN ~ 100000 * userId 42 template welcome
```

`MAXLEN`을 이용하면 stream 길이를 제한할 수 있습니다. 단, 너무 짧게 잡으면 아직 처리하지 못한 메시지가 사라질 수 있으니 consumer 처리 속도와 보관 요구사항을 같이 봐야 합니다.

## 실무 예시: 메일 발송 작업

회원가입 요청에서 바로 메일을 보내지 않고 Streams에 메시지만 넣을 수 있습니다.

```bash
XADD stream:mail * userId 42 template welcome email user@example.com
```

worker는 consumer group으로 메시지를 읽고 메일 발송에 성공하면 ACK합니다. 실패하면 재시도하거나 실패 저장소로 보냅니다.

이렇게 하면 사용자의 회원가입 응답은 빠르게 유지하면서, 메일 발송 실패를 별도로 처리할 수 있습니다.

## 운영에서 자주 하는 실수

### 1) ACK를 빼먹는다

처리 성공 후 ACK하지 않으면 pending이 계속 쌓입니다. 나중에 재처리 로직에서 같은 메시지를 다시 보게 될 수 있습니다.

### 2) pending 메시지를 보지 않는다

pending이 계속 늘면 worker가 처리하지 못하고 있거나 실패가 누적되고 있다는 뜻입니다.

### 3) stream을 무한 보관한다

메시지를 계속 쌓기만 하면 메모리 문제가 됩니다. 보관 기간과 trim 기준을 정해야 합니다.

### 4) 작업을 idempotent하게 만들지 않는다

재처리 구조에서는 같은 메시지가 두 번 처리될 가능성을 고려해야 합니다. 메일이 두 번 갈 수 있는지, 포인트가 두 번 적립될 수 있는지 확인해야 합니다.

## 정리

Redis Streams는 단순 비동기 작업을 운영 가능한 큐에 가깝게 만들 수 있는 도구입니다. consumer group, ACK, pending, trim 같은 개념을 이해하면 List보다 훨씬 안정적으로 작업을 처리할 수 있습니다.

하지만 Streams도 마법은 아닙니다. 실패 재시도, 메시지 보관, 중복 처리 방지, 모니터링을 애플리케이션에서 같이 설계해야 진짜 운영 가능한 큐가 됩니다.
