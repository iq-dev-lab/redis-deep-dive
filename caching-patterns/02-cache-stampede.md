# Cache Stampede — PER과 Mutex Lock 해결법

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Cache Stampede(Thundering Herd)가 발생하는 정확한 조건과 메커니즘은?
- Mutex Lock 방식이 Stampede를 방지하지만 새로운 문제(Lock 대기 지연)를 만드는 이유는?
- PER(Probabilistic Early Recomputation)이 만료 전 확률적으로 갱신하는 수식은?
- "외부 재계산"(Early Recomputation) 트리거 확률이 TTL 잔여 시간에 따라 변하는 원리는?
- 실무에서 Stampede를 가장 간단하게 방지하는 방법은?
- PER의 `beta` 파라미터를 조정하면 무엇이 달라지는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

인기 있는 키의 TTL이 만료되는 순간, 수백~수천 개의 요청이 동시에 DB로 쏟아진다. 이것이 Cache Stampede다. DB가 이 폭발적인 부하를 감당 못하면 타임아웃이 발생하고, 그로 인해 더 많은 재요청이 생겨 DB가 다운될 수 있다. 인기 콘텐츠의 TTL 관리가 왜 중요한지 이해해야 이 장애를 예방할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Hot Key에 고정 TTL 설정

  코드:
    cache.setex("popular_article:1", 300, article_data)  # 5분 TTL

  문제:
    초당 10,000 요청이 오는 인기 글
    5분마다 정확히 같은 순간 TTL 만료
    → 10,000 요청 × 몇 초 동안 모두 cache miss
    → 수만 건 동시 DB 쿼리
    → DB CPU 100% → 쿼리 타임아웃 → 더 많은 재시도 → 장애 연쇄

실수 2: TTL 만료 직후 첫 번째 스레드만 DB 조회하도록 했지만 Lock 구현 오류

  잘못된 Lock 구현:
    if not cache.get(key):
        if cache.setnx(lock_key, 1):   # 락 획득
            data = db.query()
            cache.set(key, data)
            cache.delete(lock_key)
        else:
            time.sleep(0.1)            # 락 대기
            return cache.get(key)      # 이 시점에도 없으면 None 반환

  문제:
    sleep(0.1) 후 재조회 시 여전히 miss일 수 있음
    → 일부 클라이언트가 None(nil) 받음
    → 잘못된 응답

실수 3: Stampede를 인지 못하고 DB 스펙을 높임

  현상: 5분마다 주기적으로 DB CPU 스파이크
  대응: DB 인스턴스 업그레이드
  결과: 스파이크는 여전히, 비용만 증가
  근본 원인: Stampede (캐시 계층 문제) → DB 스펙으로 해결 불가
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
가장 간단한 방법: TTL 지터(Jitter) 추가

  # 300초 ± 30초 랜덤 (250~330초)
  import random
  ttl = 300 + random.randint(-30, 30)
  cache.setex(key, ttl, data)
  
  → 여러 인스턴스의 캐시가 다른 시점에 만료
  → 동시 만료 제거 → Stampede 해소

Mutex Lock (DB 조회 1회로 제한):
  # Redis SET NX EX로 분산 락 구현
  acquired = cache.set(lock_key, "1", nx=True, ex=30)
  if acquired:
      data = db.query()
      cache.setex(key, ttl, data)
      cache.delete(lock_key)
  else:
      # 락 대기 후 재조회 (재시도 루프)
      for _ in range(10):
          time.sleep(0.05)
          data = cache.get(key)
          if data: return data

PER (고급, 만료 전 선제 갱신):
  # TTL이 얼마 안 남았을 때 확률적으로 미리 갱신
  # → 만료 순간 Miss 자체를 없앰
