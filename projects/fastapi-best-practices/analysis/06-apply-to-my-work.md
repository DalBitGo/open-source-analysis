# FastAPI Best Practices 실전 적용

> **목표**: 학습한 패턴을 바로 적용 가능한 형태로 정리

---

## 핵심 패턴 요약 (15개)

### 1. 프로젝트 구조
| 패턴 | 핵심 | 적용 시점 |
|------|------|----------|
| **도메인 중심 구조** | `src/{domain}/` 폴더별 캡슐화 | ✅ 새 프로젝트 시작 시 |
| **파일 역할 명확화** | router/service/schemas/models 분리 | ✅ 항상 |
| **명시적 Import** | `from src.auth import constants as auth_constants` | ✅ 항상 |

### 2. Async/Sync
| 패턴 | 핵심 | 적용 시점 |
|------|------|----------|
| **I/O → async** | `async def` + `await` (httpx, asyncpg) | ✅ I/O 작업 시 |
| **Sync SDK → threadpool** | `run_in_threadpool(requests.get, ...)` | ⚠️ Async SDK 없을 때 |
| **CPU → 별도 프로세스** | Celery + Redis | ⚠️ 무거운 계산 시 |

### 3. Dependency Injection
| 패턴 | 핵심 | 적용 시점 |
|------|------|----------|
| **Validation DI** | `Depends(valid_post_id)` | ✅ 검증 로직 재사용 시 |
| **Chain Dependencies** | `Depends(A)` 안에서 `Depends(B)` | ✅ 복잡한 검증 시 |
| **Request Caching** | 같은 Dependency 1번만 실행 | ✅ 자동 |

### 4. Pydantic
| 패턴 | 핵심 | 적용 시점 |
|------|------|----------|
| **과도 사용** | EmailStr, Enum, Field 최대 활용 | ✅ 항상 |
| **Custom BaseModel** | 전역 datetime 형식 통일 | ⚠️ 표준화 필요 시 |
| **BaseSettings 분리** | 도메인별 Config 클래스 | ⚠️ 설정 10개+ |

### 5. Database
| 패턴 | 핵심 | 적용 시점 |
|------|------|----------|
| **SQL-first** | Join/집계는 DB에서 | ✅ 항상 |
| **네이밍 컨벤션** | lower_snake_case, 단수형 | ✅ 항상 |
| **Index 네이밍** | MetaData naming_convention | ✅ 프로젝트 시작 시 |

---

## 새 FastAPI 프로젝트 시작하기

### Phase 1: 프로젝트 구조 (Day 1)

```bash
# 1. 프로젝트 생성
mkdir my-fastapi-project
cd my-fastapi-project

# 2. 가상환경
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# 3. 의존성 설치
pip install fastapi uvicorn[standard] pydantic-settings sqlalchemy alembic

# 4. 디렉토리 구조
mkdir -p src/{auth,posts,config}
mkdir -p tests/{auth,posts}
mkdir -p requirements

# 5. 필수 파일 생성
touch src/main.py
touch src/config.py
touch src/database.py
touch .env
touch .gitignore
```

**src/ 구조**:
```
src/
├── auth/
│   ├── __init__.py
│   ├── router.py
│   ├── schemas.py
│   ├── models.py
│   ├── service.py
│   ├── dependencies.py
│   └── config.py
├── posts/
│   └── (same structure)
├── main.py
├── config.py
└── database.py
```

---

### Phase 2: 기본 설정 (Day 1)

**src/config.py**:
```python
from pydantic import PostgresDsn
from pydantic_settings import BaseSettings


class Config(BaseSettings):
    DATABASE_URL: PostgresDsn
    ENVIRONMENT: str = "development"

    APP_VERSION: str = "1.0.0"

    class Config:
        env_file = ".env"


settings = Config()
```

**.env**:
```bash
DATABASE_URL=postgresql://user:pass@localhost/dbname
ENVIRONMENT=development
```

**src/main.py**:
```python
from fastapi import FastAPI

from src.config import settings

app = FastAPI(
    title="My API",
    version=settings.APP_VERSION,
)


@app.get("/health")
async def health():
    return {"status": "ok"}


if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

**실행**:
```bash
python -m src.main
# → http://localhost:8000/docs
```

---

### Phase 3: 첫 도메인 추가 (Day 2)

**src/posts/schemas.py**:
```python
from pydantic import BaseModel, Field, UUID4


class PostCreate(BaseModel):
    title: str = Field(min_length=1, max_length=200)
    content: str


class PostResponse(BaseModel):
    id: UUID4
    title: str
    content: str
```

**src/posts/router.py**:
```python
from fastapi import APIRouter

