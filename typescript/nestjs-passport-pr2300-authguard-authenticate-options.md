# [Nest OSS 기여] nestjs/passport - `AuthGuard#getAuthenticateOptions()` 타입과 authenticate 옵션 경계 정리 (PR #2300)

> 오픈소스 기여 기록 및 학습 노트. 이 문서는 `nestjs/passport` PR #2300 하나를 다룬다. **최종 상태: 🟢 MERGED.**
>
> 이번 작업은 처음 보면 "타입 조금 바꾸고 옵션 몇 개 빼는 PR"처럼 보인다. 하지만 이해하려면 Nest의 Guard, Passport의 Strategy, `PassportModule.register()` 옵션, `passport.authenticate()` 옵션, 그리고 `getAuthenticateOptions()` hook이 각각 어느 층의 책임인지 먼저 구분해야 한다.
>
> 핵심은 다음 한 문장이다: `getAuthenticateOptions()`는 어떤 strategy를 쓸지 고르는 곳이 아니라, 이미 선택된 strategy를 실행할 때 `passport.authenticate()`에 넘길 세부 옵션을 만드는 곳이다. 기존 타입은 이 hook에서 `defaultStrategy`를 반환해도 되는 것처럼 보이게 했고, 런타임도 AuthModule 전용 옵션을 그대로 strategy 쪽 options로 흘려보내고 있었다.

---

## 1. 개요

### 1.1 대상

| 구분 | 내용 |
|---|---|
| 원본 이슈 | https://github.com/nestjs/passport/issues/947 |
| 제출한 PR | https://github.com/nestjs/passport/pull/2300 |
| 대상 저장소 | https://github.com/nestjs/passport |
| 내 Fork | https://github.com/GiHoon1123/passport |
| 작업 브랜치 | `fix-947-authenticate-options` |
| base 브랜치 | `master` |
| 최초 제출 커밋 | `823089372506def6dbe72d3753f9773913da18a0` |
| 머지 전 충돌 해소 커밋 | `849d11c6c7b0065b0708e49b79c1ee1835666041` |
| 머지 커밋 | `28e6701abb27a3d7a732d952b3a6967bc2daecf2` |
| PR 제목 | `fix(passport): strip auth module options from authenticate` |
| 상태 | 🟢 MERGED (2026-08-17 17:53:17 UTC / 2026-08-18 02:53:17 KST) |

### 1.2 한 줄 요약

> `AuthGuard#getAuthenticateOptions()`의 반환 타입을 `passport.authenticate()`에 실제로 넘기는 옵션에 가깝게 바꾸고, `passport.authenticate()` 호출 직전에 `defaultStrategy`/`property` 같은 AuthModule 전용 옵션을 제거했다. 기존 `PassportModule.register()` 설정 모양은 바꾸지 않았고, `property`는 여전히 `AuthGuard` 내부에서 request에 user를 넣을 때 사용된다.

### 1.3 건드린 파일

| 파일 | 역할 |
|---|---|
| `lib/auth.guard.ts` | `getAuthenticateOptions()` 반환 타입 수정, `AuthGuardAuthenticateOptions` 추가, `omitAuthModuleOptions()` 추가 |
| `test/common/app.e2e-spec.ts` | 새 e2e test suite 연결 |
| `test/options-check/options-check.module.ts` | 테스트용 Nest module |
| `test/options-check/options-check.controller.ts` | 테스트 endpoint |
| `test/options-check/options-check.strategy.ts` | strategy에 어떤 options가 전달되는지 검사하는 테스트용 Passport strategy |

---

## 2. 먼저 이해해야 하는 개념

이번 PR은 인증 흐름을 다룬다. 그래서 "Guard가 뭔지", "Strategy가 뭔지", "`defaultStrategy`가 어디서 쓰이는지"를 먼저 잡아야 한다.

### 2.1 Guard란

NestJS에서 Guard는 요청이 controller handler까지 들어가도 되는지 먼저 판단하는 문지기다.

예를 들어 이런 controller가 있다고 하자.

```ts
@Get('/profile')
getProfile() {
  return 'my profile';
}
```

이 상태면 아무 요청이나 `/profile`에 접근할 수 있다. 인증된 사용자만 접근하게 하려면 route 앞에 guard를 붙인다.

```ts
@UseGuards(AuthGuard('jwt'))
@Get('/profile')
getProfile() {
  return 'my profile';
}
```

이 뜻은 다음과 같다.

```txt
1. /profile 요청이 들어온다.
2. controller를 실행하기 전에 AuthGuard('jwt')를 먼저 실행한다.
3. guard가 통과시키면 getProfile()이 실행된다.
4. guard가 막으면 401 Unauthorized 같은 에러가 난다.
```

즉 Guard의 질문은 하나다.

```txt
이 요청을 통과시킬 것인가, 막을 것인가?
```

### 2.2 Strategy란

Strategy는 Passport에서 "인증을 어떤 방식으로 할 것인가"를 정의한 구현이다.

예를 들면:

| Strategy | 하는 일 |
|---|---|
| `jwt` strategy | `Authorization: Bearer <token>` 헤더를 읽고 JWT를 검증한다. |
| `local` strategy | username/password를 받아서 사용자를 찾고 비밀번호를 검증한다. |
| `google` strategy | Google OAuth 로그인 흐름을 시작하거나 callback을 처리한다. |

Guard와 Strategy의 관계는 다음처럼 이해하면 된다.

```txt
Guard = 입구 직원
Strategy = 검사 매뉴얼

AuthGuard('jwt') = 입구 직원에게 "JWT 검사 매뉴얼로 확인해"라고 시키는 것
AuthGuard('local') = 입구 직원에게 "username/password 검사 매뉴얼로 확인해"라고 시키는 것
```

Nest의 `AuthGuard('jwt')`는 내부적으로 결국 Passport에게 이런 일을 시킨다.

```ts
passport.authenticate('jwt', options)
```

여기서 `'jwt'`가 strategy 이름이다.

### 2.3 `AuthGuard('jwt')`와 `AuthGuard()`의 차이

보통은 guard에 strategy 이름을 직접 준다.

```ts
@UseGuards(AuthGuard('jwt'))
```

그러면 strategy 선택이 명확하다.

```txt
jwt strategy를 사용해라.
```

그런데 매번 `'jwt'`를 쓰기 귀찮거나, module 전체 기본값을 정하고 싶으면 `PassportModule.register()`에 `defaultStrategy`를 줄 수 있다.

```ts
PassportModule.register({
  defaultStrategy: 'jwt'
})
```

