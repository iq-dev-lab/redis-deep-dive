# Redis Sentinel — 자동 장애감지와 Failover

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Sentinel이 Master 장애를 감지하는 과정에서 Quorum이 필요한 이유는?
- `sdown`(주관적 다운)과 `odown`(객관적 다운)은 어떻게 다른가?
- Failover에서 새 Master는 어떤 기준으로 선출되는가?
- 클라이언트가 Sentinel을 통해 새 Master 주소를 찾는 방법은?
- Split-Brain 시나리오에서 데이터가 어떻게 유실되는가?
- Sentinel 자체의 가용성을 보장하려면 몇 개가 필요한가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Sentinel은 Redis의 자동 Failover를 담당한다. Quorum, sdown/odown 개념을 모르면 Sentinel이 왜 특정 상황에서 Failover를 하지 않는지, 왜 Split-Brain이 발생하는지 이해할 수 없다. 클라이언트가 새 Master 주소를 찾는 방법을 모르면 Failover 후 애플리케이션이 계속 장애 상태가 된다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Sentinel 1개만 운영

  구성: Sentinel 1개 + Master 1개 + Replica 1개
  장애: Sentinel 서버 장애
  결과: Sentinel 없음 → 자동 Failover 불가
        Master 장애가 발생해도 아무도 승격시키지 않음
        → 수동 복구까지 서비스 다운

  올바른 최소 구성: Sentinel 3개 (홀수, 과반수 Quorum)

실수 2: 클라이언트가 Master 주소를 하드코딩

  코드:
    redis = Redis(host="10.0.0.1", port=6379)  # Master 직접 연결

  Failover 시:
    Master(10.0.0.1) 장애 → Sentinel이 Replica(10.0.0.2)를 승격
    하지만 클라이언트는 여전히 10.0.0.1에 연결 시도
    → 연결 실패 → 서비스 다운

  올바른 방법:
    Sentinel 주소로 연결 → Sentinel에서 현재 Master 주소를 받아옴
    Spring: spring.redis.sentinel.master + sentinel.nodes 설정

실수 3: Split-Brain 상황을 고려하지 않은 설계

  네트워크 파티션 발생:
    Sentinel A, B가 Master와 통신 불가 → Failover 시작
    Sentinel C가 Master와 통신 가능 → Failover 반대 (Quorum 미달)
    클라이언트 일부는 구 Master에 여전히 쓰기
    클라이언트 일부는 새 Master에 쓰기
    → 두 Master에 다른 데이터 → 파티션 해소 후 데이터 충돌
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
Sentinel 최소 구성:
  3개 이상 홀수 배치 (Quorum = 과반수 = 2)
  각 Sentinel을 서로 다른 서버/AZ에 배치
  
  sentinel.conf:
    sentinel monitor mymaster <master-ip> 6379 2  # quorum=2
    sentinel down-after-milliseconds mymaster 5000
    sentinel failover-timeout mymaster 60000

클라이언트 Sentinel 연결 (Spring Boot 예시):
  spring:
    redis:
      sentinel:
        master: mymaster
        nodes: sentinel1:26379, sentinel2:26379, sentinel3:26379

Split-Brain 완화:
  min-replicas-to-write 1  # Replica 없으면 쓰기 거부
  min-replicas-max-lag 10  # Replica 지연 10초 이상이면 쓰기 거부
  → 파티션된 구 Master는 Replica가 없어 쓰기 거부
  → 데이터 충돌 최소화 (가용성 일부 희생)

Sentinel 상태 모니터링:
  redis-cli -p 26379 SENTINEL masters
  redis-cli -p 26379 SENTINEL slaves mymaster
  redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster
```

---

## 🔬 내부 동작 원리

### 1. Sentinel 아키텍처 — 3개가 필요한 이유

```
Sentinel 구성:

  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
  │ Sentinel A  │   │ Sentinel B  │   │ Sentinel C  │
  │ 서버 1       │   │ 서버 2       │   │ 서버 3       │
  └──────┬──────┘   └──────┬──────┘   └──────┬──────┘
         │                 │                 │
         └─────────────────┼─────────────────┘
                           │ (서로 감시 + 통신)
                    ┌──────┴──────┐
                    │   Master    │
                    │  서버 4      │
                    └──────┬──────┘
                           │
                    ┌──────┴──────┐
                    │   Replica   │
                    │  서버 5      │
                    └─────────────┘

