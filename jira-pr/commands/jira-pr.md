---
description: Create JIRA tickets (Story/Subtask/Task) on JUNGLETFT, and optionally implement code + open a PR + run self security review
---

You are creating JIRA tickets on the **JUNGLETFT** project — and optionally implementing the code + opening a PR — applying the rules below strictly. Two flows are supported and the user's prompt selects between them.

---

## Project context

```
cloudId    : 82e07c0e-2b44-4f8f-bf33-d7a59c5ccf0f
projectKey : JUNGLETFT
```

### Epic mapping

| Epic Key | 용도 |
|---|---|
| JUNGLETFT-251 | 추가 기능 개발 |
| JUNGLETFT-258 | 안정화 / 버그 |
| JUNGLETFT-250 | 운영 / VOC |

**에픽 지정 규칙 (반자동):**

- **Story** — epic 필수. 생략 시 user 에게 묻고, "네가 골라" 하면 신기능→251 / 버그·안정화→258 / 운영·VOC→250 으로 추론하고 근거 1줄 표시.
- **Subtask** — epic 개념 없음. parent = 상위 Story key.
- **Task** — epic 없이 단독 생성이 기본. user 가 명시하면 해당 epic 의 parent 로 붙임.

### Assignees

| Name | Role | accountId |
|---|---|---|
| Benson 백병수 | Plan | `712020:fc106f35-8a67-42a6-873a-8b561b1fb835` |
| Martin 권주오 | Design | `712020:88eaa498-07c8-459d-a0f2-f09c9bc35ccd` |
| Leon | FE | `712020:37f69dd2-253c-4fd7-938c-b3373c1f11fd` |
| Riley 최범조 | BE (OCR/pipeline) | `712020:454f0674-5a8b-4879-aa0a-205b6487faf9` |
| Jude 임주원 | BE (platform/credit/payment/auth) | `712020:795a1d4b-701c-4f6b-b084-508a51a0d77d` |
| Lesser 장병익 | AI (Levi/OCR engine) | `712020:0a654e23-7dad-4134-bcef-973bb517c29c` |

---

## Two flows — selected by trigger phrase in user prompt

| Flow | Trigger phrases | Behavior |
|---|---|---|
| **Ticket only** | (none) | Create JIRA ticket(s) and stop |
| **Ticket + PR** | "PR까지 만들어줘" / "코드 수정하고 PR 올려줘" / "브랜치까지 올려줘" | Create ticket → branch → implement → ruff/mypy/pytest → commit → push → PR → `/security-review` → JIRA sync. **Run all steps continuously without asking for confirmation between them** (see stop conditions below). |

---

## Ticket creation rules

### Story (스토리)

- `issueTypeName`: `스토리`
- `assignee_account_id`: Benson 의 accountId (planner)
- `parent`: epic key
- `priority`: `Highest` / `High` / `Medium` / `Low` / `Lowest`
- `duedate`: priority High+ → 다음 배포 날짜, Medium- → 그 다음 배포 날짜 (user 가 명시하면 그대로)
- Summary format: `[카테고리] 기능명 (부연 설명)` — 예: `[Credit] 워크스페이스 추출 크레딧 페이지 수 비례 차감 (5장당 1크레딧)`
- Description sections **in this order**:
  1. `### 기능 의도` — 1~3 문장
  2. `### 유저 플로우` — UI 흐름 있는 기능에만 (선택)
  3. `### 기능 정책` — 테이블 권장
  4. `### 구현 범위` — `#### Design` / `#### AI` / `#### BE` / `#### FE` 서브섹션. BE/AI 섹션은 파일 경로:라인번호 명시
  5. `### 인수 조건` — 검증 가능한 체크박스 ("개선" 같은 모호한 표현 금지, "N크레딧 차감됨" 처럼 테스트 가능하게)
  6. `### 참고` — 연관 코드 경로, 연관 JIRA 키

### Subtask (하위 작업)

