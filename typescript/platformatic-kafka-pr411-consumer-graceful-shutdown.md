# [OSS 기여] platformatic/kafka - 브로커 장애 중 Consumer 종료 멈춤 수정 (PR #411, MERGED)

작성일: 2026-09-22
저장소: https://github.com/platformatic/kafka
이슈: https://github.com/platformatic/kafka/issues/408
PR: https://github.com/platformatic/kafka/pull/411
상태: Merged (2026-09-22, `main`)
작업 브랜치: `fix-408-consumer-graceful-shutdown`
머지 커밋: `07bc8fb`

---

## 1. 한 줄 요약

`retries: true`인 Consumer는 Kafka 브로커 연결이 끊긴 상태에서 `close()`를 호출하면 그룹 탈퇴 요청을
무한히 재시도하느라 종료되지 않았다. 종료 중에는 Coordinator 탐색과 metadata 갱신을 포함한 일반 그룹
작업 경로를 사용하지 않고, 이미 연결된 Group Coordinator에 탈퇴 요청을 한 번만 보낸 뒤 로컬 자원을
정리하도록 수정했다.

정상적으로 Coordinator에 연결할 수 있을 때는 기존처럼 탈퇴 요청을 보낸다. 사용자가 직접 호출하는
`leaveGroup()`의 오류 처리도 바꾸지 않았다.

---

## 2. 먼저 알아야 할 Kafka 구성 요소

### 2.1 Producer, Broker, Consumer

Kafka에서 Producer는 메시지를 보내고, Broker는 메시지를 저장하며, Consumer는 저장된 메시지를 읽는다.

```text
Producer
   │ 메시지 전송
   ▼
Kafka Broker
   │ 메시지 전달
   ▼
Consumer
```

Broker는 Kafka 서버 한 대를 뜻한다. Kafka 클러스터는 보통 여러 Broker로 구성되고, 각 Broker는 메시지
저장뿐 아니라 metadata 응답, consumer group 관리 같은 역할도 나눠 맡는다.

### 2.2 Topic과 Partition

메시지는 Topic에 저장되고, Topic은 병렬 처리를 위해 여러 Partition으로 나뉠 수 있다.

```text
orders topic

Partition 0: [A] → [B] → [C]
Partition 1: [D] → [E] → [F]
Partition 2: [G] → [H] → [I]
```

같은 Consumer Group 안에서는 하나의 Partition을 동시에 두 Consumer가 담당하지 않는다. Partition이 세
개이고 Consumer도 세 개라면 각 Consumer가 하나씩 맡을 수 있다. Consumer가 더 적으면 한 Consumer가
여러 Partition을 담당하고, Consumer가 더 많으면 일부 Consumer는 할당을 받지 못한다.

### 2.3 Consumer Group

Consumer Group은 여러 Consumer를 하나의 논리적인 작업자로 묶는다.

```text
Consumer Group: payment-service

Partition 0 → Consumer A
Partition 1 → Consumer B
Partition 2 → Consumer C
```

Consumer 하나가 사라지면 해당 Consumer가 담당하던 Partition을 남은 Consumer들에게 다시 나눈다. 이
과정이 rebalance다. 반대로 서로 다른 Consumer Group은 같은 Topic의 메시지를 각자 독립적으로 소비할 수
있다.

### 2.4 Group Coordinator도 Broker다

Group Coordinator라는 별도 서버가 있는 것은 아니다. Kafka 클러스터의 Broker 중 하나가 특정 Consumer
Group을 관리하는 Coordinator 역할을 맡는다.

```text
Kafka Cluster

Broker 1
Broker 2  ← payment-service 그룹의 Coordinator
Broker 3
```

Consumer는 먼저 접속 가능한 Broker에 `FindCoordinator`를 보내 자기 그룹의 Coordinator를 찾는다. 이후
`JoinGroup`, `Heartbeat`, `LeaveGroup` 같은 그룹 요청은 Coordinator 역할을 맡은 Broker로 보낸다.

```text
Consumer ── JoinGroup ──▶ Coordinator Broker
Consumer ── Heartbeat ──▶ Coordinator Broker
Consumer ── LeaveGroup ─▶ Coordinator Broker
```

코드의 `#coordinatorId`는 이 Broker의 node ID를 저장한다. 값이 `null`이면 Coordinator를 아직 모르거나,
기존 Coordinator 연결에 문제가 생겨 다시 찾아야 한다는 뜻이다.

### 2.5 Heartbeat와 session timeout

Consumer는 자신이 살아 있다는 사실을 Coordinator에 주기적으로 알린다.

```text
Consumer ── heartbeat ──▶ Coordinator
Consumer ── heartbeat ──▶ Coordinator
```

정해진 시간 동안 heartbeat가 오지 않으면 Coordinator는 Consumer가 죽었다고 판단하고 그룹에서 제거한
뒤 rebalance를 시작한다. 따라서 Consumer가 `LeaveGroup`을 보내지 못하고 종료되더라도 Kafka는 결국
session timeout을 통해 해당 Consumer를 정리할 수 있다. 다만 정상적인 종료에서는 즉시 `LeaveGroup`을
보내는 편이 rebalance를 빠르게 시작할 수 있어 더 좋다.

---

## 3. 정상적인 `consumer.close()` 흐름

`consumer.close()`는 TCP 소켓 하나만 닫는 함수가 아니다. 실행 중인 소비 작업과 그룹 상태를 순서대로
정리한다.

