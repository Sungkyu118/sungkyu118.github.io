---
layout: post
title: "Redis for 캐시: 언제 효과가 크고, 어디서 망가지는가"
date: 2026-05-15 02:00:00 +0900
category: Redis
permalink: /redis/cache
description: "Redis 캐시가 언제 큰 효과를 내고, 캐시 무효화와 스탬피드 때문에 어디서 깨지는지 실무적으로 설명합니다."
image:
  path: "/assets/img/og/redis-cache-cover.svg"
  alt: "Redis 캐시 포스트 대표 이미지"
---

# Redis for 캐시: 언제 효과가 크고, 어디서 망가지는가

> Redis 캐시가 언제 큰 효과를 내고, 캐시 무효화와 스탬피드 때문에 어디서 깨지는지 실무적으로 설명합니다.
>
> 이전 글: [Redis 자료구조 3: Sorted Set(ZSET), 랭킹의 정석](/redis/ds-zset)
> 다음 글: [Redis for 세션: 여러 서버에서 로그인 상태를 공유하는 법](/redis/session)
> 함께 보면 좋은 글:
> - [Redis 입문 실무형 1: Redis는 언제 쓰고, 언제 쓰면 안 될까](/redis/practical-what-and-when)
> - [Redis for 세션: 여러 서버에서 로그인 상태를 공유하는 법](/redis/session)

Redis를 가장 먼저 붙이는 이유는 보통 캐시입니다. 응답 시간을 줄이고, DB 부하를 낮추고, 외부 API 호출 비용을 줄이는 데 꽤 직접적인 효과가 있기 때문입니다. 하지만 캐시는 생각보다 "넣는 것"보다 "유지하는 것"이 더 어렵습니다.

이 글에서는 Redis 캐시를 언제 쓰면 효과가 큰지, 어떤 설계가 기본인지, 그리고 운영에서 자주 무너지는 포인트가 무엇인지 정리해보겠습니다.

## 캐시가 특히 잘 맞는 데이터

캐시의 핵심은 같은 값을 여러 번 재사용하는 데 있습니다. 그래서 아래 같은 데이터에 잘 맞습니다.

- 카테고리, 설정값, 공통 메타데이터
- 상품 상세처럼 조회가 많고 변경은 상대적으로 적은 데이터
- 계산 비용이 큰 통계/집계 결과
- 외부 API 응답 중 자주 반복되는 결과

반대로 캐시가 잘 안 맞는 경우도 있습니다.

- 거의 매번 바뀌는 데이터
- 사용자마다 값이 달라 재사용성이 낮은 데이터
- stale 허용이 매우 어려운 핵심 원장 데이터

도입 판단 기준은 [Redis는 언제 쓰고, 언제 쓰면 안 될까](/redis/practical-what-and-when)에서 먼저 잡고 오는 편이 좋습니다.

## 가장 무난한 시작점: Cache-Aside

실무에서 가장 흔한 패턴은 Cache-Aside입니다.

흐름은 단순합니다.

1. 먼저 Redis에서 읽는다.
2. 없으면 DB나 외부 API에서 조회한다.
3. 조회한 값을 Redis에 TTL과 함께 저장한다.
4. 응답한다.

이 방식이 많이 쓰이는 이유는 책임이 비교적 명확하기 때문입니다. 애플리케이션이 캐시를 직접 제어하므로 디버깅이 쉽고, 기존 DB 중심 구조에도 붙이기 수월합니다.

## 코드 예시: Cache-Aside

```java
import java.time.Duration;
import org.springframework.data.redis.core.ReactiveStringRedisTemplate;
import reactor.core.publisher.Mono;

public class ProductService {
  private final ReactiveStringRedisTemplate redis;
  private final ProductRepository productRepository;

  public ProductService(ReactiveStringRedisTemplate redis, ProductRepository productRepository) {
    this.redis = redis;
    this.productRepository = productRepository;
  }

  public Mono<Product> getProduct(String id) {
    String key = "product:" + id;

    return redis.opsForValue().get(key)
        .flatMap(json -> Mono.just(Product.fromJson(json)))
        .switchIfEmpty(
            productRepository.findById(id)
                .flatMap(product ->
                    redis.opsForValue()
                        .set(key, product.toJson(), Duration.ofMinutes(10))
                        .thenReturn(product)
                )
        );
  }
}
```

여기서 중요한 건 코드가 아니라 운영 감각입니다.

- 키 이름이 읽히는가?
- TTL이 데이터 성격에 맞는가?
- miss가 몰릴 때 DB가 버틸 수 있는가?

이 세 가지를 놓치면 캐시는 금방 문제를 만들기 시작합니다.

## 캐시에서 가장 흔한 사고 1: stale 데이터

캐시는 원본이 아니라 복제본이기 때문에, 원본 데이터가 바뀐 뒤에도 예전 값이 잠시 남을 수 있습니다. 이걸 허용할 수 있는지 먼저 정해야 합니다.

예:

- 카테고리 설명: 몇 분 stale 가능
- 상품 가격: 훨씬 더 민감
- 권한 정보: stale 허용이 위험할 수 있음

그래서 TTL은 성능 숫자가 아니라, **얼마나 오래된 값을 보여줘도 되는가**의 기준으로 정해야 합니다. 이 부분은 [키 설계와 TTL](/redis/practical-key-ttl)에서 더 자세히 다룹니다.

