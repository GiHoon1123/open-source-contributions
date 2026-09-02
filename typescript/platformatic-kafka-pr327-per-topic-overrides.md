# [Kafka OSS 기여] platformatic/kafka - createTopics 토픽별 설정 지원 (PR #327)

> 오픈소스 기여 기록 및 학습 노트. 이 문서는 PR #327 하나만 다룬다 (같은 세션에서 나온 PR #329는 별도 문서 `platformatic-kafka-pr329-configs-schema-bug.md`에 정리).

---

## 1. 개요

### 1.1 프로젝트 소개

- **라이브러리**: `@platformatic/kafka`
- **저장소**: https://github.com/platformatic/kafka
- **누가 만드나**: Platformatic (Node.js 생태계의 핵심 인물들이 만든 회사) — 실제 커밋 히스토리에서 **Matteo Collina**(Node.js 코어 멤버), **Paolo Insogna**(Platformatic 메인테이너) 등의 이름이 반복적으로 등장함
- **뭐 하는 라이브러리인가**: Node.js용 Kafka 클라이언트. `confluent-kafka-javascript`와 달리 **네이티브 C++ 바인딩(librdkafka)을 전혀 안 쓰고, Kafka 와이어 프로토콜을 처음부터 순수 TypeScript/JavaScript로 직접 구현**한 라이브러리. `CONTRIBUTING.md`에 명시된 표현: *"pure TypeScript/JavaScript type safe client for Apache Kafka"*.
- **버전**: 2.5.0 — 이미 1.0을 넘긴 안정 버전. 즉 하위호환을 깨는 변경에 신중해야 하는 단계.

### 1.2 왜 이 프로젝트를 골랐나 (배경 스토리)

원래는 NestJS + Kafka 조합의 "깊이"를 증명하려고 `nest-native/kafka`라는 커뮤니티 프로젝트를 보고 있었다. 그런데 그 전에, "이게 NestJS 공식 재단이 인정한 라이브러리 맞나?"를 검증하려고 NestJS 저장소의 이슈 `nestjs/nest#13223`("KafkaJs seems not maintained anymore", 60개 이상 코멘트)를 처음부터 끝까지 추적했다.

그 결과 밝혀진 사실:
- **`kamilmysliwiec`(NestJS의 실제 창시자/코어 메인테이너)**가 이 스레드에 지속적으로 코멘트를 남기고 있었고, 본인이 직접 **`@platformatic/kafka`를 위한 커스텀 transport strategy**를 만들어서 커뮤니티에 공유하고 있었다.
- 2026-03-04, 누군가 "이거 지금 써도 되냐"고 묻자 kamilmysliwiec 본인이 **"that is the official recommendation"**이라고 답함.
- 반면 `confluent-kafka-javascript`에 대해서는 2025-05-20에 **"Confluent's wrapper appears to have some significant shortcomings at the moment"**이라고 부정적으로 언급함.
- 원래 후보였던 `nest-native/kafka`는 그 저자 본인이 스레드에 딱 한 번(2026-06-22) 자기 홍보성으로 남긴 코멘트일 뿐, NestJS 코어팀의 반응은 전무했음.

즉 "NestJS 생태계에서 실질적으로 미는 라이브러리"는 `nest-native/kafka`가 아니라 **`@platformatic/kafka`**라는 결론이 나서, 대상을 이쪽으로 바꿨다.

### 1.3 이 저장소의 이슈 선정 기준

`platformatic/kafka`는 열린 이슈가 4개뿐인 매우 건강한 저장소였고, 그중 **`#198`이 정확히 `good first issue` + `waiting-external-pr` 라벨이 동시에 붙어있었다.** `waiting-external-pr`은 "메인테이너가 외부인이 PR 내주길 명시적으로 기다린다"는 뜻이라, 방향성 자체를 반려당할 위험이 거의 없는 이슈였다.

### 1.4 관련 링크

