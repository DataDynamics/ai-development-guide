# 학습용 PNG 그림 목록

7개 모듈의 핵심 개념을 21개 PNG로 설명합니다. 각 레슨의 개념 시작 부분에서 그림과 읽는 순서를 함께 확인합니다. 모듈 README에는 대표 그림을 배치합니다.

그림은 **datadynamics architecture style 1.0**을 바탕으로 내장 `image_gen` 도구에서 생성합니다. 문서·폴더·모니터·체크리스트처럼 익숙한 사물로 개발 과정과 산출물을 표현합니다. 생성일은 2026-10-08입니다.

- 작성 기준은 [이미지 스타일](./architecture-image-style.md)에서 확인합니다.
- 재생성 입력은 [공통·개별 프롬프트와 수정 기록](./illustration-prompts.json)에서 확인합니다.
- 원본은 각 모듈의 `images/`에 보관합니다. 아래 PNG 링크에서 열어 제안서에 사용합니다.
- 기존 Mermaid는 세부 분기와 조건을 설명하는 용도로 함께 제공합니다.

## 모듈별 그림

크기는 실제 저장된 파일을 기준으로 기록합니다. 생성 요청의 목표 크기와 실제 출력 크기는 구분합니다.

### Module 00 — Foundations (기초)

[모듈 첫 화면](../../modules/00-foundations/README.md)에서 대표 그림을 확인합니다.

| 주제 | 레슨 | 원본 PNG | 실제 크기 |
|---|---|---|---|
| 에이전트가 파일과 테스트를 다루는 순서 | [00-1](../../modules/00-foundations/00-1-how-agents-work.md) | [agent-loop.png](../../modules/00-foundations/images/agent-loop.png) | 1672 × 941 |
| 옆에서 대화하기, 자동 실행하기, 원격에 맡기기 | [00-3](../../modules/00-foundations/00-3-execution-modes.md) | [execution-modes.png](../../modules/00-foundations/images/execution-modes.png) | 1672 × 941 |
| 실패 신호를 보고 대응 방법을 고릅니다 | [00-4](../../modules/00-foundations/00-4-failure-patterns.md) | [failure-patterns.png](../../modules/00-foundations/images/failure-patterns.png) | 1672 × 941 |

### Module 01 — Environment Setup (환경 구성)

[모듈 첫 화면](../../modules/01-environment-setup/README.md)에서 대표 그림을 확인합니다.

| 주제 | 레슨 | 원본 PNG | 실제 크기 |
|---|---|---|---|
| 공통 규칙을 한곳에 두고 두 도구에 연결합니다 | [01-1](../../modules/01-environment-setup/01-1-project-memory.md) | [project-memory.png](../../modules/01-environment-setup/images/project-memory.png) | 1672 × 941 |
| 작업 공간과 실행 권한을 따로 확인합니다 | [01-2](../../modules/01-environment-setup/01-2-permissions-and-sandbox.md) | [permission-boundary.png](../../modules/01-environment-setup/images/permission-boundary.png) | 1672 × 941 |
| MCP는 외부 도구를 연결하는 통로입니다 | [01-3](../../modules/01-environment-setup/01-3-mcp-servers.md) | [mcp-tools.png](../../modules/01-environment-setup/images/mcp-tools.png) | 1672 × 941 |

### Module 02 — Spec-Driven Development (명세 주도 개발)

[모듈 첫 화면](../../modules/02-spec-driven-development/README.md)에서 대표 그림을 확인합니다.

| 주제 | 레슨 | 원본 PNG | 실제 크기 |
|---|---|---|---|
| 요청을 개발에 사용할 파일로 바꿉니다 | [02-1](../../modules/02-spec-driven-development/02-1-prd-with-agents.md) | [spec-artifacts.png](../../modules/02-spec-driven-development/images/spec-artifacts.png) | 1672 × 941 |
| 큰 기능을 한 번에 검증할 작업으로 나눕니다 | [02-3](../../modules/02-spec-driven-development/02-3-task-breakdown.md) | [task-breakdown.png](../../modules/02-spec-driven-development/images/task-breakdown.png) | 1672 × 941 |
| 프롬프트를 작업 의뢰서처럼 구성합니다 | [02-5](../../modules/02-spec-driven-development/02-5-prompt-engineering.md) | [prompt-structure.png](../../modules/02-spec-driven-development/images/prompt-structure.png) | 1672 × 941 |

### Module 03 — Agentic Workflow (에이전트 작업 흐름)

[모듈 첫 화면](../../modules/03-agentic-workflow/README.md)에서 대표 그림을 확인합니다.

