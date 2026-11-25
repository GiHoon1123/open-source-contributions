# KotlinPoet: reified 타입 매개변수 생성 예제 추가 (PR #2427)

작성일: 2026-11-11.

현재 상태: **OPEN**. 문서 변경을 제출했고 사람의 리뷰를 기다린다.
CLA 양식은 사용자가 제출했다고 확인했으며, PR의 체크박스를 반영한 뒤
`Task List Completed`가 성공으로 바뀌었다. 이는 체크리스트 완료 확인이며,
CLA 서명 내용에 대한 별도 법적 검토나 병합 승인을 뜻하지 않는다.

## 1. 작업 링크와 범위

| 항목 | 내용 |
| --- | --- |
| 원본 저장소 | https://github.com/square/kotlinpoet |
| 이슈 | https://github.com/square/kotlinpoet/issues/939 |
| PR | https://github.com/square/kotlinpoet/pull/2427 |
| 사용자 포크 | https://github.com/GiHoon1123/kotlinpoet |
| 브랜치 | `docs/reified-type-parameters` |
| 커밋 | `582f82d6f48e1d6676dfc8c6cf3289271a904920` |
| 기준 main | `f4e88d36` |
| 로컬 코드 위치 | `/Users/dustin/Desktop/common/open-source/kotlinpoet` |
| 변경 파일 | `docs/functions.md` |
| 변경 규모 | 24줄 추가 |
| 작성자 | `GiHoon1123 <rlaejrqo465@naver.com>` |

코틀린 공부와 오픈소스 기여를 함께 시작하는 첫 작업으로 문서 이슈를 골랐다.
라이브러리 구현을 수정하지 않고 이미 제공되는 API의 사용법을 보완했다.
조사 시점에 같은 내용을 처리하는 열린 PR은 찾지 못했다.
이슈에 연결된 과거 PR #961은 닫혀 있었고, 이번 문서 예제 추가와 같은 변경은 아니었다.

## 2. KotlinPoet은 무엇을 하는 라이브러리인가

KotlinPoet은 코틀린 소스 코드를 생성하는 라이브러리다.
개발자가 생성할 클래스·함수·타입·본문 등을 API로 지정하면 그에 맞는 소스 코드를 만든다.

예를 들어 다음은 우리가 직접 작성할 수 있는 함수다.

```kotlin
fun hello() {
  println("안녕")
}
```

KotlinPoet을 쓰는 프로그램은 이런 함수를 코드로 만들어낼 수 있다.

```kotlin
val hello = FunSpec.builder("hello")
  .addStatement("println(%S)", "안녕")
  .build()
```

`FunSpec`은 생성할 함수의 구조를 표현한다.
`builder("hello")`는 함수 이름을 정하고, `addStatement()`는 본문에 넣을 문장을 지정한다.
`%S`는 전달한 값을 소스 코드의 문자열 리터럴로 출력하기 위한 자리다.
`build()`는 설정한 내용으로 함수 정보를 완성한다.

여기서 중요한 구분은 함수 실행과 소스 생성이다.

| 행동 | 결과 |
| --- | --- |
| 생성된 `hello()` 호출 | 화면에 `안녕` 출력 |
| KotlinPoet 빌더 실행 | 출력문이 들어 있는 함수 정보 생성 |
| 함수 정보를 소스 문자열로 출력 | `public fun hello() { ... }`라는 코드 생성 |

`addStatement()`에 출력문을 넣었다고 그 순간 `안녕`이 출력되는 것은 아니다.
그 출력문이 나중에 들어갈 함수의 코드를 만들고 있는 것이다.

```text
코드 생성 프로그램 실행
  ↓
KotlinPoet으로 함수·클래스 구조 구성
  ↓
코틀린 소스 생성
  ↓
생성된 소스 컴파일
  ↓
생성된 프로그램 실행
```

DB 테이블이나 프로토콜 정의에서 반복되는 클래스를 만드는 도구 등을 생각하면 된다.
무엇을 만들지는 개발자가 정하고, 반복적인 소스 작성을 라이브러리에 맡긴다.

### 바이브 코딩 도구인가