```

---

## 🔬 내부 동작 원리

### 1. Cache Stampede 발생 메커니즘

```
타임라인:

  t=0:    article:1 캐시 저장 (TTL 300초)
  t=299:  초당 10,000 요청 중 → cache hit (정상)
  t=300:  TTL 만료 순간

  t=300.000: 요청 #1  → cache miss → DB 쿼리 시작
  t=300.000: 요청 #2  → cache miss → DB 쿼리 시작  (동시!)
  t=300.000: 요청 #3  → cache miss → DB 쿼리 시작  (동시!)
  ...
  t=300.001: 요청 #N  → cache miss → DB 쿼리 시작

  DB 동시 쿼리 수: 10,000 req/sec × 0.1~0.5초 응답 시간 = 1,000~5,000건!

  DB 결과:
    CPU 100% 포화 → 쿼리 타임아웃 (5초 이상)
    → 클라이언트 재시도 → 쿼리 더 쌓임 → DB 다운

  연쇄 장애:
    DB 다운 → 캐시 갱신 불가 → 모든 요청 DB로 → 더 심한 과부하
    → "Thundering Herd" (우레 소리처럼 몰려드는 요청)

발생 조건:
  1. 인기 있는 키 (높은 요청 빈도)
  2. 동일 TTL로 여러 인스턴스에서 만료 (동기화된 만료)
  3. DB 재계산 비용이 큰 경우
```

### 2. 해결책 1: TTL 지터

```
원리: 서로 다른 인스턴스가 다른 시점에 캐시를 갱신하도록 분산

  인스턴스 A: TTL 287초 후 만료 → DB 조회 → 캐시 갱신
  인스턴스 B: TTL 312초 후 만료 → 이미 A가 갱신한 캐시 hit
  인스턴스 C: TTL 294초 후 만료 → cache hit

  → 동시 DB 조회 요청 수 크게 감소

구현:
  # 기본 TTL 300초, ±30초 지터
  base_ttl = 300
  jitter = random.randint(-30, 30)
  actual_ttl = base_ttl + jitter  # 270~330초
  
  redis.setex(key, actual_ttl, value)

단점:
  완벽한 방지 아님 (여전히 일부 동시 만료 가능)
  TTL 예측이 어려워짐 (운영 복잡도 약간 증가)
  → 단독 사용보다 다른 방법과 조합 권장
```

### 3. 해결책 2: Mutex Lock (한 번만 DB 조회)

```
원리: 캐시 miss 시 단 하나의 스레드/프로세스만 DB를 조회하고 나머지는 대기

Redis SET NX EX로 분산 뮤텍스 구현:

  ┌───────────────────────────────────────────────────────┐
  │  def get_with_lock(key, db_query, ttl=300):           │
  │    # 1. 캐시 조회                                       │
  │    cached = redis.get(key)                            │
  │    if cached: return cached                           │
  │                                                       │
  │    lock_key = f"{key}:lock"                           │
  │                                                       │
  │    # 2. 락 획득 시도 (NX EX = 원자적 락)                   │
  │    acquired = redis.set(lock_key, "1",                │
  │                         nx=True, ex=30)               │
  │                                                       │
  │    if acquired:                                       │
  │      try:                                             │
  │        # 3. 내가 락 → DB 조회 → 캐시 갱신                  │
  │        data = db_query()                              │
  │        redis.setex(key, ttl, data)                    │
  │        return data                                    │
  │      finally:                                         │
  │        redis.delete(lock_key)                         │
  │    else:                                              │
  │      # 4. 락 없음 → 대기 후 재시도                         │
  │      for attempt in range(20):                        │
  │        time.sleep(0.1)                                │
  │        cached = redis.get(key)                        │
  │        if cached: return cached                       │
  │      return None  # 20번 대기 후도 없으면 None            │
  └───────────────────────────────────────────────────────┘

Mutex Lock의 문제점:
  ① 락 대기 지연: 락을 가진 스레드 DB 응답 대기 중 (예: 200ms)
     나머지 모든 스레드: 200ms 이상 대기 후 캐시 재조회
     → 요청 처리 시간 증가

  ② 락 홀더 장애: DB 조회 중 프로세스 크래시
     락 EX 30초 → 30초 동안 모든 요청이 대기 or None 반환

  ③ 락 대기 중 None 반환:
     대기 후 재조회했는데 여전히 miss → None(nil) 반환
     → 불완전한 응답 가능성

