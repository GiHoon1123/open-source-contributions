# [Nest OSS 기여] nestjs/graphql - `fast-glob`에서 `tinyglobby`로 교체하면서 `typePaths` 동작을 유지한 작업 (PR #4078)

> 오픈소스 기여 기록 및 학습 노트. 이 문서는 PR #4078 하나만 다룬다. **최종 상태: 🟢 MERGED.**
>
> 이번 작업은 겉으로 보면 단순한 의존성 교체다.  
> 하지만 실제 핵심은 `fast-glob`과 `tinyglobby`의 기본 동작 차이를 확인하고, Nest GraphQL 사용자가 기존 `typePaths` 설정을 바꾸지 않아도 되도록 중간 helper에서 그 차이를 흡수한 것이다.

---

## 1. 개요

### 1.1 프로젝트 소개

- **저장소**: https://github.com/nestjs/graphql
- **npm 패키지**: `@nestjs/graphql`
- **성격**: NestJS에서 GraphQL 서버를 만들 수 있게 해주는 공식 패키지
- **작업 위치**: `/Users/dustin/Desktop/common/open-source/nestjs-graphql`
- **작업 브랜치**: `fix-3731-switch-to-tinyglobby`
- **이슈**: https://github.com/nestjs/graphql/issues/3731
- **PR**: https://github.com/nestjs/graphql/pull/4078
- **최초 제출 커밋**: `222fe75ebfd61c4444a284d266dc2f2f5fcea89b`
- **최종 작업 커밋**: `dd941d8222cde69c5fc600cf60d990c92dce90fa`
- **머지 전 충돌 해소 커밋**: `1d9e7ada2d230039140c34d85061db6ae484baa1`
- **머지 커밋**: `7687fc66f6b41017de4b66edc333ac8a0875b8e6`
- **상태**: 🟢 MERGED (2026-08-17 09:47:14 UTC / 2026-08-17 18:47:14 KST)

### 1.2 이슈 (#3731)

**제목**: `Switch from fast-glob to tinyglobby`

이슈의 요구사항은 `@nestjs/graphql` 패키지에서 `fast-glob` 의존성을 제거하고 `tinyglobby`를 사용하자는 것이다.

이슈 작성자는 다음 이유를 들었다:

- `fast-glob`은 의존성 수가 더 많다.
- `tinyglobby`는 더 작은 dependency graph를 가진다.
- 최근 JS/TS 생태계에서 `tinyglobby` 채택이 늘고 있다.
- Vite, SWC, tsup, nx, lerna 같은 도구들이 이미 `tinyglobby`를 사용한다.

기존에도 같은 방향의 PR #3677이 있었지만, 당시에는 maintainer가 "다른 패키지로 옮길 계획이 없다"고 닫았다.  
이번에는 이슈 #3731에 내가 "작게 잡고 검증해서 작업해보겠다"고 코멘트를 남겼고, Kamil이 `sounds good`이라고 답해서 진행했다.

### 1.3 한 줄 요약

> `@nestjs/graphql`의 GraphQL schema file lookup 경로에서 `fast-glob`을 `tinyglobby`로 교체하되, absolute path, relative path, negative pattern, repeated pattern, directory pattern 처리 방식이 기존과 달라지지 않도록 `globPaths()` helper를 추가하고 테스트로 고정했다.

### 1.4 건드린 파일

| 파일 | 역할 |
|---|---|
| `packages/graphql/package.json` | `fast-glob` 제거, `tinyglobby` 추가 |
| `yarn.lock` | dependency graph 갱신 |
| `packages/graphql/lib/graphql-types.loader.ts` | GraphQL SDL 파일 탐색 시 `globPaths()` 사용 |
| `packages/graphql/lib/graphql-definitions.factory.ts` | definitions watch mode에서 `globPaths()` 사용 |
| `packages/graphql/lib/utils/glob.util.ts` | `tinyglobby`를 감싼 내부 helper 추가 |
| `packages/graphql/tests/graphql-types.loader.spec.ts` | 실제 `.graphql` 파일 로딩 흐름 테스트 |
| `packages/graphql/tests/graphql-definitions.factory.spec.ts` | watch mode가 glob 결과를 `chokidar.watch()`에 넘기는지 테스트 |
| `packages/graphql/tests/utils/glob.util.spec.ts` | helper의 compatibility behavior 테스트 |

---

## 2. 배경 지식

### 2.1 GraphQL이란

GraphQL은 API를 만드는 방식 중 하나다.

REST에서는 보통 URL을 여러 개 둔다:

```txt
GET /users/1
GET /users/1/posts
GET /users/1/followers
```

GraphQL에서는 보통 endpoint 하나를 둔다:

```txt
POST /graphql
```

그리고 클라이언트가 요청 본문에 "내가 원하는 데이터 모양"을 적는다.

```graphql
query {
  user(id: 1) {
    id
    name
    posts {
      title
    }
  }
}
```

