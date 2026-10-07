---
id: 01-5
title: "재현 가능한 개발 환경 (devcontainer, 셋업 스크립트, 클라우드 세션)"
module: 01-environment-setup
level: L2
duration: 2h
last-verified: 2026-10-07
tools:
  claude-code: 2.1.292
  codex: 0.160.1
---

# 01-5. 재현 가능한 개발 환경 (devcontainer, 셋업 스크립트, 클라우드 세션)

## 🎯 학습 목표
- "내 PC에서는 된다"가 에이전트 개발에서 왜 더 큰 문제가 되는지 설명할 수 있다.
- 멱등(idempotent) 셋업 스크립트와 devcontainer로 TaskFlow 개발 환경을 코드로 정의할 수 있다.
- `SessionStart` hook으로 세션이 시작될 때 의존성을 자동으로 확인·설치하고, 그 결과를 에이전트 컨텍스트에 넣을 수 있다.
- 클라우드 세션의 setup script와 `SessionStart` hook의 역할 차이를 구분하고, Claude Code·Codex 클라우드에서 같은 저장소를 실행할 수 있다.

## 📋 사전 준비
- 선행 레슨: [01-1](./01-1-project-memory.md) ~ [01-4](./01-4-extensions.md), [00-3 대화형/헤드리스/클라우드 실행 모드](../00-foundations/00-3-execution-modes.md)
- 필요 도구/계정
  - Claude Code 2.1.292 이상, Codex 0.160.1 이상
  - Docker, VS Code + Dev Containers 확장(또는 Dev Containers 스펙을 지원하는 다른 편집기)
  - 클라우드 실습용: claude.ai의 Claude Code 클라우드 세션을 쓸 수 있는 요금제, ChatGPT의 Codex Cloud 사용 권한, GitHub 연결
- 실습 저장소 상태: 01-4까지 커밋된 TaskFlow 저장소가 GitHub에 push되어 있다(`docker-compose.yml`의 `postgres` 서비스 포함).

## 💡 개념

### 에이전트는 "새로 온 팀원"을 매일 수십 번 반복한다

사람은 한 번 환경을 맞추면 몇 달을 쓴다. 에이전트는 다르다. worktree를 새로 만들 때, 클라우드 세션을 띄울 때, CI에서 실행될 때마다 **빈 환경에서 시작**한다. 의존성이 안 깔려 있으면 에이전트는 테스트를 못 돌리고, 테스트를 못 돌리면 "아마 될 것"이라고 보고한다. Verify, Don't Trust 원칙이 환경 문제 하나로 무너진다.

| 실행 위치 | 환경을 준비하는 수단 | 저장소에 남는 것 |
|---|---|---|
| 개인 PC (로컬) | 셋업 스크립트, `SessionStart` hook | `scripts/setup.sh`, hook 설정 |
| devcontainer / Codespaces | `devcontainer.json`, Dockerfile, features | `.devcontainer/` |
| Claude Code 클라우드 세션 | 클라우드 환경의 **setup script**(VM 준비) + 저장소의 `SessionStart` hook(프로젝트 준비) | hook 설정 (setup script는 claude.ai 환경 설정에 저장) |
| Codex Cloud | 클라우드 환경의 설치 스크립트·인터넷 접근 설정 | `AGENTS.md` (환경 설정은 ChatGPT 쪽에 저장) |

### 계층으로 나누어 생각한다

```mermaid
flowchart TB
    subgraph L1["① 머신/VM 계층 (드물게 바뀜)"]
        A1["OS 패키지, Node 22, pnpm, Docker"]
    end
    subgraph L2["② 프로젝트 계층 (lockfile이 바뀔 때마다)"]
        B1["pnpm install --frozen-lockfile"]
        B2["DB 컨테이너 기동 + 마이그레이션"]
    end
    subgraph L3["③ 세션 계층 (매 세션)"]
        C1["상태 확인 → 에이전트 컨텍스트에 요약 주입"]
    end
    L1 --> L2 --> L3
    D1[".devcontainer/<br/>클라우드 setup script"] -.담당.-> L1
    D2["scripts/setup.sh"] -.담당.-> L2
    D3["SessionStart hook"] -.담당.-> L3
    D3 -. "필요할 때만 호출" .-> D2
```

