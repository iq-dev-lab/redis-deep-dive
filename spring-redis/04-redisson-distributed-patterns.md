# 분산 환경 패턴 — Redisson 분산 락과 세마포어

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Redisson `RLock`이 Lua 스크립트로 락 획득/해제를 구현하는 원리는?
- Pub/Sub 기반 락 대기가 Busy-Waiting보다 효율적인 이유는?
- Watchdog이 락 갱신을 자동화하는 방식은?
- `RSemaphore`로 동시 처리 수를 제한하는 패턴과 분산 Rate Limiting 구현은?
- `RLock` vs `RedissonRedLock` vs 단순 `SET NX EX`의 선택 기준은?
- Spring Batch + Redis `ItemReader`로 분산 환경 배치 처리 시 고려사항은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

분산 환경에서 "단 하나의 프로세스만 실행"이나 "최대 N개 동시 처리"를 보장하려면 분산 락과 세마포어가 필요하다. 직접 구현한 `SET NX EX` + Lua 스크립트는 Watchdog, Pub/Sub 대기, Redlock 등을 직접 구현해야 해서 복잡하다. Redisson은 이 모든 것을 검증된 방식으로 제공하지만, 내부 동작을 모르면 Watchdog 기본 설정으로 인한 메모리 누수나 락 해제 실패를 디버깅하기 어렵다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 락 해제 없이 예외 처리 누락

  코드:
    RLock lock = redisson.getLock("stock:lock:item:1");
    lock.lock();
    processOrder();  // 예외 발생!
    lock.unlock();   // ← 실행 안 됨!
  
  결과:
    Watchdog가 기본 30초(lockWatchdogTimeout) 동안 TTL 갱신
    → 30초 동안 다른 프로세스 락 획득 불가
    → 처리 지연 발생
  
  올바른 코드:
    lock.lock();
    try {
        processOrder();
    } finally {
        lock.unlock();  // 항상 실행!
    }

실수 2: Semaphore permits를 Redis에 초기화 안 함

  코드:
    RSemaphore semaphore = redisson.getSemaphore("api:rate:limit");
    semaphore.acquire();  // 영원히 대기!
  
  원인: RSemaphore의 현재 permits가 0 (초기화 안 됨)
  해결: semaphore.trySetPermits(100) 먼저 실행

실수 3: tryLock 없이 lock()을 무한 대기

  코드:
    lock.lock();  // 락 획득될 때까지 무한 대기
    // 다른 서버가 lock.unlock() 안 하면 → 영원히 대기!
  
  운영 환경 장애:
    락 보유 서버 크래시 → Watchdog 중단 → TTL 만료 후 자동 해제 (30초)
    그 30초 동안 다른 모든 서버 대기
    
  올바른 코드:
    boolean acquired = lock.tryLock(5, 30, TimeUnit.SECONDS);
    // 5초 대기, 30초 TTL
    if (!acquired) {
        throw new LockAcquisitionException("락 획득 실패");
    }
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
RLock 올바른 사용 패턴:
  RLock lock = redisson.getLock("order:lock:" + orderId);
  
  // tryLock: 타임아웃 있는 락 획득
  boolean acquired = lock.tryLock(
      5,   // waitTime: 최대 5초 대기
      30,  // leaseTime: 30초 TTL (-1이면 Watchdog 사용)
      TimeUnit.SECONDS
  );
  
  if (!acquired) {
      throw new BusinessException("현재 처리 중인 주문입니다");
  }
  
  try {
      // 임계 구역
      processOrder(orderId);
  } finally {
      lock.unlock();  // 반드시 finally에서
  }

RSemaphore로 동시 처리 제한:
  RSemaphore semaphore = redisson.getSemaphore("payment:concurrent");
  semaphore.trySetPermits(10);  // 최대 10개 동시 처리 (초기화, 최초 1회)
  
  boolean acquired = semaphore.tryAcquire(5, TimeUnit.SECONDS);
  if (!acquired) {
      throw new ServiceUnavailableException("처리 용량 초과");
  }
  try {
      processPayment();
  } finally {
      semaphore.release();
  }

Watchdog 비활성화 (고정 TTL 사용):
  // leaseTime > 0 으로 고정 TTL 설정 시 Watchdog 비활성화
  lock.tryLock(5, 60, TimeUnit.SECONDS)  // Watchdog 없이 60초 TTL
