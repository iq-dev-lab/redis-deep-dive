# Redis 모니터링 — INFO 지표와 Prometheus

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `INFO` 섹션별 핵심 지표를 어떻게 읽는가?
- `used_memory`와 `used_memory_rss`의 차이는?
- `rdb_last_bgsave_status`가 `err`일 때 어떻게 대응하는가?
- `instantaneous_ops_per_sec`으로 무엇을 판단할 수 있는가?
- `redis_exporter` + Prometheus + Grafana 구성 방법은?
- 어떤 지표에 알림을 설정해야 하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Redis 장애는 대부분 예고 없이 오지 않는다. `repl_backlog_size` 부족은 복제 지연으로 이어지고, `used_memory`가 `maxmemory`에 근접하면 eviction이 시작된다. 이 신호들을 미리 감지하는 것이 모니터링의 핵심이다. INFO의 수백 개 지표 중 실제로 중요한 것이 무엇인지 알아야 올바른 알림을 설정할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: INFO keyspace만 확인하고 메모리 문제 놓침

  모니터링 코드:
    redis.info("keyspace")  # 키 수만 확인
  
  놓친 것:
    used_memory가 maxmemory의 95%에 도달
    → eviction 시작 → 중요 데이터 삭제
    → 서비스 이상 발생 후에야 확인
  
  필요했던 것:
    used_memory / maxmemory × 100 > 80 → 알림

실수 2: rdb_last_bgsave_status 무시

  상황: BGSAVE가 OOM으로 실패 (자식 프로세스 메모리 부족)
  INFO persistence:
    rdb_last_bgsave_status:err  ← 무시됨!
  
  결과: 수 시간 동안 RDB 저장 실패
  서버 재시작 시 데이터 손실 가능
  알림이 없었기 때문에 모름

실수 3: connected_clients만 모니터링

  모니터링: connected_clients > 1000 → 알림
  누락된 지표:
    blocked_clients (블로킹 대기 중인 클라이언트)
    → blocked_clients 급증 = BLPOP/BRPOP 대기 폭발 or 느린 명령어
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
핵심 지표 3가지 카테고리:

① 메모리 건강:
  used_memory / maxmemory > 0.85  → 경고
  mem_fragmentation_ratio > 2.0   → 경고
  mem_fragmentation_ratio < 1.0   → 긴급 (스왑!)

② 영속성 상태:
  rdb_last_bgsave_status = "err"  → 긴급
  aof_last_write_status = "err"   → 긴급
  rdb_changes_since_last_save > 임계값 → 경고 (마지막 저장 후 너무 많은 변경)

③ 복제 건강:
  connected_slaves = 0 (마스터인데 슬레이브 없음) → 경고
  slave0 lag > 10초              → 경고
  master_link_status = "down"    → 긴급

Prometheus 알림 규칙:
  - alert: RedisMemoryHigh
    expr: redis_memory_used_bytes / redis_memory_max_bytes > 0.85
    for: 5m
  
  - alert: RedisBgsaveFailed
    expr: redis_rdb_last_bgsave_status == 0  # 0=fail
    for: 1m
  
  - alert: RedisReplicationLag
    expr: redis_connected_slaves < 1
    for: 2m
```

---

## 🔬 내부 동작 원리

### 1. INFO 섹션 구조

```
INFO 전체 섹션:
  redis-cli INFO          → 모든 섹션
  redis-cli INFO server   → 서버 기본 정보
  redis-cli INFO clients  → 클라이언트 연결 상태
  redis-cli INFO memory   → 메모리 사용 현황
  redis-cli INFO stats    → 처리량 통계
  redis-cli INFO replication → 복제 상태
  redis-cli INFO cpu      → CPU 사용
  redis-cli INFO keyspace → DB별 키 통계
  redis-cli INFO persistence → RDB/AOF 상태

