# [TypeScript OSS 기여] toss/es-toolkit - `merge()` 반환 타입이 deep merge 동작을 반영하지 못하던 버그 수정 (PR #1959)

> 오픈소스 기여 기록 및 학습 노트. 이 문서는 PR #1959 하나만 다룬다. **2026-08-22 머지 완료.**
>
> 이번 작업은 런타임 로직을 바꾸는 수정이 아니라, 이미 deep merge로 동작하던 `merge()`의 TypeScript 반환 타입을 실제 동작에 맞게 바꾼 작업이다. 겉으로는 타입 한 줄을 바꾸는 일처럼 보이지만, 핵심은 TypeScript의 intersection type(`T & S`)이 런타임의 "재귀 병합"을 표현하지 못한다는 점을 정확히 짚는 것이다.

---

## 1. 개요

### 1.1 프로젝트 소개

- **저장소**: https://github.com/toss/es-toolkit
- **npm 패키지**: `es-toolkit`
- **성격**: JavaScript/TypeScript utility library. Lodash 계열 유틸 함수를 더 작은 번들, tree-shaking 친화적인 구조, TypeScript 타입 지원 중심으로 제공한다.
- **작업 위치**: `/Users/dustin/Desktop/common/open-source/es-toolkit`
- **작업 브랜치**: `fix-1554-merge-deep-type`
- **이슈**: https://github.com/toss/es-toolkit/issues/1554
- **PR**: https://github.com/toss/es-toolkit/pull/1959
- **커밋**: `e1e6eb79d0440d5fc2713f6b4029ecc878059489`
- **상태**: MERGED (2026-08-22, maintainer `raon0211`)

### 1.2 이슈 (#1554)

**제목**: `merge typing is wrong`

이슈의 핵심은 `merge()` 함수의 반환 타입이 실제 동작보다 얕게 표현되어 있다는 것이다.

기존 타입은 다음 형태였다.

```ts
export function merge<T extends Record<PropertyKey, any>, S extends Record<PropertyKey, any>>(
  target: T,
  source: S
): T & S;
```

`T & S`는 "T이면서 S인 타입"이라는 뜻이다. top-level key만 보면 얼핏 괜찮아 보인다. 예를 들어:

```ts
const result = merge({ a: 1 }, { b: "x" });
```

이 경우 `T & S`는 대략 이런 타입처럼 동작한다.

```ts
{
  a: number;
  b: string;
}
```

하지만 nested object가 겹치면 문제가 생긴다.

```ts
const result = merge(
  { a: { x: 1, y: "target" } },
  { a: { y: 2, z: true } }
);
```

런타임 결과는 이렇게 된다.

```ts
{
  a: {
    x: 1,
    y: 2,
    z: true,
  },
}
```

즉 `a.y`는 source의 값 `2`로 덮어써진다.

하지만 기존 `T & S` 타입은 이걸 deep merge로 이해하지 않는다. `a` 자체를 이렇게 교차시킨다.

```ts
{
  x: number;
  y: string;
} & {
  y: number;
  z: boolean;
}
```

그러면 `y`는 동시에 `string`이면서 `number`여야 한다.

```ts
string & number
```

이건 불가능한 타입이므로 `never`가 된다.

결국 런타임에서는 정상 값이 있는데 TypeScript에서는 해당 값이 `never`로 무너질 수 있다. 이게 #1554의 핵심 버그다.

### 1.3 한 줄 요약

> `merge()`는 런타임에서 nested object를 재귀적으로 병합하지만 반환 타입은 얕은 `T & S`라서, 겹치는 nested property가 `never`로 무너질 수 있었다. 반환 타입을 `MergeDeep<T, S>`로 바꿔 런타임의 deep merge 동작을 타입에 반영했다.

---

## 2. 배경 지식

### 2.1 `merge()`는 무엇을 하는 함수인가

`merge()`는 source object의 값을 target object에 병합한다.

```ts
const target = { a: 1, b: { x: 1, y: 2 } };
const source = { b: { y: 3, z: 4 }, c: 5 };

const result = merge(target, source);
```

결과:

```ts
{
  a: 1,
  b: {
    x: 1,
    y: 3,
    z: 4,
  },
  c: 5,
}
```

여기서 중요한 점은 `b`를 통째로 바꾸지 않는다는 것이다.

