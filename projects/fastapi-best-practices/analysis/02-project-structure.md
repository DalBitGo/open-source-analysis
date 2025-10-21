# FastAPI 프로젝트 구조 패턴

> **핵심**: 도메인 중심 구조 (Domain-Driven Structure)

---

## 문제: 전통적인 파일 타입별 구조

### Before: 파일 타입별 (Type-based)

```
fastapi-project/
├── crud/
│   ├── user.py
│   ├── post.py
│   ├── comment.py
│   └── payment.py
├── routers/
│   ├── user.py
│   ├── post.py
│   ├── comment.py
│   └── payment.py
├── models/
│   ├── user.py
│   ├── post.py
│   ├── comment.py
│   └── payment.py
└── schemas/
    ├── user.py
    ├── post.py
    ├── comment.py
    └── payment.py
```

### 문제점

**1. 도메인 로직 분산**
```
"User 관련 코드 수정하려면?"
→ crud/user.py
→ routers/user.py
→ models/user.py
→ schemas/user.py
→ 4개 파일 왔다갔다!
```

**2. Import 지옥**
```python
# routers/user.py에서
from crud.user import get_user, create_user, update_user
from models.user import User
from schemas.user import UserCreate, UserUpdate, UserResponse
from crud.post import get_user_posts  # 도메인 경계 불명확
from crud.payment import get_user_payments
# 15줄 import...
```

**3. Microservice 전환 어려움**
```
"User 서비스 분리하려면?"
→ crud/user.py만 가져가면 안 됨
→ routers/, models/, schemas/ 각각에서 찾아야 함
→ 의존성 파악 어려움
```

**4. 팀 협업 충돌**
```
"A팀: User 작업, B팀: Post 작업"
→ routers/user.py (A팀)
→ routers/post.py (B팀)
→ 같은 routers/ 폴더 → Git 충돌 가능성
```

---

## 해결책: 도메인 중심 구조 (Domain-Driven)

### After: Netflix Dispatch 패턴

```
fastapi-project/
├── alembic/                  # DB 마이그레이션
├── src/
│   ├── auth/                 # 도메인 1: 인증
│   │   ├── router.py
│   │   ├── schemas.py        # Pydantic models
│   │   ├── models.py         # DB models
│   │   ├── dependencies.py
│   │   ├── service.py
│   │   ├── config.py         # 로컬 설정
│   │   ├── constants.py
│   │   ├── exceptions.py
│   │   └── utils.py
│   │
│   ├── posts/                # 도메인 2: 게시글
│   │   ├── router.py
│   │   ├── schemas.py
│   │   ├── models.py
│   │   ├── dependencies.py
│   │   ├── service.py
│   │   ├── constants.py
│   │   ├── exceptions.py
│   │   └── utils.py
│   │
│   ├── aws/                  # 도메인 3: 외부 서비스
│   │   ├── client.py         # 외부 API 통신
│   │   ├── schemas.py
│   │   ├── config.py
│   │   ├── constants.py
│   │   ├── exceptions.py
│   │   └── utils.py
│   │
│   ├── config.py             # 글로벌 설정
│   ├── models.py             # 글로벌 모델
│   ├── exceptions.py         # 글로벌 예외
│   ├── pagination.py         # 글로벌 모듈
│   ├── database.py           # DB 연결
│   └── main.py               # FastAPI app
│
├── tests/
│   ├── auth/
│   ├── posts/
│   └── aws/
│
├── templates/
│   └── index.html
│
├── requirements/
│   ├── base.txt
│   ├── dev.txt
│   └── prod.txt
│
├── .env
├── .gitignore
├── logging.ini
└── alembic.ini
```

---

## 파일별 역할 정의

### 도메인 레벨 파일

| 파일 | 역할 | 예시 |
|------|------|------|
| **router.py** | API 엔드포인트 정의 | `@router.get("/posts/{post_id}")` |
| **schemas.py** | Pydantic 모델 (요청/응답) | `class PostCreate(BaseModel)` |
| **models.py** | DB 모델 (SQLAlchemy) | `class Post(Base)` |
| **service.py** | 비즈니스 로직 | `async def create_post(...)` |
| **dependencies.py** | 라우터 의존성 | `async def valid_post_id(...)` |
| **constants.py** | 상수, 에러 코드 | `MAX_POST_LENGTH = 5000` |
| **config.py** | 로컬 환경 변수 | `class AuthConfig(BaseSettings)` |
| **exceptions.py** | 도메인 예외 | `class PostNotFound(Exception)` |
| **utils.py** | 유틸리티 함수 | `def normalize_title(...)` |

