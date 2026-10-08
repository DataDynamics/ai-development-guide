---
id: 02-3
title: "작업 분해: Epic → Story → Task"
module: 02-spec-driven-development
level: L2
duration: 2.5h
last-verified: 2026-10-07
tools:
  claude-code: 2.1.292
  codex: 0.160.1
---

# 02-3. 작업 분해: Epic → Story → Task

## 🎯 학습 목표
- PRD의 기능 요구사항을 Epic → Story → Task의 3단계로 분해할 수 있습니다.
- "에이전트가 한 세션에 끝낼 수 있는 크기"를 측정 가능한 기준으로 정의하고 Task에 적용할 수 있습니다.
- `templates/task-spec.md` 형식의 Task 명세 20개를 에이전트로 생성하고, 의존성과 요구사항 추적성을 기계적으로 검증할 수 있습니다.
- 다른 도구로 Task 크기를 검토하고, 너무 큰 Task를 다시 쪼갤 수 있습니다.

## 📋 사전 준비
- 선행 레슨: [02-1 PRD](./02-1-prd-with-agents.md), [02-2 아키텍처와 ADR](./02-2-architecture-and-adr.md)
- 필요 도구/계정: Claude Code 2.1.292 이상, Codex 0.160.1 이상, `jq`
- 실습 저장소 상태: TaskFlow 저장소에 `docs/prd/taskflow-prd.md`, `docs/architecture/drivers.md`, `docs/adr/0001~0002-*.md`가 커밋되어 있습니다. API 계약([02-4](./02-4-interface-first.md))은 아직 없어도 됩니다.
- 템플릿: [templates/task-spec.md](../../templates/task-spec.md)

```bash
cd taskflow
mkdir -p docs/backlog/tasks
cp <가이드 저장소>/templates/task-spec.md docs/backlog/tasks/TASK-000-template.md
```

## 💡 개념

### 왜 "한 세션 크기"인가
에이전트에게 "알림 기능 만들어 주십시오"를 맡기면 처음 30분은 순조롭습니다. 그러다 컨텍스트 윈도우가 차면서 앞에서 읽은 ADR을 잊고, 테스트가 깨진 채로 "완료했습니다"라고 보고합니다(설계서 원칙 2: Context is a Budget). 반대로 "함수 하나 이름 바꿔 주십시오" 수준으로 잘게 쪼개면 사람이 지시하고 확인하는 비용이 더 커집니다.

적정 크기는 그 사이, **에이전트가 한 세션 안에서 탐색 → 계획 → 구현 → 검증까지 마치고, 사람이 diff를 10~15분 안에 리뷰할 수 있는 크기**입니다. 이 크기로 쪼개 두면 다음이 가능해집니다.

