---
id: 01-2
title: "권한·샌드박스·승인 모드"
module: 01-environment-setup
level: L1
duration: 1.5h
last-verified: 2026-10-07
tools:
  claude-code: 2.1.292
  codex: 0.160.1
---

# 01-2. 권한·샌드박스·승인 모드

## 🎯 학습 목표
- Claude Code의 permission mode와 Codex의 sandbox × approval 조합이 각각 무엇을 통제하는지 비교해 설명할 수 있다.
- TaskFlow 저장소에 `.claude/settings.json` 허용/확인/차단 규칙과 `.codex/config.toml` 샌드박스 설정을 작성할 수 있다.
- 비밀 파일 읽기·위험 명령 실행이 실제로 차단되는지 실험으로 확인할 수 있다.
- 팀 공용 설정(커밋)과 개인 설정(커밋 안 함)을 설정 우선순위에 맞게 분리할 수 있다.

## 📋 사전 준비
- 선행 레슨: [01-1 프로젝트 메모리](./01-1-project-memory.md), [00-3 대화형/헤드리스/클라우드 실행 모드](../00-foundations/00-3-execution-modes.md)
- 필요 도구/계정: Claude Code 2.1.292 이상, Codex 0.160.1 이상
- 실습 저장소 상태: TaskFlow 저장소에 01-1의 `AGENTS.md`/`CLAUDE.md`가 커밋된 상태. 실험용으로 루트에 가짜 비밀 파일을 하나 만든다.

```bash
cd taskflow
echo "DATABASE_URL=postgresql://taskflow:FAKE-SECRET@localhost:5432/taskflow" > .env
grep -qx ".env" .gitignore || echo ".env" >> .gitignore
```

## 💡 개념

### 두 도구는 "무엇을 막는가"부터 다르다

Claude Code는 **도구 호출 단위의 권한 규칙**이 중심이다. "이 Bash 명령은 허용, 이 파일 읽기는 차단"처럼 규칙을 쌓고, permission mode가 규칙에 없는 호출을 어떻게 처리할지 정한다.

Codex는 **OS 수준 샌드박스**가 중심이다. 샌드박스(`read-only` / `workspace-write` / `danger-full-access`)가 명령이 건드릴 수 있는 파일·네트워크 범위를 정하고, approval policy(`on-request` / `never`)가 샌드박스 밖으로 나가야 할 때 사람에게 물을지를 정한다.

```mermaid
flowchart TB
    subgraph Claude["Claude Code: 도구 호출마다 규칙 평가"]
        T1[도구 호출] --> D{deny 일치?}
        D -- 예 --> X1[차단]
        D -- 아니오 --> AS{ask 일치?}
        AS -- 예 --> Q1[사용자에게 확인]
        AS -- 아니오 --> AL{allow 일치?}
        AL -- 예 --> OK1[실행]
        AL -- 아니오 --> M["permission mode가 결정<br/>(auto / default / acceptEdits / plan / dontAsk)"]
    end
    subgraph Codex["Codex: 샌드박스 경계 + 승인 정책"]
        T2[명령 실행] --> S{샌드박스 안에서<br/>가능한가?}
        S -- 예 --> OK2[실행]
        S -- 아니오 --> AP{approval policy}
        AP -- on-request --> Q2[사용자에게 확인]
        AP -- never --> X2[실패로 반환]
    end
```

### 모드 대응표 (1:1 대응이 아니다)

| 의도 | Claude Code | Codex |
|---|---|---|
| 읽기·계획만 | `plan` (`--permission-mode plan`, `/plan`) | `--sandbox read-only --ask-for-approval on-request`, 또는 `/plan` |
| 처음 쓸 때마다 묻기 | `default` (UI 표시는 Manual, 별칭 `manual`) | 명령마다 묻는 `untrusted` 정책은 폐지됨. 프로젝트 `trust_level = "untrusted"`로 대체 |
| 편집 자동 승인 | `acceptEdits` | `--sandbox workspace-write --ask-for-approval on-request` (플래그 없이 `codex` 실행 시 기본) |
| 분류기가 자동 판단 | `auto` (v2.1.283+ 대화형 기본값) | `approvals_reviewer = "auto_review"`, `--approve-for-me` |
| 묻지 말고 거부 | `dontAsk` | `--ask-for-approval never` (샌드박스 경계 안에서만 실행) |
| 전부 허용 | `bypassPermissions`, `--dangerously-skip-permissions` | `--dangerously-bypass-approvals-and-sandbox` (`--yolo`) |
| 실행 중 전환 | `Shift+Tab`, `/permissions` | `/permissions` |