서버는 그 모양대로 데이터를 돌려준다.

### 2.2 GraphQL schema란

GraphQL 서버는 어떤 데이터를 줄 수 있는지 미리 정의해야 한다.  
이 정의를 schema라고 한다.

예:

```graphql
type User {
  id: ID!
  name: String!
}

type Query {
  user(id: ID!): User
}
```

뜻은 다음과 같다:

- `User`라는 타입이 있다.
- `User`는 `id`, `name` 필드를 가진다.
- 클라이언트는 `user(id: ID!)` 쿼리를 호출할 수 있다.
- 그 결과는 `User`다.

### 2.3 Nest GraphQL의 schema-first 방식

Nest GraphQL에서는 schema를 만드는 방식이 크게 두 가지다.

첫 번째는 code-first 방식이다.  
TypeScript class와 decorator로 schema를 만든다.

두 번째는 schema-first 방식이다.  
개발자가 직접 `.graphql` 파일을 만들고 Nest가 그 파일들을 읽는다.

예:

```graphql
type Cat {
  id: ID!
  name: String!
}

type Query {
  cat: Cat
}
```

그리고 Nest 설정에 이렇게 적는다:

```ts
GraphQLModule.forRoot({
  typePaths: ['./src/**/*.graphql'],
});
```

이 `typePaths`가 이번 작업의 중심이다.

### 2.4 `typePaths`란

`typePaths`는 Nest GraphQL이 읽어야 할 `.graphql` 파일 위치를 나타내는 설정이다.

예:

```ts
typePaths: ['./src/**/*.graphql']
```

이 뜻은:

```txt
src 폴더 아래 모든 하위 폴더를 뒤져서 .graphql 파일을 찾아라
```

파일 구조가 이렇게 있으면:

```txt
src/
  users/
    user.graphql
  posts/
    post.graphql
```

`./src/**/*.graphql`은 두 파일을 모두 찾는다.

```txt
src/users/user.graphql
src/posts/post.graphql
```

### 2.5 glob pattern이란

glob pattern은 파일 경로를 찾는 규칙이다.

대표적인 예:

| pattern | 의미 |
|---|---|
| `*.graphql` | 현재 폴더의 `.graphql` 파일 |
| `src/*.graphql` | `src` 바로 아래의 `.graphql` 파일 |
| `src/**/*.graphql` | `src` 아래 모든 하위 폴더의 `.graphql` 파일 |
| `!src/**/*.generated.graphql` | 이 패턴에 맞는 파일은 제외 |

정규식이 문자열을 찾는 규칙이라면, glob은 파일 경로를 찾는 규칙이라고 보면 된다.

---

## 3. 기존 코드는 어떻게 동작했나

### 3.1 기존 의존성: `fast-glob`

기존 `@nestjs/graphql`은 GraphQL schema 파일을 찾을 때 `fast-glob`을 사용했다.

기존 `packages/graphql/package.json`:

```json
{
  "dependencies": {
    "fast-glob": "3.3.3"
  }
}
```

### 3.2 `GraphQLTypesLoader`

핵심 파일:

- `packages/graphql/lib/graphql-types.loader.ts`

기존 코드는 대략 이렇게 생겼다:

```ts
import glob from 'fast-glob';

const filePaths = await glob(paths, {
  ignore: includeNodeModules ? [] : ['node_modules'],
});
```

이 흐름을 풀면 다음과 같다.

1. 사용자가 `typePaths`를 넘긴다.
2. Nest GraphQL이 `GraphQLTypesLoader.mergeTypesByPaths()`를 호출한다.
3. 내부에서 glob으로 실제 `.graphql` 파일 목록을 찾는다.
4. 찾은 파일들을 `readFile()`로 읽는다.
5. 읽은 SDL 문자열들을 `mergeTypeDefs()`로 합친다.

코드 흐름:

```ts
async mergeTypesByPaths(paths: string | string[]): Promise<string> {
  if (!paths || paths.length === 0) {
    return null;
  }
  const types = await this.getTypesFromPaths(paths);
  const flatTypes = flatten(types);

  return mergeTypeDefs(flatTypes, {
    throwOnConflict: true,
    commentDescriptions: true,
    reverseDirectives: true,
  });
}
```

여기서 중요한 부분은 `getTypesFromPaths()`다.

```ts
private async getTypesFromPaths(paths: string | string[]): Promise<string[]> {
  const includeNodeModules = this.includeNodeModules(paths);

  paths = Array.isArray(paths)
    ? paths.map((path) => normalize(path))
    : normalize(paths);

  const filePaths = await glob(paths, {
    ignore: includeNodeModules ? [] : ['node_modules'],
  });

  if (filePaths.length === 0) {
    throw new Error(
      `No type definitions were found with the specified file name patterns: "${paths}". Please make sure there is at least one file that matches the given patterns.`,
    );
  }

  const fileContentsPromises = filePaths.sort().map((filePath) => {
    return readFile(filePath.toString(), 'utf8');
  });

  return Promise.all(fileContentsPromises);
}
```

