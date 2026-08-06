# [OSS 기여] nodejs/undici - 인터셉터 조합 순서에 따라 헤더 정규화가 깨지는 문제 수정 (PR #5535) — 🟢 Merged (테스트 전용 PR로 재조준)

> 오픈소스 기여 기록 및 학습 노트. 이 문서는 PR #5535 하나만 다룬다. **최신 상태: 메인테이너 `mcollina`가 같은 버그를 자기 PR #5536으로 먼저 머지하면서 이 PR은 CONFLICTING 상태로 전락했고, 겹치지 않는 부분(통합 테스트)만 추려 "테스트 전용 PR"로 재조준해 MERGEABLE로 복구했다. 8번 섹션에 이 경위 전체를 추가했다. 2026-07-28 12:54, 겹치는 구현 변경분을 최종적으로 걷어냈다는 코멘트를 남겼고, 같은 날 21:28 `mcollina`가 머지함.**
>
> **이번 건은 사용자가 이슈를 직접 찾아서 골랐고, 재현/원인분석/수정/테스트 작성을 전부 별도로 진행**했다. 이후 두 차례의 자체/외부 검토와 메인테이너의 실제 리뷰, 그리고 8번 섹션의 충돌 해결/재조준 과정을 거쳤다.

---

## 1. 개요

### 1.1 프로젝트 소개

- **저장소**: https://github.com/nodejs/undici — **Node.js 코어 팀이 직접 관리하는 공식 HTTP 클라이언트 라이브러리**. Node.js의 전역 `fetch()`가 내부적으로 이 라이브러리를 씀.
- **건드린 부분**: `lib/util/cache.js`의 `normalizeHeaders()` — `cache`/`deduplicate` 인터셉터가 공유해서 쓰는 "헤더를 표준 모양으로 통일하는" 유틸 함수.

### 1.2 왜 이 저장소/이슈를 골랐나

