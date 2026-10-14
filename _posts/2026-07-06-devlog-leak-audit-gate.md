---
layout: post
title: "'유출은 있다'고 가정하는 게시 파이프라인"
description: "build-in-public 과정에서 기밀이 글에 딸려 나가지 않도록 만든 게시 파이프라인. 에이전트의 넉넉한 초안, 사람의 선별, 「유출은 있다」를 전제로 한 읽기 전용 감사 에이전트의 반박 검사까지."
date: 2026-07-06
tags: [claude-code, agents, build-in-public, devlog]
---

*🇬🇧 [English version below](#english-version)*

이 블로그의 개설 시점은 이번 주. 이 글은 두 번째 글.

블로그 개설 직후 부딪힌 문제는 글감 부족이 아니었다는 점. 하루 종일 무언가를 만드는 생활 — 토스에 출시하는 게임, 로컬에서만 도는 개인 에이전트, 스튜디오 운영을 지탱하는 도구들. 글감은 충분. 문제는 **그 작업의 절반이 남에게 보여 주면 안 되는 것들과 한 레포 안에 뒤엉켜 있다**는 점.

토스 심사·정산·mTLS 관련 기밀, 게임 밸런스의 정확한 수치, 자격증명 값, 로컬 절대경로. build-in-public을 하겠다며 에이전트에게 "오늘 한 일 블로그 글로 써줘"라고 시키는 순간, 이 중 하나가 초안에 딸려 나올 가능성은 분명히 존재. 그리고 한 번 게시되면 삭제해도 캐시·인덱스에 잔존. 한번 저지르면 되돌릴 수 없는 실수.

그래서 정한 게시 자동화의 목표: *글 쓰는 노동은 자동화하되, 에이전트가 기밀을 흘릴 수 없는 상태로.*

## 넘치게 쓰고, 사람이 선별

대부분의 「AI 글쓰기」 흐름은 판단을 에이전트에게 위임 — 무엇을 쓸지, 무엇을 뺄지, 어디까지 공개할지. 이 파이프라인에서는 그 판단권을 회수.

에이전트의 역할은 **넉넉한** 초안 작성. 최종본보다 많이, 공유 가능한 것들을 전부 항목으로 나열. 그다음 **표**로 제시. 항목별 구성은 `무슨 내용 · 기술 깊이(상/중/하) · 민감도(🟢/🟡/🔴)` 세 칸. 사람의 역할은 편집자로서 [포함/제외/수정] 지시뿐.

핵심은 **기술 깊이와 프로젝트 공개도의 비례**. 오픈소스 프로젝트는 실제 코드·아키텍처까지 풀 깊이, 상용·미출시 프로젝트는 「문제 → 접근 → 교훈」 수준까지만, 독점 튜닝값은 전면 제외. 「이 정도는 괜찮겠지」 판단을 에이전트에게 맡기지 않는 구조. 항목 나열, 게이트에서 정지, 사람의 선별 전에는 조립 단계 진입 불가.

## git 재탐색 대신 비워지는 버퍼

처음 구상은 게시 때마다 git 로그를 훑는 방식. 두 가지 이유로 불채택.

첫째, 하루에 여러 세션으로 작업하는 환경. 세션마다 git 재스캔은 고비용. 둘째, **미커밋 작업은 git log에 미포착.** 특히 미출시 게임 쪽은 커밋 없이 로컬에서 진행하는 작업이 많아, git만 보면 「오늘 작업 0」으로 오인.

그래서 영구 장부가 아니라 **비워지는 큐**를 도입. 어떤 세션이든 공유할 만한 작업을 하면 월별 버퍼 파일에 두세 줄 기록 — `무엇을 왜 했나 · 민감도`. 자격증명 값이나 밸런스 정확값은 버퍼에도 기록 금지, 요약만 허용. 게시 명령은 이 버퍼 한 곳만 읽고, 처리한 항목은 게시·건너뜀과 무관하게 삭제. 로그는 항상 소규모 유지, 영구 기록은 어차피 git 히스토리·게시된 글·프로젝트 문서에 잔존하므로 버퍼 비우기도 안전.

## 「유출은 있다」 전제의 감사

조립이 끝난 최종본은 **독립 감사 에이전트**에게 전달. 이 에이전트의 프롬프트는 "유출 있는지 봐줘"가 아니라 **"유출은 있다고 가정하고, 없다는 걸 반박해봐라(refute)."** 기본값을 유죄로 두고 무죄 입증을 요구하는 구조.

감사자는 **읽기 전용**. 도구 권한에 Write/Edit 부재. [지난 글의 AgentRoom](/2026/07/05/agentroom-human-gated-orchestration/) QA 에이전트와 같은 원칙. 검수자가 조용히 고쳐 놓을 수 있다면 그건 검수가 아니라는 판단. 감사자는 문제 발견·보고만 담당, 수정은 본체가 한 뒤 **재감사**. 통과 전 게시 불가.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/devlog-leak-audit-gate/pipeline.ko.svg"><img src="/assets/img/posts/devlog-leak-audit-gate/pipeline.ko.svg" width="720" height="300" loading="lazy" alt="작업로그 버퍼에서 글감 메뉴 표를 작성, 관리자가 고른 것만 조립. 초안은 유출 전제의 읽기 전용 독립 감사로 전달, 플래그가 나오면 본체가 수정 후 재감사. 감사 결과를 관리자에게 함께 제시, 관리자의 게시 지시가 있을 때만 게시."></a></div><figcaption>에이전트는 넉넉한 초안 작성만, 선별과 게시 결정은 사람 담당. 그 사이에 유출 전제의 감사.</figcaption></figure>

검사 대상은 4계층:

1. **🔴 자격증명** — API 키, 토큰, 서비스 계정 키, `.env` 값, Firebase/Supabase 설정
2. **🟠 토스 기밀** — 참고자료 출처의 결제·mTLS·심사·정산 관련 인용
3. **🟡 게임 밸런스 정확값** — 미출시 경쟁 스펙, 튜닝된 수치
4. **🟢 로컬 절대경로·이메일**

소스가 완전 공개 프로젝트뿐이면 기계적 grep 검사로 충분, 민감 프로젝트가 섞이면 독립 감사 에이전트를 강제 호출. 그리고 감사 결과(PASS / 플래그 목록)를 사람에게 **함께** 제시. 감사의 몰래 통과를 막는 장치.

## 같은 원칙, 다른 파이프라인

돌아보면 지난 글과 완전히 같은 발상의 재적용.

AgentRoom에서 얻은 교훈은 두 가지 — **주장을 믿지 말고 상태를 검증하라**, 그리고 **되돌리기 어려운 일은 사람에게 물어라.** 게시 파이프라인은 그 둘을 그대로 이식. 에이전트의 "안전합니다"를 믿는 대신 독립 감사자가 실제 텍스트를 검증(주장 불신), 게시·커밋·삭제는 자동 실행 없이 사람의 "올려" 지시가 있을 때만 실행(되돌리기 어려운 일은 사람 게이트). 게시는 삭제해도 캐시에 남는, 가장 되돌리기 어려운 행동이기 때문.

그리고 정직한 고지: 지금 읽는 이 글도 게시 전에 그 감사 게이트를 통과한 결과물. 유출이 있다는 가정에서 출발해, 반박을 거친 뒤의 게시.

---

## English version

I opened this blog this week. This is the second post.

The problem I hit right after opening it wasn't a shortage of topics. I build things all day — a game I ship to Toss, a personal agent that runs only on my machine, tools that keep the studio running. There's plenty to write about. The problem was that **half of that work is tangled, inside a single repo, with things that must not be shown to anyone.**

Toss review/settlement/mTLS confidential material, exact game-balance numbers, credential values, local absolute paths. The moment you tell an agent "turn today's work into a blog post" as your build-in-public routine, the odds that one of those slips into the draft aren't zero. And once something is published, deleting it doesn't remove it from caches and indexes. It's the irreversible kind of mistake.

So I set the goal of publishing automation like this: *automate the writing labor — but in a state where the agent cannot leak a secret.*

### Overflow the draft, then I pick

Most "AI writes it for you" flows let the agent decide — what to write, what to cut, how far to go public. I took that decision away.

The agent drafts **generously**. More than the final post, laying out every shareable item as a row. Then it hands me a **table** — for each item, `what it is · technical depth (high/med/low) · sensitivity (🟢/🟡/🔴)`. As the editor, I only issue [include / exclude / edit].

The key move is making **technical depth proportional to how public the project is.** Open-source projects get full depth, down to actual code and architecture; commercial or unreleased projects only get "problem → approach → lesson"; proprietary tuning values are excluded entirely. The agent doesn't judge "this much is probably fine." It lays out the items, stops at a gate, and doesn't move to assembly until a human has picked.

### It doesn't dig through git every time — a buffer that gets drained

At first I had it scan the git log on every publish. That failed for two reasons.

One, I work in multiple sessions a day; re-scanning git per session is expensive. Two, **uncommitted work doesn't show up in git log.** The unreleased game especially has a lot of work that runs locally without commits, so git alone makes it look like "did nothing today."

So instead of a permanent ledger, I keep a **queue that gets emptied.** Whenever any session does something shareable, it leaves two or three lines in a monthly buffer file — `what I did and why · sensitivity`. Credential values and exact balance numbers don't go in this buffer either. Summaries only. The publish command reads only this one buffer, and removes each item it processed, whether it got published or skipped. The log stays small, and since the permanent record lives in git history, published posts, and project docs anyway, the buffer is safe to empty.

### It starts by assuming the leak is already there

Once assembly is done, the final draft goes to an **independent audit agent.** Its prompt isn't "check whether there's a leak." It's **"assume a leak exists, and try to refute that there isn't one."** The default is guilty; innocence has to be proven.

The auditor is **read-only.** Its tool permissions don't include Write or Edit. This is the exact same principle as the QA agent in [AgentRoom, from the last post](/2026/07/05/agentroom-human-gated-orchestration/) — if a reviewer can quietly fix things, it isn't a review. The auditor only finds and reports; the main process fixes, then submits for a **re-audit.** Nothing publishes until it passes.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/devlog-leak-audit-gate/pipeline.en.svg"><img src="/assets/img/posts/devlog-leak-audit-gate/pipeline.en.svg" width="720" height="300" loading="lazy" alt="Items from the worklog buffer become a menu table; only the ones I pick get assembled. The draft goes to a read-only independent audit that assumes a leak exists; flags go back for fixes and a re-audit. The audit result is shown to me too, and it ships only when I say so."></a></div><figcaption>The agent only over-drafts; picking and publishing stay with a human, with a leak-by-default audit in between.</figcaption></figure>

The check runs in four layers:

1. **🔴 Credentials** — API keys, tokens, service account keys, `.env` values, Firebase/Supabase config
2. **🟠 Toss confidential** — payment/mTLS/review/settlement references sourced from internal material
3. **🟡 Exact game-balance values** — unreleased competitive specs, tuned numbers
4. **🟢 Local absolute paths and email**

If the sources are all fully-public projects, a mechanical grep pass is enough; but if any sensitive project is mixed in, the independent audit agent above is invoked, mandatorily. And it shows me the audit result (PASS / list of flags) **alongside** the draft — to stop the audit from being quietly waved through.

### Same thesis, different pipeline

Looking back, this is a straight rerun of the same idea as the last post.

AgentRoom taught me two things — **don't trust claims, verify state**, and **stop and ask the human for anything hard to reverse.** The publishing pipeline just ports both over. Instead of trusting the agent's "it's safe," an independent auditor verifies the actual text (distrust the claim); and publish/commit/delete never run automatically — the command only executes when I say "ship it" (human-gate the irreversible). Publishing is the least reversible action there is: delete it and it still lives in a cache.

And to be honest about it — this very post you're reading passed through that audit gate before it went up. After the leak was assumed to exist, and refuted.
