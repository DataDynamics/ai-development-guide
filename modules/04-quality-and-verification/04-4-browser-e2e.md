---
id: 04-4
title: "브라우저/E2E 검증: 에이전트가 화면을 확인하게 하기"
module: 04-quality-and-verification
level: L2
duration: 2h 30m
last-verified: 2026-10-07
tools:
  claude-code: 2.1.292
  codex: 0.160.1
---

# 04-4. 브라우저/E2E 검증: 에이전트가 화면을 확인하게 하기

## 🎯 학습 목표
- 에이전트가 브라우저로 실제 화면을 열어 UI 변경을 스스로 확인하게 할 수 있습니다(Playwright MCP).
- UI 변경 전/후 스크린샷을 남기고 에이전트가 그 차이를 판정하는 자가 검증 루프를 만들 수 있습니다.
- Playwright Test로 UI 회귀 테스트(흐름 테스트 + 스크린샷 비교)를 작성하고 `make verify`에 연결할 수 있습니다.
- 불안정한(flaky) E2E 테스트를 에이전트와 함께 trace로 진단할 수 있습니다.

## 📋 사전 준비
- 선행 레슨: [04-1 검증 계층](./04-1-verification-layers.md), [01-3 MCP 서버 연결](../01-environment-setup/01-3-mcp-servers.md)
- 필요 도구/계정: Claude Code 2.1.292, Codex 0.160.1, Node.js 22+, pnpm, Docker
- 실습 저장소 상태: Project B(TaskFlow) B3 완료(웹 UI의 프로젝트 보드·작업 카드 화면이 있습니다) + 04-1의 `make verify`. 로컬에서 `pnpm dev`로 웹(`http://localhost:3000`)과 API가 뜹니다.
- 테스트용 시드 데이터: `pnpm --filter @taskflow/db seed:e2e` 같은 시드 스크립트로 고정된 사용자·프로젝트·작업을 만듭니다(없다면 Step 1에서 만듭니다).

## 💡 개념

### 왜 에이전트에게 "눈"이 필요한가
린트·타입·단위 테스트가 모두 통과해도 화면은 깨질 수 있습니다. 버튼이 다른 요소에 가려지거나, 다크 모드에서 글자가 사라지거나, 모바일 폭에서 레이아웃이 무너집니다. 에이전트는 코드를 읽고 "아마 이렇게 보일 것"이라고 **추측**할 뿐입니다. 사람이 매번 브라우저를 열어 확인하면, 04-1에서 말한 "사람이 병목" 문제가 UI에서 그대로 재현됩니다.

브라우저 검증은 두 가지 용도로 나뉩니다.

| 용도 | 방법 | 언제 | 결과물 |
|---|---|---|---|
| **탐색적 자가 확인** | 에이전트가 MCP로 브라우저를 직접 조작하고 스크린샷을 봅니다 | 구현 중, UI를 바꿀 때마다 | 스크린샷, 관찰 보고 |
| **회귀 테스트** | Playwright Test 코드(흐름 + `toHaveScreenshot`) | 완료 선언 전, CI | 반복 실행 가능한 테스트 파일 |

탐색적 확인은 빠르지만 남지 않습니다. 회귀 테스트는 만들기 번거롭지만 매번 같은 기준으로 돕니다. 대규모 개발에서는 **탐색으로 확인한 것을 회귀 테스트로 고정**하는 흐름이 핵심입니다. 탐색만 하면 다음 변경에서 같은 버그가 돌아오고, 테스트만 쓰면 테스트가 실제 화면과 동떨어집니다.

```text
  UI 변경 요청
       │
       ▼
  ┌──────────────┐   Playwright MCP    ┌───────────────────┐
  │ 에이전트 구현 │ ──────────────────▶ │ 브라우저(localhost) │
  └──────┬───────┘ ◀────────────────── └───────────────────┘
         │          스냅샷·스크린샷
         ▼
  "기대와 다름" ──▶ 다시 수정 (자가 검증 루프)
         │
         ▼ "기대와 같음"
  ┌──────────────────────────────┐
  │ Playwright Test로 고정        │  e2e/*.spec.ts
  │  - 흐름 단언 (role 기반)      │  toHaveScreenshot() 기준 이미지
  │  - 스크린샷 비교              │  ← 기준 갱신은 사람이 승인
  └──────────────┬───────────────┘
                 ▼
         make verify / CI (05-4)
```

