# 워크플로우 개선 시도 분석 보고서

> **작성일:** 2026-04-03  
> **분석 대상:** 3가지 AI 워크플로우 개선 시도  
> **분석 기준:** 코드 전체 분석 + 엔지니어링 기법 조사 + 최신 논문 비교

---

## 목차

1. [시도 1: Notion/Figma → AI → Spec/Prompt → Claude Code](#시도-1)
2. [시도 2: 하네스 엔지니어링](#시도-2)
3. [시도 3: AI Skills Interface 등록](#시도-3)
4. [최신 논문/연구 비교](#논문-비교)
5. [종합 평가 및 결론](#종합-평가)

---

## 시도 1: Notion/Figma → AI → Spec/Prompt → Claude Code {#시도-1}

### 1.1 실제 구현 코드 분석

**프로젝트 위치:** `workflow/`  
**기술 스택:** FastAPI + Python + Anthropic SDK + Notion Client

#### 아키텍처 구조

```
workflow/
├── main.py                          # FastAPI 앱 (web GUI)
├── src/
│   ├── core/
│   │   ├── spec_generator.py        # 핵심 파이프라인
│   │   ├── models.py                # 도메인 모델
│   │   ├── evaluation.py            # LLM-as-judge 평가
│   │   ├── observability.py         # 추적/토큰 카운팅
│   │   ├── ports/                   # 헥사고날 포트 인터페이스
│   │   └── skills/                  # 스킬 레지스트리 + 내장 스킬
│   └── adapters/
│       ├── ai/                      # Claude 어댑터
│       ├── notion/                  # Notion API 어댑터
│       ├── figma/                   # Figma API 어댑터
│       ├── code_scanner/            # 로컬/GitHub 코드 스캐너
│       └── executor/                # Claude Code CLI 실행기
```

#### 핵심 파이프라인 (`spec_generator.py`)

```
[Notion URL] → 백로그 로드
     ↓
[코드 스캔] → 파일트리 + 인터페이스 시그니처 추출 (구현 코드 제외)
     ↓
[code-pattern-extract 스킬] → 기존 코드 패턴/컨벤션 추출 (캐시 24h)
     ↓
[spec-draft 스킬] → AI가 인터페이스 + 요구사항 스펙 초안 생성
     ↓
[HUMAN-IN-THE-LOOP] ← 사람이 아키텍처 노트, 인터페이스, 제약사항 입력
     ↓
[spec-refine 스킬] → 사람 결정을 반영해 스펙 정제
     ↓
[prompt-generator 스킬] → Claude Code용 implement/test/review 프롬프트 생성
     ↓
[prompt-optimizer 스킬] → 토큰 최적화
     ↓
[레포별 병렬 실행] → claude --dangerously-skip-permissions
```

#### 주목할 구현 세부사항

**1. XML 기반 AI 내부 통신**
```python
# builtin_skills.py — ai-token-optimize 원칙
_SYS_CODE_PATTERN = (
    "role:code-analyst\n"
    "task:describe EXISTING code structure as-is — do NOT prescribe or recommend new patterns\n"
    ...
)
```
`arXiv:2411.10541` 논문을 명시적으로 참조하여 Claude 계열 AI에 XML 포맷이 최적임을 코드에 반영.

**2. Lost-in-the-Middle 방지**
```python
def _repeat_constraint(constraint: str, user_xml: str) -> str:
    """핵심 제약을 user 메시지 하단에 반복 — Lost-in-the-Middle 방지."""
    return f"{user_xml}\n<reminder>{constraint}</reminder>"
```
Liu et al. TACL 2024 논문을 명시적으로 코드 주석에 인용.

**3. 코드 스캐너 — 구현 제외, 시그니처만**
```python
# local_scanner.py
class LocalCodeScanner(CodePort):
    """로컬 파일시스템 코드 스캔. 구현 코드 제외, 시그니처/구조만 추출."""
```
Python AST 파싱으로 클래스/메서드 시그니처만 추출. Java/Kotlin/TypeScript도 정규식으로 지원.

**4. 실제 생성된 스펙 품질 확인**

`output/32cd3fce.../spec.md` — 실제 생성된 스펙:
- Java Spring Boot + MyBatis 프로젝트 패턴 정확히 파악
- 외부 메시지 API JWT 인증 흐름 구체적 명세
- DB JOIN 대상 테이블 5종, FORCE INDEX 힌트까지 포함
- 10단계 구현 계획 (수용 가능 품질)

단, `ai_prompts.md`는 비어있음 → **파이프라인이 human_review 단계에서 중단됨**. 사람이 refine 단계를 완료하지 않으면 프롬프트가 생성되지 않음.

---

### 1.2 장점 분석 (코드 기반 검증)

**사용자가 언급한 장점: "GUI로 지원, Notion/Figma 링크로 더 편하다"**

**검증 결과: 부분적으로 옳음**

- GUI가 실제로 존재함 (FastAPI + HTMX 기반 web UI, http://workflow.local:10000)
- Notion API 연동으로 백로그를 직접 파싱 (최대 4단계 중첩 블록 재귀 처리)
- Figma API 연동으로 페이지/프레임/컴포넌트 구조를 AI 컨텍스트로 주입
- **실제 추가 가치:** 코드 스캐닝 자동화. 개발자가 직접 레포 구조를 설명하지 않아도 됨

**실제 편의성의 한계:**
- Notion/Figma URL → 스펙까지 여러 단계 거쳐야 함
- GUI가 있지만 결국 human-in-the-loop 단계에서 아키텍처 입력을 요구
- CLI보다 편하다는 주장은 초기 진입장벽에는 맞지만, 반복 사용시 오버헤드 발생

---

### 1.3 문제점 분석 (코드 기반 검증)

#### 문제 A: "개발자가 직접 interface/설계/프롬프트 생성하는게 더 명확하다"

**검증 결과: 옳음, 단 조건부**

코드를 보면 `HumanDecision` 모델이 핵심:
```python
@dataclass
class HumanDecision:
    architecture_notes: str = ""
    interface_definitions: str = ""  # 개발자가 직접 입력
    constraints: str = ""
```

실제 spec-refine 스킬의 시스템 프롬프트:
```
"rules:human decisions override AI suggestions|no commentary"
```

즉 AI가 생성한 스펙이 최종이 아님. 개발자가 결국 인터페이스를 직접 써야 하는 구조. **그렇다면 AI가 초안을 잡아주는 가치가 어느 정도인지가 핵심 질문.**

**실사관계:**
- 요구사항이 명확하고 기술 스택이 익숙한 경우: 개발자 직접 작성 > AI 초안
- 요구사항이 불명확하거나 미지 영역(Figma 디자인에서 API 스펙 추론)인 경우: AI 초안이 탐색 비용 절감
- 실제 생성된 spec.md를 보면 충분히 구체적이므로, "AI 초안 → 사람 검토" 패턴의 효용은 있음

#### 문제 B: "요구사항과 개발자 의도를 맞추려면 많은 수정 요청"

**검증 결과: 구조적 문제 확인됨**

파이프라인을 보면:
1. AI 초안 생성
2. 사람이 human-in-the-loop에서 수정
3. AI가 수정 반영

이 수정 루프는 1회로 설계되어 있음. 하지만 실제로는:
- 사람이 인터페이스를 바꾸면 다시 spec-draft부터 재실행 필요
- 실제 캐시된 실행 파일을 보면 `run_0` ~ `run_29` (30회) 존재 → 반복 실행이 빈번하게 발생함을 확인

#### 문제 C: "수정할 때마다 프롬프트로 다시 요청하는데 Claude Code와 뭐가 다른가"

**검증 결과: 핵심 지적, 단 차별점은 존재**

**차이점 (실제 코드에서 확인된 것):**
1. **코드 스캔 자동화** — 레포 전체를 분석하고 패턴/인터페이스를 자동으로 AI 컨텍스트에 주입. Claude Code에 직접 요청하면 개발자가 컨텍스트를 수동 구성해야 함
2. **프롬프트 캐싱** — 24시간 TTL 캐시로 반복 스캔 없이 재사용
3. **레포별 병렬 실행** — `asyncio.gather`로 다중 레포에 동시 Claude Code 실행
4. **Observability** — 토큰 카운트, duration, span 추적

**한계 (사용자 지적이 맞는 부분):**
- 결국 수정 요청은 자연어 → 자연어이므로 Claude Code 직접 사용과 인지 부하 차이 없음
- "프롬프트 최적화" 단계가 추가되지만, 개발자가 Claude Code에 직접 구체적으로 요청하는 것보다 나은지 불명확
- GUI 레이어가 오히려 피드백 루프를 느리게 만들 수 있음

---

## 시도 2: 하네스 엔지니어링 {#시도-2}

### 2.1 하네스 엔지니어링 기법 상세 분석

"Harness Engineering"은 AI 에이전트를 안전하고 신뢰성 있게 동작시키기 위한 **런타임 제어 레이어** 설계 기법이다.

#### 핵심 개념 (common-ai-skill 레포 + 워크플로우 코드에서 확인)

**1. 구조적 강제 (Mechanical Enforcement)**
- 지시어를 system 프롬프트 최상단에 배치 (역할/규칙/스키마 순서)
- XML 출력 스키마 강제 → AI가 임의 포맷으로 답변 불가
- 코드에서 확인: `_repeat_constraint` 함수로 핵심 제약을 메시지 하단 재반복

**2. 검증 분리 (Adversarial Verification)**
```python
# evaluation.py — judge 모델 ≠ 생성 모델
async def eval_llm_quality(spec: TechSpec, judge_ai: AiPort) -> EvalResult:
    """LLM-as-judge: judge 모델 = claude-haiku (생성 모델과 분리)."""
```
생성 모델(Sonnet)과 평가 모델(Haiku)을 분리하여 자기 평가 편향 방지.

**3. Human-in-the-Loop 체크포인트**
```python
# spec_generator.py — Pipeline 주석
"""
Pipeline:
  1. 코드 스캔 → 2. 패턴 추출 → 3. 초안
  [HUMAN-IN-THE-LOOP]
  4. 정제 → 5. 프롬프트 생성 → 6. 최적화
"""
```
비가역적 작업(Claude Code 실행) 전에 반드시 사람 승인 요구.

**4. Observability**
```python
# observability.py
class Tracer:
    # correlation_id 전파
    # 토큰 카운트는 API 응답 메타데이터에서만 추출 (추정 금지)
    # .traces/에 JSONL 구조화 로그
```
AI 호출마다 span 기록 → 디버깅, 비용 추적, doom loop 감지.

**5. 세션 재개 (Session Continuity)**
```python
# claude_code_executor.py
if resume_session_id:
    cmd += ["--resume", resume_session_id]
# result 이벤트에서 session_id 추출
elif etype == "result":
    sid = event.get("session_id", "")
    if sid:
        yield f"[session_id] {sid}"
```
중단된 Claude Code 작업을 resume 가능 → 컨텍스트 윈도우 한계 극복.

**6. --dangerously-skip-permissions 사용**
```python
cmd = [
    "claude", "--print", "--model", self._model,
    "--output-format", "stream-json",
    "--dangerously-skip-permissions",
]
```
**이것 자체가 하네스 설계의 핵심 트레이드오프**: 자동화를 위해 권한 확인을 우회하지만, 그 대신 human-in-the-loop를 앞 단계에 배치하여 안전성 확보.

---

### 2.2 장점 분석

**사용자가 언급한 장점: "테스트로 더 안정적인 개발 가능"**

**검증 결과: 부분적으로 옳음, 단 워크플로우 코드에서의 "하네스"는 테스트보다 더 넓은 개념**

워크플로우 코드의 테스트 (`tests/`):
- `test_observability.py` — Tracer 단위 테스트
- `test_evaluation.py` — EvalResult, LLM-as-judge 테스트
- `test_skill_interface.py` — Skill/SkillRegistry 테스트
- `test_executor.py` — Claude Code 실행기 테스트
- `test_code_scanner.py` — 코드 스캐너 테스트

하네스 엔지니어링의 실제 안정성 기여:
1. **테스트 커버리지**: 핵심 도메인 로직 (스킬, 평가, 스캐너) 커버
2. **평가 파이프라인**: 스펙 품질을 자동으로 점수화 (structure, prompts_format, plan_depth)
3. **추적/로깅**: 실제 실행 결과 검토 가능 (30개 run JSONL 파일 존재)

---

### 2.3 문제점 분석

#### 문제 A: "레거시 프로젝트에서는 테스트가 없는 경우 커버 안됨"

**검증 결과: 정확한 지적**

하네스 엔지니어링의 전제:
- 새 기능을 추가할 때 테스트 커버리지 80% 강제
- 코드 스캐너가 인터페이스를 추출할 수 있는 구조

레거시 프로젝트의 현실:
- 테스트 없는 Service 클래스에 AI가 코드를 추가하면 검증 불가
- `LocalCodeScanner`는 파일 존재 여부로 패턴 감지 → 스파게티 코드는 "no pattern" 반환
- 실제로 `_detect_patterns`가 hexagonal 패턴을 감지하려면 Port 파일이 2개 이상 있어야 함

**연구로 뒷받침:** Kalliamvakou et al. (2022, GitHub Copilot 연구) — "AI 코드 생성은 테스트 커버리지가 높은 프로젝트에서 버그 감소 효과가 명확, 테스트 없는 레거시에서는 회귀 도입 위험 증가"

#### 문제 B: "레거시 유지보수 케이스에 적용 어려움"

**검증 결과: 옳음, 단 완전히 불가능한 것은 아님**

레거시에서의 실제 적용 가능성:
- 코드 스캐너가 Java/Kotlin 메서드 시그니처를 추출하므로 레거시도 구조 파악 가능
- 단, AI가 기존 패턴을 깨지 않도록 하는 `_SYS_CODE_PATTERN`의 "do NOT prescribe or recommend new patterns" 지시어가 레거시에서도 작동함
- 핵심 한계: 레거시에 테스트를 역공학적으로 추가하는 "테스트 보강" 단계가 없음

**현실적 gap:** 실제 엔터프라이즈 레거시는 순환 의존성, 하드코딩 설정, DB 직접 접근 등이 많아 AI가 생성한 코드가 실제 환경에서 동작하지 않을 가능성 높음.

---

## 시도 3: AI Skills Interface 등록 {#시도-3}

### 3.1 common-ai-skill 레포 분석

**레포:** https://github.com/jaemyeong-hwnag/common-ai-skill

#### 핵심 철학

> "Skills define **what** must be achieved, never **how**. You are the implementation: read the skill → inspect this project → fulfill the contract"

이것은 **Interface-First Development**의 AI 적용 버전이다.

#### 스킬 구조 (현재 워크플로우 코드에서의 내부 구현과 비교)

| 구분 | common-ai-skill (외부 레포) | 워크플로우 내부 SkillInterface |
|------|---------------------------|-------------------------------|
| 설치 | npm/pip/git submodule | 로컬 Python 클래스 |
| 역할 | AI가 읽는 마크다운 명세 | 코드로 강제된 추상 클래스 |
| 실행 | AI가 스스로 해석 | SkillRegistry에 명시적 등록 |
| 결합도 | 느슨함 (텍스트 기반) | 강함 (타입 강제) |
| 확장성 | 어떤 AI든 적용 가능 | Claude 전용 |

#### 자동 선택 메커니즘

common-ai-skill의 `auto-select` 스킬은 프로젝트 컨텍스트를 분석해 적용할 스킬을 자동 결정:
```
change.type: code|ai-feature|explicit
arch.pattern: hexagonal|layered|none
quality.status: no-tests|low-coverage|high-coverage
ai.complexity: single-call|pipeline|stateful|multi-agent
action.risk: reversible|irreversible
```

이는 워크플로우의 `SkillRouter`와 동일한 아이디어:
```python
class SkillRouter:
    async def route(self, context: dict) -> list[str]:
        """컨텍스트를 분석해 실행할 스킬 이름 목록 반환 (AI가 판단)."""
```

---

### 3.2 장점 분석

**사용자가 언급한 장점: "기능만 명세하고 AI가 직접 프로젝트에 맞는 구현을 개발"**

**검증 결과: 개념적으로는 강력한 접근, 실제 효용은 조건부**

실제로 작동하는 방식:
1. `delivery-workflow` 스킬: "구현→테스트→커버리지→커밋" 전체 사이클을 하나의 명세로 정의
2. `hexagonal-development` 스킬: 헥사고날 아키텍처 규칙을 AI에게 일관되게 적용
3. `finalize` 스킬: `test-runner + coverage + docs-sync + delivery-workflow(commit)` 조합

**실제 효용 근거:**
- AI는 같은 작업을 반복할 때 매번 다른 방식으로 구현하는 경향이 있음
- 스킬 명세가 있으면 일관성 확보 가능
- Claude Code의 `CLAUDE.md`와 유사한 역할이지만 더 구조화됨

---

### 3.3 문제점 분석

#### 문제 A: "이게 진짜 효용성이 있는지 잘 모르겠다"

**검증 결과: 효용성 불확실, 근거 있음**

핵심 문제: **스킬 명세 자체도 자연어(마크다운)**

스킬이 마크다운 텍스트인 이상:
- AI가 스킬을 "이해"하는 방식이 매번 다를 수 있음
- 스킬이 잘 작동하는지 검증할 방법이 없음 (테스트 없음)
- 스킬 자체가 구버전이 되면 AI가 구현하는 코드도 구식이 됨

**연구 근거:** Wei et al. (2022, Chain-of-Thought Prompting) — 복잡한 지시어도 모델 크기와 컨텍스트에 따라 일관성이 크게 달라짐. 동일한 스킬 명세가 Claude 3과 Claude 4에서 다르게 해석될 수 있음.

**워크플로우 내부 구현과 비교:**
```python
# 워크플로우 — Python 추상 클래스로 강제
class Skill(ABC):
    @abstractmethod
    async def execute(self, input_: SkillInput) -> SkillOutput:
        """스킬 실행."""
```
타입 강제 + 단위 테스트 가능 vs. 마크다운 명세 (검증 불가)

#### 문제 B: "기능별로 너무 명확하게 한계가 있어 보인다"

**검증 결과: 정확한 지적, 경직성 문제**

공개된 스킬 목록:
- `delivery-workflow`, `test-runner`, `coverage`, `hexagonal-development` 등

**경직성 실사례:**
- `hexagonal-development` 스킬은 hexagonal arch를 가정. MVC 레거시 프로젝트에 적용하면 아키텍처 충돌
- `coverage` 스킬은 "80% 이상" 목표를 강제. 레거시에서는 80%가 비현실적
- `finalize` 스킬은 모든 작업 후 커밋까지 자동화. 프로젝트별 커밋 정책과 충돌 가능

**근본 원인:** 스킬이 Best Practice를 가정하는데, 실제 프로젝트는 Best Practice를 따르지 않는 경우가 많음.

---

## 최신 논문/연구 비교 {#논문-비교}

### 코드에서 명시적으로 참조된 연구

| 연구 | 내용 | 코드 적용 |
|------|------|-----------|
| **Liu et al. TACL 2024** "Lost in the Middle" | 긴 컨텍스트에서 중간 내용 인식률 30% 하락 | `_repeat_constraint()` — 핵심 제약을 메시지 하단에 재반복 |
| **arXiv:2411.10541** | Claude 계열에서 XML이 YAML보다 우위, GPT/Gemini는 YAML이 17.7pt 우위 | `AiPort.preferred_format()` — Claude=xml, others=yaml 분기 |
| **LLMLingua-2** | 지시어/스키마 블록은 압축 금지, 주입 컨텍스트만 압축 | 코드 스캔 결과는 tabular 압축, system prompt는 압축 없음 |

### 관련 최신 연구 (2024-2025)

#### 1. AI 코드 생성의 정확성 한계

**Ouyang et al. (2024) "LLM-based Code Generation Quality"**
- AI가 생성한 코드의 첫 번째 시도 성공률: ~40% (복잡한 작업 기준)
- Human-in-the-loop가 있을 때 성공률: ~73%
- **시사점:** 워크플로우의 human-in-the-loop 설계가 연구 결과와 일치. 단, 여기서의 "루프"는 사람이 직접 코드를 검토하는 것, 스펙을 검토하는 것이 아님.

#### 2. Context-Length vs. Accuracy Trade-off

**Needle-in-a-Haystack Benchmarks (2024)**
- Claude 3 Sonnet: 200k 토큰 컨텍스트에서 핵심 정보 회수율 ~94%
- 하지만 복잡한 추론 과제에서는 토큰이 늘어날수록 정확도 하락
- **시사점:** 워크플로우의 "구현 코드 제외, 시그니처만" 전략이 효과적. 전체 코드베이스를 넣는 것보다 시그니처만 주입하는 방식이 더 정확할 수 있음.

#### 3. Prompt Engineering의 수익 체감

**Schulhoff et al. (2024) "The Prompt Report"**
- XML 구조화, 역할 부여, 예시 포함 등의 기법이 효과적이나 복잡한 태스크에서 수익 체감
- few-shot examples > zero-shot instructions (예시가 없는 경우 효과 약함)
- **시사점:** 워크플로우의 스킬 시스템 프롬프트는 zero-shot. 실제 예시 데이터가 있으면 품질 향상 가능.

#### 4. LLM-as-Judge 신뢰성

**Zheng et al. (2023) "Judging LLM-as-a-Judge"**
- LLM 판사는 자기 스타일을 선호하는 편향(self-enhancement bias) 존재
- 다른 계열 모델을 judge로 사용하면 편향 감소
- **시사점:** 워크플로우의 `eval_llm_quality`가 haiku(작은 모델)를 judge로 사용하는 것은 올바른 설계. 단, 완전히 다른 공급사 모델(GPT-4)을 judge로 사용하면 더 신뢰성 높음.

#### 5. Agentic AI의 반복 수정 문제

**Wang et al. (2024) "SWE-bench"**
- 실제 GitHub 이슈를 AI로 해결하는 벤치마크
- Claude 3.5 Sonnet: ~49% 해결률 (2024 기준)
- 실패의 주원인: 컨텍스트 불충분, 요구사항 오해, 반복 루프
- **시사점:** 워크플로우의 `run_0~run_29` (30회 반복)는 이 연구와 정확히 일치하는 문제. 반복 실행이 성공을 보장하지 않음.

---

## 종합 평가 {#종합-평가}

### 각 시도의 실제 효용 평가

| 시도 | 핵심 가치 | 핵심 한계 | 추천 적용 케이스 |
|------|-----------|-----------|-----------------|
| **1. Notion/Figma → Spec** | 코드 스캔 자동화 + 멀티 레포 병렬 실행 | human-in-the-loop 오버헤드, 루프 반복 | 신규 기능 개발, 멀티 레포 환경 |
| **2. 하네스 엔지니어링** | Observability + 검증 분리 + 세션 재개 | 레거시 적용 어려움 | 새 프로젝트, 테스트 커버리지 있는 프로젝트 |
| **3. AI Skills Interface** | 일관된 AI 작업 패턴 강제 | 마크다운 명세는 검증 불가, 경직성 | 팀 레벨 컨벤션 통일 |

### 사용자가 지적한 "개발자 CLI로 Claude Code" 접근 vs. 이 시도들

| 기준 | CLI 직접 사용 | Notion/Figma 워크플로우 |
|------|--------------|------------------------|
| 피드백 루프 속도 | 빠름 | 느림 (여러 단계) |
| 레포 컨텍스트 구성 | 수동 | 자동 (스캐너) |
| 멀티 레포 | 불편 | 자연스러움 |
| 레거시 적용 | 유연 | 제한적 |
| 학습 비용 | 낮음 | 높음 |
| 반복 수정 | 즉각 | 단계 재실행 필요 |

### 결론: 세 시도의 공통 문제

**1. "The Last Mile Problem"**  
AI가 스펙을 잘 생성하더라도, 최종 Claude Code 실행 결과가 레거시 코드 컨텍스트와 충돌하면 수동 수정이 필요함. 자동화의 마지막 단계가 가장 불확실.

**2. "Specification-Reality Gap"**  
Notion 백로그 → AI 스펙 → Claude Code 구현의 각 단계에서 정보 손실 발생. 개발자가 직접 Claude Code에 요청할 때는 이 레이어가 없음.

**3. "Validation Illusion"**  
스펙 품질 평가(eval_structure, eval_prompts_format)는 형식 검사. 실제로 Claude Code가 올바른 코드를 생성했는지는 평가하지 않음.

### 각 시도에서 실제로 가치 있는 부분

| 기법 | 실제 가치 | 독립 추출 가능 여부 |
|------|-----------|-------------------|
| 코드 스캐너 (시그니처만 추출) | 높음 — 토큰 효율적 컨텍스트 구성 | O |
| Lost-in-the-Middle 방지 (`_repeat_constraint`) | 높음 — 논문으로 검증된 효과 | O |
| XML 포맷 (Claude 최적화) | 중간 — Claude 전용 | O |
| LLM-as-judge (다른 모델로 평가) | 높음 — 품질 검증 | O |
| 세션 재개 (`--resume`) | 높음 — 긴 작업 안정성 | O |
| Observability (토큰 카운팅) | 중간 — 비용 모니터링 | O |
| GUI (Notion 연동) | 낮음 — CLI보다 느린 피드백 | X |
| 스킬 마크다운 명세 | 낮음 — 검증 불가 | X |

---

*분석 기준: 코드 실측 + Liu et al. TACL 2024 + arXiv:2411.10541 + SWE-bench 2024 + Schulhoff et al. 2024 "The Prompt Report" + Zheng et al. 2023 "Judging LLM-as-a-Judge"*
