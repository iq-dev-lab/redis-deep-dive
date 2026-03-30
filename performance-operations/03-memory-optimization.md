# 메모리 최적화 실전 — 인코딩 임계값 튜닝

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `hash-max-listpack-entries`를 조정하면 실제로 메모리가 얼마나 변하는가?
- `MEMORY USAGE key`와 `DEBUG OBJECT key`의 차이는?
- `redis-cli --bigkeys`로 대형 키를 탐색하는 원리와 한계는?
- `OBJECT FREQ`로 접근 빈도를 분석하는 방법은?
- 메모리 단편화(fragmentation)가 높을 때 해결하는 방법은?
- 데이터 구조별 임계값 조정 시 성능 vs 메모리 트레이드오프는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Redis 메모리 최적화는 수십 MB에서 수 GB 차이를 만들 수 있다. 인코딩 임계값 하나 설정으로 동일 데이터를 절반 크기로 줄이기도 한다. 하지만 측정 없이 임계값을 변경하면 오히려 성능이 저하된다. MEMORY USAGE와 --bigkeys로 먼저 현황을 파악하고, 임계값 조정의 효과를 측정 후 적용해야 한다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 인코딩 임계값 조정 없이 메모리 초과

  상황: 유저 프로필 100만개 저장 (필드 10개 이하)
  설정: 기본 hash-max-listpack-entries 128 (조정 안 함)
  결과: listpack 유지 → 메모리 효율 최대 (아무 문제 없음)
  
  나쁜 상황: 필드가 200개로 늘어남
  결과: hashtable 전환 → 메모리 3~5배 폭발
  → 임계값 조정 타이밍을 놓침 → 설계 변경 필요

실수 2: --bigkeys 결과를 절대 수치로 오해

  redis-cli --bigkeys 결과:
    Biggest string found 'cache:data' with 50 bytes
  
  "50 bytes니까 bigkey 아니잖아?"
  → --bigkeys는 같은 타입 중 가장 큰 것만 보여줌
  → 50 bytes가 가장 크다 = 전체 String 키가 작다는 의미
  → 실제 문제는 Hash 타입에서 발생 중일 수 있음

실수 3: mem_fragmentation_ratio > 2를 방치

  INFO memory:
    used_memory: 1 GB (Redis가 사용 중이라 생각하는 메모리)
    used_memory_rss: 2.5 GB (OS가 할당한 실제 메모리)
    mem_fragmentation_ratio: 2.5 (비정상)
  
  "Redis가 1 GB만 쓰니까 괜찮겠지" → 실제로는 2.5 GB 점유
  → 다른 프로세스 메모리 부족 → OOM 위험
  → activedefrag 또는 재시작으로 해소 필요
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
메모리 최적화 순서:
  1. 현황 측정: INFO memory, MEMORY USAGE, --bigkeys
  2. 문제 키 식별: 타입별 상위 메모리 사용 키
  3. 인코딩 확인: OBJECT ENCODING
  4. 임계값 조정 (필요시): 변경 전후 MEMORY USAGE 비교
  5. 단편화 대응: mem_fragmentation_ratio 모니터링

인코딩 임계값 조정 기준:
  Hash 필드 수가 항상 50개 이하 → hash-max-listpack-entries 64 유지 OK
  Hash 필드 수가 200개 → Sharding으로 128개 이하로 분할 (임계값 올리기 전에)
  
  임계값 올리기 조건:
    HGET 빈도가 낮음 (O(N) 탐색 허용)
    데이터 크기가 작음 (listpack 이득 큼)
    측정으로 성능 저하 없음 확인

단편화 해소:
  mem_fragmentation_ratio 1.5~2.0: CONFIG SET activedefrag yes
  mem_fragmentation_ratio > 2.0: 점검 시간에 Replica Failover + Master 재시작
