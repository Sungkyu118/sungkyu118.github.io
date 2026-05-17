---
layout: post
title: "SpringBoot 입문 24: Docker Compose로 앱+DB 함께 실행"
date: 2026-05-17 22:20:00 +0900
category: SpringBoot
permalink: /springboot/mvc-docker-compose-basics
---

# SpringBoot 입문 24: Docker Compose로 앱+DB 함께 실행

이번 글에서는 Spring Boot 애플리케이션과 MySQL을 Docker Compose로 함께 실행한다.

## 핵심 포인트
- `docker-compose.yml` 하나로 다중 컨테이너 관리
- 서비스 이름으로 DB 호스트를 지정 (`db`)
- 로컬 개발 환경을 팀원과 동일하게 맞출 수 있음

## 실행 순서
```bash
docker compose up --build
docker compose down
```

## 정리
Compose를 사용하면 스프링 앱과 데이터베이스를 한 번에 올리고 내릴 수 있어 개발 반복 속도가 빨라진다.
