---
layout: post
title: "Redis 자료구조 2: List와 Set, 순서와 중복을 다루는 법"
date: 2026-05-15 00:09:00 +0900
category: Redis
permalink: /redis/ds-lists-sets
description: "Redis List와 Set으로 큐, 중복 제거, 순서 보장 문제를 다루는 방법을 예제와 함께 정리합니다."
image:
  path: "/assets/img/og/redis-series-cover.svg"
  alt: "Redis 시리즈 공통 대표 이미지"
---

# Redis 자료구조 2: List와 Set, 순서와 중복을 다루는 법

> Redis List와 Set으로 큐, 중복 제거, 순서 보장 문제를 다루는 방법을 예제와 함께 정리합니다.
>
> 이전 글: [Redis 자료구조 1: String과 Hash, 실무 모델링 감각](/redis/ds-strings-hashes)
> 다음 글: [Redis 자료구조 3: Sorted Set(ZSET), 랭킹의 정석](/redis/ds-zset)
> 함께 보면 좋은 글:
> - [Redis for 큐: List로 시작하고 Streams를 고민하는 기준](/redis/queue)
> - [Redis Pub/Sub vs Streams: 이벤트 전달을 어디에 써야 할까](/redis/pubsub-vs-streams)

Redis의 List와 Set은 이름은 단순하지만 실무에서 꽤 자주 등장합니다. 둘의 차이는 분명합니다. List는 순서가 중요할 때 쓰고, Set은 중복 제거와 포함 여부가 중요할 때 씁니다.

이 차이를 놓치면 최근 목록을 Set으로 만들거나, 좋아요 목록을 List로 만들어서 나중에 중복 처리와 조회 성능 때문에 고생할 수 있습니다.

## List: 순서가 중요한 컬렉션

List는 왼쪽과 오른쪽에 값을 넣고 뺄 수 있는 순서 있는 구조입니다.

```bash
LPUSH recent:products:42 1001
LPUSH recent:products:42 1002
LRANGE recent:products:42 0 9
LTRIM recent:products:42 0 19
```

대표적인 사용처는 최근 본 상품, 최근 검색어, 간단한 작업 큐입니다.

최근 본 상품을 예로 들면 사용자가 상품을 볼 때마다 `LPUSH`로 앞에 넣고, `LTRIM`으로 최대 길이를 제한하면 됩니다. 이렇게 하면 메모리가 무한히 늘어나는 것도 막을 수 있습니다.

## List 사용 시 주의할 점

List는 순서에는 강하지만 중복 제거에는 약합니다. 같은 상품을 여러 번 보면 같은 ID가 여러 번 들어갈 수 있습니다. "최근 본 상품이니 중복 제거가 필요하다"면 애플리케이션에서 중복을 처리하거나 다른 구조와 조합해야 합니다.

또한 List를 작업 큐로 사용할 때는 조심해야 합니다. 단순히 `LPUSH`, `RPOP`만 쓰면 consumer가 작업을 가져간 뒤 죽었을 때 재처리가 애매해질 수 있습니다. 재처리와 consumer group이 필요하다면 [Redis Streams 실전](/redis/streams-queue)을 보는 게 더 적절합니다.

## Set: 중복 없는 집합

Set은 순서보다 중복 제거와 포함 여부가 중요한 구조입니다.

```bash
SADD post:100:likes user:42
SISMEMBER post:100:likes user:42
SCARD post:100:likes
SREM post:100:likes user:42
```

좋아요를 누른 사용자 목록, 이벤트 참여자 목록, 태그 집합처럼 "이미 들어갔는가"가 중요한 곳에 잘 맞습니다.

Set의 장점은 같은 값을 여러 번 넣어도 중복으로 저장되지 않는다는 점입니다. 좋아요 버튼을 여러 번 눌러도 사용자 ID가 하나만 남는 구조를 만들 수 있습니다.

## Set 사용 시 주의할 점

Set은 중복 제거에는 좋지만 정렬이 필요하면 맞지 않습니다. 상위 점수, 최신순, 우선순위가 필요하면 Sorted Set이나 List를 검토해야 합니다.

