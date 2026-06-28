---
layout: post
title: "Redis 입문 실무형 3: 운영 기본, 메모리와 eviction을 먼저 보자"
date: 2026-05-15 00:07:00 +0900
category: Redis
permalink: /redis/practical-ops-basics
description: "Redis 메모리, eviction, persistence, 모니터링의 기본을 초보자도 따라올 수 있게 실무 흐름으로 정리합니다."
image:
  path: "/assets/img/og/redis-series-cover.svg"
  alt: "Redis 시리즈 공통 대표 이미지"
---

# Redis 입문 실무형 3: 운영 기본, 메모리와 eviction을 먼저 보자

> Redis 메모리, eviction, persistence, 모니터링의 기본을 초보자도 따라올 수 있게 실무 흐름으로 정리합니다.
>
> 이전 글: [Redis 입문 실무형 2: 키 설계와 TTL, 운영에서 덜 망하는 법](/redis/practical-key-ttl)
> 다음 글: [Redis 데이터 구조: String, Hash, List, Set, Sorted Set을 언제 쓸까](/redis/basis)
> 함께 보면 좋은 글:
> - [Redis 운영 심화: 메모리, eviction, 핫키 사고 패턴](/redis/memory-eviction-hotkeys)
> - [Redis 입문 실무형 2: 키 설계와 TTL, 운영에서 덜 망하는 법](/redis/practical-key-ttl)

Redis는 개발 환경에서는 정말 조용합니다. 캐시도 잘 되고, 응답도 빠르고, 문제 없어 보입니다. 그런데 운영에 올라가면 이야기가 달라집니다. 갑자기 hit rate가 떨어지거나, 메모리가 급증하거나, 특정 시간대에 DB 부하가 치솟는 식으로 문제가 드러납니다.

이럴 때 핵심은 복잡한 튜닝보다 먼저 **어디를 봐야 하는지 아는 것**입니다. Redis 운영에서 가장 먼저 잡아야 할 감각은 메모리, hit/miss, latency, eviction입니다.

## Redis 운영에서 가장 먼저 보는 것: 메모리

Redis는 기본적으로 메모리 기반 저장소입니다. 그래서 메모리 사용량이 올라가는 패턴을 모른 채 운영하면, 어느 날 갑자기 eviction이 시작되고 그때부터 장애가 전파되기 쉽습니다.

운영에서 자주 보는 흐름은 이렇습니다.

1. 캐시를 붙인 뒤 초반 성능은 좋아진다.
2. 키 수와 값 크기가 계속 증가한다.
3. 어느 시점부터 메모리 여유가 줄어든다.
4. `maxmemory`에 가까워지면 eviction이 시작된다.
5. 캐시 적중률이 흔들리고, 원본 DB/API 부하가 커진다.

그래서 Redis 운영은 결국 "메모리를 얼마나 쓰고 있는가"를 보는 일에서 출발합니다.

## eviction은 설정이 아니라 사고 방식이다

eviction은 메모리가 부족할 때 어떤 키를 버릴지 정하는 정책입니다. 설정 이름만 외우는 것보다, "우리 서비스에서 어떤 키가 버려져도 괜찮은가?"를 먼저 생각하는 편이 맞습니다.

대표 정책 예:

- `allkeys-lru`: 전체 키 중 최근 덜 사용된 키부터 제거
- `volatile-lru`: TTL이 있는 키 중 최근 덜 사용된 키부터 제거
- `volatile-ttl`: TTL이 짧은 키부터 우선 제거

중요한 점은 eviction이 시작됐다는 사실 자체가 이미 주의 신호라는 것입니다. "정책이 있으니 괜찮다"가 아니라, 왜 여기까지 왔는지 원인을 봐야 합니다.

더 자세한 사례는 [Redis 운영 심화: 메모리, eviction, 핫키 사고 패턴](/redis/memory-eviction-hotkeys)에서 이어집니다.

## hit/miss 비율은 캐시가 실제로 일하고 있는지 보여준다

캐시를 붙였다고 해서 자동으로 효과가 생기지는 않습니다. hit가 충분히 나지 않으면 Redis는 운영 복잡도만 늘리고 성능 이득은 거의 못 주는 구조가 됩니다.

hit/miss 비율을 볼 때는 단순 숫자보다 맥락이 중요합니다.

- TTL이 너무 짧아서 miss가 많은가?
- 사용자별 데이터라 재사용성이 낮은가?
- 캐시 키 설계가 잘못되어 같은 값을 반복 저장하고 있는가?
- 특정 시점에 만료가 몰려 miss가 폭증하는가?

이런 문제는 [키 설계와 TTL](/redis/practical-key-ttl), [Redis for 캐시](/redis/cache)와 같이 봐야 원인이 더 잘 보입니다.

## hot key는 "Redis가 느리다"처럼 보이는 대표 원인이다

모든 요청이 특정 키 하나에 몰리면 Redis 전체가 느려진 것처럼 보일 수 있습니다. 사실은 특정 키 하나가 병목을 만들고 있는 경우가 많습니다.

