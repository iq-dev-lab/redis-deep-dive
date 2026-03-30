# 분산 락 — SET NX EX와 Redlock 논쟁

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `SET key value NX EX` 단일 명령어가 원자적 락을 구현하는 원리는?
- 락 해제 시 반드시 Lua 스크립트를 사용해야 하는 이유는?
- Redlock 알고리즘의 핵심 아이디어와 5개 노드 과반수 획득 방식은?
- Martin Kleppmann이 제기한 "시간 가정(timing assumption)" 문제는 무엇인가?
- Antirez의 반론은 무엇이고, 이 논쟁이 결론을 내지 못하는 이유는?
- 실무에서 단순 SET NX EX와 Redlock 중 어느 것을 써야 하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

분산 환경에서 "동시에 한 프로세스만 실행"을 보장하려면 분산 락이 필요하다. Redis의 `SET NX EX`는 구현이 단순하지만 올바르게 구현하지 않으면 락이 해제되지 않거나, 다른 프로세스의 락을 실수로 해제하는 버그가 생긴다. Redlock은 이를 개선했지만 세계적인 분산 시스템 전문가들 사이에서 아직도 논쟁 중인 알고리즘이다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: SETNX와 EXPIRE를 두 개 명령어로 분리

  잘못된 코드:
    if redis.setnx(lock_key, "1"):    # 락 획득
        redis.expire(lock_key, 30)    # TTL 설정
        do_work()
        redis.delete(lock_key)

  문제:
    SETNX 성공 → 프로세스 크래시 → EXPIRE 실행 전!
    → lock_key에 TTL 없음 → 영구 잠금 (데드락)
    
  해결: SET key value NX EX 30 (단일 원자적 명령)

실수 2: 락 해제 시 소유 확인 없이 DELETE

  잘못된 코드:
    redis.set(lock_key, "1", nx=True, ex=30)
    do_work()
    redis.delete(lock_key)  # 소유 확인 없음!

  문제:
    프로세스 A: 락 획득 (value="1")
    작업이 30초 이상 걸림 → TTL 만료 → 락 자동 해제
    프로세스 B: 락 재획득 (value="1")
    프로세스 A: 작업 완료 → DELETE 실행
    → B의 락을 A가 해제! → B도 자신의 락이 없다고 착각

실수 3: 락 갱신(renewal) 없이 장기 작업 실행

  설계: 락 EX = 60초, 작업 = 120초
  결과: 60초 후 TTL 만료 → 다른 프로세스가 락 획득
        두 프로세스 동시 작업 실행 (임계 구역 보호 실패)
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
올바른 단일 Redis 분산 락 구현:

  # 락 획득: 고유 토큰으로 소유자 식별
  import uuid
  token = str(uuid.uuid4())
  acquired = redis.set(lock_key, token, nx=True, ex=30)
  
  if not acquired:
      return "락 획득 실패"
  
  try:
      do_work()
  finally:
      # 락 해제: Lua 스크립트로 원자적 확인 + 삭제
      lua_script = """
      if redis.call('GET', KEYS[1]) == ARGV[1] then
          return redis.call('DEL', KEYS[1])
      else
          return 0
      end
      """
      redis.eval(lua_script, 1, lock_key, token)

Redlock 사용 여부 판단:
  단순 락 (단일 Redis 노드):
    ✓ 단일 Redis Master (Sentinel 없음)
    ✓ Redis 재시작 시 락 유실 허용
    ✓ 일시적 중복 실행 허용 (비즈니스 로직에서 멱등성 보장)
  
  Redlock 고려:
    ✗ Redis Master 장애 시 절대 두 프로세스 동시 실행 불가
    ✗ 강한 일관성 보장 필요
    → 하지만 Kleppmann의 비판을 이해하고 수용 여부 결정
    → Redisson(Java) 등 검증된 라이브러리 사용 권장
```

---

## 🔬 내부 동작 원리

### 1. SET NX EX — 원자적 락의 기반

```
SET key value NX EX seconds 명령어:
  NX: 키가 존재하지 않을 때만 SET (Not eXists)
  EX: 만료 시간 설정 (seconds)
  단일 명령어로 SET + 조건체크 + TTL 설정 원자적 실행

