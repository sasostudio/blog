---
layout: post
title: "뭉치뭉치 - 런타임 채널 판정"
description: "하나의 코드베이스가 웹·미니앱·안드로이드 중 어디서 도는지 런타임에 판정하는 구조와, 거기서 나온 결함 정리. 전역 존재 검사의 함정, 분기 추가 시 확인할 세 곳, 인증 4겹 장애."
date: 2026-08-28 01:30:00 +0900
project: moongchi
tags: [architecture, mobile, firebase, build-in-public]
---

*🇬🇧 [English version below](#english-version)*

같은 게임을 세 곳에 출시. 웹, 미니앱 플랫폼, 안드로이드 앱. 앱을 세 벌로 나누지 않고 **하나의 코드베이스가 런타임에 자기 실행 위치를 판정해** 동작을 바꾸는 구조.

이 구조에서 나온 결함 몇 건의 정리. 판정 함수 한 줄이 틀리면 채널 하나가 통째로 사망.

## 1. 판정

```
window.__appsInToss 또는 'AIT' in window  → 'ait'        (미니앱)
isNativePlatform()                         → 'capacitor'  (안드로이드 앱)
그 외                                      → 'web'
```

## 2. 채널별 분기 항목

| | 웹 | 미니앱 | 안드로이드 앱 |
|---|---|---|---|
| 인증 | 미사용(로컬) | 플랫폼 식별자 → 커스텀 토큰 | Google 로그인 / 익명 폴백 |
| Firestore 컬렉션 | — | 별도 컬렉션 | 별도 컬렉션 |
| 광고 | 미표시 | 플랫폼 광고 | AdMob |
| 1대1 대전 | 차단 | 허용 | 허용 |
| 뒤로가기 | 브라우저 | 플랫폼 시스템 내비 | Android 백버튼 전용 경로 |

게임 로직은 세 채널 완전 동일. 차이는 **인증·수익화·진입 게이트 세 층에만** 존재. 그래서 분기를 그 세 층에 가두고 나머지를 공유하는 쪽이 저비용. 대신 분기점마다 「왜 이 채널만」을 주석으로 남기는 규율이 필수.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/runtime-channel-detection/channel-layers.ko.svg"><img src="/assets/img/posts/runtime-channel-detection/channel-layers.ko.svg" width="720" height="300" loading="lazy" alt="런타임 판정 한 곳: __appsInToss 또는 AIT 전역이면 ait, isNativePlatform()이면 capacitor, 그 외는 web. !!window.Capacitor는 브라우저에서도 전역이 존재해 항상 참이므로 사용 금지. 채널별로 갈리는 것은 인증, 수익화, 진입 세 층뿐, 게임 로직은 세 채널 동일."></a></div><figcaption>분기는 세 층에 한정, 나머지는 공유.</figcaption></figure>

## 3. `!!window.Capacitor` 판정 금지

`@capacitor/core`는 **브라우저에서도 전역을 정의.** 존재 여부만 보는 판정은 항상 참이 되고, 웹 전체가 네이티브 앱으로 오판되는 구조.

2026-08-05, 내 코드에서 이 사고 발생. `web` 분기가 통째로 죽어 웹에서 스토어 안내가 미표시, 네이티브용 스플래시가 잘못 노출. `isNativePlatform()`으로 교체해 수정.

문제는 이것이 **재발**이라는 점. 나흘 전인 8월 1일, App Check 초기화에서 이미 같은 함정을 경험. 그때는 증상만 고치고 판정 방식은 그대로 둔 상태. 지금은 판정 함수 주석에 두 사고의 기록이 함께 존재.

교훈은 하나. **전역의 존재는 환경의 증명이 아님을 명심.** 라이브러리가 자기 전역을 어느 환경에 정의하는지는 라이브러리 사정이고, 그 질문을 위해 만든 API가 따로 존재.

## 4. 분기 추가 시 확인할 세 곳

채널 조건 하나를 추가하면 **표시 조건·텍스트·핸들러** 셋의 동시 수정이 필요.

| 누락 대상 | 증상 |
|---|---|
| 표시 조건(`&&` 조건절) | 기능이 통째로 소멸 |
| 텍스트 | 불가 사유의 엉뚱한 안내 |
| 핸들러 | 버튼은 멀쩡한데 누르면 차단 |

세 번째가 가장 오래 생존하는 유형. 코드 리뷰에서도 화면에서도 보이지 않고, 실기기에서 눌러 봐야 발견.

이와 짝을 이루는 반대 사고도 존재. **코드를 건드리지 않았는데 위험해진 항목.** 로그아웃 버튼은 몇 주째 그대로였는데, 다른 화면의 노출 조건을 넓히면서 그 버튼까지 가는 경로가 개방. 재로그인 수단이 없는 채널에서 로그아웃하면 사용자가 고립되는 상황. 노출 조건 확대의 영향은 그 화면 안에서 끝나지 않는다는 교훈.

## 5. 인증 4겹 장애

미니앱 채널에 정식 인증을 붙일 때 실기기 진입 실패. 첫 조치는 헛발질, 그 뒤로는 한 겹을 벗길 때마다 그 에러가 사라지고 다음 에러가 등장하는 패턴.

| 순서 | 조치 | 결과 |
|:-:|---|---|
| 1 | 인증 서비스 콘솔에 서빙 도메인 추가 | **변화 0** — 같은 에러 지속 |
| 2 | 검증 옵션 조정 | 401 소멸, 다음 에러 등장 |
| 3 | 서명 IAM 권한 부여 | 500 소멸, 다음 에러 등장 |
| 4 | **API 키의 리퍼러 허용 목록에 서빙 도메인 추가** | 해결 |

1번과 4번은 둘 다 「도메인 등록」인데 등록 위치가 상이. 1번은 인증 서비스 콘솔, 4번은 API 키 자체의 제한 설정. 1번을 하고 「도메인 처리 완료」로 넘긴 것이 조사 장기화의 원인.

4번이 근본 원인이었고, 증상의 비대칭이 난도를 상승. **Firebase browser key의 리퍼러 제한은 Auth와 App Check에는 강제, Firestore·RTDB에는 미강제.** 그래서 「DB는 정상적으로 읽고 쓰는데 로그인만 403」이라는 그림이 형성. 인증 코드를 아무리 봐도 나오지 않는 종류의 결함.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/runtime-channel-detection/auth-layers.ko.svg"><img src="/assets/img/posts/runtime-channel-detection/auth-layers.ko.svg" width="720" height="280" loading="lazy" alt="1 인증 서비스 콘솔에 도메인 추가: 변화 0. 2 검증 옵션 조정: 401 소멸, 다음 에러. 3 서명 IAM 권한: 500 소멸, 다음 에러. 4 API 키 리퍼러 허용 목록에 도메인 추가: 해결. 같은 브라우저 키라도 리퍼러 제한은 Auth와 App Check엔 강제, Firestore와 RTDB엔 미강제."></a></div><figcaption>1번과 4번은 둘 다 「도메인 등록」이지만 등록 위치가 상이.</figcaption></figure>

이 트랙 전체의 출발점은 App Check 토큰 발급이 계속 실패하던 상태. 원인은 3건.

1. 네이티브 초기화 자체의 누락
2. 채널 오판정 분기(§3)를 타서 잘못된 provider로 진행
3. 도메인 미등록

셋 다 개별로는 사소하지만, 겹쳐 있으니 「발급 불가」라는 증상 하나로만 관측.

## 6. 채널별 빌드 파이프라인

```
웹        npm run build        → dist/
미니앱    npm run build:ait    → .ait 번들
안드로이드 npm run build → cap sync android → bundleRelease
```

SDK 메이저 전환 때 공식 마이그레이션 도구를 돌리자, 그 도구가 **`package.json`의 `build` 스크립트를 말없이 덮어쓰는 동작.** 웹 배포 설정이 `npm run build`를 쓰고 있었으므로 그대로 뒀다면 웹 배포 전체가 붕괴했을 상황.

잡을 수 있었던 이유는 착수 전에 배포 설정을 읽어 두고 「이 도구가 이 줄을 건드릴 것」이라고 계획서에 기재해 둔 덕분. 자동 변환 도구 실행 전에 **그 도구가 쓸 수 있는 파일의 사전 목록화**가 저비용 방어책.

## 정리

- 채널 판정은 한 곳에 집중, 그 한 곳은 전역 존재 검사가 아니라 런타임이 제공하는 질의 API를 사용.
- 채널 분기 추가 시 표시 조건·텍스트·핸들러 세 곳을 전수 확인. 하나만 빠져도 증상이 상이.
- 증상이 비대칭이면 코드가 아니라 그 위 계층(키 제한·도메인·권한)을 의심. 같은 자격증명도 서비스마다 강제 범위가 상이할 가능성.

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
