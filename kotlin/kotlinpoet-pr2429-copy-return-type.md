# KotlinPoet: copy 결과의 구체적인 타입 유지 (PR #2429)

작성: 2026-11-13.

현재 상태: **OPEN**. 구현·회귀 테스트·전체 빌드와 기존 호출 코드의 실행 검증을 마친 뒤 제출했다.
마지막 상태 조회에서는 사람의 리뷰가 없고 Task List Completed는 SUCCESS였다.
mergeStateStatus는 BLOCKED였으며, 이를 코드 검사 실패로 해석하지 않는다.

## 1. 작업 정보

| 항목 | 내용 |
| --- | --- |
| 저장소 | https://github.com/square/kotlinpoet |
| 이슈 | https://github.com/square/kotlinpoet/issues/1967 |
| PR | https://github.com/square/kotlinpoet/pull/2429 |
| 포크 | https://github.com/GiHoon1123/kotlinpoet |
| 브랜치 | `fix/type-name-copy-return-type` |
| 기준 main | `f4e88d36` |
| 구현 커밋 | `2fa300e2` |
| changelog PR 링크 커밋 | `584dd0544f2aeea72557b7a9717e9a1dd226137a` |
| 작성자·커미터 | `GiHoon1123 <rlaejrqo465@naver.com>` |
| 코드 위치 | `/Users/dustin/Desktop/common/open-source/kotlinpoet` |

이전 [reified 예제 기여](kotlinpoet-pr2427-reified-type-parameters.md)와 독립된 작업이다.
문서 PR 브랜치 위에 이어서 수정하지 않고 upstream/main에서 새 브랜치를 만들었다.
제출 직전 다시 fetch했으며 따라잡아야 할 main의 추가 커밋은 없었다.
동일 이슈를 처리하는 열린 PR도 조회 시점에는 없었다.

## 2. 문제를 한 문장으로 설명하면

ClassName을 복사한 실제 객체는 ClassName인데,
특정 copy 오버로드의 반환 타입은 더 넓은 TypeName으로 선언돼 있었다.
그래서 호출자가 복사 결과의 ClassName 전용 기능을 바로 사용할 수 없었다.

복사된 객체가 망가지거나 클래스 이름이 사라지는 런타임 데이터 손상 문제는 아니다.
정적 타입 선언이 실제 결과의 구체적인 타입을 충분히 표현하지 못하는 API 문제다.

## 3. KotlinPoet과 ClassName부터 다시 보기

KotlinPoet은 코틀린 소스 코드를 생성하는 라이브러리다.
생성할 소스 안에서 사용할 클래스 이름도 객체로 표현한다.

```kotlin
val original = ClassName("example", "User")
```

이는 example 패키지에 있는 User라는 클래스의 이름 정보를 만든다.
이 객체가 나타내는 이름은 `example.User`다.

```kotlin
User()
```

위와 같이 실제 User 인스턴스를 만드는 것과 구분해야 한다.
ClassName은 생성할 코드에서 어떤 클래스 이름을 쓸 것인지 나타내는 정보다.

KotlinPoet에는 클래스 이름 외에도 여러 타입 표현이 있다.

| 표현할 타입 예시 | KotlinPoet 타입 |
| --- | --- |
| `User` | ClassName |
| `List<String>` | ParameterizedTypeName |
| `T` | TypeVariableName |
| `(Int) -> Boolean` | LambdaTypeName |
| `out Number`, `in String` | WildcardTypeName |

이들은 공통 상위 클래스 TypeName을 상속한다.

```text
TypeName
  ├─ ClassName
  ├─ ParameterizedTypeName
  ├─ TypeVariableName
  ├─ LambdaTypeName
  ├─ WildcardTypeName
  └─ Dynamic
```

ClassName은 TypeName의 한 종류다.
자바에서 Dog가 Animal의 한 종류라고 표현하는 상속 관계와 같은 방향이다.
Dynamic은 뒤에서 설명하듯 복사를 지원하지 않는 별도 경우다.

## 4. copy와 nullable은 무슨 뜻인가

코틀린에서 타입 뒤의 `?`는 null을 허용함을 나타낸다.

```kotlin
User   // null을 허용하지 않는 타입
User?  // null을 허용하는 타입
```

KotlinPoet에서 같은 클래스 이름 정보를 바탕으로 nullable 타입을 표현하려면 다음처럼 복사한다.

```kotlin
val original = ClassName("example", "User")
val copied = original.copy(nullable = true)
```

