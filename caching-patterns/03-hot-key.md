# Hot Key 문제 — 읽기 vs 쓰기 Hot Key 해결

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Hot Key가 단일 Redis 노드의 CPU/네트워크 병목을 일으키는 원리는?
- 읽기 Hot Key와 쓰기 Hot Key는 왜 해결 방법이 다른가?
- 로컬 캐시(JVM 메모리) + Redis 이중 레이어가 효과적인 이유는?
- 키 복제(Key Replication)로 읽기 Hot Key를 분산하는 방법은?
- 카운터 분산(Counter Sharding)으로 쓰기 Hot Key를 해결하는 방법은?
- `redis-cli --hotkeys`로 Hot Key를 어떻게 진단하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Redis Cluster는 데이터를 여러 노드에 분산하지만, 특정 키에 트래픽이 집중되면 그 키를 담당하는 단일 노드가 병목이 된다. 초당 수십만 요청이 한 노드로 몰리면 Redis 자체가 CPU 100%에 도달하고, 해당 노드를 거치는 모든 요청이 느려진다. Cluster라도 Hot Key 문제는 해결되지 않는다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Cluster 구성으로 Hot Key가 해결된다고 착각

  팀 결정: "트래픽이 너무 많으니까 Redis Cluster 구성"
  결과: flash_sale:event:1 키에 초당 100,000 요청 집중
        CRC16("flash_sale:event:1") % 16384 = 5000번 슬롯 → Node B 담당
        Node B 혼자 모든 100,000 req/sec 처리
        → Node B CPU 100% → 응답 시간 급증
  
  Cluster는 키를 분산하지, 단일 키 트래픽을 분산하지 않음

실수 2: Hot Key를 모르고 Redis 전체 업그레이드

  현상: Redis 응답 시간 간헐적 급등
  대응: Redis 인스턴스를 더 큰 사양으로 업그레이드
  결과: 약간 나아지지만 Hot Key 문제는 지속
  
  진단 없이 스펙 업그레이드 → 비용 낭비
  redis-cli --hotkeys로 먼저 진단했다면 알 수 있었음

실수 3: 카운터를 단일 키로 관리

  코드: INCR "global:like_count"  # 모든 좋아요를 하나의 키로
  초당 50,000번 INCR → 단일 키 → 단일 노드 병목
  
  해결: 카운터 샤딩 (Counter Sharding)
    INCR "global:like_count:shard:3"  # 샤드 번호로 분산
    읽기 시 모든 샤드 합산
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
Hot Key 진단 먼저:
  # 방법 1: --hotkeys 옵션 (LFU 정책 필요)
  redis-cli CONFIG SET maxmemory-policy allkeys-lfu
  redis-cli --hotkeys
  
  # 방법 2: MONITOR (운영 환경 주의)
  redis-cli MONITOR | head -1000 | awk '{print $NF}' | sort | uniq -c | sort -rn | head -20
  
  # 방법 3: keyspace 통계
  redis-cli INFO keyspace

읽기 Hot Key 해결 (가장 효과적):
  이중 레이어 캐시:
    1단계: 로컬 캐시 (JVM Caffeine, 각 서버 인스턴스 내)
    2단계: Redis (공유 캐시)
    
    인기 상품 1개가 100대 서버 × 초당 1,000 요청 = 초당 100,000 Redis 조회
    → 로컬 캐시 적용 후: 각 서버 초당 1번 Redis 조회 = 초당 100회 Redis 조회
    → Redis 부하 1,000배 감소

쓰기 Hot Key 해결 (카운터 샤딩):
  INCR "likes:{article_id}:shard:{random 1-10}"  # 10개 샤드로 분산
  읽기: 10개 샤드 합산 = 전체 좋아요 수
```

---

## 🔬 내부 동작 원리

### 1. Hot Key가 단일 노드 병목을 일으키는 원리

```
단순 Master-Replica:
  Hot Key "popular:article:1" → Master 한 대에 집중
  초당 50,000 요청 × 평균 1 KB 응답 = 50 MB/초 네트워크 + CPU

