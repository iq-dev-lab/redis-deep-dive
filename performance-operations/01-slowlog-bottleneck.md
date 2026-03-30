# SLOWLOG와 병목 찾기

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `slowlog-log-slower-than`은 무엇을 기준으로 측정하는가?
- `SLOWLOG GET N`의 출력을 어떻게 읽는가?
- `KEYS *`, `SMEMBERS`, `LRANGE 0 -1`이 왜 단일 스레드를 블로킹하는가?
- `SCAN` 커서 방식이 O(N) 명령어보다 안전한 이유는?
- `commandstats`로 명령어별 누적 지연을 분석하는 방법은?
- SLOWLOG에 잡히지 않는 지연(네트워크, 커넥션 풀 등)은 어떻게 진단하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Redis 응답이 갑자기 느려졌을 때 원인은 두 가지다 — Redis 자체가 느린 것과, Redis가 빠른데 다른 요인(네트워크, 커넥션 부족)이 느린 것. SLOWLOG는 전자를 진단하는 핵심 도구다. O(N) 명령어가 이벤트 루프를 수백 ms 동안 독점하는 현상을 데이터로 증명하지 못하면, 엉뚱한 곳에서 원인을 찾게 된다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: SLOWLOG 확인 없이 서버 스펙 업그레이드

  현상: 5분마다 Redis 응답 시간 500ms 이상
  대응: Redis 인스턴스를 r5.xlarge → r5.2xlarge로 업그레이드
  결과: 여전히 5분마다 스파이크 발생, 비용만 증가

  근본 원인 (SLOWLOG에 있었던 것):
    SLOWLOG GET 5
    # usec: 450,000 (450ms!)
    # 명령어: KEYS user:*

  배치 스크립트가 5분마다 KEYS user:* 실행
  → 1M 키 → O(N) 순회 → 이벤트 루프 블로킹 450ms
  → SCAN으로 교체 → 문제 즉시 해소

실수 2: slowlog-log-slower-than 기본값(10ms)으로 중간 지연 놓침

  기본값: 10,000 μs = 10 ms
  실제 문제 명령어: 2~5 ms (10ms 미만이라 SLOWLOG 미기록)
  → 초당 수천 번 2~5ms 명령어 = 이벤트 루프 점유 증가
  → 응답 지연이 쌓이지만 SLOWLOG에 아무것도 없음

  해결: 1ms(1,000 μs)로 낮춰 더 세밀하게 탐지

실수 3: MONITOR로 운영 환경 디버깅

  MONITOR 명령어: 모든 명령어를 실시간 출력
  → Redis CPU 2배 이상 증가, 응답 지연 급증
  → 운영 환경에서 수 분 이상 사용 시 서비스 영향
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
SLOWLOG 우선 확인:
  redis-cli SLOWLOG GET 20    # 최근 20개 느린 명령어
  redis-cli SLOWLOG LEN       # 현재 기록된 수
  redis-cli SLOWLOG RESET     # 분석 후 초기화

임계값 조정 (더 세밀한 탐지):
  redis-cli CONFIG SET slowlog-log-slower-than 1000  # 1ms 이상
  redis-cli CONFIG SET slowlog-max-len 256           # 최대 256개 보관

O(N) 명령어 즉시 교체:
  KEYS *        → SCAN 0 MATCH * COUNT 100   (커서 방식)
  SMEMBERS key  → SSCAN key 0 COUNT 100       (Set 분할)
  HGETALL key   → HSCAN key 0 COUNT 50        (Hash 분할)
  LRANGE 0 -1   → LRANGE 0 99 (페이징)        (크기 제한)

commandstats로 전체 명령어 성능 분석:
  redis-cli INFO commandstats | awk -F'[=,]' '
  /cmdstat_/ {
    cmd=$1; calls=$3; usec=$5; per_call=$7
    if (per_call+0 > 500) printf "%s: avg %.0f μs (calls=%s)\n", cmd, per_call, calls
  }' | sort -t: -k2 -rn | head -20
  # 평균 500μs 초과 명령어 상위 추출
```

---

## 🔬 내부 동작 원리

### 1. SLOWLOG 동작 원리

```
Redis SLOWLOG 기록 조건:
  명령어 실행 시간 > slowlog-log-slower-than (μs)

측정 대상:
  ① 명령어 큐에서 꺼내는 시간
  ② 명령어 파싱 시간
  ③ 실제 명령어 실행 시간 (메모리 접근)
  
  측정 제외:
  → 소켓 읽기 시간 (I/O 대기)
  → 응답 전송 시간
  → 클라이언트 측 처리 시간
  
  의미:
    SLOWLOG에 100ms 나옴 = 이벤트 루프가 100ms 블로킹됨
    = 이 시간 동안 다른 모든 클라이언트 요청 대기

