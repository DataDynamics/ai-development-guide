---
id: 03-5
title: "디버깅 협업: 재현 → 가설 → 계측 → 수정"
module: 03-agentic-workflow
level: L2
duration: 2h
last-verified: 2026-10-07
tools:
  claude-code: 2.1.292
  codex: 0.160.1
---

# 03-5. 디버깅 협업: 재현 → 가설 → 계측 → 수정

## 🎯 학습 목표
- 버그 리포트를 **실패하는 테스트**로 먼저 재현하게 하고, 재현 전에는 수정을 시키지 않을 수 있습니다.
- 에이전트에게 가설을 여러 개 세우게 하고, 가설마다 근거와 확인 실험을 요구할 수 있습니다.
- 임시 계측(로그, 단언)과 비대화형 로그 분석(`claude -p`, `codex exec`), `git bisect`로 가설을 검증할 수 있습니다.
- 최소 수정 + 회귀 테스트 + 계측 제거까지 끝내고, 버그 리포트에서 수정 PR까지의 과정을 기록할 수 있습니다.

## 📋 사전 준비
- 선행 레슨: [03-2 에이전트와 TDD](./03-2-tdd-with-agents.md), [03-3 Git 전략](./03-3-git-strategy.md), [00-4 실패 패턴 카탈로그](../00-foundations/00-4-failure-patterns.md)
- 필요 도구/계정: Claude Code 2.1.292 이상, Codex 0.160.1 이상, `jq`(선택)
- 실습 저장소 상태: Project B(TaskFlow) B3 진행 중. 작업 목록 API에 마감일 필터(`dueBefore`)가 있습니다.

### 실습용 버그 심기
스터디라면 파트너가 심고, 혼자라면 직접 심은 뒤 에이전트에게는 diff를 보지 말라고 지시합니다. 아래는 예시입니다. 본인 코드에 맞게 비슷한 버그를 만듭니다.

1. `apps/api/src/modules/tasks/task.repository.ts`의 마감일 조건을 다음처럼 바꿉니다.
   - 쿼리 문자열 `dueBefore=2026-10-07`을 `new Date("2026-10-07")`(UTC 자정)로 바꾸고, `due_at < $1`로 비교합니다.
   - 결과: 프로젝트 시간대가 `Asia/Seoul`일 때 **10월 7일 당일 마감인 작업이 목록에서 빠집니다.**
2. 눈에 띄지 않는 메시지로 커밋합니다. `git commit -am "refactor(api): simplify task query builder"`
3. 버그 리포트를 `docs/bugs/BUG-007.md`로 만듭니다.

```markdown
# BUG-007: 마감일 필터에서 당일 마감 작업이 누락됩니다

- 보고자: QA / 환경: staging, 프로젝트 시간대 Asia/Seoul
- 증상: "오늘까지" 필터(dueBefore=오늘 날짜)를 켜면 오늘 마감인 작업이 보이지 않습니다.
- 재현: 오늘 18:00 마감 작업 생성 → GET /projects/:id/tasks?dueBefore=<오늘 날짜>
- 기대: 목록에 포함 / 실제: 누락
- 빈도: 항상 (단, 오전 9시 이전 마감 작업은 보인다는 제보 있음)
```

## 💡 개념

### 에이전트가 디버깅에서 실패하는 방식
에이전트에게 "이 버그 고쳐 주십시오"라고 하면 대개 곧바로 그럴듯한 코드를 고칩니다. 대규모 코드베이스에서는 이것이 위험합니다.

- **증상을 가립니다.** 원인을 모른 채 `if` 하나를 추가해 리포트의 재현 케이스만 통과시킵니다. 다른 시간대에서는 여전히 틀립니다.
- **첫 가설에 매달립니다.** 처음 떠올린 원인이 틀렸는데, 그 가설 안에서 수정 → 실패 → 수정을 반복합니다([00-4](../00-foundations/00-4-failure-patterns.md)의 무한 수정 루프).
- **흔적을 남깁니다.** 디버깅용 로그와 임시 코드가 수정 커밋에 섞여 들어갑니다.

해결책은 사람 개발자의 디버깅 규율을 **단계로 강제**하는 것입니다. 각 단계의 산출물이 다음 단계의 입력이 됩니다.

