# Kotest #6290 — 음수 홀수를 짝수로 판정하던 문제

- [PR #6290](https://github.com/kotest/kotest/pull/6290) · 2026-11-18 GiHoon1123으로 제출.
- 포크 브랜치 `fix/negative-odd-matchers`, 커밋 `8d3601d`.

Kotest 6.2.7에서 `(-3).shouldBeOdd()`가 실패했다. Kotlin에서 `-3 % 2`는 `-1`인데, Int와 Long의 홀수 판정은 나머지가 `1`인 경우만 받아들였다. 반대 단언인 `(-3).shouldNotBeOdd()`도 잘못 통과했다.

두 타입 모두 나머지가 0이 아닌지 검사하도록 고쳤다. 공개 단언 두 개에 양수·음수 홀수와 짝수, 0, 정수 범위의 경계값을 넣어 확인했다. 수정 전 회귀 테스트 두 개가 실패했고, 수정 후 assertions-core JVM 테스트는 3,155개 통과, 3개 스킵이었다. 같은 모듈의 API 호환성 검사도 통과했다. JS와 Native는 실행하지 않았다.

2026-11-20 리뷰 반영. 일반 사용자 문서 변경을 되돌리고 Int의 기존 isOdd 테스트에 회귀 사례를 합쳤다. Long도 홀수 사례를 한 블록으로 모았다. 6.2.7 주의사항을 메서드 예시 옆으로 옮겼다. assertions-core JVM 테스트와 apiCheck 통과.

2026-11-20 19:29 KST에 alfonsoristorato가 `master`에 머지했다. [머지 커밋 1bdf1431](https://github.com/kotest/kotest/commit/1bdf1431c1e665db79eeb39dab9e6a5427eb8d49)
