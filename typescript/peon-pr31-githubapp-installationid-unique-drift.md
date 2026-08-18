# [OSS 기여] Peon-sh/Peon - `schema.prisma`가 실제 DB 유니크 제약 변경을 못 따라간 drift 수정 (PR #31) — 🟢 Merged

> 오픈소스 기여 기록 및 학습 노트. 이 문서는 PR #31(이슈 #29) 하나만 다룬다. **2026-08-03 제출, 2026-08-11 `hironate`가 승인 후 머지.**

---

## 0. 관련 링크 모음

| 종류 | 링크 | 상태 |
| --- | --- | --- |
| 대상 이슈 | https://github.com/Peon-sh/Peon/issues/29 | Merged로 닫힘 |
| 제출한 PR | https://github.com/Peon-sh/Peon/pull/31 | 🟢 Merged (2026-08-11) |

- 패키지: Peon-sh/Peon — 셀프호스팅 배포 플랫폼(Vercel/Heroku의 오픈소스 대체)
- 작업 브랜치: `fix/github-app-installation-id-unique-drift` (fork: GiHoon1123)
- 커밋: `7536554` — 1개 파일, +3/-1
- 검증: `prisma migrate dev --create-only`로 빈 마이그레이션 확인, 관련 유닛 테스트 7개 통과, tsc는 무관한 기존 에러만 남아있음을 stash로 확인

---

## 1. 이번 건은 평소와 다르게 찾은 후보다 — 메일로 기여 초대/추천을 받음

지금까지의 후보들은 전부 정해둔 검색 프롬프트(공식성 검증 → 활동성 확인 → 경쟁 PR 확인 → 재현)로 능동적으로 찾은 것들이었는데, 이번엔 다르다. **Peon 창업자(Hiren Kavad)가 GitHub 프로필을 보고 개인 메일로 직접 연락**해서, "우리가 막 오픈소스로 낸 배포 플랫폼인데 관심 있으면 별표/포크 눌러달라, 기여도 환영한다"는 콜드 아웃리치를 보내왔다.

메일 자체는 마케팅성 요청(별표/포크 부탁)이 주였지만, "무조건 밀어내지 말고 일단 확인부터 해보자"는 방침으로 저장소를 직접 검증했다:

- 생성 3주 정도 된 신생 프로젝트(2026-07-10 생성)지만 최근까지도 활발히 커밋됨
- **외부 기여자가 실제로 있고 머지도 됨** — `JamBalaya56562`(보안 관련 PR 3개: SSH 호스트 키 검증, shell injection 방지 등), `parth-advant019`(1개), 그리고 검토 시점에 `JiruGutema`가 올린 PR이 리뷰 대기 중이었음
- `SECURITY.md`가 있고, 자체 보안 감사(이슈 #17)를 돌려서 체계적으로 취약점을 추적 중
- 이슈 #29는 독립된 사용자(`mbaechtold`)가 실제로 `pnpm db:migrate`를 돌리다 겪은 문제를 재현 로그까지 통째로 올려둔, 구체적이고 조작되지 않은 리포트

**결론: 마케팅 메일과 프로젝트의 진위 여부는 별개 문제.** 별표/포크 요청 자체엔 응하지 않았지만, 프로젝트가 진짜 활동 있는 소프트웨어이고 외부 기여 track record도 있다는 걸 확인한 뒤 이슈 #29를 골라 진행했다.

---

## 2. Peon이 뭐 하는 소프트웨어인지

**"내 서버에 직접 배포하는, 오픈소스 Vercel/Heroku 대체품."** Heroku/Vercel처럼 `git push`로 자동 빌드·배포·도메인 연결·HTTPS 발급까지 해주는데, 그 대상 서버가 그 회사 인프라가 아니라 **내가 갖고 있거나 빌린 서버(Hetzner, DigitalOcean 등)**라는 게 핵심 차이. 구조는 Next.js 앱(대시보드+API) + SQS 큐를 처리하는 worker + cron 역할을 하는 schedule + 터미널 세션용 websocket 서버로 이루어져 있다.

---

## 3. 이슈 — `pnpm db:migrate`가 non-interactive 환경에서 실패함

이슈 #29는 정확히 이 커밋(`6783b6f`)에서 `pnpm db:migrate`(=`prisma migrate dev`)를 돌렸더니, 24개 기존 마이그레이션을 다 적용한 뒤 이런 경고를 내며 "새 마이그레이션을 만들지 확인해달라"고 인터랙티브 프롬프트를 띄운다는 내용이었다:

> A unique constraint covering the columns `[installationId]` on the table `GithubApp` will be added. If there are existing duplicate values, this will fail.

배포 자동화 스크립트처럼 사람이 답을 못 하는 환경(non-interactive)에서는 이 프롬프트 자체가 곧바로 에러로 죽어버린다.

### Docker로 직접 재현

Postgres 컨테이너를 띄우고, 리포지토리를 클론해서 `pnpm db:migrate`를 그대로 돌려봤더니 **이슈에 적힌 경고와 한 글자도 다르지 않게 재현됐다.**

---

## 4. 원인 — schema.prisma가 실제 DB 변경 이력을 못 따라감

`prisma/migrations/` 폴더의 SQL을 시간순으로 직접 읽어서 `GithubApp.installationId`에 무슨 일이 있었는지 추적했다:

1. `20260710180000_init`: `installationId`에 아무 제약도 없음
2. `20260715220000_github_app_kind_status`: 단순 인덱스 생성
3. `20260715230000_github_app_installation_unique`: 그 인덱스를 지우고, **컬럼 하나만으로 유니크**하게 만듦
4. `20260716140000_github_installation_per_workspace`: **그 단일 컬럼 유니크를 다시 지우고**, `(workspaceId, installationId)` **조합 유니크**로 바꿈 — "같은 GitHub 설치를 여러 워크스페이스에서 각자 연결 가능하게" 하려는 의도로 보임

근데 `schema.prisma`는 3번 상태(`installationId String? @unique`) 그대로 멈춰있었다. 즉 **실제 DB는 4번(워크스페이스별 유니크)까지 진행했는데, 설계도만 3번에 머물러 있던 것** — Prisma가 매번 "설계도대로 맞추려면 단일 컬럼 유니크를 다시 만들어야 한다"고 판단해서 계속 프롬프트를 띄운 거다.

**코드 자체가 이미 4번(워크스페이스별) 전제로 짜여있다는 근거**도 찾았다: `sources.ts`의 `findFirst({ where: { installationId, workspaceId } })`가 정확히 워크스페이스 단위로 조회하고 있었고, 테스트 파일 이름도 `platform-multi-install.test.ts`("여러 워크스페이스에 같은 installationId 설치")였다.

---

## 5. 수정

```diff
- installationId String?         @unique
+ installationId String?
  ...
+ @@unique([workspaceId, installationId])
  @@index([workspaceId])
  @@index([appId])
+ @@index([installationId])
```

**`@@index([installationId])`를 빠뜨릴 뻔한 실수**: 처음엔 `@@unique`만 추가했는데, `prisma migrate dev --create-only`로 다시 확인해보니 Prisma가 기존에 실제로 쓰이고 있던 단순 인덱스(`GithubApp_installationId_idx`)를 삭제 대상으로 잡았다. `setInstallationStatus()`가 `updateMany({ where: { installationId } })`로 이 인덱스를 그대로 쓰고 있어서, 그냥 뒀으면 성능 회귀가 생길 뻔했다 — `@@index([installationId])`를 명시적으로 추가해서 보존했다.

---

## 6. 검증

- **disable-and-reconfirm 두 번**: 스키마를 되돌려 원래 경고 재현 확인 → 복원 후 사라짐 확인
- `npx prisma migrate dev --create-only`로 drift 확인 → **완전히 빈 마이그레이션** — 새 SQL 파일이 하나도 필요 없는, 순수 스키마 선언 수정임을 증명
- 이슈에 있던 정확한 명령(`pnpm db:migrate`) 재실행 → "Already in sync, no schema change or pending migration was found."
- 관련 유닛 테스트 7개 통과
- tsc 에러 2건이 있었지만, `git stash`로 fix를 뺀 순정 상태에서도 똑같이 있는 무관한 기존 에러라는 것까지 확인

### 자체검토에서 추가로 확인한 것

- `githubApp.create()`/`upsert()` 호출 전체를 훑어서, `installationId` 하나만으로 unique 조회를 강제하는 곳이 없는지 확인
- `P2002`(unique 제약 위반) 에러코드를 잡아 처리하는 코드가 있는지 전체 검색 — 0건, 제약 범위가 바뀌어도 깨질 로직 없음
- 생성된 Prisma client가 `.gitignore`에 있어 커밋 대상이 아님을 확인 — 스키마 파일 하나만 고치면 충분

---

## 7. PR 본문에서 잡힌 AI 냄새

1차 초안에 이런 문장이 있었다:

> "so the composite unique is the correct direction — schema.prisma just needed to catch up."

**이게 전형적인 AI 요약체였다** — "이게 맞는 방향이고 스키마가 못 따라간 거다"라고 스스로 결론을 내려주는 문장. 사람이면 그냥 사실(코드가 이미 워크스페이스 단위로 조회한다)만 던지지, 그걸 "correct direction"이라고 정당화하지 않는다. 그 외에도 "which is exactly what #29 reports", "confirming schema and DB history match" 같은 자기 서술형 wrap-up 문장들이 있어서 전부 걷어내고, "Change:" 헤더도 빼서 바로 불릿으로 넘어가게 다듬었다.

---

## 8. 메일 답장 — 지금 보낼지 머지 후로 미룰지

PR을 올린 뒤, Hiren한테 보낼 답장 타이밍을 두고 짧게 논의했다. "PR이 머지된 다음에 답장하는 게 더 강한 증거 아닐까"라는 질문이 나왔는데:

- 초안 문구가 "PR을 올렸다"고만 했지 "머지됐다"고 과장하지 않아서, 지금 보내도 정직함
- 머지 시점은 메인테이너(수신자 본인)한테 달린 거라 기약이 없음
- 어차피 Hiren이 메인테이너라 PR 알림을 이미 받을 거고, 답장에 링크를 넣는 게 자연스러운 리마인드 역할도 함

**결론: 머지를 기다리지 않고 바로 답장 보냄.** 별표/포크 요청에는 응하지 않고, "이슈 확인하다가 하나 처리해서 PR 올렸다"는 사실만 링크와 함께 전달.

---

## 9. 핵심 교훈

1. **콜드 아웃리치 메일과 프로젝트의 실제 가치는 별개로 판단해야 한다.** "별표 눌러달라"는 요청 자체는 순수 마케팅이지만, 그 요청을 보낸 프로젝트가 진짜 활동 있는 소프트웨어일 수도 있다 — 둘을 섞어서 통째로 무시하거나 통째로 받아들이면 안 된다.
2. **schema.prisma 같은 "설계도" 파일은 실제 DB 마이그레이션 이력과 어긋날 수 있고, 그 어긋남 자체가 버그가 된다.** 이번 버그는 새로운 로직 결함이 아니라, 이미 몇 주 전에 의도적으로 바뀐 DB 상태를 설계도가 못 따라간 것뿐이었다.
3. **인덱스/제약을 고칠 때는 "빠진 것"만큼 "의도치 않게 지워지는 것"도 확인해야 한다.** `@@unique`만 추가하고 넘어갔으면 실제로 쓰이던 인덱스가 삭제될 뻔했다 — `--create-only`로 실제 diff를 매번 눈으로 확인하는 습관이 이걸 잡아냈다.
4. **PR 본문에서 "이게 맞는 방향이다"처럼 스스로 결론을 정당화하는 문장은 AI 냄새의 핵심 지표다.** 사실만 나열하고 결론은 리뷰어가 내리게 두는 게 사람이 쓰는 글에 더 가깝다.

---

## 10. 결과

- [x] PR #31 메인테이너 리뷰 결과 — `hironate`가 코멘트 없이 승인 후 바로 머지(2026-08-11)
- [ ] Hiren의 메일 답장 반응 (있다면)
