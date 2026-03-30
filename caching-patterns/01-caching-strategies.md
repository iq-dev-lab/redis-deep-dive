# 캐싱 전략 비교 — Cache-Aside / Write-Through / Write-Back

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Cache-Aside, Write-Through, Write-Back, Read-Through는 각각 언제 적합한가?
- 각 전략에서 캐시-DB 불일치(stale data)가 발생하는 상황은?
- Write-Back의 쓰기 증폭(Write Amplification) 감소 효과와 데이터 유실 위험은?
- Spring `@Cacheable`이 내부적으로 구현하는 전략은?
- 캐시 무효화(Cache Invalidation)가 "CS에서 가장 어려운 문제" 중 하나인 이유는?
- 읽기 전용 데이터 vs 자주 변경되는 데이터에서 전략 선택 기준은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

캐싱 전략 선택은 성능과 일관성 사이의 트레이드오프를 결정한다. 전략을 모르고 Redis를 단순히 "빠른 DB 앞단"으로 쓰면 캐시와 DB 사이에 스테일 데이터가 쌓이거나, 불필요한 쓰기 증폭이 발생하거나, 장애 시 데이터를 잃게 된다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Cache-Aside에서 쓰기 후 캐시를 업데이트(SET)하는 패턴

  잘못된 코드:
    def update_user(user_id, data):
        db.update(user_id, data)       # DB 업데이트
        cache.set(f"user:{user_id}", data)  # 캐시도 업데이트

  문제:
    동시 요청 A: db.update(user=1, name="Alice")
    동시 요청 B: db.update(user=1, name="Bob")
    실행 순서:  A.db → B.db → B.cache → A.cache
    결과:       DB에는 "Bob"(최신), 캐시에는 "Alice"(구버전)
    → 스테일 캐시 발생, 시간이 지나야(TTL) 해소

  올바른 패턴: 쓰기 시 캐시를 DELETE (무효화)
    def update_user(user_id, data):
        db.update(user_id, data)
        cache.delete(f"user:{user_id}")  # SET 대신 DELETE!

실수 2: Write-Back에서 Redis 장애 시 데이터 유실

  설계: 모든 쓰기를 Redis에만 하고 주기적으로 DB에 플러시
  장애: Redis 서버 다운 (AOF 없음)
  결과: 플러시 안 된 쓰기 전부 손실
  
  "Redis가 빠르니까 다 거기다 쓰자"는 설계가 위험한 이유

실수 3: TTL 없이 캐시 저장

  cache.set("product:1", product_data)  # TTL 없음!
  
  결과: DB에서 상품 정보가 변경돼도 캐시는 영원히 구버전
  → 사용자가 잘못된 가격/재고 정보를 봄
  → TTL이나 명시적 무효화가 없으면 캐시는 레거시가 됨
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
쓰기 시 캐시 무효화 패턴 (Cache-Aside):
  읽기: cache miss → DB 조회 → cache SET (TTL 설정)
  쓰기: DB UPDATE → cache DELETE (SET 아님!)
  → 다음 읽기 시 최신 DB 값으로 재적재

전략별 적합한 상황:
  Cache-Aside:    읽기 많음, 쓰기 드문 데이터 (상품 정보, 설정값)
  Write-Through:  쓰기와 읽기 일관성 모두 중요 (세션, 사용자 프로필)
  Write-Back:     쓰기 폭발적, 약간의 유실 허용 (조회수, 좋아요 수)
  Read-Through:   캐시 로직을 중앙화하고 싶을 때 (캐시 라이브러리)

Spring @Cacheable 동작 이해:
  @Cacheable("users")     → Cache-Aside 읽기 (miss면 DB 조회 + 캐시 저장)
  @CacheEvict("users")    → 명시적 무효화
  @CachePut("users")      → Write-Through 쓰기 (항상 캐시+DB 동시 업데이트)
  
  주의: @Cacheable는 쓰기 시 자동 무효화 안 함
  → @CacheEvict 또는 @CachePut을 명시적으로 붙여야 함