Redis Cluster의 한계:
  Cluster는 키를 노드에 분산:
    Node A: user:* 키들
    Node B: article:* 키들
    Node C: product:* 키들
  
  하지만 "article:top_trending"이 인기라면:
    모든 요청이 Node B로 → Node B만 과부하
    Node A, C는 한가한 상태
  
  Cluster가 Hot Key를 분산 못하는 이유:
    동일 키는 동일 슬롯 → 동일 노드 (변경 불가)
    키 이름이 다르면 다른 슬롯/노드로 분산됨

Hot Key 병목 신호:
  redis-cli INFO clients | grep connected_clients (증가)
  redis-cli INFO stats | grep instantaneous_ops_per_sec (특정 노드만 높음)
  redis-cli INFO cpu | grep used_cpu_sys (CPU 높음)
  특정 노드 응답 시간만 느림
```

### 2. 읽기 Hot Key 해결: 이중 레이어 캐시

```
구조:
  ┌─────────────────────────────────────────────────────┐
  │              애플리케이션 서버 (100대)                   │
  │                                                     │
  │  ┌────────────────────────────────────────────────┐ │
  │  │  로컬 캐시 (Caffeine, 각 서버의 JVM 메모리)          │ │
  │  │  TTL: 10초, 최대 크기: 1,000개                    │ │
  │  └────────────────────────────────────────────────┘ │
  │              ↓ 로컬 캐시 miss                         │
  │  ┌────────────────────────────────────────────────┐ │
  │  │  Redis 클라이언트                                 │ │
  │  └────────────────────────────────────────────────┘ │
  └─────────────────────────────────────────────────────┘
              ↓ Redis miss
  ┌────────────────────────────────────────────────────┐
  │  Redis (공유 캐시, TTL: 300초)                        │
  └────────────────────────────────────────────────────┘
              ↓ Redis miss
  ┌────────────────────────────────────────────────────┐
  │  DB                                                │
  └────────────────────────────────────────────────────┘

트래픽 감소 효과:
  인기 키 1개, 서버 100대, 서버당 초당 1,000 요청:
  
  로컬 캐시 없음:
    초당 100,000 Redis 요청 (모두 Redis로)
  
  로컬 캐시 TTL 10초 적용:
    각 서버가 10초마다 1회 Redis 조회
    초당 Redis 요청 = 100대 / 10초 = 10 req/sec
    → 10,000배 Redis 부하 감소!

로컬 캐시 TTL 선택:
  짧을수록: Redis 부하 감소 효과 적음, 데이터 최신성 높음
  길수록:   Redis 부하 크게 감소, 데이터 일관성 낮아짐
  
  Hot Key 해소가 목표라면: 10~60초 (서비스 허용 스테일 기준)
  실시간성이 중요하면: 1~5초

Spring Boot + Caffeine 로컬 캐시 설정:
  @Bean
  CacheManager cacheManager() {
      CaffeineCacheManager mgr = new CaffeineCacheManager();
      mgr.setCaffeine(Caffeine.newBuilder()
          .expireAfterWrite(10, TimeUnit.SECONDS)
          .maximumSize(1000));
      return mgr;
  }
  
  @Cacheable("hot-articles")  // Caffeine 로컬 캐시 사용
  public Article findHotArticle(Long id) { ... }
```

### 3. 읽기 Hot Key 해결: 키 복제 (Key Replication)

```
원리: 동일 데이터를 여러 키에 복제 → 여러 노드에 분산

  "article:1" 데이터를 복제:
    article:1:replica:0  → Node A (슬롯 1234)
    article:1:replica:1  → Node B (슬롯 5678)
    article:1:replica:2  → Node C (슬롯 9012)

읽기 시:
  random_shard = random.randint(0, 2)
  key = f"article:1:replica:{random_shard}"
  data = redis.get(key)

쓰기 시:
  for i in range(3):  # 모든 복제본 업데이트
      redis.set(f"article:1:replica:{i}", data, ex=300)

트래픽 분산:
  초당 30,000 요청 → 3개 노드에 각 10,000씩 분산 (3배 향상)

단점:
  쓰기 증폭: 1번 쓰기 → N번 복제 쓰기
  일시적 불일치: 일부 복제본이 갱신 전 구버전 서빙 가능
  관리 복잡도 증가

