# [OSS 기여] nodejs/node - QUIC 단방향 스트림의 지연된 reader에서 수신 데이터가 사라지는 문제 (이슈 #66347)

> 현재 상태 (2026-11-03 확인): **MERGED** — [PR #66407](https://github.com/nodejs/node/pull/66407)은 한국시간 2026-11-02 21:22:48에 main에 반영됐다. 최종 커밋은 [`1389475f66`](https://github.com/nodejs/node/commit/1389475f66a722eab83547797491c3e879de3f7f)이다. 아래 OPEN·CLOSED 표시와 로컬 검증 결과는 각 시점의 기록이며, 최종 결과는 문서 끝에 추가했다.

> 현재 상태 (2026-10-24): **OPEN — 작성자가 재오픈했고, MikeMcC399가 다른 PR을 열어도 된다고 확인했다.** 커밋 메시지 정리와 리베이스를 로컬에서 진행했다. 아래 10월 20일 상태와 초기 테스트 결과는 과거 기록이며, 이번 검증 결과와 혼동하지 않는다. 최신 경과는 문서 끝에 이어 적는다.

> 현재 상태 (2026-10-20 확인): **⚪ CLOSED — 미머지 · 재오픈 의견 대기** — [PR #66407](https://github.com/nodejs/node/pull/66407). 최신 경과는 문서 끝의 상태 점검 절에 추가했다. 아래 기존 설명과 검증 결과는 각 작성 시점의 기록이다.

상태: 🟡 PR OPEN, 리뷰 대기
저장소: https://github.com/nodejs/node
이슈: https://github.com/nodejs/node/issues/66347
PR: https://github.com/nodejs/node/pull/66407
브랜치: `fix/quic-unidirectional-reader`
로컬 커밋: `2526c29286` (`quic: retain readers for readable streams`)

## 1. 이슈 요약

QUIC의 원격 단방향(unidirectional) 스트림에서 서버가 데이터를 보낸 뒤 FIN을 즉시 전송하면, 클라이언트가 스트림의 async iterator를 먼저 만들어 두고 실제 첫 `next()` 호출을 나중에 할 때 이미 받은 데이터가 사라질 수 있다.

문제가 되는 사용 순서는 다음과 같다.

```js
clientSession.onstream = async (stream) => {
  const iterator = stream[Symbol.asyncIterator]();

  // 다른 초기화 작업을 기다리는 동안 서버가 데이터와 FIN을 보냄
  await someOtherInitialization();

  const first = await iterator.next();
}
```

`QuicStream`의 async generator는 `[Symbol.asyncIterator]()`를 호출하는 순간 본문을 실행하지 않는다. 실제 native reader를 만드는 코드는 첫 `iterator.next()`가 실행될 때까지 미뤄져 있었다. 그 사이 peer가 데이터와 FIN을 모두 보내고 native stream handle이 닫히면, JS가 나중에 reader를 만들 시점에는 수신 큐가 이미 정리되어 있다.

이슈 제보자는 workaround로 iterator를 만든 직후 `next()`를 호출하고 그 Promise를 나중에 기다리는 방법을 사용하고 있었다. 이 방법은 사용자가 reader를 일찍 당겨야 한다는 부담을 남기므로, 스트림 객체가 생성될 때 readable side의 reader 수명을 확보하는 방향을 검토했다.

## 2. QUIC 스트림 방향과 이번 경로

QUIC 스트림은 bidirectional과 unidirectional로 나뉜다.

- bidirectional stream: 양쪽이 읽고 쓸 수 있다.
- unidirectional stream: 생성한 쪽은 쓰기만 하고, 반대편은 읽기만 한다.

이번 재현에서는 서버가 단방향 스트림을 만들고 body를 전송한다. 따라서 서버 쪽 스트림은 writable-only이고, 클라이언트가 받는 스트림은 readable-only인 remote unidirectional stream이다.

Node의 `QuicStream` JS 객체 생성자는 다음 정보를 받는다.

- native `handle`
- 소속 `QuicSession`
- 방향(`bidi` 또는 `uni`)
- 현재 프로세스가 만든 스트림인지 여부(`isLocal`)
- 읽기 종료 시 truncation 처리 방식

`isLocal`은 스트림의 readable 여부를 직접 뜻하지 않는다. 방향과 조합해서 해석해야 한다.

| 스트림 | `isLocal` | 읽을 수 있는가 | reader 필요 여부 |
| --- | ---: | ---: | --- |
| 로컬 unidirectional | `true` | 아니오 | 만들면 안 됨 |
| 원격 unidirectional | `false` | 예 | 생성 시 확보해야 함 |
| 로컬 bidirectional | `true` | 예 | 생성 시 확보해야 함 |
| 원격 bidirectional | `false` | 예 | 생성 시 확보해야 함 |

그래서 수정 조건은 local unidirectional만 제외하고 readable stream 전체에 적용했다.

## 3. 기존 JS 동작

기존 `QuicStream` constructor는 native handle과 JS 객체를 연결한 뒤 상태를 초기화했다. reader는 constructor에서 만들지 않고 async iterator 본문에서 다음처럼 지연 생성했다.

```js
async *[SymbolAsyncIterator]() {
  const inner = this.#inner;

  if (inner.iteratorLocked) {
    throw new ERR_INVALID_STATE('Stream is already being read');
  }
  inner.iteratorLocked = true;

  inner.reader ??= this.#handle?.getReader();
  if (inner.reader) {
    yield* createBlobReaderIterable(inner.reader);
  }
}
```

여기서 중요한 점은 async generator의 지연 실행이다.

```js
const iterator = stream[Symbol.asyncIterator]();
```

이 코드는 iterator 객체만 만든다. generator 본문은 아직 실행되지 않으므로 `getReader()`도 호출되지 않는다. 아래 코드가 실행될 때 비로소 첫 pull과 reader 생성이 시작된다.

```js
await iterator.next();
```

따라서 `iterator`를 먼저 저장해 둔 뒤 네트워크 이벤트와 close 처리가 진행되면, 사용자가 기대하는 “iterator를 만들었으니 데이터가 보존된다”는 보장이 성립하지 않았다.

## 4. native 수명과 데이터가 사라지는 지점

`src/quic/streams.cc`의 native `Stream`은 수신 데이터를 두 단계로 보관한다.

1. 작은 수신 조각은 `recv_accumulator_`에 모은다.
2. flush 시 `inbound_`라는 `DataQueue`에 넣는다.

`FinishReadable()` 또는 `TruncateReadable()`은 `CapReadable()`을 거치며 먼저 accumulator를 `inbound_`로 flush한 뒤 readable side를 cap하고 reader를 깨운다.

문제는 close 때의 정리 순서다.

```cpp
void Stream::Destroy(QuicError error) {
  TruncateReadable();
  recv_accumulator_.reset();
  inbound_->removeBackpressureListener(this);
  inbound_.reset();
  reader_.reset();
  EmitClose(error);
}
```

`EmitClose()`가 JS의 `onStreamClose()`와 `[kFinishClose]`를 실행하기 전에 native `inbound_`와 native reader가 정리된다. JS가 아직 reader를 만들지 않았다면 `DataQueue`를 붙잡고 있는 객체가 없기 때문에, flush되어 있던 데이터도 함께 수명을 잃는다.

반대로 JS 쪽 `inner.reader`가 이미 `Blob::Reader`를 보유하고 있으면 reader가 `inbound_`의 queue를 계속 참조할 수 있다. native의 `reader_` 포인터를 reset하는 것과 JS가 보유한 reader 객체 자체가 즉시 사라지는 것은 다르다. 이 차이가 이번 수정의 핵심이다.

## 5. 수정 내용

`QuicStream` constructor에서 readable stream인지 확인한 뒤 native reader를 즉시 생성하도록 했다.

```js
const inner = this.#inner;

// Keep a reader alive for readable streams before the native handle can
// be destroyed. A peer may send data and FIN before the consumer asks
// for its async iterator.
if (!isLocal || direction === kStreamDirectionBidirectional) {
  inner.reader = handle.getReader();
}
```

기존 async iterator의 지연 생성 코드는 그대로 남겨 두었다.

```js
inner.reader ??= this.#handle?.getReader();
```

이 fallback은 다음과 같은 기존 경로를 안전하게 유지한다.

- constructor에서 reader를 만들 수 없는 특수 상태
- 이미 handle이 사라진 stream
- 읽을 수 없는 local unidirectional stream

실제로 readable stream에서는 constructor에서 reader가 먼저 생기므로 fallback이 중복 호출되지 않는다. local unidirectional stream은 native `getReader()`가 `undefined`를 반환하는 기존 규칙을 그대로 따른다.

## 6. 왜 native `NotifyPull()`만 추가하지 않았는가

이슈 댓글에는 `Stream::get_reader()`가 reader를 만든 직후 `recv_accumulator_`에 데이터가 있으면 `NotifyPull()`을 호출하자는 방향도 제안돼 있었다.

이 변경은 reader가 이미 존재하는 상태에서 데이터가 도착했지만 wakeup 통지가 빠지는 경우에는 유용하다. 그러나 이번 문제의 핵심은 reader가 만들어지기 전에 native `Destroy()`가 실행되어 `inbound_` 자체가 정리되는 것이다.

즉 다음 두 문제는 다르다.

- reader는 존재하지만 데이터 도착 통지가 없음: `NotifyPull()` 문제
- reader가 없고 close 정리 과정에서 queue도 소멸함: reader 수명 문제

이번 회귀 테스트는 두 번째 경우를 대상으로 하므로, reader를 스트림 생성 시점에 확보하는 JS 수정이 직접적인 해결책이다. native `NotifyPull()`을 별도로 추가하지 않아도 async iterator의 첫 pull은 queue에 이미 들어온 데이터를 동기적으로 확인할 수 있다.

## 7. 회귀 테스트

추가한 테스트 파일:

`test/parallel/test-quic-stream-uni-delayed-reader.mjs`

테스트 순서는 다음과 같다.

1. QUIC 서버를 시작한다.
2. 서버가 body를 가진 unidirectional stream을 만든다.
3. 클라이언트가 원격 stream을 받는다.
4. 클라이언트가 `stream[Symbol.asyncIterator]()`만 먼저 호출한다.
5. 첫 pull은 실행하지 않고 `await stream.closed`로 미룬다.
6. stream close 이후 `bytes(iterator)`로 첫 pull을 시작한다.
7. 서버가 보낸 bytes가 그대로 반환되는지 검사한다.

핵심 부분은 다음과 같다.

```js
clientSession.onstream = mustCall(async (stream) => {
  const iterator = stream[Symbol.asyncIterator]();

  await stream.closed;

  const received = await bytes(iterator);
  assert.deepStrictEqual(received, expected);
});
```

이 테스트는 구현 내부의 `inner.reader`를 검사하지 않고, 사용자가 받는 async iterator 결과를 검사한다. 따라서 reader를 eager allocation하는 방식이 바뀌더라도 데이터 보존이라는 외부 동작이 유지되면 테스트를 통과할 수 있다.

## 8. 수정 전후 검증

처음에는 기존 QUIC 비활성 바이너리에서 테스트가 스킵되는 것을 확인했다. 이 상태만으로는 회귀 여부를 판단할 수 없으므로 저장소의 bundled `ngtcp2`를 사용해 QUIC 활성화 빌드를 새로 만들었다.

```sh
./configure --experimental-quic
make -j2
```

### 수정 전

수정된 constructor 코드를 되돌린 뒤 증분 재빌드하고 같은 테스트를 실행했다.

```sh
out/Release/node \
  --experimental-quic \
  --experimental-stream-iter \
  --no-warnings \
  --test test/parallel/test-quic-stream-uni-delayed-reader.mjs
```

수정 전에는 첫 pull이 끝나지 않아 10초 timeout으로 종료됐다. 즉 테스트가 원래의 lazy reader 수명 문제를 실제로 잡고 있다는 것을 확인했다.

### 수정 후

수정 코드를 복원하고 다시 증분 빌드한 뒤 같은 명령을 실행했다.

결과:

```text
✔ test/parallel/test-quic-stream-uni-delayed-reader.mjs
ℹ tests 1
ℹ pass 1
ℹ fail 0
```

### 관련 기존 테스트

다음 네 개를 함께 실행했고 모두 통과했다.

```sh
out/Release/node \
  --experimental-quic \
  --experimental-stream-iter \
  --no-warnings \
  --test \
  test/parallel/test-quic-stream-uni-server-initiated.mjs \
  test/parallel/test-quic-stream-uni-delayed-reader.mjs \
  test/parallel/test-quic-stream-iteration-nonreadable.mjs \
  test/parallel/test-quic-stream-uni-basic.mjs
```

결과:

```text
ℹ tests 4
ℹ pass 4
ℹ fail 0
```

추가 검증:

- `make lint-js`: 통과
- `git diff --check`: 통과
- `node --check lib/internal/quic/quic.js`: 통과
- `node --check test/parallel/test-quic-stream-uni-delayed-reader.mjs`: 통과

## 9. 변경 범위와 비용

수정은 JS `QuicStream` constructor 한 곳과 회귀 테스트 한 파일로 제한했다. C++ queue 구조나 close 순서를 바꾸지 않았고, 공개 API도 추가하지 않았다.

reader를 생성 시점에 확보하므로 readable stream마다 reader 객체가 조금 일찍 만들어진다. 대신 사용자가 iterator를 즉시 pull하지 않아도 native close 과정에서 수신 queue의 수명이 끊기지 않는다. local unidirectional stream은 읽을 수 없으므로 reader를 만들지 않아 불필요한 객체 생성은 피했다.

bidirectional stream도 readable side를 가지므로 같은 close-before-first-pull 경로가 가능하다. remote unidirectional stream에만 좁게 적용하면 같은 수명 문제가 다른 readable stream에서 남을 수 있어, readable stream 전체를 동일한 방식으로 처리했다.

## 10. 현재 상태와 다음 단계

- [x] 이슈와 댓글 확인
- [x] native close 및 queue 정리 순서 추적
- [x] 수정 전 회귀 재현
- [x] 수정 후 회귀 테스트 통과
- [x] 관련 기존 테스트 통과
- [x] JS lint 및 diff 검사 통과
- [x] 사람의 최종 코드 검토
- [x] Node.js 저장소 규칙에 맞는 로컬 커밋 작성
- [x] 사용자가 직접 sign-off와 AI-assisted disclosure를 확인
- [x] 사용자 주도로 fork 브랜치 push 및 PR 생성
- [ ] Node.js QUIC 담당자 리뷰
- [ ] Jenkins 및 전체 CI 결과 확인

Node.js 저장소의 `AGENTS.md`는 에이전트가 직접 push하거나 PR을 생성하는 것을 금지하고 있다. 사용자가 직접 fork 브랜치를 push하고 PR #66407을 생성했다. PR은 현재 OPEN 상태이며 `@nodejs/quic` 팀 리뷰를 기다리고 있다. 초기 자동화 작업 중 commit-message lint만 진행 중이고, 나머지 기본 GitHub Actions 작업은 성공 또는 스킵 상태다. 첫 기여 PR이라 collaborator 승인과 Jenkins 시작이 필요하다는 안내도 함께 등록되어 있다.


## 2026-10-20 PR 상태 점검

- 현재 상태: **⚪ CLOSED — 미머지 · 재오픈 의견 대기** — [GitHub PR](https://github.com/nodejs/node/pull/66407)
- 확인한 head: `2526c29286eb7ac14b88ab1d9fc810729c2630a2`
- 종료 시각: `2026-09-30T08:32:02Z` (UTC). 머지 없이 종료.
- GitHub 집계 리뷰 상태: `REVIEW_REQUIRED`
- 현재 head의 체크·상태: FAILURE 2개, SKIPPED 8개, SUCCESS 32개.

실패로 보고된 검사:

- [lint-commit-message](https://github.com/nodejs/node/actions/runs/36687459287/job/109796484186)
- [test-quic](https://github.com/nodejs/node/actions/runs/36687459442/job/109802616280)

9월 30일 작성자가 첫 기여자의 동시 PR 가이드를 확인하고 직접 닫았다. 코드 수정이 거절돼 닫힌 것이 아니며, 현재도 재오픈되지 않았다.

10월 19일 martenrichter는 reader를 이 시점에 확보하는 수정이 문제를 해결할 것 같고, 다른 방식도 생각했지만 대안을 찾지 못했다고 [의견](https://github.com/nodejs/node/pull/66407#issuecomment-6260819578)을 남겼다.
작성자는 닫아 둔 이유를 설명하고 [재오픈해도 되는지 물었다](https://github.com/nodejs/node/pull/66407#issuecomment-6261507571).
그 뒤 martenrichter는 [다른 참여자들의 의견을 기다려 보자며, 그들이 다시 열 수 있을 것 같다고 답했다](https://github.com/nodejs/node/pull/66407#issuecomment-6262141278).

긍정적인 기술 의견과 재오픈 허락, 공식 승인, 머지는 서로 다르다. 아직 재오픈 확정으로 읽을 답변은 아니다. 실제 PR 상태에 맞춰 문서는 `close`로 옮기되, 재개 논의 중이라는 점을 남긴다. 재오픈되면 같은 문서를 `open`으로 되돌리면 된다.

현재 head에는 `lint-commit-message`와 `test-quic` 실패가 남아 있다. 로그에서 전자는 본문 4번째 줄이 269자로 72자 제한을 넘은 것으로 확인했다.
후자는 `parallel/test-quic-stream-truncated-reads-destroy.mjs:56`에서 기대값 0과 실제값 1000이 달라 실패했다. 이는 테스트 assertion 실패이며, 빌드 환경 문제로 단정하거나 긍정적인 댓글만으로 무시하면 안 된다. 수정과의 인과관계는 이번 문서 순회에서 재현·분석하지 않았으므로 별도 확인이 필요하다.
초기 문서의 검사 진행 중이라는 설명을 현재 상태로 읽지 않도록 결과를 추가한다.

이 절은 GitHub의 PR 상태·댓글·리뷰·체크 결과를 조회한 기록이다. 체크 수에는 자동화 상태가 포함될 수 있으며, 전체 테스트 실행 수가 아니다. 로컬 테스트 재실행, PR 재오픈·종료, 리뷰 스레드 처리나 공개 댓글 작성은 하지 않았다.

## 2026-10-24 재오픈과 리뷰 대응

### 재오픈까지의 흐름

martenrichter는 이 PR을 우연히 본 사람이 아니라 원래 이슈 #66347의 작성자다.
WebTransport 개발 중 서버가 단방향 스트림에 데이터를 보내고 FIN을 보낸 뒤,
클라이언트가 뒤늦게 iterator를 읽으면 데이터가 사라지는 문제를 신고했다.
10월 19일에는 jasnell과 pimterry를 태그하며 reader를 미리 만드는 수정이 해결책으로 보인다고 말했다.
작성자가 첫 기여자 가이드 때문에 닫았다고 설명하자, 처음에는 다른 사람들의 의견을 기다리자고 답했다.

10월 24일에는 자신의 WebTransport 브랜치에 변경을 cherry-pick했다며 재오픈을 권했다.
작성자는 한국 시각 15시 22분에 직접 PR을 다시 열고, 다음 댓글을 남겼다.

> Thanks for the review! I've reopened the PR.

관련 기록:

- [원래 이슈 #66347](https://github.com/nodejs/node/issues/66347)
- [martenrichter의 재오픈 권유](https://github.com/nodejs/node/pull/66407#issuecomment-6380396728)
- [작성자의 재오픈 답변](https://github.com/nodejs/node/pull/66407#issuecomment-6380864964)

### 권한과 승인 상태를 잘못 해석했던 부분

martenrichter는 당시 공식 Collaborators 명단에는 없었다. 그의 LGTM은 관련 개발자이자
신고자의 기술적 지지이며, 공식 Collaborator의 머지 승인과 같지는 않다.
하지만 직함이 없다는 이유로 구현을 검토하고 자기 브랜치에 적용한 사실의 가치까지 낮춰서는 안 된다.

이 과정에서 #64627의 종합 상태가 `REVIEW_REQUIRED`라는 이유만으로 승인 기록이 없다고 판단했다.
그 판단은 틀렸다. 실제 리뷰 목록에는 다음 승인이 있었다.

- 2026-07-25: avivkeller, `APPROVED`
- 2026-08-02: jasnell, `APPROVED`

PR의 현재 집계 상태, 과거에 받은 승인, 최신 커밋의 머지 요건 충족 여부는 서로 다른 정보다.
집계 상태 하나로 과거 승인 자체가 없었다고 결론 내리면 안 된다.
또 첫 기여자 가이드에서 말하는 기준은 첫 PR의 **승인**이지 **머지 완료**가 아니었다.

MikeMcC399는 과거 댓글의 역할 표시가 `CONTRIBUTOR`였지만,
이번에 확인한 공식 README의 Collaborators 명단에는 올라 있었다.
댓글 배지만으로 현재 권한을 확정하지 않고 공식 명단과 실제 리뷰를 함께 확인해야 한다.

### MikeMcC399의 새 안내

[10월 24일 댓글](https://github.com/nodejs/node/pull/66407#issuecomment-6381564363)에서
MikeMcC399는 #64627이 승인됐으므로 다른 PR을 열어도 된다고 명시했다.
아직 머지 완료된 PR은 없으므로 CI 실행은 매번 수동 승인이 필요하다는 설명도 덧붙였다.

새 CI를 요청하기 전에 요구한 작업은 두 가지였다.

1. 커밋 메시지의 72자 줄 길이 제한 위반을 고친다.
2. 최신 upstream 변경이 포함되도록 PR을 리베이스한다.

이 댓글은 재오픈을 질책하거나 구현을 거절한 내용이 아니다.
재오픈 가능 여부를 확인하고, CI를 다시 돌리기 전에 제출 상태를 정리하라는 안내다.

### 커밋 메시지와 리베이스

작업 전 워킹트리는 깨끗했고, 로컬 QUIC 브랜치와 포크의 head는 모두
`2526c29286eb7ac14b88ab1d9fc810729c2630a2`였다.
원래 브랜치는 `backup/quic-before-rebase-20261024`로 보존했다.

커밋 본문이 269자 한 줄로 작성돼 있던 것을 줄바꿈했다.
문장의 의미와 제목은 바꾸지 않았으며, 기존 `Assisted-by: Codex`와 사용자 `Signed-off-by`도 유지했다.
자연스러운 문장을 쓰는 것과 저장소가 요구하는 공개 표기를 제거하는 것은 다른 문제다.

리베이스 기준은 upstream main의 `763524d434aa6b40fecbaf088b0fa788b3aa0375`다.
충돌 없이 끝났고, 메시지 정리 후 로컬 커밋은 `f057aabf68e7fe89a416819909d274f1c3c601df`가 됐다.
`git range-diff`로 기존 패치와 비교했으며 코드 변경은 동일하고 커밋 본문 줄바꿈만 달라졌다.
PR의 변경 파일은 여전히 구현 한 파일과 신규 테스트 한 파일, 총 53줄 추가다.

CI와 같은 `core-validate-commit@6.0.0 --no-validate-metadata --tap -` 검사에서
8개 검사가 통과했고, `fixes-url` 검사는 설정에 따라 제외됐다.
`make lint-js`와 `git diff --check`도 통과했다.

### 과거 QUIC 실패를 함께 확인한 이유

10월 20일 점검 기록에는 커밋 메시지 린트 외에
`test-quic-stream-truncated-reads-destroy.mjs:56`의 `1000 !== 0` 실패가 남아 있었다.
메시지 수정과 리베이스가 끝났다는 이유만으로 이 실패까지 해결됐다고 말할 수 없다.

해당 테스트는 세션을 강제로 종료한 뒤 첫 읽기를 시작하면 수신량이 0이라고 기대한다.
우리 수정은 reader를 미리 만들어 이미 받은 바이트를 보존하므로, 이 기대와 충돌할 가능성이 있다.
테스트는 pimterry의 커밋 `11ed32572b`와 PR #63967에서 추가됐다.

이번 진단에서 기존 9월 30일 QUIC 활성 바이너리로 두 테스트를 실행했다.

```sh
out/Release/node --experimental-quic --experimental-stream-iter \
  --no-warnings --test --test-timeout=30000 \
  test/parallel/test-quic-stream-uni-delayed-reader.mjs \
  test/parallel/test-quic-stream-truncated-reads-destroy.mjs
```

새 delayed-reader 테스트는 통과했고, 기존 destroy 테스트는 같은 `1000 !== 0`으로 실패했다.
이 실행은 오래된 바이너리를 사용한 진단이지, 리베이스한 최신 소스의 검증 결과가 아니다.
최신 소스는 `make -j6`으로 QUIC 활성 빌드를 진행해 별도로 확인한다.

실패를 없애려고 기존 테스트의 기대값만 임의로 바꾸지 않는다.
데이터 보존과 명시적 취소의 계약을 구분하고, 기존 테스트 변경이 필요하면 사용자의 판단을 먼저 받는다.

### 최신 소스로 다시 빌드한 결과

`make -j6`이 성공했고, 생성된 `out/Release/node`에서
`process.config.variables.node_use_quic === true`를 확인했다.
빌드 과정에서 Rust 도구를 찾지 못해 Temporal 지원은 비활성화됐다는 경고가 있었지만,
QUIC 활성 빌드는 성공했다. 이 결과를 Temporal까지 검증한 것으로 표현하지 않는다.

Node 내부 JS는 현재 설정에서 실행 파일 안에 포함된다.
따라서 작업 트리의 `lib/internal/quic/quic.js`만 바꾸고 예전 실행 파일로 테스트하면
바뀐 JS가 아니라 실행 파일에 들어 있던 코드를 검사할 수 있다.
이번 리베이스에는 V8와 여러 의존성 변경도 포함돼 재빌드 범위가 컸다.
일반적인 npm 라이브러리의 파일 수정 후 테스트 실행과는 다른 점이다.

최신 빌드로 다음 테스트를 저장소 테스트 러너에서 실행했다.

```sh
python3 tools/test.py --mode=release -j4 -t20 -p tap \
  parallel/test-quic-stream-uni-delayed-reader \
  parallel/test-quic-stream-uni-basic \
  parallel/test-quic-stream-uni-server-initiated \
  parallel/test-quic-stream-uni-no-onstream \
  parallel/test-quic-stream-iteration-nonreadable \
  parallel/test-quic-stream-truncated-reads \
  parallel/test-quic-stream-truncated-reads-timeout \
  parallel/test-quic-stream-truncated-reads-destroy
```

결과는 **7개 통과, 1개 실패**다. 스킵으로 통과 처리된 테스트는 없다.
실패는 기존 CI와 같은 destroy 테스트 56번째 줄의 `1000 !== 0`이다.
리베이스와 커밋 메시지 정리만으로 이 실패까지 사라진 것은 아니다.

기존 테스트를 바꾸지 않고, 별도 임시 스크립트에서 `readStream`과 `stallingBody(1000)`으로
실제 반환값과 오류를 관찰했다. 모두 첫 읽기 전에 데이터 수신을 확인하고 강제로 종료했다.

| 종료 대상 | `truncatedReads` | 수신량 | 읽기 오류 | `closed` 오류 |
| --- | --- | ---: | --- | --- |
| session | `error` | 1,000 | `ERR_QUIC_STREAM_ABORTED` | 없음 |
| session | `ignore` | 1,000 | 없음 | 없음 |
| stream | `error` | 1,000 | `ERR_QUIC_STREAM_ABORTED` | 없음 |
| stream | `ignore` | 1,000 | 없음 | 없음 |

즉 현재 수정이 잘린 스트림의 오류를 삼키는 현상은 이 네 사례에서 관찰되지 않았다.
바뀐 것은 늦게 읽기 시작해도 이미 받은 바이트가 남아 있다는 점이다.
이 관찰이 모든 종료 경로의 안전성을 증명하는 것은 아니며, 기존 테스트와 기대가 다른 사실도 남는다.

### 구현을 고쳐야 하는가, 테스트를 고쳐야 하는가

이슈의 재현은 상대가 모든 데이터를 보내고 정상 FIN을 보낸 뒤 늦게 읽는 상황이다.
실패한 기존 테스트는 수신 도중 사용자가 세션 또는 스트림을 강제로 종료하고 뒤늦게 읽는 상황이다.
reader 선생성은 두 경로 모두에서 큐를 보존하므로, 정상 종료뿐 아니라 강제 종료도 검토 대상이다.

공식 `sessionOptions.truncatedReads` 문서는 받은 데이터를 먼저 전달하고 이후 오류를 알린다고 설명한다.
native `Stream::Destroy()`도 `TruncateReadable()`에서 수신 데이터를 flush한 다음,
JS reader가 큐를 참조하면 데이터가 더 오래 살아 있을 수 있다고 주석으로 설명한다.
즉 강제 종료가 언제나 읽지 않은 데이터를 무조건 버린다고 단정할 근거는 부족하다.

그렇더라도 테스트를 통과시키려는 목적으로 숫자만 `0`에서 `1000`으로 바꿔서는 안 된다.
판단할 기준은 다음과 같다.

- 명시적 강제 종료 뒤의 데이터 폐기가 보존할 계약이면 구현에서 그 경로를 구분해야 한다.
- reader 존재 시 이미 허용하던 데이터 보존을 늦은 읽기에도 적용하는 것이 맞으면,
  오류 검사를 유지하면서 기존 테스트의 바이트 기대값과 설명을 보완할 수 있다.

기존 테스트의 assertion을 약화하거나 삭제하지 않은 상태로 결과를 먼저 사용자에게 설명한다.

### 선생성 코드를 끈 비교 검증

원인을 구분하기 위해 `QuicStream` 생성자에 추가한 6줄만 임시로 제거했다.
이때 `lib/internal/quic/quic.js`는 upstream main과 차이가 없음을 확인했다.
다시 빌드한 뒤 동일한 진단을 실행하면 네 경우 모두 수신량이 0으로 바뀌었고,
오류 및 `closed` 결과는 선생성 코드가 있을 때와 같았다.

| 검사 | 선생성 없음 | 선생성 있음 |
| --- | --- | --- |
| 늦게 읽는 정상 FIN 회귀 테스트 | 30초 제한에서 시간 초과 | 통과 |
| 기존 destroy 테스트 | 통과 | `1000 !== 0` assertion 실패 |
| session/stream 종료 후 첫 읽기 | 0바이트 | 1,000바이트 |
| 기본 정책의 truncation 오류 | `ERR_QUIC_STREAM_ABORTED` | `ERR_QUIC_STREAM_ABORTED` |
| `ignore` 정책의 읽기 오류 | 없음 | 없음 |

처음 비교 실행은 테스트 러너에 `-t10`을 주었고, 실제 적용된 약 30초 제한에
기존 destroy 테스트까지 걸렸다. 이 timeout을 기능 실패로 확정하지 않았다.
`-t60`으로 해당 테스트를 단독 재실행하자 약 30.15초에 통과했다.
새 delayed-reader 테스트는 선생성 코드가 없는 상태에서 시간 초과가 발생했다.

따라서 기존 CI의 바이트 assertion 실패는 이번 리베이스가 새로 만든 것이 아니며,
reader를 먼저 생성하는 변경과 직접 연결된다. 다만 “변경 때문에 기대값이 달라졌다”와
“그 변경이 잘못됐다”는 다른 판단이다. 버그 수정으로 기존의 데이터 유실이 사라진 것인지,
별도로 보존해야 할 취소 동작을 바꾼 것인지는 계약과 사용 사례를 기준으로 결정해야 한다.

비교 후 임시로 제거한 6줄을 그대로 복원했고 `git diff --exit-code`로
작업 트리가 커밋 `f057aabf68`과 일치하는 것을 확인했다.
기존 테스트는 수정하지 않았고, 아직 이 결과를 근거로 CI 재실행을 요청하지 않았다.

### 이번 작업의 완료 범위와 보류 사항

- 커밋 메시지 줄 길이 수정, 최신 main 리베이스, 커밋 린트와 JS 린트는 완료했다.
- QUIC 활성 빌드와 비교 검증을 마쳤고, 임시 진단 변경은 모두 복원했다.
- 복원 뒤 빌드도 성공했고, delayed-reader와 nonreadable 테스트를 다시 실행해 둘 다 통과했다.
- 기존 테스트 파일은 바꾸지 않았다. 이번에 커밋에 추가된 기능 수정도 없다.
- 로컬 브랜치 head는 `f057aabf68`, 원격 PR head는 여전히 `2526c29286`이다.
- 알려진 assertion 실패가 남아 있어 코드 브랜치 push와 CI 재실행 요청은 보류했다.
- PR 댓글과 본문은 수정하지 않았다.
- 전체 Node 테스트 및 Linux/Jenkins 검증을 통과했다고 주장하지 않는다.

다음 판단에서는 데이터 보존 자체와 truncation 오류 전달을 분리해서 본다.
현 단계에서는 기존 문서와 native 종료 코드, 비교 결과를 근거로
바이트 기대값을 보완하는 방향이 타당해 보이지만, 기존 테스트를 고치는 것은
메시지 정리·리베이스와 다른 변경이므로 사용자 검토 후 진행한다.

## 2026-10-24 추가 검토와 종료 테스트 보강

앞의 검토 결과를 설명한 뒤, 사용자가 다시 검토를 요청했다. 정상 FIN 뒤의 데이터 유실이
해결된 것만으로 충분한지, 강제 종료 뒤에도 데이터를 보존하는 것이 맞는지부터 다시 확인했다.
검토 내용을 설명하고 사용자가 진행을 요청한 뒤 기존 테스트와 종료 동작 설명을 보강했다.
아래 기록은 앞의 실패 결과와 보류 판단을 지우지 않고 그 다음 과정을 이어 적은 것이다.

### 기존 테스트 실패는 무관한 문제가 아니었다

기존 `test-quic-stream-truncated-reads-destroy.mjs`에는 읽기를 시작하기 전에
세션이나 스트림을 종료하면 읽은 바이트 수가 0이어야 한다는 assertion이 있었다.
reader 선생성 이후에는 이미 받은 1,000바이트가 그대로 전달되어 이 assertion이 실패했다.

따라서 이 실패는 단순한 테스트 환경 문제나 upstream의 별도 실패로 분리할 수 없다.
이번 수정이 정상 FIN뿐 아니라 로컬 강제 종료에도 영향을 준다는 증거다.
정상 종료의 데이터 유실을 없애는 것과 강제 종료 때의 기존 동작을 바꾸는 것은
사용자 입장에서 서로 다른 변화이므로, 후자를 설명하지 않고 넘어가면 안 된다.

### 서로 다른 설명을 어떻게 판단했는가

`QuicStream.destroy()`의 내부 JSDoc에는 대기 중인 데이터를 버린다는 문장이 남아 있었다.
이 문장은 초기 QUIC 구현의 커밋 `cf91d181fbd0`에서 들어왔다.
반면 이후 truncation 처리를 추가한 커밋 `11ed32572b39`와
[PR #63967](https://github.com/nodejs/node/pull/63967)은 받은 데이터를 먼저 전달하고,
마지막 읽기에서 불완전한 종료를 알리는 방향을 설명한다.

현재 `sessionOptions.truncatedReads` 공개 문서도 같은 순서를 설명한다.
native `Stream::Destroy()`는 수신 큐의 데이터를 flush한 뒤 참조를 해제하고,
JS reader가 있으면 데이터가 더 오래 살아 있을 수 있음을 명시한다.

이 근거를 종합해, reader를 늦게 요청했다는 이유로 이미 받은 데이터를 잃게 하기보다는
받은 데이터를 전달하되 종료 오류를 유지하는 방향으로 보강했다.
이것이 메인테이너가 이번 테스트 변경까지 승인했다는 뜻은 아니다.
PR에는 강제 종료의 관찰 가능한 동작도 달라진다는 사실을 설명해야 한다.

### 0을 1000으로만 바꾸면 안 되는 이유

기존 테스트는 다음 조건으로 기다렸다.

```js
while (stream.stats.bytesReceived === 0n) await setTimeout(5);
```

이 조건은 첫 데이터가 도착했는지만 확인한다. 서버가 1,000바이트를 보냈다는 사실과
클라이언트가 그 1,000바이트를 모두 받았다는 사실은 다르다.
일부만 받은 시점에 강제 종료해 놓고 정확히 1,000바이트를 읽어야 한다고 검사하면,
실행 타이밍에 따라 실패하는 테스트를 만들 수 있다.

따라서 종료 전에 전체 수신을 기다리도록 바꿨다.

```js
while (stream.stats.bytesReceived < 1000n) await setTimeout(5);
session.destroy();
```

스트림 자체를 종료하는 경우에도 같은 대기 조건을 사용한다.
그 다음 읽은 양이 정확히 1,000바이트인지 확인한다.
`ERR_QUIC_STREAM_ABORTED` 검사와 오류 없는 `destroy()`에서 `closed`가 정상 완료되는
기존 검사는 유지했다. 테스트를 없애거나 오류 검사를 약하게 만든 것이 아니다.
기다림이 끝나지 않는 경우는 테스트 러너의 시간 제한으로 실패하도록 한다.

### 데이터를 전달하면서 원래 오류를 잃지는 않는가

종료 뒤에도 데이터가 남는다는 사실만으로 오류 처리까지 안전하다고 볼 수는 없다.
추가한 네 사례는 다음 조합을 검증한다.

| 종료 대상 | truncation 정책 | 종료 호출 |
| --- | --- | --- |
| 세션 | `error` | `session.destroy(boom)` |
| 세션 | `ignore` | `session.destroy(boom)` |
| 스트림 | `error` | `stream.destroy(boom)` |
| 스트림 | `ignore` | `stream.destroy(boom)` |

각 사례에서 서버는 128KiB를 보낸 뒤 FIN을 보내지 않고 대기한다.
클라이언트는 128KiB 전체 수신을 확인하고, 첫 읽기 전에 명시적인 오류로 종료한다.
이 크기는 reader의 64KiB 배치 한 번보다 커서, 여러 번 읽은 뒤에도 오류가 남는지 확인한다.
전체 데이터를 읽기 전에 받을 수 있도록 테스트의 스트림·연결 수신 한도를 충분히 설정한다.

각 사례의 검사는 다음 세 가지다.

```js
assert.strictEqual(received, size);
assert.strictEqual(threw, boom);
assert.strictEqual(closedError, boom);
```

오류 메시지가 비슷한지만 보는 것이 아니라, 호출자가 넘긴 바로 그 오류 객체인지 확인한다.
`ignore`는 오류 없는 truncation을 허용하는 설정이지 명시적인 오류까지 무시하는 설정이 아니다.
이 차이를 받은 바이트 수와 별도로 검증한다.

### 종료 설명과 자원 수명

내부 JSDoc에서 모든 대기 데이터를 버린다는 설명을 수신 버퍼가 읽을 수 있는 상태로
남는다는 설명으로 바꿨다. 공개 `stream.destroy()` 문서에도 읽기를 시작하지 않았더라도
수신 버퍼는 읽을 수 있고, FIN 없이 끝났다면 데이터를 전달한 뒤 정책에 따라
truncation이나 오류를 보고한다는 설명을 추가했다.

reader를 미리 만든다고 즉시 데이터를 읽거나 복사하는 것은 아니다.
그러나 reader가 수신 큐를 참조하므로, 읽지 않은 데이터가 스트림 종료 직후 사라지는 대신
JS 참조가 유지되는 동안 더 오래 남을 수 있다. 늦은 읽기를 지원하는 데 따르는 비용이다.
이를 메모리 누수라고 판정하지는 않았다. 이번 검증에서는 GC 회수 시점이나 사용량을 측정하지 않았다.
native 종료 코드의 listener 해제와 flow-control credit 반환도 함께 확인했다.

이번 보강은 기존 reader 선생성 로직을 그대로 둔다. 추가한 것은 종료 테스트,
내부 설명, 공개 API 문서이며, 별도의 production 분기나 새로운 종료 방식을 도입하지 않았다.

### 보강 후 검증 결과

내부 JS 파일의 주석도 변경했으므로 `make -j6`으로 증분 빌드를 완료한 뒤 검증했다.
새로운 C++ 변경은 없으며, 앞서 끝낸 전체 빌드를 처음부터 반복하지 않았다.

```sh
python3 tools/test.py --mode=release -j6 -t60 -p tap 'parallel/test-quic*'
```

결과는 **271개 대상 중 269개 통과, 2개 스킵, 실패 0개**다.
기존에 실패했던 `test-quic-stream-truncated-reads-destroy`도 추가한 네 사례와 함께
약 50.4초에 통과했다. 원래 이슈의 delayed-reader 테스트도 통과했다.

스킵된 항목은 다음 두 개다.

- `test-quic-test-client`: `ngtcp2_test_client` 실행 파일이 빌드되지 않음.
- `test-quic-test-server`: `ngtcp2_test_server` 실행 파일이 빌드되지 않음.

테스트 결과에는 flow-control credit 반환, 종료한 스트림의 재등장 방지,
비가독 스트림, iteration, HTTP/3 등 인접 QUIC 경로도 포함된다.
이 결과를 전체 Node 테스트, 다른 운영체제, 원격 Jenkins CI의 통과로 확대해서 말하지 않는다.

변경한 JS 파일과 기존 delayed-reader 회귀 테스트의 ESLint, `doc/api/quic.md`의
Markdown lint, 두 저장소의 `git diff --check`도 통과했다.
문서 린트는 처음에 `unified` 의존성이 없어 실행하지 못했다.
`tools/lint-md`에서 `npm ci --ignore-scripts`로 잠금 파일에 맞게 설치한 뒤 다시 실행해 통과했다.
패키지 파일이나 잠금 파일 변경은 없다.

이 단계에서는 코드와 복습 문서의 로컬 변경까지만 완료했다.
추가 커밋, 코드·문서 push, PR 본문 수정이나 댓글 게시는 하지 않았다.

### 사용자 확인 후 커밋·push·PR 갱신

사용자가 로컬 보강 결과를 확인한 뒤 커밋, push와 PR 변경을 요청했다.
Git author/email과 GitHub 활성 계정이 모두 `GiHoon1123`인지 확인하고 진행했다.

- `f057aabf68`: 기존 reader 선생성 수정. main `763524d434` 위로 리베이스하고
  커밋 메시지를 72열 이내로 줄바꿈한 커밋이다.
- `c3cf3300a1` (`test: cover QUIC reads after local teardown`): 종료 테스트 보강,
  내부 JSDoc와 공개 `stream.destroy()` 설명을 담은 후속 커밋이다.

두 커밋 모두 `Assisted-by: Codex`와 사용자의 `Signed-off-by`를 유지했다.
공동 저자 표시는 추가하지 않았다. Node 저장소가 요구하는 공개 표기와 DCO를 따른다.

CI에서 사용하는 `core-validate-commit@6.0.0 --no-validate-metadata --tap -`로
두 커밋 메시지를 검사해 통과했다. 처음에는 원시 메시지를 전달해 JSON 입력 오류가 났고,
다음에는 `git show`의 출력용 개행까지 넣어 trailer 판정이 실패했다.
GitHub API의 메시지 형태에 맞춰 출력용 끝 개행을 제거한 JSON 배열로 다시 실행하자
총 18개 검사에서 16개 통과, `fixes-url` 검사 2개 스킵으로 종료 코드 0을 확인했다.
이 입력 오류를 고치려고 커밋의 필수 trailer를 삭제하거나 메시지를 다시 바꾸지는 않았다.

원격 브랜치가 여전히 `2526c29286eb7ac14b88ab1d9fc810729c2630a2`인지 확인한 뒤,
그 값을 명시한 `--force-with-lease`로 **사용자 포크**의
`fix/quic-unidirectional-reader`를 갱신했다. `nodejs/node`에는 직접 push하지 않았다.
이후 PR API에서도 최신 head가 `c3cf3300a1`임을 확인했다.

PR 본문은 기존 문제 설명과 eager allocation의 비용 설명을 유지했다.
강제 종료 뒤의 수신 데이터 보존, 전체 수신을 기다리는 테스트 조건,
128KiB 데이터와 명시적 오류의 검증, 리베이스와 커밋 메시지 정리,
로컬 테스트 269개 통과·2개 스킵을 추가했다. 별도 댓글은 게시하지 않았다.

push 직후 GitHub에서 `lint-commit-message` 검사가 실행 중인 것을 확인했다.
전체 원격 CI가 통과한 상태라고 기록하지 않는다. 추가 CI는 메인테이너가 설명한
수동 실행 절차가 필요하며, 로컬 테스트 통과와는 별개다.

## 2026-11-03 확인 — 승인과 main 반영 완료

PR은 한국시간 **2026-11-02 21:22:48**에 머지됐다. UTC로는 `2026-11-02T12:22:48Z`다.
GitHub의 상태는 MERGED이며, 반영 계정은 `nodejs-github-bot`이다.
[반영 완료 댓글](https://github.com/nodejs/node/pull/66407#issuecomment-6577025826)과
[최종 커밋](https://github.com/nodejs/node/commit/1389475f66a722eab83547797491c3e879de3f7f)을 확인했다.
이에 따라 기록을 `typescript/open`에서 `typescript/merge`로 옮겼다.

### 재오픈 뒤의 논의와 최종 승인

이슈 제보자 `martenrichter`는 다른 방법도 생각했지만 readable stream의 reader를
미리 만드는 접근이 필요해 보인다고 의견을 남겼다. 이후 자기 WebTransport 작업 브랜치에
이 변경을 cherry-pick했고, 우리 PR을 다시 열도록 권했다.
이는 단순히 코드를 참고했다는 뜻이 아니라 커밋을 가져가 적용했다는 뜻이다.
우리 수정을 대체하는 새 PR을 제출하겠다는 발언으로 해석하지 않았다.

한국시간 10월 31일 오전 7시 23분, `pimterry`가 최신 제출 커밋 `c3cf3300a1`을
[APPROVED](https://github.com/nodejs/node/pull/66407#pullrequestreview-5719316007)했다.
좋은 개선이라는 데 동의하며 진행시키겠다는 의견을 남겼다.
같은 날 오후 12시 38분에는 `bjohansebas`의 추가 승인도 있었다.
새로운 인라인 수정 요청은 없었다.

승인 직후 [Jenkins CI #78753](https://ci.nodejs.org/job/node-test-pull-request/78753/)의
실행 링크가 게시됐다. 당시 Jenkins 상세 API는 403으로 접근하지 못했으므로,
그 내부 결과를 직접 읽었다고 기록하지 않는다.
11월 3일 GitHub에 연결된 체크는 SUCCESS 59개, SKIPPED 12개였고 실패는 없었다.
체크 개수는 테스트 케이스 수가 아니며, 머지 후 자동화가 추가되면서
11월 2일 조회의 스킵 4개와도 달라질 수 있다.

### 실제 반영된 범위

최종 main 커밋에는 다음 네 파일이 포함돼 있다.

- `lib/internal/quic/quic.js`: readable stream의 reader를 미리 만들고 종료 설명을 보완했다.
- `test/parallel/test-quic-stream-uni-delayed-reader.mjs`: 정상 FIN 뒤 늦은 읽기의 회귀 테스트다.
- `test/parallel/test-quic-stream-truncated-reads-destroy.mjs`: 로컬 종료 뒤 수신 데이터와 오류 전달을 검증한다.
- `doc/api/quic.md`: 종료 후 수신 버퍼의 가독성과 truncation 처리 순서를 설명한다.

처음 제출한 데이터 유실 수정뿐 아니라, 이후 보강한 강제 종료 테스트와 문서까지
최종 반영 커밋에 포함된 것을 확인했다. Node의 JavaScript QUIC 계층과 native 자원의
수명 경계를 고친 코어 기여이며, 별도 npm 패키지만 수정한 작업은 아니다.

반영 커밋 메시지에는 사용자의 `Signed-off-by`, `Assisted-by: Codex`, PR·이슈 링크와
Tim Perry의 `Reviewed-By`가 남아 있다. GitHub API에서 최종 커밋의 author 표시 이름은
`Dustin`, 이메일은 `85157729+GiHoon1123@users.noreply.github.com`으로 확인됐다.
이는 앞서 로컬에서 확인한 `GiHoon1123 <rlaejrqo465@naver.com>`와 표시가 다르므로
둘을 같은 메타데이터였던 것처럼 기록하지 않는다. PR 작성 계정은 `GiHoon1123`이다.

main 반영과 릴리스 배포는 별개다. 이 기록에서는 특정 Node 배포 버전에 포함됐다고
주장하지 않으며, 문서 갱신을 위해 코드를 수정하거나 테스트를 다시 실행하지 않았다.
