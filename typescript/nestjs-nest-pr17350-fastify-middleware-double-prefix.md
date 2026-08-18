# [Nest OSS 기여] nestjs/nest - Fastify plugin prefix 환경에서 Nest middleware 경로가 두 번 prefix 되는 버그 수정 (PR #17350) — 🟢 Merged

> 오픈소스 기여 기록 및 학습 노트. 이 문서는 PR #17350 하나만 다룬다.  
> **2026-08-11 `kamilmysliwiec`가 머지.** 2026-07-21에 이 버그가 vendored 파일의 upstream 동기화 누락 때문이라는 배경을 확인해 코멘트로 남김 — 8.3절 참고.
>
> Nest가 Fastify 위에서 어떻게 동작하는지, middleware와 plugin scope가 왜 엮이는지, 그리고 왜 이 버그가 `instance.use()` 재호출 한 줄에서 생겼는지까지 코드 기준으로 정리한다.

---

## 1. 개요

### 1.1 프로젝트 소개

- **저장소**: https://github.com/nestjs/nest
- **작업 위치**: `/Users/dustin/Desktop/common/open-source/nestjs-nest`
- **작업 브랜치**: `master`
- **관련 PR**: https://github.com/nestjs/nest/pull/17350
- **이슈**: https://github.com/nestjs/nest/issues/11802
- **대상 패키지**: `packages/platform-fastify`

### 1.2 한 줄 요약

> Nest의 Fastify 어댑터가 부모 scope의 middleware를 자식 Fastify plugin으로 넘길 때, 이미 prefix가 반영된 middleware를 다시 `instance.use()`로 등록해서 경로가 `/queues/queues`처럼 두 번 prefix 되는 버그를 수정했다.

### 1.3 이 문서의 핵심

이 버그는 middleware가 안 도는 것보다 좀 더 근본적인 문제였다.

- Nest는 애플리케이션 구조를 담당한다.
- Fastify는 HTTP 요청 처리와 plugin scope를 담당한다.
- Nest Fastify 어댑터는 둘 사이를 이어준다.
- 그 연결부에서 middleware를 자식 plugin으로 복사하는 방식이 잘못돼 경로가 한 번 더 계산됐다.

즉, **Nest 내부 로직만 보면 맞아 보이는데, Fastify plugin scope와 만나는 지점에서 깨진 버그**다.

---

## 2. 먼저 용어 정리

이 부분은 대화에서 가장 많이 반복한 내용이라, 문서에도 남겨둔다.

### 2.1 Nest는 무엇인가

Nest는 Node.js 백엔드 프레임워크다.  
핵심 역할은 HTTP 요청을 "직접" 처리하는 것보다, 서버 앱의 구조를 정리하는 것이다.

Nest가 제공하는 것:

- `Module`
- `Controller`
- `Service`
- `Middleware`
- `Guard`
- `Interceptor`
- `Pipe`
- `Filter`

즉 Nest는 "요청을 받아서 처리하는 코드" 그 자체보다, **그 코드를 어떻게 나눠서 관리할지**를 정해준다.

### 2.2 Express / Fastify는 무엇인가

Express와 Fastify도 프레임워크다.  
다만 Nest 입장에서는 이 둘이 **HTTP 요청을 실제로 받는 하위 플랫폼**처럼 동작한다.

- Express: 전통적으로 널리 쓰이는 HTTP 서버 프레임워크
- Fastify: 더 엄격한 plugin 구조와 빠른 라우팅을 가진 HTTP 서버 프레임워크

Nest는 이 둘 위에 올라간다.

```txt
브라우저 요청
  -> Node.js
  -> Express 또는 Fastify
  -> Nest 어댑터
  -> Nest middleware / guard / controller
```

즉, Nest가 혼자 모든 HTTP 처리를 새로 구현하는 구조가 아니다.

### 2.3 middleware는 무엇인가

middleware는 요청이 controller에 도달하기 전에 중간에서 끼어드는 함수다.

예:

- 로그 찍기
- 인증 확인
- 헤더 검사
- 요청 차단

흐름은 보통 이렇다.

```txt
request -> middleware -> controller -> response
```

Nest에서는 middleware를 자주 쓰지만, 실무에선 Guard / Interceptor / Pipe가 더 자주 보인다.  
그래도 middleware는 여전히 중요한 도구고, Fastify 어댑터에서는 plugin scope와 얽히면서 예민한 부분이 된다.

### 2.4 Fastify plugin scope와 자식 plugin

