# [OSS 기여] platformatic/kafka - `close(true)` 반환 타입 오버로드 수정 (PR #410)

작성일: 2026-09-22  
저장소: https://github.com/platformatic/kafka  
이슈: https://github.com/platformatic/kafka/issues/409  
PR: https://github.com/platformatic/kafka/pull/410  
상태: Merged (2026-09-22, `main`)
작업 커밋: `e27eb8e7d740d22c06dc42570f3f2de0226fbf46`
머지 커밋: `5fb95a7`

---

## 1. 한 줄 요약

`Consumer.close(true)`와 `Producer.close(true)`는 런타임에는 `Promise<void>`를 반환하고 있었지만,
TypeScript가 먼저 선언된 콜백 오버로드를 선택해서 반환 타입을 `void`로 추론했다. Promise 오버로드를
앞으로 옮기고 기존 force-close 테스트에 명시적인 `Promise<void>` 타입 검증을 추가했다.

실행 코드는 바꾸지 않았다. 이번 수정은 실제 동작과 공개 타입 선언이 서로 다르던 문제를 바로잡은
타입 수정이다.

---

## 2. 프로젝트와 관련 객체

### 2.1 `@platformatic/kafka`

`@platformatic/kafka`는 Node.js에서 Kafka를 사용할 수 있게 해 주는 TypeScript 기반 클라이언트다.
브로커 연결, 프로토콜 요청과 응답, producer, consumer, consumer group, 스트림 처리 등을 저장소 안에서
구현한다.

이번 수정과 직접 관련된 공개 객체는 다음 두 개다.

- `Consumer`: 토픽의 레코드를 읽고, consumer group 상태와 fetch 세션 및 consume stream을 관리한다.
- `Producer`: Kafka로 레코드를 보내고, producer stream과 transaction 등 생산 관련 상태를 관리한다.

두 객체는 모두 `close()`를 제공한다. 사용자는 콜백 방식과 Promise 방식 중 하나를 선택할 수 있고,
열려 있는 스트림을 강제로 닫을지 나타내는 `force` 값도 전달할 수 있다.

### 2.2 `force`가 필요한 이유

consumer나 producer에 활성 stream이 남아 있으면 일반 종료는 진행 중인 작업을 갑자기 끊지 않도록
종료를 거부할 수 있다. `close(true)`는 활성 stream까지 닫고 객체를 종료하겠다는 명시적인 요청이다.

예를 들어 producer에는 다음과 같은 흐름이 있다.

1. `close()`가 현재 stream 목록을 확인한다.
2. stream이 남아 있고 `force`가 `false`이면 오류를 콜백으로 전달하고 닫힌 상태를 되돌린다.
3. stream이 남아 있고 `force`가 `true`이면 모든 stream을 닫는다.
4. stream 정리가 끝나면 기반 client 연결을 닫는다.
5. callback을 호출하거나, 내부 callback에 연결된 Promise를 완료한다.

따라서 `close(true)`는 단순한 boolean setter가 아니라 비동기 정리 작업을 시작하는 호출이다. 콜백을
주지 않았다면 호출자는 그 작업이 끝날 때까지 기다릴 Promise를 받아야 한다.

---

## 3. 이 API가 콜백과 Promise를 동시에 지원하는 방식

### 3.1 사용자가 호출할 수 있는 형태

의도한 공개 API를 호출 형태별로 풀어 쓰면 다음과 같다.

| 호출 | 의미 | 의도한 반환 타입 |
|---|---|---|
| `close()` | 일반 종료, Promise 사용 | `Promise<void>` |
| `close(false)` | 일반 종료, Promise 사용 | `Promise<void>` |
| `close(true)` | 강제 종료, Promise 사용 | `Promise<void>` |
| `close(callback)` | 일반 종료, callback 사용 | `void` |
| `close(false, callback)` | 일반 종료, callback 사용 | `void` |
| `close(true, callback)` | 강제 종료, callback 사용 | `void` |

같은 JavaScript 함수가 인자의 모양에 따라 두 스타일을 모두 처리한다.

### 3.2 `CallbackWithPromise`

