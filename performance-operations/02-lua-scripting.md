# Lua 스크립트 — 원자적 복합 연산

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Lua 스크립트가 `MULTI/EXEC` 없이도 원자적인 이유는?
- `EVAL`과 `EVALSHA`의 차이는 무엇이고, 왜 EVALSHA가 효율적인가?
- Lua 스크립트 실행 중 Redis 이벤트 루프가 블로킹되는 주의사항은?
- `KEYS[]`와 `ARGV[]`를 분리하는 이유와 Cluster 환경에서의 제약은?
- Lua에서 Redis 명령어 결과를 처리하는 방법은?
- `redis.pcall` vs `redis.call`의 차이는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Redis에서 "잔액 확인 후 차감"처럼 조건부 복합 연산을 원자적으로 실행하려면 Lua 스크립트가 유일한 방법이다. MULTI/EXEC는 조건 분기가 불가능하고, 두 번의 별도 명령어는 Race Condition이 생긴다. 하지만 잘못 작성한 Lua 스크립트는 수초 동안 이벤트 루프를 독점해 전체 Redis를 마비시킨다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 조건부 연산을 별도 명령어로 구현 (Race Condition)

  코드:
    balance = redis.get("balance")      # GET
    if int(balance) >= amount:
        redis.decrby("balance", amount)  # DECRBY
  
  문제:
    GET과 DECRBY 사이에 다른 스레드가 DECRBY 실행
    → 잔액이 100인데 50씩 두 번 차감 = -0 (둘 다 GET 시 100 확인 후 각각 차감)
    → Race Condition으로 음수 잔액 가능

실수 2: 긴 루프가 있는 Lua 스크립트

  스크립트:
    for i = 1, 1000000 do
        redis.call('INCR', 'counter')
    end
  
  결과:
    이벤트 루프 수 초 동안 독점
    모든 클라이언트 요청 타임아웃
    lua-time-limit(5000ms) 초과 시 SCRIPT KILL로만 종료 가능

실수 3: EVAL에 스크립트 전체 텍스트를 매번 전송

  매 요청마다:
    redis.eval("""
    local balance = tonumber(redis.call('GET', KEYS[1]))
    if balance >= tonumber(ARGV[1]) then
        redis.call('DECRBY', KEYS[1], ARGV[1])
        return 1
    end
    return 0
    """, 1, key, amount)
  
  문제:
    매번 스크립트 전체를 네트워크로 전송 (불필요한 대역폭 낭비)
    해결: EVALSHA로 SHA1 해시만 전송
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
Lua 스크립트로 원자적 조건부 연산:
  lua_script = """
  local balance = tonumber(redis.call('GET', KEYS[1]))
  if balance == nil then return -1 end  -- 키 없음
  if balance >= tonumber(ARGV[1]) then
      redis.call('DECRBY', KEYS[1], ARGV[1])
      return 1  -- 성공
  end
  return 0  -- 잔액 부족
  """
  result = redis.eval(lua_script, 1, "balance:user:1", "100")
  # -1: 키 없음, 0: 잔액 부족, 1: 성공

EVALSHA로 네트워크 최적화:
  # 최초 1회: 스크립트 등록
  sha = redis.script_load(lua_script)
  
  # 이후: SHA1만 전송 (20 bytes)
  result = redis.evalsha(sha, 1, "balance:user:1", "100")

스크립트 설계 원칙:
  짧게 유지: 목표 1ms 이내
  루프 제한: 대량 데이터 처리 금지
  오류 처리: redis.pcall 사용 (예외로 인한 이벤트 루프 중단 방지)
  KEYS[] 사용: Cluster 호환성
