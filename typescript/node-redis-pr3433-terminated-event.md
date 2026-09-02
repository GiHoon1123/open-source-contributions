# [OSS 기여] redis/node-redis - 재연결 포기 상태를 알리는 `terminated` 이벤트 추가 (PR #3433, MERGED)

> 오픈소스 기여 기록 및 기술 복습 노트. 이 문서는 PR #3433의 문제 분석, 구현, 리뷰 대응 과정을 정리한다.
> **2026-09-01 MERGED.**

---

## 1. 프로젝트와 작업 범위

### 1.1 프로젝트

- 저장소: https://github.com/redis/node-redis
- npm 패키지: `redis`, `@redis/client`
- 핵심 작업 파일: `packages/client/lib/client/socket.ts`
- 소켓 테스트: `packages/client/lib/client/socket.spec.ts`
- 클라이언트 전달 테스트: `packages/client/lib/client/index.spec.ts`
- 작업 브랜치: `fix-2948-terminated-event`
- 대상 브랜치: `master`

`node-redis`는 Node.js에서 Redis 서버와 통신하는 클라이언트 라이브러리다. 사용자가 직접 TCP 소켓을 다루지 않아도 되도록 연결, 인증, 명령 전송, 응답 큐, 재연결, 이벤트 전달을 내부에서 관리한다.

이번 작업은 Redis 프로토콜 명령을 추가하는 일이 아니라, 연결이 끊겼을 때 클라이언트가 어떤 상태에 있는지 애플리케이션이 구분할 수 있도록 연결 생명주기 이벤트를 추가하는 작업이었다.

### 1.2 관련 링크

| 구분 | 링크 |
|---|---|
| 원본 저장소 | https://github.com/redis/node-redis |
| Fork | https://github.com/GiHoon1123/node-redis |
| 이슈 #2948 | https://github.com/redis/node-redis/issues/2948 |
| PR #3433 | https://github.com/redis/node-redis/pull/3433 |
| 작업 브랜치 | `fix-2948-terminated-event` |
| 현재 상태 | ✅ MERGED (2026-09-01) |

### 1.3 커밋

- `b2705d2bb9` `fix(client): emit termination event when reconnecting stops`
- `6b9516352c` `docs(client): document terminated event`
- `87a1fc7299` `fix(client): emit termination before error`
- `36a1037980` `fix(client): order terminal socket events`
- `a04d4f81d4` `fix(client): handle terminal event edge cases`
- `104bc2e919` `fix(client): preserve terminal error handling`
- `f74c465dae` `docs(client): clarify terminated event ordering`

---

## 2. 이슈의 문제

### 2.1 기존 `error` 이벤트만으로는 상태를 알 수 없었다

기존 `RedisSocket`은 연결 오류가 발생하면 `error` 이벤트를 발생시켰다. 문제는 `error`가 다음 두 상황에서 모두 발생한다는 점이었다.

```text
일시적인 연결 오류
  -> error
  -> reconnecting
  -> 다시 연결

재연결을 포기한 오류
  -> error
  -> 더 이상 재연결하지 않음
```

애플리케이션 입장에서는 첫 번째 `error`만 보고 재연결이 계속될지, 클라이언트를 폐기해야 할지 판단하기 어렵다.

예를 들어 사용자가 연결을 교체하는 로직을 가지고 있다고 하자.

```ts
client.on('error', error => {
  console.error(error);
  // 이 시점에는 일시적인 오류인지 최종 종료인지 알 수 없다.
});
```

이슈 #2948은 `reconnectStrategy`가 더 이상 재연결하지 않기로 결정했을 때 별도의 이벤트가 필요하다는 내용이었다.

### 2.2 `reconnectStrategy`가 의미하는 것

`reconnectStrategy`는 연결이 끊긴 뒤 다음 행동을 결정한다.

```ts
createClient({
  socket: {
    reconnectStrategy(retries, cause) {
      if (retries > 5) return false;
      return Math.min(retries * 100, 3000);
    }
  }
});
```

반환값의 의미는 다음과 같다.

| 반환값 | 의미 |
|---|---|
| 숫자 | 해당 밀리초 후 재연결을 시도한다 |
| `false` | 재연결하지 않는다 |
| `Error` | 재연결하지 않고 해당 오류를 최종 원인으로 사용한다 |

기존 구현은 `false`나 `Error`를 받아도 `error`만 발생시켰다. 따라서 이번 수정은 “새 오류를 추가”한 것이 아니라, 재연결이 끝났다는 상태 신호를 별도로 노출하는 것이 목적이다.

---

## 3. node-redis 연결 구조

### 3.1 `RedisClient`와 `RedisSocket`

구조를 단순화하면 다음과 같다.

```text
사용자 코드
    |
    v
RedisClient
    |
    |  공개 API와 명령 큐 관리
    v
RedisSocket
    |
    |  TCP/TLS socket, 연결과 재연결 관리
    v
Node.js net.Socket 또는 tls.TLSSocket
```

`RedisSocket`은 실제 소켓의 상태와 재연결을 관리한다. `RedisClient`는 명령 큐와 사용자에게 공개되는 API를 관리한다.

