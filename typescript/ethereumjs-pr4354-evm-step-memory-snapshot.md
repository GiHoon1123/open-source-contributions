# [블록체인 OSS 기여] ethereumjs/ethereumjs-monorepo - EVM step 이벤트 memory live view 오염 버그 수정 (PR #4354) — 🟢 Merged

> 오픈소스 기여 기록 및 학습 노트. 이 문서는 PR #4354 하나만 다룬다. **2026-08-14 `holgerd77`가 승인 후 머지.**
>
> 이번 건은 기존 GitHub 이슈를 잡은 게 아니라, 이전 EVM `step` 이벤트 작업 경험을 바탕으로 같은 코드 영역을 다시 읽다가 직접 발견한 작은 correctness bug다. 수정은 `subarray()`를 `slice()`로 바꾸는 한 줄이지만, 의미를 이해하려면 EVM opcode 실행 순서, `step` 이벤트의 역할, `Uint8Array` view/copy 차이를 정확히 알아야 한다.

---

## 1. 개요

### 1.1 프로젝트 소개

- **저장소**: https://github.com/ethereumjs/ethereumjs-monorepo
- **성격**: Ethereum L1 Protocol Stack and EVM in TypeScript. Ethereum Foundation이 관리하는 TypeScript 기반 이더리움 실행 계층 모노레포다.
- **건드린 패키지**: `@ethereumjs/evm`
- **주요 파일**:
  - `packages/evm/src/interpreter.ts`
  - `packages/evm/test/runCall.spec.ts`

`@ethereumjs/evm`은 이더리움 바이트코드를 opcode 단위로 실행하는 EVM 구현체다. 이번 수정은 합의 로직이나 opcode semantics를 바꾸는 변경이 아니라, 디버거/트레이서가 구독하는 `step` 이벤트 payload의 memory 값이 나중 opcode 실행에 의해 오염되는 문제를 고친다.

### 1.2 관련 링크