예:

- `feed:home` 같은 공용 피드
- 이벤트 중인 인기 상품 정보
- 로그인 제한 키처럼 특정 엔드포인트에 집중되는 키

증상:

- 특정 시간대에 latency 급증
- hit rate는 나쁘지 않은데 체감 속도는 느림
- 일부 서버나 shard에 트래픽이 유독 몰림

이런 경우에는 단순 증설보다 키 성격과 트래픽 패턴을 먼저 봐야 합니다.

## 만료가 동시에 몰리는 구조는 피해야 한다

운영에서 자주 놓치는 부분이 "TTL을 잘 줬다"에서 끝나는 경우입니다. 문제는 키들이 같은 시점에 생성되고 같은 시점에 만료되면, TTL을 잘 줘도 실제로는 부하가 한 번에 몰릴 수 있다는 점입니다.

대표적인 완화 방법:

- TTL에 작은 랜덤 지연을 추가한다.
- 아주 비싼 키는 백그라운드 재생성 전략을 고려한다.
- hot key는 별도 갱신 전략을 둔다.

이런 구조를 미리 넣어두면 캐시 스탬피드와 원본 DB 급증을 많이 줄일 수 있습니다.

## 최소한 봐야 하는 운영 지표

처음부터 대단한 대시보드를 만들 필요는 없습니다. 대신 아래 지표는 꾸준히 보는 편이 좋습니다.

- 메모리 사용량
- keyspace hit/miss
- latency
- 연결 수
- eviction 발생 여부
- 키 수 증가 추세

이 중에서 특히 `메모리 사용량 + eviction + hit/miss` 조합은 Redis 건강 상태를 빠르게 파악하는 데 도움이 됩니다.

## 자주 생기는 실수

### 1) Redis가 빨라서 모니터링이 덜 필요하다고 생각한다

처음에는 빠르기 때문에 방심하기 쉽습니다. 하지만 Redis 장애는 보통 "갑자기" 오는 것처럼 보일 뿐, 실제로는 메모리 증가나 hit rate 악화가 먼저 쌓여 있었던 경우가 많습니다.

### 2) eviction이 시작돼도 당장 장애가 아니라 넘긴다

eviction은 단순 로그가 아니라 서비스 품질이 흔들리기 시작하는 신호일 수 있습니다. 특히 캐시 적중률과 DB 부하가 같이 흔들리면 바로 봐야 합니다.

### 3) hot key를 애플리케이션 문제로만 본다

애플리케이션 코드도 중요하지만, Redis 키 설계와 만료 전략이 원인인 경우도 많습니다. 그래서 코드와 운영 지표를 같이 봐야 합니다.

<!-- codex-category-inline-links:start -->

지금 읽고 계신 주제가 아직 조금 추상적으로 느껴지신다면 [Redis 운영 심화: 메모리, eviction, 핫키 사고 패턴](/redis/memory-eviction-hotkeys), [Redis for 캐시: 언제 효과가 크고, 어디서 망가지는가](/redis/cache), [Redis for 랭킹: Sorted Set으로 실시간 순위 만들기](/redis/ranking) 글도 함께 읽어보시면 좋겠습니다. 같은 Redis 흐름 안에서 앞단의 배경과 다음 단계의 확장 포인트를 같이 보실 수 있어서, 지금 배우는 내용이 실제 프로젝트에서 어디에 연결되는지 훨씬 더 선명하게 이해하실 수 있습니다.

<!-- codex-category-inline-links:end -->
## 정리

Redis 운영의 출발점은 복잡한 명령어가 아니라 **메모리, hit/miss, latency, eviction을 꾸준히 보는 습관**입니다.

캐시가 잘 되는지, 메모리가 위험해지는지, 특정 키가 병목인지가 보이기 시작하면 Redis는 훨씬 다루기 쉬운 도구가 됩니다. 반대로 이 감각 없이 운영하면 "원래 빠르던 Redis가 왜 갑자기 장애를 만들었지?"라는 상황을 자주 겪게 됩니다.

다음 단계에서는 메모리/eviction/hot key 사고 패턴을 더 구체적으로 보는 [Redis 운영 심화: 메모리, eviction, 핫키 사고 패턴](/redis/memory-eviction-hotkeys)으로 넘어가면 좋습니다.

<!-- codex-category-links:start -->

## 이어서 읽어보시면 좋습니다

- [Redis 운영 심화: 메모리, eviction, 핫키 사고 패턴](/redis/memory-eviction-hotkeys)
- [Redis for 캐시: 언제 효과가 크고, 어디서 망가지는가](/redis/cache)
- [Redis for 랭킹: Sorted Set으로 실시간 순위 만들기](/redis/ranking)

지금 글과 바로 이어서 읽기 좋은 흐름으로 묶어두었으니, 개념을 비교해보시거나 다음 실습으로 넘어가실 때 차근차근 따라가보시면 좋겠습니다.

<!-- codex-category-links:end -->
