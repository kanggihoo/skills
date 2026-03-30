# Spring Boot 프로젝트 분석 가이드

## 엔트리포인트 찾기

```bash
# Application 클래스 찾기
find . -name "*Application.java" | grep -v test

# 의존성 확인
cat pom.xml | grep -E "<artifactId>" | head -30
# 또는
cat build.gradle | grep -E "implementation|api" | head -30
```

## 표준 레이어 구조

```
src/main/java/com/example/
├── ExampleApplication.java     ← @SpringBootApplication, 엔트리포인트
├── controller/                 ← @RestController, HTTP 요청 처리
│   └── UserController.java
├── service/                    ← @Service, 비즈니스 로직
│   ├── UserService.java        (인터페이스)
│   └── UserServiceImpl.java    (구현체)
├── repository/                 ← @Repository, DB 접근 (JPA)
│   └── UserRepository.java
├── domain/ (또는 entity/)      ← @Entity, JPA 엔티티
│   └── User.java
├── dto/                        ← 데이터 전송 객체
│   ├── request/
│   └── response/
├── config/                     ← @Configuration, 빈 설정
│   └── SecurityConfig.java
└── exception/                  ← 예외 처리
    └── GlobalExceptionHandler.java
```

## API 엔드포인트 추적 패턴

```bash
# 모든 매핑 찾기
grep -rn "@GetMapping\|@PostMapping\|@RequestMapping" src/main/java/ | grep -v test

# 특정 URL 패턴 추적
grep -rn '"/api/users"' src/main/java/

# 서비스 계층 메서드 찾기
grep -rn "UserService" src/main/java/
```

## 핵심 어노테이션 의미

| 어노테이션 | 위치 | 역할 |
|---|---|---|
| `@RestController` | Controller | HTTP 요청 처리 |
| `@Service` | Service | 비즈니스 로직 |
| `@Repository` | Repository | 데이터 접근 |
| `@Entity` | Domain | JPA 엔티티 |
| `@Configuration` | Config | 빈 설정 |
| `@Component` | 어디서나 | 일반 컴포넌트 |

## HTTP 요청 처리 흐름

```
HTTP 요청
    ↓
@RestController
    ↓
@Service (비즈니스 로직)
    ↓
@Repository (JPA)
    ↓
DB
```

## 테스트 패턴

```bash
# 전체 테스트 실행
./gradlew test

# 특정 테스트 실행
./gradlew test --tests "com.example.service.UserServiceTest"

# 테스트 레이어별 확인
# @SpringBootTest → 통합 테스트
# @WebMvcTest → Controller 단위 테스트
# @DataJpaTest → Repository 단위 테스트
```

## 설정 파일 확인

```bash
# 환경별 설정
cat src/main/resources/application.yml
cat src/main/resources/application-dev.yml

# 활성 프로파일 확인
grep -n "spring.profiles" src/main/resources/application.yml
```
