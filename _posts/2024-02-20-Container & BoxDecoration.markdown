---
layout: post
title: "Container와 BoxDecoration: 색상 충돌 에러부터 실전 스타일링까지"
date: 2024-02-20 21:00:00 +0900
category: Flutter
permalink: /flutter/containercolor
description: "Container와 BoxDecoration으로 스타일을 구성할 때 자주 만나는 색상 충돌 에러와 해결법을 설명합니다."
image:
  path: "/assets/img/og/flutter-series-cover.svg"
  alt: "Flutter 시리즈 공통 대표 이미지"
tags: [Container, BoxDecoration]
---

# Container와 BoxDecoration: 색상 충돌 에러부터 실전 스타일링까지

> Container와 BoxDecoration으로 스타일을 구성할 때 자주 만나는 색상 충돌 에러와 해결법을 설명합니다.
>
> 이전 글: [MaterialApp 이해하기: Flutter 앱의 가장 바깥 구조](/flutter/materialapp)
> 다음 글: [Flutter Image 등록 및 사용: assets, pubspec.yaml, 에러 해결](/flutter/images)
> 함께 보면 좋은 글:
> - [Flutter Image 등록 및 사용: assets, pubspec.yaml, 에러 해결](/flutter/images)
> - [LayoutBuilder로 Flutter 반응형 레이아웃 만들기](/flutter/responsive-layoutbuilder)

Flutter에서 `Container`는 가장 자주 쓰는 위젯 중 하나입니다. 크기, 여백, 배경색, 테두리, 정렬을 한 번에 다룰 수 있어서 편리합니다. 하지만 편리한 만큼 처음에는 오해도 많습니다. 특히 `color`와 `decoration`을 동시에 지정했을 때 발생하는 에러는 Flutter 입문자들이 자주 만나는 문제입니다.

이 글에서는 `Container`가 어떤 역할을 하는지, `BoxDecoration`을 언제 사용하는지, 왜 `color`와 `decoration`을 동시에 쓰면 안 되는지, 그리고 실전에서 카드 UI를 어떻게 만드는지 단계적으로 정리합니다.

## Container의 기본 역할

`Container`는 자식 위젯 주변에 크기, 여백, 정렬, 장식을 입힐 수 있는 위젯입니다.

```dart
Container(
  width: 200,
  height: 80,
  alignment: Alignment.center,
  color: Colors.blue,
  child: const Text(
    'Hello',
    style: TextStyle(color: Colors.white),
  ),
)
```

위 코드는 너비 200, 높이 80의 파란 박스를 만들고, 그 안에 텍스트를 가운데 배치합니다. `alignment`는 child를 Container 내부에서 어디에 둘지 정합니다.

## padding과 margin

`padding`과 `margin`은 비슷해 보이지만 방향이 다릅니다. `padding`은 Container 안쪽 여백이고, `margin`은 Container 바깥쪽 여백입니다.

```dart
Container(
  margin: const EdgeInsets.all(16),
  padding: const EdgeInsets.all(20),
  color: Colors.amber,
  child: const Text('안쪽과 바깥쪽 여백'),
)
```

텍스트와 박스 테두리 사이를 띄우고 싶다면 `padding`, 박스와 다른 위젯 사이를 띄우고 싶다면 `margin`을 사용합니다.

## BoxDecoration이 필요한 경우

단순 배경색만 필요하면 `color` 속성을 사용해도 됩니다. 하지만 둥근 모서리, 테두리, 그림자, gradient 같은 스타일이 필요하면 `decoration`에 `BoxDecoration`을 사용합니다.

```dart
Container(
  padding: const EdgeInsets.all(16),
  decoration: BoxDecoration(
    color: Colors.white,
    borderRadius: BorderRadius.circular(16),
    border: Border.all(color: Colors.black12),
    boxShadow: const [
      BoxShadow(
        color: Color(0x22000000),
        blurRadius: 12,
        offset: Offset(0, 6),
      ),
    ],
  ),
  child: const Text('카드처럼 보이는 박스'),
)
```

