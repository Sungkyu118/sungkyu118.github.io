---
layout: post
title: "SpringBoot 입문 29: QueryDSL로 동적 조회 만들기"
date: 2026-05-17 22:20:00 +0900
category: SpringBoot
permalink: /springboot/mvc-querydsl-intro
---

# SpringBoot 입문 29: QueryDSL로 동적 조회 만들기

검색 조건이 많아질수록 JPQL 문자열 조립은 유지보수가 어려워진다.

## 핵심 포인트
- 타입 세이프한 쿼리 작성
- 조건 조합이 많은 화면에서 가독성 향상
- 페이징/정렬과 함께 사용하기 쉬움

## 정리
복잡한 검색 API에서는 QueryDSL이 코드 품질과 개발 생산성을 동시에 높여준다.