```text
consumer.close()
       │
       ├─ Consumer를 종료 상태로 표시
       ├─ lag monitoring과 heartbeat 중단
       ├─ 활성 MessagesStream 종료
       ├─ Coordinator에 LeaveGroup 전송
       ├─ memberId, generationId, assignment 정리
       ├─ 기반 ConnectionPool 종료
       └─ close() 완료
```

### 3.1 `kClosed`의 의미

종료가 시작되면 먼저 다음 상태를 기록한다.

```ts
this[kClosed] = true
```

이 시점에는 네트워크 연결과 그룹 membership이 아직 남아 있을 수 있다. 따라서 `kClosed`는 모든 정리가
끝났다는 표시라기보다 새로운 작업을 받지 않고 종료 절차를 진행 중이라는 표시다.

### 3.2 MessagesStream을 먼저 닫는 이유

`MessagesStream`은 Kafka에 Fetch 요청을 반복하며 메시지를 기다린다. Consumer를 닫는 동안 stream을
그대로 두면 이미 `kClosed`가 `true`인 Consumer에 다음 Fetch를 요청해 `Client is closed` 오류가 발생할
수 있다. 강제 종료에서는 활성 stream을 먼저 닫고, stream 종료가 끝난 뒤 그룹 탈퇴를 진행한다.

### 3.3 `LeaveGroup`을 보내는 이유

Consumer가 통보 없이 사라지면 Coordinator는 session timeout까지 기다려야 한다. `LeaveGroup`을 보내면
Coordinator는 즉시 해당 member를 그룹에서 제거하고 필요한 rebalance를 시작할 수 있다.

classic protocol의 요청은 개념적으로 다음 정보를 가진다.

```text
LeaveGroupRequest {
  groupId: "payment-service",
  members: [{ memberId: "consumer-abc123" }]
}
```

코드에서는 그룹 작업 공통 경로를 통해 Coordinator 탐색, metadata 조회, 연결 획득, API 실행 순으로
진행한다.

```text
#performLeaveGroup()
       ↓
#performDeduplicateGroupOperaton()
       ↓
#performGroupOperation()
       ├─ Coordinator 탐색
       ├─ metadata에서 Broker 주소 확인
       ├─ Coordinator 연결 획득
       └─ LeaveGroup API 실행
```

---

## 4. 문제: 무한 재시도와 종료 순서가 서로 물렸다

### 4.1 `retries: true`

Kafka 연결은 Broker 재시작이나 일시적인 네트워크 단절로 실패할 수 있다. `retries: true`는 이런 오류가
복구될 때까지 계속 재시도하게 한다. 평상시에는 Broker가 돌아왔을 때 메시지 소비를 자동으로 재개할 수
있어 유용하다.

하지만 `LeaveGroup`도 같은 그룹 작업 재시도 경로를 사용했다.

```text
consumer.close()
       ↓
LeaveGroup 요청
       ↓
Broker 연결 실패
       ↓
retries: true
       ↓
재시도 대기
       ↓
Coordinator 탐색 및 연결 재시도
       ↓
계속 실패하면 무한 반복
```

### 4.2 ConnectionPool 종료까지 도달하지 못함

Consumer의 기반 연결을 닫는 `super.close()`는 그룹 탈퇴 callback이 끝난 뒤 실행된다.

```ts
closer(force, error => {
  if (error) {
    // 오류 처리
    return
  }

  super.close(/* ... */)
})
```

`LeaveGroup`이 무한 재시도 중이면 callback이 돌아오지 않고, `super.close()`에도 도달하지 못한다.

```text
LeaveGroup 완료 필요
       ↓
super.close()
       ↓
ConnectionPool 종료
       ↓
client:close 이벤트로 남은 retry 정리
```

결국 재시도를 정리하려면 기반 client 종료가 필요하지만, 기반 client를 종료하려면 먼저 재시도 중인
`LeaveGroup`이 끝나야 하는 순환 구조가 됐다. 이 때문에 `await consumer.close()`가 완료되지 않았다.

### 4.3 중복 그룹 작업 제거와 이미 예약된 retry

그룹 작업은 같은 operation을 동시에 여러 번 보내지 않도록 deduplication을 사용한다. 같은 작업이 이미
진행 중이면 새 호출은 기존 작업의 결과를 함께 기다린다.

```text
기존 그룹 작업
   └─ retry timer 대기 중
           ▲
           │ 같은 operationId
close()의 LeaveGroup
   └─ 기존 작업 결과를 기다림
```

따라서 종료 이후에 새 retry만 막는 것으로는 부족했다. 브로커 장애가 `close()`보다 먼저 발생해 기존
그룹 작업의 retry timer가 이미 예약돼 있으면, 종료 작업이 그 결과를 기다릴 수 있기 때문이다.

---

## 5. 재현 과정에서 확인한 함정

처음에는 Consumer와 Kafka 사이에 단순 TCP proxy를 두고 proxy를 내리는 방식으로 장애를 만들었다.
그러나 Kafka의 bootstrap 주소는 최초 진입점일 뿐이다. Consumer는 metadata를 받은 뒤 Broker가 광고한
실제 주소로 직접 연결할 수 있다.

```text
최초 연결:
Consumer → TCP proxy → Kafka

metadata 수신 후:
Consumer ────────────→ Kafka의 advertised address
```

따라서 proxy만 중지한 결과로 실제 Broker 장애가 재현됐다고 볼 수 없었다. 최종 재현에서는 다음 순서를
사용했다.

