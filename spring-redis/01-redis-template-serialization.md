# Spring Data Redis — RedisTemplate 직렬화 전략

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `JdkSerializationRedisSerializer`, `Jackson2JsonRedisSerializer`, `StringRedisSerializer`의 실제 저장 크기와 성능 차이는?
- `GenericJackson2JsonRedisSerializer`가 타입 정보를 포함하는 이유와 그로 인한 문제는?
- Redis CLI에서 한글이나 객체가 깨져 보이는 이유는?
- `@class` 필드가 JSON에 포함되면 어떤 보안/운영 문제가 생기는가?
- 직렬화 방식 전환 시 기존 캐시 데이터와 호환성 문제를 피하는 방법은?
- 키와 값에 서로 다른 직렬화를 적용하는 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

직렬화 방식은 Redis에 저장되는 데이터 크기, 역직렬화 속도, redis-cli 가독성을 동시에 결정한다. 잘못된 직렬화 선택은 저장 크기를 5~10배 늘리거나, 타입 변경 시 역직렬화 실패로 서비스 장애를 일으킨다. 특히 `JdkSerializationRedisSerializer`(Spring Boot 기본값)는 Java 객체 바이너리를 그대로 저장해 redis-cli에서 읽을 수 없고, 다른 언어 클라이언트와 호환이 안 된다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: JDK 직렬화 기본값 그대로 사용

  코드:
    @Autowired
    RedisTemplate<String, Object> redisTemplate;
    
    redisTemplate.opsForValue().set("user:1", userObject);

  결과:
    redis-cli GET user:1
    → "\xac\xed\x00\x05sr\x00\x1acom.example.domain.User..."
    → 바이너리로 저장됨, redis-cli에서 읽을 수 없음!
    
    저장 크기: 바이너리 직렬화로 JSON 대비 3~5배 큰 경우 많음
    문제: User 클래스가 변경되면 역직렬화 실패 (invalidClassException)
    문제: Java 외 다른 언어 클라이언트에서 읽기 불가

실수 2: GenericJackson2JsonRedisSerializer 사용 시 @class 필드 문제

  코드:
    @Bean
    RedisTemplate<String, Object> redisTemplate() {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }

  저장된 JSON:
    {
      "@class": "com.example.domain.User",
      "id": 1,
      "name": "Alice"
    }
  
  문제:
    패키지 이동 시 (com.example → com.newpackage) → 역직렬화 실패
    보안: @class 필드를 이용한 Jackson Gadget 공격 가능성

실수 3: 직렬화 방식 변경 시 기존 캐시 호환성 무시

  "JSON으로 바꾸면 성능이 좋겠지" → 직렬화 변경 후 배포
  결과: 기존 JDK 직렬화 데이터를 Jackson으로 역직렬화 시도 → 에러
  → 모든 캐시 키에서 오류 발생 → 서비스 장애
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
권장 직렬화 설정:
  키: StringRedisSerializer (문자열 키, redis-cli 가독성)
  값: Jackson2JsonRedisSerializer (JSON, 타입 안전, 가독성)
       또는 GenericJackson2JsonRedisSerializer (다형성 필요 시만)
  
  @Bean
  public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
      RedisTemplate<String, Object> template = new RedisTemplate<>();
      template.setConnectionFactory(factory);
      
      // 키: 문자열
      template.setKeySerializer(new StringRedisSerializer());
      template.setHashKeySerializer(new StringRedisSerializer());
      
      // 값: Jackson JSON
      ObjectMapper mapper = new ObjectMapper();
      mapper.activateDefaultTyping(
          LaissezFaireSubTypeValidator.instance,
          ObjectMapper.DefaultTyping.NON_FINAL,
          JsonTypeInfo.As.PROPERTY
      );
      Jackson2JsonRedisSerializer<Object> serializer =
          new Jackson2JsonRedisSerializer<>(mapper, Object.class);
      
      template.setValueSerializer(serializer);
      template.setHashValueSerializer(serializer);
      
      template.afterPropertiesSet();
      return template;
  }

