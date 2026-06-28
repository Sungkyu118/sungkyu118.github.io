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

<!-- codex-springboot-detail:start -->

## 여기서 한 단계 더 이해해보겠습니다

Docker를 사용하면 "내 컴퓨터에서는 되는데 다른 환경에서는 안 된다"는 문제를 많이 줄일 수 있습니다. 애플리케이션 실행 환경을 이미지로 묶어두기 때문에, 배포 단위가 더 명확해지고 개발 환경과 배포 환경의 차이도 줄어듭니다. Spring Boot에서는 JAR 실행 흐름을 알면 Docker화도 훨씬 쉽게 이해하실 수 있습니다.

처음 Docker를 접하실 때는 명령어가 낯설 수 있지만, 핵심은 매우 단순합니다. JAR 파일을 컨테이너 안에 복사하고, JRE가 있는 베이스 이미지에서 `java -jar`로 실행하는 것입니다. 이 구조만 이해하셔도 이후 Compose나 CI/CD까지 연결하기 쉬워집니다.

이 주제에서 특히 중요하게 보셔야 하는 부분은 JAR 산출물을 Docker 이미지에 담고, 컨테이너에서 표준적인 실행 명령으로 띄우는 흐름 입니다. 처음 읽으실 때는 모든 문장을 외우려고 하기보다, 코드가 어느 계층에 놓이고 어떤 순서로 실행되는지를 천천히 따라가보시면 좋습니다. Spring Boot 학습은 각각의 기술을 따로 외우는 공부라기보다, 여러 조각이 실제 요청 흐름 안에서 어떻게 맞물리는지 이해하는 과정에 가깝습니다.

## 예시 코드로 흐름을 잡아보겠습니다

```dockerfile
FROM eclipse-temurin:17-jre

WORKDIR /app

COPY build/libs/demo-0.0.1-SNAPSHOT.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```
```bash
docker build -t demo-app .
docker run -p 8080:8080 demo-app
```

위 예시는 Docker 기본 이미지화을 가장 작게 체험할 수 있는 흐름으로 구성했습니다. 처음에는 코드 한 줄 한 줄의 문법보다, 선언 위치와 실행 시점을 중심으로 읽어보시는 편이 훨씬 이해가 잘 되실 것입니다. 예를 들어 설정 클래스인지, Controller인지, Service인지에 따라 책임이 달라지고, 그 책임이 곧 Spring Boot 구조를 읽는 기준이 됩니다.

또 하나 중요한 점은 '지금 당장 돌아가는 코드'와 '나중에도 유지보수하기 쉬운 코드'는 조금 다를 수 있다는 사실입니다. 그래서 예제를 실행해보신 뒤에는 변수명, 메서드명, 경로, 설정 값을 직접 한두 번 바꿔보시면서 어떤 부분이 고정 규칙이고 어떤 부분이 프로젝트 상황에 따라 달라지는지 구분해보시면 정말 큰 도움이 됩니다.

## 자주 하는 실수와 에러도 같이 보겠습니다

- COPY 경로에 실제 JAR 파일명이 맞지 않으면 이미지 빌드가 되더라도 컨테이너 실행 시 파일을 찾지 못할 수 있습니다.
- 컨테이너 안에서는 `localhost`가 자기 자신을 의미하므로, DB 연결 주소를 그대로 두면 외부 DB에 붙지 못할 수 있습니다.
- 포트 매핑을 빼먹으면 컨테이너는 잘 떠 있어도 브라우저에서는 접속이 안 됩니다.

이런 실수들은 대부분 개념을 몰라서라기보다, 실행 순서나 설정 위치를 한 번에 다 보지 못해서 생깁니다. 그래서 에러가 나면 조급하게 코드를 많이 바꾸기보다, 로그와 요청 흐름을 기준으로 '어디까지는 정상이고 어디서부터 어긋났는지'를 차분히 확인해보시는 편이 좋습니다.

## 실무에서는 이렇게 적용하시면 좋습니다

- 처음에는 가장 단순한 Dockerfile로 시작하시고, 익숙해진 뒤에 멀티 스테이지나 레이어 최적화를 고민하시는 편이 좋습니다.
- 이미지 빌드 전에 JAR 파일이 실제로 생성되었는지 먼저 확인하는 습관이 중요합니다.
- Docker는 "실행 환경을 고정한다"는 관점으로 이해하시면 훨씬 편합니다.

처음부터 완벽한 구조를 만들려고 하실 필요는 없습니다. 다만 작은 예제라도 책임 분리, 예외 처리, 설정 관리, 로그 확인 같은 기본 습관을 함께 가져가시면 나중에 프로젝트가 커졌을 때 훨씬 덜 흔들리게 됩니다. 특히 Spring Boot는 편의 기능이 많아서 '왜 이렇게 동작하는지'를 놓치기 쉬우므로, 동작 원리를 설명할 수 있을 정도로 천천히 정리해두시는 것이 좋습니다.

## 마무리 정리

SpringBoot 입문 23: Docker로 실행하기(가장 단순한 Dockerfile) 글을 공부하실 때는 한 번 읽고 끝내기보다, 직접 실행해보고 일부러 에러도 만들어보시고, 설정이나 코드 값을 조금씩 바꿔보시는 방식으로 접근해보시길 권합니다. 그렇게 하시면 단순 요약 글보다 훨씬 오래 기억에 남고, 다음 주제인 데이터 접근, 보안, 테스트, 운영 자동화로 넘어갈 때도 이해가 자연스럽게 이어집니다. 지금 단계에서 가장 중요한 것은 완벽함보다 흐름을 잡는 것이니, 천천히 따라오셔도 충분합니다.

<!-- codex-springboot-detail:end -->

## 추가로 연습해보시면 좋습니다

Docker 기본 이미지화 주제는 문법을 한 번 읽는 것만으로 끝내기보다, 작은 요구사항을 붙여가며 반복 연습할수록 이해가 훨씬 깊어집니다. 예를 들어 응답 메시지를 바꿔보거나, 예외 상황을 일부러 만들어보거나, 설정 값을 변경해본 뒤 결과가 어떻게 달라지는지 확인해보시면 '아는 것 같은 상태'에서 '설명할 수 있는 상태'로 빠르게 넘어가실 수 있습니다.

또한 이 단계에서는 정답 코드를 외우기보다, 왜 이런 구조를 선택했는지를 설명해보시는 연습이 정말 중요합니다. 스스로 소리 내어 '이 어노테이션은 왜 붙였는지', '이 설정은 어느 계층에 영향을 주는지', '이 에러는 왜 발생했는지'를 정리해보시면 학습 밀도가 훨씬 올라갑니다. 친절한 입문 글일수록 바로 따라 할 수 있어야 하고, 동시에 왜 그렇게 하는지도 이해할 수 있어야 한다는 점을 계속 기억해주시면 좋겠습니다.
