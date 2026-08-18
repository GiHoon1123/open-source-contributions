# [OSS 기여] brianc/node-postgres - client.query() config 객체 재사용 시 조용한 undefined 반환 수정 (PR #3720) — 🟢 Merged

> 오픈소스 기여 기록 및 학습 노트. 이 문서는 PR #3720 하나만 다룬다. **2026-08-12 원작자 `brianc`가 직접 머지.**
>
> 같은 저장소에 이미 다른 PR([[node-postgres-pr3708-rows-query-error-sync]])을 올려둔 상태에서, 별도 이슈를 새로 찾아 진행한 두 번째 기여다.

---

## 1. 개요

### 1.1 프로젝트 소개

- **저장소**: https://github.com/brianc/node-postgres
- **npm 패키지명**: `pg` — Node.js PostgreSQL 클라이언트의 사실상 표준. Sequelize, TypeORM, knex 등 대부분의 상위 ORM이 내부적으로 이 라이브러리 위에 얹혀 있다.
- **건드린 부분**: `packages/pg/lib/utils.js`의 `normalizeQueryConfig()` — 문자열/객체/콜백 등 다양한 형태로 들어오는 `client.query()` 인자를 하나의 config 객체로 정리해주는 내부 함수.

### 1.2 이슈 (#2651)

**제목**: "client.query() ... silently fail" (config 재사용 시 무음 실패)

리포터: `client.query()`를 프로미스 방식(콜백 없이)으로 쓸 때, 내부의 `normalizeQueryConfig()`가 `config.callback`을 설정하는데, **같은 config 객체로 이후에 다시 `client.query()`를 호출하면 아무 에러 메시지 없이 그냥 `undefined`가 반환된다.** 원문: "Passed config should never be modified, this is a client-owned object."

메인테이너(`brianc`, 원작자)가 직접 동의했다: "I agree with you. Do you have code to reproduce this? I can fix p quick." — 그런데 재현 코드가 안 달렸고, 이후 고쳐지지 않은 채 방치됐다. 이후 시간차를 두고 두 명(`jbilous`, `luni-moon`)이 독립적으로 같은 증상("콜백에서 프로미스로 바꾸다가 가끔 undefined가 온다")을 재확인하는 코멘트를 남겼다.

### 1.3 관련 링크

