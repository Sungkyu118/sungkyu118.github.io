---
layout: post
title: "Memoization과 AsyncMemoizer: 같은 비동기 작업 반복 실행 막기"
date: 2026-05-15 01:10:00 +0900
category: Flutter
permalink: /flutter/memoization-async-memoizer
description: "AsyncMemoizer로 같은 비동기 작업의 중복 실행을 막고 초기 로딩을 안정화하는 패턴을 정리합니다."
image:
  path: "/assets/img/og/flutter-series-cover.svg"
  alt: "Flutter 시리즈 공통 대표 이미지"
---

# Memoization과 AsyncMemoizer: 같은 비동기 작업 반복 실행 막기

> AsyncMemoizer로 같은 비동기 작업의 중복 실행을 막고 초기 로딩을 안정화하는 패턴을 정리합니다.
>
> 이전 글: [freezed와 json_serializable로 안전한 모델 만들기](/flutter/freezed-json-basics)
> 다음 글: [Flutter Widget Test 입문: 버튼 클릭부터 비동기 화면까지](/flutter/widget-test-basics)
> 함께 보면 좋은 글:
> - [Flutter isolate와 compute: 무거운 작업으로 UI가 멈출 때](/flutter/isolate-compute)
> - [Riverpod 기본 사용법: Provider, StateProvider, AsyncValue까지](/flutter/riverpod-basics)

앱을 만들다 보면 같은 계산이나 같은 API 호출이 여러 번 반복되는 상황이 생깁니다. 사용자는 한 번만 눌렀다고 생각했는데 화면 rebuild 때문에 Future가 다시 생성되거나, 탭을 이동할 때마다 같은 초기화 작업이 반복될 수 있습니다. 이런 문제는 성능을 떨어뜨릴 뿐 아니라 중복 요청, 깜빡임, 데이터 불일치로 이어질 수 있습니다.

Memoization은 이미 계산한 결과를 기억해두고 같은 입력이 들어오면 다시 계산하지 않는 기법입니다. Flutter에서는 동기 계산뿐 아니라 비동기 초기화에도 이 개념이 자주 쓰입니다. 이번 글에서는 간단한 memoization부터 `AsyncMemoizer`로 비동기 작업을 한 번만 실행하는 방법까지 정리해보겠습니다.

## 같은 계산을 반복하는 문제

아래 함수는 숫자가 소수인지 검사합니다.

```dart
bool isPrime(int value) {
  if (value < 2) return false;

  for (var i = 2; i * i <= value; i++) {
    if (value % i == 0) return false;
  }

  return true;
}
```

작은 숫자라면 문제가 없지만, 큰 숫자를 같은 값으로 계속 검사하면 낭비입니다. 결과를 Map에 저장하면 반복 계산을 줄일 수 있습니다.

```dart
class PrimeChecker {
  final Map<int, bool> _cache = {};

  bool isPrimeMemoized(int value) {
    final cached = _cache[value];
    if (cached != null) {
      return cached;
    }

    final result = isPrime(value);
    _cache[value] = result;
    return result;
  }
}
```

이것이 memoization의 기본 형태입니다. 입력값을 key로 두고 결과를 저장합니다.

## build 안의 Future 생성 문제

Flutter에서 더 자주 만나는 문제는 `FutureBuilder`와 함께 발생합니다.

```dart
FutureBuilder<User>(
  future: fetchUser(),
  builder: (context, snapshot) {
    if (!snapshot.hasData) {
      return const CircularProgressIndicator();
    }
    return Text(snapshot.data!.name);
  },
)
```

이 코드는 build가 다시 실행될 때마다 `fetchUser()`가 새로 호출될 수 있습니다. 부모 위젯이 rebuild되거나 화면 상태가 바뀌면 같은 API 요청이 반복될 위험이 있습니다. 해결 방법은 Future를 필드에 저장하는 것입니다.

```dart
class UserPage extends StatefulWidget {
  const UserPage({super.key});

  @override
  State<UserPage> createState() => _UserPageState();
}

class _UserPageState extends State<UserPage> {
  late final Future<User> _userFuture;

  @override
  void initState() {
    super.initState();
    _userFuture = fetchUser();
  }

  @override
  Widget build(BuildContext context) {
    return FutureBuilder<User>(
      future: _userFuture,
      builder: (context, snapshot) {
        if (!snapshot.hasData) {
          return const CircularProgressIndicator();
        }
        return Text(snapshot.data!.name);
      },
    );
  }
}
```

이제 build가 여러 번 호출되어도 Future는 `initState`에서 한 번만 생성됩니다.

## AsyncMemoizer 사용하기

`AsyncMemoizer`는 비동기 작업을 한 번만 실행하고 그 결과를 재사용하도록 도와줍니다. `async` 패키지에 포함되어 있습니다.

```yaml
dependencies:
  async: ^2.11.0
```

사용 예시는 다음과 같습니다.

