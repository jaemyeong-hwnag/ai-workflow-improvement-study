# AI 코딩 워크플로우 가이드

> **목적:** 신규 프로젝트 vs 레거시 유지보수 상황별 실제 적용 방법  
> **기준:** 실제 시도한 3가지 방식 분석 + Claude Code 공식 문서 + 논문

---

## 핵심 정리: 언제 무엇을 쓰는가

```
신규 기능 개발 (새 프로젝트, 테스트 있음)
  → CLAUDE.md + Claude Code CLI 직접

레거시 유지보수 (테스트 없음, 기존 코드 건드려야 함)
  → Plan Mode 먼저 + 검증 가능한 기준 제시 + Claude Code CLI

멀티 레포 대규모 작업 (여러 레포에 동시 반영)
  → 워크플로우 서버 (Notion/Figma → Spec → 병렬 실행) 의미 있음
```

---

## Part 1. 신규 프로젝트

### 기본 패턴: Explore → Plan → Implement → Commit

```bash
# 1. 레포에 들어가서 Claude Code 시작
cd your-project
claude

# 2. Plan Mode로 탐색 (파일 건드리지 않음)
# Shift+Tab 두 번 → Plan Mode
"이 기능을 추가하려면 어떤 파일을 바꿔야 해? 계획 세워줘"

# 3. 계획 검토 후 구현 모드로 전환
# Shift+Tab → Normal Mode
"계획대로 구현해. 구현 후 테스트 실행하고 통과되면 커밋해"
```

### CLAUDE.md 작성 — 프로젝트 루트에 한 번만

```markdown
# CLAUDE.md (레포 루트)

## Architecture
- Hexagonal pattern (ports/adapters)
- Controller → Service → Mapper 레이어 순서 고정
- Works 메시지 발송은 core 패키지에서만

## Code Style
- Kotlin: KDoc 필수 (public API)
- REST 엔드포인트 변경 시 OpenAPI 3.x 주석 추가
- 테스트는 given/when/then 구조

## Commands
- 빌드: ./gradlew build
- 테스트: ./gradlew test
- 특정 모듈: ./gradlew :module:test

## Do NOT
- git add -A 사용 금지
- .env 파일 수정 금지
- 기존 인터페이스 시그니처 변경 금지
```

**규칙:** 200줄 미만 유지. Claude가 코드에서 알 수 있는 것은 쓰지 말 것. 틀린 행동이 반복될 때만 추가.

### 검증 기준을 반드시 포함

```
# 나쁜 요청
"알림 발송 기능 구현해줘"

# 좋은 요청
"POST /your-domain/notify 엔드포인트 구현해줘.
구현 후 curl -X POST http://localhost:8080/your-domain/notify?date=2025-01-01 
실행해서 HTTP 200 나오면 성공. 테스트도 작성하고 실행해."
```

Claude Code는 검증 기준이 있을 때 성능이 극적으로 향상됨 (공식 문서: "single highest-leverage thing").

### Hooks로 자동화 (설정 1회, 매번 자동 실행)

```json
// .claude/settings.json (프로젝트 루트)
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "cd $REPO_ROOT && ./gradlew ktlintFormat 2>/dev/null || true"
          }
        ]
      }
    ]
  }
}
```

CLAUDE.md에 "포맷팅 해줘" 쓰는 것과 달리 Hooks는 매번 반드시 실행됨.

---

## Part 2. 레거시 유지보수

### 레거시의 핵심 문제

테스트 없는 레거시에 AI가 코드를 추가하면:
- 변경 전 동작 확인 불가
- 회귀 감지 불가
- AI 스스로 검증 불가 → 품질 보장 없음

SWE-bench 2024 결과: 최고 성능 AI(Claude 3.7)도 실제 GitHub 이슈 38%는 해결 실패. 컨텍스트 불충분한 레거시에서는 더 낮음.

### 레거시 전용 CLAUDE.md 패턴

```markdown
# CLAUDE.md (레거시 프로젝트용)

## Critical Rules
- 기존 동작 변경 금지. 새 기능만 추가
- 변경 전 반드시 기존 동작을 Characterization Test로 캡처
- 컴파일 에러 0 확인 후 커밋
- 기존 패턴(설령 나빠도)을 그대로 따를 것. 리팩토링 하지 말 것

## Before Any Change
1. 변경 대상 클래스/메서드 현재 동작 파악
2. 현재 동작을 테스트로 고정
3. 새 기능 추가
4. 기존 테스트 전부 통과 확인

## Do NOT
- 기존 코드 리팩토링 금지
- 아키텍처 패턴 변경 금지
- 의존성 추가 금지 (승인 없이)
```

