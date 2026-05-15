---
layout: post
title: "Redis 레이트리밋: Lua로 원자성 보장하기 (WebFlux 적용 아이디어)"
date: 2026-05-15 03:00:00 +0900
category: Redis
permalink: /redis/rate-limit-lua
---

# Redis 레이트리밋: Lua로 원자성 보장하기 (WebFlux 적용 아이디어)

레이트리밋은 "요청 폭주로부터 시스템을 지키는 마지막 안전장치"입니다. Redis로 레이트리밋을 구현할 때 제일 중요한 건 **원자성**이에요. 분산 환경에서 "체크 + 증가"가 레이스로 깨지면 레이트리밋은 장식이 됩니다.

이 글은 다음을 다룹니다.

1. 왜 Lua가 필요한지(트레이드오프)
2. Fixed window 레이트리밋을 Lua로 원자 처리
3. WebFlux에 어떻게 붙일지(필터/키 설계)
4. 운영 포인트(키 폭발, TTL, 429)

## 1) 왜 Lua인가

### 문제: INCR와 EXPIRE를 따로 하면 레이스가 날 수 있음

가장 단순한 구현은 `INCR`로 카운트를 올리고, 첫 요청일 때만 `EXPIRE`를 거는 방식입니다. 하지만 "첫 요청" 판단이 동시 요청에서 흔들릴 수 있고, 네트워크 왕복(RTT)이 늘어나면 더 취약해질 수 있어요.

Lua 스크립트는 Redis 서버 내부에서 실행되므로, 여러 명령을 **한 덩어리로 원자 실행**할 수 있습니다.

### 트레이드오프

- 장점: 원자성 확보, RTT 감소, 구현이 한 곳에 모임
- 단점: 스크립트 운영(버전/테스트)이 필요, 스크립트가 너무 복잡해지면 디버깅 난이도 증가

## 2) Lua: Fixed Window 레이트리밋 예제

요구사항 예:

- 1분에 최대 30회 허용
- 초과하면 거절(429)

키는 예를 들어 `rl:login:{ip}`처럼 엔드포인트 + 식별자로 만듭니다.

Lua 스크립트(개념):

- `INCR key`
- 결과가 1이면 `PEXPIRE key windowMs`
- 카운트가 limit 초과면 거절

```lua
-- KEYS[1] = key
-- ARGV[1] = limit
-- ARGV[2] = window_ms

local current = redis.call("INCR", KEYS[1])
if current == 1 then
  redis.call("PEXPIRE", KEYS[1], ARGV[2])
end

if current > tonumber(ARGV[1]) then
  return {0, current}  -- blocked
end

return {1, current}    -- allowed
```

반환값을 `{allowed, currentCount}` 형태로 두면 애플리케이션에서 로깅/헤더 구성에 유리합니다.

## 3) WebFlux에 붙이는 위치(아이디어)

대부분은 `WebFilter`에서 처리합니다.

- 사용자 식별: 로그인 전이면 IP, 로그인 후면 userId
- 키 구성: `rl:{endpoint}:{id}`
- 초과 시: 429 + 필요하면 `Retry-After` 헤더

키 설계 팁:

- 엔드포인트별로 분리: `rl:login:{ip}`, `rl:sms:{phone}`, `rl:search:{userId}`
- "공격 표면"이 큰 곳부터 적용(로그인/가입/인증번호)

## 4) 흔한 실수/운영 포인트

### (1) 키 폭발

IP 기반 레이트리밋을 무차별 적용하면 키가 아주 많이 생길 수 있습니다.

- TTL을 반드시 설정
- 엔드포인트를 최소화(정말 필요한 곳부터)
- 익명 사용자는 IP 외에 User-Agent/디바이스 토큰 등 추가 식별을 고려(상황에 따라)

### (2) limit 정책이 너무 빡셈

정상 사용자가 막히면 "보안 강화"가 아니라 "서비스 품질 저하"입니다. 운영 중에 조정할 수 있도록 설정값(초당/분당)을 외부화하는 편이 좋습니다.

### (3) 초과 응답을 일관되게

429로 통일하고, 프론트/클라이언트에서 재시도 정책을 갖게 만들면 장애 전파를 줄일 수 있습니다.

## 정리

- 레이트리밋은 원자성이 핵심이고, Lua가 그걸 깔끔하게 해결해준다
- 키 설계/TTL/정책 조정이 운영 품질을 좌우한다
- WebFlux에서는 필터 계층에 붙이는 게 실전에서 가장 단순하다