```

---

## 🔬 내부 동작 원리

### 1. Lua 스크립트의 원자성 보장

```
Redis 이벤트 루프 + Lua 실행:

  이벤트 루프:
    ① 소켓 읽기 (EVAL 명령어 수신)
    ② 명령어 처리: Lua 인터프리터 실행
       ┌─────────────────────────────────────────┐
       │  Lua 스크립트 실행 (이벤트 루프 독점)          │
       │  redis.call('GET', key)   ← Redis 내부   │
       │  if balance >= amount ... ← Lua 로직     │
       │  redis.call('DECRBY', key, amount)      │
       └─────────────────────────────────────────┘
    ③ 다음 이벤트 처리

  원자성의 근거:
    Lua 스크립트 실행 중 Redis 이벤트 루프는 다른 명령어를 처리하지 않음
    → GET 결과를 확인하고 DECRBY를 실행하는 사이에
      다른 클라이언트가 끼어들 수 없음
    → 완벽한 원자성 보장

MULTI/EXEC와의 차이:
  MULTI/EXEC:
    조건부 실행 불가 (큐잉 후 일괄 실행)
    각 명령어 결과를 확인해서 다음 명령어 조건 변경 불가
  
  Lua 스크립트:
    GET 결과를 Lua 변수로 받아 조건문 사용 가능
    훨씬 유연한 원자적 복합 연산
```

### 2. EVAL 문법과 KEYS/ARGV

```
EVAL script numkeys [key [key ...]] [arg [arg ...]]

  script:   Lua 스크립트 문자열
  numkeys:  KEYS 배열의 크기
  key...:   KEYS[1], KEYS[2]... 로 참조
  arg...:   ARGV[1], ARGV[2]... 로 참조

예시:
  EVAL "return redis.call('SET', KEYS[1], ARGV[1])" 1 mykey myvalue
       │                                             │  │     │
       └── 스크립트                                    │  └─────┴── ARGV[1]="myvalue"
                                          numkeys=1  │
                                                     └── KEYS[1]="mykey"

왜 KEYS와 ARGV를 분리하는가?
  Redis Cluster 호환성:
    KEYS[]에 있는 키들의 슬롯을 Redis가 분석
    → 같은 슬롯 키들에 대한 스크립트만 단일 노드에서 실행 가능
    → Cluster에서 다른 슬롯 키를 KEYS[]에 넣으면 오류
  
  ARGV[]는 슬롯 계산에 사용되지 않음 (값, 설정 등)

Lua에서 Redis 명령어 호출:
  redis.call('GET', KEYS[1])    ← 오류 시 예외 발생 (스크립트 중단)
  redis.pcall('GET', KEYS[1])   ← 오류 시 오류 테이블 반환 (계속 실행)

redis.call vs redis.pcall:
  redis.call: 명령어 실패 시 Lua 예외 → 스크립트 중단 + 클라이언트에 오류 반환
  redis.pcall: 명령어 실패 시 {err="..."} 테이블 반환 → 스크립트 계속 실행
  
  일반적으로 redis.pcall 권장 (안전한 오류 처리)
```

### 3. EVAL vs EVALSHA — 네트워크 최적화

```
EVAL (매번 스크립트 전송):
  클라이언트 → Redis: EVAL "local balance = ..." 1 key amount
  전송 크기: 스크립트 길이만큼 (100~수 KB)
  
  Redis 내부:
    스크립트 SHA1 계산 (캐시 키)
    스크립트 캐시에 없으면 컴파일 + 캐시
    스크립트 캐시에 있으면 캐시에서 실행

EVALSHA (SHA1만 전송):
  SCRIPT LOAD로 사전 등록:
    redis-cli SCRIPT LOAD "local balance = ..."
    → SHA1 반환: "a42059b356c875f0717db19a51f6aaca9ae659ea"
  
  이후 실행:
    EVALSHA a42059b356c875f0717db19a51f6aaca9ae659ea 1 key amount
    전송 크기: 명령어 + 40자(SHA1) = ~50 bytes (스크립트 대비 소형)

