---
layout: post
title: "SpringBoot 입문 29: QueryDSL로 동적 조회 만들기"
date: 2026-05-17 22:20:00 +0900
category: SpringBoot
permalink: /springboot/mvc-querydsl-intro
description: "Spring Boot에서 QueryDSL로 동적 조회 조건을 만드는 기본 흐름, 왜 필요한지와 JPA 조회 코드를 어떻게 더 읽기 좋게 만들 수 있는지 설명합니다."
image: "/assets/img/og/springboot-series-cover.svg"
---


# SpringBoot 입문 29: QueryDSL로 동적 조회 만들기

> 이 글에서는 조건이 늘어나는 조회 API를 QueryDSL로 더 유연하게 만드는 기본 흐름을 소개합니다.
>
> 이전 글: [SpringBoot 입문 28: 스케줄러로 배치 작업 시작하기](/springboot/mvc-scheduler-batch-basics)
> 다음 글: [SpringBoot 입문 30: JWT 인증 기본 흐름](/springboot/mvc-jwt-auth-basics)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 27: 이벤트 기반 처리와 @Async](/springboot/mvc-async-event-listener)
> - [SpringBoot 입문 31: 공통 응답 포맷과 예외 코드 표준화](/springboot/mvc-global-response-format)


검색 조건이 많아질수록 JPQL 문자열 조립은 유지보수가 어려워진다.

## 핵심 포인트
- 타입 세이프한 쿼리 작성
- 조건 조합이 많은 화면에서 가독성 향상
- 페이징/정렬과 함께 사용하기 쉬움

## 정리
복잡한 검색 API에서는 QueryDSL이 코드 품질과 개발 생산성을 동시에 높여준다.

<!-- codex-springboot-detail:start -->

## 여기서 한 단계 더 이해해보겠습니다

조회 조건이 많아지는 순간부터는 단순 메서드 이름 기반 조회나 문자열 JPQL 조립이 빠르게 버거워집니다. QueryDSL은 이런 문제를 타입 세이프하게 해결하도록 도와주는 도구입니다. 특히 관리자 검색 화면처럼 필터가 자주 바뀌는 기능에서는 가독성과 유지보수성 차이가 꽤 크게 느껴집니다.

예를 들어 이름은 있을 수도 있고 없을 수도 있고, 상태도 선택일 수 있고, 날짜 범위도 선택일 수 있는 검색 API를 만든다고 생각해보시면 좋습니다. 이런 조건을 if문으로 문자열에 계속 이어 붙이기보다, 조건 메서드를 잘게 나누고 필요한 것만 조합하는 방식이 훨씬 안정적입니다.

이 주제에서 특히 중요하게 보셔야 하는 부분은 조건 메서드를 분리하고, null 조건은 제외하면서 동적 where 절을 만드는 흐름 입니다. 처음 읽으실 때는 모든 문장을 외우려고 하기보다, 코드가 어느 계층에 놓이고 어떤 순서로 실행되는지를 천천히 따라가보시면 좋습니다. Spring Boot 학습은 각각의 기술을 따로 외우는 공부라기보다, 여러 조각이 실제 요청 흐름 안에서 어떻게 맞물리는지 이해하는 과정에 가깝습니다.

## 예시 코드로 흐름을 잡아보겠습니다

```java
@Repository
@RequiredArgsConstructor
public class UserQueryRepository {

  private final JPAQueryFactory queryFactory;

  public List<UserEntity> search(String name, UserStatus status) {
    QUserEntity user = QUserEntity.userEntity;

    return queryFactory
        .selectFrom(user)
        .where(
            containsName(name),
            eqStatus(status)
        )
        .fetch();
  }

  private BooleanExpression containsName(String name) {
    return StringUtils.hasText(name) ? QUserEntity.userEntity.name.contains(name) : null;
  }

  private BooleanExpression eqStatus(UserStatus status) {
    return status != null ? QUserEntity.userEntity.status.eq(status) : null;
  }
}
```

위 예시는 QueryDSL을 가장 작게 체험할 수 있는 흐름으로 구성했습니다. 처음에는 코드 한 줄 한 줄의 문법보다, 선언 위치와 실행 시점을 중심으로 읽어보시는 편이 훨씬 이해가 잘 되실 것입니다. 예를 들어 설정 클래스인지, Controller인지, Service인지에 따라 책임이 달라지고, 그 책임이 곧 Spring Boot 구조를 읽는 기준이 됩니다.

또 하나 중요한 점은 '지금 당장 돌아가는 코드'와 '나중에도 유지보수하기 쉬운 코드'는 조금 다를 수 있다는 사실입니다. 그래서 예제를 실행해보신 뒤에는 변수명, 메서드명, 경로, 설정 값을 직접 한두 번 바꿔보시면서 어떤 부분이 고정 규칙이고 어떤 부분이 프로젝트 상황에 따라 달라지는지 구분해보시면 정말 큰 도움이 됩니다.

## 자주 하는 실수와 에러도 같이 보겠습니다

- Q 클래스가 생성되지 않으면 컴파일 자체가 되지 않으므로, annotation processor 설정을 먼저 점검하셔야 합니다.
- 조건 메서드가 null을 반환할 수 있다는 규칙을 모르고 그대로 사용하면 where 절 조합 방식이 낯설게 느껴질 수 있습니다.
- 검색 로직이 너무 많은 책임을 지기 시작하면 QueryDSL 도입 후에도 코드가 여전히 복잡할 수 있습니다.

