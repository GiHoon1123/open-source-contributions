# [Redis OSS 기여] redis/ioredis - 깊게 중첩된 RESP 응답으로 인한 프로세스 크래시 수정 (PR #2138)

> 오픈소스 기여 기록 및 학습 노트. 이슈 #2108 하나에 대한 재현·원인 분석·수정·검증 과정을 시간순으로 다룬다. **최초 작성 2026-07-14, 2026-07-28에 RESP3 리팩터(#2127)와의 충돌 해결 및 스코프 재조정(코드 수정 → 회귀 테스트만)까지 갱신. 2026-08-06 머지됨(12절 참고).**

---

## 1. 개요

### 1.1 프로젝트 소개

- **저장소**: https://github.com/redis/ioredis — Node.js용 서드파티(비공식) Redis 클라이언트. `redis/node-redis`(공식 클라이언트, 같은 조직이 관리하지만 완전히 별개의 코드베이스)와는 다른 프로젝트다.
- **건드린 부분**: `lib/DataHandler.ts` — 소켓에서 들어오는 raw 데이터를 받아 파서에 넘기고, 파싱된 응답을 대기 중인 명령과 매칭시키는 클래스.

### 1.2 이슈 (#2108)

**제목**: "Deeply nested RESP aggregates can trigger uncaught RangeError and crash the process"

리포터의 시나리오: RESP 프로토콜의 배열(Array) 타입은 중첩이 가능한데(배열 안에 배열), ioredis가 의존하는 `redis-parser` 패키지가 이 중첩을 파싱할 때 재귀 함수 호출로 처리하며 깊이 제한이 없다. 충분히 깊게 중첩된 응답(악의적이거나 손상된 서버가 보낼 수 있는)이 오면 콜스택이 넘쳐서 `RangeError`가 발생하는데, 이걸 잡아주는 코드가 없어서 프로세스 전체가 죽는다는 리포트. 리포터는 두 가지 수정 방향을 제안했다 — ① `redis-parser` 자체에 깊이 제한 추가, ② ioredis 쪽에서 파서 예외를 잡아서 방어. 이번 PR은 ②만 다룬다.

### 1.3 관련 링크