EVALSHA 오류 처리:
  스크립트가 캐시에 없으면:
    NOSCRIPT No matching script. Please use EVAL.
  
  Spring/Jedis에서 자동 폴백:
    EVALSHA 실패 → 자동으로 EVAL 재시도
    → 개발자가 직접 처리할 필요 없음

Script 캐시 관리:
  SCRIPT EXISTS sha1 → 캐시 여부 확인
  SCRIPT FLUSH       → 캐시 전체 삭제 (BGREWRITEAOF 후 권장)
  
  캐시 지속성:
    Redis 재시작 시 스크립트 캐시 초기화
    → 재시작 후 EVALSHA 사용 전 SCRIPT LOAD 재실행 필요
    → 또는 EVAL 폴백 로직 구현
```

### 4. lua-time-limit — 이벤트 루프 보호

```
lua-time-limit 설정 (기본 5000ms = 5초):
  Lua 스크립트가 이 시간을 초과하면 "느린 스크립트" 경고
  이 시간 동안 다른 클라이언트는 대기

  주의: lua-time-limit 초과 시 즉시 중단되지 않음!
        → 다른 클라이언트가 SCRIPT KILL 명령어를 보낼 수 있게 해줌

SCRIPT KILL:
  현재 실행 중인 느린 Lua 스크립트를 중단
  단, 스크립트가 이미 데이터를 수정했으면 SCRIPT KILL 불가
  → 데이터 일관성 보호 (부분 실행 방지)
  → 이 경우 SHUTDOWN NOSAVE로 Redis 재시작 필요 (최후 수단)

느린 스크립트 방지:
  ① 루프에서 대량 데이터 처리 금지
  ② 최대 처리 수 제한
  ③ 시간 측정 후 초과 시 중단

  -- 안전한 루프 예시
  local count = 0
  local MAX = 1000  -- 최대 1,000개
  while count < MAX do
      local key = redis.call('RPOP', KEYS[1])
      if not key then break end
      -- 처리
      count = count + 1
  end
  return count
```

### 5. 실용적인 Lua 스크립트 패턴

```
패턴 1: 원자적 잔액 차감
  local balance = tonumber(redis.call('GET', KEYS[1]))
  if balance == nil then return {err="KEY_NOT_FOUND"} end
  local amount = tonumber(ARGV[1])
  if balance < amount then return {err="INSUFFICIENT_BALANCE"} end
  redis.call('DECRBY', KEYS[1], amount)
  return redis.call('GET', KEYS[1])  -- 차감 후 잔액 반환

패턴 2: 분산 락 해제 (소유자 확인)
  if redis.call('GET', KEYS[1]) == ARGV[1] then
      return redis.call('DEL', KEYS[1])
  else
      return 0
  end

패턴 3: Rate Limiting (초당 요청 제한)
  local current = redis.call('INCR', KEYS[1])
  if current == 1 then
      redis.call('EXPIRE', KEYS[1], ARGV[2])  -- TTL 설정 (첫 요청 시만)
  end
  if current > tonumber(ARGV[1]) then
      return 0  -- 제한 초과
  end
  return 1  -- 허용

패턴 4: 원자적 Set + TTL 갱신 (캐시 갱신 전략)
  redis.call('SET', KEYS[1], ARGV[1])
  redis.call('EXPIRE', KEYS[1], ARGV[2])
  return 1
  -- 참고: Redis 2.6+에서는 SET key val EX n으로 단일 명령어 가능
  --       Lua 스크립트 필요 없음 (이 패턴은 교육 목적)
```

---

## 💻 실전 실험

### 실험 1: Race Condition 재현 vs Lua 해결

```bash
# 잔액 설정
redis-cli SET balance 100

# Race Condition 발생하는 코드 시뮬레이션 (두 개 동시)
# (실제로는 sleep 없이 거의 동시에 실행)
(
  BAL=$(redis-cli GET balance)
  echo "Thread1 읽은 잔액: $BAL"
  sleep 0.01  # 처리 시간 시뮬레이션
  redis-cli DECRBY balance 80 > /dev/null
  echo "Thread1 80 차감"
) &

