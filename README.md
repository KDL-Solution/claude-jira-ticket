# claude-jira-pr

JUNGLETFT JIRA 티켓 생성 / 수정 + (선택) 코드 수정 → PR 자동화를 **단일 `/jira-ticket` 슬래시 커맨드**로 묶은 Claude Code 플러그인.

`/jira-ticket` 한 번이면 KDL 티켓 컨벤션(Story/Subtask/Task 구조, 에픽 매핑, 담당자 ID, description 포맷) + 기존 티켓 수정 + `develop` 기반 PR 플로우 + `/security-review` 셀프 보안 리뷰까지 자동 적용된다.

---

## Quick Start

### 1. 사전 요구사항 (1회 setup)

#### 1-1. Atlassian MCP 연결

Claude Code 에서 JIRA 호출이 가능해야 한다.

- Claude Code → ⌘⇧P → `MCP: Add Server` → `Atlassian` 선택 → 브라우저에서 OAuth 로그인
- 연결 확인: Claude 에게 `"JUNGLETFT 프로젝트 조회해줘"` 라고 물어 → 정보 응답이 오면 OK

자세한 안내: https://docs.claude.com/en/docs/claude-code/mcp

#### 1-2. GitHub CLI 인증 (Code+PR 모드 사용 시)

```bash
gh auth status || gh auth login
```

#### 1-3. (선택) 프로젝트 `CLAUDE.md` 에 한 줄 추가

매번 가이드 읽기 명령을 안 줘도 자동 적용되게 하려면 작업 레포 루트의 `CLAUDE.md` 에 추가:

```markdown
# JIRA 티켓 / PR 자동화 규칙
JIRA 티켓 생성 / 수정 또는 코드+PR 자동화 요청 시 `/jira-ticket` 슬래시 커맨드의 규칙을 따른다.
```

### 2. 플러그인 설치

Claude Code 안에서:

```
/plugin marketplace add https://github.com/KDL-Solution/claude-jira-pr.git
/plugin install jira-ticket@claude-jira-pr
```

설치 후 Claude Code 재시작 (⌘⇧P → `Developer: Reload Window`).

### 3. 사용

`/jira-ticket` 다음에 자연어로 요청을 적는다. 프롬프트 내용으로 **3가지 모드**가 자동 분기된다.

| Mode | 분기 조건 | 예시 |
|---|---|---|
| **Create** | 새 티켓 묘사 | "Jude 담당으로 작업 티켓 만들어줘…" |
| **Update** | `JUNGLETFT-XXX` 키 + 변경 동사 (마감/담당/우선순위/상태/코멘트/링크) | "JUNGLETFT-656 마감 4/30, 담당 Jude" |
| **Code + PR** | 트리거 문구 포함: `"PR까지 만들어줘"` / `"코드 수정하고 PR 올려줘"` / `"브랜치까지 올려줘"` | "JUNGLETFT-834 코드 수정하고 PR 올려줘" |

---

## 케이스 예시

### 케이스 A — Task 티켓만 생성

```
/jira-ticket Jude 담당으로 작업 티켓 하나 만들어줘.
- 제목: [BE] 5월 1일부터 신규 가입 유저 Free 플랜 초기 크레딧 100 → 10 변경
- 마감: 4/30
- 우선순위: High
- 연관 이슈: JUNGLETFT-656 (Relates)
```

### 케이스 B — Task 티켓 + PR 까지 (Code+PR 모드)

```
/jira-ticket Jude 담당으로 작업 티켓 + PR 같이 부탁해.
- 제목: [BE] 신규 가입 유저 Free 플랜 초기 크레딧 100 → 10 변경
- 관련 파일: src/modules/credit/
- 마감: 4/30, Priority High
```

→ 티켓 생성 → `develop` 최신화 → 브랜치 → 구현 → ruff/mypy/pytest → commit → push → PR → 셀프 보안 리뷰 → JIRA sync 까지 한 번에.

### 케이스 C — 기존 티켓 기반 PR

```
/jira-ticket JUNGLETFT-834 기반으로 코드 수정하고 PR 올려줘.
```

### 케이스 D — Story + 하위 작업 한 번에

