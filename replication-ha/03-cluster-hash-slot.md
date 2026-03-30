# Redis Cluster — 해시 슬롯과 Gossip 프로토콜

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 16384개 해시 슬롯을 사용하는 이유는? 왜 하필 16384인가?
- `CRC16(key) % 16384`가 정확히 어떻게 계산되는가?
- `{hash-tag}`로 여러 키를 같은 슬롯에 강제 배치하는 방법과 주의점은?
- Gossip 프로토콜이 클러스터 상태를 전파하는 방식은?
- `CLUSTER NODES`와 `CLUSTER INFO`로 무엇을 알 수 있는가?
- 멀티키 명령어(`MGET`, `MSET`)가 Cluster에서 제한되는 이유는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Redis Cluster는 데이터를 16384개 슬롯으로 분산한다. 슬롯 계산 방식을 모르면 `MGET`이 왜 오류를 내는지, `{hash-tag}`를 왜 써야 하는지 이해할 수 없다. Gossip 프로토콜의 수렴 시간을 모르면 노드 추가 후 클러스터가 불안정한 이유를 파악할 수 없다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Cluster에서 MGET/MSET이 오류 나는 이유를 모름

  코드:
    redis.mget("user:1", "user:2", "user:3")

  오류:
    CROSSSLOT Keys in request don't hash to the same slot

  원인 모를 때:
    "MGET이 왜 안 되지? 버그인가?" → 스택오버플로우 검색
    → "Cluster에서는 MGET을 못 쓴다" → 각각 GET 3번으로 교체
    → RTT 3배 증가

  올바른 해결:
    {hash-tag} 사용으로 같은 슬롯에 배치
    redis.mget("{user}:1", "{user}:2", "{user}:3")  # 모두 같은 슬롯
    → MGET 가능

실수 2: MULTI/EXEC (트랜잭션)이 Cluster에서 제한적임을 모름

  코드:
    MULTI
    SET order:123 "paid"
    INCR stats:total_orders
    EXEC

  오류: order:123과 stats:total_orders가 다른 슬롯
  → CROSSSLOT 오류

  해결:
    MULTI/EXEC를 단일 슬롯 키들로만 제한
    또는 Lua 스크립트로 같은 슬롯에서 원자적 실행
    EVAL "redis.call('SET', KEYS[1], ARGV[1])..." 1 "{order}:123" "paid"

실수 3: 노드 추가 후 자동으로 데이터가 이동한다고 착각

  "노드 3개에서 6개로 늘렸으니까 자동으로 데이터가 분산되겠지"
  → 실제: 리샤딩(resharding)을 명시적으로 실행해야 함
  → redis-cli --cluster reshard <host>:<port>
  → 리샤딩 전까지 새 노드에 데이터 없음 (슬롯 미배정)
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
Cluster 사용 전 멀티키 명령어 감사:
  애플리케이션 코드에서 MGET, MSET, SUNIONSTORE, ZUNIONSTORE 등 사용 확인
  → 모두 hash-tag 전략으로 같은 슬롯에 배치하거나
  → Pipeline으로 단건 명령어 다수로 교체 (RTT는 1회)

hash-tag 설계 원칙:
  같이 사용되는 키들은 같은 태그로 묶기
    user:1:profile, user:1:settings → {user:1}:profile, {user:1}:settings
  
  주의: 모든 키를 같은 태그로 묶으면 → 단일 노드에 집중 → Cluster 의미 없음
    bad:  {app}:user:1, {app}:order:1  # 모두 같은 슬롯 → 편중
    good: {user:1}:profile, {user:1}:orders  # user별 분산

슬롯 분배 확인:
  redis-cli --cluster check <host>:<port>  # 슬롯 배분 현황
  redis-cli CLUSTER KEYSLOT user:1          # 특정 키의 슬롯 번호
  redis-cli CLUSTER KEYSLOT "{user:1}:profile"  # hash-tag 적용 슬롯

