# Spring Session — HttpSession을 Redis에

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@EnableRedisHttpSession`이 등록하는 `SessionRepositoryFilter`는 어떻게 `HttpSession`을 가로채는가?
- 세션 데이터는 Redis에서 어떤 자료구조와 키 형식으로 저장되는가?
- TTL과 세션 만료 이벤트(`SessionExpiredEvent`)는 어떻게 처리되는가?
- Redis Cluster 환경에서 세션 키가 여러 노드에 분산될 때 생기는 문제와 해결책은?
- Spring Session의 직렬화 방식을 변경하는 방법과 기존 세션과의 호환성은?
- `maxInactiveIntervalInSeconds` vs Redis TTL의 관계는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

다중 서버 환경에서 로드밸런서가 각 요청을 다른 서버로 라우팅하면 서버 로컬 세션이 유실된다. Spring Session + Redis는 이를 해결하지만, 세션 직렬화 방식, 만료 처리, Cluster 환경에서의 키 배치 문제를 모르면 운영 중 세션이 갑자기 끊기거나 특정 노드에 부하가 집중되는 문제가 발생한다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Sticky Session으로 Redis Session을 대체하려 함

  로드밸런서: IP Hash로 항상 같은 서버로 요청 보냄
  → "세션 문제 없다" 생각
  
  문제:
    서버 재배포 시 해당 서버의 세션 전부 소실
    서버 장애 시 해당 서버 사용자 전원 로그아웃
    서버별 부하 불균형 (인기 있는 IP 범위가 특정 서버 집중)
  
  Redis Session의 장점:
    세션이 Redis에 있어 서버는 완전 무상태(Stateless)
    어떤 서버에 요청이 가도 동일한 세션 접근
    서버 재배포/장애에도 세션 유지

실수 2: 세션 만료 이벤트를 TTL에만 의존

  설정:
    @EnableRedisHttpSession(maxInactiveIntervalInSeconds = 1800)
  
  기대: 30분 비활동 후 세션 자동 만료 + 정리
  실제:
    Redis TTL로 세션 키 만료 → 자동 삭제 OK
    하지만 SessionExpiredEvent 발생 안 함 (TTL 만료는 이벤트 없음)
    세션 만료 시 로그아웃 처리, 감사 로그 기록 등이 동작 안 함

실수 3: Redis Cluster에서 세션 관련 멀티키 명령어 오류

  상황: Redis Cluster + Spring Session
  오류: 특정 요청에서 CROSSSLOT 오류 발생
  원인:
    Spring Session이 세션 조회 시 HMGET, EXPIRE 등을 실행
    세션 관련 여러 키가 다른 슬롯에 배치될 경우 크로스슬롯 오류
  해결: {hash-tag}를 세션 ID에 적용해 같은 슬롯에 배치
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
기본 설정:
  // build.gradle
  implementation 'org.springframework.session:spring-session-data-redis'
  
  // application.yml
  spring:
    session:
      store-type: redis
      redis:
        flush-mode: on-save       # 세션 저장 시점 (on-save/immediate)
        namespace: spring:session  # Redis 키 네임스페이스
      timeout: 30m                # 세션 타임아웃

  // Config 클래스
  @EnableRedisHttpSession(maxInactiveIntervalInSeconds = 1800)

세션 만료 이벤트 활성화:
  // Redis Keyspace Notification 활성화
  redis-cli CONFIG SET notify-keyspace-events "Kgxe"
  // K=Keyspace, g=Generic, x=Expired, e=Evicted
  
  @Configuration
  public class SessionConfig {
      @Bean
      public SessionEventHttpSessionListenerAdapter sessionEventAdapter() {
          return new SessionEventHttpSessionListenerAdapter(/* listeners */);
      }
  }
  
  @Component
  public class SessionExpiredListener implements ApplicationListener<SessionExpiredEvent> {
      @Override
      public void onApplicationEvent(SessionExpiredEvent event) {
          String sessionId = event.getSessionId();
          // 세션 만료 처리: 감사 로그, 리소스 정리 등
      }
  }

Redis Cluster 환경:
  @EnableRedisHttpSession에서 Hash Tag 설정 불가 (Spring Session 한계)
  → 모든 세션 키를 동일 슬롯에 배치: 비권장 (노드 편중)
  → 권장: Redis Cluster 대신 Redis Sentinel 사용 (세션 서버에 적합)
  → 또는 세션 전용 단일 Redis 인스턴스 분리
```

