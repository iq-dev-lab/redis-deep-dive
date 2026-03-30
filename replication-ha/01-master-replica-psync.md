# Master-Replica 복제 — PSYNC 내부

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Replica가 최초 연결 시 Full Resync가 발생하는 정확한 과정은?
- `repl_backlog_buffer`란 무엇이고, Partial Resync가 가능한 조건은?
- 복제가 비동기인 이유와 그로 인해 발생하는 데이터 유실 시나리오는?
- 네트워크 단절 후 재연결 시 Full Resync vs Partial Resync 중 어느 것이 선택되는가?
- `INFO replication`의 각 필드가 의미하는 것은?
- Replica에서 읽기 요청을 처리할 때 발생하는 데이터 불일치 가능성은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Master-Replica 복제는 Redis 고가용성의 기반이다. Full Resync는 RDB 생성과 전송을 포함하므로 대용량 인스턴스에서 수 분간 복제 지연이 발생한다. Partial Resync 조건을 모르면 짧은 네트워크 단절이 Full Resync를 유발해 Master에 큰 부하를 준다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Replica 재시작마다 Full Resync 발생 (backlog 설정 부족)

  상황: Replica 서버가 30초 재배포로 재시작
  기대: "짧게 끊겼으니 빠르게 재동기화되겠지"
  실제: repl_backlog_buffer 기본값 1 MB
        30초 × 10 MB/초 쓰기 = 300 MB 밀린 데이터 > 1 MB backlog
        → Full Resync 발생
        → Master BGSAVE → RDB 전송 (수 GB) → 수 분간 복제 지연
        → Master에 큰 부하 + Replica 이 시간 동안 구버전 데이터 서빙

실수 2: Replica 읽기를 완전히 신뢰하는 설계

  코드: "읽기는 Replica에서, 쓰기는 Master에서"
  결과: Master에서 SET key 1 실행
        Replica에서 즉시 GET key → 0 반환 (아직 복제 안 됨!)
        → "방금 저장했는데 왜 못 찾지?" 버그 리포트
        
  원인: 비동기 복제 → Master 쓰기와 Replica 읽기 사이에 지연 존재
        repl_backlog offset 차이 = 지연된 데이터 범위

실수 3: 복제 상태 모니터링 없이 운영

  "Master-Replica 구성 해놨으니 괜찮겠지"
  → 실제로 Replica가 동기화 실패 상태 (replication lag 수백 ms~수 초)
  → 장애 시 Failover 했더니 Replica에 수천 건 데이터 없음
  → INFO replication을 정기적으로 확인하지 않았기 때문
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
repl_backlog_buffer 적절히 설정:
  쓰기 속도 × 최대 단절 예상 시간 × 2
  
  쓰기 10 MB/초 + 재배포 60초 예상:
    10 MB/초 × 60초 × 2(여유) = 1,200 MB
    repl-backlog-size 1200mb  → Partial Resync 보장

INFO replication으로 복제 상태 확인:
  redis-cli INFO replication
  # master_repl_offset: 123456789   ← Master 현재 offset
  # slave0: ip=...,offset=123456780 ← Replica offset
  # offset 차이 = 9 bytes → 거의 동기화됨
  # offset 차이가 크면 → lag 발생 중 → 알림 설정

Replica 읽기 일관성 전략:
  쓰기 직후 읽기 필요 → Master에서 읽기 (일관성 보장)
  약간의 지연 허용 → Replica에서 읽기 (부하 분산)
  애플리케이션에서 명시적 분기 처리 필요

복제 상태 알림:
  # connected_slaves가 0이면 알림
  redis-cli INFO replication | grep connected_slaves
  # master_last_io_seconds_ago가 크면 (예: > 30초) 알림
  redis-cli INFO replication | grep master_last_io_seconds_ago
