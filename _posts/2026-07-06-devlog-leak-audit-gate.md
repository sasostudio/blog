---
layout: post
title: "'유출은 있다'고 가정하는 게시 파이프라인"
description: "build-in-public을 하면서 기밀이 글에 딸려 나가지 않게 만든 게시 파이프라인. 에이전트가 넉넉하게 초안을 뽑으면 사람이 고르고, 읽기 전용 감사 에이전트가 「유출은 있다」고 가정한 채 반박 검사를 한다."
date: 2026-07-06
tags: [claude-code, agents, build-in-public, devlog]
---

*🇬🇧 [English version below](#english-version)*

이 블로그는 이번 주에 열었다. 이건 두 번째 글이다.

블로그를 열자마자 부딪힌 문제는 글감이 아니었다. 나는 하루 종일 뭔가를 만든다 — 토스에 출시하는 게임, 로컬에서만 도는 개인 에이전트, 스튜디오 운영을 지탱하는 도구들. 글감은 넘친다. 문제는 **그 작업의 절반이 남에게 보여주면 안 되는 것들과 한 레포 안에 뒤엉켜 있다**는 거였다.

토스 심사·정산·mTLS 관련 기밀, 게임 밸런스의 정확한 수치, 자격증명 값, 로컬 절대경로. build-in-public을 하겠다면서 에이전트한테 "오늘 한 일 블로그 글로 써줘"라고 시키는 순간, 이것들 중 하나가 초안에 딸려 나올 가능성은 분명히 있다. 그리고 한 번 게시되면 삭제해도 캐시·인덱스에 남는다. 한번 저지르면 되돌릴 수 없는 실수다.

그래서 게시 자동화의 목표를 이렇게 잡았다: *글 쓰는 노동은 자동화하되, 에이전트가 기밀을 흘릴 수 없는 상태로.*

## 넘치게 쓰고, 내가 고른다

대부분의 "AI가 글 써주는" 흐름은 에이전트가 알아서 판단하게 둔다 — 뭘 쓸지, 뭘 뺄지, 어디까지 공개할지. 나는 그 판단권을 뺏었다.

에이전트는 **넉넉하게** 초안을 뽑는다. 최종본보다 많이, 공유 가능한 것들을 전부 항목으로 늘어놓는다. 그다음 나에게 **표**로 들이민다 — 각 항목마다 `무슨 내용 · 기술 깊이(상/중/하) · 민감도(🟢/🟡/🔴)`. 나는 편집자로서 [포함/제외/수정]만 지시하면 된다.

핵심은 **기술 깊이를 프로젝트 공개도에 비례**시킨 것이다. 오픈소스 프로젝트는 실제 코드·아키텍처까지 풀 깊이로, 상용·미출시 프로젝트는 "문제 → 접근 → 교훈" 수준까지만, 독점 튜닝값은 아예 제외. 에이전트가 "이 정도는 괜찮겠지"를 판단하지 않는다. 항목을 늘어놓고, 게이트에서 멈추고, 사람이 고르기 전엔 조립으로 넘어가지 않는다.

## git을 매번 뒤지지 않는다 — 비워지는 버퍼

처음엔 게시할 때마다 git 로그를 훑게 하려 했다. 두 가지 이유로 안 됐다.

하나, 나는 하루에 여러 세션으로 일한다. 세션마다 git을 재스캔하면 비싸다. 둘, **미커밋 작업은 git log에 안 잡힌다.** 특히 아직 출시 안 한 게임 쪽은 커밋 없이 로컬에서 굴러가는 작업이 많아서, git만 보면 "오늘 아무것도 안 함"으로 보인다.

그래서 영구 장부가 아니라 **비워지는 큐**를 뒀다. 어떤 세션이든 공유할 만한 일을 하면 월별 버퍼 파일에 두세 줄 남긴다 — `무엇을 왜 했나 · 민감도`. 자격증명 값이나 밸런스 정확값은 이 버퍼에도 안 적는다. 요약만. 게시 명령은 이 버퍼 한 곳만 읽고, 처리한 항목은 게시했든 스킵했든 제거한다. 로그는 항상 작게 유지되고, 영구 기록은 어차피 git 히스토리·게시된 글·프로젝트 문서에 남으니 버퍼는 비워도 안전하다.

## 유출은 "있다"고 가정하고 시작한다

조립이 끝나면 최종본은 **독립 감사 에이전트**한테 넘어간다. 이 에이전트의 프롬프트는 "유출 있는지 봐줘"가 아니다. **"유출은 있다고 가정하고, 없다는 걸 반박해봐라(refute)."** 기본값을 유죄로 놓고, 무죄를 입증하게 만든다.

감사자는 **읽기 전용**이다. 도구 권한에 Write/Edit가 없다. 이건 [지난 글에서 만든 AgentRoom](/2026/07/05/agentroom-human-gated-orchestration/)의 QA 에이전트와 똑같은 원칙이다. 검수자가 조용히 고쳐놓을 수 있으면 그건 검수가 아니다. 감사자는 문제를 찾아 보고만 하고, 수정은 본체가 한 뒤 **재감사**를 받는다. 통과 전엔 게시 안 된다.

검사는 4계층으로 나뉜다:

1. **🔴 자격증명** — API 키, 토큰, 서비스 계정 키, `.env` 값, Firebase/Supabase 설정
2. **🟠 토스 기밀** — 참고자료 출처의 결제·mTLS·심사·정산 관련 인용
3. **🟡 게임 밸런스 정확값** — 미출시 경쟁 스펙, 튜닝된 수치
4. **🟢 로컬 절대경로·이메일**

소스가 완전 공개 프로젝트뿐이면 기계적 grep 검사로 충분하지만, 민감 프로젝트가 섞이면 위의 독립 감사 에이전트를 강제 호출한다. 그리고 감사 결과(PASS / 플래그 목록)를 나에게 **함께** 보여준다. 감사를 몰래 통과시키는 걸 막기 위해서다.

## 같은 원칙, 다른 파이프라인

돌아보면 이건 지난 글과 완전히 같은 생각의 재탕이다.

AgentRoom에서 배운 건 두 가지였다 — **주장을 믿지 말고 상태를 검증하라**, 그리고 **되돌리기 어려운 일은 사람에게 물어라.** 게시 파이프라인은 그 둘을 그대로 옮겨왔다. 에이전트의 "안전합니다"를 믿는 대신 독립 감사자가 실제 텍스트를 검증하고(주장 불신), 게시·커밋·삭제는 절대 자동 실행하지 않고 내가 "올려"라고 할 때만 명령을 실행한다(되돌리기 어려운 일은 사람 게이트). 게시는 삭제해도 캐시에 남는, 가장 되돌리기 어려운 행동이니까.

그리고 정직하게 밝히자면, 당신이 지금 읽고 있는 이 글도 게시되기 전에 그 감사 게이트를 통과했다. 유출은 있다고 가정하고, 없다는 걸 반박당한 뒤에.

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