---

## 🔬 내부 동작 원리

### 1. SessionRepositoryFilter — HttpSession 가로채기

```
@EnableRedisHttpSession 동작:
  Spring Boot 자동 설정 활성화 또는 @Configuration에 직접 선언
  → RedisOperationsSessionRepository 빈 생성
  → SessionRepositoryFilter<RedisSession> 빈 생성
  → Filter Chain에 최상위로 등록 (ORDER = Integer.MIN_VALUE + 50)

요청 처리 흐름:

  HTTP 요청
       ↓
  SessionRepositoryFilter.doFilter()
       ↓
  SessionRepositoryRequestWrapper 생성
  (HttpServletRequest를 래핑 — getSession() 오버라이드)
       ↓
  다음 필터/서블릿으로 전달
       ↓
  애플리케이션 코드: request.getSession()
       ↓
  SessionRepositoryRequestWrapper.getSession() 실행
  (HttpServletRequest의 getSession()이 아닌 오버라이드된 메서드!)
       ↓
  쿠키에서 SESSION ID 추출 (기본: "SESSION" 쿠키)
       ↓
  RedisOperationsSessionRepository.findById(sessionId)
  → Redis HGETALL spring:session:sessions:{sessionId}
       ↓
  RedisSession 객체 반환 (Redis에서 읽은 세션)
  
  응답 완료 시:
  SessionRepositoryResponseWrapper가 세션 변경 감지
  → 변경이 있으면 Redis에 저장

쿠키 전략:
  기본: "SESSION" 이름의 쿠키, HttpOnly, SameSite=Lax
  커스텀:
    @Bean
    public CookieSerializer cookieSerializer() {
        DefaultCookieSerializer serializer = new DefaultCookieSerializer();
        serializer.setCookieName("JSESSIONID");  // 기존 이름 유지
        serializer.setDomainNamePattern("^.+?\\.(\\w+\\.[a-z]+)$");
        serializer.setUseSecureCookie(true);
        serializer.setSameSite("Strict");
        return serializer;
    }
```

### 2. Redis 저장 구조

```
Spring Session의 Redis 저장 키 구조:

세션 데이터 (Hash):
  spring:session:sessions:{sessionId}
  → HSET 명령어로 Hash에 저장
  
  Hash 필드:
    creationTime         → Long (Unix timestamp ms)
    lastAccessedTime     → Long (마지막 접근 시각)
    maxInactiveInterval  → int (초, 비활동 만료 시간)
    sessionAttr:{key}    → 직렬화된 세션 속성값
  
  예시:
  redis-cli HGETALL "spring:session:sessions:abc123"
  # 1) "creationTime"
  # 2) "\xac\xed..." (JDK 직렬화 기본값)
  # 3) "lastAccessedTime"
  # 4) "\xac\xed..."
  # 5) "sessionAttr:userId"
  # 6) "\xac\xed..."

세션 만료 추적:
  spring:session:sessions:expires:{sessionId}
  → String 타입, 실제 TTL 보유 키 (만료 이벤트 감지용)
  
  spring:session:expirations:{expiration_rounded}
  → Set 타입, 특정 시각에 만료될 세션 ID 집합
  (배경 스레드가 주기적으로 이 Set을 체크)

두 개의 만료 메커니즘:
  1. Redis TTL: sessions:expires:{id} 키가 TTL 만료 시 삭제
     → Redis Keyspace Notification으로 감지 → SessionExpiredEvent 발생
  2. expirations Set: SessionExpirationPolicy가 주기적으로 만료 세션 정리
     → Keyspace Notification 없이도 만료 처리 (폴백)

maxInactiveIntervalInSeconds vs TTL 관계:
  maxInactiveInterval = 1800 (30분)
  
  Redis TTL:
    sessions:{id}: 1800초 TTL (데이터 보관)
    sessions:expires:{id}: 1800초 TTL (만료 이벤트 트리거용)
  
  lastAccessedTime 갱신 시:
    → TTL이 1800초로 리셋 (슬라이딩 만료)
    → 활동하는 동안 만료 안 됨
```

### 3. 세션 직렬화 전략