```

---

## 🔬 내부 동작 원리

### 1. 인코딩 임계값 조정의 메모리 효과

```
Hash 인코딩 임계값:
  hash-max-listpack-entries 128  (최대 필드 수)
  hash-max-listpack-value   64   (최대 값 크기, bytes)

  조건 둘 다 충족 시: listpack 인코딩 (메모리 효율)
  하나라도 초과 시: hashtable 인코딩 (메모리 비효율)

실제 메모리 차이 (필드 10개, 값 평균 20 bytes):
  listpack:  ~300 bytes per Hash
  hashtable: ~1,500 bytes per Hash (5배!)
  
  100만개 Hash:
    listpack:  300 MB
    hashtable: 1.5 GB (1.2 GB 추가 낭비)

다른 타입별 임계값:
  List:
    list-max-listpack-size -2  (기본: 8KB per 노드)
    
  Set:
    set-max-intset-entries 512   (정수 Set, intset 임계값)
    set-max-listpack-entries 128 (문자열 Set)
    set-max-listpack-value   64
    
  Sorted Set:
    zset-max-listpack-entries 128
    zset-max-listpack-value   64
```

### 2. MEMORY USAGE — 정확한 메모리 측정

```
MEMORY USAGE key [SAMPLES count]

반환값: 해당 키가 사용하는 실제 메모리 (bytes)
        키 이름 + 값 + 자료구조 오버헤드 + jemalloc 슬롯 포함

SAMPLES 파라미터:
  중첩 자료구조의 샘플링 깊이 (기본 1)
  복잡한 중첩 Hash/List: SAMPLES 5~10으로 더 정확하게 측정

예시:
  redis-cli SET "key" "hello"
  redis-cli MEMORY USAGE key        → ~56 bytes
  
  redis-cli HSET hash f1 v1 f2 v2
  redis-cli MEMORY USAGE hash       → ~120 bytes
  
  redis-cli MEMORY USAGE bigkey SAMPLES 5  → 더 정확한 측정

DEBUG OBJECT key 와의 차이:
  MEMORY USAGE:  실제 RAM 사용 (jemalloc 슬롯 포함)
  DEBUG OBJECT:  직렬화 크기 (RDB 저장 시 크기), 인코딩 정보 포함
  
  예시:
    MEMORY USAGE hash → 3,200 bytes (실제 RAM)
    DEBUG OBJECT hash → serializedlength:289  (RDB 압축 후 289 bytes)
    
  용도:
    메모리 sizing → MEMORY USAGE
    네트워크 전송/RDB 크기 예측 → DEBUG OBJECT serializedlength
```

### 3. redis-cli --bigkeys — 대형 키 탐색

```
동작 원리:
  내부적으로 SCAN을 실행해 모든 키를 순회
  각 키의 TYPE과 STRLEN/LLEN/HLEN/SCARD/ZCARD로 크기 측정
  타입별로 "가장 큰 키"를 추적하여 결과 출력

실행:
  redis-cli --bigkeys
  
  # -------- summary -------
  # Sampled 1234567 keys in the keyspace!
  # Total key length in bytes is 23456789 (avg len 19.00)
  
  # Biggest string found 'user:popular' with 1.2 MB
  # Biggest list found 'feed:viral' with 50000 items
  # Biggest hash found 'product:detail:1' with 512 fields
  # Biggest set found 'tags:all' with 100000 members
  # Biggest zset found 'ranking:global' with 1000000 members
  
  # String  70.12% (  865432 keys avg size 1234.56 bytes)
  # List     0.01% (      123 keys avg size 5000.00 items)
  # Hash    29.87% (  368765 keys avg size 15.20 fields)

한계:
  타입별로 가장 큰 것만 추출 (순위 1개만)
  크기 = 원소/필드 수 기준 (bytes 기준 아님)
    String: bytes 기준
    Hash/List/Set/ZSet: 원소/필드 수 기준 (MEMORY USAGE와 다름)

더 정확한 bigkey 탐색:
  # MEMORY USAGE 기반 (상위 N개 추출)
  redis-cli --no-auth-warning SCAN 0 COUNT 1000 | tail -n +2 | \
    xargs -I{} redis-cli MEMORY USAGE {} 2>/dev/null | \
    paste - - | sort -k2 -rn | head -20
  # 실제 메모리 기준 상위 20개 키