각 Sentinel의 역할:
  1. Master를 주기적으로 ping (down-after-milliseconds 기준)
  2. 다른 Sentinel들과 Master 상태를 교환
  3. Quorum 달성 시 Failover 실행

2개 Sentinel이 안 되는 이유:
  Sentinel 1개 장애 → 남은 1개만으로 Failover 결정
  → 네트워크 분리 상황에서 "내가 옳다"고 결정 → Split-Brain 위험
  3개: 2개 동의 = 과반수 → 안전한 다수결
  5개: 3개 동의 = 과반수 → 더 견고 (대규모 환경)
```

### 2. 장애 감지 — sdown에서 odown으로

```
sdown (Subjective Down, 주관적 다운):
  단일 Sentinel이 Master에 ping 후 응답 없을 때
  Sentinel A가 혼자 판단 → "Master가 죽은 것 같다"
  down-after-milliseconds(기본 30초) 동안 응답 없으면 sdown 선언

odown (Objective Down, 객관적 다운):
  quorum 수 이상의 Sentinel이 sdown에 동의할 때
  Sentinel A: sdown 선언
  Sentinel B에게 "Master가 죽었나요?" 확인 → B도 sdown 동의
  A + B = 2 ≥ quorum(2) → **odown 선언** → Failover 시작!

타임라인:
  t=0:   Master 장애 발생 (응답 없음)
  t=5s:  Sentinel A가 ping 응답 없음 감지
  t=30s: down-after-milliseconds 도달 → Sentinel A sdown 선언
  t=31s: Sentinel A가 다른 Sentinel에게 확인 요청
  t=32s: Sentinel B도 sdown 동의 → quorum 도달 → odown
  t=33s: Failover 리더 선출 (Raft-like 알고리즘)
  t=35s: Failover 실행 시작

  총 감지 시간: ~30~35초 (down-after-milliseconds에 크게 의존)

down-after-milliseconds 트레이드오프:
  값 작음 (5,000 ms = 5초):  빠른 Failover, 네트워크 순간 지연에도 오탐 위험
  값 큼 (60,000 ms = 60초): 안전하지만 장애 시 1분 다운
  권장: 15,000~30,000 ms (일반 서비스)
```

### 3. Failover 절차 — 새 Master 선출

```
Failover 단계별 흐름:

Step 1: Failover 리더 선출
  odown을 선언한 Sentinel들 중 리더 선출 (Raft 투표)
  리더가 Failover 전체를 조율

Step 2: 새 Master 선택 (최적 Replica 선택 알고리즘)
  후보에서 제외:
    - 지난 5 × down-after-milliseconds 동안 Master와 연결 단절된 Replica
    - slave-priority = 0인 Replica (수동으로 후보 제외 가능)
  
  남은 후보에서 우선순위 결정:
    1. slave-priority 낮은 값 우선 (작을수록 우선, 기본 100)
    2. 동점 시: replication offset 큰 것 우선 (데이터 가장 최신)
    3. 동점 시: runid 사전순으로 가장 앞선 것

Step 3: 새 Master 승격
  Sentinel → Replica A: REPLICAOF NO ONE
  Replica A가 Master가 됨

Step 4: 다른 Replica 재연결
  Sentinel → Replica B: REPLICAOF <new-master-ip> <port>
  Replica B가 새 Master를 따르기 시작

Step 5: 구 Master 처리 (복구 후)
  구 Master가 재시작 → Sentinel이 Replica로 재구성
  Sentinel → 구 Master: REPLICAOF <new-master-ip> <port>
  → 구 Master가 Replica가 됨

Step 6: 클라이언트 공지
  Sentinel이 Pub/Sub 채널로 Failover 완료 알림
  채널: +switch-master
  클라이언트: Sentinel의 Pub/Sub 구독 → 새 Master 주소 수신
```

### 4. 클라이언트의 새 Master 주소 발견

```
방법 1: Sentinel API 직접 쿼리
  redis-cli -h sentinel1 -p 26379 SENTINEL get-master-addr-by-name mymaster
  # 반환: ["10.0.0.2", "6379"]  ← 현재 Master 주소

방법 2: Sentinel Pub/Sub 구독 (동적 업데이트)
  클라이언트: SUBSCRIBE +switch-master (Sentinel에 구독)
  Failover 시 Sentinel이 메시지 발행:
    "+switch-master mymaster 10.0.0.1 6379 10.0.0.2 6379"
  → 클라이언트가 새 Master(10.0.0.2)로 연결 전환