직렬화 전환 시 안전한 방법:
  1. 새 직렬화로 전환 전 FLUSHDB (캐시 전용 Redis)
  2. 또는 키 버전 추가: "user:v2:1" vs "user:1"
  3. 전환 중 양쪽 직렬화로 읽기 시도 (Graceful Migration)
```

---

## 🔬 내부 동작 원리

### 1. RedisTemplate 직렬화 계층

```
RedisTemplate<K, V>의 직렬화 설정:

  setKeySerializer()      → 키 직렬화 (모든 ops에서 키 변환)
  setValueSerializer()    → opsForValue() 의 값 직렬화
  setHashKeySerializer()  → opsForHash() 의 field 직렬화
  setHashValueSerializer()→ opsForHash() 의 value 직렬화

직렬화 흐름:
  Java 객체 (User userObj)
       ↓ valueSerializer.serialize(userObj)
  byte[] bytes
       ↓ Redis 명령어 실행 (SET key bytes)
  Redis에 저장

역직렬화 흐름:
  Redis에서 읽기 (GET key) → byte[] bytes
       ↓ valueSerializer.deserialize(bytes)
  Java 객체 (User userObj)
```

### 2. JdkSerializationRedisSerializer — 기본값의 함정

```
저장 형식:
  Java ObjectOutputStream으로 직렬화된 바이너리
  Java 전용 형식

실제 저장 내용 (redis-cli):
  GET user:1
  "\xac\xed\x00\x05sr\x00\x1acom.example.demo.User\xa7\xc5\xb5\x..."
  → 완전히 읽을 수 없는 바이너리

크기 문제:
  User { id=1, name="Alice", age=30 }
  JDK 직렬화: ~300~500 bytes (클래스 메타데이터 포함)
  JSON: ~40 bytes
  → 7~12배 크기 차이!

역직렬화 실패 시나리오:
  User 클래스에 필드 추가 (serialVersionUID 미설정)
    → InvalidClassException: local class incompatible
  패키지 이동
    → ClassNotFoundException

serialVersionUID 설정으로 완화:
  @Serial
  private static final long serialVersionUID = 1L;
  → 필드 추가/제거 시에도 역직렬화 가능 (null/기본값)
  → 하지만 패키지 변경은 여전히 실패

Spring Boot 기본값 주의:
  RedisTemplate<Object, Object> → JdkSerializationRedisSerializer 기본 사용
  StringRedisTemplate → StringRedisSerializer 사용 (String 전용)
```

### 3. Jackson2JsonRedisSerializer — JSON 직렬화

```
저장 형식:
  Jackson ObjectMapper로 JSON 직렬화
  사람이 읽을 수 있는 텍스트

예시:
  User { id=1, name="Alice", age=30 }
  → {"id":1,"name":"Alice","age":30}

redis-cli GET user:1
  → "{\"id\":1,\"name\":\"Alice\",\"age\":30}"

특성:
  타입 정보 없음 (순수 JSON)
  → 역직렬화 시 목표 타입을 명시해야 함

두 가지 방식:
  Jackson2JsonRedisSerializer<User> serializer =
      new Jackson2JsonRedisSerializer<>(User.class);
  → User 타입으로만 역직렬화 가능 (타입 안전)
  
  Jackson2JsonRedisSerializer<Object> serializer =
      new Jackson2JsonRedisSerializer<>(Object.class);
  → LinkedHashMap으로 역직렬화됨 (타입 정보 없어서)
  → User userObj = (User) template.opsForValue().get(key) → ClassCastException!

올바른 사용:
  // 타입 지정 직렬화
  Jackson2JsonRedisSerializer<User> userSerializer =
      new Jackson2JsonRedisSerializer<>(User.class);
  
  // 제네릭 타입 처리
  JavaType type = mapper.getTypeFactory()
      .constructParametricType(List.class, User.class);
  Jackson2JsonRedisSerializer<List<User>> listSerializer =
      new Jackson2JsonRedisSerializer<>(type);
```

### 4. GenericJackson2JsonRedisSerializer — @class 타입 정보

```
저장 형식:
  @class 타입 정보를 JSON에 포함

예시:
  {
    "@class": "com.example.demo.User",
    "id": 1,
    "name": "Alice",
    "age": 30
  }

