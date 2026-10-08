---
id: 02-1
title: 에이전트와 함께 PRD 쓰기 (인터뷰 방식)
module: 02-spec-driven-development
level: L2
duration: 2h
last-verified: 2026-10-07
tools:
  claude-code: 2.1.292
  codex: 0.160.1
---

# 02-1. 에이전트와 함께 PRD 쓰기 (인터뷰 방식)

## 🎯 학습 목표
- 에이전트가 질문하고 사람이 답하는 **인터뷰 방식**으로 PRD 초안을 완성할 수 있습니다.
- 인터뷰 세션을 읽기 전용(계획) 모드로 열어 에이전트가 코드를 건드리지 않게 통제할 수 있습니다.
- 기능 요구사항에 `FR-001` 형식의 ID를 붙이고, 각 요구사항을 검증 가능한 문장으로 바꿀 수 있습니다.
- 한 도구가 쓴 PRD를 다른 도구로 검토시켜 모호한 표현과 누락을 3건 이상 찾아낼 수 있습니다.

## 📋 사전 준비
- 선행 레슨: [01-1 프로젝트 메모리](../01-environment-setup/01-1-project-memory.md), [01-2 권한·샌드박스](../01-environment-setup/01-2-permissions-and-sandbox.md)
- 필요 도구/계정: Claude Code 2.1.292 이상, Codex 0.160.1 이상, 각 도구 로그인 완료
- 실습 저장소 상태: Project B(TaskFlow)의 **B0 완료 상태**. 즉 pnpm + Turborepo 모노레포 뼈대, 루트 `AGENTS.md`와 `@AGENTS.md`를 import하는 `CLAUDE.md`가 있습니다.
- 템플릿: [templates/prd.md](../../templates/prd.md)를 `docs/prd/`로 복사해 둡니다.

```bash
cd taskflow
mkdir -p docs/prd
cp <가이드 저장소>/templates/prd.md docs/prd/taskflow-prd.md
```

## 💡 개념

### 왜 PRD부터입니까
설계서 원칙 1(Spec First)이 말하듯 대규모 개발의 실패 원인 1순위는 **모호한 지시**입니다. 작은 스크립트라면 "할 일 앱 만들어 주십시오"로도 그럴듯한 결과가 나옵니다. 하지만 3~5만 라인짜리 TaskFlow를 수십 개 세션과 여러 에이전트가 나눠 만들면, 각 세션이 빈칸을 **제각각 추측**해서 채웁니다. 한 세션은 "작업은 한 명에게만 배정됩니다"라고 가정하고, 다른 세션은 "여러 명에게 배정됩니다"라고 가정합니다. 이 불일치는 통합 단계에서야 드러나고, 그때는 이미 수천 라인을 고쳐야 합니다.

PRD는 이 추측을 **한 번, 사람이 확정**하는 문서입니다. 이후의 ADR([02-2](./02-2-architecture-and-adr.md)), Task 명세([02-3](./02-3-task-breakdown.md)), API 계약([02-4](./02-4-interface-first.md))은 모두 PRD의 요구사항 ID를 참조합니다.

### 왜 "인터뷰 방식"인가
에이전트에게 "PRD 써 주십시오"라고만 하면 일반론으로 가득 찬 그럴듯한 문서가 나옵니다. 문서는 길지만 **결정은 하나도 들어 있지 않습니다**. 인터뷰 방식은 역할을 뒤집습니다.

| 방식 | 누가 결정합니까 | 결과 |
|---|---|---|
| "PRD 써 주십시오" | 에이전트가 추측 | 일반론, 숨은 가정이 많습니다 |
| 인터뷰 방식 | 사람이 답하고 에이전트는 질문·정리 | 결정이 명시되고, 모르는 것은 미결 사항으로 남습니다 |

에이전트는 질문을 빠짐없이 던지는 일(범위, 예외, 권한, 규모, 실패 시나리오)에 강합니다. 사람은 비즈니스 맥락과 우선순위를 압니다. 둘을 합치면 혼자 쓸 때보다 빠르고, 에이전트 혼자 쓸 때보다 정확합니다.