```ts
target.b = source.b;
```

처럼 얕게 덮어쓰는 게 아니라, `b` 내부로 들어가서 key 단위로 병합한다.

```text
target.b.x 유지
target.b.y는 source.b.y로 덮어씀
source.b.z 추가
```

이런 동작을 deep merge라고 부른다.

### 2.2 shallow merge와 deep merge

shallow merge는 한 단계만 본다.

```ts
const target = { a: { x: 1, y: 2 } };
const source = { a: { y: 3, z: 4 } };
```

shallow merge라면 `a` 자체를 source의 `a`로 바꾼다.

```ts
{
  a: {
    y: 3,
    z: 4,
  },
}
```

`x`는 사라진다.

deep merge는 nested object 안으로 들어간다.

```ts
{
  a: {
    x: 1,
    y: 3,
    z: 4,
  },
}
```

`merge()`의 런타임은 deep merge 쪽이다.

### 2.3 TypeScript의 intersection type

TypeScript에서 `A & B`는 "A와 B 조건을 모두 만족하는 타입"이다.

예:

```ts
type A = { a: number };
type B = { b: string };

type C = A & B;
```

`C`는 이렇게 생각할 수 있다.

```ts
{
  a: number;
  b: string;
}
```

서로 다른 key만 있으면 괜찮다.

문제는 같은 key가 서로 다른 타입을 가지는 경우다.

```ts
type A = { value: string };
type B = { value: number };

type C = A & B;
```

`C["value"]`는 다음과 비슷하다.

```ts
string & number
```

문자열이면서 동시에 숫자인 값은 없다. 그래서 `never`가 된다.

이게 이번 이슈에서 터진 지점이다. `merge()`는 source가 target을 덮어쓰는데, `T & S`는 "둘 다 동시에 만족해야 한다"고 해석한다. 즉 런타임의 overwrite semantics를 표현하지 못한다.

### 2.4 왜 top-level에서는 잘 되는 것처럼 보였나

기존 타입이 항상 틀려 보이는 건 아니다.

```ts
const result = merge({ a: 1 }, { b: "x" });
```

이 경우에는 key가 겹치지 않는다.

```text
target keys: a
source keys: b
```

그래서 `T & S`도 결과를 그럭저럭 잘 표현한다.

```ts
{
  a: number;
  b: string;
}
```

하지만 key가 겹치기 시작하면 문제가 보인다.

```ts
const result = merge({ a: "target" }, { a: 1 });
```

런타임 결과는 `a: 1`이다. source가 덮어쓴다.

그런데 기존 타입은:

```ts
{ a: string } & { a: number }
```

이므로 `a`가 `never`가 된다.

nested object에서도 같은 문제가 더 눈에 안 띄게 발생한다.

---

## 3. 기존 코드 흐름

### 3.1 기존 `merge()` 타입

기존 코드는 구현과 타입이 한 함수 선언에 같이 있었다.

```ts
export function merge<T extends Record<PropertyKey, any>, S extends Record<PropertyKey, any>>(
  target: T,
  source: S
): T & S {
  const sourceKeys = Object.keys(source) as Array<keyof S>;

  // ...

  return target;
}
```

반환 타입은 `T & S`였다.

이 타입은 "target과 source의 속성을 모두 가진다"는 의미를 표현하기에는 간단하지만, 다음을 표현하지 못한다.

```text
같은 key가 있으면 source가 덮어쓴다.
양쪽 값이 merge 가능한 object/array면 안으로 들어가서 재귀 병합한다.
source 값이 undefined면 target 값을 덮어쓰지 않는다.
```

### 3.2 기존 런타임 병합 로직

실제 병합 로직은 대략 이렇게 동작한다.

```ts
const sourceValue = source[key];
const targetValue = target[key];

if (isMergeableValue(sourceValue) && isMergeableValue(targetValue)) {
  target[key] = merge(targetValue, sourceValue);
} else if (Array.isArray(sourceValue)) {
  target[key] = merge([], sourceValue);
} else if (isPlainObject(sourceValue)) {
  target[key] = merge({}, sourceValue);
} else if (targetValue === undefined || sourceValue !== undefined) {
  target[key] = sourceValue;
}
```

각 분기를 풀어보면 다음과 같다.

