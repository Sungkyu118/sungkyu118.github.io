---
layout: post
title: "CustomScrollView와 Sliver: AppBar, 목록, 그리드를 한 스크롤로 묶기"
date: 2026-05-15 00:10:00 +0900
category: Flutter
permalink: /flutter/customscrollview-slivers
description: "CustomScrollView와 Sliver를 이용해 AppBar, 리스트, 그리드를 하나의 스크롤로 묶는 방법을 설명합니다."
image:
  path: "/assets/img/og/flutter-series-cover.svg"
  alt: "Flutter 시리즈 공통 대표 이미지"
---

# CustomScrollView와 Sliver: AppBar, 목록, 그리드를 한 스크롤로 묶기

> CustomScrollView와 Sliver를 이용해 AppBar, 리스트, 그리드를 하나의 스크롤로 묶는 방법을 설명합니다.
>
> 이전 글: [LayoutBuilder로 Flutter 반응형 레이아웃 만들기](/flutter/responsive-layoutbuilder)
> 다음 글: [Flutter 성능 최적화: 불필요한 rebuild 줄이기](/flutter/performance-rebuilds)
> 함께 보면 좋은 글:
> - [LayoutBuilder로 Flutter 반응형 레이아웃 만들기](/flutter/responsive-layoutbuilder)
> - [Flutter 성능 최적화: 불필요한 rebuild 줄이기](/flutter/performance-rebuilds)

Flutter에서 스크롤 화면을 만들 때 가장 먼저 떠오르는 위젯은 `ListView`입니다. 단순한 목록이라면 `ListView.builder`만으로 충분합니다. 하지만 실제 앱에서는 상단에 접히는 앱바가 있고, 그 아래에 프로필 영역이 있고, 다시 가로 카드 목록이나 그리드가 이어지는 식으로 화면이 복잡해집니다. 이때 `Column` 안에 `ListView`를 넣고, 또 그 안에 `GridView`를 넣기 시작하면 스크롤 충돌과 높이 에러가 자주 발생합니다.

`CustomScrollView`와 `Sliver`는 이런 복합 스크롤 화면을 하나의 스크롤 컨텍스트로 구성하기 위한 도구입니다. 처음 보면 이름이 낯설지만, 핵심은 간단합니다. **스크롤 가능한 조각들을 sliver라는 단위로 이어 붙인다**고 생각하면 됩니다.

## ListView로 충분하지 않은 상황

다음과 같은 화면을 만든다고 해봅시다.

1. 상단에는 접히는 큰 이미지 앱바
2. 그 아래에는 소개 카드
3. 그 아래에는 상품 목록
4. 중간에는 2열 그리드

이 구조를 `SingleChildScrollView`와 `Column`으로 만들 수도 있습니다.

```dart
SingleChildScrollView(
  child: Column(
    children: [
      Header(),
      ProfileCard(),
      ProductList(),
      ProductGrid(),
    ],
  ),
)
```

하지만 `ProductList`가 많은 데이터를 한 번에 모두 만들면 성능이 나빠집니다. `ListView.builder`처럼 화면에 보이는 항목만 lazily build하는 장점을 잃기 쉽습니다. 그래서 복합 스크롤 화면에서는 `CustomScrollView`가 더 안전한 선택이 됩니다.

## 기본 구조

`CustomScrollView`는 `slivers` 배열을 받습니다.

```dart
class StorePage extends StatelessWidget {
  const StorePage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: CustomScrollView(
        slivers: [
          const SliverAppBar(
            expandedHeight: 220,
            pinned: true,
            flexibleSpace: FlexibleSpaceBar(
              title: Text('Flutter Store'),
              background: ColoredBox(color: Colors.blue),
            ),
          ),
          SliverToBoxAdapter(
            child: Padding(
              padding: const EdgeInsets.all(16),
              child: _IntroCard(),
            ),
          ),
          SliverList(
            delegate: SliverChildBuilderDelegate(
              (context, index) => ListTile(
                title: Text('상품 $index'),
                subtitle: const Text('SliverList 예제'),
              ),
              childCount: 20,
            ),
          ),
        ],
      ),
    );
  }
}
```

`SliverAppBar`는 스크롤에 반응하는 앱바입니다. `pinned: true`를 주면 접힌 뒤에도 상단에 남아 있습니다. `SliverToBoxAdapter`는 일반 위젯을 sliver 안에 넣기 위한 어댑터입니다. 일반 `Container`, `Padding`, `Card` 같은 위젯은 바로 `slivers`에 넣을 수 없기 때문에 이 어댑터가 필요합니다.

## SliverList와 SliverGrid

목록은 `SliverList`, 그리드는 `SliverGrid`를 사용합니다.

```dart
SliverGrid(
  delegate: SliverChildBuilderDelegate(
    (context, index) {
      return Card(
        child: Center(
          child: Text('카드 $index'),
        ),
      );
    },
    childCount: 8,
  ),
  gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
    crossAxisCount: 2,
    mainAxisSpacing: 8,
    crossAxisSpacing: 8,
    childAspectRatio: 1.3,
  ),
)
```

