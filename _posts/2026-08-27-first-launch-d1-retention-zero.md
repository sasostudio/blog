---
layout: post
title: "수분 코치 - 첫 출시작의 D1 리텐션 0"
description: "심사를 통과해 출시까지 간 수분 코치의 종료 회고. 유입 22명 중 다음 날 재방문 0명. 심사 방어 논리를 제품 가치로 착각, 계측 없는 유입 집행, 사용자 0명 상태에서의 재방문 엔진 선제작이 원인."
date: 2026-08-27
project: hydro-coach
tags: [build-in-public, postmortem, product, apps-in-toss, public-api]
---

*🇬🇧 [English version below](#english-version)*

열 번째 프로젝트 종료. 앞선 아홉 개와 다른 점이 하나. 그것들은 전부 출시 전에 종료, 사용자를 한 명도 만나지 못한 채 「이건 아닌 것 같다」로 마무리. 수분 코치는 심사 통과, 출시, 실사용자 30명과의 만남까지 도달. 그 뒤에 데이터가 앱을 기각.

착수부터 종료까지 31일, 실작업은 20일 안팎. 코드 약 18,585줄, 파일 134개. 토스 미니앱 플랫폼(앱인토스)의 7월 챌린지 출품작.

## 기상청 오픈API 위에 세운 앱

물 마시기 앱은 흔한 장르. 차별화를 위해 목표량을 날씨에서 산출. 오늘 덥고 습하면 목표 상향, 폭염경보가 뜨면 추가 상향.

<figure class="shot"><img src="/assets/img/posts/first-launch-d1-retention-zero/hydro-coach-home.jpg" width="440" height="694" loading="lazy" alt="수분 코치 홈 화면 — 직접 입력 칸, 시간대별 기온 그래프와 폭염 단계, 자외선·습도, 오늘의 물 기록"><figcaption>수분 코치 홈 화면. 시간대별 기온과 폭염 단계가 목표량을 조정.</figcaption></figure>

재료는 전부 공공데이터포털의 기상청 오픈API. `apis.data.go.kr/1360000` 아래 네 종류를 사용.

| 서비스 | 오퍼레이션 | 사용 이유 |
|---|---|---|
| 단기예보 조회서비스 | `getVilageFcst` | 기온·습도 → 체감온도 → 목표량 |
| 단기예보 조회서비스 | `getUltraSrtNcst` | 예보가 아닌 실제 관측값 |
| 기상특보 조회서비스 | `getWthrWrnList` | 폭염주의보·경보 |
| 생활기상지수 조회서비스 V5 | `getUVIdxV5` | 자외선 지수 |

**미사용 API가 더 중요했다는 점.** 체감온도지수(`getSenTaIdxV4`)는 2026년 5월 1일자로 종료. 그래서 기온과 습도만으로 체감온도를 직접 산출. 외부 지수를 하나 더 받는 대신 계산식을 코드에 넣은 셈인데, 결과적으로 이쪽이 유리. 호출 하나 감소, 종료 공지에 흔들릴 의존성도 하나 감소.

호출은 앱에서 직접 하지 않고 Cloudflare Worker 프록시를 거쳐 중계. 이유는 두 가지.

- **API 키 은닉.** 키는 Worker Secret에만 존재, 앱 번들에는 미포함.
- **위치정보 미수집.** 앱의 GPS 사용 0. 사용자가 고른 지역의 격자 좌표(nx/ny)와 지점 코드만 전송, 프록시는 위경도 자체를 미인지. 시/도·시군구 256건을 격자·지점 코드로 사전 매핑.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/first-launch-d1-retention-zero/architecture.ko.svg"><img src="/assets/img/posts/first-launch-d1-retention-zero/architecture.ko.svg" width="720" height="230" loading="lazy" alt="앱은 GPS 미사용, 고른 지역의 격자 좌표와 지점 코드만 전송. Cloudflare Worker는 API 키를 Secret에만 보관, 격자와 발표 시각 슬롯을 키로 1시간 캐시, 기상청 API 4종 호출."></a></div><figcaption>키는 앱 번들에 미포함, 위경도는 어디에도 부재.</figcaption></figure>

캐시는 KV에 1시간, 키를 「격자 좌표 + 발표 시각 슬롯」으로 잡아 전 사용자가 공유하는 구조. 기상청 쿼터 보호가 목적이었는데, 나중에 이 구조가 비용 구조의 첫 병목으로 부상. 그 이야기는 [2편](/2026/08/27/false-pass-signals/)에서 정리.

## 이미 출시된 버전의 폭염 배너 사망

두 번째 버전의 심사 제출 직전, 잠복 결함 하나 발견.

특보 통보문 파싱에서 `발표`와 `해제`만 읽던 상태. 그런데 기상청은 등급 변경 시 `변경`으로 통보. 폭염주의보가 경보로 격상되면 통보문 종류가 `변경`이고, 우리 코드는 그걸 못 읽어 **그 시점부터 특보를 「없음」으로 판정**.

파장은 알림에 그치지 않는 수준. 같은 판정 결과를 앱 화면의 폭염 배너와 목표량 상향이 함께 사용하는 구조. 즉 **이미 출시된 버전에서, 폭염경보가 떠도 배너 미표시·목표량 미상향.** 앱의 정체성에 해당하는 기능이 조용히 죽어 있던 상황.

수정 방향은 목록 확장이 아니라 판정 반전. `발표`·`변경`을 나열하는 방식은 기상청이 새 표기를 하나 더 쓰는 순간 다시 구멍. 그래서 **「해제가 아니면 유효」**로 변경하고, 실제로 받은 통보문 원문을 테스트에 고정.

## 실측: 22명 중 다음 날 재방문 0명

8월 초 유입 광고 집행.

| 단계 | 수치 | 판정 |
|---|---|---|
| 도달 | 5,201건 | 정상 |
| 클릭 | 27건 (CTR 0.52%) | 업계 정상 범위 |
| 신규 유입 | 16명 (8/8~8/11) | 광고의 역할은 완수 |
| **재방문** | **0명** | 🔴 |

코호트로 추적한 22명 전원이 D1 미복귀. 건강·습관 앱의 D1 벤치마크는 20~30%.

표본이 작아서가 아니냐는 반문도 가능. 계산 결과, **실제 D1이 10%라고 가정해도 22명 전원 미복귀 확률은 9.8%.** 이 결과가 우연이려면 열 번에 한 번 나오는 일이 하필 일어나야 하는 상황. 우리 D1은 10% 미만으로 보는 것이 타당.

꾸준히 들어오는 사용자는 딱 한 명, 정황상 제작자 본인.

**마케팅 문제와는 무관. 0에 5,000을 곱해도 0.**

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/first-launch-d1-retention-zero/funnel.ko.svg"><img src="/assets/img/posts/first-launch-d1-retention-zero/funnel.ko.svg" width="720" height="270" loading="lazy" alt="도달 5,201, 클릭 27(CTR 0.52%), 신규 유입 16, 다음 날 재방문 0. 막대 길이는 비율이 아니라 단계 표시."></a></div><figcaption>광고는 역할 완수. 다음 날 재방문자는 0명.</figcaption></figure>

## 헛돈 이유

종료하며 규명한 것 중 가장 값진 결과는 실패 원인. 세 가지, 그중 첫 번째는 처음 보는 패턴.

### 1. 심사 방어 논리를 제품 가치로 착각

「날씨로 목표량을 계산한다」가 이 앱의 차별점이라는 믿음. 그런데 그 논리는 **원래 심사용.** 물 마시기 앱으로만 내면 단순 리워드성 미니앱으로 읽힐까 봐, 「고정값 2L가 아니라 기상 데이터 기반 산출」이라는 근거를 준비한 것. 현황서에도 그 맥락으로 「계산 알고리즘이 정체성」이라는 기재.

같은 문장이 두 사람에게 전혀 다르게 읽히는 구조.

| 읽는 사람 | 반응 |
|---|---|
| 심사관 | 「고정값이 아니라 기상 데이터 기반 산출」 → 통과 |
| 사용자 | 「오늘 목표가 1,900ml네」 → **그래서 뭐** |

체감온도가 목표를 1,800에서 1,900으로 바꾼다는 사실은 앱을 매일 열 이유로 불충분. **심사를 통과시키는 논리와 사람이 앱을 여는 이유는 별개.**

체크리스트에 항목 하나 추가. *「이 차별점은 규정 대응용인가, 사용자 가치인가?」* 규정 대응용이라면 사용자가 매일 여는 이유를 별도로 제시 필요.

### 2. 계측 없이 유입부터 집행

온보딩 완료·물 기록·목표 달성, 세 이벤트 계측은 끝까지 코드에 미반영. 현황서에 미완료 표시로 계속 남은 상태에서 광고 5,201건을 전부 소진.

| 알고 싶은 것 | 확인 가능 여부 |
|---|---|
| 지역 선택 3단계 온보딩에서 이탈했나 | ❌ |
| 물 한 번 기록하고 「이게 다야?」 했나 | ❌ |
| 기록은 했는데 다음 날 잊었나 | ❌ |

세 경우의 처방은 완전히 상이. 각각 온보딩 축소, 가치 제시 재설계, 알림 옵트인. 그런데 **22명의 이탈 지점은 이제 영원히 확인 불가.** 유입 실험 한 번을 통째로 날린 셈.

원칙으로 정리. **유입 집행 전에 깔때기 계측 우선.** 계측 없는 유입은 데이터가 아니라 소음.

### 3. 사용자 0명 상태에서 재방문 엔진 3주 제작

개인화 푸시에 3주 투입. mTLS 클라이언트 인증, D1, cron 판정, 발송 배치, fail-closed 설계, 법적 문서 게시까지 전부 구현.

**완성 시점의 알림 활성 사용자는 1명.** 그마저도 정황상 제작자 본인.

22명이 앱에 들어왔지만 알림 활성은 0명. 재방문을 만드는 유일한 훅이 사실상 아무에게도 연결되지 않은 채 완성된 상황.

직전 프로젝트에서 이미 겪은 패턴. 「끝장」에서 사용자가 없는 채로 멀티플레이를 구현해 실제 대전이 불가능했던 것과 동일. 형태만 멀티플레이에서 푸시 엔진으로 변경. 푸시 엔진 3주 대신 **옵트인 전환율 측정 3시간**이 먼저였어야 했다는 반성.

여기에 시한 요인 하나 추가. 체감온도와 폭염특보가 목표를 흔드는 것이 이 앱의 정체성인데, 9월이면 그 변수가 소멸. 이름만 계절 중립으로 바꿨을 뿐 기능은 여름 그대로였고, **7월 중순 착수·8월 출시라 검증 창이 구조적으로 4주뿐.**

## 잘 만든 것과 내 실수

fail-closed 설계는 만족스러운 부분. 「켠 적 없는 사용자에게는 미발송」을 서버가 강제하도록 설계했는데, 이 동작을 **라이브 로그에서 실제로 관측.** 옵트인 행이 DB에 있는데도 발송 후보가 0건으로 찍히는 순간이 있었고, 그게 설계대로의 동작. 코드가 그렇게 돼 있다는 것과 운영 중 그렇게 동작하는 것을 봤다는 것은 무게가 별개.

반대로 사고 하나도 기록. 이 프로젝트는 코딩 에이전트와 짝을 이뤄 개발했는데, 설정 추가 여부를 묻고 대기하던 상황에서 에이전트가 **실행할 명령 블록을 먼저 보여 주고 질문을 그 뒤에 첨부.** 나는 이미 반영된 것으로 읽고 배포를 실행. 다행히 같은 코드의 재배포라 사고는 없었지만 목적 기능은 미적용.

교훈: **실행 명령을 제시할 거라면 변경을 먼저 적용하거나, 적용 전이라는 사실을 명령 블록 안에 명시.** 명령 블록은 그 자체로 「준비 완료」 신호로 읽히는 물건. 사람이든 에이전트든 동일.

## 삭제가 아니라 동결

종료 방식은 동결로 결정. 앱은 라이브에 그대로 두고 개발만 중단.

이유는 두 가지. 유지비가 실제로 월 0원이라 내릴 이유가 비용이 아니라는 점. 그리고 심사 통과부터 출시까지 뚫어 둔 경로 자체가 다음 프로젝트를 며칠 단축시키는 자산이라는 점. 재사용 가능한 것 13종을 목록으로 정리. mTLS 푸시 서버, 기상청 4종 프록시, 체감온도 자체 산출 로직, 로그인 없이 익명 키로 사용자를 식별하는 노선, 법적 문서 3종과 게시 라우트 등.

다만 **동결해도 사라지지 않는 의무가 하나.** 개인정보 처리방침을 실제로 게시했고 거기에 파기 조항을 기재. 앱을 내리지 않아도 그 약속은 유효. 기한을 문서에 명시. 게시한 고지는 개발 중단과 함께 멈추지 않는 성격.

## 남은 것

기술적으로는 성공, 제품으로는 실패. 열 번째도 같은 결론.

다만 이번엔 처음으로 **실사용자 데이터가 그 결론을 결정.** 유통 채널 개방, 정책 리스크 통과, 외부 API 연동까지 완료된 상태에서의 실패. 즉 남은 병목은 **만드는 능력이 아니라 무엇을 만들지 고르는 단계**라는 의미.

수분 코치는 애초에 걸렸어야 할 질문에서 탈락. *「이 문제 때문에 지난주에 실제로 불편했는가?」* 물을 안 마셔서 불편했던 경험은 부재. 안 마셔도 당장 아프지 않고, 정말 필요할 땐 이미 목이 마른 상태. 앱이 없애 주는 불편이 앱을 여는 수고보다 작았다는 결론.

다음 프로젝트의 과제는 더 잘 만들기가 아니라 그 질문의 실제 통과.

---

## English version

I shut down my tenth project. This one differed from the previous nine in a single way: those all died before launch, ending on a vague "this isn't it" without ever meeting a user. Hydro Coach passed review, shipped, and reached 30 real users. Then the data rejected it.

Thirty-one days start to finish, about twenty of them actual work. About 18,585 lines across 134 files. It was my entry for the July challenge on Apps in Toss, Toss's mini-app platform.

### Built on Korea's public weather API

Water-tracking apps are a dime a dozen. To make one that wasn't, I derived the daily target from the weather. Hot and humid today, the target goes up. Heat advisory in effect, it goes up again.

<figure class="shot"><img src="/assets/img/posts/first-launch-d1-retention-zero/hydro-coach-home.jpg" width="440" height="694" loading="lazy" alt="Hydro Coach home screen — a manual entry field, an hourly temperature chart with heat levels, UV and humidity, and today’s water log"><figcaption>Hydro Coach’s home screen. Hourly temperature and heat levels move the target.</figcaption></figure>

Every input came from the Korea Meteorological Administration's open API, served through the national open-data portal at `apis.data.go.kr/1360000`. Four endpoints:

| Service | Operation | Why |
|---|---|---|
| Short-term forecast | `getVilageFcst` | temp + humidity → apparent temp → target |
| Short-term forecast | `getUltraSrtNcst` | actual observations, not forecasts |
| Weather warnings | `getWthrWrnList` | heat advisories and warnings |
| Living weather index V5 | `getUVIdxV5` | UV index |

**The endpoint I didn't use mattered more.** The apparent-temperature index (`getSenTaIdxV4`) was retired on May 1, 2026. So I computed apparent temperature myself from temperature and humidity alone. Trading one more remote call for a formula in my own code turned out to be the better deal: one fewer request, and one fewer dependency that a deprecation notice can knock over.

The app never calls the API directly. A Cloudflare Worker proxy sits in between, for two reasons:

- **Key concealment.** The API key lives only in a Worker Secret and never appears in the app bundle.
- **No location collection.** The app doesn't touch GPS. It sends only the grid coordinates (nx/ny) and station code for a region the user picked from a list, and the proxy never sees latitude or longitude at all. I pre-mapped 256 administrative regions to their grid and station codes.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/first-launch-d1-retention-zero/architecture.en.svg"><img src="/assets/img/posts/first-launch-d1-retention-zero/architecture.en.svg" width="720" height="230" loading="lazy" alt="The app uses no GPS and sends only the chosen region’s grid coordinates and station code. A Cloudflare Worker keeps the API key in Secrets, caches by grid and issue-time slot for an hour, and calls four weather APIs."></a></div><figcaption>The key never ships in the app bundle, and latitude/longitude never exist anywhere.</figcaption></figure>

Responses cache in KV for an hour, keyed by grid coordinates plus the forecast's publication slot, so every user shares the same entry. That was there to respect the KMA's quota. It later turned out to be the first bottleneck in the cost structure — [part two](/2026/08/27/false-pass-signals/) covers that.

### The heat banner was dead in a version already shipped

Right before submitting the second version for review, I found a latent defect.

My warning parser read only `발표` (issued) and `해제` (lifted). But the KMA sends `변경` (changed) when a warning is upgraded or downgraded. When a heat advisory escalates to a heat warning, the notice type is `변경` — which my code couldn't read, so **from that moment on it judged there to be no warning at all.**

The blast radius went past notifications. The same judgment fed the in-app heat banner and the target increase. Which means **in a version that was already live, a heat warning could be in effect while the banner stayed hidden and the target never rose.** The feature the whole app was built around was silently dead.

Fixing it, I inverted the test rather than extending the list. Enumerating `발표` and `변경` breaks again the moment the KMA introduces one more label. So it became **"valid unless lifted,"** with the real notice text I'd received pinned into a test.

### The measurement: zero of 22 came back the next day

I ran an acquisition campaign in early August.

| Stage | Number | Verdict |
|---|---|---|
| Impressions | 5,201 | normal |
| Clicks | 27 (0.52% CTR) | within industry range |
| New users | 16 (Aug 8–11) | the ads did their job |
| **Returned** | **0** | 🔴 |

All 22 users in the tracked cohort failed to return on D1. The D1 benchmark for health and habit apps is 20–30%.

Small sample, you might object. I ran the numbers. **Even assuming a true D1 of 10%, the chance that all 22 fail to return is 9.8%.** For this result to be luck, a one-in-ten event would have to have landed on me. The honest read is that my D1 is under 10%.

Exactly one user kept coming back, and circumstantially that user is me.

**This is not a marketing problem. Multiply zero by five thousand and you still have zero.**

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/first-launch-d1-retention-zero/funnel.en.svg"><img src="/assets/img/posts/first-launch-d1-retention-zero/funnel.en.svg" width="720" height="270" loading="lazy" alt="Reach 5,201, clicks 27 (CTR 0.52%), new users 16, back the next day 0. Bar lengths mark stages, not ratios."></a></div><figcaption>The ads did their job. Nobody came back the next day.</figcaption></figure>

### Three reasons it went nowhere

The most valuable thing I got out of shutting it down was the diagnosis. Three causes, and the first one was new to me.

**1. I mistook a review-defense argument for product value.** I believed "the target is calculated from weather" was the app's differentiator. That argument was **built for the reviewers.** I was worried a plain water tracker would read as a trivial reward app, so I prepared grounds to say "not a flat 2L, but derived from meteorological data." My own status doc says "the calculation algorithm is the identity" — in exactly that context.

The same sentence lands completely differently depending on who reads it:

| Reader | Reaction |
|---|---|
| Reviewer | "derived from weather data, not a constant" → approved |
| User | "today's target is 1,900ml" → **so what** |

That apparent temperature nudged the goal from 1,800 to 1,900 is not a reason to open an app every day. **The argument that gets you through review and the reason a person opens your app are different objects.** New checklist item: *is this differentiator serving a regulation, or serving a user?* If it's the regulation, you still owe a separate answer for the daily open.

**2. I burned traffic before instrumenting the funnel.** Three events — onboarding completed, water logged, goal reached — never made it into the code. They sat marked incomplete in the status doc the whole time, and I spent all 5,201 impressions in that state.

| What I want to know | Can I? |
|---|---|
| Did they drop off in the three-step region picker? | ❌ |
| Did they log water once and think "that's it?" | ❌ |
| Did they log it and forget by the next day? | ❌ |

Those three call for completely different fixes: shorten onboarding, redesign the value proposition, push the notification opt-in. **I will now never know where those 22 people leaked out.** An entire acquisition experiment, wasted. The rule I wrote down: instrument the funnel before you buy the traffic. Traffic without instrumentation isn't data, it's noise.

**3. I spent three weeks on a re-engagement engine with zero users.** Personalized push took three weeks: mTLS client auth, D1, cron-based eligibility, send batching, fail-closed design, published legal documents. **When it was done, one user had notifications turned on** — again, circumstantially me.

Twenty-two people came into the app and none of them enabled notifications. The only hook capable of producing a return visit shipped connected to essentially nobody. This is the same pattern as my previous project, where I built multiplayer with no user base and no real match was ever possible. Only the shape changed, from multiplayer to a push engine. Three hours measuring opt-in conversion should have come before three weeks of engine.

There was a deadline on top of it. Apparent temperature and heat warnings moving the target *is* the app's identity, and in September those variables disappear. I'd renamed it to something season-neutral, but the features were still pure summer — starting mid-July and shipping in August left a validation window structurally capped at about four weeks.

### One thing I got right, one I got wrong

The fail-closed design held up. "Never send to a user who hasn't opted in" is enforced server-side, and I **watched it work in live logs** — there were moments with opt-in rows sitting in the database and zero send candidates, exactly as designed. Knowing the code says so and having seen it behave that way in production are different weights of knowledge.

A near-miss also went on the record. I built this project paired with a coding agent, and while it was waiting on my decision about adding a config value, it **showed the command block first and put the question after it.** I read that as already applied and ran the deploy. It happened to be a redeploy of identical code so nothing broke, but the intended feature never turned on. The lesson: if you're going to hand someone a command to run, either apply the change first or state inside the block that it isn't applied yet. A command block reads as "ready" all by itself — to people and agents alike.

### Frozen, not deleted

I chose to freeze rather than take it down. The app stays live; only development stops.

Two reasons. Running cost is genuinely $0/month, so cost isn't a reason to pull it. And the path I cut from review submission through to launch is itself an asset that shortens the next project by days. I catalogued 13 reusable pieces: the mTLS push server, the four-endpoint KMA proxy, the apparent-temperature calculation, the anonymous-key approach that identifies users without a login, three legal documents and the routes that serve them.

But **one obligation survives the freeze.** I actually published a privacy policy, and it contains a data-destruction clause. That promise holds whether or not the app comes down, so the deadline is pinned in the docs. A published notice doesn't stop when development does.

### What's left

Technically a success, a failure as a product. Same conclusion for the tenth time.

The difference is that this time **real user data delivered it.** The distribution channel was open, the policy risk was cleared, the external APIs were all wired up. Failing from *that* position means the remaining bottleneck isn't the ability to build — it's the step where you choose what to build.

Hydro Coach failed the question it should have failed at the start: *did this problem actually inconvenience me last week?* Not drinking water has never inconvenienced me. It doesn't hurt right away, and by the time it matters I'm already thirsty. The friction the app removed was smaller than the friction of opening it.

The next project's job isn't to build better. It's to actually pass that question.
