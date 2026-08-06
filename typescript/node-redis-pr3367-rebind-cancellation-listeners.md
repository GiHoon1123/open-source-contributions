# [Redis OSS 기여] redis/node-redis - 클러스터 슬롯 재배치 중 abort/timeout 리스너가 옛 큐를 가리키는 문제 수정 (PR #3367) — 🟢 Merged

> 오픈소스 기여 기록 및 학습 노트. 이 문서는 PR #3367 하나만 다룬다. **2026-07-28 16:44 머지됨. 6절 참고.**
>
> 이 PR은 PR #3364(`extractAllCommands` 드레인 버그 수정)에 Codex 리뷰봇이 남긴 두 가지 지적 중 하나(이슈 #3365)를 실제로 고친 것이다. #3364 문서의 6.5절에서 발견 경위를 다뤘고, 이 문서는 그 수정 자체에 집중한다.

---

## 1. 개요

### 1.1 이슈 (#3365)

**제목**: `Abort/timeout listener not rebound when a queued command is moved to another node during cluster slot migration`

`redis/node-redis` PR #3364를 올린 직후 Codex 리뷰봇이 지적한 내용을 우리 스스로 검증한 뒤, 별도 이슈로 등록했다. 핵심:

> `addCommand()`의 abort/timeout 리스너가 커맨드가 처음 큐잉된 큐/큐 항목을 클로저로 붙잡고 있는데, 클러스터 슬롯 재배치로 커맨드가 다른 노드의 큐로 옮겨지면 리스너는 그 이동을 모른 채 옛 위치를 계속 가리킨다.

### 1.2 관련 링크

