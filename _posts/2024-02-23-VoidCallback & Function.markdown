---
layout: post
title: "VoidCallback과 Function: Flutter 위젯 사이에 이벤트 전달하기"
date: 2024-02-23 20:30:00 +0900
category: Flutter
permalink: /flutter/callback
description: "VoidCallback과 Function 차이, 콜백 전달, 파라미터 처리 방식을 위젯 분리 관점에서 설명합니다."
image:
  path: "/assets/img/og/flutter-series-cover.svg"
  alt: "Flutter 시리즈 공통 대표 이미지"
tags: [VoidCallback, Function]
---

# VoidCallback과 Function: Flutter 위젯 사이에 이벤트 전달하기

> VoidCallback과 Function 차이, 콜백 전달, 파라미터 처리 방식을 위젯 분리 관점에서 설명합니다.
>
> 이전 글: [BottomNavigationBar: 하단 탭 화면 전환을 안정적으로 구성하기](/flutter/navigator)
> 다음 글: [go_router로 Flutter 라우팅 구성하기: 기본 이동, 파라미터, redirect](/flutter/go-router)
> 함께 보면 좋은 글:
> - [Riverpod 기본 사용법: Provider, StateProvider, AsyncValue까지](/flutter/riverpod-basics)
> - [Flutter Widget Test 입문: 버튼 클릭부터 비동기 화면까지](/flutter/widget-test-basics)

Flutter를 처음 배울 때 부모 위젯과 자식 위젯 사이에서 값을 주고받는 부분이 꽤 헷갈립니다. 특히 버튼은 자식 위젯 안에 있는데 실제로 바꾸고 싶은 상태는 부모 위젯에 있을 때가 많습니다. 이때 자식이 부모의 상태를 직접 수정하려고 하면 구조가 금방 꼬입니다. Flutter에서는 보통 **부모가 함수를 만들어 자식에게 전달하고, 자식은 필요한 순간 그 함수를 호출**하는 방식으로 이벤트를 위로 올립니다.

이 글에서는 `VoidCallback`과 `Function`을 단순히 "콜백 함수"라고 외우는 대신, 언제 어떤 타입을 쓰면 좋은지, 실습 코드에서 어떤 에러가 자주 나는지까지 단계적으로 정리해보겠습니다. 이전 글인 [BottomNavigationBar 글](/flutter/navigator)에서 탭 전환을 상태로 관리했다면, 이번 글은 그 상태 변경을 다른 위젯으로 분리하는 첫걸음이라고 보면 좋습니다.

## 콜백을 쓰는 이유

예를 들어 화면에는 카운터 숫자가 있고, 버튼은 별도 위젯으로 분리하고 싶다고 해봅시다.

```dart
class CounterPage extends StatefulWidget {
  const CounterPage({super.key});

  @override
  State<CounterPage> createState() => _CounterPageState();
}

class _CounterPageState extends State<CounterPage> {
  int count = 0;

  void increase() {
    setState(() {
      count++;
    });
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      mainAxisAlignment: MainAxisAlignment.center,
      children: [
        Text('Count: $count', style: const TextStyle(fontSize: 32)),
        CounterButton(onPressed: increase),
      ],
    );
  }
}
```

여기서 중요한 점은 `CounterButton`이 `count`를 모른다는 것입니다. 자식 버튼은 "눌렸다"는 사실만 부모에게 알려주면 됩니다. 실제로 숫자를 증가시키는 책임은 부모가 가집니다. 이렇게 책임을 나누면 버튼 위젯은 다른 화면에서도 재사용하기 쉬워집니다.

## VoidCallback으로 값 없는 이벤트 전달하기

`VoidCallback`은 인자도 없고 반환값도 없는 함수 타입입니다. Flutter에서 버튼 클릭, 탭, 새로고침 같은 단순 이벤트에 가장 자주 쓰입니다.