```

### 4. OBJECT FREQ — LFU 접근 빈도 분석

```
LFU 정책 설정 시 접근 빈도 추적:
  maxmemory-policy allkeys-lfu 또는 volatile-lfu

OBJECT FREQ key:
  반환값: LFU 카운터 (0~255)
  높을수록 자주 접근된 키

활용:
  # 가장 많이 접근된 키 찾기
  redis-cli --no-auth-warning SCAN 0 COUNT 1000 | tail -n +2 | \
    xargs -I{} sh -c 'echo "$(redis-cli OBJECT FREQ {}) {}"' | \
    sort -rn | head -10
  
  # Hot Key 탐지
  redis-cli --hotkeys  # LFU 정책 필요, 전체 키스페이스 스캔

OBJECT IDLETIME:
  마지막 접근 후 경과 시간 (초)
  LFU 정책에서는 사용 불가 (OBJECT FREQ 사용)
  
  Cold Key 탐지:
  redis-cli --no-auth-warning SCAN 0 COUNT 1000 | tail -n +2 | \
    xargs -I{} sh -c 'echo "$(redis-cli OBJECT IDLETIME {} 2>/dev/null) {}"' | \
    sort -rn | head -10
  # idle time이 긴 키 = 오래 접근 안 됨 = cold key = TTL 설정 고려
```

### 5. 메모리 단편화 — 진단과 해소

```
단편화 비율 확인:
  INFO memory
  mem_fragmentation_ratio = used_memory_rss / used_memory
  
  정상:   1.0~1.5
  주의:   1.5~2.0 (activedefrag 고려)
  심각:   > 2.0  (재시작 고려)
  위험:   < 1.0  (스왑 발생 중!)

단편화 원인:
  다양한 크기의 키/값을 계속 Set/Del 반복
  → jemalloc 크기 클래스 간 단편화 발생
  → 실제 데이터 1 GB인데 OS에는 2.5 GB 할당

해소 방법:

  방법 1: activedefrag (점진적 백그라운드 정리)
    redis-cli CONFIG SET activedefrag yes
    redis-cli CONFIG SET active-defrag-ignore-bytes 100mb
    redis-cli CONFIG SET active-defrag-threshold-lower 10
    redis-cli CONFIG SET active-defrag-threshold-upper 100
    
    → 백그라운드에서 단편화된 메모리를 점진적으로 정리
    → CPU 사용 약간 증가 (대부분 허용 범위)
    → 완전 해소까지 수 시간~수일

  방법 2: MEMORY PURGE (즉각 반환 시도)
    redis-cli MEMORY PURGE
    → jemalloc에게 미사용 메모리를 OS에 반환 요청
    → 이벤트 루프 잠시 블로킹 (주의)
    → 단편화 자체는 해소 안 됨

  방법 3: 재시작 (가장 확실)
    재시작 → 모든 메모리 새로 할당 → 단편화 없음
    Sentinel/Cluster: Replica Failover 후 구 Master 재시작
    → 무중단으로 단편화 해소 가능
```

---

## 💻 실전 실험

### 실험 1: 인코딩 임계값 변경 전후 메모리 비교

```bash
# 기본 설정으로 Hash 생성
redis-cli CONFIG SET hash-max-listpack-entries 128

redis-cli DEL small_hash big_hash

# listpack 범위 (128개 이하)
for i in $(seq 1 100); do
  redis-cli HSET small_hash "field$i" "value-with-padding-$i" > /dev/null
done
redis-cli OBJECT ENCODING small_hash  # listpack
SMALL_MEM=$(redis-cli MEMORY USAGE small_hash)

# hashtable 전환 (129개)
for i in $(seq 1 129); do
  redis-cli HSET big_hash "field$i" "value-with-padding-$i" > /dev/null
