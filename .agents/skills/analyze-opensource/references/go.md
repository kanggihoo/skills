# Go 프로젝트 분석 가이드

## 엔트리포인트 찾기

```bash
# main 패키지 찾기
find . -name "main.go" | grep -v vendor

# 일반적인 위치
cmd/<appname>/main.go   ← 권장 구조 (Clean Architecture)
main.go                 ← 단순 프로젝트

# 모듈 정보 확인
cat go.mod
```

## 표준 프로젝트 레이아웃

```
cmd/
└── <appname>/
    └── main.go         ← 진입점, 의존성 주입
internal/               ← 외부 패키지에서 import 불가
├── handler/            ← HTTP 핸들러
├── service/            ← 비즈니스 로직
├── repository/         ← DB 접근
└── domain/             ← 도메인 모델
pkg/                    ← 외부 패키지에서 import 가능한 공개 라이브러리
api/                    ← API 정의 (OpenAPI 등)
```

## 핵심 추적 패턴

```bash
# HTTP 라우터 등록 찾기 (chi, gin, echo 패턴)
grep -rn "router.Get\|r.GET\|e.GET\|mux.Handle" .

# 인터페이스 정의 찾기
grep -rn "type.*interface" internal/ | head -20

# 구현체 찾기
grep -rn "func.*Service\|func.*Handler\|func.*Repo" internal/

# 특정 함수 호출 추적
grep -rn "\.CreateUser\|\.FindUser" internal/
```

## Go 특유의 패턴

```go
// 인터페이스 기반 설계
type UserRepository interface {
    Create(ctx context.Context, user *User) error
    FindByID(ctx context.Context, id int64) (*User, error)
}

// 의존성 주입 (cmd/main.go에서)
repo := postgres.NewUserRepository(db)
svc := service.NewUserService(repo)
handler := handler.NewUserHandler(svc)
```

→ `main.go`를 보면 전체 의존성 그래프를 파악 가능.

## 테스트 패턴

```bash
# 전체 테스트 실행
go test ./...

# 특정 패키지 테스트
go test ./internal/service/...

# 특정 테스트 함수만
go test -run TestCreateUser ./internal/service/

# 커버리지
go test -cover ./...
```

## 의존성 확인

```bash
# 외부 라이브러리 목록
cat go.mod | grep -v "^//"

# 의존성 트리
go mod graph | head -20
```
