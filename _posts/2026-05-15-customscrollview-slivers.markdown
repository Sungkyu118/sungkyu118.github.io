---
layout: post
title: "[Flutter] CustomScrollView + Sliver로 복합 스크롤 화면 구성하기"
date: 2026-05-15 01:00:00 +0900
category: Flutter
permalink: /flutter/customscrollview-slivers
---

# [Flutter] CustomScrollView + Sliver로 복합 스크롤 화면 구성하기

Flutter에서 "헤더 + 섹션 + 리스트 + 그리드" 같은 복합 스크롤 화면을 만들 때 `ListView`만으로는 한계가 금방 옵니다. 이럴 때 `CustomScrollView`와 `Sliver`를 쓰면 스크롤을 자연스럽게 하나로 묶을 수 있어요.

이 글은 아래 4가지를 모두 포함합니다.

1. 언제/왜 Sliver가 필요한지(트레이드오프)
2. 실전 예제 1개를 끝까지(헤더 + 섹션 + 리스트 + 그리드)
3. 흔한 실수/디버깅 포인트
4. 대안 비교(NestedScrollView, ListView 조합, shrinkWrap)

## 1) 언제/왜 Sliver가 필요하나 (트레이드오프)

### Sliver가 특히 좋은 상황

- `SliverAppBar` 같은 "스크롤과 상호작용하는 헤더"가 필요
- 다양한 섹션을 한 스크롤 안에서 자연스럽게 합치고 싶다
- 중첩 스크롤을 피하고 싶다(성능/버그 예방)

### 단점/주의점

- 처음엔 API가 낯설다(`SliverToBoxAdapter` 같은 개념)
- 레이아웃 구조를 잘못 잡으면 디버깅이 어려울 수 있다

하지만 패턴만 익히면 오히려 구조가 더 명확해집니다.

## 2) 실전 예제: 헤더 + 섹션 + 리스트 + 그리드 한 번에

```dart
class ComplexScrollPage extends StatelessWidget {
  const ComplexScrollPage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: CustomScrollView(
        slivers: [
          SliverAppBar(
            pinned: true,
            expandedHeight: 180,
            flexibleSpace: const FlexibleSpaceBar(
              title: Text("Dashboard"),
            ),
          ),
          SliverToBoxAdapter(
            child: Padding(
              padding: const EdgeInsets.all(16),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: const [
                  Text("오늘의 요약", style: TextStyle(fontSize: 18)),
                  SizedBox(height: 8),
                  Text("한 스크롤 안에서 섹션을 자연스럽게 조합해봅니다."),
                ],
              ),
            ),
          ),
          SliverPadding(
            padding: const EdgeInsets.symmetric(horizontal: 12),
            sliver: SliverGrid(
              delegate: SliverChildBuilderDelegate(
                (context, index) => Card(
                  child: Center(child: Text("card $index")),
                ),
                childCount: 6,
              ),
              gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
                crossAxisCount: 3,
                mainAxisSpacing: 8,
                crossAxisSpacing: 8,
                childAspectRatio: 1.6,
              ),
            ),
          ),
          const SliverToBoxAdapter(child: SizedBox(height: 12)),
          SliverList(
            delegate: SliverChildBuilderDelegate(
              (context, index) => ListTile(
                title: Text("item $index"),
                subtitle: const Text("sliver list"),
              ),
              childCount: 30,
            ),
          ),
        ],
      ),
    );
  }
}
```

핵심 아이디어:
- 일반 위젯은 `SliverToBoxAdapter`로 감싸서 끼워 넣기
- 그리드는 `SliverGrid`, 리스트는 `SliverList`
- 패딩은 `SliverPadding`으로 sliver 단위로 적용

## 3) 흔한 실수/디버깅 포인트

### (1) Sliver 안에 일반 위젯을 그냥 넣기

`slivers:`에는 Sliver 타입만 들어갈 수 있어요. 일반 위젯을 넣고 싶으면 `SliverToBoxAdapter`로 감싸야 합니다.

### (2) 중첩 스크롤 + shrinkWrap로 해결하려다 느려짐

`ListView(shrinkWrap: true)`를 여러 개 겹치면 측정/레이아웃 비용이 커져 스크롤이 버벅일 수 있어요. 가능하면 스크롤을 하나로 만들고 sliver로 합치는 편이 안정적입니다.

### (3) SliverAppBar 옵션 조합 헷갈림

- `pinned`: 스크롤해도 상단에 붙어있음
- `floating`: 스크롤 방향에 반응해 나타남
- `snap`: floating일 때 빠르게 붙는 효과

이 옵션 조합이 UX를 크게 바꿉니다.

## 4) 대안 비교

### NestedScrollView

탭/상단 헤더와 본문 스크롤을 함께 다루는 경우에 선택할 수 있습니다. 다만 구조가 복잡해질 수 있어요.

### ListView 조합

단순한 화면이면 ListView + header로도 충분합니다. 하지만 "스크롤 헤더 효과"나 "섹션/그리드 조합"이 늘어나면 Sliver가 더 깔끔해지는 경우가 많습니다.

## 정리

- 복합 스크롤은 `CustomScrollView + Sliver*`로 구성하면 레이아웃도 깔끔하고 성능도 좋은 편
- 일반 위젯은 `SliverToBoxAdapter`로 끼워 넣기
- 중첩 스크롤을 `shrinkWrap`로 억지로 풀기보단 sliver로 합치기

