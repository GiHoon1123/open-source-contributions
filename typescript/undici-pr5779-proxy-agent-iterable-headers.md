# [OSS 기여] nodejs/undici - ProxyAgent에서 반복 헤더가 사라지는 문제 (PR #5779, MERGED)

> `ProxyAgent`가 보안 검사를 위해 iterable 헤더를 객체로 바꾸는 과정에서 같은 이름의 헤더를 하나만 남기던 문제를 추적하고 수정한 기록이다. 작업 중 확인한 재현 방법, 코드 흐름, 테스트 이유를 나중에 다시 따라갈 수 있도록 정리했다.

## 1. 개요

### 1.1 프로젝트

- 저장소: https://github.com/nodejs/undici
- 작업 파일: `lib/dispatcher/proxy-agent.js`, `test/proxy-agent.js`
- 관련 보안 수정: https://github.com/nodejs/undici/commit/a94ab884
- 이번 PR: https://github.com/nodejs/undici/pull/5779 (🟢 MERGED, 2026-09-13)
- 작업 브랜치: `fix-proxy-agent-iterable-headers`
- 커밋: `b7026b55700693c796bd22e052714720bd1f3b0f`
- 머지 커밋: `f554b2d3ca660fe1543f8312a507353329391711` (mcollina)

Undici는 Node.js 생태계에서 HTTP 요청을 처리하는 라이브러리다. 전역 `fetch()`의 기반으로도 쓰이고, 낮은 수준의 `Dispatcher` API와 `ProxyAgent`도 제공한다. 이번에는 프록시를 거치는 요청에서 헤더를 검사하고 전달하는 경로를 살펴봤다.

### 1.2 한 줄 요약

> `Dispatcher.RequestOptions.headers`가 지원하는 iterable에서 같은 헤더 이름이 여러 번 나오면, `ProxyAgent`가 마지막 값만 보존하던 문제를 수정했다.

## 2. 먼저 확인한 저장소 규칙

작업을 시작하기 전에 저장소의 `CONTRIBUTING.md`, PR 템플릿, `SECURITY.md`를 확인했다.

- 버그 수정에는 재현을 고정하는 테스트가 필요하다.
- 테스트와 lint를 로컬에서 실행한다.
- 커밋에는 Developer Certificate of Origin 서명(`Signed-off-by`)을 포함한다.
- 보안 문제의 상세 내용을 공개 이슈로 다룰 때는 이미 공개된 수정 범위를 넘어서지 않도록 한다.

이번 PR은 이미 공개된 보안 패치 이후 생긴 동작 회귀를 다루므로, 보안 취약점 자체의 재현이 아니라 헤더 전달 의미가 깨진 부분만 테스트했다.

## 3. HTTP 헤더가 코드에서 가질 수 있는 모양

Undici의 저수준 요청 API에서 헤더는 한 가지 자료형으로만 들어오지 않는다. 대표적으로 다음 세 가지가 있다.

```js
// 객체
{ 'x-example': 'value' }

// 이름과 값이 번갈아 나오는 배열
['x-example', 'value']

// iterable
{
  *[Symbol.iterator] () {
    yield ['x-example', 'value']
  }
}
```

특히 타입 정의의 `Iterable<[string, string | string[] | undefined]>`는 같은 이름의 항목이 반복될 수 있는 형태다. 예를 들어 다음 iterable은 유효한 입력이다.

```js
const headers = {
  *[Symbol.iterator] () {
    yield ['x-duplicate', 'first']
    yield ['x-duplicate', 'second']
  }
}
```

HTTP에서는 같은 헤더가 여러 번 전달되는 것이 의미가 있을 수 있다. Node의 HTTP 서버에서 이 두 값은 일반적으로 `first, second`처럼 합쳐져 관찰된다. 따라서 단순히 객체의 속성 하나로 바꾸면 안 된다.

## 4. 문제를 찾은 계기

Undici의 보안 패치 `a94ab884`는 Proxy-Authorization 헤더가 실제 원 서버 요청으로 빠져나가지 않는지 확인하기 위해, iterable 형태의 헤더도 먼저 객체로 변환하도록 바꿨다.

