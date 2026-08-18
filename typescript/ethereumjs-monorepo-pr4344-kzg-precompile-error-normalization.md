# [블록체인 OSS 기여] ethereumjs/ethereumjs-monorepo - EVM precompile들이 Error가 아닌 값을 던지면 잘못 처리되는 문제 수정 (PR #4344) — 🟢 Merged

> 오픈소스 기여 기록 및 학습 노트. 이 문서는 PR #4344 하나만 다룬다. **2026-07-21, `gabrocheleau`가 직접 머지.**
>
> 이 PR은 두 단계로 진행했다. 처음엔 KZG point-evaluation precompile 하나(`0a-kzg-point-evaluation.ts`)만 직접 고쳤고, 그 뒤 "같은 패턴이 여기저기 더 있는데 이것도 다 고칠까?"라는 생각이 들어서 전체 인벤토리를 조사한 뒤 나머지 bn254/bls12 precompile 10개와 테스트 모킹 버그까지 AI 도구의 도움을 받아 확장했다. 확장 작업에서 "9개 다 고쳤다"고 스스로 확인했던 게 실제로는 `0f-bls12-pairing.ts` 하나를 빼먹은 상태였다는 걸, 나중에 diff를 다시 훑어보다가 직접 잡아냈다(4.7절) — "다 됐다"는 확인도 재검증이 필요하다는 걸 이번에 체감했다.

---

## 1. 개요

### 1.1 프로젝트/이슈 소개

- **저장소**: https://github.com/ethereumjs/ethereumjs-monorepo (PR #4343과 동일 — [[ethereumjs-monorepo-pr4343-step-event-oog-consistency]] 참고)
- **건드린 파일**: `packages/evm/src/precompiles/0a-kzg-point-evaluation.ts` — EIP-4844(블롭 트랜잭션)의 "포인트 평가(point evaluation)" precompile. KZG(Kate-Zaverucha-Goldberg) 커밋먼트 증명을 검증하는 로직.
- **이슈**: https://github.com/ethereumjs/ethereumjs-monorepo/issues/3687 — "Monorepo: Suggestion for a better (and any-less) Error Handling". **모노레포 전체**에 퍼져있는 `catch (e: any)` 패턴을 개선하자는 **광범위한 제안형 이슈**. 라벨: `type: refactor`, `eval: needs discussion`, `effort: E2 days`, `type: enhancement`, `package: monorepo`.

### 1.2 이슈 #3687의 성격 — PR #4343과 다른 점

PR #4343(이슈 #4313)은 **구체적이고 좁은 버그 리포트**였던 반면, 이번 이슈 #3687은:
- **모노레포 전체**에 걸친 문제 제기 — 특정 파일이 아니라 "이 패턴이 코드베이스 여기저기에 있다"는 관찰
- `eval: needs discussion` 라벨 — 메인테이너 팀이 아직 정확한 해결 방식을 확정하지 않은, **논의 중인 제안**이라는 뜻
- 이슈 작성자(메인테이너로 보임) 본인이 이슈 본문에 **두 가지 구체적 해결 방향**을 제시함:
  - **프로덕션 코드**: `any` 타입을 없애고, MDN 문서가 제안하는 정규화 패턴(`if (!(e instanceof Error)) { e = new Error(e) }`)을 적용
  - **테스트 코드**: 그냥 `Error`로 타입만 달아주면 충분
  - 그리고 스스로 이렇게 덧붙임: *"Might be worth to do this on a per-library basis (or 2-3 libraries) to keep things under control since this work might break some test cases along the line."* — **파일/라이브러리 단위로 나눠서 하는 걸 미리 승인**해놓은 상태.

이번 PR은 이 승인된 방식대로, **`packages/evm`의 precompile 중 하나만 골라서** 시범적으로 적용한 것.

### 1.3 관련 링크

