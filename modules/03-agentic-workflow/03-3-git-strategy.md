---
id: 03-3
title: "Git 전략: 브랜치, 커밋 단위, PR 크기"
module: 03-agentic-workflow
level: L2
duration: 2h
last-verified: 2026-10-07
tools:
  claude-code: 2.1.292
  codex: 0.160.1
---

# 03-3. Git 전략: 브랜치, 커밋 단위, PR 크기

## 🎯 학습 목표
- 에이전트 작업에 맞는 브랜치 이름, 커밋 단위, PR 크기 규칙을 정의하고 `AGENTS.md`에 고정할 수 있다.
- 뒤섞인 작업 트리를 에이전트에게 논리 단위 커밋으로 나누게 할 수 있다.
- 커밋 메시지와 PR 설명 작성을 Skill로 패키징해 Claude Code와 Codex에서 같은 규칙으로 쓸 수 있다.
- PR 템플릿에 "검증 증거"와 "AI 사용 기록"을 넣어 리뷰어가 에이전트 산출물을 판단할 수 있게 할 수 있다.

## 📋 사전 준비
- 선행 레슨: [03-1 Explore → Plan → Implement → Verify](./03-1-explore-plan-implement-verify.md), [03-2 에이전트와 TDD](./03-2-tdd-with-agents.md), [01-4 확장 기능](../01-environment-setup/01-4-extensions.md)
- 필요 도구/계정: Claude Code 2.1.292 이상, Codex 0.160.1 이상, `git`, GitHub 계정과 `gh` CLI(로그인 상태)
- 실습 저장소 상태: Project B(TaskFlow) B2 진행 중, GitHub에 원격 저장소 `origin`이 있고 기본 브랜치는 `main`이다.

## 💡 개념

### 에이전트는 커밋을 "너무 크게" 또는 "너무 아무렇게나" 만든다
에이전트에게 Git 규칙을 주지 않으면 두 가지 극단이 나온다.

- **거대한 단일 커밋**: 세션 끝에 "변경 사항 커밋해줘" 한 번으로 테스트·구현·리팩터링·포맷 변경이 한 커밋에 들어간다. 되돌리기도, 리뷰하기도 어렵다.
- **의미 없는 잦은 커밋**: "fix", "update" 같은 메시지가 수십 개 쌓인다. 이력에서 의도를 읽을 수 없다.

사람 개발자 한 명이 하루에 여는 PR이 1~2개라면, 에이전트를 쓰는 개발자는 그 몇 배를 연다. 병렬 에이전트([05-1](../05-scaling-up/05-1-parallel-agents.md))까지 쓰면 더 늘어난다. **리뷰가 병목**이 되고, 그 병목을 줄이는 수단이 작은 PR과 읽히는 커밋 이력이다. 대규모 개발에서 Git 규칙은 취향이 아니라 처리량 문제다.

### 세 가지 단위를 맞춘다

```text
Task 명세 1개 ──► 브랜치 1개 ──► PR 1개 (리뷰 가능한 크기)
                                  │
                                  ├─ 커밋: test(api): ...      ← 각 커밋은 하나의 의도
                                  ├─ 커밋: feat(api): ...      ← 각 커밋에서 pnpm verify 통과
                                  └─ 커밋: refactor(api): ...
```

| 단위 | 규칙 (TaskFlow 기준) | 이유 |
|---|---|---|
| 브랜치 | `<type>/<TASK-ID>-<slug>` 예: `feat/TASK-032-task-filter` | 브랜치 이름만 보고 명세를 찾는다 |
| 커밋 | Conventional Commits `type(scope): 요약`, 본문에 이유, 꼬리말 `Refs: TASK-032` | 의도 단위로 되돌리기, 변경 이력 자동 생성 |
| 커밋 상태 | 각 커밋에서 빌드·테스트가 통과한다 (TDD의 Red 커밋은 예외로 명시) | `git bisect`로 원인 커밋을 찾을 수 있다([03-5](./03-5-debugging.md)) |
| PR 크기 | 변경 400줄 이하 권장 (lock 파일·생성 코드 제외), 넘으면 분할 제안 | 리뷰 품질은 PR 크기에 반비례한다 |
| PR 본문 | 템플릿: 요약, 관련 Task, 검증 증거, AI 사용 기록, 리뷰 포인트 | 리뷰어가 "무엇을 믿어도 되는지" 판단한다 |

