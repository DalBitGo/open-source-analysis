# Pydantic & Database 패턴

> **핵심**: Pydantic 과도 사용 + SQL-first 접근

---

## Part 1: Pydantic 패턴

### 원칙: Excessively Use Pydantic

**철학**: Pydantic이 할 수 있으면 Pydantic에게

```python
from enum import Enum
from pydantic import BaseModel, EmailStr, Field, AnyUrl


class MusicBand(str, Enum):
   AEROSMITH = "AEROSMITH"
   QUEEN = "QUEEN"
   ACDC = "AC/DC"


class UserBase(BaseModel):
    first_name: str = Field(min_length=1, max_length=128)
    username: str = Field(
        min_length=1,
        max_length=128,
        pattern="^[A-Za-z0-9-_]+$"  # 정규식 검증
    )
    email: EmailStr  # 이메일 자동 검증
    age: int | None = Field(ge=18, default=None)  # 18세 이상
    favorite_band: MusicBand | None = None  # Enum만 허용
    website: AnyUrl | None = None  # URL 검증
```

**Pydantic이 제공하는 것**:
- ✅ 타입 검증
- ✅ 정규식 (`pattern`)
- ✅ 범위 검증 (`ge`, `le`, `gt`, `lt`)
- ✅ 길이 검증 (`min_length`, `max_length`)
- ✅ 이메일 검증 (`EmailStr`)
- ✅ URL 검증 (`AnyUrl`, `HttpUrl`)
- ✅ Enum 검증
- ✅ 커스텀 Validator

---

### Custom BaseModel

**문제**: 모든 datetime을 표준 형식으로 변환하고 싶다

```python
from datetime import datetime
from zoneinfo import ZoneInfo

from fastapi.encoders import jsonable_encoder
from pydantic import BaseModel, ConfigDict


def datetime_to_gmt_str(dt: datetime) -> str:
    """UTC 기준 ISO 8601 형식"""
    if not dt.tzinfo:
        dt = dt.replace(tzinfo=ZoneInfo("UTC"))

    return dt.strftime("%Y-%m-%dT%H:%M:%S%z")


class CustomModel(BaseModel):
    model_config = ConfigDict(
        json_encoders={datetime: datetime_to_gmt_str},
        populate_by_name=True,  # alias 지원
    )

    def serializable_dict(self, **kwargs):
        """직렬화 가능한 dict 반환"""
        default_dict = self.model_dump()
        return jsonable_encoder(default_dict)
```

**사용**:
```python
class Post(CustomModel):  # BaseModel 대신
    id: UUID4
    title: str
    created_at: datetime  # 자동으로 ISO 8601 변환!


post = Post(id=uuid4(), title="Test", created_at=datetime.now())
print(post.model_dump_json())
# {"id": "...", "title": "Test", "created_at": "2025-01-18T10:30:00+0000"}
```

---

### BaseSettings 분리

**문제**: 단일 Settings 클래스가 거대해짐

**Before** ❌:
```python
class Settings(BaseSettings):
    # Auth (10개)
    JWT_SECRET: str
    JWT_ALG: str
    JWT_EXP: int
    # ...

    # Database (5개)
    DATABASE_URL: PostgresDsn
    # ...

    # AWS (15개)
    AWS_KEY: str
    # ...

    # Redis (3개)
    # 총 50개 설정!
```

**After** ✅:
```python
# src/auth/config.py
from pydantic_settings import BaseSettings


class AuthConfig(BaseSettings):
    JWT_ALG: str
    JWT_SECRET: str
    JWT_EXP: int = 5  # minutes

    REFRESH_TOKEN_KEY: str
    REFRESH_TOKEN_EXP: timedelta = timedelta(days=30)

    SECURE_COOKIES: bool = True


auth_settings = AuthConfig()  # .env에서 자동 로드


# src/config.py (글로벌만)
class Config(BaseSettings):
    DATABASE_URL: PostgresDsn
    REDIS_URL: RedisDsn

    SITE_DOMAIN: str = "myapp.com"
    ENVIRONMENT: Environment = Environment.PRODUCTION

    SENTRY_DSN: str | None = None

    CORS_ORIGINS: list[str]
    CORS_HEADERS: list[str]

    APP_VERSION: str = "1.0"


settings = Config()
```

