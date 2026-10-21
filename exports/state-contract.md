---
title: qa-ticket-driver 상태 계약
created: 2026-10-21
updated: 2026-10-21
tags: [domain/인공지능, area/ai에이전트]
source: ai
---

# qa-ticket-driver 상태 계약

> 상태 디렉터리의 파일 배치와 티켓 단계 판정 규칙. 이 스킬 밖(대시보드 등)이 읽어도 되는 공개 면이다.

## 위치

`$QA_DRIVER_HOME/<부모 키 소문자>/` (기본 `~/.qa-driver/`). 예: `~/.qa-driver/qa-100/`.

| 경로 | 쓰는 쪽 | 내용 |
| --- | --- | --- |
| `config.env` · `context.md` | 사람 | 설정 · 기능 맥락 |
| `claimed/<KEY>` | 드라이버 | 기동마다 한 줄 `시각 \t epoch \t 코디네이터 터미널 \t 제목`. 줄 수 = 회차 |
| `tickets/<KEY>.json` | 드라이버 | `{key, parent, summary, round, worktree{path,id,created_at}, coordinator{handle,at,round}, workers{<이름>: {handle, submodule, at}}, updated_at}` |
| `briefs/<KEY>.md` | 드라이버 | 렌더된 코디네이터 브리프 |
| `handoffs/<KEY>-<이름>.md` | 코디네이터 | 서브모듈별 인계 브리프. 있으면 그 워커가 기동됐다 |
| `done/<KEY>-<이름>.md` | 워커 | 서브모듈별 커밋 완료 |
| `ready/<KEY>.md` | 코디네이터 | 검토 완료 — 반출 승인 대기 |
| `approved/<KEY>.md` | 사용자(대시보드·`qa-driver approve`) | 반출(push·PR) 승인 |
| `delivered/<KEY>.md` | 코디네이터 | 저장소별 PR 주소 |
| `cleaned/<KEY>` | `qa-driver cleanup` | 워크트리 정리 완료 |
| `blocked/<KEY>[-<이름>].md` | 코디네이터·워커 | 막힘. 셋째 줄이 한 줄 요약 |
| `ignored/<KEY>` | 사용자 | 선별에서 제외 |
| `rounds/<KEY>/<n>/<종류>/` | 드라이버 | 재오픈 때 옮긴 이전 회차 신호 |
| `log/<YYYYMMDD>.log` | 드라이버 | 회차·기동·승인·정리 기록 |

`<이름>` 은 서브모듈 경로의 `/` 를 `-` 로 바꾼 것(서브모듈이 없는 저장소는 `root`).

## 단계

위에서부터 처음 맞는 것이 그 티켓의 단계다. `blocked` 와 `ignored` 는 단계가 아니라 덧붙는 표시다.

| 순서 | 단계 | 조건 |
| --- | --- | --- |
| 9 | `cleaned` 정리됨 | `cleaned/<KEY>` |
| 8 | `delivered` 반출됨 | `delivered/<KEY>.md` |
| 7 | `approved` 반출 승인 | `approved/<KEY>.md` |
| 6 | `committed` 검토 완료 | `ready/<KEY>.md` |
| 5 | `reviewing` 검토 중 | `handoffs/` 의 이 티켓 파일(`-deliver.md` 제외) 수 ≤ `done/` 의 이 티켓 파일 수, 그리고 하나 이상 |
| 4 | `implementing` 구현 중 | `handoffs/` 에 이 티켓 파일이 있다 |
| 3 | `analysis` 분석 중 | 코디네이터가 기동됐다 (`tickets/<KEY>.json` 의 `coordinator`, 티켓 기록이 없는 옛 워처면 `claimed/<KEY>`) |
| 2 | `worktree` 워크트리 준비 | `tickets/<KEY>.json` 에 `worktree` 만 있다 |
| 1 | `selected` 선별됨 | 그 밖 |

- 신호 파일 이름은 `<KEY>.md` 또는 `<KEY>-<이름>.md` 다. `<KEY>.round1.md` 처럼 점이 든 이름은 옛 워처의 회차 사본이라 세지 않는다.
- 진행 중 티켓 수(동시 상한 `QA_MAX_ACTIVE` 에 쓰는 값) = `delivered`·`cleaned` 가 아니고, 막힘·제외도 아닌 티켓.
- `qa-driver status <PARENT> --json` 이 같은 규칙으로 판정한 결과를 낸다.
