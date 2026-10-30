---
name: qa-ticket-driver
description: QA 부모 티켓(릴리스 QA 에픽 등) 하나를 주기적으로 조회해, 신규·재오픈 하위 티켓마다 전용 워크트리를 만들고 그 안에 코디네이터 에이전트를 브리프로 띄우는 드라이버. 코디네이터는 원인을 코드로 짚어 기록하고 같은 워크트리에 서브모듈별 워커를 띄워 구현시킨 뒤 검토하고 실측 워커로 화면을 재게 한 다음, 사용자 승인 뒤에만 반출한다. "QA 드라이버 설정", "QA 티켓마다 워크트리·코디네이터 띄워", "QA 에픽 자동 처리", 드라이버 시작·중지·현황·승인·정리 요청, 또는 이 드라이버의 브리프로 기동된 세션에서 쓴다. 티켓 하나를 손으로 고치는 일에는 쓰지 않는다.
---

# QA 티켓 드라이버

드라이버 하나 = QA 부모 티켓 하나. 회차마다 하위 티켓 중 신규·재오픈을 고르고, 티켓마다 **전용 워크트리 하나**를 만들어(재오픈이면 재사용) 그 안의 새 터미널에 코디네이터를 띄운다. `QA_PARENT_WORKTREE` 를 두면 티켓 워크트리는 그 에픽 워크트리의 Orca 자식이 된다.
드라이버 자신은 분석·코드 수정·Jira 쓰기를 하지 않는다. 기동 뒤는 브리프를 따르는 세션의 몫이다.

```
선별(run) ─► 워크트리(drive) ─► 코디네이터: 분석·기록 ─► 워커(같은 워크트리, 서브모듈당 1) ─► 코디네이터 검토
   ─► 실측 워커(화면을 바꾸는 수정만) ─► ready ─► 사용자 승인(approve) ─► 코디네이터 반출(push·PR) ─► delivered ─► 사용자 머지 지시(merge) ─► 코디네이터 머지 ─► 정리(cleanup)
```

**소유하는 것과 넘기는 것.** 이 스킬은 오케스트레이션만 소유한다 — 선별, claim, 워크트리·터미널 기동, 브리프, 상태 디렉터리, 코디네이터·워커 사이 신호. 티켓 규칙(Jira 분석·방향 확인·상태 전이·코멘트, 버그 수정 절차)과 저장소 규칙(베이스·커밋 형식)은 소유하지 않는다 — `config.env` 의 `QA_TICKET_SKILLS` 와 `QA_HARNESS_DOCS` 가 가리키는 쪽이 정본이고, 브리프는 그것을 가리키기만 한다.

## 전제
- `orca` CLI(워크트리·터미널), `git`, Python 3, Jira Cloud REST 토큰.
- 티켓 워크트리를 만들 저장소가 Orca 에 등록돼 있다. 서브모듈을 쓰는 저장소면 주 체크아웃(`QA_REPO`)에 서브모듈이 채워져 있어야 `--reference` 로 빠르게 채운다.

## 파일
| 위치 | 내용 |
| --- | --- |
| `bin/qa-driver` | CLI — `init` · `run` · `drive` · `worker` · `status` · `approve` · `merge` · `tell` · `cleanup` · `ignore` · `restart` · `close` · `retire` · `adopt` (`qa-driver --help`) |
| `references/brief-coordinator.md` · `brief-reopen.md` | 코디네이터 브리프 (재오픈 = 머리말 + 본 브리프) |
| `references/worker-duties.md` · `screen-duties.md` | 코드 워커·화면 실측 워커 의무. 드라이버가 상태 디렉터리로 렌더한다 |
| `references/config.env.example` · `context.md.example` | `init` 이 복사하는 견본 |
| `exports/state-contract.md` | 상태 디렉터리 배치와 단계 판정 — 대시보드 등 밖에서 읽는 공개 면 |
| `$QA_DRIVER_HOME/<key>/` (기본 `~/.qa-driver/`) | 설정·신호·로그. **저장소에 두지 않는다** — 여러 워크트리가 함께 쓰고 워크트리보다 오래 산다 |

