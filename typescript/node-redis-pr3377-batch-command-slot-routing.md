# [Redis OSS 기여] redis/node-redis - 슬롯 마이그레이션 중 배치(MULTI/pipeline) 커맨드 라우팅 수정 (PR #3377) — 🟢 Merged

> 오픈소스 기여 기록 및 학습 노트. 이슈 #3366 하나에 대한 설계·구현·자체검토(4차)·메인테이너 리뷰 과정을 시간순으로 다룬다. **2026-08-05 `master`에 머지 완료**(머지 커밋 `d4eba09`).
>
> 이 이슈는 `#3364`(extractAllCommands 순회 버그)를 고치는 과정에서 파생됐고, `#3367`(abort/timeout 리스너 재바인딩) 위에 쌓아서 작업했다. 셋 다 같은 `packages/client/lib/client/commands-queue.ts` / `packages/client/lib/cluster/cluster-slots.ts`를 건드리는 연쇄 작업이었다. 이번 문서는 그중 `#3366` 하나만 다룬다.

---

## 1. 개요

### 1.1 이슈 (#3366)

**제목**: 배치(MULTI/pipeline) 커맨드가 슬롯 정보 없이 큐잉되어, 슬롯 마이그레이션 중 엉뚱한 목적지로 이동하거나 트랜잭션이 쪼개질 수 있음

메인테이너 `nkaradzhov`가 `#3364` PR 리뷰 중 직접 확인해준 코멘트:

> "The cluster MULTI and pipeline executors drop the slot number, so batches are skipped by extractCommandsForSlots and forwarded to whichever destination was processed last. Only observable once #3364 lands, reach is limited to maintenance notifications with a batch queued during the window, and the failure is a visible error, so tracking as valid low priority. The fix should tag the whole chain with the routing key slot and only relocate fully queued chains."

핵심 요구사항 두 가지가 이 한 문장에 다 들어있다: ① 체인 전체에 슬롯을 태그할 것, ② **완전히 큐잉된 체인만** 옮길 것.

### 1.2 관련 링크

