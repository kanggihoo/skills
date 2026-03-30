# Node.js / TypeScript 프로젝트 심층 분석 가이드

> **원칙**: 아래의 "일반적인 경로"는 참고용이다. 반드시 실제 탐색으로 확인한 뒤 적용한다.

---

## Step 1: 프로젝트 유형 판별 (탐색 먼저)

```bash
# 1. 루트 파일 목록 확인
ls -la

# 2. package.json 핵심 필드 확인
cat package.json | grep -E '"name"|"type"|"main"|"module"|"bin"|"scripts"' -A 2

# 3. 구성 파일로 프레임워크 감지
ls tsconfig*.json next.config.* vite.config.* vitest.config.* \
   jest.config.* .eslintrc* prettier.config.* 2>/dev/null

# 4. 의존성으로 프레임워크 감지
cat package.json | grep -E '"next"|"react"|"vue"|"express"|"fastify"|"hono"|"nestjs"|"elysia"' -i
```

---

## Step 2: 실제 소스 구조 탐색

```bash
# node_modules, dist, .git 제외하고 전체 파일 트리 확인
find . -maxdepth 3 \
  -not -path '*/node_modules/*' \
  -not -path '*/.git/*' \
  -not -path '*/dist/*' \
  -not -path '*/build/*' \
  -not -path '*/out/*' \
  -not -path '*/.next/*' \
  | sort

# TypeScript 소스 파일만 목록
find . -name "*.ts" -o -name "*.tsx" \
  | grep -v node_modules | grep -v dist | grep -v ".d.ts" | sort

# 파일 수로 규모 파악
find . -name "*.ts" -not -path '*/node_modules/*' | wc -l
```

---

## Step 3: 유형별 분석 방법

### 🔧 CLI 도구

**판별**: `package.json`에 `"bin"` 필드 존재

```bash
# 엔트리포인트 확인
cat package.json | grep -A 5 '"bin"'

# 명령어 라우팅 패턴 찾기 (commander.js / yargs / meow 등)
grep -rn "\.command\|\.action\|program\." <엔트리파일> | head -20

# 각 명령어 핸들러 위치 확인
grep -rn "import.*from\|require(" <엔트리파일> | head -20
```

**탐색 포인트**:
- `bin` → CLI 진입 파일
- 명령어 파싱 파일 (commander/yargs 초기화)
- 각 명령어 구현 파일 (실제 경로는 import 추적으로 확인)

---

### 🌐 Express / Fastify / Hono API 서버

**판별**: `package.json` dependencies에 `express`, `fastify`, `hono` 등 존재

```bash
# 앱 초기화 파일 찾기
grep -rn "express()\|fastify()\|new Hono\|app\.listen" src/ --include="*.ts" -l

# 라우터 등록 패턴 찾기
grep -rn "\.use\|\.get\|\.post\|\.put\|\.delete\|router\." src/ --include="*.ts" | head -30

# 미들웨어 체인 확인
grep -rn "app\.use(" src/ --include="*.ts"
```

**추적 순서**: 앱 초기화 → 라우터 등록 → 컨트롤러 → 서비스 → DB 레이어

---

### ⚛️ Next.js 앱

**판별**: `package.json`에 `"next"` 의존성 존재

```bash
# App Router vs Pages Router 판별
ls app/ 2>/dev/null && echo "App Router (Next 13+)" || ls pages/ 2>/dev/null && echo "Pages Router"

# 레이아웃 구조 확인
find app/ -name "layout.tsx" -o -name "layout.ts" 2>/dev/null | sort
find pages/ -name "_app.tsx" -o -name "_app.js" 2>/dev/null

# API 라우트 위치
find app/ -name "route.ts" 2>/dev/null | sort       # App Router API
find pages/api/ -type f 2>/dev/null                  # Pages Router API

# 서버 액션 확인
grep -rn '"use server"\|"use client"' app/ --include="*.tsx" --include="*.ts" -l
```

