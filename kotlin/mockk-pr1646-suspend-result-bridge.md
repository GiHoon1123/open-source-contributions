# MockK #1646 — 제네릭 인터페이스로 호출한 suspend 함수의 Result 반환

- [이슈 #1639](https://github.com/mockk/mockk/issues/1639) · [PR #1646](https://github.com/mockk/mockk/pull/1646)
- 2026-11-20 GiHoon1123으로 제출. 포크 브랜치 `fix/suspend-result-bridge`, 커밋 `9a43e8bd`.

구현 클래스에 설정한 응답을 제네릭 인터페이스로 호출하면 성공값 자리에 Result 객체가 남아 ClassCastException이 발생했다. JVM 브리지의 마지막 매개변수가 Continuation이면 브리지 본문이 실행되도록 두고, 위임받은 구현 메서드에서 MockK가 응답하게 했다.

제출 직전 upstream master(9ce7b518)에서도 수정 전 오류를 재현했다. 수정 후 성공값과 실패 예외, 직접 호출과 인터페이스 호출, 호출 횟수를 확인하는 테스트 다섯 개와 Gradle check가 통과했다. 로컬에 SDK가 없어 Android 모듈은 제외됐다.

2026-11-20 22:46 KST에 Raibaz가 `master`에 머지했다. [머지 커밋 87168d46](https://github.com/mockk/mockk/commit/87168d469ad99e7b91090ececd2cb13662bbd882)
