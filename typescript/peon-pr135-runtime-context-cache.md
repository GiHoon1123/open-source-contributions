# [OSS 기여] Peon-sh/Peon - 서버 변경 뒤 이전 서버를 계속 사용하는 런타임 캐시 (PR #135) — 🟢 Merged

## 관련 링크

| 구분 | 링크 | 상태 |
| --- | --- | --- |
| 저장소 | https://github.com/Peon-sh/Peon | 머지 완료 |
| 관련 이슈 | https://github.com/Peon-sh/Peon/issues/134 | CLOSED |
| 제출한 PR | https://github.com/Peon-sh/Peon/pull/135 | MERGED (2026-09-25) |

- 작업 브랜치: `fix/invalidate-runtime-context-on-server-change`
- 대상 브랜치: `staging`
- 커밋: `a58b3cd`
- 커밋 제목: `fix(service): invalidate runtime context when server changes`
- PR 생성일: 2026-09-23 (KST)
- 머지 커밋: `fd4e1b0` (`staging`)

## Peon에서 말하는 서버와 서비스

Peon은 배포 대상 서버를 등록하고, 그 서버에 애플리케이션·데이터베이스·Docker Compose 서비스를
배포하고 관리하는 플랫폼이다.

- **서버(Server)**: SSH로 접속하는 실제 배포 대상 컴퓨터다. VPS, 클라우드 VM, 물리 서버 등이
  해당한다. Docker 컨테이너는 이 컴퓨터에서 실행된다.
- **서비스(Service)**: Peon이 배포하고 관리하는 애플리케이션 단위다. Git 애플리케이션, Docker
  이미지, 데이터베이스, Compose 프로젝트 등이 서비스가 될 수 있다.
- **컨테이너(Container)**: 서비스가 서버에서 실제로 실행되는 Docker 프로세스다.

예를 들어 `my-api` 서비스가 `production-1` 서버에 배정되어 있으면, Peon은 해당 서버에 SSH로
접속해 `my-api` 컨테이너의 로그를 읽거나 명령을 실행한다.

```text
Peon 서비스(my-api)
  └─ 배정된 서버(production-1)
       └─ Docker 컨테이너(my-api-...)
```

서비스를 다른 서버로 옮기면 이후 로그, 상태 조회, 터미널 명령은 새 서버를 사용해야 한다.

## 문제

서비스 런타임 코드는 서버 접속 정보와 컨테이너 이름을 매번 계산하지 않고 잠시 캐시한다.
`src/services/internal/service/runtime.ts`의 `containerContext()`가 서비스 ID를 키로 다음 정보를
저장한다.

- SSH 대상 서버의 주소, 포트, 사용자, 복호화된 private key
- 현재 활성 컨테이너 이름
- raw Compose 서비스에서 계산한 컨테이너 정보

캐시 유효 시간은 60초다.

```ts
const contextCache = new Map<string, { ctx: ContainerContext; expires: number }>();
const CACHE_TTL_MS = 60_000;
```

배포가 컨테이너를 교체할 때는 이미 `ServiceRuntime.invalidate(serviceId)`를 호출한다. 그러나
서비스의 `serverId`가 바뀌는 설정 경로에서는 이 캐시를 지우지 않았다.

그 결과 다음 순서가 가능했다.

1. 서비스가 서버 A에 배정되어 있다.
2. 로그·상태·명령 실행 중 하나가 `containerContext()`를 호출한다.
3. 서버 A의 SSH 대상과 컨테이너 정보가 서비스 ID 기준으로 캐시된다.
4. 사용자가 서비스의 서버를 서버 B로 변경한다.
5. DB에는 서버 B가 저장되지만, 캐시는 아직 60초 동안 살아 있다.
6. 그 사이의 로그·상태·`exec` 요청이 서버 B가 아니라 서버 A를 사용한다.

서비스를 옮긴 직후 이전 서버의 로그를 보거나 이전 서버의 컨테이너에 명령을 실행할 수 있는
문제다. 서비스의 데이터가 손상된다는 뜻은 아니지만, 화면에 보이는 상태와 실제 요청 대상이
달라져 운영자가 잘못된 서버를 조작할 수 있다.

## 코드 흐름

### 로그·명령 실행이 런타임 컨텍스트를 사용하는 과정

서비스 명령 실행 API는 권한을 확인한 뒤 `ServiceRuntime.exec(serviceId, command)`를 호출한다.

```text
POST /api/services/:serviceId/exec
  → ServiceRuntime.exec()
    → containerContext(serviceId)
      → 서비스와 서버 정보를 읽거나 캐시에서 반환
    → SSH 연결
      → docker exec ...
```