그 뒤에는 이렇게 strategy 이름을 생략할 수 있다.

```ts
@UseGuards(AuthGuard())
```

이 경우 `AuthGuard()`는 내부에서 다음처럼 판단한다.

```txt
AuthGuard()에 type 인자가 없네?
그러면 PassportModule.register()에서 받은 defaultStrategy를 쓰자.
```

즉 이 두 코드는 거의 같은 효과를 낸다.

```ts
@UseGuards(AuthGuard('jwt'))
```

```ts
PassportModule.register({ defaultStrategy: 'jwt' })

@UseGuards(AuthGuard())
```

### 2.4 `PassportModule.register()` 옵션의 의미

이번 PR에서 중요한 옵션은 세 개다.

```ts
PassportModule.register({
  defaultStrategy: 'jwt',
  property: 'account',
  session: false
})
```

각각 의미가 다르다.

| 옵션 | 어느 층의 옵션인가 | 의미 |
|---|---|---|
| `defaultStrategy` | Nest/AuthGuard 옵션 | `AuthGuard()`처럼 strategy 이름을 생략했을 때 기본으로 쓸 strategy |
| `property` | Nest/AuthGuard 옵션 | 인증 성공한 user를 request의 어느 필드에 넣을지 결정 |
| `session` | Passport authenticate 옵션 | 인증 결과를 session에 저장할지 여부 |

여기서 중요한 구분:

```txt
defaultStrategy/property는 Nest AuthGuard가 이해하는 옵션이다.
session은 Passport가 이해하는 authenticate 옵션이다.
```

### 2.5 `property` 옵션은 뭘 하나

기본적으로 Passport 인증이 성공하면 user는 보통 `req.user`에 들어간다.

```ts
req.user = user;
```

그런데 `property: 'account'`를 주면 `@nestjs/passport`의 `AuthGuard`는 인증된 user를 `req.account`에 넣는다.

```ts
PassportModule.register({
  property: 'account'
})
```

그러면 성공 후 request는 이렇게 된다.

```ts
req.account = user;
req.user // 없음
```

실제 `auth.guard.ts`에도 이 코드가 있다.

```ts
request[options.property || defaultOptions.property] = user;
```

`options.property`가 있으면 그 key에 user를 넣고, 없으면 기본값인 `'user'`를 쓴다.

### 2.6 `session: false` 옵션은 뭘 하나

Passport는 session 기반 로그인을 지원한다. session을 쓰면 서버가 "이 사용자는 로그인된 상태"라는 정보를 저장하고, 브라우저는 cookie를 들고 다닌다.

```txt
1. 로그인 성공
2. 서버 session store에 로그인 상태 저장
3. 브라우저는 session cookie를 받음
4. 다음 요청부터 cookie로 로그인 상태를 확인
```

하지만 JWT 기반 API 서버에서는 보통 session을 쓰지 않는다. 매 요청마다 token을 보내고 서버는 token을 검증한다.

```txt
Authorization: Bearer <jwt>
```

이때 `session: false`는 Passport에게 이렇게 말하는 옵션이다.

```txt
인증 결과를 서버 session에 저장하지 마라.
이번 요청에서만 인증 처리해라.
```

따라서 `session`은 `passport.authenticate()`에 넘겨야 하는 진짜 Passport 옵션이다. 이번 PR에서 `session`을 제거하면 안 되는 이유가 여기 있다.

### 2.7 `prompt: 'login'` 같은 옵션은 뭔가

`prompt`는 모든 strategy가 공통으로 쓰는 옵션은 아니고, Google/OIDC 같은 OAuth 계열 strategy에서 주로 의미가 있다.

```ts
getAuthenticateOptions() {
  return {
    prompt: 'login'
  };
}
```

서버가 브라우저 창을 직접 띄우는 건 아니다. OAuth 로그인 시작 endpoint에서 서버는 보통 302 redirect 응답을 보낸다.

```txt
HTTP/1.1 302 Found
Location: https://accounts.google.com/o/oauth2/v2/auth?...&prompt=login
```

브라우저가 이 응답을 보고 Google 로그인 페이지로 이동한다. Google은 URL에 붙은 `prompt=login`을 보고 "이미 로그인되어 있어도 다시 로그인 화면을 보여주자"고 판단한다.

즉:

```txt
AuthGuard('google')
= google strategy를 써라.

prompt: 'login'
= google strategy를 실행할 때 Google 인증 URL에 prompt=login을 붙여라.
```

이런 옵션은 strategy 선택 옵션이 아니라, 이미 선택된 strategy를 실행할 때 주는 세부 옵션이다.

### 2.8 `getAuthenticateOptions()`는 무엇인가

`getAuthenticateOptions()`는 `AuthGuard`를 상속해서 request/context별로 Passport authenticate 옵션을 동적으로 추가할 수 있게 해주는 hook이다.

예:

```ts
class GoogleAuthGuard extends AuthGuard('google') {
  getAuthenticateOptions(context: ExecutionContext) {
    return {
      prompt: 'login'
    };
  }
}
```

또는 async로 DB/config를 보고 옵션을 만들 수도 있다.

```ts
class MyAuthGuard extends AuthGuard('jwt') {
  async getAuthenticateOptions(context: ExecutionContext) {
    const req = context.switchToHttp().getRequest();

    return {
      session: false,
      audience: req.headers['x-audience']
    };
  }
}
```

핵심은 이것이다.

```txt
getAuthenticateOptions()는 어떤 strategy를 쓸지 고르는 곳이 아니다.
이미 선택된 strategy를 실행할 때 넘길 세부 options를 만드는 곳이다.
```

그래서 여기서 `defaultStrategy`를 반환하는 것은 위치가 틀린 사용이다.

```ts
// 잘못된 기대
getAuthenticateOptions() {
  return {
    defaultStrategy: 'google'
  };
}
```

이렇게 써도 strategy 선택은 바뀌지 않는다. strategy 선택은 `AuthGuard(type)` 또는 `PassportModule.register({ defaultStrategy })`에서 이미 끝난다.

---

## 3. 이슈 #947의 문제

### 3.1 이슈 원문이 지적한 것

이슈 #947은 `getAuthenticateOptions()`의 반환 타입이 misleading/incorrect하다고 지적했다.

기존 타입은 다음과 같았다.

```ts
getAuthenticateOptions(
  context: ExecutionContext
): IAuthModuleOptions | undefined;
```

그리고 `IAuthModuleOptions`는 다음처럼 생겼다.