1. `broker-single` 컨테이너에 Topic을 만들었다.
2. Producer로 메시지 한 개를 전송했다.
3. `retries: true`인 Consumer가 그룹에 참여하고 실제 메시지를 받게 했다.
4. `broker-single` 컨테이너 자체를 중지했다.
5. Broker disconnect와 retry 진입을 확인했다.
6. stream loop를 끝낸 뒤 `consumer.close()`를 호출했다.
7. 수정 전에는 2초 안에 종료되지 않았고, 수정 후에는 제한 시간 안에 완료되는 것을 확인했다.

메시지를 실제로 받은 뒤 Broker를 내린 이유는 Coordinator 탐색, 그룹 참여, Fetch와 네트워크 연결이
모두 만들어진 상태에서 이슈와 같은 실패 경로를 검증하기 위해서다.

---

## 6. 검토한 해결 방법과 한계

### 6.1 종료 후 새 retry만 막기

공통 retry 함수는 다음 재시도를 생략할 조건을 받을 수 있다.

```ts
() => this[kClosed]
```

이 조건은 종료 중 발생한 새 오류가 다음 retry를 예약하지 못하게 한다. 하지만 `close()` 전에 이미
예약된 retry와 deduplicated operation을 즉시 끝내지는 못했다. 단위 테스트는 통과했지만 실제 Broker를
중지한 재현에서는 여전히 종료가 멈췄다.

### 6.2 Consumer 전체의 `isConnected()` 확인

다음으로 Consumer 전체 연결 상태를 확인하는 방법을 검토했다. 하지만 Consumer는 여러 Broker와 연결될
수 있고 `LeaveGroup`에 필요한 것은 전체 연결이 아니라 현재 Group Coordinator 연결이다.

```text
Consumer
   ├─ Broker 1 연결됨
   ├─ Broker 2 연결 끊김 ← Group Coordinator
   └─ Broker 3 연결됨
```

이 상태에서 일부 Broker 연결이 살아 있다는 사실은 `LeaveGroup` 성공 가능성을 보장하지 않는다. 반대로
다른 Broker 연결 일부가 끊겨도 Coordinator 연결이 살아 있다면 정상적으로 탈퇴 요청을 보낼 수 있다.
그래서 검사 범위를 현재 Coordinator 연결 하나로 좁혔다.

### 6.3 Coordinator 연결이 없으면 항상 생략하기

처음에는 Coordinator 연결이 없으면 항상 요청을 생략하도록 구현했다.

```ts
if (!this.#isGroupCoordinatorConnected()) {
  finish()
  return
}
```

이 방식은 장애 중 `close()`에는 맞지만 사용자가 직접 호출하는 `leaveGroup()`까지 바꿨다. 일반
`leaveGroup()`는 Consumer 객체와 ConnectionPool을 계속 사용할 수 있으므로, established connection이
없다면 기존처럼 Coordinator를 찾고 연결을 만들어 탈퇴를 시도해야 한다.

전체 consumer 테스트에서 구버전 `LeaveGroup` API 호출을 검사하는 기존 테스트가 실패하면서 이 회귀를
확인했다. 1차 PR 구현에서는 `kClosed`를 함께 검사해 생략 범위를 실제 종료 과정으로 제한했다.

---

## 7. 첫 PR 구현

### 7.1 현재 Coordinator 연결만 확인하는 helper

```ts
#isGroupCoordinatorConnected(): boolean {
  if (this.#coordinatorId === null) {
    return false
  }

  const broker = this.currentMetadata?.brokers.get(this.#coordinatorId)
  return broker
    ? this[kConnections].getEstablishedConnection(broker)?.isConnected() === true
    : false
}
```

이 helper는 새 연결을 만들지 않는다. 현재 metadata에서 Coordinator Broker를 찾고 ConnectionPool에 이미
성립된 연결이 있는지만 확인한다. 종료 가능 여부를 알아보는 과정에서 새 연결과 retry를 시작하지 않기
위한 선택이다.

### 7.2 종료 중이고 Coordinator 연결도 없을 때만 요청 생략

classic protocol과 consumer group protocol의 종료 경로에 같은 원칙을 적용했다.

```ts
if (this[kClosed] && !this.#isGroupCoordinatorConnected()) {
  finish()
  return
}
```

경우의 수는 다음과 같다.

| 종료 중 | Coordinator 연결 | 동작 |
|---:|---:|---|
| 아니요 | 있음 | 기존처럼 `LeaveGroup` 전송 |
| 아니요 | 없음 | 기존처럼 Coordinator 탐색 및 연결 시도 |
| 예 | 있음 | 가능하면 `LeaveGroup` 전송 |
| 예 | 없음 | 원격 요청을 생략하고 로컬 종료 계속 |

### 7.3 요청 도중 연결이 끊기는 경쟁 상태

연결 확인 직후 Broker가 죽을 수 있다.

```text
Coordinator 연결 확인: 연결됨
       ↓
바로 다음 순간 Broker 장애
       ↓
LeaveGroup 전송 중 NetworkError
```

그래서 사전 검사뿐 아니라 callback에서도 종료 중 발생한 재시도 가능한 네트워크 오류를 구분한다.

```ts
const unavailableBrokerWhileClosing =
  this[kClosed] && NetworkError.isRetryable(error)
```

