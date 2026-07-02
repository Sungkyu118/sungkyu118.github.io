---
layout: post
title: "Redis 레이트 리밋: Lua로 원자성 보장하기"
date: 2026-05-15 03:00:00 +0900
category: Redis
permalink: /redis/rate-limit-lua
description: "Lua 스크립트로 Redis 레이트 리밋의 원자성을 보장하는 이유와 구현 흐름을 예제와 함께 설명합니다."
image:
  path: "/assets/img/og/redis-series-cover.svg"
  alt: "Redis 시리즈 공통 대표 이미지"
---

# Redis 레이트 리밋: Lua로 원자성 보장하기

> Lua 스크립트로 Redis 레이트 리밋의 원자성을 보장하는 이유와 구현 흐름을 예제와 함께 설명합니다.
>
> 이전 글: [Redis for 레이트 리밋: 요청 폭주와 남용을 막는 기본기](/redis/rate-limit)
> 다음 글: [Redis for 큐: List로 시작하고 Streams를 고민하는 기준](/redis/queue)
> 함께 보면 좋은 글:
> - [Redis for 레이트 리밋: 요청 폭주와 남용을 막는 기본기](/redis/rate-limit)
> - [Redis 분산락: SET NX PX로 시작하는 실전 가이드](/redis/distributed-lock)

레이트 리밋은 요청 폭주와 남용을 막는 장치입니다. 하지만 분산 환경에서 레이트 리밋을 구현할 때는 단순 카운터만으로 부족할 수 있습니다. 여러 서버가 동시에 같은 key를 증가시키고, TTL을 설정하고, limit을 비교해야 하기 때문입니다.

이때 Redis Lua 스크립트를 쓰면 여러 명령을 Redis 서버 안에서 하나의 작업처럼 실행할 수 있습니다. 핵심은 **체크와 증가를 원자적으로 처리하는 것**입니다.

기본 개념은 [Redis for 레이트 리밋](/redis/rate-limit)을 먼저 보면 더 자연스럽습니다.

## 단순 INCR + EXPIRE의 문제

가장 단순한 구현은 아래처럼 보입니다.

```bash
INCR rl:login:ip:1.2.3.4
EXPIRE rl:login:ip:1.2.3.4 60
```

처음 요청이면 TTL을 걸고, 이후 요청은 카운트만 증가시키는 방식입니다. 문제는 이 흐름이 여러 명령으로 나뉘어 있다는 점입니다.

예를 들어 네트워크 오류나 서버 중단이 `INCR`과 `EXPIRE` 사이에서 발생하면 TTL 없는 key가 남을 수 있습니다. 또 동시 요청이 몰리면 "첫 요청 판단"이 애매해질 수 있습니다.

운영 트래픽이 적을 때는 티가 안 나지만, 로그인/인증번호/검색처럼 요청이 몰리는 구간에서는 이런 작은 틈이 문제를 만들 수 있습니다.

## Lua로 Fixed Window 구현

아래 스크립트는 fixed window 방식의 기본 예시입니다.

```lua
-- KEYS[1] = rate limit key
-- ARGV[1] = limit
-- ARGV[2] = window milliseconds

local current = redis.call("INCR", KEYS[1])

if current == 1 then
  redis.call("PEXPIRE", KEYS[1], ARGV[2])
end

if current > tonumber(ARGV[1]) then
  return {0, current, redis.call("PTTL", KEYS[1])}
end

return {1, current, redis.call("PTTL", KEYS[1])}
```

반환값은 이렇게 해석할 수 있습니다.

- 첫 번째 값: 허용 여부
- 두 번째 값: 현재 카운트
- 세 번째 값: 남은 TTL

이렇게 반환하면 애플리케이션에서 429 응답과 `Retry-After` 계산을 하기 좋습니다.

## 왜 Lua가 도움이 될까

Lua 스크립트는 Redis 서버 안에서 실행됩니다. 그래서 애플리케이션과 Redis 사이의 여러 round trip을 줄이고, 관련 명령을 한 번에 처리할 수 있습니다.

장점:

- 원자성 확보
- 네트워크 왕복 감소
- 체크/증가/TTL 처리 로직을 한곳에 모음

주의점:

- 스크립트가 너무 복잡하면 Redis를 오래 붙잡을 수 있다.
- 배포 시 스크립트 버전 관리가 필요하다.
- Redis Cluster에서는 key slot을 고려해야 한다.

## WebFlux에 붙이는 흐름

Spring WebFlux에서는 보통 `WebFilter`에서 레이트 리밋을 검사합니다.

흐름:

1. 요청에서 사용자 ID, IP, endpoint를 추출한다.
2. Redis key를 만든다.
3. Lua 스크립트를 실행한다.
4. 허용이면 다음 필터로 넘긴다.
5. 초과면 429 응답을 반환한다.

key 예:

- `rl:login:ip:1.2.3.4`
- `rl:sms:phone:01012345678`
- `rl:search:user:42`