| 구분 | 링크 |
|---|---|
| 원본 저장소 | https://github.com/platformatic/kafka |
| 내 Fork | https://github.com/GiHoon1123/kafka |
| 이슈 (#198) | https://github.com/platformatic/kafka/issues/198 |
| PR (#327) | https://github.com/platformatic/kafka/pull/327 |
| 작업 브랜치 | `feat/create-topics-per-topic-overrides` |

### 1.5 한 줄 요약

> `createTopics()`가 파티션 수/복제 팩터/파티션 배치를 호출 전체에 하나씩만 적용할 수 있던 것을, **토픽별로 다르게 지정할 수 있도록 확장**. 기존 `topics: string[]` 호출 방식은 완전히 그대로 유지(하위호환).

### 1.6 저장소 기본 정보 (2026-07 기준)

- 언어: 순수 TypeScript
- ⭐ 400+, 열린 이슈 4개 (매우 적게 유지되는 활발한 저장소)
- 최근 push: 며칠 단위로 매우 활발

---

## 2. 배경 지식

### 2.1 Kafka Admin API와 `createTopics`

Kafka 클라이언트는 크게 세 역할로 나뉜다: **Producer**(메시지 발행), **Consumer**(메시지 소비), **Admin**(토픽 생성/삭제, 컨슈머 그룹 관리 등 클러스터 관리 작업). `createTopics()`는 Admin API의 하나로, 브로커에게 "이런 이름의 토픽을 이런 설정으로 만들어줘"라고 요청하는 기능이다.

토픽을 만들 때 지정 가능한 주요 값들:
- **파티션 수(numPartitions)**: 토픽을 몇 개의 파티션으로 나눌지. 병렬 처리량과 직결됨.
- **복제 팩터(replicationFactor)**: 각 파티션을 몇 개의 브로커에 복제해둘지. 장애 내구성과 직결됨.
- **파티션 배치(assignments)**: 특정 파티션을 특정 브로커 조합에 수동으로 배치. 보통은 안 쓰고 브로커가 자동 배치하게 둠.
- **configs**: 토픽 단위 설정값(예: `cleanup.policy=compact`). 이번 PR과는 무관하지만 관련 문서(#329)에서 다룸.

### 2.2 이 라이브러리의 구조 — 순수 TS/JS 구현체

`confluent-kafka-javascript`(직전에 기여했던 라이브러리)와 결정적으로 다른 점: **네이티브 애드온이 없다.** Kafka와 통신하는 모든 바이트 단위 프로토콜 처리를 `src/apis/` 아래 TypeScript 코드로 직접 구현한다. 예를 들어 `src/apis/admin/create-topics-v7.ts`는 "CreateTopics API의 버전 7" 요청/응답 형식을 TS 인터페이스와 직렬화 함수로 그대로 표현한다:

```ts
export interface CreateTopicsRequestTopic {
  name: string
  numPartitions: number
  replicationFactor: number
  assignments: CreateTopicsRequestTopicAssignment[]
  configs: CreateTopicsRequestTopicConfig[]
}
```

이 인터페이스가 바로 Kafka 브로커에게 실제로 전송되는 요청의 "토픽 하나"에 해당하는 모양이다. **처음부터 토픽별로 다른 값을 가질 수 있게 설계돼 있었다** — 이게 이번 버그(제약)의 핵심 단서였다.

### 2.3 JSON Schema 검증 (ajv)와 `strict` 모드

이 라이브러리는 사용자가 넘긴 옵션 객체가 올바른 모양인지 **JSON Schema**로 미리 정의해두고, `ajv`(JSON Schema 검증 라이브러리)로 검사한다.

```ts
export const ajv = new Ajv2020({ allErrors: true, coerceTypes: false, strict: true })
```

그런데 중요한 사실 하나: 이 검증은 **클라이언트가 `{ strict: true }` 옵션으로 생성됐을 때만 실제로 동작**한다.

```ts
// src/clients/base/base.ts
[kValidateOptions] (target, validator, targetName, throwOnErrors = true): Error | null {
  if (!this[kOptions].strict) {
    return null   // strict가 아니면 검증 자체를 안 함
  }
  const valid = validator(target)
  ...
}
```

이건 이번 PR과는 직접 관련 없지만, 같은 파일을 보다가 발견한 사실이라 여기 기록해둔다 (자세한 내용은 PR #329 문서 참고).

### 2.4 TypeScript 오버로드 (복습)

`createTopics`는 콜백 스타일과 Promise 스타일을 동시에 지원한다:
```ts
createTopics (options: CreateTopicsOptions, callback: Callback<CreatedTopic[]>): void
createTopics (options: CreateTopicsOptions): Promise<CreatedTopic[]>
createTopics (options, callback?): void | Promise<CreatedTopic[]> {
  if (!callback) {
    callback = createPromisifiedCallback<CreatedTopic[]>()
  }
  ...
}
```
"이렇게 부를 수도, 저렇게 부를 수도 있다"를 여러 시그니처로 나열하는 TS 오버로드 패턴 — 이전 `confluent-kafka-javascript` 기여 때 배운 개념이 여기서도 그대로 쓰였다.

---

## 3. 이슈 분석 (#198)

### 3.1 이슈 제목과 요청 내용

**"Partitions configuration per topic"**

요청자는 kafkajs의 `admin.createTopics()`가 지원하는 방식(`ITopicConfig`: 토픽마다 `numPartitions`, `replicationFactor`, `replicaAssignment`, `configEntries`를 개별 지정 가능)을 원했다. 이슈에 직접 제안된 코드:
```ts
await admin.createTopics({
    topics: [{
        topic: 'topic',
        partitions: 10,
    }, ...],
});
```

### 3.2 원래 코드의 제약

`#createTopics` 내부 로직(`src/clients/admin/admin.ts`)을 열어보면:
```ts
#createTopics (options: CreateTopicsOptions, callback): void {
  const numPartitions = options.partitions ?? -1        // 전역값, 딱 한 번 계산
  const replicationFactor = options.replicas ?? -1      // 전역값, 딱 한 번 계산
  const assignments: CreateTopicsRequestTopicAssignment[] = []
  // ... options.assignments를 변환해서 assignments에 채움 (역시 전역, 한 번)

  const requests: CreateTopicsRequestTopic[] = []
  for (const topic of options.topics) {          // topics: string[] — 이름만 순회
    requests.push({
      name: topic,
      numPartitions,        // 모든 토픽에 동일한 값 재사용
      replicationFactor,    // 모든 토픽에 동일한 값 재사용
      assignments,          // 모든 토픽에 동일한 값 재사용
      configs
    })
  }
  ...
}
```

**핵심 발견**: 2.2절에서 본 `CreateTopicsRequestTopic` (와이어 프로토콜 레벨 타입)은 이미 "토픽 하나당 객체 하나" 구조라 토픽마다 다른 값을 담을 준비가 완벽히 돼 있다. 그런데 이 반복문이 **전역으로 미리 계산해둔 하나의 값을 모든 토픽에 복사**하고 있어서, 그 능력을 못 쓰고 있었다. 즉 프로토콜/네트워크 레이어의 한계가 아니라, **공개 API 레이어(`CreateTopicsOptions.topics: string[]`)가 "토픽 이름"만 받을 수 있어서 애초에 개별 설정을 전달할 방법 자체가 없었던 것.**

---

## 4. 설계 결정 — 두 가지 옵션 비교

같은 파일에 있는 `CreatePartitionsOptions`를 보면 이미 이런 패턴이 있었다:
```ts
export interface CreatePartitionsOptions {
  topics: CreatePartitionsRequestTopic[]   // 저수준 프로토콜 타입을 그대로 공개 API로 노출
  validateOnly?: boolean
}
```

### 옵션 A — `CreatePartitionsOptions`처럼 저수준 타입을 그대로 노출

```ts
topics: CreateTopicsRequestTopic[]   // 무조건 객체 배열
```
**문제**: 기존에 `topics: ['a', 'b']`로 호출하던 모든 사용자 코드가 깨진다. 이 라이브러리는 버전 `2.5.0`(1.0 이후 안정판)이라 이런 breaking change는 부적절하다고 판단.

### 옵션 B (채택) — 문자열/객체 유니온 타입

```ts
export interface CreateTopicsTopicOptions {
  topic: string
  partitions?: number
  replicas?: number
  assignments?: BrokerAssignment[]
}

export interface CreateTopicsOptions {
  topics: (string | CreateTopicsTopicOptions)[]
  ...
}
```

**장점**: 문자열로 주면 기존처럼 전역값 사용(하위호환 100% 유지), 객체로 주면 그 토픽만 개별 오버라이드. "단순한 경우는 단순하게, 복잡한 경우만 명시적으로" 확장하는 표준적인 TypeScript 패턴.

```ts
await admin.createTopics({
  topics: [
    'topic-with-defaults',                                    // 전역값 사용
    { topic: 'topic-with-overrides', partitions: 10, replicas: 3 }  // 개별 오버라이드
  ],
  partitions: 1,
  replicas: 1
})
```

---

## 5. 수정 내용

### 5.1 변경 파일 3개 (+테스트 1개)

| 파일 | 역할 |
|---|---|
| `src/clients/admin/types.ts` | `CreateTopicsTopicOptions` 신설, `CreateTopicsOptions.topics` 유니온 타입으로 확장 |
| `src/clients/admin/options.ts` | JSON Schema에 `oneOf`(문자열\|객체) 추가 |
| `src/clients/admin/admin.ts` | 실제 오버라이드 로직 + 헬퍼 메서드 분리 |
| `test/clients/admin/admin.test.ts` | 통합 테스트 1개 + strict mode 검증 테스트 3개 |

### 5.2 `types.ts`

```ts
export interface CreateTopicsTopicOptions {
  topic: string
  partitions?: number
  replicas?: number
  assignments?: BrokerAssignment[]
}

export interface CreateTopicsOptions {
  topics: (string | CreateTopicsTopicOptions)[]
  partitions?: number
  replicas?: number
  assignments?: BrokerAssignment[]
  configs?: CreateTopicsRequestTopicConfig[]
}
```

### 5.3 `options.ts` — JSON Schema

기존 `assignments` 스키마를 상수(`createTopicAssignmentsSchema`)로 추출해서, 전역용과 per-topic용에 **같은 스키마 객체를 재사용**하도록 만들었다 (중복 방지).

```ts
const createTopicAssignmentsSchema = {
  type: 'array',
  items: {
    type: 'object',
    properties: {
      partition: { type: 'number', minimum: 0 },
      brokers: { type: 'array', items: { type: 'number' }, minItems: 1 }
    },
    required: ['partition', 'brokers'],
    additionalProperties: false
  },
  minItems: 1
}

export const createTopicOptionsSchema = {
  type: 'object',
  properties: {
    topics: {
      type: 'array',
      items: {
        oneOf: [
          idProperty,   // 기존 문자열 검증 그대로 재사용
          {
            type: 'object',
            properties: {
              topic: idProperty,
              partitions: { type: 'number' },
              replicas: { type: 'number' },
              assignments: createTopicAssignmentsSchema
            },
            required: ['topic'],
            additionalProperties: false
          }
        ]
      }
    },
    partitions: { type: 'number' },
    replicas: { type: 'number' },
    assignments: createTopicAssignmentsSchema
  },
  required: ['topics'],
  additionalProperties: false
}
```

`oneOf`는 "둘 중 정확히 하나의 스키마에만 맞아야 한다"는 JSON Schema 키워드. 이 파일 안에 이미 `alterClientQuotasOptionsSchema` 등에서 쓰던 패턴이라 관례를 그대로 따랐다.

### 5.4 `admin.ts` — 핵심 로직

```ts
#mapTopicAssignments (assignments?: BrokerAssignment[]): CreateTopicsRequestTopicAssignment[] {
  const mapped: CreateTopicsRequestTopicAssignment[] = []
  for (const { partition, brokers } of assignments ?? []) {
    mapped.push({ partitionIndex: partition, brokerIds: brokers })
  }
  return mapped
}

#createTopics (options: CreateTopicsOptions, callback): void {
  const numPartitions = options.partitions ?? -1
  const replicationFactor = options.replicas ?? -1
  const assignments = this.#mapTopicAssignments(options.assignments)
  const configs = options.configs ?? []

  const requests: CreateTopicsRequestTopic[] = []
  for (const topic of options.topics) {
    if (typeof topic === 'string') {
      requests.push({ name: topic, numPartitions, replicationFactor, assignments, configs })
      continue
    }

    requests.push({
      name: topic.topic,
      numPartitions: topic.partitions ?? numPartitions,             // 없으면 전역값
      replicationFactor: topic.replicas ?? replicationFactor,        // 없으면 전역값
      assignments: topic.assignments ? this.#mapTopicAssignments(topic.assignments) : assignments,
      configs
    })
  }

  this[kPerformDeduplicated](
    `createTopics-${options.topics.map(topic => (typeof topic === 'string' ? topic : topic.topic)).join(',')}`,
    ...
  )
}
```

**부수적으로 고친 버그**: dedup 캐시 키를 만드는 부분이 원래 `options.topics.join(',')`였는데, `topics`가 이제 객체를 포함할 수 있으므로 그대로 두면 `[object Object]` 같은 쓰레기 문자열이 생성된다. `.map(topic => typeof topic === 'string' ? topic : topic.topic)`으로 먼저 이름만 뽑아내도록 수정.

**변수명 관련 주의점**: `#mapTopicAssignments`를 별도 메서드로 뽑아낸 이유는, 원래 코드에서 전역 `assignments`를 만드는 로직(for 루프)이 인라인으로 있었는데, per-topic에서도 똑같은 변환이 필요해서 중복을 피하려고 분리한 것.

### 5.5 테스트

**기능 테스트** (`createTopics should support per-topic overrides alongside global defaults`): 실제 로컬 Kafka 브로커에 문자열 토픽 하나 + 객체 토픽 하나를 동시에 생성 요청해서, 문자열 쪽은 전역값(`partitions: 2`)을, 객체 쪽은 개별값(`partitions: 5`)을 갖는지 검증.

**strict mode 검증 테스트 3개**:
1. 정상적인 per-topic 객체가 통과하는지
2. `topic` 필드 누락 시 거부되는지
3. 정의 안 된 프로퍼티(`invalidProperty`)가 있으면 거부되는지

---

## 6. 스코프 조정 히스토리 — 중요한 교훈

### 6.1 처음엔 `configs`도 같이 지원하려고 했다

이슈 #198이 요청한 4가지(파티션 수, 복제 팩터, 파티션 배치, **configs**) 중 `configs`를 구현하려고 코드를 보다가, **`createTopicOptionsSchema`에 `configs` 프로퍼티 자체가 통째로 빠져있는 것**을 발견했다 (이건 이 PR과 무관한 별개의 기존 버그였음 — 자세한 내용은 PR #329 문서). 처음엔 "어차피 같은 스키마 파일이니 김에 같이 고치자"고 판단해서, per-topic `configs`도 타입/스키마/로직에 다 넣고, 전역 `configs` 스키마 버그도 같이 고쳐서 테스트까지 다 통과시켰다.

### 6.2 되돌린 이유

작업을 다 마친 뒤 "이거 PR 통과할 수 있을까?"라는 질문을 스스로 던지다가, "원래 PR은 하나의 기능/목적만 다뤄야 하는 거 아닌가? 이거 나눠달라고 하거나 반려당할 것 같은데"라는 지적이 나왔다. 다시 생각해보니:

- `configs` 버그는 **발생 원인이 완전히 다름** (2025년 PR #75가 `configs` 기능을 추가하면서 스키마 업데이트를 빠뜨린 것 — 우리 이슈 #198과 무관)
- **영향 범위가 다름** (전역 `configs` 옵션 전체에 영향, per-topic 기능 유무와 무관하게 이미 존재하던 문제)
- "같은 스키마 파일을 건드린다"는 **우연히 위치가 겹치는 것**이지 "같은 목적의 변경"이라는 원칙적 근거가 아니었음

### 6.3 되돌린 작업

- `CreateTopicsTopicOptions`에서 `configs` 필드 제거
- 스키마에서 전역/per-topic `configs` 추가 부분 전부 되돌림 (원래 버그 상태로 복원 — 별도 PR에서 고치기 위해)
- `admin.ts`에서 `configs: topic.configs ?? configs` → 그냥 `configs`(전역값만 사용)로 되돌림
- 테스트에서 `configs` 검증 부분 제거, 관련 테스트 하나는 "per-topic 객체가 strict mode에서 통과하는지"로 목적을 바꿔서 재활용

이 판단과 실제 작업 결과, `configs` 버그는 **PR #329**로 완전히 분리해서 별도 제출했다 (해당 문서 참고).

---

## 7. 기여 프로세스

### 7.1 Fork & Clone

```bash
gh repo fork platformatic/kafka --clone
```
`GiHoon1123/kafka` fork 생성, `origin`(fork)/`upstream`(원본) 리모트 자동 설정. `open-source/` 디렉토리 안이라 git 계정도 자동으로 `Gihoon1123`으로 적용됨 (이전 `confluent-kafka-javascript` 기여 때 만들어둔 디렉토리별 git 설정 덕분).

### 7.2 브랜치 관리 — 실수 기록

**솔직한 기록**: 코드 수정을 시작할 때 브랜치를 새로 만드는 걸 깜빡하고 `main`에서 바로 작업했다. 커밋 직전에 `git status`로 확인하다가 발견해서, 뒤늦게 `git checkout -b feat/create-topics-per-topic-overrides`로 브랜치를 만들고 진행했다. (이후 PR #329 작업 때는 처음부터 브랜치를 만들고 시작하도록 교훈을 적용함.)

### 7.3 개발 환경 세팅

```bash
pnpm install                # 순수 TS/JS라 네이티브 컴파일 불필요, 빠름
docker compose up -d        # 로컬 Kafka 클러스터 (3-broker cluster, SASL, SASL+Kerberos, single-broker, schema registry 등 총 9개 컨테이너)
```

### 7.4 커밋 컨벤션

`CONTRIBUTING.md`에 명시된 두 가지 필수 요건:

**(1) DCO(Developer Certificate of Origin) 서명**:
```bash
git commit -s -m "..."
```
`-s` 플래그가 자동으로 `Signed-off-by: Gihoon1123 <...>` 트레일러를 추가.

**(2) AI 도움받았으면 명시**: *"If working with AI assistance, ensure both contributors are signed off."*

이 요구가 형식적인지 실제로 지켜지는지 궁금해서 **실제 커밋 히스토리를 직접 검색**했다:
```bash
git log upstream/main --format="%B" | grep -i "co-authored-by" 
```
결과: AI 코딩 도구를 가리키는 co-author 트레일러가 실제로 **Matteo Collina, Paolo Insogna 같은 핵심 메인테이너들의 이미 머지된 커밋**에 들어있는 것을 확인. 즉 이 요구는 형식적 문구가 아니라 실제로 지켜지는 프로젝트 문화였음. 같은 형식을 그대로 따라서 커밋:

```
feat(admin): support per-topic overrides in createTopics

createTopics() currently applies partitions, replicas, and assignments
globally to every topic in a single call, even though the underlying
CreateTopics protocol request already supports different values per
topic.

Extend the topics array to accept either a plain topic name (using the
call's global defaults, unchanged behavior) or an object specifying
per-topic partitions/replicas/assignments overrides, matching the
usage kafkajs exposes.

Fixes #198.

Co-Authored-By: AI Assistant <ai-assistant@example.com>
Signed-off-by: Gihoon1123 <rlaejrqo465@naver.com>
```

### 7.5 로컬 검증 — 제출 전 필수 게이트

`CONTRIBUTING.md`가 요구하는 것보다 한 단계 더 철저하게, PR 만들기 전에 **전체 CI 파이프라인을 로컬에서 미리 통과**시켰다:
```bash
npm run ci   # = npm run build && npm run lint && npm run test:ci
```
- 빌드, 린트, **전체 1804개 테스트**, **커버리지 90% 이상 강제**(`--test-coverage-lines=90 --test-coverage-branches=90 --test-coverage-functions=90`)
- 결과: 전체 통과, 우리가 새로 만든 코드(`types.ts`, `options.ts`)는 **100% 커버리지**

### 7.6 PR 생성

```bash
gh pr create --repo platformatic/kafka \
  --title "feat(admin): support per-topic overrides in createTopics" \
  --body "Fixes #198. ..."
```
→ https://github.com/platformatic/kafka/pull/327

`CONTRIBUTING.md`의 관례대로 PR 본문 맨 위에 `Fixes #198.`을 명시.

### 7.7 Code Review 자동화 체크

PR을 열자마자 **DCO 체크가 즉시 초록불로 통과**됨(별도 봇이 자동 검사). 반면 실제 테스트를 도는 GitHub Actions("Run Tests")는 `conclusion: action_required` 상태 — **처음 기여하는 외부인의 PR은 보안상 자동으로 CI가 안 돌고, 메인테이너가 수동으로 "Approve and run workflow"를 눌러야 실행된다.** 코드 문제가 아니라 정상적인 보안 절차.

---

## 8. 트러블슈팅

### 8.1 (참고) Docker 브로커 컨테이너가 죽었던 사건

이 PR 작업 도중에는 발생하지 않았지만, 같은 로컬 환경을 계속 쓰다가 **다음 작업(PR #329)에서 `broker-cluster-3` 컨테이너가 OOM으로 죽는 사건**이 있었다. 자세한 원인/해결 과정은 PR #329 문서에 기록했지만, 이 PR의 테스트들("createTopics should retarget controller when needed" 등 3-broker 클러스터에 의존하는 테스트 포함)도 만약 그 시점에 실행했다면 같은 영향을 받았을 것이라 여기도 참고로 남겨둔다. **로컬 통합 테스트 환경(특히 다중 브로커 클러스터)은 오래 띄워두면 리소스 문제로 죽을 수 있으니, 테스트 실패 시 "내 코드 문제"로 바로 단정하지 말고 `docker compose ps -a`로 컨테이너 상태부터 확인하는 습관이 중요.**

### 8.2 typecheck의 무관한 기존 에러

`pnpm run typecheck` 실행 시 다음 두 에러가 항상 뜬다:
```
test/apis/admin/create-topics-v7.test.ts(453,6): error TS2554: Expected 2-4 arguments, but got 1.
test/clients/consumer/consumer.test.ts(4275,5): error TS2322: ...
```
`git stash`로 우리 변경을 임시로 치워두고 동일 명령을 돌려서, **이 에러들이 우리 변경과 무관하게 원본 `main`에도 이미 존재**한다는 걸 확인했다. 이런 식으로 "내가 만든 에러인지 원래 있던 에러인지"를 `git stash` 비교로 검증하는 습관을 들였다.

---

## 9. 결과 및 회고

### 9.1 PR 상태

> 이 섹션은 결과가 나오는 대로 업데이트 예정.

- **현재 (작성 시점, 2026-07-08)**: PR #327, Open, DCO 통과, 테스트 CI는 메인테이너 수동 승인 대기 중(`action_required`), 리뷰/코멘트 없음
- **2026-07-28**: 3주 넘게 무응답이라 "When you have a chance, could you take a look? Thanks!"라는 리마인드 코멘트를 남김.

**갱신 (2026-08-18)**: 메인테이너 `ShogunPanda`가 처음 반응 — "Can you also update docs and rebase? After that I'll be happy to merge." (PR #329보다 진전된 반응 — 머지 의사를 명확히 밝힘). 대응한 내용:
1. `upstream/main` 기준 rebase (충돌 없음)
2. `docs/admin.md`의 `createTopics` 옵션 표를 갱신 — `topics`가 이제 `(string | CreateTopicsTopicOptions)[]`를 받는다는 것과, 객체 형태로 넘기면 `partitions`/`replicas`/`assignments`를 토픽별로 오버라이드한다는 설명 추가 (기존 `assignments`/`configs` 행과 같은 스타일 유지, `prettier`로 표 정렬)
3. `test/clients/admin/admin.test.ts` 169개 전부 통과, `lint`/`typecheck` 확인(타입체크의 기존 무관 에러 2건은 8.2절 그대로 — `git stash` 비교로 이번 변경과 무관함 재확인)
4. fork에 force-push, PR에 완료 코멘트 남김

**최종 결과 (2026-08-20)**: 메인테이너 `ShogunPanda`가 PR #327을 머지했다. 이슈 #198에서 요청한 토픽별 `partitions`/`replicas`/`assignments` 오버라이드 지원이 `main`에 반영됐다.

### 9.2 배운 점

- **API 설계 관점**: 기존 공개 API(`topics: string[]`)를 깨지 않으면서 기능을 확장하는 표준 패턴(문자열/객체 유니온 타입)을 실전에 적용해봄
- **범위 관리(scope discipline)**: "같은 파일을 고친다"는 "같은 목적"의 근거가 안 된다는 걸 뼈아프게 배움. 처음엔 편의상 묶었다가, 원칙에 따라 다시 풀어내는 과정을 실제로 겪어봄
- **관례를 문서가 아니라 커밋 히스토리로 직접 검증하는 습관**: `CONTRIBUTING.md`에 적힌 규칙이 실제로 지켜지는지, `git log --grep`으로 직접 확인하는 습관
- **로컬 사전 검증의 가치**: PR 올리기 전에 전체 CI를 미리 로컬에서 통과시켜두면, 실제 리뷰 단계에서 코드 품질 논쟁 없이 설계 논의에만 집중할 수 있음

### 9.3 다음 계획

- 같은 작업에서 분리한 `configs` 스키마 버그 수정도 **PR #329로 머지됨** (별도 문서 `platformatic-kafka-pr329-configs-schema-bug.md` 참고)
- 이후에는 consumer 쪽처럼 더 깊은 Kafka 개념이 필요한 이슈를 다음 기여 후보로 검토