```

---

## 🔬 내부 동작 원리

### 1. Cache-Aside (Lazy Loading)

```
읽기 흐름:
  클라이언트 → GET user:1
               │
               ▼ Cache hit?
         ┌─ Yes → 캐시 값 반환 (빠름)
         │
         └─ No (cache miss)
               │
               ▼
           DB 조회 (느림)
               │
               ▼
           cache.SET user:1 {data} EX 3600  ← 다음 요청을 위해 저장
               │
               ▼
           클라이언트에 반환

쓰기 흐름 (올바른 패턴):
  클라이언트 → UPDATE user:1 name="Bob"
               │
               ▼
           DB UPDATE user SET name="Bob" WHERE id=1
               │
               ▼
           cache.DEL user:1  ← 무효화 (SET 아님!)
           
  다음 읽기 → cache miss → DB 조회 → 캐시 저장 (최신값으로 갱신)

Cache-Aside의 불일치 위험:
  DB 업데이트와 캐시 DELETE 사이에 크래시 발생 시:
    DB: name="Bob" (최신)
    Cache: name="Alice" (구버전, TTL 전까지 유지)
  → 최대 TTL 시간 동안 스테일 캐시 서빙 가능
  → 허용 범위: TTL 설정값이 스테일 허용 시간

특성:
  장점: 구현 단순, 필요한 데이터만 캐시 (Lazy)
  단점: 첫 요청 항상 느림 (Cold Miss), 약간의 불일치 가능
  적합: 읽기 많고 쓰기 드문 데이터
```

### 2. Write-Through

```
쓰기 흐름:
  클라이언트 → UPDATE user:1 name="Bob"
               │
               ▼
           cache.SET user:1 {name: "Bob"} EX 3600
               │
               ▼
           DB UPDATE user SET name="Bob" WHERE id=1
               │
               ▼
           클라이언트에 OK 반환
           (DB 쓰기 완료 후 반환 → 쓰기 지연 증가)

읽기 흐름: Cache-Aside와 동일

Write-Through의 특성:
  장점:
    캐시-DB 항상 일관성 (쓰기 직후 읽어도 최신값)
    캐시 miss 감소 (쓸 때마다 캐시 채워짐)
  단점:
    쓰기 지연 증가 (캐시 + DB 둘 다 써야 함)
    쓰기 증폭: 쓰기는 적고 캐시된 데이터를 아무도 읽지 않아도 캐시에 저장
    → 메모리 낭비 (자주 읽히지 않는 데이터도 캐시에 올라감)
  적합: 쓰기 후 즉시 읽기 일관성이 중요한 경우 (세션, 프로필)

Spring @CachePut:
  @CachePut("users")
  public User updateUser(Long id, UserDto dto) {
      User user = userRepository.save(dto.toEntity(id));
      return user;  // 반환값이 캐시에 저장됨
  }
  → 항상 캐시 + DB 동시 업데이트 (Write-Through 패턴)
```

### 3. Write-Back (Write-Behind)

```
쓰기 흐름:
  클라이언트 → UPDATE view_count:article:1 +1
               │
               ▼
           cache.INCR view_count:article:1  ← Redis에만 즉시 반영
               │
               ▼
           클라이언트에 즉시 OK
           
  (별도 플러셔가 주기적으로 DB에 반영):
  
  Flusher(비동기):
    KEYS view_count:* → 수집
    UPDATE articles SET view_count = {redis_value} WHERE id = {id}
    (또는 일괄 INCR)