대화 중 처음 나온 질문이다. AI를 사용해 자연어 요청을 해석하는 도구는 아니다.
개발자가 이름·타입·본문을 구체적으로 지정하는 코드 생성 API다.
AI가 작성할 코드를 판단하는 것과, 프로그램으로 정해진 구조의 코드를 생성하는 것은 다르다.

## 3. 먼저 알아야 할 제네릭

일반적인 함수는 매개변수 타입을 구체적으로 지정할 수 있다.

```kotlin
fun printValue(value: String) {
  println(value)
}
```

`fun`은 함수를 선언하는 키워드다.
위 함수는 문자열을 받으므로 `printValue("안녕")`은 가능하지만 `printValue(123)`은 불가능하다.
코틀린에서 `Int`는 정수 타입이다.

여러 타입에 대해 같은 동작을 하려면 타입 매개변수를 선언할 수 있다.

```kotlin
fun <T> printValue(value: T) {
  println(value)
}
```

`T`는 개발자가 붙인 이름이다. 어떤 타입인지는 사용할 때 정하고,
그 타입을 함수 선언에서는 T라고 부른다. 이런 함수를 제네릭 함수라고 한다.

```kotlin
printValue<String>("안녕")
printValue<Int>(123)
```

첫 번째 호출의 T는 String, 두 번째 호출의 T는 Int다.

```text
printValue<String>("안녕")
           타입      값
```

설명할 때는 이를 타입을 전달한다고 표현할 수 있다.
다만 아래에서 다루듯, 이것이 클래스 정보를 실행 인자로 넘긴다는 뜻은 아니다.

코틀린은 인자의 값으로 타입을 추론할 수 있으면 다음처럼 생략하는 것도 허용한다.

```kotlin
printValue("안녕")
printValue(123)
```

## 4. T 타입의 값을 쓰는 것과 T의 클래스 정보를 조회하는 것은 다르다

다음 목표를 생각해봤다.

```kotlin
printType<String>()
```

값은 받지 않고 전달한 타입의 클래스 정보를 출력하는 함수다.
`String::class`는 String의 클래스 정보를 나타내는 표현이다.
그렇다면 다음과 같이 작성하고 싶어진다.

```kotlin
// 컴파일되지 않는 예제
fun <T> printType() {
  println(T::class)
}
```

일반적인 T는 타입 검사에 사용할 수 있지만 이런 클래스 리터럴에는 사용할 수 없다.
Kotlin/JVM의 일반적인 제네릭은 타입 소거를 사용한다.
호출을 검사할 때 T가 String임을 알았다는 것과,
실행 중 함수 본문에서 T의 클래스 정보를 사용할 수 있다는 것은 별개다.

```kotlin
fun <T> printValue(value: T) {
  println(value) // 실제 인자로 받은 값을 사용하는 것은 가능
}
```

위 함수에는 실행 중 사용할 값이 실제로 전달된다.
반면 일반적인 `printType<String>()` 호출의 `<String>`은
String의 클래스 정보를 숨겨진 실행 인자로 자동 전달하는 장치가 아니다.

이 설명은 일반적인 제네릭 호출에 관한 것이다.
JVM에 모든 타입 관련 메타데이터가 전혀 남지 않는다거나,
객체 자체의 클래스를 알아낼 수 없다는 뜻은 아니다.

## 5. inline이 본문을 펼친다는 뜻

먼저 타입과 무관한 작은 함수를 예로 들었다.

```kotlin
fun double(number: Int): Int {
  return number * 2
}

fun main() {
  val result = double(3)
}
```

일반적인 함수 호출은 인자를 전달하고 함수가 계산한 결과를 받는 흐름으로 이해할 수 있다.
다음처럼 `inline`을 붙이면 컴파일러는 함수 본문을 호출 지점에 반영한다.

```kotlin
inline fun double(number: Int): Int {
  return number * 2
}
```

개념적으로는 다음과 같은 변환이다.

```kotlin
// 작성한 호출
val result = double(3)

// 호출 자리에 계산을 펼친 모습
val result = 3 * 2
```

