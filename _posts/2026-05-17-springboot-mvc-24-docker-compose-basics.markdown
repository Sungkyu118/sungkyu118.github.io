---
layout: post
title: "SpringBoot 입문 24: Docker Compose로 앱+DB 함께 실행"
date: 2026-05-17 22:20:00 +0900
category: SpringBoot
permalink: /springboot/mvc-docker-compose-basics
description: "Spring Boot 애플리케이션과 데이터베이스를 Docker Compose로 함께 실행하는 방법, 서비스 이름 기반 연결과 로컬 개발 환경 통일 포인트를 정리합니다."
image: "/assets/img/og/springboot-series-cover.svg"
---


# SpringBoot 입문 24: Docker Compose로 앱+DB 함께 실행

> 이 글에서는 애플리케이션과 데이터베이스를 Compose로 같이 띄워 로컬 환경을 더 배포에 가깝게 맞춥니다.
>
> 이전 글: [SpringBoot 입문 23: Docker로 실행하기(가장 단순한 Dockerfile)](/springboot/mvc-dockerize-basics)
> 다음 글: [SpringBoot 입문 25: JPA N+1 문제와 fetch join](/springboot/mvc-jpa-n-plus-one)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 22: JAR 빌드(bootJar)하고 실행하기](/springboot/mvc-build-jar-run)
> - [SpringBoot 입문 26: Redis 캐시 기본 적용](/springboot/mvc-redis-cache-basics)


이번 글에서는 Spring Boot 애플리케이션과 MySQL을 Docker Compose로 함께 실행한다.

## 핵심 포인트
- `docker-compose.yml` 하나로 다중 컨테이너 관리
- 서비스 이름으로 DB 호스트를 지정 (`db`)
- 로컬 개발 환경을 팀원과 동일하게 맞출 수 있음

## 실행 순서
```bash
docker compose up --build
docker compose down
```

## 정리
Compose를 사용하면 스프링 앱과 데이터베이스를 한 번에 올리고 내릴 수 있어 개발 반복 속도가 빨라진다.

<!-- codex-springboot-detail:start -->

## 여기서 한 단계 더 이해해보겠습니다

애플리케이션만 컨테이너로 띄우는 단계에서 한 걸음 더 가면, DB나 Redis처럼 함께 필요한 서비스도 같이 올리고 싶어집니다. Docker Compose는 이런 여러 컨테이너를 한 번에 정의하고 실행하기 좋습니다. 특히 Spring Boot + DB 조합을 로컬에서 재현할 때 매우 유용합니다.

처음에는 Compose 파일 문법이 조금 낯설 수 있지만, 결국 "어떤 서비스를 어떤 이미지로 띄우고", "어떤 포트를 열고", "환경 변수는 무엇으로 줄지"를 한 파일에 적는 것입니다. 이 흐름을 이해하면 팀원과 개발 환경을 맞추는 데도 큰 도움이 됩니다.

이 주제에서 특히 중요하게 보셔야 하는 부분은 앱과 데이터베이스를 서비스 단위로 정의하고, 서비스 이름 기반 네트워크 연결을 이해하는 것 입니다. 처음 읽으실 때는 모든 문장을 외우려고 하기보다, 코드가 어느 계층에 놓이고 어떤 순서로 실행되는지를 천천히 따라가보시면 좋습니다. Spring Boot 학습은 각각의 기술을 따로 외우는 공부라기보다, 여러 조각이 실제 요청 흐름 안에서 어떻게 맞물리는지 이해하는 과정에 가깝습니다.

## 예시 코드로 흐름을 잡아보겠습니다

```yaml
version: "3.8"

services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: demo
      MYSQL_ROOT_PASSWORD: root
    ports:
      - "3306:3306"

  app:
    build: .
    depends_on:
      - mysql
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/demo
      SPRING_DATASOURCE_USERNAME: root
      SPRING_DATASOURCE_PASSWORD: root
    ports:
      - "8080:8080"
```

