---
layout: post
title: "SpringBoot 입문 1: IntelliJ + Java + Gradle로 프로젝트 생성부터 실행까지"
date: 2026-05-17 00:10:00 +0900
category: SpringBoot
permalink: /springboot/mvc-setup-intellij
description: "Spring Boot에서 IntelliJ와 Gradle로 첫 프로젝트를 만들고 Java 17 설정, Spring Initializr 선택, 실행 확인까지 한 번에 따라갈 수 있도록 단계별로 정리합니다."
image: "/assets/img/og/springboot-series-cover.svg"
---


# SpringBoot 입문 1: IntelliJ + Java + Gradle로 프로젝트 생성부터 실행까지

> 이 글에서는 Spring Boot 프로젝트를 처음 만들고 실행하는 흐름을 실습 기준으로 정리합니다.
>
> 이전 글: 없음
> 다음 글: [SpringBoot 입문 2: 첫 Controller 만들기 + Debug로 흐름 이해하기](/springboot/mvc-hello-controller-debug)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 4: Gradle(build.gradle) 기본과 실행/빌드 흐름](/springboot/mvc-gradle-build-basics)
> - [SpringBoot 입문 5: application.yml과 설정 관리(Profiles) 기본](/springboot/mvc-application-yml-profiles)

이 글은 SpringBoot가 처음인 사람도 **이 글만 따라하면** 로컬에서 프로젝트를 만들고 실행해서 브라우저로 확인할 수 있게 만드는 게 목표입니다.

이 글은 Spring MVC 기준입니다. (내 블로그에 WebFlux 글/프로젝트도 있지만, 이 시리즈는 MVC로만 갑니다.)

## 준비물(버전)

- Java: 17 (LTS)
- 빌드: Gradle (Wrapper 사용)
- IDE: IntelliJ IDEA
- Spring Boot: 3.x

버전은 꼭 위와 동일하지 않아도 되지만, 초보자라면 **Java 17 + Spring Boot 3.x** 조합을 추천합니다.

## 1) Java 17 설치 확인

이미 설치되어 있다면 확인부터 합니다.

Windows PowerShell:

```powershell
java -version
```

정상이라면 `17.x`가 나옵니다.

만약 Java가 없거나 8/11 같은 낮은 버전이면 17을 설치하고 다시 확인하세요.

## 2) IntelliJ 설치/설정(처음 한 번)

- IntelliJ 설치 후 실행
- `New UI` 여부는 취향이지만 기본값으로 진행해도 됩니다.

프로젝트 생성 단계에서 Gradle과 JDK 설정을 같이 잡는 게 중요합니다.

## 3) 프로젝트 만들기(가장 쉬운 경로)

초보자에게 가장 헷갈리는 건 "프로젝트를 어디서 만들고, 뭘 체크해야 하는지"입니다.

추천 흐름:

1. IntelliJ에서 `New Project`
2. `Spring Initializr` 선택
3. Language: `Java`
4. Build system: `Gradle`
5. JDK: `17`
6. Dependencies: `Spring Web` (MVC)

아래 이미지는 "어떤 화면에서 뭘 고르는지" 감을 잡기 위한 예시입니다.

![IntelliJ New Project (Spring)](/assets/img/intellij/spring_new_project.svg)

## 4) 패키지 구조 정하기(처음부터 깔끔하게)

패키지는 보통 도메인 기반으로 갑니다.

예시:

- Group: `com.sungkyu`
- Artifact: `demo`
- Package: `com.sungkyu.demo`

이대로 만들면 기본 메인 클래스가 `com.sungkyu.demo.DemoApplication`처럼 생성됩니다.

## 5) 의존성 선택(최소로 시작)

첫 글에서는 복잡하게 넣지 않고 `Spring Web`만으로 시작합니다.

추가로 나중에 자주 붙이는 것:

- Lombok(선호에 따라)
- Spring Boot DevTools(자동 리로드)
- Validation(입력 검증)
- Spring Data JPA(DB)

하지만 처음엔 일단 실행부터 성공시키는 게 중요합니다.

## 6) Gradle 동기화가 끝날 때까지 기다리기

