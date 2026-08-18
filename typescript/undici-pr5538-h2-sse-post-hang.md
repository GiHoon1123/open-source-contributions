# [OSS 기여] nodejs/undici - HTTP/2 세션에서 SSE 스트림이 열려있으면 POST가 영원히 멈추는 문제 수정 (PR #5538) — ✅ 머지됨 + 이미 열려있는 메인테이너 PR과의 관계 조율

> 오픈소스 기여 기록 및 학습 노트. 이 문서는 PR #5538 하나만 다룬다. **머지 완료 (9번 섹션 참고).**
>
> 코드와 테스트는 직접 작성했고, 안전성 근거를 코드로 끝까지 추적하는 리뷰는 AI 도구의 도움을 받았다. **이번엔 지금까지 없었던 새로운 상황을 하나 마주쳤다** — 고치려던 코드와 정확히 같은 코드를 이미 지우는, 다른 이슈를 위해 열린 메인테이너 PR이 존재하고 있었다는 것. 이 문서는 그 발견과 대응 과정도 자세히 다룬다.

---

## 1. 개요

### 1.1 프로젝트 소개

- **저장소**: https://github.com/nodejs/undici (PR #5535과 동일 저장소 — [[undici-pr5535-normalize-headers-interceptor-order]] 참고)
- **건드린 부분**: `lib/dispatcher/client-h2.js`의 `busy(request)` — HTTP/2 클라이언트가 "지금 이 요청을 당장 보낼 수 있는지, 아니면 미뤄야 하는지"를 판단하는 함수.

### 1.2 이슈 (#5524)

**제목**: "HTTP/2: `fetch` POST hangs while an SSE stream is open"

이슈 등록자가 이미 스스로 상당히 깊은 원인 분석을 해놓은, 매우 잘 작성된 버그 리포트였음. 라벨 `bug`, 등록 2026-07-08.

### 1.3 관련 링크