내부 동작:
  if (key가 존재하지 않음):
      SET key = value
      EXPIRE key = seconds
      return "OK"
  else:
      return nil  # 이미 락이 있음

락 획득 흐름:
  Process A: SET lock:job "token-A" NX EX 30
             → OK (락 획득)
  Process B: SET lock:job "token-B" NX EX 30
             → nil (이미 락 있음, 획득 실패)

왜 NX와 EX를 분리하면 안 되는가:
  SETNX lock:job "1"     → 성공
  ← 프로세스 크래시 (EXPIRE 실행 전) →
  EXPIRE lock:job 30     → 실행 안 됨!
  → lock:job TTL 없이 영구 잠금 (데드락)

  하지만 SET lock:job "1" NX EX 30은 원자적:
  NX 조건 확인 + SET + EXPIRE가 단일 명령어 → 분리 불가
```

### 2. 올바른 락 해제 — Lua 스크립트 필수

```
잘못된 해제 (DELETE만):
  GET → 확인 → DELETE 사이에 다른 프로세스가 개입 가능

타임라인 (잘못된 예):
  t=0:  A 락 획득 (value="token-A"), EX=10초
  t=9:  A: 작업 완료, GET lock:job → "token-A" (내 것 확인)
  t=10: TTL 만료 → 락 자동 해제
  t=10: B: 락 획득 (value="token-B")
  t=10: A: DELETE lock:job  ← B의 락을 삭제!
  t=10: B는 락이 있다고 생각하지만 실제로 없음 (보호 실패)

올바른 해제 — Lua 스크립트 (원자적 GET + DELETE):
  -- KEYS[1] = lock_key, ARGV[1] = 내 token
  if redis.call("GET", KEYS[1]) == ARGV[1] then
      return redis.call("DEL", KEYS[1])
  else
      return 0
  end

Lua 스크립트의 원자성:
  Redis 이벤트 루프: 단일 스레드
  Lua 스크립트 실행 = 이벤트 루프 독점
  → GET과 DEL 사이에 다른 명령어 끼어들기 불가
  → 원자적 확인 + 삭제 보장

고유 토큰의 역할:
  UUID4를 값으로 사용:
    token = str(uuid.uuid4())  # "f47ac10b-58cc-4372-a567-0e02b2c3d479"
    redis.set(lock_key, token, nx=True, ex=30)
  
  해제 시:
    GET lock_key → token 확인 → 내 token이면 DELETE
    다른 프로세스의 token이면 DELETE 하지 않음
  
  → 락 소유자만 해제 가능 보장
```

### 3. 락 갱신 (Renewal/Heartbeat)

```
문제: 작업이 EX보다 오래 걸릴 때

  EX = 30초, 작업 = 60초:
  t=0:  락 획득 (EX=30초)
  t=30: TTL 만료 → 락 해제
  t=30: Process B 락 획득
  t=30: A와 B 동시 작업 (임계 구역 보호 실패!)

해결 1: 넉넉한 EX 설정
  예상 최대 작업 시간 × 3배로 EX 설정
  작업 = 60초 → EX = 180초
  단점: 프로세스 크래시 시 180초 동안 락 잠금

해결 2: Watchdog (Background Renewal)
  별도 스레드가 주기적으로 TTL을 갱신:
  
  def watchdog(lock_key, token, ex=30):
      while holding_lock:
          # 내 토큰일 때만 갱신
          lua = """
          if redis.call('GET', KEYS[1]) == ARGV[1] then
              return redis.call('EXPIRE', KEYS[1], ARGV[2])
          else
              return 0
          end
          """
          result = redis.eval(lua, 1, lock_key, token, ex)
          if result == 0:
              break  # 내 락이 아님 → watchdog 중단
          time.sleep(ex / 3)  # TTL의 1/3마다 갱신

  # 락 획득 후 watchdog 시작
  threading.Thread(target=watchdog, args=(lock_key, token)).start()

Redisson(Java)의 구현:
  Redisson은 Watchdog를 자동으로 처리
  기본 lockWatchdogTimeout = 30초
  락을 보유하는 동안 30초마다 TTL 자동 갱신
  → 작업 길이에 무관하게 락 유지
```

### 4. Redlock 알고리즘

```
배경:
  단일 Redis Master + Sentinel 환경:
    Master 장애 → Sentinel이 Replica를 새 Master로 승격
    하지만 복제는 비동기 → 새 Master에 락 정보가 없을 수 있음
    → 두 프로세스가 동시에 락 보유 가능 (장애 시나리오)