- **병렬화**: 의존성 없는 Task를 worktree나 클라우드 세션에 동시에 맡깁니다([05-1](../05-scaling-up/05-1-parallel-agents.md)).
- **재시도 비용 감소**: 실패한 Task 하나만 버리고 다시 돌립니다.
- **검증 가능성**: 완료 조건이 명령 하나(`pnpm test -- <path>`)로 확인됩니다(원칙 3: Verify, Don't Trust).

### 3단계 분해

```text
PRD (FR-001 … FR-030)
 │
 ├─ Epic: 사람에게 의미 있는 기능 묶음 (수 주)        예) E3 작업(Task) 관리
 │   │
 │   ├─ Story: 사용자 관점의 수직 슬라이스 (1~3일)     예) S3.2 담당자를 배정합니다
 │   │   │      수용 기준: Given/When/Then
 │   │   │
 │   │   ├─ Task: 에이전트 1세션 (30~90분)              예) TASK-014 PATCH /tasks/{id} 담당자 필드 + 서비스 로직
 │   │   ├─ Task                                           TASK-015 담당자 변경 이벤트 → 알림 큐 발행
 │   │   └─ Task                                           TASK-016 웹: 담당자 선택 드롭다운
```

- **Epic**은 사람이 로드맵을 읽는 단위입니다. 에이전트에게 직접 맡기지 않습니다.
- **Story**는 사용자가 체감하는 가치 단위이고, 수용 기준이 PRD의 FR에서 나옵니다.
- **Task**는 에이전트가 실행하는 단위이고, 완료 조건이 **명령으로 검증 가능**해야 합니다.

### 이 가이드의 "한 세션 크기" 기준
아래 기준은 출발점입니다. 팀마다 회고([06-4](../06-team-and-operations/06-4-knowledge-loop.md))로 조정합니다.

| 기준 | 권장 값 | 넘으면 |
|---|---|---|
| 주로 바꾸는 패키지 | 1개 (+ 계약·공유 타입) | 패키지별로 쪼갭니다 |
| 예상 변경 파일 | 10개 이하 | 레이어(라우트/서비스/저장소)로 쪼갭니다 |
| 예상 diff | 400줄 이하(테스트 포함) | 기능을 절반으로 쪼갭니다 |
| 완료 조건 | 검증 명령 1~3개로 확인 | 완료 조건이 모호하면 Story로 되돌립니다 |
| 읽어야 할 문서 | PRD 섹션 + ADR + 계약, 합쳐 3개 이내 | 문서를 요약해 Task에 붙입니다 |
| 의존 작업 | 미완료 의존 0개(시작 시점) | 순서를 정해 대기열에 넣습니다 |

### 추적성(traceability)
모든 Task는 `관련 요구사항: FR-xxx`를 가집니다. 이 연결이 있어야 "FR-012는 어느 Task가 구현합니까", "어느 FR이 아직 Task가 없습니까"를 기계적으로 확인할 수 있습니다. Step 4에서 이것을 `jq`로 검사합니다.

## 👣 따라하기

### Step 1. PRD에서 Epic 도출
목적: FR을 4~7개의 Epic으로 묶어 백로그의 큰 틀을 잡습니다.

**Claude Code 레시피**

```bash
claude --permission-mode plan
```

```text
> docs/prd/taskflow-prd.md 와 docs/adr/ 를 읽고 Epic을 도출해 주십시오.
  - Epic은 4~7개. 각 Epic에 ID(E1…), 한 줄 목표, 포함하는 FR ID 목록을 붙입니다.
  - 모든 FR은 정확히 하나의 Epic에 속해야 합니다. 어디에도 넣기 어려운 FR은 따로 표시합니다.
  - Epic 사이 의존 관계(예: E2 인증이 E3 작업 관리보다 먼저)를 한 줄씩 적습니다.
  - 마지막에 "FR → Epic" 대응표를 출력해서 누락이 없는지 보여 주십시오.
```

**Codex 레시피**

```bash
codex --sandbox read-only
```

```text
> /plan
> (위와 같은 지시)
```

결과를 확인하고 쓰기 모드로 바꾼 뒤(Claude `Shift+Tab`, Codex `/permissions`) `docs/backlog/epics.md`로 저장하게 합니다.

**기대 결과**: `docs/backlog/epics.md`에 다음과 같은 표가 생깁니다.

| Epic | 목표 | FR | 선행 |
|---|---|---|---|
| E1 | 인증과 사용자 | FR-020~023 | – |
| E2 | 조직·프로젝트·멤버십 | FR-030~034 | E1 |
| E3 | 작업(Task) 관리 | FR-001~009 | E2 |
| E4 | 알림 | FR-010~013 | E3, ADR-0002 |
| E5 | CLI | FR-040~042 | E3 |

FR → Epic 대응표에서 빠진 FR이 없어야 합니다.

### Step 2. Epic을 Story로 분해
목적: 각 Epic을 사용자 관점의 Story로 쪼개고, Given/When/Then 수용 기준을 붙입니다.

**Claude Code 레시피** (같은 세션, 쓰기 가능 모드)

```text
> docs/backlog/epics.md 의 각 Epic을 Story로 분해해서 docs/backlog/stories.md 에 써 주십시오.
  - Story ID는 S<Epic번호>.<순번> (예: S3.2)
  - 형식: "<역할>로서 <목적>을 위해 <기능>을 합니다" + 수용 기준 2~4개(Given/When/Then)
  - 수용 기준은 PRD의 FR 수용 기준에서 가져옵니다. PRD에 없는 기준을 만들지 말고, 필요하면 "PRD 보완 필요"로 표시합니다.
  - Story 하나는 API·웹·워커 중 필요한 레이어를 모두 포함하는 수직 슬라이스로 만듭니다.
```

**Codex 레시피**

```text
> (Codex 대화형 세션, workspace-write 상태에서 같은 지시)
```

**기대 결과**: `docs/backlog/stories.md`에 Story 12~20개가 생깁니다. 예:

```markdown
### S3.2 담당자를 배정합니다 (FR-002)
프로젝트 멤버로서 업무 책임을 분명히 하기 위해 작업에 담당자를 지정합니다.
- Given 멤버 권한의 사용자, When 작업의 담당자를 다른 멤버로 바꾸면, Then 작업 상세에 새 담당자가 표시됩니다.
- Given 게스트 사용자, When 담당자 변경을 시도하면, Then 403을 받습니다.
- Given 담당자가 바뀌면, Then 이전·새 담당자에게 알림 작업이 생성됩니다(FR-010).
```

"PRD 보완 필요" 표시가 나오면 PRD를 먼저 고칩니다. 백로그가 PRD를 앞서 나가면 안 됩니다.

### Step 3. Task 명세 20개 생성
목적: Story를 "한 세션 크기" 기준에 맞는 Task로 쪼개고, 템플릿 형식의 개별 파일로 만듭니다. 파일을 여러 개 만드는 작업이므로 비대화형으로 실행해도 좋습니다.

먼저 기준을 담은 지시 파일을 만듭니다. 두 도구가 같은 지시를 쓰게 하기 위해서입니다.

```bash
cat > docs/backlog/breakdown-prompt.md <<'EOF'
docs/backlog/stories.md 의 Story 중 E1~E3에 속한 것을 Task로 분해해 정확히 20개의 Task 명세 파일을 만드십시오.

형식:
- docs/backlog/tasks/TASK-000-template.md 의 구조와 섹션 제목을 그대로 따릅니다.
- 파일명: docs/backlog/tasks/TASK-NNN-<kebab-slug>.md (NNN은 001부터)
- "관련 요구사항"에는 FR ID와 Story ID를 함께 적습니다. 예: FR-002, S3.2
- "의존 작업"에는 TASK ID 또는 "없음"
- "예상 변경 범위"에는 실제 모노레포 경로를 씁니다 (apps/api, apps/web, apps/worker, packages/contracts 등)

크기 기준(모두 만족해야 합니다):
- 주로 바꾸는 패키지 1개 (+ packages/contracts 허용)
- 예상 변경 파일 10개 이하, 예상 diff 400줄 이하(테스트 포함)
- 완료 조건은 실행 가능한 검증 명령 1~3개 + 관찰 가능한 결과. "잘 동작합니다" 같은 문장 금지
- 범위 밖 섹션에 인접 Task가 할 일을 명시해 경계를 긋습니다

금지:
- ADR(docs/adr/)과 충돌하는 구현 방식을 쓰지 않습니다
- 존재하지 않는 스크립트 이름을 지어내지 않습니다. 루트 package.json의 scripts를 확인하고, 없으면 "TODO: 스크립트 추가 필요"라고 씁니다

마지막에 생성한 파일 목록과 각 Task의 (패키지, 예상 파일 수, 의존 작업)을 표로 출력하십시오.
EOF
```

**Claude Code 레시피**

파일 생성만 허용하면 되므로 `acceptEdits` 모드로 비대화형 실행합니다.

```bash
claude -p "$(cat docs/backlog/breakdown-prompt.md)" --permission-mode acceptEdits
```

**Codex 레시피**

`codex exec`는 기본이 read-only이므로 쓰기 샌드박스를 명시합니다.

```bash
codex exec --sandbox workspace-write "$(cat docs/backlog/breakdown-prompt.md)"
```

**기대 결과**: `docs/backlog/tasks/`에 `TASK-001-*.md` ~ `TASK-020-*.md`가 생깁니다. 하나를 열어 보면 다음과 같은 모양입니다.

```markdown
# TASK-014: 작업 담당자 변경 API

> 에이전트가 한 세션 안에 끝낼 수 있는 크기로 작성합니다.

- 관련 요구사항: FR-002, S3.2
- 의존 작업: TASK-011 (작업 조회 API), TASK-006 (프로젝트 멤버십)
- 예상 변경 범위: apps/api/src/modules/tasks/, packages/contracts/openapi.yaml

## 목표
PATCH /projects/{projectId}/tasks/{taskId} 에서 assigneeId 변경을 지원합니다.

## 상세 요구사항
- assigneeId는 null 또는 해당 프로젝트 멤버의 사용자 ID만 허용합니다.
- 게스트 역할은 403, 멤버가 아닌 사용자 ID는 422를 반환합니다.

## 완료 조건 (검증 가능한 형태)
- [ ] `pnpm --filter api test -- tasks/assign` 통과 (정상, 403, 422 케이스 포함)
- [ ] `pnpm --filter api typecheck` 통과
- [ ] 계약 테스트에서 PATCH 응답이 openapi.yaml 스키마와 일치

## 범위 밖
- 담당자 변경 알림 발행 (TASK-015)
- 웹 UI (TASK-016)

## 참고 (API 계약, 관련 코드 위치)
- packages/contracts/openapi.yaml 의 updateTask 오퍼레이션
- ADR-0002 (알림은 이 Task에서 다루지 않음)
```

### Step 4. 백로그 인덱스 생성과 기계적 검증
목적: 20개 Task를 구조화된 JSON 인덱스로 뽑아 **의존성 순환, 존재하지 않는 의존, Task 없는 FR**을 사람 눈이 아니라 `jq`로 찾습니다.

인덱스 스키마를 먼저 만듭니다. 두 도구에서 같은 스키마를 쓰려면 루트를 객체로 두고, 모든 속성을 `required`에 넣고, `additionalProperties: false`를 붙인 엄격한 형태로 쓰는 것이 안전합니다. (Codex `--output-schema`가 느슨한 스키마를 어디까지 허용하는지는 TODO(verify))

```bash
cat > docs/backlog/index.schema.json <<'EOF'
{
  "type": "object",
  "additionalProperties": false,
  "required": ["tasks"],
  "properties": {
    "tasks": {
      "type": "array",
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": ["id", "title", "requirements", "dependsOn", "package", "estimatedFiles"],
        "properties": {
          "id":             { "type": "string" },
          "title":          { "type": "string" },
          "requirements":   { "type": "array", "items": { "type": "string" } },
          "dependsOn":      { "type": "array", "items": { "type": "string" } },
          "package":        { "type": "string" },
          "estimatedFiles": { "type": "integer" }
        }
      }
    }
  }
}
EOF
```

**Claude Code 레시피**

`--json-schema`를 주면 결과 JSON의 `structured_output`에 스키마를 따른 값이 들어옵니다.

```bash
claude -p "docs/backlog/tasks/TASK-0*.md 파일(TASK-000 제외)을 모두 읽고 인덱스를 만드십시오. requirements에는 FR ID만 넣습니다." \
  --permission-mode plan --output-format json \
  --json-schema "$(cat docs/backlog/index.schema.json)" \
  | jq '.structured_output' > docs/backlog/index.json
```

**Codex 레시피**

```bash
codex exec --output-schema docs/backlog/index.schema.json -o docs/backlog/index.json \
  "docs/backlog/tasks/TASK-0*.md 파일(TASK-000 제외)을 모두 읽고 인덱스를 만드십시오. requirements에는 FR ID만 넣습니다."
```

이제 검증은 도구와 무관합니다. 에이전트의 판단이 아니라 `jq`가 판정합니다.

```bash
# 1) Task 개수
jq '.tasks | length' docs/backlog/index.json

# 2) 존재하지 않는 Task를 의존하는 경우
jq -r '(.tasks | map(.id)) as $ids
       | .tasks[] | .id as $t | .dependsOn[]
       | select(. as $d | $ids | index($d) | not)
       | "\($t) -> \(.) (없는 Task)"' docs/backlog/index.json

# 3) 크기 기준 위반 (예상 파일 10개 초과)
jq -r '.tasks[] | select(.estimatedFiles > 10) | "\(.id) \(.estimatedFiles) files"' docs/backlog/index.json

# 4) Task가 하나도 없는 FR (PRD의 FR 목록과 비교)
grep -oE 'FR-[0-9]{3}' docs/prd/taskflow-prd.md | sort -u > /tmp/fr-prd.txt
jq -r '.tasks[].requirements[]' docs/backlog/index.json | sort -u > /tmp/fr-tasks.txt
comm -23 /tmp/fr-prd.txt /tmp/fr-tasks.txt
```

의존성 순환은 `jq`로 검사하기 번거로우므로, 에이전트에게 인덱스를 Mermaid 그래프로 그리게 해서 눈으로 확인합니다.

```text
> docs/backlog/index.json 의 dependsOn으로 Mermaid flowchart를 만들어 docs/backlog/dependency-graph.md 에 저장하고,
  순환 의존이 있으면 경로를 알려 주십시오. 의존이 없는 Task(바로 시작 가능한 것) 목록도 함께.
```

**기대 결과**: 2)와 3)의 출력이 비어 있습니다. 4)의 출력은 E4·E5 등 이번에 분해하지 않은 Epic의 FR만 나와야 합니다. E1~E3의 FR이 나오면 해당 FR에 Task가 빠진 것입니다. 의존 없는 Task가 3개 이상이면 병렬 진행 여지가 있습니다.