따라서 `terminated`를 내부 `RedisSocket`에서만 발생시키면 사용자는 받을 수 없다. 내부 이벤트를 공개 클라이언트로 전달하는 forwarding이 필요하다.

```ts
// RedisClient.#attachListeners()
socket
  .on('error', error => this.emit('error', error))
  .on('connect', () => this.emit('connect'))
  .on('ready', () => this.emit('ready'))
  .on('reconnecting', () => this.emit('reconnecting'))
  .on('terminated', cause => this.emit('terminated', cause));
```

이 패턴은 어댑터와 비슷하다. 내부 객체의 이벤트를 그대로 외부로 복사해 공개 API의 이벤트로 전달한다. 실제 원인은 내부 소켓에 있지만, 사용자는 `RedisClient`만 사용하면 된다.

### 3.2 연결 상태

`RedisSocket`에는 여러 상태 값이 있다.

- `#isOpen`: 클라이언트가 연결을 유지하려는 상태인지 나타낸다.
- `#isReady`: 연결과 초기화가 끝나 명령을 처리할 수 있는지 나타낸다.
- `#socket`: 현재 사용하는 실제 네트워크 소켓이다.

대략적인 상태 변화는 다음과 같다.

```text
connect()
  #isOpen = true
  socket 생성
  connect 이벤트
  initiator 실행
  ready 이벤트
  #isReady = true

오류
  #isReady = false
  소켓 정리
  reconnectStrategy 확인

숫자 반환
  error
  reconnecting
  다시 connect()

false 또는 Error 반환
  #isOpen = false
  terminated
  error
  connect() reject
```

---

## 4. 초기 구현

### 4.1 `#shouldReconnect()`의 역할

재연결 여부를 결정하는 중앙 지점은 `RedisSocket.#shouldReconnect()`다. 이 메서드는 전략을 실행하고 반환값에 따라 종료 또는 재시도 흐름을 선택한다.

초기 구현의 핵심은 다음과 같다.

```ts
if (retryIn === false) {
  this.#isOpen = false;
  this.emit('terminated', cause);
  this.emit('error', cause);
  return cause;
}

if (retryIn instanceof Error) {
  this.#isOpen = false;
  const terminatedBy = new ReconnectStrategyError(retryIn, cause);
  this.emit('terminated', terminatedBy);
  this.emit('error', cause);
  return terminatedBy;
}

return retryIn;
```

`reconnectStrategy`가 `Error`를 반환하는 경우에는 전략이 반환한 오류와 실제 소켓 오류가 다를 수 있다.

```text
실제 소켓 오류: Connection timeout
전략 반환 오류: stop reconnecting
```

그래서 `terminated`에는 두 정보를 조합한 `ReconnectStrategyError`를 전달하고, 기존 `error`에는 실제 소켓 원인인 `cause`를 전달했다.

### 4.2 왜 `error`를 없애지 않았는가

기존 사용자는 `error` 이벤트에 의존하고 있다. `terminated`를 추가한다고 해서 기존 `error`를 제거하면 호환성이 깨진다.

이번 작업의 계약은 다음과 같다.

```text
재시도하는 오류:
  error -> reconnecting

최종 종료 오류:
  terminated -> error
```

즉 `error`는 계속 발생하지만, `terminated`가 함께 발생하면 재연결이 끝났다는 것을 알 수 있다.

---

## 5. 이벤트 순서와 Node.js `EventEmitter`

### 5.1 이벤트는 기본적으로 동기적으로 전달된다

Node.js `EventEmitter.emit()`은 리스너를 즉시 호출한다.

```ts
emitter.on('event', () => {
  console.log('listener');
});

console.log('before');
emitter.emit('event');
console.log('after');
```

출력은 다음과 같다.

```text
before
listener
after
```

따라서 이벤트 순서는 문서상의 설명이 아니라 실제 동작 계약이다. `terminated`와 `error` 중 무엇을 먼저 발생시키는지에 따라 사용자의 상태 처리 결과가 달라진다.

### 5.2 `events.once()`의 주의점

Node.js의 `events.once()`는 Promise를 반환한다.

```ts
const [cause] = await once(client, 'terminated');
```

하지만 `once()`는 대상 이벤트가 아닌 `error` 이벤트도 특별히 감시한다. 같은 동기 실행 안에서 `terminated`와 `error`가 연달아 발생하면, 대상 Promise가 정리되기 전에 `error`가 Promise를 reject할 수 있다.

따라서 이번 테스트에서는 `events.once(socket, 'terminated')`만 사용하는 대신, 일반 이벤트 리스너로 두 이벤트의 실제 순서를 기록했다.

```ts
const events: string[] = [];
let terminatedCause: Error | undefined;

socket.on('terminated', cause => {
  events.push('terminated');
  terminatedCause = cause;
});
socket.on('error', () => events.push('error'));

await assert.rejects(socket.connect());

assert.deepEqual(events.slice(-2), ['terminated', 'error']);
```

이 테스트는 Promise API 사용 가능 여부를 과장하지 않고, 이 PR이 보장해야 하는 핵심인 이벤트 발생 순서를 직접 검증한다.