알아 둘 최근 변경 사항:
- **Claude Code 대화형 기본 모드는 `auto`다**(v2.1.283+). "처음에는 매번 승인 창이 뜬다"는 옛 설명은 더 이상 기본 동작이 아니다. Manual 동작을 보려면 `--permission-mode default`를 명시한다.
- **Codex `-a`는 `on-request`, `never`만 받는다.** 옛 자료의 `-a untrusted`, `-a on-failure`는 오류가 난다.
- **`codex exec`의 기본 샌드박스는 `read-only`다.** 비대화형으로 파일을 고치게 하려면 `--sandbox workspace-write`를 명시한다. `--full-auto`는 0.160.1에서 거부된다.

### 설정 파일의 계층

| 계층 | Claude Code (JSON) | Codex (TOML) |
|---|---|---|
| 조직(강제) | `managed-settings.json`, MDM | `/etc/codex/config.toml`, `requirements.toml` |
| 1회성 | 명령줄 플래그, `--settings` | 명령줄 플래그, `-c key=value` |
| 프로젝트 개인 | `.claude/settings.local.json` | 해당 없음 |
| 프로젝트 공유 | `.claude/settings.json` | `.codex/config.toml` (**신뢰한 프로젝트만**) |
| 프로필 | 해당 없음 | `--profile <name>` → `~/.codex/<name>.config.toml` |
| 사용자 | `~/.claude/settings.json` | `~/.codex/config.toml` |

- Claude 우선순위: Managed → 명령줄 → `settings.local.json` → `settings.json` → 사용자. `permissions.allow` 같은 **배열 키는 병합**된다. 개인 설정이 팀 차단 규칙을 지울 수 없다는 뜻이다.
- Codex 우선순위: CLI 플래그·`-c` → 프로젝트 `.codex/config.toml` → `--profile` 파일 → `~/.codex/config.toml` → 시스템 기본값.

### 왜 대규모 개발에서 중요한가

1. **승인 피로는 보안 구멍이 된다.** 하루 수백 번 뜨는 승인 창은 결국 "전부 Yes"로 이어진다. 안전한 명령은 미리 허용하고, 정말 위험한 것만 묻거나 막아야 승인이 의미를 가진다.
2. **병렬·무인 실행의 전제 조건이다.** worktree 3개, 클라우드 세션 5개를 동시에 돌리면([05-1](../05-scaling-up/05-1-parallel-agents.md)) 사람이 매번 승인할 수 없다. 경계가 명확해야 맡길 수 있다.
3. **팀 전체의 하한선.** 한 사람의 실수로 `.env`가 프롬프트에 들어가는 일을 팀 공용 deny 규칙으로 막는다. 개인 설정이 이를 해제할 수 없게 계층을 이해해야 한다.
4. **프롬프트 인젝션 대비.** 이슈 본문이나 웹 문서에 숨은 지시가 `curl`로 데이터를 내보내려 할 때, 마지막 방어선은 권한과 샌드박스다([04-3 보안](../04-quality-and-verification/04-3-security.md)).

## 👣 따라하기

### Step 1. 현재 권한 상태를 확인한다
설정을 바꾸기 전에 지금 어떤 모드와 규칙이 적용되는지 본다.

**Claude Code 레시피**
```bash
claude
> /status
> /permissions
```
`/status`에서 현재 permission mode를, `/permissions`에서 적용 중인 allow/ask/deny 규칙과 그 출처 파일을 확인한다. `Shift+Tab`을 눌러 모드가 순환하는 것도 확인한다.