done
redis-cli OBJECT ENCODING big_hash   # hashtable
BIG_MEM=$(redis-cli MEMORY USAGE big_hash)

echo "100 필드 (listpack): $SMALL_MEM bytes"
echo "129 필드 (hashtable): $BIG_MEM bytes"
echo "비율: $(echo "scale=1; $BIG_MEM / $SMALL_MEM" | bc)x"

# 임계값 낮춰서 조기 전환 관찰
redis-cli CONFIG SET hash-max-listpack-entries 5
redis-cli DEL test_hash
for i in $(seq 1 6); do redis-cli HSET test_hash "f$i" "v$i" > /dev/null; done
redis-cli OBJECT ENCODING test_hash  # hashtable (6 > 5)
redis-cli MEMORY USAGE test_hash

# 복원
redis-cli CONFIG SET hash-max-listpack-entries 128
```

### 실험 2: --bigkeys와 MEMORY USAGE 비교

```bash
# 다양한 타입 키 생성
redis-cli SET large_string "$(python3 -c "print('x'*10000)")"
for i in $(seq 1 500); do redis-cli HSET big_hash "f$i" "v$i" > /dev/null; done
for i in $(seq 1 10000); do redis-cli SADD big_set $i > /dev/null; done

# --bigkeys로 탐색
redis-cli --bigkeys 2>/dev/null

# MEMORY USAGE로 실제 메모리 확인 (bytes 기준)
echo "=== 실제 메모리 사용 (bytes) ==="
for key in large_string big_hash big_set; do
  mem=$(redis-cli MEMORY USAGE $key)
  echo "$key: $mem bytes"
done
```

### 실험 3: 단편화 시뮬레이션과 해소

```bash
# 단편화 유발: 다양한 크기 키 반복 Set/Del
echo "=== 단편화 유발 중 ==="
for round in $(seq 1 3); do
  for size in 10 100 1000 50 500 5000 20 200 2000; do
    redis-cli SET "frag:key:$size:$round" "$(python3 -c "print('x'*$size)")" > /dev/null
  done
done
for round in $(seq 1 2); do
  for size in 10 100 1000 50 500 5000; do
    redis-cli DEL "frag:key:$size:$round" > /dev/null
  done
done

# 단편화 비율 확인
redis-cli INFO memory | grep -E "used_memory_human|used_memory_rss_human|mem_fragmentation_ratio"

# activedefrag 활성화
redis-cli CONFIG SET activedefrag yes
redis-cli CONFIG SET active-defrag-ignore-bytes 1mb
redis-cli CONFIG SET active-defrag-threshold-lower 5

# 잠시 대기 후 단편화 비율 변화 확인
sleep 10
redis-cli INFO memory | grep mem_fragmentation_ratio
```

### 실험 4: Cold Key 탐지 (LRU 기준)

```bash
# 다양한 접근 패턴 생성
redis-cli SET hot_key "hot"
for i in $(seq 1 1000); do redis-cli GET hot_key > /dev/null; done

redis-cli SET warm_key "warm"
for i in $(seq 1 10); do redis-cli GET warm_key > /dev/null; done

redis-cli SET cold_key "cold"
# cold_key는 접근 안 함

sleep 5

# OBJECT IDLETIME으로 cold key 탐지
echo "=== 마지막 접근 이후 경과 시간 ==="
for key in hot_key warm_key cold_key; do
  idle=$(redis-cli OBJECT IDLETIME $key)
  echo "$key: ${idle}초 전에 마지막 접근"
done
```

---

## 📊 성능/비용 비교

```
인코딩별 메모리 비교 (동일 데이터):

Hash 10 필드, 값 평균 20 bytes:
  listpack:   ~350 bytes   (포인터 오버헤드 없음)
  hashtable:  ~1,700 bytes (버킷 + 포인터 + SDS)
  비율: 4.8배

Sorted Set 50 원소, 멤버 평균 10 bytes:
  listpack:   ~2,100 bytes
  skiplist:   ~8,500 bytes
  비율: 4.0배

