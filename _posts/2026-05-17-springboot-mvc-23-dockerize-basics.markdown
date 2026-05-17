---
layout: post
title: "SpringBoot 입문 23: Docker로 실행하기(가장 단순한 Dockerfile)"
date: 2026-05-17 03:50:00 +0900
category: SpringBoot
permalink: /springboot/mvc-dockerize-basics
---

# SpringBoot 입문 23: Docker로 실행하기(가장 단순한 Dockerfile)

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

