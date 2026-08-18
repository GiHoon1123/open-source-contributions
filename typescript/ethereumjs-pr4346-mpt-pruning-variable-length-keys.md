# [OSS 기여] ethereumjs/ethereumjs-monorepo - MPT 가변 길이 키 pruning 데이터 손실 버그 수정 (PR #4346) — 🟢 Merged

> 오픈소스 기여 기록 및 학습 노트. 이 문서는 PR #4346 하나만 다룬다. **최초 작성 2026-07-13, 2026-07-21에 메인테이너 리뷰 반영해서 갱신 — 버그 자체는 확인받았고, 머지 전 요청 2건에 대한 반영이 남은 상태(10번 섹션 참고). 2026-07-22, 3절의 오류 정정 — `559303c1…`은 op2의 루트 해시가 아니라 op5(직전 연산)의 루트 해시였다. 재현 스크립트를 다시 실행해 실제 해시값을 직접 확인하고 바로잡았다. 2026-07-24, 10번 섹션 갱신 — 머지 전 요청 2건(PR 본문 스코프 노트, 가변 길이 키 랜덤 삭제 테스트)이 실제로는 2026-07-22에 이미 커밋(`2159bf325`)돼 푸쉬까지 끝나 있었으나, 그걸 메인테이너에게 알리는 후속 코멘트가 빠져 있던 걸 발견해서 코멘트를 추가로 남겼다. 2026-07-28 14:25, 메인테이너 `gabrocheleau`가 머지함.**
>
> 4년 묵은 MPT 데이터 손실급 버그의 재현 → 축소 → 진단 → 근본 원인 → 수정 → 검증 전 과정을, **기초 개념부터 코드 한 줄까지** 전부 기록한 학습용 완전판. 특별한 점: ① 해시가 뭔지부터 시작하는 완전판 ② 근본 원인 발견과 수정은 직접 했고, 재현·축소·적대적 검토는 AI 도구의 도움을 받았다.

---

## 0. 관련 링크 모음

| 종류                           | 링크                                                          | 상태                                                         |
| ------------------------------ | ------------------------------------------------------------- | ------------------------------------------------------------ |
| **대상 이슈**                  | https://github.com/ethereumjs/ethereumjs-monorepo/issues/2347 | Open (2022-10 제기, 4년 방치)                                |
| **제출한 PR**                  | https://github.com/ethereumjs/ethereumjs-monorepo/pull/4346   | Open (2026-07-13 제출)                                       |
| 연관 이슈 (트라이 테스트 부족) | https://github.com/ethereumjs/ethereumjs-monorepo/issues/3645 | Open — holgerd77: "trie는 checkpoint 로직 테스트가 부족하다" |

- 패키지: `@ethereumjs/mpt` — 이더리움 실행 계층의 머클 패트리샤 트라이 구현 (**활성** 패키지)
- 작업 브랜치: `fix-2347-mpt-variable-length-key-pruning` (fork: GiHoon1123)
- 커밋: `d8475d19c` — 3개 파일, +90/−4
- 검증: mpt 스위트 1511개 + statemanager 822개 + 스트레스 수백 사이클 전부 통과

---

## 1. 기초 개념 — 밑바닥부터 하나씩

이 장은 순서대로 읽으면 다음 장의 모든 용어가 이해되도록 설계했다. 이미 아는 소절은 건너뛰어도 된다.

### 1-1. 해시 함수 — 데이터의 지문

**해시 함수**는 임의 길이의 데이터를 받아 고정 길이의 "지문"을 내놓는 함수다. 이더리움은 **keccak256**을 쓰고, 출력은 항상 **32바이트**다.

```
keccak256("hello")  = 0x1c8aff95...  (32바이트)
keccak256("hello!") = 0xe4e9a3f7...  (완전히 다른 32바이트)
```

세 가지 성질이 모든 것의 기반이다:

1. **결정성**: 같은 입력 → 항상 같은 출력. 어제 계산하든 다른 컴퓨터에서 계산하든 동일
2. **눈사태 효과**: 입력이 1비트만 달라져도 출력은 완전히 달라짐 → 지문만 비교해도 변조를 안다
3. **역산 불가**: 지문에서 원본을 복원할 수 없고, 같은 지문을 갖는 다른 입력을 만들어낼 수도 없다(실질적으로)

### 1-2. 머클 트리 — 지문의 지문의 지문

데이터가 수백만 개면 지문도 수백만 개다. 이걸 또 요약하려고 **지문들을 둘씩 묶어 다시 해시**하는 걸 반복하면 피라미드가 만들어진다:

```
        Root = H(H12 ‖ H34)         ← 꼭대기의 "머클 루트" 하나가
       /              \                아래 데이터 전체를 요약한다
   H12 = H(H1‖H2)   H34 = H(H3‖H4)
   /       \          /      \
  H1=H(D1) H2=H(D2) H3=H(D3) H4=H(D4)
  |        |        |        |
  D1       D2       D3       D4
```

이 구조가 주는 두 가지 힘:

**① 변조 감지**: D3이 1비트 바뀌면 → H3이 바뀌고 → H34가 바뀌고 → Root가 바뀐다. 두 사람이 **Root 32바이트만** 비교하면 수백만 개 데이터의 완전 일치를 확인한 것과 같다. 이더리움 합의의 본질이 이것이다 — "모든 노드가 같은 상태 루트를 계산했는가".

**② 머클 증명 (부분 증명)**: "D3이 이 트리에 들어 있다"를 증명하는 데 트리 전체가 필요 없다. D3과, 경로의 형제 해시들(H4, H12)만 주면 받는 쪽이 Root를 재계산해 대조할 수 있다. 데이터 100만 개짜리 트리도 증명은 log₂(100만) ≈ 20개 해시면 된다. 휴대폰 지갑(라이트 클라이언트)이 체인 전체를 내려받지 않고도 "내 잔고가 진짜 이만큼인가"를 검증하는 원리.

### 1-3. 트라이(Trie) — 키 자체가 길이 되는 트리

일반적인 이진 탐색 트리는 "값을 비교해서 왼쪽/오른쪽"으로 내려간다. **트라이는 다르다: 키의 글자 하나하나가 그대로 이동 경로다.**

영어 단어 사전을 트라이로 만들면:

```
        (root)
          |
          c
          |
          a
        /   \
       r     t          "car"는 c→a→r 경로의 끝에,
       |     |          "cat"은 c→a→t 경로의 끝에 값이 있다
     [car] [cat]        c→a 구간은 두 단어가 공유
```

- **같은 접두사를 가진 키들은 경로를 공유**한다 → 공간 절약
- 조회 비용은 트리에 담긴 키 개수가 아니라 **찾는 키의 길이**에 비례한다
- 그리고 이번 버그의 핵심 전제: **한 키가 다른 키의 접두사일 수 있다.** "ca"라는 단어가 사전에 있다면? c→a 지점, 즉 갈림길 **중간**에 값이 놓여야 한다. 이 상황을 처리하는 자리가 뒤에 나올 "branch의 value 슬롯"이다.

### 1-4. 니블(nibble) — 반 바이트

**니블 = 4비트 = 16진수 문자 1개.** 1바이트는 니블 2개다:

```
바이트 0x5A  =  2진수 0101 1010  =  니블 [0x5, 0xA]  =  [5, 10]
바이트 0x04  →  니블 [0, 4]
0x04c7 (2바이트)  →  니블 [0, 4, 12, 7]   (c=12)
```

이더리움 MPT는 **키를 니블 단위로 쪼개서 경로로 쓴다.** 왜 하필 니블인가:

| 경로 단위 | 갈림길의 갈래 수 | 문제                                                           |
| --------- | ---------------- | -------------------------------------------------------------- |
| 비트      | 2                | 트리가 너무 깊어짐 (32바이트 키 = 깊이 256) → 조회·증명이 느림 |
| **니블**  | **16**           | **깊이 64, 노드 크기 적당 — 채택**                             |
| 바이트    | 256              | 갈림길 노드가 자식 슬롯 256개 — 노드가 비대해져 증명 크기 폭발 |

이번 버그의 재료를 니블로 미리 번역해두자:

```
0x51        → [5, 1]
0x57179cc0  → [5, 7, 1, 7, 9, 12, 12, 0]     ← 0x51과 니블 [5]를 공유
0x04        → [0, 4]
0x04c7      → [0, 4, 12, 7]                   ← 0x04의 "접두사 확장"!
```

`[0,4]`는 `[0,4,12,7]`의 접두사다. 사전의 "ca"⊂"cat" 상황 — **갈림길 중간에 값이 놓이는** 상황이 정확히 만들어진다.

### 1-5. MPT = 머클 트리 + 패트리샤 트라이

이더리움의 **머클 패트리샤 트라이**는 두 아이디어의 결합이다:

> 트라이 구조로 키→값을 저장하되, **부모가 자식을 가리킬 때 메모리 포인터가 아니라 "자식 노드의 keccak256 해시"로 가리킨다.**

그러면 루트 노드의 해시 하나가 (머클 트리처럼) 트라이 전체 내용을 요약하게 된다. 값 하나가 바뀌면 그 리프의 해시가 바뀌고 → 부모가 담고 있는 자식 해시가 달라지니 부모의 해시도 바뀌고 → 루트까지 연쇄된다.

"패트리샤"는 **경로 압축**을 뜻한다. 갈림길 없이 한 줄로만 이어지는 긴 복도를 노드 하나로 접는다. 그 결과 노드가 3종이다:

**① Branch (갈림길)** — 슬롯 17개짜리 배열:

```
[슬롯0, 슬롯1, ..., 슬롯15, value]
  ↑ 다음 니블이 0인 키들의 자식     ↑ "정확히 여기서 끝나는 키"의 값!
```

17번째 **value 슬롯**이 §1-3에서 예고한 "갈림길 중간에 놓이는 값"의 자리다. `0x04`와 `0x04c7`이 공존하려면: 경로 `[0,4]` 지점의 branch가 value 슬롯에 `0x04`의 값을 들고, 슬롯 12(c)로는 `0x04c7`쪽 자식을 가진다. **이 슬롯이 이번 버그가 생기는 지점이다.**

**② Extension (복도)** — `(공유 니블 경로, 자식 포인터 1개)`:

```
Ext([5]) → X   :  "니블 5를 소비하고 X로 가라"
```

`0x51`과 `0x57179cc0`만 있는 트라이에서 첫 니블 5는 무조건 지나야 하는 복도 → Ext([5])로 압축.

**③ Leaf (종착지)** — `(남은 니블 경로, 값)`. 더 이상 갈림길이 없으면 남은 경로를 통째로 갖고 끝난다.

### 1-6. RLP와 노드 저장 방식 — DB에는 뭐가 들어가나

노드를 해시하려면 먼저 바이트로 **직렬화**해야 한다. 이더리움의 직렬화 포맷이 **RLP**(Recursive Length Prefix)다 — 중첩 배열과 바이트열을 규칙적으로 한 줄의 바이트로 펴는 방법이라고만 알면 충분하다.

노드의 저장 규칙:

```
serialized = RLP(노드)
DB에 저장:  키 = keccak256(serialized), 값 = serialized
부모 노드는 자식을 keccak256(serialized) 32바이트로 참조
```

여기서 두 가지 결정적인 성질이 나온다.

**성질 A — 내용 주소화(content-addressing)**: DB 키가 내용의 해시이므로 **내용이 같으면 DB 키도 같다.** 트라이에 뭔가 넣었다 빼서 **과거와 똑같은 모양**이 되면, "새로 만든" 노드의 해시가 과거 노드의 해시와 정확히 일치한다. 이것은 버그가 아니라 정상 성질인데 — 이번 버그의 가장 극적인 증상("자기가 방금 저장한 노드를 자기가 지움")의 재료가 된다.

**성질 B — 임베디드(인라인) 노드**: `serialized`가 **32바이트 미만**이면? 해시(32바이트)로 가리키는 것보다 원본을 통째로 부모 안에 넣는 게 더 싸다. 그래서 작은 노드는 DB에 따로 저장하지 않고 **부모의 슬롯에 원본 배열 그대로 인라인**된다. 코드에서 `isRawMPTNode()`가 "이 참조가 해시(Uint8Array)냐 인라인 원본(Uint8Array[])이냐"를 구별한다. → 검증기 크래시 버그(§6-D)가 바로 이 두 형태를 구별 안 해서 났다.

### 1-7. 이더리움은 MPT를 어디에 쓰나

블록 헤더에는 트라이 루트가 여러 개 박혀 있다:

| 트라이           | 키 → 값                                                       | 역할                                                                              |
| ---------------- | ------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| **State trie**   | keccak256(계정주소) → RLP(잔고, nonce, codeHash, storageRoot) | 전 계정의 상태. 루트가 헤더의 `stateRoot`                                         |
| **Storage trie** | keccak256(슬롯번호) → 슬롯 값                                 | 컨트랙트마다 하나. 그 루트가 계정 레코드 안의 `storageRoot` (트라이 안에 트라이!) |
| Transaction trie | RLP(블록 내 인덱스) → tx                                      | 블록의 tx 목록 요약                                                               |
| Receipt trie     | RLP(인덱스) → 실행 영수증                                     | 실행 결과 요약                                                                    |

"이더리움의 상태"란 결국 이 트라이들이다. 합의 = "모두가 같은 stateRoot에 도달"이고, 머클 증명 덕에 라이트 클라이언트가 잔고·스토리지를 검증할 수 있다.

### 1-8. 왜 이더리움 본체에서는 이 버그가 4년간 안 터졌나 — secure trie

위 표를 다시 보면 state/storage trie의 키가 주소 그 자체가 아니라 **keccak256(주소)** 다. 이렇게 키를 한 번 해시해서 쓰는 걸 **secure trie**라 부른다 (원래 목적: 공격자가 특정 접두사의 키를 대량 생산해 트라이를 한쪽으로 길게 늘어뜨리는 DoS 방지 — 해시를 거치면 키 분포를 조작할 수 없다).

부수 효과가 이번 이슈의 운명을 갈랐다: **keccak256 출력은 전부 32바이트 고정**이다. 길이가 같은 두 키는 서로의 접두사가 될 수 없다(접두사이면서 같은 길이면 동일한 키다). 접두사 키가 없으면:

- "branch에서 끝나는 키"가 존재하지 않고
- `_deleteNode`의 `value(null)` 경로가 **실행될 일 자체가 없다**

즉 이더리움 노드를 돌리는 경로에서는 4년간 단 한 번도 버그 코드가 실행되지 않았다. 터지는 곳은 **키를 해시하지 않고 raw로 쓰는 사용처** — 이슈 리포터(kiwistand라는 앱의 set reconciliation 저장소), MPT를 범용 "검증 가능한 KV 스토어"로 쓰는 모든 애플리케이션이다. 라이브러리가 공식 지원하는 사용법인데 주력 경로가 아니라서 방치된 것.

> 교훈 미리보기: "주력 사용처에서 안 터진다" ≠ "코드가 옳다". 지원하는 입력 공간 전체가 테스트 대상이다.

### 1-9. 삭제는 언제 일어나고, pruning·useRootPersistence는 무엇인가

**이더리움 레벨에서 트라이 삭제가 실제로 일어나는 순간:**

- 컨트랙트가 스토리지 슬롯에 **0을 쓸 때**: `SSTORE slot, 0`은 "그 슬롯을 storage trie에서 삭제"로 구현된다 (상태를 줄여줘서 가스 환급까지 주며 장려하는 연산)
- 계정 소멸: 과거 `SELFDESTRUCT`, 빈 계정 정리 — state trie에서 계정 삭제
- 앱 레벨: raw trie를 KV로 쓰는 코드의 `trie.del(key)` 직접 호출 ← **이번 버그의 현장**

**pruning (`useNodePruning: true`)**: MPT는 사실상 불변 구조다 — 값 하나를 바꾸면 그 경로의 노드가 전부 **새 해시의 새 노드로 다시 만들어진다.** 그럼 옛 노드들은? DB에 그대로 남는다. 히스토리 조회가 필요하면 자산이지만, 필요 없으면 무한히 쌓이는 쓰레기다. pruning 옵션은 "쓰기·삭제로 참조가 끊긴 옛 노드를 그 자리에서 DB에서 지워라"이다. 구현은: 연산 전에 "이번에 갈아치워질 노드들"의 해시 목록(`_createPruneDeleteOps`)을 만들어뒀다가, 새 노드 저장이 끝난 뒤 배치로 삭제한다.