문제는 일반적인 객체 변환의 동작이다.

```js
const headersPair = {}

for (const [key, value] of headers) {
  headersPair[key] = value
}
```

위 코드에 같은 키가 두 번 들어오면 두 번째 대입이 첫 번째 값을 덮어쓴다.

```js
headersPair['x-duplicate'] = 'first'
headersPair['x-duplicate'] = 'second'

// 결과
// { 'x-duplicate': 'second' }
```

즉, 보안 검사를 추가하는 과정에서 `Dispatcher.RequestOptions`가 원래 허용하던 반복 헤더의 의미가 사라졌다.

## 5. 재현 과정

### 5.1 서버와 ProxyAgent 구성

먼저 원 서버와 프록시를 띄운 뒤, 두 서버를 거쳐 요청을 보냈다. 핵심은 객체나 `Headers`가 아니라 직접 만든 iterable을 전달하는 것이다.

```js
const headers = {
  *[Symbol.iterator] () {
    yield ['x-duplicate', 'first']
    yield ['x-duplicate', 'second']
  }
}

await request(serverUrl, {
  dispatcher: new ProxyAgent(proxyUrl),
  headers
})
```

### 5.2 패치 전 결과

패치 전 코드는 iterable을 객체로 만들면서 마지막 값만 남겼다. 원 서버에서 확인한 값은 다음과 같았다.

```js
req.headers['x-duplicate']
// 'second'
```

동일한 요청을 `a94ab884` 바로 이전 커밋에서 실행하면 원 서버에는 두 값이 모두 도착했다.

```js
req.headers['x-duplicate']
// 'first, second'
```

이 비교로 문제가 ProxyAgent의 프록시 연결 자체가 아니라, 보안 검사 직전의 헤더 변환에서 발생한다는 것을 확인했다.

## 6. 코드 흐름

ProxyAgent의 요청 흐름은 대략 다음과 같다.

```text
request()
  -> ProxyAgent.dispatch()
    -> buildHeaders(opts.headers)
      -> Proxy-Authorization 검사
    -> proxy CONNECT
    -> 원 서버로 요청 전달
```

`buildHeaders()`는 다음 두 목적을 동시에 수행한다.

1. 사용자가 넘긴 여러 헤더 표현을 검사 가능한 객체로 만든다.
2. 프록시 인증 헤더가 일반 요청 헤더에 섞여 있는지 확인한다.

두 번째 목적은 보안 패치로 추가된 것이므로 제거할 수 없다. 해결해야 할 부분은 첫 번째 단계에서 반복 값을 잃지 않는 것이었다.

## 7. 수정 방법

반복되는 키를 만났을 때 기존 값과 새 값을 배열로 모았다. 이미 배열인 값도 있으므로 한 번 평탄화한 다음 저장한다.

```js
if (!Object.hasOwn(headersPair, key)) {
  headersPair[key] = value
  continue
}

const previous = headersPair[key]
const values = []

if (previous !== undefined) {
  values.push(...(Array.isArray(previous) ? previous : [previous]))
}

if (value !== undefined) {
  values.push(...(Array.isArray(value) ? value : [value]))
}

headersPair[key] = values.length > 1 ? values : values[0]
```

이렇게 하면 입력에 따라 다음처럼 보존된다.

```js
[
  ['x-duplicate', 'first'],
  ['x-duplicate', 'second']
]

// {
//   'x-duplicate': ['first', 'second']
// }
```

배열 값과 `undefined`도 기존 `UndiciHeaders` 타입과 맞춰 처리했다. 첫 번째 값이 `undefined`이고 두 번째 값만 실제 값인 경우에는 불필요한 `undefined`를 배열에 넣지 않는다.

## 8. 회귀 테스트

`test/proxy-agent.js`에 반복 iterable 헤더를 ProxyAgent로 전달하는 테스트를 추가했다.