### 전체 흐름

```mermaid
flowchart LR
    A[사람: 한 문단 브리프] --> B[에이전트: 질문 1개씩]
    B --> C[사람: 답변 / 모름]
    C --> B
    C -->|충분히 모임| D[에이전트: templates/prd.md에 맞춰 초안]
    D --> E[다른 도구: 모호성·누락 검토]
    E --> F[사람: 미결 사항 확정]
    F --> G[docs/prd/taskflow-prd.md 커밋]
```

### 좋은 PRD 요구사항의 조건
- **ID가 있습니다**: `FR-012`처럼 고유 ID를 붙여야 Task 명세와 테스트가 참조할 수 있습니다.
- **검증 가능합니다**: "빠르게 응답합니다"가 아니라 "작업 목록 API는 p95 300ms 이내로 응답합니다".
- **범위가 닫혀 있습니다**: 비목표(non-goal)를 명시해서 에이전트가 기능을 부풀리지 못하게 합니다.
- **모르는 것은 모른다고 씁니다**: 추측으로 채우지 않고 `7. 미결 사항`에 남깁니다.

## 👣 따라하기

### Step 1. 한 문단 브리프 작성 (도구 무관)
목적: 에이전트가 질문을 시작할 출발점을 사람이 직접 씁니다. 이 단계는 도구와 무관합니다.

`docs/prd/brief.md`에 5~10줄로 씁니다. 완벽할 필요는 없습니다. 오히려 빈칸이 있어야 인터뷰가 의미 있습니다.

```markdown
# TaskFlow 브리프
- 10~50명 규모 개발팀이 쓰는 업무 관리 서비스.
- 프로젝트 안에 작업(Task)을 만들고, 담당자·상태·마감일을 관리합니다.
- 웹 UI, REST API, 알림 워커, CLI를 제공합니다.
- 기존에 스프레드시트와 메신저로 관리하던 팀이 대상입니다.
- 첫 릴리스는 3개월 안에 사내 2개 팀에 배포합니다.
```

**기대 결과**: `docs/prd/brief.md`가 생깁니다. 아직 커밋하지 않아도 됩니다.

### Step 2. 읽기 전용 모드로 인터뷰 세션 열기
목적: 인터뷰 중에 에이전트가 코드나 설정 파일을 수정하지 못하게 막고, 질문에만 집중시킵니다.

**Claude Code 레시피**

Plan 모드는 읽기와 계획만 허용합니다. 시작 플래그로 지정하거나, 세션 안에서 `/plan`으로 전환합니다.

```bash
claude --permission-mode plan
```

```text
> docs/prd/brief.md 를 읽고 TaskFlow PRD를 위한 인터뷰를 진행해 주십시오.
  규칙:
  1. 한 번에 질문은 하나만 합니다. 답을 듣고 나서 다음 질문을 합니다.
  2. 질문 순서: 사용자와 역할 → 핵심 시나리오 → 범위 밖 → 권한/보안 → 규모/성능 → 실패·예외 상황 → 성공 지표.
  3. 제가 "모름" 또는 "나중에"라고 답하면 추측하지 말고 미결 사항 목록에 적어둡니다.
  4. 질문할 때 선택지가 있으면 2~4개 보기를 함께 제시합니다.
  5. 7개 영역을 모두 다루면 "인터뷰 종료"라고 말하고 지금까지의 답을 표로 요약합니다.
  파일은 아직 만들지 마십시오.
```

**Codex 레시피**

Codex는 읽기 전용 샌드박스로 시작한 뒤, 세션 안에서 `/plan`을 써서 계획 중심으로 대화합니다.

```bash
codex --sandbox read-only --ask-for-approval on-request
```

```text
> /plan
> docs/prd/brief.md 를 읽고 TaskFlow PRD를 위한 인터뷰를 진행해 주십시오.
  (위 Claude Code 레시피와 같은 규칙 1~5를 붙여 넣습니다)
```