```text
original → example.User
copied   → example.User?
```

원본 객체의 설정을 바꾸는 것이 아니라 새 객체를 반환한다.
복사 로직 자체는 기존에도 올바른 ClassName을 만들었다.
문제는 copied 변수에 대해 컴파일러가 알게 되는 타입이었다.

## 5. 실제 객체의 타입과 컴파일러가 보는 타입

수정 전에는 다음 코드가 컴파일되지 않았다.

```kotlin
val copied: ClassName = original.copy(nullable = true)
```

오류의 의미는 ClassName을 기대하는 자리에 TypeName이 반환됐다는 것이다.
함수가 TypeName을 반환한다고 선언돼 있으므로 컴파일러가 임의로 ClassName이라고 판단할 수 없다.

예를 들어 ClassName에는 다음 기능이 있다.

```kotlin
original.simpleName // "User"
```

그러나 복사 결과를 TypeName으로 알게 되면 ClassName 전용 기능을 바로 쓸 수 없다.

```kotlin
val copied = original.copy(nullable = true)
copied.simpleName // 수정 전에는 오류
```

실제 실행 때 ClassName 객체가 나온다는 것과,
컴파일 시점에 어떤 프로퍼티와 함수를 허용할지는 다른 판단이다.
상위 타입에는 클래스 이름이 아닌 다른 타입 표현도 있으므로 이 제한은 컴파일러 입장에서는 정상이다.
개선할 부분은 컴파일러의 검사가 아니라 라이브러리의 반환 타입 선언이다.

## 6. 왜 tags를 넣으면 문제가 없었나

이름이 같고 인자 구성이 다른 함수를 오버로드라고 한다.
TypeName에는 다음 두 copy가 있었다.
아래는 실제 선언이다.

```kotlin
public fun copy(
  nullable: Boolean = this.isNullable,
  annotations: List<AnnotationSpec> = this.annotations.toList(),
): TypeName {
  return copy(nullable, annotations, this.tags)
}

public abstract fun copy(
  nullable: Boolean = this.isNullable,
  annotations: List<AnnotationSpec> = this.annotations.toList(),
  tags: Map<KClass<*>, Any> = this.tags,
): TypeName
```

annotations는 타입에 붙는 애노테이션 목록이고,
tags는 코드를 생성하는 도구가 보조 정보를 붙일 수 있는 맵이다.
이번 문제에서는 tags의 구체적 사용처보다 인자의 존재가 오버로드 선택에 영향을 준다는 점이 중요하다.

| 호출 | 선택되는 경로 | 수정 전 반환 타입 |
| --- | --- | --- |
| `original.copy()` | 인자 두 개짜리 오버로드의 기본값 사용 | TypeName |
| `original.copy(nullable = true)` | 인자 두 개짜리 오버로드 | TypeName |
| `original.copy(annotations = annotations)` | 인자 두 개짜리 오버로드 | TypeName |
| `original.copy(true, annotations)` | 인자 두 개짜리 오버로드 | TypeName |
| `original.copy(nullable = true, tags = original.tags)` | tags를 받는 오버로드 | ClassName |

ClassName은 tags를 받는 copy만 재정의하고 그 반환 타입을 ClassName으로 선언했다.
tags 없는 copy는 부모의 구현을 물려받아 반환 타입도 TypeName인 상태였다.

첫 번째 copy도 내부에서는 두 번째 copy로 위임한다.

```text
tags 없는 copy 호출
  ↓
부모 구현에서 this.tags를 넣어 tags 있는 copy 호출
  ↓
ClassName의 재정의된 구현이 ClassName 생성
  ↓
바깥 함수의 반환 타입 선언은 여전히 TypeName
```

이 흐름이 실제 객체는 맞는데 정적 타입이 넓어지는 이유다.

## 7. 코틀린의 open과 override

자바와 달리 코틀린은 일반적인 클래스와 메서드가 기본적으로 final이다.
상속과 재정의를 허용하려면 필요한 위치에 open을 붙인다.

```kotlin
open class Animal {
  fun eat() {}
  open fun speak() {}
}

class Dog : Animal() {
  override fun speak() {}
  // override fun eat() {} // 오류: eat은 final
}
```

클래스의 open은 상속을 허용하고, 메서드의 open은 메서드 재정의를 허용한다.
클래스가 상속 가능하다고 모든 메서드가 자동으로 재정의 가능한 것은 아니다.
abstract 메서드와 재정의 가능한 인터페이스 메서드에는 open을 따로 붙이지 않는다.
override한 메서드는 별도로 final 처리하지 않는 한 다시 재정의할 수 있다.

