---
layout: post
title: "SpringBoot 입문 23: Docker로 실행하기(가장 단순한 Dockerfile)"
date: 2026-05-17 03:50:00 +0900
category: SpringBoot
permalink: /springboot/mvc-dockerize-basics
description: "Spring Boot 애플리케이션을 Dockerfile로 이미지화하고 JAR 실행부터 컨테이너 기동까지 따라가며, 초보자가 자주 헷갈리는 Docker 기본 흐름을 정리합니다."
image:
  path: "/assets/img/og/springboot-docker-cover.svg"
  alt: "SpringBoot Docker basics article cover"
---

# SpringBoot 입문 23: Docker로 실행하기(가장 단순한 Dockerfile)

> 이 글에서는 Spring Boot 애플리케이션을 JAR로 만든 뒤 Docker 이미지와 컨테이너로 실행하는 가장 기본적인 배포 전환 흐름을 익힙니다.
>
> 이전 글: [SpringBoot 입문 22: JAR 빌드(bootJar)하고 실행하기](/springboot/mvc-build-jar-run)
> 다음 글: [SpringBoot 입문 24: Docker Compose로 앱+DB 함께 실행](/springboot/mvc-docker-compose-basics)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 33: GitHub Actions로 CI 자동화](/springboot/mvc-github-actions-ci)
> - [SpringBoot 입문 8: JPA + H2로 DB 시작하기 (Entity/Repository)](/springboot/mvc-jpa-h2-first-db)

Docker는 초보자에게 무섭게 느껴지지만, 배포/운영에서는 사실상 표준이 된 경우가 많습니다.

목표:

- jar를 만들고
- Docker 이미지로 감싸서
- 컨테이너로 실행한다

## 1) Dockerfile(가장 단순)

```dockerfile
FROM eclipse-temurin:17-jre
WORKDIR /app
COPY build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","/app/app.jar"]
```

## 2) 빌드/실행

```bash
./gradlew bootJar
docker build -t demo:local .
docker run --rm -p 8080:8080 demo:local
```

## 3) 초보자 팁

- 로컬에서는 IntelliJ 실행이 제일 빠름
- 배포/운영에 가까워질수록 Docker가 필요해짐

## 정리

- Docker는 jar 실행을 "환경까지 포함해서" 묶는 도구
- 초보자는 단순 Dockerfile부터 시작하면 된다