```ts
export interface IAuthModuleOptions {
  defaultStrategy?: string | string[];
  session?: boolean;
  property?: string;
  [key: string]: any;
}
```

문제는 `IAuthModuleOptions` 안에 `defaultStrategy`가 들어 있다는 점이다. 그래서 TypeScript는 이런 코드를 허용했다.

```ts
class MyGuard extends AuthGuard() {
  getAuthenticateOptions() {
    return {
      defaultStrategy: 'google'
    };
  }
}
```

하지만 이 코드는 사용자가 기대하는 대로 동작하지 않는다. `getAuthenticateOptions()` 반환값의 `defaultStrategy`는 strategy 선택에 쓰이지 않는다.

### 3.2 실제 strategy 선택은 어디서 끝나는가

`auth.guard.ts`의 핵심 흐름은 다음과 같다.

```ts
const user = await passportFn(
  type || this.options.defaultStrategy!,
  options,
  (err, user, info, status) =>
    this.handleRequest(err, user, info, context, status)
);
```

여기서 첫 번째 인자가 strategy 이름이다.

```ts
type || this.options.defaultStrategy!
```

즉:

```txt
1. AuthGuard('jwt')처럼 type이 있으면 그 type을 쓴다.
2. type이 없으면 PassportModule.register()에서 받은 this.options.defaultStrategy를 쓴다.
```

`getAuthenticateOptions()`에서 반환한 `defaultStrategy`는 이 식에 들어가지 않는다.

### 3.3 왜 타입이 misleading한가

기존 타입은 사용자에게 이렇게 말하는 셈이었다.

```txt
getAuthenticateOptions()에서는 IAuthModuleOptions를 반환해도 됩니다.
IAuthModuleOptions에는 defaultStrategy가 있습니다.
따라서 defaultStrategy를 반환해도 됩니다.
```

하지만 실제 동작은 이렇다.

```txt
getAuthenticateOptions()에서 defaultStrategy를 반환해도 strategy 선택에 쓰이지 않습니다.
그 값은 그냥 passport.authenticate() options 객체 안에 섞여 들어갑니다.
```

즉 타입이 "가능하다"고 안내하는데, 런타임은 그 값을 의미 있게 사용하지 않는다. 이게 이슈의 핵심이다.

### 3.4 Promise 반환 타입도 빠져 있었다

기존 코드는 이미 `await`를 사용하고 있었다.

```ts
const options = {
  ...defaultOptions,
  ...this.options,
  ...(await this.getAuthenticateOptions(context))
};
```

`await this.getAuthenticateOptions(context)`라고 되어 있으므로, 런타임 관점에서는 `getAuthenticateOptions()`가 Promise를 반환해도 처리할 수 있다.

```ts
async getAuthenticateOptions() {
  return {
    session: false
  };
}
```

하지만 기존 public 타입에는 Promise가 빠져 있었다.

```ts
getAuthenticateOptions(
  context: ExecutionContext
): IAuthModuleOptions | undefined;
```

따라서 실제 런타임은 async 반환을 받을 수 있는데, 타입은 그 사실을 보여주지 않았다. 이것도 이슈 #947의 요구사항 중 하나였다.

---

## 4. 기존 런타임 코드 흐름

### 4.1 옵션이 한 객체로 합쳐지는 구조

기존 `canActivate()`는 options를 이렇게 만들었다.

```ts
const options = {
  ...defaultOptions,
  ...this.options,
  ...(await this.getAuthenticateOptions(context))
};
```

각각의 의미:

```ts
defaultOptions
```

기본값이다.

```ts
export const defaultOptions = {
  session: false,
  property: 'user'
};
```

```ts
this.options
```

`PassportModule.register()`에서 받은 module-level options다.

```ts
PassportModule.register({
  defaultStrategy: 'options-check',
  property: 'account',
  session: false
})
```

```ts
await this.getAuthenticateOptions(context)
```

Guard가 request/context별로 추가한 authenticate options다.

예:

```ts
{
  prompt: 'login'
}
```

이 세 개가 합쳐지면 이런 객체가 된다.

```ts
{
  session: false,
  property: 'account',
  defaultStrategy: 'options-check',
  prompt: 'login'
}
```

### 4.2 기존에는 이 객체가 그대로 Passport로 갔다

기존 코드는 합쳐진 `options` 전체를 그대로 `passportFn`에 넘겼다.

```ts
const user = await passportFn(
  type || this.options.defaultStrategy!,
  options,
  callback
);
```

`passportFn`은 내부에서 결국 `passport.authenticate()`를 호출한다.

```ts
passport.authenticate(type, options, (err, user, info, status) => {
  ...
})
```

따라서 실제 호출은 이런 꼴이 된다.

```ts
passport.authenticate('options-check', {
  session: false,
  property: 'account',
  defaultStrategy: 'options-check',
  prompt: 'login'
});
```

여기서 `session`과 `prompt`는 Passport 또는 strategy가 이해할 수 있는 options다.

하지만 `defaultStrategy`와 `property`는 `@nestjs/passport`의 AuthGuard 내부 옵션이다. `passport.authenticate()`에 넘길 필요가 없다.

### 4.3 왜 이게 런타임에서도 정리되어야 하는가

타입만 고쳐도 `defaultStrategy`를 `getAuthenticateOptions()`에서 반환하는 신규 코드는 줄어든다. 하지만 기존 `PassportModule.register()`에서 온 `defaultStrategy`/`property`는 여전히 merged options에 들어간다.

예:

```ts
PassportModule.register({
  defaultStrategy: 'jwt',
  property: 'account',
  session: false
})
```

이 설정만으로도 merged options에는 `defaultStrategy`와 `property`가 들어간다.

```ts
{
  defaultStrategy: 'jwt',
  property: 'account',
  session: false
}
```

그러므로 `passport.authenticate()`에 넘기기 전에는 AuthModule 전용 옵션을 제거해야 한다.

---

## 5. 메인테이너 코멘트와 이전 PR들

### 5.1 PR #1610이 왜 닫혔는가

기존에 PR #1610이 있었다.

그 PR은 문제를 더 크게 해결하려고 `authenticateOptions` 같은 nested configuration object를 도입하는 방향이었다.

예를 들어 사용자가 기존에 이렇게 쓰던 것을:

```ts
PassportModule.register({
  defaultStrategy: 'local',
  session: true,
  failureRedirect: '/access-denied',
  successRedirect: '/home'
});
```

이런 식으로 바꾸게 만드는 방향이었다.

```ts
PassportModule.register({
  defaultStrategy: 'local',
  session: true,
  authenticateOptions: {
    failureRedirect: '/access-denied',
    successRedirect: '/home'
  }
});
```

