---
layout: post
title: "Redis 분산락: SET NX PX로 시작하는 실전 가이드"
date: 2026-05-15 03:10:00 +0900
category: Redis
permalink: /redis/distributed-lock
description: "Redis 분산락의 기본 개념과 SET NX PX, 만료 시간, 중복 실행 방지 주의점을 실무 기준으로 설명합니다."
image:
  path: "/assets/img/og/redis-lock-cover.svg"
  alt: "Redis 분산락 포스트 대표 이미지"
---

# Redis 분산락: SET NX PX로 시작하는 실전 가이드

> Redis 분산락의 기본 개념과 SET NX PX, 만료 시간, 중복 실행 방지 주의점을 실무 기준으로 설명합니다.
>
> 이전 글: [Redis for 랭킹: Sorted Set으로 실시간 순위 만들기](/redis/ranking)
> 다음 글: [Redis 운영 심화: 메모리, eviction, 핫키 사고 패턴](/redis/memory-eviction-hotkeys)
> 함께 보면 좋은 글:
> - [Redis 입문 실무형 1: Redis는 언제 쓰고, 언제 쓰면 안 될까](/redis/practical-what-and-when)
> - [Redis 레이트 리밋: Lua로 원자성 보장하기](/redis/rate-limit-lua)

분산락은 여러 서버가 동시에 같은 작업을 처리하지 못하게 막는 장치입니다. 서버가 한 대뿐이라면 JVM 안의 `synchronized`나 `ReentrantLock`으로도 어느 정도 해결할 수 있습니다. 하지만 서버가 여러 대가 되는 순간, 프로세스 내부 락은 서로를 알 수 없습니다.

예를 들어 쿠폰을 100장만 발급해야 하는데 서버가 3대라면, 각 서버가 동시에 "아직 재고가 있네"라고 판단할 수 있습니다. 이때 Redis 같은 공용 저장소를 이용해 "지금 이 작업은 누가 처리 중인지"를 관리하는 방식이 분산락입니다.

## 가장 기본: SET NX PX

Redis에서 분산락의 출발점은 `SET key value NX PX milliseconds`입니다.

```bash
SET lock:coupon:1001 request-uuid NX PX 3000
```

의미는 이렇습니다.

- `NX`: key가 없을 때만 설정
- `PX 3000`: 3초 뒤 자동 만료
- `request-uuid`: 락을 잡은 주체를 구분하기 위한 값

락 획득에 성공하면 Redis는 `OK`를 반환합니다. 이미 누군가 락을 잡고 있으면 실패합니다.

## 왜 만료 시간이 꼭 필요할까

락에 TTL이 없으면 위험합니다. 락을 잡은 서버가 작업 도중 죽거나 네트워크 문제로 해제하지 못하면, 락이 영원히 남을 수 있기 때문입니다. 그러면 해당 리소스는 계속 처리 불가능한 상태가 됩니다.

그래서 분산락에는 반드시 만료 시간이 필요합니다.

하지만 TTL을 너무 짧게 잡아도 문제입니다. 작업이 아직 끝나지 않았는데 락이 먼저 풀리면 다른 서버가 같은 작업을 시작할 수 있습니다. 결국 중복 실행을 막으려고 락을 썼는데 중복 실행이 다시 생깁니다.

TTL은 "대충 3초"가 아니라, 실제 작업 시간의 상한과 장애 시 복구 시간을 고려해서 잡아야 합니다.

## 락 해제는 반드시 value를 확인하고 해야 한다

락을 해제할 때 단순히 `DEL lock:coupon:1001`을 하면 위험합니다.

상황을 생각해보면:

1. A 서버가 락을 잡는다.
2. A 서버 작업이 예상보다 오래 걸려 락 TTL이 만료된다.
3. B 서버가 새로 락을 잡는다.
4. A 서버가 뒤늦게 작업을 끝내고 `DEL`을 실행한다.
5. B 서버의 락이 지워진다.

이 문제를 막으려면 락을 잡을 때 넣은 value를 확인하고, 내가 잡은 락일 때만 지워야 합니다. 보통 Lua 스크립트를 사용합니다.

```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
  return redis.call("DEL", KEYS[1])
end
return 0
```

## Java 코드 형태

개념적으로는 아래 흐름입니다.

```java
String key = "lock:coupon:" + couponId;
String token = UUID.randomUUID().toString();
Boolean locked = redis.opsForValue().setIfAbsent(key, token, Duration.ofSeconds(3));

if (!Boolean.TRUE.equals(locked)) {
  throw new IllegalStateException("이미 처리 중인 요청입니다.");
}

try {
  issueCoupon(couponId, userId);
} finally {
  releaseLockOnlyWhenTokenMatches(key, token);
}
```

핵심은 `finally`에서 락을 해제하되, 반드시 token을 비교해야 한다는 점입니다.

## 분산락이 잘 맞는 상황

분산락은 이런 상황에서 도움이 됩니다.

