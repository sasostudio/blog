---
layout: page
title: About
permalink: /about/
---

<img src="/assets/img/saso-mark.png" alt="Saso Studio 로고" style="max-width:200px" loading="lazy">

## Saso Studio

사소 스튜디오는 게임과 그 운영을 지탱하는 도구를 만드는 개발 스튜디오. 기획부터 구현·검수·배포·운영까지 전부 직접 제작.

일하는 방식의 요약은 세 마디 — **제작, 검증, 공개.**

- **제작**: 아이디어를 빠르게 최소 기능으로 만들어 실사용으로 판정. 답이 없는 프로젝트는 미련 없이 정리하고 회고 작성.
- **검증**: 에이전트든 사람이든 "됐습니다"라는 주장 대신 상태를 직접 확인. 이 원칙에서 AgentRoom이 탄생했고, 지금은 오픈소스 작업 엔진 [AgentWorkflow](https://github.com/sasostudio/agentworkflow)로 계승.
- **공개**: 만드는 과정에서 배운 것과 부서진 것을 이 사이트의 Dev Log에 그대로 기록.

## 운영 중

| 프로젝트 | 소개 | 상태 |
|---|---|---|
| [뭉치뭉치](/projects/moongchi/) | 물리 기반 머지 퍼즐 게임 — 웹 · 원스토어 · Google Play · 앱인토스(토스 앱) 네 채널 출시 | LIVE |

## 지난 프로젝트

운영을 마친 프로젝트는 [Archive](/archive/)에 보관.

| 프로젝트 | 소개 | 상태 |
|---|---|---|
| [AgentRoom](/projects/agentroom/) | 사람이 게이트를 쥐는 Claude Code 멀티에이전트 오케스트레이션 — 2026-10 개인용 작업 엔진으로 통합, 레포 비공개 전환(후속 공개판 AgentWorkflow) | CLOSED |
| [밤샘팟](/projects/night-pod/) | 오늘 밤 같이 안 자는 사람 수를 보여주는 앱인토스 미니앱 — 2026-09-30 운영 종료 | CLOSED |
| [수분 코치](/projects/hydro-coach/) | 체감온도로 수분 목표를 동적 산정하는 앱인토스 미니앱 — D1 리텐션 0으로 개발 중단, 2026-09-30 운영 종료 | CLOSED |
| [아홉 개의 마침표](/projects/nine-postmortem/) | 종료한 프로젝트 9개의 회고 — 빠르게 검증하고 정리하는 규율 | CLOSED |

## 기술 스택

- **프론트/앱**: TypeScript · React · Vite · WebView 미니앱(앱인토스) · Capacitor(Android)
- **백엔드/인프라**: Firebase(Auth·Firestore) · Cloudflare Workers(D1·KV·cron) · Vercel · Supabase
- **게임**: 물리 기반 캐주얼(웹) · 실시간 1대1 대전(서버 권위 검증)
- **AI/에이전트**: Claude Code 멀티에이전트 오케스트레이션
- **운영**: 스토어 출시·심사 대응(원스토어·앱인토스) · build-in-public 기록 파이프라인

## Contact

- Email: [sasostudio.dev@gmail.com](mailto:sasostudio.dev@gmail.com)
- GitHub: [@sasostudio](https://github.com/sasostudio)

---

## English

Saso Studio is a development studio building games and the tooling that keeps them running — planning, building, reviewing, shipping, and operating everything in-house.

Three words sum up how the studio works — **build, verify, publish.**

- **Build**: ship a minimal core fast, judge it by real use, and close projects that don't earn their place — with a postmortem every time.
- **Verify**: never trust "done" claims, from agents or from ourselves; check the actual state. That principle produced AgentRoom, now carried on by the open-source work engine [AgentWorkflow](https://github.com/sasostudio/agentworkflow).
- **Publish**: everything we learn (and break) along the way goes into the Dev Log on this site.

### Live

| Project | What it is | Status |
|---|---|---|
| [MoongchiMoongchi](/projects/moongchi/) | Physics-based merge puzzle game — shipped on web, ONE store, Google Play, and Apps in Toss | LIVE |

### Archive

Closed projects live in the [Archive](/archive/).

| Project | What it is | Status |
|---|---|---|
| [AgentRoom](/projects/agentroom/) | Human-gated multi-agent orchestration for Claude Code — merged into a personal work engine in 2026-10; repository made private (successor: AgentWorkflow) | CLOSED |
| [Night Pod](/projects/night-pod/) | Apps-in-Toss mini-app that counts who else is still awake tonight — closed 2026-09-30 | CLOSED |
| [Hydro Coach](/projects/hydro-coach/) | Apps-in-Toss mini-app that sets daily hydration goals from real weather data — development stopped on zero D1 retention, closed 2026-09-30 | CLOSED |
| [Nine Full Stops](/projects/nine-postmortem/) | Postmortem of nine killed projects — the discipline of fast validation | CLOSED |

### Contact

- Email: [sasostudio.dev@gmail.com](mailto:sasostudio.dev@gmail.com)
- GitHub: [@sasostudio](https://github.com/sasostudio)