Fastify는 `register(plugin, { prefix })`를 쓰면 plugin마다 scope를 나눈다.

이때 새로 등록된 plugin은 부모 scope와 연결되지만, 동시에 **자기 prefix와 자기 hook 체인**을 가진다.  
이걸 흔히 자식 plugin, child scope라고 부른다.

예를 들면:

```ts
app.register(async (instance) => {
  instance.register(async (child) => {
    // child scope
  }, { prefix: '/queues' });
});
```

이 구조에서는 middleware도 단순히 "한 번 등록하면 끝"이 아니다.  
부모에 등록된 middleware를 자식 scope에서도 계속 써야 하므로, Fastify 어댑터가 그 middleware를 자식 instance로 복사해줘야 한다.

---

## 3. 이 버그는 왜 생겼나

### 3.1 문제 상황

이슈 #11802의 상황을 아주 단순하게 쓰면 이렇다.

1. Nest middleware를 `/queues`에 붙인다.
2. Fastify plugin도 `prefix: '/queues'`로 등록한다.
3. Fastify 어댑터가 부모 middleware를 자식 plugin으로 넘긴다.
4. 이때 공개용 `instance.use()`를 다시 호출한다.
5. `instance.use()`는 현재 scope의 prefix를 또 붙인다.
6. 결과적으로 경로가 `/queues/queues`처럼 두 번 prefix 된다.

즉, 원래는 한 번만 계산돼야 할 경로가 자식 plugin에서 다시 계산됐다.

### 3.2 Nest 개발자 시점에서 보면

Nest 쪽 코드만 보면 그냥 이거다.

```ts
consumer.apply(CountingMiddleware).forRoutes('/queues');
```

"이 middleware는 `/queues`에 붙어야 한다"는 뜻이다.  
그 자체는 맞다.

문제는 Nest가 Fastify 어댑터를 통해 그 설정을 Fastify plugin scope로 옮기는 과정에서 생겼다.

### 3.3 Fastify가 제멋대로 prefix를 붙인 게 아니다

Fastify는 원래 plugin scope를 기준으로 경로를 계산한다.

- 부모 plugin
- 자식 plugin
- 각 scope의 prefix

이 구조가 있기 때문에, Fastify는 `use()` 같은 공개 API에서 현재 `this.prefix`를 반영한다.  
그건 Fastify의 정상 동작이다.

문제는 **이미 prefix가 반영된 middleware를 다시 "새 middleware 등록"처럼 넣은 것**이다.

즉:

- Fastify가 잘못한 게 아니라
- Nest Fastify 어댑터가 그 API를 잘못 쓴 것이다

---

## 4. 실제 코드에서 무슨 일이 벌어졌나

문제의 파일은 여기다.

