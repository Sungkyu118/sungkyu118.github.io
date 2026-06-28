---
layout: post
title: "LayoutBuilder로 Flutter 반응형 레이아웃 만들기"
date: 2026-05-15 01:20:00 +0900
category: Flutter
permalink: /flutter/responsive-layoutbuilder
description: "LayoutBuilder를 사용해 화면 크기에 따라 반응형 UI를 나누는 기준과 구현 패턴을 정리합니다."
image:
  path: "/assets/img/og/flutter-series-cover.svg"
  alt: "Flutter 시리즈 공통 대표 이미지"
---

# LayoutBuilder로 Flutter 반응형 레이아웃 만들기

> LayoutBuilder를 사용해 화면 크기에 따라 반응형 UI를 나누는 기준과 구현 패턴을 정리합니다.
>
> 이전 글: [Dio Interceptor로 토큰, 로그, 에러 처리를 공통화하기](/flutter/dio-interceptor)
> 다음 글: [CustomScrollView와 Sliver: AppBar, 목록, 그리드를 한 스크롤로 묶기](/flutter/customscrollview-slivers)
> 함께 보면 좋은 글:
> - [Container와 BoxDecoration: 색상 충돌 에러부터 실전 스타일링까지](/flutter/containercolor)
> - [CustomScrollView와 Sliver: AppBar, 목록, 그리드를 한 스크롤로 묶기](/flutter/customscrollview-slivers)

Flutter 앱은 휴대폰, 태블릿, 웹, 데스크톱까지 다양한 화면에서 실행될 수 있습니다. 처음에는 특정 기기 크기에 맞춰 `SizedBox(width: 360)` 같은 값을 넣어도 화면이 그럭저럭 보일 수 있습니다. 하지만 기기가 바뀌거나 가로 모드가 되거나 웹 브라우저 크기가 달라지면 레이아웃이 쉽게 깨집니다. 반응형 레이아웃은 이런 상황을 대비해 화면 크기에 따라 배치를 바꾸는 방식입니다.

`LayoutBuilder`는 부모가 자식에게 줄 수 있는 크기 제약을 기준으로 UI를 다르게 구성할 수 있게 해줍니다. 단순히 전체 화면 크기만 보는 `MediaQuery`와 달리, 특정 영역 안에서 사용할 수 있는 너비를 기준으로 판단할 수 있다는 점이 장점입니다.

## LayoutBuilder 기본 구조

`LayoutBuilder`는 `builder`에서 `BoxConstraints`를 제공합니다.

```dart
LayoutBuilder(
  builder: (context, constraints) {
    final width = constraints.maxWidth;

    if (width < 600) {
      return const MobileLayout();
    }

    return const TabletLayout();
  },
)
```

`constraints.maxWidth`는 해당 위젯이 사용할 수 있는 최대 너비입니다. 이것을 기준으로 모바일, 태블릿, 데스크톱 레이아웃을 나눌 수 있습니다.

## MediaQuery와의 차이

`MediaQuery.of(context).size.width`는 보통 전체 화면의 너비를 알려줍니다.

```dart
final screenWidth = MediaQuery.of(context).size.width;
```

반면 `LayoutBuilder`는 부모가 이 위젯에게 허용한 너비를 알려줍니다. 예를 들어 데스크톱 화면에서 오른쪽 사이드 패널 안에 들어간 위젯은 전체 화면은 넓어도 실제 사용할 수 있는 영역은 좁을 수 있습니다. 이때는 MediaQuery보다 LayoutBuilder가 더 정확한 판단 기준이 됩니다.

## 모바일과 태블릿 레이아웃 나누기

상품 목록 화면을 예로 들어보겠습니다. 좁은 화면에서는 1열 목록, 넓은 화면에서는 2열 그리드로 보여주고 싶습니다.

```dart
class ProductResponsivePage extends StatelessWidget {
  const ProductResponsivePage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Products')),
      body: LayoutBuilder(
        builder: (context, constraints) {
          final crossAxisCount = constraints.maxWidth < 600 ? 1 : 2;

          return GridView.builder(
            padding: const EdgeInsets.all(16),
            gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(
              crossAxisCount: crossAxisCount,
              mainAxisSpacing: 12,
              crossAxisSpacing: 12,
              childAspectRatio: crossAxisCount == 1 ? 4 : 1.4,
            ),
            itemCount: 20,
            itemBuilder: (context, index) {
              return Card(
                child: Center(child: Text('상품 $index')),
              );
            },
          );
        },
      ),
    );
  }
}
```

여기서 `childAspectRatio`도 화면 구조에 따라 다르게 주었습니다. 모바일 1열 카드에서는 가로로 긴 카드가 자연스럽고, 태블릿 2열에서는 조금 더 박스형 카드가 어울릴 수 있습니다.

## 데스크톱에서는 사이드바 추가하기

화면이 충분히 넓으면 모바일과 완전히 다른 구조를 사용할 수도 있습니다.

```dart
class ResponsiveShell extends StatelessWidget {
  const ResponsiveShell({super.key});

  @override
  Widget build(BuildContext context) {
    return LayoutBuilder(
      builder: (context, constraints) {
        if (constraints.maxWidth >= 900) {
          return Row(
            children: const [
              SizedBox(
                width: 240,
                child: NavigationRailExample(),
              ),
              VerticalDivider(width: 1),
              Expanded(child: ContentArea()),
            ],
          );
        }

        return const ContentArea();
      },
    );
  }
}
```

모바일에서는 하단 탭을 쓰고, 데스크톱에서는 사이드바를 쓰는 구조도 가능합니다. 하단 탭 구성은 [BottomNavigationBar 글](/flutter/navigator)에서 다룬 개념과 연결됩니다.

