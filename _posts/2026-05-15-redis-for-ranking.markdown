---
layout: post
title: "Redis for 랭킹: Sorted Set으로 실시간 순위 만들기"
date: 2026-05-15 02:40:00 +0900
category: Redis
permalink: /redis/ranking
description: "Redis Sorted Set으로 실시간 랭킹을 구현할 때 필요한 명령어와 운영 포인트를 예제와 함께 설명합니다."
image:
  path: "/assets/img/og/redis-series-cover.svg"
  alt: "Redis 시리즈 공통 대표 이미지"
---

# Redis for 랭킹: Sorted Set으로 실시간 순위 만들기

> Redis Sorted Set으로 실시간 랭킹을 구현할 때 필요한 명령어와 운영 포인트를 예제와 함께 설명합니다.
>
> 이전 글: [Redis Pub/Sub vs Streams: 이벤트 전달을 어디에 써야 할까](/redis/pubsub-vs-streams)
> 다음 글: [Redis 분산락: SET NX PX로 시작하는 실전 가이드](/redis/distributed-lock)
> 함께 보면 좋은 글:
> - [Redis 자료구조 3: Sorted Set(ZSET), 랭킹의 정석](/redis/ds-zset)
> - [Redis 데이터 구조: String, Hash, List, Set, Sorted Set을 언제 쓸까](/redis/basis)

랭킹 기능은 보기에는 단순합니다. 점수가 높은 사람을 위에 보여주면 됩니다. 그런데 실무에서는 생각보다 고려할 것이 많습니다. 점수가 얼마나 자주 바뀌는지, 기간별 랭킹이 필요한지, 내 순위를 빠르게 보여줘야 하는지, 동점 처리는 어떻게 할지 정해야 합니다.

Redis는 이런 랭킹 기능에 잘 맞습니다. 특히 Sorted Set을 쓰면 점수 업데이트, 상위 N개 조회, 내 등수 조회가 단순해집니다.

## 기본 구조

Sorted Set은 member와 score를 함께 저장합니다.

```bash
ZADD ranking:daily:2026-06-19 1200 user:42
ZINCRBY ranking:daily:2026-06-19 50 user:42
ZREVRANGE ranking:daily:2026-06-19 0 9 WITHSCORES
ZREVRANK ranking:daily:2026-06-19 user:42
```

여기서 핵심은 `ZREVRANGE`와 `ZREVRANK`입니다. 보통 랭킹은 높은 점수가 앞이므로 reverse 계열 명령어를 많이 씁니다.

자료구조 자체 설명은 [Redis 자료구조 3: Sorted Set](/redis/ds-zset)에서 더 자세히 볼 수 있습니다.

## 기간별 key를 나누는 이유

랭킹은 거의 항상 기간 요구사항이 생깁니다.

- 오늘 랭킹
- 이번 주 랭킹
- 이번 달 랭킹
- 전체 랭킹

이걸 하나의 key에 모두 넣으면 나중에 조회와 초기화가 어려워집니다. 처음부터 기간을 key에 넣는 편이 운영하기 쉽습니다.

예:

- `ranking:daily:2026-06-19`
- `ranking:weekly:2026-W25`
- `ranking:monthly:2026-06`
- `ranking:all`

일간 랭킹은 일정 시간이 지나면 필요 없어질 수 있으므로 TTL을 줄 수 있습니다.

```bash
EXPIRE ranking:daily:2026-06-19 604800
```

## 내 순위 보여주기

서비스에서는 상위 10명만 보여주는 것보다 "내 순위"가 더 중요할 때가 많습니다. 이때 `ZREVRANK`를 쓰면 됩니다.

```bash
ZREVRANK ranking:daily:2026-06-19 user:42
ZSCORE ranking:daily:2026-06-19 user:42
```

주의할 점은 Redis rank가 0부터 시작한다는 것입니다. 사용자에게 보여줄 때는 보통 `rank + 1`을 사용합니다.

## 동점 처리

랭킹의 정책에서 동점은 꼭 정해야 합니다. Redis Sorted Set은 score가 같을 때 사전순에 가까운 내부 기준으로 정렬될 수 있습니다. 사용자가 기대하는 정책과 다를 수 있습니다.

가능한 정책:

- 같은 점수면 같은 등수로 표시
- 먼저 점수를 달성한 사람이 위
- 최근 업데이트한 사람이 위
- 별도 tie-breaker를 애플리케이션에서 처리

동점 정책이 중요하다면 score 하나로 모든 걸 해결하려고 하기보다, 보조 데이터를 같이 저장하는 것이 더 명확할 수 있습니다.

## 랭킹이 커질 때의 문제

랭킹 member가 계속 늘어나면 메모리가 증가합니다. 이벤트성 랭킹이라면 기간별 key와 TTL로 정리할 수 있지만, 상시 랭킹은 보관 정책이 필요합니다.

예:

