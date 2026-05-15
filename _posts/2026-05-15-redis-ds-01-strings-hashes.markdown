---
layout: post
title: "Redis 자료구조 1: String과 Hash (실무 모델링 감각)"
date: 2026-05-15 00:08:00 +0900
category: Redis
permalink: /redis/ds-strings-hashes
---

# Redis 자료구조 1: String과 Hash (실무 모델링 감각)

Redis는 "자료구조 서버"라는 말이 더 정확할 때가 많습니다. 어떤 자료구조를 선택하느냐가 곧 성능/메모리/기능을 결정해요.

이 글은 String/Hash의 실무 감각을 정리합니다.

## 1) String: 가장 기본, 가장 강력

String은 "값" 하나를 저장합니다.

실무에서 흔한 용도:

- 캐시 값(JSON 문자열)
- 카운터(INCR)
- 토큰/세션 ID 매핑

대표 명령:

- `SET key value EX 60`
- `GET key`
- `INCR key`

카운터 예:

- 조회수/좋아요 수 같은 증가 값에 잘 맞습니다.

## 2) Hash: 필드 단위로 쪼개 저장

Hash는 `key` 아래에 `field -> value`를 저장합니다.

실무에서 좋은 경우:

- 객체의 일부 필드만 자주 갱신한다
- JSON 전체를 다시 쓰는 비용을 줄이고 싶다

대표 명령:

- `HSET user:1 name "Sungkyu" age "20"`
- `HGET user:1 name`
- `HGETALL user:1`

## 3) String(JSON) vs Hash(필드) 선택 기준

String(JSON)이 좋은 경우:

- 값 전체를 통째로 읽고 쓴다
- 직렬화/역직렬화로 충분하다

Hash가 좋은 경우:

- 필드 단위로 부분 업데이트가 많다
- 객체가 크고, 일부 필드만 자주 바뀐다

주의:

- Hash라고 무조건 "정답"은 아닙니다. 앱 코드가 복잡해질 수 있어요.

## 4) 흔한 실수

- 키 네이밍이 중구난방이라 운영 중에 무슨 키인지 모름
- TTL을 안 걸어서 데이터가 영원히 쌓임
- 객체 전체를 너무 큰 JSON으로 저장(메모리/네트워크 비용)

## 정리

- String은 캐시/카운터/토큰에 매우 자주 쓰인다
- Hash는 "부분 업데이트"가 잦을 때 빛난다
- 키 설계와 TTL이 같이 따라와야 운영이 안정적이다

