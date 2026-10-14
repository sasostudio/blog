---
layout: post
title: "클라이언트를 믿지 않는 대전 설계"
description: "퍼즐 게임에 사람 대 사람 대전을 붙이며 세운 원칙: 클라이언트의 말은 단서일 뿐, 증거는 서버 기록. 자기 패배만 신고하는 판정, ID로 방장을 정하는 매칭, 트로피와 같은 트랜잭션 안의 정산 원장으로 해결."
date: 2026-07-27 10:00:00 +0900
project: moongchi
tags: [game-dev, multiplayer, firebase, build-in-public]
---

*🇬🇧 [English version below](#english-version)*

직접 만드는 퍼즐 게임에 1대1 대전 추가. 첫 상대는 봇. 봇전의 난도는 그리 높지 않았던 편. 상대의 모든 것이 내 기기 안에서 돌기 때문에, 승패도 결국 내 코드가 아는 구조.

<figure class="shot"><img src="/assets/img/posts/you-cant-report-your-own-win/moongchi-play.jpg" width="440" height="751" loading="lazy" alt="뭉치뭉치 플레이 화면 — 나무 틀 보드 안에서 동물 캐릭터들이 합쳐지며 BOOM 콤보가 터지는 장면"><figcaption>이 글의 게임, 뭉치뭉치. 여기에 사람 대 사람 대전을 추가.</figcaption></figure>

사람 대 사람으로 넘어가는 순간 그 전제는 통째로 붕괴. 이제 상대는 남의 폰에서 실행되고, 그 폰이 무슨 말을 하든 확인할 방법이 부재. 지난 3주의 대부분은 이 한 문장을 감당하는 작업.

## 승자의 자기 승리 신고 금지

가장 먼저 정한 규칙은 단순. 자기 승리 신고는 불가, 자기 패배 신고만 허용.

이유는 명확. 「내가 이겼다」를 클라이언트가 보낼 수 있다면 그 메시지 하나가 곧 랭킹 조작 버튼. 반대로 「내가 졌다」는 위조할 이유 자체가 부재. 그래서 각자는 자기 패배만 신고하고, 승리는 그 반대편에서 파생되는 구조.

그런데 이것만으로는 불충분. 대전이 끝나는 경로가 하나가 아니기 때문. 한쪽 보드가 꽉 차서 끝날 수도, 제한 시간이 다 될 수도, 한쪽이 나가버릴 수도 있는 상황. 경로마다 두 기기가 서로 조금씩 다른 시점에 결론에 도달. 각자 자기 결론을 자기 화면에 그리면, 같은 경기를 두고 두 사람이 다른 결과를 보는 사태 발생.

그래서 결과의 개별 판단을 금지. 어느 경로로 끝나든 확정은 두 기기가 공유하는 결과 자리를 반드시 경유. **먼저 그 자리에 남은 기록이 그 경기의 결론이고, 늦게 도착한 쪽은 자기 판단을 버리고 그 결론을 읽어서 표시.** 양쪽 화면의 불일치가 구조적으로 불가능한 설계.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/you-cant-report-your-own-win/result-slot.ko.svg"><img src="/assets/img/posts/you-cant-report-your-own-win/result-slot.ko.svg" width="720" height="330" loading="lazy" alt="기기 A와 기기 B가 각자 결론을 공유 결과 자리로 전송. 먼저 도착한 기록 1건이 결론, 늦게 도착한 쪽은 자기 판단 폐기. 두 화면은 같은 기록을 읽어 A는 패배, B는 승리를 표시."></a></div><figcaption>어느 경로로 끝나든 결론은 공유 결과 자리에 먼저 남은 1건. 승리는 신고가 아니라 상대의 패배 신고에서 파생.</figcaption></figure>

나머지는 경로별 예외 처리. 두 보드가 사실상 동시에 끝나면 사망 시각을 비교해 재판정, 시간 종료는 상대 보고를 기다리되 앱이 백그라운드로 내려가 시계가 밀리는 경우에 대비해 대기에 절대 상한 설정. 한쪽이 나가면 잠깐 대기 후 남은 사람의 승리로 처리. 이 이탈 처리는 나중에 실기기 두 대 검증에서 크게 깨져 전면 재설계 대상이 되는데, 그 이야기는 다음 글에서 별도 정리.

## 방 생성 주체의 결정

다음 과제는 자동 매칭. 방 코드를 주고받는 사설 대전은 이미 동작 중이었지만, 아무나 눌러도 상대가 붙는 랭크 매치가 필요.

문제는 매치메이커 서버의 부재. 대기열을 실시간 DB에 두고 클라이언트끼리 붙게 하면, 둘이 동시에 서로를 발견하는 순간이 발생. 둘 다 방을 만들면 방이 두 개 생기고, 각자 빈 방에서 상대를 기다리는 상황. 전형적인 경쟁 상태(race condition).

해결 방향은 중재자 추가가 아니라 결론이 하나로만 나오는 규칙. **두 사람의 ID 중 작은 쪽이 방을 생성.** 쌍마다 유일하게 정해지는 값이라 누가 먼저 상대를 발견했든 결론은 동일. 작은 쪽이 방을 만들고 자기 대기열 항목에 방 번호를 기록하면, 큰 쪽이 그 번호를 읽고 입장하며 입장 흔적을 기록. 경합의 여지 자체가 소멸.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/you-cant-report-your-own-win/matching.ko.svg"><img src="/assets/img/posts/you-cant-report-your-own-win/matching.ko.svg" width="720" height="400" loading="lazy" alt="플레이어 A(ID 작음), 대기열, 플레이어 B(ID 큼)의 순서도. 거의 동시에 서로를 발견한 뒤 A만 방을 만들어 자기 항목에 방 번호를 기록, B는 그 번호를 읽어 입장하며 입장 흔적을 기록."></a></div><figcaption>규칙이 쌍마다 하나의 답을 내므로, 누가 먼저 상대를 발견했든 방은 하나만 생성.</figcaption></figure>

권한도 함께 잠금. 대기열에서는 자기 항목만 쓰기 가능. 남의 항목을 건드릴 수 있다면 남의 매칭을 가로채거나 망가뜨릴 경로가 열리기 때문.

사실 첫 설계는 상대 항목에 플래그를 심어 넘기는 방식이었는데, 방금 그 권한 규칙과 정면 충돌. 검수에서 반려됐고, 그때 지금의 방식으로 전면 교체. 권한을 먼저 확정해 둔 덕분에 설계 오류가 스스로 드러난 셈.

한 가지 더. 대기자는 실력 구간으로 나눠 조회하는데, 구간 경계를 사이에 두고 선 두 사람은 실력이 거의 같아도 서로를 영영 발견 불가. 자기 구간뿐 아니라 인접 구간까지 함께 조회하도록 범위를 넓혀 해결.

## 마커와 보상의 저장소 분리 문제

가장 오래 붙잡은 과제는 트로피 정산.

랭크 경기가 끝나면 서버가 양쪽 트로피를 조정. 여기서 같은 경기가 두 번 정산되면 랭킹 전체가 부풀어 오르는 문제. 그래서 「이 경기는 정산됨」 마커를 남기고, 마커가 이미 있으면 아무것도 하지 않는 방식을 계획.

처음 짠 구조는 마커는 실시간 DB, 트로피 변경은 문서 DB. 검수에서 두 번 반려. 당시엔 답답했지만 지적은 정확.

서로 다른 저장소라 하나의 트랜잭션으로 묶기 불가. 두 쓰기 사이에서 함수가 죽으면 둘 중 하나가 발생. 마커만 남고 트로피가 미반영이면, 그 경기는 재시도해도 「이미 처리됨」으로 걸러져 영구 미지급. 트로피만 반영되고 마커가 없으면, 재시도할 때마다 중복 지급. 멱등성을 지키려고 넣은 장치가, 놓인 위치 때문에 미지급과 중복 지급을 동시에 열어 둔 상황.

수정 방향은 마커를 효과와 같은 곳으로 이동. 정산 원장을 트로피와 같은 DB에 두고, 하나의 트랜잭션 안에서 원장을 먼저 조회. 이미 있으면 아무것도 하지 않고 종료. 없으면 양쪽 트로피 변경과 원장 기록을 한 번에 커밋. 재시도해도 원장이 이미 있으니 아무 일도 발생하지 않는 구조.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/you-cant-report-your-own-win/settlement.ko.svg"><img src="/assets/img/posts/you-cant-report-your-own-win/settlement.ko.svg" width="720" height="360" loading="lazy" alt="전: 정산 마커는 실시간 DB, 트로피는 문서 DB에 있어 두 쓰기 사이에서 함수가 죽으면 영구 미지급이나 중복 지급 발생. 후: 문서 DB의 트랜잭션 하나 안에서 원장을 먼저 조회, 없을 때만 트로피와 원장을 한 번에 커밋."></a></div><figcaption>멱등성 마커가 보상과 다른 저장소에 있으면 실패 경로가 두 갈래. 같은 트랜잭션에 넣으면 재시도는 조용히 무효 처리.</figcaption></figure>

위조 방어도 같은 트랜잭션에 포함. 두 참가자 모두 그 방에 입장한 흔적이 있고, 각 흔적의 신원이 배정된 역할과 일치할 때만 정산이 유효. 방에 입장한 적도 없는 사람을 패자로 지목하는 요청은 통과 불가.

## 관통하는 원칙

세 문제를 한 표로 정리하면 아래와 같은 구성.

| 영역 | 취약 지점 | 해결 |
|------|----------|------|
| 승패 판정 | 클라이언트의 「이겼다」 전송 가능 | 자기 패배만 신고 + 공유 결과 자리에 선착순 확정 |
| 자동 매칭 | 동시 발견 시 방 2개 생성(경쟁 상태) | 작은 ID가 방을 만드는 결정적 규칙으로 경합 제거 |
| 트로피 정산 | 멱등성 마커와 보상의 저장소 분리 | 원장을 보상과 같은 DB로 이동해 단일 트랜잭션 커밋 |

세 가지 모두 같은 지점에서 출발. 클라이언트의 말은 단서일 뿐 증거가 아니라는 사실.

예전에 에이전트 도구를 만들면서 「주장을 믿지 말고 상태를 검증하라」를 원칙으로 삼았는데, 대전을 붙이며 그 문장을 다시 만난 셈. 상대가 무슨 말을 하든, 끝나고 남는 건 서버에 원자적으로 기록된 결과뿐.

작은 스튜디오에는 서버 팀도 매치메이커도 부재. 대신 가진 강점 하나는 어떤 값이 위조되면 곤란한지를 처음부터 안다는 점. 그 자리마다 원자적 기록을 하나씩 두는 것만으로도 상당한 거리까지 도달 가능.

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
