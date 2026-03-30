# Spring Cache 추상화 — @Cacheable 내부 AOP

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@Cacheable` 호출 시 내부 AOP 체인은 어떤 순서로 실행되는가?
- `SimpleKeyGenerator`는 어떤 규칙으로 캐시 키를 생성하는가?
- `@CacheEvict(allEntries = true)`는 어떻게 동작하고, Redis에서 어떤 명령어를 실행하는가?
- `@CachePut`이 `@Cacheable`과 다른 실행 시점은?
- `@Cacheable`이 `@Transactional`과 함께 사용될 때 어떤 순서 문제가 생기는가?
- `condition`과 `unless`의 차이는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

`@Cacheable`은 단순히 "메서드 결과를 캐시한다"는 것보다 훨씬 복잡한 동작을 한다. AOP 체인의 실행 순서, 키 생성 방식, `@Transactional`과의 조합에서 예상치 못한 동작이 발생한다. 특히 `@CacheEvict(allEntries=true)`가 Redis KEYS 명령어를 실행한다는 사실을 모르면 운영 중 성능 장애를 유발할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 같은 클래스 내에서 @Cacheable 메서드를 직접 호출

  코드:
    @Service
    public class UserService {
        @Cacheable("users")
        public User findById(Long id) { ... }
        
        public void processAll() {
            List<Long> ids = getIds();
            for (Long id : ids) {
                User user = findById(id);  // ← 직접 호출!
                process(user);
            }
        }
    }
  
  결과: 캐시가 전혀 동작하지 않음!
  이유: Spring AOP는 프록시 패턴 → 외부에서 호출할 때만 인터셉트
        같은 클래스 내부 호출 = 프록시 우회 = 캐시 미작동

실수 2: @CacheEvict(allEntries=true)를 대형 캐시에 사용

  코드:
    @CacheEvict(value = "products", allEntries = true)
    public void clearAllProducts() { ... }
  
  내부 동작:
    keys = redisTemplate.keys("products::*")  ← KEYS 패턴 명령어!
    redisTemplate.delete(keys)
  
  운영 환경:
    "products" 캐시에 키 100만개 → KEYS 명령어 → 수백 ms 이벤트 루프 블로킹!
    → 전체 서비스 지연

실수 3: @Transactional + @CacheEvict 순서 문제

  코드:
    @Transactional
    @CacheEvict("users")
    public void updateUser(Long id, UserDto dto) {
        userRepository.save(dto.toEntity(id));
    }
  
  문제:
    @CacheEvict가 메서드 완료 후 실행됨
    하지만 트랜잭션 커밋은 그보다 늦음
    → 캐시 삭제됨 → 다른 스레드가 DB 읽기 시도
    → DB에 아직 커밋 안 됨 → 구버전 데이터 캐시에 저장됨!
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
AOP 프록시 외부 호출 보장:
  // 방법 1: ApplicationContext로 자기 자신 참조
  @Service
  public class UserService {
      @Autowired
      private ApplicationContext applicationContext;
      
      public void processAll() {
          UserService proxy = applicationContext.getBean(UserService.class);
          for (Long id : ids) {
              User user = proxy.findById(id);  // 프록시를 통한 호출
          }
      }
  }
  
  // 방법 2: 별도 서비스로 분리 (더 깔끔)
  @Service
  public class UserCacheService {
      @Cacheable("users")
      public User findById(Long id) { ... }
  }
  
  @Service
  public class UserService {
      @Autowired
      private UserCacheService userCacheService;
      
      public void processAll() {
          userCacheService.findById(id);  // 외부 호출 = 캐시 동작
      }
  }

@CacheEvict allEntries 대안:
  # allEntries=true 대신 Redis SCAN으로 안전하게 삭제
  @Autowired
  private RedisTemplate<String, Object> redisTemplate;
  
  public void clearProductCache() {
      // SCAN으로 안전하게 순회 삭제
      ScanOptions options = ScanOptions.scanOptions().match("products::*").count(100).build();
      redisTemplate.scan(options).forEachRemaining(key -> redisTemplate.delete(key));
  }

@Transactional + @CacheEvict 안전 조합:
  @CacheEvict(value = "users", afterInvocation = true)
  // afterInvocation=true (기본값): 메서드 완료 후 캐시 삭제
  // 트랜잭션 커밋 후 삭제하려면 TransactionSynchronizationManager 사용
  
  @Transactional
  public void updateUser(Long id, UserDto dto) {
      userRepository.save(dto.toEntity(id));
      TransactionSynchronizationManager.registerSynchronization(
          new TransactionSynchronizationAdapter() {
              @Override
              public void afterCommit() {
                  cacheManager.getCache("users").evict(id);  // 커밋 후 삭제
              }
          });
  }
```

