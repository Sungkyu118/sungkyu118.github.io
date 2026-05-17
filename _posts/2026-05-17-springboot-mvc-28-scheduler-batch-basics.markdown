---
layout: post
title: "SpringBoot 입문 28: 스케줄러로 배치 작업 시작하기"
date: 2026-05-17 22:20:00 +0900
category: SpringBoot
permalink: /springboot/mvc-scheduler-batch-basics
---

# SpringBoot 입문 28: 스케줄러로 배치 작업 시작하기

주기적으로 실행해야 하는 작업은 스케줄러를 활용하면 관리가 편하다.

## 핵심 포인트
- `@EnableScheduling`, `@Scheduled` 기본 사용법
- 고정 주기와 cron 표현식의 차이 이해
- 배치 작업은 멱등성을 고려해 설계

## 정리
스케줄러 작업은 실패 재시도와 중복 실행 방지 전략까지 함께 설계해야 안정적이다.
