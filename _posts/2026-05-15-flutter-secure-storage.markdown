---
layout: post
title: "flutter_secure_storage로 토큰 안전하게 저장하기 (실전 주의점)"
date: 2026-05-15 01:10:00 +0900
category: Flutter
permalink: /flutter/flutter-secure-storage
---

# flutter_secure_storage로 토큰 안전하게 저장하기 (실전 주의점)

로그인 토큰/세션 정보는 `SharedPreferences`에 그대로 저장하면 안 됩니다. 민감 정보는 플랫폼 보안 저장소(iOS Keychain / Android Keystore)를 활용하는 게 기본이고, Flutter에서는 `flutter_secure_storage`가 가장 흔한 선택입니다.

이 글은 아래 4가지를 전부 담습니다.

1. 언제/왜 secure storage가 필요한지(트레이드오프)
2. 실전 예제 1개를 끝까지(저장/읽기/로그아웃/인터셉터 연결)
3. 흔한 실수/디버깅 포인트(재설치/기기별 예외/만료 처리)
4. 대안 비교(SharedPreferences, 플랫폼 직접, 암호화 저장)

## 1) 언제/왜 쓰나 (트레이드오프)

### secure storage가 필요한 상황

- access token / refresh token 저장
- 자동 로그인 세션 키, 민감한 사용자 식별자 저장

### 단점/주의점

- 완벽한 보안은 아닙니다(루팅/탈옥, 디버깅 가능한 환경에서는 방어가 제한적)
- 기기/OS/제조사 정책에 따라 예외가 생길 수 있습니다
- "앱 삭제/재설치"에서 기대와 다른 동작이 나올 수 있습니다(특히 iOS)

결론: 일반적인 모바일 앱의 "토큰 저장 기본기"로는 강력 추천이지만, 보안은 저장만이 아니라 만료/갱신/로그아웃 정책까지 함께 설계해야 합니다.

## 2) 설치

```yaml
dependencies:
  flutter_secure_storage: ^9.0.0
```

## 3) 실전 예제: TokenStore + Dio Interceptor 연결

### (1) TokenStore 구현 (테스트/교체 가능하게 주입)

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

### (2) Dio 요청마다 Authorization 자동 첨부

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

### (1) iOS: 재설치했는데 토큰이 남아있는 것처럼 보임

iOS Keychain은 설정/정책에 따라 앱 삭제 후에도 남는 것처럼 느껴지는 케이스가 있습니다. "재설치했는데 자동 로그인"이 의심되면,

- 로그인 시점에 토큰이 정말로 남아있는지 읽어서 확인(로그)
- 토큰이 읽히면 서버에서도 세션이 살아있는지 확인
- 필요하다면 첫 실행/재설치 판단 후 clear 정책을 명확히 적용

같은 방식으로 원인을 분리하는 게 좋습니다.

### (2) Android: 특정 기기/OS에서 예외가 남

Keystore는 기기 환경(제조사, 백업/복원, 보안 설정 등)과 엮여 예외가 날 수 있습니다. 중요한 건 "예외 발생 시 안전한 fallback"입니다.

- 읽기 실패/예외면 세션을 안전하게 초기화하고 로그인 화면으로 보내기
- 토큰이 없다고 가정하고 다시 로그인 유도(UX 메시지 준비)

### (3) 토큰 저장만 하고 만료/갱신 설계를 안 함

access token은 보통 만료됩니다. 실무에서는 아래 2가지를 같이 설계합니다.

- refresh token으로 재발급(성공 시 저장 갱신)
- refresh 실패 시 강제 로그아웃(clear + 로그인 이동)

여기까지 들어가야 "자동 로그인"이 안정적으로 돌아가요.

### (4) 실제로 도움이 되는 로그 예시

이런 로그를 한 번만 찍어둬도 문제 원인 찾기가 빨라집니다.

- 앱 시작 시: access/refresh 존재 여부(값 자체는 찍지 말고 존재만)
- 401 발생 시: 재시도/갱신 시도 여부
- 갱신 성공/실패 시: clear 여부

## 5) 대안 비교

### SharedPreferences

민감 정보 저장에는 추천하지 않습니다. "첫 실행 여부" 같은 비민감 플래그에만 쓰는 편이 안전합니다.

### 플랫폼 Keychain/Keystore 직접 사용

세밀한 제어가 가능하지만, 개발/유지 비용이 늘어납니다. 대부분은 `flutter_secure_storage`로 충분합니다.

### 암호화해서 파일 저장

암호화도 결국 키를 안전하게 보관해야 해서, "키 저장" 문제로 돌아옵니다. 보통은 secure storage를 먼저 사용하고, 추가 보호가 필요할 때 계층적으로 접근합니다.

## 6) 테스트 관점 (유용한 포인트)

UI/비즈니스 로직 테스트에서 실제 secure storage를 쓰면 느리고 환경 의존적입니다. 위 예제처럼 `TokenStore({FlutterSecureStorage? storage})`로 주입 가능하게 해두면,

- 테스트에서는 메모리 기반 fake storage로 교체
- 토큰 저장/삭제/헤더 첨부 로직을 빠르게 검증

같은 구조로 테스트가 쉬워집니다.

## 7) 적용 체크리스트

- 토큰은 `SharedPreferences`가 아니라 secure storage에 저장
- 앱 시작 시 토큰 존재 여부만 로그로 확인
- 401 처리(갱신/실패 시 로그아웃) 플로우를 명확히
- 읽기/쓰기 실패 시 fallback(세션 초기화 + 재로그인 유도)
- 로그/크래시 리포트에 토큰 값 자체는 절대 남기지 않기