Set 200 정수:
  intset:     ~1,620 bytes (정수 배열)
  hashtable:  ~12,000 bytes
  비율: 7.4배

임계값 조정 시 성능 vs 메모리:

                  임계값 높임         임계값 낮춤
──────────────────────────────────────────────────────
listpack 유지 범위  확대              축소
메모리 효율         높음              낮음
탐색 복잡도         O(N) 증가         O(1) 빠름
적합               메모리 절약 우선   성능 우선
```

---

## ⚖️ 트레이드오프

```
임계값 높이기 (hash-max-listpack-entries 256 등):
  장점: listpack 유지 범위 확대 → 메모리 절약
  단점: O(N) 탐색 범위 증가 → HGET이 최대 O(256)
  권장: HGET 빈도 낮고 데이터 작을 때

임계값 낮추기:
  장점: 빠르게 hashtable 전환 → O(1) 탐색
  단점: 메모리 증가
  권장: HGET 빈도 매우 높을 때 (Hot Key 패턴)

activedefrag vs 재시작:
  activedefrag: 무중단, 점진적 해소, CPU 증가
  재시작: 즉각 해소, 순간 서비스 중단 (Sentinel/Cluster 환경에서 무중단 가능)

MEMORY USAGE 비용:
  O(1) for String/int
  O(N) SAMPLES 깊이까지 순회 (중첩 자료구조)
  빈번한 호출 주의 → 가끔 샘플링 또는 배치 분석 권장
```

---

## 📌 핵심 정리

```
메모리 최적화 핵심:

측정 도구:
  MEMORY USAGE key       → 실제 RAM 바이트
  DEBUG OBJECT key       → 직렬화 크기, 인코딩
  redis-cli --bigkeys    → 타입별 최대 크기 키
  OBJECT ENCODING key    → 현재 인코딩
  OBJECT FREQ key        → LFU 접근 빈도

인코딩 임계값 (메모리 최적화):
  hash-max-listpack-entries: 128 (기본)
  hash-max-listpack-value: 64 bytes
  zset-max-listpack-entries: 128
  set-max-intset-entries: 512
  → 초과 시 listpack → hashtable 전환 (메모리 3~8배 증가)
  → 설계 단계에서 임계값 내 유지 전략 수립

단편화 관리:
  mem_fragmentation_ratio:
    1.0~1.5: 정상
    > 2.0:   심각 → activedefrag 또는 재시작
    < 1.0:   스왑 중 → 즉각 대응

최적화 순서:
  1. MEMORY USAGE / --bigkeys로 현황 파악
  2. 대형 키 처리 (분할, TTL 설정, 정리)
  3. 인코딩 임계값 검토 (listpack 유지 전략)
  4. 단편화 대응 (activedefrag)
```

---

## 🤔 생각해볼 문제

**Q1.** `redis-cli --bigkeys`가 타입별 "가장 큰 1개"만 보여준다. 실제 메모리를 가장 많이 쓰는 상위 N개 키를 찾는 방법은?

<details>
<summary>해설 보기</summary>

`--bigkeys`의 한계: 각 타입에서 가장 큰 것만 1개씩 표시한다. 두 번째, 세 번째 대형 키는 보이지 않는다.

**실제 메모리 기준 상위 N개 탐색:**

```bash
# 방법 1: SCAN + MEMORY USAGE (정확, 느림)
redis-cli SCAN 0 COUNT 100 | tail -n +2 | while read key; do
    mem=$(redis-cli MEMORY USAGE "$key" 2>/dev/null)
    echo "$mem $key"
done | sort -rn | head -20

# 방법 2: Redis 7.0+ LMPOP / OBJECT HELP 활용
# 방법 3: Lua 스크립트로 배치 처리
redis-cli EVAL "
local keys = redis.call('SCAN', '0', 'COUNT', '100')
local result = {}
for i, k in ipairs(keys[2]) do
    local mem = redis.call('MEMORY', 'USAGE', k)
    table.insert(result, {k, mem})