### 3.3 양쪽이 merge 가능한 값이면 재귀 병합

```ts
if (isMergeableValue(sourceValue) && isMergeableValue(targetValue)) {
  target[key] = merge(targetValue, sourceValue);
}
```

`isMergeableValue()`는 plain object 또는 array인지 확인한다.

```ts
function isMergeableValue(value: unknown) {
  return isPlainObject(value) || Array.isArray(value);
}
```

즉 target과 source 양쪽 값이 모두 object/array라면 그 안으로 들어가서 다시 `merge()`한다.

예:

```ts
merge(
  { b: { x: 1, y: 2 } },
  { b: { y: 3, z: 4 } }
);
```

`b`에서 양쪽 값이 모두 plain object이므로:

```ts
merge({ x: 1, y: 2 }, { y: 3, z: 4 });
```

를 다시 실행한다.

### 3.4 source가 array면 새 array에 복사 병합

```ts
else if (Array.isArray(sourceValue)) {
  target[key] = merge([], sourceValue);
}
```

source 값이 array인데 target 값은 merge 가능한 값이 아니면, source array를 그대로 참조하지 않고 새 array로 병합한다.

### 3.5 source가 plain object면 새 object에 복사 병합

```ts
else if (isPlainObject(sourceValue)) {
  target[key] = merge({}, sourceValue);
}
```

source 값이 plain object인데 target 값은 merge 가능한 값이 아니면, source object를 그대로 참조하지 않고 새 object로 병합한다.

### 3.6 source가 `undefined`면 기존 값을 유지

```ts
else if (targetValue === undefined || sourceValue !== undefined) {
  target[key] = sourceValue;
}
```

이 조건은 처음 보면 조금 헷갈린다.

핵심은 이거다.

```text
sourceValue가 undefined이고 targetValue가 이미 있으면 덮어쓰지 않는다.
```

예:

```ts
merge({ a: 1 }, { a: undefined });
```

결과는:

```ts
{ a: 1 }
```

이다.

반대로 target에도 값이 없다면 source의 `undefined`를 넣어도 결과적으로 값은 `undefined`다.

```ts
merge({}, { a: undefined });
```

이 경우 `a`는 `undefined`가 된다.

이번 타입 수정에서 `undefined`를 별도로 처리한 이유가 바로 이 분기 때문이다.

---

## 4. 문제가 된 타입 예시

### 4.1 nested object overlap

테스트로 추가한 핵심 케이스는 다음과 같다.

```ts
const target = { a: 1, b: { x: 1, y: 2 } };
const source = { b: { y: 3, z: 4 }, c: 5 };
const result = merge(target, source);
```

런타임 결과는:

```ts
{
  a: 1,
  b: {
    x: 1,
    y: 3,
    z: 4,
  },
  c: 5,
}
```

따라서 타입도 대략 이렇게 나와야 한다.

```ts
{
  a: number;
  b: {
    x: number;
    y: number;
    z: number;
  };
  c: number;
}
```

하지만 기존 `T & S`는 `b` 내부를 merge하지 않고, `target.b`와 `source.b`를 intersection으로 붙인다.

```ts
{
  x: number;
  y: number;
} & {
  y: number;
  z: number;
}
```

여기서는 `y`가 둘 다 `number`라 문제가 덜 드러나지만, 서로 다른 타입이면 바로 무너진다.

```ts
merge(
  { b: { y: "target" } },
  { b: { y: 3 } }
);
```

기존 타입에서는:

```ts
string & number
```

가 되어 `never`가 된다.

### 4.2 defined source overwrite

다음 케이스도 중요하다.

```ts
const target = { a: 1 as const };
const source = { a: 2 as const };
const result = merge(target, source);
```

런타임 결과는 무조건:

```ts
{ a: 2 }
```

이다.

따라서 타입도:

```ts
2
```

가 맞다.

처음 작성한 타입에서는 이 부분이 살짝 넓었다. source가 확실히 `2`인데도 결과 타입을 `1 | 2`처럼 만들 수 있었다. 자체 검토 중 이 부분을 고쳤다.

### 4.3 optional source property

반대로 source 값이 없을 수도 있으면 target 값이 살아남을 수 있다.

```ts
const target: { b: string | undefined } = { b: undefined };
const source: { b?: string } = {};
const result = merge(target, source);
```

