---
layout: post
title: "수분 코치 - 거짓 통과 신호 10가지"
description: "수분 코치를 만든 한 달 동안 「통과했다」는 신호가 실제로는 아무것도 보지 않았던 10가지 사례. 실패에 붙은 HTTP 200, 코드를 바꿔도 통과하는 테스트, 214건이 통과하는 가운데 살아남은 버그 등."
date: 2026-08-27
project: hydro-coach
tags: [build-in-public, engineering, testing, observability, indie-dev]
---

*🇬🇧 [English version below](#english-version)*

[앞 글](/2026/08/27/first-launch-d1-retention-zero/)에서 접은 앱 이야기를 했다. 이 글은 그 앱을 만든 한 달 동안 잡은 다른 것이다.

한 달 내내 반복된 일이 하나 있다. **뭔가가 "통과했다"고 말하는데, 그 신호가 실제로 무엇을 봤는지는 확인이 안 된 상태.** 열 번 겪었고, 그중 몇 개는 의심하지 않았으면 라이브에서 터졌을 것이다.

이 프로젝트는 코딩 에이전트와 짝을 이뤄 만들었다. 그래서 아래 목록에는 두 종류가 섞여 있다. 에이전트가 통과라고 보고한 걸 내가 되물어 잡은 것, 그리고 둘 다 통과라고 믿고 있다가 나중에 깨진 것. 앱은 접었지만 이 목록은 다음 프로젝트로 그대로 넘어간다.

## 1. 실패에 200을 붙여 보내는 서버

외부 발송 API에 첫 호출을 던졌더니 HTTP 200이 왔다. 본문을 열어 보니 발송 건수가 0이고, 사유가 "이 사용자가 수신 동의를 안 했음"이었다.

내 구현 계획서는 상태 코드로 성패를 가르게 돼 있었다. 그대로 만들었으면 **발송이 실패했는데 성공으로 기록되고 재시도도 안 되는 조용한 버그**가 났을 것이다. 실사용자 폰에서만 드러나는 종류다.

판정 순서를 문서에 못 박았다. 상태 코드는 "연결이 됐다"까지만 말한다. 성패는 본문이 말한다.

## 2. 한 명에게 한 번 보냈는데 카운트가 2

동의를 받고 실제로 발송이 뚫린 뒤, 성공 응답에서 함정을 하나 더 건졌다. 한 사람에게 한 건 보냈는데 **처리 건수가 2로 찍혔다.** 푸시와 앱 안 알림함에 동시에 쌓이기 때문이었다.

"1이면 성공"으로 짰으면 정상 발송을 전부 실패로 기록하고 무한 재시도를 돌 뻔했다. 남의 API가 세는 단위가 내가 세는 단위와 같다고 가정하면 안 된다.

## 3. 대조군 없는 관측은 결론이 아니다

두 번 겪었다.

**첫 번째.** 푸시 서버를 배포하고 인증 통로를 점검했더니 "인증 정보를 찾을 수 없다"는 응답이 왔다. 에이전트는 이걸 통과로 보고했다. 내가 "그게 통과가 맞냐"고 되물었다. 서버가 인증서를 검사한 끝에 준 답인지, 인증서를 아예 안 보고 준 답인지 그 응답만으로는 구분이 안 되기 때문이다. 같은 요청을 **인증서 있이 한 번, 없이 한 번** 보내는 대조 검사를 만들어 돌리니 인증서 없는 쪽만 아예 거절됐고, 그제야 통과가 입증됐다.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/false-pass-signals/control-test.ko.svg"><img src="/assets/img/posts/false-pass-signals/control-test.ko.svg" width="720" height="270" loading="lazy" alt="같은 요청을 인증서와 함께 보내면 앱 단계까지 들어가 인증 정보 없음 응답을 받고, 인증서 없이 보내면 연결 단계에서 아예 거절된다. 결과가 갈렸으므로 서버가 인증서를 실제로 검사한다는 것이 입증된다."></a></div><figcaption>응답 하나로는 구분이 안 되던 두 경우를, 대조군 하나가 갈라 놓았다.</figcaption></figure>

**두 번째.** 실행 환경의 외부 호출 한도를 재는데 기대치의 4배를 넘겨도 전부 통과했다. "한도가 크다"고 적고 싶었지만 **무효로 기록했다.** 내가 세는 대상이 실제로 제한받는 대상과 다를 수 있는데 비교 대조군이 없어서, "한도가 크다"와 "이 종류는 애초에 안 세진다"를 구분할 수 없었기 때문이다.

나중에 측정 방식을 갈아엎어 다시 쟀다. 서버가 자기 페이지를 반복 호출하게 만들고, 같은 실행에서 DB 조회를 51번 더 섞었다. 실패 지점이 1도 안 밀렸다. 그걸로 DB 조회는 애초에 안 세진다는 게 확정됐다. **대조군 하나가 추측 몇 개를 지웠다.**

## 4. 코드를 바꿔도 통과하는 테스트

게시 문서와 코드의 수치가 어긋나지 않도록 테스트로 묶었다고 썼다. 사실이 아니었다. 기대값을 글자 그대로 박아놔서 **코드를 바꿔도 그대로 통과하는 테스트**였다.

고친 뒤에는 실제 설정값을 읽어 기대값을 만들도록 바꾸고, 값을 일부러 세 번 틀리게 바꿔 각각 잡히는지 확인한 다음 되돌렸다.

비슷한 게 하나 더 있었다. 발송 인원 상한을 올렸더니 기존 경계 테스트가 **조용히 무의미해졌다.** 숫자가 하드코딩돼 있어서, 상한이 올라가자 테스트 대상이 전부 한 배치에 들어가 초과 상황 자체가 만들어지지 않았다. 상한 상수에서 계산되도록 고쳤다.

테스트가 통과하는 것과 테스트가 무언가를 지키고 있는 건 다르다. 확인하는 방법은 하나뿐이다. **일부러 깨 보고 빨간불이 켜지는지 본다.**

## 5. 테스트 214건이 통과하는데 살아남은 버그

D1 카운터가 날짜가 바뀌어도 초기화되지 않는 버그가 있었다. 갱신 전 값을 기준으로 계산하는 특성 때문이었는데, 며칠 뒤부터 알림이 하루 1건으로 막히는 결함이다.

**테스트 214건이 전부 통과하는 상태였다.** 이유는 단순하다. 그 지점을 강제하는 단언이 하나도 없었다. 커버리지 숫자가 높은 것과 중요한 경로가 지켜지는 건 무관하다.

수정할 때는 결함을 일부러 되살려 테스트가 실제로 잡는지 확인한 뒤 되돌렸다.

## 6. 인증서를 못 읽은 건 인증서 탓이 아니었다

서버 간 통신에 mTLS를 쓰는데 첫 연결이 계속 실패했다. "인증서가 잘못 발급된 건가" 싶었다. 여기서 재발급을 눌렀으면 멀쩡한 자산을 버리고 발급 절차를 처음부터 다시 밟았을 것이다.

원인은 인증서가 아니라 **명령을 실행한 도구**였다. Windows에 깔린 curl 두 종류가 전부 Schannel 빌드였고, Schannel은 PEM 형식 클라이언트 인증서를 읽지 못한다. 파일은 정상적으로 열었고, 그다음 PKCS#12로 해석하려다 실패한 것이다.

결정적 단서는 오류 문구였다. "파일을 못 열었다"가 아니라 **"읽어서 해석하는 데 실패했다"** 였다. 이 한 줄이 경로 문제와 권한 문제를 동시에 배제해 줬다. 도구를 Node/OpenSSL로 바꿔 다시 던지니 한 번에 통과했다.

여기서 살린 건 코드가 아니라 방침이다. **"원인을 모를 때 재발급하지 않는다."** 이 원칙이 실제로 멀쩡한 걸 버리는 사고를 막았다.

## 7. 주석은 검증이 아니다

실기기 점검에서 결함 하나가 잡혔다. 알림을 끄려다 실패했는데 **화면 스위치만 꺼졌다.** 기기에도 서버에도 켜짐 기록이 남아 있으니, 사용자는 껐다고 믿는데 알림은 계속 갈 수 있는 상태였다.

원인은 켜기와 끄기 두 흐름을 한 줄로 판정한 코드였다. 그런데 그 코드 바로 위 주석에는 **"실패하면 되돌린다"**고 적혀 있었다. 의도는 정확히 적혀 있었고 구현이 그걸 안 지켰다.

주석은 의도의 기록이지 동작의 증거가 아니다. 헷갈리기 쉬운 이유는 주석이 코드보다 읽기 편하고 대체로 더 그럴듯하기 때문이다.

## 8. 에러 없이 조용히 죽는 캐시

운영비를 계산하다가 예상 밖의 걸 발견했다. **가장 먼저 무너지는 한도가 사용자 수와 무관했다.**

날씨 캐시를 지역별로 저장하는 구조라, KV 쓰기 한도가 사용자 수가 아니라 **서로 다른 지역 개수**에 걸린다. 대략 20개 지역이면 상한이었다.

더 나쁜 건 초과했을 때의 동작이다. 캐시 저장 실패는 일부러 조용히 넘기게 짜뒀다. 캐시가 안 되는 건 기능 장애가 아니니 크래시를 내면 안 된다고 판단했던 것이다. 그래서 이런 경로가 생긴다.

| 단계 | 일어나는 일 | 어디서 보이나 |
|:--:|---|---|
| 1 | 고유 지역이 20개를 넘는다 | 안 보임 |
| 2 | 캐시 쓰기가 실패하기 시작 (에러 없음) | 안 보임 |
| 3 | 캐시가 사실상 죽고 외부 API 호출이 사용자 수에 비례해 폭증 | 클라우드 대시보드만 |
| 4 | 기상청 쿼터 초과 → 날씨 기능 전면 정지 | 사용자가 먼저 안다 |

우리 로그에는 3번까지 아무것도 안 잡힌다. **조용한 열화는 조용한 만큼 늦게 발견된다.** 해소 자체는 월 몇 달러짜리 유료 전환 하나로 병목 네 개가 동시에 풀리는 구조라, 인프라 비용은 사실상 문제가 아니었다. 문제는 그 시점을 관측할 수 있느냐였다.

## 9. 로그가 보존됐다는 증거는 PC가 꺼져 있었다는 사실이었다

"사용자가 늘어 알림이 상한에 밀렸을 때 그걸 알 방법이 있나"를 규명했다. 결론은 외부 콘솔로는 원리적으로 불가능이었다. 상한에 걸려 밀린 사람에겐 발송 요청 자체를 안 보내니 외부 통계에 흔적이 0이다. 외부는 "보낸 것"만 안다.

밀린 인원을 세는 지표는 이미 서버에 있었다. 없던 건 판단 로직이 아니라 **로그 보존**이었다. 실시간 스트림으로만 볼 수 있어서, 아침 특정 시각 기록을 보려면 매일 그 시간에 터미널을 켜고 있어야 했다.

보존을 켜고 검증했는데 통과 근거가 재미있었다. 한 시간 전 기록이 조회됐다는 것만으로는 부족하다. 실시간으로 보고 있었을 가능성이 남기 때문이다. 결정타는 **그 시각에 PC가 꺼져 있었다는 사실**이었다. 관측자가 존재할 수 없었으므로 그건 보존된 기록이다.

두 신호가 서로 다른 걸 답한다는 것도 문서에 못 박았다.

| 신호 | 아는 것 | 모르는 것 |
|---|---|---|
| 내부 지표 | 안 보낸 것 (상한에 밀림) | 보낸 게 도착했는지 |
| 외부 통계 | 보냈는데 안 닿은 것 | 애초에 안 보낸 건 |

## 10. 범위를 안 밝힌 전수 주장

검수 단계에서 보고 하나가 반증된 적이 있다. "전수 확인했고 0건"이라고 단언했는데, 규범 문서만 훑고 이력 문서를 빼먹은 상태였다.

결론 자체는 영향이 없었다. 문제는 그게 **검증 없는 단언**이었다는 것이다. 이후로는 확인한 파일 범위를 함께 적기로 했다.

**범위를 안 밝힌 전수 주장은 애초에 검증할 수 없는 주장이다.** 반증하려는 사람이 어디를 봐야 하는지 알 수 없기 때문이다.

## 하나로 줄이면

열 개가 전부 같은 모양이다. 어떤 신호가 통과라고 말하는데, 그 신호가 **실제로 무엇을 봤는지**는 확인되지 않았다.

- 200은 연결을 봤지 발송을 안 봤다
- 처리 건수 2는 두 채널을 봤지 사람 수를 안 봤다
- 초록불 214개는 코드를 봤지 자정 넘김을 안 봤다
- 조회된 로그는 기록을 봤지 그게 보존된 건지는 안 봤다

그래서 물어야 할 질문은 "통과했나"가 아니다. **"이 신호가 실패했다면 어떻게 달라 보였을까"** 다. 답이 "똑같아 보였을 것"이면 그건 통과가 아니라 아무것도 아니다.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/false-pass-signals/signal-blind-spots.ko.svg"><img src="/assets/img/posts/false-pass-signals/signal-blind-spots.ko.svg" width="720" height="268" loading="lazy" alt="네 가지 통과 신호가 본 것과 안 본 것. HTTP 200은 연결을 봤고 발송은 안 봤다. 처리 건수 2는 두 채널을 봤고 사람 수는 안 봤다. 테스트 214개 통과는 코드를 봤고 자정 넘김은 안 봤다. 조회된 로그는 기록을 봤고 보존 여부는 안 봤다."></a></div><figcaption>통과 신호마다 실제로 본 것과 보지 못한 것이 있다.</figcaption></figure>

앱은 접었다. 이 목록은 안 접었다.

---

## English version

[The previous post](/2026/08/27/first-launch-d1-retention-zero/) was about the app I shut down. This one is about something else I caught during the month I spent building it.

One thing kept recurring: **something reports "pass," but nobody has checked what that signal actually looked at.** It happened ten times, and a few of them would have blown up in production if I hadn't pushed on them.

I built this project paired with a coding agent, so the list below mixes two kinds of case: ones where the agent reported a pass and I pushed back, and ones where both of us believed the pass until it broke later. The app is dead; the list carries forward.

### 1. A server that stamps 200 on failure

First call to an external send API came back HTTP 200. Opened the body: zero sends, reason "this user has not consented to receive notifications."

My implementation plan decided success from the status code. Built as written, that produces **a silent bug where a failed send is recorded as success and never retried** — the kind that only surfaces on a real user's phone.

I pinned the order of judgment into the doc. The status code tells you the connection happened. The body tells you whether anything worked.

### 2. One send to one person, counted as two

After consent came through and sends actually started landing, the success response handed me another trap. One notification to one person came back with **a processed count of 2** — because it lands in the push tray and the in-app inbox at the same time.

Had I coded "1 means success," every healthy send would have been logged as a failure and retried forever. Never assume somebody else's API counts the same unit you do.

### 3. An observation without a control is not a conclusion

Twice.

**First.** I deployed the push server and probed the auth path. The response said "credentials not found." The agent reported that as a pass. I pushed back — that response alone can't distinguish a server that inspected my certificate from one that never looked at it. So we built a paired check, the same request **once with the client certificate, once without.** Only the certificate-less call was rejected outright. That's when the pass was actually established.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/false-pass-signals/control-test.en.svg"><img src="/assets/img/posts/false-pass-signals/control-test.en.svg" width="720" height="270" loading="lazy" alt="The same request with the certificate reaches the app layer and gets “credentials not found”; without the certificate it is rejected outright at the connection. Because the results split, the server demonstrably checks the certificate."></a></div><figcaption>One control separated two cases a single response could not.</figcaption></figure>

**Second.** Measuring the runtime's outbound-call limit, I blew past 4× the expected ceiling with everything still succeeding. I wanted to write down "the limit is generous." I **recorded the measurement as void instead.** What I was counting might not be what's actually rate-limited, and with no control group I couldn't distinguish "the limit is high" from "this kind of call isn't counted at all."

Later I rebuilt the measurement: had the server hammer its own endpoint, and mixed in 51 extra DB queries in the same run. The failure point didn't shift by one. That settled it — DB queries aren't counted. **One control erased several guesses.**

### 4. A test that passes no matter what the code does

I wrote that the published document and the code were now bound together by a test, so they couldn't drift. Not true. The expected value was hardcoded, so **the test passed regardless of what the code said.**

I fixed it to derive expectations from the actual config, then deliberately broke the value three different ways to confirm each one got caught, then reverted.

There was a sibling. When I raised the per-batch send cap, an existing boundary test **quietly became meaningless** — the count was hardcoded, so once the cap went up every test subject fit in one batch and the overflow case stopped existing. I made it compute from the cap constant.

A passing test and a test that's guarding something are different things. There's exactly one way to tell: **break it on purpose and watch for red.**

### 5. A bug that survived 214 passing tests

A D1 counter failed to reset across midnight, because of how the database evaluates against the pre-update value. Symptom: a few days in, notifications get capped at one per day.

**All 214 tests were green.** Simple reason — not one assertion forced that path. A high coverage number and your important paths being protected are unrelated facts.

Fixing it, I reintroduced the defect on purpose to confirm the new test actually caught it, then reverted.

### 6. The certificate wasn't the problem

Server-to-server calls use mTLS, and the first connection kept failing. My instinct was that the certificate had been issued wrong. Had I hit reissue there, I'd have thrown away a perfectly good asset and restarted the whole provisioning process.

The cause was the **tool running the command.** Both curl builds installed on Windows were Schannel builds, and Schannel can't read PEM client certificates. It opened the file just fine, then failed trying to parse it as PKCS#12.

The decisive clue was the wording of the error. Not "couldn't open the file" but **"failed to read and parse it."** That single line ruled out path problems and permission problems simultaneously. Switching to Node/OpenSSL, the same request passed on the first try.

What saved me wasn't code, it was a policy: **don't reissue while the cause is unknown.** That rule prevented a real loss.

### 7. A comment is not verification

An on-device check surfaced a defect: turning notifications *off* failed, but **the UI switch flipped anyway.** The enabled record persisted on both device and server, so the user believes they're off while notifications can still arrive.

The cause was a single line judging both the enable and disable flows the same way. And the comment directly above that line read **"revert on failure."** The intent was stated precisely. The implementation ignored it.

A comment records intent, not behavior. It's easy to be fooled because comments are more readable than code, and usually more convincing.

### 8. A cache that dies silently

Working out the running costs, I found something I didn't expect: **the first limit to break had nothing to do with user count.**

Weather responses cache per region, so the KV write limit binds on **the number of distinct regions**, not users. Roughly 20 regions was the ceiling.

Worse is what happens past it. I'd deliberately made cache-write failures swallow silently — a cache miss isn't a functional outage, so it shouldn't crash anything. Which creates this path:

| Step | What happens | Where it's visible |
|:--:|---|---|
| 1 | Distinct regions exceed ~20 | nowhere |
| 2 | Cache writes start failing (no error) | nowhere |
| 3 | Cache effectively dies; upstream API calls scale with users | cloud dashboard only |
| 4 | KMA quota exceeded → weather features stop entirely | users notice first |

Nothing in my own logs registers through step 3. **Silent degradation is discovered exactly as late as it is silent.** The fix was easy — a few dollars a month unlocks four bottlenecks at once, so infrastructure cost was never the real issue. The issue was whether I could see the moment arrive.

### 9. The proof the logs persisted was that the PC was off

I set out to answer: if user growth pushes notifications past the send cap, is there any way to know? Externally, no — structurally impossible. Anyone dropped by the cap never gets a send request in the first place, so they leave zero trace in external stats. Outside systems only know what was sent.

The metric counting dropped recipients already existed server-side. What was missing wasn't logic, it was **log retention.** Records were visible only as a live stream, so seeing a specific morning's run meant sitting at a terminal at that hour every day.

I enabled retention and verified it, and the grounds for the pass are my favorite part. Retrieving an hour-old record isn't sufficient on its own, because I might have been streaming it live. The clincher was **that the machine was powered off at that time.** No observer could have existed, therefore the record was retained.

I also pinned down that the two signals answer different questions:

| Signal | Knows | Doesn't know |
|---|---|---|
| Internal metric | what was never sent (dropped by cap) | whether sends arrived |
| External stats | what was sent but didn't land | what was never sent |

### 10. An exhaustiveness claim with no stated scope

A review pass refuted one of the reports. It asserted "checked exhaustively, zero occurrences" on the basis of the normative docs alone, having skipped the history docs.

The conclusion happened to be unaffected. The problem was that it was **an assertion without verification.** Since then I state the file scope I actually covered.

**An exhaustiveness claim with no stated scope isn't verifiable in principle** — whoever wants to refute it has no way to know where to look.

### Compressed to one thing

All ten have the same shape. A signal reports pass, and nobody checked **what that signal actually observed.**

- The 200 observed a connection, not a delivery
- The count of 2 observed two channels, not two people
- The 214 green checks observed the code, not the midnight rollover
- The retrieved log observed a record, not whether it was retained

So the question isn't "did it pass." It's **"if this had failed, what would look different?"** If the answer is "nothing," you don't have a pass. You have nothing.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/false-pass-signals/signal-blind-spots.en.svg"><img src="/assets/img/posts/false-pass-signals/signal-blind-spots.en.svg" width="720" height="268" loading="lazy" alt="What four passing signals looked at and missed: HTTP 200 saw the connection, not the delivery; a count of 2 saw two channels, not head count; 214 green tests saw the code, not crossing midnight; a retrieved log saw the record, not whether it was retained."></a></div><figcaption>Every passing signal looked at something, and missed something.</figcaption></figure>

The app is shut down. The list isn't.
