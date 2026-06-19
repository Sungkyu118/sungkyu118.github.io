---
layout: post
title: "Riverpod 기본 사용법: Provider, StateProvider, AsyncValue까지"
date: 2026-04-29 00:00:00 +0900
category: Flutter
permalink: /flutter/riverpod-basics
description: "Riverpod의 Provider, StateProvider, AsyncValue를 이용한 상태 관리 기본 흐름을 예제와 함께 정리합니다."
image:
  path: "/assets/img/og/flutter-riverpod-cover.svg"
  alt: "Flutter Riverpod 포스트 대표 이미지"
---

# Riverpod 기본 사용법: Provider, StateProvider, AsyncValue까지

> Riverpod의 Provider, StateProvider, AsyncValue를 이용한 상태 관리 기본 흐름을 예제와 함께 정리합니다.
>
> 이전 글: [go_router로 Flutter 라우팅 구성하기: 기본 이동, 파라미터, redirect](/flutter/go-router)
> 다음 글: [Dio Interceptor로 토큰, 로그, 에러 처리를 공통화하기](/flutter/dio-interceptor)
> 함께 보면 좋은 글:
> - [Dio Interceptor로 토큰, 로그, 에러 처리를 공통화하기](/flutter/dio-interceptor)
> - [VoidCallback과 Function: Flutter 위젯 사이에 이벤트 전달하기](/flutter/callback)

Flutter 앱을 만들다 보면 어느 순간 `setState`만으로는 화면 상태를 관리하기 버거워집니다. 화면 하나 안에서 숫자를 바꾸는 정도라면 괜찮지만, 로그인 사용자 정보, API 응답, 설정값, 캐시된 목록처럼 여러 화면이 함께 쓰는 상태는 한 위젯 안에 두기 어렵습니다. 이때 상태 관리 도구가 필요하고, 그중 Riverpod은 Flutter에서 많이 사용하는 선택지입니다.

Riverpod을 처음 보면 `ref.watch`, `ref.read`, `ProviderScope` 같은 용어가 한꺼번에 나와서 어렵게 느껴집니다. 하지만 핵심은 단순합니다. **상태나 의존성을 provider로 선언하고, 위젯은 ref를 통해 그것을 읽는다**는 구조입니다. 이 글에서는 가장 기본적인 `Provider`, `StateProvider`, `FutureProvider`, `AsyncValue`를 실습 흐름으로 정리해보겠습니다.

## 설치와 ProviderScope

`pubspec.yaml`에 `flutter_riverpod`을 추가합니다.

```yaml
dependencies:
  flutter_riverpod: ^2.0.0
```

앱의 가장 바깥을 `ProviderScope`로 감싸야 Riverpod provider들이 동작합니다.

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

void main() {
  runApp(const ProviderScope(child: MyApp()));
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return const MaterialApp(home: CounterPage());
  }
}
```

`ProviderScope`를 빠뜨리면 provider를 읽는 순간 에러가 납니다. Riverpod을 도입했는데 앱 시작 시 이상한 provider 관련 에러가 나온다면 가장 먼저 `main()`을 확인해보면 됩니다.

## Provider: 변하지 않는 값이나 의존성 제공

`Provider`는 읽기 전용 값이나 객체를 제공할 때 사용합니다. 예를 들어 API base URL, repository, service 객체처럼 앱 여러 곳에서 쓰지만 위젯 상태처럼 직접 변하지 않는 값에 적합합니다.

```dart
final apiBaseUrlProvider = Provider<String>((ref) {
  return 'https://api.example.com';
});
```

위젯에서 provider를 읽으려면 `ConsumerWidget`을 사용합니다.

```dart
class DebugConfigPage extends ConsumerWidget {
  const DebugConfigPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final baseUrl = ref.watch(apiBaseUrlProvider);

    return Scaffold(
      appBar: AppBar(title: const Text('Config')),
      body: Center(child: Text(baseUrl)),
    );
  }
}
```

`ref.watch`는 provider의 값을 구독합니다. 값이 바뀌면 해당 위젯이 다시 build 됩니다. 반대로 이벤트 핸들러 안에서 한 번만 읽고 싶다면 `ref.read`를 사용합니다.

## StateProvider: 단순한 화면 상태

숫자, 선택된 탭, 토글 여부처럼 단순한 상태는 `StateProvider`로 시작해도 충분합니다.

```dart
final counterProvider = StateProvider<int>((ref) => 0);
```

카운터 화면을 만들어보겠습니다.

```dart
class CounterPage extends ConsumerWidget {
  const CounterPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final count = ref.watch(counterProvider);

