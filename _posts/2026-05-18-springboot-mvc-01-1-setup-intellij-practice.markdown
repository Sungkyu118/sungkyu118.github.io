---
layout: post
title: "SpringBoot 입문 1-1: IntelliJ로 첫 프로젝트 만들고 실행까지 (완전 실습 가이드)"
date: 2026-05-18 00:25:00 +0900
category: SpringBoot
permalink: /springboot/mvc-01-1-setup-intellij-practice
description: "Spring Boot 입문자가 IntelliJ에서 첫 프로젝트를 실제로 만들고 실행하기까지, 설치 확인과 흔한 에러 포인트를 더 자세한 실습 흐름으로 안내합니다."
image: "/assets/img/og/springboot-series-cover.svg"
---


# SpringBoot 입문 1-1: IntelliJ로 첫 프로젝트 만들고 실행까지 (완전 실습 가이드)

> 이 글에서는 완전 초보자 기준으로 Spring Boot 첫 프로젝트를 실제로 만들고 실행해보는 더 자세한 실습 흐름을 다룹니다.
>
> 이전 글: [SpringBoot 입문 1: IntelliJ + Java + Gradle로 프로젝트 생성부터 실행까지](/springboot/mvc-setup-intellij)
> 다음 글: [SpringBoot 입문 2: 첫 Controller 만들기 + Debug로 흐름 이해하기](/springboot/mvc-hello-controller-debug)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 4: Gradle(build.gradle) 기본과 실행/빌드 흐름](/springboot/mvc-gradle-build-basics)
> - [SpringBoot 입문 5: application.yml과 설정 관리(Profiles) 기본](/springboot/mvc-application-yml-profiles)


이번 글은 "진짜 처음" 시작하는 사람을 기준으로 작성했다.  
중간에 막히지 않도록, 왜 이 설정이 필요한지와 어떤 에러가 나오는지까지 같이 다룬다.

이번 1-1 글의 목표는 아래 딱 3가지다.

1. Java 17이 제대로 설치되어 있는지 확인한다.
2. IntelliJ에서 Spring Boot 프로젝트를 정확한 옵션으로 생성한다.
3. 프로젝트를 실제로 실행해서 서버가 켜지는 로그까지 확인한다.

---

## 0) 시작 전에 반드시 확인할 것

### 왜 이 단계를 먼저 하냐?

Spring Boot 프로젝트는 **JDK 버전**, **빌드 도구(Gradle)**, **IDE 설정**이 조금만 어긋나도 바로 에러가 난다.  
특히 초반에는 코드가 문제가 아니라 환경이 문제인 경우가 훨씬 많다.

### 준비물

- 운영체제: Windows (이 글 기준)
- JDK: 17
- IDE: IntelliJ IDEA (Community/Ultimate 모두 가능)
- 네트워크: 초기 의존성 다운로드가 가능해야 함

---

## 1) Java 17 설치 확인

### 1-1. PowerShell에서 버전 확인

```powershell
java -version
```

정상이라면 `17.x.x` 형태가 보인다.

예시:

```text
openjdk version "17.0.12" 2024-07-16
```

### 1-2. 여기서 자주 발생하는 에러

#### 에러 A: `java`를 인식하지 못함

증상:

```text
'java'은(는) 내부 또는 외부 명령, 실행할 수 있는 프로그램...
```

원인:

- JDK 미설치
- PATH 환경 변수 미설정
- 설치는 했는데 터미널을 재시작하지 않음

해결:

1. JDK 17 설치
2. `JAVA_HOME`을 JDK 경로로 지정
3. `Path`에 `%JAVA_HOME%\bin` 추가
4. PowerShell 완전히 종료 후 다시 열기

#### 에러 B: Java 8/11이 출력됨

원인:

- 여러 Java가 설치되어 있고, 오래된 버전이 PATH에서 우선순위가 높음

해결:

- PATH에서 Java 17 경로를 위로 올리거나,
- 불필요한 구버전 Java 경로를 정리

---

## 2) IntelliJ에서 Spring Boot 프로젝트 생성

## 2-1. New Project 진입

- IntelliJ 실행
- `New Project` 클릭
- 좌측에서 `Spring Initializr` 선택

### 2-2. 기본 옵션 정확히 맞추기

아래는 이번 시리즈에서 기준으로 사용할 권장값이다.

- Name: `demo`
- Location: 본인이 관리하기 쉬운 작업 폴더
- Language: `Java`
- Type: `Gradle`
- JDK: `17`
- Packaging: `Jar`
- Java: `17`
- Spring Boot: 3.x

### 2-3. 의존성 선택

이번 1-1에서는 최소 구성으로 시작한다.

- `Spring Web`

왜 최소로 시작하나?