클러스터 상태 모니터링:
  redis-cli CLUSTER INFO | grep cluster_state  # ok 이어야 함
  redis-cli CLUSTER NODES | grep fail         # 장애 노드 확인
```

---

## 🔬 내부 동작 원리

### 1. 16384개 해시 슬롯 — 왜 하필 16384인가

```
설계 결정 배경:

  "슬롯 수가 너무 적으면":
    1000개 슬롯 + 노드 1000개 → 노드당 슬롯 1개 → 확장 한계
    슬롯 이동이 큰 단위 → 리샤딩 시 오랜 불균형

  "슬롯 수가 너무 많으면":
    1,000,000개 슬롯 → Gossip 메시지에 슬롯 정보 포함
    → 슬롯 수 비례 Gossip 메시지 크기 증가
    → 노드 간 네트워크 오버헤드 과다

  16384 = 2^14 선택 이유:
    CRC16 최대값 65536(2^16)의 1/4
    16384개 슬롯 비트맵: 16384 bits = 2 KB
    Gossip 메시지 크기가 2 KB (클러스터 노드 1000개에도 관리 가능)
    노드 최대 1000개 × 16384 슬롯 → 노드당 평균 16개 슬롯

  실제 권장 노드 수: 최대 1000개 (Redis 공식 권장)
  실무 일반적 구성: 3~9개 마스터 노드

슬롯 계산:
  slot = CRC16(key) & 0x3FFF  (= CRC16(key) % 16384)
  
  CRC16: 16비트 해시 → 0~65535 범위
  & 0x3FFF: 하위 14비트 추출 → 0~16383 범위
  
  예시:
    CRC16("user:1") = 47812
    47812 & 0x3FFF = 47812 % 16384 = 15044
    → slot 15044번 노드에서 처리

hash-tag 처리:
  키에 {와 }가 있으면 {} 안의 내용만 해싱
  
  "{user:1}:profile" → CRC16("user:1") % 16384
  "{user:1}:settings" → CRC16("user:1") % 16384
  → 두 키가 같은 슬롯에 배치됨!
  
  "{}" 또는 "{" 없는 경우 → 전체 키 해싱
  "{user:1}{extra}:key" → 첫 번째 {} 안인 "user:1"만 해싱
```

### 2. 슬롯 배분과 클러스터 노드 구성

```
3-마스터 클러스터의 슬롯 배분 예시:

  Node A (10.0.0.1): 슬롯 0~5460
  Node B (10.0.0.2): 슬롯 5461~10922
  Node C (10.0.0.3): 슬롯 10923~16383

  각 마스터 + Replica:
  Node A Master ←→ Node A Replica (Node D)
  Node B Master ←→ Node B Replica (Node E)
  Node C Master ←→ Node C Replica (Node F)

슬롯 매핑 저장:
  각 노드는 전체 슬롯-노드 매핑을 메모리에 유지
  → 어떤 노드에 연결해도 모든 슬롯의 위치를 알고 있음
  → MOVED 리다이렉션으로 올바른 노드로 안내

CLUSTER SLOTS 명령어:
  redis-cli CLUSTER SLOTS
  # 1) 1) (integer) 0          ← 시작 슬롯
  #    2) (integer) 5460       ← 끝 슬롯
  #    3) 1) "10.0.0.1"        ← Master IP
  #       2) (integer) 6379    ← Master Port
  #       3) "node_id_A"
  #    4) 1) "10.0.0.4"        ← Replica IP
  #       2) (integer) 6379
  #       3) "node_id_D"

CLUSTER KEYSLOT 명령어:
  redis-cli CLUSTER KEYSLOT user:1        # → 15044
  redis-cli CLUSTER KEYSLOT "{user:1}:p"  # → 15044 (같은 슬롯)
  redis-cli CLUSTER KEYSLOT order:123     # → 7038 (다른 슬롯)