Write-Back의 특성:
  장점:
    쓰기 응답 시간 최소화 (Redis만 쓰고 즉시 반환)
    쓰기 증폭 감소: N번 업데이트 → DB에는 1번만 반영 가능
    DB 쓰기 폭발 완화 (배치 플러시)
  단점:
    Redis 장애 시 미플러시 데이터 유실 위험
    복잡도 증가 (플러셔 컴포넌트 별도 운영 필요)
    캐시-DB 불일치: 플러시 전까지 DB가 최신값 아님
  적합: 조회수, 좋아요 수, 실시간 카운터 (약간의 유실 허용 가능)

Redis 구현 예시:
  # 쓰기: Redis에만
  INCR view_count:article:1
  
  # 플러셔 (주기적 실행, 예: 10초마다)
  for key in redis.scan_iter("view_count:*"):
      count = redis.getset(key, 0)  # 현재값 가져오고 0으로 초기화
      article_id = key.split(":")[-1]
      db.execute("UPDATE articles SET view_count = view_count + %s WHERE id = %s",
                 [count, article_id])
```

### 4. Read-Through

```
Read-Through는 캐시 계층이 DB 조회를 직접 담당:

  클라이언트 → 캐시 계층에만 요청
  캐시 miss → 캐시 계층이 DB 조회 → 저장 → 클라이언트에 반환
  
  Cache-Aside와의 차이:
    Cache-Aside: 애플리케이션이 캐시와 DB를 직접 모두 호출
    Read-Through: 애플리케이션은 캐시만 호출, 캐시가 DB 호출 담당

Redis로 직접 구현하기 어려운 패턴:
  Redis 자체는 DB 조회 로직이 없음
  → 라이브러리(NCache, Ehcache + Redis)가 제공하는 패턴
  → Spring Cache + CustomCacheManager로 근사 구현 가능

Spring Cache 추상화:
  @Cacheable("users")         → Read-Through 근사
  CacheManager → RedisCacheManager → Redis 연동
  Cache miss 시 메서드 실행(DB 조회) → 결과를 Redis에 자동 저장
```

### 5. 캐시 무효화 전략

```
세 가지 무효화 방법:

① TTL 기반 (가장 단순):
  SETEX user:1 3600 {data}  # 1시간 후 자동 만료
  → 최대 TTL 시간만큼 스테일 데이터 존재 가능
  → 허용 범위: 상품 가격(1분), 환율(30분), 정적 설정(1시간)

② 이벤트 기반 (명시적 무효화):
  쓰기 발생 시: cache.DEL user:1
  → 즉각 무효화, 다음 읽기 시 최신값 재로드
  → 구현 복잡도 증가 (모든 쓰기 경로에 무효화 코드 필요)

③ 버전 기반:
  키에 버전 포함: user:1:v5
  데이터 변경 시: 버전 증가 → user:1:v6 생성, user:1:v5 자연 만료
  → 복잡하지만 무효화 누락 방지

"캐시 무효화는 CS에서 가장 어려운 문제" — Phil Karlton
  이유:
    분산 환경에서 여러 서버의 캐시를 동시에 무효화하는 것의 어려움
    쓰기 실패 + 캐시 무효화 성공 → DB는 구버전, 캐시는 비어있음 (역전 불일치)
    무효화 누락 → 영구 스테일 데이터
    과도한 무효화 → 캐시 효율 저하 (Cache Stampede 유발)
```

---

## 💻 실전 실험

### 실험 1: Cache-Aside 패턴 구현

```bash
# 읽기: Cache miss → DB 조회 → 캐시 저장
get_user() {
  USER_ID=$1
  CACHED=$(redis-cli GET "user:$USER_ID")
  
  if [ -z "$CACHED" ]; then
    echo "Cache MISS — DB 조회"
    # DB 조회 시뮬레이션
    DATA='{"id":'$USER_ID',"name":"Alice","age":30}'
    redis-cli SETEX "user:$USER_ID" 3600 "$DATA" > /dev/null
    echo $DATA
  else
    echo "Cache HIT"
    echo $CACHED
  fi
}

get_user 1  # miss → DB 조회
get_user 1  # hit  → 캐시 반환