Redlock의 해결 아이디어:
  독립적인 Redis 노드 N개(보통 5개)에서 과반수(≥3) 획득 시 락 성립
  단일 노드 장애 시에도 나머지 과반수가 살아있으면 락 보장

알고리즘 (5개 노드 기준):
  1. 현재 시각 t1 기록
  2. 모든 5개 노드에 동시에 SET NX EX 시도
  3. 3개 이상 노드에서 획득 성공 AND 획득에 걸린 시간(t2-t1) < TTL:
     → 락 성공! 유효 TTL = TTL - (t2-t1) - clock_drift
  4. 그렇지 않으면:
     → 획득한 모든 노드에서 즉시 해제
     → 락 실패 (재시도 or 포기)

과반수의 의미:
  5개 노드 중 3개(과반수) 획득:
    어떤 노드 2개가 다운돼도 락 보장
    노드 3개가 동시에 다운되면 락 무효화 가능 (허용되는 수준)

시간 조건 (유효 TTL 계산):
  t2 - t1: 락 획득에 걸린 실제 시간 (네트워크 지연)
  clock_drift: 노드 간 시계 오차 (통상 수 ms)
  유효 TTL = TTL - (t2-t1) - clock_drift
  → 유효 TTL이 양수여야 의미 있는 락 보유 시간 확보
```

### 5. Martin Kleppmann vs Antirez 논쟁

```
Martin Kleppmann의 비판 (2016):
  "How to do distributed locking" 블로그 포스트

  핵심 주장:
  Redlock은 안전성을 위해 "정확한 시계"와 "최대 지연 시간"을 가정
  하지만 실제 분산 환경에서 이 가정은 보장 불가

  시나리오 1: GC Pause
    Process A: 락 획득 (5개 중 3개)
    Process A: GC Pause 발생 (JVM GC로 수십 초 멈춤)
    TTL 만료 → 락 자동 해제
    Process B: 락 획득
    Process A: GC에서 깨어남 → 자신이 락을 가졌다고 생각
    → A와 B 동시 실행!

  시나리오 2: 네트워크 지연
    A가 노드 5개에 락 요청
    일부 노드의 응답이 지연되다가 TTL 만료 후 도착
    A는 과반수 응답을 받았지만, 실제로 일부 노드에서는 TTL이 만료됨
    → 락 유효 기간에 대한 판단 오류

  Kleppmann의 결론:
    "완벽한 분산 락이 필요하다면 ZooKeeper + fencing token 사용"
    Fencing Token: 락 획득 시 단조 증가 번호를 받아 DB에 전달
    DB는 더 높은 번호를 가진 요청만 허용 → GC Pause에서 깨어난 구버전 요청 거부

Antirez(Redis 창시자)의 반론:
  "Is Redlock safe?" 블로그 포스트

  핵심 반론:
  "Kleppmann의 시나리오는 NTP 동기화가 잘못됐거나 GC가 극단적으로 긴 경우"
  "실제 운영 환경에서 GC Pause가 TTL(수 초)을 초과하는 경우는 드묾"
  "완벽한 시스템 모델을 가정하면 어떤 분산 락도 안전하지 않음"
  "Redlock은 '좋은 엔지니어링 상식' 범위 내에서 안전"

논쟁의 본질:
  두 사람 모두 틀리지 않음:
    Kleppmann: "이론적으로 안전하지 않을 수 있다"
    Antirez: "실용적으로 충분히 안전하다"
  
  → 선택은 요구사항에 따라:
    "절대 안전" 필요: ZooKeeper + Fencing Token (복잡도 높음)
    "실용적 안전" 충분: Redlock (Redis로 구현 가능)
    "단순 락" 충분: 단일 Redis SET NX EX + 멱등성 로직
```

---

## 💻 실전 실험

### 실험 1: 올바른 분산 락 구현

```bash
# 락 획득
LOCK_KEY="job:processing:lock"
TOKEN=$(cat /proc/sys/kernel/random/uuid)
ACQUIRED=$(redis-cli SET $LOCK_KEY $TOKEN NX EX 30)
echo "락 획득: $ACQUIRED (token=$TOKEN)"

