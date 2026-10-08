---
id: 04-1
title: "검증 계층: 린트·타입·단위·통합·E2E"
module: 04-quality-and-verification
level: L2
duration: 2h
last-verified: 2026-10-07
tools:
  claude-code: 2.1.292
  codex: 0.160.1
---

# 04-1. 검증 계층: 린트·타입·단위·통합·E2E

## 🎯 학습 목표
- 린트·타입체크·단위·통합·E2E 검증 계층이 각각 무엇을 잡고 얼마나 걸리는지 설명할 수 있습니다.
- 에이전트가 사람 도움 없이 실행할 수 있는 검증 커맨드(`make verify-fast`, `make verify`)를 정의할 수 있습니다.
- 검증 커맨드를 `AGENTS.md`에 적어 Claude Code와 Codex가 같은 기준으로 "완료"를 판단하게 할 수 있습니다.
- 비대화형 실행(`claude -p`, `codex exec`)으로 "검증 실패 → 수정 → 재검증" 루프를 돌릴 수 있습니다.

## 📋 사전 준비
- 선행 레슨: [03-1 Explore → Plan → Implement → Verify](../03-agentic-workflow/03-1-explore-plan-implement-verify.md), [03-2 에이전트와 TDD](../03-agentic-workflow/03-2-tdd-with-agents.md), [01-4 확장 기능](../01-environment-setup/01-4-extensions.md)(hooks)
- 필요 도구/계정: Claude Code 2.1.292, Codex 0.160.1, Node.js 22+, pnpm, Docker(통합 테스트용 PostgreSQL), GNU Make
- 실습 저장소 상태: Project B(TaskFlow)의 B3 완료 상태. 아래 구조를 가정합니다.

```text
taskflow/
├── apps/
│   ├── api/        # Fastify + PostgreSQL
│   └── web/        # Next.js
├── packages/
│   ├── db/         # 스키마, 마이그레이션
│   └── shared/     # 타입 계약, 공용 유틸
├── turbo.json
├── package.json
├── AGENTS.md
└── CLAUDE.md       # 내용: @AGENTS.md + Claude 전용 지시
```

## 💡 개념

### 왜 "검증 계층"인가
설계 원칙 3번 **Verify, Don't Trust**는 "에이전트가 됐다고 함"을 완료로 인정하지 않는다는 뜻입니다. 그런데 검증이 사람 손에 달려 있으면 병목이 됩니다. 에이전트가 하루에 PR을 10개 만들어도 사람이 매번 터미널에서 테스트를 돌려 봐야 한다면, 처리량은 사람의 속도를 넘지 못합니다.

그래서 대규모 개발에서는 **에이전트가 스스로 돌릴 수 있는 검증 커맨드**가 필수입니다. 이 커맨드는 다음 세 가지 조건을 만족해야 합니다.

1. **한 줄입니다** — `make verify`처럼 외울 필요가 없는 하나의 진입점이 있습니다. 커맨드가 여러 개로 흩어지면 에이전트는 그중 일부만 돌리고 "완료"라고 말합니다.
2. **결정적입니다** — 같은 코드에 같은 결과를 냅니다. 불안정한(flaky) 테스트는 에이전트를 "무한 수정 루프"([00-4](../00-foundations/00-4-failure-patterns.md))에 빠뜨립니다.
3. **계층화되어 있습니다** — 빠른 검사는 자주, 느린 검사는 마무리할 때 돌립니다. 편집할 때마다 E2E를 돌리면 컨텍스트와 시간을 낭비합니다.

### 계층별로 무엇을 잡습니까

```text
           느림 · 비쌈 · 현실에 가까움
                    ▲
        ┌───────────────────────┐
        │  E2E (Playwright)     │  화면 흐름, 배포 설정, 브라우저 동작      (04-4)
        ├───────────────────────┤
        │  통합 (API + DB)      │  SQL, 트랜잭션, HTTP 계약, 마이그레이션
        ├───────────────────────┤
        │  단위 (Vitest)        │  도메인 로직, 경계값, 에러 처리
        ├───────────────────────┤
        │  타입 (tsc --noEmit)  │  환각 API, 잘못된 import, 계약 위반
        ├───────────────────────┤
        │  린트·포맷 (ESLint)   │  스타일, 금지 패턴, 미사용 코드
        └───────────────────────┘
                    ▼
           빠름 · 쌈 · 자주 돌림

  make verify-fast = 린트 + 타입 + 단위 (변경 패키지만)   → 편집 루프마다
  make verify      = verify-fast + 통합 + E2E (전체)      → 완료 선언 전, CI
```