- ①은 무겁고 드물게 바뀐다. 이미지·캐시로 만들어 두는 것이 이득이다. Claude Code 클라우드는 setup script가 약 5분 안에 끝나면 파일시스템을 스냅숏으로 캐시하고 다음 세션부터 건너뛴다.
- ②는 lockfile이 바뀔 때만 다시 하면 된다. **멱등** 스크립트로 만들어 몇 번을 실행해도 안전하게 한다.
- ③은 매 세션 실행되므로 빨라야 한다. "이미 설치됐으면 건너뛰기"를 반드시 넣는다.

### Claude Code 클라우드: setup script vs SessionStart hook

| | Setup script | SessionStart hook |
|---|---|---|
| 설정 위치 | claude.ai/code의 클라우드 환경 설정 대화상자 | 저장소의 `.claude/settings.json` |
| 실행 시점 | Claude Code 시작 **전**. 캐시된 환경이 있으면 건너뜀 | Claude Code 시작 **후**, 재개 포함 매 세션 |
| 실행 위치 | 클라우드 세션만 | 로컬과 클라우드 모두 |
| 적합한 일 | 기본 이미지에 없는 도구 설치 (root로 실행, `apt install` 가능) | `pnpm install` 같은 프로젝트 준비 |

클라우드 VM에서는 환경 변수 `CLAUDE_CODE_REMOTE`가 `true`다. 로컬에서는 절대 `true`가 아니므로, hook 스크립트에서 이 값으로 클라우드 전용 동작을 나눌 수 있다.

### 왜 대규모 개발에서 중요한가

1. **병렬 실행의 전제.** worktree 3개, 클라우드 세션 5개를 동시에 돌리려면([05-1](../05-scaling-up/05-1-parallel-agents.md)) 각 환경이 사람 손 없이 준비되어야 한다.
2. **결과의 비교 가능성.** Claude 구현을 Codex가 리뷰할 때, 둘이 다른 Node 버전·다른 의존성에서 테스트를 돌리면 교차 검증이 의미를 잃는다.
3. **온보딩 비용 절감.** 사람 팀원도 같은 이득을 본다. "신규 팀원 10분 안에 에이전트 실행"은 이 레슨의 숙제 목표이자 B0의 완료 기준이다.
4. **보안 경계.** devcontainer는 에이전트 명령을 호스트가 아닌 컨테이너 안에서 실행하게 한다. 위험한 권한 모드를 써야 할 때 최소한의 격리를 제공한다.

## 👣 따라하기

### Step 1. 멱등 셋업 스크립트를 만든다
모든 환경(로컬·devcontainer·클라우드)이 공유할 "프로젝트 계층" 준비를 스크립트 하나로 만든다. 도구 독립적인 단계지만, 에이전트에게 작성을 맡기고 사람이 검토한다.

**Claude Code 레시피**
```bash
claude
> scripts/setup.sh를 만들어. 요구사항:
> - bash, set -euo pipefail, 여러 번 실행해도 안전(멱등)
> - corepack으로 package.json의 packageManager에 맞는 pnpm 활성화
> - pnpm install --frozen-lockfile
> - docker가 있으면 docker compose up -d postgres 후 pg_isready로 최대 30초 대기
> - pnpm --filter @taskflow/db migrate 스크립트가 있으면 실행, 없으면 건너뛰고 안내
> - 각 단계 시작/완료를 한 줄로 출력, 마지막에 "SETUP: OK"
> 작성 후 두 번 연속 실행해서 두 번째도 성공하는지 확인해.
```

**Codex 레시피**
```bash
codex --sandbox workspace-write
> Create scripts/setup.sh with these requirements:
> - bash, set -euo pipefail, idempotent (safe to run repeatedly)
> - enable pnpm via corepack according to package.json packageManager
> - pnpm install --frozen-lockfile
> - if docker exists: docker compose up -d postgres, wait up to 30s with pg_isready
> - run `pnpm --filter @taskflow/db migrate` only if that script exists
> - print one line per step, end with "SETUP: OK"
> Then run it twice and confirm the second run also succeeds.
```
> Codex `workspace-write`는 기본적으로 네트워크가 막혀 있어(01-2에서 `network_access = false`) `pnpm install`이나 `docker compose`는 승인 요청이 뜬다. 내용을 확인하고 승인한다.

