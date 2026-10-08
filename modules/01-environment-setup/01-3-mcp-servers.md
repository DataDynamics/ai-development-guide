---
id: 01-3
title: "MCP 서버 연결 (GitHub, DB, 문서, 브라우저)"
module: 01-environment-setup
level: L2
duration: 2h
last-verified: 2026-10-07
tools:
  claude-code: 2.1.292
  codex: 0.160.1
---

# 01-3. MCP 서버 연결 (GitHub, DB, 문서, 브라우저)

## 🎯 학습 목표
- MCP가 에이전트의 "도구 목록"을 어떻게 넓히는지, 그리고 그만큼 무엇이 위험해지는지 설명할 수 있습니다.
- Claude Code와 Codex에 원격(HTTP) MCP 서버와 로컬(stdio) MCP 서버를 각각 연결할 수 있습니다.
- 개인 범위와 프로젝트 범위를 구분해, 팀이 공유할 MCP 구성을 저장소에 커밋할 수 있습니다.
- MCP 서버 도입 전 보안 검토(권한 범위, 자격 증명, 데이터 유출 경로)를 문서로 남길 수 있습니다.

## 📋 사전 준비
- 선행 레슨: [01-2 권한·샌드박스·승인 모드](./01-2-permissions-and-sandbox.md)
- 필요 도구/계정
  - Claude Code 2.1.292 이상, Codex 0.160.1 이상
  - Node.js 22 이상(`npx`), Docker(로컬 PostgreSQL용)
  - GitHub 계정과 **읽기 권한만 준** fine-grained Personal Access Token(대상: TaskFlow 저장소 1개, 권한: Contents·Issues·Pull requests Read-only)
- 실습 저장소 상태: 01-2까지 완료한 TaskFlow 저장소가 GitHub에 push되어 있습니다. 로컬 DB는 아래처럼 띄웁니다.

```yaml
# docker-compose.yml (TaskFlow 루트)
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: taskflow
      POSTGRES_PASSWORD: taskflow
      POSTGRES_DB: taskflow
    ports: ["5432:5432"]
```

```bash
docker compose up -d postgres
# 에이전트 전용 읽기 전용 계정을 만듭니다
docker compose exec -T postgres psql -U taskflow -d taskflow <<'SQL'
CREATE ROLE agent_ro LOGIN PASSWORD 'agent_ro';
GRANT CONNECT ON DATABASE taskflow TO agent_ro;
GRANT USAGE ON SCHEMA public TO agent_ro;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO agent_ro;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO agent_ro;
SQL
```

## 💡 개념

### MCP = 에이전트에게 "새 도구"를 꽂는 표준 소켓

에이전트는 기본적으로 파일 읽기/쓰기와 셸 명령만 쓸 수 있습니다. MCP(Model Context Protocol) 서버를 연결하면, 서버가 제공하는 도구(예: `list_pull_requests`, `query`, `browser_navigate`)가 에이전트의 도구 목록에 추가됩니다. 같은 서버를 Claude Code와 Codex가 함께 쓸 수 있다는 점이 핵심입니다.

```mermaid
flowchart LR
    subgraph Agent["에이전트 세션 (Claude Code / Codex)"]
        LLM[모델] --> TL[도구 목록]
    end
    TL -- "stdio (로컬 프로세스)" --> DB["DB MCP<br/>npx @bytebase/dbhub"]
    TL -- "HTTP (원격)" --> GH["GitHub MCP<br/>api.githubcopilot.com/mcp/"]
    TL -- "stdio" --> BR["브라우저 MCP<br/>(Playwright 등, 04-4)"]
    DB --> PG[(PostgreSQL<br/>agent_ro 계정)]
    GH --> REPO[(GitHub 저장소<br/>읽기 전용 토큰)]
```

| 전송 방식 | 동작 | 예 | 주의 |
|---|---|---|---|
| stdio | 에이전트가 로컬 프로세스를 띄워 표준 입출력으로 통신 | DB, 로컬 파일 도구 | 서버 바이너리가 내 PC 권한으로 실행됩니다 |
| HTTP (streamable) | 원격 URL에 요청 | GitHub, Sentry, 사내 서비스 | 토큰 관리, 네트워크 허용 목록 |
| SSE | (deprecated) | – | 새로 쓰지 않습니다 |