```mermaid
flowchart TD
    R[1. 재현<br/>실패하는 테스트] --> H[2. 가설<br/>3개 이상 + 근거 + 확인 실험]
    H --> I[3. 계측<br/>로그·단언·bisect]
    I --> J{가설 확정?}
    J -- 기각 --> H
    J -- 확정 --> F[4. 수정<br/>최소 변경]
    F --> V[5. 검증<br/>재현 테스트 + make verify<br/>+ 계측 제거]
    V -- 실패 --> H
    V -- 통과 --> P[버그 리포트 갱신 + PR]
```

| 단계 | 에이전트 권한 | 사람이 확인할 것 |
|---|---|---|
| 재현 | 테스트 파일만 쓰기 | 테스트가 **리포트의 증상 때문에** 실패합니까 |
| 가설 | 읽기 전용 | 가설이 서로 다른가, 근거가 코드 위치로 제시됐습니까 |
| 계측 | 임시 코드 쓰기 (표식 필수) | 실험 결과가 가설을 확정·기각합니까 |
| 수정 | 구현 쓰기 | 수정 범위가 근본 원인에 맞습니까 |
| 검증 | 테스트 실행 | 회귀 테스트, 전체 검증, 계측 흔적 0개 |

### 왜 대규모에서 중요한가
작은 프로젝트의 버그는 파일 한두 개 안에 있습니다. 큰 프로젝트에서는 원인이 증상과 먼 곳(공용 날짜 유틸, DB 드라이버 설정, 다른 팀의 미들웨어)에 있는 경우가 많습니다. 에이전트는 많은 파일을 빠르게 읽을 수 있다는 강점이 있지만, 방향을 잘못 잡으면 그만큼 빠르게 엉뚱한 곳을 고칩니다. **가설과 실험을 명시적으로 분리**하면 에이전트의 탐색 속도를 살리면서 방향은 사람이 통제할 수 있습니다.

## 👣 따라하기

### Step 1. 재현 — 실패하는 테스트부터 만듭니다
목적: 리포트의 증상을 자동으로 재현하는 테스트를 만듭니다. 재현되기 전에는 아무것도 고치지 않습니다.

**Claude Code 레시피**
```bash
claude -n "BUG-007"
```
```text
> docs/bugs/BUG-007.md 를 읽으십시오. git log 와 git diff 는 보지 마십시오(실습 규칙).
  이 버그를 재현하는 테스트를 apps/api/src/modules/tasks/due-date-filter.test.ts 에 작성하십시오.
  구현 코드는 수정하지 마십시오.
  - 프로젝트 시간대 Asia/Seoul, 오늘 18:00(KST) 마감 작업과 오늘 08:00(KST) 마감 작업을 만듭니다.
  - dueBefore=<오늘 날짜> 로 조회했을 때 두 작업이 모두 포함되어야 합니다.
  - 시각은 고정 clock 을 주입해서 실행 시점과 무관하게 만듭니다.
  테스트를 실행하고 어떤 단언이 어떤 값으로 실패하는지 보여 주십시오.
```

**Codex 레시피**
```bash
codex --sandbox workspace-write --ask-for-approval on-request
```
```text
> (Claude Code와 같은 프롬프트)
```

**기대 결과**
- 테스트가 실패합니다. 18:00 마감 작업이 누락되고, 08:00 마감 작업은 포함됩니다(리포트의 "오전 9시 이전은 보입니다"와 일치).
- 실패 이유가 DB 연결이나 import 오류가 아니라 **리포트의 증상**입니다. 그렇지 않으면 재현이 아닙니다.
- 재현 테스트를 커밋합니다(Codex는 사람이 실행). `test(api): reproduce BUG-007 due date filter` — 본문에 "intentionally failing"을 적습니다([03-3](./03-3-git-strategy.md) 규칙).

> 💡 재현이 어려운 버그(간헐적, 환경 의존)라면 이 단계에서 시간을 가장 많이 씁니다. "재현 조건을 좁히기 위한 실험 3가지를 제안하십시오"라고 시키고, 재현되기 전에는 Step 2로 넘어가지 않습니다.