로그와 상태 조회도 같은 `containerContext()`를 공유한다. 따라서 캐시가 잘못되면 특정 API 하나가
아니라 여러 런타임 기능이 동시에 이전 서버를 바라본다.

### 서버 배정이 바뀌는 두 경로

서비스의 서버 정보는 다음 두 경로에서 바뀔 수 있다.

1. 일반 서비스 수정 경로인 `lifecycle.update()`
2. 서버만 따로 배정하는 `lifecycle.setServer()`

처음에는 `update()`만 보면 충분해 보이지만, 실제로는 두 경로 모두 캐시를 무효화해야 한다. 한
경로만 고치면 UI나 내부 호출자가 어떤 API를 사용하느냐에 따라 버그가 남는다.

## 어떻게 발견했나

코드를 읽을 때 상태 일관성을 기준으로 후보를 좁혔다.

> 상태가 바뀐 뒤에도 캐시나 메모리가 옛날 값을 계속 사용하지 않는가?

먼저 Peon의 핵심 흐름을 서비스 조회, 서버 배정, 컨테이너 실행, 로그·터미널·명령 실행으로
나눴다. 그다음 로그와 명령 실행이 어디서 서버 정보를 얻는지 호출자를 따라갔다.

`ServiceRuntime.exec()`와 `logs()`가 공통으로 `containerContext(serviceId)`를 호출했고, 그 안에서
서비스별 캐시를 발견했다. 이후 `ServiceRuntime.invalidate()`의 모든 호출처를 검색했다.

- 배포가 끝난 뒤 컨테이너가 바뀌는 경로에서는 호출됨
- 서비스 `serverId`를 수정하는 `update()`에서는 호출되지 않음
- 별도 서버 배정 함수인 `setServer()`에서도 호출되지 않음

마지막으로 `updateServiceSchema`에 `serverId`가 실제 입력 필드로 포함되어 있는지 확인했다. 따라서
이 문제는 사용되지 않는 내부 함수가 아니라 실제 서비스 수정 요청으로 발생할 수 있는 경로였다.

## 재현

실제 서버나 운영 DB를 변경하지 않고 Vitest에서 런타임 경로를 재현했다. Prisma 조회, SSH 연결,
암호 복호화와 Docker helper는 mock으로 대체하고, 서비스 조회 결과만 서버 A와 서버 B로 다르게
반환했다.

재현 순서는 다음과 같다.

```text
첫 번째 resolveContainer()
  → DB에서 서버 A 조회
  → 서버 A 정보가 캐시에 저장됨

서비스가 서버 B로 바뀐 상태를 가정

두 번째 resolveContainer()
  → 캐시가 유효하므로 DB를 다시 조회하지 않음
  → 서버 A 정보 반환
```

검증한 값은 세 가지다.

- 첫 번째 결과의 호스트가 서버 A인지
- 두 번째 결과도 서버 A로 남는지
- 두 번째 호출에서 Prisma 조회가 다시 일어나지 않는지

수정 전 임시 재현 테스트는 버그가 “서버 B를 읽었는데 잘못 반환하는 문제”가 아니라 “서버 B를
읽는 단계 자체를 캐시가 건너뛰는 문제”임을 보여줬다.

## 수정

변경 범위를 캐시 무효화 호출 두 곳으로 제한했다.

### `update()`

일반 수정 요청에서 `serverId`가 전달됐고 기존 값과 달라졌을 때만 캐시를 지운다.

```ts
if (input.serverId !== undefined && input.serverId !== existing.serverId) {
  ServiceRuntime.invalidate(serviceId);
}
```

이 조건은 다음 경우를 구분한다.

- 서버 A에서 서버 B로 변경: 무효화
- 서버 배정을 해제해 `null`로 변경: 무효화
- 서버를 바꾸지 않고 다른 서비스 설정만 수정: 불필요한 무효화 없음
- `serverId` 필드가 요청에 없음: 무효화 없음

### `setServer()`

`setServer()`는 함수 자체가 서버 배정 작업이므로 DB 업데이트 직후 항상 해당 서비스의 캐시를
무효화한다.

이렇게 하면 다음 런타임 요청이 `containerContext()`를 다시 계산하고, DB에 저장된 새 서버의 SSH
대상과 컨테이너 정보를 사용한다.

### 왜 캐시 구조를 바꾸지 않았나

