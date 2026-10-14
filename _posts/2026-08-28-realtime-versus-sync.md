---
layout: post
title: "뭉치뭉치 - 실시간 대전 동기화"
description: "싱글 플레이용 물리 퍼즐 엔진을 고치지 않고 1대1 실시간 대전을 붙인 구조. 보드 1장 = 참가자 1명, 봇과 원격 상대를 같은 Opponent 계약으로 교체, 스냅샷 스트림, 고정 논리 치수로 지킨 공정성."
date: 2026-08-28 02:00:00 +0900
project: moongchi
tags: [game-dev, multiplayer, architecture, build-in-public]
---

*🇬🇧 [English version below](#english-version)*

싱글 플레이용으로 만든 물리 퍼즐 엔진에 1대1 실시간 대전을 붙였다. 봇전을 먼저 내고 사람 대 사람을 나중에 열었는데, **엔진을 수정하지 않고** 두 모드를 같은 코드로 돌린다.

승패 판정을 서버로 옮긴 이야기는 [이전 글](/2026/07/27/you-cant-report-your-own-win/)에 썼다. 이 글은 그 아래 구조다.

## 1. 보드 1장 = 참가자 1명

엔진의 진입점은 `createGame(canvas, skills, levels, opts)` 하나다. 캔버스 한 장을 받아 물리·상태·루프를 통째로 소유하고 핸들 객체를 반환한다.

대전은 별도 엔진을 만들지 않았다. `combatant.ts`가 `createGame()`을 감싸 **대전용 오버라이드만 얹는다.**

| `opts` | 뜻 |
|---|---|
| `layout.*` | 캔버스 안 컨테이너 기하 |
| `forcedZone` | 중력·발사 방향 고정(`top`/`bottom`) — 대전 전용 |
| `logicalSize` | 논리 좌표계 고정 |

그래서 물리·합체·콤보·게임오버 로직이 두 모드에서 한 벌이다. 밸런스 조정 하나가 양쪽에 동시에 반영되고, 반대로 한쪽만 고치는 게 불가능하다는 뜻이기도 하다.

## 2. `Opponent` 계약 — 교체점 하나

```
opponent.ts        인터페이스(공격 이벤트 · 스냅샷 · fx)
 ├ botOpponent.ts     로컬 봇: 두 번째 물리 보드를 실제로 돌린다
 └ remoteOpponent.ts  원격: 물리를 돌리지 않는다 — 수신 스냅샷을 재렌더만 한다
```

총괄 상태머신(`matchController.ts`)은 **상대가 봇인지 사람인지 모른다.** 봇 → 원격 전환이 주입 교체 한 번이다.

이 설계 덕분에 대전을 단계로 출시할 수 있었다. 1단계는 봇전만 내고 게임성과 UI를 확정한다. 2단계에서 통신을 붙이는데, 이때 손대는 건 주입되는 구현체와 그 아래 통신 계층뿐이다.

한 가지 규칙이 이 구조를 지탱한다. **`versus/net/`은 격리 계층이고, 여기서 게임 로직을 참조하기 시작하면 교체점이 무너진다.**

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/realtime-versus-sync/structure.ko.svg"><img src="/assets/img/posts/realtime-versus-sync/structure.ko.svg" width="720" height="320" loading="lazy" alt="matchController 상태머신은 상대의 정체를 모르고 Opponent 계약(공격 이벤트·스냅샷·fx)만 안다. 그 아래 botOpponent는 두 번째 물리 보드를 실제로 돌리고, remoteOpponent는 물리 없이 수신 스냅샷만 재렌더한다. 엔진은 한 벌이고 combatant.ts가 대전용 오버라이드만 얹는다. versus/net/은 격리 계층이다."></a></div><figcaption>봇에서 원격으로의 전환은 주입되는 구현체 하나를 바꾸는 일이다.</figcaption></figure>

## 3. 공격 모델은 순수함수로 떼어낸다

| 파일 | 줄 수 | 역할 |
|---|:-:|---|
| `attackModel.ts` | 44 | 콤보 × 병합 레벨 → 방해 개수. 순수함수 |
| `attackQueue.ts` | 108 | 예고 지연 큐 + 상쇄(받는 쪽이 콤보를 내면 차감). 상태는 호출자 소유, 여기는 변환만 |

밸런스 공식은 자주 바뀌고 **틀려도 티가 안 난다.** 부작용을 없애면 Node로 즉시 검증된다. 이 프로젝트는 테스트 러너를 쓰지 않고 `*.selfcheck.ts`를 순수 모듈 옆에 두고 직접 실행한다.

```
npx tsx src/versus/net/matchResult.selfcheck.ts
```

브라우저·DOM·Firebase 의존이 있으면 대상이 안 된다. 그래서 **검증 가능한 형태로 쪼개는 것이 테스트 인프라를 세우는 것보다 먼저**가 된다. 경계값을 코드로 박아두면 밸런스 수치를 고칠 때 무엇이 깨지는지 즉시 나온다.

## 4. 스냅샷 스트림

| 파일 | 역할 |
|---|---|
| `snapshotStream.ts` | 보드 상태 델타 인코딩 + 양자화 |
| `localSnapshotSource.ts` | 내 보드 → 15Hz 송신 |
| `snapshotRenderer.ts` | 수신 스냅샷 렌더 |
| `rtdbRelay.ts` | 중계 |

두 가지가 대역폭을 결정한다.

**양자화가 정지 판정을 대신한다.** 오브젝트가 멈췄는지 따로 계산하지 않는다. 양자화한 값이 직전과 같으면 그 오브젝트는 델타에서 자동으로 빠진다. 합체 퍼즐은 대부분의 오브젝트가 대부분의 시간 동안 쌓여 있으므로, 이것만으로 전송량이 크게 줄어든다.

**송신은 엔진을 수정하지 않고 붙였다.** `localSnapshotSource`는 엔진이 이미 노출하고 있는 오브젝트 위치만 읽는 "읽기 탭"이다. 엔진 쪽에 송신용 훅을 뚫지 않았기 때문에 통신 기능이 추가돼도 싱글 플레이 경로의 동작은 변하지 않는다.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/realtime-versus-sync/snapshot-flow.ko.svg"><img src="/assets/img/posts/realtime-versus-sync/snapshot-flow.ko.svg" width="720" height="200" loading="lazy" alt="내 보드의 위치값을 읽기만 하는 탭 → 양자화·델타(멈춘 오브젝트는 자동으로 빠짐) → 15Hz 송신 → RTDB 중계 → 상대 화면에서 봇전과 같은 그리기 함수로 재렌더."></a></div><figcaption>엔진에 송신용 훅을 뚫지 않았으므로 싱글 플레이 경로는 변하지 않는다.</figcaption></figure>

## 5. 원격 보드를 봇전과 똑같이 그리기

원격 상대의 보드도 봇전과 **완전히 같은** 연출·프레임으로 그려야 한다. 다르면 사람들은 그걸 렉이나 버그로 읽는다.

그러려면 렌더 코드를 공유해야 하는데, `renderer.ts`는 원칙적으로 수정 금지 파일이다. 그래서 **본문을 한 글자도 바꾸지 않고** `fxRender.ts`·`frameRender.ts`로 옮겼다. diff 0을 검증 대상으로 명시하고, 새 파일 헤더에 왜 예외를 뒀는지 남겼다.

여기서 얻은 규칙: **공유가 필요할 때 리팩토링과 기능 변경을 같은 커밋에 섞지 않는다.** 섞으면 "동작이 달라진 게 이동 때문인지 변경 때문인지"를 나중에 못 가른다.

대가도 있다. 이 세 파일 중 하나를 고치면 싱글·봇전·원격 세 화면에 동시에 반영된다. 한 곳을 고치면 세 곳을 봐야 한다.

## 6. 화면 크기가 승패에 개입하지 않게

물리 보드가 캔버스 픽셀에서 유도되면 **큰 화면 = 큰 수용량**이 된다. 합체 퍼즐에서 더 많이 담긴다는 건 곧 유리하다는 뜻이고, 대전에서는 불공정이다.

그래서 대전은 논리 치수를 고정한다(`360×390`). 물리는 그 위에서 돌고, 화면은 결과를 늘려서 보여주기만 한다. 창 크기를 바꿔도 물리적으로는 아무 일도 일어나지 않는다.

보드 위아래 배치를 바꾸는 기능도 같은 원칙으로 처리했다. 방향(중력·발사 원점·바구니 위치)이 정준 좌표계에 각인돼 있어서 DOM 이동만으로는 재배향이 불가능한데, 좌표계를 건드리면 상대 화면과 정산까지 영향이 간다. 그래서 **캔버스 출력을 CSS `scaleY(-1)`로 뒤집고 텍스트만 미리 뒤집었다.** 물리·스냅샷·네트워크는 무변경이다.

## 7. 로비와 재접속

방을 여는 것보다 **양쪽이 진짜 준비됐는지**가 어려웠다.

- 로비는 **양측 ready + presence가 동시에 충족될 때만** 시작한다. ready 플래그만 보면, 창을 닫아버린 사람의 ready가 그대로 남아 유령 상태로 대전이 시작된다.
- 재입장은 **같은 `clientId`만** 허용한다. 방 선점의 원자성을 유지하기 위해서다.
- 대전 중 새로고침 복귀는 1Hz 키프레임을 로컬로 복사해 두고, 복귀 시 엔진의 additive API로 보드를 재구성한다. 엔진 무수정 원칙의 명시적 예외 1건이다.

이걸 만들다 설계 전제 하나가 실측으로 깨졌다. **RTDB push key가 항상 증가한다고 가정했는데, 송신 측이 서버 시계 오프셋을 재보정하면 역행할 수 있다.** 그래서 마지막으로 처리한 key를 단조 증가로만 갱신하는 게이트를 뒀다. 역행한 key가 들어오면 이미 처리한 것으로 취급된다.

## 정리

- 두 모드를 하나의 엔진으로 돌리려면 엔진에 모드를 넣지 말고, 엔진을 감싸는 얇은 층에 오버라이드를 얹는다.
- 봇과 원격을 같은 인터페이스로 만들면 단계 출시가 가능해진다. 그 대신 그 인터페이스 아래로 게임 로직이 새지 않게 지켜야 한다.
- 공정성에 관계된 값은 화면에서 유도하지 않는다. 논리 치수를 고정하고 표시만 늘린다.
- 코드를 옮기는 커밋과 동작을 바꾸는 커밋을 분리한다. 원본 대비 diff 0을 검증 항목으로 둘 수 있으면 그렇게 한다.

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
