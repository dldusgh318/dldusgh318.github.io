---
title: "Spring Cache와 Redis 실전: 어노테이션부터 운영 주의사항까지"
date: 2026-10-23 18:42:00 +0900
categories: [Talks, 동아리 백엔드 세션]
tags: [cache, redis, spring, backend]
series: 캐시
media_subpath: /assets/img/posts/cache
description: Spring Cache 어노테이션으로 캐시를 구현하는 법, Self-invocation 함정, Redis를 캐시로 쓰는 이유와 Caffeine·Memcached 비교, KEYS 금지·maxmemory·단편화·영속성 같은 운영 주의사항, 그리고 배포 전략까지 정리했다.
---

> 신촌연합 IT동아리 백엔드 파트 세션 발표, 캐시 시리즈 2편입니다. [1편: 캐시 전략]({% post_url 2026-10-23-캐시-전략 %})에서 배운 전략을 이번엔 코드로 옮겨 봅니다.
{: .prompt-info }

## Spring Cache.. 어노테이션이 존재한다?

캐시 전략을 배웠으면 이제 실제로 구현해 봐야겠죠? 아마 코드로 짜면 이렇게 될 거예요.

```java
public Post getPost(Long id) {
    String key = "posts:" + id;
    Post cached = redis.get(key);               // 캐시 확인
    if (cached != null) return cached;          // 히트

    Post post = postRepository.findById(id);    // 미스 → DB
    redis.set(key, post, TTL_10MIN);            // 캐시 저장
    return post;
}
```

근데 이 로직을 메서드마다 반복하기엔 너무 귀찮고, 실수할 수도 있겠죠?

→ Spring은 이 반복을 없애 주는 **어노테이션 기반 캐시 추상화**를 제공합니다. 오늘은 이걸 배워 봅시다!

### 기본 세팅

```groovy
// build.gradle
implementation 'org.springframework.boot:spring-boot-starter-cache'
```

```yaml
# application.yml
spring:
  data:
    redis:
      host: localhost
      port: 6379
```

```java
@SpringBootApplication
@EnableCaching
public class Application { ... }
```

<details markdown="1">
<summary>오잉? 그럼 Spring Cache는 어떤 캐시를 쓰는 건가요?</summary>

Spring은 캐시 구현체에 직접 의존하지 않도록 추상화돼 있어요. 그래서 `CacheManager`가 어떤 캐시를 쓸지 정하는데, Config가 없어도 Spring Boot가 클래스패스에 있는 라이브러리를 보고 **CacheManager를 자동으로 등록**해 줍니다.

ex) `spring-boot-starter-data-redis`가 있으면 `RedisCacheManager`, 아무것도 없으면 `ConcurrentMapCacheManager`(JVM 메모리)

