# [OSS 기여] Peon-sh/Peon - disconnect 이후 SSH 연결이 풀에 되살아나는 문제 (PR #87) — 🟢 Merged

## 관련 링크

| 구분 | 링크 | 상태 |
| --- | --- | --- |
| 대상 이슈 | https://github.com/Peon-sh/Peon/issues/86 | OPEN |
| 제출한 PR | https://github.com/Peon-sh/Peon/pull/87 | 🟢 Merged (2026-08-27) |

- 저장소: https://github.com/Peon-sh/Peon
- 작업 브랜치: `fix/invalidate-disconnected-ssh-connect`
- 커밋: `8803ea8`
- 머지 커밋: `7cab4336`
- 머지 시각: 2026-08-27 15:45 (KST)

## 문제

SSH 풀은 서버별 연결 정보를 맵에 저장하고, 연결 중인 Promise도 같은 엔트리에 보관한다. 그런데 `disconnect()`가 실행 중인 연결을 기다리는 동안 맵 엔트리만 삭제한다.

이후 연결이 완료되면 `getConnection()`이 그 결과를 확인 없이 다시 맵에 넣는다. 서버 IP, 사용자, 키가 바뀌어 기존 연결을 버린 상황에서 오래된 연결이 되살아날 수 있고, 같은 키로 새 연결이 만들어진 경우 새 엔트리를 덮어쓸 수도 있다.

## 원인

연결 시도와 풀 엔트리 사이에 세대 정보가 없었다. `disconnect()`가 엔트리를 삭제해도 이미 실행 중인 Promise에는 그 사실이 전달되지 않아서, Promise가 resolve된 뒤에도 원래 연결 시도가 유효하다고 가정했다.

## 수정

연결 시도마다 `Symbol` 세대 토큰을 발급하고 풀 엔트리에 저장한다. 연결 Promise가 완료되면 현재 맵 엔트리의 토큰과 시작 당시 토큰을 비교한다.

- 엔트리가 삭제됐거나 토큰이 달라졌으면 연결을 dispose한다.
- 대기 중인 호출에는 `SSH connection was disconnected while connecting.` 오류를 반환한다.
- 실패 처리에서도 현재 엔트리가 같은 연결 시도일 때만 맵에서 삭제한다. 새 연결의 엔트리를 늦게 완료된 이전 시도가 지우지 못하게 하기 위한 조건이다.

## 검증

지연된 `NodeSSH.connect()`를 mock으로 만들어 연결 중 `disconnect()`를 호출하는 테스트를 추가했다. 연결 Promise가 나중에 완료돼도 연결이 풀에 재삽입되지 않고 dispose되며, 대기 중인 `exec()`가 실패하는지 확인한다.

- `pnpm test:unit`: 89개 파일, 480개 테스트 통과
- SSH pool 관련 테스트 4개 통과
- `git diff --check` 통과