이 오류만 종료 가능한 상태로 처리하고, 인증 실패나 일반 API 오류처럼 무시하면 안 되는 오류는 기존처럼
호출자에게 전달한다. 기존에 허용하던 `unknownMemberId`도 그대로 유지한다. Coordinator 입장에서 이미
해당 member가 없다면 그룹 탈퇴라는 목적은 달성된 상태이기 때문이다.

### 7.4 종료 중 새 그룹 작업 retry 방지

공통 그룹 작업의 retry 조건에도 종료 상태를 전달했다.

```ts
this[kPerformWithRetry](
  operationId,
  operation,
  callback,
  0,
  [],
  () => this[kClosed]
)
```

Coordinator 연결이 살아 있어 요청을 시작했지만 전송 중 연결이 끊어진 경우, 다음 retry를 새로 예약하지
않도록 하는 안전장치다.

### 7.5 공통 로컬 정리 함수

정상적인 원격 탈퇴와 장애 중 요청 생략이 같은 로컬 정리를 수행하도록 기존 정리 코드를 `finish()`로
묶었다.

```ts
const finish = () => {
  this.memberId = null
  this.generationId = 0
  this.assignments = null
  this.#syncPreferredReadReplicas()
  callback(null)
}
```

consumer group protocol에서는 `memberEpoch`, 내부 assignment 목록도 함께 초기화한다. 두 경로가 별도의
정리 코드를 가지면 이후 필드가 추가됐을 때 한쪽만 갱신될 수 있으므로 같은 함수를 사용했다.

---

## 8. 첫 PR 구현의 종료 흐름

### 8.1 Broker가 정상인 경우

```text
consumer.close()
       ↓
Coordinator 연결 확인: 연결됨
       ↓
LeaveGroup 전송 및 성공
       ↓
로컬 그룹 상태 정리
       ↓
ConnectionPool 종료
       ↓
close() 완료
```

### 8.2 Broker가 이미 끊긴 경우

```text
consumer.close()
       ↓
Coordinator 연결 확인: 연결 없음
       ↓
LeaveGroup 네트워크 요청 생략
       ↓
로컬 그룹 상태 정리
       ↓
ConnectionPool 종료
       ↓
close() 완료
```

### 8.3 요청 도중 연결이 끊긴 경우

```text
Coordinator 연결 확인: 연결됨
       ↓
LeaveGroup 전송 중 NetworkError
       ↓
종료 중이므로 다음 retry 예약 안 함
       ↓
해당 네트워크 오류에 한해 로컬 종료 계속
       ↓
ConnectionPool 종료
```

---

## 9. 회귀 테스트

### 9.1 Broker가 사라진 뒤 `LeaveGroup`을 반복하지 않는지 확인

첫 번째 테스트는 실제 그룹에 참여한 Consumer의 연결 획득이 항상 `NetworkError`를 반환하도록 만들고
`retries: true`에서 `close()`를 호출한다.

```ts
const consumer = createConsumer(t, { retries: true })
await consumer.joinGroup()

let attempts = 0

mockConnectionPoolGet(
  consumer[kConnections],
  () => true,
  undefined,
  undefined,
  (_original, _broker, callback) => {
    attempts++
    callback(new NetworkError('Broker is unavailable.'))
    return true
  }
)

await consumer.close()
```

검증 항목은 다음과 같다.

- 네트워크 작업은 한 번만 시도되고 추가 retry가 생기지 않는다.
- `memberId`가 `null`로 초기화된다.
- `generationId`가 `0`으로 초기화된다.
- Consumer가 닫힌 상태가 된다.

### 9.2 ConnectionPool이 이미 닫힌 상태에서 종료 가능한지 확인

두 번째 테스트는 그룹 참여 후 ConnectionPool을 먼저 닫고 `consumer.close()`를 호출한다.

```ts
const consumer = createConsumer(t)
await consumer.joinGroup()
await consumer[kConnections].close()

await consumer.close()
```

Coordinator에 보낼 수 없는 `LeaveGroup` 때문에 종료가 멈추지 않고, 동일한 로컬 상태 정리가 완료되는지
검증한다.

### 9.3 기존 테스트가 잡아낸 범위 확대

전체 consumer 테스트 첫 실행에서는 기존의 `leaveGroup should use a negotiated legacy API with one member`
테스트가 실패했다. 이 실패로 Coordinator 연결이 없을 때 항상 요청을 생략한 첫 구현이 일반
`leaveGroup()`까지 바꾼다는 사실을 확인했다. 조건을 `this[kClosed]`로 제한한 뒤 이 테스트와 새 종료
테스트가 모두 통과했다.

---

## 10. 첫 PR 구현의 검증 결과

- 실제 `broker-single` 컨테이너 중지 재현: retry 진입 후 `consumer.close()`가 2초 안에 완료
- 종료 관련 대상 테스트: 통과
- `test/clients/consumer/consumer.test.ts`: 197/197 통과
- `npm run typecheck`: 통과
- `npm run lint`: 통과
- `npm run build`: 통과
- `git diff --check`: 통과
- 전체 `npm test`: 2672/2673 통과, 1개 skip

PR CI에서도 같은 `#updateAssignments should handle metadata error` 테스트가 Node 22/Kafka 8.0,
Node 26/Kafka 8.0, Node 26/Kafka 8.1 조합에서만 실패했다. 나머지 matrix와 DCO, API compatibility,
Redpanda smoke test는 통과했다. 이 테스트는 변경 파일과 무관한 consumer group protocol 경로이고,
같은 Node 26 조합은 직전 PR에서도 실패 이력이 있어 이번 수정의 회귀로 보지 않았다. 저장소 권한상
실패 job 재실행은 할 수 없었다.

