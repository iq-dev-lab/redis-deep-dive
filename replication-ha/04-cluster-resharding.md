# 클러스터 리샤딩 — ASK/MOVED 리다이렉션

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `MOVED`와 `ASK` 리다이렉션의 차이는 무엇인가?
- 슬롯 마이그레이션 중에 어떤 흐름으로 키가 이동하는가?
- 클라이언트가 `MOVED`를 받았을 때 해야 하는 행동과 `ASK`를 받았을 때 해야 하는 행동은 어떻게 다른가?
- Spring Data Redis의 Lettuce/Jedis가 리다이렉션을 자동 처리하는 방식은?
- 리샤딩 중 성능 영향을 최소화하는 방법은?
- `CLUSTER SETSLOT`의 importing/migrating 상태가 의미하는 것은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

클러스터 노드를 추가/제거하면 슬롯을 재배치(리샤딩)해야 한다. 이 과정에서 슬롯이 이동 중일 때 `MOVED`와 `ASK` 두 가지 리다이렉션이 발생한다. 이 차이를 모르면 리다이렉션 오류를 올바르게 처리하지 못하고, 클라이언트 라이브러리가 왜 특정 동작을 하는지 이해할 수 없다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: MOVED와 ASK를 동일하게 처리

  코드:
    try:
        result = redis.get("user:1")
    except RedirectionError as e:
        # MOVED든 ASK든 그냥 재시도
        new_host, new_port = parse_redirect(e.message)
        result = Redis(new_host, new_port).get("user:1")

  문제:
    ASK 처리 시에는 ASKING 명령어를 먼저 보내야 함
    ASKING 없이 ASK 대상 노드에 GET 요청 → MOVED 응답 다시 받음
    → 무한 리다이렉션 루프!

실수 2: 리샤딩 중 슬롯 매핑 캐시를 즉시 업데이트

  내부 동작 오해:
    MOVED를 받으면 → "슬롯 매핑 완전히 바뀐 것" → 전체 캐시 재로드
    
  실제:
    MOVED = 슬롯 이동 완료 → 새 노드로 영구 업데이트 OK
    ASK = 슬롯 이동 중 → 임시 리다이렉션 → 캐시 업데이트 금지!
    ASK에서 캐시 업데이트하면 이동 완료 전 슬롯 매핑 오염

실수 3: 리샤딩을 운영 시간에 실행

  "슬롯 이동은 백그라운드니까 서비스 영향 없겠지"
  
  실제:
    각 슬롯의 키를 MIGRATE 명령어로 이동 (블로킹 명령어)
    키가 많은 슬롯: 수만 개 키 이동 → 수십 초 동안 해당 슬롯 요청 지연
    리샤딩 속도 조절 없이 실행 → ASK 리다이렉션 폭증 → 응답 지연
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
클라이언트 라이브러리의 올바른 리다이렉션 처리:
  MOVED 응답 시:
    → 슬롯 매핑 캐시 영구 업데이트
    → 새 노드에 직접 재요청 (ASKING 없이)
  
  ASK 응답 시:
    → 캐시 업데이트 하지 않음 (임시 리다이렉션)
    → 지정된 노드에 ASKING 먼저 전송
    → 그 다음 원래 명령어 재전송

Lettuce/Jedis는 자동 처리:
  Spring Data Redis (Lettuce): 자동 MOVED/ASK 처리 내장
  → 별도 코드 불필요 (라이브러리가 처리)
  → 단, @EnableRedisCluster 또는 Cluster 설정 필요

리샤딩 트래픽 적은 시간에 실행:
  # 슬롯 이동 속도 제한 (pipeline-size와 sleep 조절)
  redis-cli --cluster reshard <host>:<port> \
    --cluster-from <source-node-id> \
    --cluster-to <dest-node-id> \
    --cluster-slots 100 \
    --cluster-pipeline 10  # 동시 이동 키 수 제한

