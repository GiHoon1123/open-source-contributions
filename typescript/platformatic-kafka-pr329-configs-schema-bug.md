# [Kafka OSS 기여] platformatic/kafka - createTopics configs 스키마 누락 버그 수정 (PR #329)

> 오픈소스 기여 기록 및 학습 노트. 이 문서는 PR #329 하나만 다룬다. 같은 저장소에 대한 첫 번째 기여(PR #327, 토픽별 설정 지원)는 별도 문서 `platformatic-kafka-pr327-per-topic-overrides.md`에 정리돼 있고, 이 문서는 그 작업 도중 발견한 별개의 버그를 다룬다.

---

## 1. 개요

### 1.1 프로젝트 소개 (간략, 자세한 배경은 PR #327 문서 참고)

- **라이브러리**: `@platformatic/kafka` — Node.js용 순수 TypeScript/JavaScript Kafka 클라이언트(네이티브 바인딩 없음)
- **저장소**: https://github.com/platformatic/kafka
- NestJS 창시자 `kamilmysliwiec`가 실질적으로 미는 라이브러리 (자세한 근거는 PR #327 문서 1.2절)

### 1.2 이 PR의 성격 — "발견형" 기여

이번 기여는 이슈 트래커에서 이슈를 골라잡은 게 아니라, **PR #327(토픽별 설정 지원) 작업 도중 코드를 읽다가 우연히 발견한 별개의 버그**를 고친 것이다. 처음에는 PR #327에 같이 넣으려고 했으나, "한 PR은 하나의 목적만 다뤄야 한다"는 원칙에 따라 분리해서 별도로 제출했다 (분리 과정의 자세한 논의는 PR #327 문서의 6번 섹션 참고).

### 1.3 관련 링크

| 구분 | 링크 |
|---|---|
| 원본 저장소 | https://github.com/platformatic/kafka |
| 내 Fork | https://github.com/GiHoon1123/kafka |
| 관련 이슈 | 없음 — 코드 조사 중 직접 발견한 버그라 별도 이슈 없이 바로 PR로 제출 |
| 원인이 된 과거 PR | https://github.com/platformatic/kafka/pull/75 ("feat: Added configs option for createTopics method", 2025-07-29 머지) |
| 이번 PR (#329) | https://github.com/platformatic/kafka/pull/329 |
| 작업 브랜치 | `fix/create-topics-configs-schema` |

### 1.4 한 줄 요약

> `createTopics()`의 `configs` 옵션이 실제로는 정상적으로 지원되는 기능인데, 검증 스키마에 등록이 안 돼 있어서 **`strict: true` 모드로 클라이언트를 만든 사용자한테만 조건부로 발현되는** 버그를 수정.

---

## 2. 배경 지식

### 2.1 ajv와 JSON Schema 검증 (PR #327 문서 2.3절과 동일한 내용, 이 문서만 봐도 이해되게 다시 정리)

이 라이브러리는 `ajv`라는 JSON Schema 검증 라이브러리로 사용자가 넘긴 옵션 객체의 모양을 검사한다.

```ts
export const ajv = new Ajv2020({ allErrors: true, coerceTypes: false, strict: true })
```

`createTopicOptionsSchema`처럼 "이 옵션 객체는 이런 프로퍼티들만 가질 수 있다"를 JSON으로 정의해두고, `ajv.compile(schema)`로 실제 검증 함수를 만든다. 스키마에 `additionalProperties: false`가 있으면, **정의 안 된 프로퍼티가 하나라도 있으면 검증 전체가 실패**한다.

### 2.2 핵심 메커니즘 — 검증은 `strict: true`일 때만 동작한다

이번 버그를 이해하는 데 가장 중요한 사실. `src/clients/base/base.ts`:

```ts
[kValidateOptions] (
  target: unknown,
  validator: ValidateFunction<unknown>,
  targetName: string,
  throwOnErrors: boolean = true
): Error | null {
  if (!this[kOptions].strict) {
    return null   // ← 여기가 핵심. strict 옵션이 없으면 검증을 아예 안 하고 그냥 통과시킴
  }

  const valid = validator(target)

  if (!valid) {
    const error = new UserError(this[kFormatValidationErrors](validator, targetName))
    if (throwOnErrors) {
      throw error
    }
    return error
  }

  return null
}
```

즉 클라이언트를 `new Admin({ ...접속정보 })`처럼 만들면 **옵션 검증이 완전히 스킵**되고, `new Admin({ ...접속정보, strict: true })`처럼 만들어야만 검증이 실제로 동작한다. 이게 왜 중요하냐면 — **버그가 있어도 strict 모드를 안 쓰는 대다수 사용자는 절대 못 느낀다**는 뜻이기 때문이다.

### 2.3 `CreateTopicsOptions.configs`란

Kafka 토픽 생성 시 토픽 단위 설정(예: `cleanup.policy=compact`, `retention.ms=...`)을 지정하는 필드. 타입 정의(`src/clients/admin/types.ts`)에는 원래부터 존재했다:
```ts
export interface CreateTopicsOptions {
  topics: string[]
  partitions?: number
  replicas?: number
  assignments?: BrokerAssignment[]
  configs?: CreateTopicsRequestTopicConfig[]   // ← 타입에는 있음
}
```

---

## 3. 버그 분석

### 3.1 발견 경위

PR #327(토픽별 설정 지원) 작업 중, 이슈 #198이 요청한 4가지 항목(파티션 수/복제 팩터/파티션 배치/**configs**) 중 `configs`까지 토픽별로 지원하려고 `src/clients/admin/options.ts`의 `createTopicOptionsSchema`를 들여다봤다. 그런데:

```ts
export const createTopicOptionsSchema = {
  type: 'object',
  properties: {
    topics: { type: 'array', items: idProperty },
    partitions: { type: 'number' },
    replicas: { type: 'number' },
    assignments: { /* ... */ }
    // configs가 없다!
  },
  required: ['topics'],
  additionalProperties: false
}
```

**`configs`가 `properties`에 아예 없었다.** `additionalProperties: false`와 결합하면, `strict: true` 모드에서 `configs`를 넘기는 순간 "정의 안 된 프로퍼티"로 거부당한다는 뜻.

### 3.2 실제로 재현되는지 직접 검증

이론만으로 끝내지 않고, 실제 컴파일된 검증기를 직접 호출해서 확인했다:
```js
// verify_schema_tmp.mjs (프로젝트 루트에 임시로 작성, 검증 후 삭제)
import { createTopicsOptionsValidator } from './src/clients/admin/options.ts'

const result = createTopicsOptionsValidator({
  topics: ['test'],
  configs: [{ name: 'cleanup.policy', value: 'compact' }]
})

console.log('valid:', result)
console.log('errors:', JSON.stringify(createTopicsOptionsValidator.errors, null, 2))
```

결과:
```
valid: false
errors: [
  {
    "instancePath": "",
    "schemaPath": "#/additionalProperties",
    "keyword": "additionalProperties",
    "params": { "additionalProperty": "configs" },
    "message": "must NOT have additional properties"
  }
]
```

**버그가 실제로 재현됨을 코드 레벨에서 직접 증명.**

### 3.3 그런데 왜 기존 테스트는 통과했나? (헷갈렸던 부분)

`test/clients/admin/admin.test.ts`에 이미 `configs`를 쓰는 실제 통합 테스트("createTopics with custom configuration")가 있는데, 이게 실제 로컬 Kafka 브로커를 상대로 **항상 통과하고 있었다.** 처음엔 "버그가 아닌가?"하고 혼란스러웠는데, 원인은 2.2절의 메커니즘 때문이었다 — **그 테스트가 `createAdmin(t)`로 클라이언트를 만들어서 `strict: true`를 안 쓰고 있어서, 애초에 검증 자체가 스킵되고 있었던 것.** 반면 `strict: true`를 쓰는 "createTopics should validate options in strict mode" 테스트는 `configs`를 아예 테스트하지 않고 있었다 — 그래서 이 조합(strict 모드 + configs)이 지금까지 아무 테스트에도 안 걸렸던 것.

이걸 직접 검증하려고, `git stash`로 원본 코드로 되돌린 상태에서 같은 검증 스크립트를 실행해서 **`kValidateOptions`가 `strict` 체크로 조기 리턴하는 것 때문에 실제 CRUD 테스트는 영향을 안 받는다**는 걸 재차 확인했다.

### 3.4 원인 추적 — 언제, 왜 이렇게 됐나

`git log -p -S"configs"` 방식으로 히스토리를 추적한 결과:
- **PR #75** ("feat: Added configs option for createTopics method", 2025-07-29 머지)가 `configs` 타입과 `#createTopics` 로직 지원을 추가했다.
- 그 PR의 diff를 직접 확인(`curl https://github.com/platformatic/kafka/pull/75.diff`)한 결과, **`options.ts`(스키마 파일)는 전혀 건드리지 않았다.**

즉 기능(타입+로직)은 추가했는데 검증 스키마 업데이트를 깜빡한, 전형적인 "한쪽만 고치고 다른 쪽을 빠뜨린" 버그.

---

## 4. 수정 내용

### 4.1 변경 파일 2개

| 파일 | 역할 |
|---|---|
| `src/clients/admin/options.ts` | `createTopicOptionsSchema`에 `configs` 프로퍼티 추가 |
| `test/clients/admin/admin.test.ts` | strict mode에서 `configs`가 정상 동작하는지 + 잘못된 타입이 거부되는지 테스트 추가 |

### 4.2 `options.ts`

```ts
export const createTopicOptionsSchema = {
  type: 'object',
  properties: {
    topics: { type: 'array', items: idProperty },
    partitions: { type: 'number' },
    replicas: { type: 'number' },
    assignments: { /* 기존과 동일 */ },
    configs: {                                    // ← 새로 추가
      type: 'array',
      items: {
        type: 'object',
        properties: {
          name: { type: 'string' },
          value: { type: ['string', 'null'] }
        },
        required: ['name'],
        additionalProperties: false
      }
    }
  },
  required: ['topics'],
  additionalProperties: false
}
```

`name`/`value` 필드 모양은 이미 존재하던 타입 `CreateTopicsRequestTopicConfig`(`{ name: string, value?: string | null }`)와 정확히 맞춰서 정의했다. `value`가 `['string', 'null']` 유니온인 이유는 원본 타입이 `NullableString = string | null`이기 때문.

### 4.3 테스트 추가

**정상 케이스** (strict mode에서 `configs`가 실제로 동작하는지, 회귀 테스트):
```ts
test('createTopics should accept the configs option in strict mode', async t => {
  const admin = createAdmin(t, { strict: true })
  const topicName = `test-topic-${randomUUID()}`

  const created = await admin.createTopics({
    topics: [topicName],
    configs: [{ name: 'cleanup.policy', value: 'compact' }]
  })

  strictEqual(created[0].configuration['cleanup.policy'], 'compact')
  await admin.deleteTopics({ topics: [topicName] })
})
```

**부정 케이스** (타입이 틀리면 여전히 잘 거부하는지, 기존 "should validate options in strict mode" 테스트 블록에 추가):
```ts
// Test with invalid configs type
try {
  await admin.createTopics({ topics: ['test-topic'], configs: 'not-an-array' })
  throw new Error('Expected error not thrown')
} catch (error) {
  strictEqual(error.message.includes('configs'), true)
}
```

---

## 5. 왜 PR #327과 분리했는가

### 5.1 처음엔 같이 넣으려고 했다

같은 스키마 파일(`options.ts`)을 고치는 김에, 이슈 #198이 요청한 `configs`까지 한 번에 지원하려고 PR #327 작업 중 이 수정을 같이 넣었었다. 실제로 그 상태로 테스트까지 다 통과시켰다.

### 5.2 되돌리고 분리한 이유

"원래 PR은 하나의 기능만 다뤄야 하지 않냐"는 지적을 받고 재검토한 결과:

| 기준 | PR #327 (토픽별 설정) | 이 버그 (`configs` 누락) |
|---|---|---|
| 발생 원인 | 우리가 새로 설계하는 기능 | 2025년 PR #75의 실수 |
| 영향 범위 | 앞으로 새로 쓰일 per-topic API | 이미 존재하던 전역 `configs` 옵션 전체 |
| #198과의 관계 | #198이 직접 요청한 기능 | #198과 우연히 같은 파일을 건드릴 뿐, 논리적 연관 없음 |

"같은 파일을 고친다"는 "같은 목적"이라는 뜻이 아니라는 게 핵심 결론. 그래서 PR #327에서는 `configs` 관련 코드를 전부 되돌리고(per-topic `configs` 지원도 함께 포기), 이 버그만 따로 뽑아서 이 PR로 제출했다.

### 5.3 분리 후 이점

- PR #327이 이 버그 수정 건의 반려/논쟁에 영향받지 않고 독립적으로 진행 가능
- 이 PR 자체도 매우 작고 명확해서(+12줄), 리뷰 부담이 낮음
- 같은 저장소에 짧은 시간 안에 잘 만들어진 PR을 두 개 제출함으로써, 신뢰도를 쌓는 효과도 기대

---

## 6. 기여 프로세스

### 6.1 브랜치 관리 — 이전 실수를 교훈 삼음

PR #327 때는 브랜치 만드는 걸 깜빡했었는데, 이번엔 **처음부터** 제대로 했다:
```bash
git checkout main
git pull upstream main     # 혹시 그 사이 누가 이미 이 버그를 고쳤는지 최신화해서 확인
git checkout -b fix/create-topics-configs-schema
```
`git pull upstream main` 후 `git log main..upstream/main`으로 새 커밋이 없는지, 그리고 `options.ts`의 과거 diff에서 이미 이 버그를 고친 흔적이 없는지도 확인했다 (중복 작업 방지).

### 6.2 커밋

PR #327과 동일한 컨벤션(DCO 서명 + AI 공동저자 명시, 근거는 PR #327 문서 7.4절 참고):

```bash
git commit -s -m "$(cat <<'EOF'
fix(admin): allow configs in createTopics options schema

CreateTopicsOptions.configs is a documented, tested field (see the
"createTopics with custom configuration" test), but
createTopicOptionsSchema never declared it. Validation is skipped
unless the client is created with strict: true, so this only
surfaces for strict-mode users, which is likely why it went
unnoticed since configs support was added in #75.

Co-Authored-By: AI Assistant <ai-assistant@example.com>
EOF
)"
```
`-s` 플래그로 `Signed-off-by: Gihoon1123 <...>`가 자동으로 붙음.

### 6.3 PR 생성

```bash
gh pr create --repo platformatic/kafka \
  --title "fix(admin): allow configs in createTopics options schema" \
  --body "..."
```
→ https://github.com/platformatic/kafka/pull/329

PR 본문에 "Context" 섹션을 따로 만들어서, **이게 왜 별도 PR로 왔는지**(PR #327 작업 중 발견했지만 무관한 기존 버그라 분리했다는 것)를 리뷰어가 헷갈리지 않게 명시했다.

---

## 7. 트러블슈팅

### 7.1 Docker 브로커 컨테이너가 OOM으로 죽어서 테스트 15개 실패

**증상**: 테스트를 재실행했더니 갑자기 168개 중 15개가 실패(`pass 153, fail 15`)로 나왔다. 전날(PR #327 작업 시점)에는 전부 통과했었는데, 우리 코드 변경은 아주 작은 스키마 추가뿐이라 이 정도 실패가 나올 이유가 없어서 의심스러웠다.

**원인 조사**:
```bash
docker compose ps -a --format "table {{.Name}}\t{{.Status}}"
```
```
broker-cluster-1       Up 23 hours (healthy)
broker-cluster-2       Up 23 hours (healthy)
broker-cluster-3       Exited (137) 18 hours ago    ← 여기!
...
```
**`broker-cluster-3`가 18시간 전에 exit code 137(대개 OOM/SIGKILL을 의미)로 죽어있었다.** 3개 브로커로 구성된 클러스터를 전제로 하는 테스트들(예: "createTopics should retarget controller when needed")이, 실제로는 브로커 2개짜리 클러스터를 상대하면서 실패한 것 — **완전히 환경 문제였고 우리 코드와는 무관.**

**해결**:
```bash
docker compose up -d   # 죽은 컨테이너만 다시 살아남
sleep 15
docker compose ps -a   # 9개 컨테이너 전부 healthy 확인
```
재실행 결과 168/168 전부 통과.

**교훈**: 오래 띄워둔 로컬 통합 테스트 환경(특히 여러 컨테이너로 구성된 클러스터)은 시간이 지나면서 리소스 문제로 일부가 죽을 수 있다. 테스트 실패가 나오면 **바로 "내 코드가 잘못됐다"고 결론짓지 말고, 먼저 `docker compose ps -a`로 인프라 상태부터 점검하는 습관**이 중요하다는 걸 실전에서 배움.

### 7.2 백그라운드 실행 로그가 잘려서 원인 파악이 늦어짐

**증상**: 처음 테스트 실패를 발견했을 때, 백그라운드로 실행한 명령의 로그 파일을 열어봤더니 **전체 168개 테스트 중 맨 마지막 몇 개(167, 168번)만 남아있고 실패 원인이 담긴 부분은 잘려있었다.** `wc -l`로 확인해보니 파일이 36줄뿐이었는데, 실제로는 수백 줄이 출력됐어야 정상.

**대응**: 원인을 깊이 파기보다(로그 캡처 메커니즘 자체의 한계로 추정), **`> /tmp/xxx.log 2>&1`로 직접 파일 리다이렉트**해서 같은 명령을 다시 실행하는 방식으로 전환. 이렇게 하니 전체 로그가 온전히 남았고, 그 사이 7.1절의 진짜 원인(Docker 컨테이너 사망)도 별도로 `docker compose ps -a`를 직접 실행해서 찾아냈다.

**교훈**: 백그라운드 작업의 로그가 의심스러울 때는, 도구의 자체 로그 캡처에만 의존하지 말고 **직접 파일로 리다이렉트해서 재실행**하는 게 확실하다.

---

## 8. 결과 및 회고

### 8.1 PR 상태

> 이 섹션은 결과가 나오는 대로 업데이트 예정.

- **현재 (작성 시점, 2026-07-08)**: PR #329, Open, 방금 제출, 리뷰/코멘트 없음

**갱신 (2026-07-28 확인)**: 여전히 리뷰/코멘트 없음(`REVIEW_REQUIRED`도 아닌 완전 무응답). CI(`mergeStateStatus: UNSTABLE`)는 Node/Kafka 버전 매트릭스 다수가 `fail`로 떠 있는데, 실패 로그를 확인해보니 원인은 우리 변경과 무관하다 — 실패한 유일한 테스트는 `test/issues/gh-300.test.ts`의 `survives pause/resume before initial offsets refresh completes (consumer)`로, consumer pause/resume 타이밍에 관한 별개 이슈(#300) 테스트다. 우리 변경은 스키마 필드 추가뿐이라 이 실패와 연결될 이유가 없음 — CI 자체의 사전 존재 flakiness로 보이며, 별도 조치 없이 리뷰만 기다리면 될 것으로 판단.

**갱신 (2026-08-06 확인)**: 여전히 무응답이라 확인 코멘트 남김.

**갱신 (2026-08-18)**: 메인테이너 `ShogunPanda`가 처음으로 반응 — "Can you please rebase?" 코멘트. `upstream/main` 기준으로 rebase(충돌 없이 clean) 후 `test/clients/admin/admin.test.ts`(168개, 3-broker 클러스터 대상) 전부 통과 확인하고 fork에 force-push, PR에 "Rebased onto main." 코멘트 남김. 약 6주 만의 첫 메인테이너 반응이라 리뷰가 재개될 가능성이 높아짐.

**최종 결과 (2026-08-20)**: 메인테이너 `ShogunPanda`가 PR #329를 머지했다. `strict: true`에서 정상적인 `createTopics({ configs })` 호출이 스키마 검증에 막히던 버그 수정이 `main`에 반영됐다.

### 8.2 배운 점

- **"strict 모드에서만 발현되는 버그"라는 개념**: 검증 로직이 조건부로 스킵될 수 있다는 걸 놓치면, "테스트가 통과하니까 버그가 아니다"라고 잘못 판단하기 쉽다. 실제로 처음엔 이것 때문에 혼란스러웠고, `kValidateOptions` 구현을 직접 읽고서야 정확한 그림을 이해했다.
- **주장을 코드 실행으로 직접 검증하는 습관**: "스키마에 `configs`가 없어 보인다"는 추측에서 끝내지 않고, 실제 컴파일된 validator를 직접 호출해서 `valid: false`라는 확정적 증거를 얻은 것이 스스로도, 그리고 나중에 PR 리뷰어를 설득하는 데도 중요했다.
- **"관련 있어 보이는 것"과 "같은 목적인 것"은 다르다는 원칙을 두 번째로 실전 적용**: PR #327에서 배운 교훈을 이번엔 처음부터 적용해서, 애초에 두 관심사를 섞지 않고 진행할 수 있었다.
- **인프라 문제와 코드 문제를 구분하는 디버깅 습관**: 테스트 실패 = 내 코드 탓이라고 성급하게 결론 내리지 않고, 환경 상태부터 점검하는 순서를 체화함.

### 8.3 다음 계획

- PR #327과 #329 모두 머지 완료.
- 이후 `platformatic/kafka`에서는 consumer 쪽처럼 더 깊은 Kafka 프로토콜 이해가 필요한 이슈를 다음 기여 후보로 검토.