### 5.3 Node.js의 microtask queue와 task queue

이번 리뷰에서 말한 “다음 큐로 넘긴다”는 표현은 Redis 내부 명령 큐와 Node.js 실행 큐를 구분해서 이해해야 한다.

#### EventEmitter 리스너는 현재 호출 스택에서 실행된다

```ts
emitter.emit('terminated');
client.destroy();
```

`terminated` 리스너가 `destroy()`를 호출하면 `destroy()`는 나중에 실행되는 것이 아니라 현재 `emit()` 호출 중에 즉시 실행된다.

```text
emit('terminated')
  └─ terminated listener
       └─ destroy() 즉시 실행
```

#### `queueMicrotask()`는 현재 동기 작업 뒤로 실행을 미룬다

```ts
client.on('terminated', () => {
  queueMicrotask(() => client.destroy());
});
```

이 코드는 `destroy()`를 별도의 긴 작업으로 만드는 것이 아니다. 현재 JavaScript 호출 스택이 끝나고, 같은 실행 흐름에서 예약된 동기 이벤트 처리가 끝난 직후 실행하도록 미룬다.

```text
현재 호출 스택
  ├─ terminated listener 실행
  ├─ destroy()를 microtask queue에 등록
  └─ companion error listener 실행

현재 호출 스택 종료
  └─ microtask queue의 destroy() 실행
```

일반적인 순서는 다음처럼 이해할 수 있다.

```text
동기 코드
  ↓
process.nextTick queue (Node.js 특수 큐)
  ↓
Promise callback / queueMicrotask (microtask queue)
  ↓
timer, I/O, setImmediate 등의 다음 이벤트 루프 단계
```

정확한 이벤트 루프 단계는 실행 상황에 따라 달라질 수 있지만, 이번 수정에서 필요한 사실은 하나다. `queueMicrotask()`로 예약한 `destroy()`는 같은 `emit()` 호출 안에서 이어지는 `error` 처리보다 뒤에 실행된다.

#### `setTimeout()`과 `queueMicrotask()`는 다르다

```ts
queueMicrotask(() => client.destroy()); // 현재 작업 직후
setTimeout(() => client.destroy(), 0);  // timer 단계에서 나중에
```

`setTimeout(..., 0)`도 실행을 미룰 수 있지만 최소 한 번의 timer 단계가 필요하다. 이번 문제는 companion `error` 이벤트만 먼저 처리하면 되므로 더 짧은 지연을 사용하는 `queueMicrotask()`가 적절하다.

#### Redis 명령 큐는 Node.js 실행 큐와 다르다

`RedisCommandsQueue`는 Node.js의 microtask queue가 아니다. Redis 명령을 소켓에 바로 쓸 수 없을 때 보관하는 애플리케이션 레벨 자료구조다.

```text
사용자 명령
  ↓
RedisCommandsQueue
  ↓ 연결 가능
RedisSocket.write()
  ↓
TCP socket
```

반면 `queueMicrotask()`는 JavaScript 코드의 실행 시점을 조정한다.

```text
RedisCommandsQueue: Redis 명령을 보관하는 큐
microtask queue: JavaScript callback 실행을 미루는 큐
```

`client.destroy()`는 Redis 명령 큐에 남아 있는 명령을 `DisconnectsClientError`로 정리한다. 그래서 이번 문제에서는 두 큐가 연관된다.

```text
terminated listener
  └─ destroy()를 즉시 호출
       └─ RedisCommandsQueue를 먼저 flush
            └─ 명령이 DisconnectsClientError를 받음

terminated listener
  └─ queueMicrotask에 destroy() 예약
       └─ companion error가 먼저 명령 큐를 처리
       └─ 그 다음 destroy()가 리소스를 정리
```

즉 “task queue로 옮긴다”는 말은 Redis 명령을 다른 큐로 옮긴다는 뜻이 아니라, `destroy()` callback의 실행을 Node.js 이벤트 루프의 뒤쪽 시점으로 미룬다는 뜻이다.

### 5.4 `on()`과 `once()`의 차이

Node.js `EventEmitter`에는 같은 이벤트를 여러 번 받을지 한 번만 받을지에 따라 두 API가 있다.

```ts
socket.on('error', handler);   // 등록된 동안 모든 error 수신
socket.once('error', handler); // 다음 error 한 번만 수신 후 자동 제거
```

연결 오류는 retry 중 여러 번 발생할 수 있으므로 일반적인 오류 처리에는 `on()`이 맞다.

```ts
client.on('error', error => {
  console.error(error);
});
```

반대로 테스트에서 “이번 연결 시도에서 발생하는 첫 번째 오류를 기다린다”는 의미라면 `once()`가 편리하다.

```ts
const [error] = await once(socket, 'error');
```

다만 `once()`는 Promise를 반환하고 `error` 이벤트를 특별히 다루므로, `terminated`와 `error`가 같은 동기 호출에서 연달아 발생하는 테스트에는 주의해야 한다. 이번 테스트에서는 이벤트 발생 순서 자체가 검증 대상이므로 `on()`으로 순서를 기록했다.