리샤딩 중 모니터링:
  redis-cli --cluster check <host>:<port>
  # 이동 중 슬롯의 상태 확인
  redis-cli -p <port> CLUSTER NODES | grep "migrating\|importing"
```

---

## 🔬 내부 동작 원리

### 1. MOVED 리다이렉션 — 슬롯이 완전히 이동됨

```
MOVED 발생 조건:
  클라이언트가 노드 A에 요청 → 해당 슬롯은 노드 B에 있음
  (리샤딩이 아니어도 발생 — 초기 클러스터 구성 시도 포함)

MOVED 처리 흐름:
  ┌─────────────────────────────────────────────────────────┐
  │  클라이언트 → Node A: GET user:1                           │
  │  Node A → 클라이언트: MOVED 15044 10.0.0.3:6379            │
  │  (user:1은 slot 15044, 현재 Node C에 있음)                 │
  │                                                         │
  │  클라이언트:                                               │
  │  1. 슬롯 매핑 캐시 업데이트 (slot 15044 → Node C)             │
  │  2. Node C에 GET user:1 재요청                            │
  │  3. Node C → 클라이언트: "alice"                           │
  └─────────────────────────────────────────────────────────┘

MOVED의 의미:
  "이 슬롯은 영구적으로 새 노드에 있음"
  → 클라이언트는 캐시를 영구 업데이트
  → 다음 요청부터 새 노드로 직접

MOVED 최소화:
  클라이언트 시작 시 CLUSTER SLOTS 조회로 전체 슬롯 매핑 캐시 구성
  → 이후 대부분 직접 올바른 노드로 연결
  → MOVED는 리샤딩이나 Failover 후 캐시가 낡았을 때만 발생
```

### 2. ASK 리다이렉션 — 슬롯이 이동 중

```
ASK 발생 조건:
  슬롯이 Node A → Node B로 이동 중인 상태
  일부 키는 A에 있고, 일부 키는 B로 이동됨

슬롯 이동 중 상태:
  Node A: CLUSTER SETSLOT <slot> MIGRATING <node-B-id>
          (이 슬롯은 B로 이동 중 → migrating 상태)
  Node B: CLUSTER SETSLOT <slot> IMPORTING <node-A-id>
          (이 슬롯을 A에서 받는 중 → importing 상태)

ASK 처리 흐름:
  ┌─────────────────────────────────────────────────────────┐
  │  클라이언트 → Node A: GET user:1                           │
  │  Node A 확인: user:1이 아직 A에 있으면 → 정상 반환             │
  │  Node A 확인: user:1이 이미 B로 이동됐으면                    │
  │  Node A → 클라이언트: ASK 15044 10.0.0.2:6379              │
  │  ("임시로 B에서 찾아보세요, 하지만 캐시 업데이트 금지")             │
  │                                                         │
  │  클라이언트:                                               │
  │  1. 슬롯 캐시 업데이트 금지! (임시 리다이렉션)                    │
  │  2. Node B에 ASKING 먼저 전송                              │
  │     (importing 상태 슬롯 접근 허가 요청)                      │
  │  3. Node B에 GET user:1 재요청                             │
  │  4. Node B → 클라이언트: "alice"                           │
  └─────────────────────────────────────────────────────────┘

ASKING 명령어의 역할:
  Node B는 importing 상태의 슬롯에 대해 기본적으로 MOVED를 반환
  → "이 슬롯은 A에서 받는 중이니 A에 물어보세요"
  But ASKING을 받으면 → "한 번은 받아드림" (1회 한정 허가)
  → 이후 GET 명령어가 ASKING 플래그를 따라 처리됨

ASK vs MOVED 요약:
  MOVED: "슬롯이 완전히 저쪽에 있음" → 캐시 업데이트 O
  ASK:   "지금은 저쪽에 있을 수도 있음" → 캐시 업데이트 X, ASKING 필요
```

### 3. 슬롯 마이그레이션 상세 절차

```
슬롯 마이그레이션 (Node A의 슬롯 7000번을 Node B로 이동):

