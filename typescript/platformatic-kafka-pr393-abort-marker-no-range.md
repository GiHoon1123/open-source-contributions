# [OSS 기여] platformatic/kafka - 매칭되는 aborted transaction이 없는 control marker 처리 (PR #393, MERGED)

상태: 🟢 MERGED (2026-09-07)  
저장소: https://github.com/platformatic/kafka  
이슈: https://github.com/platformatic/kafka/issues/391 (머지로 자동 종료)  
PR: https://github.com/platformatic/kafka/pull/393  
브랜치: `fix-391-abort-marker-no-range`  
최종 커밋: `3309535d4f0f6b0337d4d8d0e4674f7419032bb2`  
머지 커밋: `c92e42961af9fa1f0b1a062331679a47c930463c` (ShogunPanda)

이 문서는 PR의 결과만 적는 기록이 아니라, Kafka의 fetch 응답을 어떻게 해석해야 하는지와 이 버그를 어떤 순서로 좁혔는지를 다시 볼 수 있도록 작성한 복습 노트다.

## 1. 먼저 이해할 것: Kafka가 반환하는 두 종류의 정보

이 문제는 일반 레코드만 읽을 때는 드러나지 않는다. `READ_COMMITTED` 소비자는 fetch 응답에서 다음 두 정보를 함께 사용한다.

- `records`: 실제 데이터 레코드와 트랜잭션 종료를 알리는 control record가 섞인 레코드 배치
- `abortedTransactions`: 응답에 포함된 aborted transaction의 producer ID와 시작 오프셋

Kafka의 트랜잭션 데이터는 대략 다음 순서로 로그에 들어간다.

```text
offset 0  transaction data (aborted)
offset 1  abort control marker
offset 2  ordinary committed data
```

control marker 자체도 `records`에 포함될 수 있다. 하지만 브로커가 `abortedTransactions`를 구성할 때는 fetch 요청 오프셋과 트랜잭션의 시작 오프셋을 기준으로 범위를 잘라낸다. 요청이 이미 트랜잭션 시작 지점을 지난 뒤라면 marker는 보이는데 해당 producer의 transaction entry는 응답 목록에 없을 수 있다.

따라서 다음 두 목록이 항상 일대일로 대응한다고 가정하면 안 된다.

```text
abortedTransactions: producer 99
records:             producer 5의 abort marker, producer 5의 committed record
```

이 불일치는 브로커의 잘못이라기보다 fetch 응답이 표현할 수 있는 정상적인 경계 조건이다. 클라이언트는 목록에 없는 producer를 만났을 때도 안전하게 처리해야 한다.

## 2. 원래 코드의 동작

문제가 있던 코드는 `src/clients/consumer/consumer.ts`의 `#filterUncommittedMessages` 안에 있었다.

```ts
const abortedRanges = new Map<bigint, [bigint, bigint]>()

for (const aborted of abortedTransactions) {
  abortedRanges.set(aborted.producerId, [aborted.firstOffset, -1n])
}

for (const batch of partition.records!) {
  if (batch.attributes & IS_CONTROL && batch.records[0].key?.readInt16BE(2) === 0) {
    abortedRanges.get(batch.producerId)![1] = batch.firstOffset
  }
}
```

처리 순서는 다음과 같다.

1. fetch 응답의 `abortedTransactions`를 producer ID별 map으로 만든다.
2. 각 transaction의 종료 marker를 `records`에서 찾는다.
3. marker의 `producerId`를 map에서 찾아 `[firstOffset, endOffset]` 범위를 완성한다.
4. 완성된 범위에 속한 일반 레코드를 `READ_COMMITTED` 결과에서 제거한다.

문제는 3번에서 `Map.get()`이 항상 값을 반환한다고 가정한 점이다. 매칭되는 entry가 없으면 `undefined`인데, non-null assertion(`!`)은 런타임 검사가 아니므로 다음 대입이 그대로 실행된다.

```ts
undefined[1] = batch.firstOffset
// TypeError: Cannot set properties of undefined (setting '1')
```

즉 필터링 결과가 조금 틀리는 수준이 아니라, 소비자 fetch 자체가 예외로 끝난다.

## 3. 이슈의 재현 조건

이슈 #391은 다음 상황에서 발생한다.

1. 한 파티션에 aborted transaction의 데이터와 abort marker를 기록한다.
2. 같은 consumer group의 다른 소비자가 `READ_UNCOMMITTED`로 앞부분을 읽고 오프셋을 커밋한다.
3. 다음 소비자가 `READ_COMMITTED`로 그 오프셋부터 fetch한다.
4. 요청 오프셋은 transaction의 시작 오프셋보다 뒤에 있지만, 응답 레코드에는 abort marker가 포함된다.
5. 브로커 응답의 `abortedTransactions`에는 marker의 producer가 포함되지 않을 수 있다.
6. `abortedRanges.get(batch.producerId)`가 `undefined`가 되어 consumer가 크래시한다.

