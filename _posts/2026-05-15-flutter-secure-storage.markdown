---
layout: post
title: "[Flutter] flutter_secure_storage로 토큰 안전하게 저장하기 (주의점 포함)"
date: 2026-05-15 01:10:00 +0900
category: Flutter
permalink: /flutter/flutter-secure-storage
---

# [Flutter] flutter_secure_storage로 토큰 안전하게 저장하기 (주의점 포함)

로그인 토큰/세션 정보는 보통 `SharedPreferences`에 그대로 저장하면 안 됩니다. 민감 정보는 플랫폼의 보안 저장소(iOS Keychain / Android Keystore)를 활용하는 게 기본이고, Flutter에서는 `flutter_secure_storage`가 가장 흔한 선택입니다.

## 1) 설치

```yaml
dependencies:
  flutter_secure_storage: ^9.0.0
```

## 2) 기본 사용법

```dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

class TokenStore {
  static const _storage = FlutterSecureStorage();
  static const _keyAccessToken = "access_token";

  Future<void> saveAccessToken(String token) {
    return _storage.write(key: _keyAccessToken, value: token);
  }

  Future<String?> readAccessToken() {
    return _storage.read(key: _keyAccessToken);
  }

  Future<void> clear() async {
    await _storage.delete(key: _keyAccessToken);
  }
}
```

## 3) 실전에서 꼭 나오는 주의점

### iOS

- Keychain은 앱 삭제 후에도 남을 수 있어요(설정/옵션에 따라).
- "앱 재설치했는데 자동 로그인됨" 같은 현상이 생기면 Keychain 정책을 점검해야 합니다.

### Android

- 기기 보안 설정(화면 잠금) 여부, 제조사 정책에 따라 동작이 달라질 수 있습니다.
- 백업/복원, OS 업데이트 등 환경 변화에서 예외 케이스가 발생할 수 있어요.

## 4) 완벽한 보안은 아니다

`secure_storage`는 "기본값으로 안전한 저장소"를 제공하지만, 앱 자체가 탈취되거나 루팅/탈옥, 디버깅이 가능해진 환경에서는 완벽하지 않습니다.

그래도 일반 앱에서 요구하는 수준의 "토큰 저장 기본기"로는 매우 유용합니다.

## 정리

- 토큰은 `SharedPreferences` 대신 `flutter_secure_storage`를 기본으로 고려
- iOS Keychain/Android Keystore 특성 때문에 재설치/복원 등 예외 케이스를 염두
- 보안은 저장만이 아니라 "만료/갱신/로그아웃" 정책까지 함께 설계