- 처음에는 "실행 성공" 경험이 가장 중요하다.
- 의존성을 많이 넣을수록 초기 에러 원인이 늘어난다.

---

## 3) 프로젝트 생성 직후 반드시 해야 할 확인

프로젝트를 만들면 IntelliJ가 Gradle 의존성을 다운로드한다.  
이때 impatient하게 바로 실행하면 실패하는 경우가 많다.

### 확인할 것

- 우하단 진행 표시(인덱싱/Gradle import)가 끝났는지
- `build.gradle`에 오류 표시가 없는지

### 자주 발생하는 에러와 대응

#### 에러 C: Gradle download 실패

증상 예시:

- `Could not resolve ...`
- `Read timed out`
- `PKIX path building failed`

원인:

- 회사/학교 네트워크 프록시
- 인증서 문제
- 일시적인 네트워크 장애

해결:

1. 개인 네트워크로 재시도
2. Gradle JVM이 올바른 JDK(17)인지 확인
3. 프록시 환경이면 IntelliJ/Gradle 프록시 설정 적용

#### 에러 D: Gradle JVM이 17이 아님

증상:

- 빌드 시 버전 관련 오류
- `Unsupported class file major version` 류 메시지

해결:

- IntelliJ 설정에서 `Gradle JVM`을 17로 맞춘다.

---

## 4) 첫 실행: 서버가 실제로 뜨는지 확인

### 4-1. 실행 방법

- `DemoApplication` 클래스 열기
- `main` 함수 왼쪽 실행 버튼 클릭

### 4-2. 성공 로그 기준

아래와 유사한 로그가 보이면 성공이다.

- `Tomcat started on port(s): 8080 (http)`
- `Started DemoApplication in ... seconds`

### 4-3. 브라우저 확인

`http://localhost:8080` 접속 시 404가 떠도 정상이다.  
아직 컨트롤러를 만들지 않았기 때문이다.

초보자가 여기서 가장 많이 오해한다.

- 오해: 404니까 실행 실패다.
- 실제: 서버는 성공적으로 실행 중이고, 매핑된 URL이 아직 없는 상태다.

---

## 5) 실행 단계에서 가장 흔한 문제 2가지

### 문제 1) `Port 8080 was already in use`

원인:

- 다른 스프링 앱 또는 다른 프로그램이 이미 8080 점유

해결 방법 A (권장): 기존 프로세스 종료  
해결 방법 B: 포트 변경

`src/main/resources/application.yml` 생성 후:

```yaml
server:
  port: 8081
```

주의:

- 포트 바꾸고도 브라우저를 8080으로 열면 계속 실패한다.
- 반드시 `http://localhost:8081`로 확인해야 한다.

### 문제 2) 앱은 켜졌는데 바로 종료됨

원인 후보:

- `spring-boot-starter-web` 누락
- 잘못된 실행 구성(Run Configuration)

해결:

- `build.gradle`의 Web 의존성 확인
- 실행 대상이 `DemoApplication`인지 확인

---

## 6) 이번 단계에서 꼭 기억할 핵심 습관

- 에러가 나면 코드부터 의심하지 말고 **환경(JDK/Gradle/네트워크/포트)** 먼저 점검한다.
- 로그 마지막 20줄을 집중해서 읽는다.
- "실행 성공"과 "기능 동작 성공"은 다른 단계다. (지금은 실행 성공이 목표)

---

## 마무리

여기까지 완료했다면, Spring Boot 입문의 가장 어려운 첫 관문을 통과한 것이다.  
다음 글(입문 1-2)에서는 `@RestController`와 `GET /hello`를 직접 만들고,  
브라우저와 Postman에서 응답까지 확인해보겠다.

<!-- codex-springboot-detail:start -->

## 여기서 한 단계 더 이해해보겠습니다

환경 설정을 끝내셨다면 바로 손으로 한 번 구현해보는 과정이 꼭 필요합니다. 눈으로만 따라 보실 때는 쉬워 보여도, 실제로 Controller 파일을 만들고 주소를 호출해보면 패키지 위치, 어노테이션, 실행 포트 같은 기본 요소들이 어떻게 연결되는지 훨씬 또렷하게 보이기 시작합니다.

특히 입문자분들은 "프로젝트가 켜졌다"와 "내가 요청을 보내서 응답을 받았다"를 같은 단계로 생각하시는 경우가 많은데, 실제로는 이 둘 사이에 중요한 이해 차이가 있습니다. 첫 실습에서는 브라우저나 curl로 직접 요청을 보내보면서 서버가 어떤 식으로 응답하는지 꼭 체감해보시는 것이 좋습니다.

