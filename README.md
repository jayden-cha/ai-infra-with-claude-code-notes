# AI 시대에 개발자가 알아야 할 인프라 구성 배포 with 클로드코드 - 읽기 노트

이 저장소는 『AI 시대에 개발자가 알아야 할 인프라 구성 배포 with 클로드코드』를 읽으며 정리한 개인 학습 노트입니다.

책의 내용을 그대로 옮기기보다, 제가 이해한 방식으로 다시 정리하고 개발/인프라 업무 관점에서 느낀 점을 함께 기록합니다.

## 정리 목적

- AI 시대에 개발자가 인프라를 어떻게 이해해야 하는지 정리하기
- Kubernetes, GitOps, GitAIOps 개념을 학습하기
- Claude Code를 활용한 인프라 작업 흐름 이해하기
- 자연어, Git, AI 에이전트, 배포 자동화가 어떻게 연결되는지 살펴보기
- 실무에 적용할 수 있는 아이디어를 일반화해서 기록하기

## 정리 원칙

- 책의 원문을 길게 인용하지 않고, 내가 이해한 방식으로 정리한다.
- 중요한 개념은 예시와 함께 풀어서 설명한다.
- 개인적인 생각은 포함하되, 특정 회사나 내부 시스템 정보는 제외한다.
- 명령어는 가능한 한 재사용하기 쉬운 형태로 정리한다.
- 챕터별 정리와 개념 정리를 분리해 나중에 다시 찾아보기 쉽게 만든다.

## 목차

| 범위 | 제목 | 핵심 키워드 | 링크 |
|---|---|---|---|
| 지은이의 말 | GitAIOps를 향한 문제의식 | GitAIOps, 자연어 인프라, Git 기반 통제 | [00-authors-note.md](./chapters/00-authors-note.md) |
| Chapter 1 | AI 시대, 개발자의 인프라 | Kubernetes, GitOps, GitAIOps, 문서화 | [01-ai-era-developer-infra.md](./chapters/01-ai-era-developer-infra.md) |
| Chapter 2 | 환경 구성 | Claude Code, CLAUDE.md, JOURNEY.md, Cloud Build | [02-environment-setup.md](./chapters/02-environment-setup.md) |

## 별도 정리

| 문서 | 설명 |
|---|---|
| [concepts.md](./docs/concepts.md) | 책에서 반복해서 등장하는 핵심 개념 정리 |
| [commands.md](./docs/commands.md) | 실습 중 사용한 명령어 정리 |
| [ai-agent-rules.md](./docs/ai-agent-rules.md) | AI 에이전트와 함께 작업할 때 필요한 규칙 정리 |
| [questions.md](./docs/questions.md) | 더 알아볼 것과 학습 질문 정리 |

## 관심 있게 보고 있는 주제

- GitOps와 기존 CI/CD 배포 방식의 차이
- Argo CD와 Flux의 역할
- 자연어 기반 인프라 관리 방식
- Claude Code의 프로젝트 규칙 관리 방식
- `CLAUDE.md`를 활용한 AI 작업 가드레일
- 인프라 변경 사항을 Git 기반으로 기록하고 검토하는 방법
- AI 에이전트가 만든 변경 사항을 안전하게 검증하는 방법

## 주의 사항

이 저장소는 개인 학습 목적으로 작성한 정리입니다.  
책의 내용을 그대로 복제하기보다, 학습 과정에서 이해한 내용과 생각을 중심으로 정리합니다.

또한 실제 업무 환경, 내부 시스템명, 민감한 설정 정보는 포함하지 않습니다.