장점:
  다형성(Polymorphism) 지원
  Object 타입으로 저장/조회 시 타입 보존
  → Animal 인터페이스를 Dog, Cat으로 저장하고 그대로 꺼내기 가능

단점 1: 패키지 이동 시 역직렬화 실패
  com.example.demo.User → com.newpackage.User 이동 시
  기존 캐시에는 com.example.demo.User가 기록됨
  → ClassNotFoundException 발생

단점 2: 보안 위험 (Jackson Polymorphic Deserialization)
  @class를 악용한 Gadget 공격 가능성
  신뢰할 수 없는 입력에서 @class를 통한 임의 클래스 인스턴스화
  → 안전한 Validator 등록 필요:
  
  PolymorphicTypeValidator ptv = BasicPolymorphicTypeValidator
      .builder()
      .allowIfSubType("com.example.")  // 허용 패키지만
      .build();
  mapper.activateDefaultTyping(ptv, ObjectMapper.DefaultTyping.NON_FINAL);

타입 레지스트리로 안전하게 사용:
  // 명시적 타입 매핑 (패키지명 대신 짧은 이름)
  ObjectMapper mapper = new ObjectMapper();
  mapper.registerSubtypes(new NamedType(User.class, "User"));
  mapper.registerSubtypes(new NamedType(Admin.class, "Admin"));
  // → "@class": "User" (패키지 없음, 이동해도 안전)
```

### 5. 직렬화 방식별 성능 비교

```
벤치마크 시나리오: User 객체 1,000번 직렬화/역직렬화

User { id, name(10자), email(20자), age, createdAt }

직렬화:
  JDK:     ~450 bytes, ~85 μs/op
  Jackson: ~120 bytes, ~15 μs/op (5배 빠름)
  Gson:    ~120 bytes, ~20 μs/op

역직렬화:
  JDK:     ~85 μs/op (ObjectInputStream 오버헤드)
  Jackson: ~12 μs/op (7배 빠름)
  Gson:    ~18 μs/op

네트워크 영향:
  1MB/초 대역폭 환경에서 User 객체 10,000개 전송:
  JDK:     4,500 MB → 4.5초
  Jackson: 1,200 MB → 1.2초
  → 3.75배 차이

Redis 메모리 영향:
  User 객체 100만개 저장:
  JDK:     ~450 MB
  Jackson: ~120 MB
  → 330 MB 절약 (인스턴스 비용 절감)
```

### 6. 실전 직렬화 설정 패턴

```java
// 권장 설정 (Spring Boot 3.x)
@Configuration
public class RedisConfig {
    
    @Bean
    public RedisTemplate<String, Object> redisTemplate(
            RedisConnectionFactory connectionFactory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);
        
        // 키: String (redis-cli 가독성 보장)
        StringRedisSerializer stringSerializer = new StringRedisSerializer();
        template.setKeySerializer(stringSerializer);
        template.setHashKeySerializer(stringSerializer);
        
        // 값: Jackson JSON (타입 안전, 가독성)
        Jackson2JsonRedisSerializer<Object> jacksonSerializer = 
            jackson2JsonRedisSerializer();
        template.setValueSerializer(jacksonSerializer);
        template.setHashValueSerializer(jacksonSerializer);
        
        template.afterPropertiesSet();
        return template;
    }
    
    private Jackson2JsonRedisSerializer<Object> jackson2JsonRedisSerializer() {
        ObjectMapper mapper = new ObjectMapper();
        // Java 8 날짜/시간 지원
        mapper.registerModule(new JavaTimeModule());
        mapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
        // null 필드 제외
        mapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);
        
        // 타입 정보 포함 (다형성이 필요한 경우)
        // 필요 없으면 이 설정 제거 → @class 없는 순수 JSON
        PolymorphicTypeValidator ptv = BasicPolymorphicTypeValidator
            .builder()
            .allowIfSubType("com.example.")
            .build();
        mapper.activateDefaultTyping(
            ptv,
            ObjectMapper.DefaultTyping.NON_FINAL,
            JsonTypeInfo.As.PROPERTY
        );
        
        return new Jackson2JsonRedisSerializer<>(mapper, Object.class);
    }
    
    // RedisCacheManager 설정
    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(30))
            .serializeKeysWith(
                RedisSerializationContext.SerializationPair
                    .fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair
                    .fromSerializer(jackson2JsonRedisSerializer()))
            .disableCachingNullValues();
        
        return RedisCacheManager.builder(factory)
            .cacheDefaults(config)
            .build();
    }
}
```

---

## 💻 실전 실험

### 실험 1: 직렬화별 저장 크기 비교

```bash
# Redis에서 직접 확인 (Spring 애플리케이션 실행 후)
# JDK 직렬화로 저장된 데이터 확인
redis-cli GET "jdk:user:1"
# → 바이너리 출력 (깨진 문자)