```js
test('use proxy-agent with repeated iterable headers', async (t) => {
  t = tspl(t, { plan: 1 })

  // 서버와 프록시 구성 생략

  server.on('request', (req, res) => {
    t.strictEqual(req.headers['x-duplicate'], 'first, second')
    res.end()
  })

  const headers = {
    *[Symbol.iterator] () {
      yield ['x-duplicate', 'first']
      yield ['x-duplicate', 'second']
    }
  }

  await request(serverUrl, { dispatcher: proxyAgent, headers })
})
```

테스트가 단순히 `buildHeaders()`의 반환값만 검사하지 않고 실제 프록시와 원 서버를 통과하게 한 이유는, 최종적으로 Node HTTP 서버가 두 값을 어떻게 받는지가 사용자에게 보이는 동작이기 때문이다.

## 9. 확인한 경계 사례

### 9.1 Proxy-Authorization 검사는 유지

이번 수정은 헤더 보존 로직만 바꿨다. `Proxy-Authorization`을 일반 요청에 넣었을 때 차단하는 기존 검사는 그대로 남아 있다. 따라서 보안 패치의 목적을 약화시키지 않는다.

### 9.2 Host와 Content-Length 중복 입력

iterable을 객체 형태로 모으는 시점이 바뀌면서 중복된 `host`나 `content-length`가 있을 때 최종 오류가 `invalid host header`, `invalid content-length header`로 나타나는 경로를 확인했다. 요청이 허용되는 회귀는 아니며, 기존 검증이 중복 입력을 차단한다는 사실도 함께 확인했다. 이번 PR에서는 오류 문구를 별도 변경하지 않았다.

### 9.3 일반 객체와 배열 입력

이번 변경은 iterable 분기에만 적용된다. 기존의 일반 객체 및 flat array 처리 경로는 건드리지 않았다. 변경 범위를 좁혀 ProxyAgent의 기존 입력 동작을 불필요하게 바꾸지 않도록 했다.

## 10. 테스트 결과

작업 브랜치에서 다음을 실행했다.

```text
npx borp --timeout 180000 test/proxy-agent.js
39 passed

npx eslint lib/dispatcher/proxy-agent.js test/proxy-agent.js
passed

git diff --check
passed
```

패치 전 커밋과 현재 수정 브랜치를 각각 실행해 원 서버에 도착한 헤더 값을 비교했고, 수정 후에는 `first, second`가 유지되는 것을 확인했다.

## 11. 리뷰와 현재 상태

PR 본문에는 보안 패치 `#5758`의 후속 수정이라는 점, 반복 헤더가 마지막 값으로 덮이는 원인, 변경 파일과 테스트 명령을 짧게 적었다. 커밋과 PR에는 AI 공동 저자나 자동 생성 표기를 넣지 않았다.

로컬 커밋은 하나였다.

```text
b7026b55700693c796bd22e052714720bd1f3b0f
fix(proxy-agent): preserve repeated iterable headers
```

리뷰는 추가 커밋 없이 그대로 통과했다.

- 2026-09-10: `metcoder95` 승인
- 2026-09-13: `mcollina`가 "lgtm"과 함께 승인 및 머지 (`f554b2d3ca`), 커밋 제목·본문 변경 없이 그대로 병합됨
- 전체 CI(여러 Node.js 버전·운영체제 조합) 전부 통과, 실패한 job 없음

## 12. 이번 작업에서 배운 점

1. 보안 수정은 원래 기능을 지키는 것만큼 기존 입력 형식을 보존하는지도 함께 확인해야 한다.
2. `Iterable`은 단순히 `Object.fromEntries()`로 바꿔도 되는 자료형이 아니다. 같은 키가 반복될 수 있기 때문이다.
3. 변환 함수의 단위 테스트만으로는 프록시·원 서버 사이에서 최종 헤더가 어떻게 보이는지 알기 어렵다. 실제 요청 경로를 통과하는 회귀 테스트가 필요하다.
4. 공개 타입 정의에 적힌 입력 범위는 구현을 읽을 때 중요한 단서다. 이번 문제도 `Iterable<[string, string | string[] | undefined]>` 지원과 실제 구현의 차이를 비교하면서 발견했다.