결과물은 대략 다음과 같아야 한다. 사람이 읽고 확인한다.
```bash
#!/usr/bin/env bash
set -euo pipefail
cd "$(git rev-parse --show-toplevel)"

echo "[setup] pnpm 활성화"
corepack enable >/dev/null 2>&1 || true

echo "[setup] 의존성 설치"
pnpm install --frozen-lockfile

if command -v docker >/dev/null 2>&1; then
  echo "[setup] PostgreSQL 기동"
  docker compose up -d postgres
  for i in $(seq 1 30); do
    docker compose exec -T postgres pg_isready -U taskflow >/dev/null 2>&1 && break
    sleep 1
  done
fi

if pnpm --filter @taskflow/db run 2>/dev/null | grep -q "migrate"; then
  echo "[setup] 마이그레이션"
  pnpm --filter @taskflow/db migrate
else
  echo "[setup] migrate 스크립트 없음 (B1 이후 추가) - 건너뜀"
fi
echo "SETUP: OK"
```

**기대 결과**: `bash scripts/setup.sh`를 두 번 실행해도 모두 `SETUP: OK`로 끝난다. `AGENTS.md`의 명령어 섹션에 `- 환경 준비: bash scripts/setup.sh`를 추가한다.

### Step 2. devcontainer로 머신 계층을 고정한다
누가 어디서 열든 같은 OS·Node·도구 버전을 쓰게 한다. 에이전트 명령은 호스트가 아니라 컨테이너 안에서 실행된다.

```bash
mkdir -p .devcontainer
cat > .devcontainer/devcontainer.json <<'EOF'
{
  "name": "taskflow",
  "image": "mcr.microsoft.com/devcontainers/base:ubuntu",
  "features": {
    "ghcr.io/devcontainers/features/node:1": { "version": "22" },
    "ghcr.io/devcontainers/features/docker-in-docker:2": {},
    "ghcr.io/anthropics/devcontainer-features/claude-code:1.0": {}
  },
  "remoteUser": "vscode",
  "mounts": [
    "source=claude-code-config-${devcontainerId},target=/home/vscode/.claude,type=volume",
    "source=codex-home-${devcontainerId},target=/home/vscode/.codex,type=volume"
  ],
  "containerEnv": {
    "CLAUDE_CONFIG_DIR": "/home/vscode/.claude",
    "CODEX_HOME": "/home/vscode/.codex"
  },
  "postCreateCommand": "npm install -g @openai/codex@0.160.1 && bash scripts/setup.sh"
}
EOF
```

설계 포인트:
- **Claude Code**는 공식 Dev Container Feature(`ghcr.io/anthropics/devcontainer-features/claude-code`)로 설치한다. 태그 `:1.0`은 설치 스크립트 버전이고 Claude Code 자체는 최신이 설치된다. CLI 버전까지 고정하려면 Dockerfile에서 `npm install -g @anthropic-ai/claude-code@2.1.292`로 설치하고 `containerEnv`에 `DISABLE_AUTOUPDATER=1`을 둔다.
- **인증 유지**: 재빌드하면 홈 디렉터리가 사라진다. `~/.claude` 볼륨과 `CLAUDE_CONFIG_DIR`을 같은 경로로 맞춰야 `.claude.json`(로그인 정보)까지 볼륨에 저장된다. Codex는 `CODEX_HOME`으로 같은 효과를 낸다.
- **Codex**는 `postCreateCommand`에서 npm으로 버전을 고정해 설치한다. 컨테이너 안에서 브라우저 콜백이 어려우면 `codex login --device-auth`를 쓴다.
- `remoteUser`는 root가 아니어야 한다. Claude Code는 root로 실행할 때 `--dangerously-skip-permissions`를 거부한다.
- MCP 서버는 01-3처럼 저장소의 `.mcp.json`/`.codex/config.toml`에 두면 컨테이너에서도 그대로 쓴다. stdio 서버가 필요로 하는 바이너리는 이미지에 설치한다.

**Claude Code 레시피**
```bash
# VS Code: Command Palette → "Dev Containers: Rebuild Container" 후 통합 터미널에서
claude --version
claude
> /status
> /memory
> bash scripts/setup.sh 를 실행하고 결과를 요약해줘.
```

**Codex 레시피**
```bash
codex --version          # codex-cli 0.160.1
codex login --device-auth
codex "List the instruction sources you loaded, then run bash scripts/setup.sh and summarize."
```

