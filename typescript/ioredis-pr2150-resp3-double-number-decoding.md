# [Redis OSS 기여] redis/ioredis - RESP3 double 디코딩이 JS `Number()`와 다르게 반올림되던 버그 수정 (PR #2150)

> 오픈소스 기여 기록 및 학습 노트. 이 문서는 PR #2150 하나만 다룬다. **최초 작성 2026-07-27, 같은 날 메인테이너 승인(APPROVED), 2026-08-06 머지까지 갱신. 13절 참고.**
>
> 이번 건은 Redis 프로토콜(RESP3)로 들어온 double 응답을 ioredis가 JavaScript `number`로 바꾸는 경로에서 발생한 correctness bug다. 문제 자체는 숫자 4개가 잘못 디코딩되는 작은 현상처럼 보이지만, 원인을 이해하려면 "프로토콜 wire format", "decoder", "부동소수점 문자열 변환", "네트워크 chunk boundary"를 같이 봐야 한다.

---

## 1. 개요

### 1.1 프로젝트 소개

- **저장소**: https://github.com/redis/ioredis
- **npm 패키지**: `ioredis`
- **성격**: Node.js용 Redis 클라이언트. Redis 공식 Node 클라이언트는 `redis/node-redis`지만, ioredis도 오래된 사용자층이 있고 실제 서비스에서 많이 쓰이는 Redis 클라이언트다.
- **건드린 부분**:
  - `lib/resp/decoder.ts`
  - `test/unit/resp/decoder.ts`

ioredis는 사용자 코드가 호출한 Redis 명령을 Redis wire protocol로 인코딩해서 서버에 보내고, Redis 서버가 보내는 응답을 다시 JavaScript 값으로 디코딩해준다. 이번 수정은 "보낼 때"가 아니라 **받을 때**, 그중에서도 RESP3 double 응답을 JS `number`로 바꾸는 부분을 고쳤다.

### 1.2 이슈 (#2149)

**제목**: `RESP3 double decoding diverges from Number(): max double becomes Infinity, sub-1e-17 values become 0`

리포터는 RESP3 decoder의 double 경로가 직접 숫자를 계산하는 방식을 보고, 실제 `Decoder` 클래스에 wire frame을 넣어 결과를 비교했다.

문제가 된 값들:

```text
RESP3 frame                       기존 decoder 결과              기대값
--------------------------------------------------------------------------------
,1.7976931348623157e308\r\n        Infinity                       Number.MAX_VALUE
,5e-324\r\n                        0                              Number.MIN_VALUE
,0.000000000000000001\r\n          0                              1e-18
,0.30000000000000004\r\n           0.3000000000000001              0.30000000000000004
```

즉 Redis 서버가 정상적인 double 값을 보냈는데, ioredis가 JS 값으로 변환하는 과정에서 값이 달라진다.

### 1.3 관련 링크