사용자가 직접 이슈(#5522)를 찾아서 제시함. 진행 전에 이 저장소의 리뷰 속도를 확인함 — 최근 머지된 PR들을 조사한 결과, **진짜 버그 수정 PR이 몇 시간 안에 머지**되는 걸 확인함 (예: 외부 기여자로 보이는 `spokodev`가 3.4시간, `Ram-blip`이 9.2시간). 이건 이전에 조사했던 `ethereumjs-monorepo`나 `argotorg/solc-js`(리뷰가 며칠~수년씩 걸리거나 무응답)와는 완전히 다른, 훨씬 건강한 리뷰 사이클을 가진 저장소라는 뜻.

### 1.3 관련 링크

| 구분 | 링크 |
|---|---|
| 원본 저장소 | https://github.com/nodejs/undici |
| 내 Fork | https://github.com/GiHoon1123/undici |
| 이슈 (#5522) | https://github.com/nodejs/undici/issues/5522 |
| PR (#5535) | https://github.com/nodejs/undici/pull/5535 |
| 작업 브랜치 | `fix-issue-5522-normalize-headers-interceptor-order` |
| 커밋 | `9be4822a2c4a09826395e7872ebeca019315c2c8` (2026-07-09 17:32 KST) |
| PR 생성 시각 | 2026-07-09 08:33:44 UTC (17:33 KST) |
| 상태 | 🟡 OPEN |

### 1.4 한 줄 요약

> undici의 인터셉터(요청을 가로채서 처리하는 중간 계층)를 조합하는 순서에 따라, `dns` 인터셉터가 헤더를 자기만의 저수준 배열 포맷으로 바꿔놓고 `cache`/`deduplicate` 인터셉터에게 넘기는 경우가 생기는데, 그 인터셉터들의 `normalizeHeaders()`가 이 포맷을 몰라서 `Error: opts.headers is not a valid header map`로 크래시함. `normalizeHeaders()`가 이 포맷도 알아보게 고침.

---

## 2. 배경 지식 — 하나도 빼놓지 않고 처음부터

이 섹션은 실제로 사용자가 "용어도 하나도 이해가 안 간다"고 해서, 대화 중에 완전히 처음부터 다시 설명했던 내용을 정리한 것이다.

### 2.1 인터셉터(Interceptor)란

요청을 실제로 보내기 전에 **중간에서 가로채서 뭔가를 처리하는 계층**. 택배 비유:
- `dns` 인터셉터: 도메인 이름(`www.google.com`)을 실제 IP 주소로 바꿔주는 역할 (배송 주소를 실제 GPS 좌표로 바꿔주는 사람)
- `cache` 인터셉터: 예전에 보낸 것과 같은 요청이면 저장해둔 응답을 재활용하는 역할 (창고에서 이미 배송한 물건을 재활용하는 사람)
- `deduplicate` 인터셉터: 지금 동시에 똑같은 요청이 여러 개 나가고 있으면, 하나만 실제로 보내고 나머지는 그 결과를 같이 나눠 쓰는 역할

### 2.2 헤더(Header)의 여러 표현 방식

헤더는 요청에 붙는 "메모 쪽지"(`Content-Type: text/plain` 같은 키:값). 코드에서 이걸 표현하는 방식이 최소 3가지 있다:

```js
{ 'host': 'example.com', 'foo': 'bar' }              // ① 그냥 객체
[['host', 'example.com'], ['foo', 'bar']]            // ② [키,값] 쌍들의 배열
['host', 'example.com', 'foo', 'bar']                // ③ 키,값이 번갈아 나오는 "평평한(flat)" 배열
```

이 3가지가 다 "같은 헤더"를 뜻하지만 **모양이 다르다**는 게 이번 버그의 핵심이다.

### 2.3 DNS 조회는 왜 비동기(async)인가

도메인 이름은 사람이 읽기 편한 별명이고, 실제 통신에는 IP 주소(숫자)가 필요하다. 이 별명→숫자 변환을 **DNS 조회**라 하는데:
- 이미 최근에 물어본 도메인이면 로컬 캐시에서 바로 답 (빠름)
- 처음 물어보는 도메인이면 **진짜 네트워크로 DNS 서버에 물어보고 답을 받아와야 함** (느림, 비동기)

비동기라는 건 "물어보고, 답이 오면 그때 실행할 함수(콜백)를 등록해두고, 그 사이엔 다른 일을 한다"는 뜻. 이게 이번 버그가 "왜 하필 이 타이밍에" 터지는지의 핵심 배경이다.

### 2.4 `compose()` — 인터셉터를 감싸는 순서가 직관과 반대인 이유

**왜 "그냥 순서대로 하나씩 실행" 안 하고 감싸는가**: `cache` 인터셉터는 요청을 보내기 "전"(캐시 확인)과 응답이 온 "후"(캐시 저장) **둘 다** 처리해야 한다. 이건 그냥 배열을 순서대로 실행해서는 표현이 안 되고, "① 앞처리 → ② 다음 애 호출 → ③ 뒤처리"를 하려면 다음 애를 자기 안에 갖고 있어야(=감싸야) 한다. 이건 Express.js 미들웨어, Redux 미들웨어 같은 데서도 쓰는 흔한 패턴(미들웨어/데코레이터 패턴)이다.

**실제 코드**:
```js
compose (...interceptors) {
  let dispatch = 진짜_요청_보내는_함수   // 제일 안쪽 ("코어")
  for (const interceptor of interceptors) {
    dispatch = interceptor(dispatch)   // 지금까지 만든 걸 한 겹 더 감싼다
  }
  return dispatch
}
```

`compose(cache(), dns())`를 호출하면:
```
1단계: [ 진짜 요청 함수 ]
2단계: cache()로 감싸기 → [ cache [ 진짜 요청 함수 ] ]
3단계: dns()로 감싸기   → [ dns [ cache [ 진짜 요청 함수 ] ] ]
```

**택배 포장을 생각하면**: 먼저 물건 넣고 첫 포장지(cache)를 두르고, 그 위에 포장지(dns)를 한 번 더 둘렀다. 받는 사람이 제일 먼저 벗기는 포장지는 **맨 마지막에 두른 것(dns)**. 즉 요청이 오면 **바깥쪽(dns)이 먼저 실행되고, dns가 자기 "next"인 cache를 부른다.**

**여기서 함정**: `compose(cache(), dns())`라는 코드를 보고 "cache 먼저"라고 생각하기 쉬운데, **실제 실행 순서는 반대(dns가 먼저, cache가 안쪽/나중)**다. 이번 이슈 원문의 변수 이름(`cacheThenDns`)도 이 함정에 걸려서 오해를 부르는 이름이었다.

### 2.5 Host 헤더와 `withHostHeader()`

HTTP 요청에는 "나 지금 어떤 도메인으로 접속하려는 거야"를 알려주는 `Host` 헤더가 필요하다 (하나의 IP에 여러 도메인이 얹혀있을 수 있어서). `dns` 인터셉터가 도메인을 IP로 바꿔버리기 때문에, **원래 도메인이 뭐였는지를 헤더에 남겨줘야** 한다. 이걸 하는 함수가 `dns.js`의 `withHostHeader(host, headers)`다.

### 2.6 실제 크래시가 나는 시간순 흐름

`compose(cache(), dns())` = dns가 바깥(먼저), cache가 안쪽(다음).

```
[1차 실행]
1. 요청 시작
2. dns 인터셉터 실행 시작
3. dns: "도메인 IP로 바꿔야 하네" → 네트워크로 DNS 조회 시작
4. dns: "답 올 때까지 기다려야 하니, 여기서 멈춤(비동기)"
   ※ 이 시점까지 cache는 아직 한 번도 실행 안 됨

... (DNS 서버 응답 대기) ...

[2차 실행 — DNS 응답 도착, 콜백 실행]
5. dns: "IP 알아냈다"
6. dns: "Host 헤더 추가해야지" → withHostHeader() 실행 → 헤더가 "평평한 배열"로 바뀜
7. dns: "다음 애(cache)한테 넘기자" → dispatch() 호출
8. cache 인터셉터가 이제서야 처음 실행됨
9. cache: normalizeHeaders() 호출해서 헤더를 살펴봄
10. 💥 평평한 배열 모양을 몰라서 에러
```

**핵심**: cache가 실행되는 시점에는, 그보다 먼저 실행된 dns가 이미 헤더 모양을 바꿔놓은 상태다. cache는 이걸 전혀 모른 채로 "지금 이 모양 이상한데?" 하고 에러를 던진다.

### 2.7 "캐시가 IP를 캐싱할지 판단한다"는 오해 정정

`cache` 인터셉터가 헤더를 살펴보는 이유는 **IP 캐싱과 무관**하다. `cache`가 확인하려는 건 **요청을 만든 애플리케이션 코드가 직접 넣은 `cache-control` 헤더**(예: `no-store`)나, 서버 응답의 `Cache-Control` 헤더 — "이 요청/응답을 캐싱해도 되는지"를 판단하려는 것. `host` 헤더는 cache가 전혀 관심 없는 헤더다. dns가 host 헤더를 넣어주든 안 넣어주든 cache 입장에선 상관없다 — 문제는 dns가 헤더 전체의 **모양(shape)**을 바꿔놓았다는 것.

(참고로 `dns` 인터셉터 자체도 "도메인→IP" 매핑을 기억해두는 내부 캐시를 갖고 있는데, 이건 `cache` 인터셉터의 HTTP 응답 캐시와는 완전히 다른, 이름만 같은 별개의 개념이다.)

---

## 3. 이슈 분석 (#5522)

### 3.1 이슈 원문 요약 (2026-07-07 등록)

**제목**: "`normalizeHeaders` fails with `opts.headers is not a valid header map` depending on interceptor ordering"

**재현 예시 1**: `headers = []`(빈 배열)로 요청 시, `compose(dns(), cache())`는 정상, `compose(cache(), dns())`는 에러.
**재현 예시 2**: 헤더 없이 리다이렉트가 걸리는 요청에서도 같은 방식으로 순서에 따라 에러.

라벨: `bug`. 등록 2일 후 확인 시점 기준 댓글 0개, unassigned.

### 3.2 실제 재현 (google.com 대상, 직접 실행)

```js
import { Agent, interceptors, request } from "undici";
const { cache, dns } = interceptors;
const dnsThenCache = new Agent().compose(dns(), cache());
const cacheThenDns = new Agent().compose(cache(), dns());
const headers = [];
await request("https://www.google.com", { dispatcher: dnsThenCache, headers });  // 200, 정상
await request("https://www.google.com", { dispatcher: cacheThenDns, headers });  // 💥 크래시
```

결과: `dnsThenCache`는 200, `cacheThenDns`는 정확히 이슈에 적힌 그 에러와 스택트레이스로 재현됨.

---

## 4. 근본 원인 — 코드로 끝까지 추적

### 4.1 `dns.js`가 헤더를 "평평한 배열"로 바꾸는 지점

```js
// lib/interceptor/dns.js
function withHostHeader (host, headers) {
  const normalizedHeaders = normalizeHeaders(headers)   // dns.js 자체의 로컬 normalizeHeaders (cache.js의 것과 다른 별개 함수!)

  if (hasHostHeader(normalizedHeaders)) {
    return normalizedHeaders
  }
  if (Array.isArray(normalizedHeaders)) {
    return ['host', host, ...normalizedHeaders]   // ← 평평한 배열로 리턴
  }
  ...
}
```

`headers = []`이 들어오면: dns.js의 로컬 `normalizeHeaders([])`가 `[]`을 그대로 리턴(빈 배열이라 별도 처리 없음) → `hasHostHeader([])`는 `false` → `Array.isArray([])`는 `true` → **`['host', 'www.google.com', ...[]] = ['host', 'www.google.com']`**을 리턴.

### 4.2 이 값이 재-dispatch되어 어디로 가는지

```js
// lib/interceptor/dns.js, runLookup()의 콜백 안
instance.runLookup(origin, origDispatchOpts, (err, newOrigin) => {
  const dispatchOpts = {
    ...origDispatchOpts,
    origin: newOrigin.origin,
    headers: withHostHeader(origin.host, origDispatchOpts.headers)   // 평평한 배열로 교체됨
  }
  dispatch(dispatchOpts, ...)   // ← 이 dispatch가 "다음 애"
})
```

`compose()`의 동작 방식(2.4절)에 따라, `compose(cache(), dns())`에서 dns의 "다음"은 **cache**다. 그래서 이 `dispatch()` 호출이 cache의 인터셉터 함수로 들어간다.

### 4.3 `cache.js`가 이걸 받아서 죽는 지점

```js
// lib/interceptor/cache.js
opts = {
  ...opts,
  headers: normalizeHeaders(opts)   // cache.js의 normalizeHeaders (util/cache.js에서 import)
}
```

```js
// lib/util/cache.js (수정 전)
function normalizeHeaders (opts) {
  ...
  if (hasSafeIterator(opts.headers)) {
    for (const x of opts.headers) {          // ['host', 'www.google.com']을 순회
      if (!Array.isArray(x)) {               // x = 'host' (문자열) → 배열 아님!
        throw new Error('opts.headers is not a valid header map')   // 💥
      }
      ...
    }
  }
}
```

`['host', 'www.google.com']`을 순회하면 첫 원소가 `'host'`(그냥 문자열)인데, 이 코드는 "각 원소는 `['키','값']` 형태의 배열이어야 해"라고 기대하고 있어서 바로 에러.

### 4.4 왜 반대 순서(`dns()`가 안쪽)에서는 안 터지는가

`compose(dns(), cache())`면 dns의 "다음"이 cache가 아니라 **base 함수(진짜 코어 함수)**다. 코어 함수(`core/util.js`, `dispatcher/client-h1.js`)는 원래부터 이 "평평한 배열" 포맷을 정확히 이해하도록 만들어져 있어서(`for (let i = 0; i < headers.length; i += 2)`로 순회) 문제없이 처리된다. cache를 다시 거치지 않으니 이 충돌 자체가 안 생긴다.

### 4.5 `deduplicate` 인터셉터도 똑같이 취약함을 추가로 발견

`lib/interceptor/deduplicate.js`도 **동일한 `util/cache.js`의 `normalizeHeaders`/`makeCacheKey`를 그대로 import해서 씀**:
```js
const { normalizeHeaders, makeCacheKey, makeDeduplicationKey } = require('../util/cache.js')
...
opts = { ...opts, headers: normalizeHeaders(opts) }   // cache.js와 완전히 동일한 패턴
```
`compose(deduplicate(), dns())`로 직접 재현 → **똑같은 에러, 똑같은 스택트레이스 패턴**으로 확인함. `util/cache.js`의 공유 함수 하나를 고치는 것이므로, 이 fix는 `cache`뿐 아니라 `deduplicate`도 자동으로 같이 고쳐줌.

---

## 5. 수정 내용

### 5.1 핵심 diff

```diff
+function isValidHeaderValue (val) {
+  if (typeof val === 'string') {
+    return true
+  }
+  return Array.isArray(val) && val.every(v => typeof v === 'string')
+}

 function normalizeHeaders (opts) {
   let headers
   if (opts.headers == null) {
     headers = {}
+  } else if (Array.isArray(opts.headers) && (opts.headers.length === 0 || !Array.isArray(opts.headers[0]))) {
+    // Flat alternating header array, e.g. ['key', 'value', ...].
+    headers = {}
+    for (let i = 0; i < opts.headers.length; i += 2) {
+      const key = opts.headers[i]
+      const val = opts.headers[i + 1]
+      if (typeof key !== 'string' || !isValidHeaderValue(val)) {
+        throw new Error('opts.headers is not a valid header map')
+      }
+      headers[key.toLowerCase()] = val
+    }
   } else if (typeof opts.headers === 'object') {
     headers = {}
     if (hasSafeIterator(opts.headers)) {
       for (const x of opts.headers) {
         if (!Array.isArray(x)) {
           throw new Error('opts.headers is not a valid header map')
         }
         const [key, val] = x
-        if (typeof key !== 'string' || typeof val !== 'string') {
+        if (typeof key !== 'string' || !isValidHeaderValue(val)) {
           throw new Error('opts.headers is not a valid header map')
         }
         headers[key.toLowerCase()] = val
       }
     }
     ...
```

### 5.2 "평평한 배열인지" 판단하는 방법

```js
Array.isArray(opts.headers) && (opts.headers.length === 0 || !Array.isArray(opts.headers[0]))
```
배열의 **첫 번째 원소가 또 배열인지**로 구분한다:
- `[['host','v'], ['foo','bar']]` → 첫 원소가 배열 → "쌍 배열" → 기존 로직
- `['host', 'v']` → 첫 원소가 문자열 → "평평한 배열" → 새 분기

이 판단 기준은 `dns.js` 자신이 내부적으로 이미 쓰고 있던 것과 동일한 방식을 그대로 가져온 것 — 같은 문제를 겪는 코드끼리 같은 기준을 쓰는 게 일관적이라고 판단.

### 5.3 값 추출 — `i += 2`로 두 칸씩 건너뛰기

```js
for (let i = 0; i < opts.headers.length; i += 2) {
  const key = opts.headers[i]
  const val = opts.headers[i + 1]
```
`['host', 'www.google.com']`을 대입해서 따라가면: `i=0`일 때 `key='host'`, `val='www.google.com'` → `headers.host = 'www.google.com'`. `i=2`는 배열 길이(2)와 같아서 반복 종료. 결과: `{ host: 'www.google.com' }` — cache가 원래 알던 "보통 객체" 모양이 정확히 나옴.

### 5.4 JSDoc 정정

```diff
-/**
- * @param {Record<string, string[] | string>}
- * @returns {Record<string, string[] | string>}
- */
+/**
+ * @param {import('../../types/dispatcher.d.ts').default.DispatchOptions} opts
+ * @returns {Record<string, string | string[]>}
+ */
 function normalizeHeaders (opts) {
```
바로 위 형제 함수 `makeCacheKey`가 이미 쓰던 것과 같은 정확한 타입 참조로 통일. `DispatchOptions.headers`의 실제 타입(`UndiciHeaders = OutgoingHttpHeaders | string[] | Iterable<[...]> | null`)이 이미 지금 처리하는 모든 형태(객체/flat배열/쌍배열/Map)를 정확히 커버하고 있어서, 새 타입을 만들 필요 없이 정확한 기존 타입을 가리키게만 고침.

---

## 6. 검토 — 세 번, 시간순 그대로

### 6.1 1차 검토 (자체 검토, 첫 커밋 전)

- **stash A/B로 실제 크래시 재현**: 이슈의 두 예시 다 fix 전에는 크래시, fix 후에는 정상 200을 확인.
- **`deduplicate` 인터셉터도 취약함을 스스로 찾아냄** (4.5절) — 같은 fix로 같이 해결됨을 재현/확인.
- **`makeCacheKey` 실행 순서 확인**: `normalizeHeaders(opts)`의 결과가 `opts`에 재대입된 *이후에* `makeCacheKey(opts)`가 호출되는 걸 코드로 직접 확인 — 캐시 키에 항상 정규화된(일관된) 헤더 모양이 저장되므로, 헤더 표현 방식이 달라져서 캐시 키가 어긋나는 2차 문제는 없음을 확인.
- 회귀 위험 분석: 원래 코드는 "첫 원소가 배열 아닌 배열"을 만나면 무조건 에러였음 → 새 분기는 그중 "진짜 유효한 flat 헤더"만 골라 통과시키는 것 → 예전에 통과하던 입력이 이제 막히는 경우는 없음(안전).
- 회귀 테스트 추가: `test/interceptors/dns.js`에 로컬 서버 + mock lookup 기반 hermetic 테스트 2개 (`cache`용, `deduplicate`용).
- 검증: `dns.js` 29/32(3 skip) 통과, `cache*.js` 63/63, `deduplicate.js` 33/33, eslint 클린.

### 6.2 2차 검토 (외부 검토, 세 가지 지적)

**지적 1 — value가 항상 string이 아닐 수 있다**: `client-h1.js:1293`을 직접 다시 확인한 결과, 실제 저수준 소비 코드가 `Array.isArray(val)`인 경우(예: `Set-Cookie`처럼 같은 헤더 이름이 여러 값을 가질 때)를 정식으로 처리하고 있었음. `dns.js`의 로컬 `normalizeHeaders`도 `[key, [v1,v2]]` 같은 배열 값을 그대로 flat 배열에 실어 나를 수 있는 걸 코드로 재확인. **1차 fix는 이 케이스를 놓치고 있었고, 실제로 다시 던질 수 있었음.**
→ `isValidHeaderValue()` 헬퍼 추가(문자열 또는 문자열 배열 허용), **새 flat-배열 분기와 기존 쌍-배열 분기 둘 다에** 적용.

**지적 2 — util 단위 테스트가 없다**: `test/cache-interceptor/cache-utils.js`에 `normalizeHeaders`를 직접 부르는 단위 테스트 6개 추가 (flat 배열, 빈 flat 배열, 배열 값 있는 flat/tuple 형태, 잘못된 값 두 케이스).

**지적 3 — 프로덕션 코드 주석이 너무 길다**: 코드베이스의 실제 주석 밀도(대부분 1~2줄)를 직접 확인한 뒤, `.compose(cache(), dns())` 시나리오를 통째로 설명하던 7줄 주석을 `// Flat alternating header array, e.g. ['key', 'value', ...].` 한 줄로 줄임.

### 6.3 3차 검토 (같은 검토자, 재검증)

**Finding 1**: 큰 버그 없음, `isValidHeaderValue()` 방향 확인.
**Finding 2**: JSDoc이 여전히 낡음(`@param {Record<string, string[] | string>}`가 실제 입력 형태와 안 맞음) — 실제로 `normalizeHeaders`는 object뿐 아니라 flat 배열/tuple 배열/Map까지 받는데 주석이 그걸 반영 못 함.
→ `makeCacheKey`와 동일한 정확한 타입 참조로 교체 (5.4절).
**Finding 3**: 유닛 테스트 추가는 좋음, 이전보다 훨씬 설득력 있어짐.
**Finding 4**: 테스트 파일의 남은 주석도 살짝 길다(블로커 아님) → 4줄→2줄로 추가 트림.
**환경 이슈 제보**: `node --test test/interceptors/dns.js` 전체 실행 시 `ERR_SSL_EE_KEY_TOO_SMALL`로 1개 실패 관찰됨.
→ **stash로 우리 변경 없이 다시 실행해서 확인**: 변경 있든 없든 `node --test`로는 이 에러 자체가 재현이 안 됨(둘 다 29 pass/3 skip/0 fail) — 우리 변경과 무관한, 실행 환경/러너(`borp` vs `node --test`) 차이에서 오는 플레이키로 결론.

**최종 검증**: `cache-utils.js` 8/8, `dns.js` 29/32(3 skip, 0 fail), eslint 클린, 이슈 원문 재현 스크립트(google.com 실제 요청) 재통과.

### 6.4 실제 메인테이너 리뷰 — `mcollina`의 인라인 코멘트 (2026-07-10)

PR을 올리고 하루 뒤, undici 코어 메인테이너 `mcollina`가 실제로 코드에 인라인 리뷰를 남김:

> `lib/util/cache.js:42` — "use a proper for loop, `.every()` is slow."

우리가 5번 섹션에서 넣은 `isValidHeaderValue()`의 이 부분을 지적한 것:
```js
return Array.isArray(val) && val.every(v => typeof v === 'string')
```

**왜 `.every()`가 느리다고 지적했는지**: `normalizeHeaders`는 undici에서 **HTTP 요청마다 매번 호출되는 hot path 함수**임. `.every()`는 배열을 순회할 때마다 콜백 함수를 호출하는 방식이라, 단순 `for` 루프보다 함수 호출 오버헤드가 더 들어감. undici처럼 성능에 민감한 라이브러리는 이런 사소해 보이는 차이도 리뷰에서 짚는다는 걸 실전으로 확인함.

**수정**:
```js
function isValidHeaderValue (val) {
  if (typeof val === 'string') {
    return true
  }

  if (!Array.isArray(val)) {
    return false
  }

  for (let i = 0; i < val.length; i++) {
    if (typeof val[i] !== 'string') {
      return false
    }
  }

  return true
}
```
콜백 함수 호출 없이 인덱스로 직접 순회, 문자열 아닌 값을 만나면 즉시 `return false`. **동작(어떤 입력에 어떤 결과를 내는지)은 완전히 동일** — 빈 배열이 `true`를 리턴하는 것까지 `.every()`의 기존 동작과 일치함을 직접 확인. `cache-utils.js` 8/8 재통과, eslint 클린.

이 지적에 대해서는 별도 코멘트 남기지 않음 — 커밋 자체가 정확히 요청받은 대로 고친 것이라 코드로 답하는 걸로 충분하다고 판단 (PR #5538 쪽의 더 복잡한 CI 실패 건과는 다르게, 이건 코멘트로 부연 설명할 게 딱히 없는 단순 수정).

---

## 7. PR 본문 & 기여 프로세스

### 7.1 undici 자체 PR 템플릿 사용

이 저장소는 `.github/PULL_REQUEST_TEMPLATE.md`라는 자체 템플릿이 있어서(This relates to / Rationale / Changes / Features / Bug Fixes / Breaking Changes / Status 체크리스트), 처음부터 그 구조를 그대로 따라 작성함.

### 7.2 "Documented" 체크박스를 정직하게 비워둠

템플릿에 `- [ ] Documented` 항목이 있는데, 이건 내부 버그 수정이라 공개 API/문서 변경이 없어서 **체크하지 않고 비워둠** — [[feedback_honest_pr_checklists]]와 같은 원칙.

### 7.3 DCO(Developer's Certificate of Origin)

`CONTRIBUTING.md`에 Node.js 코어와 동일한 표준 DCO 문구가 있음. `git commit -s`를 강제하는 봇/CI 체크는 안 보였고, PR 템플릿의 체크박스로 "동의했다"고 자기 확인하는 방식 — 체크박스 동의로 처리.

### 7.4 AI 언급 여부

사용자에게 확인 → **"언급 안 함"** 선택 (confluentinc/node-redis/ethereumjs와 동일). 커밋 메시지 + PR 본문을 AI 도구 관련 키워드로 grep해서 확인, 아무것도 안 걸림.

### 7.5 Fork & 브랜치

```bash
gh repo fork nodejs/undici --clone=true
npm install
git checkout -b fix-issue-5522-normalize-headers-interceptor-order
```

### 7.6 최종 커밋 메시지

```
fix: normalizeHeaders throws on flat header arrays from dns re-dispatch

normalizeHeaders() in lib/util/cache.js (shared by the cache and
deduplicate interceptors) only recognized headers as a plain object or
an iterable of [key, value] pairs. It didn't recognize the flat,
alternating [key, value, key, value, ...] array that the dns
interceptor's withHostHeader() produces when it re-dispatches after
resolving an address -- the same low-level shape core/util.js and
dispatcher/client-h1.js already consume.

Because compose() wraps arguments innermost-first, .compose(cache(),
dns()) makes dns the outer interceptor and cache its "next", so dns's
post-resolution re-dispatch lands back in cache's normalizeHeaders()
with headers already in that shape, misread as a broken pairs array.

Detect the flat-array shape by checking whether the first element is
itself an array. Also widened the value check (both the new branch and
the existing pairs branch) to accept string[] as well as string,
matching what client-h1.js already accepts for repeated headers like
Set-Cookie -- the stricter string-only check would have thrown again
on that case.
```

---

## 8. 가로채기당한 PR을 테스트 PR로 부활시키기 (2026-07-13)

> 이 섹션부터는 코드 딥다이브가 아니라 **OSS 정치·충돌 해결 전략**이 주제다.

### 8.1 전체 타임라인 — 14분의 간극

```
07-09 08:33  내가 #5535 오픈 — normalizeHeaders 버그 수정 + 단위 테스트 + 통합 테스트
07-09 08:47  ← 14분 뒤. mcollina(리드 메인테이너)가 같은 버그를 고치는 #5536을 직접 오픈
07-09 08:51  그가 내 PR에는 리뷰 코멘트 하나만 남김 ("use a proper for loop, .every() is slow")
07-10        나는 그 코멘트대로 수정해 push — CI 전부 통과 (여기까진 순항으로 보였음)
07-11 15:38  그가 자기 #5536을 셀프 머지. 내 이름·PR 언급·co-author 전부 없음
             → 내 PR은 CONFLICTING/DIRTY 상태로 전락
07-13        상황 파악 → 구제 결정 → 충돌 해결 → 테스트 PR로 재조준 → MERGEABLE 복구
```

핵심 장면: 그는 내 PR을 **보고**(4분 뒤 코멘트가 증거), 자기 버전을 **이미 만들어둔 상태**(14분 전 오픈)에서 나에게 스타일 수정을 시켰다. for 루프를 고치는 동안 그의 PR이 먼저 들어간 것. (선점 PR: https://github.com/nodejs/undici/pull/5536, Merged — 크레딧 없이 같은 버그를 수정)

### 8.2 원래 #5535가 고치던 버그의 3개 구성 요소와 각각의 운명

| 구성 | 내용 | 운명 |
|---|---|---|
| `lib/util/cache.js` | flat 배열 처리 추가 | 그의 #5536이 같은 수정을 머지 → **무효화** |
| `test/cache-interceptor/cache-utils.js` | 단위 테스트 | 그의 PR에도 있음 → **무효화** |
| `test/interceptors/dns.js` (+88) | **compose 체인 통합 테스트 2개** | 그의 PR에 **없음** → 구제 대상 ★ |

### 8.3 상황 판단 — "충돌 해결"의 의미가 다르다

머지 충돌이 떴을 때 물어야 했던 질문: **이 충돌은 무엇을 의미하는가?** 일반적인 충돌은 "내 변경과 main의 변경을 잘 섞어라"는 신호지만, 이 경우 main에는 **내 변경과 같은 목적의 코드가 이미(그의 버전으로) 들어와** 있었다. 기계적으로 충돌 마커를 풀며 병합하면 이미 고쳐진 걸 다르게 또 고치는 무의미한 diff가 남는다. 올바른 해석: **덜어내기(rescope)** — 겹치는 부분은 main의 버전을 그대로 수용(내 쪽 폐기)하고, main이 커버하지 않는 부분만 남긴다.

선행 조사 두 가지가 결정을 뒷받침했다:
1. **커버리지 비교**: `gh api …/pulls/5536/files`로 그의 PR이 건드린 파일 목록 확인 → `dns.js`가 없음 = 내 통합 테스트는 살릴 가치가 있음
2. **dns.js의 main 변동 확인**: `gh api "commits?path=test/interceptors/dns.js&since=…"` → #5536 이후 아무도 안 건드림 = **파일 단위로 통째로 가져와도 안전**

### 8.4 충돌 해결 실전 — 리베이스가 아니라 재건축

**시도: 정석 리베이스 → 즉시 포기**
```bash
git rebase origin/main
# → Could not apply 9be4822a... (첫 커밋부터 충돌)
git rebase --abort
```
커밋이 2개(원 수정 + for 루프 수정)인데 둘 다 `cache.js`를 건드리므로 커밋마다 충돌을 풀어야 하고, 어차피 해결 결과는 "전부 main 버전 채택"이다. 무의미한 노동.

**채택: main에서 재건축 + 파일 단위 체리픽**
```bash
git checkout -B fix-issue-5522-normalize-headers-interceptor-order origin/main
git checkout af670a29 -- test/interceptors/dns.js
git diff --stat HEAD   # 정확히 +88/−1 (dns.js만)
```
이 방법이 성립한 전제는 8.3절의 조사 2번(dns.js가 main에서 변하지 않음)이다. 충돌 해결이 끝났는데 충돌 마커를 한 번도 안 봤다 — **충돌의 의미를 먼저 해석하면 기계적 해결 자체를 우회할 수 있다.**

**검증 → 커밋 → force-push**
```bash
node --test test/interceptors/dns.js   # 29 passing / 0 failing (신규 2개 포함, '머지된 그의 구현' 위에서 통과 확인이 핵심)
npm run lint                           # 클린
git commit -m "test(interceptors): add cache/dedupe + dns re-dispatch integration tests …"
git push --force-with-lease fork <branch>   # 역사를 다시 썼으므로 force 필수
```
`--force-with-lease`: 무조건 밀어붙이는 `--force`와 달리 "원격이 내가 아는 상태일 때만" 덮어쓴다. push 직후 GitHub이 PR을 재평가: **CONFLICTING → MERGEABLE.**

### 8.5 PR 재조준 — 제목과 본문의 정체성 교체

코드만 바꾸면 반쪽이다. `gh pr edit`으로:
- **제목**: `fix: normalizeHeaders throws …` → **`test: add cache/dedupe + dns re-dispatch integration tests`**
- **본문**: 새 서사로 전면 교체 — ① "본체 버그는 #5536이 고쳤다"를 첫 줄에(경위를 담백하게, 원망 없이) ② 이 PR의 새 존재 이유: #5536의 단위 테스트가 커버하지 않는 **실제 발생 경로(compose 체인)의 통합 회귀 테스트** ③ `deduplicate()`도 같은 유틸을 소비하므로 동일하게 앓았던 경로 — 어디에도 커버가 없다는 점 ④ 상태 명시: main 리베이스 완료, 이전 lib 변경은 #5536 채택으로 폐기, 테스트 29/0, 린트 클린

리뷰어(= 그 mcollina)의 계산: 자기 수정의 회귀 방어 테스트를 공짜로 얻는다. 거절할 명분이 없다.

### 8.6 재조준 결과

| 항목 | 재조준 전 | 재조준 후 |
|---|---|---|
| 머지 가능성 | ❌ CONFLICTING/DIRTY | ✅ **MERGEABLE** |
| 정체성 | 이미 머지된 수정의 중복 | 테스트 전용 additive PR |
| 커밋 | 3개 (2/3 무효) | 1개 클린 커밋 (+88/−1) |
| AI 흔적 | — | 0 (커밋 메시지 grep 검증) |

### 8.7 이 과정의 핵심 교훈

1. **충돌 마커를 풀기 전에 충돌의 의미부터 해석하라.** "내 변경 vs main의 변경"이 경쟁 관계(같은 목적의 다른 구현)라면, 정답은 병합이 아니라 **한쪽의 폐기**다.
2. **파일 단위 체리픽의 전제조건을 먼저 확인하라.** `git checkout <옛커밋> -- <파일>`이 안전하려면 그 파일이 main에서 안 변했어야 한다. `gh api "commits?path=…&since=…"` 한 줄이 그 확인이다.
3. **가로채기당했을 때 감정과 실리를 분리하라.** 크레딧 없는 재구현은 어느 커뮤니티에서든 무례지만, 항의 코멘트는 실익이 없다. 대신 물을 것: "그가 안 가져간 것 중 가치 있는 게 뭔가?"
4. **버그 발견 자체가 이전 불가능한 성과다.** 메인테이너가 14분 만에 자기 버전을 만들 수 있었던 건 문제 정의·재현·수정 방향이 내 PR에 다 차려져 있었기 때문이다. 코드는 뺏겨도 "그 버그를 찾아낸 사람"이라는 사실은 남는다.
5. **재조준한 PR은 정체성(제목·본문)까지 갈아입혀라.** diff만 바꾸고 제목이 옛날 것이면 리뷰어는 맥락을 재구성하느라 피로하다.
6. **같은 저장소 안에서도 PR 유형별 리스크가 다르다.** 같은 주에 같은 사람(mcollina)이 코어 버그 수정(#5535)은 가로챘고, 코어 동작 수정([[undici-pr5538-h2-sse-post-hang]])은 lgtm 승인했다. 이 저장소에서 버그 수정 PR은 스나이핑 리스크를 안고 가되, 테스트·주변부 PR은 안전하다는 것이 경험칙.

### 8.8 개념 미니 노트 (복습용)

- **CONFLICTING / DIRTY**: GitHub이 "이 PR을 지금 main에 합치면 충돌"이라 판정한 상태.
- **rescope(재조준)**: PR의 범위·정체성을 줄이거나 바꾸는 것. 경쟁 구현에 선점당했을 때의 표준 생존 전략.
- **`git checkout -B <branch> <base>`**: 브랜치를 base 위치로 강제 재생성 — "역사를 버리고 재건축"의 도구.
- **`git checkout <commit> -- <path>`**: 특정 커밋의 특정 파일만 현 작업 트리로 — 파일 단위 체리픽.
- **`--force-with-lease`**: 원격이 예상 상태일 때만 덮어쓰는 안전한 force push.
- **단위 테스트 vs 통합 테스트의 상보성**: #5536은 함수 하나를 찔렀고(단위), 구제한 테스트는 버그가 실제로 발생한 경로 전체를 찌른다(통합). 같은 버그라도 커버하는 회귀의 표면이 다르다.

---

## 9. 결과 및 회고 — 🟡 결과가 나오면 이 섹션을 갱신할 것

### 9.1 현재 상태 (2026-07-20 기준)

- PR #5535 **MERGED** (2026-07-28 21:28, `mcollina`) — 8번 섹션의 재조준 이후. 정체성은 "버그 수정"에서 "테스트 전용 additive PR"로 완전히 바뀐 채로 머지됨
- 원래 고치던 버그는 메인테이너의 선점 PR #5536(Merged)으로 이미 해결됨 — 크레딧 없음
- 메인테이너 `mcollina`가 재조준 전 실제로 인라인 리뷰를 남겼었음(6.4절) — 이 저장소 리뷰 속도가 빠르다는 사전 조사(1.2절)가 실전으로 확인됨
- **`metcoder95`(메인테이너)가 2026-07-16 APPROVED**
- **2026-07-28**, 관련 있던 또 다른 선점 PR `#5536`이 머지되자, 그와 겹치는 구현 변경분을 마저 걷어내고 "테스트만 남았다, 머지 준비됐다"는 코멘트를 남겼고, 같은 날 `mcollina`가 머지함

### 9.1-1 CI 관련 참고

이 PR 자체는 CI 실패가 없었음 — CI 이슈는 같은 날 함께 작업한 PR #5538 쪽에서 발생 ([[undici-pr5538-h2-sse-post-hang]] 참고).

### 9.2 지금까지 배운 점

- **`compose()`의 감싸기 순서가 직관과 반대라는 게, 이 이슈 전체의 근본 원인**이었음 — 이런 "코드를 안 읽고 이름만 보면 오해하기 쉬운" 함정이 실제 프로덕션 버그로 이어진 좋은 사례.
- **비동기 콜백이 나중에 다시 파이프라인에 재진입할 때, 그 사이에 자기 자신이 데이터의 모양을 바꿔놓았다는 걸 깜빡하기 쉽다**는 일반적인 교훈 — dns 인터셉터는 자기 다음 애가 항상 "저수준 코어"라고 가정하고 짜여 있었는데, 실제로는 컴포지션 순서에 따라 다른 인터셉터가 올 수도 있다는 걸 놓친 것.
- **하나의 유틸 함수를 여러 인터셉터가 공유하면, 그 함수의 버그가 한 곳만이 아니라 공유하는 모든 곳에 동시에 존재한다** — `cache`뿐 아니라 `deduplicate`도 똑같이 취약했던 걸 스스로 찾아낸 게 이번 기여의 가치를 높인 지점.
- **리뷰가 "버그가 있냐 없냐"만 보는 게 아니라 "고친 게 원래 문제를 완전히 커버하냐"까지 봐야 한다** — 1차 fix는 이슈의 정확한 재현 케이스(`headers: []`)는 고쳤지만, "flat 배열이 들어올 수 있다"는 일반화 자체가 살짝 좁아서(문자열만 허용) 실제 undici가 지원하는 다른 케이스(배열 값)에서는 또 터질 수 있었음 — 실제 다운스트림 소비 코드(`client-h1.js`)까지 직접 대조해서 확인해야 이런 좁은 일반화를 잡아낼 수 있었음.
- **"이 실패가 우리 변경 때문인지" 항상 stash로 재확인하는 습관이 이번에도 유효했음** — `ERR_SSL_EE_KEY_TOO_SMALL`이라는 낯선 실패를 보고 바로 "우리 탓인가"로 넘어가지 않고, 변경 없이도 재현되는지부터 확인해서 무관하다는 걸 직접 확인함.
- 이번 세션에서 처음으로, **사용자가 이슈를 스스로 찾아 제시하고, 재현부터 수정, 여러 차례의 검토 반영까지 전 과정을 별도로 진행**한 사례 — 지금까지의 5개 저장소/8개 PR 중 협업 방식이 가장 "표준적인 AI 페어 프로그래밍" 형태에 가까웠음.
- **머지된 버그 수정 자체보다 "겹치지 않는 나머지 가치"를 정확히 골라내는 능력이 최종 결과를 갈랐다** — 8번 섹션 참고. 코드를 뺏겨도 통합 테스트라는 잔여 가치가 있었고, 그걸 알아본 게 이 PR을 살렸다.

### 9.3 다음에 채워야 할 것 (9.1 시점 기준)

- [x] 재조준된 테스트 전용 PR에 대한 메인테이너 리뷰 코멘트 내용과 그에 대한 대응 — 겹치는 구현 변경분 최종 제거로 마무리
- [x] 최종 머지 여부 및 머지까지 걸린 시간 — 2026-07-28 21:28 머지 (PR 제출 후 약 19일)
- [ ] `evm.ts`처럼 더 central한 파일을 다루는 유사 패턴이 undici에도 있는지 살펴볼지 여부