Step 1: 목적지 노드를 importing 상태로 설정
  Node B: CLUSTER SETSLOT 7000 IMPORTING <node-A-id>
  → "슬롯 7000을 A에서 받을 준비"

Step 2: 출발지 노드를 migrating 상태로 설정
  Node A: CLUSTER SETSLOT 7000 MIGRATING <node-B-id>
  → "슬롯 7000의 키들을 B로 보낼 준비"

Step 3: 슬롯 7000의 키들을 하나씩 이동
  Node A: CLUSTER GETKEYSINSLOT 7000 100  # 한 번에 100개씩 조회
  → 반환: ["key1", "key2", ..., "key100"]
  
  Node A: MIGRATE <B-ip> <B-port> "" 0 5000 KEYS key1 key2 ... key100
  → key1~key100을 atomic하게 B로 이전
  → 성공 시 A에서 삭제, B에 저장

Step 4: 반복 (슬롯의 모든 키 이동 완료까지)
  CLUSTER GETKEYSINSLOT 7000 100 → 더 이상 키 없을 때까지

Step 5: 슬롯 배정 완료 선언
  Node A: CLUSTER SETSLOT 7000 NODE <node-B-id>  # "7000은 이제 B꺼"
  Node B: CLUSTER SETSLOT 7000 NODE <node-B-id>  # 동일 확인
  → Gossip으로 전체 클러스터에 전파

키 이동 중 접근:
  key1이 아직 A에 있을 때 → Node A가 정상 반환
  key1이 B로 이동된 후 클라이언트가 A에 요청 → ASK 15044 B
  B에서 key1 조회 → ASKING + GET → 정상 반환
```

### 4. redis-cli --cluster reshard 실행 흐름

```
redis-cli --cluster reshard 127.0.0.1:7001:

  1. 클러스터 정보 조회 (CLUSTER NODES)
  2. 이동할 슬롯 수와 출발/목적지 노드 입력 받기
  3. 슬롯 이동 계획 생성
     (출발 노드에서 목적지 노드로 슬롯 X개 이동)
  4. 각 슬롯에 대해 위 마이그레이션 절차 실행
  5. MIGRATE 명령어로 키 단위 이전
  6. 모든 슬롯 이동 완료 후 Gossip 전파

--cluster-pipeline 옵션:
  MIGRATE 시 한 번에 보내는 키 수 (기본 10)
  늘리면 빠르지만 MIGRATE 블로킹 시간 증가
  줄이면 느리지만 서비스 영향 최소

자동 리샤딩 진행상황 확인:
  redis-cli --cluster check 127.0.0.1:7001
  # [OK] All 16384 slots covered → 완료
  # [ERR] Not all 16384 slots are covered → 진행 중

리샤딩 재개 (중단 후):
  redis-cli --cluster reshard 127.0.0.1:7001 --cluster-yes
  → 이전 상태에서 이어서 진행 (migrating/importing 상태 복원)
```

---

## 💻 실전 실험

### 실험 1: ASK vs MOVED 직접 관찰

```bash
# 클러스터 설정 (6개 노드 기가정)
# user:1의 슬롯 확인
SLOT=$(redis-cli -p 7001 CLUSTER KEYSLOT user:1)
echo "user:1 is in slot $SLOT"

# 잘못된 노드에 직접 요청 (-c 없이)
redis-cli -p 7001 GET user:1
# 만약 7001이 해당 슬롯 담당 아니면:
# (error) MOVED <slot> 127.0.0.1:<correct-port>

# MOVED 자동 처리 (-c 옵션)
redis-cli -c -p 7001 GET user:1
# -> Redirected to slot [<slot>] located at 127.0.0.1:<port>
# "value"

# 리샤딩 시작 (슬롯 이동 중 ASK 발생시키기)
# Step 1: importing 설정
SOURCE_NODE=$(redis-cli -p 7001 CLUSTER NODES | grep "$SLOT" | awk '{print $1}')
DEST_NODE=$(redis-cli -p 7001 CLUSTER NODES | grep "7004" | awk '{print $1}')