## 캐시에서 가장 흔한 사고 2: 캐시 스탬피드

여러 요청이 동시에 같은 키를 읽는데, 그 순간 캐시가 만료되어 모두 miss가 나면 어떻게 될까요? 다 같이 DB나 외부 API로 몰립니다. 이것이 캐시 스탬피드입니다.

대표 증상:

- 평소엔 빠르다가 특정 시간대에 갑자기 느려짐
- DB가 순간적으로 급증
- 캐시가 있는데도 장애처럼 보임

완화 방법:

- TTL에 jitter를 넣어 만료 시점 분산
- 비싼 키는 백그라운드 재생성 고려
- hot key는 별도 갱신 전략 검토

## 캐시에서 가장 흔한 사고 3: null 캐싱을 안 해서 miss가 계속 난다

존재하지 않는 데이터에 대한 요청이 반복되면, 매번 DB까지 다녀오는 구조가 됩니다. 이럴 때는 "없다"는 결과도 짧게 캐싱하는 negative cache가 도움이 됩니다.

예:

- 존재하지 않는 상품 ID
- 삭제된 사용자 프로필
- 조건에 맞지 않는 조회 결과

물론 TTL은 짧게 두는 편이 안전합니다. 너무 길면 실제로 새 데이터가 생겼을 때 반영이 늦어질 수 있기 때문입니다.

## 무효화 전략을 정하지 않으면 결국 사고 난다

캐시는 채우는 것보다 지우는 규칙이 중요합니다. 보통 세 가지 방식이 있습니다.

- TTL 자연 만료
- 데이터 변경 시 명시적 삭제
- 변경 후 새 값으로 즉시 덮어쓰기

어떤 방식을 쓸지는 데이터 성격에 따라 달라집니다. 중요한 건 "어떻게든 되겠지" 상태로 두지 않는 것입니다. 쓰기 경로에서 캐시 처리 책임이 빠지면 운영 중에 아주 애매한 버그가 생깁니다.

## 실무에서 자주 하는 실수

### 1) 캐시 적중률보다 응답 시간만 본다

응답은 빨라졌는데 hit rate가 낮으면, 사실상 Redis가 큰 역할을 못 하고 있을 수 있습니다.

### 2) 모든 데이터에 캐시를 붙인다

캐시가 만능처럼 보일 때 많이 생기는 실수입니다. 재사용성이 낮거나 stale에 민감한 데이터는 효과보다 비용이 더 클 수 있습니다.

### 3) 메모리와 eviction을 안 본다

캐시 키가 계속 늘어나면 메모리 문제가 시작됩니다. 그리고 메모리가 차기 시작하면 결국 eviction과 hit rate 흔들림으로 이어집니다. 이건 [운영 기본](/redis/practical-ops-basics)과 [메모리/eviction/hot key](/redis/memory-eviction-hotkeys) 쪽과 연결됩니다.

<!-- codex-category-inline-links:start -->

지금 읽고 계신 주제가 아직 조금 추상적으로 느껴지신다면 [Redis 입문 실무형 1: Redis는 언제 쓰고, 언제 쓰면 안 될까](/redis/practical-what-and-when), [Redis 입문 실무형 2: 키 설계와 TTL, 운영에서 덜 망하는 법](/redis/practical-key-ttl), [Redis 운영 심화: 메모리, eviction, 핫키 사고 패턴](/redis/memory-eviction-hotkeys) 글도 함께 읽어보시면 좋겠습니다. 같은 Redis 흐름 안에서 앞단의 배경과 다음 단계의 확장 포인트를 같이 보실 수 있어서, 지금 배우는 내용이 실제 프로젝트에서 어디에 연결되는지 훨씬 더 선명하게 이해하실 수 있습니다.

<!-- codex-category-inline-links:end -->
## 정리

Redis 캐시는 제대로만 설계하면 아주 강력합니다. 하지만 "빨라진다"는 장점만 보고 들어가면 stale 데이터, 캐시 스탬피드, 무효화 누락, 메모리 증가 같은 문제를 곧 만나게 됩니다.

그래서 캐시는 항상 세 가지를 같이 봐야 합니다.

- 이 데이터는 캐시에 맞는가?
- TTL과 무효화 전략이 있는가?
- miss가 몰릴 때 원본 시스템을 보호할 수 있는가?

이 기준이 잡혀 있으면 Redis 캐시는 성능 최적화가 아니라 안정적인 운영 도구가 됩니다.

<!-- codex-category-links:start -->

## 이어서 읽어보시면 좋습니다

- [Redis 입문 실무형 1: Redis는 언제 쓰고, 언제 쓰면 안 될까](/redis/practical-what-and-when)
- [Redis 입문 실무형 2: 키 설계와 TTL, 운영에서 덜 망하는 법](/redis/practical-key-ttl)
- [Redis 운영 심화: 메모리, eviction, 핫키 사고 패턴](/redis/memory-eviction-hotkeys)

지금 글과 바로 이어서 읽기 좋은 흐름으로 묶어두었으니, 개념을 비교해보시거나 다음 실습으로 넘어가실 때 차근차근 따라가보시면 좋겠습니다.

<!-- codex-category-links:end -->