`GridView`와 비슷해 보이지만 `CustomScrollView` 안에서는 `GridView`가 아니라 `SliverGrid`를 쓰는 것이 자연스럽습니다. 이렇게 하면 앱바, 카드, 목록, 그리드가 모두 하나의 스크롤로 움직입니다.

## 일반 위젯을 넣을 때는 SliverToBoxAdapter

가장 흔한 에러 중 하나는 `slivers`에 일반 위젯을 바로 넣는 것입니다.

```dart
// 잘못된 예
CustomScrollView(
  slivers: [
    Container(height: 100, color: Colors.red),
  ],
)
```

`slivers`에는 sliver protocol을 따르는 위젯만 들어갈 수 있습니다. 일반 박스 위젯은 다음처럼 감싸야 합니다.

```dart
CustomScrollView(
  slivers: [
    SliverToBoxAdapter(
      child: Container(
        height: 100,
        color: Colors.red,
      ),
    ),
  ],
)
```

이 규칙만 이해해도 CustomScrollView의 절반은 익힌 셈입니다.

## 빈 상태와 로딩 상태 표현하기

목록 데이터가 없을 때도 sliver 구조를 유지하는 편이 좋습니다.

```dart
SliverFillRemaining(
  hasScrollBody: false,
  child: Center(
    child: Column(
      mainAxisSize: MainAxisSize.min,
      children: [
        const Icon(Icons.inbox, size: 48),
        const SizedBox(height: 12),
        const Text('표시할 데이터가 없습니다.'),
        TextButton(
          onPressed: () {},
          child: const Text('다시 시도'),
        ),
      ],
    ),
  ),
)
```

`SliverFillRemaining`은 남은 영역을 채울 때 유용합니다. 단순히 빈 메시지를 `SliverToBoxAdapter`로 넣으면 화면 가운데가 아니라 위쪽에 붙어 보일 수 있습니다.

## Nested scroll을 피해야 하는 이유

초보자 코드에서 자주 보이는 형태가 `SingleChildScrollView` 안에 `ListView.builder`를 넣고 `shrinkWrap: true`, `NeverScrollableScrollPhysics()`를 붙이는 방식입니다.

```dart
ListView.builder(
  shrinkWrap: true,
  physics: const NeverScrollableScrollPhysics(),
  itemCount: 100,
  itemBuilder: (context, index) => Text('$index'),
)
```

이 방식이 항상 틀린 것은 아니지만, 긴 목록에서는 성능 문제가 생길 수 있습니다. `shrinkWrap`은 자식 크기를 계산하기 위해 더 많은 레이아웃 비용을 요구합니다. 항목이 적은 설정 화면 정도라면 괜찮지만, 피드나 상품 목록처럼 데이터가 많다면 sliver로 구성하는 편이 좋습니다.

## 실전 화면 예시

아래는 앱바, 안내 영역, 가로 구분 제목, 그리드, 목록을 한 스크롤로 묶은 예시입니다.

```dart
class DashboardPage extends StatelessWidget {
  const DashboardPage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: CustomScrollView(
        slivers: [
          const SliverAppBar(
            title: Text('대시보드'),
            pinned: true,
            expandedHeight: 180,
            flexibleSpace: FlexibleSpaceBar(
              background: DecoratedBox(
                decoration: BoxDecoration(
                  gradient: LinearGradient(
                    colors: [Colors.indigo, Colors.cyan],
                  ),
                ),
              ),
            ),
          ),
          const SliverToBoxAdapter(
            child: Padding(
              padding: EdgeInsets.all(16),
              child: Text('오늘 확인할 항목을 모아봤습니다.'),
            ),
          ),
          SliverPadding(
            padding: const EdgeInsets.symmetric(horizontal: 16),
            sliver: SliverGrid(
              delegate: SliverChildBuilderDelegate(
                (context, index) => Card(
                  child: Center(child: Text('요약 $index')),
                ),
                childCount: 4,
              ),
              gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
                crossAxisCount: 2,
                mainAxisSpacing: 12,
                crossAxisSpacing: 12,
              ),
            ),
          ),
          SliverList.builder(
            itemCount: 30,
            itemBuilder: (context, index) => ListTile(
              title: Text('알림 $index'),
              trailing: const Icon(Icons.chevron_right),
            ),
          ),
        ],
      ),
    );
  }
}
```

## 정리

`CustomScrollView`는 복잡한 스크롤 화면을 하나의 스크롤로 묶기 위한 위젯입니다. 일반 위젯은 `SliverToBoxAdapter`, 목록은 `SliverList`, 그리드는 `SliverGrid`, 접히는 앱바는 `SliverAppBar`를 사용합니다. `Column`과 여러 스크롤 위젯을 억지로 중첩하기 전에, 화면이 하나의 스크롤 흐름인지 먼저 생각해보면 구조를 훨씬 깔끔하게 잡을 수 있습니다.
