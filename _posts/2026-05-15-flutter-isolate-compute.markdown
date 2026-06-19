---
layout: post
title: "Flutter isolate와 compute: 무거운 작업으로 UI가 멈출 때"
date: 2026-05-15 00:20:00 +0900
category: Flutter
permalink: /flutter/isolate-compute
---

# Flutter isolate와 compute: 무거운 작업으로 UI가 멈출 때

Flutter 앱은 기본적으로 UI를 그리는 main isolate에서 Dart 코드를 실행합니다. 보통의 버튼 클릭, 간단한 상태 변경, 짧은 API 응답 처리 정도는 문제가 없습니다. 하지만 큰 JSON을 파싱하거나, 이미지 처리, 암호화, 긴 반복문 같은 CPU 작업을 main isolate에서 실행하면 화면이 버벅이거나 멈춘 것처럼 보일 수 있습니다.

`async`와 `await`를 쓰면 모든 문제가 해결된다고 생각하기 쉽지만, 이것은 절반만 맞습니다. `await`는 비동기 대기를 깔끔하게 표현해주지만, CPU를 오래 붙잡는 계산 자체를 다른 스레드처럼 자동으로 옮겨주지는 않습니다. 이럴 때 `isolate` 또는 Flutter의 편의 함수인 `compute`를 사용할 수 있습니다.

## UI가 멈추는 간단한 예

아래 코드는 버튼을 누르면 큰 숫자 범위를 순회합니다.

```dart
int heavySum(int max) {
  var sum = 0;
  for (var i = 0; i < max; i++) {
    sum += i;
  }
  return sum;
}

ElevatedButton(
  onPressed: () {
    final result = heavySum(100000000);
    print(result);
  },
  child: const Text('계산 시작'),
)
```

이 코드는 실행되는 동안 main isolate를 계속 사용합니다. 그 사이 Flutter는 프레임을 그릴 기회를 얻지 못합니다. 결과적으로 로딩 인디케이터도 돌지 않고, 버튼도 눌리지 않고, 앱이 잠깐 멈춘 것처럼 보일 수 있습니다.

## Future만 감싸면 충분할까?

다음처럼 `Future`로 감싸면 괜찮아질 것처럼 보입니다.

```dart
Future<int> heavySumAsync(int max) async {
  return heavySum(max);
}
```

하지만 내부에서 실행되는 계산이 같은 isolate에서 돌아간다면 UI 멈춤 문제는 여전히 남을 수 있습니다. 네트워크 요청이나 파일 읽기처럼 실제로 기다리는 시간이 많은 작업은 `await`가 적합하지만, CPU를 계속 사용하는 순수 계산은 isolate로 분리하는 것이 더 효과적입니다.

## compute 사용하기

Flutter는 간단한 isolate 작업을 위해 `compute` 함수를 제공합니다.

```dart
import 'package:flutter/foundation.dart';

int heavySum(int max) {
  var sum = 0;
  for (var i = 0; i < max; i++) {
    sum += i;
  }
  return sum;
}

Future<void> runCalculation() async {
  final result = await compute(heavySum, 100000000);
  print(result);
}
```

`compute`는 첫 번째 인자로 실행할 함수, 두 번째 인자로 그 함수에 전달할 값을 받습니다. 함수는 top-level 함수이거나 static 함수여야 합니다. 위젯 클래스 안의 인스턴스 메서드를 그대로 넘기면 실패할 수 있습니다.

```dart
// 권장: top-level 함수
int parseCount(String text) {
  return int.parse(text);
}
```

## JSON 파싱 예시

실무에서 자주 쓰는 예는 큰 JSON 문자열 파싱입니다.

```dart
import 'dart:convert';
import 'package:flutter/foundation.dart';

class Product {
  const Product({
    required this.id,
    required this.name,
  });

  final int id;
  final String name;

  factory Product.fromJson(Map<String, dynamic> json) {
    return Product(
      id: json['id'] as int,
      name: json['name'] as String,
    );
  }
}

List<Product> parseProducts(String responseBody) {
  final decoded = jsonDecode(responseBody) as List<dynamic>;
  return decoded
      .map((item) => Product.fromJson(item as Map<String, dynamic>))
      .toList();
}

Future<List<Product>> parseProductsInBackground(String body) {
  return compute(parseProducts, body);
}
```

