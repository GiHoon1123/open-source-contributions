# [OSS 기여] Peon-sh/Peon - 배포 중 suspend 요청이 컨테이너에 반영되지 않는 문제 (PR #83) — 🟢 Merged

## 관련 링크

| 구분 | 링크 | 상태 |
| --- | --- | --- |
| 대상 이슈 | https://github.com/Peon-sh/Peon/issues/82 | OPEN |
| 제출한 PR | https://github.com/Peon-sh/Peon/pull/83 | 🟢 Merged (2026-08-31) |

- 저장소: https://github.com/Peon-sh/Peon
- 작업 브랜치: `fix/recheck-suspension-during-deployment`
- 커밋: `3895583`
- 머지 커밋: `52aadc6a`
- 머지 시각: 2026-08-31 13:45 (KST)

## 문제

서비스를 suspend하면 API가 `suspendedAt`을 기록하고 컨트롤 작업을 큐에 넣는다. 배포 작업은 시작 전에만 이 값을 확인했다.

그 상태에서 이미지 빌드가 진행 중이면, 사용자가 suspend를 눌러도 이미 실행 중인 배포는 계속 진행된다. 빌드가 끝난 뒤 `docker compose up -d`를 실행하고 서비스 상태를 `RUNNING`으로 기록하므로, suspend된 서비스의 컨테이너가 다시 실행될 수 있다.

## 원인

`runDeployment()`이 처음 읽은 서비스 객체의 `suspendedAt`을 배포가 끝날 때까지 사용했다. `suspendedAt`은 다른 요청에서 바뀌지만, 메모리에 있는 `svc` 객체는 자동으로 갱신되지 않는다.

기존의 `assertNotCancelled()`는 deployment의 `status`만 확인하고 서비스의 desired state는 확인하지 않았다. 따라서 배포가 취소되지 않은 한 suspend 이후에도 다음 단계로 넘어갈 수 있었다.

## 수정

이미지 빌드와 pre-deploy 작업 뒤, 새 컨테이너를 시작하기 직전에 서비스의 `suspendedAt`을 다시 DB에서 조회한다.

- suspend가 확인되면 컨테이너를 시작하지 않고 배포를 `CANCELLED`로 정리한다.
- `compose up` 직후 suspend가 발생한 경우에는 compose 프로젝트를 중지한다.
- rolling 배포처럼 별도 Compose project name을 사용하는 경우에도 같은 project name으로 중지한다.
- readiness 대기 중 suspend가 발생한 경우에도 완료 처리 전에 다시 확인하고 compose project를 중지한다.
- 이 경우 서비스 상태를 `SUSPENDED`로 남겨 배포 완료 처리에서 `RUNNING`으로 덮어쓰지 않는다.
- preview 배포에서 같은 예외가 발생해도 deployment가 `IN_PROGRESS`에 남지 않도록 `CANCELLED`로 정리한다.
- compose 중지 명령이 실패하면 stdout/stderr와 종료 코드를 배포 로그에 남긴다.
- 기존의 배포 시작 전 suspend 확인과 취소 처리 흐름은 유지했다.

## 검증

- `pnpm test:unit`: 90개 파일, 488개 테스트 통과
- suspend 상태를 나타내는 `DeploymentSuspendedError` 테스트 추가
- `git diff --check` 통과

전체 타입 검사는 현재 checkout의 Prisma 생성물과 스키마가 맞지 않는 기존 오류 때문에 별도로 통과하지 못했다. ESLint와 Prettier도 로컬 의존성/설정 문제로 전체 검증이 불가능했다.
