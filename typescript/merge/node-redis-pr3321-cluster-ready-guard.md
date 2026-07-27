# [Redis OSS 기여] redis/node-redis - 클러스터 연결 전 명령 실행 시 크래시 수정 (PR #3321) — ✅ MERGED

> 오픈소스 기여 기록 및 학습 노트. 이 문서는 PR #3321 하나만 다룬다. **이 PR은 2026-07-08에 실제로 머지됐다** — 우리 4개 PR 중 첫 머지 사례.

---

## 1. 개요

### 1.1 프로젝트 소개

- **라이브러리**: `redis` (npm 패키지명), 저장소명 `node-redis`
- **저장소**: https://github.com/redis/node-redis
- **누가 만드나**: **Redis Inc.** (Redis 데이터베이스 자체를 만드는 회사). GitHub organization `redis`가 `is_verified: true`로 인증돼 있고, 공식 블로그가 `redis.io`로 연결됨. README에 Redis University, Redis Cloud, Redis Launchpad, 채용 페이지까지 링크돼 있어서 **명백한 Redis Inc. 공식 프로젝트**.
- **뭐 하는 라이브러리인가**: Node.js용 Redis 클라이언트. Standalone Redis, Redis Cluster, Redis Sentinel을 전부 지원하는 모노레포 구조(`packages/client`, `packages/bloom`, `packages/json`, `packages/search` 등).

### 1.2 왜 이 저장소를 골랐나

이전에 확인했던 `confluent-kafka-javascript`와 같은 급의 **"진짜 공식" 라이브러리**라서 골랐다. 검증 방법도 동일했다:
- GitHub org `redis`의 `is_verified: true` 확인
- npm 패키지명이 그냥 `redis`(수식어 없는 가장 대표적인 이름)
- README가 Redis Inc.의 공식 자원들로 도배됨

참고로 `redis/ioredis`(다른 저장소, 이름이 비슷해서 헷갈리기 쉬움)는 **공식이 아닌 서드파티 클라이언트**다 — 혼동 주의.

### 1.3 관련 링크