SLOWLOG 내부 구조 (메모리 내 순환 버퍼):
  slowlog-max-len: 보관할 최대 엔트리 수 (기본 128)
  초과 시 가장 오래된 것부터 삭제

SLOWLOG GET 출력 형식:
  redis-cli SLOWLOG GET 3
  
  1) 1) (integer) 42          ← 로그 ID (단조 증가)
     2) (integer) 1711234567  ← Unix timestamp (발생 시각)
     3) (integer) 450234      ← 실행 시간 (마이크로초, 450ms)
     4) 1) "KEYS"             ← 명령어
        2) "user:*"           ← 인자
     5) "127.0.0.1:54321"     ← 클라이언트 주소
     6) ""                    ← 클라이언트 이름 (CLIENT SETNAME으로 설정)
  
  2) 1) (integer) 41
     2) (integer) 1711234500
     3) (integer) 12450       ← 12ms
     4) 1) "LRANGE"
        2) "timeline:user:1"
        3) "0"
        4) "-1"               ← 전체 조회 (위험!)
     5) "10.0.0.5:32100"
     6) "web-server-01"       ← CLIENT SETNAME으로 설정한 이름
```

### 2. O(N) 명령어 — 이벤트 루프 블로킹 원리

```
Redis 이벤트 루프:
  단일 스레드에서 순서대로 명령어 실행
  한 명령어 실행 중 다른 명령어 대기

O(N) 명령어가 위험한 이유:
  N = 전체 키 수 or 자료구조 크기

  KEYS pattern:
    전체 keyspace 해시 테이블을 순회
    N = 전체 키 수 (예: 100만개)
    실행 시간: 100만 × 수십 ns = 수십~수백 ms
    이 시간 동안 모든 GET/SET 요청이 대기!

  명령어별 복잡도:
    KEYS *          O(N) — N = 전체 키 수
    SMEMBERS key    O(N) — N = Set 원소 수
    HGETALL key     O(N) — N = Hash 필드 수
    LRANGE 0 -1     O(N) — N = List 길이
    SORT key        O(N log N)
    SUNIONSTORE     O(N) — N = 모든 Set 원소 합
    FLUSHDB         O(N) — N = DB의 키 수

  실제 영향 측정:
    100만 키 KEYS * → ~200ms 이벤트 루프 블로킹
    이 시간 동안 초당 100,000 요청 중:
      ~20,000 요청이 200ms 이상 대기 → 타임아웃 폭발
```

### 3. SCAN 커서 방식 — 안전한 대안

```
SCAN 동작 원리:
  SCAN cursor MATCH pattern COUNT count
  
  커서(cursor):
    해시 테이블 내 탐색 위치를 나타내는 정수
    0으로 시작, 0이 반환될 때 한 바퀴 완료
  
  COUNT:
    이번 호출에서 탐색할 슬롯 수 (힌트, 정확하지 않음)
    반환되는 키 수가 COUNT와 정확히 같지 않을 수 있음

SCAN의 특성:
  전체 키스페이스를 한 번에 스캔하지 않음
  매 SCAN 호출마다 이벤트 루프를 잠시만 사용
  → 이벤트 루프에 양보하며 분할 탐색

KEYS * vs SCAN 비교:
  KEYS *:
    1회 호출로 전체 키 반환
    이벤트 루프 100ms+ 독점 (키 100만개)
  
  SCAN 루프:
    SCAN 0 COUNT 100 → 약 100개 키 반환 + 다음 커서
    SCAN <cursor> COUNT 100 → 약 100개 키 반환 + 다음 커서
    ... (수만 번 반복)
    SCAN 0 반환 → 완료
    
    각 SCAN 호출: ~0.1ms (이벤트 루프에 영향 최소)
    전체 완료: 수 초 걸릴 수 있지만 서비스 영향 없음

SCAN 보장 사항:
  ✓ 탐색 중 추가된 키: 반환될 수도, 안 될 수도 있음
  ✓ 탐색 중 삭제된 키: 반환될 수도, 안 될 수도 있음
  ✓ 탐색 시작 시 존재했던 키: 반환됨 (완전성 보장)
  ✗ 중복 반환 가능성 있음 → 클라이언트에서 중복 처리 필요

