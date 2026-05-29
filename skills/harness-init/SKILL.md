---
name: harness-init
description: 프로젝트에 강제 하네스를 설치합니다. 구조화된 기능 md 자동생성, 파일확장자 기반 스킬 라우팅, 변경이력 자동기록, 코드리뷰 자동트리거, script 기반 hooks 강제 게이트(컨텍스트 주입 + 미검증 커밋 차단)를 포함합니다.
user-invocable: true
---

# Harness Init v3 — 완전 자동화 하네스

## 해결하는 5가지 문제

| # | 문제 | 해결 방식 |
|---|------|-----------|
| 1 | md를 안 읽음 | 기능별 md를 초기 생성 + 00-INDEX에 매핑 채움 + hooks가 Read 강제 |
| 2 | 수정 내용 기록 안 함 | 구체적 기록 형식 템플릿 + STEP에서 정확한 기록 위치/포맷 지정 |
| 3 | md가 Claude 비친화적 | frontmatter + 고정 필드 + 짧은 구조화 블록 |
| 4 | 스킬/플러그인 안 씀 | 파일 확장자/디렉토리 기반 자동 판단 규칙 |
| 5 | 리뷰 안 함 | 코드 수정 후 ccpp:review MUST + 미검증 커밋을 hooks가 exit 2로 차단 |

---

## Phase 1: 프로젝트 분석

아래를 Glob/Grep/Read로 자동 수집한다:

```
1. 기술 스택
   - package.json / Cargo.toml / go.mod / requirements.txt / pyproject.toml
   - tsconfig.json / vite.config.* / next.config.*

2. 소스 구조
   - 소스 루트 (src/, client/, server/, app/, lib/)
   - 테스트 (tests/, __tests__/, spec/, *_test.go)
   - docs/ 존재 여부

3. 주요 소스 파일 목록
   - Glob으로 소스 파일 전체 스캔
   - 파일 확장자별 분류 (.tsx, .ts, .py, .go, .rs 등)
   - 디렉토리별 기능 그룹 추론 (auth/, api/, components/, pages/ 등)

4. 배포 환경
   - railway.json, vercel.json, Dockerfile, fly.toml
   - .github/workflows/

5. 기존 CLAUDE.md / docs/ 확인
   - 있으면 → 개선 모드 (기존 보존 + 누락 섹션 추가)
   - 없으면 → 신규 생성
```

---

## Phase 2: 사용자 인터뷰 (AskUserQuestion)

```
Q1: 이 프로젝트의 핵심 기능 3-5개
    (예: "로그인, 결제, 대시보드, 알림")

Q2: 빌드/실행 명령
    (예: npm run build / python -m pytest / cargo build)

Q3: 절대 보존해야 할 기능이나 주의사항
```

---

## Phase 3: 파일 생성

