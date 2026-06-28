---
layout: post
title: "Flutter Widget Test 입문: 버튼 클릭부터 비동기 화면까지"
date: 2026-05-15 00:50:00 +0900
category: Flutter
permalink: /flutter/widget-test-basics
description: "Widget Test로 버튼 클릭, 비동기 화면, 위젯 상태를 검증하는 기본 흐름을 예제와 함께 설명합니다."
image:
  path: "/assets/img/og/flutter-widget-test-cover.svg"
  alt: "Flutter Widget Test 포스트 대표 이미지"
---

# Flutter Widget Test 입문: 버튼 클릭부터 비동기 화면까지

> Widget Test로 버튼 클릭, 비동기 화면, 위젯 상태를 검증하는 기본 흐름을 예제와 함께 설명합니다.
>
> 이전 글: [Memoization과 AsyncMemoizer: 같은 비동기 작업 반복 실행 막기](/flutter/memoization-async-memoizer)
> 함께 보면 좋은 글:
> - [VoidCallback과 Function: Flutter 위젯 사이에 이벤트 전달하기](/flutter/callback)
> - [Riverpod 기본 사용법: Provider, StateProvider, AsyncValue까지](/flutter/riverpod-basics)

Flutter 테스트는 크게 unit test, widget test, integration test로 나눌 수 있습니다. 그중 widget test는 실제 기기나 에뮬레이터를 띄우지 않고도 위젯이 어떻게 그려지는지, 버튼을 눌렀을 때 텍스트가 바뀌는지, 로딩 상태가 표시되는지 확인할 수 있는 테스트입니다. 화면 단위 로직을 빠르게 검증할 수 있어서 Flutter 프로젝트에서 가장 먼저 익히기 좋은 테스트 방식입니다.

테스트를 어렵게 느끼는 이유는 "무엇을 테스트해야 하는지"가 흐릿하기 때문입니다. 처음에는 복잡한 아키텍처를 검증하려고 하기보다, 사용자가 보는 화면과 누르는 행동을 기준으로 작성하면 됩니다. 이 글에서는 카운터, 콜백, 비동기 화면을 예제로 widget test의 기본 흐름을 정리해보겠습니다.

## 기본 테스트 구조

Flutter 프로젝트를 만들면 `test/widget_test.dart` 파일이 기본으로 생성됩니다. widget test에서는 `testWidgets`를 사용합니다.

```dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';

void main() {
  testWidgets('텍스트가 화면에 표시된다', (tester) async {
    await tester.pumpWidget(
      const MaterialApp(
        home: Scaffold(
          body: Text('Hello Flutter'),
        ),
      ),
    );

    expect(find.text('Hello Flutter'), findsOneWidget);
  });
}
```

`pumpWidget`은 테스트 환경에 위젯을 그립니다. `find.text`는 화면에서 텍스트를 찾고, `expect`는 기대한 결과와 맞는지 확인합니다. `findsOneWidget`은 정확히 하나의 위젯을 찾았다는 뜻입니다.

## 버튼 클릭 테스트

카운터 화면을 테스트해보겠습니다.

```dart
class CounterPage extends StatefulWidget {
  const CounterPage({super.key});

  @override
  State<CounterPage> createState() => _CounterPageState();
}

class _CounterPageState extends State<CounterPage> {
  int count = 0;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(child: Text('Count: $count')),
      floatingActionButton: FloatingActionButton(
        onPressed: () => setState(() => count++),
        child: const Icon(Icons.add),
      ),
    );
  }
}
```

테스트 코드는 다음과 같습니다.

```dart
testWidgets('버튼을 누르면 카운트가 증가한다', (tester) async {
  await tester.pumpWidget(
    const MaterialApp(home: CounterPage()),
  );

  expect(find.text('Count: 0'), findsOneWidget);

  await tester.tap(find.byIcon(Icons.add));
  await tester.pump();

  expect(find.text('Count: 1'), findsOneWidget);
});
```

