---
layout: post
title: "SpringBoot 입문 10: 목록 API의 기본 (Pagination/Sorting)"
date: 2026-05-17 01:40:00 +0900
category: SpringBoot
permalink: /springboot/mvc-pagination-sorting
description: "Spring Boot 목록 API에서 Pagination과 Sorting을 적용하는 방법, 페이지 번호와 크기 설계 시 주의할 점을 실습 기준으로 설명합니다."
image: "/assets/img/og/springboot-series-cover.svg"
---


# SpringBoot 입문 10: 목록 API의 기본 (Pagination/Sorting)

> 이 글에서는 목록 API에 페이지네이션과 정렬을 붙일 때 어떤 요청 형식과 응답 구조가 필요한지 정리합니다.
>
> 이전 글: [SpringBoot 입문 9: CRUD API 끝까지 만들기 (Controller/Service/Repository)](/springboot/mvc-crud-api-end-to-end)
> 다음 글: [SpringBoot 입문 11: 트랜잭션(@Transactional) 기본과 흔한 오해](/springboot/mvc-transaction-basics)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 8: JPA + H2로 DB 시작하기 (Entity/Repository)](/springboot/mvc-jpa-h2-first-db)
> - [SpringBoot 입문 12: 단위 테스트(JUnit5 + Mockito)로 Service 검증하기](/springboot/mvc-unit-test-junit-mockito)


목록 API를 만들 때 가장 흔한 실수는 "전체 데이터를 다 내려주는 것"입니다. 데이터가 늘면 바로 느려지고, 결국 장애가 됩니다.

목표:

- 페이지네이션(Page) 기본을 익힌다
- 정렬(Sort)을 안전하게 제공한다

## 1) Spring Data JPA Page 기본

Repository는 `JpaRepository`만 상속해도 페이지 기능을 기본 제공합니다.

```java
Page<UserEntity> page = userRepository.findAll(PageRequest.of(0, 20));
```

## 2) Controller에서 Pageable 받기

```java
import org.springframework.data.domain.Pageable;
import org.springframework.data.web.PageableDefault;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class UserListController {
  private final UserRepository userRepository;

  public UserListController(UserRepository userRepository) {
    this.userRepository = userRepository;
  }

  @GetMapping("/users")
  public Page<UserResponse> list(@PageableDefault(size = 20) Pageable pageable) {
    return userRepository.findAll(pageable).map(u -> new UserResponse(u.getId(), u.getName()));
  }
}
```

이제 호출 예:

- `/users?page=0&size=20`
- `/users?page=1&size=20`
- `/users?page=0&size=20&sort=id,desc`

## 3) 정렬은 "허용 컬럼만" 열어두기

초보자 단계에서는 정렬을 다 열어두기 쉽지만, 운영에서는 쿼리 비용이 폭발할 수 있습니다.

실무에서는:

- 정렬 가능한 필드를 제한하거나
- 별도의 sort 파라미터를 정의해서 분기

## 4) 흔한 실수

- size를 너무 크게 허용해서 한 번에 과도한 데이터가 내려감
- 정렬을 아무 컬럼이나 허용해서 인덱스 없는 컬럼 정렬로 느려짐

## 정리

- 목록 API는 Page가 기본
- size/sort는 정책으로 제한하는 게 운영에서 안전하다

<!-- codex-springboot-detail:start -->

## 여기서 한 단계 더 이해해보겠습니다

목록 API를 만들 때 모든 데이터를 한 번에 내려주기 시작하면 성능과 사용성 두 측면에서 금방 한계가 옵니다. 페이징과 정렬은 단순한 옵션처럼 보여도, 실제 서비스에서는 가장 자주 쓰이는 기본기입니다. 특히 관리자 화면, 검색 화면, 게시판, 로그 조회 같은 기능에서는 거의 필수라고 보셔도 좋습니다.

입문 단계에서는 "page=0, size=10" 같은 값이 조금 낯설 수 있지만, 이것만 이해하셔도 대량 데이터 처리 방식이 달라집니다. 데이터를 잘라서 주는 방식과 정렬 기준을 명확히 하는 방식은 이후 QueryDSL, 캐시, 인덱스 튜닝으로도 이어지는 중요한 출발점입니다.

이 주제에서 특히 중요하게 보셔야 하는 부분은 `Pageable`로 요청을 받고, `Page<T>`를 응답으로 다루며, 정렬 기준을 명시적으로 관리하는 흐름 입니다. 처음 읽으실 때는 모든 문장을 외우려고 하기보다, 코드가 어느 계층에 놓이고 어떤 순서로 실행되는지를 천천히 따라가보시면 좋습니다. Spring Boot 학습은 각각의 기술을 따로 외우는 공부라기보다, 여러 조각이 실제 요청 흐름 안에서 어떻게 맞물리는지 이해하는 과정에 가깝습니다.