**`verifyPrunedIntegrity()` (테스트용 검증기)**: pruning이 잘 됐는지 확인하는 함수. DB의 **모든** 항목에 대해 "루트에서 트라이를 걸어 내려가면 너를 참조하는 노드가 하나라도 있는가"를 검사한다. 아무도 참조하지 않는 항목(고아)이 있으면 false = pruning 실패.

**`useRootPersistence`**: 트라이 루트 해시를 DB의 고정 키(`ROOT_DB_KEY`)에 저장해서 프로세스 재시작 후에도 루트를 알 수 있게 하는 옵션. 저장 경로에 분기를 하나 더 만들기 때문에 테스트는 이 옵션 **양쪽 모두**를 커버해야 한다 (이슈 원문도 이 옵션이 켜진 상태를 지목했었다).

### 1-10. 이번 이슈가 성립하는 최소 조건 정리

여기까지의 개념을 조합하면 이 버그가 성립하는 조건이 전부 갖춰진다:

```
raw 키 (해시 안 함)              → 가변 길이 가능
+ 가변 길이                      → 접두사 키 쌍 가능 (0x04 ⊂ 0x04c7)
+ 접두사 키 쌍                   → "branch에서 끝나는 키" 발생 (value 슬롯 사용)
+ 그 키를 삭제                   → _deleteNode가 value(null) 호출
+ useNodePruning                → 옛 노드 삭제 장부까지 얽힘
= 4년간 아무도 밟지 않은 지뢰밭
```

---

## 2. 전체 타임라인

```
2022-10-10  이슈 제기: "가변 길이 키에서 pruning이 고장난다" + 랜덤 재현 스크립트 첨부
2026-07-13  [선정] 활성 패키지 이슈 재스캔 — 선점 PR 없음 + 재현 스크립트 내장이 결정 이유
2026-07-13  [재현] 현재 master에서 0.5초 만에 실패 — 버그 생존 확인
2026-07-13  [1차 축소 시도 → 실패] 2~3키 전수조합 탐색으로는 안 터짐 (교훈: §4-2)
2026-07-13  [2차 축소 성공] 델타 디버깅 하네스 → 112-op 랜덤 실패를 6-op 결정적 재현으로
2026-07-13  [진단 ①] DB vs 도달가능 집합 비교 → "고아 1 + 실종 1(새 루트!)" — 데이터 손실 확정
2026-07-13  [진단 ②] DB 연산 스파이 → "같은 배치에서 put 559303c1 + del 559303c1" 포착
2026-07-13  [근본 원인] 소스에서 진짜 원인 발견: BranchMPTNode.value(null)이 조용히 무시됨
2026-07-13  [1차 수정 + 검토] 6-op 해결 — 그러나 스트레스에서 신규 실패 2종
            → 재축소: [B] 2-op, [A] 7-op / 수정 전 master에서 재현해서 "기존 버그"로 판정
2026-07-13  [2차 수정 + 검토] 3개 버그 수정 완료 — 빌드 타입 에러 1건 → 타입 가드로 마감
2026-07-13  커밋 d8475d19c → push → PR #4346 (AI 흔적 0 검증 포함)
```

---

## 3. 6-op 재현의 해부 — 트라이가 연산마다 어떻게 변하는가

축소된 6-op 시퀀스를 트라이 그림으로 따라가면 버그의 모든 재료가 보인다. (그림은 진단 출력의 DB 항목 수와 일치하도록 재구성한 것)

```
put(0x51,       '16d0')     니블 [5,1]
put(0x57179cc0, '3dbc')     니블 [5,7,1,7,9,c,c,0]
put(0x04,       '721')      니블 [0,4]
put(0x04c7,     '967')      니블 [0,4,c,7]
del(0x04c7)
del(0x04)                   ← 여기서 붕괴
```

**op1 — put(0x51)**: 키 하나뿐. 루트 = Leaf([5,1], '16d0'). DB 1개. ✓(진단: DB=1)

**op2 — put(0x57179cc0)**: 두 키가 니블 [5]를 공유 → 복도 + 갈림길 생성:

```
루트: Ext([5]) ──→ Branch
                    ├─ 슬롯1 → Leaf([], '16d0')        (0x51의 나머지)
                    └─ 슬롯7 → Leaf([1,7,9,c,c,0], '3dbc')
```

Leaf들은 작아서(32바이트 미만) branch 안에 임베디드. DB에는 Ext(루트)와 Branch, 2개가 해시로 저장. ✓(진단: DB=2) — 이 시점의 실제 루트 해시는 `a333cee2…`다(재현 스크립트로 직접 확인한 값).

**op3 — put(0x04)**: 니블 [0,...]은 기존 복도 [5]와 첫 니블부터 다름 → 루트가 갈림길로 교체:

```
루트: Branch
       ├─ 슬롯0 → Leaf([4], '721')        (임베디드)
       └─ 슬롯5 → (해시) → Branch(1,7)     (op2의 그 갈림길이 그대로 재사용됨)
```

Ext([5])는 니블 5가 루트 branch의 슬롯 번호로 흡수되며 소멸. pruning이 옛 루트(Ext)를 DB에서 삭제. DB: 새 루트 Branch + Branch(1,7) = 2개. ✓

**op4 — put(0x04c7)**: [0,4,c,7]은 기존 [0,4]와 [4]까지 공유 후 연장 → **접두사 상황 발생**:

```
루트: Branch
       ├─ 슬롯0 → Ext([4]) → Branch ★     ← "branch-with-value"!
       │                      ├─ value = '721'      (0x04이 여기서 끝남)
       │                      └─ 슬롯c → Leaf([7], '967')   (0x04c7)
       └─ 슬롯5 → (해시) → Branch(1,7)
```

★이 §1-5에서 설명한 value 슬롯의 실전 등장이다. 새 구조는 전부 작아서 임베디드 → DB는 여전히 2개. ✓

**op5 — del(0x04c7)**: Leaf([7])를 branch★에서 제거. branch★는 **자식 0개 + value '721'**만 남는다. 규범적으로는 Leaf로 승격돼야 하지만(버그 ②의 영역), 옛 코드는 그 케이스가 없어 이 기형 branch를 그대로 저장. 그래도 값 읽기는 되니 검증은 아직 통과. ✓(진단: verify=true) — **이 시점의 루트 해시가 `559303c1…`다.** 이 값이 바로 다음 연산(op6)에서 충돌을 일으키는 핵심 재료이므로 정확히 기억해두자.

**op6 — del(0x04)**: 운명의 연산. `del()`의 진행 순서를 정확히 나눠서 보면:

```ts
// 1단계 — del() 진입 직후, 아직 아무것도 안 지운 "현재" 상태를 스냅샷
ops = this._createPruneDeleteOps(stack); // "이 스택(루트=559303c1 포함)은 곧 교체될 테니 지워도 된다"는 목록

// 2단계 — 실제 삭제 시도
if (lastNode instanceof BranchMPTNode) {
  lastNode.value(null); // "값을 지워라" → 버그 ①: 조용히 무시됨!!
}
```

1단계 시점의 스택은 **op5가 끝난 시점의, 아직 멀쩡하고 실제로 쓰이고 있는 상태**다. 이 스택을 "지워도 되는 목록"에 넣은 이유는 "곧 0x04가 진짜로 빠진 새 구조로 교체될 거니까"라는 전제 때문이다.

그런데 2단계가 버그 때문에 실패해서, branch★는 **내용이 하나도 안 바뀐다**. 이후 collapse 로직은 이 안 바뀐(유령 값이 그대로인) 노드를 "새로 갱신된 노드"인 것처럼 다시 저장하는데, **내용이 진짜로 하나도 안 바뀌었으니 직렬화 결과도, 해시도 op5 시점 그대로(`559303c1…`)**다. 즉 "새로 계산했다"는 결과물이 사실 1단계에서 스냅샷 떴던 바로 그 상태와 완전히 동일하다.

배치는 순서대로 적용된다:

```
DB.batch put 559303c1…   ← 새로 계산했다며 저장 (실제로는 op5 시점과 완전히 동일한 내용)
DB.batch del 559303c1…   ← 1단계에서 "곧 교체될 옛날 것"이라며 미리 지워라고 찜해둔 바로 그 해시
DB.batch del fc65e5d7…
DB.batch del 9733cfc3…
```

**저장과 삭제가 서로 다른 두 데이터를 겨냥한 게 아니다.** DB는 해시 하나당 칸이 하나뿐인 저장소라서, "이 칸에 다시 넣어라"(put)와 "이 칸을 비워라"(del)가 같은 배치 안에서 순서대로 실행되면 최종적으로 그 칸은 비워진다. 이번 경우 그 칸(`559303c1…`)에는 **다른 무언가가 아니라, 이 삭제 작업이 시작되기 직전까지 실제로 살아서 쓰이고 있던 루트**가 들어있었다 — "곧 교체되니 지워도 된다"고 판단해서 지운 건데, 교체가 실제로는 일어나지 않아서(내용이 안 바뀌었으니) 결국 **유일하게 존재하던 그 데이터**가 지워진 것이다.

결과(진단 출력 원문):

```
ORPHAN in DB:    2dc99227… (옛 branch — 지웠어야 하는데 남음)
MISSING from DB: 559303c1… (op5 시점부터 쓰이던 루트 — 있어야 하는데 없음!)
```

**루트가 DB에 없다** = 살아남은 키 0x51, 0x57179cc0을 읽는 순간 "Missing node" 크래시. pruning의 미관 문제가 아니라 **데이터 손실**이다.

> **참고**: fix가 적용된 버전으로 이 6-op를 다시 돌리면, op6이 실제로 값을 지우는 데 성공해서 트라이가 진짜로 op2와 같은 모양까지 완전히 되돌아가고, 그 결과 루트 해시도 op2의 실제 루트 해시(`a333cee2…`)와 정확히 일치한다 — §1-6 성질 A(내용 주소화)가 말하는 "같은 모양 = 같은 해시"가 실제로 이 지점에서 성립하는 것을 재현 스크립트로 직접 확인했다. 버그가 있는 버전은 이 단계까지 가지도 못하고, 그 한 단계 전(op5의 상태, `559303c1…`)에서 자기 자신과 충돌해 멈춘다.

---

## 4. 디버깅 상세 — 도구를 만들며 좁혀간 과정

### 4-1. 0단계: 재현 확인

이슈의 처방("기존 prune 테스트를 가변 길이 키로 바꿔라")대로 재현 스펙을 작성했다. 핵심 한 줄:

```ts
// 기존 테스트: 고정 32바이트 키 → 접두사 불가능 → 통과
keys.push(randomBytes(32));
// 이슈의 처방: 1~31바이트 가변 → 접두사 쌍 발생 → ?
keys.push(randomBytes(Math.floor(Math.random() * 31) + 1));
```

이 워크로드(랜덤 put/del 수백 회 + 단계별 `verifyPrunedIntegrity()`)를 돌리자 **0.5초 만에** 실패:

```
AssertionError: pruned integrity after delete-all: expected false to be true
```

`useNodePruning`만 켜도 실패 → 이슈 제목의 `useRootPersistence`는 필요조건이 아니었다 (보고보다 범위가 넓은 버그).

### 4-2. 1차 축소 시도 — 실패한 접근도 기록한다

첫 시도는 "짧은 키 풀에서 2~3개 조합 × 삽입/삭제 순열 전수 탐색"이었다. 결과: **한 건도 실패하지 않음.**

왜 실패했나 — 나중에 알고 보니 랜덤 재현에는 있고 전수 탐색에는 없던 요소가 있었다:

- **같은 키의 덮어쓰기(update)** 와 **삭제 후 재삽입** — 노드 교체 이력이 쌓여야 pruning 장부가 얽힌다
- **작은 값** — 값이 짧아야 노드가 32바이트 미만이 되어 임베디드 경로(§1-6 성질 B)를 탄다

> 교훈: 손으로 고른 "그럴듯한 최소 케이스"는 상상력의 한계에 갇힌다. 실패를 찾는 건 무작위에게, 줄이는 건 기계에게 맡기는 게 낫다. → 델타 디버깅으로 전환.

### 4-3. 2차 축소 — 델타 디버깅 하네스 (코드 포함)

**델타 디버깅**의 아이디어는 단순하다: "실패하는 입력에서 조각을 하나 빼보고, 여전히 실패하면 그 조각은 무죄이므로 영구히 제거한다"의 반복.

하네스는 세 부품으로 구성했다:

**부품 1 — 결정적 재생기 `replayFails(ops)`**: 연산 목록을 처음부터 재생하고 실패 여부만 돌려준다. 두 가지 설계 포인트:

```ts
async function replayFails(ops: Op[]): Promise<boolean> {
  const trie = new MerklePatriciaTrie({ useNodePruning: true });
  try {
    for (const op of ops) {
      if (op.t === "put") await trie.put(hexToBytes(op.k), utf8ToBytes(op.v));
      else await trie.del(hexToBytes(op.k));
    }
    return !(await trie.verifyPrunedIntegrity()); // ① 무결성 붕괴 = 실패
  } catch {
    return true; // ② 예외(스택 언더플로 등)도 실패로 집계
  }
}
```

①과 ②를 모두 실패로 치는 이유: 이슈 원문이 "검증기가 false를 주거나, **혹은** put에서 stack underflow가 난다"고 두 증상을 언급했기 때문. 어느 쪽으로 죽든 같은 병의 다른 얼굴일 수 있으니 축소 대상에 포함한다.

**부품 2 — 실패 채굴기 `findFailingSequence()`**: 랜덤 워크로드를 돌리되 **모든 연산을 기록**하면서, 매 연산 후 무결성을 검사해 첫 실패 순간의 전체 기록을 반환한다. 키는 1~4바이트로 짧게(접두사 충돌 확률 극대화), put 60% / del 40%.

**부품 3 — 최소화기 `minimize(ops)`**:

```ts
async function minimize(ops: Op[]): Promise<Op[]> {
  let current = ops;
  let changed = true;
  while (changed) {
    // 고정점(더 못 줄임)까지 반복
    changed = false;
    for (let i = current.length - 1; i >= 0; i--) {
      // 뒤에서부터 하나씩
      const candidate = [...current.slice(0, i), ...current.slice(i + 1)];
      if (candidate.length > 0 && (await replayFails(candidate))) {
        current = candidate; // 빼도 실패 → 그 연산은 무죄, 영구 제거
        changed = true;
      }
    }
  }
  return current;
}
```

한 바퀴에서 하나라도 제거되면 처음부터 다시 도는 이유: 연산 간 상호작용 때문에, A를 제거한 뒤에야 B도 제거 가능해지는 경우가 있다.

**곁가지 — vitest가 console.log를 삼킨 사건**: 결과를 `console.log`로 찍었는데 이 저장소의 vitest 설정이 출력을 표시하지 않았다. 해결: `writeFileSync('/tmp/…')`로 파일에 쓰고 cat. (도구가 말을 안 들으면 도구와 싸우지 말고 우회하라.)

**결과**:

```
초기 실패 시퀀스: 112 ops
=== 최소 재현 (6 ops) ===
put(0x51, '16d0') … del(0x04)
재검증 1/2/3: 실패 재현 O / O / O     ← 결정성 3회 확인
```

112 → 6. 그리고 6-op는 그대로 회귀 테스트가 됐다.

### 4-4. 진단 ① — 이중 장부: DB vs 도달가능 집합

`verifyPrunedIntegrity()`가 false라는 건 "뭔가 나쁨"만 알려준다. 게다가 이 검증기 **자체에 버그가 있음**이 이미 드러난 상태였다(§4-6). 오염됐을지 모르는 판정 도구에 의존하는 대신, 진단 스크립트가 자체적으로 두 집합을 계산해 대조했다:

```
(A) DB에 실제로 있는 노드 해시들        ← trie['_db'] 내부 Map을 직접 열람
(B) 있어야 하는 노드 해시들             ← 루트부터 직접 걸어 내려가며 수집
                                        (임베디드 자식은 재귀만 하고 집합엔 미포함 —
                                         DB에 없는 게 정상이므로)
A − B = 고아(pruning 실패)    B − A = 실종(데이터 손실!)
```