위 예시는 Docker Compose을 가장 작게 체험할 수 있는 흐름으로 구성했습니다. 처음에는 코드 한 줄 한 줄의 문법보다, 선언 위치와 실행 시점을 중심으로 읽어보시는 편이 훨씬 이해가 잘 되실 것입니다. 예를 들어 설정 클래스인지, Controller인지, Service인지에 따라 책임이 달라지고, 그 책임이 곧 Spring Boot 구조를 읽는 기준이 됩니다.

또 하나 중요한 점은 '지금 당장 돌아가는 코드'와 '나중에도 유지보수하기 쉬운 코드'는 조금 다를 수 있다는 사실입니다. 그래서 예제를 실행해보신 뒤에는 변수명, 메서드명, 경로, 설정 값을 직접 한두 번 바꿔보시면서 어떤 부분이 고정 규칙이고 어떤 부분이 프로젝트 상황에 따라 달라지는지 구분해보시면 정말 큰 도움이 됩니다.

## 자주 하는 실수와 에러도 같이 보겠습니다

- Compose 안에서 DB 호스트를 `localhost`로 쓰면 앱 컨테이너 자기 자신을 보게 되어 연결에 실패합니다.
- `depends_on`은 시작 순서만 보장할 뿐 DB가 실제로 준비 완료되었는지는 보장하지 않는다는 점을 놓치기 쉽습니다.
- 개발용 비밀번호를 Compose 파일에 그대로 두고 운영에도 같은 방식을 가져가면 보안상 좋지 않습니다.

이런 실수들은 대부분 개념을 몰라서라기보다, 실행 순서나 설정 위치를 한 번에 다 보지 못해서 생깁니다. 그래서 에러가 나면 조급하게 코드를 많이 바꾸기보다, 로그와 요청 흐름을 기준으로 '어디까지는 정상이고 어디서부터 어긋났는지'를 차분히 확인해보시는 편이 좋습니다.

## 실무에서는 이렇게 적용하시면 좋습니다

- 서비스 이름을 호스트명처럼 쓴다는 개념을 꼭 익혀두시면 Compose를 이해하는 데 큰 도움이 됩니다.
- 처음에는 로컬 개발 편의를 위한 구성으로 시작하고, 운영 배포 구성은 별도로 분리해 생각하시는 편이 좋습니다.
- 로그를 볼 때는 `docker compose logs -f`처럼 각 서비스 로그를 함께 보시는 습관을 들이시면 원인 추적이 빨라집니다.

처음부터 완벽한 구조를 만들려고 하실 필요는 없습니다. 다만 작은 예제라도 책임 분리, 예외 처리, 설정 관리, 로그 확인 같은 기본 습관을 함께 가져가시면 나중에 프로젝트가 커졌을 때 훨씬 덜 흔들리게 됩니다. 특히 Spring Boot는 편의 기능이 많아서 '왜 이렇게 동작하는지'를 놓치기 쉬우므로, 동작 원리를 설명할 수 있을 정도로 천천히 정리해두시는 것이 좋습니다.

<!-- codex-springboot-inline-links:start -->

지금 읽고 계신 주제가 아직 조금 추상적으로 느껴지신다면 [SpringBoot 입문 5: application.yml과 설정 관리(Profiles) 기본](/springboot/mvc-application-yml-profiles), [SpringBoot 입문 23: Docker로 실행하기(가장 단순한 Dockerfile)](/springboot/mvc-dockerize-basics), [SpringBoot 입문 32: Testcontainers로 MySQL 통합 테스트](/springboot/mvc-testcontainers-mysql) 글도 함께 읽어보시면 좋습니다. 앞단의 준비 과정과 다음 단계의 확장 흐름이 자연스럽게 이어져서, 지금 배우는 내용이 프로젝트 안에서 어디에 연결되는지 더 분명하게 감을 잡으실 수 있습니다.

