# [OSS 기여] nodejs/corepack - Node 26.5.0(Undici 8.7.0) 프록시 동작 변화로 깨진 PROXY 테스트 수정 (PR #869) — 🟢 Merged

> 오픈소스 기여 기록 및 학습 노트. 이 문서는 PR #869 하나만 다룬다. **2026-08-12 `aduh95`가 승인 후 머지.**
>
> [[node-postgres-pr3708-rows-query-error-sync]]와 같은 작업 방식 — fork/브랜치에서 코드를 완전히 직접 작성한 뒤 AI 도구로 리뷰만 받았다. **같은 시기에 다룬 undici(#5539 작업)와 직접 연결되는 지점**이 있었다 — 원인 자체가 "Undici 8.7.0의 프록시 처리 방식 변화"였다.

---

## 1. 개요

### 1.1 프로젝트 소개

- **저장소**: https://github.com/nodejs/corepack
- **뭐 하는 도구인가**: Node.js에 기본 내장된 패키지 매니저 관리 도구. `yarn`, `pnpm` 같은 패키지 매니저를 프로젝트별로 지정된 버전으로 자동 다운로드해서 실행해주는 역할을 한다 (`package.json`의 `packageManager` 필드를 읽어서 동작). Node.js 프로젝트 자체에 포함되는 공식 도구.

### 1.2 이슈 (#868)

**제목**: "CI failure with Node.js 26.5.0 testing PROXY"

Corepack의 `AUTH_TYPE=PROXY` 테스트 그룹이 Node.js 26.5.0에서 실패한다는 CI 실패 리포트. 원인 분석까지 마친 상태에서 작업을 시작했다.

### 1.3 관련 링크

| 구분 | 링크 |
|---|---|
| 원본 저장소 | https://github.com/nodejs/corepack |
| 내 Fork | https://github.com/GiHoon1123/corepack |
| 이슈 (#868) | https://github.com/nodejs/corepack/issues/868 |
| PR (#869) | https://github.com/nodejs/corepack/pull/869 |
| 리뷰 제안 | https://github.com/nodejs/corepack/pull/869#discussion_r3624151298 |
| 리뷰 답변 | https://github.com/nodejs/corepack/pull/869#discussion_r3655681799 |
| 핑 답변 | https://github.com/nodejs/corepack/pull/869#issuecomment-5089290483 |
| 작업 브랜치 | `fix-issue-868-proxy-test-node-26` |
| 커밋 | `9638fee`, `a15c217` |
| 상태 | 🟢 Merged (2026-08-12) |

`MikeMcC399`가 first-time contributor 워크플로우 승인과 재현 테스트를 여러 차례 대신 해주면서 메인테이너 `aduh95`에게 몇 번 더 리뷰를 독려해줬고, `aduh95`가 최종 승인 후 직접 머지했다.

### 1.4 한 줄 요약

> Node 26.5.0에 들어간 Undici 8.7.0이 `http://` 요청을 프록시로 보내는 방식을 바꿨는데(CONNECT 터널 대신 절대 URI로 직접 요청), 테스트용 mock 프록시가 이 새로운 방식을 아예 처리 안 하고 무조건 `"okay"`만 응답해서, corepack이 그걸 레지스트리 JSON/tarball로 파싱하려다 실패했다. mock 프록시가 이 요청을 실제 mock 레지스트리 핸들러로 넘기도록 고쳤다.

---

## 2. 배경 지식

### 2.1 HTTP 프록시를 거칠 때 요청이 오는 두 가지 방식

**방식 A — CONNECT 터널링**: HTTPS처럼 암호화된 트래픽을 프록시에 넘길 때 쓴다. 클라이언트가 프록시한테 "나 `example.com:80`이랑 통신할 거니까 파이프처럼 연결만 뚫어줘"(`CONNECT example.com:80`)라고 요청하면, 프록시는 그 뒤로 내용을 안 보고 바이트를 그대로 통과시킨다.

**방식 B — 일반 HTTP 프록시 요청**: 암호화 안 된 `http://`는 클라이언트가 요청 첫 줄에 **전체 주소**를 써서 프록시한테 직접 보낸다:
```
GET http://example.com/some-package HTTP/1.1
```
(프록시를 안 거치면 `GET /some-package HTTP/1.1`처럼 경로만 쓰지만, 프록시를 거치면 전체 URL을 쓰는 게 HTTP 프록시의 표준 방식이다.) 이 경우 프록시가 그 요청을 직접 이해하고 응답을 만들어줘야 한다.

**이번 문제의 핵심**: Node 26.5.0(Undici 8.7.0)이 `http://` 요청을 방식 B로 보내기 시작했는데, corepack 테스트의 mock 프록시는 방식 A만 구현돼 있었다.

### 2.2 `req.url`이 두 방식에서 다르게 생기는 이유

Node의 HTTP 서버는 요청 첫 줄의 그 부분을 그대로 `req.url`에 담아준다.
- 방식 A(CONNECT)는 별도 이벤트(`'connect'`)로 처리되어 이번 문제와 무관.
- 방식 B로 들어오면 `req.url`이 **전체 주소 문자열**(`"http://example.com/some-package"`)이 된다 — 원래 mock 프록시는 이런 형태를 상상도 안 하고 그냥 `"okay"`만 응답하고 있었다.

---

## 3. 원인 분석

`tests/_registryServer.mjs`에서 진짜 mock 레지스트리 응답 로직은 `server`라는 객체(`createServer((req, res) => { ... })`)에 이미 다 구현돼 있다. `AUTH_TYPE=PROXY`일 때는 별도로 `proxy`라는 서버를 하나 더 띄워서, 그게 진짜 프록시처럼 동작하는지 흉내낸다.

```js
// 수정 전
const proxy = createServer((req, res) => {
  res.writeHead(200, {[`Content-Type`]: `text/plain`});
  res.end(`okay`);   // <- 방식 B로 들어온 요청도 무조건 이렇게만 응답
});
```

Node 26.5.0에서 corepack이 방식 B로 레지스트리에 요청을 보내면, 이 `proxy`가 무조건 `"okay"`라는 평문을 응답으로 돌려준다. corepack은 이걸 JSON이나 tarball로 파싱하려다:
```
SyntaxError: Unexpected token 'o', "okay" is not valid JSON
```
또는
```
TAR_BAD_ARCHIVE
```
로 실패한다.

---

## 4. 수정 내용

```diff
 const proxy = createServer((req, res) => {
-  res.writeHead(200, {[`Content-Type`]: `text/plain`});
-  res.end(`okay`);
+  const url = URL.parse(req.url);
+
+  if (url?.protocol !== `http:` || url.host !== `example.com`) {
+    res.writeHead(404).end(`Not Found`);
+    return;
+  }
+
+  req.url = `${url.pathname}${url.search}`;
+  server.emit(`request`, req, res);
 });
```

### 4.1 각 줄이 하는 일

1. **`URL.parse(req.url)`**: `req.url`이 `"http://example.com/some-package"`처럼 전체 주소 형태면 파싱에 성공해서 `url.protocol`(`"http:"`), `url.host`(`"example.com"`), `url.pathname`(`"/some-package"`) 등을 조각조각 꺼낼 수 있다. URL이 아니면 예외를 던지지 않고 `null`을 반환한다.
2. **`url?.protocol !== 'http:' || url.host !== 'example.com'` 체크**: 이 mock 프록시는 테스트용 가짜 호스트(`example.com`)만 다룬다는 안전장치 — CONNECT 핸들러 쪽의 `req.url !== 'example.com:80'` 체크와 같은 스타일로 맞춘 것. `URL.parse()`가 실패해서 `url`이 `null`이면 `url?.protocol`은 `undefined`가 되고, 조건문에서 자연스럽게 404로 빠진다.
3. **`req.url = url.pathname + url.search; server.emit('request', req, res)`**: 여기가 핵심. 진짜 응답 로직은 `server`가 갖고 있으니, 이 요청을 새 연결로 다시 보내는 대신 **`req.url`을 상대 경로로 바꿔치기하고 `server`의 요청 핸들러를 직접 호출**한다. `req`/`res`는 여전히 `proxy`가 만든 실제 소켓에 연결돼 있어서, `server`의 로직이 `res.end(...)`를 하면 그 응답이 실제로 (프록시를 거쳐) corepack한테 그대로 전달된다.

비유하면: `server`는 "진짜 요리사", `proxy`는 "주문 받는 창구"다. 방식 B로 주문이 오면 창구 직원이 직접 요리하는 대신, 주문서(`req.url`)만 살짝 고쳐서 요리사(`server`)한테 그대로 넘겨버리는 것 — 새 주방을 여는 게 아니라 이미 있는 로직을 재사용하는 기법이다.

CONNECT 핸들러(방식 A)는 손대지 않았다 — 여전히 유효한 경로(예: 오래된 Node/Undici, 실제 TLS 터널이 필요한 경우)라서 안 건드리는 게 맞다.

### 4.2 리뷰 반영: `new URL()` + `try/catch`를 `URL.parse()`로 단순화

최초 PR 커밋(`9638fee`)에서는 이 부분을 이렇게 작성했다.

```js
let url;
try {
  url = new URL(req.url);
} catch {
  res.writeHead(404).end(`Not Found`);
  return;
}

if (url.protocol !== `http:` || url.host !== `example.com`) {
```

리뷰어 `aduh95`는 이걸 아래처럼 줄이자고 제안했다.

```js
const url = URL.parse(req.url);

if (url?.protocol !== `http:` || url.host !== `example.com`) {
```

둘의 차이는 실패 처리 방식이다.

- `new URL(req.url)`은 URL이 잘못되면 예외를 던진다. 그래서 `try/catch`가 필요하다.
- `URL.parse(req.url)`은 URL이 잘못되면 `null`을 반환한다. 그래서 `try/catch` 없이 `if` 조건문 안에서 처리할 수 있다.

이 변경은 기능을 바꾸는 게 아니라, 같은 404 처리 흐름을 더 짧고 직접적으로 표현하는 리팩터링이다. Corepack의 현재 `engines`는 `^22.22.2 || ^24.15.0 || >=26.0.0`이고, 로컬 Node `v22.22.3`에서도 `URL.parse`가 존재하는 걸 확인했으므로 버전 호환성 문제도 없었다.

---

## 5. 검증 — Node 26.5.0을 직접 설치해서 재현

이슈에 있던 실패를 그대로 재현하기 위해, `nvm install 26.5.0`으로 정확히 그 버전을 설치했다.

```
$ node -e "console.log(process.versions.undici)"
8.7.0
```

이슈에서 말한 그대로 Undici 8.7.0이 번들된 걸 확인.

**픽스를 잠깐 끄고**(`git checkout --`) PROXY 테스트 그룹(`vitest run -t "PROXY"`)을 Node 26.5.0으로 돌렸더니:

```
FAIL  tests/main.test.ts > custom registry with auth PROXY > should download custom package manager
Error: TAR_BAD_ARCHIVE: Unrecognized archive format
Tests  4 failed | 110 skipped (114)
```

이슈에 있던 증상과 정확히 일치. 픽스를 복구하고 다시 돌리니:

```
Test Files  1 passed (1)
Tests  4 passed | 110 skipped (114)
```

전체 `main.test.ts`(114개)도 Node 26.5.0과 Node 22 양쪽으로 돌려서 **완전히 동일한 결과**(113 통과 + `it.fails`로 의도된 실패 1개, 이번 수정과 무관) 확인 — 다른 `AUTH_TYPE`(COREPACK_NPM_REGISTRY, COREPACK_NPM_TOKEN, COREPACK_NPM_PASSWORD)이나 다른 테스트에 회귀 없음.

### 5.1 리뷰 제안 반영 후 재검증

`a15c217`에서 `URL.parse()` 제안을 반영한 뒤에는 현재 로컬 Node `v22.22.3`에서 다시 확인했다.

```bash
yarn install
```

의존성 설치는 통과.

처음에 바로 테스트를 돌렸을 때는 실패했다.

```bash
yarn test tests/main.test.ts
```

실패 원인은 코드 변경이 아니라 `dist/corepack.js`가 없어서였다.

```text
Cannot find module '../dist/corepack.js'
```

이 저장소의 `main.test.ts`는 빌드 산출물인 `dist/corepack.js`를 실행하므로, 테스트 전에 빌드가 필요하다. 그래서 먼저 빌드를 돌렸다.

```bash
yarn build
```

결과:

```text
dist/lib/corepack.cjs  507.4kb
All shims have been generated.
```

그 뒤 다시 전체 `main.test.ts`를 돌렸다.

```bash
yarn test tests/main.test.ts
```

결과:

```text
Test Files  1 passed (1)
Tests  113 passed | 1 expected fail (114)
Duration  135.77s
```

추가로 `yarn lint`와 `git diff --check`도 통과했다.

---

## 6. 작업 중 특이사항 — fork 리모트가 없었음

이 저장소는 이전에 다룬 다른 저장소들과 달리 로컬 clone에 `origin`(nodejs/corepack 본체)만 있고 fork가 설정 안 돼 있었다. `gh repo fork nodejs/corepack --remote-name=fork`로 fork를 만들었는데, 명령어가 성공했다고 보고했음에도 실제로는 리모트가 안 잡혀서, `git remote add fork https://github.com/GiHoon1123/corepack.git`으로 수동으로 다시 추가해야 했다.

---

## 7. 리뷰/설명 — 직접 짠 코드를 다시 재설명 받음

코드는 직접 작성했지만, AI 도구에게 리뷰를 받는 중 설명 들은 내용(HTTP 프록시의 두 방식, `server.emit('request', ...)` 기법)이 한 번에 이해가 안 가서 다시 설명을 요청했다. 두 번째 설명에서는 방식 A/B의 차이부터, `req.url`의 형태 차이, `new URL()`의 파싱-실패를 판별 수단으로 쓰는 방식, 그리고 "창구 직원이 주문서를 고쳐서 요리사한테 넘긴다"는 비유까지 단계별로 풀어서 이해에 도움이 됐다 — [[node-redis-pr3329-sentinel-unreachable-master-error-event]]에서 이미 확인한 "설명 요청이 새 버그를 찾으려는 게 아니라 이해를 확인하는 것"이라는 패턴이, 이번엔 **직접 짠 코드를 다시 설명 들으며 확인하고 싶어지는** 형태로 한 번 더 나타났다.

---

## 8. 결과 및 회고

### 8.1 현재 상태 (2026-08-12 기준, 🟢 Merged)

- PR #869 **머지됨** (2026-08-12, `aduh95`)
- 현재까지 `nodejs/corepack`에 올린 내 PR은 #869 하나만 확인됨. 같은 저장소에 작성한 이슈는 없음.
- 커밋 2개:
  - `9638fee` — PROXY mock이 plain HTTP proxy 요청을 mock registry로 위임하도록 수정
  - `a15c217` — 리뷰 제안 반영: `new URL()` + `try/catch` 대신 `URL.parse()` 사용
- 코드는 전부 직접 작성, 원인 재현·실행 검증·PR 본문 작성·코드 설명은 AI 도구의 도움을 받아 진행
- **원본 이슈(#868) 리포터 `MikeMcC399`가 코멘트로 격려**: 자기 fork에서 워크플로우를 직접 돌려봤고, 메인테이너가 워크플로우 실행을 승인하면 통과해서 머지 준비가 될 것 같다고 언급.
- 2026-07-21에 `aduh95`가 `URL.parse()` 사용을 제안하는 리뷰 코멘트를 남김.
- 2026-07-27에 `MikeMcC399`가 리뷰 제안에 응답 가능한지 핑했고, 제안 반영 후 일반 코멘트로 답변함. 해당 답변에는 하트 반응이 붙음.
- 같은 날(2026-07-27) `MikeMcC399`가 이어서 코멘트: "첫 기여자라 워크플로우가 자동으로 안 돌아서, 자기 fork에서 직접 클론해서 돌려봤고 성공했다. 자기가 보기엔 이 PR은 준비됐다(ready)"고 명시. `mergeStateStatus`는 여전히 `BLOCKED`인데, 이건 코드에 문제가 있어서가 아니라 첫 기여자 PR은 메인테이너가 워크플로우 실행을 승인해야 CI가 도는 corepack 저장소 정책 때문(정상 상태).
- 2026-07-29에 `MikeMcC399`가 `aduh95`한테 "제안하신 변경이 반영됐으니 다시 봐달라"고 재확인 요청.
- 2026-08-05, Node.js 26.7.0이 실제로 릴리즈되자 `MikeMcC399`가 **Windows 11 25H2 + Node 26.7.0에서 직접 테스트**해서 이 PR이 실제 이슈(#715, `exitCode: 3221226505`)를 해결한다고 확인해줌 — 로컬 재현/유닛 테스트를 넘어, 문제를 처음 보고했던 실제 버전으로 실사용 환경에서 검증까지 된 상태.
- 2026-08-12, `aduh95`가 승인(APPROVED)하고 8분 뒤 바로 머지. 별도 코멘트 없이 승인만 남기고 머지 — `MikeMcC399`의 반복된 핑과 실사용 환경 검증이 리뷰 재개의 실질적 계기였던 것으로 보인다.

### 8.2 지금까지 배운 점

- **"실행 검증"의 가장 확실한 형태는 이슈가 보고한 정확한 버전을 직접 설치해서 재현하는 것이다** — 이번엔 `nvm install 26.5.0`으로 정확히 그 버전을 깔고, 번들된 Undici 버전(`process.versions.undici`)까지 이슈 내용과 일치하는지 확인한 뒤 재현했다. "비슷한 버전이면 되겠지"가 아니라 정확히 같은 버전으로 맞추는 게, 재현 여부에 대한 확신의 질을 완전히 다르게 만든다.
- **테스트 mock을 고칠 때, 새로 필요해진 경로만 추가하고 기존에 잘 동작하던 경로(CONNECT)는 안 건드리는 최소 변경 원칙**이 이번에도 유효했다.
- **기존 로직을 재사용하는 `emit()` 기반 위임 기법**은 목(mock) 서버를 여러 개 두고 그중 하나가 다른 하나의 로직을 빌려 쓰고 싶을 때 유용한 패턴이다 — 소켓을 새로 만들지 않고 이벤트 핸들러만 재호출하는 방식.
- **그날 undici → node-redis → node-postgres → corepack 순서로 작업이 이어지면서, 앞서 다룬 라이브러리(undici)의 변경이 뒤에서 다루는 완전히 다른 프로젝트(corepack)의 버그 원인이 되는 걸 직접 목격했다** — 오픈소스 생태계가 서로 얼마나 촘촘하게 얽혀있는지 체감하는 계기였다.
- **리뷰어가 항상 "더 큰 수정"을 요구하는 건 아니다.** 이번 리뷰는 동작을 더 바꾸라는 게 아니라, `new URL()`의 예외 흐름을 `URL.parse()`의 `null` 반환 흐름으로 바꿔 더 단순하게 쓰자는 제안이었다. 이런 코멘트는 길게 논쟁하지 말고, 지원 버전만 확인한 뒤 그대로 반영하는 게 낫다.
- **테스트 실패 원인을 구분해야 한다.** 리뷰 제안 반영 후 첫 `main.test.ts` 실패는 코드 회귀가 아니라 빌드 산출물(`dist/corepack.js`)이 없어서 난 환경 준비 문제였다. `yarn build` 후 같은 테스트가 통과했으므로, 실패 로그를 보고 바로 코드 문제로 단정하지 않는 게 중요하다.

### 8.3 다음에 채워야 할 것 (결과가 나오면)

- [x] 리뷰 제안 1건 반영
- [x] 리뷰 제안 반영 후 로컬 테스트 재실행
- [x] 추가 리뷰/승인 여부 — 리포터가 자기 fork에서 워크플로우 직접 실행해서 통과 확인, "이 PR은 준비됐다"고 코멘트 (2026-07-27)
- [x] 실사용 환경 검증 — 리포터가 Node 26.7.0 릴리즈 직후 Windows 11에서 실제로 확인 (2026-08-05)
- [x] 메인테이너의 워크플로우 승인 및 실제 CI 결과 — 통과, `aduh95`가 직접 승인
- [x] 머지 여부와 소요 시간 — 2026-08-12 머지, 승인 후 8분