핵심 지표별 위치:
  used_memory              → INFO memory
  maxmemory               → INFO memory
  mem_fragmentation_ratio  → INFO memory
  connected_clients        → INFO clients
  blocked_clients          → INFO clients
  instantaneous_ops_per_sec → INFO stats
  keyspace_hits            → INFO stats
  keyspace_misses          → INFO stats
  evicted_keys             → INFO stats
  rdb_last_bgsave_status   → INFO persistence
  aof_last_write_status    → INFO persistence
  connected_slaves         → INFO replication
  master_repl_offset       → INFO replication
  slave0: lag=N            → INFO replication
  latest_fork_usec         → INFO stats
```

### 2. 메모리 지표 완전 분석

```
INFO memory 핵심 지표:

used_memory: 1073741824          # Redis가 C malloc으로 할당한 메모리 (바이트)
used_memory_human: 1.00G         # 사람이 읽기 쉬운 형식
used_memory_rss: 1610612736      # OS가 Redis 프로세스에 할당한 물리 메모리
used_memory_rss_human: 1.50G     # RSS (실제 RAM 점유)
used_memory_peak: 1342177280     # 과거 최대 사용 메모리
used_memory_peak_human: 1.25G
used_memory_overhead: 123456     # Redis 내부 자료구조 오버헤드
used_memory_dataset: 950285376   # 실제 데이터 크기 (overhead 제외)
mem_fragmentation_ratio: 1.50    # RSS / used_memory
mem_allocator: jemalloc-5.3.0    # 할당기
maxmemory: 2147483648            # 설정된 한도 (0 = 무제한)
maxmemory_human: 2.00G
maxmemory_policy: allkeys-lru    # eviction 정책

계산식:
  메모리 사용률 = used_memory / maxmemory × 100
  실제 데이터 = used_memory_dataset
  단편화 = (used_memory_rss - used_memory) / used_memory × 100

주의: maxmemory = 0이면 메모리 무제한 사용
  → used_memory / maxmemory 계산 불가
  → used_memory / total_system_memory로 대체
```

### 3. 처리량과 캐시 효율 지표

```
INFO stats 핵심:

instantaneous_ops_per_sec: 45678  # 현재 초당 명령어 수 (1초 이동 평균)
instantaneous_input_kbps: 12345   # 현재 입력 대역폭 (kbps)
instantaneous_output_kbps: 34567  # 현재 출력 대역폭 (kbps)
total_commands_processed: 12345678 # 시작 이후 총 명령어 (누적)

keyspace_hits: 5678901    # 캐시 히트 횟수 (누적)
keyspace_misses: 123456   # 캐시 미스 횟수 (누적)
# 히트율 = hits / (hits + misses)

evicted_keys: 1234        # maxmemory-policy로 제거된 키 수 (누적)
expired_keys: 234567      # TTL 만료로 삭제된 키 수 (누적)
rejected_connections: 0   # maxclients 초과로 거부된 연결 수

latest_fork_usec: 350000  # 최근 fork() 소요 시간 (마이크로초)
                           # 350000 = 350ms (큰 인스턴스에서 주의)

해석:
  ops/sec이 갑자기 떨어짐 → 느린 명령어 or 연결 문제
  evicted_keys 증가 → maxmemory 임박, 데이터 손실 가능
  keyspace_misses 급증 → 캐시 히트율 저하, Stampede 가능성
  latest_fork_usec > 500ms → BGSAVE 지연 주의
```

### 4. 복제 상태 지표

```
INFO replication (Master 기준):

role: master
connected_slaves: 1
slave0: ip=10.0.0.2,port=6379,state=online,offset=12345678,lag=0

master_failover_state: no-failover
master_replid: a1b2c3d4e5f6...    # 현재 복제 계보 ID
master_repl_offset: 12345678       # Master의 현재 복제 오프셋
repl_backlog_active: 1
repl_backlog_size: 1048576         # backlog 크기 설정 (1 MB)
repl_backlog_first_byte_offset: 11297103  # backlog 시작 오프셋
repl_backlog_histlen: 1048575      # backlog에 저장된 데이터 길이

INFO replication (Replica 기준):