---

## 🔬 내부 동작 원리

### 1. @Cacheable AOP 체인

```
@Cacheable 메서드 호출 흐름:

외부 코드 → Spring Bean (Proxy)
              │
              ▼
    CacheInterceptor.invoke()
    (org.springframework.cache.interceptor)
              │
              ▼
    CacheAspectSupport.execute()
              │
       [캐시 조회 단계]
              │
              ▼
    CacheResolver.resolveCaches()
    → RedisCacheManager.getCache("users")
    → RedisCache 인스턴스 반환
              │
              ▼
    CacheAspectSupport.findCachedItem()
    → KeyGenerator.generate(target, method, params)
    → SimpleKeyGenerator: "users::1" 형태 키 생성
    → RedisCache.get(key)
    → RedisTemplate.opsForValue().get(key)
              │
       [캐시 히트?]
       ├── YES: 캐시 값 반환 (메서드 실행 안 함!)
       └── NO: 실제 메서드 실행
              │
              ▼
       [메서드 실행]
    실제 비즈니스 로직 (DB 조회 등)
              │
              ▼
       [캐시 저장]
    RedisCache.put(key, result)
    → RedisTemplate.opsForValue().set(key, result, ttl)
              │
              ▼
    결과 반환

AOP 프록시의 한계:
  Spring AOP는 프록시 기반 (CGLib or JDK Dynamic Proxy)
  외부에서 프록시를 통해 호출 → AOP 동작
  같은 클래스 내 직접 호출 → 프록시 우회 → AOP 미동작
  → @Cacheable 무시됨!
```

### 2. 키 생성 전략 — SimpleKeyGenerator

```java
// SimpleKeyGenerator 규칙:
// 파라미터 0개: SimpleKey.EMPTY
// 파라미터 1개: 파라미터 값 그대로
// 파라미터 2개+: SimpleKey(params) 래퍼

@Cacheable("users")
User findById(Long id) → 키: "users::1"
// → cacheName + "::" + id.toString()

@Cacheable("users")
User findByNameAndAge(String name, int age) → 키: "users::SimpleKey [Alice,30]"
// → 복합 키

// 기본 키 형식:
// "{cacheName}::{keyValue}"
// RedisCacheManager의 keyPrefix 설정으로 변경 가능

// 커스텀 KeyGenerator:
@Component("userKeyGenerator")
public class UserKeyGenerator implements KeyGenerator {
    @Override
    public Object generate(Object target, Method method, Object... params) {
        return "user:" + Arrays.stream(params)
            .map(Object::toString)
            .collect(Collectors.joining(":"));
    }
}

@Cacheable(value = "users", keyGenerator = "userKeyGenerator")
User findById(Long id) → 키: "users::user:1"

// SpEL 키 표현식 (가장 유연):
@Cacheable(value = "users", key = "#id")
User findById(Long id) → 키: "users::1"

@Cacheable(value = "users", key = "'user:' + #id")
User findById(Long id) → 키: "users::user:1"

@Cacheable(value = "orders", key = "#userId + ':' + #status")
List<Order> findByUserAndStatus(Long userId, String status)
→ 키: "orders::1:PAID"
```

### 3. @CacheEvict 내부 동작