```

---

## 🔬 내부 동작 원리

### 1. RLock 내부 — Lua 스크립트 원자적 락

```
RLock.lock() 내부 Lua 스크립트:

락 획득 스크립트:
  -- KEYS[1]: 락 키 이름 (예: "order:lock:123")
  -- ARGV[1]: 락 만료 시간 (ms)
  -- ARGV[2]: 락 소유자 ID (clientId:threadId)
  
  if (redis.call('exists', KEYS[1]) == 0) then
      -- 락이 없으면 새로 생성 (재진입 카운트 = 1)
      redis.call('hincrby', KEYS[1], ARGV[2], 1);
      redis.call('pexpire', KEYS[1], ARGV[1]);
      return nil;  -- 성공
  end;
  
  if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then
      -- 이미 내가 보유한 락 (재진입)
      redis.call('hincrby', KEYS[1], ARGV[2], 1);
      redis.call('pexpire', KEYS[1], ARGV[1]);
      return nil;  -- 재진입 성공
  end;
  
  -- 다른 클라이언트가 보유 중
  return redis.call('pttl', KEYS[1]);  -- 남은 TTL 반환 (대기 시간 계산용)

Redis 저장 구조:
  "order:lock:123" → Hash 타입
  {
    "client123:thread1": 1  # 소유자:카운트 (재진입 지원)
  }

HSET vs SET 이유:
  Hash 사용으로 재진입(Reentrant) 락 구현 가능
  "client123:thread1" 필드에 카운트 → 같은 스레드가 중첩 락 가능
  unlock 시 카운트 감소 → 0이 되면 DEL

락 해제 스크립트:
  if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then
      return nil;  -- 내 락이 아님, 무시
  end;
  
  local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1);
  if (counter > 0) then
      -- 재진입 카운트가 남음, TTL 갱신만
      redis.call('pexpire', KEYS[1], ARGV[2]);
      return 0;
  else
      -- 완전 해제
      redis.call('del', KEYS[1]);
      -- Pub/Sub으로 대기 중인 클라이언트에게 알림
      redis.call('publish', KEYS[2], ARGV[1]);
      return 1;
  end;
```

### 2. Pub/Sub 기반 락 대기 — Busy-Waiting 제거

```
전통적 Busy-Waiting (비효율):
  while (!tryLock()) {
      Thread.sleep(100);  // 100ms마다 재시도
  }
  → 계속 Redis에 요청 (CPU + 네트워크 낭비)
  → 락 해제 후 최대 100ms 지연 발생

Redisson의 Pub/Sub 방식 (효율적):

  락 획득 실패 시:
  1. Redis의 채널 구독: "redisson_lock_channel:{lockName}"
  2. 블로킹 대기 (스레드 sleep)

  락 해제 시:
  1. 락 보유자가 DEL + PUBLISH "redisson_lock_channel:{lockName}" "0"
  2. 구독 중인 클라이언트들이 메시지 수신
  3. 수신 즉시 락 재획득 시도

  장점:
    락 해제 즉시 통보 받음 (지연 최소화, 수 ms)
    대기 중 Redis 요청 없음 (CPU + 네트워크 절약)
    서버 100개가 대기 중이어도 Redis 부하 없음

실제 성능 차이:
  Busy-Waiting (100ms 간격):
    100개 서버, 모두 대기 중
    초당 Redis 요청: 100 / 0.1초 = 1,000 req/sec (낭비)
  
  Pub/Sub:
    100개 서버, 모두 Pub/Sub 구독 대기
    초당 Redis 요청: 0 (락 해제 전까지)
    락 해제 시: 1개 PUBLISH → 100개 SUBSCRIBE 메시지 수신
```

### 3. Watchdog — 자동 TTL 갱신

```
Watchdog 동작 원리:

  lock.lock() 호출 시:
    → leaseTime = -1 (기본값) → Watchdog 활성화
    → 초기 TTL = lockWatchdogTimeout (기본 30초)
    → 백그라운드 스레드(Watchdog) 시작
       ↓ 주기: lockWatchdogTimeout / 3 마다 (기본 10초)
    → 락이 아직 내 것이면 TTL을 30초로 리셋
    
  lock.unlock() 호출 시:
    → 락 해제
    → Watchdog 중단

Watchdog의 역할:
  작업이 30초 이상 걸려도 락 유지
  작업 완료 후 unlock() 호출 → Watchdog 자동 중단