end
return result
" 0

# 방법 4: redis-memory-for-key (외부 도구)
pip install rdbtools
rdb --command memory dump.rdb | sort -t, -k4 -rn | head -20
# RDB 파일 기반 오프라인 분석 (운영 영향 없음)
```

**권장 접근:**
- 소규모 환경: SCAN + MEMORY USAGE 루프
- 대규모 환경: RDB 파일 오프라인 분석 (`rdbtools`, `redis-rdb-tools`)
- 주기적 분석: Prometheus + redis_exporter 메트릭 활용

</details>

---

**Q2.** `mem_fragmentation_ratio`가 0.7이다. 이것이 단편화가 적은 좋은 상태인가?

<details>
<summary>해설 보기</summary>

**전혀 아니다. 매우 위험한 상태다.**

```
mem_fragmentation_ratio = used_memory_rss / used_memory

0.7 = used_memory_rss / used_memory
→ used_memory_rss < used_memory
→ OS가 Redis에 할당한 물리 메모리 < Redis가 사용 중이라 생각하는 메모리
```

이것은 **메모리 스왑(Swap)이 발생 중**이라는 의미다:
- Redis의 일부 데이터가 RAM에서 디스크 스왑 파티션으로 내려감
- `used_memory`(Redis 관점)는 전체 데이터 크기를 반영
- `used_memory_rss`(OS 관점)는 실제 RAM에 있는 부분만 반영
- ratio < 1.0 = 일부가 RAM 밖에 있음

**증상:**
- Redis 응답 시간 수십 ms → 수 초로 폭등
- 스왑 디스크 I/O 급증

**즉각 대응:**
```bash
# 스왑 사용량 확인
cat /proc/$(pgrep redis-server)/status | grep VmSwap
# VmSwap:  1,024,000 kB  ← 1 GB 스왑 중!

# 대응
# 1. maxmemory 낮춤
redis-cli CONFIG SET maxmemory 4gb  # 현재 사용량보다 낮게

# 2. eviction 정책으로 메모리 확보
redis-cli CONFIG SET maxmemory-policy allkeys-lru

# 3. 서버 메모리 증설 또는 데이터 이전
```

</details>

---

**Q3.** `hash-max-listpack-entries`를 128에서 256으로 늘렸다. 기존에 이미 hashtable로 전환된 Hash들은 listpack으로 돌아오는가?

<details>
<summary>해설 보기</summary>

**돌아오지 않는다.** Redis 인코딩 전환은 단방향이다.

- listpack → hashtable: 임계값 초과 시 자동 전환 (진행)
- hashtable → listpack: **자동 역전환 없음**

임계값을 128 → 256으로 올려도:
- 기존 hashtable 인코딩 Hash: 그대로 hashtable 유지
- 새로 추가되는 Hash: 256개 이하라면 listpack으로 시작

**기존 Hash를 listpack으로 되돌리는 유일한 방법:**
```bash
# 해당 Hash를 완전히 재생성
redis-cli HGETALL old_hash | while read k; read v; do
    redis-cli HSET new_hash "$k" "$v"
done
redis-cli RENAME new_hash old_hash
```

또는 Redis를 재시작하면 RDB에서 로드 시 현재 임계값 기준으로 인코딩을 재결정한다:
```bash
# 임계값 변경 후 재시작
# redis.conf: hash-max-listpack-entries 256
redis-cli SHUTDOWN SAVE
redis-server redis.conf
# → 로드 시 필드 수 ≤ 256인 Hash가 listpack으로 인코딩됨
```

재시작이 어려우면, 해당 Hash 키들을 순차적으로 DEL → 재삽입하는 마이그레이션 스크립트를 작성해야 한다.

</details>

---

<div align="center">

**[⬅️ 이전: Lua 스크립팅](./02-lua-scripting.md)** | **[홈으로 🏠](../README.md)** | **[다음: Redis 모니터링 ➡️](./04-monitoring.md)**

</div>