에이전트 특유의 실패와 계층을 연결하면 왜 각 계층이 필요한지 분명해집니다.

| 에이전트 실패 패턴 | 가장 먼저 잡는 계층 |
|---|---|
| 존재하지 않는 함수·옵션을 지어냄(환각 API) | 타입체크 |
| 범위를 벗어나 다른 패키지 코드를 고침 | 린트(경계 규칙), 단위 테스트 |
| 테스트를 통과시키려고 테스트를 약화함 | 통합·E2E(사람이 쓴 시나리오), 리뷰(04-2) |
| "로컬에서는 됐습니다"라는 환경 의존 코드 | 통합(실제 DB), E2E |

### 검증 커맨드는 "계약"입니다
검증 커맨드를 `AGENTS.md`에 적는 순간, 그것은 사람과 에이전트 사이의 **완료 정의(Definition of Done)** 가 됩니다. Claude Code는 `CLAUDE.md`의 `@AGENTS.md` import로, Codex는 `AGENTS.md`를 직접 읽어서 같은 계약을 공유합니다. 05-4에서는 같은 커맨드를 CI가 실행합니다. 로컬·에이전트·CI가 **같은 한 줄**을 돌리는 것이 목표입니다.

## 👣 따라하기

### Step 1. 현재 검증 수단을 조사합니다
목적: 저장소에 이미 있는 검증 스크립트와 빠진 계층을 에이전트에게 읽기 전용으로 조사시킵니다.

이 단계는 파일을 바꾸지 않으므로 두 도구 모두 읽기 전용 모드로 실행합니다.

**Claude Code 레시피**
```bash
claude --permission-mode plan
```
```text
> 이 모노레포의 검증 수단을 조사해 주십시오. 루트와 각 패키지의 package.json scripts,
  turbo.json의 tasks, CI 워크플로(.github/workflows)를 읽고 아래 표로 정리하십시오.
  | 계층(lint/typecheck/unit/integration/e2e) | 패키지 | 실행 커맨드 | 예상 소요 | 비고 |
  빠진 계층과, 패키지마다 이름이 다른 스크립트(예: test vs test:unit)도 지적하십시오.
  파일은 수정하지 마십시오.
```

**Codex 레시피**
```bash
codex --sandbox read-only --ask-for-approval on-request
```
```text
> Inventory the verification commands in this monorepo. Read root and package
  package.json scripts, turbo.json tasks, and .github/workflows. Output a table:
  | layer (lint/typecheck/unit/integration/e2e) | package | command | est. time | notes |
  Point out missing layers and inconsistent script names. Do not modify files.
```

**기대 결과**: 계층별 표가 나옵니다. TaskFlow B3 상태라면 보통 다음과 같은 지적이 나옵니다.
- `apps/web`에는 `test` 스크립트가 없거나 `apps/api`와 이름이 다릅니다.
- 통합 테스트가 단위 테스트와 같은 `test` 스크립트에 섞여 있어 DB 없이 돌리면 실패합니다.
- E2E 계층이 비어 있습니다(04-4에서 채웁니다).

표를 `docs/verification.md` 초안으로 저장해 둡니다. 두 도구의 표가 다르면 실제 파일을 열어 어느 쪽이 맞는지 확인합니다. 조사 결과도 검증 대상입니다.

### Step 2. 계층별 스크립트 이름을 통일합니다
목적: 모든 패키지가 같은 스크립트 이름을 갖게 해서 Turborepo 한 줄로 계층을 실행할 수 있게 합니다.

목표 규칙은 다음과 같습니다.