Watchdog가 중단되는 경우:
  unlock() 정상 호출 → 중단
  프로세스 강제 종료 (kill -9) → Watchdog 스레드 종료
    → TTL 갱신 중단 → 마지막 TTL 만료 후 자동 해제

Watchdog 설정:
  Config config = new Config();
  config.setLockWatchdogTimeout(30000);  // 30초 (밀리초)
  RedissonClient redisson = Redisson.create(config);

leaseTime 고정 vs Watchdog:
  lock.lock()           → Watchdog 사용 (무한 연장)
  lock.lock(30, SECONDS) → Watchdog 없음, 30초 고정 TTL
  lock.tryLock(5, 30, SECONDS) → 5초 대기, 30초 고정 TTL, Watchdog 없음

  권장:
    작업 시간 예측 가능: 고정 TTL (leaseTime 지정, Watchdog 없음)
    작업 시간 가변적: Watchdog 사용 (leaseTime = -1)
    → 고정 TTL이 더 명확하고 예측 가능 (Watchdog 버그 리스크 없음)
```

### 4. RSemaphore — 동시 처리 수 제한

```
RSemaphore 동작 원리:

  초기화:
    RSemaphore sem = redisson.getSemaphore("payment:semaphore");
    sem.trySetPermits(10);  // 최대 10개 허용
    
    Redis 저장:
    "payment:semaphore" → String: "10" (현재 남은 permits)

  acquire():
    -- Lua 스크립트
    local value = redis.call('get', KEYS[1]);
    if (value ~= false and tonumber(value) > 0) then
        redis.call('decr', KEYS[1]);  -- permits 1 감소
        return 1;  -- 획득 성공
    end;
    return 0;  -- 획득 실패

  release():
    redis.call('incr', KEYS[1]);  -- permits 1 증가
    -- 대기 중인 스레드에 Pub/Sub 알림

분산 Rate Limiting 구현:
  // 초당 1,000 요청 제한 (Sliding Window)
  RRateLimiter rateLimiter = redisson.getRateLimiter("api:rate:user:" + userId);
  rateLimiter.trySetRate(
      RateType.OVERALL,    // 전체 분산 환경에서 공유
      1000,                // 1000 요청
      1,                   // 1
      RateIntervalUnit.SECONDS  // 초
  );
  
  boolean allowed = rateLimiter.tryAcquire(1);
  if (!allowed) {
      throw new RateLimitExceededException("요청 한도 초과");
  }

RSemaphore 활용 패턴:
  // 외부 API 동시 호출 수 제한 (예: 외부 결제 API 최대 5개 동시)
  RSemaphore externalApiSemaphore = redisson.getSemaphore("external:payment:api");
  externalApiSemaphore.trySetPermits(5);
  
  boolean acquired = externalApiSemaphore.tryAcquire(1, 10, TimeUnit.SECONDS);
  if (!acquired) {
      throw new ServiceBusyException("외부 API 처리 용량 초과");
  }
  try {
      callExternalPaymentApi(order);
  } finally {
      externalApiSemaphore.release();
  }
```

### 5. Spring Batch + Redis — 분산 배치 처리

```java
// 문제: 여러 서버에서 동시에 배치 Job 실행 방지

// 방법 1: RedissonJobLock으로 단일 실행 보장
@Component
public class UserBatchJob {
    
    @Autowired
    private RedissonClient redisson;
    
    @Scheduled(cron = "0 0 2 * * ?")  // 매일 새벽 2시
    public void runDailyBatch() {
        RLock lock = redisson.getLock("batch:daily:user:processing");
        
        boolean acquired = lock.tryLock(0, 60 * 60, TimeUnit.SECONDS);
        // waitTime=0: 즉시 실패 (다른 서버가 실행 중이면 포기)
        // leaseTime=3600: 최대 1시간 (배치 예상 시간)
        
        if (!acquired) {
            log.info("다른 서버에서 배치 처리 중, 건너뜀");
            return;
        }
        
        try {
            jobLauncher.run(userBatchJob, jobParameters);
        } finally {
            lock.unlock();
        }
    }
}

// 방법 2: Redis ItemReader로 분산 배치 처리 (파티션)
// 여러 서버가 Redis Queue에서 작업 가져가 병렬 처리

@Component
public class RedisItemReader<T> implements ItemReader<T> {
    
    private final RQueue<T> queue;
    