이건 기존 사용자 설정 모양을 바꾸는 큰 변경이다. 그래서 Kamil이 "major breaking change"라고 봤고, 새 nested config를 만들기보다 AuthModule-specific props만 strip하는 쪽을 제안했다.

이 코멘트의 의미는 다음처럼 해석했다.

```txt
기존 PassportModule.register() API 모양을 바꾸지 마라.
사용자가 기존에 쓰던 session/failureRedirect 같은 옵션을 다른 곳으로 옮기게 만들지 마라.
그냥 passport.authenticate()에 넘기기 전에 Nest AuthModule 전용 옵션만 빼라.
```

### 5.2 PR #2259는 어떤 상태였나

그 뒤 PR #2259가 올라왔다. 방향은 이번 PR과 매우 비슷했다.

핵심은:

```txt
1. getAuthenticateOptions()에 Promise 반환 타입 반영
2. passport.authenticate() 호출 전에 module-only options strip
3. 관련 e2e 테스트 추가
```

하지만 그 PR은 메인테이너 리뷰 없이 작성자가 직접 닫았다. 즉 "이 접근은 메인테이너가 거절했다"는 근거는 없었다.

그래서 이번 작업의 판단은 다음과 같았다.

```txt
#1610처럼 새 nested config를 만들면 안 된다.
#2259처럼 strip 방식으로 가는 건 메인테이너 코멘트와 맞다.
다만 #947 원문이 타입 문제를 직접 말하므로, 타입도 조금 더 정확하게 다듬는다.
```

---

## 6. 최종 구현

### 6.1 타입 변경 전

기존 `IAuthGuard`의 `getAuthenticateOptions()` 타입은 다음과 같았다.

```ts
export type IAuthGuard = CanActivate & {
  ...
  getAuthenticateOptions(
    context: ExecutionContext
  ): IAuthModuleOptions | undefined;
  ...
};
```

이 타입은 두 가지 문제가 있었다.

```txt
1. IAuthModuleOptions라서 defaultStrategy를 반환할 수 있는 것처럼 보인다.
2. Promise 반환이 빠져 있다.
```

### 6.2 타입 변경 후

새 타입을 추가했다.

```ts
export type AuthGuardAuthenticateOptions = passport.AuthenticateOptions & {
  defaultStrategy?: never;
  [key: string]: any;
};
```

그리고 `getAuthenticateOptions()` 반환 타입을 바꿨다.

```ts
getAuthenticateOptions(
  context: ExecutionContext
):
  | Promise<AuthGuardAuthenticateOptions>
  | AuthGuardAuthenticateOptions
  | undefined;
```

의미는 다음과 같다.

```txt
getAuthenticateOptions()는 passport.authenticate()에 넘길 options를 반환한다.
그러므로 passport.AuthenticateOptions 기반으로 잡는다.
defaultStrategy는 여기서 의미가 없으므로 반환하지 못하게 한다.
async 반환도 실제 코드처럼 허용한다.
custom strategy option은 있을 수 있으므로 index signature는 유지한다.
```

### 6.3 `defaultStrategy?: never`의 의미

이 코드는 `defaultStrategy`를 "있으면 안 되는 property"로 표현한다.

```ts
defaultStrategy?: never;
```

그래서 이런 코드는 타입 에러가 나야 한다.

```ts
getAuthenticateOptions() {
  return {
    defaultStrategy: 'google'
  };
}
```

왜냐하면 `defaultStrategy`가 있다면 그 값은 `never` 타입이어야 하는데, 실제 값은 string이기 때문이다.

반면 이런 코드는 허용된다.

```ts
getAuthenticateOptions() {
  return {
    session: false,
    prompt: 'login'
  };
}
```

`session`은 `passport.AuthenticateOptions`에 있는 정상 옵션이고, `prompt`도 `@types/passport`에 들어 있는 authenticate option이다.

또 이런 custom strategy option도 허용된다.

```ts
getAuthenticateOptions() {
  return {
    customTenant: 'acme'
  };
}
```

`[key: string]: any`를 유지했기 때문에 strategy별 custom option을 막지 않는다.

### 6.4 왜 `property?: never`는 타입에서 뺐나

처음 검토할 때는 `property?: never`까지 넣는 선택도 있었다.

```ts
export type AuthGuardAuthenticateOptions = passport.AuthenticateOptions & {
  defaultStrategy?: never;
  property?: never;
  [key: string]: any;
};
```

하지만 최종 PR에서는 `property?: never`를 넣지 않았다.

이유는 최소 변경 관점 때문이다.

이슈 #947 원문은 `defaultStrategy`가 `getAuthenticateOptions()`에서 허용되는 것처럼 보이는 문제를 직접 지적했다. expected behavior도 `defaultStrategy`를 제외하는 쪽이었다.

`property`는 `passport.authenticate()`에 넘길 옵션은 아니지만, `AuthGuard` 내부에서 실제로 request field를 정하는 데 쓰인다.

```ts
request[options.property || defaultOptions.property] = user;
```

그래서 runtime에서는 `property`를 `passport.authenticate()`에 넘기지 않도록 strip하지만, 타입 레벨에서까지 `property`를 금지하는 것은 이슈 원문보다 한 발 더 나가는 변경으로 보일 수 있었다.

최종 판단:

```txt
runtime에서는 defaultStrategy/property 둘 다 strip한다.
type에서는 이슈 원문이 직접 문제 삼은 defaultStrategy만 막는다.
```

이렇게 하면 PR이 더 보수적으로 보인다.

### 6.5 런타임 변경 전

기존에는 merged options 전체가 그대로 `passportFn`에 넘어갔다.

```ts
const user = await passportFn(
  type || this.options.defaultStrategy!,
  options,
  (err, user, info, status) =>
    this.handleRequest(err, user, info, context, status)
);
```

예를 들어 options가 이렇게 만들어졌다면:

```ts
{
  defaultStrategy: 'options-check',
  property: 'account',
  session: false
}
```

Passport strategy는 이 전체를 받았다.

```ts
{
  defaultStrategy: 'options-check',
  property: 'account',
  session: false
}
```

### 6.6 런타임 변경 후

이제 `passportFn` 호출 전 `omitAuthModuleOptions()`를 거친다.

```ts
const user = await passportFn(
  type || this.options.defaultStrategy!,
  omitAuthModuleOptions(options),
  (err, user, info, status) =>
    this.handleRequest(err, user, info, context, status)
);
```

helper는 다음과 같다.