적합한 경우:
  읽기 비율이 압도적으로 높음 (99% 읽기, 1% 쓰기)
  약간의 스테일 데이터 허용 가능
  로컬 캐시보다 더 강한 분산 필요
```

### 4. 쓰기 Hot Key 해결: 카운터 샤딩

```
문제: INCR "global:likes" → 단일 키 단일 노드 집중

해결: N개 샤드로 분산

  # 쓰기
  INCR "likes:shard:{0~9}"  # 0~9 중 랜덤
  
  # 구체적으로:
  def increment_likes(article_id):
      shard = random.randint(0, 9)  # 10개 샤드
      key = f"likes:{article_id}:shard:{shard}"
      redis.incr(key)

  # 읽기 (합산)
  def get_total_likes(article_id):
      total = 0
      for shard in range(10):
          key = f"likes:{article_id}:shard:{shard}"
          count = redis.get(key)
          total += int(count or 0)
      return total

트래픽 분산:
  기존: INCR "likes:article:1" → Node A에 초당 50,000 INCR
  샤딩 후:
    각 샤드로 분산 → 노드당 5,000 req/sec (10배 분산)
    각 샤드 키가 다른 노드로 해시될 수 있음

단점:
  읽기 시 N번 조회 → RTT × N (Pipeline으로 완화 가능)
  Pipeline 사용:
    pipe = redis.pipeline()
    for shard in range(10):
        pipe.get(f"likes:{article_id}:shard:{shard}")
    results = pipe.execute()  # 단일 RTT로 10개 조회

정확도:
  INCR는 원자적 → 각 샤드에서 정확한 카운트
  합산 시 정확한 전체 카운트
  → 분산했지만 정확성 유지
```

### 5. Hot Key 진단 방법

```
방법 1: redis-cli --hotkeys (LFU 정책 필요)

  # LFU 정책 활성화 (또는 allkeys-lfu)
  redis-cli CONFIG SET maxmemory-policy allkeys-lfu
  
  # Hot Key 분석
  redis-cli --hotkeys
  # ====== Scanning the entire keyspace to find hot keys ======
  # [00.00%] Hot key 'article:1' found so far with counter 42
  # [00.00%] Hot key 'product:trending' found so far with counter 38
  # 
  # -------- summary -------
  # Sampled 10000 keys in the keyspace!
  # hot key found with counter: 42   keyname: article:1
  # hot key found with counter: 38   keyname: product:trending

  단점: LFU 정책 필요 (eviction 정책 변경 영향)

방법 2: MONITOR + 통계 (주의: 운영 영향)

  # MONITOR는 모든 명령어를 출력 → 성능 영향 큼
  # 짧은 시간만 실행
  timeout 10 redis-cli MONITOR | \
    grep -oP '"[^"]*"' | head -50000 | \
    sort | uniq -c | sort -rn | head -20

방법 3: Slow Log + INFO stats

  # 특정 키 패턴의 명령어 통계
  redis-cli INFO commandstats | grep "hget\|get\|set"
  
  # 전체 키 접근 통계 (패턴별)
  redis-cli DEBUG SLEEP 0  # 연결 확인
  redis-cli INFO keyspace   # DB별 키 수, 만료 수

방법 4: 외부 모니터링 (운영 환경 권장)

  Redis Exporter + Prometheus + Grafana:
    redis_key_hit_rate → 히트율 대시보드
    redis_commands_total{cmd="get"} → GET 명령어 빈도
    → 특정 시점 spike 감지 → Hot Key 의심

실시간 감지 패턴:
  알림 조건: instantaneous_ops_per_sec > 임계값
  → Redis 응답 시간 급등 → Hot Key 의심 → --hotkeys 실행
```

---

## 💻 실전 실험

### 실험 1: Hot Key 진단

```bash
# LFU 정책으로 변경
redis-cli CONFIG SET maxmemory-policy allkeys-lfu

# 불균등 키 접근 패턴 생성
for i in $(seq 1 1000); do
  redis-cli GET "hot:article:1" > /dev/null  # 많이 접근
done
for i in $(seq 1 10); do
  redis-cli GET "cold:article:2" > /dev/null  # 적게 접근