400줄은 이 가이드가 정한 출발점일 뿐이다. 팀 지표([06-2](../06-team-and-operations/06-2-metrics.md))를 보며 조정한다.

### 도구별 Git 권한 차이
| 항목 | Claude Code | Codex |
|---|---|---|
| 에이전트의 `git commit` | 권한 모드와 허용 규칙에 따른다. 예: `"allow": ["Bash(git commit *)"]`, `"ask": ["Bash(git push *)"]` | `workspace-write`에서도 `.git`은 읽기 전용으로 보호된다. 커밋은 샌드박스 밖 실행 승인이 필요하거나 사람이 실행한다 |
| 네트워크(`git push`, `gh pr create`) | 허용 규칙으로 제어 | `workspace-write`는 기본적으로 네트워크를 막는다(`[sandbox_workspace_write] network_access`로 허용 가능) |

이 차이 때문에 이 레슨의 Codex 레시피는 "Codex가 커밋 계획과 메시지를 만들고, 사람이 실행한다"는 흐름을 기본으로 한다.

## 👣 따라하기

### Step 1. Git 규칙을 `AGENTS.md`와 권한 설정에 고정한다
목적: 브랜치·커밋·PR 규칙을 두 도구가 공통으로 읽게 하고, 위험한 Git 명령은 권한으로 막는다.

**Claude Code 레시피**
```bash
claude
```
```text
> AGENTS.md 에 "## Git 규칙" 섹션을 추가해라. 내용:
  - 브랜치: <type>/<TASK-ID>-<slug>, type 은 feat|fix|refactor|test|docs|chore
  - 커밋: Conventional Commits. 형식 "type(scope): 요약(50자 이내, 한국어 또는 영어 통일)".
    scope 는 api|web|db|shared|worker|cli|repo. 본문에는 "왜" 를 쓴다. 꼬리말에 Refs: TASK-xxx.
  - 한 커밋에는 한 가지 의도만 담는다. 포맷 변경과 기능 변경을 섞지 않는다.
  - 각 커밋에서 pnpm verify 가 통과해야 한다. TDD 의 test: 커밋만 예외이며 본문에 "intentionally failing" 을 적는다.
  - PR 은 Task 1개당 1개, 변경 400줄 이하 권장(pnpm-lock.yaml, 생성 코드 제외). 넘으면 분할안을 먼저 제안한다.
  - 금지: git push --force, main 브랜치 직접 커밋, git commit --amend 로 이미 push 한 커밋 수정.
  다른 섹션은 건드리지 마라.
```
그리고 `.claude/settings.json`에 Git 관련 규칙을 추가한다(팀 공용).
```json
{
  "permissions": {
    "allow": ["Bash(git status *)", "Bash(git diff *)", "Bash(git log *)", "Bash(git add *)", "Bash(git commit *)"],
    "ask":   ["Bash(git push *)", "Bash(gh pr create *)"],
    "deny":  ["Bash(git push --force *)", "Bash(git push -f *)", "Bash(git reset --hard *)"]
  }
}
```