개선: 스테일 데이터 반환 패턴 (Stale-While-Revalidate)
  캐시 만료 시 즉시 삭제 대신:
  만료된 데이터를 잠시 더 서빙 + 백그라운드에서 DB 조회
  
  구현: TTL이 0에 가까울 때 만료 예비 플래그 설정
    redis.setex("stale:article:1", 600, old_data)  # 여유 TTL로 스테일 보관
    비동기 갱신 태스크 실행
```

### 4. 해결책 3: PER (Probabilistic Early Recomputation)

```
논문 출처: "Optimal Probabilistic Cache Stampede Prevention"
           (Vattani, Chierichetti, Lowenstein, 2015)

핵심 아이디어:
  TTL 만료 직전에 "미리" 갱신을 시작 → 만료 순간 Miss 자체를 없앰
  누가, 언제 갱신할지를 확률로 결정 → Stampede 방지

PER 수식:
  현재 시각:  t_now
  캐시 만료:  t_expiry
  잔여 TTL:  ttl_remaining = t_expiry - t_now
  재계산 시간: delta (DB 조회에 걸리는 시간, 예: 0.1초)
  베타 파라미터: beta (기본 1.0, 클수록 더 일찍 갱신)

  조기 갱신 여부 결정:
    if t_now - delta * beta * log(random()) >= t_expiry:
        # 재계산 실행 (갱신)
    else:
        # 캐시 사용 (갱신 없음)

수식 해석:
  random() → 0~1 균일 분포 → log(random()) → 음수 (-∞~0)
  -log(random()) → 양수 지수 분포
  
  ttl_remaining이 클 때 (만료까지 멀 때):
    t_now - delta * beta * |log(random())| << t_expiry
    → 조기 갱신 확률 매우 낮음
  
  ttl_remaining이 작을 때 (만료 임박):
    조건을 만족할 확률 증가
    → 갱신 트리거될 가능성 높아짐
  
  ttl = 0 (만료 순간):
    항상 t_now ≥ t_expiry → 항상 갱신
    단, 여러 스레드 중 일부만 갱신 (확률적 분산)

Python 구현:
  import math, random, time

  def get_with_per(key, compute_fn, ttl=300, beta=1.0):
      # 캐시에서 값과 만료 시각 함께 저장
      cached = redis.hgetall(f"per:{key}")
      
      if cached:
          value = cached["value"]
          expiry = float(cached["expiry"])
          delta = float(cached["delta"])  # 마지막 재계산 시간
          
          # PER 조건 계산
          now = time.time()
          if now - delta * beta * math.log(random.random()) < expiry:
              return value  # 캐시 사용
          # else: 조기 갱신 (계속 진행)
      
      # 재계산 (DB 조회)
      start = time.time()
      value = compute_fn()
      delta = time.time() - start  # 재계산 소요 시간
      
      expiry = time.time() + ttl
      redis.hset(f"per:{key}", mapping={
          "value": value,
          "expiry": expiry,
          "delta": delta
      })
      redis.expireat(f"per:{key}", int(expiry) + 10)  # 약간 여유
      
      return value

beta 파라미터 효과:
  beta = 0.5: 갱신 더 늦게 시작 (만료에 가까워야 갱신)
  beta = 1.0: 기본값, 균형
  beta = 2.0: 갱신 더 일찍 시작 (여유 있게 미리 갱신)
  
  → beta 증가 = 더 공격적인 선제 갱신 = 더 많은 DB 조회
  → DB 재계산 비용 낮으면 beta 높게, 높으면 낮게
```

---

## 💻 실전 실험

### 실험 1: Cache Stampede 재현

```bash
# 10초 TTL의 캐시 설정
redis-cli SETEX stampede:test 10 "cached_value"

# 만료 시간 확인
redis-cli TTL stampede:test  # 10