done

# Hot Key 분석
redis-cli --hotkeys 2>/dev/null | head -20
# hot:article:1 이 상위에 나타남

# OBJECT FREQ로 개별 키 빈도 확인
redis-cli OBJECT FREQ "hot:article:1"   # 높은 수
redis-cli OBJECT FREQ "cold:article:2"  # 낮은 수
```

### 실험 2: 카운터 샤딩 구현

```bash
ARTICLE_ID="article:1"
SHARD_COUNT=10

# 쓰기: 랜덤 샤드에 INCR
for i in $(seq 1 100); do
  SHARD=$((RANDOM % SHARD_COUNT))
  redis-cli INCR "likes:${ARTICLE_ID}:shard:${SHARD}" > /dev/null
done

# 읽기: 전체 샤드 합산
echo "=== 샤드별 카운트 ==="
total=0
for shard in $(seq 0 $((SHARD_COUNT - 1))); do
  count=$(redis-cli GET "likes:${ARTICLE_ID}:shard:${shard}")
  count=${count:-0}
  echo "  shard $shard: $count"
  total=$((total + count))
done
echo "총 좋아요: $total"

# Pipeline으로 효율적 합산 (단일 RTT)
redis-cli PIPELINED << 'EOF'
GET likes:article:1:shard:0
GET likes:article:1:shard:1
GET likes:article:1:shard:2
GET likes:article:1:shard:3
GET likes:article:1:shard:4
GET likes:article:1:shard:5
GET likes:article:1:shard:6
GET likes:article:1:shard:7
GET likes:article:1:shard:8
GET likes:article:1:shard:9
EOF
```

### 실험 3: 키 복제로 읽기 분산

```bash
# 원본 데이터 복제 (3개 복사본)
ORIGINAL_DATA='{"id":1,"title":"Trending Article","views":1000000}'
for replica in 0 1 2; do
  redis-cli SET "article:1:replica:$replica" "$ORIGINAL_DATA" EX 300 > /dev/null
done

echo "=== 복제 키 생성 완료 ==="
redis-cli KEYS "article:1:replica:*"

# 읽기: 랜덤 복제본에서 조회
for i in $(seq 1 10); do
  REPLICA=$((RANDOM % 3))
  KEY="article:1:replica:$REPLICA"
  VALUE=$(redis-cli GET $KEY)
  echo "요청 $i → replica:$REPLICA → ${VALUE:0:30}..."
done

# 업데이트 시 모든 복제본 갱신
NEW_DATA='{"id":1,"title":"Trending Article Updated","views":1000001}'
for replica in 0 1 2; do
  redis-cli SET "article:1:replica:$replica" "$NEW_DATA" EX 300 > /dev/null
done
echo "=== 모든 복제본 갱신 완료 ==="
```

### 실험 4: 실시간 Hot Key 모니터링

```bash
# 실시간 명령어 통계 (10초 간격)
PREV=$(redis-cli INFO stats | grep total_commands_processed | awk -F: '{print $2}' | tr -d ' \r')
sleep 10
CURR=$(redis-cli INFO stats | grep total_commands_processed | awk -F: '{print $2}' | tr -d ' \r')
echo "초당 명령어 처리: $(( (CURR - PREV) / 10 )) ops/sec"

# 키스페이스 히트/미스 비율
redis-cli INFO stats | grep -E "keyspace_hits|keyspace_misses"

# 클러스터 노드별 처리량 비교 (Hot Node 탐지)
for port in 7001 7002 7003; do
  OPS=$(redis-cli -p $port INFO stats | grep instantaneous_ops_per_sec | awk -F: '{print $2}' | tr -d ' \r')
  echo "Node:$port → $OPS ops/sec"
done
# 특정 노드만 높으면 Hot Key 의심
```

---

## 📊 성능/비용 비교

```
Hot Key 해결책별 비교:

                  이중 레이어 캐시  | 키 복제            | 카운터 샤딩