**Codex 레시피**
```bash
codex --sandbox workspace-write --ask-for-approval on-request
```
```text
> (Claude Code와 같은 AGENTS.md 프롬프트)
```
Codex는 `.git` 보호와 승인 정책으로 기본적인 안전장치가 있다. 샌드박스 밖 명령을 더 세밀하게 제어하는 `.rules` 파일은 [Rules 문서](https://learn.chatgpt.com/docs/agent-configuration/rules)를 참고한다. 위치와 문법 예시는 이 가이드에서 아직 검증하지 않았다. TODO(verify)

**기대 결과**
- `AGENTS.md`에 Git 규칙 섹션이 있다.
- Claude Code에서 `git push --force`를 시키면 거부된다. `Bash` 인수 패턴은 옵션 순서를 바꾸면 우회될 수 있으므로 이 규칙은 실수 방지용이지 보안 장치가 아니라는 점을 기억한다([01-2](../01-environment-setup/01-2-permissions-and-sandbox.md)).

### Step 2. 뒤섞인 작업 트리를 논리 단위 커밋으로 나눈다
목적: 에이전트가 만든 큰 변경을 리뷰 가능한 커밋 여러 개로 나누는 법을 익힌다.

실습 준비: TASK-032(작업 목록 필터) 구현을 커밋하지 않은 상태로 둔다. 실습을 위해 일부러 포맷 변경과 관련 없는 오타 수정을 섞어 둔다.

```bash
git switch -c feat/TASK-032-task-filter
git status --short
```

**Claude Code 레시피**
```text
> 현재 커밋되지 않은 변경 전체를 분석해서 커밋 계획을 세워라. 아직 커밋하지 마라.
  형식: 커밋마다 (순서, 메시지 초안, 포함할 파일 또는 hunk, 이 커밋만 적용했을 때 pnpm verify 가 통과할지 예상).
  TASK-032 와 관련 없는 변경은 별도 커밋이나 별도 브랜치로 분리하자고 제안해라.
```
계획을 검토한 뒤 실행시킨다.
```text
> 계획대로 커밋해라. 파일 단위는 git add <경로>, 한 파일 안의 일부 hunk 만 필요하면
  해당 hunk 만 담은 패치 파일을 만들고 git apply --cached 로 스테이징해라.
  커밋마다 git show --stat HEAD 를 보여줘라.
```

**Codex 레시피**
```text
> 현재 커밋되지 않은 변경 전체를 분석해서 커밋 계획을 세워라. 아직 커밋하지 마라.
  (형식은 Claude Code와 같다)
  계획이 확정되면 각 커밋을 만드는 셸 명령(git add ..., git commit -F <메시지 파일>)을 순서대로 출력하고,
  커밋 메시지는 .git 밖의 tmp/commit-msgs/01.txt 같은 파일로 작성해라.
```
사람이 명령을 확인하고 실행한다.
```bash
git add apps/api/src/modules/tasks/ packages/shared/src/task.ts
git commit -F tmp/commit-msgs/01.txt
```

**기대 결과**
```bash
$ git log --oneline main..HEAD
e5f6a7b feat(api): filter tasks by status and assignee
d4e5f6a feat(shared): add TaskFilter schema
c3d4e5f style(api): apply formatter to tasks module
```
- 기능 커밋과 포맷 커밋이 분리됐다.
- TASK-032와 관련 없는 오타 수정은 이 브랜치에 없다(별도 브랜치로 옮겼거나 stash).
- `git add -p` 같은 대화형 명령은 에이전트가 쓰기 어렵다. 에이전트에게는 파일 경로 지정이나 패치 파일 방식을 쓰게 한다.

> 💡 커밋 계획을 먼저 받는 것은 03-1의 Plan 단계를 Git에 적용한 것이다. 계획 없이 "나눠서 커밋해줘"라고 하면 대부분 파일 단위로 기계적으로 나눈다.

### Step 3. 커밋 메시지 규칙을 Skill로 패키징한다
목적: 커밋 메시지 작성 절차를 두 도구에서 같은 내용의 Skill로 재사용한다.

두 도구 모두 `SKILL.md`(front matter `name`, `description`)를 쓴다. 위치만 다르다. 같은 내용을 두 위치에 두거나, 한쪽을 원본으로 두고 다른 쪽에 복사한다.

`.claude/skills/taskflow-commit/SKILL.md` 와 `.agents/skills/taskflow-commit/SKILL.md`:
```markdown
---
name: taskflow-commit
description: Write a commit message for staged changes following the TaskFlow Git rules. Use when the user asks to commit or to write a commit message.
---
스테이징된 변경으로 커밋 메시지를 작성한다.

1. `git diff --cached --stat` 과 `git diff --cached` 로 변경을 확인한다. 스테이징된 것이 없으면 멈추고 알린다.
2. 변경이 두 가지 이상의 의도를 담고 있으면 커밋하지 말고 분할을 제안한다.
3. AGENTS.md 의 "Git 규칙"에 맞춰 메시지를 쓴다.
   - 제목: type(scope): 요약
   - 본문: 왜 바꿨는지 2~4줄. 무엇을 바꿨는지는 diff 가 말해 준다.
   - 꼬리말: Refs: TASK-xxx (브랜치 이름에서 추출)
4. 메시지를 tmp/commit-msgs/ 아래 파일로 저장하고 경로를 알려 준다.
5. 커밋 권한이 있으면 `git commit -F <파일>` 로 커밋한다. 권한이 없으면 실행할 명령을 출력한다.
```

**Claude Code 레시피**
```text
> /taskflow-commit
```

**Codex 레시피**
```text
> $taskflow-commit
```
`/skills`로 목록에서 고를 수도 있다.

**기대 결과**
- 두 도구가 같은 형식의 메시지 파일을 만든다.
- 의도가 섞인 스테이징이면 커밋하지 않고 분할을 제안한다. 일부러 두 의도를 섞어 스테이징해 보고 동작을 확인한다.
- Claude Code는 커밋까지 하고, Codex는 실행할 명령을 출력한다(또는 승인을 요청한다).

### Step 4. PR 템플릿을 만들고 PR 설명을 생성한다
목적: 리뷰어가 에이전트 산출물을 판단하는 데 필요한 정보를 PR 본문의 고정 섹션으로 만든다.

`.github/pull_request_template.md`:
```markdown
## 요약
<!-- 무엇을, 왜. 3줄 이내 -->

## 관련 Task
- Refs: TASK-xxx (docs/tasks/TASK-xxx.md)

## 변경 사항
<!-- 커밋 단위로 한 줄씩 -->

## 검증 증거
- [ ] `pnpm verify` 통과 (출력 마지막 줄 붙여넣기)
- [ ] 새/수정 테스트 목록:
- [ ] 수동 확인 (필요한 경우):

## AI 사용 기록
- 사용 도구: Claude Code / Codex
- 사람 개입 지점: <!-- 계획 수정, 테스트 보강, 되돌린 변경 등 -->
- 에이전트 리뷰 결과: <!-- /code-review 또는 codex review 요약과 처리 -->

## 리뷰 포인트
<!-- 리뷰어가 특히 봐야 할 곳. 에이전트가 확신하지 못한 부분 -->

## 범위 밖
<!-- 이 PR에서 일부러 하지 않은 것과 후속 Task -->
```

이어서 PR 설명 작성도 Skill로 만든다. `.claude/skills/taskflow-pr/SKILL.md` 와 `.agents/skills/taskflow-pr/SKILL.md`:
```markdown
---
name: taskflow-pr
description: Draft a pull request description for the current branch using the repository PR template. Use when the user asks to open or describe a PR.
---
1. `git log --oneline origin/main..HEAD` 와 `git diff --stat origin/main...HEAD` 로 범위를 확인한다.
2. 변경 줄 수를 계산한다: `git diff --numstat origin/main...HEAD -- . ':(exclude)pnpm-lock.yaml'`
   합계가 400줄을 넘으면 PR 설명 대신 분할안(브랜치별 커밋 목록)을 먼저 제시하고 멈춘다.
3. `.github/pull_request_template.md` 의 모든 섹션을 채운다.
   - 검증 증거는 직접 `pnpm verify` 를 실행한 결과로 채운다. 실행하지 않았으면 "미실행"이라고 쓴다.
   - AI 사용 기록은 이 세션에서 실제로 있었던 일만 쓴다.
4. 결과를 tmp/pr-body.md 로 저장하고 `gh pr create --title "<제목>" --body-file tmp/pr-body.md` 명령을 출력한다.
```

**Claude Code 레시피**
```text
> /taskflow-pr
```
출력된 `gh pr create` 명령을 확인한다. `ask` 규칙에 의해 실행 전 승인을 요청받는다.

**Codex 레시피**
```text
> $taskflow-pr
```
`gh pr create`는 네트워크가 필요하다. 기본 `workspace-write` 샌드박스에서는 막히므로 출력된 명령을 사람이 실행한다.
```bash
gh pr create --title "feat(api): filter tasks by status and assignee" --body-file tmp/pr-body.md
```

**기대 결과**
- PR 본문에 템플릿의 여섯 섹션이 모두 채워져 있다.
- "검증 증거"에 실제 실행 출력이 있다. "통과함" 같은 문장만 있으면 Skill의 3단계를 지키지 않은 것이다.
- `tmp/`는 `.gitignore`에 넣어 커밋되지 않게 한다.

### Step 5. PR 크기를 초과하면 분할한다
목적: 400줄을 넘는 변경을 리뷰 가능한 PR 여러 개로 나누는 흐름을 연습한다.

실습: TASK-033(웹 UI 필터)까지 한 브랜치에서 구현해 600줄 이상이 된 상황을 가정한다.

**Claude Code 레시피**
```text
> 현재 브랜치 변경이 400줄을 넘는다. 분할안을 제시해라.
  조건: 각 PR 은 독립적으로 pnpm verify 를 통과해야 하고, 머지 순서가 명확해야 한다.
  형식: PR 마다 (브랜치 이름, 포함할 커밋, 예상 변경 줄 수, 선행 PR).
```
분할안을 승인한 뒤 실행시킨다.
```text
> 분할안대로 브랜치를 만들어라. 첫 번째 브랜치는 main 에서, 두 번째 브랜치는 첫 번째 브랜치에서 분기하고
  git cherry-pick 으로 해당 커밋만 옮겨라. 각 브랜치에서 pnpm verify 를 실행하고 결과를 보여줘라.
```

**Codex 레시피**
```text
> (같은 분할안 프롬프트)
  분할안이 확정되면 브랜치 생성과 cherry-pick 명령을 순서대로 출력해라.
```
사람이 명령을 실행하고, 각 브랜치에서 Codex에게 검증을 시킨다.
```bash
git switch -c feat/TASK-032-task-filter-api main
git cherry-pick d4e5f6a e5f6a7b
```
```text
> pnpm verify 를 실행하고 결과 마지막 20줄을 보여줘라.
```

**기대 결과**
- PR이 API(TASK-032)와 웹 UI(TASK-033) 두 개로 나뉘고, 각각 400줄 이하다.
- 두 번째 PR 본문의 "범위 밖" 또는 "관련 Task"에 선행 PR이 적혀 있다.

### Step 6. 머지 전 브랜치 전체를 리뷰한다
목적: PR을 열기 전에 브랜치 단위 리뷰로 커밋 이력과 변경을 함께 점검한다.

**Claude Code 레시피**
```text
> /code-review medium feat/TASK-032-task-filter-api
```

**Codex 레시피**
```bash
codex review --base main
```

**기대 결과**: 리뷰 지적을 처리할 때 **새 커밋**으로 고친다(`fix(api): ...` 또는 `refactor(api): ...`). 이미 push한 커밋을 `--amend`로 바꾸지 않는다. 처리 결과는 PR 본문 "AI 사용 기록 → 에이전트 리뷰 결과"에 적는다.

## ✅ 체크포인트
- [ ] `AGENTS.md`에 Git 규칙 섹션이 있고, Claude Code 권한 설정에 force push 거부 규칙이 있다.
- [ ] 뒤섞인 작업 트리를 커밋 계획 → 실행 순서로 3개 이상의 논리 커밋으로 나눴다.
- [ ] `taskflow-commit` Skill을 두 도구에서 각각 한 번 이상 실행했다.
- [ ] `.github/pull_request_template.md`가 있고, 그 형식으로 PR을 1개 이상 열었다.
- [ ] PR 본문 "검증 증거"에 실제 실행 출력이 있다.
- [ ] 400줄 초과 변경을 2개 이상의 PR로 분할해 봤다.

## 🏠 숙제
| # | 유형 | 난이도 | 과제 | 완료 조건 |
|---|---|---|---|---|
| HW1 | 🔁 Reproduce | ★ | 따라하기 Step 2~4를 재현해 TaskFlow에 PR 1개를 연다 | 커밋 3개 이상이 Conventional Commits 형식이고, PR 본문이 템플릿 6개 섹션을 모두 채움. PR 링크 제출 |
| HW2 | 🛠 Apply | ★★ | 본인 팀(또는 TaskFlow)용 커밋 컨벤션 문서와 PR 템플릿을 정의하고, 두 Skill(`*-commit`, `*-pr`)을 두 도구 위치에 모두 배치한다 | `docs/conventions/git.md`(브랜치·커밋·PR 크기 규칙과 근거), `.github/pull_request_template.md`, `.claude/skills/`와 `.agents/skills/`의 Skill 파일, 이 규칙으로 연 PR 2개 이상 |
| HW3 | 🚀 Challenge | ★★★ | 규칙을 사람이 아니라 기계가 강제하게 만든다: 커밋 메시지 검사와 PR 크기 검사를 CI(GitHub Actions)에 추가하고, 위반 PR을 에이전트가 스스로 고치게 한다 | 워크플로 파일, 일부러 위반한 PR에서 CI가 실패한 로그, 에이전트가 그 실패를 보고 수정(메시지 정리 또는 PR 분할)한 뒤 CI가 통과한 로그 |

제출: `hw/03-3` 브랜치, `submissions/03-3.md` (템플릿: [docs/design/homework-template.md](../../docs/design/homework-template.md))

## ⚠️ 흔한 실수
- **세션 끝에 거대한 커밋 하나가 생긴다** → "커밋해줘"만 지시했다 → 커밋 계획을 먼저 받고(Step 2), Skill에 "의도가 둘 이상이면 분할 제안" 규칙을 넣는다.
- **Codex가 커밋이나 PR 생성을 못 한다** → `workspace-write`에서도 `.git`이 보호되고 네트워크가 기본 차단이다 → Codex에게는 메시지 파일과 명령을 만들게 하고 사람이 실행한다. 반복된다면 승인 요청에 응하는 흐름을 팀 규칙으로 정한다.
- **PR 본문이 그럴듯하지만 사실과 다르다** ("모든 테스트 통과" 등) → 에이전트가 실행하지 않은 검증을 추정해서 썼다 → PR Skill에 "직접 실행한 결과만, 미실행이면 미실행이라고 쓴다"를 넣고, 리뷰어는 CI 결과와 대조한다.
- **리뷰 지적을 고치면서 이력을 망가뜨린다** → 에이전트가 `--amend`나 force push를 썼다 → `AGENTS.md` 금지 목록과 Claude Code `deny` 규칙에 넣고, 수정은 새 커밋으로 한다. 정리가 필요하면 머지 방식(squash merge 등)으로 해결한다.
- **분할한 PR이 각자 깨진다** → 커밋 단위가 빌드 가능한 상태가 아니었다 → "각 커밋에서 `pnpm verify` 통과"를 규칙으로 두고, 분할 후 브랜치마다 검증을 돌린다.

## 🔗 참고 자료
- Claude Code: [Permissions](https://code.claude.com/docs/en/permissions), [Skills](https://code.claude.com/docs/en/skills), [Common workflows](https://code.claude.com/docs/en/common-workflows), [Code review](https://code.claude.com/docs/en/code-review)
- Codex: [Approvals & security](https://learn.chatgpt.com/docs/agent-approvals-security), [Skills](https://learn.chatgpt.com/docs/build-skills), [Rules](https://learn.chatgpt.com/docs/agent-configuration/rules), [Code review](https://learn.chatgpt.com/docs/code-review)
- 외부: [Conventional Commits](https://www.conventionalcommits.org/), [GitHub CLI `gh pr create`](https://cli.github.com/manual/gh_pr_create)
- 이 저장소: [도구 레퍼런스 §5, §7, §10](../../docs/reference/tool-reference.md), [Task 명세 템플릿](../../templates/task-spec.md)
- 관련 레슨: [03-2 에이전트와 TDD](./03-2-tdd-with-agents.md), [03-5 디버깅 협업](./03-5-debugging.md), [04-2 교차 리뷰](../04-quality-and-verification/04-2-cross-review.md), [05-1 병렬 에이전트](../05-scaling-up/05-1-parallel-agents.md), [05-4 CI/CD 통합](../05-scaling-up/05-4-ci-cd-integration.md)
- 프로젝트: [Project B — TaskFlow](../../projects/B-development-project/README.md)