- `issueTypeName`: `하위 작업`
- `parent`: 상위 Story key (필수)
- `assignee_account_id`: 역할별 담당자
- `priority`, `duedate`: 상위 Story 와 동일
- `additional_fields`: `{"customfield_10132": [{"accountId": "<assignee accountId>"}]}` (참여자 — 필수)
- Summary format: `[역할] 기능명 — 구체적 작업` — 예: `[BE] 크레딧 차감 시점 변경 + 5장당 1크레딧 계산 로직`
- 역할 접두어: `[Design]` / `[AI]` / `[BE]` / `[FE]`
- Description sections:
  ```markdown
  ## 구현 범위
  - 구체적 항목 (파일 경로:라인번호 권장)

  ## 참고할 기존 패턴
  - 재사용할 컴포넌트/함수/패턴 경로

  ## 인수 조건
  - [ ] 완료 기준

  ## 연관 이슈
  Blocks / Relates to [키]

  > ※ 기능 의도/정책은 상위 Story 참조
  ```
- **Subtask 마지막 줄에 `> ※ 기능 의도/정책은 상위 Story 참조` 항상 포함** — Story 내용 중복 금지

### Task (작업)

- `issueTypeName`: `작업`
- `assignee_account_id`: 실제 담당자
- `priority`, `duedate`: 명시
- `parent`: 기본 없음. user 가 epic 명시하면 해당 epic 의 key
- Summary format: `[BE]` / `[FE]` / `[운영]` / `[인프라]` 접두어

### Description 작성 원칙

- JIRA 사이드바에 있는 정보(담당자/마감일/우선순위) 를 description 에 반복 금지
- 하위 작업 목록 description 에 직접 나열 금지 (JIRA 가 자동 표시)
- BE/AI 섹션은 관련 파일 경로:라인번호 포함

### Issue links

| 관계 | type | 사용 시점 |
|---|---|---|
| 연관 | `Relates` | FE-BE 동일 기능 |
| 선행 필요 | `Blocks` | A 완료 후 B |
| 중복 | `Duplicate` | 동일 내용 |

### Priority guidelines

| Priority | 기준 |
|---|---|
| Highest | 서비스 장애 / 결제 오류 / 보안 / 법적 의무 |
| High | 다음 배포 필수 / 사용자 흐름 차단 버그 |
| Medium | 개선 / UX |
| Low | Nice-to-have / 장기 |
| Lowest | 미착수 / 언젠가 |

---

## Code + PR flow (when triggered)

### When NOT to use auto-flow — create ticket only and ask user

- 새 모듈 / 아키텍처 설계
- 복잡한 DB migration (큰 스키마 변경)
- 디자인 / 기획 조율 미완 기능
- 영향 범위가 불확실한 변경

### Steps — run continuously without confirmation

| # | 단계 | 동작 |
|---|---|---|
| 1 | Pre-check | `git status` → dirty 면 중단하고 user 확인 |
| 2 | Base 최신화 | `git fetch origin && git checkout develop && git pull origin develop` |
| 3 | 티켓 확보 | 기존 티켓이면 description 읽기, 신규면 위 규칙대로 생성 |
| 4 | 브랜치 | `git checkout -b <type>/JUNGLETFT-<key>-<slug>` (type ∈ feat/fix/refactor/perf/chore/docs/test/style, slug ≤ 30자 lowercase + hyphen) |
| 5 | 구현 | 티켓의 파일 경로 / 인수 조건 / 참고 패턴 따라 변경 |
| 6 | 로컬 검증 | `uv run ruff check . && uv run mypy . && uv run pytest` (또는 pre-commit 으로 자동 검증) |
| 7 | Commit | Conventional Commits + `(JUNGLETFT-XXX)` suffix. 본문은 "왜" 중심 |
| 8 | Push | `git push -u origin <branch>` |
| 9 | PR | `gh pr create --base develop --title <title> --body <body>` (본문 템플릿 아래) |
| 10 | 셀프 리뷰 | `/security-review` 자동 실행 → 결과를 `gh pr comment` 로 PR 에 게시 (read-only). 발견 없음이면 "이슈 없음" 명시. Critical/High 발견 시 user 에게 보고하고 자동 수정 commit 추가 금지 |
| 11 | JIRA sync | (a) `addCommentToJiraIssue` — body: `PR: [<owner>/<repo>#<number>](<pr url>)`, contentFormat: `markdown`. (b) `transitionJiraIssue` — id `"3"` (`수정 완료`) |
| 12 | 보고 | PR URL · JIRA 상태 변경 · 셀프 리뷰 요약 한 번에 알림 |

