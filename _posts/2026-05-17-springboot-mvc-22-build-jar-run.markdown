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

<!-- codex-springboot-detail:start -->

## 여기서 한 단계 더 이해해보겠습니다

개발 환경에서 IntelliJ 실행 버튼으로 켜는 것과, 실제 배포 가능한 산출물인 JAR 파일을 만들어 실행하는 것은 다른 단계입니다. JAR 실행 흐름을 이해하시면 서버에 애플리케이션을 올릴 때 무엇이 필요한지, 어떤 설정이 외부에서 주입되는지, 왜 Java 버전이 중요한지 훨씬 분명하게 보이기 시작합니다.

입문자분들은 종종 "IDE에서는 잘 되는데 서버에서는 왜 안 되지?"를 처음 여기서 경험하십니다. 보통은 JDK 버전 차이, 환경 변수 미주입, 외부 설정 파일 누락 같은 운영 차이 때문인데, JAR 실행을 직접 해보면 이런 문제를 미리 많이 줄일 수 있습니다.

이 주제에서 특히 중요하게 보셔야 하는 부분은 `bootJar`로 실행 가능한 fat jar를 만들고, `java -jar`로 외부 환경과 함께 실행하는 흐름 입니다. 처음 읽으실 때는 모든 문장을 외우려고 하기보다, 코드가 어느 계층에 놓이고 어떤 순서로 실행되는지를 천천히 따라가보시면 좋습니다. Spring Boot 학습은 각각의 기술을 따로 외우는 공부라기보다, 여러 조각이 실제 요청 흐름 안에서 어떻게 맞물리는지 이해하는 과정에 가깝습니다.

## 예시 코드로 흐름을 잡아보겠습니다

```bash
./gradlew clean bootJar

java -jar build/libs/demo-0.0.1-SNAPSHOT.jar

java -jar build/libs/demo-0.0.1-SNAPSHOT.jar \
  --spring.profiles.active=dev \
  --server.port=8081
```

위 예시는 JAR 빌드와 실행을 가장 작게 체험할 수 있는 흐름으로 구성했습니다. 처음에는 코드 한 줄 한 줄의 문법보다, 선언 위치와 실행 시점을 중심으로 읽어보시는 편이 훨씬 이해가 잘 되실 것입니다. 예를 들어 설정 클래스인지, Controller인지, Service인지에 따라 책임이 달라지고, 그 책임이 곧 Spring Boot 구조를 읽는 기준이 됩니다.

또 하나 중요한 점은 '지금 당장 돌아가는 코드'와 '나중에도 유지보수하기 쉬운 코드'는 조금 다를 수 있다는 사실입니다. 그래서 예제를 실행해보신 뒤에는 변수명, 메서드명, 경로, 설정 값을 직접 한두 번 바꿔보시면서 어떤 부분이 고정 규칙이고 어떤 부분이 프로젝트 상황에 따라 달라지는지 구분해보시면 정말 큰 도움이 됩니다.

## 자주 하는 실수와 에러도 같이 보겠습니다

- 일반 `jar`와 Spring Boot 실행 JAR를 혼동하면 필요한 의존성이 빠져 실행이 실패할 수 있습니다.
- 서버 Java 버전이 로컬과 다르면 class file version 오류가 발생할 수 있습니다.
- 프로필이나 환경 변수를 전달하지 않고 실행해놓고, 로컬과 다른 DB에 붙지 않는다고 당황하시는 경우가 많습니다.

이런 실수들은 대부분 개념을 몰라서라기보다, 실행 순서나 설정 위치를 한 번에 다 보지 못해서 생깁니다. 그래서 에러가 나면 조급하게 코드를 많이 바꾸기보다, 로그와 요청 흐름을 기준으로 '어디까지는 정상이고 어디서부터 어긋났는지'를 차분히 확인해보시는 편이 좋습니다.

## 실무에서는 이렇게 적용하시면 좋습니다

- IDE 실행만 믿지 마시고, JAR을 직접 만들어 한 번은 CLI로 실행해보시는 습관이 좋습니다.
- 산출물 이름과 위치를 익혀두시면 Docker, CI, 배포 자동화 단계로 넘어갈 때 수월합니다.
- 실행 로그를 보면서 실제 어떤 프로필과 포트로 올라왔는지 꼭 확인해보시는 것이 좋습니다.

처음부터 완벽한 구조를 만들려고 하실 필요는 없습니다. 다만 작은 예제라도 책임 분리, 예외 처리, 설정 관리, 로그 확인 같은 기본 습관을 함께 가져가시면 나중에 프로젝트가 커졌을 때 훨씬 덜 흔들리게 됩니다. 특히 Spring Boot는 편의 기능이 많아서 '왜 이렇게 동작하는지'를 놓치기 쉬우므로, 동작 원리를 설명할 수 있을 정도로 천천히 정리해두시는 것이 좋습니다.