이 주제에서 특히 중요하게 보셔야 하는 부분은 메인 클래스 실행, Controller 작성, URL 호출, 응답 확인이라는 가장 기본적인 왕복 흐름 입니다. 처음 읽으실 때는 모든 문장을 외우려고 하기보다, 코드가 어느 계층에 놓이고 어떤 순서로 실행되는지를 천천히 따라가보시면 좋습니다. Spring Boot 학습은 각각의 기술을 따로 외우는 공부라기보다, 여러 조각이 실제 요청 흐름 안에서 어떻게 맞물리는지 이해하는 과정에 가깝습니다.

## 예시 코드로 흐름을 잡아보겠습니다

```java
package com.example.demo.web;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloController {

  @GetMapping("/hello")
  public String hello() {
    return "안녕하세요. Spring Boot 첫 실습이 잘 동작하고 있습니다.";
  }
}
```
```bash
curl http://localhost:8080/hello
```

위 예시는 SpringBoot 첫 실습을 가장 작게 체험할 수 있는 흐름으로 구성했습니다. 처음에는 코드 한 줄 한 줄의 문법보다, 선언 위치와 실행 시점을 중심으로 읽어보시는 편이 훨씬 이해가 잘 되실 것입니다. 예를 들어 설정 클래스인지, Controller인지, Service인지에 따라 책임이 달라지고, 그 책임이 곧 Spring Boot 구조를 읽는 기준이 됩니다.

또 하나 중요한 점은 '지금 당장 돌아가는 코드'와 '나중에도 유지보수하기 쉬운 코드'는 조금 다를 수 있다는 사실입니다. 그래서 예제를 실행해보신 뒤에는 변수명, 메서드명, 경로, 설정 값을 직접 한두 번 바꿔보시면서 어떤 부분이 고정 규칙이고 어떤 부분이 프로젝트 상황에 따라 달라지는지 구분해보시면 정말 큰 도움이 됩니다.

## 자주 하는 실수와 에러도 같이 보겠습니다

- Controller 클래스를 `@SpringBootApplication`이 있는 패키지 바깥에 두면 스캔되지 않아 404가 나올 수 있습니다.
- 8080 포트를 다른 프로그램이 이미 사용 중이면 애플리케이션이 켜지지 않고 port already in use 오류가 발생합니다.
- 브라우저에서 `/hello`를 호출했는데 Whitelabel Error Page가 보이면 URL, 실행 포트, 컨트롤러 등록 여부를 차례대로 확인해보셔야 합니다.

이런 실수들은 대부분 개념을 몰라서라기보다, 실행 순서나 설정 위치를 한 번에 다 보지 못해서 생깁니다. 그래서 에러가 나면 조급하게 코드를 많이 바꾸기보다, 로그와 요청 흐름을 기준으로 '어디까지는 정상이고 어디서부터 어긋났는지'를 차분히 확인해보시는 편이 좋습니다.

## 실무에서는 이렇게 적용하시면 좋습니다

- 첫 실습은 반드시 손으로 직접 입력해보시고, 응답 문구도 스스로 한두 번 바꿔보시면 기억에 오래 남습니다.
- 브라우저 호출과 curl 호출을 둘 다 해보시면 사람이 읽는 화면과 도구로 테스트하는 흐름의 차이를 자연스럽게 익히실 수 있습니다.
- 잘 되면 바로 끝내지 마시고, 포트를 8081로 바꿔보거나 경로를 `/hi`로 바꿔보는 식으로 작은 변형도 함께 해보시면 좋습니다.

처음부터 완벽한 구조를 만들려고 하실 필요는 없습니다. 다만 작은 예제라도 책임 분리, 예외 처리, 설정 관리, 로그 확인 같은 기본 습관을 함께 가져가시면 나중에 프로젝트가 커졌을 때 훨씬 덜 흔들리게 됩니다. 특히 Spring Boot는 편의 기능이 많아서 '왜 이렇게 동작하는지'를 놓치기 쉬우므로, 동작 원리를 설명할 수 있을 정도로 천천히 정리해두시는 것이 좋습니다.

## 마무리 정리

SpringBoot 입문 1-1: IntelliJ로 첫 프로젝트 만들고 실행까지 (완전 실습 가이드) 글을 공부하실 때는 한 번 읽고 끝내기보다, 직접 실행해보고 일부러 에러도 만들어보시고, 설정이나 코드 값을 조금씩 바꿔보시는 방식으로 접근해보시길 권합니다. 그렇게 하시면 단순 요약 글보다 훨씬 오래 기억에 남고, 다음 주제인 데이터 접근, 보안, 테스트, 운영 자동화로 넘어갈 때도 이해가 자연스럽게 이어집니다. 지금 단계에서 가장 중요한 것은 완벽함보다 흐름을 잡는 것이니, 천천히 따라오셔도 충분합니다.

<!-- codex-springboot-detail:end -->