### 접근성 스냅샷 vs 스크린샷
Playwright MCP는 두 가지 관찰 수단을 줍니다. `browser_snapshot`은 페이지의 접근성 트리를 텍스트로 돌려주고, 클릭·입력 같은 동작의 기준이 됩니다. `browser_take_screenshot`은 이미지를 돌려주며 시각적 확인용입니다. 문서도 "동작은 스냅샷으로 하십시오"라고 안내합니다. 에이전트에게 **동작은 스냅샷, 판정은 스크린샷**으로 하라고 지시하면 토큰을 아끼면서 정확도가 올라갑니다.

## 👣 따라하기

### Step 1. Playwright Test를 설치하고 실행 환경을 고정합니다
목적: 에이전트가 한 줄로 돌릴 수 있는 E2E 실행 환경(앱 기동 + 시드 데이터 + 브라우저)을 만듭니다.

**Claude Code 레시피**
```bash
claude
```
```text
> apps/e2e 패키지를 새로 만들어 Playwright Test를 설정해 주십시오.
  - @playwright/test 를 devDependency로 추가하고 chromium만 설치하십시오.
  - playwright.config.ts: baseURL http://localhost:3000, webServer로 API와 웹을 띄우고,
    globalSetup에서 e2e 시드를 넣으십시오. trace는 'on-first-retry', retries는 CI에서만 2.
  - package.json scripts: test:e2e, test:e2e:update
  - turbo.json의 test:e2e task와 연결하고, 루트 Makefile의 e2e 타깃이 이걸 실행하게 바꿔.
  - 샘플 테스트 1개(로그인 후 보드 화면 제목 확인)를 만들고 실행해서 통과를 보여 주십시오.
```

**Codex 레시피**
```bash
codex --sandbox workspace-write --ask-for-approval on-request \
  -c sandbox_workspace_write.network_access=true
```
```text
> Create an apps/e2e package with Playwright Test.
  - Add @playwright/test as a devDependency and install chromium only.
  - playwright.config.ts: baseURL http://localhost:3000, webServer starts API and web,
    globalSetup seeds e2e data, trace 'on-first-retry', retries 2 only in CI.
  - scripts test:e2e and test:e2e:update; wire turbo task test:e2e and the root Makefile e2e target.
  - Add one sample test (log in, assert board heading) and run it.
```

> 브라우저 바이너리 다운로드와 패키지 설치에는 네트워크가 필요합니다. 위에서는 이 세션만 `network_access=true`로 열었습니다. 설치가 끝나면 네트워크 없이 다시 시작합니다. Codex 샌드박스 안에서 Chromium 실행과 `localhost` 접속이 허용되는지는 OS와 샌드박스 구현에 따라 다릅니다. 실패하면 오류 메시지를 기록하고 E2E 실행만 사람이 터미널에서 합니다. TODO(verify)

**기대 결과**: 다음 명령이 통과합니다.
```bash
pnpm --filter e2e exec playwright install chromium   # 최초 1회
make e2e                                             # 앱 기동 → 시드 → 테스트 → 종료
```
`apps/e2e/playwright.config.ts`의 핵심 부분은 대략 다음과 같습니다.
```ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  retries: process.env.CI ? 2 : 0,
  use: { baseURL: 'http://localhost:3000', trace: 'on-first-retry' },
  projects: [{ name: 'chromium', use: { ...devices['Desktop Chrome'] } }],
  globalSetup: './global-setup.ts',
  webServer: [
    { command: 'pnpm --filter api start', url: 'http://localhost:4000/health', reuseExistingServer: !process.env.CI },
    { command: 'pnpm --filter web start', url: 'http://localhost:3000', reuseExistingServer: !process.env.CI },
  ],
});
```

### Step 2. 에이전트에 브라우저를 연결합니다 (Playwright MCP)
목적: 에이전트가 구현 중에 브라우저를 직접 열고 조작할 수 있게 합니다.

**Claude Code 레시피**
```bash
claude mcp add playwright -- npx @playwright/mcp@latest --headless --isolated
claude mcp list
```
팀이 함께 쓰려면 `--scope project`를 붙여 `.mcp.json`에 저장합니다. 세션 안에서 `/mcp`로 연결 상태를 확인합니다. 브라우저 도구 호출이 매번 승인을 묻는 것이 번거롭다면 `.claude/settings.json`의 `allow`에 `"mcp__playwright"`를 추가합니다. 단, 이 브라우저로 **외부 사이트를 열면 그 페이지 내용이 컨텍스트에 들어옵니다**(프롬프트 인젝션 경로, [04-3](./04-3-security.md)). 실습에서는 `localhost`만 열게 지시합니다.