> 핵심 원칙:
> - CLAUDE.md = 모든 강제 지시의 단일 소스. 워크플로우 지시를 docs/로 분리하지 않는다.
> - docs/features/*.md = 기능별 구조화 문서. harness-init 시 초기 생성한다 (빈 파일 금지).
> - 모든 md는 frontmatter + 고정 필드 구조로 작성한다.

---

### 생성 파일 1: CLAUDE.md (프로젝트 루트)

```markdown
# {프로젝트명} — 프로젝트 규칙

## MANDATORY WORKFLOW — 모든 코드 수정에 적용

이 워크플로우는 선택이 아닌 필수다. 어떤 단계도 건너뛸 수 없다.

### STEP 1: 문서 읽기 (코드 수정 전)

코드를 건드리기 전에 반드시 아래를 순서대로 실행한다:

1. `docs/00-INDEX.md`를 Read한다
2. 요청 키워드로 매핑 테이블에서 관련 문서를 찾는다
3. 찾은 문서를 Read하여 기존 동작/의도/변경이력을 파악한다
4. 관련 문서가 없으면:
   - Grep/Explore로 코드를 탐색한다
   - 탐색 결과를 바탕으로 `docs/features/{기능명}.md`를 아래 템플릿으로 새로 생성한다
   - `docs/00-INDEX.md` 매핑 테이블에 행을 추가한다

**금지: 이 단계를 건너뛰고 바로 Edit/Write를 사용하는 행위**

### STEP 2: 요청 분석 → 스킬 선행 호출 (코드 작성 전)

사용자 요청의 키워드를 확인하고, 아래 매핑에 해당하면 코드를 작성하기 전에 해당 스킬을 먼저 호출한다.
이 단계의 목적: 적합한 패턴/설계 가이드를 먼저 로딩하여 더 좋은 코드를 작성하기 위함.

#### 프론트엔드/UI

| 요청 키워드 | 선행 호출 스킬 | 용도 |
|-------------|---------------|------|
| UI, 화면, 페이지, 컴포넌트, 디자인 | `Skill: skill="frontend-design:frontend-design"` | UI 설계 가이드 |
| React, 리액트, 훅, 상태관리 | `Skill: skill="ccpp:react-patterns"` | React 19 패턴 |
| Next.js, SSR, 라우팅 | `Skill: skill="ccpp:vercel-react-best-practices"` | Next.js 최적화 |
| Tailwind, 스타일, CSS | `Skill: skill="ccpp:tailwind-design-system"` | 디자인 시스템 |
| shadcn, 폼, 다이얼로그, 테이블 | `Skill: skill="ccpp:shadcn-ui"` | shadcn/ui 컴포넌트 |
| 랜딩페이지, 프리미엄 UI | `Skill: skill="ccpp:ui-ux-pro-max"` | 빅테크 스타일 UI |

#### 백엔드/API

| 요청 키워드 | 선행 호출 스킬 | 용도 |
|-------------|---------------|------|
| API, 엔드포인트, REST, GraphQL | `Skill: skill="ccpp:api-design-principles"` | API 설계 원칙 |
| FastAPI, 파이썬 서버 | `Skill: skill="ccpp:fastapi-templates"` | FastAPI 템플릿 |
| Django | `Skill: skill="everything-claude-code:django-patterns"` | Django 패턴 |
| Spring Boot, 자바 백엔드 | `Skill: skill="everything-claude-code:springboot-patterns"` | Spring Boot 패턴 |
| 비동기, async, 동시성 | `Skill: skill="ccpp:async-python-patterns"` | 비동기 패턴 |

#### 보안/인증

| 요청 키워드 | 선행 호출 스킬 | 용도 |
|-------------|---------------|------|
| 로그인, 인증, OAuth, JWT, 세션 | `Skill: skill="everything-claude-code:security-review"` | 보안 체크리스트 |
| 결제, 주문, 트랜잭션, 민감 데이터 | `Skill: skill="everything-claude-code:security-review"` | 보안 패턴 |

#### 데이터베이스

| 요청 키워드 | 선행 호출 스킬 | 용도 |
|-------------|---------------|------|
| DB, 데이터베이스, 스키마, 마이그레이션 | `Skill: skill="everything-claude-code:database-migrations"` | DB 마이그레이션 |
| PostgreSQL, SQL, 쿼리 최적화 | `Skill: skill="everything-claude-code:postgres-patterns"` | PostgreSQL 패턴 |

#### 테스트

| 요청 키워드 | 선행 호출 스킬 | 용도 |
|-------------|---------------|------|
| 테스트, TDD, 테스트 먼저 | `Skill: skill="ccpp:tdd"` | TDD 워크플로우 |
| E2E, 브라우저 테스트, Playwright | `Skill: skill="everything-claude-code:e2e-testing"` | E2E 테스트 |
| pytest, 파이썬 테스트 | `Skill: skill="ccpp:python-testing-patterns"` | pytest 패턴 |

#### DevOps/배포

| 요청 키워드 | 선행 호출 스킬 | 용도 |
|-------------|---------------|------|
| Docker, 컨테이너, 도커 | `Skill: skill="everything-claude-code:docker-patterns"` | Docker 패턴 |
| 배포, CI/CD, 파이프라인 | `Skill: skill="everything-claude-code:deployment-patterns"` | 배포 전략 |

#### 언어별 패턴

| 요청 키워드 | 선행 호출 스킬 | 용도 |
|-------------|---------------|------|
| Python, 파이썬 | `Skill: skill="everything-claude-code:python-patterns"` | Python 패턴 |
| Go, 고랭 | `Skill: skill="everything-claude-code:golang-patterns"` | Go 패턴 |
| Rust, 러스트 | `Skill: skill="everything-claude-code:rust-patterns"` | Rust 패턴 |
| Kotlin, 코틀린 | `Skill: skill="everything-claude-code:kotlin-patterns"` | Kotlin 패턴 |
| TypeScript 타입, 제네릭 | `Skill: skill="ccpp:typescript-advanced-types"` | TS 고급 타입 |

#### 리서치/문서

| 요청 키워드 | 선행 호출 스킬 | 용도 |
|-------------|---------------|------|
| 라이브러리 사용법, 문서 확인 | context7 MCP: `mcp__context7__resolve-library-id` → `mcp__context7__query-docs` | 최신 문서 조회 |
| 조사, 리서치, 분석 | `Skill: skill="everything-claude-code:deep-research"` | 웹 리서치 |

#### 코드 품질/리팩토링

| 요청 키워드 | 선행 호출 스킬 | 용도 |
|-------------|---------------|------|
| 리팩토링, 단순화, 정리 | `Skill: skill="ccpp:simplify"` | 코드 단순화 |
| 기술부채, 정리, 클린업 | `Skill: skill="ccpp:techdebt"` 또는 `Skill: skill="everything-claude-code:prune"` | 기술 부채 정리 |

#### 모바일

| 요청 키워드 | 선행 호출 스킬 | 용도 |
|-------------|---------------|------|
| Android, 안드로이드 | `Skill: skill="everything-claude-code:android-clean-architecture"` | Android 아키텍처 |
| SwiftUI, iOS | `Skill: skill="everything-claude-code:swiftui-patterns"` | SwiftUI 패턴 |
| Flutter, Dart | `Skill: skill="everything-claude-code:flutter-dart-code-review"` | Flutter 리뷰 |

#### 미디어/콘텐츠

| 요청 키워드 | 선행 호출 스킬 | 용도 |
|-------------|---------------|------|
| 이미지 생성, 썸네일, 아이콘 | `Skill: skill="ccpp:nano-banana"` | 이미지 생성 |
| 프레젠테이션, PPT, 슬라이드 | `Skill: skill="everything-claude-code:frontend-slides"` | HTML 슬라이드 |
| 글쓰기, 블로그, 아티클 | `Skill: skill="everything-claude-code:article-writing"` | 글 작성 |

> **복수 매칭 시**: 해당하는 스킬을 모두 호출한다. 예: "React 로그인 페이지" → `react-patterns` + `security-review` + `frontend-design`

**금지: 매칭되는 키워드가 있는데 스킬을 호출하지 않는 행위**

### STEP 3: 계획 (3파일 이상 수정 시)

수정 대상 파일이 3개 이상이면 반드시:
```
Skill 도구 호출: skill="ccpp:plan"
```
1~2파일이면 직접 진행.

### STEP 4: 구현 — 파일 기반 스킬 자동 호출

수정하는 파일의 확장자와 디렉토리를 확인하고, 아래 조건에 해당하면 반드시 해당 스킬을 먼저 호출한다:

#### 파일 확장자 기반 (MUST)

| 수정 파일 확장자 | 필수 스킬 호출 |
|-----------------|---------------|
| `.tsx`, `.jsx`, `.vue`, `.svelte` | `Skill: skill="frontend-design:frontend-design"` |
| `.css`, `.scss`, `.tailwind` | `Skill: skill="ccpp:tailwind-design-system"` |
| `.test.ts`, `.test.js`, `.spec.*`, `*_test.*` | `Skill: skill="ccpp:tdd"` |

#### 디렉토리 기반 (MUST)

| 수정 파일 디렉토리 | 필수 스킬 호출 |
|-------------------|---------------|
| `auth/`, `login/`, `session/`, `token/` | `Agent: subagent_type="security-reviewer"` |
| `api/`, `routes/`, `controllers/`, `endpoints/` | `Agent: subagent_type="security-reviewer"` |
| `components/`, `pages/`, `views/`, `ui/` | `Skill: skill="frontend-design:frontend-design"` |

#### 상황 기반 (MUST)

| 상황 | 필수 액션 |
|------|----------|
| 외부 라이브러리 API 사용법이 불확실할 때 | context7 MCP: `mcp__context7__resolve-library-id` → `mcp__context7__query-docs` |
| 빌드 실패 시 | `Skill: skill="ccpp:build-fix"` |

**금지: 위 조건에 해당하는데 스킬을 호출하지 않는 행위**

### STEP 5: 코드 리뷰 (코드 수정 후, 커밋 전) — MUST

코드를 1줄이라도 수정했으면, 커밋 전에 반드시 아래를 실행한다:

```
Skill 도구 호출: skill="ccpp:review"
```

추가로 `docs/code-review-checklist.md`를 Read하여 과거 교훈이 반복되지 않는지 확인한다.

리뷰에서 CRITICAL/HIGH 이슈 → 즉시 수정.
리뷰에서 MEDIUM 이슈 → 가능하면 수정.

**금지: 리뷰 없이 커밋하는 행위**

### STEP 6: 빌드 확인

```bash
{프로젝트 빌드 명령}
```

빌드 실패 시: `Skill: skill="ccpp:build-fix"` 호출

### STEP 7: 문서 기록 (변경이력 필수 작성)

코드를 수정했으면 반드시 아래를 모두 수행한다:

#### 6-1. 기능 문서 업데이트

해당 기능의 `docs/features/{기능명}.md`를 열고:
- `last_modified` 날짜를 오늘로 변경
- `files` 목록에 수정된 파일 추가 (없으면)
- `변경 이력` 테이블에 아래 형식으로 행 추가:

```
| {오늘 날짜} | {변경 요약 1줄} | {수정된 파일 쉼표 구분} | {왜 변경했는지 1줄} |
```

#### 6-2. INDEX 업데이트

새 문서를 추가했으면 `docs/00-INDEX.md` 매핑 테이블에 행을 추가한다.

#### 6-3. CLAUDE.md 업데이트

기능 목록/기술 스택/스크립트가 변경되었으면 이 CLAUDE.md의 해당 섹션을 업데이트한다.

#### 6-4. 교훈 기록

리뷰에서 발견된 패턴이나 버그 원인이 있으면 `docs/code-review-checklist.md`의 교훈 테이블에 추가한다.

**금지: 코드만 수정하고 문서를 업데이트하지 않는 행위**

### STEP 8: 커밋

{사용자 커밋 방식}

---

## 절대 규칙: 기존 기능 보존

- 사용자가 명시적으로 요청하지 않는 한, 기존 기능/로직/UI를 제거하지 않는다
- 수정 전에 영향받는 기능 목록을 먼저 알려주고 확인받는다
- 새 기능 추가 시에도 기존 기능이 깨지지 않는지 확인한다

## 주요 기능 목록 (삭제 금지)

{사용자 인터뷰 Q1에서 획득한 기능 목록}

## 프로젝트 구조

{자동 감지된 디렉토리 트리}

## 스크립트

```bash
{package.json scripts 또는 Makefile 등에서 추출}
```

## 기술 스택

| 분류 | 기술 | 버전 |
|------|------|------|
| {자동 감지된 dependencies 기반} | | |

## 배포 환경

{자동 감지 결과}

## 보안 규칙

- 세션/토큰은 httpOnly 쿠키 또는 안전한 저장소로만 관리
- 모든 사용자 데이터 API에 소유권/권한 확인 필수
- 비밀번호 해시, API 키 응답에 포함 금지
- console.log로 민감 정보 출력 금지

## Gotchas (과거 교훈)

{Q3 답변 + 향후 누적}
```

---

### 생성 파일 2: .claude/hooks/workflow-guard.cjs (강제 게이트 스크립트)

> **왜 echo가 아니라 스크립트인가:**
> Claude Code의 PreToolUse/PostToolUse 훅은 exit 0으로 끝나면 stdout이
> **모델 컨텍스트에 주입되지 않는다**(트랜스크립트에만 남음). 그래서 기존
> `echo [HARNESS...]` 훅은 모델이 읽지 못해 사실상 무력했다.
> 실제로 강제하려면:
> - 컨텍스트 주입: stdout에 `{"hookSpecificOutput":{"hookEventName":...,"additionalContext":...}}` JSON 출력
> - 차단: exit code 2 + stderr (모델이 읽고 멈춤)
>
> 이 스크립트는 코드 수정 시 STEP 리마인더를 컨텍스트에 주입하고,
> 검증(리뷰+빌드+문서)이 끝나지 않은 상태의 `git commit`을 exit 2로 차단한다.

> 프로젝트 루트의 `.claude/hooks/workflow-guard.cjs`로 생성한다.
> `isCodeFile`의 소스 디렉토리/확장자는 프로젝트 언어에 맞게 조정한다.

```javascript
#!/usr/bin/env node
/**
 * 하네스 워크플로우 가드 (CLAUDE.md MANDATORY WORKFLOW 강제)
 *
 * echo 훅은 PreToolUse/PostToolUse exit-0 stdout이 모델 컨텍스트로
 * 주입되지 않아 무력했음. 이 스크립트는:
 *  - pre-edit  : 코드 수정 전 STEP1-2 리마인더를 additionalContext로 주입(비차단)
 *  - post-edit : 코드 수정 후 STEP5-7 리마인더 주입 + 검증 플래그 무효화
 *  - pre-bash  : git commit 시 코드 수정됐는데 검증 미완료면 exit 2로 차단
 *  - verify    : 리뷰/빌드/문서 완료를 수동 선언(검증 플래그 생성)
 *  - reset     : 커밋 성공 후 플래그 정리
 */
