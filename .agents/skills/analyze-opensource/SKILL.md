---
name: analyze-opensource
description: Quickly analyze an open-source repository to understand its structure, architecture, and codebase, then plan how to add new features. Use this skill whenever the user wants to understand how an open-source project works, clone and explore a new repository, trace how a feature is implemented, find the right place to add code, or prepare to contribute to or fork an open-source project. Trigger this skill even when the user says things like "I want to add a feature to X", "how does this project work", "I forked this repo and don't know where to start", "analyze this codebase for me", or "어떻게 구현되어 있어", "이 프로젝트 분석해줘", "기능 추가하려는데 어디에 넣어야 해".
---

# Open-Source Analyzer

A skill for rapidly understanding an open-source codebase and planning feature additions on top of it.

**핵심 철학**: 파일을 전부 읽으려 하지 말고, **직접 탐색 → 패턴 발견 → 경로 추적** 순서를 지켜라. 특히 디렉토리 구조는 가정(assumption)하지 말고 **반드시 직접 확인**한다.

---

## Phase 1: Macro-Level Recon (목적과 기술 스택 파악)

> **목표: "이 프로젝트가 무엇을 하는가? 어떤 기술을 쓰는가?"**

### 1-1. 필수 파일 읽기

아래 순서로 읽되, 모두 읽을 필요 없다 — 핵심만 파악한다.

| 파일 | 확인할 것 |
| ---- | --------- |
| `README.md` | 프로젝트 목적, 핵심 기능, 설치 방법 |
| `package.json` / `pom.xml` / `go.mod` / `pyproject.toml` | 기술 스택, 실행 스크립트, 주요 의존성 |
| `CONTRIBUTING.md` | 코드 컨벤션, PR 규칙, 브랜치 전략 |
| `CHANGELOG.md` | 최근 변경 이력 → 현재 개발 방향 파악 |

### 1-2. 기술 스택 판별

`package.json` 또는 설정 파일을 통해 아래를 확인한다:

```bash
# JS/TS 프로젝트 판별
cat package.json | grep -E '"type"|"main"|"bin"|"scripts"' -A 3

# TypeScript 사용 여부
ls tsconfig*.json 2>/dev/null && echo "TypeScript" || echo "Plain JS"

# 번들러/런타임 확인
cat package.json | grep -E 'vite|webpack|esbuild|rollup|tsx|ts-node|bun' -i
```

---

## Phase 2: 디렉토리 구조 직접 탐색 (아키텍처 파악)

> **목표: "실제로 어떤 디렉토리와 파일이 존재하는가?"**

> ⚠️ **절대 하지 말 것**: `src/`, `lib/`, `app/` 등의 경로를 가정하고 바로 읽기 시작하는 것. 반드시 아래 **탐색 → 분류 → 추론** 순서를 따른다.

### 2-1. 루트 → 핵심 디렉토리 순서로 탐색

```bash
# Step 1: 루트 레벨 디렉토리/파일 확인
ls -la

# Step 2: 상위 디렉토리 구조 확인 (node_modules 제외)
find . -maxdepth 2 -not -path '*/node_modules/*' -not -path '*/.git/*' | sort

# Step 3: 소스 파일이 어디 있는지 확인
find . -maxdepth 3 -name "*.ts" -o -name "*.tsx" -o -name "*.js" -o -name "*.mjs" \
  | grep -v node_modules | grep -v dist | grep -v ".d.ts" | head -40
```

### 2-2. 발견된 구조 기반으로 역할 분류

탐색으로 발견한 실제 디렉토리를 아래 기준으로 분류한다:

| 디렉토리 패턴 | 일반적 역할 | 확인 방법 |
| ------------ | ---------- | --------- |
| `src/`, `lib/`, `packages/` | 핵심 소스 코드 | `ls` 후 파일 목록 확인 |
| `tests/`, `__tests__/`, `spec/` | 테스트 코드 | 파일명 패턴 `.test.`, `.spec.` |
| `scripts/`, `tools/` | 빌드/자동화 | `ls` 후 파일 이름 확인 |
| `examples/`, `sample/` | 사용 예시 | 내용 훑기 |
| `dist/`, `build/`, `out/` | 빌드 산출물 | 분석 불필요 (skip) |
| `docs/`, `.agents/` | 문서/설정 | 필요시만 확인 |