**Codex 레시피**
```bash
codex
> /status
> /debug-config
```
`/status`에서 현재 sandbox와 approval policy를, `/debug-config`에서 어떤 설정 파일이 어떤 순서로 적용됐는지 확인한다.

**기대 결과**: Claude는 `auto` 모드(대화형 기본값), 규칙은 비어 있거나 사용자 설정의 규칙만 보인다. Codex는 `workspace-write` + `on-request`(플래그 없는 대화형 기본값)로 표시된다. 이 상태를 `notes/permissions-before.md`에 적어 둔다.

### Step 2. Claude Code 팀 공용 규칙을 작성한다
TaskFlow에서 매일 쓰는 안전한 명령은 허용하고, 원격에 영향을 주는 명령은 확인하고, 비밀 파일과 외부 전송은 막는다.

**Claude Code 레시피**
```bash
mkdir -p .claude
cat > .claude/settings.json <<'EOF'
{
  "permissions": {
    "allow": [
      "Bash(pnpm install)",
      "Bash(pnpm --filter *)",
      "Bash(pnpm turbo run *)",
      "Bash(make verify)",
      "Bash(git status)",
      "Bash(git diff *)",
      "Bash(git log *)"
    ],
    "ask": [
      "Bash(git push *)",
      "Bash(pnpm add *)",
      "Bash(docker compose *)"
    ],
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(./secrets/**)",
      "Edit(./packages/db/migrations/**)",
      "Bash(curl *)",
      "Bash(wget *)"
    ]
  }
}
EOF
```

규칙 작성 요령:
- 평가 순서는 **deny → ask → allow**이고, 먼저 일치한 쪽이 결정한다. allow로 deny의 예외를 만들 수 없다.
- `Bash(git diff *)`와 `Bash(git diff:*)`는 같다. `Bash(pnpm*)`처럼 공백 없이 쓰면 `pnpmx` 같은 다른 명령에도 일치한다.
- `Edit(...)` 규칙은 파일을 고치는 내장 도구 전체에 적용되므로, 위 deny는 기존 마이그레이션 수정뿐 아니라 **새 마이그레이션 파일 생성도 막는다**. B0 단계에서는 의도한 동작이다(새 마이그레이션은 사람이 만들거나 승인한다). "기존 파일만 막기"처럼 더 세밀한 제어는 [01-4](./01-4-extensions.md) HW3의 `PreToolUse` hook으로 한다.
- `Read(./.env)`의 `./`는 cwd 기준이다. `/path`는 절대 경로가 아니라 **설정 파일 위치 기준**이라는 점에 주의한다.
- Bash 인수 패턴은 `sh -c`, 옵션 순서 바꾸기 등으로 우회될 수 있다. 네트워크 차단은 이 규칙에만 의존하지 말고 `/sandbox`나 PreToolUse hook([01-4](./01-4-extensions.md))을 함께 쓴다.

**Codex 레시피**