role: slave
master_host: 10.0.0.1
master_port: 6379
master_link_status: up            # up/down
master_last_io_seconds_ago: 1     # 마지막 통신 이후 경과 시간
master_sync_in_progress: 0        # 동기화 진행 중 여부
slave_repl_offset: 12345670        # Replica 현재 오프셋
slave_priority: 100
slave_read_only: 1

복제 지연 계산:
  lag_bytes = master_repl_offset - slave_repl_offset
  # = 12345678 - 12345670 = 8 bytes (매우 작음, 정상)
  
  slave0의 lag=0 → Replica가 거의 동기화됨
  slave0의 lag=5 → 5초 지연 (확인 필요)
```

### 5. redis_exporter + Prometheus + Grafana

```
구성 개요:
  Redis → redis_exporter → Prometheus → Grafana 대시보드
                        ↓
                    Alertmanager → PagerDuty/Slack 알림

redis_exporter 실행:
  docker run -d \
    --name redis_exporter \
    -p 9121:9121 \
    oliver006/redis_exporter \
    --redis.addr redis://localhost:6379

Prometheus 설정 (prometheus.yml):
  scrape_configs:
    - job_name: 'redis'
      static_configs:
        - targets: ['localhost:9121']
      scrape_interval: 15s

주요 Prometheus 메트릭 (redis_exporter 제공):
  redis_memory_used_bytes           → used_memory
  redis_memory_max_bytes            → maxmemory
  redis_connected_clients           → connected_clients
  redis_blocked_clients             → blocked_clients
  redis_keyspace_hits_total         → keyspace_hits
  redis_keyspace_misses_total       → keyspace_misses
  redis_evicted_keys_total          → evicted_keys
  redis_rdb_last_bgsave_status      → 0=err, 1=ok
  redis_aof_last_write_status       → 0=err, 1=ok
  redis_connected_slaves            → connected_slaves
  redis_replication_lag             → Replica 지연 초
  redis_commands_processed_total    → total_commands_processed
  redis_instantaneous_ops_per_sec   → instantaneous_ops_per_sec

Grafana 대시보드:
  ID 11835: Redis Dashboard for Prometheus (커뮤니티 제공)
  
  주요 패널:
    Connected Clients (over time)
    Memory Usage (used vs max vs rss)
    Operations per Second
    Cache Hit Rate (hits / (hits + misses))
    Evicted Keys per Second
    Replication Lag (각 Replica)
```

### 6. 알림 임계값 설정 가이드

```
Prometheus Alerting Rules:

groups:
  - name: redis_alerts
    rules:
    
    # 메모리 임박 경고
    - alert: RedisMemoryHigh
      expr: redis_memory_used_bytes / redis_memory_max_bytes > 0.85
      for: 5m
      annotations:
        summary: "Redis 메모리 85% 초과"
    
    # 스왑 발생 (긴급)
    - alert: RedisSwapDetected
      expr: redis_mem_fragmentation_ratio < 1.0
      for: 1m
      annotations:
        summary: "Redis 스왑 발생 (fragmentation ratio < 1.0)"
    
    # RDB 저장 실패
    - alert: RedisBgsaveFailed
      expr: redis_rdb_last_bgsave_status == 0
      for: 1m
      annotations:
        summary: "Redis BGSAVE 실패"
    
    # 복제 연결 끊김
    - alert: RedisNoReplica
      expr: redis_connected_slaves == 0 AND redis_instance_info{role="master"} == 1
      for: 2m
      annotations:
        summary: "Redis Master에 Replica 없음"
    
    # 복제 지연 심각
    - alert: RedisReplicationLag
      expr: redis_replication_lag > 30
      for: 5m
      annotations:
        summary: "Redis 복제 지연 30초 초과"
    
    # 캐시 히트율 저하
    - alert: RedisCacheHitRateLow
      expr: >
        redis_keyspace_hits_total / 
        (redis_keyspace_hits_total + redis_keyspace_misses_total) < 0.8
      for: 10m
      annotations:
        summary: "Redis 캐시 히트율 80% 미만"
    
    # eviction 발생 (중요 데이터 삭제 가능)
    - alert: RedisEvictionOccurring
      expr: rate(redis_evicted_keys_total[1m]) > 0
      for: 5m
      annotations:
        summary: "Redis eviction 발생 중"