## 고정 크기 남용 피하기

반응형 레이아웃에서 자주 하는 실수는 모든 것을 고정 크기로 맞추는 것입니다.

```dart
Container(
  width: 360,
  height: 200,
  child: const Text('고정 크기'),
)
```

이런 코드는 특정 기기에서는 예쁘게 보이지만, 작은 화면에서는 overflow가 나고 큰 화면에서는 어색하게 남는 공간이 생깁니다. 가능한 경우 `Expanded`, `Flexible`, `FractionallySizedBox`, `ConstrainedBox`를 활용하는 편이 좋습니다.

```dart
ConstrainedBox(
  constraints: const BoxConstraints(maxWidth: 720),
  child: const Padding(
    padding: EdgeInsets.all(16),
    child: Text('너무 넓어지지 않는 본문'),
  ),
)
```

웹이나 태블릿에서는 본문이 화면 전체 너비로 늘어나면 읽기 어려울 수 있습니다. 그래서 최대 너비를 제한하고 가운데 정렬하는 패턴이 자주 쓰입니다.

## SafeArea와 스크롤 고려하기

반응형 화면에서는 단순히 너비만 볼 것이 아니라 안전 영역과 스크롤도 고려해야 합니다. 노치, 상태바, 하단 제스처 영역 때문에 콘텐츠가 가려질 수 있습니다.

```dart
SafeArea(
  child: LayoutBuilder(
    builder: (context, constraints) {
      return SingleChildScrollView(
        child: ConstrainedBox(
          constraints: BoxConstraints(minHeight: constraints.maxHeight),
          child: const Center(
            child: Text('내용'),
          ),
        ),
      );
    },
  ),
)
```

입력 폼처럼 키보드가 올라올 수 있는 화면은 스크롤 가능하게 만드는 편이 안전합니다. 작은 화면에서 버튼이 키보드에 가려지는 문제를 줄일 수 있습니다.

## Breakpoint를 상수로 관리하기

프로젝트가 커지면 화면마다 `600`, `900` 같은 숫자를 직접 쓰는 대신 breakpoint를 상수로 관리하는 것이 좋습니다.

```dart
class Breakpoints {
  static const mobile = 600.0;
  static const tablet = 900.0;
}

bool isMobile(BoxConstraints constraints) {
  return constraints.maxWidth < Breakpoints.mobile;
}

bool isDesktop(BoxConstraints constraints) {
  return constraints.maxWidth >= Breakpoints.tablet;
}
```

이렇게 하면 디자인 기준이 바뀌었을 때 한곳에서 조정할 수 있습니다.

## LayoutBuilder 남용 주의

`LayoutBuilder`가 편하다고 모든 위젯에 넣을 필요는 없습니다. 대부분의 단순 UI는 `Expanded`, `Wrap`, `Flexible`, `GridView`의 설정만으로도 충분합니다. LayoutBuilder는 "사용 가능한 공간에 따라 구조 자체가 바뀌어야 할 때" 사용하는 것이 좋습니다.

또한 LayoutBuilder의 builder는 레이아웃 과정에서 호출됩니다. 내부에서 무거운 계산이나 API 호출을 하면 안 됩니다. 이 주의점은 [성능 최적화 글](/flutter/performance-rebuilds)의 build 원칙과도 같습니다.

<!-- codex-category-inline-links:start -->

지금 읽고 계신 주제가 아직 조금 추상적으로 느껴지신다면 [MaterialApp 이해하기: Flutter 앱의 가장 바깥 구조](/flutter/materialapp), [Container와 BoxDecoration: 색상 충돌 에러부터 실전 스타일링까지](/flutter/containercolor), [CustomScrollView와 Sliver: AppBar, 목록, 그리드를 한 스크롤로 묶기](/flutter/customscrollview-slivers) 글도 함께 읽어보시면 좋겠습니다. 같은 Flutter 흐름 안에서 앞단의 배경과 다음 단계의 확장 포인트를 같이 보실 수 있어서, 지금 배우는 내용이 실제 프로젝트에서 어디에 연결되는지 훨씬 더 선명하게 이해하실 수 있습니다.

<!-- codex-category-inline-links:end -->
## 정리

`LayoutBuilder`는 부모가 허용한 크기를 기준으로 UI를 바꾸는 위젯입니다. 전체 화면 크기만 보는 MediaQuery보다 특정 영역의 실제 너비에 맞춰 반응형 판단을 할 수 있습니다. 좁은 화면에서는 1열, 넓은 화면에서는 2열이나 사이드바 구조로 바꾸는 식의 레이아웃에 잘 어울립니다. 고정 크기를 남용하지 말고, breakpoint를 일관되게 관리하며, 필요한 곳에만 LayoutBuilder를 사용하는 것이 좋은 반응형 설계의 시작입니다.

<!-- codex-category-links:start -->

## 이어서 읽어보시면 좋습니다

- [MaterialApp 이해하기: Flutter 앱의 가장 바깥 구조](/flutter/materialapp)
- [Container와 BoxDecoration: 색상 충돌 에러부터 실전 스타일링까지](/flutter/containercolor)
- [CustomScrollView와 Sliver: AppBar, 목록, 그리드를 한 스크롤로 묶기](/flutter/customscrollview-slivers)

지금 글과 바로 이어서 읽기 좋은 흐름으로 묶어두었으니, 개념을 비교해보시거나 다음 실습으로 넘어가실 때 차근차근 따라가보시면 좋겠습니다.

<!-- codex-category-links:end -->