처음 프로젝트를 만들면 Gradle이 의존성을 받으면서 인덱싱을 합니다.

이때 많이 나오는 초보자 실수:

- 기다리기 전에 바로 실행 눌러서 빌드가 꼬임
- 네트워크 문제로 의존성 다운로드 실패

오른쪽 아래 Gradle 작업이 끝날 때까지 기다린 뒤 진행하세요.

## 7) 실행하기(가장 기본)

메인 클래스(`...Application`)를 실행합니다.

정상이라면 콘솔에 대략 이런 로그가 보입니다.

- `Tomcat started on port(s): 8080`
- `Started ... in ... seconds`

예시 이미지:

![IntelliJ Run Output](/assets/img/intellij/spring_run_output.svg)

## 8) 브라우저로 확인하기

아직 컨트롤러가 없으니 `http://localhost:8080/`는 404가 뜰 수 있습니다. 그게 정상입니다.

다음 글에서 "Hello API"를 만들고 브라우저에서 응답을 확인합니다.

## 9) 자주 막히는 문제 해결

### (1) 8080 포트가 이미 사용 중

에러에 `Port 8080 was already in use` 같은 문구가 나오면, 다른 프로세스가 8080을 쓰는 겁니다.

간단 해결:

- 해당 프로세스를 종료하거나
- Spring Boot 포트를 바꿉니다.

포트 변경(예: 8081):

`src/main/resources/application.yml`

```yaml
server:
  port: 8081
```

### (2) JDK가 다른 버전으로 잡힘

프로젝트 JDK/Gradle JDK가 17이 아니면 빌드가 꼬일 수 있어요.

확인 포인트:

- IntelliJ Project SDK
- Gradle JVM

## 정리

이제 준비는 끝났습니다.

다음 글에서는:

- 첫 컨트롤러 만들기
- 브라우저로 `GET /hello` 확인
- 디버그 모드로 한 줄씩 따라가기

까지 진행합니다.

<!-- codex-springboot-detail:start -->

## 여기서 한 단계 더 이해해보겠습니다

Spring Boot를 처음 시작하실 때 가장 많이 막히는 부분은 코드 문법이 아니라 실행 환경입니다. 프로젝트가 생성은 되었는데 Gradle sync가 오래 걸리거나, JDK 버전이 맞지 않거나, 메인 클래스는 있는데 실행 버튼이 비활성화되는 식으로 첫 단추에서 시간이 많이 빠질 수 있습니다.

그래서 입문 단계에서는 "어떤 기능을 만들까"보다 "이 프로젝트가 내 컴퓨터에서 안정적으로 켜지는가"를 먼저 확인하셔야 합니다. IntelliJ, JDK, Gradle, Spring Boot 버전이 한 번 맞아떨어지면 이후 학습 속도가 정말 빨라지기 때문에, 첫 설정은 번거롭더라도 차근차근 확인해두시는 편이 좋습니다.

이 주제에서 특히 중요하게 보셔야 하는 부분은 JDK 버전, Gradle 플러그인, Spring Boot 스타터 의존성, 메인 애플리케이션 클래스 위치를 한 흐름으로 묶어서 보는 것 입니다. 처음 읽으실 때는 모든 문장을 외우려고 하기보다, 코드가 어느 계층에 놓이고 어떤 순서로 실행되는지를 천천히 따라가보시면 좋습니다. Spring Boot 학습은 각각의 기술을 따로 외우는 공부라기보다, 여러 조각이 실제 요청 흐름 안에서 어떻게 맞물리는지 이해하는 과정에 가깝습니다.

## 예시 코드로 흐름을 잡아보겠습니다

```gradle
plugins {
  id 'java'
  id 'org.springframework.boot' version '3.3.0'
  id 'io.spring.dependency-management' version '1.1.5'
}

group = 'com.example'
version = '0.0.1-SNAPSHOT'

java {
  toolchain {
    languageVersion = JavaLanguageVersion.of(17)
  }
}

repositories {
  mavenCentral()
}

dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-web'
  testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```