| 구분 | 링크 |
|---|---|
| 원본 저장소 | https://github.com/redis/node-redis |
| 내 Fork | https://github.com/GiHoon1123/node-redis |
| 이슈 (#2704) | https://github.com/redis/node-redis/issues/2704 |
| PR (#3321) | https://github.com/redis/node-redis/pull/3321 |
| 작업 브랜치 | `fix/cluster-command-before-ready` |
| 머지 커밋 | `b393c0e7595ea8bf4c5d9ebb95822c18fd195b77` |

### 1.4 한 줄 요약

> `RedisCluster` 클라이언트가 클러스터 토폴로지(슬롯 맵)를 아직 다 못 읽은 상태에서 명령어를 실행하면 `TypeError: Cannot read properties of undefined (reading 'replicas')`로 크래시 나던 걸, 명확한 에러(`ClientClosedError`/`ClientOfflineError`)로 바꿈. 리뷰 과정에서 자동 리뷰 봇이 지적한 동시성 버그(`destroy()`와 겹치는 경우)도 추가로 수정.

### 1.5 결과

**✅ 2026-07-08 10:29 UTC 머지 완료.** PR 생성(06:57)부터 머지까지 **약 3시간 32분**. 메인테이너 `nkaradzhov`가 승인(`"CLGTM"`) 후 바로 머지.

---

## 2. 배경 지식

### 2.1 Redis Cluster의 슬롯(slot) 개념

Redis Cluster는 데이터를 **16384개의 슬롯**으로 나눠서 여러 노드(서버)에 분산시킨다. 어떤 key든 해시 함수로 슬롯 번호가 정해지고, 그 슬롯을 담당하는 노드(마스터 + 레플리카들)에게 요청을 보내야 한다. 그래서 클라이언트는 **"어느 슬롯이 어느 노드에 있는지"를 미리 알고 있어야** 명령어를 올바른 노드로 라우팅할 수 있다.

### 2.2 `RedisClusterSlots` — 이 라우팅 정보를 관리하는 내부 클래스

`packages/client/lib/cluster/cluster-slots.ts`에 정의된 클래스. `RedisCluster`(사용자가 직접 쓰는 공개 클래스)는 내부적으로 이 클래스에 라우팅을 위임한다.

### 2.3 `connect()`, `#discoverWithRootNodes()`(디스커버리), `destroy()`가 각각 하는 일

**`connect()`**: 클러스터에 연결을 시작하는 진입점.
```ts
async connect() {
  if (this.#isOpen) throw new Error('Cluster already open');
  this.#isOpen = true;
  try {
    await this.#discoverWithRootNodes();
    this.#isReady = true;
    this.#emit('connect');
  } catch (err) {
    this.#isOpen = false;
    this.#isReady = false;
    throw err;
  }
}
```

**`#discoverWithRootNodes()`(디스커버리)**: 사용자가 설정한 "root node"(초기 접속 주소)에 실제로 네트워크 연결을 시도해서 "클러스터 구성이 어떻게 되나요?"를 물어보는 **비동기 네트워크 작업**. 성공하면 라우팅 테이블(`this.slots`, `this.masters`, `this.replicas`)을 채운다.

**`destroy()`**: 클러스터를 즉시 닫는 메서드. `isOpen`/`isReady`를 `false`로 만들고, 개별 노드 연결을 전부 끊고, 라우팅 테이블을 비운다(`#resetSlots()`).

### 2.4 `isOpen`과 `isReady`의 차이 (우리가 이번에 만든 개념)

- `isOpen`: "클러스터 객체가 열려있냐" — `connect()` 호출 시점에 바로 `true`가 됨
- `isReady`: "라우팅 테이블이 실제로 다 채워졌냐" — 디스커버리가 **끝난 후에야** `true`가 됨

이 둘이 분리돼 있는 이유가 이번 버그의 핵심이다.

### 2.5 Redis Cluster는 왜 하필 16384개의 슬롯으로 나누는가

Redis Cluster 스펙(Redis 창시자 antirez가 설계)에 명시된 이유가 있다. 슬롯 수를 정할 때 두 가지가 충돌한다:

- **슬롯이 너무 적으면**: 노드가 많아질수록(수백~수천 대) 슬롯 하나당 담당 범위가 너무 커져서 세밀한 분산이 안 됨
- **슬롯이 너무 많으면**: 클러스터 내부적으로 각 노드가 "나는 어떤 슬롯들을 담당한다"는 정보를 **비트맵(bitmap)** 형태로 만들어서 다른 노드들에게 주기적으로 전파(gossip)하는데, 슬롯 수가 늘어날수록 이 비트맵 크기가 커져서 매번 보내는 하트비트 패킷이 무거워짐

**16384(=2^14)개**는 이 둘 사이의 절충점이다: 비트맵으로 표현하면 `16384 / 8 = 2048바이트(2KB)`가 되는데, 이 정도는 클러스터 하트비트 메시지에 매번 실어 보내도 부담이 적으면서, 실전에서 예상하는 최대 클러스터 규모(수백~1000대 노드 수준)에는 충분히 세밀한 분산을 제공한다. 즉 "메시지 크기 vs 분산 세밀도"의 엔지니어링 트레이드오프에서 나온 숫자다.

**어떤 key가 어느 슬롯으로 가는지**는 `CRC16(key) % 16384` 공식으로 계산한다(`packages/client`가 의존하는 `cluster-key-slot` 패키지가 이 계산을 담당 — 우리 테스트 코드에서도 `import calculateSlot from 'cluster-key-slot'`으로 쓰는 걸 확인했다). 여러 key를 한 번에 다루는 명령(`MSET`, 트랜잭션 등)을 같은 노드로 보내고 싶으면, key 이름에 `{...}` 형태의 "해시 태그"를 넣어서 그 부분만 해시 계산에 쓰도록 강제할 수 있다 (예: `user:{123}:profile`과 `user:{123}:settings`는 `{123}` 부분만 해시되어 항상 같은 슬롯으로 감).

### 2.6 슬롯 소유자가 바뀌면? — `MOVED`/`ASK`와 `rediscover()`

클러스터를 운영하다 보면 슬롯을 다른 노드로 옮기는 일(리밸런싱, 노드 추가/제거)이 생긴다. 이때 클라이언트가 들고 있는 라우팅 테이블은 순간적으로 낡은 정보가 된다. Redis Cluster 프로토콜은 이걸 두 가지 에러로 클라이언트에게 알려준다:

- **`MOVED <slot> <ip>:<port>`**: "그 슬롯은 이제 완전히 다른 노드로 이사갔다. 앞으로는 그 노드로 보내라" — 클라이언트는 이 응답을 받으면 라우팅 테이블을 갱신해야 함
- **`ASK <slot> <ip>:<port>`**: "그 슬롯은 지금 이사 **중**이다. 이번 요청만 저 노드로 다시 보내되(그 전에 `ASKING` 명령을 먼저 보내야 함), 라우팅 테이블 자체는 아직 영구적으로 바꾸지 마라" — 이사가 다 끝나기 전까지 임시로 양쪽 노드에 데이터가 걸쳐 있을 수 있기 때문

`cluster-slots.ts`의 `rediscover()`/`#rediscover()`가 바로 이런 상황에서 라우팅 테이블을 다시 읽어오는 역할을 한다. 우리가 4번 섹션에서 "회귀 위험 없음"을 검증할 때 확인했던 게 바로 이거다 — `rediscover()`는 `isReady`를 안 건드리니까, 슬롯이 실시간으로 재배치되는 중에도 이미 연결된 클라이언트가 갑자기 명령을 거부당하는 일은 없다.

### 2.7 JS/TS의 `#` 문법 — "진짜" private과 TypeScript `private`의 차이

이 코드베이스는 `#isOpen`, `#isReady`처럼 `#` 접두사를 쓴다. TypeScript에는 원래 `private isOpen`처럼 쓰는 키워드도 있는데, 이건 **컴파일 시점에만 체크되는 타입 시스템의 약속**일 뿐이다 — 컴파일된 JS 코드에서는 그냥 평범한 프로퍼티라 `obj['isOpen']`으로 외부에서 얼마든지 읽고 쓸 수 있다.

반면 `#isOpen`(ECMAScript 표준의 **진짜 private 필드** 문법)은 **JS 엔진 자체가 런타임에 강제로 막는다.** 클래스 바깥에서는 `obj.#isOpen`이라고 쓰는 것 자체가 문법 오류이고, `obj['#isOpen']`처럼 우회하려고 해도 접근이 안 된다. 이게 왜 우리 작업과 관련 있었냐면 — 이번에 레이스 컨디션을 재현하는 테스트를 만들려고 했을 때, `#isReady`/`#discoverWithRootNodes()` 같은 진짜 private 멤버들을 **테스트 코드에서 강제로 조작하거나 타이밍을 주입할 방법이 없었다.** 그래서 "이 정확한 순간에 destroy()를 호출해줘" 같은 결정적인(deterministic) 테스트를 못 만들고, 코드를 직접 추적해서 논리적으로 검증하는 방식으로 대신했던 것.

### 2.8 이번 버그의 진짜 근본 원리 — "JS의 Promise는 취소가 안 된다"

여기가 이번 학습에서 가장 일반적으로 써먹을 수 있는 개념이다. JavaScript의 `Promise`(그리고 `async`/`await`)는 **한번 시작되면 외부에서 강제로 멈출 방법이 기본적으로 없다.**

```ts
const p = someAsyncNetworkCall();   // 일단 시작되면
// 여기서 "이제 이 결과 필요없어, 취소해줘"라고 할 방법이 없음
// p는 결국 성공하거나 실패하거나 둘 중 하나로 "언젠가는" 끝난다
```

`connect()` 안에서 `await this.#discoverWithRootNodes()`를 호출한 순간, 이 네트워크 요청은 **이미 발사된 화살**과 같다. 중간에 `destroy()`를 아무리 불러도, 이미 날아간 네트워크 패킷과 그 응답을 기다리는 Promise 자체는 계속 살아있다가 **나중에 반드시 resolve되거나 reject된다.** 문제는 그 Promise가 끝나고 실행되는 이어지는 코드(`.then()` 이후 부분, 즉 `await` 다음 줄들)가 **"내가 시작됐을 때의 상황"이 지금도 여전히 유효하다고 착각하고 그대로 실행**된다는 것 — 이번 버그에서는 `isReady = true`가 바로 그 "착각하고 실행되는 코드"였다.

**일반적인 해결 패턴 두 가지** (참고로 알아두면 좋음):
1. **우리가 쓴 방식 — 재확인(guard) 패턴**: `await` 끝난 직후에 "지금 상황이 아직도 내가 기대하던 상황이 맞나?"를 다시 체크(`if (this.#isOpen)`)하고, 아니면 그냥 무시. 간단하지만 상태가 복잡해지면(동시에 여러 세대의 connect()가 겹치는 경우 등) 완벽하지 않을 수 있음.
2. **더 견고한 방식 — 세대(generation)/토큰 카운터**: `connect()`가 호출될 때마다 고유 번호(`generationId`)를 하나 발급하고, `await` 끝난 뒤 "지금의 최신 번호랑 내가 시작할 때 받은 번호가 같은지"를 비교. 같을 때만 상태를 반영. 이러면 "옛날 connect() 시도"와 "방금 시작한 새 connect() 시도"를 확실하게 구분할 수 있음. (실제 브라우저 API인 `AbortController`/`AbortSignal`도 결국 비슷한 발상 — "이 작업에 대한 관심을 취소했다"는 신호를 명시적으로 전파하는 것)

이번 PR에서는 리뷰에서 지적된 정확한 범위(1번 방식)만 고쳤는데, 만약 여기서 "동시에 여러 번 connect()/destroy()가 번갈아 빠르게 호출되는" 더 복잡한 시나리오까지 완벽하게 막고 싶다면 2번 방식으로 가는 게 더 안전하다 — 다음에 비슷한 동시성 버그를 만나면 고려해볼 만한 패턴.

---

## 3. 이슈 분석 (#2704)

### 3.1 이슈 원문 (2024-02-16 등록, **2년 넘게 방치된 이슈**)

**제목**: "Cannot read properties of undefined (reading 'replicas') [calling "get"]"

**환경**: Node 18, AWS ElastiCache(Redis 6.2.6), 2-shard 클러스터

**보고자가 남긴 실제 스택트레이스**:
```
TypeError: Cannot read properties of undefined (reading 'replicas')
    at RedisClusterSlots.getSlotRandomNode (.../cluster-slots.js:118:19)
    at RedisClusterSlots.getClient (.../cluster-slots.js:110:37)
    at Commander._RedisCluster_execute (.../cluster/index.js:217:79)
    ...
```

**흥미로운 점**: 보고자의 코드를 보면 **`await redisClient.connect()`를 이미 제대로 하고 있다.**
```ts
redisClient = redis.createCluster({ rootNodes: [{ url: config.url }], ... });
await redisClient.connect();   // ← 제대로 await 하고 있음
return redisClient;
```
그런데도 "간헐적으로(intermittent)" 이 에러가 난다고 보고함. 즉 **"연결을 안 기다려서 생기는 단순한 실수"보다 더 깊은 원인이 있을 가능성**을 시사하는 대목 — 이건 나중에 우리가 발견한 2차 버그(동시성 문제, 5번 섹션)와 관련 있을 수도 있다고 추정되지만, **정확히 이 보고자의 케이스가 어떤 경로로 터졌는지까지는 확실히 추적하지 못했다** (2년 전 리포트라 재현 환경이 없음). 정직하게 기록해두는 지점.

### 3.2 근본 원인

`getSlotRandomNode(slotNumber)` 내부:
```ts
getSlotRandomNode(slotNumber: number) {
  const slot = this.slots[slotNumber];
  if (!slot.replicas?.length) {   // ← slot이 undefined면 여기서 크래시
    return slot.master;
  }
  ...
}
```
디스커버리가 아직 안 끝났으면 `this.slots`(라우팅 테이블) 배열이 비어있어서 `this.slots[slotNumber]`가 `undefined`. 그 상태에서 `.replicas`에 접근하니 `TypeError`.

**참고로 `slot.master`/`slot.replicas`가 왜 둘 다 있는지**: Redis Cluster에서 슬롯 하나는 마스터 노드 1개 + 그 마스터를 복제하는 레플리카 노드 여러 개로 구성된다(장애 대비용 복제). 쓰기(`SET` 등)는 무조건 마스터로 가야 하지만, 읽기 전용 명령(`GET` 등)은 설정에 따라 **레플리카로 분산해서 읽기 부하를 나눌 수 있다**(read scaling). `getSlotRandomNode()`가 `replicas` 배열이 있으면 그중 하나를 무작위로 골라 리턴하는 이유가 이거다. 이번 크래시는 읽기 명령(`GET`)에서 나서, 하필 이 replicas 조회 로직을 타고 들어가다 터진 것.

### 3.3 실제 재현 (직접 검증)

이론으로 끝내지 않고, 원본 코드에 임시 진단 코드를 넣어서 직접 재현했다:
```
REJECTED with TypeError: Cannot read properties of undefined (reading 'replicas')
    at RedisClusterSlots.getSlotRandomNode (.../cluster-slots.ts:2:19420)
    at RedisClusterSlots.getClientAndSlotNumber (.../cluster-slots.ts:2:18224)
    at Class._execute (.../cluster/index.ts:2:6317)
    at Class.get (.../cluster/index.ts:2:2095)
```
이슈에 보고된 것과 **함수 이름, 에러 메시지가 정확히 일치**하는 걸 확인 후 수정 작업 시작.

**참고로 스택트레이스의 `_execute`가 뭐하는 애인지**: `RedisCluster`에서 `.get()` 같은 명령을 부르면 실제로는 `index.ts`의 `_execute()`가 받아서 이런 일을 한다 —
```ts
async _execute(firstKey, isReadonly, options, fn) {
  let { client, slotNumber } = await this._slots.getClientAndSlotNumber(firstKey, isReadonly);  // ① 라우팅
  while (true) {
    try {
      return await myFn(client, opts);   // ② 실제 명령 실행
    } catch (err) {
      if (err.message.startsWith('ASK')) { ... }    // ③ ASK/MOVED 리다이렉트면 재시도
      ...
    }
  }
}
```
①번이 바로 우리가 고친 `getClientAndSlotNumber()`고, ③번이 2.6절에서 설명한 `MOVED`/`ASK` 처리가 실제로 구현된 곳이다(최대 `maxCommandRedirections`번, 기본값 16번까지 재시도). 즉 `_execute()`는 "① 어디로 보낼지 정하고 → ② 보내고 → ③ 클러스터가 재배치 중이라고 하면 다시 ①부터"를 반복하는 **명령 실행의 총괄 지휘자**다. 우리 버그는 바로 이 ①번 단계에서, 라우팅 테이블이 아직 없는데 라우팅을 시도하다가 터진 것.

---

## 4. 1차 수정 (첫 커밋)

### 4.1 설계

`RedisClusterSlots`에 `#isReady` 플래그 추가, `isOpen`과 별도로 관리:

```ts
#isReady = false;
get isReady() { return this.#isReady; }

#assertReady() {
  if (!this.#isOpen) throw new ClientClosedError();
  if (!this.#isReady) throw new ClientOfflineError();
}
```

명령어 라우팅 진입점 4곳(`getClientAndSlotNumber`, `getClient`, `getPubSubClient`, `getShardedPubSubClient`)에 `this.#assertReady()`를 호출부 맨 앞에 추가.

`connect()`에서 디스커버리 성공 시 `#isReady = true`, 실패/`destroy()` 시 `#isReady = false`로 관리.

### 4.2 두 가지 에러로 구분한 이유

- `ClientClosedError`: 아예 `connect()`를 안 불렀거나 닫힌 상태
- `ClientOfflineError`: `connect()`는 불렀는데 아직 토폴로지 로딩 중

사용자가 "내가 뭘 안 했는지"를 정확히 알 수 있게 구분.

### 4.3 회귀 위험 사전 점검

`#isReady`가 런타임 중 정상적인 토폴로지 재조정(`rediscover()`, resharding 대응)에서 실수로 꺼지지 않는지 코드를 직접 추적해서 확인 — `#isReady`는 `connect()`/`destroy()`/`#destroy()` 세 곳에서만 건드리고, `rediscover()`/`#rediscover()`/`#discover()`는 건드리지 않음을 확인. 즉 이미 연결된 클러스터가 슬롯 재배치 중에 갑자기 명령을 거부하는 회귀는 없음.

### 4.4 테스트 추가

이슈의 두 시나리오를 그대로 재현하는 테스트 2개 추가 (`Cluster command lifecycle` describe 블록):
1. `connect()`를 아예 안 부른 경우 → `ClientClosedError`
2. `connect()`는 불렀지만 `await` 안 하고 바로 명령 실행 (응답 없는 IP `203.0.113.1`로 일부러 멈춰있게 만듦) → `ClientOfflineError`

둘 다 **실제 Redis 서버 없이도** 통과하는 순수 단위 테스트.

---

## 5. 리뷰 라운드 — 시간순 그대로 (뭉개지 않고)

### 5.1 1차 커밋 push (2026-07-08 06:57 UTC경)

커밋 `78799f8c30` — "fix(cluster): reject commands before cluster topology is ready". PR #3321 생성.

### 5.2 Cursor Bugbot 자동 리뷰 (06:58:52 UTC, 약 1~2분 후)

`redis/node-redis` 저장소가 **모든 PR에 자동으로 돌리도록 설정해둔 AI 코드 리뷰 봇**(Cursor Bugbot)이 리뷰를 남김. 원문(정확히 인용):

> ### Stale isReady bypasses connect guard
> **High Severity**
>
> `connect()` does not clear `#isReady` when a new session opens, and it sets `#isReady` after discovery without checking `#isOpen`. If discovery finishes after `destroy()`, or a later `connect()` runs while `#isReady` is still true, `#assertReady()` passes during in-flight topology setup and routing can hit an empty slot map again.

### 5.3 문제를 정확히 이해하기 위한 분석 (직접 시간순 테이블로 추적)

이 지적이 정확히 뭘 말하는지 실제 코드로 시간순으로 추적:

| 시점 | 무슨 일이 벌어짐 | `isOpen` | `isReady` (1차 커밋 버전) |
|---|---|---|---|
| t0 | `connect()` 호출 시작 | `false → true` | `false` |
| t1 | 디스커버리 네트워크 요청 발송 (`await` 중) | `true` | `false` |
| t2 | **그 와중에 `destroy()` 호출됨** | `true → false` | `false` |
| t3 | `destroy()`가 연결 정리 + 라우팅 테이블(`this.slots`) 비움 | `false` | `false` |
| t4 | t1의 네트워크 요청이 **destroy와 무관하게 계속 살아있다가, 뒤늦게 성공 응답 도착** | `false` | `false` |
| t5 | 디스커버리 성공 처리 — `isReady = true` **무조건 실행** | `false` | **`true`** ← 모순! |

t5에서 `isOpen=false`인데 `isReady=true`가 되는 모순 상태가 생김. 이후 `connect()`를 다시 호출하면, 새 디스커버리가 안 끝났는데도 이 stale한 `true` 때문에 `#assertReady()`가 통과해버려서 **원래 고치려던 크래시가 재발할 수 있는 구멍**이었다.

### 5.4 2차 수정 (커밋 `7f6fc90ad2`, 같은 날)

```ts
this.#isOpen = true;
this.#isReady = false;              // ① 세션 시작 시 무조건 리셋 (안전장치)
try {
  await this.#discoverWithRootNodes();
  if (this.#isOpen) {               // ② 진짜 핵심 수정: destroy 됐으면 무시
    this.#isReady = true;
    this.#emit('connect');
  }
} catch (err) {
  this.#isOpen = false;
  this.#isReady = false;
  throw err;
}
```

- **②번**이 5.3절 t5 시점의 문제를 직접 막음: `destroy()`가 이미 `isOpen`을 `false`로 바꿔놨으니, t5에서 `if (this.#isOpen)`이 거짓이 되어 `isReady = true`가 실행조차 안 됨.
- **①번**은 별도의 이중 안전장치: 혹시 다른 경로로 `isReady`가 잘못 `true`로 남아있어도, 새 `connect()`가 시작하자마자 강제로 `false`로 밀어버려서 새 세션에 영향 못 주게 함.

수정 후 빌드/린트/기존 테스트 2개 재확인, 모두 통과.

### 5.5 Bugbot 재검토 (2차 커밋 기준, 자동)

새 커밋(`7f6fc90ad2`)에 대해 Bugbot이 다시 스캔 → **`conclusion: success`, 추가 코멘트 없음** (문제를 못 찾으면 코멘트 없이 체크만 통과시키는 방식). 즉 2차 수정이 지적된 문제를 제대로 해결했다고 자동 리뷰 통과.

### 5.6 메인테이너 승인 및 머지 (10:28~10:29 UTC)

메인테이너 `nkaradzhov`가 리뷰 남김: **`"CLGTM"`** (state: `APPROVED`, 10:28:34 UTC). 1분 뒤인 10:29:11 UTC에 머지 완료.

**전체 소요 시간**: PR 생성(06:57:25) → 머지(10:29:11) = **약 3시간 32분**. 저번에 확인했던 이 저장소의 "작고 명확한 fix는 1일 안에도 머지된다"는 패턴보다도 훨씬 빨랐다.

---

## 6. 기여 프로세스

### 6.1 Fork & 리모트 설정

```bash
gh repo fork redis/node-redis
git remote rename origin upstream
git remote add origin https://github.com/GiHoon1123/node-redis.git
```

### 6.2 브랜치

`git checkout -b fix/cluster-command-before-ready` — 이번엔 (이전 platformatic 작업 때의 실수를 교훈 삼아) 코드 수정 전에 먼저 브랜치를 만들고 시작했다.

### 6.3 커밋 컨벤션 확인

`CONTRIBUTING.md`, README, PR 템플릿을 다 확인한 결과 **DCO도 CLA도 요구하지 않음**을 확인. 최근 커밋 15개 중 3개(20%)에서만 `Signed-off-by`가 발견돼서, 이것도 강제가 아니라 일부 기여자의 개인 습관 수준이라고 판단.

### 6.4 AI 언급 여부

사용자 요청에 따라 **커밋 메시지/PR 본문에 AI 도구 관련 언급을 전혀 넣지 않음**. 실제로 push 후 커밋 메시지와 PR 본문을 다시 검색해서 AI 도구 관련 키워드가 없는 것까지 확인함.

### 6.5 PR 체크리스트를 정직하게 작성

PR 템플릿의 체크리스트 3개 중 2개만 체크:
```
- [x] Does `npm test` pass with this change (including linting)?
- [x] Is the new or changed code fully tested?
- [ ] Is a documentation update included...? 
      → isReady는 작은 추가 getter라 별도 문서는 안 건드림, 필요하면 추가하겠다고 코멘트로 남김
```
실제로 안 한 걸 체크 안 하고 이유를 남기는 방식 — 이후 사용자가 이 태도를 긍정적으로 평가함.

---

## 7. 트러블슈팅

### 7.1 macOS Docker Desktop의 `--network host` 문제

**증상**: `testWithCluster` 기반의 넓은 통합 테스트가 로컬에서 계속 타임아웃 남.

**원인**: `packages/test-utils/lib/dockers.ts`의 `spawnRedisServerDocker`가 컨테이너를 `--network host`로 띄우는데, **macOS Docker Desktop에서는 이게 실제 Mac 호스트가 아니라 Docker Desktop 내부 Linux VM의 네트워크에 붙어서, 우리 Node.js 테스트 프로세스가 `127.0.0.1:<포트>`로 절대 접속 못 함.** `docker inspect`로 `NetworkMode: host`, `Ports: {}` 확인, `docker exec ... redis-cli ping`으로 컨테이너 내부는 멀쩡히 `PONG` 응답하는 것도 확인해서 원인 특정.

**로컬 검증용 임시 우회** (커밋에는 포함 안 함): `'--network', 'host'` → `'-p', '${port}:${port}'`로 임시 패치해서 재빌드 후 넓은 테스트 재실행. 검증 끝나고 `git checkout HEAD -- packages/test-utils/lib/dockers.ts`로 완전히 원복.

**남은 실패의 원인 구분**: 이 우회 방식으로도 `FUNCTION LOAD`, 일반 `Client` 테스트 몇 개가 `ECONNRESET`으로 실패했는데, **우리 수정을 스태시로 치우고 원본 코드로 똑같이 돌려도 동일하게 실패**하는 것을 확인 → 우리 fix와 무관한, 컨테이너를 짧은 시간에 여러 개 띄울 때 생기는 **로컬 Docker 리소스 경쟁 문제**로 결론. CI는 리눅스 네이티브 Docker라 이 문제 자체가 없음.

### 7.2 백그라운드 실행 로그가 잘려서 원인 파악이 늦어짐

첫 번째 넓은 테스트 실행 로그 파일이 실제 출력보다 훨씬 짧게(36줄) 잘려 있어서 원인 파악에 혼선이 있었음. `> /tmp/xxx.log 2>&1`로 직접 리다이렉트하는 방식으로 바꿔서 해결.

---

## 8. 결과 및 회고

### 8.1 최종 결과

**✅ 머지 완료** (2026-07-08 10:29 UTC, 머지 커밋 `b393c0e7`). PR 생성부터 머지까지 3시간 32분 — 우리 4개 PR 중 첫 번째로 실제 머지에 도달한 사례.

### 8.2 배운 점

- **자동 AI 리뷰 봇(Cursor Bugbot)이 실제로 유효한 동시성 버그를 잡아냈다.** 우리가 처음 낸 수정 자체가 이미 원래 버그는 고쳤지만, **새로운 종류의 버그(동시성 레이스)를 만들어낼 뻔했다.** 코드 리뷰(사람이든 봇이든)가 "일단 돌아가긴 하는" 수정과 "정말 견고한" 수정을 가르는 지점이라는 걸 실전에서 확인함.
- **비동기 코드에서 "이 작업이 끝났을 때, 그 사이에 다른 이벤트가 이미 일어났을 수도 있다"는 걸 항상 재확인해야 한다**는 일반적인 동시성 프로그래밍 원칙을 구체적인 실제 사례(디스커버리 vs destroy)로 체득함.
- **자동 리뷰든 사람 리뷰든, 지적을 받으면 "왜 그런지" 실제 코드로 시간순으로 완전히 이해한 다음에 고쳐야 한다** — 대충 지적된 줄만 고치고 넘어가면 또 다른 구멍이 생길 수 있음.
- 원본 이슈(2년 전 보고, AWS ElastiCache 프로덕션 환경)에서 보고자가 이미 `await connect()`를 제대로 하고 있었는데도 간헐적으로 터졌다는 점은, 우리가 고친 두 가지 버그(단순 미대기 케이스 + 동시성 레이스 케이스) 중 어느 쪽이 정확히 원인이었는지 확정할 수는 없었지만, **적어도 같은 클래스의 문제(토폴로지 미준비 상태에서의 라우팅 시도)를 포괄적으로 막는 수정**이 됐다는 점에서 의미가 있었음.
- **로컬 개발 환경(특히 Docker)의 문제와 코드 자체의 회귀를 구분하는 방법론**(스태시 기반 베이스라인 비교)을 두 번째로 실전 적용함 (첫 번째는 platformatic/kafka에서).

### 8.3 다음 계획

- 이걸로 "진짜 공식" 라이브러리(confluent-kafka-javascript, node-redis) 우선순위 전략이 검증됨 — 앞으로도 이 두 저장소에 계속 기여 예정
- 나머지 3개 PR(confluentinc #506, platformatic/kafka #327/#329)도 계속 지켜보는 중