> **중요**: 위 표는 "가능성"이지 "확정"이 아니다. 실제 발견한 디렉토리 이름이 다를 수 있다. 직접 본 것만 문서화한다.

### 2-3. 각 핵심 파일의 역할 추론 (처음 20~30줄만)

전체를 읽지 마라. **import/export 패턴만 보면 역할이 보인다.**

```bash
# 특정 파일의 상단만 확인
head -30 <파일경로>

# 모든 핵심 소스 파일의 export 확인
grep -rn "^export " src/ --include="*.ts" | head -30

# 엔트리포인트 후보 찾기 (package.json의 main/bin이 가리키는 파일)
cat package.json | grep -E '"main"|"bin"|"module"'
```

### 2-4. JS/TS 프로젝트 유형 판별 및 구조 매핑

탐색 결과를 바탕으로 아래 유형 중 하나로 분류하고 실제 경로를 매핑한다.

**판별 기준**:

```bash
# CLI 도구 여부
cat package.json | grep '"bin"'

# 프레임워크 감지
cat package.json | grep -E '"next"|"react"|"vue"|"express"|"fastify"|"hono"|"nestjs"'

# 모노레포 여부
ls packages/ 2>/dev/null || ls apps/ 2>/dev/null
```

**유형별 확인 포인트** (직접 탐색 후 실제 경로로 채울 것):

- **CLI 도구**: `bin` 필드가 가리키는 파일 → 명령어 라우팅 패턴 찾기
- **Express/Fastify API**: 미들웨어 등록 파일 → 라우터 → 컨트롤러 순서
- **Next.js**: `app/` or `pages/` 중 실제 존재하는 것 → 레이아웃 → 페이지
- **라이브러리**: `main`/`module` 필드 파일 → 공개 API 확인
- **모노레포**: `packages/*/package.json` 순회 → 각 패키지 역할 파악

상세 가이드: `references/nodejs.md` 참고

---

## Phase 3: 핵심 로직 추적 (Call Flow 분석)

> **목표: "명령어/기능 하나가 실행될 때 어떤 순서로 코드가 흐르는가?"**

이것이 가장 중요한 단계다. **하나의 경로를 끝까지 따라가라.**

### 3-1. 엔트리포인트 → 핵심 로직 추적

```
1. 분석할 기능/명령어 하나 선택
   예: `skills add <package>`, POST /api/users, getServerSideProps

2. 엔트리포인트에서 시작
   → package.json bin 필드 or app.ts or pages/index.tsx

3. 핸들러/라우터 → 서비스/비즈니스 로직 → 데이터 레이어 순서로 따라가기

4. 각 단계의 입력/출력 파악
   install(url, options) → { success: boolean, path: string }
```

### 3-2. JS/TS 코드 추적 도구

```bash
# 함수 정의 위치 찾기
grep -rn "function <이름>\|const <이름>\|export.*<이름>" src/ --include="*.ts"

# 특정 심볼의 모든 사용처
grep -rn "<함수명>(" src/ --include="*.ts" --include="*.tsx"

# import 관계 파악
grep -rn "from.*<모듈명>\|require.*<모듈명>" src/

# 타입 정의 추적
grep -rn "interface\|type " src/ --include="*.ts" | grep "<타입명>"
```

**VSCode 단축키:**
- `F12` — Go to Definition (정의로 이동)
- `Shift+F12` — Find All References
- `Ctrl+Shift+F` — 전역 검색

### 3-3. Call Flow 다이어그램 작성

```
[사용자 입력 / 요청]
     ↓
<엔트리파일>: 진입점 처리
     ↓
<라우터/파서>: 분기 처리
     ↓
<핵심 로직 파일>: 비즈니스 로직
     ↓
<데이터/외부 연동>: 저장/호출
     ↓
<결과 반환>
```

> 이 다이어그램을 작성하면 "내가 어디에 끼어들어야 하는가"가 명확해진다.

---

## Phase 4: 테스트 코드 분석

> **목표: "각 모듈의 공식 스펙을 테스트에서 파악한다."**

테스트 코드는 주석보다 정확한 명세서다.

### 4-1. 테스트 파일 탐색

```bash
# JS/TS 테스트 파일 위치 파악
find . -name "*.test.ts" -o -name "*.spec.ts" -o -name "*.test.tsx" \
  -o -name "*.test.js" | grep -v node_modules | sort

# 테스트 러너 확인
cat package.json | grep -E '"jest"|"vitest"|"mocha"|"playwright"'

# 테스트 실행
npm test / pnpm test / yarn test
```

