# 워크플로우 개선점 및 추가 레퍼런스

> **작성일:** 2026-04-03  
> **기반:** 코드 전체 분석 + Claude Code 공식 문서 + 최신 연구 결과

---

## 목차

1. [코드에서 발견된 추가 개선 가능 영역](#코드-개선)
2. [Claude Code 공식 Best Practices와 비교](#공식-비교)
3. [핵심 대안: MCP 접근법](#mcp-대안)
4. [최신 연구 결과 추가](#최신-연구)
5. [구체적 개선 제안 목록](#개선-제안)

---

## 코드에서 발견된 추가 개선 가능 영역 {#코드-개선}

### execute.py 분석 — 추가 발견사항

`execute.py`는 Claude Code 실행과 git 통합의 핵심 파일로, 이전에 분석하지 않은 부분.

#### 1. `git add -A` 자동 커밋 — 보안 위험

```python
# execute.py:218
async def _git_auto_commit(repo_path: str, message: str) -> str | None:
    await _run(["git", "add", "-A"])  # 모든 파일 스테이징
    rc, _ = await _run(["git", "commit", "-m", message])
```

**문제:** `git add -A`는 `.env`, 시크릿, 바이너리 등 의도치 않은 파일을 모두 스테이징.  
Claude Code Best Practices: "stage changes with specific file names rather than using `git add -A`"  
**개선 방향:** git이 추적하는 변경 파일만 스테이징하거나, `.gitignore` 확인 후 스테이징.

#### 2. SSE 폴링 구조 — 연결 끊김 시 상태 불일치

```python
# execute.py:583-597
while len(done_set) < len(target_prompts):
    if await request.is_disconnected():
        break  # 연결 끊겨도 Claude Code는 계속 실행됨
    for name, q in queues.items():
        try:
            kind, val = q.get_nowait()
            ...
        except asyncio.QueueEmpty:
            pass
    await asyncio.sleep(0.05)  # 50ms 폴링 — 고빈도
```

**문제:** 브라우저 연결이 끊기면 SSE 루프 탈출하지만 백그라운드 Claude Code 태스크는 계속 실행됨. 사용자가 재연결하면 기존 실행 상태 복구 로직이 별도로 존재하지만, 실행 중 상태에서 재연결 시 폴링 방식으로 따라잡음 → 과도한 I/O.

**실제 확인:** `run_0~run_29` (30회) 파일이 존재 → 사용자가 반복 재실행. 매 실행이 JSONL로 저장되므로 디스크 누적.

#### 3. 수정 요청 컨텍스트 — 4000자 제한

```python
# execute.py:475
diff = await _git_branch_diff(repo_path_for_diff, branch_type)
context = f"<code_changes>{diff[:4000]}</code_changes>\n\n" if diff.strip() else ""
```

**문제:** git diff를 4000자로 잘라 주입. 실제 중요한 변경사항이 잘릴 수 있음. 또한 diff는 줄 단위 변경이므로 4000자 중간에 잘리면 XML이 오염될 수 있음.

#### 4. 스킬 주입 방식 — 컨텍스트 오염

```python
# spec_generator.py:536-538
for sid in (spec.enabled_skills or []):
    if sid in SELECTABLE_SKILLS:
        prompt += SELECTABLE_SKILLS[sid]["instruction"]
```

현재 `SELECTABLE_SKILLS`에는 `code-documentation` 하나만 정의되어 있음. 스킬을 프롬프트 끝에 추가(append)하는 방식은 Lost-in-the-Middle 문제와 동일하게 무시될 가능성이 있음.

#### 5. settings.json에 토큰 평문 저장

```json
{
  "backlog": {
    "notion": { "token": "ntn_..." },
    "figma": { "token": "figd_..." }
  }
}
```

**보안 이슈:** Notion 통합 토큰과 Figma 개인 토큰이 평문으로 파일에 저장됨. git에 실수로 커밋될 수 있음.

#### 6. 평가 레이어의 실질적 한계 확인

```python
# evaluation.py — Code Evaluators
def eval_prompts_format(spec: TechSpec) -> EvalResult:
    valid = sum(1 for p in spec.ai_prompts if "<" in p.prompt and ">" in p.prompt)
```

XML에 `<` `>` 문자가 있으면 통과. 실제 XML 구조 유효성을 검사하지 않음. `<text>` 하나만 있어도 pass.  
**결론:** 평가 파이프라인이 있지만 실질적 품질 보증 기능이 약함. "Validation Illusion" 확인.

---

## Claude Code 공식 Best Practices와 비교 {#공식-비교}

Claude Code 공식 문서(code.claude.com/docs/en/best-practices.md)와 현재 워크플로우를 비교.

### A. 컨텍스트 윈도우 관리 — 현재 구조의 근본 문제

공식 문서 핵심 주장:
> "Most best practices are based on one constraint: **Claude's context window fills up fast, and performance degrades as it fills**."
> "A single debugging session or codebase exploration might generate and consume tens of thousands of tokens."

현재 워크플로우의 컨텍스트 구성 (`generate_repo_prompts` 기준):
```
code_xml (레포 전체 시그니처) 
+ user_input  
+ overview + architecture + interfaces + steps  
+ repo_xml (레포별)  
+ user_prefix  
+ skill_instructions  
+ prompt-optimizer 추가 처리
```

**문제:** 최종 Claude Code에 전달되는 프롬프트가 누적 레이어로 구성됨. 정보 밀도는 높지만 컨텍스트 소비가 예측 불가능.

**공식 권장:**
```
# 효과적인 접근
"look at how existing widgets are implemented on the home page 
to understand the patterns. HotDogWidget.php is a good example."
```
→ 파일 직접 참조 `@` 방식이 미리 압축된 XML 주입보다 Claude Code의 내부 컨텍스트 관리와 더 잘 통합됨.

### B. 검증 기준 제공 — 가장 높은 레버리지 방법

공식 문서:
> "Claude performs dramatically better when it can verify its own work. This is the single highest-leverage thing you can do."

| 현재 워크플로우 | 공식 권장 |
|----------------|----------|
| implementation_plan (요구사항 목록) | 검증 가능한 테스트 케이스 또는 성공 기준 |
| 수용 기준(AC) 텍스트로 제공 | 실행 가능한 명령어로 검증 가능하게 제공 |

현재 spec의 acceptance criteria는 텍스트. Claude Code가 엔드포인트를 구현하고 나서 그것이 올바른지 스스로 확인할 방법이 없음.

**개선 방향:**
```xml
<verification>
  <cmd>curl -X POST http://localhost:8080/your-endpoint?date=2025-01-01</cmd>
  <expected_status>200</expected_status>
  <expected_channel_message>exists</expected_channel_message>
</verification>
```

### C. Explore → Plan → Implement → Commit 4단계 분리

공식 문서의 권장 워크플로우:
```
1. Explore (Plan Mode) — 파일 읽기만, 변경 없음
2. Plan (Plan Mode) — 구현 계획 작성, Ctrl+G로 편집
3. Implement (Normal Mode) — 코딩 + 검증
4. Commit — 커밋 및 PR
```

현재 워크플로우 vs 공식 권장 비교:
```
현재: [AI 스펙 생성] → [사람 입력] → [AI 프롬프트 생성] → [Claude Code 직접 실행]
공식: [사람이 Claude에게 Explore 요청] → [사람이 Plan 검토] → [Implement] → [Commit]
```

**핵심 차이:** 공식 방식은 Claude Code가 **직접 레포를 탐색**하므로 외부 스펙 생성 레이어가 불필요. 워크플로우의 코드 스캐너가 하는 일을 Claude Code 자체가 Plan Mode에서 함.

### D. CLAUDE.md 전략 — 현재 워크플로우가 놓친 부분

공식 문서에서 CLAUDE.md 핵심 가이드라인:
```
- 200줄 미만 유지
- Claude가 코드에서 추론할 수 없는 내용만 포함
- "IMPORTANT" 등 강조어로 중요 규칙 표시
- 자주 틀리는 부분이 있으면 CLAUDE.md에 추가
```

**현재 워크플로우의 공백:** 타겟 레포에 CLAUDE.md가 없음.  
AI가 생성한 프롬프트를 매번 주입하는 대신, **레포에 CLAUDE.md를 한 번 작성**하면 모든 작업에서 공유됨.

현재 접근 vs CLAUDE.md 접근:
```
현재: 매 실행마다 code_xml + pattern + architecture + ... → Claude Code 주입
CLAUDE.md: 
  # Architecture
  - Hexagonal pattern (ports/adapters)
  - MyBatis Mapper in com.yourcompany.api.*.mapper
  - Controller → Service → Mapper 레이어
  - 외부 메시지 발송은 core 패키지에서만
  → 영구적, 컨텍스트 효율적
```

### E. Skills vs Hooks vs CLAUDE.md 구분

공식 문서의 명확한 구분:
| 기법 | 사용 시점 | 특성 |
|------|----------|------|
| **CLAUDE.md** | 항상 적용되는 규칙 | 매 세션 자동 로드 |
| **Skills** | 특정 작업에만 필요한 지식 | 호출 시 또는 자동 선택 |
| **Hooks** | 반드시 실행되어야 하는 동작 | 결정론적, 우회 불가 |
| **Subagents** | 격리된 탐색/검증 | 별도 컨텍스트 |

현재 `SELECTABLE_SKILLS` (`code-documentation`)는 프롬프트 주입 방식 → 공식의 Hooks로 구현하면 결정론적으로 보장됨:
```json
// .claude/settings.json — 타겟 레포에 위치
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Edit|Write",
      "hooks": [{"type": "command", "command": "ktlint --format"}]
    }]
  }
}
```

### F. Writer/Reviewer 패턴 — 현재 없음

공식 문서의 다중 세션 활용:
```
Session A (Writer): "implement rate limiter"
Session B (Reviewer): "review @src/middleware/rateLimiter.ts for edge cases"
Session A: "address review feedback: [Session B output]"
```

현재 워크플로우에는 리뷰 프롬프트(`type="review"`)가 생성되지만 실제로 별도 세션으로 실행되지 않음. 생성 세션과 리뷰 세션을 분리하면 **자기 평가 편향** 없이 품질 개선 가능.

---

## 핵심 대안: MCP 접근법 {#mcp-대안}

### "claude code랑 뭐가 다른가"에 대한 직접적 답변

공식 문서:
> "Run `claude mcp add` to connect external tools like **Notion, Figma**, or your database."

즉, 현재 워크플로우(Notion/Figma → AI → Spec → Claude Code)의 핵심 기능을 MCP로 직접 구현 가능:

```bash
# MCP 서버 설치
claude mcp add notion -- npx @anthropic/notion-mcp-server
claude mcp add figma -- npx @anthropic/figma-mcp-server

# 이후 Claude Code에서 직접:
claude "이 Notion 페이지의 요구사항을 구현해줘: [URL]
       Figma 디자인도 참고해: [URL]
       기존 코드 패턴 파악하고 구현해"
```

**장점:**
- 별도 워크플로우 서버 불필요
- Claude Code가 직접 실시간으로 Notion 내용 읽기
- Figma 디자인 직접 분석
- 레포 코드도 Claude Code가 직접 탐색

**한계 (현재 워크플로우 대비 부족한 부분):**
- 멀티 레포 병렬 실행은 직접 구현해야 함
- 실행 이력/로그 영속화 없음
- 스펙 버전 관리 없음

### 워크플로우 vs MCP 직접 사용 의사결정 트리

```
작업 규모가 크고 멀티 레포인가?
  └─ YES → 워크플로우 의미 있음 (병렬 실행, 로그 영속화)
  └─ NO  → MCP + Claude Code 직접 사용이 빠름

반복 작업인가? (같은 패턴의 기능 추가)
  └─ YES → CLAUDE.md + Skills로 패턴 영구화 → Claude Code 직접
  └─ NO  → 워크플로우의 스펙 생성 레이어가 탐색 비용 절감

레거시 프로젝트인가?
  └─ YES → 어느 방식이든 한계 있음, Claude Code 직접 + CLAUDE.md가 유연
  └─ NO  → 워크플로우의 하네스 엔지니어링이 효과적
```

---

## 최신 연구 결과 추가 {#최신-연구}

### 1. SWE-RL (arXiv:2502.18449, 2025)

**제목:** "SWE-RL: LLM Reasoning via Reinforcement Learning on Software Evolution"

**핵심 발견:**
- Llama3-SWE-RL-70B: SWE-bench Verified에서 **41% solve rate** (100B 이하 모델 최고)
- 오픈소스 소프트웨어 진화 데이터(코드 스냅샷, 변경 이력, 이슈, PR)로 RL 학습
- **소프트웨어 엔지니어링 데이터로 훈련했는데 수학/추론 일반화 성능도 향상**됨 (예상외 결과)

**현재 워크플로우 시사점:**
- git history (PR, 이슈, 변경 이력)를 컨텍스트로 활용하면 AI 성능 향상 가능
- 현재 코드 스캐너는 현재 상태만 분석, git log/blame 활용 없음
- `execute.py`의 `_git_branch_diff`가 이 방향에 있지만 최근 커밋 히스토리까지 확장 가능

### 2. AutoDev (arXiv:2403.08299, 2024)

**제목:** "AutoDev: Automated AI-Driven Development"

**핵심 발견:**
- Docker 컨테이너 격리: 파일 편집, 빌드, 테스트, git 작업을 안전하게 자동화
- HumanEval: **91.5% Pass@1** (코드 생성), **87.8%** (테스트 생성)
- 핵심 교훈: "보안과 자동화 수준은 Docker 격리로 동시 달성 가능"

**현재 워크플로우 시사점:**
- `--dangerously-skip-permissions`를 사용하는 현재 방식 대신 Docker 격리 환경에서 Claude Code 실행 가능
- Claude Code 공식의 `/sandbox` 기능과 동일한 방향
- 레거시 프로젝트 유지보수에서도 격리 환경이면 파괴적 변경 위험 감소

### 3. SWE-bench 2024 최신 결과

SWE-bench Verified (2024-2025 최신 리더보드):
| 시스템 | Solve Rate |
|--------|-----------|
| OpenAI o3 (high compute) | ~71% |
| Claude 3.7 Sonnet (기준) | ~62% |
| SWE-RL 70B | ~41% |
| Claude 3.5 Sonnet (2024 기준) | ~49% |

**시사점:** 최고 성능 모델조차 실제 GitHub 이슈의 29~60%는 해결 못 함. 자동화 워크플로우 설계 시 human-in-the-loop와 검증 단계는 선택이 아닌 필수.

### 4. "The Prompt Report" (Schulhoff et al. 2024)

58개 프롬프트 기법을 체계적으로 평가한 메타 연구.

**현재 워크플로우에 적용된 기법의 효과:**
| 기법 | 효과 | 현재 적용 여부 |
|------|------|--------------|
| 역할 부여 (role:) | 중간~높음 | ✅ 모든 스킬에 적용 |
| XML 구조화 | 높음 (Claude 전용) | ✅ |
| 제약 반복 (constraint repeat) | 높음 | ✅ `_repeat_constraint` |
| Few-shot 예시 | 매우 높음 | ❌ 없음 |
| Chain-of-Thought | 높음 | ❌ step-by-step 없음 |
| 검증 가능한 성공 기준 | 매우 높음 | ❌ 없음 |

**가장 큰 공백: Few-shot 예시**  
실제 프로젝트에서 잘 작성된 코드 예시 1-2개를 스펙과 함께 주입하면 품질이 크게 향상될 수 있음. 현재 코드 스캐너가 인터페이스 시그니처를 추출하지만, 구현 예시(few-shot)는 제외하고 있음.

---

## 구체적 개선 제안 목록 {#개선-제안}

### 즉시 적용 가능 (코드 변경 없음)

| 개선 | 방법 | 기대 효과 |
|------|------|-----------|
| **타겟 레포에 CLAUDE.md 작성** | `your-repo/CLAUDE.md`에 아키텍처, 컨벤션 기록 | 매 실행 프롬프트 크기 감소, 일관성 향상 |
| **레포에 .claude/skills/ 추가** | 자주 쓰는 패턴을 스킬 파일로 작성 | 재사용 가능한 워크플로우 |
| **settings.json 시크릿 분리** | 환경 변수 or keychain으로 토큰 관리 | 보안 위험 제거 |
| **검증 명령어를 스펙에 포함** | implementation_plan에 `curl` 테스트 명령어 추가 | Claude Code가 자기검증 가능 |

### 코드 개선

| 개선 | 위치 | 방법 |
|------|------|------|
| **`git add -A` → selective staging** | `execute.py:219` | `git diff --name-only HEAD` 기반 스테이징 |
| **Few-shot 예시 주입** | `spec_generator.py` | 코드 스캐너에서 유사 구현 파일 추출, 예시로 주입 |
| **평가 개선: XML 구조 유효성** | `evaluation.py:57` | `ET.fromstring(p.prompt)` 파싱 성공 여부로 검사 |
| **git history 컨텍스트** | `local_scanner.py` | `git log --oneline -20` 결과를 레포 컨텍스트에 포함 |
| **Writer/Reviewer 분리** | `execute.py` | 구현 후 별도 Claude Code 실행으로 리뷰 |

### 아키텍처 수준 개선

| 개선 | 설명 | 난이도 |
|------|------|--------|
| **MCP 서버 통합** | Notion/Figma MCP를 Claude Code에 직접 연결, 워크플로우 레이어 단순화 | 중간 |
| **Plan Mode 활용** | Claude Code에 `--print` 외에 Plan Mode API 활용, 스펙 생성 전 탐색 | 중간 |
| **Docker 샌드박스** | `--dangerously-skip-permissions` 대신 컨테이너 격리 | 높음 |
| **SubAgent 검증 루프** | 구현 후 별도 SubAgent로 검증 실행 | 중간 |
| **git history 학습** | PR 메시지, 이슈 내용을 컨텍스트로 활용 (SWE-RL 연구 방향) | 높음 |

### 레거시 프로젝트 전용 개선

현재 워크플로우의 레거시 한계를 극복하는 별도 전략:

```
레거시 전용 CLAUDE.md 템플릿:
  - "테스트가 없는 코드는 변경 전 Characterization Test를 먼저 작성"
  - "DB 직접 접근 코드는 별도 분리 없이 현 패턴 유지"  
  - "컴파일 에러 0 보장 후 커밋"
  - "기존 동작 변경 금지 — 새 기능만 추가"

+ 변경 전 자동 스냅샷 스킬:
  1. 현재 동작 테스트 캡처 (Characterization Test)
  2. 변경 수행
  3. 캡처된 테스트 재실행으로 회귀 확인
```

---

## 참고 자료

| 종류 | 출처 | 핵심 내용 |
|------|------|-----------|
| **공식 문서** | [code.claude.com/docs/en/best-practices.md](https://code.claude.com/docs/en/best-practices.md) | 컨텍스트 관리, 검증 기준, CLAUDE.md 설계 |
| **공식 문서** | [code.claude.com/docs/en/memory.md](https://code.claude.com/docs/en/memory.md) | CLAUDE.md 계층, 200줄 제한, Auto Memory |
| **공식 문서** | [code.claude.com/docs/en/sub-agents.md](https://code.claude.com/docs/en/sub-agents.md) | SubAgent 격리 실행 |
| **논문** | arXiv:2502.18449 (SWE-RL, 2025) | RL on git history → 41% SWE-bench solve rate |
| **논문** | arXiv:2403.08299 (AutoDev, 2024) | Docker 격리 AI 에이전트 → 91.5% HumanEval |
| **논문** | Liu et al. TACL 2024 "Lost in the Middle" | 중간 컨텍스트 30% 정확도 하락 — 코드에 적용됨 |
| **논문** | Schulhoff et al. 2024 "The Prompt Report" | Few-shot이 가장 효과적, 현재 미적용 |
| **논문** | arXiv:2411.10541 | Claude=XML, GPT=YAML 17.7pt 차이 — 코드에 적용됨 |
| **벤치마크** | SWE-bench Verified 2025 | Claude 3.7 Sonnet ~62%, 나머지 38%는 실패 |