    return Scaffold(
      appBar: AppBar(title: const Text('Riverpod Counter')),
      body: Center(
        child: Text(
          '$count',
          style: const TextStyle(fontSize: 40),
        ),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () {
          ref.read(counterProvider.notifier).state++;
        },
        child: const Icon(Icons.add),
      ),
    );
  }
}
```

여기서 읽을 때는 `ref.watch(counterProvider)`를 사용하고, 수정할 때는 `ref.read(counterProvider.notifier).state`를 사용했습니다. build 안에서는 화면을 다시 그려야 하므로 watch가 자연스럽고, 버튼 클릭 핸들러에서는 그 순간 상태를 변경하기만 하면 되므로 read가 어울립니다.

## watch와 read를 헷갈리면 생기는 문제

`ref.read`로 값을 읽으면 그 provider가 바뀌어도 위젯이 자동으로 다시 그려지지 않습니다. 그래서 화면에 보여줄 값은 보통 `watch`를 사용해야 합니다.

```dart
// 화면 표시용 값은 watch
final count = ref.watch(counterProvider);

// 버튼 클릭처럼 이벤트 처리에서는 read
onPressed: () {
  ref.read(counterProvider.notifier).state = 0;
}
```

반대로 이벤트 핸들러 안에서 매번 `watch`를 쓰려고 하면 문맥상 맞지 않거나 불필요한 rebuild 의존성이 생깁니다. 간단히 기억하면 "화면에 그릴 값은 watch, 행동할 때는 read"입니다.

## FutureProvider와 AsyncValue

API 호출처럼 비동기 데이터는 `FutureProvider`로 표현할 수 있습니다.

```dart
class User {
  const User({required this.name});

  final String name;
}

final userProvider = FutureProvider<User>((ref) async {
  await Future<void>.delayed(const Duration(seconds: 1));
  return const User(name: 'Sungkyu');
});
```

`FutureProvider`를 watch하면 `AsyncValue<T>`가 반환됩니다. 이 값은 loading, error, data 세 가지 상태를 안전하게 다룰 수 있게 해줍니다.

```dart
class UserPage extends ConsumerWidget {
  const UserPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final userAsync = ref.watch(userProvider);

    return Scaffold(
      appBar: AppBar(title: const Text('User')),
      body: Center(
        child: userAsync.when(
          loading: () => const CircularProgressIndicator(),
          error: (error, stackTrace) => Text('에러: $error'),
          data: (user) => Text('안녕하세요, ${user.name}님'),
        ),
      ),
    );
  }
}
```

비동기 처리를 직접 `bool isLoading`, `String? error`, `User? data`로 나눠서 관리할 수도 있지만, 상태 조합이 많아지면 실수하기 쉽습니다. `AsyncValue`를 사용하면 로딩, 실패, 성공을 빠뜨리지 않고 다룰 수 있습니다.

## repository를 provider로 분리하기

실제 앱에서는 API 호출 코드를 provider 안에 직접 많이 쓰기보다 repository로 분리합니다. 네트워크 계층은 [Dio Interceptor 글](/flutter/dio-interceptor)처럼 별도 클라이언트로 만들고, provider는 의존성을 연결하는 역할을 하게 두면 좋습니다.

```dart
class UserRepository {
  Future<User> fetchMe() async {
    await Future<void>.delayed(const Duration(milliseconds: 500));
    return const User(name: 'Sungkyu');
  }
}

final userRepositoryProvider = Provider<UserRepository>((ref) {
  return UserRepository();
});

final meProvider = FutureProvider<User>((ref) async {
  final repository = ref.watch(userRepositoryProvider);
  return repository.fetchMe();
});
```

이 구조의 장점은 테스트가 쉬워진다는 것입니다. 테스트에서는 `userRepositoryProvider`를 가짜 repository로 바꿔치기할 수 있습니다.

## 자주 만나는 에러와 주의사항

`ProviderScope`가 없다는 에러가 나면 앱 최상단을 확인해야 합니다. `runApp(const MyApp())`처럼 되어 있으면 `ProviderScope`로 감싸야 합니다.

provider 안에서 `BuildContext`를 과하게 사용하려는 것도 좋지 않습니다. provider는 UI와 분리된 상태/의존성 계층으로 보는 편이 안전합니다. 화면 이동이나 dialog 표시 같은 UI 작업은 위젯에서 처리하고, provider는 데이터와 상태 변화에 집중시키는 것이 좋습니다.

`StateProvider`에 로직이 계속 늘어나는 것도 신호입니다. 처음에는 단순 카운터였는데 검증, API 호출, 여러 필드 변경이 들어가기 시작하면 notifier 기반 구조로 옮기는 것이 좋습니다. 단순 상태는 `StateProvider`, 복잡한 상태 변경 규칙은 notifier, 비동기 데이터는 `FutureProvider` 또는 `AsyncNotifier` 쪽으로 확장한다고 생각하면 됩니다.

## 정리

Riverpod은 처음부터 모든 개념을 한 번에 외우려고 하면 어렵습니다. 먼저 `ProviderScope`로 앱을 감싸고, `Provider`로 의존성을 제공하고, `StateProvider`로 단순 상태를 바꾸고, `FutureProvider`와 `AsyncValue`로 비동기 데이터를 표시하는 흐름부터 익히면 됩니다. 이후 앱이 커지면 repository, notifier, 테스트 override로 자연스럽게 확장할 수 있습니다.
