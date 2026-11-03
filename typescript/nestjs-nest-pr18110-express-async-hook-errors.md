# NestJS — Express 어댑터의 비동기 훅 오류 전달

상태: 후속 PR #18110 Merged. 2026-10-12 18:37:04 KST에 kamilmysliwiec이 병합했다. 최초 PR #18094는 수정 방향을 안내받고 Closed.

1~17절은 최초 제출 당시의 기록이다. 당시의 판단과 코드 설명을 보존했다.
특히 9.2절의 늦은 오류 전달과 11절의 once guard 판단은 이후 리뷰로 수정되었다.
종료 사유, 재현, 완료 보호 처리와 후속 PR의 현재 동작은 18절부터 읽는다.
27절에는 메인테이너의 마지막 수정과 머지 결과를 기록했다. 앞선 테스트 수와 코드 예시는 각 작업 시점의 기록이다.

| 항목 | 내용 |
| --- | --- |
| 저장소 | [nestjs/nest](https://github.com/nestjs/nest) |
| 이슈 | [#18093 — Express adapter leaves async lifecycle hook errors unhandled](https://github.com/nestjs/nest/issues/18093) |
| 현재 PR | [#18110 — fix(express): guard async request hook completion](https://github.com/nestjs/nest/pull/18110) |
| 이전 PR | [#18094 — fix(express): handle async lifecycle hook errors, Closed](https://github.com/nestjs/nest/pull/18094) |
| 종료 리뷰 | [kamilmysliwiec의 완료 처리 피드백](https://github.com/nestjs/nest/pull/18094#issuecomment-6120382505) |
| 공개 재현 | [Nest 12.1.2 재현 코드](https://gist.github.com/GiHoon1123/453781bd6d97ca397daae5ea52f5221a) |
| 관련 작업 | [#17997 — Fastify async lifecycle hooks](https://github.com/nestjs/nest/pull/17997) |
| 포크 | [GiHoon1123/nest](https://github.com/GiHoon1123/nest) |
| 현재 브랜치 | `fix/express-hook-completion` |
| 최초 브랜치 | `fix/express-async-hook-errors` |
| 최종 PR 커밋 | [`f99178258`](https://github.com/nestjs/nest/commit/f9917825857d508c59ab8c2a4e088ed06c8a774b) |
| 머지 커밋 | [`c9c251ca1`](https://github.com/nestjs/nest/commit/c9c251ca1ad14a467e62980127ad3367bbb027ca) |
| 최초 커밋 | [`d0b09439a`](https://github.com/nestjs/nest/commit/d0b09439a8a12902fcfb77741b9ff060cab0160a) |
| 현재 기준 upstream | `7036f3dc0`, `master` |
| 최초 기준 upstream | `2731c1805`, `master` |
| 로컬 환경 | macOS, Node.js 22.22.3, npm 10.9.8 |

## 1. 어떤 문제를 찾았나

Express 어댑터에는 요청 시작과 응답 완료 시점에 함수를 실행할 수 있는 두 메서드가 있다.

- `setOnRequestHook()`: 요청 처리가 시작될 때 실행할 함수 등록.
- `setOnResponseHook()`: 응답 전송이 끝난 뒤 실행할 함수 등록.

두 메서드는 Promise를 반환하는 함수도 받는다. 그런데 기존 구현은 그 Promise의 실패를 처리하지 않았다.

요청 훅에서 비동기 작업이 실패하면 Express 오류 처리기로 전달되지 않았다. 훅이 `done()`에 도달하기 전에
실패한 경우에는 다음 처리도 실행되지 않아 요청이 대기 상태로 남았다. 응답 훅에서는 응답을 보낸 다음
발생한 실패가 처리되지 않은 Promise rejection으로 남았다.

핵심은 비동기 함수를 호출했는지가 아니다. **그 함수가 반환한 Promise를 누가 관찰하는가**다.

이번 수정은 요청 훅의 Promise를 Express에 돌려주고, 응답 훅의 오류는 완료 이벤트 안에서 잡아 기록한다.
요청을 다음 단계로 보내는 기존 `done()` 사용법은 바꾸지 않았다.

## 2. Nest, Express, Node는 각각 무엇을 하나

Nest로 서버를 작성하면 보통 컨트롤러와 서비스를 만든다.

```ts
@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Get()
  findAll() {
    return this.usersService.findAll();
  }
}
```

이 코드만 보면 브라우저 요청을 Nest가 가장 먼저 받는 것처럼 느껴진다. 하지만 일반적인 Node HTTP 서버의
실행 경로를 더 아래까지 내려가면 역할이 나뉜다.

```text
브라우저의 HTTP 요청
  → 운영체제의 네트워크 처리
  → Node HTTP 서버
  → Express의 요청 처리 함수와 미들웨어
  → Nest가 Express에 등록한 라우트 처리 함수
  → Nest의 가드·파이프·인터셉터 등
  → 컨트롤러와 서비스
  → 응답 처리
```

이 그림은 주요 호출 관계를 나타낸다. Nest 미들웨어도 Express 미들웨어 체인에 등록될 수 있고,
인터셉터는 컨트롤러 실행 전후를 감싸므로 모든 기능이 한 방향의 단순 목록으로만 실행되는 것은 아니다.

Node는 HTTP 서버와 소켓 같은 실행 기반을 제공한다. Express는 라우팅, 미들웨어 호출과 오류 전달을 맡는다.
Nest는 그 위에 모듈, 의존성 주입, 컨트롤러, 서비스, 가드, 파이프와 같은 애플리케이션 구조를 제공한다.

따라서 “Nest는 백엔드 구조를 잡고 HTTP 처리는 Express나 Fastify를 이용한다”는 설명이 나오는 것이다.
Nest가 HTTP 처리에 전혀 관여하지 않는다는 뜻은 아니다. Nest도 요청을 해석하고 응답을 구성하지만,
그 처리를 실제 HTTP 라이브러리와 연결하는 계층이 따로 있다.

## 3. ExpressAdapter는 왜 필요한가

Nest가 Express의 모든 메서드를 곳곳에서 직접 호출하면 Fastify를 지원하기 어려워진다.
같은 일을 하는 메서드도 두 라이브러리에서 이름이나 사용법이 다를 수 있기 때문이다.

어댑터는 이 차이를 연결한다.

```text
Nest가 요구하는 HTTP 기능
  ├─ ExpressAdapter → Express의 구현
  └─ FastifyAdapter → Fastify의 구현
```

예를 들어 Nest는 어댑터를 통해 라우트를 등록하고, 응답을 보내고, 헤더를 설정한다.
어댑터는 그 요청을 자신이 담당하는 라이브러리의 호출로 바꾼다.

이번 버그는 Express 자체의 Promise 오류 처리 기능이 없어서 생긴 문제가 아니다.
Nest의 ExpressAdapter가 Promise를 Express에 전달하지 않아 그 기능을 사용하지 못한 문제다.

수정 파일은 `packages/platform-express/adapters/express-adapter.ts`다.
Nest의 모든 HTTP 어댑터를 바꾸거나 Express 라이브러리를 수정한 작업이 아니다.

## 4. 훅이란 무엇인가

훅은 특정 시점에 실행할 수 있도록 등록해 두는 함수다.
이름이 특별해 보이지만 기본 구조는 콜백 함수 등록이다.

```ts
adapter.setOnRequestHook((_req, _res, done) => {
  console.log('request started');
  done();
});
```

이 코드는 등록하는 순간 요청을 처리하지 않는다. 나중에 요청이 들어오면 어댑터가 등록된 함수를 호출한다.

Nest의 공통 HTTP 어댑터 문서에서는 이 훅들을 instrumentation과 devtools 용도로 설명한다.
요청 정보를 기록하거나, 실행 시간을 측정하거나, 개발 도구에 이벤트를 보내는 상황을 생각하면 된다.
일반적인 컨트롤러 API를 대체하라는 의미는 아니다.

### 4.1 `recordRequest()`는 Nest의 메서드인가

설명에 사용한 `recordRequest()`는 사용자가 작성하는 예시 함수다. Nest가 제공하는 특별한 API가 아니다.

```ts
import { appendFile } from 'node:fs/promises';

async function recordRequest(req: { method: string; url: string }) {
  await appendFile('requests.log', `${req.method} ${req.url}\n`);
}

adapter.setOnRequestHook(async (req, _res, done) => {
  await recordRequest(req);
  done();
});
```

이 예에서는 파일에 요청 정보를 쓴 뒤 다음 단계로 진행한다.
실제 운영용 로깅 구조를 추천하는 예제가 아니라, 비동기 실패가 생기는 위치를 보여 주기 위한 코드다.

파일 쓰기가 실패하면 `await recordRequest(req)`에서 예외가 발생한다.
그 아래의 `done()`은 실행되지 않는다. `async` 함수가 반환한 Promise는 rejected 상태가 된다.
이 Promise를 아무도 처리하지 않는 것이 이번 문제의 출발점이다.

## 5. `_req`, `_res`의 밑줄은 무슨 뜻인가

밑줄은 “인자를 받기는 하지만 이 함수에서는 사용하지 않는다”는 관례다.
JavaScript의 특별한 문법이 아니며, 인자 전달을 차단하는 기능도 없다.

```ts
adapter.setOnRequestHook((_req, _res, done) => {
  done();
});
```

이 함수는 세 번째 인자인 `done`만 필요하다. 하지만 인자는 위치에 따라 들어온다.

```ts
hook(req, res, next);
```

첫 번째 매개변수에는 `req`, 두 번째에는 `res`, 세 번째에는 `next`가 들어간다.
앞의 두 이름을 없애고 `(done) => done()`이라고 쓰면 첫 번째 값인 `req`를 `done`이라는 이름으로 받게 된다.
이름을 바꾼다고 위치가 바뀌지는 않는다.

그래서 `_req`, `_res`라는 이름으로 자리를 유지한다. 값은 실제로 전달되지만 함수 본문이 읽지 않을 뿐이다.
`req`, `res`라고 이름을 붙여도 실행 의미는 같다. 사용하지 않는 인자의 lint 처리는 저장소 설정에 달려 있다.

## 6. 등록 단계와 실행 단계를 나눠서 보기

“Nest가 Express에게 훅을 호출했는데 왜 갑자기 미들웨어가 나오나?”라는 혼동은 두 시점을 섞으면 생긴다.

### 6.1 서버를 구성할 때

ExpressAdapter의 생성자는 Express 인스턴스에 미들웨어 하나를 등록한다.

```ts
this.instance!.use((req, res, next) => {
  // 요청이 들어왔을 때 실행할 코드
});
```

`use()`는 지금 요청을 보내거나 훅을 실행하는 코드가 아니다.
Express가 나중에 호출할 함수를 등록하는 코드다.

`setOnRequestHook(fn)`은 그 어댑터의 `onRequestHook` 필드에 함수를 저장한다.
요청이 들어오면 앞에서 등록한 미들웨어가 이 필드를 확인하고 호출한다.

### 6.2 요청이 들어왔을 때

```text
Express가 어댑터의 미들웨어를 호출
  → 미들웨어가 this.onRequestHook을 확인
  → 등록된 훅을 req, res, next와 함께 호출
  → 훅이 done()을 호출
  → Express가 다음 미들웨어/라우트로 이동
```

여기서 훅의 `done`과 Express의 `next`는 별도 함수가 아니다.
어댑터가 세 번째 인자로 넘긴 바로 그 함수에 사용자가 `done`이라는 이름을 붙인 것이다.

`done()`은 현재 함수에서 `return`하는 것과도 다르다.
Express에 다음 처리를 진행하라고 알릴 뿐, 훅 본문의 나머지 코드를 강제로 종료하지 않는다.

## 7. 기존 코드는 어디서 오류를 놓쳤나

문제의 요청 경로는 다음과 같았다.

```ts
this.instance!.use((req, res, next) => {
  if (this.onRequestHook) {
    void this.onRequestHook.apply(this, [req, res, next]);
  } else {
    next();
  }
});
```

`apply(this, [req, res, next])`는 함수를 호출하는 방법이다.
인자는 배열에 담아 전달하고, 일반 함수 안의 `this`는 어댑터로 지정한다.
`apply()`도 호출한 함수의 반환값을 반환한다. Promise가 반환되면 그 Promise를 받을 수 있다.
화살표 함수의 `this`는 이런 방식으로 다시 지정되지 않는다는 점은 별개다.

여기서 중요한 것은 `apply()`가 아니다. 호출 결과를 바깥 미들웨어의 반환값으로 연결하지 않았다는 점이다.

### 7.1 일반 숫자로 바꿔 보면

```ts
function inner() {
  return 10;
}

function outer() {
  inner();
}

console.log(outer()); // undefined
```

안쪽 함수가 `10`을 반환했어도 바깥 함수가 자동으로 `10`을 반환하지 않는다.

```ts
function outer() {
  return inner();
}

console.log(outer()); // 10
```

Promise도 같은 원리다.

```ts
function outer() {
  return asyncWork();
}
```

이제 `outer()`를 호출한 쪽이 `asyncWork()`의 Promise를 받아 성공과 실패를 관찰할 수 있다.

### 7.2 `void`가 오류를 잡아 주지는 않는다

```ts
void asyncWork();
```

이때 `void`는 표현식을 실행한 뒤 결과를 `undefined`로 만드는 JavaScript 연산자다.
오류를 잡거나 Promise를 취소하거나 처리가 끝날 때까지 기다려 주는 연산자가 아니다.
TypeScript의 반환 타입 `void`와도 구분해야 한다.

다만 `void` 글자만 지우는 것으로는 해결되지 않는다.

```ts
this.instance!.use((req, res, next) => {
  this.onRequestHook.apply(this, [req, res, next]);
});
```

이 코드도 `return`이 없으므로 Express가 받는 것은 `undefined`다.
실제 이력을 확인해 보니 `void`는 스타일 정리 과정에서 추가됐고, 그 이전에도 Promise를 반환하지 않았다.
따라서 그 스타일 변경을 버그의 최초 원인이라고 지목하지 않았다.

## 8. 왜 동기 오류는 전달됐는데 비동기 오류는 빠졌나

일반 함수가 실행 중 바로 `throw`하면 호출자의 `try/catch`가 잡을 수 있다.

```ts
try {
  hook();
} catch (error) {
  next(error);
}
```

하지만 `async` 함수의 오류는 반환된 Promise의 rejection으로 표현된다.
`async` 함수 본문에서 첫 `await` 전에 던져도 이 점은 같다.

```ts
async function hook() {
  throw new Error('failed');
}

try {
  hook(); // rejected Promise를 돌려받지만 여기서는 관찰하지 않는다.
} catch (error) {
  // Promise rejection을 이 catch가 처리하지 않는다.
}
```

Promise를 `await`하거나 rejection handler를 등록하거나, 그 일을 해 줄 호출자에게 반환해야 한다.

이번에 확인한 Express 5의 router는 미들웨어 반환값을 확인해 Promise의 rejection을 처리한다.
설명용으로 줄이면 다음 구조다.

```ts
try {
  const result = middleware(req, res, next);
  if (isPromise(result)) {
    result.then(null, error => {
      next(error || new Error('Rejected promise'));
    });
  }
} catch (error) {
  next(error);
}
```

실제 확인 위치는 설치된 `router/lib/layer.js`의 `handleRequest()`다.
Nest 미들웨어가 Promise 대신 `undefined`를 반환하면 위의 Promise 처리 경로에 들어가지 못한다.
Express가 내부에서 호출된 모든 비동기 작업을 자동으로 찾아낼 수는 없다.

## 9. 요청 훅은 어떻게 고쳤나

```ts
if (this.onRequestHook) {
  return this.onRequestHook.apply(this, [req, res, next]);
} else {
  next();
}
```

반환값을 Express에 연결했다.

```text
요청 훅의 Promise가 reject
  → 같은 Promise를 반환한 어댑터 미들웨어
  → Express의 Promise rejection handler
  → next(error)
  → Express에 등록된 오류 처리기
  → Nest 애플리케이션에서는 연결된 예외 처리 경로
```

Promise의 실패 처리 로직을 어댑터에 새로 복제하지 않았다.
이미 사용하는 Express 5의 기능을 이용하면 이유 없는 rejection의 처리도 Express와 같아진다.
`Promise.reject()`처럼 값이 없는 실패도 Express가 `Error('Rejected promise')`로 바꿔 전달한다.

### 9.1 성공하면 자동으로 다음 단계로 가나

아니다. Express의 이 경로는 Promise가 성공했다고 자동으로 `next()`를 호출하지 않는다.
훅이 `done()`을 호출하는 기존 성공 경로를 유지했다.

```ts
adapter.setOnRequestHook(async (req, _res, done) => {
  await recordRequest(req);
  done();
});
```

이 코드는 `done()`에서 한 번 진행한다. 그 뒤 Promise가 resolve된다고 또 진행하지 않는다.
반대로 Promise가 먼저 resolve되고 `done()`이 나중에 호출되면 그때까지 라우트를 실행하지 않는다.

단순히 오류만 전달하려다가 성공 시점을 바꾸면 기존 프로그램의 실행 순서를 바꾸게 된다.
그래서 “반환만 하면 동작하네”에서 끝내지 않고 이 성공 경로도 별도로 테스트했다.

### 9.2 `done()`을 호출한 뒤 실패하면

그 rejection도 Express에 전달한다. 한 번 진행했으니 모든 후속 오류를 버리는 once guard를 추가하지 않았다.

오류 처리기에 도착했을 때 이미 응답 헤더를 보냈을 수 있다.
그 경우 무엇을 할지는 Express의 오류 처리 흐름과 등록된 handler의 책임이다.
기본 오류 처리기는 이미 전송을 시작한 응답의 연결을 닫을 수도 있으므로,
“늦은 오류여도 언제나 응답에는 영향이 없다”고 일반화하면 안 된다.

회귀 테스트에서는 오류를 기록하는 사용자 handler를 등록했다.
그 조건에서 200 응답과 함께 늦은 오류가 한 번 전달되고 `headersSent`가 참인 것을 확인했다.

## 10. 응답 훅은 왜 같은 방법으로 해결하지 않았나

기존 응답 훅은 다음 코드로 호출됐다.

```ts
res.on('finish', () => {
  void this.onResponseHook!.apply(this, [req, res]);
});
```

이 콜백은 Express의 미들웨어가 아니라 응답 객체의 이벤트 리스너다.
Express의 router가 이 리스너 반환값을 받아 검사하는 구조가 아니다.
따라서 이 안에서 Promise를 반환하는 것만으로는 요청 훅과 같은 효과를 얻지 못한다.

수정은 다음과 같다.

```ts
res.on('finish', async () => {
  try {
    await this.onResponseHook!.apply(this, [req, res]);
  } catch (err) {
    this.logger.error(err);
  }
});
```

동기 `throw`는 `try`에서 잡고, Promise rejection은 `await`를 통해 같은 `catch`에서 잡는다.
오류는 기존 어댑터 logger로 기록한다.

`finish`는 서버가 응답 데이터 전송을 하위 시스템에 넘긴 완료 시점이다.
클라이언트가 모든 바이트를 읽고 처리했다는 확인 신호는 아니다.
하지만 새 HTTP 오류 응답을 보내기에는 이미 늦은 시점이라는 점은 같다.
응답 후 계측 작업이 실패했다고 이미 전송한 200 응답을 새 500 응답으로 교체할 수는 없다.

그래서 두 훅의 오류를 억지로 같은 처리 함수에 모으지 않았다.
요청 전 오류 전달과 응답 후 오류 기록은 생명주기상 책임이 다르다.

## 11. Fastify 수정과 무엇이 다른가

관련 PR #17997은 Fastify의 비동기 lifecycle hook과 완료 처리를 다룬다.
이를 통해 비슷한 경계가 다른 어댑터에도 있다는 단서는 얻었지만 코드를 그대로 옮기지는 않았다.

Express와 Fastify는 훅 완료 규칙과 호출 주체가 다르다.
Fastify 쪽의 완료 guard를 복사하면 Express에서도 Promise resolve를 요청 진행 신호로 삼거나,
`done()` 이후의 rejection을 무시하는 동작을 추가할 수 있다.

이번에 해결하려는 문제는 Express에서 빠진 오류 전달이다.
모든 어댑터의 callback/Promise 완료 규칙을 통일하는 것은 별도의 API 설계 문제다.
공통 주석만 보고 두 구현이 완전히 같다고 가정하지 않고, Express의 기존 실행 결과를 기준으로 범위를 정했다.

## 12. 배포 버전으로 독립 재현하기

수정한 저장소 테스트만 통과하는 것으로 끝내지 않고 별도 디렉터리에서 배포 패키지를 설치했다.

```json
{
  "private": true,
  "type": "module",
  "scripts": {
    "repro:request": "node repro.mjs request",
    "repro:response": "node repro.mjs response"
  },
  "dependencies": {
    "@nestjs/common": "12.1.2",
    "@nestjs/core": "12.1.2",
    "@nestjs/platform-express": "12.1.2",
    "reflect-metadata": "0.2.2",
    "rxjs": "7.8.2"
  }
}
```

재현 코드는 실제 `NestFactory.create()`로 작은 애플리케이션을 띄운다.
포트는 `0`으로 지정해 사용 가능한 포트를 배정받고, `127.0.0.1`로 요청을 보낸다.

요청 모드에서는 훅 안의 Promise를 reject시키고, 응답 모드에서는 응답 훅을 reject시킨다.
`unhandledRejection` 리스너는 오류를 고치기 위해서가 아니라 프로세스를 살려 HTTP 결과까지 관찰하기 위해 달았다.
따라서 그 리스너가 있다는 사실과 이유를 재현 README 및 이슈에 명시했다.

실행 결과:

```text
npm run repro:request
unhandledRejection: request hook failed
HTTP: TimeoutError

npm run repro:response
unhandledRejection: response hook failed
HTTP: 200 ok
```

`TimeoutError`는 예제의 2초 제한에 걸린 결과다. Nest가 2초 뒤에 요청을 정상 종료했다는 뜻이 아니다.
원래 요청은 오류 응답도 다음 처리도 없이 대기하고 있었다.

수정 후에는 빌드한 패키지로 실제 Nest 앱을 실행하고 `--unhandled-rejections=strict`를 사용했다.
이 검증에는 전역 `unhandledRejection` 리스너를 달지 않았다.
요청 훅 실패는 500 응답, 응답 훅 실패는 완료된 200 응답과 오류 기록으로 처리되고 프로세스가 완료됐다.

## 13. 회귀 테스트가 확인하는 것

새 파일은 `packages/platform-express/test/adapters/express-async-hooks.spec.ts`다.
실제 ExpressAdapter와 supertest를 사용한다. 단순히 훅 함수를 직접 호출해 catch되는지만 확인하지 않는다.

| 경우 | 확인한 결과 |
| --- | --- |
| 요청 훅이 Error로 reject | 오류 처리기 한 번 호출, 500 응답 |
| 요청 훅이 이유 없이 reject | Express의 `Rejected promise` 오류로 500 응답 |
| 요청 훅이 동기 throw | 기존처럼 오류 처리기로 전달 |
| callback 형태의 두 훅 | 일반 함수의 `this`가 어댑터로 유지 |
| async 훅에서 done 후 resolve | 성공 처리를 다시 진행하지 않음 |
| Promise resolve 후 나중에 done | done 전에는 라우트 실행 안 함, 이후 한 번 실행 |
| done 이후 reject | 원래 오류가 한 번 전달되고 headersSent 상태 확인 |
| 응답 훅의 동기 throw | 200 응답 후 logger에 한 번 기록 |
| 응답 훅의 Promise reject | 200 응답 후 logger에 한 번 기록 |

### 13.1 테스트에서 미리 catch를 단 이유

일부 테스트에는 다음 코드가 있다.

```ts
const result = Promise.reject(error);
result.catch(() => {});
return result;
```

수정 전 코드를 검증할 때 test runner 자체의 unhandled rejection으로 결과가 흐려지는 것을 막기 위한 장치다.
`catch()`가 반환한 새 Promise를 반환하는 것이 아니라 원래 rejected Promise인 `result`를 반환한다.
원래 Promise가 fulfilled 상태로 바뀌는 것은 아니다.

따라서 Express의 오류 전달이 빠져 있으면 여전히 HTTP 요청이 끝나지 않거나 오류 handler 호출 검증이 실패한다.
응답 훅도 logger 호출 검증이 실패한다. 미리 단 catch가 수정 구현의 역할을 대신해 테스트를 통과시키지 않는다.

전역 리스너 없이 strict 모드로 실행한 별도의 실제 앱 검증도 이 테스트 장치와 구분해서 수행했다.

### 13.2 성공 조건을 시간에 의존시키지 않기

요청이 멈추는 회귀를 테스트할 때 무한 대기를 피하려고 2초 timeout을 둔다.
timeout 발생은 성공 조건이 아니라 테스트 실패를 제한 시간 안에 끝내기 위한 안전장치다.

지연된 `done()` 테스트는 훅 호출 알림을 Promise로 받고, 직접 보관한 `done`을 호출한다.
임의로 몇백 밀리초 기다렸으니 괜찮다고 판단하는 방식이 아니다.
`setImmediate()`는 완료와 rejection 처리가 진행될 기회를 준 뒤 호출 횟수를 확인할 때 사용했다.

## 14. 실제 검증 결과와 환경 문제

초기 회귀 테스트는 수정 전 7개 중 4개가 실패했다.
요청 rejection 두 경우는 타임아웃, 늦은 rejection은 오류 전달 누락, 응답 rejection은 로그 누락으로 실패했다.
그 뒤 동기 응답 오류와 지연된 `done()` 등 경계 검증을 보강해 최종 9개로 구성했다.

| 명령/검증 | 결과 |
| --- | --- |
| 새 회귀 테스트 | 9개 통과 |
| `env -u NO_COLOR npm test -- --reporter=dot --silent` | 314개 파일, 3,998개 테스트 통과 |
| `npm run build` | 통과 |
| `npm run lint:ci` | 통과, 기존 warning 1개 |
| 변경한 두 파일 Prettier 검사 | 통과 |
| `git diff --check` | 통과 |
| `npm run test:integration -- integration/nest-application --reporter=dot --silent` | 15개 파일, 107개 테스트 통과 |
| 실제 Nest 앱, strict rejection 모드 | 요청 실패 500, 응답 후 실패 처리 확인 |

처음 전체 unit test를 실행했을 때는 3,997개가 통과하고 logger 색상 테스트 한 개가 실패했다.
실행 환경에 `NO_COLOR=1`이 설정돼 있었는데 해당 테스트는 ANSI 색상 코드를 기대했다.
공통 색상 유틸리티는 `NO_COLOR`가 있으면 색상을 사용하지 않는다.

관련 logger 코드와 테스트가 upstream과 같다는 것을 확인한 뒤, 테스트 명령에서만 `NO_COLOR`를 제거했다.
제품 코드나 기존 테스트를 수정해 이 환경 차이를 숨기지 않았다.

lint warning은 `packages/websockets/socket-module.ts`의 기존 optional chain/non-null assertion 관련 항목이다.
이번 변경 파일에서 생긴 경고가 아니다.

전체 Docker 의존 integration suite와 sample 빌드 전체는 로컬에서 실행하지 않았다.
따라서 107개의 Nest 애플리케이션 integration test 통과를 “전체 integration 통과”라고 기록하지 않는다.
지원하는 모든 Node 버전과 운영체제에서 검증했다는 의미도 아니다.

## 15. 조사와 제출 과정

### 2026-10-11: 후보 조사, 재현, 수정

처음에는 Nest의 여러 경계를 조사했다. DI 쪽 가설은 이 작업의 확정된 버그로 이어지지 않았고,
HTTP/2와 SSE 관련 후보 역시 이번 기여로 채택하지 않았다.
후보를 많이 나열하는 대신 실제 호출 흐름과 실패 증거가 확보된 Express 훅으로 좁혔다.

Fastify의 관련 수정과 Express 구현을 비교한 뒤 요청/응답 훅을 각각 재현했다.
수정 전 실패를 확인하고, 성공 경로와 늦은 실패까지 포함해 테스트를 확장했다.
전체 unit, 빌드, lint, HTTP 관련 integration 및 실제 Nest 앱 검증을 수행했다.

### 2026-10-12: 설명, 제출 승인, 공개

Nest와 Express의 역할, 훅과 미들웨어의 관계, `_` 매개변수, `recordRequest()` 예제,
내부 반환값과 바깥 반환값의 차이를 순서대로 설명했다.
문제를 이해한 뒤 이슈, 커밋, 포크 push, PR, 기록 문서 작성까지 진행하기로 했다.

공개 전에 upstream을 다시 가져왔다. 검증한 기준 커밋과 동일했다.
`setOnRequestHook`, Express async hook 관련 키워드로 이슈와 PR을 검색했고,
같은 Express 수정은 찾지 못했다. 이미 병합된 Fastify #17997은 관련 선례로 구분했다.
경쟁 기여가 없다는 판단은 이 확인 시점 기준이지 이후에도 변하지 않는 보장이 아니다.

저장소 기여 가이드, 이슈 양식, PR 양식, 커밋 규칙과 CI 설정을 확인했다.
이슈 양식이 최소 재현 링크를 요구해서 공개 Gist를 만들고, 배포 버전으로 직접 실행한 결과를 넣었다.
수정한 어댑터의 JSDoc에도 오류 처리와 `done()`의 의미를 추가했다.

계정과 Git 작성자, 포크 remote를 각각 확인했다.
GitHub 인증 계정과 커밋 작성자 이름은 `GiHoon1123`이다.
브랜치를 포크에 push하고 upstream `master`를 대상으로 PR을 생성했다.
커밋 훅의 Prettier와 commitlint도 우회하지 않고 실행했다.

PR 변경 파일은 어댑터와 회귀 테스트 두 개뿐이다.
재현 프로젝트, 설치 결과, 생성된 빌드 파일과 개인 학습 문서는 Nest PR에 넣지 않았다.
커밋은 `fix(express): handle async lifecycle hook errors`, 종료 연결은 `Fixes #18093`이다.

제출 직후 확인한 상태는 PR content guard 통과, CodeQL과 CircleCI build-and-test 진행 중이었다.
이 시점에 원격 CI까지 통과했다고 결론 내리지 않았다.

## 16. 이번 수정에서 남길 판단 기준

코드 한 줄로 오류 전달을 고칠 수 있어도 확인할 것은 한 줄이 아니다.
반환값을 연결한 뒤 기존 성공 흐름, 오류 handler의 호출 횟수, 늦은 rejection과 응답 완료 이후 처리까지 봐야 한다.

다른 어댑터에 유사한 코드가 있다고 무조건 같은 helper로 묶지 않는다.
호출 주체가 Promise를 어떻게 소비하는지, 성공과 실패를 어떤 신호로 해석하는지부터 확인한다.

“최소 수정”은 단순히 줄 수를 줄이는 일이 아니다.
이번 문제를 해결하면서 기존 성공 경로와 라이브러리의 책임 분담을 유지하고,
그 경계를 테스트와 API 주석으로 설명할 수 있는 수정이어야 한다.

이후 리뷰, 추가 커밋, CI 결과와 최종 merge/close는 아래에 시간순으로 이어 쓴다.
리뷰가 오기 전에 합의된 설계라고 표현하거나 병합을 확정해서 기록하지 않는다.

## 17. 제출 후 첫 상태 확인 — 2026-10-12

문서 작성 후 다시 확인했을 때 PR content guard와 CodeQL 검사는 통과했다.
CircleCI `build-and-test`는 아직 진행 중이었고, 리뷰나 새 코멘트는 없었다.
확인한 PR head는 로컬 및 포크와 같은 `d0b09439a8a12902fcfb77741b9ff060cab0160a`다.

## 18. 첫 PR이 닫힌 이유 — 2026-10-12

메인테이너 kamilmysliwiec은 `done()` 전에 발생한 rejection을 Express에 반환해서 처리하는 효과는 인정했다.
하지만 훅이 요청을 한 번만 진행시키도록 보장하지 못한다고 지적했다.
오류가 전달된다는 사실만으로 그 전달 시점과 대상까지 올바른 것은 아니었다.

첫 번째 문제는 훅이 `done()`을 호출한 뒤 Promise가 실패하는 경우였다.

```text
요청 훅이 done() 호출
  → 컨트롤러 시작
  → 컨트롤러는 DB 조회 같은 비동기 작업을 기다림

요청 훅의 Promise가 뒤늦게 reject
  → Express가 next(error) 호출
  → 오류 처리기가 먼저 500 응답

컨트롤러의 비동기 작업 완료
  → 정상 응답 전송 시도
  → 이미 헤더를 보낸 응답에 다시 쓰려고 해서 오류 발생
```

두 번째 문제는 `done()`을 두 번 호출하는 경우였다.
첫 호출로 라우트를 실행한 뒤 두 번째 호출이 체인을 다시 진행시켜 not-found handler의 404를 보낼 수 있었다.
중복 콜백 문제는 최초 PR의 `return`이 새로 만든 문제와 구분해야 한다.
하지만 둘 다 이번 수정이 지켜야 할 “한 훅은 한 번만 완료한다”는 조건에 속했다.

메인테이너는 Fastify의 `runLifecycleHook()`처럼 완료 보호 처리를 사용하라고 제안했다.
Promise를 Express에 직접 반환하지 말고, 콜백과 오류가 동일한 보호 장치를 거치게 하며,
이미 완료한 후의 오류는 로그로 남기는 방향이었다.
그 방향으로 수정한 버전은 다시 검토하겠다는 답변과 함께 PR을 닫았다.

버그 자체를 부정한 것이 아니라 해결책이 요청 흐름에 새 문제를 만들 수 있다는 지적이었다.
테스트 개수가 많았고 모두 통과했어도, 올바르지 않은 기대값을 테스트로 고정하면 이 문제를 발견하지 못한다.

## 19. 첫 자체검토에서 놓친 것

기존 늦은 rejection 테스트의 라우트는 즉시 `res.send('ok')`로 응답했다.
오류 handler는 호출 여부만 기록하고 실제 응답은 보내지 않는 빈 함수였다.

그 조건에서는 이미 보낸 정상 응답과 기록된 오류가 공존했다.
하지만 실제 문제가 되는 조건은 “컨트롤러가 아직 작업 중이고, 오류 handler가 먼저 응답하는 경우”였다.
오류 처리기로 한 번 전달되었다는 것만 확인해서는 응답 경쟁을 검출할 수 없었다.

더 중요한 잘못은 그 전달 자체를 정답으로 간주한 것이다.
첫 구현은 Express가 제공하는 Promise 처리에 맡겼다는 점에서는 단순했지만,
Nest의 lifecycle hook이 완료된 뒤에도 Express의 요청 흐름을 다시 진행시킬 수 있게 했다.

관련 Fastify 코드를 읽고도 once guard를 어댑터 차이에 따른 별도 문제로 너무 빨리 분리했다.
구현을 똑같이 복사하지 않는 것과, 그 구현이 지키는 제약을 무시하는 것은 다르다.
이번에는 “어느 라이브러리의 오류 처리 기능을 쓸까”보다 “누가 요청 진행을 몇 번 결정할 수 있나”를 먼저 봐야 했다.

## 20. done과 next를 다시 정리하기

이 부분은 대화에서 여러 번 혼동했으므로 함수의 전달부터 나눠 적는다.

### 20.1 사용자가 등록한 것은 훅 함수다

```ts
adapter.setOnRequestHook(function myHook(req, res, done) {
  console.log('요청 기록');
  done();
});
```

개발자가 특정 시점에 실행하도록 등록한 함수는 `myHook`이다.
`done`이 훅 자체이거나, 개발자가 등록한 작업 전체를 뜻하는 것은 아니다.
훅의 본문이 요청 기록 같은 작업을 수행하고, `done()`은 다음 처리로 넘기는 역할을 한다.

요청이 들어온다는 사건은 실행 계기다. 훅은 그 시점에 실행하도록 등록한 함수다.
따라서 “훅은 트리거이고 done은 트리거 때 수행할 일”로 이해하면 두 역할이 섞인다.

### 20.2 누가 done에 함수를 넣는가

설명용으로 줄인 수정 전 어댑터의 코드는 다음과 같다.

```ts
expressApp.use((req, res, next) => {
  myHook(req, res, next);
});

function myHook(req, res, done) {
  done();
}
```

Express가 미들웨어에 `next`를 제공한다.
어댑터는 그 함수를 사용자 훅의 세 번째 인자로 전달한다.
사용자는 그 매개변수에 `done`이라는 이름을 붙여 받는다.

```text
Express가 제공한 next 함수
  → 어댑터가 myHook의 세 번째 인자로 전달
  → 사용자는 done이라는 이름으로 받음
  → done()으로 전달받은 함수 실행
```

함수도 값이기 때문에 다른 함수의 인자로 전달할 수 있다.
`next`는 함수 자체를 가리키고, `next()`는 그 함수를 실행하는 표현이다.
매개변수 이름을 `continueRequest`로 바꿔도 전달되는 값은 같다.

```ts
adapter.setOnRequestHook((req, res, continueRequest) => {
  continueRequest();
});
```

수정 전에는 `done`이 Express의 `next`와 같은 함수였다.
수정 후에는 어댑터가 `next`를 감싼 `doneOnce`를 넘긴다.
개발자는 여전히 `done()`을 호출하지만 실제로는 한 번만 진행시키는 보호 함수를 실행한다.

### 20.3 next가 훅을 실행하는 것은 아니다

훅은 어댑터의 미들웨어 안에서 이미 실행되고 있다.
그 훅 안에서 `next()` 또는 전달받은 이름인 `done()`을 호출하면 다음 처리로 넘어간다.
다음 대상이 다른 미들웨어일 수도 있고, 이후 Nest 컨트롤러로 연결될 수도 있다.
Nest 컨트롤러로 들어가는 전용 함수는 아니다.

또한 “다음 요청”을 시작하는 것이 아니다. 같은 HTTP 요청의 다음 처리 단계다.

### 20.4 done은 return도 await도 아니다

```ts
async function myHook(req, res, done) {
  done();
  await sendMonitoringData();
}
```

`done()` 자체는 비동기 작업을 선언하는 문법이 아니다.
호출되면 다음 처리가 시작될 수 있고, 그 처리가 비동기 작업을 기다리는 동안 훅의 뒤쪽 코드도 진행될 수 있다.
`await`가 없다고 자동으로 비동기 함수가 되는 것도 아니다.

`done()`은 현재 함수를 끝내는 `return`이 아니므로 그 아래의 전송 코드도 실행된다.
컨트롤러의 DB 조회가 정상이어도 훅의 모니터링 전송은 별도로 실패할 수 있다.
“done이 끝나기 전에 오류”보다는 “done으로 시작시킨 컨트롤러가 아직 일을 마치기 전에 훅 오류”가 정확하다.

모니터링 전송의 성공이 다음 처리의 필수 조건이면 순서를 바꿔야 한다.

```ts
await sendMonitoringData();
done();
```

이 경우 실패하면 `done()`에 도달하지 않고 오류 처리로 넘긴다.
반대로 `done()` 뒤의 작업은 여전히 실행하지만, 뒤늦은 실패로 요청 흐름을 다시 바꾸지 않는다.
작업 코드를 로그로 대체하는 것이 아니라 그 작업의 오류를 로그로 처리하는 것이다.

## 21. 컨트롤러 오류와 훅의 늦은 오류는 다르다

```ts
@Get('users/:id')
async findUser() {
  const user = await this.usersService.findUser();
  if (!user) {
    throw new NotFoundException('사용자가 없습니다');
  }
  return user;
}
```

컨트롤러나 서비스의 오류는 Nest의 예외 처리 흐름으로 연결된다.
사용자 예외 필터가 동작할 수 있고, 기본 처리에서는 HTTP 예외에 맞는 상태나 일반 오류의 500 응답을 만든다.

이번 변경은 이미 완료된 요청 훅의 뒤늦은 오류만 기록하는 것이다.
컨트롤러 자체의 오류를 모두 로그로만 바꾸거나 정상 응답으로 덮지 않는다.
실제 통합 테스트에서 컨트롤러가 `BadRequestException`을 던졌을 때 기존처럼 400 응답이 나오는 것을 확인했다.

오류 처리기를 실행하는 것과 기존 작업을 취소하는 것도 다르다.
훅 오류 때문에 500을 보냈다고 DB 작업이 자동 취소되지는 않는다.
예를 들어 주문 저장은 성공했는데 클라이언트는 500을 받아 다시 시도할 수 있다.
이는 호출 흐름 변경의 가능한 업무 영향을 설명하는 예시이지 실제 주문 DB로 재현한 결과는 아니다.

## 22. 완료 보호 처리의 구조

새 `runRequestHook()`은 요청마다 별도의 완료 상태를 만든다.

```ts
let completed = false;
const doneOnce = (err?: unknown) => {
  if (completed) {
    if (err) {
      this.logger.error(err);
    }
    return;
  }
  completed = true;
  next(err);
};
```

다음 처리를 호출하기 전에 `completed = true`로 바꾼다.
`next()`는 다음 함수를 즉시 호출할 수도 있으므로 외부 코드를 실행하기 전에 내부 상태부터 확정한다.
상태를 어댑터 전체에 두지 않기 때문에 다른 요청의 완료 여부와 섞이지 않는다.

```text
아직 완료하지 않음
  ├─ done() → 정상 진행, 완료로 표시
  └─ 오류 → 오류 진행, 완료로 표시

이미 완료함
  ├─ done() → 무시
  └─ 오류 → 로그 기록, 다시 진행하지 않음
```

Promise는 Express로 반환하지 않고 어댑터가 rejection을 관찰한다.
동기 예외도 같은 보호 경로로 보낸다.
이유 없는 rejection은 `Error('Rejected promise')`로 만들어 정상 진행 신호와 구분한다.

응답 훅은 요청의 `done` 또는 `next`를 쓰지 않는다.
최초 PR에서 추가한 `finish` 리스너의 `try/await/catch`와 오류 기록을 유지했다.
요청 훅과 억지로 같은 완료 메서드에 합치지 않았다.

## 23. Promise 성공도 완료로 볼 것인가

자체검토 중 Fastify와의 차이를 다시 지적했다.
Fastify의 helper에는 Promise 성공 handler가 있다.

```ts
result.then(
  () => doneOnce(),
  err => doneOnce(err || fallbackError),
);
```

따라서 Fastify에서는 `done()`을 호출하지 않은 async 훅도 Promise가 성공하면 진행한다.
관련 테스트는 Promise 성공 후 핸들러가 실행된 다음, 보관해 둔 `done()`을 늦게 호출해도 다시 실행되지 않음을 확인한다.

현재 Express 수정안은 이 부분이 다르다.

```ts
result.then(undefined, err =>
  doneOnce(err || new Error('Rejected promise')),
);
```

Promise 성공만으로 진행하지 않는다. `done()`을 끝내 호출하지 않으면 요청은 계속 대기한다.
이는 이번 보호 처리로 새로 생긴 동작이 아니라 기존 Express의 동작을 유지한 선택이다.

사용자의 피드백은 기술적으로 정확했다.
resolve handler를 추가하면 Promise-only 훅의 대기를 해결할 수 있다.
다만 자동 진행을 추가하면 다음과 같은 기존 호출 순서도 바뀐다.

```ts
adapter.setOnRequestHook(async (req, res, done) => {
  startCallbackWork(() => {
    req.context = '준비 완료';
    done();
  });
});
```

콜백 작업을 시작한 뒤 async 함수가 먼저 끝나면 Promise도 먼저 성공한다.
기존 방식은 콜백이 값을 준비하고 `done()`을 호출할 때 진행한다.
resolve를 완료로 취급하면 값이 준비되기 전에 컨트롤러가 실행될 수 있다.

두 변형을 실제 Express 요청으로 비교했다.

| 검증 | done 기반 진행 | resolve도 완료로 처리 |
| --- | --- | --- |
| 콜백이 나중에 준비한 context | 컨트롤러에서 `ready` | 컨트롤러에서 `missing` |
| Promise 성공 후 done 호출 전 handler 실행 횟수 | 0 | 1 |

완료를 한 번만 허용하는 것과 무엇을 완료 신호로 볼 것인가는 별도 결정이다.
once guard는 중복 호출을 막지만 첫 호출 시점이 빨라지는 것까지 막지는 않는다.

논의 중 자동 진행을 반영하자는 권고와 계약 확인이 필요하다는 권고가 오갔다.
새 근거 없이 권고를 단정적으로 바꿔 혼란을 만든 점은 잘못이었다.
확인한 구현 차이, 메인테이너가 직접 요구한 내용, 의도에 대한 추론을 구분해야 했다.

최종적으로 사용자는 이번 PR에서는 기존 Express 성공 흐름을 유지하기로 결정했다.
메인테이너에게 추가 질문은 보내지 않고 새 PR 본문에 이 선택을 명시하기로 했다.
이는 메인테이너가 Promise 성공 처리를 금지했다고 확정한 것이 아니다.
Fastify와의 완료 규칙 전체가 합의됐다고 주장하지도 않는다.

새 PR에는 Promise 성공만으로 진행하지 않는다는 점을 직접 적었다.
메인테이너가 성공 처리까지 통일하길 요구하면 그때 구현, 주석과 테스트의 기대값을 함께 바꿔야 한다.

## 24. 다시 만든 테스트와 추가 검증

먼저 기존 PR 구현에서 비동기 라우트를 사용해 리뷰의 문제를 재현했다.
`done()` 이후 동기 throw와 Promise rejection은 정상 응답 대신 500,
중복 `done()`은 정상 응답 대신 404로 실패했다.
수정 후에는 세 경우 모두 정상 응답을 유지했다.

단위 테스트만으로 끝내지 않고 실제 Nest 컨트롤러가 있는 통합 테스트를 추가했다.
컨트롤러는 `setImmediate`를 기다려 아직 처리 중인 구간을 만들고 나중에 응답한다.
임의로 긴 시간을 기다리는 성공 조건 대신 실행 순서를 만들고 결과를 검증한다.

완료 guard를 우회해 이전처럼 Promise를 Express에 반환하면 통합 테스트도 500·404로 실패했다.
변경을 다시 적용한 뒤 통과하는 것까지 확인했다.

### 24.1 POST의 업무 처리 횟수

POST 컨트롤러는 입력 ID를 배열에 추가하고 201 응답을 보낸다.
실제 DB를 붙인 테스트는 아니며, 한 번만 실행되어야 하는 저장 동작을 배열 기록으로 표현했다.

async 훅이 `done()`을 호출한 뒤 성공, 실패 또는 중복 콜백을 발생시키는 세 경우를 확인한다.
HTTP 결과만 보지 않고 `createdIds`에 정확히 한 항목만 생겼는지도 검사한다.

### 24.2 늦게 도착한 두 번째 done

첫 `done()`으로 요청을 끝낸 뒤, 보관해 둔 콜백을 나중에 다시 호출한다.
라우트 뒤의 별도 미들웨어가 실행되지 않았는지 확인한다.
이미 응답이 끝나서 외부 HTTP 결과가 달라지지 않더라도 체인이 다시 진행되면 실패하도록 했다.

### 24.3 응답 완료 후 rejection

응답을 실제로 받은 다음 보관한 Promise를 reject시킨다.
일반 Error와 이유 없는 rejection을 각각 확인한다.

처음 임시 검증에서는 로그가 한 번 기록되는지만 확인했다.
하지만 Nest의 기본 오류 처리기도 오류를 기록하므로 이것만으로는 올바른 경로인지 구분할 수 없었다.
정식 테스트에서는 이미 완료된 응답에 대해 어댑터의 `end()`가 다시 호출되지 않는지도 확인한다.

이전 구현으로 실행하면 로그는 남아도 `end()`가 한 번 호출되어 테스트가 실패한다.
오류를 보았다는 사실과 요청 흐름을 건드리지 않았다는 사실은 다른 검증이다.

### 24.4 동시 요청별 완료 상태

서로 다른 ID를 가진 POST 세 개를 동시에 보낸다.
각 훅의 콜백을 저장하고 세 요청이 모두 들어온 뒤 `b`, `a`, `c` 순서로 진행시킨다.
세 응답이 모두 201이고 각 ID가 한 번씩 기록되는지 확인한다.
요청 도착 순서에 의존하지 않고 완료 상태가 서로 독립적임을 검사한다.

추가한 일곱 테스트 중 다섯 개는 이전 PR 구현에서 실패했다.
나머지 두 개는 성공 경로와 동시 요청의 기존 동작이 유지되는지를 확인하는 검사다.
모든 새 테스트가 반드시 과거 코드에서 실패한다고 부풀려 기록하지 않는다.

## 25. 최신 upstream 반영과 후속 PR 제출

발행 직전 사용자가 자체검토하며 코드를 수정했을 수 있다고 알려서 작업 트리를 다시 읽었다.
추적 중인 어댑터·단위 테스트와 새 통합 테스트를 모두 확인했고,
앞서 검증한 변경 외에 추가로 다른 수정은 발견하지 못했다. 기존 파일을 되돌리거나 덮어쓰지 않았다.

최초 기준 이후 upstream에 새 변경들이 들어와 있었다.
기존 원격 브랜치는 그대로 두고 `fix/express-hook-completion` 브랜치를 만들었다.
변경을 커밋으로 보존한 뒤 `7036f3dc0` 위로 두 커밋을 충돌 없이 rebase했다.
새 브랜치를 포크에 push했으므로 기존 PR 브랜치를 강제 갱신하지 않았다.

의존성도 최신 lockfile에 맞춰 설치한 뒤 다시 검증했다.
기존 기준에서 4,002개였던 전체 단위 테스트는 upstream의 테스트 증가로 4,121개가 됐다.
테스트 수의 증가 전체가 이번 PR의 추가분이라는 뜻은 아니다.

| 최종 검증 | 결과 |
| --- | --- |
| 전체 unit | 315개 파일, 4,121개 통과 |
| Nest 애플리케이션 관련 integration | 16개 파일, 117개 통과 |
| 이번 훅 단위 테스트 | 13개 |
| 이번 훅 통합 테스트 | 10개 |
| `npm run build` | 통과 |
| `npm run lint:ci` | 통과, 기존 websocket 경고 1개 |
| 추가 integration 파일 lint | 통과 |
| 변경 파일 Prettier / diff 공백 검사 | 통과 |
| Docker 기반 전체 integration | 실행하지 않음 |

커밋 및 GitHub 작성 계정은 `GiHoon1123`으로 확인했다.
후속 PR은 #18110이며 base는 upstream `master`, head는 포크의 새 브랜치다.
세 파일에 총 496줄 추가, 3줄 삭제이고 대부분은 회귀 테스트다.
최초 수정 커밋과 완료 보호 처리 커밋 두 개가 포함된다.

PR 본문에는 #18094와 해당 리뷰를 연결하고 `Closes #18093`을 적었다.
기존 PR에 별도의 역방향 코멘트는 남기지 않았다. 새 PR에서 이전 맥락을 확인할 수 있게 하기로 한 결정이다.
Promise 성공만으로 요청을 진행하지 않는다는 점도 본문에 명시했다.

문서는 재작업 사례의 운영 규칙에 따라 새 활성 PR 번호로 파일명을 바꾸고 `open`에 유지했다.
이전 613줄의 설명과 시행착오는 삭제하지 않고, 현재 상태와 링크를 갱신한 뒤 이 후속 기록을 이어 붙였다.

## 26. 후속 PR 최초 CI 확인 — 2026-10-12

PR #18110의 `aafb224d22eb89e71ef070d22032c30a4958fdbf` 기준으로 CodeQL과 PR content guard가 통과했다.
CircleCI `build-and-test`는 진행 중이고, 이 시점에는 새 리뷰나 코멘트가 없었다.
원격 전체 CI 통과나 메인테이너의 완료 규칙 동의를 아직 확인한 것은 아니다.

## 27. 메인테이너의 마지막 수정과 머지 — 2026-10-13 확인

후속 [PR #18110](https://github.com/nestjs/nest/pull/18110)이 머지됐다.
GitHub의 병합 시각은 `2026-10-12T09:37:04Z`, 한국시간으로 10월 12일 18시 37분 4초다.
확인한 날짜는 10월 13일이지만 실제 머지는 전날 이루어졌다.
병합한 사람은 `kamilmysliwiec`이고, 머지 커밋은
[`c9c251ca1`](https://github.com/nestjs/nest/commit/c9c251ca1ad14a467e62980127ad3367bbb027ca)이다.
연결한 [이슈 #18093](https://github.com/nestjs/nest/issues/18093)도 같은 날 18시 37분 6초에
`completed` 사유로 닫혔다.

### 27.1 제출한 코드 그대로 머지된 것은 아니다

최초 제출 기록의 마지막 커밋은 `aafb224d2`였다.
메인테이너가 10월 12일 18시 27분 47초 KST에
[`f99178258`](https://github.com/nestjs/nest/commit/f9917825857d508c59ab8c2a4e088ed06c8a774b)를 추가했다.
커밋 제목은 `test(express): cover hook errors reaching the exception filters`다.
별도의 새 리뷰 문장만 볼 것이 아니라 이 커밋의 실제 diff까지 읽어야 최종 결과를 알 수 있다.

완료 보호 처리의 핵심은 유지됐다.
요청 훅의 Promise를 Express로 반환하지 않고 어댑터 안에서 처리하며,
첫 완료 이후의 실패는 로그로 남기고 요청을 다시 진행시키지 않는다.
응답 훅의 오류를 `finish` 리스너에서 잡아 기록하는 처리도 유지됐다.
대신 테스트 범위를 정리하고 호출 구조와 타입, 주석을 다듬었다.

### 27.2 요청 훅의 실패가 Nest 예외 필터까지 도달하는지 검증

우리가 제출한 통합 테스트는
`integration/nest-application/lifecycle-hooks/e2e/express.spec.ts`의 207줄이었다.
메인테이너는 이 파일을 제거하고
`integration/nest-application/request-hooks/e2e/express.spec.ts`에 63줄짜리 테스트를 추가했다.
최종 통합 테스트는 두 경우에 집중한다.

1. 요청 훅이 `done()`을 호출하기 전에 `UnauthorizedException`으로 실패하면
   실제 Nest 애플리케이션이 401 응답과 `{ message: 'Unauthorized', statusCode: 401 }` 본문을 반환한다.
2. 요청 훅이 `done()`을 호출한 다음 실패하면, 비동기 컨트롤러의 200 응답과 `ok` 본문은 유지되고
   해당 오류만 한 번 기록된다.

첫 테스트는 단순히 Express의 오류 미들웨어가 호출되는지 확인하는 것보다 한 단계 더 나아간다.
Nest에서 정의한 예외의 HTTP 상태와 본문이 실제 응답에 반영되는지 확인한다.
어댑터가 오류를 발견하는 것과 Nest의 예외 처리 경로까지 올바르게 연결하는 것은 서로 다른 검증이다.

기존 통합 테스트의 컨트롤러가 `BadRequestException`을 던지는 경우도 유용한 보존 검사였지만,
그것만으로는 요청 훅에서 발생한 실패가 같은 예외 처리 경로에 들어가는지 증명하지 못한다.
메인테이너의 새 테스트는 이번에 수정한 경계에서 직접 오류를 발생시킨다.

두 번째 테스트의 컨트롤러는 `setImmediate`를 기다린 뒤 응답한다.
따라서 훅이 실패하는 시점에 요청 처리가 아직 진행 중인 조건을 유지한다.
바로 응답을 끝내 버리는 테스트로 돌아간 것이 아니다.

### 27.3 단위 테스트와 호출 구조 정리

단위 테스트에서는 다음 두 항목이 제거됐다.

- async 요청 훅이 `done()`을 호출하고 정상 종료했을 때 다시 진행하지 않는 검사.
- 응답 완료 후 보관한 `done()`을 다시 호출했을 때 라우트 뒤 미들웨어가 실행되지 않는 검사.

메인테이너는 커밋 설명에서 수정 없이도 통과하는 단위 테스트를 덜어냈다고 적었다.
이전에 우리가 보존 검사나 임시 실험에서 확인한 내용을 전부 무효라고 해석할 필요는 없다.
다만 로컬에서 넓게 조사한 사례를 모두 저장소의 영구 회귀 테스트로 남겨야 하는 것도 아니다.
수정이 없을 때 실패하는지, 같은 조건을 다른 테스트가 이미 검증하는지,
각 테스트가 유지 비용에 맞는 정보를 제공하는지 따로 판단해야 한다.

`runRequestHook()`에는 등록된 훅을 별도 인자로 넘기지 않도록 정리했다.
최종 코드는 다음처럼 어댑터에 저장된 훅을 직접 사용한다.

```ts
this.runRequestHook(req, res, next);

// runRequestHook 내부
const result = this.onRequestHook!.call(this, req, res, doneOnce);
```

`!`는 TypeScript의 non-null assertion이다. 실행 시 새 검사를 하는 문법은 아니다.
호출부에서 훅이 있는 경우에만 이 메서드로 들어온다는 조건을 전제로 한다.
`.call(this, ...)`를 유지했으므로 훅의 `this`를 어댑터로 바인딩하는 동작은 그대로다.

이유 없는 Promise rejection의 대체 오류 문구도 `Rejected promise`에서
`Request hook failed`로 바뀌어 동기 실패의 대체 문구와 같아졌다.
앞 절에 남아 있는 이전 문구는 제출 당시 코드의 기록이다.

### 27.4 done(err) 타입과 Promise 성공 규칙 명시

등록된 훅의 필드와 공개 setter 양쪽에서 콜백 타입이 바뀌었다.

```ts
// 제출 당시
done: () => void

// 최종 머지본
done: (err?: Error) => void
```

오류 없이 `done()`을 호출하는 기존 사용법은 유지하면서,
`done(new Error('...'))`로 실패를 전달하는 사용법도 타입으로 표현한다.
이는 메인테이너가 마지막 커밋에서 보강한 부분이다.
최종 변경을 설명할 때 공개 타입은 전혀 바뀌지 않았다고 적으면 부정확하다.

Promise가 성공했다는 이유만으로 요청을 진행시키지 않는 선택도 최종 코드에 남았다.
메인테이너는 setter 주석에 반환된 Promise의 성공은 요청을 계속 진행시키지 않는다는 설명을 추가했다.
`done()` 호출을 기다리는 단위 테스트도 유지했다.

앞선 논의에서는 우리가 보존하기로 선택한 동작이었고, 메인테이너의 동의는 미확인이었다.
최종적으로는 이 동작을 명시한 주석과 테스트가 포함된 코드가 머지됐다.
이를 Fastify의 모든 완료 규칙까지 통일하기로 합의했다고 확대해 해석하지는 않는다.

### 27.5 최종 변경량과 CI

최종 PR은 커밋 3개, 파일 3개, 318줄 추가·5줄 삭제다.
25절의 496줄 추가·3줄 삭제와 테스트 수는 메인테이너 수정 전 제출본의 기록으로 남긴다.

최종 head `f9917825857d508c59ab8c2a4e088ed06c8a774b`에서 GitHub가 보고한 검사 결과는 다음과 같다.

| 검사 | 결과 |
| --- | --- |
| build-and-test | success |
| CodeQL | success |
| Scan changed files | success |
| Analyse | success |

이번 문서 갱신에서는 GitHub의 최종 커밋, 변경 diff와 검사 결과를 확인했다.
메인테이너 수정본을 로컬에서 다시 빌드하거나 테스트한 것은 아니다.
앞서 기록한 4,121개 단위 테스트와 117개 통합 테스트 통과를 최종 head에서 새로 실행한 결과처럼 쓰지 않는다.
머지는 확인했지만 npm 배포 버전까지 확인한 것은 아니다.

### 27.6 기록의 최종 위치

최초 PR #18094의 종료, 그 과정에서 잘못 판단한 내용, 후속 PR #18110의 수정과 검증 기록은 그대로 남겼다.
상단 상태와 최종 커밋을 갱신하고 이 절을 덧붙인 뒤 문서를 `typescript/open`에서 `typescript/merge`로 옮겼다.
기여 방법론의 문서 링크도 새 위치로 연결했다.

이번 작업은 버그를 직접 찾았다는 데서 끝나지 않았다.
첫 수정에서 놓친 요청 완료 규칙을 리뷰로 확인했고, 그 규칙을 지키는 후속 수정이 머지됐다.
마지막에는 메인테이너가 테스트를 더 줄이면서도 Nest 예외 필터라는 중요한 연결 지점을 보강했다.
조사 범위를 넓게 잡는 것과 최종 변경에 모든 조사 사례를 남기는 것은 별개의 판단이라는 점도 기록해 둔다.