저장소에는 일반 callback에 선택적인 Promise를 붙일 수 있는 내부 타입이 있다.

```ts
export type CallbackWithPromise<ReturnType> =
  Callback<ReturnType> & { [kCallbackPromise]?: Promise<ReturnType> }
```

`createPromisifiedCallback()`은 다음 두 역할을 함께 하는 함수를 만든다.

1. 평범한 error-first callback처럼 `(error, payload)`를 받는다.
2. callback 객체의 `kCallbackPromise` 심볼 프로퍼티에 Promise를 보관한다.

개념적으로 단순화하면 다음과 같다.

```ts
function createPromisifiedCallback() {
  const { promise, resolve, reject } = Promise.withResolvers()

  function callback(error, payload) {
    if (error) reject(error)
    else resolve(payload)
  }

  callback[kCallbackPromise] = promise
  return callback
}
```

이 구조 덕분에 실제 종료 로직은 callback 기반으로 한 번만 구현하면서, callback을 넘기지 않은 호출자에게는
같은 완료 신호를 Promise로 돌려줄 수 있다.

### 3.3 실제 `close()` 구현 흐름

Consumer와 Producer의 `close()`는 세 단계로 인자를 정리한다.

```ts
if (typeof force === 'function') {
  callback = force
  force = false
}

if (!callback) {
  callback = createPromisifiedCallback<void>()
}
```

1. 첫 인자가 함수라면 `close(callback)` 형태이므로 그 함수를 callback으로 옮기고 `force`를 `false`로 둔다.
2. callback이 없다면 Promise에 연결된 내부 callback을 만든다.
3. 종료 작업에는 항상 callback을 넘기고, 마지막에는 `callback[kCallbackPromise]`를 반환한다.

사용자가 callback을 직접 넘겼다면 그 callback에는 내부 Promise가 없으므로 반환값은 `undefined`, 즉
공개 타입으로는 `void`다. callback을 넘기지 않았다면 새로 만든 callback에 Promise가 있으므로 그
Promise가 반환된다.

```text
close(true)
  └─ callback 없음
      └─ createPromisifiedCallback()
          ├─ 종료 로직에는 callback 전달
          └─ 호출자에게 callback[kCallbackPromise] 반환
              └─ Promise<void>

close(true, callback)
  └─ 사용자가 준 callback 사용
      ├─ 종료 로직에는 callback 전달
      └─ 내부 Promise 없음
          └─ void
```

런타임 구현은 이 구분을 이미 정확히 하고 있었다. 문제는 TypeScript 선언이 이 실행 결과를 정확히
표현하지 못했다는 점이다.

---

## 4. TypeScript 오버로드의 기본 구조

### 4.1 오버로드 시그니처와 구현 시그니처

TypeScript에서 하나의 함수가 여러 호출 형태를 지원할 때, 외부에 보여 줄 오버로드 시그니처를 먼저
나열하고 마지막에 실제 구현 시그니처를 둔다.

수정 전 코드는 다음과 같았다.

```ts
close(
  force: boolean | CallbackWithPromise<void>,
  callback?: CallbackWithPromise<void>
): void
close(force?: boolean): Promise<void>
close(
  force?: boolean | CallbackWithPromise<void>,
  callback?: CallbackWithPromise<void>
): void | Promise<void> {
  // 실제 구현
}
```

앞의 두 줄이 사용자가 호출할 때 검사되는 오버로드 시그니처다. 마지막 줄은 구현이 모든 입력과 반환을
처리할 수 있는지 검사하기 위한 구현 시그니처다. 일반 호출자에게 마지막의 넓은
`void | Promise<void>` 시그니처가 별도의 선택지로 제공되는 것은 아니다.

### 4.2 왜 선언 순서가 중요한가

호출 인자가 여러 오버로드에 모두 들어맞으면 TypeScript는 위에서부터 검사해서 먼저 일치하는
오버로드를 선택한다. 따라서 오버로드끼리 겹치는 API에서는 더 구체적이거나 의도한 반환 타입을 가진
선언을 먼저 둬야 한다.