# 다른 프로세스 획득 시도 (실패)
TOKEN2=$(cat /proc/sys/kernel/random/uuid)
ACQUIRED2=$(redis-cli SET $LOCK_KEY $TOKEN2 NX EX 30)
echo "다른 프로세스 획득: $ACQUIRED2 (nil이어야 함)"

# 현재 락 소유자 확인
redis-cli GET $LOCK_KEY  # token과 일치해야 함

# 올바른 락 해제 (Lua 스크립트)
RESULT=$(redis-cli EVAL "
if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])
else
    return 0
end
" 1 $LOCK_KEY $TOKEN)
echo "락 해제 결과: $RESULT (1이어야 함)"

# 잘못된 토큰으로 해제 시도
redis-cli SET $LOCK_KEY $TOKEN NX EX 30
WRONG_RESULT=$(redis-cli EVAL "
if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])
else
    return 0
end
" 1 $LOCK_KEY "wrong-token-xxx")
echo "잘못된 토큰 해제: $WRONG_RESULT (0이어야 함)"
redis-cli EXISTS $LOCK_KEY  # 1 (삭제 안 됨)
```

### 실험 2: 락 소유자만 해제할 수 있음을 증명

```bash
# Process A: 락 획득
TOKEN_A="token-process-a"
redis-cli SET lock:critical $TOKEN_A NX EX 60

# Process B: 자신의 토큰으로 해제 시도 (실패)
TOKEN_B="token-process-b"
RESULT=$(redis-cli EVAL "
if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])
else
    return 0
end
" 1 lock:critical $TOKEN_B)
echo "B가 A 락 해제 시도: $RESULT"  # 0 (실패)
redis-cli GET lock:critical  # token-process-a (A 락 유지)

# Process A: 올바르게 해제
RESULT=$(redis-cli EVAL "
if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])
else
    return 0
end
" 1 lock:critical $TOKEN_A)
echo "A가 자신의 락 해제: $RESULT"  # 1 (성공)
```

### 실험 3: 데드락 시뮬레이션과 TTL 자동 해제

```bash
# TTL 만료로 데드락 자동 해소 확인
redis-cli SET lock:test "owner-token" NX EX 5
echo "락 보유 중 (5초 TTL)"
redis-cli TTL lock:test

sleep 6  # TTL 만료 대기
redis-cli EXISTS lock:test  # 0 (자동 해제)
echo "TTL 만료로 락 자동 해제됨"

# 새 프로세스 획득 가능
redis-cli SET lock:test "new-token" NX EX 30
echo "새 토큰으로 락 획득 성공"
redis-cli GET lock:test
```

### 실험 4: Redlock 패턴 시뮬레이션 (3개 독립 Redis)

```bash
# 3개 독립 Redis (포트 6379, 6380, 6381)
TOKEN=$(cat /proc/sys/kernel/random/uuid)
LOCK_KEY="distributed:job:lock"
TTL=30

echo "=== Redlock: 3개 노드에 동시 락 획득 시도 ==="
START=$(date +%s%3N)

# 각 노드에 락 획득 시도
R1=$(redis-cli -p 6379 SET $LOCK_KEY $TOKEN NX EX $TTL)
R2=$(redis-cli -p 6380 SET $LOCK_KEY $TOKEN NX EX $TTL)
R3=$(redis-cli -p 6381 SET $LOCK_KEY $TOKEN NX EX $TTL)

END=$(date +%s%3N)
ELAPSED=$((END - START))

# 과반수 확인 (2개 이상 성공)
SUCCESS=0
[ "$R1" = "OK" ] && SUCCESS=$((SUCCESS + 1))
[ "$R2" = "OK" ] && SUCCESS=$((SUCCESS + 1))
[ "$R3" = "OK" ] && SUCCESS=$((SUCCESS + 1))

VALID_TTL=$((TTL * 1000 - ELAPSED - 50))  # 클락 드리프트 50ms 여유

echo "획득: $SUCCESS/3 노드, 소요: ${ELAPSED}ms, 유효 TTL: ${VALID_TTL}ms"

if [ $SUCCESS -ge 2 ] && [ $VALID_TTL -gt 0 ]; then
  echo "Redlock 락 획득 성공!"
