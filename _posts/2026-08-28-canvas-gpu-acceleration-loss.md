---
layout: post
title: "뭉치뭉치 - 캔버스 GPU 가속 상실"
description: "전면 광고를 본 뒤 게임이 느려지던 결함의 원인은 광고가 아니라 캔버스 크기 재할당이었다. 같은 값을 대입해도 GPU 가속이 풀리는 동작을 실측으로 확정하고, 계측이 네 번 틀린 과정까지 남겼다."
date: 2026-08-28 01:00:00 +0900
project: moongchi
tags: [performance, canvas, mobile, build-in-public]
---

*🇬🇧 [English version below](#english-version)*

물리 합체 퍼즐 게임을 Canvas 2D 한 장으로 그린다. WebGL은 쓰지 않는다. 이 게임에 며칠 동안 닫히지 않은 성능 결함이 하나 있었다. **전면 광고를 본 뒤 게임으로 돌아오면 화면 전체가 슬로우모션이 된다.** 실기기에서만 재현되고, 앱을 껐다 켜면 사라진다.

원인은 광고가 아니었다. 캔버스 크기 재할당이었다.

## 1. 확정 모델

실측으로 확정한 동작은 이렇다.

> 2D 캔버스는 **생성·리사이즈 직후엔 비가속**이다. DOM에 붙은 채 계속 그려지며 프레임을 넘겨야 GPU 가속으로 승격된다. **전면 웹뷰(광고)를 본 뒤에는 이 승격이 일어나지 않는다.** 이미 승격된 캔버스는 광고를 겪어도 유지된다.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/canvas-gpu-acceleration-loss/canvas-states.ko.svg"><img src="/assets/img/posts/canvas-gpu-acceleration-loss/canvas-states.ko.svg" width="720" height="300" loading="lazy" alt="생성·리사이즈 직후 캔버스는 비가속(픽셀당 비용 3~4배)이다. DOM에 붙은 채 계속 그리면 GPU 가속으로 승격되고, 승격된 캔버스는 광고를 겪어도 유지된다. 전면 광고를 본 뒤에는 승격이 일어나지 않는다. canvas.width를 대입하면(같은 값이어도) 재할당되고, 그리지 않고 방치하면 회수되어 비가속으로 돌아간다."></a></div><figcaption>광고는 원인이 아니었다. 광고 뒤에 새로 만들어지거나 재할당된 캔버스가 승격되지 못한 것이다.</figcaption></figure>

비가속이 되면 픽셀당 비용이 3~4배가 된다. 게임 로직이 느려지는 게 아니라 fill-rate 병목이라 화면 전체가 균일하게 느려진다.

## 2. 실측치

같은 실행 안에서 만든 비가속 눈금 대비 ns/px다.

| 관측 대상 | 값 | 해석 |
|---|---|---|
| 광고 0회, 갓 만든 캔버스 | `newNow` 1.26 (눈금 1.27) | 새 캔버스는 평시에도 비가속 |
| 광고 후 계속 써온 캔버스 | `single` 0.19 (눈금 2.40) | 승격된 건 광고를 견딘다 |
| **같은 엘리먼트를 리사이즈** | 0.14 → 0.97 | **리사이즈가 승격을 푼다** |
| 부팅 때 만들어 상시 그린 2장 | `preAdA+B` 0.65 (눈금 2.40) | 2장 동시에도 가속 유지 |
| 굽고 방치한 캔버스 | `preAdCold` 1.08 vs `preAdA` 0.12 | 회수된다 |
| DOM 미부착 | 0.45~1.42 | 일관성 없음 — 붙여둬야 한다 |

세 번째 행이 결론이다. `canvas.width`에 값을 대입하는 순간 백킹스토어가 재할당되고, **대입한 값이 이전과 같아도 재할당은 일어난다.**

## 3. 수정

| 위치 | 조치 | 이유 |
|---|---|---|
| `sharedCanvas.ts` | 캔버스를 앱 수명 동안 **키별로 재사용**(`single` / `versusMine` / `versusOpp`) | 화면마다 새로 만들면 광고 후 비가속 |
| `sharedCanvas.ts` `bootVersusCanvases()` | 부팅 시 대전 보드 2장을 **최종 크기로** 생성 → 3px 숨은 컨테이너에 부착 → 매 프레임 소량 드로잉 | 광고보다 먼저 승격시키고, 회수되지 않게 유지 |
| `renderer.ts` `initCanvas` | 백킹 크기가 같으면 **재할당하지 않는다** | §2 세 번째 행 |
| 같은 곳 | `ctx.scale(dpr)` → `ctx.setTransform(dpr, …)` | 재할당을 건너뛰면 변환이 초기화되지 않아 scale이 누적된다 |
| `VersusGame.tsx` | 보드 캔버스를 React가 만들지 않고 **프리웜 엘리먼트를 옮겨 꽂는다** | 위 전부를 성립시키는 마지막 조각 |
| `fxRender.ts` `getRenderDpr()` | DPR 상한 2 | 픽셀 수 = 비용. 상한이 없으면 dpr 3 기기에서 2.25배 |
| `offscreenCache.ts` | 단일 슬롯 → **키별** 캐시 | 두 보드가 교대로 다른 키를 요청해 매 프레임 재생성되던 회귀 |

`setTransform` 항목은 수정의 부작용이다. 재할당을 건너뛰면 컨텍스트 상태가 살아남으므로, 매 프레임 `scale`을 곱하던 코드가 그대로면 배율이 누적된다.

## 4. 기각된 해법

전부 실측으로 기각했다. 재시도 금지 목록으로 문서에 남겼다.

| 해법 | 결과 |
|---|---|
| 백킹스토어 재초기화 / 배경 숨김 / 캐시 비우기 | 무반응 |
| 프리웜 v1(미부착·1픽셀) · v2(2px) | 효과 없음 |
| 프리웜 v3(화면 크기·30프레임) | **1회 성공 후 재현 실패** — 굽고 방치해 회수됐다 |
| 페이지 리로드 | 무효. 전부 재생성되어 싱글까지 느려짐 |
| 공유 캔버스에 **새 키** 추가 | 새 키가 대전 첫 진입 때 태어난다 → 광고 뒤면 여전히 비가속 |
| 싱글 캔버스를 대전 보드에 꽂기 | 크기가 달라 재할당된다 (ON 1.19 = OFF 1.16) |
| 렌더 DPR 낮추기 | 효과는 있으나 화질 저하로 채택 불가 |

v3은 검증 번들까지 만들었다가 철회했다. 유저에게 나가지는 않았다. **1회 통과를 해결로 판단한 게 원인이다.** 재현 시험을 붙이기 전에는 성공으로 취급하지 않는 규칙을 이때 넣었다.

## 5. 계측이 네 번 틀렸다

이게 더 옮겨 쓸 만한 부분이다. 원인 규명보다 **재는 방법을 만드는 데** 시간이 더 들었다.

| 시도 | 왜 0이 나오나 |
|---|---|
| `fillRect` 전후 `performance.now()` | 래스터가 JS 호출 밖에서 일어난다 |
| `rAF` 프레임 간격 | 프레임이 늘어나려면 합성기 역압이 필요하다. 화면에 안 뜬 캔버스는 래스터가 생략되고, 헤드리스엔 vsync 역압조차 없다 — 부하를 512배까지 올려도 0 |
| `getImageData(0,0,1,1)`로 강제 flush | **그 1픽셀에 필요한 만큼만 다시 그린다.** 전면 fill 24회가 1픽셀 fill 24회로 줄어든다 |
| 사본에 복사 후 전면 읽기(절대값) | GPU→CPU 회수 비용이 가속 쪽에만 붙어 **신호가 뒤집힌다** (GPU 0.88 vs SW 0.20) |
| ✅ **기울기 `t(4N) − t(N)`** | 고정비용이 상쇄된다. 검증값 GPU 0.01~0.02 vs SW 0.11~0.13 |

여기에 두 가지를 덧붙였다.

- **비가속 눈금 자가 생성** — 매 실행 `willReadFrequently: true` 캔버스를 만들어 "이 기기에서 비가속이면 얼마"를 재고, 모든 판정을 그 대비로 낸다. 기기·해상도가 달라도 절대값 선입견에 기대지 않는다.
- **부하 자동 보정** — 데스크톱은 폰보다 100배 빠르다. 고정 부하로는 양쪽에서 못 쓴다. 눈금 캔버스로 프레임이 목표치를 넘을 때까지 부하를 2배씩 올린다.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/canvas-gpu-acceleration-loss/slope-measure.ko.svg"><img src="/assets/img/posts/canvas-gpu-acceleration-loss/slope-measure.ko.svg" width="720" height="280" loading="lazy" alt="그리기 횟수를 N과 4N으로 바꿔 시간 차를 보면 고정비용이 상쇄된다. 비가속은 기울기가 가파르고 GPU 가속은 완만하다. 검증값은 GPU 0.01~0.02, 비가속 0.11~0.13 ns/px."></a></div><figcaption>절대값이 아니라 기울기를 잰다. 측정 자체의 비용이 빠진다.</figcaption></figure>

## 6. 계측을 프로덕션에 넣는 방법

이 버그는 실기기에서만 재현된다. 즉 **프로덕션 번들에서 수치를 읽을 수 있어야** 조사가 가능하다.

처음엔 `import.meta.env.DEV` 가드를 썼는데, 프로덕션 빌드에서 계측이 통째로 빠져 실기기 수치가 0이었다. Vite `define` 빌드 상수로 바꿨다.

```
$env:PERF="1"; npm run build:ait   # 계측 번들
npm run build                       # 출시 번들
```

기본값 false에서 계측 코드가 전부 트리셰이킹되므로 출시 산출물이 불변이고, **출시 전에 grep으로 계측 문자열 0건을 기계 확인할 수 있다.** 가드를 고르는 기준은 "개발 중에 켜지는가"가 아니라 "필요할 때 프로덕션에서 켤 수 있는가"였다.

## 7. 남은 제약

- `initCanvas`는 싱글·대전·재초기화 공용이다. 여기 손대면 재할당이 되살아난다.
- 대전 캔버스 크기를 바꾸는 변경(레이아웃·DPR·논리 치수)은 프리웜 크기와 정확히 맞아야 한다. 어긋나면 재할당이 일어나 같은 증상이 돌아온다.
- 배경 에셋을 늘린 뒤에는 가속을 재측정해야 한다. 메모리 압박으로 캔버스가 회수되면 §2 다섯 번째 행 상태가 된다.

## 정리

- 성능을 게임 프레임으로 추론하면 안 된다. 엔진 2개·물리·합성이 뒤섞여 신호가 나와도 원인이 분리되지 않는다. 대상 하나만 재는 전용 하네스가 필요하다.
- 대역(surrogate)으로 재면 "대역은 이런데 진짜 그 화면은?"이 끝내 안 갈린다. 실물을 등록해 직접 잰다.
- 기각한 해법은 기각 이유와 함께 남긴다. 없으면 다음 사람이 다시 시도한다. 이번엔 그 다음 사람이 나였다.

---

## English version

The game is a physics merge puzzle rendered on a single Canvas 2D surface — no WebGL. One performance defect stayed open for days: **coming back from a fullscreen ad left the whole screen in slow motion.** It only reproduced on real devices and disappeared after an app restart.

The ad wasn't the cause. Canvas backing-store reallocation was.

### The model

> A 2D canvas is **unaccelerated right after creation or resize.** It gets promoted to GPU compositing only after staying attached to the DOM and being drawn across frames. **After a fullscreen WebView (an ad), that promotion no longer happens.** A canvas already promoted survives the ad.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/canvas-gpu-acceleration-loss/canvas-states.en.svg"><img src="/assets/img/posts/canvas-gpu-acceleration-loss/canvas-states.en.svg" width="720" height="300" loading="lazy" alt="A canvas is unaccelerated right after creation or resize (3–4× cost per pixel). Attached to the DOM and drawn, it is promoted to GPU and survives ads; after a full-screen ad, promotion never happens. Assigning canvas.width (even unchanged) reallocates it, and an idle canvas is reclaimed back to unaccelerated."></a></div><figcaption>The ad wasn’t the cause. Canvases created or reallocated after it never got promoted.</figcaption></figure>

Unaccelerated costs 3–4× per pixel. Nothing in the game loop slows down; it's a fill-rate bottleneck, so the entire screen degrades uniformly.

### Measurements

ns/px, relative to an unaccelerated reference generated in the same run.

| Subject | Value | Reading |
|---|---|---|
| Zero ads, freshly created canvas | `newNow` 1.26 (ref 1.27) | New canvases are unaccelerated even without ads |
| Canvas in continuous use across an ad | `single` 0.19 (ref 2.40) | Promotion survives the ad |
| **Same element, resized** | 0.14 → 0.97 | **Resize revokes promotion** |
| Two canvases warmed since boot | `preAdA+B` 0.65 (ref 2.40) | Two stay accelerated simultaneously |
| Warmed then left idle | `preAdCold` 1.08 vs `preAdA` 0.12 | It gets reclaimed |
| Detached from DOM | 0.45–1.42 | Inconsistent — keep it attached |

Row three is the finding. Assigning to `canvas.width` reallocates the backing store, **and it reallocates even when the assigned value is identical to the current one.**

### The fix

| Where | What | Why |
|---|---|---|
| `sharedCanvas.ts` | Reuse canvases **by key** for the app's lifetime (`single` / `versusMine` / `versusOpp`) | Creating one per screen means unaccelerated after an ad |
| `bootVersusCanvases()` | At boot, create both versus boards **at final size**, attach to a 3px hidden container, draw a little every frame | Promote before any ad, and keep it from being reclaimed |
| `renderer.ts` `initCanvas` | Skip reallocation when backing size is unchanged | Row three above |
| same | `ctx.scale(dpr)` → `ctx.setTransform(dpr, …)` | Skipping reallocation preserves context state, so a per-frame `scale` would compound |
| `VersusGame.tsx` | React doesn't create the board canvas; the pre-warmed element is moved in | The piece that makes the rest work |
| `getRenderDpr()` | Cap DPR at 2 | Pixels are the cost; uncapped, a dpr-3 device pays 2.25× |
| `offscreenCache.ts` | Single slot → **keyed** cache | Two boards alternating keys regenerated the cache every frame |

The `setTransform` line is a consequence of the fix, not an independent change. Once reallocation is skipped, context state survives, and the old per-frame `scale` call starts stacking.

### Rejected approaches

All rejected by measurement, and recorded as a do-not-retry list.

| Approach | Outcome |
|---|---|
| Backing-store reinit / hiding backgrounds / clearing caches | No effect |
| Pre-warm v1 (detached, 1px) and v2 (2px) | No effect |
| Pre-warm v3 (full size, 30 frames) | **Passed once, then failed to reproduce** — warmed and left idle, so it was reclaimed |
| Page reload | Useless; everything is recreated, so single-player slows down too |
| Adding a **new key** to the shared canvas | The new key is born on first versus entry, so post-ad it's still unaccelerated |
| Reusing the single-player canvas for a versus board | Different size → reallocation (ON 1.19 = OFF 1.16) |
| Lowering render DPR | Works, but rejected on image quality |

v3 got as far as a verification bundle before being pulled; it never reached users. The root cause was **treating one passing run as a fix.** The rule that came out of it: nothing counts as solved until it has a reproduction test.

### Four wrong ways to measure this

This is the transferable part. Building a way to measure took longer than finding the cause.

| Attempt | Why it reads zero |
|---|---|
| `performance.now()` around `fillRect` | Rasterization happens outside the JS call |
| `rAF` frame intervals | Frame time only grows under compositor back-pressure. An offscreen canvas skips rasterization entirely, and headless has no vsync back-pressure at all — zero even at 512× load |
| `getImageData(0,0,1,1)` to force a flush | **It redraws only what that one pixel needs.** 24 full-surface fills collapse into 24 single-pixel fills |
| Copy to a scratch canvas and read it whole (absolute) | GPU→CPU readback cost lands only on the accelerated side, so **the signal inverts** (GPU 0.88 vs SW 0.20) |
| ✅ **Slope, `t(4N) − t(N)`** | Fixed costs cancel. Validation: GPU 0.01–0.02 vs SW 0.11–0.13 |

Two supporting pieces:

- **Self-generated unaccelerated reference.** Every run creates a `willReadFrequently: true` canvas to answer "what does unaccelerated cost *on this device*," and every verdict is expressed relative to that. No absolute thresholds baked in.
- **Auto-calibrated load.** A desktop is ~100× faster than a phone, so a fixed workload is useless on one end or the other. The reference canvas doubles the load until frame time clears a target.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/canvas-gpu-acceleration-loss/slope-measure.en.svg"><img src="/assets/img/posts/canvas-gpu-acceleration-loss/slope-measure.en.svg" width="720" height="280" loading="lazy" alt="Timing N draws and 4N draws and taking the difference cancels fixed overhead. The unaccelerated line is steep, the GPU line shallow. Verified: GPU 0.01–0.02, unaccelerated 0.11–0.13 ns/px."></a></div><figcaption>Measure the slope, not the absolute value, and the cost of measuring drops out.</figcaption></figure>

### Instrumentation that survives a production build

The bug reproduced only on real devices, which means investigation required reading numbers **from a production bundle.**

The first attempt used an `import.meta.env.DEV` guard. Production builds dropped the instrumentation entirely, so on-device readings were zero. It moved to a Vite `define` build constant instead.

```
$env:PERF="1"; npm run build:ait   # instrumented bundle
npm run build                       # release bundle
```

Defaulting to false lets the whole instrumentation path tree-shake out, so release artifacts are unchanged and **a pre-release grep can mechanically confirm zero instrumentation strings.** The criterion for choosing a guard wasn't "is it on during development" but "can I turn it on in production when I need it."

### Constraints that remain

- `initCanvas` is shared by single-player, versus, and reinit paths. Touching it can bring reallocation back.
- Any change to versus canvas size — layout, DPR, logical dimensions — must match the pre-warm size exactly. A mismatch reallocates and the symptom returns.
- Adding background assets requires re-measuring acceleration. Under memory pressure the canvas gets reclaimed, which lands on row five of the table.

### Takeaways

- Don't infer rendering performance from game frame rate. Two engines, physics, and compositing are mixed in; even a real signal doesn't isolate a cause. Build a harness that measures one subject.
- Measuring a surrogate never closes the question "the surrogate reads this, but what about the actual surface?" Register the real thing and measure it directly.
- Record rejected approaches with the reason they were rejected. Without that, the next person retries them. Here the next person was me.