| 구분 | 링크 |
|---|---|
| 원본 저장소 | https://github.com/ethereumjs/ethereumjs-monorepo |
| 내 Fork | https://github.com/GiHoon1123/ethereumjs-monorepo |
| 이슈 (#3687) | https://github.com/ethereumjs/ethereumjs-monorepo/issues/3687 |
| PR (#4344) | https://github.com/ethereumjs/ethereumjs-monorepo/pull/4344 |
| 작업 브랜치 | `fix-precompile-kzg-error-normalization` (upstream/master 기준 새로 분리 — 3.1절) |
| 커밋 | `f61a46f2f4db6efc52b03cde9815b5eafe2dba56` (2026-07-09 15:29 KST) |
| PR 생성 시각 | 2026-07-09 06:29:55 UTC (15:29 KST) |
| 상태 | 🟢 Merged |

### 1.4 한 줄 요약

> KZG point-evaluation precompile의 `catch (err: any)` 블록이 `err.message`가 항상 문자열로 존재한다고 가정하고 있었는데, 이건 JS에서 보장되지 않는 가정임. 실제로 내부 KZG 라이브러리가 `Error`가 아닌 값(예: 그냥 문자열)을 던지면, **catch 블록 안에서 또 다른 예외가 터져서** 정상적으로 revert 결과를 리턴해야 할 자리에서 처리되지 않은 크래시가 발생함. `err`를 항상 `Error`로 정규화하도록 고침 — 단순 타입 정리가 아니라 **실제 크래시를 막는 수정.** 이후 **같은 패키지의 나머지 bn254/bls12 precompile 10개도 동일한 패턴으로 확장**(1차 시도에서 그중 1개 `0f-bls12-pairing.ts`를 빼먹었다가 재검토로 잡아내서 추가 수정, 4.7절) — 최종적으로 이 PR은 `packages/evm/src/precompiles/` 안의 `any`-typed catch **11개(1+10)를 전부 정리함.**

### 1.5 스코프가 커진 과정 — "이슈 밖의 걸 고치고 있는 거 아닌가?"

9개 precompile을 추가로 고치던 도중, 파일 개수가 갑자기 늘어난 걸 보고 **"이거 이슈 범위 밖의 걸 고치고 있는 거 아니냐"고 스스로 다시 확인**했다. 확인 결과:

- **9개 precompile**: 이슈 #3687 본문이 **정확히 이 중 하나(`07-bn254-mul.ts`)를 예시로 직접 링크**하고 있어서, 벗어난 게 아니라 이슈가 지목한 패턴 그 자체였음.
- **테스트 모킹 타입 버그 수정**(`0a-pointevaluation.spec.ts`): 이건 이슈 #3687과 무관하지만, **"남의 버그"가 아니라 "이 PR에서 이미 만들어 넣은 내 실수"**라서 범위 이탈이 아니라고 판단함 (5.7절에서 자세히 다룸) — 새로운 스코프를 추가하는 게 아니라, 이 PR 안에서 이미 저지른 결함을 마무리하는 것.

이 확인 과정 자체를 8.2절 회고에 남겨둔다 — "파일 개수가 갑자기 늘어나면 불안해지는 게 정상이고, 그때마다 각 파일이 정확히 무엇 때문에 들어왔는지 한 번씩 짚어보는 습관"이 실제로 유효했던 사례.

---

## 2. 배경 지식

### 2.1 KZG point-evaluation precompile이 뭐 하는 애인가

EIP-4844(블롭 트랜잭션, "proto-danksharding")가 도입한 precompile(0x0a 주소). 블롭 데이터에 대한 KZG 커밋먼트가 특정 지점(z)에서 특정 값(y)으로 평가된다는 걸 증명하는 암호학적 증명(proof)을 검증함. 내부적으로 `micro-eth-signer` 같은 외부 KZG 라이브러리의 `verifyProof()` 함수를 호출해서 실제 암호학 연산을 수행함.

### 2.2 `catch (e: any)`가 왜 위험한가 — 이슈 #3687의 핵심 주장

TypeScript의 `catch` 블록에 잡히는 값(`e`)은, **JS 언어 자체가 `throw` 뒤에 무슨 값이든 올 수 있게 허용하기 때문에** 원래는 타입을 알 수 없는 값(`unknown`)이어야 함. 그런데 관습적으로 다들 `catch (e: any)`라고 써버리는 경우가 많음 — `any`는 TypeScript에게 "이 값에 대해서는 타입 체크를 하지 마라"는 뜻이라, `e.message`처럼 실제로 존재 안 할 수도 있는 속성에 접근해도 컴파일러가 아무 경고도 안 해줌. 결과: 라이브러리가 실수로(또는 의도적으로) `Error` 객체가 아닌 걸 던지면, 이 코드는 **런타임에 조용히 크래시**함.

### 2.3 이번에 실제로 재현한 크래시

```ts
} catch (err: any) {
  if (((err.message.includes('C_KZG_BADARGS') === true) === true) === true) {
    ...
```
`err`가 `Error` 인스턴스가 아니라 그냥 문자열(예: `throw 'boom'`)이면, `err.message`는 `undefined`. `undefined.includes(...)`를 호출하는 순간 **`TypeError: Cannot read properties of undefined (reading 'includes')`**가 새로 발생함. 이 새 에러는 원래의 `catch` 블록이 처리하려던 로직(정상적으로 revert 결과를 리턴하는 것) 자체를 대체해버려서, **의도한 대로 revert되는 게 아니라 처리되지 않은 예외로 EVM 실행이 죽음.**

### 2.4 `unknown` vs `any` — TypeScript가 catch 변수에 대해 원래 하려던 것

TypeScript는 이런 문제 때문에 `useUnknownInCatchVariables`라는 컴파일러 옵션(TS 4.4+, 최신 버전에서는 기본 활성화)을 통해 `catch` 변수의 기본 타입을 `any`가 아니라 **`unknown`**으로 만들 수 있게 함. `unknown`은 `any`와 달리 **아무 속성에도 바로 접근할 수 없고, 먼저 타입을 좁혀야(narrowing) 함** — 그래서 `err.message`처럼 무작정 접근하는 코드는 `unknown` 타입에서는 컴파일 자체가 안 되고, 개발자가 `err instanceof Error` 같은 체크를 반드시 거치도록 강제됨. 이번 fix가 `catch (err: any)` → `catch (err: unknown)`으로 바꾼 것도 이 원리를 그대로 적용한 것.

### 2.5 에러 정규화(normalization) 패턴

"어떤 값이든 일단 `Error` 인스턴스로 통일시켜놓고 그 다음부터는 항상 `.message`에 안전하게 접근할 수 있게 만드는" 패턴. MDN이 제안하는 가장 단순한 형태:
```js
try { ... } catch (e) {
  if (!(e instanceof Error)) {
    e = new Error(e)
  }
  console.error(e.message)
}
```
이번 fix는 이 아이디어를 살리되, 한 단계 더 세심하게 짬 — 4번 섹션에서 다룸.

---

## 3. 사전 작업 — 브랜치 분리

### 3.1 왜 새 브랜치가 필요했나

이 수정 작업을 시작한 시점에, 로컬 작업 디렉토리는 **이미 PR #4343(이슈 #4313, `fix-dynamic-gas-oog-step-event` 브랜치)이 push된 상태**였고, 이번 수정(KZG precompile)은 **그 브랜치에 커밋되지 않은 워킹트리 변경사항**으로 얹혀 있었음. 이대로 커밋하면 서로 무관한 두 fix가 같은 PR에 섞여 들어가서, "한 이슈, 한 PR" 원칙이 깨짐.

**해결**:
```bash
git stash push -u -- packages/evm/src/precompiles/0a-kzg-point-evaluation.ts packages/evm/test/precompiles/0a-pointevaluation.spec.ts
git checkout -b fix-precompile-kzg-error-normalization upstream/master   # PR #4343의 커밋을 포함하지 않는 깨끗한 지점에서 새 브랜치 생성
git stash pop
```
`upstream/master`(당시 `cb0d09451`)에서 새 브랜치를 따서, PR #4343의 커밋(`a161bd5ae`)이 아예 포함되지 않은 깨끗한 브랜치를 만들고 그 위에 이번 변경사항만 얹음.

---

## 4. 수정 내용

이 부분은 직접 구현했다.

### 4.1 실제 diff

```diff
-  } catch (err: any) {
-    if (((err.message.includes('C_KZG_BADARGS') === true) === true) === true) {
+  } catch (err: unknown) {
+    const error =
+      err instanceof Error
+        ? err
+        : typeof err === 'object' && err !== null && 'message' in err
+          ? new Error(String((err as { message?: unknown }).message))
+          : new Error(String(err))
+    if (error.message.includes('C_KZG_BADARGS') === true) {
       if (opts._debug !== undefined) {
         opts._debug(`${pName} failed: INVALID_INPUTS`)
       }
       return EVMErrorResult(new EVMError(EVMError.errorMessages.INVALID_INPUTS), opts.gasLimit)
     }
     if (opts._debug !== undefined) {
-      opts._debug(`${pName} failed: Unknown error - ${err.message}`)
+      opts._debug(`${pName} failed: Unknown error - ${error.message}`)
     }
```

### 4.2 정규화 로직의 3단 분기

1. **`err instanceof Error`** — 이미 진짜 `Error`면 그대로 사용
2. **`typeof err === 'object' && err !== null && 'message' in err`** — `Error`는 아니지만 `.message` 속성이 있는 객체라면, 그 `.message`만 뽑아서 `new Error(String(...))`로 감쌈 (원본 메시지 문자열을 보존하려는 의도)
3. **그 외 전부** (문자열, 숫자, `null`, `.message` 없는 객체 등) — `new Error(String(err))`로 통째로 문자열화

`typeof null === 'object'`가 `true`라는 JS의 유명한 함정 때문에 `err !== null` 체크가 2번 분기에 명시적으로 들어가 있음 — 이게 없으면 `null`이 2번 분기로 잘못 들어가서 `null['message']` 접근 시도로 또 다른 크래시가 날 수 있었음. **이 체크가 정확히 들어가 있는지 리뷰 때 확인함.**

### 4.3 MDN의 단순 패턴과 다른 점

MDN 패턴(`e = new Error(e)`)은 `Error`가 아닌 값이면 무조건 그 값 전체를 `Error` 생성자에 넘겨서 문자열화함 — 객체를 넘기면 `String(객체)`가 실행되어 보통 `'[object Object]'`처럼 **정보가 손실되는 문자열**이 됨. 이번 fix는 "혹시 `.message`라는 필드가 있으면, 객체 전체가 아니라 그 필드의 값을 보존하자"는 한 단계 더 나간 처리 — `{ message: 'C_KZG_BADARGS: ...' }` 같은, `Error`는 아니지만 Error-like한 모양의 객체를 던지는 라이브러리에 대응하기 위함.

### 4.4 스코프 확장 — 나머지 `catch (e: any)` 전체 인벤토리

"evm 쪽에 이 패턴이 더 있는데 다 고칠까?"라는 생각이 들어서, `grep -rn "catch\s*(\s*e[a-zA-Z]*\s*:\s*any\s*)" packages/evm/src packages/vm/src`로 **`packages/evm`과 `packages/vm` 전체를 뒤져서 남은 걸 전부 목록으로 정리**했다:

| 파일 | `e`를 실제로 어떻게 쓰나 | 위험도 |
|---|---|---|
| `packages/vm/src/runBlock.ts` | 안 씀 — journal revert하고 그냥 재던지기 | 🟢 무위험 |
| `packages/vm/src/runTx.ts` | 안 씀 — 동일 | 🟢 무위험 |
| `packages/evm/src/evm.ts` (2곳) | `errorMessage = e`로 저장 → 나중에 `exceptionError`로 리턴. `_addToBalance()`를 직접 확인해보니 **`VALUE_OVERFLOW` 하나만 던지는 완전히 통제된 내부 함수**라 실제로는 안전 | 🟡 낮음 (다만 `evm.ts`는 패키지 전체에서 제일 핫한 파일이라 신중하게 나중에) |
| `packages/evm/src/precompiles/*.ts` (9개, bn254/bls12) | `${e.message}` 로그 + `EVMErrorResult(e, ...)`로 그대로 리턴 — **이슈 본문이 예시로 직접 링크한 파일(`07-bn254-mul.ts`) 포함** | 🟡 낮음~중간 |
| `packages/evm/src/interpreter.ts:355` | `e.errorType`, `e.error`로 **분기 로직**에 사용, opcode 실행 루프의 핵심 catch (PR #4343 리뷰 때 이미 한 번 마주친 코드) | 🔴 가장 delicate |

**정한 순서**: ① 9개 bn254/bls12 precompile을 이 PR(#4344)에 추가 (같은 패키지, 이미 unreviewed 상태) → ② `runBlock.ts`/`runTx.ts`는 다른 패키지(`packages/vm`)라 별도 PR로 → ③ `evm.ts`는 마지막에 신중하게. `interpreter.ts`는 가장 delicate하다는 이유로 이번엔 제외하기로 함.

### 4.5 9개 precompile 수정 (AI 도구 도움을 받아 구현)

9개 파일 다 구조가 거의 동일했음(`06-bn254-add`, `07-bn254-mul`, `08-bn254-pairing`, `0b-bls12-g1add`, `0c-bls12-g1msm`, `0d-bls12-g2add`, `0e-bls12-g2msm`, `10-bls12-map-fp-to-g1`, `11-bls12-map-fp2-to-g2`). 대표 예시(`07-bn254-mul.ts`):

```diff
-  } catch (e: any) {
-    if (opts._debug !== undefined) {
-      opts._debug(`${pName} failed: ${e.message}`)
-    }
-    return EVMErrorResult(e, opts.gasLimit)
-  }
+  } catch (e: unknown) {
+    const error = e instanceof EVMError ? e : new EVMError(EVMError.errorMessages.REVERT)
+    if (opts._debug !== undefined) {
+      opts._debug(`${pName} failed: ${e instanceof Error ? e.message : undefined}`)
+    }
+    return EVMErrorResult(error, opts.gasLimit)
+  }
```

**왜 `EVMErrorResult`에 그대로 넘기면 안전하지 않았는지 — 실제 라이브러리 코드까지 추적**: 이 9개 파일이 감싸는 함수(`bls.addG1`, `_bn254.mul` 등)의 내부(`packages/evm/src/precompiles/bn254/noble.ts`, `bls12_381/noble.ts`)를 직접 읽어봤다. `toG1Point()`/`toFp2Point()` 같은 내부 검증 함수는 실패 시 **명시적으로 `EVMError`를 던지지만**, 그 안에서 호출하는 `G1.assertValidity()` 같은 **외부 `@noble/curves` 라이브러리 자체의 검증 함수**가 실패하면 그건 **일반 `Error`**를 던짐 — 즉 같은 catch 블록에 도달하는 예외가 상황에 따라 `EVMError`일 때도 있고 순수 `Error`일 때도 있는 게 실제로 코드상 확인됨. 정규화 없이 `EVMErrorResult(e, ...)`에 그대로 넘기면, 후자의 경우 `exceptionError`가 `.error`/`.errorType` 필드 없는 반쪽짜리 객체가 되어, 이후 그 필드를 검사하는 코드가 조용히 `undefined`를 받게 됨.

**의도적으로 안 고친 것 — 디버그 로그의 `.message`**: `EVMError` 클래스는 `.message` 필드가 없음(`error`/`errorType`만 있고, native `Error`를 상속하지도 않음). 그래서 원래부터, 내부 검증 함수가 `EVMError`를 던지는 경로에서는 `${e.message}` 로그가 **항상 "undefined"만 찍혔던, 이미 있던 별개의 버그**였음. 이것도 같이 고칠지 고민했지만 — **"기존 버그는 새로운 이슈에서 고쳐야지 여기서 고치면 섞인다"**고 판단해서 그대로 유지하기로 했다. `e instanceof Error ? e.message : undefined`로 타입만 안전하게 만들고, 출력 내용 자체는 이전과 100% 동일하게 유지함 (5.6절에서 이 결정을 더 자세히 다룸).

### 4.6 부수적으로 발견한, 이 PR 자체의 타입 버그 — 클래스 인스턴스 스프레드의 함정

9개 파일을 고치고 `npx tsc --noEmit -p .`을 돌렸더니, **오늘 작업과 무관한 곳에서 에러가 남** — 바로 전에 이미 push했던 `0a-kzg-point-evaluation.ts`용 테스트(`0a-pointevaluation.spec.ts`)의 KZG 모킹 객체였음. 그때는 `npx vitest run`만 돌리고 `tsc --noEmit`을 따로 안 돌려서 놓쳤던 것.

**1차 시도 (틀림)**: 실제 KZG 인스턴스(`const kzg = new microEthKZG(trustedSetup)`)를 스프레드해서 `verifyProof`만 덮어씌우려 함:
```ts
kzg: {
  ...kzg,
  verifyProof: () => { throw 'boom' },
},
```
이게 여전히 타입 에러가 남 — 이유를 직접 실험으로 확인함:
```ts
class Foo {
  a(): string { return 'a' }
  b(): number { return 1 }
}
const foo = new Foo()
const bar = { ...foo, a: () => 'override' }
bar.b()   // TS2339: Property 'b' does not exist on type '{ a: () => string; }'
```
**클래스 인스턴스를 스프레드(`{...instance}`)하면, 그 클래스의 메서드는 전혀 복사가 안 됨** — 이유는 클래스 메서드가 인스턴스 자신의 속성이 아니라 `프로토타입`에 있기 때문. 스프레드/`Object.assign`류의 연산은 **"자기 자신의 열거 가능한 속성"만** 복사하는데, 메서드는 거기 안 낌. 이건 TypeScript 타입 시스템만의 문제가 아니라 **실제 런타임에서도 그대로 벌어지는 일** — `{...kzg}`를 실제로 실행해봤어도 메서드 없는 빈 객체나 다름없었을 것.

**2차 시도 (해결)**:
```ts
kzg: Object.assign(Object.create(kzg), {
  verifyProof: () => { throw 'boom' },
}),
```
`Object.create(kzg)`는 새 객체를 만들면서 **그 객체의 프로토타입 체인에 `kzg` 인스턴스 자체를 넣어줌** — 그러면 새 객체에서 `.verifyProof` 말고 다른 메서드를 호출하면, 자기 자신엔 없으니 프로토타입 체인을 따라 `kzg` 쪽으로 넘어가서 실제 메서드를 찾아 씀. 그 위에 `Object.assign(..., { verifyProof: ... })`로 원하는 메서드만 자기 자신의 속성으로 덮어씀. 타입 체크도 통과(`Object.create`의 반환 타입이 `any`라 이후 어떤 타입에 할당해도 에러가 안 남)하고, 실제 동작도 정확함(오버라이드 안 한 메서드는 진짜 KZG 구현으로 정상 위임됨).

### 4.7 직접 잡아낸 누락 — `0f-bls12-pairing.ts`를 완전히 빼먹음

9개 precompile을 다 고치고 커밋/푸쉬/PR 본문 업데이트까지 마친 뒤, **diff를 다시 훑어보다가 `0f-bls12-pairing.ts`가 여전히 예전 코드 그대로라는 걸 발견**했다. 4.4절에서 조사했던 인벤토리 grep 결과를 다시 실행해보니, 이번엔 정확히 `0f-bls12-pairing.ts:71`도 잡혔음 — 왜 처음엔 이 파일이 목록에 안 잡혔는지 정확한 원인은 못 찾았지만(터미널 출력을 훑어보다 놓쳤을 가능성이 높음), **"9개를 다 고쳤다"는 확인 자체가 틀렸었다는 사실**은 명확했다.

**재검증 절차** (코드를 고치더라도 바로 커밋하지 않고, 검토 먼저 마친 뒤 커밋하기로 함):
1. `packages/evm/src`, `packages/vm/src`뿐 아니라 **`test/` 디렉토리까지 포함해서** `catch (e: any)` 전체를 다시 정밀 스캔 — 이번엔 `packages/vm/test/` 안에도 수십 개가 더 있다는 걸 확인했지만, 그건 **이슈 #3687 자체가 "테스트 코드는 그냥 `Error`로 타입만 달면 충분하다"고 별도로 낮은 우선순위를 준 영역**이라 지금 스코프(프로덕션 코드, bn254/bls12 precompile)에는 포함하지 않기로 함 — 새로운 스코프 확장이 아니라 오히려 스코프를 더 명확히 좁히는 결정.
2. `0f-bls12-pairing.ts`도 같은 라인에서 **`getPrecompileName('11')`**이라는, `0f`인데 `'11'`(다른 precompile 번호)을 넘기는 **완전히 별개의 pre-existing 버그**를 발견함 — 디버그 로그에 항상 잘못된 precompile 이름이 찍히는 문제. 이슈 #3687과 무관해서 이번에도 손 안 대고 발견 사실만 남김.
3. `catch (e: any)` → 나머지 9개와 동일한 패턴으로 수정.

### 4.8 "테스트가 없다"는 것도 실제로 맞았음 — 직접 재현 스크립트로 확인

`0f-bls12-pairing.ts`를 고치는 경로를 검증하는 테스트가 아직 없다는 것도 확인이 필요했다. 확인 방법: 기존 공유 테스트(`eip-2537-bls.spec.ts`)가 실제로 `0f`의 `catch` 블록을 타는지, 그리고 이 fix가 막으려는 정확한 상황(라이브러리가 `EVMError`가 아닌 값을 던지는 것)을 검증하는지를 **직접 실행 가능한 재현 스크립트**로 확인했다 (`npx tsx`로 실제 `precompile0f`를 호출, 잠깐 `console.error`를 임시로 넣어서 어느 분기를 타는지 추적).

**결과**: `catch` 블록 자체는 실제로 타고 있었음(`bls.pairingCheck`가 "not on curve" 입력에 대해 정말로 예외를 던짐) — 하지만 **MCL 라이브러리가 이 경우 이미 `EVMError`를 직접 던져줌**(`EVMError { error: 'point not on curve' }`). 그러니까 fix 전 코드(`EVMErrorResult(e, ...)`를 그대로 씀)도 **이 특정 케이스에서는 우연히 문제없이 동작**했던 것 — 막으려는 "라이브러리가 EVMError가 아닌 걸 던지는 경우"는 기존 테스트 스위트의 22개 fail-case 어디에서도 실제로 발생하지 않았다. 즉 **"324개 테스트가 다 통과한다"는 이전 확인이, 이 fix가 실제로 필요하다는 증거는 전혀 아니었다.**

**새 테스트 작성**(`test/precompiles/0f-bls12-pairing.spec.ts`, 새 파일 — `0f`는 전용 spec 파일이 원래 없었음):
- `createEVM({ common, bls })`가 커스텀 `bls` 구현을 주입받을 수 있다는 걸 발견 (KZG의 `customCrypto.kzg`와 같은 포지션의 주입 지점).
- 진짜 `MCLBLS` 인스턴스를 만들고, 4.6절에서 배운 `Object.create` 델리게이션 패턴으로 `pairingCheck`만 일반 `Error`를 던지도록 오버라이드.
- 구조적으로 유효한(길이/제로바이트 사전 체크를 통과하는) 실제 pairing 테스트 벡터의 입력값을 재사용 — 내용 자체는 안 중요함(`pairingCheck`를 무조건 던지게 만들었으니), 사전 체크만 넘기면 됨.
- **stash A/B로 재확인**: fix 없이 돌리면 `res.exceptionError?.error`가 `undefined`(깨진 상태) — `expected undefined to equal 'revert'`로 정확히 실패. fix 있으면 통과.

**PR 본문에도 정직하게 반영**: "9개 파일도 324개 테스트로 다 증명됐다"고 단정하지 않고, **"0f는 확인했고 실제로 갭이 있었다, 나머지 9개는 개별적으로 재검증 안 했다"**고 명시했다 — 확인 안 한 걸 확인한 것처럼 쓰지 않는다는 원칙을 여기서도 적용했다.

---

## 5. 코드 리뷰

### 5.1 검증 1 — 실제로 크래시가 재현되는지 (stash 기반 A/B)

fix 없이 새 테스트("should normalize unexpected thrown values to revert", KZG mock이 `throw 'boom'`을 던지도록 구성)를 돌려봤다:
```
TypeError: Cannot read properties of undefined (reading 'includes')
 ❯ precompile0a src/precompiles/0a-kzg-point-evaluation.ts:67:23
```
**이론이 아니라 실제로 크래시가 재현됨** — 2.3절에서 설명한 시나리오가 정확히 이 지점에서 터짐. fix를 다시 적용하니 통과.

### 5.2 검증 2 — 새 테스트 2개 중 정확히 몇 개가 이 fix를 "증명"하는지

두 번째 새 테스트("should map object errors with C_KZG_BADARGS message to invalid inputs", KZG mock이 `throw { message: 'C_KZG_BADARGS: ...' }`를 던지도록 구성)도 같은 방식(fix만 stash로 빼고 재실행)으로 확인했다 — **이 테스트는 fix 없이도 통과함.** 이유: 이 케이스는 던져진 객체가 이미 `.message`라는 진짜 문자열 속성을 갖고 있어서, 원래의 나이브한 `err.message.includes(...)` 코드로도 우연히 잘 동작하기 때문. 즉 이 테스트는 **정당한 커버리지지만, "이 fix가 필요하다"는 증거는 아님** — 오직 첫 번째 테스트("boom" 문자열을 던지는 것)만이 실제로 fix의 유무를 가르는 테스트임.

이 구분을 PR 본문에 정직하게 적어둠 (6.3절) — [[feedback_honest_pr_checklists]]와 같은 원칙.

### 5.3 검증 3 — 관련 테스트 + 린트

- `packages/evm/test/precompiles/0a-pointevaluation.spec.ts`: **3/3 통과** (기존 1개 + 신규 2개)
- `npx @biomejs/biome check` — 클린
- 커밋 시 pre-commit hook(biome `--write`), push 시 pre-push hook(eslint) 둘 다 클린 통과

### 5.4 부수적으로 발견한 것 — 삼중 `=== true` 비교

원본 코드에 `((err.message.includes('C_KZG_BADARGS') === true) === true) === true`라는, `=== true`를 세 번 겹쳐 쓴 이상한 표현이 있었음. `A`가 불리언이면 `A === true`를 몇 번 반복해도 항상 같은 값이 나오므로(불리언에 대한 항등 비교는 idempotent), 이걸 `error.message.includes(...) === true`로 단순화한 건 **완전히 동작 동일, 순수 스타일 정리**임을 직접 검증했다. 이 줄 자체가 `err.message` → `error.message`로 바뀌면서 어차피 손대야 하는 줄이라, 별도 PR로 분리할 만큼 크지 않다고 판단 — 다만 PR 본문에 언급은 해둠 (숨기지 않는 원칙).

### 5.5 이슈 라벨(`eval: needs discussion`)에 대한 프로세스 판단

이 라벨이 붙어 있어서 "PR 열기 전에 이슈에 댓글부터 남길지" 고민했다. 최종 판단:
- 이슈 본문 자체가 이미 "per-library로 나눠서 해도 된다"고 사전 승인해놓은 상태라, 별도로 허락을 구할 필요성이 크지 않음
- 이 저장소는 이미 외부 기여자에 대한 리뷰 회전이 느리다는 걸 앞서(#4313 조사 때) 확인했는데, "댓글 남기고 답 기다렸다가 PR" 순서로 가면 왕복이 하나 더 생겨서 오히려 더 느려짐
- 코드가 이미 완성/검증된 상태라, "구현 전에 방향을 물어봐야 하는" 상황(예: 이전에 있었던 nestjs/terminus #2667 케이스)과는 다름
- **결론**: 별도 댓글 없이, PR 본문 자체에 "왜 이 파일만 골랐는지, 왜 이슈의 단순 패턴보다 조금 더 세심하게 했는지, 이 방향이 맞으면 다른 파일도 이어서 하겠다"는 내용을 다 담아서 PR 하나로 제출 — PR 자체가 반응을 이끌어내기 쉬운, 구체적인 결과물이기 때문.

---

## 6. PR 본문

### 6.1 구조와 의도

- **Summary**: 스코프(이슈 #3687 중 이 파일 하나만) 먼저 명시 + **"단순 타입 정리가 아니라 실제 크래시를 막는 fix"라는 핵심 근거를 첫 문단에 배치** — 제목만 보고 "그냥 잡일이구나"하고 지나칠까봐 가장 중요한 정보를 앞에 둠.
- **Fix**: 뭘 바꿨는지 + 왜 이슈의 단순 패턴보다 더 세심하게 했는지 설명하면서 "과하면 단순화해도 된다"고 여지를 열어둠. 삼중 비교문 정리도 숨기지 않고 언급.
- **Test plan**: 테스트 2개 중 1개만 실제로 fix를 증명한다는 걸 그대로 밝힘 (5.2절 내용을 정직하게 반영) — "둘 다 증거"라고 부풀리지 않음.
- **마지막 문장**: "이 방식이 맞으면 다른 precompile/패키지에도 이어서 적용해보겠다, 피드백 달라"로 마무리 — 별도 이슈 댓글 없이도 방향 확인을 요청하는 효과.

### 6.2 "Fixes" 대신 "Addresses"를 쓴 이유

이슈 #3687은 모노레포 전체를 대상으로 하는 광범위한 이슈라, 이 PR 하나로 이슈 자체가 완전히 해결되는 게 아님. "Fixes #3687"이라고 쓰면 머지 시 이슈가 자동으로 닫혀버리는데, 실제로는 다른 수많은 파일들이 아직 그대로 남아있는 상태이므로 부정확함. "Addresses #3687 for one precompile"로 정확한 스코프를 명시함 — PR #3325(node-redis)에서 "Fixes" 대신 "Related to"를 썼던 것과 같은 원칙.

### 6.3 AI 언급 여부 & DCO

PR #4343과 같은 저장소라 동일하게 **"언급 안 함"**을 적용. 커밋 메시지 + PR 본문을 AI 도구 관련 키워드로 grep해서 확인함 — 아무것도 안 걸림. DCO/CLA 요구사항도 이 저장소 전체에 적용되는 게 없다는 걸 PR #4343 작업 때 이미 확인해둔 상태라 재확인 안 함.

---

## 7. 기여 프로세스

### 7.1 브랜치

```bash
gh repo fork ethereumjs/ethereumjs-monorepo   # 이미 PR #4343에서 fork 완료된 상태, 재사용
git checkout -b fix-precompile-kzg-error-normalization upstream/master
```

### 7.2 커밋 메시지

```
fix(evm): normalize non-Error throws in KZG point-evaluation precompile

catch (err: any) assumed err always has a string .message, which isn't
guaranteed in JS -- if the underlying KZG library throws something
that isn't an Error (e.g. a plain string), accessing err.message
crashes inside the catch block itself instead of returning the
intended revert result.

Normalize err to an Error before use: keep it as-is if it's already
an Error, wrap its .message if it's an object with one, otherwise
fall back to String(err). Addresses #3687 for this one precompile.
```

---

## 8. 결과 및 회고

### 8.1 현재 상태 (2026-07-21 기준) — Merged

- PR #4344 **Merged** (2026-07-21T18:08:30Z), 메인테이너 `gabrocheleau`가 직접 머지. 최종 커밋 5개(리뷰 반영 커밋 1개가 4번 항목 이후 추가로 붙음).
  1. `f61a46f2f` — KZG precompile 정규화 (직접 구현)
  2. `99f857ba1` — KZG 테스트 모킹의 사전 타입 버그 수정 (AI 도움 받아 구현)
  3. `a34ca7eb3` — bn254/bls12 9개 정규화 (AI 도움 받아 구현) — **당시엔 10개 중 9개, `0f` 누락 상태**
  4. `66aab07e8` — 누락됐던 `0f-bls12-pairing.ts` 정규화 + 전용 테스트 신규 작성 (AI 도움 받아 구현, 누락은 직접 잡아냄)
  5. 리뷰 코멘트 반영 커밋 1개 (머지 전 추가)
- 별도 코멘트나 요청 없이, PR 본문에 미리 적어둔 스코프/근거 설명 그대로 승인·머지됨 — 같은 날 열린 PR #4343(아래 별도 문서)이 리뷰 코멘트를 거쳐 다른 PR로 대체된 것과 대조적으로, 이 PR은 그대로 통과.
- 같은 저장소에 동시에 열려 있던 두 번째 PR(#4343)과 결과가 갈린 사례 — 이슈 성격 차이(스펙이 명확한 좁은 버그 vs 메인테이너가 방향을 다시 정리하고 싶어한 이슈)가 실제로 결과에 반영된 것으로 보인다.

### 8.2 지금까지 배운 점

- **"이슈가 넓고 모호할 때"와 "이슈가 좁고 구체적일 때" PR을 쓰는 전략이 달라야 한다** — #4313(좁고 구체적, 메인테이너가 원인까지 파악해둔 상태)와 #3687(넓고 `eval: needs discussion`)은 같은 저장소, 비슷한 파일(둘 다 `packages/evm`)인데도 접근을 다르게 가야 했음. 넓은 이슈에서는 "왜 이 작은 조각만 골랐는지, 왜 이 방식으로 했는지"를 훨씬 더 적극적으로 설명해야 리뷰어가 스코프를 오해하지 않음.
- **테스트가 2개 있다고 해서 둘 다 같은 무게의 증거는 아니다** — stash A/B 비교를 이번에도 각 테스트마다 개별로 실행해봐야, 어느 게 진짜 fix를 검증하는지 정확히 알 수 있었다. 눈으로 코드만 보고 "테스트 2개 다 통과하네, 됐다"고 넘겼으면 놓쳤을 구분.
- **"any 타입 정리"라는 이슈의 프레이밍을 그대로 따라가지 않고, "실제로 이게 왜 위험한지"(진짜 크래시 재현)를 직접 검증해서 PR의 핵심 근거로 삼은 것**이 이번 작업에서 가장 가치있었던 부분 — 스타일 이슈처럼 보이는 티켓도, 파고들면 진짜 런타임 버그가 숨어있을 수 있다는 걸 실전에서 확인했다.
- 같은 리포지토리 안에서 서로 무관한 두 fix를 작업할 때, **브랜치를 반드시 공통 조상(이 경우 `upstream/master`)에서부터 새로 따야 한다**는 걸 실전에서 직접 처리함 (3.1절) — 그냥 지금 브랜치에서 새 브랜치를 따면 이전 PR의 커밋이 그대로 섞여 들어간다는 흔한 함정.
- **가장 뼈아팠던(그리고 가장 값진) 교훈 — "9개 파일을 다 고쳤다"는 확인 자체가 틀렸었다.** grep으로 인벤토리를 뽑고 직접 9개를 고치고 "다 됐다"고 스스로 확인했는데, 실제로는 `0f-bls12-pairing.ts`를 처음부터 빼먹고 있었음 — 그리고 그걸 잡아낸 건 diff를 다시 훑어본 것. **"인벤토리 다 훑었다"는 확인도 반드시 재검증 대상이라는 것**을 이번에 실전으로 체감했다. 다행히 재검증 과정(전체 재스캔 → 재현 스크립트로 실제 동작 확인 → 새 테스트 작성)을 거치면서, "테스트가 없다"는 두 번째 걱정까지도 사실이었다는 걸 추가로 확인했다 — 한 번의 재검토가 두 개의 진짜 문제(파일 누락 + 테스트 공백)를 동시에 드러낸 셈.
- **"324개 테스트 통과"가 항상 "이 fix가 필요하다는 증거"는 아니다** — 기존 테스트가 catch 블록을 타긴 해도, 그 안에서 실제로 벌어지는 값(이번엔 MCL이 이미 EVMError를 던져줌)에 따라 fix가 하는 일이 전혀 검증되지 않을 수 있다. 코드를 읽는 것만으로는 부족했고, **직접 실행 가능한 재현 스크립트로 실제 분기를 추적**해야 확실히 알 수 있었다.

### 8.3 실제로 어떻게 됐나

- [x] 메인테이너가 "per-library 접근"에 동의 — 별도 반론 없이 그대로 머지됨
- [ ] "나머지 9개는 개별 재검증 안 함"에 대한 메인테이너의 별도 언급은 없었음 (코멘트 내용 자체를 다 확인하지는 못함 — 리뷰 코멘트 반영 커밋 1개가 붙은 걸로 봐서 사소한 요청은 있었을 것으로 추정)
- [ ] `0f-bls12-pairing.ts`의 `getPrecompileName('11')`이 이름을 잘못 반환하는 버그는 이번 PR에서 다루지 않았고, 별도 이슈로도 아직 안 올림
- [x] PR #4343과 이 PR의 결과가 갈림 — 이 PR(#4344)은 그대로 머지, #4343은 리뷰 후 다른 PR(#4349)로 대체됨. 좁고 구체적인 이슈(#4344 근거인 #3687도 사실 넓은 이슈이긴 하지만 스코프를 이 PR에서 명확히 좁혀 제출한 것)와, 메인테이너가 "다르게 해보자"고 판단한 이슈의 차이가 실제로 결과에 반영된 사례
- [x] 최종 머지: 2026-07-21, PR 오픈(2026-07-09)부터 머지까지 약 12일
- [ ] 다른 precompile/패키지로 이어서 적용할지는 아직 별도로 안 정함