수집 워커의 뼈대:

```ts
const visit = async (ref: Uint8Array | Uint8Array[]) => {
  let node;
  if (isRawMPTNode(ref)) {
    // 임베디드: DB 조회 없이 원본을 바로 디코드
    node = decodeRawMPTNode(ref);
  } else {
    // 해시 참조: DB에서 찾아와 디코드
    const raw = db.get(bytesToUnprefixedHex(ref));
    if (raw === undefined) {
      found.set(hex, "MISSING FROM DB!");
      return;
    } // ← 실종 검출
    node = decodeMPTNode(raw);
    found.set(hex, describe(node));
  }
  if (node instanceof BranchMPTNode)
    for (const child of node._branches) await visit(child);
  else if (node instanceof ExtensionMPTNode) await visit(node.value());
};
```

각 연산 후 이 대조를 돌린 결과가 §3 끝의 "고아 1 + 실종 1(새 루트)"이다. **실종의 발견이 사건의 등급을 바꿨다** — 이슈 제목("pruning이 안 됨" = 고아가 남음 = 디스크 낭비)보다 훨씬 심각한, 트라이 판독 불가.

### 4-5. 진단 ② — DB 연산 스파이 (스모킹 건)

"루트가 왜 없지?"의 다음 질문은 "**루트에게 무슨 일이 있었나**"다. DB 객체의 메서드를 몽키패치해서 마지막 연산 동안의 모든 트래픽을 녹취했다:

```ts
const origBatch = db.batch.bind(db);
db.batch = async (ops) => {
  for (const o of ops)
    spy.push(`DB.batch ${o.type} ${hex(o.key).slice(0, 16)}…`);
  return origBatch(ops);
};
```

녹취록:

```
DB.batch put 559303c1…    ← saveStack이 새 루트 저장
DB.batch del 559303c1…    ← pruning 삭제 목록이 같은 해시를 삭제!!
DB.batch del fc65e5d7…
DB.batch del 9733cfc3…
```

**자기가 방금 저장한 것을 자기가 지운다.** §1-6 성질 A(내용 주소화) 때문에 "새 루트 = op2의 옛 루트와 같은 내용 = 같은 해시"가 성립하고, 오염된 삭제 장부에 그 해시가 들어 있던 것.

### 4-6. 덤으로 잡힌 검증기 크래시

축소 실행 중 vitest가 "Unhandled Rejection 20건"을 뿜었다:

```
Error: expected Uint8Array, got type=object
  at bytesToUnprefixedHex …
  at WalkController.onNode (mpt.ts:882)      ← verifyPrunedIntegrity 내부!
```

882행: `bytesToUnprefixedHex(node.value())` — Extension의 자식 참조를 무조건 해시(Uint8Array)로 간주했는데, §1-6 성질 B대로 **임베디드 원본(배열)일 수 있다.** 게다가 이 에러는 walk 콜백(fire-and-forget 비동기) 안에서 발생해 검증기의 promise로 전파되지 않고 **unhandled rejection으로 새어나갔다** — 검증기 자체의 견고성 버그.

### 4-7. 근본 원인 — 소스에서 진짜 원인을 찾다

여기까지의 가설은 "저장 목록과 삭제 목록의 교집합을 검사하지 않는다"(배치 레벨)였다. 그런데 소스를 직접 파면서 더 깊은 지점을 찾았다:

```ts
// packages/mpt/src/node/branch.ts — getter/setter 겸용 메서드
value(v?: Uint8Array | null): Uint8Array | null {
  if (v !== null && v !== undefined) {   // ← v가 null이면 대입을 건너뜀!
    this._value = v
  }
  return this._value
}
```

이 메서드의 계약: 인자 없이 부르면 읽기(getter), 인자를 주면 쓰기(setter). "인자를 안 줬음"의 표식(sentinel)은 `undefined`다. 그런데 가드가 `null`까지 sentinel 취급해서, **`value(null)` = "값을 삭제하라"는 유효한 쓰기 요청이 무시**된다.

그리고 그 호출은 모노레포 전체에서 정확히 한 곳 —

```ts
// mpt.ts _deleteNode: 삭제할 키가 branch에서 끝나는 경우 (= 접두사 키!)
if (lastNode instanceof BranchMPTNode) {
  lastNode.value(null); // 4년간 no-op이었던 줄
}
```

**배치의 put-후-del은 증상이었고, 진짜 원인은 따로 있었다.** 유령 값이 남은 branch를 기준으로 collapse와 pruning 장부가 진행되니 하류가 전부 오염된 것. 증상 레벨(배치 교집합 필터)에서 고쳤다면 유령 값은 살아남아 다른 형태로 계속 문제를 일으켰을 것이다.

### 4-8. 1차 수정 후 — "버그는 무리 지어 산다"

setter를 고치자 6-op 재현이 통과했다. 하지만 검토의 스트레스 재실행에서 **새로운 실패 2종**:

```
[useRootPersistence=false] integrity broken at cycle 3, op 132
[useRootPersistence=true]  empty root, cycle 0    ← 전부 지웠는데 안 비워짐!
```

같은 델타 디버깅 하네스에 실패 판정만 바꿔 재투입 (모드 B는 "모든 키 삭제 후 루트 ≠ 빈 루트"를 실패로 정의):

```
[A] 최소 재현 (7 ops): put(0x3bd5c8) put(0xeef88c) put(0x3b)
                        del(0x3bd5c8) put(0xe52b) del(0x3b) del(0x3b)
[B] 최소 재현 (2 ops!): put(0x7170, …) put(0x71, …)
                        → 이후 둘을 지워도 루트가 안 비워짐
```

**중요한 검증 — 이게 이 수정이 만든 회귀인가?** `git stash`로 수정을 잠시 걷어내고 수정 전 master에서 [B]를 재생 → **역시 실패.** 즉 ①과 독립적으로 원래부터 있던 기존 버그다. 이 판정은 PR 방어 논리("both reproduce on master independently of the setter fix")로 직행했다.

왜 이제야 보이나: 랜덤 테스트는 확률적으로 가장 흔한 버그 ①에서 항상 먼저 죽는다. ①이 사라지자 그 뒤에 숨어 있던 ②③이 관측 가능해진 것. **"물 새는 집에서 제일 큰 구멍을 막으면, 그제야 작은 구멍들이 보인다."**

---

## 5. 세 버그와 수정 — 코드 한 줄씩

### 버그 ① 수정 — `branch.ts`: sentinel은 undefined 하나로

```diff
 value(v?: Uint8Array | null): Uint8Array | null {
-  if (v !== null && v !== undefined) {
+  if (v !== undefined) {
     this._value = v
   }
   return this._value
 }
```

**동작 표로 보는 변화:**

| 호출           | 수정 전         | 수정 후        |
| -------------- | --------------- | -------------- |
| `value()`      | 읽기            | 읽기 (동일)    |
| `value(bytes)` | 쓰기            | 쓰기 (동일)    |
| `value(null)`  | **무시 (버그)** | **값 삭제 ✅** |

한 줄이지만 노드 코어 클래스의 의미 변경이므로 파급 검증이 필수였다: `grep -rn "\.value(null)" packages/*/src` → **모노레포 전체에서 호출처가 `mpt.ts:687` 단 한 곳.** 의미 변경의 영향 범위가 정확히 버그 지점에만 국한된다. (이 grep 한 줄이 "이 수정은 안전하다"의 근거 전부다 — 리뷰어에게도 같은 논리가 통한다.)

### 버그 ③ 수정 — `mpt.ts` `del()`: "도달했다" ≠ "존재한다"

**배경 지식**: `findPath(key)`는 키 니블을 따라 내려가 도달한 노드를 돌려준다. 그런데 접두사 키를 지운 자리에는 **값 없이 갈림길 역할만 하는 branch**가 남아 있을 수 있다. 이 branch에 "도달"했다고 해서 그 키가 "존재"하는 건 아니다 — value 슬롯이 비어 있으니까.

수정 전 코드는 이 구별을 안 했다:

```ts
const { node, stack } = await this.findPath(appliedKey);
if (node) {
  // 도달만 하면 삭제 실행!
  await this._deleteNode(appliedKey, stack);
}
```