```

### 3. Gossip 프로토콜 — 클러스터 상태 전파

```
Gossip의 목적:
  각 노드가 전체 클러스터의 상태(노드 목록, 슬롯 배분, 장애 여부)를 알아야 함
  중앙 집중식 메타데이터 서버 없이 분산 방식으로 상태 전파

Gossip 동작 원리:
  각 노드가 주기적으로 (cluster-node-timeout / 2마다):
    랜덤으로 몇 개 노드 선택
    자신이 알고 있는 클러스터 상태 정보를 담은 PING 메시지 전송
    → 수신 노드는 PONG으로 자신의 최신 상태 응답

  Gossip 메시지 내용:
    ① 자신의 ID, IP, 포트, 담당 슬롯 범위
    ② 알고 있는 다른 노드들의 상태 (랜덤 일부)
    ③ 장애로 의심되는 노드 정보 (pfail 플래그)

  상태 수렴:
    각 노드가 여러 이웃으로부터 Gossip을 받으면서
    점차 전체 클러스터 상태를 파악
    노드 N개 → 완전 수렴까지 약 O(log N) Gossip 라운드

노드 장애 감지 (Cluster에서의 Failover):
  ① 노드 A가 노드 X에 PING → 응답 없음 (cluster-node-timeout/2)
  ② 노드 A: X를 pfail(Possible Failure) 표시
  ③ Gossip으로 다른 노드들에게 "X가 응답 없다" 전파
  ④ 다수 노드(과반수)가 X를 pfail로 표시 → X를 fail로 결정
  ⑤ Fail 전파 → X의 Replica 중 하나가 새 Master로 승격
  
  (Sentinel 없이 자체 Failover!)

Gossip 메시지 크기:
  슬롯 비트맵: 2 KB (16384 bits)
  노드 정보: 노드당 약 50 bytes
  100개 노드 클러스터: ~7 KB per Gossip 메시지
  → 1000개 노드: ~52 KB → 여전히 관리 가능
```

### 4. 클러스터에서의 명령어 제한

```
단일 슬롯 명령어 (항상 가능):
  GET, SET, HGET, HSET, LPUSH 등 단일 키 명령어
  → 해당 키의 슬롯을 담당하는 노드에서 실행

멀티키 명령어 제한 (같은 슬롯일 때만 가능):
  MGET user:1 user:2  → user:1과 user:2가 다른 슬롯 → CROSSSLOT 오류
  MSET user:1 "a" user:2 "b" → 같음
  SUNIONSTORE, ZUNIONSTORE, SINTERSTORE → 모든 키가 같은 슬롯이어야 함
  RENAME src dst → src와 dst가 같은 슬롯이어야 함

해결: hash-tag로 같은 슬롯에 강제 배치:
  MGET {user}:1 {user}:2  → 둘 다 CRC16("user") 슬롯 → 가능!

MULTI/EXEC 트랜잭션:
  같은 슬롯의 키들로만 트랜잭션 구성 가능
  다른 슬롯 키 혼합 → CROSSSLOT 오류

Lua 스크립트:
  모든 KEYS[]가 같은 슬롯이어야 함
  EVAL "redis.call('SET', KEYS[1], ARGV[1])" 1 "{order}:123" "paid"
  → {order} 태그로 같은 슬롯 보장 가능

SELECT 명령어:
  Cluster에서 DB 번호 변경 불가 (항상 DB 0)
  → 논리적 DB 분리 불가 → 키 네임스페이스로 대신 구분
```

---

## 💻 실전 실험

### 실험 1: 클러스터 생성 및 슬롯 확인

```bash
# Redis Cluster 생성 (6개 노드: 3 Master + 3 Replica)
redis-cli --cluster create \
  127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003 \
  127.0.0.1:7004 127.0.0.1:7005 127.0.0.1:7006 \
  --cluster-replicas 1

# 클러스터 상태 확인
redis-cli -p 7001 CLUSTER INFO
# cluster_state:ok
# cluster_slots_assigned:16384
# cluster_known_nodes:6

