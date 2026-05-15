---
layout: post
title: "Redis for 캐시"
date: 2026-05-15 02:00:00 +0900
category: Redis
permalink: /redis/cache
---

# Redis for 캐시

Redis를 가장 먼저 쓰는 이유는 보통 "응답 시간을 줄이기"입니다. WebFlux에서는 논블로킹 흐름을 유지하면서도 캐시를 잘 설계해야 병목이 안 생깁니다.

이 글은 다음을 목표로 합니다.

1. 캐시를 언제 쓰는지(트레이드오프)
2. WebFlux에서 Cache-Aside 패턴을 구현하는 기본 형태
3. 실무에서 터지는 문제(스탬피드/TTL/키 설계)와 방어

## 1) 언제/왜 쓰나 (트레이드오프)

### 쓰면 좋은 경우

- 자주 조회되지만 자주 바뀌지 않는 데이터(설정, 카테고리, 인기 목록 등)
- 계산 비용이 큰 결과(집계, 추천 결과 일부)
- 외부 API 호출 결과(레이트 제한이 있는 API)

### 단점/주의점

- 캐시는 "정합성"을 포기하고 "속도"를 얻는 경우가 많음
- TTL/갱신 정책을 안 잡으면 오래된 데이터가 계속 나갈 수 있음
- 캐시가 살아있을 때는 빠르지만, 만료 순간에 부하가 폭발할 수 있음(캐시 스탬피드)

## 2) 키 설계 기본

키는 "무엇을 캐시했는지"가 명확해야 디버깅이 쉽습니다.

- 예: `user:{id}`
- 예: `product:{id}`
- 예: `feed:home:{region}`

가능하면 값 자체는 JSON(또는 직렬화된 문자열)로 저장하고 TTL을 붙이는 편이 운영이 편합니다.

## 3) WebFlux 캐시(기본): Cache-Aside

전형적인 흐름:

1. Redis에서 먼저 찾기
2. 없으면 DB/API에서 조회
3. Redis에 저장(적절한 TTL)
4. 응답 반환

Spring WebFlux + Reactive RedisTemplate 예시:

```java
import java.time.Duration;
import org.springframework.data.redis.core.ReactiveStringRedisTemplate;
import reactor.core.publisher.Mono;

public class UserService {
  private final ReactiveStringRedisTemplate redis;
  private final UserRepository userRepository;

  public UserService(ReactiveStringRedisTemplate redis, UserRepository userRepository) {
    this.redis = redis;
    this.userRepository = userRepository;
  }

  public Mono<User> getUser(String id) {
    String key = "user:" + id;

    return redis.opsForValue().get(key)
        .flatMap(json -> Mono.just(User.fromJson(json)))
        .switchIfEmpty(
            userRepository.findById(id)
                .flatMap(user ->
                    redis.opsForValue()
                        .set(key, user.toJson(), Duration.ofMinutes(10))
                        .thenReturn(user)
                )
        );
  }
}
```

포인트:

- `switchIfEmpty(...)`로 캐시 미스 처리
- 저장은 `.thenReturn(user)`로 파이프라인에 자연스럽게 붙이기
- TTL은 데이터 성격에 맞게(너무 길면 stale, 너무 짧으면 미스 증가)

## 4) 흔한 문제와 방어

### (1) 캐시 스탬피드

만료 순간에 동일 키를 동시에 여러 요청이 미스 내면, DB/API가 동시에 때려맞습니다.

완화 방법:

- TTL에 약간의 랜덤 지터(jitter) 추가(동시에 만료되지 않게)
- 키 단위 락(가벼운 분산락)으로 1개 요청만 갱신하게 만들기
- stale-while-revalidate 패턴(일부는 오래된 값으로라도 응답, 백그라운드 갱신)

### (2) null 캐싱(negative cache)

없는 데이터(404)도 잠깐 캐시하면, 동일한 없는 요청이 계속 DB를 치는 걸 막을 수 있습니다.

### (3) TTL/무효화 정책

데이터가 수정될 때 캐시를 어떻게 처리할지(삭제 or 갱신)를 정해두지 않으면 운영에서 결국 꼬입니다.

## 5) 정리

- 캐시는 Cache-Aside로 시작하면 된다
- 키 설계 + TTL 정책이 절반이다
- 스탬피드/정합성 문제는 "언제 stale을 허용할지" 기준부터 잡는 게 빠르다