> 두 도구의 "계획 모드"는 1:1 대응이 아닙니다. Claude Code의 `plan`은 권한 모드이고, Codex는 `read-only` 샌드박스와 `/plan` 명령의 조합입니다. 이 실습에서는 둘 다 "파일을 바꾸지 않습니다"라는 목적만 달성하면 됩니다.

**기대 결과**: 에이전트가 첫 질문 하나만 던집니다. 예를 들어 다음과 같습니다.

```text
질문 1/7 (사용자와 역할): TaskFlow의 사용자 역할은 어떻게 나뉩니까?
  (a) 관리자 / 멤버 2단계
  (b) 조직 관리자 / 프로젝트 관리자 / 멤버 / 게스트 4단계
  (c) 역할 없이 모든 사용자 동일
```

질문을 한꺼번에 10개씩 쏟아내면 규칙 1을 다시 상기시킵니다.

### Step 3. 인터뷰에 답하기 — "모름"을 두려워하지 않습니다
목적: 결정할 수 있는 것은 결정하고, 결정할 수 없는 것은 미결 사항으로 넘깁니다.

인터뷰는 15~25개 질문 정도에서 끝나는 것이 적당합니다. 답변 요령은 다음과 같습니다.

| 상황 | 답변 예 |
|---|---|
| 결정할 수 있습니다 | "(b) 4단계. 단, 게스트는 읽기 전용입니다." |
| 숫자가 필요합니다 | "동시 사용자 500명, 프로젝트당 작업 최대 1만 개를 가정합니다." |
| 지금은 모릅니다 | "모름. 보안팀 확인 필요." → 미결 사항으로 넘어갑니다 |
| 범위를 줄이고 싶습니다 | "첫 릴리스에서는 하지 않습니다. 비목표로 적어 주십시오." |

인터뷰 중 실제로 나올 법한 TaskFlow 질문과 답의 예:

```text
Q: 작업(Task)을 여러 명에게 동시에 배정할 수 있습니까?
A: 아닙니다. 담당자는 0~1명입니다. 대신 "참여자(watcher)"를 여러 명 둘 수 있습니다.

Q: 알림은 어떤 채널로 보냅니까? (a) 이메일 (b) 웹 알림 (c) Slack (d) 모두
A: 첫 릴리스는 (a)와 (b)만. Slack은 비목표.

Q: 삭제된 작업은 복구할 수 있어야 합니까?
A: 모름. 감사(audit) 요구사항을 확인해야 합니다.
```

7개 영역을 모두 다루면 에이전트가 요약표를 보여줍니다.

**기대 결과**: 에이전트가 "인터뷰 종료"와 함께 질문·답변 요약표, 그리고 미결 사항 목록(보통 3~6개)을 출력합니다. 이 요약을 확인하고 잘못 이해한 부분이 있으면 바로 고칩니다.

### Step 4. 템플릿에 맞춰 PRD 초안 작성
목적: 인터뷰 결과를 `templates/prd.md`의 7개 섹션 구조에 맞춰 문서로 만듭니다. 이제 파일을 써야 하므로 쓰기 권한이 있는 모드로 바꿉니다.

**Claude Code 레시피**

`Shift+Tab`으로 권한 모드를 순환해서 Plan 모드를 빠져나오거나 `/permissions`로 바꿉니다. 같은 세션을 이어가야 인터뷰 내용이 컨텍스트에 남아 있습니다.

```text
> 인터뷰 결과로 docs/prd/taskflow-prd.md 를 채워 주십시오.
  - 파일에 있는 7개 섹션 제목과 순서를 그대로 유지합니다.
  - 기능 요구사항은 FR-001부터 번호를 붙이고, 각 항목은 "누가 / 무엇을 / 어떤 조건에서" 형태의 한 문장 + 수용 기준 1~3개로 씁니다.
  - 비기능 요구사항은 NFR-001부터 번호를 붙이고 반드시 수치를 포함합니다. 수치를 모르면 미결 사항으로 보냅니다.
  - 제가 답하지 않은 내용은 절대 추가하지 않습니다. 추가하고 싶으면 7. 미결 사항에 "제안:"으로 적습니다.
  - 마지막에 각 FR이 인터뷰의 어느 답변에서 나왔는지 대응표를 출력해 주십시오(파일에는 넣지 않습니다).
```

