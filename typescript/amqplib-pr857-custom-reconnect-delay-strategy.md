# [OSS 기여] amqp-node/amqplib - 커스텀 재연결 지연 전략(`calculateDelay`) 옵션 추가 (PR #857)

> 오픈소스 기여 기록 및 학습 노트. 이 문서는 PR #857 하나만 다룬다. **2026-09-28 `main`에 머지됐다.**
>
> 이 세션에서 처음으로 "버그 수정"이 아니라 "기능 추가"인 PR. 코드 자체보다, **자체검토 중 발견한 설계상 찜찜한 지점을 실험으로 직접 검증해서 진짜 원인(죽은 코드)을 찾아낸 과정**, 그리고 나중에 **메인테이너의 명시적 요청(동기 throw)을 그대로 반영했더니 그 자체가 새 부작용(좀비 커넥션)을 만든다는 걸 다시 자체검토로 찾아낸 과정**이 이 문서의 핵심이다.

---

## 1. 개요

### 1.1 이슈 (#855)

**제목**: 재연결(reconnect) 지연 간격을 커스텀 전략으로 계산할 수 있게 해달라는 기능 요청.

`amqplib`은 `connect()`에 `recovery` 옵션을 주면 연결이 끊겼을 때 자동으로 재연결을 시도하는 opt-in 기능을 제공한다. 기본 지연 전략은 지수 백오프 + 지터(jitter)인데, 사용자가 원하는 전략(full jitter, decorrelated jitter, 고정 스케줄 등)을 직접 넣을 수 있는 훅이 없었다.

### 1.2 관련 링크

