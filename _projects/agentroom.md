---
layout: project
title: AgentRoom
headline: 에이전트의 "완료"를 믿지 않는 오케스트레이션
status: CLOSED · 2026.10
archived: true
closed: 2026-10-03
order: 4
blurb: 사람이 게이트를 쥐는 Claude Code 멀티에이전트 오케스트레이션 — planner·dev·qa 순환 + 필요할 때만 합류하는 researcher, 되돌리기 어려운 일은 사람의 승인 필수.
meta: [Claude Code, v1.1.0, 2026]
steps:
  - k: "문제 / Problem"
    h: 코드를 쓴 에이전트의 셀프 합격 판정
    p: 한동안은 AI 에이전트 「팀」의 수동 운영. 총괄 세션 하나, 개발 세션 하나, 검수 세션 하나를 띄워 두고 한쪽의 결과물을 복사해 다른 쪽에 붙여넣는 방식. 역할 분리로 품질은 향상됐지만 하루 종일 말을 나르는 심부름꾼 신세. 그렇다고 완전 자동 오케스트레이터로 가면 다른 문제 발생 — 방향이 틀어져도 끝까지 질주, 무엇보다 코드를 쓴 에이전트가 "완료했습니다"라는 판정까지 자체 결정. 검증 수단이 없는 자기 보고.
  - k: "현황 / Status"
    h: 작업 엔진으로 통합, 그 엔진의 공개
    p: 슬래시 커맨드 1개와 마크다운 에이전트 정의 4개(planner·dev·qa + 기본 꺼짐 researcher)로 구성된 v1.1.0까지 MIT로 공개·운영. 2026년 10월, 모델 성능 향상으로 역할을 잘게 나눌수록 얻는 것보다 잃는 것이 커진 상황 — 같은 원칙(주장을 믿지 않는 읽기 전용 평가 · 되돌리기 어려운 일은 사람 게이트)을 「만드는 AI 하나 + 평가하는 AI 하나」로 줄인 개인용 작업 엔진에 통합. AgentRoom 레포는 비공개 전환, 같은 원칙을 이어받은 엔진은 2026년 10월 AgentWorkflow로 공개. 글은 그대로 보존.
  - k: "강점 / Why it works"
    h: 주장 불신, 되돌릴 수 없는 일은 사람의 확인
    p: 읽기 전용 qa가 실제 파일을 직접 읽어 dev의 주장을 반박, qa의 APPROVED 없이는 작업 종료 불가. push·삭제·배포처럼 되돌리기 어려운 일은 전부 사람 게이트 경유. 별도 서버나 프레임워크 없이 구독만으로 동작, 능력 추가보다 미사용 조건 결정에 무게를 둔 설계 — 네 번째 에이전트 researcher의 기본값이 꺼짐인 이유.
links:
  - { label: "후속 공개판: AgentWorkflow (GitHub · MIT)", url: "https://github.com/sasostudio/agentworkflow" }
  - { label: "Dev Log: 만든 이야기", url: "/2026/07/05/agentroom-human-gated-orchestration/" }
  - { label: "Dev Log: v1.1.0 — 네 번째 에이전트", url: "/2026/07/07/agentroom-fourth-agent-default-off/" }
---
