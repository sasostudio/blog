---
layout: post
title: "네 번째 에이전트 - researcher"
description: "AgentRoom v1.1.0에 조사 에이전트 researcher를 추가하면서, 잘 만드는 법보다 언제 부르지 않을지를 먼저 정했다. 기본은 꺼짐이고, 부르면 독립 컨텍스트에서 마음껏 조사한 뒤 출처가 붙은 요약만 돌려준다."
date: 2026-07-07
project: agentroom
tags: [claude-code, agents, agentroom, build-in-public]
---

*🇬🇧 [English version below](#english-version)*

지난주에 [AgentRoom](https://github.com/sasostudio/agentroom)을 공개했다. Claude Code 세션 하나가 총괄(director)이 되어 planner·dev·qa 세 서브에이전트를 오가게 하는 오케스트레이션 레이어다. [첫 글](/2026/07/05/agentroom-human-gated-orchestration/)의 원칙은 두 문장이었다. **주장을 믿지 말고 상태를 검증하라**, 그리고 **되돌리기 어려운 일은 사람에게 물어라.**

이번 주에 v1.1.0을 냈다. 눈에 띄는 변화는 하나다: **네 번째 역할, `researcher`(조사)를 추가했다.** 설계 전에 웹을 뒤져 사례를 모으고, 쓰려는 외부 API·라이브러리가 실제로 있는지, 무료 할당량은 얼마인지, 문서 URL은 뭔지 검증하는 에이전트다. planner가 가정 위에 설계하는 걸 막아준다.

그런데 정작 시간을 쓴 건 "조사 에이전트를 어떻게 잘 만들까"가 아니었다. **"이걸 언제 부르지 않을까"** 였다.

## 능력을 더하는 건 쉽다. 안 쓸 때를 정하는 게 어렵다.

조사 에이전트를 붙일 때 가장 자연스러운 충동은 "혹시 모르니 매번 돌리자"다. 어떤 작업이든 관련 사례나 외부 정보가 있을 수 있으니까. 그런데 이건 함정이다. 새 서브에이전트를 spawn할 때마다 CLAUDE.md와 스킬이 통째로 다시 로드되고, 그만큼 토큰이 크게 나간다. 그리고 대부분의 버그 수정·소규모 변경은 **외부 정보가 전혀 필요 없다.** "혹시 모르니"로 매번 조사자를 부르면, 그 대부분은 그냥 낭비다.

그래서 researcher는 **기본이 꺼짐(default-off)**이다. 딱 세 경우에만 합류한다:

1. **관리자가 명시적으로 요청**할 때 (`--members researcher`, "조사해서 …")
2. **외부 정보 없이는 불가능한 작업** — 사례·벤치마킹 수집, 외부 API·라이브러리의 실존·할당량·가격 검증, 최신 정책·트렌드 확인
3. **진행 중 planner나 dev가 "외부 정보가 필요하다"고 올릴 때** — 단, 곧바로 부르지 않고 **관리자 게이트를 거친 뒤** 투입

운영 지침에 한 줄을 박아뒀다: **"있으면 좋을 듯은 투입 사유가 아니다."** 애매하면 부르지 말고 그냥 시작한다. 추측성 spawn이 이 시스템에서 토큰이 가장 크게 새는 지점이기 때문이다.

## 대신, 부르면 마음껏 파게 했다

역설적으로, 투입 조건은 빡빡하게 잠갔지만 **일단 부르면 내부 조사량에는 제한을 두지 않았다.** 필요한 만큼 검색하고 열람해도 된다. 어설프게 조사해서 재조사로 왕복하는 게 더 큰 낭비니까.

이게 가능한 이유는 researcher가 **독립된 컨텍스트**에서 돌기 때문이다. 조사자가 웹페이지 수십 개를 읽든 말든 그 원문은 총괄의 컨텍스트로 들어오지 않는다. 조사자는 마지막에 **출처 URL이 붙은 구조화된 요약만** 반환한다. 웹 원문 통짜 덤프는 금지다.

이건 qa에서 이미 쓴 수와 똑같다. qa도 독립 컨텍스트에서 실제 파일을 마음껏 읽지만, 총괄에게는 판정(APPROVED/CHANGES_REQUESTED)과 발견 문제 요약만 돌려준다. **격리는 방어만을 위한 게 아니다. 깊이까지 가능하게 하는 장치다.** 무제한 깊이, 그리고 범위를 정한 반환. 오케스트레이터를 오염시키지 않으면서 서브에이전트가 깊이 파게 하는 방법.

## 조건부로만 켜지는 디자인 검수

같은 "필요할 때만 켠다" 원칙을 스킬에도 적용했다. v1.1.0은 두 개의 디자인 스킬을 조건부로 붙였다:

- **qa**: 총괄이 검수 지시에 **"디자인 검수 포함"**을 명시했을 때만 — 즉 UI 파일(html/css/레이아웃)이 바뀐 검수에서만 — `web-design-guidelines` 스킬을 로드해 **객관적 위반**(대비·간격·계층·반응형)을 추가로 검수한다.
- **dev**: 화면을 만드는 작업(HTML/CSS 시안·레이아웃·테마)에 착수하기 직전에만 `artifact-design`을 로드한다.

핵심은 두 가지다. 하나, **"예쁜가"는 판정하지 않는다.** 시각 취향은 관리자 몫이고, 스킬은 객관 위반만 잡는다. 둘, **비UI 작업엔 로드하지 않는다.** 로직·백엔드·문서 검수에 디자인 스킬을 얹으면 토큰만 태운다. 로드는 딱 1회, 조건 충족 시에만.

## 공개판으로 옮기며 — 나와 도구를 분리하기

여기서 성가신 문제가 하나 있었다. 내 개인 버전의 서브에이전트들은 **내 개인 모드 스킬**을 계승한다 — dev는 내 `developer-mode`, qa는 내 `audit-inspector`, researcher는 내 `idea-researcher`. 이 스킬들은 내 기계에만 있다. 그대로 공개하면 남의 환경엔 존재하지 않는 스킬을 물고 늘어져 깨진다.

그래서 공개판 v1.1.0으로 옮길 때 여덟 가지를 조정했는데, 핵심은 넷이다:

1. **개인 스킬 계승 제거** — 공개판 에이전트는 내 개인 스킬을 물지 않는다. 대신 사용자가 `skills:` 필드에 자기 프로젝트 표준을 꽂을 수 있게 열어뒀다.
2. **스킬 부재 시 우아한 강등** — 조건부 디자인 스킬은 그 스킬이 환경에 **있을 때만** 로드하고, 없으면 그 사실을 요약에 적고 기본 기준으로 진행한다. 도구 오류로 로드가 안 돼도 우회하지 않고 정직하게 보고한다.
3. **모델 질문 시점** — researcher는 리서치 작업에만 합류하므로, 그 모델은 매번 미리 묻지 않고 **실제로 합류할 때만** 묻는다(또는 `--models researcher=`로 지정).
4. **개인 요소 유입 0 검증** — 로컬 경로·개인 프로젝트명 같은 게 공개 산출물에 섞이지 않았는지 전수 확인.

돌아보면 이건 [지난 글의 유출 감사 파이프라인](/2026/07/06/devlog-leak-audit-gate/)과 같은 문제다. 내 것과 남에게 줄 것 사이의 경계를 긋는 일.

## 같은 원칙, 이번엔 '완료'가 아니라 '능력'에

첫 글의 원칙은 **"에이전트의 '완료했습니다'를 검증 없이 믿지 마라"** 였다. 게이트는 이렇게 물었다. *정말 끝났나?*

v1.1.0에서 게이트가 던지는 질문이 하나 늘었다. *이게 정말 돌아가야 하나?*

추측으로 부른 에이전트는 **검증되지 않은 비용**이다. 검증되지 않은 "완료"와 같은 족보다. 둘 다 "그럴듯해서" 통과시키면 새는 거다. default-off는 그 원칙을 완료 판정이 아니라 **능력 발동**에 적용한 것뿐이다. 능력을 더하는 일은 언제나 쉽다. 어렵고 정작 중요한 일은, 그걸 언제 **부르지 않을지**를 정하는 것이었다.

그리고 정직하게 밝히자면: 이 네 번째 에이전트도 지난 두 글처럼, 켜야 할 이유를 스스로 증명해야만 켜진다.

---

## English version

Last week I published [AgentRoom](https://github.com/sasostudio/agentroom) — an orchestration layer for Claude Code where one session becomes the **director** and ping-pongs three subagents: planner, dev, qa. [The first post's](/2026/07/05/agentroom-human-gated-orchestration/) thesis was two sentences: **don't trust claims, verify state**, and **stop and ask the human for anything hard to reverse.**

This week I shipped v1.1.0. The visible change is one thing: **a fourth role, `researcher`.** An agent that searches the web for prior art before design, and verifies that the external API or library you're about to use actually exists — its free quota, its docs URL. It keeps planner from designing on top of assumptions.

But the part I actually spent time on wasn't "how do I make a good research agent." It was **"when do I *not* call it."**

### Adding capability is easy. Deciding when not to use it is hard.

The most natural instinct when bolting on a research agent is "run it every time, just in case." Any task might have relevant prior art or external context. But that's a trap. Every subagent spawn reloads CLAUDE.md and skills from scratch — that's a lot of tokens. And most bug fixes and small changes need **no external information at all.** Calling the researcher "just in case" on every run means most of those calls are pure waste.

So the researcher is **default-off.** It joins in exactly three cases:

1. **The manager explicitly asks** (`--members researcher`, "go research …").
2. **The task is impossible without external info** — collecting examples/benchmarks, verifying an external API/library's existence, quota, or price, checking the latest policy or trend.
3. **Mid-run, planner or dev raises "I need external info"** — and even then it isn't called immediately; it goes through a **human gate** first.

There's a literal line in the operating rules: **"a 'might be nice to have' is not a reason to invoke."** If it's ambiguous, don't spawn — just start. Speculative spawning is the single biggest token leak in this system.

### But once you call it, let it dig freely

Paradoxically, while the *trigger* is locked down hard, once it *is* called I put **no cap on how much it researches internally.** Search and read as much as the question needs — a shallow pass that forces a re-research round-trip is the bigger waste.

This works because the researcher runs in an **isolated context.** Whether it reads dozens of web pages or not, none of that raw text enters the director's context. The researcher returns only a **structured summary with source URLs.** Dumping raw web content is forbidden.

This is the exact same move as qa. qa also reads the actual files freely in its own isolated context, but returns only a verdict (APPROVED/CHANGES_REQUESTED) and a summary of findings to the director. **Isolation isn't only defense — it's what enables depth.** Uncapped depth, bounded return. It's how you let a subagent dig deep without polluting the orchestrator.

### Design review that only turns on conditionally

I applied the same "on only when needed" principle to skills. v1.1.0 attaches two design skills, conditionally:

- **qa** loads the `web-design-guidelines` skill only when the director marks the review instruction with **"design review included"** — i.e. only on reviews where UI files (html/css/layout) changed — to additionally check **objective violations** (contrast, spacing, hierarchy, responsiveness).
- **dev** loads `artifact-design` only right before screen-building work (HTML/CSS mockups, layouts, themes).

Two things matter here. One, **it doesn't judge "is it pretty"** — visual taste is the manager's call; the skill only catches objective violations. Two, **it doesn't load for non-UI work.** Piling a design skill onto a logic, backend, or docs review just burns tokens. Load exactly once, only when the condition is met.

### Porting to public — separating me from the tool

There was one annoying problem here. My personal version's subagents inherit **my personal mode-skills** — dev inherits my `developer-mode`, qa my `audit-inspector`, researcher my `idea-researcher`. Those skills exist only on my machine. Ship them as-is and they'd grab at skills that don't exist in anyone else's environment and break.

So porting to the public v1.1.0 took eight adjustments; four of them are the core:

1. **Removed personal skill inheritance** — the public agents don't grab my personal skills. Instead I opened up the `skills:` field so users can plug in their own project standards.
2. **Graceful degradation when a skill is absent** — the conditional design skills load **only if** the skill exists in the environment; if not, the agent notes that and proceeds on baseline criteria. Even if a tool error blocks the load, it doesn't work around it — it reports honestly.
3. **When the model is asked** — since the researcher only joins research-needed tasks, its model isn't asked upfront every run; it's asked **only when it actually joins** (or set via `--models researcher=`).
4. **Verified zero personal-element leak** — a full pass to confirm nothing like local paths or personal project names slipped into the public artifacts.

Looking back, this is the same problem as [last post's leak-audit pipeline](/2026/07/06/devlog-leak-audit-gate/): drawing the line between what's mine and what I hand to others.

### Same thesis — this time on capability, not completion

The first post's thesis was **"don't trust an agent's 'done' without verification."** The gate asked: *did it really finish?*

In v1.1.0 the gate gained a second question: *does this really need to run?*

An agent invoked on a guess is an **unverified cost** — same family as an unverified "done." Wave either through because it "seems fine" and you're leaking. default-off is just that thesis applied to *capability* instead of *completion*. Adding capability is always easy. The hard part — and the part that actually mattered — was deciding when **not** to call it.

And to be honest about it: like the two posts before it, this fourth agent has to prove its reason to run before it runs at all.
