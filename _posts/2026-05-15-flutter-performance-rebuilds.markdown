---
layout: post
title: "[Flutter] 성능 기본기: 불필요한 rebuild 줄이기 (const, 분리, Selector)"
date: 2026-05-15 00:00:00 +0900
category: Flutter
permalink: /flutter/flutter-performance-rebuilds
---

# [Flutter] 성능 기본기: 불필요한 rebuild 줄이기 (const, 분리, Selector)

Flutter에서 체감 성능 문제는 의외로 "너무 많은 위젯이 자주 rebuild 된다"에서 시작하는 경우가 많습니다. 이 글은 **큰 설계 변경 없이도 바로 적용 가능한** 리빌드 최적화 기본기를 정리합니다.

## 1) const를 습관처럼 붙이기

`const` 위젯은 같은 구성이라면 재사용될 수 있고(특히 빌드 트리에서), 불필요한 객체 생성을 줄입니다.

```dart
return const Text("Hello");
```

특히 자주 rebuild 되는 화면에서 `const`가 눈에 띄게 효과가 납니다.

## 2) 화면을 "작게 쪼개기"

`setState`나 Provider/Riverpod 상태 변경이 발생할 때, 빌드 범위를 줄이는 게 핵심입니다.

나쁜 예(큰 build가 한 번에 다시 그려짐):

```dart
class BigPage extends StatefulWidget {
  const BigPage({super.key});
  @override
  State<BigPage> createState() => _BigPageState();
}

class _BigPageState extends State<BigPage> {
  int count = 0;

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Text("count: $count"),
        // ... (여기 아래가 화면 대부분)
      ],
    );
  }
}
```

좋은 예(변하는 부분을 별도 위젯으로):

```dart
class BigPage extends StatefulWidget {
  const BigPage({super.key});
  @override
  State<BigPage> createState() => _BigPageState();
}

class _BigPageState extends State<BigPage> {
  int count = 0;

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        _CounterText(count: count),
        const _HeavyArea(),
      ],
    );
  }
}

class _CounterText extends StatelessWidget {
  final int count;
  const _CounterText({required this.count});

  @override
  Widget build(BuildContext context) => Text("count: $count");
}

class _HeavyArea extends StatelessWidget {
  const _HeavyArea();
  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      shrinkWrap: true,
      itemCount: 200,
      itemBuilder: (_, i) => ListTile(title: Text("item $i")),
    );
  }
}
```

이렇게만 해도 상태 변화가 생길 때 "무거운 영역"이 재계산되는 걸 크게 줄일 수 있습니다.

## 3) ListView에서 item을 최대한 const/작게

리스트 스크롤이 버벅거릴 때는 보통 **아이템 빌드 비용**이 큽니다.

- 아이템 위젯을 분리해서 재사용
- 불필요한 `Opacity`, `ClipRRect`, `BackdropFilter` 같은 비싼 위젯 남발 주의
- 네트워크 이미지면 캐싱 라이브러리 사용 고려

## 4) 상태관리에서 "전체 watch" 피하기 (Selector/부분 watch)

Provider 계열을 예로 들면, 큰 모델을 통째로 watch하면 작은 변화에도 넓게 rebuild됩니다.

`Selector`로 필요한 값만 구독:

```dart
Selector<MyModel, int>(
  selector: (_, model) => model.count,
  builder: (_, count, __) => Text("$count"),
)
```

Riverpod을 쓰는 경우에도 `ref.watch(provider.select((s) => s.someField))` 같은 "부분 구독" 패턴이 핵심입니다.

## 5) 진짜로 리빌드가 문제인지 확인하기

체감이 애매하면 "왜 느린지"부터 확인하는 게 좋습니다.

- Flutter DevTools의 Performance 탭에서 프레임 드랍 확인
- Widget rebuild가 많은지(불필요한 setState 등) 구조 점검
- 애니메이션/스크롤 중이면 레이아웃/페인트 비용 큰 위젯 확인

## 정리

- `const` 붙이기
- 변화 영역을 작은 위젯으로 분리
- 리스트 아이템 빌드 비용 줄이기
- 상태관리에서 부분 구독(Selector/select) 사용

이 기본기만 챙겨도, 앱이 커져도 "느려지는 시점"을 꽤 뒤로 미룰 수 있어요.

