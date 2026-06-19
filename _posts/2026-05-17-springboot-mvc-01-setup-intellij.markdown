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

