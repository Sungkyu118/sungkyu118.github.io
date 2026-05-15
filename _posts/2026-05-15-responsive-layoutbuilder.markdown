---
layout: post
title: "[Flutter] 반응형 기본: LayoutBuilder로 화면 크기별 UI 나누기"
date: 2026-05-15 00:50:00 +0900
category: Flutter
permalink: /flutter/responsive-layoutbuilder
---

# [Flutter] 반응형 기본: LayoutBuilder로 화면 크기별 UI 나누기

Flutter 앱은 폰/태블릿/웹까지 한 코드로 커버하는 경우가 많습니다. 이럴 때 가장 안전한 기본기는 `LayoutBuilder`입니다. "실제 사용할 수 있는 폭"을 기준으로 UI를 분기할 수 있어요.

## 1) LayoutBuilder 기본

```dart
LayoutBuilder(
  builder: (context, constraints) {
    final width = constraints.maxWidth;

    if (width >= 1024) {
      return const DesktopHome();
    } else if (width >= 600) {
      return const TabletHome();
    } else {
      return const MobileHome();
    }
  },
)
```

## 2) Row/Column 전환 예시

모바일에서는 세로, 큰 화면에서는 좌우 분할.

```dart
LayoutBuilder(
  builder: (context, c) {
    final isWide = c.maxWidth >= 800;

    if (!isWide) {
      return Column(
        children: const [
          Header(),
          Expanded(child: Content()),
        ],
      );
    }

    return Row(
      children: const [
        SizedBox(width: 280, child: Sidebar()),
        Expanded(child: Content()),
      ],
    );
  },
)
```

## 3) MediaQuery와 차이

- `MediaQuery.of(context).size`는 "화면 전체 크기"
- `LayoutBuilder`는 "현재 위젯이 배치된 영역의 크기"

큰 화면에서 일부 영역만 반응형으로 만들 때는 `LayoutBuilder`가 더 정확합니다.

## 4) 실전 팁

- 브레이크포인트를 앱 전역 상수로 통일하면 유지보수가 쉬워요.
- 텍스트가 길어지는 언어(한글/영문)도 고려해서 여유 폭을 잡는 게 좋습니다.
- 웹/데스크톱은 hover/shortcut 같은 UX도 같이 챙기면 완성도가 올라갑니다.

## 정리

`LayoutBuilder`로 "폭 기준 분기"만 잘 잡아도, 폰과 태블릿/웹에서 화면이 어색하게 깨지는 문제를 대부분 예방할 수 있어요.