수정 전 `close(true)`를 대입해 보면 다음과 같다.

첫 번째 오버로드:

```ts
close(
  force: boolean | CallbackWithPromise<void>,
  callback?: CallbackWithPromise<void>
): void
```

- `true`는 `boolean`에 해당한다.
- 두 번째 `callback` 인자는 optional이므로 생략해도 된다.
- 따라서 `close(true)`와 일치한다.
- 이 오버로드의 반환 타입은 `void`다.

두 번째 오버로드:

```ts
close(force?: boolean): Promise<void>
```

- `true`는 `boolean`에 해당한다.
- 이 오버로드에도 일치한다.
- 반환 타입은 `Promise<void>`다.

둘 다 일치하지만, 수정 전에는 `void` 오버로드가 먼저였다. TypeScript는 첫 번째 일치 항목을 선택했고
`close(true)`를 `void`로 추론했다.

```text
close(true)
  ├─ 1번 오버로드와 일치 → void
  └─ 2번 오버로드와도 일치 → Promise<void>

TypeScript가 먼저 만난 1번을 선택
→ 정적 타입: void
→ 실제 런타임 값: Promise<void>
```

이것이 이슈 #409의 근본 원인이다.

---

## 5. 왜 `await close(true)`만으로는 문제가 잘 드러나지 않았나

기존 테스트에는 이미 다음 코드가 있었다.

```ts
await consumer.close(true)
```

겉으로 보면 `close(true)`가 Promise인지 확인하는 것처럼 보이지만, TypeScript의 `await`는 Promise가
아닌 값에도 사용할 수 있다.

```ts
await undefined
await 123
await someVoidReturningFunction()
```

이 코드는 문법 및 타입 차원에서 허용된다. Promise가 아닌 값이면 즉시 완료된 값처럼 처리된다. 따라서
`close(true)`가 잘못 `void`로 추론되어도 단순히 `await`만 붙인 기존 테스트는 타입 오류를 만들지 않았다.

런타임에서는 구현이 실제 Promise를 반환하므로 테스트도 정상적으로 종료를 기다렸다. 결과적으로 다음
두 사실이 함께 존재했다.

- 런타임 테스트는 통과했다.
- TypeScript 사용자에게 보이는 반환 타입은 틀렸다.

이런 종류의 버그는 실행 결과만 확인하는 테스트로는 잡기 어렵고, 추론된 타입을 명시적으로 제한하는
검증이 필요하다.

---

## 6. 수정 내용

### 6.1 Promise 오버로드를 앞으로 이동

Consumer와 Producer에 같은 변경을 적용했다.

```diff
- close(force: boolean | CallbackWithPromise<void>, callback?: CallbackWithPromise<void>): void
  close(force?: boolean): Promise<void>
+ close(force: boolean | CallbackWithPromise<void>, callback?: CallbackWithPromise<void>): void
```

이제 `close(true)`는 위에서부터 다음 순서로 검사된다.

1. `close(force?: boolean): Promise<void>`와 일치한다.
2. TypeScript가 이 오버로드를 선택하고 탐색을 끝낸다.
3. 반환 타입이 실제 런타임과 같은 `Promise<void>`가 된다.

callback을 전달하는 호출은 첫 번째 Promise 오버로드와 맞지 않으므로 다음 오버로드로 내려간다.

```ts
consumer.close(true, callback) // void 오버로드 선택
consumer.close(callback)       // void 오버로드 선택
```

따라서 Promise 호출의 타입만 바로잡고 기존 callback 사용법은 그대로 유지된다.

### 6.2 Consumer와 Producer를 함께 수정한 이유

이슈의 예시는 Consumer였지만 Producer에도 동일한 오버로드와 동일한 런타임 변환 구조가 있었다.
Producer에서도 `producer.close(true)`가 `void`로 추론됐으므로 같은 결함이다.

두 구현은 공개 API의 대칭을 이루고 있고, 수정도 동일한 선언 순서 변경이다. 한쪽만 고치면 같은
라이브러리에서 같은 이름과 사용법을 가진 메서드의 타입이 서로 달라진다. 별도 기능을 추가한 것이
아니라 같은 원인을 가진 동일 결함을 두 위치에서 바로잡은 것이다.