이 조건에서 marker는 필터링할 데이터가 아니라 transaction 범위를 닫는 보조 정보다. 닫을 시작 범위가 map에 없으므로 marker를 무시하는 것이 안전한 동작이다.

## 4. 실제 브로커에서 재현을 시도한 과정

이슈에 포함된 재현 스크립트를 로컬 Kafka 브로커에 실행했다. 각 fetch 오프셋에서 `abortedTransactions`와 `records`를 함께 출력해 브로커가 실제로 어떤 응답을 만드는지 확인했다.

단일 브로커에서는 다음과 같이 나왔다.

```text
fetch offset 0: abortedTransactions=[{ producerId: 2, firstOffset: 0 }]
               records=[marker@1, data@2, marker@3]

fetch offset 1: abortedTransactions=[{ producerId: 2, firstOffset: 0 }]
               records=[marker@1, data@2, marker@3]

fetch offset 2: abortedTransactions=[]
               records=[data@2, marker@3]
```

이 환경에서는 marker와 transaction 데이터가 같은 로그 세그먼트에 있어서 이슈의 `undefined` 상태가 만들어지지 않았다. 세그먼트 롤과 fetch 시작 위치까지 통제해야 실제 브로커에서 같은 응답을 재현할 수 있는데, 그 작업은 이번 PR의 목적보다 범위가 컸다.

여기서 중요한 결론은 “로컬 브로커에서 예외가 안 나왔다”가 버그가 없다는 뜻이 아니라는 점이다. 코드가 `abortedTransactions`에 없는 producer를 처리할 수 있어야 한다는 조건은 브로커 응답만으로도 명확했고, 브로커의 세그먼트 배치에 테스트를 묶는 것은 회귀 테스트로 적절하지 않았다.

## 5. synthetic FetchResponse로 만든 회귀 테스트

실제 브로커 타이밍 대신 `test/clients/consumer/consumer.test.ts`에서 fetch 응답을 직접 구성했다. 이 테스트는 네트워크나 로그 세그먼트에 의존하지 않고 문제의 입력만 고정한다.

```ts
const response: FetchResponse = {
  // ...
  responses: [{
    topicId,
    partitions: [{
      partitionIndex: 0,
      abortedTransactions: [
        { producerId: 99n, firstOffset: 0n }
      ],
      records: [
        abortMarkerForProducer5,
        committedBatchForProducer5
      ]
    }]
  }]
}
```

여기서 의도적으로 `abortedTransactions`에는 producer 99만 넣고, `records`에는 producer 5의 abort marker를 넣었다. fetch는 `READ_COMMITTED`와 `fetchOffset: 1n`으로 실행한다.

테스트가 확인하는 사용자-visible 결과는 두 가지다.

```ts
const result = await consumer.fetch({
  isolationLevel: FetchIsolationLevels.READ_COMMITTED,
  fetchOffset: 1n,
  // ...
})

const records = result.responses[0].partitions[0].records!
strictEqual(records.length, 2)
strictEqual(records[1].records[0].value!.toString(), 'committed')
```

marker를 처리하는 과정에서 예외가 나지 않아야 하고, 뒤의 committed record는 결과에 남아야 한다. 테스트는 구현 내부의 map 상태가 아니라 fetch 결과와 크래시 여부를 검사하므로, 이후 내부 자료구조가 바뀌어도 같은 계약을 유지한다.

## 6. 수정 내용

수정은 `Map.get()` 결과를 확인하는 가드 하나로 제한했다.

```diff
if (batch.attributes & IS_CONTROL && batch.records[0].key?.readInt16BE(2) === 0) {
-  abortedRanges.get(batch.producerId)![1] = batch.firstOffset
+  const abortedRange = abortedRanges.get(batch.producerId)
+  if (abortedRange) {
+    abortedRange[1] = batch.firstOffset
+  }
}
```

매칭되는 범위가 있으면 기존과 동일하게 end offset을 기록한다. 매칭되는 범위가 없으면 marker만 건너뛴다. 이 경우 닫을 범위가 없으므로 별도의 상태를 만들거나 일반 레코드를 제거할 필요가 없다.

이 선택이 안전한 이유는 다음과 같다.

- `abortedTransactions`에 있는 transaction은 기존 로직대로 범위를 닫고 필터링한다.
- 목록에 없는 producer는 브로커가 이미 시작 지점을 지난 transaction이므로, 이 응답에서 제거할 범위를 클라이언트가 알 수 없다.
- 알 수 없는 범위를 추측해 레코드를 삭제하는 것보다 marker를 무시하는 편이 데이터 손실을 만들지 않는다.
- 공개 API, isolation level 선택, offset 관리와 consumer 상태 전이는 바뀌지 않는다.