### Stop conditions (require user confirmation, halt the flow)

- `git status` dirty
- 테스트 / lint 3회 이상 실패
- DB migration 자동 생성 필요
- Public API 시그니처 변경 (하위호환 여부 먼저 논의)
- 변경 규모가 티켓 스펙 대비 현저히 큼 (예상 파일 수 ×2 초과)
- 셀프 리뷰에서 **Critical / High** 등급 발견 (Medium 이하는 보고만 하고 진행)

### Branch naming

```
<type>/JUNGLETFT-<key>-<slug>
```

예: `feat/JUNGLETFT-834-reduce-free-credit`, `fix/JUNGLETFT-721-levi-sse-retry`

### Commit message

```
<type>: <제목> (JUNGLETFT-XXX)

<본문 — 왜 이 변경을 하는지>
```

- **Claude 관련 트레일러/언급 금지**: `Co-Authored-By: Claude`, `Generated with Claude Code` 등 모두 금지
- 본문은 "왜" 중심, "무엇을" 은 diff 가 설명

### PR body template

```markdown
## Summary
- <변경 요점 1~3줄>

## Related
- JIRA: [JUNGLETFT-XXX](https://koreadeep.atlassian.net/browse/JUNGLETFT-XXX)

## Test plan
- [ ] <검증 항목 1>
- [ ] <검증 항목 2>
```

- Base: `develop`
- 리뷰어: `.github/CODEOWNERS` 자동 지정 — `--reviewer` 플래그 불필요
- PR 본문에도 Claude 관련 표기 금지

### Self-review (step 10) skip

- Prompt 에 `"셀프 리뷰 스킵"` / `"보안 리뷰 빼"` 포함 시 step 10 생략
- `/simplify` 는 코드 수정 동반 가능 → 자동 실행 안 함. user 가 명시하면 실행

### Backwards compatibility

- Public API: 기존 경로 / 필드 제거·rename 금지
- 신규 기능: optional 필드 또는 새 엔드포인트
- 내부 리팩터링은 자유, 외부 계약(노출 경로·스키마)은 고정

### Never merge

Claude 는 PR 생성까지만. **`gh pr merge` 절대 실행 금지** — review/merge 는 작성자가 직접. 머지 후 원격 브랜치 삭제는 user 가 처리.

### 중복 티켓 처리

JIRA API 로 삭제 불가 → (a) summary 앞에 `[중복]` 접두어, (b) `transitionJiraIssue` id `"41"` (완료) 로 닫기. 실제 삭제는 admin 웹 UI 에서.

---

## Now do this

1. **Read the user's request.**
2. **Determine flow:** is one of the trigger phrases present? → Ticket+PR. Otherwise → Ticket only.
3. **Determine issue type:** Story / Subtask / Task (and sub-rules above).
4. **Required fields** — assignee, priority, duedate, epic (Story only). If anything missing for the chosen type, ask the user once and proceed.
5. **Apply the rules above** strictly. For Ticket+PR flow, run steps 1–12 continuously; do **not** pause between steps unless a stop condition triggers.
6. **At the end**, report: ticket key + URL, PR URL (if applicable), JIRA status transition, and self-review summary.

`$ARGUMENTS` is the user's request body — read it carefully before any tool call.