```
@CacheEvict 방식 1: 특정 키 삭제
  @CacheEvict(value = "users", key = "#id")
  void deleteUser(Long id)
  
  실행: RedisCache.evict("1")
  → RedisTemplate.delete("users::1")
  → Redis: DEL users::1
  → O(1), 안전

@CacheEvict 방식 2: allEntries = true
  @CacheEvict(value = "users", allEntries = true)
  void clearAllUsers()
  
  내부 동작 (Spring Boot 기본):
  // RedisCache.clear() 내부
  byte[] pattern = (cacheKeyPrefix + "*").getBytes();
  Set<byte[]> keys = connection.keys(pattern);  ← KEYS 명령어!
  connection.del(keys);
  
  → "users::*" 패턴으로 KEYS 실행
  → 100만개 키 환경에서 수백 ms 블로킹!
  
  안전한 대안 설정:
  RedisCacheConfiguration.defaultCacheConfig()
      .enableCleanup()  // SCAN 방식 사용 (Spring 3.x+)
  
  또는 수동 SCAN:
  @CacheEvict(value = "users", allEntries = true)에 의존하지 말고
  직접 Redis SCAN + DELETE 구현

afterInvocation 옵션:
  @CacheEvict(afterInvocation = true)  // 기본값: 메서드 실행 후 삭제
  @CacheEvict(afterInvocation = false) // 메서드 실행 전 삭제
  
  예외 발생 시:
    afterInvocation = true: 메서드 예외 → 캐시 삭제 안 됨 (일관성 유지)
    afterInvocation = false: 메서드 예외 → 캐시는 이미 삭제됨 (다음 조회 시 DB에서 재로드)
```

### 4. @CachePut vs @Cacheable

```
@Cacheable: 캐시 히트 시 메서드 실행 안 함
  → 성능 최적화 목적

@CachePut: 항상 메서드 실행 + 결과를 항상 캐시에 저장
  → 캐시 업데이트 목적

비교 예시:

@Cacheable("users")
User findById(Long id)
  → 첫 호출: DB 조회 + 캐시 저장
  → 이후 호출: 캐시에서 바로 반환 (DB 조회 없음)

@CachePut(value = "users", key = "#user.id")
User updateUser(User user)
  → 항상 실행: DB 업데이트 + 결과를 캐시에 저장
  → Write-Through 패턴 구현에 사용

같은 키에 함께 사용:
  @Cacheable("users")   // 읽기 시 캐시 사용
  User findById(Long id) { ... }
  
  @CachePut(value = "users", key = "#user.id")  // 쓰기 시 캐시 갱신
  User save(User user) { ... }
  
  → 읽기는 캐시, 쓰기는 항상 DB + 캐시 업데이트 (Write-Through)

condition vs unless:
  condition: 메서드 실행 전 평가 (캐시 자체 활성화 조건)
    @Cacheable(condition = "#id > 0")  // id가 양수일 때만 캐싱
    
  unless: 메서드 실행 후 평가 (반환값 기반 캐시 저장 조건)
    @Cacheable(unless = "#result == null")  // null 결과는 캐시 안 함
    @Cacheable(unless = "#result.isEmpty()") // 빈 결과는 캐시 안 함
    
  차이:
    condition: 캐시 조회도, 저장도 안 함
    unless:    캐시 조회는 하지만 저장은 안 함 (결과를 알아야 판단)
```

### 5. @Transactional과 @Cacheable 조합 주의사항

