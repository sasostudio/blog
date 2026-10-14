---
layout: post
title: "뭉치뭉치 - 런타임 채널 판정"
description: "하나의 코드베이스가 웹·미니앱·안드로이드 중 어디서 도는지 런타임에 판정하는 구조와, 거기서 나온 결함들. 전역 존재 검사의 함정, 분기를 추가할 때 봐야 할 세 곳, 인증 4겹 장애."
date: 2026-08-28 01:30:00 +0900
project: moongchi
tags: [architecture, mobile, firebase, build-in-public]
---

*🇬🇧 [English version below](#english-version)*

같은 게임을 세 곳에 낸다. 웹, 미니앱 플랫폼, 안드로이드 앱이다. 앱을 세 벌로 나누지 않고 **하나의 코드베이스가 런타임에 자기가 어디서 돌고 있는지 판정해** 동작을 바꾼다.

이 구조에서 나온 결함 몇 개를 정리한다. 판정 함수 한 줄이 틀리면 채널 하나가 통째로 죽는다.

## 1. 판정

```
window.__appsInToss 또는 'AIT' in window  → 'ait'        (미니앱)
isNativePlatform()                         → 'capacitor'  (안드로이드 앱)
그 외                                      → 'web'
```

## 2. 채널별로 갈리는 것

| | 웹 | 미니앱 | 안드로이드 앱 |
|---|---|---|---|
| 인증 | 없음(로컬) | 플랫폼 식별자 → 커스텀 토큰 | Google 로그인 / 익명 폴백 |
| Firestore 컬렉션 | — | 별도 컬렉션 | 별도 컬렉션 |
| 광고 | 없음 | 플랫폼 광고 | AdMob |
| 1대1 대전 | 차단 | 허용 | 허용 |
| 뒤로가기 | 브라우저 | 플랫폼 시스템 내비 | Android 백버튼 전용 경로 |

게임 로직은 세 채널이 완전히 동일하다. 차이는 **인증·수익화·진입 게이트 세 층에만** 있다. 그래서 분기를 그 세 층에 가두고 나머지를 공유하는 쪽이 싸다. 대신 분기점마다 "왜 이 채널만"을 주석으로 남기는 규율이 필수다.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/runtime-channel-detection/channel-layers.ko.svg"><img src="/assets/img/posts/runtime-channel-detection/channel-layers.ko.svg" width="720" height="300" loading="lazy" alt="런타임 판정 한 곳: __appsInToss 또는 AIT 전역이면 ait, isNativePlatform()이면 capacitor, 그 외는 web. !!window.Capacitor는 브라우저에서도 전역이 있어 항상 참이라 쓰면 안 된다. 채널별로 갈리는 건 인증, 수익화, 진입 세 층뿐이고 게임 로직은 세 채널이 동일하다."></a></div><figcaption>분기를 세 층에 가두고 나머지는 공유한다.</figcaption></figure>

## 3. `!!window.Capacitor`로 판정하면 안 된다

`@capacitor/core`는 **브라우저에서도 전역을 정의한다.** 존재 여부만 보는 판정은 항상 참이 되고, 웹이 전부 네이티브 앱으로 오판된다.

2026-08-05에 이 사고를 냈다. `web` 분기가 통째로 죽어서 웹에서 스토어 안내가 안 뜨고 네이티브용 스플래시가 잘못 노출됐다. `isNativePlatform()`으로 교체해 고쳤다.

문제는 이게 **재발**이라는 점이다. 나흘 전인 8월 1일에 App Check 초기화에서 이미 같은 함정을 밟았다. 그때는 증상만 고치고 판정 방식을 안 바꿨다. 지금은 판정 함수 주석에 두 사고의 기록이 같이 들어가 있다.

교훈은 하나다. **전역의 존재는 환경을 증명하지 않는다.** 라이브러리가 자기 전역을 어느 환경에 정의하는지는 라이브러리 사정이고, 그걸 물어보라고 만든 API가 따로 있다.

## 4. 분기를 추가할 때 봐야 하는 세 곳

채널 조건을 하나 추가하면 **표시 조건·텍스트·핸들러** 셋을 동시에 고쳐야 한다.

| 빠뜨린 것 | 증상 |
|---|---|
| 표시 조건(`&&` 조건절) | 기능이 통째로 사라진다 |
| 텍스트 | 안 되는 이유를 엉뚱하게 안내한다 |
| 핸들러 | 버튼은 멀쩡한데 눌러야 막힌다 |

세 번째가 제일 오래 산다. 코드 리뷰에서도 화면에서도 안 보이고, 실기기에서 눌러봐야 나온다.

이것과 짝인 반대 사고도 있었다. **코드를 안 건드렸는데 위험해진 항목**이다. 로그아웃 버튼은 몇 주째 그대로였는데, 다른 화면의 노출 조건을 넓히면서 그 버튼까지 가는 경로가 열렸다. 재로그인 수단이 없는 채널에서 로그아웃하면 갇힌다. 노출 조건을 넓히는 변경은 그 화면 안에서만 끝나지 않는다.

## 5. 인증 4겹 장애

미니앱 채널에 정식 인증을 붙일 때, 실기기에서 진입이 안 됐다. 첫 조치는 헛발이었고, 그 뒤로는 한 겹을 벗길 때마다 그 에러가 사라지고 다음 에러가 나왔다.

| 순서 | 조치 | 결과 |
|:-:|---|---|
| 1 | 인증 서비스 콘솔에 서빙 도메인 추가 | **변화 없음** — 같은 에러 지속 |
| 2 | 검증 옵션 조정 | 401 소멸, 다음 에러 등장 |
| 3 | 서명 IAM 권한 부여 | 500 소멸, 다음 에러 등장 |
| 4 | **API 키의 리퍼러 허용 목록에 서빙 도메인 추가** | 해결 |

1번과 4번은 둘 다 "도메인을 등록한다"인데 등록하는 곳이 다르다. 1번은 인증 서비스 콘솔이고 4번은 API 키 자체의 제한 설정이다. 1번을 하고 "도메인은 처리했다"고 넘긴 것이 조사를 길게 만들었다.

4번이 근본 원인이었고, 증상이 비대칭이라 더 어려웠다. **Firebase browser key의 리퍼러 제한은 Auth와 App Check에는 강제되고 Firestore·RTDB에는 강제되지 않는다.** 그래서 "DB는 멀쩡히 읽고 쓰는데 로그인만 403"이라는 그림이 나온다. 인증 코드를 아무리 봐도 안 나오는 종류의 결함이다.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/runtime-channel-detection/auth-layers.ko.svg"><img src="/assets/img/posts/runtime-channel-detection/auth-layers.ko.svg" width="720" height="280" loading="lazy" alt="1 인증 서비스 콘솔에 도메인 추가: 변화 없음. 2 검증 옵션 조정: 401 소멸, 다음 에러. 3 서명 IAM 권한: 500 소멸, 다음 에러. 4 API 키 리퍼러 허용 목록에 도메인 추가: 해결. 같은 브라우저 키라도 리퍼러 제한은 Auth와 App Check엔 강제되고 Firestore와 RTDB엔 강제되지 않는다."></a></div><figcaption>1번과 4번은 둘 다 “도메인 등록”이지만 등록하는 곳이 달랐다.</figcaption></figure>

이 트랙 전체는 App Check 토큰 발급이 계속 실패하던 상태에서 시작했다. 원인이 3건이었다.

1. 네이티브 초기화가 아예 빠져 있었다
2. 채널 오판정 분기(§3)를 타서 잘못된 provider로 갔다
3. 도메인이 등록돼 있지 않았다

셋 다 각각으로는 사소하고, 겹쳐 있으니 "발급이 안 된다"는 증상 하나로만 보였다.

## 6. 빌드 파이프라인도 채널마다 다르다

```
웹        npm run build        → dist/
미니앱    npm run build:ait    → .ait 번들
안드로이드 npm run build → cap sync android → bundleRelease
```

SDK 메이저 전환 때 공식 마이그레이션 도구를 돌렸더니, 그 도구가 **`package.json`의 `build` 스크립트를 말없이 덮어썼다.** 웹 배포 설정이 `npm run build`를 쓰고 있었으므로 그대로 뒀으면 웹 배포가 통째로 깨진다.

잡을 수 있었던 건 착수 전에 배포 설정을 읽어두고 "이 도구가 이 줄을 건드릴 것"이라고 계획서에 적어뒀기 때문이다. 자동 변환 도구를 돌리기 전에 **그 도구가 쓸 수 있는 파일을 먼저 목록화**하는 게 값싼 방어다.

## 정리

- 채널 판정은 한 곳에 있어야 하고, 그 한 곳은 전역 존재 검사가 아니라 런타임이 제공하는 질의 API를 써야 한다.
- 채널 분기를 추가하면 표시 조건·텍스트·핸들러 세 곳을 전수 확인한다. 하나만 빠져도 증상이 다르다.
- 증상이 비대칭이면 코드가 아니라 그 위 계층(키 제한·도메인·권한)을 의심한다. 같은 자격증명이 서비스마다 다르게 강제될 수 있다.

---

## English version

The same game ships to three places: web, a mini-app platform, and an Android app. Rather than maintaining three builds, **one codebase determines at runtime where it is running** and adjusts behavior.

What follows is a set of defects that came out of that structure. When the detection function is wrong by one expression, an entire channel dies.

### Detection

```
window.__appsInToss or 'AIT' in window  → 'ait'        (mini-app)
isNativePlatform()                       → 'capacitor'  (Android app)
otherwise                                → 'web'
```

### What actually differs

| | Web | Mini-app | Android app |
|---|---|---|---|
| Auth | none (local) | platform identifier → custom token | Google sign-in / anonymous fallback |
| Firestore collection | — | separate | separate |
| Ads | none | platform ads | AdMob |
| 1v1 versus | blocked | allowed | allowed |
| Back navigation | browser | platform system nav | Android back button path |

Game logic is identical across all three. The differences live in exactly three layers — auth, monetization, entry gating — so confining branches to those layers and sharing everything else is the cheaper structure. The cost is a discipline: every branch point carries a comment explaining why this channel only.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/runtime-channel-detection/channel-layers.en.svg"><img src="/assets/img/posts/runtime-channel-detection/channel-layers.en.svg" width="720" height="300" loading="lazy" alt="One runtime check: __appsInToss or an AIT global means ait, isNativePlatform() means capacitor, otherwise web. !!window.Capacitor must not be used — the global exists in browsers too. Only three layers differ per channel — auth, monetization, entry — and game logic is identical."></a></div><figcaption>Branches are confined to three layers; everything else is shared.</figcaption></figure>

### Don't detect with `!!window.Capacitor`

`@capacitor/core` **defines its global in browsers too.** A presence check is therefore always true, and web gets classified as native every time.

It was found and fixed on 2026-08-05. The entire `web` branch had died: store guidance never rendered on the web build, and a native-only splash appeared where it shouldn't. The fix was `isNativePlatform()`.

The uncomfortable part is that it was a **repeat.** Four days earlier, App Check initialization hit the same trap. That time the symptom got fixed and the detection style didn't. The detection function now carries both incidents in its header comment.

One rule came out of it: **the presence of a global does not prove an environment.** Which environments a library defines its global in is the library's business, and there is usually a dedicated API for asking the actual question.

### Three places to change per branch

Adding one channel condition means changing **the visibility condition, the copy, and the handler** together.

| Missed | Symptom |
|---|---|
| Visibility condition (the `&&` clause) | The feature disappears entirely |
| Copy | It explains the wrong reason for being unavailable |
| Handler | The button looks fine and fails on tap |

The third survives longest. It's invisible in review and invisible on screen; it only shows up when someone taps it on a device.

There's a mirror-image failure worth pairing with it: **an item that became dangerous without its code being touched.** A sign-out button sat unchanged for weeks, then a separate change widened the visibility condition of another screen and opened a path to it. On a channel with no way to sign back in, signing out strands the user. Widening a visibility condition doesn't stay inside that screen.

### A four-layer auth failure

When proper authentication went onto the mini-app channel, entry failed on real devices. The first action was a false start; after that, each layer peeled by **fixing one thing, watching that error disappear, and getting the next error.**

| Step | Action | Result |
|:-:|---|---|
| 1 | Add the serving domain in the auth service console | **No change** — same error |
| 2 | Adjust the verification option | 401 gone, next error appears |
| 3 | Grant signing IAM permission | 500 gone, next error appears |
| 4 | **Add the serving domain to the API key's HTTP referrer allowlist** | Resolved |

Steps 1 and 4 are both "register the domain," but in different places — one is the auth service console, the other is the restriction config on the API key itself. Doing step 1 and mentally checking off "domain handled" is what stretched the investigation.

Step 4 was the root cause, and the asymmetry made it harder. **A Firebase browser key's referrer restriction is enforced for Auth and App Check but not for Firestore or RTDB.** The resulting picture is "the database reads and writes fine, only sign-in returns 403" — a defect no amount of staring at auth code will surface.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/runtime-channel-detection/auth-layers.en.svg"><img src="/assets/img/posts/runtime-channel-detection/auth-layers.en.svg" width="720" height="280" loading="lazy" alt="1 add domain in the auth console: no change. 2 adjust verification options: 401 gone, next error. 3 grant signing IAM: 500 gone, next error. 4 add domain to the API key referrer list: fixed. With the same browser key, referrer limits apply to Auth and App Check but not to Firestore and RTDB."></a></div><figcaption>Steps 1 and 4 are both “register the domain” — in different places.</figcaption></figure>

The whole track started from App Check token issuance failing continuously. There were three causes:

1. Native initialization was missing entirely
2. The channel misdetection above routed it to the wrong provider
3. The domain was not registered

Each is minor alone. Stacked, they present as a single symptom: nothing is issued.

### Build pipelines differ per channel too

```
web        npm run build        → dist/
mini-app   npm run build:ait    → .ait bundle
Android    npm run build → cap sync android → bundleRelease
```

During a major SDK migration, the official migration tool **silently overwrote the `build` script in `package.json`.** Web deployment was configured to run `npm run build`, so leaving it would have broken web deploys entirely.

It got caught because the deploy config was read before starting and the plan explicitly noted "this tool will touch this line." Listing the files an automated migration tool is allowed to write, before running it, is a cheap defense.

### Takeaways

- Channel detection belongs in one place, and that place should query a runtime API rather than test for the presence of a global.
- Adding a channel branch means auditing three sites: visibility condition, copy, handler. Each omission produces a different symptom.
- When a symptom is asymmetric, suspect the layer above the code — key restrictions, domains, permissions. The same credential can be enforced differently per service.
