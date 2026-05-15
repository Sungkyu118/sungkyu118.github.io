---
layout: post
title: "[Flutter] CustomScrollView + Sliver로 스크롤 UI 구성하기"
date: 2026-05-15 01:00:00 +0900
category: Flutter
permalink: /flutter/customscrollview-slivers
---

# [Flutter] CustomScrollView + Sliver로 스크롤 UI 구성하기

Flutter에서 "헤더 + 리스트 + 중간 섹션" 같은 복합 스크롤 화면을 만들 때 `ListView`만으로는 한계가 금방 옵니다. 이럴 때 `CustomScrollView`와 `Sliver`를 쓰면 스크롤을 자연스럽게 하나로 묶을 수 있어요.

## 1) 기본 구조

```dart
CustomScrollView(
  slivers: [
    SliverAppBar(
      pinned: true,
      expandedHeight: 160,
      flexibleSpace: const FlexibleSpaceBar(
        title: Text("Title"),
      ),
    ),
    SliverToBoxAdapter(
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Text("섹션 설명"),
      ),
    ),
    SliverList(
      delegate: SliverChildBuilderDelegate(
        (context, index) => ListTile(title: Text("item $index")),
        childCount: 50,
      ),
    ),
  ],
)
```

## 2) SliverToBoxAdapter가 중요한 이유

일반 위젯(Column, Container 등)을 sliver 리스트 사이에 끼워 넣으려면 `SliverToBoxAdapter`로 감싸야 합니다.

## 3) SliverGrid도 쉽게

```dart
SliverGrid(
  delegate: SliverChildBuilderDelegate(
    (context, index) => Card(child: Center(child: Text("$index"))),
    childCount: 30,
  ),
  gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
    crossAxisCount: 3,
    mainAxisSpacing: 8,
    crossAxisSpacing: 8,
  ),
)
```

## 4) 실전 팁

- `shrinkWrap: true` + 중첩 스크롤은 성능 이슈가 생기기 쉬워요.
- 가능하면 "스크롤은 한 개"로 만들고, 내부는 sliver로 조합하는 방식이 안정적입니다.
- `SliverAppBar`의 `pinned/floating/snap` 조합으로 UX가 크게 달라집니다.

## 정리

복합 스크롤 화면은 `CustomScrollView + Sliver*`로 구성하면 레이아웃도 깔끔해지고 성능도 좋아지는 경우가 많습니다.