    public RedisItemReader(RedissonClient redisson, String queueName) {
        this.queue = redisson.getQueue(queueName);
    }
    
    @Override
    public T read() throws Exception {
        return queue.poll();  // null 반환 시 배치 스텝 완료
    }
}

// 배치 파티션 설정:
@Bean
public Step partitionedStep(StepBuilderFactory factory) {
    return factory.get("partitionedStep")
        .partitioner("workerStep", partitioner())
        .step(workerStep())
        .gridSize(10)  // 10개 파티션 (서버 10대)
        .taskExecutor(taskExecutor())
        .build();
}

// 파티셔너: 데이터를 Redis Queue에 분배
@Component
public class RedisPartitioner implements Partitioner {
    @Override
    public Map<String, ExecutionContext> partition(int gridSize) {
        // 전체 작업 목록을 gridSize개 파티션으로 분할
        // 각 파티션을 Redis Queue에 삽입
        Map<String, ExecutionContext> contexts = new HashMap<>();
        List<Long> allIds = fetchAllIds();
        
        List<List<Long>> partitions = Lists.partition(allIds, 
            allIds.size() / gridSize + 1);
        
        for (int i = 0; i < partitions.size(); i++) {
            RQueue<Long> queue = redisson.getQueue("batch:partition:" + i);
            queue.addAll(partitions.get(i));
            
            ExecutionContext context = new ExecutionContext();
            context.putString("queueName", "batch:partition:" + i);
            contexts.put("partition" + i, context);
        }
        return contexts;
    }
}
```

### 6. RLock vs RedissonRedLock vs SET NX EX

```
선택 기준:

SET NX EX (직접 구현):
  적합: 단순 락, Redis 라이브러리 없는 환경
  장점: 가볍고, 의존성 없음
  단점: Watchdog, Pub/Sub 대기, Redlock 직접 구현 필요
  
  코드:
    String token = UUID.randomUUID().toString();
    Boolean acquired = redisTemplate.opsForValue()
        .setIfAbsent(lockKey, token, Duration.ofSeconds(30));

RLock (Redisson 기본 락):
  적합: 대부분의 분산 락 요구사항
  장점: Watchdog, 재진입, Pub/Sub 대기, Lua 원자성
  단점: Redis 라이브러리 의존
  
  Redis Master 장애 시:
    락 정보가 Replica에 복제 안 됐을 수 있음
    Sentinel 자동 Failover 후 새 Master → 락 정보 없음
    → 두 클라이언트가 동시에 락 획득 가능 (이론적)

RedissonRedLock (Redlock 구현):
  적합: Redis 단일 노드 장애에도 완벽한 락 보장 필요
  특징: 5개 독립 Redis에서 과반수(3개) 획득
  장점: 단일 Redis 장애에도 락 유지
  단점: 5개 Redis 노드 필요, Kleppmann 논쟁 (GC Pause 문제)
  
  코드:
    RLock lock1 = redisson1.getLock("lock");
    RLock lock2 = redisson2.getLock("lock");
    RLock lock3 = redisson3.getLock("lock");
    RedissonRedLock redLock = new RedissonRedLock(lock1, lock2, lock3);
    
    // Redis 3.0 deprecated, RedLock으로 대체
    boolean acquired = redLock.tryLock(5, 30, TimeUnit.SECONDS);

실무 권장:
  단순 중복 실행 방지: RLock (대부분 충분)
  강한 일관성: RLock + 멱등성 설계 (비즈니스 레벨 보장)
  이론적 완벽: Zookeeper + Fencing Token (Redis 불필요)
```

---

## 💻 실전 실험

### 실험 1: RLock 동작 확인

```java
@SpringBootTest
class RLockTest {
    
    @Autowired
    private RedissonClient redisson;
    
    @Test
    void lockAndUnlock() throws InterruptedException {
        RLock lock = redisson.getLock("test:lock:1");
        
        // 락 획득
        lock.lock();
        System.out.println("락 획득: " + lock.isHeldByCurrentThread());
        
        // Redis에서 락 확인
        // redis-cli HGETALL "test:lock:1"
        // → {clientId:threadId: 1}
        
        Thread t = new Thread(() -> {
            boolean acquired = false;
            try {
                acquired = lock.tryLock(2, TimeUnit.SECONDS);
                System.out.println("다른 스레드 락 획득: " + acquired);  // false
            } finally {
                if (acquired) lock.unlock();
            }
        });
        t.start();
        t.join();
        
        lock.unlock();
        System.out.println("락 해제 후 재획득: " + lock.tryLock(0, 1, TimeUnit.SECONDS));
        lock.unlock();
    }
    