### 6.3 실행 코드를 건드리지 않은 이유

`typeof force === 'function'` 분기, `createPromisifiedCallback()`, 실제 자원 정리, 에러 전달,
`kCallbackPromise` 반환은 이미 올바르게 동작했다. 실행 코드를 수정하면 다음과 같은 불필요한 위험만
늘어난다.

- stream 강제 종료 순서가 달라질 수 있다.
- callback 호출 시점이나 에러 전달 방식이 바뀔 수 있다.
- Consumer와 Producer의 기존 종료 의미가 달라질 수 있다.

문제는 타입 검사 단계의 오버로드 선택뿐이었으므로 선언 순서만 바꾸는 것이 가장 작은 수정이다.

---

## 7. 회귀 테스트

### 7.1 타입을 명시적으로 고정

기존 force-close 테스트를 다음과 같이 바꿨다.

```ts
const closing: Promise<void> = consumer.close(true)
await closing
```

Producer 테스트에도 같은 검증을 넣었다.

```ts
const closing: Promise<void> = producer.close(true)
await closing
```

이 코드는 런타임 동작과 정적 타입을 한 번에 확인한다.

- 타입 검사: `close(true)`가 반드시 `Promise<void>`여야 대입된다.
- 런타임 검사: 그 Promise를 기다린 뒤 객체와 stream이 실제로 닫혔는지 기존 assertion이 확인한다.

### 7.2 수정 전 코드에서 실제로 실패하는 이유

오버로드 순서를 원래대로 되돌리면 `close(true)`는 `void`가 된다. 그러면 대입문에서 다음 종류의
TypeScript 오류가 발생한다.

```text
Type 'void' is not assignable to type 'Promise<void>'.
```

즉 새 테스트는 단순히 수정된 코드를 실행하는 테스트가 아니다. 이번 버그의 핵심인 잘못된 반환 타입을
직접 검출한다.

### 7.3 callback 호환성

이번 변경은 callback 오버로드의 내용이나 구현을 삭제하지 않았다. 아래 호출들은 계속 `void`로
처리된다.

```ts
consumer.close(error => {
  // 일반 종료 완료
})

consumer.close(true, error => {
  // 강제 종료 완료
})
```

Promise 호출과 callback 호출을 구분하는 런타임 분기도 바뀌지 않았다.

---

## 8. 검토한 다른 설계

겹치는 오버로드 자체를 없애기 위해 callback 호출을 더 세분화할 수도 있다.

```ts
close(force?: boolean): Promise<void>
close(callback: CallbackWithPromise<void>): void
close(force: boolean, callback: CallbackWithPromise<void>): void
```

이 방식은 각 호출 형태를 더 정확히 분리한다는 장점이 있다. 특히 callback용 두 번째 인자를 optional로
두지 않으므로 boolean 하나만 전달한 호출이 callback 오버로드에 겹치지 않는다.

하지만 이번 이슈에는 적용하지 않았다.

- 이슈가 요청한 해결 방향은 잘못된 오버로드 순서를 바로잡는 것이었다.
- 기존 공개 선언 형태를 크게 재구성할 필요가 없었다.
- 두 줄의 순서 변경만으로 실제 동작과 타입을 일치시킬 수 있었다.
- 작은 타입 버그 수정에 API 선언 정리까지 섞으면 리뷰 범위가 불필요하게 넓어진다.

현재 수정은 최소 범위로 원인을 해결한다. 메인테이너가 오버로드를 서로 겹치지 않게 분리하길 원한다면
리뷰에서 위 형태로 조정할 수 있다.

---

## 9. 호환성과 위험 분석

### 9.1 런타임 호환성

런타임 구현은 한 줄도 바뀌지 않았다. JavaScript 사용자가 받는 값, callback 호출 시점, 강제 종료
동작과 에러 처리에는 변화가 없다.

### 9.2 TypeScript 호환성