```dart
import 'package:async/async.dart';

class AppInitializer {
  final AsyncMemoizer<void> _memoizer = AsyncMemoizer<void>();

  Future<void> initialize() {
    return _memoizer.runOnce(() async {
      await Future<void>.delayed(const Duration(seconds: 1));
      print('초기화 완료');
    });
  }
}
```

`initialize()`를 여러 번 호출해도 내부 작업은 한 번만 실행됩니다. 이미 실행 중이라면 같은 Future를 기다리고, 완료되었다면 완료된 결과를 재사용합니다.

## 화면 초기화에 적용하기

앱 설정을 한 번만 불러오는 화면을 예로 들어보겠습니다.

```dart
class SettingsLoader {
  final AsyncMemoizer<Map<String, dynamic>> _memoizer =
      AsyncMemoizer<Map<String, dynamic>>();

  Future<Map<String, dynamic>> load() {
    return _memoizer.runOnce(() async {
      await Future<void>.delayed(const Duration(milliseconds: 500));
      return {
        'theme': 'dark',
        'notification': true,
      };
    });
  }
}
```

위젯에서는 loader를 주입받아 사용합니다.

```dart
class SettingsPage extends StatelessWidget {
  const SettingsPage({
    super.key,
    required this.loader,
  });

  final SettingsLoader loader;

  @override
  Widget build(BuildContext context) {
    return FutureBuilder<Map<String, dynamic>>(
      future: loader.load(),
      builder: (context, snapshot) {
        if (!snapshot.hasData) {
          return const CircularProgressIndicator();
        }

        final data = snapshot.data!;
        return Text('theme: ${data['theme']}');
      },
    );
  }
}
```

이 구조에서는 build가 반복되어도 loader 내부의 실제 로딩은 한 번만 수행됩니다.

## 언제 캐시를 비워야 할까

Memoization은 결과를 기억하는 기술이므로, 데이터가 바뀌는 상황에서는 캐시 무효화가 필요합니다. 예를 들어 사용자 설정을 저장한 뒤에도 이전 설정을 계속 보여주면 안 됩니다.

`AsyncMemoizer`는 한 번 실행한 결과를 계속 재사용하는 성격이 강하므로, 새로고침이 필요한 데이터에는 직접 Future를 다시 만드는 방식이 더 단순할 수 있습니다.

```dart
class RefreshableUserController extends ChangeNotifier {
  Future<User>? _future;

  Future<User> load() {
    return _future ??= fetchUser();
  }

  void refresh() {
    _future = fetchUser();
    notifyListeners();
  }
}
```

이처럼 "정말 한 번만 하면 되는 초기화"인지, "사용자가 새로고침할 수 있는 데이터"인지 구분해야 합니다.

## Riverpod과의 관계

Riverpod을 사용한다면 provider 자체가 캐싱 역할을 해주는 경우가 많습니다. 예를 들어 [Riverpod 기본 글](/flutter/riverpod-basics)의 `FutureProvider`는 provider 생명주기 안에서 Future 상태를 관리합니다. 그래서 모든 곳에 `AsyncMemoizer`를 직접 넣을 필요는 없습니다.

```dart
final userProvider = FutureProvider<User>((ref) async {
  return fetchUser();
});
```

이 경우 같은 provider를 watch하는 위젯들은 동일한 비동기 상태를 공유할 수 있습니다. 다만 provider가 dispose되거나 invalidate되면 다시 실행될 수 있으므로, 원하는 생명주기에 맞춰 설계해야 합니다.

## 자주 하는 실수

첫 번째는 캐시하면 안 되는 데이터를 캐시하는 것입니다. 검색 결과, 실시간 재고, 알림 목록처럼 자주 바뀌는 데이터는 무조건 오래 기억하면 사용자에게 오래된 정보를 보여줄 수 있습니다.

두 번째는 입력값을 고려하지 않는 것입니다. `userId`가 다른데 같은 결과를 돌려주면 심각한 버그입니다. 입력이 있는 memoization은 key를 반드시 포함해야 합니다.

```dart
final Map<String, Future<User>> _userCache = {};

Future<User> loadUser(String id) {
  return _userCache[id] ??= fetchUserById(id);
}
```

세 번째는 에러도 캐시될 수 있다는 점입니다. 첫 요청이 실패했는데 그 실패 Future를 계속 재사용하면 재시도해도 계속 실패처럼 보일 수 있습니다. 실패 시 캐시를 비우는 정책이 필요한지 확인해야 합니다.

## 정리

Memoization은 같은 작업을 반복하지 않도록 결과를 기억하는 기법입니다. Flutter에서는 build 안에서 Future를 계속 새로 만들지 않도록 `initState`에 저장하거나, 정말 한 번만 실행할 비동기 초기화에 `AsyncMemoizer`를 사용할 수 있습니다. 하지만 캐시는 항상 무효화 전략과 함께 생각해야 합니다. 데이터가 바뀔 수 있는지, 입력값이 달라지는지, 실패를 재시도해야 하는지까지 고려해야 안전하게 사용할 수 있습니다.
