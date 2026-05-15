---
layout: post
title: "[Flutter] initState에서 중복 호출 방지: Future 캐싱으로 한 번만 로드하기"
date: 2026-05-15 00:40:00 +0900
category: Flutter
permalink: /flutter/future-cache-load-once
---

# [Flutter] initState에서 중복 호출 방지: Future 캐싱으로 한 번만 로드하기

Flutter에서 `FutureBuilder`를 쓰다가 실수로 "매 빌드마다 API 호출"을 해버리는 일이 자주 생깁니다. 해결은 간단해요. **Future를 상태로 들고 캐싱**하면 됩니다.

## 1) 흔한 실수: build에서 Future 생성

```dart
@override
Widget build(BuildContext context) {
  return FutureBuilder<User>(
    future: api.fetchUser(), // build마다 실행될 수 있음
    builder: ...,
  );
}
```

## 2) 해결: initState에서 Future를 만들어 보관

```dart
class ProfilePage extends StatefulWidget {
  const ProfilePage({super.key});
  @override
  State<ProfilePage> createState() => _ProfilePageState();
}

class _ProfilePageState extends State<ProfilePage> {
  late final Future<User> _future;

  @override
  void initState() {
    super.initState();
    _future = api.fetchUser();
  }

  @override
  Widget build(BuildContext context) {
    return FutureBuilder<User>(
      future: _future,
      builder: (context, snapshot) {
        if (snapshot.connectionState != ConnectionState.done) {
          return const Center(child: CircularProgressIndicator());
        }
        if (snapshot.hasError) {
          return Center(child: Text("error: ${snapshot.error}"));
        }
        final user = snapshot.data!;
        return Center(child: Text(user.name));
      },
    );
  }
}
```

이렇게 하면 "처음 한 번만" 로드되고, rebuild가 되어도 Future 자체는 바뀌지 않습니다.

## 3) 새로고침이 필요하면

당연히 어떤 화면은 refresh가 필요합니다. 그때는 Future를 다시 할당하면 됩니다.

```dart
Future<void> _reload() async {
  setState(() {
    _future = api.fetchUser(); // late final이면 final 제거
  });
}
```

실전에서는 pull-to-refresh나 아이콘 버튼으로 이 패턴을 많이 씁니다.

## 4) mounted 체크(비동기 후 setState 안전)

async 작업 후 `setState`가 늦게 실행되면, 화면이 이미 pop 된 경우 에러가 날 수 있어요.

```dart
if (!mounted) return;
setState(() { ... });
```

## 정리

- `FutureBuilder`에서 `future:`는 "새 Future를 계속 만들지 않게" 캐싱한다
- 가장 간단한 방법은 `initState`에서 Future 생성 후 멤버로 보관
- refresh는 Future를 재할당하는 방식으로 구현