```

---

## 💻 실전 실험

### 실험 1: INFO 핵심 지표 추출 스크립트

```bash
# 주요 지표 일괄 확인
redis_status() {
  echo "=== Redis 상태 대시보드 ==="
  
  # 메모리
  echo "--- 메모리 ---"
  redis-cli INFO memory | grep -E "^used_memory_human:|^used_memory_rss_human:|^mem_fragmentation_ratio:|^maxmemory_human:"
  
  # 연결
  echo "--- 연결 ---"
  redis-cli INFO clients | grep -E "^connected_clients:|^blocked_clients:|^rejected_connections:"
  
  # 처리량
  echo "--- 처리량 ---"
  redis-cli INFO stats | grep -E "^instantaneous_ops_per_sec:|^keyspace_hits:|^keyspace_misses:|^evicted_keys:|^latest_fork_usec:"
  
  # 영속성
  echo "--- 영속성 ---"
  redis-cli INFO persistence | grep -E "^rdb_last_bgsave_status:|^aof_last_write_status:|^rdb_changes_since_last_save:"
  
  # 복제
  echo "--- 복제 ---"
  redis-cli INFO replication | grep -E "^role:|^connected_slaves:|slave.*lag=|^master_link_status:"
  
  # 캐시 히트율 계산
  HITS=$(redis-cli INFO stats | grep "^keyspace_hits:" | awk -F: '{print $2}' | tr -d ' \r')
  MISSES=$(redis-cli INFO stats | grep "^keyspace_misses:" | awk -F: '{print $2}' | tr -d ' \r')
  if [ $((HITS + MISSES)) -gt 0 ]; then
    echo "캐시 히트율: $(echo "scale=1; $HITS * 100 / ($HITS + $MISSES)" | bc)%"
  fi
}

redis_status
```

### 실험 2: redis_exporter 설치 및 확인

```bash
# Docker로 redis_exporter 실행
docker run -d \
  --name redis_exporter \
  --network host \
  oliver006/redis_exporter \
  --redis.addr redis://127.0.0.1:6379

# 메트릭 확인
curl -s http://localhost:9121/metrics | grep -E "^redis_memory|^redis_connected|^redis_commands"

# 주요 메트릭 추출
curl -s http://localhost:9121/metrics | grep "redis_memory_used_bytes"
# redis_memory_used_bytes 1073741824
```

### 실험 3: 실시간 모니터링 스크립트

```bash
# 1초 간격 실시간 모니터링
while true; do
  OPS=$(redis-cli INFO stats | grep "^instantaneous_ops_per_sec:" | awk -F: '{print $2}' | tr -d ' \r')
  MEM=$(redis-cli INFO memory | grep "^used_memory_human:" | awk -F: '{print $2}' | tr -d ' \r')
  CLIENTS=$(redis-cli INFO clients | grep "^connected_clients:" | awk -F: '{print $2}' | tr -d ' \r')
  EVICT=$(redis-cli INFO stats | grep "^evicted_keys:" | awk -F: '{print $2}' | tr -d ' \r')
  
  printf "\r$(date +%T) | OPS: %-8s | MEM: %-8s | Clients: %-5s | Evicted: %s    " \
    "$OPS" "$MEM" "$CLIENTS" "$EVICT"
  sleep 1
done
```

### 실험 4: 알림 조건 시뮬레이션

```bash
# eviction 유발 (maxmemory 매우 낮게)
redis-cli CONFIG SET maxmemory 1mb
redis-cli CONFIG SET maxmemory-policy allkeys-lru

for i in $(seq 1 1000); do
  redis-cli SET "evict:test:$i" "$(python3 -c "print('x'*100)")" > /dev/null
done

# evicted_keys 확인
redis-cli INFO stats | grep evicted_keys
# evicted_keys:N (N > 0이면 eviction 발생 중)