의도한 변경은 `close(true)`와 `close(false)`의 추론 결과가 `void`에서 `Promise<void>`로 바뀌는 것이다.
이는 실제 반환값과 문서화된 Promise 사용 방식에 맞춘 수정이다.

다음 호출들의 타입 의미는 그대로다.

- `close()` → `Promise<void>`
- `close(callback)` → `void`
- `close(force, callback)` → `void`

### 9.3 변경 범위

변경 파일은 네 개다.

| 파일 | 변경 |
|---|---|
| `src/clients/consumer/consumer.ts` | `close()` 오버로드 순서 변경 |
| `src/clients/producer/producer.ts` | `close()` 오버로드 순서 변경 |
| `test/clients/consumer/consumer.test.ts` | force-close 반환 타입 검증 |
| `test/clients/producer/producer.test.ts` | force-close 반환 타입 검증 |

총 diff는 6 insertions, 4 deletions다.

---

## 10. 저장소 규칙 확인

작업 전에 저장소의 기여 가이드와 최근 커밋을 확인했다.

- 이슈를 해결하는 PR 본문 첫 부분에 `Fixes #409.`를 넣었다.
- 코드 스타일과 기존 테스트 위치를 유지했다.
- 별도의 리팩터링이나 관련 없는 변경을 포함하지 않았다.
- 저장소가 요구하는 DCO를 위해 커밋에 개인 계정의 `Signed-off-by`를 넣었다.
- 커밋 작성자와 GitHub PR 작성자는 모두 `GiHoon1123`인지 확인했다.
- 커밋, PR 본문과 트레일러에는 도구 이름이나 공동 작성자 표기를 넣지 않았다.

최종 커밋 메시지는 다음과 같다.

```text
fix: correct close overload resolution

Signed-off-by: GiHoon1123 <rlaejrqo465@naver.com>
```

---

## 11. 로컬 검증

### 11.1 실행한 검사

다음 검사를 모두 통과했다.

```text
npm run typecheck
npm run lint
npm run build
npm test
git diff --check
```

전체 테스트 결과는 다음과 같다.

```text
Execution PASSED after 1 minute and 52.426 seconds with
2670 tests passing out of 2671 tests over 315 files
(1 test was skipped).
```

프로세스 종료 코드는 `0`이었다.

### 11.2 처음 발생한 실패와 코드 결함의 구분

첫 전체 테스트에서는 로컬 서비스 상태 때문에 일부 테스트가 실패했다. 두 종류의 환경 문제가 있었다.

1. 다른 로컬 개발 프로세스가 테스트에서 사용하는 8000번 포트를 점유하고 있었다.
2. Kafka 테스트 컨테이너 일부가 이전 실행의 상태와 볼륨을 가진 채 남아 있어 Kerberos keytab 및 브로커
   연결 상태가 현재 구성과 맞지 않았다.

8000번 포트를 사용하던 로컬 watch 프로세스를 종료하고, 문제가 있던 Kafka/Kerberos 테스트 컨테이너를
현재 구성으로 다시 만든 뒤 전체 테스트를 처음부터 실행했다. 그 결과 위의 2670 통과, 1 skip 결과를
얻었다.

이 과정이 중요한 이유는 테스트 실패를 곧바로 코드 회귀로 단정하지도, 반대로 환경 문제라고 추측한 채
넘기지도 않았기 때문이다. 포트 점유 프로세스와 컨테이너 상태를 직접 확인하고 정리한 뒤 동일한 코드로
전체 스위트를 다시 통과시켰다.

### 11.3 변경 후 자체 검토 항목

- boolean 하나를 넘긴 호출이 이제 Promise 오버로드를 먼저 선택하는가: 예.
- callback만 넘긴 호출이 여전히 callback 오버로드를 선택하는가: 예.
- boolean과 callback을 함께 넘긴 호출이 여전히 `void`인가: 예.
- Consumer와 Producer에 동일한 결함이 남아 있지 않은가: 두 위치 모두 수정.
- 테스트가 수정 전 코드에서도 통과하는 형식적인 테스트인가: 아니오. 수정 전에는 명시적
  `Promise<void>` 대입이 타입 오류를 낸다.