# 동시 요청 시뮬레이션 (만료 후)
sleep 11  # TTL 만료 대기

# 100개 동시 miss 시뮬레이션
for i in $(seq 1 100); do
  (
    result=$(redis-cli GET stampede:test)
    if [ -z "$result" ]; then
      echo "miss #$i — DB 조회 시뮬레이션"
    fi
  ) &
done
wait
# → 100개 모두 miss → 100번 "DB 조회" 발생
```

### 실험 2: Mutex Lock으로 Stampede 방지

```bash
# Mutex Lock 패턴 구현
get_with_lock() {
  KEY="stampede:article:1"
  LOCK_KEY="${KEY}:lock"
  
  # 1. 캐시 조회
  VALUE=$(redis-cli GET $KEY)
  if [ -n "$VALUE" ]; then
    echo "HIT: $VALUE"
    return
  fi
  
  # 2. 락 획득 시도 (NX EX 30 = 원자적)
  ACQUIRED=$(redis-cli SET $LOCK_KEY "1" NX EX 30)
  
  if [ "$ACQUIRED" = "OK" ]; then
    echo "락 획득 — DB 조회"
    sleep 0.1  # DB 조회 시뮬레이션 (100ms)
    VALUE="fresh_article_data"
    redis-cli SETEX $KEY 300 $VALUE > /dev/null
    redis-cli DEL $LOCK_KEY > /dev/null
    echo "캐시 갱신 완료: $VALUE"
  else
    echo "락 대기 중..."
    for i in $(seq 1 20); do
      sleep 0.1
      VALUE=$(redis-cli GET $KEY)
      if [ -n "$VALUE" ]; then
        echo "락 대기 후 HIT: $VALUE"
        return
      fi
    done
    echo "락 대기 타임아웃"
  fi
}

# 캐시 없는 상태에서 동시 호출
redis-cli DEL stampede:article:1
for i in $(seq 1 10); do
  (get_with_lock) &
done
wait
```

### 실험 3: TTL 지터 적용

```bash
# TTL 지터 적용 비교
# 지터 없음: 모두 300초 후 동시 만료
for i in $(seq 1 10); do
  redis-cli SETEX "no_jitter:$i" 300 "value$i" > /dev/null
done

# 지터 있음: 270~330초, 분산 만료
for i in $(seq 1 10); do
  JITTER=$((RANDOM % 61 - 30))  # -30 ~ +30
  TTL=$((300 + JITTER))
  redis-cli SETEX "with_jitter:$i" $TTL "value$i" > /dev/null
done

echo "=== TTL 비교 ==="
echo "지터 없음:"
for i in $(seq 1 10); do
  echo "  no_jitter:$i TTL=$(redis-cli TTL no_jitter:$i)"
done

echo "지터 있음:"
for i in $(seq 1 10); do
  echo "  with_jitter:$i TTL=$(redis-cli TTL with_jitter:$i)"