# 캐시 히트율 저하 시뮬레이션
redis-cli CONFIG SET maxmemory 0  # 무제한
for i in $(seq 1 100); do
  redis-cli GET "nonexistent:$i" > /dev/null  # miss
done

HITS=$(redis-cli INFO stats | grep "^keyspace_hits:" | awk -F: '{print $2}' | tr -d ' \r')
MISSES=$(redis-cli INFO stats | grep "^keyspace_misses:" | awk -F: '{print $2}' | tr -d ' \r')
echo "히트율: $(echo "scale=1; $HITS * 100 / ($HITS + $MISSES)" | bc)%"
```

---

## 📊 성능/비용 비교

```
INFO 명령어 오버헤드:
  redis-cli INFO all: ~0.5ms (수백 개 지표 수집)
  redis-cli INFO memory: ~0.1ms (특정 섹션만)
  → 15초 간격 수집은 오버헤드 무시 수준

모니터링 방식별 비교:
  방식              비용       실시간성    장기 추이    알림
  수동 INFO 스크립트  낮음       수동        없음        없음
  redis_exporter    낮음       15초 간격   90일+       가능
  MONITOR 명령어     매우 높음   실시간      없음        없음
  LATENCY 명령어     낮음       이벤트 기반  없음        없음

핵심 알림 우선순위:
  P1 (즉각): 스왑 발생, RDB/AOF 실패, Master에 Replica 없음
  P2 (5분): 메모리 85% 초과, eviction 발생, 복제 지연 30초+
  P3 (10분): 캐시 히트율 80% 미만, fork 지연 > 500ms
```

---

## ⚖️ 트레이드오프

```
모니터링 수집 간격:
  짧을수록: 더 빠른 이상 감지, Redis 부하 약간 증가
  길수록:  Redis 부하 최소, 이상 감지 지연
  → 15초(Prometheus 기본): 대부분 서비스에 충분

알림 임계값:
  너무 낮으면: 알림 폭풍 (false positive) → 알림 무시 문화 형성
  너무 높으면: 장애 직전에야 알림 → 대응 시간 부족
  → 히스토리 분석 후 임계값 조정 (처음엔 높게 → 점진적 낮춤)

INFO 섹션별 수집:
  INFO all: 한 번에 모두 (편리하지만 불필요한 섹션도 포함)
  INFO memory + INFO stats: 필요한 것만 (약간 효율적)
  → 일반적으로 INFO all 수집이 더 단순하고 차이 없음
```

---

## 📌 핵심 정리

```
Redis 모니터링 핵심 지표:

메모리:
  used_memory / maxmemory            → 사용률 (85% 경고)
  mem_fragmentation_ratio            → 단편화 (> 2.0 경고, < 1.0 긴급)
  evicted_keys                       → eviction 발생 여부

영속성:
  rdb_last_bgsave_status             → ok / err
  aof_last_write_status              → ok / err

복제:
  connected_slaves                   → Replica 수
  slave0 lag                         → Replica 지연 (초)
  master_link_status                 → up / down

성능:
  instantaneous_ops_per_sec          → 현재 처리량
  keyspace_hits / keyspace_misses    → 캐시 히트율
  latest_fork_usec                   → BGSAVE 지연

Prometheus 구성:
  Redis → redis_exporter:9121 → Prometheus → Grafana
  알림: Alertmanager → Slack/PagerDuty

권장 대시보드: Grafana ID 11835
```

---

## 🤔 생각해볼 문제

**Q1.** `keyspace_hits`와 `keyspace_misses`로 계산한 캐시 히트율이 90%인데, 특정 인기 API의 응답 시간이 갑자기 느려졌다. 이 두 데이터가 모순되지 않는 이유는?

<details>
<summary>해설 보기</summary>

전혀 모순되지 않는다. `keyspace_hits/misses`는 **전체 Redis 요청의 집계**다.

**시나리오:**
- 전체 GET 요청의 90%: 일반 키 조회 (모두 hit)
- 특정 인기 API의 GET 요청: 매우 많지만 이 API의 히트율이 0% (해당 키 만료됨)

예:
```
초당 요청:
  일반 GET: 10,000건 (hit 9,000 + miss 1,000) → 90% hit
  인기 API GET: 1,000건 (모두 miss, Cache Stampede!)

