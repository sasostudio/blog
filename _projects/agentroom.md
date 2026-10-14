---
layout: project
title: AgentRoom
headline: 에이전트의 "완료"를 믿지 않는 오케스트레이션
status: CLOSED · 2026.10
archived: true
closed: 2026-10-03
order: 4
blurb: 사람이 게이트를 쥐는 Claude Code 멀티에이전트 오케스트레이션 — planner·dev·qa 핑퐁 + 필요할 때만 붙는 researcher, 되돌리기 어려운 건 사람이 승인.
meta: [Claude Code, v1.1.0, 2026]
steps:
  - k: "문제 / Problem"
    h: 코드를 쓴 에이전트가 합격 판정까지 스스로 냈다
    p: 한동안은 AI 에이전트 "팀"을 손으로 굴렸다. 총괄 세션 하나, 개발 세션 하나, 검수 세션 하나를 띄워놓고 한쪽의 결과물을 복사해 다른 쪽에 붙여넣었다. 역할을 나누니 품질은 올라갔지만 내가 하루 종일 말을 나르는 심부름꾼이 됐다. 그렇다고 완전 자동 오케스트레이터로 가면 다른 문제가 생긴다 — 방향이 틀어져도 끝까지 달리고, 무엇보다 코드를 쓴 에이전트가 "완료했습니다"라는 판정까지 스스로 내린다. 검증할 수단이 없는 자기 보고였다.
  - k: "현황 / Status"
    h: 작업 엔진으로 합치고, 그 엔진을 공개했다
    p: 슬래시 커맨드 1개와 마크다운 에이전트 정의 4개(planner·dev·qa + 기본 꺼짐 researcher)로 이뤄진 v1.1.0까지 MIT로 공개해 운영했다. 2026년 10월, 모델이 강해지면서 역할을 잘게 나눌수록 얻는 것보다 잃는 것이 커졌다 — 같은 원칙(주장을 믿지 않는 읽기 전용 평가 · 되돌리기 어려운 일은 사람 게이트)을 「만드는 AI 하나 + 평가하는 AI 하나」로 줄인 개인용 작업 엔진에 합쳤다. AgentRoom 레포는 비공개로 돌렸고, 같은 원칙을 이어받은 엔진을 2026년 10월 AgentWorkflow로 공개했다. 글은 그대로 남겨 둔다.
  - k: "강점 / Why it works"
    h: 주장을 믿지 않고, 되돌릴 수 없는 일은 사람에게 묻는다
    p: 읽기 전용 qa가 실제 파일을 직접 읽어 dev의 주장을 반박하고, qa의 APPROVED 없이는 작업이 끝나지 않는다. push·삭제·배포처럼 되돌리기 어려운 일은 전부 사람 게이트를 거친다. 별도 서버나 프레임워크 없이 구독만으로 돌고, 능력을 더하는 대신 안 쓸 때를 정하는 쪽으로 설계했다 — 네 번째 에이전트 researcher가 기본 꺼짐인 이유다.
links:
  - { label: "후속 공개판: AgentWorkflow (GitHub · MIT)", url: "https://github.com/sasostudio/agentworkflow" }
  - { label: "Dev Log: 만든 이야기", url: "/2026/07/05/agentroom-human-gated-orchestration/" }
  - { label: "Dev Log: v1.1.0 — 네 번째 에이전트", url: "/2026/07/07/agentroom-fourth-agent-default-off/" }
---