### Step 2. 가설 — 서로 다른 가설 여러 개를 세웁니다
목적: 첫 가설에 매달리지 않도록, 근거와 확인 실험이 붙은 가설 목록을 받습니다.

**Claude Code 레시피** — 읽기 전용으로 전환합니다(`Shift+Tab`으로 plan 모드, 또는 `/plan`).
```text
> 재현 테스트의 실패를 설명할 수 있는 가설을 3개 이상 세우십시오. 아직 아무것도 수정하지 마십시오.
  가설마다 다음을 적으십시오:
  - 가설 (한 문장)
  - 근거: 관련 코드 위치(파일:함수)와 그 코드가 왜 의심스러운지
  - 반대 근거: 이 가설로 설명되지 않는 관찰
  - 확인 실험: 코드를 고치지 않고 가설을 확정·기각할 수 있는 가장 싼 실험
  리포트의 "오전 9시 이전 마감 작업은 보입니다"라는 관찰을 모든 가설이 설명하는지 확인하십시오.
```

**Codex 레시피** — Plan 모드로 전환합니다.
```text
> /plan
```
```text
> (Claude Code와 같은 프롬프트)
```

**기대 결과**: 예를 들어 다음과 같은 목록이 나옵니다.

| # | 가설 | 확인 실험 |
|---|---|---|
| H1 | `dueBefore` 날짜 문자열을 UTC 자정으로 해석해 KST 09:00 이전만 포함됩니다 | 쿼리에 전달되는 파라미터 값을 로그로 찍습니다 |
| H2 | 비교 연산자가 `<`라서 경계값이 빠집니다 | 마감 시각을 정확히 경계값으로 둔 테스트를 추가합니다 |
| H3 | DB 세션 시간대가 UTC라 `due_at`이 다르게 저장됩니다 | 테스트 DB에서 저장된 `due_at` 원값과 `SHOW TIME ZONE` 결과를 확인합니다 |

- 가설이 서로 다릅니다(같은 가설의 변형 세 개가 아닙니다).
- "9시"라는 관찰이 KST(UTC+9)와 연결된다는 점을 짚는 가설이 있습니다. 이 관찰을 설명하지 못하는 가설은 우선순위를 낮춥니다.
- 사람이 실험 순서를 정합니다. 보통 가장 싸고 가장 많은 가설을 가르는 실험부터 합니다.

### Step 3. 계측 — 실험으로 가설을 확정하거나 기각합니다
목적: 표식이 붙은 임시 계측으로 데이터를 모으고, 로그는 대화가 아니라 파일로 분석합니다.

**Claude Code 레시피** — 쓰기 가능한 모드로 돌아옵니다.
```text
> H1, H3 을 확인하는 계측을 추가하십시오. 규칙:
  - 모든 임시 코드 줄에 // DEBUG(BUG-007) 주석을 붙입니다.
  - 로그는 재현 테스트를 실행할 때만 출력되게 합니다.
  재현 테스트를 실행하고 전체 출력을 tmp/logs/bug-007.log 에 저장하십시오. 대화에는 마지막 10줄만 보여 주십시오.
```

**Codex 레시피**
```text
> (Claude Code와 같은 프롬프트)
```

로그가 길면 **새 컨텍스트에서 비대화형으로 분석**합니다. 본 세션의 컨텍스트를 아끼고, 앞선 가설에 끌려가지 않은 시각으로 로그를 읽게 하는 효과가 있습니다([03-4](./03-4-context-management.md)).

**Claude Code 레시피** (별도 터미널)
```bash
cat tmp/logs/bug-007.log | claude -p "이 로그는 BUG-007 재현 테스트 출력입니다. 쿼리에 전달된 dueBefore 파라미터 값, DB 시간대, 각 작업의 due_at 값을 표로 정리하고, 18:00 KST 작업이 왜 제외되는지 설명하십시오. 로그에 없는 내용은 추측하지 말고 '로그에 없음'이라고 쓰십시오."
```