```

---

## 🔬 내부 동작 원리

### 1. Full Resync — 최초 연결 또는 backlog 소진 시

```
Replica 최초 연결 시 (또는 backlog 소진 시) Full Resync 흐름:

  Replica → Master: PSYNC <replid> <offset>
  
  replid = "?" (최초), offset = -1 (최초)
  또는 replid = 이전 Master ID, offset = 마지막 처리 위치

  Master 판단:
    if (replid 불일치 OR offset이 backlog 범위 밖):
        → Full Resync 결정
    else:
        → Partial Resync (아래 설명)

  Full Resync 과정:
  ┌────────────────────────────────────────────────────────────┐
  │  Master                          Replica                   │
  │                                                            │
  │  "FULLRESYNC <replid> <offset>"  ─────────────────────►    │
  │                                                            │
  │  BGSAVE 실행 (fork())                                       │
  │  ↓ RDB 생성 중 (수십 초~수 분)                                 │
  │  이 시간 동안 새 쓰기는 replication buffer에 버퍼링               │
  │                                                            │
  │  RDB 파일 전송 ──────────────────────────────────────────►   │
  │  (수 MB ~ 수 GB, 네트워크 대역폭 소비)                           │  RDB 로드 (메모리 복원)
  │                                                            │
  │  버퍼링된 명령어 전송 ──────────────────────────────────►       │  명령어 재실행
  │  (RDB 생성 중 쌓인 변경사항)                                    │
  │                                                            │
  │  이후 실시간 복제 스트림 유지                                     │
  └────────────────────────────────────────────────────────────┘

Full Resync의 비용:
  Master: BGSAVE fork() 비용 + RDB 전송 대역폭 소비
  Replica: 기존 데이터 폐기 후 RDB 로드 (이 시간 동안 구데이터 서빙)
  네트워크: RDB 크기만큼의 전송 (4 GB Redis → 수 GB 전송)
  시간: 수십 초 ~ 수 분 (용량/대역폭에 따라)
```

### 2. repl_backlog_buffer와 Partial Resync

```c
/* replication.c */
/* repl_backlog: 순환 버퍼 (circular buffer) */
/* 최근 N bytes의 복제 스트림을 저장 */
/* Replica 재연결 시 backlog에서 이어서 전송 가능 */

구조:
  repl_backlog: [cmd1][cmd2][cmd3]...[cmdN][cmd1][cmd2]... (순환)
  repl_backlog_size: 기본 1 MB (설정 가능)
  repl_backlog_off:  backlog 시작 위치의 전체 offset
  repl_backlog_histlen: 현재 유효 데이터 길이

Partial Resync 조건:
  ① Replica가 가진 replid = Master의 현재 replid (동일 복제 계보)
  ② Replica의 offset이 repl_backlog 범위 안에 있음
     즉, (replica_offset) >= (master_repl_offset - repl_backlog_histlen)
  
  조건 충족 시:
    Master → Replica: "CONTINUE"
    Replica offset부터 backlog의 명령어만 전송
    Full RDB 전송 없음 → 수 ms~수 초로 재동기화 완료

Partial Resync 실패 → Full Resync로 폴백:
  ┌──────────────────────────────────────────────┐
  │  Replica offline 기간 동안 Master에 쓰기 발생     │
  │                                              │
  │  backlog_size = 1 MB                         │
  │  쓰기 속도 = 5 MB/초                            │
  │  offline 시간 = 30초                           │
  │  밀린 데이터 = 5 MB/초 × 30초 = 150 MB           │
  │  → backlog 1 MB 초과 → 데이터 덮어씌워짐           │
  │  → Partial Resync 불가 → Full Resync!         │
  └──────────────────────────────────────────────┘

설정 권장:
  repl-backlog-size 128mb  # 대용량 서버
  # 쓰기 속도 × 최대 단절 예상 시간 × 2 계산
```

### 3. 비동기 복제의 원리와 데이터 유실

```
Redis 복제는 기본적으로 비동기:

  Master:                Replica:
  SET key "value"        ← 아직 미수신
  클라이언트에게 OK 반환  복제 스트림 수신 중...
  ↓ (즉시)              ↓ (약간 후)
  다음 명령어 처리        SET key "value" 적용

  비동기 이유:
    동기 복제 = SET 후 Replica ACK 대기 = 응답 시간 급증
    Redis의 핵심 가치인 낮은 지연을 포기해야 함
    → 비동기 복제로 낮은 지연 유지, 데이터 유실 위험 감수

데이터 유실 시나리오:
  t=0:  Master SET key "order:123" (클라이언트에게 OK 반환)
  t=1:  Master → Replica 복제 스트림 전송 중
  t=2:  Master 서버 장애 (전원 차단)
        복제 스트림이 Replica에 아직 미도달!
  t=3:  Sentinel이 Replica를 새 Master로 승격
  t=4:  클라이언트가 새 Master에서 GET order:123 → nil !
        → 데이터 유실 확정