# JSON 직렬화로 저장된 데이터 확인
redis-cli GET "json:user:1"
# → {"id":1,"name":"Alice","age":30}

# 크기 비교
redis-cli STRLEN "jdk:user:1"   # 예: 350 bytes
redis-cli STRLEN "json:user:1"  # 예: 120 bytes
redis-cli MEMORY USAGE "jdk:user:1"
redis-cli MEMORY USAGE "json:user:1"
```

```java
// Spring 테스트 코드
@SpringBootTest
class SerializationTest {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Autowired
    private StringRedisTemplate stringRedisTemplate;
    
    @Test
    void compareSerializationSize() {
        User user = new User(1L, "Alice", "alice@example.com", 30);
        
        // JDK 직렬화
        RedisTemplate<String, Object> jdkTemplate = new RedisTemplate<>();
        jdkTemplate.setConnectionFactory(redisTemplate.getConnectionFactory());
        jdkTemplate.afterPropertiesSet();
        jdkTemplate.opsForValue().set("jdk:user:1", user);
        
        // JSON 직렬화 (설정된 redisTemplate)
        redisTemplate.opsForValue().set("json:user:1", user);
        
        Long jdkSize = redisTemplate.execute(
            conn -> conn.stringCommands().strLen("jdk:user:1".getBytes()));
        Long jsonSize = redisTemplate.execute(
            conn -> conn.stringCommands().strLen("json:user:1".getBytes()));
        
        System.out.println("JDK size: " + jdkSize);
        System.out.println("JSON size: " + jsonSize);
        System.out.printf("비율: %.1fx%n", (double) jdkSize / jsonSize);
    }
}
```

### 실험 2: GenericJackson2JsonRedisSerializer @class 확인

```bash
# Spring 애플리케이션에서 GenericJackson2JsonRedisSerializer로 저장 후
redis-cli GET "generic:user:1"
# 출력: {"@class":"com.example.demo.User","id":1,"name":"Alice","age":30}
# @class 필드가 포함됨!

# @class 없는 Jackson2JsonRedisSerializer
redis-cli GET "typed:user:1"
# 출력: {"id":1,"name":"Alice","age":30}
# 더 깔끔한 JSON
```

### 실험 3: 직렬화 전환 시 호환성 테스트

```java
@Test
void serializationMigration() {
    // 기존: JDK 직렬화로 저장
    RedisTemplate<String, Object> jdkTemplate = createJdkTemplate();
    jdkTemplate.opsForValue().set("user:1", new User(1L, "Alice"));
    
    // 새: JSON 직렬화로 읽기 시도
    RedisTemplate<String, Object> jsonTemplate = createJsonTemplate();
    
    assertThatThrownBy(() -> jsonTemplate.opsForValue().get("user:1"))
        .isInstanceOf(SerializationException.class);
    // → 직렬화 방식이 다르면 호환 안 됨!
    
    // 마이그레이션 전략: 키 삭제 후 재저장
    jdkTemplate.delete("user:1");
    jsonTemplate.opsForValue().set("user:1", new User(1L, "Alice"));
    
    User result = (User) jsonTemplate.opsForValue().get("user:1");
    assertThat(result.getName()).isEqualTo("Alice");
}
```

---

## 📊 성능/비용 비교

```
직렬화 방식별 종합 비교:

                      JDK         Jackson JSON  GenericJackson  String