**추적 순서**: `layout.tsx` → `page.tsx` → 서버 컴포넌트 → API route / Server Action → 데이터 레이어

---

### 📦 라이브러리 / npm 패키지

**판별**: `"main"`, `"module"`, `"exports"` 필드 존재 + `"bin"` 없음

```bash
# 공개 API 진입점 확인
cat package.json | grep -E '"main"|"module"|"exports"' -A 5

# 공개 export 목록
cat <main 필드가 가리키는 파일> | grep "^export"

# 타입 정의 확인
cat package.json | grep '"types"\|"typings"'
```

---

### 🏗️ 모노레포 (Turborepo / Nx / pnpm workspaces)

**판별**: 루트에 `packages/`, `apps/`, `libs/` 디렉토리 존재 또는 `pnpm-workspace.yaml`

```bash
# 워크스페이스 구조 확인
cat pnpm-workspace.yaml 2>/dev/null || cat package.json | grep '"workspaces"' -A 5

# 각 패키지 목록 및 역할
ls packages/ 2>/dev/null; ls apps/ 2>/dev/null

# 각 패키지의 목적 파악
for pkg in packages/*/; do echo "=== $pkg ==="; cat "$pkg/package.json" | grep '"name"\|"description"'; done
```

---

## Step 4: 핵심 로직 추적 패턴

```bash
# 특정 함수 정의 위치
grep -rn "export.*function <이름>\|export const <이름>\|export default" \
  src/ --include="*.ts" --include="*.tsx"

# 특정 심볼의 사용처 (모든 호출 위치)
grep -rn "<함수명>(" src/ --include="*.ts" --include="*.tsx"

# import 그래프 한 단계 추적 (A가 무엇을 가져오는지)
grep -n "^import\|^export" <파일경로> | head -20

# 타입/인터페이스 정의 위치
grep -rn "^export interface\|^export type\|^interface\|^type " \
  src/ --include="*.ts" | grep "<타입명>"
```

---

## Step 5: 의존성 및 설정 파악

```bash
# 런타임 의존성과 개발 의존성 분리 확인
cat package.json | grep -A 30 '"dependencies"'
cat package.json | grep -A 30 '"devDependencies"'

# TypeScript 설정 확인 (경로 별칭, 빌드 타겟)
cat tsconfig.json | grep -E '"paths"|"baseUrl"|"target"|"module"|"outDir"'

# 린터/포맷터 설정
cat .eslintrc.json 2>/dev/null || cat eslint.config.* 2>/dev/null | head -20
cat prettier.config.* 2>/dev/null || cat .prettierrc* 2>/dev/null | head -10
```

---

## Step 6: 테스트 구조 파악

```bash
# 테스트 러너 파악
cat package.json | grep -E '"jest"|"vitest"|"mocha"|"playwright"|"cypress"'

# 테스트 파일 위치
find . -name "*.test.ts" -o -name "*.spec.ts" -o -name "*.test.tsx" \
  | grep -v node_modules | sort

# 특정 모듈 테스트 확인
cat <모듈명>.test.ts | grep "describe\|it(\|test(" | head -20

# Mock 패턴 파악 (의존성 경계 확인)
grep -rn "vi\.mock\|jest\.mock\|mock(" tests/ --include="*.ts" | head -15
```

---

## 빠른 참고: 설정 파일 → 프레임워크 매핑

| 발견된 파일 | 해당 기술 |
| ---------- | --------- |
| `next.config.*` | Next.js |
| `vite.config.*` | Vite (React/Vue/Vanilla) |
| `vitest.config.*` | Vitest 테스트 |
| `jest.config.*` | Jest 테스트 |
| `turbo.json` | Turborepo 모노레포 |
| `nx.json` | Nx 모노레포 |
| `tsup.config.*` | tsup 빌더 (라이브러리) |
| `rollup.config.*` | Rollup 번들러 |
| `drizzle.config.*` | Drizzle ORM |
| `prisma/schema.prisma` | Prisma ORM |
| `wrangler.toml` | Cloudflare Workers |