유실 최소화 방법:
  ① min-replicas-to-write 1  # 최소 1개 Replica가 동기화 확인 후 쓰기 허용
     min-replicas-max-lag 10  # 10초 이상 지연된 Replica는 카운트 안 함
     → Replica가 없거나 지연이 크면 쓰기 거부 (가용성 vs 일관성 트레이드오프)
  
  ② WAIT 명령어 활용 (중요한 쓰기 후):
     SET order:123 "paid"
     WAIT 1 1000  # 1개 Replica에 복제 확인, 1000ms 타임아웃
     → 반환값: 실제로 복제된 Replica 수
     → 0이면 복제 실패 (타임아웃) → 재처리 고려

복제 지연(replication lag) 측정:
  INFO replication의 slave0 offset vs master_repl_offset 차이
  lag = (master_repl_offset - slave_repl_offset) bytes
  bytes를 명령어 수로 환산 불가 (가변 크기)
  하지만 지속적으로 증가하면 → Replica가 따라오지 못하는 신호
```

### 4. 복제 스트림 유지 — 실시간 명령어 전파

```
Full/Partial Resync 완료 후 실시간 복제:

  Master의 모든 쓰기 명령어 → replication buffer에 기록
  → Replica로 실시간 전파 (TCP 스트림)

  전파 형식: RESP 프로토콜 (클라이언트와 동일)
    SET foo bar → Master → Replica: "*3\r\n$3\r\nSET\r\n$3\r\nfoo\r\n$3\r\nbar\r\n"

  Replica의 처리:
    수신한 RESP 명령어를 자신의 메모리에 그대로 실행
    → Master와 동일한 데이터 상태 유지

  ping/pong으로 연결 유지:
    Master → Replica: PING (repl-ping-replica-period, 기본 10초)
    Replica → Master: ACK (offset 포함)
    → 연결 이상 감지 + 복제 지연 계산

복제 필터 (Replica에서 실행 안 되는 것):
  EXPIRE 명령어: Master가 처리 후 DEL 전파 (Replica 독자 처리 없음)
  WAIT 명령어: Replica에서 의미 없음 (Master에서만 의미 있음)
  DEBUG SLEEP: Replica에 전파되지 않음
```

---

## 💻 실전 실험

### 실험 1: Full Resync vs Partial Resync 관찰

```bash
# Master-Replica 설정 (Docker Compose 예시)
docker run -d --name redis-master -p 6379:6379 redis:7.0
docker run -d --name redis-replica -p 6380:6379 redis:7.0 \
  redis-server --replicaof localhost 6379 --repl-backlog-size 1mb

# 초기 Full Resync 확인
docker logs redis-replica | grep -E "SYNC|PSYNC|Full|Partial"
# [SYNC] Full resync from master: <replid>:<offset>

# INFO replication 확인
redis-cli -p 6379 INFO replication
# role:master
# connected_slaves:1
# slave0:ip=127.0.0.1,port=6380,state=online,offset=123,lag=0
# master_repl_offset:123
# repl_backlog_size:1048576  (1 MB)

redis-cli -p 6380 INFO replication
# role:slave
# master_host:127.0.0.1
# master_port:6379
# master_link_status:up
# master_last_io_seconds_ago:1
# master_repl_id:<replid>
# master_repl_offset:123

# Partial Resync 테스트
# 1. Replica 잠시 중단
docker stop redis-replica

# 2. Master에 소량 쓰기 (backlog 안에 들어갈 만큼)
for i in $(seq 1 100); do redis-cli -p 6379 SET "k$i" "v$i" > /dev/null; done

# 3. Replica 재시작
docker start redis-replica
sleep 2

# Partial Resync 확인
docker logs redis-replica | tail -5
# Trying a partial resynchronization (request <replid>:<offset>).
# Successful partial resynchronization with master.
```

### 실험 2: repl_backlog 소진 → Full Resync 강제

```bash
# backlog를 매우 작게 설정
redis-cli -p 6379 CONFIG SET repl-backlog-size 1  # 1 byte!

# Replica 중단
docker stop redis-replica

# Master에 대량 쓰기 (backlog 확실히 초과)
for i in $(seq 1 10000); do redis-cli -p 6379 SET "overflow:$i" "value-$i" > /dev/null; done

# Replica 재시작
docker start redis-replica
sleep 3

# Full Resync 발생 확인
docker logs redis-replica | grep -E "Full|Partial"
# Full resync from master (backlog가 초과됐으므로)

# backlog 복원
redis-cli -p 6379 CONFIG SET repl-backlog-size 1048576
```

### 실험 3: 비동기 복제 지연 측정

```bash
# 복제 지연 측정 스크립트
while true; do
  MASTER_OFF=$(redis-cli -p 6379 INFO replication | grep master_repl_offset | awk -F: '{print $2}' | tr -d ' \r')
  REPLICA_OFF=$(redis-cli -p 6379 INFO replication | grep "slave0:.*offset" | grep -oP "offset=\K[0-9]+")
  LAG=$((MASTER_OFF - REPLICA_OFF))
  echo "$(date +%T) Master: $MASTER_OFF, Replica: $REPLICA_OFF, Lag: $LAG bytes"
  sleep 1