# 쓰기: DB 업데이트 + 캐시 무효화
update_user() {
  USER_ID=$1
  DATA=$2
  echo "DB 업데이트 + 캐시 무효화"
  # DB 업데이트 시뮬레이션
  redis-cli DEL "user:$USER_ID" > /dev/null  # 캐시 DELETE
  echo "완료"
}

update_user 1 '{"name":"Bob"}'
get_user 1  # miss → 최신 DB 값으로 재로드
```

### 실험 2: Write-Back 카운터 패턴

```bash
# 조회수 카운터 (Write-Back 패턴)

# 쓰기: Redis에만 즉시 반영
redis-cli INCR "view_count:article:1"
redis-cli INCR "view_count:article:1"
redis-cli INCR "view_count:article:1"

# 플러셔 (10초마다 DB에 반영하는 스크립트 시뮬레이션)
flush_to_db() {
  echo "=== DB 플러시 시작 ==="
  for key in $(redis-cli SCAN 0 MATCH "view_count:*" COUNT 100 | tail -n +2); do
    count=$(redis-cli GETSET "$key" 0)  # 현재값 가져오고 0으로 초기화
    article_id=$(echo $key | cut -d: -f3)
    echo "  article $article_id: +$count views → DB UPDATE"
  done
  echo "=== 플러시 완료 ==="
}

flush_to_db

# 플러시 후 Redis 값 확인
redis-cli GET "view_count:article:1"  # 0 (초기화됨)
```

### 실험 3: Spring @Cacheable 동작 확인 (redis-cli로 검증)

```bash
# Spring @Cacheable이 저장하는 키 형식 확인
# Spring Boot + Spring Cache + RedisCache 사용 시

# @Cacheable("users") findById(1L) 실행 후
redis-cli KEYS "users*"
# users::1  또는  users::SimpleKey [1]  (버전마다 다름)

redis-cli GET "users::1"
# 직렬화된 Java 객체 (바이너리)

redis-cli TTL "users::1"
# RedisCacheConfiguration에 설정한 TTL 확인

# @CacheEvict("users") 실행 후
redis-cli EXISTS "users::1"  # 0 (삭제됨)
```

### 실험 4: 캐시 무효화 패턴 비교

```bash
# 전략 1: TTL 기반
redis-cli SETEX "product:1" 60 '{"price":1000}'  # 60초 TTL
redis-cli TTL "product:1"  # 59 (TTL 확인)

# DB에서 가격 변경됐다고 가정
# → 최대 60초 동안 구버전 가격 서빙 가능

# 전략 2: 이벤트 기반 즉시 무효화
update_product_price() {
  PRODUCT_ID=$1
  NEW_PRICE=$2
  # DB 업데이트
  echo "DB: product $PRODUCT_ID price → $NEW_PRICE"
  # 캐시 즉시 무효화
  redis-cli DEL "product:$PRODUCT_ID"
  echo "캐시 무효화 완료"
}

update_product_price 1 1500
redis-cli EXISTS "product:1"  # 0 (즉시 무효화)

# 전략 3: 버전 기반
redis-cli SET "product:1:v1" '{"price":1000}'
redis-cli SET "product:1:version" "1"

# 업데이트 시
redis-cli SET "product:1:v2" '{"price":1500}'  # 새 버전 생성
redis-cli SET "product:1:version" "2"          # 버전 포인터 업데이트
# 이후 읽기: GET product:1:version → v2 → GET product:1:v2
```

---

## 📊 성능/비용 비교

```
캐싱 전략별 특성 비교:

전략           | 읽기 성능  | 쓰기 성능  | 일관성     | 구현 복잡도   | 적합한 경우
──────────────┼──────────┼──────────┼──────────┼────────────┼────────────────
Cache-Aside   | 높음      | 중간      | 최종 일관성 | 낮음        | 읽기 많은 데이터
Write-Through | 높음      | 낮음      | 강한 일관성 | 중간        | 쓰기 후 즉시 읽기
Write-Back    | 높음      | 최고      | 약한 일관성 | 높음        | 쓰기 폭발적 데이터
Read-Through  | 높음      | 중간      | 최종 일관성 | 높음        | 캐시 로직 중앙화

캐시 히트율 목표:
  일반 캐시: 80% 이상
  고성능 캐시: 95% 이상
  
  히트율 = (캐시 반환 횟수) / (전체 요청 횟수) × 100
  redis-cli INFO stats | grep keyspace_hits  # 캐시 히트
  redis-cli INFO stats | grep keyspace_misses # 캐시 미스

Write Amplification 비교:
  Cache-Aside: 쓰기 1회 (DB만)
  Write-Through: 쓰기 2회 (캐시 + DB)
  Write-Back: 쓰기 1회 (캐시) + 주기적 DB (N→1 통합)
  → Write-Back이 쓰기 폭발 시 DB 부하 가장 적음
```

---

## ⚖️ 트레이드오프

```
일관성 vs 성능:
  Write-Through: 일관성 높음, 쓰기 지연 증가
  Cache-Aside:   성능 우선, 최대 TTL만큼 스테일 허용
  Write-Back:    성능 최대, 유실 위험 존재

TTL 설정의 트레이드오프:
  짧은 TTL: 스테일 데이터 최소, 캐시 히트율 감소, DB 부하 증가
  긴 TTL:  캐시 히트율 높음, 스테일 데이터 길게 유지
  → 데이터 변경 빈도에 맞게 TTL 결정

Write-Back 유실 위험 완화:
  Redis AOF everysec → 최대 1초 유실
  플러셔가 주기적으로 DB에 반영 → 미반영 데이터 최소화
  → 완전한 유실 방지는 불가 (설계상 허용)
```

---

## 📌 핵심 정리

```
캐싱 전략 선택 기준:

Cache-Aside (가장 일반적):
  읽기: miss → DB → 캐시 저장 (TTL 필수)
  쓰기: DB 업데이트 → 캐시 DELETE (SET 아님!)
  적합: 읽기 위주, 약간의 스테일 허용 가능

Write-Through:
  쓰기 시 캐시 + DB 동시 업데이트
  적합: 쓰기 후 즉시 일관성 필요

Write-Back:
  쓰기 시 캐시만 → 나중에 DB 배치 반영
  적합: 쓰기 폭발적, 약간의 유실 허용

캐시 무효화 원칙:
  쓰기 시 SET 대신 DELETE (Race Condition 방지)
  TTL 없이 캐시 저장 금지
  DB 업데이트와 캐시 무효화는 원자적으로 처리 어려움
  → 무효화 실패 대비 TTL 병행

Spring:
  @Cacheable → Cache-Aside 읽기
  @CacheEvict → 명시적 무효화
  @CachePut  → Write-Through
```

---

## 🤔 생각해볼 문제

**Q1.** Cache-Aside에서 쓰기 시 캐시를 DELETE 대신 SET(업데이트)을 했을 때 발생하는 Race Condition을 단계별로 설명하라.

<details>
<summary>해설 보기</summary>

**Race Condition 시나리오 (캐시 SET 패턴):**

```
Thread A: 사용자 1의 이름을 "Alice"로 업데이트
Thread B: 사용자 1의 이름을 "Bob"으로 업데이트 (동시)

t=1: A → DB UPDATE name="Alice"
t=2: B → DB UPDATE name="Bob"   ← DB 최종값: "Bob"
t=3: B → cache.SET user:1 {name:"Bob"}
t=4: A → cache.SET user:1 {name:"Alice"}  ← 오래된 값으로 덮어씀!

결과:
  DB: name="Bob" (최신)
  Cache: name="Alice" (구버전!) ← TTL 전까지 영구 스테일