> ⚠️ **위험 플래그 경고**: devcontainer는 무인 실행(`claude --dangerously-skip-permissions`, `codex --dangerously-bypass-approvals-and-sandbox`)을 검토할 수 있는 장소지만, 컨테이너 안에서 접근 가능한 모든 것(바인드 마운트된 작업 디렉터리, `~/.claude`의 자격 증명 포함)은 여전히 유출될 수 있다. 신뢰하는 저장소에서만, 호스트의 `~/.ssh`나 클라우드 자격 증명 파일을 마운트하지 않은 상태에서, 네트워크 egress 제한과 함께 쓴다. 참고 구현은 `anthropics/claude-code` 저장소의 `.devcontainer/`(`devcontainer.json`, `Dockerfile`, `init-firewall.sh`)에 있다.

**기대 결과**: 컨테이너 안에서 두 도구가 실행되고, 메모리 파일·skill·hook이 로컬과 똑같이 로드된다. 컨테이너를 재빌드한 뒤에도 다시 로그인하지 않아도 된다.

### Step 3. SessionStart hook으로 세션 시작 시 의존성을 자동 확인한다
세션이 시작될 때마다 "의존성이 최신인가"를 확인하고, 필요할 때만 설치한 뒤 상태 요약을 에이전트 컨텍스트에 넣는다. 두 도구 모두 `SessionStart` hook의 stdout(일반 텍스트)을 컨텍스트에 추가한다.

공용 스크립트:
```bash
cat > scripts/agent-hooks/session-start.sh <<'EOF'
#!/usr/bin/env bash
# 세션 시작 시 실행. stdout은 에이전트 컨텍스트에 추가되므로 짧게 쓴다.
set -uo pipefail
cd "$(git rev-parse --show-toplevel)"

status=()
# lockfile이 node_modules보다 새로우면(또는 설치 흔적이 없으면) 재설치
if [ ! -f node_modules/.modules.yaml ] || [ pnpm-lock.yaml -nt node_modules/.modules.yaml ]; then
  if pnpm install --frozen-lockfile >/tmp/taskflow-install.log 2>&1; then
    status+=("deps: installed")
  else
    status+=("deps: INSTALL FAILED (see /tmp/taskflow-install.log)")
  fi
else
  status+=("deps: up to date")
fi

# 클라우드 세션(Claude Code)에서만 DB를 띄운다
if [ "${CLAUDE_CODE_REMOTE:-}" = "true" ] && command -v docker >/dev/null 2>&1; then
  docker compose up -d postgres >/dev/null 2>&1 && status+=("db: started") || status+=("db: FAILED")
fi

# Claude Code: 이후 Bash 명령에서 쓸 환경 변수 저장
if [ -n "${CLAUDE_ENV_FILE:-}" ]; then
  echo 'export DATABASE_URL=postgresql://taskflow:taskflow@localhost:5432/taskflow' >> "$CLAUDE_ENV_FILE"
fi

echo "TaskFlow session check: ${status[*]}. Run 'make verify' before reporting completion."
exit 0
EOF
chmod +x scripts/agent-hooks/session-start.sh
```

**Claude Code 레시피**

`.claude/settings.json`의 `hooks`에 `SessionStart`를 추가한다(01-4의 `PostToolUse`와 나란히 둔다).
```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|resume",
        "hooks": [
          {
            "type": "command",
            "command": "bash \"$CLAUDE_PROJECT_DIR\"/scripts/agent-hooks/session-start.sh",
            "timeout": 300
          }
        ]
      }
    ]
  }
}
```
- matcher 값: `startup`(새 세션), `resume`(`--resume`/`--continue`/`/resume`), `clear`, `compact`, `fork`.
- `CLAUDE_ENV_FILE`에 `export` 문을 **추가(`>>`)** 하면 이후 Bash 명령에 환경 변수가 유지된다. 다른 hook이 쓴 내용을 지우지 않도록 `>`를 쓰지 않는다.
- `command` hook의 기본 제한 시간은 600초다. 여기서는 `timeout`(초)을 300으로 줄였다.
- 대화형 세션에서 SessionStart hook은 백그라운드로 실행되고, Claude의 첫 응답은 hook이 끝날 때까지 기다린다.

확인:
```bash
rm -rf node_modules
claude
> 세션 시작 때 받은 TaskFlow session check 내용을 그대로 알려줘.
```

**Codex 레시피**