**Codex 레시피** (별도 터미널)
```bash
codex exec "이 로그는 BUG-007 재현 테스트 출력입니다. 쿼리에 전달된 dueBefore 파라미터 값, DB 시간대, 각 작업의 due_at 값을 표로 정리하고, 18:00 KST 작업이 왜 제외되는지 설명하십시오. 로그에 없는 내용은 추측하지 말고 '로그에 없음'이라고 쓰십시오." < tmp/logs/bug-007.log
```
`codex exec`는 프롬프트와 stdin을 함께 주면 stdin을 `<stdin>` 블록으로 붙입니다. 기본 샌드박스가 read-only이므로 분석만 하고 파일은 바꾸지 않습니다.

**기대 결과**
- 로그에서 `dueBefore` 파라미터가 `2026-10-07T00:00:00.000Z`(= KST 09:00)로 전달되는 것이 보입니다. H1 확정.
- 저장된 `due_at`은 올바른 UTC 값입니다. H3 기각.
- 실험 결과를 버그 리포트에 "가설과 검증 결과" 섹션으로 적게 합니다.

#### (선택) 회귀 버그라면 `git bisect`
"예전에는 됐습니다"라는 버그라면 원인 커밋을 이분 탐색으로 찾습니다. 재현 테스트를 판정 스크립트로 씁니다. 도구와 무관한 Git 기능입니다.

```bash
git bisect start
git bisect bad HEAD
git bisect good <정상으로 알려진 커밋>
git bisect run pnpm --filter @taskflow/api test -- due-date-filter
git bisect reset
```
- 재현 테스트 파일이 과거 커밋에 없으면 `bisect run` 전에 저장소 밖(예: `/tmp`)으로 복사해 두고 판정 스크립트에서 매번 복사해 넣습니다.
- Claude Code: 허용 규칙에 따라 에이전트가 실행할 수 있습니다. 실행 후 `git bisect reset`까지 했는지 확인합니다.
- Codex: `git bisect`는 `.git`에 쓰므로 `workspace-write` 샌드박스에서 막힙니다. 명령은 사람이 실행하고, 결과(원인 커밋 해시)를 Codex에게 알려 줍니다.

**기대 결과**: `git bisect run`이 "refactor(api): simplify task query builder" 커밋을 첫 번째 bad 커밋으로 지목합니다.

### Step 4. 수정 — 근본 원인에 맞는 최소 수정
목적: 확정된 원인만 고치고, 재현 테스트를 회귀 테스트로 남깁니다.

**Claude Code 레시피**
```text
> H1 이 확정됐습니다. 근본 원인을 한 문장으로 설명하고, 최소 수정을 제안하십시오. 아직 수정하지 마십시오.
  조건: dueBefore 는 "프로젝트 시간대 기준 그 날짜의 끝까지"를 뜻합니다. 공용 날짜 유틸이 있으면 그것을 씁니다.
  재현 테스트는 수정하지 마십시오.
```
제안을 검토하고 승인합니다.
```text
> 제안대로 수정하십시오. 재현 테스트를 실행해 통과를 보여주고,
  경계 케이스(23:59:59 KST, 다음 날 00:00 KST, 시간대 UTC 프로젝트) 테스트를 추가하십시오.
```
수정 시도가 엉뚱한 방향으로 가면 `/rewind`로 되돌린 뒤 다시 지시합니다.

**Codex 레시피**
```text
> (Claude Code와 같은 두 프롬프트)
```
되돌릴 때는 `git restore <파일>`로 작업 트리를 되돌리거나, `/fork`로 분기해 다른 수정을 시도합니다.

**기대 결과**
- 근본 원인 설명이 "날짜 문자열을 UTC 자정으로 해석해 프로젝트 시간대 기준 하루의 끝까지 포함하지 못함"처럼 **증상이 아니라 메커니즘**을 말합니다.
- 수정이 쿼리 빌더의 날짜 해석 한 곳에 집중됩니다. 리포트 케이스만 피하는 특수 분기(`if (hour >= 9)`)가 있으면 증상 가리기입니다. 거절합니다.
- 재현 테스트와 경계 케이스 테스트가 모두 통과합니다.

### Step 5. 검증과 정리 — 흔적 없이 끝냅니다
목적: 계측을 제거하고, 전체 검증을 통과시키고, 다른 도구로 수정을 리뷰합니다.