소스 파일을 직접 고쳐 쓰는 것이 아니라 컴파일 과정에서 실행 코드를 만드는 방식이다.
위 예제는 개념을 설명하기 위한 것이고, 실제 바이트코드가 반드시 그대로 나온다는 뜻은 아니다.
JVM의 실행 중 최적화도 별도로 존재한다.

일부 함수 호출 비용을 줄일 수 있고, 코틀린에서는 특히 람다를 받는 함수에서 도움이 될 수 있다.
하지만 본문이 여러 호출 지점에 들어가면 결과 코드 크기가 커질 수 있다.
작은 함수에 무조건 붙이는 성능 향상 표시로 이해하면 안 된다.
이번 작업에서 inline이 중요한 이유는 속도 측정이 아니라 reified 타입 처리를 가능하게 하기 때문이다.

## 6. reified와 inline의 관계

```kotlin
inline fun <reified T> printType() {
  println(T::class)
}
```

`reified`는 타입 매개변수 T의 실제 타입을 함수 본문에서 사용할 수 있게 한다.
`inline`은 그 처리를 호출 지점에 반영할 수 있도록 한다.

```kotlin
printType<String>()
printType<Int>()
```

개념적으로 각 호출은 다음처럼 이해할 수 있다.

```kotlin
println(String::class)
println(Int::class)
```

컴파일러는 각 호출 지점에서 실제 타입을 알고 있다.
그 자리로 본문을 펼치면서 T가 쓰인 부분에 실제 타입을 반영할 수 있다.

| 조합 | 가능 여부 |
| --- | --- |
| inline만 사용 | 가능 |
| inline과 reified 함께 사용 | 가능 |
| 일반 함수에 reified만 사용 | 불가능 |

### 왜 reified만으로는 안 되는가

대화에서 가장 오래 확인한 부분이다.
`reified`라는 표시만 붙인다고 클래스 정보가 실행 인자로 새로 전달되는 것은 아니다.
코틀린은 이 기능을 inline 함수의 호출 지점에서 타입을 반영하는 방식으로 제공한다.
따라서 일반 함수에 reified 타입 매개변수만 붙일 수 없다.
다른 언어 설계로도 불가능한 일이라는 뜻이 아니라, 코틀린이 제공하는 방식의 제약이다.

inline 없이도 클래스 정보를 실제 값으로 받는 함수는 만들 수 있다.

```kotlin
fun printType(type: Class<*>) {
  println(type)
}

printType(String::class.java)
```

이것은 JVM 예제다. `Class<*>`는 구체적인 대상 타입을 제한하지 않는 Java 클래스 정보 타입이다.
`String::class.java`로 실제 클래스 정보를 인자로 전달하므로 본문에서 그 값을 사용할 수 있다.

```text
클래스 정보를 실제 인자로 전달
  → printType(String::class.java)

inline + reified로 컴파일할 때 타입을 반영
  → printType<String>()
```

### 대화 중 표현을 바로잡은 부분

처음에는 제네릭을 타입을 전달하는 기능으로 설명했다.
이것이 실행 중에 클래스 정보를 값처럼 넘긴다는 의미로 이해될 수 있어 보완했다.
정확히는 일반적인 제네릭의 타입 인자는 컴파일러의 타입 검사에 사용되고,
inline + reified에서는 호출 지점의 실제 타입이 생성 코드에 반영된다.

reified가 있다고 중첩된 제네릭 타입 인자의 모든 정보까지 자동으로 복구되는 것은 아니다.
이번 설명과 문서 예제는 T의 클래스 정보를 사용하는 범위다.

## 7. 이슈 #939는 어떤 혼동을 보고했나

작성자는 다음 형태의 함수를 KotlinPoet으로 생성하고 싶었다.

```kotlin
inline fun <reified T> foo()
```

아래는 그 의도를 자연스럽게 표현한 것처럼 보인다.

```kotlin
TypeVariableName("T", KModifier.REIFIED)
```

그러나 이 호출의 두 번째 인자는 임의의 수식어를 설정하는 자리가 아니다.
다음처럼 `variance`라는 이름을 가진다.

```kotlin
TypeVariableName(name = "T", variance = ...)
```