else
  echo "Redlock 락 획득 실패 → 모든 노드 해제"
  # 해제
  for port in 6379 6380 6381; do
    redis-cli -p $port EVAL "
    if redis.call('GET', KEYS[1]) == ARGV[1] then
        return redis.call('DEL', KEYS[1])
    end
    return 0" 1 $LOCK_KEY $TOKEN
  done
fi
```

---

## 📊 성능/비용 비교

```
분산 락 방식별 비교:

방식             | 구현 복잡도   | Redis 노드 수  | 장애 시 안전성  | 오버헤드
────────────────┼────────────┼──────────────┼──────────────┼──────────
SET NX EX       | 낮음        | 1개           | 낮음          | 최소 (1 RTT)
SET NX + Watchdog| 중간       | 1개           | 중간          | 추가 쓰레드
Redlock         | 높음        | 5개           | 높음          | 5 RTT (병렬)
Zookeeper       | 매우 높음    | 별도 클러스터    | 매우 높음      | 크로스 클러스터

락 획득 지연:
  SET NX EX:  ~0.5 ms (단일 Redis)
  Redlock:    ~2~5 ms (5개 노드 병렬 요청 중 가장 느린 응답까지)
  ZooKeeper:  ~5~20 ms (별도 클러스터 통신)

Redlock 5개 노드 획득 시간:
  병렬 요청 → 모든 응답 대기 or 과반수 확인
  최대 지연 = 가장 느린 노드의 응답 시간
  네트워크 조건 양호 시: ~1~3 ms 추가
```

---

## ⚖️ 트레이드오프

```
단순 SET NX EX vs Redlock:

단순 SET NX EX 선택:
  Redis Master 하나 (또는 Sentinel 없음)
  Redis 재시작 시 모든 락이 사라져도 됨 (멱등성으로 보완)
  구현 단순 + 저지연

Redlock 고려:
  복수 Redis 노드에서 강한 보장 필요
  Redis 단일 노드 장애 시 두 프로세스 동시 실행 절대 금지
  단, Kleppmann 논쟁을 이해하고 사용 여부 결정

Kleppmann의 Fencing Token 대안:
  락 획득 시 단조 증가 번호를 함께 받음
  락을 사용하는 리소스(DB 등)가 최신 번호만 허용
  → GC Pause에서 깨어난 구버전 요청이 거부됨
  → 락 메커니즘 밖에서 안전성 보장
  → 지원하지 않는 리소스에는 적용 불가 (Redis, 파일 시스템 등)

실용적 선택 기준:
  "혹시 두 프로세스가 동시에 실행돼도 괜찮은가?"
  → 예 (멱등성 보장): 단순 SET NX EX
  → 아니오 (절대 안 됨): Redlock + 멱등성 모두 구현
                          또는 ZooKeeper + Fencing Token
```

---

## 📌 핵심 정리

```
분산 락 핵심:

SET NX EX (기본 구현):
  SET key token NX EX seconds  # 원자적 락 획득
  해제: Lua 스크립트로 GET(확인) + DEL(삭제) 원자적 실행
  token: UUID로 소유자 식별 → 다른 프로세스가 해제 불가

흔한 실수:
  SETNX + EXPIRE 분리 → 데드락 위험
  토큰 확인 없이 DELETE → 다른 프로세스 락 해제 위험
  갱신 없이 장기 작업 → TTL 만료 후 보호 실패

Redlock:
  5개 독립 노드에서 과반수(≥3) 획득
  유효 TTL = TTL - 획득 소요 시간 - 클락 드리프트
  
  논쟁:
    Kleppmann: GC Pause/시계 오차로 안전하지 않을 수 있음
    Antirez: 실용적으로 충분히 안전
  
  실무 선택: Redisson 같은 검증된 라이브러리 사용

선택 기준:
  단순 락 충분: SET NX EX + 멱등성 로직
  강한 보장 필요: Redlock (라이브러리 사용)
  이론적 완벽: ZooKeeper + Fencing Token