redis-cli -p 7004 CLUSTER SETSLOT $SLOT IMPORTING $SOURCE_NODE
redis-cli -p 7001 CLUSTER SETSLOT $SLOT MIGRATING $DEST_NODE

# user:1 저장
redis-cli -c -p 7001 SET user:1 "alice"

# ASK 관찰: 키가 아직 7001에 있지만 슬롯은 이동 중
# 7004에서 직접 요청
redis-cli -p 7004 GET user:1
# (error) MOVED ... → ASKING 없이는 거부

# ASKING 후 요청
redis-cli -p 7004 ASKING
redis-cli -p 7004 GET user:1
# (error) ASK ... → user:1이 아직 7001에 있음을 알림
```

### 실험 2: 슬롯 마이그레이션 단계별 실행

```bash
SLOT=100  # 예시 슬롯 번호

# 해당 슬롯에 테스트 데이터 생성
for i in $(seq 1 100); do
  redis-cli -c -p 7001 SET "slot100:key:$i" "value-$i" 2>/dev/null
done

# 슬롯 100의 키 수 확인
redis-cli -p 7001 CLUSTER GETKEYSINSLOT 100 1000 | wc -l

# 마이그레이션 시작
SOURCE_ID=$(redis-cli -p 7001 CLUSTER NODES | grep "7001.*master" | awk '{print $1}')
DEST_ID=$(redis-cli -p 7004 CLUSTER NODES | grep "7004.*master" | awk '{print $1}')

redis-cli -p 7004 CLUSTER SETSLOT 100 IMPORTING $SOURCE_ID
redis-cli -p 7001 CLUSTER SETSLOT 100 MIGRATING $DEST_ID

# 키 이동 (10개씩)
while true; do
  KEYS=$(redis-cli -p 7001 CLUSTER GETKEYSINSLOT 100 10)
  [ -z "$KEYS" ] && break
  redis-cli -p 7001 MIGRATE 127.0.0.1 7004 "" 0 5000 KEYS $KEYS
  echo "이동 완료: $KEYS"
done

# 슬롯 배정 완료
redis-cli -p 7001 CLUSTER SETSLOT 100 NODE $DEST_ID
redis-cli -p 7004 CLUSTER SETSLOT 100 NODE $DEST_ID

echo "슬롯 100 마이그레이션 완료"
redis-cli -p 7004 CLUSTER NODES | grep "$DEST_ID"
```

### 실험 3: redis-cli --cluster reshard 실행

```bash
# 전체 클러스터 상태 확인
redis-cli --cluster check 127.0.0.1:7001

# 대화형 리샤딩 (1000개 슬롯 이동)
redis-cli --cluster reshard 127.0.0.1:7001

# 비대화형 리샤딩 (스크립트용)
redis-cli --cluster reshard 127.0.0.1:7001 \
  --cluster-from <source-node-id> \
  --cluster-to <dest-node-id> \
  --cluster-slots 1000 \
  --cluster-yes \
  --cluster-pipeline 10

# 진행 상황 모니터링
watch -n 2 'redis-cli --cluster check 127.0.0.1:7001 2>/dev/null | tail -5'
```

### 실험 4: Lettuce 자동 리다이렉션 확인 (Spring)

```java
// Spring Boot application.yml
// spring.redis.cluster.nodes: 127.0.0.1:7001, 127.0.0.1:7002, 127.0.0.1:7003

// Java 코드 (자동 리다이렉션 동작)
@Autowired
StringRedisTemplate redisTemplate;

// 리샤딩 중에도 자동으로 올바른 노드로 재요청
public void testAutoRedirect() {
    // Lettuce가 내부적으로:
    // 1. 슬롯 매핑 캐시에서 노드 찾기
    // 2. MOVED → 캐시 업데이트 + 재요청
    // 3. ASK → 임시 재요청 (캐시 업데이트 없음) + ASKING 전송
    redisTemplate.opsForValue().set("user:1", "alice");  // 자동 처리
    String val = redisTemplate.opsForValue().get("user:1");  // 자동 처리
    System.out.println(val);  // "alice"
}
```

---

## 📊 성능/비용 비교

```
리다이렉션 오버헤드:

MOVED 1회 = 추가 RTT 1번
  클라이언트 → 잘못된 노드 (~0.5 ms)
  MOVED 응답 수신
  클라이언트 → 올바른 노드 (~0.5 ms)
  총 추가 시간: ~1 ms (캐시 업데이트로 다음 요청부터 0)

ASK 1회 = 추가 RTT 2번
  클라이언트 → 잘못된 노드 (~0.5 ms)
  ASK 응답 수신
  클라이언트 → ASKING 전송 (~0.5 ms)
  클라이언트 → 원래 명령어 전송 (~0.5 ms)
  총 추가 시간: ~1.5 ms (캐시 업데이트 없으므로 반복 발생)

리샤딩 중 ASK 비율:
  리샤딩 속도 느림: 슬롯 이동 시간 길어짐 → ASK 지속 시간 길어짐
  리샤딩 속도 빠름: 서비스 영향 더 큼 (MIGRATE 블로킹)
  권장: --cluster-pipeline 10~50, 트래픽 적은 시간대 실행

마이그레이션 성능:
  MIGRATE 1000개 키 (키당 100 bytes): ~수백 ms
  슬롯당 키 많을수록 이동 시간 증가
  이동 중 해당 슬롯 키 접근: ASK 리다이렉션 발생 → ~1.5 ms 추가
```

---

## ⚖️ 트레이드오프

```
리샤딩 속도 vs 서비스 영향:
  빠른 리샤딩 (--cluster-pipeline 100):
    이동 완료 빠름, ASK 기간 짧음
    MIGRATE 블로킹이 길어져 해당 슬롯 요청 순간 지연
  
  느린 리샤딩 (--cluster-pipeline 10):
    이동 완료 느림, ASK 기간 길어짐
    개별 MIGRATE는 짧아 순간 지연 최소

hash-tag 사용의 트레이드오프:
  멀티키 명령어 사용 가능 → 개발 편의성 향상
  특정 노드에 부하 집중 가능 → 클러스터 불균형
  핫스팟 주의 필요 (모니터링 필수)

ASK/MOVED 처리의 클라이언트 부담:
  올바른 처리 필수 (잘못하면 무한 루프)
  Lettuce/Jedis 자동 처리 → 별도 구현 불필요
  단, 커스텀 Redis 클라이언트 작성 시 주의
```

---

## 📌 핵심 정리

```
리다이렉션 요약:

MOVED <slot> <ip>:<port>:
  의미: 슬롯이 완전히 저 노드에 있음
  처리: 슬롯 매핑 캐시 영구 업데이트 + 새 노드에 재요청
  발생: 초기 클러스터 연결, 리샤딩 완료 후 캐시 낡을 때, Failover 후

ASK <slot> <ip>:<port>:
  의미: 슬롯이 이동 중, 임시로 저 노드에서 찾아볼 것
  처리: 캐시 업데이트 금지 + ASKING 명령어 + 재요청
  발생: 리샤딩 진행 중

슬롯 마이그레이션 단계:
  1. CLUSTER SETSLOT importing (목적지)
  2. CLUSTER SETSLOT migrating (출발지)
  3. CLUSTER GETKEYSINSLOT + MIGRATE 반복
  4. CLUSTER SETSLOT node (양쪽 모두)

redis-cli --cluster reshard:
  위 과정 자동화
  --cluster-pipeline: 동시 이동 키 수 (속도 vs 영향)

Spring 클라이언트:
  Lettuce: MOVED/ASK 자동 처리 내장 (cluster 설정 필요)
  Jedis: JedisCluster가 자동 처리