`source.b`는 타입상 `string | undefined`다.

런타임에서는 두 경우가 가능하다.

```text
source.b가 string이면 target.b를 덮어쓴다.
source.b가 undefined면 target.b를 덮어쓰지 않는다.
```

그래서 결과 타입은:

```ts
string | undefined
```

가 된다.

이 케이스는 단순히 `never`가 안 되는지만 보는 게 아니라, `undefined` source를 skip하는 기존 런타임 동작을 타입에서도 보존하는지 확인한다.

---

## 5. 수정 내용

### 5.1 `MergeDeep<T, S>` 추가

새로 추가한 타입은 다음과 같다.

```ts
export type MergeDeep<T extends Record<PropertyKey, any>, S extends Record<PropertyKey, any>> = {
  [K in keyof T | keyof S]: K extends keyof S
    ? K extends keyof T
      ? MergeDeepValue<T[K], S[K]>
      : S[K]
    : K extends keyof T
      ? T[K]
      : never;
};
```

하나씩 풀어보면:

```ts
[K in keyof T | keyof S]
```

target과 source의 key를 모두 순회한다.

예:

```ts
type T = { a: number; b: { x: number } };
type S = { b: { y: number }; c: string };
```

그러면 key union은:

```ts
"a" | "b" | "c"
```

가 된다.

각 key에 대해 다음을 판단한다.

```text
source에도 있고 target에도 있나?
  -> 두 값을 MergeDeepValue로 합친다.

source에만 있나?
  -> source 타입을 쓴다.

target에만 있나?
  -> target 타입을 쓴다.
```

코드로 보면:

```ts
K extends keyof S
  ? K extends keyof T
    ? MergeDeepValue<T[K], S[K]>
    : S[K]
  : K extends keyof T
    ? T[K]
    : never;
```

### 5.2 `MergeDeepValue<TargetValue, SourceValue>`

겹치는 key의 value를 어떻게 합칠지 결정하는 타입이다.

```ts
type MergeDeepValue<TargetValue, SourceValue> =
  SourceValue extends Record<PropertyKey, any>
    ? TargetValue extends Record<PropertyKey, any>
      ? MergeDeep<TargetValue, SourceValue>
      : SourceValue
    : undefined extends SourceValue
      ? TargetValue | Exclude<SourceValue, undefined>
      : SourceValue;
```

분기별로 보면 다음과 같다.

### 5.3 양쪽 값이 object면 재귀 병합

```ts
SourceValue extends Record<PropertyKey, any>
  ? TargetValue extends Record<PropertyKey, any>
    ? MergeDeep<TargetValue, SourceValue>
    : SourceValue
```

source와 target이 둘 다 object 계열이면 다시 `MergeDeep`으로 들어간다.

예:

```ts
type TargetValue = { x: number; y: string };
type SourceValue = { y: number; z: boolean };
```

결과는:

```ts
{
  x: number;
  y: number;
  z: boolean;
}
```

`y`는 intersection이 아니라 source 쪽 타입인 `number`가 된다.

### 5.4 source가 object인데 target이 object가 아니면 source

```ts
TargetValue extends Record<PropertyKey, any>
  ? MergeDeep<TargetValue, SourceValue>
  : SourceValue
```

source는 object인데 target은 primitive라면 source가 덮어쓴다.

예:

```ts
merge({ a: 1 }, { a: { b: "x" } });
```

런타임 결과는:

```ts
{ a: { b: "x" } }
```

이므로 타입도 source 쪽 object가 맞다.

### 5.5 source가 `undefined`일 수 있으면 target도 유지

```ts
undefined extends SourceValue
  ? TargetValue | Exclude<SourceValue, undefined>
  : SourceValue;
```

이 부분은 이번 수정에서 가장 조심한 부분이다.

source가 확실히 값이면 source가 덮어쓴다.

```ts
merge({ a: 1 as const }, { a: 2 as const });
```

결과 타입:

```ts
2
```

하지만 source가 `undefined`일 수도 있으면 target이 남을 수 있다.

```ts
const source: { a?: 2 } = {};
merge({ a: 1 as const }, source);
```

결과 타입:

```ts
1 | 2
```