### Step 5. 다른 도구로 크기 검토 후 재분해
목적: 숫자 기준은 통과했지만 실제로는 너무 큰 Task를 다른 도구의 눈으로 찾아 다시 쪼갭니다.

Task를 Claude로 만들었다면 Codex가, Codex로 만들었다면 Claude가 검토합니다.

**Codex 레시피 (검토)**

```bash
codex exec -o docs/backlog/size-review.md \
  "docs/backlog/tasks/ 의 Task 명세를 검토하십시오. 파일은 수정하지 마십시오.
   각 Task에 대해 '한 세션에 끝낼 수 있습니까'를 S(충분히 작음)/M(경계)/L(너무 큼)으로 판정하고,
   L에는 쪼개는 방법(새 Task 2~3개의 제목과 경계)을 제안하십시오.
   또한 완료 조건이 명령으로 검증 불가능한 Task, 범위 밖이 비어 있는 Task를 따로 표시하십시오."
```

**Claude Code 레시피 (검토)**

```bash
claude -p "docs/backlog/tasks/ 의 Task 명세를 검토하십시오. 파일은 수정하지 마십시오.
  (위와 같은 S/M/L 판정 지시)" --permission-mode plan > docs/backlog/size-review.md
```

L 판정을 받은 Task는 사람이 판단한 뒤 다시 쪼갭니다. 쪼갠 후에는 Step 4의 인덱스 생성과 `jq` 검증을 다시 돌립니다.

