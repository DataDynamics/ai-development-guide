---
title: OpenAI·Anthropic 공식 개발 가이드 적용 기준
last-verified: 2026-10-08
verification-method: official-documentation
---

# OpenAI·Anthropic 공식 개발 가이드 적용 기준

이 문서는 AI 코딩 도구를 사용해 소프트웨어를 만드는 방법을 다룹니다. 모델 API를 제품에 넣는 방법은 별도 주제입니다. 공식 문서는 아래 날짜에 원문으로 확인했으며, 이 문서의 확인은 CLI 실기 검증을 의미하지 않습니다. 명령어·설정의 세부 검증은 [도구 레퍼런스](./tool-reference.md)를 따릅니다.

## 공식 자료와 적용 위치

| 공식 자료 | 확인한 권고·기능 | 이 저장소의 적용 |
|---|---|---|
| OpenAI [Best practices](https://learn.chatgpt.com/guides/best-practices) | 목표·맥락·제약·완료 기준을 전달하고, 복잡한 작업은 계획하며, 결과를 검증합니다 | [실행 가이드](../workflow/README.md)와 [프롬프트 레슨](../../modules/02-spec-driven-development/02-5-prompt-engineering.md) |
| OpenAI [Custom instructions with AGENTS.md](https://learn.chatgpt.com/docs/agent-configuration/agents-md) | 프로젝트 지침을 파일로 관리하며, 시작 위치에 따른 지침 탐색 순서가 있습니다 | [AGENTS.md 템플릿](../../templates/AGENTS.md.template) |
| OpenAI [Modernizing your Codebase with Codex](https://developers.openai.com/cookbook/examples/codex/code_modernization) | 시범 범위의 실행 계획·현황·설계·검증 문서를 연결합니다 | 기존 프로젝트의 기준선 기록과 점진적 적용 |
| Anthropic [Best practices for Claude Code](https://code.claude.com/docs/en/best-practices) | 먼저 탐색·계획하고, 테스트 같은 확인 수단을 제공합니다. 명확하고 작은 변경은 절차를 간소화합니다 | Task 단위 실행과 검증·재계획 루프 |
| Anthropic [Common workflows](https://code.claude.com/docs/en/common-workflows) | 탐색·버그 수정·리팩터링·테스트·PR의 작업별 레시피를 제공합니다 | 단계별 프롬프트와 기존 Module 03 |
| Anthropic [How Claude remembers your project](https://code.claude.com/docs/en/memory) | `CLAUDE.md`에서 `@path`로 파일을 가져올 수 있고, 지침은 짧고 구체적으로 유지합니다 | [CLAUDE.md 템플릿](../../templates/CLAUDE.md.template)과 공통 규칙 분리 |

확인일은 모든 행에 대해 **2026-10-08**입니다. 최신 기능을 추가할 때는 해당 문서를 다시 확인하고, 문서 확인과 실제 실행 여부를 구별해서 기록합니다.

## 이 저장소가 정한 운영 규칙

다음은 공식 문서의 필수 규격이 아니라, 두 도구로 같은 프로젝트를 진행하기 위해 이 가이드가 정한 규칙입니다.

- `FR-001 → TASK-001 → AC-001 → 테스트 → 검증 기록`으로 변경의 이유와 증거를 연결합니다.
- 단계마다 입력·산출물·완료 조건을 정의합니다. 검증에 실패하면 구현 또는 계획으로 돌아갑니다.
- 산출물 경로와 템플릿은 [템플릿 목록](../../templates/README.md)으로 통일합니다. 기존 프로젝트의 구조는 필요한 부분만 대응시킵니다.
- 공통 지침의 원본은 `AGENTS.md`로 하고, `CLAUDE.md`는 이를 import합니다. 프로젝트별 적용 확인은 실제 세션에서 수행합니다.
- `make verify`는 이 가이드의 권장 진입점 이름입니다. 도구에 내장된 명령이 아니며, 프로젝트에서 직접 구성하고 실행한 후에만 지침에 등록합니다.
- 작성 예제와 예상 결과를 실제 통과 기록으로 사용하지 않습니다. 미실행 검증은 `미실행`, 판단에 필요한 미확인 기능은 `TODO(verify)`로 표시합니다.

## 유지보수

공식 문서가 바뀌면 먼저 영향을 받는 템플릿과 레슨을 찾고, 원문·확인일·변경 이유를 함께 갱신합니다. 모델명·가격·권한 옵션을 보편적인 개발 원칙과 섞지 않습니다. 새 모델이 나왔다는 이유만으로 모든 프로젝트의 지침을 늘리지 않습니다.