변성은 제네릭 타입 사이의 관계를 다루는 별도 개념이다.
여기서는 선택적인 variance 인자에 `KModifier.IN`이나 `KModifier.OUT`을 지정하며,
지정하지 않을 때는 null이라는 점까지만 이해하면 된다.
이번 변경은 변성 규칙 자체를 수정하지 않는다.

IN, OUT, REIFIED가 모두 KModifier라는 같은 타입에 속하므로
잘못된 호출도 인자 타입 검사만으로는 걸러지지 않는다.
라이브러리가 실행 중 허용하는 값을 검사할 때 예외가 발생한다.

```text
컴파일: 두 번째 인자가 KModifier인가?
  → REIFIED도 KModifier이므로 통과

실행: variance에 허용된 값인가?
  → REIFIED는 허용되지 않으므로 IllegalArgumentException
```

이슈의 원래 실행 환경은 annotation processing 경로였고,
바깥에서는 InvocationTargetException 등으로 감싸져 보였다.
하지만 내부 원인은 REIFIED를 variance로 사용한 것이었다.
이번 로컬 작업에서 당시 kapt 프로젝트 전체를 재현한 것은 아니다.
현재 API와 기존 테스트를 확인하고, 올바른 문서 예제를 실행해 검증했다.

## 8. 올바른 KotlinPoet 설정

```kotlin
TypeVariableName("T").copy(reified = true)
```

먼저 T 타입 매개변수 정보를 만든다.
그다음 `copy()`로 reified 설정이 켜진 새 객체를 만든다.
기존 객체를 제자리에서 수정하는 호출이 아니다.

소스에서 확인한 이름도 구분해야 한다.

| 위치 | 이름 |
| --- | --- |
| 객체가 보유한 프로퍼티 | `isReified` |
| copy 함수의 인자 | `reified` |

조사 중 프로퍼티 이름을 보고 사용법도 isReified라고 설명할 뻔했지만,
실제 예제에는 `.copy(reified = true)`가 맞다.
프로퍼티 이름과 함수의 이름 있는 인자를 같다고 가정하지 말아야 한다.

## 9. 실제 추가한 문서 예제

`docs/functions.md`의 Single-expression functions와 Default function arguments 사이에
Reified type parameters 절을 추가했다.

```kotlin
val printMembers = FunSpec.builder("printMembers")
  .addModifiers(KModifier.INLINE)
  .addTypeVariable(TypeVariableName("T").copy(reified = true))
  .addStatement("println(T::class.members)")
  .build()
```

생성 결과는 다음과 같다.

```kotlin
public inline fun <reified T> printMembers() {
  println(T::class.members)
}
```

`public`은 외부에서 접근할 수 있는 공개 함수라는 뜻이다.
KotlinPoet이 명시적인 접근 제한자를 출력하므로 실제 출력과 맞춰 적었다.

`T::class.members`는 클래스의 함수·프로퍼티 등 멤버 정보를 조회한다.
생성된 함수를 JVM에서 실행하려면 이 리플렉션 기능에 필요한 kotlin-reflect가 있어야 한다.
설명에 사용한 간단한 printType 예제와 실제 제출한 printMembers 예제는 구분한다.
실제 예제는 기존 TypeVariableNameTest의 reified 사례와 같은 구조를 사용했다.

예제 뒤에는 variance에 REIFIED를 넣을 수 없다는 점과,
reified 타입 매개변수는 inline 함수에서만 허용된다는 점을 짧게 적었다.

## 10. 왜 API를 고치지 않았나

이슈에는 사용법을 더 명확히 하자는 요구와 새 타입 수식어 API를 만들자는 논의가 함께 있었다.
메인테이너는 현재 API에서 두 번째 인자는 variance라는 점을 설명했고,
문서를 개선하는 PR도 환영한다고 답했다.

현재 필요한 기능은 이미 제공되고 있다.
이번 기여는 올바른 사용 경로를 문서에서 찾을 수 있게 하는 것으로 범위를 정했다.
KModifier나 TypeVariableName의 공개 API를 바꾸거나 호환성을 건드리지 않았다.
문서 예제 하나뿐 아니라 잘못 쓰기 쉬운 인자의 용도와 inline 조건을 함께 설명한 이유다.

## 11. 포크와 저장소 규칙