```
실행 순서 문제:

1. @Cacheable + @Transactional (읽기):
   @Cacheable("users")
   @Transactional(readOnly = true)
   User findById(Long id)
   
   실행 순서:
   CacheInterceptor (캐시 조회) → TransactionInterceptor (트랜잭션 시작)
   → 메서드 실행 → TransactionInterceptor (트랜잭션 종료) → CacheInterceptor (캐시 저장)
   
   캐시 히트 시:
   CacheInterceptor가 먼저 → 캐시에서 반환 → 트랜잭션 시작도 안 함
   → 정상

2. @CacheEvict + @Transactional (쓰기) 위험:
   @Transactional
   @CacheEvict("users")
   void updateUser(Long id, dto)
   
   실행 순서:
   TransactionInterceptor (트랜잭션 시작)
   → 메서드 실행 (DB UPDATE, 아직 커밋 안 됨)
   → @CacheEvict 실행 (캐시 삭제됨!)
   → 동시 요청: 캐시 miss → DB 조회 → 아직 커밋 안 된 구버전 읽음 → 캐시 저장
   → TransactionInterceptor (트랜잭션 커밋)
   → 캐시에는 이미 구버전 데이터!

안전한 패턴:
  방법 1: @CacheEvict를 트랜잭션 밖에서 실행
    Service에서 트랜잭션 없는 메서드가 @CacheEvict + 트랜잭션 메서드 호출
  
  방법 2: TransactionSynchronization 사용 (커밋 후 삭제)
    TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {
        @Override
        public void afterCommit() {
            cache.evict(id);  // 커밋 완료 후 캐시 삭제
        }
    });
  
  방법 3: Spring Cache Tx Support
    // application.yml
    spring.cache.redis.enable-statistics: true
    // @EnableCaching(proxyTargetClass = true)
```

---

## 💻 실전 실험

### 실험 1: @Cacheable 동작 확인

```java
@SpringBootTest
class CacheableTest {
    
    @Autowired
    private UserService userService;
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Test
    void cacheableWorks() {
        // 1. 첫 호출: DB 조회 + 캐시 저장
        User user1 = userService.findById(1L);
        
        // 2. Redis에 캐시됐는지 확인
        Object cached = redisTemplate.opsForValue().get("users::1");
        assertThat(cached).isNotNull();
        
        // 3. 두 번째 호출: 캐시에서 반환 (DB 조회 안 함)
        User user2 = userService.findById(1L);
        assertThat(user1).isEqualTo(user2);
        
        // 4. 캐시 히트율 확인 (INFO stats)
        // keyspace_hits가 증가하고 keyspace_misses는 1회만
    }
    
    @Test
    void selfInvocationBypassesCache() {
        // 같은 클래스 내 직접 호출 → 캐시 미동작
        // userService.processAll() 내부에서 findById() 직접 호출
        // → 캐시가 아닌 항상 DB 조회됨을 확인
    }
}
```

### 실험 2: 캐시 키 확인

```bash
# Spring 애플리케이션에서 @Cacheable 호출 후
redis-cli KEYS "users::*"
# users::1
# users::2
# users::SimpleKey [Alice,30]

redis-cli TTL "users::1"
# 1800 (30분 TTL 설정 시)

redis-cli GET "users::1"
# JSON으로 저장된 User 객체

# @CacheEvict 실행 후
redis-cli KEYS "users::*"
# 빈 결과 (삭제됨)
```

### 실험 3: @CacheEvict allEntries SCAN vs KEYS 비교

```java
// 테스트: allEntries=true 실행 시 Redis 명령어 확인
@Test
void cacheEvictAllEntriesUsesKeys() {
    // 10,000개 캐시 키 생성
    for (int i = 1; i <= 10000; i++) {
        userService.findById((long) i);
    }
    
    // SLOWLOG 리셋
    redisTemplate.execute((RedisCallback<Object>) conn -> {
        conn.serverCommands().slowLogReset();
        return null;
    });
    
    // allEntries 삭제 실행
    long start = System.currentTimeMillis();
    userService.clearAllUsers();  // @CacheEvict(allEntries=true)
    long elapsed = System.currentTimeMillis() - start;
    
    System.out.println("allEntries 삭제 시간: " + elapsed + "ms");
    
    // SLOWLOG 확인 → KEYS 명령어가 느린지 확인
    // redis-cli SLOWLOG GET 5
}
```

### 실험 4: @Transactional + @CacheEvict 순서 검증

```java
@Test
void transactionalCacheEvictOrder() {
    // 캐시 미리 채움
    User cached = userService.findById(1L);
    
    // 업데이트 (트랜잭션 + 캐시 무효화)
    // 트랜잭션 커밋 전 캐시가 삭제되는지 확인
    // → 테스트 프레임워크로 트랜잭션 커밋 시점 제어
}
```