전체 히트율: 9,000 / (9,000 + 2,000) = 81%

→ 히트율은 81%로 여전히 양호해 보이지만
→ 특정 API는 100% miss → DB에 1,000 req/sec 폭발
```

**올바른 모니터링:**
- 전체 히트율 외에 특정 키 패턴별 모니터링 필요
- SLOWLOG로 느린 요청 확인
- 애플리케이션 레벨에서 특정 캐시 미스 카운터 추가
- `SCAN 0 MATCH api:*` 또는 `INFO keyspace`로 특정 패턴 확인

</details>

---

**Q2.** `latest_fork_usec: 350000` (350ms)가 나왔다. 이것이 문제인가? 어떤 영향을 미치는가?

<details>
<summary>해설 보기</summary>

**BGSAVE나 AOF Rewrite 시 350ms 동안 이벤트 루프가 부분적으로 블로킹된다는 의미다.**

`fork()` 시스템 콜은 메인 프로세스(Redis)에서 실행된다:
1. fork() 호출 → 페이지 테이블 복사 (메모리 크기에 비례)
2. fork() 완료 후 메인 프로세스가 정상 동작 재개

350ms 동안:
- 새 클라이언트 연결이 지연될 수 있음
- 이미 연결된 클라이언트의 명령어 처리도 지연
- SLOWLOG에 350ms+ 응답 시간 급증 가능

**허용 기준:**
- < 50ms: 일반적으로 허용 가능
- 50~200ms: 주의 (서비스 지연 티켓 발생 가능)
- > 200ms: 문제 (BGSAVE 간격 조정 또는 다른 대책 필요)

**350ms 완화 방법:**
```bash
# 1. Replica에서 BGSAVE 실행 (Master 부하 없음)
# Master: CONFIG SET save ""
# Replica: CONFIG SET save "900 1"

# 2. THP 비활성화 (fork 지연 감소)
echo never > /sys/kernel/mm/transparent_hugepage/enabled

# 3. maxmemory를 낮춰 메모리 크기 감소 (근본적)
```

</details>

---

**Q3.** `evicted_keys`가 증가하고 있지만 `maxmemory-policy`가 `allkeys-lru`다. 이것은 중요한 데이터가 삭제되고 있다는 의미인가?

<details>
<summary>해설 보기</summary>

**설계 의도에 따라 다르다.** `allkeys-lru`는 모든 키를 LRU로 제거할 수 있으므로:

**캐시 전용 Redis라면:** 정상 동작이다. LRU가 오래된 캐시를 제거하고 새 데이터를 위한 공간을 확보하는 것이 의도된 동작이다.

**중요 데이터가 있다면:** 큰 문제다. 제거된 키가 캐시인지 중요 데이터인지 Redis는 구별하지 않는다.

**진단:**
```bash
# 최근 eviction이 어떤 키들인지 추적 불가 (Redis는 기록 안 함)
# 단, Keyspace Notification으로 사전 감지 가능

CONFIG SET notify-keyspace-events "Ke"  # K=Keyevent, e=evicted
SUBSCRIBE __keyevent@0__:evicted
# → 어떤 키가 eviction됐는지 실시간 확인
```

**올바른 설계 원칙:**
- 캐시 전용: `allkeys-lru` + eviction 허용 (의도적)
- 중요 데이터 포함: `noeviction` + `maxmemory` 알림으로 사전 확장
- 또는 역할 분리: 캐시 Redis(allkeys-lru) + 데이터 Redis(noeviction) 분리

</details>

---

<div align="center">

**[⬅️ 이전: 메모리 최적화](./03-memory-optimization.md)** | **[홈으로 🏠](../README.md)** | **[다음: 운영 장애 패턴 ➡️](./05-operational-failures.md)**

</div>