**사용**:
```python
# src/auth/service.py
from .config import auth_settings

def create_jwt(user_id: str) -> str:
    payload = {"id": user_id}
    token = jwt.encode(
        payload,
        auth_settings.JWT_SECRET,
        algorithm=auth_settings.JWT_ALG
    )
    return token
```

---

### ValueError → ValidationError 변환

```python
from pydantic import BaseModel, field_validator


class ProfileCreate(BaseModel):
    username: str
    password: str

    @field_validator("password", mode="after")
    @classmethod
    def valid_password(cls, password: str) -> str:
        if not re.match(STRONG_PASSWORD_PATTERN, password):
            raise ValueError(
                "Password must contain at least "
                "one lower character, "
                "one upper character, "
                "digit or "
                "special symbol"
            )

        return password
```

**응답 예시**:
```json
{
  "detail": [
    {
      "type": "value_error",
      "loc": ["body", "password"],
      "msg": "Value error, Password must contain at least one lower character, one upper character, digit or special symbol"
    }
  ]
}
```

**장점**:
- 사용자 친화적
- 자동 포맷팅
- 422 상태 코드

---

## Part 2: Database 패턴

### DB 네이밍 컨벤션

**원칙**:
1. `lower_case_snake`
2. 단수형 (`post`, `user`, not `posts`, `users`)
3. 모듈 Prefix (`payment_account`, `payment_bill`)
4. 일관성 유지

**예시**:
```python
# Good ✅
post
post_like
post_view
user_profile
payment_account
payment_transaction

# Bad ❌
Posts  # 대문자
post_likes  # 복수형
postLike  # camelCase
```

**FK 네이밍**:
```python
# 일반적으로 profile_id
posts.profile_id
comments.profile_id

# 구체적 의미가 있으면 구체적으로
chapters.course_id  # course_id (course만 들어감)
posts.creator_id    # creator_id (creator만 들어감)
```

**Datetime 네이밍**:
```python
created_at   # datetime
updated_at   # datetime
deleted_at   # datetime (soft delete)

opened_date  # date only
closed_date  # date only
```

---

### Index 네이밍 컨벤션

```python
from sqlalchemy import MetaData

POSTGRES_INDEXES_NAMING_CONVENTION = {
    "ix": "%(column_0_label)s_idx",
    "uq": "%(table_name)s_%(column_0_name)s_key",
    "ck": "%(table_name)s_%(constraint_name)s_check",
    "fk": "%(table_name)s_%(column_0_name)s_fkey",
    "pk": "%(table_name)s_pkey",
}

metadata = MetaData(naming_convention=POSTGRES_INDEXES_NAMING_CONVENTION)
```

**결과**:
```sql
-- 자동 생성되는 이름
posts_pkey              -- Primary Key
posts_title_idx         -- Index
posts_creator_id_fkey   -- Foreign Key
```

---

### Alembic 마이그레이션

**규칙**:

1. **설명적 이름**
```bash
# Bad
alembic revision -m "update"

# Good
alembic revision -m "add_user_email_index"
```

2. **날짜 포함 템플릿**
```ini
# alembic.ini
file_template = %%(year)d-%%(month).2d-%%(day).2d_%%(slug)s
```

**결과**:
```
2025-01-18_add_user_email_index.py
2025-01-19_create_posts_table.py
```

3. **Revertable**
```python
def upgrade():
    op.create_table('posts', ...)


def downgrade():
    op.drop_table('posts')  # 항상 작성!
```

---

### SQL-first, Pydantic-second

**원칙**: DB에서 할 수 있으면 DB에서

**나쁜 예** ❌:
```python
# Python에서 Join & 집계
async def get_posts(creator_id: UUID4):
    # 1. Posts 가져오기
    posts = await db.fetch_all(
        "SELECT * FROM posts WHERE creator_id = :id",
        {"id": creator_id}
    )

    # 2. 각 Post마다 Creator 정보 가져오기
    for post in posts:
        creator = await db.fetch_one(
            "SELECT * FROM profiles WHERE id = :id",
            {"id": post["creator_id"]}
        )
        post["creator"] = creator  # Python에서 조합

    return posts
```