여기서 `Exclude<SourceValue, undefined>`를 쓰는 이유는, 실제로 source 값이 `undefined`인 경우에는 덮어쓰지 않기 때문이다. 그러면 source 쪽에서 의미 있게 남는 값은 `undefined`를 제외한 값이다.

### 5.6 `merge()` 반환 타입 변경

기존:

```ts
export function merge<T extends Record<PropertyKey, any>, S extends Record<PropertyKey, any>>(
  target: T,
  source: S
): T & S {
  // implementation
}
```

변경 후:

```ts
export function merge<T extends Record<PropertyKey, any>, S extends Record<PropertyKey, any>>(
  target: T,
  source: S
): MergeDeep<T, S>;

export function merge(target: Record<PropertyKey, any>, source: Record<PropertyKey, any>): Record<PropertyKey, any> {
  // implementation
}
```

여기서 overload를 쓴 이유는 외부에 보이는 타입과 내부 구현 타입을 분리하기 위해서다.

외부 사용자가 보는 타입:

```ts
merge<T, S>(target: T, source: S): MergeDeep<T, S>
```

내부 구현에서 쓰는 타입:

```ts
merge(target: Record<PropertyKey, any>, source: Record<PropertyKey, any>): Record<PropertyKey, any>
```

런타임 구현은 그대로다. 바뀐 건 TypeScript가 사용자에게 보여주는 반환 타입이다.

overload를 쓰면 내부 구현의 타입 안전성은 느슨해진다. 구현부 시그니처(`Record<PropertyKey, any>`)의 재귀 호출 결과가 공개 시그니처인 `MergeDeep<T, S>`의 약속을 실제로 지키는지, TypeScript는 자동으로 검증해주지 않는다. 이 부분은 기존 런타임 테스트 스위트가 대신 보증한다 — 타입은 느슨해졌지만, 실제 실행 결과는 테스트로 이미 커버되어 있다.

---

## 6. 테스트

### 6.1 nested object deep merge 타입 테스트

```ts
it('should type nested objects as a deep merge, not a shallow intersection', () => {
  const target = { a: 1, b: { x: 1, y: 2 } };
  const source = { b: { y: 3, z: 4 }, c: 5 };
  const result = merge(target, source);

  expectTypeOf(result).toEqualTypeOf<{
    a: number;
    b: { x: number; y: number; z: number };
    c: number;
  }>();
});
```

이 테스트는 `merge()` 반환 타입이 nested object를 실제 deep merge 결과처럼 추론하는지 확인한다.

기존 `T & S`였다면 nested object를 재귀적으로 합치는 타입이 아니므로 이 케이스를 정확히 표현하지 못한다.

### 6.2 source 값이 확실하면 source 타입을 사용

```ts
it('should use the source type when a defined source property overwrites the target', () => {
  const target = { a: 1 as const };
  const source = { a: 2 as const };
  const result = merge(target, source);

  expectTypeOf(result.a).toEqualTypeOf<2>();
});
```

이 테스트는 자체 검토 중 발견한 타입 과대 추론을 막는다.

런타임에서 source 값 `2`가 확실히 들어오면 결과는 `1 | 2`가 아니라 `2`다.

### 6.3 optional source property는 target을 유지할 수 있음

```ts
it('does not collapse to never when an optional source property may be skipped', () => {
  const target: { b: string | undefined } = { b: undefined };
  const source: { b?: string } = {};
  const result = merge(target, source);

  expectTypeOf(result.b).not.toBeNever();
  expectTypeOf(result.b).toEqualTypeOf<string | undefined>();
});
```

이 테스트는 source property가 optional일 때 `undefined` skip 동작이 타입에도 반영되는지 확인한다.

### 6.4 한쪽에만 있는 key는 보존

```ts
it('keeps a plain intersection for keys present on only one side', () => {
  const target = { a: 1 };
  const source = { b: 'x' };
  const result = merge(target, source);

  expectTypeOf(result).toEqualTypeOf<{ a: number; b: string }>();
});
```

이 케이스는 기존 `T & S`가 잘 표현하던 단순 병합도 계속 유지되는지 확인한다.

---

## 7. 검증 결과

PR 본문에는 이 변경과 직접 연결된 결과만 넣었다.

```text
yarn vitest run src/object/merge

Test Files  2 passed (2)
Tests       23 passed (23)
```

