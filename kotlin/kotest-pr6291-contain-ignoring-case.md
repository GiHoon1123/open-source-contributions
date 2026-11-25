# Kotest #6291 — 대소문자를 무시한 검색에서 원본 위치 표시

- [PR #6291](https://github.com/kotest/kotest/pull/6291) · 2026-11-18 GiHoon1123으로 제출.
- 포크 브랜치 `fix/contain-ignoring-case-index`, 커밋 `20be65c`.

`"İx" shouldNotContainIgnoringCase "X"`는 실패 메시지에서 x의 위치를 2로 표시했다. 문자열 전체를 소문자로 바꾸면 İ가 i와 결합 점으로 풀려서 x의 위치가 밀린다. 원본에서 `indexOf(substr, ignoreCase = true)`로 검색하도록 바꿔 실제 위치인 1을 표시한다. 결합 점만 검색하면 기존에는 변환 중에 생긴 점을 찾았지만 이제 원본의 İ와 일치하지 않는다.

위치를 확인하는 회귀 테스트가 수정 전에 실패하고 수정 후 통과했다. 유니코드 문자의 일치 여부도 양쪽 단언으로 검사했다. assertions-core JVM 테스트 3,154개 통과, 3개 스킵. 모듈 API 호환성 검사를 통과했고 로컬 검증 범위는 JVM이다.

2026-11-20 리뷰 반영. 개발 브랜치 설명을 삭제하고 6.2.7에서 유니코드 검색 위치가 밀릴 수 있다는 주의사항을 shouldContainIgnoringCase 예시 옆 한 줄로 줄였다.

2026-11-20 09:44 KST에 alfonsoristorato가 `master`에 머지했다. [머지 커밋 9bcd6a21](https://github.com/kotest/kotest/commit/9bcd6a21df532b5983a2cc62b0235269fecd57f9)