즉 `fast-glob` 결과는 바로 `readFile()`에 들어간다.

그래서 glob 결과가 absolute path인지 relative path인지가 중요하다.  
결과 경로가 바뀌면 파일을 읽는 위치도 바뀔 수 있다.

### 3.3 `GraphQLDefinitionsFactory`

또 다른 사용처:

- `packages/graphql/lib/graphql-definitions.factory.ts`

이 파일은 GraphQL schema를 기반으로 TypeScript definitions를 생성하는 역할을 한다.

`watch: true`일 때는 `.graphql` 파일 변경을 감시한다.

기존 코드:

```ts
const watcher = chokidar.watch(await glob(options.typePaths));
```

흐름:

```txt
options.typePaths
  -> fast-glob으로 실제 파일 경로 찾기
  -> chokidar.watch(filePaths)
  -> 파일 변경 시 definitions 재생성
```

여기도 glob 결과가 중요하다.  
잘못된 경로가 `chokidar.watch()`에 들어가면 watch 대상이 달라질 수 있다.

---

## 4. 왜 단순 교체가 위험했나

처음 보면 그냥 import만 바꾸면 될 것처럼 보인다.

기존:

```ts
import glob from 'fast-glob';
```

변경:

```ts
import { glob } from 'tinyglobby';
```

하지만 실제로 비교해보면 `fast-glob`과 `tinyglobby`는 기본 동작이 완전히 같지 않다.

이번 작업에서 신경 쓴 차이는 크게 다섯 가지다.

1. absolute path 결과
2. relative path 결과
3. directory pattern 자동 확장
4. negative pattern
5. repeated pattern dedupe

### 4.1 absolute path와 relative path

경로에는 두 종류가 있다.

relative path:

```txt
./src/schema.graphql
src/schema.graphql
```

현재 작업 디렉토리를 기준으로 한 경로다.

absolute path:

```txt
/Users/dustin/project/src/schema.graphql
```

파일 시스템 루트부터 시작하는 전체 경로다.

기존 `fast-glob`은 absolute pattern을 넣으면 결과도 absolute path로 돌려준다.

입력:

```txt
/tmp/app/schema/*.graphql
```

결과:

```txt
/tmp/app/schema/user.graphql
```

반대로 relative pattern을 넣으면 relative path를 돌려준다.

입력:

```txt
src/**/*.graphql
```

결과:

```txt
src/users/user.graphql
```

이 동작은 Nest GraphQL 입장에서 중요하다.  
왜냐하면 `GraphQLTypesLoader`는 glob 결과를 그대로 `readFile()`에 넘기기 때문이다.

```ts
readFile(filePath.toString(), 'utf8');
```

### 4.2 `tinyglobby`의 `absolute` 옵션

`tinyglobby`는 `absolute` 옵션을 가진다.

```ts
glob(patterns, {
  absolute: true,
});
```

`absolute: true`면 absolute path를 반환한다.  
`absolute: false`면 relative path를 반환한다.

문제는 이 옵션이 호출 단위로 적용된다는 점이다.

사용자가 이런 식으로 섞어서 넣을 수 있다:

```ts
typePaths: [
  './src/**/*.graphql',
  '/absolute/shared/**/*.graphql',
]
```

이 경우 첫 번째 pattern은 relative 결과가 맞고, 두 번째 pattern은 absolute 결과가 맞다.

그런데 `tinyglobby`를 한 번만 호출하면서 `absolute: true`를 주면 둘 다 absolute가 된다.  
반대로 `absolute: false`를 주면 둘 다 relative가 된다.

기존 `fast-glob`과 같은 결과를 내려면 pattern별로 absolute 여부를 판단해야 한다.

### 4.3 directory pattern 자동 확장

기존 `fast-glob`은 그냥 directory path만 넣으면 파일을 찾지 않는다.

예:

```txt
/tmp/schema
```

결과:

```ts
[]
```

반면 `tinyglobby`는 기본값으로 directory pattern을 자동 확장한다.

예를 들어 `schema/` 폴더 안에 파일이 있으면:

```txt
schema/
  user.graphql
```

`tinyglobby('schema')`가 내부 파일을 찾는 식으로 동작할 수 있다.

이건 기능처럼 보일 수 있지만, 이번 PR에서는 위험하다.

왜냐하면 이번 PR의 목적은 다음이 아니기 때문이다:

```txt
typePaths에 directory만 넣어도 자동으로 파일을 찾게 하자
```

이번 PR의 목적은 이것이다:

```txt
기존 glob 라이브러리만 교체하자
```

내부 의존성이 바뀌었다고 사용자의 설정 의미까지 바뀌면 안 된다.

그래서 `expandDirectories: false`를 명시했다.

```ts
const globOptions = {
  ...options,
  expandDirectories: false,
};
```

### 4.4 negative pattern

glob pattern에는 제외 패턴이 있다. 보통 `!`로 시작한다.