작업 전에 README, 기여 가이드, PR 템플릿, 빌드 설정과 CI를 확인했다.
코드 저장소 안에 별도 AGENTS.md는 찾지 못했다.

- origin은 GiHoon1123/kotlinpoet, upstream은 square/kotlinpoet으로 설정했다.
- 포크를 로컬에 내려받은 다음 docs/reified-type-parameters 브랜치를 만들었다.
- 제출 전 upstream/main을 다시 fetch했고 추가로 따라잡을 커밋은 없었다.
- 문서 변경은 PR 템플릿의 예외에 따라 changelog를 추가하지 않았다.
- PR 템플릿의 체크리스트는 그대로 유지했다.
- CLA는 사용자가 직접 검토하고 제출해야 하는 항목으로 구분했다.

기여 가이드에는 일반적인 changelog 추가 지시가 있지만,
PR 템플릿은 문서·테스트·빌드 변경을 명시적으로 예외 처리한다.
이번에는 문서 전용 변경에 맞는 구체적인 예외를 적용했다.

## 12. 빌드 환경에서 확인한 것

| 항목 | 사용한 설정 |
| --- | --- |
| 로컬 실행 JDK | Temurin 21.0.11, arm64 |
| Gradle | 저장소 wrapper의 9.9.1 |
| Kotlin 플러그인 | 2.4.21 |
| JVM 산출물 target | 1.8 |
| 프로젝트 대상 | JVM, JavaScript, WebAssembly JS |

Gradle을 실행하는 JDK 버전과 산출물의 JVM target은 같은 개념이 아니다.
로컬 JDK가 21이라고 생성된 라이브러리의 JVM target도 21이 되는 것은 아니다.

Gradle wrapper는 저장소가 지정한 Gradle을 실행하기 위한 장치다.
빌드 도구인 Gradle과 코틀린 소스를 컴파일하는 Kotlin 컴파일러의 버전도 별도로 관리된다.
첫 실행에서는 wrapper와 의존성 다운로드가 있었다.
환경 문제를 해결하려고 임의로 버전을 올리지 않고 저장소 설정을 그대로 사용했다.

## 13. 검증 과정과 결과

모든 Gradle 명령은 로컬 kotlinpoet 저장소 루트에서 실행했다.

### 기존 관련 테스트

```sh
./gradlew :kotlinpoet:jvmTest \
  --tests com.squareup.kotlinpoet.TypeVariableNameTest \
  --tests com.squareup.kotlinpoet.FunSpecTest
```

기존 reified 생성 사례와 inline이 없는 함수에서 reified를 거부하는 사례를 확인했다.
관련 테스트 실행은 성공했다.
실제 제품 코드에는 변경이 없으므로 문서 수정 전후의 런타임 버그 재현을 주장하지 않는다.

### 문서 예제를 그대로 읽어 검증

검증용 임시 테스트에서 다음 순서로 확인했다.

1. 수정한 functions.md의 Reified type parameters 절을 읽었다.
2. Kotlin 코드 블록 두 개를 추출했다.
3. 첫 번째 블록을 감싼 생성 프로그램을 KotlinCompilation으로 컴파일했다.
4. 생성 프로그램을 실행해 함수 소스 문자열을 얻었다.
5. 그 문자열이 두 번째 코드 블록과 정확히 같은지 비교했다.
6. 생성된 함수와 `printMembers<String>()` 호출 코드를 다시 컴파일했다.
7. 컴파일된 호출 함수를 실제 실행했다.

별도로 손으로 옮긴 유사 예제만 실행한 것이 아니라 문서의 코드 블록을 사용했다.
생성 프로그램과 생성 결과는 각각 컴파일을 통과했고, 실제 멤버 정보 출력도 확인했다.

검증용 임시 테스트는 실행 후 제거했다.
기존 테스트가 이미 같은 기능을 다루므로 문서 기여에 중복 테스트를 영구 추가하지 않았다.
임시 검증 파일이 커밋되지 않았는지도 확인했다.

### 전체 빌드

```sh
./gradlew clean build
```

결과는 BUILD SUCCESSFUL, 소요 시간은 약 1분 8초였다.
91개 task 중 89개가 실행됐고 2개가 up-to-date였다.