### 5.5 테스트를 API 호출 순서대로 작성하는 이유

이벤트 기반 코드는 호출 결과만 확인하면 중간 순서의 오류를 놓칠 수 있다. 예를 들어 다음 테스트는 `terminated`가 발생했다는 사실만 확인한다.

```ts
let terminated = false;
socket.on('terminated', () => {
  terminated = true;
});

await assert.rejects(socket.connect());
assert.equal(terminated, true);
```

하지만 다음과 같은 잘못된 구현도 통과할 수 있다.

```text
error
terminated
error
```

그래서 이번 테스트는 API의 실행 순서를 그대로 기록한다.

```ts
const events: string[] = [];

socket.on('terminated', () => events.push('terminated'));
socket.on('error', () => events.push('error'));

await assert.rejects(socket.connect());

assert.deepEqual(events.slice(-2), ['terminated', 'error']);
```

테스트를 실제 API 순서대로 작성하면 다음을 동시에 확인할 수 있다.

1. 어떤 이벤트가 발생했는가
2. 이벤트가 몇 번 발생했는가
3. 이벤트가 어떤 순서로 발생했는가
4. 이벤트 리스너가 실행 중 상태를 바꿔도 다음 동작이 안전한가

이번 PR에서 테스트가 길어진 이유도 이벤트 이름 하나만 확인하는 것이 아니라 연결 상태 전환이라는 계약을 확인했기 때문이다.

### 5.6 `error` 이벤트가 특별한 이유

Node.js `EventEmitter`에서 `error`는 일반 이벤트와 다르다. 등록된 `error` 리스너가 없으면 `emit('error', error)`가 예외를 던지고 프로세스가 종료될 수 있다.

```ts
const emitter = new EventEmitter();
emitter.emit('error', new Error('failed')); // error listener가 없으면 throw
```

따라서 `node-redis` 사용자는 항상 `error` 리스너를 등록해야 한다.

```ts
const client = createClient();
client.on('error', error => {
  console.error('Redis Client Error', error);
});
```

이번 PR에서도 내부 소켓과 공개 클라이언트 모두 기존 `error` 이벤트를 계속 발생시킨다. `terminated`를 추가했다고 해서 `error`를 대체하면 안 되는 이유가 여기에 있다.

### 5.7 이벤트와 Promise는 서로 다른 제어 흐름이다

`socket.connect()`는 Promise를 반환하지만, 연결 상태 변화는 EventEmitter 이벤트로 전달된다.

```ts
await client.connect();

client.on('error', error => {
  // connect() 이후에도 네트워크 오류가 발생할 수 있다.
});
```

따라서 Promise가 resolve 또는 reject됐다고 해서 이후 이벤트가 더 이상 발생하지 않는 것은 아니다. 이번 PR의 terminal 상황도 다음 두 결과를 동시에 가진다.

```text
이벤트: terminated, error
Promise: connect() reject
```

테스트에서는 Promise의 결과와 이벤트를 각각 확인해야 한다.

```ts
const events: string[] = [];
socket.on('terminated', () => events.push('terminated'));
socket.on('error', () => events.push('error'));

await assert.rejects(socket.connect());
assert.deepEqual(events.slice(-2), ['terminated', 'error']);
```

### 5.8 `isOpen`과 `isReady`는 다른 상태다

두 상태를 하나로 생각하면 이번 문제를 이해하기 어렵다.

```text
isOpen = 연결을 유지하려는 클라이언트 상태
isReady = Redis 명령을 처리할 준비가 끝난 상태
```

가능한 조합은 다음과 같다.

| `isOpen` | `isReady` | 의미 |
|---|---|---|
| `false` | `false` | 닫혔거나 terminal 상태 |
| `true` | `false` | 연결 중, handshake 중, 또는 retry 대기 중 |
| `true` | `true` | 명령 처리 가능한 정상 상태 |
| `false` | `true` | 종료를 시작하는 순간 잠깐 나타날 수 있는 전환 상태 |

`error` 리스너가 `destroy()`를 호출하는 문제는 `isOpen`과 관련된다. `isReady`가 false가 되는 것만으로는 재연결을 막을 수 없다. 사용자가 종료를 요청했는지 확인하려면 `isOpen`을 다시 확인해야 한다.

### 5.9 소켓 오류와 명령 오류의 전달

소켓에서 오류가 발생하면 `RedisClient`의 오류 리스너는 명령 큐를 정리한다.

```text
net.Socket error
  ↓
RedisSocket error
  ↓
RedisClient error
  ↓
RedisCommandsQueue가 대기 중 명령을 reject
```

이때 어떤 오류 객체를 먼저 전달하는지가 중요하다. 실제 네트워크 오류를 먼저 전달해야 대기 중인 명령이 의미 있는 원인을 받는다.

```text
좋은 순서:
실제 socket error
  ↓
명령 큐에 실제 원인 전달
  ↓
strategy error 전달

문제가 있는 순서:
strategy error
  ↓
명령 큐가 strategy error를 받음
  ↓
실제 socket error가 뒤늦게 발생
```

