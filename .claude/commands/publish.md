---
description: 민감정보 검사 후 정리된 커밋 메시지로 변경사항을 커밋하고 푸쉬한다
argument-hint: "[커밋 메시지 (선택)]"
allowed-tools: Read, Glob, Grep, Bash
---

변경사항을 커밋하고 원격 레포에 푸쉬한다. 푸쉬 전 민감정보를 검사한다.

## 작업 순서

### 1. 변경사항 파악
```
git status
git diff --stat
```

### 2. 민감정보 사전 검사
변경된 파일들에서 다음 패턴을 검색한다:
- 토큰/키 패턴: `ntn_`, `figd_`, `sk-`, `ghp_`, `Bearer `
- 이메일 패턴: `@` + 도메인
- 로컬 경로: `/Users/`, `/home/`
- 크리덴셜처럼 보이는 긴 무작위 문자열

민감정보가 발견되면 **즉시 중단**하고 사용자에게 보고한다.
`/sanitize` 스킬을 먼저 실행하도록 안내한다.

### 3. 커밋 메시지 작성
인자로 메시지가 제공된 경우: `$ARGUMENTS` 사용
제공되지 않은 경우: 변경 내용을 분석하여 Conventional Commits 형식으로 작성

**Conventional Commits 형식:**
```
<type>: <subject>

[선택: 변경 내용 상세]
```

타입 선택 기준:
- `docs`: 문서 추가/수정
- `feat`: 새 스킬/기능 추가
- `fix`: 오류 수정
- `refactor`: 구조 개선 (기능 변경 없음)
- `chore`: 설정, 유지보수

**좋은 커밋 메시지 예시:**
- `docs: add workflow attempt analysis for MCP integration`
- `feat: add sanitize skill for sensitive info removal`
- `docs: update references with SWE-bench 2025 results`

### 4. 스테이징 및 커밋
변경된 문서 파일들만 선택적으로 스테이징한다 (`git add -A` 대신 파일별 지정).

### 5. 푸쉬
```
git push origin main
```

푸쉬 성공 후 커밋 URL을 출력한다.

## 주의사항

- 민감정보가 있으면 절대 푸쉬하지 않는다
- 빈 커밋은 생성하지 않는다 (변경사항 없으면 보고만 한다)
- force push는 명시적 요청이 없는 한 하지 않는다