자료구조별 SCAN 대응:
  KEYS pattern  → SCAN 0 MATCH pattern COUNT 100
  SMEMBERS key  → SSCAN key 0 COUNT 100
  HGETALL key   → HSCAN key 0 COUNT 100
  LRANGE 0 -1   → LRANGE 0 99 (페이징)
```

### 4. commandstats — 전체 명령어 성능 분석

```
INFO commandstats 출력:
  redis-cli INFO commandstats
  
  cmdstat_get:calls=45678,usec=123456,usec_per_call=2.70
  cmdstat_set:calls=23456,usec=78901,usec_per_call=3.36
  cmdstat_hgetall:calls=100,usec=890123,usec_per_call=8901.23  ← 위험!
  
  필드 의미:
    calls:          총 호출 횟수 (Redis 시작 이후 누적)
    usec:           총 실행 시간 (마이크로초)
    usec_per_call:  평균 실행 시간 (μs)

O(N) 명령어 탐지:
  usec_per_call이 높은 것 = 느린 명령어
  calls가 높고 usec_per_call도 높으면 = 심각한 문제

  redis-cli INFO commandstats | \
    grep -v "^#" | \
    awk -F'[=,]' '{
      if ($7+0 > 1000)  # 1ms 이상
        printf "%s avg=%.0fμs calls=%s\n", $1, $7, $3
    }' | sort -t= -k2 -rn

  # 출력 예시:
  # cmdstat_hgetall avg=8901μs calls=100    ← HGETALL 느림!
  # cmdstat_smembers avg=1203μs calls=5000  ← SMEMBERS 느림!
```

### 5. LATENCY 명령어 — 지연 히스토리

```
LATENCY 모니터링 활성화:
  redis-cli CONFIG SET latency-monitor-threshold 10  # 10ms 이상

LATENCY HISTORY:
  redis-cli LATENCY HISTORY event
  # 특정 이벤트의 지연 히스토리
  # 이벤트: bgsave, aof-stat, command 등

LATENCY LATEST:
  redis-cli LATENCY LATEST
  # event       latest_time  latest_usec  max_usec
  # command     1711234567   456          2345678
  # bgsave      1711234500   350000       500000
  # → command의 max가 2345ms → 특정 시점 O(N) 실행됨

LATENCY RESET:
  redis-cli LATENCY RESET  # 히스토리 초기화

DEBUG SLEEP을 이용한 지연 측정:
  redis-cli DEBUG SLEEP 0.5  # 500ms 강제 지연
  # → SLOWLOG에 기록됨
  # → 실제 장애 재현 및 모니터링 테스트에 활용
```

---

## 💻 실전 실험

### 실험 1: SLOWLOG 설정 및 O(N) 명령어 포착

```bash
# SLOWLOG 세밀하게 설정
redis-cli CONFIG SET slowlog-log-slower-than 1000   # 1ms 이상
redis-cli CONFIG SET slowlog-max-len 256

# 테스트 데이터 생성
for i in $(seq 1 100000); do
  redis-cli SET "user:$i" "value-$i" > /dev/null
done
echo "10만 키 생성 완료"

# O(N) 명령어 실행
time redis-cli KEYS "user:*" | wc -l
# → 수백 ms 소요

# SLOWLOG 확인
redis-cli SLOWLOG GET 5
# usec: 수백만 μs (수백 ms) 확인

# SLOWLOG 통계
redis-cli SLOWLOG LEN
```

### 실험 2: SCAN으로 KEYS 대체

```bash
# SCAN으로 안전한 순회
scan_all_keys() {
  cursor=0
  total=0
  while true; do
    result=$(redis-cli SCAN $cursor MATCH "user:*" COUNT 100)
    cursor=$(echo "$result" | head -1)
    count=$(echo "$result" | tail -n +2 | wc -l)
    total=$((total + count))
    [ "$cursor" -eq 0 ] && break
  done
  echo "총 키 수: $total"
}

time scan_all_keys
# → 여러 번 나눠 실행되지만 서비스 영향 없음

# SLOWLOG에는 개별 SCAN이 기록되지 않음 확인
redis-cli SLOWLOG LEN  # 증가 없음 (각 SCAN은 빠름)
```

### 실험 3: commandstats로 느린 명령어 찾기

```bash
# 다양한 명령어 실행 후 통계 분석
redis-cli HSET user:1 $(for i in $(seq 1 150); do echo "f$i v$i"; done)
for i in $(seq 1 100); do redis-cli HGETALL user:1 > /dev/null; done
for i in $(seq 1 1000); do redis-cli GET user:1 > /dev/null; done