────────────────────┼───────────┼─────────────┼───────────────┼──────
저장 크기             │ 크다       │ 작다         │ 약간 더 큼      │ 최소
직렬화 속도            │ 느리다     │ 빠르다        │ 빠르다         │ 빠르다
역직렬화 속도          │ 느리다     │ 빠르다        │ 빠르다         │ 빠르다
redis-cli 가독성     │ 없음       │ 있음         │ 있음           │ 최고
타입 안전성           │ 높음       │ 중간         │ 높음           │ 없음
다형성 지원           │ 있음       │ 없음         │ 있음           │ 없음
패키지 리팩토링        │ 실패       │ 안전         │ 실패(기본설정)    │ 해당없음
다른 언어 호환        │ 없음       │ 있음         │ 있음            │ 있음
권장                │ ✗         │ ✓✓          │ 신중히          │ 문자열만

실제 User 객체 저장 크기 예시:
  JDK:              ~380 bytes
  Jackson JSON:      ~95 bytes (4배 작음)
  GenericJackson:   ~130 bytes (@class 포함)
  String (직렬화 후): ~95 bytes
```

---

## ⚖️ 트레이드오프

```
JDK 직렬화 vs JSON:
  JDK: Java 객체 그대로, 컬렉션/제네릭 자동, 하지만 크기 크고 느리고 Java만
  JSON: 가볍고 빠르고 언어 중립, 하지만 타입 지정 필요

GenericJackson2Json의 @class:
  다형성 지원 → 편리, 하지만 패키지명 변경 시 실패
  → 사용 조건: 다형성이 반드시 필요하고, 패키지 리팩토링 없을 것이 확실한 경우
  → 대안: 명시적 타입별 직렬화, 별도 캐시 키 관리

타입 정보 포함 여부:
  포함: 다형성 지원, 저장 크기 약간 증가, 클래스 구조 변경에 취약
  미포함: 가볍고 안전, 역직렬화 시 목표 타입 명시 필요

직렬화 전환 비용:
  캐시는 재생성 가능 → 전환 시 전체 삭제 후 재시작이 가장 안전
  세션 등 삭제 불가 → 전환 자체가 불가 (신중한 설계 필요)
```

---

## 📌 핵심 정리

```
직렬화 전략 핵심:

JdkSerializationRedisSerializer (기본값):
  금지 수준 — 크기 크고, 느리고, Java만, redis-cli 불가
  Legacy 코드에서 교체 권장

Jackson2JsonRedisSerializer<T>:
  권장 — 가볍고, 빠르고, 가독성, 타입 안전
  특정 타입 지정 (Jackson2JsonRedisSerializer<User>)

GenericJackson2JsonRedisSerializer:
  @class 포함 → 다형성 지원
  신중하게 — 패키지 리팩토링 시 실패 위험
  PolymorphicTypeValidator로 허용 클래스 제한

StringRedisSerializer:
  키 직렬화의 표준 (항상 사용)
  문자열 값에만 사용 (redis-cli 최대 호환)

권장 조합:
  키:  StringRedisSerializer
  값:  Jackson2JsonRedisSerializer (다형성 불필요 시)
       GenericJackson2JsonRedisSerializer (다형성 필요 시 + 보안 설정)

직렬화 전환:
  캐시: FLUSHDB 후 전환 (재시작 시 자동 재적재)
  세션/중요 데이터: 키 버전 관리 또는 점진적 마이그레이션
```

---

## 🤔 생각해볼 문제

**Q1.** `StringRedisTemplate`과 일반 `RedisTemplate<String, Object>`의 차이는? 어떤 상황에서 각각을 사용해야 하는가?

<details>
<summary>해설 보기</summary>

**StringRedisTemplate:**
- `RedisTemplate<String, String>`의 편의 클래스
- 키와 값 모두 `StringRedisSerializer` 사용
- 문자열 데이터만 저장/조회 가능

**RedisTemplate<String, Object>:**
- 키는 String, 값은 설정된 직렬화 방식 사용
- 복잡한 객체 저장 가능

**사용 시나리오:**

```java
// StringRedisTemplate: 단순 문자열 저장 (카운터, 플래그, 토큰 등)
@Autowired
StringRedisTemplate stringTemplate;

