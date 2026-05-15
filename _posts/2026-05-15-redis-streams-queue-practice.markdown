---
layout: post
title: "Redis Streams 실전: 작업 큐로 쓰는 법 (consumer group, 재처리 감각)"
date: 2026-05-15 03:20:00 +0900
category: Redis
permalink: /redis/streams-queue
---

# Redis Streams 실전: 작업 큐로 쓰는 법 (consumer group, 재처리 감각)

Redis를 "큐"로 쓰려면, List보다 Streams가 더 적합한 경우가 많습니다. Streams는 소비자 그룹, pending(처리중) 개념이 있어서 **유실/재처리**를 설계할 수 있어요.

이 글은 다음을 다룹니다.

1. List 큐 vs Streams 큐 차이(트레이드오프)
2. Streams 핵심 개념(consumer group, pending)
3. 실전에서 꼭 필요한 재처리/운영 감각

## 1) List vs Streams 선택 기준

### List가 맞는 경우

- "가벼운 비동기 분리"가 목적
- 유실이 아주 치명적이지 않음
- 단순 구현이 최우선

### Streams가 맞는 경우

- 작업 유실을 줄이고 싶다
- 여러 소비자가 같은 스트림을 나눠 처리해야 한다(consumer group)
- 처리 실패/재처리가 필요하다(pending + ack)

## 2) Streams의 핵심 개념

### 생산: XADD

스트림에 이벤트/작업을 추가합니다.

예:

- stream: `jobs:email`
- fields: `to`, `template`, `payload`

### 소비: XREADGROUP

consumer group 단위로 메시지를 읽습니다. 같은 group 내에서 여러 consumer가 나눠서 처리할 수 있어요.

### 완료: XACK

성공적으로 처리한 메시지는 ack로 완료 처리합니다. ack가 없으면 pending으로 남습니다.

## 3) 실전 설계 포인트

### (1) 재처리 전략을 먼저 정하기

실패했을 때:

- 몇 번까지 재시도할지
- 재시도 간격(즉시/지연)
- 최종 실패는 어디로 보낼지(DLQ 비슷한 스트림)

### (2) pending 모니터링

pending이 계속 쌓이면 소비자가 죽었거나, 처리 로직이 느리거나, 예외가 반복되는 겁니다. 운영에서는 "pending이 늘어나는가"를 중요한 신호로 봅니다.

### (3) 메시지 크기

Streams에 큰 payload를 그대로 넣으면 메모리/네트워크 비용이 커집니다. 보통은:

- payload는 ID로 두고
- 실제 데이터는 DB/오브젝트 스토리지에 둔 뒤
- consumer가 ID로 조회

같은 패턴이 운영이 편합니다.

## 4) 흔한 실수

- ack를 안 해서 pending이 계속 쌓임
- 소비자 그룹을 만들었지만, 소비자 장애 시 reclaim(재할당)을 설계 안 함
- Streams에 너무 큰 데이터를 넣어 메모리 비용 급증

## 정리

- "작업 큐"라면 Streams가 List보다 실전 기능이 풍부하다
- consumer group + ack + pending이 핵심
- 재처리/운영(모니터링) 설계를 같이 해야 시스템이 안정된다

