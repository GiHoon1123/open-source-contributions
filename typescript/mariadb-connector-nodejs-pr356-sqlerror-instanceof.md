# [OSS 기여] mariadb-corporation/mariadb-connector-nodejs - ESM 네임스페이스 import 때문에 `SqlError`가 constructor를 잃어버리는 회귀 수정 (PR #356) — 🟢 Merged

> 오픈소스 기여 기록 및 학습 노트. 이 문서는 PR #356(이슈 #348) 하나만 다룬다. **2026-07-31 제출, 2026-07-31 Merged.**

---

## 0. 관련 링크 모음

| 종류 | 링크 | 상태 |
| --- | --- | --- |
| 대상 이슈 | https://github.com/mariadb-corporation/mariadb-connector-nodejs/issues/348 | Closed (머지되며 함께 닫힘) |
| 제출한 PR | https://github.com/mariadb-corporation/mariadb-connector-nodejs/pull/356 | Merged |

- 패키지: `mariadb` (npm) — MariaDB Corporation 공식 Node.js 커넥터
- 작업 브랜치: `fix-sqlerror-instanceof` (fork: GiHoon1123)
- 커밋: `f7009c4` — 4개 파일, +13/-3
- 검증: `test:imports`(실제 esbuild 빌드 결과물로 ESM/CJS 양쪽 확인), `test:cts`, `test:lint`, `test:unit` 206개 전부 통과

---

## 1. 후보를 어떻게 찾았나

