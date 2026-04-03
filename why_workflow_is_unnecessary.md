# 워크플로우 프로젝트가 필요 없는 이유

> 감정이나 추상적 판단이 아닌, 실제 실행 데이터 기반으로 분석

---

## 실제 사용 데이터

```
날짜: 2026-04-02 하루

실행 횟수: 30회
실행 시간: 12:25 ~ 16:07 (약 3시간 42분)
총 로그 라인: 405줄
생성된 프롬프트 크기: 9,952자
```

---

## 이유 1. 워크플로우가 해결하려는 문제를 본인이 더 잘 안다

워크플로우가 생성한 프롬프트의 `user_constraints` 섹션:

```
스케쥴러는 없고 일단 api 로 비지니스 로직만 구현
works 발송은 core 에 구현
https://developers.worksmobile.com/kr/docs/bot-channel-message-send 참고
```

이건 **개발자가 직접 입력한 것**이다. Notion에서 읽어온 게 아니라, GUI에서 직접 타이핑한 내용이다.

즉 아키텍처 결정(모듈 위치, 패키지 구조)은 이미 개발자 머릿속에 있었고, 워크플로우는 그걸 XML로 감싸줬을 뿐이다.

**Claude Code에 직접 쓰는 것과 차이가 없다.**

---

## 이유 2. 중간 레이어(스펙)가 실제로는 건너뛰어졌다

파이프라인 설계:
```
Notion → spec-draft → [HUMAN REVIEW] → spec-refine → prompt-generator → Claude Code
```

실제 결과:
```
spec.md:      status: "human_review" (완료 안 됨)
ai_prompts.md: 비어 있음 (프롬프트 생성 안 됨)
```

human_review 단계에서 멈추고, 개발자가 **직접 수정한 프롬프트**로 Claude Code를 30회 실행했다.

스펙 생성 레이어는 통과되었지만 실제 작업에 쓰이지 않았다. 우회된 것이다.

---

## 이유 3. 30회 반복은 워크플로우가 막지 못했다

run_0의 첫 번째 오류:
```
RSA Private Key가 없어서 loadPrivateKey() IOException 발생.
signing-key에 Webhook secret을 넣었는데 이건 RSA 키가 아님.
```

이 오류는 **스펙 생성 단계에서 잡을 수 없다.** 실제 실행해봐야 나오는 오류다.

30회 실행 이유: 코드 실행 → 런타임 오류 → 수정 → 재실행. 이건 워크플로우 유무와 관계없이 발생하는 디버깅 사이클이다.

스펙이 아무리 잘 만들어져도 런타임 오류는 막지 못한다. Claude Code에 직접 "오류가 났어, 고쳐줘"라고 하는 게 동일하다.

---

## 이유 4. MCP로 대체 가능

워크플로우가 하는 일:
```
Notion URL → API 호출 → 마크다운 파싱 → AI 압축 → 컨텍스트 주입
Figma URL  → API 호출 → 구조 추출 → XML 변환 → 컨텍스트 주입
```

Claude Code + MCP로 동일하게:
```bash
claude mcp add notion
claude mcp add figma
# 이후 Claude Code가 직접 Notion/Figma 읽음
```

**별도 서버, GUI, 파이프라인 불필요.**

---

## 이유 5. Claude Code가 이미 탐색을 한다

워크플로우의 코드 스캐너가 하는 일:
```
레포 파일 트리 추출
클래스/메서드 시그니처 추출 (Python AST, Java 정규식)
아키텍처 패턴 감지
→ XML로 압축해서 AI에 주입
```

Claude Code가 Plan Mode에서 하는 일:
```
claude --permission-mode plan
"이 레포 구조 파악하고 구현 계획 세워줘"
→ Claude Code가 직접 파일 읽음
```

Claude Code가 직접 읽는 것이 중간 압축보다 정확하다. 시그니처만 추출하면 구현 맥락이 빠지지만, Claude Code는 실제 코드를 읽는다.

---

## 이유 6. 워크플로우 자체를 만드는 데 Claude Code가 쓰였다

이 워크플로우 프로젝트의 코드(`spec_generator.py`, `builtin_skills.py`, `claude_code_executor.py` 등)는 Claude Code로 작성되었을 가능성이 높다. Claude Code로 "Claude Code를 더 잘 쓰기 위한 도구"를 만든 것이다.

즉 **Claude Code를 Claude Code로 감쌌다.**

---

## 유일하게 의미 있었을 케이스 (하지만 실제론 그렇지 않았다)

```
가설: 여러 레포에 동시 반영 + 팀 공유 스펙이 필요한 경우

실제:
  - 레포 하나만 사용됨
  - 팀 공유 없이 혼자 작업
  - 30회 반복 → 스펙이 아닌 직접 수정으로 해결
```

멀티 레포 병렬 실행이 실제로 쓰이지 않았다. 이 경우에는 의미가 없었다.

---

## 결론

| 워크플로우가 제공한 것 | 실제 필요했던 것 |
|----------------------|----------------|
| Notion → XML 변환 | 개발자가 이미 알고 있었음 |
| 스펙 초안 생성 | 실제로 사용되지 않음 (human_review에서 중단) |
| 프롬프트 최적화 | Claude Code가 직접 처리 |
| GUI | 오히려 피드백 루프를 느리게 만듦 |
| 실행 이력 30회 저장 | 시크릿이 평문으로 캐시됨 |

**Claude Code에 직접 요청하는 것과 결과가 동일했고, 중간 레이어가 오히려 복잡도를 더했다.**