존재하지 않는 키에 대해 `_deleteNode`가 실행되며 pruning 장부(`_createPruneDeleteOps`)까지 오염된다. [A] 재현의 `del(0x3b)` 두 번째 호출이 정확히 이 경로였다.

수정:

```ts
const { node, stack } = await this.findPath(appliedKey);
const nodeHasValue =
  node !== null && (!(node instanceof BranchMPTNode) || node.value() !== null);
//     ↑ 도달 여부         ↑ Leaf/Ext면 도달=존재       ↑ Branch면 value 슬롯까지 확인

let ops: BatchDBOp[] = [];
if (this._opts.useNodePruning && nodeHasValue) {
  ops = this._createPruneDeleteOps(stack); // 진짜 삭제할 때만 장부 작성
}
if (nodeHasValue) {
  await this._deleteNode(appliedKey, stack); // 진짜 삭제할 때만 구조 변경
}
```

조건식 풀이: Leaf나 Extension에 도달했다면 그 자체가 값의 존재다(Leaf는 값을 직접 가짐). **Branch에 도달한 경우에만** value 슬롯의 null 여부를 추가로 본다. 값이 없으면 두 갈래(장부 작성, 구조 변경) 모두 건너뛰어 del이 안전한 no-op이 된다.

### 버그 ② 수정 — `mpt.ts` `_deleteNode`: 자식 0개 collapse 신설

**배경 지식 — collapse란**: 삭제 후 트라이는 규범형(canonical form)으로 축약돼야 한다. 규범형의 불문율: branch는 존재 이유(2개 이상의 항목)가 있어야 하고, 그렇지 않으면 더 단순한 노드로 접혀야 한다. 기존 코드는 "자식 1개 남음 → 그 자식을 위로 합침"(`length === 1`)만 있었고 **"자식 0개"가 없었다.**

자식 0개가 되는 경로: 접두사 쌍에서 긴 키를 먼저 지우면 — [B] 재현 `put(0x7170), put(0x71), del(0x7170)` — branch가 자식(0x7170의 leaf)을 잃고 **value('0x71'의 값)만 남는다.** 이때:

```ts
if (branchNodes.length === 0) {
  const branchValue = lastNode.value();

  if (branchValue !== null) {
    // ── 케이스 1: 값만 남은 branch → Leaf로 승격 ──────────────────
    // "갈림길인데 갈 곳이 하나도 없고 값만 있다" = 사실상 종착지 = Leaf가 규범형
    const leafNode = new LeafMPTNode([], branchValue);
    //                               ↑ branch는 니블을 소비하지 않으므로 leaf 키는 빈 배열에서 시작
    if (parentNode instanceof ExtensionMPTNode) {
      // 부모가 복도(Ext)면: Ext([k])→Leaf([]) 는 Leaf([k]) 하나와 같다.
      // 복도의 경로를 leaf 키로 흡수하고 Ext는 소멸시킨다 (경로 압축 유지)
      leafNode.key(parentNode.key());
      stack.push(leafNode); // Ext 대신 leaf가 그 자리에
    } else {
      if (parentNode instanceof BranchMPTNode) stack.push(parentNode);
      stack.push(leafNode); // 부모 branch 밑에 leaf로
    }
    await this.saveStack(pathNibbles, stack, opStack); // 새 구조 저장 + 옛 노드 장부 처리
    return;
  }

  // ── 케이스 2: 값도 자식도 없는 branch → 존재 이유 없음, 제거 ────
  if (parentNode === undefined || parentNode instanceof ExtensionMPTNode) {
    this.root(this.EMPTY_TRIE_ROOT); // 루트였다면 트라이 전체가 빈 것
    return;
  }
  if (!(parentNode instanceof BranchMPTNode)) {
    throw EthereumJSErrorWithoutCode("Expected branch node"); // ← 타입 가드 (아래 참조)
  }
  parentNode.setBranch(pathNibbles.pop()!, null); // 부모의 해당 슬롯을 비움
  stack.push(parentNode);
  await this.saveStack(pathNibbles, stack, opStack);
  return;
}
```

**케이스 1 승격의 정합성**: 왜 leaf 키가 `[]`에서 시작하나 — branch 자체는 경로 니블을 소비하지 않는다(니블은 부모의 슬롯 번호로 소비됨). 그 branch가 있던 "위치"까지의 경로는 부모들이 이미 표현하고 있으므로, leaf는 빈 키로 태어나서 부모가 Ext면 그 복도 경로만 물려받으면 된다.

**케이스 2가 사실상 방어 코드인 이유**: 규범형 트라이에서 "값도 자식도 없는 branch"는 존재할 수 없다(만들어질 이유가 없다). 위 케이스 1과 버그 ①·③ 수정이 규범형을 회복시키므로 이 분기는 정상 흐름에서 도달 불가 — 가혹 스트레스 800사이클에서도 미도달. 그래도 방어적으로 남겨뒀다.

**타입 가드의 사연** (검증 단계의 가치): 이 블록의 첫 버전은 `parentNode.setBranch(...)`를 바로 호출했는데 — **vitest는 esbuild로 타입을 벗겨내고 실행하므로 1511개 테스트가 전부 통과**했다. 그런데 `tsc` 빌드에서:

```
error TS2339: Property 'setBranch' does not exist on type 'BranchMPTNode | LeafMPTNode'
```

그 지점의 `parentNode` 타입이 유니온이라(undefined/Ext만 걸러진 상태) TS가 `setBranch` 존재를 보장 못 한다. 같은 함수 위쪽에 이미 있던 관용구(`if (!(x instanceof BranchMPTNode)) throw`)를 재사용해 마감. **"테스트 전부 통과"와 "빌드 통과"는 독립된 검증**이라는 교훈의 실물.

### 버그 ④(검증기) 수정 — `verifyPrunedIntegrity`: 참조의 두 형태

```diff
 if (node instanceof ExtensionMPTNode) {
-  if (bytesToUnprefixedHex(node.value()) === dbKeyHex) {
+  // Inline child nodes are not stored in the DB and therefore do not
+  // correspond to a database key that can be checked here.
+  if (!isRawMPTNode(node.value()) && bytesToUnprefixedHex(node.value()) === dbKeyHex) {
     found = true
     return
   }
```

Extension의 자식 참조(`node.value()`)는 §1-6대로 **두 형태**다: 32바이트 해시(Uint8Array) 또는 임베디드 원본(Uint8Array**[]**). 임베디드 자식은 DB에 저장되지 않으므로 "DB 키와 대조"라는 이 검사 자체가 성립하지 않는다 — 건너뛰는 게 옳다. (Branch 쪽 루프는 원래부터 `isRawMPTNode` 분기가 있었다. Extension만 빠져 있던 비대칭.)

### 회귀 테스트 3종 — `prune.spec.ts`

```ts
// ① 6-op: 버그 ①의 "루트 증발" — useRootPersistence 양쪽 매트릭스
it('should retain a promoted node when pruning variable-length keys', async () => {
  for (const useRootPersistence of [false, true]) {
    const trie = new MerklePatriciaTrie({ useNodePruning: true, useRootPersistence })
    /* put×4, del×2 — §3의 그 시퀀스 그대로 */
    assert.isTrue(await trie.verifyPrunedIntegrity())          // 고아 없음
    assert.deepEqual(await trie.get(hexToBytes('0x51')), …)    // ★ 생존 키가 "읽히는가"
    assert.deepEqual(await trie.get(hexToBytes('0x57179cc0')), …)
    assert.isNull(await trie.get(hexToBytes('0x04')))          // 삭제 키는 없어야
  }
})
```

★가 핵심 단언이다 — 버그의 최종 증상이 "루트 실종 → 읽기 크래시"였으므로, 무결성 검사만으론 부족하고 **실제 읽기**가 성공해야 한다.

```ts
// ② 2-op 접두사 쌍: 버그 ②의 "삭제가 끝나지 않음"
it("should collapse an empty branch after deleting a longer key first", async () => {
  /* put(0x7170), put(0x71), del(0x7170), del(0x71) */
  assert.isTrue(equalsBytes(trie.root(), KECCAK256_RLP)); // 루트가 빈 트라이 상수로
  assert.isTrue(await trie.verifyPrunedIntegrity());
  assert.isEmpty((trie["_db"].db as any)._database.keys()); // DB도 완전히 비어야
});
```