**Claude Code 레시피 / Codex 레시피** (두 도구에 같은 프롬프트)
```text
> DEBUG(BUG-007) 표식이 붙은 코드를 모두 제거하십시오. 그다음 아래를 차례로 실행하고 결과를 보여 주십시오.
  1. grep -rn "DEBUG(BUG-007)" apps packages   (출력이 없어야 합니다)
  2. make verify   (마지막 20줄)
```

사람이 직접 확인합니다.
```bash
grep -rn "DEBUG(BUG-007)" apps packages || echo "OK: 계측 흔적 없음"
git diff --stat main...HEAD
make verify
```

수정한 도구와 **다른 도구**로 리뷰합니다.

**Claude Code로 수정했다면 → Codex로 리뷰**
```bash
codex review --base main
```

**Codex로 수정했다면 → Claude Code로 리뷰**
```text
> /code-review medium
```

**기대 결과**
- 계측 흔적이 0개입니다.
- `git diff --stat`에 재현·경계 테스트 파일과 수정 파일만 있습니다. `tmp/logs/`는 `.gitignore`에 있어 커밋되지 않습니다.
- 리뷰에서 "같은 패턴의 날짜 해석이 다른 곳에도 있습니다"라는 지적이 나오면 별도 Task로 분리합니다(이번 PR 범위 밖).

### Step 6. 버그 리포트를 완성하고 PR을 엽니다
목적: 버그 리포트 → 수정 PR 전 과정을 기록으로 남겨 팀의 지식으로 만듭니다.

**Claude Code 레시피 / Codex 레시피** (두 도구에 같은 프롬프트)
```text
> docs/bugs/BUG-007.md 에 다음 섹션을 추가하십시오:
  ## 가설과 검증 결과 (가설별 실험과 결과, 확정/기각)
  ## 근본 원인 (메커니즘 한 문단 + 원인 커밋 해시)
  ## 수정 (변경 파일과 이유)
  ## 회귀 방지 (추가한 테스트 이름)
  ## 재발 방지 제안 (규칙·린트·유틸 개선 후보)
```
그다음 [03-3](./03-3-git-strategy.md)의 PR Skill로 PR을 엽니다(Claude Code `/taskflow-pr`, Codex `$taskflow-pr`). PR 본문 "관련 Task"에 `BUG-007`을 적습니다.

**기대 결과**
- 버그 리포트 하나에 증상 → 재현 → 가설 → 실험 → 원인 → 수정 → 회귀 방지가 모두 있습니다.
- "재발 방지 제안" 중 하나(예: "날짜 필터는 `toProjectDayEnd()` 유틸을 씁니다")를 `AGENTS.md` 규칙 후보로 올립니다([06-4](../06-team-and-operations/06-4-knowledge-loop.md)).

## ✅ 체크포인트
- [ ] 수정 전에 리포트의 증상 때문에 실패하는 재현 테스트가 커밋되어 있습니다.
- [ ] 가설 3개 이상을 근거·반대 근거·확인 실험과 함께 받았고, 읽기 전용 상태에서 세웠습니다.
- [ ] 계측 결과(로그 파일)로 최소 한 가설을 확정하고 한 가설을 기각했습니다.
- [ ] 수정이 근본 원인에 맞고, 재현 테스트를 수정하지 않았습니다.
- [ ] `grep -rn "DEBUG(BUG-007)"` 결과가 비어 있고 `make verify`가 통과합니다.
- [ ] 수정하지 않은 도구로 리뷰를 실행했습니다.
- [ ] `docs/bugs/BUG-007.md`에 가설·원인·수정·회귀 방지가 기록됐습니다.