| 테스트 결과 디렉터리 | 전체 | 통과 | 스킵 | 실패·오류 |
| --- | ---: | ---: | ---: | ---: |
| kotlinpoet JVM | 1,011 | 971 | 40 | 0 |
| kotlin-metadata | 158 | 131 | 27 | 0 |
| KSP test-processor | 20 | 20 | 0 | 0 |
| 합계 | 1,189 | 1,122 | 67 | 0 |

테스트 수는 최종 전체 빌드의 XML 결과를 집계한 값이다.
앞서 실행하고 삭제한 임시 문서 검증 테스트 1개는 이 합계에 포함하지 않는다.

**JavaScript와 WebAssembly는 빌드는 성공했지만 테스트 소스가 없어 테스트가 스킵됐다.**
처음 진행 보고에서 두 대상의 테스트도 통과했다고 표현했지만,
최종 자체검토에서 NO-SOURCE와 SKIPPED 로그를 확인하고 바로잡았다.
빌드 성공을 모든 플랫폼의 실제 테스트 실행 성공으로 확대하면 안 된다.

MkDocs 사이트 생성은 별도로 실행하지 않았다.
문서 코드 검증과 전체 Gradle 빌드 성공을 문서 사이트 빌드 성공으로 표현하지 않는다.
JDK 8 target 관련 경고와 Gradle 10 호환성 관련 deprecation 안내는 있었지만 빌드는 성공했다.

## 14. 자체검토에서 확인한 범위

- 이슈가 보고한 잘못된 인자와 올바른 설정을 함께 설명했는가.
- copy의 실제 인자 이름과 프로퍼티 이름을 혼동하지 않았는가.
- reified만 설정하고 inline은 빠뜨리지 않았는가.
- 문서의 생성 결과가 실제 출력과 일치하는가.
- 생성 결과도 컴파일·실행 가능한가.
- API와 동작을 바꾸지 않고 문서 요구를 해결했는가.
- 주변 문서와 코드 블록 형식이 맞는가.
- 임시 파일과 무관한 변경이 남지 않았는가.
- 스킵과 실제 테스트 통과를 구분했는가.
- CLA 등 제출 요건을 별도로 확인했는가.

필수 수정이 필요한 문제는 발견하지 못했다.
문서 파일 하나의 24줄 추가만 커밋했고 `git diff --check`도 통과했다.

## 15. 커밋·푸시·PR

사용자에게 개념을 설명하고 검토받은 뒤 커밋·푸시·PR 생성을 진행했다.
공개 작업 직전 GitHub CLI 계정이 GiHoon1123인지 확인했다.

커밋 제목과 PR 제목은 다음과 같다.

```text
docs: show how to generate reified type parameters
```

PR 본문은 추가한 예제, variance와 reified의 차이, Fixes #939,
예제 검증과 전체 빌드 결과를 짧게 적었다.
불필요한 설명이나 임의의 공동 작성자 표기는 추가하지 않았다.
커밋 작성자와 커미터 모두 GiHoon1123으로 확인했다.

## 16. CLA 제출 과정

첫 제출 때는 CLA 서명 여부를 확인하지 못해 해당 체크박스를 비워뒀다.
Square의 작업 목록 봇이 CLA signed 항목을 남은 작업으로 표시했고,
Task List Completed 상태가 ERROR였다. 코드 검사 실패와는 별개였다.

사용자가 서명 양식의 개인정보 항목을 보고 왜 필요한지 질문했다.
저장소 기여 가이드가 직접 연결한 양식인지 확인했고,
양식에는 Block, Inc.에서 생성했다는 표시가 있었다.

양식은 본인에게 기여물을 제공할 권한이 있는지 등의 확인과,
기여물을 사용·수정·배포할 수 있는 저작권 및 관련 특허 사용 허락을 다룬다.
원문은 기여자가 자신의 권리를 유지한다고 설명한다.
회사에 권리가 있는 기여라면 관련 허락에 대한 조건도 포함돼 있었다.
이 기록은 계약 원문이나 개별 법률 판단을 대신하지 않는다.

