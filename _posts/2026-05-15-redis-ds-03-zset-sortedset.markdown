---
layout: post
title: "Redis 자료구조 3: Sorted Set(ZSET), 랭킹의 정석"
date: 2026-05-15 00:10:00 +0900
category: Redis
permalink: /redis/ds-zset
description: "Redis Sorted Set으로 랭킹, 점수 기반 정렬, 범위 조회를 구현하는 기본기를 실전 예시와 함께 설명합니다."
image:
  path: "/assets/img/og/redis-series-cover.svg"
  alt: "Redis 시리즈 공통 대표 이미지"
---

# Redis 자료구조 3: Sorted Set(ZSET), 랭킹의 정석

> Redis Sorted Set으로 랭킹, 점수 기반 정렬, 범위 조회를 구현하는 기본기를 실전 예시와 함께 설명합니다.
>
> 이전 글: [Redis 자료구조 2: List와 Set, 순서와 중복을 다루는 법](/redis/ds-lists-sets)
> 다음 글: [Redis for 캐시: 언제 효과가 크고, 어디서 망가지는가](/redis/cache)
> 함께 보면 좋은 글:
> - [Redis for 랭킹: Sorted Set으로 실시간 순위 만들기](/redis/ranking)
> - [Redis 데이터 구조: String, Hash, List, Set, Sorted Set을 언제 쓸까](/redis/basis)

Sorted Set은 Redis를 랭킹 시스템에 쓰게 만드는 대표 자료구조입니다. Set처럼 member는 중복되지 않지만, 각 member에 score가 붙고 score 기준으로 정렬할 수 있습니다.

일반 Set은 "포함되어 있는가"를 다루는 데 강하고, Sorted Set은 "몇 점이고 몇 등인가"를 다루는 데 강합니다. 그래서 게임 점수, 인기 게시글, 실시간 리더보드, 우선순위 큐 같은 곳에서 자주 등장합니다.

## 기본 개념

Sorted Set은 member와 score의 조합입니다.

```bash
ZADD ranking:daily 1500 user:42
ZADD ranking:daily 1800 user:77
ZREVRANGE ranking:daily 0 9 WITHSCORES
ZRANK ranking:daily user:42
ZREVRANK ranking:daily user:42
```

`ZREVRANGE`는 높은 점수부터 조회할 때 자주 씁니다. 랭킹은 보통 점수가 높은 사람이 위로 올라가기 때문입니다.

## 랭킹에 왜 잘 맞을까

SQL로 랭킹을 구현할 수도 있습니다. 하지만 점수가 자주 바뀌고 상위 N개를 계속 보여줘야 하면 DB 부하가 커질 수 있습니다. Redis Sorted Set은 점수 업데이트와 상위 조회가 단순합니다.

예:

- 점수 갱신: `ZINCRBY ranking:daily 10 user:42`
- 상위 10명 조회: `ZREVRANGE ranking:daily 0 9 WITHSCORES`
- 내 등수 조회: `ZREVRANK ranking:daily user:42`

랭킹은 [Redis for 랭킹](/redis/ranking) 글과 같이 보면 실무 구조를 잡기 좋습니다.

## 기간별 랭킹은 key를 분리하자

랭킹에서 가장 중요한 운영 포인트는 기간입니다. 일간, 주간, 월간 랭킹을 하나의 key에 섞으면 초기화와 조회가 복잡해집니다.

추천 형태:

- `ranking:daily:2026-06-19`
- `ranking:weekly:2026-W25`
- `ranking:monthly:2026-06`

이렇게 나누면 TTL을 주기 쉽고, 과거 랭킹 보관 정책도 명확해집니다.

```bash
ZINCRBY ranking:daily:2026-06-19 10 user:42
EXPIRE ranking:daily:2026-06-19 604800
```

## 동점 처리는 미리 정해야 한다

랭킹에서 은근히 자주 놓치는 부분이 동점 처리입니다. score가 같을 때 어떤 사용자를 먼저 보여줄지 정책이 필요합니다.

가능한 방식:

- 같은 점수면 같은 등수로 보여준다.
- 먼저 달성한 사람을 위로 둔다.
- userId나 timestamp를 보조 기준으로 둔다.

Redis Sorted Set 자체는 score 기준 정렬이 핵심이기 때문에, 복잡한 동점 정책은 애플리케이션 로직이나 score 설계로 보완해야 합니다.

예를 들어 "먼저 달성한 사람이 위"라는 정책이 중요하다면 score에 timestamp를 섞는 방식도 고민할 수 있습니다. 다만 score를 복잡하게 만들면 이해하기 어려워지므로 운영자가 설명 가능한 구조여야 합니다.

## 값이 계속 커지는 문제

