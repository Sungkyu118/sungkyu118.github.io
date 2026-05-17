---
layout: post
title: "SpringBoot 입문 4: Gradle(build.gradle) 기본과 실행/빌드 흐름"
date: 2026-05-17 00:40:00 +0900
category: SpringBoot
permalink: /springboot/mvc-gradle-build-basics
---

# SpringBoot 입문 4: Gradle(build.gradle) 기본과 실행/빌드 흐름

이 글은 초보자가 가장 헷갈려 하는 `Gradle`의 기본을 잡는 글입니다. 목표는 단순합니다.

- `build.gradle`이 뭘 하는지 이해한다
- Gradle 작업(동기화/빌드/실행)을 IntelliJ에서 안전하게 할 수 있다
- 의존성 추가/수정 후 어디를 봐야 하는지 안다

## 1) Gradle은 무엇인가

Gradle은 빌드 도구입니다. Java 프로젝트에서 Gradle은 대략 아래 일을 합니다.

- 의존성 다운로드(라이브러리 가져오기)
- 컴파일
- 테스트 실행
- JAR 만들기

초보자가 기억하면 좋은 한 줄:

> `build.gradle`은 "프로젝트의 레시피"다.

## 2) Wrapper(gradlew)를 꼭 쓰는 이유

프로젝트에는 보통 `gradlew`(Windows는 `gradlew.bat`)가 있습니다. 이게 Wrapper예요.

- 팀원마다 Gradle 버전이 달라도 동일한 빌드가 되게 해줌
- CI(GitHub Actions)에서도 같은 버전으로 빌드 가능

그래서 시스템에 Gradle을 따로 설치했더라도, 프로젝트에서는 Wrapper 사용을 권장합니다.

## 3) build.gradle에서 제일 자주 보는 부분

초보자 기준으로, 처음엔 이 4개만 보면 됩니다.

1. plugins
2. group/version (프로젝트 좌표)
3. dependencies (의존성)
4. tasks (옵션)

의존성 예:

```gradle
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-web'
  testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

- `implementation`: 실행에 필요한 라이브러리
- `testImplementation`: 테스트에서만 필요한 라이브러리

## 4) 의존성 추가 후 꼭 해야 할 것: Gradle Sync

의존성을 추가했는데 import가 안 된다면, 거의 항상 "Sync가 아직 안 된 상태"입니다.

IntelliJ에서:

- 우측 Gradle 탭에서 Refresh(동기화)
- 또는 상단에 뜨는 Sync 안내 배너 클릭

예시 이미지:

![Gradle Sync](/assets/img/intellij/gradle_sync.svg)

## 5) 자주 쓰는 Gradle 명령(필수 3개)

프로젝트 루트에서:

```bash
./gradlew test
./gradlew bootRun
./gradlew bootJar
```

Windows라면:

```powershell
.\gradlew.bat test
.\gradlew.bat bootRun
.\gradlew.bat bootJar
```

## 6) 빌드가 실패할 때 보는 순서

초보자에게 추천하는 디버깅 순서:

1. 에러의 "첫 번째 stacktrace 줄"이 아니라, "실제 원인 문장"을 찾는다
2. 의존성 다운로드 실패라면 네트워크/프록시를 의심한다
3. Java 버전이 맞는지(Project SDK/Gradle JVM) 확인한다

## 정리

- `build.gradle`은 의존성과 빌드 규칙을 정의하는 파일
- 의존성 추가 후에는 Gradle Sync가 필수
- `test`, `bootRun`, `bootJar`만 익혀도 개발 시작이 가능