예:

```ts
[
  'src/**/*.graphql',
  '!src/**/*.generated.graphql',
]
```

뜻:

```txt
src 아래 graphql 파일을 찾되,
generated.graphql 파일은 제외하라
```

중간 검토 단계에서 "pattern별로 glob을 돌리자"고 바꿨을 때 문제가 생길 수 있었다.

잘못된 방식:

```ts
patternsArray.map((pattern) => glob(pattern, ...))
```

이렇게 하면 `!src/**/*.generated.graphql`도 하나의 독립 pattern처럼 처리된다.  
그 결과 exclude 의미가 깨진다.

그래서 최종 helper에서는 pattern을 두 그룹으로 나눈다.

```ts
const negativePatterns = patternsArray.filter(isNegativePattern);
const positivePatterns = patternsArray.filter(
  (pattern) => !isNegativePattern(pattern),
);
```

그리고 각 positive pattern을 실행할 때 negative pattern을 같이 넘긴다.

```ts
glob([pattern, ...negativePatterns], ...)
```

이렇게 해야 다음 의미가 유지된다:

```txt
이 include pattern으로 찾되, exclude pattern은 빼라
```

### 4.5 repeated pattern dedupe

사용자가 같은 pattern을 두 번 넣을 수도 있다.

```ts
[
  'src/**/*.graphql',
  'src/**/*.graphql',
]
```

기존 `fast-glob`은 결과를 중복으로 돌려주지 않는다.

잘못 구현하면 같은 파일이 두 번 나온다.

```ts
[
  'src/schema.graphql',
  'src/schema.graphql',
]
```

그러면 Nest GraphQL이 같은 schema file을 두 번 읽고 merge할 수 있다.  
경우에 따라 duplicate type definition 문제가 생길 수도 있다.

그래서 최종 helper는 `Set`으로 dedupe한다.

```ts
return [...new Set(matches.flat())];
```

---

## 5. 최종 helper 코드 설명

추가한 파일:

- `packages/graphql/lib/utils/glob.util.ts`

최종 코드:

```ts
import { isAbsolute, win32 } from 'path';
import { glob } from 'tinyglobby';
import type { GlobOptions } from 'tinyglobby';

type GlobPattern = string | string[];

export async function globPaths(
  patterns: GlobPattern,
  options: Omit<
    GlobOptions,
    'absolute' | 'expandDirectories' | 'patterns'
  > = {},
): Promise<string[]> {
  const patternsArray = Array.isArray(patterns) ? patterns : [patterns];
  const negativePatterns = patternsArray.filter(isNegativePattern);
  const positivePatterns = patternsArray.filter(
    (pattern) => !isNegativePattern(pattern),
  );
  const globOptions = {
    ...options,
    expandDirectories: false,
  };

  const matches = await Promise.all(
    positivePatterns.map((pattern) =>
      glob([pattern, ...negativePatterns], {
        ...globOptions,
        absolute: isAbsoluteGlob(pattern),
      }),
    ),
  );

  return [...new Set(matches.flat())];
}

function isNegativePattern(pattern: string): boolean {
  return pattern[0] === '!' && pattern[1] !== '(';
}

function isAbsoluteGlob(pattern: string): boolean {
  return isAbsolute(pattern) || win32.isAbsolute(pattern);
}
```

### 5.1 `GlobPattern`

```ts
type GlobPattern = string | string[];
```

기존 `fast-glob` 호출부는 `string | string[]`를 받을 수 있었다.  
`GraphQLTypesLoader.mergeTypesByPaths()`도 같은 타입을 쓴다.

예:

```ts
typePaths: './src/**/*.graphql'
```

또는:

```ts
typePaths: [
  './src/**/*.graphql',
  './libs/**/*.graphql',
]
```

그래서 helper도 같은 입력 형태를 받는다.

### 5.2 문자열 하나를 배열로 통일

```ts
const patternsArray = Array.isArray(patterns) ? patterns : [patterns];
```

이후 로직은 배열 기준으로 처리한다.  
문자열 하나가 들어오면 배열 하나짜리로 감싼다. 예를 들어:

```ts
'./src/**/*.graphql'
```

이 값을 이렇게 만든다:

```ts
['./src/**/*.graphql']
```

### 5.3 negative pattern 분리

```ts
const negativePatterns = patternsArray.filter(isNegativePattern);
const positivePatterns = patternsArray.filter(
  (pattern) => !isNegativePattern(pattern),
);
```

예:

```ts
[
  'src/**/*.graphql',
  '!src/**/*.generated.graphql',
]
```

이면:

```ts
positivePatterns = ['src/**/*.graphql'];
negativePatterns = ['!src/**/*.generated.graphql'];
```

이렇게 나눈다.

### 5.4 `isNegativePattern()`

```ts
function isNegativePattern(pattern: string): boolean {
  return pattern[0] === '!' && pattern[1] !== '(';
}
```