서버에서 수천 개 이상의 항목이 내려오거나, JSON 구조가 크고 복잡하다면 파싱만으로도 프레임 드랍이 생길 수 있습니다. 이럴 때 compute로 분리하면 UI 반응성을 지키는 데 도움이 됩니다.

## compute에 전달할 수 있는 값

`compute`에는 isolate 사이에서 전달 가능한 값만 넘기는 것이 안전합니다. 문자열, 숫자, bool, List, Map처럼 단순 데이터는 괜찮습니다. 하지만 `BuildContext`, `TextEditingController`, `Dio`, `Database` 연결 객체 같은 것은 넘기면 안 됩니다.

```dart
// 잘못된 방향: UI 객체를 isolate로 넘기지 않습니다.
compute(doSomething, context);
```

isolate는 메모리를 공유하지 않습니다. 서로 메시지를 주고받는 방식으로 동작합니다. 그래서 복잡한 객체나 플랫폼 채널에 얽힌 객체를 넘기려고 하면 에러가 나거나 예상대로 동작하지 않을 수 있습니다.

## 로딩 상태와 함께 사용하기

계산이 오래 걸릴 수 있으므로 UI에서는 로딩 상태를 보여주는 것이 좋습니다.

```dart
class ComputeDemoPage extends StatefulWidget {
  const ComputeDemoPage({super.key});

  @override
  State<ComputeDemoPage> createState() => _ComputeDemoPageState();
}

class _ComputeDemoPageState extends State<ComputeDemoPage> {
  bool isLoading = false;
  int? result;

  Future<void> _calculate() async {
    setState(() => isLoading = true);

    try {
      final value = await compute(heavySum, 100000000);
      if (!mounted) return;
      setState(() => result = value);
    } finally {
      if (mounted) {
        setState(() => isLoading = false);
      }
    }
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      mainAxisAlignment: MainAxisAlignment.center,
      children: [
        if (isLoading) const CircularProgressIndicator(),
        if (result != null) Text('결과: $result'),
        ElevatedButton(
          onPressed: isLoading ? null : _calculate,
          child: const Text('계산하기'),
        ),
      ],
    );
  }
}
```

여기서도 비동기 작업 후 `mounted`를 확인합니다. 사용자가 계산 중 화면을 떠났는데 `setState`를 호출하면 에러가 날 수 있기 때문입니다. 이 패턴은 [VoidCallback 글](/flutter/callback)에서 본 비동기 버튼 처리와도 연결됩니다.

## isolate를 남용하면 안 되는 이유

isolate는 공짜가 아닙니다. 새 isolate를 만들고 데이터를 주고받는 비용이 있습니다. 아주 작은 계산을 매번 compute로 보내면 오히려 느려질 수 있습니다. 기준은 "UI 프레임에 영향을 줄 만큼 CPU 작업이 큰가"입니다. 단순한 문자열 가공, 짧은 리스트 필터링, 작은 JSON 파싱은 main isolate에서 처리해도 괜찮은 경우가 많습니다.

또 하나의 주의점은 디버깅입니다. isolate 경계가 생기면 에러 추적이 조금 더 복잡해질 수 있습니다. 그래서 처음부터 모든 로직을 isolate로 보내기보다, 실제로 프레임 드랍이 있는지 확인하고 필요한 부분만 분리하는 것이 좋습니다. Flutter DevTools의 Performance 탭을 함께 보면 판단이 쉬워집니다.

## 정리

`async`와 `await`는 비동기 흐름을 표현하는 문법이고, CPU를 오래 사용하는 작업을 자동으로 다른 isolate로 보내주지는 않습니다. 큰 JSON 파싱, 이미지 처리, 무거운 계산처럼 UI를 멈출 수 있는 작업은 `compute`로 분리하는 것을 고려해야 합니다. 다만 isolate 생성과 데이터 전달 비용이 있으므로 작은 작업까지 무조건 분리하는 것은 좋지 않습니다. 성능 문제는 감으로만 판단하지 말고, 실제 버벅임과 측정 결과를 보고 필요한 부분에 적용하는 것이 가장 안전합니다.