| 스크립트 | 의미 | 외부 의존 |
|---|---|---|
| `lint` | ESLint + Prettier 검사 | 없음 |
| `typecheck` | `tsc --noEmit` | 없음 |
| `test:unit` | Vitest 단위 테스트 | 없음 |
| `test:integration` | API + 실제 PostgreSQL | Docker |
| `test:e2e` | Playwright | 앱 실행 |

**Claude Code 레시피**
```bash
claude
```
```text
> 모든 패키지의 package.json scripts를 아래 규칙으로 통일해 주십시오.
  lint, typecheck(tsc --noEmit), test:unit, test:integration (해당 패키지만).
  turbo.json에 같은 이름의 task를 정의하고, typecheck와 test:unit은 ^build에 의존하게 하십시오.
  기존 테스트 파일은 옮기지 말고, 통합 테스트는 파일명 *.int.test.ts로 구분해서
  vitest 설정의 include/exclude로 분리하십시오.
  끝나면 pnpm turbo run lint typecheck test:unit 을 실행해서 결과를 보여 주십시오.
```

**Codex 레시피**
```bash
codex --sandbox workspace-write --ask-for-approval on-request
```
```text
> Normalize package.json scripts across all packages:
  lint, typecheck (tsc --noEmit), test:unit, test:integration (only where relevant).
  Define matching tasks in turbo.json; typecheck and test:unit depend on ^build.
  Do not move test files; split integration tests by the *.int.test.ts suffix via
  vitest include/exclude. Then run `pnpm turbo run lint typecheck test:unit` and show the result.
```

**기대 결과**: `turbo.json`이 대략 다음 모양이 됩니다. 실제 키 이름은 사용하는 Turborepo 버전을 따릅니다.

```json
{
  "tasks": {
    "build":            { "dependsOn": ["^build"], "outputs": ["dist/**", ".next/**"] },
    "lint":             {},
    "typecheck":        { "dependsOn": ["^build"] },
    "test:unit":        { "dependsOn": ["^build"] },
    "test:integration": { "dependsOn": ["^build"], "cache": false },
    "test:e2e":         { "dependsOn": ["build"], "cache": false }
  }
}
```

`pnpm turbo run lint typecheck test:unit`이 Docker 없이 통과해야 합니다. 통과하지 않으면 실패한 패키지를 확인하고, 에이전트가 테스트를 `skip` 처리해서 통과시키지 않았는지 `git diff`로 확인합니다.

### Step 3. `make verify` 한 줄 진입점을 만듭니다
목적: 계층을 조합한 두 개의 진입점 `verify-fast`(편집 루프용)와 `verify`(완료 선언용)를 만듭니다.

**Claude Code 레시피**
```text
> 루트에 Makefile을 만들어 주십시오. 타깃은 verify-fast, verify, test-integration, e2e.
  - verify-fast: 변경된 패키지만 lint, typecheck, test:unit (turbo --filter 사용, 기준은 origin/main)
  - test-integration: docker compose로 postgres를 띄우고 test:integration 실행 후 종료하십시오
  - verify: 전체 lint typecheck test:unit + test-integration (+ e2e는 04-4에서 추가)
  - 실패하면 0이 아닌 종료 코드를 반환해야 합니다. 출력은 실패한 부분만 보이도록 간결하게.
  만든 뒤 make verify-fast 와 make verify 를 각각 실행하고 소요 시간을 알려 주십시오.
```

**Codex 레시피**
```text
> Create a root Makefile with targets verify-fast, verify, test-integration, e2e.
  - verify-fast: lint, typecheck, test:unit for changed packages only (turbo --filter vs origin/main)
  - test-integration: start postgres with docker compose, run test:integration, tear down
  - verify: full lint typecheck test:unit + test-integration (e2e added in lesson 04-4)
  Must exit non-zero on failure; keep output concise. Run both targets and report durations.
```

> Codex의 `workspace-write` 샌드박스는 기본적으로 네트워크를 막습니다. `docker compose`나 패키지 설치가 실패하면 승인 요청에 응답하거나, 이 세션만 `-c sandbox_workspace_write.network_access=true`로 시작합니다. Docker 소켓 접근이 샌드박스에서 허용되는지는 환경마다 다릅니다. TODO(verify)

**기대 결과**: 다음과 비슷한 Makefile이 생깁니다.