| 구분 | 링크 |
|---|---|
| 원본 저장소 | https://github.com/ethereumjs/ethereumjs-monorepo |
| 내 Fork | https://github.com/GiHoon1123/ethereumjs-monorepo |
| PR (#4354) | https://github.com/ethereumjs/ethereumjs-monorepo/pull/4354 |
| 작업 브랜치 | `fix-evm-step-memory-snapshot` |
| 커밋 | `d54a71af55c4a3f8df085f1adf69291d820669d9` |
| 상태 | 🟢 Merged (2026-08-14) |

### 1.3 한 줄 요약

> EVM `step` 이벤트는 opcode 실행 직전의 stack/memory 상태를 외부 tracer에게 전달한다. 그런데 `memory`가 `subarray()`로 만들어진 내부 memory buffer의 live view라서, 이벤트를 받은 쪽이 과거 `event.memory`를 저장해도 나중 `MSTORE`가 같은 memory range를 쓰면 과거 이벤트 값까지 같이 바뀌었다. `slice()`로 바꿔 이벤트 발생 시점의 memory snapshot을 넘기도록 수정했다.

---

## 2. 배경 지식

### 2.1 EVM이란

EVM은 Ethereum Virtual Machine의 약자다. 스마트 컨트랙트 bytecode를 실행하는 가상머신이다.

개발자는 Solidity 같은 언어로 코드를 작성하지만, 실제 블록체인 노드가 실행하는 건 더 낮은 수준의 bytecode다. 이 bytecode는 opcode라는 작은 명령어들의 나열이다.

예를 들어 JavaScript에서 이렇게 쓰는 느낌의 일을:

```js
memory[0] = 1;
memory[0] = 2;
```

EVM bytecode에서는 여러 opcode로 쪼개 실행한다:

```text
PUSH1 0x01
PUSH1 0x00
MSTORE
PUSH1 0x02
PUSH1 0x00
MSTORE
STOP
```

### 2.2 opcode란

**opcode**는 EVM이 실행하는 가장 작은 명령어다.

대표적으로:

| opcode | 의미 |
|---|---|
| `PUSH1` | 1바이트 값을 stack에 넣는다 |
| `MSTORE` | memory에 32바이트 값을 저장한다 |
| `SSTORE` | contract storage에 값을 저장한다 |
| `CALL` | 다른 contract/account를 호출한다 |
| `STOP` | 실행을 종료한다 |

EVM은 bytecode를 한 번에 통째로 실행하지 않고, program counter가 한 opcode씩 이동하면서 실행한다.

### 2.3 stack이란

EVM은 stack machine이다. 대부분의 opcode는 인자를 함수 파라미터처럼 직접 받지 않고, stack 위에 쌓인 값을 꺼내서 사용한다.

예:

```text
PUSH1 0x01
PUSH1 0x00
MSTORE
```

실행 흐름:

```text
처음:
stack = []

PUSH1 0x01 실행 후:
stack = [1]

PUSH1 0x00 실행 후:
stack = [1, 0]

MSTORE 실행:
offset = 0
value = 1
memory[0..31] = 1
stack = []
```

정확한 pop 순서는 EVM 구현에 따라 stack top을 기준으로 설명해야 하지만, 이 테스트 이해에는 "값과 offset을 stack에 올려두고 `MSTORE`가 그걸 꺼내 memory에 쓴다" 정도면 충분하다.

### 2.4 memory란

EVM memory는 contract 실행 중에만 존재하는 임시 메모리다.

storage와 다르다:

| 구분 | 설명 |
|---|---|
| memory | 트랜잭션 실행 중에만 쓰는 임시 공간. 실행이 끝나면 사라짐 |
| storage | contract의 영구 저장소. 블록체인 state에 남음 |

이번 버그는 storage가 아니라 **memory**에 관한 문제다. 즉 합의 state root나 contract storage 값을 직접 바꾸는 문제가 아니라, tracer/debugger가 보는 실행 중 memory snapshot이 잘못되는 문제다.

### 2.5 `MSTORE`란

`MSTORE`는 EVM memory에 32바이트 word를 저장하는 opcode다.

예:

```text
PUSH1 0x01
PUSH1 0x00
MSTORE
```

의미:

```text
memory[0..31] = 0x0000000000000000000000000000000000000000000000000000000000000001
```

왜 32바이트냐면 EVM의 기본 word size가 256비트, 즉 32바이트이기 때문이다.

그래서 `1`이라는 작은 값도 memory에 들어갈 때는 다음처럼 32바이트로 저장된다:

```text
0x0000000000000000000000000000000000000000000000000000000000000001
```

### 2.6 step이란

EVM에서 **step**은 opcode 하나를 실행하는 단위다.

예:

```text
PUSH1 0x01
PUSH1 0x00
MSTORE
STOP
```

이면:

```text
step 1: PUSH1 0x01
step 2: PUSH1 0x00
step 3: MSTORE
step 4: STOP
```

정도로 보면 된다.

### 2.7 `step` 이벤트란

EthereumJS EVM은 opcode를 실행할 때마다 외부에 `step` 이벤트를 발행할 수 있다.

사용 예:

```ts
evm.events.on('step', event => {
  console.log(event.opcode.name);
  console.log(event.stack);
  console.log(event.memory);
});
```

이 이벤트는 보통 다음 용도로 쓰인다:

- EVM debugger
- transaction tracer
- opcode 단위 실행 로그
- gas 분석 도구
- 테스트 코드

중요한 점:

> `step` 이벤트는 해당 opcode 실행 **직전**에 발생한다.

즉 `MSTORE` 이벤트의 `memory`는 `MSTORE`가 memory에 write하기 전 상태여야 한다.

예:

```text
PUSH1 0x01
PUSH1 0x00
MSTORE
PUSH1 0x02
PUSH1 0x00
MSTORE
STOP
```

이 경우 이벤트 관점의 memory 상태:

```text
첫 번째 MSTORE 이벤트:
아직 첫 번째 MSTORE가 실행되기 전이므로 memory는 비어 있음

첫 번째 MSTORE 실행 후:
memory[0..31] = 1

두 번째 MSTORE 이벤트:
아직 두 번째 MSTORE가 실행되기 전이므로 memory에는 1이 있어야 함

두 번째 MSTORE 실행 후:
memory[0..31] = 2
```

따라서 두 번째 `MSTORE` step event의 memory는 `2`가 아니라 `1`이어야 한다.

---

## 3. 어떻게 이 문제를 찾았나

이번 PR은 기존 이슈를 집은 게 아니다. 직접 코드 리딩으로 찾은 버그다.

흐름은 이렇다:

1. 이전에 같은 저장소에서 EVM `step` 이벤트 관련 PR을 진행했다.
2. 그 과정에서 `packages/evm/src/interpreter.ts`의 `runStep()`과 `_runStepHook()` 흐름을 이미 파악했다.
3. 같은 영역을 다시 보던 중 `InterpreterStep` 객체 생성부에서 `stack`과 `memory`가 다르게 처리되는 걸 봤다.

문제의 코드:

```ts
const eventObj: InterpreterStep = {
  ...
  stack: this._runState.stack.getStack(),
  ...
  memory: this._runState.memory._store.subarray(0, Number(memorySize) * 32),
  ...
};
```

`stack`은 `getStack()`을 타고 들어가면 복사본을 반환한다:

```ts
getStack() {
  return this._store.slice(0, this._len)
}
```

반면 `memory`는 `subarray()`를 사용하고 있었다.

여기서 의심이 생긴다:

```text
step event는 opcode 실행 당시 상태를 외부에 알려주는 이벤트다.
stack은 복사본으로 나가는데, memory는 내부 mutable buffer view로 나간다.
그러면 나중 opcode가 memory를 바꿀 때, 과거 event.memory도 같이 바뀌는 것 아닌가?
```

이 가설을 검증하려고 같은 memory offset에 두 번 write하는 최소 bytecode를 만든 것이다.

---

## 4. 원인 분석

### 4.1 기존 코드

기존 `interpreter.ts`의 `_runStepHook()` 안에서 memory는 이렇게 만들어졌다:

```ts
memory: this._runState.memory._store.subarray(0, Number(memorySize) * 32),
```

여기서 핵심은 `subarray()`다.

### 4.2 `subarray()`란

`Uint8Array.prototype.subarray()`는 원본을 복사하지 않는다. 대신 원본 `ArrayBuffer`의 일부 구간을 바라보는 view를 만든다.

예:

```ts
const original = new Uint8Array([1, 2, 3, 4]);
const view = original.subarray(0, 2);

console.log(view); // [1, 2]

original[0] = 9;

console.log(view); // [9, 2]
```

`view`는 독립된 복사본이 아니라 원본을 계속 바라본다. 그래서 원본이 바뀌면 view도 바뀐다.

이번 버그에서는 `event.memory`가 바로 이런 view였다.

```text
event.memory
-> EVM 내부 memory buffer의 일부를 바라보는 live view
```

그래서 이벤트를 받는 순간에는 값이 맞아 보여도, 나중 opcode가 내부 memory buffer를 바꾸면 과거 이벤트의 `memory`도 같이 바뀐다.

### 4.3 `slice()`란

`Uint8Array.prototype.slice()`는 새 `Uint8Array`를 만들고 바이트를 복사한다.

예:

```ts
const original = new Uint8Array([1, 2, 3, 4]);
const copy = original.slice(0, 2);

console.log(copy); // [1, 2]

original[0] = 9;

console.log(copy); // [1, 2]
```

원본이 나중에 바뀌어도 `copy`는 유지된다.

정확히 말하면 객체 그래프를 재귀적으로 복사하는 "deep copy"라기보다는, `Uint8Array`의 바이트를 새 buffer에 복사하는 것이다. 이번 케이스에서는 바이트 배열이므로 이게 곧 필요한 snapshot이다.

### 4.4 왜 이게 버그인가

`step` 이벤트는 "현재 실행 중인 내부 memory에 접근할 수 있는 view"를 제공하는 API가 아니다.

의미상:

```text
이 opcode를 실행하기 직전 상태는 이렇다
```

를 알려주는 이벤트다.

따라서 이벤트를 받은 쪽이 다음처럼 로그를 저장하면:

```ts
const logs: InterpreterStep[] = [];

evm.events.on('step', event => {
  logs.push(event);
});
```

나중에 `logs`를 봤을 때 각 step 당시의 상태가 남아 있어야 한다.

기존 구현에서는:

```text
과거 이벤트 로그에 현재 memory buffer view가 들어감
-> 나중 opcode가 memory를 바꿈
-> 과거 로그가 현재 memory 상태로 오염됨
```

이 된다.

### 4.5 stack과 비교하면 더 명확하다

같은 `InterpreterStep` 안의 stack은 이미 복사본으로 나간다:

```ts
stack: this._runState.stack.getStack(),
```

`getStack()` 내부:

```ts
/**
 * Returns a copy of the current stack.
 */
getStack() {
  return this._store.slice(0, this._len)
}
```

주석도 명확하다:

```text
Returns a copy of the current stack.
```

즉 `InterpreterStep` payload는 내부 mutable state를 그대로 노출하기보다는, 이벤트 시점의 상태를 복사해서 주는 모델에 가깝다. memory만 이 모델에서 벗어나 있었다.

### 4.6 `Memory.read()`와 비교해도 같다

`packages/evm/src/memory.ts`의 `read()`도 기본값은 복사본을 반환한다:

```ts
read(offset: number, size: number, avoidCopy?: boolean): Uint8Array<ArrayBuffer> {
  this.extend(offset, size)

  const loaded = this._store.subarray(offset, offset + size) as Uint8Array<ArrayBuffer>
  if (avoidCopy === true) {
    return loaded
  }
  const returnBytes = new Uint8Array(size)
  returnBytes.set(loaded)

  return returnBytes
}
```

성능 때문에 view가 필요한 경우는 `avoidCopy: true`를 명시적으로 넘긴다. 기본값은 copy다.

따라서 외부 observer에게 노출되는 `step` event memory도 copy가 맞다.

---

## 5. 수정 내용

수정은 한 줄이다.

### 5.1 Before

```ts
memory: this._runState.memory._store.subarray(0, Number(memorySize) * 32),
```

### 5.2 After

```ts
memory: this._runState.memory._store.slice(0, Number(memorySize) * 32),
```

### 5.3 의미

기존:

```text
event.memory = 내부 memory buffer를 바라보는 live view
```

수정 후:

```text
event.memory = 이벤트 발생 시점의 memory bytes copy
```

즉 과거 이벤트가 나중 opcode write에 의해 바뀌지 않는다.

### 5.4 왜 `slice()`를 선택했나

`Uint8Array` 복사 방법은 여러 가지가 있다:

```ts
arr.slice(start, end)
new Uint8Array(arr.subarray(start, end))
Uint8Array.from(arr.subarray(start, end))
const copy = new Uint8Array(length); copy.set(arr.subarray(start, end))
```

이번 상황에서는 `slice()`가 가장 낫다.

- 의도가 명확하다: "이 구간을 복사한다"
- 코드가 한 줄이다
- `stack.getStack()`과도 같은 `slice()` 계열이다
- `Memory.read()`의 기본 복사 semantics와도 맞다
- step hook은 listener가 있거나 DEBUG일 때만 실행되므로 일반 execution hot path 영향이 제한적이다

주의할 점:

Node.js `Buffer.prototype.slice()`는 역사적으로 view처럼 동작해서 헷갈릴 수 있다. 하지만 여기 타입은 `Buffer`가 아니라 `Uint8Array`다. `Uint8Array.prototype.slice()`는 새 buffer에 바이트를 복사한다.

---

## 6. 테스트 설계

### 6.1 추가한 테스트

파일:

```text
packages/evm/test/runCall.spec.ts
```

추가한 테스트:

```ts
it('step event memory snapshots are not mutated by later writes', async () => {
  const evm = await createEVM()
  // PUSH1 0x01 PUSH1 0x00 MSTORE
  // PUSH1 0x02 PUSH1 0x00 MSTORE
  // STOP
  const code = hexToBytes('0x6001600052600260005200')
  const mstoreMemories: Uint8Array[] = []

  evm.events.on('step', (e) => {
    if (e.opcode.name === 'MSTORE') {
      mstoreMemories.push(e.memory)
    }
  })

  await evm.runCode({ code })

  assert.strictEqual(mstoreMemories.length, 2)
  assert.strictEqual(bytesToHex(mstoreMemories[0]), '0x')
  assert.strictEqual(
    bytesToHex(mstoreMemories[1]),
    '0x0000000000000000000000000000000000000000000000000000000000000001',
  )
})
```

### 6.2 bytecode 분석

테스트 bytecode:

```ts
const code = hexToBytes('0x6001600052600260005200')
```

hex를 opcode 단위로 나누면:

```text
0x60 01
0x60 00
0x52
0x60 02
0x60 00
0x52
0x00
```

opcode 이름:

```text
PUSH1 0x01
PUSH1 0x00
MSTORE
PUSH1 0x02
PUSH1 0x00
MSTORE
STOP
```

의미:

```text
memory[0..31] = 1
memory[0..31] = 2
stop
```

왜 같은 offset에 두 번 쓰나?

이번 버그는 "나중 write가 과거 event memory를 오염시키는가"를 보는 문제다. 같은 memory range에 `1`을 썼다가 `2`를 쓰면, 두 번째 `MSTORE` 직전 event memory가 나중에 `2`로 바뀌는지 쉽게 검출할 수 있다.

### 6.3 왜 `MSTORE` 이벤트만 모으나

테스트에서는 모든 step event를 저장하지 않는다:

```ts
evm.events.on('step', (e) => {
  if (e.opcode.name === 'MSTORE') {
    mstoreMemories.push(e.memory)
  }
})
```

이 bytecode에는 `PUSH1`, `MSTORE`, `STOP` 이벤트가 모두 발생한다. 하지만 memory write 전후를 검증하려면 `MSTORE` 직전 상태만 보면 된다.

그래서:

```text
첫 번째 MSTORE 이벤트 memory
두 번째 MSTORE 이벤트 memory
```

만 저장한다.

### 6.4 왜 `e.memory`를 그대로 저장하나

이 부분이 테스트의 핵심이다:

```ts
mstoreMemories.push(e.memory)
```

만약 이렇게 작성했다면:

```ts
mstoreMemories.push(bytesToHex(e.memory))
```

이 테스트는 버그를 제대로 못 잡을 수 있다. 이벤트를 받는 순간 문자열로 변환하면, 그 시점의 값이 문자열로 복사되기 때문이다.

이번 버그는:

```text
event.memory 객체 자체가 나중에 바뀌는 문제
```

다.

그래서 일부러 `Uint8Array` 객체를 그대로 저장해두고, EVM 실행이 완전히 끝난 뒤에 `bytesToHex()`로 확인한다.

즉:

```text
이벤트 수신 시점 값이 맞는가?
```

가 아니라:

```text
이벤트 수신 후 나중 opcode가 실행되어도 저장해둔 event.memory가 그대로 유지되는가?
```

를 검증한다.

### 6.5 기대값

첫 번째 `MSTORE` 이벤트:

```ts
assert.strictEqual(bytesToHex(mstoreMemories[0]), '0x')
```

이유:

```text
step event는 opcode 실행 전 발생한다.
첫 번째 MSTORE 전에는 아직 memory에 아무것도 쓰지 않았다.
따라서 memory는 비어 있어야 한다.
```

두 번째 `MSTORE` 이벤트:

```ts
assert.strictEqual(
  bytesToHex(mstoreMemories[1]),
  '0x0000000000000000000000000000000000000000000000000000000000000001',
)
```

이유:

```text
첫 번째 MSTORE는 이미 실행되어 memory[0..31]에 1을 썼다.
두 번째 MSTORE 이벤트는 두 번째 MSTORE 실행 전 발생한다.
따라서 이 시점 memory는 2가 아니라 1이어야 한다.
```

### 6.6 fix 전 실패 양상

fix 없이 테스트만 추가하면 실패한다.

기대값:

```text
0x0000000000000000000000000000000000000000000000000000000000000001
```

실제값:

```text
0x0000000000000000000000000000000000000000000000000000000000000002
```

왜냐하면 두 번째 `MSTORE` 이벤트의 `memory`는 처음에는 `...01` 상태의 내부 buffer를 바라보지만, 두 번째 `MSTORE`가 실행된 뒤 같은 buffer가 `...02`로 바뀌기 때문이다.

이게 바로 live view 오염이다.

### 6.7 테스트 위치

처음에는 파일 끝에 테스트가 붙어 있었다. 자체검토 중 `runCall.spec.ts` 안에 이미 다음 테스트가 있는 걸 확인했다:

```text
step event: ensure EVM memory and not internal memory gets reported
```

그래서 새 테스트를 그 바로 아래로 옮겼다. 동작에는 차이가 없지만, reviewer가 step event memory 관련 테스트를 한 덩어리로 읽을 수 있어 가독성이 낫다.

---

## 7. 실제 코드 흐름

### 7.1 step event는 언제 발행되는가

`interpreter.ts`의 `runStep()` 흐름은 대략 다음과 같다:

```text
현재 opcode를 읽음
동적 gas 계산
step event 발행
gas 차감
opcode handler 실행
다음 opcode로 이동
```

즉 step event는 opcode handler 실행 전이다.

그래서:

```text
MSTORE event memory = MSTORE 실행 전 memory
```

이어야 한다.

### 7.2 `_runStepHook()`가 이벤트 객체를 만든다

문제의 코드는 `_runStepHook()` 안에 있다.

```ts
async _runStepHook(dynamicFee: bigint, gasLeft: bigint, memorySize: bigint): Promise<void> {
  ...

  const eventObj: InterpreterStep = {
    pc: this._runState.programCounter,
    gasLeft,
    gasRefund: this._runState.gasRefund,
    opcode: {
      name: opcodeInfo.fullName,
      fee: opcodeInfo.fee,
      dynamicFee,
      isAsync: opcodeInfo.isAsync,
      code: opcodeInfo.code,
    },
    stack: this._runState.stack.getStack(),
    depth: this._env.depth,
    address: this._env.address,
    account: this._env.contract,
    memory: this._runState.memory._store.slice(0, Number(memorySize) * 32),
    memoryWordCount: memorySize,
    codeAddress: this._env.codeAddress,
    stateManager: this._runState.stateManager,
    eofSection: section,
    immediate,
    error,
    eofFunctionDepth: ...,
  }

  ...
}
```

여기서 `memorySize`는 현재 EVM memory word count다. memory는 32바이트 word 단위이므로:

```ts
Number(memorySize) * 32
```

만큼만 잘라서 event에 넣는다. 이건 내부 `_store` 전체 capacity를 노출하지 않고, 실제 EVM memory로 확장된 영역만 보고하기 위한 기존 동작이다.

이번 fix는 이 범위 계산을 바꾸지 않는다. 단지 같은 범위를 live view로 줄지, copy로 줄지만 바꾼다.

### 7.3 왜 `memory.read(0, size)`를 쓰지 않았나

대안으로 이런 코드도 생각할 수 있다:

```ts
memory: this._runState.memory.read(0, Number(memorySize) * 32)
```

하지만 기존 코드는 `_store`를 직접 slice/subarray 하고 있었다. `memory.read()`는 내부적으로 `extend(offset, size)`를 호출한다. 지금은 이미 `memorySize` 기준으로 읽으므로 큰 차이는 없겠지만, event 생성 경로에서 memory extension side effect가 생길 여지를 만들 필요가 없다.

따라서 기존 접근은 유지하고 다음 한 곳만 바꾸는 게 가장 작은 수정이다:

```text
_store.subarray(...) -> _store.slice(...)
```

---

## 8. 검증 기록

### 8.1 재현 검증

처음에는 test path를 잘못 줘서 다음 명령이 실패했다:

```bash
npm run test:node --workspace @ethereumjs/evm -- packages/evm/test/runCall.spec.ts -t "step event memory snapshots"
```

이유:

`--workspace @ethereumjs/evm`로 실행하면 working directory가 `packages/evm` 기준이 된다. 따라서 test path는 `test/runCall.spec.ts`로 줘야 한다.

올바른 명령:

```bash
npm run test:node --workspace @ethereumjs/evm -- test/runCall.spec.ts -t "step event memory snapshots"
```

fix 없이 새 테스트만 있으면 실패:

```text
Expected second memory: ...0001
Received:               ...0002
```

fix 적용 후 통과.

### 8.2 실행한 테스트

```bash
npm run test:node --workspace @ethereumjs/evm -- test/runCall.spec.ts -t "step event memory snapshots"
```

결과:

```text
1 passed
```

```bash
npm run test:node --workspace @ethereumjs/evm -- test/runCall.spec.ts -t "step event"
```

결과:

```text
4 passed
```

```bash
npm run test:node --workspace @ethereumjs/evm -- test/runCall.spec.ts
```

결과:

```text
23 passed
```

```bash
npm run lint --workspace @ethereumjs/evm
```

결과:

```text
biome checked 141 files
eslint passed
```

```bash
git diff --check
```

결과:

```text
passed
```

### 8.3 전체 EVM suite 시도

다음 명령도 시도했다:

```bash
npm run test:node --workspace @ethereumjs/evm
```

결과는 일부 실패:

```text
Cannot find module ../../../ethereum-tests/EOFTests/EIP5450/validInvalid.json
ENOENT scandir packages/ethereum-tests/EOFTests
```

이건 변경 때문이 아니라 local checkout에 `packages/ethereum-tests/EOFTests` fixture/submodule이 없는 문제다. 실패 전까지는 다수 테스트가 통과했고, 동일하게 `tsc`도 이 fixture import 문제에 걸렸다.

PR 본문에는 이 전체 suite 실패를 길게 넣지 않았다. 관련 범위 검증과 lint가 통과했고, 전체 실패 원인은 local fixture 누락이라 noise가 될 수 있기 때문이다.

---

## 9. PR 본문

실제 PR 본문은 과하게 장황하게 쓰지 않았다.

```md
## Summary

The EVM `step` event exposed `memory` with `subarray()`, which returns a view into the interpreter's mutable memory buffer.

That meant a listener storing `event.memory` could see an older step's memory change after a later opcode wrote to the same memory range.

This switches the step event memory value to `slice()`, so listeners receive a snapshot of memory at the time the step event is emitted.

## Test

Added a regression test that runs two `MSTORE` operations against the same memory offset and stores the raw `event.memory` values from the `step` events.

Without the fix, the second `MSTORE` event's memory is later mutated from `...01` to `...02`. With the fix, it stays at the pre-opcode snapshot.

```bash
npm run test:node --workspace @ethereumjs/evm -- test/runCall.spec.ts
npm run lint --workspace @ethereumjs/evm
```
```

이슈가 없기 때문에 `Fixes #...`는 넣지 않았다.

---

## 10. 자체검토

### 10.1 수정 방향이 맞는가

맞다.

이벤트 payload는 외부 observer가 나중에 저장/분석할 수 있는 실행 로그에 가깝다. 과거 event memory가 미래 opcode 실행으로 바뀌면 trace 자체를 신뢰할 수 없게 된다.

### 10.2 성능 문제는 없는가

일반 EVM 실행 전체에 항상 copy가 추가되는 건 아니다.

`_runStepHook()`는 다음 조건일 때만 실행된다:

```ts
if (this._evm.events.listenerCount('step') > 0 || this._evm.DEBUG) {
  await this._runStepHook(...)
}
```

즉 step listener가 있거나 DEBUG 모드일 때만 memory copy가 발생한다. tracer/debugger 모드에서는 정확한 snapshot이 성능보다 중요하다.

### 10.3 `subarray()`가 의도된 최적화였을 가능성은?

가능성은 있지만, 코드베이스의 다른 신호와 맞지 않는다.

- `stack`은 이미 copy다.
- `Memory.read()`는 기본 copy다.
- `avoidCopy: true`라는 명시적 escape hatch가 따로 있다.
- PR #2598처럼 step event memory 크기를 실제 EVM memory size로 제한하려는 과거 작업도 있었다. event memory는 어디까지나 외부에 보고할 값으로 관리하려는 거지, 내부 buffer view를 그대로 노출하려는 게 아니다.

따라서 `subarray()`는 의도된 public live view라기보다 copy가 빠진 실수로 보는 게 자연스럽다.

### 10.4 테스트가 너무 작지 않은가

작지만 충분하다.

이 버그의 본질은 복잡한 contract call이나 gas 계산이 아니라, "같은 memory buffer를 나중에 write하면 과거 event memory가 바뀌는가"다. 두 번의 `MSTORE`로 정확히 재현된다.

좋은 회귀 테스트의 조건:

- fix 없으면 실패한다.
- fix 있으면 통과한다.
- 외부 fixture가 필요 없다.
- 실패 원인이 한 가지로 좁혀진다.

이번 테스트는 이 조건을 만족한다.

### 10.5 큰 리스크

큰 리스크는 없다. 다만 리뷰에서 나올 수 있는 질문은 있다:

1. `slice()` copy로 인한 tracer 성능 영향은 괜찮은가?
2. memory를 정말 snapshot으로 보장해야 하는가?
3. `Memory.read()`를 쓰는 게 더 낫지 않은가?

답변 방향:

```text
step 이벤트는 opcode 실행 시점의 관찰 결과이며, stack은 이미 copy로 제공된다. memory만 live view면 listener가 저장한 과거 event가 나중 opcode write로 바뀐다. step hook은 listener/debug 모드에서만 실행되므로 일반 execution path 영향은 제한적이다. 기존 `_store` 범위 계산은 그대로 유지하고 view를 copy로 바꾼 최소 수정이다.
```

---

## 11. 이번 작업에서 배운 것

### 11.1 작은 한 줄 수정도 깊이는 있을 수 있다

코드 diff만 보면:

```diff
- subarray(...)
+ slice(...)
```

다.

하지만 이 한 줄을 제대로 설명하려면 다음을 알아야 했다:

- EVM이 opcode를 step 단위로 실행한다는 것
- `step` 이벤트가 opcode 실행 전 발행된다는 것
- `MSTORE`가 memory에 32바이트 word를 쓴다는 것
- `Uint8Array.subarray()`는 live view라는 것
- `Uint8Array.slice()`는 바이트 복사본이라는 것
- tracer/debugger가 event payload를 나중에 저장해서 볼 수 있다는 것

즉 구현 난이도는 낮지만, 발견하고 검증하려면 코드 의미를 제대로 이해해야 했다.

### 11.2 "참조를 넘긴다"는 말의 실전 의미

이번 버그를 가장 쉽게 표현하면:

```text
이벤트를 날릴 때 당시 메모리 상태를 값으로 복사해서 준 게 아니라,
내부 메모리를 바라보는 참조/view를 줬다.
그래서 이벤트를 받은 뒤 처리하는 사이 내부 메모리가 바뀌면,
그 이벤트 안의 memory도 같이 바뀌었다.
```

프론트/백엔드 코드에서도 비슷한 문제가 자주 있다:

```ts
const snapshot = obj.items;
obj.items.push(newItem);
```

이 경우 `snapshot`은 과거 배열이 아니라 같은 배열 참조다. 과거 상태를 남기려면 복사해야 한다.

```ts
const snapshot = [...obj.items];
```

이번 건은 그 배열 버전이 `Uint8Array.subarray()`와 `Uint8Array.slice()`로 나타난 것이다.

### 11.3 "이슈 없는 PR"도 가능하다

이 PR은 기존 이슈가 없다.

그래도 올릴 수 있었던 이유:

- 설계 변경이 아니다.
- public API 추가가 아니다.
- 작은 correctness bug다.
- 재현 테스트가 있다.
- 최근 이 저장소에서 외부 PR이 실제로 리뷰/머지되고 있다.
- 같은 `step` 이벤트 영역에서 이미 이전 작업 히스토리가 있다.

기준:

```text
큰 기능/설계 변경 -> 이슈 먼저
작은 버그 + 명확한 테스트 -> 바로 PR 가능
```

이번 건은 후자다.

---

## 12. 최종 상태

- PR: https://github.com/ethereumjs/ethereumjs-monorepo/pull/4354
- 상태: 🟢 Merged (2026-08-14, `holgerd77`)
- 변경 파일: 2개
- diff: +25 / -1
- 커밋: `d54a71af55c4a3f8df085f1adf69291d820669d9`
- 커밋 메시지: `fix(evm): snapshot step event memory`
- AI/Co-authored trailer: 없음
- `holgerd77`가 승인과 동시에 남긴 코멘트: "Sorry for the delay, thanks a lot, really important fix, will take in!" — 이슈 없이 코드 리딩만으로 올린 PR이라 리뷰가 3주 가까이 밀렸지만, 별도 논의나 수정 요청 없이 원안 그대로 머지됐다.

검증 요약:

```text
targeted regression: pass
step event tests: pass
runCall suite: pass
EVM lint: pass
diff whitespace: pass
full EVM suite: local ethereum-tests EOF fixture 누락으로 일부 실패, 변경과 무관
```

이번 기여의 성격:

```text
이슈 기반 PR 아님
코드 리딩으로 발견한 correctness fix
EVM tracer/debugger event payload 정확성 개선
한 줄 수정 + 회귀 테스트
```

