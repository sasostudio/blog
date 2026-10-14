---
layout: post
title: "밤샘팟 - 그림에 없는 관절"
description: "밤샘팟의 베개 캐릭터를 리깅 도구 없이 직접 짠 메시 변형으로 움직인 기록. 35벌이 될 뻔한 그림을 7 + 5로 줄이고, 원화에 없는 관절 때문에 팔을 여섯 번 고쳤다."
date: 2026-08-27 11:00:00 +0900
project: night-pod
tags: [webgl, animation, mobile, build-in-public]
---

*🇬🇧 [English version below](#english-version)*

앱인토스 8월 챌린지에 미니앱을 하나 냈다. 밤에 안 자는 사람이 열면 지금 같이 깨어 있는 사람 수와 그 이유 순위를 보여주는 앱이고, 화면 한가운데 베개 캐릭터가 서 있다. 캐릭터가 주인공이라 "이걸 어떻게 움직이나"가 첫 관문이었다.

<figure class="shot"><img src="/assets/img/posts/mesh-deformation-from-scratch/night-pod-home.jpg" width="440" height="725" loading="lazy" alt="밤샘팟 홈 화면 — 잠옷을 입고 안대를 쓴 베개 캐릭터가 방 한가운데 서 있고, 아래에 오늘 밤샘 인원과 이유 쪽지들이 있다"><figcaption>밤샘팟 홈 화면. 이 베개 캐릭터를 코드로 움직이는 게 첫 관문이었다.</figcaption></figure>

## 리깅 도구를 버렸다

처음엔 업계에서 쓰는 2D 리깅 도구를 채택했다. 웹 연동 사전조사까지 마치고 나서야 문제가 드러났다. 그 도구는 사람이 마우스로 정점을 찍고 변형을 잡는 물건이다. 계획서에는 "도구로 리깅한다"고 적혀 있었는데, 그 일을 할 사람이 비어 있었다.

그래서 원리만 직접 구현했다. 그림을 격자로 나누고 정점 위치마다 다른 각도를 주면 관절이 꺾이지 않고 활처럼 휜다. 변형 수학을 화면에서 떼어내 순수 함수로 만들고 검사부터 붙였더니 그 자리에서 결함이 하나 나왔다. 휘는 기준선을 조각 가장자리에 뒀더니 길이가 12% 줄어들었다. 기준선을 중앙으로 옮기니 보존됐다.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/mesh-deformation-from-scratch/bend-baseline.ko.svg"><img src="/assets/img/posts/mesh-deformation-from-scratch/bend-baseline.ko.svg" width="720" height="300" loading="lazy" alt="같은 띠를 휘었을 때, 기준선을 조각 가장자리에 두면 길이가 12% 줄고 가운데에 두면 길이가 보존된다."></a></div><figcaption>휘는 기준선 위치 하나가 길이 12%를 갉아먹고 있었다.</figcaption></figure>

성능은 덤이었다. 안 쓰는 기능이 없으니 원래 도구보다 30~50배 빨랐다.

## 35벌을 7 + 5로

캐릭터는 사용자가 고른 이유대로 행동한다. 당시 이유가 7종이고(나중에 8종이 됐다) 표정이 5종이니, 조합을 그림으로 만들면 35벌이다. 그렇게 갈 뻔했다.

축을 나눠서 막았다. 포즈와 소품은 이유가 정하고, 표정은 시각이 정한다. 두 축이 독립이면 자산은 7 + 5로 끝난다. 이미지 생성에 하루 횟수 제한이 걸려 있어서 이 판단이 그대로 일정이 됐다.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/mesh-deformation-from-scratch/asset-axes.ko.svg"><img src="/assets/img/posts/mesh-deformation-from-scratch/asset-axes.ko.svg" width="720" height="250" loading="lazy" alt="이유 7종과 표정 5종을 조합마다 그리면 35벌이지만, 이유가 포즈·소품을, 시각이 표정을 정하도록 축을 나누면 7 + 5 = 12벌이다."></a></div><figcaption>두 축이 독립이면 자산은 곱이 아니라 합이 된다.</figcaption></figure>

덤으로 표현 하나가 딸려 왔다. 동작이 한 바퀴 도는 동안 앞쪽은 이유 표정, 뒤쪽은 시각 표정이 나오고 그 비율이 곧 그 시간대의 활기다. 초저녁엔 내내 이유 표정이지만 새벽 세 시엔 대부분 감긴 눈이 이긴다. 공부하다 자꾸 눈이 감기는 상태가 숫자 하나로 표현된다.

## 팔을 여섯 번 고쳤다

하루를 통째로 팔에 썼다.

| 차수 | 무엇을 고쳤나 | 결과 |
|---|---|---|
| 1 | 팔 영역을 사각형이 아니라 그림에서 잰 경계로 | 몸통이 옆으로 늘어나던 것 해결. 여전히 어색 |
| 2 | 회전축을 소매 뿌리에서 몸통 안쪽으로 | 소매가 통째로 들림. 여전히 어색 |
| 3 | 위치를 섞던 것을 각도를 섞는 것으로 | 팔 길이 보존. 여전히 어색 |
| 4 | 변형 순서를 부위 먼저·전신 나중으로 | 팔 뿌리 35px 어긋남 해결. 여전히 어색 |
| 5 | 팔을 그림에서 떼어 몸통 뒤로 | 이음매가 사라짐. 대신 뻣뻣해짐 |
| 6 | 떼어낸 조각에 다시 격자를 줌 | 뻣뻣함 해소 |

앞의 네 번은 전부 틀린 층위를 고친 것이었다. 매번 뭔가 나아졌고 매번 관리자는 여전히 어색하다고 했다.

다섯 번째에 인정했다. 원화가 팔 내린 자세로 그려져 있어서, 팔을 올리면 옷 어깨 모양이 그림에 없다. 없는 정보는 어떤 계산으로도 만들 수 없다. 버그가 아니라 방식의 한계였다. 관리자가 "너 솔직히 못 고치지?"라고 물었을 때 그렇다고 답한 게 전환점이었다. 그때서야 "어떻게 더 잘 휠까"에서 "휘지 말자"로 질문이 바뀌었다.

팔을 그림에서 떼어 몸통 뒤에 깔면 뿌리가 가려져 이음매가 존재할 수 없다. 회전 상한이 세 배가 됐다.

여기에 반전이 있다. 낱개 팔 자산은 이미 있었다. 뿌리가 둥근 캡으로, 애초에 가려서 붙이라고 그려진 모양이었다. 몇 시간 전에 내가 "조각내 붙이는 방식은 연결부가 드러나서 폐기"라고 문서에 적어둔 것이 이 선택지를 막고 있었다. 그때 실패한 건 캐릭터 전체를 조각내 같은 평면에 나란히 붙인 것이고, 지금은 팔만 떼어 뒤로 숨긴 것이다. 같은 단어로 묶어둔 바람에 다른 방법이 안 보였다.

여섯 번째는 그 대가였다. 떼어낸 팔은 네 점짜리 판때기라 통째로만 돌아서 목각인형처럼 보였다. 분리와 휨을 둘 중 하나를 고르는 문제로 본 게 잘못이었다. 분리는 자유도를 주고 휨은 자연스러움을 준다. 떼어낸 조각에 다시 격자를 줬다.

여섯 번으로 끝나지도 않았다. 몇 시간 뒤 소품을 붙이자 몸통 뒤에 깔린 팔을 소품이 덮어서, 아무도 물건을 쥐고 있지 않은 그림이 됐다. 팔을 가장 앞으로 옮겼다. 감추려던 어깨 이음매가 드러나지만 뿌리가 둥근 캡이라 옷의 솔기처럼 읽힌다. 최종본은 그 상태다.

## 스프링이 먹어버린 것

움직임은 사인파 대신 스프링으로 굴린다. 코드는 가야 할 자세만 정하고 사이는 물리가 메운다. 계수를 강성·감쇠 그대로 두지 않고 감쇠비로 잡은 게 핵심이다. 값 하나로 "지나치느냐"가 정해지니 감으로 만지지 않고 시험으로 확인할 수 있다. 팔은 37% 지나치고 몸통은 거의 안 지나친다. 이 대비가 살아 있는 느낌을 만든다.

대가도 있었다. 손을 바쁘게 움직이려고 잔진동을 넣었는데 화면에 아무것도 안 나왔다. 팔 스프링의 고유진동수가 초당 한 번꼴인데 초당 예닐곱 번 떠는 신호를 넣었으니 남을 리가 없다. 스프링이 다 먹은 것이다. 물리를 넣어 자연스러움을 얻은 대가로 빠른 표현을 잃은 셈이라, 잔진동만 스프링을 건너뛰어 팔 각도에 직접 더하는 길을 따로 냈다.

## 배운 것

- 변형 수학을 화면에서 떼어내면 눈으로 못 잡는 결함이 검사로 잡힌다. 길이 12% 손실도, 60fps와 30fps에서 궤적이 1.6% 갈리던 것도 그렇게 나왔다.
- "구조적 한계"라고 결론 내리기 전에 한 번 더 재라. 팔 영역을 눈대중 사각형으로 잡던 동안은 회전 상한을 낮춰 증상만 눌렀는데, 경계를 그림에서 실제로 재고 나니 그 제약의 원인 자체가 없어졌다.
- 교훈을 뭉뚱그려 적으면 다음 사람이 정답을 못 본다. 이번엔 그 다음 사람이 몇 시간 뒤의 나였다.

---

## English version

I shipped a mini-app for an August build challenge on the Toss platform. You open it at 2 a.m., it tells you how many other people are awake right now and why, and a pillow character stands in the middle of the screen. The character is the whole app, so the first question was how to animate it.

<figure class="shot"><img src="/assets/img/posts/mesh-deformation-from-scratch/night-pod-home.jpg" width="440" height="725" loading="lazy" alt="Night Pod home screen — a pillow character in pajamas and a sleep mask stands mid-room, with tonight’s count and reason notes below"><figcaption>Night Pod’s home screen. Making this pillow character move in code was the first hurdle.</figcaption></figure>

### Dropping the rigging tool

I'd picked a standard 2D rigging tool and finished the research on wiring it into the web before the real problem surfaced: a human sits down with a mouse and places the deformation by hand. The plan said "rig it with the tool." It never said who.

So I implemented the underlying idea instead. Split the sprite into a grid, apply a different angle per vertex, and the joint bends like a bow rather than creasing. I pulled the deformation math out of the render path into pure functions and wrote the tests first — which immediately surfaced a defect. With the bend axis on the edge of the piece, the arm lost 12% of its length. Moving the axis to the center preserved it.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/mesh-deformation-from-scratch/bend-baseline.en.svg"><img src="/assets/img/posts/mesh-deformation-from-scratch/bend-baseline.en.svg" width="720" height="300" loading="lazy" alt="Bending the same strip: with the baseline on the piece’s edge the length shrinks by 12%; with it in the middle the length is kept."></a></div><figcaption>Where the bend baseline sat was quietly eating 12% of the length.</figcaption></figure>

The performance was a bonus: 30–50x faster than the tool, because none of the features I wasn't using were there.

### 35 combinations down to 7 + 5

The character acts out whichever reason you picked. Seven reasons at the time — eight in the shipped version — and five expressions. Drawn as combinations, that's 35 sprite sets.

Splitting the axes killed it. Pose and props come from the reason; expression comes from the clock. Independent axes mean 7 + 5 assets instead of 7 × 5. Image generation had a daily quota, so this decision *was* the schedule.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/mesh-deformation-from-scratch/asset-axes.en.svg"><img src="/assets/img/posts/mesh-deformation-from-scratch/asset-axes.en.svg" width="720" height="250" loading="lazy" alt="Drawing every combination of 7 reasons and 5 expressions takes 35 sets; letting the reason set pose and prop and the time set the expression takes 7 + 5 = 12."></a></div><figcaption>Independent axes turn a product into a sum.</figcaption></figure>

It bought an expression I hadn't planned for. Across one animation cycle the first half shows the reason's expression and the second half shows the hour's, and the ratio between them is how awake that hour is. At 8 p.m. it's all reason. At 3 a.m. the closing eyes win most of the cycle. "Studying, but my eyes keep shutting" turns out to be a single number.

### Six passes at one arm

An entire day.

| Pass | What I changed | Outcome |
|---|---|---|
| 1 | Arm region: bounding box → boundary measured from the sprite | Torso stopped stretching sideways. Still wrong |
| 2 | Pivot: sleeve root → inside the torso | Whole sleeve lifts now. Still wrong |
| 3 | Blend positions → blend angles | Arm length preserved. Still wrong |
| 4 | Reorder: limb deform first, whole-body deform second | Fixed a 35px pivot drift. Still wrong |
| 5 | Cut the arm out, render it behind the torso | Seam gone. Now it's stiff |
| 6 | Give the cut-out piece its own grid | Stiffness resolved |

The first four all fixed the wrong layer. Each one improved something, and each time the answer came back: still looks off.

On the fifth I admitted the real constraint. The source art is drawn arms-down, so when the arm lifts, the shoulder of the shirt *doesn't exist in the image*. No amount of math invents information that was never drawn. It wasn't a bug, it was the limit of the approach. The turning point was being asked, point blank, "you can't actually fix this, can you?" — and saying no. Only then did the question change from "how do I bend this better" to "what if I don't bend it."

Cut the arm out, render it behind the torso, and the root is occluded — there is no seam to hide. Rotation range tripled.

The twist: the separate arm asset already existed, with a rounded cap at the root, drawn from the start to be tucked under something. What blocked me was a note I had written a few hours earlier: *cutting the character into parts is rejected, the joins show*. That failure was slicing the whole character into coplanar pieces. This was cutting one limb and hiding it behind. Filing both under the same word made the second one invisible to me.

The sixth pass was the bill for the fifth. A cut-out arm is a four-vertex quad, so it only rotates as a rigid slab — a wooden puppet. Treating separation and bending as an either/or was the mistake. Separation buys range; bending buys naturalness. The cut-out piece got its own grid.

Six wasn't the end either. Hours later, once props went in, the prop covered an arm that was sitting behind the torso — nobody appeared to be holding anything. The arm moved to the very front. The shoulder join I'd been hiding is now visible, but the rounded cap at the root reads as a sleeve seam. That's what shipped.

### What the spring ate

Motion runs on springs rather than hand-authored sine waves. The code only sets a target pose; physics fills in the space between. The key decision was parameterizing by damping ratio instead of raw stiffness and damping — one number decides whether it overshoots, which makes it testable instead of a feel thing. The arm overshoots 37%, the torso barely at all, and that contrast is most of what reads as "alive."

It came with a cost. I added a fast jitter so the hands would look busy, and nothing showed up on screen. The arm spring's natural frequency is about 1 Hz and I was feeding it a 6–7 Hz signal — the spring absorbed all of it. Naturalness from physics, paid for in fast motion. The jitter now bypasses the spring and adds straight to the arm angle.

### Takeaways

- Pull deformation math out of the render path and tests catch what eyes can't. Both the 12% length loss and a 1.6% trajectory divergence between 60fps and 30fps showed up that way.
- Measure once more before concluding "structural limit." While the arm region was a hand-guessed bounding box, I just lowered the rotation ceiling to suppress the symptom; once the boundary was measured off the sprite itself, the cause was gone.
- A lesson written too broadly hides the answer from whoever reads it next. Here, that was me, a few hours later.