redis-cli -p 7001 CLUSTER NODES
# <id> 127.0.0.1:7001 master - 0 ... 0 connected 0-5460
# <id> 127.0.0.1:7002 master - 0 ... 0 connected 5461-10922
# <id> 127.0.0.1:7003 master - 0 ... 0 connected 10923-16383

# 슬롯 배분 상세 확인
redis-cli -p 7001 CLUSTER SLOTS
```

### 실험 2: hash-tag 동작 확인

```bash
# 슬롯 계산 확인
redis-cli -p 7001 CLUSTER KEYSLOT user:1      # 어떤 슬롯?
redis-cli -p 7001 CLUSTER KEYSLOT user:2      # 다른 슬롯?
redis-cli -p 7001 CLUSTER KEYSLOT "{user}:1"  # user 해싱
redis-cli -p 7001 CLUSTER KEYSLOT "{user}:2"  # 같은 슬롯!

# MGET 오류 확인
redis-cli -c -p 7001 MGET user:1 user:2
# CROSSSLOT (다른 슬롯)

# hash-tag로 해결
redis-cli -c -p 7001 SET "{user}:1" "alice"
redis-cli -c -p 7001 SET "{user}:2" "bob"
redis-cli -c -p 7001 MGET "{user}:1" "{user}:2"
# 성공! (같은 슬롯)

# hash-tag 없이 키 분포 확인
for i in $(seq 1 100); do
  KEY="test:$i"
  SLOT=$(redis-cli -p 7001 CLUSTER KEYSLOT $KEY)
  echo "$KEY → slot $SLOT"
done | awk -F' → ' '{print $2}' | sort -n | uniq -c | head -20
# 슬롯이 다양하게 분산됨
```

### 실험 3: MOVED 리다이렉션 관찰

```bash
# -c 옵션 없이 연결 (자동 리다이렉션 없음)
redis-cli -p 7001 SET user:1 "alice"
# (error) MOVED 15044 127.0.0.1:7003
# user:1은 슬롯 15044 → 7003 노드에 있음!

# -c 옵션으로 자동 리다이렉션
redis-cli -c -p 7001 SET user:1 "alice"
# -> Redirected to slot [15044] located at 127.0.0.1:7003
# OK

# 슬롯 담당 노드 직접 확인
SLOT=$(redis-cli -p 7001 CLUSTER KEYSLOT user:1)
echo "user:1 is in slot $SLOT"
redis-cli -p 7001 CLUSTER SLOTS | grep -A5 "slot $SLOT"
```

### 실험 4: Gossip 프로토콜 통계

```bash
# 노드 간 통신 통계
redis-cli -p 7001 INFO stats | grep -E "total_connections|connected_slaves"

# Gossip 메시지 통계
redis-cli -p 7001 DEBUG SLEEP 0  # 연결 확인
redis-cli -p 7001 INFO cluster | grep -E "cluster_stats_messages"
# cluster_stats_messages_ping_sent: Gossip PING 발송 수
# cluster_stats_messages_pong_received: Gossip PONG 수신 수
# cluster_stats_messages_meet_received: 새 노드 join 메시지

# 클러스터 건강 확인
redis-cli --cluster check 127.0.0.1:7001
# OK (모든 슬롯 커버됨, 노드 정상)
```

---

## 📊 성능/비용 비교

```
Cluster vs Sentinel 비교:

                Cluster                 Sentinel
────────────────────────────────────────────────────────
수평 확장       ✓ (노드 추가)           ✗ (단일 Master 용량)
자동 Failover   ✓ (내장)                ✓ (별도 Sentinel 서버)
복잡도          높음                    낮음
멀티키 명령어   제한 (같은 슬롯)        제한 없음
트랜잭션        같은 슬롯만             제한 없음
Pub/Sub         ✓ (부분 제한)           ✓
최소 서버 수    6대 (3 Master + 3 Replica) 3대 (1 Master + 1 Replica + Sentinel 3개)
권장 데이터 크기 수십 GB~수백 GB+       수십 GB 이하