이런 코드는 실제 앱에서 카드, 프로필 박스, 안내 배너를 만들 때 자주 사용합니다.

## color와 decoration을 동시에 쓰면 안 되는 이유

아래 코드는 에러가 납니다.

```dart
Container(
  color: Colors.red,
  decoration: BoxDecoration(
    borderRadius: BorderRadius.circular(12),
  ),
  child: const Text('Error'),
)
```

Flutter는 `Container.color`가 내부적으로 decoration의 단축 표현이라고 봅니다. 그래서 `color`와 `decoration`을 동시에 지정하면 "색상을 어디에 적용해야 하는가"가 충돌합니다. 해결 방법은 색상도 `BoxDecoration` 안으로 넣는 것입니다.

```dart
Container(
  decoration: BoxDecoration(
    color: Colors.red,
    borderRadius: BorderRadius.circular(12),
  ),
  child: const Text('OK'),
)
```

규칙은 간단합니다. `decoration`을 쓰기 시작했다면 배경색도 `decoration.color`에 넣습니다.

## Container를 남용하지 않기

모든 것을 Container로 감쌀 필요는 없습니다. 여백만 필요하면 `Padding`이 더 명확합니다.

```dart
const Padding(
  padding: EdgeInsets.all(16),
  child: Text('여백만 필요할 때'),
)
```

크기만 필요하면 `SizedBox`가 더 적합합니다.

```dart
const SizedBox(
  width: 120,
  height: 48,
  child: Center(child: Text('고정 크기')),
)
```

Container는 여러 역할을 한 번에 해야 할 때 좋습니다. 하지만 코드 의도를 분명히 하고 싶다면 더 구체적인 위젯을 선택하는 습관이 좋습니다.

## 이미지와 둥근 모서리

Container에 `borderRadius`를 주면 배경은 둥글어지지만 child 이미지까지 자동으로 잘리지는 않는 경우가 많습니다. 이미지까지 둥글게 자르려면 `ClipRRect`를 사용합니다.

```dart
ClipRRect(
  borderRadius: BorderRadius.circular(16),
  child: Image.asset(
    'assets/images/sample.png',
    width: 200,
    height: 120,
    fit: BoxFit.cover,
  ),
)
```

이미지 등록과 `Image.asset` 사용법은 [Flutter Image 글](/flutter/images)에서 이어서 볼 수 있습니다.

## Row 안에서 overflow가 날 때

Container에 고정 너비를 많이 주면 작은 화면에서 overflow가 발생할 수 있습니다.

```dart
Row(
  children: [
    Container(width: 250, height: 80, color: Colors.red),
    Container(width: 250, height: 80, color: Colors.blue),
  ],
)
```

화면 너비가 500보다 작으면 노란/검은 overflow 표시가 날 수 있습니다. 이때는 `Expanded`를 사용해 남은 공간을 나눠 쓰도록 만들 수 있습니다.

```dart
Row(
  children: [
    Expanded(child: Container(height: 80, color: Colors.red)),
    Expanded(child: Container(height: 80, color: Colors.blue)),
  ],
)
```

Flutter 레이아웃은 "자식이 원하는 크기"와 "부모가 허용하는 크기"의 협상입니다. Container만 보는 것이 아니라 부모 위젯이 어떤 제약을 주는지도 함께 봐야 합니다.

## 정리

`Container`는 크기, 여백, 정렬, 장식을 다룰 수 있는 강력한 위젯입니다. 단순 배경색만 필요하면 `color`, 둥근 모서리나 테두리 같은 스타일이 필요하면 `BoxDecoration`을 사용합니다. `color`와 `decoration`을 동시에 쓰면 충돌하므로 색상도 decoration 안에 넣어야 합니다. 그리고 여백만 필요하면 `Padding`, 크기만 필요하면 `SizedBox`처럼 더 명확한 위젯을 선택하면 코드가 읽기 쉬워집니다.