done &

# 대량 쓰기로 지연 유발
for i in $(seq 1 100000); do redis-cli -p 6379 SET "lag:$i" "value-$i" > /dev/null; done
```

### 실험 4: WAIT로 동기 복제 확인

```bash
# 중요 쓰기 후 Replica 복제 확인
redis-cli -p 6379 SET order:123 "paid"
REPLICAS=$(redis-cli -p 6379 WAIT 1 1000)
echo "복제된 Replica 수: $REPLICAS"
# 1이면 복제 성공, 0이면 타임아웃(복제 미완료)

# WAIT 성능 영향 테스트
time redis-cli -p 6379 SET important:key "data"
time redis-cli -p 6379 WAIT 1 5000
# WAIT 추가 시 응답 시간 증가 확인
```

---

## 📊 성능/비용 비교

```
Full Resync vs Partial Resync 비용:

                      Full Resync              Partial Resync
────────────────────────────────────────────────────────────────
소요 시간             수십 초 ~ 수 분           수 ms ~ 수 초
Master 부하           BGSAVE fork() + 전송      backlog 전송만
네트워크 소비         RDB 크기 (수 GB)           backlog 차이 (수 KB~수 MB)
Replica 불일치 시간   전체 복원 시간             복제 재개까지만
트리거 조건           최초 연결, replid 불일치,  동일 replid + offset이 backlog 내
                      backlog 소진

복제 방식별 데이터 안전성:

비동기 (기본):
  응답 시간: 최소 (~1 ms)
  데이터 유실: Failover 시 최근 쓰기 소수 가능
  적합: 캐시, 세션 (일부 유실 허용)

WAIT 1 100 (약한 반동기):
  응답 시간: +1~10 ms (Replica ACK 대기)
  데이터 유실: 타임아웃 내 복제 안 되면 여전히 유실 가능
  적합: 중요 데이터 (적절한 트레이드오프)

min-replicas-to-write 1:
  응답 시간: Replica 없거나 지연 시 쓰기 거부
  데이터 유실: Replica 최소 1개 복제 확인 후 OK
  적합: 높은 일관성 요구 (가용성 일부 포기)

repl_backlog_size 설정 권장:
  쓰기 속도   | 예상 단절 시간   | 권장 backlog
  ──────────┼──────────────┼──────────────
  1 MB/s    | 30초          | 64 MB
  5 MB/s    | 60초          | 600 MB
  10 MB/s   | 60초          | 1.2 GB
  → 여유있게 2배 설정 권장
```

---

## ⚖️ 트레이드오프

```
비동기 복제의 근본 트레이드오프:
  낮은 지연(Low Latency) vs 강한 일관성(Strong Consistency)
  
  Redis의 선택: 낮은 지연 우선
    SET은 Replica 확인 없이 즉시 OK
    → 초당 수십만 건 처리 가능
    → Failover 시 최근 쓰기 소수 유실 가능

  WAIT로 절충:
    중요한 쓰기 후에만 WAIT 사용
    모든 쓰기에 WAIT = 응답 시간 급증 = Redis를 PostgreSQL처럼 쓰는 것

  min-replicas-to-write의 가용성 트레이드오프:
    Replica가 1개인데 min-replicas-to-write 1 설정
    → Replica 장애 시 Master 쓰기 불가! → 서비스 다운
    → 일관성 높이면 가용성 낮아짐

Replica 수와 복제 부하:
  Replica 많을수록 Master의 복제 스트림 전송 부담 증가
  대형 인스턴스: Replica 2~3개 이하 권장
  더 많은 Replica 필요 시: Replica of Replica 구성
    Master → Replica A → Replica B (A를 통해 B가 복제)
    → Master 복제 부하 분산 (단, B의 지연 증가)
```

---

## 📌 핵심 정리

```
Master-Replica 복제 핵심:

Full Resync (비용 큰 경우):
  최초 연결 또는 backlog 소진 시 발생
  Master BGSAVE → RDB 전송 → 버퍼 명령어 전송
  수십 초 ~ 수 분 소요

Partial Resync (효율적):
  동일 replid + offset이 backlog 범위 내
  밀린 backlog 명령어만 전송 → 수 ms~수 초
  
  핵심 설정:
  repl-backlog-size = 쓰기 속도 × 단절 시간 × 2