key 설계는 [Redis 입문 실무형 2: 키 설계와 TTL](/redis/practical-key-ttl)과도 연결됩니다.

## 429 응답은 친절해야 한다

레이트 리밋에 걸렸을 때는 단순히 실패만 내려주면 클라이언트가 계속 재시도할 수 있습니다. 가능하면 남은 대기 시간을 알려주는 편이 좋습니다.

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
```

사용자 메시지도 중요합니다. "요청이 너무 많습니다"보다 "잠시 후 다시 시도해주세요"처럼 다음 행동을 알려주는 문구가 낫습니다.

## 운영에서 자주 하는 실수

### 1) IP만 기준으로 제한한다

IP 기준은 간단하지만, 같은 네트워크를 쓰는 여러 사용자가 함께 제한될 수 있습니다. 로그인 전에는 IP가 필요할 수 있지만, 로그인 후에는 userId 기준이 더 적절한 경우가 많습니다.

### 2) Lua 스크립트를 너무 복잡하게 만든다

Redis는 빠르지만 싱글 스레드 이벤트 루프 특성이 있습니다. 오래 걸리는 Lua 스크립트는 Redis 전체 latency에 영향을 줄 수 있습니다.

### 3) limit 조정 로그가 없다

레이트 리밋은 운영하면서 조정하는 기능입니다. 어떤 key가 얼마나 자주 막히는지 로그가 없으면 정상 사용자와 비정상 트래픽을 구분하기 어렵습니다.

<!-- codex-category-inline-links:start -->

Lua를 붙이는 순간 레이트 리밋 구현이 갑자기 어려워 보일 수 있지만, 사실 핵심은 "여러 연산을 한 번에 안전하게 묶어야 한다"는 점입니다. [Redis for 레이트 리밋: 요청 폭주와 남용을 막는 기본기](/redis/rate-limit), [Redis 분산락: SET NX PX로 시작하는 실전 가이드](/redis/distributed-lock), [Redis 입문 실무형 2: 키 설계와 TTL, 운영에서 덜 망하는 법](/redis/practical-key-ttl) 글을 함께 읽어보시면, 원자성이라는 개념이 왜 분산락과 레이트 리밋에서 반복해서 중요하게 등장하는지 더 자연스럽게 이해하실 수 있습니다.

<!-- codex-category-inline-links:end -->
## 정리

Redis Lua는 레이트 리밋에서 체크, 증가, TTL 설정을 원자적으로 묶는 데 유용합니다. 단순한 fixed window도 Lua로 처리하면 운영 중 애매한 race condition을 줄일 수 있습니다.

다만 Lua 자체가 목적은 아닙니다. 중요한 것은 어떤 기준으로 제한할지, 초과 시 어떻게 응답할지, 정상 사용자를 얼마나 보호할지입니다. 이 기준이 있어야 레이트 리밋이 보안 장치이면서도 사용자 경험을 해치지 않습니다.

<!-- codex-category-links:start -->

## 이어서 읽어보시면 좋습니다

- [Redis for 레이트 리밋: 요청 폭주와 남용을 막는 기본기](/redis/rate-limit)

  Lua 기반 구현을 보기 전에 기본 레이트 리밋 흐름을 다시 정리하면 훨씬 이해가 쉽습니다. 기본 글을 함께 읽으면 카운터, 시간 창, TTL이라는 뼈대를 먼저 잡은 뒤 Lua가 어느 지점을 더 안전하게 만들어주는지 분명하게 볼 수 있습니다.

- [Redis 분산락: SET NX PX로 시작하는 실전 가이드](/redis/distributed-lock)

  Lua와 분산락은 모두 동시성 상황에서 여러 요청이 같은 키를 건드릴 때의 위험을 줄이는 데 관심이 있습니다. 분산락 글을 함께 읽으면 원자성이라는 개념이 레이트 리밋뿐 아니라 여러 서버가 공유 자원을 다루는 전반적인 문제로 확장된다는 점을 이해할 수 있습니다.

- [Redis 입문 실무형 2: 키 설계와 TTL, 운영에서 덜 망하는 법](/redis/practical-key-ttl)

  Lua로 카운터를 안전하게 올리더라도 키 이름과 TTL 정책이 어설프면 운영에서 해석하기 어려운 제한 규칙이 됩니다. 키 설계 글을 함께 보면 원자적인 실행과 별개로, 제한 기준을 어떤 키로 표현하고 언제 만료시킬지까지 함께 설계해야 한다는 점을 잡을 수 있습니다.

위 글들은 지금 읽은 내용과 바로 이어지는 흐름으로 묶어두었습니다. 천천히 따라가시면 개념을 따로 외우는 느낌보다, Redis를 실제 서비스 요구사항에 맞게 고르고 운영하는 감각으로 자연스럽게 이어가실 수 있습니다.

<!-- codex-category-links:end -->