const fs = require('fs');
const path = require('path');

const mode = process.argv[2] || '';
const PROJECT = path.resolve(__dirname, '..', '..');
const STATE_DIR = path.join(PROJECT, '.claude', '.workflow-state');
const EDIT_FLAG = path.join(STATE_DIR, 'code-edited.flag');
const VERIFIED = path.join(STATE_DIR, 'verified.flag');

function ensureDir() { try { fs.mkdirSync(STATE_DIR, { recursive: true }); } catch (e) {} }
function readStdin() { try { return fs.readFileSync(0, 'utf8'); } catch (e) { return ''; } }
function parse(s) { try { return JSON.parse(s || '{}'); } catch (e) { return {}; } }

// 프로젝트 언어/구조에 맞게 조정: 소스 디렉토리 + 코드 확장자만 코드 파일로 간주
// (docs/*.md, 설정 파일은 제외하여 게이트 오발동 방지)
function isCodeFile(p) {
  if (!p) return false;
  const n = p.replace(/\\/g, '/');
  const inSrc = /\/(src|server|client|app|lib|pkg|internal|components|pages|services|routes|api|shared)\//.test(n);
  const codeExt = /\.(ts|tsx|js|jsx|cjs|mjs|py|go|rs|java|kt|rb|php|c|cc|cpp|h|hpp|swift|vue|svelte)$/.test(n);
  return inSrc && codeExt;
}

