# Pub/Sub vs Stream — 메시지 보관과 재처리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Pub/Sub의 "Fire-and-Forget" 특성이 비즈니스 이벤트에 왜 위험한가?
- Stream과 Pub/Sub의 결정적 차이 3가지는?
- Consumer Group의 `last-delivered-id`와 PEL이 처리 보장에서 하는 역할은?
- Redis Stream과 Kafka의 적절한 포지셔닝은?
- Pub/Sub이 적합한 사용 사례는 언제인가?
- 메시지 손실 없는 이벤트 처리를 위한 최소 설정은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Redis Pub/Sub과 Stream은 둘 다 "메시지를 보낸다"는 개념이지만, 메시지 보관, 재처리, 순서 보장에서 근본적으로 다르다. 잘못 선택하면 구독자가 없는 순간 발생한 이벤트가 영원히 사라지거나, 처리 도중 크래시한 메시지를 재처리할 방법이 없게 된다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 비즈니스 이벤트에 Pub/Sub 사용

  코드:
    PUBLISH order:events '{"order_id":123,"status":"paid"}'

  장애 시나리오:
    Consumer 서버 재배포(30초) 동안 발행된 이벤트 → 모두 소실
    → 결제된 주문이 처리 안 됨

실수 2: Pub/Sub을 "메시지 큐"라고 오해

  "Redis Pub/Sub으로 작업 큐를 만들었어요"
  → Consumer 1대: SUBSCRIBE tasks
  → Publisher: PUBLISH tasks {...}
  → Consumer가 처리 중 크래시 → 해당 메시지 재처리 방법 없음
  → ACK 개념 없음 → 처리 보장 불가
  → "메시지 큐" 아님, "브로드캐스트 채널"

실수 3: Stream을 도입했지만 XACK를 빠뜨림

  while True:
      msgs = xreadgroup(group, consumer, ">", count=10)
      for msg in msgs:
          process(msg)
      # XACK 누락!
  
  결과: PEL이 계속 쌓임 → XAUTOCLAIM이 같은 메시지를 반복 재전달
  → 중복 처리 + 메모리 누수
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
Pub/Sub이 적합한 경우 (손실 허용):
  실시간 알림 (채팅 메시지 표시)
  라이브 대시보드 업데이트
  캐시 무효화 브로드캐스트
  → 손실 허용 + 즉각 전달 + 영구 저장 불필요

Stream이 적합한 경우 (처리 보장):
  주문/결제 이벤트 처리
  분산 작업 큐
  감사 로그
  → 메시지 보존 + 재처리 보장 + Consumer Group

올바른 Stream 패턴:
  def process_messages():
      # 1. 자신의 pending 먼저 처리 (재시작 후 미처리 복구)
      pending = xreadgroup(group, consumer, "0-0", count=10)
      if pending:
          for msg in pending:
              process_and_ack(msg)
      
      # 2. 새 메시지 처리
      new_msgs = xreadgroup(group, consumer, ">", count=10, block=5000)
      for msg in new_msgs:
          process_and_ack(msg)

  def process_and_ack(msg):
      try:
          do_business_logic(msg)
          xack(stream, group, msg.id)  # 반드시 ACK
      except Exception:
          pass  # ACK 안 함 → 재처리 가능
```

---

## 🔬 내부 동작 원리

### 1. Pub/Sub — Fire-and-Forget 구조

```
Pub/Sub 동작:
  Publisher: PUBLISH channel message
  Subscriber: SUBSCRIBE channel

메모리 구조:
  Redis 내부: 채널별 구독자 목록 (링크드 리스트)
  
  채널 "news": [client_A, client_B, client_C]
  
  PUBLISH news "Hello":
    → client_A에게 전송
    → client_B에게 전송
    → client_C에게 전송
    → 완료, 메시지 메모리에서 즉시 제거

Fire-and-Forget의 의미:
  ① 메시지 저장 없음: PUBLISH 후 Redis에 메시지 없음
  ② 구독자 없으면 소실: 구독자 0명이면 메시지 아무도 못 받음
  ③ ACK 없음: 수신 확인 개념 없음
  ④ 재전달 없음: 수신 실패 시 재시도 불가

PUBLISH 반환값:
  PUBLISH channel message → 정수 (수신한 구독자 수)
  반환값 0 → 아무도 못 받음 (손실)

