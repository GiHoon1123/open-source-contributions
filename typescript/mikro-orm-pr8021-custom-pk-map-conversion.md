# [ORM OSS 기여] mikro-orm/mikro-orm - `em.map()`에서 DB-form custom primary key를 다시 DB-form으로 변환하던 버그 수정 (PR #8021) — 🟢 Merged

> 오픈소스 기여 기록 및 학습 노트. 이 문서는 PR #8021 하나만 다룬다. **최초 작성 시점 OPEN, 2026-07-27 19:27 메인테이너 `B4nan`이 승인과 동시에 MERGED. 12절 참고.**
>
> 이번 건은 "DB 값과 JS 값이 다를 수 있다"는 custom type 개념 자체보다, 그 값이 ORM 내부의 `map()` / `merge()` / identity map 조회 경로를 지나며 **어느 형식으로 취급되어야 하는지**를 끝까지 따라가는 게 핵심이었다.

---

## 1. 개요

### 1.1 프로젝트 소개

- **저장소**: https://github.com/mikro-orm/mikro-orm
- **npm 패키지**: `@mikro-orm/*`
- **성격**: TypeScript ORM. Data Mapper, Unit of Work, Identity Map 패턴을 기반으로 MongoDB, MySQL, PostgreSQL, SQLite 등을 지원한다.
- **왜 이 타겟을 골랐나**:
  - NestJS/Node.js 백엔드 개발자 입장에서 ORM 내부는 실무와 직접 연결된다.
  - 이슈가 2026-07-23에 올라온 신선한 버그였고, 경쟁 PR이 없었다.
  - MikroORM은 최근 외부 PR도 실제로 머지되고 있었다. 예: `xianjianlf2`의 PR #8004가 2026-07-18에 머지됨.
  - 이슈 재현이 작고, 원인이 `EntityManager.merge()`와 `UnitOfWork.tryGetById()` 사이의 옵션 전달 누락으로 좁혀졌다.

### 1.2 이슈 (#8020)

**제목**: `em.map calls convertToDatabaseValue on PK value in database format`

이슈의 핵심은 이렇다:

`em.map()`은 raw DB result를 entity로 매핑하는 API다. 그래서 input data는 이미 DB-form이다. 그런데 custom type primary key가 있는 entity를 `em.map()`으로 매핑하면, 이미 DB-form인 PK가 `convertToDatabaseValue()`에 다시 들어가며 에러가 난다.

이슈 재현용 custom type은 일부러 JS-form과 DB-form을 명확히 구분한다:

```text
JS-form: js-1
DB-form: db-1
```

정상 변환 방향은 다음과 같다:

```text
convertToDatabaseValue('js-1') -> 'db-1'
convertToJSValue('db-1')       -> 'js-1'
```

그런데 버그 상태에서는 다음처럼 잘못 호출됐다:

```text
convertToDatabaseValue('db-1')
```

즉 이미 DB-form인 값을 JS-form이라고 착각하고 다시 DB-form으로 변환하려 한 것이다.

### 1.3 관련 링크

