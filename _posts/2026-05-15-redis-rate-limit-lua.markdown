---
layout: post
title: "Redis 레이트 리밋: Lua로 원자성 보장하기"
date: 2026-05-15 03:00:00 +0900
category: Redis
permalink: /redis/rate-limit-lua
---

# Redis 레이트 리밋: Lua로 원자성 보장하기

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

## 정리

Redis Lua는 레이트 리밋에서 체크, 증가, TTL 설정을 원자적으로 묶는 데 유용합니다. 단순한 fixed window도 Lua로 처리하면 운영 중 애매한 race condition을 줄일 수 있습니다.

다만 Lua 자체가 목적은 아닙니다. 중요한 것은 어떤 기준으로 제한할지, 초과 시 어떻게 응답할지, 정상 사용자를 얼마나 보호할지입니다. 이 기준이 있어야 레이트 리밋이 보안 장치이면서도 사용자 경험을 해치지 않습니다.