이번 TypeName은 sealed 클래스이고 해당 하위 타입들은 이미 상속하고 있었다.
수정해야 한 것은 상속 구조가 아니라 tags 없는 copy 메서드가 final인 부분이었다.

## 8. 구현 변경

부모의 두 인자 copy를 다음처럼 열었다.

```kotlin
public open fun copy(
  nullable: Boolean = this.isNullable,
  annotations: List<AnnotationSpec> = this.annotations.toList(),
): TypeName {
  return copy(nullable, annotations, this.tags)
}
```

기존 메서드를 삭제하거나 이름·인자를 바꾸지 않았다.
그다음 다섯 하위 타입에 반환 타입을 구체화한 재정의를 추가했다.
ClassName의 구현은 다음과 같다.

```kotlin
override fun copy(
  nullable: Boolean,
  annotations: List<AnnotationSpec>,
): ClassName {
  return copy(nullable, annotations, this.tags)
}
```

실제 소스에서는 함수 선언이 한 줄로 포맷되지만 내용은 같다.
부모의 기본 인자는 재정의된 메서드에도 적용되므로 하위 선언에 기본값을 다시 작성하지 않는다.

같은 변경을 적용한 타입은 다음과 같다.

- ClassName
- ParameterizedTypeName
- LambdaTypeName
- TypeVariableName
- WildcardTypeName

실제 복사는 기존 tags 포함 copy에 위임한다.
생성자 호출과 복사 규칙을 새로 복제하지 않았고, tags도 this.tags로 유지한다.
LambdaTypeName과 TypeVariableName의 추가 옵션을 받는 기존 오버로드도 그대로 둔다.
Dynamic은 원래 복사를 지원하지 않으므로 구체적인 복사 반환 타입을 새로 추가하지 않았다.

## 9. 반환 타입을 구체화해도 되는 이유

ClassName은 TypeName의 하위 타입이다.
ClassName을 반환하는 함수 결과는 여전히 TypeName을 받는 자리에서 사용할 수 있다.

```kotlin
val specific: ClassName = original.copy(nullable = true)
val general: TypeName = original.copy(nullable = true)
```

수정 후에는 둘 다 가능하다.
부모 메서드의 반환 타입보다 더 구체적인 하위 타입을 반환하는 재정의는 허용된다.
이를 공변 반환 타입이라고 부른다.

반대로 TypeName이 나올 수 있는 결과를 확인 없이 ClassName으로 받는 것은 안전하지 않다.
앞서 컴파일 오류가 났던 이유다.
이번 변경은 실제 복사 결과에 맞춰 반환 타입을 좁힌 것이다.

다만 타입 이론상 가능한 것과 JVM 바이너리 호환성까지 확인한 것은 구분해야 한다.
이슈의 메인테이너 논의에서도 반환 타입을 하위 타입으로 바꾸는 것은 괜찮다는 방향에 합의했지만,
실제 컴파일·실행 결과를 별도로 검증했다.

## 10. 과거 코드가 왜 이렇게 되어 있었나

git log와 과거 커밋을 읽었다.
두 인자 copy는 과거 ABI 호환성을 복구하려고 추가된 메서드였다.

- 과거 커밋: `97e9a1b5`, Restore ABI compatibility to TypeName#copy (#783)
- 이후 tags를 보존하는 동작도 유지돼 왔다.

따라서 중복처럼 보인다는 이유로 두 인자 메서드를 지우면 안 된다.
이번에는 기존 선언과 기본 인자 진입점을 유지하면서 하위 타입의 재정의만 추가했다.
이력은 수정할 코드의 이유를 이해하고 구현 범위를 정하는 근거가 됐다.

## 11. 추가로 제거해야 했던 형변환

반환 타입이 구체화되면서 기존 코드의 다음 형변환이 불필요해졌다.

```kotlin
// 수정 전 필요한 우회
className.copy(nullable = false) as ClassName

// 수정 후
className.copy(nullable = false)
```

이 저장소는 Kotlin 경고를 오류로 처리한다.
처음 빌드에서는 CodeWriter의 Cast is redundant 경고가 실패 원인이 됐다.
따라서 관련 캐스트가 남은 곳을 찾아 제거했다.

- CodeWriter
- KSP 연동의 KsTypes
- CodeBlockTest
- KotlinPoetTest
- TypeSpecTest