| 구분 | 링크 |
|---|---|
| 원본 저장소 | https://github.com/mikro-orm/mikro-orm |
| 내 Fork | https://github.com/GiHoon1123/mikro-orm |
| 이슈 (#8020) | https://github.com/mikro-orm/mikro-orm/issues/8020 |
| PR (#8021) | https://github.com/mikro-orm/mikro-orm/pull/8021 |
| 작업 브랜치 | `fix-8020-map-custom-pk-db-value` |
| 커밋 | `f310b2db45a4f17ab341d3e58b68b767e96d1caa` |
| 상태 | 🟢 MERGED (2026-07-27) |

### 1.4 한 줄 요약

> `em.map()`은 raw DB row를 entity로 바꾸는 API라 `convertCustomTypes: true`를 들고 `merge()`를 호출한다. 그런데 `merge()`가 identity map에서 기존 entity를 먼저 찾는 `tryGetById()` 경로에는 그 플래그를 넘기지 않아, 이미 DB-form인 custom primary key가 JS-form처럼 취급되고 `convertToDatabaseValue()`에 다시 들어갔다. `convertCustomTypes` 플래그를 `merge()` → `tryGetById()` → `getById()`까지 전달해서 고쳤다.

---

## 2. 배경 지식

### 2.1 ORM은 DB row와 JS 객체 사이에서 번역한다

ORM은 DB에 있는 row와 애플리케이션 코드에서 쓰는 객체 사이를 오간다.

DB row는 이런 모양일 수 있다:

```text
authors table

id
----
db-1
```

JS 코드에서는 entity 객체로 다룬다:

```ts
author.id === 'js-1'
```

보통은 DB 값과 JS 값이 같은 문자열일 수도 있지만, custom type을 쓰면 둘이 달라질 수 있다.

실무 예시:

```text
JS 코드: UUID 문자열
DB: Buffer
```

또는:

```text
JS 코드: Date 객체
DB: timestamp 문자열
```

이번 이슈 재현은 이해를 쉽게 하려고 다음처럼 일부러 구분했다:

```text
JS 값: js-1
DB 값: db-1
```

### 2.2 custom type의 두 변환 방향

MikroORM의 custom type은 대략 두 방향의 변환 함수를 가진다.

#### JS -> DB

```ts
convertToDatabaseValue(value)
```

예:

```text
js-1 -> db-1
```

#### DB -> JS

```ts
convertToJSValue(value)
```

예:

```text
db-1 -> js-1
```

중요한 점:

```text
convertToDatabaseValue()는 JS-form을 받아야 한다.
convertToJSValue()는 DB-form을 받아야 한다.
```

이번 이슈에서는 `convertToDatabaseValue()`에 이미 DB-form인 `db-1`이 들어가서 터졌다.

### 2.3 `em.map()`은 무엇인가

`em.map()`은 raw DB result를 MikroORM entity로 바꾸는 API다.

예:

```ts
const row = { id: 'db-1' };
const author = em.map(Author, row);
```

여기서 `row.id`는 DB에서 온 값이므로 DB-form이다. 따라서 정상 결과는 다음이어야 한다:

```ts
author.id === 'js-1';
```

즉 `em.map()`의 역할은:

```text
DB row { id: 'db-1' }
-> Author entity { id: 'js-1' }
```

이다.

### 2.4 identity map은 무엇인가

MikroORM은 같은 DB row에 대해 entity 객체를 계속 새로 만들지 않는다.

예를 들어 같은 row를 두 번 매핑한다:

```ts
const a1 = em.map(Author, { id: 'db-1' });
const a2 = em.map(Author, { id: 'db-1' });
```

둘은 같은 DB row이므로 같은 객체여야 한다:

```ts
a1 === a2;
```

그래야 같은 row인데 메모리에 객체가 여러 개 생기는 문제가 없다.

예를 들어 객체가 둘이면 이런 상황이 생긴다:

```ts
a1.name = 'Kim';
a2.name = 'Lee';
```

같은 DB row인데 메모리 객체가 둘이라면, ORM은 어떤 상태가 진짜인지 판단하기 어렵다. 그래서 내부에 이미 만든 entity를 보관하는 저장소가 있다. 이게 identity map이다.

대략 이런 저장소라고 보면 된다:

```text
Author + primary key hash -> Author 객체
```

이번 이슈에서는 이 identity map 조회 과정에서 PK 형식 정보가 끊겼다.

---

## 3. 이슈 재현

### 3.1 이슈의 재현 custom type

이슈의 gist는 다음과 같은 custom type을 사용한다.

```ts
class RemappingType extends Type<string | null | undefined> {
  override convertToDatabaseValue(value: string | null | undefined): string | null | undefined {
    if (typeof value !== 'string') {
      return value;
    }

    if (!value.startsWith('js-')) {
      throw new Error(`Unexpected value for RemappingType.convertToDatabaseValue: ${value}.`);
    }

    return value.replace(/^js-/, 'db-');
  }

  override convertToJSValue(value: string | null | undefined): string | null | undefined {
    if (typeof value !== 'string') {
      return value;
    }

    if (!value.startsWith('db-')) {
      throw new Error(`Unexpected value for RemappingType.convertToJSValue: ${value}.`);
    }

    return value.replace(/^db-/, 'js-');
  }
}
```

이 custom type은 일부러 매우 엄격하다.

- `convertToDatabaseValue()`는 `js-`로 시작하는 값만 받는다.
- `convertToJSValue()`는 `db-`로 시작하는 값만 받는다.

그래서 잘못된 방향으로 변환이 일어나면 바로 에러가 난다.

### 3.2 재현 코드

```ts
const rawAuthor = {
  id: 'db-1',
};

em.map(Author, rawAuthor);
```

기대:

```text
db-1 -> js-1
```

실제 master에서 재현한 에러:

```text
Unexpected value for RemappingType.convertToDatabaseValue: db-1.
```

스택은 다음 경로로 떨어졌다:

```text
RemappingType.convertToDatabaseValue
  -> UnitOfWork.getById
  -> UnitOfWork.tryGetById
  -> EntityManager.merge
  -> EntityManager.map
```

이 스택이 핵심이었다. `em.map()`이 entity를 만들기 전에 `merge()`를 호출하고, `merge()`가 기존 entity를 찾으려고 `tryGetById()`를 부르며, 그 내부에서 `getById()`가 잘못된 변환을 시도했다.

---

## 4. 내부 흐름 이해

### 4.1 `em.map()`의 내부 흐름

문제의 시작은 `EntityManager.map()`이다.

`packages/core/src/EntityManager.ts`의 `map()`은 raw result를 받아 driver의 `mapResult()`를 거친 뒤 `merge()`를 호출한다.

핵심 부분:

```ts
return this.merge<Entity>(entityName, data as EntityData<Entity>, {
  convertCustomTypes: true,
  refresh: true,
  validate: false,
  ...rest,
});
```

여기서 `convertCustomTypes: true`가 중요하다.

이 플래그의 의미:

```text
지금 들어온 data는 DB-form이다.
custom type 값을 JS-form으로 변환해서 entity에 넣어라.
```

즉 `em.map()`은 `db-1`을 `js-1`로 바꾸는 게 맞다.

### 4.2 `merge()`는 새 entity를 만들기 전에 identity map을 먼저 확인한다

`merge()`는 대략 다음 역할을 한다:

```text
1. 이 PK를 가진 entity가 이미 identity map에 있는지 확인한다.
2. 있으면 그 entity를 재사용한다.
3. 없으면 새 entity를 만든다.
4. data를 entity에 hydrate한다.
5. entity를 UnitOfWork/identity map에 등록한다.
```

기존 `merge()`의 문제 지점:

```ts
let entity = em.#unitOfWork.tryGetById<Entity>(
  entityName,
  data as FilterQuery<Entity>,
  options.schema,
  false,
);
```

여기서 `options.convertCustomTypes`를 넘기지 않았다.

즉 `merge()`는 다음 정보를 알고 있었다:

```text
이 data는 DB-form이다.
```

하지만 `tryGetById()`에는 그 정보를 전달하지 않았다.

### 4.3 `tryGetById()`는 PK만 뽑고 `getById()`로 넘긴다

기존 `tryGetById()`:

```ts
tryGetById<T extends object>(
  entityName: EntityName<T>,
  where: FilterQuery<T>,
  schema?: string,
  strict = true,
): T | null {
  const pk = Utils.extractPK(where, this.#metadata.find<T>(entityName), strict);

  if (!pk) {
    return null;
  }

  return this.getById<T>(entityName, pk as Primary<T>, schema)!;
}
```

이 함수는 `where`에서 PK를 뽑은 뒤 `getById()`로 넘긴다.

문제는 `getById()`가 이미 `convertCustomTypes` 옵션을 받을 수 있는 함수였다는 점이다. 즉 낮은 레벨에는 이미 필요한 기능이 있었다. 중간 함수인 `tryGetById()`가 그 옵션을 받지 않아서 정보가 끊겼을 뿐이다.

### 4.4 `getById()`는 PK hash를 만든다

`UnitOfWork.getById()`는 identity map에서 entity를 찾기 위해 PK hash를 만든다.

대략 이런 역할이다:

```text
Author + primary key hash -> managed Author entity
```

custom type이 있는 복합 PK에서는 `convertCustomTypes` 값에 따라 변환 여부를 다르게 처리한다:

```ts
if (!convertCustomTypes && p.customType) {
  return p.customType.convertToDatabaseValue(keys[i], this.#platform, {
    key: p.name,
    mode: 'hydration',
  });
}
```

여기서 의미는:

- `convertCustomTypes`가 false 또는 undefined이면: 들어온 값이 JS-form이라고 보고 DB-form으로 바꿔 hash를 만든다.
- `convertCustomTypes`가 true이면: 들어온 값이 이미 DB-form이라고 보고 다시 변환하지 않는다.

이번 버그는 `em.map()`이 DB-form 값을 넘겼는데, `convertCustomTypes`가 `getById()`까지 전달되지 않아서 `undefined`로 취급된 것이다.

결과:

```text
db-1을 JS-form이라고 착각
-> convertToDatabaseValue('db-1') 호출
-> 에러
```

---

## 5. 근본 원인

### 5.1 값 자체가 틀린 게 아니라 "값의 형식 정보"가 끊겼다

이번 버그는 `convertToDatabaseValue()`나 `convertToJSValue()` 구현이 틀린 문제가 아니다.

문제는:

```text
이 값이 JS-form인지 DB-form인지 알려주는 플래그가 중간 함수에서 끊겼다.
```

흐름으로 보면:

```text
em.map()
  -> raw DB result를 받음
  -> convertCustomTypes: true 설정

merge()
  -> 아직 convertCustomTypes: true를 알고 있음

tryGetById()
  -> 기존 코드에서는 그 정보를 못 받음

getById()
  -> convertCustomTypes가 undefined라서
     "이 PK는 JS-form일 것"이라고 보고 DB-form으로 변환 시도
```

### 5.2 왜 기본값은 JS-form으로 보는가

대부분의 ORM API는 개발자가 JS 코드에서 직접 호출한다.

예를 들어 일반적인 `merge()`:

```ts
em.merge(Author, { id: 'js-1' });
```

이 경우 `id`는 JS 코드에서 다루는 값이므로 JS-form이다. identity map의 key/hash를 DB-form 기준으로 맞춰야 한다면 다음 변환이 필요하다:

```text
js-1 -> db-1
```

그래서 `getById()`가 `convertCustomTypes`를 받지 못하면 기본적으로 JS-form처럼 취급하는 것은 자연스럽다.

하지만 `em.map()`은 다르다:

```ts
em.map(Author, { id: 'db-1' });
```

`em.map()`은 raw DB result를 받으므로 `id`가 이미 DB-form이다. 이 경우에는 다시 변환하면 안 된다.

### 5.3 한 문장으로

> `em.map()`으로 들어온 primary key는 이미 DB-form인데, `merge()`가 identity map 조회를 위해 `tryGetById()`를 호출할 때 `convertCustomTypes: true`를 넘기지 않아 그 값이 JS-form처럼 취급됐고, 결국 `convertToDatabaseValue()`가 한 번 더 호출됐다.

---

## 6. 수정 내용

### 6.1 `UnitOfWork.tryGetById()`에 optional 인자 추가

수정 전:

```ts
tryGetById<T extends object>(
  entityName: EntityName<T>,
  where: FilterQuery<T>,
  schema?: string,
  strict = true,
): T | null {
  const pk = Utils.extractPK(where, this.#metadata.find<T>(entityName), strict);

  if (!pk) {
    return null;
  }

  return this.getById<T>(entityName, pk as Primary<T>, schema)!;
}
```

수정 후:

```ts
tryGetById<T extends object>(
  entityName: EntityName<T>,
  where: FilterQuery<T>,
  schema?: string,
  strict = true,
  convertCustomTypes?: boolean,
): T | null {
  const pk = Utils.extractPK(where, this.#metadata.find<T>(entityName), strict);

  if (!pk) {
    return null;
  }

  return this.getById<T>(entityName, pk as Primary<T>, schema, convertCustomTypes)!;
}
```

핵심:

```ts
convertCustomTypes?: boolean
```

을 맨 뒤 optional parameter로 추가하고, 기존에 이미 이 옵션을 지원하던 `getById()`로 그대로 넘겼다.

### 6.2 `EntityManager.merge()`에서 플래그 전달

수정 전:

```ts
let entity = em.#unitOfWork.tryGetById<Entity>(
  entityName,
  data as FilterQuery<Entity>,
  options.schema,
  false,
);
```

수정 후:

```ts
let entity = em.#unitOfWork.tryGetById<Entity>(
  entityName,
  data as FilterQuery<Entity>,
  options.schema,
  false,
  options.convertCustomTypes,
);
```

이제 `em.map()`에서 시작된 `convertCustomTypes: true` 정보가 다음 흐름까지 전달된다:

```text
em.map()
  -> merge(..., { convertCustomTypes: true })
  -> tryGetById(..., convertCustomTypes: true)
  -> getById(..., convertCustomTypes: true)
```

### 6.3 왜 이 수정이 작고 안전한가

1. **public API 변경이 아니다.**
   - `tryGetById()`는 내부 `UnitOfWork` 메서드다.

2. **새 인자는 optional이고 맨 뒤에 붙었다.**
   - 기존 호출자는 그대로 동작한다.

3. **호출자는 현재 두 군데뿐이다.**
   - `findOne()`의 기존 호출은 그대로다.
   - `merge()`만 `options.convertCustomTypes`를 넘기도록 바뀌었다.

4. **낮은 레벨 함수인 `getById()`는 이미 `convertCustomTypes`를 지원했다.**
   - 새로운 변환 규칙을 만든 게 아니라, 원래 있던 옵션이 중간에서 끊기지 않게 전달한 것이다.

---

## 7. 회귀 테스트

### 7.1 테스트 파일

추가한 파일:

```text
tests/issues/GH8020.test.ts
```

MikroORM은 기존에도 이슈별 회귀 테스트를 `tests/issues/GHxxxx.test.ts` 형태로 많이 두고 있다. 최근 예:

- `tests/issues/GH8003.test.ts`
- `tests/issues/GH8000.test.ts`
- `tests/issues/GH7966.test.ts`

그래서 새 테스트도 같은 위치와 네이밍을 따랐다.

### 7.2 테스트 entity

```ts
const AuthorSchema = defineEntity({
  name: 'Author',
  properties: {
    id: p.type(RemappingType).primary(),
  },
});

class Author extends AuthorSchema.class {}
AuthorSchema.setClass(Author);
```

primary key 자체가 custom type이다. 이게 중요하다. 이슈는 일반 필드가 아니라 **custom type primary key**에서 터진다.

### 7.3 테스트 본문

```ts
test('GH #8020: em.map does not convert database-form custom primary key again', () => {
  const em = orm.em.fork();
  const author = em.map(Author, { id: 'db-1' });
  const authorAgain = em.map(Author, { id: 'db-1' });

  expect(author.id).toBe('js-1');
  expect(authorAgain).toBe(author);
});
```

이 테스트는 두 가지를 확인한다.

#### 첫 번째 확인

```ts
expect(author.id).toBe('js-1');
```

DB-form `db-1`이 entity 내부에서 JS-form `js-1`로 hydrate됐는지 확인한다.

수정 전에는 여기까지 오기도 전에 다음 에러로 실패했다:

```text
Unexpected value for RemappingType.convertToDatabaseValue: db-1.
```

#### 두 번째 확인

```ts
expect(authorAgain).toBe(author);
```

같은 DB-form PK를 다시 `em.map()`했을 때 같은 entity 객체를 돌려주는지 확인한다.

이건 identity map 경로를 직접 검증한다. 단순히 "에러가 안 난다"만 보는 테스트였다면, `tryGetById()` 경로가 실제로 제대로 동작하는지 확신하기 어려웠을 것이다. 같은 PK 재매핑에서 같은 객체가 나온다는 건 `merge()`의 identity map lookup이 올바른 key 형식으로 작동했다는 뜻이다.

### 7.4 소스 수정 제거 시 실패 확인

검증 과정에서 source fix만 잠깐 제거한 상태로 새 테스트를 돌렸다. 결과는 의도대로 실패했다:

```text
FAIL tests/issues/GH8020.test.ts
Error: Unexpected value for RemappingType.convertToDatabaseValue: db-1.

RemappingType.convertToDatabaseValue
  -> UnitOfWork.getById
  -> UnitOfWork.tryGetById
  -> EntityManager.merge
  -> EntityManager.map
```

즉 이 테스트는 실제로 #8020 버그를 잡는다. 이게 회귀 테스트의 핵심이다.

---

## 8. 검증

### 8.1 단독 회귀 테스트

```sh
yarn test tests/issues/GH8020.test.ts
```

결과:

```text
Test Files  1 passed (1)
Tests       1 passed (1)
```

### 8.2 관련 기존 suite

`merge()`와 identity map 영향이 있으므로 기존 SQLite EntityManager suite도 돌렸다.

```sh
yarn test tests/features/entity-manager/EntityManager.sqlite.test.ts
```

결과:

```text
Test Files  1 passed (1)
Tests       199 passed (199)
```

이후 새 회귀 테스트와 함께 한 번 더 돌림:

```sh
yarn test tests/issues/GH8020.test.ts tests/features/entity-manager/EntityManager.sqlite.test.ts
```

결과:

```text
Test Files  2 passed (2)
Tests       200 passed (200)
```

### 8.3 build / format / lint

```sh
yarn build
```

결과:

```text
Done
```

```sh
yarn oxfmt packages/core/src/EntityManager.ts packages/core/src/unit-of-work/UnitOfWork.ts tests/issues/GH8020.test.ts --check
```

결과:

```text
All matched files use the correct format.
```

```sh
yarn oxlint packages/core/src/EntityManager.ts packages/core/src/unit-of-work/UnitOfWork.ts tests/issues/GH8020.test.ts --tsconfig=tsconfig.json --type-aware --type-check
```

결과:

```text
passed
```

### 8.4 테스트 종료 경고

Vitest 실행 후 다음 경고가 출력됐다:

```text
close timed out after 10000ms
Tests closed successfully but something prevents the main process from exiting
```

하지만 exit code는 0이었고, 새 테스트만이 아니라 기존 `EntityManager.sqlite.test.ts`에서도 같은 경고가 출력됐다. 따라서 이번 변경으로 생긴 실패나 회귀로 보지는 않았다.

---

## 9. 자체 검토

### 9.1 메인테이너 컨벤션 확인

MikroORM은 최근에도 작은 core bugfix에 대해 다음 패턴을 많이 쓴다:

```text
fix(core): ...
tests/issues/GHxxxx.test.ts 추가
```

최근 예:

- `fix(core): populate 1:m collections of parents with falsy PK values (#8002)`
- `fix(core): fix infinite recursion when flushing a mapToPk FK with value 0 (#8004)`
- `fix(core): keep custom-typed PK snapshots in DB-form to avoid spurious updates (#7973)`

이번 PR도 같은 패턴이다:

```text
fix(core): respect custom type conversion when merging mapped data
tests/issues/GH8020.test.ts
```

### 9.2 수정 범위 점검

`tryGetById()` 호출자는 현재 두 군데였다:

```text
EntityManager.findOne()
EntityManager.merge()
```

우리가 바꾼 건 `merge()` 호출뿐이다.

`findOne()`은 그대로:

```ts
em.#unitOfWork.tryGetById(entityName, where, options.schema);
```

새 인자는 optional이라 기존 호출이 깨지지 않는다.

### 9.3 왜 `tryGetById()`에 인자를 추가하는 게 맞는가

처음엔 "너무 낮은 레벨 API에 옵션을 흘리는 것 아닌가?"를 의심했다.

하지만 이미 `getById()`는 `convertCustomTypes`를 받고 있었다:

```ts
getById<T extends object>(
  entityName: EntityName<T>,
  id: Primary<T> | Primary<T>[],
  schema?: string,
  convertCustomTypes?: boolean,
): T | undefined
```

즉 낮은 레벨에는 이미 필요한 옵션이 존재했다. 문제는 중간 함수인 `tryGetById()`가 그 옵션을 받지 않아 전달이 끊긴 것이다.

그래서 `tryGetById()`에 optional 인자를 추가해 `getById()`로 넘기는 게 가장 작은 수정이다. 다른 곳에서 PK를 미리 변환하거나, `merge()`에서 별도 우회 로직을 넣는 것보다 기존 구조에 더 잘 맞는다.

### 9.4 테스트 범위 점검

추가 테스트는 public API인 `em.map()`을 기준으로 한다.

내부 원인은 `merge()`지만, 이슈가 신고한 public entrypoint는 `em.map()`이므로 테스트도 이 사용 경로를 잡는 게 맞다.

다만 테스트 안에서 같은 DB-form PK를 두 번 `em.map()`한다:

```ts
const author = em.map(Author, { id: 'db-1' });
const authorAgain = em.map(Author, { id: 'db-1' });

expect(authorAgain).toBe(author);
```

이렇게 해서 내부 identity map lookup 경로도 같이 검증한다.

---

## 10. PR

### 10.1 커밋

```text
f310b2db45a4f17ab341d3e58b68b767e96d1caa
fix(core): respect custom type conversion when merging mapped data
```

커밋 author/committer:

```text
GiHoon1123 <rlaejrqo465@naver.com>
```

커밋 메시지에는 AI/Codex/Co-authored trailer를 넣지 않았다.

### 10.2 PR 본문

PR template은 없었다. `.github`에는 issue templates와 workflows만 있었다.

최종 PR 본문:

```md
Fixes #8020

`em.map()` already marks raw result data with `convertCustomTypes: true`, but `merge()` dropped that flag before checking the identity map.

As a result, a custom primary key that was already in database form could be converted as if it were a JS value.

This passes the flag through the identity map lookup and adds a regression test for the reported case.
```

의도:

- 너무 장황하게 검증 내역을 나열하지 않음
- 문제와 원인을 짧게 설명
- `Fixes #8020`로 이슈 자동 연결
- AI 보고서 같은 톤을 피함

---

## 11. 개념 복습

### 11.1 "DB 형식으로 들어왔으면 그냥 주면 되는 것 아닌가?"

맞다. 이게 이번 버그의 핵심 질문이었다.

`em.map()`으로 들어온 값은 이미 DB-form이다:

```ts
em.map(Author, { id: 'db-1' });
```

그러면 identity map에서 찾을 때도 그냥 DB-form으로 찾으면 된다.

문제는 기존 코드가 그 값을 JS-form으로 착각했다는 점이다. 대부분의 ORM API는 JS 코드에서 직접 호출되므로 기본적으로 JS-form 값을 받는 경우가 많다:

```ts
em.merge(Author, { id: 'js-1' });
```

이 경우엔 identity map hash를 만들기 위해 다음 변환이 필요할 수 있다:

```text
js-1 -> db-1
```

하지만 `em.map()`은 raw DB row를 받으므로 이미:

```text
db-1
```

이다. 여기에 다시:

```text
convertToDatabaseValue(db-1)
```

를 호출하면 안 된다.

### 11.2 택배 보관함 비유

- `js-1`: 사람이 부르는 이름
- `db-1`: 택배 보관함 번호

택배 보관함에서 물건을 찾으려면 보관함 번호가 필요하다.

#### 상황 A

사람 이름 `js-1`을 들고 왔다:

```text
js-1 -> db-1로 바꿔서 찾기
```

#### 상황 B

이미 보관함 번호 `db-1`을 들고 왔다:

```text
그냥 db-1로 찾기
```

이번 버그는 상황 B에서 이런 일이 벌어진 것이다:

```text
이미 db-1을 들고 왔는데
이걸 js-1이라고 착각하고
또 db-1로 바꾸려고 함
```

그래서 custom type이:

```text
이건 JS-form이 아닌데?
```

하고 에러를 던졌다.

### 11.3 회귀 테스트란

회귀 테스트는:

```text
한 번 고친 버그가 나중에 다시 살아나면 바로 잡아내는 테스트
```

이번 테스트는 #8020에서 신고된 상황을 그대로 코드로 박아둔다:

```ts
em.map(Author, { id: 'db-1' });
```

수정 전에는 이 코드가 터졌다.

수정 후에는:

```ts
author.id === 'js-1'
```

이어야 한다.

앞으로 누가 `merge()`나 `tryGetById()`를 다시 수정하다가 `convertCustomTypes` 전달을 깨뜨리면, 이 테스트가 다시 실패한다.

---

## 12. 결과 및 회고

### 12.1 현재 상태 (1차 제출 시점)

- PR #8021: OPEN
- 커밋 1개
- 변경 파일 3개
- 변경량: +73 / -2

**이 상태는 2026-07-27에 MERGED로 바뀐다 — 12.4절 참고.**

### 12.2 배운 점

1. **값 자체보다 값의 "형식 정보"가 중요할 때가 있다.** 이번 버그는 `db-1`이라는 값 자체가 이상한 게 아니라, 그 값이 DB-form이라는 정보가 중간 함수에서 끊겨 생겼다.

2. **ORM 내부에서는 entity를 만들기 전에 identity map을 먼저 확인한다.** 처음에는 `em.map()`이 그냥 DB row를 entity로 바꾸는 함수라고만 보면 충분해 보였지만, 실제로는 "이미 같은 PK의 entity가 있는가?"를 먼저 본다.

3. **public API 테스트 + 내부 경로 검증을 같이 잡는 게 좋다.** 테스트는 `em.map()`이라는 public API로 작성했지만, 같은 PK를 두 번 map해서 identity map 경로까지 검증했다.

4. **가장 작은 수정이 꼭 가장 얕은 수정은 아니다.** `tryGetById()`에 인자를 하나 추가하는 건 내부 메서드 시그니처를 바꾸는 일이지만, 이미 `getById()`가 같은 옵션을 받는 구조였기 때문에 오히려 가장 자연스러운 수정이었다.

5. **PR 본문은 길수록 좋은 게 아니다.** 복습 문서는 길게 써도 되지만, PR 본문은 "문제/원인/수정"만 짧게 남기는 편이 이 저장소에는 더 맞아 보였다.

### 12.3 다음에 확인할 것 (1차 제출 시점)

- [x] 메인테이너 리뷰 — 2026-07-27, `B4nan` 승인과 동시에 머지
- [x] CI 결과 — 머지됐으므로 통과로 간주

### 12.4 최종 상태 — MERGED (2026-07-27 19:27)

메인테이너 `B4nan`이 `APPROVED` 리뷰(코멘트 없이 승인만)를 남긴 직후(13초 뒤) 직접 머지했다. 별도의 수정 요청 과정은 없었다.

머지된 PR의 최종 커밋은 3개다:

```text
f310b2db  fix(core): respect custom type conversion when merging mapped data
d2177978  fix(core): respect custom type conversion when assigning to collections
93a186f6  fix(core): respect custom type conversion in identity map lookups
```

10.1절에 기록된 건 최초 제출 시점의 커밋 1개(`f310b2db`)뿐이었는데, 최종 머지본에는 관련 경로(컬렉션 대입, identity map 조회)에 대한 커밋이 2개 더 붙어 있다. 이 두 커밋을 추가한 과정은 이 세션에서 직접 다루지 않아 문서에 기록이 없다 — 크로스 세션 작업이 메모리에 안 남는다는 걸 보여주는 사례라, 사실만 기록해둔다. `git log`로 커밋 메시지/스코프만 확인 가능하고 그 이상의 맥락(왜 확장했는지, 메인테이너가 요청했는지 자체 판단인지)은 이 문서에서 재구성할 수 없다.
- [ ] 필요하면 `merge(..., { convertCustomTypes: true })` 직접 테스트 추가 여부 검토