`tester.tap`은 사용자가 버튼을 누르는 행동을 흉내 냅니다. 그다음 `tester.pump()`를 호출해야 상태 변경 이후 화면이 다시 그려집니다. 버튼은 눌렀는데 결과가 바뀌지 않는 테스트 실패가 나온다면 `pump` 호출을 빠뜨렸는지 확인해보면 됩니다.

## Key를 사용해 안정적으로 찾기

텍스트나 아이콘으로 위젯을 찾는 것도 가능하지만, 화면 문구가 바뀌면 테스트가 깨질 수 있습니다. 중요한 버튼이나 입력창에는 `Key`를 붙이는 방식이 안정적입니다.

```dart
ElevatedButton(
  key: const Key('submitButton'),
  onPressed: onSubmit,
  child: const Text('저장'),
)
```

테스트에서는 다음처럼 찾습니다.

```dart
await tester.tap(find.byKey(const Key('submitButton')));
await tester.pump();
```

Key는 테스트만을 위한 장치라기보다, 위젯 트리에서 특정 위젯을 식별하는 이름표입니다. 남용할 필요는 없지만 사용자 행동의 핵심이 되는 요소에는 붙여두면 테스트가 단단해집니다.

## TextField 입력 테스트

입력창에 값을 넣고 버튼을 눌렀을 때 결과가 표시되는 화면을 만들어봅시다.

```dart
class GreetingPage extends StatefulWidget {
  const GreetingPage({super.key});

  @override
  State<GreetingPage> createState() => _GreetingPageState();
}

class _GreetingPageState extends State<GreetingPage> {
  final controller = TextEditingController();
  String message = '';

  @override
  void dispose() {
    controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      home: Scaffold(
        body: Column(
          children: [
            TextField(
              key: const Key('nameInput'),
              controller: controller,
            ),
            ElevatedButton(
              key: const Key('greetButton'),
              onPressed: () {
                setState(() {
                  message = '안녕하세요, ${controller.text}님';
                });
              },
              child: const Text('인사하기'),
            ),
            Text(message),
          ],
        ),
      ),
    );
  }
}
```

테스트는 다음처럼 작성합니다.

```dart
testWidgets('이름을 입력하면 인사 문구가 표시된다', (tester) async {
  await tester.pumpWidget(const GreetingPage());

  await tester.enterText(find.byKey(const Key('nameInput')), '성규');
  await tester.tap(find.byKey(const Key('greetButton')));
  await tester.pump();

  expect(find.text('안녕하세요, 성규님'), findsOneWidget);
});
```

`enterText`는 입력창에 텍스트를 넣는 동작입니다. 실제 키보드를 띄우지 않고도 입력 결과를 테스트할 수 있습니다.

## 비동기 화면 테스트

비동기 로딩 화면은 widget test에서 자주 막히는 부분입니다. 예제를 보겠습니다.

```dart
class AsyncMessagePage extends StatefulWidget {
  const AsyncMessagePage({super.key});

  @override
  State<AsyncMessagePage> createState() => _AsyncMessagePageState();
}

class _AsyncMessagePageState extends State<AsyncMessagePage> {
  String? message;

  @override
  void initState() {
    super.initState();
    _load();
  }

  Future<void> _load() async {
    await Future<void>.delayed(const Duration(milliseconds: 300));
    if (!mounted) return;
    setState(() => message = '로딩 완료');
  }

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      home: Scaffold(
        body: Center(
          child: message == null
              ? const CircularProgressIndicator()
              : Text(message!),
        ),
      ),
    );
  }
}
```

테스트에서는 처음에는 로딩이 보이고, 시간이 지난 뒤 완료 문구가 보여야 합니다.

```dart
testWidgets('비동기 로딩 후 메시지가 표시된다', (tester) async {
  await tester.pumpWidget(const AsyncMessagePage());

  expect(find.byType(CircularProgressIndicator), findsOneWidget);

  await tester.pump(const Duration(milliseconds: 300));
  await tester.pump();

  expect(find.text('로딩 완료'), findsOneWidget);
});
```

