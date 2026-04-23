[English](README.md) | 한국어

# haroom_plugins

> **haroom** 가 만든 Claude Code 플러그인 모음.

Claude Code 로 실제 작업하다 보면 터지는 특정 문제들을 겨냥한 작은 플러그인 모음입니다: 병렬 세션이 서로 코드를 덮어쓰는 문제, AI 가 자기 코드를 자기가 리뷰하는 문제, AI 가 파싱 못하는 디자인 문서 문제. 각 플러그인은 자기 repo 에서 별도 관리되며, 이 aggregator marketplace 가 4개 전부를 묶어 `marketplace add` 한 번으로 설치할 수 있게 합니다.

---

## 어떤 플러그인을 써야 하나

| 이런 상황이라면... | 추천 |
|---|---|
| "Claude Code 세션 여러 개 돌리는데 서로 충돌한다" | **[ddaro](https://github.com/minwoo-data/ddaro)** |
| "위험한 PR merge 전에 5각도 코드 리뷰가 필요하다" | **[prism](https://github.com/minwoo-data/prism)** |
| "이 디자인 문서는 AI 에이전트도 읽을 텐데 완벽하게 만들고 싶다" | **[triad](https://github.com/minwoo-data/triad)** |
| "Claude 와 Codex 로 파일이 수렴할 때까지 cross-review 하고 싶다" | **[mangchi](https://github.com/minwoo-data/mangchi)** |

아래에서 각 플러그인 상세.

---

## Quick Start

### 1. 마켓플레이스 등록 (처음 한 번만)

```
/plugin marketplace add https://github.com/minwoo-data/haroom_plugins.git
```

### 2. 원하는 플러그인 설치

```
/plugin install ddaro
/plugin install prism
/plugin install triad
/plugin install mangchi
```

### 3. Claude Code 재시작

설치 / 업데이트 후 반드시 필요.

### 4. 업데이트

```
/plugin update
```

한 줄로 설치된 모든 haroom 플러그인을 한꺼번에 갱신.

---

## 플러그인

### [ddaro](https://github.com/minwoo-data/ddaro)

**문제:** 같은 repo 를 편집하는 두 Claude Code 세션이 서로의 작업을 덮어씀. 세션이 컨텍스트 없이 죽음.

**해결:** 세션마다 git worktree 하나씩 (일회용 폴더 + 브랜치) 생성, 매 commit 자동 push, 매번 plain-text 컨텍스트 스냅샷을 디스크에 기록해서 크래시가 나도 그림이 사라지지 않음. `main` 의 파일이 조용히 사라질 상황이면 merge 거부.

**사용 시점:** Claude Code 세션 2개 이상 병렬 운영, 위험한 리팩터링, 장기 실험 브랜치 유지, 크래시 잦은 장비에서 작업.

### [prism](https://github.com/minwoo-data/prism)

**문제:** 리뷰어 한 명이 다른 각도에서 봤으면 잡혔을 걸 놓침. LLM 셀프 리뷰는 예측 가능한 맹점이 있음.

**해결:** 5명의 특화된 리뷰 에이전트 (충돌 탐지 / 개선 / Devil's advocate / 코드 리뷰 / 견고성) 가 타겟을 병렬로 봄. 한 명만 찾은 finding 은 별도 Verifier pass 로 cross-check 해서 false positive 가 시간을 낭비하지 않게 함.

**사용 시점:** 큰 아키텍처 결정 직전, 확신 안 서는 PR merge 직전, skill / 디자인 문서 / 워크플로우 감사.

### [triad](https://github.com/minwoo-data/triad)

**문제:** 디자인 문서를 확정했는데 나중에 AI 가 이해 못함, 아키텍처가 일찍 낡음, 새 사용자가 읽고 무슨 말인지 모름.

**해결:** 3 개의 독립 에이전트가 한 문서를 고정 렌즈 (LLM 명확성 / 아키텍처 수명 / 엔드유저 이해도) 로 라운드를 반복하며 검토. 세 관점이 합의하거나 cap 에 도달할 때까지. 합의는 `updated.md` 에 기록, 원본은 건드리지 않음.

**사용 시점:** 프로젝트 지시 문서 (`CLAUDE.md`, 에이전트 프롬프트), ADR, 공개 문서, PR 근거 확정 직전.

### [mangchi](https://github.com/minwoo-data/mangchi)

**문제:** 한 모델이 자기 코드를 리뷰하면 자기 맹점을 강화함. 같은 버그가 여러 리뷰 pass 를 통과해 살아남음.

**해결:** Claude 가 편집, Codex 가 5개 축 (정확성 / 보안 / 가독성 / 성능 / 설계) 을 순환하며 비판. 두 모델이 서로의 사각지대를 잡아주며 코드가 수렴할 때까지 반복.

**사용 시점:** 릴리스 전 프로덕션 코드 하드닝, 프롬프트 주입 / 보안 방어 코드 리뷰, Claude Max 토큰 여유로 외부 cross-model 리뷰 돌리고 싶을 때.

---

## Repo 구조

각 플러그인은 **git submodule** 로 `plugins/<name>/` 에 포함되며, 각자의 GitHub repo 를 가리킵니다. 플러그인 이 새 릴리스를 올리면 이 aggregator 의 submodule 포인터는 매일 06:00 UTC 에 [.github/workflows/daily-submodule-update.yml](./.github/workflows/daily-submodule-update.yml) 이 자동으로 bump. 사용자는 다음 `/plugin update` 때 새 버전을 받습니다.

---

## License

MIT - [LICENSE](./LICENSE) 참조. 각 플러그인 폴더도 원본 LICENSE 를 유지해 provenance 보존.

## Author

haroom - [github.com/minwoo-data](https://github.com/minwoo-data)
