---
layout: post
title: "SpringBoot 입문 22: JAR 빌드(bootJar)하고 실행하기"
date: 2026-05-17 03:40:00 +0900
category: SpringBoot
permalink: /springboot/mvc-build-jar-run
---

# SpringBoot 입문 22: JAR 빌드(bootJar)하고 실행하기

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