stringTemplate.opsForValue().set("token:abc", "user:1");
stringTemplate.opsForValue().increment("counter:views");

// RedisTemplate<String, Object>: 복잡한 객체 저장
@Autowired
RedisTemplate<String, Object> objectTemplate;

objectTemplate.opsForValue().set("user:1", new User(1L, "Alice"));
User user = (User) objectTemplate.opsForValue().get("user:1");

// 주의: StringRedisTemplate로 저장한 키를 RedisTemplate으로 읽으면?
stringTemplate.opsForValue().set("key", "hello");
Object val = objectTemplate.opsForValue().get("key");
// → StringRedisSerializer로 저장된 "hello"를 Jackson이 역직렬화 시도
// → 오류 가능성 (직렬화 불일치)
```

**권장:** 동일한 키에 대해 항상 동일한 직렬화 방식을 사용한다. 직렬화 방식을 혼용하면 예측 불가능한 오류가 발생한다.

</details>

---

**Q2.** `RedisCacheConfiguration.defaultCacheConfig().disableCachingNullValues()`를 설정하는 이유는? null 캐싱이 허용되면 어떤 문제가 생기는가?

<details>
<summary>해설 보기</summary>

**null 캐싱이 허용되면:**

```java
@Cacheable("users")
public User findById(Long id) {
    return userRepository.findById(id).orElse(null);  // 없으면 null
}
```

존재하지 않는 ID로 조회 → DB에서 null 반환 → **null이 캐시에 저장됨**

이후 동일 ID 조회 → **캐시에서 null 반환** → DB 조회 없음 → **정상처럼 보이지만 사실 null**

**null 캐싱이 유용한 경우 (Cache Negative):**
- DB에 없는 키를 반복 조회하는 패턴 방지 (Cache Stampede 변형)
- 없는 ID 1,000번 조회 → DB에 1,000번 쿼리 → null 캐싱으로 DB 보호

**`disableCachingNullValues()` 설정 시:**
- null 반환 값은 캐시에 저장되지 않음
- 다음 조회에서 다시 DB 조회 실행
- DB에 생성되면 자연스럽게 캐시에 적재

**실무 권장:**
- null 캐싱이 필요한 경우: 기본값 그대로 (null 허용)
- null 반환은 캐시하면 안 되는 경우: `disableCachingNullValues()` 설정
- Optional을 반환하는 메서드에 `@Cacheable` 주의 (Optional<null>은 캐시됨)

</details>

---

**Q3.** JSON 직렬화로 저장된 캐시에서 `User` 클래스에 새 필드 `phone`을 추가했다. 기존 캐시 데이터는 `phone` 없이 저장됐다. 이 데이터를 역직렬화하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

**기본 Jackson 동작:** `phone` 필드가 없는 JSON을 `User`로 역직렬화하면 `phone`은 `null`이 된다. **오류 없음.**

```java
// 기존 캐시 JSON: {"id":1,"name":"Alice","age":30}
// 새 User 클래스: id, name, age, phone (추가)

User user = jackson.readValue(existingJson, User.class);
// user.getPhone() == null (오류 없이 null)
```

**반대 경우 (필드 삭제):**
```java
// 기존 캐시 JSON: {"id":1,"name":"Alice","age":30,"phone":"010-1234"}
// 새 User 클래스에서 phone 삭제

// DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES = true (기본):
// → UnrecognizedPropertyException 발생!

// 해결: Jackson 설정
mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
// 또는 클래스 레벨
@JsonIgnoreProperties(ignoreUnknown = true)
public class User { ... }
```

**실무 권장:**
```java
ObjectMapper mapper = new ObjectMapper();
mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
// → 필드 추가/삭제에 유연하게 대응
```

JSON 직렬화는 JDK 직렬화보다 스키마 진화(Schema Evolution)에 훨씬 관대하다. 이것이 JSON 직렬화를 권장하는 또 다른 이유다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: Spring Cache @Cacheable ➡️](./02-spring-cache-cacheable.md)**

</div>