| 구분         | 링크                                                                                            |
| ------------ | ----------------------------------------------------------------------------------------------- |
| 원본 저장소  | https://github.com/redis/node-redis                                                             |
| 내 Fork      | https://github.com/GiHoon1123/node-redis                                                        |
| 이슈 (#3365) | https://github.com/redis/node-redis/issues/3365                                                 |
| PR (#3367)   | https://github.com/redis/node-redis/pull/3367                                                   |
| 작업 브랜치  | `fix-3365-rebind-cancellation-listeners` (현재 upstream/master 기준, #3364 머지 후 rebase 예정) |
| 커밋         | `c3a528085b`                                                                                    |
| 상태         | 🟢 MERGED (2026-07-28 16:44)                                                                    |

### 1.3 한 줄 요약

> `addCommand()`가 만드는 abort/timeout 리스너는 "취소되면 이 큐, 이 큐 항목을 지워라"를 클로저로 고정해서 기억한다. 커맨드가 다른 클러스터 노드의 큐로 옮겨지면(전체/부분 슬롯 재배치) 이 리스너는 갱신되지 않아서, 취소돼도 유저의 Promise는 reject되지만 실제 커맨드는 새 큐에 그대로 남아 서버로 전송될 수 있다. 커맨드가 옮겨지는 지점(`prependCommandsToWrite`)에서 리스너를 새 큐/새 큐 항목 기준으로 다시 만들어 갈아끼우도록 고쳤다.

---

## 2. 배경 지식

### 2.1 리스너의 역할 — 취소를 실제로 작동시키는 연결고리

`AbortSignal`은 JS의 범용 "취소 신호" 메커니즘이다. 신호가 울린다고 저절로 뭔가 취소되는 게 아니라, 울렸을 때 뭘 할지를 미리 등록해둬야 한다. `addCommand()`가 등록해두는 리스너는 정확히 두 가지 일을 한다:

```ts
listener: () => {
  this.#toWrite.remove(node); // ① 큐에서 이 커맨드를 실제로 치운다
  value.reject(new AbortError()); // ② 유저가 기다리던 Promise를 reject한다
};
```

① 없이 ②만 하면, 취소 알림은 가지만 커맨드는 큐에 남아서 나중에 진짜로 서버에 전송·실행된다. 그래서 "치우기"와 "알리기"는 항상 한 세트로 실행돼야 진짜 취소가 완성된다.

### 2.2 비유 — 택배 위치가 바뀌었는데 취소 메모는 안 바뀐 경우

택배를 주문하면서 "10분 안에 안 오면 취소해줘"라는 규칙을 걸었다고 하자. 택배회사는 이 규칙을 이렇게 적어둔다:

> "취소되면 **A창고 3번 선반**에 있는 이 물건을 치워라"

그런데 회사가 A창고를 닫으면서 물건을 B창고로 몰래 옮겼다. 취소 규칙 메모는 안 바뀌고 여전히 "A창고 3번 선반"을 가리킨다. 10분 뒤 취소가 발동하면:

- "취소됐습니다"는 정상적으로 옴 (유저는 취소됐다고 믿음)
- 실제로 치우려고 간 곳은 이미 텅 빈 A창고 3번 선반 — 아무것도 안 지워짐
- 진짜 물건은 B창고에 그대로 남아서, 결국 배송될 수 있음

### 2.3 이 비유를 코드로 — 클러스터 슬롯 재배치

`cluster-slots.ts`가 클러스터 노드 사이에서 커맨드를 옮기는 두 지점:

```ts
// 부분 슬롯 재배치
const commandsForDestination = sourceNode.client
  ._getQueue()
  .extractCommandsForSlots(destinationSlots);
destMasterNode.client
  ?._getQueue()
  .prependCommandsToWrite(commandsForDestination);

// 전체 슬롯 손실 (PR #3364가 고친 그 경로)
const remainingCommands = sourceNode.client._getQueue().extractAllCommands();
lastDestNode.client?._getQueue().prependCommandsToWrite(remainingCommands);
```

두 경로 다 결국 `prependCommandsToWrite`를 거쳐 목적지 큐로 들어간다 — 즉 리바인딩 로직을 이 한 곳에 넣으면 두 마이그레이션 경로 다 커버된다. 다만 목적지 클라이언트가 없는 경우에는 명령어를 새 큐에 넣을 수 없으므로, 그대로 버려지거나 대기 상태로 남지 않게 reject해야 한다.

`prependCommandsToWrite`가 옮기는 건 커맨드의 `value`(내용)뿐이다:

```ts
prependCommandsToWrite(commands) {
  for (...) {
    this.#toWrite.unshift(commands[i]);   // 목적지 큐에 완전히 새 큐 항목 생성
  }
}
```

`value.abort`/`value.timeout`에 저장된 리스너는 **원래(출발) 큐, 원래(이제 사라질) 큐 항목**을 그대로 기억한 채로 이 이동에 아무 영향도 안 받는다.

---

## 3. 원인 분석

### 3.1 원래 코드

```ts
const signal = options?.abortSignal;
if (signal) {
  value.abort = {
    signal,
    listener: () => {
      this.#toWrite.remove(node); // 이 시점의 this(큐), node(큐 항목)를 클로저로 고정
      value.reject(new AbortError());
    },
  };
  signal.addEventListener("abort", value.abort.listener, { once: true });
}
```

`timeout`도 같은 패턴이다. 커맨드가 처음 큐잉된 곳에서 그대로 처리되는 한 문제없다. 문제는 슬롯 재배치로 이 `value`가 통째로 다른 큐로 이동할 때다:

1. `extractAllCommands()`/`extractCommandsForSlots()` — 출발 큐에서 큐 항목을 제거하고 `value`만 뽑아 반환
2. `prependCommandsToWrite()` — 그 `value`를 목적지 큐에 `unshift`, **완전히 새로운 큐 항목** 생성
3. `value.abort.listener`는 여전히 **1번 이전에 만들어진 클로저** 그대로 — 출발 큐(`this`)와 이미 제거된 옛 큐 항목(`node`)을 가리킴

이 상태에서 abort/timeout이 발동하면 `value.reject()`는 실행되지만(유저는 취소됐다고 믿음), `this.#toWrite.remove(node)`는 이미 사라진 출발 큐의 이미 제거된 항목을 지우는 거라 아무 효과가 없다 — **목적지 큐엔 커맨드가 그대로 남아 서버로 전송될 수 있다.**

### 3.2 추가로 발견한 케이스 — 옮기기 전에 이미 발동한 경우

리뷰 중 하나 더 확인해야 했던 케이스: 만약 abort/timeout이 **`prependCommandsToWrite`가 실행되기 전에** 이미 발동해버렸다면? `AbortSignal`은 한 번 발동하면(`aborted === true`) 그 이후에 새로 등록하는 리스너는 **절대 호출되지 않는다** — 이걸 직접 실험으로 확인했다:

```js
const controller = new AbortController();
controller.abort();
controller.signal.addEventListener("abort", () => console.log("FIRED"), {
  once: true,
});
// "FIRED"는 절대 안 찍힘
```

즉 "리바인딩만 하면 끝"이 아니라, **이미 발동된 신호에 새 리스너를 붙이는 건 죽은 코드**라는 걸 별도로 처리해야 했다.

---

## 4. 수정 내용

### 4.0 이동 전후의 차이

수정 전에는 커맨드를 추출할 때 큐 노드와 리스너를 함께 옮기지 않았다. 연결 리스트의 노드는 원래 큐에서 제거되고 `CommandToWrite` 값만 배열로 반환된다.

```ts
const command = current.value;
RedisCommandsQueue.#removeAbortListener(command);
RedisCommandsQueue.#removeTimeoutListener(command);
result.push(command);
this.#toWrite.remove(current);
```

그 다음 목적지 큐에서 새 노드를 만들었다.

```ts
const node = this.#toWrite.unshift(value);
```

문제는 `value.abort.listener`와 `value.timeout.listener`가 원래 큐와 원래 노드를 클로저로 기억한다는 점이다. 따라서 이동 후 기존 리스너가 실행되면 Promise는 reject하지만 목적지 큐의 새 노드는 제거하지 못한다.

수정 후에는 새 노드를 만든 직후 기존 리스너를 제거하고, 새 큐와 새 노드를 캡처하는 리스너를 다시 등록한다.

```ts
const node = this.#toWrite.unshift(value);

if (value.timeout) {
  RedisCommandsQueue.#removeTimeoutListener(value);

  if (value.timeout.signal.aborted) {
    this.#rejectCommand(node, value, new TimeoutError());
  } else {
    value.timeout.listener = this.#createTimeoutListener(
      node,
      value,
      value.timeout.effectiveTimeout,
      value.timeout.wasInMaintenance,
    );
    value.timeout.signal.addEventListener("abort", value.timeout.listener, {
      once: true,
    });
  }
}

if (value.abort) {
  RedisCommandsQueue.#removeAbortListener(value);

  if (value.abort.signal.aborted) {
    this.#rejectCommand(node, value, new AbortError());
  } else {
    value.abort.listener = this.#createAbortListener(node, value);
    value.abort.signal.addEventListener("abort", value.abort.listener, {
      once: true,
    });
  }
}
```

`signal.aborted`를 먼저 확인하는 이유는 이미 발생한 `AbortSignal`에 나중에 리스너를 등록해도 그 리스너가 다시 호출되지 않기 때문이다. 이 경우에는 새 리스너를 등록하는 대신 즉시 `#rejectCommand()`를 호출해 목적지 큐의 노드까지 제거한다.

### 4.1 리스너 생성 로직 공용화

`addCommand`, `setMaintenanceCommandTimeout`(유지보수 모드 진입 시 기존 커맨드들의 타임아웃을 재설정하는 코드), `prependCommandsToWrite` 세 곳이 전부 같은 모양의 리스너를 만들고 있어서, 팩토리 메서드 두 개로 뽑았다:

```ts
#createAbortListener(node: DoublyLinkedNode<CommandToWrite>, value: CommandToWrite) {
  return () => {
    this.#toWrite.remove(node);
    value.reject(new AbortError());
  };
}

#createTimeoutListener(
  node: DoublyLinkedNode<CommandToWrite>,
  value: CommandToWrite,
  effectiveTimeout: number,
  wasInMaintenance: boolean,
) {
  return () => {
    this.#toWrite.remove(node);
    value.reject(
      wasInMaintenance
        ? new CommandTimeoutDuringMaintenanceError(effectiveTimeout)
        : new TimeoutError(),
    );
  };
}
```

`CommandToWrite.timeout`에 `effectiveTimeout`/`wasInMaintenance` 필드를 추가했다 — 재바인딩 시점에 원래와 같은 에러 타입(`CommandTimeoutDuringMaintenanceError` vs `TimeoutError`)을 재구성하려면 이 정보가 필요하다.

### 4.2 `addCommand` 순서 변경 — 직접 만든 버그와 그 수정

처음 리팩터링할 때 `#createAbortListener(node, value)`처럼 `node`를 **파라미터로 넘겼는데**, 원래 코드는 `node`가 아직 `undefined`인 시점에 클로저가 그 변수를 **참조로** 캡처해서(나중에 재할당되면 클로저도 최신값을 보는 JS 클로저 특성 이용), `node = this.#toWrite.add(value, ...)`가 나중에 실행돼도 문제없이 동작하는 구조였다. 근데 이 클로저를 별도 메서드 파라미터로 넘기면, 메서드가 받는 시점(아직 `undefined`)의 값이 그대로 고정돼버려서 영원히 `undefined`를 참조하는 버그가 됐다.

고친 방법: `node`가 실제로 만들어진 뒤에 리스너를 생성하도록 순서를 바꿨다.

```diff
   return new Promise((resolve, reject) => {
-    // eslint-disable-next-line prefer-const -- assigned after closures that reference it are constructed
-    let node: DoublyLinkedNode<CommandToWrite>;
     const value: CommandToWrite = { ... };
     value.slotNumber = options?.slotNumber;

+    const node = this.#toWrite.add(value, options?.asap);
+
     // timeout/abort 리스너 설정 (이제 node는 이미 만들어진 값)
     ...
-
-    node = this.#toWrite.add(value, options?.asap);
   });
```

### 4.3 `prependCommandsToWrite` — 리바인딩 + 이미 발동한 경우 처리

```ts
prependCommandsToWrite(commands: CommandToWrite[]) {
  if (!commands.length) return;

  for (let i = commands.length - 1; i >= 0; i--) {
    const value = commands[i];
    const node = this.#toWrite.unshift(value);   // 목적지 큐에 새 큐 항목 생성

    if (value.timeout) {
      RedisCommandsQueue.#removeTimeoutListener(value);   // 옛 리스너 회수
      if (value.timeout.signal.aborted) {
        // 옮기기 전에 이미 발동한 경우 — 새 리스너를 붙여봐야 절대 안 불리므로
        // 그 자리에서 바로 처리
        this.#toWrite.remove(node);
        value.reject(
          value.timeout.wasInMaintenance
            ? new CommandTimeoutDuringMaintenanceError(value.timeout.effectiveTimeout)
            : new TimeoutError(),
        );
        continue;
      }
      value.timeout.listener = this.#createTimeoutListener(
        node, value, value.timeout.effectiveTimeout, value.timeout.wasInMaintenance,
      );
      value.timeout.signal.addEventListener("abort", value.timeout.listener, { once: true });
    }

    if (value.abort) {
      RedisCommandsQueue.#removeAbortListener(value);
      if (value.abort.signal.aborted) {
        this.#toWrite.remove(node);
        value.reject(new AbortError());
      } else {
        value.abort.listener = this.#createAbortListener(node, value);
        value.abort.signal.addEventListener("abort", value.abort.listener, { once: true });
      }
    }
  }
}
```

`signal.aborted` 체크는 처음에 없었다 — 사용자가 자체 검토(self-review)하면서 직접 추가했다. "이미 발동한 신호에 리스너를 새로 붙이면 절대 안 불린다"는 것까지 정확히 캐치한 것.

### 4.4 목적지 큐가 없을 때 reject

**이 항목은 최초 제출본엔 없었고, 메인테이너 리뷰 요청을 반영해 나중에 추가된 것이다 — 6절 참고.**

명령어를 추출한 뒤 목적지 노드에 클라이언트가 없으면 `prependCommandsToWrite()`를 호출할 수 없다. 이 경우 명령어의 기존 리스너를 정리하고 `DisconnectsClientError`로 reject하도록 `rejectCommands()`를 추가했다. 목적지를 찾지 못한 명령어가 큐 밖에 남아 Promise를 끝내지 못하는 상태를 피하기 위한 처리다.

이 경로에서는 실제로 명령어를 목적지 큐에 넣지 않았으므로, 로그도 `prepended`가 아니라 `rejected`로 남긴다.

클러스터 슬롯 이동 코드에서는 두 경로를 처리한다.

```ts
const commandsForDestination =
  sourceQueue.extractCommandsForSlots(destinationSlots);

if (destMasterNode.client) {
  destMasterNode.client
    ._getQueue()
    .prependCommandsToWrite(commandsForDestination);
} else {
  sourceQueue.rejectCommands(
    commandsForDestination,
    new DisconnectsClientError(),
  );
}
```

소스 노드가 슬롯을 전부 잃어 slotless 커맨드를 마지막 목적지로 옮기는 경로도 같은 규칙을 사용한다.

```ts
const remainingCommands = sourceQueue.extractAllCommands();

if (lastDestNode?.client) {
  lastDestNode.client._getQueue().prependCommandsToWrite(remainingCommands);
} else {
  sourceQueue.rejectCommands(remainingCommands, new DisconnectsClientError());
}
```

`rejectCommands()`는 이미 추출된 명령어를 다시 큐에 넣지는 않는다. 리스너를 정리하고 각 Promise를 `DisconnectsClientError`로 reject한다.

```ts
rejectCommands(commands: CommandToWrite[], err: Error) {
  for (const command of commands) {
    if (command.rejected) continue;

    command.rejected = true;
    RedisCommandsQueue.#removeCommandListeners(command);
    command.reject(err);
  }
}
```

---

## 5. 테스트

### 5.1 새 테스트 (`commands-queue.spec.ts`)

```ts
it("rebinds an abort listener so it removes the command from its new queue", async () => {
  const source = createQueue();
  const destination = createQueue();
  const controller = new AbortController();

  const promise = source.addCommand(["CMD"], {
    abortSignal: controller.signal,
  });
  promise.catch(() => {});

  const [command] = source.extractAllCommands();
  destination.prependCommandsToWrite([command]);

  controller.abort();
  await assert.rejects(promise, AbortError);

  assert.strictEqual(destination.extractAllCommands().length, 0);
});

it("does not keep a command if its abort signal fired before prepend", async () => {
  // ... controller.abort()를 prependCommandsToWrite 호출 전에 실행 ...
});

// timeout 버전도 각각 동일하게 2개
```

abort/timeout 각각에 대해 이동 후 신호 발생과 이동 전 신호 발생을 확인하고, 리스너 정리와 추출 후 목적지에 넣지 못한 명령어의 reject도 검증한다. 새 회귀 테스트는 총 8개이며, 전체 `@redis/client` 테스트는 `2657 passing, 4 pending`으로 통과했다. 목적지 큐 관련 회귀 테스트에서 `destination.extractAllCommands().length === 0`을 확인하는 것이 핵심이다 — Promise가 reject되는 것뿐 아니라 **큐에서 실제로 지워졌는지**를 검증한다.

### 5.2 disable-and-reconfirm

fix만 stash로 빼고 돌리니 4개 전부 `1 !== 0`으로 실패(커맨드가 목적지 큐에 그대로 남음) — 정확히 예상대로. 복구 후 4개 전부 통과 재확인.

### 5.3 브랜치 분리 — PR #3364와 독립시키기

이 fix 작업은 원래 PR #3364용 브랜치(`fix-3363-extract-all-commands`) 위에 커밋 안 된 상태로 얹혀 있었다. 사용자가 "이대로 커밋하면 #3364 PR이 업데이트되는 거 아니냐"고 지적 — 정확한 지적이었다.

현재 #3367은 `extractAllCommands`의 순회 순서 fix(#3364)와 같은 루프를 수정한다. 따라서 두 변경을 따로 유지할 수는 있지만, #3364가 먼저 머지되면 그 위로 rebase하고 겹치는 부분을 정리해야 한다.

그래서:

1. 작업 중이던 변경사항을 `git stash`
2. `upstream/master` 기준으로 새 브랜치(`fix-3365-rebind-cancellation-listeners`) 생성
3. `git stash pop` — `commands-queue.spec.ts`는 `upstream/master`에 아예 없던 파일이라 modify/delete 충돌 발생, 수동 해결
4. #3364와 겹치는 `extractAllCommands` 변경은 #3364 머지 후 rebase하면서 정리하기로 함
5. 이 브랜치에서 빌드·린트·전체 테스트를 다시 실행 — 전부 통과 확인

결과: 이 fix는 별도 PR #3367로 유지하고, #3364가 머지된 뒤 겹치는 변경을 rebase로 정리할 예정이다.

---

## 6. 메인테이너 리뷰와 반영 (2026-07-27)

### 6.1 리뷰 코멘트

`nkaradzhov`가 리뷰를 남겼다:

> "@GiHoon1123, thanks, the underlying bug is valid and this is the right layer for the fix.
>
> Two points before merge.
>
> First, extraction now detaches listeners unconditionally, so a command that is extracted but never prepended (destMasterNode.client is optional-chained and lastDestNode can be undefined) loses cancellation entirely and its promise never settles, whereas master would still reject it. Please guarantee reject-or-rebind and add a test for that path.
>
> Second, this edits the same extractAllCommands loop that #3364 fixes, so please rebase on top of #3364 once it lands."

두 가지 요청:

1. **reject-or-rebind 보장** — extraction이 리스너를 무조건 떼는데, 추출된 커맨드가 어떤 목적지에도 prepend 안 되는 경로(`destMasterNode.client`가 optional chaining이라 없을 수 있고, `lastDestNode`도 undefined일 수 있음)가 있으면 그 커맨드는 취소 수단을 완전히 잃고 promise가 영영 안 풀린다는 지적. 실제로 `cluster-slots.ts`의 `destMasterNode.client?._getQueue().prependCommandsToWrite(...)`가 optional chaining이라 이 경로가 실제로 있음을 코드로 확인했다.
2. **#3364와 겹치는 부분 리베이스** — 같은 `extractAllCommands` 루프를 건드리니 #3364가 머지된 뒤 그 위로 리베이스하라는 요청.

### 6.2 반영

두 요청 모두 반영해서 커밋 `c3a52808`("fix(cluster): reject commands without migration destination")로 push했다. 4.4절의 `rejectCommands()`가 이 커밋에서 추가된 것 — 목적지가 없어 prepend할 수 없는 커맨드를 그대로 버려두지 않고 명시적으로 reject하도록 했다. 관련 회귀 테스트도 같은 커밋에 추가.

답변 코멘트:

> "Addressed the two cases you mentioned. Extracted commands are now rejected when no destination queue is available, and abort/timeout listeners are rebound when commands are prepended to a new queue. I also added a regression test for the extracted-but-not-prepended case. I'll rebase #3367 on top of ..."

리베이스는 #3364가 머지된 뒤에 하기로 했다 — #3364가 아직 머지 전이라(승인은 됐고 CI도 통과했지만 머지 대기 중, `node-redis-pr3364-extract-all-commands-drain.md` 참고), 지금 리베이스하면 곧 바뀔 코드 위에서 작업하는 셈이라 헛수고가 될 수 있다는 판단.

---

## 7. 배운 점

### 7.1 클로저를 메서드로 뽑을 때 "언제 캡처되는가"를 다시 확인해야 한다

원래 코드가 `let node; ...클로저가 node 참조...; node = 나중에 할당`이라는 패턴으로 동작했던 건, JS 클로저가 변수를 **참조로** 캡처하기 때문이다. 이 로직을 별도 메서드로 추출하면서 `node`를 **파라미터로** 넘기면, 그 시점의 값(아직 `undefined`)이 고정돼버려 완전히 다른 동작이 된다. 리팩터링 시 "이 변수가 클로저 안에서 어떻게 캡처되고 있었는지"를 반드시 재확인해야 한다.

### 7.2 `AbortSignal`은 "이미 발동한 뒤엔 새 리스너를 절대 안 부른다"

한 번 발동한 시그널에 나중에 `addEventListener`로 리스너를 추가해도 그 리스너는 절대 안 불린다. 리바인딩처럼 "커맨드가 이동한 뒤 리스너를 다시 건다"는 로직을 짤 때는, "이동하는 그 순간에 이미 발동해있었을 가능성"을 반드시 별도로 처리해야 한다 — 그냥 리바인딩만 하면 조용히 죽은 코드가 된다.

### 7.3 사용자의 self-review가 실제 gap을 잡았다

위 6.2번 케이스는 원래 fix에 없었고, 사용자가 직접 테스트 2개를 추가로 짜면서(자체 검토) 코드에도 `signal.aborted` 체크를 직접 넣었다. 리뷰 담당(Claude)이 만든 fix를 사용자가 다시 검토해서 실제 gap을 잡아낸 사례.

### 7.4 브랜치 분리는 "의존성이 있는지"로 판단한다

"별개 이슈니까 별도 PR로"라는 원칙은 있었지만, 실제로 분리 가능한지는 "이 fix가 다른 fix의 코드에 의존하는가"로 판단해야 한다. 이번엔 두 fix가 같은 파일 안에서도 서로 다른 메서드를 건드려서 의존성이 없었고, 그래서 `upstream/master` 기준 새 브랜치로 깔끔하게 분리할 수 있었다.

### 7.5 리뷰가 지적한 엣지 케이스는 실제 코드 경로로 확인해야 한다

"목적지가 없을 수도 있다"는 지적을 받았을 때, 말로만 수긍하지 않고 `destMasterNode.client?._getQueue()...`가 실제로 optional chaining이라는 것, `lastDestNode`가 실제로 undefined일 수 있다는 것을 코드에서 직접 짚어서 확인한 뒤 수정했다. 리뷰 지적을 그대로 받아들이기 전에 "그 경로가 코드상 정말 존재하는가"를 확인하는 습관이 유효했다.

---

## 8. 다음에 할 일

- [x] 메인테이너 리뷰 유무 및 반응 — 2026-07-27, `nkaradzhov`가 2가지 요청(reject-or-rebind 보장 + #3364 위로 리베이스)과 함께 리뷰
- [x] 리뷰 요청 반영 — 커밋 `c3a52808`로 반영, 답변 코멘트 게시 완료
- [x] CI 결과 — 통과
- [x] #3364 머지 후 리베이스 — #3364 머지(2026-07-28) 후 리베이스 완료
- [x] 머지 여부와 소요 시간 — 2026-07-28 16:44 머지됨
- [x] #3366 (배치 커맨드 슬롯 라우팅) — #3364, #3367 둘 다 머지된 뒤 PR #3377로 제출. 자세한 내용은 `node-redis-pr3377-batch-command-slot-routing.md` 참고
