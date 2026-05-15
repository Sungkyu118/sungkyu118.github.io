---
layout: post
title: "반응형 기본: LayoutBuilder로 화면 크기별 UI 나누기"
date: 2026-05-15 00:50:00 +0900
category: Flutter
permalink: /flutter/responsive-layoutbuilder
---

# 반응형 기본: LayoutBuilder로 화면 크기별 UI 나누기

Flutter 앱을 폰/태블릿/웹까지 한 코드로 커버하려면 반응형이 필수입니다. 그중에서도 `LayoutBuilder`는 가장 안전한 기본기예요. "현재 위젯이 실제로 배치된 영역의 크기"를 기준으로 UI를 분기할 수 있습니다.

이 글은 아래 4가지를 모두 포함합니다.

1. 언제/왜 LayoutBuilder를 쓰는지(트레이드오프)
2. 실전 예제 1개를 끝까지(브레이크포인트 + 레이아웃 전환)
3. 흔한 실수/디버깅 포인트
4. 대안 비교(MediaQuery, OrientationBuilder, 반응형 패키지)

## 1) 언제/왜 쓰나 (트레이드오프)

### LayoutBuilder가 특히 좋은 상황

- 같은 화면을 폰/태블릿에서 다르게 보여줘야 한다
- 웹에서 사이드바(좌측 네비) 같은 레이아웃이 필요하다
- "전체 화면 크기"가 아니라 "현재 영역" 기준으로 반응형을 하고 싶다

### 단점/주의점

- 분기 로직이 여기저기 흩어지면 유지보수가 어려워질 수 있음
- 브레이크포인트를 팀에서 통일하지 않으면 화면마다 기준이 달라짐

그래서 브레이크포인트는 상수로 모아두는 편이 좋습니다.

## 2) 실전 예제: 브레이크포인트 상수 + 레이아웃 전환

```dart
class Breakpoints {
  static const double tablet = 600;
  static const double desktop = 1024;
}
```

화면에서 분기:

```dart
class HomeShell extends StatelessWidget {
  const HomeShell({super.key});

  @override
  Widget build(BuildContext context) {
    return LayoutBuilder(
      builder: (context, constraints) {
        final w = constraints.maxWidth;

        if (w >= Breakpoints.desktop) {
          return const _DesktopHome();
        }
        if (w >= Breakpoints.tablet) {
          return const _TabletHome();
        }
        return const _MobileHome();
      },
    );
  }
}
```

### Row/Column 전환 패턴

모바일: 세로, 큰 화면: 좌우 분할(사이드바 + 컨텐츠)

```dart
class _DesktopHome extends StatelessWidget {
  const _DesktopHome();

  @override
  Widget build(BuildContext context) {
    return Row(
      children: const [
        SizedBox(width: 280, child: _Sidebar()),
        VerticalDivider(width: 1),
        Expanded(child: _Content()),
      ],
    );
  }
}
```

## 3) 흔한 실수/디버깅 포인트

### (1) MediaQuery만 믿고 분기해서 깨짐

`MediaQuery.size`는 화면 전체 크기라서, 예를 들어 웹에서 좌측 패널/다이얼로그/분할뷰 안에서는 의도와 다르게 동작할 수 있어요. 영역 기반이면 `LayoutBuilder`가 더 정확합니다.

### (2) 브레이크포인트가 화면마다 다름

한 화면은 700에서 태블릿, 다른 화면은 600에서 태블릿이면 UX가 흔들립니다. `Breakpoints` 상수로 통일 추천.

### (3) 텍스트 오버플로우

반응형이라고 레이아웃만 바꾸고 텍스트 줄바꿈/최소 폭을 안 챙기면 `RenderFlex overflow`가 나기 쉽습니다.

- `Expanded/Flexible` 사용
- 긴 텍스트는 `maxLines`, `overflow: TextOverflow.ellipsis`

## 4) 대안 비교

### MediaQuery

화면 전체 기준으로 분기할 때 간단합니다. 다만 "영역 기반"이 아니라는 점이 가장 큰 차이입니다.

### OrientationBuilder

가로/세로 방향에 따라 분기하는 경우에 좋습니다. 하지만 태블릿/웹처럼 "폭 자체가 큰 환경"에서는 orientation만으로는 부족한 경우가 많아요.

### 반응형 패키지(예: responsive_framework 등)

프로젝트가 커지면 패키지 도입도 고려할 수 있습니다. 다만 처음엔 `LayoutBuilder + Breakpoints`로 충분히 깔끔하게 갈 수 있어요.

## 정리

- `LayoutBuilder`는 "영역 크기" 기반 반응형에 강하다
- 브레이크포인트를 상수로 통일하면 유지보수가 쉬워진다
- 오버플로우/최소 폭 같은 디테일을 같이 챙겨야 UX가 좋아진다
