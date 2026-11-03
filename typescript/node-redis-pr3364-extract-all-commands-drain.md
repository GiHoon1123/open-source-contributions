# [Redis OSS 기여] redis/node-redis - 클러스터 슬롯 재배치 중 커맨드 큐가 첫 항목만 옮겨지는 버그 수정 (PR #3364) — 🟢 Merged

> 오픈소스 기여 기록 및 학습 노트. 이 문서는 PR #3364 하나만 다룬다. **2026-07-28 08:10 머지됨. 9절 참고.**

---

## 1. 개요

### 1.1 프로젝트 소개

- **저장소**: https://github.com/redis/node-redis
- **npm 패키지명**: `redis` — GitHub org verified, Redis Inc.가 직접 운영하는 공식 Node.js 클라이언트
- **건드린 파일**: `packages/client/lib/client/commands-queue.ts`(수정), `packages/client/lib/client/commands-queue.spec.ts`(신규)
- **탐색 방식**: 이번엔 서브에이전트 없이 `gh search issues` + 직접 코드 확인만으로 순차 탐색(토큰 절약 요청에 따름). NestJS·Node.js core 쪽은 넓게 훑었지만 전부 경쟁 PR이 있거나 이미 upstream에서 조용히 고쳐져 있어서 제외했고, 이 이슈만 경쟁자 없이 남아있었다.

### 1.2 이슈 (#3363)

**제목**: `extractAllCommands() returns only the first queued command`

리포터(`morgan-coded`)가 이미 원인 위치를 정확히 짚어서 올린 이슈:

> `RedisCommandsQueue.extractAllCommands()` reads `current.next` after `remove(current)` has already cleared it, so the loop exits after one iteration.

5개 커맨드가 든 큐로 직접 재현한 결과까지 첨부돼 있었다: `["CMD0"]`만 반환되고 나머지 4개는 큐에 그대로 남는다는 것.