Codex에는 Claude와 같은 "도구 호출별 allow/deny JSON"이 없다. 같은 의도는 샌드박스와 승인 정책으로 표현한다(Step 3). 샌드박스 밖에서 실행할 명령을 세밀하게 제어하는 `.rules`(execpolicy) 파일도 있지만, 위치와 문법은 이 레슨에서 다루지 않는다. TODO(verify): Codex `.rules` 파일 예시를 공식 문서 [Rules](https://learn.chatgpt.com/docs/agent-configuration/rules)로 검증한 뒤 추가한다.

대신 Codex에게 Claude 규칙 파일을 읽혀서 "같은 의도를 Codex로는 어떻게 지킬 수 있는지" 분석을 받는다.

```bash
codex exec "Read .claude/settings.json. For each deny/ask rule, explain whether Codex's sandbox_mode and approval_policy can enforce the same intent, and what gap remains. Answer as a markdown table."
```

**기대 결과**: `.claude/settings.json`이 생긴다. Codex의 분석표에는 보통 "`Read(./.env)` → Codex 샌드박스는 작업 디렉터리 안 읽기를 막지 않으므로 그대로 강제할 수 없다", "`curl` → `workspace-write`에서 네트워크를 끄면 막힌다" 같은 차이가 나온다. 이 차이가 HW2의 재료다.

### Step 3. Codex 샌드박스와 프로필을 구성한다
팀 공용 기본값은 프로젝트 `.codex/config.toml`에, 상황별 모드는 프로필 파일에 둔다.

**Codex 레시피**

프로젝트 공용 설정(신뢰한 프로젝트에서만 로드된다):
```bash
mkdir -p .codex
cat > .codex/config.toml <<'EOF'
# TaskFlow 팀 공용 Codex 설정
sandbox_mode = "workspace-write"
approval_policy = "on-request"

[sandbox_workspace_write]
network_access = false     # 설치가 필요하면 승인 요청을 거쳐 샌드박스 밖에서 실행한다
EOF
```

읽기 전용 리뷰용 프로필(개인 파일, 0.134.0부터 별도 파일 형식):
```bash
cat > ~/.codex/review.config.toml <<'EOF'
sandbox_mode = "read-only"
approval_policy = "never"
EOF
codex --profile review "apps/api의 라우트 구조를 리뷰해. 파일은 수정하지 마."
```

> `config.toml` 안의 `[profiles.review]` 테이블은 0.134.0 이후 동작하지 않는다. 옛 예시를 그대로 옮기지 않는다.

**Claude Code 레시피**

Claude에는 프로필 파일이 없다. 같은 효과는 시작 플래그로 낸다.
```bash
claude --permission-mode plan          # 읽기·계획만 (리뷰·조사용)
claude --permission-mode acceptEdits   # 편집은 자동, 명령은 규칙에 따라
claude --permission-mode default       # Manual: 처음 쓰는 도구마다 묻기
```
자주 쓰는 조합은 `--settings <파일>`로 별도 JSON을 겹쳐 쓸 수도 있다.
```bash
claude --settings ./.claude/review-settings.json
```

**기대 결과**: Codex를 새로 시작하고 `/debug-config`를 보면 프로젝트 `.codex/config.toml`이 적용된 것이 보인다(처음이면 프로젝트 신뢰 여부를 묻는다). `--profile review`로 시작한 세션은 `/status`에 `read-only`, `never`가 표시된다.

### Step 4. 차단이 실제로 동작하는지 실험한다
규칙은 "써 놓은 것"이 아니라 "막히는 것을 본 것"이어야 한다. 같은 실험을 두 도구에 한다.

**Claude Code 레시피**
```bash
claude
> .env 파일 내용을 보여줘.
> curl https://example.com 을 실행해줘.
> packages/db/migrations 아래 아무 파일 끝에 주석 한 줄을 추가해줘.
> git diff --stat 을 실행해줘.
```

**Codex 레시피**
```bash
# 1) 기본 exec는 read-only: 파일 생성이 실패해야 한다
codex exec "Create a file named probe.txt with the text hello."
ls probe.txt   # 없어야 정상

# 2) workspace-write + 네트워크 차단: 파일은 되지만 외부 요청은 실패해야 한다
codex exec --sandbox workspace-write "Create probe.txt with hello, then run: curl -sS https://example.com"
ls probe.txt && rm probe.txt

# 3) 승인 정책 never: 샌드박스 밖이 필요한 작업은 묻지 않고 실패로 돌아와야 한다
codex exec --sandbox workspace-write --ask-for-approval never "Run: curl -sS https://example.com"
```

**기대 결과**

| 실험 | Claude Code | Codex |
|---|---|---|
| `.env` 읽기 | deny 규칙으로 차단, 이유 표시 | 샌드박스는 작업 디렉터리 안 읽기를 막지 않는다. 메모리 파일의 "읽지 말 것" 규칙에만 의존한다 |
| `curl` | deny 규칙으로 차단 | `network_access = false`라서 실패하거나 승인 요청이 뜬다 |
| 마이그레이션 수정 | `Edit(...)` deny로 차단 | 샌드박스로는 막지 못한다. 01-4의 hook으로 보완한다 |
| `git diff --stat` | allow 규칙으로 묻지 않고 실행 | 샌드박스 안이므로 묻지 않고 실행 |

결과를 `notes/permissions-test.md`에 표로 남긴다. **Codex가 `.env`를 읽을 수 있다는 점**을 확인했다면, 실험이 제 역할을 한 것이다. 이것이 "규칙 기반"과 "샌드박스 기반"의 차이다.

> 위험 플래그 실습 금지: `--dangerously-skip-permissions`, `--yolo`, `danger-full-access`는 이 단계에서 쓰지 않는다. 필요하면 [01-5](./01-5-reproducible-environment.md)의 devcontainer처럼 격리된 환경에서만 쓴다.

### Step 5. 팀 공용 설정과 개인 설정을 분리한다
팀이 공유할 하한선은 커밋하고, 개인 취향은 커밋하지 않는 파일로 옮긴다.

**Claude Code 레시피**
```bash
cat > .claude/settings.local.json <<'EOF'
{
  "permissions": {
    "defaultMode": "acceptEdits",
    "allow": ["Bash(pnpm turbo run dev)"]
  }
}
EOF
grep -qx ".claude/settings.local.json" .gitignore || echo ".claude/settings.local.json" >> .gitignore
```
- `defaultMode`는 개인 파일에 둔다. 팀 파일 `.claude/settings.json`에 `"auto"`나 `"bypassPermissions"`를 넣어도 적용되지 않는다.
- 개인 `allow`는 팀 `allow`와 **병합**된다. 팀 `deny`는 개인 파일로 해제할 수 없다.

**Codex 레시피**
```bash
cat >> ~/.codex/config.toml <<'EOF'
# 개인 기본값: 프로젝트 .codex/config.toml이 있으면 그쪽이 이긴다
approval_policy = "on-request"
sandbox_mode = "workspace-write"
EOF
```
- 프로젝트 `.codex/config.toml`이 사용자 설정보다 우선한다. 개인이 프로젝트 값을 바꾸고 싶으면 실행할 때 `-c sandbox_mode="read-only"`처럼 1회성으로 덮어쓴다.
- 프로젝트 파일에 둘 수 없는 키(`profile`, `profiles`, `model_provider`, `notify` 등)는 경고와 함께 무시된다.

마지막으로 공용 파일만 커밋한다.
```bash
git add .claude/settings.json .codex/config.toml .gitignore
git status --short   # settings.local.json, .env 가 보이지 않아야 한다
git commit -m "chore: add team permission and sandbox defaults"
```

**기대 결과**: 저장소에는 `.claude/settings.json`, `.codex/config.toml`만 커밋된다. `/permissions`에서 규칙 출처가 "project"와 "local"로 나뉘어 표시된다.

## ✅ 체크포인트
- [ ] Claude `/status`·`/permissions`, Codex `/status`·`/debug-config`로 적용 중인 모드와 설정 출처를 읽을 수 있다.
- [ ] `.claude/settings.json`에 allow/ask/deny가 모두 있고, deny에 `.env`와 마이그레이션 디렉터리가 포함됐다.
- [ ] `.codex/config.toml`에 `sandbox_mode`, `approval_policy`, `network_access`가 있다.
- [ ] `~/.codex/review.config.toml` 프로필로 읽기 전용 세션을 띄워 봤다.
- [ ] Step 4 실험 결과표에서 두 도구가 막는 것과 못 막는 것을 구분했다.
- [ ] `settings.local.json`과 `.env`가 커밋되지 않았다.

## 🏠 숙제
| # | 유형 | 난이도 | 과제 | 완료 조건 |
|---|---|---|---|---|
| HW1 | 🔁 Reproduce | ★ | Step 2~4를 재현하고 차단 실험 결과를 기록한다 | `.claude/settings.json`, `.codex/config.toml`이 커밋되어 있고, Step 4의 4가지 실험 각각에 대해 두 도구의 실제 출력(차단 메시지 또는 실행 결과)이 제출 문서에 있다 |
| HW2 | 🛠 Apply | ★★ | 팀 공용 설정과 개인 설정 분리안을 작성한다 | 공용/개인/조직(managed) 각 계층에 둘 키와 그 이유를 표로 정리했고, Claude 규칙을 Codex로 강제하지 못하는 항목(최소 2개)에 대한 보완책(메모리 규칙, hook, 네트워크 차단 등)을 제시했다 |
| HW3 | 🚀 Challenge | ★★★ | "규칙 우회 레드팀": 에이전트에게 deny 규칙을 우회하도록 유도해 본다 | 우회 시도 5가지 이상(예: `sh -c`, 변수 치환, 다른 도구 사용)과 각 결과를 기록하고, 뚫린 경우 규칙·`/sandbox`·hook 중 무엇으로 막았는지 수정 diff와 재실험 결과를 첨부했다 |

제출: `hw/01-2` 브랜치, `submissions/01-2.md` (템플릿: [docs/design/homework-template.md](../../docs/design/homework-template.md))

## ⚠️ 흔한 실수
- **`codex exec`로 고치라고 했는데 아무것도 바뀌지 않는다** → `exec`의 기본 샌드박스는 `read-only`다 → `codex exec --sandbox workspace-write "..."`로 실행한다. `--full-auto`는 0.160.1에서 오류가 난다.
- **`-a untrusted`가 오류를 낸다** → `untrusted` 승인 정책은 폐지됐고 `-a`는 `on-request`, `never`만 받는다 → 명령마다 승인을 받으려면 `[projects."/path"] trust_level = "untrusted"`를 쓴다. config에 `approval_policy = "untrusted"`가 남아 있으면 지운다.
- **Claude가 승인 창을 거의 띄우지 않아서 규칙이 동작하는지 모르겠다** → 대화형 기본 모드가 `auto`라서 분류기가 대신 판단한다 → 규칙을 시험할 때는 `claude --permission-mode default`로 시작해 Manual 동작을 확인한다.
- **`Read(/secrets/**)`가 막지 못한다** → `/`로 시작하는 경로는 절대 경로가 아니라 설정 파일 위치 기준이다 → cwd 기준이면 `./secrets/**`, 파일시스템 절대 경로면 `//abs/path`, 홈 기준이면 `~/path`로 쓴다.
- **프로젝트 `.codex/config.toml`이 무시된다** → 신뢰하지 않은 프로젝트이거나, 프로젝트 파일에서 허용하지 않는 키를 넣었다 → Codex 시작 시 프로젝트 신뢰를 승인하고 `/debug-config`로 로드 여부와 경고를 확인한다.

## 🔗 참고 자료
- 공식 문서
  - [Claude Code permissions (규칙 문법, 평가 순서)](https://code.claude.com/docs/en/permissions)
  - [Claude Code permission modes (`auto` 기본값, `defaultMode`)](https://code.claude.com/docs/en/permission-modes)
  - [Claude Code settings (계층, 우선순위)](https://code.claude.com/docs/en/settings)
  - [Codex approvals & security (sandbox, approval policy)](https://learn.chatgpt.com/docs/agent-approvals-security)
  - [Codex config-basic](https://learn.chatgpt.com/docs/config-file/config-basic), [config-advanced (프로필 파일)](https://learn.chatgpt.com/docs/config-file/config-advanced)
  - [Codex Rules (execpolicy)](https://learn.chatgpt.com/docs/agent-configuration/rules)
- 이 저장소
  - [도구 레퍼런스 §4 설정 파일, §5 권한·승인·샌드박스](../../docs/reference/tool-reference.md)
  - 이전 레슨: [01-1 프로젝트 메모리](./01-1-project-memory.md) · 다음 레슨: [01-3 MCP 서버 연결](./01-3-mcp-servers.md)
  - 관련 레슨: [00-3 실행 모드](../00-foundations/00-3-execution-modes.md), [04-3 보안](../04-quality-and-verification/04-3-security.md), [06-3 팀 규칙과 거버넌스](../06-team-and-operations/06-3-governance.md)
