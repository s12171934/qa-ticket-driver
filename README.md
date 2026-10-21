---
title: qa-ticket-driver
created: 2026-10-21
updated: 2026-10-21
tags: [domain/인공지능, area/ai에이전트]
source: ai
---

# qa-ticket-driver

> QA 부모 티켓의 신규·재오픈 하위 티켓마다 전용 워크트리와 코디네이터 에이전트를 자동으로 띄우고, 코디네이터가 같은 워크트리의 워커로 구현·검토한 뒤 사용자 승인을 받아 반출하게 하는 드라이버 스킬이다.

## 하는 일

- 주기 회차(`qa-driver run`)에서 Jira 로 대상 티켓을 고르고, 티켓마다 Orca 워크트리를 만들어 코디네이터를 브리프로 기동한다.
- 코디네이터·워커·사용자 사이의 진행은 상태 디렉터리의 신호 파일로 주고받는다. 단계 판정 규칙은 공개 계약으로 둔다.
- 반출 승인·워크트리 정리·제외를 CLI 로 제공한다. 대시보드가 같은 CLI 와 계약으로 제어·표시할 수 있다.

## 설치

```bash
git clone https://github.com/s12171934/qa-ticket-driver.git ~/.claude/skills/qa-ticket-driver
# 다른 도구는 그 도구의 스킬 경로에 링크한다. 예: ln -s ~/.claude/skills/qa-ticket-driver ~/.codex/skills/qa-ticket-driver
```

- 전제: macOS, `orca` CLI, `git`, Python 3, Jira Cloud API 토큰.

## 구성과 공개 면

- [[SKILL|SKILL.md]] — 스킬 본문. 형식은 에이전트 도구가 정한다.
- `bin/qa-driver` — CLI (Python 3 표준 라이브러리만).
- `references/` — 브리프·워커 의무·설정 견본 (내부 전용).
- [[exports/EXPORTS|exports/EXPORTS.md]] — 공개 파일 목록. [[exports/state-contract|상태 계약]] 이 상태 디렉터리 배치와 단계 판정을 정한다.
- 공개 면은 스킬 이름(`qa-ticket-driver`), `bin/qa-driver` 의 하위 명령, `exports/` 다.