| 주제 | 레슨 | 원본 PNG | 실제 크기 |
|---|---|---|---|
| 탐색부터 검증까지 한 작업을 닫습니다 | [03-1](../../modules/03-agentic-workflow/03-1-explore-plan-implement-verify.md) | [development-loop.png](../../modules/03-agentic-workflow/images/development-loop.png) | 1672 × 941 |
| 실패하는 테스트를 먼저 확인합니다 | [03-2](../../modules/03-agentic-workflow/03-2-tdd-with-agents.md) | [tdd-cycle.png](../../modules/03-agentic-workflow/images/tdd-cycle.png) | 1672 × 941 |
| 새 세션에는 대화 대신 작업 상태를 전달합니다 | [03-4](../../modules/03-agentic-workflow/03-4-context-management.md) | [context-handoff.png](../../modules/03-agentic-workflow/images/context-handoff.png) | 1672 × 941 |

### Module 04 — Quality & Verification (품질과 검증)

[모듈 첫 화면](../../modules/04-quality-and-verification/README.md)에서 대표 그림을 확인합니다.

| 주제 | 레슨 | 원본 PNG | 실제 크기 |
|---|---|---|---|
| 파일 한 줄부터 사용자 흐름까지 확인합니다 | [04-1](../../modules/04-quality-and-verification/04-1-verification-layers.md) | [verification-layers.png](../../modules/04-quality-and-verification/images/verification-layers.png) | 1672 × 941 |
| 구현과 리뷰의 관점을 나누고 사람이 판단합니다 | [04-2](../../modules/04-quality-and-verification/04-2-cross-review.md) | [cross-review.png](../../modules/04-quality-and-verification/images/cross-review.png) | 1672 × 941 |
| 외부 자료와 작업 지시의 경계를 구분합니다 | [04-3](../../modules/04-quality-and-verification/04-3-security.md) | [trust-boundary.png](../../modules/04-quality-and-verification/images/trust-boundary.png) | 1672 × 941 |

### Module 05 — Scaling Up (규모 확장)

[모듈 첫 화면](../../modules/05-scaling-up/README.md)에서 대표 그림을 확인합니다.

| 주제 | 레슨 | 원본 PNG | 실제 크기 |
|---|---|---|---|
| 같은 저장소에서 작업 공간을 나눠 개발합니다 | [05-1](../../modules/05-scaling-up/05-1-parallel-agents.md) | [parallel-worktrees.png](../../modules/05-scaling-up/images/parallel-worktrees.png) | 1672 × 941 |
| 역할을 나누고 필요한 결과만 돌려받습니다 | [05-2](../../modules/05-scaling-up/05-2-orchestration.md) | [orchestration.png](../../modules/05-scaling-up/images/orchestration.png) | 1672 × 941 |
| 자동화 결과를 PR과 검증을 거쳐 반영합니다 | [05-4](../../modules/05-scaling-up/05-4-ci-cd-integration.md) | [ci-pipeline.png](../../modules/05-scaling-up/images/ci-pipeline.png) | 1672 × 941 |

### Module 06 — Team & Operations (팀과 운영)

[모듈 첫 화면](../../modules/06-team-and-operations/README.md)에서 대표 그림을 확인합니다.

| 주제 | 레슨 | 원본 PNG | 실제 크기 |
|---|---|---|---|
| 속도·품질·사용량을 함께 봅니다 | [06-2](../../modules/06-team-and-operations/06-2-metrics.md) | [metrics-balance.png](../../modules/06-team-and-operations/images/metrics-balance.png) | 1672 × 941 |
| 정책을 설정과 기록으로 연결합니다 | [06-3](../../modules/06-team-and-operations/06-3-governance.md) | [governance.png](../../modules/06-team-and-operations/images/governance.png) | 1672 × 941 |
| 반복되는 문제를 재사용할 규칙으로 바꿉니다 | [06-4](../../modules/06-team-and-operations/06-4-knowledge-loop.md) | [knowledge-loop.png](../../modules/06-team-and-operations/images/knowledge-loop.png) | 1672 × 941 |

## 사용과 해석

이미지 안의 화면·로그·체크 표시는 개념을 설명하기 위한 예시이며 실제 실행 증거가 아닙니다. 실행 절차와 완료 조건은 각 레슨의 텍스트와 Claude Code·Codex 레시피를 기준으로 적용합니다.

슬라이드에는 이미지의 가로세로 비율을 유지하여 배치합니다. 제목과 긴 설명은 슬라이드에서 별도로 작성합니다. 한글이 작아지는 배치에서는 그림을 여러 슬라이드로 나누거나 해당 프롬프트로 더 큰 출력을 생성합니다.