주소·국가·전화번호까지 필수였지만 그 항목들의 구체적 수집 목적과 보관 기간은
확인한 양식 본문에 설명돼 있지 않았다. 공식 링크라는 사실과
모든 개인정보 요구를 충분히 설명한다는 평가는 구분했다.

사용자는 직접 양식을 제출했다고 알렸다.
이를 근거로 PR 본문의 CLA 체크박스를 완료로 수정했다.
양식 접수 데이터나 제출 내용을 에이전트가 직접 조회한 것은 아니다.
이 문서에는 실제 주소·전화번호 같은 개인정보를 남기지 않는다.

체크박스 반영 후 봇 댓글이 All tasks completed로 바뀌고,
Task List Completed가 SUCCESS가 된 것을 확인했다.

## 17. 문서 작성 시점의 원격 상태

2026-11-11 확인 결과다.

| 항목 | 상태 |
| --- | --- |
| PR | OPEN |
| 사람의 리뷰 | 아직 없음 |
| Semgrep OSS | SUCCESS |
| zizmor | SUCCESS |
| Task List Completed | SUCCESS |
| GitHub mergeStateStatus | BLOCKED |

조회된 체크 목록에는 일반 Gradle 빌드 job의 결과가 없었다.
따라서 로컬 전체 빌드 성공과 GitHub 빌드 CI 성공은 구분한다.
작업 목록 검사가 성공한 뒤에도 mergeStateStatus는 BLOCKED였다.
이를 CLA 미완료나 코드 오류라고 다시 단정하지 않는다.
반영 여부와 다음 리뷰는 이후 실제 상태를 확인해 이 문서 끝에 추가한다.

## 18. 이번에 남긴 학습 포인트

문서 기여도 실제 API의 이름과 사용 조건을 읽고 예제를 실행해야 한다.
공개 API를 바꾸지 않아도 잘못된 사용으로 이어지는 지점을 설명해 기여할 수 있다.

설명에서는 한 번에 KotlinPoet, 제네릭, 타입 소거, inline, reified를 모두 소개했다가
이해하기 어렵다는 피드백을 받았다. 이후 코드 생성과 실행의 구분부터 시작해
타입 매개변수, 클래스 정보 조회, inline의 개념, reified가 필요한 이유를 순서대로 설명했다.
특히 타입을 전달한다는 표현이 실행 인자 전달로 오해될 수 있다는 점을 확인했다.

검증에서도 생성 도구가 동작한다는 사실과 생성된 코드가 동작한다는 사실을 나눴다.
또한 실제 테스트가 없는 대상의 빌드 성공을 테스트 통과라고 말했던 부분을 바로잡았다.
공개 PR의 짧은 설명과 별개로 이런 이해 과정과 정정 내용은 복습 기록에 남긴다.

## 참고 자료

- [KotlinPoet 소개](https://square.github.io/kotlinpoet/)
- [이슈 #939와 논의](https://github.com/square/kotlinpoet/issues/939)
- [추가한 문서 변경](https://github.com/square/kotlinpoet/pull/2427/files)
- [Kotlin inline과 reified](https://kotlinlang.org/docs/inline-functions.html#reified-type-parameters)
- [KotlinPoet 기여 가이드](https://github.com/square/kotlinpoet/blob/main/docs/contributing.md)
- [TypeVariableName 구현](https://github.com/square/kotlinpoet/blob/582f82d6f48e1d6676dfc8c6cf3289271a904920/kotlinpoet/src/jvmMain/kotlin/com/squareup/kotlinpoet/TypeVariableName.kt)
- [기존 TypeVariableNameTest](https://github.com/square/kotlinpoet/blob/582f82d6f48e1d6676dfc8c6cf3289271a904920/kotlinpoet/src/jvmTest/kotlin/com/squareup/kotlinpoet/TypeVariableNameTest.kt)

## 2026-11-25 상태 확인

[PR #2427](https://github.com/square/kotlinpoet/pull/2427)은 한국 시간 11월 14일 02:29에 머지됐다. [머지 커밋 95e3d930](https://github.com/square/kotlinpoet/commit/95e3d93020a364221e1feecee2b8e51a86a03143)