## 설정
1. `qa-driver init <PARENT>` 로 상태 디렉터리를 만든다.
2. **방향 게이트를 여기서 한 번 묻는다.** 「원인·방향이 코드로 확정되면 묻지 않고 구현할까요?」 — 답을 `QA_AUTO_HANDOFF=yes|no` 로 적는다. 추측해 채우지 않는다(비어 있으면 `run` 이 거부한다).
3. `config.env` 의 나머지(필수: `JIRA_SITE` `JIRA_EMAIL` `QA_REPO` `QA_STATUS_NEW` `QA_STATUS_REOPEN`, 토큰 키체인)와 `context.md` 를 채운다. **기능 전용 지식은 `context.md` 에만** 쓴다.
4. `qa-driver run <PARENT> --dry-run` — 부작용 없이 토큰·JQL·기동될 티켓을 확인한다.
5. **TC 시트 링크를 여기서 한 번 묻는다.** 「QA 진행도를 대시보드에 띄울 TC 스프레드시트 링크가 있나요?」 — 없다고 하면 건너뛴다. 추측해 채우지 않는다. 링크가 있으면:
   - 링크의 `/d/<fileId>/` 에서 fileId 를 뽑고, 6번 자동화 프롬프트의 「다른 일은 하지 않는다.」 앞에 다음을 넣는다: `그다음 TC 시트 진행도를 갱신한다: Google Drive 커넥터의 download_file_content 를 fileId <fileId> 로(exportMimeType 없이) 호출하고, 받은 content 문자열을 그대로 <agent-dashboard>/server/aqa sheet <PARENT> '<content>' 로 넘긴 뒤 aqa 의 출력 한 줄을 보고하라. 시트 갱신이 실패해도 드라이버 결과는 그대로 보고한다.`
   - agent-dashboard 의 `server/prefs.json` `qa_sets` 에서 이 에픽 항목에 `"sheet": "<링크>"` 를 넣고(항목이 없으면 `epic`·`project`·`watch` 와 함께 더한다) 대시보드 서버를 재시작한다(`launchctl kickstart -k gui/$(id -u)/local.agent-dashboard`, prefs 는 시작할 때만 읽는다).
   - 시트의 첫 탭이 「Test Summary」 양식(총 TC·Pass·Fail·Blocked·Dropped·N-Test)이어야 한다. 처음 한 번 위 문장대로 직접 받아 `aqa` 가 읽는지 확인한다. 갱신은 에이전트가 뜨는 회차에만 일어난다(precheck 가 건너뛰는 회차에는 없음).
6. 스케줄을 건다. Orca automation 이 기본이다(Orca 에서 보이고 대시보드에서 켜고 끌 수 있다):
   ```bash
   orca automations create --name "<PARENT> QA 드라이버" --trigger "*/10 * * * *" --workspace path:<QA_REPO> \
     --precheck "<스킬>/bin/qa-driver run <PARENT> --pending" \
     --prompt "<스킬>/bin/qa-driver run <PARENT> 를 한 번 실행하고 마지막 줄만 보고하라. 다른 일은 하지 않는다." --provider claude
   ```
   precheck 가 할 일이 없는 회차를 건너뛰므로 한가한 회차에는 에이전트가 뜨지 않는다. 스케줄 실행은 셸 프로필 없이 뜨므로 토큰은 키체인(`QA_JIRA_TOKEN_KEYCHAIN_SERVICE`)에 둔다. launchd·cron 도 된다 — 회차는 lock 과 claim 으로 멱등이다.

## 선별 규칙
- JQL 기본값 `parent = <PARENT> AND status IN (<신규>, <재오픈>)`. 직계 하위가 아니면 `QA_JQL` 로 덮어쓴다.
- 한 번 기동한 티켓은 **마지막 재오픈 시각(changelog 기준)이 마지막 claim 보다 늦을 때만** 다시 기동한다. 재기동 때 이전 회차 신호는 `rounds/<KEY>/<n>/` 로 옮기고, 같은 워크트리를 다시 쓴다.
- `ignored/<KEY>` 는 영구 제외(`qa-driver ignore`).
- 진행 중 티켓이 `QA_MAX_ACTIVE` 에 닿으면 새 티켓을 띄우지 않는다. 회차당 상한은 `QA_MAX_NEW_PER_RUN`.
- 워크트리·터미널 생성이나 브리프 전송이 실패하면 claim 을 남기지 않아 다음 회차가 다시 시도한다.

## 코디네이터와 워커 (브리프가 정한다)
- 코디네이터: 분석 → 기록 → 필요한 서브모듈 populate·브랜치 → 서브모듈마다 `handoffs/<KEY>-<이름>.md` 를 쓰고 `qa-driver worker` 로 워커 기동 → `done/` 을 받아 diff·검증 재실행 → 화면을 바꾸는 수정이면 `handoffs/<KEY>-screen.md` 를 쓰고 `qa-driver worker <PARENT> <KEY> --screen` 으로 실측 워커 기동 → `done/<KEY>-screen.md` 대조 → `ready/<KEY>.md` → 승인 대기 → 반출 → `delivered/<KEY>.md`.
- 워커: 맡은 서브모듈 하나만 수정·검증(red 확인)·로컬 커밋 → `done/<KEY>-<이름>.md`. push 하지 않는다.
- 실측 워커: 코드를 건드리지 않고 브랜치를 실제 화면에 띄워 시나리오별 기대값을 재고 녹화한다(`QA_SCREEN_SKILLS`) → `done/<KEY>-screen.md`. 코디네이터는 직접 재지 않는다.
- 워커가 도는 동안 코디네이터는 그 워크트리에서 checkout·편집하지 않는다.

