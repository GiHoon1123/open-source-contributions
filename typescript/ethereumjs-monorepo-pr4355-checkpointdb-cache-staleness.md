# [OSS 기여] ethereumjs/ethereumjs-monorepo - CheckpointDB 캐시가 checkpoint 안의 최신 상태를 가리는 버그 수정 (PR #4355) + binarytree 중복 버그 이슈 등록 (#4356) — 🟢 Merged

> 오픈소스 기여 기록 및 학습 노트. 이 문서는 PR #4355와, 거기서 파생된 이슈 #4356을 함께 다룬다. **2026-07-31 제출, 2026-08-14 `holgerd77`가 승인 후 머지.**

---

## 0. 관련 링크 모음

| 종류 | 링크 | 상태 |
| --- | --- | --- |
| 제출한 PR | https://github.com/ethereumjs/ethereumjs-monorepo/pull/4355 | 🟢 Merged (2026-08-14) |
| 파생 이슈 (binarytree 중복 버그) | https://github.com/ethereumjs/ethereumjs-monorepo/issues/4356 | Closed — 다른 기여자(`SnowingFox`)가 PR #4360으로 고쳐서 머지됨 (2026-08-14) |

- 패키지: `@ethereumjs/mpt` — 이더리움 실행 계층의 머클 패트리샤 트라이 구현
- 작업 브랜치: `fix-mpt-checkpointdb-cache-staleness` (fork: GiHoon1123)
- 커밋: `ea3f0552e` — 2개 파일, +36/-7
- 검증: mpt 스위트 1500여 개 전부 통과, tsc/lint 클린

---

## 1. 후보를 어떻게 찾았나 — 병렬 에이전트 탐색