| 구분 | 링크 |
|---|---|
| 원본 저장소 | https://github.com/redis/ioredis |
| 내 Fork | https://github.com/GiHoon1123/ioredis |
| 이슈 (#2149) | https://github.com/redis/ioredis/issues/2149 |
| PR (#2150) | https://github.com/redis/ioredis/pull/2150 |
| 작업 브랜치 | `fix-2149-resp3-double-decoding` |
| 커밋 | `1f91f7d5f4a53acb73191c653f6cde9b2e66c3c6` |
| 상태 | **MERGED** (2026-08-06) |

### 1.4 한 줄 요약

> RESP3 double decoder가 decimal digit을 직접 누적해서 JS number를 만들고 있었는데, 이 직접 구현이 JavaScript `Number()`의 문자열-to-double 변환과 다르게 반올림되어 경계값에서 틀린 결과를 냈다. double token을 기존 string decoder로 끝까지 읽은 뒤, RESP3 특수값(`inf`, `-inf`, `nan`)만 직접 처리하고 일반 숫자는 `Number(value)`에 맡기도록 수정했다.

---

## 2. 배경 지식

### 2.1 ioredis가 하는 일

사용자 입장에서는 Redis 명령을 이렇게 호출한다:

```ts
await redis.zscore("rank", "user1");
```

하지만 네트워크에 실제로 이 JavaScript 코드가 그대로 나가는 건 아니다. ioredis는 내부에서 다음 일을 한다:

```text
사용자 코드
  -> ioredis가 Redis 명령을 RESP로 인코딩
  -> Redis 서버로 전송
  -> Redis 서버가 RESP로 응답
  -> ioredis가 RESP 응답을 JS 값으로 디코딩
  -> 사용자 코드가 결과를 받음
```

이번 버그는 마지막 단계다:

```text
RESP 응답 -> JS 값
```

특히:

```text
RESP3 double -> JS number
```

변환이 틀렸다.

### 2.2 RESP는 무엇인가

RESP는 REdis Serialization Protocol의 약자다. Redis 클라이언트와 Redis 서버가 데이터를 주고받기 위해 사용하는 통신 규약이다.

Redis 명령:

```text
SET name kim
```

은 네트워크에서는 대략 이런 형태로 나간다:

```text
*3\r\n
$3\r\n
SET\r\n
$4\r\n
name\r\n
$3\r\n
kim\r\n
```

의미:

```text
*3    배열 원소 3개
$3    길이 3짜리 문자열
SET
$4    길이 4짜리 문자열
name
$3    길이 3짜리 문자열
kim
```

이렇게 길이와 타입을 명시하면 값에 공백, 줄바꿈, 빈 문자열, 바이너리 데이터가 있어도 애매하지 않다.

### 2.3 RESP3 double

RESP3는 RESP2보다 타입이 더 많다. 그중 double 타입은 `,`로 시작한다.

예:

```text
,1.5\r\n
```

의미:

```text
,      이 값은 double이다
1.5    숫자 문자열
\r\n   값 끝
```

ioredis는 이걸 읽고 사용자에게:

```ts
1.5
```

를 돌려줘야 한다.

특수값도 있다:

```text
,inf\r\n
,+inf\r\n
,-inf\r\n
,nan\r\n
```

이들은 각각 JS에서는:

```ts
Infinity
Infinity
-Infinity
NaN
```

으로 매핑되어야 한다.

### 2.4 왜 프로토콜 decoder는 지저분해 보이는가

일반 서비스 코드는 "우리 서비스에서 들어오는 값"만 다루는 경우가 많다. 하지만 라이브러리의 protocol decoder는 전제가 적다.

고려해야 하는 것:

```text
값이 한 chunk로 올 수도 있고, 여러 chunk로 잘려 올 수도 있음
RESP2/RESP3 mapping이 다름
Buffer로 받아야 하는 경로와 string/number로 받아야 하는 경로가 다름
숫자, 문자열, 배열, map, push frame, error frame을 모두 구분해야 함
Redis 서버 버전과 프로토콜 버전에 따라 응답 모양이 다를 수 있음
```

그래서 decoder에는 분기가 많다. 이번 수정에서도 이 점이 중요했다. 단순히 `Number()` 한 줄로 끝나는 게 아니라, 기존 string/Buffer mapping과 streaming continuation을 깨지 않도록 기존 decoder 구조에 맞춰 넣어야 했다.

### 2.5 JS `number`와 double

JavaScript의 `number`는 IEEE 754 double precision floating point다.

즉 JS에서:

```ts
Number("0.30000000000000004")
```

를 하면 JS 엔진이 표준 알고리즘으로 문자열을 double 값으로 변환한다.

문제는 이 변환을 직접 흉내 내기 어렵다는 점이다.

예를 들어:

```text
"0.30000000000000004"
```

를 보고:

```ts
0
  + 3 * 1e-1
  + 0 * 1e-2
  + 0 * 1e-3
  ...
  + 4 * 1e-17
```

처럼 직접 계산한다고 해서 항상 `Number("0.30000000000000004")`와 같은 결과가 나오지 않는다. 부동소수점 연산은 중간 계산마다 반올림이 들어가기 때문이다.

이번 버그는 정확히 이 지점에서 발생했다.

---

## 3. 원인 분석

### 3.1 수정 전 double decoder

수정 전 `lib/resp/decoder.ts`에는 double을 직접 파싱하는 코드가 있었다.

핵심 구조:

```text
1. 정수부를 digit 단위로 읽어서 integer를 만든다.
2. 소수점이 나오면 decimal digit을 multiplier table로 더한다.
3. exponent가 나오면 10 ** exponent를 곱한다.
4. 최종 number를 반환한다.
```

소수부 처리를 위해 이런 테이블이 있었다:

```ts
static #DOUBLE_DECIMAL_MULTIPLIERS = [
  1e-1, 1e-2, 1e-3, 1e-4, 1e-5, 1e-6, 1e-7, 1e-8, 1e-9,
  1e-10, 1e-11, 1e-12, 1e-13, 1e-14, 1e-15, 1e-16, 1e-17,
];
```

의도는 성능이었다. 문자열 전체를 만들고 `Number()`에 넘기지 않고, 바이트를 읽으면서 바로 숫자를 조립하려는 fast path다.

하지만 이 방식은 JS `Number()`와 같은 correctly-rounded decimal-to-double 변환이 아니다.

### 3.2 `Number.MAX_VALUE`가 `Infinity`가 된 이유

문제 frame:

```text
,1.7976931348623157e308\r\n
```

이 값은 JS에서:

```ts
Number.MAX_VALUE
```

다.

기존 decoder는 먼저 mantissa를 직접 만들었다:

```text
1.7976931348623157
```

그런 다음:

```ts
* 10 ** 308
```

을 적용했다.

문제는 mantissa를 직접 누적하는 과정에서 아주 미세하게 큰 값이 만들어질 수 있다는 점이다. `Number.MAX_VALUE`는 JS가 표현 가능한 가장 큰 finite number라서, 이 값보다 조금만 커져도 결과가:

```ts
Infinity
```

가 된다.

즉:

```text
정상: 1.7976931348623157e308
기존 decoder: 중간 계산이 약간 커짐 -> Infinity
```

### 3.3 `Number.MIN_VALUE`가 `0`이 된 이유

문제 frame:

```text
,5e-324\r\n
```

이 값은 JS에서:

```ts
Number.MIN_VALUE
```

다. JS가 표현 가능한 가장 작은 양수다.

기존 decoder는 decimal/exponent를 직접 계산했다. 이런 극단적으로 작은 값은 직접 곱셈 경로에서 언더플로우 처리가 JS `Number()`의 문자열 변환과 다르게 나올 수 있다.

결과적으로:

```text
기대값: 5e-324
기존 결과: 0
```

이 됐다.

### 3.4 `1e-18`이 `0`이 된 이유

문제 frame:

```text
,0.000000000000000001\r\n
```

이 값은:

```ts
1e-18
```

이다.

기존 decoder의 decimal multiplier table은 `1e-17`까지만 있었다. 그리고 코드에는 이런 guard가 있었다:

```ts
if (decimalIndex < Decoder.#DOUBLE_DECIMAL_MULTIPLIERS.length) {
  double +=
    (byte - ASCII["0"]) *
    Decoder.#DOUBLE_DECIMAL_MULTIPLIERS[decimalIndex++];
}
```

즉 소수점 아래 18번째 자리 이후의 digit은 계산에 반영되지 않는다.

`0.000000000000000001`은 의미 있는 `1`이 18번째 자리에 있으므로 기존 코드에서는 그 digit이 버려진다.

결과:

```text
기대값: 1e-18
기존 결과: 0
```

### 3.5 `0.30000000000000004`가 다르게 반올림된 이유

문제 frame:

```text
,0.30000000000000004\r\n
```

JS 표준 변환:

```ts
Number("0.30000000000000004")
// 0.30000000000000004
```

기존 decoder:

```ts
0.3000000000000001
```

이건 극단값 문제가 아니라 평범한 누적 오차다. digit마다 `digit * 10^-k`를 더하는 방식은 중간중간 부동소수점 반올림을 거치기 때문에 최종 결과가 JS 문자열 변환과 달라질 수 있다.

---

## 4. 수정 내용

### 4.1 수정 방향

수정 전:

```text
RESP3 double payload를 byte 단위로 직접 읽는다
-> integer/decimal/exponent를 직접 계산한다
-> JS number를 직접 만든다
```

수정 후:

```text
RESP3 double payload를 기존 string decoder로 끝까지 읽는다
-> inf/+inf/-inf/nan만 직접 처리한다
-> 일반 숫자는 Number(value)에 맡긴다
```

핵심은 "문자열 읽기"와 "숫자 변환"을 분리한 것이다.

문자열 읽기는 protocol decoder가 잘해야 한다:

```text
\r\n까지 정확히 읽기
chunk가 중간에 끊기면 continuation으로 이어 읽기
```

숫자 변환은 JS 엔진이 잘해야 한다:

```text
"1.7976931348623157e308" -> Number.MAX_VALUE
"5e-324" -> Number.MIN_VALUE
"0.30000000000000004" -> 0.30000000000000004
```

### 4.2 수정된 코드

```ts
#decodeDouble(type, chunk) {
  // RESP2 sent doubles as bulk strings, so the legacy mapping decodes the
  // raw textual form (Buffer or String) and lets `replyEncoding` decide the
  // final shape per command. resp3 mapping leaves `type` unset -> JS number.
  if (type === String || type === Buffer) {
    return this.#decodeSimpleString(type, chunk);
  }

  const value = this.#decodeSimpleString(String, chunk);
  return typeof value === "function"
    ? this.#continueDecodeDouble.bind(this, value)
    : Decoder.#parseDouble(value);
}
```

먼저 기존 mapping을 유지한다:

```ts
if (type === String || type === Buffer) {
  return this.#decodeSimpleString(type, chunk);
}
```

이 부분이 중요하다. RESP2 compatible mapping이나 Buffer variant에서는 double을 number로 바꾸지 않고 기존처럼 raw textual form으로 줘야 한다. 이번 수정은 RESP3 native number mapping만 바꾼다.

일반 RESP3 double 경로는:

```ts
const value = this.#decodeSimpleString(String, chunk);
```

로 `\r\n` 전까지 문자열을 읽는다.

예:

```text
,0.30000000000000004\r\n
```

에서 type marker `,`는 이미 소비된 상태고, 여기서는:

```text
0.30000000000000004
```

를 읽는다.

### 4.3 streaming continuation 처리

네트워크 데이터는 항상 한 번에 오지 않는다.

예를 들어 하나의 frame이:

```text
,1.7976931348623157e308\r\n
```

이라도 실제로는 이렇게 나뉘어 들어올 수 있다:

```text
,1.797693
1348623157e308\r\n
```

기존 decoder는 이런 상황을 위해 "아직 다 못 읽었으면 function을 반환하고, 다음 chunk에서 이어 읽는" 구조를 가지고 있다.

그래서 수정 코드도 이 패턴을 따른다:

```ts
return typeof value === "function"
  ? this.#continueDecodeDouble.bind(this, value)
  : Decoder.#parseDouble(value);
```

`value`가 문자열이면 바로 parse한다. `value`가 함수면 아직 `\r\n`까지 못 읽은 것이므로 continuation으로 넘긴다.

이어지는 함수:

```ts
#continueDecodeDouble(cb, chunk) {
  const value = cb(chunk);
  return typeof value === "function"
    ? this.#continueDecodeDouble.bind(this, value)
    : Decoder.#parseDouble(value);
}
```

이 구조 덕분에 chunk boundary가 어디에 있든 같은 결과가 나온다.

### 4.4 특수값 처리

```ts
static #parseDouble(value) {
  switch (value) {
    case "inf":
    case "+inf":
      return Infinity;
    case "-inf":
      return -Infinity;
    case "nan":
      return NaN;
    default:
      return Number(value);
  }
}
```

왜 `inf`와 `nan`은 직접 처리하나?

RESP3 spelling은:

```text
inf
+inf
-inf
nan
```

이다.

그런데 JS `Number()`는:

```ts
Number("Infinity") // Infinity
Number("inf")      // NaN
```

이다.

즉 RESP3의 `inf` spelling은 JS `Number()`가 바로 이해하지 못한다. 그래서 이 값들은 직접 매핑한다.

일반 숫자 문자열은:

```ts
Number(value)
```

에 맡긴다.

---

## 5. 테스트

### 5.1 기존 decoder 테스트 구조

`test/unit/resp/decoder.ts`에는 이미 좋은 helper가 있었다.

각 테스트 케이스는 자동으로 세 가지 방식으로 실행된다:

```text
1. single chunk
2. byte by byte
3. split at every byte boundary
```

즉 하나의 frame에 대해:

```text
,1.7976931348623157e308\r\n
```

를 한 번에 넣어도 검사하고, 1바이트씩 넣어도 검사하고, 가능한 모든 split 위치에서 두 chunk로 나눠 넣어도 검사한다.

그래서 별도의 split test를 새로 만들 필요 없이 기존 `Double` 섹션에 케이스를 추가하는 게 가장 자연스러웠다.

### 5.2 추가한 테스트

```ts
test("Number.MAX_VALUE", {
  toWrite: Buffer.from(",1.7976931348623157e308\r\n"),
  replies: [Number.MAX_VALUE],
});

test("Number.MIN_VALUE", {
  toWrite: Buffer.from(",5e-324\r\n"),
  replies: [Number.MIN_VALUE],
});

test("sub-1e-17 decimal", {
  toWrite: Buffer.from(",0.000000000000000001\r\n"),
  replies: [1e-18],
});

test("correctly rounded decimal", {
  toWrite: Buffer.from(",0.30000000000000004\r\n"),
  replies: [0.30000000000000004],
});
```

이 4개는 이슈에서 지목한 실패값을 그대로 회귀 테스트로 만든 것이다.

각 케이스는 실제로 내부 helper에 의해:

```text
single chunk
byte by byte
split at every byte boundary
```

를 모두 돈다.

### 5.3 RED 확인

수정 전 테스트는 실패했다.

실패 형태:

```text
expected Infinity to equal 1.7976931348623157e+308
expected +0 to equal 5e-324
expected +0 to equal 1e-18
expected 0.3000000000000001 to equal 0.30000000000000004
```

즉 테스트가 실제 버그를 잡는다는 것을 확인했다.

---

## 6. 검증 기록

### 6.1 설치

```bash
npm install
```

결과:

```text
설치 완료
audit warning은 있었지만 이번 수정 범위와 무관
```

### 6.2 decoder target test

```bash
TS_NODE_TRANSPILE_ONLY=true NODE_ENV=test ./node_modules/.bin/mocha --no-experimental-strip-types test/unit/resp/decoder.ts
```

결과:

```text
189 passing
```

### 6.3 unit suite

```bash
TS_NODE_TRANSPILE_ONLY=true NODE_ENV=test ./node_modules/.bin/mocha --no-experimental-strip-types "test/helpers/*.ts" "test/unit/**/*.ts"
```

결과:

```text
372 passing
```

### 6.4 build

```bash
npm run build
```

결과:

```text
통과
```

### 6.5 lint

```bash
npm run lint
```

결과:

```text
0 errors
23 warnings
```

warning은 기존 파일의 member ordering / unused type parameter 관련 warning이고, 이번 수정 파일에서 새로 만든 error는 없다.

### 6.6 functional reply mapping

Docker compose로 test Redis를 띄우려고 했지만 로컬 `6379` 포트가 이미 사용 중이라 실패했다.

```text
Bind for 0.0.0.0:6379 failed: port is already allocated
```

실패한 compose 리소스는 정리했다.

이후 이미 떠 있는 로컬 Redis에 붙어서 RESP3 reply mapping functional test를 실행했다:

```bash
TS_NODE_TRANSPILE_ONLY=true NODE_ENV=test ./node_modules/.bin/mocha --no-experimental-strip-types test/helpers/*.ts test/functional/reply_mapping.ts
```

결과:

```text
9 passing
```

### 6.7 diff check

```bash
git diff --check
```

결과:

```text
통과
```

---

## 7. 성능 확인

### 7.1 왜 성능을 봤나

기존 코드는 주석으로 성능 의도가 드러나 있었다:

```text
Precalculated multipliers for decimal points to improve performance
```

즉 직접 digit을 읽어 number를 만드는 방식은 우연히 복잡했던 게 아니라, 문자열 할당과 `Number()` parsing을 피하려는 최적화였다.

하지만 그 최적화가 JS `Number()`와 다른 결과를 만들고 있었다.

그래서 PR 전에 간단한 local decoder loop로 `upstream/main`과 현재 브랜치를 비교했다. 정식 benchmark suite는 아니고, 리뷰 중 "성능은 확인했냐"는 질문에 답할 수 있을 정도의 sanity check다.

### 7.2 벤치 조건

```text
iterations: 2,000,000
rounds: 5
대상: Decoder.write(Buffer.from(frame))
비교: upstream/main vs 현재 fix 브랜치
```

### 7.3 결과

```text
frame                         upstream/main     current fix
----------------------------------------------------------------
,1.5                          0.182us/op        0.213us/op
,0.30000000000000004          0.340us/op        0.264us/op
,1.7976931348623157e308       0.447us/op        0.275us/op
,5e-324                       0.278us/op        0.233us/op
,0.000000000000000001         0.338us/op        0.238us/op
```

해석:

- 짧은 double literal인 `,1.5\r\n`는 기존 직접 parser가 더 빠르다.
- 이슈에서 문제가 된 boundary 값들은 이번 `Number()` 경로가 더 빠르다.
- 모든 측정값은 microsecond 이하 수준이다.

PR 본문에는 이걸 과하게 방어적으로 쓰지 않고, 짧게만 남겼다:

```text
Short literals like ,1.5\r\n were slightly slower, while the boundary cases above were faster with this change.
```

### 7.4 왜 이 방향을 택했나

라이브러리 decoder에서 성능은 중요하다. 하지만 parser가 숫자 값을 틀리게 만들면 더 큰 문제다.

이번 선택은:

```text
직접 만든 decimal-to-double 변환을 계속 유지
vs
JS 엔진의 Number() 변환에 맡김
```

중 후자를 택한 것이다.

이유:

```text
1. JS 사용자에게 돌려주는 값은 결국 JS number다.
2. 그러면 문자열-to-number 변환은 JS Number semantics와 일치해야 한다.
3. 직접 parser를 정확하게 고치려면 correctly-rounded decimal-to-binary conversion을 사실상 재구현해야 한다.
4. 그 복잡도까지 이 라이브러리의 RESP decoder가 떠안을 필요는 없다.
```

즉 단순히 "성능 최적화를 제거했다"가 아니라:

```text
값을 틀리게 만들던 별도 변환 경로를 제거하고, JS 표준 숫자 변환으로 맞춘 것
```

이다.

---

## 8. 자체 검토 중 잡은 실수

### 8.1 테스트 파일을 새 파일로 착각한 문제

처음에는 `test/unit/resp/decoder.ts`를 새 테스트 파일처럼 보고, 별도 helper를 넣어 작성했다.

그런데 다시 diff를 보니 이 파일은 이미 500줄짜리 RESP decoder 종합 테스트 파일이었다. 새 파일을 만든 게 아니라 기존 테스트를 크게 덮어쓴 상태였다.

이건 PR에 올라갔으면 큰 문제였다:

```text
기존 decoder coverage를 삭제하는 diff
리뷰어가 바로 reject할 변경
```

그래서 원본을 `upstream/main` 기준으로 복구하고, 기존 `Double` 섹션에 필요한 테스트 4개만 추가하는 방식으로 수정했다.

### 8.2 Prettier diff 노이즈 제거

`prettier --write`를 돌리면서 기존 HGETALL 테스트의 긴 라인 2개가 같이 줄바꿈됐다.

동작 변경은 아니지만 이번 PR과 관계없는 diff noise다. 그래서 해당 라인은 원래 형태로 되돌렸다.

최종 diff는 다음 두 가지로만 남겼다:

```text
1. RESP3 double parser 변경
2. Double 섹션 boundary test 4개 추가
```

---

## 9. PR 본문

실제 PR 본문은 짧게 작성했다.

```md
Fixes #2149.

RESP3 double decoding currently builds the number manually by accumulating decimal digits and then applying the exponent. That does not always match JavaScript's `Number()` conversion, so a few boundary values decode incorrectly:

- `1.7976931348623157e308` becomes `Infinity`
- `5e-324` becomes `0`
- `0.000000000000000001` becomes `0`
- `0.30000000000000004` rounds to `0.3000000000000001`

This changes RESP3 double decoding to reuse the existing string decoding path and lets `Number()` perform the numeric conversion. RESP3 spellings for `inf`, `+inf`, `-inf`, and `nan` are still handled explicitly.

I also checked a small local decoder loop: short literals like `,1.5\r\n` were slightly slower, while the boundary cases above were faster with this change.

Tests added for the boundary values. The existing decoder test helper covers single chunk, byte-by-byte, and split-at-every-byte-boundary decoding for each case.
```

의도:

- 이슈 번호를 명확히 연결한다.
- 기존 결과와 기대값을 숨기지 않는다.
- 성능 이야기는 과하게 방어하지 않고, 확인한 사실만 한 줄로 둔다.
- 테스트가 chunk boundary까지 커버한다는 점을 짧게 설명한다.

---

## 10. 배운 점

### 10.1 통신 규약은 애매함을 없애기 위한 장치다

Redis와 ioredis는 RESP라는 규약으로 대화한다.

규약이 없으면:

```text
값에 공백이 있으면?
값에 줄바꿈이 있으면?
값이 null이면?
숫자 문자열과 숫자를 어떻게 구분하지?
배열은 어디서 끝나지?
네트워크 chunk가 중간에 끊기면 어떻게 알지?
```

같은 문제가 계속 생긴다.

RESP는 앞 글자와 길이, CRLF 같은 규칙으로 이 애매함을 없앤다.

### 10.2 라이브러리 코드는 일반 서비스 코드보다 분기가 많다

이번 코드만 봐도 분기가 많다:

```text
RESP2 compatible mapping이면 string/Buffer 유지
RESP3 native mapping이면 number 반환
chunk가 끝까지 왔으면 바로 parse
중간에 끊겼으면 continuation
inf/nan 특수값이면 직접 처리
일반 숫자면 Number()
```

일반 서비스 코드는 우리 서비스의 전제를 깔 수 있지만, 라이브러리는 수많은 사용자와 환경을 상대한다. 그래서 코드가 깔끔한 레이어드 구조처럼 보이기보다, 많은 예외와 경계조건을 처리하는 상태 머신처럼 보이는 경우가 많다.

### 10.3 직접 최적화한 parser는 correctness 검증이 어렵다

`digit * 10^-k` 방식은 직관적으로는 맞아 보인다.

하지만 decimal string을 IEEE 754 double로 정확히 바꾸는 것은 단순 덧셈 문제가 아니다.

이번 버그는 좋은 예다:

```text
직접 parser는 빠르게 보였지만
Number.MAX_VALUE, Number.MIN_VALUE, 1e-18, rounding edge에서 틀렸다
```

값을 직접 만드는 최적화는 그 값의 semantics를 완전히 재현할 수 있을 때만 안전하다.

### 10.4 회귀 테스트는 이슈의 실패값을 그대로 담는 게 좋다

이번 테스트는 추상적인 "double should be accurate"가 아니다.

실패한 값 자체를 넣었다:

```text
1.7976931348623157e308
5e-324
0.000000000000000001
0.30000000000000004
```

그래서 나중에 누군가 decoder를 다시 최적화하더라도 이 경계값들을 깨면 바로 실패한다.

### 10.5 기존 테스트 helper를 먼저 읽어야 한다

처음에 기존 `test/unit/resp/decoder.ts` 구조를 제대로 못 보고 새 테스트처럼 작성했다가, 기존 coverage를 덮어쓰는 실수를 했다.

오픈소스에서는 새 코드를 넣기 전에 기존 테스트 컨벤션을 읽는 게 특히 중요하다.

이번 경우 기존 helper가 이미:

```text
single chunk
byte by byte
split at every byte boundary
```

를 전부 커버하고 있었기 때문에, 그걸 활용하는 게 가장 좋은 방식이었다.

---

## 11. 최종 정리

이번 PR의 핵심은 다음 문장으로 정리할 수 있다:

> Redis RESP3 double 응답을 JS `number`로 바꿀 때, ioredis가 직접 구현한 decimal-to-double 변환이 JS `Number()`와 다른 반올림 결과를 만들었다. double token은 기존 string decoder로 안전하게 읽고, 일반 숫자 변환은 `Number()`에 맡기도록 바꿔 JS number semantics와 맞췄다.

수정 규모는 작지만, 라이브러리 코드에서 프로토콜 파싱과 숫자 semantics가 얼마나 민감한지 잘 보여주는 작업이었다.

---

## 12. 메인테이너 승인 (2026-07-27)

Cursor Bugbot의 자동 리뷰(사소한 이슈 1건) 이후, 메인테이너 `nkaradzhov`가 직접 리뷰를 남겼다:

> "@GiHoon1123, thanks for the fix! LGTM."

`APPROVED`로 승인됨. 짧게 감사 인사로 답변했다. `mergeStateStatus: CLEAN`, `mergeable: MERGEABLE` — 코드/리뷰 관점에서 머지를 막는 요소는 없고, 메인테이너가 실제로 머지 버튼을 누르는 것만 남은 상태.

---

## 13. 머지 (2026-08-06)

승인된 채로 며칠 그대로 있어서, 확인 코멘트를 남겼다:

> "Just checking in on this one. It looks like the PR has approval and the checks are green, but no rush at all. Please let me know if there's anything else needed from my side."

다른 메인테이너 `PavelPashov`가 곧바로 머지하며 답변:

> "Merged now, thank you for the contribution"

승인만 나고 며칠 방치되는 경우, 재촉하는 게 아니라 "필요한 게 더 있는지" 묻는 정도의 확인 코멘트로도 충분히 진행되는 경우가 있다는 걸 보여준 사례.