```text
> docs/backlog/size-review.md 에서 L 판정을 받은 TASK-009를 제안대로 TASK-009, TASK-021로 쪼개 주십시오.
  TASK-009를 의존하던 다른 Task의 "의존 작업"도 함께 갱신해 주십시오.
```

**기대 결과**: L 판정 Task가 0개가 됩니다. 최종 Task 수는 20개를 조금 넘을 수 있습니다. 커밋합니다.

```bash
git add docs/backlog
git commit -m "docs(backlog): epics, stories, 20+ task specs with index"
```

> 파일럿 권장: 백로그를 확정하기 전에 의존 없는 Task 하나를 골라 실제로 계획만 세우게 해 봅니다.
> `claude -p "docs/backlog/tasks/TASK-001-*.md 를 구현하는 계획을 세우십시오. 바꿀 파일 목록과 예상 줄 수를 포함하십시오." --permission-mode plan`
> 또는 `codex exec "(같은 지시)"`. 계획에 나온 파일 수가 명세의 예상치보다 두 배 이상 많으면 기준을 다시 조정합니다.

## ✅ 체크포인트
- [ ] `docs/backlog/epics.md`의 FR → Epic 대응표에 빠진 FR이 없습니다.
- [ ] `docs/backlog/stories.md`의 모든 Story에 FR ID와 Given/When/Then 수용 기준이 있습니다.
- [ ] `docs/backlog/tasks/`에 템플릿 구조를 따르는 Task 명세가 20개 이상 있습니다.
- [ ] `docs/backlog/index.json`에 대한 `jq` 검사에서 없는 의존, 크기 기준 위반이 0건입니다.
- [ ] 다른 도구의 크기 검토에서 L 판정을 받은 Task를 모두 재분해했습니다.