Claude Code에는 사람의 Chrome을 쓰는 `--chrome`(Claude in Chrome 연동) 옵션도 있습니다. 로그인 세션이 필요한 화면 확인에 편하지만, 실제 계정과 쿠키를 쓰므로 이 레슨에서는 격리된 Playwright 브라우저를 씁니다.

**Codex 레시피**
```bash
codex mcp add playwright -- npx @playwright/mcp@latest --headless --isolated
codex mcp list
```
세션 안에서 `/mcp`로 도구 목록을 확인합니다. `~/.codex/config.toml`에는 다음과 같이 저장됩니다. MCP 서버가 처음 뜰 때 `npx` 다운로드로 시간이 걸리면 `startup_timeout_sec`(기본 10)을 늘립니다.
```toml
[mcp_servers.playwright]
command = "npx"
args = ["@playwright/mcp@latest", "--headless", "--isolated"]
startup_timeout_sec = 60
```
MCP 서버 프로세스에 Codex 샌드박스가 어떻게 적용되는지(예: `localhost` 접근)는 문서에서 확인하지 못했습니다. TODO(verify) Codex에는 `browser_use` 기능 플래그(`codex features list`에서 stable로 표시)도 있지만, 사용법은 이 레슨에서 다루지 않습니다. TODO(verify)

**기대 결과**: 두 도구에서 다음 프롬프트가 동작합니다.
```text
> http://localhost:3000 에 e2e 시드 계정(e2e@taskflow.test)으로 로그인하고,
  "Demo" 프로젝트 보드를 열어서 접근성 스냅샷으로 컬럼 이름과 카드 수를 보고해 주십시오.
  localhost 외의 주소는 열지 마십시오.
```
에이전트가 `browser_navigate`, `browser_snapshot`, `browser_click` 같은 도구를 호출하고, 시드 데이터와 일치하는 컬럼·카드 수를 보고합니다.

### Step 3. UI 변경을 스크린샷으로 자가 검증합니다
목적: 에이전트가 UI를 고친 뒤 전/후 스크린샷을 비교해 스스로 완료 여부를 판정하게 합니다.

B4 과제 예: **작업 카드에 마감일 배지 추가** — 마감일이 지난 카드는 빨간 "Overdue" 배지, 3일 이내는 주황 "Due soon" 배지.

**Claude Code 레시피**
```text
> 작업 카드에 마감일 배지를 추가해 주십시오. 규칙: 지난 마감 → 빨간 "Overdue", 3일 이내 → 주황 "Due soon".
  절차:
  1) 구현 전에 Playwright MCP로 Demo 보드를 1280x800으로 열고 screenshots/before-board.png 로 저장하십시오.
  2) 구현하고 make verify-fast 를 통과시키십시오.
  3) 같은 화면을 screenshots/after-board.png 로 저장하고, 375px 폭(모바일)도 after-board-mobile.png 로 저장하십시오.
  4) 두 스크린샷을 직접 보고 다음을 판정해서 표로 보고하십시오:
     배지가 시드의 overdue/due-soon 카드에만 보입니까, 카드 제목이 잘리지 않습니까,
     모바일에서 배지가 줄바꿈되며 겹치지 않습니까, 색 대비가 충분한가.
  5) 하나라도 '아니오'면 고치고 3)부터 반복하십시오. 최대 3회.
```

**Codex 레시피**
```text
> Add due-date badges to task cards: past due → red "Overdue", within 3 days → orange "Due soon".
  Steps:
  1) Before coding, open the Demo board at 1280x800 via Playwright MCP and save screenshots/before-board.png.
  2) Implement and make `make verify-fast` pass.
  3) Save screenshots/after-board.png and a 375px-wide screenshots/after-board-mobile.png.
  4) Inspect both screenshots and report a yes/no table: badges only on seeded overdue/due-soon cards,
     titles not truncated, no overlap on mobile, sufficient color contrast.
  5) If any is "no", fix and repeat from step 3, at most 3 times.
```