보통 negative pattern은 `!foo` 형태다.

하지만 glob에는 extglob이라는 문법도 있다.

예:

```txt
!(foo)
```

이건 "negative pattern"이라기보다 glob 문법 안의 extglob 표현이다.  
그래서 `pattern[1] !== '('` 조건을 둬서 `!(...)`를 exclude pattern으로 오해하지 않게 했다.

### 5.5 `expandDirectories: false`

```ts
const globOptions = {
  ...options,
  expandDirectories: false,
};
```

이 부분이 `fast-glob` migration에서 중요하다.

`tinyglobby`는 기본적으로 directory pattern을 자동 확장할 수 있다.  
하지만 기존 `fast-glob`은 directory path만 들어왔을 때 빈 배열을 돌려줬다.

그래서:

```ts
expandDirectories: false
```

를 넣어 기존 동작에 맞춘다.

### 5.6 pattern별 glob 실행

```ts
const matches = await Promise.all(
  positivePatterns.map((pattern) =>
    glob([pattern, ...negativePatterns], {
      ...globOptions,
      absolute: isAbsoluteGlob(pattern),
    }),
  ),
);
```

여기서 핵심은 두 가지다.

첫째, positive pattern마다 glob을 따로 돌린다.

이유:

```ts
[
  './src/**/*.graphql',
  '/absolute/shared/**/*.graphql',
]
```

처럼 relative pattern과 absolute pattern이 섞일 수 있기 때문이다.

`tinyglobby`의 `absolute` 옵션은 호출 전체에 적용된다.  
따라서 한 번에 호출하면 둘 중 하나의 동작을 포기해야 한다.

둘째, 각 positive pattern에 negative pattern을 같이 붙인다.

```ts
glob([pattern, ...negativePatterns], ...)
```

그래야 exclude pattern 의미가 유지된다.

### 5.7 `absolute: isAbsoluteGlob(pattern)`

```ts
absolute: isAbsoluteGlob(pattern),
```

pattern이 absolute path면 결과도 absolute path로 받는다.  
pattern이 relative path면 결과도 relative path로 받는다.

이게 기존 `fast-glob` 동작에 가깝다.

### 5.8 Windows absolute path

```ts
function isAbsoluteGlob(pattern: string): boolean {
  return isAbsolute(pattern) || win32.isAbsolute(pattern);
}
```

`path.isAbsolute()`는 현재 OS 기준으로 동작한다.  
macOS/Linux에서 Windows 경로인 `C:\foo\bar.graphql`을 넣으면 absolute로 못 볼 수 있다.

Nest GraphQL 쪽은 이미 `normalize-path`를 써서 backslash를 slash로 바꾸는 흐름이 있다.  
그래도 Windows drive path 자체를 고려하려고 `win32.isAbsolute()`도 같이 사용했다.

### 5.9 dedupe

```ts
return [...new Set(matches.flat())];
```

각 positive pattern을 따로 돌리면 repeated pattern에서 같은 파일이 여러 번 나올 수 있다.

예:

```ts
[
  'src/**/*.graphql',
  'src/**/*.graphql',
]
```

그래서 마지막에 `Set`으로 중복을 제거한다.

중요한 건 `Set`이 insertion order를 유지한다는 것이다.  
그래서 먼저 발견된 결과 순서를 크게 흔들지 않으면서 중복만 제거한다.

---

## 6. 실제 적용 코드

### 6.1 `GraphQLTypesLoader`

변경 전:

```ts
import glob from 'fast-glob';
```

변경 후:

```ts
import { globPaths } from './utils/glob.util';
```

변경 전:

```ts
const filePaths = await glob(paths, {
  ignore: includeNodeModules ? [] : ['node_modules'],
});
```

변경 후:

```ts
const filePaths = await globPaths(paths, {
  ignore: includeNodeModules ? [] : ['node_modules'],
});
```

이 변경은 public API를 바꾸지 않는다.

사용자는 여전히 이렇게 쓴다:

```ts
GraphQLModule.forRoot({
  typePaths: ['./src/**/*.graphql'],
});
```

바뀐 건 내부에서 파일을 찾는 라이브러리뿐이다.

### 6.2 `GraphQLDefinitionsFactory`

변경 전:

```ts
import { glob } from 'fast-glob';
```

변경 후:

```ts
import { globPaths } from './utils/glob.util';
```

변경 전:

```ts
const watcher = chokidar.watch(await glob(options.typePaths));
```

변경 후:

```ts
const watcher = chokidar.watch(await globPaths(options.typePaths));
```

이 코드는 `watch: true`일 때 사용된다.

흐름:

```txt
typePaths
  -> globPaths()
  -> 실제 .graphql 파일 경로 목록
  -> chokidar.watch()
  -> 파일 변경 감지
  -> definitions 재생성
```

### 6.3 왜 `utils/index.ts`에 export하지 않았나

`globPaths()`는 내부 구현 detail이다.