- 실행 코드나 종료 순서를 건드렸는가: 아니오.
- 관련 없는 파일이 커밋에 포함됐는가: 아니오.

---

## 12. 제출 내용

### 12.1 브랜치와 커밋

- 브랜치: `fix-409-close-overload`
- fork: `GiHoon1123/kafka`
- 커밋: `e27eb8e7d740d22c06dc42570f3f2de0226fbf46`
- 커밋 작성자: `GiHoon1123 <rlaejrqo465@naver.com>`

### 12.2 PR 본문

```markdown
Fixes #409.

`close(true)` returns a promise when no callback is supplied, but TypeScript
selected the callback overload first and inferred `void`. Put the promise
overload first for both consumers and producers, and assert the force-close
return type in their existing tests.

## Tests

- `npm run typecheck`
- `npm run lint`
- `npm run build`
- `npm test`
```

PR 본문은 문제, 수정 범위와 실행한 테스트만 짧게 적었다. 상세한 학습 내용은 공개 PR에 길게 붙이지
않고 이 개인 복습 문서에 남겼다.

### 12.3 생성 직후 상태

- PR 번호: #410
- 상태: Open
- GitHub 기준 mergeable: `MERGEABLE`
- DCO: 통과
- Node.js 및 Kafka 버전별 CI 매트릭스: 실행 중
- 리뷰: 대기 중

---

## 13. 이번 작업에서 배운 핵심

### 13.1 런타임과 타입 선언은 따로 검증해야 한다

코드가 실제로 Promise를 반환한다고 해서 TypeScript가 자동으로 그 사실을 정확히 알 수 있는 것은 아니다.
오버로드가 있는 공개 API에서는 사용자가 보는 타입이 별도의 계약이다. 실행 테스트와 타입 테스트를 둘 다
봐야 한다.

### 13.2 오버로드는 집합이 아니라 우선순위가 있는 목록이다

처음에는 두 오버로드 중 하나가 알아서 가장 적절하게 선택될 것처럼 보일 수 있다. 하지만 겹치는
시그니처에서는 선언 순서가 선택 결과를 바꾼다. 특히 optional 인자가 있으면 생각보다 많은 호출이 앞선
오버로드에 들어맞을 수 있다.

### 13.3 `await`가 Promise 타입 검증은 아니다

`await expression`이 컴파일된다는 사실만으로 `expression`의 타입이 Promise라는 결론을 내릴 수 없다.
반환 타입 자체가 계약인 버그에서는 명시적인 변수 타입, 타입 assertion 도구 또는 컴파일 실패 테스트가
필요하다.

### 13.4 작은 diff도 원인 분석은 작지 않을 수 있다

최종 코드 변경은 오버로드 두 쌍의 순서를 바꾸고 테스트 두 곳에 변수를 추가한 것이 전부다. 하지만 이
수정이 맞다고 판단하려면 다음을 모두 확인해야 했다.

- 실제 `close()`의 callback/Promise 변환 방식
- `CallbackWithPromise`와 `kCallbackPromise`의 관계
- Consumer와 Producer의 종료 흐름
- TypeScript의 오버로드 선택 규칙
- `await`가 허용하는 표현식 범위
- 기존 callback 호출의 호환성
- 회귀 테스트가 수정 전 타입을 실제로 거부하는지

코드 줄 수보다 중요한 것은 실제 동작, 공개 타입과 테스트가 같은 계약을 가리키도록 만드는 것이다.

---

## 14. 이후 기록할 내용

이 문서는 PR이 열려 있는 동안 `typescript/open`에 둔다. 이후 리뷰가 오면 다음 내용을 시간순으로
추가한다.

- CI 최종 결과와 실패가 있다면 원인
- 메인테이너 또는 리뷰 봇의 지적
- 리뷰에 따라 바꾼 코드와 판단 근거
- 추가로 실행한 테스트
- merge 또는 close 날짜와 최종 결과

PR이 merge되면 최신 상태를 덧붙인 뒤 파일을 `typescript/merge`로 옮긴다. merge 없이 종료되면 종료
이유를 기록하고 `typescript/close`로 옮긴다.