스크린샷을 에이전트가 직접 볼 수 있는지는 도구와 설정에 따라 다릅니다. 스크린샷 내용이 판정에 반영되지 않는 것 같으면 이미지를 명시적으로 첨부해서 다시 판정시킵니다.
```bash
# Codex: 이미지 첨부 옵션(-i)으로 판정만 따로 요청
codex exec -i screenshots/after-board.png -i screenshots/after-board-mobile.png \
  "Judge these screenshots against the badge rules in docs/tasks/B4-06-due-badges.md. Answer as a yes/no table."
```
Claude Code에서는 프롬프트에 파일 경로를 주고 "이 이미지를 열어서 판정해 주십시오"라고 하면 Read 도구로 이미지를 읽습니다.

**기대 결과**:
- `screenshots/` 아래에 before/after/mobile 이미지 3장과 판정표가 있습니다.
- 사람이 이미지를 직접 열어 에이전트의 판정과 비교합니다. 에이전트가 "문제없음"이라고 했는데 사람이 보기에 문제가 있다면 그 사례를 기록합니다(HW 자료). 에이전트의 시각 판정은 레이아웃 붕괴·누락은 잘 잡지만, 미세한 정렬이나 색감은 놓치기 쉽습니다.
- `screenshots/`는 탐색용 산출물이므로 `.gitignore`에 넣고, PR 본문에만 첨부합니다.

### Step 4. 탐색 결과를 UI 회귀 테스트로 고정합니다
목적: Step 3에서 확인한 동작을 Playwright Test로 남겨 다음 변경에서도 자동으로 검증되게 합니다.

**Claude Code 레시피**
```text
> 방금 확인한 마감일 배지 동작을 apps/e2e/tests/task-badges.spec.ts 로 고정해 주십시오.
  - 로케이터는 getByRole / getByText 기준으로, CSS 클래스 선택자는 쓰지 마십시오.
  - 시간 의존성을 없애기 위해 page.clock 으로 현재 시각을 시드 기준 날짜에 고정하십시오.
  - 흐름 단언: overdue 카드에만 "Overdue", due-soon 카드에만 "Due soon" 배지.
  - 시각 단언: 보드 영역에 toHaveScreenshot('board-badges.png'), 데스크톱/모바일 두 뷰포트.
  - 기준 이미지는 test:e2e:update 로 만들되, 만든 뒤 저에게 이미지 경로를 알려주고 승인을 받으십시오.
```

**Codex 레시피**
```text
> Lock the due-badge behavior into apps/e2e/tests/task-badges.spec.ts.
  - Use getByRole/getByText locators; no CSS class selectors.
  - Freeze time with page.clock to the seed reference date.
  - Behavior assertions: "Overdue" only on overdue cards, "Due soon" only on due-soon cards.
  - Visual assertion: toHaveScreenshot('board-badges.png') on the board region for desktop and mobile.
  - Generate baselines with test:e2e:update, then list the baseline image paths for my approval.
```

**기대 결과**: 다음과 비슷한 테스트가 생깁니다.
```ts
import { test, expect } from '@playwright/test';

test.describe('task due badges', () => {
  test.beforeEach(async ({ page }) => {
    await page.clock.setFixedTime(new Date('2026-10-01T09:00:00Z'));
    await page.goto('/projects/demo/board');
  });

  test('overdue and due-soon cards show the right badge', async ({ page }) => {
    const overdue = page.getByRole('article', { name: 'Fix login redirect' });
    await expect(overdue.getByText('Overdue')).toBeVisible();
    const later = page.getByRole('article', { name: 'Write release notes' });
    await expect(later.getByText(/Overdue|Due soon/)).toHaveCount(0);
  });

  test('board visual', async ({ page }) => {
    await expect(page.getByRole('main')).toHaveScreenshot('board-badges.png');
  });
});
```
- 기준 이미지가 `apps/e2e/tests/task-badges.spec.ts-snapshots/` 아래에 생깁니다. 사람이 이미지를 열어 승인한 뒤 커밋합니다.
- **회귀 확인**: 배지 색을 일부러 바꾸고 `make e2e`를 실행하면 `toHaveScreenshot`이 실패하고 diff 이미지가 `test-results/`에 생깁니다. 확인 후 되돌립니다.
- 이 레슨의 숙제인 UI 회귀 테스트 3개 중 1개가 완성됐습니다.