`pumpAndSettle()`을 쓰면 애니메이션이나 예약된 프레임이 끝날 때까지 기다릴 수 있지만, 무한 애니메이션이 있는 화면에서는 테스트가 끝나지 않을 수 있습니다. `CircularProgressIndicator`처럼 계속 도는 애니메이션이 있으면 필요한 시간만큼 `pump(Duration)`을 사용하는 편이 더 명확할 때가 많습니다.

## 테스트하기 쉬운 위젯 만들기

테스트가 어려운 위젯은 보통 의존성이 내부에 숨어 있습니다. 예를 들어 위젯 안에서 직접 API 객체를 만들면 테스트에서 가짜 응답을 주입하기 어렵습니다.

```dart
class ProfilePage extends StatelessWidget {
  const ProfilePage({
    super.key,
    required this.loadName,
  });

  final Future<String> Function() loadName;

  @override
  Widget build(BuildContext context) {
    return FutureBuilder<String>(
      future: loadName(),
      builder: (context, snapshot) {
        if (!snapshot.hasData) {
          return const CircularProgressIndicator();
        }

        return Text(snapshot.data!);
      },
    );
  }
}
```

테스트에서는 가짜 함수를 넣습니다.

```dart
testWidgets('프로필 이름을 표시한다', (tester) async {
  await tester.pumpWidget(
    MaterialApp(
      home: ProfilePage(
        loadName: () async => 'Sungkyu',
      ),
    ),
  );

  await tester.pump();

  expect(find.text('Sungkyu'), findsOneWidget);
});
```

이런 구조는 [VoidCallback 글](/flutter/callback)에서 다룬 콜백 전달과도 같은 원리입니다. 외부 의존성을 주입하면 테스트가 쉬워지고 위젯 재사용성도 좋아집니다.

<!-- codex-category-inline-links:start -->

지금 읽고 계신 주제가 아직 조금 추상적으로 느껴지신다면 [VoidCallback과 Function: Flutter 위젯 사이에 이벤트 전달하기](/flutter/callback), [Riverpod 기본 사용법: Provider, StateProvider, AsyncValue까지](/flutter/riverpod-basics), [flutter_secure_storage로 토큰 안전하게 저장하기](/flutter/secure-storage) 글도 함께 읽어보시면 좋겠습니다. 같은 Flutter 흐름 안에서 앞단의 배경과 다음 단계의 확장 포인트를 같이 보실 수 있어서, 지금 배우는 내용이 실제 프로젝트에서 어디에 연결되는지 훨씬 더 선명하게 이해하실 수 있습니다.

<!-- codex-category-inline-links:end -->
## 정리

Widget test는 실제 사용자의 행동을 작은 단위로 검증하는 도구입니다. `pumpWidget`으로 화면을 그리고, `find`로 위젯을 찾고, `tap`이나 `enterText`로 행동을 수행한 뒤, `expect`로 결과를 확인합니다. 상태 변경 뒤에는 `pump`가 필요하고, 비동기 화면에서는 기다릴 시간을 명확히 제어해야 합니다. 처음부터 모든 화면을 테스트하려고 하기보다 로그인 버튼, 입력 검증, 중요한 상태 전환처럼 실패하면 치명적인 흐름부터 하나씩 추가하는 것이 좋습니다.

<!-- codex-category-links:start -->

## 이어서 읽어보시면 좋습니다

- [VoidCallback과 Function: Flutter 위젯 사이에 이벤트 전달하기](/flutter/callback)
- [Riverpod 기본 사용법: Provider, StateProvider, AsyncValue까지](/flutter/riverpod-basics)
- [flutter_secure_storage로 토큰 안전하게 저장하기](/flutter/secure-storage)

지금 글과 바로 이어서 읽기 좋은 흐름으로 묶어두었으니, 개념을 비교해보시거나 다음 실습으로 넘어가실 때 차근차근 따라가보시면 좋겠습니다.

<!-- codex-category-links:end -->