- [`packages/platform-fastify/adapters/middie/fastify-middie.ts`](file:///Users/dustin/Desktop/common/open-source/nestjs-nest/packages/platform-fastify/adapters/middie/fastify-middie.ts)

### 4.1 `use()`는 무엇을 하는가

이 함수는 middleware를 등록하는 공개 API다.

```ts
function use(this: FastifyInstance, path: string | null, fn?: Function) {
  if (typeof path === 'string') {
    const prefix = this.prefix;
    path = prefix + (path === '/' && prefix.length > 0 ? '' : path);
  }

  this[kMiddlewares].push([path, fn]);

  if (fn == null) {
    this[kMiddie].use(path);
  } else {
    this[kMiddie].use(path, fn);
  }

  this[kMiddieHasMiddlewares] = true;
  return this;
}
```

핵심은 이 부분이다.

```ts
const prefix = this.prefix;
path = prefix + ...
```

즉 `use('/queues', fn)`을 호출하면, 현재 instance의 prefix를 다시 붙인다.

이건 **새 middleware를 현재 scope 기준으로 등록할 때는 맞는 동작**이다.  
문제는 이 함수를 **상속된 middleware를 자식 plugin으로 복사하는 과정**에서도 다시 써버렸다는 점이다.

### 4.2 `onRegister()`는 무엇을 하는가

Fastify가 새 plugin을 만들면 `onRegister()`가 호출된다.

```ts
function onRegister(instance: FastifyInstance) {
  const middlewares = instance[kMiddlewares].slice() as Array<Array<unknown>>;
  instance[kMiddlewares] = [];
  instance[kMiddie] = middie(onMiddieEnd, instance.initialConfig);
  instance[kMiddieHasMiddlewares] = false;
  instance.decorate('use', use as any);
  for (const middleware of middlewares) {
    (instance.use as any)(...middleware);
  }
}
```

이 함수의 목적은:

- 부모 scope의 middleware를 꺼내서
- 자식 scope에 다시 심고
- nested plugin에서도 계속 동작하게 하는 것

문제는 마지막 줄이다.

```ts
(instance.use as any)(...middleware);
```

이건 공개 `use()` 래퍼를 다시 타는 것이다.  
즉 prefix를 다시 계산한다.

### 4.3 왜 이게 버그였나

부모 scope에서 이미 저장된 middleware는 path가 사실상 완성된 상태다.  
그걸 자식 scope에서 다시 `use()`로 넣으면:

- 부모에서 `/queues`
- 자식에서도 prefix `/queues`
- 다시 `/queues`가 붙음

결과:

```txt
/queues -> /queues/queues
```

이게 핵심 버그다.

---

## 5. 어떻게 고쳤나

수정 방향은 단순하다.

**상속된 middleware를 자식 scope에 넘길 때 공개 `use()`를 다시 타지 않는다.**

### 5.1 변경 전

```ts
for (const middleware of middlewares) {
  (instance.use as any)(...middleware);
}
```

### 5.2 변경 후

```ts
for (const middleware of middlewares) {
  instance[kMiddlewares].push(middleware as any);
  (instance[kMiddie].use as any)(...middleware);
}
```

### 5.3 이 변경의 의미

#### `instance[kMiddie].use(...)`

이건 내부 middie 엔진에 middleware를 직접 넣는 것이다.

- 공개 `use()` 래퍼를 안 탄다
- 현재 `prefix`를 다시 계산하지 않는다
- 이미 계산된 path를 그대로 쓴다

즉 prefix 중복을 막는다.

#### `instance[kMiddlewares].push(...)`

이건 자식 instance의 middleware 목록에도 다시 넣는 것이다.

왜 필요하냐면, Fastify plugin은 더 깊은 nested plugin을 가질 수 있기 때문이다.

예:

- 부모 -> child
- child -> grandchild

child에 middleware를 넘겼는데 `kMiddlewares`에 안 넣어두면 grandchild로 다시 물려줄 수 없다.  
그래서 저장소에도 다시 넣는다.

#### `instance[kMiddieHasMiddlewares] = middlewares.length > 0`

이 플래그는 요청 처리 시 middleware가 있는지 빠르게 확인하기 위한 것이다.

즉 이 수정은 단순히 한 줄만 바꾼 게 아니라:

- prefix 재계산은 막고
- nested plugin 상속은 유지하고
- 실행 플래그도 맞춰준 것이다

---

## 6. 테스트는 어떻게 검증했나

이번 건은 유닛 테스트만으로는 부족해서, 실제 e2e 재현을 같이 넣었다.

### 6.1 보조 유닛 테스트

파일:

- [`packages/platform-fastify/test/adapters/middie.spec.ts`](file:///Users/dustin/Desktop/common/open-source/nestjs-nest/packages/platform-fastify/test/adapters/middie.spec.ts)

이 테스트는 Fastify 내부의 plugin 상속을 수동으로 모킹해서:

- child scope
- grandchild scope

까지 middleware가 유지되는지 확인한다.

장점:

- onRegister 동작과 `kMiddlewares` 복사 로직을 직접 검증할 수 있다

한계:

- Fastify 실제 plugin lifecycle 전체를 돌리지는 않는다

그래서 보조 테스트로 두는 게 맞다.

### 6.2 핵심 e2e 재현

파일:

- [`integration/hello-world/e2e/repro-11802.spec.ts`](file:///Users/dustin/Desktop/common/open-source/nestjs-nest/integration/hello-world/e2e/repro-11802.spec.ts)

이 테스트는 실제로:

- `NestFastifyApplication`
- `FastifyAdapter`
- `MiddlewareConsumer.apply(...).forRoutes('/queues')`
- `app.init()` 이후 Fastify plugin 등록
- `app.getHttpAdapter().getInstance().register(thirdPartyPlugin, { prefix: '/queues' })`

를 그대로 사용한다.

중요한 점은 `thirdPartyPlugin`이 `fastify-plugin`으로 감싼 플러그인이 아니라는 것이다.  
이건 의도적이다.

왜냐하면 `fastify-plugin`으로 감싸면 encapsulation이 깨질 수 있고, 이번 버그는 바로 **plugin scope가 실제로 생기는 상황**에서 봐야 하기 때문이다.

테스트에서 확인하는 것:

- plugin 안의 route가 실제로 응답하는지
- middleware가 정확히 1번 실행되는지
- 일반 Nest route도 같은 앱에서 여전히 정상인지

### 6.3 실제 실행 결과

검증 시 다음을 돌렸다.

```bash
node --loader ts-node/esm ./node_modules/mocha/bin/mocha.js integration/hello-world/e2e/repro-11802.spec.ts
node --loader ts-node/esm ./node_modules/mocha/bin/mocha.js packages/platform-fastify/test/**/*.spec.ts
./node_modules/.bin/eslint integration/hello-world/e2e/repro-11802.spec.ts
```

결과:

- e2e 통과
- 보조 유닛 테스트 통과
- ESLint 통과

---

## 7. 내가 이 작업에서 이해한 핵심

### 7.1 Nest와 Fastify의 관계

1.3절에서 설명한 관계(Nest가 구조를 잡고, Fastify가 요청을 처리하고, 어댑터가 둘을 잇는다) 그대로다.

### 7.2 이 버그의 진짜 본질

문제는 "middleware를 붙였다"가 아니라:

- Nest가 정한 middleware path
- Fastify plugin prefix
- 그리고 어댑터의 상속 로직

이 셋이 만나는 지점에서 path가 다시 계산된 것이다.

### 7.3 왜 `instance.use()`를 다시 타면 안 됐는가

`instance.use()`는 새 middleware 등록 API다.  
그래서 현재 scope prefix를 다시 붙인다.

하지만 `onRegister()`에서 하고 싶은 일은 "새로 등록"이 아니라 "이미 있던 middleware를 자식 scope로 그대로 물려주기"다.  
그 작업을 공개 API로 처리하면 prefix가 두 번 붙는다.

---

## 8. 결과

### 8.1 수정 결과

- Fastify plugin prefix 환경에서 middleware path 중복 계산 문제를 해결
- nested plugin scope에서도 middleware 상속 유지
- 실제 e2e 재현과 unit test 모두 추가

### 8.2 PR 상태

- PR: https://github.com/nestjs/nest/pull/17350 — 🟢 Merged (2026-08-11, `kamilmysliwiec`)
- 이슈: https://github.com/nestjs/nest/issues/11802
- 별도 리뷰 코멘트 없이 머지됨 — 8.3절의 vendoring 배경 코멘트가 유일한 논의였음

### 8.3 이 버그의 배경 — vendoring이 upstream 수정을 못 따라간 것 (2026-07-21)

이 버그를 고친 파일(`fastify-middie.ts`)은 원래 외부 의존성 `@fastify/middie`를 그대로 쓰던 게 아니라, 2025-12-29(`c4cedda`)에 middie 취약점 패치를 위해 **소스를 통째로 복사해온(vendored)** 상태였다. 관련 PR `#16840`(같은 이슈 `#11802`를 다루려던 시도, 머지되지 않고 CLOSED)의 계획은 "나중에 다시 `@fastify/middie`를 직접 참조하는 방식으로 되돌리자"는 것이었다.

그런데 확인해보니, **`@fastify/middie` 본체는 2026-04-16(`fastify/middie@ca42dd7`)에 이미 이 정확히 같은 버그를 고쳤다** — `onRegister`가 `use()`를 다시 호출해서 재등록하던 방식을 버리고, `kMiddlewares`/`kMiddie`에 직접 push하는 방식으로 바꾼 것. Nest가 vendoring한 시점이 이 upstream 수정보다 먼저라서, vendored 사본은 그 수정을 받아본 적이 없다.

즉 지금 Nest 앱에서 여전히 겪는 `/queues` 같은 경로의 이중 prefix 문제는, **upstream이 3개월 전에 이미 고친 버그를 vendored 사본만 못 따라간 것**이다. 이 사실을 코멘트로 남기면서, "만약 `@fastify/middie`로 되돌리는 작업(de-vendoring)이 곧 있을 예정이면 이 PR은 닫아도 된다 — 아니라면 그때까지 버그는 여전히 살아있다"고 정리했다.

이건 우리 수정 방식이 임의로 지어낸 게 아니라 **upstream 메인테이너가 실제로 선택한 것과 같은 방향(재호출 대신 내부 컬렉션에 직접 push)**이라는 걸 뒷받침하는 근거이기도 하다.

---

## 9. 한 줄 결론

> Nest Fastify 어댑터가 부모 middleware를 자식 plugin에 넘길 때 공개 `use()`를 다시 써서 prefix를 한 번 더 붙인 것이 버그였고, 내부 middie 인스턴스에 직접 등록하도록 바꾸면서 해결했다.
