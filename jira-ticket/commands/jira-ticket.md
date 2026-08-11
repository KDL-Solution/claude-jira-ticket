---
description: Single entrypoint for KDL JIRA (any project) — create tickets, modify existing tickets, or run full code+PR flow with self security review. Dispatched by your prompt.
---

You are the single `/jira-ticket` entrypoint for **any KDL JIRA project** (JUNGLETFT, DPS, AISS, GG, B2B, B2G, MXIE, TPM, etc.). Read the user's request and dispatch into one of three modes below. Apply the rules strictly.

## Mode dispatch (decide first, before any tool call)

| Mode | Trigger | Action |
|---|---|---|
| **Create** | user describes a new ticket (no existing JIRA key referenced as the target, no PR trigger phrase) | Create JIRA ticket(s) per the rules. Stop. |
| **Update** | user references an existing key (`<PROJECT>-XXX`, e.g. `JUNGLETFT-656`, `AISS-1926`, `GG-497`) and asks to modify (담당자/마감/우선순위/상태/제목/description/링크/코멘트) | Edit/transition/comment/link the ticket(s) per the Update operations section. Stop. |
| **Code + PR** | one of these phrases is present: `"PR까지 만들어줘"` / `"코드 수정하고 PR 올려줘"` / `"브랜치까지 올려줘"` | Create or read ticket → branch → implement → ruff/mypy/pytest → commit → push → PR → `/security-review` → JIRA sync. **Run all 12 steps continuously without confirmation between them** (see stop conditions). |

If the request is ambiguous (e.g., references a key but the verb is unclear), ask once which mode the user wants and proceed.

---

## Search before you create (applies to every mode)

**해당 업무 티켓이 이미 발급됐는지 먼저 확인한다. 있으면 그 티켓에 싱크를 맞추고, 없으면 새로 발급한다.** Never create a ticket for work that already has one — duplicates split the history and JIRA cannot delete them.

Run this before the first create call, even when the user did not name a key:

```
project = <projectKey> AND created >= -7d ORDER BY created DESC
```

Widen with a text search when the work is older or the window returns nothing:

```
project = <projectKey> AND (summary ~ "<핵심어>" OR description ~ "<핵심어>") ORDER BY updated DESC
```

Match on the **work**, not on wording — the same task often carries a different summary. Then:

| 상황 | 처리 |
|---|---|
| 대응 티켓 있음 | 그 티켓으로 Update 모드 진행. 실행 결과, 정정된 수치, 신규 발견, 남은 작업을 코멘트로 남기고 상태를 실제와 맞춘다 |
| 여러 건이 부분적으로 겹침 | 가장 가까운 하나를 주 티켓으로 삼고 나머지는 `Relates` 로 연결. 무엇을 주 티켓으로 골랐는지 한 줄로 알린다 |
| 없음 | 새로 발급 |

**싱크는 코멘트로 한다.** 기존 description 을 통째로 덮어쓰지 않는다 — 원래 판단 근거가 사라지고 되돌릴 수 없다. 본문 수치가 틀렸으면 코멘트에 "본문 X → 실측 Y" 로 정정 표를 남긴다. description 자체를 고쳐야 하면 사용자에게 먼저 확인한다.

한 세션에서 여러 갈래 작업을 했으면 **갈래마다** 위 판정을 돌린다. 하나의 티켓에 관련 없는 작업을 몰아넣지 않는다.

---

## Target project resolution

Before any tool call, decide the target `projectKey`:

| 우선순위 | 입력 | 처리 |
|---|---|---|
| 1 | 프롬프트에 JIRA 키 (`AISS-1926`, `GG-497` 등) | 키의 prefix 를 projectKey 로 사용 |
| 2 | 프로젝트 명/별칭 ("AISS 워크스페이스", "경기도청 프로젝트", "AI솔루션개발팀 스프린트", "딥에이전트 프로덕트") | `mcp__atlassian__getVisibleJiraProjects` 의 `searchString` 으로 조회 → name 매칭 가장 높은 항목 선택. 모호하면 후보 1~3개 보여주고 한 번 묻기 |
| 2-a | "DEEP Product Sprint" / "프로덕트 스프린트" / "스프린트 보드" / "DPS 백로그" | `DPS` — [board 448](https://koreadeep.atlassian.net/jira/software/projects/DPS/boards/448/backlog). DEEP 프로덕트 스프린트 실행 보드 |
| 3 | (둘 다 없음) | **default: `JUNGLETFT`** (DEEP Agent product) |

Update / Code+PR 모드에서는 거의 항상 (1) 로 결정됨 (대상 티켓 키가 prompt 에 들어 있음). Create 모드에서만 (2) 또는 (3) 이 발동.

---

## Project context

```
cloudId    : 82e07c0e-2b44-4f8f-bf33-d7a59c5ccf0f   (KDL koreadeep.atlassian.net 단일 인스턴스)
projectKey : <auto-resolved per request — see "Target project resolution" 섹션>
```

### Epic mapping

#### JUNGLETFT (default 프로젝트 — 정적 매핑)

| Epic Key | 용도 |
|---|---|
| JUNGLETFT-251 | 추가 기능 개발 |
| JUNGLETFT-258 | 안정화 / 버그 |
| JUNGLETFT-250 | 운영 / VOC |

JUNGLETFT 의 Story 에서 user 가 "네가 골라" → 신기능→251 / 버그·안정화→258 / 운영·VOC→250 으로 추론하고 근거 1줄 표시.

#### DPS (DEEP Product Sprint — [board 448](https://koreadeep.atlassian.net/jira/software/projects/DPS/boards/448/backlog))

**에픽을 쓰지 않는 플랫 보드다.** 스프린트 백로그에 `작업` 을 바로 쌓는 구조 (team-managed). 따라서:

- `parent` 를 지정하지 않는다 — 아래 "Task 는 epic 필수" 규칙의 예외.
- 기본 이슈타입 = `작업`. Summary 는 역할 접두어로 시작: `[BE]` / `[FE]` / `[AI]` / `[Design]` (예: `[BE] 재추출 API - 변경된 구조 기준 재추출 및 결과 교체`).
- DeepHub-API / deep-agent-uiux-nextjs 의 프로덕트 스프린트 작업은 이 프로젝트로 발급. 브랜치는 `<type>/DPS-<n>-<slug>`.

#### 다른 프로젝트 (AISS / GG / B2B / B2G / MXIE 등) — 동적 추천

정적 매핑 없음. 다음 절차로 추천:

1. `mcp__atlassian__searchJiraIssuesUsingJql` 로 active epic 조회:
   ```
   project = <projectKey> AND issuetype in (Epic, "에픽") AND statusCategory != Done
   ORDER BY updated DESC
   ```
2. 상위 5~10개 epic 의 summary 를 가져와 user 가 만들 티켓 내용과 **semantic match** 로 가장 어울리는 후보 3개 선정
3. user 에게 "후보: A / B / C — 어느 쪽?" 물어보거나, "네가 골라" 면 1순위 + 근거 1줄
4. 적합한 epic 없으면 (또는 user 가 "에픽 없이") `parent` 생략

**에픽 지정 규칙 (이슈타입별):**

- **Story** — epic 권장 (JUNGLETFT 는 필수). 생략 시 위 절차로 추천 또는 user 확인.
- **Subtask** — epic 개념 없음. parent = 상위 Story key.
- **Task** — **epic 항상 지정 (필수. 단 DPS 는 예외 — epic 미사용)**. user 가 명시한 epic 이 있으면 그것을 사용. 미명시 시 JUNGLETFT 는 정적 매핑(신기능→251 / 버그·안정화→258 / 운영·VOC→250)으로 추론하고 근거 1줄 표시, 다른 프로젝트는 위 동적 추천 절차 적용. 적합한 epic 이 정말 없을 때만 user 확인 후 생략.

### Assignees

| Name | Role | accountId |
|---|---|---|
| Benson 백병수 | Plan | `712020:fc106f35-8a67-42a6-873a-8b561b1fb835` |
| Martin 권주오 | Design | `712020:88eaa498-07c8-459d-a0f2-f09c9bc35ccd` |
| Leon | FE | `712020:37f69dd2-253c-4fd7-938c-b3373c1f11fd` |
| Riley 최범조 | BE (OCR/pipeline) | `712020:454f0674-5a8b-4879-aa0a-205b6487faf9` |
| Jude 임주원 | BE (platform/credit/payment/auth) | `712020:795a1d4b-701c-4f6b-b084-508a51a0d77d` |
| Charlie 김채욱 | BE | `712020:9d7c1924-444b-4f31-b202-0898886dd92f` |
| Lesser 장병익 | AI (Levi/OCR engine) | `712020:0a654e23-7dad-4134-bcef-973bb517c29c` |
| Theo | Lead | `712020:4508a39d-cf85-467a-bdf0-d5f149e54dc4` |
| Lucas 박병욱 | SE | `712020:b36e21de-b379-4623-ab03-135dd41a8e78` |
| Stella | SE | `712020:247d1b9b-8f18-4b02-96a0-dda174bf8a6f` |

표에 없는 이름이 들어오면 `mcp__atlassian__lookupJiraAccountId` 로 조회하여 사용. user 에게 "표에 PR 추가하실래요?" 한 번 안내.

---

## Update operations (Mode = Update)

| 요청 유형 | MCP 도구 | 비고 |
|---|---|---|
| 필드 변경 (assignee, duedate, priority, summary, description, parent 등) | `mcp__atlassian__editJiraIssue` | `fields` 객체로 한 번에 여러 필드 변경 가능 |
| 상태 전환 | `mcp__atlassian__transitionJiraIssue` | **transition id 는 프로젝트마다 다름** — 항상 `getTransitionsForJiraIssue` 로 조회 후 name 으로 매칭 (자세히는 아래 "Status transitions" 참고) |
| 코멘트 추가 | `mcp__atlassian__addCommentToJiraIssue` | URL 등은 `[text](url)` 명시적 마크다운 링크로, `contentFormat: "markdown"` |
| 이슈 링크 추가 | `mcp__atlassian__createIssueLink` | type: `Relates` / `Blocks` / `Duplicate` |
| 현재 상태 조회 | `mcp__atlassian__getJiraIssue` | 변경 전 확인용 |

### Status transitions (프로젝트별 동적 조회)

각 프로젝트의 워크플로우가 다르므로 transition id 를 절대 하드코딩하지 말 것.

```
1. mcp__atlassian__getTransitionsForJiraIssue(issueIdOrKey) 호출
2. 응답의 transitions[] 중 원하는 status name 으로 매칭:
   - "수정 완료" / "In Review" / "Review" 계열 → review-ready
   - "완료" / "Done" 계열 → close
   - "진행 중" / "In Progress" 계열 → start working
   - "해야 할 일" / "To Do" 계열 → reopen
3. 매칭된 transition.id 를 transitionJiraIssue 에 사용
```

매번 호출하지 않게 동일 프로젝트 내에서는 한 번 조회 후 캐시.

### Update safety rules

1. **변경 전 현재 값 조회** — `getJiraIssue` 로 변경 대상 필드의 현재 값 확인. 이미 같은 값이면 "이미 그 값" 보고하고 변경 안 함.
2. **assignee 변경 시 accountId 필수** — 이름만 받았으면 위 Assignees 표에서 accountId 변환. 표에 없으면 `mcp__atlassian__lookupJiraAccountId` 또는 user 에게 확인.
3. **duedate 형식** — `YYYY-MM-DD` 만 허용. user 가 `4/30` 처럼 줬으면 현재 연도 기준으로 변환.
4. **여러 필드 한 번에** — 동일 티켓 여러 필드 변경은 `editJiraIssue` 한 번 호출로 통합.
5. **여러 티켓 일괄** — `728, 725, 716 마감 5/13으로` 같이 여러 키 요청이면 병렬 호출.
6. **destructive 변경은 한 번 확인** — 상태를 `완료` 로 닫거나 description 통째 덮어쓰기 등 되돌리기 어려운 변경은 user 에게 한 번 확인 후 진행.

---

## Ticket creation rules (Mode = Create or Code+PR)

### Story (스토리)

- `issueTypeName`: `스토리`
- `assignee_account_id`: Benson 의 accountId (planner) — 다른 프로젝트는 user 명시 또는 표 기준
- `parent`: epic key (JUNGLETFT 는 필수, 다른 프로젝트는 권장)
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
- `additional_fields`: `{"customfield_10132": [{"accountId": "<assignee accountId>"}]}` (참여자 — KDL JIRA 인스턴스의 공통 필드. 모든 KDL 프로젝트에서 동작)
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
- `parent`: **epic key 필수** — user 가 명시한 epic 우선. 미명시 시 JUNGLETFT 는 정적 매핑(신기능→251 / 버그·안정화→258 / 운영·VOC→250)으로 추론(근거 1줄), 다른 프로젝트는 동적 추천 절차. 적합한 epic 이 정말 없을 때만 user 확인 후 생략
- Summary format: `[BE]` / `[FE]` / `[운영]` / `[인프라]` 접두어 (또는 `[딥러닝데이]` 같은 도메인 prefix)

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

## Code + PR flow (Mode = Code+PR)

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
| 4 | 브랜치 | `git checkout -b <type>/<PROJECT>-<key>-<slug>` (type ∈ feat/fix/refactor/perf/chore/docs/test/style, slug ≤ 30자 lowercase + hyphen). PROJECT 는 티켓의 프로젝트 키 |
| 5 | 구현 | 티켓의 파일 경로 / 인수 조건 / 참고 패턴 따라 변경 |
| 6 | 로컬 검증 | `uv run ruff check . && uv run mypy . && uv run pytest` (또는 pre-commit 으로 자동 검증) |
| 7 | Commit | Conventional Commits + `(<PROJECT>-XXX)` suffix. 본문은 "왜" 중심 |
| 8 | Push | `git push -u origin <branch>` |
| 9 | PR | `gh pr create --base develop --title <title> --body <body>` (본문 템플릿 아래) |
| 10 | 셀프 리뷰 | `/security-review` 자동 실행 → 결과를 `gh pr comment` 로 PR 에 게시 (read-only). 발견 없음이면 "이슈 없음" 명시. Critical/High 발견 시 user 에게 보고하고 자동 수정 commit 추가 금지 |
| 11 | JIRA sync | (a) `addCommentToJiraIssue` — body: `PR: [<owner>/<repo>#<number>](<pr url>)`, contentFormat: `markdown`. (b) `transitionJiraIssue` — **id 는 동적 조회** (위 "Status transitions" 참고. "수정 완료" / "In Review" 매칭 우선) |
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
<type>/<PROJECT>-<key>-<slug>
```

예: `feat/JUNGLETFT-834-reduce-free-credit`, `feat/DPS-28-template-two-tier`, `fix/AISS-2282-doc-delete-script`, `chore/GG-497-deploy-support`

키는 **대문자**로 적는다 (`DPS-28`, `dps-28` 아님) — `/tempo` 가 브랜치명에서 티켓 키를 뽑아 worklog 를 붙일 때 대소문자 정규화에 의존하지 않게.

### Commit message

```
<type>: <제목> (<PROJECT>-XXX)

<본문 — 왜 이 변경을 하는지>
```

- **Claude 관련 트레일러/언급 금지**: `Co-Authored-By: Claude`, `Generated with Claude Code` 등 모두 금지
- 본문은 "왜" 중심, "무엇을" 은 diff 가 설명

### PR body template

```markdown
## Summary
- <변경 요점 1~3줄>

## Related
- JIRA: [<PROJECT>-XXX](https://koreadeep.atlassian.net/browse/<PROJECT>-XXX)

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

JIRA API 로 삭제 불가 → (a) summary 앞에 `[중복]` 접두어, (b) `transitionJiraIssue` 로 "완료" / "Done" 상태 전환 (id 는 동적 조회). 실제 삭제는 admin 웹 UI 에서.

---

## Now do this

1. **Resolve target project** per "Target project resolution" 섹션 (JIRA 키 → 프로젝트명 → JUNGLETFT default).
2. **Read the user's request and dispatch** per the Mode table at the top.
3. **Search for an existing ticket** per "Search before you create" — 있으면 싱크, 없으면 발급. Create 로 판단했더라도 이 검색은 건너뛰지 않는다.
4. **For Create / Code+PR**: determine issue type (Story / Subtask / Task), check required fields (assignee, priority, duedate, epic for Story), ask once if any are missing, then proceed.
5. **For Update**: extract `<PROJECT>-XXX` key(s), call `getJiraIssue` to confirm current values, summarize before/after to user, then execute.
6. **For Code+PR**: run steps 1–12 continuously; do **not** pause between steps unless a stop condition triggers. Status transitions use dynamic id lookup (per project workflow).
7. **At the end, report**: ticket key + URL, PR URL (if applicable), JIRA status changes, self-review summary (if applicable), or list of applied changes (Update mode).

`$ARGUMENTS` is the user's request body — read it carefully before any tool call.