적합한 사용 사례:
  실시간성이 중요하고 손실 허용:
    채팅 메시지 표시 (일부 메시지 못 봐도 무방)
    라이브 스코어 업데이트 (다음 업데이트가 오면 됨)
    캐시 무효화 (못 받아도 TTL 만료 후 자연 갱신)
    리얼타임 알림 (스낵바, 토스트 메시지)

부적합한 사용 사례:
  결제, 주문, 재고 변경 등 비즈니스 이벤트
  감사 로그 (모든 이벤트 보존 필요)
  분산 작업 큐 (처리 보장 필요)
```

### 2. Redis Stream — 영구 저장 + 처리 보장

```
Stream의 핵심 특성:
  ① 메시지 영구 저장: XADD 후 메시지가 Stream에 보관
  ② Consumer Group: 여러 Consumer가 메시지를 나눠 처리 (단일 전달)
  ③ PEL (Pending Entry List): 미ACK 메시지 추적 → 재처리 보장
  ④ 시간 기반 ID: 조회, 재처리 범위 지정 가능

Pub/Sub vs Stream 비교:

              Pub/Sub                Stream
─────────────────────────────────────────────────────────
메시지 저장    없음 (Fire-and-Forget)  영구 저장 (MAXLEN까지)
구독자 없을 때 소실                    저장됨, 나중에 소비 가능
ACK           없음                    XACK로 처리 확인
재전달        없음                    PEL + XCLAIM/XAUTOCLAIM
Consumer Group 없음 (브로드캐스트)      지원 (단일 전달 보장)
순서 보장     채널 내 순서              Stream 전체 순서 보장
재조회        불가 (사라짐)            ID 범위로 과거 조회 가능
```

### 3. Consumer Group과 PEL 상호작용

```
Consumer Group 생성:
  XGROUP CREATE orders processor $ MKSTREAM
  → $: 이후 추가되는 메시지만 처리 (기존 무시)
  → 0: 처음부터 처리

메시지 읽기:
  XREADGROUP GROUP processor consumer1 COUNT 10 STREAMS orders >
  → >: last-delivered-id 이후 새 메시지만
  → 읽은 메시지: PEL에 자동 추가 (pending 상태)

PEL (Pending Entry List):
  PEL 엔트리: {message_id, consumer_name, delivery_time, delivery_count}
  
  주문 이벤트 처리 흐름:
  
  t=0:  consumer1: XREADGROUP → 메시지 ID 1711234567890-0 수신
  t=0:  PEL: {1711234567890-0, consumer1, 1711234567890, 1}
  t=1:  consumer1: 처리 완료 → XACK orders processor 1711234567890-0
  t=1:  PEL: 1711234567890-0 제거 (처리 완료 확인)

ACK 빠뜨렸을 때:
  PEL: {1711234567890-0, consumer1, 1711234567890, 1} (누적)
  다음번 XREADGROUP 0-0 → pending 메시지 재전달
  XAUTOCLAIM으로 일정 시간 후 다른 Consumer에 재할당

재처리 흐름 (Consumer1 크래시 후):
  XPENDING orders processor - + 10 → pending 목록 확인
  XCLAIM orders processor consumer2 60000 1711234567890-0
  → consumer2가 해당 메시지 처리
  consumer2: XACK orders processor 1711234567890-0 → PEL 제거
```

### 4. Stream vs Kafka 포지셔닝

```
Redis Stream 적합:
  메시지 수 수백만 건 이하
  메모리 내 처리 (빠른 응답)
  Redis를 이미 사용 중 (추가 인프라 불필요)
  Consumer Group 패턴 필요하지만 단순한 경우
  TTL + MAXLEN으로 메시지 크기 제한 허용

Kafka 적합:
  메시지 수 수십억 건 이상
  장기 보존 (수일~수달)
  다수 Consumer Group이 독립적으로 재처리 필요
  높은 처리량 (초당 수백만 건)
  Exactly-Once 의미론 필요

Stream의 실용적 한계:
  메모리 기반 → 용량 제한
  파티셔닝 없음 (Cluster로 근사 가능)
  Exactly-Once 보장 없음 (At-Least-Once)
  복잡한 스트림 처리(Join, Window) 불가

