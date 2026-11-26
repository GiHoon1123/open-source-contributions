# libuv #5443 — kqueue 파일 감시 등록 시점 수정

작성일: 2026-11-13. 제출 계정: GiHoon1123. 현재 상태: MERGED (2026-11-25).

- [PR #5443](https://github.com/libuv/libuv/pull/5443)
- [기존 PR #5309](https://github.com/libuv/libuv/pull/5309)
- [이전 분석·구현·리뷰 기록](../close/libuv-node-53111-macos-fs-watch-immediate-event.md)
- [관련 Node.js 이슈 #53111](https://github.com/nodejs/node/issues/53111)
- 포크 브랜치: GiHoon1123/libuv의 fix/fs-event-solaris-5309
- 대상 브랜치: v1.x
- 제출 head: d590bd58ea31aaa2d406efef3f3e6e0ff58fb07f

## 다시 제출한 이유

기존 PR은 11월 10일 작성자가 닫았고 원본 포크도 삭제돼 있었다.
다시 포크한 뒤 기존 PR 참조에서 커밋을 복구했지만 GitHub가 재오픈을 거부했다.
따라서 기존 커밋 네 개를 그대로 보존한 후속 PR을 제출했다.
메인테이너의 코드 수정과 커밋 정보도 보존했다.

## 문제와 변경

kqueue 파일 watcher 등록이 다음 이벤트 루프 poll까지 미뤄지면,
uv_fs_event_start() 직후 동기적으로 발생한 파일 변경을 놓칠 수 있었다.
등록 시점을 앞당기고 watcher 상태도 맞춰 중복 등록을 방지한다.
메인테이너가 추가한 EV_CLEAR 변경은 콜백 내부에서 발생한 파일 변경이
watcher 재등록 사이에 누락되지 않도록 등록을 유지한다.

기존 PR 이후 추가한 변경은 test/test-fs-event.c의 5줄이다.
fs_event_watch_file_modified_in_cb를 __sun 플랫폼에서 제외하고,
Solaris에도 같은 문제가 있는 것으로 보인다는 이유를 주석에 남겼다.
이는 [vtjnash의 답변](https://github.com/libuv/libuv/pull/5309#issuecomment-6827776432)을 반영한 것이다.
Solaris backend를 수정하거나 다른 플랫폼의 테스트를 제외하지 않았다.

## 검증과 한계

- macOS에서 cmake -B build -DBUILD_TESTING=ON 및 cmake --build build -j 4 성공.
- build/uv_run_tests_a로 fs_event_ 테스트 20개 실행: 모두 통과, 스킵 없음.
- git diff --check 통과.
- 최신 upstream/v1.x와 merge-tree 검사에서 충돌 없음.
- 최신 upstream과 합친 트리의 테스트, 전체 libuv 테스트 스위트, Solaris 실행은 수행하지 않았다.
- 로컬 결과와 새 PR의 원격 CI 결과는 별개이며 제출 직후 CI 완료를 주장하지 않는다.

## 다른 PR과의 관계

[PR #5316](https://github.com/libuv/libuv/pull/5316)은 기존 #5309 위에 쌓인 후속 작업이다.
기존 네 커밋에 파일 이벤트 종류를 바로잡는 별도 커밋을 더한 것이므로,
그 추가 변경을 이번 PR에 섞지 않았다. 새 PR 본문에 이 관계를 명시했다.
기존 PR의 승인을 새 PR의 승인으로 취급하지 않는다.

## 2026-11-25 머지

vtjnash의 승인 후 PR #5443이 2026-11-25 13:30 KST에 libuv의 `v1.x`에 머지됐다. 머지 커밋은 `de4c8602d54255c4e7f3fa91dd6f44b682679724`다. [PR #5443](https://github.com/libuv/libuv/pull/5443) · [머지 커밋](https://github.com/libuv/libuv/commit/de4c8602d54255c4e7f3fa91dd6f44b682679724)