**문제**:
- N+1 쿼리
- 느림
- 메모리 많이 사용

---

**좋은 예** ✅:
```python
# SQL에서 Join & JSON 집계
async def get_posts(creator_id: UUID4) -> list[dict]:
    select_query = (
        select(
            posts.c.id,
            posts.c.slug,
            posts.c.title,
            func.json_build_object(
               text("'id', profiles.id"),
               text("'first_name', profiles.first_name"),
               text("'last_name', profiles.last_name"),
               text("'username', profiles.username"),
            ).label("creator"),  # JSON 형태로 집계!
        )
        .select_from(posts.join(profiles, posts.c.owner_id == profiles.c.id))
        .where(posts.c.owner_id == creator_id)
        .limit(10)
        .order_by(desc(posts.c.created_at))
    )

    return await database.fetch_all(select_query)
```

**Pydantic 모델**:
```python
class Creator(BaseModel):
    id: UUID4
    first_name: str
    last_name: str
    username: str


class Post(BaseModel):
    id: UUID4
    slug: str
    title: str
    creator: Creator  # Nested model


# Response
@router.get("/creators/{creator_id}/posts", response_model=list[Post])
async def get_creator_posts(creator_id: UUID4):
    posts = await service.get_posts(creator_id)
    return posts  # Pydantic이 자동 검증!
```

**장점**:
- 1번 쿼리
- 빠름
- DB가 최적화
- Pydantic은 검증만

---

## 기타 Best Practices

### 1. Response Serialization 주의

**문제**: Pydantic 객체 2번 생성

```python
@router.get("/", response_model=ProfileResponse)
async def root():
    return ProfileResponse()  # 1번 생성

# FastAPI 내부에서
# 1. ProfileResponse를 dict로 변환
# 2. response_model로 다시 검증
# 3. ProfileResponse 또 생성 (2번!)
```

**해결**: dict 반환
```python
@router.get("/", response_model=ProfileResponse)
async def root():
    return {"id": "...", "name": "..."}  # dict 반환!
```

---

### 2. Docs 설정

```python
from fastapi import FastAPI
from starlette.config import Config

config = Config(".env")

ENVIRONMENT = config("ENVIRONMENT")
SHOW_DOCS_ENVIRONMENT = ("local", "staging")

app_configs = {"title": "My Cool API"}
if ENVIRONMENT not in SHOW_DOCS_ENVIRONMENT:
   app_configs["openapi_url"] = None  # Docs 숨김 (Prod)

app = FastAPI(**app_configs)
```

---

### 3. Async Test Client

```python
import pytest
from httpx import AsyncClient, ASGITransport

from src.main import app


@pytest.fixture
async def client():
    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test"
    ) as client:
        yield client


@pytest.mark.asyncio
async def test_create_post(client):
    resp = await client.post("/posts", json={"title": "Test"})
    assert resp.status_code == 201
```

---

### 4. Ruff (Linter)

```shell
#!/bin/sh -e
set -x

ruff check --fix src
ruff format src
```

**대체하는 도구**:
- black (formatter)
- isort (import 정렬)
- flake8 (linter)
- autoflake

---

## 핵심 요약

### Pydantic

1. **과도 사용 권장**
   - EmailStr, AnyUrl, Enum 등 최대 활용

2. **Custom BaseModel**
   - 전역 설정 (datetime 형식 등)

3. **BaseSettings 분리**
   - 도메인별 Config 클래스

---

### Database

1. **네이밍 컨벤션**
   - lower_snake_case, 단수형, 일관성

2. **SQL-first**
   - Join, 집계는 DB에서
   - Pydantic은 검증만

3. **Alembic**
   - 설명적 이름, 날짜 포함

---

## 다음 문서

**06-apply-to-my-work.md**:
- 전체 패턴 요약
- 프로젝트 시작 체크리스트
- 단계별 적용 계획