```java
// 기본값: JdkSerializationRedisSerializer (JDK 직렬화)
// → 세션 속성이 바이너리로 저장됨
// → redis-cli에서 읽을 수 없음
// → 클래스 변경 시 역직렬화 실패 위험

// JSON 직렬화로 변경:
@Bean
public RedisSerializer<Object> springSessionDefaultRedisSerializer() {
    // 이 이름의 빈을 등록하면 Spring Session이 자동으로 사용
    return new GenericJackson2JsonRedisSerializer();
}

// 더 세밀한 제어:
@Bean
public RedisSessionRepository redisSessionRepository(
        RedisConnectionFactory factory) {
    RedisTemplate<String, Object> template = new RedisTemplate<>();
    template.setConnectionFactory(factory);
    template.setKeySerializer(new StringRedisSerializer());
    template.setHashKeySerializer(new StringRedisSerializer());
    template.setHashValueSerializer(
        new GenericJackson2JsonRedisSerializer());
    template.afterPropertiesSet();
    
    return new RedisIndexedSessionRepository(template);
}

// 직렬화 후 Redis에서 보이는 내용 (JSON):
// spring:session:sessions:abc123 →
// {
//   "creationTime": 1711234567000,
//   "lastAccessedTime": 1711234580000,
//   "maxInactiveInterval": 1800,
//   "sessionAttr:userId": 1,
//   "sessionAttr:userName": "Alice"
// }
```

### 4. 세션 만료 이벤트 처리

```
만료 이벤트 흐름:

  세션 비활동 1800초 경과
       ↓
  Redis TTL 만료: spring:session:sessions:expires:{sessionId} 삭제
       ↓
  Redis Keyspace Notification 발생
  (notify-keyspace-events: "Kgxe" 설정 필요)
       ↓
  RedisOperationsSessionRepository의 MessageListener가 수신
       ↓
  SessionDeletedEvent 또는 SessionExpiredEvent 발생
       ↓
  ApplicationListener<SessionExpiredEvent> 호출

필수 Redis 설정:
  notify-keyspace-events "Kgxe"
  
  K: Keyspace 이벤트 (키 이름 포함)
  g: Generic 명령어 (DEL, EXPIRE 등)
  x: TTL 만료 이벤트 (expired)
  e: Eviction 이벤트 (maxmemory-policy로 제거)

Spring Boot 자동 설정:
  spring.session.redis.configure-action=notify-keyspace-events
  # 또는
  spring:
    session:
      redis:
        configure-action: NOTIFY_KEYSPACE_EVENTS  # Spring이 자동 설정

세션 만료 이벤트 리스너:
@Component
public class SessionExpiredListener 
    implements ApplicationListener<SessionExpiredEvent> {
    
    @Override
    public void onApplicationEvent(SessionExpiredEvent event) {
        String sessionId = event.getSessionId();
        
        // 만료된 세션에서 속성 읽기 (이미 삭제됐으므로 null 가능)
        Session session = event.getSession();
        if (session != null) {
            Long userId = (Long) session.getAttribute("userId");
            // 감사 로그: 사용자 {userId}의 세션 {sessionId} 만료
            auditLogger.logSessionExpired(userId, sessionId);
        }
    }
}

주의사항:
  SessionExpiredEvent 발생 시점: TTL 만료 후 Keyspace Notification 처리 시
  → 세션 데이터는 이미 Redis에서 삭제됨
  → event.getSession()으로 속성 접근 시 null 또는 빈 세션 가능
  → 중요 속성은 별도 저장 또는 만료 직전에 처리 필요
```

### 5. Redis Cluster 환경에서의 세션

```
문제: 세션 관련 키들이 다른 슬롯에 배치

Spring Session이 단일 세션에 생성하는 키들:
  spring:session:sessions:{sessionId}         → 슬롯 A
  spring:session:sessions:expires:{sessionId} → 슬롯 B (다를 수 있음!)
  spring:session:expirations:{timestamp}      → 슬롯 C
  
  → CROSSSLOT 오류 발생 가능

해결책 1: 세션 전용 Redis 단일 인스턴스 (권장)
  Redis Cluster는 캐시 등 다른 용도로 사용
  세션 전용 단일 Redis Master (또는 Sentinel)
  
  application.yml:
    spring:
      session:
        store-type: redis
      # 별도 Redis 연결 설정
      data:
        redis:
          host: session-redis.internal
          port: 6379

해결책 2: Hash Tag 적용 (Spring Session 기본 지원 없음)
  Spring Session 4.x에서 일부 Hash Tag 지원 시작
  이전 버전: 커스텀 구현 필요
  
  커스텀 SessionIdGenerator로 {prefix} 포함:
    @Bean
    public SessionIdGenerator sessionIdGenerator() {
        return () -> "{session}:" + UUID.randomUUID().toString().replace("-", "");
    }
  → 모든 세션 ID가 같은 슬롯 (부하 집중 주의)

해결책 3: Redis Sentinel 사용 (실무 권장)
  세션은 Sentinel, 캐시는 Cluster로 분리
  Sentinel: 고가용성 단일 Redis, Cluster 복잡도 없음
```