그래서 최종 구현에서는 `ReconnectStrategyResult`가 재연결 시간과 전략 예외를 함께 표현하되, 예외 이벤트를 즉시 발생시키지 않는다. 호출자가 실제 소켓 오류를 먼저 발생시킨 뒤 전략 예외를 발생시킨다.

### 5.10 `Promise.race()`와 소켓 생존 여부

소켓 initiator는 인증이나 초기화 명령을 수행하는 비동기 함수다. initiator가 끝나지 않은 상태에서 소켓이 죽으면 initiator Promise만 기다리는 코드는 영원히 끝나지 않을 수 있다.

```ts
const initiated = this.#initiator();
await initiated; // initiator가 영원히 pending이면 connect도 끝나지 않음
```

그래서 소켓 오류와 initiator를 경쟁시킨다.

```ts
const socketDied = new Promise<never>((_, reject) => {
  socket.once('error', reject);
  socket.once('close', reject);
});

const initiated = Promise.resolve().then(() => this.#initiator());
await Promise.race([initiated, socketDied]);
```

둘 중 먼저 끝나는 쪽이 전체 연결 시도의 결과가 된다.

```text
initiator 완료       -> ready 처리
socket 오류/close    -> 현재 시도 reject -> 재연결 전략 실행
```

이 구조가 있어야 handshake 중 소켓이 끊겨도 재연결 로직으로 돌아갈 수 있다. `finally`에서 소켓 오류 리스너를 제거하는 것도 중요하다. 제거하지 않으면 이미 끝난 연결 시도에 남은 리스너가 나중에 실행될 수 있다.

### 5.11 직접 오류와 재연결 종료 오류의 구분

`reconnectStrategy`가 반환하는 `Error`는 실제 소켓 오류와 다르다.

```text
cause: 실제 TCP/handshake 오류
retryIn: 전략이 반환한 재연결 종료 오류
```

node-redis는 두 오류를 합친 `ReconnectStrategyError`를 `terminated`의 원인으로 전달한다.

```ts
const terminatedBy = new ReconnectStrategyError(retryIn, cause);
this.emit('terminated', terminatedBy);
this.#emitError(cause);
```

사용자는 `terminated`에서 재연결이 끝난 이유를 확인하고, `error`에서는 실제 연결 오류를 확인할 수 있다.

### 5.12 `destroy()`, `close()`, `quit()`의 역할

세 메서드는 모두 연결 종료와 관련 있지만 의미가 다르다.

- `destroy()`: 소켓과 명령 큐를 즉시 정리한다.
- `close()`: 연결을 유지하지 않도록 닫기 흐름을 시작한다.
- `quit()`: Redis에 정상 종료 명령을 보내고 종료한다.

`terminated`는 재연결 전략이 포기했다는 알림이지, 애플리케이션의 모든 리소스를 자동으로 정리했다는 뜻은 아니다. 기존 클라이언트를 교체해야 한다면 `destroy()`가 필요하지만, companion `error` 처리보다 먼저 호출하면 안 된다.

### 5.13 이번 PR을 통해 정리한 핵심 개념

```text
EventEmitter.emit()은 동기적이다.
error 이벤트는 Node.js에서 특별하다.
on()은 반복 이벤트, once()는 한 번의 이벤트에 사용한다.
Promise 완료와 EventEmitter 이벤트 발생은 별개의 흐름이다.
Redis 명령 큐와 Node.js microtask queue는 서로 다르다.
isOpen은 재연결 의지, isReady는 명령 처리 준비 상태다.
재연결 전략 오류와 실제 소켓 오류는 서로 다른 원인이다.
이벤트 리스너 실행 후에는 상태를 다시 확인해야 한다.
소켓 오류와 initiator는 Promise.race()로 경쟁시켜야 hang을 막을 수 있다.
```

---

## 6. 리뷰 과정에서 발견된 추가 문제

처음 수정 후 자동 리뷰를 통해 이벤트 순서만 바꾸는 것으로 끝나지 않는다는 점을 확인했다.

### 6.1 post-ready 오류에서 첫 번째 `error`가 먼저 발생하던 문제

이미 연결이 준비된 뒤 소켓이 끊기는 경로는 `#onSocketError()`에서 시작한다.

초기에는 다음과 같은 구조였다.

```ts
// 기존 흐름
this.emit('error', err);
this.#shouldReconnect(0, err);
```

그러면 `#shouldReconnect()` 내부에서 `terminated`를 먼저 발생시키도록 바꿔도 실제 순서는 다음과 같다.

```text
error          // #onSocketError()
terminated     // #shouldReconnect()
error          // #shouldReconnect()
```

이를 해결하기 위해 `#onSocketError()`에서 ready 상태인 소켓은 먼저 재연결 전략을 확인하도록 바꿨다.

```ts
const { retryIn, strategyError } = this.#shouldReconnect(0, err);

if (typeof retryIn !== 'number') return;

this.#emitError(err);
if (strategyError) this.#emitError(strategyError);
```

이제 terminal이면 `#shouldReconnect()`가 다음 순서로 이벤트를 끝내고:

```text
terminated -> error
```

숫자형 retry이면 기존 동작을 유지한다.