### 레거시 작업 흐름

```bash
# 1. Plan Mode로 영향 범위 파악 먼저
claude --permission-mode plan
"YourService 클래스에 새 메서드 추가하려고 해.
 기존에 어떤 의존성이 있는지, 어떤 파일들이 영향받는지 파악해줘.
 변경은 하지 말고."

# 2. 계획 검토 후 구현
# Shift+Tab으로 Normal Mode 전환
"기존 테스트 먼저 실행해서 현재 상태 확인하고,
 이상 감지 메서드만 추가해. 기존 테스트 깨지면 안 됨."
```

### Characterization Test 자동 생성 프롬프트

```
레거시 코드 변경 전에 쓰는 프롬프트:

"YourService.findByDate() 메서드의 현재 동작을 
 Characterization Test로 캡처해줘.
 실제 실행해서 현재 반환값을 확인하고, 
 그 값을 expected로 하는 테스트 작성.
 내 의도가 아니라 현재 코드가 실제로 하는 일을 테스트해야 함."
```

---

## Part 3. 시도한 3가지 방식의 실제 결론

### 시도 1: Notion/Figma → AI → Spec → Claude Code

**실제로 가치 있는 경우:**
- 여러 레포(frontend + backend + infra)에 동시 반영해야 할 때
- 팀 전체가 같은 스펙을 공유해야 할 때
- Notion 백로그가 수십 개고 각각 Claude Code 실행이 필요할 때

**가치 없는 경우 (CLI 직접이 더 빠름):**
- 레포 하나에 기능 하나 추가할 때
- 요구사항이 이미 명확할 때
- 혼자 작업할 때

**"Claude Code랑 뭐가 다른가" 정직한 답변:**
```
MCP 설치로 동일한 기능 대체 가능:
  claude mcp add notion
  claude mcp add figma
  → Claude Code가 직접 Notion/Figma 읽음

워크플로우 서버만의 실제 차별점:
  1. 멀티 레포 병렬 실행 (asyncio.gather)
  2. 실행 이력 영속화 (run_0~run_29 JSONL)
  3. 스펙 버전 관리 (output/ 디렉토리)
  
단, 이 3가지가 필요 없다면 → CLI 직접 사용이 빠름
```

### 시도 2: 하네스 엔지니어링

**신규 프로젝트:** 효과적

```python
# 실제로 가치 있는 패턴들
1. Human-in-the-Loop: --dangerously-skip-permissions 쓰기 전 반드시 사람 검토
2. LLM-as-Judge: 생성 모델(Sonnet) ≠ 평가 모델(Haiku)
3. Session Resume: --resume으로 긴 작업 재개
4. Observability: 토큰 카운팅, 실행 시간 추적
```

**레거시 프로젝트:** 제한적

```
- 테스트가 없으면 하네스 자체가 작동 안 함
- 코드 스캐너가 패턴을 못 찾으면 AI가 추측으로 생성
- 해결책: 레거시에는 Characterization Test 먼저 → 그 다음 하네스 적용
```

### 시도 3: AI Skills Interface

**결론:** 팀 컨벤션 통일에는 유용, 개인 작업에는 과도한 복잡성

```
팀에서 쓸 때 의미 있는 것:
  - delivery-workflow 스킬: 팀 전체가 커밋 규칙 공유
  - hexagonal-development 스킬: 아키텍처 규칙 강제

개인이 혼자 쓸 때:
  - CLAUDE.md에 직접 쓰는 게 동일한 효과
  - 스킬 파일 관리 오버헤드가 더 큼
```

---

## Part 4. 실제 상황별 체크리스트

### 신규 기능 추가 (테스트 있는 프로젝트)

```
□ 레포에 CLAUDE.md 있는가?
  없으면: claude → "/init" 실행 후 수정
  
□ 요청에 검증 기준 포함했는가?
  없으면: curl 명령어 또는 테스트 이름 추가
  
□ Plan Mode로 영향 범위 먼저 파악했는가?
  복잡한 작업: 반드시 먼저
  단순 작업: 생략 가능
  
□ 구현 후 테스트 실행을 요청에 포함했는가?
```

### 레거시 버그 수정

```
□ 버그 재현 방법이 명확한가?
  명확하면: 재현 명령어를 프롬프트에 포함
  
□ 영향 범위 파악이 되었는가?
  Plan Mode로 먼저 파악
  
□ 기존 동작 보호 장치가 있는가?
  없으면: 변경 전 Characterization Test 먼저 요청
  
□ CLAUDE.md에 "기존 동작 변경 금지" 규칙 있는가?
```

### 대규모 기능 (멀티 레포)