일반적인 코드 정리 작업을 섞은 것이 아니라 이번 반환 타입 변경으로 불필요해진 캐스트만 제거했다.
포매터에 따른 줄바꿈도 해당 표현에 함께 적용됐다.
이 변경으로 저장소 내부 호출도 더 구체적인 타입을 직접 사용하는 사례가 된다.

## 12. 회귀 테스트

새 파일 TypeNameCopyTest.kt에 테스트 6개를 추가했다.
다섯 구체 타입과 복사를 지원하지 않는 Dynamic을 다룬다.

각 구체 타입에서는 결과를 해당 하위 타입의 리스트로 명시한다.

```kotlin
val copies: List<ClassName> = listOf(
  type.copy(),
  type.copy(nullable = true),
  type.copy(annotations = annotations),
  type.copy(true, annotations),
)
```

이 명시적인 타입이 중요하다.
결과를 TypeName으로만 받으면 기존 코드에서도 테스트가 컴파일돼 정적 타입 문제를 놓칠 수 있다.
명시적 하위 타입을 요구하므로 이전 구현으로는 테스트 자체가 컴파일되지 않는다.

실제로 수정 전 다섯 타입에서 각각 List<구체 타입> 대신 List<TypeName>이 추론되는 오류를 확인했다.
수정 후에는 모두 컴파일되고 실행됐다.

### 보존할 정보

- 원래 애노테이션과 변경할 애노테이션.
- null 허용 여부와 원본의 불변성.
- tags 맵.
- 중첩 클래스 이름.
- 제네릭 타입 인자와 바깥 타입.
- 함수 타입의 receiver·매개변수·반환 타입·context parameter·suspend 설정.
- 타입 변수의 bounds·variance·reified 설정.
- wildcard의 생산·소비 방향.

구체 타입별 입력을 구성하고 기존 tags 포함 복사 결과와 비교했다.
equals에서 tags는 동등성 대상이 아니므로 tags는 별도 assertion으로 확인했다.
복사본을 TypeName으로 받은 경로에서도 nullable·annotations 변경이 올바르게 되는지 확인했다.
Dynamic은 기존과 같이 UnsupportedOperationException을 던지는지 확인했다.

## 13. 로컬 검증 명령

모든 Gradle 명령은 로컬 kotlinpoet 저장소 루트에서 실행했다.

수정 전 새 테스트 컴파일:

```sh
./gradlew :kotlinpoet:compileTestKotlinJvm
```

다섯 하위 타입의 반환 타입 불일치로 실패했다.
테스트 초안의 tags 맵 타입 추론 문제는 먼저 수정했고,
그 후 문제의 반환 타입 오류만 남는 것을 확인한 로그를 기준 재현으로 삼았다.

수정 후 관련 테스트·API 덤프·포맷 적용:

```sh
./gradlew :kotlinpoet:jvmTest \
  --tests com.squareup.kotlinpoet.TypeNameCopyTest \
  :kotlinpoet:apiDump \
  :kotlinpoet:spotlessApply \
  :interop:ksp:spotlessApply
```

관련 테스트 6개 모두 성공했고 스킵은 없었다.
공개 API가 추가되므로 저장소가 관리하는 kotlinpoet.api도 생성 도구로 갱신했다.
API 덤프를 새 결과에 맞추는 것만으로 기존 바이너리와 호환된다고 주장하지 않는다.

전체 검증:

```sh
./gradlew clean build
```

결과는 BUILD SUCCESSFUL이었다.
약 31초가 걸렸고, 91개 task 중 79개 실행·12개 up-to-date였다.
전체 테스트 XML 집계는 1,195개 중 1,128개 통과·67개 스킵·실패 및 오류 0개다.
새 테스트 6개는 이 합계에 포함된다.
JavaScript와 WebAssembly는 빌드 성공과 테스트 소스 부재로 인한 스킵을 구분한다.
빌드 경고를 플랫폼 테스트 실행 성공으로 오해하지 않는다.

## 14. 바이너리 호환성은 어떻게 확인했나

기존 소스가 새 라이브러리로 다시 컴파일되는지와,
예전에 컴파일한 파일이 재컴파일 없이 새 라이브러리에서 실행되는지는 다른 검사다.