## 7. 처음 고려한 접근과 버린 접근

### 실제 브로커 재현만 고집하지 않은 이유

실제 재현은 브로커 버전과 로그 세그먼트 배치에 강하게 의존했다. 세그먼트 롤을 강제로 만드는 설정까지 추가하면 이슈의 핵심보다 테스트 환경이 더 복잡해진다. 클라이언트가 처리해야 하는 응답 형태를 synthetic response로 고정하는 편이 빠르고 안정적이다.

### 없는 범위를 임의로 생성하지 않은 이유

marker를 보고 `[batch.firstOffset, batch.firstOffset]` 같은 기본 범위를 만들면 실제 aborted transaction의 시작 오프셋을 알 수 없는 상태에서 레코드를 잘못 버릴 수 있다. 이 버그는 크래시를 없애는 것이 목적이지, 불완전한 정보로 필터링 범위를 추측하는 것이 목적이 아니다.

### 다른 control marker 처리 지점을 함께 바꾸지 않은 이유

저장소 전체에서 `IS_CONTROL` 사용처를 확인했다. 다른 경로는 marker를 만나면 해당 배치를 건너뛰고 `abortedRanges` map을 조회하지 않는다. 동일한 결함이 확인된 지점은 `#filterUncommittedMessages` 한 곳뿐이므로, 수정 범위를 넓히지 않았다.

## 8. 검증 기록

### 로컬 검증

- consumer 회귀 테스트: synthetic fetch 응답으로 unmatched marker 입력을 검증
- 타입 검사: 통과
- 변경 파일 ESLint: 통과
- `git diff --check`: 통과
- 원본 코드로 되돌렸을 때 `Cannot set properties of undefined`가 재현되고, 가드 복구 후 사라지는 것을 확인

실제 브로커 재현은 앞서 설명한 세그먼트 조건 때문에 실패했지만, 이는 코드 테스트 실패가 아니라 재현 환경의 한계다. 이 사실과 synthetic 테스트로 전환한 이유를 분리해서 기록했다.

### GitHub CI

PR 실행에서 Apache Kafka API 호환성, Confluent 호환성, Node.js 22/24/26 조합 대부분은 통과했다. 현재 남은 실패는 다음 하나다.

```text
Node.js 22.x / Confluent Kafka 7.9.0
```

실패 로그는 테스트 assertion이 아니라 Docker Hub에서 `confluentinc/cp-kafka:7.9.0` manifest를 요청하는 중 `connection reset by peer`가 발생한 것이다. 같은 job은 직전 PR 실행과 기본 브랜치 실행에서 통과했으므로 코드 회귀로 볼 근거가 없다. 저장소 관리자 권한이 없어 실패 job 재실행은 하지 못했다.

## 9. 커밋과 PR에서 확인한 것

- 변경 파일은 `src/clients/consumer/consumer.ts`와 `test/clients/consumer/consumer.test.ts` 두 개다.
- 커밋 제목은 저장소의 기존 `fix: ...` 형식에 맞췄다.
- 커밋에는 `Signed-off-by`를 포함했고, 자동 생성된 공동 저자나 도구 attribution은 넣지 않았다.
- PR 본문은 이슈의 원인, guard의 의도와 테스트 방법만 설명한다.
- PR은 2026-09-07 머지됐다. CI의 Docker 네트워크 실패는 재실행 후 통과로 해결됨(11절 참고).

## 10. 복습용 요약

이번 버그에서 기억할 핵심은 Kafka의 `records`와 `abortedTransactions`가 서로 다른 기준으로 잘릴 수 있다는 점이다. control marker가 응답에 있다는 사실만으로 같은 producer의 aborted transaction 설명이 map에 있다고 가정할 수 없다.

코드에서 `Map.get()` 뒤에 non-null assertion을 붙였을 때는 “이 값이 없는 응답은 프로토콜상 불가능한가?”를 먼저 확인해야 한다. 이번 응답은 프로토콜상 가능했고, 가장 작은 수정은 없는 값에 대해 no-op으로 지나가는 것이었다. 실제 브로커에서 재현되지 않는 경계 조건도 응답 구조를 단위 테스트로 고정하면 안정적으로 검증할 수 있다.

## 11. 최종 상태

- 2026-09-05: `mcollina`가 승인("lgtm")
- `Node.js 22.x / Confluent Kafka 7.9.0` job은 재실행 후 통과 — Docker Hub manifest 요청 실패였던 최초 원인 그대로, 코드 회귀가 아니었음을 재실행 결과로 확인
- 2026-09-07: `ShogunPanda`가 최종 승인 및 머지 (`c92e42961a`), 커밋 제목은 원본 그대로 유지됨
- 이슈 #391은 머지로 자동 종료됨