done
# → 지터 있음: 각기 다른 TTL → 분산 만료
```

### 실험 4: 실시간 Stampede 모니터링

```bash
# 캐시 miss율 모니터링
watch -n 1 'redis-cli INFO stats | grep -E "keyspace_hits|keyspace_misses" | awk -F: "
{
  gsub(/[[:space:]]/, \"\", \$2)
  if (\$1 == \"keyspace_hits\") hits = \$2
  if (\$1 == \"keyspace_misses\") misses = \$2
}
END {
  total = hits + misses
  if (total > 0) printf \"Hit rate: %.1f%% (hits=%d, misses=%d)\\n\", hits*100/total, hits, misses
}"'

# Stampede 발생 시 miss율 급등 확인
```

---

## 📊 성능/비용 비교

```
Stampede 방지 방법별 비교:

방법             | 구현 복잡도   | 대기 지연    | DB 조회 횟수 | 응답 일관성
────────────────┼────────────┼────────────┼────────────┼────────────
TTL 지터         | 최하        | 없음        | 줄어들지만 0은 아님 | 높음
Mutex Lock      | 낮음        | Lock 대기   | 1회         | 낮음 (대기 중 nil 가능)
Stale-While-    | 중간        | 없음        | 백그라운드    | 중간 (구버전 잠시)
Revalidate      |            |            |            |
PER             | 높음        | 없음        | 분산 트리거   | 높음

Stampede 발생 시 DB 부하:

초당 10,000 요청, 캐시 재계산 100ms 가정:
  대책 없음:   10,000 × 0.1초 = 1,000건 동시 DB 쿼리
  TTL 지터:    ~100건 동시 쿼리 (10배 감소)
  Mutex Lock:  1건 (나머지 대기)
  PER:         ~10건 분산 트리거 (TTL 만료 전 선제 갱신)

PER beta 파라미터 효과:
  beta=0.5: 만료 직전에만 갱신 → DB 부하 적음, 만료 순간 소량 miss 가능
  beta=1.0: 균형 (권장)
  beta=2.0: 여유있게 미리 갱신 → DB 부하 약간 증가, miss 거의 없음
```

---

## ⚖️ 트레이드오프

```
Mutex Lock의 트레이드오프:
  장점: DB 조회 1회로 제한, 구현 단순
  단점: 락 대기 시간만큼 응답 지연
        락 홀더 장애 시 최대 EX 시간 동안 응답 없음

PER의 트레이드오프:
  장점: 대기 없음, miss 없음, DB 부하 분산
  단점: 구현 복잡도 높음, delta(재계산 시간) 측정 필요
        beta 튜닝 필요

가장 실용적인 조합:
  1. TTL 지터: 항상 적용 (무조건 추천, 비용 없음)
  2. 중요 데이터: Mutex Lock or PER 추가
  3. 읽기 트래픽 극히 높은 경우: 로컬 캐시(JVM 메모리) 레이어 추가

Stampede vs Hot Key 구분:
  Stampede: 순간적 동시 Miss (TTL 만료 순간)
  Hot Key:  지속적인 단일 키 집중 (01-hot-key.md 참고)
  → 둘 다 DB 폭발 유발, 해결책 다름
```

---

## 📌 핵심 정리

```
Cache Stampede 핵심:

발생 조건:
  인기 키 + TTL 만료 + 다수 동시 요청 → DB에 요청 폭발

해결책 1: TTL 지터 (가장 간단)
  TTL = base ± random(N)
  → 동시 만료 분산 → Stampede 약화

해결책 2: Mutex Lock (DB 조회 1회 보장)
  SET lock_key 1 NX EX 30  → 락 획득 경쟁
  락 획득: DB 조회 → 캐시 갱신 → 락 해제
  락 실패: 대기 후 캐시 재조회

해결책 3: PER (Probabilistic Early Recomputation)
  만료 전 확률적으로 미리 갱신
  조건: t_now - delta * beta * log(random()) >= t_expiry
  beta 클수록 더 일찍 갱신 시작
  Miss 자체를 없애는 가장 우아한 방법

실무 권장:
  기본: TTL 지터 (300 ± 30초)
  중요 데이터: Mutex Lock 추가
  극고트래픽: PER + 로컬 캐시 레이어
```

---

## 🤔 생각해볼 문제

**Q1.** TTL 지터를 적용해 만료를 분산시켰는데, 여전히 Cold Start 시(서비스 시작 직후) Stampede가 발생했다. 왜 지터가 Cold Start에는 효과가 없는가?

<details>
<summary>해설 보기</summary>

TTL 지터는 **기존 캐시의 만료 시점을 분산**하는 것이다. 하지만 Cold Start 시에는 캐시 자체가 없으므로, 모든 요청이 동시에 cache miss가 된다. 이 상황은 TTL 만료가 아니라 "캐시가 한 번도 채워지지 않은 상태"이므로 지터가 의미없다.

**Cold Start Stampede 해결책:**

1. **워밍업 스크립트 (예열):**
```bash
# 서비스 시작 전 인기 키를 미리 캐싱
warmup.py  # DB에서 인기 콘텐츠 조회 → Redis에 프리로드
```

2. **Mutex Lock 조합:**
```python
# 첫 번째 요청만 DB 조회, 나머지는 대기
get_with_lock("article:1", lambda: db.query())
```

3. **구 인스턴스 캐시 유지 (Rolling Deploy):**
```
배포 시 구 인스턴스를 즉시 종료하지 않고, 신규 인스턴스가 캐시를 채울 때까지 유지
```

4. **Canary Deploy 시 예열:**
```
트래픽 1% → 신규 인스턴스로 → 캐시 채워짐 → 100% 전환
```

Cold Start와 TTL 만료 Stampede는 다른 문제이므로, 해결책도 다르다.

</details>

---

**Q2.** Mutex Lock 구현에서 `redis.set(lock_key, "1", nx=True, ex=30)`의 `ex=30`(30초)이 너무 길면 어떤 문제가 생기고, 너무 짧으면 어떤 문제가 생기는가?

<details>
<summary>해설 보기</summary>

**너무 길 때 (ex=300 = 5분):**
- 락을 획득한 프로세스가 DB 조회 중 크래시 → 5분 동안 아무도 락 획득 불가
- 이 5분 동안 모든 요청이 대기하거나 None 반환
- 서비스 품질 심각하게 저하

**너무 짧을 때 (ex=1 = 1초):**
- DB 조회가 1초 이상 걸리는 경우 → 락 자동 해제 → 두 번째 프로세스가 락 획득
- 두 프로세스 동시 DB 조회 → 작은 규모 Stampede 재발
- 락 홀더가 DB 결과를 가져와 캐시 SET 했는데, 두 번째도 SET → 중복 SET (무해하지만 비효율)

**권장:**
- `ex`는 DB 조회 예상 최대 시간 × 3~5배로 설정
- 예: 일반 DB 쿼리 100ms → ex=3~5초
- 락 해제를 반드시 `finally` 블록에서 처리

```python
try:
    data = db.query()  # 100ms 예상
    cache.setex(key, ttl, data)
finally:
    cache.delete(lock_key)  # 크래시 안 나도 해제
```

</details>

---

**Q3.** PER에서 `beta = 1.0`, `delta = 0.1초`(DB 조회 100ms), TTL = 300초일 때, 캐시 만료 10초 전(잔여 TTL 10초)에 갱신이 트리거될 확률은 대략 얼마인가?

<details>
<summary>해설 보기</summary>

PER 조건: `t_now - delta * beta * log(random()) >= t_expiry`

`t_now - t_expiry = -10` (만료 10초 전)

`delta * beta = 0.1 * 1.0 = 0.1`

조건 정리:
`-10 - 0.1 * log(r) >= 0`
`-0.1 * log(r) >= 10`
`log(r) <= -100`
`r <= e^(-100) ≈ 3.7 × 10^-44`

**확률 ≈ 3.7 × 10^-44 (사실상 0%)**

10초 전에는 갱신 확률이 극히 낮다는 의미다.

만료 1초 전(`t_now - t_expiry = -1`)일 때:
`r <= e^(-10) ≈ 0.0000454`
**확률 ≈ 0.0045%**

만료 0.1초 전일 때:
`r <= e^(-1) ≈ 0.368`
**확률 ≈ 36.8%**

만료 순간(`t_expiry = t_now`)일 때:
`r <= e^0 = 1` → 항상 갱신 → **100%**

**결론:** PER은 TTL 잔여 시간이 `delta`에 근접할수록 갱신 확률이 급격히 증가한다. `delta = 0.1초`이면 만료 0.1초 전부터 갱신이 집중적으로 트리거된다.

</details>

---

<div align="center">

**[⬅️ 이전: 캐싱 전략](./01-caching-strategies.md)** | **[홈으로 🏠](../README.md)** | **[다음: Hot Key 문제 ➡️](./03-hot-key.md)**

</div>