이더리움js 쪽(PR #4355/이슈 #4356) 작업과 같은 세션에서, 병렬 에이전트로 새 후보를 찾을 때 함께 발견됐다. mysql2/drizzle-orm처럼 "이미 너무 활발해서 눈에 띄는 이슈는 전부 경쟁 PR이 붙어있는" 저장소들을 지나, mariadb-connector-nodejs에서 걸린 두 이슈(#348, #349) 중 근본 원인이 명확하고 경쟁 PR이 없는 쪽을 먼저 골랐다.

### 공식성 검증 — 근거로

- GitHub org `mariadb-corporation`의 blog 필드가 `https://mariadb.com`으로 연결
- npm 패키지명이 그냥 `mariadb` — 생태계 정석 이름
- `package.json`의 author가 `Diego Dupin <diego.dupin@mariadb.com>` — **회사 도메인 이메일로 직접 확인**, 제일 강력한 근거
- 주간 다운로드 약 69.7만

---

## 2. 문제 — ESM import 방식 두 가지가 어긋난 것

`lib/misc/errors.js`는 이렇게 내보낸다:

```js
export default class SqlError extends Error { ... }
```

`export default`로 내보낸 걸 가져오는 올바른 방법은 default import(`import SqlError from ...`)인데, `promise.js`/`callback.js`는 둘 다 네임스페이스 import(`import * as SqlError from ...`)를 쓰고 있었다. 이러면 `SqlError`엔 클래스가 아니라 **"이 파일이 export하는 모든 것을 담은 객체"**가 담긴다:

```js
SqlError = {
  default: [class SqlError extends Error],   // 진짜 클래스는 여기 안에
  client: { ... },
  createError: [Function],
  ...
}
```

`instanceof`는 오른쪽에 반드시 함수(생성자)가 와야 하는데 이건 평범한 객체라서:

```
TypeError: Right-hand side of 'instanceof' is not callable
```

직접 재현해서 이슈에 적힌 것과 한 글자도 다르지 않은 에러 메시지까지 확인했다.

**타입 선언은 이미 constructor를 기대하고 있었다** — `types/share.d.ts:424`: `declare const SqlError: SqlErrorConstructor`. 즉 런타임과 타입이 서로 어긋나있던, 명백한 회귀였다.

### 왜 하필 지금 터졌나

저장소가 CJS → ESM 마이그레이션을 진행하는 과정에서 export 방식이 바뀌었는데, `promise.js`/`callback.js`의 import 문이 예전 방식 그대로 남아 어긋난 것. 같은 마이그레이션 때문에 생긴 형제 회귀(#346, #347)는 PR #355로 이미 고쳐졌는데, 이 이슈(#348)만 남아있었다.

---

## 3. 재현

```js
import { SqlError } from './promise.js';
console.log(typeof SqlError, SqlError);
// object, [Module: null prototype] { default: [class SqlError ...], client: {...}, ... }

new Error('test') instanceof SqlError
// TypeError: Right-hand side of 'instanceof' is not callable
```

내부 어디에서도 `SqlError.client`, `SqlError.createError`처럼 네임스페이스 형태로 쓰는 코드가 없다는 것도 `grep`으로 확인해서, default import로 바꿔도 다른 곳이 안 깨진다는 근거를 확보했다.

---

## 4. 수정

```diff
- import * as SqlError from './lib/misc/errors.js'
+ import SqlError from './lib/misc/errors.js'
```

`promise.js`, `callback.js` 두 파일 각각 한 줄씩.

### 왜 기존 테스트가 이걸 못 잡았나

`test/import/esm.mjs`/`cjs.cjs`에 이미 export 확인용 스모크 테스트가 있었는데(형제 이슈 #346 회귀 방지용으로 만들어진 것):

```js
assert.ok(SqlError, 'named SqlError missing')   // 네임스페이스 객체도 truthy라 통과해버림
```

**"존재하냐"만 확인하고 "생성자냐"는 안 물어봤다.** 그래서 회귀 테스트에 두 가지를 추가했다:

```js
assert.strictEqual(typeof SqlError, 'function', ...)
assert.ok(new SqlError('x') instanceof SqlError, ...)
```

`promise`/`callback` 양쪽, ESM/CJS 양쪽 전부 커버하도록 4곳(cjs.cjs의 공용 `check()` 함수 1곳 + esm.mjs의 top-level/`/callback` subpath 2곳)에 나눠 추가했다.

---

## 5. 검증

- **disable-and-reconfirm 두 번**:
  1. fix 적용 전 → 강화된 테스트가 정확히 예상한 에러(`SqlError is not a constructor`)로 실패 → fix 적용 후 통과
  2. `callback.js`만 다시 되돌려서, **CJS 체크에 기대지 않고 ESM 검증 자체가 독립적으로 잡는지** `node test/import/esm.mjs`로 단독 실행해서 재확인 → 정확히 새로 추가한 `/callback named SqlError is not a constructor` 라인에서 실패
- `npm run test:imports` — 실제 esbuild로 빌드한 `dist/*.cjs` 결과물로 ESM/CJS 양쪽 확인, 통과
- `npm run test:cts`, `npm run test:lint` 통과
- `npm run test:unit` 206개 전부 통과 (DB 연결 불필요한 순수 유닛 테스트)
- `git diff --check` 클린

### 중간 실수 하나

disable-and-reconfirm 첫 시도에서 `git checkout HEAD~0 -- promise.js callback.js`(사실상 아무 일도 안 하는 명령) 다음에 경로 지정 없이 `git stash`를 실행해서, **fix뿐 아니라 방금 추가한 테스트 강화분까지 같이 스태시돼버린** 적이 있었다. 그 상태로 돌리니 당연히 "OK"가 나왔는데, 이건 fix와 테스트가 둘 다 없어져서 나온 "가짜 통과"였다. 알아채고 `git stash pop` 후 다시 정확하게 재현해서 바로잡았다.

---

## 6. 리뷰 — 자체 검토로 잡힌 것

PR을 올리기 전, 자체 검토로 직접 다음을 짚었고 전부 반영했다:

1. **ESM `/callback` 서브패스의 `SqlError`가 검증에서 빠져있음** — `esm.mjs`가 `/callback` subpath에서 `createPool`만 확인하고 `SqlError`는 import조차 안 하고 있었다(CJS 쪽은 이미 양쪽 다 확인 중이었음). `SqlErrorCb` import + `instanceof`/constructor 검증 3줄 추가, disable-and-reconfirm으로 독립적으로 잡히는 것까지 재확인.
2. **PR 본문 방향 조언** — "다운로드 수/공식성 얘기는 빼고, 'runtime export가 타입 선언대로 constructor가 되도록 고친다'에만 집중하라"는 조언을 그대로 반영해서 PR 본문을 작성함(이 문서 1절의 공식성 근거는 후보 선정 기록용으로만 남기고, PR 본문에는 안 넣음).

---

## 7. 핵심 교훈

1. **`export default` + `import * as`는 문법 에러가 안 나서 조용히 잘못된 값을 만든다.** TypeScript 타입 체크 없이 순수 JS로 마이그레이션하면 이런 종류의 어긋남이 컴파일 타임에 안 걸리고 런타임까지 살아남는다.
2. **"존재 확인(truthy)"과 "형태 확인(타입/instanceof)"은 다른 수준의 테스트다.** 네임스페이스 객체도 truthy라서, 진짜 필요한 검증(생성자인가, instanceof가 되는가)을 안 하면 정확히 이런 버그를 놓친다.
3. **disable-and-reconfirm을 할 때 `git stash`에 경로를 안 지정하면 의도치 않게 다른 변경사항까지 같이 날아갈 수 있다.** 이번에 그것 때문에 한 번 "가짜 통과"를 봤다 — 매번 stash 후에는 diff를 다시 확인하는 습관이 필요하다.
4. **PR 본문에 넣을 근거와, 후보를 고를 때 참고한 근거는 다르다.** 공식성/다운로드 수는 "이 저장소를 왜 골랐는가"의 근거지, "이 버그를 왜 고쳐야 하는가"의 근거가 아니다 — 메인테이너가 읽을 PR 본문은 후자에만 집중해야 한다.

---

## 8. 다음에 채워야 할 것 (결과가 나오면)

- [ ] PR #356 메인테이너 리뷰 결과
- [ ] 같은 저장소(mariadb-connector-nodejs)에서 추가로 찾은 후보 — #349(pipelining hang) 진행 여부는 이 문서 갱신 시 반영
