# AI Workflow Improvement Study

AI 코딩 워크플로우 개선 시도에 대한 분석 및 연구 문서.

---

## 배경

Claude Code를 활용한 개발 생산성 향상을 위해 3가지 워크플로우를 직접 시도하고,  
실제 실행 데이터와 논문/공식 문서 기반으로 각 방식의 효용을 분석했다.

---

## 시도한 워크플로우

### 1. Notion/Figma → AI → Spec/Prompt → Claude Code
Notion 백로그와 Figma 디자인 링크를 AI에 넘겨 인터페이스 설계 및 프롬프트를 자동 생성한 뒤 Claude Code를 실행하는 방식.

### 2. 하네스 엔지니어링 (Harness Engineering)
AI 에이전트의 안정적 동작을 위한 런타임 제어 레이어 설계.  
테스트 기반 검증, Human-in-the-Loop, Observability, LLM-as-Judge 패턴 적용.

### 3. AI Skills Interface 등록
기능 명세만 작성하면 AI가 프로젝트에 맞는 구현을 자동으로 수행하는 스킬 등록 방식.  
참고: [common-ai-skill](https://github.com/jaemyeong-hwnag/common-ai-skill)

---

## 문서 구조

```
├── docs/
│   ├── 01_workflow_improvement_analysis.md   # 3가지 시도 상세 분석 + 엔지니어링 기법 조사
│   ├── 02_workflow_improvement_v2.md         # 추가 개선점 및 레퍼런스
│   ├── 03_why_workflow_is_unnecessary.md     # 시도 1이 불필요한 이유 (실행 데이터 기반)
│   └── 04_ai_workflow_guide.md              # 상황별 실전 가이드 (신규/레거시/멀티레포)
```

---

## 결론 요약

| 상황 | 권장 방식 |
|------|----------|
| 신규 기능 개발 | CLAUDE.md 작성 + Claude Code CLI 직접 |
| 레거시 유지보수 | Plan Mode + Characterization Test + 최소 변경 |
| 멀티 레포 대규모 작업 | MCP 연동 또는 워크플로우 서버 |
| 모든 경우 공통 | Human-in-the-Loop 필수 (AI solve rate 최대 62%) |

---

## 주요 참고 자료

| 분류 | 출처 |
|------|------|
| 공식 문서 | [Claude Code Best Practices](https://code.claude.com/docs/en/best-practices.md) |
| 공식 문서 | [Claude Code Memory (CLAUDE.md)](https://code.claude.com/docs/en/memory.md) |
| 논문 | Liu et al. "Lost in the Middle" TACL 2024 |
| 논문 | arXiv:2411.10541 — XML vs YAML for LLM |
| 논문 | arXiv:2502.18449 — SWE-RL (2025) |
| 논문 | arXiv:2403.08299 — AutoDev (2024) |
| 논문 | Schulhoff et al. "The Prompt Report" 2024 |
| 벤치마크 | SWE-bench Verified 2024-2025 |