## 예시 코드로 흐름을 잡아보겠습니다

```java
package com.example.demo.user;

import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;

public interface UserRepository extends JpaRepository<UserEntity, Long> {
  Page<UserEntity> findAllByNameContaining(String keyword, Pageable pageable);
}
```
```java
package com.example.demo.web;

import com.example.demo.user.UserEntity;
import com.example.demo.user.UserRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.web.bind.annotation.*;

@RestController
@RequiredArgsConstructor
@RequestMapping("/users")
public class UserPagingController {
  private final UserRepository userRepository;

  @GetMapping
  public Page<UserEntity> list(@RequestParam(defaultValue = "") String keyword,
                               Pageable pageable) {
    return userRepository.findAllByNameContaining(keyword, pageable);
  }
}
```

위 예시는 페이징과 정렬을 가장 작게 체험할 수 있는 흐름으로 구성했습니다. 처음에는 코드 한 줄 한 줄의 문법보다, 선언 위치와 실행 시점을 중심으로 읽어보시는 편이 훨씬 이해가 잘 되실 것입니다. 예를 들어 설정 클래스인지, Controller인지, Service인지에 따라 책임이 달라지고, 그 책임이 곧 Spring Boot 구조를 읽는 기준이 됩니다.

또 하나 중요한 점은 '지금 당장 돌아가는 코드'와 '나중에도 유지보수하기 쉬운 코드'는 조금 다를 수 있다는 사실입니다. 그래서 예제를 실행해보신 뒤에는 변수명, 메서드명, 경로, 설정 값을 직접 한두 번 바꿔보시면서 어떤 부분이 고정 규칙이고 어떤 부분이 프로젝트 상황에 따라 달라지는지 구분해보시면 정말 큰 도움이 됩니다.

## 자주 하는 실수와 에러도 같이 보겠습니다

- Spring Data의 page 번호는 0부터 시작하는데, 이를 1부터라고 생각하고 쓰면 첫 페이지가 비어 보이거나 어긋날 수 있습니다.
- 존재하지 않는 필드명으로 정렬을 요청하면 런타임 오류가 발생할 수 있으니 클라이언트 입력을 그대로 신뢰하시면 안 됩니다.
- Entity를 그대로 응답하면 필요한 것보다 많은 필드가 노출되거나, 연관관계 때문에 예기치 않은 데이터가 같이 나갈 수 있습니다.

이런 실수들은 대부분 개념을 몰라서라기보다, 실행 순서나 설정 위치를 한 번에 다 보지 못해서 생깁니다. 그래서 에러가 나면 조급하게 코드를 많이 바꾸기보다, 로그와 요청 흐름을 기준으로 '어디까지는 정상이고 어디서부터 어긋났는지'를 차분히 확인해보시는 편이 좋습니다.

## 실무에서는 이렇게 적용하시면 좋습니다

- 기본 page size 상한선을 정해두시면 무거운 요청을 미리 막는 데 도움이 됩니다.
- 처음에는 `Page<T>`를 그대로 반환해도 되지만, 실무에서는 DTO와 메타 정보를 분리해주는 쪽이 더 안전합니다.
- 정렬은 클라이언트가 원하는 값을 다 허용하기보다 화이트리스트 방식으로 관리해보시는 것을 추천드립니다.

처음부터 완벽한 구조를 만들려고 하실 필요는 없습니다. 다만 작은 예제라도 책임 분리, 예외 처리, 설정 관리, 로그 확인 같은 기본 습관을 함께 가져가시면 나중에 프로젝트가 커졌을 때 훨씬 덜 흔들리게 됩니다. 특히 Spring Boot는 편의 기능이 많아서 '왜 이렇게 동작하는지'를 놓치기 쉬우므로, 동작 원리를 설명할 수 있을 정도로 천천히 정리해두시는 것이 좋습니다.

## 마무리 정리

SpringBoot 입문 10: 목록 API의 기본 (Pagination/Sorting) 글을 공부하실 때는 한 번 읽고 끝내기보다, 직접 실행해보고 일부러 에러도 만들어보시고, 설정이나 코드 값을 조금씩 바꿔보시는 방식으로 접근해보시길 권합니다. 그렇게 하시면 단순 요약 글보다 훨씬 오래 기억에 남고, 다음 주제인 데이터 접근, 보안, 테스트, 운영 자동화로 넘어갈 때도 이해가 자연스럽게 이어집니다. 지금 단계에서 가장 중요한 것은 완벽함보다 흐름을 잡는 것이니, 천천히 따라오셔도 충분합니다.

<!-- codex-springboot-detail:end -->
