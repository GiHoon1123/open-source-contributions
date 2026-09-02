# [OSS 기여] Peon-sh/Peon - stale stop 작업이 suspend 상태를 덮어쓰는 문제 (PR #85) — 🟢 Merged

## 관련 링크

| 구분 | 링크 | 상태 |
| --- | --- | --- |
| 대상 이슈 | https://github.com/Peon-sh/Peon/issues/84 | OPEN |
| 제출한 PR | https://github.com/Peon-sh/Peon/pull/85 | 🟢 Merged (2026-08-31) |

- 저장소: https://github.com/Peon-sh/Peon
- 작업 브랜치: `fix/preserve-suspended-status-on-stale-stop`
- 커밋: `1b7d3c2`
- 머지 커밋: `88d28295`
- 머지 시각: 2026-08-31 13:52 (KST)

## 문제

서비스에서 stop을 누른 뒤 suspend를 누르면 두 컨트롤 작업이 큐에 들어간다. 큐 처리 순서가 바뀌어 stop 작업이 나중에 실행될 수 있는데, 호스트에서 `docker compose stop`을 실행하는 것 자체는 안전했다.

문제는 명령 실행 후 서비스 상태를 항상 `STOPPED`로 기록한 점이다. 이후 `suspendedAt`은 그대로 남아 desired state는 suspend인데, 저장된 status는 stopped인 상태가 된다. API의 status reconciliation이 일부 응답에서는 이를 `SUSPENDED`로 보정하지만, 원본 DB 상태를 읽는 경로에서는 값이 어긋난다.

## 원인

`controlService()`가 stop 처리 후 다음과 같이 조건 없이 상태를 기록했다.

```ts
await prisma.service.update({
  where: { id: serviceId },
  data: { status: statusAfterControl(action) },
});
```

`statusAfterControl('stop')`은 항상 `STOPPED`를 반환하고, 상태 기록 시점의 `suspendedAt`을 확인하지 않았다.

## 수정

stop 상태 기록을 두 단계 조건부 갱신으로 바꿨다.

1. `suspendedAt: null`인 서비스만 `STOPPED`로 갱신한다.
2. 첫 갱신이 적용되지 않으면 `suspendedAt`이 설정된 서비스를 `SUSPENDED`로 갱신한다.

두 갱신 모두 DB 조건에 포함되므로 stop 작업이 실행되는 순간 suspend가 먼저 기록돼 있으면 `STOPPED`가 `SUSPENDED`를 덮어쓰지 않는다. compose stop 명령과 start/restart/resume 처리에는 영향을 주지 않았다.

## 검증

stale stop을 재현하는 테스트를 추가했다. 서비스가 이미 suspend된 상황에서 첫 번째 조건부 갱신이 적용되지 않고, 두 번째 갱신이 `SUSPENDED`를 기록하는지 확인한다.

- `pnpm test:unit`: 88개 파일, 480개 테스트 통과
- 관련 control service 테스트 25개 통과
- `git diff --check` 통과
