---
layout: post
title: "클라이언트를 믿지 않는 대전 설계"
description: "퍼즐 게임에 사람 대 사람 대전을 붙이며 세운 원칙은 「클라이언트의 말은 단서일 뿐 증거가 아니다」였다. 자기 패배만 신고하는 판정, ID로 방장을 정하는 매칭, 트로피와 같은 트랜잭션 안의 정산 원장으로 풀었다."
date: 2026-07-27 10:00:00 +0900
project: moongchi
tags: [game-dev, multiplayer, firebase, build-in-public]
---

*🇬🇧 [English version below](#english-version)*

직접 만드는 퍼즐 게임에 1대1 대전을 붙였다. 처음 상대는 봇이었다. 봇전은 크게 어렵지 않았다. 상대의 모든 것이 내 기기 안에서 돌아가니까, 누가 이겼는지도 결국 내 코드가 알고 있었다.

<figure class="shot"><img src="/assets/img/posts/you-cant-report-your-own-win/moongchi-play.jpg" width="440" height="751" loading="lazy" alt="뭉치뭉치 플레이 화면 — 나무 틀 보드 안에서 동물 캐릭터들이 합쳐지며 BOOM 콤보가 터지는 장면"><figcaption>이 글의 게임, 뭉치뭉치. 여기에 사람 대 사람 대전을 붙였다.</figcaption></figure>

사람 대 사람으로 넘어가는 순간 그 전제가 통째로 무너졌다. 이제 상대는 남의 폰에서 돌아가고, 그 폰이 뭐라고 말하든 나는 그걸 확인할 방법이 없다. 지난 3주는 대부분 이 한 문장을 감당하는 일이었다.

## 이긴 사람이 이겼다고 말하면 안 된다

제일 먼저 정한 규칙은 단순하다. 자기 승리는 신고할 수 없고, 자기 패배만 신고할 수 있다.

이유는 뻔하다. "내가 이겼다"를 클라이언트가 보낼 수 있으면 그 메시지 하나가 곧 랭킹 조작 버튼이 된다. 반대로 "내가 졌다"는 굳이 위조할 이유가 없다. 그래서 각자는 자기 패배만 말하고, 승리는 그 반대편에서 파생된다.

그런데 이것만으로는 부족했다. 대전이 끝나는 경로가 하나가 아니기 때문이다. 한쪽 보드가 꽉 차서 끝날 수도 있고, 제한 시간이 다 될 수도 있고, 한쪽이 나가버릴 수도 있다. 경로마다 두 기기가 서로 조금씩 다른 시점에 결론을 낸다. 각자 자기 결론을 자기 화면에 그리면, 같은 경기를 두고 두 사람이 다른 결과를 보는 사태가 열린다.

그래서 결과를 각자 판단하지 않게 만들었다. 어느 경로로 끝나든 확정은 두 기기가 공유하는 결과 자리를 반드시 거친다. **먼저 그 자리에 남은 기록이 그 경기의 결론이고, 늦게 도착한 쪽은 자기 판단을 버리고 그 결론을 읽어서 표시한다.** 양쪽 화면이 어긋날 수 없는 구조가 된다.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/you-cant-report-your-own-win/result-slot.ko.svg"><img src="/assets/img/posts/you-cant-report-your-own-win/result-slot.ko.svg" width="720" height="330" loading="lazy" alt="기기 A와 기기 B가 각자 결론을 공유 결과 자리로 보낸다. 먼저 도착한 기록 1건이 결론이 되고, 늦게 도착한 쪽은 자기 판단을 버린다. 두 화면은 같은 기록을 읽어 A는 패배, B는 승리를 표시한다."></a></div><figcaption>어느 경로로 끝나든 결론은 공유 결과 자리에 먼저 남은 1건이다. 승리는 신고하는 게 아니라 상대의 패배 신고에서 나온다.</figcaption></figure>

나머지는 경로별 예외 처리였다. 두 보드가 사실상 동시에 끝나면 사망 시각을 비교해 다시 판정하고, 시간 종료는 상대 보고를 기다리되 앱이 백그라운드로 내려가 시계가 밀리는 경우를 대비해 기다림에 절대 상한을 뒀다. 한쪽이 나가면 잠깐 기다린 뒤 남은 사람의 승리로 넘겼다. 이 이탈 처리는 나중에 실기기 두 대 검증에서 크게 깨져 전면 재설계하게 되는데, 그건 다음 글에서 따로 다룬다.

## 방은 누가 만드는가

다음은 자동 매칭이었다. 방 코드를 주고받는 사설 대전은 이미 돌고 있었지만, 아무나 눌러도 상대가 붙는 랭크 매치가 필요했다.

문제는 나에게 매치메이커 서버가 없다는 것이다. 대기열을 실시간 DB에 두고 클라이언트끼리 붙게 하면, 둘이 동시에 서로를 발견하는 순간이 온다. 둘 다 방을 만들면 방이 두 개 생기고 각자 빈 방에서 상대를 기다린다. 전형적인 경쟁 상태(race condition)다.

해결은 중재자를 두는 대신 결론이 하나로만 나오게 만드는 쪽이었다. **두 사람의 ID 중 작은 쪽이 방을 만든다.** 쌍마다 유일하게 정해지는 값이라 누가 먼저 상대를 발견했든 결론이 같다. 작은 쪽이 방을 만들고 자기 대기열 항목에 방 번호를 적으면, 큰 쪽이 그걸 읽고 들어가면서 입장 흔적을 남긴다. 경합할 여지 자체가 사라진다.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/you-cant-report-your-own-win/matching.ko.svg"><img src="/assets/img/posts/you-cant-report-your-own-win/matching.ko.svg" width="720" height="400" loading="lazy" alt="플레이어 A(ID 작음), 대기열, 플레이어 B(ID 큼)의 순서도. 거의 동시에 서로를 발견한 뒤 A만 방을 만들어 자기 항목에 방 번호를 적고, B는 그 번호를 읽어 입장하며 입장 흔적을 남긴다."></a></div><figcaption>규칙이 쌍마다 하나의 답을 내므로, 누가 먼저 상대를 발견했든 방은 하나만 생긴다.</figcaption></figure>

권한도 같이 잠갔다. 대기열에서는 자기 항목만 쓸 수 있다. 남의 항목을 건드릴 수 있으면 남의 매칭을 가로채거나 망가뜨릴 길이 열리니까.

사실 첫 설계는 상대 항목에 플래그를 심어 넘기는 방식이었는데, 방금 그 권한 규칙과 정면으로 부딪혔다. 검수에서 반려됐고, 그때 지금의 방식으로 전면 교체했다. 권한을 먼저 못 박아두니 설계 오류가 스스로 드러난 셈이다.

한 가지 더 있었다. 대기자는 실력 구간으로 나눠 조회하는데, 구간 경계를 사이에 두고 선 두 사람은 서로를 영영 못 본다. 실력이 거의 같은데도 그렇다. 자기 구간뿐 아니라 인접 구간까지 함께 조회하도록 넓혀서 메웠다.

## 마커와 보상이 다른 곳에 있으면

가장 오래 붙잡은 건 트로피 정산이었다.

랭크 경기가 끝나면 서버가 양쪽 트로피를 조정한다. 여기서 같은 경기가 두 번 정산되면 랭킹 전체가 부풀어버린다. 그래서 "이 경기는 정산됐다"는 마커를 남기고, 이미 있으면 아무것도 하지 않게 만들려 했다.

내가 처음 짠 건 마커를 실시간 DB에, 트로피 변경을 문서 DB에 두는 구조였다. 검수가 이걸 두 번 반려했다. 그때는 좀 답답했는데, 지적이 정확했다.

서로 다른 저장소라 하나의 트랜잭션으로 묶을 수가 없다. 두 쓰기 사이에서 함수가 죽으면 둘 중 하나가 벌어진다. 마커만 남고 트로피가 안 들어갔으면 그 경기는 재시도해도 "이미 처리됨"으로 걸러져 영영 미지급이다. 트로피만 들어가고 마커가 안 남았으면 재시도할 때마다 또 들어간다. 멱등성을 지키려고 넣은 장치가, 놓인 위치 때문에 미지급과 중복 지급을 동시에 열어둔 것이다.

고친 방식은 마커를 효과와 같은 곳으로 옮기는 것이었다. 정산 원장을 트로피와 같은 DB에 두고, 하나의 트랜잭션 안에서 원장을 먼저 읽는다. 이미 있으면 아무것도 하지 않고 끝낸다. 없으면 양쪽 트로피 변경과 원장 기록을 한 번에 커밋한다. 재시도해도 원장이 이미 있으니 조용히 아무 일도 일어나지 않는다.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/you-cant-report-your-own-win/settlement.ko.svg"><img src="/assets/img/posts/you-cant-report-your-own-win/settlement.ko.svg" width="720" height="360" loading="lazy" alt="전: 정산 마커는 실시간 DB, 트로피는 문서 DB에 있어 두 쓰기 사이에서 함수가 죽으면 영영 미지급이나 중복 지급이 생긴다. 후: 문서 DB의 트랜잭션 하나 안에서 원장을 먼저 읽고, 없을 때만 트로피와 원장을 한 번에 커밋한다."></a></div><figcaption>멱등성 마커가 보상과 다른 저장소에 있으면 실패가 두 갈래로 열린다. 같은 트랜잭션에 넣으면 재시도는 조용히 아무 일도 하지 않는다.</figcaption></figure>

위조 방어도 같은 트랜잭션 안에 넣었다. 두 참가자 모두 그 방에 들어온 흔적이 남아 있고, 각 흔적의 신원이 배정된 역할과 일치할 때만 정산이 유효하다. 방에 들어온 적도 없는 사람을 패자로 지목하는 요청은 통과하지 못한다.

## 관통하는 것

세 문제를 한 표로 놓으면 이렇다.

| 영역 | 취약 지점 | 해결 |
|------|----------|------|
| 승패 판정 | 클라이언트가 "이겼다"를 보낼 수 있음 | 자기 패배만 신고 + 공유 결과 자리에 선착순 확정 |
| 자동 매칭 | 동시 발견 시 방 2개 생성(경쟁 상태) | 작은 ID가 방을 만드는 결정적 규칙으로 경합 제거 |
| 트로피 정산 | 멱등성 마커와 보상이 다른 저장소에 분리 | 원장을 보상과 같은 DB로 옮겨 단일 트랜잭션 커밋 |

세 가지 다 결국 같은 자리에서 나왔다. 클라이언트가 하는 말은 단서일 뿐 증거가 아니라는 것.

전에 내 에이전트 도구를 만들면서 "주장을 믿지 말고 상태를 검증하라"를 원칙으로 삼았는데, 대전을 붙이면서 그 문장을 다시 만났다. 상대가 뭐라고 말하든, 끝나고 남는 건 서버에 원자적으로 기록된 것뿐이다.

작은 스튜디오에는 서버 팀도 없고 매치메이커도 없다. 대신 가진 게 하나 있다. 어떤 값이 위조되면 곤란한지 처음부터 알고 있다는 것. 그 자리마다 원자적 기록을 하나씩 놓는 걸로 꽤 멀리 갈 수 있었다.

---

## English version

I added 1v1 matches to my own puzzle game. The first opponent was a bot, and that part was easy. Everything about the opponent ran on my own device, so my code always knew who had won.

<figure class="shot"><img src="/assets/img/posts/you-cant-report-your-own-win/moongchi-play.jpg" width="440" height="751" loading="lazy" alt="Moongchi gameplay — animal pieces merging inside a wooden-framed board, with BOOM combos going off"><figcaption>The game in this post, Moongchi. This is what got a human-vs-human mode.</figcaption></figure>

That assumption collapsed the moment two humans faced each other. Now the opponent runs on someone else's phone, and whatever that phone says, I have no way to check it. Most of the last three weeks went into absorbing that one sentence.

### You don't get to announce your own win

The first rule I settled on was simple: you can never report your own victory, only your own defeat.

The reasoning is obvious enough. If a client can send "I won", that message *is* the rank-manipulation button. Nobody has a reason to forge "I lost". So each side reports only its own loss, and the win is derived from the other end.

That alone wasn't enough, though, because a match can end through more than one path: a board fills up, the clock runs out, someone walks away. Each path has the two devices reaching a conclusion at slightly different moments. If each one renders its own conclusion, you get two players staring at contradictory results for the same match.

So I stopped letting them decide independently. However a match ends, finalizing has to pass through one shared result slot. **Whichever record lands there first is the outcome, and the late arrival discards its own verdict and displays what it reads.** The two screens can no longer disagree.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/you-cant-report-your-own-win/result-slot.en.svg"><img src="/assets/img/posts/you-cant-report-your-own-win/result-slot.en.svg" width="720" height="330" loading="lazy" alt="Device A and device B each send their conclusion to a shared result slot. The first record to land becomes the outcome; the late arrival drops its own verdict. Both screens read the same record: A shows a loss, B shows a win."></a></div><figcaption>However a match ends, the outcome is the first record in the shared slot. A win is never reported — it is derived from the other side’s reported loss.</figcaption></figure>

The rest was per-path handling. When both boards die at effectively the same instant, the tie is re-judged by comparing death timestamps. Timeouts wait for the opponent's report, but with a hard ceiling on that wait, since a backgrounded app can drift badly. If someone left, a brief wait ran and the match went to whoever stayed. That last piece fell apart badly once I tested on two real devices and had to be redesigned from scratch, which is a story for the next post.

### Who creates the room?

Automatic matchmaking came next. Private matches with a room code already worked, but ranked play needed strangers to pair up on a single tap.

The problem: I have no matchmaker server. Park a queue in the realtime DB and let clients pair themselves, and sooner or later two players spot each other in the same instant. If both create a room, you get two rooms and two people waiting alone.

Instead of adding an arbiter, I made the outcome impossible to disagree on. **Of the two IDs, the smaller one creates the room.** It's a value that resolves identically for a given pair, so it doesn't matter who noticed whom first. The smaller ID creates the room and writes the room number into its own queue entry; the larger one reads it, joins, and leaves a join marker. There's simply nothing left to race over.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/you-cant-report-your-own-win/matching.en.svg"><img src="/assets/img/posts/you-cant-report-your-own-win/matching.en.svg" width="720" height="400" loading="lazy" alt="Sequence of player A (smaller ID), the queue, and player B (larger ID). After spotting each other at once, only A creates the room and writes its number to its own entry; B reads it, joins, and leaves a join marker."></a></div><figcaption>The rule yields one answer per pair, so only one room is ever created, no matter who noticed whom first.</figcaption></figure>

I locked the permissions to match: in the queue, you can only write your own entry. Anything looser opens a path to hijacking or wrecking someone else's match.

My first design actually did pass state by planting a flag on the *opponent's* entry — which ran straight into that permission rule. Review rejected it, and that's when it was replaced wholesale with the handshake above. Nailing down the permission first is what made the design error visible.

One more wrinkle: waiting players are bucketed by rating, and two people sitting on opposite sides of a bucket boundary never see each other, even when their ratings are nearly identical. Widening the query to cover adjacent buckets closed that gap.

### When the marker and the reward live apart

Trophy settlement took the longest.

When a ranked match ends, the server adjusts both players' trophies. Settle the same match twice and the whole ladder inflates, so I wanted a marker saying "this match is settled" and a no-op whenever it already exists.

My first version put the marker in the realtime DB and the trophy change in the document DB. Review rejected that twice. It was frustrating at the time, and it was also right.

Two different stores can't be wrapped in a single transaction. If the function dies between the two writes, one of two things happens. Marker written, trophies not applied: every retry sees "already handled" and the match never pays out. Trophies applied, marker missing: every retry pays out again. The mechanism I added *for* idempotency was, by virtue of where it sat, holding both double-payment and permanent non-payment open at once.

The fix was to move the marker next to the effect. The settlement ledger now lives in the same DB as the trophies, and a single transaction reads the ledger first. If the entry exists, it does nothing. If it doesn't, both trophy deltas and the ledger entry commit together. A retry finds the ledger and quietly does nothing at all.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/you-cant-report-your-own-win/settlement.en.svg"><img src="/assets/img/posts/you-cant-report-your-own-win/settlement.en.svg" width="720" height="360" loading="lazy" alt="Before: the settled marker lives in the realtime DB and trophies in the document DB, so a crash between the writes causes permanent non-payment or double payment. After: one document-DB transaction reads the ledger first and, only if absent, commits trophies and ledger together."></a></div><figcaption>An idempotency marker stored apart from the reward opens two failure paths. Put them in one transaction and a retry quietly does nothing.</figcaption></figure>

Forgery defense went into the same transaction. Settlement is only valid if both participants left a join marker in that room and each marker's identity matches the role it was assigned. A request naming someone who never entered the room doesn't get through.

### The thread running through it

The three problems, side by side:

| Area | Vulnerable point | Fix |
|------|------------------|-----|
| Verdicts | A client could send "I won" | Report only your own loss + first write to a shared result slot wins |
| Matchmaking | Simultaneous discovery creates two rooms (race condition) | Deterministic rule — the smaller ID creates the room — removes the race |
| Trophy settlement | Idempotency marker and reward split across two stores | Ledger moved next to the reward, committed in one transaction |

All three came from the same place: what a client says is a lead, not evidence.

When I built my agent tooling earlier, I landed on "don't trust claims, verify state" as a principle. Wiring up multiplayer, I ran into that sentence again. Whatever the opponent's device says, what survives is only what was atomically recorded on the server.

A small studio has no backend team and no matchmaker. But it comes with one advantage: from day one you know exactly which values would hurt if forged. Putting a single atomic record at each of those spots got me surprisingly far.