또한 큰 Set에 대해 `SMEMBERS`를 자주 호출하면 부담이 커질 수 있습니다. 운영 데이터가 커질 가능성이 있다면 처음부터 전체 조회를 전제로 설계하지 않는 편이 좋습니다.

## List와 Set 선택 기준

이렇게 고르면 됩니다.

- 순서가 중요하면 List
- 중복 제거가 중요하면 Set
- 순서와 점수가 모두 중요하면 Sorted Set
- 작업 큐 재처리가 중요하면 Streams

예를 들어 "최근 본 상품"은 순서가 중요하니 List가 자연스럽습니다. "좋아요를 누른 사용자 목록"은 중복 제거와 포함 여부가 중요하니 Set이 잘 맞습니다. "인기 글 순위"는 score가 필요하니 Sorted Set이 맞습니다.

## 실무 예시: 최근 본 상품

```bash
LPUSH recent:products:user:42 9001
LTRIM recent:products:user:42 0 19
LRANGE recent:products:user:42 0 9
```

여기서 `LTRIM`이 중요합니다. 최근 목록은 보통 무한히 저장할 필요가 없습니다. 길이를 제한하지 않으면 사용자가 오래 쓸수록 메모리 사용량이 늘어납니다.

## 실무 예시: 이벤트 참여 여부

```bash
SADD event:2026-summer:users user:42
SISMEMBER event:2026-summer:users user:42
SCARD event:2026-summer:users
```

이 구조는 참여 중복 방지에 좋습니다. 다만 참여자가 매우 많고 전체 목록을 자주 내려받아야 한다면 별도 저장소나 배치 처리를 함께 고려해야 합니다.

## 운영에서 자주 하는 실수

### 1) List를 쓰면서 중복 제거를 기대한다

List는 같은 값을 여러 번 넣을 수 있습니다. 중복 제거가 요구사항이면 Set이나 애플리케이션 로직을 같이 봐야 합니다.

### 2) Set을 쓰면서 정렬을 기대한다

Set은 순서를 보장하는 구조가 아닙니다. 순위나 최신순이 필요하면 Sorted Set을 쓰는 편이 맞습니다.

### 3) 컬렉션 길이를 제한하지 않는다

List든 Set이든 계속 쌓이면 메모리 문제가 됩니다. TTL, 길이 제한, 보관 정책을 처음부터 정해야 합니다.

<!-- codex-category-inline-links:start -->

지금 읽고 계신 주제가 아직 조금 추상적으로 느껴지신다면 [Redis for 큐: List로 시작하고 Streams를 고민하는 기준](/redis/queue), [Redis Pub/Sub vs Streams: 이벤트 전달을 어디에 써야 할까](/redis/pubsub-vs-streams), [Redis Streams 실전: 작업 큐로 쓰는 법](/redis/streams-queue) 글도 함께 읽어보시면 좋겠습니다. 같은 Redis 흐름 안에서 앞단의 배경과 다음 단계의 확장 포인트를 같이 보실 수 있어서, 지금 배우는 내용이 실제 프로젝트에서 어디에 연결되는지 훨씬 더 선명하게 이해하실 수 있습니다.

<!-- codex-category-inline-links:end -->
## 정리

List와 Set은 각각 "순서"와 "중복 제거"라는 아주 분명한 목적이 있습니다. 이 목적에 맞게 쓰면 단순하고 강력하지만, 목적을 헷갈리면 나중에 코드와 운영이 복잡해집니다.

List는 최근 목록과 간단한 순서 처리에, Set은 좋아요/참여자/태그처럼 포함 여부와 중복 제거가 중요한 곳에 먼저 떠올리면 됩니다.

<!-- codex-category-links:start -->

## 이어서 읽어보시면 좋습니다

- [Redis for 큐: List로 시작하고 Streams를 고민하는 기준](/redis/queue)
- [Redis Pub/Sub vs Streams: 이벤트 전달을 어디에 써야 할까](/redis/pubsub-vs-streams)
- [Redis Streams 실전: 작업 큐로 쓰는 법](/redis/streams-queue)

지금 글과 바로 이어서 읽기 좋은 흐름으로 묶어두었으니, 개념을 비교해보시거나 다음 실습으로 넘어가실 때 차근차근 따라가보시면 좋겠습니다.

<!-- codex-category-links:end -->