**Codex 레시피**

`/permissions`로 쓰기를 허용하는 프리셋(Auto: `workspace-write` + `on-request`)으로 바꿉니다. 인터뷰 세션을 이미 종료했다면 `codex resume --last`로 그 세션을 다시 열어 이어갑니다.

```bash
codex resume --last
```

```text
> /permissions
  (workspace-write 프리셋 선택)
> 인터뷰 결과로 docs/prd/taskflow-prd.md 를 채워 주십시오.
  (위 Claude Code 레시피와 같은 지시를 붙여 넣습니다)
```

**기대 결과**: `docs/prd/taskflow-prd.md`가 7개 섹션으로 채워집니다. 기능 요구사항 부분은 대략 다음과 같은 모양이어야 합니다.

```markdown
## 4. 기능 요구사항

### FR-001 작업 생성
프로젝트 멤버 이상의 역할을 가진 사용자는 프로젝트 안에 작업을 생성할 수 있습니다.
- 수용 기준: 제목(1~200자)은 필수, 설명·마감일·담당자는 선택입니다.
- 수용 기준: 게스트가 생성을 시도하면 403을 받습니다.

### FR-002 담당자 배정
작업의 담당자는 0명 또는 1명이며, 참여자(watcher)는 여러 명 지정할 수 있습니다.
- 수용 기준: 담당자를 바꾸면 이전 담당자와 새 담당자 모두에게 알림이 생성됩니다(FR-010 참조).
```

`git diff --stat`으로 `docs/prd/` 외의 파일이 바뀌지 않았는지 확인합니다.

```bash
git diff --stat
```

### Step 5. 다른 도구로 교차 검토
목적: 쓴 도구와 다른 도구가 PRD를 읽고 모호한 표현, 검증 불가능한 문장, 누락된 예외를 찾게 합니다(설계서 원칙 5: Two Models, Cross-Check). 검토는 파일을 바꾸지 않으므로 비대화형으로 실행합니다.

Claude Code로 PRD를 썼다면 Codex로, Codex로 썼다면 Claude Code로 검토합니다. 두 레시피를 모두 보입니다.

**Codex 레시피 (Claude가 쓴 PRD를 검토)**

`codex exec`의 기본 샌드박스는 read-only이므로 검토에 그대로 쓰기 좋습니다. 결과는 `-o`로 파일에 저장합니다.

```bash
codex exec -o docs/prd/review-codex.md \
  "docs/prd/taskflow-prd.md 를 검토해 주십시오. 파일은 수정하지 마십시오.
   다음 네 가지 범주로 지적 사항을 표로 정리합니다:
   1) 검증 불가능한 표현(예: 빠르게, 적절히, 쉽게)
   2) 요구사항끼리 모순
   3) 누락된 예외·오류 시나리오
   4) 비목표와 충돌하는 요구사항
   각 지적에는 관련 ID(FR-xxx/NFR-xxx), 문제 문장 인용, 수정 제안을 붙입니다."
```

**Claude Code 레시피 (Codex가 쓴 PRD를 검토)**

```bash
claude -p "docs/prd/taskflow-prd.md 를 검토해 주십시오. 파일은 수정하지 마십시오.
  (위와 같은 네 가지 범주와 형식)" \
  --permission-mode plan --output-format json | jq -r '.result' > docs/prd/review-claude.md
```

**기대 결과**: 검토 결과 파일에 지적 사항이 표로 정리됩니다. 보통 다음과 같은 지적이 나옵니다.