첫 전체 실행에서는 Kafka 클러스터를 재생성한 직후 Schema Registry의 `_schemas` Topic이 `delete`
정책으로 남아 HTTP 500을 내는 환경 문제가 있었다. Topic을 `compact`로 복구하고 Registry 컨테이너를
재기동한 뒤 Schema Registry 테스트 48개를 다시 실행해 모두 통과시켰고, 전체 `npm test`도 다시 실행해
최종 결과를 확인했다. Consumer group protocol 테스트의 단독 재실행도 통과했다.

Docker 컨테이너를 직접 중지하는 진단 테스트는 로컬 재현에만 사용하고 최종 diff에서는 제거했다. 일반
CI에서 특정 컨테이너 이름과 실행 순서에 의존하며 병렬 Kafka 테스트에 영향을 줄 수 있기 때문이다.

---

## 11. 변경 범위와 호환성

공개 API와 타입은 바꾸지 않았다. 정상적인 종료에서는 이미 연결된 Coordinator에 기존처럼 탈퇴 요청을
전송하며, 사용자가 직접 호출하는 `leaveGroup()`도 기존 Coordinator 탐색, metadata 갱신과 오류 전달
방식을 유지한다.

동작이 달라지는 범위는 다음 조건으로 제한된다.

```text
Consumer.close()가 진행 중
```

이 경우 새 Coordinator 탐색과 metadata 갱신을 하지 않는다. 이미 연결된 Coordinator가 있으면 그
연결로 한 번만 탈퇴를 요청하고, 없다면 원격 탈퇴 통보보다 로컬 애플리케이션 종료를 우선한다. 요청을
보낼 수 없었던 Kafka 쪽 membership은 heartbeat가 중단된 뒤 session timeout으로 정리된다.

---

## 12. PR과 커밋

커밋:

```text
f57bcc7 fix: allow consumer close after broker disconnect
952c407 fix: bound group leave during consumer shutdown
```

커밋에는 저장소 규칙에 맞춰 다음 DCO sign-off를 포함했다.

```text
Signed-off-by: GiHoon1123 <rlaejrqo465@naver.com>
```

PR 본문에는 이슈를 연결하는 `Fixes #408`, 원인과 수정 범위, build/typecheck/lint/전체 테스트 결과,
실제 Broker 중지 재현 결과만 적었다. 작업 브랜치에는 작성자 외의 공동 저자나 도구 표기를 추가하지
않았다.

## 13. 이번 작업에서 배운 점

1. Group Coordinator는 별도 서비스가 아니라 특정 Consumer Group을 담당하는 Kafka Broker다.
2. 정상 종료의 `LeaveGroup`은 빠른 rebalance를 위한 최적화지만, Broker 장애 중 로컬 종료의 필수 조건이
   되어서는 안 된다.
3. 무한 retry가 있는 코드에서는 새 retry 생성뿐 아니라 이미 예약된 timer와 deduplicated operation까지
   확인해야 한다.
4. Kafka bootstrap 주소 앞의 TCP proxy만 끊어서는 장애를 제대로 재현하지 못할 수 있다. metadata가 실제
   Broker의 advertised address를 제공하기 때문이다.
5. 연결 상태는 Client 전체가 아니라 현재 작업에 필요한 대상, 이번 경우에는 Group Coordinator 연결을
   기준으로 판단해야 한다.
6. 새 회귀 테스트만 통과해도 기존 공개 동작을 넓게 바꿨을 수 있다. 전체 대상 테스트에서 발견된 구버전
   `LeaveGroup` 회귀 덕분에 종료 경로와 일반 탈퇴 경로를 정확히 분리할 수 있었다.

최종 원칙은 간단하다. Coordinator에 연결할 수 있을 때는 예의 있게 그룹 탈퇴를 알리고, 이미 연결할 수
없는 종료 상황에서는 복구 시점을 알 수 없는 원격 서버 때문에 로컬 프로세스까지 영원히 붙잡아 두지
않는다.

---

## 14. 1차 구현을 이해하기 위해 보충한 세부 흐름

이 절은 메인테이너 리뷰 전 1차 구현의 판단 과정을 기록한다. 리뷰 후 최종 종료 경로는 15절처럼 일반
그룹 작업과 중첩 retry를 우회하도록 변경됐다.

### 14.1 Broker와 Coordinator를 구분하는 법

`Broker`는 Kafka 서버라는 물리적·논리적 구성 요소를 가리키는 이름이고, `Group Coordinator`는 그
Broker가 특정 Consumer Group에 대해 맡은 역할을 가리키는 이름이다.

```text
Broker 1: 일반 요청 처리
Broker 2: payment-service의 Group Coordinator 역할
Broker 3: 다른 그룹의 Group Coordinator 역할
```

따라서 다음 두 문장은 서로 모순되지 않는다.

```text
Consumer가 Kafka Broker에 LeaveGroup을 보낸다.
Consumer가 Group Coordinator에 LeaveGroup을 보낸다.
```

더 정확하게는 “해당 Consumer Group의 Coordinator 역할을 맡고 있는 Broker에 보낸다”이다. Consumer는
먼저 `FindCoordinator(groupId)`로 Broker의 node ID를 확인하고, metadata에서 그 node ID의 host와 port를
찾아 연결한다.