```
/jira-ticket [기능명] Story 만들어줘.
- 기능 요약: [무엇을, 왜]
- 에픽: 추가 기능 개발
- 우선순위: High
- 마감: 4/29
- 하위 작업: Design(Martin), BE(Jude), FE(Leon) 포함
- 정책:
  · [결정 항목 1]: [선택지]
  · [결정 항목 2]: [선택지]
```

→ Story 1개 + Subtask 3개 병렬 생성, 담당자 / 필드 자동 배정.

### 케이스 E — 기존 티켓 수정 (Update 모드)

```
/jira-ticket JUNGLETFT-656 마감일 4/30, Riley 대신 Jude로 담당 변경해줘.
```

```
/jira-ticket 728, 725, 716 마감 5/13으로 한꺼번에 바꿔줘.
```

### 케이스 F — 셀프 리뷰 스킵

PR 본문에 셀프 리뷰 코멘트가 따로 붙지 않게 하려면 프롬프트에 `"셀프 리뷰 스킵"` 포함:

```
/jira-ticket JUNGLETFT-834 기반으로 PR 올려줘. 셀프 리뷰 스킵.
```

---

## 자동화되는 KDL 컨벤션 요약

- **에픽 매핑**: 추가 기능 개발 = JUNGLETFT-251, 안정화/버그 = JUNGLETFT-258, 운영/VOC = JUNGLETFT-250
- **에픽 지정**: Story 필수, Subtask 없음, Task 기본 없음. "네가 골라" 시 내용 기반 추론 + 근거 표시
- **Description 포맷**: Story 는 `기능 의도 → 유저 플로우 → 기능 정책 → 구현 범위 → 인수 조건 → 참고`, Subtask 는 `구현 범위 → 참고할 기존 패턴 → 인수 조건 → 연관 이슈 → ※ 기능 의도/정책은 상위 Story 참조`
- **Subtask 참여자 필드**: `customfield_10132` 자동 입력
- **브랜치 네이밍**: `<type>/JUNGLETFT-<key>-<slug>` (Conventional Commits type)
- **커밋 메시지**: `<type>: <제목> (JUNGLETFT-XXX)` — Claude 관련 트레일러 자동 제외
- **PR 본문 템플릿**: Summary / Related / Test plan
- **JIRA 동기화**: PR 마크다운 링크 코멘트 + `수정 완료` (transition id `3`) 자동 전환
- **셀프 보안 리뷰**: `/security-review` 자동 실행 → PR 코멘트 게시
- **하위호환성**: Public API 변경 시 시그니처 보존 강제

---

## 자동으로 멈추는 조건 (stop conditions)

다음 상황에서는 user 확인 없이 진행하지 않는다 (Code+PR 모드):

- `git status` dirty
- 테스트 / lint 3회 이상 실패
- DB migration 자동 생성 필요
- Public API 시그니처 변경 (하위호환 가능성 검토)
- 변경 규모가 티켓 스펙 대비 현저히 큼
- 셀프 리뷰에서 **Critical / High** 발견 (Medium 이하는 보고만 하고 진행)

Update 모드에서는 destructive 변경(상태 `완료` 로 닫기, description 통째 덮어쓰기 등)에 한 번 확인을 요청한다.

---

## 머지는 수동

플러그인은 PR 생성까지만 한다. **`gh pr merge` 는 절대 실행하지 않는다** — 리뷰/머지는 작성자가 직접. 머지 후 원격 브랜치 삭제(`git push origin --delete <branch>`)도 user 가 처리.

---

## 트러블슈팅

| 증상 | 원인 / 해결 |
|---|---|
| `JIRA 프로젝트 조회 못 함` | Atlassian MCP 미연결 — 1-1 단계 재확인 |
| `gh: command not found` | GitHub CLI 미설치 → `brew install gh && gh auth login` |
| `pre-commit 실패` | `uv sync --all-groups` 후 다시 시도. 3회 실패 시 플러그인이 자동 중단 |
| `transition id 3 invalid` | JIRA 워크플로우 변경됨 — 관리자에게 `수정 완료` transition id 확인 후 `jira-ticket/commands/jira-ticket.md` Mode=Code+PR step 11 수정 |
| `epic 자동 추론이 자꾸 틀림` | 프롬프트에 epic 명시. "에픽 지정 규칙" 참고 |

---

## 라이선스

MIT