### 6. flush-mode 설정

```
spring.session.redis.flush-mode:

on-save (기본값):
  세션 속성 변경 시 응답 완료 후 일괄 저장
  → 성능 우선 (Redis 쓰기 횟수 최소화)
  → 응답 중 서버 크래시 시 세션 변경사항 손실 가능

immediate:
  세션 속성 변경 즉시 Redis에 반영 (setAttribute 호출마다)
  → 강한 내구성 (크래시 시에도 세션 보존)
  → Redis 쓰기 횟수 증가 (성능 저하 가능)

선택 기준:
  온라인 쇼핑몰 장바구니: immediate (결제 직전 크래시 시 장바구니 보존)
  일반 사용자 세션: on-save (성능 우선, 세션 재로그인 정도 허용)
```

---

## 💻 실전 실험

### 실험 1: Redis에 저장된 세션 데이터 확인

```bash
# 서버에서 로그인 후 세션 ID 확인 (쿠키에서)
SESSION_ID="abc123def456..."  # 실제 세션 ID

# 세션 데이터 조회
redis-cli HGETALL "spring:session:sessions:$SESSION_ID"
# Hash 필드와 값 출력

# 세션 TTL 확인
redis-cli TTL "spring:session:sessions:$SESSION_ID"
redis-cli TTL "spring:session:sessions:expires:$SESSION_ID"

# 세션 만료 트리거 키 확인
redis-cli KEYS "spring:session:expirations:*"

# 세션 속성 직접 확인 (JSON 직렬화 시)
redis-cli HGET "spring:session:sessions:$SESSION_ID" "sessionAttr:userId"

# 모든 세션 키 확인 (운영 환경 주의!)
redis-cli SCAN 0 MATCH "spring:session:sessions:*" COUNT 10
```

### 실험 2: 세션 만료 이벤트 설정 확인

```bash
# Keyspace Notification 설정 확인
redis-cli CONFIG GET notify-keyspace-events
# "Kgxe" 이어야 함

# 없다면 설정
redis-cli CONFIG SET notify-keyspace-events "Kgxe"

# 세션 만료 이벤트 구독 (별도 터미널)
redis-cli SUBSCRIBE "__keyevent@0__:expired"

# 다른 터미널에서 세션 TTL 강제 만료
redis-cli EXPIRE "spring:session:sessions:expires:$SESSION_ID" 1
sleep 2

# 구독 터미널에서 만료 이벤트 수신 확인
# message
# __keyevent@0__:expired
# spring:session:sessions:expires:abc123...
```

### 실험 3: 세션 속성 성능 비교

```java
@RestController
public class SessionTestController {
    
    @GetMapping("/session/set")
    public String setSession(HttpSession session) {
        // 다양한 크기의 세션 데이터 저장
        session.setAttribute("small", "hello");  // 5 bytes
        session.setAttribute("medium", generateString(1000));  // 1KB
        session.setAttribute("large", generateString(100000));  // 100KB
        return "OK";
    }
    
    @GetMapping("/session/get")
    public String getSession(HttpSession session) {
        return (String) session.getAttribute("medium");
    }
}

// Redis에서 세션 크기 확인
// redis-cli MEMORY USAGE "spring:session:sessions:{id}"
// 작은 세션: ~2KB
// 큰 세션: ~100KB+
// → 세션에 대용량 데이터 저장 주의
```

### 실험 4: Redis Cluster에서 세션 키 분포 확인

```bash
# Cluster 환경에서 세션 키 슬롯 확인
SESSION_ID="abc123"

redis-cli CLUSTER KEYSLOT "spring:session:sessions:$SESSION_ID"
redis-cli CLUSTER KEYSLOT "spring:session:sessions:expires:$SESSION_ID"
redis-cli CLUSTER KEYSLOT "spring:session:expirations:1234567890"

# 세 키의 슬롯이 다를 수 있음 → CROSSSLOT 잠재적 문제
# → Redis Sentinel 사용 권장
```