`KECCAK256_RLP` = `keccak256(RLP(빈 바이트))` = **빈 트라이의 규범 루트 상수.** "전부 지우면 이 값으로 돌아와야 한다"가 트라이의 기본 계약이다.

```ts
// ③ 7-op 이중 삭제: 버그 ③의 "가짜 삭제"
it('should not mutate pruning state when deleting a missing prefix key twice', async () => {
  /* …del(0x29), del(0x29) — 같은 접두사 키를 두 번 삭제 */
  assert.isTrue(await trie.verifyPrunedIntegrity())
  assert.deepEqual(await trie.get(hexToBytes('0xda767b99')), …)   // 무관한 키들 무사 확인
})
```

세 시퀀스 모두 손으로 지은 게 아니라 **델타 디버깅 축소본** — "이 매직 넘버들은 어디서 왔나"는 리뷰어 질문에 대한 답을 PR 본문에 명시했다.

---

## 6. 검토 기록 — AI 도구로 적대적 검토

### 1차 검토: "수정은 옳다, 그러나 불완전하다"

| 항목                         | 판정                                                          |
| ---------------------------- | ------------------------------------------------------------- |
| 근본 원인 진단               | ✅ 정확 — 배치 가설은 증상, `value(null)` 발견이 진짜 원인 |
| 파급 반경 (호출처 전수 grep) | ✅ 모노레포에서 1곳                                           |
| 6-op 회귀                    | ✅ 통과                                                       |
| 전체 mpt 스위트              | ✅ 무손상                                                     |
| **랜덤 스트레스**            | ❌ **여전히 실패** → 재축소로 [B] 2-op, [A] 7-op 발굴         |
| [A][B]가 수정이 만든 회귀?   | git stash → 수정 전 master에서 재현 → **기존 버그** 판정             |