function inject(event, msg) {
  process.stdout.write(JSON.stringify({
    hookSpecificOutput: { hookEventName: event, additionalContext: msg }
  }));
}

const data = parse(readStdin());
const ti = data.tool_input || {};

if (mode === 'pre-edit') {
  const fp = ti.file_path || ti.path || '';
  if (isCodeFile(fp)) {
    ensureDir();
    fs.writeFileSync(EDIT_FLAG, fp + '\n', { flag: 'a' });
    inject('PreToolUse',
      '[HARNESS STEP1-2] 코드 수정 감지. 확인: (1) docs/00-INDEX.md + 관련 features md를 Read했는가? ' +
      '(2) 요청 키워드에 맞는 스킬을 선행 호출했는가? 안 했으면 지금 먼저 수행할 것.');
  }
  process.exit(0);
}

if (mode === 'post-edit') {
  const fp = ti.file_path || ti.path || '';
  if (isCodeFile(fp)) {
    try { fs.unlinkSync(VERIFIED); } catch (e) {}
    inject('PostToolUse',
      '[HARNESS STEP5-7] 코드 수정됨. 커밋 전 필수: (1) Skill ccpp:review (2) 빌드 명령 통과 ' +
      '(3) docs/features md 변경이력 기록. 완료 후 `node .claude/hooks/workflow-guard.cjs verify` 실행해야 커밋 가능.');
  }
  process.exit(0);
}