`.codex/hooks.json`에 `SessionStart`를 추가한다(01-4의 `PostToolUse`와 같은 파일). Codex의 SessionStart matcher는 `startup`, `resume`, `clear`, `compact`다.
```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|resume",
        "hooks": [
          {
            "type": "command",
            "command": "/bin/bash \"$(git rev-parse --show-toplevel)/scripts/agent-hooks/session-start.sh\"",
            "timeout": 300,
            "statusMessage": "Checking TaskFlow dependencies"
          }
        ]
      }
    ]
  }
}
```
hook 파일이 바뀌었으므로 **다시 신뢰**한다.
```bash
rm -rf node_modules
codex
> /hooks
```
신뢰한 뒤 새 세션을 시작해 확인한다.
```bash
codex "What did the TaskFlow session check report at startup?"
```
> Codex에는 `CLAUDE_ENV_FILE`에 해당하는 기능이 레퍼런스에 없다. 스크립트의 해당 블록은 Codex에서 그냥 건너뛴다. Codex 세션에 환경 변수가 필요하면 셸에서 미리 설정하거나 `.env.example`을 안내한다. TODO(verify): Codex hook에서 세션 환경 변수를 지속시키는 방법이 있는지.

**기대 결과**: `node_modules`를 지운 상태로 두 도구를 시작하면 hook이 설치를 수행하고, 에이전트는 `deps: installed`가 포함된 요약을 알고 있다. 다시 시작하면 `deps: up to date`로 바로 끝난다.

### Step 4. 클라우드 세션에서 같은 저장소를 실행한다
로컬과 devcontainer에서 맞춘 환경이 클라우드에서도 재현되는지 확인한다. 클라우드 세션은 **저장소에 커밋된 것만** 쓴다는 점을 기억한다.

먼저 모든 설정을 커밋하고 push한다.
```bash
git add scripts/setup.sh scripts/agent-hooks .devcontainer .claude/settings.json .codex/hooks.json AGENTS.md
git commit -m "chore: reproducible env (setup script, devcontainer, SessionStart hook)"
git push
```

**Claude Code 레시피**

1. claude.ai/code에서 클라우드 환경을 연다(처음이면 온보딩이 **Default** 환경을 **Trusted** 네트워크 수준으로 만든다). 기본 이미지에는 Node.js 22, pnpm, Docker, PostgreSQL 16 등이 이미 있으므로 TaskFlow에는 setup script가 필수는 아니다. 기본 이미지에 없는 도구가 필요할 때만 **Setup script** 칸에 적는다.
   ```bash
   # 예: 기본 이미지에 없는 도구를 VM 계층에 추가 (root로 실행된다)
   apt update && apt install -y shellcheck
   ```
2. 터미널에서 클라우드 세션을 시작한다. 클라우드 VM은 로컬 체크아웃이 아니라 **현재 브랜치의 GitHub 원격**을 클론한다.
   ```bash
   claude --cloud "TaskFlow 세션 시작 hook이 보고한 상태를 알려주고, make verify를 실행해서 결과를 요약해줘."
   ```
3. 결과를 로컬로 가져오려면 `claude --teleport`(세션 목록) 또는 세션 안에서 `/teleport`를 쓴다.

클라우드 세션에서 확인할 것:
- 저장소의 `CLAUDE.md`, `.claude/settings.json`의 hook·권한 규칙, `.mcp.json`, `.claude/skills/`, `.claude/agents/`는 그대로 쓰인다(저장소가 하나인 세션 기준).
- `~/.claude/CLAUDE.md`, `~/.claude/skills/`, 사용자 설정의 hook, local·user 범위로 추가한 MCP 서버는 **쓰이지 않는다**.
- 환경 변수는 환경 설정에 `.env` 형식(`KEY=value` 한 줄씩)으로 넣는다. 환경을 쓰는 사람은 누구나 값과 setup script를 읽을 수 있으므로 비밀 값을 넣지 않는다.

**Codex 레시피**

1. ChatGPT의 Codex 설정에서 GitHub를 연결하고 TaskFlow 저장소용 클라우드 환경을 만든다. 환경에는 의존성 설치 스크립트, 환경 변수/시크릿, 인터넷 접근 설정이 있다. 인터넷 접근은 **기본적으로 꺼져 있으므로** 설치에 필요한 패키지 레지스트리만 허용한다. 화면 구성과 이름은 자주 바뀌므로 여기서는 개념만 적는다. TODO(verify): Codex Cloud 환경의 설치 스크립트 실행 시점과 캐시 동작, 저장소 `.codex/hooks.json`이 클라우드 작업에서 실행되는지.
2. 설치 스크립트 칸에는 Step 1의 스크립트를 재사용한다.
   ```bash
   bash scripts/setup.sh
   ```