──────────────────────────────────────────────────────────────────────
대상              읽기 Hot Key    | 읽기 Hot Key       | 쓰기 Hot Key
Redis 부하 감소   극적 (10~1000배)  | 복제본 수배         | 샤드 수배
데이터 일관성     로컬 TTL 만큼 지연   | 쓰기 전파 전 지연    | 합산 시 정확
구현 복잡도       낮음              | 중간              | 중간
추가 메모리       로컬 서버 메모리     | 복제본 × 데이터 크기 | 샤드 × 카운터

이중 레이어 캐시 효과 계산:
  서버 100대, 각 서버 초당 1,000 요청, 로컬 TTL 10초:
    Redis 요청 = 100대 / 10초 = 10 req/sec
    감소율 = 100,000 / 10 = 10,000배!

카운터 샤딩 효과:
  단일 키: 초당 50,000 INCR → 단일 노드 부하
  10개 샤드: 노드당 5,000 INCR (10배 분산)
  읽기: 10번 GET (Pipeline으로 단일 RTT)
```

---

## ⚖️ 트레이드오프

```
이중 레이어 캐시:
  Redis 부하 극적 감소 ↔ 로컬 TTL만큼 데이터 최신성 저하
  서버별 캐시 독립 → 서버마다 다른 데이터 가능 (약한 일관성)
  배포 시 로컬 캐시 워밍업 필요

키 복제:
  읽기 분산 ↔ 쓰기 증폭 (N배)
  업데이트 실패 시 복제본 간 불일치
  원자적 업데이트 어려움

카운터 샤딩:
  쓰기 분산 ↔ 읽기 시 N번 조회 필요
  Pipeline 사용으로 RTT 최소화 가능
  실시간 정확한 카운트 가능 (각 샤드가 원자적)

로컬 캐시 vs Redis 비교:
  로컬 캐시:  낮은 지연 (~μs), 서버당 독립, 일관성 낮음
  Redis:      중간 지연 (~ms), 공유 상태, 일관성 높음
  → Hot Key 해소: 로컬 캐시 우선
  → 공유 상태 필요: Redis 유지
```

---

## 📌 핵심 정리

```
Hot Key 핵심:

정의: 단일 키에 과도한 트래픽이 집중 → 해당 노드 CPU/네트워크 포화
Cluster도 해결 못함: 동일 키 = 동일 슬롯 = 동일 노드

진단:
  redis-cli --hotkeys  (LFU 정책 필요)
  MONITOR + grep (짧은 시간만)
  instantaneous_ops_per_sec 노드별 비교

읽기 Hot Key 해결:
  이중 레이어 캐시 (로컬 + Redis):
    가장 효과적, Redis 부하 수백~수천 배 감소
    로컬 TTL = 스테일 허용 범위
  
  키 복제:
    복제본 수만큼 부하 분산
    쓰기 증폭 단점

쓰기 Hot Key 해결:
  카운터 샤딩:
    INCR 를 N개 샤드에 분산
    읽기 시 Pipeline으로 합산

모니터링:
  특정 노드만 ops/sec 높으면 Hot Key 의심
  OBJECT FREQ key로 개별 키 빈도 확인