```text
error -> reconnecting -> connect()
```

### 6.2 handshake 오류의 차이

TCP 연결 성공과 Redis 클라이언트 준비 완료는 같은 일이 아니다.

```text
TCP connect
  ↓
인증 및 초기화 명령
  ↓
ready
```

TCP 연결 직후 handshake가 끝나기 전에 소켓이 끊기면 `#isReady`는 `false`다. 이때 `#onSocketError()`가 즉시 `error`를 발생시키면 `#connect()`가 재연결 전략을 판단하기 전에 오류가 노출된다.

최종 수정에서는 `#onSocketError()`가 아직 ready가 아닌 연결의 오류를 열어 둔 채 `#connect()`의 예외 흐름으로 전달하도록 했다.

```ts
if (!wasReady) {
  if (!this.#isOpen) {
    this.#emitError(err);
  }
  return;
}
```

그 뒤 `#connect()`가 전략을 판단한다.

```text
handshake socket error
  ↓
#connect() catch
  ↓
reconnectStrategy
  ↓
terminated -> error       // terminal
또는
error -> reconnecting      // retry
```

### 6.3 `error` 리스너가 동기적으로 종료할 수 있는 문제

이벤트 리스너는 `emit()` 안에서 동기적으로 실행된다.

```ts
socket.on('error', () => {
  socket.destroy();
});
```

재연결을 시도하는 숫자형 전략에서 `error`를 발생시킨 뒤 곧바로 `reconnecting`과 `connect()`를 실행하면, 사용자가 이미 종료한 소켓이 다시 살아날 수 있다.

그래서 `error`를 발생시킨 뒤 상태를 재확인한다.

```ts
this.#emitError(err);

if (!this.#isOpen) return;

this.emit('reconnecting');
this.#connect();
```

이것은 이벤트 기반 코드에서 자주 필요한 패턴이다. 이벤트를 발생시키기 전의 상태만 확인하면 안 되고, 리스너 실행 이후 상태가 바뀌었는지 다시 확인해야 한다.

### 6.4 `terminated`에서 즉시 `destroy()`하면 오류 원인이 바뀌는 문제

`RedisClient`는 `destroy()`할 때 대기 중인 명령을 `DisconnectsClientError`로 reject한다.

```ts
client.on('terminated', () => {
  client.destroy();
});
```

이 리스너가 `terminated` 이벤트 안에서 즉시 실행되면 companion `error`가 처리되기 전에 명령 큐가 닫힐 수 있다.

```text
terminated
  ↓
destroy()
  ↓
대기 중 명령이 DisconnectsClientError를 받음
  ↓
error 이벤트 처리
```

그러면 실제 원인이 `Connection timeout`이어도 명령은 `client is closed`에 해당하는 오류를 받게 된다.

문서 예시는 `queueMicrotask()`로 정리 시점을 뒤로 미뤘다.

```ts
client.on('terminated', cause => {
  console.error('client will not reconnect:', cause);
  queueMicrotask(() => client.destroy());
});
```

실행 순서는 다음과 같다.

```text
terminated
  ↓
destroy()를 microtask 큐에 등록
  ↓
companion error 처리
  ↓
대기 중 명령에 실제 연결 오류 전달
  ↓
microtask 실행
  ↓
destroy()
```

`queueMicrotask()`는 현재 동기 실행이 끝난 직후 실행되므로, 불필요하게 긴 지연을 추가하지 않으면서 같은 `emit()` 호출 안의 `error` 처리를 먼저 완료하게 한다.

### 6.5 reconnect strategy 자체가 예외를 던지는 문제

사용자 정의 전략은 잘못된 값을 반환하거나 예외를 던질 수 있다.

```ts
reconnectStrategy() {
  throw new Error('strategy failed');
}
```

기존 구현은 `#createReconnectStrategy()`의 `catch`에서 전략 예외를 즉시 `error`로 발생시켰다. 그런데 post-ready 경로에서 전략을 소켓 오류보다 먼저 실행하도록 바꾸면 다음 문제가 생긴다.

```text
strategy failed error
  ↓
명령 큐가 strategy failed로 reject
  ↓
실제 socket error 처리
```

실제 연결 오류가 사라지는 셈이다.

최종 구현은 전략 결과와 전략 예외를 분리해서 반환한다.

```ts
type ReconnectStrategyResult = {
  retryIn: false | Error | number;
  error?: Error;
};
```

전략이 예외를 던지면 즉시 이벤트를 발생시키지 않고 결과에 보관한다.

```ts
catch (err) {
  return {
    retryIn: this.defaultReconnectStrategy(retries, err),
    error: err as Error
  };
}
```

호출자는 먼저 실제 소켓 오류를 전달하고, 그 다음 전략 예외를 전달한다.

```ts
const { retryIn, strategyError } = this.#shouldReconnect(0, err);

this.#emitError(err);
if (strategyError) this.#emitError(strategyError);
```

이제 대기 중인 명령은 실제 연결 오류를 먼저 받는다.

```text
socket error
  ↓
명령 큐에 실제 socket error 전달
  ↓
strategy error 전달
```