if (mode === 'pre-bash') {
  const cmd = ti.command || '';
  if (/\bgit\s+commit\b/.test(cmd)) {
    const edited = fs.existsSync(EDIT_FLAG);
    const verified = fs.existsSync(VERIFIED);
    if (edited && !verified) {
      let files = '';
      try { files = fs.readFileSync(EDIT_FLAG, 'utf8').trim(); } catch (e) {}
      process.stderr.write(
        '[HARNESS GATE] 커밋 차단: 이번 세션 코드 수정 후 검증이 완료되지 않음.\n' +
        '수정된 코드 파일:\n' + files + '\n\n' +
        '커밋 전 반드시 완료:\n' +
        '  1) Skill ccpp:review 리뷰 (CRITICAL/HIGH 수정)\n' +
        '  2) 빌드 명령 통과\n' +
        '  3) docs/features md 변경이력 기록\n\n' +
        '위 3가지를 실제로 끝냈으면 다음을 실행한 뒤 다시 커밋:\n' +
        '  node .claude/hooks/workflow-guard.cjs verify');
      process.exit(2);
    }
  }
  process.exit(0);
}

if (mode === 'post-bash') {
  const cmd = ti.command || '';
  if (/\bgit\s+commit\b/.test(cmd)) {
    try { fs.unlinkSync(EDIT_FLAG); } catch (e) {}
    try { fs.unlinkSync(VERIFIED); } catch (e) {}
  }
  process.exit(0);
}

