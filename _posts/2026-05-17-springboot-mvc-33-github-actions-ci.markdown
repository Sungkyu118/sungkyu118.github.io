---
layout: post
title: "SpringBoot 입문 33: GitHub Actions로 CI 자동화"
date: 2026-05-17 22:20:00 +0900
category: SpringBoot
permalink: /springboot/mvc-github-actions-ci
description: "Spring Boot 프로젝트에 GitHub Actions CI를 붙여 push와 pull request마다 Gradle 빌드와 테스트를 자동 실행하는 방법을 입문자 기준으로 정리합니다."
image: "/assets/img/favicons/android-chrome-512x512.png"
---

# SpringBoot 입문 33: GitHub Actions로 CI 자동화

> 이 글에서는 Spring Boot 프로젝트를 GitHub Actions로 자동 빌드하고 테스트해서, 푸시와 PR 단계에서 문제를 먼저 잡는 흐름을 설명합니다.
>
> 이전 글: [SpringBoot 입문 32: Testcontainers로 MySQL 통합 테스트](/springboot/mvc-testcontainers-mysql)
> 다음 글: 없음
> 함께 보면 좋은 글:
> - [SpringBoot 입문 12: 단위 테스트(JUnit5 + Mockito)로 Service 검증하기](/springboot/mvc-unit-test-junit-mockito)
> - [SpringBoot 입문 23: Docker로 실행하기(가장 단순한 Dockerfile)](/springboot/mvc-dockerize-basics)

푸시/PR마다 빌드와 테스트를 자동 실행하면 품질 관리가 쉬워진다.

## 핵심 포인트
- 워크플로우에서 JDK 설정 후 Gradle 빌드
- 테스트 실패 시 머지 전 즉시 확인
- 브랜치 보호 규칙과 함께 쓰면 효과적

## 정리
CI 자동화는 협업 품질의 기본 장치이며, 작은 프로젝트일수록 초기에 도입하는 편이 유리하다.