```text
Consumer
  │ FindCoordinator(groupId)
  ▼
접속 가능한 Broker
  │ "Coordinator는 node 2"
  ▼
metadata.brokers.get(2)
  │
  ▼
Broker 2에 JoinGroup / Heartbeat / LeaveGroup 전송
```

### 14.2 왜 `close()` 전에 LeaveGroup을 시도하는가

`LeaveGroup`은 원격 그룹 상태를 즉시 갱신하기 위한 요청이다. Consumer가 먼저 로컬 ConnectionPool을
닫으면 Coordinator에 정상 탈퇴를 알릴 방법이 없어진다. 그래서 Broker가 살아 있는 정상 종료에서는
다음 순서가 필요하다.

```text
stream 종료
  ↓
heartbeat 중단
  ↓
Coordinator에 LeaveGroup
  ↓
memberId / generationId / assignments 초기화
  ↓
ConnectionPool 종료
```

반대로 Coordinator에 연결할 방법이 없는데도 이 순서를 무조건 지키면 종료 자체가 원격 복구를 기다리는
상태가 된다. 이번 수정은 정상 경로의 순서를 바꾼 것이 아니라, 원격 요청을 수행할 수 없는 경우에만
로컬 정리로 넘어가는 예외 경로를 추가한 것이다.

### 14.3 `kClosed`와 실제 연결 상태는 서로 다른 축이다

종료 상태와 네트워크 상태는 하나의 boolean으로 표현할 수 없다.

```text
                         Coordinator 연결
                      없음              있음
Consumer 종료 중 아님  일반 leaveGroup   정상 요청 가능
Consumer 종료 중       로컬 종료 우선    요청 후 종료
```

`kClosed`만 확인하면 Broker가 살아 있는데도 LeaveGroup을 생략할 수 있다. 연결 상태만 확인하면 일반
`leaveGroup()` 호출에서 기존 동작을 바꿀 수 있다. 최종 조건이 두 값을 함께 검사하는 이유다.

### 14.4 이미 예약된 retry와 새 retry는 다르다

`shouldSkipRetry`는 오류 callback 시점에 다음 timer를 예약할지 결정한다.

```text
작업 실패
  ↓
shouldSkipRetry(error)
  ├─ false → setTimeout으로 다음 작업 예약
  └─ true  → callback으로 최종 오류 전달
```

하지만 다음 상황에서는 timer가 이미 만들어져 있다.

```text
t0: Broker 장애
t1: 그룹 작업 실패
t2: 3초 뒤 실행할 retry timer 생성
t3: 사용자 코드가 consumer.close() 호출
```

`t3`에서 `kClosed`를 true로 바꾸는 것만으로 `t2`의 timer가 과거로 돌아가 취소되지는 않는다. 또한
같은 operation ID의 deduplicated 작업이 있으면 close 경로가 새 작업 대신 그 작업의 완료를 기다릴 수
있다. 그래서 1차 PR 구현은 다음 세 상황을 모두 다룬다.

1. close 시점에 Coordinator 연결이 이미 없으면 그룹 네트워크 작업을 시작하지 않는다.
2. close 이후 발생한 그룹 작업 오류에는 새 retry를 예약하지 않는다.
3. 요청 도중 발생한 재시도 가능한 네트워크 오류는 종료 callback에서 성공적인 로컬 정리로 전환한다.

### 14.5 Classic protocol과 consumer protocol

저장소에는 두 그룹 프로토콜 경로가 있다.

```text
classic protocol
  └─ LeaveGroup API 호출

consumer group protocol
  └─ ConsumerGroupHeartbeat에 memberEpoch = -1 전송
```

둘 다 목적은 그룹에서 빠지는 것이지만 원격 요청의 모양이 다르다. 따라서 한 경로에만 연결 검사를
추가하면 다른 설정의 Consumer에서 같은 hang이 남을 수 있다. 최종 변경은 두 경로 모두에서:

- 종료 중이고 Coordinator 연결이 없으면 원격 요청을 생략한다.
- 공통 로컬 정리 함수로 member 상태와 assignment를 초기화한다.

Consumer group protocol 경로는 원래 heartbeat callback의 오류를 종료 과정에서 허용하는 구조였고,
classic 경로는 `unknownMemberId`와 오류 종류를 구분한다. 이번 변경은 각 경로의 기존 오류 정책을
없애지 않고 Broker 장애 중 종료라는 조건만 추가했다.

### 14.6 실제 장애 재현의 상태 전이

컨테이너 중지 실험에서 관찰한 상태는 다음과 같다.

```text
Consumer 생성
  ↓
Topic metadata 확보
  ↓
Group Coordinator 발견
  ↓
Group join 완료 (memberId / generationId 존재)
  ↓
MessagesStream이 record 수신
  ↓
broker-single 컨테이너 중지
  ↓
broker disconnect 이벤트
  ↓
retry 작업 시작
  ↓
stream.close()
  ↓
consumer.close()
```

수정 전에는 마지막 단계가 기존 retry와 LeaveGroup callback을 기다리며 끝나지 않았다. 수정 후에는
Coordinator 연결이 없다는 사실을 확인하고 다음으로 진행했다.

```text
consumer.close()
  ↓
kClosed = true
  ↓
#isGroupCoordinatorConnected() = false
  ↓
finish()
  ↓
super.close()
  ↓
ConnectionPool의 모든 연결 종료
```

### 14.7 왜 session timeout을 기다려도 괜찮은가