## 🏠 숙제
| # | 유형 | 난이도 | 과제 | 완료 조건 |
|---|---|---|---|---|
| HW1 | 🔁 Reproduce | ★ | 따라하기 Step 1~4를 재현해 TaskFlow 설계서를 Task 명세 20개로 분해합니다 | `docs/backlog/tasks/`에 Task 20개, `index.json`과 `jq` 검사 4종의 출력 로그를 제출합니다. 없는 의존·크기 위반이 0건입니다 |
| HW2 | 🛠 Apply | ★★ | Task 명세 템플릿에 맞춰 TaskFlow 전체(E1~E5) 백로그를 구성합니다. B1 마일스톤 기준인 Task 30개 이상을 만듭니다 | Task 30개 이상, PRD의 모든 FR이 하나 이상의 Task에 연결됩니다(Step 4의 4번 검사 출력이 비어 있습니다). 의존성 그래프와 "바로 시작 가능한 Task" 목록, 다른 도구의 S/M/L 크기 검토 결과와 재분해 내역을 제출합니다 |
| HW3 | 🚀 Challenge | ★★★ | "한 세션 크기" 기준을 실측으로 보정합니다. 크기가 다른 Task 3개(S, M, 경계선 L)를 실제로 에이전트에게 구현시키고 결과를 기록합니다 | Task 3개 각각에 대해 사용한 도구, 소요 시간, 실제 변경 파일 수·diff 줄 수, 사람 개입 횟수, 완료 조건 통과 여부를 표로 제출합니다. 그 결과로 개념 섹션의 기준표 중 최소 1개 값을 조정한 근거를 씁니다 |

