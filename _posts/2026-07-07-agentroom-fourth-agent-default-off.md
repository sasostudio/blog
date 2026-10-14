---
layout: post
title: "네 번째 에이전트 - researcher"
description: "AgentRoom v1.1.0의 조사 에이전트 researcher 추가 기록. 잘 만드는 법보다 「언제 부르지 않을지」를 먼저 결정. 기본은 꺼짐, 호출 시 독립 컨텍스트에서 자유롭게 조사한 뒤 출처가 붙은 요약만 반환."
date: 2026-07-07
project: agentroom
tags: [claude-code, agents, agentroom, build-in-public]
---

*🇬🇧 [English version below](#english-version)*

지난주 AgentRoom 공개(현재는 비공개 — 후속 엔진은 [AgentWorkflow](https://github.com/sasostudio/agentworkflow)). Claude Code 세션 하나가 총괄(director)이 되어 planner·dev·qa 세 서브에이전트를 오가게 하는 오케스트레이션 레이어. [첫 글](/2026/07/05/agentroom-human-gated-orchestration/)의 원칙은 두 문장. **주장을 믿지 말고 상태를 검증하라**, 그리고 **되돌리기 어려운 일은 사람에게 물어라.**

이번 주 v1.1.0 출시. 눈에 띄는 변화는 하나: **네 번째 역할, `researcher`(조사) 추가.** 설계 전에 웹에서 사례를 수집하고, 쓰려는 외부 API·라이브러리의 실존 여부, 무료 할당량, 문서 URL을 검증하는 에이전트. planner가 가정 위에 설계하는 상황의 예방이 목적.

그런데 정작 시간이 많이 든 질문은 「조사 에이전트를 어떻게 잘 만들까」가 아니라 **「언제 부르지 않을까」**.

## 능력 추가는 쉽고, 미사용 조건 결정은 어려움

조사 에이전트를 붙일 때 가장 자연스러운 충동은 「혹시 모르니 매번 실행」. 어떤 작업이든 관련 사례나 외부 정보가 있을 수 있다는 이유. 그런데 이건 함정. 새 서브에이전트를 spawn할 때마다 CLAUDE.md와 스킬 전체가 다시 로드되고, 그만큼 토큰 소모도 대량. 그리고 대부분의 버그 수정·소규모 변경에는 **외부 정보가 전혀 불필요.** 「혹시 모르니」로 매번 조사자를 부르면 대부분은 그대로 낭비.

그래서 researcher의 기본값은 **비활성(default-off)**. 합류 조건은 딱 세 가지:

1. **관리자의 명시적 요청** (`--members researcher`, "조사해서 …")
2. **외부 정보 없이는 불가능한 작업** — 사례·벤치마킹 수집, 외부 API·라이브러리의 실존·할당량·가격 검증, 최신 정책·트렌드 확인
3. **진행 중 planner나 dev의 「외부 정보 필요」 요청** — 단, 즉시 호출이 아니라 **관리자 게이트 통과 후** 투입

운영 지침에 넣은 한 줄: **"있으면 좋을 듯은 투입 사유가 아니다."** 애매하면 호출 없이 바로 시작. 추측성 spawn이 이 시스템에서 토큰이 가장 크게 새는 지점이기 때문.

## 대신, 호출 후 조사량은 무제한

역설적으로, 투입 조건은 엄격하게 제한했지만 **일단 호출하면 내부 조사량은 무제한.** 필요한 만큼 검색·열람 허용. 어설픈 조사 후 재조사 왕복이 더 큰 낭비라는 판단.

이게 가능한 이유는 researcher가 **독립된 컨텍스트**에서 실행되는 구조이기 때문. 조사자가 웹페이지 수십 개를 읽어도 그 원문은 총괄의 컨텍스트로 유입 불가. 조사자의 최종 반환물은 **출처 URL이 붙은 구조화된 요약뿐**. 웹 원문 통째 덤프는 금지.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/agentroom-fourth-agent-default-off/researcher-gate.ko.svg"><img src="/assets/img/posts/agentroom-fourth-agent-default-off/researcher-gate.ko.svg" width="720" height="280" loading="lazy" alt="researcher는 기본 꺼짐, 세 조건에서만 합류: 관리자의 명시 요청, 외부 정보 없이 불가능한 작업, planner나 dev의 요청이 관리자 게이트를 통과한 경우. 합류 후에는 독립 컨텍스트에서 무제한 조사, 총괄에게는 구조화된 요약과 출처 URL만 반환."></a></div><figcaption>투입 조건은 좁게, 조사는 깊게, 반환은 좁게. 웹 원문은 총괄의 컨텍스트로 유입 불가.</figcaption></figure>

qa에서 이미 쓴 방식과 동일한 수. qa도 독립 컨텍스트에서 실제 파일을 자유롭게 읽지만, 총괄에게는 판정(APPROVED/CHANGES_REQUESTED)과 발견 문제 요약만 반환. **격리는 방어만을 위한 장치가 아니라 깊이까지 가능하게 하는 장치.** 무제한 깊이와 범위가 정해진 반환. 오케스트레이터를 오염시키지 않으면서 서브에이전트의 깊은 조사를 허용하는 방법.

## 조건부로만 켜지는 디자인 검수

같은 「필요할 때만 켬」 원칙을 스킬에도 적용. v1.1.0에서 두 디자인 스킬을 조건부로 추가:

- **qa**: 총괄이 검수 지시에 **"디자인 검수 포함"**을 명시했을 때만 — 즉 UI 파일(html/css/레이아웃)이 바뀐 검수에서만 — `web-design-guidelines` 스킬을 로드해 **객관적 위반**(대비·간격·계층·반응형)을 추가 검수.
- **dev**: 화면 제작 작업(HTML/CSS 시안·레이아웃·테마) 착수 직전에만 `artifact-design` 로드.

핵심은 두 가지. 첫째, **「예쁜가」는 판정 대상 외.** 시각 취향은 관리자 몫, 스킬의 역할은 객관 위반 검출뿐. 둘째, **비UI 작업에는 로드 금지.** 로직·백엔드·문서 검수에 디자인 스킬을 얹으면 토큰 낭비. 로드는 조건 충족 시 딱 1회.

## 공개판 이전 — 나와 도구의 분리

여기서 번거로운 문제 하나. 개인 버전의 서브에이전트들은 **개인 모드 스킬**을 계승하는 구조 — dev는 `developer-mode`, qa는 `audit-inspector`, researcher는 `idea-researcher`. 이 스킬들은 내 기계에만 존재. 그대로 공개하면 남의 환경에서는 없는 스킬을 참조하다 오작동.

그래서 공개판 v1.1.0으로 옮기며 여덟 가지를 조정, 그중 핵심은 넷:

1. **개인 스킬 계승 제거** — 공개판 에이전트는 개인 스킬 미참조. 대신 사용자가 `skills:` 필드에 자기 프로젝트 표준을 지정하는 구조로 개방.
2. **스킬 부재 시 우아한 강등** — 조건부 디자인 스킬은 환경에 **존재할 때만** 로드, 부재 시 그 사실을 요약에 기록하고 기본 기준으로 진행. 도구 오류로 로드 실패 시에도 우회 없이 정직하게 보고.
3. **모델 질문 시점** — researcher는 리서치 작업에만 합류하므로, 모델 확인도 매번 사전이 아니라 **실제 합류 시점에만**(또는 `--models researcher=`로 지정).
4. **개인 요소 유입 0 검증** — 로컬 경로·개인 프로젝트명 같은 요소가 공개 산출물에 섞였는지 전수 확인.

돌아보면 [지난 글의 유출 감사 파이프라인](/2026/07/06/devlog-leak-audit-gate/)과 같은 문제. 내 것과 남에게 줄 것 사이의 경계 설정.

## 같은 원칙, 이번엔 「완료」가 아니라 「능력」에 적용

첫 글의 원칙은 **"에이전트의 '완료했습니다'를 검증 없이 믿지 마라"**. 게이트의 질문은 *정말 끝났나?*

v1.1.0에서 게이트의 질문이 하나 추가. *이게 정말 돌아가야 하나?*

추측으로 부른 에이전트는 **검증되지 않은 비용**. 검증되지 않은 「완료」와 같은 계열. 둘 다 「그럴듯하다」는 이유로 통과시키면 누수 발생. default-off는 그 원칙을 완료 판정이 아니라 **능력 발동**에 적용한 결과일 뿐. 능력 추가는 언제나 쉬운 일. 어렵고도 정작 중요한 일은 **언제 부르지 않을지**의 결정.

그리고 정직한 고지: 이 네 번째 에이전트도 지난 두 글처럼, 켜야 할 이유를 스스로 증명해야만 작동.

---

## English version

Last week I published AgentRoom (now private — its successor is [AgentWorkflow](https://github.com/sasostudio/agentworkflow)) — an orchestration layer for Claude Code where one session becomes the **director** and ping-pongs three subagents: planner, dev, qa. [The first post's](/2026/07/05/agentroom-human-gated-orchestration/) thesis was two sentences: **don't trust claims, verify state**, and **stop and ask the human for anything hard to reverse.**

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

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/agentroom-fourth-agent-default-off/researcher-gate.en.svg"><img src="/assets/img/posts/agentroom-fourth-agent-default-off/researcher-gate.en.svg" width="720" height="280" loading="lazy" alt="researcher is off by default and joins only on an explicit request, for work impossible without outside information, or when a planner or dev request clears the human gate. Once in, it digs without limits in an isolated context and returns only a structured summary with source URLs."></a></div><figcaption>Narrow conditions to join, deep digging inside, a narrow return. Raw web text never reaches the director.</figcaption></figure>

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