| 구분 | 링크 |
|---|---|
| 원본 저장소 | https://github.com/brianc/node-postgres |
| 내 Fork | https://github.com/GiHoon1123/node-postgres |
| 이슈 (#2651) | https://github.com/brianc/node-postgres/issues/2651 |
| PR (#3720) | https://github.com/brianc/node-postgres/pull/3720 |
| 리뷰 코멘트 | https://github.com/brianc/node-postgres/pull/3720#discussion_r3653996350 |
| 리뷰 답변 | https://github.com/brianc/node-postgres/pull/3720#discussion_r3654758091 |
| 작업 브랜치 | `fix-2651-query-config-mutation` |
| 커밋 | `ea6cf43`, `f0bbfcfc`, `b4d11e47` |
| 상태 | 🟢 Merged (2026-08-12) |

`brianc`가 머지하며 남긴 코멘트: "groovy! thanks for the code AND for the existing PR review to catch a potentially breaking change. 😄" — 코드뿐 아니라 다른 PR([[node-postgres-pr3708-rows-query-error-sync]])에서 breaking change 가능성을 리뷰로 잡아준 것까지 같이 언급했다.

### 1.4 한 줄 요약

> `normalizeQueryConfig()`가 호출자 소유의 config 객체를 복사하지 않고 그 위에 직접 `callback`/`values`를 써넣어서, 콜백 방식으로 한 번 쓴 config를 나중에 프로미스 방식으로 재사용하면 이전 콜백이 남아있는 채로 오판되어 Promise가 아예 안 만들어지고 `undefined`가 반환됐다. 최종 수정은 config를 새 객체로 복사하되, 단순 object spread가 아니라 기존 prototype과 property descriptor까지 보존하는 `cloneQueryConfig()`를 사용한다.

---

## 2. 배경지식

### 2.1 `client.query()`의 다양한 호출 방식

```js
client.query('SELECT 1', callback)                      // 문자열 + 콜백
client.query({ text: 'SELECT 1', callback })             // 객체 안에 콜백 내장
client.query({ text: 'SELECT 1' }, callback)              // 객체 + 콜백 별도
await client.query({ text: 'SELECT 1' })                  // 콜백 없음 = 프로미스 방식
```

콜백이 올 수 있는 자리가 두 곳(객체 안 / 별도 인자)이라, 내부적으로 이걸 하나의 형태로 정리하는 함수가 필요하다. 그게 `normalizeQueryConfig(config, values, callback)`이고, "config, values, callback을 다 받아서 `{ text, values, callback }` 모양의 객체 하나로 합쳐 돌려준다"가 역할이다.

### 2.2 콜백 방식 vs 프로미스 방식을 가르는 유일한 기준

`Client.query()`(`packages/pg/lib/client.js`)의 판단 로직:

```js
query = new Query(config, values, callback)
if (!query.callback) {                    // 콜백이 "없으면"
  result = new this._Promise((resolve, reject) => {
    query.callback = (err, res) => (err ? reject(err) : resolve(res))
  })...
}
```

`query.callback`이 있으면 콜백 방식(이미 처리할 사람이 있다고 보고 Promise를 안 만듦), 없으면 프로미스 방식. **이 판단은 오직 `query.callback`이라는 값 하나만 본다** — 이번 버그가 터지는 지점이 바로 여기다.

---

## 3. 원인 분석 — 코드 레벨 추적

`normalizeQueryConfig` (수정 전, `packages/pg/lib/utils.js:144`):

```js
function normalizeQueryConfig(config, values, callback) {
  config = typeof config === 'string' ? { text: config } : config   // 객체면 복사 없이 원본 그대로
  if (values) {
    if (typeof values === 'function') {
      config.callback = values           // ★ 원본 객체에 직접 씀
    } else {
      config.values = values
    }
  }
  if (callback) {
    config.callback = callback           // ★ 여기도 원본에 직접 씀
  }
  return config
}
```

객체가 넘어오면 `config = config`(복사 없음)이라서, 이 함수 안의 `config`와 호출자가 들고 있는 원본 객체는 **완전히 같은 메모리상 물건**이다. 여기에 값을 쓰면 호출자의 원본도 그대로 오염된다.

**타임라인으로 본 버그 재현 경로**:

```
① client.query(cfg, callback1)
   → normalizeQueryConfig(cfg, callback1, undefined) 내부에서
     cfg.callback = callback1   (cfg 자체가 오염됨, 콜백 방식으로는 정상 동작)

② (한참 뒤) await client.query(cfg)   ← 이번엔 콜백을 안 넘김
   → new Query(cfg, undefined, undefined)
   → normalizeQueryConfig(cfg, undefined, undefined)
     values도 callback도 인자로 안 왔으니 이 함수는 아무것도 안 하고 cfg를 그대로 반환
     → 근데 cfg엔 ①에서 남은 callback1이 이미 붙어있음 (지우는 코드가 없음)
   → Query 생성자: this.callback = config.callback = callback1 (유령)
   → Client.query(): if (!query.callback) → callback1이 truthy → false → Promise 생성 스킵
   → result는 함수 맨 위 `let result`의 기본값 undefined 그대로 반환
```

`await`는 반환된 게 Promise가 아니라 그냥 `undefined`이므로 기다릴 대상이 없어 즉시 통과한다. 콜백(`callback1`)은 실제로는 나중에 ②번 쿼리의 진짜 결과와 함께 호출되지만, ②번을 호출한 쪽(`await`)은 이미 `undefined`를 받고 다음 줄로 넘어간 뒤라 그 결과를 받을 방법이 없다 — 데이터가 유실되는데 에러조차 안 난다.

---

## 4. 실제 재현 (Docker Postgres)

로컬 Postgres 컨테이너에 실제로 연결해서 확인:

```
--- ① 콜백 스타일로 한 번 사용 ---
callback got rows: [ { n: 1 } ]
--- ② 같은 cfg 객체로 프로미스 스타일 재사용 ---
typeof result: undefined   result === undefined: true
★★★ 재현 성공: config 재사용 시 조용히 undefined 반환 (에러 없음) ★★★
```

수정 후 동일 스크립트:

```
--- ② 같은 cfg 객체로 프로미스 스타일 재사용 ---
typeof result: object   result === undefined: false
정상: Promise 반환됨, rows = [ { n: 1 } ]
```

---

## 5. 수정 내용

### 5.1 첫 수정: 호출자 config를 직접 만지지 않기

```diff
 function normalizeQueryConfig(config, values, callback) {
   // can take in strings or config objects
-  config = typeof config === 'string' ? { text: config } : config
+  // the config object is caller-owned and must never be mutated -- copy it
+  // so callback/values assigned below don't leak into the caller's object
+  // and corrupt subsequent, unrelated invocations that reuse the same config
+  config = typeof config === 'string' ? { text: config } : { ...config }
   if (values) {
```

첫 PR은 한 줄(`{ ...config }`)로 얕은 복사본을 만들어서 그 위에만 쓰도록 바꿨다. 이슈 #2651의 핵심인 "호출자 객체를 오염시키지 말라"는 이 방식으로 해결된다. 문자열이 아닌 입력에 대해 항상 새 객체를 반환하므로, `callback`/`values`를 붙여도 호출자가 들고 있는 원본 config에는 남지 않는다.

다만 리뷰에서 이 방식의 호환성 문제가 지적됐다.

### 5.2 리뷰에서 지적받은 문제: object spread는 prototype을 복사하지 않는다

리뷰어(`charmander`) 코멘트:

> This is a potential breaking change for query objects that don't have own enumerable query config properties.

핵심은 이거다. `{ ...config }`는 "그 객체 자신이 직접 가지고 있는 열거 가능한 속성"만 복사한다. 자바스크립트 객체는 자기 자신에게 없는 속성을 prototype chain을 따라 위쪽 객체에서 찾을 수 있는데, object spread는 그 위쪽까지 같이 가져오지 않는다.

예를 들면 이런 query config 객체가 가능하다:

```js
class QueryConfig {
  constructor() {
    this._text = 'SELECT 1'
  }

  get text() {
    return this._text
  }
}

const original = new QueryConfig()
original.text // 'SELECT 1'
```

겉으로는 `original.text`가 잘 읽힌다. 하지만 `text`는 `original` 객체 자신에게 있는 일반 속성이 아니라 `QueryConfig.prototype`에 있는 getter다.

```js
Object.hasOwn(original, 'text') // false

const copied = { ...original }
copied.text // undefined
```

이렇게 되면 `normalizeQueryConfig()`가 원본을 오염시키지 않는 대신, 원래는 동작하던 query config 객체를 망가뜨린다. `pg`는 오래된 범용 라이브러리라 사용자가 꼭 plain object만 넘긴다고 단정할 수 없다. 클래스 인스턴스, getter 기반 객체, prototype을 이용한 query object가 들어올 수 있고, 이런 형태를 깨뜨리면 "버그 수정"이 호환성 깨짐이 된다.

### 5.3 최종 수정: prototype과 descriptor를 보존하는 복사

리뷰 반영 후 최종 코드는 이렇게 바꿨다:

```js
function normalizeQueryConfig(config, values, callback) {
  // can take in strings or config objects
  // Copy config so normalization does not mutate the caller's object.
  config = typeof config === 'string' ? { text: config } : cloneQueryConfig(config)
  if (values) {
    if (typeof values === 'function') {
      config.callback = values
    } else {
      config.values = values
    }
  }
  if (callback) {
    config.callback = callback
  }
  return config
}

function cloneQueryConfig(config) {
  if (config == null) {
    return config
  }
  return Object.defineProperties(Object.create(Object.getPrototypeOf(config)), Object.getOwnPropertyDescriptors(config))
}
```

이 코드는 두 가지를 동시에 만족한다.

1. `clone !== config`라서 `callback`/`values`를 써도 호출자의 원본 객체는 오염되지 않는다.
2. `clone`이 원본과 같은 prototype을 바라보므로, prototype에 있는 `text` getter 같은 기존 접근 방식이 그대로 동작한다.

`cloneQueryConfig()`를 쪼개면 이렇다.

```js
Object.getPrototypeOf(config)
```

원본 객체가 어떤 prototype을 타고 속성을 찾는지 가져온다. 클래스 인스턴스라면 보통 `QueryConfig.prototype`이 나온다.

```js
Object.create(Object.getPrototypeOf(config))
```

같은 prototype을 바라보는 새 객체를 만든다. 이 객체는 원본과 다른 객체지만, 원본과 같은 방식으로 prototype getter/method를 찾을 수 있다.

```js
Object.getOwnPropertyDescriptors(config)
```

원본 객체 자신이 직접 가진 속성들을 descriptor 형태로 가져온다. 단순 값뿐 아니라 `enumerable`, `writable`, `configurable`, getter/setter 같은 속성의 성격까지 포함한다.

```js
Object.defineProperties(newObject, descriptors)
```

그 descriptor들을 새 객체에 그대로 정의한다. 그래서 own property도 최대한 원래 모양을 유지한다.

결국 이 수정은 "복사는 하되, 사용자가 만든 config 객체의 모양은 최대한 유지한다"는 쪽이다. 단순 object spread보다 조심스럽고, node-postgres처럼 오래된 공개 API를 가진 라이브러리에 더 맞는 방향이다.

---

## 6. 자체 검토 — 파급 반경 확인과 자기 정정

### 6.1 호출처 안전성 확인

`normalizeQueryConfig`를 부르는 곳은 `packages/pg/lib/query.js`(순수 JS)와 `packages/pg/lib/native/query.js`(네이티브 바인딩) 두 곳뿐이고, **둘 다 반환값을 지역변수로 재할당해서 쓰고 원본 참조를 다시 안 본다**:

```js
config = utils.normalizeQueryConfig(config, values, callback)
this.text = config.text
...
```

복사본을 반환하도록 바꿔도 두 경로 모두 안전하다는 걸 확인했다.

### 6.2 진행 중 스스로 뒤집은 것

`native/client.js:172`에서 `config.callback = values`를 발견하고 처음엔 "우리가 못 고친 두 번째 버그"라고 판단했다. 조사해보니 이건 `typeof config.submit === 'function'` 분기 — 즉 **이미 만들어진 Query 인스턴스 자체**를 config 자리에 넘기는 케이스였다. 이때 `config`는 재사용 가능한 평범한 객체가 아니라 Query 객체 그 자체라 변형이 의도된 동작이고, 순수 JS `client.js`의 동일한 분기(635-643행)와 완전히 같은 패턴이다. 진짜 네이티브 쪽 plain-config 재사용 경로(`NativeQuery` 생성자)는 이미 고친 공유 `utils.normalizeQueryConfig`를 그대로 쓰고 있어서 별도 수정이 불필요했다 — **추가 수정 없이 한 줄로 JS/네이티브 양쪽 다 커버된다**는 걸 재확인.

### 6.3 기존 테스트 관용구(`assert.same`)와의 호환 확인

`test/unit/utils-tests.js`에 이미 있던 기존 검증(`assert.same(config, { text: 'TEXT' })`)이 참조 동일성을 요구하는 건 아닌지 확인했다. `assert.same`의 정의(`test/test-helper.js:83`)를 직접 열어보니 `expected`의 키들만 도는 **부분 값 비교**였다 — 매번 새 객체를 반환하도록 바꿔도 이 기존 테스트는 깨지지 않는다.

### 6.4 리뷰 반영 후 다시 확인한 호환성

리뷰 전 수정인 `{ ...config }`는 이슈 재현은 해결하지만, prototype getter를 쓰는 query config를 깨뜨릴 수 있었다. 그래서 `cloneQueryConfig()` 방식으로 바꾼 뒤 다음 조건을 다시 확인했다.

- plain object는 그대로 동작한다.
- 문자열 query는 기존처럼 `{ text: config }`로 처리한다.
- `callback`/`values`는 복사본에만 붙어서 호출자 객체를 오염시키지 않는다.
- 클래스 인스턴스처럼 query 속성이 prototype getter로 제공되는 객체도 `normalized.text`로 계속 읽힌다.
- 기존 `utils.normalizeQueryConfig()` 반환 형태는 그대로 config 객체 하나다. 호출처 시그니처를 바꾸지 않았기 때문에 `query.js`, `native/query.js`의 기존 사용 방식도 유지된다.

리뷰어가 언급한 "다른 반환 형태를 고려할 수도 있다"는 방향은 API 내부 모양을 더 크게 바꾸는 쪽이다. 이번 PR은 버그 수정 범위를 좁게 유지하는 게 낫다고 보고, 반환 형태는 유지하면서 복사 방식만 더 보수적으로 바꿨다.

---

## 7. 테스트

### 7.1 유닛 테스트 (`test/unit/utils-tests.js`)

```js
test('normalizeQueryConfig does not mutate the passed-in config object', function () {
  const original = { text: 'TEXT' }
  const callback = function () {}
  const normalized = utils.normalizeQueryConfig(original, [10], callback)
  assert.equal(original.callback, undefined)
  assert.equal(original.values, undefined)
  assert.deepEqual(normalized, { text: 'TEXT', values: [10], callback: callback })
})
```

### 7.2 리뷰 대응 테스트: prototype getter 보존

```js
test('normalizeQueryConfig preserves inherited config properties', function () {
  class QueryConfig {
    constructor() {
      this._text = 'TEXT'
    }

    get text() {
      return this._text
    }
  }

  const original = new QueryConfig()
  const callback = function () {}

  const normalized = utils.normalizeQueryConfig(original, [10], callback)

  assert.equal(original.callback, undefined)
  assert.equal(original.values, undefined)
  assert.equal(normalized.text, 'TEXT')
  assert.deepEqual(normalized.values, [10])
  assert.equal(normalized.callback, callback)
})
```

이 테스트는 리뷰어가 지적한 케이스를 직접 겨냥한다. `text`는 `original`의 own property가 아니라 `QueryConfig.prototype`의 getter다. `{ ...original }`로 복사하면 `_text`만 복사되고 `text` getter는 사라진다. 최종 수정처럼 같은 prototype을 가진 새 객체를 만들면 `normalized.text`가 계속 `'TEXT'`로 읽힌다.

동시에 `original.callback`/`original.values`가 여전히 `undefined`인지 확인한다. 즉 "prototype 접근은 유지하면서도 원본 오염은 막는다"는 두 조건을 한 테스트 안에서 같이 검증한다.

### 7.3 실제 버그 시나리오 재현 테스트 (`test/unit/client/simple-query-tests.js`)

```js
test('reusing a config object across calls', function () {
  test('does not leak callback state into a later promise-style call', function () {
    const client = helper.client()
    const config = { text: 'SELECT $1', values: [1] }
    client.query(config, function () {})
    const result = client.query(config)
    assert.ok(result instanceof Promise, 'expected client.query() to return a Promise')
    result.catch(() => {})
  })
})
```

`helper.client()`가 모의(mock) 커넥션을 쓰는 유닛 테스트라 실제 DB 없이도 이 시나리오(콜백 스타일 → 프로미스 스타일 재사용)를 검증한다.

### 7.4 전체 스위트

```
make test-unit
exit code: 0
✔ 273개, 실패 0개
```

최종 리뷰 반영 후 확인한 명령:

```bash
node packages/pg/test/unit/utils-tests.js
node packages/pg/test/unit/client/simple-query-tests.js
yarn lint
yarn build
make test-unit
git diff --check
```

전부 통과했다. `utils-tests.js`에는 원본 오염 방지 테스트와 prototype getter 보존 테스트가 들어갔고, `simple-query-tests.js`에는 실제 `Client.query()` 레벨에서 콜백 방식 호출 뒤 같은 config를 프로미스 방식으로 재사용하는 회귀 테스트가 들어갔다.

---

## 8. 결과 및 회고

### 8.1 현재 상태 (2026-08-12 기준, 🟢 Merged)

- PR #3720 **머지됨**(2026-08-12, `brianc`)
- 커밋 3개:
  - `ea6cf43` — 원본 config 객체를 직접 mutate하지 않도록 수정
  - `f0bbfcfc` — 테스트 주석 정리
  - `b4d11e47` — 리뷰 반영: object spread 대신 prototype/property descriptor 보존 복사로 변경
- 리뷰어 `charmander`가 `{ ...config }`의 호환성 문제를 지적함
- `b4d11e47` 푸시 후 리뷰 스레드에 답변 완료: prototype과 property descriptor를 유지하는 방식으로 바꿨고, inherited `text` getter 테스트를 추가했다고 설명함
- 이후 추가 리뷰 없이 `brianc`가 바로 승인·머지

### 8.2 배운 점

- **메인테이너가 이미 "동의하고 고치겠다"고 한 이슈는 좋은 타깃이다.** 방향 논쟁이 필요 없고, 그 뒤로 방치된 것뿐이라 재현만 갖춰서 제출하면 된다.
- **참조 vs 값 복사라는 자바스크립트 기초 개념이 실제 라이브러리 버그의 전체 원인이었다.** 객체를 함수에 넘기면 "주소"만 넘어가므로, 함수 안에서 그 매개변수에 값을 쓰면 호출자의 원본도 같이 바뀐다 — 이걸 피하려면 명시적으로 복사해야 한다.
- **하지만 라이브러리 코드에서는 "복사한다"도 끝이 아니다.** `{ ...obj }`는 plain object에는 충분하지만 prototype getter, non-enumerable property, descriptor를 가진 객체까지 보존하지는 않는다. 공개 라이브러리는 사용자가 어떤 형태의 객체를 넘길지 넓게 봐야 하므로, 단순한 해결책도 호환성 관점에서 다시 검토해야 한다.
- **prototype은 실전 호환성 문제로 바로 튀어나온다.** "클래스 인스턴스도 객체니까 spread하면 되겠지"가 아니라, 클래스 메서드/getter는 보통 instance 자신이 아니라 `ClassName.prototype`에 있다. object spread는 그 경로를 끊어버릴 수 있다.
- **"콜백이 있다/없다"만 보는 이 로직은, 그 값이 지금 이 호출 때문에 생긴 건지 과거 호출의 잔재인지 구분하지 못한다.** `if (!query.callback)`이라는 조건문 자체는 잘못이 없고, 오염된 입력을 정직하게 반영했을 뿐이다 — 버그는 항상 조건문보다 한 단계 앞, "그 값이 왜 거기 있는지"를 결정하는 지점에 있었다.
- **자체 검토 중 스스로를 뒤집는 것도 정상적인 과정이다.** `native/client.js`에서 두 번째 버그를 찾았다고 생각했다가, 실제로는 의도된 별개 분기(Query 인스턴스 자체를 넘기는 케이스)였음을 재확인하고 스스로 정정했다 — 성급하게 "버그 2개 발견"으로 보고하지 않고 한 번 더 추적한 게 정확한 결론으로 이어졌다.
- **기존 테스트 헬퍼(`assert.same`)의 정확한 동작을 직접 소스에서 확인하고 나서야 회귀 없음을 확신할 수 있었다** — "그럴듯해 보이니 괜찮겠지"가 아니라 실제 정의를 열어봐야 한다.
- **좋은 리뷰는 코드 한 줄만 보고도 더 깊은 언어 개념까지 짚어준다.** 이번 코멘트는 단순히 "object spread 쓰지 마라"가 아니라, 자바스크립트의 own property/prototype property 차이를 실제 API 호환성 문제로 지적한 것이다. 이걸 이해해야 다음 PR에서 같은 실수를 줄일 수 있다.

### 8.3 결과

- [x] 리뷰어 코멘트 1건 반영 완료
- [x] 추가 리뷰/승인 — 별도 재요청 없이 `brianc`가 바로 승인
- [x] 머지 — 2026-08-12

---

## 9. 개념 정리 (참고용)

- **객체는 "주소"로 전달된다**: 함수에 객체를 넘기면 내용물의 복사본이 아니라 그 객체를 가리키는 참조가 전달된다. 함수 안에서 매개변수의 속성을 바꾸면, 호출자가 들고 있는 원본도 같은 물건이므로 함께 바뀐다. 숫자/문자열 같은 원시값은 이와 달리 값 자체가 복사되어 전달된다.
- **얕은 복사(`{ ...obj }`)**: 새 객체를 만들어 최상위 속성만 복사한다. 이후 새 객체에 속성을 추가/변경해도 원본은 영향받지 않는다.
- **own property**: 객체 자신이 직접 들고 있는 속성. `obj.hasOwnProperty('text')` 또는 `Object.hasOwn(obj, 'text')`가 `true`인 속성이다.
- **prototype property**: 객체 자신에게는 없지만, `[[Prototype]]` 링크를 따라 올라가서 찾을 수 있는 속성. 클래스 메서드와 getter는 보통 여기에 있다.
- **prototype chain**: 어떤 객체에서 속성을 찾을 때 `obj` → `Object.getPrototypeOf(obj)` → 그 위 prototype → ... → `null` 순서로 올라가는 탐색 경로다. 예를 들어 배열은 보통 `array instance -> Array.prototype -> Object.prototype -> null` 순서다. 클래스 인스턴스는 `instance -> ClassName.prototype -> Object.prototype -> null` 순서다.
- **`Object.prototype`**: 대부분의 일반 객체가 마지막 근처에서 거쳐 가는 공통 prototype이다. 직접 부모인 경우도 있지만, 배열/함수/클래스 인스턴스에서는 중간에 `Array.prototype`, `Function.prototype`, `ClassName.prototype` 같은 더 구체적인 prototype이 먼저 온다.
- **클래스와 prototype**: `class`는 완전히 별개의 객체 모델이라기보다 prototype 기반 객체 모델 위에 얹힌 문법이다. `class QueryConfig { get text() {} }`라고 쓰면 `text` getter는 각 인스턴스마다 복사되는 게 아니라 `QueryConfig.prototype`에 한 번 올라간다.
- **object spread의 한계**: `{ ...obj }`는 own enumerable property만 복사한다. prototype에 있는 getter/method, non-enumerable property, property descriptor의 세부 성격은 그대로 보존하지 않는다.
- **property descriptor**: 속성의 값뿐 아니라 `writable`, `enumerable`, `configurable`, getter/setter 같은 메타데이터까지 포함한 설명서다. `Object.getOwnPropertyDescriptors(obj)`로 가져오고, `Object.defineProperties(target, descriptors)`로 다른 객체에 다시 정의할 수 있다.
- **`Object.create(proto)`**: 지정한 prototype을 가진 새 객체를 만든다. 이번 수정에서는 원본과 같은 prototype을 바라보는 새 config 객체를 만들기 위해 썼다.
- **최종 복사 방식**: `Object.defineProperties(Object.create(Object.getPrototypeOf(config)), Object.getOwnPropertyDescriptors(config))`는 "원본과 같은 prototype을 가진 새 객체를 만들고, 원본 own property의 descriptor를 그대로 복사한다"는 뜻이다. 그래서 원본 오염은 막으면서도 prototype getter 기반 query config를 깨뜨리지 않는다.
- **`!`(논리 NOT)**: 진리값을 반대로 뒤집는다. `!query.callback`은 "callback이 없으면"이라는 뜻이며, 콜백이 진짜 함수든 과거의 잔재든 "값이 있다"는 사실 자체는 구분하지 못한다.
- **`await`는 Promise가 아닌 값을 억지로 기다리게 하지 않는다**: 오른쪽 값이 Promise면 완료될 때까지 대기하지만, Promise가 아니면(예: `undefined`) 기다릴 대상이 없으므로 그 값을 즉시 그대로 통과시킨다.
- **`assert.same`의 실제 동작**: 참조 동일성이 아니라 `expected`의 키들만 순회하며 값을 비교하는 부분 검증이다. 반환 객체를 새로 만들어도 이 종류의 테스트는 깨지지 않는다.