| 구분 | 링크 |
|---|---|
| 원본 저장소 | https://github.com/redis/node-redis |
| 내 Fork | https://github.com/GiHoon1123/node-redis |
| 이슈 (#3366) | https://github.com/redis/node-redis/issues/3366 |
| PR (#3377) | https://github.com/redis/node-redis/pull/3377 |
| 경쟁 PR (#3368, 다른 컨트리뷰터) | https://github.com/redis/node-redis/pull/3368 |
| 의존 PR | `#3364`(머지됨), `#3367`(머지됨) |
| 작업 브랜치 | `fix-3366-batch-slot-routing` |
| 머지 커밋 | `d4eba0928110687faa37e867d9ed2fa938fe8a5a` |
| 상태 | 🟢 Merged (2026-08-05) |

### 1.3 한 줄 요약

> `MULTI`/파이프라인 커맨드에 슬롯 번호(`slotNumber`)를 붙이고, 마이그레이션 중 남은 커맨드를 목적지별로 정확히 나눠 보내도록 고쳤다. 그 과정에서 "이미 반쯤 전송된 체인을 잘못 옮기면 트랜잭션이 두 노드에 쪼개진다"는 위험을 발견해서, 노드 전체 유실 시엔 거부(reject), 부분 마이그레이션 시엔 원래 연결에 그대로 두는 걸로(그리고 그 뒤에 큐잉된 커맨드까지 같이 묶어서) 나눠 처리했다.

---

## 2. 배경 지식

### 2.1 Redis Cluster의 슬롯

Redis Cluster는 전체 키 공간을 16384개의 **슬롯**으로 미리 나눠놓고, 각 노드가 이 슬롯 중 일부 구간을 담당한다. 키 이름을 해시 계산하면 항상 같은 슬롯 번호가 나오고, 그 슬롯을 담당하는 노드로 요청을 보내야 한다.

### 2.2 슬롯 마이그레이션

노드를 추가/제거하거나(스케일링), Redis Enterprise가 점검을 위해 노드를 재시작하려 할 때(`SMIGRATED` 이벤트) 슬롯이 다른 노드로 옮겨진다. 이 저장소에서는 `cluster-slots.ts`의 `#handleSmigrated`가 이 이벤트를 처리한다.

이때 두 가지 경로로 나뉜다:

- **부분 마이그레이션**: 소스 노드가 슬롯을 일부 남기고 있음 → 연결을 계속 씀 → `extractCommandsForSlots(destinationSlots)`로 옮겨지는 슬롯에 해당하는 커맨드만 뽑아서 목적지로 보냄
- **노드 전체 유실**: 소스 노드가 슬롯을 하나도 안 남김 → 연결을 파괴(`destroy()`)함 → `extractAllCommands()`로 남은 전부를 뽑아서 각자의 새 소유자에게 재배치함

### 2.3 클라이언트의 두 대기줄 — `#toWrite` / `#waitingForReply`

`RedisCommandsQueue`(`commands-queue.ts`)는 명령어를 두 단계로 관리한다.

- **`#toWrite`**: 아직 소켓으로 안 나간, 나갈 차례를 기다리는 커맨드들
- **`#waitingForReply`**: 이미 소켓으로 나갔고, 서버 응답만 기다리는 커맨드들

`#toWrite`에 있는 커맨드는 아직 우리 통제 하에 있어서 다른 목적지로 옮길 수 있지만, `#waitingForReply`로 넘어간 커맨드는 이미 나가버려서 되돌릴 수 없다.

### 2.4 MULTI/EXEC(트랜잭션)와 `chainId`

`MULTI`, 그 안에 담긴 커맨드들, `EXEC`는 "하나의 묶음"으로 처리돼야 한다 — 반드시 **하나의 노드 연결 안에서** 순서대로, 중간에 안 끊기고 실행돼야 한다. 이 저장소는 이 묶음을 `chainId`(체인마다 고유한 `Symbol`)로 표시한다. `MULTI`~`EXEC` 사이의 모든 `addCommand()` 호출에 같은 `chainId`가 붙는다.

Redis 서버 입장에서 `MULTI`~`EXEC`는: `MULTI`를 받으면 "접수 모드"로 들어가서 그 사이 명령어들을 실행 안 하고 접수만 해두고, `EXEC`가 오면 그제서야 한꺼번에 실행한다. 그러니까 `MULTI`와 `EXEC`가 서로 다른 노드로 가버리면, 접수 모드에 들어간 노드는 영원히 `EXEC`를 못 받고, `EXEC`를 받은 노드는 애초에 `MULTI`를 받은 적이 없어서 에러가 난다 — 트랜잭션 자체가 증발한다.

---

## 3. 경쟁 PR 확인 (#3368)

작업 시작 전에 이슈에 달린 다른 PR이 있는지 확인했다. `#3368`(다른 컨트리뷰터 `aartisonigra`)이 이미 있었는데:

- 메인테이너가 **방향은 맞다**고 인정(`slotNumber`를 `_executeMulti`/`_executePipeline`로 넘기는 접근)
- 근데 관련 없는 변경사항(Sentinel 병합, FT.SEARCH PARAMS, MOVED 핸들링, 문서 등)이 잔뜩 섞여있어서 `CHANGES_REQUESTED` 상태
- 커밋 히스토리를 보니 저자의 개인 브랜치를 정리 안 하고 그대로 PR을 올린 것으로 보임(6월 초 무관한 커밋 + 7월 27일 관련 커밋이 섞여있음)
- 결정적으로, **"완전히 큐잉된 체인만 옮겨라"** 조건은 이 PR에도 반영이 안 돼있었음
- 리뷰 요청 이후 저자가 하루 넘게 무응답

경쟁 PR이 스코프도 뒤섞여 있고 핵심 요구사항도 안 채워진 채로 멈춰있는 걸 확인한 뒤, 별도로 진행하기로 했다. PR 본문에는 `#3368`을 굳이 언급하지 않고 이슈 자체를 정확히 고치는 데 집중했다.

---

## 4. 설계와 구현 — 3단계

### 4.1 Part 1 — 슬롯 태깅

`client/index.ts`의 `_executeMulti`/`_executePipeline`에 `slotNumber?: number` 파라미터를 추가하고, `MULTI`/각 커맨드/`EXEC` 전부에 **똑같은 값**을 넘겼다.

```ts
async _executeMulti(
  commands: Array<RedisMultiQueuedCommand>,
  selectedDB?: number,
  slotNumber?: number
) {
  ...
  const chainId = Symbol('MULTI Chain');
  const promises: Array<Promise<unknown>> = [
    this._self.#queue.addCommand(['MULTI'], { chainId, slotNumber }),
  ];

  for (const { args } of commands) {
    promises.push(
      this._self.#queue.addCommand(args, { chainId, typeMapping, slotNumber })
    );
  }

  promises.push(
    this._self.#queue.addCommand(['EXEC'], { chainId, slotNumber })
  );
```

`cluster/index.ts`의 `MULTI(routing)`은 원래도 `getClientAndSlotNumber(firstKey, isReadonly)`를 호출해서 `slotNumber`를 알고 있었는데, 그동안 그냥 버리고 있었다. 이제 실제로 넘긴다.

```ts
const { client, slotNumber } = await this._self._slots.getClientAndSlotNumber(firstKey, isReadonly);
return client._executeMulti(commands, undefined, slotNumber);
```

`slotNumber`는 함수 파라미터 하나이기 때문에, 한 체인 안의 커맨드들이 서로 다른 슬롯 번호를 가질 방법이 코드 구조상 없다 — 애초에 "하나의 트랜잭션은 하나의 슬롯"이라는 Redis Cluster의 전제와 일치한다.

### 4.2 Part 2 — 목적지별 그룹핑

기존 코드는 노드 전체 유실 시 남은 커맨드를 전부 무조건 `lastDestNode`(마지막으로 처리된 목적지)로 몰아넣었다:

```ts
// 기존 (버그)
if (lastDestNode?.client) {
  lastDestNode.client._getQueue().prependCommandsToWrite(remainingCommands);
  ...
```

여러 슬롯이 서로 다른 목적지로 옮겨지는 상황이면, 이건 완전히 틀린 목적지로 커맨드를 보내는 셈이다(그 노드는 그 슬롯 담당이 아니므로 `MOVED` 에러로 되돌아옴). `groupCommandsByDestination()`을 새로 만들어서 각 커맨드를 실제 슬롯 소유자별로 나눴다:

```ts
export function groupCommandsByDestination(commands, slots, fallback) {
  const byDestination = new Map();
  const unrouted = [];

  for (const command of commands) {
    const destination = command.slotNumber === undefined ?
      fallback :
      slots[command.slotNumber]?.master ?? fallback;

    if (!destination) {
      unrouted.push(command);
      continue;
    }

    const group = byDestination.get(destination);
    if (group) group.push(command);
    else byDestination.set(destination, [command]);
  }

  return { byDestination, unrouted };
}
```

### 4.3 Part 3 — "완전히 큐잉된 체인만" (가장 까다로웠던 부분)

`MULTI`가 이미 소켓으로 나가서 `#waitingForReply`에 있고, `SET`/`EXEC`만 아직 `#toWrite`에 남아있는 상태에서 마이그레이션이 오면? 이 꼬리를 그냥 슬롯 기준으로 옮겨버리면, 트랜잭션이 두 노드에 걸쳐 쪼개진다(2.4절에서 본 그 위험).

이걸 감지하려고 큐에 `chainInExecution` getter를 추가했다 — "방금 막 소켓으로 나간 커맨드가 어느 체인 소속이었는지"를 기억한다.

```ts
get chainInExecution(): symbol | undefined {
  return this.#chainInExecution;
}
```

(`#chainInExecution`은 원래도 있던 내부 필드였다 — `commandsToWrite()` 제너레이터가 커맨드를 실제로 전송할 때마다 갱신되고, 연결 끊김 시 `flushWaitingForReply()`가 이걸로 "전송 중이던 체인의 나머지도 같이 정리"하는 데 이미 쓰고 있었다. 그걸 외부(큐 소비자)에서도 읽을 수 있게 getter로 노출한 것.)

이 신호를 노드 전체 유실과 부분 마이그레이션에서 **다르게** 썼다:

**노드 전체 유실** (`extractAllCommands` 호출부, `cluster-slots.ts`): 연결이 곧 파괴되니 "그대로 두자"가 성립 안 함 → 체인 꼬리를 발견하면 **거부(reject)**

```ts
const chainInExecution = sourceNode.client._getQueue().chainInExecution;
const inFlightChainTail = chainInExecution === undefined
  ? []
  : remainingCommands.filter(command => command.chainId === chainInExecution);
const relocatable = inFlightChainTail.length === 0
  ? remainingCommands
  : remainingCommands.filter(command => command.chainId !== chainInExecution);

if (inFlightChainTail.length > 0) {
  sourceNode.client._getQueue().rejectCommands(inFlightChainTail, new DisconnectsClientError());
}
```

**부분 마이그레이션** (`extractCommandsForSlots` 자체, `commands-queue.ts`): 연결이 계속 살아있으니 "옮기지 말고 원래 자리에 그대로 두면" 알아서 정상 전송됨 → **추출 자체를 안 함**

```ts
extractCommandsForSlots(slots: Set<number>): CommandToWrite[] {
  const result: CommandToWrite[] = [];
  let current = this.#toWrite.head;
  while (current !== undefined) {
    if (current.value.slotNumber === undefined || !slots.has(current.value.slotNumber)) {
      current = current.next;
      continue;
    }

    if (this.#chainInExecution !== undefined && current.value.chainId === this.#chainInExecution) {
      break;  // 여기서 스캔 자체를 멈춤 — 4.5절 참고
    }

    // ... 추출 로직
  }
  return result;
}
```

---

## 5. 자체검토 — 4차, 매번 진짜 버그가 나왔다

### 5.1 1차 (내 자체검토) — 큰 문제 없음, 사소한 중복만 발견

diff 전체를 다시 읽고, 각 파일별 변경사항이 서로 모순 없는지 확인. `client/index.ts`의 3번째 파라미터가 sentinel/pool/legacy-mode 등 다른 호출부에 영향 없는지도 확인(다들 옵셔널이라 안전). 발견한 건 사소한 것 하나뿐(같은 배열을 두 번 필터링하는 비효율) — 버그는 아니라서 그대로 둠.

### 5.2 2차 (유저 자체검토) — 진짜 버그 2개

1. **목적지 못 찾으면 조용히 사라짐**: `groupCommandsByDestination()`의 `if (!destination) continue;`가 그냥 커맨드를 버렸다. `lastDestNode`가 `undefined`일 수 있는 타입인데, 그 경우 슬롯 소유자도 못 찾은 커맨드는 어느 그룹에도 안 들어가서 **prepend도 reject도 안 된 채 사라졌다** — 정확히 `#3367`이 막으려던 "extracted but never settled" 버그가 재발하는 형태였다.
   - **수정**: `groupCommandsByDestination`이 `{ byDestination, unrouted }`를 반환하도록 바꾸고, 호출부에서 `unrouted`를 명시적으로 `rejectCommands()`

2. **부분 마이그레이션에서 in-flight 체인을 reject하면 위험**: 처음엔 두 경로(전체 유실/부분 마이그레이션) 다 "체인 꼬리를 reject"하는 걸로 짰었는데, 부분 마이그레이션은 **소스 연결이 안 죽는다**. reject해봤자 서버 쪽에 남은 "접수 모드" 상태는 안 지워지고, 그 연결이 계속 재사용되면서 이상하게 남은 트랜잭션 상태 위에 다음 커맨드들이 얹힐 위험이 있었다.
   - **수정**: 부분 마이그레이션 경로에서는 in-flight 체인 특수처리를 제거(원래 동작으로 복귀) — 신규 위험 아니라 안전한 방향으로 되돌린 것뿐

### 5.3 3차 (유저 자체검토) — 순서 역전 버그

부분 마이그레이션 fix(5.2-②)를 반영하고 나니, **새로운** 문제가 생겼다: `MULTI`/`EXEC` 커맨드도 이제 Part 1 덕분에 `slotNumber`를 갖게 됐는데, 이게 `extractCommandsForSlots()`의 매칭 대상이 되면서 — **체인 꼬리 하나만 건너뛰고 그 뒤에 큐잉된 무관한 커맨드는 계속 추출**하고 있었다.

유저가 직접 큐만으로 재현:
```ts
queue.addCommand(['MULTI'], { chainId, slotNumber: 1 });
queue.addCommand(['SET', 'k', 'v'], { chainId, slotNumber: 1 });
queue.addCommand(['EXEC'], { chainId, slotNumber: 1 });
queue.addCommand(['GET', 'k'], { slotNumber: 1 });  // 체인 뒤에 큐잉된 무관한 커맨드

writer.next(); // MULTI 전송
queue.extractCommandsForSlots(new Set([1]));
// => ['SET', 'EXEC', 'GET'] 다 뽑혀나옴 (버그)
```

`GET`이 새 노드로 옮겨져서 트랜잭션이 옛날 노드에서 끝나기도 전에 먼저 실행되면, 유저는 "트랜잭션 끝난 다음 GET 했으니 새 값 보이겠지"라고 기대하는데 옛날 값을 읽어버릴 수 있다 — 순서 역전.

- **수정**: 체인 꼬리를 만나면 그 지점에서 **스캔 자체를 멈춤**(`continue`가 아니라 `break`). 그 뒤에 큐잉된 건 뭐든 원래 연결에 같이 남겨서 순서를 지킨다.
- 이 수정 과정에서 **내가 또 실수를 했다** — `chainId !== chainInExecution` 조건에 undefined 가드를 안 넣어서, `chainId`도 `chainInExecution`도 둘 다 `undefined`인 일반(체인 아닌) 커맨드까지 걸러지는 회귀를 만들었다가, 전체 테스트를 돌리자마자 기존 테스트 4개가 깨지면서 바로 잡혔다.
- 테스트를 다시 짜다가, "체인보다 먼저 큐잉된 커맨드는 정상 추출된다"는 테스트를 만들려다 **논리적으로 불가능한 시나리오**라는 걸 깨달았다 — `commandsToWrite()`가 엄격히 FIFO라서, 체인이 in-flight가 된 시점엔 그보다 먼저 큐잉된 건 이미 다 전송된 뒤이기 때문. 대신 "이미 끝난 체인은 영향 없다"는 테스트로 바꿨다.

### 5.4 4차 (유저 자체검토) — 등급 낮은 지적 2개

1. **테스트 방어력 부족(P2)**: `batch-routing.spec.ts`가 `_executeMulti`/`_executePipeline`를 fake로 바꿔서 "slotNumber가 전달되는지"만 보는데, 실제 `client/index.ts`의 진짜 구현(addCommand에 slotNumber 붙이는 부분)이 지워져도 이 테스트는 안 깨진다. 확인해보니 이 실제 구현을 검증하는 테스트는 전체 코드베이스에 하나도 없었다 — `_executeMulti`/`_executePipeline`가 첫 줄에서 `#socket.isOpen` 체크로 막혀서, 소켓 없이 "연결된 것처럼" 흉내내는 기존 유닛테스트 패턴이 이 저장소에 없기 때문. Docker 기반 functional 테스트 영역으로 넘어가야 해서, 이번 PR 스코프에서는 보류(P2급, 블로커 아님).
2. **주석 표현 부정확(P3)**: "그 뒤엔 기존 retry path를 탄다"고 썼는데, `cluster/index.ts`를 다시 보니 MOVED/ASK 재시도 루프(`_execute()`)는 `sendCommand`(단일 커맨드)만 타고, `MULTI(routing)`은 이 루프를 아예 안 거친다 — 배치 커맨드는 자동 재시도가 없다. "existing retry path"라는 표현을 고쳤다.

---

## 6. 리베이스 히스토리 — 의존 PR이 머지되길 두 번 기다림

이 작업은 `#3364`(먼저 머지되어야 배치-라우팅 버그 자체가 관찰 가능해짐)와 `#3367`(같은 파일을 건드리는 별개 fix)에 순차적으로 의존했다.

1. **`#3364` 대기 후 첫 구현**: `#3364`가 머지되기 전엔 이 버그가 "관찰 불가능"(메인테이너 코멘트)이라 대기. 머지 후 Part 1/2 로컬 구현.
2. **`#3367` 위로 리베이스**: `rejectCommands()`(우리 코드가 의존하는 헬퍼)가 `#3367`에서 새로 만들어진 것이라, 그 브랜치 위에 얹어서 작업(Part 3 포함).
3. **`#3367` 머지 후 재작업**: `#3367`이 실제로 머지되자, 기존 브랜치 전체를 rebase하려니 이미 머지된 커밋들과 충돌이 났다. 히스토리 전체를 재생하는 대신, **`#3367` 위에만 있던 우리 고유 커밋 3개를 새 `upstream/master` 위에 `cherry-pick`**하는 방식으로 깔끔하게 정리했다.

---

## 7. 검증

- 자체검토 4차 전부 disable-and-reconfirm(고친 걸 되돌려서 테스트가 정확히 실패하는지 확인)으로 검증
- `groupCommandsByDestination`의 `unrouted` 처리를 되돌리면 → 테스트가 정확히 실패(커맨드가 사라짐)
- `extractCommandsForSlots`의 `break`를 `continue`로 되돌리면 → `GET`이 다시 잘못 추출되며 테스트 실패
- 최종 실행 결과:

```text
npm run test-single -- packages/client/lib/cluster/batch-routing.spec.ts packages/client/lib/cluster/cluster-slots.spec.ts packages/client/lib/client/commands-queue.spec.ts
24 passing

npx eslint ... --max-warnings=0
0 errors

npm run test:types --workspace @redis/client
클린
```

---

## 8. PR 본문

```md
Fixes #3366.

Rebased after #3367 landed.

Cluster MULTI/pipeline execution was already selecting the right slot, but the queued commands
did not keep that `slotNumber`. During maintenance slot migration, the full source-node
cleanup path could then fall back to the last processed destination for remaining queued batch
commands.

This tags the whole MULTI/pipeline chain with the selected slot and uses that slot when moving
queued commands during source cleanup. Commands without a route are rejected instead of being
dropped.

An already in-flight chain (its head already written to the socket, tail still queued) is
handled differently depending on whether the connection survives:
- During partial slot migration, the connection keeps serving its other slots, so the tail is
  left queued there instead of being split onto another connection.
- During full node loss, the connection is destroyed anyway, so the tail is rejected instead of
  risking a split transaction.

Tested with:
...

24 tests passing, 0 lint errors, type check clean.
```

---

## 9. 메인테이너 리뷰 — CHANGES_REQUESTED 1차 후 머지

### 9.1 `nkaradzhov`의 리뷰

PR 본문을 올린 뒤 메인테이너 `nkaradzhov`가 `CHANGES_REQUESTED`로 리뷰를 남겼다. 두 가지를 요청했다:

> - Please add a test for the case where a second chain is fully queued but not yet written behind an already-in-flight chain, asserting the second chain relocates atomically and isn't mistaken for the in-flight tail.
> - Please add an assertion that on full node loss, the already-sent head of an in-flight chain (in waitingForReply) is also rejected, so the whole transaction fails consistently rather than half-committing.

첫 번째는 정확히 5.3절에서 다룬 "체인 꼬리 뒤에 큐잉된 무관한 커맨드" 시나리오를 한 단계 더 밀어붙인 것 — "그 뒤에 있는 게 무관한 단일 커맨드"가 아니라 "완전히 큐잉된 또 다른 체인"이어도 통째로 옮겨지는지(꼬리로 오인해서 반쪽만 옮겨지지 않는지)를 물었다. 두 번째는 4.3절의 노드 전체 유실 경로에서, 이미 `#waitingForReply`로 나가버린 체인의 머리(head) 부분도 꼬리와 함께 일관되게 reject되는지를 물었다 — 머리는 살아남고 꼬리만 reject되면, 서버는 `MULTI`를 받았는데 `EXEC`를 영원히 못 받는 반쪽짜리 트랜잭션이 남는다.

두 요청 다 이미 구현된 로직이 처리하고 있었지만(4.3절 코드가 `chainId` 전체를 단위로 다루므로), **그걸 명시적으로 검증하는 테스트가 없었다**는 게 정확한 지적이었다.

같은 리뷰 스레드에 자동 리뷰 봇(`chatgpt-codex-connector`, "Cursor Bugbot"과는 별개)도 코멘트를 남겼는데, 이번엔 로직 자체보다 스타일/범위에 관한 제안들이라 이번 PR 스코프 밖으로 판단했다.

### 9.2 반영

두 케이스를 테스트로 추가하고 리뷰에 답했다:

> Thanks for the review. I pushed an update.
>
> This now covers the extractCommandsForSlots case (a second, fully-queued chain behind an in-flight one on a different slot relocates as a whole instead of being mistaken for its tail), the full-node-loss case with an in-flight chain followed by a fully queued chain, and also checks that the already-sent head in `waitingForReply` gets rejected together with the queued tail.
>
> I also looked at the automated review comments. They look like broader follow-ups to me, so I left them out for now unless you'd rather include them in this PR.

자동 리뷰 봇 코멘트를 이번 PR에 포함시킬지 말지를 메인테이너 판단에 맡긴다고 명시적으로 밝힌 점 - 봇 제안을 조용히 무시하지도, 무비판적으로 다 반영하지도 않고 스코프 판단 근거를 남겼다.

### 9.3 두 번째 리베이스와 머지

업데이트를 푸시한 뒤, `master`에 있던 다른 변경과 충돌이 나서 다시 리베이스했다:

> Rebased on master, conflicts resolved.

이후 메인테이너가 승인했다:

> @GiHoon1123 thanks, this is looking good. Please resolve the merge conflicts and we should be able to merge it

충돌 해결 후 최종 승인(`APPROVED`)이 한 번 더 있었고, CI(TypeScript 분석/JSDoc/lint/여러 Redis 버전 조합 테스트/CodeQL/Cursor Bugbot 포함 15개 체크) 전부 통과한 뒤 2026-08-05 `master`에 머지됐다(`d4eba09`). 최종 diff는 7개 파일, +543/-43줄.

---

## 10. 배운 점

1. **"완전히 큐잉된 체인만 옮겨라"는 한 문장이 실제로는 두 개의 서로 다른(심지어 반대되는) 정책을 요구했다.** 연결이 죽는 경로(노드 전체 유실)와 살아있는 경로(부분 마이그레이션)에서 "안전한 선택"이 정반대(거부 vs 그대로 두기)였다 — 요구사항 문장 하나를 코드 경로별로 정확히 나눠서 적용해야 했다.
2. **fix 하나가 다른 곳의 전제를 깨뜨릴 수 있다.** Part 1(슬롯 태깅)이 배치 커맨드에 `slotNumber`를 붙이자, 그동안 "슬롯 없어서 애초에 안 걸리던" `extractCommandsForSlots`의 매칭 대상이 되면서 순서 역전이라는 완전히 새로운 문제가 생겼다 — 한 부분을 고치면 그 변경이 다른 코드의 암묵적 전제를 건드리지 않는지 다시 확인해야 한다.
3. **직접 재현이 논쟁을 끝낸다.** "체인 꼬리 뒤의 무관한 커맨드도 같이 옮겨진다"는 지적을, 큐 하나만 가지고 직접 코드로 재현해서 확인한 게 결정적이었다 — 말로 설명하는 것보다 몇 줄짜리 재현 스크립트가 훨씬 빠르고 확실했다.
4. **자체검토를 반복할수록 등급이 낮아졌다** — 1차는 "새로 발견 없음", 2·3차는 P1급 실제 버그, 4차는 P2/P3급 사소한 지적. 이 패턴 자체가 "핵심 로직은 안정화됐고 남은 건 다듬기 수준"이라는 신호로 읽을 수 있었다.
5. **경쟁 PR을 무시하지 말고, 왜 진행해도 되는지 근거를 남겨야 한다.** `#3368`이 있었지만 방향은 맞아도 스코프가 뒤섞이고 핵심 요구사항이 빠진 채 멈춰있다는 걸 직접 확인한 뒤에 진행했다 — 단순히 "저쪽은 무시하고 내가 낸다"가 아니라 근거를 갖춘 판단이었다.
6. **메인테이너 리뷰는 "로직이 틀렸다"보다 "이미 맞게 짠 걸 증명하는 테스트가 없다"는 지적으로도 온다.** 실제로 요청받은 두 가지(체인 뒤에 또 다른 완전 체인이 와도 안 섞이는지, 노드 전체 유실 시 머리/꼬리가 같이 reject되는지) 다 4.3절 코드가 이미 처리하고 있었지만, 그걸 명시적으로 보여주는 테스트가 없었다 — "동작은 맞는데 증거가 없다"도 리뷰 대상이라는 걸 확인했다.

### 10.1 결과

- [x] 메인테이너 리뷰 — `nkaradzhov`가 CHANGES_REQUESTED 1차(9장), 테스트 보강 후 승인
- [x] CI 결과 — TypeScript 분석/JSDoc/lint/Redis 5개 버전 조합 테스트/CodeQL/Cursor Bugbot 등 15개 체크 전부 통과
- [x] 머지 여부 — 2026-08-05 `master`에 머지(`d4eba09`)
- [x] `#3368`과의 관계 — 이 PR 리뷰가 진행되기 전인 2026-07-29에 이미 별도로 closed(머지 안 됨), 메인테이너가 별도로 정리한 것으로 보임