# commandstats 분석
redis-cli INFO commandstats | grep -E "cmdstat_hgetall|cmdstat_get|cmdstat_set"
# hgetall avg이 높을 것

# 명령어별 평균 시간 정렬
redis-cli INFO commandstats | grep "usec_per_call" | \
  awk -F'[=,]' '{if($7+0 > 0) printf "%-20s %8.1f μs/call  calls=%s\n", $1, $7, $3}' | \
  sort -k2 -rn | head -15
```

### 실험 4: LATENCY 모니터링

```bash
# LATENCY 임계값 설정
redis-cli CONFIG SET latency-monitor-threshold 10  # 10ms

# 강제 지연 발생
redis-cli DEBUG SLEEP 0.5  # 500ms 강제 대기

# LATENCY 확인
redis-cli LATENCY LATEST
redis-cli LATENCY HISTORY command

# 실시간 latency 모니터링
redis-cli --latency          # 실시간 지연 (1초 간격)
redis-cli --latency-history  # 지연 히스토리 (15초 간격)
```

---

## 📊 성능/비용 비교

```
O(N) 명령어 vs SCAN 이벤트 루프 영향:

명령어          키/원소 수  실행 시간   이벤트 루프 블로킹
──────────────┼──────────┼──────────┼──────────────────
KEYS *         1,000,000  ~200ms     200ms (서비스 마비 수준)
SMEMBERS key   100,000    ~10ms      10ms
HGETALL key    10,000     ~1ms       1ms
SCAN 0 COUNT 100 전체     ~0.1ms    0.1ms per 호출

SLOWLOG 임계값별 탐지 범위:
  10,000 μs (10ms): 심각한 병목만 탐지 (기본값)
  1,000 μs (1ms):   중간 수준 병목 탐지 (권장)
  100 μs (0.1ms):   세밀한 분석 (오버헤드 주의)

SLOWLOG 메모리 비용:
  slowlog-max-len 128 × 엔트리당 ~200 bytes = ~25 KB (무시 가능)
  max-len 1024로 늘려도 ~200 KB (부담 없음)
```

---

## ⚖️ 트레이드오프

```
slowlog-log-slower-than 임계값:
  낮을수록: 더 많은 명령어 탐지, SLOWLOG 빠르게 가득 참
  높을수록: 심각한 경우만 탐지, 중간 수준 문제 놓침
  → 개발/스테이징: 100μs (세밀 분석)
  → 운영: 1,000μs (1ms)

SCAN의 트레이드오프:
  장점: 이벤트 루프에 영향 없음, 중복 반환 허용
  단점: 단일 KEYS보다 많은 시간 (전체 기준)
       결과가 순서 없음
       중복 반환 가능 → 클라이언트 처리 필요

MONITOR 사용 주의:
  짧은 시간(1~2초): 개발 환경 디버깅에 유용
  운영 환경 장시간: Redis CPU 급증, 절대 금지
  대안: SLOWLOG + commandstats + LATENCY LATEST
```

---

## 📌 핵심 정리

```
SLOWLOG 분석:
  CONFIG SET slowlog-log-slower-than 1000  # 1ms
  SLOWLOG GET 20  → 느린 명령어 확인 (usec, 명령어, 클라이언트)
  SLOWLOG RESET   → 분석 후 초기화

O(N) 명령어 → SCAN 교체:
  KEYS *  → SCAN 루프
  SMEMBERS → SSCAN
  HGETALL  → HSCAN

commandstats:
  INFO commandstats | usec_per_call 높은 것 → 느린 명령어 패턴

LATENCY:
  CONFIG SET latency-monitor-threshold 10
  LATENCY LATEST  → 최근 최대 지연
  LATENCY HISTORY → 시간대별 지연 추이

진단 순서:
  1. SLOWLOG GET 20  → 특정 명령어 병목 확인
  2. INFO commandstats → 명령어별 평균 지연
  3. LATENCY LATEST  → 이벤트별 최대 지연
  4. redis-cli --latency → 현재 실시간 지연
```

---

## 🤔 생각해볼 문제

**Q1.** SLOWLOG에 기록되는 시간과 클라이언트에서 측정한 Redis 응답 시간이 다를 수 있다. 왜 클라이언트 측 응답 시간이 더 길 수 있는가?

<details>
<summary>해설 보기</summary>

SLOWLOG는 Redis 서버에서 명령어를 실제로 처리한 시간만 측정한다. 클라이언트 관점의 응답 시간은 다음을 모두 포함한다:

```
클라이언트 응답 시간 = 
  네트워크 RTT (왕복)
  + 커넥션 풀 대기 시간 (풀 고갈 시)
  + 직렬화/역직렬화 시간 (JSON 등)
  + Redis 명령어 실행 시간 ← SLOWLOG가 측정하는 것
  + 이벤트 루프 대기 시간 (앞 요청이 오래 걸리면)
