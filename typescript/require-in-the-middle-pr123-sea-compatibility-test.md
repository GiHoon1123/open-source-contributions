# [Node.js OSS 기여] nodejs/require-in-the-middle - Node SEA(`require.resolve` 없는 환경) 회귀 테스트 추가 (PR #123) — 🟢 Merged

> 오픈소스 기여 기록 및 학습 노트. 이 문서는 PR #123 하나만 다룬다. **2026-07-28 14:33, 메인테이너 `timfish`가 머지함.**
>
> 이번 건은 [[require-in-the-middle-pr122-require-cache-undefined]]와 같은 저장소를 대상으로 한 별개 작업이다. #122가 실제 런타임 코드를 고치는 fix였다면, 이번 #123은 **코드 수정이 전혀 없는, 순수 회귀 테스트 추가**다 — 버그 자체는 이미 예전에 고쳐졌는데, 그 사실을 지켜주는 테스트가 없어서 다시 깨질 위험이 있었다.

---

## 1. 개요

### 1.1 프로젝트 소개

- **저장소**: https://github.com/nodejs/require-in-the-middle
- **뭐 하는 라이브러리인가**: `require()`/`import`로 모듈이 로드되는 순간을 가로채서(hook), 그 모듈의 export를 감시하거나 감싸는 라이브러리. APM(Application Performance Monitoring) 도구들이 "이 라이브러리가 로드되면 자동으로 계측 코드를 심는다"는 기능을 구현할 때 핵심적으로 쓰는 저수준 유틸리티다.

### 1.2 이슈 (#109)

**제목**: "Add tests for Node Single Executable Applications"

이슈 본문:

> Node supports bundling into a Single Executable Application (SEA). We have already broken this before (#105), so we should add tests to ensure we don't do it again!

핵심은 "이미 한 번 망가뜨린 적이 있다(#105)"는 것 — 재발 방지용 테스트를 만들어달라는 요청이다.

### 1.3 배경 — 이미 있었던 버그(#105)와 그 수정

이슈 #105("Error when using with Node SEA")의 내용:

> Node has a single executable application mode where `require.resolve` does not exist as it expects everything to be bundled in. This causes a run-time exception with this module when it tries to feature detect `require.resolve.paths`. This change was introduced with #96.

즉 과거 PR #96이 `require.resolve.paths`라는 속성을 feature-detect(존재 여부로 분기)하려고 시도하는 코드를 넣었는데, Node SEA(Single Executable Application) 환경에는 `require.resolve` 자체가 없어서 이 코드가 그냥 죽었다. 이 문제(#105)는 그 이후 어느 시점에 고쳐졌고 지금은 CLOSED 상태다 — 실제로 현재 `index.js`를 보면 `require.resolve(...)` 호출이 이미 `try/catch`로 감싸여 있다:

```js
let res
try {
  res = require.resolve(moduleName, { paths: [basedir] })
} catch (e) {
  debug('could not resolve module: %s', moduleName)
  self._cache.set(filename, exports, core)
  return exports // abort if module could not be resolved
}
```

**문제는 이 수정을 지켜주는 회귀 테스트가 없었다는 것.** 코드는 고쳐졌지만, "SEA 환경(= `require.resolve`가 없는 환경)에서도 이 라이브러리가 안 죽는다"를 검증하는 테스트가 스위트에 없어서, 나중에 누군가 이 `try/catch`를 실수로 지우거나 리팩터링해도 CI가 못 잡아낼 위험이 그대로 남아있었다. #109는 정확히 이 gap을 메워달라는 요청이다.

### 1.4 관련 링크

| 구분 | 링크 |
|---|---|
| 원본 저장소 | https://github.com/nodejs/require-in-the-middle |
| 내 Fork | https://github.com/GiHoon1123/require-in-the-middle |
| 이슈 (#109) | https://github.com/nodejs/require-in-the-middle/issues/109 |
| 관련 과거 이슈 (#105, CLOSED) | https://github.com/nodejs/require-in-the-middle/issues/105 |
| PR (#123) | https://github.com/nodejs/require-in-the-middle/pull/123 |
| 작업 브랜치 | `test-109-sea-compatibility` |
| 커밋 | `47693ec8` |
| 상태 | 🟡 OPEN (메인테이너 승인 완료, 머지 대기) |

### 1.5 한 줄 요약

> Node SEA 환경엔 `require.resolve`가 없는데, 이 라이브러리가 그 환경에서도 안 죽는다는 걸 보장하는 회귀 테스트가 없었다. 실제 SEA 빌드 도구 없이, `vm` + `Module.wrap()`으로 `require.resolve`가 없는 `require` 함수를 흉내 내서 `index.js`를 직접 로드·hook·unhook하는 테스트를 추가했다.

---

## 2. 왜 실제 SEA 빌드 없이 테스트가 가능한가

### 2.1 Node SEA란

Node.js의 Single Executable Application 기능은 Node 프로젝트 전체를 하나의 실행 파일로 묶어주는 기능이다. 이 모드에서는 일반 Node 환경과 달리 `require.resolve`가 아예 없는 `require` 함수가 쓰인다 — SEA는 "필요한 건 전부 이미 번들에 들어있다"고 가정하기 때문에, 파일 경로를 찾아주는 `resolve` 기능 자체가 무의미하다.

### 2.2 진짜 SEA 빌드를 CI에 넣지 않은 이유

SEA로 실제 실행 파일을 빌드하려면 별도 빌드 도구, 빌드 단계, 그리고 그 실행 파일을 실행해서 검증하는 절차가 추가로 필요하다. PR 설명에 명시된 대로:

> The test uses `vm` to reproduce the SEA-specific loader condition without adding a SEA build tool or changing the supported Node.js test matrix.

즉 SEA를 "완전히 재현"하는 대신, **이 버그와 직접 관련된 차이점 하나(=`require.resolve`가 없다는 것)만 정확히 흉내 내는 가벼운 방법**을 선택했다. `vm.runInThisContext` + `Module.wrap()`으로 `index.js`의 소스 코드를 그대로 가져와서, `require.resolve`가 없는 커스텀 `require` 함수를 넘겨 실행시키는 방식이다.

---

## 3. 테스트 코드

```js
test('loads and hooks modules when require.resolve is unavailable', function (t) {
  const filename = path.join(__dirname, '..', 'index.js')
  const source = fs.readFileSync(filename, 'utf8')
  const indexModule = {
    exports: {},
    filename,
    paths: Module._nodeModulePaths(path.dirname(filename))
  }
  const requireFromIndex = require('module').createRequire
    ? require('module').createRequire(filename)
    : require
  const seaRequire = function (id) {
    return requireFromIndex(id)
  }

  // Node SEA does not expose require.resolve. Keep the other properties used
  // by the module loader so this only models that SEA-specific difference.
  seaRequire.cache = require.cache

  const wrapper = vm.runInThisContext(Module.wrap(source), { filename })
  t.doesNotThrow(function () {
    wrapper.call(
      indexModule.exports,
      indexModule.exports,
      seaRequire,
      indexModule,
      filename,
      path.dirname(filename)
    )
  })

  t.equal(typeof indexModule.exports.Hook, 'function')

  const hook = new indexModule.exports.Hook(['path'], function (exports) {
    return exports
  })
  t.doesNotThrow(function () {
    requireFromIndex('path')
  })
  hook.unhook()

  t.end()
})
```

### 3.1 각 부분이 하는 일

1. **`seaRequire` 함수 만들기**: 진짜 `require`처럼 모듈을 로드할 수 있는 함수(`requireFromIndex`)를 감싸되, **의도적으로 `.resolve` 속성을 안 붙인다.** `.cache`는 붙여둔다 — SEA의 차이는 정확히 "resolve가 없다"는 것뿐이지, 다른 기능까지 다 없는 건 아니기 때문에 그 차이만 재현한다.
2. **`Module.wrap(source)`**: Node가 내부적으로 모든 모듈 파일을 `function (exports, require, module, __filename, __dirname) { ... 원본 코드 ... }` 형태로 감싸는데(CommonJS 모듈의 4개 인자가 여기서 나온다), 이걸 그대로 흉내 내서 `index.js`의 진짜 소스 코드를 이 래퍼 함수 형태로 컴파일한다.
3. **`vm.runInThisContext(...)`**: 이 래퍼 함수를 실제로 실행 가능한 함수 객체로 만든다.
4. **`wrapper.call(exports, seaRequire, module, filename, dirname)`**: 이 함수를 호출하면서, 진짜 `require` 대신 우리가 만든 `seaRequire`(resolve 없는 버전)를 넘긴다. `index.js`가 내부에서 `require.resolve`를 쓰는 지점이 있다면 여기서 바로 터진다.
5. **`t.doesNotThrow(...)`**: 이 실행 자체가 예외 없이 끝나야 한다 — SEA 환경에서도 모듈 로드 자체가 죽지 않는다는 것의 직접적인 증거.
6. **`Hook` 생성 + `unhook()`**: 단순히 로드만 되는 게 아니라, 이 라이브러리의 핵심 기능(모듈을 hook해서 가로채는 것)이 실제로 동작하는지까지 확인한다 — `path` 내장 모듈을 hook했다가 다시 unhook해서, 실제 사용 시나리오까지 SEA 환경에서 멀쩡히 도는지 검증.

---

## 4. 검증

PR 본문에 명시된 테스트 결과:

```text
npm test
```

전체 스위트(lint + tape 테스트)가 통과했다. 코드 수정이 없는 순수 테스트 추가 PR이라, 회귀 확인(fix를 제거하고 실패하는지 보는 disable-and-reconfirm) 대신 **"이 테스트가 실제로 SEA의 문제 상황을 재현하는가"**가 핵심이었다. `require.resolve`가 없는 환경을 정확히 흉내 냈는지, 그리고 그 환경에서 `index.js`가 실제로 무너지지 않는지를 코드 레벨로 확인하는 것이 disable-and-reconfirm을 대신했다.

---

## 5. PR 본문

```md
Fixes #109

Node SEA does not provide `require.resolve`. This has caused `require-in-the-middle` to fail while loading before.

Add a regression test that loads `index.js` with a require function that has no `resolve` property. The test also creates a `Hook`, hooks the builtin `path` module, and unhooks it.

The test uses `vm` to reproduce the SEA-specific loader condition without adding a SEA build tool or changing the supported Node.js test matrix.

Tests:

- `npm test`
```

---

## 6. 결과 및 회고

### 6.1 현재 상태 (2026-07-28 기준)

- PR #123 **MERGED** (2026-07-28 14:33, `timfish`), 커밋 1개(`47693ec8`), 파일 1개 추가(`test/sea.test.js`, +51/-0)
- 메인테이너 `timfish`가 2026-07-21 코멘트 없이 승인(`APPROVED`)했고, 이후 별다른 추가 대응 없이 그대로 머지됨

### 6.2 배운 점

1. **"버그가 고쳐졌다"와 "그 버그가 다시 안 생긴다"는 다른 문제다.** #105는 이미 CLOSED였고 실제 코드(`try/catch`)도 고쳐져 있었지만, 그걸 지켜주는 테스트가 없어서 리팩터링 한 번이면 다시 깨질 수 있는 상태였다. 코드가 고쳐진 것만 확인하고 넘어가지 않고, "이걸 지켜주는 안전망이 있는가"까지 봐야 한다.
2. **환경을 통째로 재현하지 않고, 문제와 직접 관련된 차이 하나만 정밀하게 재현하는 게 더 실용적일 수 있다.** 실제 SEA 빌드 도구를 CI에 추가하는 대신, `require.resolve`가 없다는 그 한 가지 특성만 `vm` + 커스텀 `require` 함수로 재현해서, 빌드 파이프라인을 안 건드리고도 정확히 같은 실패 조건을 만들 수 있었다.
3. **코드 수정이 없는 PR도 여전히 검증이 필요하다.** disable-and-reconfirm(fix를 빼고 실패를 확인하는 것) 대신, "이 테스트가 실제로 문제 상황을 재현하는가"를 코드 레벨로 직접 추적하는 것이 이런 종류의(코드는 그대로, 테스트만 추가) PR의 검증 방법이었다.

### 6.3 다음에 확인할 것

- [x] 메인테이너 리뷰 — `timfish` 승인 (2026-07-21)
- [x] 머지 여부와 소요 시간 — 2026-07-28 14:33 머지됨 (승인 후 약 7일)