이런 실수들은 대부분 개념을 몰라서라기보다, 실행 순서나 설정 위치를 한 번에 다 보지 못해서 생깁니다. 그래서 에러가 나면 조급하게 코드를 많이 바꾸기보다, 로그와 요청 흐름을 기준으로 '어디까지는 정상이고 어디서부터 어긋났는지'를 차분히 확인해보시는 편이 좋습니다.

## 실무에서는 이렇게 적용하시면 좋습니다

- 조건 하나당 메서드 하나로 분리하는 습관을 들이시면 동적 조회가 훨씬 읽기 쉬워집니다.
- 처음에는 엔티티 조회부터 익히고, 익숙해지면 DTO 프로젝션으로 확장해보시는 것이 좋습니다.
- QueryDSL은 만능이 아니라 "복잡한 조회를 읽기 좋게 만드는 도구"라는 관점으로 접근하시면 부담이 줄어듭니다.

처음부터 완벽한 구조를 만들려고 하실 필요는 없습니다. 다만 작은 예제라도 책임 분리, 예외 처리, 설정 관리, 로그 확인 같은 기본 습관을 함께 가져가시면 나중에 프로젝트가 커졌을 때 훨씬 덜 흔들리게 됩니다. 특히 Spring Boot는 편의 기능이 많아서 '왜 이렇게 동작하는지'를 놓치기 쉬우므로, 동작 원리를 설명할 수 있을 정도로 천천히 정리해두시는 것이 좋습니다.

<!-- codex-springboot-inline-links:start -->

지금 읽고 계신 주제가 아직 조금 추상적으로 느껴지신다면 [SpringBoot 입문 10: 목록 API의 기본 (Pagination/Sorting)](/springboot/mvc-pagination-sorting), [SpringBoot 입문 25: JPA N+1 문제와 fetch join](/springboot/mvc-jpa-n-plus-one), [SpringBoot 입문 8: JPA + H2로 DB 시작하기 (Entity/Repository)](/springboot/mvc-jpa-h2-first-db) 글도 함께 읽어보시면 좋습니다. 앞단의 준비 과정과 다음 단계의 확장 흐름이 자연스럽게 이어져서, 지금 배우는 내용이 프로젝트 안에서 어디에 연결되는지 더 분명하게 감을 잡으실 수 있습니다.

<!-- codex-springboot-inline-links:end -->
## 마무리 정리

SpringBoot 입문 29: QueryDSL로 동적 조회 만들기 글을 공부하실 때는 한 번 읽고 끝내기보다, 직접 실행해보고 일부러 에러도 만들어보시고, 설정이나 코드 값을 조금씩 바꿔보시는 방식으로 접근해보시길 권합니다. 그렇게 하시면 단순 요약 글보다 훨씬 오래 기억에 남고, 다음 주제인 데이터 접근, 보안, 테스트, 운영 자동화로 넘어갈 때도 이해가 자연스럽게 이어집니다. 지금 단계에서 가장 중요한 것은 완벽함보다 흐름을 잡는 것이니, 천천히 따라오셔도 충분합니다.

<!-- codex-springboot-detail:end -->


## 추가로 연습해보시면 좋습니다

QueryDSL 주제는 문법을 한 번 읽는 것만으로 끝내기보다, 작은 요구사항을 붙여가며 반복 연습할수록 이해가 훨씬 깊어집니다. 예를 들어 응답 메시지를 바꿔보거나, 예외 상황을 일부러 만들어보거나, 설정 값을 변경해본 뒤 결과가 어떻게 달라지는지 확인해보시면 '아는 것 같은 상태'에서 '설명할 수 있는 상태'로 빠르게 넘어가실 수 있습니다.

또한 이 단계에서는 정답 코드를 외우기보다, 왜 이런 구조를 선택했는지를 설명해보시는 연습이 정말 중요합니다. 스스로 소리 내어 '이 어노테이션은 왜 붙였는지', '이 설정은 어느 계층에 영향을 주는지', '이 에러는 왜 발생했는지'를 정리해보시면 학습 밀도가 훨씬 올라갑니다. 친절한 입문 글일수록 바로 따라 할 수 있어야 하고, 동시에 왜 그렇게 하는지도 이해할 수 있어야 한다는 점을 계속 기억해주시면 좋겠습니다.

<!-- codex-springboot-links:start -->

## 이어서 읽어보시면 좋습니다

- [SpringBoot 입문 10: 목록 API의 기본 (Pagination/Sorting)](/springboot/mvc-pagination-sorting)
- [SpringBoot 입문 25: JPA N+1 문제와 fetch join](/springboot/mvc-jpa-n-plus-one)
- [SpringBoot 입문 8: JPA + H2로 DB 시작하기 (Entity/Repository)](/springboot/mvc-jpa-h2-first-db)

지금 글과 연결되는 흐름으로 골라두었으니, 바로 다음 실습이나 비교 학습을 이어가실 때 참고해보시면 좋겠습니다.

<!-- codex-springboot-links:end -->