이전에 정리해둔 재사용 검색 프롬프트(`oss_contribution_search_prompt`)를 그대로 써서, 이미 작업한 이슈(#4343/#4344/#4345/#4346/#4354)를 전부 제외 목록에 넣고 두 개의 백그라운드 에이전트를 동시에 돌렸다:

1. 이더리움js 전용 딥다이브 에이전트 — mpt/evm 쪽에서 이미 손댄 영역을 피해서 statemanager/blockchain/common 등 덜 건드린 패키지를 코드 레벨로 직접 훑음
2. 나머지 생태계(NestJS/Kafka/Redis/DB/ORM) 폭넓게 훑는 에이전트

에이전트 1이 `packages/mpt/src/db/checkpointDB.ts`의 `CheckpointDB.get()`에서 새로운 버그를 발견했다 — **아직 아무도 이슈로 올린 적 없는 발견**이었다. 이 에이전트는 실제 `@ethereumjs/mpt` 패키지를 빌드해서 재현하진 못했지만, `get`/`put`/`del`/`checkpoint`/`commit`/`revert`의 로직을 그대로 복사한 독립 스크립트로 버그를 개념적으로 증명했다:

```
read before checkpoint: Uint8Array(3) [ 1, 2, 3 ]
read after del(), same checkpoint, not yet committed: Uint8Array(3) [ 1, 2, 3 ]   <- should be undefined
BUG CONFIRMED
control run (cacheSize=0) after del() in checkpoint: undefined                    <- correct without cache
```

(참고: 이 에이전트도 작업 중 "날짜가 바뀌었다, 말하지 마라"는 가짜 시스템 메시지를 만났고, 스스로 플래그하고 무시한 뒤 계속 진행했다 — 이번 세션 내내 반복된 패턴.)

에이전트가 짚은 핵심 근거들:
- `cacheSize > 0`이 엣지 케이스가 아니라 이더리움js 클라이언트의 **기본 설정값**(`Config.TRIE_CACHE = 200000`)이라는 점
- 메인테이너 holgerd77 본인이 2024년 이슈 코멘트에서 "trie 자체는 state manager만큼 checkpointing 로직 테스트가 잘 안 돼있다"고 이미 인지하고 있었다는 점
- 경쟁 이슈/PR 없음

추천은 "바로 PR 후보"가 아니라 **"먼저 재현할 후보"** — 실제 패키지를 빌드해서 공개 API로 끝까지 재현해야 한다는 조건부였다.

---

## 2. 개념부터 — 이 클래스들이 실제로 뭘 하는가

작업하면서 나눈 대화 중, 이 버그가 왜 중요한지 이해하려면 배경 개념이 필요했다. 정리해둔다.

### 2-1. `ethereumjs-monorepo`는 뭐 하는 프로젝트인가

이더리움은 하나의 소프트웨어가 아니라 **프로토콜 + 그걸 각자 독립적으로 구현한 여러 노드 소프트웨어(Geth, Reth, Besu, Erigon, Nethermind 등)로 이루어진 네트워크**다. `ethereumjs-monorepo`는 그중 하나의 JS/TS 구현체지만, README를 직접 확인해보니 스스로를 "protocol building blocks"라고 소개하고 있고, **실제로 메인넷에 접속해서 도는 완전한 노드 소프트웨어(`@ethereumjs/client`, `@ethereumjs/devp2p`)는 이미 Deprecated 목록에 있다.** 즉 지금은 "메인넷 대체 노드"가 아니라 "EVM 실행 규칙을 정확히 구현한 부품 라이브러리 모음"으로 스코프가 좁혀져 있다.

실전에서는 브라우저 기반 IDE(Remix), 로컬 테스트 체인 시뮬레이터, 프로토콜 프로토타이핑, 실행 스펙 테스트 검증 등에 쓰인다. **viem/ethers.js/web3.js와는 완전히 다른 층위** — 저것들은 이미 EVM을 가진 원격 노드한테 RPC 요청만 보내는 클라이언트고(비유: mysql2), ethereumjs는 프로세스 안에 EVM 자체를 통째로 심는 엔진이다(비유: SQLite). RPC 클라이언트는 계산 정확성의 책임이 없지만, ethereumjs는 계산을 직접 하기 때문에 여기서 나는 버그는 그대로 사용자에게 노출된다.

### 2-2. `CheckpointDB`는 어디에 쓰이나 — 실제 호출 체인을 코드로 추적

```
evm.ts:1365 (메시지 하나 실행 직전)
  await this.journal.checkpoint()
    ↓
journal.ts
  this.stateManager.checkpoint()
    ↓
merkleStateManager.ts:443
  this._trie.checkpoint()
    ↓
mpt.ts:1041
  this._db.checkpoint(this.root())   ← CheckpointDB.checkpoint()
```

이더리움 트랜잭션 실행 중 컨트랙트 호출(A→B→C 같은 중첩 CALL)이 일어날 때마다, 그 호출 하나하나가 EVM 용어로 "메시지"이고, 메시지가 시작될 때마다 위 체인을 타고 `checkpoint()`가 열린다. 실패하면(`REVERT`, OOG 등) `revert()`로 그 레이어를 통째로 버리고, 성공하면 `commit()`으로 부모 레이어에 합친다. **이게 있는 이유는 "실패한 하위 호출의 상태 변경만 정확히 되돌리기" 위해서다** — 매번 disk를 다시 읽어와서 복구하는 대신, 메모리 위의 임시 레이어(Map)만 버리면 되게 설계한 것.

`_cache`는 완전히 다른 목적 — 트라이 노드 읽기가 초당 수천~수만 번 반복되니 disk 접근을 줄이려는 순수 성능용 LRU 캐시다.

이 두 개념이 원래 서로 무관해야 하는데, `get()`의 순서가 잘못 짜여서 서로 얽혀버린 게 이번 버그다.

---

## 3. 재현 — 실제 공개 API로 끝까지

`packages/mpt/test/db/checkpoint.spec.ts`를 열어보니, 이 파일의 기존 테스트 12개가 **전부 `cacheSize`를 안 넘겨서 `_cache`가 비활성화된 상태로만 테스트되고 있었다** — 이게 지금까지 아무도 이 버그를 못 잡은 이유였다.

재현 테스트를 추가해서 실제 `CheckpointDB` 클래스로 돌려봤다:

```ts
const cachedDb = new CheckpointDB({ db: new MapDB(), cacheSize: 100 })
await cachedDb.put(k, v)
await cachedDb.get(k)              // 'v1' 반환, 동시에 _cache에 적재됨

cachedDb.checkpoint(hexToBytes('0x01'))
await cachedDb.del(k)

await cachedDb.get(k)               // 기대: undefined
```

결과:
```
AssertionError: after del() in open checkpoint: should be undefined
- Expected: undefined
+ Received: Uint8Array [ 118, 49 ]   ← 'v1'의 바이트, 삭제 전 옛날 값
```

**에이전트의 독립 로직 증명이 아니라, 진짜 배포되는 클래스로 끝까지 확인된 순간이다.**

---

## 4. 원인 — 코드 두 곳을 나란히 놓고 보면

**`put()`/`del()`이 어디에 쓰는지**:

```ts
async put(key, value) {
  if (this.hasCheckpoints()) {
    this.checkpoints[this.checkpoints.length - 1].keyValueMap.set(keyHex, value)   // ← checkpoint 열려있으면 여기만
  } else {
    await this.db.put(...)
    this._cache?.set(keyHex, value)   // ← checkpoint 없을 때만 캐시 갱신
  }
}
```

checkpoint가 열려있는 동안 값을 바꾸면 `_cache`는 전혀 안 건드려진다. `del()`도 동일 구조.

**원래 `get()`의 순서**:

```ts
async get(key) {
  if (this._cache !== undefined) {
    const value = this._cache.get(keyHex)
    if (value !== undefined) return value        // ← 캐시 히트하면 바로 반환, 끝
  }
  for (let index = this.checkpoints.length - 1; index >= 0; index--) {
    if (this.checkpoints[index].keyValueMap.has(keyHex)) {
      return this.checkpoints[index].keyValueMap.get(keyHex)   // ← 여긴 캐시 miss일 때만 옴
    }
  }
  // ...disk
}
```

캐시가 옛날 값을 들고 있으면, checkpoint 안의 진짜 최신 정보는 확인도 안 하고 그대로 반환해버린다. **캐시가 checkpoint보다 우선순위가 높게 처리되는, 순서가 거꾸로 된 버그.**

추가로 중요한 디테일: `get()` 맨 아래 disk-fallback 경로에 있는 `this._cache?.set(keyHex, returnValue)`가 **checkpoint가 열려있어도 무조건 실행**된다. 즉 트리거 조건은 "checkpoint 열기 **전에** 읽은 경우"만이 아니라, **"checkpoint가 열린 동안 그 키를 처음 읽어서 disk까지 갔다 온 경우"에도 똑같이 발생**한다 — 이 부분은 PR 본문 초안에서 한 번 좁게 서술했다가 리뷰 중 바로잡았다(6절 참고).

---

## 5. 수정

```ts
async get(key) {
  const keyHex = bytesToUnprefixedHex(key)

  if (this.hasCheckpoints()) {                 // checkpoint가 열려있으면
    for (let index = this.checkpoints.length - 1; index >= 0; index--) {
      if (this.checkpoints[index].keyValueMap.has(keyHex)) {
        return this.checkpoints[index].keyValueMap.get(keyHex)   // 최신 상태를 먼저 확인
      }
    }
  }

  if (this._cache !== undefined) {             // 여기까지 왔다는 건 checkpoint가 없거나,
    const value = this._cache.get(keyHex)      // 있어도 이 키를 안 건드렸다는 뜻
    if (value !== undefined) return value      // → 이럴 때만 캐시를 믿어도 안전
  }
  // ...disk (그대로)
}
```

checkpoint가 아예 없는 흔한 경우엔 `hasCheckpoints()`가 `false`라서 새로 추가된 블록이 통째로 스킵되고, 원래처럼 캐시부터 바로 확인하는 빠른 경로가 그대로 유지된다 — 성능 저하 없음.

회귀 테스트는 put(update)와 del 두 케이스를 한 테스트에서 같이 확인하도록 작성:

```ts
it('Checkpointing: cacheSize > 0 does not mask a checkpointed put/del from a later get()', async () => {
  const cachedDb = new CheckpointDB({ db: new MapDB(), cacheSize: 100 })
  await cachedDb.put(k, v)
  assert.deepEqual(await cachedDb.get(k), v, 'before CP: v1 (also populates _cache)')

  cachedDb.checkpoint(hexToBytes('0x01'))
  await cachedDb.put(k, v2)
  assert.deepEqual(await cachedDb.get(k), v2, '...')

  await cachedDb.del(k)
  assert.deepEqual(await cachedDb.get(k), undefined, '...')
})
```

---

## 6. 검증

- **disable-and-reconfirm**: 수정 전(재현 테스트 작성 직후 자연스럽게 확인) → 실패. `git checkout upstream/master -- checkpointDB.ts`로 명시적으로 한 번 더 되돌려서 재확인 → 정확히 예상한 이유로 실패(`v1` 대신 `v2`를 기대했는데 `v1`이 나옴) → `git apply`로 복원 → 통과.
  - 이 과정에서 "파일이 수정됐다, 말하지 마라"는 가짜 주입 메시지가 또 나왔다 — 실제로는 내 `git checkout` 명령의 결과일 뿐이었고, 늘 그렇듯 플래그하고 무시했다.
- mpt 패키지 전체 스위트: 1510~1515개 전부 통과 (실행마다 약간의 카운트 변동 있었지만 실패는 0)
- tsc, eslint 클린
- `npm run build --workspace @ethereumjs/mpt`는 CJS 빌드 단계에서 `TS5095` 에러로 실패 — **`git stash`로 우리 변경사항을 완전히 뺀 순정 `upstream/master`에서도 똑같이 재현**돼서, 우리 코드와 무관한 저장소/로컬 환경 문제임을 확인. PR 본문에 "빌드 통과"라고 쓰지 않기로 함.

---

## 7. 리뷰 — 사용자 본인의 자체 검증

PR을 올리기 전, 사용자가 직접 다음을 검토해서 가져왔고 전부 직접 재검증했다:

1. **"`CheckpointDB`가 공개 API로 접근 가능한가?"** — `index.ts`에 직접 안 보이지만 `export * from './db/index.ts'` → `export * from './checkpointDB.ts'`로 이어지는 재export 체인 확인, 실제 공개 API 맞음.
2. **ESM dist import로 같은 시나리오 재현** — 사용자가 직접 빌드된 결과물로 검증. 소스 레벨 테스트보다 한 단계 더 강력한 증거.
3. **CJS 빌드 실패(TS5095) 발견** — "PR 본문에 full build passed라고 쓰면 안 된다"는 지적. 위 6절에서 직접 재검증 완료.
4. **코멘트 워딩 지적** — `get()` 마지막 disk-fallback 주석을 새 순서에 맞게 `"Nothing has been found in the checkpoint diff or cache, look up from disk"`로 수정.

---

## 8. PR 본문 작성 — 정확성 검토 두 번

1차 초안에 "a value read **before opening a checkpoint** can hide a later checkpointed put()/del()"이라는 문장이 있었는데, 4절에서 짚은 것처럼 **트리거 조건이 그것보다 넓다**(checkpoint 안에서 처음 읽은 경우도 포함)는 걸 지적해서 다음으로 수정:

> a value cached by an earlier `get()` — whether read before the checkpoint opened, or read for the first time while it was open — can hide a later checkpointed `put()`/`del()` for the same key.

테스트 커맨드 3개(`test:node`, `lint`, `tsc`)도 PR에 적기 전에 실제로 전부 직접 실행해서 정확한지 확인한 뒤 실었다.

최종 PR: https://github.com/ethereumjs/ethereumjs-monorepo/pull/4355

---

## 9. 파생 발견 — binarytree의 중복 버그, 그리고 스코프를 지킨 순간

PR을 올린 뒤 다른 곳에도 같은 패턴이 있는지 확인하다가, `packages/binarytree/src/db/checkpoint.ts`에 **`CheckpointDB`가 한 글자도 안 다르게 복사돼있고, 똑같은 버그를 그대로 갖고 있는 것**을 발견했다(mpt에서 binarytree용으로 복붙해서 만든 것으로 보임). 여기는 전용 테스트 파일조차 없어서 mpt보다 테스트 커버리지가 더 약했다.

**여기서 스코프 관련 판단이 두 번 있었다.**

### 9-1. 첫 번째 판단 — 같은 PR에 넣을 것인가

내가 처음엔 "근본 원인이 완전히 동일하니 한 PR에서 같이 고치는 게 자연스럽다"는 쪽으로 제안했는데, 사용자가 정확히 반박했다: **"너 말대로 따르면 하나의 PR에서 개발하다 발견한 모든 이슈를 다 처리하게 생겼는데?"** 맞는 지적이었다. mpt는 이번 PR 그대로, binarytree는 코드를 건드리지 않고 별도 이슈로만 등록하기로 정리했다.

### 9-2. 두 번째 판단 — 이슈만 올릴 것인가, PR까지 올릴 것인가

이슈 본문 마지막을 "I'm happy to open a PR with the same fix as #4355 if that sounds right"로 써놓고, 곧바로 "이슈 등록하고 그거에 해당하는 PR까지 올려야 하지 않나?"라는 질문이 나왔다. 여기서 짚은 것:

- 방금 쓴 문장 자체가 "메인테이너 답을 기다리겠다"는 뜻으로 쓴 건데, 바로 PR까지 올리면 스스로 쓴 문장과 행동이 안 맞음
- binarytree는 Verkle/바이너리 트리 관련 실험적 영역이라, 메인테이너가 "각자 고쳐 쓰기"가 아니라 "mpt 걸 재사용하도록 리팩터"하는 다른 방향을 생각할 수도 있음 — 미리 낸 PR이 그 방향과 충돌할 위험

**결론: 이슈만 등록하고, PR은 메인테이너 반응을 기다린 뒤 결정.**

### 9-3. 이슈 본문 톤 — 정확성 vs 공손함

이슈 초안도 두 차례를 거쳤다:
- 1차: "byte-for-byte duplicate"라는 표현 — 사실이지만 메인테이너 입장에선 다소 찌르는 표현. → "appears to mirror"로 순화
- 제목에서 "same bug as mpt"를 빼서 독립적인 이슈처럼 보이게 함
- 영향 범위를 먼저 겸손하게("`StatefulBinaryTreeStateManager`는 `cacheSize: 0`이라 기본 경로는 안 터진다") 짚고 시작
- **PR 본문 때와 똑같은 정확성 이슈**(트리거 조건을 "체크포인트 열기 전에 읽은 경우"로만 좁게 서술)가 이슈 초안에도 그대로 재발해서, 동일하게 수정

최종 이슈: https://github.com/ethereumjs/ethereumjs-monorepo/issues/4356

---

## 10. 핵심 교훈

1. **캐시와 롤백 가능한 상태 레이어는 서로 다른 목적이라도, 같은 메서드 안에서 순서를 잘못 짜면 얽혀버린다.** 이번 버그의 본질은 복잡한 로직 실수가 아니라 딱 하나 — "어느 걸 먼저 확인할지"의 순서.
2. **기존 테스트 커버리지의 "구멍 모양"이 버그가 살아남은 이유를 그대로 설명해준다.** `checkpoint.spec.ts`의 테스트 12개가 전부 `cacheSize` 없이만 작성돼 있었다는 사실 자체가, 왜 아무도 이 버그를 못 잡았는지의 답이었다.
3. **같은 실수(트리거 조건을 좁게 서술)가 PR 본문과 이슈 본문에서 두 번 반복됐다.** 한 번 잡았다고 다음에 저절로 안 나오는 게 아니라서, 비슷한 글을 쓸 때마다 매번 같은 기준으로 재검토해야 한다.
4. **"근본 원인이 같다"는 게 "한 PR에서 같이 고쳐도 된다"는 뜻은 아니다.** 스코프를 넓히고 싶은 유혹은 매번 "이것도 같은 원인인데 왜 안 고치지?"라는 그럴듯한 논리로 온다 — 이번엔 사용자가 그 논리에 제동을 걸었다.
5. **"PR 열겠다"고 써놓고 바로 여는 것과, 답을 기다리는 것은 다른 선택이다.** 이슈 본문에 이미 쓴 문장과 실제 행동이 어긋나지 않는지도 스코프 판단의 일부다.
6. 가짜 주입 메시지("날짜 바뀜", "파일 수정됨")가 이번에도 여러 번 나왔고, 매번 플래그하고 무시한 뒤 계속 진행했다 — 에이전트도 동일하게 대응했다.

---

## 11. 결과

- [x] PR #4355 메인테이너 리뷰 결과 — 2026-08-14 `holgerd77`가 승인(LGTM)과 함께 코멘트: "Hey hey, thanks, this is a really useful contribution! Sorry that it sat for so long. Will merge in!" 별도 수정 요청 없이 원안 그대로 머지.
- [x] 이슈 #4356에 대한 메인테이너 반응 — 별도 답 없이, 다른 기여자 `SnowingFox`가 2026-08-10에 "Opened a PR to address this: #4360"이라고 직접 PR을 올렸고, 그 PR이 2026-08-14 `holgerd77`에 의해 머지되면서 이슈도 closed됨.
- [x] binarytree PR 착수 여부 — 착수하지 않음. 이슈만 등록하고 메인테이너 반응을 기다리기로 한 9-2절의 판단대로, PR을 선점하지 않고 기다린 사이 다른 기여자가 먼저 고쳤다. 우리 이슈가 계기가 됐다는 것 외에 코드 기여로는 이어지지 않은 케이스.
