---
layout: post
title: "SpringBoot 입문 4: Gradle(build.gradle) 기본과 실행/빌드 흐름"
date: 2026-05-17 00:40:00 +0900
category: SpringBoot
permalink: /springboot/mvc-gradle-build-basics
description: "Spring Boot에서 build.gradle이 어떤 역할을 하는지, 의존성 추가 후 sync와 build, run 흐름을 어떻게 이해하면 되는지 초보자 기준으로 설명합니다."
image: "/assets/img/og/springboot-series-cover.svg"
---


# SpringBoot 입문 4: Gradle(build.gradle) 기본과 실행/빌드 흐름

> 이 글에서는 Gradle이 의존성, 빌드, 실행을 어떻게 묶어주는지부터 IntelliJ에서 어디를 봐야 하는지까지 정리합니다.
>
> 이전 글: [SpringBoot 입문 3: 요청 받기(@RequestBody) + Validation + 예외 처리](/springboot/mvc-request-validation-exception)
> 다음 글: [SpringBoot 입문 5: application.yml과 설정 관리(Profiles) 기본](/springboot/mvc-application-yml-profiles)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 2: 첫 Controller 만들기 + Debug로 흐름 이해하기](/springboot/mvc-hello-controller-debug)
> - [SpringBoot 입문 6: REST 기본(@RequestParam/@PathVariable)과 HTTP 상태코드](/springboot/mvc-rest-basics-params-status)


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

<!-- codex-springboot-detail:start -->

## 여기서 한 단계 더 이해해보겠습니다

Spring Boot 프로젝트를 다루실 때 Gradle은 단순히 의존성 목록을 적는 파일이 아니라, 프로젝트를 어떻게 빌드하고 실행하며 테스트할지 정하는 중심 도구입니다. build.gradle을 읽을 수 있게 되면 새 라이브러리를 붙이는 일, 테스트 실행, JAR 생성, CI 설정까지 대부분의 작업을 훨씬 덜 두려워하게 됩니다.

처음에는 implementation, testImplementation, plugins 같은 키워드가 낯설게 느껴지실 수 있는데, 실제로는 "무엇을 가져오고", "어떻게 실행하고", "어떤 버전 기준으로 맞출지"를 선언하는 문장이라고 이해하시면 됩니다. 이 감각이 잡히면 오류 메시지도 훨씬 잘 읽히기 시작합니다.

이 주제에서 특히 중요하게 보셔야 하는 부분은 플러그인, 저장소, 의존성, bootRun과 bootJar 같은 기본 태스크를 연결해서 보는 것 입니다. 처음 읽으실 때는 모든 문장을 외우려고 하기보다, 코드가 어느 계층에 놓이고 어떤 순서로 실행되는지를 천천히 따라가보시면 좋습니다. Spring Boot 학습은 각각의 기술을 따로 외우는 공부라기보다, 여러 조각이 실제 요청 흐름 안에서 어떻게 맞물리는지 이해하는 과정에 가깝습니다.

## 예시 코드로 흐름을 잡아보겠습니다

```gradle
plugins {
  id 'java'
  id 'org.springframework.boot' version '3.3.0'
  id 'io.spring.dependency-management' version '1.1.5'
}

group = 'com.example'
version = '0.0.1-SNAPSHOT'

repositories {
  mavenCentral()
}

dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-web'
  implementation 'org.springframework.boot:spring-boot-starter-validation'
  testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

tasks.named('test') {
  useJUnitPlatform()
}
```
```bash
./gradlew bootRun
./gradlew test
./gradlew bootJar
```

위 예시는 Gradle 빌드 기본을 가장 작게 체험할 수 있는 흐름으로 구성했습니다. 처음에는 코드 한 줄 한 줄의 문법보다, 선언 위치와 실행 시점을 중심으로 읽어보시는 편이 훨씬 이해가 잘 되실 것입니다. 예를 들어 설정 클래스인지, Controller인지, Service인지에 따라 책임이 달라지고, 그 책임이 곧 Spring Boot 구조를 읽는 기준이 됩니다.

또 하나 중요한 점은 '지금 당장 돌아가는 코드'와 '나중에도 유지보수하기 쉬운 코드'는 조금 다를 수 있다는 사실입니다. 그래서 예제를 실행해보신 뒤에는 변수명, 메서드명, 경로, 설정 값을 직접 한두 번 바꿔보시면서 어떤 부분이 고정 규칙이고 어떤 부분이 프로젝트 상황에 따라 달라지는지 구분해보시면 정말 큰 도움이 됩니다.

