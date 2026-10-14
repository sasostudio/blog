---
layout: post
title: "실기기 검증 포스트모템"
description: "로컬 탭 두 개에서는 잘 돌던 1대1 대전이 실기기 두 대에서 깨졌다. 화면 크기 유불리, 연출 불일치, 끊김 판정, 시스템 글꼴 배율, 버튼 겹침을 잡고, 결함이 아니었던 1건은 고치지 않은 기록."
date: 2026-07-27 11:00:00 +0900
project: moongchi
tags: [game-dev, multiplayer, mobile, build-in-public]
---

*🇬🇧 [English version below](#english-version)*

[지난 글](/2026/07/27/you-cant-report-your-own-win/)에서 1대1 대전의 판정과 정산을 서버 쪽으로 옮긴 이야기를 썼다. 그렇게 해놓고 로컬에서 브라우저 탭 두 개를 띄워 붙여보면 잘 돌았다. 판정도 맞고, 매칭도 붙고, 정산도 됐다.

실기기 두 대에서 검증을 돌리자 하나씩 깨졌다. 결함 5건을 잡고 1건은 결함이 아님을 확인한 기록이다.

## 큰 화면이 유리했다

물리 세계가 화면 픽셀 위에 그대로 세워져 있었다. 캔버스 크기를 재서 그 자리에 벽을 세우고, 떨어지는 오브젝트는 고정된 픽셀 크기였다. 화면이 크면 벽 사이가 넓어지는데 오브젝트 크기는 그대로다. 데스크톱에서는 같은 보드에 세 배 넘게 담겼다.

합체 퍼즐에서 더 담긴다는 건 그냥 유리하다는 뜻이다. 화면 큰 사람이 이긴다.

급한 대로 보드 크기에 상한을 걸어 편차를 폰 범위로 좁혔다. 근본 해결은 좌표계를 떼어내는 것이었다. 물리는 고정된 논리 치수 위에서 돌고, 화면은 그 결과를 늘려서 보여주기만 한다. 창 크기를 바꿔도 물리적으로는 아무 일도 일어나지 않는다.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/it-broke-on-two-real-devices/coordinates.ko.svg"><img src="/assets/img/posts/it-broke-on-two-real-devices/coordinates.ko.svg" width="720" height="320" loading="lazy" alt="전: 물리가 화면 픽셀 위에 있어 같은 크기 공이 큰 데스크톱 화면에 폰보다 몇 배 많이 담긴다. 후: 고정 치수의 논리 보드에서 물리를 돌리고, 폰과 데스크톱은 같은 판을 늘려서 보여주기만 한다."></a></div><figcaption>물리는 고정된 논리 보드에서 돌고 화면은 확대만 한다. 창 크기를 바꿔도 물리적으로는 아무 일도 일어나지 않는다.</figcaption></figure>

엔진은 싱글 모드와 공유하고 있어서 함부로 건드릴 수 없었다. 옵션을 하나 더하되 지정하지 않으면 기존 경로가 그대로 도는 방식으로 넣었다. 터치 좌표만 화면 비율에 맞춰 보정해주면 나머지는 그대로 맞아떨어졌다.

## 같은 연출이 안 나왔다

상대 보드에는 합체 파문이나 콤보 팝업 같은 연출이 나오지 않았다. 처음에는 축약해서 만들었는데, 내 화면과 상대 화면의 연출이 눈에 띄게 달라서 반려됐다.

원인을 찾다가 접근 자체가 틀렸다는 걸 알았다. 데이터를 더 보내면 되는 문제가 아니었다. 받는 쪽이 아예 다른 그리기 코드로 그리고 있었던 것이다.

그래서 봇전에서 쓰던 그리기 함수들을 공유 모듈로 옮겼다. 본문은 한 줄도 고치지 않고 위치만 바꿨다. 받는 쪽은 이벤트로 연출 상태를 재구성해서 같은 함수를 호출한다. 이제 픽셀 단위로 같고, 앞으로 연출을 손보면 양쪽이 자동으로 함께 바뀐다.

옮기는 김에 그리기 모듈이 빌드 도구 전용 문법에 묶여 있어 테스트에서 열리지 않던 문제도 걷어냈다. 순수한 계산 부분을 따로 떼어내는 쪽으로 풀었다.

## 끊기면 누가 심판인가

가장 크게 깨진 지점이다.

한쪽 네트워크가 끊겼을 때를 처리하는 로직은 이미 있었다. 실기기 두 대로 시험하니 세 가지가 한꺼번에 드러났다. 끊긴 쪽 화면에는 "내가 끊겼다"와 "상대가 끊겼다"가 동시에 떠 있었다. 잠깐 끊겼다가 금방 돌아와도 양쪽 모두 패배로 처리됐다. 돌아오지 않은 사람 화면에는 자기가 이겼다고 떠 있었다.

뿌리는 둘이었다. 하나는 끊긴 기기에게도 판정 권한이 남아 있었다는 것. 다른 하나는 상대 신호가 얼마나 최근 것인지 재는 기준 시점이 양쪽에서 어긋나 있었다는 것이다. 그래서 돌아온 사람을 안 돌아온 사람으로 읽었다.

끊긴 쪽의 심판 자격을 아예 없앴다. 남아 있는 쪽만 판정한다. 신선도를 재는 기준도 절대 시각 하나로 통일했다. 그리고 한쪽이 끊기면 양쪽 보드를 함께 멈춘다. 돌아오면 카운트다운을 세고 재개하고, 끝내 안 돌아오면 그때 남은 사람이 결론을 낸다. 남은 사람이 결론을 낼 때도 서버에 먼저 자리를 잡은 뒤에 화면을 끝내도록 순서를 바꿨다.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/it-broke-on-two-real-devices/disconnect.ko.svg"><img src="/assets/img/posts/it-broke-on-two-real-devices/disconnect.ko.svg" width="720" height="330" loading="lazy" alt="한쪽 연결이 끊기면 양쪽 보드가 함께 멈춘다. 돌아오면 카운트다운 후 재개하고, 돌아오지 않으면 남은 쪽만 판정해 서버에 결과 자리를 먼저 확보한 뒤 화면을 끝낸다. 끊긴 기기는 판정 권한이 없고, 나가기 버튼은 즉시 자기 패배다."></a></div><figcaption>끊긴 기기에서 판정 권한을 없애고, 남은 쪽은 서버에 결론을 먼저 남긴 뒤에 화면을 닫는다.</figcaption></figure>

나가기 버튼은 따로 다뤘다. 통신이 끊긴 것과 스스로 나간 것은 성격이 다르다. 나가기를 누르면 그 자리에서 자기 패배로 확정된다.

한 가지 더 정했다. 멈췄다가 다시 이어진 경기는 결과를 기록하지 않는다. 아직 안 끝난 경기니까.

## 같은 기기인데 화면이 달랐다

같은 기기인데 화면이 딴판으로 보인다는 제보를 받고 한참 헤맸다.

범인은 시스템 글꼴 크기 설정이었다. 안드로이드 웹뷰는 사용자가 설정에서 글꼴을 키우면 그만큼 텍스트를 키워서 그린다. 그런데 내 레이아웃은 버튼 최소 크기와 비율 기반 높이를 섞어 쓰고 있어서, 글자가 커지는 순간 요소가 밀려 나가거나 잘렸다.

브라우저를 화면 없이 띄워 화면 크기와 글꼴 배율 조합을 만들어두고 픽셀로 재현했다. 실기기를 계속 붙잡고 있지 않아도 원인을 좁힐 수 있었다. 결국 앱에서는 시스템 배율을 무시하도록 고정했다. 어떤 배율에서도 같은 픽셀이 나온다.

## 겹치지 않게가 아니라, 겹칠 수 없게

홈 화면 왼쪽 버튼들이 서로 겹쳤다. 각 요소에 절대 좌표를 줬는데 라벨 길이가 달라지면서 밀린 것이다.

좌표를 조정해서 안 겹치게 만들 수도 있었다. 그렇게 하지 않았다. 셋을 하나의 흐름 배치 안에 넣었다. 이제 라벨이 몇 줄이 되든 위아래로 밀릴 뿐, 겹치는 일 자체가 불가능하다.

높이도 고정값에서 최소값으로 바꿨다. 내용이 커지면 상자가 따라 커진다.

## 고치지 않기로 한 것

빌드 버전이 안 올라간 것 같다는 얘기가 나왔다.

설정 파일을 봤더니 이미 정확했다. 다른 곳에서 덮어쓰는지 전부 뒤졌지만 그런 곳은 없었다. 마지막으로 방금 만들어진 빌드 산출물을 열어 안에 박힌 버전 정보를 직접 꺼내봤다. 새 버전이 정확히 들어 있었다.

그래서 아무것도 고치지 않았다. 예전 파일이나 캐시된 화면을 봤을 가능성이 있다고 알렸다.

문제가 없는데 뭔가 고치는 쪽이 더 위험하다. 원인을 모르는 채로 손을 대면 그때부터 진짜 문제가 생긴다.

## 배운 것

여섯 건을 한 표로 정리하면 이렇다.

| 증상 | 근본 원인 | 조치 |
|------|----------|------|
| 큰 화면이 유리 | 물리 좌표계가 화면 픽셀에 결합 | 논리 좌표계 분리, 화면은 스케일링만 담당 |
| 상대 보드 연출 누락 | 수신 측이 별도 렌더링 코드로 그림 | 그리기 함수를 공유 모듈로 이동 |
| 끊김 판정 오류 3종 | 끊긴 기기에 판정 권한 잔존 + 신선도 기준 시각 불일치 | 잔류 측 단독 판정, 절대 시각 통일, 동반 일시정지 |
| 같은 기기, 다른 화면 | 시스템 글꼴 배율이 WebView 텍스트를 확대 | 앱에서 배율 고정 |
| 홈 버튼 겹침 | 절대 좌표 + 가변 라벨 길이 | 플로우 배치 전환, 고정 높이 → 최소 높이 |
| 버전 미반영 제보 | 결함 아님 — 구 산출물/캐시 열람 추정 | 산출물 직접 확인 후 수정 없이 원인 공유 |

여섯 가지 모두 로컬에서는 초록불이었다. 개발용 탭 두 개로 돌리면 화면 크기가 같고, 네트워크가 안 끊기고, 글꼴 배율이 기본값이고, 라벨이 짧다.

실제 기기와 실제 사용자는 그 전제를 하나씩 무너뜨린다. 그래서 여섯 번 다 같은 자리로 돌아왔다. 조건을 맞춰서 문제가 안 나게 하는 것과, 조건이 어긋나도 문제가 생길 수 없게 만드는 것은 다르다. 앞의 것은 다음 기기에서 또 터진다.

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