```
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class DemoApplication {
  public static void main(String[] args) {
    SpringApplication.run(DemoApplication.class, args);
  }
}
```

위 예시는 SpringBoot 개발 환경 설정을 가장 작게 체험할 수 있는 흐름으로 구성했습니다. 처음에는 코드 한 줄 한 줄의 문법보다, 선언 위치와 실행 시점을 중심으로 읽어보시는 편이 훨씬 이해가 잘 되실 것입니다. 예를 들어 설정 클래스인지, Controller인지, Service인지에 따라 책임이 달라지고, 그 책임이 곧 Spring Boot 구조를 읽는 기준이 됩니다.

또 하나 중요한 점은 '지금 당장 돌아가는 코드'와 '나중에도 유지보수하기 쉬운 코드'는 조금 다를 수 있다는 사실입니다. 그래서 예제를 실행해보신 뒤에는 변수명, 메서드명, 경로, 설정 값을 직접 한두 번 바꿔보시면서 어떤 부분이 고정 규칙이고 어떤 부분이 프로젝트 상황에 따라 달라지는지 구분해보시면 정말 큰 도움이 됩니다.

## 자주 하는 실수와 에러도 같이 보겠습니다

- IntelliJ 프로젝트 JDK와 Gradle JVM이 서로 다르면 빌드는 되는데 실행이 실패하거나, 반대로 실행은 되는데 컴파일 오류가 나는 경우가 있습니다.
- 메인 클래스가 루트 패키지보다 아래가 아닌 엉뚱한 패키지에 있으면 컴포넌트 스캔 범위가 꼬여서 Controller나 Service가 등록되지 않을 수 있습니다.
- 의존성을 바꿨는데 Gradle sync를 하지 않으면 "클래스를 찾을 수 없음" 오류가 계속 보여 혼란스러우실 수 있습니다.

이런 실수들은 대부분 개념을 몰라서라기보다, 실행 순서나 설정 위치를 한 번에 다 보지 못해서 생깁니다. 그래서 에러가 나면 조급하게 코드를 많이 바꾸기보다, 로그와 요청 흐름을 기준으로 '어디까지는 정상이고 어디서부터 어긋났는지'를 차분히 확인해보시는 편이 좋습니다.

## 실무에서는 이렇게 적용하시면 좋습니다

- 처음 프로젝트는 옵션을 최소화해서 만드시고, `spring-boot-starter-web` 하나로 실행부터 확인하시는 편이 좋습니다.
- 실행에 성공하면 바로 Hello API 하나를 만들어 두시면 이후 환경 문제인지 코드 문제인지 구분하기 쉬워집니다.
- 버전은 최신이라고 무조건 좋은 것이 아니라, JDK와 IntelliJ가 안정적으로 지원하는 조합을 고르는 것이 훨씬 중요합니다.

처음부터 완벽한 구조를 만들려고 하실 필요는 없습니다. 다만 작은 예제라도 책임 분리, 예외 처리, 설정 관리, 로그 확인 같은 기본 습관을 함께 가져가시면 나중에 프로젝트가 커졌을 때 훨씬 덜 흔들리게 됩니다. 특히 Spring Boot는 편의 기능이 많아서 '왜 이렇게 동작하는지'를 놓치기 쉬우므로, 동작 원리를 설명할 수 있을 정도로 천천히 정리해두시는 것이 좋습니다.

## 마무리 정리

SpringBoot 입문 1: IntelliJ + Java + Gradle로 프로젝트 생성부터 실행까지 글을 공부하실 때는 한 번 읽고 끝내기보다, 직접 실행해보고 일부러 에러도 만들어보시고, 설정이나 코드 값을 조금씩 바꿔보시는 방식으로 접근해보시길 권합니다. 그렇게 하시면 단순 요약 글보다 훨씬 오래 기억에 남고, 다음 주제인 데이터 접근, 보안, 테스트, 운영 자동화로 넘어갈 때도 이해가 자연스럽게 이어집니다. 지금 단계에서 가장 중요한 것은 완벽함보다 흐름을 잡는 것이니, 천천히 따라오셔도 충분합니다.

<!-- codex-springboot-detail:end -->
