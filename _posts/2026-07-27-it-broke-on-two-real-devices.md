---
layout: post
title: "실기기 검증 포스트모템"
description: "로컬 탭 두 개에서는 정상이던 1대1 대전의 실기기 두 대 검증 기록. 화면 크기 유불리, 연출 불일치, 끊김 판정, 시스템 글꼴 배율, 버튼 겹침 결함 5건 수정과, 결함이 아니었던 1건의 미수정 판단까지."
date: 2026-07-27 11:00:00 +0900
project: moongchi
tags: [game-dev, multiplayer, mobile, build-in-public]
---

*🇬🇧 [English version below](#english-version)*

[지난 글](/2026/07/27/you-cant-report-your-own-win/)의 주제는 1대1 대전의 판정과 정산을 서버로 옮긴 과정. 그 상태로 로컬에서 브라우저 탭 두 개를 띄워 대전시키면 정상 동작. 판정 일치, 매칭 성공, 정산 완료.

실기기 두 대로 검증을 돌리자 하나씩 붕괴. 결함 5건 수정, 1건은 결함이 아님을 확인한 기록.

## 큰 화면의 우위

물리 세계가 화면 픽셀 위에 그대로 구축된 구조. 캔버스 크기를 재서 그 자리에 벽을 세우고, 떨어지는 오브젝트는 고정 픽셀 크기. 화면이 크면 벽 사이 간격은 넓어지는데 오브젝트 크기는 그대로. 데스크톱에서는 같은 보드에 세 배 넘게 수용.

합체 퍼즐에서 더 많이 담긴다는 건 곧 유리하다는 의미. 결과는 화면 큰 사람의 승리.

임시 대응으로 보드 크기에 상한을 걸어 편차를 폰 범위로 축소. 근본 해결은 좌표계 분리. 물리는 고정된 논리 치수 위에서 돌고, 화면은 그 결과를 확대 표시만 담당. 창 크기를 바꿔도 물리적 변화는 0.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/it-broke-on-two-real-devices/coordinates.ko.svg"><img src="/assets/img/posts/it-broke-on-two-real-devices/coordinates.ko.svg" width="720" height="320" loading="lazy" alt="전: 물리가 화면 픽셀 위에 있어 같은 크기 공이 큰 데스크톱 화면에 폰보다 몇 배 많이 수용. 후: 고정 치수의 논리 보드에서 물리 실행, 폰과 데스크톱은 같은 판을 확대 표시만 담당."></a></div><figcaption>물리는 고정된 논리 보드에서 실행, 화면은 확대만 담당. 창 크기를 바꿔도 물리적 변화는 0.</figcaption></figure>

엔진을 싱글 모드와 공유하는 구조라 함부로 수정 불가. 옵션을 하나 추가하되, 지정하지 않으면 기존 경로가 그대로 도는 방식으로 도입. 터치 좌표만 화면 비율에 맞춰 보정하자 나머지는 그대로 정합.

## 연출 불일치

상대 보드에는 합체 파문이나 콤보 팝업 같은 연출이 미표시. 처음엔 축약판으로 만들었는데, 내 화면과 상대 화면의 연출 차이가 눈에 띄어 반려.

원인을 추적하다 접근 자체의 오류를 확인. 데이터를 더 보내면 되는 문제가 아니라, 받는 쪽이 아예 다른 그리기 코드로 그리는 구조가 원인.

그래서 봇전에서 쓰던 그리기 함수들을 공유 모듈로 이동. 본문은 한 줄도 수정하지 않고 위치만 변경. 받는 쪽은 이벤트로 연출 상태를 재구성해 같은 함수를 호출. 결과는 픽셀 단위 일치, 앞으로 연출을 손보면 양쪽에 자동 반영.

이동하는 김에, 그리기 모듈이 빌드 도구 전용 문법에 묶여 테스트에서 열리지 않던 문제도 정리. 순수한 계산 부분을 따로 분리하는 방식으로 해결.

## 끊김 시의 심판 주체

가장 크게 깨진 지점.

한쪽 네트워크가 끊겼을 때의 처리 로직은 이미 존재. 실기기 두 대로 시험하자 세 가지 문제가 한꺼번에 노출. 끊긴 쪽 화면에 「내가 끊겼다」와 「상대가 끊겼다」가 동시 표시. 잠깐 끊겼다 금방 돌아와도 양쪽 모두 패배 처리. 돌아오지 않은 사람 화면에는 자기 승리 표시.

근본 원인은 둘. 하나는 끊긴 기기에도 남아 있던 판정 권한. 다른 하나는 상대 신호의 신선도를 재는 기준 시점의 양쪽 불일치. 그 탓에 돌아온 사람을 미복귀자로 판독.

끊긴 쪽의 심판 자격은 전면 박탈. 판정은 남아 있는 쪽 단독. 신선도 측정 기준도 절대 시각 하나로 통일. 그리고 한쪽이 끊기면 양쪽 보드를 함께 정지. 돌아오면 카운트다운 후 재개, 끝내 미복귀 시 그때 남은 사람이 결론 확정. 결론을 낼 때도 서버에 먼저 자리를 확보한 뒤 화면을 종료하도록 순서 변경.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/it-broke-on-two-real-devices/disconnect.ko.svg"><img src="/assets/img/posts/it-broke-on-two-real-devices/disconnect.ko.svg" width="720" height="330" loading="lazy" alt="한쪽 연결이 끊기면 양쪽 보드 동시 정지. 복귀 시 카운트다운 후 재개, 미복귀 시 남은 쪽만 판정해 서버에 결과 자리를 먼저 확보한 뒤 화면 종료. 끊긴 기기는 판정 권한 박탈, 나가기 버튼은 즉시 자기 패배."></a></div><figcaption>끊긴 기기의 판정 권한 박탈, 남은 쪽은 서버에 결론을 먼저 기록한 뒤 화면 종료.</figcaption></figure>

나가기 버튼은 별도 처리. 통신 끊김과 자발적 퇴장은 성격이 다른 사건. 나가기를 누르면 그 자리에서 자기 패배로 확정.

추가로 정한 규칙 하나. 정지 후 재개된 경기는 결과 미기록. 아직 끝나지 않은 경기이기 때문.

## 같은 기기, 다른 화면

같은 기기인데 화면이 딴판으로 보인다는 제보, 원인 파악에 한참 소요.

범인은 시스템 글꼴 크기 설정. 안드로이드 웹뷰는 사용자가 설정에서 글꼴을 키우면 그만큼 텍스트를 확대해 렌더링. 그런데 내 레이아웃은 버튼 최소 크기와 비율 기반 높이를 섞어 쓰는 구조라, 글자가 커지는 순간 요소가 밀려나거나 잘리는 현상 발생.

브라우저를 화면 없이 띄워 화면 크기와 글꼴 배율 조합을 만들고 픽셀 단위로 재현. 실기기를 계속 붙잡지 않고도 원인 범위 축소 가능. 최종적으로 앱에서 시스템 배율을 무시하도록 고정. 어떤 배율에서도 같은 픽셀 출력.

## 「겹치지 않게」가 아니라 「겹칠 수 없게」

홈 화면 왼쪽 버튼들의 상호 겹침. 각 요소에 절대 좌표를 준 상태에서 라벨 길이가 달라지며 밀린 것이 원인.

좌표를 조정해 겹침만 피하는 방법도 가능. 그 대신 셋을 하나의 흐름 배치 안에 배치. 이제 라벨이 몇 줄이 되든 위아래로 밀릴 뿐, 겹침 자체가 불가능한 구조.

높이도 고정값에서 최소값으로 변경. 내용이 커지면 상자도 함께 확장.

## 미수정 판단

「빌드 버전이 안 올라간 것 같다」는 문제 제기.

설정 파일은 이미 정확. 다른 곳에서 덮어쓰는지 전부 조사했지만 해당 지점 부재. 마지막으로 방금 만들어진 빌드 산출물을 열어 안에 박힌 버전 정보를 직접 추출. 새 버전이 정확히 포함된 상태.

그래서 수정 없음으로 결정. 예전 파일이나 캐시된 화면을 봤을 가능성을 공유.

문제가 없는데 무언가를 고치는 쪽이 더 위험. 원인을 모르는 채로 손을 대면 그때부터 진짜 문제가 시작.

## 배운 것

여섯 건을 한 표로 정리하면 아래와 같은 구성.

| 증상 | 근본 원인 | 조치 |
|------|----------|------|
| 큰 화면이 유리 | 물리 좌표계가 화면 픽셀에 결합 | 논리 좌표계 분리, 화면은 스케일링만 담당 |
| 상대 보드 연출 누락 | 수신 측의 별도 렌더링 코드 | 그리기 함수를 공유 모듈로 이동 |
| 끊김 판정 오류 3종 | 끊긴 기기에 판정 권한 잔존 + 신선도 기준 시각 불일치 | 잔류 측 단독 판정, 절대 시각 통일, 동반 일시정지 |
| 같은 기기, 다른 화면 | 시스템 글꼴 배율의 WebView 텍스트 확대 | 앱에서 배율 고정 |
| 홈 버튼 겹침 | 절대 좌표 + 가변 라벨 길이 | 플로우 배치 전환, 고정 높이 → 최소 높이 |
| 버전 미반영 제보 | 결함 아님 — 구 산출물/캐시 열람 추정 | 산출물 직접 확인 후 수정 없이 원인 공유 |

여섯 가지 모두 로컬에서는 초록불. 개발용 탭 두 개로 돌리면 화면 크기 동일, 네트워크 끊김 0, 글꼴 배율 기본값, 라벨도 짧은 상태.

실제 기기와 실제 사용자는 그 전제를 하나씩 붕괴시키는 존재. 그래서 여섯 번 모두 같은 지점으로 귀결. 조건을 맞춰서 문제가 안 나게 하는 것과, 조건이 어긋나도 문제가 생길 수 없게 만드는 것은 별개. 앞의 방식은 다음 기기에서 재발.

---

## English version

[In the previous post](/2026/07/27/you-cant-report-your-own-win/) I wrote about moving 1v1 judgment and settlement onto the server. With that in place, two browser tabs on my laptop played beautifully. Verdicts agreed, matchmaking paired, trophies settled.

Then I turned on two real phones, and it came apart one piece at a time.

### The bigger screen was winning

The physics world had been built directly on screen pixels. The canvas was measured, walls were placed at its edges, and falling objects were a fixed pixel size. A bigger screen meant wider walls with the same-size objects. On a desktop, more than three times as much fit on the same board.

In a merge puzzle, fitting more is simply an advantage. Whoever had the bigger screen won.

As a stopgap I capped the board size to pull the spread back into phone range. The real fix was detaching the coordinate system: physics now runs on fixed logical dimensions, and the screen only stretches the result for display. Resizing the window does nothing physical at all.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/it-broke-on-two-real-devices/coordinates.en.svg"><img src="/assets/img/posts/it-broke-on-two-real-devices/coordinates.en.svg" width="720" height="320" loading="lazy" alt="Before: physics sits on screen pixels, so a big desktop screen fits several times more same-size pieces than a phone. After: physics runs on a fixed-size logical board, and phone and desktop only scale that same board for display."></a></div><figcaption>Physics runs on a fixed logical board; the screen only scales it. Resizing the window does nothing physical.</figcaption></figure>

The engine is shared with single-player, so I couldn't just rewrite it. I added one option that, when left unset, leaves the original path untouched. Correcting touch coordinates for the display ratio was the only other piece; everything else lined up on its own.

### The effects didn't match

The opponent's board had no merge ripples, no combo popups. I'd built a trimmed-down version at first, and it looked visibly different from my own side, so it got sent back.

Digging into it, I realized the whole approach was wrong. It wasn't a matter of transmitting more data. The receiving side was drawing with entirely different code.

So I moved the drawing functions from bot matches into a shared module — relocated, not rewritten, without touching a line of the bodies. The receiving side now reconstructs the effect state from events and calls those same functions. It matches pixel for pixel, and any future change to the effects updates both sides automatically.

While I was in there, I also cleared up something that had been blocking tests: the drawing module was tied to build-tool-specific syntax and wouldn't open outside the bundler. Splitting the pure computation into its own layer solved it properly.

### When the connection drops, who is the referee?

This is where it broke worst.

There was already logic for one side losing its connection. Testing on two real devices exposed three failures at once. The disconnected side's screen showed "you dropped" and "your opponent dropped" simultaneously. A brief drop with a quick return still charged both players a loss. And the player who never came back saw a victory screen.

There were two roots. One was that the disconnected device still held judging authority. The other was that each side measured the freshness of the opponent's last signal from a different reference point, so a player who had come back could read as one who never did.

I stripped that authority entirely: only the side that stayed connected judges. Freshness is now measured against one absolute clock. And when one side drops, both boards pause together — return, and a countdown resumes play; never return, and the remaining player finalizes. I also reordered that finalization so the remaining player claims the outcome on the server *before* ending the match locally.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/it-broke-on-two-real-devices/disconnect.en.svg"><img src="/assets/img/posts/it-broke-on-two-real-devices/disconnect.en.svg" width="720" height="330" loading="lazy" alt="When one side drops, both boards pause. If it returns, a countdown resumes play; if not, only the remaining side judges, claiming the result slot on the server before ending the match locally. The dropped device has no judging authority, and quitting is an instant own loss."></a></div><figcaption>The dropped device loses all judging authority, and the remaining side records the outcome on the server before closing the match.</figcaption></figure>

Quitting is handled separately. Losing a connection and choosing to leave aren't the same thing, so pressing quit resolves immediately as your own defeat.

One more rule: a match that paused and resumed doesn't get a result recorded. It isn't over yet.

### Same device, different screen

I got a report that the same device could render the UI completely differently, and it took a while to track down.

The culprit was the system font size setting. Android's WebView scales text up when the user raises the font size in system settings. My layout mixed minimum button sizes with ratio-based heights, so the moment text grew, elements pushed out of place or got clipped.

I ran a headless browser across combinations of viewport size and font scale and reproduced it pixel by pixel, which meant I could narrow the cause without keeping a phone in my hand. In the end the app pins the system scale so the pixels are identical at any setting.

### Not "don't overlap" — "can't overlap"

The buttons on the left side of the home screen were overlapping each other. Each element had absolute coordinates, and varying label lengths pushed them into one another.

I could have nudged the coordinates until they didn't collide. I didn't. I put all three inside a single flow container. Now, however many lines a label wraps to, they push each other down instead — overlapping isn't possible.

Heights moved from fixed to minimum as well, so a box grows with its contents.

### The thing I decided not to fix

Someone raised that the build version didn't look like it had gone up.

The config file was already correct. I checked everywhere that could override it and found nothing. Finally I opened the freshly built artifact and pulled the version information straight out of it. The new version was there, exactly as expected.

So I changed nothing, and passed along that an older artifact or a cached screen was the likely explanation.

Editing code that isn't broken is the more dangerous move. Touch it without knowing the cause and that's when you get a real problem.

### What I took from it

The six cases, in one table:

| Symptom | Root cause | Fix |
|---------|-----------|-----|
| Bigger screen wins | Physics coupled to screen pixels | Detached logical coordinates; screen only scales |
| Opponent's effects missing | Receiving side drew with separate code | Drawing functions moved into a shared module |
| Three disconnect-verdict failures | Judging authority left on the dropped device + mismatched freshness clocks | Only the connected side judges; one absolute clock; paired pause |
| Same device, different screen | System font scale inflating WebView text | Scale pinned in the app |
| Overlapping home buttons | Absolute coordinates + variable label lengths | Flow layout; fixed heights → minimums |
| "Version didn't bump" report | Not a defect — stale artifact/cache | Verified the artifact directly; changed nothing, shared the cause |

All six were green locally. Two dev tabs mean identical screen sizes, no dropped connections, default font scaling, short labels.

Real devices and real users dismantle those assumptions one by one. And all six times I landed in the same place: arranging conditions so a problem doesn't appear is not the same as making the problem impossible when conditions go wrong. The first kind comes back on the next device.
