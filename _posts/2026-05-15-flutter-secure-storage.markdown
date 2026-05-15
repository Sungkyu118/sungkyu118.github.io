---
layout: post
title: "[Flutter] flutter_secure_storage로 토큰 안전하게 저장하기 (실전 주의점)"
date: 2026-05-15 01:10:00 +0900
category: Flutter
permalink: /flutter/flutter-secure-storage
---

# [Flutter] flutter_secure_storage로 토큰 안전하게 저장하기 (실전 주의점)

로그인 토큰/세션 정보는 `SharedPreferences`에 그대로 저장하면 안 됩니다. 민감 정보는 플랫폼의 보안 저장소(iOS Keychain / Android Keystore)를 활용하는 게 기본이고, Flutter에서는 `flutter_secure_storage`가 가장 흔한 선택입니다.

이 글은 아래 4가지를 모두 포함합니다.

1. 언제/왜 secure storage가 필요한지(트레이드오프)
2. 실전 예제 1개를 끝까지(저장/읽기/로그아웃/인터셉터 연결)
3. 흔한 실수/디버깅 포인트(재설치, 예외 케이스)
4. 대안 비교(SharedPreferences, Keychain 직접, 암호화 저장)

## 1) 언제/왜 쓰나 (트레이드오프)

### secure_storage가 필요한 상황

- access token / refresh token 같은 인증 정보 저장
- 사용자 식별자, 세션 키 등 민감한 값 저장

### 단점/주의점

- 완벽한 보안은 아니다(루팅/탈옥/디버깅 가능한 환경에서는 방어가 제한적)
- 기기/OS/제조사 정책에 따라 예외가 생길 수 있다
- "앱 삭제/재설치" 시 동작이 기대와 다를 수 있다(특히 iOS)

즉, "최소한의 기본기"로는 강력 추천이지만, 보안은 저장만이 아니라 만료/갱신/로그아웃 정책까지 같이 설계해야 합니다.

## 2) 설치

```yaml
dependencies:
  flutter_secure_storage: ^9.0.0
```

## 3) 실전 예제: TokenStore + Dio 인터셉터 연결

### (1) TokenStore 구현

```dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

class TokenStore {
  TokenStore({FlutterSecureStorage? storage})
      : _storage = storage ?? const FlutterSecureStorage();

  final FlutterSecureStorage _storage;

  static const _kAccess = "access_token";
  static const _kRefresh = "refresh_token";

  Future<void> save({required String access, String? refresh}) async {
    await _storage.write(key: _kAccess, value: access);
    if (refresh != null) {
      await _storage.write(key: _kRefresh, value: refresh);
    }
  }

  Future<String?> readAccessToken() => _storage.read(key: _kAccess);
  Future<String?> readRefreshToken() => _storage.read(key: _kRefresh);

  Future<void> clear() async {
    await _storage.delete(key: _kAccess);
    await _storage.delete(key: _kRefresh);
  }
}
```

### (2) Dio에서 자동으로 헤더 붙이기

```dart
import 'package:dio/dio.dart';

class AuthInterceptor extends Interceptor {
  AuthInterceptor(this.tokenStore);
  final TokenStore tokenStore;

  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) async {
    final token = await tokenStore.readAccessToken();
    if (token != null && token.isNotEmpty) {
      options.headers["Authorization"] = "Bearer $token";
    }
    handler.next(options);
  }
}
```

앱 초기화에서 주입:

```dart
final tokenStore = TokenStore();
final dio = Dio();
dio.interceptors.add(AuthInterceptor(tokenStore));
```

로그아웃:

```dart
await tokenStore.clear();
```

## 4) 흔한 실수/디버깅 포인트

### (1) iOS 재설치했는데 토큰이 남아있음

Keychain은 설정/정책에 따라 앱 삭제 후에도 남을 수 있습니다. "재설치했는데 자동 로그인됨" 같은 이슈가 나오면 Keychain 정책을 점검해야 해요.

### (2) Android에서 특정 기기만 예외

제조사/OS 버전에 따라 Keystore 동작이나 백업/복원 시나리오에서 예외가 생길 수 있습니다. 이런 경우에는

- 예외를 잡아서 재로그인 유도
- 토큰이 읽히지 않으면 안전하게 세션 초기화

같은 방어 로직이 필요합니다.

### (3) 토큰 저장만 해놓고 만료/갱신을 안 함

access token은 보통 만료됩니다. refresh token 플로우(갱신, 실패 시 로그아웃)를 함께 설계해야 사용자 경험이 안정됩니다.

## 5) 대안 비교

### SharedPreferences

민감 정보 저장에는 추천하지 않습니다. 간단한 플래그(첫 실행 여부 등) 같은 비민감 데이터에만 쓰는 편이 안전합니다.

### 플랫폼 키체인/키스토어 직접 사용

네이티브 코드를 직접 다루면 세밀한 제어가 가능하지만, 개발/유지 비용이 올라갑니다. 대부분은 `flutter_secure_storage`로 충분합니다.

### 암호화해서 파일 저장

암호화도 결국 키를 안전하게 보관해야 하므로, "키 저장" 문제로 돌아옵니다. 보통은 secure storage를 먼저 쓰고, 추가 보호가 필요할 때 계층적으로 접근합니다.

## 정리

- 토큰/민감 정보는 secure storage를 기본으로 고려
- 저장만이 아니라 만료/갱신/로그아웃 정책까지 함께 설계
- iOS 재설치/Android 기기별 예외 같은 케이스를 방어 로직으로 흡수