랭킹 key는 시간이 지나면 member가 계속 늘 수 있습니다. 이벤트 랭킹처럼 기간이 명확하면 TTL을 주면 됩니다. 하지만 상시 랭킹이라면 오래된 member를 어떻게 정리할지 정해야 합니다.

예:

- 상위 10만 명만 유지
- 일정 기간 활동 없는 user 제거
- 기간별 key로 분리 후 TTL 처리

`ZREMRANGEBYRANK`나 `ZREMRANGEBYSCORE`를 이용해 정리할 수 있지만, 언제 어떤 기준으로 지울지 정책이 먼저 있어야 합니다.

## 실무에서 자주 하는 실수

### 1) 모든 랭킹을 하나의 key에 넣는다

처음에는 편하지만, 기간별 조회와 삭제가 어려워집니다. 일간/주간/월간 요구사항이 나올 가능성이 있으면 처음부터 key를 분리하는 편이 좋습니다.

### 2) score 의미가 문서화되어 있지 않다

score가 점수인지, 누적 조회 수인지, 가중치를 섞은 값인지 모르면 나중에 운영자가 해석하기 어렵습니다.

### 3) 랭킹 조회만 생각하고 업데이트 비용을 놓친다

점수 업데이트가 매우 자주 발생하면 Redis에도 부담이 됩니다. 랭킹은 빠르지만 무한히 공짜는 아닙니다.

<!-- codex-category-inline-links:start -->

Sorted Set은 점수 기반 정렬이라는 설명만으로는 감이 덜 오실 수 있어서, 실제 사용 장면을 같이 보시는 편이 훨씬 좋습니다. [Redis for 랭킹: Sorted Set으로 실시간 순위 만들기](/redis/ranking), [Redis 운영 심화: 메모리, eviction, 핫키 사고 패턴](/redis/memory-eviction-hotkeys), [Redis 데이터 구조: String, Hash, List, Set, Sorted Set을 언제 쓸까](/redis/basis) 글을 함께 보시면, ZSET이 왜 랭킹에 강한지뿐 아니라 운영에서 어디서 비용이 커지는지도 같이 감을 잡으실 수 있습니다. 특히 랭킹은 잘 만들기보다 오래 안정적으로 운영하는 쪽이 더 어렵기 때문에, 구조와 운영 글을 같이 보시는 것이 도움이 됩니다.

<!-- codex-category-inline-links:end -->
## 정리

Sorted Set은 Redis에서 랭킹을 만들 때 가장 먼저 떠올릴 수 있는 자료구조입니다. 점수 갱신, 상위 N개 조회, 내 등수 조회가 단순하기 때문입니다.

하지만 실무에서는 기간별 key 분리, TTL, 동점 정책, member 정리 기준까지 함께 정해야 합니다. 이 기준이 있어야 랭킹 기능이 단순한 데모를 넘어 운영 가능한 기능이 됩니다.

<!-- codex-category-links:start -->

## 이어서 읽어보시면 좋습니다

- [Redis for 랭킹: Sorted Set으로 실시간 순위 만들기](/redis/ranking)

  ZSET을 가장 직관적으로 체감할 수 있는 예시는 실시간 랭킹입니다. 점수 업데이트, 순위 조회, 상위 N개 조회가 어떻게 서비스 기능으로 이어지는지 보면 Sorted Set을 왜 별도로 배워야 하는지 아주 자연스럽게 납득할 수 있습니다.

- [Redis 운영 심화: 메모리, eviction, 핫키 사고 패턴](/redis/memory-eviction-hotkeys)

  랭킹이나 정렬 데이터는 트래픽이 몰리기 쉬워 핫키와 메모리 문제로 이어질 수 있습니다. 운영 심화 글을 함께 읽으면 ZSET을 잘 쓰는 것과 오래 안정적으로 운영하는 것이 서로 다른 문제라는 점을 현실적으로 이해하게 됩니다.

- [Redis 데이터 구조: String, Hash, List, Set, Sorted Set을 언제 쓸까](/redis/basis)

  기본 자료구조 글로 돌아가 보면 ZSET이 다른 구조와 어떤 기준으로 구분되는지 다시 정리할 수 있습니다. 특히 String, Hash, List, Set과 비교하면서 보면 Sorted Set을 써야 하는 상황과 쓰지 않아도 되는 상황을 더 차분하게 판단할 수 있습니다.

위 글들은 지금 읽은 내용과 바로 이어지는 흐름으로 묶어두었습니다. 천천히 따라가시면 개념을 따로 외우는 느낌보다, Redis를 실제 서비스 요구사항에 맞게 고르고 운영하는 감각으로 자연스럽게 이어가실 수 있습니다.

<!-- codex-category-links:end -->