이번 문제의 원인은 캐시를 사용하는 것 자체가 아니라 상태 변경 시 무효화가 빠진 것이었다. TTL을
줄이거나 캐시를 완전히 제거하면 증상을 줄일 수는 있지만, 불필요한 SSH 대상 재계산과 원래 의도된
캐시 동작을 바꾸게 된다. 기존 `invalidate()` helper를 서버 배정 변경 경로에 연결하는 것이 가장
작고 저장소의 기존 구조에도 맞는 수정이었다.

## 테스트

서버 A의 컨텍스트를 캐시한 뒤 `ServiceRuntime.invalidate()`를 호출하고, 다음 조회에서 서버 B의
호스트를 다시 읽는 런타임 테스트를 추가했다. 이 테스트만으로는 호출자가 무효화를 빼먹어도
통과할 수 있으므로, 별도로 서비스 lifecycle 테스트를 추가해 `update()`와 `setServer()`가 실제로
`ServiceRuntime.invalidate(serviceId)`를 호출하는지도 확인했다.

테스트가 확인하는 것은 단순히 같은 문자열이 나오는지가 아니다. 첫 번째 조회와 두 번째 조회 사이에
캐시를 무효화했을 때 Prisma 조회가 두 번 실행되고, 두 번째 결과의 호스트가 실제로 새 서버로
바뀌는지를 확인한다.

실행 결과:

- `pnpm test`: **98개 테스트 파일, 525개 테스트 통과**
- `git diff --check`: 통과

로컬 `pnpm typecheck`도 실행했지만 이번 변경과 무관한 기존 오류가 있었다. `posthog-js` 타입 누락,
Prisma 생성 타입과 스키마 불일치, Stripe API 버전 타입 불일치 등이 이미 여러 파일에서 발생했다.
따라서 해당 명령의 실패를 이번 변경의 회귀로 기록하지 않았다.

로컬 ESLint는 설치된 `node_modules`에서 `@typescript-eslint/parser`를 찾지 못해 실행되지 않았다.
또한 Prettier 설정이 `app/globals.css`를 참조하지만 현재 저장소의 파일은 `src/app/globals.css`에
있어서 기본 formatter 검증도 환경 오류로 중단됐다. 코드 변경 자체는 `git diff --check`와 전체
Vitest 결과로 확인했다.

## 이슈와 PR을 만든 과정

최신 `upstream/staging`을 기준으로 중복 이슈와 PR을 검색했다. 같은 런타임 캐시 문제를 다루는 열린
이슈나 PR은 없었다.

그 뒤 기존 작업 브랜치와 섞이지 않도록 다음 브랜치를 새로 만들었다.

```text
fix/invalidate-runtime-context-on-server-change
```

Peon의 기여 가이드에 맞춰 PR 대상은 `main`이 아니라 `staging`으로 지정했다. 이슈 #134에는 증상,
60초 캐시, 영향받는 기능과 수정 방향을 적었고, PR #135에는 실제 변경 내용과 전체 테스트 결과를
간단히 기록했다.

커밋에는 별도 공동 저자나 자동 생성 attribution을 넣지 않았다. 커밋 작성자와 PR 작성자는
`GiHoon1123` 계정으로 확인했다.

## 현재 상태와 남은 확인

- 이슈 #134: OPEN
- PR #135: OPEN
- PR 대상 브랜치: `staging`
- 로컬 전체 테스트: 통과
- GitHub CI: PR 생성 직후 Unit tests와 API integration tests가 실행 중이었고, Secret scan과
  Dependency audit은 대기 중이었다.

PR이 merge되면 이 문서를 `typescript/merge/`로 옮기고 머지 커밋과 최종 CI 결과를 갱신한다. PR이
수정 요청을 받으면 현재 문서 아래에 리뷰 라운드와 반영 내용을 시간순으로 덧붙인다.

## 이번 작업에서 남긴 기준

1. 캐시를 발견하면 값을 쓰는 지점뿐 아니라 값을 바꾸는 모든 경로와 무효화 호출처를 함께 검색한다.
2. `serverId`처럼 같은 객체의 위치를 바꾸는 상태는 DB 업데이트만으로 완료되지 않는다. 메모리 캐시,
   연결 풀, 컨테이너 이름처럼 파생된 값도 함께 갱신해야 한다.
3. TTL은 정합성 보장이 아니다. 60초 뒤에 자연 만료된다는 사실이 서버 변경 직후의 오래된 참조를
   안전하게 만들지는 않는다.
4. 서비스 변경 경로가 여러 개라면 하나만 고치지 말고 `update()`와 전용 setter를 모두 확인한다.
5. mock 재현 테스트는 외부 서버 없이도 캐시의 상태 전이를 검증할 수 있지만, 실제 SSH 연결과 운영
   DB를 검증한 것으로 과장해서는 안 된다.