비동기 복제 데이터 유실:
  WAIT numreplicas timeout → 복제 확인 (선택적 동기화)
  min-replicas-to-write → 쓰기 시 최소 복제 보장 (가용성 트레이드오프)

INFO replication 핵심 필드:
  master_repl_offset    → Master 현재 위치
  slave0: offset        → Replica 위치 (차이 = lag)
  master_link_status    → up/down (복제 연결 상태)
  master_last_io_seconds_ago → 마지막 통신 경과 시간
```

---

## 🤔 생각해볼 문제

**Q1.** Replica가 30초간 네트워크 단절 후 재연결됐다. repl_backlog_size가 1 MB이고 Master의 초당 쓰기가 100 KB라면, Partial Resync가 가능한가?

<details>
<summary>해설 보기</summary>

계산:
- 단절 시간: 30초
- 초당 쓰기: 100 KB
- 밀린 데이터: 30 × 100 KB = 3,000 KB = 3 MB

repl_backlog_size = 1 MB이므로 3 MB > 1 MB → **backlog 소진** → **Full Resync 발생**.

Partial Resync를 유지하려면:
- repl-backlog-size를 최소 6 MB (3 MB × 2 여유) 이상으로 설정
- 또는 단절 시간을 10초 이내로 제한 (100 KB × 10초 = 1 MB 이하)

```bash
redis-cli CONFIG SET repl-backlog-size 10485760  # 10 MB
```

이것이 재배포 전 backlog 크기를 적절히 계산해야 하는 이유다.

</details>

---

**Q2.** `WAIT 1 0`을 실행했더니 0을 반환했다. 이것은 복제가 실패했다는 의미인가?

<details>
<summary>해설 보기</summary>

**반드시 복제 실패는 아니다.** `WAIT 1 0`에서 timeout=0은 "즉시 반환"을 의미한다. 반환값 0은 **현재 시점에 복제를 확인한 Replica가 0개**라는 의미다.

가능한 시나리오:
1. Replica가 없는 경우 → 0 반환 (당연)
2. Replica가 있지만 아직 복제 스트림이 도달하지 않은 경우 → 0 반환 (비동기라 당연히 가능)
3. Replica와 연결이 끊긴 경우 → 0 반환

`WAIT 1 1000`(1초 대기)을 써야 의미 있는 결과를 얻을 수 있다. timeout=0은 사실상 "지금 이 순간 이미 복제된 Replica 수"를 물어보는 것으로, 방금 쓴 데이터의 복제 확인에는 적합하지 않다.

```bash
SET key value
WAIT 1 1000  # 1초 안에 1개 Replica에 복제됐는지 확인
# 반환값 1: 복제 성공
# 반환값 0: 1초 타임아웃 (복제 미완료 또는 Replica 없음)
```

</details>

---

**Q3.** Master-Replica 구성에서 Master가 장애로 종료됐다. Replica를 수동으로 새 Master로 승격시킬 때 (`REPLICAOF NO ONE`), 이전 Master에 있던 데이터 중 일부가 Replica에 없을 수 있는 이유와 이를 최소화하는 방법은?

<details>
<summary>해설 보기</summary>

**이유: 비동기 복제 지연 (replication lag)**

Master에서 쓰기가 발생하고 OK를 반환했지만, 그 쓰기가 Replica에 전파되기 전에 Master가 장애로 종료되면 해당 데이터는 Replica에 없다. Master의 `master_repl_offset`과 Replica의 `slave_repl_offset` 차이만큼 데이터가 손실된다.

**최소화 방법:**

1. **WAIT 사용 (중요한 쓰기 후):**
   ```bash
   SET critical:data "value"
   WAIT 1 500  # 500ms 내 복제 확인
   ```

2. **min-replicas-to-write + min-replicas-max-lag:**
   ```conf
   min-replicas-to-write 1
   min-replicas-max-lag 10
   ```
   → Replica가 10초 이상 지연되면 Master 쓰기 거부 → 유실 방지 (가용성 포기)

3. **AOF everysec + 복제 조합:**
   Master에 AOF 설정 → 장애 복구 시 AOF로 최대 1초 복원 후 Replica와 비교

4. **Sentinel 또는 Cluster 사용:**
   자동 Failover 시 "가장 최신 offset을 가진 Replica"를 새 Master로 선택 → 손실 최소화

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: Redis Sentinel ➡️](./02-sentinel-failover.md)**

</div>