---

## 📊 성능/비용 비교

```
세션 저장 방식별 비교:

방식             | 서버 재배포  | 장애 복구  | 서버 간 세션 공유   | 비용
────────────────┼───────────┼──────────┼─────────────────┼──────────
로컬 세션         | 손실       | 손실      | 불가              | 0
Sticky Session  | 일부 손실   | 일부 손실  | 제한적            | 로드밸런서
Redis Session   | 유지       | 유지      | 완전 지원          | Redis 운영

세션 크기 vs 성능:
  세션 속성 총 크기: 5KB → Redis GET 5KB: ~0.5ms (OK)
  세션 속성 총 크기: 100KB → Redis GET 100KB: ~2ms (주의)
  세션 속성 총 크기: 1MB → Redis GET 1MB: ~20ms (위험!)
  
  권장: 세션에는 ID, 권한, 최소 필수 정보만 저장
        대용량 데이터는 Redis 또는 DB에서 세션 ID로 조회

flush-mode 성능:
  on-save: 응답당 1회 Redis 쓰기
  immediate: setAttribute 호출마다 Redis 쓰기
  → 10번 setAttribute 시:
    on-save:   1회 Redis 쓰기
    immediate: 10회 Redis 쓰기 (10배 부하)
```

---

## ⚖️ 트레이드오프

```
Redis Session의 트레이드오프:

장점:
  서버 무상태화 → 수평 확장 용이
  세션 지속성 → 재배포/장애에도 로그인 유지
  세션 중앙 관리 → 강제 로그아웃, 동시 로그인 제한 구현 용이

단점:
  Redis 추가 인프라 비용
  모든 요청에서 Redis 조회 (~0.5ms 추가)
  Redis 장애 시 세션 전체 영향
  
  Redis 장애 대비:
    Redis Sentinel로 자동 Failover
    일시적 Redis 장애: 세션 조회 실패 → 재로그인 유도 (허용 가능)
    
maxInactiveInterval 설정:
  짧게: 보안 강화, 사용자 불편 증가
  길게: 사용자 편의, 보안 약화, 메모리 사용 증가
  → 서비스 특성에 맞게: 금융(30분), SNS(7일), API(1시간)

세션 vs JWT:
  세션: 서버 제어 가능 (강제 만료, 동시 세션 관리), Redis 필요
  JWT:  서버리스 가능, 만료 전 토큰 무효화 어려움 (Redis 블랙리스트 필요)
  → 세션 제어가 중요: Spring Session
  → API 서버, 마이크로서비스: JWT + Redis 블랙리스트
```

---

## 📌 핵심 정리

```
Spring Session + Redis 핵심:

동작 원리:
  SessionRepositoryFilter가 HttpSession 가로채기
  → 쿠키에서 세션 ID 추출
  → Redis HGETALL로 세션 데이터 로드
  → 응답 후 변경사항 Redis에 저장

Redis 키 구조:
  spring:session:sessions:{id}          → 세션 데이터 (Hash)
  spring:session:sessions:expires:{id}  → 만료 트리거 (String)
  spring:session:expirations:{time}     → 만료 예정 세션 목록 (Set)

TTL:
  maxInactiveIntervalInSeconds → Redis TTL 설정
  lastAccessedTime 갱신 시 TTL 리셋 (Sliding Expiration)

세션 만료 이벤트:
  Redis Keyspace Notification 필요 (Kgxe)
  ApplicationListener<SessionExpiredEvent> 구현

직렬화:
  기본: JDK → JSON으로 변경 권장
  @Bean springSessionDefaultRedisSerializer()로 교체

Redis Cluster:
  세션 관련 키들이 다른 슬롯 → CROSSSLOT 위험
  → 권장: 세션 전용 Redis Sentinel 분리
```

---

## 🤔 생각해볼 문제

**Q1.** 사용자가 동시에 여러 탭/기기에서 로그인할 때, Spring Session으로 "최대 1개 세션"을 강제하는 방법은?

<details>
<summary>해설 보기</summary>

**Spring Security SessionManagementFilter + Spring Session 연동:**