## 🏠 숙제
| # | 유형 | 난이도 | 과제 | 완료 조건 |
|---|---|---|---|---|
| HW1 | 🔁 Reproduce | ★ | 따라하기 Step 1~5를 재현해 심어 둔 버그를 찾습니다 (파트너가 심은 버그면 더 좋습니다) | 재현 테스트 실패 로그, 가설 표(3개 이상), 계측 로그 분석 결과, 수정 후 `make verify` 통과 로그, 계측 흔적 grep 결과 |
| HW2 | 🛠 Apply | ★★ | 실제 버그 1건(본인 프로젝트 또는 TaskFlow 개발 중 발견한 것)을 버그 리포트 → 수정 PR까지 이 레슨의 절차로 처리합니다 | `docs/bugs/BUG-xxx.md`(이 레슨 6개 섹션 모두), 재현 테스트가 먼저 커밋된 이력, 다른 도구의 리뷰 결과, PR 링크와 세션 기록(사람 개입 지점 포함) |
| HW3 | 🚀 Challenge | ★★★ | 같은 버그를 Claude Code와 Codex로 각각 독립적으로 디버깅하고 비교합니다. 재현 → 가설 → 계측 → 수정 절차를 디버깅 Skill(`.claude/skills/debug/`, `.agents/skills/debug/`)로 패키징합니다 | 도구별 가설 목록과 첫 가설 적중 여부, 원인 확정까지 걸린 턴 수·사람 개입 횟수 비교표, Skill 파일 2개, Skill을 써서 다른 버그 1건을 처리한 기록 |

제출: `hw/03-5` 브랜치, `submissions/03-5.md` (템플릿: [docs/design/homework-template.md](../../docs/design/homework-template.md))

## ⚠️ 흔한 실수
- **재현 없이 고쳤더니 다른 케이스에서 다시 터집니다** → "버그 고쳐 주십시오"로 바로 수정을 시켰습니다 → 재현 테스트를 먼저 커밋하고, 재현되기 전에는 수정 프롬프트를 보내지 않습니다.
- **에이전트가 첫 가설만 붙잡고 수정을 반복합니다** → 가설을 하나만 요구했거나 가설과 수정을 한 번에 시켰습니다 → 가설 3개 이상 + 반대 근거 + 확인 실험을 읽기 전용 상태에서 받고, 같은 수정이 두 번 실패하면 Step 2로 돌아갑니다.
- **수정 PR에 디버그 로그가 섞였습니다** → 계측 코드에 표식이 없었습니다 → `DEBUG(<버그ID>)` 표식을 규칙으로 두고, 검증 단계에서 `grep`으로 0개를 확인합니다. 반복되면 검증 커맨드에 이 검사를 넣습니다.
- **로그 분석에서 에이전트가 없는 값을 지어냅니다** → 긴 로그를 본 세션에 붙여 넣고 "원인을 찾으십시오"라고만 했습니다 → 로그는 파일로 저장해 `claude -p`·`codex exec`로 새 컨텍스트에서 분석하고, "로그에 없으면 '로그에 없음'이라고 쓰십시오"를 명시합니다.
- **Codex에서 `git bisect`가 실패합니다** → `workspace-write` 샌드박스가 `.git`을 보호합니다 → `bisect`는 사람이 실행하고 결과만 Codex에 전달합니다.

## 🔗 참고 자료
- Claude Code: [Common workflows](https://code.claude.com/docs/en/common-workflows), [Headless](https://code.claude.com/docs/en/headless) (`claude -p`, stdin), [Permission modes](https://code.claude.com/docs/en/permission-modes), [Commands](https://code.claude.com/docs/en/commands) (`/rewind`, `/code-review`)
- Codex: [Non-interactive mode](https://learn.chatgpt.com/docs/non-interactive-mode) (`codex exec`, stdin), [Approvals & security](https://learn.chatgpt.com/docs/agent-approvals-security) (`.git` 보호), [Code review](https://learn.chatgpt.com/docs/code-review), [Slash commands](https://learn.chatgpt.com/docs/developer-commands?surface=cli)
- 외부: [git bisect 문서](https://git-scm.com/docs/git-bisect)
- 이 저장소: [도구 레퍼런스 §2, §5, §10](../../docs/reference/tool-reference.md)
- 관련 레슨: [00-4 실패 패턴 카탈로그](../00-foundations/00-4-failure-patterns.md), [03-2 에이전트와 TDD](./03-2-tdd-with-agents.md), [03-3 Git 전략](./03-3-git-strategy.md), [03-4 컨텍스트 관리](./03-4-context-management.md), [04-2 교차 리뷰](../04-quality-and-verification/04-2-cross-review.md)
- 프로젝트: [Project B — TaskFlow](../../projects/B-development-project/README.md)