6. 작은 수정이라도 수정 전 재현, 수정 후 회귀 테스트, 전체 테스트 결과를 모두 남겨야 다음 리뷰에서
   변경의 경계를 설명할 수 있다.

## 메인테이너 리뷰 반영

첫 번째 PR 커밋 이후 메인테이너가 캐시가 프로세스마다 따로 존재한다는 점을 짚었다. 일반 API는
Next.js 앱 프로세스에서 실행되지만, WebSocket 터미널은 별도 `socket` 컨테이너의
`worker/terminal-server.ts`에서 실행된다. 두 프로세스가 같은 `ServiceRuntime` 모듈을 읽더라도
메모리 맵은 공유되지 않으므로, 앱 프로세스에서 호출한 `invalidate()`만으로는 이미 실행 중인
터미널 프로세스의 60초 캐시를 지울 수 없다.

리뷰에는 두 번째 경쟁 조건도 있었다. 오래된 `containerContext()` 조회가 서버 변경과
`invalidate()`보다 먼저 시작된 경우, 조회가 나중에 끝나면서 이전 서버 정보를 다시 캐시에 넣을 수
있다. 단순히 무효화 호출을 추가하는 것만으로는 이 순서를 막을 수 없다.

### 리뷰 반영 내용

1. 캐시 적중 때 `service.serverId`를 다시 읽는다. 캐시된 SSH 대상의 서버 ID와 현재 DB 값이
   다르면 캐시를 버리고 전체 컨텍스트를 다시 계산한다. 이 확인은 별도 터미널 프로세스처럼
   `invalidate()`를 직접 받지 못하는 프로세스에서도 서버 이동을 감지하기 위한 것이다.
2. 서비스별 세대 번호를 둔다. `invalidate()`가 호출될 때 세대를 증가시키고, 조회 시작 시 저장한
   세대와 컨텍스트 계산이 끝난 뒤의 세대가 다르면 결과를 캐시에 넣지 않는다. 이전 조회가 새
   서버용 캐시를 덮어쓰는 것을 막는다.
3. `update()`에서 서버뿐 아니라 컨테이너 선택에 영향을 주는 `name`, `dockerComposeRaw`,
   `isRawComposeDeploymentEnabled`의 변경도 캐시를 무효화한다.
4. `setServer()`의 무효화는 그대로 유지한다. 전용 서버 배정 경로를 우회하지 않도록 두 경로를
   모두 테스트한다.

### 추가 테스트

기존 lifecycle 호출 테스트에 다음 검증을 추가했다.

- 캐시가 남아 있는 상태에서 다른 프로세스의 DB 값이 서버 B로 바뀌면 서버 B를 다시 읽는지
- `invalidate()`와 오래된 조회가 겹쳐도 오래된 결과가 캐시에 재삽입되지 않는지
- `update()`에서 서버명, raw Compose, raw Compose 설정 변경이 모두 무효화를 호출하는지

수정 후 로컬 전체 테스트 결과는 **98개 테스트 파일, 528개 테스트 통과**다. 리뷰 반영 커밋은
`90fdf504b9ab90389ec64e09039187d9558ebe92`이며, PR 본문과 리뷰 스레드에 변경 내용과 테스트
결과를 짧게 남겼다.

리뷰 반영 후 GitHub CI에서는 Secret scan과 Dependency audit이 먼저 통과했고, Unit tests와 API
integration tests는 재실행 중이다. 최종 결과가 나오면 이 문서의 상태 기록도 이어서 갱신한다.

## 자체 검토 추가 기록

리뷰 반영 뒤 테스트가 실제 경쟁 조건을 잡는지 다시 확인했다. 세대 번호 비교를 임시로 제거하고
테스트를 실행하자, 무효화 직전에 시작한 조회가 오래된 컨테이너 이름을 다시 반환하는 실패가
재현됐다. 세대 번호를 복구한 뒤에는 같은 테스트가 다시 통과했다.

이 검증으로 다음을 구분했다.

- 서버 ID가 달라졌을 때 캐시 적중 검사가 새 서버를 찾아가는지
- 서버 ID는 같지만 컨테이너 선택 결과가 바뀐 상태에서 오래된 조회가 새 캐시를 덮어쓰지 않는지
- `update()`와 `setServer()`가 실제로 무효화를 호출하는지

자체 검토 후 테스트 파일의 경쟁 조건 사례를 보강했고 최종 커밋은 `6b939e0`으로 갱신했다.
로컬 전체 테스트는 다시 **98개 테스트 파일, 528개 테스트 통과**를 확인했다.
