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