### 두 도구의 MCP 구성 비교

| 항목 | Claude Code | Codex |
|---|---|---|
| stdio 추가 | `claude mcp add <name> -- <command> [args...]` | `codex mcp add <name> -- <command>...` |
| HTTP 추가 | `claude mcp add --transport http <name> <url>` | `codex mcp add <name> --url <URL>` |
| 토큰 | `--header "Authorization: Bearer ..."` 또는 `.mcp.json`의 `${VAR}` | `--bearer-token-env-var <ENV_VAR>` |
| 개인 범위 | `--scope local`(기본) / `user` → `~/.claude.json` | `~/.codex/config.toml` |
| 팀 공유 | `--scope project` → 저장소 루트 `.mcp.json` | `.codex/config.toml` (신뢰한 프로젝트만) |
| 목록·삭제 | `claude mcp list`, `get`, `remove` | `codex mcp list`, `get`, `remove` |
| 세션 안 상태 | `/mcp` | `/mcp` (`/mcp verbose`) |
| 도구 단위 제한 | 권한 규칙 `mcp__<server>__<tool>` | `enabled_tools`, `disabled_tools` |

### 왜 대규모 개발에서 중요한가

1. **사람이 복사·붙여넣기 하던 컨텍스트를 에이전트가 직접 가져옵니다.** PR 리뷰 코멘트, 실제 DB 스키마, 에러 트래커의 스택 트레이스를 에이전트가 직접 읽으면 "설명하다 틀리는" 단계가 사라집니다.
2. **검증 수단이 늘어납니다.** "마이그레이션 후 테이블이 생겼습니까"를 에이전트가 DB에 직접 물어볼 수 있습니다. Verify, Don't Trust 원칙을 에이전트 스스로 실행하게 됩니다.
3. **그만큼 공격 표면도 커집니다.** MCP 서버는 외부 콘텐츠(이슈 본문, 웹 페이지)를 에이전트에게 넣는 통로입니다. 문서도 "외부 콘텐츠를 가져오는 서버는 프롬프트 인젝션 위험에 노출될 수 있으니 신뢰하는 서버만 연결하십시오"라고 경고합니다. 쓰기 권한이 있는 토큰과 결합하면 사고 규모가 커집니다.
4. **컨텍스트 예산을 씁니다.** 연결한 서버의 도구 설명과 결과는 컨텍스트를 차지합니다. Claude Code는 MCP 도구 출력이 10,000 토큰을 넘으면 경고하고 기본 25,000 토큰에서 자릅니다(`MAX_MCP_OUTPUT_TOKENS`로 조정). "일단 다 연결"은 비용과 정확도를 모두 떨어뜨립니다.

### 최소 권한 원칙 체크리스트

| 질문 | TaskFlow 적용 |
|---|---|
| 이 서버가 정말 필요한가? 셸 명령(`gh`, `psql`)으로 충분하지 않은가? | GitHub는 PR 코멘트·리뷰 스레드 탐색이 잦아서 MCP 사용, DB는 스키마 탐색용 |
| 토큰 권한은 최소입니까? | GitHub: 저장소 1개, Read-only. DB: `agent_ro` SELECT 전용 |
| 운영 데이터에 닿습니까? | 로컬 Docker DB만. 운영 DSN은 절대 연결하지 않습니다 |
| 서버 코드의 출처는 신뢰할 수 있습니까? | 공식/유명 조직 배포본, 버전 고정 여부 확인 |
| 자격 증명이 저장소에 남습니까? | 환경 변수 참조(`${VAR}`, `--bearer-token-env-var`)만 커밋 |

## 👣 따라하기

### Step 1. 연결할 서버를 고르고 보안 검토 초안을 씁니다
도구 독립적인 단계입니다. 어떤 서버를 왜 연결하는지 먼저 문서로 정하고, 에이전트에게 검토를 맡깁니다.