Broker에 연결할 수 없는 상황에서는 `LeaveGroup`을 보내는 것 자체가 불가능하다. 이때 Consumer가
로컬에서 모든 네트워크 자원을 정리하면 heartbeat도 더 이상 전달되지 않는다. Coordinator는 설정된
session timeout 동안 heartbeat를 기다린 후 해당 member를 만료시킨다.

```text
로컬 Consumer 종료
  ↓ heartbeat 중단
Coordinator가 heartbeat 부재를 관찰
  ↓ session timeout
그룹 membership 만료
  ↓
rebalance
```

정상 연결에서 즉시 탈퇴하는 것보다 늦지만, Broker가 죽어 있는 동안 애플리케이션 종료를 무한히
막아 두는 것보다는 올바른 절충이다. 특히 사용자가 새 Consumer를 생성하거나 프로세스를 종료해야
하는 graceful shutdown 상황에서는 이 차이가 중요하다.

### 14.8 테스트가 확인하는 사용자 관점

테스트는 private helper의 반환값만 확인하지 않는다. 사용자가 실제로 보게 되는 결과를 고정한다.

```text
재시도 가능한 장애
  ├─ close() Promise가 끝나는가
  ├─ Consumer가 closed 상태인가
  ├─ memberId가 비워졌는가
  ├─ generationId가 초기화됐는가
  └─ retry가 한 번만 실행됐는가
```

또한 기존 `leaveGroup()` 테스트와 전체 consumer 테스트를 함께 실행해 “종료 중 장애”와 “사용자가
계속 사용할 Consumer에서의 명시적 탈퇴”를 구분했다. 이 구분이 없으면 새 버그를 고치는 과정에서
기존 API의 네트워크 동작을 조용히 바꿀 수 있다.

---

## 15. 메인테이너 리뷰와 2차 수정

### 15.1 리뷰에서 발견된 남은 문제

메인테이너는 최초 수정이 실제 deadlock과 연결이 끊긴 Coordinator에 원격 탈퇴를 시도하지 않는 방향을
올바르게 다뤘다고 확인했다. 다만 다음 종료 조건만으로는 전체 종료 경로가 제한되지 않는다고 지적했다.

```ts
() => this[kClosed]
```

이 조건은 `#performGroupOperation()`을 감싸는 바깥 retry만 중단한다. 내부에서 호출하는
`#findGroupCoordinator()`와 `this[kMetadata]()`는 각각 독립적인 retry 경로를 갖는다.

```text
close()
  ↓
Coordinator 연결 검사 통과
  ↓
#performGroupOperation()
  ├─ #findGroupCoordinator()  ← 자체 retry
  ├─ this[kMetadata]()        ← 자체 retry
  └─ LeaveGroup 요청
```

따라서 연결 검사 직후 metadata 갱신이 실패하고 `retries: true`라면, 바깥 retry 조건을 확인하기 전에
metadata 내부에서 계속 재시도할 수 있었다. 이 경우 그룹 작업 callback이 반환되지 않아
`super.close()`도 실행되지 않는다.

리뷰 요청은 다음 두 가지였다.

1. 중첩된 Coordinator 탐색과 metadata 갱신까지 포함해 종료 시 그룹 탈퇴 경로 전체를 제한하거나 취소할
   수 있게 만들 것
2. `retries: true` 경쟁 상태와 classic 및 consumer group protocol 양쪽의 회귀 테스트를 추가할 것

리뷰: https://github.com/platformatic/kafka/pull/411#issuecomment-5774898280

### 15.2 선택한 해결 방법

종료용 그룹 탈퇴에 일반 그룹 작업 경로를 재사용하지 않도록 바꿨다. `close()`가 시작되면
`kClosed`가 이미 `true`이므로, 캐시된 metadata와 ConnectionPool에서 현재 Coordinator에 이미 성립된
연결만 가져온다.

```ts
#getGroupCoordinatorConnection(): Connection | null {
  if (this.#coordinatorId === null) {
    return null
  }

  const broker = this.currentMetadata?.brokers.get(this.#coordinatorId)
  const connection = broker
    ? this[kConnections].getEstablishedConnection(broker)
    : undefined

  return connection?.isConnected() === true ? connection : null
}
```

종료 중에는 이 연결에 탈퇴 요청을 한 번만 직접 보낸다.

```text
close() 중 그룹 탈퇴
  ├─ 연결된 Coordinator 있음
  │    └─ 기존 연결로 원격 탈퇴 요청을 한 번 전송
  │         └─ 성공 또는 실패 후 로컬 상태 정리
  └─ 연결된 Coordinator 없음
       └─ 원격 요청 없이 로컬 상태 정리
```

이 경로는 `#performDeduplicateGroupOperaton()`과 `#performGroupOperation()`을 호출하지 않는다. 따라서
종료 과정에서 새 Coordinator를 찾거나 metadata를 갱신하지 않고, 그 안의 독립적인 retry loop에도
진입하지 않는다. 요청 자체는 기존 Connection의 timeout 범위 안에서 끝나며 실패 후 다시 보내지 않는다.

일반 사용자가 직접 호출하는 `leaveGroup()`은 `kClosed`가 `false`이므로 기존 그룹 작업 경로를 그대로
사용한다. Coordinator를 다시 찾고 metadata를 갱신하는 정상적인 복구 동작은 유지된다.

기존 바깥 retry 중단 조건도 제거하지 않았다.

```ts
() => this[kClosed]
```

새 종료 경로에는 필요하지 않지만, 종료 직전에 이미 실행 중이던 다른 그룹 작업이 바깥 retry를 새로
예약하지 못하게 하는 보호장치로 남겼다. 두 장치의 역할은 다음처럼 다르다.