| 범주 | ID | 지적 | 제안 |
|---|---|---|---|
| 검증 불가 | NFR-002 | "알림은 신속하게 전달됩니다" | "작업 변경 후 60초 이내에 웹 알림 생성" |
| 누락 | FR-002 | 담당자가 프로젝트에서 제외되면? | 담당자 자동 해제 + 알림 |
| 모순 | FR-007 / 비목표 | 비목표에 "외부 연동 없음"인데 FR-007이 이메일 발송 | 이메일은 예외로 명시 |

### Step 6. 미결 사항 확정과 커밋
목적: 검토 결과를 사람이 판단해서 반영하고, 남은 미결 사항에는 담당자와 기한을 붙입니다.

지적 사항마다 **수용 / 기각 / 미결로 이동** 중 하나를 사람이 결정합니다. 반영 작업은 다시 에이전트에게 맡깁니다.

**Claude Code 레시피**

```text
> docs/prd/review-codex.md 의 지적 중 1, 2, 4번은 수용, 3번은 기각(이유: 첫 릴리스 범위 밖), 5번은 미결로 옮깁니다.
  수용한 항목을 docs/prd/taskflow-prd.md 에 반영하고, 7. 미결 사항의 각 항목에 "담당: / 기한:" 칸을 추가해 주십시오.
  반영 후 FR/NFR 개수와 미결 사항 개수를 알려 주십시오.
```

**Codex 레시피**

```text
> (같은 지시를 Codex 대화형 세션에 입력합니다. workspace-write 상태여야 합니다)
```

마지막으로 사람이 PRD를 처음부터 끝까지 한 번 읽고 커밋합니다. 검토 파일은 이력으로 남기고 싶으면 함께 커밋하고, 아니면 지웁니다.

```bash
git add docs/prd/
git commit -m "docs(prd): TaskFlow PRD v1 (interview-based)"
```

**기대 결과**: `docs/prd/taskflow-prd.md`에 FR 15개 이상, NFR 5개 이상, 담당·기한이 붙은 미결 사항이 있습니다. 이 문서가 B1 마일스톤의 첫 산출물이 됩니다.

## ✅ 체크포인트
- [ ] 인터뷰 세션을 Claude `plan` 모드 또는 Codex `read-only` + `/plan`으로 진행했고, 그동안 파일이 바뀌지 않았습니다.
- [ ] 에이전트가 한 번에 질문을 하나씩 던졌고, "모름"으로 답한 항목이 미결 사항으로 넘어갔습니다.
- [ ] `docs/prd/taskflow-prd.md`가 `templates/prd.md`의 7개 섹션 구조를 유지합니다.
- [ ] 모든 기능 요구사항에 `FR-NNN` ID와 수용 기준이 있습니다.
- [ ] 모든 비기능 요구사항에 수치가 있습니다(없으면 미결 사항에 있습니다).
- [ ] 다른 도구의 검토 결과를 받아 수용/기각/미결을 사람이 결정했습니다.

## 🏠 숙제
| # | 유형 | 난이도 | 과제 | 완료 조건 |
|---|---|---|---|---|
| HW1 | 🔁 Reproduce | ★ | 따라하기 Step 1~4를 본인 환경에서 재현해 TaskFlow PRD 초안을 만듭니다 | `docs/prd/taskflow-prd.md`가 7개 섹션을 갖추고 FR 10개 이상이 있습니다. 인터뷰 질문·답변 요약표를 `submissions/02-1.md`에 첨부합니다 |
| HW2 | 🛠 Apply | ★★ | Project B의 PRD를 완성합니다. Step 5의 교차 검토를 거쳐 v1으로 확정합니다 | FR 15개 이상(모두 수용 기준 포함), NFR 5개 이상(모두 수치 포함), 미결 사항마다 담당·기한이 있습니다. 교차 검토 지적 사항과 각각의 처리 결과(수용/기각/미결)를 표로 제출합니다 |
| HW3 | 🚀 Challenge | ★★★ | 인터뷰 진행 규칙을 재사용 가능한 Skill로 패키징합니다. Claude는 `.claude/skills/prd-interview/SKILL.md`, Codex는 `.agents/skills/prd-interview/SKILL.md`에 같은 내용을 둡니다 | 두 도구에서 각각 Skill을 호출(Claude `/prd-interview`, Codex `$prd-interview`)해 다른 주제(예: TaskFlow의 "반복 작업" 기능)의 미니 PRD를 만든 로그를 제출합니다. 두 결과의 질문 순서·누락 차이를 비교한 짧은 표를 포함합니다 |

