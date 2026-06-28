---
layout: post
title: "Flutter 성능 최적화: 불필요한 rebuild 줄이기"
date: 2026-05-15 00:30:00 +0900
category: Flutter
permalink: /flutter/performance-rebuilds
description: "불필요한 rebuild를 줄여 Flutter 렌더링 성능을 개선하는 기본 원칙과 점검 포인트를 설명합니다."
image:
  path: "/assets/img/og/flutter-series-cover.svg"
  alt: "Flutter 시리즈 공통 대표 이미지"
---

# Flutter 성능 최적화: 불필요한 rebuild 줄이기

> 불필요한 rebuild를 줄여 Flutter 렌더링 성능을 개선하는 기본 원칙과 점검 포인트를 설명합니다.
>
> 이전 글: [CustomScrollView와 Sliver: AppBar, 목록, 그리드를 한 스크롤로 묶기](/flutter/customscrollview-slivers)
> 다음 글: [Flutter isolate와 compute: 무거운 작업으로 UI가 멈출 때](/flutter/isolate-compute)
> 함께 보면 좋은 글:
> - [Flutter isolate와 compute: 무거운 작업으로 UI가 멈출 때](/flutter/isolate-compute)
> - [CustomScrollView와 Sliver: AppBar, 목록, 그리드를 한 스크롤로 묶기](/flutter/customscrollview-slivers)

Flutter에서 rebuild는 나쁜 것이 아닙니다. Flutter는 위젯을 다시 build하는 것을 전제로 설계되어 있고, 작은 위젯 트리는 매우 빠르게 다시 그릴 수 있습니다. 문제는 rebuild 자체가 아니라, **너무 넓은 범위가 너무 자주 다시 build되거나, build 안에서 무거운 작업을 반복하는 것**입니다.

초보 단계에서는 `setState`를 호출하면 "전체 화면이 다시 그려져서 느려지는 것 아닐까?"라고 걱정하기 쉽습니다. 정확히는 해당 `State`의 `build`가 다시 실행되고, Flutter가 변경된 부분을 효율적으로 반영합니다. 그래도 화면이 커지고 이미지, 리스트, 계산 로직이 섞이면 성능 문제가 생길 수 있습니다. 이번 글에서는 실무에서 바로 적용할 수 있는 rebuild 줄이기 방법을 정리해보겠습니다.

## build는 여러 번 호출될 수 있다

`build` 메서드는 우리가 생각하는 것보다 자주 호출될 수 있습니다. 부모 상태 변경, 화면 회전, 테마 변경, 애니메이션, provider 상태 변경 등 다양한 이유로 다시 실행됩니다. 그래서 build 안에는 "여러 번 실행되어도 괜찮은 코드"만 두는 것이 좋습니다.

```dart
@override
Widget build(BuildContext context) {
  print('build 호출');
  return const Text('Hello');
}
```

디버깅용으로 print를 넣어보면 생각보다 자주 찍힐 수 있습니다. 이것만으로 문제가 되는 것은 아니지만, build 안에서 API 호출이나 복잡한 정렬을 수행하면 문제가 됩니다.

## build 안에서 API 호출하지 않기

아래 코드는 흔한 실수입니다.

```dart
@override
Widget build(BuildContext context) {
  fetchProducts();
  return const ProductList();
}
```

build가 다시 호출될 때마다 API가 다시 호출됩니다. 이러면 서버 요청이 중복되고 화면이 깜빡이며, 상태가 예상과 다르게 꼬일 수 있습니다. API 호출은 `initState`, 이벤트 핸들러, 또는 상태 관리 provider 안에서 수행하는 편이 좋습니다.

```dart
@override
void initState() {
  super.initState();
  fetchProducts();
}
```

Riverpod을 쓴다면 [Riverpod 기본 글](/flutter/riverpod-basics)처럼 `FutureProvider`로 비동기 데이터를 선언하고 화면은 결과를 구독하게 만들 수도 있습니다.

## const 생성자를 적극적으로 사용하기

변하지 않는 위젯에는 `const`를 붙입니다.

```dart
return const Padding(
  padding: EdgeInsets.all(16),
  child: Text('변하지 않는 안내 문구'),
);
```

`const`는 단순히 경고를 없애는 장식이 아닙니다. Flutter가 같은 위젯 인스턴스를 재사용할 수 있게 도와주고, 코드상으로도 "이 부분은 상태와 관계없이 변하지 않는다"는 의도를 드러냅니다.

```dart
Column(
  children: [
    const HeaderTitle(),
    Text('Count: $count'),
    const FooterInfo(),
  ],
)
```

카운터 값이 바뀌어도 `HeaderTitle`, `FooterInfo`는 변하지 않는다는 점이 명확해집니다.

## setState 범위를 좁히기

상태를 화면 최상단에 모두 몰아두면 작은 값 하나가 바뀌어도 큰 build 메서드가 다시 실행됩니다. 상태가 필요한 부분을 별도 위젯으로 분리하면 rebuild 범위를 줄일 수 있습니다.

```dart
class CounterSection extends StatefulWidget {
  const CounterSection({super.key});

  @override
  State<CounterSection> createState() => _CounterSectionState();
}

class _CounterSectionState extends State<CounterSection> {
  int count = 0;

  @override
  Widget build(BuildContext context) {
    return Row(
      children: [
        Text('$count'),
        IconButton(
          onPressed: () => setState(() => count++),
          icon: const Icon(Icons.add),
        ),
      ],
    );
  }
}
```

이렇게 하면 카운터 상태 변경은 `CounterSection` 내부에 머뭅니다. 부모 화면 전체가 복잡할수록 이런 분리가 도움이 됩니다.

