# ai-development-guide

**Claude Code와 Codex로 대규모 개발하기** — 따라하기와 숙제로 익히는 실전 가이드

장난감 예제가 아니라 수만 라인 규모의 프로젝트에 AI 코딩 에이전트를 적용하는 방법을 다룹니다.
모든 레슨은 **개념 → 따라하기 → 숙제 → 회고** 순서로 진행합니다.

## 구성

| 구분 | 내용 |
|---|---|
| [modules/](./modules) | 짧은 레슨 단위로 익히는 기법 (7개 모듈) |
| [projects/](./projects) | 모듈을 관통하는 장기 실습 프로젝트 (문서화 / 개발 / 레거시) |
| [templates/](./templates) | 바로 복사해 쓰는 AGENTS.md, CLAUDE.md, PRD, ADR, Task 명세, 핸드오프 노트 |
| [docs/design/](./docs/design) | 커리큘럼 설계서, 레슨·숙제 템플릿 |
| [docs/reference/tool-reference.md](./docs/reference/tool-reference.md) | Claude Code·Codex 명령어/설정 검증 레퍼런스 (레슨의 사실 기준) |

### 모듈
| # | 모듈 | 핵심 |
|---|---|---|
| 00 | [Foundations](./modules/00-foundations) | 에이전트 동작 원리, Claude vs Codex |
| 01 | [Environment Setup](./modules/01-environment-setup) | 메모리 파일, 권한, MCP, Skills/Hooks |
| 02 | [Spec-Driven Development](./modules/02-spec-driven-development) | PRD, ADR, 작업 분해, 인터페이스 우선 |
| 03 | [Agentic Workflow](./modules/03-agentic-workflow) | Plan→Implement→Verify, TDD, 컨텍스트 관리 |
| 04 | [Quality & Verification](./modules/04-quality-and-verification) | 검증 계층, 교차 리뷰, 보안 |
| 05 | [Scaling Up](./modules/05-scaling-up) | 병렬 에이전트, 오케스트레이션, CI 자동화 |
| 06 | [Team & Operations](./modules/06-team-and-operations) | 비용, 지표, 거버넌스 |

### 프로젝트
| # | 프로젝트 | 내용 |
|---|---|---|
| A | [문서화 프로젝트](./projects/A-documentation-project) | Docs-as-Code 문서 사이트 구축·운영 |
| B | [개발 프로젝트 TaskFlow](./projects/B-development-project) | 웹 + API + 워커 + CLI, 3~5만 라인 |
| C | [레거시 마이그레이션](./projects/C-legacy-migration-project) | 모르는 코드베이스의 대규모 변경 |

## 학습 경로
| 경로 | 대상 | 순서 | 기간 (주 5시간) |
|---|---|---|---|
| Fast Track | 실무자 | 01 → 02 → 03 → Project B (B0~B4) | 4주 |
| Full Track | 입문자 | 00 → 01 → A → 02 → 03 → 04 → B → 05 → 06 | 12주 |
| Lead Track | 리드/아키텍트 | 01 → 04 → 05 → 06 → Project C | 6주 |

## 숙제 제출
1. 저장소를 fork 하고 숙제마다 `hw/<모듈>-<레슨>` 브랜치를 만듭니다.
2. 결과물과 `submissions/<모듈>-<레슨>.md`([템플릿](./docs/design/homework-template.md))를 커밋합니다.
3. PR 본문에 에이전트 리뷰 결과와 자기 평가를 첨부합니다.

자세한 설계는 [커리큘럼 설계서](./docs/design/curriculum-design.md)를 참고하세요.