3. CLI에서 작업을 보내고 결과를 가져온다(`codex cloud`는 0.160.1에서 `[EXPERIMENTAL]`로 표시된다).
   ```bash
   codex cloud exec --env <ENV_ID> "Run make verify and summarize the result. Do not change code."
   codex cloud list
   codex cloud diff <TASK_ID>   # 변경이 있다면 확인 (TASK_ID는 list 출력에서)
   ```

**기대 결과**: 두 클라우드 세션 모두 사람이 개입하지 않고 의존성 설치와 `make verify` 실행까지 도달한다(아직 `make verify`가 없다면 "명령 없음"을 정확히 보고하는 것까지가 성공이다). 실패하면 원인이 "커밋하지 않은 설정", "네트워크 허용 목록", "기본 이미지에 없는 도구" 중 무엇인지 분류해 기록한다.

### Step 5. 10분 온보딩을 리허설한다
모든 장치를 합쳐 "신규 팀원이 클론부터 첫 에이전트 작업까지 10분"이 되는지 직접 재 본다. 도구 독립적인 단계다.

```bash
cd "$(mktemp -d)"
time ( git clone <TaskFlow 저장소 URL> taskflow && cd taskflow && bash scripts/setup.sh )
```

**Claude Code 레시피**
```bash
cd taskflow
claude -p "docs/onboarding.md 초안을 작성해. 대상: 첫날 신규 팀원. 섹션: 사전 요구사항 / 5분 설치(로컬 또는 devcontainer) / 에이전트 첫 실행(Claude Code, Codex 각각) / Codex /hooks 신뢰 / 클라우드 세션 / 문제 해결. 저장소의 scripts/setup.sh, .devcontainer, .claude, .codex 설정을 근거로 쓰고, 확인하지 않은 내용은 TODO로 표시해."
```

**Codex 레시피**
```bash
cd taskflow
codex exec "Review docs/onboarding.md against the actual repo files (scripts/setup.sh, .devcontainer, .claude, .codex). List every step that is missing, wrong, or unverifiable, as a checklist. Do not edit files."
```

**기대 결과**: `time` 결과가 10분 이내이고, `docs/onboarding.md`에 Claude가 쓴 초안과 Codex의 교차 검토로 찾은 누락이 반영되어 있다. 10분을 넘겼다면 가장 오래 걸린 단계(보통 `pnpm install`이나 이미지 빌드)를 기록하고 캐시 전략(devcontainer 이미지 사전 빌드, 클라우드 환경 캐시)을 HW에서 개선한다.

## ✅ 체크포인트
- [ ] `scripts/setup.sh`가 두 번 연속 실행해도 `SETUP: OK`로 끝난다.
- [ ] devcontainer 안에서 Claude Code와 Codex가 실행되고, 재빌드 후에도 로그인이 유지된다.
- [ ] `node_modules`를 지운 뒤 두 도구를 시작하면 SessionStart hook이 설치를 수행하고 요약을 컨텍스트에 넣는다.
- [ ] Codex `/hooks`에서 수정된 hook을 다시 신뢰했다.
- [ ] Claude Code 클라우드 세션과 Codex Cloud 작업이 사람 개입 없이 검증 명령까지 도달했다.
- [ ] 클론부터 셋업 완료까지 걸린 시간을 측정했다.

## 🏠 숙제
| # | 유형 | 난이도 | 과제 | 완료 조건 |
|---|---|---|---|---|
| HW1 | 🔁 Reproduce | ★ | SessionStart hook으로 의존성 자동 설치를 재현한다 | `node_modules` 삭제 후 두 도구에서 세션을 시작한 로그(hook 출력, 에이전트가 요약을 인용한 응답)와 재시작 시 `deps: up to date`가 나온 로그가 제출 문서에 있다 |
| HW2 | 🛠 Apply | ★★ | 신규 팀원이 10분 안에 에이전트를 돌릴 수 있는 온보딩 문서를 완성한다 | `docs/onboarding.md`가 커밋되어 있고, 본인이 아닌 사람(동료 또는 새 계정·새 머신) 1명이 문서만 보고 따라 한 소요 시간과 막힌 지점 기록이 있으며, 막힌 지점을 문서·스크립트에 반영한 diff가 있다 |
| HW3 | 🚀 Challenge | ★★★ | 세 환경(로컬, devcontainer, 클라우드 세션 1종 이상)에서 같은 작업을 돌려 재현성을 검증한다 | 세 환경에서 `make verify`(또는 동등한 검증) 결과가 같음을 보이는 로그, 환경별 준비 시간 비교표, 캐시 적용 전/후 시간 비교(devcontainer 이미지 사전 빌드 또는 클라우드 환경 캐시 중 1개 이상)가 있다 |