Spring Boot Jedis/Lettuce Sentinel 설정:
  Jedis:
    JedisSentinelPool("mymaster",
      new HashSet<>(Arrays.asList("s1:26379", "s2:26379", "s3:26379")))
    → Sentinel에서 Master 주소 자동 조회 + Failover 시 자동 전환

  Lettuce (Spring Data Redis):
    RedisClient.create(RedisURI.Builder
      .sentinel("s1", 26379, "mymaster")
      .withSentinel("s2", 26379)
      .withSentinel("s3", 26379)
      .build())

  application.yml:
    spring.redis.sentinel.master: mymaster
    spring.redis.sentinel.nodes:
      - sentinel1:26379
      - sentinel2:26379
      - sentinel3:26379
```

### 5. Split-Brain 시나리오와 대응

```
Split-Brain 발생 조건:

  네트워크 파티션:
    [Sentinel A, B] ← 파티션 → [Sentinel C, Master, Replica]
    
    A, B: Master에 접근 불가 → sdown → odown (A+B = quorum 2)
    → Failover: Replica를 새 Master로 승격
    
    C: Master에 접근 가능 → sdown 아님
    
    결과:
      구 Master: C를 통해 일부 클라이언트 쓰기 수신
      새 Master: A, B를 통해 다른 클라이언트 쓰기 수신
      → 두 "Master"에 서로 다른 데이터 존재

  파티션 해소 후:
    구 Master가 Replica로 강등
    구 Master의 고유 데이터는 새 Master로 덮어씌워짐 → 손실

Split-Brain 완화:
  min-replicas-to-write 1
  min-replicas-max-lag 10

  파티션 시:
    구 Master의 Replica가 새 Master로 이동됨
    → 구 Master: Replica 없음 → min-replicas-to-write 미달 → 쓰기 거부
    → 구 Master에 더 이상 쓰이지 않음 → Split-Brain 데이터 충돌 최소화

  단점:
    파티션 기간 동안 구 Master 쪽 서비스 쓰기 불가
    → 가용성 vs 일관성 트레이드오프
```

---

## 💻 실전 실험

### 실험 1: Sentinel 설정 및 상태 확인

```bash
# Sentinel 설정 파일 (sentinel.conf)
cat > sentinel.conf << 'EOF'
port 26379
sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
sentinel parallel-syncs mymaster 1
EOF

# Sentinel 시작
redis-sentinel sentinel.conf &

# Sentinel 상태 확인
redis-cli -p 26379 SENTINEL masters
redis-cli -p 26379 SENTINEL slaves mymaster
redis-cli -p 26379 SENTINEL sentinels mymaster
redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster
# 현재 Master 주소 반환
```

### 실험 2: Failover 시뮬레이션

```bash
# 현재 Master 확인
redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster

# Sentinel Pub/Sub 구독 (별도 터미널)
redis-cli -p 26379 SUBSCRIBE +switch-master &

# Master 강제 종료 (장애 시뮬레이션)
redis-cli -p 6379 DEBUG SLEEP 100  # 100초 응답 없음

# 또는
# kill -9 $(pgrep redis-server | head -1)

# Failover 로그 모니터링
# Sentinel 로그에서:
# +sdown master mymaster 127.0.0.1 6379
# +odown master mymaster 127.0.0.1 6379 #quorum 2/2
# +start-failover-state-select-slave ...
# +selected-slave slave 127.0.0.1:6380 ...
# +failover-state-send-slaveof-noone slave 127.0.0.1:6380 ...
# +promoted-slave slave 127.0.0.1:6380 ...
# +switch-master mymaster 127.0.0.1 6379 127.0.0.1 6380

# Failover 후 새 Master 확인
redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster
# ["127.0.0.1", "6380"]  ← 새 Master
```

### 실험 3: Sentinel Pub/Sub으로 Failover 감지

```bash
# Sentinel의 이벤트 채널 모두 구독
redis-cli -p 26379 PSUBSCRIBE '*'

# Failover 시 출력 예시:
# +sdown master mymaster 127.0.0.1 6379
# +odown master mymaster 127.0.0.1 6379 #quorum 2/2
# +selected-slave slave 127.0.0.1:6380 mymaster 127.0.0.1 6379
# +switch-master mymaster 127.0.0.1 6379 127.0.0.1 6380

