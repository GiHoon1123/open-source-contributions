# [OSS 기여] toss/es-git - `skipLibCheck: false`에서 실패하는 생성 타입 선언 수정 (PR #227)

- 상태: 🟢 MERGED — 2026-10-11 15:53:29 KST, marshallku
- 저장소: https://github.com/toss/es-git
- 이슈: https://github.com/toss/es-git/issues/215
- PR: https://github.com/toss/es-git/pull/227
- 브랜치: `fix/skiplibcheck-declaration-errors`
최종 PR 커밋: `cdd7a99bef88f031d35aa206cd344db6890d0543`
머지 커밋: [`83524a91`](https://github.com/toss/es-git/commit/83524a91a3d6d9a17ddfc460b961b7763c98899a)

1~29절은 작업 당시의 설명과 검증 기록이다. 최초 `IterableIterator` 방식은 22절 이후 리뷰를 거쳐
클래스와 `IteratorObject` 인터페이스를 병합하는 방식으로 수정됐다. 최종 머지와 CI 결과는 30절에 추가했다.

이 문서는 변경된 파일만 나열하는 기록이 아니다. Rust로 작성한 기능이 napi-rs를 거쳐 Node.js와
TypeScript 사용자에게 전달되는 과정, `.d.ts`가 맡는 역할, Rust의 타입 별칭이 생성 코드에 새어 나온
이유, JavaScript iterator와 TypeScript 제네릭의 의미를 나중에 처음부터 다시 공부할 수 있도록 정리한
복습 노트다.

## 1. 이슈를 한 문장으로 정리하면

`es-git`의 실제 Git 기능은 정상적으로 실행되지만, npm 패키지에 함께 배포되는 `index.d.ts`가 잘못
생성되어 `skipLibCheck: false`를 사용하는 TypeScript 프로젝트에서 타입 검사가 실패했다.

오류는 서로 독립적인 두 가지였다.

```text
Cannot find name 'GitReference'. Did you mean 'Reference'?
Cannot extend an interface 'Iterator'. Did you mean 'implements'?
```

첫 번째는 Rust 파일 안에서만 쓰던 `GitReference`라는 별칭이 TypeScript 공개 선언에 노출된 문제다.
두 번째는 생성기가 `Iterator`를 항상 상속 가능한 클래스라고 가정했지만, ES2022 타입 환경에서는
`Iterator`가 인터페이스이기 때문에 생긴 문제다.

이것은 Git 저장소를 열거나 브랜치를 읽는 런타임 로직의 버그가 아니다. 실제 코드의 사용 방법을
TypeScript에 알려주는 타입 선언이 잘못된 배포 타입 버그다.

## 2. es-git은 어떤 라이브러리인가

`es-git`은 Git 기능을 Rust로 구현하고 Node.js에서 호출할 수 있게 제공한다. JavaScript만으로 Git을
다시 구현하는 대신 Rust의 `git2` 계열 기능을 네이티브 모듈로 연결한다.

전체 구조는 다음과 같이 볼 수 있다.

```text
Rust 소스 (`src/*.rs`)
        │
        │ napi-rs 빌드와 코드 생성
        ▼
Node.js에서 불러오는 네이티브 모듈과 JavaScript 진입점
        │
        ├── `index.js`: 실제로 실행되는 API
        └── `index.d.ts`: TypeScript용 타입 설명
```

사용자는 JavaScript 또는 TypeScript에서 다음처럼 쓴다.

```ts
import { openRepository } from 'es-git';

const repository = await openRepository('.');
const head = repository.head();
```

`openRepository()`가 실제로 저장소를 여는 일은 Rust와 네이티브 모듈이 담당한다. TypeScript는 Rust
코드를 직접 읽지 않고 `index.d.ts`를 읽어 함수의 매개변수와 반환 타입을 판단한다.

## 3. `.d.ts`는 무엇인가

`.d.ts`는 TypeScript declaration file, 즉 타입 선언 파일이다. 구현을 실행하는 파일이 아니라 다른
코드가 어떤 모양인지 TypeScript에 설명하는 파일이다.

예를 들어 실제 JavaScript 구현이 다음과 같다고 하자.

```js
export function add(left, right) {
  return left + right;
}
```

TypeScript 사용자를 위한 선언은 다음처럼 작성할 수 있다.

```ts
export declare function add(left: number, right: number): number;
```

두 파일의 역할은 다르다.

```text
JavaScript 또는 네이티브 모듈
실제 계산과 I/O를 수행한다.

.d.ts
함수에 어떤 타입을 넣고 어떤 타입을 받는지 설명한다.
```

`declare`는 “이 함수의 실제 구현은 다른 곳에 있으니 여기서는 타입만 설명한다”는 뜻이다. 따라서
`.d.ts`에 적힌 함수 본문은 없고, 이 파일만 실행해서 Git 작업을 수행할 수도 없다.

`es-git/package.json`에는 실제 진입점과 타입 진입점이 따로 지정되어 있다.

```json
{
  "main": "index.js",
  "types": "index.d.ts"
}
```

Node.js는 `main`을 사용해 코드를 실행하고, TypeScript는 `types`를 사용해 사용자 코드를 검사한다.
실제 제품이 정상이어도 설명서가 존재하지 않는 타입을 요구하면 TypeScript 사용자는 빌드할 수 없다.

## 4. `skipLibCheck`가 무엇인가

TypeScript는 애플리케이션 코드뿐 아니라 의존성이 제공하는 `.d.ts`도 검사할 수 있다.

```json
{
  "compilerOptions": {
    "skipLibCheck": true
  }
}
```

`skipLibCheck: true`는 의존성 선언 파일 내부의 검사를 상당 부분 건너뛴다. 라이브러리 타입을 전혀
사용하지 않는다는 뜻은 아니다. 사용자 코드에서 API를 잘못 호출했는지는 계속 검사하지만, 라이브러리
선언 내부의 모순 때문에 전체 빌드가 막히는 일을 줄여준다.

반대로 다음 설정은 의존성 선언 파일 자체도 검사한다.

```json
{
  "compilerOptions": {
    "skipLibCheck": false
  }
}
```

이슈 #215의 재현 설정은 다음과 같다.

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "skipLibCheck": false,
    "types": ["node"],
    "noEmit": true
  }
}
```

사용자 코드는 평범하다.

```ts
import { openRepository } from 'es-git';

async function main() {
  const repo = await openRepository('.');
  repo.statuses();
  repo.getAnnotatedCommitFromReference(repo.head());
}

void main();
```

이 코드를 `tsc`로 검사하면 사용자 코드가 아니라 `es-git/index.d.ts`에서 두 오류가 발생했다.

## 5. napi-rs는 무엇인가

Node.js는 JavaScript를 실행하고, es-git의 핵심 구현은 Rust로 작성되어 있다. 두 언어의 함수와 값을
서로 전달하려면 연결 계층이 필요하다.

Node-API는 Node.js와 C, C++, Rust 같은 네이티브 코드가 통신하기 위한 안정된 API 규격이다. napi-rs는
Rust 개발자가 Node-API를 직접 다루는 부담을 줄여주는 Rust 라이브러리와 코드 생성 도구다.

```text
JavaScript 값              Rust 값
------------------------------------------------
number                <->  i32, u32, f64 등
string                <->  String
JavaScript object     <->  Rust 구조체
throw                  <->  Result의 오류
JavaScript function   <->  `#[napi]` Rust 함수
```

Rust 코드에 다음과 같이 표시하면 napi-rs가 Node.js에 공개할 연결 코드를 만든다.

```rust
#[napi]
pub fn add(left: i32, right: i32) -> i32 {
  left + right
}
```

JavaScript에서는 일반 함수처럼 사용할 수 있다.

```js
import { add } from 'native-package';

add(1, 2);
```

napi-rs는 TypeScript 사용자를 위한 선언도 생성한다.

```ts
export declare function add(left: number, right: number): number;
```

이번 버그는 Rust 구현에서 JavaScript API를 만드는 과정 자체보다, Rust 타입 이름과 iterator 정보를
TypeScript 선언으로 옮기는 코드 생성 경계에서 발생했다.

## 6. 첫 번째 버그: Rust의 지역 별칭이 공개 타입 이름으로 노출됨

### 6.1 같은 이름을 가진 두 타입

문제가 있던 `src/annotated_commit.rs`는 이름이 같은 두 `Reference` 타입을 함께 사용한다.

```text
`crate::reference::Reference`
es-git이 제공하는 Git reference 타입

`napi::bindgen_prelude::Reference<T>`
JavaScript 객체의 참조를 보관하기 위한 napi-rs 타입
```

둘 다 짧은 이름이 `Reference`라서 한 파일에서 그대로 가져오면 어떤 타입을 뜻하는지 구분하기 어렵다.
기존 코드는 es-git의 타입에 지역 별칭을 붙였다.

```rust
use crate::reference::Reference as GitReference;
```

이 문장은 `crate::reference::Reference`의 이름을 프로젝트 전체에서 바꾸는 것이 아니다. 현재 Rust
파일 안에서만 `GitReference`라고 부르겠다는 뜻이다.

함수는 두 Reference를 다음처럼 구분했다.

```rust
pub fn get_annotated_commit_from_reference(
  &self,
  reference: &GitReference,
  this: Reference<Repository>,
  env: Env,
) -> crate::Result<AnnotatedCommit> {
  // ...
}
```

```text
`GitReference`
현재 파일에서 별칭을 붙인 es-git 타입

`Reference<Repository>`
napi-rs의 제네릭 reference 타입
```

Rust 컴파일러에는 문제가 없는 구분이다.

### 6.2 코드 생성기에서 생긴 문제

napi-rs의 타입 선언 생성기는 함수 시그니처에서 보이는 지역 식별자 `GitReference`를 사용해 다음 선언을
만들었다.

```ts
getAnnotatedCommitFromReference(reference: GitReference): AnnotatedCommit;
```

하지만 npm 패키지가 공개하는 TypeScript 타입에는 `GitReference`라는 이름이 없다. 실제 공개 클래스는
다음 이름이다.

```ts
export declare class Reference {
  // ...
}
```

따라서 TypeScript는 `GitReference`를 찾을 수 없었다.

```text
Rust 파일 내부 이름: GitReference
              │
              │ 코드 생성
              ▼
TypeScript 공개 선언: GitReference
              │
              ▼
공개된 타입 목록에 없음 -> 컴파일 오류
```

### 6.3 전체 경로를 사용한 이유

수정은 지역 별칭을 제거하고 필요한 위치에서 타입의 전체 경로를 적는 것이다.

```rust
pub fn get_annotated_commit_from_reference(
  &self,
  reference: &crate::reference::Reference,
  this: Reference<Repository>,
  env: Env,
) -> crate::Result<AnnotatedCommit> {
  // ...
}
```

이 코드에서 두 이름은 더 이상 모호하지 않다.

```text
`crate::reference::Reference`
crate의 reference 모듈에 있는 Reference라고 주소를 모두 적음

`Reference<Repository>`
현재 스코프에 짧은 이름으로 가져온 napi-rs의 Reference
```

사람 이름으로 비유하면 같은 이름을 억지로 바꾸는 대신 “개발팀의 김민수”라고 소속까지 말한 것과 같다.
전체 경로는 다른 타입과 충돌하지 않고, 코드 생성기에도 실제 공개 타입 이름인 `Reference`를 전달한다.

생성 결과는 다음처럼 바뀐다.

```ts
getAnnotatedCommitFromReference(reference: Reference): AnnotatedCommit;
```

별칭 문제가 시작된 Rust 소스에서 바로 고쳤기 때문에 생성된 `index.d.ts`만 문자열로 사후 교체하는 것보다
원인에 가까운 수정이다.

## 7. 두 번째 버그를 이해하기 위한 JavaScript iterator 기초

### 7.1 반복 가능한 객체

배열은 `for...of`로 값을 하나씩 읽을 수 있다.

```js
for (const value of [10, 20, 30]) {
  console.log(value);
}
```

이렇게 반복할 수 있는 객체를 iterable이라고 한다. es-git의 브랜치 목록도 배열은 아니지만 반복할 수
있다.

```ts
const branches = repo.branches();

for (const branch of branches) {
  console.log(branch.name);
}
```

### 7.2 iterator의 `next()`

Iterator는 현재 위치를 기억하면서 값을 하나씩 꺼내는 객체다. 핵심 메서드는 `next()`다.

```js
const iterator = [10, 20][Symbol.iterator]();

iterator.next(); // { value: 10, done: false }
iterator.next(); // { value: 20, done: false }
iterator.next(); // { value: undefined, done: true }
```

`done: false`는 아직 반복 중이라는 뜻이고 `done: true`는 모든 값을 읽었다는 뜻이다.

### 7.3 `[Symbol.iterator]()`

`for...of`는 객체의 `[Symbol.iterator]()`를 호출해 iterator를 얻는다. Iterator 객체가 자기 자신을
반환하면 직접 `next()`로 읽을 수도 있고 `for...of`에도 사용할 수 있다.

```js
class Values {
  [Symbol.iterator]() {
    return this;
  }

  next() {
    // 다음 값을 반환한다.
  }
}
```

TypeScript에서 두 역할을 모두 만족하는 공통 인터페이스가 `IterableIterator<T>`다.

```text
Iterable
`for...of`가 iterator를 얻을 수 있음

Iterator
`next()`로 다음 값을 꺼낼 수 있음

IterableIterator
두 조건을 모두 만족함
```

## 8. 인터페이스, 클래스, `extends`, `implements`

### 8.1 인터페이스

인터페이스는 객체가 가져야 하는 모양을 설명한다. 실제 객체나 실행 코드를 만들지는 않는다.

```ts
interface Movable {
  move(): void;
}
```

이 인터페이스는 “Movable이라면 `move()`가 있어야 한다”는 타입 규칙이다.

### 8.2 클래스

클래스는 실제 객체를 만들 수 있는 설계도다.

```ts
class Animal {
  move() {
    console.log('moving');
  }
}

const animal = new Animal();
animal.move();
```

### 8.3 `extends`

클래스가 다른 클래스의 실제 구현과 프로토타입을 상속할 때 `extends`를 사용한다.

```ts
class Dog extends Animal {
  bark() {
    console.log('woof');
  }
}
```

### 8.4 `implements`

클래스가 인터페이스의 규칙을 만족한다고 선언할 때 `implements`를 사용한다.

```ts
class Car implements Movable {
  move() {
    console.log('driving');
  }
}
```

인터페이스가 `Car`에 실제 코드를 물려주는 것은 아니다. TypeScript가 `Car`에 필요한 멤버가 있는지
검사할 뿐이다.

```text
`extends`
다른 클래스의 구현을 상속한다.

`implements`
인터페이스가 요구하는 모양을 직접 구현했다고 선언한다.
```

## 9. `<>`와 제네릭을 읽는 법

`Iterator<BranchesItem, void, void>`에서 `<>`는 실행할 때 값을 전달하는 괄호가 아니다. 타입의 빈칸을
채우는 제네릭 type argument 문법이다.

```text
`openRepository('.')`
`()` 안에 실제 문자열 값을 전달한다.

`Array<string>`
`<>` 안에 배열 원소의 타입을 지정한다.
```

제네릭 인터페이스는 사용할 타입을 나중에 결정할 수 있는 설계도다.

```ts
interface Box<T> {
  value: T;
}
```

문자열 상자로 사용하면 `T`에 `string`을 넣는다.

```ts
const stringBox: Box<string> = {
  value: 'hello'
};
```

숫자 상자로 사용하면 `T`에 `number`를 넣는다.

```ts
const numberBox: Box<number> = {
  value: 10
};
```

`Iterator`는 타입 빈칸이 세 개다. 개념적으로 다음과 같이 정의된다.

```ts
interface Iterator<T, TReturn, TNext> {
  next(value?: TNext): IteratorResult<T, TReturn>;
}
```

각 자리는 다음 뜻이다.

```text
T
반복 중 하나씩 내보내는 값의 타입

TReturn
반복이 완전히 끝났을 때의 최종 반환 타입

TNext
호출자가 `next()` 안으로 전달할 수 있는 값의 타입
```

따라서 다음 타입은 이렇게 읽는다.

```ts
Iterator<BranchesItem, void, void>
```

```text
T       = BranchesItem
브랜치 조회 결과를 하나씩 내보낸다.

TReturn = void
반복이 끝난 뒤 별도의 최종 결과가 없다.

TNext   = void
`next()`를 호출할 때 의미 있는 입력값을 전달하지 않는다.
```

사람 말로 옮기면 다음과 같다.

> `BranchesItem`을 하나씩 내보내고, 반복 종료 시 별도 결과가 없으며, `next()` 호출에도 별도 값을 받지
> 않는 iterator.

반복 중에는 다음 모양이 나온다.

```ts
{
  value: branchItem, // BranchesItem
  done: false
}
```

반복이 끝나면 다음 모양이 나온다.

```ts
{
  value: undefined, // void
  done: true
}
```

`BranchesItem`은 여기서 실제 브랜치 객체를 전달한 것이 아니라 타입 이름이다. `<>` 안의 타입 정보는
TypeScript 검사에 사용되고 JavaScript 실행 시에는 대부분 사라진다.

## 10. 기존 iterator 선언이 왜 실패했는가

napi-rs가 생성한 선언은 다음 모양이었다.

```ts
export declare class Branches extends Iterator<BranchesItem, void, void> {
  next(value?: void): IteratorResult<BranchesItem, void>;
}
```

문제는 TypeScript가 불러오는 표준 라이브러리 정의에 따라 `Iterator`의 의미가 달라질 수 있다는 점이다.

ES2022 환경에서 `Iterator`는 주로 다음과 같은 인터페이스로 보인다.

```ts
interface Iterator<T, TReturn, TNext> {
  next(value?: TNext): IteratorResult<T, TReturn>;
}
```

인터페이스는 타입 규칙이며 상속할 실제 클래스 구현이 아니다. 따라서 클래스 선언이 인터페이스를
`extends`하려 하면 TypeScript가 오류를 낸다.

```text
`Branches`는 class
`Iterator`는 ES2022 타입 정의에서 interface
class extends interface -> 오류
```

오류 메시지의 의미도 그대로다.

```text
Cannot extend an interface 'Iterator'. Did you mean 'implements'?
```

반면 최신 `esnext.iterator` 정의와 런타임에는 iterator helper를 제공하는 전역 `Iterator` 클래스가
있을 수 있다.

```js
iterator.map(...)
iterator.filter(...)
iterator.take(...)
```

기존 생성 선언은 이 최신 환경을 전제로 하면 자연스럽지만, ES2022만 사용하는 소비자에게는 유효하지
않았다.

## 11. 런타임의 Iterator와 타입 선언을 구분해야 한다

리뷰 중 처음에는 “napi-rs iterator는 런타임에 `next()`와 `[Symbol.iterator]()`만 구현한다”고
정리했다. napi-rs 코드를 다시 확인하니 이 설명은 정확하지 않았다.

napi-rs는 런타임에 전역 `Iterator` 생성자가 있으면 iterator 클래스의 프로토타입을
`Iterator.prototype`에 연결한다. 최신 런타임에서는 iterator helper를 상속할 수 있다.

```text
전역 Iterator가 있는 런타임

Branches.prototype
        │
        ▼
Iterator.prototype
```

전역 `Iterator`가 없는 런타임에서는 기본 iterator protocol만 사용할 수 있다. 따라서 모든 지원 환경에서
공통으로 약속할 수 있는 최소 타입은 다음 두 멤버다.

```ts
next(): IteratorResult<T>;
[Symbol.iterator](): IterableIterator<T>;
```

이번 수정은 최신 런타임에서 helper가 절대 없다고 주장하는 것이 아니다. 타입 선언을 특정 최신
런타임에 묶지 않고 모든 지원 환경에서 공통으로 제공되는 형태로 낮춘 것이다.

이 재검토 때문에 구현 주석과 생성된 `index.d.ts`의 문서도 다시 고쳤다. 현재 설명은 다음과 같다.

```text
This type implements the standard iterable iterator protocol.
```

## 12. iterator 선언을 어떻게 수정했는가

최종 선언은 다음 형태다.

```ts
export declare class Branches implements IterableIterator<BranchesItem> {
  next(value?: void): IteratorResult<BranchesItem, void>;
  [Symbol.iterator](): IterableIterator<BranchesItem>;
}
```

이 선언은 `Branches`가 모든 런타임에서 제공하는 두 동작을 직접 표현한다.

```text
`next()`
다음 브랜치 결과를 꺼낸다.

`[Symbol.iterator]()`
`for...of`가 이 객체를 iterator로 사용할 수 있게 한다.
```

ES2022에서도 인터페이스를 올바르게 `implements`하므로 타입 검사할 수 있고, ESNext에서도 동일하게
유효하다.

## 13. 생성 파일만 손으로 고치지 않은 이유

`index.d.ts`는 저장소에 커밋되어 있지만 빌드가 다시 생성하는 파일이다. 이 파일만 직접 고치면 다음
빌드에서 원래 잘못된 선언으로 돌아간다.

저장소에는 이미 `transforms/dts.mjs`라는 후처리 단계가 있다.

```text
Rust와 napi-rs가 원본 `index.d.ts` 생성
                │
                ▼
`transforms/dts.mjs` 실행
                │
                ├── 기존 string enum 변환
                ├── 기존 Credential union 변환
                └── 새 iterator 선언 변환
```

새 `transformIteratorClasses()`는 AST에서 `Iterator`를 상속하는 클래스 선언을 찾는다.

```js
.find(
  j.ClassDeclaration,
  node =>
    node.superClass?.type === 'Identifier' &&
    node.superClass.name === 'Iterator'
)
```

찾은 선언에서 다음 작업을 수행한다.

1. `Iterator`의 첫 번째 제네릭 타입인 yield 타입을 보존한다.
2. `extends Iterator<...>`를 제거한다.
3. `implements IterableIterator<Yield>`를 추가한다.
4. `[Symbol.iterator]()` 선언이 없으면 추가한다.
5. 더 이상 사실과 맞지 않는 생성 문서의 iterator helper 설명을 바꾼다.

변환 대상은 현재 11개다.

```text
BlameHunks
BlameHunksByLine
Branches
ConfigEntries
Deltas
IndexEntries
Notes
ReflogIter
StashListIter
StatusesIter
TreeIter
```

변환을 다시 실행해도 파일이 더 바뀌지 않는 것도 확인했다. 한 번 변환된 클래스는 더 이상
`extends Iterator`가 아니므로 같은 변경이 중복 적용되지 않는다.

## 14. 회귀 테스트

`tests/typecheck-consumer`는 라이브러리 저장소 자체가 아니라 실제 소비자 프로젝트의 관점으로 타입을
검사하기 위해 추가했다.

```text
tests/typecheck-consumer/
├── consumer.ts
└── tsconfig.json
```

`consumer.ts`에는 두 오류를 각각 통과해야 하는 코드가 있다.

```ts
import { openRepository } from '../../index.js';

async function main() {
  const repo = await openRepository('.');

  for (const branch of repo.branches()) {
    branch.name;
  }

  const reference = repo.head();
  repo.getAnnotatedCommitFromReference(reference);
}

void main();
```

첫 번째 부분은 iterator 선언을 검증한다.

```ts
for (const branch of repo.branches()) {
  branch.name;
}
```

`repo.branches()`가 ES2022 타입 환경에서도 반복 가능한지 확인한다.

두 번째 부분은 공개 Reference 타입을 검증한다.

```ts
const reference = repo.head();
repo.getAnnotatedCommitFromReference(reference);
```

`repo.head()`가 반환한 공개 `Reference`를 그대로 함수에 전달할 수 있어야 한다. 생성 선언에
`GitReference`가 다시 나타나면 이 타입 검사에서 실패한다.

테스트의 `tsconfig.json`은 이슈의 소비자 조건을 고정한다.

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "skipLibCheck": false,
    "types": ["node"],
    "noEmit": true
  },
  "include": ["consumer.ts"]
}
```

패키지 스크립트에는 다음 명령을 추가했다.

```json
{
  "scripts": {
    "test:types": "tsc -p tests/typecheck-consumer"
  }
}
```

그리고 GitHub Actions의 `check` job에서 실행한다.

```yaml
- name: Typecheck the generated bindings under a consumer config
  run: yarn test:types
```

이 검사는 저장소의 편한 `ESNext + skipLibCheck: true` 설정만 통과하는지 보지 않고, 실제로 문제가
발생했던 엄격한 소비자 설정에서도 배포 선언이 유효한지 확인한다.

## 15. 왜 기존 CI가 발견하지 못했는가

es-git 자체 `tsconfig.json`의 주요 조건은 다음과 같다.

```json
{
  "compilerOptions": {
    "target": "ESNext",
    "lib": ["ESNext", "DOM", "DOM.Iterable"],
    "skipLibCheck": true
  }
}
```

두 설정이 문제를 가렸다.

```text
`ESNext`
상속 가능한 최신 Iterator 타입 정의를 볼 수 있다.

`skipLibCheck: true`
생성된 선언 파일 내부 오류를 엄격하게 검사하지 않는다.
```

```text
저장소 자체 검사
ESNext + skipLibCheck true
            │
            ▼
          통과

실제 이슈 소비자
ES2022 + skipLibCheck false
            │
            ▼
          실패
```

라이브러리의 개발 환경만 검사하면 소비자가 사용하는 더 보수적인 표준 라이브러리나 엄격한 선언 검사
환경을 놓칠 수 있다는 사례다.

## 16. 검토 과정에서 수정한 설명과 문서

첫 PR 버전은 iterator 변환 방향 자체는 맞았지만 그 근거를 다음처럼 설명했다.

```text
These classes only ever implement `next()` and `[Symbol.iterator]()` at runtime.
```

napi-rs의 `create_iterator`만 보면 그렇게 판단하기 쉽지만, 클래스 등록 단계의
`setup_iterator_class`는 전역 `Iterator`가 있을 때 `Iterator.prototype`을 연결한다. 따라서 “런타임에는
helper가 절대 없다”는 설명은 과도한 단정이었다.

재검토 후 다음 세 부분을 함께 고쳤다.

- `transforms/dts.mjs`의 구현 설명을 런타임 조건에 맞게 수정
- 생성된 11개 클래스의 문서에서 무조건 `Iterator`를 상속한다는 설명 제거
- PR 본문을 구현 세부 주장보다 실제 변경과 회귀 테스트 중심으로 축약

이 과정에서 코드가 통과하는지만 확인하지 않고 주석과 PR의 근거가 실제 upstream 코드와 일치하는지도
검증해야 한다는 점을 다시 확인했다.

## 17. 로컬 검증 기록

최신 커밋을 만들기 전에 다음 검증을 실행했다.

- `yarn install --immutable`: 통과
- `yarn test:types`: 통과
- `yarn biome check`: 통과
- `git diff --check`: 통과
- `yarn transform:dts` 재실행: 추가 변경 없음

현재 컴퓨터에는 `cargo`가 설치되어 있지 않아 `cargo fmt --all --check`, `cargo check`와 전체 네이티브
빌드는 실행하지 못했다. Rust 변경은 타입 경로 표현 한 곳이지만, 최종 전체 빌드는 GitHub Actions가
실행된 뒤 다시 확인해야 한다.

저장소 CI에는 macOS aarch64 빌드 결과가 커밋된 `index.js`, `index.d.ts`와 다른지 확인하는 단계가 이미
있다. 이 job이 실행되면 Rust 소스와 변환 단계에서 다시 생성한 선언이 커밋 결과와 같은지도 검증한다.

## 18. 커밋과 PR 정리

처음 원격 커밋은 `0cebbc6368d9caed71eaef4538bf9f1673725f39`였다. 런타임 설명과 생성 문서를
고친 뒤 기존 한 커밋에 amend하고 `--force-with-lease`로 다음 커밋을 게시했다.

```text
1030f63006f0e07355123453805cb08fe401af25
fix: generated declarations fail with skipLibCheck disabled
```

강제 푸시 전에 원격 브랜치가 예상한 기존 SHA인지 확인했고, 정확히 일치할 때만 갱신되도록 lease에
기존 SHA를 지정했다. 다른 컴퓨터나 작업자가 원격 브랜치를 먼저 갱신했다면 푸시가 실패하도록 한
것이다.

커밋 작성자는 `GiHoon1123` 한 명이며 공동 저자나 도구 attribution은 없다. PR 본문은 다음 정보만 남겨
짧게 정리했다.

- `skipLibCheck: false` 소비자에서 선언이 실패한다는 문제
- 공개 `Reference` 타입 사용
- iterator를 이식 가능한 `IterableIterator` 형태로 선언
- 소비자 타입 검사 fixture와 CI 추가
- `Closes #215`

## 19. 현재 PR 상태

2026-09-22 확인 결과는 다음과 같다.

- PR은 OPEN이며 draft가 아니다.
- 최신 커밋은 `1030f63006f0e07355123453805cb08fe401af25`다.
- base와 충돌하지 않아 `MERGEABLE`이다.
- 코드 오너의 리뷰가 아직 없어 `REVIEW_REQUIRED`다.
- 전체 merge state는 `BLOCKED`다.
- 사람 리뷰와 메인테이너 댓글은 아직 없다.

GitHub Actions 상태는 `action_required`다. 테스트 assertion 실패가 아니라 외부 기여자의 workflow를
메인테이너가 승인해야 실행할 수 있다는 뜻이다. 아직 최신 SHA에 대한 실제 CI 빌드가 시작되지 않았다.

Vercel은 failure로 표시되지만 외부 기여자가 Toss 팀 배포를 실행할 권한이 없어서 발생한 상태다. 코드
품질이나 테스트 실패로 분류하지 않는다.

## 20. 이번 작업에서 기억할 것

### 생성 파일의 오류는 생성 원인까지 고친다

`index.d.ts`만 손으로 바꾸면 다음 빌드에서 돌아온다. Rust 타입 이름 문제는 Rust 소스에서 고치고,
napi-rs가 반복해서 만드는 iterator 선언은 기존 dts transform 단계에서 고쳤다.

### Rust 내부 이름이 공개 API에 새어 나올 수 있다

지역 별칭은 Rust 안에서만 편리한 이름이지만 코드 생성기는 소스의 식별자를 공개 선언에 사용할 수
있다. 매크로와 바인딩 생성기를 사용할 때는 내부 이름이 어느 결과물까지 전달되는지 확인해야 한다.

### `<>`는 값을 넘기는 문법이 아니다

`Iterator<BranchesItem, void, void>`는 iterator 객체에 값을 넣는 코드가 아니다. 제네릭 타입 정의의 세
빈칸을 `BranchesItem`, `void`, `void`라는 타입으로 채운 결과다. 실제 값 전달은 `next(value)`의 `()`에서
일어난다.

### 런타임 기능과 최소 공개 타입은 다를 수 있다

최신 런타임이 iterator helper를 제공하더라도 모든 지원 런타임이 제공한다고 선언하면 안 된다. 공개
타입은 지원 범위 전체에서 지킬 수 있는 계약이어야 한다.

### 저장소 자체 설정만으로 소비자 환경을 대표할 수 없다

라이브러리 저장소가 ESNext와 `skipLibCheck: true`를 사용한다고 모든 소비자가 같은 설정을 사용하는 것은
아니다. 배포 타입 버그는 작은 consumer fixture가 가장 직접적으로 잡을 수 있다.

### 리뷰에서는 설명의 사실성도 확인한다

수정 방향이 맞아도 근거가 틀릴 수 있다. 이번에는 napi-rs의 다른 초기화 경로를 확인하면서 런타임
설명을 바로잡았다. 코드, 주석, 커밋 설명과 PR 본문은 서로 같은 사실을 말해야 한다.

## 21. 다음에 할 일

- [ ] 메인테이너가 GitHub Actions 실행을 승인했는지 확인
- [ ] `check`, Rust format/clippy와 플랫폼별 build 결과 확인
- [ ] macOS aarch64의 generated bindings 일치 검사 확인
- [ ] 코드 오너 리뷰가 오면 요구 의도부터 확인하고 필요한 변경과 검증을 추가
- [ ] 새 리뷰 라운드는 기존 기록을 지우지 않고 이 문서 아래에 시간순으로 추가
- [ ] PR이 merge되면 문서를 `typescript/merge/`로 옮기고 최종 결과 기록

## 22. 2026-10-08 리뷰: ES2022 오류를 고치며 ESNext 타입을 잃었다

Marshall Ku가 PR #227의 원래 재현과 현재 수정 결과를 확인했다. `main`의 선언 파일에서는
`skipLibCheck: false` 소비자가 `TS2689` 11건과 `GitReference` 관련 오류 등을 냈고, 당시 PR
브랜치에서는 이 오류가 사라졌다. 그러나 `Branches` 같은 iterator 클래스가 제공하던 helper 타입도
함께 사라졌다. 리뷰어는 ESNext 소비자에서 다음 코드가 `main`에서는 컴파일되지만 PR에서는
`TS2339: Property 'map' does not exist on type 'Branches'`로 실패함을 확인했다.

```ts
repo.branches().map(branch => branch.name).toArray();
```

이전 변환은 `extends Iterator<BranchesItem, void, void>`를 없애고
`implements IterableIterator<BranchesItem>`를 붙였다. 여기서 `implements`는 클래스가 해당 타입의
조건을 만족하는지 확인할 뿐, 인터페이스에 적힌 메서드를 클래스 타입에 새로 붙여 주지 않는다.
변환기가 `[Symbol.iterator]()`를 직접 추가하고 생성된 `next()`를 유지했기 때문에 기본 반복은
가능했다. 하지만 `IterableIterator`에는 `map()`과 `toArray()` 타입이 없어, 최신 런타임에서 실제로
사용 가능한 helper를 TypeScript 소비자는 호출할 수 없게 됐다.

`skipLibCheck: false` 오류를 고친 것과 기존 소비자의 타입을 유지하는 것은 별개의 조건이었다.
ES2022 테스트 하나만으로 두 조건을 모두 확인했다고 판단한 것이 이번 누락의 원인이다.

## 23. iterator 선언 수정

리뷰어가 손으로 `Branches` 선언을 바꿔 두 소비자 설정에서 검증한 형태를 변환기에 적용했다.

```ts
export declare class Branches {
  next(value?: void): IteratorResult<BranchesItem, void>;
  [Symbol.iterator](): Branches;
}

export interface Branches extends IteratorObject<BranchesItem, void, void> {}
```

같은 이름의 클래스와 인터페이스는 TypeScript에서 하나의 인스턴스 타입으로 합쳐진다. 클래스
선언은 실제 export가 클래스라는 사실과 기본 메서드를 표현한다. 인터페이스는
`IteratorObject`의 타입을 합쳐 ESNext에서 helper 메서드가 보이게 한다. 클래스가 `Iterator`를
직접 상속하지 않으므로 ES2022의 `TS2689`도 다시 발생하지 않는다. `IteratorObject`는
TypeScript 5.6 이상에서 제공되며, 이전 `extends Iterator` 선언도 같은 버전의 타입 정의에
의존하고 있었다.

`transforms/dts.mjs`는 napi-rs가 출력한 `extends Iterator<Yield, Return, Next>`의 세 타입
인자를 그대로 `IteratorObject`에 전달한다. 클래스의 `extends`를 제거하고, 자기 자신을 반환하는
`[Symbol.iterator]()`를 선언한 뒤 같은 이름의 인터페이스를 클래스 다음에 넣는다. 수정된
`index.d.ts`의 11개 iterator 클래스가 모두 같은 형태인지 확인했다.

첫 PR 변환기는 napi-rs의 긴 주석 문구를 정규식으로 정확히 맞춰 짧은 문장으로 바꿨다. 단어가
조금만 바뀌어도 주석만 그대로 남는다는 리뷰 지적이 있었다. 새 변환에서는 그 주석 치환 코드가
필요 없어져 제거했다. 기존 helper 문서 링크는 생성 선언에 남아 있다.

## 24. 생성 형식과 소비자 테스트의 경계

기존 코드는 `Iterator`에 타입 인자가 없을 때 yield 타입을 `any`로 대체했다. 이 경우 생성기
출력이 예상과 달라져도 공개 타입만 조용히 느슨해진다. 이제 세 타입 인자가 없으면 변환기가
`Expected Iterator<Yield, Return, Next> for <클래스 이름>` 오류를 낸다. 타입 없는
`extends Iterator` 예제로 실제 오류가 발생하는 것도 확인했다.

ESNext용 소비자 파일과 `tsconfig.esnext.json`도 추가했다. 여기서는 `lib: ["ESNext"]`로
`repo.branches().map(...).toArray()`의 반환값을 `string[]`에 넣는다. 기존 ES2022 소비자
테스트는 그대로 유지했고, `yarn test:types`가 두 설정을 차례로 검사하도록 바꿨다. CI의 기존
`yarn test:types` 단계도 같은 명령을 사용하므로 별도 workflow 변경은 필요하지 않다.

## 25. Reference 이름과 문서 동기화

Rust 함수는 `crate::reference::Reference`라는 전체 경로를 사용한다. 같은 파일의 napi-rs
`Reference<Repository>`와 이름이 겹쳐 별칭을 다시 쓰고 싶어질 수 있지만, napi-rs는 소스에
적힌 타입 이름을 `index.d.ts`에 옮긴다. 이를 알 수 있도록 매개변수 바로 위에 한 줄 주석을
남겼다.

영문 API 문서는 생성 명령으로 갱신했고, 한글 문서의 매개변수 타입도 `GitReference`에서
`Reference`로 고쳤다. 같은 영문 설명에 있던 `Creates a Annotated Commit`과
`Reference to creates a Annotated Commit` 문법도 Rust 원본, 생성 선언, 영문 문서에서 함께
수정했다. 새 영문 문서 생성 결과는 의도한 한 페이지에만 변경이 생겼다.

## 26. 커밋과 검증

리뷰 지적을 다음 네 커밋으로 나눠 `GiHoon1123:fix/skiplibcheck-declaration-errors`에
푸시했다. PR은 그대로 #227이다.

```text
6d71ba2 fix: retain iterator helper types in declarations
f707470 fix: reject incomplete generated iterator types
1275cf4 test: cover iterator helpers in ESNext consumers
a4c236d docs: correct annotated commit reference type
```

로컬에서 `yarn install --immutable`, `yarn test:types`, `yarn biome check`,
`git diff --check`가 통과했다. `yarn transform:dts`를 다시 실행해도 `index.d.ts`는 바뀌지
않았다. 이전 원본 선언에 이번 Rust 타입 이름과 문구 수정을 반영해 변환한 결과도 커밋된
`index.d.ts`와 일치한다.

이 컴퓨터에는 `cargo`와 `rustfmt`가 없어 Rust 전체 빌드와 포맷 검사는 실행하지 못했다.
Rust의 문서 문구와 주석만 바꿨지만, CI의 Rust 검사와 napi-rs 재생성 검사는 별도로 확인해야 한다.
리뷰가 게시된 시점에는 외부 기여자의 GitHub Actions 실행 승인이 아직 필요한 상태였다.

푸시 뒤 PR 본문의 낡은 `IterableIterator` 설명도 실제 구현에 맞게 고쳤다. 현재 본문은
`Reference` 타입, 클래스와 `IteratorObject` 인터페이스의 병합, ES2022·ESNext 소비자 검사를
짧게 설명한다.

## 27. 주석 변환 재검토

네 커밋을 푸시한 뒤 생성된 `index.d.ts`를 다시 읽다가, 이전 napi-rs 설명에
“This type extends JavaScript's `Iterator`”가 그대로 남아 있음을 발견했다. 긴 문구를
정확히 일치시키던 정규식을 제거한 것만으로는 리뷰어의 문서 지적을 끝낸 게 아니었다.

변환기는 이제 각 iterator 클래스 바로 앞의 주석에서 핵심 문구와 `@see` 링크를 찾아
설명을 “Iterator helper methods are available when supported by the runtime.”으로 바꾼다.
링크는 보존한다. 예상한 주석이 없으면 클래스 이름을 포함한 오류를 내므로, napi-rs가 설명을
바꾸었을 때 잘못된 공개 문서가 조용히 배포되지 않는다.

AST의 클래스 선언과 주석을 한 번에 수정하면서 인터페이스를 삽입하면 recast가 주석 위치를
재구성하다 `Invariant failed`를 냈다. 클래스와 인터페이스를 AST로 먼저 출력한 다음,
변환된 각 클래스 앞에 붙은 주석만 검사하고 교체하는 방식으로 정리했다. 주석을 지운
가상 입력에서는 `Expected iterator helper comment for Missing` 오류가 발생했고,
정상 원본에서는 11개 주석 모두 바뀌었다.

별도 커밋 `3a32322 fix: keep iterator helper comments in sync`를 같은 PR 브랜치에
푸시했다. 이후 `yarn biome check`, `yarn test:types`, `git diff --check`가 통과했고,
`yarn transform:dts` 재실행 결과 `index.d.ts` 해시는 변하지 않았다.

## 28. 리뷰어에게 수정 내용 전달

2026-10-08 `GiHoon1123` 계정으로 PR에 한 개의 답글을 남겼다.

https://github.com/toss/es-git/pull/227#issuecomment-6055359061

iterator helper 타입 회귀를 잡아 준 점에 감사하고, 클래스와 `IteratorObject` 인터페이스
병합, ESNext 소비자 검사, 예상 밖 타입 인자·주석의 오류 처리, 영문·한글 `Reference` 문서
수정을 완료했다고 알렸다. 로컬에서 통과한 명령은 `yarn test:types`와
`yarn biome check` 두 개만 명시했다.

## 29. 2026-10-08 Rust 빌드와 생성 파일 확인

앞선 검증에서는 Rust toolchain이 없어 `index.d.ts`를 실제 napi-rs 빌드로 다시 만들지
못했다. 저장소 CI의 macOS arm64 빌드와 같은 Rust 1.91.0을 설치해 다시 확인했다.

첫 `yarn build --target aarch64-apple-darwin`은 `libz-sys` 빌드 중 CMake를 찾지 못해
멈췄다. `cmake`를 설치한 뒤 같은 명령을 재실행했고 release 빌드와 선언 변환이
완료됐다. 이후 `git diff -- index.js index.d.ts`에는 변경이 없었다. 따라서 커밋된
바인딩 파일이 이번 macOS arm64 빌드 결과와 일치한다.

추가로 다음 검사가 통과했다.

```text
cargo fmt --all --check
cargo clippy
yarn test:types
git diff --check
```

검증 후 es-git 작업 트리는 깨끗하다. 이 결과는 로컬 macOS arm64 빌드에 대한
확인이다. PR의 다른 운영체제와 아키텍처 빌드는 CI 결과를 확인해야 한다.

## 30. 리뷰 반영 후 머지 — 2026-10-13 확인

[PR #227](https://github.com/toss/es-git/pull/227)은 2026-10-11에 머지됐다.
GitHub에 기록된 시각은 `2026-10-11T06:53:29Z`, 한국시간으로 15시 53분 29초다.
병합한 메인테이너는 `marshallku`이며, 머지 커밋은
[`83524a91`](https://github.com/toss/es-git/commit/83524a91a3d6d9a17ddfc460b961b7763c98899a)이다.
[이슈 #215](https://github.com/toss/es-git/issues/215)도 1초 뒤 `completed` 사유로 닫혔다.

### 30.1 리뷰에서 지적한 호환성을 유지한 결과

최초 수정은 ES2022 소비자의 선언 오류를 없앴지만, ESNext 소비자가 사용하던 iterator helper 타입을
잃게 했다. 최종 수정에서는 이 두 조건을 함께 만족시키는 방향으로 바꿨다.

- Rust의 지역 별칭 `GitReference` 대신 공개 타입 `Reference`가 생성 선언에 나타나도록 한다.
- iterator 클래스와 같은 이름의 `IteratorObject` 인터페이스를 병합한다.
- ES2022 소비자의 `skipLibCheck: false` 검사와 ESNext 소비자의 `.map().toArray()` 검사를 함께 실행한다.
- 예상하지 못한 타입 인자나 helper 주석을 만나면 변환기가 오류를 내도록 한다.
- 영문·한글 API 문서의 `Reference` 타입과 생성된 iterator 설명을 맞춘다.

메인테이너는 [10월 8일 리뷰](https://github.com/toss/es-git/pull/227#pullrequestreview-5456538344)에
“LGTM! Thanks for your contribution”이라고 남겼다.
10월 13일 조회한 해당 리뷰 객체의 상태는 `DISMISSED`다. 따라서 이를 현재 유효한 승인 상태로
기록하지는 않는다. 긍정적인 리뷰 문구와 실제 머지 여부는 각각 확인된 사실로 구분한다.

### 30.2 최종 브랜치에는 main 병합도 포함됐다

문서에 기록한 마지막 수정 커밋은 `3a323225`였다. 이후 메인테이너가 `main`을 PR 브랜치에
병합한 커밋 `f57731b9`, `cdd7a99b`가 추가됐다. 최종 PR head는
`cdd7a99bef88f031d35aa206cd344db6890d0543`이다.

현재 GitHub의 PR 커밋 목록과 파일 diff에는 별도 작업인 commit signing buffer API 변경
([PR #221](https://github.com/toss/es-git/pull/221))도 보인다.
이 부분을 이번 선언 수정에서 직접 구현한 기능으로 세지 않는다.
머지 결과를 기록할 때는 최종 diff 전체와 우리가 작성한 변경의 범위를 구분해야 한다.

### 30.3 최종 CI 검사 37개 통과

최종 head의 GitHub check runs를 확인한 결과, 37개 모두 `completed / success`였다.

| 검사 범위 | 개수 | 결과 |
| --- | --- | --- |
| Linux·macOS·Windows의 Node 20·24·26 테스트 조합 | 24 | 모두 통과 |
| 운영체제·아키텍처별 네이티브 빌드 | 10 | 모두 통과 |
| check, format, clippy | 3 | 모두 통과 |

Linux 테스트에는 GNU와 musl, x86_64와 aarch64 조합이 포함됐다.
macOS와 Windows 테스트도 x86_64·aarch64에서 실행됐다.
빌드 목록에는 Android 대상 두 개도 포함돼 있지만, 이를 Android 런타임 테스트 통과로
확대해 기록하지 않는다.

Vercel은 별도의 commit status로 `failure`가 남아 있었다.
설명은 `Authorization required to deploy.`다.
이는 Toss 팀의 배포 승인이 필요하다는 상태이지, 위 빌드·타입 검사·테스트가 실패했다는 뜻은 아니다.
따라서 모든 상태가 녹색이었다고 적는 대신, CI check runs 37개 통과와 Vercel 승인 필요를 나눠 기록한다.

이번 갱신에서는 GitHub의 PR, 리뷰, 커밋과 CI 결과를 확인했다.
로컬 Rust 빌드와 테스트를 새로 실행하지 않았으며, 29절의 로컬 검증은 당시 커밋에 대한 기록으로 남긴다.
머지와 이슈 종료는 확인했지만, 수정이 포함된 npm 배포 버전은 이번에 확인하지 않았다.

### 30.4 문서 이동과 남길 교훈

상단 상태와 최종 커밋을 갱신하고 이 절을 덧붙인 뒤, 문서를 `typescript/open`에서
`typescript/merge`로 옮겼다. 기존 설명과 시행착오, 리뷰 대응, 로컬 검증 내용은 삭제하지 않았다.
21절의 체크리스트와 19절의 PR 상태도 당시 상황을 보여 주는 기록으로 유지한다.

이번에는 실행 코드가 아니라 타입 선언만 바꿨는데도 기존 소비자의 기능을 잃게 할 수 있었다.
처음 보고된 환경에서 오류가 사라진다는 것과, 이미 지원하던 다른 환경의 사용법이 유지된다는 것은
별개의 검증이다. ES2022와 ESNext 소비자 테스트를 함께 둔 이유도 여기에 있다.