## 리스트는 builder 사용하기

항목이 많은 리스트를 `children`에 직접 모두 넣으면 한 번에 많은 위젯을 만들 수 있습니다.

```dart
ListView(
  children: products.map((product) => ProductTile(product)).toList(),
)
```

항목 수가 적다면 괜찮지만, 많아질 수 있다면 `ListView.builder`를 사용하는 것이 좋습니다.

```dart
ListView.builder(
  itemCount: products.length,
  itemBuilder: (context, index) {
    final product = products[index];
    return ProductTile(product: product);
  },
)
```

`builder`는 화면에 필요한 항목을 중심으로 위젯을 생성하기 때문에 긴 목록에서 더 적합합니다. 복합 스크롤 화면이라면 [CustomScrollView 글](/flutter/customscrollview-slivers)의 `SliverList`도 고려할 수 있습니다.

## build 안의 무거운 계산 분리하기

정렬이나 필터링을 build마다 수행하면 데이터가 많을 때 느려질 수 있습니다.

```dart
@override
Widget build(BuildContext context) {
  final visibleProducts = products
      .where((product) => product.isVisible)
      .toList()
    ..sort((a, b) => a.name.compareTo(b.name));

  return ProductList(products: visibleProducts);
}
```

데이터가 작으면 문제가 없지만, 많아지면 build가 호출될 때마다 같은 계산을 반복합니다. 상태가 바뀌는 시점에 미리 계산하거나, memoization을 사용할 수 있습니다. 이 주제는 [AsyncMemoizer 글](/flutter/memoization-async-memoizer)과도 연결됩니다.

```dart
late List<Product> visibleProducts;

void updateVisibleProducts() {
  visibleProducts = products
      .where((product) => product.isVisible)
      .toList()
    ..sort((a, b) => a.name.compareTo(b.name));
}
```

## 위젯 분리는 성능과 가독성을 함께 개선한다

큰 build 메서드는 성능뿐 아니라 읽기도 어렵습니다. UI 덩어리를 작은 위젯으로 분리하면 어떤 상태가 어느 부분에 영향을 주는지 명확해집니다.

```dart
class ProductTile extends StatelessWidget {
  const ProductTile({
    super.key,
    required this.product,
    required this.onTap,
  });

  final Product product;
  final VoidCallback onTap;

  @override
  Widget build(BuildContext context) {
    return ListTile(
      title: Text(product.name),
      subtitle: Text('${product.price}원'),
      onTap: onTap,
    );
  }
}
```

작은 위젯은 `const` 적용도 쉬워지고, 테스트하기도 쉬워집니다. 다만 무조건 파일을 많이 쪼개는 것이 목표는 아닙니다. 의미 있는 UI 단위와 상태 단위로 분리하는 것이 중요합니다.

## 측정 없이 최적화하지 않기

성능 최적화에서 가장 위험한 것은 "느릴 것 같다"는 감각만으로 코드를 복잡하게 만드는 것입니다. Flutter DevTools의 Performance 탭을 사용하면 프레임 드랍, build 비용, raster 비용을 확인할 수 있습니다. 먼저 실제 문제가 있는지 보고, 문제가 있는 위치를 좁힌 뒤 최적화하는 편이 좋습니다.

`debugPrintRebuildDirtyWidgets = true;` 같은 디버그 도구도 도움이 됩니다. 어떤 위젯이 rebuild되는지 확인할 수 있기 때문입니다. 다만 디버그 모드는 릴리즈 모드보다 느리므로 최종 성능 판단은 프로파일 모드에서 확인하는 것이 더 정확합니다.

<!-- codex-category-inline-links:start -->

지금 읽고 계신 주제가 아직 조금 추상적으로 느껴지신다면 [Riverpod 기본 사용법: Provider, StateProvider, AsyncValue까지](/flutter/riverpod-basics), [Flutter isolate와 compute: 무거운 작업으로 UI가 멈출 때](/flutter/isolate-compute), [LayoutBuilder로 Flutter 반응형 레이아웃 만들기](/flutter/responsive-layoutbuilder) 글도 함께 읽어보시면 좋겠습니다. 같은 Flutter 흐름 안에서 앞단의 배경과 다음 단계의 확장 포인트를 같이 보실 수 있어서, 지금 배우는 내용이 실제 프로젝트에서 어디에 연결되는지 훨씬 더 선명하게 이해하실 수 있습니다.

<!-- codex-category-inline-links:end -->
## 정리

Flutter에서 rebuild는 자연스러운 동작입니다. 그러나 build 안에서 API 호출, 무거운 계산, 긴 리스트 전체 생성이 반복되면 성능 문제가 됩니다. 변하지 않는 위젯에는 `const`를 붙이고, 상태 범위를 작게 나누고, 긴 목록은 builder를 사용하고, 무거운 작업은 상태 변경 시점이나 별도 isolate로 분리하는 것이 좋습니다. 성능 개선은 감으로 시작하지 말고 측정으로 위치를 찾은 뒤 적용해야 합니다.

<!-- codex-category-links:start -->

## 이어서 읽어보시면 좋습니다

- [Riverpod 기본 사용법: Provider, StateProvider, AsyncValue까지](/flutter/riverpod-basics)
- [Flutter isolate와 compute: 무거운 작업으로 UI가 멈출 때](/flutter/isolate-compute)
- [LayoutBuilder로 Flutter 반응형 레이아웃 만들기](/flutter/responsive-layoutbuilder)

지금 글과 바로 이어서 읽기 좋은 흐름으로 묶어두었으니, 개념을 비교해보시거나 다음 실습으로 넘어가실 때 차근차근 따라가보시면 좋겠습니다.

<!-- codex-category-links:end -->