---

### 글로벌 레벨 파일

| 파일 | 역할 | 예시 |
|------|------|------|
| **src/main.py** | FastAPI 앱 초기화 | `app = FastAPI()` |
| **src/config.py** | 전역 설정 | `DATABASE_URL`, `ENVIRONMENT` |
| **src/models.py** | 공통 DB 모델 | `class TimestampMixin` |
| **src/exceptions.py** | 전역 예외 | `class Unauthorized(HTTPException)` |
| **src/pagination.py** | 공통 모듈 | `class Page(Generic[T])` |
| **src/database.py** | DB 연결 관리 | `engine`, `SessionLocal` |

---

## 핵심 원칙

### 1. 도메인 캡슐화

**원칙**: 하나의 도메인은 하나의 폴더에
```
src/posts/
  - 게시글 관련 모든 것
  - 독립적으로 이해 가능
  - 다른 도메인 최소 의존
```

**예시**:
```python
# src/posts/ 안에서만 작업
# router.py
from .schemas import PostCreate, PostResponse  # 상대 import
from .service import create_post
from .dependencies import valid_post_id
```

---

### 2. 명시적 크로스 도메인 Import

**나쁜 예** ❌:
```python
# src/posts/service.py
from ..auth.dependencies import get_current_user  # 헷갈림
```

**좋은 예** ✅:
```python
# src/posts/service.py
from src.auth import constants as auth_constants
from src.notifications import service as notification_service
from src.posts.constants import ErrorCode as PostsErrorCode
```

**이유**:
- 의존성 명확
- 이름 충돌 방지
- IDE 자동완성 정확

---

### 3. 외부 서비스는 `client.py`

**패턴**: 외부 API는 도메인처럼 취급
```
src/aws/
  ├── client.py       # boto3 래핑
  ├── schemas.py      # S3 응답 모델
  ├── config.py       # AWS_KEY 등
  └── exceptions.py   # S3UploadFailed 등
```

**예시**:
```python
# src/aws/client.py
class S3Client:
    def __init__(self, config: AWSConfig):
        self.s3 = boto3.client('s3', ...)

    async def upload_file(self, file: UploadFile) -> str:
        # S3 업로드 로직
        pass

# src/posts/service.py
from src.aws.client import S3Client

async def create_post_with_image(data: PostCreate, image: UploadFile):
    image_url = await s3_client.upload_file(image)
    # ...
```

---

## Before/After 비교

### 시나리오: "Post 도메인 수정"

**Before (파일 타입별)**:
```
1. routers/post.py 열기
2. crud/post.py 찾아가기
3. models/post.py 확인
4. schemas/post.py 수정
5. crud/post.py로 돌아가기
6. routers/post.py로 돌아가기

→ 6번 파일 이동
→ 20번 스크롤
→ 5분 소요
```

**After (도메인별)**:
```
1. src/posts/ 폴더 열기
2. 필요한 파일만 수정
   - service.py (비즈니스 로직)
   - schemas.py (응답 모델)

→ 1번 폴더 이동
→ 5번 스크롤
→ 1분 소요
```

---

### 시나리오: "Auth 서비스 분리" (Microservice)

**Before**:
```bash
# 각 폴더에서 auth 관련 파일 찾기
cp crud/user.py auth-service/
cp routers/auth.py auth-service/
cp models/user.py auth-service/
cp schemas/auth.py auth-service/

# 의존성 찾기 (여러 파일 분산)
grep -r "import.*user" .  # 50개 결과...
```

**After**:
```bash
# 폴더 하나만 복사
cp -r src/auth/ auth-service/

# 의존성 명확 (Import 보면 알 수 있음)
grep -r "from src.auth" .  # 10개 결과
```

---

## 실전 적용

### 1. 새 도메인 추가 체크리스트

**Step 1: 폴더 생성**
```bash
mkdir -p src/payments
```

**Step 2: 필수 파일 생성**
```bash
cd src/payments
touch __init__.py router.py schemas.py models.py service.py
```