<!-- codex-springboot-inline-links:start -->

지금 읽고 계신 주제가 아직 조금 추상적으로 느껴지신다면 [SpringBoot 입문 4: Gradle(build.gradle) 기본과 실행/빌드 흐름](/springboot/mvc-gradle-build-basics), [SpringBoot 입문 23: Docker로 실행하기(가장 단순한 Dockerfile)](/springboot/mvc-dockerize-basics), [SpringBoot 입문 33: GitHub Actions로 CI 자동화](/springboot/mvc-github-actions-ci) 글도 함께 읽어보시면 좋습니다. 앞단의 준비 과정과 다음 단계의 확장 흐름이 자연스럽게 이어져서, 지금 배우는 내용이 프로젝트 안에서 어디에 연결되는지 더 분명하게 감을 잡으실 수 있습니다.

<!-- codex-springboot-inline-links:end -->
## 마무리 정리

SpringBoot 입문 22: JAR 빌드(bootJar)하고 실행하기 글을 공부하실 때는 한 번 읽고 끝내기보다, 직접 실행해보고 일부러 에러도 만들어보시고, 설정이나 코드 값을 조금씩 바꿔보시는 방식으로 접근해보시길 권합니다. 그렇게 하시면 단순 요약 글보다 훨씬 오래 기억에 남고, 다음 주제인 데이터 접근, 보안, 테스트, 운영 자동화로 넘어갈 때도 이해가 자연스럽게 이어집니다. 지금 단계에서 가장 중요한 것은 완벽함보다 흐름을 잡는 것이니, 천천히 따라오셔도 충분합니다.

<!-- codex-springboot-detail:end -->

## 추가로 연습해보시면 좋습니다

JAR 빌드와 실행 주제는 문법을 한 번 읽는 것만으로 끝내기보다, 작은 요구사항을 붙여가며 반복 연습할수록 이해가 훨씬 깊어집니다. 예를 들어 응답 메시지를 바꿔보거나, 예외 상황을 일부러 만들어보거나, 설정 값을 변경해본 뒤 결과가 어떻게 달라지는지 확인해보시면 '아는 것 같은 상태'에서 '설명할 수 있는 상태'로 빠르게 넘어가실 수 있습니다.

또한 이 단계에서는 정답 코드를 외우기보다, 왜 이런 구조를 선택했는지를 설명해보시는 연습이 정말 중요합니다. 스스로 소리 내어 '이 어노테이션은 왜 붙였는지', '이 설정은 어느 계층에 영향을 주는지', '이 에러는 왜 발생했는지'를 정리해보시면 학습 밀도가 훨씬 올라갑니다. 친절한 입문 글일수록 바로 따라 할 수 있어야 하고, 동시에 왜 그렇게 하는지도 이해할 수 있어야 한다는 점을 계속 기억해주시면 좋겠습니다.


## 추가로 연습해보시면 좋습니다

JAR 빌드와 실행 주제는 문법을 한 번 읽는 것만으로 끝내기보다, 작은 요구사항을 붙여가며 반복 연습할수록 이해가 훨씬 깊어집니다. 예를 들어 응답 메시지를 바꿔보거나, 예외 상황을 일부러 만들어보거나, 설정 값을 변경해본 뒤 결과가 어떻게 달라지는지 확인해보시면 '아는 것 같은 상태'에서 '설명할 수 있는 상태'로 빠르게 넘어가실 수 있습니다.

또한 이 단계에서는 정답 코드를 외우기보다, 왜 이런 구조를 선택했는지를 설명해보시는 연습이 정말 중요합니다. 스스로 소리 내어 '이 어노테이션은 왜 붙였는지', '이 설정은 어느 계층에 영향을 주는지', '이 에러는 왜 발생했는지'를 정리해보시면 학습 밀도가 훨씬 올라갑니다. 친절한 입문 글일수록 바로 따라 할 수 있어야 하고, 동시에 왜 그렇게 하는지도 이해할 수 있어야 한다는 점을 계속 기억해주시면 좋겠습니다.

<!-- codex-springboot-links:start -->

## 이어서 읽어보시면 좋습니다

- [SpringBoot 입문 4: Gradle(build.gradle) 기본과 실행/빌드 흐름](/springboot/mvc-gradle-build-basics)
- [SpringBoot 입문 23: Docker로 실행하기(가장 단순한 Dockerfile)](/springboot/mvc-dockerize-basics)
- [SpringBoot 입문 33: GitHub Actions로 CI 자동화](/springboot/mvc-github-actions-ci)

지금 글과 연결되는 흐름으로 골라두었으니, 바로 다음 실습이나 비교 학습을 이어가실 때 참고해보시면 좋겠습니다.

<!-- codex-springboot-links:end -->
