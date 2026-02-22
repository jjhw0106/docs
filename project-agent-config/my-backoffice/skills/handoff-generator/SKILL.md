# Handoff Generator Skill

Gemini에게 전달하는 구현 핸드오프 문서를 생성하는 표준 스킬.

## 출력 위치
`docs/handoff/pending/handoff-{service}-{layer}-v{N}.md`

## 핸드오프 문서 작성 단계

### 1. 전제 확인
- 대응하는 PRD가 `docs/prd/`에 존재하는지 확인
- 기존 핸드오프 파일이 있으면 버전 확인 (스펙 변경 시만 버전업)

### 2. 핸드오프 문서 구조

```markdown
# Handoff: {서비스} {레이어} - {기능명}

> 작성일: {date} | 버전: v{N} | 대상: Gemini {Backend/Frontend} Developer
> 상태: PENDING | 관련 PRD: docs/prd/prd-{feature}.md

---

## 1. 구현 목표
## 2. 기술 스택 및 환경
## 3. 프로젝트 구조
## 4. 구현 상세 스펙

### 4.1 {컴포넌트/모듈명}
**목적:**
**구현 내용:**
**코드 예시 또는 패턴:**

## 5. 의존성 및 통합 포인트
## 6. 완료 기준 체크리스트
- [ ] 항목 1
- [ ] 항목 2

## 7. 주의사항 및 제약조건

---

## Q&A 섹션
> Gemini의 질문은 여기에 추가됨

### QUESTIONS
(Gemini가 작성)

### ANSWERS
(Main이 작성)

### CONFIRMED
(Gemini가 확인 후 표시)

### RESOLVED
(Main만 체크 가능 — 모든 Q&A 완료 후)
```

### 3. 저장 및 상태 관리
- 신규: `docs/handoff/pending/handoff-{service}-{layer}-v1.md`
- 스펙 변경 시: 버전업 (v2, v3 ...)
- Q&A는 같은 파일에 append (버전업 없음)
- 구현 완료 후 리뷰 완료: `docs/handoff/reviewed/`로 이동 + 파일명에 `-final` 추가

## 네이밍 규칙
- `handoff-{service}-{layer}-v{N}.md`
- service: `auth`, `gateway`, `scraper`, `lotto`, `frontend`
- layer: `backend`, `frontend`
- 예: `handoff-lotto-backend-v1.md`, `handoff-frontend-resume-v2.md`