그래서 `packages/graphql/lib/utils/index.ts` 같은 barrel export에 넣지 않았다.  
괜히 export하면 public처럼 보이는 surface가 늘어난다.

이번 PR의 목표는 API 추가가 아니다.

목표는:

```txt
내부 glob 라이브러리 교체
```

이다.

그래서 helper는 직접 import로만 사용했다.

```ts
import { globPaths } from './utils/glob.util';
```

---

## 7. 테스트 설명

이번 작업에서 테스트는 크게 세 가지로 나눠서 작성했다.

1. helper 단위 테스트
2. 실제 schema file loading 테스트
3. definitions watch mode 테스트

### 7.1 helper 테스트

파일:

- `packages/graphql/tests/utils/glob.util.spec.ts`

#### absolute pattern

```ts
it('preserves absolute matches for absolute patterns', async () => {
  await expect(globPaths(join(directory, '*.graphql'))).resolves.toEqual([
    join(directory, 'schema.graphql'),
  ]);
});
```

검증하는 것:

```txt
absolute pattern을 넣으면 absolute path가 나온다.
```

#### relative pattern

```ts
it('preserves relative matches for relative patterns', async () => {
  const relativePattern = relative(
    process.cwd(),
    join(directory, '*.graphql'),
  );
  const relativeSchemaPath = relative(
    process.cwd(),
    join(directory, 'schema.graphql'),
  );

  await expect(globPaths(relativePattern)).resolves.toEqual([
    relativeSchemaPath,
  ]);
});
```

검증하는 것:

```txt
relative pattern을 넣으면 relative path가 나온다.
```

#### mixed absolute / relative pattern order

```ts
it('preserves pattern order when absolute and relative patterns are mixed', async () => {
  const relativePattern = relative(
    process.cwd(),
    join(directory, '*.graphql'),
  );
  const relativeSchemaPath = relative(
    process.cwd(),
    join(directory, 'schema.graphql'),
  );
  const absolutePattern = join(absoluteDirectory, '*.graphql');
  const absoluteSchemaPath = join(absoluteDirectory, 'schema.graphql');

  await expect(
    globPaths([relativePattern, absolutePattern]),
  ).resolves.toEqual([relativeSchemaPath, absoluteSchemaPath]);
});
```

검증하는 것:

```txt
relative pattern이 먼저 들어오고 absolute pattern이 나중에 들어오면,
결과도 그 순서를 따른다.
```

처음에는 absolute pattern끼리, relative pattern끼리 묶어서 처리했었다.  
그 방식은 결과 순서가 바뀔 수 있어서 pattern별 처리로 바꿨다.

#### negative pattern

```ts
it('applies negative patterns', async () => {
  await writeFile(
    join(directory, 'skip.graphql'),
    'type Query { skip: Boolean }',
  );
  const relativePattern = relative(
    process.cwd(),
    join(directory, '*.graphql'),
  );
  const relativeSchemaPath = relative(
    process.cwd(),
    join(directory, 'schema.graphql'),
  );
  const relativeSkipPath = relative(
    process.cwd(),
    join(directory, 'skip.graphql'),
  );

  await expect(
    globPaths([relativePattern, `!${relativeSkipPath}`]),
  ).resolves.toEqual([relativeSchemaPath]);
});
```

검증하는 것:

```txt
include pattern으로 파일을 찾되,
!exclude pattern에 걸리는 파일은 빠진다.
```

#### repeated pattern dedupe

```ts
it('deduplicates matches from repeated patterns', async () => {
  const relativePattern = relative(
    process.cwd(),
    join(directory, '*.graphql'),
  );
  const relativeSchemaPath = relative(
    process.cwd(),
    join(directory, 'schema.graphql'),
  );

  await expect(
    globPaths([relativePattern, relativePattern]),
  ).resolves.toEqual([relativeSchemaPath]);
});
```

검증하는 것:

```txt
같은 pattern이 두 번 들어와도 같은 파일을 두 번 돌려주지 않는다.
```

#### directory pattern

```ts
it('does not expand directory patterns', async () => {
  await expect(globPaths(directory)).resolves.toEqual([]);
});
```

검증하는 것:

```txt
directory path만 넣었을 때 내부 파일을 자동으로 찾지 않는다.
```

이 테스트가 `expandDirectories: false`를 고정한다.

### 7.2 실제 loader 테스트

파일:

- `packages/graphql/tests/graphql-types.loader.spec.ts`

테스트 코드:

```ts
it('loads type definitions from glob patterns', async () => {
  await writeFile(
    join(directory, 'cat.graphql'),
    'type Cat { id: ID!, name: String! }',
  );
  await writeFile(
    join(directory, 'query.graphql'),
    'type Query { cat: Cat }',
  );

  const loader = new GraphQLTypesLoader();
  const typeDefs = await loader.mergeTypesByPaths(
    join(directory, '*.graphql'),
  );

  expect(typeDefs).toContain('type Query');
  expect(typeDefs).toContain('type Cat');
});
```