```bash
mkdir -p docs/ops
cat > docs/ops/mcp-review.md <<'EOF'
# TaskFlow MCP 보안 검토

| 서버 | 용도 | 전송 | 자격 증명 | 권한 범위 | 데이터 유출 경로 | 결정 |
|---|---|---|---|---|---|---|
| github | PR·이슈·리뷰 코멘트 조회 | HTTP | GITHUB_PAT (fine-grained) | TaskFlow 저장소 Read-only | 이슈 본문의 프롬프트 인젝션 | 도입 |
| db | 로컬 스키마·데이터 조회 | stdio | agent_ro (로컬 전용) | SELECT only | 로컬 데이터만 | 도입 |
| browser | UI 자가 검증 | stdio | 없음 | 로컬 브라우저 | 임의 웹 페이지 방문 | 04-4에서 결정 |
EOF
```

**Claude Code 레시피**
```bash
claude --permission-mode plan
> docs/ops/mcp-review.md를 검토하십시오. 각 서버별로 빠진 위험 요소와
> 권한을 더 줄일 수 있는 방법을 제안하십시오. 파일은 고치지 말고 제안만 하십시오.
```

**Codex 레시피**
```bash
codex --profile review "docs/ops/mcp-review.md를 검토하십시오. 각 서버별로 빠진 위험 요소와 권한을 더 줄일 수 있는 방법을 제안하십시오. 파일은 고치지 마십시오."
```
(`review` 프로필은 01-2에서 만든 `~/.codex/review.config.toml` 읽기 전용 프로필입니다.)

**기대 결과**: 두 도구가 "GitHub 토큰 만료 기간 설정", "DB 서버가 쓰기 쿼리를 거부하는지 확인", "도구 단위 허용 목록" 같은 보완점을 제안합니다. 받아들일 제안을 표에 반영합니다.

### Step 2. GitHub MCP(원격 HTTP)를 개인 범위로 연결합니다
토큰은 셸 환경 변수로만 두고, 명령줄이나 파일에 원문을 남기지 않습니다.

```bash
export GITHUB_PAT="<읽기 전용 fine-grained 토큰>"   # 셸 프로필이나 비밀 관리 도구로 주입
```

**Claude Code 레시피**
```bash
claude mcp add --transport http github https://api.githubcopilot.com/mcp/ \
  --header "Authorization: Bearer ${GITHUB_PAT}"
claude mcp list
claude mcp get github
```
> 이 방식은 셸이 `${GITHUB_PAT}`를 펼친 **값**을 `~/.claude.json`(local 범위)에 저장합니다. 파일은 저장소 밖에 있지만 평문입니다. 팀 공유용으로는 Step 4의 `.mcp.json` 변수 참조 방식을 씁니다.

**Codex 레시피**
```bash
codex mcp add github --url https://api.githubcopilot.com/mcp/ --bearer-token-env-var GITHUB_PAT
codex mcp list
codex mcp get github
```
> Codex는 토큰 값이 아니라 **환경 변수 이름**을 저장합니다. Codex를 띄우는 셸에 `GITHUB_PAT`가 있어야 합니다.

**기대 결과**: 두 도구의 `mcp list`에 `github`가 보입니다. 세션 안에서 `/mcp`를 열면 연결 상태와 제공 도구 목록이 표시됩니다.

### Step 3. DB MCP(로컬 stdio)를 연결합니다
읽기 전용 계정으로 로컬 PostgreSQL에 붙는 서버를 띄웁니다. 여기서는 Claude Code 문서의 예시인 DBHub(`@bytebase/dbhub`)를 씁니다.

**Claude Code 레시피**
```bash
claude mcp add --transport stdio db -- npx -y @bytebase/dbhub \
  --dsn "postgresql://agent_ro:agent_ro@localhost:5432/taskflow"
```
- `--` 뒤는 서버 명령입니다. `--`를 빠뜨리면 `--dsn`을 Claude 자신의 옵션으로 해석합니다.
- 환경 변수를 넘길 때는 `-e KEY=value`(`--env`) 바로 뒤에 서버 이름을 두지 않습니다. 이름까지 `KEY=value`로 읽히므로 `--transport` 같은 다른 옵션을 사이에 둡니다.