슬롯 계산 성능:
  CRC16 계산: 수 ns (무시 가능)
  MOVED 리다이렉션: 1회 추가 RTT (~0.5~1 ms)
  hash-tag 사용 시 리다이렉션 없음: 추가 RTT 없음

해시 슬롯 분배 균등성:
  키 100만 개를 균일 랜덤으로 분산 시:
  3 노드 → 슬롯당 ~61개 키 (표준편차 ~7.8)
  → 노드 간 키 수 거의 균등 (CRC16의 고른 분포 덕분)
```

---

## ⚖️ 트레이드오프

```
hash-tag의 트레이드오프:
  장점: MGET, MSET, 트랜잭션, Lua 스크립트 사용 가능
  단점: 동일 태그 키가 모두 같은 슬롯 → 특정 노드에 부하 집중 가능
       (예: {user:popular}:* 키가 수백만 개라면 해당 노드 핫스팟)

Cluster의 멀티키 제한:
  단점: 기존 싱글 Redis 코드를 Cluster로 마이그레이션 시 대규모 수정 필요
  완화: Pipeline으로 단건 명령어 묶어 RTT 감소
       hash-tag로 관련 키 슬롯 통일

노드 수와 복잡도:
  노드 증가 → Gossip 메시지 오버헤드 증가
  → 1000개 노드에서도 허용 가능 수준
  → 실무: 50개 노드 이상은 운영 복잡도 고려

리샤딩(resharding) 비용:
  슬롯 이동 시 해당 슬롯의 키 전체 마이그레이션
  → MIGRATE 명령어로 키 단위 이동 → 수 시간 소요 가능
  → 리샤딩 중 ASK/MOVED 리다이렉션 발생 → 약간의 지연 증가
```

---

## 📌 핵심 정리

```
Redis Cluster 핵심:

해시 슬롯:
  slot = CRC16(key) % 16384  (또는 & 0x3FFF)
  16384개 슬롯을 노드에 균등 배분
  16384 = Gossip 메시지 크기와 확장성의 균형점

hash-tag:
  {tag} 형태로 키 이름에 포함 → {} 안의 내용만 해싱
  같이 사용되는 키들을 같은 슬롯에 배치
  MGET, MSET, 트랜잭션 사용 가능해짐
  단, 같은 태그 과도 집중 → 핫스팟 주의

Gossip 프로토콜:
  노드 간 P2P 상태 전파
  중앙 집중식 없이 클러스터 상태 수렴
  노드 장애 감지 → 자체 Failover (Sentinel 불필요)

주요 명령어:
  CLUSTER INFO         → 클러스터 전체 상태
  CLUSTER NODES        → 노드 목록 + 슬롯 배분
  CLUSTER KEYSLOT key  → 특정 키의 슬롯 번호
  CLUSTER SLOTS        → 슬롯별 노드 정보
  --cluster check      → 클러스터 건강 확인

Cluster vs Sentinel:
  Cluster: 수평 확장 필요 + 멀티키 제한 허용
  Sentinel: 단순 구성 + 멀티키/트랜잭션 자유롭게 사용