```dart
class CounterButton extends StatelessWidget {
  const CounterButton({
    super.key,
    required this.onPressed,
  });

  final VoidCallback onPressed;

  @override
  Widget build(BuildContext context) {
    return ElevatedButton(
      onPressed: onPressed,
      child: const Text('증가'),
    );
  }
}
```

여기서 `onPressed: onPressed`가 낯설 수 있습니다. 오른쪽 `onPressed`는 부모가 전달해준 함수이고, 왼쪽 `onPressed`는 `ElevatedButton`의 속성 이름입니다. 버튼이 눌리면 Flutter가 전달받은 함수를 실행합니다.

실수로 다음처럼 작성하면 즉시 함수가 실행됩니다.

```dart
// 잘못된 예: build 되는 순간 increase()가 실행됩니다.
CounterButton(onPressed: increase());
```

`increase()`는 함수를 "호출"하는 문법이고, `increase`는 함수 자체를 "전달"하는 문법입니다. 콜백에서 가장 많이 만나는 실수라서 꼭 구분해야 합니다.

## Function으로 값을 전달하기

이번에는 버튼마다 증가시킬 값을 다르게 하고 싶다고 해봅시다. 예를 들어 `+1`, `+5`, `+10` 버튼을 만들 수 있습니다. 이때는 값 하나를 받는 함수 타입을 선언하면 됩니다.

```dart
class AddButton extends StatelessWidget {
  const AddButton({
    super.key,
    required this.amount,
    required this.onAdd,
  });

  final int amount;
  final void Function(int amount) onAdd;

  @override
  Widget build(BuildContext context) {
    return OutlinedButton(
      onPressed: () => onAdd(amount),
      child: Text('+$amount'),
    );
  }
}
```

`void Function(int amount)`는 "정수 하나를 받고 아무것도 반환하지 않는 함수"라는 뜻입니다. `Function`이라고만 쓰는 것보다 훨씬 안전합니다.

```dart
class AddCounterPage extends StatefulWidget {
  const AddCounterPage({super.key});

  @override
  State<AddCounterPage> createState() => _AddCounterPageState();
}

class _AddCounterPageState extends State<AddCounterPage> {
  int count = 0;

  void add(int amount) {
    setState(() {
      count += amount;
    });
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      mainAxisAlignment: MainAxisAlignment.center,
      children: [
        Text('$count', style: const TextStyle(fontSize: 40)),
        Wrap(
          spacing: 8,
          children: [
            AddButton(amount: 1, onAdd: add),
            AddButton(amount: 5, onAdd: add),
            AddButton(amount: 10, onAdd: add),
          ],
        ),
      ],
    );
  }
}
```

## `Function`보다 구체적인 타입을 권장하는 이유

아래처럼 `Function`만 쓰면 어떤 인자를 받아야 하는지 코드만 보고 알기 어렵습니다.

```dart
final Function callback;
```

이 경우 `callback('문자열')`, `callback(1, 2)`, `callback()`처럼 잘못 호출해도 컴파일 단계에서 충분히 잡아주지 못하는 상황이 생깁니다. 반대로 다음처럼 구체적으로 쓰면 의도가 명확해집니다.

```dart
final VoidCallback onTap;
final void Function(int id) onSelect;
final Future<void> Function(String keyword) onSearch;
```

실무에서는 콜백 이름도 중요합니다. `callback`처럼 추상적인 이름보다 `onTap`, `onChanged`, `onSubmitted`, `onRetry`, `onDelete`처럼 "언제 호출되는지"가 드러나는 이름이 좋습니다.

## 비동기 콜백 다루기

API 호출이나 저장 같은 작업은 `Future<void>`를 반환하는 경우가 많습니다. 버튼 안에서 로딩 상태까지 처리하고 싶다면 비동기 콜백 타입을 사용할 수 있습니다.