계층적 사용 패턴:
  내부 서비스 통신 → Redis Stream (간단, 빠름)
  서비스 간 이벤트 → Kafka (신뢰성, 확장성)
  실시간 알림 → Redis Pub/Sub (즉각성)
```

---

## 💻 실전 실험

### 실험 1: Pub/Sub 손실 시나리오 재현

```bash
# 터미널 1: 구독 시작
redis-cli SUBSCRIBE news

# 터미널 2: 메시지 발행
redis-cli PUBLISH news "Hello subscribers!"
# 반환: 1 (구독자 1명 수신)

# 터미널 1 종료 (구독자 없음)
# 터미널 2: 메시지 발행 (구독자 없을 때)
redis-cli PUBLISH news "Lost message"
# 반환: 0 (아무도 못 받음!) ← 손실

# 터미널 1 재시작 후: 이 메시지는 영원히 사라짐
redis-cli SUBSCRIBE news  # 재연결해도 "Lost message" 못 받음
```

### 실험 2: Stream 처리 보장 확인

```bash
# Stream 생성 및 Consumer Group 설정
redis-cli XADD orders '*' order_id 123 status paid amount 29900
redis-cli XGROUP CREATE orders processor 0 MKSTREAM

# Consumer1 읽기
redis-cli XREADGROUP GROUP processor consumer1 COUNT 1 STREAMS orders ">"
# 1) 1) "orders"
#    2) 1) 1) "1711234567890-0"
#          2) 1) "order_id" 2) "123" ...

# ACK 없이 Consumer1 종료 시뮬레이션
# PEL 확인 (pending 메시지)
redis-cli XPENDING orders processor - + 10
# 1) 1) "1711234567890-0"      ← 메시지 ID
#    2) "consumer1"             ← 미처리 Consumer
#    3) (integer) 5000          ← 경과 시간 (ms)
#    4) (integer) 1             ← 전달 횟수

# 5초 후 Consumer2가 재처리
sleep 5
redis-cli XAUTOCLAIM orders processor consumer2 4000 0-0 COUNT 10
# → consumer2가 처리 인수

# Consumer2 처리 + ACK
redis-cli XACK orders processor "1711234567890-0"

# PEL 비어있음 확인
redis-cli XPENDING orders processor - + 10
# (empty array)
```

### 실험 3: Stream과 Pub/Sub 동시 사용 패턴

```bash
# 실시간 알림 (Pub/Sub) + 영구 저장 (Stream) 동시 처리
send_order_event() {
  ORDER_ID=$1
  STATUS=$2
  PAYLOAD="{\"order_id\":$ORDER_ID,\"status\":\"$STATUS\"}"
  
  # 1. Stream에 영구 저장 (처리 보장)
  redis-cli XADD order:events MAXLEN "~" 10000 '*' \
    order_id $ORDER_ID status $STATUS
  
  # 2. Pub/Sub으로 실시간 알림 (손실 허용)
  redis-cli PUBLISH order:realtime "$PAYLOAD"
}

send_order_event 123 "paid"
send_order_event 124 "shipped"

# Stream에서 처리 (보장)
redis-cli XRANGE order:events - +

# Pub/Sub에서 실시간 수신 (손실 가능)
# 구독 중인 클라이언트가 있으면 즉시 수신
```

### 실험 4: PEL 모니터링 및 독소 메시지 처리

```bash
redis-cli XGROUP CREATE tasks workers 0 MKSTREAM

# 메시지 추가
for i in $(seq 1 5); do
  redis-cli XADD tasks '*' task_id $i > /dev/null
done

# Worker가 읽고 ACK 하지 않음 (크래시 시뮬레이션)
redis-cli XREADGROUP GROUP workers worker1 COUNT 5 STREAMS tasks ">" > /dev/null

# PEL 상태 확인
redis-cli XPENDING tasks workers - + 10
# 5개 메시지 pending

# 30초 후 독소 메시지 처리 (delivery_count > threshold)
sleep 2  # 짧은 테스트용 대기

# XAUTOCLAIM으로 오래된 메시지 재할당
redis-cli XAUTOCLAIM tasks workers worker2 1000 0-0 COUNT 10