```

**DELETE 패턴은 이 문제를 회피한다:**
```
t=1: A → DB UPDATE name="Alice"
t=2: B → DB UPDATE name="Bob"   ← DB 최종값: "Bob"
t=3: B → cache.DEL user:1
t=4: A → cache.DEL user:1 (이미 없음, 무해)

다음 읽기 → cache miss → DB 조회 → "Bob" (최신값 반환)
```

DELETE가 더 안전한 이유: 삭제는 멱등성이 있다. 여러 번 삭제해도 결과는 "캐시 없음"으로 동일하다. 반면 SET은 마지막에 쓴 값이 남으므로, 쓰기 순서가 보장되지 않는 분산 환경에서 구버전이 남는 위험이 있다.

</details>

---

**Q2.** Write-Back 패턴에서 조회수(view_count)를 Redis에 누적하다가 플러셔가 DB에 반영하기 전에 Redis 서버가 다운됐다. AOF everysec 설정이 있다면 데이터 유실 범위는?

<details>
<summary>해설 보기</summary>

AOF everysec 설정으로 최대 **1초치** Redis 쓰기가 유실될 수 있다.

구체적으로:
- AOF는 1초마다 fsync → 마지막 fsync 이후 ~ 다운까지 최대 1초 간격의 `INCR` 명령어들이 유실
- 초당 1,000번 조회가 발생하는 콘텐츠라면 → 최대 ~1,000회 조회수 유실
- 플러셔 주기가 10초라면 → 최근 플러시 이후 누적된 카운터 유실 (최대 수만 건)

**중요한 구분:**
- AOF가 보호하는 것: Redis 재시작 후 데이터 복원 (서버 재시작 시나리오)
- AOF가 보호 못하는 것: 마지막 fsync 이후 쓰기 (1초 이내)

**설계 시 허용 기준:**
조회수의 경우 수십~수백 건 유실은 일반적으로 허용된다. 하지만 재고 수량이나 포인트 같은 중요 카운터라면 Write-Back보다 더 강한 일관성 보장이 필요하다.

```bash
# 보완: BGSAVE를 플러시 직후 실행
flush_to_db()
redis-cli BGSAVE  # RDB로 현재 상태 스냅샷
```

</details>

---

**Q3.** `@Cacheable("products") findById(1L)`을 호출하는 서비스가 있고, 동시에 10개 스레드가 동일 메서드를 처음 호출한다(캐시 cold start). 어떤 일이 발생하고, 이를 어떻게 방지하는가?

<details>
<summary>해설 보기</summary>

**Cache Stampede 발생:**
10개 스레드 모두 캐시 miss → 10개 스레드 모두 DB 조회 실행 → DB에 동일 쿼리 10번 발생.

**Spring의 기본 @Cacheable 동작:**
Spring은 기본적으로 캐시 조회와 저장 사이에 Lock을 걸지 않는다. 동시에 모든 스레드가 DB를 조회한다.

**방지 방법:**

1. **`sync = true` 옵션 (Spring 4.3+):**
```java
@Cacheable(value = "products", sync = true)
public Product findById(Long id) {
    return productRepository.findById(id);
}
```
→ 동일 키에 대해 단 하나의 스레드만 메서드 실행, 나머지는 결과 대기
→ Caffeine, Ehcache 등 일부 CacheManager에서 지원 (Redis는 제한적)

2. **분산 Mutex Lock (Redisson):**
```java
RLock lock = redissonClient.getLock("product:" + id + ":lock");
if (lock.tryLock(0, 30, TimeUnit.SECONDS)) {
    try {
        // Cache-Aside 로직
    } finally {
        lock.unlock();
    }
}
```

3. **PER (Probabilistic Early Recomputation):** → 다음 문서(02-cache-stampede.md) 참고

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: Cache Stampede ➡️](./02-cache-stampede.md)**

</div>
