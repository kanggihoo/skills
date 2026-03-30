# Python 프로젝트 분석 가이드

## 엔트리포인트 찾기

```bash
# 실행 방법 확인
cat pyproject.toml | grep -A 10 "\[project.scripts\]"
cat setup.py | grep "entry_points" -A 5

# 일반적인 위치
python -m <package_name>   # __main__.py 존재 시
src/<package>/__main__.py
src/<package>/cli.py
```

## 프로젝트 유형별 구조

### CLI 도구 (예: black, ruff)
```
src/<package>/
├── __init__.py
├── __main__.py         ← python -m <pkg> 실행 시 진입점
├── cli.py              ← Click/Typer 명령어 정의
├── core.py             ← 핵심 로직
└── utils.py            ← 유틸리티
```

### FastAPI / Flask 앱
```
src/
├── main.py             ← 앱 초기화, 라우터 등록
├── api/
│   └── routes/         ← 엔드포인트 정의
├── services/           ← 비즈니스 로직
├── models/             ← Pydantic / SQLAlchemy 모델
├── repositories/       ← DB 접근
└── core/
    └── config.py       ← 설정값
```

## 핵심 추적 패턴

```bash
# Click 명령어 찾기
grep -rn "@click.command\|@app.command" src/

# FastAPI 라우터 찾기
grep -rn "@router.get\|@router.post\|@app.get" src/

# 클래스 상속 관계 파악
grep -rn "class.*Base\|class.*ABC" src/

# 특정 함수 사용처 찾기
grep -rn "from.*import.*MyClass\|import.*MyClass" src/
```

## 의존성 분석

```bash
# 의존성 확인
cat pyproject.toml
cat requirements.txt

# 설치 및 실행
pip install -e .
# 또는
uv sync && uv run python -m <package>
```

## 테스트 패턴

```bash
# pytest 실행
pytest
pytest tests/test_core.py -v
pytest -k "test_add" -v   # 특정 이름 패턴

# 커버리지
pytest --cov=src/
```

## 데코레이터 패턴 이해

Python에서 데코레이터는 기능 주입의 핵심이다:

```python
# CLI 진입점
@click.command()
@click.argument("url")
def add(url: str):
    ...

# FastAPI 라우터
@router.post("/users")
async def create_user(body: UserCreate):
    ...
```

이런 패턴을 보면 → 해당 함수가 그 명령어/엔드포인트의 핸들러임을 즉시 파악 가능.