from .schemas import PostCreate, PostResponse

router = APIRouter(prefix="/posts", tags=["posts"])


@router.post("/", response_model=PostResponse, status_code=201)
async def create_post(data: PostCreate):
    # TODO: DB 저장
    return {
        "id": "...",
        "title": data.title,
        "content": data.content
    }


@router.get("/{post_id}", response_model=PostResponse)
async def get_post(post_id: str):
    # TODO: DB 조회
    return {"id": post_id, "title": "Test", "content": "..."}
```

**src/main.py에 등록**:
```python
from src.posts.router import router as posts_router

app.include_router(posts_router)
```

---

### Phase 4: Database 연결 (Day 3)

**src/database.py**:
```python
from sqlalchemy import create_engine, MetaData
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

from src.config import settings

POSTGRES_INDEXES_NAMING_CONVENTION = {
    "ix": "%(column_0_label)s_idx",
    "uq": "%(table_name)s_%(column_0_name)s_key",
    "ck": "%(table_name)s_%(constraint_name)s_check",
    "fk": "%(table_name)s_%(column_0_name)s_fkey",
    "pk": "%(table_name)s_pkey",
}

metadata = MetaData(naming_convention=POSTGRES_INDEXES_NAMING_CONVENTION)
Base = declarative_base(metadata=metadata)

engine = create_engine(str(settings.DATABASE_URL))
SessionLocal = sessionmaker(bind=engine)


def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

**src/posts/models.py**:
```python
from sqlalchemy import Column, String, Text
from sqlalchemy.dialects.postgresql import UUID
import uuid

from src.database import Base


class Post(Base):
    __tablename__ = "post"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    title = Column(String(200), nullable=False)
    content = Column(Text, nullable=False)
```

**Alembic 초기화**:
```bash
alembic init alembic

# alembic.ini 수정
# file_template = %%(year)d-%%(month).2d-%%(day).2d_%%(slug)s

# alembic/env.py 수정
from src.database import Base
from src.posts.models import Post  # Import all models

target_metadata = Base.metadata

# 마이그레이션 생성
alembic revision --autogenerate -m "create_post_table"
alembic upgrade head
```

---

### Phase 5: Service Layer (Day 4)

**src/posts/service.py**:
```python
from sqlalchemy.orm import Session
from uuid import UUID

from .models import Post
from .schemas import PostCreate


async def create_post(db: Session, data: PostCreate) -> Post:
    post = Post(**data.model_dump())
    db.add(post)
    db.commit()
    db.refresh(post)
    return post


async def get_by_id(db: Session, post_id: UUID) -> Post | None:
    return db.query(Post).filter(Post.id == post_id).first()
```

**src/posts/router.py 수정**:
```python
from fastapi import APIRouter, Depends
from sqlalchemy.orm import Session

from src.database import get_db
from . import service
from .schemas import PostCreate, PostResponse

router = APIRouter(prefix="/posts", tags=["posts"])


@router.post("/", response_model=PostResponse, status_code=201)
async def create_post(
    data: PostCreate,
    db: Session = Depends(get_db)
):
    post = await service.create_post(db, data)
    return post


@router.get("/{post_id}", response_model=PostResponse)
async def get_post(
    post_id: UUID4,
    db: Session = Depends(get_db)
):
    post = await service.get_by_id(db, post_id)
    if not post:
        raise HTTPException(404, "Post not found")
    return post
```

---

### Phase 6: Dependencies (Day 5)

**src/posts/dependencies.py**:
```python
from fastapi import Depends, HTTPException
from sqlalchemy.orm import Session
from uuid import UUID

from src.database import get_db
from . import service


async def valid_post_id(
    post_id: UUID,
    db: Session = Depends(get_db)
) -> dict:
    post = await service.get_by_id(db, post_id)
    if not post:
        raise HTTPException(404, "Post not found")

    return post
```

**src/posts/router.py 리팩토링**:
```python
@router.get("/{post_id}", response_model=PostResponse)
async def get_post(
    post: dict = Depends(valid_post_id)  # 이미 검증됨!
):
    return post


@router.put("/{post_id}", response_model=PostResponse)
async def update_post(
    post: dict = Depends(valid_post_id),  # 재사용!
    data: PostUpdate,
    db: Session = Depends(get_db)
):
    updated = await service.update(db, post["id"], data)
    return updated
```

---

## 기존 프로젝트 리팩토링

### Week 1: 구조 개선

**Task 1: 파일 타입별 → 도메인별 이동**
```bash
# Before
crud/user.py, crud/post.py
routers/user.py, routers/post.py

# After
src/users/service.py, src/users/router.py
src/posts/service.py, src/posts/router.py
```