**Codex 레시피**
```bash
codex mcp add db -- npx -y @bytebase/dbhub \
  --dsn "postgresql://agent_ro:agent_ro@localhost:5432/taskflow"
```
결과는 `~/.codex/config.toml`에 다음처럼 저장됩니다.
```toml
[mcp_servers.db]
command = "npx"
args = ["-y", "@bytebase/dbhub", "--dsn", "postgresql://agent_ro:agent_ro@localhost:5432/taskflow"]
```
`npx` 첫 실행은 패키지 다운로드 때문에 느립니다. 시작 시간 초과가 나면 `startup_timeout_sec`(기본 10초)를 늘립니다.
```toml
[mcp_servers.db]
startup_timeout_sec = 30
```

> 이 DSN은 로컬 Docker 전용 계정이라 평문으로 둡니다. 운영·공유 DB라면 DSN을 환경 변수로 넘기는 방식을 서버 문서에서 확인하고 씁니다. TODO(verify): DBHub의 환경 변수 기반 DSN 설정 방법.

**기대 결과**: 두 도구에서 `db` 서버가 연결됩니다. Claude는 `MCP_TIMEOUT=30000 claude`처럼 시작 시간 제한을 늘릴 수 있습니다.

### Step 4. 팀이 공유할 구성을 저장소에 커밋합니다
개인 PC에서만 되는 MCP는 팀원·CI·클라우드 세션에서 재현되지 않습니다. 공유할 서버는 프로젝트 범위로 옮깁니다.

**Claude Code 레시피**

`.mcp.json`은 `${VAR}`, `${VAR:-default}` 확장을 지원하므로 토큰 값 대신 변수 이름을 커밋합니다.
```bash
claude mcp remove github          # 개인(local) 범위 항목 제거
cat > .mcp.json <<'EOF'
{
  "mcpServers": {
    "github": {
      "type": "http",
      "url": "https://api.githubcopilot.com/mcp/",
      "headers": { "Authorization": "Bearer ${GITHUB_PAT}" }
    },
    "db": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@bytebase/dbhub", "--dsn", "${TASKFLOW_AGENT_DSN:-postgresql://agent_ro:agent_ro@localhost:5432/taskflow}"]
    }
  }
}
EOF
claude mcp remove db
```
> CLI로 만들려면 `claude mcp add --transport http github --scope project https://api.githubcopilot.com/mcp/`처럼 `--scope project`를 붙입니다. 이 경우 헤더 값은 직접 `${GITHUB_PAT}` 형태로 편집합니다.

MCP 도구에도 01-2의 권한 규칙을 적용합니다. `.claude/settings.json`의 `permissions`에 다음을 병합합니다.
```json
{
  "permissions": {
    "allow": ["mcp__github__get_*", "mcp__github__list_*", "mcp__db"],
    "deny":  ["mcp__github__create_*", "mcp__github__merge_*", "mcp__github__delete_*"]
  }
}
```
TODO(verify): GitHub MCP 서버의 실제 도구 이름은 `/mcp`의 도구 목록에서 확인하고 패턴을 맞춥니다.

**Codex 레시피**

프로젝트 `.codex/config.toml`(01-2에서 만든 파일)에 서버를 추가합니다. 신뢰한 프로젝트에서만 로드됩니다.
```toml
[mcp_servers.github]
url = "https://api.githubcopilot.com/mcp/"
bearer_token_env_var = "GITHUB_PAT"
disabled_tools = ["create_pull_request", "merge_pull_request", "delete_file"]

[mcp_servers.db]
command = "npx"
args = ["-y", "@bytebase/dbhub", "--dsn", "postgresql://agent_ro:agent_ro@localhost:5432/taskflow"]
startup_timeout_sec = 30
```
`url`, `bearer_token_env_var`, `command`, `args`는 0.160.1에서 `codex mcp add`가 실제로 생성하는 키와 같습니다(Step 2·3에서 만든 `~/.codex/config.toml`을 열어 비교해 봅니다). TODO(verify): `disabled_tools`에 넣을 GitHub MCP 도구 이름은 `/mcp verbose` 출력과 맞춥니다.

