---
layout: post
title: "[Flutter] initState 중복 호출 방지: Future 캐싱으로 한 번만 로드하기"
date: 2026-05-15 00:40:00 +0900
category: Flutter
permalink: /flutter/future-cache-load-once
---

# [Flutter] initState 중복 호출 방지: Future 캐싱으로 한 번만 로드하기

`FutureBuilder`를 쓰다가 "매 빌드마다 API 호출"을 해버리는 실수는 정말 흔합니다. 해결은 간단해요. **Future를 상태로 들고 캐싱**하면 됩니다.

이 글은 아래 4가지를 모두 포함합니다.

1. 언제/왜 이 문제가 생기는지(트레이드오프)
2. 실전 예제 1개를 끝까지(새로고침까지)
3. 흔한 실수/디버깅 포인트
4. 대안 비교(FutureProvider/Riverpod, Stream, 캐시 레이어 등)

## 1) 왜 문제가 생기나 (언제/왜)

Flutter의 `build()`는 "한 번만" 호출되는 함수가 아닙니다.

- `setState`가 불리면 다시 호출
- 상위 위젯이 rebuild되면 하위도 다시 호출될 수 있음
- MediaQuery(회전), 테마 변경 등 다양한 이유로 재빌드 가능

그런데 `build()` 안에서 `api.fetchUser()` 같은 Future를 새로 만들어버리면, "재빌드 = 재호출"이 되어버립니다.

## 2) 흔한 실수: build에서 Future 생성

```dart
@override
Widget build(BuildContext context) {
  return FutureBuilder<User>(
    future: api.fetchUser(), // build마다 새 Future 생성
    builder: ...,
  );
}
```

## 3) 해결(정석): initState에서 Future를 만들어 보관

```dart
class ProfilePage extends StatefulWidget {
  const ProfilePage({super.key});
  @override
  State<ProfilePage> createState() => _ProfilePageState();
}

class _ProfilePageState extends State<ProfilePage> {
  late Future<User> _future; // refresh를 위해 final은 피하는 편이 편함

  @override
  void initState() {
    super.initState();
    _future = api.fetchUser();
  }

  Future<void> _reload() async {
    setState(() {
      _future = api.fetchUser();
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text("Profile"),
        actions: [
          IconButton(
            onPressed: _reload,
            icon: const Icon(Icons.refresh),
          ),
        ],
      ),
      body: FutureBuilder<User>(
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
      ),
    );
  }
}
```

이제 rebuild가 여러 번 일어나도 "Future 자체는 그대로"라서 중복 호출이 발생하지 않습니다.

## 4) 실전 팁: mounted 체크

async 작업 후 `setState`를 하려면 화면이 이미 dispose 되었는지 체크하는 습관이 좋습니다.

```dart
if (!mounted) return;
setState(() { ... });
```

## 5) 흔한 실수/디버깅 포인트

### (1) refresh 구현하다가 다시 build에서 Future 만들기

새로고침도 결국 `_future`를 "재할당"하는 패턴으로 통일하는 게 깔끔합니다.

### (2) FutureBuilder가 null data로 터짐

`snapshot.data!`는 `hasData`가 보장될 때만 쓰는 게 안전합니다. 예외 케이스가 있으면 `hasData` 체크를 추가하세요.

### (3) 요청이 한 번만 간다고 믿었는데 여러 번 감

원인을 분리해서 봐야 합니다.

- Future가 한 번만 생성되는지(로그/브레이크포인트)
- API 클라이언트 내부에서 재시도/리다이렉트가 있는지
- interceptor가 중복 호출을 유발하는지

## 6) 대안 비교

### Riverpod/Provider로 올리기

화면마다 같은 데이터를 쓰거나, 앱 전체에서 캐시를 공유하고 싶다면 상태관리 레이어로 올리는 게 좋습니다.

- Riverpod: `FutureProvider`/`AsyncNotifier`
- Provider: `ChangeNotifier` + repository 캐시

### 캐시 레이어(Repository)에서 해결

UI는 단순히 `repo.getUser()`를 호출하고, repo가 내부적으로 메모이제이션(메모리 캐시)을 하는 구조도 흔합니다.

### Stream이 더 맞는 경우

실시간으로 계속 값이 변하는 데이터(예: 소켓, 위치, 센서)는 Future보다 Stream이 자연스럽습니다.

## 정리

- `build()`는 여러 번 호출될 수 있으니, `future:`에 새 Future를 계속 만들지 말기
- `initState`에서 Future를 생성해 멤버로 보관하면 중복 호출을 막을 수 있음
- 스케일이 커지면 상태관리/Repository 캐시로 해결하는 게 더 낫다

