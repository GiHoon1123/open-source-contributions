# [OSS 기여] Peon-sh/Peon - 예약 작업 실패 시 실제 오류 출력이 사라지는 문제 (PR #81) — 🟢 Merged

## 관련 링크

| 구분 | 링크 | 상태 |
| --- | --- | --- |
| 대상 이슈 | https://github.com/Peon-sh/Peon/issues/80 | OPEN |
| 제출한 PR | https://github.com/Peon-sh/Peon/pull/81 | 🟢 Merged (2026-08-27) |

- 저장소: https://github.com/Peon-sh/Peon
- 작업 브랜치: `fix/keep-scheduled-task-error-output`
- 커밋: `8ba3a37`
- 머지 커밋: `9ff6834a`
- 머지 시각: 2026-08-27 15:08 (KST)

## 문제

예약 작업은 SSH로 컨테이너 안의 명령을 실행하고, 결과의 stdout과 stderr를 `ScheduledTaskExecution`에 저장한다. 명령이 성공하면 이 기록이 그대로 남지만, 종료 코드가 0이 아니면 바로 뒤에서 `Task exited non-zero.` 예외를 던진다.

예외는 같은 함수의 `catch`에서 다시 실행 기록에 저장된다. 이때 방금 저장한 실제 명령 출력 대신 예외 메시지만 기록하므로, 실패 원인이 데이터베이스 기록에서 사라진다. 실패 알림도 같은 예외 메시지를 사용해서 실제 stderr를 보여주지 못한다.

## 원인

```ts
await prisma.scheduledTaskExecution.update({
  data: { status: 'FAILED', message: (res.stdout + res.stderr).slice(0, 10000) },
});
if (res.code !== 0) throw new Error('Task exited non-zero.');
```

실패 결과를 한 번 기록한 뒤, 실제 결과와 관계없는 일반적인 오류를 throw하는 구조였다. catch 블록은 `err.message`를 다시 저장하므로 두 번째 기록이 첫 번째 기록을 덮어쓴다.

## 수정

종료 코드가 0이 아닐 때 stdout과 stderr를 같은 10,000자 제한으로 잘라 예외 메시지로 사용하도록 바꿨다. 출력이 전혀 없는 실패만 기존의 `Task exited non-zero.` 메시지를 사용한다.

이렇게 하면 기존 catch 경로를 그대로 사용하면서 실행 기록과 실패 알림 모두 실제 명령 출력을 유지한다. 성공 경로와 출력 길이 제한은 변경하지 않았다.

## 검증

실제 stdout과 stderr를 반환하는 실패 상황을 테스트로 추가했다. 테스트는 다음을 확인한다.

- `runScheduledTask()`가 실제 출력과 함께 실패하는지
- 실행 기록의 최종 `message`가 실제 출력인지
- 실패 알림에 stderr가 포함되는지

검증 결과:

- `pnpm test:unit`: 89개 파일, 480개 테스트 통과
- 추가 회귀 테스트 통과
- ESLint는 로컬 의존성에서 `@typescript-eslint/parser`를 찾지 못해 실행하지 못함
- 타입 검사는 기존 Prisma 생성물과 현재 스키마가 어긋난 오류가 있어 전체 통과하지 못함