```

**예시:**
- SLOWLOG: 2ms (명령어 실행)
- 실제 응답: 50ms
- 차이 48ms = 커넥션 풀 대기(40ms) + 네트워크(8ms)

**진단 방법:**
- 커넥션 풀 대기: `INFO clients`의 `blocked_clients`, `connected_clients`
- 이벤트 루프 대기: 앞 요청의 SLOWLOG 확인
- 네트워크 지연: `redis-cli --latency` (서버 측 ping RTT)

SLOWLOG가 깨끗해도 클라이언트가 느리면 → 네트워크/커넥션 풀 문제를 의심해야 한다.

</details>

---

**Q2.** `SCAN 0 MATCH user:* COUNT 100`이 실제로 100개의 키를 반환하지 않는 경우가 있다. 왜 그런가?

<details>
<summary>해설 보기</summary>

SCAN의 `COUNT` 파라미터는 "반환할 키 수"가 아니라 "탐색할 내부 해시 테이블 슬롯 수"에 대한 힌트다.

**실제 반환 수가 COUNT와 다른 이유:**

1. **MATCH 필터링:** `COUNT 100` 슬롯을 탐색했지만, `user:*` 패턴에 맞는 키가 없으면 0개 반환. 패턴이 드문 경우 매우 적게 반환된다.

2. **인코딩 특성:** 해시 테이블이 `listpack` 인코딩인 경우 (키가 적을 때), 단 1번의 SCAN으로 모든 키를 반환하고 커서 0을 반환할 수 있다.

3. **해시 테이블 분포:** COUNT 100 슬롯에 키가 몰려 있으면 100개 이상, 희박하면 100개 이하 반환.

**올바른 SCAN 루프:**
```bash
cursor=0
keys=()
while true; do
    result=$(redis-cli SCAN $cursor MATCH "user:*" COUNT 100)
    cursor=$(echo "$result" | head -1)
    while IFS= read -r key; do
        keys+=("$key")
    done < <(echo "$result" | tail -n +2)
    [ "$cursor" -eq 0 ] && break
done
echo "${#keys[@]} 키 수집"
```

커서가 0으로 돌아올 때까지 반복하는 것이 핵심이다. COUNT는 성능 힌트일 뿐, 반환 수 보장이 아니다.

</details>

---

**Q3.** 운영 중 SLOWLOG를 주기적으로 비워야 한다면 자동화하는 방법은? SLOWLOG RESET만 하면 충분한가?

<details>
<summary>해설 보기</summary>

SLOWLOG RESET만 하면 데이터가 사라져 분석이 불가능해진다. 먼저 수집(export)하고 초기화해야 한다.

**자동화 패턴:**

```bash
#!/bin/bash
# slow_log_collector.sh (cron으로 1분마다 실행)

REDIS_HOST="localhost"
REDIS_PORT="6379"
LOG_FILE="/var/log/redis/slowlog_$(date +%Y%m%d).log"

# 현재 SLOWLOG 수집
redis-cli -h $REDIS_HOST -p $REDIS_PORT SLOWLOG GET 100 | \
  awk '
  /^[0-9]+\)/ { entry_num=$2 }
  /\(integer\)/ {
    if (!timestamp) timestamp=$2
    else if (!usec) usec=$2
  }
  /"/ { cmd=cmd " " $0 }
  /^$/ {
    if (timestamp) {
      printf "%s\t%s\t%s\t%s\n", strftime("%Y-%m-%d %H:%M:%S", timestamp), usec, cmd, entry_num
      timestamp=""; usec=""; cmd=""
    }
  }' >> $LOG_FILE

# 초기화
redis-cli -h $REDIS_HOST -p $REDIS_PORT SLOWLOG RESET
```

**더 나은 방법: redis_exporter + Prometheus**
```yaml
# prometheus.yml
- job_name: redis
  static_configs:
    - targets: ['localhost:9121']
# → redis_slowlog_last_id, redis_commands_duration_seconds 등 메트릭 자동 수집
```

Prometheus + Grafana를 쓰면 SLOWLOG를 수동으로 관리하지 않아도 된다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: Lua 스크립팅 ➡️](./02-lua-scripting.md)**

</div>