# 처리 후 ACK
for id in $(redis-cli XRANGE tasks - + | awk 'NR%2==1{print $1}'); do
  redis-cli XACK tasks workers $id > /dev/null
done
redis-cli XPENDING tasks workers - + 10  # 빈 리스트
```

---

## 📊 성능/비용 비교

```
Pub/Sub vs Stream vs Kafka:

특성             | Pub/Sub        | Redis Stream   | Kafka
────────────────┼────────────────┼────────────────┼─────────────────
메시지 보존       | 없음             | 메모리 내        | 디스크 (무제한)
처리량 (ops/sec) | 수백만            | 수십만~수백만     | 수백만
지연 시간         | ~0.1 ms         | ~0.5~2 ms     | ~5~20 ms
Consumer Group  | 없음 (브로드캐스트) | 지원            | 지원
재처리            | 불가             | PEL + XCLAIM  | Offset Reset
운영 복잡도        | 최하             | 낮음           | 높음
추가 인프라        | Redis만          | Redis만       | 별도 Kafka 클러스터

XADD 처리량 (Redis 7.0):
  순수 XADD: ~300,000 ops/sec (단순 추가)
  XADD + Consumer Group: ~150,000 ops/sec (PEL 업데이트 포함)
  
PUBLISH 처리량:
  구독자 1명: ~600,000 ops/sec
  구독자 10명: ~100,000 ops/sec (브로드캐스트 부하)
```

---

## ⚖️ 트레이드오프

```
Pub/Sub의 단순성 vs Stream의 신뢰성:

Pub/Sub:
  장점: 구현 단순, 초저지연, 브로드캐스트 효율적
  단점: 손실 가능, 재처리 불가, Consumer Group 없음
  → "보내고 잊어도 되는" 알림에 최적

Stream:
  장점: 영구 저장, 재처리 보장, Consumer Group
  단점: 복잡도 높음, 메모리 사용, PEL 관리 필요
  → "반드시 한 번은 처리돼야 하는" 이벤트에 최적

MAXLEN 설정의 트레이드오프:
  MAXLEN 없음: 무한 메시지 누적 → 메모리 폭발
  MAXLEN 작게: 오래된 메시지 빠르게 삭제 → 재처리 범위 제한
  
  권장: 최대 허용 메모리 기준으로 MAXLEN 설정
  XADD stream MAXLEN ~ 1000000 '*' ...  # 최대 100만 건

At-Least-Once 처리 (Stream):
  장점: 메시지 손실 없음
  단점: 중복 처리 가능 (ACK 전 크래시 시 재전달)
  → 멱등성(Idempotency) 구현 필요
  예: 주문 ID로 중복 처리 방지
```

---

## 📌 핵심 정리

```
Pub/Sub vs Stream:

Pub/Sub:
  Fire-and-Forget: 보내는 순간 저장 없음
  구독자 없으면 소실, ACK 없음
  적합: 실시간 알림, 캐시 무효화 브로드캐스트

Stream:
  영구 저장 (MAXLEN까지)
  Consumer Group: 각 메시지 하나의 Consumer에게만
  PEL: 미ACK 메시지 추적 → 재처리 보장

올바른 Stream 패턴:
  시작 시 pending 먼저 처리 (0-0으로 조회)
  이후 새 메시지 처리 (>로 조회)
  처리 완료 시 반드시 XACK
  독소 메시지 처리 (delivery_count 임계값 초과 시 DLQ)

선택 기준:
  "손실 허용 + 브로드캐스트" → Pub/Sub
  "처리 보장 + 단일 전달"   → Stream
  "대용량 + 장기 보존"      → Kafka
```

---

## 🤔 생각해볼 문제

**Q1.** Redis Pub/Sub에서 구독자가 10명인 채널에 1초에 10만 건을 PUBLISH한다. 구독자 중 1명이 처리 속도가 느려 메시지를 따라잡지 못한다. 어떤 일이 발생하는가?

<details>
<summary>해설 보기</summary>

**클라이언트 출력 버퍼 과부하:**

Redis는 Pub/Sub 메시지를 구독자의 클라이언트 출력 버퍼(client output buffer)에 저장한다. 구독자가 메시지를 빠르게 소비하지 못하면 버퍼가 쌓인다.

```
redis.conf:
client-output-buffer-limit pubsub 32mb 8mb 60
                              │      │   │  └─ 60초간 8mb 이상이면
                              │      │   └─────── 소프트 한계
                              └──────┴─────────── 하드 한계 (즉시 연결 차단)
