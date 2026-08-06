# [Redis OSS 기여] redis/node-redis - Sentinel 마스터 연결 실패 시 무한 대기 문제 수정 (PR #3329 → 재작업 PR #3331) — 🟢 PR #3331 Merged

> 오픈소스 기여 기록 및 학습 노트. 이 문서는 이슈 #3066 하나에 대한 기여 과정 전체(첫 PR #3329의 제출·클로즈 → 원인 재분석 → 재작업 PR #3331)를 시간순으로 다룬다. **PR #3329는 Closed(머지 안 됨), 재작업한 PR #3331이 2026-07-13 Merged.**

---

## 1. 개요

### 1.1 프로젝트 소개

- **저장소**: https://github.com/redis/node-redis ([[node-redis-pr3321-cluster-ready-guard]], [[node-redis-pr3325-cluster-docker-networking]]와 동일 저장소)
- **건드린 부분**: `packages/client/lib/sentinel/index.ts` — Sentinel 모드에서 연결(및 재연결)을 담당하는 로직 전반. 첫 PR은 재시도 루프의 로깅 한 줄을 고쳤고, 재작업 PR은 마스터/레플리카 연결 실패를 실제로 관찰 가능하게 만드는 구조적 수정이다.

### 1.2 이슈 (#3066)

**제목**: "createSentinel promise doesn't resolve" (구체적으로는 "Client hangs on socket error during connection initialization"으로도 언급됨). 리포터(alsarakbi)의 시나리오: Sentinel 접속은 성공하는데, Sentinel이 알려준 **마스터로의 접속은 실패**(TLS 키 설정 문제)하는 상황에서 `await sentinel.connect()`가 **에러도 타임아웃도 없이 영원히 멈춘다.** 리포터는 `sentinel.on("error", ...)` 리스너까지 달아놨는데 거기에도 아무것도 안 들어온다고 명시했고, 문제 위치(소스의 `transform()` 마스터 연결 블록)까지 정확히 짚어놓았다.

### 1.3 관련 링크