```

---

## 🤔 생각해볼 문제

**Q1.** 이중 레이어 캐시(로컬 + Redis)에서 상품 가격이 변경됐다. 로컬 캐시 TTL이 60초라면, 가격 변경 후 최악의 경우 얼마나 오래 구버전 가격이 사용자에게 보이는가?

<details>
<summary>해설 보기</summary>

**최악 시나리오:**
1. 서버 A의 로컬 캐시에 가격 1,000원이 방금 저장됨 (TTL 60초 시작)
2. DB에서 가격 1,500원으로 변경 → Redis 무효화 (DELETE)
3. 서버 A의 로컬 캐시는 아직 만료되지 않음 → 60초 후 만료

**최악 경우:** 서버 A는 **최대 60초** 동안 구버전 1,000원 가격을 서빙한다.

**완화 방법:**

1. **가격 변경 시 모든 서버의 로컬 캐시 직접 무효화 (복잡):**
   Redis Pub/Sub을 이용해 모든 서버에 "캐시 무효화" 메시지 브로드캐스트
   ```python
   # 가격 변경 시
   redis.delete("product:1")  # Redis 무효화
   redis.publish("cache:invalidate", "product:1")  # 모든 서버에 알림
   
   # 각 서버에서 구독
   def on_invalidate(message):
       local_cache.invalidate(message)
   ```

2. **TTL 줄이기:** 60초 → 5초 (스테일 허용 범위 축소)

3. **가격처럼 정확성이 중요한 데이터는 로컬 캐시 제외** (Redis만 사용)

결론: "최대 허용 스테일 시간"을 기준으로 로컬 캐시 TTL을 설정해야 한다.

</details>

---

**Q2.** 카운터 샤딩에서 10개 샤드를 사용한다. 특정 샤드(shard:3)에 Redis 장애가 발생했다. 이 상황에서 INCR과 합산 읽기는 어떻게 되는가?

<details>
<summary>해설 보기</summary>

**INCR 시 shard:3에 쓰기 시도:**
- `INCR likes:article:1:shard:3` → 연결 실패 → 예외 발생
- 이 요청의 좋아요 수가 **유실됨**
- 다른 샤드(0~2, 4~9)는 정상 동작

**합산 읽기 시:**
- shard:3 GET 실패 → 해당 샤드를 0으로 처리
- 실제보다 적은 좋아요 수 반환 (shard:3 카운트 누락)

**내결함성 구현:**
```python
def get_total_likes_safe(article_id):
    total = 0
    pipe = redis.pipeline()
    for shard in range(10):
        pipe.get(f"likes:{article_id}:shard:{shard}")
    try:
        results = pipe.execute()
        for r in results:
            total += int(r or 0)
    except Exception as e:
        # 부분 실패 시 기존 집계값 반환 or 재시도
        pass
    return total
```

**카운터 샤딩의 내결함성 특성:**
- 장애 샤드를 제외하고 나머지 샤드로 서빙 가능 (부분 정확도)
- 전체 합산이 정확하지 않아도 서비스 중단은 아님
- 조회수, 좋아요 수처럼 "근사치 허용" 데이터에 적합

</details>

---

**Q3.** `redis-cli --hotkeys`가 LFU 정책이 필요한 이유는? allkeys-lru에서 Hot Key를 탐지하려면 어떻게 해야 하는가?

<details>
<summary>해설 보기</summary>

**LFU 정책이 필요한 이유:**
`--hotkeys`는 각 키의 `OBJECT FREQ`(접근 빈도 카운터)를 사용한다. 이 카운터는 LFU(Least Frequently Used) 정책에서만 유지된다. LRU 정책에서는 빈도 카운터가 없어서 접근 빈도를 알 수 없다.

```bash
redis-cli OBJECT FREQ some_key
# LFU 정책: (integer) 42
# LRU 정책: (error) ERR object freq is not allowed when maxmemory-policy is not set to an LFU policy.
```

**allkeys-lru에서 Hot Key 탐지 방법:**

1. **MONITOR + 파이프라인 집계 (짧은 시간):**
```bash
# 30초 동안 모든 명령어 캡처
timeout 30 redis-cli MONITOR 2>/dev/null | \
  grep -oP '"[^"]*"' | sort | uniq -c | sort -rn | head -20
```

2. **Redis Exporter + Prometheus:**
   `redis_keyspace_hits_total` 메트릭으로 패턴 파악 (키 이름까지는 어려움)

3. **애플리케이션 레벨 추적:**
```python
# 캐시 조회 시 카운터 기록
def cache_get(key):
    counter_key = f"access_count:{key}"
    pipe = redis.pipeline()
    pipe.incr(counter_key)
    pipe.expire(counter_key, 3600)
    pipe.get(key)
    results = pipe.execute()
    return results[2]

# 주기적으로 상위 접근 키 확인
top_keys = redis.zrevrange("access_count:*", 0, 20)
```

LRU에서 LFU로 임시 전환하는 것도 가능하지만, 정책 변경은 eviction 동작을 바꾸므로 운영 환경에서 신중해야 한다.

</details>

---

<div align="center">

**[⬅️ 이전: Cache Stampede](./02-cache-stampede.md)** | **[홈으로 🏠](../README.md)** | **[다음: 분산 락 ➡️](./04-distributed-lock.md)**

</div>
