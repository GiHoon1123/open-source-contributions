# ioredis #2271 — 여러 클러스터 노드에 명령을 보내는 README 예제 수정

> 현재 상태 (2026-11-03 확인): **MERGED**. 한국시간 2026-11-02 22:32:10에 반영됐다.

## 1. 작업 정보

| 항목 | 내용 |
| --- | --- |
| 저장소 | [redis/ioredis](https://github.com/redis/ioredis) |
| PR | [#2271](https://github.com/redis/ioredis/pull/2271) |
| 제목 | `docs: fix multi-node cluster examples` |
| 작성 계정 | `GiHoon1123` |
| 제출 커밋 | `6e5c7de37f9ba0533e89845df294dc3cf4d23cc1` |
| 머지 커밋 | [`87204176fa`](https://github.com/redis/ioredis/commit/87204176fa8abd6add5535d4170fed8c09aa754d) |
| 머지한 계정 | `PavelPashov` |
| 변경 파일 | `README.md` 한 개, 5줄 추가·9줄 삭제 |

문서 저장소를 pull한 뒤에도 기존 #2271 기록은 없었다. 이 문서는 PR 본문과 실제 diff,
머지 메타데이터를 읽고 작성한 기록이다. 작업 당시 로컬 실행 로그를 새로 확보하거나
같은 검증을 다시 실행한 것은 아니다. 아래 검증 내용은 PR 작성자가 제출 당시 보고한 범위다.

## 2. 예제가 설명하려던 동작

ioredis는 Node.js에서 Redis에 접속하는 클라이언트다. Redis Cluster는 여러 노드에
키를 나눠 저장하므로, 하나의 노드에 명령을 보내는 것과 모든 master에 보내는 것은 다르다.

`cluster.nodes(role)`은 역할에 맞는 Redis 클라이언트 객체들의 배열을 반환한다.
예제에서 사용하는 `"master"`는 쓰기와 슬롯별 데이터 처리를 담당하는 노드,
`"slave"`는 해당 API가 사용하는 replica 역할 이름이다.

```js
const masters = cluster.nodes("master");
const requests = masters.map((node) => node.keys("*"));
```

`masters`는 클라이언트 객체의 배열이고, `requests`는 각 명령이 반환한 Promise의 배열이다.
`.map()`은 각 배열 원소에 함수를 적용해 새 배열을 만든다. 그 배열 자체가 Promise로
바뀌는 것은 아니다.

## 3. 첫 번째 문제: then을 배열에 호출했다

기존 예제는 아래 구조였다.

```js
Promise.all(
  masters
    .map((node) => node.keys())
    .then((keys) => {
      // keys: [['key1', 'key2'], ['key3', 'key4']]
    })
);
```

실행 순서를 따라가면 `masters.map(...)`의 결과에 `.then()`을 호출한다.
결과는 일반 배열이므로 `.then` 메서드가 없다. 바깥에 `Promise.all()`이 있다고 해서
안쪽의 배열에 Promise 메서드를 사용할 수 있는 것은 아니다.

수정은 명령들의 배열을 먼저 `Promise.all()`에 넘기고, 그 반환 Promise에 `.then()`을 붙인다.

```js
const masters = cluster.nodes("master");
Promise.all(masters.map((node) => node.keys("*"))).then((keys) => {
  // keys: [['key1', 'key2'], ['key3', 'key4']]
});
```

`Promise.all()`은 모든 명령이 성공하면 각 결과를 입력 배열 순서대로 모은다.
각 노드의 `KEYS`가 키 배열을 반환하므로 최종 `keys`는 배열 안에 배열이 있는 형태다.
노드별 실행 완료 순서에 맞춰 결과를 재정렬하는 것은 아니다.
하나라도 거절되면 전체 Promise도 거절되며, 다른 노드에 이미 보낸 명령이 취소되는 것은 아니다.

## 4. 두 번째 문제: KEYS 패턴이 빠져 있었다

Redis의 `KEYS` 명령에는 검색 패턴 인자가 필요하다. 기존 `node.keys()`에는 그 값이 없었다.
예제의 목적이 각 master의 키를 조회하는 것이므로 `node.keys("*")`로 고쳤다.
`"*"`는 모든 키 이름과 일치하는 패턴이다.

각 master에 별도로 조회하므로, 결과에는 해당 master가 가진 키들이 들어간다.
이 수정은 README의 기존 명령을 올바르게 호출하도록 만든 것이며,
클러스터 탐색 알고리즘이나 새로운 운영용 키 스캔 API를 추가한 것은 아니다.

## 5. 세 번째 문제: replica 예제에서 FLUSHDB를 사용했다

기존 예제는 replica들을 가져온 뒤 `FLUSHDB`를 호출했다.

```js
const slaves = cluster.nodes("slave");
Promise.all(slaves.map((node) => node.flushdb()));
```

일반적인 읽기 전용 replica에서는 이런 쓰기 명령이 거부된다.
여러 노드에 명령을 보낸다는 설명을 위해 데이터 삭제 명령을 사용할 이유도 없다.
따라서 연결 응답을 확인할 수 있는 `PING`으로 바꿨다.

```js
const slaves = cluster.nodes("slave");
Promise.all(slaves.map((node) => node.ping()));
```

각 replica에서 `PONG`을 받으면 fan-out 호출이 동작함을 확인할 수 있다.
이 예제는 명령을 보내는 방법만 보여 주므로 결과를 출력하는 `.then()`은 따로 추가하지 않았다.

## 6. 제출 당시 검증과 최종 결과

PR 본문에는 Redis 7.2.4의 로컬 클러스터에서 master 3개와 replica 3개를 사용해
README 예제를 확인했다고 적혀 있다. 모든 replica가 `PONG`을 반환했고,
master 키 조회 결과에는 미리 넣어 둔 키들이 포함됐다고 보고했다.
빌드와 린트도 통과했으며 린트에는 기존 경고가 있었다고 명시했다.
전체 테스트 스위트는 실행하지 않았고, 런타임 코드 변경도 없다.

자동 리뷰에는 추가 지적이 없었다. 작성자가 남긴 커밋은 한 개이며,
한국시간 2026-11-02 21:41:11 제출 커밋 생성 후 같은 날 22:32:10에 머지됐다.
원격 PR 상태와 머지 커밋을 확인해 처음부터 `typescript/merge`에 기록한다.
npm 패키지 배포 여부는 이번 확인 범위에 포함하지 않았다.

이 작업의 핵심은 예제의 괄호와 호출 순서도 실제 코드처럼 확인해야 한다는 점이다.
배열을 반환하는 `map()`, Promise를 반환하는 `Promise.all()`, Redis 명령의 필수 인자,
노드 역할별 허용 명령이 한 예제 안에서 모두 맞아야 독자가 그대로 실행할 수 있다.
