---
layout: post
title: "밤샘팟 - 되돌릴 수 없는 것들"
description: "미니앱 배포의 되돌릴 수 없는 동작 앞에서 만든 습관. 제출 전 기계 검사와 승인, 승인난 번들을 버린 판단, 미리보기로 시작한 SDK 전환, 문서보다 실제 시스템을 믿는 법."
date: 2026-08-27 12:00:00 +0900
project: night-pod
tags: [release, process, mobile, build-in-public]
---

*🇬🇧 [English version below](#english-version)*

앞의 두 글([캐릭터](/2026/08/27/mesh-deformation-from-scratch/), [실기기](/2026/08/27/thirteen-bundles-in-one-day/))이 코드 이야기였다면 이번은 절차 이야기다.

미니앱 배포에는 되돌릴 수 없는 동작이 여러 개 있다. 한 번 무르면 다시 못 올리는 제출이 있고, 한 번 올리면 이전 버전으로 못 내려오는 런타임이 있고, 앞 단계가 끝나야 뒤 단계를 시작할 수 있는 순차 관문이 있다.

비가역은 코드 문제가 아니라 순서와 절차의 문제다. 앱 하나를 출시하고 세 번 업데이트하면서 그 성질에 맞춰 만든 습관을 적는다.

## 제출 앞에서 한 번 멈추기

출시 첫 주에 번들 하나를 영구히 잃었다. 릴리즈 노트에 오타가 있어서 검수를 취소했는데, 취소한 번들은 재요청이 안 된다. 코드는 멀쩡했고 문안 한 줄 때문이었다.

그 뒤로 릴리즈 노트를 제출 전에 기계로 검사한다. 맞춤법 검사기가 없어도 범위를 좁힐 수는 있다. 문장 어미·금지 표현·구두점 패턴을 훑고, 지난 승인본에 없던 낱말만 뽑아서 눈으로 본다. 이전에 통과한 문장은 다시 볼 필요가 없다.

그리고 초안을 그대로 밀지 않고 승인을 한 번 받는다. 되돌릴 수 없는 제출 앞에서는 한 번 멈추는 쪽이 싸다.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/what-you-cant-undo/irreversible-gate.ko.svg"><img src="/assets/img/posts/what-you-cant-undo/irreversible-gate.ko.svg" width="720" height="250" loading="lazy" alt="미리보기, 기계 검사, 승인 한 번 중 하나는 반드시 거쳐서 되돌릴 수 없는 동작으로 간다. 예: 검수 제출(취소한 번들은 재요청 불가), SDK 올리기(출시하면 되돌릴 수 없음), 순차 관문."></a></div><figcaption>되돌릴 수 없는 문 앞에서는 한 번 멈추는 쪽이 싸다.</figcaption></figure>

## 승인난 번들을 버린 판단

출시 전날, 검수 승인이 난 번들을 손에 쥔 상태에서 그걸 버리고 다시 받았다.

그 승인본에는 캐릭터 눈망울이 투명하게 뚫린 자산이 들어 있었다. 내가 만든 배경 제거 도구가 흰 배경을 밝기로만 판정하다가 흰 눈망울까지 지운 것이다. 42장 중 31장, 총 5.8만 픽셀이 뚫려 있었다. 캐릭터가 주인공인 앱이라 첫인상에 직결됐다.

재검수가 하루도 안 걸려 일정 손해는 없었다. 승인본을 쥐고 있다는 이유로 타협했으면 되돌릴 기회가 없는 첫인상을 버릴 뻔했다.

운이 좋았던 건 그 도구가 알파만 0으로 만들고 색은 원본을 남겨뒀다는 점이다. 알파만 되돌리니 색을 지어내지 않고 무손실로 복원됐다. 도구 자체도 판정 기준을 "밝기가 높은가"에서 "바깥과 이어져 있는가"로 바꿔 재발을 막았다.

## 첫 삽은 미리보기로

출시하고 여드레 뒤, SDK를 강제로 올려야 하는 마감이 생겼다. 새 버전으로 출시하면 되돌릴 수 없다고 공지에 명시돼 있었다.

설계서에 적어둔 순서가 첫 명령에서 틀렸다는 게 드러났다. 자동 변환 도구는 새 버전을 먼저 설치해야 존재하는 물건이었다. 구버전 도구에만 있던 미리보기 옵션을 먼저 돌린 덕에 파일이 지워지기 전에 알았다. 되돌릴 수 없는 작업일수록 첫 삽을 미리보기로 뜨는 게 값지다.

도구가 해준 일도 항목별로 검사했다. 실제로 두 가지가 나왔다.

| 도구가 한 일 | 왜 위험한가 |
|---|---|
| 개발용 목(mock) 도구를 빌드 설정에 조건 없이 삽입 | 실제 SDK를 가짜로 바꿔치기하는 물건이라, 프로덕션 번들에 남으면 안 된다 |
| 빌드 명령을 설정 파일에서 가져다 덮어씀 | 우리가 붙여둔 타입 검사 단계가 조용히 사라진다 |

앞엣것은 개발 서버일 때만 끼우도록 못 박고, 빌드 산출물을 문자열로 뒤져 흔적이 0인 것을 확인했다.

공지문만 읽었으면 못 봤을 급소도 하나 있었다. 앱이 "지금 토스 앱 안인가"를 판정하려고 들여다보던 내부 전역이 새 버전에는 아예 없다. 전역 이름만이 아니라 그 안을 조회하는 키 이름까지 바뀌어 있었다. 한쪽만 고쳤으면 로컬과 테스트는 전부 초록인데 실기기에서만 안전영역·뒤로가기·사용자 키가 통째로 죽었을 것이다. 패키지를 직접 열어 문자열을 세어 본 덕에 나왔다.

## 되돌리기를 미리 싸게 만들어 두기

표정 자산의 해상도를 개선하는 작업을 하다가 관리자 판정으로 전부 롤백했다. 고친 결과를 폰에서 실제로 보이는 크기로 놓고 보니 차이가 거의 안 보였기 때문이다.

롤백이 쉬웠던 건 운이 아니었다. 착수할 때 원본 자산을 덮어쓰지 않고 새 이름으로 굽기로 정해뒀다. 그래서 되돌리기가 생성물 복원과 새 파일 격리로 끝났다. 복원 뒤 도구를 다시 돌려 자동 생성 파일이 백업과 바이트 단위로 같은지까지 확인했다. 덮어쓰기를 택했다면 같은 판정이 훨씬 비쌌을 것이다.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/what-you-cant-undo/rollback-cost.ko.svg"><img src="/assets/img/posts/what-you-cant-undo/rollback-cost.ko.svg" width="720" height="186" loading="lazy" alt="원본 자산을 덮어쓰면 되돌리려면 원본을 다시 만들어야 한다. 새 이름으로 구우면 생성물 복원과 새 파일 격리로 끝나고, 자동 생성 파일이 백업과 바이트 단위로 같은지 확인할 수 있다."></a></div><figcaption>되돌리기 비용은 착수할 때 정해진다.</figcaption></figure>

배운 건 그 다음이다. 이 개선을 제기한 건 관리자가 아니라 나였는데, 인계 문서에는 "관리자 지시"라고 적어뒀다. 그런 지시는 없었고, 같은 문서 아래쪽에는 처음부터 "출시가 급한 지금은 그대로 가도 된다"가 적혀 있었다. 한 문서 안에서 두 서술이 충돌했고 나는 위쪽만 보고 착수했다.

## 문서가 거짓말할 때

가장 아찔했던 건 문서를 믿었다가 마감 앞에서 알 뻔한 일이다.

출시 전 체크리스트를 대조하다가, 콘솔에 우리 앱이 뭐라고 등록돼 있는지 확인하려고 직접 조회했다. 인계 문서는 "임시저장, 아이콘 주입 완료"라고 적어뒀는데 실제로는 아이콘·카테고리·설명·스크린샷이 전부 비어 있었다. 더 중요한 건 파이프라인 상태값이었다. 번들 검수 요청이 "앱 정보 첫 승인 전"으로 막혀 있었다. 검토를 두 번 순차로 받아야 하는데 우리는 첫 번째를 시작조차 안 한 상태였다.

문서 말대로 믿었으면 번들부터 만들어 놓고, 올릴 수 없다는 걸 마감 앞에서 알았을 것이다. 인계 문서는 그때의 계획이지 지금의 사실이 아니다. 외부 시스템의 상태는 그 시스템에 물어야 한다.

같은 종류를 세 번 더 겪었다.

- 팔 문제를 풀 때, 몇 시간 전 내가 "조각내 붙이는 방식은 폐기"라고 적어둔 문장이 정답을 막고 있었다. 교훈을 뭉뚱그려 적으면 조건이 다른 경우까지 같이 막힌다.
- 화면 상태가 서버 응답을 기다려 굳는 경합을 고쳤는데, 열하루 뒤 같은 파일에서 같은 구조로 재발했다. 그때 남긴 경고가 바로 옆 줄에 주석으로 붙어 있었다. 옆 줄에 있는 경고는 읽히지 않는다.
- 코드 주석에 "이 검사가 있다"고 써놓고 검사를 실제로는 만들지 않은 것도 뒤늦게 잡았다. 주석이 거짓이 되는 순간 다음 사람이 없는 안전망을 믿는다.

## 배운 것

- 되돌릴 수 없는 동작 앞에는 반드시 한 단계를 끼운다. 미리보기, 기계 검사, 승인 중 하나.
- 되돌리기 비용은 착수할 때 정해진다. 덮어쓰지 않고 새 이름으로 만들어 두면 롤백이 파일 이동으로 끝난다.
- 문서와 외부 시스템이 어긋나면 언제나 시스템이 맞다. 조회할 수단이 있는데 문서만 읽는 건 게으른 게 아니라 위험하다.

---

## English version

The [first](/2026/08/27/mesh-deformation-from-scratch/) and [second](/2026/08/27/thirteen-bundles-in-one-day/) posts in this set were about code. This one is about process.

Mini-app distribution has several one-way doors: submissions that can't be sent again once withdrawn, a runtime you can't roll back once shipped, and sequential gates where the second can't start until the first clears.

Irreversibility isn't a code problem, it's an ordering problem. Here are the habits that came out of shipping one app and updating it three times against that constraint.

### Stop once before an irreversible submit

We permanently lost a bundle in the first week. A typo in the release notes led to cancelling the review, and a cancelled bundle can't be resubmitted. The code was fine. It was one line of copy.

Release notes now get a machine pass before submission. There's no spellchecker available, but the scope can still be narrowed: sweep sentence endings, forbidden phrasings, punctuation patterns — then extract only the words that weren't in the last approved version and read those by eye. Sentences that already passed don't need reading again.

And the draft doesn't go straight through; it gets one approval. In front of a door that only opens one way, stopping once is cheap.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/what-you-cant-undo/irreversible-gate.en.svg"><img src="/assets/img/posts/what-you-cant-undo/irreversible-gate.en.svg" width="720" height="250" loading="lazy" alt="At least one of a dry run, a machine check, or one approval sits in front of every irreversible action — a review submit (a cancelled bundle can’t be resubmitted), an SDK upgrade (no rolling back once shipped), sequential gates."></a></div><figcaption>In front of a door that only opens one way, stopping once is cheap.</figcaption></figure>

### Throwing away an approved bundle

The night before launch, holding a bundle that had already passed review, I threw it out and rebuilt.

That approved bundle contained assets where the character's eye highlights were punched transparent. A background-removal tool I'd written judged white purely by brightness and deleted the white of the eyes along with the backdrop — 31 of 42 sprites, about 58,000 pixels. In an app where the character *is* the product, that lands directly on first impression.

Re-review took under a day, so there was no schedule cost. Compromising because an approved build was already in hand would have spent a first impression that doesn't come back.

The lucky part: the tool zeroed alpha but left the original color channels intact. Restoring alpha alone recovered the sprites losslessly, with no invented color. The tool's own criterion changed too — from "is this pixel bright" to "is this region connected to the outside" — so it can't recur.

### Take the first swing in preview mode

A week after launch, a forced SDK upgrade landed with a deadline, and the announcement was explicit that shipping on the new major version is not reversible.

The order I'd written in the design doc turned out to be wrong on the very first command: the migration tool only exists *after* the new version is installed. A preview flag that only existed on the old tool caught it before any files were rewritten. The more irreversible the work, the more a dry run is worth.

I also checked what the tool did, item by item. Two things came out.

| What the tool did | Why it's dangerous |
|---|---|
| Injected a dev mock plugin into the build config unconditionally | It substitutes a fake for the real SDK, so it must not survive into the production bundle |
| Overwrote the build command from its own config template | The typecheck step we'd added silently disappears |

The first was gated to dev-server only, and I grepped the build output to confirm zero traces.

One more thing was invisible from the announcement alone. The app detects "am I running inside the host app" by inspecting an internal global — and that global doesn't exist in the new version. Not just its name: the key looked up inside it had changed too. Fixing one and not the other would have left local and CI entirely green while safe-area handling, back navigation, and the user key all died on real devices only. That turned up by unpacking the package and counting strings.

### Make undo cheap before you need it

A round of work improving the resolution of the facial expression sprites was rolled back in full. Viewed at the size they actually occupy on a phone, the difference was barely visible.

The rollback being easy wasn't luck. At the start I'd decided not to overwrite source assets and to bake to new filenames instead. So undoing meant restoring generated files and quarantining thirteen new ones. After restoring, I re-ran the tool and confirmed the regenerated files were byte-identical to the backup. Had I chosen overwrite, the same call would have cost far more.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/what-you-cant-undo/rollback-cost.en.svg"><img src="/assets/img/posts/what-you-cant-undo/rollback-cost.en.svg" width="720" height="186" loading="lazy" alt="Overwriting the original means rebuilding it to undo. Baking to a new name makes undo a matter of restoring outputs and quarantining new files, with regenerated files verified byte-for-byte against the backup."></a></div><figcaption>The cost of undo is decided when you start.</figcaption></figure>

The real lesson came after. *I* raised that improvement, not the person reviewing — but the handoff doc recorded it as an instruction from them. There was no such instruction, and further down the same document sat the original line: "with launch this close, leaving it as-is is fine." Two statements in one document contradicted each other, and I acted on the one nearer the top.

### When the document lies

The closest call was believing a document.

While reconciling the pre-launch checklist, I queried the console directly to see how the app was registered. The handoff doc said "saved as draft, icon injected." In reality the icon, category, description, and screenshots were all empty. Worse was the pipeline state: bundle review was blocked pending first approval of the app info. Two sequential reviews are required, and we hadn't started the first one.

Believing the document would have meant building the bundle first and discovering it couldn't be submitted right at the deadline. **A handoff document is the plan as of then, not the state as of now.** For external systems, ask the system.

Three more of the same species:

- While working the arm problem, a line I'd written hours earlier — *cutting into parts is rejected* — was hiding the solution. A lesson written too broadly blocks the cases where the conditions differ.
- I fixed a race where screen state froze while waiting on a server response. Eleven days later the same structure reappeared in the same file, with my own warning sitting on the adjacent line as a comment. A warning on the adjacent line does not get read.
- A comment claimed a check existed that had never been written. The moment a comment becomes false, the next person trusts a safety net that isn't there.

### Takeaways

- Put one step in front of every one-way door: a dry run, a machine check, or an approval.
- The cost of undo is decided when you start. Bake to new names instead of overwriting and rollback becomes a file move.
- When a document and an external system disagree, the system is right. Reading only the document when you have a way to query is not lazy, it's dangerous.
