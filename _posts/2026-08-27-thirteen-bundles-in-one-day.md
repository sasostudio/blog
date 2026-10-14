---
layout: post
title: "밤샘팟 - 실기기 13차"
description: "밤샘팟 출시 직전 하루 동안 번들을 13번 다시 만든 실기기 대응 기록. 틀린 성능 계측, 평균이 못 보는 반복 자국, 추정값으로 내린 「불가능」 판정, 발열을 만든 렌더 루프를 다룬다."
date: 2026-08-27 11:30:00 +0900
project: night-pod
tags: [mobile, testing, performance, build-in-public]
---

*🇬🇧 [English version below](#english-version)*

[지난 글](/2026/08/27/mesh-deformation-from-scratch/)에서 캐릭터를 코드로 움직인 이야기를 썼다. 이번은 그 캐릭터가 실기기에서 깨진 이야기다.

이 프로젝트에는 브라우저 자동화 도구가 없다. 테스트는 node 환경이라 레이아웃을 못 본다. 검사 961건이 통과해도 화면이 어떻게 생겼는지에 대해서는 아무 말도 하지 않는다. 그래서 판정을 숫자로 하는 도구를 여러 개 만들었고, 그 도구들이 먼저 틀렸다.

## 계측이 먼저 틀렸다

리깅에 사흘을 쓰기 전에 "이 방식이 폰에서 버티나"를 확인하려고 실기기 성능 게이트를 앞에 뒀다. 관리자가 열어보니 실패가 떴다. 확인해 보니 세 가지가 겹친 계측 결함이었고 대상은 멀쩡했다.

| 증상 | 실제 원인 |
|---|---|
| 로딩 시간이 13초 → 27초 → 33초로 계속 늘어남 | 재측정할 때 시작 시각을 다시 안 잡아서, 버튼 누른 순간까지 흐른 시간이 통째로 로딩으로 찍힘 |
| 해상도 버튼을 눌러도 아무 일이 없음 | 기기 배율과 비교해 더 작은 값을 쓰게 돼 있어 배율 100% 모니터에서는 무효 |
| 최저 성능이 실제 179인데 1로 집계 | 첫 그림 직후 리소스를 올리느라 튀는 몇 프레임이 최저값으로 잡힘 |

지표 자체도 바꿨다. 초당 프레임 수는 주사율에 막혀서(폰 60, 데스크톱 180) 여유가 얼마나 남았는지를 못 보여준다. 한 프레임에 실제로 쓴 작업 시간으로 바꾸니 60Hz 예산 16.7ms 중 0.7ms를 쓴다는 값이 나왔고, 그게 "폰이 훨씬 느려도 들어온다"는 판단의 근거가 됐다. 로딩도 부팅·자산·첫 그림 세 구간으로 쪼갰다. 4.27초 중 3.54초가 개발 서버 몫이었다. 안 쪼갰으면 엉뚱한 데를 최적화했을 것이다.

## 평균이 못 보는 것

배경 그림은 요즘 폰 비율에 비해 세로가 모자라서, 맨 아래 띠를 잘라 반복해 채운다. 이음매가 보이면 안 되니 "띠 위아래 색 차이"를 검사로 걸어뒀고, 새 배경이 3.3으로 여유롭게 통과했다(기준 10).

미리보기를 열어 보니 화면 오른쪽에 작은 자국이 세로로 줄지어 찍혀 있었다. 띠 맨 위 세 줄에 책 더미 아래끝이 걸쳐 들어왔고, 그게 화면 아래로 반복된 것이었다. 기존 검사가 못 잡은 이유가 핵심이다. 그 검사는 열 평균을 보는데, 가로로 좁고 어두운 자국은 48줄 평균에 묻혀 편차가 4.2밖에 안 된다. 평균은 "작은 것이 여러 번 반복된다"를 못 본다. 픽셀 단위 이물 탐지로 바꾸니 같은 띠에서 39개가 잡혔다.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/thirteen-bundles-in-one-day/mean-blind-spot.ko.svg"><img src="/assets/img/posts/thirteen-bundles-in-one-day/mean-blind-spot.ko.svg" width="720" height="260" loading="lazy" alt="반복 타일로 쓰는 48줄 띠에 작은 자국이 흩어져 있다. 열 평균 검사는 편차 4.2로 기준 10 아래라 통과시키고, 픽셀 단위 이물 탐지는 39개를 잡아 실패시킨다."></a></div><figcaption>평균은 작은 것이 여러 번 반복되는 걸 못 본다.</figcaption></figure>

같은 계열의 실수가 색에도 있었다. 이유 8종에 색을 줬는데 헷갈린다는 지적을 받았다. 기존 검사는 "색이 서로 다르다"만 봐서 값이 1만 달라도 통과한다. 화면에서는 같은 색인데 테스트가 안심시키고 있었던 것이다. 사람 눈 가중치를 준 거리로 다시 재니 최소 67.6이었고, 색상환에서 50도씩 벌려 재배치해 131.6이 됐다.

수치가 통과했다고 끝난 게 아니라, 통과한 수치가 무엇을 안 보는지를 물어야 했다.

## 하루에 번들 13차

출시 직전 하루는 통째로 실기기 대응이었다. 그날 번들을 열세 번 다시 만들었다. 하단 잘림, 상단 여백, 내비바, 팝업 투명, 색 구분, 발열.

가장 뼈아픈 건 화면 하단이 잘리는 문제였다. 처음엔 "방 배경이 폭에 비례해 세로를 다 먹는다"로 진단하고 배경 크기를 줄이는 수정에 들어갔다. 관리자가 멈춰 세우고 한마디를 보탰다. "같은 기기에서 다른 앱 두 개는 멀쩡했어."

그 한마디가 진단을 뒤집었다. 같은 기기에서 다른 앱이 멀쩡하면 기기 문제가 아니다. 대조하니 차이는 루트 레이아웃 한 줄이었다. 다른 앱은 최소 높이라 넘치면 페이지가 길어지고, 이 앱만 높이를 못 박고 넘치는 것을 잘라내고 있었다.

그 한 줄이 전날 내가 넣은 수정이었다. "카드를 펼치면 목록이 화면 밖으로 잘려 나간다"를 고치려고 높이를 고정했고, 그게 다른 조건에서 하단을 먹는 새 문제를 만들었다. 당시 설계 기준 폰에서 여유가 7px이었는데 통과했으니 괜찮다고 봤다. 여유 7px은 통과가 아니라 경고였다.

## 141px

같은 날 오후, 세 번째 지적("책상 앞면이 안 보인다")에 대해서는 못 고친다는 결론을 계산표와 함께 냈다. 화면 세로 731 중 방 배경이 610을 먹고 하단 UI가 238을 필요로 하니 141px이 물리적으로 모자라다고.

틀렸다. 관리자가 화면을 보고 한마디로 뒤집었다. "아니 지금 공간이 남잖아."

진짜 원인은 쪽지 줄의 flex 설정이었다. 남는 높이를 위아래 줄이 균등 분배받게 해뒀더니, 쪽지 두 장짜리 줄이 자기 내용보다 훨씬 큰 높이를 빈 여백으로 차지하고 있었다. 공간이 없던 게 아니라 낭비되고 있었다.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/thirteen-bundles-in-one-day/flex-waste.ko.svg"><img src="/assets/img/posts/thirteen-bundles-in-one-day/flex-waste.ko.svg" width="720" height="300" loading="lazy" alt="원래 레이아웃에서는 남는 높이를 균등 분배해 쪽지 두 장짜리 줄이 빈 여백을 차지했다. 내용 크기만 차지하게 바꾸자 여백이 사라지고 책상 앞면이 보였다."></a></div><figcaption>공간이 모자랐던 게 아니라 낭비되고 있었다.</figcaption></figure>

오판의 구조가 뼈아프다. 내 계산은 화면 높이도 요소 높이도 전부 추정값이었는데, 그걸 "물리적으로 불가능"이라는 단정으로 포장했다. 추정 하나가 틀리자 결론이 통째로 뒤집혔다. 화면을 볼 수 없는 쪽이 예산 계산으로 불가능을 선언하면 안 된다. 실기기를 보는 사람의 관찰이 언제나 더 강하다.

폐기된 계산표는 지우지 않고 문서에 남기면서 "다음 세션도 이걸 근거로 배경 축소를 꺼내지 말 것"이라고 못을 박았다.

## 폰이 뜨겁다

발열 지적을 받고 렌더 루프를 열었다. 매 프레임 캔버스의 `clientWidth`/`clientHeight`를 읽고 있었다. 이 속성을 읽으면 브라우저가 그 자리에서 레이아웃을 강제로 계산한다. 초당 60번 레이아웃이 돈 셈이다. 더 허탈한 건 이 앱에 이미 크기를 추적하는 옵저버가 있었다는 점이다. 루프가 그걸 안 쓰고 매번 DOM에 물어봤다.

두 번째는 매 프레임 배열 할당이었다. 드로우 함수가 호출마다 배열을 복사·정렬해 새로 만드는데, 그 함수가 한 프레임에 대여섯 번 불린다. 초당 300개가 넘는 임시 배열이 GC를 계속 깨웠다. 버퍼 재사용으로 바꿨다.

여기서 일부러 한 번에 다 바꾸지 않았다. 발열의 본질은 픽셀 처리량(초당 1.4억 픽셀)이고 그걸 줄이는 수단은 프레임 제한이나 해상도 축소인데, 둘 다 체감이 바뀐다. 무해한 것만 먼저 넣고 효과를 재기로 했다. 한꺼번에 손대면 무엇이 효과였는지 알 수 없다.

## 배운 것

- 판정 도구를 먼저 검증하지 않으면 판정이 뒤집힌다. 성능 게이트에서 한 번, 반복 타일에서 한 번, 색 구분에서 한 번 겪었다.
- 통과한 수치가 무엇을 안 보는지 물어야 한다. 평균은 반복되는 작은 것을 못 보고, "다르다"는 "얼마나 다른가"를 못 본다.
- 검증할 수 없는 것을 통과로 포장하지 않는다. 레이아웃 수정 하나는 "고쳤다"가 아니라 "논리적 근거뿐이고 실기기 확인이 남았다"로 보고했다. 도구가 없어 못 본 것을 못 봤다고 말하는 게 그 상황에서 할 수 있는 최선이다.

---

## English version

[The previous post](/2026/08/27/mesh-deformation-from-scratch/) covered animating the character in code. This one is about that character breaking on real devices.

There's no browser automation in this project. Tests run in node, so they can't see layout. 961 passing checks say nothing about what the screen looks like. So I built several tools that judge things numerically — and those tools were wrong first.

### The measurement was wrong before the thing being measured

Before spending three days on rigging, I put a real-device performance gate in front of it: does this approach survive on a phone at all? It came back FAILED. Three separate instrumentation defects, stacked. The subject was fine.

| Symptom | Actual cause |
|---|---|
| Load time climbing 13s → 27s → 33s | The start timestamp wasn't reset between runs, so all the idle time before the button press counted as loading |
| Resolution buttons did nothing | The control took the smaller of requested and device scale, so on a 100%-scale monitor every button was a no-op |
| Worst-case frame reported as 1 when it was really 179 | The few spiky frames right after first paint, while resources upload, were sampled as the minimum |

I changed the metric too. Frames per second is capped by refresh rate (60 on the phone, 180 on the desktop), so it can't show how much headroom is left. Switching to actual work time per frame gave 0.7ms against a 16.7ms budget at 60Hz — and *that* is what justified "a much slower phone still fits." I split load time into boot, assets, and first paint as well: 3.54 of the 4.27 seconds belonged to the dev server. Without the split I'd have optimized the wrong thing.

### What an average can't see

The room background is too short for modern phone aspect ratios, so the bottom strip is cut and tiled downward. Seams would be obvious, so there's a check on the color difference between the strip's top and bottom edges. The new background passed comfortably at 3.3 (threshold 10).

Then I opened the preview and found small marks repeating in a vertical column down the right side. The bottom edge of a book stack intruded into the top three rows of the strip, and that got tiled all the way down. Why the existing check missed it is the interesting part: it compares column *averages*, and a narrow dark mark disappears into a 48-row average — deviation 4.2. An average cannot see "something small, repeated many times." Switching to per-pixel outlier detection found 39 in the same strip.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/thirteen-bundles-in-one-day/mean-blind-spot.en.svg"><img src="/assets/img/posts/thirteen-bundles-in-one-day/mean-blind-spot.en.svg" width="720" height="260" loading="lazy" alt="A 48-row tiled strip with small marks scattered across it. The column-mean check passes it with a deviation of 4.2 under the limit of 10; a per-pixel foreign-mark check finds 39 and fails it."></a></div><figcaption>An average can’t see something small repeated many times.</figcaption></figure>

The same class of mistake showed up in color. Eight reasons, eight colors, and the report was that they were hard to tell apart. The existing test only asserted the colors *differed*, so a difference of 1 passed. Identical on screen; reassuring in CI. Re-measuring with perceptually weighted distance gave a minimum of 67.6; redistributing them 50° apart on the wheel brought it to 131.6.

Passing a threshold isn't the end. The question is what the passing number isn't looking at.

### Thirteen bundles in one day

The day before launch was entirely real-device work. Thirteen rebuilds: bottom clipping, top inset, nav bar, transparent popup, color separation, thermals.

The worst was the bottom of the screen getting cut off. My first diagnosis was that the room background scales with width and eats the full height, and I started shrinking the background. Then: "the other two apps on the same device were fine."

That flipped the diagnosis. If other apps are fine on the same device, it isn't the device. The difference turned out to be one line in the root layout — the other apps used a minimum height so overflow lengthens the page, while this one pinned the height and clipped the overflow.

That line was my own fix from the day before. I'd pinned the height to stop an expanded card's list from spilling off screen, and it created a new failure under different conditions. On the reference phone at the time the margin was 7px, and I called that a pass. 7px of margin isn't a pass, it's a warning.

### 141px

That afternoon, for a third complaint ("the front of the desk isn't visible"), I filed a "can't be fixed" with a budget table: of 731 vertical pixels, the room background takes 610 and the bottom UI needs 238, so we're 141px short — physically impossible.

Wrong. One look at the actual screen: "there's clearly space right now."

The real cause was the flex configuration on the sticky-note row. Leftover height was distributed evenly between rows, so a row holding two small notes claimed far more height than its contents and sat there as empty space. The space wasn't missing. It was being wasted.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/thirteen-bundles-in-one-day/flex-waste.en.svg"><img src="/assets/img/posts/thirteen-bundles-in-one-day/flex-waste.en.svg" width="720" height="300" loading="lazy" alt="In the original layout the leftover height was split evenly, so the two-note row held empty padding. Sizing it to its content removed the padding and the desk front came into view."></a></div><figcaption>Space wasn’t short — it was being wasted.</figcaption></figure>

The shape of the error is what stings. Every number in my calculation — screen height, element heights — was an estimate, and I packaged the result as a categorical impossibility. One bad estimate and the whole conclusion inverted. **Someone who can't see the screen shouldn't declare impossibility from a budget table.** The observation from the person holding the device always outranks it.

I kept the discredited table in the doc rather than deleting it, with a note: don't reopen "shrink the background" on the strength of this.

### The phone is hot

The render loop was reading the canvas's `clientWidth`/`clientHeight` every frame. Reading those properties forces the browser to compute layout on the spot — sixty forced layouts per second. The deflating part: the app already had a resize observer tracking exactly that. The loop just didn't use it and asked the DOM instead.

Second waste: per-frame allocation. The draw function copied and sorted an array on every call, and it's called five or six times per frame. Over 300 temporary arrays per second, keeping GC awake. Switched to a reused buffer.

I deliberately did *not* change everything at once. The real driver of the heat is pixel throughput (about 140 million pixels/sec), and the levers for that are frame limiting or resolution reduction — both of which change how the app feels. Land the harmless ones first, then measure. Change everything together and you learn nothing about which change mattered.

### Takeaways

- Validate the judging tool before trusting the judgment. Three times: the performance gate, the tiling check, the color check.
- Ask what a passing number ignores. Averages hide small repeated defects; "different" hides "how different."
- Don't dress up unverifiable work as verified. One layout fix was reported as "the only support for this is reasoning; real-device confirmation is still outstanding," not as "fixed." Saying you couldn't see it is the best available move when you have no tool that can.