제출: `hw/01-5` 브랜치, `submissions/01-5.md` (템플릿: [docs/design/homework-template.md](../../docs/design/homework-template.md))

## ⚠️ 흔한 실수
- **클라우드 세션에서 hook·skill·MCP가 안 보인다** → 개인 경로(`~/.claude/settings.json`, `~/.claude/skills/`, local 범위 MCP)에만 있거나, 커밋은 했지만 push하지 않았다 → 저장소의 `.claude/`, `.mcp.json`에 커밋하고 push한다. `claude --cloud`는 로컬 체크아웃이 아니라 원격 브랜치를 클론한다.
- **세션을 열 때마다 1~2분씩 기다린다** → SessionStart hook이 매번 `pnpm install`을 무조건 실행한다 → lockfile과 설치 흔적을 비교해 필요할 때만 설치한다. 무거운 도구 설치는 devcontainer 이미지나 클라우드 setup script(캐시됨)로 옮긴다.
- **클라우드 setup script를 바꿨는데 기존 세션에 반영되지 않는다** → setup script는 캐시가 없을 때 Claude Code 시작 전에만 실행되고, 유휴 후 복원된 VM에서는 다시 실행되지 않는다 → 새 세션을 시작하거나 세션 안에서 명령을 직접 실행한다. 매 세션 필요한 준비는 SessionStart hook에 둔다.
- **devcontainer를 재빌드할 때마다 다시 로그인한다** → `~/.claude`만 볼륨으로 잡고 `CLAUDE_CONFIG_DIR`을 설정하지 않아 `~/.claude.json`이 사라진다 → 볼륨 경로와 `CLAUDE_CONFIG_DIR`을 같게 맞추고, Codex는 `CODEX_HOME`을 볼륨에 둔다.
- **Codex Cloud에서 `pnpm install`이 실패한다** → Codex Cloud는 인터넷 접근이 기본적으로 꺼져 있다 → 환경 설정에서 패키지 레지스트리 도메인을 허용한다. Claude Code 클라우드도 네트워크 수준이 **None**이면 설치가 실패한다(기본값 **Trusted**는 npm 레지스트리를 허용).

## 🔗 참고 자료
- 공식 문서
  - [Claude Code development containers](https://code.claude.com/docs/en/devcontainer)
  - [Claude Code hooks: SessionStart, `CLAUDE_ENV_FILE`](https://code.claude.com/docs/en/hooks)
  - [Claude Code cloud environments (setup script, 캐시, 네트워크 수준, `CLAUDE_CODE_REMOTE`)](https://code.claude.com/docs/en/cloud-environments)
  - [Claude Code on the web (`--cloud`, `--teleport`)](https://code.claude.com/docs/en/claude-code-on-the-web)
  - [Codex Cloud](https://learn.chatgpt.com/docs/cloud), [Codex cloud environments](https://learn.chatgpt.com/docs/environments/cloud-environments)
  - [Codex hooks (SessionStart, 신뢰 절차)](https://learn.chatgpt.com/docs/hooks)
  - [Codex auth (`--device-auth`)](https://learn.chatgpt.com/docs/auth)
  - [Dev Containers 스펙](https://containers.dev/), [Claude Code 참고 devcontainer](https://github.com/anthropics/claude-code/tree/main/.devcontainer)
- 이 저장소
  - [도구 레퍼런스 §1 설치·인증, §7.2 Hooks, §9 병렬 실행·클라우드](../../docs/reference/tool-reference.md)
  - 이전 레슨: [01-4 확장 기능](./01-4-extensions.md)
  - 관련 레슨: [00-3 실행 모드](../00-foundations/00-3-execution-modes.md), [05-1 병렬 에이전트](../05-scaling-up/05-1-parallel-agents.md), [05-4 CI/CD 통합](../05-scaling-up/05-4-ci-cd-integration.md)