```ts
function omitAuthModuleOptions(
  options: IAuthModuleOptions
): AuthGuardAuthenticateOptions {
  const { defaultStrategy, property, ...authenticateOptions } = options;
  return authenticateOptions;
}
```

이 코드는 object rest destructuring을 사용한다.

```ts
const { defaultStrategy, property, ...authenticateOptions } = options;
```

뜻은:

```txt
options에서 defaultStrategy를 꺼낸다.
options에서 property를 꺼낸다.
나머지 property들을 authenticateOptions라는 새 객체에 담는다.
```

예:

```ts
const options = {
  defaultStrategy: 'options-check',
  property: 'account',
  session: false,
  prompt: 'login'
};

const { defaultStrategy, property, ...authenticateOptions } = options;
```

그러면 `authenticateOptions`는 다음과 같다.

```ts
{
  session: false,
  prompt: 'login'
}
```

즉 Passport에는 이 값만 넘어간다.

```ts
passport.authenticate('options-check', {
  session: false,
  prompt: 'login'
});
```

### 6.7 원본 `options`를 mutate하지 않는 점

`omitAuthModuleOptions()`는 원본 객체에서 property를 삭제하지 않는다.

이런 방식이 아니다.

```ts
delete options.defaultStrategy;
delete options.property;
return options;
```

대신 새 객체를 만든다.

```ts
const { defaultStrategy, property, ...authenticateOptions } = options;
return authenticateOptions;
```

따라서 `canActivate()`의 뒤쪽에서는 기존 `options.property`를 그대로 쓸 수 있다.

```ts
request[options.property || defaultOptions.property] = user;
```

이게 중요하다. 만약 원본 `options.property`를 지워버렸다면, `property: 'account'` 설정이 깨져서 user가 `req.account`가 아니라 `req.user`에 들어갈 수 있다.

최종 구현은 이 문제를 피한다.

```txt
Passport에 넘기는 객체에서만 defaultStrategy/property를 제거한다.
AuthGuard 내부에서 쓰는 원본 options는 유지한다.
```

---

## 7. 코드 흐름을 실제 예시로 따라가기

테스트에서 사용한 설정을 그대로 보자.

```ts
PassportModule.register({
  defaultStrategy: 'options-check',
  property: 'account',
  session: false
})
```

그리고 controller는 이렇게 되어 있다.

```ts
@UseGuards(AuthGuard())
@Get('options-check')
checkOptions(@Req() req: any) {
  return {
    account: req.account,
    authInfo: req.authInfo,
    user: req.user ?? null
  };
}
```

### 7.1 요청이 들어온다

테스트는 다음 endpoint를 호출한다.

```ts
await spec()
  .get('/options-check')
```

이 route에는 `AuthGuard()`가 붙어 있다.

```ts
@UseGuards(AuthGuard())
```

`AuthGuard()`는 strategy 이름이 없다. 그러면 `PassportModule.register()`에서 받은 `defaultStrategy`를 사용한다.

```ts
type || this.options.defaultStrategy!
```

여기서:

```txt
type = undefined
this.options.defaultStrategy = 'options-check'
```

결과:

```ts
'options-check'
```

따라서 Passport는 `options-check` strategy를 실행한다.

### 7.2 options 객체가 만들어진다

`defaultOptions`:

```ts
{
  session: false,
  property: 'user'
}
```

`this.options`:

```ts
{
  defaultStrategy: 'options-check',
  property: 'account',
  session: false
}
```

`getAuthenticateOptions()`:

```ts
undefined
```

합치면:

```ts
{
  session: false,
  property: 'account',
  defaultStrategy: 'options-check'
}
```

### 7.3 Passport에 넘기기 전에 strip한다

이제 이 객체를 그대로 넘기지 않고 `omitAuthModuleOptions()`에 넣는다.

```ts
omitAuthModuleOptions({
  session: false,
  property: 'account',
  defaultStrategy: 'options-check'
})
```

반환값:

```ts
{
  session: false
}
```

따라서 strategy는 다음 options만 받는다.

```ts
{
  session: false
}
```

### 7.4 strategy가 검사한다

테스트용 strategy는 받은 options에 `defaultStrategy`나 `property`가 있으면 실패하게 되어 있다.

```ts
const leakedOptionKeys = ['defaultStrategy', 'property'].filter(
  (key) => key in options
);

if (leakedOptionKeys.length > 0) {
  return this.fail(
    { message: `Leaked options: ${leakedOptionKeys.join(', ')}` },
    401
  );
}
```

수정 전이면 strategy가 이런 options를 받는다.

```ts
{
  session: false,
  property: 'account',
  defaultStrategy: 'options-check'
}
```

그러면 `leakedOptionKeys`는 다음이 된다.

```ts
['defaultStrategy', 'property']
```

결과적으로 `this.fail(..., 401)`이 호출되고 테스트는 실패해야 한다.

수정 후에는 strategy가 이런 options를 받는다.

```ts
{
  session: false
}
```

그러면 `leakedOptionKeys`는 빈 배열이다.

```ts
[]
```

strategy는 성공 처리한다.

```ts
return this.success(
  { id: 'account-1' },
  {
    receivedSessionOption: options.session
  }
);
```

여기서 user는:

```ts
{ id: 'account-1' }
```

authInfo는:

```ts
{ receivedSessionOption: false }
```

### 7.5 AuthGuard가 user를 request에 넣는다

strategy가 성공하면 `AuthGuard`는 user를 request에 저장한다.

```ts
request[options.property || defaultOptions.property] = user;
```

여기서 중요한 점은 `options.property`가 아직 살아 있다는 것이다.

```ts
options.property = 'account'
```

따라서 실제로는:

```ts
request['account'] = user;
```

즉:

```ts
req.account = { id: 'account-1' };
```

`req.user`는 설정하지 않는다.

### 7.6 controller 응답

controller는 다음을 반환한다.

```ts
return {
  account: req.account,
  authInfo: req.authInfo,
  user: req.user ?? null
};
```

기대값은:

```ts
{
  account: { id: 'account-1' },
  authInfo: { receivedSessionOption: false },
  user: null
}
```

각 값의 의미:

| 응답 필드 | 의미 |
|---|---|
| `account: { id: 'account-1' }` | `property: 'account'`가 여전히 AuthGuard 내부에서 정상 동작한다. |
| `authInfo.receivedSessionOption: false` | `session: false`는 strip되지 않고 strategy까지 전달된다. |
| `user: null` | user가 `req.user`가 아니라 `req.account`에 들어갔다. |

---

## 8. 테스트 코드 상세 해설

