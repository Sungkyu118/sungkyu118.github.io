---
layout: post
title: "Redis for 세션"
date: 2026-05-15 02:10:00 +0900
category: Redis
permalink: /redis/session
---

# Redis for 세션

WebFlux에서 "로그인 상태 유지"를 구현할 때 Redis는 세션 저장소로 자주 선택됩니다. 서버가 여러 대로 늘어나는 순간, 메모리 세션은 바로 한계를 드러내니까요.

이 글은 다음을 목표로 합니다.

1. 세션을 Redis에 두는 이유(트레이드오프)
2. WebFlux에서 Spring Session + Redis 개념 잡기
3. TTL/보안/운영에서 주의할 점

## 1) 언제/왜 쓰나 (트레이드오프)

### Redis 세션을 쓰는 이유

- 서버가 여러 대일 때 세션 공유 필요(스케일 아웃)
- 재시작/배포 시 세션 유지(정책에 따라)
- 세션 TTL을 중앙에서 관리

### 단점/주의점

- Redis 장애가 곧 인증/세션 장애로 이어질 수 있음(구성 중요)
- 세션에 큰 데이터를 넣으면 Redis 메모리를 빠르게 잡아먹음
- 쿠키 설정(도메인, SameSite, Secure)까지 같이 설계해야 함

## 2) Spring Session + WebFlux 개념

Spring Session은 WebFlux에서 `WebSession`을 Redis에 저장하도록 연결해줍니다. 핵심은 "세션 ID는 쿠키로, 세션 데이터는 Redis로"입니다.

## 3) 의존성(개념)

보통 아래 조합으로 갑니다.

- `spring-boot-starter-data-redis-reactive`
- `spring-session-data-redis`

그리고 Redis 연결은 `application.yml`에서 잡습니다.

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
```

## 4) 세션에 무엇을 넣을까

권장:

- 사용자 식별자(userId)
- 권한/역할(role) 같은 작은 값

비권장:

- 큰 프로필 전체 JSON
- 화면 렌더링에 필요한 대량 데이터

세션은 "필요 최소한"이 좋습니다. 나머지는 캐시/DB로 분리하세요.

## 5) TTL/로그아웃 정책

세션 TTL은 보통 "보안"과 "UX" 사이의 타협입니다.

- 너무 짧으면 사용자 재로그인 불편
- 너무 길면 탈취 시 피해가 커짐

로그아웃은 "서버 세션 삭제 + 쿠키 만료"까지 함께 이루어져야 합니다.

## 6) 운영 포인트

- Redis 메모리 사용량과 eviction 정책 확인
- 세션 키가 너무 많아지는지(유령 세션) 모니터링
- Redis 장애 시 fallback(로그인 차단/일시 오류 처리) 정책 정하기

## 정리

- WebFlux에서도 세션은 Redis로 중앙화할 수 있다(Spring Session)
- 세션에는 최소 데이터만 저장
- TTL/쿠키/장애 대응까지 한 묶음으로 설계해야 안정적이다