```text
yarn tsc --noEmit

0 errors
```

로컬에서 추가로 확인한 것:

```text
yarn test

Test Files  647 passed (647)
Tests       4462 passed | 2 skipped (4464)
```

```text
yarn typecheck

passed
```

```text
yarn build

passed
```

```text
yarn workspace type-tests test

Test Files  1 passed (1)
Tests       1 passed (1)
Type Errors no errors
```

```text
yarn prettier --check src/object/merge.ts src/object/merge.spec.ts

All matched files use Prettier code style
```

```text
git diff --check

passed
```

`yarn lint` 전체 실행은 실패했다. 원인은 이번 변경 파일이 아니라 기존 docs Vue 파일의 curly rule 에러였다.

```text
docs/.vitepress/components/FpLazySimulation.vue
  8 curly errors
```

변경 파일만 대상으로 한 lint는 에러 없이 warning만 있었다.

```text
yarn lint src/object/merge.ts src/object/merge.spec.ts

0 errors
11 warnings
```

warning은 `Record<PropertyKey, any>`와 기존 테스트의 `(nestedArray.x as any).a` 같은 `any` 관련 경고다. 기존 코드 스타일과 이어지는 부분이라 이번 PR에서는 별도로 건드리지 않았다.

### 7.1 `yarn vitest run`과 `yarn tsc --noEmit`을 둘 다 확인한 이유

`expectTypeOf(...).toEqualTypeOf<...>()` 같은 타입 테스트는 `yarn vitest run`만으로는 검증되지 않는다.

`vitest run`은 esbuild로 TypeScript 타입 표기를 지운 뒤 남은 JavaScript만 실행한다. 타입이 맞는지 보는 게 아니라, 타입 라벨을 지운 채로 실제 값만 비교한다.

실제로 disable-and-reconfirm 중 `MergeDeep`을 원래 `T & S`로 되돌린 상태에서 `yarn vitest run`을 돌렸는데, 타입 테스트 4개를 포함해 전부 통과했다. 타입이 깨진 상태에서도 테스트가 통과한 것이다.

`tsc --noEmit`은 반대로 코드를 실행하지 않고 타입만 검사한다. 같은 되돌린 상태에서 돌리자 `Expected: string, Actual: never` 실패가 정확히 나왔다.

그래서 타입만 바뀌는 PR은 `vitest run`과 `tsc --noEmit`을 각각 확인해야 한다. 하나는 런타임 동작을, 하나는 타입 정합성을 검사하기 때문에 하나만 보면 놓치는 부분이 생긴다.

---

## 8. 자체 검토 메모

### 8.1 처음 구현에서 고친 부분

처음 타입은 primitive overwrite 경로를 이렇게 잡고 있었다.

```ts
TargetValue | Exclude<SourceValue, undefined>
```

이 방식은 source가 확실히 값이 있는 경우에도 target 타입을 남긴다.

예:

```ts
merge({ a: 1 as const }, { a: 2 as const });
```

런타임 결과는 `2`인데 타입은 `1 | 2`가 될 수 있다.

그래서 다음처럼 source type에 `undefined`가 들어올 수 있는 경우에만 target을 살리도록 고쳤다.

```ts
undefined extends SourceValue
  ? TargetValue | Exclude<SourceValue, undefined>
  : SourceValue
```

이게 런타임의 `sourceValue !== undefined` 조건과 더 잘 맞는다.

### 8.2 `compat`을 건드리지 않은 이유

es-toolkit에는 일반 API와 lodash 호환 API가 따로 있다.

이번 이슈는 main package의 `merge()` 타입 문제다.

`compat` 쪽은 lodash compatibility가 중요한 영역이라, 타입을 더 정확하게 바꾸는 것보다 lodash와 맞추는 쪽이 우선일 수 있다. 이슈 댓글에서도 "main package only, compat remains `T & S`" 방향이 언급되어 있었다.

그래서 이번 PR은 `src/object/merge.ts`만 바꾸고 `src/compat/object/merge.ts`는 건드리지 않았다.

### 8.3 열린 PR #1692와의 관계

이미 열린 PR #1692가 있었다.

```text
feat: add MergeDeep type and merge.deep/mergeWith.deep/toMerged.deep APIs
```

