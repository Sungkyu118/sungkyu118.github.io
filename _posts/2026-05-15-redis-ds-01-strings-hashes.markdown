---
layout: post
title: "Redis 자료구조 1: String과 Hash, 실무 모델링 감각"
date: 2026-05-15 00:08:00 +0900
category: Redis
permalink: /redis/ds-strings-hashes
description: "Redis String과 Hash의 차이, 키 설계, 조회와 갱신 패턴을 실무 모델링 관점에서 설명합니다."
image:
  path: "/assets/img/og/redis-series-cover.svg"
  alt: "Redis 시리즈 공통 대표 이미지"
---

# Redis 자료구조 1: String과 Hash, 실무 모델링 감각

> Redis String과 Hash의 차이, 키 설계, 조회와 갱신 패턴을 실무 모델링 관점에서 설명합니다.
>
> 이전 글: [Redis 데이터 구조: String, Hash, List, Set, Sorted Set을 언제 쓸까](/redis/basis)
> 다음 글: [Redis 자료구조 2: List와 Set, 순서와 중복을 다루는 법](/redis/ds-lists-sets)
> 함께 보면 좋은 글:
> - [Redis 데이터 구조: String, Hash, List, Set, Sorted Set을 언제 쓸까](/redis/basis)
> - [Redis 입문 실무형 2: 키 설계와 TTL, 운영에서 덜 망하는 법](/redis/practical-key-ttl)

Redis에서 가장 자주 만나는 자료구조는 String과 Hash입니다. 둘 다 "값을 저장한다"는 점에서는 비슷해 보이지만, 실무에서는 꽤 다른 문제를 풀 때 사용합니다.

String은 통째로 저장하고 통째로 읽는 데 좋고, Hash는 하나의 객체를 여러 필드로 나누어 관리할 때 좋습니다. 이 차이를 모르면 모든 것을 JSON String으로 저장하게 되고, 나중에 일부 필드만 바꾸거나 카운터를 증가시킬 때 불편해집니다.

## String은 Redis의 기본 단위다

String은 Redis에서 가장 단순한 구조입니다.

```bash
SET app:notice "점검 예정" EX 600
GET app:notice
```

숫자도 String으로 저장되지만 Redis 명령어로 증가/감소가 가능합니다.

```bash
INCR view:post:100
INCRBY stock:product:9001 -1
```

String이 잘 맞는 경우는 다음과 같습니다.

- 응답 JSON을 통째로 캐시할 때
- 조회 수, 요청 수 같은 숫자 카운터
- 간단한 feature flag나 설정값
- 짧은 TTL을 가진 임시 상태

캐시 용도로 쓸 때는 [Redis for 캐시](/redis/cache)와 같이 보면 좋습니다.

## String 사용 시 주의할 점

String은 단순해서 좋지만, 너무 큰 값을 넣기 시작하면 운영이 힘들어집니다. 예를 들어 상품 상세 JSON이 점점 커지고, 사용자별로 다른 값을 많이 저장하면 메모리 사용량이 빠르게 늘 수 있습니다.

또한 JSON 안의 일부 필드만 바꾸고 싶어도 전체 값을 다시 읽고, 파싱하고, 수정하고, 저장해야 합니다. 이 과정에서 동시성 문제가 생길 수도 있습니다.

그래서 이런 질문을 해봐야 합니다.

- 이 값은 항상 통째로 읽는가?
- 일부 필드만 자주 바뀌는가?
- 숫자 증가처럼 Redis 명령어로 처리할 수 있는가?
- 값 크기가 너무 커지지 않는가?

## Hash는 객체를 필드 단위로 다룰 때 좋다

Hash는 하나의 key 안에 여러 field/value를 넣습니다.

```bash
HSET user:42 name "kim" grade "gold" loginCount 3
HGET user:42 grade
HINCRBY user:42 loginCount 1
HGETALL user:42
```

사용자 프로필처럼 여러 속성이 있는 데이터를 다룰 때 유용합니다. 특히 일부 필드만 자주 읽거나 바꾸는 경우 String JSON보다 깔끔할 수 있습니다.