| 구분                     | 링크                                                                  | 상태                                         |
| ------------------------ | --------------------------------------------------------------------- | -------------------------------------------- |
| 원본 저장소              | https://github.com/redis/node-redis                                   | —                                            |
| 내 Fork                  | https://github.com/GiHoon1123/node-redis                              | —                                            |
| 이슈 (#3066)             | https://github.com/redis/node-redis/issues/3066                       | Open (재작업 PR #3331이 Fixes 대상)          |
| **첫 PR (#3329)**        | https://github.com/redis/node-redis/pull/3329                         | **Closed** (2026-07-10, 메인테이너가 클로즈) |
| 메인테이너 클로징 코멘트 | https://github.com/redis/node-redis/pull/3329#issuecomment-4933984889 | —                                            |
| **재작업 PR (#3331)**    | https://github.com/redis/node-redis/pull/3331                         | Open (2026-07-11 제출)                       |
| 첫 PR 작업 브랜치        | `fix-issue-3066-sentinel-master-unreachable-silent-retry`             | —                                            |
| 첫 PR 커밋               | `e6df6a2989`                                                          | —                                            |
| 재작업 PR 브랜치         | `fix-3066-sentinel-unreachable-master-hang`                           | —                                            |
| 재작업 PR 커밋           | `5023b48c54`                                                          | —                                            |

### 1.4 한 줄 요약

> **1차 시도(#3329)**: Sentinel 모드에서 마스터로 연결하는 재시도 루프가 실패할 때마다 `console.log`만 하고 아무 이벤트도 안 냈다고 이해하고, `console.log(e)`를 `this.emit('error', err)`로 바꿨다. 메인테이너는 이 수정이 **실행조차 되지 않는 지점**을 고친 것이라며 클로즈하고, 진짜 원인(마스터 connect promise가 영원히 pending 상태로 멈춰있음)을 짚어줬다.
>
> **2차 시도(#3331)**: 마스터/레플리카 클라이언트의 `client.connect()`를 감싸는 `#connectOrFail` 함수를 만들어, 소켓 레벨에서 이벤트로만 알려지던 연결 실패를 promise의 reject로 바꿔줬다. 이렇게 하면 `Promise.all`이 실패를 즉시 감지해서 재시도 루프가 실제로 동작할 수 있게 되고, 최종 포기 시 백그라운드에 남아있는 클라이언트를 정리하는 로직도 추가했다.

---

## 2. 배경 지식

### 2.1 Sentinel 모드 vs Cluster 모드

**Sentinel**: 마스터 1개(+레플리카)로 된 **하나의 데이터셋**을 고가용성(HA)하게 만드는 방식. 별도의 Sentinel 프로세스들이 마스터/레플리카를 감시하다가, 마스터가 죽으면 레플리카를 자동으로 새 마스터로 승격시킨다(자동 페일오버). 데이터는 안 쪼개진다.

**Cluster**([[node-redis-pr3321-cluster-ready-guard]] 참고): 데이터를 여러 마스터로 **쪼개서**(샤딩) 저장하는 방식. 목적이 다르다 — Sentinel은 "하나의 데이터셋을 안 죽게", Cluster는 "데이터를 나눠서 더 크게".

`createSentinel()`은 내부적으로 "지금 마스터가 어디야?"를 Sentinel들한테 물어보고, 그 결과에 따라 실제 마스터 클라이언트를 연결하는 로직을 갖고 있다 — 이 로직이 이번 버그의 위치다.

### 2.2 `#connect()`의 재시도 루프 구조 (원본)

```ts
async #connect() {
  let count = 0;
  while (true) {
    count += 1;
    if (this.#destroy) { return; }
    try {
      await this.transform(this.analyze(await this.observe()));
      return; // 성공
    } catch (e) {
      const err = e as Error;
      if (!this.#isReady && count > this.#maxCommandRediscovers) {
        throw e; // 16번(기본값) 넘게 실패해야 여기로 옴
      }
      if (err.message !== 'no valid master node') {
        console.log(e); // <- 1차 수정 대상 지점
      }
      await setTimeout(1000);
    }
  }
}
```

- `observe()`: Sentinel들한테 SENTINEL 커맨드로 물어봐서 지금 마스터가 어딘지 알아냄
- `analyze()`: 그 응답을 분석해서 유효한 마스터 정보로 변환 (마스터가 s_down/o_down으로 표시돼 있으면 `"no valid master node"` 에러를 던짐 — 이건 정상 페일오버 도중에도 항상 잠깐 발생하는 흔한 전이 상태)
- `transform()`: 실제로 마스터 주소에 클라이언트를 연결시킴 — **이슈의 시나리오에서 실패하는 지점이 여기**

`observe()`/`analyze()`는 문제없이 성공하고, `transform()`이 실제 연결을 시도하다가 실패하는 경우가 이번 이슈의 핵심 시나리오다.

---

## 3. #3066 — 첫 시도(PR #3329)의 (잘못된) 진단과 수정

### 3.1 당시의 이해

"마스터 연결 실패가 **콘솔에만 찍히고 조용히 삼켜지는** 것이 문제다"라고 이해했다. 조사 중에 2.2절 코드의 catch에서 발견한 `console.log(e)`가 근거였다.

### 3.2 당시의 수정 (한 줄 + 테스트)

```diff
  if (err.message !== 'no valid master node') {
-   console.log(e);
+   this.emit('error', err);
  }
```

- `console.log`를 `emit('error')`로 변경
- `nodeAddressMap`으로 마스터만 unreachable 주소로 remap하는 회귀 테스트 추가, `once(sentinel, 'error')`로 이벤트 발생을 검증

**왜 안전하다고 판단했나**: `RedisSentinel`(공개 클래스)의 생성자가 `connect()`가 호출되기 전부터 이미 `this.#internal.on('error', err => this.emit('error', err))`를 걸어두기 때문에, `RedisSentinelInternal` 인스턴스의 `'error'` emit은 항상 리스너가 최소 1개 있어서 크래시 위험이 없다고 봤다. 이 정확히 같은 패턴이 같은 클래스 안 다른 곳(`observe()`의 sentinel별 실패, `PubSubProxy` 콜백)에서 이미 쓰이고 있다는 것도 확인했다.

### 3.3 왜 틀렸나 — 문제 위치 자체가 틀림 (재작업 착수 후 밝혀진 것)

**`console.log(e)`가 나쁜 코드인 건 사실이지만, #3066 시나리오에서는 그 코드에 실행이 도달하지 않는다.**

- 그 catch는 `observe()`/`analyze()` — 즉 **sentinel discovery가 실패**했을 때 도달하는 곳
- #3066은 discovery는 성공하고 **마스터 접속만 실패**하는 경우 → 실패 지점은 `transform()` 내부
- `transform()`의 마스터 connect promise는 (5장에서 설명할 이유로) **reject되지 않고 영원히 pending 상태** → throw 자체가 없음 → catch 미도달

즉 "연결이 안 된다"는 신고를 받고 정작 고친 곳은 **연결이 실패했을 때 실행되는 로그 출력 한 줄**이었는데, 실제 시나리오에서는 그 코드 자체가 한 번도 실행되지 않는다 — 수정 위치가 실제 실행 경로 밖에 있었던 것.

---

## 4. 정적 분석만으로 진행한 1차 리뷰의 한계

1차 PR(#3329)을 제출하기 전, 세 가지 관점으로 자체 리뷰를 했었다:

1. **API 안전성**: `'error'` 리스너 0개 상태에서 emit하면 Node가 크래시시키는 게 맞지만, 이 클래스는 이미 그 리스크를 감수하는 패턴을 다른 곳에서도 쓰고 있어서 새로운 리스크가 아니라고 판단.
2. **노이즈/플러딩**: 정상 페일오버는 보통 몇 초 안에 끝나서 재시도 횟수 자체가 적다고 판단.
3. **테스트 품질**: 스코프가 적절하고, `destroy()`의 안전한 정리를 코드로 확인. 유일한 한계는 새 테스트를 로컬에서 끝까지 실행 못 해본 것 — PR 본문에 정직하게 명시.

이 세 가지 리뷰는 전부 **코드를 실제로 실행해보지 않고, 코드를 읽기만 하면서** 진행됐다. 그 결과 "수정 위치 자체가 실행되지 않는 경로에 있다"는 가장 근본적인 문제를 못 잡아냈다 — 코드를 읽고 논리적으로 맞는지 따지는 것과, 실제로 그 코드에 실행이 도달하는지 확인하는 것은 다른 검증이라는 걸 보여주는 사례다.

---

## 5. 검증 시도 (1차 PR #3329) — 로컬 실행이 막혀서 거의 1시간을 쓴 환경 디버깅

**목표**: `nodeAddressMap`으로 마스터 주소만 아무도 안 듣는 포트로 remap해서, Sentinel 자체는 멀쩡한 채로 마스터만 도달 불가능한 상황을 실제 docker Sentinel 토폴로지로 재현.

**막힌 지점**: 이 저장소의 `SentinelFramework`(docker 기반 테스트 하니스)로 토폴로지를 띄우려 했는데, `beforeEach` 훅의 15초 타임아웃 안에 컨테이너 스폰이 끝나지 않고 계속 실패함.

**시도한 것들** (순서대로):

1. 독립 `tsx` 스크립트로 직접 실행 → `@redis/test-utils`의 docker 헬퍼가 Mocha 전역 `after(...)`를 직접 호출해서, Mocha 안에서만 동작한다는 걸 발견 → `.spec.ts` 파일로 재작성
2. 타임아웃을 2분 → 5분으로 늘려봄 → 여전히 5개 중 1개 컨테이너만 뜨고 멈춤
3. 도커 리소스 경합을 의심 → Postgres 빼고 다른 프로젝트용 컨테이너를 전부 `docker stop` → **변화 없음**
4. 파다가 **완전히 별개의 진짜 버그를 발견**: `sentinel/test-util.ts`의 `spawnRedisSentinelNodes()`가 마스터/레플리카를 스폰할 때 `DEBUG_MODE_ARGS` 상수 배열을 **스프레드 없이 그대로** 넘김. 근데 이 값을 캐싱하는 `spawnRedisServer()`가 배열을 **레퍼런스로** 캐시 키를 삼아서, 같은 레퍼런스가 들어오면 첫 번째로 만든 컨테이너를 그대로 재사용해버림. 코드베이스의 다른 모든 호출부는 `[...DEBUG_MODE_ARGS]`로 스프레드해서 새 배열을 넘기는데, 이 파일만 빠뜨림.
5. 이 스프레드 누락을 로컬에서 임시로 고치고, 기존의 **손 안 댄** `'stop master'` 테스트를 재실행 → **여전히 똑같은 15초 타임아웃** → 이 가설은 틀렸다는 게 확인됨. 변경분은 `git checkout --`으로 되돌림.

**최종 결론(당시)**: 진짜 느려지는 원인은 끝내 못 찾음. `DEBUG_MODE_ARGS` 스프레드 누락 자체는 여전히 진짜 버그로 보이지만(다른 모든 호출부의 컨벤션과 명백히 다름), 이번 증상의 원인은 아니었음.

**정리(2026-07-11 시점)**: 이 macOS 도커 실행 불가 문제 자체는, 별개 PR(#3325, [[node-redis-pr3325-cluster-docker-networking]])의 조사와 메인테이너 코멘트를 통해 **"Docker Desktop의 host networking 토글을 켜면 해결된다"**는 게 나중에 밝혀졌다. 즉 위 4~5번에서 찾던 원인은 애초에 이 문제가 아니었고, 진짜 원인은 훨씬 단순한 환경 설정(Docker Desktop 버전 + 설정)이었다 — 7장 참고.

`tsc --build` / eslint는 클린, 도커 없이 도는 기존 sentinel 유닛 테스트도 전부 통과 확인. 다만 **새 회귀 테스트 자체를 로컬에서 끝까지 실행해서 통과를 확인하는 것은 결국 못 함** — PR 본문에 이 한계를 명시하고 CI에 맡긴 채로 #3329를 제출.

---

## 6. 메인테이너(nkaradzhov)의 #3329 클로징 코멘트 — 3가지 지적

원문 요지 (2026-07-10):

1. **수정 위치가 문제 발생 지점이 아님.** #3066에서 마스터 클라이언트는 기본 `reconnectStrategy`로 무한 재시도하므로 `client.connect()`가 영원히 완료되지 않고, `transform()`이 `Promise.all(...)`에서 무기한 블록됨. #3329가 고친 `#connect()`의 catch는 **실행조차 되지 않음**. 이슈가 "~16초 재시도 루프"가 아니라 "영구 hang"으로 보고된 이유가 바로 이것.
2. **추가한 테스트도 통과 불가.** `passthroughClientErrorEvents` 기본값이 `false`라서 마스터 클라이언트 연결 에러는 `'error'`가 아니라 `'client-error'`로 emit됨 → 테스트의 `once(sentinel, 'error')`는 영원히 resolve되지 않음.
3. **새 버그 유발 가능.** 가드 없는 `this.emit('error', err)`를 재시도 루프에 넣으면, `'error'` 리스너가 없는 앱에서 Node의 `ERR_UNHANDLED_ERROR`가 던져져 루프가 죽고, 백그라운드 scan 타이머에서 `#connect`가 돌 때는 unhandled rejection까지 발생.

메인테이너가 제시한 올바른 방향 (재작업의 설계 지침):

> "The real fix for #3066 is **making initial master connection failure observable to `#connect()`** — e.g. **awaiting the master connect promises with failure/timeout awareness** so **`connect()` rejects instead of hanging**. Closing as-is, but **happy to review that approach**."

요구사항 분해:

- ① 초기 마스터 연결 실패를 `#connect()`(재시도 루프)가 **관찰 가능**하게 만들 것
- ② 마스터 connect promise를 **실패 또는 타임아웃을 인지하는 상태로** await할 것
- ③ `connect()`가 hang 대신 **reject**하게 할 것

("failure/timeout"의 슬래시는 "실패 또는 타임아웃 — 두 종류의 신호를 인지하라"는 뜻이지 "타이머 기반으로 구현하라"는 설계 지시가 아니다. 문장에 "e.g."도 붙어 있다. 재작업은 둘 다 인지한다 — 9장 참고.)

---

## 7. 재작업 — 근본 원인 재분석

### 7.1 세 가지 코드가 맞물린 결과

**(1) 소켓의 기본 재연결 전략 = 무한 재시도** (`packages/client/lib/client/socket.ts:456`)

```ts
defaultReconnectStrategy(retries: number, cause: unknown) {
  if (cause instanceof SocketTimeoutError) return false;  // 이 경우만 포기
  const jitter = Math.floor(Math.random() * 200);
  const delay = Math.min(Math.pow(2, retries) * 50, 2000);
  return delay + jitter;                                   // 나머지는 전부 재시도
}
```

TLS 핸드셰이크 실패, ECONNREFUSED, 방화벽 등은 `SocketTimeoutError`가 아니므로 전부 무한 재시도 대상이다.

**(2) 소켓 connect 루프는 promise를 끝내지 않은 채 이벤트만 내보냄** (`socket.ts:244`)

```ts
async #connect(): Promise<void> {
  let retries = 0;
  do {
    try {
      this.#socket = await this.#createSocket();   // 연결 시도
      ...
      this.#isReady = true;                        // 성공 → 루프 탈출 → promise 성공으로 확정
    } catch (err) {
      const retryIn = this.#shouldReconnect(retries++, err);
      if (typeof retryIn !== 'number') throw retryIn;  // ← 실패로 확정되는 유일한 출구
      this.emit('error', err);                     // ← 매 실패마다 이벤트 발행
      await setTimeout(retryIn);
      this.emit('reconnecting');
    }
  } while (this.#isOpen && !this.#isReady);
}
```

기본 전략이 항상 숫자(재시도 간격)를 반환하므로 `throw retryIn` 출구에 절대 도달하지 않는다. **`'error'` 이벤트는 매번 발행되지만, 이 함수가 반환하는 promise는 성공도 실패도 아닌 pending 상태로 계속 남는다.**

**(3) `transform()`은 그 promise들을 `Promise.all`로 무기한 대기** (`sentinel/index.ts`)

```ts
for (let i = 0; i < this.#masterPoolSize; i++) {
  ...
  masterPromises.push(client.connect());   // ← 영원히 pending인 promise를 배열에 담음
}
...
await Promise.all(promises);               // ← 전부 끝나기를 기다림 → 영원히 대기
```

### 7.2 세 가지를 합치면

`client.connect()`가 pending 상태에서 안 벗어나니, 그걸 기다리는 `Promise.all`도 pending에서 안 벗어난다. `Promise.all`이 안 끝나니, 그걸 `await`하는 `transform()`도 안 끝난다. `transform()`이 예외를 던지지 않으니, 2.2절의 재시도/포기 루프에 있는 `catch` 블록에는 애초에 도달할 수가 없다 — **재시도 루프 자체는 멀쩡했지만, 그 루프에 들어갈 신호(예외)가 발생한 적이 없었다.** 이게 원래 이슈가 "N초짜리 재시도"가 아니라 "타임아웃도 없이 영원히 멈춤"으로 보고된 이유다.

### 7.3 왜 sentinel 클라이언트는 같은 문제를 안 겪었나

```ts
#createClient(node, clientOptions, reconnectStrategy?: false) {
  ...
  socket: {
    ...
    ...(reconnectStrategy !== undefined && { reconnectStrategy })  // false 주입
  }
}
```

sentinel discovery용 클라이언트는 `#createClient(node, opts, false)`로 만들어 **`reconnectStrategy: false`가 강제**된다 → 접속 실패 시 `connect()`가 즉시 reject → catch 도달 → 재시도 루프 정상 작동. **마스터/레플리카 클라이언트만 이 실패 신호가 없었다.** 즉 "초기 연결 단계 클라이언트는 실패를 즉시 알리고, 재시도는 바깥 루프가 담당한다"는 패턴이 이 코드베이스에 이미 있었고, 재작업은 마스터/레플리카를 그 패턴에 맞춘 것이다.

### 7.4 핵심 개념 — 이벤트와 promise는 서로 다른 통로

| 통로                          | 의미                                                        | 이번 케이스              |
| ----------------------------- | ------------------------------------------------------------ | ------------------------ |
| 이벤트 (`emit('error')`)      | "방금 시도 하나가 실패했다"는 알림. 듣는 사람이 없으면 그냥 사라짐 | 매 실패마다 발행됨 (O)   |
| Promise 확정 (resolve/reject) | `await`하는 쪽으로 전달되는 최종 결과                          | 영원히 발생하지 않음 (X) |

- promise는 호출 시점에 이미 시작된 작업의 결과를 나중에 알려주는 객체다. `client.connect()`를 부르는 순간 연결 시도는 이미 시작되고, 반환된 promise는 그 결과를 추적하는 용도다.
- `Promise.all`은 배열에 담긴 모든 promise가 확정되기를 기다린다. 하나라도 실패하면 즉시 전체가 실패로 확정되고(fail-fast), 하나라도 계속 pending이면 `Promise.all` 자신도 계속 pending이다.
- **reject는 알림일 뿐 취소가 아니다.** promise를 실패로 확정해도, 그 뒤에서 실제로 돌고 있던 작업(이 경우 소켓의 재시도 루프)은 멈추지 않고 계속 돈다 — 이 문제는 별도로 처리해야 한다 (10장).

---

## 8. `connect`라는 이름이 여러 계층에 있어 생기는 혼동 정리

`connect`라는 메서드 이름이 5개 계층에 걸쳐 존재해서, 어느 계층 얘기인지 구분이 필요하다.

```
[계층1] RedisSentinel.connect()              사용자가 직접 호출하는 공개 API. 계층2로 위임만 함
  └─[계층2] RedisSentinelInternal.connect()   연결 시도 전체를 관리. ★catch에 destroy를 추가한 지점
      └─[계층3] RedisSentinelInternal.#connect()  1초 간격, 최대 17회 재시도하는 루프. transform 호출
          └─ transform()
              └─[계층4] RedisClient.connect()     서버 한 대로의 연결. ★#connectOrFail로 감싼 지점
                  └─[계층5] RedisSocket.connect()  TCP 연결 + 무한 재시도 + error 이벤트 발행 (문제의 근원지, 수정하지 않음)
```

| 계층                        | 역할                         | 재작업에서의 수정                                 |
| --------------------------- | ---------------------------- | ------------------------------------------------- |
| 계층1 RedisSentinel.connect | 위임                         | —                                                   |
| 계층2 Internal.connect      | 성공/실패 마무리             | catch에 `this.destroy()` 추가, scan 타이머 조건 추가 |
| 계층3 Internal.#connect     | 재시도 루프                 | `console.log` 삭제                                 |
| 계층4 RedisClient.connect   | 노드 한 대로의 연결          | `#connectOrFail`로 감쌈                            |
| 계층5 RedisSocket.connect   | TCP 연결, 무한 재시도, error 이벤트 발행 | 수정하지 않음                                       |

### `transform()`의 역할

| 단계 | 메서드        | 하는 일                                                                             |
| ---- | ------------- | ------------------------------------------------------------------------------------ |
| 조사 | `observe()`   | sentinel들에게 "마스터가 누구고 레플리카는 뭐냐"고 질의                              |
| 분석 | `analyze()`   | 그 응답과 현재 연결 상태를 비교해서 무엇을 바꿔야 하는지 계산                        |
| 적용 | `transform()` | ① 기존 클라이언트 정리 → ② 새 클라이언트 생성 및 연결 시작 → ③ `Promise.all`로 완료 대기 |

"마스터 풀 클라이언트": `RedisSentinel`은 마스터 한 대에 커넥션을 `masterPoolSize`개(기본 1) 열어서 풀로 관리한다 (블로킹 명령/WATCH 트랜잭션이 커넥션을 점유하기 때문). `getClientLease()`/`releaseClientLease()`로 대여·반납한다. 레플리카도 `replicaPoolSize`(기본 0)로 동일한 구조다.

---

## 9. 재작업 수정 내용 — 코드 레벨 설명

### 9.1 `#connectOrFail` 함수

```ts
#connectOrFail(client: RedisClientType<...>) {
  return new Promise<unknown>((resolve, reject) => {
    const onError = (err: Error) => reject(err);
    client.once('error', onError);
    client.connect()
      .then(resolve, reject)
      .finally(() => client.off('error', onError));
  });
}
```

**이 함수의 역할**: 원래는 그냥 흘려보내지던 `'error'` 이벤트를, promise의 reject로 바꿔주는 어댑터다. 두 가지 완료 조건 중 먼저 발생하는 쪽이 이 promise의 결과를 결정한다:

- 조건 A: `client`에서 첫 `'error'` 이벤트가 발생함 → 실패로 확정
- 조건 B: `client.connect()`가 성공함 → 성공으로 확정

동작 분기:

| 시나리오        | 결과                                                                                                 |
| --------------- | ------------------------------------------------------------------------------------------------------ |
| 연결 성공       | 조건 B가 먼저 발생 → 기존과 동일하게 성공으로 확정. `finally`가 아직 발화하지 않은 `'error'` 리스너을 제거 (안 떼면 운영 중에 발생하는 에러까지 이 콜백이 잡아버림) |
| 연결 실패       | 조건 A가 먼저 발생(첫 실패의 `'error'` 이벤트) → 실패로 확정. `once`로 등록했으므로 리스너는 자동 제거됨 |
| 뒤늦게 오는 결과 | promise는 먼저 발생한 조건만 반영하고, 이미 확정된 뒤에 오는 resolve/reject 시도는 무시됨 — 안전함        |

핵심은 계층5의 재시도 자체를 멈추는 게 아니라, **그 재시도 루프가 내는 첫 실패 신호만 붙잡아서 promise를 확정시킨다**는 점이다. 여전히 백그라운드에서 재시도 중인 클라이언트를 실제로 정리하는 건 별도 로직이다 (10장).

### 9.2 마스터/레플리카에 적용 (각 한 줄)

```diff
  this.#masterClients.push(client);
- masterPromises.push(client.connect());
+ masterPromises.push(this.#connectOrFail(client));
```

```diff
  this.#replicaClients.push(client);
- promises.push(client.connect());
+ promises.push(this.#connectOrFail(client));
```

`Promise.all`에 들어가는 promise가 "영원히 pending일 수 있는 것"에서 "첫 에러에 즉시 reject되는 것"으로 바뀌면서, `Promise.all`의 fail-fast 동작이 실제로 작동하게 된다 → `transform`이 예외를 던짐 → 계층3의 catch에 도달.

레플리카도 포함한 이유: 기존 코드에서도 레플리카가 도달 불가능하면 같은 `Promise.all`에서 동일하게 멈춰있었다. 마스터만 고치면 같은 종류의 문제가 절반만 남는다.

### 9.3 계층2 `connect()` — 최종 실패 시 완전한 정리

```diff
  try {
    this.#isOpen = true;
    this.#connectPromise = this.#connect();
    await this.#connectPromise;
    this.#isReady = true;
+ } catch (err) {
+   this.#connectPromise = undefined;
+   await this.destroy();
+   throw err;
  } finally {
    this.#connectPromise = undefined;
-   if (this.#scanInterval > 0) {
+   if (this.#isReady && this.#scanInterval > 0) {
      this.#scanTimer = setInterval(this.#reset.bind(this), this.#scanInterval);
    }
  }
```

- **catch에서 `#connectPromise = undefined`를 `destroy()`보다 먼저 하는 이유**: `destroy()`는 내부적으로 `await this.#connectPromise`를 하는데, 그 promise가 방금 실패한 그 promise 그대로라면 `destroy()` 자체가 그 에러 때문에 실패해서 정리가 안 된다.
- **`throw err`**: 정리는 하되 에러 자체는 삼키지 않고 호출한 쪽에 그대로 전달한다.
- **`finally`의 `#isReady` 조건 추가**: 기존 코드는 연결이 **실패한 뒤에도** scan 타이머를 켰다. 준비되지 않은 상태에서는 `#reset()`이 아무 일도 안 하니 실질적인 오작동은 없지만, 살아있는 `setInterval`이 이벤트 루프를 붙잡아서 프로세스가 종료되지 않는 잠재적 문제가 있었다 — "연결이 성공했을 때만 켠다"로 수정했다.

### 9.4 계층3의 `console.log(e)` 삭제

```diff
  if (!this.#isReady && count > this.#maxCommandRediscovers) {
    throw e;
  }
- if (err.message !== 'no valid master node') {
-   console.log(e);
- }
  await setTimeout(1000);
```

`emit('error')`로 바꾸지 않고 그냥 삭제한 이유: #3329에서 지적받았던 `ERR_UNHANDLED_ERROR` 문제를 아예 만들지 않기 위해서다. 실패 자체는 이제 `connect()`의 reject로 사용자에게 전달되므로, 이 지점에서 별도로 이벤트를 낼 필요가 없다.

---

## 10. 백그라운드에 남는 클라이언트 정리 (reject는 취소가 아니라는 문제의 해법)

`#connectOrFail`이 reject를 해도, 그 클라이언트 객체 자체는 백그라운드에서 재시도를 계속한다 (JS에서 promise를 reject하는 것은 결과를 알리는 것일 뿐, 그 뒤의 실제 작업을 멈추는 게 아니기 때문). 이걸 정리하는 지점은 두 군데다.

### 10.1 시도 사이 정리 (기존에 있던 코드)

```ts
if (analyzed.masterToOpen) {
  for (const client of this.#masterClients) {
    if (client.isOpen) client.destroy();
  }
  this.#masterClients = [];
  ...
}
```

이 로직은 원래부터 있었다. **왜 확실히 실행되는지 확인한 부분**: `analyze()`가 비교 기준으로 쓰는 `getMasterNode()`는 **연결에 성공한(`isReady`) 클라이언트가 있을 때만** 주소를 반환한다. 실패한 시도의 클라이언트들은 전부 준비되지 않은 상태이므로 "현재 마스터 없음"으로 판정되고, 이는 "마스터가 없다가 생긴다"는 변화로 인식되어 `masterToOpen`이 설정된다 → 정리와 재생성이 반드시 실행된다.

### 10.2 최종 포기 시 정리 (재작업에서 새로 추가)

17번째 시도에서 실패하면 "다음 시도"가 없으므로, 위 10.1의 정리 로직 자체가 호출되지 않는다 — 그러면 마지막 시도에서 만든 클라이언트가 아무도 정리해주지 않는 채로 계속 재시도만 하게 된다. 이걸 막기 위해 계층2의 catch에 `await this.destroy()`를 추가했다. (자체 검토 1차에서 발견한 문제.)

### 10.3 실제 타임라인 (마스터가 도달 불가능한 경우, 기본 설정 기준)

```
t=0ms      [1차 시도] transform: 클라이언트①을 생성하고 연결 시작
t=5ms      ① 첫 연결 실패 → error 이벤트 발생 → #connectOrFail이 reject → Promise.all 실패 → 계층3 catch 도달
t=5~1005ms 계층3이 1초 대기하는 동안, ①은 백그라운드에서 재시도 계속 (50ms → 100ms → …)
t=1005ms   [2차 시도] transform 재진입 → ① 정리(destroy) → ② 생성 및 연결 시작
…          (17차까지 반복. 어느 시점이든 정리 안 된 채 남아있는 건 직전 시도의 클라이언트 하나뿐)
t≈17s      최종 포기 → 계층2 catch: destroy()로 마지막 시도의 클라이언트 정리 → 사용자에게 에러 전달
```

---

## 11. 자체 검토 3차와 반영 내역 (재작업 PR)

### 1차

| 지적                                                                                            | 판정                                                                                                            | 조치                                    |
| ------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------- | ---------------------------------------- |
| (높음) 첫 에러에서 바깥 promise만 reject되고, client는 백그라운드에서 계속 재접속한다. 정리가 필요함 | 유효함 (최종 포기 케이스). 재시도 타이머가 이벤트 루프를 붙잡아 **reject 후에도 프로세스가 종료되지 않는** 실질적 문제 | 계층2 catch에 `await this.destroy()` 추가 |
| "connect()가 안 끝나서 finally의 리스너 제거도 실행 안 됨"                                        | 사실이 아님 — `once()`는 발화하는 순간 리스너가 자동으로 제거됨                                                   | 수정 불필요                              |
| (중간) 레플리카까지 적용하는 건 범위가 넓어진 것 아니냐                                          | 부분적으로만 맞음 — 기존에도 레플리카가 도달 불가능하면 동일하게 멈췄다. 마스터만 고치면 절반짜리 수정이 됨       | 유지하고 PR 본문에 근거 명시             |
| (낮음) package-lock.json에 무관한 변경이 섞여 있음                                                | 유효함 (`npm install` 실행 부산물)                                                                                | `git checkout`으로 원복                  |

### 2차

| 지적                                                                | 판정                                                                                                                                        | 조치                                      |
| -------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------- |
| (높음) 일시적인 오류까지 즉시 실패로 처리하면 `reconnectStrategy`의 의미가 훼손되는 것 아니냐 | 절반만 맞음 — 첫 에러로 확정되는 건 `sentinel.connect()` 전체의 실패가 아니라 **한 차례**의 실패다. 재시도 자체가 없어진 게 아니라, 위치가 클라이언트 내부에서 바깥 루프로 옮겨간 것 | 코드는 유지하고 PR 본문에 이 트레이드오프를 명시 |
| 테스트가 ECONNREFUSED 케이스만 검증한다. TLS/무응답 케이스도 추가해야 함 | TLS 케이스: 추가함. 무응답(블랙홀) 케이스: `'error'` 이벤트 자체가 발생하지 않아서 이번 수정의 범위 밖              | TLS 테스트 추가, 블랙홀 한계는 PR 본문에 명시 |

### 3차 (사용자 질문)

- "메인테이너가 실패 또는 **타임아웃**을 인지해야 한다고 했으니, 타이머 기반으로 구현해야 하는 것 아니냐?" → 슬래시는 "실패 또는 타임아웃, 두 종류의 신호를 인지하라"는 뜻이고, 첫-에러 방식은 이미 둘 다 인지한다: 즉시 발생하는 실패(ECONNREFUSED 등)는 밀리초 단위로, 타임아웃류는 `connectTimeout`(기본 5초) 만료 시 `ConnectionTimeoutError`가 동일하게 `'error'` 이벤트로 발생해서 같은 방식으로 잡힌다 → 현재 방식을 유지하고, PR 본문에 "타이머 기반 구현을 선호하면 그쪽으로 다시 작업할 의향이 있다"고 명시.

---

## 12. 재작업 테스트 설계

### 테스트 1 — 마스터가 도달 불가능한 경우 (즉시 실패 경로 + 정리 검증)

- `nodeAddressMap`으로 **마스터만** `127.0.0.1:1`(즉시 ECONNREFUSED가 나는 주소)로 remap → sentinel discovery는 계속 정상 = **#3066과 동일한 상황**
- `maxCommandRediscovers: 2`로 낮춰서 3차 시도로 단축 (약 3초)
- 검증 항목 3가지: `assert.rejects(sentinel.connect())` — 무한 대기 대신 reject된다는 계약 검증 / `sentinel.isOpen === false` — 정리가 됐는지 검증 / reject 후 1초 동안 `client-error` 이벤트가 0건 — 백그라운드에서 재시도하는 게 남아있지 않다는 증거

### 테스트 2 — TLS 설정 오류 (원 이슈 재현 + 타임아웃 경로)

- `nodeClientOptions: { socket: { tls: true } }`로 평문 Redis에 TLS 핸드셰이크를 시도 → 인증서 설정 없이 "sentinel은 붙고 마스터 TLS만 실패"하는 원본 이슈 상황을 재현
- `connectTimeout`(기본 5초) 만료 → `ConnectionTimeoutError` 발생 → 즉시 실패와는 다른 경로(타임아웃 후 에러)를 검증

---

## 13. 검증 환경 구축 (macOS 도커) 및 결과

- 문제: 테스트가 `docker run --network host`를 사용하는데, macOS의 구버전 Docker Desktop에서는 이게 동작하지 않아서 도커 기반 테스트 자체를 실행할 수 없었다 (5장에서 겪은 문제).
- 해결: 공식 CONTRIBUTING.md에 문서화된 방법 그대로 —
  1. Docker Desktop **4.27.2 → 4.81.0**으로 업데이트
  2. `settings-store.json`에 `HostNetworkingEnabled: true` 추가 후 재시작
  3. 사전 정리: 기존 컨테이너·이미지·볼륨 전량 삭제(39.6GB 회수), 필요한 이미지만 다시 받음
- 검증 결과:

| 항목                   | 결과                                           |
| ---------------------- | ---------------------------------------------- |
| `tsc --build`          | 클린                                           |
| eslint (변경 파일)     | 클린                                           |
| 도커 없이 성립하는 가정 확인 | 확인됨                                    |
| 회귀 테스트 2개        | 통과 (3.1초 / 17.0초)                          |
| **sentinel 전체 스위트** | **93 passing, 1 pending, 0 failing** (약 3분) |

1차 PR의 가장 큰 약점("로컬에서 검증을 못 했고 CI가 확인해달라")이, 재작업에서는 "메인테이너 본인이 문서화한 방법으로 macOS에서 전체 스위트를 통과시켰다"로 바뀐 지점이다.

---

## 14. PR #3331 본문에 담은 내용

1. #3329 클로징 리뷰의 제안을 구현한 후속 작업임을 명시
2. 실패/타임아웃 두 신호 모두 인지한다는 설명과, 타이머 기반 방식을 선호하면 재작업 의향이 있다는 언급
3. 첫-에러 방식의 근거: sentinel discovery 클라이언트가 이미 쓰고 있는 `reconnectStrategy: false` 패턴과의 대칭성
4. 레플리카까지 포함한 근거 (기존에도 동일하게 멈추는 문제가 있었음)
5. 최종 실패 시 정리 로직과 scan 타이머 조건 추가 설명
6. 무응답 서버(블랙홀) 케이스는 이번 수정의 범위 밖이라는 점 명시
7. `console.log`를 이벤트로 바꾸지 않고 삭제한 이유 (#3329에서 지적받은 문제 회피)
8. 체크리스트에 macOS 전체 스위트 93 passing 명시

---

## 15. 결과 및 회고 — ✅ 머지됨

### 15.1 현재 상태 (2026-07-14 기준)

- PR #3329 **Closed** (2026-07-10, 메인테이너)
- PR #3331 **Merged** (2026-07-13T09:10:48Z, merge commit `7764108b668835b413c51a5386d43e5efa345cb9`) — 메인테이너 `nkaradzhov`가 "CLGTM"으로 승인 후 머지. 커밋 1개(`5023b48c54`), 파일 2개(+99/−6)

### 15.2 핵심 교훈

1. **이슈의 증상 서술을 문자 그대로 읽어라.** 리포터는 "promise doesn't resolve", "no timeout no nothing"이라고 명시했고 문제 위치까지 짚어줬는데, 조사 중 발견한 나쁜 코드(`console.log`)에 꽂혀서 그것이 원인이라고 역추론했다. 나쁜 코드를 발견한 것과, 그 코드가 지금 보고된 버그의 원인이라는 것은 별개의 주장이다.
2. **"에러가 안 보인다"에는 두 가지 다른 상황이 있다.** (a) 에러는 발생하는데 전달 경로가 끊겨있는 경우 — 전달 경로를 고치면 된다. (b) 에러라는 판정 자체가 영원히 나지 않는(hang) 경우 — 판정이 나도록 새로 만들어야 한다. #3066은 (b)였는데, 1차 PR은 (a)에 해당하는 수정이었다.
3. **이벤트와 promise는 서로 다른 통로다.** 이벤트는 계속 발행되는데 promise는 확정되지 않는 상태가 있을 수 있다. 이 둘을 연결하는 것이 이번 수정의 핵심(`#connectOrFail`)이다.
4. **reject는 취소가 아니다.** promise를 실패로 확정해도 그 뒤에서 실제로 돌고 있던 작업은 멈추지 않는다. 자원을 실제로 정리하는 책임이 누구에게 있는지 항상 따져봐야 한다.
5. **수정으로 새로 도달 가능해진 실행 경로 끝까지 따라가라.** `transform`이 예외를 던질 수 있게 되자, 계층2가 실패로 끝날 수 있게 됐고, 그 결과 남아있는 자원(백그라운드에서 재시도 중인 클라이언트, scan 타이머)이라는 후속 문제가 연쇄적으로 드러났다.
6. **테스트는 구현 세부사항이 아니라 계약을 검증해야 한다.** `once(sentinel, 'error')`는 `passthroughClientErrorEvents`라는 내부 옵션의 기본값에 의존하는 취약한 검증이었다. `assert.rejects(connect())`는 "이 함수는 실패 시 reject한다"는 계약 자체를 검증하므로 구현이 바뀌어도 안정적이다.
7. **메인테이너의 클로징 코멘트는 사실상의 요구사항 명세서다.** "happy to review that approach"라는 문장은 설계 방향을 준 것이나 다름없었다 — 그 문장을 요구사항 세 가지로 분해해서 구현과 하나씩 대응시켰다.
8. **로컬 검증 불가는 그 자체로 포기할 이유가 아니라 별도로 해결해야 할 과제다.** 1차 PR은 "환경 문제로 못 돌려봤다"고 명시하고 CI에 맡겼다. 재작업에서는 Docker Desktop 업데이트와 host networking 설정으로 환경 자체를 고쳐서 전체 스위트를 직접 돌렸다.
9. **여러 관점으로 나눠 진행한 자체 검토가 실제로 문제를 찾아냈다.** 3차에 걸친 검토에서: 최종 포기 시 정리되지 않는 클라이언트(실질적 버그), package-lock 오염, TLS 테스트 누락을 각각 찾아냈다.
10. **예상되는 지적을 PR 본문에서 먼저 다루는 것이 신뢰 회복에 도움이 된다.** 리뷰어가 지적하기 전에 예상 쟁점을 먼저 설명하면, 검토가 충분히 이뤄졌다는 신호가 되고, 1차 PR이 클로즈되면서 깎인 신뢰를 회복하는 데도 도움이 된다.
11. **코드를 읽기만 하는 검토는 "수정 위치 자체가 틀렸다"는 근본적인 문제를 잡아내지 못할 수 있다.** 4장의 세 가지 관점 리뷰가 그 예시다. 실행 검증(재현)이 불가능한 상황에서는, 아무리 여러 번 검토해도 "이 코드에 실행이 실제로 도달하는가"라는 가장 기초적인 질문을 건너뛸 위험이 있다.

### 15.3 남은 항목

- [x] PR #3331에 대한 메인테이너 리뷰 — `nkaradzhov` "CLGTM" 승인
- [x] 머지 여부 — 2026-07-13T09:10:48Z 머지됨 (제출로부터 약 이틀)
- [ ] 발견했던 `DEBUG_MODE_ARGS` 스프레드 누락 버그의 별도 리포트 여부 — 미정

---

## 16. 개념 정리 (참고용)

- **Promise의 세 가지 상태**: 대기(pending) / 성공(fulfilled) / 실패(rejected). 성공이나 실패로 결과가 나는 것을 확정(settle)이라고 부른다. 아무도 확정시키지 않으면 영원히 대기 상태로 남는다 — 자동으로 타임아웃되는 것이 아니다.
- **promise의 의미**: `client.connect()`를 호출하는 시점에 실제 작업은 이미 시작된다. 반환되는 promise는 그 작업의 결과를 나중에 추적하기 위한 객체다.
- **`Promise.all`**: 배열에 담긴 모든 promise가 확정되기를 기다린다. 하나라도 실패하면 즉시 전체가 실패로 확정되고(fail-fast), 하나라도 계속 대기 상태면 자기 자신도 계속 대기 상태다. `Promise.all` 자체가 새로운 작업을 시작하거나 기존 작업을 취소하지는 않는다.
- **`new Promise((resolve, reject) => ...)`**: 성공과 실패, 두 가지 확정 방법을 함수 인자로 받는다. 둘 중 먼저 호출된 것만 유효하고, 이후의 호출은 무시된다 — 이 성질이 `#connectOrFail`에서 활용된 안전장치다.
- **`once`와 `on`의 차이**: `once`는 한 번 수신하면 리스너가 자동으로 제거된다. `on`은 계속 수신한다.
- **`try/catch/finally`**: `catch`는 예외가 던져져야 실행된다. `finally`는 성공/실패와 무관하게 항상 실행된다.
- **`setInterval`과 프로세스 종료**: 살아있는 타이머는 Node 프로세스가 종료되지 않게 붙잡아둔다.
- **`EventEmitter`의 `'error'` 이벤트 특수 동작**: 리스너가 하나도 없는 상태에서 `emit('error')`를 호출하면 Node가 `ERR_UNHANDLED_ERROR`를 던진다 — 1차 PR의 세 번째 문제였고, 재작업에서 `emit`을 다시 추가하지 않은 이유다.
