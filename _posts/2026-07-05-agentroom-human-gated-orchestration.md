---
layout: post
title: "'완료했습니다'를 못 믿어서 만든 오케스트레이션"
description: "에이전트의 「완료했습니다」를 검증 없이 믿지 않으려고 만든 Claude Code 오케스트레이션 AgentRoom. dev의 자체 완료 선언 불가, 읽기 전용 qa의 실제 파일 기준 판정, 되돌리기 어려운 일은 사람에게 확인 요청."
date: 2026-07-05
project: agentroom
tags: [claude-code, agents, agentroom, build-in-public]
---

*🇬🇧 [English version below](#english-version)*

직접 만드는 것이 많은 편. 내 컴퓨터에서만 도는 로컬 LLM 개인 에이전트, 그리고 스튜디오 운영을 지탱하는 온갖 도구들.

한동안은 AI 에이전트 「팀」을 수동 운영. 총괄 세션 하나, 개발 세션 하나, 검수 세션 하나. Claude Code 세션을 여러 개 띄워 놓고 한쪽의 결과물과 계획서를 복사해 다른 쪽에 붙여넣는 방식의 지시 전달. 역할 분리 덕분에 품질은 확실히 향상. 문제는 하루 종일 에이전트 사이를 오가며 말을 나르는 심부름꾼 신세가 된 점.

그 과정에서 얻은 더 뼈아픈 교훈. 에이전트가 **"완료했습니다, 빌드 통과합니다"** 라고 해도 그대로 믿으면 안 된다는 사실. 실제로 돌려 보지 않고 완료라고 하거나, 실패를 성공처럼 보고하는 일을 몇 번 겪은 뒤 작업 규칙 문서에 **"거짓 완료 보고 금지"** 조항 추가. 사람이 규칙 문서에 이런 문장까지 적게 됐다는 사실만으로도 문제의 크기가 짐작 가능.

완전 자동 오케스트레이터도 여럿 조사. 그런데 대부분은 사람을 더 빼는 방향. 방향이 2단계에서 어긋나도 10단계까지 질주, 코드를 쓴 에이전트가 완료 판정까지 자체 결정. 중계 노동은 없애 주겠지만, 가장 두려운 방향 이탈과 거짓 「완료」는 오히려 키울 것이라는 판단.

## 문제는 자동화가 아니라 검증 없는 신뢰

대부분의 오케스트레이터가 최적화하는 질문은 하나: *사람 없이 에이전트가 얼마나 멀리 갈 수 있는가?*

내가 원한 질문은 별개: *중계 노동은 자동화하되, 에이전트가 거짓말할 수 없는 상태로 얼마나 멀리 갈 수 있는가?*

그래서 만든 것이 AgentRoom. Claude Code 세션 하나가 **총괄(director)** 역할로 세 서브에이전트 **planner**(설계)·**dev**(구현)·**qa**(검수)를 오가게 하는 작은 오케스트레이션 레이어. 설계의 핵심은 흔한 「자율성 우선」 가정의 반전:

1. **dev의 자체 「완료」 선언 불가.** 완료는 반드시 qa 경유. 총괄의 작업 종료는 qa의 `APPROVED` 반환 이후에만 가능.
2. **qa는 물리적으로 읽기 전용.** 도구 권한에 Write/Edit 자체가 없어 「조용히 고쳐놓기」 불가. 디스크의 실제 파일을 직접 읽고 dev의 *주장*과 대조한 뒤 판정. 주장은 단서, 증거는 파일.
3. **되돌리기 어려운 일은 정지 후 사람에게 확인.** 방향 전환, git push, 파일 삭제, 배포. 이런 일 앞에서는 총괄의 정지와 확인 요청이 의무. 무인 실행은 되돌리기 쉬운 일(코드 작성, 검수)에 한정.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/agentroom-human-gated-orchestration/roles.ko.svg"><img src="/assets/img/posts/agentroom-human-gated-orchestration/roles.ko.svg" width="720" height="350" loading="lazy" alt="관리자 아래 총괄(director), 총괄이 planner·dev·qa 세 에이전트를 순회. qa는 읽기 전용, dev의 완료는 qa 경유. 방향 전환·push·삭제·배포 앞에서는 총괄이 정지 후 관리자에게 확인 요청."></a></div><figcaption>총괄이 세 에이전트를 순회하되, 완료는 읽기 전용 qa 경유, 되돌리기 어려운 일은 사람의 확인 필요.</figcaption></figure>

매 턴은 재개용 필드와 함께 transcript 파일에 누적. 세션이 중간에 죽어도 새 세션이 마지막 상태를 누군가의 기억이 아니라 디스크에서 복원해 재개. 실행 시작 시 역할마다 사용할 모델을 사람에게 확인 요청. 문제를 못 잡을 만큼 약한 qa는 이 설계 전체의 붕괴 요인이기 때문.

## 출시 당일, 내 도구에 설득당한 순간

이 도구의 공개를 결심하게 만든 사건 하나.

첫 end-to-end 테스트에서 첫 dev 에이전트가 **소리 없이 증발**. 서브에이전트 호출이 기본적으로 백그라운드로 도는 구조인데, 총괄이 dev가 「아직 작업 중」인 상태에서 자기 턴을 종료한 것이 원인. 에러도, 출력도 전무.

그런데 시스템은 정확히 설계대로 동작. 총괄이 dev가 만들었어야 할 파일을 디스크에서 탐색, 부재를 확인한 뒤 에이전트 유실로 판정하고 재호출. 그 결과를 qa로 전달, qa는 실제 파일을 읽고 승인. 실행은 자기 인프라의 실패를 *우회해* 정상 종료.

그 실패의 도착 형태는 거짓 「완료」가 아니라 복구 과정이 담긴 transcript 한 줄. 근본 원인은 당일 수정(총괄 규칙에 동기 호출 의무화) 후 v1.0 공개.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/agentroom-human-gated-orchestration/launch-day-recovery.ko.svg"><img src="/assets/img/posts/agentroom-human-gated-orchestration/launch-day-recovery.ko.svg" width="720" height="220" loading="lazy" alt="dev 호출이 소리 없이 증발하자 총괄이 디스크에서 결과 파일 부재를 확인, 유실로 판정해 재호출. qa가 실제 파일을 읽고 승인, 관리자에게는 transcript 한 줄로 도착."></a></div><figcaption>출시 당일의 첫 실행. 자기 인프라의 실패를 상태 검증으로 우회한 사례.</figcaption></figure>

「주장을 믿지 말고 상태를 검증하라」를 원칙으로 내세운 도구가, 정작 그 원칙대로 상태를 검증해 자기 출시 버그를 잡은 사례. 이 정도면 충분하다는 판단.

## 실체

코드 0줄. 슬래시 커맨드 1개 + 마크다운 에이전트 정의 3개가 전부, Claude Code 내장 서브에이전트 기능 위에서 동작. 이미 쓰는 구독으로 동작 가능. API 키, 서버, 프레임워크 모두 불필요.

- 레포: 현재 비공개 — 같은 원칙을 이어받은 후속 엔진은 [AgentWorkflow](https://github.com/sasostudio/agentworkflow)로 공개 (2026-10)
- 설치: `.claude/` 폴더를 프로젝트에 복사 → 세션 재시작 → `/agentroom <작업>`
- 정직한 고지: Opus + 높은 reasoning effort 기준으로 설계·튜닝. 약한 모델은 약한 검수자로 귀결.

에이전트의 "완료했습니다"에 데인 경험이 있다면 어떤 상황이었는지 궁금. 읽기 전용 검수자가 있었다면 잡을 수 있었을지도 궁금.

---

## English version

I build a lot of things myself: a local-LLM personal agent, and all the tooling that keeps the studio running.

For a while, I ran my AI agent "team" by hand. One Claude Code session played the director, another the developer, another the reviewer — and I was the messenger between them, copying results and plans out of one session and pasting them into the next, relaying instructions all day. The role separation genuinely improved quality. It also turned me into a carrier pigeon.

Along the way I learned something more painful: an agent's **"Done. Build passes."** cannot be trusted without verification. After getting burned a few times — work reported as done that was never run, failures reported as successes — my personal working-rules file gained a literal clause: **"no false completion reports."** The fact that a human had to write that sentence down says everything about the size of the problem.

I did look into the fully-automated orchestrators. But most of them run in the opposite direction — removing the human further. Wrong direction at step 2, still sprinting to step 10; the agent that wrote the code also declares it finished. They would have automated away my relay labor — and amplified the two things I feared most: drift, and the false "done".

### The problem isn't automation. It's unverified trust.

Most orchestrators optimize for one question: *how far can agents run without you?*

The question I actually cared about was different: *automate the relaying — but how far can agents run without being able to lie to me?*

So I built AgentRoom — a small orchestration layer for Claude Code where one session becomes the **director** and ping-pongs three subagents: **planner** (design), **dev** (implementation), **qa** (review). The design inverts the usual autonomy-first assumptions:

1. **dev can never declare itself done.** Completion always routes through qa. The director terminates the run only after qa returns `APPROVED`.
2. **qa is read-only — physically.** Its tool permissions don't include Write or Edit, so it can't "fix things quietly". It has to read the actual files on disk, compare them against what dev *claimed*, and return a verdict. Claims are leads; files are evidence.
3. **Irreversible things stop and ask the human.** Direction changes, git pushes, deletions, deploys — the director must stop and ask me. Reversible work (writing code, reviewing it) runs unattended.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/agentroom-human-gated-orchestration/roles.en.svg"><img src="/assets/img/posts/agentroom-human-gated-orchestration/roles.en.svg" width="720" height="350" loading="lazy" alt="A director sits under the human and moves between planner, dev, and qa. qa is read-only and every “done” goes through it. Before direction changes, pushes, deletions, or deploys, the director stops and asks the human."></a></div><figcaption>The director moves between three agents; “done” goes through a read-only qa, and anything hard to undo goes to a human.</figcaption></figure>

Every turn gets appended to a transcript file with resume fields, so if the session dies mid-run, a fresh session picks up exactly where the last one stopped — from disk, not from anyone's memory. And at the start of each run it asks which model to use for each role, because a QA agent that's too weak to catch problems quietly defeats the whole design.

### The launch-day bug that sold me on my own tool

Here's the part that convinced me to publish it.

During the very first end-to-end test, the first dev agent **silently vanished**. Subagent calls run in the background by default, and the director ended its turn while dev was "still working" — the spawned agent was simply lost. No error, no output, nothing.

And then the system did exactly what it was designed to do: the director checked the disk for the file dev was supposed to create, found nothing, declared the agent lost, re-spawned it, and routed the result through qa — which read the actual file and approved. The run finished correctly *around* its own infrastructure failure.

The failure never reached me as a false "done". It reached me as a transcript line showing a recovery. I fixed the root cause the same day (synchronous agent calls are now mandatory in the director's rules) and shipped v1.0.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/agentroom-human-gated-orchestration/launch-day-recovery.en.svg"><img src="/assets/img/posts/agentroom-human-gated-orchestration/launch-day-recovery.en.svg" width="720" height="220" loading="lazy" alt="A dev spawn silently vanishes; the director checks the disk, finds no output file, marks it lost and re-spawns it. qa reads the real files and approves, and the human receives a single transcript line."></a></div><figcaption>The first launch-day run: the tool routed around its own infrastructure failure by verifying state.</figcaption></figure>

A tool whose whole thesis is "don't trust claims, verify state" caught its own launch bug by not trusting a claim and verifying state. I'll take that.

### What it literally is

Zero lines of code. One slash command plus three markdown agent definitions — the whole thing runs on Claude Code's built-in subagent machinery, on the subscription you already have. No API keys, no server, no framework.

- Repo: now private — its successor engine, built on the same principles, is public as [AgentWorkflow](https://github.com/sasostudio/agentworkflow) (2026-10)
- Install: copy `.claude/` into your project, restart your session, run `/agentroom <task>`
- Honest caveat: it was designed and tuned on Opus with high reasoning effort. Weaker models make weaker reviewers.

If you've ever been burned by an agent's "done", I'd like to hear how — and whether a read-only reviewer would have caught it.
