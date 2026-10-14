---
layout: post
title: "'완료했습니다'를 못 믿어서 만든 오케스트레이션"
description: "에이전트의 「완료했습니다」를 검증 없이 믿지 않으려고 만든 Claude Code 오케스트레이션 AgentRoom. dev는 스스로 완료를 선언할 수 없고, 읽기 전용 qa가 실제 파일로 판정하며, 되돌리기 어려운 일은 사람에게 묻는다."
date: 2026-07-05
project: agentroom
tags: [claude-code, agents, agentroom, build-in-public]
---

*🇬🇧 [English version below](#english-version)*

나는 많은 걸 직접 만든다. 내 컴퓨터에서만 도는 로컬 LLM 개인 에이전트, 그리고 스튜디오 운영을 지탱하는 온갖 도구들.

한동안은 AI 에이전트 "팀"을 손으로 굴렸다. 총괄 세션 하나, 개발 세션 하나, 검수 세션 하나. Claude Code 세션을 여러 개 띄워놓고 한쪽의 결과물과 계획서를 복사해 다른 쪽에 붙여넣으며 지시를 날랐다. 역할을 나누니 품질은 확실히 좋아졌다. 문제는 내가 하루 종일 에이전트 사이를 오가며 말을 나르는 심부름꾼이 됐다는 거다.

그 과정에서 더 뼈아픈 것도 배웠다. 에이전트가 **"완료했습니다, 빌드 통과합니다"** 라고 해도 그대로 믿으면 안 된다는 것. 실제로 안 돌려보고 됐다고 하고, 실패를 성공처럼 보고하는 일을 몇 번 겪은 뒤, 내 작업 규칙 문서에는 **"거짓 완료 보고 금지"** 라는 조항이 생겼다. 사람이 규칙 문서에 이런 문장까지 적게 됐다는 것만 봐도 문제가 얼마나 큰지 짐작이 간다.

완전 자동 오케스트레이터도 여럿 봤다. 그런데 대부분은 사람을 더 빼는 쪽으로 가고 있었다. 방향이 2단계에서 어긋나도 10단계까지 내달리고, 코드를 쓴 에이전트가 완료 판정까지 스스로 내린다. 내 중계 노동이야 없애주겠지만, 정작 내가 제일 무서워하는 방향 이탈과 거짓 "완료"는 오히려 더 키울 것 같았다.

## 문제는 자동화가 아니라, 검증 없는 신뢰다

대부분의 오케스트레이터는 한 가지 질문에 최적화돼 있다: *사람 없이 에이전트가 얼마나 멀리 갈 수 있는가?*

내가 원한 질문은 달랐다: *중계 노동은 자동화하되, 에이전트가 나에게 거짓말할 수 없는 상태로 얼마나 멀리 갈 수 있는가?*

그래서 AgentRoom을 만들었다. Claude Code 세션 하나가 **총괄(director)**을 맡아 세 서브에이전트 **planner**(설계)·**dev**(구현)·**qa**(검수)를 오가게 하는 작은 오케스트레이션 레이어다. 설계는 흔한 "자율성 우선" 가정을 뒤집는다:

1. **dev는 스스로 "완료"를 선언할 수 없다.** 완료는 반드시 qa를 거친다. 총괄은 qa가 `APPROVED`를 반환한 뒤에만 작업을 종료한다.
2. **qa는 물리적으로 읽기 전용이다.** 도구 권한에 Write/Edit 자체가 없어서 "조용히 고쳐놓기"가 불가능하다. 디스크의 실제 파일을 직접 읽고, dev의 *주장*과 대조한 뒤 판정을 내린다. 주장은 단서일 뿐, 증거는 파일이다.
3. **되돌리기 어려운 일은 멈추고 사람에게 묻는다.** 방향 전환, git push, 파일 삭제, 배포. 이런 일 앞에서는 총괄이 반드시 멈추고 나에게 물어야 한다. 되돌리기 쉬운 일(코드 작성, 검수)만 무인으로 달린다.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/agentroom-human-gated-orchestration/roles.ko.svg"><img src="/assets/img/posts/agentroom-human-gated-orchestration/roles.ko.svg" width="720" height="350" loading="lazy" alt="관리자 아래 총괄(director)이 있고, 총괄이 planner·dev·qa 세 에이전트를 오간다. qa는 읽기 전용이며 dev의 완료는 qa를 거친다. 방향 전환·push·삭제·배포 앞에서는 총괄이 멈추고 관리자에게 묻는다."></a></div><figcaption>총괄이 세 에이전트를 오가되, 완료는 읽기 전용 qa를 거치고 되돌리기 어려운 일은 사람에게 묻는다.</figcaption></figure>

매 턴은 재개용 필드와 함께 transcript 파일에 쌓인다. 세션이 중간에 죽어도 새 세션이 마지막 상태를 누군가의 기억이 아니라 디스크에서 복원해 이어간다. 실행을 시작할 때는 역할마다 어떤 모델을 쓸지 묻는다. 문제를 못 잡을 만큼 약한 qa는 이 설계 전체를 무너뜨리니까.

## 출시 당일, 내 도구에 설득당한 순간

이 도구를 공개하기로 결심하게 만든 사건이 있다.

첫 end-to-end 테스트에서 첫 dev 에이전트가 **소리 없이 증발했다**. 서브에이전트 호출이 기본적으로 백그라운드로 돌아가는데, 총괄이 dev가 "아직 작업 중"인 상태로 자기 턴을 끝내버린 것이다. 에러도, 출력도, 아무것도 없었다.

그런데 시스템이 정확히 설계대로 움직였다. 총괄은 dev가 만들었어야 할 파일을 디스크에서 찾았고, 없다는 걸 확인한 뒤 에이전트가 유실됐다고 판정하고 다시 불렀다. 그리고 그 결과를 qa로 넘겼고, qa는 실제 파일을 읽고 승인했다. 실행은 자기 인프라의 실패를 *우회해서* 올바르게 끝났다.

그 실패는 나에게 거짓 "완료"로 도착하지 않았다. 복구 과정이 담긴 transcript 한 줄로 도착했다. 근본 원인은 그날 바로 고쳤고(총괄 규칙에 동기 호출 의무화), v1.0을 공개했다.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/agentroom-human-gated-orchestration/launch-day-recovery.ko.svg"><img src="/assets/img/posts/agentroom-human-gated-orchestration/launch-day-recovery.ko.svg" width="720" height="220" loading="lazy" alt="dev 호출이 소리 없이 증발하자 총괄이 디스크에서 결과 파일이 없음을 확인하고 유실로 판정해 다시 부른다. qa가 실제 파일을 읽고 승인했고, 관리자에게는 transcript 한 줄로 도착했다."></a></div><figcaption>출시 당일의 첫 실행. 도구가 자기 인프라의 실패를 상태 검증으로 우회했다.</figcaption></figure>

"주장을 믿지 말고 상태를 검증하라"를 원칙으로 내세운 도구가, 정작 그 말대로 주장을 믿지 않고 상태를 검증해 자기 출시 버그를 잡은 것이다. 이 정도면 됐다 싶었다.

## 실체는 이렇다

코드 0줄. 슬래시 커맨드 1개 + 마크다운 에이전트 정의 3개가 전부고, Claude Code에 내장된 서브에이전트 기능 위에서 돌아간다. 이미 쓰고 있는 구독으로 동작한다. API 키도, 서버도, 프레임워크도 없다.

- 레포: 현재 비공개 — 같은 원칙을 이어받은 후속 엔진은 [AgentWorkflow](https://github.com/sasostudio/agentworkflow)로 공개했다 (2026-10)
- 설치: `.claude/` 폴더를 프로젝트에 복사 → 세션 재시작 → `/agentroom <작업>`
- 정직한 고지: Opus + 높은 reasoning effort 기준으로 설계·튜닝됐다. 약한 모델은 약한 검수자가 된다.

에이전트의 "완료했습니다"에 데인 적이 있다면 어떤 상황이었는지 듣고 싶다. 읽기 전용 검수자가 있었다면 그걸 잡을 수 있었을까?

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