이 테스트는 helper만 보는 게 아니다.  
실제 `GraphQLTypesLoader`가 `.graphql` 파일을 찾고 읽고 merge하는 흐름을 검증한다.

흐름:

```txt
임시 폴더 생성
  -> cat.graphql 생성
  -> query.graphql 생성
  -> mergeTypesByPaths("*.graphql")
  -> globPaths()
  -> readFile()
  -> mergeTypeDefs()
  -> 결과 문자열에 type Query / type Cat 포함 확인
```

### 7.3 definitions watch mode 테스트

파일:

- `packages/graphql/tests/graphql-definitions.factory.spec.ts`

테스트 코드:

```ts
it('watches files resolved from type path glob patterns', async () => {
  await writeFile(
    join(directory, 'schema.graphql'),
    'type Query { ok: Boolean }',
  );

  const factory = new GraphQLDefinitionsFactory();
  await factory.generate({
    typePaths: [join(directory, '*.graphql')],
    path: join(directory, 'graphql.ts'),
    outputAs: 'class',
    watch: true,
    debug: false,
  });

  expect(chokidar.watch).toHaveBeenCalledWith([
    join(directory, 'schema.graphql'),
  ]);
});
```

이 테스트는 `watch: true`일 때 `chokidar.watch()`가 glob pattern 원본이 아니라 실제 파일 경로 목록을 받는지 확인한다.

즉:

```txt
typePaths: ["/tmp/.../*.graphql"]
```

가:

```txt
chokidar.watch(["/tmp/.../schema.graphql"])
```

로 이어지는지 확인한다.

이 테스트가 필요한 이유는 이슈 코멘트에서 내가 watch-mode behavior도 보겠다고 했기 때문이다.  
helper 테스트만 있으면 "watch path도 실제로 바뀌었나?"가 약할 수 있어서, definitions factory 경로까지 얇게 덮었다.

---

## 8. 직접 비교했던 `fast-glob` vs `tinyglobby`

작업 중 임시 프로젝트를 만들어 `fast-glob@3.3.3`과 `tinyglobby@0.2.17` 결과를 직접 비교했다.

대표 케이스:

```ts
[
  'a/*.graphql',
  '!a/skip.graphql',
]
```

기존 `fast-glob`:

```ts
['a/keep.graphql']
```

잘못된 pattern별 tinyglobby 호출:

```ts
['a/keep.graphql', 'a/skip.graphql']
```

문제:

```txt
!a/skip.graphql 제외 패턴이 깨진다.
```

그래서 negative pattern을 분리하고, 각 positive pattern 호출에 같이 넘기는 방식으로 수정했다.

또 다른 케이스:

```ts
[
  'a/*.graphql',
  'a/*.graphql',
]
```

기존 `fast-glob`:

```ts
['a/keep.graphql', 'a/skip.graphql']
```

잘못된 pattern별 tinyglobby 호출:

```ts
[
  'a/keep.graphql',
  'a/skip.graphql',
  'a/keep.graphql',
  'a/skip.graphql',
]
```

문제:

```txt
같은 파일이 중복으로 나온다.
```

그래서 최종 결과를 `Set`으로 dedupe했다.

absolute/relative mixed case도 비교했다.

```ts
[
  '/tmp/.../a/*.graphql',
  'b/*.graphql',
]
```

기존 `fast-glob`은 absolute pattern 결과는 absolute로, relative pattern 결과는 relative로 돌려줬다.

최종 helper도 같은 방식으로 맞췄다.

---

## 9. 검증 결과

### 9.1 실행한 명령

```bash
yarn lint
```

결과:

```txt
pass
```

```bash
yarn test:e2e tests/graphql-definitions.factory.spec.ts tests/graphql-types.loader.spec.ts tests/utils/glob.util.spec.ts
```

결과:

```txt
Test Files  3 passed (3)
Tests       8 passed (8)
```

```bash
yarn test:e2e:graphql
```

결과:

```txt
Test Files  44 passed (44)
Tests       230 passed (230)
```

```bash
git diff --check
```

결과:

```txt
pass
```

### 9.2 build 확인

다음 명령도 확인했다.

```bash
./node_modules/.bin/tsc -p packages/graphql/tsconfig.build.json --pretty false --noEmit
```

결과는 실패했다.

에러:

```txt
packages/graphql/lib/services/gql-subscription.service.ts(11,27): error TS2307: Cannot find module 'graphql-ws/use/ws' or its corresponding type declarations.
```

이 파일은 이번 PR에서 건드린 파일이 아니다.  
기존 `graphql-ws/use/ws` module resolution 문제로 보인다.

이번 glob helper 쪽 타입 에러는 보이지 않았다.

---

## 10. PR 본문

최종 PR 본문은 일부러 짧게 작성했다.

너무 길게 쓰면 오히려 과장돼 보일 수 있다.  
maintainer 입장에서는 "무엇을 바꿨는지", "왜 helper가 있는지", "테스트는 뭘 돌렸는지"만 빠르게 보면 된다.