**Step 3: 선택적 파일 (필요 시)**
```bash
touch dependencies.py config.py constants.py exceptions.py utils.py
```

**Step 4: main.py에 라우터 등록**
```python
# src/main.py
from src.payments.router import router as payments_router

app.include_router(payments_router, prefix="/payments", tags=["payments"])
```

---

### 2. 파일별 템플릿

**router.py**:
```python
from fastapi import APIRouter, Depends

from .schemas import PaymentCreate, PaymentResponse
from .dependencies import valid_payment_id
from . import service

router = APIRouter()


@router.get("/{payment_id}", response_model=PaymentResponse)
async def get_payment(payment: dict = Depends(valid_payment_id)):
    return payment


@router.post("/", response_model=PaymentResponse, status_code=201)
async def create_payment(data: PaymentCreate):
    payment = await service.create_payment(data)
    return payment
```

**schemas.py**:
```python
from pydantic import BaseModel, Field, UUID4


class PaymentBase(BaseModel):
    amount: int = Field(gt=0)
    currency: str = Field(pattern="^[A-Z]{3}$")


class PaymentCreate(PaymentBase):
    pass


class PaymentResponse(PaymentBase):
    id: UUID4
    status: str
```

**service.py**:
```python
from typing import Any

from .schemas import PaymentCreate


async def create_payment(data: PaymentCreate) -> dict[str, Any]:
    # 비즈니스 로직
    # DB 저장
    # 외부 API 호출
    pass


async def get_by_id(payment_id: str) -> dict[str, Any] | None:
    # DB 조회
    pass
```

**dependencies.py**:
```python
from fastapi import Depends
from pydantic import UUID4

from . import service
from .exceptions import PaymentNotFound


async def valid_payment_id(payment_id: UUID4) -> dict:
    payment = await service.get_by_id(payment_id)
    if not payment:
        raise PaymentNotFound()

    return payment
```

---

### 3. 언제 파일을 만드는가?

| 파일 | 만드는 시점 |
|------|-----------|
| **router.py** | ✅ 항상 (필수) |
| **schemas.py** | ✅ 항상 (필수) |
| **models.py** | ✅ DB 사용 시 (거의 항상) |
| **service.py** | ✅ 비즈니스 로직 있으면 (거의 항상) |
| **dependencies.py** | ⚠️ 검증 로직 2개 이상 |
| **config.py** | ⚠️ 환경 변수 3개 이상 |
| **constants.py** | ⚠️ 상수 5개 이상 |
| **exceptions.py** | ⚠️ 커스텀 예외 2개 이상 |
| **utils.py** | ⚠️ 유틸 함수 3개 이상 |

**원칙**: YAGNI (You Ain't Gonna Need It)
- 필요할 때 만들기
- 미리 만들지 않기

---

## 구조 선택 기준

### Microservice (작은 프로젝트)
→ **파일 타입별 구조** OK
```
3-5개 엔드포인트
단일 도메인
→ 복잡도 낮음
```

### Monolith (큰 프로젝트)
→ **도메인 중심 구조** 필수
```
20+ 엔드포인트
5+ 도메인
→ 도메인별 분리 필수
```

---

## Netflix Dispatch 참고

**원본**: https://github.com/Netflix/dispatch

**구조**:
```
dispatch/
├── src/
│   ├── dispatch/
│   │   ├── incident/
│   │   │   ├── service.py
│   │   │   ├── models.py
│   │   │   └── views.py
│   │   ├── individual/
│   │   ├── team/
│   │   └── ...
```

**차이점**:
- Dispatch: `views.py` (Django 스타일)
- 우리: `router.py` (FastAPI 스타일)

**공통점**:
- 도메인 중심
- 파일 역할 명확
- 독립적 모듈

---

## 핵심 요약

### 구조 원칙

1. **도메인 = 폴더**
   - `src/auth/`, `src/posts/`, `src/payments/`

2. **명시적 Import**
   - `from src.auth import constants as auth_constants`

3. **외부 서비스 = 도메인**
   - `src/aws/client.py`, `src/stripe/client.py`

4. **파일 역할 명확**
   - `router.py` (API), `service.py` (로직), `schemas.py` (검증)

---

## 다음 문서

**03-async-patterns.md**:
- Async vs Sync 선택 기준
- I/O Intensive vs CPU Intensive
- Thread Pool 함정
- 실전 사례