이 검토가 없었으면 부분 수정 PR이 나갔고, 메인테이너가 이슈의 스트레스 재현을 돌리는 순간 반려됐을 것이다 (node-redis #3329의 "부분 수정" 반려 코스와 같은 패턴).

### 2차 검토: 완료 판정 + 마감 1건

| 항목                                                                                                                                                            | 결과                                                  |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| 회귀 3종 + 랜덤 스트레스 (1~8바이트 키, 300사이클×양쪽 설정)                                                                                                    | ✅                                                    |
| **가혹 스트레스** (1~3바이트 키 = 접두사 밀도 극대화, 존재 무관 삭제 포함, 400사이클×양쪽, **매 연산** 무결성 검사, 사이클마다 delete-all→빈루트→빈DB 3중 확인) | ✅                                                    |
| 전체 mpt 스위트                                                                                                                                                 | ✅ 1511/1511                                          |
| statemanager 교차 검증 (최대 트라이 소비처, 822개)                                                                                                              | ✅                                                    |
| biome                                                                                                                                                           | ✅                                                    |
| **빌드(tsc)**                                                                                                                                                   | ❌ 타입 에러 1건 → 타입 가드로 마감 (5장 버그 ② 참조) |

---

## 7. PR #4346 본문 전략

1. **근본 원인을 서사로**: setter 무시 → 유령 값 → collapse 오염 → "같은 배치에서 새 루트 저장·삭제" → **트라이 판독 불가** (pruning 미관 문제가 아니라 데이터 손실임을 각인)
2. **[A][B]가 수정 전 master에서 독립적으로 재현됨을 명시** — "이 PR이 새 버그를 만든 것 아니냐"를 선제 차단
3. **회귀 테스트가 델타 디버깅 축소본임을 명시** — 매직 넘버 시퀀스의 출처 설명
4. **검증 수치를 구체적으로** — 수백 사이클, 매 연산 무결성 검사, 1~3바이트 키, 1511+822
5. AI 흔적 0 (커밋 메시지 grep 검증까지 완료 후 push)

---

## 8. 핵심 교훈 (회고)

1. **버그는 무리 지어 산다.** 랜덤 테스트는 항상 가장 흔한 버그에서 먼저 죽으므로, 뒤의 버그들은 첫 버그가 사라져야 관측된다. "수정 → 스트레스 재실행 → 신규 실패 축소"의 루프를 **실패가 마를 때까지** 돌려야 다 잡힌 것이다.

2. **증상과 진짜 원인을 구별하라.** 배치의 put-후-del은 극적인 증거였지만 하류의 증상이었다. 그 레벨에서 고쳤다면(교집합 필터) 유령 값은 살아남아 다른 데서 또 문제를 일으켰을 것. "이 이상한 상태는 **어디서 처음** 만들어졌나"를 계속 거슬러 올라가야 진짜 원인이 나온다.

3. **손으로 고른 최소 케이스는 상상력의 한계에 갇힌다.** 2~3키 전수 탐색(1차 축소)은 실패했다 — update와 작은 값이라는 재료를 몰랐으니까. 실패 찾기는 무작위에게, 줄이기는 델타 디버깅에게. 하네스 제작 30분, 산출물은 "2-op 재현 + 그대로 회귀 테스트".

4. **검증기부터 검증하라.** verifyPrunedIntegrity 자체가 크래시 버그를 갖고 있었다. 판정 도구가 오염되면 모든 판정이 무의미 — 독립적인 이중 장부(DB vs 도달가능 집합)로 도구와 본체를 분리 검증했다.

5. **테스트 러너와 타입 체커는 다른 것을 본다.** vitest(esbuild)는 타입을 안 본다. 1511개 전부 통과 ≠ 빌드 통과. 검증 단계에는 반드시 `tsc`가 들어가야 한다.

6. **getter/setter 겸용 메서드의 sentinel은 `undefined` 하나로 족하다.** "인자 없음"과 "명시적 null"을 같게 취급한 가드 한 줄이 4년짜리 데이터 손실 버그가 됐다. null은 "지워라"라는 유효한 신호일 수 있다.

7. **고정 길이 키의 그늘.** secure trie(keccak 키)에서는 접두사 키가 불가능해 버그 코드가 실행조차 안 됐다. "주력 사용처에서 안 터진다" ≠ "코드가 옳다" — 지원하는 입력 공간 전체가 테스트 대상이다.

8. **수정 전 파급 반경 측정은 grep 한 줄부터.** `value(null)` 호출처가 모노레포에서 1곳뿐이라는 사실이 "코어 클래스의 의미 변경"을 안전하게 만든 근거의 전부였고, 리뷰 방어 논리로도 그대로 쓰였다.

9. **"이 수정이 만든 회귀인가"는 stash로 즉답할 수 있다.** 새 실패 발견 → `git stash` → 수정 전 상태에서 재생 → `git stash pop`. 1분짜리 실험이 PR의 방어 문장("pre-existing, reproduces on master")을 만들어준다.

10. **구현과 검토를 분리하는 실험.** 근본 원인 발견·수정은 직접 했고, 재현·축소·적대적 검토·검증 단계는 AI 도구의 도움을 받았다. 만든 사람과 검토하는 관점을 분리하는 게 왜 강력한지 실물로 확인했다 — 1차 검토가 부분 수정의 조기 출하를 막았고, 2차 검토가 타입 에러를 잡았다.

---

## 9. 개념 미니 노트 (복습용 축약판)

- **keccak256**: 이더리움의 해시 함수, 32바이트 고정 출력. 결정성·눈사태·역산불가
- **머클 루트**: 트리 전체를 요약하는 최상단 해시. 32바이트 비교 = 전체 데이터 비교
- **머클 증명**: 경로의 형제 해시들만으로 소속을 증명. 라이트 클라이언트의 원리
- **트라이**: 키의 글자가 곧 경로. 접두사 공유 = 경로 공유. 한 키가 다른 키의 접두사일 수 있음
- **니블**: 4비트 = 헥스 문자 1개. MPT의 경로 단위 (16-way 분기의 근거)
- **Branch**: 슬롯 16 + **value 슬롯**(여기서 끝나는 키의 값 — 이번 버그가 생기는 지점)
- **Extension**: 한 갈래 복도의 압축. 자식 참조는 해시 또는 임베디드 원본
- **Leaf**: 남은 경로 + 값. 종착지
- **내용 주소화**: DB 키 = 내용의 해시. 같은 모양으로 회귀 = 같은 해시 (정상 성질이자 이번 증상의 재료)
- **임베디드 노드**: 직렬화 32바이트 미만이면 DB 저장 없이 부모에 인라인. `isRawMPTNode`로 판별
- **RLP**: 이더리움의 직렬화 포맷. `KECCAK256_RLP` = 빈 트라이의 규범 루트 상수
- **secure trie**: 키를 keccak해서 사용(state/storage). 32바이트 고정 → 접두사 불가 → 이 버그 면역
- **SSTORE 0**: 스토리지 슬롯 삭제. 이더리움에서 트라이 삭제가 일상적으로 일어나는 경로
- **pruning**: 참조 끊긴 옛 노드의 즉시 삭제 옵션. 장부(삭제 목록)를 먼저 만들고 새 노드 저장 후 배치 삭제
- **collapse**: 삭제 후 규범형 복원 — 자식 1개면 위로 합침, 0개면(이번 신설) 값 유무 따라 Leaf 승격/제거
- **델타 디버깅**: "빼도 실패하면 빼라"의 고정점 반복. 실패 입력의 자동 최소화
- **sentinel**: "인자 없음"의 표식. 옵셔널 인자에선 `undefined`만 써라 — null은 데이터다

---

## 10. 결과 및 회고 — 🟡 요청 2건 반영·푸쉬 완료, 후속 코멘트까지 남김, 메인테이너 재검토 대기

### 10.0 현재 상태 (2026-07-24 기준) — 반영은 끝났는데 알림 코멘트가 빠져 있던 걸 뒤늦게 발견

10.2절의 요청 2건은 실제로는 **2026-07-22에 이미 커밋(`2159bf325`, `test(mpt): add randomized variable-length-key delete stress test`)되고 fork에 푸쉬까지 끝나 있었다** — 이 문서가 2026-07-21 시점에서 갱신을 멈췄던 탓에 "미반영"으로 잘못 남아있던 것. PR 본문에도 이미 "Scope / compatibility" 섹션이 반영돼 있었다.

문제는 **그 작업을 했다는 걸 메인테이너에게 알리는 후속 코멘트가 없었다는 것.** 이슈 코멘트 / 정식 리뷰 / 인라인 리뷰 코멘트 세 API를 전부 조회해서 확인한 결과, 07-21 `gabrocheleau`의 리뷰 이후 남은 코멘트는 codecov 봇의 자동 갱신(07-23, 새 푸쉬 감지로 봇이 자기 코멘트를 스스로 갱신한 것일 뿐 우리가 남긴 게 아님)뿐이었다. 즉 **요청을 이미 다 반영해놓고, 그 사실을 알리는 절차만 빠뜨린 상태**로 이틀이 지나 있었다.

재검증도 다시 했다: `git checkout d8475d19c~1 -- packages/mpt/src/mpt.ts packages/mpt/src/node/branch.ts`로 fix 소스만 잠깐 되돌리고 새 테스트를 돌려서 실제로 실패(`trie is empty after deleting all keys: expected false to be true`)하는 걸 확인 → 원복 후 `prune.spec.ts` 14/14 재통과 확인. 메인테이너가 보고한 수치를 그대로 믿지 않고 직접 재현했다.

이후 다음 코멘트를 남겼다 (https://github.com/ethereumjs/ethereumjs-monorepo/pull/4346#issuecomment-5064842983):

```markdown
Pushed both updates.

Added the scope note to the PR body, and added a randomized delete-all test in `prune.spec.ts` with 1-4 byte keys so prefix pairs can occur. The test checks pruned integrity and the empty root after all keys are deleted.

The new test fails on `master` without the fix and passes with this branch.
```

**교훈**: 반영 작업 자체를 끝내는 것과, 그걸 상대(메인테이너)에게 알리는 것은 별개의 단계다. 특히 문서/기록이 작업 세션 경계를 넘어가면(이번처럼 다른 세션에서 조용히 커밋+푸쉬만 하고 끝난 경우), "다 했다"와 "다 했다고 알렸다" 사이의 간극이 기록에서 누락되기 쉽다. PR 작업의 완료 기준에 "메인테이너에게 통지"까지 명시적으로 포함시켜야 한다.

### 10.1 이전 상태 기록 (2026-07-21 기준) — 메인테이너 리뷰 완료, 머지 전 요청 2건 대기

- PR #4346 **여전히 OPEN**, 아직 머지 안 됨
- 메인테이너 `gabrocheleau`가 코멘트로 직접 재현·검증까지 마치고 **"real, correctly-fixed bug"**라고 확인함. 직접 재현한 결과를 표로 남김:

  |                    | `get(0x51)` (삭제 후)             | 전체 삭제 후 루트            |
  | ------------------ | --------------------------------- | ---------------------------- |
  | master, pruning 끔 | `"short"` ❌ (삭제가 조용히 실패) | 비어있지 않음 ❌             |
  | master, pruning 켬 | `null`                            | 비어있지 않음 ❌ (고아 노드) |
  | 이 PR (양쪽 모두)  | `null` ✅                         | `KECCAK256_RLP` ✅           |

- 근본 원인 진단(`BranchMPTNode.value(null)`이 조용히 no-op이었다는 것, §4-7)에 대해 **"spot on"이라고 정확히 동의**함.
- **범위(scope)에 대한 메인테이너의 확인**: 이 버그는 **가변 길이 키(raw / `skipKeyTransform`)를 쓰는 트라이에만** 영향을 준다는 것을 메인테이너 본인이 재확인해줌 — secure state/storage trie(고정 32바이트 keccak 키)는 한 키가 다른 키의 접두사가 될 수 없어서 이 경로를 절대 안 타고, 따라서 **이더리움 상태 루트/합의에는 영향 없음**. 이게 바로 공식 fixture 스위트가 4년간 이 버그를 못 잡은 이유. 이 문서 §1-8에서 이미 다뤘던 가설이 메인테이너에 의해 정확히 확인된 것.

### 10.2 머지 전 요청 2건 (아직 반영 안 됨 — 다음 작업)

1. **PR 본문에 스코프/호환성 note를 명시적으로 추가하라.** "raw 트라이 전용, 합의 영향 없음"이라는 확인 내용을 PR 설명에 넣어서, 리뷰어가 한눈에 이게 상태 루트에 영향 없는 수정임을 알 수 있게 하라는 요청. 부수적으로 `value(null)`이 이제 no-op이 아니라 실제로 값을 지운다는 것도 한 줄 언급해달라고 함(`put()`이 빈 값을 `del()`에 위임하는 구조라 내부적으로만 쓰이는 변경이긴 함).
2. **가변 길이 키 전용 랜덤 삭제 테스트를 추가하라.** 기존 스트레스 테스트가 전부 고정 길이 키(`randomBytes(8/20/32)`)만 써서 **구조적으로 접두사 관계가 생길 수 없다** — 이게 이 버그가 살아남은 이유이기도 하다고 정확히 지적함. 실제로 메인테이너가 제안한 가변 길이(1~4바이트) 랜덤 삭제 테스트를 50회 돌려보니 **master에서 약 18/50 실패, 이 PR에서는 0/50**이라는 구체적 수치까지 코멘트에 남기고, 테스트 코드 초안까지 통째로 제공함.

- 그 외 요청 없음 — "everything else looks good, typecheck clean and the full MPT suite... passes unchanged"라고 확인. 리뷰 자체는 **"돕고 싶다"는 협조적인 톤**("Happy to help get these in if useful").

### 10.3 다음에 할 일

- [x] PR 본문에 스코프/호환성 note 추가 (2026-07-22, PR 본문에 이미 반영됨)
- [x] 메인테이너가 제안한 가변 길이 키 랜덤 삭제 테스트를 `prune.spec.ts`에 추가 (2026-07-22, 커밋 `2159bf325`)
- [x] 두 요청 반영 후 다시 push (2026-07-22 완료), 반영 사실을 알리는 후속 코멘트 (2026-07-24 완료)
- [x] 메인테이너 최종 승인 대기 — `gabrocheleau`가 머지로 마무리
- [ ] 연관 이슈 #3645(트라이 checkpoint 테스트 부족)와의 연계 여부
- [x] 최종 머지 여부 및 머지까지 걸린 시간 — 2026-07-28 14:25 머지됨