개인 범위 항목은 정리합니다.
```bash
codex mcp remove github
codex mcp remove db
```

**기대 결과**
- Claude를 새로 시작하면 `.mcp.json` 서버 사용 승인을 처음 한 번 묻습니다. 승인 기록을 초기화하려면 `claude mcp reset-project-choices`를 씁니다.
- Codex는 프로젝트 신뢰 후 `/mcp`에 `github`, `db`가 보입니다.
- `git diff`에 토큰 원문이 없어야 합니다.

```bash
git add .mcp.json .codex/config.toml .claude/settings.json docs/ops/mcp-review.md docker-compose.yml
git diff --cached | grep -iE "ghp_|github_pat_" && echo "토큰 유출! 커밋 중단" || git commit -m "chore: add shared MCP servers (github, db)"
```

### Step 5. 실제 작업에 써 보고 경계를 시험합니다
연결만으로는 의미가 없습니다. TaskFlow 작업에 쓰고, 막혀야 할 것이 막히는지 봅니다.

**Claude Code 레시피**
```bash
claude
> /mcp
> GitHub MCP로 TaskFlow 저장소의 열린 PR과 이슈를 나열하고, 라벨별로 묶어 주십시오.
> db MCP로 public 스키마의 테이블과 컬럼을 조회해서 docs/ops/schema-snapshot.md로 정리하십시오.
> db MCP로 users 테이블에 테스트 행을 하나 INSERT 해 보십시오.
```

**Codex 레시피**
```bash
codex
> /mcp verbose
> Use the github MCP server to list open PRs and issues in the TaskFlow repo, grouped by label.
> Use the db MCP server to list tables and columns in the public schema and write docs/ops/schema-snapshot.md.
> Use the db MCP server to INSERT a test row into the users table.
```

**기대 결과**
- 조회 작업은 두 도구 모두 성공하고 `docs/ops/schema-snapshot.md`가 생깁니다(B0 시점이라 테이블이 없으면 "테이블 없음"이 정상입니다).
- `INSERT`는 `agent_ro` 권한 부족으로 **DB가 거부**해야 합니다. 에이전트가 다른 계정이나 `docker compose exec psql`로 우회하려 하면 거절합니다. 이 우회 시도 자체가 보안 검토서에 적을 만한 관찰입니다.
- 쓰기 계열 GitHub 도구는 Claude에서는 deny 규칙으로, Codex에서는 `disabled_tools`로 목록에서 빠져 있어야 합니다.

> 문서·브라우저 MCP: 라이브러리 문서 검색용 서버(예: Codex 문서 예시의 `context7`)와 Playwright 같은 브라우저 서버도 같은 절차로 연결합니다. 브라우저 서버는 [04-4 브라우저/E2E 검증](../04-quality-and-verification/04-4-browser-e2e.md)에서 실제로 다룹니다.

## ✅ 체크포인트
- [ ] `docs/ops/mcp-review.md`에 서버별 용도·자격 증명·권한 범위·유출 경로·결정이 있습니다.
- [ ] GitHub(HTTP)와 DB(stdio) 서버가 두 도구 모두에서 `/mcp`에 보입니다.
- [ ] 팀 공유 구성(`.mcp.json`, `.codex/config.toml`)에 토큰 원문이 없습니다.
- [ ] MCP 도구에 권한 규칙(Claude) 또는 `disabled_tools`(Codex)를 적용했습니다.
- [ ] 읽기 전용 DB 계정으로 쓰기 쿼리가 거부되는 것을 확인했습니다.