제출: `hw/02-1` 브랜치, `submissions/02-1.md` (템플릿: docs/design/homework-template.md)

## ⚠️ 흔한 실수
- **질문이 한꺼번에 쏟아집니다** → 원인: "인터뷰해 주십시오"만 쓰고 진행 규칙을 주지 않았습니다. 에이전트는 기본적으로 한 턴에 최대한 많이 도우려 합니다 → 대응: "한 번에 질문 하나" 규칙을 프롬프트 맨 앞에 두고, 어기면 바로 지적합니다. 반복해서 쓰는 규칙이면 HW3처럼 Skill로 만듭니다.
- **PRD에 제가 말하지 않은 기능이 들어 있습니다** → 원인: 에이전트가 빈칸을 "일반적인 업무 관리 서비스" 지식으로 채웠습니다 → 대응: "답하지 않은 내용은 추가하지 않습니다, 제안은 미결 사항에 씁니다"를 명시하고, Step 4처럼 FR ↔ 인터뷰 답변 대응표를 요구해 출처 없는 FR을 찾아냅니다.
- **인터뷰 도중 에이전트가 코드를 만들기 시작합니다** → 원인: 쓰기 권한이 있는 모드(Claude 기본 `auto`, Codex 기본 Auto 프리셋)로 시작했습니다 → 대응: Step 2처럼 Claude `--permission-mode plan`, Codex `--sandbox read-only`로 시작합니다. 이미 바뀐 파일은 `git status`로 확인하고 되돌립니다.
- **`codex exec`로 PRD를 반영하라고 했는데 아무것도 바뀌지 않습니다** → 원인: `codex exec`의 기본 샌드박스는 read-only입니다 → 대응: 쓰기가 필요한 비대화형 실행에는 `codex exec --sandbox workspace-write "..."`를 명시합니다(`--full-auto`는 0.160.1에서 거부됩니다).
- **NFR이 "빠르게", "안정적으로"로 끝납니다** → 원인: 사람도 수치를 정하지 않았습니다 → 대응: 수치를 정할 수 없으면 억지로 쓰지 말고 미결 사항으로 보낸 뒤 담당자를 지정합니다. 수치 없는 NFR은 테스트로 검증할 수 없습니다.

## 🔗 참고 자료
- [Claude Code permission modes (plan 모드)](https://code.claude.com/docs/en/permission-modes)
- [Claude Code headless (`-p`, `--output-format`)](https://code.claude.com/docs/en/headless)
- [Codex approvals & security (sandbox, approval)](https://learn.chatgpt.com/docs/agent-approvals-security)
- [Codex non-interactive mode (`codex exec`, `-o`)](https://learn.chatgpt.com/docs/non-interactive-mode)
- [Codex slash commands (`/plan`, `/permissions`)](https://learn.chatgpt.com/docs/developer-commands?surface=cli)
- [Claude Code skills](https://code.claude.com/docs/en/skills), [Codex skills](https://learn.chatgpt.com/docs/build-skills) — HW3
- 템플릿: [templates/prd.md](../../templates/prd.md)
- 이 가이드의 관련 레슨: [00-3 실행 모드](../00-foundations/00-3-execution-modes.md), [01-4 확장 기능](../01-environment-setup/01-4-extensions.md), 다음 레슨 [02-2 아키텍처 설계와 ADR](./02-2-architecture-and-adr.md)
- 프로젝트: [Project B — TaskFlow](../../projects/B-development-project/README.md) (B1 마일스톤)
