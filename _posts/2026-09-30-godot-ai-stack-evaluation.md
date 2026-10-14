---
layout: post
title: "Godot + AI 조합 평가"
description: "스팀 게임을 Godot과 AI 코딩 에이전트로 만들어도 되는지에 대한 평가 기록. 검증된 턴제 게임의 규칙을 프로토타입으로 확인한 뒤 열흘간 Godot으로 재현. 규칙 시뮬레이션, 테스트 러너의 거짓 통과, 영상 프레임 대조, 사람이 직접 해 봐야 보이는 버그까지 정리."
date: 2026-09-30
tags: [build-in-public, godot, game-dev, testing, ai-agents]
---

*🇬🇧 [English version below](#english-version)*

스팀에 낼 게임을 준비하며 게임보다 먼저 답해야 할 질문 하나. **앞으로 스팀 프로젝트를 Godot 엔진과 AI 코딩 에이전트(Claude) 조합으로 진행해도 되는가.**

이 질문에 답하려고 게임 하나를 제작. 이미 시장에서 검증된 턴제 심리전 게임의 규칙을 그대로 재현하되, 원작의 아트·사운드·이름은 미사용. 모델과 사운드는 직접 제작하거나 무료 에셋 사용. 게임 아이디어의 가치가 아니라 도구 조합의 역량만 보는 테스트 케이스라서, 배포도 판매도 하지 않는 내부 평가용으로 한정.

결론부터. **이 조합으로 계속 진행.** 9월 14일 규칙을 프로토타입으로 먼저 검증, 15일부터 24일까지 열흘간 Godot으로 규칙 엔진부터 3D 연출, 사운드까지 완성. 직접 플레이하고 남긴 최종 평가는 «아주 훌륭하게 끝난 듯». 다만 가는 길에 이 조합이 어디서 거짓 신호를 내는지도 여러 번 확인. 이 글은 그 기록.

## 판정 기준

평가 축은 처음에 셋으로 고정.

| 축 | 평가 대상 |
|---|---|
| 에이전트의 기술력 | 설계·구현·검증의 실제 수행 여부. 버그의 자체 발견 여부. 실측 없는 「완료」 선언 여부 |
| Godot 활용 | 씬·노드·시그널·입력·3D·오디오 같은 엔진 고유 영역의 활용 수준 |
| 결과물 만족도 | 직접 플레이했을 때의 조작·템포·연출 만족도 |

여기에 기록할 지표로 **사람 개입 횟수**, **에이전트가 막힌 지점과 유형**, **규칙 변경부터 검증까지 걸리는 시간**, **자동 검증의 실패 검출 여부**를 추가.

## 도구 구성

| 역할 | 도구 |
|---|---|
| 엔진 | Godot 4.7.2 표준판 (GDScript) |
| 테스트 | GUT 9.7.1, 헤드리스 실행 |
| 3D 모델 | Blender 4.5 LTS + Blender MCP (에이전트가 스크립트로 모델링), 일부 Poly Haven 에셋 |
| 영상 분석 | ffmpeg |
| 사운드 | 외부 음원 0. 효과음 35종과 음악 8곡을 전부 코드로 파형 합성 |

처음 계획은 Unity. 새 리서치 결과를 받고 Godot으로 전환했는데, 전환하며 가장 먼저 걸린 것은 엔진 기능이 아니라 **사라진 안전장치**. 예전 엔진에서는 규칙을 어기면 컴파일러가 빌드를 차단. 새 엔진은 그대로 실행. 조용한 붕괴는 자동화 루프를 무의미하게 만들기 때문에, 컴파일러가 하던 역할을 테스트로 대체.

비슷한 오해 하나도 사전 차단. Godot의 씬 파일은 텍스트라 사람이 읽을 수 있는 형식. 그렇다고 에이전트의 임의 수정은 금지. 내부 식별자 때문에 손으로 고치면 파손. 「읽힌다」와 「고쳐도 된다」는 별개라서 이 구분을 작업 문서에 명시. 자동화 플래그 중 셋이 에디터 전용이라 출시용 실행 파일에는 미포함이라는 사실도 이때 확인.

## 규칙부터 증명

화면 제작 전에 규칙의 정확성부터 확인. 화면이 붙으면 버그의 출처가 규칙인지 연출인지 가르기 곤란.

9월 14일, 화면 의존이 전혀 없는 JS 프로토타입으로 규칙을 짜고 네 가지 대진 조건으로 합계 48,000판을 자동 실행. 크래시와 불변식 위반은 0건. 다음 날 같은 규칙을 GDScript로 **독립 재구현**하고 그중 한 조건을 골라 대조.

| 조건 | 3라운드 전승 확률 |
|---|---:|
| 이론값 (실력이 같으면 라운드마다 1/2) | 12.5% |
| JS · 추론 AI 대 추론 AI | 12.6% |
| JS · 무작위 대 무작위 | 12.2% |
| Godot · 무작위 대 무작위 (5,000판, 429ms) | 13.0% |

실력이 같은 대진은 두 구현 모두 이론값 근처에 집중. 무작위 대 무작위라는 같은 조건에서 독립된 두 구현이 12.2%와 13.0%를 산출했으니, 한쪽에만 있는 왜곡이라면 이렇게 일치하기 어려운 결과. JS 측정에서는 무작위 플레이어가 추론 AI를 상대하면 클리어율이 0.1%로 급락. 확률 추론이 이 게임의 핵심 실력이라는 의미. 이 교차 검증까지의 기술적 개입 횟수는 0회.

난이도는 한 번 오독. 처음 측정에서는 라운드별 승률이 49.6% → 48.4% → 50.9%로 거의 같아서, 체력과 자원을 같이 늘리면 양쪽이 상쇄된다는 결론. 그래서 수치는 그대로 두고 **AI 상대의 행동만** 단계별로 설계. 정보를 먼저 확인하는지, 아이템을 조합해서 쓰는지 같은 차이. 같은 설정에서 AI 단계만 바꿔도 플레이어 승률이 89%에서 33%까지 변동, 단계를 라운드에 배치하자 57% → 50% → 35%의 곡선 형성.

그런데 9월 19일 영상 대조(아래)에서 마지막 라운드의 체력 수치를 처음부터 잘못 잡고 있었다는 사실이 드러난 상황. 바로잡고 재측정하자 처음 분석 자체가 오류. 상쇄가 아니라, 라운드가 길어질수록 더 잘 두는 쪽으로 기우는 구조. 그 사이에 정한 「1라운드부터 아이템 지급」 결정까지 반영한 현행 곡선은 아래와 같은 수치.

| 라운드 | 1 | 2 | 3 |
|---|---:|---:|---:|
| 라운드 단독 클리어율 (AI 플레이어(레벨 2) 대 라운드별 AI, 4,000판씩) | 79.0% | 50.3% | 29.7% |

이 곡선은 AI 단계 배치, 아이템 결정, 수치 정정 세 가지의 합작. 틀린 옛 분석은 지우지 않고 「폐기」 표시와 함께 사양서에 보존.

## 자동화의 신뢰 가능성

환경을 세우고 가장 먼저 한 일은 테스트 러너의 종료 코드 실측. 일부러 실패하는 테스트는 1, 통과하는 테스트는 0. 여기까지는 신뢰 가능.

문제는 그다음. **테스트 파일이 파싱에 실패하면 GUT는 경고만 찍고 그 파일을 건너뛴 뒤 "All tests passed"와 종료 코드 0을 출력.** 종료 코드 자체는 정직했지만, 테스트가 아예 실행되지 않은 경우까지는 방어 불가. 그래서 러너를 감싸는 스크립트가 출력에서 `Ignoring script`, `SCRIPT ERROR`, `Parse Error`를 찾으면 강제로 1을 반환하도록 구성. 이 가드는 평가 기간에 최소 세 번 실제로 작동.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/godot-ai-stack-evaluation/runner-guard.ko.svg"><img src="/assets/img/posts/godot-ai-stack-evaluation/runner-guard.ko.svg" width="720" height="300" loading="lazy" alt="테스트 파일이 파싱에 실패하면 GUT는 경고만 찍고 파일을 건너뛴 뒤 All tests passed와 종료 코드 0을 출력. 래퍼 스크립트가 출력에서 Ignoring script, SCRIPT ERROR, Parse Error를 찾으면 종료 코드 1로 강제."></a></div><figcaption>종료 코드는 정직했지만 실행되지 않은 테스트까지는 방어 불가. 출력 검사 가드로 거짓 통과를 실패로 반전.</figcaption></figure>

그 밖에 이 조합에서 마주친 함정 목록.

| 증상 | 원인 | 성격 |
|---|---|---|
| 새로 만든 클래스를 테스트가 못 찾음 | 전역 클래스 캐시는 `--import` 실행으로만 갱신 | Godot 고유 |
| 초기화 함수 전체의 무력화 | 노드 타입 변경 후 `@onready` 선언 미수정 → 첫 줄에서 정지, 이후 참조 전부 null | GDScript 고유 |
| 카메라의 천장 지향 | `Transform3D` 텍스트 표기는 행 우선인데 에이전트가 열 벡터로 기재 | Godot 고유 |
| 같은 코드의 통과·실패 반복 | 매번 무작위 시드로 시작하는데 「값이 줄었는가」만 단언 | 테스트 설계 결함 |
| 모델 일부 면의 재질 어긋남 | Blender에서 면마다 찍은 재질 인덱스가 여러 연산을 거치며 몇 면씩 누출 | Blender 고유 |

불안정한 테스트는 시작 상태를 고정하고 「정확히 1만큼 줄고 턴이 유지되는가」를 보도록 변경한 뒤, 세 번 연속 같은 결과인지 확인. 재질 인덱스는 인덱스 대신 「이 위치에 있으면 이 재질」이라는 기하 규칙으로 덮어쓰고, 눈으로 본 결함을 어서션으로 고정. 그 어서션이 다음 빌드에서 오염된 면 10개를 즉시 검출.

테스트는 24개에서 시작해 최종 141개, 규칙을 건드릴 때마다 5,000판 자동 대전에서 불변식 위반 0건 확인.

## 규칙 이식은 됐지만 경험 이식은 실패

규칙과 AI가 끝나고 2D 화면을 붙여 「플레이 가능한 상태」를 완성. 한 판 해 보고 즉시 파악. 이건 채팅으로 하는 게임. 원작은 공간과 물건을 직접 만지는 감각이 경험의 대부분인데, 그 부분의 이식은 0. 에이전트가 이 단계를 「UI」로 좁혀 해석했고, 그 좁은 범위를 충실히 지킨 결과.

그래서 3D로 전면 재제작. 이때 규칙 코드의 폐기는 0줄. 규칙을 화면과 분리된 모듈로 먼저 짜 둔 덕분에 화면만 교체 가능. 기존 2D 화면은 디버그용으로 보존, 3D 조작을 얹는 동안에도 예전 버튼 경로를 살려 둬서 그 경로를 검증하던 테스트 15개가 계속 회귀를 검출.

사운드도 같은 방식. 음원을 구매하지 않고 노이즈와 감쇠 곡선으로 파형을 합성. 라이선스 걱정이 없는 대신 헤드리스 환경에서는 청취 불가라서 소리를 수치로 검사. 그 검사에서 효과음 두 개가 0dB에서 잘리던 기존 결함 발견(봉우리 값 32767), 정규화를 넣고 회귀 테스트에 추가.

## 영상에서 연출 추출

연출을 맞추려면 원작의 실제 움직임 파악이 필수. 참고 영상 8개(12분, 2560×1440)를 에이전트에게 통째로 보여 주면 토큰 감당 불가. 그래서 ffmpeg로 프레임을 뽑아 4×4 격자 한 장에 16프레임씩 담은 컨택트시트로 훑고, 결정적인 순간만 원본 해상도로 재추출. 프레임당 토큰 13배 절감.

첫 대조에서 규칙 수치 하나가 조사 자료와 다르다는 사실 발견. 그 값에 기대던 밸런스 실측을 전부 무효로 돌리고 재측정. 검색으로 모은 규칙보다 실물이 저렴하고 정확하다는 교훈.

두 번째 교훈은 간격.

| 간격 | 보인 것 | 놓친 것 |
|---|---|---|
| 2초 | 장면 구성, 물건 배치, 규칙 | 카메라 움직임, 템포 |
| 0.25초 | 카메라 5개 위치와 전환, 한 턴 5~6초, 아이템 사용 동작 | — |
| 0.125초 (밝기 곡선) | 순간적인 섬광, 화면 암전과 복귀 | — |

직접 플레이 후 「너무 빠르다」는 체감이 계기. 2초 간격 시트로는 움직임 자체가 비가시. 0.25초로 다시 떠서 카메라와 템포를 전면 교체, 섬광처럼 순간적인 장면은 프레임별 평균 밝기 곡선에서 급등과 암전 구간을 찾아 시점 측정. 이후로는 새 연출을 넣기 전에 해당 구간을 0.25초 간격으로 먼저 뜨는 것이 규칙.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/godot-ai-stack-evaluation/frame-intervals.ko.svg"><img src="/assets/img/posts/godot-ai-stack-evaluation/frame-intervals.ko.svg" width="720" height="320" loading="lazy" alt="같은 4초 구간을 2초, 0.25초, 0.125초 밝기 곡선으로 본 비교. 2초 간격은 카메라 이동 구간을 통째로 누락, 0.25초 간격은 그 구간에 여러 프레임이 포함, 밝기 곡선은 섬광과 암전 시점을 노출."></a></div><figcaption>2초 간격은 카메라 이동 구간을 통째로 누락. 0.25초 재추출로 움직임과 템포가 가시화.</figcaption></figure>

## 테스트의 사각지대

이 평가에서 가장 값진 데이터는 테스트가 전부 초록불일 때 사람이 찾은 버그들.

| 증상 | 당시 테스트 | 원인 |
|---|---|---|
| 강화 아이템이 효과가 안 난 턴에도 풀리지 않아 두 턴 뒤까지 적용 | 54/54 통과 | 효과가 났을 때만 해제하던 구조. 테스트도 그 경우만 확인 |
| 정보 아이템을 써도 결과 미표시 | 통과 | 「쓴 사람만 안다」를 구현하다 쓴 사람에게도 미표시 |
| 클릭해도 게임 진행 0 | 129/129 통과 | 화면을 덮는 UI 컨테이너가 기본 설정으로 마우스 입력을 흡수 |

세 번째가 가장 극적인 사례. 테스트는 전부 통과했는데, 실제 창에서 마우스로 누르면 반응 0. 원인은 실제 창에서 클릭을 재현해 발견. 자동 검증은 검증하려고 생각한 것만 보는 도구. 헤드리스로는 화면 확인이 불가해서 초반(9월 16일)에 화면을 PNG로 떠 주는 도구를 만들어 뒀지만, 조작 확인은 사람의 직접 플레이 외에 대안 부재. 마지막 두 차례 플레이 테스트에서 나온 지적 12건을 반영하고 평가 종료.

## 판정

| 축 | 결과 |
|---|---|
| 에이전트의 기술력 | 설계·구현·검증을 자체 완수. 러너의 거짓 통과, 불안정한 테스트, 사운드 잘림 같은 결함도 자체 발견. 다만 범위 해석이 좁으면 그대로 좁게 제작, 사람이 조작해야 보이는 버그는 미발견 |
| Godot 활용 | 씬 구성, 3D 조명과 카메라 전환, 후처리 셰이더, 오디오 버스, 마우스 입력까지 엔진 고유 영역을 두루 활용. 마주친 함정은 전부 재현 조건과 함께 문서화 |
| 결과물 만족도 | 플레이 테스트 지적 반영 후 «아주 훌륭하게 끝난 듯» |

그래서 다음 스팀 게임도 이 조합으로 진행. 대신 이번에 얻은 조건 세 가지를 처음부터 기본 탑재.

- 테스트 러너에는 **건너뛴 파일을 실패로 만드는 가드**를 기본 장착.
- 계획표에 **사람이 직접 한 판 하는 단계**를 고정. 테스트 통과로 대체 불가.
- 연출은 **0.25초 간격 대조**부터 시작, 범위는 「UI」가 아니라 「경험」으로 기재.

---

## English version

Before building a Steam game, I needed an answer to one question: **should my Steam projects be built with Godot and an AI coding agent (Claude)?**

To answer it, I built a game. I reproduced the rules of an existing, market-proven turn-based mind game without using any of its art, audio, or names; models and sounds were made in-house or came from free asset libraries. The point wasn't the game idea — only the toolchain was on trial — so it stayed internal: not released, not sold.

The verdict up front: **I'm keeping this stack.** After validating the rules in a prototype on Sept 14, it took ten days in Godot (Sept 15–24) to go from a rules engine to a finished 3D game with its own audio. The note I left after my final play-through: "looks like this ended really well." Along the way, though, the stack gave me false signals more than once. That's what this post is about.

### The rubric

I fixed three axes before starting: the agent's engineering (does it design, build, and verify on its own, find its own bugs, and never claim "done" without measuring?), use of Godot (scenes, nodes, signals, input, 3D, audio — the engine-specific parts), and whether the result actually feels good when I play it. I also tracked how often I had to step in, where the agent got stuck, how long a rule change took to verify, and whether automated checks caught failures as failures.

### The toolchain

| Role | Tool |
|---|---|
| Engine | Godot 4.7.2 standard (GDScript) |
| Tests | GUT 9.7.1, headless |
| 3D models | Blender 4.5 LTS + Blender MCP (the agent models via scripts), a few Poly Haven assets |
| Video analysis | ffmpeg |
| Audio | No external audio. 35 SFX and 8 music tracks, all synthesized in code |

The original plan was Unity; I switched to Godot after new research came in. The first thing that bit wasn't a missing feature but a missing guardrail. In the old engine, breaking certain rules failed the build. In Godot the code just runs. A silent failure would hollow out the whole automation loop, so tests now do the job the compiler used to do. I also wrote down early that "scene files are readable text" doesn't mean "the agent may hand-edit them" — internal IDs break when you do.

### Prove the rules first

Before any visuals, I wanted proof the rules were right, because once a screen is attached you can no longer tell a rules bug from a presentation bug.

On Sept 14 I wrote the rules as a UI-free JS prototype and ran 48,000 automated games across four matchups: zero crashes, zero invariant violations. The next day the same rules were reimplemented independently in GDScript and checked against one of those matchups.

| Matchup | Chance of winning all 3 rounds |
|---|---:|
| Theory (equal skill, 1/2 per round) | 12.5% |
| JS · reasoning AI vs reasoning AI | 12.6% |
| JS · random vs random | 12.2% |
| Godot · random vs random (5,000 games, 429 ms) | 13.0% |

Equal-skill matchups cluster around theory, and on the identical random-vs-random matchup two independent implementations gave 12.2% and 13.0%, which is hard to get if either one were quietly skewed. In the JS run, a random player cleared only 0.1% against the reasoning AI: probabilistic reasoning is the core skill of this game. I made zero technical interventions up to this point.

I misread the difficulty once. The first measurement had round win rates nearly flat (49.6% → 48.4% → 50.9%), and I concluded that scaling health and resources together cancels out. So I left the numbers alone and tiered only the **AI opponent's behavior**: whether it gathers information first, whether it chains items. Changing just the AI tier moved the player's win rate from 89% to 33%, and placing tiers per round produced a 57% → 50% → 35% curve.

Then the Sept 19 video comparison (below) showed I'd had the final round's health value wrong from the start. With it corrected, the original analysis itself turned out wrong: longer rounds don't cancel out, they tilt toward whoever plays better. Including a separate decision to hand out items from round 1, the current curve is:

| Round | 1 | 2 | 3 |
|---|---:|---:|---:|
| Single-round clear rate (level-2 AI player vs per-round AI, 4,000 games each) | 79.0% | 50.3% | 29.7% |

That curve comes from all three together: AI tiers, the item decision, and the corrected value. The wrong analysis stays in the spec, marked as discarded.

### Can the automation be trusted?

First thing after setup, I measured the test runner's exit codes: 1 for a deliberately failing test, 0 for a passing one. So far, trustworthy.

Then: **when a test file fails to parse, GUT prints a warning, skips the file, and reports "All tests passed" with exit code 0.** The exit code was honest; it just had no way to cover tests that never ran. I wrapped the runner so that `Ignoring script`, `SCRIPT ERROR`, or `Parse Error` in the output forces exit code 1. That guard fired for real at least three times during the evaluation.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/godot-ai-stack-evaluation/runner-guard.en.svg"><img src="/assets/img/posts/godot-ai-stack-evaluation/runner-guard.en.svg" width="720" height="300" loading="lazy" alt="When a test file fails to parse, GUT warns, skips it, and reports All tests passed with exit code 0. A wrapper script scans the output for Ignoring script, SCRIPT ERROR, or Parse Error and forces exit code 1."></a></div><figcaption>The exit code was honest, but it couldn’t cover tests that never ran. A guard on the output turns the false pass into a failure.</figcaption></figure>

Other traps worth knowing:

- New classes invisible to tests until `--import` refreshes the global class cache.
- Changing a node's type without updating its `@onready` declaration kills the whole init function on line one; every later reference is null.
- `Transform3D` in text form is row-major; the agent wrote columns and the camera stared at the ceiling.
- A test that flipped between pass and fail because games start from a random seed and it only asserted "the value went down." Fixed by pinning the starting state and asserting an exact decrement, then requiring three identical runs.
- Per-face material indices set in Blender leaked a few faces through later operations. I stopped trusting indices, reassigned materials from geometric rules, and turned the defect I'd seen by eye into an assertion — which caught 10 contaminated faces on the very next build.

Tests grew from 24 to 141, and every rule change was followed by 5,000 automated games with zero invariant violations.

### I ported the rules, not the experience

With rules and AI done, I put a 2D UI on it and played. It was a chat game. The original's whole experience, the feel of handling objects in a physical space, hadn't been ported at all. The agent had read that step as "UI" and built exactly that narrow scope.

So everything was rebuilt in 3D, without discarding a single line of rules code. Because the rules lived in a module separate from presentation, only the screen had to change. The 2D screen stayed as a debug view, and the old button path was kept alive under the 3D controls so the 15 tests built on it kept catching regressions.

Audio followed the same approach: waveforms synthesized from noise and envelopes. No licensing worries, but nothing can be heard in a headless run, so sounds are checked numerically. That check exposed an existing bug, two sound effects clipping at 0 dB (peak 32767), now normalized and covered by a regression test.

### Extracting motion from video

Feeding eight reference videos (12 minutes, 2560×1440) straight to the agent would blow the token budget. Instead ffmpeg pulls frames into 4×4 contact sheets, and only decisive moments get re-extracted at full resolution — about 13× cheaper per frame. The first pass also revealed that one rule value in my research was wrong, invalidating every balance measurement built on it. The real footage was cheaper and more accurate than search results.

The bigger lesson was sampling interval. At 2-second intervals I could see layout, objects, and rules — but no camera motion or pacing, which is exactly what felt "too fast" when I played. Re-sampling at 0.25 s gave five camera positions and their transitions, a 5–6 second turn, and the item animations. For split-second moments, I found the timing from per-frame average brightness (where a flash spikes, where the screen blacks out), measured in 0.125 s steps. Now every new piece of motion starts with a 0.25 s pass over the relevant footage.

<figure class="diagram"><div class="fig-scroll"><a href="/assets/img/posts/godot-ai-stack-evaluation/frame-intervals.en.svg"><img src="/assets/img/posts/godot-ai-stack-evaluation/frame-intervals.en.svg" width="720" height="320" loading="lazy" alt="The same 4-second span sampled every 2 s, every 0.25 s, and as a 0.125 s brightness curve. 2 s sampling skips the camera move entirely, 0.25 s lands several frames inside it, and the brightness curve exposes the flash and blackout timing."></a></div><figcaption>Sampling every 2 s skips a camera move entirely. Re-sampling at 0.25 s brought motion and pacing into view.</figcaption></figure>

### What the tests couldn't see

The most valuable data came from bugs found by a human while every test was green.

| Symptom | Tests at the time | Cause |
|---|---|---|
| A buff item stayed active through a turn where it didn't trigger, and applied two turns later | 54/54 passing | It was only cleared when it triggered, and the tests only checked that case |
| Using the info item showed nothing | passing | "Only the user knows" was implemented so that not even the user knew |
| Clicking did nothing at all | 129/129 passing | A full-screen UI container was swallowing mouse input by default |

The last one is the clearest: every test passed, while a real mouse in a real window did nothing. It was found by reproducing clicks in a real window. Automated checks only see what you thought to check. Headless runs can't see the screen, so early on (Sept 16) I'd built a tool that renders it to PNGs, but controls still need someone to play. Twelve issues from the final two play-tests were fixed before I signed off.

### Verdict

The agent designed, built, and verified on its own, and found its own defects — the runner's false pass, the flaky test, the clipping audio. Its limits were equally clear: a narrow scope gets built narrowly, and bugs that only show up under a human's hands stay invisible to it. Godot held up across scenes, 3D lighting and camera work, post-processing shaders, audio buses, and input, and every trap is documented with repro conditions.

So the next Steam game uses this stack, starting with three defaults from day one: a runner guard that fails on skipped files, a fixed "a human plays it" stage in every plan (passing tests never substitute for it), and motion work that begins with a 0.25 s video pass — scoped as "the experience," not "the UI."