```

---

## 🤔 생각해볼 문제

**Q1.** `SET lock:job token NX EX 30`으로 락을 획득했는데, 작업이 45초 걸린다. Watchdog 없이 이 상황을 처리하는 다른 방법은?

<details>
<summary>해설 보기</summary>

**방법 1: 넉넉한 EX 설정**
```python
redis.set(lock_key, token, nx=True, ex=300)  # 5분 (예상 작업의 5배)
```
단점: 프로세스 크래시 시 5분간 다른 프로세스 대기

**방법 2: 중간 갱신 (수동 Heartbeat)**
```python
redis.set(lock_key, token, nx=True, ex=30)
# 15초마다 TTL 갱신
for step in range(3):  # 45초 = 15초 × 3
    do_work_step(step)  # 15초씩 작업 분할
    # 내 토큰이면 갱신
    redis.eval("""
    if redis.call('GET', KEYS[1]) == ARGV[1] then
        return redis.call('EXPIRE', KEYS[1], ARGV[2])
    end
    return 0""", 1, lock_key, token, "30")
```

**방법 3: 작업을 청크로 분할**
```python
while items:
    batch = items[:100]
    with distributed_lock(lock_key):  # 각 배치마다 락 획득/해제
        process_batch(batch)
    items = items[100:]
```
→ 락 보유 시간 단축 → 짧은 EX로 충분

**방법 4: 멱등성으로 중복 실행 허용**
락이 만료돼서 다른 프로세스가 같은 작업을 실행해도 결과가 동일하도록 설계. 락은 "주로 방지"용, 중복 실행 시 DB 레벨에서 처리.

</details>

---

**Q2.** Lua 스크립트로 락 해제를 구현하지 않고 아래처럼 구현했다. 이 코드의 취약점은?

```python
if redis.get(lock_key) == token:
    redis.delete(lock_key)
```

<details>
<summary>해설 보기</summary>

**Time-of-Check-Time-of-Use (TOCTOU) 취약점:**

```
Thread A: redis.get(lock_key) → "token-A" (내 토큰 확인)
Thread A: ← 여기서 GC Pause 발생 (30초) →
Thread A: TTL 만료 → 락 자동 해제
Thread B: redis.set(lock_key, "token-B", nx=True, ex=30) → OK (B 락 획득)
Thread A: GC에서 깨어남 → redis.delete(lock_key)  ← B의 락을 삭제!
Thread B: 락이 없어졌지만 자신이 가졌다고 생각
```

GET과 DELETE 사이에 다른 프로세스가 개입할 수 있다. 이것이 두 연산을 원자적으로 실행해야 하는 이유다.

**Lua 스크립트의 원자성:**
```lua
if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])
else
    return 0
end
```
이 스크립트는 Redis 이벤트 루프에서 중단 없이 실행된다. GET과 DEL 사이에 어떤 명령어도 끼어들 수 없다.

</details>

---

**Q3.** Redlock을 5개 노드 대신 3개 노드로 구현하면 어떤 차이가 있는가?

<details>
<summary>해설 보기</summary>

**3개 노드 Redlock:**
- 과반수 = 2개 노드 획득 필요
- 허용 장애 수 = 3 - 2 = **1개 노드 장애 허용**

**5개 노드 Redlock:**
- 과반수 = 3개 노드 획득 필요
- 허용 장애 수 = 5 - 3 = **2개 노드 장애 허용**

**가용성 vs 안전성:**
- 노드 수 많을수록: 더 많은 장애를 감당, 하지만 더 많은 인프라 비용
- 노드 수 적을수록: 인프라 절약, 하지만 장애 허용 수 감소

**실용적 고려:**
- 3개 노드: 단일 노드 장애까지 허용, 일반적인 서비스에 충분
- 5개 노드: 두 노드 동시 장애 허용, 고가용성 필요한 서비스

주의: 노드 수를 늘려도 Kleppmann이 제기한 GC Pause/시계 오차 문제는 해결되지 않는다. 이는 Redlock의 구조적 한계다.

```bash
# 3개 노드 Redlock (과반수 = 2)
acquired_nodes=0
for port in 6379 6380 6381; do
    result=$(redis-cli -p $port SET lock $token NX EX 30)
    [ "$result" = "OK" ] && acquired_nodes=$((acquired_nodes + 1))
done
[ $acquired_nodes -ge 2 ] && echo "락 획득 성공" || echo "실패"
```

</details>

---

<div align="center">

**[⬅️ 이전: Hot Key 문제](./03-hot-key.md)** | **[홈으로 🏠](../README.md)** | **[다음: Pub/Sub vs Stream ➡️](./05-pubsub-vs-stream.md)**

</div>