(
  BAL=$(redis-cli GET balance)
  echo "Thread2 읽은 잔액: $BAL"
  sleep 0.01
  redis-cli DECRBY balance 80 > /dev/null
  echo "Thread2 80 차감"
) &
wait
redis-cli GET balance  # 음수 가능! (Race Condition)

# Lua 스크립트로 해결
redis-cli SET balance 100
SCRIPT="
local balance = tonumber(redis.call('GET', KEYS[1]))
local amount = tonumber(ARGV[1])
if balance >= amount then
    redis.call('DECRBY', KEYS[1], amount)
    return 1
end
return 0"

(redis-cli EVAL "$SCRIPT" 1 balance 80 && echo "Thread1 성공") &
(redis-cli EVAL "$SCRIPT" 1 balance 80 && echo "Thread2 성공") &
wait
redis-cli GET balance  # 20 (하나만 성공) or 100 (둘 다 실패)
# → 절대 음수가 되지 않음
```

### 실험 2: EVALSHA 등록 및 사용

```bash
# 스크립트 등록
SCRIPT="
local balance = tonumber(redis.call('GET', KEYS[1]))
if balance == nil then return -1 end
if balance >= tonumber(ARGV[1]) then
    redis.call('DECRBY', KEYS[1], ARGV[1])
    return 1
end
return 0"

SHA=$(redis-cli SCRIPT LOAD "$SCRIPT")
echo "SHA: $SHA"

# SHA 캐시 확인
redis-cli SCRIPT EXISTS $SHA  # 1 (존재)

# EVALSHA로 실행
redis-cli SET balance 100
redis-cli EVALSHA $SHA 1 balance 30  # 1 (성공)
redis-cli GET balance                 # 70
redis-cli EVALSHA $SHA 1 balance 100  # 0 (잔액 부족)
redis-cli GET balance                 # 70 (변경 없음)
```

### 실험 3: Rate Limiting 구현

```bash
# Rate Limit: 초당 5회 제한
RATE_LIMIT_SCRIPT="
local current = redis.call('INCR', KEYS[1])
if current == 1 then
    redis.call('EXPIRE', KEYS[1], ARGV[2])
end
if current > tonumber(ARGV[1]) then
    return 0
end
return current"

SHA=$(redis-cli SCRIPT LOAD "$RATE_LIMIT_SCRIPT")

USER="user:123"
LIMIT=5
WINDOW=1  # 1초

for i in $(seq 1 8); do
  result=$(redis-cli EVALSHA $SHA 1 "ratelimit:$USER" $LIMIT $WINDOW)
  if [ "$result" -gt 0 ]; then
    echo "요청 $i: 허용 (count=$result)"
  else
    echo "요청 $i: 거부 (제한 초과)"
  fi
done
```

### 실험 4: lua-time-limit 동작 확인

```bash
# 느린 Lua 스크립트 (주의: 테스트 환경에서만!)
redis-cli CONFIG SET lua-time-limit 1000  # 1초로 단축

# 느린 스크립트 실행 (별도 터미널에서)
# redis-cli EVAL "local i = 0; while i < 100000000 do i = i + 1 end; return i" 0

# 다른 터미널에서
# redis-cli PING  # 스크립트 실행 중 응답 없음
# redis-cli SCRIPT KILL  # 스크립트 강제 종료

# 스크립트 실행 확인
redis-cli INFO server | grep lua
redis-cli CONFIG SET lua-time-limit 5000  # 기본값 복원
```

---

## 📊 성능/비용 비교

```
EVAL vs EVALSHA 네트워크 비용:

스크립트 크기     EVAL 전송 크기  EVALSHA 전송 크기  절약
──────────────┼──────────────┼─────────────────┼──────
100 bytes      ~100 bytes     ~50 bytes (SHA40)  50%
500 bytes      ~500 bytes     ~50 bytes          90%
2,000 bytes    ~2,000 bytes   ~50 bytes          97.5%

