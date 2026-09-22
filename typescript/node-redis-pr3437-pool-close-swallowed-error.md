# [OSS 기여] redis/node-redis - `pool.close()` 종료 오류 처리 (PR #3437, MERGED)

상태: 🟢 MERGED (2026-09-06)  
저장소: https://github.com/redis/node-redis  
PR: https://github.com/redis/node-redis/pull/3437  
브랜치: `fix-pool-close-swallowed-error`  
최종 커밋: `2a8f5a8bb9abf974bde527bf81fcfe646ba0d04b`  
머지 커밋: `d33e4af24725052b8743197d1481147b67bdc1ff` (squash, nkaradzhov)

이 문서는 변경 목록이 아니라 `RedisClientPool.close()`의 상태 전이와 오류 전달을 복습하기 위한 기록이다.

## 1. 풀 종료가 하는 일

`RedisClientPool`은 여러 RedisClient를 묶어 하나의 풀처럼 제공한다. `execute()`가 작업을 받으면 idle client를 빌려주고, 작업이 끝나면 다시 idle 목록으로 돌려보낸다.

```text
execute() -> idle client를 빌림 -> 작업 실행 -> client를 반환
```

`close()`는 즉시 소켓을 끊는 함수가 아니다.

1. 진행 중인 작업과 대기 중인 작업이 끝날 때까지 기다린다.
2. idle client를 닫는다.
3. client 목록과 cache를 정리한다.
4. 풀이 닫혔음을 기록한다.

종료 중에는 새 `execute()`를 받으면 안 된다. 호출자는 `pool.close()`의 성공 또는 실패를 알 수 있어야 한다.

## 2. 처음 발견한 버그

기존 구현은 idle client 종료를 `Promise.all()`로 기다리고, 바깥 `catch {}`에서 오류를 버리고 있었다.

```ts
try {
  const promises = []
  for (const client of this._self.#idleClients) {
    promises.push(client.close())
  }
  await Promise.all(promises)
  this._self.#idleClients.reset()
  this._self.#clientsInUse.reset()
} catch {
  // 아무 것도 하지 않음
} finally {
  this._self.#isClosing = false
  this._self.#isOpen = false
}
```

두 문제가 겹쳤다.

- `Promise.all()`은 하나가 reject되면 즉시 reject한다. 다른 client가 아직 닫히는 중이어도 기다리지 않는다.
- 빈 `catch`가 오류를 삼킨다. 호출자는 `close()`가 성공한 것처럼 보지만 목록 정리는 건너뛸 수 있다.

그 결과 client 하나의 `close()`가 실패하면 호출자는 오류를 받지 못하고, 풀 내부에는 닫히지 않은 client가 남을 수 있었다.

## 3. 첫 번째 수정

실패한 client는 `destroy()`로 정리하고, 모든 client의 종료 시도는 `Promise.allSettled()`로 끝까지 기다리도록 바꿨다.

```ts
const promises = []
for (const client of this._self.#idleClients) {
  promises.push(client.close().catch(err => {
    try {
      client.destroy()
    } catch {
      // 원래 close 오류를 유지한다.
    }
    throw err
  }))
}

const results = await Promise.allSettled(promises)
for (const result of results) {
  if (result.status === 'rejected') throw result.reason
}
```

정리 작업은 `finally`로 옮겼다. 따라서 종료 중 오류가 나도 client 목록과 풀 상태는 정리되고, 첫 번째 close 오류는 호출자에게 전달된다.

## 4. 리뷰에서 드러난 추가 상태 문제

### 4.1 동시에 `close()`를 호출하는 경우

기존에는 두 번째 호출이 `#isClosing`을 보고 즉시 resolve됐다. 이제 첫 번째 `close()`가 실제 오류를 reject할 수 있으므로 두 호출의 결과가 달라졌다.

이를 해결하기 위해 진행 중인 promise를 `#closePromise`에 저장하고, 동시 호출자는 같은 promise를 반환하도록 했다.

```ts
if (this._self.isClosing) return this._self.#closePromise
```

### 4.2 cache cleanup이 원래 오류를 덮는 경우

`finally` 안에서 `onPoolClose()`가 throw하면 JavaScript의 `finally` 규칙 때문에 이미 발생한 client close 오류가 cache 오류로 바뀔 수 있다. cache hook도 별도 `try/catch`로 감싸 원래 close 오류를 보존했다.

### 4.3 close promise 게시 순서

`this.#closePromise = this.#doClose()`는 우변인 `#doClose()`가 먼저 실행된다. client의 동기 shutdown hook이 실행되는 순간 아직 `#closePromise`가 설정되지 않아, 재진입한 작업이 종료 중인 풀에 들어갈 수 있다.

```ts
this._self.#closePromise = Promise.resolve().then(() => this._self.#doClose())
```