```

---

## 🤔 생각해볼 문제

**Q1.** `{order}:1`, `{order}:2`, `{order}:3` 키를 대량으로 저장한다. hash-tag `{order}`를 사용했을 때의 장점과 발생할 수 있는 문제는?

<details>
<summary>해설 보기</summary>

**장점:**
- 세 키가 모두 같은 슬롯에 있으므로 `MGET {order}:1 {order}:2 {order}:3` 가능
- `MULTI/EXEC`로 세 키에 대한 원자적 트랜잭션 가능
- Lua 스크립트에서 세 키를 함께 처리 가능

**문제:**
만약 `{order}:*` 키가 수백만 개라면, 이 모든 키가 **하나의 슬롯**에 집중된다. 6노드 클러스터에서 이론적으로 데이터가 6노드에 분산되어야 하는데, `{order}` 슬롯이 있는 노드 1개에 수백만 개 키가 몰리게 된다. → **Hot Node 문제**

**올바른 설계:**
주문 ID를 tag에 포함해 분산:
```bash
{order:1}:status   # order:1 관련 키들만 같은 슬롯
{order:2}:status   # order:2 관련 키들은 다른 슬롯
```
→ 각 주문의 관련 키들은 같은 슬롯에 (트랜잭션 가능)
→ 다른 주문들은 다른 슬롯에 (노드 분산)

```bash
redis-cli CLUSTER KEYSLOT "{order:1}:status"   # 슬롯 X
redis-cli CLUSTER KEYSLOT "{order:2}:status"   # 슬롯 Y (다름)
```

</details>

---

**Q2.** `CRC16("user:1") % 16384 = 15044`이다. 6노드 클러스터에서 `user:1` 키를 저장하면 어느 노드에 가는가? 그리고 노드를 3개에서 6개로 추가하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

**저장 노드:**
슬롯 15044가 어느 노드에 배정됐는지에 따라 결정된다. 기본 3노드 클러스터에서 슬롯 배분이 다음과 같다면:
- Node A: 0~5460
- Node B: 5461~10922
- Node C: 10923~16383

슬롯 15044는 **Node C (10923~16383)**에 속하므로 Node C에 저장된다.

**노드 6개로 추가 시:**
노드 추가만으로는 데이터가 자동으로 이동하지 않는다. 리샤딩을 명시적으로 실행해야 한다:

```bash
redis-cli --cluster reshard 127.0.0.1:7001
```

리샤딩 후 슬롯이 재배분되면 (예: 노드당 2730~2731개 슬롯):
- 슬롯 15044가 Node F(새 노드)로 이동
- user:1 키도 Node F로 마이그레이션

**리샤딩 중 user:1 접근:**
마이그레이션 진행 중 `user:1`에 접근하면 `ASK` 리다이렉션이 발생한다 (04-cluster-resharding.md 참고).

</details>

---

**Q3.** 클러스터 Gossip 프로토콜에서 노드 1000개 환경에 노드 하나가 새로 참여하면, 전체 클러스터가 새 노드의 존재를 인식하는 데 얼마나 걸리는가?

<details>
<summary>해설 보기</summary>

Gossip은 **확률적 에피데믹 프로토콜**이다. 새 노드가 기존 노드 중 하나에 `CLUSTER MEET` 명령으로 합류하면:

1. 합류 대상 노드가 새 노드 정보를 Gossip으로 이웃에 전파
2. 이웃이 또 이웃에게 전파 → 지수적 전파
3. O(log N) Gossip 라운드 후 전체 수렴

노드 1000개, Gossip 간격 ~100ms(cluster-node-timeout/2 기준):
- log₂(1000) ≈ 10 라운드
- 10 × 100ms = ~1초

**실제 수렴 시간:** 대략 1~5초 (네트워크 조건에 따라)

이 수렴 시간 동안 일부 노드는 새 노드를 모를 수 있다. 클러스터 리샤딩은 Gossip 수렴 완료 후 진행하는 것이 안전하다.

```bash
# Gossip 수렴 확인
redis-cli -p 7001 CLUSTER NODES | wc -l  # 전체 노드 수
redis-cli -p 7007 CLUSTER NODES | wc -l  # 새 노드에서 보이는 노드 수
# 두 값이 같으면 수렴 완료
```

</details>

---

<div align="center">

**[⬅️ 이전: Redis Sentinel](./02-sentinel-failover.md)** | **[홈으로 🏠](../README.md)** | **[다음: 클러스터 리샤딩 ➡️](./04-cluster-resharding.md)**

</div>