이번에는 변경 전 main으로 만든 baseline JAR를 별도로 보관했다.
그 JAR만 KotlinPoet 구현으로 사용하는 클래스패스에서 Kotlin과 Java 호출 코드를 컴파일했다.
검증용 클래스패스에서 작업 중인 새 main 클래스 디렉터리를 제외해
실수로 새 라이브러리를 보고 컴파일하지 않게 했다.

```text
변경 전 라이브러리
  ↓ 이를 기준으로 호출 코드 컴파일
기존 Kotlin·Java 바이트코드
  ├─ 변경 전 라이브러리로 실행 → 정상
  └─ 변경 후 라이브러리로 실행 → 정상
```

Kotlin 호출 코드는 다섯 타입에 대해 기본 인자 사용,
nullable만 지정, annotations만 지정, 두 인자 명시, tags 지정 경로를 포함했다.
TypeName으로 받은 호출과 Dynamic의 기존 예외 동작도 확인했다.

Java 호출은 기존에 상속받아 사용하던 두 인자 copy와
TypeName을 통한 호출, tags 포함 세 인자 copy를 실행했다.
실행에는 `-Xverify:all`을 사용했다.
Kotlin 호출 코드 안에서 실제 로드된 라이브러리의 codeSource 경로도 확인했다.
그래서 실행만 된 것이 아니라 의도한 JAR가 사용됐다는 점도 검증했다.

검증 스크립트에서는 macOS의 /tmp와 /private/tmp 경로 차이를 만났다.
정규화하지 않은 문자열 비교에서만 실패했고, canonicalPath로 비교한 뒤 정상적으로 검증했다.
이는 라이브러리의 복사 동작 실패가 아니라 검증 도구의 경로 비교 문제였다.

추가로 Java 소스에서도 다섯 copy 결과를 각 구체 타입으로 반환하는 코드가 컴파일되는지 확인했다.
기존 바이너리 호출과 새 소스의 구체 타입 사용을 나눠 확인한 것이다.

이 결과는 테스트한 main 기준·JDK 환경·호출 패턴에 대한 호환성 근거다.
모든 과거 버전과 모든 호출 형태를 검증했다고 확대하지 않는다.
검증용 코드와 JAR는 /tmp/kotlinpoet-copy-compat 아래에 두었고 PR에는 포함하지 않았다.

## 15. JVM 브리지 메서드와 API 덤프

JVM의 메서드 식별에는 반환 타입을 포함한 descriptor가 사용된다.
Kotlin은 공변 반환 타입을 재정의할 때 부모 반환 타입으로 호출하는 경로를 위한
브리지 메서드를 생성한다.

API 덤프에는 ClassName에 대해 다음 두 메서드가 보인다.

```text
copy(Boolean, List): ClassName
copy(Boolean, List): TypeName  // synthetic bridge
```

정확한 JVM descriptor는 커밋의 kotlinpoet.api에 있다.
부모 TypeName의 기존 두 인자 메서드와 기본 인자를 처리하는 copy$default 진입점도 남아 있다.
변경은 부모 메서드의 final 해제와 다섯 타입의 재정의·브리지 추가다.
기존 오버로드나 인자 순서를 삭제·교체하는 접근을 피했다.

## 16. 자체검토 결과

이슈가 예로 든 ClassName만 수정하면 같은 문제를 가진 다른 타입을 남기게 된다.
TypeName의 모든 직접 하위 타입을 확인하고 복사가 지원되는 다섯 타입에 적용했다.
Dynamic은 기존 계약을 유지했다.

확인한 질문은 다음과 같다.

1. tags 없는 호출이 구체적인 타입으로 컴파일되는가.
2. 인자 없음·이름 있는 인자·위치 인자 모두 올바른 오버로드로 연결되는가.
3. 기존 tags 포함 오버로드는 유지되는가.
4. 새 두 인자 구현이 기존 세 인자 구현으로 위임하며 재귀하지 않는가.
5. 기본 인자가 상속돼 기존 호출 형식이 유지되는가.
6. 태그와 각 하위 타입의 고유 정보가 유지되는가.
7. 원본의 null 허용 여부와 애노테이션이 바뀌지 않는가.
8. 부모 타입으로 받은 호출과 Dynamic의 예외 동작이 유지되는가.
9. 기존에 컴파일한 Kotlin·Java 호출이 새 JAR에서 실행되는가.
10. 빌드 경고를 만드는 기존 형변환과 API 덤프를 함께 처리했는가.

필수로 추가 수정할 문제는 발견하지 못했다.
전체 빌드가 끝난 후 제품 코드에 추가 변경은 없었다.
PR 생성 뒤 두 번째 커밋은 changelog에 확정된 PR 번호를 연결하는 문서 변경뿐이다.