---

## 📊 성능/비용 비교

```
@Cacheable 오버헤드:

캐시 히트 시:
  CacheInterceptor 오버헤드: ~0.1ms
  Redis GET: ~0.5ms
  총: ~0.6ms (DB 조회 제거)
  
  DB 조회가 50ms라면 → 83배 빠름

캐시 미스 시:
  CacheInterceptor + Redis GET + DB 조회 + Redis SET
  오버헤드: ~1ms 추가 (첫 번째 호출만)

@CacheEvict allEntries vs 특정 키:

  allEntries (10,000개 키):
    KEYS "users::*" → ~5~50ms (키 수에 비례)
    이벤트 루프 블로킹!
  
  특정 키 삭제:
    DEL "users::1" → ~0.1ms
    
키 생성 오버헤드:
  SimpleKeyGenerator: ~0.01ms (무시 가능)
  SpEL 표현식: ~0.1ms (복잡한 표현식 더 느림)
```

---

## ⚖️ 트레이드오프

```
@Cacheable의 일관성 vs 성능:
  TTL 길게: 캐시 히트율 높음, 스테일 데이터 오래 유지
  TTL 짧게: 항상 최신 데이터, DB 조회 빈번
  → 데이터 변경 빈도에 맞는 TTL 설정

@CacheEvict allEntries:
  편리: 패턴 기반 전체 삭제
  위험: KEYS 명령어로 운영 환경 블로킹
  → 소규모(<1,000개): 허용 가능
  → 대규모(>10,000개): 절대 금지, SCAN 방식 사용

@CachePut vs @CacheEvict + @Cacheable:
  @CachePut: Write-Through, 항상 최신 상태 유지
  @CacheEvict: 다음 조회 시 DB 재로드 (Lazy Reload)
  → 읽기 빈번하면 @CachePut (즉시 최신화)
  → 쓰기 후 읽기 드물면 @CacheEvict (간단)
```

---

## 📌 핵심 정리

```
@Cacheable AOP 흐름:
  CacheInterceptor → KeyGenerator → RedisCache.get() → 히트/미스
  미스: 메서드 실행 → RedisCache.put() → 결과 반환
  
  AOP 프록시 주의: 같은 클래스 내 직접 호출 = 캐시 미동작

키 생성:
  SimpleKeyGenerator: "cacheName::파라미터" 형태
  SpEL: @Cacheable(key = "#id") 로 커스텀
  커스텀: KeyGenerator 인터페이스 구현

@CacheEvict:
  특정 키: DEL (O(1), 안전)
  allEntries=true: KEYS 패턴 (운영 환경 위험!)
  → Spring 3.x+ enableCleanup() 또는 SCAN 방식 사용

@CachePut:
  항상 메서드 실행 + 캐시 저장 (Write-Through 패턴)

@Transactional + @CacheEvict:
  캐시 삭제가 트랜잭션 커밋 전 실행됨 → 구버전 데이터 캐시 위험
  → TransactionSynchronization.afterCommit()에서 삭제

condition vs unless:
  condition: 메서드 실행 전 판단 (캐시 전체 비활성화 가능)
  unless: 메서드 실행 후 판단 (반환값으로 저장 여부 결정)
```

---

## 🤔 생각해볼 문제

**Q1.** `@Cacheable`과 `@Transactional`을 동시에 적용한 읽기 메서드에서, 캐시 히트 시 트랜잭션이 시작되는가?

<details>
<summary>해설 보기</summary>

**캐시 히트 시 트랜잭션이 시작되지 않는다.**

AOP 체인 실행 순서: `CacheInterceptor` → `TransactionInterceptor` → 메서드 실행

캐시 히트 시:
1. `CacheInterceptor`가 캐시에서 값을 찾음
2. 값이 있으면 즉시 반환 → **AOP 체인이 더 이상 진행되지 않음**
3. `TransactionInterceptor`가 실행되지 않음 → 트랜잭션 없음