# 주요 채널:
# +sdown:       단일 Sentinel이 sdown 선언
# +odown:       quorum 도달, Failover 시작
# +switch-master: 새 Master 주소 변경
# +slave:       Replica 재연결 완료
```

### 실험 4: Sentinel으로 Master 주소 동적 조회

```bash
# 클라이언트 코드 패턴 시뮬레이션
get_master() {
  RESULT=$(redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster)
  HOST=$(echo $RESULT | awk '{print $1}')
  PORT=$(echo $RESULT | awk '{print $2}')
  echo "Current Master: $HOST:$PORT"
  redis-cli -h $HOST -p $PORT PING
}

# Failover 전
get_master

# Failover 실행 (별도 터미널에서 Master 중단)
# sleep 30 (Failover 대기)

# Failover 후 자동으로 새 Master 주소 반환
get_master
```

---

## 📊 성능/비용 비교

```
Sentinel Failover 타임라인:

단계                                  소요 시간
──────────────────────────────────────────────────────────────
Master 장애 발생                       0s
첫 번째 Sentinel이 ping 응답 없음 감지  ~1s
down-after-milliseconds 도달           30s (기본)
sdown 선언 + odown 확인                 ~1s
Failover 리더 선출                      ~2s
REPLICAOF NO ONE 전송                   ~1s
Replica가 Master로 승격                 ~1s
다른 Replica 재연결                     ~2s
클라이언트 알림 (Pub/Sub)               ~1s
──────────────────────────────────────────────────────────────
총 Failover 완료 시간:                 ~38초 (down-after=30초 기준)

서비스 다운타임:
  Master 장애부터 클라이언트 재연결까지 = ~40~60초
  down-after-milliseconds를 낮추면 빠르지만 오탐 위험 증가

Sentinel 구성별 가용성:

구성             | 과반수 필요 | Sentinel 허용 장애 수
────────────────┼────────────┼─────────────────────
Sentinel 3개     | 2/3       | 1개 장애 시도 Failover 가능
Sentinel 5개     | 3/5       | 2개 장애 시도 Failover 가능
Sentinel 1개     | N/A       | SPOF (절대 금지)
Sentinel 2개     | 2/2       | 1개 장애 시 Failover 불가
```

---

## ⚖️ 트레이드오프

```
down-after-milliseconds 설정:
  낮은 값 (5초):  빠른 장애 감지, 네트워크 순간 지연으로 오탐 가능
                  불필요한 Failover = 데이터 유실 위험
  높은 값 (60초): 오탐 없음, 하지만 실제 장애 시 1분 다운타임

Sentinel vs Cluster:
  Sentinel: Master-Replica + 자동 Failover, 단일 노드 용량 제한
  Cluster:  자동 Failover + 수평 확장 (16384 슬롯), 복잡도 높음
  → 수십 GB 이하 + 단순 구성: Sentinel
  → 수백 GB 이상 + 수평 확장 필요: Cluster

Split-Brain vs 가용성:
  min-replicas-to-write 설정:
    파티션 시 쓰기 거부 → 일관성 보장, 가용성 희생
    미설정 → Split-Brain 가능, 가용성 유지
  → 비즈니스 요구사항에 따라 선택
```

---

## 📌 핵심 정리

```
Redis Sentinel 핵심:

장애 감지:
  sdown: 단일 Sentinel의 주관적 판단 (ping 응답 없음)
  odown: quorum 수 이상 동의 → 객관적 다운 → Failover 시작
  
  설정: sentinel monitor mymaster <ip> <port> <quorum>
  권장 quorum = Sentinel 수의 과반수

Failover 순서:
  odown → 리더 선출 → 최적 Replica 선택 → REPLICAOF NO ONE
  → 다른 Replica 재연결 → Pub/Sub 알림 → 클라이언트 전환

새 Master 선택 기준:
  1. slave-priority 낮은 값
  2. replication offset 큰 값 (가장 최신 데이터)
  3. runid 사전순

클라이언트 설정:
  Sentinel 주소로 연결 (Master 직접 주소 금지)
  Spring: spring.redis.sentinel.master + sentinel.nodes

Split-Brain 완화:
  min-replicas-to-write 1
  min-replicas-max-lag 10

최소 권장 구성:
  Sentinel 3개 (서로 다른 서버/AZ)
  quorum = 2