| 구분        | 링크                                              | 상태     |
| ----------- | ------------------------------------------------- | -------- |
| 원본 저장소 | https://github.com/redis/ioredis                  | —        |
| 내 Fork     | https://github.com/GiHoon1123/ioredis              | —        |
| 이슈 (#2108) | https://github.com/redis/ioredis/issues/2108      | Open     |
| **PR (#2138)** | https://github.com/redis/ioredis/pull/2138     | **Merged** (2026-07-14 제출, 2026-08-06 머지) |
| 작업 브랜치 | `fix-2108-parser-recursion-crash`                  | —        |
| 커밋        | `ac38e15`                                          | —        |

### 1.4 한 줄 요약

> `redis-parser`가 RESP 배열 중첩을 깊이 제한 없이 재귀로 파싱하다가 던지는 `RangeError`가, `DataHandler`의 소켓 `"data"` 리스너에 `try/catch`가 없어서 잡히지 않고 프로세스를 그대로 죽였다. `parser.execute()` 호출을 `try/catch`로 감싸고, 이미 존재하던 `returnFatalError → recoverFromFatalError` 복구 경로(파서가 스스로 감지한 프로토콜 오류를 처리하던 것과 동일한 경로)로 흘려보내도록 수정했다.

---

## 2. 배경 지식

### 2.1 RESP와 `redis-parser`

RESP(REdis Serialization Protocol)는 타입을 나타내는 첫 바이트로 시작하는 텍스트 기반 프로토콜이다. `*3\r\n...`은 원소 3개짜리 배열을 뜻하고, 그 원소 자리에 또 배열이 올 수 있어 중첩이 가능하다.

`redis-parser`는 ioredis의 **외부 npm 의존성**(`package.json`에 버전 고정, 별도 저장소)으로, 이 바이트 스트림을 JS 값으로 변환하는 역할을 한다. 배열 중첩은 `parseArray → parseArrayElements → parseType → parseArray → ...` 식으로, 중첩 1단계마다 함수 호출 1번을 쓰는 재귀로 구현돼 있다. **이 재귀에 깊이 제한이 없다.**

### 2.2 `DataHandler`와 소켓 `"data"` 이벤트

`lib/DataHandler.ts`는 생성자에서 이렇게 소켓에 리스너를 건다 (수정 전):

```ts
redis.stream.prependListener("data", (data) => {
  parser.execute(data); // try/catch 없음
});
redis.stream.resume();
```

`grep -rln "new DataHandler"`로 확인한 결과 `DataHandler`는 `lib/redis/event_handler.ts` **딱 한 곳**에서만 인스턴스화된다 — 즉 이 수정은 standalone/Sentinel/Cluster 등 모든 연결 토폴로지에 동일하게 적용된다.

### 2.3 이미 있던 복구 경로 — `returnFatalError` / `recoverFromFatalError`

`redis-parser`가 스스로 "이건 프로토콜 규격에 안 맞는다"고 감지할 수 있는 경우(예: 알 수 없는 타입 바이트)엔, 파서가 그 사실을 콜백으로 알려주고 ioredis는 `returnFatalError`를 거쳐 `recoverFromFatalError`를 호출한다:

```ts
// lib/Redis.ts
recoverFromFatalError(_commandError: Error, err: Error, options: FlushQueueOptions) {
  this.flushQueue(err, options);
  this.silentEmit("error", err);
  this.disconnect(true);
}
```

이건 새로 만든 게 아니라 **원래부터 있던, 이미 검증되고 쓰이던 정상 장애 복구 경로**다. 문제는 `RangeError`(스택 오버플로우)가 `redis-parser`가 "스스로 감지한 에러"가 아니라 **JS 엔진이 직접 던지는 예외**라서, 이 fatal-error 콜백 경로를 타지 않고 그냥 예외로 밖으로 새어나간다는 것 — 이미 만들어둔 안전망에 구멍이 하나 있었던 셈이다.

---

## 3. 왜 프로세스가 죽는가 — Node.js 이벤트 루프/콜스택 모델

이 이슈를 정확히 이해하려면 "예외가 안 잡히면 왜 프로세스 전체가 죽는가"를 코드가 아니라 런타임 모델 수준에서 봐야 한다.

### 3.1 콜스택과 재귀

JS는 싱글 스레드고, 함수 호출은 콜스택에 프레임을 쌓으며 실행된다. 재귀 함수가 자기 자신을 계속 호출하면 그때마다 콜스택에 프레임이 쌓이는데, 이 스택 크기는 유한하다. 중첩이 끝없이 깊어지면 스택이 꽉 차서 `RangeError: Maximum call stack size exceeded`가 발생한다.

### 3.2 소켓 데이터는 어떻게 콜백으로 들어오는가 — libuv

Node의 네트워크 I/O(소켓)는 전부 **libuv** 위에서 동작한다. libuv는 OS 커널이 제공하는 비동기 I/O API(리눅스 epoll, macOS/BSD kqueue, Windows IOCP)를 감싸서 통일된 인터페이스로 제공하는 라이브러리다 — 별도 스레드 없이, OS가 "이 소켓에 읽을 데이터가 생겼다"고 알려주면 그 신호를 받아 JS 콜백을 호출해주는 방식으로 동작한다. (반대로 파일시스템처럼 OS가 좋은 비동기 API를 안 주는 경우엔 libuv가 자체 스레드풀에 블로킹 호출을 떠넘겨서 흉내낸다 — 소켓은 이 경로를 타지 않는다.)

Redis 서버가 응답을 보내면: OS가 소켓 읽기 버퍼에 데이터를 채움 → libuv가 이를 감지해 실제로 읽어옴 → **이벤트 루프의 poll phase 큐에 완료된 콜백을 넣음** → 이벤트 루프가 poll phase 차례가 되고 콜스택이 비어있을 때 그 콜백을 꺼내 콜스택에 올려 실행 — 이게 `DataHandler`의 `"data"` 리스너가 호출되는 순간이다.

(참고로 이 poll phase 큐는 Promise 콜백이 쌓이는 마이크로태스크 큐와는 다른 큐다. 이벤트 루프는 콜스택이 빌 때마다 먼저 마이크로태스크 큐를 전부 비우고, 그 다음 현재 phase의 큐에서 다음 작업을 꺼내는 식으로 순환한다.)

### 3.3 예외의 전파(unwind)와 "이벤트 루프까지 튀어오른다"의 의미

`"data"` 리스너가 호출되면 콜스택은 이렇게 쌓인다:

```
[libuv가 이 콜백을 부른 지점]
  → EventEmitter.emit("data", ...)
    → DataHandler의 "data" 리스너
      → parser.execute(data)
        → parseArray → parseArrayElements → parseType → parseArray → ... (중첩 깊이만큼 반복)
```

`RangeError`는 이 재귀의 밑바닥에서 발생한다. JS의 예외 전파는 "예외가 발생하면 현재 함수를 빠져나가 자신을 호출한 함수로 올라가고, 거기 `catch`가 있으면 잡히고 없으면 계속 위로"를 반복하는 것이다. 수정 전에는 이 목록 어디에도 `try/catch`가 없었다 — 파서 재귀에도, `"data"` 리스너에도. 그래서 예외는 재귀를 다 빠져나오고, 리스너를 빠져나오고, `emit()` 내부를 빠져나와서 결국 libuv가 이 콜백을 호출했던 지점까지 되감긴다 — 더 이상 올라갈 JS 프레임이 없는 지점, 즉 이 하나의 I/O 이벤트를 처리하려고 이벤트 루프가 진입했던 최상단에 도달한다.

### 3.4 `process.on('uncaughtException')`가 잡는 이유

Node 런타임은 이 최상단 지점에 자체 안전장치를 심어놓는다. 예외가 여기까지 되감겨 올라오면(아무도 못 잡았으면), Node는 전역 `process` 객체(이것도 하나의 EventEmitter)에 `"uncaughtException"`을 emit한다. 리스너가 없으면 Node의 기본 동작(스택 트레이스 출력 + `process.exit(1)`)이 실행되고, 리스너가 있으면 그 함수가 대신 호출되어 프로세스가 즉시 죽지는 않는다. 다만 이건 진짜 복구가 아니라 최후의 방어선 — 이 시점엔 이미 어떤 자원이 정리되다 만 상태일 수 있다.

**정리**: 이 이슈는 "파서 버그"이기 이전에 "예외가 콜스택을 다 빠져나가 이벤트 루프 최상단까지 도달했는데, 그 경로 어디에도 안전망이 없었던" 문제다. 수정의 본질은 이 unwind가 `"data"` 리스너 지점에서 멈추도록 안전망 하나를 추가하는 것.

---

## 4. 재현

이슈 리포트를 그대로 믿지 않고 직접 검증했다.

1. **파서 단독 검증**: `redis-parser`를 직접 호출해 깊게 중첩된 버퍼(`*1\r\n`을 N번 반복 + `:1\r\n`)를 먹여봄 → 실제로 `RangeError` 발생 확인. 다만 터지는 깊이는 로컬 환경에서 대략 3000~5000 사이 — 이슈 리포트의 "깊이 10000" 과 정확히 일치하진 않았다. 콜스택 오버플로우 깊이는 OS/Node 버전/스택 크기 설정에 따라 달라지는 환경 의존 값이라, 정확한 숫자보다 "현상 자체가 실제로 벌어지는가"가 중요했고 그건 확인됐다.
2. **엔드투엔드 재현**: 가짜 악의적 TCP 서버를 만들고, 진짜 `Redis` 클라이언트를 연결시킨 뒤 서버가 깊게 중첩된 배열 응답을 보내게 함 → `RangeError`가 **uncaught exception**으로 튀어나오며 프로세스가 죽는 것을 확인 (테스트 하네스에 명시적으로 `process.on('uncaughtException')`을 걸어놔야만 겨우 잡혔음 — 즉 실전 애플리케이션이었으면 그냥 죽었을 것).

---

## 5. 수정

```diff
  redis.stream.prependListener("data", (data) => {
+   try {
      parser.execute(data);
+   } catch (err) {
+     this.returnFatalError(
+       new Error(
+         `Unhandled error while parsing the Redis reply: ${(err as Error).message}`
+       )
+     );
+   }
  });
```

핵심 판단: **새 복구 메커니즘을 발명하지 않고, 이미 있던 `returnFatalError → recoverFromFatalError` 경로에 태워보냄.** 이미 프로덕션에서 검증되고 쓰이던 안전한 경로를 재사용하는 것이라 리스크가 작고, 변경 범위도 최소화된다.

스코프를 명확히 구분: 이건 **`redis-parser` 자체(별도 외부 패키지)에 깊이 제한을 추가하는 게 아니다** — 그건 다른 저장소의 일이다. 이 PR은 이슈가 제안한 두 방향 중 ioredis 쪽 방어(containment)만 다룬다.

### 5.1 `recoverFromFatalError`가 실제로 복구하는 방식

```ts
recoverFromFatalError(_commandError: Error, err: Error, options: FlushQueueOptions) {
  this.flushQueue(err, options);   // ① 큐 정리
  this.silentEmit("error", err);   // ② 안전하게 에러 이벤트 발행
  this.disconnect(true);           // ③ 재연결 트리거
}
```

**① `flushQueue(err, { offlineQueue: false })`** — ioredis는 명령 큐를 두 개로 나눠 관리한다: `offlineQueue`(사용자가 호출했지만 아직 소켓으로 나가지 않고 연결 준비를 기다리는 명령들)와 `commandQueue`(이미 소켓으로 나갔고 응답만 기다리는 명령들). `{ offlineQueue: false }`를 넘기는 이유:
- `offlineQueue`는 건드리지 않고 보존 — 아직 안 보낸 명령이니 재연결되면 그대로 재전송하면 됨
- `commandQueue`(기본값)는 전부 실패 처리 — 이미 보낸 명령이 서버에서 처리됐는지 알 수 없는 애매한 상태이므로 안전하게 reject

**② `silentEmit("error", err)`** — Node의 `EventEmitter`는 `"error"` 이벤트에 리스너가 하나도 없으면 그 자체로 예외를 던지는 특칙이 있는데, 이걸 피해 안전하게 알림만 보내는 ioredis 자체 헬퍼.

**③ `disconnect(true)`** — `reconnect=true`로 호출되므로 "일부러 끊는 것"이 아니라 "재연결을 위한 끊기"로 처리됨(`manuallyClosing` 플래그를 세우지 않음). 소켓이 닫히면 `closeHandler`가 `manuallyClosing === false`이고 `retryStrategy`가 설정돼 있는 걸 보고, 계산된 딜레이 후 `connect()`를 다시 호출 → 새 소켓과 새 `DataHandler`가 만들어지고, 보존해둔 `offlineQueue`의 명령들이 자동 재전송된다.

---

## 6. 테스트

```ts
it("recovers instead of crashing on a reply that overflows the parser stack | https://github.com/redis/ioredis/issues/2108", () => {
  let dataListener: (data: Buffer) => void;
  const prependListener = (event, handler) => { dataListener = handler; };
  const recoverFromFatalError = sinon.spy();

  new DataHandler(
    { stream: { prependListener, resume: sinon.spy() }, recoverFromFatalError } as any,
    {} as any
  );

  let deeplyNested = ":1\r\n";
  for (let i = 0; i < 50000; i++) deeplyNested = "*1\r\n" + deeplyNested;

  expect(() => dataListener(Buffer.from(deeplyNested))).to.not.throw();
  expect(recoverFromFatalError.calledOnce).to.be.true;

  const [commandError, err, options] = recoverFromFatalError.firstCall.args;
  expect(commandError).to.be.instanceOf(Error);
  expect(err).to.be.instanceOf(Error);
  expect(commandError.message).to.include("Unhandled error while parsing the Redis reply");
  expect(options).to.deep.equal({ offlineQueue: false });
});
```

**Disable-and-reconfirm**: fix 코드만 잠깐 빼고 (`git stash push -- lib/DataHandler.ts`) 같은 테스트를 재실행 → 원래 `RangeError`가 uncaught로 그대로 던져지며 테스트가 정확히 실패하는 것 확인 — 이 테스트가 실제로 regression을 잡아낼 수 있다는 것까지 검증.

---

## 7. 외부 리뷰 4가지 포인트 검증

작성 중 pasted external review 형태로 4가지를 점검했다.

**① try/catch가 실제로 동작하는가** — 4장의 엔드투엔드 재현으로 확인 완료.

**② 테스트가 `recoverFromFatalError` 호출 인자를 충분히 검증하는가** — 처음엔 첫 번째 인자의 메시지만 확인했는데, `calledOnce` + 두 인자 모두 `Error` 인스턴스인지 + 세 번째 인자가 정확히 `{ offlineQueue: false }`인지까지 검증하도록 강화함 (6장 코드).

**③ 무거운 오토파이프라이닝 테스트가 타임아웃 나는 것 아니냐** — 처음엔 전체 스위트 1회 클린 통과(625 passing / 135 pending / 0 failing)로 넘어가려 했으나, "1회 통과가 타임아웃 미재현을 증명하지 못한다"는 지적을 받고 재검증: 스위트에서 가장 무거운 `"should handle large pipelines without RangeError"`(1MB×600개 동시 SET/GET) 테스트를 fix 적용/미적용 각각 5회씩 반복 측정 — 965ms~2.27s vs 1.58~1.66s로 사실상 동일 범위, mocha 타임아웃(8000ms) 대비 5배 이상 여유. 전체 `autopipelining.ts` 파일도 3회 반복 클린 통과. fix가 이 테스트 타이밍에 실질적 영향이 없다는 것을 반복 측정으로 확인.

**④ 넓은 catch 범위가 무관한 예외(예: 사용자 pub/sub 리스너가 던지는 예외)까지 삼키는 것 아니냐** — 실제로 그런 경우가 이 catch에 걸리는 건 맞음. 하지만 `git stash`로 A/B 비교한 결과, **fix 이전에도 그 상황은 프로세스를 그대로 죽였음** — 새로 생긴 회귀가 아니라 원래 있던 특성이고, fix가 그 경우까지 우연히 복구시켜주는 부수 효과일 뿐. 다만 에러 메시지("Unhandled error while parsing the Redis reply")가 이 경우엔 다소 부정확한 표현이라는 점은 남아있는 사소한 개선 여지.

---

## 8. PR #2138 본문에 담은 내용

1. 근본 원인 (파서의 무제한 재귀 → RangeError → uncaught exception)
2. 수정 방식과 기존 `returnFatalError` 경로 재사용 근거
3. 스코프 한정 — `redis-parser` 자체 depth-limit은 별도 이슈/저장소 몫
4. 엔드투엔드 재현 및 fix 적용 후 재검증 결과
5. 테스트 상세 및 disable-and-reconfirm 결과
6. 전체 로컬 스위트 결과와 무관한 사전 존재 flaky 테스트 1건 명시
7. 체크리스트에 "Maintainer review" 항목만 미완료로 정직하게 표기

---

## 9. 결과 및 회고

### 9.1 현재 상태 (2026-07-14 기준, 1차 제출 시점)

- PR #2138 **Open** (2026-07-14 제출), 메인테이너 리뷰 대기
- **이 상태는 2026-07-28에 크게 바뀐다 — 11절 참고.**

### 9.2 핵심 교훈

1. **AI가 작성한 것으로 보이는 이슈 리포트도 독립적으로 재검증해야 한다.** "Reproduction summary", 구체적인 깊이 숫자(10000) 같은 형식을 갖췄지만, 실제 재현 깊이는 환경에 따라 3000~5000으로 달랐다 — 리포트의 정확한 수치를 그대로 믿지 않고 직접 재현해서 현상이 실제로 일어나는지를 확인하는 것이 중요했다.
2. **이미 있는 복구 메커니즘을 재사용하는 것이 새로 만드는 것보다 안전하다.** `recoverFromFatalError`는 이미 프로덕션에서 검증된 경로였고, 이번 fix는 그 경로로 들어가는 입구를 하나 더 추가한 것뿐이라 리스크가 작다.
3. **"1회 클린 통과"는 타임아웃/플레키니스 주장을 반증하기에 충분하지 않다.** 반복 측정(5회) + 대조군(fix 없는 상태) 비교까지 해야 "영향 없음"을 신뢰성 있게 주장할 수 있다.
4. **넓은 예외 처리 범위에 대한 지적은 "새 회귀인가 기존 특성인가"를 먼저 구분해야 한다.** `git stash`를 이용한 A/B 비교로, 지적된 부작용이 fix가 만든 새로운 문제가 아니라 원래도 있던 특성이라는 것을 확인할 수 있었다.
5. **콜스택 unwind → 이벤트 루프 → uncaughtException으로 이어지는 런타임 모델을 이해하면, "왜 하필 이 위치에 try/catch가 없어서 전체가 죽는가"를 코드 레벨을 넘어 설명할 수 있다.** 이런 이해는 유사한 "특정 콜백에 안전망이 없어서 생기는 크래시" 부류의 버그를 찾을 때도 재사용 가능한 사고방식이다.

### 9.3 남은 항목 (1차 제출 시점)

- [x] 메인테이너 리뷰 — 2026-07-28, 다만 리뷰라기보다 base 변경으로 인한 충돌 발견. 11절 참고
- [ ] 머지 여부

---

## 10. 개념 정리 (참고용)

### 10.1 이슈 관련 개념

- **RESP 배열 중첩**: `*N\r\n` 뒤에 N개의 원소가 오고, 그 원소 자리에 또 배열이 올 수 있다. `redis-parser`는 이 중첩을 재귀로 파싱하며 깊이 제한이 없다.
- **재귀와 콜스택**: 함수 호출마다 콜스택에 프레임이 쌓이고, 이 스택 크기는 유한하다. 무제한 재귀는 결국 `RangeError: Maximum call stack size exceeded`로 이어진다.
- **fatal error 복구 경로 (`returnFatalError` → `recoverFromFatalError`)**: 파서가 스스로 감지한 프로토콜 오류를 처리하던, 원래부터 있던 경로. 대기 중인 명령을 정리하고, 안전하게 에러를 알리고, 재연결을 트리거한다.
- **`offlineQueue` vs `commandQueue`**: 전자는 아직 소켓으로 나가지 않은(전송 대기 중인) 명령, 후자는 이미 전송했고 응답을 기다리는 명령. 연결 오류 시 전자는 보존(재전송 가능)하고 후자는 실패 처리하는 것이 안전하다(서버가 처리했는지 알 수 없는 애매한 상태이므로).

### 10.2 Node.js 런타임 core 개념

- **콜스택(call stack)**: 현재 실행 중인 함수 호출들이 쌓이는 곳. 재귀 함수는 호출할 때마다 프레임을 쌓는다.
- **libuv**: Node의 크로스플랫폼 비동기 I/O 라이브러리. OS 커널의 비동기 API(리눅스 epoll, macOS/BSD kqueue, Windows IOCP)를 통일된 인터페이스로 감싼다. 하드웨어와 직접 통신하는 게 아니라, OS 커널이 제공하는 시스템콜을 감싸는 계층이다.
  - **네트워크 소켓**: OS가 진짜 비동기 API(epoll/kqueue)를 제공하므로, libuv는 별도 스레드 없이 OS의 이벤트 알림을 받아 JS 콜백을 호출한다.
  - **파일시스템/DNS 조회 등**: OS가 좋은 비동기 API를 안 주는 경우, libuv가 자체 스레드풀(기본 4개, `UV_THREADPOOL_SIZE`)에 블로킹 호출을 떠넘겨 비동기처럼 동작하게 만든다.
- **이벤트 루프(event loop)**: 큐가 아니라, 콜스택이 비었는지 계속 확인하면서 정해진 phase(timers → pending callbacks → poll → check → close callbacks)를 순환하며 각 phase의 큐에서 콜백을 꺼내 콜스택에 올려 실행하는 스케줄러. 소켓의 `"data"` 콜백은 **poll phase**의 큐에서 나온다.
- **마이크로태스크 큐 vs 매크로태스크 큐**: Promise 콜백(`.then/catch/finally`), `queueMicrotask`, (Node의 `process.nextTick` 전용 큐)은 마이크로태스크 큐에 쌓이고, 콜스택이 빌 때마다(하나의 콜백 실행이 끝날 때마다) 매크로태스크보다 먼저, 완전히 비워진다. 반면 소켓 I/O 완료 콜백, 타이머 콜백 등은 각 phase 전용 매크로태스크 큐에 쌓이며, 이벤트 루프가 해당 phase 차례일 때 꺼내 실행한다. 둘은 서로 다른 큐이고 처리 순서 규칙도 다르다.
- **예외 전파(unwind)**: 예외가 발생하면 현재 함수를 빠져나가 호출자로 올라가고, 그 과정에서 `catch`를 만나면 잡히고, 아무 데도 없으면 콜스택 전체를 다 빠져나가 이벤트 루프가 이 콜백을 시작했던 최상단 지점까지 도달한다.
- **`process.on('uncaughtException')`**: 예외가 콜스택 전체를 빠져나가 아무도 못 잡은 상태로 이 최상단 지점에 도달하면, Node는 전역 `process` 객체에 `"uncaughtException"`을 emit한다. 리스너가 없으면 스택 트레이스 출력 후 `process.exit(1)`, 리스너가 있으면 그 함수가 대신 호출되어 즉시 죽지는 않는다 — 다만 이는 정상적인 복구가 아니라 최후의 방어선이다.

---

## 11. RESP3 리팩터와의 충돌, 그리고 스코프 재조정 (2026-07-28)

### 11.1 발견 경위

전체 오픈 PR 목록을 훑어보다가(다른 저장소 코멘트 작업 도중) "리베이스가 필요한 PR이 있는지" 점검하는 과정에서 `mergeable: CONFLICTING`으로 뜨는 걸 발견했다. PR을 올린 시점(2026-07-14)엔 문제없었는데, 그 사이 `main`에 다른 컨트리뷰터의 대형 PR(`#2127 "Add RESP3"`, 2026-07-20 머지)이 먼저 들어오면서 우리가 건드린 `lib/DataHandler.ts`를 완전히 갈아엎었기 때문 — `git log upstream/main --since` 로 확인해서, 우리 잘못이 아니라 base가 앞서나간 것뿐임을 먼저 확인했다.

### 11.2 단순 충돌이 아니었던 이유

`#2127`은 `redis-parser`(외부 npm 의존성)를 완전히 걷어내고, `node-redis`에서 이식해온 자체 `Decoder` 클래스(`lib/resp/decoder.ts`)로 교체했다. 그런데 `DataHandler.ts`의 새 코드를 보니:

```ts
redis.stream.prependListener("data", (data) => {
  try {
    decoder.write(data);
  } catch (err) {
    this.returnFatalError(err as Error);
  }
});
```

이미 `decoder.write()`를 try/catch로 감싸고 `returnFatalError → recoverFromFatalError`로 보내고 있었다 — 우리가 짠 것과 구조적으로 동일한 보호 장치였다. 그냥 리베이스로 기계적으로 충돌만 풀면 "이미 있는 걸 다시 넣는" 중복이 될 위험이 있어서, 진행 전에 세 가지를 확인했다.

### 11.3 확인 ① — 새 `Decoder`도 같은 취약점이 있는가

말로 판단하지 않고 직접 실험했다. `git worktree add`로 `upstream/main`을 별도 폴더에 체크아웃하고, `ts-node`로 새 `Decoder`를 직접 인스턴스화해서 원본 재현 코드와 동일한 5만 단계 중첩 배열(`*1\r\n` 5만 번 감싼 `:1\r\n`)을 먹여봤다.

```
THREW: RangeError Maximum call stack size exceeded
```

새 `Decoder`도 `#decodeArrayItems → #decodeNestedType → #decodeNestedTypeValue → #decodeArray → ...` 형태로 중첩 1단계당 함수 호출 1번을 쓰는 재귀 구조를 그대로 갖고 있었다(코드 레벨로 `#decodeArrayItems`, `#decodeNestedType` 정의를 직접 확인). 즉 **근본 취약점(무제한 재귀 깊이)은 파서를 통째로 갈아엎은 뒤에도 그대로 남아있다.**

### 11.4 확인 ② — 우리 PR이 "회귀 테스트 추가"로 축소되는 게 맞는가

`returnFatalError`의 구현을 다시 확인:

```ts
private returnFatalError(err: Error) {
  err.message += ". Please report this.";
  this.redis.recoverFromFatalError(err, err, { offlineQueue: false });
}
```

우리가 짠 복구 로직과 완전히 동일한 경로다. `main`의 `test/unit/DataHandler.ts` 전체를 훑어봤는데 "깊게 중첩된 응답이 프로세스를 안 죽인다"는 케이스를 검증하는 테스트는 하나도 없었다. **결론: 코드 수정은 이제 중복이라 버리고, 테스트만 새 `Decoder` 기반 코드 위에 다시 얹는 게 맞다.**

### 11.5 확인 ③ — RESP3 PR로 이슈 #2108이 자동으로 닫히는가

GitHub GraphQL로 이슈 #2108의 타임라인을 조회 — `state: OPEN`, `#2127`의 본문 어디에도 `#2108` 언급 없음, 우리 PR `#2138`만 이 이슈를 참조하고 있었다. 자동으로 닫힐 일은 없다는 것을 확인.

### 11.6 리베이스와 재작성

`git rebase upstream/main`을 실행하니 `lib/DataHandler.ts`와 `test/unit/DataHandler.ts` 양쪽에서 충돌이 났다. `lib/DataHandler.ts`는 `--ours`(리베이스 target인 `upstream/main` 쪽)를 그대로 채택 — 우리 diff가 완전히 사라져서, 이 파일은 이제 `upstream/main`과 diff가 0이다. `test/unit/DataHandler.ts`도 `main` 쪽 최신 버전(`setupDataHandler()` 헬퍼 기반의 새 테스트 패턴)을 채택한 뒤, 그 패턴에 맞춰 회귀 테스트를 새로 작성해서 추가했다.

```ts
it("recovers instead of crashing on a reply that overflows the decoder's call stack | https://github.com/redis/ioredis/issues/2108", () => {
  const setup = setupDataHandler();

  let deeplyNested = ":1\r\n";
  for (let i = 0; i < 50000; i++) {
    deeplyNested = "*1\r\n" + deeplyNested;
  }

  expect(() => setup.write(deeplyNested)).to.not.throw();

  expect(setup.redis.recoverFromFatalError.calledOnce).to.be.true;
  const [commandError, err, options] =
    setup.redis.recoverFromFatalError.firstCall.args;
  expect(commandError).to.be.instanceOf(RangeError);
  expect(err).to.equal(commandError);
  expect(commandError.message).to.include("Maximum call stack size exceeded");
  expect(options).to.deep.equal({ offlineQueue: false });
});
```

기존 테스트는 `dataListener`를 직접 호출하는 방식이었는데, `main`의 새 헬퍼 `setupDataHandler()` / `setup.write()` 패턴에 맞춰 다시 작성했다. 커밋 메시지 타입도 `fix:` 대신 `test:`로 바꿨다 — Conventional Commits 기준으로 실제로 바뀐 게 테스트뿐이라서.

### 11.7 disable-and-reconfirm 재검증

리팩터 후 `try/catch` 부분을 임시로 제거하고 같은 테스트를 돌려서, 테스트가 실제로 회귀를 잡아내는지 재확인했다.

```
1) recovers instead of crashing on a reply that overflows the decoder's call stack | https://github.com/redis/ioredis/issues/2108:
     AssertionError: expected [Function] to not throw an error but 'RangeError: Maximum call stack size e…' was thrown
```

정확히 예상한 대로 실패했다. 원상 복구 후 전체 `DataHandler` 스위트 14개 통과, `eslint` 클린.

### 11.8 자체검토 — 무관한 줄까지 건드릴 뻔한 것

`npx prettier --write`를 파일 전체에 돌렸더니, 우리가 손대지 않은 다른 기존 테스트(`resolves uppercase unsubscribe commands`)의 한 줄까지 재포맷됐다. 확인해보니 그 줄은 `upstream/main` 원본부터 이미 prettier 기준을 안 지키고 있던 것(별도로 `/tmp`에 받아서 `prettier --check`로 확인) — 우리 탓이 아니었고, 이번 PR 스코프도 아니라서 되돌려서 diff를 새 테스트 23줄로만 좁혔다.

### 11.9 반영

- 리베이스한 브랜치를 `git push --force-with-lease`로 fork에 강제 푸시
- PR 제목: `test(DataHandler): cover deeply nested reply causing decoder stack overflow`
- PR 본문: "코드 수정은 이제 필요 없고(#2127이 이미 같은 보호 장치를 갖고 있음), 이 동작을 검증하는 테스트가 없어서 추가했다"고 명시 — 실제로 안 한 "버그 수정"을 한 것처럼 쓰지 않도록 정직하게 스코프를 좁혀서 서술
- 확인 결과: `mergeable: MERGEABLE`(충돌 해소됨), `mergeStateStatus: UNSTABLE`(방금 push해서 CI가 막 다시 도는 중, 정상)

### 11.10 핵심 교훈

1. **머지 충돌은 항상 "누가 잘못했나"가 아니라 "base가 무엇을 바꿨나"부터 확인해야 한다.** `git log --since`로 시간순을 먼저 맞춰보는 것만으로 "내 PR이 고장 났다"는 오해를 피할 수 있었다.
2. **다른 PR이 우연히 같은 문제를 해결했을 가능성을, 충돌을 기계적으로 푸는 대신 먼저 검증해야 한다.** 검증 없이 리베이스했으면 이미 있는 보호 장치 위에 중복 코드를 얹을 뻔했다.
3. **"코드 재작성이 문제를 없앴다"와 "근본 원인이 없어졌다"는 다른 질문이다.** `Decoder`로 파서를 통째로 바꿨지만 무제한 재귀라는 근본 설계는 그대로였고, 이건 직접 실행해서 확인하지 않았으면 "리팩터됐으니 아마 괜찮겠지"로 넘어갈 뻔한 지점이었다.
4. **기여의 성격이 바뀌면 커밋 메시지/PR 본문/타입(`fix:` → `test:`)도 정직하게 같이 바꿔야 한다.** 코드 수정이 없어졌는데 "고쳤다"는 표현을 그대로 두면 리뷰어에게 잘못된 인상을 준다.
5. **prettier처럼 파일 전체를 건드리는 도구는 우리 스코프 밖의 줄까지 같이 고칠 수 있다.** 적용 전 diff를 diff 단위로 확인해서, 우리 변경과 무관한 재포맷은 되돌리는 습관이 필요하다.

---

## 12. 머지 (2026-08-06)

메인테이너(`PavelPashov`)가 처음엔 "#2127도 같이 확인해보라"고만 했고, 이후 별다른 코멘트 없이 조용히 진행되다가 2026-08-06에 그대로 머지됨(`dfd5428`). 리뷰 코멘트가 따로 남지 않아서, 회귀 테스트로 스코프를 좁힌 판단(11절)이 그대로 받아들여진 것으로 보인다.

- 최종 diff: `test/unit/DataHandler.ts` 한 파일, 회귀 테스트만
- 최종 커밋 메시지: `test: add regression coverage for deeply nested reply causing a decoder stack overflow`