```makefile
.PHONY: verify-fast verify test-integration e2e

verify-fast:
	pnpm turbo run lint typecheck test:unit --filter='...[origin/main]' --output-logs=errors-only

test-integration:
	docker compose -f docker-compose.test.yml up -d --wait
	pnpm turbo run test:integration --output-logs=errors-only; \
	  status=$$?; docker compose -f docker-compose.test.yml down; exit $$status

e2e:
	@echo "04-4에서 추가합니다" && exit 0

verify:
	pnpm turbo run lint typecheck test:unit --output-logs=errors-only
	$(MAKE) test-integration
	$(MAKE) e2e
```

확인할 것:
- `make verify-fast`는 수십 초 안에 끝납니다. 변경이 없는 패키지는 Turborepo 캐시로 건너뜁니다.
- 일부러 `apps/api/src/tasks/service.ts`에 타입 오류를 넣고 `make verify-fast; echo $?`를 실행하면 0이 아닌 값이 나옵니다. 확인 후 되돌립니다.
- `test-integration`이 실패해도 컨테이너가 내려가는지(`docker ps`) 확인합니다.

### Step 4. 검증 계약을 `AGENTS.md`에 적고 hook으로 강제합니다
목적: 두 도구가 같은 완료 정의를 읽게 하고, 편집 직후 빠른 검사가 자동으로 돌게 합니다.

이 저장소는 `CLAUDE.md`가 `@AGENTS.md`를 import하므로, 공통 규칙은 `AGENTS.md` 한 곳에만 씁니다.

```markdown
## 검증 (Definition of Done)
- 편집 중에는 `make verify-fast`를 수시로 실행합니다.
- 작업 완료를 선언하기 전에 `make verify`를 실행하고, 마지막 출력 요약을 보고에 포함합니다.
- 검증을 통과시키려고 테스트를 삭제·skip·약화하지 않습니다. 테스트가 틀렸다고 판단되면 수정하지 말고 이유를 보고합니다.
- 검증이 3회 연속 같은 이유로 실패하면 멈추고 가설과 로그를 보고합니다.
```