```

---

## 🤔 생각해볼 문제

**Q1.** Sentinel 3개로 구성된 환경에서, Sentinel 2개가 장애가 났다. 이 상태에서 Master가 장애를 일으키면 자동 Failover가 발생하는가?

<details>
<summary>해설 보기</summary>

**발생하지 않는다.** Quorum = 2인데 살아있는 Sentinel이 1개뿐이다. odown을 선언하려면 최소 2개의 Sentinel이 동의해야 하므로, 1개만으로는 odown 선언 자체가 불가능하다.

이 상태에서는:
- 남은 Sentinel 1개가 sdown을 선언해도 odown으로 진행 불가
- 자동 Failover 없음 → 수동 개입 필요

이것이 Sentinel을 3개 이상 홀수로 구성하는 이유다. 장애 허용 수 = Sentinel 수 - quorum:
- Sentinel 3개, quorum 2: 1개 장애 허용
- Sentinel 5개, quorum 3: 2개 장애 허용

**결론:** Sentinel 1개 장애까지는 자동 Failover 가능. 2개 이상 장애 시 수동 복구 필요.

</details>

---

**Q2.** Failover 후 구 Master가 복구돼서 재시작됐다. 이 순간 무슨 일이 일어나는가? 구 Master의 데이터는 어떻게 되는가?

<details>
<summary>해설 보기</summary>

**과정:**
1. 구 Master 재시작 → Sentinel에 연결
2. Sentinel이 구 Master에게 `REPLICAOF <new-master-ip> <port>` 전송
3. 구 Master가 **Replica로 강등**되어 새 Master를 따르기 시작
4. Full Resync 또는 Partial Resync 실행 (대부분 Full Resync)

**구 Master의 데이터:**
- 구 Master가 Failover 이후에 받은 쓰기 데이터 (Split-Brain 상황이었다면)는 **새 Master의 데이터로 덮어씌워져 손실**된다.
- 이것이 Split-Brain의 핵심 위험이다.

**이를 방지하는 방법:**
```conf
min-replicas-to-write 1
min-replicas-max-lag 10
```
파티션 시 Replica가 없는 구 Master는 쓰기를 거부하므로, 복구 후 덮어씌워질 데이터가 없거나 최소화된다.

**실무 체크리스트:**
- 구 Master 복구 후 `INFO replication`으로 role: slave 확인
- 새 Master와 offset 동기화 완료 확인
- 이후 안정적 운영 재개

</details>

---

**Q3.** `sentinel down-after-milliseconds mymaster 5000`으로 설정했더니 네트워크 순간 지연(500ms)이 발생할 때마다 불필요한 Failover가 발생했다. 해결 방법은?

<details>
<summary>해설 보기</summary>

5,000ms(5초)는 일반적인 네트워크 지연(수 ms~수백 ms)에 비해 충분히 크지만, 네트워크 글리치가 500ms라면 5초 안에 ping 응답이 없는 구간이 누적되거나 패킷 드롭이 반복되면 sdown이 선언될 수 있다.

**해결 방법:**

1. **down-after-milliseconds 증가:**
   ```bash
   redis-cli -p 26379 SENTINEL SET mymaster down-after-milliseconds 15000
   ```
   15초로 늘리면 500ms 지연으로 Failover되지 않음. 단, 실제 장애 시 15초 다운타임.

2. **네트워크 안정화:**
   순간 지연의 근본 원인 해결 (스위치/라우터 설정, NIC 드라이버, CPU 부하 등)

3. **replica-priority 0 설정 (Failover 방지용):**
   특정 Replica를 Master 후보에서 제외해 잘못된 Failover의 영향 최소화

4. **Sentinel 개수 증가 + quorum 조정:**
   Sentinel 5개, quorum 3 → 더 엄격한 다수결 → 오탐 감소

**근본 원칙:**
down-after-milliseconds는 "이 시간이 지나도 응답 없으면 진짜 장애"라는 임계값이다. 서비스의 허용 다운타임과 네트워크 안정성을 함께 고려해 설정해야 한다. 일반적으로 15,000~30,000ms를 권장한다.

</details>

---

<div align="center">

**[⬅️ 이전: Master-Replica 복제](./01-master-replica-psync.md)** | **[홈으로 🏠](../README.md)** | **[다음: Redis Cluster ➡️](./03-cluster-hash-slot.md)**

</div>