if (mode === 'verify') {
  ensureDir();
  fs.writeFileSync(VERIFIED, new Date().toISOString());
  process.stdout.write('[HARNESS] 검증 완료 표시됨. 이제 git commit 가능.');
  process.exit(0);
}

if (mode === 'reset') {
  try { fs.unlinkSync(EDIT_FLAG); } catch (e) {}
  try { fs.unlinkSync(VERIFIED); } catch (e) {}
  process.stdout.write('[HARNESS] 워크플로우 상태 초기화됨.');
  process.exit(0);
}

process.exit(0);
```

> `.claude/.workflow-state/`는 런타임 플래그 디렉토리다. `.gitignore`에 추가 권장.

---

### 생성 파일 3: .claude/settings.json (프로젝트 hooks)

> 프로젝트 루트의 `.claude/settings.json`에 hooks를 설치한다.
> 이미 존재하면 hooks 키만 추가/병합하되, **기존 echo 기반 Pre/PostToolUse 훅은
> 무력하므로 아래 script 기반 훅으로 교체**한다.
> UserPromptSubmit echo는 exit-0 stdout이 주입되므로 그대로 유효하다.

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "type": "command",
        "command": "echo [HARNESS] 필수 8단계: 1)docs Read 2)요청 키워드→스킬 선행 호출 3)계획 4)구현(파일 기반 스킬) 5)ccpp:review 필수 6)빌드 확인 7)기능 md 변경이력 기록 8)커밋. 건너뛰기 금지."
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Edit|Write|MultiEdit",
        "hooks": [
          { "type": "command", "command": "node \".claude/hooks/workflow-guard.cjs\" pre-edit" }
        ]
      },
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "node \".claude/hooks/workflow-guard.cjs\" pre-bash" }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write|MultiEdit",
        "hooks": [
          { "type": "command", "command": "node \".claude/hooks/workflow-guard.cjs\" post-edit" }
        ]
      },
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "node \".claude/hooks/workflow-guard.cjs\" post-bash" }
        ]
      }
    ]
  }
}
```