**Task 2: Import 명시화**
```python
# Before
from crud.user import get_user

# After
from src.users import service as users_service
```

---

### Week 2: Async 전환

**Task 1: Sync → Async 라이브러리 교체**
```bash
pip uninstall requests psycopg2
pip install httpx asyncpg
```

**Task 2: 코드 수정**
```python
# Before
import requests

def get_data():
    response = requests.get("...")
    return response.json()

# After
import httpx

async def get_data():
    async with httpx.AsyncClient() as client:
        response = await client.get("...")
        return response.json()
```

---

### Week 3: Dependency 도입

**Task 1: 검증 로직 Dependency로 추출**
```python
# Before (3곳에 중복)
@router.get("/posts/{post_id}")
async def endpoint1(post_id: UUID):
    post = await service.get_by_id(post_id)
    if not post:
        raise HTTPException(404)
    # ...

# After (1곳에만)
async def valid_post_id(post_id: UUID) -> dict:
    post = await service.get_by_id(post_id)
    if not post:
        raise HTTPException(404)
    return post

@router.get("/posts/{post_id}")
async def endpoint1(post: dict = Depends(valid_post_id)):
    # 이미 검증됨!
```

---

## 체크리스트

### 새 프로젝트 시작 시

- [ ] 도메인 중심 구조 (`src/{domain}/`)
- [ ] .env + pydantic-settings
- [ ] Async 라이브러리 (httpx, asyncpg)
- [ ] Alembic 설정 (파일 템플릿, 네이밍 컨벤션)
- [ ] Custom BaseModel (필요시)
- [ ] Ruff 설정

---

### 새 도메인 추가 시

- [ ] `src/{domain}/` 폴더 생성
- [ ] `router.py` (필수)
- [ ] `schemas.py` (필수)
- [ ] `models.py` (DB 사용 시)
- [ ] `service.py` (비즈니스 로직)
- [ ] `dependencies.py` (검증 2개 이상)
- [ ] `main.py`에 라우터 등록

---

### 새 엔드포인트 작성 시

- [ ] Pydantic Schema 정의
- [ ] `response_model` 설정
- [ ] `status_code` 설정
- [ ] Async/Sync 올바른 선택
- [ ] Dependency로 검증 재사용
- [ ] 에러 처리

---

## 성능 최적화 팁

### 1. DB 쿼리

**❌ N+1 쿼리**:
```python
posts = await db.query(Post).all()
for post in posts:
    creator = await db.query(User).filter(User.id == post.creator_id).first()
```

**✅ Join**:
```python
posts = await db.query(Post).join(User).all()
```

---

### 2. Async 최대 활용

**❌ 순차 호출**:
```python
user = await get_user(user_id)
posts = await get_posts(user_id)
comments = await get_comments(user_id)
# 3초 = 1초 + 1초 + 1초
```

**✅ 병렬 호출**:
```python
import asyncio

user, posts, comments = await asyncio.gather(
    get_user(user_id),
    get_posts(user_id),
    get_comments(user_id)
)
# 1초 = max(1초, 1초, 1초)
```

---

### 3. Response 크기

**❌ 모든 필드 반환**:
```python
@router.get("/users")
async def get_users():
    users = await db.query(User).all()
    return users  # 50개 필드!
```

**✅ 필요한 필드만**:
```python
class UserBrief(BaseModel):
    id: UUID4
    username: str

@router.get("/users", response_model=list[UserBrief])
async def get_users():
    users = await db.query(User).all()
    return users  # 2개 필드만
```

---

## 예상 효과

### 구조 개선
- ✅ 코드 탐색 시간: 50% 단축
- ✅ 신규 팀원 온보딩: 3일 → 1일
- ✅ Microservice 전환: 쉬움

### Async 전환
- ✅ 동시 요청 처리: 40개 → 1000개+
- ✅ 응답 시간: 30% 단축

### Dependency 도입
- ✅ 검증 코드 중복: 80% 제거
- ✅ 테스트 작성: 3배 빠름

### Pydantic 활용
- ✅ 런타임 에러: 50% 감소
- ✅ 문서 품질: 자동 생성

---

## 다음 단계

### 1주: Mini Project
- [ ] FastAPI 프로젝트 시작
- [ ] 2-3개 도메인 구현
- [ ] Best Practices 적용

### 1개월: 실무 적용
- [ ] 기존 프로젝트 리팩토링
- [ ] 팀 공유
- [ ] 성과 측정

### 3개월: 고급 패턴
- [ ] Auth (JWT, OAuth2)
- [ ] Background Tasks (Celery)
- [ ] Monitoring (Sentry, Prometheus)

---

**분석 완료**: 2025-01-18