- 상위 100,000명만 유지
- 일정 기간 활동 없는 사용자 제거
- 전체 랭킹은 DB에 저장하고 Redis는 실시간 상위권만 유지

무작정 모든 사용자를 계속 Redis에 넣는 구조는 장기적으로 부담이 됩니다.

메모리와 eviction 관점도 같이 봐야 합니다. 랭킹 key가 너무 커지고 Redis 메모리가 부족해지면, 설정에 따라 예기치 않은 key가 eviction될 수 있습니다. 랭킹이 서비스 핵심 기능이라면 [Redis 운영 심화](/redis/memory-eviction-hotkeys)에서 다룬 메모리 기준과 함께 모니터링하는 편이 안전합니다.

## DB와 Redis의 역할을 나누자

실무에서는 Redis에 랭킹만 두고 끝내기보다, 원본 이벤트는 DB나 로그 저장소에 남겨두는 경우가 많습니다. Redis는 빠른 조회용 뷰로 쓰고, 원본 기록은 나중에 재계산이나 감사가 가능하도록 보관하는 방식입니다.

이렇게 해두면 랭킹 계산식이 바뀌었을 때도 다시 만들 수 있습니다. 반대로 Redis에만 점수를 남겨두면 "왜 이 사용자가 이 점수인지"를 나중에 설명하기 어려워질 수 있습니다.

## 실무에서 자주 하는 실수

### 1) 점수 업데이트와 원본 저장을 헷갈린다

Redis 랭킹은 빠른 조회용으로 좋지만, 영구 기록의 원본까지 Redis 하나로 처리할지는 신중해야 합니다. 이벤트/점수 이력은 DB에 남기고 Redis는 랭킹 뷰로 사용하는 구조가 더 안전한 경우가 많습니다.

### 2) 랭킹 초기화를 수동으로 한다

기간별 key를 쓰면 TTL이나 스케줄러로 정리하기 쉽습니다. 하나의 key를 직접 비우는 방식은 운영 실수가 생기기 쉽습니다.

### 3) 랭킹 산정 기준이 바뀌는 경우를 고려하지 않는다

가중치나 점수 계산식이 바뀌면 기존 Redis score를 어떻게 재계산할지 정해야 합니다.

<!-- codex-category-inline-links:start -->

?ㅼ떆媛???궧? ?덉떆濡?蹂닿린?먮뒗 ?щ??덉?留? ?ㅼ젣 援ы쁽?먯꽌???뺣젹 援ъ“ ?좏깮怨??댁쁺 鍮꾩슜??媛숈씠 蹂댁? ?딆쑝硫?湲덈갑 ?⑥닚 ?곕え??癒몃Т瑜닿쾶 ?⑸땲?? 洹몃옒??[Redis ?먮즺援ъ“ 3: Sorted Set(ZSET), ??궧???뺤꽍](/redis/ds-zset), [Redis ?낅Ц ?ㅻТ??3: ?댁쁺 湲곕낯, 硫붾え由ъ? eviction??癒쇱? 蹂댁옄](/redis/practical-ops-basics), [Redis ?댁쁺 ?ы솕: 硫붾え由? eviction, ?ロ궎 ?ш퀬 ?⑦꽩](/redis/memory-eviction-hotkeys) 湲???④퍡 ?쎌뼱蹂댁떆硫?醫뗪쿋?듬땲?? ?대젃寃?媛숈씠 蹂댁떆硫???궧 援ы쁽 ?먯껜肉??꾨땲?? ?멸린 ?ㅺ? 紐곕졇?????대뼡 臾몄젣媛 ?앷만 ???덈뒗吏?????꾩떎?곸쑝濡?蹂댁씠??寃껋엯?덈떎.

<!-- codex-category-inline-links:end -->
## 정리

Redis Sorted Set은 실시간 랭킹에 아주 잘 맞습니다. 하지만 운영 가능한 랭킹을 만들려면 기간별 key, TTL, 내 순위 조회, 동점 정책, 보관 정책까지 같이 설계해야 합니다.

작게 시작할 때는 일간 랭킹 key 하나로 충분하지만, 기능이 커질 가능성이 있다면 처음부터 key 이름과 기간 정책을 명확히 두는 것이 나중에 훨씬 편합니다.

<!-- codex-category-links:start -->

## 이어서 읽어보시면 좋습니다

- [Redis 자료구조 3: Sorted Set(ZSET), 랭킹의 정석](/redis/ds-zset)
- [Redis 입문 실무형 3: 운영 기본, 메모리와 eviction을 먼저 보자](/redis/practical-ops-basics)
- [Redis 운영 심화: 메모리, eviction, 핫키 사고 패턴](/redis/memory-eviction-hotkeys)

지금 글과 바로 이어서 읽기 좋은 흐름으로 묶어두었으니, 개념을 비교해보시거나 다음 실습으로 넘어가실 때 차근차근 따라가보시면 좋겠습니다.

<!-- codex-category-links:end -->