## 17. 제출과 규칙 준수

사용자에게 이슈를 기초부터 설명하고, 구현·검토 후 제출 승인을 받았다.
Git 작성자, 커미터, 원격 저장소, GitHub CLI 활성 계정을 각각 확인했다.
모두 GiHoon1123으로 제출했다.

구현 커밋과 PR 제목:

```text
Preserve subtype return types when copying type names
```

PR 본문에는 구체적인 실패 예시, 두 인자 copy의 재정의,
기존 오버로드 보존, 형변환 제거, 회귀 테스트와 빌드·호환성 결과를 짧게 적었다.
Fixes #1967로 이슈를 연결했다.
저장소 PR 템플릿을 유지했고, 사용자가 앞선 기여에서 제출한 CLA와 changelog 항목을 체크했다.
공개 문구는 실제 변경과 검증 결과를 중심으로 작성했고 임의의 공동 작성자 표기는 넣지 않았다.

PR 번호가 확정된 뒤 changelog의 Unreleased 항목에 (#2429)를 추가했다.
그 연결을 위한 두 번째 커밋도 포크 브랜치로 푸시했다.
별도 재현 파일·검증 JAR·로그는 코드 저장소 변경에 포함하지 않았다.

## 18. 제출 직후 상태

2026-11-13 제출 직후 조회 기준이다.

- PR은 OPEN이며 head는 584dd054다.
- 사람의 리뷰와 댓글은 아직 없었다.
- Task List Completed는 SUCCESS였다.
- mergeStateStatus는 BLOCKED였다.
- 이 시점에 조회된 목록에는 Gradle 빌드 CI 결과가 없었다.

로컬 빌드 성공을 원격 CI 전체 성공으로 표현하지 않는다.
리뷰·원격 검사·병합 경과는 이후 문서 끝에 추가한다.

## 19. 이번 작업에서 공부한 것

ClassName 객체는 User 객체 자체가 아니라 생성할 소스 안의 클래스 이름 정보다.
복사 결과의 런타임 클래스와 컴파일러가 사용하는 반환 타입은 다를 수 있다.
오버로드는 인자 구성으로 선택되고, 오버라이딩은 상속받은 함수를 재정의하는 것이다.
이번 문제는 두 개념이 함께 작동해 생긴 차이였다.

코틀린의 클래스·메서드는 기본 final이므로 재정의하려면 필요한 위치를 open으로 열어야 한다.
상속이 가능한 클래스라고 내부 메서드까지 모두 재정의 가능한 것은 아니다.
반환 타입을 하위 타입으로 구체화하면 기존 상위 타입 사용을 유지하면서
호출자가 하위 타입의 기능을 바로 사용할 수 있다.

최소 수정은 반환 타입 한 줄을 바꾸는 것만을 뜻하지 않았다.
같은 문제를 가진 타입 전체, 기존 API와 ABI, 새로 불필요해진 캐스트,
컴파일 단계에서 회귀를 잡는 테스트, 변경 이력과 API 덤프까지 포함해야 했다.
반대로 실제 복사 로직과 API 전체를 제네릭으로 재설계하는 작업은 하지 않았다.

## 참고

- [이슈와 메인테이너 논의](https://github.com/square/kotlinpoet/issues/1967)
- [PR 변경](https://github.com/square/kotlinpoet/pull/2429/files)
- [기존 ABI 복구 커밋](https://github.com/square/kotlinpoet/commit/97e9a1b5972d6eb02ff6faa71299a4b9296007c6)
- [회귀 테스트](https://github.com/square/kotlinpoet/blob/584dd0544f2aeea72557b7a9717e9a1dd226137a/kotlinpoet/src/jvmTest/kotlin/com/squareup/kotlinpoet/TypeNameCopyTest.kt)
- [부모 구현](https://github.com/square/kotlinpoet/blob/584dd0544f2aeea72557b7a9717e9a1dd226137a/kotlinpoet/src/jvmMain/kotlin/com/squareup/kotlinpoet/TypeName.kt)

## 2026-11-25 상태 확인

[PR #2429](https://github.com/square/kotlinpoet/pull/2429)은 한국 시간 11월 14일 02:31에 머지됐다. [머지 커밋 b8af3e6f](https://github.com/square/kotlinpoet/commit/b8af3e6f174d4a4b39a4de6f0764320f24d4a6d6)