예를 들어 로그인 횟수만 증가시킨다면 JSON 전체를 다시 저장할 필요 없이 `HINCRBY`로 처리할 수 있습니다.

## Hash가 잘 맞는 상황

Hash는 이런 상황에서 좋습니다.

- 사용자 프로필 일부 필드를 자주 읽는다.
- 상품 상태 중 일부 값만 바뀐다.
- 설정값 묶음을 하나의 key 아래에서 관리하고 싶다.
- 숫자 필드를 개별적으로 증가/감소해야 한다.

다만 Hash도 RDB의 테이블을 대체하는 도구는 아닙니다. Redis는 여전히 메모리 기반이고, TTL/eviction/장애 상황을 고려해야 합니다. 핵심 원본 데이터는 보통 DB에 두고, Redis는 빠른 조회나 임시 상태 저장으로 쓰는 편이 안정적입니다.

## String JSON vs Hash 선택 기준

간단하게 말하면:

- 통째로 읽고 통째로 쓰면 String JSON
- 필드 일부만 읽거나 바꾸면 Hash
- 숫자 카운터 하나면 String
- 객체 안 숫자 필드를 자주 바꾸면 Hash

예를 들어 상품 상세 페이지 응답 전체를 캐시한다면 String JSON이 편합니다. 반면 사용자 상태, 등급, 로그인 횟수처럼 필드별로 접근하는 데이터는 Hash가 더 자연스럽습니다.

## 운영에서 자주 하는 실수

### 1) 모든 객체를 큰 JSON String으로 저장한다

처음에는 빠릅니다. 하지만 필드 단위 변경이 많아질수록 코드가 복잡해지고, 값 크기가 커질수록 메모리 비용도 커집니다.

### 2) Hash에 너무 많은 field를 넣는다

Hash도 무한히 편한 구조는 아닙니다. field가 지나치게 많거나, 동적으로 계속 늘어나는 구조라면 key 설계부터 다시 봐야 합니다.

### 3) TTL 전략 없이 저장한다

String이든 Hash든 TTL 없이 계속 쌓으면 메모리가 차오릅니다. 키 설계와 TTL은 [Redis 입문 실무형 2](/redis/practical-key-ttl)에서 다룬 기준을 같이 적용해야 합니다.

<!-- codex-category-inline-links:start -->

지금 읽고 계신 주제가 아직 조금 추상적으로 느껴지신다면 [Redis 입문 실무형 2: 키 설계와 TTL, 운영에서 덜 망하는 법](/redis/practical-key-ttl), [Redis for 세션: 여러 서버에서 로그인 상태를 공유하는 법](/redis/session), [Redis for 캐시: 언제 효과가 크고, 어디서 망가지는가](/redis/cache) 글도 함께 읽어보시면 좋겠습니다. 같은 Redis 흐름 안에서 앞단의 배경과 다음 단계의 확장 포인트를 같이 보실 수 있어서, 지금 배우는 내용이 실제 프로젝트에서 어디에 연결되는지 훨씬 더 선명하게 이해하실 수 있습니다.

<!-- codex-category-inline-links:end -->
## 정리

String과 Hash의 차이는 "저장할 수 있느냐"가 아니라 "어떻게 읽고 바꿀 것이냐"입니다.

String은 단순 캐시와 카운터에 강하고, Hash는 필드 단위 객체 관리에 강합니다. Redis 자료구조를 잘 고르면 애플리케이션 코드가 단순해지고, 잘못 고르면 운영 중에 캐시 무효화와 메모리 문제가 더 어려워집니다.

<!-- codex-category-links:start -->

## 이어서 읽어보시면 좋습니다

- [Redis 입문 실무형 2: 키 설계와 TTL, 운영에서 덜 망하는 법](/redis/practical-key-ttl)
- [Redis for 세션: 여러 서버에서 로그인 상태를 공유하는 법](/redis/session)
- [Redis for 캐시: 언제 효과가 크고, 어디서 망가지는가](/redis/cache)

지금 글과 바로 이어서 읽기 좋은 흐름으로 묶어두었으니, 개념을 비교해보시거나 다음 실습으로 넘어가실 때 차근차근 따라가보시면 좋겠습니다.

<!-- codex-category-links:end -->