```
□ 각 레포에 CLAUDE.md가 있는가?
□ 인터페이스 계약이 먼저 정의되었는가?
  (Interface-First: 구현 전 시그니처 합의)
□ 레포 간 의존성 순서가 명확한가?
  (ex: core 변경 → api 변경 → frontend 변경)
□ 각 레포 변경을 독립적으로 검증할 수 있는가?
```

---

## Part 5. 레퍼런스 목록

### Claude Code 공식 문서

| 문서 | 핵심 내용 | URL |
|------|-----------|-----|
| Best Practices | 컨텍스트 관리, CLAUDE.md 설계, 검증 기준 | code.claude.com/docs/en/best-practices.md |
| Memory (CLAUDE.md) | 계층 구조, 200줄 제한, Auto Memory | code.claude.com/docs/en/memory.md |
| Common Workflows | Plan Mode, 버그 수정, 테스트 작성 패턴 | code.claude.com/docs/en/common-workflows.md |
| How Claude Code Works | Agentic Loop, 컨텍스트 윈도우 구조 | code.claude.com/docs/en/how-claude-code-works.md |
| Sub-agents | 격리 실행, Writer/Reviewer 패턴 | code.claude.com/docs/en/sub-agents.md |
| Hooks | PostToolUse, 결정론적 자동화 | code.claude.com/docs/en/hooks.md |
| Skills | CLAUDE.md vs Skills 구분 | code.claude.com/docs/en/skills.md |
| MCP | Notion, Figma 직접 연결 | code.claude.com/docs/en/mcp.md |

### 논문 및 연구

| 논문 | 핵심 발견 | 적용 포인트 |
|------|-----------|------------|
| **Liu et al. "Lost in the Middle" (TACL 2024)** | 긴 컨텍스트 중간 내용 30% 정확도 하락 | 핵심 제약을 메시지 하단에 반복 |
| **arXiv:2411.10541** | Claude=XML 최적, GPT/Gemini=YAML 17.7pt 우위 | AI 내부 통신 포맷 선택 기준 |
| **LLMLingua-2 (Microsoft 2024)** | 지시어/스키마 압축 금지, 주입 데이터만 압축 | 코드 스캔 결과만 압축 |
| **SWE-RL (arXiv:2502.18449, 2025)** | git history(PR, 이슈)로 RL 학습 → 41% solve rate | git history를 컨텍스트로 활용 |
| **AutoDev (arXiv:2403.08299, 2024)** | Docker 격리 + AI 에이전트 → 91.5% Pass@1 | 레거시 변경 시 샌드박스 실행 |
| **RE-Bench (arXiv:2411.15114, 2024)** | AI가 2시간 작업 성능 4x, 32시간은 인간이 2x | 작업 단위를 짧게 분할해야 함 |
| **SWE-bench Verified (2024-2025)** | Claude 3.7 Sonnet ~62% solve rate | 38%는 항상 실패. Human-in-Loop 필수 |
| **Schulhoff et al. "The Prompt Report" (2024)** | Few-shot이 가장 효과적, 역할 부여 중간 효과 | 예시 코드 포함이 가장 큰 레버리지 |
| **Zheng et al. "Judging LLM-as-a-Judge" (2023)** | 같은 모델로 자기평가 시 편향 발생 | 생성 모델 ≠ 평가 모델 |

### 핵심 패턴 레퍼런스

| 패턴 | 출처 | 설명 |
|------|------|------|
| **Characterization Test** | Michael Feathers "Working Effectively with Legacy Code" (2004) | 레거시 코드 변경 전 현재 동작 고정 |
| **Hexagonal Architecture** | Alistair Cockburn (2005) | 포트/어댑터 패턴, 외부 의존성 격리 |
| **Interface-First Development** | Domain-Driven Design 원칙 | 구현 전 계약 정의 |
| **Plan Mode** | Claude Code 공식 | Shift+Tab×2, read-only 탐색 |
| **Writer/Reviewer Pattern** | Claude Code Best Practices | 구현 세션 ≠ 리뷰 세션 |
| **Git Worktree 병렬 실행** | Claude Code Common Workflows | `claude --worktree feature-x` |
| **LLM-as-Judge** | Zheng et al. 2023 | 품질 평가 모델 분리 |

---

## 한 줄 요약

```
신규 프로젝트:  CLAUDE.md 먼저 + 검증 기준 포함 + Claude Code CLI
레거시:        Plan Mode + Characterization Test + 최소 변경
멀티 레포:     워크플로우 서버 또는 MCP + Claude Code
모든 경우:     Human-in-the-Loop는 선택이 아닌 필수 (AI는 38~60% 실패)
```