### 6.6 handshake 직후 소켓 종료에서 `error`가 누락될 수 있던 문제

initiator가 끝난 직후 소켓 상태를 확인하는 별도 분기가 있다.

```ts
if (!this.#socket || this.#socket.destroyed || ...) {
  const retryIn = this.#shouldReconnect(...);
  if (typeof retryIn !== 'number') throw retryIn;
  await setTimeout(retryIn);
  this.emit('reconnecting');
  continue;
}
```

이 경로에는 retry일 때 `error`를 발생시키는 코드가 없었다. 그래서 소켓이 죽었는데도 `reconnecting`만 발생할 수 있었다.

최종 구현은 재연결 판단 후 retry인 경우에도 오류를 명시적으로 전달한다.

```ts
const error = new SocketClosedUnexpectedlyError();
const { retryIn, strategyError } = this.#shouldReconnect(retries++, error);

if (typeof retryIn !== 'number') throw retryIn;

this.#emitError(error);
if (strategyError) this.#emitError(strategyError);
```

---

## 7. 최종 이벤트 흐름

### 7.1 정상적인 재시도

```text
소켓 오류
  ↓
reconnectStrategy -> 숫자
  ↓
error
  ↓
reconnecting
  ↓
connect
  ↓
ready
```

### 7.2 ready 이후 terminal 종료

```text
소켓 오류
  ↓
reconnectStrategy -> false 또는 Error
  ↓
terminated
  ↓
error
  ↓
connect() reject
```

### 7.3 handshake 중 retry

```text
TCP connect 성공
  ↓
initiator 또는 handshake 중 소켓 종료
  ↓
reconnectStrategy -> 숫자
  ↓
error
  ↓
reconnecting
  ↓
재연결
```

### 7.4 handshake 중 terminal 종료

```text
TCP connect 성공
  ↓
initiator 또는 handshake 중 소켓 종료
  ↓
reconnectStrategy -> false 또는 Error
  ↓
terminated
  ↓
error
  ↓
connect() reject
```

### 7.5 사용자가 오류 리스너에서 종료

```text
error
  ↓
사용자 리스너가 destroy()
  ↓
#isOpen = false
  ↓
재연결하지 않음
```

---

## 8. 테스트 설계

### 8.1 초기 연결에서 `false`

연결할 수 없는 주소와 `reconnectStrategy: false`를 사용해 최종 종료를 만든다.

검증 내용:

- `terminated`가 발생하는가
- 원인이 `Error`인가
- 마지막 이벤트 순서가 `terminated -> error`인가
- 소켓이 열려 있지 않은가

### 8.2 사용자 정의 전략이 `Error` 반환

첫 번째 시도에서는 재시도하고, 다음 시도에서는 `Error`를 반환하도록 구성한다.

검증 내용:

- `terminated` 원인이 `ReconnectStrategyError`인가
- 이전 retry에서 발생한 `error`와 terminal 시점의 이벤트를 구분하는가
- 마지막 두 이벤트가 `terminated -> error`인가

### 8.3 ready 이후 연결 종료

로컬 TCP 서버에 연결한 뒤 서버 소켓을 끊는다.

검증 내용:

- ready 이후에도 `terminated`가 발생하는가
- `terminated`가 실제 `error`와 같은 원인을 전달하는가
- 이벤트 순서가 `terminated -> error`인가

### 8.4 retry 중에는 `terminated`가 발생하지 않음

`reconnectStrategy: 0`으로 계속 재시도하게 만든다.

검증 내용:

- `error`는 발생하는가
- `reconnecting`이 진행되는가
- `terminated`는 발생하지 않는가

### 8.5 오류 리스너에서 destroy

ready 이후 연결을 끊고 `error` 리스너에서 `destroy()`를 호출한다.

검증 내용:

- `end`가 발생하는가
- 추가 TCP 연결이 생성되지 않는가
- `isOpen`이 `false`인가

이 테스트가 없으면 `error` 리스너가 상태를 바꾼 뒤에도 `reconnecting`과 `connect()`가 실행되는 회귀를 놓칠 수 있다.

### 8.6 handshake 실패

TCP 연결은 성공시키고 initiator를 끝나지 않게 만든 뒤 서버에서 연결을 끊는다.

검증 내용:

- ready 전 오류에서도 `terminated`가 발생하는가
- `error`가 `terminated`보다 먼저 발생하지 않는가
- 최종 이벤트 순서가 `terminated -> error`인가

---

## 9. 문서화와 공개 API

### 9.1 README 이벤트 표

공개 이벤트 목록에 다음 항목을 추가했다.

```text
terminated | reconnectStrategy가 false 또는 Error를 반환해 재연결이 중단됨 | (cause: Error)
```

루트 `README.md`와 `packages/redis/README.md`를 모두 수정했다. 두 표 중 하나만 수정하면 사용자가 어떤 문서를 읽느냐에 따라 API 설명이 달라지므로 양쪽을 맞췄다.

### 9.2 연결 설정 문서

`docs/client-configuration.md`에 다음 내용을 추가했다.