### 8.1 테스트용 module

```ts
@Module({
  controllers: [OptionsCheckController],
  imports: [
    PassportModule.register({
      defaultStrategy: 'options-check',
      property: 'account',
      session: false
    })
  ],
  providers: [OptionsCheckStrategy]
})
export class OptionsCheckModule {}
```

이 module은 일부러 세 옵션을 모두 넣는다.

```txt
defaultStrategy: strategy 선택용 Nest/AuthGuard 옵션
property: request field 선택용 Nest/AuthGuard 옵션
session: Passport authenticate 옵션
```

테스트 목표는:

```txt
defaultStrategy/property는 strategy에 넘어가면 안 된다.
session은 strategy에 넘어가야 한다.
```

### 8.2 테스트용 strategy

```ts
@Injectable()
export class OptionsCheckStrategy extends passport.Strategy {
  name = 'options-check';

  constructor() {
    super();
    passport.use(this);
  }

  authenticate(req: any, options: Record<string, any>) {
    ...
  }
}
```

`name = 'options-check'`이므로 `AuthGuard()`가 `defaultStrategy: 'options-check'`를 사용하면 이 strategy가 실행된다.

`passport.use(this)`는 이 strategy 인스턴스를 Passport registry에 등록한다.

### 8.3 왜 `passport.Strategy`를 직접 상속했나

처음에는 `passport-strategy` 패키지를 직접 import하는 방법도 가능했다.

```ts
import { Strategy } from 'passport-strategy';

export class OptionsCheckStrategy extends Strategy {
  ...
}
```

하지만 이 repo의 `package.json`에는 `passport-strategy`가 직접 dependency로 들어 있지 않다. `passport`가 내부적으로 가져오는 transitive dependency다.

테스트 코드가 transitive dependency를 직접 import하면 약간 지저분하다. 그래서 최종적으로 `passport`가 export하는 `Strategy`를 사용했다.

```ts
import * as passport from 'passport';

export class OptionsCheckStrategy extends passport.Strategy {
  ...
}
```

이렇게 하면 `package.json`에 직접 선언된 `passport`만 사용한다.

### 8.4 strategy에서 leaked options를 잡는 방식

```ts
const leakedOptionKeys = ['defaultStrategy', 'property'].filter(
  (key) => key in options
);
```

이 코드는 strategy가 받은 `options` 객체 안에 `defaultStrategy`나 `property`가 있는지 확인한다.

수정 전:

```ts
options = {
  defaultStrategy: 'options-check',
  property: 'account',
  session: false
}
```

결과:

```ts
leakedOptionKeys = ['defaultStrategy', 'property']
```

그러면 실패:

```ts
return this.fail(
  { message: `Leaked options: ${leakedOptionKeys.join(', ')}` },
  401
);
```

수정 후:

```ts
options = {
  session: false
}
```

결과:

```ts
leakedOptionKeys = []
```

그러면 성공:

```ts
return this.success(
  { id: 'account-1' },
  {
    receivedSessionOption: options.session
  }
);
```

### 8.5 왜 `session` 값을 authInfo에 넣었나

테스트는 단지 `defaultStrategy/property`가 제거되었는지만 보면 부족하다.

만약 구현을 잘못해서 options 전체를 비워버리면:

```ts
passport.authenticate('options-check', {})
```

`defaultStrategy/property`는 안 넘어가지만, `session: false`도 같이 사라진다. 그건 잘못된 수정이다.

그래서 strategy는 `options.session`을 authInfo에 담아 반환한다.

```ts
{
  receivedSessionOption: options.session
}
```

그리고 e2e test는 이 값을 확인한다.

```ts
authInfo: { receivedSessionOption: false }
```

이 검증이 말하는 것:

```txt
AuthModule 전용 옵션만 제거했다.
진짜 Passport option인 session은 유지했다.
```

### 8.6 e2e test

```ts
describe('Passport authenticate options', () => {
  let app: INestApplication;

  beforeAll(async () => {
    const modRef = await Test.createTestingModule({
      imports: [OptionsCheckModule]
    }).compile();
    app = modRef.createNestApplication();
    await app.listen(0);
    const url = (await app.getUrl()).replace('[::1]', 'localhost');
    request.setBaseUrl(url);
  });

  it('should not pass module-only options to the strategy', async () => {
    await spec()
      .get('/options-check')
      .expectStatus(200)
      .expectBody({
        account: { id: 'account-1' },
        authInfo: { receivedSessionOption: false },
        user: null
      });
  });

  afterAll(async () => {
    await app.close();
  });
});
```

이 테스트 하나가 검증하는 내용은 세 가지다.

```txt
1. defaultStrategy/property가 strategy options로 leak되지 않는다.
2. session은 strategy options로 전달된다.
3. property는 여전히 AuthGuard 내부에서 request assignment에 사용된다.
```

---

## 9. 왜 이 변경이 메인테이너 요구와 맞는가

### 9.1 메인테이너가 원하지 않았던 방향

메인테이너가 PR #1610에서 우려한 것은 "새 nested config object"를 추가하는 방식이었다.

그 방식은 사용자가 기존 설정을 바꿔야 한다.

```ts
// 기존
PassportModule.register({
  defaultStrategy: 'local',
  failureRedirect: '/access-denied'
});

// 새 방식
PassportModule.register({
  defaultStrategy: 'local',
  authenticateOptions: {
    failureRedirect: '/access-denied'
  }
});
```

이건 기존 사용자에게 migration을 요구한다. 그래서 breaking change로 볼 수 있다.

### 9.2 이번 PR이 택한 방향

이번 PR은 설정 구조를 바꾸지 않았다.

사용자는 여전히 이렇게 쓸 수 있다.

```ts
PassportModule.register({
  defaultStrategy: 'local',
  failureRedirect: '/access-denied',
  session: false
});
```

대신 내부에서 Passport로 넘길 때만 AuthModule 전용 옵션을 제거한다.

```ts
function omitAuthModuleOptions(
  options: IAuthModuleOptions
): AuthGuardAuthenticateOptions {
  const { defaultStrategy, property, ...authenticateOptions } = options;
  return authenticateOptions;
}
```

즉:

```txt
외부 API shape는 유지한다.
내부 boundary에서만 잘못 섞인 옵션을 분리한다.
```

이게 Kamil이 말한 "AuthModule-specific props만 strip" 방향과 맞다.

### 9.3 타입 변경은 왜 포함했나

이슈 #947은 런타임 leak만 말한 이슈가 아니다. 제목부터 `getAuthenticateOptions` 결과 타입이 misleading/incorrect하다는 내용이었다.