```dart
class SubmitButton extends StatefulWidget {
  const SubmitButton({
    super.key,
    required this.onSubmit,
  });

  final Future<void> Function() onSubmit;

  @override
  State<SubmitButton> createState() => _SubmitButtonState();
}

class _SubmitButtonState extends State<SubmitButton> {
  bool isLoading = false;

  Future<void> _handlePressed() async {
    if (isLoading) return;

    setState(() => isLoading = true);
    try {
      await widget.onSubmit();
    } finally {
      if (mounted) {
        setState(() => isLoading = false);
      }
    }
  }

  @override
  Widget build(BuildContext context) {
    return ElevatedButton(
      onPressed: isLoading ? null : _handlePressed,
      child: Text(isLoading ? '저장 중...' : '저장'),
    );
  }
}
```

여기서 `mounted` 체크는 중요합니다. 비동기 작업이 끝나기 전에 사용자가 화면을 떠나면 해당 위젯은 이미 dispose 되었을 수 있습니다. 그 상태에서 `setState`를 호출하면 `setState() called after dispose()` 에러가 발생합니다.

## 자주 만나는 에러와 주의사항

`The argument type 'void' can't be assigned...` 에러는 보통 함수를 전달해야 하는 곳에 함수 실행 결과를 전달했을 때 발생합니다. `onPressed: save()`가 아니라 `onPressed: save` 또는 `onPressed: () => save()`로 작성해야 합니다.

`setState() or markNeedsBuild() called during build` 에러는 build 중에 콜백이 즉시 실행되었을 때 자주 나옵니다. 위젯을 그리는 시점에는 상태를 바꾸지 말고, 사용자의 이벤트가 발생했을 때 상태를 바꿔야 합니다.

콜백이 너무 많아지는 것도 주의해야 합니다. 작은 컴포넌트라면 괜찮지만, 화면 전체 로직이 여러 콜백으로 흩어지면 흐름을 따라가기 어렵습니다. 이때는 [Riverpod 기본 글](/flutter/riverpod-basics)처럼 상태 관리 도구로 로직을 분리하는 것이 더 나을 수 있습니다.

<!-- codex-category-inline-links:start -->

지금 읽고 계신 주제가 아직 조금 추상적으로 느껴지신다면 [Riverpod 기본 사용법: Provider, StateProvider, AsyncValue까지](/flutter/riverpod-basics), [Flutter Widget Test 입문: 버튼 클릭부터 비동기 화면까지](/flutter/widget-test-basics), [BottomNavigationBar: 하단 탭 화면 전환을 안정적으로 구성하기](/flutter/navigator) 글도 함께 읽어보시면 좋겠습니다. 같은 Flutter 흐름 안에서 앞단의 배경과 다음 단계의 확장 포인트를 같이 보실 수 있어서, 지금 배우는 내용이 실제 프로젝트에서 어디에 연결되는지 훨씬 더 선명하게 이해하실 수 있습니다.

<!-- codex-category-inline-links:end -->
## 정리

`VoidCallback`은 값 없이 이벤트만 알려줄 때 사용하고, `void Function(T value)`는 자식 위젯이 부모에게 값을 함께 전달해야 할 때 사용합니다. 핵심은 자식이 부모의 상태를 직접 바꾸는 것이 아니라, 부모가 제공한 함수를 자식이 호출하도록 만드는 것입니다. 이 패턴을 익히면 버튼, 입력창, 리스트 아이템, 모달, 커스텀 위젯을 훨씬 깔끔하게 분리할 수 있습니다.

<!-- codex-category-links:start -->

## 이어서 읽어보시면 좋습니다

- [Riverpod 기본 사용법: Provider, StateProvider, AsyncValue까지](/flutter/riverpod-basics)
- [Flutter Widget Test 입문: 버튼 클릭부터 비동기 화면까지](/flutter/widget-test-basics)
- [BottomNavigationBar: 하단 탭 화면 전환을 안정적으로 구성하기](/flutter/navigator)

지금 글과 바로 이어서 읽기 좋은 흐름으로 묶어두었으니, 개념을 비교해보시거나 다음 실습으로 넘어가실 때 차근차근 따라가보시면 좋겠습니다.

<!-- codex-category-links:end -->