- `error`는 retry 중에도 발생한다.
- `terminated`는 재연결이 영구적으로 끝났다는 신호다.
- `terminated`가 `error`보다 먼저 발생한다.
- 기존 클라이언트를 교체할 때는 리소스 정리를 위해 `destroy()`가 필요하다.
- `destroy()`는 companion `error` 처리 이후 실행되도록 `queueMicrotask()`로 지연한다.

### 9.3 `destroy()`와 리소스 정리

`terminated`는 재연결 종료를 알리는 이벤트이지, 클라이언트 전체를 자동으로 정리하는 이벤트는 아니다. `RedisClient.destroy()`는 다음과 같은 클라이언트 레벨 리소스를 정리한다.

- 명령 큐
- credentials subscription
- metrics 또는 registry 등록
- 내부 소켓

따라서 애플리케이션이 종료된 클라이언트를 교체할 때는 `destroy()`를 호출해야 한다. 다만 이벤트 리스너 안에서 즉시 호출하면 오류 원인이 바뀔 수 있으므로 문서 예시는 microtask를 사용한다.

---

## 10. 검증 결과

통과한 검증:

- `terminated` 관련 테스트 6개 통과
- 재연결 전략 및 initiator interruption 테스트 10개 통과
- `@redis/client` 타입 검사 통과
- 변경 파일 lint 통과
- `git diff --check` 통과
- 공개 `RedisClient`의 `terminated` 전달 확인

패키지 전체 테스트는 Redis Docker 컨테이너를 시작하는 단계까지 진행됐으나, 로컬 Docker 테스트 환경에서 완료되지 않았다. 따라서 직접 변경한 소켓 테스트, 타입 검사, lint를 별도로 통과시켰다.

PR CI는 최신 커밋 기준으로 성공했다.

---

## 11. 배운 점

### 11.1 이벤트 이름보다 상태 계약이 중요하다

`terminated`라는 이벤트를 추가하는 것 자체는 작은 변경처럼 보이지만, 중요한 것은 언제 발생하느냐였다. 초기 연결 실패, handshake 실패, ready 이후 오류, retry 중 오류가 서로 다른 코드 경로를 지나므로 모든 경로의 이벤트 순서를 확인해야 했다.

### 11.2 이벤트 리스너는 동기적으로 상태를 바꿀 수 있다

`emit('error')` 다음 줄이 항상 안전하게 실행된다고 가정하면 안 된다. 리스너가 그 안에서 `destroy()`, `close()`, 상태 변경을 실행할 수 있으므로 이벤트 발생 후 `isOpen`을 다시 확인해야 한다.

### 11.3 소켓 오류와 전략 오류는 다른 원인이다

연결이 끊긴 원인과 “더 이상 재연결하지 않겠다”는 전략의 결정은 서로 다르다. 두 오류를 하나로 덮어쓰면 명령 큐와 사용자 로그에 잘못된 원인이 전달된다. 내부 결과에서 둘을 분리하고, 사용자에게 전달하는 순서를 명확히 해야 한다.

### 11.4 비동기 Promise API도 EventEmitter 순서를 그대로 해결하지 않는다

`events.once()`는 편리하지만 `error` 이벤트를 특별하게 처리한다. 이벤트를 Promise로 기다릴 때는 일반 이벤트와 `error`의 상호작용을 확인해야 하며, 필요한 경우 직접 리스너를 등록해 순서를 검증하는 것이 더 정확하다.

### 11.5 테스트는 코드의 현재 동작이 아니라 계약을 설명해야 한다

이번 테스트는 단순히 `terminated`가 한 번 호출되는지만 확인하지 않는다.

- retry 중에는 호출되지 않아야 한다.
- terminal이면 호출되어야 한다.
- cause가 올바른 오류여야 한다.
- `error`와의 순서가 보장되어야 한다.
- 리스너가 종료를 요청하면 재연결하지 않아야 한다.

이런 조건이 테스트에 들어가야 다음 리팩터링에서 같은 문제가 다시 생겼을 때 잡을 수 있다.

---

## 12. 현재 상태

- PR #3433: ✅ **MERGED (2026-09-01)**
- 이슈 #2948 연결, closed by merge
- 최신 커밋: `f74c465dae`
- Fork push 완료
- 최신 CI: 성공
- 우리가 작성한 PR 진행 코멘트는 정리하고 최종 요약 코멘트 1개만 유지
- **2026-08-31 메인테이너 피드백**: 구현과 이벤트 순서 대응에 감사를 표하며, `terminated`가 companion `error`보다 먼저 발생하는 이유를 README 이벤트 표와 클라이언트 설정 문서에 명시해 달라고 요청
- 루트 README와 `packages/redis/README.md`의 `terminated` 행에 `error`보다 먼저 발생한다는 설명 추가
- `docs/client-configuration.md`에 이 순서 덕분에 `events.once(client, 'terminated')`가 원인과 함께 resolve되고, 영구 종료와 일시적 retry를 구분할 수 있다는 설명 추가
- 문서 변경 커밋을 push하고 PR에 대응 완료 댓글 게시
- **2026-09-01**: 메인테이너가 머지. 이슈 #2948이 15개월 만에 해소됨.