**동작 요약:**
- 코드 파일(`isCodeFile` 매칭) 수정 시 → STEP 리마인더가 모델 컨텍스트에 주입
- 코드 수정 후 검증 안 한 채 `git commit` 시도 → **exit 2로 차단**
- `node .claude/hooks/workflow-guard.cjs verify`로 리뷰+빌드+문서 완료 선언 후에만 커밋 가능
- 커밋 성공(post-bash) 시 플래그 자동 정리

---

### 생성 파일 4: docs/00-INDEX.md

```markdown
---
type: index
last_modified: {오늘 날짜}
---

# {프로젝트명} 문서 인덱스

> CLAUDE.md STEP 1에 의해 이 파일 읽기는 필수입니다.
> 코드를 수정하기 전에 이 매핑에서 관련 문서를 찾아 반드시 Read하세요.

## 요청 → 문서 매핑

| 키워드 | 문서 경로 | 설명 |
|--------|-----------|------|
| {기능1 키워드} | docs/features/{기능1}.md | {기능1 설명} |
| {기능2 키워드} | docs/features/{기능2}.md | {기능2 설명} |
| {기능N 키워드} | docs/features/{기능N}.md | {기능N 설명} |
| 리뷰, 체크리스트 | docs/code-review-checklist.md | 코드 리뷰 피드백 누적 |

## 카테고리 구조

```
docs/
├── 00-INDEX.md                 ← 이 파일
├── code-review-checklist.md    ← 리뷰 교훈 누적
└── features/                   ← 기능별 사양 + 변경이력
    ├── {기능1}.md
    ├── {기능2}.md
    └── {기능N}.md
```

## 새 문서 추가 규칙

1. `docs/features/{기능명}.md`를 기능 md 템플릿으로 생성
2. 이 INDEX의 매핑 테이블에 행 추가
3. CLAUDE.md의 기능 목록에도 반영
```

---

### 생성 파일 5: docs/features/{기능명}.md (기능별 — Q1에서 획득한 기능마다 1개씩)

> Phase 1에서 분석한 코드를 기반으로 각 기능의 초기 문서를 생성한다.
> 빈 파일 금지 — 최소한 핵심 파일 목록과 1줄 설명은 채운다.

```markdown
---
feature: {기능명}
status: active
last_modified: {오늘 날짜}
files:
  - {이 기능의 핵심 소스 파일 경로 1}
  - {이 기능의 핵심 소스 파일 경로 2}
tags: [{관련 키워드 쉼표 구분}]
---

# {기능명}

## 설명

{Phase 1에서 코드를 읽고 파악한 기능의 1~3줄 설명}

## 핵심 로직

- {핵심 함수/클래스와 역할 1줄씩}

## 주의사항

- {보존해야 할 동작이나 엣지 케이스}

## 변경 이력

| 날짜 | 변경 내용 | 수정 파일 | 이유 |
|------|-----------|-----------|------|
| {오늘} | 초기 문서화 | - | harness-init |
```

> **중요**: 기능 md는 코드를 실제로 읽고 파악한 내용으로 채운다.
> 추측이나 플레이스홀더로 채우지 않는다.
> Phase 1에서 수집한 소스 파일을 Read하여 핵심 로직을 정확히 기술한다.

---

### 생성 파일 6: docs/code-review-checklist.md

```markdown
---
type: checklist
last_modified: {오늘 날짜}
---

# 코드 리뷰 체크리스트

> CLAUDE.md STEP 5에서 ccpp:review 호출 시 이 파일도 함께 Read한다.

## 필수 확인 항목

- [ ] 기존 기능이 깨지지 않았는가 (CLAUDE.md "절대 규칙")
- [ ] 타입 에러 없는가
- [ ] 보안 규칙 위반 없는가 (API 키 노출, 권한 체크 누락)
- [ ] 불필요한 console.log/print 없는가
- [ ] 에러 핸들링이 적절한가
- [ ] 아래 "과거 교훈" 패턴이 반복되지 않는가

## 과거 교훈

| 날짜 | 이슈 | 원인 | 교훈 |
|------|------|------|------|
| {오늘} | 프로젝트 초기 세팅 | - | 하네스 init 완료 |

> 리뷰에서 발견한 반복 패턴이나 버그 원인을 여기에 추가한다.
> CLAUDE.md STEP 7-4 참고.
```