**선점 위험 판단**: 같은 리포터가 `redis/ioredis`에도 같은 날 비슷한 정밀도의 이슈(#2149)를 올렸고, `gajus/eslint-plugin-jsdoc`·`meriyah` 등 여러 저장소에서 이슈 등록 후 본인이 곧장 PR을 내는 이력이 확인됐다. 착수 직전 재확인 결과 경쟁 PR도, 본인 PR도 없었고, 버그 자체가 작고 원인이 이미 코드로 특정돼 있어 "설령 먼저 채여도 투자 시간이 크지 않다"는 판단으로 진행했다.

### 1.3 관련 링크

| 구분 | 링크 |
|---|---|
| 원본 저장소 | https://github.com/redis/node-redis |
| 내 Fork | https://github.com/GiHoon1123/node-redis |
| 이슈 (#3363) | https://github.com/redis/node-redis/issues/3363 |
| PR (#3364) | https://github.com/redis/node-redis/pull/3364 |
| 작업 브랜치 | `fix-3363-extract-all-commands` |
| 커밋 | `cf393bdaf0` |
| 상태 | 🟢 MERGED (2026-07-28) |

### 1.4 한 줄 요약

> 클러스터 노드가 슬롯을 전부 잃고 정리될 때, 그 노드에 대기 중이던 커맨드를 전부 다른 노드로 옮기는 `extractAllCommands()`가 연결 리스트에서 노드를 지운 다음에 `.next`를 읽어서, 지워지면서 같이 사라진 포인터를 읽어 첫 번째 커맨드만 옮기고 멈춰버렸다. 형제 메서드(`extractCommandsForSlots`)와 같은 순서(지우기 전에 next 먼저 확보)로 고쳤다.

---

## 2. 배경 지식

### 2.1 커맨드 큐는 클라이언트마다 따로 있다

`redis/node-redis`는 클러스터 모드에서 클러스터 노드 하나당 별도의 `RedisClient` 인스턴스(= 별도의 TCP 연결)를 갖는다. 그리고 이 커맨드 큐(`RedisCommandsQueue`)는 그 클라이언트 인스턴스 하나에 딸린 필드다:

```ts
readonly #toWrite = new DoublyLinkedList<CommandToWrite>();
readonly #waitingForReply = new EmptyAwareSinglyLinkedList<CommandWaitingForReply>();
```

즉 클러스터 노드마다 자기만의 `#toWrite`/`#waitingForReply` 큐 쌍이 있고, 서로 완전히 독립적이다. 큐가 노드마다 따로 있기 때문에, 노드 사이에 뭔가를 옮기려면 진짜로 한쪽 큐에서 빼서 다른 쪽 큐에 넣어주는 코드가 필요하다 — 이 이동 로직이 이번 버그가 생긴 지점이다.

### 2.2 두 단계 큐: `#toWrite` → `#waitingForReply`

유저가 `client.get('key')`를 호출하면:

1. 키를 해시해서 슬롯을 계산하고, 그 슬롯을 담당하는 클러스터 노드의 클라이언트를 찾음
2. 그 클라이언트의 `#toWrite`(양방향 연결 리스트)에 커맨드가 큐 항목으로 들어감 — "아직 소켓에 안 보낸 상태"
3. `commandsToWrite()` 제너레이터가 큐 항목을 하나씩 꺼내(shift) 소켓에 씀과 동시에 `#waitingForReply`(단방향 연결 리스트)로 옮김 — "이미 보냈고 응답 기다리는 중"
4. 서버 응답이 오면 `#waitingForReply`에서 가장 오래된 것부터 꺼내 그 커맨드의 Promise를 `resolve()`

`#toWrite`가 양방향인 이유는 타임아웃/취소로 **중간에서 임의의 큐 항목을 O(1)에 제거**해야 하기 때문이고, `#waitingForReply`는 항상 보낸 순서대로 응답이 오므로 앞에서만 빼면 돼서 단방향으로 충분하다.

### 2.3 클러스터 노드가 슬롯을 잃는 경우

Redis 클러스터는 운영 중에 노드를 추가/제거하면서 16384개 슬롯을 재분배한다(리샤딩). 어떤 클러스터 노드가 자기 슬롯을 전부 다른 노드로 넘기고 나면, 그 노드는 곧 정리(teardown) 대상이 된다. 이건 장애가 아니라 정상적인 운영 작업 — 데이터도, 클러스터 자체도 멀쩡하고, 그냥 어느 노드가 담당하는지만 바뀐 것이다.

문제는 그 노드의 `#toWrite` 큐에 **아직 안 보낸 커맨드들**(유저가 `await`하고 있는 Promise들)이 남아있을 수 있다는 것. 그 노드가 정리되면서 큐에 남은 커맨드를 그냥 버리면, 유저 입장에서는 아무 잘못도 없는데(운영팀이 리샤딩한 것뿐인데) 자기 요청이 조용히 사라지는 것과 같다. 그래서 node-redis는 "노드가 슬롯을 잃으면, 거기 남은 커맨드들을 살아있는 다른 노드로 옮겨서 계속 처리되게 한다"는 방식을 쓴다.

`cluster-slots.ts`:

```ts
} else {
  // Source has no slots left - move remaining slotless commands and cleanup
  const remainingCommands = sourceNode.client._getQueue().extractAllCommands();
  if (remainingCommands.length > 0 && lastDestNode) {
    lastDestNode.client?._getQueue().prependCommandsToWrite(remainingCommands);
    ...
```

`sourceNode.client`와 `lastDestNode.client`는 서로 다른 클라이언트 인스턴스이고, `_getQueue()`는 각자 자기 큐를 돌려준다 — 이건 진짜로 한 노드의 큐에서 다른 노드의 큐로 커맨드를 이사시키는 코드다.

---

## 3. 원인 분석

### 3.1 정상 동작 — 형제 메서드는 문제없다

`extractCommandsForSlots(slots)`는 특정 슬롯들만 재배치될 때 그 슬롯을 타겟으로 하는 커맨드만 뽑아내는 메서드다:

```ts
extractCommandsForSlots(slots: Set<number>): CommandToWrite[] {
  const result: CommandToWrite[] = [];
  let current = this.#toWrite.head;
  while (current !== undefined) {
    if (current.value.slotNumber !== undefined && slots.has(current.value.slotNumber)) {
      result.push(current.value);
      const toRemove = current;
      current = current.next;        // ← 먼저 다음 큐 항목으로 이동
      this.#toWrite.remove(toRemove); // ← 그 다음에 지움
    } else {
      current = current.next;
    }
  }
  return result;
}
```

### 3.2 버그 — `extractAllCommands()`는 순서가 반대였다

```ts
extractAllCommands(): CommandToWrite[] {
  const result: CommandToWrite[] = [];
  let current = this.#toWrite.head;
  while (current) {
    result.push(current.value);
    this.#toWrite.remove(current);   // ① 먼저 지움
    current = current.next;          // ② 그 다음에 next를 읽음
  }
  return result;
}
```

`DoublyLinkedList.remove()`의 실제 구현(`linked-list.ts`):

```ts
remove(node: DoublyLinkedNode<T>) {
  if (this.#length === 0) return;
  --this.#length;

  if (node.previous) {
    node.previous.next = node.next;
  } else {
    this.#head = node.next;
  }

  if (node.next) {
    node.next.previous = node.previous;
  } else {
    this.#tail = node.previous;
  }

  node.previous = undefined;
  node.next = undefined;   // ← 지워진 노드 자신의 next도 지움
}
```

`remove()`는 리스트에서 노드를 완전히 떼어내면서, 그 노드 자신의 `.next`/`.previous`도 `undefined`로 지운다(고아 노드가 리스트의 다른 곳을 계속 가리키지 않게 하려는 정상적인 설계). 근데 `extractAllCommands()`는 ①에서 지운 다음에 ②에서 그 노드의 `.next`를 읽으니, 이미 `undefined`가 된 값을 읽게 된다. 그래서 첫 번째 큐 항목을 처리하고 나면 루프가 바로 끝나버린다.

### 3.3 실패 체인

```
클러스터 노드 A가 슬롯을 전부 잃음 → 정리 예정
        │
        │  A의 #toWrite 큐에 커맨드 5개 남아있음
        ▼
extractAllCommands() 호출
  큐 항목1: push → remove(지우면서 next도 사라짐) → next 읽기 시도 → undefined
  → 루프 종료

  큐 항목2~5: A의 큐에 그대로 남음
        │
        │  A는 곧 정리(teardown)됨
        ▼
  큐 항목2~5의 커맨드들은 응답도 에러도 못 받고
  await하던 유저 코드가 영원히 멈춤
```

증상 자체는 예외가 던져지는 게 아니라 **조용히 멈추는 것**이라 더 발견하기 어렵다 — 함수는 정상적으로 리턴하고, 다만 큐 항목 4개를 놓친 채로 리턴한다.

---

## 4. 수정 내용

형제 메서드와 같은 순서로 바꿨다 — 지우기 전에 먼저 next를 확보한다.

```diff
   extractAllCommands(): CommandToWrite[] {
     const result: CommandToWrite[] = [];
     let current = this.#toWrite.head;
     while (current) {
       result.push(current.value);
-      this.#toWrite.remove(current);
+      const toRemove = current;
       current = current.next;
+      this.#toWrite.remove(toRemove);
     }
     return result;
   }
```

부수적으로 메서드 위의 JSDoc도 고쳤다. 원래 주석이 `"Gets all commands from the write queue without removing them."`이었는데, 이 메서드는 실제로는 큐에서 커맨드를 **제거**하면서 반환한다 — 주석이 코드 자체와 반대로 돼 있었다. `"Extracts all commands from the write queue."`로 정정했다.

---

## 5. 테스트

### 5.1 새 유닛 테스트 (`commands-queue.spec.ts`, 신규 파일)

이 파일 자체가 이 저장소에 원래 없었다 — `extractAllCommands`/`extractCommandsForSlots` 둘 다 기존에 전용 테스트가 없는 상태였다.

```ts
import assert from 'node:assert';
import RedisCommandsQueue from './commands-queue';

describe('RedisCommandsQueue', () => {
  function createQueue() {
    return new RedisCommandsQueue(3, null, () => {}, 'test-client');
  }

  describe('extractAllCommands', () => {
    it('returns and removes every queued command, not just the first one', () => {
      const queue = createQueue();
      for (let i = 0; i < 5; i++) {
        queue.addCommand([`CMD${i}`]).catch(() => {});
      }

      const extracted = queue.extractAllCommands();

      assert.strictEqual(extracted.length, 5);
      assert.deepStrictEqual(
        extracted.map(command => command.args?.[0]),
        ['CMD0', 'CMD1', 'CMD2', 'CMD3', 'CMD4'],
      );
      assert.strictEqual(queue.extractAllCommands().length, 0);
    });
  });
});
```

Docker가 필요한 통합 테스트가 아니라, `RedisCommandsQueue`를 직접 인스턴스화하는 순수 유닛 테스트다 — `addCommand()`로 5개를 큐에 넣고, `extractAllCommands()`가 5개 다 순서대로 반환하는지, 그리고 큐가 완전히 비었는지(두 번째 호출이 빈 배열을 반환하는지)를 확인한다.

### 5.2 disable-and-reconfirm

fix를 `git stash`로 잠깐 빼고 같은 테스트를 돌리자:

```
AssertionError [ERR_ASSERTION]: Expected values to be strictly equal:
1 !== 5
```

정확히 예상한 대로 실패(5개 중 1개만 반환). `git stash pop`으로 복구 후 재실행하니 통과.

### 5.3 회귀 확인

- `npx eslint` — 수정한 파일 + 새 테스트 파일 둘 다 클린
- `tsc --build` — 타입 에러 없음
- `cluster-slots.spec.ts`(Docker 불필요한 순수 유닛 테스트) 재실행 — 4개 전부 통과, 회귀 없음
- 전체 통합 스위트는 Docker가 필요해서(시간·토큰 절약 목적) 이번엔 실행하지 않음 — 이 점은 자체검토에서도, PR에도 명시하지 않았으니 필요하면 언급할 수 있음

---

## 6. PR 제목/본문 다듬기

본문 초안은 직접 작성했고(원인·수정·테스트 각 한두 문장씩, AI 냄새 없이), AI 도구로 검토는 이것만 받았다:

- **scope 선택**: `fix(client): ...` 후보가 나왔는데, `fix(cluster): ...`로 바꾸자고 제안했다. 이유는 이 버그의 실제 영향 범위가 `cluster-slots.ts`의 슬롯 재배치 경로 하나뿐이고, 이 저장소의 최근 머지 이력(`fix(cluster): reject commands before cluster topology is ready`, `fix(sentinel): reject connect() instead of hanging...`)이 파일 위치가 아니라 실제 영향 영역으로 scope를 잡는 관례였기 때문. 이미 push해둔 커밋 메시지도 `fix(cluster): ...`였어서 제목과 일치시켰다.

최종 제목: `fix(cluster): make extractAllCommands drain the write queue`

---

## 6.5 PR 올린 후 — Codex 리뷰가 찾아낸 두 가지 pre-existing 문제

PR을 올리자마자(1분 내) `chatgpt-codex-connector` 봇이 커밋 `cf393bdaf0`을 리뷰하면서 P2 등급으로 인라인 코멘트 2개를 남겼다. 스타일 지적이 아니라 실제 동작에 관한 지적이라 둘 다 코드로 직접 검증했다.

### 발견 1 — abort/timeout 리스너가 옮겨간 큐를 못 따라감

`addCommand()`의 abort/timeout 리스너:

```ts
value.abort = {
  signal,
  listener: () => {
    this.#toWrite.remove(node);   // 원래(출발) 큐, 원래 큐 항목을 클로저로 붙잡음
    value.reject(new AbortError());
  },
};
```

`extractAllCommands()`가 커맨드의 `value`만 뽑아서 목적지 큐로 옮기면(`prependCommandsToWrite` → `unshift`로 새 큐 항목 생성), 이 리스너는 여전히 원래(곧 정리될) 큐와 이미 제거된 원래 큐 항목을 가리키는 채로 남는다. 이 상태에서 abort/timeout이 이동 후에 발동하면:

- `value.reject()`는 실행됨 → 유저의 Promise는 정상적으로 취소된 것처럼 보임
- `this.#toWrite.remove(node)`는 이미 사라진 출발 큐의 이미 제거된 항목을 지우는 거라 no-op
- **목적지 큐엔 이 커맨드가 여전히 살아있어서, 결국 서버로 전송/실행될 수 있음** — 유저는 취소됐다고 알고 있는데 실제로는 실행된 상태

### 발견 2 — MULTI/파이프라인 배치가 슬롯 정보 없이 옮겨짐

`_executeMulti`/`_executePipeline`을 직접 확인 — 정말로 `slotNumber` 없이 큐잉한다:

```ts
this._self.#queue.addCommand(args, { chainId, typeMapping })  // slotNumber 없음
```

부분 슬롯 재배치(`extractCommandsForSlots`)는 `slotNumber`로 필터링하니 이런 배치 커맨드는 원래도 안 걸렸지만, 노드가 살아있는 한 문제가 안 됐다. 근데 노드가 슬롯을 전부 잃으면 `extractAllCommands()`가 슬롯 여부 상관없이 전부 긁어서 `lastDestNode`(슬롯 소유권과 무관하게 "마지막으로 처리된 목적지")로 보내버린다 — 배치가 엉뚱한 노드로 가서 `MOVED` 에러나 트랜잭션 아보트로 이어질 수 있다.

### 왜 "새 버그"가 아니라 "기존 버그가 더 드러난 것"인가

`git log -S`로 확인한 결과, 두 문제의 근본 코드(abort/timeout 리스너 패턴, `_executeMulti`/`_executePipeline`의 addCommand 호출) 전부 `V5 bringing RESP3, Sentinel and TypeMapping to node-redis`(`b2d35c5286`)라는 오래된 V5 리라이트 커밋까지 거슬러 올라간다 — 이번 PR이 건드린 적 없는 코드다.

원래 버그(`extractAllCommands`가 첫 번째만 옮기고 멈춤) 때문에 이 gap에 걸릴 수 있는 커맨드가 "운 나쁘게 첫 번째였던 것 하나"로 제한돼 있었는데, 전부 다 옮기게 고치면서 이 gap에 노출되는 빈도가 늘어난 것뿐이다.

### 대응: 스코프에 안 넣고 분리

두 문제를 이 PR에 같이 넣지 않기로 했다. 이유:
- 지금 fix는 "지우는 순서 바꾸기" 3줄인데, 두 문제를 제대로 고치려면 각각 리스너 재바인딩 설계 변경, 배치 커맨드에 슬롯 정보 전달하는 구조 변경이 필요 — 스코프가 완전히 다름
- 우리가 새로 깨뜨린 게 아니라 기존 리스크가 더 자주 보이게 된 것이라 긴급도가 다름
- 작고 검증 끝난 PR을 유지하는 게 리뷰·머지에 유리함

각 리뷰 스레드에 개별 답글로 대응하고("real, but pre-existing", V5까지 거슬러 올라간다는 근거 포함), 별도 이슈 2개로 분리했다:
- https://github.com/redis/node-redis/issues/3365 (abort/timeout 리스너 재바인딩)
- https://github.com/redis/node-redis/issues/3366 (배치 커맨드 슬롯 라우팅)

답글 초안은 처음에 "Confirmed — both real findings... Traced them via git blame..." 식으로 썼는데, 자체 검토("AI 냄새 난다")를 거쳐 "Real, but pre-existing — goes back to the V5 rewrite..."처럼 더 짧고 담백하게 다듬었다. 두 번째 답글이 "Same story"로 시작해서 첫 번째 답글을 읽었다는 걸 전제하는 문제도 지적받아서, 각 답글이 단독으로도 이해되게 고쳤다(GitHub 리뷰 스레드는 읽는 순서가 보장 안 됨).

### #3366 현재 상태 — 로컬 구현은 했지만 PR은 #3364 머지 후

#3366은 #3365와 다르게 바로 PR로 올리지 않기로 했다. 이유는 **#3366의 full-node drain 경로가 #3364의 `extractAllCommands()` 정상 동작을 전제로 하기 때문**이다.

#3365는 `prependCommandsToWrite()` 자체의 독립 invariant 문제였다. 커맨드가 어디서 왔든 목적지 큐에 다시 들어가는 순간 abort/timeout 리스너는 새 큐 항목을 바라봐야 한다. 그래서 `upstream/master` 기준 독립 브랜치로 분리해서 PR #3367을 올릴 수 있었다.

반면 #3366은 다음 흐름이 핵심이다:

```ts
const remainingCommands = sourceNode.client._getQueue().extractAllCommands();
```

노드가 슬롯을 전부 잃었을 때 남은 커맨드를 전부 drain한 뒤, 각 커맨드의 `slotNumber`를 보고 현재 슬롯 owner로 다시 보내야 한다. 그런데 master 기준 `extractAllCommands()`는 바로 이 PR(#3364)이 고치기 전까지 첫 번째 커맨드만 반환한다. 즉 #3366을 master 위에 바로 올리면 설명과 테스트가 #3364 버그에 계속 걸린다.

로컬에서는 `fix-3363-extract-all-commands` 위에서 새 브랜치 `fix-3366-batch-slot-routing`을 만들어 검토했다. 현재 구현 방향:

1. `cluster.MULTI()`가 이미 계산한 `{ client, slotNumber }`에서 `slotNumber`를 버리지 않고 `_executeMulti()` / `_executePipeline()`으로 넘긴다.
2. `_executeMulti()`는 `MULTI`, 내부 커맨드들, `EXEC` 전부 같은 `slotNumber`로 큐잉한다.
3. `_executePipeline()`도 배치 안의 커맨드들을 같은 `slotNumber`로 큐잉한다.
4. full-node drain에서 `extractAllCommands()` 결과를 `lastDestNode` 하나로 몰지 않고, `command.slotNumber` 기준 현재 `this.slots[slotNumber].master`로 목적지별 그룹핑한다.
5. `slotNumber`가 없거나 owner를 못 찾는 커맨드는 기존 동작과 비슷하게 `lastDestNode` fallback으로 보낸다.

검증도 로컬에서 끝냈다:

- `npm run test-single -- packages/client/lib/cluster/batch-routing.spec.ts packages/client/lib/cluster/cluster-slots.spec.ts packages/client/lib/client/commands-queue.spec.ts` — 9 passing
- `npm run build` — 통과
- `npm run lint:changed` — 통과
- `git diff --check` — 통과

다만 PR은 아직 올리지 않는다. 처리 순서:

1. #3364 머지 대기
2. `upstream/master` 갱신
3. `fix-3366-batch-slot-routing` rebase
4. #3366 단독 PR 제출

이렇게 해야 메인테이너가 #3364의 작은 linked-list fix와 #3366의 batch slot routing fix를 분리해서 볼 수 있다. stacked PR도 가능은 하지만, 이 저장소에서 stacked PR 리뷰가 익숙한지 확신이 없고 #3366은 설명 자체도 더 복잡하므로 머지 후 단독 PR이 더 깔끔하다.

---

## 7. 개념 설명 Q&A

이번 PR은 구현 자체보다 **연결 리스트/큐 구조를 이해하는 과정에서 나온 질문들**이 특히 길었다. 기록해둘 만한 순서:

1. "연결 리스트로 뭘 하고 뭘 저장하냐, 유저 명령어가 거기로 가냐" → `CommandToWrite` 구조체(args, resolve/reject, slotNumber 등) 설명
2. "`#toWrite`/`#waitingForReply` 둘 다 연결 리스트냐" → 맞다, 근데 종류가 다르다(양방향 vs 단방향)는 것과 그 이유(중간 삭제 필요 여부)
3. **용어 혼동 지적**: "연결 리스트 노드"와 "클러스터 노드"를 계속 섞어 썼다는 피드백을 받고, 이후 "클러스터 노드"/"큐 항목"으로 용어를 고정
4. "노드마다 연결 리스트가 따로 있냐, 아니면 하나를 여러 노드에 나눠주냐" → 노드(클라이언트 인스턴스)마다 완전히 독립된 큐 쌍이 있고, 그래서 "옮긴다"는 동작 자체가 필요하다는 것
5. "그냥 지우면 안 되냐, 언제는 지우고 언제는 옮기냐" → 리샤딩은 데이터 손실이 없는 정상 운영이라 목적지가 살아있으므로 옮겨야 하고, 진짜 실패(타임아웃/취소)는 목적지 자체가 없어서 지우는 게 맞다는 구분

## 8. 배운 점

### 8.1 "지운다"의 순서가 뒤바뀌면 조용히 실패한다

이번 버그는 예외를 던지지 않는다. 함수는 정상적으로 리턴하고, 다만 의도한 것보다 적게 처리하고 리턴한다. 연결 리스트에서 순회 중 삭제를 할 땐 "삭제가 순회 포인터에 미치는 영향"을 항상 의심해야 한다 — 이번엔 같은 파일 안에 올바른 패턴(`extractCommandsForSlots`)이 바로 위에 있어서 비교가 쉬웠다.

### 8.2 선점 위험이 있어도, 작고 원인이 명확하면 시도할 가치가 있다

리포터가 여러 저장소에 정밀한 이슈를 올리고 스스로 PR을 내는 패턴이 확인됐지만, 버그 자체가 작고(한 줄 순서 변경) 원인이 이미 코드로 특정돼 있어서 투자 대비 리스크가 낮다고 판단하고 진행했다. 착수 직전에 경쟁 PR 여부를 한 번 더 재확인하는 절차도 그대로 지켰다.

### 8.3 서브에이전트 없이 순차 탐색도 유효하다

이번 탐색은 병렬 서브에이전트 대신 직접 `gh search issues` + 경쟁 PR 확인 + 코드 읽기만으로 진행했다(토큰 절약 요청). 깊이는 서브에이전트 방식보다 얕았지만(재현 스크립트 실행 등은 생략), 후보를 걸러내는 핵심 절차(경쟁 PR 확인 → 최신 master 기준 재현 확인)는 그대로 지켜서 실제로 두 개(undici SharedArrayBuffer, mikro-orm falsy-fk)나 "이미 고쳐진 이슈"를 걸러냈다.

### 8.4 자동 리뷰 봇의 지적을 "스타일 지적"으로 넘겨짚지 않는다

CodeRabbit(sequelize PR)에 이어 이번엔 Codex가 P2 등급으로 진짜 동작 문제를 지적했다. 둘 다 처음엔 "AI 리뷰봇 코멘트"라고 가볍게 볼 수도 있었지만, 실제로 코드 추적(`git log -S`로 언제부터 있던 코드인지까지)을 해보니 둘 다 진짜였다. 봇 리뷰라고 무시하지도, 그렇다고 스코프 밖 지적까지 전부 이번 PR에 욱여넣지도 않고 — 검증 → 근거로 스코프 밖 판단 → 별도 이슈 분리, 이 순서를 지켰다.

### 8.5 "우리가 만든 버그"와 "우리가 드러낸 버그"는 다르다

이번 fix는 실제로 새 버그를 만든 게 아니라, 원래도 있던 버그(첫 번째 커맨드만 옮기던 것) 때문에 가려져 있던 다른 두 개의 pre-existing 문제를 더 잘 보이게 만들었을 뿐이다. `git log -S`로 "이 코드가 언제부터 있었나"를 확인하는 게 이 구분("새로 깨뜨림" vs "원래 있던 게 더 드러남")을 명확히 하는 가장 직접적인 방법이었다.

## 9. 다음에 할 일

- [x] 메인테이너 리뷰 유무 및 반응 — 2026-07-27, `nkaradzhov`가 승인. 코멘트: "Confirmed against master: remove() clears the node links before next is read, so only the head command was ever relocated during a full-shard handoff." 원인 진단을 메인테이너가 master에서 직접 재확인한 것.
- [x] CI 결과 — lint/CodeQL/TypeScript 분석/JSDoc/Cursor Bugbot 등 빠른 체크와 Node 20/22 × Redis 7.4/8.2/8.4/8.8/8.10-rc2 전체 테스트 매트릭스까지 모두 `pass`
- [x] 머지 여부와 소요 시간 — 2026-07-28 08:10 머지됨
- [x] #3365 (abort/timeout 리스너 재바인딩) — PR #3367로 별도 제출, 머지됨. 자세한 내용은 `node-redis-pr3367-rebind-cancellation-listeners.md` 참고
- [x] #3366 (배치 커맨드 슬롯 라우팅) — #3364, #3367 둘 다 머지된 뒤 PR #3377로 제출. 자세한 내용은 `node-redis-pr3377-batch-command-slot-routing.md` 참고
