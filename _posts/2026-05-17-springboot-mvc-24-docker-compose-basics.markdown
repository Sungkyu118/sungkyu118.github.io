---
layout: post
title: "SpringBoot 입문 24: Docker Compose로 앱+DB 함께 실행"
date: 2026-05-17 22:20:00 +0900
category: SpringBoot
permalink: /springboot/mvc-docker-compose-basics
description: "Spring Boot 애플리케이션과 데이터베이스를 Docker Compose로 함께 실행하는 방법, 서비스 이름 기반 연결과 로컬 개발 환경 통일 포인트를 정리합니다."
image: "/assets/img/og/springboot-series-cover.svg"
---

# SpringBoot 입문 24: Docker Compose로 앱+DB 함께 실행

> 이 글에서는 애플리케이션과 데이터베이스를 Compose로 같이 띄워 로컬 환경을 더 배포에 가깝게 맞춥니다.
>
> 이전 글: [SpringBoot 입문 23: Docker로 실행하기(가장 단순한 Dockerfile)](/springboot/mvc-dockerize-basics)
> 다음 글: [SpringBoot 입문 25: JPA N+1 문제와 fetch join](/springboot/mvc-jpa-n-plus-one)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 22: JAR 빌드(bootJar)하고 실행하기](/springboot/mvc-build-jar-run)
> - [SpringBoot 입문 26: Redis 캐시 기본 적용](/springboot/mvc-redis-cache-basics)


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