```

**결과:**
1. 느린 구독자의 출력 버퍼가 32MB 초과 → Redis가 해당 클라이언트 **강제 연결 해제**
2. 연결이 끊어진 구독자는 이후 재연결해도 버퍼에 쌓였던 메시지를 **받을 수 없음** (이미 소실)
3. 다른 9명의 구독자는 정상 서빙

**Redis 서버 영향:**
- 느린 구독자 하나 때문에 메모리 사용량이 32MB 증가
- 버퍼 넘치면 강제 해제 → 구독자 수 감소

**대응:**
- Pub/Sub 대신 Stream 사용 (느린 Consumer는 속도에 맞게 처리, 다른 Consumer에게 영향 없음)
- 구독자 처리 속도 개선 (고루틴/비동기 처리)

</details>

---

**Q2.** Stream Consumer Group에서 `XREADGROUP GROUP g c1 STREAMS s >`와 `XREADGROUP GROUP g c1 STREAMS s 0-0`의 차이는?

<details>
<summary>해설 보기</summary>

**`>` (Greater than):**
- `last-delivered-id` 이후의 **새 메시지**를 요청
- 아직 어떤 Consumer에게도 전달되지 않은 메시지
- 정상적인 메시지 소비에 사용

**`0-0` (or 다른 특정 ID):**
- 이 Consumer의 **PEL에 있는 pending 메시지**를 요청
- 이미 전달됐지만 ACK 안 된 메시지를 다시 받음
- 재시작 후 미처리 메시지 복구에 사용

**실용적 패턴:**
```python
def consume():
    # 1. 재시작 후 자신의 pending 먼저 처리
    pending = xreadgroup(group, consumer, "0-0", count=10)
    if pending:
        for msg in pending:
            process_and_ack(msg)
    
    # 2. 모든 pending 처리 완료 → 새 메시지 처리
    while True:
        new = xreadgroup(group, consumer, ">", count=10, block=5000)
        for msg in new:
            process_and_ack(msg)
```

이 패턴이 "정확히 한 번 처리"에 가장 근접한 방법이다. 재시작 시 0-0부터 확인하므로 크래시 후 미처리 메시지를 복구할 수 있다.

</details>

---

**Q3.** Redis Stream과 Kafka를 같이 사용하는 계층적 아키텍처가 유용한 경우는?

<details>
<summary>해설 보기</summary>

**계층적 이벤트 아키텍처:**

```
[서비스 A] ─XADD→ [Redis Stream: internal:events]
              ↓
        [Stream Processor]
              ↓ 중요 이벤트만 필터링
        [Kafka Producer]
              ↓
        [Kafka: business:events]
              ↓
    [하위 서비스들이 Kafka 구독]
```

**Redis Stream 레이어 (서비스 내부):**
- 서비스 내부의 빠른 이벤트 처리 (초저지연)
- 간단한 작업 큐 (이메일 발송, 알림 등)
- 일시적 이벤트 버퍼 (Kafka가 다운돼도 Redis가 완충)
- 메모리 한도까지 보관 (MAXLEN 설정)

**Kafka 레이어 (서비스 간):**
- 서비스 간 이벤트 전파 (영구 보존)
- 여러 독립 Consumer Group이 각자 재처리
- 대용량 이벤트 스트림
- 이벤트 소싱, 감사 로그

**구체적 예시:**
1. 주문 서비스: XADD → 빠르게 결제 서비스에 알림
2. 결제 서비스: 처리 완료 → Kafka에 publish (장기 보존, 여러 서비스 소비)
3. 정산 서비스 + 알림 서비스: Kafka에서 각자 독립적으로 소비

이 방식의 장점: 서비스 내부에서는 Redis Stream의 단순성, 서비스 간에서는 Kafka의 신뢰성을 각각 활용한다.

</details>

---

<div align="center">

**[⬅️ 이전: 분산 락](./04-distributed-lock.md)** | **[홈으로 🏠](../README.md)** | **[다음: Pipeline과 MULTI/EXEC ➡️](./06-pipeline-transaction.md)**

</div>
