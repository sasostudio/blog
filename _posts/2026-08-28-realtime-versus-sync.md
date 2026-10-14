---
layout: post
title: "뭉치뭉치 - 실시간 대전 동기화"
description: "싱글 플레이용 물리 퍼즐 엔진을 수정하지 않고 1대1 실시간 대전을 붙인 구조. 보드 1장 = 참가자 1명, 봇과 원격 상대를 같은 Opponent 계약으로 교체, 스냅샷 스트림, 고정 논리 치수로 지킨 공정성."
date: 2026-08-28 02:00:00 +0900
project: moongchi
tags: [game-dev, multiplayer, architecture, build-in-public]
---

*🇬🇧 [English version below](#english-version)*

싱글 플레이용으로 만든 물리 퍼즐 엔진에 1대1 실시간 대전을 추가. 봇전을 먼저 내고 사람 대 사람을 나중에 개방했는데, **엔진 수정 없이** 두 모드를 같은 코드로 실행하는 구조.

승패 판정을 서버로 옮긴 이야기는 [이전 글](/2026/07/27/you-cant-report-your-own-win/)에서 정리. 이 글의 주제는 그 아래 구조.

## 1. 보드 1장 = 참가자 1명

엔진의 진입점은 `createGame(canvas, skills, levels, opts)` 하나. 캔버스 한 장을 받아 물리·상태·루프를 통째로 소유하고 핸들 객체를 반환하는 구조.

대전용 별도 엔진은 미제작. `combatant.ts`가 `createGame()`을 감싸 **대전용 오버라이드만 추가.**

| `opts` | 의미 |
|---|---|
| `layout.*` | 캔버스 안 컨테이너 기하 |
| `forcedZone` | 중력·발사 방향 고정(`top`/`bottom`) — 대전 전용 |
| `logicalSize` | 논리 좌표계 고정 |

그래서 물리·합체·콤보·게임오버 로직이 두 모드에서 한 벌. 밸런스 조정 하나가 양쪽에 동시 반영되고, 반대로 한쪽만 고치는 것은 불가하다는 의미.

## 2. `Opponent` 계약 — 교체점 하나

```
opponent.ts        인터페이스(공격 이벤트 · 스냅샷 · fx)
 ├ botOpponent.ts     로컬 봇: 두 번째 물리 보드를 실제로 돌린다
 └ remoteOpponent.ts  원격: 물리를 돌리지 않는다 — 수신 스냅샷을 재렌더만 한다
```

총괄 상태머신(`matchController.ts`)은 **상대가 봇인지 사람인지 미인지.** 봇 → 원격 전환은 주입 교체 한 번.

이 설계 덕분에 대전의 단계적 출시가 가능. 1단계는 봇전만 내고 게임성과 UI를 확정. 2단계에서 통신을 붙이는데, 이때 손대는 범위는 주입되는 구현체와 그 아래 통신 계층뿐.

한 가지 규칙이 이 구조의 기둥. **`versus/net/`은 격리 계층이고, 여기서 게임 로직을 참조하기 시작하면 교체점이 붕괴.**

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/realtime-versus-sync/structure.ko.svg"><img src="/assets/img/posts/realtime-versus-sync/structure.ko.svg" width="720" height="320" loading="lazy" alt="matchController 상태머신은 상대의 정체를 모르고 Opponent 계약(공격 이벤트·스냅샷·fx)만 인지. 그 아래 botOpponent는 두 번째 물리 보드를 실제로 실행, remoteOpponent는 물리 없이 수신 스냅샷만 재렌더. 엔진은 한 벌, combatant.ts가 대전용 오버라이드만 추가. versus/net/은 격리 계층."></a></div><figcaption>봇에서 원격으로의 전환은 주입 구현체 하나의 교체.</figcaption></figure>

## 3. 공격 모델의 순수함수 분리

| 파일 | 줄 수 | 역할 |
|---|:-:|---|
| `attackModel.ts` | 44 | 콤보 × 병합 레벨 → 방해 개수. 순수함수 |
| `attackQueue.ts` | 108 | 예고 지연 큐 + 상쇄(받는 쪽이 콤보를 내면 차감). 상태는 호출자 소유, 여기는 변환만 담당 |

밸런스 공식은 자주 바뀌고 **틀려도 티가 안 나는 영역.** 부작용을 없애면 Node로 즉시 검증 가능. 이 프로젝트는 테스트 러너 대신 `*.selfcheck.ts`를 순수 모듈 옆에 두고 직접 실행하는 방식.

```
npx tsx src/versus/net/matchResult.selfcheck.ts
```

브라우저·DOM·Firebase 의존이 있으면 대상 제외. 그래서 **검증 가능한 형태로 쪼개는 작업이 테스트 인프라 구축보다 우선.** 경계값을 코드로 박아 두면 밸런스 수치를 고칠 때 무엇이 깨지는지 즉시 확인 가능.

## 4. 스냅샷 스트림

| 파일 | 역할 |
|---|---|
| `snapshotStream.ts` | 보드 상태 델타 인코딩 + 양자화 |
| `localSnapshotSource.ts` | 내 보드 → 15Hz 송신 |
| `snapshotRenderer.ts` | 수신 스냅샷 렌더 |
| `rtdbRelay.ts` | 중계 |

대역폭을 결정하는 요소는 두 가지.

**양자화가 정지 판정을 대체.** 오브젝트의 정지 여부를 별도로 계산하지 않는 구조. 양자화한 값이 직전과 같으면 그 오브젝트는 델타에서 자동 제외. 합체 퍼즐은 대부분의 오브젝트가 대부분의 시간 동안 쌓여 있으므로, 이것만으로도 전송량이 대폭 감소.

**송신은 엔진 수정 없이 연결.** `localSnapshotSource`는 엔진이 이미 노출하는 오브젝트 위치만 읽는 「읽기 탭」. 엔진 쪽에 송신용 훅을 뚫지 않았기 때문에, 통신 기능이 추가돼도 싱글 플레이 경로의 동작은 불변.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/realtime-versus-sync/snapshot-flow.ko.svg"><img src="/assets/img/posts/realtime-versus-sync/snapshot-flow.ko.svg" width="720" height="200" loading="lazy" alt="내 보드의 위치값만 읽는 탭 → 양자화·델타(멈춘 오브젝트는 자동 제외) → 15Hz 송신 → RTDB 중계 → 상대 화면에서 봇전과 같은 그리기 함수로 재렌더."></a></div><figcaption>엔진에 송신용 훅을 뚫지 않았으므로 싱글 플레이 경로는 불변.</figcaption></figure>

## 5. 원격 보드와 봇전의 동일 렌더링

원격 상대의 보드도 봇전과 **완전히 같은** 연출·프레임으로 렌더링 필요. 다르면 사람들은 그 차이를 렉이나 버그로 인식.

그러려면 렌더 코드 공유가 필수인데, `renderer.ts`는 원칙적으로 수정 금지 파일. 그래서 **본문을 한 글자도 바꾸지 않고** `fxRender.ts`·`frameRender.ts`로 이동. diff 0을 검증 대상으로 명시하고, 새 파일 헤더에 예외 사유를 기록.

여기서 얻은 규칙: **공유가 필요할 때 리팩토링과 기능 변경을 같은 커밋에 섞지 않는 원칙.** 섞으면 「동작이 달라진 게 이동 때문인지 변경 때문인지」를 나중에 판별 불가.

대가도 존재. 이 세 파일 중 하나를 고치면 싱글·봇전·원격 세 화면에 동시 반영. 한 곳을 고치면 세 곳 확인 필요.

## 6. 화면 크기의 승패 개입 차단

물리 보드가 캔버스 픽셀에서 유도되면 **큰 화면 = 큰 수용량.** 합체 퍼즐에서 더 많이 담긴다는 건 곧 유리하다는 의미, 대전에서는 불공정.

그래서 대전은 논리 치수를 고정(`360×390`). 물리는 그 위에서 돌고, 화면은 결과를 확대 표시만 담당. 창 크기를 바꿔도 물리적 변화는 0.

보드 위아래 배치를 바꾸는 기능도 같은 원칙으로 처리. 방향(중력·발사 원점·바구니 위치)이 정준 좌표계에 각인된 구조라 DOM 이동만으로는 재배향 불가, 좌표계를 건드리면 상대 화면과 정산까지 영향. 그래서 **캔버스 출력을 CSS `scaleY(-1)`로 뒤집고 텍스트만 미리 반전.** 물리·스냅샷·네트워크는 무변경.

## 7. 로비와 재접속

방 생성보다 어려웠던 과제는 **양쪽의 실제 준비 완료 여부** 판정.

- 로비 시작 조건은 **양측 ready + presence의 동시 충족.** ready 플래그만 보면, 창을 닫아 버린 사람의 ready가 그대로 남아 유령 상태로 대전이 시작되는 문제 발생.
- 재입장은 **같은 `clientId`만** 허용. 방 선점의 원자성 유지가 목적.
- 대전 중 새로고침 복귀는 1Hz 키프레임을 로컬에 복사해 두고, 복귀 시 엔진의 additive API로 보드를 재구성. 엔진 무수정 원칙의 명시적 예외 1건.

이걸 만들다 설계 전제 하나가 실측으로 붕괴. **RTDB push key가 항상 증가한다는 가정이었는데, 송신 측이 서버 시계 오프셋을 재보정하면 역행 가능.** 그래서 마지막으로 처리한 key를 단조 증가로만 갱신하는 게이트를 추가. 역행한 key가 들어오면 이미 처리한 것으로 간주.

## 정리

- 두 모드를 하나의 엔진으로 돌리려면 엔진에 모드를 넣지 말고, 엔진을 감싸는 얇은 층에 오버라이드를 추가.
- 봇과 원격을 같은 인터페이스로 만들면 단계적 출시가 가능. 대신 그 인터페이스 아래로 게임 로직이 새지 않도록 관리 필수.
- 공정성에 관계된 값은 화면에서 유도 금지. 논리 치수를 고정하고 표시만 확대.
- 코드 이동 커밋과 동작 변경 커밋은 분리. 원본 대비 diff 0을 검증 항목으로 둘 수 있다면 그대로 적용.

---

## English version

A physics puzzle engine originally built for single-player got 1v1 realtime versus bolted on. Bot matches shipped first, human-vs-human later, and both modes run on the same code **without modifying the engine.**

The server-side outcome judging is covered in an [earlier post](/2026/07/27/you-cant-report-your-own-win/). This one is about the structure underneath it.

### One board equals one combatant

The engine has a single entry point: `createGame(canvas, skills, levels, opts)`. It takes one canvas, owns physics, state, and the loop, and returns a handle.

Versus mode is not a second engine. `combatant.ts` wraps `createGame()` and layers on **only the versus overrides.**

| `opts` | Meaning |
|---|---|
| `layout.*` | Container geometry inside the canvas |
| `forcedZone` | Pin gravity and launch direction (`top`/`bottom`) — versus only |
| `logicalSize` | Fixed logical coordinate space |

Physics, merging, combos, and game-over logic therefore exist once for both modes. A balance change lands on both at the same time — which also means changing one mode alone isn't possible.

### The `Opponent` contract

```
opponent.ts        interface (attack events · snapshots · fx)
 ├ botOpponent.ts     local bot: actually runs a second physics board
 └ remoteOpponent.ts  remote: runs no physics — re-renders received snapshots
```

The match state machine (`matchController.ts`) **does not know whether the opponent is a bot or a person.** Switching from bot to remote is one injection swap.

That's what made staged shipping possible. Phase one shipped bots only and locked down feel and UI. Phase two added networking, touching only the injected implementation and the transport layer below it.

One rule holds the structure up: **`versus/net/` is an isolation layer, and the moment it starts referencing game logic, the swap point collapses.**

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/realtime-versus-sync/structure.en.svg"><img src="/assets/img/posts/realtime-versus-sync/structure.en.svg" width="720" height="320" loading="lazy" alt="The matchController state machine knows nothing about the opponent beyond the Opponent contract (attack events, snapshots, fx). Under it, botOpponent runs a second physics board and remoteOpponent runs no physics, only re-rendering snapshots. There is one engine; combatant.ts adds versus overrides. versus/net/ is an isolation layer."></a></div><figcaption>Switching from bot to remote is swapping one injected implementation.</figcaption></figure>

### Attack model as pure functions

| File | Lines | Role |
|---|:-:|---|
| `attackModel.ts` | 44 | combo × merge level → garbage count. Pure |
| `attackQueue.ts` | 108 | telegraph delay queue + cancellation (a combo on the receiving side subtracts). State is owned by the caller; this only transforms |

Balance formulas change often and **fail silently when wrong.** Removing side effects makes them verifiable from Node immediately. This project runs no test runner; `*.selfcheck.ts` files sit next to pure modules and execute directly.

```
npx tsx src/versus/net/matchResult.selfcheck.ts
```

Anything depending on the browser, the DOM, or Firebase is out of scope — which means **splitting code into a verifiable shape comes before standing up test infrastructure.** With boundary values pinned in code, changing a balance number tells you immediately what it breaks.

### Snapshot stream

| File | Role |
|---|---|
| `snapshotStream.ts` | Board state delta encoding + quantization |
| `localSnapshotSource.ts` | My board → 15Hz send |
| `snapshotRenderer.ts` | Render received snapshots |
| `rtdbRelay.ts` | Relay |

Two decisions determine bandwidth.

**Quantization substitutes for rest detection.** Nothing computes whether an object has come to rest. If the quantized value matches the previous one, that object drops out of the delta automatically. In a merge puzzle most objects are stacked and still most of the time, so this alone cuts the payload substantially.

**The send path was attached without modifying the engine.** `localSnapshotSource` is a read tap over object positions the engine already exposes. No send hook was cut into the engine, so the single-player path behaves identically whether networking is present or not.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/realtime-versus-sync/snapshot-flow.en.svg"><img src="/assets/img/posts/realtime-versus-sync/snapshot-flow.en.svg" width="720" height="200" loading="lazy" alt="A read-only tap on my board’s positions → quantize and delta (objects at rest drop out) → send at 15 Hz → RTDB relay → re-render on the opponent’s screen with the same draw code as bot matches."></a></div><figcaption>No send hook was cut into the engine, so the single-player path is unchanged.</figcaption></figure>

### Rendering the remote board exactly like a bot match

The remote opponent's board has to render with **identical** effects and framing to a bot match. Any difference reads as lag or a bug.

That requires sharing render code, and `renderer.ts` is a do-not-modify file by policy. So the code moved into `fxRender.ts` and `frameRender.ts` **without a single character of the body changing.** A diff of zero was written down as a verification item, and the new file headers record why the exception was granted.

The rule that came from it: **when sharing becomes necessary, don't mix refactoring and behavior change in the same commit.** Mixed, you can no longer tell whether a behavior difference came from the move or the change.

There is a cost. Editing any of those three files lands on three screens at once — single-player, bot match, remote. One fix, three surfaces to check.

### Keeping screen size out of the outcome

If the physics board derives from canvas pixels, **a bigger screen means bigger capacity.** In a merge puzzle, fitting more is simply an advantage, and in versus that's unfairness.

Versus therefore pins logical dimensions (`360×390`). Physics runs on that space and the screen only scales the result. Resizing the window does nothing physical.

Swapping which board sits on top uses the same principle. Orientation — gravity, launch origin, basket position — is baked into the canonical coordinate space, so DOM reordering can't reorient it, and touching the coordinate space would propagate to the opponent's view and to settlement. The canvas output is flipped with CSS `scaleY(-1)` and only text is pre-flipped. **Physics, snapshots, and networking are untouched.**

### Lobby and reconnection

Opening a room was easy. Establishing that **both sides are genuinely present** was not.

- The lobby starts only when **both ready flags and presence** are satisfied at once. Reading ready alone lets a closed window's stale ready flag start a ghost match.
- Re-entry is allowed **only for the same `clientId`**, preserving the atomicity of room claiming.
- Refresh-recovery mid-match tees 1Hz keyframes locally and rebuilds the board through the engine's additive API on return. This is the one explicit exception to the no-engine-modification rule.

Building it broke a design assumption by measurement: **RTDB push keys were assumed monotonic, but they can move backwards when the sender recalibrates its server clock offset.** The fix was a monotonic gate — the last-processed key only ever moves forward, so a regressed key is treated as already handled.

### Takeaways

- To run two modes on one engine, don't put the mode inside the engine. Put the overrides in a thin wrapper around it.
- Making bots and remote players share an interface enables staged shipping. In exchange, game logic must never leak below that interface.
- Anything that affects fairness must not be derived from the display. Pin logical dimensions and scale the presentation.
- Separate the commit that moves code from the commit that changes behavior. When a zero diff against the original can be a verification item, make it one.