**Claude Code 레시피** — `PostToolUse` hook으로 편집한 파일에 린트를 겁니다. 아래는 `.claude/settings.json`의 일부입니다.
```json
{ "hooks": { "PostToolUse": [ { "matcher": "Write|Edit",
  "hooks": [ { "type": "command", "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/lint.sh" } ] } ] } }
```
```bash
#!/usr/bin/env bash
# .claude/hooks/lint.sh — 변경된 패키지만 린트합니다
cd "$CLAUDE_PROJECT_DIR" || exit 0
if ! out=$(pnpm turbo run lint --filter='...[HEAD]' --output-logs=errors-only 2>&1); then
  echo "$out" >&2
  exit 2   # PostToolUse에서 exit 2 + stderr를 Claude에게 피드백으로 전달하는 규약 — TODO(verify)
fi
```
설정 후 세션 안에서 `/hooks`로 등록됐는지 확인합니다. hook의 종료 코드별 동작(성공·차단·경고)은 [Claude hooks](https://code.claude.com/docs/en/hooks) 문서의 이벤트별 설명을 따릅니다. 작업 종료 시점에 `make verify`를 강제하는 `Stop` hook도 가능하지만, 종료를 막는 정확한 반환 규약은 문서로 확인한 뒤 씁니다. TODO(verify)

**Codex 레시피** — `<repo>/.codex/hooks.json`에 같은 구조로 등록합니다.
```json
{ "hooks": { "PostToolUse": [ { "matcher": "Bash",
  "hooks": [ { "type": "command", "command": "bash \"$(git rev-parse --show-toplevel)/.codex/hooks/lint.sh\"" } ] } ] } }
```
Codex hook은 **`/hooks`에서 검토하고 신뢰해야 실행됩니다**. 등록만 하고 신뢰하지 않으면 아무 일도 일어나지 않습니다. Codex에서 파일 편집이 어떤 도구 이름으로 hook matcher에 잡히는지는 버전마다 다를 수 있으니 `/hooks`에서 확인합니다. TODO(verify)

**기대 결과**:
- Claude에서 아무 파일이나 고치게 하면 편집 직후 lint가 돌고, 위반이 있으면 Claude가 스스로 고칩니다.
- Codex에서 `/hooks`를 열면 hook이 보이고, 신뢰한 뒤부터 실행됩니다.
- 두 도구에 "완료 조건이 무엇입니까?"라고 물으면 둘 다 `make verify`를 답합니다.

### Step 5. 비대화형 "검증 → 수정" 루프를 돌립니다
목적: 사람 없이 에이전트가 검증 실패를 고치는 루프를 체험하고, 그 한계를 관찰합니다.

먼저 실패 상황을 만듭니다. 별도 브랜치에서 `packages/shared`의 `TaskStatus` 타입에 값을 하나 추가합니다(예: `"blocked"`). API와 웹의 `switch` 문이 망라되지 않아 타입 오류와 단위 테스트 실패가 생깁니다.

```bash
git switch -c lab/04-1-verify-loop
# packages/shared/src/task.ts 의 TaskStatus 에 "blocked" 추가 후
make verify-fast   # 실패 확인
```

**Claude Code 레시피**
```bash
claude -p "make verify-fast 가 실패합니다. 원인을 찾아 고치고 다시 실행해서 통과시키십시오. \
테스트를 삭제하거나 skip 하지 마십시오. 최대 3회 시도 후에도 실패하면 멈추고 원인을 보고하십시오. \
마지막에 변경 파일 목록과 make verify-fast 출력 요약을 보여 주십시오." \
  --allowedTools "Read" "Edit" "Bash(make verify-fast)" "Bash(pnpm turbo *)" "Bash(git diff *)" \
  --output-format json | jq -r '.result'
```

**Codex 레시피**
```bash
codex exec --sandbox workspace-write \
  "make verify-fast fails. Find the cause, fix it, and re-run until it passes. \
Do not delete or skip tests. Stop after 3 attempts and report the cause if still failing. \
Finish with the list of changed files and a summary of the make verify-fast output." \
  -o reports/04-1-codex.md
```

**기대 결과**:
- 두 도구 모두 `blocked` 상태를 처리하는 분기를 추가하고 `make verify-fast`를 통과시킵니다.
- `git diff`로 다음을 확인합니다. 테스트 파일이 약화되지 않았습니까? `default:` 분기에 `as never` 같은 타입 우회를 넣지 않았습니까? UI에서 `blocked`의 표시 문구는 제품 결정이므로 에이전트가 임의로 정했다면 표시해 둡니다.
- 마지막으로 `make verify`(통합 포함)를 사람이 한 번 실행합니다. 빠른 계층만 통과하고 통합 계층에서 실패하는 경우(예: DB enum 마이그레이션 누락)가 자주 나옵니다. 이것이 계층을 나누는 이유입니다.

## ✅ 체크포인트
- [ ] Step 1의 검증 수단 조사표를 `docs/verification.md`에 저장했습니다.
- [ ] 모든 패키지가 `lint`, `typecheck`, `test:unit`(필요 시 `test:integration`) 이름을 씁니다.
- [ ] `make verify-fast`와 `make verify`가 실패 시 0이 아닌 종료 코드를 반환합니다.
- [ ] `AGENTS.md`에 검증 계약(완료 정의)이 있고, Claude는 `@AGENTS.md`로 같은 내용을 읽습니다.
- [ ] Claude hook은 `/hooks`에 보이고, Codex hook은 `/hooks`에서 신뢰했습니다.
- [ ] 비대화형 루프 결과를 `git diff`로 직접 검토했습니다(테스트 약화 여부 포함).

## 🏠 숙제
| # | 유형 | 난이도 | 과제 | 완료 조건 |
|---|---|---|---|---|
| HW1 | 🔁 Reproduce | ★ | 따라하기 Step 1~3을 본인 TaskFlow 저장소에서 재현합니다 | 조사표, `Makefile`, `turbo.json` 변경이 커밋되어 있습니다. 타입 오류를 일부러 넣었을 때 `make verify-fast`가 실패하는 로그와, 되돌린 뒤 통과하는 로그를 첨부합니다 |
| HW2 | 🛠 Apply | ★★ | `make verify` 한 줄로 전체 검증을 구성합니다(통합 포함). Step 5의 루프를 Claude와 Codex로 각각 돌려 비교합니다 | `make verify`가 깨끗한 클론에서 통과합니다. 두 도구의 결과를 "시도 횟수 / 변경 파일 수 / 테스트 약화 여부 / 사람 개입 횟수" 표로 비교합니다 |
| HW3 | 🚀 Challenge | ★★★ | 검증 시간 예산을 설계합니다. `verify-fast`를 60초 이내로 줄이고, 계층별 소요 시간을 측정해 기록하는 스크립트를 만듭니다 | 계층별 소요 시간 측정 결과(전/후)가 표로 있습니다. 줄인 방법(캐시, `--filter`, 테스트 분할 등)과 그로 인해 놓칠 수 있는 결함 유형을 분석합니다 |

제출: `hw/04-1` 브랜치, `submissions/04-1.md` (템플릿: [homework-template.md](../../docs/design/homework-template.md))

## ⚠️ 흔한 실수
- **에이전트가 "테스트 통과"라고 보고했는데 CI에서 실패합니다** → 에이전트가 일부 패키지나 일부 계층만 돌렸습니다(예: `pnpm test`를 한 패키지에서만 실행) → 진입점을 `make verify` 하나로 정하고 `AGENTS.md`에 "완료 보고에 `make verify` 출력 요약을 포함합니다"를 넣습니다.
- **검증 루프가 끝나지 않습니다** → 불안정한 테스트나 외부 네트워크에 의존하는 테스트가 무작위로 실패해 에이전트가 계속 "수정"합니다 → 시도 횟수 상한(3회)을 프롬프트와 `AGENTS.md`에 적고, 불안정한 테스트는 격리 목록으로 옮겨 사람이 처리합니다.
- **테스트가 사라지거나 약해졌습니다** → "통과시키십시오"라는 지시를 문자 그대로 따라 `it.skip`, 느슨한 단언, 타입 단언(`as any`)으로 우회했습니다 → 금지 규칙을 `AGENTS.md`에 적고, 린트 규칙(예: 포커스·skip 테스트 금지)을 추가하고, `git diff -- '*.test.ts'`를 리뷰 체크 항목으로 둡니다(04-2).
- **Codex `exec`가 아무 파일도 바꾸지 않습니다** → `codex exec`의 기본 샌드박스는 read-only입니다 → `--sandbox workspace-write`를 명시합니다. `--full-auto`는 0.160.1에서 오류가 납니다.
- **Codex hook이 실행되지 않습니다** → hook을 등록만 하고 `/hooks`에서 신뢰하지 않았습니다 → `/hooks`에서 검토·신뢰합니다. hook 파일을 고치면 해시가 바뀌므로 다시 신뢰해야 합니다.

## 🔗 참고 자료
- [Claude Code hooks](https://code.claude.com/docs/en/hooks)
- [Claude Code headless](https://code.claude.com/docs/en/headless)
- [Claude Code CLI reference](https://code.claude.com/docs/en/cli-reference)
- [Codex non-interactive mode](https://learn.chatgpt.com/docs/non-interactive-mode)
- [Codex hooks](https://learn.chatgpt.com/docs/hooks)
- [Codex AGENTS.md](https://learn.chatgpt.com/docs/agent-configuration/agents-md)
- 이 저장소: [도구 레퍼런스](../../docs/reference/tool-reference.md), [01-1 프로젝트 메모리](../01-environment-setup/01-1-project-memory.md), [01-4 확장 기능](../01-environment-setup/01-4-extensions.md), [03-2 에이전트와 TDD](../03-agentic-workflow/03-2-tdd-with-agents.md), [04-2 교차 리뷰](./04-2-cross-review.md), [04-4 브라우저/E2E 검증](./04-4-browser-e2e.md), [05-4 CI/CD 통합](../05-scaling-up/05-4-ci-cd-integration.md)
- 템플릿: [AGENTS.md.template](../../templates/AGENTS.md.template), [task-spec.md](../../templates/task-spec.md)