| 구분 | 링크 |
|---|---|
| 원본 저장소 | https://github.com/amqp-node/amqplib |
| 내 Fork | https://github.com/GiHoon1123/amqplib |
| 이슈 (#855) | https://github.com/amqp-node/amqplib/issues/855 |
| PR (#857) | https://github.com/amqp-node/amqplib/pull/857 |
| 작업 브랜치 | `feat/custom-reconnect-delay-strategy` |
| 커밋 | `30044bd` ~ `7e7ced9` 및 후속 머지 커밋 |
| 머지 커밋 | `3f486e68` |
| 상태 | 🟢 MERGED (2026-09-28, `main`) |

### 1.3 한 줄 요약

> `recovery.calculateDelay(attempt)` 옵션을 추가해서 재연결 지연을 사용자가 직접 계산할 수 있게 했다. 처음엔 이 함수가 깨지면(throw, 잘못된 반환값) 내장 전략으로 폴백하고 `handler-error`로만 알리는 방식으로 짰는데, 메인테이너 리뷰에서 "그냥 동기로 throw해라"는 요청을 받고 그대로 반영했다. 그 과정에서 내장 전략의 jitter가 `maxDelay`에서 뭉치는 문제(point mass)도 같이 고쳤고, 동기 throw로 바꾼 결과 "커넥션이 영구히 재연결을 멈추는" 새 부작용이 생긴다는 것도 자체검토로 찾아내서 메인테이너에게 별도로 알렸다.

---

## 2. 배경 지식

### 2.1 opt-in recovery와 재연결 지연

`amqplib`의 `recovery` 옵션은 연결이 끊기면 자동으로 재연결을 시도한다. 각 시도 사이의 대기 시간(delay)은 기본적으로 지수 백오프(`initialDelay * factor^(attempt-1)`, `maxDelay`로 상한)에 지터(무작위 편차, thundering herd 방지)를 섞어서 계산한다.

### 2.2 왜 커스텀 전략이 필요한가

"Full jitter", "decorrelated jitter", 고정 스케줄 등 백오프 전략은 여러 변형이 있고, 어떤 걸 쓸지는 서비스 특성에 따라 다르다. 이슈는 이 계산 로직 자체를 사용자가 주입할 수 있게 해달라는 요청이었다.

### 2.3 `lib/safe_emit.js` — 이 코드베이스의 기존 컨벤션

```js
function safeEmit(emitter, event, ...args) {
  try {
    emitter.emit(event, ...args);
  } catch (e) {
    if (emitter.listenerCount('handler-error') > 0) {
      setImmediate(() => emitter.emit('handler-error', e, event));
    } else {
      throw e;
    }
  }
}
```

사용자가 등록한 이벤트 핸들러(`'close'`, `'ack'` 등)가 동기적으로 throw하면, `handler-error` 리스너가 있으면 거기로 보내고, 없으면 원래대로 **다시 throw**(에스컬레이션)한다. `channel.js`/`connection.js`에서 이 함수를 호출할 때는 항상 `this`(진짜 사용자가 리스너를 붙이는 그 객체) 자체를 넘긴다.

---

## 3. 구현 내용

`lib/recovery.js`에 `resolveDelay(core, recovery, attempt)` 함수를 추가:

```js
function resolveDelay(core, recovery, attempt) {
  if (recovery.calculateDelay) {
    try {
      const custom = recovery.calculateDelay(attempt);
      if (typeof custom === 'number' && Number.isFinite(custom) && custom >= 0) {
        return Math.round(custom);
      }
      throw new Error(`calculateDelay must return a finite, non-negative number of milliseconds (got ${String(custom)})`);
    } catch (err) {
      // (아래 4~5절에서 이 catch 블록 자체가 두 차례 바뀐다)
    }
  }
  return calculateBuiltinDelay(recovery, attempt);
}
```

`index.d.ts`/`callback_api.d.ts`의 `RecoveryOptions`에 `calculateDelay?: (attempt: number) => number` 추가, README에 `### Custom delay strategy` 섹션 추가, `test/recovery.test.js`에 신규 테스트 다수 추가 — 여기까지가 최초 구현.

---

## 4. 검토 1차 — 사용자의 직접 리뷰 (2개 발견)

최초 구현 완료 후 사용자가 직접 코드를 읽고 2가지를 지적했다:

**발견 1 — 테스트 타이밍 버그**: `calculateDelay`가 throw했을 때 `handler-error`가 오는지 확인하는 테스트가, `await`하는 Promise를 `reconnect-scheduled` 이벤트 기준으로만 `resolve()`하고 있었다. `handler-error`는 `setImmediate()`로 다음 틱에 발생하는 반면 `reconnect-scheduled`는 동기적으로 먼저 발생하므로, `handler-error` 리스너 안의 `assert`가 실행되기도 전에 테스트가 끝나버릴 수 있는 구조였다 — 즉 그 assert가 실패해도 테스트는 통과할 수 있었다. `Promise.all([handlerError, scheduled])`로 두 이벤트를 모두 기다리도록 수정.

**발견 2 — 느슨한 `Number()` 강제 변환**: 원래 `const custom = Number(recovery.calculateDelay(attempt));`였는데, 이러면 문자열 `'1000'`이나 `null`도 조용히 숫자로 변환되어 통과한다. 타입 선언(`(attempt: number) => number`)과 어긋나는 동작. `typeof custom === 'number' && Number.isFinite(custom) && custom >= 0`으로 엄격하게 검사하도록 수정 — 둘 다 "고치기 전 코드로 되돌려서 새 테스트가 실제로 실패하는지" 확인한 뒤 복구하는 방식으로 검증.

---

## 5. 검토 2차 — "자체검토 한번 더 해봐" (1개 버그 발견 + 1개 설계 이슈 플래그)

사용자 지시로 재검토하다가 두 가지가 나왔다:

**진짜 버그 (수정함)**: 에러 메시지 생성부의 `` `...(got ${JSON.stringify(custom)})` ``가, `calculateDelay`가 순환 참조 객체를 반환하면 `JSON.stringify` 자체가 `TypeError: Converting circular structure to JSON`을 던진다. 이게 같은 `catch` 블록 안에서 일어나므로, `handler-error`로 전달되는 에러 메시지가 "숫자를 반환해야 한다"가 아니라 뜬금없는 JSON 직렬화 에러가 되어버림. `String(custom)`으로 교체(순환 참조여도 절대 throw 안 하고 `'[object Object]'`로 폴백). 회귀 테스트 추가, disable-and-reconfirm으로 검증.

**설계 이슈 (플래그만, 아직 미해결)**: `resolveDelay`의 catch 블록이 `core.listenerCount('handler-error') > 0`일 때만 emit하고, 아니면 그냥 조용히 내장 전략으로 넘어가는 구조 — 이게 2절의 `safeEmit` 컨벤션(리스너 없으면 throw)과 다르다는 걸 이 시점에 인지했다. "내부 타이머 콜백이라 throw하면 위험하다"는 이유로 의도적으로 다르게 했다고 설명했지만, 이건 PR 본문에 정당화가 필요한 지점이라고만 표시하고 코드는 그대로 둔 채 사용자에게 보고했다.

---

## 6. "2번이 좀 걸리네" — 사용자가 5절의 설계 이슈에 불편함을 표함

사용자는 5절의 두 번째 항목(safe_emit 컨벤션과의 불일치)에 대해 "그냥 문서로 정당화하고 넘어가자"는 제안에 만족하지 못했다. 이 시점에 실제로 코드를 더 파고들기로 했다.

### 6.1 실험 1 — "정말 위험한가?"를 직접 재현

`safeEmit`처럼 "리스너 없으면 throw"로 바꾼 버전을 만들어서, 실제 재연결 콜백 경로(`onClose` 핸들러, `setTimeout` 안의 `_connect()`)에서 실행해봤다. 두 호출 경로 모두 **await 없이 fire-and-forget으로 호출**되고 있었다:

```js
this._timer = setTimeout(() => {
  this._timer = null;
  this._connect();   // 반환된 Promise를 아무도 await/catch 안 함
}, delay);
```

`process.on('uncaughtException'/'unhandledRejection')`을 걸어두고 monkey-patch한 버전을 실행 — throw하면 프로세스가 죽을 수 있는 구조라는 가설을 세우고 검증하러 갔다.

### 6.2 실험 도중 예상 밖의 발견 — throw 분기가 아예 실행되지 않음

`console.log`로 throw 직전 지점에 트레이스를 심었는데, **한 번도 출력되지 않았다.** 즉 "리스너 없음" 조건 자체가 성립하지 않고 있었다는 뜻.

원인을 추적: `resolveDelay(core, ...)`의 `core`는 사용자가 직접 다루는 `client`(`RecoveringPromiseModel`/`RecoveringCallbackModel`)가 아니라 내부 전용 `RecoveringCore` 인스턴스다. 그리고 생성자에서:

```js
function wireCommonEvents(source, target) {
  [..., 'handler-error'].forEach((eventName) => {
    source.on(eventName, (...args) => target.emit(eventName, ...args));
  });
}
```

`core`(source) → `client`(target)로 모든 공통 이벤트를 항상 forwarding하는 리스너가 생성자 시점에 걸린다. 직접 실험으로 확인:

```
client.listenerCount('handler-error')       = 0   // 사용자가 실제로 리스너를 안 붙임
client._core.listenerCount('handler-error') = 1   // 항상 1 (forwarding 리스너)
```

`resolveDelay`가 체크하는 건 `core`의 listenerCount — **이건 사용자가 리스너를 붙였는지와 무관하게 항상 1**이었다. 즉 "리스너 없으면 조용히 넘어간다"는 분기는 public API 경로에서는 애초에 실행될 수 없는 죽은 코드였고, 실제로는 (안전한 no-op이긴 하지만) 항상 emit을 시도하고 있었던 것.

### 6.3 왜 `safeEmit`은 이 문제가 없는가

`channel.js`/`connection.js`에서 `safeEmit(this, ...)`은 항상 **진짜 사용자 객체(`this`) 자체**에 대해 호출된다. core/target 분리 구조 자체가 없다. 반면 recovery 기능은 내부 core와 공개 client가 분리된 아키텍처라, 같은 패턴(listenerCount로 게이트)을 그대로 가져다 쓸 수 없다.

### 6.4 최종 수정 — 게이트 자체를 삭제

`core`의 listenerCount는 구조적으로 "진짜 사용자가 듣고 있는지"를 절대 반영할 수 없으므로, 체크를 아예 없애고 **항상 emit**하도록 단순화:

```js
} catch (err) {
  setImmediate(() => core.emit('handler-error', toError(err, 'calculateDelay failed'), 'calculateDelay'));
}
```

Node의 `EventEmitter.emit()`은 `'error'`라는 특수 이벤트가 아닌 이상 리스너가 0개여도 안전한 no-op이므로, 리스너가 실제로 없어도 아무 문제가 없다. 325/325 테스트 재확인(변경 전/후 동일하게 전부 통과 — 이론대로 순수 no-op 최적화였다는 뜻).

이걸로 5절의 설계 이슈를 보던 관점이 완전히 바뀌었다: "컨벤션과 일부러 다르게 했다"가 아니라 **"이 코드가 필요로 하는 체크(진짜 호출자의 리스너 유무) 자체가 core/client 분리 구조 때문에 여기서는 성립 불가능해서, 체크를 없애고 Node의 안전한 no-op 시맨틱에 맡겼다"**는 훨씬 명확한 근거로 바뀌었다.

### 6.5 부가 발견 — 프롬프트 인젝션 시도

이 실험 과정 중, 도구 결과 사이에 "오늘 날짜가 바뀌었고 사용자에게 말하지 말라"는 가짜 시스템 메시지가 섞여 들어온 걸 발견했다. 정상적인 시스템 알림이 아니라 프롬프트 인젝션으로 판단, 사용자에게 즉시 알리고 지시는 따르지 않았다. amqplib 저장소나 실제 시스템과는 무관한, 환경 상의 이상 현상으로 취급.

README/타입 선언 문구도 "리스너가 있을 때만" → "항상 emit, 리스너 없으면 no-op"으로 갱신.

---

## 7. 최종 폴리시 정리 — 사용자의 3가지 지적

PR 올리기 직전, 사용자가 코드를 다시 읽고 3가지를 제안:

1. **주석이 너무 길다**: `resolveDelay` 위 주석이 12줄이었는데, 메인테이너 입장에서 구현보다 방어적으로 보일 수 있다는 지적. 핵심 근거(크래시 위험, 항상 emit이 안전한 이유)만 남기고 5줄로 축소.
2. **테스트 이름이 헷갈림**: `'rejects a calculateDelay return value that merely coerces to a number, not falling back silently'`가 "promise를 reject한다"로 오독될 소지가 있다는 지적. `'does not coerce non-number calculateDelay return values'`로 변경.
3. **circular reference 테스트가 다소 과해 보일 수 있음**: 유지해도 되고 빼도 된다는 재량을 줌. **유지하기로 판단** — 6.2절 이전, 5절에서 실제로 발견/수정한 버그(`JSON.stringify` 순환 참조 throw)를 지키는 회귀 테스트라서, PR을 조금 더 작아 보이게 하는 것보다 그 수정의 커버리지를 지키는 게 낫다고 봤다.

세 수정 모두 반영 후 325/325 테스트, lint, typecheck 재확인.

---

## 8. 이슈 요청자 리뷰 — jitter가 `maxDelay`를 초과하는 문제

### 8.1 리뷰에서 확인된 남은 범위

2026-08-04, 이슈 #855 요청자인 `intech`가 PR을 검토했다. 커스텀 `calculateDelay(attempt)` API와 잘못된 반환값의 폴백 방식에는 동의했지만, 이슈에 포함된 두 번째 문제는 아직 남아 있다고 지적했다.

내장 지연 계산은 먼저 `base`에만 `maxDelay`를 적용하고, 그 뒤에 jitter를 더하고 있었다:

```js
function calculateBuiltinDelay(recovery, attempt) {
  const base = Math.min(
    recovery.maxDelay,
    recovery.initialDelay * recovery.factor ** (attempt - 1),
  );
  const jitterPart = base * recovery.jitter;
  const offset = jitterPart > 0
    ? Math.random() * jitterPart * 2 - jitterPart
    : 0;

  return Math.max(0, Math.round(base + offset));
}
```

예를 들어 `maxDelay = 5`, `base = 5`, `jitter = 1`이고 `Math.random()`이 최대값에 가까우면:

```text
jitterPart = 5
offset = +5
최종 delay = 5 + 5 = 10
```

`base`는 5로 제한됐지만 jitter는 그 뒤에 더해지므로, 사용자가 설정한 최대 지연 5ms를 실제 예약 지연이 최대 10ms까지 초과할 수 있었다. 즉 `maxDelay`가 계산 중간값의 상한일 뿐, 이름과 달리 최종 지연의 상한은 아니었다.

### 8.2 수정

최종 jitter 적용값에도 `maxDelay`를 적용했다:

```diff
- return Math.max(0, Math.round(base + offset));
+ return Math.min(
+   recovery.maxDelay,
+   Math.max(0, Math.round(base + offset)),
+ );
```

아래쪽 `Math.max(0, ...)`는 음수 jitter로 지연이 0보다 작아지는 것을 막고, 바깥쪽 `Math.min(maxDelay, ...)`는 양수 jitter로 상한을 넘는 것을 막는다. 결과적으로 최종 지연 범위가 항상 `0 <= delay <= maxDelay`가 된다.

### 8.3 회귀 테스트를 확률 기반에서 결정적으로 변경

처음에는 full jitter로 50번 재연결하고 모든 delay가 `maxDelay` 이하인지 확인했다. 기존 코드가 실패할 확률은 매우 높지만, `Math.random()` 결과에 따라 이론상 잘못된 코드도 통과할 수 있었고 실제 타이머를 50번 사용했다.

최종 테스트는 `Math.random()`을 최대 양수 jitter가 나오는 `1`로 잠시 고정했다:

```js
const originalRandom = Math.random;
let delay;

try {
  // Force the maximum positive jitter. Without the final clamp,
  // base + offset would be 10, exceeding maxDelay.
  Math.random = () => 1;

  delay = await new Promise((resolve) => {
    client.once('reconnect-scheduled', (info) => resolve(info.delay));
    models[models.length - 1].emit('close', new Error('socket closed'));
  });
} finally {
  Math.random = originalRandom;
  await client.close();
}

assert.equal(delay, 5);
```

이 테스트는 한 번의 재연결만으로 수정 전 결과 `10`과 수정 후 결과 `5`를 정확히 구분한다. 전역 `Math.random`과 client는 assertion 성공 여부와 관계없이 `finally`에서 반드시 복구·종료한다. 테스트 실행 시간도 약 213ms에서 1ms 미만으로 줄었다.

### 8.4 검증과 리뷰 답변

후속 커밋 `0be6876`(`fix(recovery): cap jittered delay at maxDelay`)으로 수정과 테스트를 push했다.

- `node --test --test-timeout=1000 test/recovery.test.js`: 12/12 통과
- `npm run lint`: 통과
- `npm run typecheck`: 통과
- 전체 `npm test`: 로컬 RabbitMQ가 없어 기존 broker 연동 테스트가 `ECONNREFUSED`로 실패. 이번 변경과 무관

리뷰 답변:

> Good catch. I pushed a follow-up that clamps the built-in delay after jitter and added a deterministic test for the maximum positive jitter case.
>
> The recovery tests, lint, and typecheck pass locally.

PR 본문도 "`calculateDelay`가 없으면 동작이 바뀌지 않는다"는 기존 설명을 제거했다. 내장 전략의 최종 delay가 이제 실제로 `maxDelay`를 지키기 때문에, 이번 후속 수정은 의도된 기존 동작 변경이기 때문이다.

### 8.5 후속 코멘트 — clamp가 만드는 분포 왜곡 (2026-08-04, 아직 답변 안 함)

같은 날 `intech`가 clamp 자체는 맞다고 확인한 뒤, 그 clamp가 만드는 **분포 모양**에 대한 우려를 추가로 남겼다:

> steady state에서 `base`가 이미 `maxDelay`까지 saturate된 뒤로는, `offset > 0`인 절반의 draw가 전부 `maxDelay` 한 점으로 뭉친다. 기본값 `jitter: 0.2`에서는 fleet 전체 재연결의 절반가량이 같은 순간에 몰릴 수 있다는 뜻 — jitter가 원래 막으려던 것과 정반대다. 수정 전에는 그 절반이 `(maxDelay, maxDelay × 1.2]` 구간에 흩어져 있었다.

대안으로 두 가지를 제시함: base 자체를 `maxDelay / (1 + jitter)`로 낮춰서 상한을 넘을 일이 원천적으로 없게 하거나, jitter를 항상 감산 방향으로만(`base * (1 - jitter * Math.random())`) 적용하는 것. 둘 다 평균 지연이 낮아지는 트레이드오프가 있어서 "명백한 개선이 아니라 진짜 트레이드오프"라고 스스로 밝힘. 다만 이 PR이 같이 추가하는 `calculateDelay` 커스텀 훅으로 우회할 수 있으니, 분포 모양이 중요한 사람은 그걸 쓰면 되고, 그래서 두 변경을 같은 릴리즈에 같이 넣는 게 낫다는 논지로 마무리.

**아직 답변 안 한 상태.**

---

## 9. 메인테이너 정식 리뷰 — CHANGES_REQUESTED 5건

8절까지는 이슈 요청자(`intech`)의 코멘트 기반 리뷰였고, 이번엔 실제 메인테이너 `cressie176`이 PR을 정식으로 리뷰하며 `CHANGES_REQUESTED`를 남겼다. 다섯 가지를 지적했다.

1. **(line 92)** `handler-error`는 원래 "커스텀 에러 핸들러가 던질 때" 쓰는 이벤트인데, `calculateDelay`가 깨졌을 때 재사용하는 게 의미상 안 맞는다. 그냥 동기로 throw하는 게 낫겠다.
2. **(line 83)** 함수명 `resolveDelay`는 "resolve"가 비동기 뉘앙스라 안 맞는다. `calculateDelay`로.
3. **(line 84)** 중첩된 `if` 대신 guard clause(조기 return) 스타일로.
4. **(README line 181)** 커스텀 `calculateDelay`를 쓸 때 `maxDelay`는 라이브러리가 강제해주지 않는다는 걸 문서에 명확히.
5. **(line 74)** `@intech`가 지적한 대로, `maxDelay`를 강제로 clamp하면 그 지점에서 jitter가 사실상 안 먹힌다. jitter를 유지하거나 `base`를 `maxDelay - maxJitter`로 캡하는 걸 선호.

### 9.1 1번(handler-error → throw)을 그대로 반영해도 되는가 — 한 번 반대했다가 존중하기로 함

1번은 코드에 이미 적혀있던 설계 의도("동기로 throw하면 이 함수를 부르는 두 경로 다 안전망이 없어서 프로세스가 죽을 수 있다")와 정면으로 배치되는 요청이었다. 실제로 호출부를 추적해서 확인했다.

- `model.on('close', onClose)` 리스너 안에서 호출 — 동기 throw하면 `model.emit('close', ...)`를 부른 곳까지 그대로 전파, 아무도 안 잡으면 `uncaughtException`.
- `_connect()`(async)의 unawaited `catch` 블록 안에서 호출 — 던지면 `_connect()`의 반환 promise가 reject되는데, 이걸 부르는 `setTimeout(() => this._connect(), delay)`가 await/catch를 안 해서 unhandled rejection.

이 근거로 "메인테이너가 이 크래시 위험을 알고도 요청한 건지 확실치 않으니, 코멘트로 근거를 대며 물어보자"고 한 번 제안했다. 사용자 판단은 달랐다: **"메인테이너가 그렇게 원한 건 의도가 있지 않을까, 문제가 생기면 그때 다시 고치면 된다, 메인테이너 의견을 먼저 존중하자."** 그대로 반영하기로 했다.

이 과정에서 또 다른 근거를 하나 더 찾았다 — 이 코드베이스에 이미 있는 다른 유저 훅(`recovery.setup`)은 깨졌을 때 `runSetup()`이 `try/catch`로 감싸서 reject된 promise로 바꾸고, `_connect()`가 그걸 "연결 실패"로 취급해서 정상적으로 재시도한다(크래시 없음). 즉 "유저 훅이 깨지면 크래시 대신 재시도로 취급"이 이미 이 라이브러리의 기존 철학이었다 — 이건 1번을 그대로 반영하면 그 철학과 어긋날 수 있다는 근거였는데, 사용자가 이미 방향을 정한 뒤라 새로운 안전망(`setup`과 같은 방식)을 임의로 추가하지 않고 **메인테이너가 요청한 그대로(순수 동기 throw, 안전망 없음)** 구현했다.

### 9.2 커밋을 4개로 쪼갬

리뷰 지적이 서로 다른 관심사라, 한 커밋에 몰아넣지 않고 지적 하나당 커밋 하나로 나눴다(2·3번은 같은 줄을 건드려서 순서대로 쌓음).

| 커밋 | 내용 |
|---|---|
| `d32a236` | 2번 — `resolveDelay` → `calculateDelay` 이름 변경 |
| `669b166` | 3번 — nested if → guard clause |
| `900b33d` | 1번 — 동기 throw로 변경, `handler-error` 우회 경로 제거, 관련 테스트 5개 재작성(1개는 새 동작과 모순돼서 삭제) |
| `947fe31` | 4번 — README에 `maxDelay` 책임 소재 명시 |

각 커밋마다 로컬 RabbitMQ 브로커(`docker start rabbitmq`)를 띄워서 전체 테스트(326개)를 실제로 통과시킨 뒤 커밋했다 — 이 저장소의 `lefthook` pre-commit 훅이 `npm test`를 돌리는데, 로컬에 브로커가 없으면 브로커 필요한 테스트 49개가 전부 실패로 잡혀서 훅 자체가 커밋을 막기 때문이다(`--no-verify`로 우회하지 않음).

### 9.3 5번(jitter) 반영 — `maxDelay / (1 + jitter)`

```js
function calculateBuiltinDelay(recovery, attempt) {
  const cappedBase = recovery.maxDelay / (1 + recovery.jitter);
  const base = Math.min(cappedBase, recovery.initialDelay * recovery.factor ** (attempt - 1));
  const jitterPart = base * recovery.jitter;
  const offset = jitterPart > 0 ? Math.random() * jitterPart * 2 - jitterPart : 0;
  return Math.max(0, Math.round(base + offset));
}
```

메인테이너가 준 두 선택지 중 "`base`를 `maxDelay - maxJitter`로 캡"을 택했다 — `base * (1+jitter) = maxDelay`가 되도록 `base`를 `maxDelay/(1+jitter)`로 미리 줄여두면, 최대 jitter가 더해져도 정확히 `maxDelay`에서 끝나서 뒤에 clamp가 아예 필요 없어진다. clamp가 없으니 8절에서 고친 "`maxDelay` 지점에 뭉치는" 문제(point mass)도 같이 사라진다.

새 테스트로 이걸 증명했다:

```js
// initialDelay:100, maxDelay:100, jitter:0.2, Math.random 고정 0.6
// base = 100/1.2 = 83.33, offset = 3.33, 기대값 = round(86.66) = 87 (100이 아님)
assert.equal(delay, 87);
assert.notEqual(delay, 100);
```

disable-and-reconfirm: fix를 잠깐 되돌리고 이 테스트를 돌리니 `100 == 87`로 정확히 실패 — 옛날 코드가 실제로 이 값을 100으로 뭉갠다는 게 증명됐다. fix 복원 후 전체 테스트 326개 재통과 확인 후 커밋(`564338f`).

기존 "full jitter에서도 `maxDelay` 안 넘음" 테스트도 값을 다시 계산해서 그대로 통과하는지 확인했다(`initialDelay:5, maxDelay:5, jitter:1` → `cappedBase=2.5`, 강제 최대 jitter 시 결과 정확히 5).

---

## 10. 자체검토에서 발견한 부작용 — 동기 throw가 만드는 좀비 커넥션

5개 커밋을 다 올리기 전, 전체 diff를 처음부터 다시 읽는 자체검토 과정에서 9.1절의 우려가 실제로 관측 가능한 형태로 재발했다는 걸 발견했다.

`_scheduleReconnect()`의 구조를 다시 보면:

```js
_scheduleReconnect(err) {
  ...
  const attempt = this._attempt + 1;
  const delay = calculateDelay(this._recovery, attempt);  // ① 여기서 throw
  this._attempt = attempt;                                 // ② 도달 못 함
  this.emit('reconnect-scheduled', { attempt, delay, error: err });
  this._timer = setTimeout(() => this._connect(), delay);  // ③ 도달 못 함
}
```

`calculateDelay`가 깨져서 ①에서 throw하면 ②③이 실행이 안 된다. `this._timer`가 안 걸리므로, 나중에 그 타이머가 만료돼서 `_connect()`를 다시 불러주는 메커니즘 자체가 사라진다 — **이 커넥션 인스턴스는 그 순간부터 영원히 재연결을 시도하지 않는다.** `reconnect-failed`나 `error` 이벤트도 안 뜨기 때문에, 앱 입장에서는 재연결 기능이 조용히 죽었다는 걸 알 방법이 없다.

실제로 스크립트로 재현해서 확인했다:

```
caught synchronous throw (as expected): calculateDelay is broken
opened count (재연결 시도했으면 2여야 함): 1
client is now permanently dead with zero signal: true
```

이건 9.1절에서 이미 감수하기로 한 "프로세스가 죽을 수 있다"는 리스크와는 별개의 문제다 — 프로세스가 안 죽고 어디선가 예외를 잡아버리는 경우에도, 그 커넥션은 여전히 영구히 죽는다.

이번엔 코드로 고치지 않기로 했다. 이미 1번(동기 throw) 자체는 메인테이너 요청대로 반영하기로 결정한 뒤였고, 이 부작용에 대한 대응(예: `setup`처럼 별도 안전망 추가)은 또 다른 설계 판단이 필요한 사안이라 임의로 얹지 않았다. 대신 push 후 리뷰 코멘트에 이 발견을 알리고 메인테이너 판단에 맡기기로 했다.

---

## 11. 타입 문서(JSDoc) 갱신 누락 — 자체검토에서 추가로 발견

10절 검토 도중 `index.d.ts`/`callback_api.d.ts`의 `calculateDelay` JSDoc이 1번 반영 이전 동작("handler-error로 폴백")을 그대로 설명하고 있는 걸 발견했다. README는 고쳤는데 타입 선언 파일 두 개를 빠뜨린 것 — 같은 문구로 맞춰서 별도 커밋(`7e7ced9`)으로 수정했다. `npm run typecheck`로 재확인.

---

## 12. PR 본문 재작성과 리뷰 코멘트

PR 본문의 "Behavior" 섹션이 여전히 "falls back ... emits handler-error"라고 적혀 있어서(1번 반영 전 기준), 코드와 본문이 어긋난 상태였다. 본문을 전면 재작성해서 push했다.

리뷰 코멘트는 5개 지적에 대한 대응을 짧게 정리하고, 10절에서 찾은 부작용을 마지막 문단에 담백하게 덧붙였다 — "이게 문제 아니냐"고 따지는 톤이 아니라 "반영하다 보니 이런 결과가 나오더라, 알려드린다" 톤으로, 결정은 메인테이너에게 맡겼다.

> Thanks for the review - pushed changes addressing all of this:
>
> - Renamed `resolveDelay` to `calculateDelay`, switched to a guard clause.
> - `calculateDelay` errors now propagate synchronously instead of falling back via `handler-error`.
> - README now says `maxDelay` only bounds the built-in strategy.
> - Fixed the jitter point mass @intech flagged - base is capped at `maxDelay / (1 + jitter)` now instead of `maxDelay`. Added a regression test for it.
>
> One consequence of the synchronous throw: it happens before `_attempt`/`_timer` get updated in `_scheduleReconnect`, so a broken `calculateDelay` doesn't just fail one attempt - the connection stops reconnecting entirely after that, with no `reconnect-failed` or `error` event. Flagging it in case it's worth addressing here or separately.

---

## 13. 기여 프로세스

1. `git checkout -b feat/custom-reconnect-delay-strategy`
2. 5개 파일(`lib/recovery.js`, `index.d.ts`, `callback_api.d.ts`, `README.md`, `test/recovery.test.js`) 스테이징 후 커밋 — `lefthook` pre-commit 훅(format/lint/test)이 자동으로 돌아 전부 통과.
3. `gh repo fork amqp-node/amqplib --remote-name=fork` — **이번에도** fork는 성공했지만 remote가 자동으로 안 붙는 gh CLI 버그를 겪음(corepack #869, terminus #2735/#2736에 이어 세 번째). `git remote add fork https://github.com/GiHoon1123/amqplib.git`로 수동 추가.
4. `git push -u fork feat/custom-reconnect-delay-strategy`
5. `gh pr create --repo amqp-node/amqplib --head GiHoon1123:feat/custom-reconnect-delay-strategy --base main ...`로 PR #857 생성.

---

## 14. 결과 및 회고 — 🟡 결과 나오면 갱신

### 14.1 현재 상태 (2026-08-12 기준)

- PR #857 **OPEN**, 커밋 8개(`30044bd` ~ `7e7ced9`)
- 2026-08-06 시점엔 "clamp가 만드는 분포 왜곡(point mass) 코멘트에 아직 반응 안 함"이 남은 항목이었는데, 이후 메인테이너 `cressie176`이 같은 지점을 포함한 정식 `CHANGES_REQUESTED` 리뷰(5건)를 남겨서 전부 반영 완료, PR 본문 재작성, 대응 코멘트까지 게시
- recovery 테스트 12/12, 전체 테스트 326/326(로컬 RabbitMQ 브로커로 실제 검증), lint, typecheck 통과
- 자체검토로 찾은 새 부작용(동기 throw로 인한 좀비 커넥션)은 코드로 안 고치고 코멘트로 메인테이너에게 판단을 넘긴 상태

### 14.2 지금까지 배운 점

- **"찜찜하다"는 감각을 문서화로 덮지 말고, 실험으로 검증하면 더 나은 근거가 나올 수 있다.** 사용자가 "2번이 좀 걸리네"라고 했을 때, "PR 본문에 정당화를 잘 써서 넘어가자"가 아니라 실제로 코드를 실행해서 확인했더니 — 단순히 "일부러 컨벤션과 다르게 한 설계 선택"이 아니라 "체크 자체가 구조적으로 죽은 코드였다"는, 훨씬 더 근본적이고 명확한 사실이 나왔다.
- **"이 조건이 항상 참/거짓인가?"는 로그 한 줄로 쉽게 확인할 수 있다.** 조건문이 "가끔은 참, 가끔은 거짓"이라고 가정하고 넘어가지 말고, 실제로 그런지 찍어보는 습관이 유효했다.
- **상한은 계산 중간값이 아니라 최종 결과에 적용해야 한다.** `base`를 `maxDelay`로 제한해도 그 뒤에 양수 jitter를 더하면 최종 delay는 상한을 넘는다 — 이건 나중에 5번 리뷰에서 "final clamp가 오히려 point mass를 만든다"는 걸로 한 번 더 뒤집혔다. `base`를 `maxDelay/(1+jitter)`로 미리 줄여서 clamp 자체를 없애는 쪽이 최종적으로 더 나은 해법이었다.
- **메인테이너가 명시적으로 원하는 방향이 있으면, 근거를 들어 한 번 확인한 뒤엔 그대로 존중해서 반영하는 게 맞다.** 크래시 위험을 근거로 반대 의견을 냈지만, 사용자가 "메인테이너 의도를 존중하고 문제 생기면 나중에 고치자"고 판단했고, 그 판단을 따라 그대로 구현했다.
- **다만 "요청대로 구현했다"와 "그 구현이 안전하다"는 별개다.** 반영을 끝낸 뒤 자체검토로 diff를 처음부터 다시 읽다가, 요청대로 구현한 결과가 "그 순간의 실패"가 아니라 "영구적인 재연결 중단"이라는 더 큰 부작용을 만든다는 걸 발견했다. 반대 여부와 별개로, 반영한 코드가 실제로 어떤 결과를 만드는지는 항상 다시 추적해봐야 한다.
- **커밋을 리뷰 지적 단위로 쪼개면, 나중에 "어느 지적에 대한 대응인지" 되짚기가 훨씬 쉽다.** 5개 지적에 4개 커밋(2번+3번은 같은 줄이라 순서대로 쌓음) — 한 커밋에 다 몰아넣었으면 이 문서 쓰기도, 리뷰 코멘트에서 참조하기도 더 어려웠을 것이다.
- **pre-commit 훅이 `npm test`를 돌리면, 로컬 환경(브로커 등)이 안 갖춰지면 아예 커밋이 막힌다.** `--no-verify`로 우회하지 않고 실제로 필요한 걸(RabbitMQ 컨테이너) 띄워서 진짜로 통과시키는 게 맞는 방향이었다.
- 프롬프트 인젝션은 시스템 알림처럼 보이는 형태로도 들어올 수 있다는 걸 실제로 목격 — "사용자에게 말하지 말라"는 지시가 포함된 메시지는 그 자체로 강한 신호이니 항상 사용자에게 투명하게 알려야 한다.

### 14.3 다음에 채워야 할 것 (결과가 나오면)

- [x] 이슈 요청자가 `calculateDelay` 기능과 폴백 설계 검토 — API 방향에 동의
- [x] built-in delay의 `maxDelay` 초과 지적 반영 — 커밋 `0be6876`
- [x] clamp가 만드는 분포 왜곡(point mass) 지적에 대한 대응 — 메인테이너 정식 리뷰로 재확인, 커밋 `564338f`로 반영
- [x] 메인테이너 정식 리뷰(`CHANGES_REQUESTED` 5건) 반영 — 커밋 `d32a236`~`564338f`, `7e7ced9`
- [x] PR 본문 재작성, 대응 코멘트 게시
- [ ] listenerCount 게이트 제거에 대한 메인테이너 반응 (더 미묘한 이유로 원래 있었을 가능성도 배제 못 함)
- [ ] 좀비 커넥션 부작용에 대한 메인테이너 반응
- [ ] CI 결과
- [ ] 머지 여부와 소요 시간