### 4-2. 테스트 코드에서 읽을 것

- `describe` / `it` 블록 이름 → 기능 목록 파악
- `expect` / `assert` → 입력 대비 기대 출력
- `mock` / `vi.mock` / `jest.mock` → 외부 의존성 경계 파악

---

## Phase 5: 기능 추가 계획

> **목표: "기존 패턴을 따라 최소한의 변경으로 새 기능을 삽입한다."**

### 5-1. 삽입 지점 결정

Phase 3의 Call Flow를 보며 질문한다:

- 새 기능은 **기존 흐름 어디에 끼어드는가?**
- 새 파일이 필요한가, 아니면 기존 파일을 수정하는가?
- 엔트리포인트(cli.ts, main.ts, app.ts)에 등록이 필요한가?

### 5-2. 기존 패턴 복붙 후 수정

> 처음부터 새로 만들지 마라. 가장 유사한 기존 파일을 복사하고 수정하라.

```
1. Phase 3에서 찾은 유사 기능 파일 복사
2. 로직을 새 기능에 맞게 수정
3. 엔트리포인트에 라우팅/등록 추가
4. 테스트 케이스 작성 (기존 테스트 파일 패턴 그대로 따라가기)
```

### 5-3. 구현 체크리스트

```
[ ] Phase 3 Call Flow 다이어그램 완성
[ ] 삽입 지점 파일명 + 함수명 결정 완료
[ ] 기존 유사 파일 복붙 후 수정
[ ] 엔트리포인트에 새 진입점 등록
[ ] 테스트 케이스 작성 (기존 테스트 파일 참고)
[ ] 포맷터 실행 (prettier / eslint --fix)
[ ] 전체 테스트 실행 → 기존 기능 깨지지 않음 확인
[ ] README 또는 CHANGELOG 업데이트 (필요시)
```

---

## 분석 결과 출력 형식

분석이 완료되면 아래 형식으로 요약을 제공한다. **"일반적으로" 또는 "보통" 같은 가정성 표현 없이, 실제 탐색으로 확인한 내용만 작성한다.**

```markdown
## 프로젝트 분석 결과: <project-name>

### 기술 스택
- 언어: TypeScript / JavaScript
- 프레임워크: (직접 확인한 것)
- 테스트: (jest/vitest 등 직접 확인)
- 빌드: (tsup/rollup/esbuild 등 직접 확인)
- 주요 의존성: (package.json에서 직접 확인)

### 실제 디렉토리 구조 (탐색 결과)

```
<직접 탐색한 실제 구조를 여기에>
├── <실제 존재하는 파일/디렉토리>
│   └── <역할: 직접 확인한 내용>
```

### 엔트리포인트

- `<실제 파일명>`: <직접 확인한 역할>

### Call Flow (핵심 기능)

<기능명>: A.ts → B() → C.ts → D()

### 기능 추가 계획

- 삽입 지점: `<실제 파일명>` `<실제 함수명>`
- 필요한 파일: <신규 생성 / 기존 수정>
- 예상 작업 순서: ...

### 주의사항

- <코딩 컨벤션, 포맷터, 특이사항 — 직접 확인한 것>
```

---

## 언어별 레퍼런스

자세한 언어별 가이드는 `references/` 폴더를 참고한다:

- **`references/nodejs.md`** — JS/TypeScript 심층 분석 패턴 (우선 참고)
- `references/spring.md` — Spring Boot 패턴
- `references/python.md` — Python 패턴
- `references/go.md` — Go 패턴

> JS/TS 프로젝트라면 Phase 2 탐색 후 반드시 `references/nodejs.md`를 읽어라.

---

## 핵심 마인드셋

| ❌ 잘못된 접근 | ✅ 올바른 접근 |
| ------------- | ------------- |
| `src/index.ts`가 있겠지 → 바로 읽기 | `ls` + `find`로 실제 구조 먼저 탐색 |
| 파일 전체를 읽고 시작 | 처음 30줄 + import/export만 확인 |
| 새로운 패턴으로 처음부터 구현 | 기존 파일 복붙 후 수정 |
| 기능 완성 후 테스트 추가 | 테스트 읽기 → 구현 → 테스트 통과 |
| 3일 분석 후 시작 | 30분 탐색 + 바로 코드 수정 시도 |