## 🏠 숙제
| # | 유형 | 난이도 | 과제 | 완료 조건 |
|---|---|---|---|---|
| HW1 | 🔁 Reproduce | ★ | MCP 서버 2개(GitHub, DB)를 두 도구에 연결하고 사용합니다 | 두 도구의 `mcp list` 출력과 `/mcp` 화면, Step 5의 조회 결과 파일, `INSERT` 거부 로그가 제출 문서에 있습니다 |
| HW2 | 🛠 Apply | ★★ | 본인 프로젝트(또는 TaskFlow 전체 로드맵)에 필요한 MCP 목록과 보안 검토서를 작성합니다 | 후보 서버 4개 이상에 대해 Step 1 표 형식(용도·전송·자격 증명·권한 범위·유출 경로·결정)을 채웠고, 도입하지 않기로 한 서버의 대안(CLI 등)과 이유가 있습니다 |
| HW3 | 🚀 Challenge | ★★★ | 프롬프트 인젝션 모의 실험: 테스트 저장소 이슈 본문에 숨은 지시를 넣고 에이전트가 GitHub MCP로 읽게 합니다 | 실험 저장소·이슈 링크, 두 도구의 반응 기록, 권한 규칙/`disabled_tools`/읽기 전용 토큰 중 어떤 방어선이 작동했는지와 개선한 설정 diff가 있습니다. 실제 운영 저장소에서는 하지 않습니다 |

제출: `hw/01-3` 브랜치, `submissions/01-3.md` (템플릿: [docs/design/homework-template.md](../../docs/design/homework-template.md))

## ⚠️ 흔한 실수
- **`claude mcp add`가 서버 옵션을 자기 옵션으로 해석해 오류가 납니다** → 서버 명령 앞에 `--`를 빠뜨렸습니다 → `claude mcp add --transport stdio db -- npx -y ...`처럼 `--`를 넣습니다. `--env` 바로 뒤에 서버 이름을 두는 실수도 확인합니다.
- **팀원 PC에서 MCP가 안 보입니다** → 기본 범위가 `local`이라 `~/.claude.json`에만 저장됐습니다 → `--scope project`(→ `.mcp.json`)나 `.codex/config.toml`로 옮겨 커밋합니다. 클라우드 세션도 저장소에 커밋된 구성만 읽습니다.
- **`.mcp.json`의 `${ANTHROPIC_API_KEY}`가 빈 값이 됩니다** → 원격 `url`·`headers`에서는 자격 증명 성격의 예약 변수를 빈 값으로 읽습니다 → `GITHUB_PAT`, `TASKFLOW_AGENT_DSN`처럼 자기만의 변수 이름을 씁니다.
- **Codex에서 서버가 시작 시간 초과로 빠집니다** → `npx` 첫 다운로드가 기본 10초를 넘습니다 → `startup_timeout_sec = 30`으로 늘리거나 패키지를 미리 설치합니다. 도구 실행이 길면 `tool_timeout_sec`(기본 60)를 조정합니다.
- **쓰기 권한 토큰을 "편해서" 연결했습니다** → 프롬프트 인젝션 한 번이 PR 머지·브랜치 삭제로 이어질 수 있습니다 → 기본은 읽기 전용 토큰, 쓰기는 별도 서버 이름과 ask/deny 규칙으로 분리합니다.

## 🔗 참고 자료
- 공식 문서
  - [Claude Code MCP (범위, `.mcp.json`, 변수 확장, 출력 제한)](https://code.claude.com/docs/en/mcp)
  - [Claude Code permissions (`mcp__<server>__<tool>` 규칙)](https://code.claude.com/docs/en/permissions)
  - [Codex MCP (`codex mcp add`, `[mcp_servers.*]`)](https://learn.chatgpt.com/docs/extend/mcp)
  - [Model Context Protocol](https://modelcontextprotocol.io/)
- 이 저장소
  - [도구 레퍼런스 §6 MCP](../../docs/reference/tool-reference.md)
  - 이전 레슨: [01-2 권한·샌드박스·승인 모드](./01-2-permissions-and-sandbox.md) · 다음 레슨: [01-4 확장 기능](./01-4-extensions.md)
  - 관련 레슨: [04-3 보안](../04-quality-and-verification/04-3-security.md), [04-4 브라우저/E2E 검증](../04-quality-and-verification/04-4-browser-e2e.md), [01-5 재현 가능한 개발 환경](./01-5-reproducible-environment.md)