## 사람이 하는 일
- **현황**: `qa-driver status <PARENT>` (또는 상태 계약을 읽는 대시보드).
- **승인**: `ready/` 의 검증(특히 red 확인)을 읽고 `qa-driver approve <PARENT> <KEY>` — `approved/<KEY>.md` 를 남기고 코디네이터 터미널에 알린다(닫혔으면 그 워크트리에 반출용 코디네이터를 새로 띄운다).
- **머지 지시**: 반출된(`delivered/`) 티켓의 PR 을 확인하고 `qa-driver merge <PARENT> <KEY>` — `merge/<KEY>.md` 를 남기고 코디네이터 터미널에 머지를 지시한다(닫혔으면 새로 띄운다). 코디네이터는 의존 저장소부터 머지하고, 못 하면 `blocked/` 에 사유를 남긴다.
- **세션 사이 메시지**: 드라이버·코디네이터·워커가 다른 세션의 터미널에 말을 넣을 때는 `qa-driver tell <PARENT> --to <handle> --type <종류> --subject … --body …` 를 쓴다. `orca terminal send` 로 입력하고 `log/tell.jsonl` 에 한 줄 남긴다(대시보드 상호작용 탭이 읽는다). 일반 터미널로 도는 코디네이터는 오케스트레이션 inbox 를 확인하지 않으므로 inbox 로 보내지 않는다. 드라이버의 승인·머지 지시와 기동 프롬프트도 이 경로로 간다.
- **정리**: `run` 회차가 `delivered/<KEY>.md` 의 PR 이 모두 머지됐거나, PR 없이 Jira 상태가 `QA_STATUS_DONE` 중 하나인 1회차 티켓(코드 수정 없이 끝난 티켓 — 2회차부터는 Jira 상태가 이전 회차 머지의 잔상일 수 있어 이번 회차 PR 머지로만 정리한다)을 — 열린 PR 이 있으면 Jira 상태와 무관하게 남긴다 — 찾아 그 터미널을 닫고 워크트리를 지운다(`gh` 필요, 드라이버가 만든 워크트리만). 지우기 전에 워크트리와 populate 된 서브모듈마다 HEAD 가 머지됐는지 확인한다(원격 베이스 브랜치에 들어 있거나, 머지된 PR 의 head — squash 대비 `delivered/` 기록과 `gh pr list --state merged`). 커밋하지 않은 변경이나 머지되지 않은 커밋이 있으면 남기고 로그에 `CLEANUP-SKIP` 을 한 번 적는다. 손으로는 `qa-driver cleanup <PARENT> <KEY>` — 같은 검사 뒤 서브모듈을 deinit 하고 워크트리를 지운다.
- **코디네이터 관리**: 멈춘 코디네이터는 `qa-driver restart <PARENT> <KEY>`(기존 브리프로 같은 워크트리에 다시 띄움, 살아 있는 터미널이 있으면 거부), 끝난 티켓의 터미널은 `qa-driver close <PARENT> <KEY>...`.
- **중지**: 스케줄을 끈다. 상태 디렉터리는 사후 확인을 위해 남긴다.
- **QA 종료**: `qa-driver retire <PARENT> --dry-run` 으로 확인한 뒤 `retire` — 에픽을 부르는 자동화를 끄고, 옛 워처의 공용 조정 워크트리(`QA_COORD_WORKTREE`)가 있으면 그 터미널을 닫고 지운다. 검토 전 티켓이 남아 있거나 조정 워크트리에 커밋하지 않은 파일이 있으면 멈춘다.

## 옛 워처에서 이관
공용 조정 워크트리에 코디네이터를 띄우던 옛 워처(상태 디렉터리 `claimed/ handoffs/ ready/ blocked/`)도 같은 상태 계약으로 읽힌다.
`qa-driver --home <옛 홈> adopt <PARENT> --check` 가 빠진 설정 키(와 `QA_REPO` 추정값)와 바꿀 자동화를 알려 주고,
`adopt --set KEY=값 ... [--automation <id>]` 가 키를 `config.env` 에 더한 뒤 그 자동화의 precheck·프롬프트를 `qa-driver run` 으로 바꾼다.
이미 claim 된 티켓은 그대로 두고, 이후 새로 열리거나 재오픈된 티켓부터 전용 워크트리로 간다. 옛 티켓의 코디네이터 재기동은 `QA_COORD_WORKTREE` 에서 한다.
