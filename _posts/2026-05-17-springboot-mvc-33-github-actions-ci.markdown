---
layout: post
title: "SpringBoot 입문 33: GitHub Actions로 CI 자동화"
date: 2026-05-17 22:20:00 +0900
category: SpringBoot
permalink: /springboot/mvc-github-actions-ci
---

# SpringBoot 입문 33: GitHub Actions로 CI 자동화

푸시/PR마다 빌드와 테스트를 자동 실행하면 품질 관리가 쉬워진다.

## 핵심 포인트
- 워크플로우에서 JDK 설정 후 Gradle 빌드
- 테스트 실패 시 머지 전 즉시 확인
- 브랜치 보호 규칙과 함께 쓰면 효과적

## 정리
CI 자동화는 협업 품질의 기본 장치이며, 작은 프로젝트일수록 초기에 도입하는 편이 유리하다.
