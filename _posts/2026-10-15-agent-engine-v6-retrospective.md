---
layout: post
title: "작업 엔진 v6 설계 회고"
description: "Claude Code를 사람 없이 오래 돌리는 작업 엔진의 v5 → v6 전환 회고. 같은 과제 3방식 실측으로 역할을 넷에서 둘로 축소, 측정 도중 비용 기록 30% 과다 발견, 첫 실전에서 드러난 「엔진의 판단 배달」 구조 제거까지. 오늘 MIT로 공개한 버전의 설계 기록."
date: 2026-10-15 00:10:00 +0900
tags: [build-in-public, ai-agents, claude-code, open-source]
---

*🇬🇧 [English version below](#english-version)*

오늘 장기 작업 엔진 [AgentWorkflow](https://github.com/sasostudio/agentworkflow)를 MIT 라이선스로 공개. Claude Code를 사람 없이 몇 시간씩 돌리는 엔진. 할 일 목록을 쓰면 엔진이 작업자 AI와 평가자 AI를 번갈아 호출하고, 사람의 결정이 필요한 순간에만 정지. 원칙이 같던 예전 오케스트레이션 레포(AgentRoom)는 이 엔진에 흡수 후 비공개 전환.

공개 대상은 v6. 2주 전까지는 v5였고, 두 버전의 구조 차이는 상당. 이 글의 주제는 v5에서 무엇을 근거로 덜어냈는지, 덜어내는 도중 무엇을 발견했는지에 대한 기록.

## v5: 역할 넷, 단계마다 승인

v5의 역할 분담은 넷. 총괄(Master), 기획, 설계와 구현, 검수. 단계마다 총괄 승인, 화면이 있는 결과물은 검수가 원작 캡처와 대조. 테스트 327개 통과, 일부러 심은 결함(변이) 21개도 테스트가 전부 검출. 구현 품질 자체는 문제없는 상태.

걸린 지점은 다른 질문. 역할을 이만큼 나누는 구조가 지금 모델에도 필요한지에 대한 의문. 최근 글을 웹에서 조사.

| 출처 | 요지 |
|---|---|
| Anthropic, 장기 실행 앱의 하네스 설계 | 새 모델 등장 후 작업 쪼개기 폐지, 평가는 마지막 1회로 축소. 하네스의 부품마다 「모델이 혼자 못 하는 것」을 가정하므로, 모델이 바뀌면 재시험 필요 |
| Anthropic, 장기 실행 에이전트용 하네스 | 초기화 에이전트와 코딩 에이전트 둘. 별도 QA·리뷰 에이전트는 미사용 |
| Cognition, 멀티 에이전트에서 실제로 되는 것 | 쓰기는 한 줄기로 유지, 다른 에이전트는 판단만 보탬 |

방향은 한쪽으로 일치. 다만 남의 숫자를 내 엔진에 그대로 옮기는 건 불가하다는 판단으로 직접 측정.

## 같은 과제, 세 방식

작은 과제 하나(버튼을 누르면 원이 빨강에서 초록으로 바뀌는 페이지)를 세 방식으로 실행. 모델은 셋 다 Haiku로 통일.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/agent-engine-v6-retrospective/three-ways.ko.svg"><img src="/assets/img/posts/agent-engine-v6-retrospective/three-ways.ko.svg" width="720" height="250" loading="lazy" alt="같은 과제를 세 방식으로 실행한 결과. v5 기본은 호출 11번, 1.545달러로 완료. v5 단순화는 호출 14번, 1.908달러에 역할 사이 오해가 이어져 사람 도움 요청으로 정지. v6 시제품은 호출 3번, 0.402달러로 통과."></a></div><figcaption>각 방식 1회, Haiku, 작은 과제라는 한계 존재. 그래도 역할을 늘린 쪽의 비용·정확도 우위는 전무.</figcaption></figure>

| 방식 | AI 호출 | 비용 | 결과 |
|---|---:|---:|---|
| v5 기본 | 11 | $1.545 | 완료 |
| v5 단순화 | 14 | $1.908 | 정지 (역할 사이 오해 연쇄) |
| v6 시제품 (작업자 + 평가자) | 3 | $0.402 | 통과 |

인상적인 지점은 v5 단순화 쪽의 호출 수 증가. 역할 사이 전달이 한 번 어긋나자 뒤 역할들이 그 어긋남을 이어받아 계속 왕복.

그래서 v5의 부품마다 같은 질문으로 점검. 이 부품이 가정하는 「모델이 혼자 못 하는 것」은 무엇이고, 그 가정이 지금도 유효한지.

| v5 부품 | 깔고 있던 가정 | v6 |
|---|---|---|
| 별도 설계 역할 | 구현자는 설계 불가 | 작업자 안으로 통합 |
| 기능 단위로 쪼갠 병렬 작업 | 긴 작업에서 방향 상실 | 묶음 단위, 작업자 1명이 순서대로 처리 |
| 호출마다 새 세션 | 맥락이 차면 품질 저하 | 묶음 동안 세션 유지 |
| 라운드마다 검수 1~2명 | 구현자 자기평가는 신뢰 불가 | 평가자 1명. 고위험 변경일 때만 두 번째 관점, 통과 직전 새 세션 반박 검토 1회 |

마지막 행은 유지. 자기가 만든 결과를 자기가 「완료」라고 선언하는 건 지금 모델에서도 신뢰 불가. 이 엔진을 만든 이유 자체가 그 지점이고, 모델이 바뀌어도 불변.

## 측정 도중 발견: 30% 부풀려진 비용 기록

세 방식을 측정하는 동안 비용 수치에 위화감. 추적 결과 원인은 엔진 쪽 기록.

CLI가 돌려주는 비용 값은 이어 쓴 세션일 경우 **그 세션의 누적값**. 엔진은 이 값을 그대로 「이번 호출 비용」으로 기록하는 상태. 같은 세션으로 세 번 호출하면 첫 호출 비용이 세 번 중복 합산되는 구조.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/agent-engine-v6-retrospective/cost-record.ko.svg"><img src="/assets/img/posts/agent-engine-v6-retrospective/cost-record.ko.svg" width="720" height="260" loading="lazy" alt="같은 세션을 이어 쓴 세 번의 호출에서 CLI 값은 세션 누적이라, 그대로 기록하면 앞 호출 비용이 겹쳐 합산. 올바른 기록은 앞 호출과의 차이. 예전 엔진 버전의 기록 979.9달러는 실제 약 689달러."></a></div><figcaption>CLI 값은 세션 누적. 호출 비용은 앞 호출과의 차이로 기록 필요.</figcaption></figure>

예전 버전(v4)의 실제 프로젝트 기록을 재계산한 결과, 기록 $979.9의 실제 값은 약 $689. 약 30% 과다 기록.

숫자보다 더 아팠던 부분은 그 숫자 위에서 내린 결정. 9월에 「세션을 이어 쓰면 새 세션보다 비싸다」고 판단해 이어 쓰기를 축소한 이력. 그때 근거로 본 이어 쓰기 비용이 바로 이 누적값. 원본 응답으로 재계산한 결과는 아래.

| 역할 | 이어 쓰기 기록 평균 | 재계산한 실제 | 같은 역할 새 세션 평균 |
|---|---:|---:|---:|
| 기획 (19건) | $5.55 | $1.96 | $2.04 |

이어 쓰기는 비싸지 않았다는 결론. 측정 도구의 결함이 설계 결정까지 끌고 간 사례. v6의 「묶음 동안 세션 이어 쓰기」도 이 재계산 덕분에 채택 가능. 캐시 유지 시간도 함께 측정: 쉬는 시간 20분까지 캐시 적중, 65분이면 맥락 전체 재생성. v6는 창과 무관하게 세션을 이어 쓰고, 50분은 캐시 만료 표시 기준으로만 사용.

## 첫 실전: 엔진의 판단 배달 오류

v6의 첫 실전 과제는 로블록스 게임의 테스트용 프로필 초기화 기능.

결과는 이상한 상태. 만든 코드는 평가 9회 모두 결함 0건, 시나리오 9개도 세 번 모두 통과. 그런데 같은 반려가 반복되다 반려 상한에 걸려 정지. 호출 15번, 비용 $11.98.

원인은 엔진. 통과 직전 반박 검토가 지적을 내면, 엔진이 그 지적을 종류(kind)별로 분류해 수신자를 결정하는 구조. 지적 중 코드 결함이 하나라도 섞이면 전부를 작업자에게 전달. 실제 지적은 「완료 조건 하나를 확인하는 시나리오 부재」라는 평가 쪽 문제였는데, 수신자는 작업자. 작업자는 고칠 대상이 없어 그대로, 평가는 같은 지적을 반복, 그렇게 세 바퀴.

처음엔 이 배달 규칙만 수정할 계획. 그 시점에 받은 지적이 "엔진 쪽으로 너무 갔다. 계획대로 구현 안 한 것 아니냐." 슬림화 계획서와 실제 구현을 대조한 결과, 계획에 없던 엔진 판단 장치 7개를 발견. 평가 호출 종류는 계획의 2개에서 7개로 증가한 상태. 평가자 출력을 엔진이 분류하고, 교차 확인하고, id를 되묻고, 시나리오 수 하한까지 강제. 역할을 덜어낸 빈자리를 엔진 규칙이 채우고 있던 셈.

그래서 기준부터 확정.

| # | 질문 | 판정 |
|:-:|---|---|
| ① | 무인 운영에 필요한 장치인지 (실행·대기·재개·기록·상한) | 엔진 |
| ② | 되돌릴 수 없는 피해를 막는 울타리인지 | 엔진 (③④보다 우선) |
| ③ | 에이전트의 판단을 엔진이 대신 해석하거나 칸으로 쪼개 경로를 결정하는지 | 에이전트가 판단, 엔진은 글을 그대로 전달 |
| ④ | 에이전트의 실수를 엔진 규칙 추가로 막았는지 | 지시나 역할로 수정 |
| ⑤ | 외부·과거 구조를 「지금 모델이 혼자 못 하는 게 뭔데?」라는 답 없이 유지했는지 | 제거 |

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/agent-engine-v6-retrospective/engine-vs-agent.ko.svg"><img src="/assets/img/posts/agent-engine-v6-retrospective/engine-vs-agent.ko.svg" width="720" height="300" loading="lazy" alt="정렬 전: 엔진이 평가자 출력을 종류별로 분류해 작업자나 평가자에게 배달, 코드 결함이 섞이면 전부 작업자로 전달. 정렬 후: 평가자가 다음 할 일(통과, 반려, 시나리오 실행, 불확실)을 직접 결정, 엔진은 글 그대로 전달·시나리오 실행·횟수 계산만 담당."></a></div><figcaption>판단은 평가자 담당. 엔진은 전달·실행·계산만.</figcaption></figure>

이 기준으로 평가 단계를 재작성. 평가자 한 세션이 판정마다 다음 할 일을 직접 결정: 통과(READY), 반려(REJECT, 작업자에게 보낼 글 포함), 시나리오 실행(RUN), 불확실(UNSURE) 중 하나. 엔진은 그 글을 손대지 않고 전달하고, 시나리오를 실행해 결과를 같은 세션에 반환하고, 횟수와 상한만 관리. 분류·교차 확인·id 되묻기·시나리오 하한은 전부 제거. 코드는 약 170줄 감소(+551 / −722). 테스트 218개 통과, 변이 13개 전부 검출.

멈춰 있던 첫 실전도 재개. 평가자가 빠져 있던 확인(퇴장 후 저장, 다음 플레이가 기본값으로 시작하는지)을 스스로 시나리오로 추가해 데이터로 확인. 플레이어가 퇴장하는 순간은 화면 캡처가 구조적으로 불가해서 평가자가 그 부분을 불확실로 상신, 총괄(Master)이 데이터 증거로 끝 승인. 재개분은 호출 5번, 약 $4.

## 정리

| 결정 | 근거 |
|---|---|
| 역할 넷 → 작업자 + 평가자 | 같은 과제 실측: 호출 11~14번 → 3번, 비용 약 1/4 |
| 세션 유지 | 「이어 쓰기가 비싸다」는 결정이 누적값 기록 결함 위에 존재 |
| 통과 직전 새 세션 반박 검토 유지 | 자기 결과에 대한 자기 「완료」 선언은 지금도 신뢰 불가 |
| 판단은 평가자, 엔진은 운영만 | 엔진이 판단을 배달하다 틀린 첫 실전 |

- 부품마다 「지금 모델이 혼자 못 하는 게 뭔데?」 점검. 답이 없으면 그 부품은 비용.
- 숫자가 이상하면 대상보다 측정 도구부터 의심. 이번 사례는 도구 결함, 그 위에 결정까지 존재.
- 엔진이 만들어 건넬 수 있는 건 사실뿐: 실행 결과, diff, 캡처, 테스트 통과 여부. 판단 해석이 시작되면 역할 축소의 의미 상실.

엔진은 Windows 전용, 프롬프트와 규칙 파일은 한국어. 레포 위치는 [github.com/sasostudio/agentworkflow](https://github.com/sasostudio/agentworkflow). 무인 실행 중 관측·정지 쪽 이야기는 [다음 글](/2026/10/15/unattended-engine-observe-and-stop/)에 별도 정리.

참고한 글: [Anthropic — Harness design for long-running application development](https://www.anthropic.com/engineering/harness-design-long-running-apps) · [Anthropic — Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) · [Cognition — Multi-Agents: What's Actually Working](https://cognition.com/blog/multi-agents-working)

---

## English version

Today I open-sourced [AgentWorkflow](https://github.com/sasostudio/agentworkflow) under MIT. It's an engine for running Claude Code unattended for hours: I write a task list, the engine alternates between a builder agent and an evaluator agent, and it stops only when a human decision is needed. My earlier orchestration repo, AgentRoom, was built on the same principles; it has been folded into this engine and made private.

What I released is v6. Two weeks ago it was v5, and the two are structurally quite different. This is about what I cut from v5, what I based those cuts on, and what I found while cutting.

### v5: four roles, an approval at every stage

v5 split the work across four roles: a Master, a planner, a designer-builder, and QA. The Master approved every stage, and anything with a screen was checked against reference captures. 327 tests passed, and the suite caught all 21 deliberately injected faults (mutants). The implementation was fine.

The question was different: do current models still need the work divided this finely? Recent writing pointed one way.

| Source | Gist |
|---|---|
| Anthropic, harness design for long-running apps | With a newer model they dropped task splitting and moved evaluation to a single pass at the end. Every harness component encodes an assumption about what the model can't do alone — re-test it when the model changes |
| Anthropic, effective harnesses for long-running agents | Two agents, an initializer and a coder. No separate QA or review agent |
| Cognition, what's actually working in multi-agent | Keep writes in a single thread; other agents only contribute judgment |

I wasn't going to transplant other people's numbers into my engine, though, so I measured.

### One task, three ways

A small task (a page whose circle turns from red to green when you press a button), run three ways, all on Haiku:

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/agent-engine-v6-retrospective/three-ways.en.svg"><img src="/assets/img/posts/agent-engine-v6-retrospective/three-ways.en.svg" width="720" height="250" loading="lazy" alt="The same task run three ways: v5 default took 11 calls and $1.545 and completed; v5 simplified took 14 calls and $1.908 and stopped after misunderstandings cascaded between roles; the v6 prototype took 3 calls and $0.402 and passed."></a></div><figcaption>One run each, on Haiku, on a small task. Even so, more roles were neither cheaper nor more accurate.</figcaption></figure>

| Approach | AI calls | Cost | Outcome |
|---|---:|---:|---|
| v5 default | 11 | $1.545 | completed |
| v5 simplified | 14 | $1.908 | stopped (misunderstandings cascading between roles) |
| v6 prototype (builder + evaluator) | 3 | $0.402 | passed |

The simplified v5 actually made *more* calls. One handoff went slightly wrong, and every role after it inherited the error and kept bouncing.

So I put each v5 component through the same question: what does it assume the model can't do alone, and is that still true?

| v5 component | Assumption | v6 |
|---|---|---|
| Separate design role | Builders can't design | Folded into the builder |
| Per-feature parallel work | Long work makes the model lose its way | One builder works through a batch in order |
| Fresh session per call | Full context breaks things | One session kept for the whole batch |
| 1–2 reviewers per round | A builder's self-assessment can't be trusted | One evaluator; a second view only for high-risk changes; one fresh-session rebuttal review right before pass |

The last row did not get cut. An agent declaring its own work done is still not trustworthy on today's models. That's the reason this engine exists, and it doesn't change with the model.

### Found while measuring: cost records inflated by 30%

The costs from those three runs looked off. I traced it back to the engine's own bookkeeping.

When a session is resumed, the cost the CLI reports is **cumulative for that session**. The engine was recording that value as "the cost of this call." Three calls on one session meant the first call's cost was counted three times.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/agent-engine-v6-retrospective/cost-record.en.svg"><img src="/assets/img/posts/agent-engine-v6-retrospective/cost-record.en.svg" width="720" height="260" loading="lazy" alt="Across three calls on a resumed session, the CLI reports a session-cumulative cost, so recording it as is stacks earlier calls on top of later ones. The correct per-call cost is the difference from the previous call. A record of $979.9 from an earlier engine version was really about $689."></a></div><figcaption>The CLI reports a session total. Per-call cost has to be the difference from the previous call.</figcaption></figure>

Recomputing a real project from the previous version (v4): $979.9 recorded, about $689 actual — roughly 30% over.

The worse part was the decision built on top of it. In September I had concluded that resuming a session costs more than starting fresh, and cut back on resuming. The resumed-session costs I'd been looking at were exactly these cumulative values. Recomputed from the raw responses:

| Role | Recorded avg, resumed | Actual, recomputed | Avg, fresh session (same role) |
|---|---:|---:|---:|
| Planning (19 calls) | $5.55 | $1.96 | $2.04 |

Resuming was never more expensive. A measurement bug had carried a design decision with it, and v6's "keep the session for the whole batch" only became possible after the recount. I measured cache lifetime while I was at it: a 20-minute idle gap still hit the cache, 65 minutes rebuilt the whole context. v6 now resumes the session regardless of any window; 50 minutes is only the threshold for flagging the cache as cold.

### The first real run: the engine delivered judgment, and got it wrong

The first real job on v6 was adding a test-only profile reset to a Roblox game.

The result made no sense at first. Across 9 evaluations the code had no defects, and all 9 scenarios passed three times over. Yet the same rejection kept coming back until it hit the rejection cap and stopped: 15 calls, $11.98.

The engine was the problem. When the rebuttal review raised findings, the engine sorted them by kind and decided who should receive them — and if any finding was a code defect, it sent *everything* to the builder. The actual finding was on the evaluation side ("there's no scenario that checks one of the completion criteria"), but it went to the builder. The builder had nothing to change, the review raised it again, and around it went three times.

My first instinct was to patch that routing rule. Then came the pushback: "You've drifted too far toward the engine — didn't you stray from the plan?" I compared the slimming plan to the code. Seven engine-side judgment mechanisms had crept in that the plan never had. Evaluation call types had grown from two to seven. The engine was classifying the evaluator's output, cross-checking it, asking back for ids, enforcing a minimum scenario count. I had removed roles, and engine rules had quietly filled the space they left.

So I wrote down the criteria first.

| # | Question | Belongs to |
|:-:|---|---|
| ① | Is it needed to run unattended (execute, wait, resume, record, cap)? | Engine |
| ② | Is it a fence against irreversible damage? | Engine (overrides ③④) |
| ③ | Does the engine interpret the agent's judgment, or split it into fields to route it? | The agent judges; the engine passes text verbatim |
| ④ | Did we patch an agent mistake by adding an engine rule? | Fix it in instructions or roles |
| ⑤ | Are we keeping an outside or legacy structure without answering "what can't today's model do alone?" | Remove it |

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/agent-engine-v6-retrospective/engine-vs-agent.en.svg"><img src="/assets/img/posts/agent-engine-v6-retrospective/engine-vs-agent.en.svg" width="720" height="300" loading="lazy" alt="Before alignment, the engine classified evaluator output by kind and routed it to the builder or evaluator, sending everything to the builder if any code defect was present. After alignment, the evaluator decides the next step itself (READY, REJECT, RUN, UNSURE) and the engine passes text through verbatim, runs scenarios, and counts."></a></div><figcaption>The evaluator judges. The engine passes text through, runs things, and counts.</figcaption></figure>

The evaluation stage was rewritten against that table. A single evaluator session picks the next step every time: READY, REJECT (with the text for the builder), RUN (scenarios), or UNSURE. The engine forwards that text untouched, runs the scenarios and returns results to the same session, and keeps the counts and caps. Classification, cross-checks, id re-asks, the scenario floor — all gone. About 170 lines came out (+551 / −722), 218 tests pass, and all 13 mutants are caught.

Resuming the stalled run, the evaluator added the missing check on its own (after a player leaves and the data saves, does the next session start from defaults?) and verified it from data. The moment a player leaves can't be captured on screen at all, so the evaluator flagged that part UNSURE, and the Master signed it off on the data evidence. The resumed portion took 5 calls, about $4.

### Summary

| Decision | Basis |
|---|---|
| Four roles → builder + evaluator | Measured on the same task: 11–14 calls → 3, about a quarter of the cost |
| Keep the session across the batch | "Resuming is expensive" rested on a cumulative-cost recording bug |
| Keep the fresh-session rebuttal review before pass | Self-declared "done" is still not trustworthy |
| The evaluator judges; the engine only runs things | A first real run where the engine routed judgment and got it wrong |

- Ask every component what today's model can't do without it. If there's no answer, the component is just cost.
- When a number looks wrong, suspect the measuring tool before the thing measured. This time the tool was wrong, and a decision was sitting on it.
- The only things an engine should hand over are facts: execution results, diffs, captures, test pass/fail. Once it starts interpreting judgment, cutting the roles was pointless.

The engine is Windows-only and its prompts and rule files are in Korean. The repo is at [github.com/sasostudio/agentworkflow](https://github.com/sasostudio/agentworkflow). The other half — watching and stopping the engine while it runs unattended — is in [the next post](/2026/10/15/unattended-engine-observe-and-stop/).

Further reading: [Anthropic — Harness design for long-running application development](https://www.anthropic.com/engineering/harness-design-long-running-apps) · [Anthropic — Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) · [Cognition — Multi-Agents: What's Actually Working](https://cognition.com/blog/multi-agents-working)