그래서 runtime strip만 하면 반쪽 해결이다.

필요한 것은 두 가지였다.

```txt
1. getAuthenticateOptions()에서 defaultStrategy를 반환할 수 있는 것처럼 보이지 않게 한다.
2. Promise 반환을 타입에 반영한다.
```

이번 PR은 이 둘을 모두 처리한다.

```ts
getAuthenticateOptions(
  context: ExecutionContext
):
  | Promise<AuthGuardAuthenticateOptions>
  | AuthGuardAuthenticateOptions
  | undefined;
```

---

## 10. "최소 변경" 관점에서의 자체 검토

### 10.1 한 일

실제 production code 변경은 `lib/auth.guard.ts` 하나다.

변경 내용:

```txt
1. getAuthenticateOptions() 반환 타입 변경
2. AuthGuardAuthenticateOptions type 추가
3. passportFn 호출 시 omitAuthModuleOptions(options) 사용
4. omitAuthModuleOptions() helper 추가
```

설정 구조 변경 없음.

```txt
PassportModule.register() API 변경 없음.
새 authenticateOptions 필드 추가 없음.
기존 options merge 순서 변경 없음.
request assignment 로직 변경 없음.
logIn() 로직 변경 없음.
handleRequest() 로직 변경 없음.
```

### 10.2 왜 helper를 추가했나

inline으로 이렇게 쓸 수도 있었다.

```ts
const { defaultStrategy, property, ...authenticateOptions } = options;

const user = await passportFn(
  type || this.options.defaultStrategy!,
  authenticateOptions,
  callback
);
```

하지만 현재 구조에서는 `options`를 만든 뒤 `passportFn` 호출만 바뀌는 편이 diff가 작고, helper 이름이 의도를 바로 보여준다.

```ts
omitAuthModuleOptions(options)
```

이 이름은 "이 객체에서 AuthModule 옵션을 빼고 Passport authenticate 옵션만 남긴다"는 의미를 가진다.

함수도 작다.

```ts
function omitAuthModuleOptions(
  options: IAuthModuleOptions
): AuthGuardAuthenticateOptions {
  const { defaultStrategy, property, ...authenticateOptions } = options;
  return authenticateOptions;
}
```

### 10.3 타입을 더 작게 바꿀 수 있었나

가능했다. 가장 보수적인 대안은 이슈 원문 expected에 더 가깝게 `Omit<IAuthModuleOptions, 'defaultStrategy'>`를 쓰는 것이다.

```ts
type AuthGuardAuthenticateOptions = Omit<IAuthModuleOptions, 'defaultStrategy'>;
```

또는 새 타입을 export하지 않고 method signature에 직접 쓸 수도 있다.

```ts
getAuthenticateOptions(
  context: ExecutionContext
):
  | Promise<Omit<IAuthModuleOptions, 'defaultStrategy'>>
  | Omit<IAuthModuleOptions, 'defaultStrategy'>
  | undefined;
```

하지만 `getAuthenticateOptions()`의 실제 목적은 `IAuthModuleOptions`가 아니라 `passport.authenticate()` options를 반환하는 것이다. 그래서 `passport.AuthenticateOptions` 기반 타입이 의미적으로 더 정확하다고 봤다.

최종 타입:

```ts
export type AuthGuardAuthenticateOptions = passport.AuthenticateOptions & {
  defaultStrategy?: never;
  [key: string]: any;
};
```

이 타입은 다음 균형을 잡는다.

```txt
passport.authenticate()의 공통 옵션을 반영한다.
defaultStrategy는 막는다.
custom strategy option은 막지 않는다.
```

### 10.4 public API 타입 추가 리스크

`lib/index.ts`는 `auth.guard.ts`를 export한다.

```ts
export * from './auth.guard';
```

따라서 새 `AuthGuardAuthenticateOptions`도 public export가 된다.

이건 의도한 공개 타입으로 볼 수 있다. 사용자가 custom guard를 작성할 때 반환 타입으로 명시할 수 있기 때문이다.

```ts
import { AuthGuardAuthenticateOptions } from '@nestjs/passport';

class MyGuard extends AuthGuard('google') {
  getAuthenticateOptions(): AuthGuardAuthenticateOptions {
    return {
      prompt: 'login'
    };
  }
}
```

다만 메인테이너가 "public type을 새로 추가하지 말고 더 작은 signature로 가자"고 요청할 가능성은 있다. 그 경우 fallback은 `Omit<IAuthModuleOptions, 'defaultStrategy'>` 쪽이다.

현재 판단은:

```txt
semantic correctness를 위해 passport.AuthenticateOptions 기반 타입을 선택했다.
하지만 maintainer가 더 작은 타입 변경을 원하면 쉽게 줄일 수 있다.
```

---

## 11. 검증

### 11.1 로컬 검증 명령

PR 전 로컬에서 실행한 명령:

```sh
npm test
npm run build
npm run lint
```

추가로 test 파일은 기본 lint 범위 밖이라 별도 확인도 했다.

```sh
./node_modules/.bin/oxlint test/options-check test/common/app.e2e-spec.ts
npx prettier --check lib/auth.guard.ts test/common/app.e2e-spec.ts test/options-check/options-check.controller.ts test/options-check/options-check.module.ts test/options-check/options-check.strategy.ts
git diff --check
```

결과:

```txt
npm test      -> 7 passed
npm run build -> passed
npm run lint  -> passed
oxlint test   -> passed
prettier      -> passed
git diff      -> passed
```

### 11.2 회귀 확인

수정 전 상태로 `omitAuthModuleOptions(options)`를 빼고 다시 `options`를 그대로 넘기면, 새 e2e 테스트는 실패해야 한다.

왜냐하면 strategy가 `defaultStrategy`/`property` leak을 보고 401을 반환하기 때문이다.

실제 실패 흐름:

```txt
AuthGuard() 실행
options = { defaultStrategy, property, session }
passport.authenticate()에 options 전체 전달
OptionsCheckStrategy.authenticate()에서 leakedOptionKeys 발견
this.fail(..., 401)
테스트는 expectStatus(200)이라 실패
```

이 테스트가 단순 snapshot이 아니라 실제 버그 경계를 잡고 있다는 뜻이다.

---

## 12. PR 본문 작성 기준

PR 본문은 일부러 너무 길게 쓰지 않았다. 메인테이너는 이미 #947/#1610 맥락을 알고 있고, 너무 방어적으로 쓰면 오히려 review surface가 커진다.