## 자주 하는 실수와 에러도 같이 보겠습니다

- 라이브러리를 추가하고도 sync를 하지 않으면 IDE가 계속 빨간 줄을 보여 혼동될 수 있습니다.
- 의존성 버전을 직접 여기저기 적다가 Spring Boot BOM과 충돌하면 예상치 못한 호환성 문제가 생길 수 있습니다.
- 빌드 오류를 맨 아래 한 줄만 보고 판단하면 실제 원인보다 결과 메시지만 보게 되는 경우가 많습니다.

이런 실수들은 대부분 개념을 몰라서라기보다, 실행 순서나 설정 위치를 한 번에 다 보지 못해서 생깁니다. 그래서 에러가 나면 조급하게 코드를 많이 바꾸기보다, 로그와 요청 흐름을 기준으로 '어디까지는 정상이고 어디서부터 어긋났는지'를 차분히 확인해보시는 편이 좋습니다.

## 실무에서는 이렇게 적용하시면 좋습니다

- 처음에는 `bootRun`, `test`, `bootJar` 세 가지 태스크만 확실히 이해하셔도 실무에서 체감이 큽니다.
- 오류가 나면 콘솔 맨 위쪽 원인 메시지부터 읽는 습관을 들이시는 것이 좋습니다.
- 의존성을 추가한 뒤에는 "왜 이 라이브러리가 필요한지"를 한 줄 메모해두시면 나중에 정리할 때 도움이 됩니다.

처음부터 완벽한 구조를 만들려고 하실 필요는 없습니다. 다만 작은 예제라도 책임 분리, 예외 처리, 설정 관리, 로그 확인 같은 기본 습관을 함께 가져가시면 나중에 프로젝트가 커졌을 때 훨씬 덜 흔들리게 됩니다. 특히 Spring Boot는 편의 기능이 많아서 '왜 이렇게 동작하는지'를 놓치기 쉬우므로, 동작 원리를 설명할 수 있을 정도로 천천히 정리해두시는 것이 좋습니다.


<!-- codex-springboot-inline-links:start -->

지금 읽고 계신 주제가 아직 조금 추상적으로 느껴지신다면 [SpringBoot 입문 1: IntelliJ + Java + Gradle로 프로젝트 생성부터 실행까지](/springboot/mvc-setup-intellij), [SpringBoot 입문 1-1: IntelliJ로 첫 프로젝트 만들고 실행까지 (완전 실습 가이드)](/springboot/mvc-01-1-setup-intellij-practice), [SpringBoot 입문 22: JAR 빌드(bootJar)하고 실행하기](/springboot/mvc-build-jar-run) 글도 함께 읽어보시면 좋습니다. 앞단의 준비 과정과 다음 단계의 확장 흐름이 자연스럽게 이어져서, 지금 배우는 내용이 프로젝트 안에서 어디에 연결되는지 더 분명하게 감을 잡으실 수 있습니다.

<!-- codex-springboot-inline-links:end -->
## 마무리 정리

SpringBoot 입문 4: Gradle(build.gradle) 기본과 실행/빌드 흐름 글을 공부하실 때는 한 번 읽고 끝내기보다, 직접 실행해보고 일부러 에러도 만들어보시고, 설정이나 코드 값을 조금씩 바꿔보시는 방식으로 접근해보시길 권합니다. 그렇게 하시면 단순 요약 글보다 훨씬 오래 기억에 남고, 다음 주제인 데이터 접근, 보안, 테스트, 운영 자동화로 넘어갈 때도 이해가 자연스럽게 이어집니다. 지금 단계에서 가장 중요한 것은 완벽함보다 흐름을 잡는 것이니, 천천히 따라오셔도 충분합니다.

<!-- codex-springboot-detail:end -->

<!-- codex-springboot-links:start -->

## 이어서 읽어보시면 좋습니다

- [SpringBoot 입문 1: IntelliJ + Java + Gradle로 프로젝트 생성부터 실행까지](/springboot/mvc-setup-intellij)
- [SpringBoot 입문 1-1: IntelliJ로 첫 프로젝트 만들고 실행까지 (완전 실습 가이드)](/springboot/mvc-01-1-setup-intellij-practice)
- [SpringBoot 입문 22: JAR 빌드(bootJar)하고 실행하기](/springboot/mvc-build-jar-run)

지금 글과 연결되는 흐름으로 골라두었으니, 바로 다음 실습이나 비교 학습을 이어가실 때 참고해보시면 좋겠습니다.

<!-- codex-springboot-links:end -->