이렇게 하면 promise를 먼저 게시하고, 실제 종료 로직은 다음 microtask에서 시작한다.

### 4.4 cache hook 동안 종료 상태 유지

마지막으로 상태를 너무 일찍 초기화하던 문제가 있었다.

```ts
// 잘못된 순서
this._self.#isOpen = false
this._self.#closePromise = undefined
this._self.#clientSideCache?.onPoolClose()
```

cache hook이 동기적으로 `pool.connect()`를 호출하면 풀이 닫힌 상태가 아닌 것으로 보일 수 있다. 현재는 hook을 먼저 실행하고, hook이 끝난 뒤 client 목록·`#isOpen`·`#closePromise`를 정리한다.

## 5. 메인테이너 요청 반영

메인테이너는 `#isClosing`과 `#closePromise`가 항상 함께 설정·해제되므로 별도 필드가 불필요하다고 지적했다.

```ts
get isClosing() {
  return this._self.#closePromise !== undefined
}
```

`#isClosing` 필드를 삭제하고 내부 참조를 getter로 통일했다. 공개 동작은 바꾸지 않으면서 상태의 단일 기준을 만들었다.

## 6. 회귀 테스트

`packages/client/lib/client/pool.spec.ts`에는 다음 경로를 검증하는 테스트를 추가했다.

- 이미 닫힌 client가 있을 때 `close()`가 `ClientClosedError`로 reject되고 풀이 정리되는지
- 한 client의 close가 실패해도 다른 client의 종료를 기다리는지
- credentials provider의 dispose와 destroy가 실패하는 경우 원래 오류를 유지하는지
- 동시 `close()` 호출자가 같은 rejection을 받는지
- client close hook에서 새 작업을 시작하려 할 때 거부되는지
- cache `onPoolClose()`에서 `connect()`를 호출해도 새 client가 생기지 않는지

마지막 두 테스트는 종료 hook의 동기 재진입을 직접 다룬다. 구현 내부 플래그보다 `pool.totalClients`, `pool.isOpen`, promise rejection처럼 호출자가 볼 수 있는 결과를 확인한다.

## 7. 검증

- `pnpm run build`: 통과
- 변경 파일 ESLint: 통과
- `git diff --check`: 통과
- 전체 pool 테스트: macOS Docker의 `--network host` 동작 때문에 setup hook이 6379 포트를 감지하지 못해 timeout

로컬에서는 테스트용 Redis 컨테이너가 host 네트워크로 남아 포트를 점유한 문제도 확인했다. 잔여 컨테이너를 중지한 뒤 새 컨테이너가 Redis ready 상태까지 올라오는 것은 확인했지만, Docker Desktop 환경에서 테스트 유틸의 포트 감지가 계속 timeout됐다. 이는 assertion 실패가 아니라 테스트 인프라 제약이다.

## 8. 리뷰 라운드 기록

처음 Codex 리뷰는 오류를 삼키는 동작을 지적했다. 이후 동시 호출, promise 게시 순서, cache hook 재진입 순서가 차례로 드러났다. 각 지적을 코드와 회귀 테스트에 반영했고, 메인테이너의 `#isClosing` 제거 요청도 적용했다. 최신 커밋은 `2a8f5a8bb9`이며 PR에는 반영 내용을 요약한 답변을 남겼다.

## 9. 복습 요약

Promise 기반 종료 코드는 “모든 자원을 닫았는가”뿐 아니라 “모든 호출자가 같은 종료 결과를 받는가”까지 봐야 한다. `finally`는 정리를 보장하지만 그 안의 예외가 원래 예외를 덮을 수 있고, async 함수는 promise를 반환하기 전에 동기 구간을 실행할 수 있다. 따라서 상태를 언제 게시하고 언제 지우는지가 실제 API 동작의 일부다.

## 10. 최종 상태

- 2026-09-05: 메인테이너 `nkaradzhov`가 1차 승인(`#isClosing`/`#closePromise` 중복 지적 포함)
- 이후 promise 게시 순서(4.3), cache hook 재진입(4.4) 문제를 추가로 고치고, `#isClosing` 필드를 getter로 교체(5절)
- 2026-09-06: `nkaradzhov`가 최종 승인, 같은 날 3개 커밋을 squash merge (`d33e4af247`)
- 머지된 최종 코드로 직접 확인: `#isClosing` 필드는 삭제되고 `get isClosing() { return this._self.#closePromise !== undefined }`로 대체됨, `#closePromise = Promise.resolve().then(() => this._self.#doClose())`로 게시 순서 수정도 그대로 반영됨
- 이 문서가 다루는 3개 원인(빈 catch, 동시 close 공유, cache 오류 은폐) 모두 머지된 코드에 남아있음을 확인