사용한 본문 핵심:

```md
Issue Number: #3731

`@nestjs/graphql` uses `fast-glob` for GraphQL type path lookups.

Uses `tinyglobby` instead.

There are a few small differences between the two packages, so I added a small helper to keep `typePaths` working the same way as before.

Ran `yarn lint` and `yarn test:e2e:graphql`.
```

이 문장은 짧지만 중요한 걸 담고 있다.

```txt
두 라이브러리가 완전히 같지 않다.
그래서 helper를 뒀다.
목표는 typePaths 동작 유지다.
```

---

## 11. 이번 작업에서 배운 점

### 11.1 의존성 교체는 단순 import 변경이 아니다

두 라이브러리가 같은 카테고리에 있어도 기본 동작은 다를 수 있다.

이번 작업도 처음에는 단순해 보였다.

```txt
fast-glob 제거
tinyglobby 추가
import 교체
```

하지만 실제로는 다음 차이를 봐야 했다.

- absolute path 반환 방식
- relative path 반환 방식
- directory pattern 확장 여부
- negative pattern 적용
- repeated pattern dedupe
- mixed pattern 순서

이런 차이를 무시하면 "테스트는 통과했는데 일부 사용자 설정이 깨지는 PR"이 된다.

### 11.2 내부 모듈이 바뀌어도 사용자는 바뀌면 안 된다

이번 PR의 철학은 이거다.

```txt
내부 의존성은 바뀐다.
외부 사용자 계약은 유지한다.
```

Nest GraphQL 사용자는 `fast-glob`을 직접 쓰는 게 아니다.  
사용자는 그냥 `typePaths`를 쓴다.

```ts
typePaths: ['./src/**/*.graphql']
```

그러면 내부에서 `fast-glob`을 쓰든 `tinyglobby`를 쓰든 사용자가 느끼는 동작은 같아야 한다.

이게 framework code에서 중요한 기준이다.

### 11.3 helper는 추상화를 위한 추상화가 아니어야 한다

이번 `globPaths()`는 추상화를 만들고 싶어서 만든 코드가 아니다.

목적이 명확하다:

```txt
tinyglobby의 기본 동작 차이를 한곳에 가둔다.
```

호출부 두 곳에서 같은 compatibility 처리를 반복하지 않고, 내부 helper 하나에 모았다.

또 public export도 하지 않았다.

이렇게 해야 helper가 "새 API"가 아니라 "내부 구현 detail"로 남는다.

### 11.4 테스트는 내가 걱정한 지점을 직접 고정해야 한다

이번 테스트는 단순히 "파일 하나 찾음"만 보지 않았다.

실제로 걱정한 지점들을 테스트로 박았다.

- absolute path
- relative path
- mixed pattern order
- negative pattern
- duplicate pattern
- directory pattern
- actual loader flow
- watch mode flow

이런 테스트는 PR 설명보다 강하다.  
리뷰어가 helper가 왜 필요한지 의심해도, 테스트 이름만 보면 의도가 보인다.

---

## 12. 머지 결과

2026-08-17 08:38:33 UTC에 Kamil이 `master`를 `fix-3731-switch-to-tinyglobby` 브랜치에 병합하는 커밋 `1d9e7ad`를 추가했다.  
이후 `build-and-test` check가 성공했고, 2026-08-17 09:47:14 UTC에 PR #4078이 merge commit `7687fc6`으로 `nestjs:master`에 병합됐다.

GitHub PR 상태는 최종적으로 `MERGED`다. 별도 review record는 없었고, maintainer의 추가 설계 변경 요청 없이 충돌 해소 후 바로 머지됐다.

관련 이슈 #3731은 2026-07-27 08:26:55 UTC에 이미 `Closed` 상태가 됐다. `gh pr view` 기준 `closingIssuesReferences`는 비어 있었으므로, PR merge가 closing keyword로 이슈를 자동 종료한 형태는 아니었다.

## 13. 체크리스트 / 최종 상태

- [x] 이슈 코멘트 작성
- [x] maintainer 확인 (`sounds good`)
- [x] fork branch 생성
- [x] 코드 수정
- [x] helper compatibility 테스트 추가
- [x] loader 실제 흐름 테스트 추가
- [x] definitions watch mode 테스트 추가
- [x] lint 통과
- [x] GraphQL e2e suite 통과
- [x] 커밋
- [x] push
- [x] PR 생성
- [x] master가 앞서나가면서 생긴 충돌 해소 (2026-08-06, master 위로 리베이스)
- [x] CI 결과 확인 (`build-and-test` 성공)
- [x] maintainer 리뷰 반응 확인 (별도 review 없이 Kamil이 직접 merge)
- [x] 머지 여부 및 소요 시간 기록 (2026-07-27 05:40:31 UTC 생성 → 2026-08-17 09:47:14 UTC 머지)