### Step 5. E2E를 검증 계층에 넣고 불안정성을 진단합니다
목적: E2E를 `make verify`에 연결하고, 실패했을 때 에이전트가 trace로 원인을 찾게 합니다.

`AGENTS.md`의 검증 섹션(04-1)에 다음 규칙을 추가합니다.
```markdown
- UI를 바꾼 작업은 `make e2e`를 통과해야 완료입니다.
- `toHaveScreenshot` 기준 이미지(`*-snapshots/`)는 에이전트가 임의로 갱신하지 않습니다.
  갱신이 필요하면 diff 이미지 경로와 이유를 보고하고 사람의 승인을 받습니다.
- 테스트를 통과시키려고 `waitForTimeout`이나 고정 sleep을 추가하지 않습니다.
```

실패 진단 실습: 시드 데이터 로딩이 끝나기 전에 단언하는 테스트를 일부러 하나 만들어 간헐 실패를 유도합니다. 여러 번 반복 실행해서 불안정성을 드러냅니다.
```bash
pnpm --filter e2e exec playwright test tests/task-badges.spec.ts --repeat-each=10 --trace=on
```

**Claude Code 레시피**
```text
> 위 명령에서 간헐적으로 실패한 테스트를 진단해 주십시오. test-results/ 의 trace.zip 과 에러 로그를 근거로
  원인을 가설로 세우고(경쟁 조건, 시드 타이밍, 애니메이션 등), waitForTimeout 없이 고치십시오.
  고친 뒤 --repeat-each=10 으로 다시 돌려서 10/10 통과를 보여 주십시오.
```

**Codex 레시피**
```bash
codex exec --sandbox workspace-write \
  "Diagnose the flaky test from the last run using test-results/ traces and logs. \
State hypotheses (race, seed timing, animation), fix without waitForTimeout, \
then re-run with --repeat-each=10 and show 10/10 passes." \
  -o reports/04-4-flaky.codex.md
```

사람이 trace를 직접 보려면 다음을 실행합니다.
```bash
pnpm --filter e2e exec playwright show-trace test-results/<테스트 폴더>/trace.zip
pnpm --filter e2e exec playwright show-report
```

**기대 결과**:
- 에이전트가 `await expect(...).toBeVisible()` 같은 자동 대기 단언이나 네트워크 응답 대기로 고치고, 고정 sleep을 넣지 않습니다. 넣었다면 `AGENTS.md` 규칙 위반으로 되돌립니다.
- `make verify`가 이제 린트 → 타입 → 단위 → 통합 → E2E를 모두 실행합니다.
- 스크린샷 비교는 OS·폰트·GPU에 따라 픽셀이 달라집니다. 기준 이미지는 CI와 같은 환경(예: Linux 컨테이너)에서 만들어야 한다는 점을 기록해 둡니다. CI 연결은 [05-4](../05-scaling-up/05-4-ci-cd-integration.md)에서 합니다.

## ✅ 체크포인트
- [ ] `make e2e`가 앱 기동 → 시드 → 테스트 → 종료까지 한 줄로 동작합니다.
- [ ] Claude Code와 Codex 양쪽에 Playwright MCP가 연결되어 `/mcp`에서 보입니다.
- [ ] UI 변경의 before/after/mobile 스크린샷과 에이전트 판정표를 만들고, 사람이 판정을 대조했습니다.
- [ ] `task-badges.spec.ts`에 흐름 단언과 `toHaveScreenshot`이 있고, 기준 이미지는 사람이 승인 후 커밋했습니다.
- [ ] 의도적인 시각 변경에 테스트가 실패하는 것을 확인했습니다.
- [ ] `AGENTS.md`에 기준 이미지 갱신 금지·sleep 금지 규칙이 있고, 불안정한 테스트를 trace로 진단해 고쳤습니다.