제출: `hw/02-3` 브랜치, `submissions/02-3.md` (템플릿: docs/design/homework-template.md)

## ⚠️ 흔한 실수
- **Task 완료 조건이 "기능이 정상 동작합니다"로 끝납니다** → 원인: 분해 지시에 검증 명령 형식을 요구하지 않았습니다 → 대응: Step 3처럼 "실행 가능한 검증 명령 1~3개 + 관찰 가능한 결과"를 형식으로 강제하고, 크기 검토에서 "명령으로 검증 불가능한 Task"를 따로 찾게 합니다.
- **Task 명세에 존재하지 않는 스크립트(`pnpm test:contract` 등)가 들어 있습니다** → 원인: 에이전트가 그럴듯한 스크립트 이름을 지어냈습니다 → 대응: 루트 `package.json`의 scripts를 확인하라고 지시하고, 없으면 "TODO: 스크립트 추가 필요"로 남기게 합니다. 필요한 스크립트 추가 자체를 별도 Task로 만듭니다.
- **`codex exec`로 Task 파일을 만들라고 했는데 파일이 하나도 없습니다** → 원인: `codex exec`의 기본 샌드박스가 read-only입니다 → 대응: `codex exec --sandbox workspace-write`를 명시합니다.
- **`--output-schema`나 `--json-schema` 결과가 비어 있거나 오류가 납니다** → 원인: 스키마의 루트가 배열이거나 `required`·`additionalProperties`가 느슨한 경우가 많습니다 → 대응: Step 4처럼 루트를 객체로 두고 모든 속성을 `required`에 넣습니다. Claude는 `--output-format json`과 함께 써야 `structured_output`을 받을 수 있습니다.
- **백로그가 레이어별로 쪼개져 Story가 끝까지 완성되지 않습니다** → 원인: "DB 스키마 전부 → API 전부 → 웹 전부" 순으로 Task를 만들었습니다 → 대응: Story를 수직 슬라이스로 유지하고, Task 순서도 Story 단위로 끝나게 배치합니다. 그래야 Story마다 데모와 통합 검증이 가능합니다.

## 🔗 참고 자료
- [Claude Code headless (`-p`, `--output-format json`, `--json-schema`)](https://code.claude.com/docs/en/headless)
- [Claude Code CLI reference](https://code.claude.com/docs/en/cli-reference)
- [Codex non-interactive mode (`--output-schema`, `-o`, `--sandbox`)](https://learn.chatgpt.com/docs/non-interactive-mode)
- [Codex approvals & security](https://learn.chatgpt.com/docs/agent-approvals-security)
- 템플릿: [templates/task-spec.md](../../templates/task-spec.md), [templates/handoff-note.md](../../templates/handoff-note.md)
- 이 가이드의 관련 레슨: 이전 레슨 [02-2 아키텍처와 ADR](./02-2-architecture-and-adr.md), 다음 레슨 [02-4 인터페이스 우선 설계](./02-4-interface-first.md), [03-1 Explore → Plan → Implement → Verify](../03-agentic-workflow/03-1-explore-plan-implement-verify.md), [03-4 컨텍스트 관리](../03-agentic-workflow/03-4-context-management.md), [05-1 병렬 에이전트](../05-scaling-up/05-1-parallel-agents.md)
- 프로젝트: [Project B — TaskFlow](../../projects/B-development-project/README.md) (B1 마일스톤: 백로그 Task 명세 30+)