<!-- codex-springboot-inline-links:end -->
## 마무리 정리

SpringBoot 입문 24: Docker Compose로 앱+DB 함께 실행 글을 공부하실 때는 한 번 읽고 끝내기보다, 직접 실행해보고 일부러 에러도 만들어보시고, 설정이나 코드 값을 조금씩 바꿔보시는 방식으로 접근해보시길 권합니다. 그렇게 하시면 단순 요약 글보다 훨씬 오래 기억에 남고, 다음 주제인 데이터 접근, 보안, 테스트, 운영 자동화로 넘어갈 때도 이해가 자연스럽게 이어집니다. 지금 단계에서 가장 중요한 것은 완벽함보다 흐름을 잡는 것이니, 천천히 따라오셔도 충분합니다.

<!-- codex-springboot-detail:end -->

## 추가로 연습해보시면 좋습니다

Docker Compose 주제는 문법을 한 번 읽는 것만으로 끝내기보다, 작은 요구사항을 붙여가며 반복 연습할수록 이해가 훨씬 깊어집니다. 예를 들어 응답 메시지를 바꿔보거나, 예외 상황을 일부러 만들어보거나, 설정 값을 변경해본 뒤 결과가 어떻게 달라지는지 확인해보시면 '아는 것 같은 상태'에서 '설명할 수 있는 상태'로 빠르게 넘어가실 수 있습니다.

또한 이 단계에서는 정답 코드를 외우기보다, 왜 이런 구조를 선택했는지를 설명해보시는 연습이 정말 중요합니다. 스스로 소리 내어 '이 어노테이션은 왜 붙였는지', '이 설정은 어느 계층에 영향을 주는지', '이 에러는 왜 발생했는지'를 정리해보시면 학습 밀도가 훨씬 올라갑니다. 친절한 입문 글일수록 바로 따라 할 수 있어야 하고, 동시에 왜 그렇게 하는지도 이해할 수 있어야 한다는 점을 계속 기억해주시면 좋겠습니다.


## 추가로 연습해보시면 좋습니다

Docker Compose 주제는 문법을 한 번 읽는 것만으로 끝내기보다, 작은 요구사항을 붙여가며 반복 연습할수록 이해가 훨씬 깊어집니다. 예를 들어 응답 메시지를 바꿔보거나, 예외 상황을 일부러 만들어보거나, 설정 값을 변경해본 뒤 결과가 어떻게 달라지는지 확인해보시면 '아는 것 같은 상태'에서 '설명할 수 있는 상태'로 빠르게 넘어가실 수 있습니다.

또한 이 단계에서는 정답 코드를 외우기보다, 왜 이런 구조를 선택했는지를 설명해보시는 연습이 정말 중요합니다. 스스로 소리 내어 '이 어노테이션은 왜 붙였는지', '이 설정은 어느 계층에 영향을 주는지', '이 에러는 왜 발생했는지'를 정리해보시면 학습 밀도가 훨씬 올라갑니다. 친절한 입문 글일수록 바로 따라 할 수 있어야 하고, 동시에 왜 그렇게 하는지도 이해할 수 있어야 한다는 점을 계속 기억해주시면 좋겠습니다.

<!-- codex-springboot-links:start -->

## 이어서 읽어보시면 좋습니다

- [SpringBoot 입문 5: application.yml과 설정 관리(Profiles) 기본](/springboot/mvc-application-yml-profiles)
- [SpringBoot 입문 23: Docker로 실행하기(가장 단순한 Dockerfile)](/springboot/mvc-dockerize-basics)
- [SpringBoot 입문 32: Testcontainers로 MySQL 통합 테스트](/springboot/mvc-testcontainers-mysql)

지금 글과 연결되는 흐름으로 골라두었으니, 바로 다음 실습이나 비교 학습을 이어가실 때 참고해보시면 좋겠습니다.

<!-- codex-springboot-links:end -->