```

---

## 🤔 생각해볼 문제

**Q1.** 클라이언트가 Node A에 `GET user:1`을 보냈더니 `ASK 15044 Node B`를 받았다. 이후 클라이언트는 Node B에 어떤 순서로 명령어를 보내야 하는가?

<details>
<summary>해설 보기</summary>

올바른 순서:
```
1. Node B에 ASKING 전송
2. Node B에 GET user:1 전송 (같은 TCP 연결에서)
```

**왜 ASKING이 필요한가?**
Node B는 슬롯 15044를 `importing` 상태로 가지고 있다. 일반적으로 `importing` 상태의 슬롯에 요청이 오면 Node B는 "이 슬롯은 아직 A에 있습니다"라고 `MOVED`를 반환한다. 하지만 `ASKING` 명령어를 받으면 Node B는 "한 번만 허용"하는 플래그를 설정하고, 이후 오는 명령어를 해당 슬롯에서 처리해준다.

**중요:** ASKING은 1회용이다. 다음 요청에도 `ASK`가 오면 또 ASKING + 요청을 반복해야 한다.

**캐시 업데이트 금지 이유:**
마이그레이션이 완료되지 않았기 때문에, `user:1`이 아직 Node A에 있는 다른 키들도 있을 수 있다. 캐시를 Node B로 업데이트하면 그 키들도 Node B에서 찾게 되어 오류가 발생한다.

</details>

---

**Q2.** 리샤딩 중 `CLUSTER SETSLOT 15044 MIGRATING <node-B-id>`를 설정한 상태에서, user:1 키가 **아직 Node A에** 있다. 클라이언트가 Node A에 `SET user:1 "new_value"`를 실행하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

**정상 처리된다.** `migrating` 상태의 슬롯이라도, 키가 아직 Node A에 있으면 Node A가 정상적으로 처리한다.

`migrating` 상태의 의미:
- "이 슬롯의 키들을 Node B로 이동 중"
- 키가 **A에 있으면**: 정상 처리 (GET/SET 모두 가능)
- 키가 **A에 없으면** (B로 이미 이동됨): `ASK Node B` 반환

따라서 `SET user:1 "new_value"` 실행 시:
1. Node A가 슬롯 15044 → `migrating` 상태 확인
2. user:1이 Node A에 있는지 확인 → 있음
3. 정상 SET 처리 → OK

하지만 이후 MIGRATE로 user:1이 Node B로 이동되면, 이 새 값("new_value")이 함께 이동된다. MIGRATE는 현재 시점의 값을 전송하므로 최신 값이 정확히 이동된다.

</details>

---

**Q3.** 클러스터에 노드를 추가하고 리샤딩 없이 운영하면 어떻게 되는가? 새 노드에 데이터가 저장되는가?

<details>
<summary>해설 보기</summary>

**저장되지 않는다.** 새 노드를 클러스터에 추가(`CLUSTER MEET`)하면 노드는 클러스터의 멤버가 되지만, 슬롯이 배정되지 않은 상태다. 슬롯이 없는 노드는 어떤 키도 담당하지 않으므로 데이터가 저장되지 않는다.

```bash
redis-cli -p 7007 CLUSTER NODES
# <id> 127.0.0.1:7007 master - 0 ... 0 connected
# "connected" 뒤에 슬롯 범위가 없음 → 슬롯 없음
```

이 상태에서 새 노드에 요청하면 모든 키에 대해 `MOVED` 리다이렉션이 발생한다.

**리샤딩으로 슬롯 배정 후 사용 가능:**
```bash
redis-cli --cluster reshard 127.0.0.1:7001 \
  --cluster-to <new-node-id> \
  --cluster-slots 1000
```

리샤딩 완료 후에야 새 노드가 데이터를 담당한다. 이것이 "노드 추가 = 자동 분산"이 아닌 이유다.

</details>

---

<div align="center">

**[⬅️ 이전: Redis Cluster 해시 슬롯](./03-cluster-hash-slot.md)** | **[홈으로 🏠](../README.md)** | **[다음: 복제 일관성 트레이드오프 ➡️](./05-replication-consistency.md)**

</div>
