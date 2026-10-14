---
layout: post
title: "프로젝트 9개 실패의 교훈"
description: "앱 9개를 만들고 9개를 전부 정리한 회고. 기술적 실패는 0건, 전부 제품으로서의 실패. 가짜 보상, 확인하지 않은 외부 API, 닫힌 유통 채널에서 뽑은 「코드 전 체크리스트」 정리."
date: 2026-07-07
project: nine-postmortem
pinned: true
tags: [build-in-public, postmortem, product, indie-dev]
---

*🇬🇧 [English version below](#english-version)*

작년부터 올해까지 앱 9개를 만들고 9개를 전부 정리. 누적 ~19.5일, 약 37,886줄, 종료율 100%. 그 무덤을 한 번에 파헤친 회고.

빠르게 만들 수 있게 된 만큼 빠르게 정리하는 법의 학습도 필요. AI로 하루면 동작하는 앱이 완성되는 시대. 그래서 더더욱, **「만들 수 있는가」는 더 이상 질문이 아니라는 결론. 진짜 질문은 「만들어야 하는가」.** 9개 중 기술적 실패는 0건. 전부 제품으로서의 실패.

## 기술은 전부 성공, 제품은 전부 실패

FitQuest는 10,150줄, 파일 78개 규모의 운동 RPG. 완성·동작까지 문제없었지만 가치는 0. 버튼을 누르면 「운동 완료」로 처리되는 게임인데, 속이는 대상이 결국 자기 자신이라 존재 이유 소멸.

이 사례가 9개 전체의 축소판. 코드는 정상 동작. 문제는 코드 바깥. 그래서 이 글의 교훈은 전부 **코드 한 줄 쓰기 전** 단계의 이야기.

## 실패 지도 — 같은 함정의 반복

9개를 한 표에 늘어놓자 반복되는 실패 패턴이 선명:

| 실패 모드 | 해당 프로젝트 |
|-----------|---------------|
| 만든 사람도 미사용 | CVForgeAI, FitQuest, Risk Roll, 충동구매, 소비투자, 투자감정일기, 이터널리턴 |
| 대체재 존재 (메모장·엑셀·기존 앱) | 대부분 |
| 경쟁 분석이 착수 이후 | CVForgeAI, FitQuest, 피싱가드 등 |
| 가짜 보상 / 외부 재료 문제 | Risk Roll, 피싱가드, 이터널리턴 |
| 유통 채널 미확보 상태로 개발 | 끝장 |
| **근본: 애초에 풀고 싶은 문제가 아니었다는 점** | **9개 전부** |

핵심은 맨 아랫줄. 9개 모두 **「있으면 좋겠다」** 수준이었고, **「없으면 못 살겠다」** 수준은 전무. 나머지 실패 모드는 전부 이 한 줄의 파생.

## 가짜 보상 삼형제

같은 원인으로 종료된 세 프로젝트. 원인은 가짜 보상.

- **Risk Roll** (크래시 도박 게임, 1,600줄): 가상 코인 베팅 구조. 잃어도 아까움 0. 도박의 핵심은 긴장감인데, 가짜 돈은 긴장감을 0으로 만드는 요인.
- **FitQuest**: 앞서 말한 그 게임. 검증 없는 명예 시스템. 클릭=운동.
- **충동구매30일룰** (1,287줄): 충동구매 방지 앱. 그런데 메모장에 "30일 뒤에 다시 생각"이라고 적으면 그걸로 해결. 앱 자체가 불필요.

교훈: **핵심 동기가 가짜면 리텐션도 0.** 가상 코인, 심리적 숫자, 자기기만 — 전부 리텐션 형성 실패.

## 가정은 버그 — 존재하지 않는 API를 3일간 개발한 사례

가장 비싼 교훈의 출처는 피싱가드(보이스피싱 탐지 미니앱, 1,469줄).

기획서의 기재 내용: "KISA 피싱 신고 DB — 무료 공공데이터", "경찰청 사이버범죄 DB — 무료 공공데이터". 이름만 보고 전화번호 조회 API가 있을 것이라 가정. 프론트엔드를 91%까지 만들고 서버 착수 단계에서야 확인:

- **KISA 피싱 DB**의 실체는 피싱 **사이트 URL** 데이터. 전화번호와 무관.
- **경찰청 보이스피싱**의 실체는 **연간 통계**(건수·피해액). 개별 번호 조회 불가.

한국에 전화번호 사기 조회용 **무료 공공 API는 부재.** 유일한 대안(에이픽·더치트)은 유료라 0원 원칙과 충돌. 확인에 5분이면 충분했던 문제. data.go.kr에서 「피싱」 검색만 해도 URL 데이터뿐이라는 사실이 바로 확인 가능.

여기에 AI 모델 문제까지 중첩. 기획서의 "Gemini 2.0 Flash 일 1,500건"은 옛 정보. 해당 모델은 퇴역 예정, 대체 모델의 무료 할당량은 일 250건으로 83% 축소.

**가정은 버그, 검증만이 사실.** 피싱가드 이후로 외부 API는 기획서 반영 전에 반드시 실제 호출로 검증.

## 완성도 95%의 함정 — 유통 채널 미확보 상태의 개발

끝장은 끝말잇기 게임(7,346줄, 412K 단어 DB, ~2주). 싱글 5모드, 멀티플레이, AI 대전, 실시간 채팅, 전국 랭킹까지 구현. 완성도 95%, 출시 가능 수준.

그런데 앱인토스 채널 승인이 미확보 상태. 채널 승인 없이는 토스 3,000만 유저에 **접근 자체가 불가.** 핵심 유통 경로가 잠긴 상태에서 완성도 95%를 만든 셈. 높은 완성도가 지속의 근거는 아니라는 판단. 뭉치뭉치 쪽의 성과 가능성이 더 구체적이었고, 기회비용을 따져 중단.

교훈: **유통 채널 확보가 개발보다 먼저.** 채널이 잠겨 있으면 완성도 100%도 출시 불가. 그리고 멀티플레이는 닭과 달걀 문제. 유저가 없으면 실제 대전이 불가하니 기능의 가치조차 검증 불가. 싱글로 먼저 검증한 뒤 멀티에 투자했어야 했다는 반성.

## 지나치게 좁은 시장

문제는 진짜였지만 **시장이 지나치게 좁았던** 세 프로젝트.

- **투자감정일기** (3,661줄): 매매 시점의 감정 기록 앱. 그런데 타깃이 「주식 투자자 중에서도 감정 기록에 관심 있는 사람」 = 극소수. 게다가 매매가 없으면 열 이유가 없어 DAU에 구조적 한계, 투자 관련이라 플랫폼 정책 리스크까지 중첩.
- **이터널리턴 전략도우미** (6,752줄): 게임 루트 편집기. 게임 패치마다 스크래핑 스크립트 30개 재실행이 필요한 구조(데이터 유지보수 부담의 과소평가). 게다가 작은 스튜디오로서 dak.gg·er.op.gg 같은 팀 단위 운영 서비스와 경쟁해야 하는 구조. 게임 도구는 그 게임의 수명에 종속.
- **소비투자판단기** (1,200줄): "커피값이 10년이면 1,800만 원"을 보여 주는 계산기. 그 인사이트는 한 번 전달되면 종료. 매일 바뀌는 변수가 없어 재방문도 0. **인사이트 전달은 콘텐츠(영상·블로그)로 충분. 앱은 반복 도구여야 한다는 결론.**

## 여기서 나온 체크리스트 — 코드 한 줄 전 단계

이 무덤들이 남긴 결과물은 코드 작성 전에 통과해야 하는 질문 목록:

- **"이 문제 때문에 지난주에 실제로 불편했는가?"** — No면 시작 금지.
- **"메모장·엑셀·기존 앱으로 대체 가능한가?"** — Yes면 앱으로 만들 가치 부재.
- **"하루 1회 이상 자연스럽게 열게 되는 트리거가 있는가?"** — No면 리텐션 부재.
- **"핵심 동기가 진짜인가?"** — 가짜 돈, 심리적 숫자면 불가.
- **"매일 바뀌는 변수가 있는가?"** — 없으면 계산기=일회용, 앱이 아니라 웹페이지로 충분.
- **"핵심 재료(외부 API·데이터)가 실존하는가?"** — 이름이 아니라 실제 호출로 확인.
- **"유통 채널이 실제로 열려 있는가?"** — 채널이 잠겨 있으면 완성도 100%도 출시 불가.

0단계(문제 검증) 미통과 시 코드 작성 자체를 거부하는 것이 지금의 원칙.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/nine-killed-projects-postmortem/seven-questions.ko.svg"><img src="/assets/img/posts/nine-killed-projects-postmortem/seven-questions.ko.svg" width="720" height="380" loading="lazy" alt="코드 전에 묻는 7개 질문과, 각 질문에서 시작을 막는 답. 지난주에 실제로 불편했는가(아니오면 중단), 메모장·엑셀·기존 앱으로 대체 가능한가(예면 중단), 매일 여는 트리거, 진짜 동기, 매일 바뀌는 변수, 핵심 재료의 실존, 열린 유통 채널(각각 아니오면 중단)."></a></div><figcaption>일곱 개 중 하나라도 걸리면 코드 작성 금지. 아홉 개 무덤이 남긴 관문.</figcaption></figure>

## 실패가 남긴 것

9개를 전부 정리했지만 빈손은 아니라는 점.

- **재사용 자산 32개** — 끝말잇기 엔진과 412K 단어 DB, Supabase 실시간 멀티 패턴, Canvas 게임 루프, Web Audio 사운드 엔진, 앱인토스 SDK 연동, 외부 API 검증 체크리스트… 다음 프로젝트의 기반 라이브러리.
- **총비용 ~$25.** 9개를 만들고 정리하는 데 든 비용은 커피 몇 잔 값.
- **손절 속도.** 종료 결정의 속도 향상. 4.5일에서 1일까지. (끝장의 2주는 예외. 우유부단이 아니라 의도한 완성도 실험.) 진짜 역량은 빠른 제작이 아니라 **빠른 종료 판단**이라는 결론.
- **프로세스 성숙.** 계획→설계→구현→검증 사이클 정착, 한 프로젝트(투자감정일기)에서는 설계-구현 100% 일치까지 달성.

빠른 개발이 축복이라는 착각. 진짜 축복은 빠른 **검증**. 하루면 만들 수 있으니 방향이 틀렸을 때 잃는 것도 하루뿐. 9개의 무덤은 그 하루들의 합계, 지금 만드는 것들의 토대.

---

## English version

From last year into this one, I built nine apps and killed all nine. About 19.5 days total, roughly 37,886 lines, a 100% shutdown rate. This is the post where I dig up the whole graveyard at once.

Once you can build fast, you have to learn to kill fast. AI gets you a working app in a day. Which is exactly why — **"can I build it" stopped being the question. The question was "should I."** Not one of the nine failed technically. All nine failed as products.

### The thesis: the tech all succeeded, the products all failed

FitQuest was a 10,150-line, 78-file workout RPG. Finished, working, and worthless. A game where pressing a button counts as "workout complete" — the person you're fooling is yourself, so the reason to exist evaporates.

That's a miniature of all nine. The code ran. The problem lived outside the code. So every lesson here is about the moment **before the first line of code.**

### A map of failure — I kept falling into the same traps

Laying all nine on one table surfaced the recurring failure modes:

| Failure mode | Projects |
|--------------|----------|
| The maker doesn't even use it | CVForgeAI, FitQuest, Risk Roll, and more |
| A substitute already exists (memo app, spreadsheet, existing apps) | most |
| Did competitive analysis *after* starting | CVForgeAI, FitQuest, PhishingGuard, etc. |
| Fake reward / broken external material | Risk Roll, PhishingGuard, Eternal Return tool |
| Built without opening a distribution channel | Kkeutjang |
| **Root: it was never a problem I wanted solved** | **all nine** |

The bottom row is the point. All nine were **"would be nice to have,"** not **"can't live without."** Every other failure mode is a derivative of that one line.

### The three fake-reward brothers

Three projects died of the same disease — the reward was fake.

- **Risk Roll** (a crash gambling game, 1,600 lines): you bet virtual coins. Losing costs nothing. Gambling's heart is tension, and fake money zeroes the tension out.
- **FitQuest**: the game above. An honor system with no verification. Click = workout.
- **Impulse-Buy 30-Day Rule** (1,287 lines): an app to stop impulse buying. But writing "reconsider in 30 days" in a memo app does the whole job. No app needed.

Lesson: **if the core motivation is fake, there's no retention.** Virtual coins, psychological numbers, self-deception — none of them produce retention.

### An assumption is a bug — I spent 3 days building against an API that didn't exist

The most expensive lesson came from PhishingGuard (a voice-phishing detection mini-app, 1,469 lines).

The plan doc said: "KISA phishing-report DB — free public data," "National Police cybercrime DB — free public data." I saw the names and assumed a phone-number lookup API existed. I built the frontend to 91% and only checked at the server stage:

- **KISA's phishing DB** was phishing **site URL** data. Not phone numbers.
- **The police "voice phishing" set** was **annual statistics** (counts, damages). Not per-number lookup.

There was **no free public API in Korea** for scam phone-number lookup. The only alternatives (APICK, TheCheat) were paid, colliding with my zero-cost rule. Checking would have taken five minutes. Search "phishing" on data.go.kr and you immediately see it's URL data.

The AI model piled on too. The plan's "Gemini 2.0 Flash, 1,500/day" was stale info — that model was slated for retirement, and the successor's free quota had been cut 83% to 250/day.

**An assumption is a bug. Only verification is fact.** Since PhishingGuard, every external API gets verified with a real call before it goes in the plan.

### The 95%-done trap — I built without opening the channel

Kkeutjang was a Korean word-chain game (7,346 lines, a 412K-word DB, ~2 weeks). Five single modes, multiplayer, AI matches, live chat, a national leaderboard — 95% done, shippable.

But it had no Apps-in-Toss channel approval. Without that approval, **you cannot reach** Toss's 30M users at all. I built to 95% with the core distribution path locked. High completeness is not a reason to keep going — Moongchi had a more concrete shot at results, so I ran the opportunity-cost math and stopped.

Lesson: **open the distribution channel before you build.** With the channel locked, even 100% done can't ship. And multiplayer is a chicken-and-egg problem — with no users there are no real matches, so you can't even validate the feature's value. I should have validated single-player first, then invested in multi.

### Niche and target — the pool was too small

Three projects had real problems, but the audience was **too narrow.**

- **Investment Emotion Diary** (3,661 lines): log your emotions when you trade. But the target — "stock investors who also care about logging emotions" — is a tiny sliver. You have no reason to open it unless you're trading, which caps DAU structurally, and being investment-adjacent added platform-policy risk.
- **Eternal Return Strategy Helper** (6,752 lines): a game-route editor. Every game patch meant re-running 30 scraping scripts — I underestimated the data-maintenance cost — and I'd be competing as a small studio against team-run services like dak.gg and er.op.gg. A game tool is bound to that game's lifespan.
- **Spend-vs-Invest Judge** (1,200 lines): a calculator showing "your coffee money over 10 years → $13K." That insight lands once and it's over. No daily-changing variable means no return visits. **Insight delivery belongs in content (video, blog). An app has to be a repeat tool.**

### The checklist that came out of it — before the first line

What the graveyard left me is a list of questions to pass before writing code:

- **"Did this problem actually inconvenience me last week?"** — If no, don't start.
- **"Can a memo app, spreadsheet, or existing app replace it?"** — If yes, it's not worth an app.
- **"Is there a trigger to naturally open it once a day or more?"** — If no, there's no retention.
- **"Is the core motivation real?"** — Not fake money, not a psychological number.
- **"Is there a daily-changing variable?"** — If not, it's a one-shot calculator; a web page beats an app.
- **"Does the core material (external API/data) actually exist?"** — Confirm with a real call, not the name.
- **"Is the distribution channel actually open?"** — If it's locked, even 100% done can't ship.

If it fails step 0 (problem validation), I now refuse to write the code.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/nine-killed-projects-postmortem/seven-questions.en.svg"><img src="/assets/img/posts/nine-killed-projects-postmortem/seven-questions.en.svg" width="720" height="380" loading="lazy" alt="Seven questions asked before any code, and the answer that stops the project: did it bother me last week (no stops it), could existing tools do it (yes stops it), a daily trigger, a real motivation, something that changes daily, inputs that exist, an open channel (no stops each)."></a></div><figcaption>Fail any one of the seven and no code gets written. The gate the nine graves left behind.</figcaption></figure>

### The dividend of failure

I killed all nine, but the graveyard wasn't empty-handed.

- **32 reusable assets** — the word-chain engine and 412K-word DB, a Supabase realtime-multiplayer pattern, a Canvas game loop, a Web Audio sound engine, Apps-in-Toss SDK integration, an external-API verification checklist… libraries the next project stands on.
- **~$25 total.** Building and killing nine apps cost a few cups of coffee.
- **Kill velocity.** The kill decisions got faster — from 4.5 days down to 1. (Kkeutjang's 2 weeks is the exception: not indecision, a deliberate completeness experiment.) The real skill wasn't building fast — it was **killing fast.**
- **Process maturity.** A plan→design→build→verify cycle settled in, and on one project (Investment Emotion Diary) it reached 100% design-implementation match.

I thought fast development was the blessing. The real blessing was fast **validation.** If you can build it in a day, a wrong direction costs you only a day. These nine graves are the sum of those days — and what I'm building now stands on top of them.