## 🏠 숙제
| # | 유형 | 난이도 | 과제 | 완료 조건 |
|---|---|---|---|---|
| HW1 | 🔁 Reproduce | ★ | Step 3을 재현합니다. UI 변경 하나를 골라 에이전트가 전/후 스크린샷으로 자가 검증하게 합니다 | before/after/mobile 스크린샷, 에이전트 판정표, 사람의 대조 결과(일치/불일치 항목)를 제출합니다 |
| HW2 | 🛠 Apply | ★★ | TaskFlow 웹에 UI 회귀 테스트 3개를 작성합니다(예: 마감일 배지, 작업 생성 모달, 보드 드래그 앤 드롭) | `apps/e2e/tests/`에 테스트 3개가 있고 각각 흐름 단언과 `toHaveScreenshot`을 포함합니다. `--repeat-each=10`에서 30/30 통과한 로그와, 각 테스트가 잡아내는 회귀를 일부러 만들어 실패시킨 로그를 첨부합니다 |
| HW3 | 🚀 Challenge | ★★★ | 같은 UI 회귀 테스트 작성 과제를 Claude Code와 Codex에 각각 맡기고, 탐색(MCP) → 테스트 고정 흐름을 비교합니다. 추가로 Linux 컨테이너에서 기준 이미지를 생성하는 스크립트를 만듭니다 | 도구별 "MCP 도구 호출 수 / 시도 횟수 / 불안정성(10회 반복 통과율) / 사람 개입" 비교표를 제출합니다. 컨테이너 스크립트로 만든 기준 이미지가 로컬(macOS 등) 생성본과 어떻게 다른지 diff 이미지로 보여줍니다 |

제출: `hw/04-4` 브랜치, `submissions/04-4.md` (템플릿: [homework-template.md](../../docs/design/homework-template.md))

## ⚠️ 흔한 실수
- **에이전트가 기준 이미지를 갱신해서 테스트를 "통과"시켰습니다** → `--update-snapshots`(`test:e2e:update`)로 실패를 새 기준으로 덮었습니다 → 기준 이미지 갱신 금지 규칙을 `AGENTS.md`에 넣고, `*-snapshots/` 변경은 PR 리뷰에서 반드시 이미지로 확인합니다(04-2 `REVIEW.md` 항목에 추가).
- **로컬에서는 통과하는데 CI에서 스크린샷 비교가 실패합니다** → OS·폰트·렌더링 차이로 픽셀이 다릅니다 → 기준 이미지를 CI와 같은 Linux 환경에서 생성합니다. 시각 비교 범위를 화면 전체가 아니라 필요한 영역(`getByRole('main')` 등)으로 좁힙니다.
- **E2E가 간헐적으로 실패하고 에이전트가 sleep을 넣습니다** → 시드 타이밍, 애니메이션, 현재 시각 의존 같은 원인을 고치지 않고 대기 시간으로 덮었습니다 → `page.clock`으로 시간을 고정하고, 자동 대기 단언을 쓰고, `--repeat-each`로 안정성을 확인합니다. trace를 근거로 원인을 찾게 합니다.
- **에이전트가 브라우저로 외부 사이트를 열었습니다** → "참고할 UI를 찾아보십시오" 같은 열린 지시로 MCP 브라우저가 외부 페이지를 읽었습니다 → 프롬프트에 "localhost 외의 주소는 열지 마십시오"를 넣고, 외부 페이지 내용은 신뢰할 수 없는 입력으로 취급합니다([04-3](./04-3-security.md)).
- **Codex 세션에서 브라우저 설치나 앱 기동이 실패합니다** → `workspace-write` 샌드박스가 기본적으로 네트워크를 막습니다 → 설치 단계만 `-c sandbox_workspace_write.network_access=true`로 열거나, 설치는 사람이 미리 해 둡니다.

## 🔗 참고 자료
- [Playwright MCP (microsoft/playwright-mcp)](https://github.com/microsoft/playwright-mcp)
- [Playwright Test 문서](https://playwright.dev/docs/intro)
- [Playwright 시각 비교 (`toHaveScreenshot`)](https://playwright.dev/docs/test-snapshots)
- [Playwright Trace Viewer](https://playwright.dev/docs/trace-viewer)
- [Playwright Clock](https://playwright.dev/docs/clock)
- [Claude Code MCP](https://code.claude.com/docs/en/mcp)
- [Codex MCP](https://learn.chatgpt.com/docs/extend/mcp)
- [Codex approvals & security](https://learn.chatgpt.com/docs/agent-approvals-security)
- 이 저장소: [도구 레퍼런스 §6 MCP](../../docs/reference/tool-reference.md), [01-3 MCP 서버 연결](../01-environment-setup/01-3-mcp-servers.md), [04-1 검증 계층](./04-1-verification-layers.md), [04-2 교차 리뷰](./04-2-cross-review.md), [04-3 보안](./04-3-security.md), [05-4 CI/CD 통합](../05-scaling-up/05-4-ci-cd-integration.md)
