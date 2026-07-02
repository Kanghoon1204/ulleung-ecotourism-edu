# 울릉군 생태관광 AI 교육 워크숍 2026

## 프로젝트 개요

울릉군 글로벌 생태관광 전문인재 양성 교육용 정적 웹사이트.
GitHub Pages로 배포되며, 모든 페이지는 단일 HTML 파일(인라인 CSS/JS)로 구성.

## 파일 구조

```
index.html              # 수업 당일 메인 실습 진행판
class.html              # index.html로 이동하는 보조 입구
prompt-guide.html       # index.html의 복사 문장 구간으로 이동하는 보조 입구
교육계획서_완성본.html    # 교육 계획서 HTML 버전
example.html            # 예시 페이지
assets/                 # 울릉도 이미지 자료
```

## 기술 스택 & 코딩 규칙

- **단일 파일 HTML**: 각 페이지는 `<style>`과 `<script>`를 인라인으로 포함 (외부 파일 분리 없음)
- **CSS 변수**: `:root`에 정의된 디자인 토큰 사용 (`--primary`, `--secondary`, `--accent` 등)
- **폰트**: `Noto Sans KR` (Google Fonts CDN)
- **언어**: 한국어 우선, 콘텐츠는 자연스러운 문체
- **반응형**: 모바일 대응 필수 (`max-width` 미디어 쿼리)
- **배포**: `main` 브랜치 push → GitHub Pages 자동 배포

## 외부 에이전트 (Scout / Gemini)

### 스크립트 위치

| 스크립트 | 모델 | 역할 |
|---------|------|------|
| `.claude/scripts/call-scout.sh` | Llama 4 Scout (OpenRouter) | 코드/콘텐츠 **생성** |
| `.claude/scripts/call-gemini.sh` | Gemini 2.5 Flash (Google) | 코드/콘텐츠 **리뷰** |
| `.claude/scripts/call-pipeline.sh` | Scout → Gemini 순차 | 생성 + 리뷰 통합 파이프라인 |

### 필수 환경변수

```bash
OPENROUTER_API_KEY   # Scout용 (OpenRouter)
GEMINI_API_KEY       # Gemini용 (Google AI Studio)
```

### 사용법

#### 1. 파이프라인 (Scout 생성 → Gemini 리뷰) — 권장

새 페이지 생성이나 대규모 수정 시 사용:

```bash
echo '{
  "task": "울릉도 독도 관광 안내 페이지 제작",
  "instructions": "index.html의 디자인 토큰과 레이아웃 패턴을 따를 것",
  "reference_files": ["index.html"],
  "output_files": ["dokdo.html"],
  "mode": "implement"
}' | bash .claude/scripts/call-pipeline.sh
```

**모드 종류:**
- `implement` — 새 코드/콘텐츠 생성
- `review` — 기존 파일 분석·검수
- `fix` — 리뷰 결과 기반 수정 (previous_review, previous_code 필드 포함)

#### 2. Scout 단독 — 빠른 생성

간단한 코드 생성이나 텍스트 작성:

```bash
echo "울릉도 성인봉 등산 코스를 소개하는 카드 컴포넌트 HTML을 만들어줘. CSS 변수: --primary: #2563eb" \
  | bash .claude/scripts/call-scout.sh
```

#### 3. Gemini 단독 — 리뷰/검수

기존 코드 리뷰나 콘텐츠 검수:

```bash
echo "아래 HTML의 접근성, 반응형, SEO 문제를 분석해줘:
$(cat class.html)" \
  | bash .claude/scripts/call-gemini.sh
```

## 워크플로 가이드

### 새 HTML 페이지 생성

1. 기존 페이지(index.html 등)를 reference_files로 지정
2. `call-pipeline.sh`를 `mode: implement`로 호출
3. 결과 JSON에서 `scout_code`를 파일로 저장
4. `gemini_review.verdict`가 `FAIL`이면 critical 항목을 수정
5. Claude가 최종 검토 후 파일 작성

### 기존 페이지 수정

- 소규모 수정: Claude가 직접 Edit 도구로 수정
- 대규모 수정: Scout에게 수정 코드 생성 위임 후 Gemini 리뷰

### 코드 리뷰

```bash
echo '{
  "task": "class.html 접근성 및 반응형 점검",
  "instructions": "WCAG 2.1 AA 기준, 모바일 360px 이상",
  "reference_files": ["class.html"],
  "mode": "review"
}' | bash .claude/scripts/call-pipeline.sh
```

### 리뷰 결과 기반 수정

```bash
echo '{
  "task": "리뷰 피드백 반영 수정",
  "instructions": "critical 항목 모두 수정",
  "reference_files": ["class.html"],
  "output_files": ["class.html"],
  "previous_review": "... gemini_review 결과 ...",
  "previous_code": "... scout_code ...",
  "mode": "fix"
}' | bash .claude/scripts/call-pipeline.sh
```

## 에이전트 호출 판단 기준

| 상황 | 사용 도구 |
|------|----------|
| 1~2줄 간단 수정 (오타, 색상 변경) | Claude 직접 Edit |
| 새 섹션/컴포넌트 추가 (50줄+) | Scout 단독 또는 파이프라인 |
| 새 HTML 페이지 전체 생성 | **파이프라인 필수** |
| 기존 코드 품질 검수 | Gemini 단독 또는 파이프라인(review) |
| 콘텐츠 텍스트만 작성 | Scout 단독 |
| 버그 수정 후 검증 | 파이프라인(fix) |

## 주의사항

- Scout/Gemini 호출 전 `OPENROUTER_API_KEY`, `GEMINI_API_KEY` 환경변수 확인
- 파이프라인은 순차 실행 (Scout → Gemini) — 응답에 30초~2분 소요
- Scout max_tokens 기본 32,000 / Gemini max_output_tokens 기본 65,536
- 파이프라인 결과는 JSON으로 반환됨. `scout_code`에서 `---FILE: path--- ... ---END---` 패턴으로 파일 추출
- `generate_docx.py` 실행 시 `python-docx` 패키지 필요 (`pip install python-docx`)
