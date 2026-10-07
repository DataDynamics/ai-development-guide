---
title: Claude Code · Codex 도구 레퍼런스 (검증된 팩트 시트)
last-verified: 2026-10-07
tools:
  claude-code: 2.1.292      # 로컬 `claude --version`, npm @anthropic-ai/claude-code 최신과 일치
  codex: 0.160.1            # npm @openai/codex 최신을 스크래치 디렉터리에 설치해 `codex --version`으로 확인
  claude-code-action: v1    # 문서의 워크플로 예시 기준. 최신 릴리스 태그는 TODO(verify)
  codex-action: v1          # 문서의 워크플로 예시 기준. 최신 릴리스 태그는 TODO(verify)
---

# Claude Code · Codex 도구 레퍼런스

> 레슨 작성자가 명령어를 복사해 쓰는 **단일 기준 문서(single source of truth)** 다.
> 넓게 다루기보다 정확하게 다루는 것을 우선한다. 이 문서에 없는 옵션은 레슨에 쓰기 전에 직접 확인하고, 확인하지 못했으면 `TODO(verify)`로 표시한다.

**검증 방법**

- Claude Code: 로컬에 설치된 `claude` 2.1.292의 `--help` 출력과 공식 문서 [code.claude.com/docs](https://code.claude.com/docs/en/overview)(각 페이지의 `.md` 원문)를 대조했다.
- Codex: `@openai/codex` 0.160.1을 `/tmp` 아래 스크래치 디렉터리에 설치해 `--help` 출력을 확인하고 공식 문서와 대조했다. `developers.openai.com/codex/...` 문서 주소는 현재 **`learn.chatgpt.com/docs/...`로 308 리다이렉트**되므로 이 문서는 리다이렉트된 뒤의 주소를 출처로 적는다.
- `--help`와 문서가 서로 다르면 둘 다 적고, 로컬 바이너리에서 실제로 동작한 쪽을 기준으로 삼는다.

---

## 1. 설치 · 인증 · 버전

| 항목 | Claude Code | Codex |
|---|---|---|
| 권장 설치 (macOS/Linux) | `curl -fsSL https://claude.ai/install.sh \| bash` | `curl -fsSL https://chatgpt.com/codex/install.sh \| sh` |
| 패키지 매니저 | `brew install --cask claude-code`, `winget install Anthropic.ClaudeCode`, `npm install -g @anthropic-ai/claude-code` (Node.js 22+) | `npm install -g @openai/codex`, `brew install --cask codex` |
| 로그인 | `claude` 실행 후 브라우저 안내, 세션 안에서 `/login`, CLI는 `claude auth login` / `claude auth status` | `codex login` (브라우저, 기본), `codex login --device-auth`, `codex login status` |
| API 키 | `ANTHROPIC_API_KEY` 환경 변수 (처음 한 번 사용 승인을 묻는다) | `printenv OPENAI_API_KEY \| codex login --with-api-key`, 일회성은 `CODEX_API_KEY=... codex exec ...` |
| CI용 장기 토큰 | `claude setup-token` (Claude 구독 필요) | `codex login --with-access-token` (stdin으로 입력) |
| 버전 / 업데이트 | `claude --version`, `claude update`, 진단 `claude doctor` | `codex --version`, `codex update`, 진단 `codex doctor` |
| 자격 증명 저장 | `~/.claude.json`에 로그인 세션 등을 저장한다 | `~/.codex/auth.json`(평문) 또는 OS 자격 증명 저장소 |

출처: [Claude setup](https://code.claude.com/docs/en/setup), [Claude settings(`~/.claude.json`)](https://code.claude.com/docs/en/settings), [Codex README](https://github.com/openai/codex/blob/main/README.md), [Codex auth](https://learn.chatgpt.com/docs/auth), [Codex non-interactive](https://learn.chatgpt.com/docs/non-interactive-mode)

- Homebrew·WinGet으로 설치한 Claude Code는 자동 업데이트되지 않는다. `brew upgrade claude-code`처럼 직접 올린다 ([setup](https://code.claude.com/docs/en/setup)).
- Codex는 ChatGPT 요금제(Plus, Pro, Business, Edu, Enterprise)로 로그인하는 방식을 권장한다 ([README](https://github.com/openai/codex/blob/main/README.md)).

---

## 2. 대화형 · 비대화형(헤드리스) · 세션 이어가기

| 항목 | Claude Code | Codex |
|---|---|---|
| 대화형 시작 | `claude` / `claude "프롬프트"` | `codex` / `codex "프롬프트"` |
| 비대화형 | `claude -p "프롬프트"` (`--print`) | `codex exec "프롬프트"` (별칭 `codex e`) |
| 표준 입력 | `cat logs.txt \| claude -p "explain"` | 프롬프트를 생략하거나 `-`를 주면 stdin을 읽는다. 프롬프트와 stdin을 함께 주면 stdin이 `<stdin>` 블록으로 붙는다 |
| 출력 형식 | `--output-format text\|json\|stream-json` (`-p`에서만 동작) | 기본: 진행 상황은 stderr, 최종 메시지만 stdout. `--json`이면 stdout이 JSONL 이벤트 스트림이 된다 |
| 구조화 출력 | `--json-schema '<JSON Schema>'` → 결과의 `structured_output` | `--output-schema <FILE>` |
| 최종 메시지 파일 저장 | (jq로 `.result` 추출) | `-o, --output-last-message <FILE>` |
| 가장 최근 세션 이어가기 | `claude -c` (`--continue`), `claude -c -p "..."` | `codex resume --last`, `codex exec resume --last "..."` |
| 특정 세션 재개 | `claude -r "<세션 ID 또는 이름>" "..."` | `codex resume <SESSION_ID>`, `codex exec resume <SESSION_ID> "..."` |
| 세션 분기 | `--fork-session` (`-r`/`-c`와 함께) | `codex fork` (`--last` 지원), `codex exec fork` |
| 세션 저장 안 함 | `--no-session-persistence` (`-p`에서만) | `--ephemeral` |
| 빠른 스크립트 모드 | `--bare`: hooks, skills, plugins, MCP, auto memory, CLAUDE.md 자동 탐색을 건너뛴다 | `--ignore-user-config`, `--ignore-rules` |
| 예산 상한 | `--max-budget-usd <amount>` (`-p`에서만) | TODO(verify) |
| Git 저장소 밖 실행 | 제한 없음 | `codex exec`는 Git 저장소가 필요하다. 밖에서는 `--skip-git-repo-check` |

출처: 로컬 `claude --help`, `codex exec --help`, [Claude CLI reference](https://code.claude.com/docs/en/cli-reference), [Claude headless](https://code.claude.com/docs/en/headless), [Codex non-interactive](https://learn.chatgpt.com/docs/non-interactive-mode)

```bash
# Claude Code: JSON 결과에서 본문만 추출
claude -p "Summarize this project" --output-format json | jq -r '.result'
# 토큰 스트리밍
claude -p "Explain recursion" --output-format stream-json --verbose --include-partial-messages
# CI에서 권장: --bare + 최소 도구
claude --bare -p "Summarize README.md" --allowedTools "Read"

# Codex: 최종 결과만 파일로
codex exec "generate release notes for the last 10 commits" | tee release-notes.md
# JSONL 이벤트 (thread.started, turn.started, turn.completed, item.* ...)
codex exec --json "summarize the repo structure" | jq
# 2단계 파이프라인
codex exec "review the change for race conditions"
codex exec resume --last "fix the race conditions you found"
```

- `claude -p`는 workspace trust 대화상자를 건너뛰고, `--bare` 없이 실행하면 프로젝트의 `.claude/settings.json` hooks와 `.mcp.json` 서버를 그대로 실행한다. 신뢰할 수 있는 디렉터리에서만 쓴다 ([headless](https://code.claude.com/docs/en/headless)).
- 문서는 `--bare`를 스크립트·SDK 호출 권장 모드로 소개하며 "앞으로 `-p`의 기본값이 될 것"이라고 예고한다 ([headless](https://code.claude.com/docs/en/headless)).
- `--output-format json` 결과에는 `total_cost_usd`가 들어 있다 ([headless](https://code.claude.com/docs/en/headless)).
- `codex exec`의 기본 샌드박스는 **read-only**다. 파일을 고치게 하려면 `--sandbox workspace-write`를 명시한다 ([non-interactive](https://learn.chatgpt.com/docs/non-interactive-mode)).

---

## 3. 프로젝트 메모리: `CLAUDE.md` / `AGENTS.md`

| 항목 | Claude Code (`CLAUDE.md`) | Codex (`AGENTS.md`) |
|---|---|---|
| 전역(사용자) | `~/.claude/CLAUDE.md` | `~/.codex/AGENTS.md` (`AGENTS.override.md`가 있으면 그것을 대신 읽는다. `CODEX_HOME`으로 경로 변경) |
| 조직 관리 | Linux/WSL `/etc/claude-code/CLAUDE.md`, macOS `/Library/Application Support/ClaudeCode/CLAUDE.md` | `requirements.toml` 등 관리 구성 (메모리 파일 자체는 아니다) |
| 프로젝트 | `./CLAUDE.md` 또는 `./.claude/CLAUDE.md` | 프로젝트 루트(보통 Git 루트)부터 cwd까지 각 디렉터리의 `AGENTS.override.md` → `AGENTS.md` → `project_doc_fallback_filenames` 순으로, 디렉터리당 최대 1개 |
| 개인(커밋 안 함) | `./CLAUDE.local.md` (`.gitignore`에 추가) | 해당 없음 (전역 override 사용) |
| 로드 방식 | cwd와 그 상위 디렉터리의 파일은 시작할 때 로드한다. 하위 디렉터리 파일은 Claude가 그 안의 파일을 Read/Write/Edit할 때 로드한다. 덮어쓰지 않고 **연결(concatenate)** 한다 | 루트부터 아래로 연결하며, cwd에 가까운 파일이 뒤에 온다. 합계가 `project_doc_max_bytes`(기본 32 KiB)에 이르면 거기서 멈춘다 |
| import | `@path/to/file` (상대 경로는 그 파일 기준, 최대 4단계 재귀, 코드 스팬 안은 무시) | 공식 문서에 import 문법이 없다 |
| 경로별 규칙 | `.claude/rules/*.md` + front matter `paths:` glob | 하위 디렉터리에 `AGENTS.md` / `AGENTS.override.md`를 둔다 |
| 생성 명령 | `/init` (기존 파일이 있으면 덮어쓰지 않고 개선안을 낸다. `CLAUDE_CODE_NEW_INIT=1`이면 대화형 흐름) | `/init` (cwd에 `AGENTS.md` 뼈대를 만든다) |
| 편집·확인 | `/memory` (파일 목록, auto memory 토글), `/context`의 **Memory files** | 시작 시 한 번 체인을 구성한다. `codex "List the instruction sources you loaded."`로 확인하는 방법을 문서가 제시한다 |
| 자동 메모리 | auto memory: `~/.claude/projects/<project>/memory/`, 세션마다 처음 200줄 또는 25KB를 로드한다 | `memories` 기능 플래그(실험적, 기본값 false), `/memories` |

출처: [Claude memory](https://code.claude.com/docs/en/memory), [Codex AGENTS.md](https://learn.chatgpt.com/docs/agent-configuration/agents-md), [Codex config-advanced](https://learn.chatgpt.com/docs/config-file/config-advanced), [Codex slash commands](https://learn.chatgpt.com/docs/developer-commands?surface=cli), 로컬 `codex features list`

### 3.1 Claude Code는 `AGENTS.md`를 직접 읽는가? — **읽는다 (v2.1.277 이상, 조건부)**

공식 문서 [memory#agents-md](https://code.claude.com/docs/en/memory)에서 확인한 내용이다.

- 기본값(`claude-md-or-agents-md`): cwd와 그 상위에 `CLAUDE.md`, `.claude/CLAUDE.md`, `CLAUDE.local.md`가 **하나도 없을 때만** `AGENTS.md`(및 `.claude/AGENTS.md`)를 읽는다. 하나라도 있으면 `CLAUDE.md` 계열만 읽는다.
- `~/.claude/CLAUDE.md`, 조직 관리 `CLAUDE.md`, `.claude/rules/`는 이 판정에 포함되지 않는다.
- `/config` → **Project instructions**를 `claude-md-and-agents-md`로 바꾸면 둘 다 읽는다. 이 값은 사용자 설정(`~/.claude/settings.json`)의 `pluginConfigs."cc-plugin-agents-md@builtin".options.instructionFiles`로도 지정할 수 있다. 프로젝트·로컬 설정에서는 무시한다.
- 읽지 못하는 경우: 내장 플러그인을 비활성화했을 때, v2.1.276 이하에서 업그레이드한 직후 첫 세션 등이다. v2.1.281 이전에는 Bedrock이나 텔레메트리를 끈 세션에서도 읽지 못했다.
- **이 저장소의 방식**(`CLAUDE.md`에 `@AGENTS.md` import)은 문서가 권장하는 호환 방식이다. 어떤 설정값에서도 두 번 읽히지 않는다.

```markdown
<!-- CLAUDE.md -->
@AGENTS.md

## Claude Code 전용
- (Claude에게만 필요한 지시)
```

- 반대 방향: Codex는 `CLAUDE.md`를 기본으로 읽지 않는다. `~/.codex/config.toml`에 `project_doc_fallback_filenames`를 지정하면 `AGENTS.md`가 없는 디렉터리에서 대체 파일명을 시도한다 ([Codex AGENTS.md](https://learn.chatgpt.com/docs/agent-configuration/agents-md)).
- 이전(migration): Claude는 `/import codex`(CLI는 `claude import codex --dry-run`), Codex는 `/import`(Claude Code 설정 가져오기)를 제공한다 ([Claude commands](https://code.claude.com/docs/en/commands), [Codex commands](https://learn.chatgpt.com/docs/developer-commands?surface=cli)).

---

## 4. 설정 파일

| 항목 | Claude Code | Codex |
|---|---|---|
| 형식 | JSON | TOML |
| 사용자 | `~/.claude/settings.json` | `~/.codex/config.toml` |
| 프로젝트(공유) | `.claude/settings.json` | `.codex/config.toml` (**신뢰한 프로젝트만**. 루트부터 cwd까지 모두 로드하며 가까운 쪽이 이긴다) |
| 프로젝트(개인) | `.claude/settings.local.json` (Claude가 만들면 git 제외 목록에 추가한다) | 해당 없음 |
| 조직 | `managed-settings.json`, MDM, claude.ai 콘솔 | `/etc/codex/config.toml`, 클라우드 관리 기본값, `requirements.toml`(강제) |
| 1회성 덮어쓰기 | `--settings <file-or-json>`, 개별 플래그 | `-c key=value` (값은 TOML로 해석), 개별 플래그 |
| 프로필 | 해당 없음 | `--profile <name>` → `~/.codex/<name>.config.toml`을 겹친다 |
| 설정 UI | `/config`, `/status`, `/permissions` | `/status`, `/debug-config`, `/permissions` |

**Claude Code 우선순위(높음 → 낮음)**: Managed → 명령줄 인수 → `.claude/settings.local.json` → `.claude/settings.json` → `~/.claude/settings.json`. `permissions.allow` 같은 **배열 키는 덮어쓰지 않고 병합**한다. MCP 서버와 로그인 세션은 별도 파일인 `~/.claude.json`에 저장한다 ([settings](https://code.claude.com/docs/en/settings)).

**Codex 우선순위(높음 → 낮음)**: CLI 플래그·`--config` → 프로젝트 `.codex/config.toml` → `--profile` 파일 → `~/.codex/config.toml` → 클라우드 관리 기본값 → `/etc/codex/config.toml` → 내장 기본값 ([config-basic](https://learn.chatgpt.com/docs/config-file/config-basic)).

```toml
# ~/.codex/config.toml — 자주 쓰는 키 (출처: config-basic, agent-approvals-security, extend/mcp)
model = "<사용 가능한 모델 ID>"
model_reasoning_effort = "medium"
approval_policy = "on-request"        # 0.160.1 CLI 기준 on-request | never (+ granular 테이블)
sandbox_mode = "workspace-write"      # read-only | workspace-write | danger-full-access
web_search = "cached"                 # cached(기본) | indexed | live | disabled

[sandbox_workspace_write]
network_access = true                 # workspace-write에서 네트워크 허용(선택)

[mcp_servers.context7]
command = "npx"
args = ["-y", "@upstash/context7-mcp"]
```

- **프로필 형식이 바뀌었다.** Codex 0.134.0부터 `--profile`은 `config.toml` 안의 `[profiles.<name>]` 테이블을 읽지 않고, 최상위 `profile = "..."` 선택자도 지원하지 않는다. `~/.codex/<name>.config.toml` 파일에 최상위 키로 작성한다 ([config-advanced#profiles](https://learn.chatgpt.com/docs/config-file/config-advanced)). 로컬 `codex --help`에도 `-p, --profile <CONFIG_PROFILE_V2>  Layer $CODEX_HOME/<name>.config.toml ...`이라고 나온다.
- 프로젝트 `.codex/config.toml`에서는 `model_provider`, `model_providers`, `notify`, `profile`, `profiles`, `otel`, `openai_base_url` 등을 무시하고 경고를 출력한다 ([config-advanced](https://learn.chatgpt.com/docs/config-file/config-advanced)).
- Codex는 이름 붙은 권한 프로필(`:read-only`, `:workspace`, `:danger-full-access`, 사용자 정의 `[permissions.<name>]` + `default_permissions`)도 지원한다. 세부 문법은 TODO(verify) ([config-basic](https://learn.chatgpt.com/docs/config-file/config-basic)).

---

## 5. 권한 · 승인 모드 · 샌드박스

### 5.1 모드 비교

| 의도 | Claude Code (permission mode) | Codex (sandbox × approval) |
|---|---|---|
| 읽기·계획만 | `plan` (`/plan`, `--permission-mode plan`) | `--sandbox read-only --ask-for-approval on-request`, 또는 `/plan` |
| 처음 쓸 때마다 묻기 | `default` (UI 표시는 Manual, 별칭 `manual`) | (명령마다 묻는 `untrusted` 정책은 폐지됨. 아래 주의 참고) |
| 편집 자동 승인 | `acceptEdits` | `--sandbox workspace-write --ask-for-approval on-request` (Auto 프리셋, 플래그 없이 `codex` 실행 시 기본) |
| 분류기/리뷰어 자동 판단 | `auto` (백그라운드 분류기가 검사) | `approvals_reviewer = "auto_review"` (`--approve-for-me` 플래그도 있음) |
| 묻지 말고 거부 | `dontAsk` (허용 규칙에 없는 것은 자동 거부) | `--ask-for-approval never` (샌드박스 경계 안에서만 실행) |
| 전부 허용 | `bypassPermissions` / `--dangerously-skip-permissions` | `--dangerously-bypass-approvals-and-sandbox` (별칭 `--yolo`) |
| 실행 중 전환 | `Shift+Tab` 순환, `/permissions` | `/permissions` |

출처: [Claude permissions](https://code.claude.com/docs/en/permissions), [Claude permission modes](https://code.claude.com/docs/en/permission-modes), 로컬 `claude --help`(`--permission-mode` 선택지: `acceptEdits, auto, bypassPermissions, manual, dontAsk, plan`), [Codex approvals & security](https://learn.chatgpt.com/docs/agent-approvals-security), 로컬 `codex --help`

- **Claude Code 기본 모드가 바뀌었다.** v2.1.283부터 대화형 터미널·VS Code 세션의 내장 시작 모드는 `auto`다. 이전 버전에서는 Pro·Max·Team 요금제에서만 `auto`였다. `claude -p`는 상황에 따라 `default` 또는 `auto`로 시작한다. `defaultMode` 설정으로 바꿀 수 있지만, `.claude/settings.json`에 `"auto"`나 `"bypassPermissions"`를 넣어도 적용되지 않는다 ([permission-modes](https://code.claude.com/docs/en/permission-modes)).
- **Codex `untrusted` 승인 정책은 폐지됐다.** `approval_policy = "untrusted"`가 남아 있으면 클라이언트가 시작되지 않을 수 있다. 명령마다 승인을 받으려면 `[projects."/path"] trust_level = "untrusted"`를 쓴다. 0.160.1의 `-a`는 `on-request`, `never`만 받는다(`on-failure`, `untrusted`를 주면 오류). 로컬에서 확인했다.
- **`--full-auto`**: 문서는 "`codex exec --full-auto`는 경고를 출력하는 deprecated 호환 경로로 남아 있다"고 적는다. 하지만 로컬 0.160.1에서는 `codex --full-auto`와 `codex exec --full-auto`가 모두 `unexpected argument` 오류를 낸다. 레슨에서는 **`--sandbox workspace-write`를 명시**한다.
- Codex `workspace-write`에서도 `.git`, `.agents`, `.codex`는 읽기 전용으로 보호한다 ([approvals & security](https://learn.chatgpt.com/docs/agent-approvals-security)).

### 5.2 Claude Code 허용/거부 규칙 문법

```json
{
  "permissions": {
    "defaultMode": "acceptEdits",
    "allow": ["Bash(npm run *)", "Bash(git diff *)", "WebFetch(domain:github.com)", "mcp__github__get_*"],
    "ask":   ["Bash(git push *)"],
    "deny":  ["Read(./.env)", "Read(./secrets/**)", "Bash(curl *)"]
  }
}
```

- 평가 순서는 **deny → ask → allow**이고, 먼저 일치한 쪽이 결정한다. allow 규칙으로 deny에 예외를 만들 수 없다.
- `Bash(ls *)`와 `Bash(ls:*)`는 같다. `:*`는 패턴 **끝에서만** 인식한다. `Bash(ls*)`는 `lsof`에도 일치한다.
- Read/Edit 경로: `//abs/path`(파일시스템 루트), `~/path`(홈), `/path`(**설정 파일 위치 기준**, 절대 경로가 아니다), `path` 또는 `./path`(cwd 기준). gitignore 패턴 문법을 쓴다.
- MCP: `mcp__<server>`, `mcp__<server>__*`, `mcp__<server>__<tool>`.
- Bash 인수를 제한하는 패턴은 우회하기 쉽다(옵션 순서, `sh -c`, 변수 등). 네트워크 제어에는 `/sandbox`(샌드박스 모드)나 PreToolUse hook을 함께 쓰도록 문서가 권장한다.

출처: [Claude permissions](https://code.claude.com/docs/en/permissions)

### 5.3 Codex 명령 규칙

Codex는 샌드박스 밖에서 실행할 수 있는 명령을 `.rules` 파일(execpolicy)로 제어한다. 위치와 문법은 [Rules](https://learn.chatgpt.com/docs/agent-configuration/rules)를 참고하고, 레슨에 쓰기 전에 예시를 직접 검증한다. TODO(verify)

---

## 6. MCP

| 항목 | Claude Code | Codex |
|---|---|---|
| stdio 추가 | `claude mcp add <name> -- <command> [args...]` | `codex mcp add <name> -- <command>...` |
| 환경 변수 | `-e, --env KEY=value` | `--env KEY=VALUE` (stdio만) |
| HTTP 추가 | `claude mcp add --transport http <name> <url>` | `codex mcp add <name> --url <URL>` (streamable HTTP) |
| 헤더/토큰 | `-H, --header "Authorization: Bearer ..."` | `--bearer-token-env-var <ENV_VAR>` |
| 범위(scope) | `-s, --scope local\|user\|project` (기본 `local`) | 사용자 `~/.codex/config.toml` / 프로젝트 `.codex/config.toml`(신뢰한 프로젝트만) |
| 저장 위치 | local·user → `~/.claude.json`, project → 저장소 루트 `.mcp.json` | `[mcp_servers.<name>]` 테이블 |
| 목록/조회/삭제 | `claude mcp list`, `get`, `remove` | `codex mcp list`, `get`, `remove` |
| OAuth | `claude mcp login <name>` / 세션 안 `/mcp` | `codex mcp login <name>` |
| JSON으로 추가 | `claude mcp add-json <name> '<json>'` | (TOML 직접 편집) |
| 세션 안 상태 | `/mcp` | `/mcp` (`/mcp verbose`) |
| 기타 | SSE는 deprecated. 프로젝트 `.mcp.json` 서버는 대화형 세션에서 처음 쓸 때 승인을 받는다. 승인 초기화는 `claude mcp reset-project-choices` | `enabled`, `required`, `enabled_tools`, `disabled_tools`, `startup_timeout_sec`(기본 10), `tool_timeout_sec`(기본 60) |

출처: 로컬 `claude mcp add --help`, [Claude MCP](https://code.claude.com/docs/en/mcp), 로컬 `codex mcp add --help`, [Codex MCP](https://learn.chatgpt.com/docs/extend/mcp)

```bash
# Claude Code
claude mcp add --transport http sentry https://mcp.sentry.dev/mcp
claude mcp add --transport http shared-server --scope project https://example.com/mcp   # → .mcp.json
claude mcp add --env KEY=value --transport stdio myserver -- python server.py --port 8080

# Codex
codex mcp add context7 -- npx -y @upstash/context7-mcp
codex mcp add docs --url https://example.com/mcp --bearer-token-env-var DOCS_TOKEN
```

`.mcp.json`(Claude Code 프로젝트 범위)은 `${VAR}`, `${VAR:-default}` 확장을 지원한다.

```json
{ "mcpServers": { "api-server": { "type": "http", "url": "${API_BASE_URL:-https://api.example.com}/mcp",
  "headers": { "Authorization": "Bearer ${MY_API_KEY}" } } } }
```

- Claude Code의 **`--` 규칙**: 서버 명령 앞에 `--`가 없으면 `--port` 같은 서버 플래그를 Claude가 자기 옵션으로 해석한다. `--env` 바로 뒤에 서버 이름을 쓰면 이름을 또 하나의 `KEY=value`로 읽으므로, 둘 사이에 다른 옵션(`--transport` 등)을 둔다 ([Claude MCP](https://code.claude.com/docs/en/mcp)).
- `.mcp.json`의 원격 `url`·`headers`에서는 `ANTHROPIC_API_KEY` 같은 자격 증명 변수를 빈 값으로 읽는다. 자기만의 변수 이름을 쓴다 ([Claude MCP](https://code.claude.com/docs/en/mcp)).

---

## 7. 확장 기능

| 항목 | Claude Code | Codex |
|---|---|---|
| 커스텀 명령 | `.claude/commands/<name>.md` → `/name` (구형식이지만 계속 동작). **skills로 통합됐다** | `~/.codex/prompts/<name>.md` → `/prompts:<name>` (**deprecated**, 저장소로 공유 불가) |
| Skills | `.claude/skills/<name>/SKILL.md` (프로젝트), `~/.claude/skills/<name>/SKILL.md` (개인), 플러그인 `skills/` | `.agents/skills/<name>/SKILL.md` (cwd부터 저장소 루트까지 탐색), `$HOME/.agents/skills`, `/etc/codex/skills` |
| Skill 호출 | `/skill-name` 또는 설명이 맞으면 자동 호출 | `/skills` 또는 `$skill-name` 멘션, 자동 호출. 생성 도우미 `$skill-creator` |
| Subagents | `.claude/agents/<name>.md` (front matter `name`, `description`, `tools`, `model`, `isolation` 등), `~/.claude/agents/`, `--agents '<json>'` | `.codex/agents/<name>.toml`, `~/.codex/agents/` (`name`, `description`, `developer_instructions`, `model`, `sandbox_mode` 등), 전역 `[agents]` |
| Hooks 위치 | `settings.json`의 `hooks` 키, 플러그인 `hooks/hooks.json`, skill/agent front matter | `~/.codex/hooks.json`, `<repo>/.codex/hooks.json`, `config.toml`의 인라인 `[hooks]`, 플러그인 `hooks/hooks.json` |
| Hooks 신뢰 | Codex 같은 해시 기반 신뢰 절차는 문서에서 확인하지 못했다. `claude -p`는 신뢰하지 않은 폴더에서도 프로젝트 hooks를 실행한다 ([headless](https://code.claude.com/docs/en/headless)). TODO(verify) | 관리형이 아닌 hook은 `/hooks`에서 검토·신뢰해야 실행된다(해시 기준). `--dangerously-bypass-hook-trust`로 건너뛸 수 있다 |
| 플러그인 | `/plugin`, `claude plugin install <plugin>@<marketplace>`, `claude plugin marketplace add <source>` | `/plugins`, `codex plugin add`, `codex plugin marketplace` |

출처: [Claude skills](https://code.claude.com/docs/en/skills), [Claude sub-agents](https://code.claude.com/docs/en/sub-agents), [Claude hooks](https://code.claude.com/docs/en/hooks), 로컬 `claude plugin --help`, [Codex custom prompts](https://learn.chatgpt.com/docs/custom-prompts), [Codex skills](https://learn.chatgpt.com/docs/build-skills), [Codex subagents](https://learn.chatgpt.com/docs/agent-configuration/subagents), [Codex hooks](https://learn.chatgpt.com/docs/hooks), 로컬 `codex plugin --help`

### 7.1 Skill 예시 (두 도구 모두 `name`·`description` front matter를 쓴다)

```markdown
---
name: summarize-changes
description: Summarize uncommitted changes. Use when the user asks what changed.
disable-model-invocation: true   # Claude Code 전용: /이름으로만 호출
allowed-tools: Read Grep         # Claude Code 전용: 호출한 턴 동안 묻지 않고 허용
---
현재 변경 사항을 요약한다. 인수: $ARGUMENTS
```

- Claude Code는 skill과 같은 이름의 `.claude/commands/` 파일이 있으면 skill을 우선한다. 개인(`~/.claude/skills`)이 프로젝트보다 우선한다 ([skills](https://code.claude.com/docs/en/skills)).
- 클라우드 세션과 routine은 `~/.claude/skills/`를 읽지 않는다. 팀이 쓸 skill은 저장소의 `.claude/skills/`에 커밋한다 ([skills](https://code.claude.com/docs/en/skills)).
- Claude Code의 `/agents`는 v2.1.198부터 대화형 마법사를 열지 않고 안내 문구만 출력한다. subagent는 파일을 직접 쓰거나 Claude에게 만들어 달라고 한다 ([commands](https://code.claude.com/docs/en/commands)).

### 7.2 Hooks

**Claude Code 이벤트**(문서 표 기준): `SessionStart`, `Setup`, `UserPromptSubmit`, `UserPromptExpansion`, `PreToolUse`, `PermissionRequest`, `PermissionDenied`, `PostToolUse`, `PostToolUseFailure`, `PostToolBatch`, `Notification`, `MessageDisplay`, `SubagentStart`, `SubagentStop`, `TaskCreated`, `TaskCompleted`, `Stop`, `StopFailure`, `TeammateIdle`, `InstructionsLoaded`, `ConfigChange`, `CwdChanged`, `DirectoryAdded`, `FileChanged`, `WorktreeCreate`, `WorktreeRemove`, `PreCompact`, `PostCompact`, `PreModelSwitch`, `PostModelSwitch`, `Elicitation`, `ElicitationResult`, `SessionEnd`. 핸들러 타입은 `command`, `http`, `mcp_tool`, `prompt`, `agent`다. `PreToolUse`에서 exit code 2를 반환하면 도구 호출을 막는다 ([hooks](https://code.claude.com/docs/en/hooks)).

**Codex 이벤트**: `PreToolUse`, `PermissionRequest`, `PostToolUse`, `PreCompact`, `PostCompact`, `UserPromptSubmit`, `SubagentStop`, `Stop`, `Interrupt`, `SessionStart`, `SubagentStart`, `SessionEnd` ([Codex hooks](https://learn.chatgpt.com/docs/hooks)). `hooks` 기능 플래그는 stable이고 기본값 true다.

두 도구의 JSON 구조는 같은 모양이다(`이벤트 → [{matcher, hooks:[{type:"command", command}]}]`).

Claude Code (`.claude/settings.json`):

```json
{ "hooks": { "PostToolUse": [ { "matcher": "Write|Edit",
  "hooks": [ { "type": "command", "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/lint.sh" } ] } ] } }
```

Codex (`<repo>/.codex/hooks.json`):

```json
{ "hooks": { "PostToolUse": [ { "matcher": "Bash",
  "hooks": [ { "type": "command", "command": "python3 .codex/hooks/post_tool_use_review.py" } ] } ] } }
```

- 설정한 hook 확인: Claude `/hooks`(읽기 전용 브라우저), Codex `/hooks`(검토·신뢰·비활성화).
- Codex 예시의 상대 경로 실행 방식은 문서 원문(`$(git rev-parse --show-toplevel)` 사용)을 단순화한 것이다. 레슨에서는 문서 원문 형태를 쓴다. TODO(verify)

---

## 8. 컨텍스트 관리

| 항목 | Claude Code | Codex |
|---|---|---|
| 새 대화 | `/clear [name]` (이전 대화는 `/resume`에서 다시 열 수 있다) | `/new` (같은 CLI 세션), `/clear` (터미널과 대화를 함께 초기화) |
| 요약 압축 | `/compact [instructions]` | `/compact` |
| 컨텍스트 시각화 | `/context [all]` (색 격자, Memory files 목록) | 전용 명령 없음. `/status`가 토큰 사용량을 보여준다. TODO(verify) |
| 비용·사용량 | `/usage` (`/cost`, `/stats`는 별칭) | `/status`(세션 토큰), `/usage`(계정 토큰·한도) |
| 되돌리기 | `/rewind` (별칭 `/checkpoint`, `/undo`) | `/fork`로 분기 |
| 곁가지 질문 | `/btw` | `/side`, `/btw` |
| 자동 압축 설정 | `--autocompact <auto\|tokens>` | `model_auto_compact_token_limit` (config 키가 존재함. 의미는 TODO(verify)) |
| 백그라운드 작업 | `/tasks` (`/bashes`) | `/ps`, `/stop` |

출처: [Claude commands](https://code.claude.com/docs/en/commands), [Claude costs](https://code.claude.com/docs/en/costs), 로컬 `claude --help`, [Codex slash commands](https://learn.chatgpt.com/docs/developer-commands?surface=cli), [Codex config reference](https://learn.chatgpt.com/docs/config-file/config-reference)

- `/compact`는 CLAUDE.md를 다시 로드하는 동작과 상호작용한다. 자세한 내용은 [context-window](https://code.claude.com/docs/en/context-window)를 확인한다. TODO(verify)

---

## 9. 병렬 실행 · 클라우드

| 항목 | Claude Code | Codex |
|---|---|---|
| worktree 실행 | `claude -w <name>` / `claude --worktree <name>` → `.claude/worktrees/<name>/` | `codex --worktree` / `codex exec --worktree` ("새 관리형 Git worktree에서 실행") |
| tmux 연동 | `--worktree`와 함께 `--tmux` | 해당 없음 |
| subagent 격리 | agent front matter `isolation: worktree` | TODO(verify) |
| 백그라운드 세션 | `claude --bg "..."`, `claude agents`, `claude attach <id>`, `claude logs <id>`, `claude stop <id>` | `codex agents` (공유 로컬 app-server 데몬의 세션 탐색) |
| 클라우드 시작 | `claude --cloud "작업 설명"` (구 `--remote`는 deprecated 별칭), 웹 [claude.ai/code](https://claude.ai/code) | `codex cloud exec --env <ENV_ID> "작업"` (`--attempts N` best-of-N, `--branch`), 웹 ChatGPT의 Codex Cloud |
| 클라우드 → 로컬 | `claude --teleport` / `/teleport` | `codex cloud diff`, `codex cloud apply`, 그리고 `codex apply` (최신 diff를 `git apply`) |
| 클라우드 작업 목록 | 웹 세션 목록 | `codex cloud list`, `codex cloud status` |
| 실행 중 세션에 후속 지시 | `claude -p "메시지" --cloud <session-id>` | `codex queue` (기존 세션에 메시지 대기열 추가) |

출처: 로컬 `claude --help`, [Claude worktrees](https://code.claude.com/docs/en/worktrees), [Claude on the web](https://code.claude.com/docs/en/claude-code-on-the-web), 로컬 `codex --help`·`codex cloud --help`·`codex cloud exec --help`, [Codex Cloud](https://learn.chatgpt.com/docs/cloud), [Codex worktrees](https://learn.chatgpt.com/docs/environments/git-worktrees)

```bash
# Claude Code: 터미널 두 개에서 병렬 작업
claude --worktree feature-auth
claude --worktree bugfix-123
# 클라우드 작업 여러 개를 병렬로
claude --cloud "Fix the flaky test in auth.spec.ts"
claude --cloud "Update the API documentation"

# Codex
codex --worktree "implement feature X"
codex cloud exec --env <ENV_ID> --attempts 2 "Fix the flaky test"
codex cloud list
```

- Claude `--worktree` 디렉터리(`.claude/worktrees/`)는 `.gitignore`에 추가하도록 문서가 권장한다. 커밋이 하나도 없는 저장소에서는 실패한다 ([worktrees](https://code.claude.com/docs/en/worktrees), [common-workflows](https://code.claude.com/docs/en/common-workflows)).
- 클라우드 세션은 **저장소에 커밋된** 설정만 쓴다. 사용자 설정의 플러그인이나 `~/.claude/skills`는 로드하지 않는다 ([skills](https://code.claude.com/docs/en/skills)).
- `codex cloud`는 `--help`에 `[EXPERIMENTAL]`로 표시된다. Codex 문서는 "Codex Cloud (Legacy)"와 현재 Codex Cloud를 구분해서 설명한다. 레슨에는 UI 경로를 쓰지 말고 개념 위주로 설명한다.

---

## 10. 코드 리뷰

| 항목 | Claude Code | Codex |
|---|---|---|
| 로컬 리뷰 | `/code-review [low\|medium\|high\|xhigh\|max\|ultra] [--fix] [--comment] [pr#\|branch\|path]`. **`/review`는 그 별칭이다** | TUI `/review` (작업 트리 리뷰, `review_model` 설정 가능) |
| 비대화형 리뷰 | `claude -p "/code-review ..."` 형태는 TODO(verify) | `codex review --uncommitted` / `--base <BRANCH>` / `--commit <SHA>`, `codex exec review` |
| 보안 리뷰 | `/security-review` (현재 브랜치와 origin 기본 브랜치의 diff) | GitHub의 Security Review ([GitHub 연동](https://learn.chatgpt.com/docs/third-party/github)) |
| 클라우드 심층 리뷰 | `/ultrareview [PR or branch]`, `claude ultrareview [target] [--post] [--json]` | Codex Cloud 자동 리뷰 |
| GitHub PR 멘션 | `@claude ...` (claude-code-action 워크플로 필요) | `@codex review` (리뷰), 그 밖의 `@codex ...`는 PR을 컨텍스트로 클라우드 작업을 시작한다 |
| 관리형 자동 PR 리뷰 | Code Review (research preview, Team·Enterprise) + `REVIEW.md`·`CLAUDE.md`로 조정 | 저장소 설정에서 자동 리뷰 활성화, `AGENTS.md`의 `## Code Review Rules` 섹션으로 조정 |
| GitHub Action | `anthropics/claude-code-action@v1` (입력: `anthropic_api_key` 또는 `claude_code_oauth_token`, `prompt`, `claude_args`) | `openai/codex-action@v1` (입력: `openai-api-key`, `prompt` 또는 `prompt-file`, `codex-args`, `model`, `effort`, `sandbox`, `output-file`) |
| 설치 도우미 | `/install-github-app` (github.com 저장소만) | ChatGPT의 Codex 설정에서 GitHub 연결 |

출처: [Claude commands](https://code.claude.com/docs/en/commands), [Claude code review](https://code.claude.com/docs/en/code-review), [Claude GitHub Actions](https://code.claude.com/docs/en/github-actions), [claude-code-action README](https://github.com/anthropics/claude-code-action), 로컬 `claude ultrareview --help`, 로컬 `codex review --help`, [Codex code review](https://learn.chatgpt.com/docs/code-review), [Codex GitHub](https://learn.chatgpt.com/docs/third-party/github), [Codex GitHub Action](https://learn.chatgpt.com/docs/github-action)

```yaml
# Claude: @claude 멘션에 응답 (문서 예시를 줄인 것)
on: { issue_comment: { types: [created] }, pull_request_review_comment: { types: [created] } }
jobs:
  claude:
    if: contains(github.event.comment.body, '@claude')
    runs-on: ubuntu-latest
    permissions: { contents: write, pull-requests: write, issues: write, id-token: write, actions: read }
    steps:
      - uses: actions/checkout@v6
        with: { fetch-depth: 1 }
      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
```

```yaml
# Codex: PR 리뷰 단계 (문서 예시 중 Codex 실행 부분)
      - uses: openai/codex-action@v1
        id: run_codex
        with:
          openai-api-key: ${{ secrets.OPENAI_API_KEY }}
          prompt-file: .github/codex/prompts/review.md
          output-file: codex-output.md
# 이후 job에서 steps.run_codex.outputs.final-message를 PR 코멘트로 게시한다
```

- claude-code-action은 `prompt` 입력이 없으면 트리거 문구(`@claude`)를 기다리는 **interactive mode**로, `prompt`가 있으면 어떤 이벤트(cron 포함)에서든 바로 실행하는 **automation mode**로 동작한다 ([GitHub Actions](https://code.claude.com/docs/en/github-actions)).
- codex-action은 Linux·macOS 러너를 전제로 한다. Windows에서는 `safety-strategy: unsafe`가 필요하다 ([Codex GitHub Action](https://learn.chatgpt.com/docs/github-action)).
- `OPENAI_API_KEY`나 `CODEX_API_KEY`를 job 수준 환경 변수로 두지 않는다. 저장소 코드(빌드 스크립트, 테스트)가 키를 읽을 수 있다 ([non-interactive](https://learn.chatgpt.com/docs/non-interactive-mode)).

---

## 11. 모델 · 비용 (개요)

| 항목 | Claude Code | Codex |
|---|---|---|
| 모델 선택 | `--model <alias\|full-id>`, `/model`, 설정 `model` | `-m, --model`, `/model`, `config.toml`의 `model` |
| 별칭 | `default`, `best`, `fable`, `opus`, `sonnet`, `haiku`, `opus[1m]`, `sonnet[1m]`, `opusplan` | 별칭 체계 없음. 모델 ID를 쓴다 (사용 가능 목록은 계정마다 다르다) |
| 추론 강도 | `--effort low\|medium\|high\|xhigh\|max`, `/effort` | `model_reasoning_effort`, `/model`에서 함께 선택 |
| 빠른 모드 | `/fast [on\|off]` | `/fast` (모델 카탈로그에 Fast tier가 있을 때만 보인다) |
| 대체 모델 | `--fallback-model <model[,model...]>` | TODO(verify) |
| 사용량 | `/usage`, `-p --output-format json`의 `total_cost_usd` | `/status`, `/usage`, `codex exec --json`의 `turn.completed.usage` |
| 로컬/오픈 모델 | Bedrock·Vertex·Foundry 등 3P 공급자 | `--oss --local-provider lmstudio\|ollama` |

출처: [Claude model config](https://code.claude.com/docs/en/model-config), [Claude costs](https://code.claude.com/docs/en/costs), 로컬 `claude --help`, 로컬 `codex --help`, [Codex models](https://learn.chatgpt.com/docs/models), [Codex config-basic](https://learn.chatgpt.com/docs/config-file/config-basic)

- 구체적인 모델 ID(예: Codex 문서 예시의 `gpt-6.1-sol`)는 계정마다 다르고 자주 바뀐다. 레슨에는 `<사용 가능한 모델 ID>`처럼 자리 표시자로 쓰고, 별칭(`opus`, `sonnet`) 수준에서 설명한다.

---

## 12. 레슨 작성자를 위한 주의사항

### 12.1 이번 검증에서 확인한 "자주 틀리는" 사실

1. **Codex `--full-auto`는 쓰지 않는다.** 문서에는 deprecated 호환 경로라고 나오지만 0.160.1 바이너리는 이 플래그를 거부한다. `codex exec --sandbox workspace-write`를 쓴다.
2. **Codex `-a untrusted` / `-a on-failure`는 사라졌다.** 현재 값은 `on-request`, `never`, 그리고 config의 `granular` 테이블이다. 예전 블로그나 강좌의 명령을 그대로 옮기지 않는다.
3. **Codex 프로필은 별도 파일이다** (`~/.codex/<name>.config.toml`). `[profiles.x]` 예시는 0.134.0 이후 동작하지 않는다.
4. **Codex `exec`의 기본 샌드박스는 read-only다.** "exec로 고쳐줘"가 아무것도 바꾸지 못하는 실습 함정이 생긴다.
5. **Claude Code의 기본 권한 모드는 `auto`다** (v2.1.283+ 대화형). "처음에는 매번 승인 창이 뜬다"는 설명은 더 이상 기본 동작이 아니다. Manual 동작을 보여주려면 `--permission-mode default`(또는 `manual`)를 명시한다.
6. **Claude Code는 `AGENTS.md`를 직접 읽지만, `CLAUDE.md`가 없을 때만 읽는다.** 두 파일을 다 쓰는 저장소는 `CLAUDE.md`에 `@AGENTS.md`를 넣는 방식이 가장 안전하다(이 저장소의 방식).
7. **`/review`의 의미가 도구마다 다르다.** Claude Code의 `/review`는 `/code-review`의 별칭이고(effort 인수, `--fix`, `--comment`), Codex의 `/review`는 작업 트리 리뷰다.
8. **Claude 커스텀 명령은 skills로 통합됐고, Codex custom prompts는 deprecated다.** 새 레슨은 두 도구 모두 `SKILL.md` 기반으로 가르친다(Claude `.claude/skills/`, Codex `.agents/skills/`).
9. **Codex hooks는 신뢰 절차가 필요하다.** hook을 만들고 나서 `/hooks`에서 신뢰하지 않으면 실행되지 않는다. 실습 체크리스트에 이 단계를 넣는다.
10. **Claude `/agents`는 더 이상 마법사가 아니다** (v2.1.198+). 오래된 스크린샷을 쓰지 않는다.
11. **`claude mcp add`에서 `--`를 빠뜨리면** 서버 플래그를 잘못 해석한다. `--env` 바로 뒤에 이름을 두는 실수도 흔하다.
12. **Codex 문서 주소**: `developers.openai.com/codex/*`는 `learn.chatgpt.com/docs/*`로 리다이렉트된다. 링크가 깨지면 두 도메인을 모두 확인한다.
13. **`codex exec`에는 `-a`가 없다.** 승인 정책 플래그는 대화형 `codex`에만 있다. `exec`는 `--sandbox`, `-p/--profile`, `-i/--image`, `-o`, `--json`, `--output-schema`를 받는다.
14. **`codex exec resume`은 `--sandbox`를 거부한다.** 샌드박스는 `resume` 앞에 둔다: `codex exec --sandbox workspace-write resume --last "..."`.
15. **`codex review` / `codex exec review`는 `--base`·`--uncommitted`·`--commit`과 사용자 프롬프트를 함께 받지 않는다.** 리뷰 기준은 `AGENTS.md`(예: `## Code Review Rules`)에 둔다.
16. **`-p`의 의미가 다르다.** Claude `-p`는 print(비대화형), Codex `-p`는 `--profile`이다. `codex exec -p "..."`는 프롬프트가 아니라 프로필 이름으로 해석된다.
17. **`claude --bare`는 OAuth/키체인 로그인을 읽지 않는다.** `ANTHROPIC_API_KEY` 또는 `apiKeyHelper`가 필요하므로 구독 로그인 사용자는 실패한다.
18. **`codex cloud status|diff|apply`는 `<TASK_ID>`가 필수다.** `diff`·`apply`는 `--attempt N`도 받는다.
19. **Codex `--add-dir`는 쓰기 가능한 디렉터리를 추가한다** (읽기 전용 추가가 아니다).
20. **Codex에는 Claude `Read(./.env)` 같은 경로 단위 읽기 차단 규칙이 확인되지 않았다.** 실제 비밀은 저장소 밖에 두고 `shell_environment_policy`로 환경변수 노출을 줄인다. TODO(verify)
21. **세션 로그 위치(문서화되지 않음, 로컬 확인)**: Claude `~/.claude/projects/<encoded-path>/<session-id>.jsonl`, Codex `~/.codex/sessions/YYYY/MM/DD/rollout-*.jsonl`. 레슨에서는 의존하지 말고 `--output-format stream-json` / `--json` 출력을 쓴다.
22. **Claude `/usage`, `total_cost_usd`는 정가 기준 달러 추정치를 보여준다** (관리자는 `modelPricing` managed setting으로 바꿀 수 있다). 이 가이드는 비용을 토큰으로만 기록한다.
23. **TaskFlow 검증 커맨드는 `make verify`로 통일한다.** B0(01-1)에서 만들고 04-1에서 `verify-fast`를 추가한다.

추가 공식 문서: [managed settings](https://code.claude.com/docs/en/managed-settings.md), [settings reference](https://code.claude.com/docs/en/settings-reference.md), [monitoring usage](https://code.claude.com/docs/en/monitoring-usage.md)

### 12.2 자주 바뀌는 영역 (레슨마다 `last-verified` 갱신 대상)

- 권한 모드 이름·기본값, Codex 승인 정책 값 (최근 1년 사이 둘 다 바뀌었다).
- 모델 별칭과 모델 ID, effort 단계 이름.
- 클라우드 기능(`claude --cloud`/`--teleport`, `codex cloud`의 EXPERIMENTAL 표시, Codex Cloud Legacy와 현재 버전의 구분).
- 코드 리뷰 제품(Claude Code Review는 research preview, `/ultrareview`, Codex 자동 리뷰).
- 슬래시 명령 목록 (Claude 문서에는 "Removed in v2.1.92" 같은 표기가 수시로 붙는다).
- Hook 이벤트 목록 (Claude는 30개 이상이고 계속 늘어난다).

### 12.3 작성 규칙

- 명령을 레슨에 넣기 전에 `claude --help`, `codex <sub> --help`로 해당 버전에서 다시 확인하고, front matter `tools:`에 버전을 적는다.
- 이 문서에 `TODO(verify)`로 표시된 항목은 레슨에서 단정하지 않는다.
- 같은 개념이라도 이름이 다르면(Claude `plan` 모드와 Codex `/plan` + `read-only`) 비교표로 차이를 드러낸다. 1:1 대응이라고 쓰지 않는다.
- 위험 플래그(`--dangerously-skip-permissions`, `--yolo`, `danger-full-access`)는 컨테이너·VM 같은 격리 환경 실습에서만 쓰고, 경고 박스를 함께 단다.

### 12.4 이 문서의 TODO(verify) 목록

- claude-code-action, codex-action의 최신 릴리스 태그 (GitHub API 접근이 제한돼 `@v1`만 확인했다)
- Codex 비대화형 실행의 비용·예산 상한 옵션 유무
- Claude Code 대화형 세션에서 프로젝트 hooks에 별도 신뢰 절차가 있는지
- Codex 이름 붙은 권한 프로필(`[permissions.<name>]`)의 세부 문법
- Codex `.rules`(execpolicy) 파일 위치와 문법 예시
- Codex hooks 예시의 command 경로 형태 (문서 원문은 `git rev-parse` 사용)
- Codex의 컨텍스트 시각화 명령 유무, `model_auto_compact_token_limit`의 정확한 의미
- `/compact`와 CLAUDE.md 재로드의 상호작용
- Codex subagent의 worktree 격리 옵션
- `claude -p`에서 `/code-review`를 실행하는 방식
- Codex의 대체 모델(fallback) 설정 유무
