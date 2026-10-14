---
layout: post
title: "뭉치뭉치 1.2.4 업데이트 회고"
description: "물리 합체 퍼즐 뭉치뭉치의 1.2.4 업데이트를 네 채널에 내기까지의 검증 기록. 게임 시작 때 BGM 지연 구조를 가짜 오디오 하네스로 재현 후 수정, 「광고 뒤 앱 종료」 증상을 라이브와 같은 코드의 대조 번들로 테스트 환경 현상으로 판정, 주간 보상 서버 함수는 에뮬레이터 실행 후 배포."
date: 2026-10-15 00:30:00 +0900
project: moongchi
tags: [build-in-public, game-dev, testing, firebase]
---

*🇬🇧 [English version below](#english-version)*

뭉치뭉치는 수박게임 계열의 물리 합체 퍼즐. 출시 채널은 웹, Google Play, 원스토어, 토스 앱 안의 미니앱까지 넷. 10월 14일, 1.2.4 업데이트를 네 채널 모두에 출시.

이번 업데이트의 내용은 네 가지. 광고 시청 후 이어하기 흐름 보완, 하트 경제 조정, 주간 랭킹 보상, BGM 지연 문제 수정. 경제 수치는 이 글에서 제외. 대신 각 변경의 검증 방법을 기록. 이번 업데이트에서 시간이 더 많이 든 쪽은 수정 자체보다 수정 여부의 입증.

## BGM: 시작 직후 효과음만, 한참 뒤 갑자기 음악

주변 사람에게서 들어온 제보. 게임을 시작하면 배경음악 없이 효과음만 나오다가, 한참 진행하면 갑자기 음악이 나온다는 내용.

코드와 음원 서버부터 점검.

| 점검 대상 | 결과 |
|---|---|
| BGM 위치 | 홈 화면 곡 포함 4곡 전부 원격 저장소. 로컬에는 시작 효과음만 존재 |
| 곡 크기 | 곡당 4.3~5.0MB |
| 저장소 주소 | 개발용 공개 주소. 캐싱 미지원, 속도 제한 가능성이 공식 문서에 명시 |
| 곡 전환 코드 | 새 곡 지정·재생 호출 뒤, **데이터 도착 여부와 무관하게** 1초 볼륨 페이드 진행 |

핵심은 마지막 행. 페이드가 데이터 도착과 상관없이 끝나는 구조라, 곡 데이터가 늦게 오면 페이드가 다 끝난 뒤에야 재생 시작. 사용자 입장에서는 「갑자기 나온다」는 체감. 게임 도중 구간 전환도 같은 구조라, 새 곡이 아직 도착 전이면 이전 곡만 꺼지고 무음 구간 발생.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/moongchi-1-2-4-update/bgm-crossfade.ko.svg"><img src="/assets/img/posts/moongchi-1-2-4-update/bgm-crossfade.ko.svg" width="720" height="270" loading="lazy" alt="수정 전: 곡 전환 페이드가 데이터 도착과 무관하게 진행, 이전 곡은 꺼지고 새 곡 데이터가 늦으면 그사이 무음. 수정 후: 새 곡의 실제 재생 시작 이후에 페이드, 그때까지 이전 곡 유지."></a></div><figcaption>페이드 기준을 타이머가 아니라 「새 곡의 실제 재생 시작」 시점으로 변경.</figcaption></figure>

| 수정 | 내용 |
|---|---|
| 페이드 시점 | 새 곡의 실제 재생 시작 이후 페이드. 대기 중에는 이전 곡 유지 |
| 재생 실패 | 새 곡 오류 시 예전 동작으로 복귀 |
| 연속 전환 | 대기 중 다음 전환이 오면 이전 대기 취소 |
| 계측 | 재생 시작까지 걸린 시간을 이벤트로 기록 |

검토 후 제외한 안도 하나. 다음 구간 곡을 미리 받아 두면 전환 지연은 확실히 해소. 그러나 대부분의 플레이어는 첫 구간을 넘기기 전에 판이 종료. 그 사람들에게 곡당 4~5MB 추가 다운로드를 강요하는 건 손해라는 판단으로 미적용.

검증은 실제 기기 반복 대신 가짜 오디오 하네스로 진행. 브라우저의 오디오 객체를 흉내 내서 「데이터 지연」, 「재생 실패」, 「대기 중 다음 전환」 같은 상황을 임의로 생성하는 방식. 새 코드에서 9개 시나리오 전부 통과, 같은 시나리오를 예전 코드에 돌리면 무음 구간이 생기는 2개가 실패. 고치기 전 코드의 실패를 직접 봐야 이 테스트가 무언가를 지킨다는 주장이 성립. 이후 실기기에서도 게임 시작과 배경 전환 구간의 끊김 0건 확인.

## 「광고를 끝까지 보고 닫으면 앱 종료」

변경을 담은 테스트 번들을 기기에서 확인하던 중, 보상형 광고를 끝까지 보고 닫으면 게임이 통째로 닫히는 증상 발생. 하필 광고 이어하기 흐름을 손본 업데이트. 내 수정을 의심하는 게 자연스러운 상황.

변경 경로를 먼저 검토한 결과 수정 탓일 가능성은 낮다고 판단. 다만 양쪽 비교가 1회씩뿐이라 결론 보류. 그래서 대조군 제작. 지금 라이브에 나가 있는 것과 **똑같은 코드**로 테스트 번들을 하나 더 만들어, 같은 테스트 배포 경로로 실행.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/moongchi-1-2-4-update/control-bundle.ko.svg"><img src="/assets/img/posts/moongchi-1-2-4-update/control-bundle.ko.svg" width="720" height="260" loading="lazy" alt="새 변경을 담은 번들과 라이브와 같은 코드의 대조 번들을 같은 테스트 배포 경로로 실행. 두 번들 모두 광고 뒤 앱이 닫혔으므로 원인은 변경이 아니라 테스트 배포 환경. 라이브에서는 같은 흐름이 정상."></a></div><figcaption>대조군에서도 같은 증상이면 원인은 변경과 무관. 대조군 하나로 추가 탐색을 생략.</figcaption></figure>

결과는 명확. 라이브와 같은 코드도 테스트 배포에서는 광고 뒤 종료. 라이브 앱에서 같은 흐름이 정상으로 이어지는 점은 이미 확인된 상태. 결론은 수정 탓이 아닌 테스트 배포 환경 고유의 현상. 이 판정 이후 코드 쪽 원인 탐색에 들인 시간은 0.

같은 확인 중에 요청 하나 접수. 광고 버튼을 누른 뒤 광고가 뜨기까지 로딩 표시를 넣어 달라는 내용. 광고 처리 중에만 「광고를 불러오는 중이에요」 창을 띄우되 포기 버튼은 가리지 않게 처리, 헤드리스 브라우저 렌더로 배치 확인 후 번들 재제작. 새 번들에서 로딩 창, 이어하기, BGM 모두 정상.

## 주간 랭킹 보상: 서버 함수 실행 후 배포

주간 랭킹 상위 플레이어에게 매주 하트를 지급하는 서버 함수를 재활성화. 보드 기준으로 순위를 다시 산정. 보유 상한 초과 금지, 같은 주 중복 지급 금지가 조건.

이런 함수는 배포 후 다음 월요일 새벽의 실제 실행 전까지 결과를 알 수 없는 구조. 그래서 배포 전에 Firestore 에뮬레이터에 데이터를 깔고, 빌드된 함수를 지급 시각으로 맞춰 직접 실행.

| 점검 항목 | 결과 |
|---|---|
| 보드 기준 순위 연속 산정 | 통과 |
| 보유 상한에서 절삭 (상한 직전·상한) | 통과 |
| 하트 필드가 없는 유저 | 통과 |
| 순위 밖·점수 0·다른 주 기록은 미지급 | 통과 |
| 다른 채널 컬렉션도 동일 처리 | 통과 |
| 같은 주 재실행 시 추가 지급 0 | 통과 |

36개 점검 전부 통과. 홈 화면 보상 팝업도 헤드리스 브라우저로 세 경우(일반 지급, 상한 때문에 일부만 지급, 몇 주 지난 기록)를 렌더해 문구와 배치 확인.

실행하지 못한 범위도 기록. 실제 서버 스케줄러가 정해진 시각에 함수를 호출하는 부분은 배포 후에만 확인 가능. 그래서 첫 실지급 날짜를 남은 확인 항목으로 등록.

에뮬레이터에서 하나 더 발견. 월요일 새벽, 주가 바뀐 뒤 지급 함수 실행 전 몇 분 사이에 플레이한 유저는 기록이 새 주로 넘어가 지난주 보상 대상에서 누락. 주 경계 처리 순서에서 생기는 수 분짜리 틈. 관리자 판단으로 이번엔 미수정. 미수정 결정도 기록.

## 인게임 공지: 과거 깨짐 이력

업데이트 공지는 앱이 서버 문서를 읽어 팝업으로 표시하는 구조. 과거에 공지 본문의 줄바꿈이 `\n` 글자 그대로 노출된 이력이 있어, 이번에는 업로드 전후 모두 점검.

| 시점 | 점검 |
|---|---|
| 업로드 전 | 본문 백슬래시 0개, 실제 줄바꿈 9개, 필드 이름 오타 0 |
| 업로드 전 | 실제 공지 컴포넌트 헤드리스 렌더(360폭, 320폭). 320폭에서는 확인 버튼까지 스크롤이 필요하지만 기존 공지와 같은 구조 |
| 업로드 전 | 이모지 2종이 앱 내장 폰트에 미포함, 기기 시스템 이모지로 표시되는 점 확인 |
| 업로드 후 | 필드 4개, 글자 그대로의 `\n` 0건, 최신 문서 = 새 공지, 기존 공지 문서 손상 0 |

## 플랫폼 기능 종료 대응

업데이트 전에 먼저 처리한 일 하나. 플랫폼 쪽 공유 보상 기능 종료로, 그 기능을 쓰던 「초대하고 하트받기」를 코드에서 제거할 필요 발생. 문제는 Play 채널에 이름이 비슷한 별개의 친구 초대 시스템이 존재한다는 점. 이름 기준으로 찾아 지우면 엉뚱한 기능까지 삭제될 위험.

그래서 삭제 대상과 유지 대상을 먼저 분리하고, 삭제 후에는 빌드 결과물의 문자열을 변경 전후로 대조. 해당 기능 호출 문자열은 5개에서 2개로 감소, 남은 2개는 SDK 라이브러리 쪽 정의라 우리 호출이 아님을 확인. 「초대하고」 문구는 0개. 변경 규모는 6개 파일, 11줄 추가·103줄 삭제.

## 정리

| 변경 | 수정 방법 | 검증 |
|---|---|---|
| BGM 지연 | 새 곡 재생 시작 이후 페이드 | 가짜 오디오 하네스 9/9, 예전 코드는 2개 실패 |
| 광고 뒤 앱 종료 | 미수정 (테스트 환경 현상) | 라이브와 같은 코드의 대조 번들 |
| 주간 보상 | 서버 함수 재활성화 | 에뮬레이터 실행 36/36, 팝업 렌더 3종 |
| 인게임 공지 | 형식 사전·사후 점검 | 실제 컴포넌트 렌더, 업로드 후 재조회 |
| 공유 보상 종료 | 해당 기능만 제거 | 빌드 결과물 문자열 전후 대조 |

- 테스트의 의미는 고치기 전 코드에서의 실패 확인이 전제.
- 증상의 원인이 내 변경인지 코드 검토로 결론이 나지 않으면 대조군 제작.
- 서버에서 정해진 시각에 도는 코드는 배포 전에 그 시각으로 맞춰 직접 실행. 실행하지 못한 범위는 날짜와 함께 남은 확인 항목으로 등록.
- 미수정 결정도 기록. 다음 사람의 같은 조사를 방지하는 목적.

---

## English version

Moongchi is a physics merge puzzle in the Suika Game vein. It ships on four channels: the web, Google Play, ONE store, and as a mini-app inside the Toss app. On October 14, version 1.2.4 went out on all four.

The update had four parts: a better rewarded-ad continue flow, a rebalanced heart economy, weekly ranking rewards, and a fix for background music starting late. I'm leaving the economy numbers out of this post. What's here is how each change was verified — this update took more time proving fixes than making them.

### BGM: only sound effects at first, then the music suddenly kicks in

The report came from someone close to me: start a game and there's no music, just sound effects, and then partway through the music suddenly appears.

I checked the code and the audio host first.

| Checked | Finding |
|---|---|
| Where the BGM lives | All four tracks, including the home-screen track, are remote. Only the start sound effect is local |
| Track size | 4.3–5.0 MB each |
| Host address | A development public URL. Its docs say it isn't cached and can be rate-limited |
| Track-switch code | Sets the new source, calls play, then runs a 1-second volume fade **whether or not any data has arrived** |

The last row was the key. The fade finishes on its own schedule, so if the track data is slow, playback only starts after the fade is already over — which sounds like the music suddenly jumping in. Section changes mid-game use the same structure: if the next track hasn't arrived, the old one fades out and you get silence.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/moongchi-1-2-4-update/bgm-crossfade.en.svg"><img src="/assets/img/posts/moongchi-1-2-4-update/bgm-crossfade.en.svg" width="720" height="270" loading="lazy" alt="Before the fix, the track-switch fade ran regardless of data arrival, so the old track faded out and silence lasted until the new track's data arrived. After the fix, the fade starts only once the new track is actually playing, and the old track keeps playing until then."></a></div><figcaption>The fade is now keyed to the moment the new track actually starts playing, not to a timer.</figcaption></figure>

| Change | What it does |
|---|---|
| Fade timing | Fade only after the new track is actually playing; keep the old track until then |
| Playback failure | If the new track errors, fall back to the old behavior |
| Back-to-back switches | A new switch cancels a pending wait |
| Instrumentation | Log how long playback took to start |

One idea was considered and dropped: prefetching the next section's track would remove the switch delay entirely. But most players' games end before they leave the first section, and making them download an extra 4–5 MB per track for nothing is a bad trade.

Instead of hammering on real devices, verification ran on a fake audio harness: a stand-in for the browser's audio object that can produce "data arrives late," "playback fails," "the next switch arrives while waiting," on demand. All 9 scenarios pass on the new code, and running the same scenarios on the old code fails the 2 that produce silence. Seeing the old code fail is what makes the test mean anything. Real devices afterwards showed no gaps at game start or on background changes.

### "Watch an ad to the end, close it, and the app quits"

While checking a test bundle with the changes on a device, the game closed entirely after a rewarded ad was watched and dismissed. This was the very update that touched the ad-continue flow, so suspecting my own change was the natural move.

A first pass over the changed code paths made my change unlikely, but with one try on each side it wasn't conclusive. So I built a control: a second test bundle from **exactly the code that was live**, sent through the same test distribution.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/moongchi-1-2-4-update/control-bundle.en.svg"><img src="/assets/img/posts/moongchi-1-2-4-update/control-bundle.en.svg" width="720" height="260" loading="lazy" alt="A bundle with the new changes and a control bundle with the same code as live ran through the same test distribution. Both closed after the ad, so the cause was the test distribution environment, not the change. The same flow worked in the live app."></a></div><figcaption>If the control shows the symptom too, it isn't your change. One control bundle made further digging unnecessary.</figcaption></figure>

The answer was clear. The live code also closed after the ad in test distribution, and the same flow had already been confirmed working in the live app. So it was a test-distribution-only behavior, not the fix. After that call, no time went into hunting for a code cause.

A request came in during the same check: show something while the ad loads after the ad button is tapped. A "Loading the ad…" panel now appears only while the ad is being handled, without hiding the give-up button; its layout was checked in a headless browser render, and the bundle was rebuilt. On the new bundle the loading panel, the continue flow, and the BGM all behaved.

### Weekly ranking rewards: run the server function before shipping it

A scheduled server function that gives hearts to the top players each week was turned back on. It re-ranks from the board. It must never push anyone past the holding cap, and never pay twice in one week.

Code like this normally reveals itself only when it fires early Monday morning after deployment. So before deploying, I seeded the Firestore emulator and ran the built function directly at the payout time.

| Check | Result |
|---|---|
| Ranks computed from the board | pass |
| Capped at the holding limit (just below and at the cap) | pass |
| Players with no heart field | pass |
| Nothing paid outside the ranks, for zero scores, or for other weeks | pass |
| The other channel's collection handled the same way | pass |
| Re-running for the same week pays nothing extra | pass |

All 36 checks passed. The home-screen reward popup was also rendered headlessly for three cases (normal payout, partial payout because of the cap, a record several weeks old) to check wording and layout.

What couldn't be run is written down too: the real scheduler waking the function at its set time can only be confirmed after deployment, so the first live payout date is on the list of remaining checks.

The emulator surfaced one more thing. Anyone who plays in the few minutes on Monday morning after the week rolls over but before the payout runs gets their record moved into the new week, and drops out of last week's reward. It's a few-minute gap in how the week boundary is processed. It was left unfixed this time by an explicit call, and that decision is on record.

### The in-game notice: it broke once before

Update notices are popups the app renders from a server document. A notice once showed its line breaks as a literal `\n`, so this time the upload was checked before and after.

| When | Check |
|---|---|
| Before | Zero backslashes in the body, 9 real line breaks, no typos in field names |
| Before | Rendered the actual notice component headlessly at 360 and 320 px wide. At 320 the confirm button needs a scroll — same structure as earlier notices |
| Before | Two of the emoji aren't in the app's bundled font and fall back to the device's system emoji |
| After | 4 fields, no literal `\n`, the newest document is the new notice, existing notice documents intact |

### Responding to a platform feature shutting down

One thing came first. The platform was ending its share-reward feature, so the "invite friends for hearts" feature built on it had to come out of the code. The catch: the Play channel had a different friend-invite system with a similar name. Delete by name and you take the wrong thing with it.

So I first split what goes from what stays, and after removal compared strings in the built output before and after. Calls to that feature dropped from 5 to 2, and the remaining 2 were definitions inside the SDK library, not our calls. The "invite" wording went to zero. Six files changed, 11 lines added and 103 removed.

### Summary

| Change | Fix | Verification |
|---|---|---|
| Late BGM | Fade after the new track starts playing | Fake audio harness 9/9; old code fails 2 |
| App closes after an ad | Not fixed (test-environment behavior) | Control bundle built from live code |
| Weekly rewards | Server function re-enabled | Emulator run 36/36; popup rendered in 3 cases |
| In-game notice | Format checks before and after | Real component render; re-read after upload |
| Share-reward shutdown | Removed only that feature | Before/after string comparison of the build |

- A test only means something once you've watched it fail on the old code.
- When code review can't settle whether a symptom is your change, build a control.
- For code that fires on a server schedule, run it yourself at that time before deploying, and put whatever you couldn't run on a dated list of remaining checks.
- Record what you decided not to fix, so the next person doesn't investigate it again.