Lua 스크립트 컴파일 오버헤드:
  최초 EVAL: 컴파일 + 캐시 (수십 μs)
  이후 EVAL or EVALSHA: 캐시에서 실행 (수 μs)
  → 동일 스크립트 반복 실행: EVALSHA 권장

원자적 연산별 비교:

방식             | 원자성      | 조건 실행   | 구현 복잡도   | 성능
────────────────┼───────────┼───────────┼────────────┼──────
개별 명령어        | 없음       | 가능       | 낮음        | 최대
WATCH+MULTI/EXEC| 낙관적 잠금  | 제한적      | 중간        | 중간
Lua 스크립트      | 완전 원자성  | 완전 가능    | 높음        | 중간
```

---

## ⚖️ 트레이드오프

```
Lua 스크립트의 장단점:

장점:
  조건부 원자적 실행 (GET-조건확인-MODIFY)
  Redis 왕복 횟수 감소 (여러 명령어를 1번 RTT로)
  EVALSHA로 대역폭 최소화

단점:
  이벤트 루프 독점 (스크립트 실행 중 다른 요청 대기)
  디버깅 어려움 (Redis에서 Lua 실행 환경 제한)
  스크립트 길어지면 유지보수 어려움
  Cluster에서 모든 KEYS[]가 같은 슬롯이어야 함

MULTI/EXEC 대비 Lua:
  MULTI/EXEC 우선: 간단한 연속 실행 (조건 없음)
  Lua 사용:        조건부 로직, Read-Modify-Write

lua-time-limit 주의:
  스크립트가 블록되는 시간 = 전체 Redis 정지 시간
  → 짧고 빠른 스크립트만 사용
  → 대량 데이터 처리는 배치로 분할
```

---

## 📌 핵심 정리

```
Lua 스크립트 핵심:

원자성:
  스크립트 전체가 이벤트 루프 독점
  GET + 조건 확인 + MODIFY를 원자적으로 실행 가능
  Race Condition 완전 방지

EVAL vs EVALSHA:
  EVAL: 스크립트 전체 전송 (최초 또는 캐시 없을 때)
  EVALSHA: SHA1(40자)만 전송 (반복 실행 시 권장)
  SCRIPT LOAD로 사전 등록 → SHA1 획득

KEYS[] vs ARGV[]:
  KEYS[]: 키 이름 (Cluster 슬롯 계산 대상)
  ARGV[]: 값/파라미터 (슬롯 계산 제외)

주의사항:
  lua-time-limit (기본 5000ms): 초과 시 SCRIPT KILL 가능
  긴 루프 금지 → 이벤트 루프 마비 위험
  redis.pcall 사용 → 오류 처리 안전

실용 패턴:
  잔액 차감, 분산 락 해제, Rate Limiting
  → 모두 GET + 조건 + MODIFY 패턴
  → Lua 스크립트의 핵심 용도