- 중복 쿠폰 발급 방지
- 동일 주문에 대한 중복 결제 처리 방지
- 특정 배치 작업이 여러 서버에서 동시에 실행되는 것 방지
- 같은 리소스의 상태 변경을 짧은 시간 동안 직렬화

공통점은 "동시에 처리하면 안 되는 짧고 명확한 작업"입니다.

## 분산락이 애매한 상황

분산락이 항상 좋은 해결책은 아닙니다. 긴 작업, 재시도 많은 작업, 정확한 순서 보장이 필요한 작업에는 다른 설계가 더 맞을 수 있습니다.

예를 들어 몇 분 이상 걸리는 작업을 Redis 락 하나로 묶어두면 TTL 연장, 작업 중단, 장애 복구 문제가 복잡해집니다. 이런 경우에는 큐나 상태 머신, DB 트랜잭션, 유니크 제약조건이 더 단순할 수 있습니다.

작업을 순서대로 뒤에서 처리하면 되는 문제라면 [Redis for 큐](/redis/queue)나 [Redis Streams 실전](/redis/streams-queue) 쪽이 더 자연스러운 해법일 수 있습니다. 분산락은 "동시에 처리하면 안 되는 짧은 구간"을 보호할 때 가장 깔끔합니다.

## 자주 생기는 장애 패턴

### 1) TTL이 작업 시간보다 짧다

락이 먼저 풀려서 중복 실행이 발생합니다. 특히 외부 API 호출이 들어간 작업은 평소보다 오래 걸릴 가능성을 고려해야 합니다.

### 2) 락 해제를 단순 DEL로 한다

다른 요청이 새로 잡은 락을 지울 수 있습니다. 이건 운영에서 매우 찾기 어려운 버그가 됩니다.

### 3) 락 획득 실패를 정상 흐름으로 설계하지 않는다

락 획득에 실패했을 때 무조건 에러를 던질지, 잠깐 재시도할지, 사용자에게 "처리 중"이라고 보여줄지 정해야 합니다.

### 4) 락으로 DB 정합성 문제까지 해결하려 한다

분산락은 보조 장치입니다. 재고 차감, 결제 상태 변경처럼 중요한 데이터는 DB의 트랜잭션과 유니크 제약조건도 함께 봐야 합니다.

<!-- codex-category-inline-links:start -->

遺꾩궛?쎌? 臾몃쾿留??몄슦硫?諛붾줈 ?ㅻТ???곸슜?????덉쓣 寃껋쿂??蹂댁씠吏留? ?ㅼ젣濡쒕뒗 留뚮즺 ?쒓컙怨??먯옄?? ?μ븷 ?곹솴??媛숈씠 ?댄빐?섏뀛?????꾪뿕?⑸땲?? 洹몃옒??[Redis ?덉씠??由щ컠: Lua濡??먯옄??蹂댁옣?섍린](/redis/rate-limit-lua), [Redis for ?몄뀡: ?щ윭 ?쒕쾭?먯꽌 濡쒓렇???곹깭瑜?怨듭쑀?섎뒗 踰?(/redis/session), [Redis ?낅Ц ?ㅻТ??2: ???ㅺ퀎? TTL, ?댁쁺?먯꽌 ??留앺븯??踰?(/redis/practical-key-ttl) 湲???④퍡 ?쎌뼱蹂댁떆硫?醫뗪쿋?듬땲?? 媛숈? Redis 湲곕뒫???곕뜑?쇰룄 "怨듭쑀 ?곹깭瑜??ㅻ（??踰?, "?먯옄?곸쑝濡?臾띕뒗 踰?, "留뚮즺瑜??대뼸寃??ㅺ퀎?댁빞 ?섎뒗吏"瑜?媛숈씠 蹂댁뀛??遺꾩궛??湲???꾪뿕 ?ъ씤?멸? ???먮졆?섍쾶 ?ㅼ뼱?듬땲??

<!-- codex-category-inline-links:end -->
## 정리

Redis 분산락은 `SET NX PX`로 시작할 수 있지만, 진짜 중요한 부분은 TTL, token 비교 해제, 실패 처리입니다.

짧고 명확한 동시성 제어에는 유용하지만, 모든 정합성 문제를 분산락으로 해결하려고 하면 구조가 더 위험해질 수 있습니다. 락은 마지막 답이 아니라, DB 제약과 작업 설계 사이에서 조심스럽게 쓰는 도구로 보는 편이 좋습니다.

<!-- codex-category-links:start -->

## 이어서 읽어보시면 좋습니다

- [Redis 레이트 리밋: Lua로 원자성 보장하기](/redis/rate-limit-lua)
- [Redis for 세션: 여러 서버에서 로그인 상태를 공유하는 법](/redis/session)
- [Redis 입문 실무형 2: 키 설계와 TTL, 운영에서 덜 망하는 법](/redis/practical-key-ttl)

지금 글과 바로 이어서 읽기 좋은 흐름으로 묶어두었으니, 개념을 비교해보시거나 다음 실습으로 넘어가실 때 차근차근 따라가보시면 좋겠습니다.

<!-- codex-category-links:end -->