---

## Phase 4: 검증

생성 후 반드시 아래를 확인한다:

1. CLAUDE.md 존재 + "MANDATORY WORKFLOW" 섹션 포함 확인
2. `.claude/hooks/workflow-guard.cjs` 존재 확인 + 동작 테스트:
   - `echo '{"tool_input":{"file_path":"src/x.ts"}}' | node .claude/hooks/workflow-guard.cjs pre-edit` → additionalContext JSON 출력되는지
   - `echo '{"tool_input":{"command":"git commit -m x"}}' | node .claude/hooks/workflow-guard.cjs pre-bash` → (플래그 있을 때) exit 2 차단되는지
   - 테스트 후 `node .claude/hooks/workflow-guard.cjs reset`으로 잔여 플래그 정리
3. `.claude/settings.json` 존재 + hooks(UserPromptSubmit echo + Pre/PostToolUse script 기반) 확인
4. `.gitignore`에 `.claude/.workflow-state/` 추가 확인
5. `docs/00-INDEX.md` 존재 + 매핑 테이블이 비어있지 않은지 확인
6. `docs/features/*.md`가 Q1에서 받은 기능 수만큼 존재하는지 확인
7. 각 기능 md의 `files:` 필드가 실제 존재하는 파일을 가리키는지 확인
8. 빌드 명령 1회 실행하여 동작 확인

---

## Phase 5: 완료 메시지

```
하네스 v3 세팅 완료!

생성된 파일:
- CLAUDE.md — 강제 8단계 워크플로우 + 요청 키워드 스킬 라우팅 + 파일확장자 라우팅
- .claude/hooks/workflow-guard.cjs — 강제 게이트 스크립트 (컨텍스트 주입 + 커밋 차단)
- .claude/settings.json — 프로젝트 hooks (script 기반)
- docs/00-INDEX.md — 기능→문서 매핑 (초기 데이터 포함)
- docs/features/{기능별}.md — 구조화된 기능 문서 (frontmatter + 변경이력)
- docs/code-review-checklist.md — 리뷰 교훈 누적

강제되는 동작:
1. 코드 수정 전 → STEP1-2 리마인더가 모델 컨텍스트에 주입 (docs Read + 스킬 선행 호출 확인)
2. 요청 키워드 분석 → 해당 스킬 선행 호출 (React→react-patterns, 인증→security-review 등)
3. 파일 확장자/디렉토리 → 구현 시 스킬 자동 호출 (.tsx→frontend, auth/→security)
4. 코드 수정 후 → STEP5-7 리마인더 주입 (ccpp:review + 빌드 + 문서)
5. 검증 안 하고 git commit 시도 → **exit 2로 차단** (verify 선언 전까지 커밋 불가)
6. 빌드 실패 → ccpp:build-fix 자동 호출

> echo 훅과 달리 이 게이트는 모델 컨텍스트에 실제로 주입/차단되어 무시할 수 없다.
```

---

## 주의사항

- 이미 CLAUDE.md가 있으면 덮어쓰지 않고 "MANDATORY WORKFLOW" 섹션만 추가
- 이미 `.claude/settings.json`이 있으면 기존 내용 보존하되, **무력한 echo 기반 Pre/PostToolUse 훅은 script 기반으로 교체**(echo는 exit-0 stdout이 모델 컨텍스트에 주입되지 않아 무효)
- `.claude/hooks/workflow-guard.cjs`의 `isCodeFile` 소스 디렉토리/확장자는 프로젝트 언어에 맞게 조정
- `.gitignore`에 `.claude/.workflow-state/` 추가 (런타임 플래그 디렉토리)
- Node가 없는 프로젝트면 동일 로직을 해당 런타임(python 등)으로 포팅하거나, 최소한 UserPromptSubmit echo는 유지
- docs/ 기존 파일은 보존하고 새 파일만 추가
- 기능 md는 코드를 실제로 Read해서 내용을 채운다 (빈 템플릿 금지)
- 프로젝트 언어에 따라 빌드 명령, 파일 확장자 라우팅을 자동 조정

---

## 사용법

```
/harness-init
```