| 구분                                      | 링크                                                                    |
| ----------------------------------------- | ----------------------------------------------------------------------- |
| 원본 저장소                               | https://github.com/nodejs/undici                                        |
| 내 Fork                                   | https://github.com/GiHoon1123/undici                                    |
| 이슈 (#5524)                              | https://github.com/nodejs/undici/issues/5524                            |
| 이슈 (#5494, 이 PR로 같이 해결됨 — 5.4절) | https://github.com/nodejs/undici/issues/5494                            |
| PR (#5538)                                | https://github.com/nodejs/undici/pull/5538                              |
| 작업 브랜치                               | `fix-issue-5524-h2-sse-post-hang`                                       |
| 커밋                                      | `bdb226f6d` (가드 삭제 + #5524 테스트), `46eda23b8` (#5494 테스트 추가) |
| PR 생성 시각                              | 2026-07-09 09:11:40 UTC (18:11 KST)                                     |
| 상태                                      | ✅ Merged (2026-07-13T23:17:07Z, `mcollina`가 직접 머지)                |
| 관련 메인테이너 PR (#5497, 우리 것 아님)  | https://github.com/nodejs/undici/pull/5497 — 5번 섹션에서 자세히 다룸   |

**참고**: 이슈 번호(#5524)보다 PR 번호(#5538)가 더 큰 건 정상이다 — GitHub은 한 저장소에서 이슈와 PR이 번호를 하나의 카운터로 같이 쓰기 때문에, 이슈가 먼저 등록되고 그 사이에 다른 이슈/PR이 몇 개 더 생긴 뒤에야 우리 PR이 올라가면 이렇게 된다. 겹치거나 잘못 연결된 게 아니다.

### 1.4 한 줄 요약

> HTTP/2 세션 위에서, 다른 스트림(예: 계속 열려있는 SSE 응답)이 아직 끝나지 않았으면 **바디가 있는 요청(스트림/이터러블/FormData 바디)을 무조건 미루는 가드**가 있었는데, SSE처럼 "영원히 안 끝나는 스트림"이 있으면 그 뒤에 걸린 요청들이 **영원히 멈춤**. 이 가드가 원래 막으려던 위험은 이미 다른 더 정교한 메커니즘(에러 격리, GOAWAY 재시도 시 바디 타입 체크)이 대체하고 있어서 안전하게 지울 수 있었음.

---

## 2. 배경 지식

### 2.1 HTTP/2의 "멀티플렉싱" — 여러 요청을 하나의 연결로

HTTP/1.1은 한 연결에서 기본적으로 요청을 하나씩 순서대로만 처리한다(파이프라이닝을 써도 제약이 많음). **HTTP/2는 하나의 연결(세션) 위에서 여러 "스트림"을 동시에 독립적으로 주고받을 수 있다** — 이게 멀티플렉싱. 그래서 HTTP/2에서는 "요청 A가 아직 응답 안 왔다고 요청 B를 못 보내는" 제약이 원래는 없어야 한다.

### 2.2 `busy(request)`가 하는 일

undici의 h2 클라이언트는 새 요청이 들어올 때마다 `busy(request)`를 호출해서 "지금 당장 이 요청을 스트림으로 열어서 보낼까, 아니면 큐에서 기다리게 할까"를 결정한다. `true`를 리턴하면 "바쁘다, 나중에"라는 뜻.

### 2.3 이번에 지운 가드가 원래 하려던 일

```js
if (
  util.bodyLength(request.body) !== 0 &&
  (util.isStream(request.body) ||
    util.isAsyncIterable(request.body) ||
    util.isFormDataLike(request.body))
)
  return true;
```

"다른 요청이 하나라도 진행 중이면, **바디가 스트림/이터러블/FormData인 요청은 무조건 미뤄라**"는 규칙. `fetch()`는 문자열/객체로 넣은 바디도 내부적으로 전부 `ReadableStream`으로 바꿔버리기 때문에, **`fetch()`로 보내는 바디 있는 요청은 거의 다 이 가드에 걸림.**

### 2.4 왜 SSE(Server-Sent Events)가 이걸 "영원한 멈춤"으로 만드는가

SSE는 서버가 연결을 계속 열어둔 채로 이벤트를 하나씩 흘려보내는 방식 — **의도적으로 절대 안 끝나는 스트림**이다. 이런 스트림이 h2 세션에 하나 떠 있으면, `client[kRunning] > 0`(진행 중인 요청이 있다)가 **영원히 참**이 되고, 위 가드는 그 뒤에 오는 모든 바디 있는 요청을 영원히 미룬다.

### 2.5 왜 "두 번째 연결을 새로 열어서 우회"도 안 되는가

보통 클라이언트가 "바쁘다"고 판단하면, 상위의 `Pool`이 "그럼 새 연결을 하나 더 열어서 그쪽으로 보내자"고 판단할 수 있다. 근데 h2는 원래 "하나의 연결로 멀티플렉싱하는 게 정상"이기 때문에, undici는 **h2 세션이 붙어있는 동안은 `kPending > 0`이어도 "바쁘다"고 보고를 안 하도록** 일부러 설계돼 있다("우리는 동시 dispatch가 세션에 멀티플렉싱되길 원한다"는 의도). 그러니까 이 가드(계속 미룸) + 이 설계(새 연결도 안 열어줌)가 합쳐지면, **바디 있는 요청은 도망갈 데가 없다.**

### 2.6 원래 가드가 걱정했던 두 가지 위험 (주석 원문)

```
// Request with stream or iterator body can error while other requests
// are inflight and indirectly error those as well.
//
// Request with stream or iterator body cannot be retried.
// Ensure that no other requests are inflight and
// could cause failure.
```

① 스트림 바디 요청이 에러나면, 같이 떠 있던 다른 요청들도 덩달아 에러날 수 있다.
② 스트림 바디는 재시도가 안 된다(한 번 읽으면 되돌릴 수 없으니까) — 다른 요청이 inflight일 때 뭔가 잘못되면 위험하다.

이 두 가지가 **지금도 여전히 사실인지**를 코드로 직접 추적한 게 이번 리뷰의 핵심이었다 (4번 섹션).

---

## 3. 재현

이슈 등록자가 제공한 스크립트를 그대로 실행 → 정확히 재현됨: SSE GET을 먼저 열어두고, 그 다음 POST를 보내면 5초 타임아웃까지 걸어놨는데도 응답이 안 옴 (`TimeoutError`).

---

## 4. 코드 리뷰 — "이 가드를 지워도 정말 안전한가?"를 코드로 직접 검증

### 4.1 검증 방법론

원 코멘트의 두 걱정(①에러 전파, ②재시도 안전성)이 **지금 코드베이스의 다른 곳에서 이미 더 정교하게 처리되고 있는지**를 실제로 소스 코드를 따라가며 확인함 — 이슈나 PR 설명을 그냥 믿지 않고 직접 검증.

### 4.2 ①번(에러 전파) 검증 — `abort()` 클로저

```js
const abort = (err, resetPendingIdx = false) => {
  ...
  if (state.stream != null) {
    clearRequestStream(request)
    const stream = state.stream
    stream.close()             // 이 스트림만 닫음
    ...
  }
  // We do not destroy the socket as we can continue using the session
  // the stream gets destroyed and the session remains to create new streams
  util.destroy(state.body, err)
}
```

**한 요청의 에러는 그 요청의 스트림 하나만 정리하고, 세션(연결) 자체는 그대로 살아있다.** 다른 요청들에게 안 번짐. 주석에도 명시적으로 "소켓은 안 부순다, 세션은 계속 쓸 수 있으니까"라고 적혀 있음.

### 4.3 ②번(재시도 안전성) 검증 — `canRetryRequestAfterGoAway()`

GOAWAY(서버가 "더 이상 새 스트림 받지 않겠다"고 알리는 프레임)를 받으면, 진행 중이던 요청들을 다시 시도할지/에러 처리할지 결정하는 함수:

```js
function canRetryRequestAfterGoAway(request) {
  const { body } = request;
  return body == null || util.isBuffer(body) || util.isBlobLike(body);
}
```

**스트림/이터러블/FormData 바디는 이 조건에 안 걸려서 재시도 대상에서 명시적으로 빠지고, 대신 그냥 에러 처리된다.** 즉 "재시도가 안 되는 바디를 억지로 재시도해서 망가지는" 상황 자체가 이미 원천적으로 차단돼 있음.

**이 함수가 도입된 시점을 git log로 확인** → 2026년 4월 26일 커밋(`5878f54db`, "fix(http2): preserve accepted streams after GOAWAY (#5090)")에서 도입, **이미 우리 기준 몇 달 전에 머지돼서 현재 브랜치에 포함돼 있음.** 즉 우리가 지우려는 낡은 가드보다 훨씬 먼저, 더 정확한 방식으로 이 문제가 해결돼 있었던 것.

### 4.4 결론

원래 가드가 막으려던 두 위험 다 이미 다른 메커니즘으로 해결됨 → **가드 자체가 중복/낡은 코드였다.** h1(HTTP/1.1) 클라이언트에는 같은 가드가 여전히 있는데, 그건 h1 파이프라이닝 특유의 제약 때문에 여전히 필요한 게 맞아서 그대로 둠(무관한 부분 건드리지 않음).

### 4.5 테스트 검증

- 새 테스트(`test/http2-pipelining-default.js`에 추가) — SSE GET 열어두고 POST 보내서, 상태코드/바디/**같은 세션 하나로 처리됐는지(`sessions.size === 1`)**까지 검증.
- stash A/B: fix 빼면 정확히 이슈와 같은 방식(5초 타임아웃)으로 재현, fix 있으면 통과.
- 회귀: `test/http2*.js`(73개) + `h2c-client.js`(12개) + `fetch/http2.js`(11개) = **96/96 통과**, eslint 클린.

---

## 5. 예상치 못한 발견 — 이미 열려있던 메인테이너 PR #5497

### 5.1 발견 경위

리뷰 마무리 단계에서 `git log --oneline --all -- lib/dispatcher/client-h2.js`로 이 파일의 전체 히스토리(모든 브랜치 포함)를 훑다가, **정확히 우리가 지운 것과 같은 9줄을 지우는 커밋(`840b81772`, "feat: enable concurrency on idempotent requests")이 이미 `origin`과 `upstream`의 `enable_concurrency`라는 브랜치에 존재**한다는 걸 발견함.

`gh pr list --repo nodejs/undici --head enable_concurrency`로 확인 → **PR #5497**, 작성자 `metcoder95`(Carlos Fuentes, undici 코어 메인테이너), 2026-07-03 생성. **(2026-07-20 갱신: 이후 2026-07-16 `metcoder95` 본인이 "Superseeded by #5538"이라며 직접 닫음 — 9.1절 참고.)**

### 5.2 PR #5497의 정확한 상태

- **본문**: "(Partially) Closes #5494" — 우리 이슈(#5524)가 아니라 **다른 이슈**를 위해 열린 것.
- **리뷰**: `mcollina`(또 다른 잘 알려진 코어 메인테이너)가 `CHANGES_REQUESTED`로 "테스트 추가해줘"라고 요청 (2026-07-07). 이후 추가 커밋 없이 **6일째 정체.**
- **이슈 #5494** ("HTTP2 POST requests over fetch() to the same origin can't run concurrently")를 확인해보니, **우리가 겪은 것과 정확히 같은 근본 원인**을 다른 증상(데드락이 아니라 "느려짐")으로 보고한 것이었음. #5494는 #5524보다 먼저 등록됐고, #5524 등록자는 #5494/#5497의 존재를 몰랐던 것으로 보임(자기가 처음부터 직접 원인 분석을 다시 함).

### 5.3 이 발견을 어떻게 다룰지 — 옵션 검토

세 가지 옵션을 놓고 고민했다:

1. PR 안 올리고 #5497에 테스트를 코멘트로 제안만 함 — 메인테이너의 막힌 지점을 직접 풀어주는 가장 "도움되는" 액션이지만, 내 이름의 머지 PR로는 안 남음.
2. **내 PR을 그대로 올리되, 본문에 #5497의 존재와 관계를 투명하게 설명하고 메인테이너에게 선택권을 줌.**
3. 아무것도 안 하고 #5497이 머지되길 기다림 — 가장 안전하지만 내가 찾은 것(연결점, 안전성 검증, 테스트)을 아예 안 씀.

**2번을 선택했다.** 이유: 실제로 보탤 게 있고(빠진 테스트, SSE 데드락 전용 커버리지), 경쟁이 아니라 투명하게 상황을 설명하고 메인테이너 판단에 맡기는 방식이 적절하다고 판단했다.

### 5.4 (PR 제출 후) "이 fix가 #5494도 실제로 고치는지" 직접 검증

PR을 이미 올린 뒤, "#5524, #5494 이 둘 다 해결한 것과 원인이 비슷한 것 같은데, 이 둘 다 처리해버리고 싶다"는 생각이 들었다. 코드상으로는 같은 9줄을 지운 거라 이론적으로는 #5494(동시 POST가 직렬화되는 문제)도 같이 고쳐질 것 같았지만, **실제로 테스트한 건 #5524의 SSE 데드락 시나리오뿐**이었고, #5494가 말하는 "동시에 보낸 요청들이 진짜 병렬로 나가는지"는 따로 검증한 적이 없었다. 이걸 직접 검증했다.

**1차 검증 시도 (실패)** — `Promise.all`로 6개 POST를 동시에 쏘는 방식으로 짜봤는데, **원본(수정 전) 코드에서도 멀티플렉싱되는 것처럼 나옴.** 원인: 6개 요청이 전부 같은 동기 루프 안에서 발사되다 보니, 어느 것도 `kRunning`이 채 오르기 전에 다 같이 dispatch 판정을 통과해버려서, 가드가 있는지 없는지와 무관하게 우연히 다 통과한 것 — 실제 버그 상황(하나가 이미 진행 중일 때 다음 게 들어오는 것)을 제대로 재현 못 한 잘못된 검증 스크립트였음.

**2차 검증 (정확한 방식)** — 요청 ①을 먼저 쏘고, **서버가 그 헤더를 실제로 받았다는 걸 확인한 뒤에야** 요청 ②를 쏘는 식으로, 이슈가 실제로 보고한 순서(하나가 이미 inflight일 때 다음 게 들어오는 것)를 정확히 재현:

- **원본(수정 전) 코드**: 요청 ②가 요청 ①이 끝나기 전까지 서버에 전혀 도달 안 함 (`false`) — #5494가 보고한 것과 정확히 일치.
- **수정 후 코드**: 요청 ②가 ①이 아직 pending인 상태에서 즉시 서버에 도달 (`true`).

→ **우리 fix 하나로 #5494도 실제로 해결됨을 확인.**

**검증 중 사고 하나**: 이 비교를 위해 `lib/dispatcher/client-h2.js`를 잠시 원본 코드로 바꿔치기해서 테스트하는 방식을 썼는데, 첫 시도에서 스크립트가 응답을 못 받은 요청 때문에 무한 대기하다가 **2분 타임아웃으로 셸 전체가 강제 종료되면서, 파일을 원상복구하는 명령까지 같이 죽어버림.** 그 결과 잠깐 동안 작업 디렉토리의 `client-h2.js`가 "원본(버그 있는) 상태"로 남아있었음. `git diff upstream/main`으로 즉시 감지해서 백업 파일로 복구했고, 이후 전체 h2 테스트 스위트(73/73)를 다시 돌려서 정상 상태임을 재확인함. **파일을 임시로 바꿔치기하는 검증을 할 때는 항상 "복구 명령이 실패해도 감지할 수 있는 방법"(예: 복구 직후 diff 확인)을 세트로 준비해야 한다**는 교훈.

**"3개를 처리한 거 아니냐"는 자문**: "PR #5538에 두 이슈를 명시해서 머지되면, 3개를 처리한 게 되는 거 아닌가?"라는 생각이 들어서 정확히 따져봤다:

- 내가 올린 PR: **1개** (#5538)
- 그 PR이 머지되면 닫히는 이슈: **2개** (#5524, #5494)
- 메인테이너의 PR #5497: 내 게 머지되면 중복이 되어 **메인테이너가 닫을 가능성이 높지만, 이건 "내가 처리한 것"으로 셀 수 없음** — 남의 PR이 자연스럽게 정리되는 것이지 내 성과가 아님

**결론: "PR 1개로 이슈 2개를 고쳤다"가 정확한 표현.** "3개를 처리했다"는 과장이라고 판단하고 정정함 — [[feedback_honest_pr_checklists]]와 같은, 부풀리지 않는 원칙을 여기서도 적용.

**#5494 전용 회귀 테스트 추가**: 검증 스크립트의 정확한 로직(요청 ①을 먼저 열고, 서버가 실제로 그 스트림을 받았다는 걸 확인한 뒤 요청 ②를 쏘고, ②도 서버에 도달하는지 확인 — 도달 안 하면 `AbortSignal.timeout(5000)`으로 5초 뒤 자동 실패)을 정식 테스트로 옮겨서 `test/http2-pipelining-default.js`에 추가. stash 없이(이미 커밋된 상태라) 원본 파일을 백업-교체하는 방식으로 A/B 검증 — 수정 전엔 정확히 5014ms에 타임아웃으로 실패, 수정 후엔 통과.

**PR 본문 갱신**: `Fixes #5524`만 있던 것에 `Fixes #5494`를 추가하고, "Relationship to #5497" 섹션의 "이 PR이 #5497에 없는 것"을 "테스트 추가함(빠져있던 거) + 이제 #5494 시나리오 자체도 전용 커버리지 있음"으로 갱신. "Bug Fixes" 섹션에도 #5494 항목 추가.

---

## 6. PR 본문 작성

### 6.1 구조

undici 자체 PR 템플릿(This relates to / Rationale / Changes / Bug Fixes / Status)을 그대로 쓰되, **"Relationship to #5497"이라는 섹션을 새로 추가**함:

- `#5497`이 같은 코드를 같은 이유로 지웠다는 것, 그 이슈(#5494)는 "느려진다"는 성능 프레이밍이고 우리 건 "영원히 멈춘다"는 더 심각한 데드락이라는 것
- **#5497을 못 찾고 독립적으로 고쳤다는 걸 그대로 밝힘** (숨기거나 나중에 들키는 것보다 먼저 밝히는 게 맞다고 판단)
- 우리 PR이 추가로 제공하는 것 두 가지를 명확히 나열: ① mcollina가 요청한 테스트(#5497엔 아직 없음), ② SSE 데드락 전용 회귀 테스트
- **"메인테이너가 #5497로 계속 가고 싶으면 이 PR은 닫아도 된다, 어느 쪽이 머지되든 상관없다"**로 마무리 — 경쟁하려는 의도가 아님을 명시

### 6.2 "This relates to..." 섹션에도 미리 안내

`Fixes #5524` 바로 아래에 "이거 #5497이랑 근본 원인 같으니 리뷰하기 전에 아래 'Relationship to #5497' 섹션부터 봐달라"는 안내를 넣음 — 리뷰어가 코드만 보고 "어? 이거 다른 PR이랑 똑같은데?"라고 당황하기 전에 먼저 짚어주려는 의도.

### 6.3 Status 체크리스트

"Documented"는 정직하게 비워둠(내부 버그 수정, 공개 문서 변경 없음) — [[feedback_honest_pr_checklists]]와 동일한 원칙.

### 6.4 AI 언급 여부

확인 → **"언급 안 함"** (지금까지와 동일). 커밋 메시지 + PR 본문 grep으로 재확인, 아무것도 안 걸림.

---

## 7. 기여 프로세스

### 7.1 브랜치

```bash
git checkout -b fix-issue-5524-h2-sse-post-hang   # main 기준, PR #5535 작업과 별개 브랜치
```

### 7.2 최종 커밋 메시지

```
fix(h2): allow stream-bodied requests to multiplex on a busy session

busy() deferred any request with a stream/async-iterable/FormData body
until every other in-flight request on the h2 session completed. For a
long-lived stream on the session (e.g. an open text/event-stream GET),
this parks bodied requests indefinitely -- the client also stops
reporting kBusy while an h2 context is attached, so the pool never
opens a second connection either.

The two hazards the guard's comments cited are already handled more
precisely elsewhere: a stream erroring mid-request now only aborts
that one stream (session stays usable, see the abort() closure), and
canRetryRequestAfterGoAway() already excludes stream/iterable/FormData
bodies from the requests it resurrects after a GOAWAY, erroring them
instead of unsafely replaying a partially-consumed body. The blanket
pre-dispatch guard was redundant given those.

Adds a regression test reproducing the exact deadlock: a POST issued
while an SSE GET is held open on the same session now resolves
immediately on the shared session instead of hanging. Fixes #5524.
```

---

## 8. 메인테이너 리뷰 & CI 실패 로컬 재현/수정 (2026-07-10)

### 8.1 실제 리뷰 도착

PR을 올린 다음 날, 두 명의 리뷰가 붙음:

- **`mcollina`(코어 메인테이너)**: `APPROVED` — "lgtm"
- **`metcoder95`(PR #5497의 작성자, 5번 섹션에서 다룬 그 사람)**: `COMMENTED` — "tests seems unhappy"

`mcollina`가 승인까지 했는데 CI가 안 좋아 보인다는 지적이 같이 붙은 상황. `gh pr checks`로 실제 CI를 확인하니 **macOS/Windows × Node 24/25/26 조합 6개가 전부 fail**(Ubuntu는 전부 pass). 실패 로그를 열어보니:

```
Test "fetch POST multiplexes while an SSE stream is open on the same h2 session (#5524)"
generated asynchronous activity after the test ended. This activity created the error
"Error: read ECONNRESET" and would have caused the test to fail, but instead triggered
an uncaughtException event.
```

우리가 4번 섹션에서 짠 `#5524` 회귀 테스트 자체가, SSE fetch의 응답 바디를 한 번도 안 읽은 채로 `sse.abort()` 직후 바로 `dispatcher.close()`를 불러서, abort가 채 정리되기 전에 dispatcher가 닫혀버리는 레이스로 추정.

### 8.2 1차 수정 (틀린 진단, 나중에 뒤집힘)

당시 진단: "abort가 정리되는 걸 안 기다려서 그렇다." 수정: SSE fetch의 Promise와 Response를 변수로 잡아뒀다가, teardown에서 그 Promise를 `await`하고 응답 body를 명시적으로 `cancel()`한 다음에야 `dispatcher.close()`를 부르도록 순서 변경. 로컬(Node 22, macOS)에서 20번 반복 실행해서 전부 통과 확인 → 커밋 & 푸시 & PR 본문에 코멘트까지 남김.

**근데 이건 나중에 8.5절에서 확인하듯 실제로는 문제를 안 고친 수정이었음** — 로컬 Node 22에서는 애초에 이 레이스 자체가 안 터져서, "고쳤다"는 확인 자체가 불가능한 상태에서 확인했다고 착각한 것.

### 8.3 CI가 자동으로 안 도는 이유 — `action_required`

새 커밋을 push한 뒤 `gh pr checks`가 "no checks reported"만 리턴함. `gh api repos/nodejs/undici/actions/runs`로 직접 워크플로우 실행 상태를 조회하니 **`conclusion: action_required`** — GitHub Actions의 보안 정책상, **fork 기반 외부 기여자의 PR에 새 커밋이 올라가면 메인테이너가 수동으로 승인해야 워크플로우가 실제로 시작**되는 방식임을 확인. 즉 우리 쪽에서 뭘 더 하든 CI 자체가 안 도는 상태.

여기서 결정을 내렸다: 오늘 안에 메인테이너 승인이 날 것 같지 않고(시차 문제 — 한국과 미국), 누구한테 기대고 있어봐야 발전이 없으니 직접 CI를 로컬에서 돌려보기로 했다. 메인테이너의 승인을 기다리는 대신, CI가 실제로 뭘 하는지 워크플로우 파일(`nodejs.yml`)을 직접 읽어서 로컬로 최대한 똑같이 재현하기로 했다.

### 8.4 로컬에서 CI 재현 — Node 24 설치부터

`.github/workflows/nodejs.yml`을 읽어서 CI가 정확히 `npm ci` → PEM 생성 → `test:unit`, `test:node-test`, `test:fetch`, ... 순서로 여러 `npm run test:*` 스크립트를 돈다는 걸 확인. 로컬엔 Node 20/22만 있고 **CI가 실패한 Node 24/25/26이 없어서**, `nvm install 24`로 직접 설치 후 `npm ci`로 네이티브 의존성까지 다시 설치.

**1차 실행 결과 (여기서 처음으로 로컬 재현 성공)**: `npm run test:unit`을 Node 24로 돌리니 **`test/http2-pipelining-default.js`에서 정확히 CI와 같은 에러로 실패**. Node 22에서는 단 한 번도 재현 안 됐던 게, Node 24에서는 **`--test-name-pattern`으로 그 테스트 하나만 골라 돌려도 5번 연속 100% 재현**됨. Node 버전별로 소켓/이벤트 루프 타이밍이 미묘하게 달라서, 낮은 버전에서는 우연히 안 걸렸던 레이스가 24부터는 항상 걸리는 것으로 추정.

### 8.5 디버깅 — 틀린 가설들을 하나씩 버림

**가설 1 (틀림)**: "서버 쪽 session에 error 리스너가 없어서 그런가?" → `session.on('error', () => {})`를 추가해봤지만 5/5 여전히 실패. 기각.

**가설 2 (틀림)**: "그냥 시간이 더 필요한 레이스인가?" → `after()` 훅 안에 `dispatcher.close()` 다음 50ms, 그다음 2000ms 그레이스 딜레이를 넣어봤지만 **둘 다 전혀 효과 없음**. 진단을 위해 `process.on('uncaughtException')`을 임시로 걸어서 타임스탬프를 찍어보니, **에러가 `dispatcher.close()`가 resolve된 지 단 몇 ms 안에(2초 딜레이가 시작되기도 전에) 이미 발생**한다는 걸 확인 — "더 기다리면 되는" 문제가 절대 아니었음.

**가설 3 (틀림, 시도 실패)**: "그럼 process 레벨에서 이 특정 에러만 잡아서 무시하면?" → 임시 `process.on('uncaughtException', ...)` 핸들러를 걸어서 `ECONNRESET`만 스루하게 해봤지만, **`node:test` 자체가 이미 같은 이벤트에 자기만의 리스너를 걸어두고 있어서, 내가 리스너를 하나 더 달아도 node:test의 리스너는 별개로 계속 실행됨** — `uncaughtException`은 여러 리스너가 동시에 다 호출되는 구조라, 하나가 "무시"한다고 다른 리스너의 판정을 막을 수 없다는 걸 실전으로 확인.

**가설 4 (틀림)**: "`sse.abort()`를 아예 빼면?" → 여전히 실패. abort 자체가 원인이 아니었음.

### 8.6 진짜 원인과 진짜 수정

`dispatcher.close()`가 resolve되는 시점에 즉시 에러가 난다는 건, **클라이언트가 "아직 열려있는" SSE 스트림을 강제로 종료시키는 행위 자체가, 소켓을 "정상 종료(clean end)"가 아니라 "리셋(reset)"으로 끊어버린다**는 뜻으로 해석. 그럼 반대로, **서버 쪽에서 그 스트림을 먼저 정상적으로 끝내주면?**

```js
server.on('stream', (stream, headers) => {
  if (headers[':path'] === '/events') {
    eventsStream = stream   // 서버 쪽 스트림 객체를 잡아둠
    ...
  }
})
...
after(async () => {
  eventsStream?.end()   // ← 클라이언트가 손대기 전에 서버가 먼저 정상 종료
  sse.abort()
  await ssePromise
  await sseResponse?.body?.cancel().catch(() => {})
  await dispatcher.close()
})
```

**Node 24로 30번 연속 실행 → 30/30 전부 통과.** 서버가 먼저 깔끔하게 끝내주니, 클라이언트가 그 뒤에 abort/cancel/close를 해도 "이미 끝난 스트림"을 다루는 거라 리셋이 안 생기는 것으로 확인. `test/http2*.js` 전체(74개), CI가 실패했던 `test:unit`(1415개) 전부 Node 24에서 0 fail로 재확인.

### 8.7 새 커밋 (amend 아님) + PR 코멘트

이전(틀렸던) 수정 커밋을 고치는 대신, **"이전 커밋의 진단이 틀렸었다"는 걸 커밋 메시지에 정직하게 남기고 새 커밋을 쌓음** (`5081722a2`) — 이 세션 내내 지켜온 "새 커밋으로, amend 하지 않기" 원칙 그대로.

`metcoder95`의 "tests seems unhappy" 코멘트에 짧게 답함:

> Found the real cause -- it wasn't a timing/wait issue (a 2s grace period didn't help either). The client abruptly aborting the still-open SSE stream before closing the dispatcher tears the socket down by reset instead of a clean end... Reproduced 100% locally under Node 24 (not on 22, which is why I hadn't caught it before) and verified 30/30 after the fix, plus a clean `test:unit` run.

`#5535`쪽 `mcollina`의 `.every()` 지적에는 별도 코멘트 안 남김(커밋 자체가 정확히 요청받은 대로 고친 단순 수정이라 코드로 답하는 걸로 충분하다고 판단) — 반면 이쪽은 실제 조사와 반전이 있었던 케이스라 코멘트로 과정을 설명하는 게 맞다고 판단한 것. 같은 "코멘트 남길지" 질문에도 상황에 따라 다르게 판단한 사례.

---

## 9. 결과 및 회고

### 9.1 현재 상태 (2026-07-20 기준)

- PR #5538 **Merged** (2026-07-13T23:17:07Z, `mcollina` 본인이 직접 머지, merge commit `a3607810040467f3d6863e24ba4b13c1765ef0b6`)
- `mcollina` **APPROVED**("lgtm"), `metcoder95`의 CI 우려 코멘트에는 근본 원인 설명으로 답변 완료 (8.7절)
- `Fixes #5524`, `Fixes #5494` 둘 다 명시했던 대로, PR 1개 머지로 이슈 2개가 동시에 닫힘
- **경쟁 PR #5497, 2026-07-16에 작성자 `metcoder95` 본인이 직접 Closed 처리** — 코멘트: "Superseeded by #5538." 예상했던 시나리오("#5497이 먼저 머지되면 우리 걸 정중히 닫는다")와 반대 순서로 흘러갔지만, 결과적으로는 가장 깔끔한 결말 — 경쟁하던 두 PR 중 하나로 완전히 정리됐고, 그 하나가 우리 것이었다.

### 9.2 지금까지 배운 점

- **"내가 새로 발견한 버그"라고 생각했던 것이, 사실은 이미 다른 이름으로 보고되고 다른 사람이 이미 고치고 있던 문제일 수 있다** — PR을 올리기 전에 `git log --all`로 관련 파일의 전체 히스토리(모든 브랜치 포함)를 훑어보는 습관이, 이번엔 실제로 "경쟁 PR을 모르고 올릴 뻔한" 상황을 미리 잡아냄.
- **"같은 코드 수정"이라도 "무엇이 부족해서 막혀있는지"까지 파악하면, 경쟁이 아니라 보완이 될 수 있다** — #5497은 코드는 맞았는데 테스트가 없어서 막혀있었고, 우리는 정확히 그 테스트를 갖고 있었음. 이 차이를 PR 본문에서 명확히 하는 게 핵심이었음.
- **원래 가드/제약이 "왜 있었는지"를 코드베이스 git 히스토리까지 거슬러 올라가서 확인하는 습관**이 이번에 또 유효했음 — `canRetryRequestAfterGoAway()`가 언제 도입됐는지 확인해서, "이 가드가 진짜로 낡은 것"이라는 확신을 이슈/PR 설명에만 의존하지 않고 직접 검증함.
- **투명성이 곧 전략이다** — 경쟁 PR의 존재를 숨기거나 모른 척하지 않고 먼저 밝히는 게, 장기적으로 메인테이너 신뢰를 얻는 데 더 유리하다고 판단하고 실행에 옮긴 첫 사례.
- **"로컬에서 안 터지니 괜찮다"는 확인은, 문제가 재현되는 정확한 환경(이번엔 Node 24)을 맞추지 않으면 아무 의미가 없다** — 1차 수정을 "20번 반복해서 통과했다"고 확인했었는데, 그건 애초에 Node 22에서는 이 레이스 자체가 안 터졌기 때문이었음. **재현이 안 되는 환경에서의 "통과"는 검증이 아니다**는 걸 뼈아프게 체득 — 실제 CI 실패 환경(Node 버전, OS)을 최대한 맞춰야 진짜 검증이 됨.
- **"기다리면 될까?"라는 가설은 실제로 시간을 재보기 전까진 그냥 추측일 뿐이다** — 2초 그레이스 딜레이를 넣어도 에러가 몇 ms 안에 바로 터진다는 걸 타임스탬프로 직접 찍어서 확인하고 나서야, "타이밍 문제"라는 가설 자체를 완전히 폐기할 수 있었음.
- **`process.on('uncaughtException')`은 "이 이벤트를 내가 처리했으니 다른 리스너는 무시해"가 안 되는 구조**라는 걸 실전에서 배움 — Node의 EventEmitter는 등록된 모든 리스너를 다 호출하고, `node:test`가 이미 자기 리스너를 걸어둔 상태라서, 내가 추가로 리스너를 달아 "무시"해도 `node:test`의 판정 자체를 못 막음. 이 사실을 몰랐으면 안 되는 방향으로 계속 시간을 썼을 것.
- **메인테이너 승인을 무작정 기다리는 대신, CI가 실제로 뭘 하는지 워크플로우 파일을 직접 읽고 로컬로 재현하는 것**이 가장 실질적으로 발전에 도움 된 판단이었다 — "누구한테 기대면 발전이 없다"는 게 정확히 맞아떨어진 사례.
