---
layout: post
title: "SpringBoot 입문 22: JAR 빌드(bootJar)하고 실행하기"
date: 2026-05-17 03:40:00 +0900
category: SpringBoot
permalink: /springboot/mvc-build-jar-run
description: "Spring Boot에서 bootJar로 실행 파일을 만들고 JAR를 직접 실행하는 방법, 로컬 실행과 배포 전 점검 포인트를 함께 설명합니다."
image: "/assets/img/og/springboot-series-cover.svg"
---

# SpringBoot 입문 22: JAR 빌드(bootJar)하고 실행하기

> 이 글에서는 코드를 JAR로 묶고 명령어로 직접 실행하는 가장 기본적인 배포 직전 흐름을 익힙니다.
>
> 이전 글: [SpringBoot 입문 21: 파일 업로드/다운로드 기본(Multipart)](/springboot/mvc-file-upload-download)
> 다음 글: [SpringBoot 입문 23: Docker로 실행하기(가장 단순한 Dockerfile)](/springboot/mvc-dockerize-basics)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 20: Spring Security 아주 기초(인증/인가 감 잡기)](/springboot/mvc-security-basic)
> - [SpringBoot 입문 24: Docker Compose로 앱+DB 함께 실행](/springboot/mvc-docker-compose-basics)


로컬에서 IntelliJ로 실행만 하다가, 배포를 하려면 결국 "JAR로 빌드해서 실행"하는 흐름을 알아야 합니다.

목표:

- `bootJar`로 jar 만들기
- jar 실행하기
- 프로필/포트 설정을 실행 옵션으로 바꾸기

## 1) bootJar

```powershell
.\gradlew.bat clean bootJar
```

결과는 보통:

- `build/libs/*.jar`

## 2) 실행

```powershell
java -jar build\\libs\\demo-0.0.1-SNAPSHOT.jar
```

## 3) 프로필/포트 변경

```powershell
java -jar build\\libs\\demo-0.0.1-SNAPSHOT.jar --spring.profiles.active=prod --server.port=8081
```

## 정리

- 배포의 시작은 jar 빌드/실행
- 실행 옵션으로 프로필/포트를 바꿀 수 있다