```java
// SecurityConfig
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .sessionManagement(session -> session
            .maximumSessions(1)  // 최대 1개 세션
            .maxSessionsPreventsLogin(false)  // true: 새 로그인 차단, false: 기존 세션 만료
            .sessionRegistry(sessionRegistry())
        );
    return http.build();
}

@Bean
public SpringSessionBackedSessionRegistry<? extends Session> sessionRegistry() {
    return new SpringSessionBackedSessionRegistry<>(
        (FindByIndexNameSessionRepository<? extends Session>) sessionRepository);
}
```

**내부 동작:**
1. 사용자 로그인 시 세션에 username 인덱스 저장
2. Redis에 `spring:session:index:org.springframework.session.FindByIndexNameSessionRepository.PRINCIPAL_NAME_INDEX_NAME:{username}` 키로 세션 목록 유지
3. 새 로그인 시 해당 사용자의 기존 세션 수 확인
4. `maximumSessions(1)` → 기존 세션을 만료시키거나 새 로그인 차단

**세션 인덱스 조회:**
```bash
# 특정 사용자의 모든 세션 확인
redis-cli SMEMBERS "spring:session:index:PRINCIPAL_NAME_INDEX:alice@example.com"
```

</details>

---

**Q2.** `flush-mode: on-save`에서 한 요청 내에 `session.setAttribute("step", 1)` 후 예외가 발생해 트랜잭션이 롤백됐다. 세션은 Redis에 저장되는가?

<details>
<summary>해설 보기</summary>

**저장되지 않는다.** `on-save` 모드에서 세션 저장 시점은 `SessionRepositoryFilter`의 `doFilter` 마지막에 실행된다. 예외가 전파되면 `doFilter`가 정상 완료되지 않으므로 세션 저장 코드가 실행되지 않는다.

**흐름:**
```
SessionRepositoryFilter.doFilter() {
    try {
        chain.doFilter(wrappedRequest, wrappedResponse);  // 예외 발생!
        session.save();  // ← 실행 안 됨!
    } catch (Exception e) {
        // 예외 재전파
    }
}
```

**`immediate` 모드라면:**
- `session.setAttribute("step", 1)` 즉시 Redis 쓰기 → Redis에 저장됨
- 트랜잭션 롤백은 DB 변경만 취소 → Redis 세션은 롤백 불가
- DB: 롤백, Redis 세션: 변경된 상태 (불일치 가능)

**실무 권장:**
- 트랜잭션 내에서 세션 속성 변경 시, 트랜잭션 성공 후에만 세션에 저장하도록 로직 설계
- 또는 `TransactionSynchronization.afterCommit()`에서 세션 속성 설정

</details>

---

**Q3.** Redis Session을 사용하는 서버가 5대 있는데, Redis 장애로 세션을 읽을 수 없게 됐다. 어떤 영향이 발생하고, 어떻게 대처하는가?

<details>
<summary>해설 보기</summary>

**영향:**
모든 HTTP 요청에서 세션 조회 실패 → 인증된 사용자도 로그인 상태 잃음 → 모든 사용자 강제 로그아웃 효과

**오류 전파 방식:**
```
SessionRepositoryFilter → findById() → Redis 연결 오류 → 예외
→ Spring Security: 인증 없음으로 판단 → 403 Forbidden 또는 로그인 페이지 리다이렉트
```

**대처 방안:**

1. **즉각 대응: Redis 복구**
   - Redis Sentinel 구성 시: 자동 Failover (~30초)
   - Redis 단일 인스턴스: 수동 재시작

2. **단기 대응: 폴백 세션 처리**
```java
@Bean
public SessionRepository<?> sessionRepository(RedisConnectionFactory factory) {
    RedisIndexedSessionRepository repo = new RedisIndexedSessionRepository(
        new RedisTemplate<>());
    // Redis 불가 시 로컬 메모리 폴백 (일시적)
    return new CompositeSessionRepository(Arrays.asList(
        repo,
        new MapSessionRepository(new ConcurrentHashMap<>())
    ));
}
```

3. **장기 대응: HA 구성**
   - Redis Sentinel (자동 Failover, Master 장애 시 Replica 승격)
   - 세션 서버를 별도 Redis 인스턴스로 분리 (캐시와 다른 Redis)
   - `spring.session.redis.cleanup-cron` 설정으로 만료 세션 정기 정리

</details>

---

<div align="center">

**[⬅️ 이전: Spring Cache @Cacheable](./02-spring-cache-cacheable.md)** | **[홈으로 🏠](../README.md)** | **[다음: Redisson 분산 패턴 ➡️](./04-redisson-distributed-patterns.md)**

</div>