즉 Config 없이도 돌아가긴 하는데, **TTL도 직렬화 설정도 없는 기본값**으로 돌아가요. 그러니 각자 Config 설정을 꼭 해 두는 걸 추천드립니다!! ([Spring Cache와 CacheManager 이해하기](https://mangkyu.tistory.com/370))

</details>

이제 핵심 어노테이션들을 알아봅시다.

### `@Cacheable`: 조회 캐싱

메서드 결과를 캐시에 저장하고, 다음 호출부터는 메서드를 실행하지 않고 캐시에서 바로 반환해요.

```java
@Cacheable(value = "posts", key = "#postId")
public PostDto getPost(Long postId) {
    // 캐시에 없을 때만 실행됨
    return postRepository.findById(postId)
        .map(PostDto::from)
        .orElseThrow();
}
```

1. `getPost(1L)` 호출
2. Spring이 `posts::1` 키로 Redis 조회
3. **있으면** → 메서드 실행 없이 바로 반환 (Cache Hit)
4. **없으면** → 메서드 실행 → 결과를 `posts::1`로 저장 → 반환 (Cache Miss)

✅ **`unless` 옵션**: 결과가 null이면 캐싱하지 않아요.

```java
@Cacheable(value = "posts", key = "#postId", unless = "#result == null")
public PostDto getPost(Long postId) { ... }
```

### `@CacheEvict`: 캐시 삭제

DB 데이터가 바뀔 때 캐시를 지워요. 다음 조회 때 DB에서 최신 데이터를 다시 가져오죠.

```java
// 삭제
@CacheEvict(value = "posts", key = "#postId")
public void deletePost(Long postId) {
    postRepository.deleteById(postId);
}

// 수정 — DB 업데이트 후 캐시 삭제
@CacheEvict(value = "posts", key = "#postId")
public PostDto updatePost(Long postId, UpdateRequest req) {
    Post post = postRepository.findById(postId).orElseThrow();
    post.update(req.title(), req.content());
    return PostDto.from(postRepository.save(post));
}
```

✅ **`allEntries` 옵션**: 특정 캐시의 키를 한 번에 전부 날릴 때 써요.

```java
@CacheEvict(value = "posts", allEntries = true)
public void clearAllPostCache() { ... }
```

<details markdown="1">
<summary>Redis를 쓰면 문제가 될 수 있다?</summary>

`@CacheEvict(allEntries = true)`는 내부적으로 Redis에 `KEYS posts::*` 같은 명령을 날릴 수 있어요. `KEYS`는 되도록 안 쓰는 게 좋은데, 이유는 아래 운영 주의사항에서 설명할게요! SCAN 기반으로 직접 구현하는 걸 추천드립니당

```java
public void evictByPattern(String pattern) {
    ScanOptions options = ScanOptions.scanOptions().match(pattern).count(100).build();
    try (Cursor<byte[]> cursor = redisTemplate.getConnectionFactory()
            .getConnection().scan(options)) {
        while (cursor.hasNext()) {
            redisTemplate.delete(new String(cursor.next()));
        }
    }
}
```

</details>

### `@CachePut`: 캐시 갱신

메서드를 **항상 실행**하고, 그 결과로 캐시를 덮어써요. `@Cacheable`과 달리 Cache Hit여도 메서드가 실행됩니다.

```java
@CachePut(value = "posts", key = "#postId")
public PostDto updatePost(Long postId, UpdateRequest req) {
    // 항상 실행됨 — 결과로 캐시 갱신
    Post post = postRepository.findById(postId).orElseThrow();
    post.update(req.title(), req.content());
    return PostDto.from(postRepository.save(post));
}
```

> 1편에서 얘기했듯 여러 서버가 동시에 수정하는 환경이라면, 갱신(`@CachePut`)보다 삭제(`@CacheEvict`)가 더 안전해요!
{: .prompt-tip }

### ⚠️ Self-invocation 문제가 생길 수 있다?

Spring 캐시는 **AOP 프록시** 기반으로 동작해요. 밖에서 메서드를 호출하면 프록시가 가로채서 캐시 로직을 처리하는 거죠.

→ 문제는 **같은 클래스 안에서 메서드를 호출하면 프록시를 안 거친다는 점..**

```java
@Service
public class PostService {

    public void processPost(Long postId) {
        // ❌ 같은 클래스 내부 호출 — 프록시 안 거침 → 캐시 미적용
        PostDto post = this.getPost(postId);
    }

    @Cacheable(value = "posts", key = "#postId")
    public PostDto getPost(Long postId) {
        return postRepository.findById(postId).map(PostDto::from).orElseThrow();
    }
}
```

```text
외부 호출: Client → [Spring 프록시] → PostService.getPost()  ← 캐시 동작 ✅
내부 호출: PostService.processPost() → this.getPost()         ← 프록시 없음 ❌
```

Spring AOP는 빈을 프록시로 감싸서 동작하는데, `this`로 부르면 프록시가 아니라 실제 객체를 직접 부르는 거라 캐시 로직이 끼어들 틈이 없죠!

**✅ solution: 캐시 메서드를 별도 빈으로 분리하자!**

```java
@Service
@RequiredArgsConstructor
public class PostService {
    private final PostCacheService postCacheService;

    public void processPost(Long postId) {
        // 외부 빈 호출 — 프록시 거침 → 캐시 적용
        PostDto post = postCacheService.getPost(postId);
    }
}

@Service
public class PostCacheService {
    @Cacheable(value = "posts", key = "#postId")
    public PostDto getPost(Long postId) { ... }
}
```

## Redis를 왜 쓰는가

그 전에! Redis가 왜 캐시로 이렇게 많이 쓰이는지 알아볼까요?

**Redis**: 원격에서 동작하는 In-memory 기반 key-value 구조의 NoSQL DB

### Redis의 특성

**1. 초고속 성능**
모든 데이터를 **RAM에 저장**해요. 디스크 I/O가 없으니 읽기/쓰기가 DB보다 빠릅니다.

**2. 다양한 자료구조**
String 말고도 **List, Set, Sorted Set, Hash** 등을 기본으로 지원해요. 덕분에 캐시를 넘어서 실시간 랭킹, 중복 체크, 세션 저장까지 Redis 하나로 해결할 수 있다는 점!!

<details markdown="1">
<summary>조금 더 깊게 알아봅시다</summary>

**Sorted Set (ZSET): 실시간 랭킹.** 값마다 score를 붙여서 자동으로 정렬된 상태를 유지해요.

```java
// 게시글 조회 시 조회수 +1
redisTemplate.opsForZSet().incrementScore("post:ranking", postId.toString(), 1);

// 조회수 상위 10개
Set<ZSetOperations.TypedTuple<String>> top10 =
    redisTemplate.opsForZSet().reverseRangeWithScores("post:ranking", 0, 9);
```

DB에 `ORDER BY view_count DESC LIMIT 10`을 매번 날리는 것보다 훨씬 빠릅니다.

**Set: 중복 체크.** 같은 사용자가 좋아요를 여러 번 누르는 걸 막고 싶을 때:

```java
Boolean added = redisTemplate.opsForSet()
    .add("post:likes:" + postId, userId.toString());

if (Boolean.FALSE.equals(added)) {
    throw new AlreadyLikedException("이미 좋아요를 눌렀습니다.");
}
```

**Hash: 연관 데이터 묶음.** 유저 프로필처럼 관련된 필드들을 한 키로 묶을 때:

```java
redisTemplate.opsForHash().putAll("user:" + userId, Map.of(
    "name", "홍길동",
    "level", "3"
));
String name = (String) redisTemplate.opsForHash().get("user:" + userId, "name");
```

</details>

**3. 서버가 여러 대여도 캐시 공유 가능**
로컬 메모리 캐시는 서버마다 따로 들고 있어서, 서버 A와 서버 B의 캐시가 달라질 수 있어요. 반면 Redis는 **외부 공유 저장소**라 모든 서버가 같은 캐시를 써요. → 분산 환경에서 유용하겠죠?

**4. 영속성 지원**
보통 캐시는 서버 재시작하면 데이터가 다 날아가는데.. Redis는 **RDB 스냅샷**과 **AOF** 두 가지 방식으로 디스크에도 저장할 수 있어요.

### Local(Caffeine) vs Remote(Redis)

**Caffeine**은 Java 기반의 고성능 로컬 인메모리 캐시 라이브러리예요. **W-TinyLFU** 알고리즘을 써서 LRU보다 히트율이 높고, 비동기 처리로 쓰기 성능도 좋습니당

| | Caffeine | Redis |
| --- | --- | --- |
| 속도 | 더 빠름 (메모리 직접 접근) | 네트워크 왕복 있음 |
| 서버 여러 대 | 서버마다 캐시가 따로 | 모든 서버가 공유 |
| 서버 재시작 | 캐시 전부 날아감 | 유지됨 |
| 자료구조 | 단순 Map | String, Hash, Set, Sorted Set 등 |
| 세션 저장 | 불가 (서버 간 공유 X) | 가능 |

→ 서버가 한 대고 단순 캐싱만 필요하면 Caffeine으로 충분해요! 서버가 여러 대거나, 세션 저장이 필요하거나, 자료구조를 활용해야 하면 Redis를 써야 합니다.

### Redis vs Memcached

Redis 말고 Memcached라는 선택지도 있는데, 간단히 비교해 봅시당

| 항목 | Redis | Memcached |
| --- | --- | --- |
| 자료구조 | String, Hash, Set, Sorted Set 등 | 문자열만 |
| 영속성 | RDB, AOF 지원 | 미지원 (항상 휘발성) |
| 스레드 | 싱글 스레드 | 멀티 스레드 |
| 용도 | 캐시, 세션, 메시지 브로커, 랭킹 등 | 단순 캐시 전용 |

- **Memcached를 고려할 때**: 날아가도 원본으로 복구 가능하고, 단순 조회 속도만 올리고 싶을 때
- **Redis를 써야 할 때**: 서버가 여러 대라 세션 공유가 필요하거나, 자료구조나 영속성이 필요할 때

## Redis 운영할 때 주의할 점

### KEYS * 금지

Redis는 싱글 스레드예요. 명령 하나가 실행되는 동안 나머지 요청은 전부 기다려야 하죠.

![KEYS와 싱글 스레드](redis-01-single-thread.png)

`KEYS *`는 저장된 키 전체를 훑는 O(N) 명령이에요. → 키가 100만 개면 100만 개를 다 뒤지고, 그동안 다른 요청은 전부 블로킹돼서 서버가 멈춘 것처럼 될 수도 있어요.

```bash
# ❌
KEYS *
KEYS posts:*

# ✅ 대신 SCAN 쓰기!! 커서 방식으로 조금씩 스캔
SCAN 0 MATCH posts:* COUNT 100
```

`SCAN`은 한 번에 일부만 훑고 커서를 돌려줘서, 블로킹 없이 안전하게 키를 순회할 수 있어용

### maxmemory-policy 설정

Redis 메모리가 꽉 차면 기본 동작은 **에러 반환**이에요. 서버가 죽는 건 아니지만 `OOM command not allowed` 에러를 내니까, 서비스에 장애가 나는 건 마찬가지입니다..

```bash
# redis.conf 또는 ElastiCache 파라미터 그룹
maxmemory 2gb
maxmemory-policy allkeys-lru
```

| 정책 | 동작 |
| --- | --- |
| `noeviction` (기본값) | 꽉 차면 에러 반환 |
| `allkeys-lru` | 전체 키 중 가장 오래 안 쓴 것부터 삭제 |
| `volatile-lru` | TTL 있는 키 중 가장 오래 안 쓴 것부터 삭제 |
| `allkeys-lfu` | 전체 키 중 가장 적게 쓰인 것부터 삭제 |

### 메모리 단편화 (Memory Fragmentation)

![메모리 단편화](redis-02-fragmentation.png)

Redis가 메모리를 반납할 때 빈 공간이 중간중간 생기는 문제예요. 총 여유 메모리는 있는데 **연속된 공간이 없어서** 데이터를 못 넣는 경우가 생깁니다. 실제로 쓰는 것보다 메모리를 더 많이 쓰는 것처럼 보이고, 이 과정에서 프로세스가 죽을 수도 있어요.

```bash
$ redis-cli info memory | grep mem_fragmentation_ratio
mem_fragmentation_ratio:1.7

# redis.conf
activedefrag yes   # 자동 조각 모음 활성화
```

⚠️ `mem_fragmentation_ratio`가 **1.5를 넘으면** 주의가 필요합니다.

### 영속성: RDB vs AOF

Redis는 기본적으로 메모리 기반이라 재시작하면 데이터가 날아가요. 영속성이 필요하면 두 가지 옵션이 있어용

**1️⃣ RDB (스냅샷)**: 설정한 주기마다 메모리 전체를 파일로 저장

```bash
save 900 1    # 900초 안에 변경이 1건 이상이면 저장
save 300 10   # 300초 안에 변경이 10건 이상이면 저장
```

- 장점: 파일 크기 작고, 재시작 시 복구가 빠름
- 단점: 마지막 스냅샷 이후 데이터는 유실

**2️⃣ AOF (Append Only File)**: 모든 쓰기 명령을 로그 파일에 순서대로 기록

```bash
appendonly yes
appendfsync everysec  # 1초마다 디스크에 동기화
```

- 장점: 최대 1초치 데이터만 유실
- 단점: 파일이 커질 수 있고, 복구가 오래 걸릴 수 있음

**❓ 어떤 걸 쓰는 게 좋을까요**

| 상황 | 선택 |
| --- | --- |
| 캐시 전용 (날아가도 됨) | 영속성 끄기 (속도 우선) |
| 세션 저장소 | AOF (최소한의 데이터 보호) |
| 주요 데이터 저장 | RDB + AOF 같이 쓰기 |

## 배포 전략

### 로컬 / 테스트 환경: Docker

```yaml
# docker-compose.yml
services:
  redis:
    image: redis:7.2
    ports:
      - "6379:6379"
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
```

### 운영 환경: AWS ElastiCache

| | Docker (직접 운영) | ElastiCache |
| --- | --- | --- |
| 운영 부담 | 높음 — 설정, 패치, 백업, 모니터링 전부 직접 | 낮음 — AWS가 처리 |
| 모니터링 | 직접 구축 | CloudWatch 연동 기본 제공 |
| 페일오버 | 직접 Sentinel 구성 | 자동 장애 감지 + 복구 |
| 확장성 | 샤딩, 노드 증감 수동 | 콘솔에서 클릭 몇 번 |
| 보안 | 네트워크, 인증, TLS 직접 | VPC, IAM, 암호화 기본 제공 |
| 비용 | 인프라 비용만 | 관리형 비용 추가 |

### Valkey

2024년 3월, Redis가 라이선스를 오픈소스에서 SSPL/RSAL로 바꾸면서 클라우드 업체들이 Redis를 관리형 서비스로 제공하기 어려워졌어요. 그래서 AWS, Google, Linux Foundation 등이 Redis 7.2.4를 포크해서 **Valkey**를 만들었습니다.

현재 상황을 요약해 보자면,,

- AWS ElastiCache 새 인스턴스의 기본값이 Valkey로 바뀌었다
- ElastiCache for Valkey는 Redis OSS보다 노드 비용이 약 20%, 서버리스 비용이 약 33% 저렴하다
- API는 Redis와 호환 → **코드 변경 없이 그대로 쓸 수 있다**

당장 크게 바뀌는 건 없지만, 새 프로젝트를 ElastiCache에 올린다면 기본값이 Valkey라는 건 알고 있기를~

## 참고 자료

- [Redis 실전 활용 (feat. AWS ElastiCache)](https://look-forest.tistory.com/165)
- [개발자가 알면 좋은 Redis 꿀팁 모음 (올리브영 테크블로그)](https://oliveyoung.tech/2025-07-23/redis-tips-for-developer/)
- [Spring Cache와 CacheManager 이해하기](https://mangkyu.tistory.com/370)