그 PR은 기존 `merge()` 반환 타입을 바꾸기보다 `merge.deep()` 같은 새 API를 추가하는 방향이다.

이번 PR은 #1554의 "merge typing is wrong" 문제에 직접 맞춰 기존 `merge()`의 반환 타입을 실제 런타임 동작에 맞추는 방향이다.

즉 겹치는 주제는 있지만 접근 방식이 다르다.

리뷰에서 maintainer가 "기존 `merge()` 타입은 유지하고 별도 deep API로 가자"고 판단할 가능성은 있다. 하지만 이슈 자체가 기존 `merge()` typing을 문제 삼고 있으므로, 이번 PR 방향도 충분히 말이 된다.

### 8.4 타입의 현실적인 한계

이번 `MergeDeep`은 런타임의 핵심 동작을 반영하기 위한 타입이다. 완벽한 runtime simulator가 아니다.

특히 deep merge 타입은 너무 정교하게 만들면 다음 문제가 생긴다.

```text
타입 코드가 지나치게 커짐
컴파일 성능이 나빠질 수 있음
리뷰어가 의도를 이해하기 어려움
작은 bugfix PR의 범위를 넘어섬
```

그래서 이번 PR은 #1554에서 제기된 핵심 문제, 즉 nested object merge와 overwrite/undefined behavior를 좁게 고정하는 쪽으로 잡았다.

---

## 9. PR 본문

실제 PR 본문은 저장소의 `AGENTS.md` 템플릿에 맞춰 짧게 작성했다.

```md
## Summary

Fixes #1554

`merge()` already does a recursive merge at runtime, but its return type was `T & S`. That can make overlapping nested properties collapse to `never`, even though the runtime result is valid.

## Changes

- Add recursive `MergeDeep<T, S>` / `MergeDeepValue<TargetValue, SourceValue>` types and use them as `merge()`'s return type
- Keep the existing `undefined` source behavior: an `undefined` source value doesn't overwrite a defined target value
- Leave `compat`'s `merge` unchanged

## Test results

- `yarn vitest run src/object/merge`: 2 files, 23 tests passed
- `yarn tsc --noEmit`: 0 errors
```

---

## 10. 결과

**최종 결과 (2026-08-22)**: 메인테이너 `raon0211`이 PR #1959를 머지했다. `merge()`의 반환 타입이 단순한 `T & S` 대신 실제 재귀 병합 동작을 반영하도록 수정되어 이슈 #1554가 해결됐다.

---

## 11. 배운 점

### 11.1 타입 버그도 런타임 동작을 기준으로 봐야 한다

이번 작업은 런타임 코드를 거의 바꾸지 않았다.

하지만 타입 수정도 결국 런타임 동작을 설명하는 일이다.

```text
런타임은 source가 덮어쓴다.
런타임은 object 안으로 재귀적으로 들어간다.
런타임은 undefined source를 건너뛴다.
```

이 세 가지를 타입에 반영해야 한다.

단순히 "deep merge니까 recursive type"이라고 생각하면 부족하다. `undefined`처럼 작은 런타임 조건까지 타입에 들어가야 실제로 쓰는 사람이 기대하는 동작과 맞아떨어진다.

### 11.2 `T & S`는 merge 결과 타입이 아니다

`T & S`는 객체 합치기처럼 보이지만, 정확히는 "두 타입을 동시에 만족"이다.

그래서 overwrite가 있는 API에는 잘 맞지 않는다.

```ts
{ a: string } & { a: number }
```

은:

```ts
{ a: never }
```

에 가까워진다.

런타임 merge는 "둘 다 만족"이 아니라 "source가 target을 덮어씀"이다. 이 차이를 이해하는 게 이번 이슈의 핵심이다.

### 11.3 타입 PR은 scope 조절이 중요하다

deep merge type을 제대로 만들려고 하면 built-in object, tuple, array, optional key, index signature까지 끝없이 들어갈 수 있다.

이번 PR에서는 이슈의 핵심 범위를 벗어나지 않도록 다음 원칙을 잡았다.

```text
런타임 구현은 건드리지 않는다.
main merge() 반환 타입만 바꾼다.
compat merge는 그대로 둔다.
테스트는 nested object, overwrite, undefined skip에 집중한다.
```

작은 PR일수록 리뷰어가 판단하기 쉽다.