최종 PR 본문 핵심:

```txt
Issue Number: #947

getAuthenticateOptions() is typed as returning IAuthModuleOptions,
so defaultStrategy looks like a valid return value even though it is ignored there.
The type also misses the Promise return case.

The merged options are also passed directly to passport.authenticate(),
including AuthModule-only fields.
```

새 동작:

```txt
getAuthenticateOptions() now uses a type closer to passport.authenticate() options
and includes the Promise return case.

Before calling passport.authenticate(), defaultStrategy and property are stripped.
The existing module options shape is unchanged.
property is still used by AuthGuard when assigning the authenticated user.
```

마지막에 #1610과의 연결도 짧게만 남겼다.

```txt
This follows the approach suggested in #1610.
```

이 정도면:

```txt
문제 설명 충분
메인테이너 코멘트 반영 표시
breaking 아님을 설명
테스트 명시
AI처럼 과하게 장황하지 않음
```

---

## 13. 이번 작업에서 배운 것

### 13.1 옵션 이름이 같아 보여도 층이 다르면 의미가 다르다

이번 이슈는 "options 객체 하나"처럼 보이지만 실제로는 여러 층의 옵션이 섞여 있었다.

```txt
Nest module option
Nest AuthGuard internal option
Passport authenticate option
Strategy-specific option
```

이걸 구분하지 않으면 `defaultStrategy`를 `getAuthenticateOptions()`에 넣어도 될 것처럼 착각한다.

### 13.2 타입은 문서 역할을 한다

TypeScript 타입은 단순히 컴파일을 통과시키는 도구가 아니다. 사용자가 API를 어떻게 이해할지 안내하는 문서 역할도 한다.

기존 타입:

```ts
getAuthenticateOptions(): IAuthModuleOptions | undefined
```

이 타입은 사용자에게 "AuthModuleOptions를 반환해도 된다"고 말한다.

하지만 실제 책임은:

```txt
passport.authenticate()에 넘길 options를 반환한다.
```

그러면 타입도 그 책임에 맞아야 한다.

### 13.3 런타임 boundary도 같이 정리해야 한다

타입만 바꾸면 신규 TypeScript 사용자에게는 도움이 된다. 하지만 이미 module options에서 온 `defaultStrategy`/`property`는 런타임 merged options에 계속 들어간다.

그래서 boundary에서 실제 객체도 정리해야 한다.

```ts
omitAuthModuleOptions(options)
```

이런 helper는 작지만 의미가 크다.

```txt
AuthGuard 내부 옵션과 Passport authenticate 옵션의 경계를 코드로 명확히 표현한다.
```

### 13.4 최소 변경과 의미적 정확성의 균형

처음에는 `property?: never`까지 타입에서 막는 선택도 고려했다. 하지만 이슈 원문은 `defaultStrategy`를 직접 문제 삼고 있었고, `property`는 AuthGuard 내부에서 실제로 쓰이는 옵션이다.

최종적으로:

```txt
runtime strip: defaultStrategy + property
type restriction: defaultStrategy only
```

이 균형이 가장 보수적이라고 판단했다.

### 13.5 이전 PR이 닫혔다고 해서 방향이 틀린 것은 아니다

PR #2259는 같은 문제를 비슷한 방향으로 처리했지만 self-close되었다. 중요한 건 "메인테이너가 왜 닫았는가"다.

이번 경우:

```txt
#1610은 메인테이너가 breaking change라고 명확히 지적했다.
#2259는 메인테이너 rejection 없이 작성자가 닫았다.
```

따라서 #2259가 닫힌 것을 "strip 방식이 틀렸다"는 신호로 해석하면 안 된다. 오히려 #1610의 메인테이너 코멘트를 기준으로 보면 strip 방식이 맞다.

---

## 14. 최종 정리

### 14.1 문제

```txt
getAuthenticateOptions()는 passport.authenticate()에 넘길 세부 options를 반환하는 hook이다.
하지만 기존 타입은 IAuthModuleOptions라서 defaultStrategy를 반환해도 되는 것처럼 보였다.
defaultStrategy는 여기서 strategy 선택에 쓰이지 않으므로 misleading했다.
또한 실제 코드는 await하고 있었지만 Promise 반환 타입이 빠져 있었다.
런타임에서도 merged options 전체가 passport.authenticate()에 전달되어 defaultStrategy/property가 strategy options로 새고 있었다.
```

### 14.2 해결

```txt
getAuthenticateOptions() 반환 타입을 AuthGuardAuthenticateOptions로 바꿨다.
AuthGuardAuthenticateOptions는 passport.AuthenticateOptions 기반이고 defaultStrategy를 막는다.
Promise 반환도 타입에 반영했다.
passport.authenticate() 호출 직전에 omitAuthModuleOptions()로 defaultStrategy/property를 제거한다.
기존 options 객체는 mutate하지 않아서 property는 request assignment에 계속 쓰인다.
```

### 14.3 테스트

```txt
OptionsCheckModule에서 defaultStrategy/property/session을 모두 설정한다.
OptionsCheckStrategy는 defaultStrategy/property가 options로 들어오면 401로 실패한다.
동시에 session 값은 authInfo로 돌려줘서 session이 유지되는지 검증한다.
controller 응답에서 account/user를 확인해 property가 여전히 AuthGuard 내부에서 동작하는지도 검증한다.
```

### 14.4 메인테이너 요구 충족 여부

```txt
#1610에서 Kamil은 새 nested config object 대신 AuthModule-specific props만 strip하는 방향을 제안했다.
이번 PR은 새 config를 만들지 않고 기존 module options shape를 유지한다.
단지 passport.authenticate() 호출 전 boundary에서 defaultStrategy/property를 제거한다.
따라서 메인테이너 코멘트의 방향과 맞다.
```

---

## 15. 머지 결과

2026-08-17 17:48:29 UTC에 관련 이슈 #947이 `Closed` 상태가 됐다.  
그 뒤 2026-08-17 17:50:02 UTC에 Kamil이 `master`를 `fix-947-authenticate-options` 브랜치에 병합하고 충돌을 해소하는 커밋 `849d11c`를 추가했다.

이후 `build-and-test` check가 성공했고, 2026-08-17 17:53:17 UTC에 PR #2300이 merge commit `28e6701`로 `nestjs:master`에 병합됐다. GitHub PR 상태는 최종적으로 `MERGED`다.

별도 review record나 PR 코멘트는 없었다. Kamil이 `bug` label을 붙인 뒤, 충돌 해소와 CI 통과를 거쳐 바로 머지한 형태다.