```

---

## 🤔 생각해볼 문제

**Q1.** Lua 스크립트 안에서 `redis.call('KEYS', '*')`를 실행하면 어떤 일이 발생하는가?

<details>
<summary>해설 보기</summary>

**이중 블로킹이 발생한다.**

Lua 스크립트 자체가 이미 이벤트 루프를 독점하고 있는 상태에서, 그 안에서 `KEYS *`(O(N) 명령어)를 실행하면:

1. Lua 스크립트: 이벤트 루프 독점
2. `KEYS *`: 전체 키 순회 (O(N))

두 블로킹이 중첩되어 `이벤트 루프 독점 시간 = Lua 오버헤드 + KEYS 실행 시간`이 된다.

100만 키 환경에서 이 스크립트를 실행하면 수백 ms 동안 Redis가 완전 정지한다.

**실무 규칙:**
- Lua 스크립트 안에서 O(N) 명령어(KEYS, SMEMBERS, HGETALL 등) 절대 금지
- 특히 크기가 예측 불가능한 자료구조에 대한 전체 조회 금지

**올바른 패턴:** Lua 스크립트는 O(1) 또는 O(log N) 명령어만 사용하고, 대량 데이터 처리는 애플리케이션 레벨에서 SCAN으로 분할 처리한다.

</details>

---

**Q2.** `redis.call('SET', KEYS[1], ARGV[1])`과 `redis.pcall('SET', KEYS[1], ARGV[1])`의 동작이 다른 시나리오를 구체적으로 설명하라.

<details>
<summary>해설 보기</summary>

**타입 오류 시나리오:**
```lua
-- KEYS[1]이 Hash 타입인데 SET을 시도
redis.call('HSET', KEYS[1], 'field', 'value')  -- Hash로 만듦
redis.call('SET', KEYS[1], 'string_value')      -- 타입 충돌!
```

**redis.call 동작:**
- `WRONGTYPE Operation against a key holding the wrong kind of value` 예외 발생
- 스크립트 즉시 중단
- 클라이언트에게 오류 반환
- `SET` 이후의 코드 실행 안 됨

**redis.pcall 동작:**
```lua
local ok, err = redis.pcall('SET', KEYS[1], 'string_value')
if err then
    -- 오류 처리 가능
    redis.log(redis.LOG_WARNING, err['err'])
    return {err = err['err']}  -- 오류를 클라이언트에 반환
end
-- 계속 실행...
```
- 오류 발생해도 스크립트 계속 실행
- 오류 내용을 테이블로 받아 처리 가능

**실무 권장:**
- 오류가 절대 안 나야 하는 핵심 연산: `redis.call` (즉각 실패로 버그 조기 발견)
- 오류가 발생할 수 있는 외부 입력 처리: `redis.pcall` (안전한 오류 핸들링)

</details>

---

**Q3.** Redis Cluster 환경에서 Lua 스크립트로 두 개의 다른 슬롯에 있는 키를 원자적으로 처리하려면 어떻게 해야 하는가?

<details>
<summary>해설 보기</summary>

**직접 방법: hash-tag로 같은 슬롯에 배치**

```lua
-- KEYS[1] = "{user:1}:balance", KEYS[2] = "{user:1}:history"
-- 둘 다 {user:1} 태그 → 같은 슬롯 → 단일 노드에서 처리 가능

local balance = tonumber(redis.call('GET', KEYS[1]))
local amount = tonumber(ARGV[1])
if balance >= amount then
    redis.call('DECRBY', KEYS[1], amount)
    redis.call('LPUSH', KEYS[2], ARGV[2])  -- 거래 이력
    redis.call('LTRIM', KEYS[2], 0, 99)    -- 최근 100건만
    return 1
end
return 0
```

**불가능한 경우 (다른 슬롯):**
- `{user:1}:balance`와 `{order:123}:status`는 다른 슬롯
- 이를 단일 Lua 스크립트로 처리하면 `CROSSSLOT` 오류

**대안:**
1. **hash-tag 재설계:** 관련 키들을 같은 tag 아래 배치
2. **2-Phase 처리:** 각 슬롯 노드에서 별도 처리 + 애플리케이션 레벨 보상
3. **단일 인스턴스 Redis:** Cluster 없이 단일 노드 사용 (스케일 제한 수용)

Cluster 환경에서 Lua 스크립트를 사용하려면, 처음부터 관련 키들을 같은 슬롯에 배치하도록 키 설계를 해야 한다.

</details>

---

<div align="center">

**[⬅️ 이전: SLOWLOG와 병목 찾기](./01-slowlog-bottleneck.md)** | **[홈으로 🏠](../README.md)** | **[다음: 메모리 최적화 ➡️](./03-memory-optimization.md)**

</div>