이것은 **성능상 유리**하다. 트랜잭션 시작/종료 오버헤드 없이 캐시에서 빠르게 반환한다.

하지만 주의사항:
- 캐시 히트 시 반환된 객체는 **영속성 컨텍스트(Persistence Context) 밖**에 있다
- Lazy Loading이 필요한 연관 관계가 있는 객체를 캐시에서 꺼냈다가 Lazy Load를 시도하면 → `LazyInitializationException`

해결:
```java
// Lazy Loading 없도록 DTO로 변환 후 캐시
@Cacheable("users")
@Transactional(readOnly = true)
UserDto findById(Long id) {
    User user = userRepository.findById(id)
        .orElseThrow();
    return UserDto.from(user);  // 모든 필드 로드 후 DTO 변환
}
```

</details>

---

**Q2.** `@Cacheable(sync = true)`를 설정하면 무엇이 달라지는가? 모든 CacheManager에서 지원되는가?

<details>
<summary>해설 보기</summary>

**`sync = true`의 동작:**

동일한 캐시 키에 대해 동시에 여러 스레드가 미스를 만났을 때, `sync = true`는 하나의 스레드만 메서드를 실행하고 나머지는 대기하도록 한다 (Cache Stampede 방지).

```java
@Cacheable(value = "products", sync = true)
Product findById(Long id) {
    // 동시에 100 스레드가 미스 발생 → 1개만 실행, 99개는 대기
    return productRepository.findById(id);
}
```

**지원 여부:**
- `RedisCacheManager` (Spring Boot 기본): `sync = true` **제한적 지원**
  - Redis 자체는 분산 락이 없어서 진정한 단일 실행 보장 어려움
  - 로컬 JVM 레벨에서 `ReentrantLock`으로 동기화 (단일 인스턴스에서만 효과)
  - 다중 서버 환경에서는 여전히 여러 서버가 동시에 DB 조회 가능
- `CaffeineCacheManager`: 완전 지원 (JVM 내 동기화)
- `EhCacheCacheManager`: 완전 지원

**실무 권장:**
- 단일 서버: `sync = true` 효과적
- 다중 서버: 분산 락(Redisson RLock) + 수동 Cache-Aside 구현이 더 신뢰성 높음

</details>

---

**Q3.** `@Cacheable` 메서드에서 반환 타입이 `Optional<User>`이다. Redis에는 어떻게 저장되고, 캐시 히트 시 Optional이 그대로 반환되는가?

<details>
<summary>해설 보기</summary>

**Spring Cache의 Optional 처리 (Spring 4.3+):**

```java
@Cacheable("users")
Optional<User> findById(Long id)
```

Spring Cache는 `Optional`을 자동으로 언래핑한다:
- `Optional.of(user)` 반환 → Redis에는 `User` 객체가 저장됨 (Optional 껍데기 없음)
- `Optional.empty()` 반환 → 캐시에 저장됨 (null 캐시 활성화 시)

캐시 히트 시:
- Redis에서 `User` 객체를 읽어 `Optional.of(user)`로 감싸서 반환
- Redis에 null(empty)이 저장됐으면 `Optional.empty()` 반환

**주의사항:**
```java
// disableCachingNullValues() 설정 시
@Cacheable(value = "users")  // unless 미설정
Optional<User> findById(Long id) {
    return userRepository.findById(id);  // 없으면 Optional.empty()
}
// Optional.empty()는 null로 취급 → disableCachingNullValues로 저장 안 됨
// → 다음 조회에서도 항상 DB 조회 (Cache 효과 없음)

// 해결: unless로 명시적 처리
@Cacheable(value = "users", unless = "#result == null || !#result.present")
Optional<User> findById(Long id) { ... }
// Optional.empty() 시 캐시 미저장, Optional.of(user)만 저장
```

</details>

---

<div align="center">

**[⬅️ 이전: RedisTemplate 직렬화](./01-redis-template-serialization.md)** | **[홈으로 🏠](../README.md)** | **[다음: Spring Session ➡️](./03-spring-session-redis.md)**

</div>