    @Test
    void reentrantLock() {
        RLock lock = redisson.getLock("reentrant:lock");
        
        lock.lock();  // 첫 번째 획득
        lock.lock();  // 재진입 (같은 스레드)
        
        // redis-cli HGETALL "reentrant:lock"
        // → {clientId:threadId: 2}  ← 카운트 2
        
        lock.unlock();  // 카운트 → 1
        lock.unlock();  // 카운트 → 0 → DEL
    }
}
```

### 실험 2: RSemaphore Rate Limiting

```java
@Test
void semaphoreRateLimiting() throws InterruptedException {
    RSemaphore semaphore = redisson.getSemaphore("test:semaphore");
    semaphore.delete();  // 초기화
    semaphore.trySetPermits(3);  // 최대 3개 동시
    
    CountDownLatch latch = new CountDownLatch(10);
    AtomicInteger success = new AtomicInteger(0);
    AtomicInteger failed = new AtomicInteger(0);
    
    for (int i = 0; i < 10; i++) {
        new Thread(() -> {
            try {
                boolean acquired = semaphore.tryAcquire(1, 2, TimeUnit.SECONDS);
                if (acquired) {
                    success.incrementAndGet();
                    Thread.sleep(500);  // 0.5초 작업
                    semaphore.release();
                } else {
                    failed.incrementAndGet();
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } finally {
                latch.countDown();
            }
        }).start();
    }
    
    latch.await(10, TimeUnit.SECONDS);
    System.out.println("성공: " + success.get() + ", 실패: " + failed.get());
    // 10개 요청, 3개 동시 최대 → 일부 실패
}
```

### 실험 3: Watchdog 동작 확인

```bash
# 1. Spring에서 lock.lock() 실행 (leaseTime = -1, Watchdog 활성)
# 2. Redis에서 TTL 확인

redis-cli PTTL "order:lock:123"
# → 30000 (30초)

# 10초 후
sleep 10
redis-cli PTTL "order:lock:123"
# → 30000 (Watchdog이 갱신!)

# unlock() 호출 후
redis-cli EXISTS "order:lock:123"
# → 0 (삭제됨)
```

### 실험 4: 분산 배치 중복 실행 방지

```java
// 두 서버에서 동시에 배치 실행 시도
@Test
void preventDuplicateBatch() throws InterruptedException {
    String lockKey = "batch:daily:" + LocalDate.now();
    CountDownLatch latch = new CountDownLatch(2);
    AtomicInteger executedCount = new AtomicInteger(0);
    
    Runnable batchTask = () -> {
        RLock lock = redisson.getLock(lockKey);
        boolean acquired = lock.tryLock(0, 3600, TimeUnit.SECONDS);
        if (acquired) {
            try {
                executedCount.incrementAndGet();
                System.out.println("배치 실행: " + Thread.currentThread().getName());
                Thread.sleep(100);  // 배치 작업 시뮬레이션
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } finally {
                lock.unlock();
            }
        } else {
            System.out.println("건너뜀: " + Thread.currentThread().getName());
        }
        latch.countDown();
    };
    
    new Thread(batchTask, "server-1").start();
    new Thread(batchTask, "server-2").start();
    
    latch.await();
    assertThat(executedCount.get()).isEqualTo(1);  // 단 1번만 실행!
}
```

---

## 📊 성능/비용 비교

```
락 구현 방식별 비교:

방식               | Redis 요청 수   | 대기 방식      | 재진입  | Watchdog | 복잡도
──────────────────┼───────────────┼──────────────┼───────┼──────────┼──────
SET NX EX (직접)   | N × 폴링       | Busy-Waiting | 없음   | 없음      | 낮음
RLock (Redisson)  | O(1) 획득      | Pub/Sub 대기  | 있음   | 있음      | 낮음
RedissonRedLock   | O(5) 획득      | Pub/Sub 대기  | 있음   | 있음      | 높음
ZooKeeper         | Znode 생성/감시 | Watcher      | 있음   | N/A      | 매우 높음

동시 대기자 100명 시 Redis 요청 (초당):
  Busy-Waiting (100ms 간격): 100 × 10 = 1,000 req/sec
  Pub/Sub (Redisson):        0 req/sec (해제 알림 대기)
  → 1,000배 효율 차이

RSemaphore vs RLock:
  RLock: 단일 실행 (permits = 1)
  RSemaphore: N개 동시 실행 (permits = N)
  
  RSemaphore 10 permits:
    초당 100 요청 시 → 10개 동시 처리 → 나머지 대기 or 실패
    외부 API 부하 제어에 효과적

RRateLimiter 정확도:
  초당 1,000 요청 제한 설정
  실제 허용: ~1,000 ± 작은 오차 (원자적 Lua 스크립트)
  분산 환경: 모든 서버 합계 1,000 req/sec 보장
```

---

## ⚖️ 트레이드오프

```
RLock Watchdog:
  장점: 작업 시간에 상관없이 락 유지
  단점: 백그라운드 스레드 오버헤드, unlock() 누락 시 30초 지연
  → 고정 TTL (leaseTime 지정)이 더 예측 가능하고 안전

Pub/Sub 대기의 트레이드오프:
  장점: Redis 부하 최소
  단점: Pub/Sub 연결 유지 필요, Redis 재시작 시 구독 끊김
  → Redisson이 자동으로 재구독 처리

RSemaphore permits 설정:
  너무 낮게: 처리 속도 느림, 대기 큐 증가
  너무 높게: 외부 리소스 과부하, 오히려 전체 성능 저하
  → 외부 API 문서의 Rate Limit 기준으로 설정

RedissonRedLock:
  Kleppmann 비판: GC Pause로 인한 이론적 안전하지 않음
  Antirez 반론: 실용적으로 충분히 안전
  → 대부분 서비스: RLock 충분 + 멱등성 설계로 보완
  → 이론적 완벽 필요: ZooKeeper + Fencing Token
```

---

## 📌 핵심 정리

```
Redisson 분산 패턴 핵심:

RLock:
  Lua 스크립트: 원자적 Hash 기반 락 (재진입 지원)
  Pub/Sub 대기: 락 해제 시 즉시 통보 (Busy-Waiting 없음)
  Watchdog: leaseTime=-1 시 30초마다 TTL 자동 갱신
  
  사용 패턴:
    lock.tryLock(waitTime, leaseTime, TimeUnit)
    try { 작업 } finally { lock.unlock() }

RSemaphore:
  permits 수만큼 동시 실행 허용
  trySetPermits(N) 초기화 필수
  acquire()/release() 짝으로 사용
  동시 처리 제한, Rate Limiting에 활용

Spring Batch:
  RLock으로 단일 서버 실행 보장
  RQueue로 분산 파티션 처리

선택 기준:
  단순 중복 방지: RLock (실무 대부분)
  동시 처리 수 제한: RSemaphore
  Rate Limiting: RRateLimiter
  이론적 완벽: ZooKeeper (Redis 불필요)

주의:
  반드시 finally에서 unlock()/release()
  leaseTime 적절히 설정 (Watchdog vs 고정 TTL)
  tryLock의 waitTime으로 타임아웃 설정
```

---

## 🤔 생각해볼 문제

**Q1.** `lock.lock()`으로 락을 획득했는데 서버가 `kill -9`로 강제 종료됐다. 이후 어떻게 락이 해제되는가?

<details>
<summary>해설 보기</summary>

**Watchdog 스레드가 종료되면서 자동으로 TTL 갱신이 중단된다.**

흐름:
1. `lock.lock()` → Watchdog 시작, 초기 TTL 30초 설정
2. `kill -9` → JVM 강제 종료 → Watchdog 스레드 종료
3. Redis의 락 키(`order:lock:123`) TTL 카운트다운 계속
4. **30초 후 TTL 만료 → Redis 자동 삭제 → 락 해제**
5. 다른 서버가 락 획득 가능

**정리:**
- `kill -9`는 JVM을 즉시 종료하므로 `unlock()`이 실행되지 않음
- Watchdog도 종료되므로 TTL 갱신 없음
- 마지막으로 설정된 TTL(30초) 후 자동 해제
- 이 30초가 "서버 장애 시 최대 잠금 지속 시간"

**최소화 방법:**
```java
// 고정 TTL 사용 (Watchdog 없음) → 서버 장애 시 해당 TTL 후 자동 해제
lock.tryLock(5, 60, TimeUnit.SECONDS)
// 60초 후 자동 해제 → 서버 장애 시 최대 60초 대기

// vs Watchdog 사용 (lock.lock())
// → 서버 장애 시 마지막 Watchdog 갱신 시점부터 30초 후 해제
// → 실제로는 0~30초 범위
```

</details>

---

**Q2.** 같은 스레드에서 `lock.lock()`을 2번 호출하면 Redis에는 어떻게 저장되고, 몇 번 `unlock()`을 호출해야 완전히 해제되는가?

<details>
<summary>해설 보기</summary>

**Redisson RLock은 재진입(Reentrant) 락이다.**

```java
RLock lock = redisson.getLock("my:lock");

lock.lock();  // 첫 번째 획득
// Redis: HSET "my:lock" "client123:thread1" 1

lock.lock();  // 재진입 (같은 스레드)
// Redis: HINCRBY "my:lock" "client123:thread1" 1
// Redis: HGET "my:lock" "client123:thread1" → 2

lock.unlock();  // 카운트 감소
// Redis: HINCRBY "my:lock" "client123:thread1" -1
// Redis: HGET "my:lock" "client123:thread1" → 1
// 락 아직 유지!

lock.unlock();  // 완전 해제
// Redis: HINCRBY "my:lock" "client123:thread1" -1 → 0
// Redis: DEL "my:lock"
// 락 해제 + Pub/Sub 알림
```

**결론: `lock()` 2번 → `unlock()` 2번 필요**

**주의사항:**
```java
// unlock()을 1번만 호출하면 락이 해제되지 않음!
lock.lock();
lock.lock();
try {
    work();
} finally {
    lock.unlock();  // 1번만 → 카운트 = 1, 락 유지
    // 두 번째 unlock() 누락 → 30초 후 Watchdog TTL 만료 시 자동 해제
}
```

**실용적 규칙:** 재진입 락은 획득 횟수와 해제 횟수를 항상 같게 맞춰야 한다. 재진입 락이 필요한 경우라면 코드 구조를 재설계해서 재진입 자체를 피하는 것이 더 안전하다.

</details>

---

**Q3.** RSemaphore permits를 10으로 설정했는데 서버 배포 재시작 후 permits가 변경되는 문제가 발생했다. 왜 이런 일이 생기고 어떻게 방지하는가?

<details>
<summary>해설 보기</summary>

**원인: `trySetPermits()`의 특성**

`trySetPermits(N)`은 **키가 없을 때만** N으로 설정한다. 이미 키가 있으면 (permits가 사용 중이어도) 무시한다.

```java
semaphore.trySetPermits(10);  // 처음 실행 → 10으로 설정
// 3개 acquire() 후 permits = 7

// 서버 재배포
semaphore.trySetPermits(10);  // 이미 "7"이 저장됨 → 무시
// permits는 여전히 7 (재배포 후 동시 처리 수가 갑자기 줄어보임)
```

**더 심각한 문제: 서버 충돌로 release() 누락**

```
서버 A: acquire() × 5 (permits = 5)
서버 A: 충돌 → release() 실행 안 됨
permits = 5 → 재시작 후 서버들이 다시 trySetPermits(10) 해도 효과 없음
→ 최대 5개만 동시 처리 가능
```

**해결 방법:**

1. **애플리케이션 시작 시 강제 초기화:**
```java
@PostConstruct
public void initSemaphore() {
    RSemaphore semaphore = redisson.getSemaphore("api:semaphore");
    semaphore.delete();  // 기존 값 삭제
    semaphore.trySetPermits(10);  // 새로 초기화
}
```
단점: 모든 서버가 동시 재시작하면 순간 permits 과잉 초기화

2. **Redis Keyspace Expiration 활용:**
```java
semaphore.expire(Duration.ofHours(1));  // 1시간마다 자동 만료
// 재시작 시 키 없으면 trySetPermits 적용
```

3. **분산 락으로 단일 초기화 보장:**
```java
RLock initLock = redisson.getLock("semaphore:init:lock");
if (initLock.tryLock(0, 10, TimeUnit.SECONDS)) {
    try {
        RSemaphore sem = redisson.getSemaphore("api:semaphore");
        sem.delete();
        sem.trySetPermits(10);
    } finally {
        initLock.unlock();
    }
}
```

</details>

---

<div align="center">

**[⬅️ 이전: Spring Session](./03-spring-session-redis.md)** | **[홈으로 🏠](../README.md)** | **[🎉 redis-deep-dive 완료!](../README.md)**

</div>