```text
기존 조건: 이미 실행 중인 일반 그룹 작업의 바깥 retry 중단
새 종료 경로: close()의 탈퇴가 중첩 retry에 들어가는 것 자체를 방지
```

### 15.3 두 그룹 프로토콜 처리

classic protocol은 기존 연결로 `LeaveGroup`을 한 번 호출한다. 종료 중 발생한 재시도 가능한
`NetworkError`와 이미 탈퇴된 `unknownMemberId`는 로컬 정리로 이어지고, 그 외 오류 전달 규칙은 유지한다.

consumer group protocol은 별도 `LeaveGroup` API 대신
`ConsumerGroupHeartbeat(memberEpoch = -1)`을 한 번 보낸다. 기존과 마찬가지로 종료 과정에서는 이 요청의
오류 때문에 로컬 종료를 막지 않는다.

### 15.4 추가한 회귀 테스트

두 프로토콜 모두 실제 Coordinator 연결을 먼저 성립시킨 뒤 테스트한다. 이렇게 해야 연결이 없어 요청을
생략한 결과를 성공으로 잘못 판단하지 않는다.

- classic protocol에서 `LeaveGroup`이 `NetworkError`를 반환해도 요청 횟수가 1회인지 확인
- consumer group protocol에서 `ConsumerGroupHeartbeat`가 실패해도 요청 횟수가 1회인지 확인
- classic protocol에서 실패하도록 만든 metadata 함수가 종료 중 한 번도 호출되지 않는지 확인
- consumer group protocol에서도 metadata 함수가 호출되지 않는지 확인
- 두 경우 모두 member 및 assignment 상태가 정리되고 `close()`가 완료되는지 확인

metadata 회귀 테스트는 리뷰에서 제시한 정확한 경쟁 상태를 고정한다. `retries: true`이고 Coordinator
연결이 존재해도 종료 경로가 metadata 갱신에 들어가지 않으므로, 실패하도록 만든 metadata callback의
호출 횟수는 `0`이어야 한다.

### 15.5 2차 수정 검증 결과

- 종료 관련 대상 테스트: 11/11 통과
- `test/clients/consumer/consumer.test.ts`: 200/200 통과
- `pnpm run build`: 통과
- `pnpm run lint`: 통과
- `git diff --check`: 통과
- DCO: 통과
- 커밋: `952c407 fix: bound group leave during consumer shutdown`
- push 후 GitHub CI: 새 커밋 기준 matrix 실행 중

이 리뷰를 통해 “바깥 retry를 멈춘다”와 “작업 전체가 종료 가능하다”는 같은 의미가 아니라는 점을
확인했다. 중첩 호출이 각자 retry를 소유한다면 가장 바깥쪽의 취소 조건만으로는 충분하지 않다. 이번
경우에는 모든 내부 retry에 취소 신호를 전달하는 것보다, 종료 시 이미 확보한 연결만 한 번 사용하는
별도 경로가 변경 범위와 종료 동작을 더 명확하게 만들었다.

### 15.6 후속 리뷰 반영

추가 리뷰에서 consumer group protocol의 다음 코드가 불필요한 래퍼라는 지적을 받았다.

```ts
const onLeave = (_error: Error | null): void => finish()
```

이 callback은 오류를 검사하지 않고 항상 같은 `finish()`만 호출한다. 따라서 별도 함수를 두지 않고
종료 callback이 필요한 위치에 `finish`를 직접 전달하도록 정리했다. 동작 변경은 없고, 종료 관련 테스트,
build와 lint를 다시 실행한 뒤 다음 커밋을 push했다.

```text
d7a1f0b refactor: pass shutdown callback directly
```

메인테이너에게 반영 내용과 검증 결과를 짧게 답글로 남겼다.

### 15.7 CI 실패 후 테스트 환경 조건 보완

후속 CI에서 새로 추가한 consumer protocol 테스트가 Kafka 7.5, 7.6 조합에서 실패했다. 실패 내용은
`ConsumerGroupHeartbeat` 요청이 재시도된 것이 아니라, 지원되지 않는 protocol을 실행해 Coordinator
연결이 만들어지지 않으면서 요청 횟수가 `0`이 된 것이었다.

저장소의 `test/clients/consumer/consumer-consumer-group-protocol.test.ts`에는 이미 다음 Kafka 버전을
skip하는 공통 조건이 있었다.

```ts
const skipConsumerGroupProtocol = {
  skip: isKafka(['7.5.0', '7.6.0', '7.7.0', '7.8.0', '7.9.0'])
}
```

추가했던 종료 회귀 테스트도 같은 조건을 사용하도록 수정했다. 따라서 지원하지 않는 Kafka matrix에서는
테스트를 건너뛰고, consumer protocol을 지원하는 환경에서는 실제로 heartbeat 탈퇴가 한 번만 시도되는지
검사한다. 테스트 로직 자체의 기대값을 `0 또는 1`로 느슨하게 바꾸지 않은 이유는, 지원 환경에서는 실제
종료 요청을 반드시 검증해야 하기 때문이다.

검증:

- consumer 관련 대상 테스트 통과
- `test/clients/consumer/consumer.test.ts`: 200/200 통과
- build 통과
- lint 통과
- 커밋: `85fe0be test: skip consumer protocol checks on unsupported kafka`
- 메인테이너에게 CI 원인과 테스트 조건 수정 내용을 답글로 전달
