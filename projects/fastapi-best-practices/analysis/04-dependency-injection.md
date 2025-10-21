# FastAPI Dependency Injection 패턴

> **핵심**: DI는 단순 주입이 아니라 Validation & Reuse 도구

---

## 문제: 검증 로직 중복

### Before: 모든 엔드포인트에서 반복

```python
@router.get("/posts/{post_id}")
async def get_post(post_id: UUID4):
    post = await service.get_by_id(post_id)
    if not post:
        raise HTTPException(404, "Post not found")
    return post


@router.put("/posts/{post_id}")
async def update_post(post_id: UUID4, data: PostUpdate):
    post = await service.get_by_id(post_id)  # 중복!
    if not post:
        raise HTTPException(404, "Post not found")  # 중복!

    updated = await service.update(post_id, data)
    return updated


@router.delete("/posts/{post_id}")
async def delete_post(post_id: UUID4):
    post = await service.get_by_id(post_id)  # 또 중복!
    if not post:
        raise HTTPException(404, "Post not found")  # 또 중복!

    await service.delete(post_id)
    return {"status": "deleted"}
```

**문제점**:
- 3개 엔드포인트, 3번 반복
- 에러 메시지 불일치 가능
- 테스트 3번 작성
- 로직 변경 시 3곳 수정

---

## 해결책: Dependency로 추상화

### After: 한 번만 정의, 여러 번 재사용

```python
# dependencies.py
async def valid_post_id(post_id: UUID4) -> dict[str, Any]:
    post = await service.get_by_id(post_id)
    if not post:
        raise PostNotFound()  # 커스텀 예외

    return post


# router.py
@router.get("/posts/{post_id}", response_model=PostResponse)
async def get_post(post: dict = Depends(valid_post_id)):
    return post  # 이미 검증됨!


@router.put("/posts/{post_id}", response_model=PostResponse)
async def update_post(
    post: dict = Depends(valid_post_id),  # 재사용!
    data: PostUpdate
):
    updated = await service.update(post["id"], data)
    return updated


@router.delete("/posts/{post_id}")
async def delete_post(post: dict = Depends(valid_post_id)):  # 재사용!
    await service.delete(post["id"])
    return {"status": "deleted"}
```

**효과**:
- ✅ 검증 로직 1곳에만
- ✅ 일관된 에러 처리
- ✅ 테스트 1번만
- ✅ 변경 시 1곳만 수정

---

## 패턴 1: Chain Dependencies

### 문제: 복잡한 검증 체인

```
"소유자만 게시글 수정 가능"

검증 순서:
1. post_id 존재 확인
2. JWT 토큰 검증
3. 사용자 = 게시글 소유자 확인
```

---

### 해결: Dependency 체이닝

```python
# dependencies.py
from fastapi.security import OAuth2PasswordBearer
from jose import JWTError, jwt


# Dependency 1: Post 존재 확인
async def valid_post_id(post_id: UUID4) -> dict[str, Any]:
    post = await service.get_by_id(post_id)
    if not post:
        raise PostNotFound()

    return post


# Dependency 2: JWT 파싱
async def parse_jwt_data(
    token: str = Depends(OAuth2PasswordBearer(tokenUrl="/auth/token"))
) -> dict[str, Any]:
    try:
        payload = jwt.decode(token, "JWT_SECRET", algorithms=["HS256"])
    except JWTError:
        raise InvalidCredentials()

    return {"user_id": payload["id"]}


# Dependency 3: 소유권 확인 (1 + 2 사용!)
async def valid_owned_post(
    post: dict = Depends(valid_post_id),          # 의존!
    token_data: dict = Depends(parse_jwt_data),   # 의존!
) -> dict[str, Any]:
    if post["creator_id"] != token_data["user_id"]:
        raise UserNotOwner()

    return post


# router.py
@router.put("/posts/{post_id}")
async def update_post(
    post: dict = Depends(valid_owned_post),  # 1+2+3 자동 실행!
    data: PostUpdate
):
    updated = await service.update(post["id"], data)
    return updated
```

**실행 순서**:
```
1. valid_owned_post 호출
   ├─ 2. valid_post_id 호출 → Post 검증
   └─ 3. parse_jwt_data 호출 → JWT 검증
4. 소유권 확인
5. update_post 본문 실행
```

---

## 패턴 2: Dependency 재사용 & 캐싱

### 핵심: 같은 Dependency는 1번만 실행

```python
# dependencies.py
async def parse_jwt_data(
    token: str = Depends(OAuth2PasswordBearer(tokenUrl="/auth/token"))
) -> dict:
    print("🔍 JWT 파싱 중...")  # 디버깅용

    try:
        payload = jwt.decode(token, "JWT_SECRET", algorithms=["HS256"])
    except JWTError:
        raise InvalidCredentials()

    return {"user_id": payload["id"]}


async def valid_active_creator(
    token_data: dict = Depends(parse_jwt_data),  # 재사용 1
):
    user = await users_service.get_by_id(token_data["user_id"])
    if not user["is_active"]:
        raise UserIsBanned()

    if not user["is_creator"]:
       raise UserNotCreator()

    return user


async def valid_owned_post(
    post: dict = Depends(valid_post_id),
    token_data: dict = Depends(parse_jwt_data),  # 재사용 2
) -> dict:
    if post["creator_id"] != token_data["user_id"]:
        raise UserNotOwner()

    return post


# router.py
@router.get("/users/{user_id}/posts/{post_id}", response_model=PostResponse)
async def get_user_post(
    worker: BackgroundTasks,
    post: dict = Depends(valid_owned_post),        # parse_jwt_data 포함
    user: dict = Depends(valid_active_creator),    # parse_jwt_data 또 포함!
):
    """Get post that belong to the active creator."""
    worker.add_task(notifications_service.send_email, user["id"])
    return post
```

**실행 결과**:
```
🔍 JWT 파싱 중...  ← 1번만 출력!
```

**FastAPI 캐싱**:
- `parse_jwt_data`가 3번 사용됨
  1. `valid_owned_post` 내부
  2. `valid_active_creator` 내부
  3. (결과적으로 2번 호출 코드)
- 하지만 **실제 실행은 1번**
- 요청 범위(Request Scope) 내 캐싱

---

## 패턴 3: 작은 Dependency로 분해

### 원칙: Single Responsibility

**나쁜 예** ❌:
```python
async def validate_everything(post_id: UUID4, token: str):
    # 1. Post 검증
    post = await service.get_by_id(post_id)
    if not post:
        raise PostNotFound()

    # 2. JWT 검증
    try:
        payload = jwt.decode(token, "SECRET", algorithms=["HS256"])
    except JWTError:
        raise InvalidCredentials()

    # 3. 소유권 검증
    if post["creator_id"] != payload["id"]:
        raise UserNotOwner()

    # 4. 활성 사용자 검증
    user = await users_service.get_by_id(payload["id"])
    if not user["is_active"]:
        raise UserIsBanned()

    return post, user
```

**문제**:
- 재사용 불가능
- 테스트 어려움
- 변경 시 영향 범위 큼

---

**좋은 예** ✅:
```python
# 1. Post 검증 (독립)
async def valid_post_id(post_id: UUID4) -> dict:
    post = await service.get_by_id(post_id)
    if not post:
        raise PostNotFound()
    return post


# 2. JWT 검증 (독립)
async def parse_jwt_data(
    token: str = Depends(OAuth2PasswordBearer(tokenUrl="/auth/token"))
) -> dict:
    try:
        payload = jwt.decode(token, "SECRET", algorithms=["HS256"])
    except JWTError:
        raise InvalidCredentials()
    return {"user_id": payload["id"]}


# 3. 소유권 검증 (1 + 2 조합)
async def valid_owned_post(
    post: dict = Depends(valid_post_id),
    token_data: dict = Depends(parse_jwt_data),
) -> dict:
    if post["creator_id"] != token_data["user_id"]:
        raise UserNotOwner()
    return post


# 4. 활성 사용자 검증 (2 활용)
async def valid_active_user(
    token_data: dict = Depends(parse_jwt_data),
) -> dict:
    user = await users_service.get_by_id(token_data["user_id"])
    if not user["is_active"]:
        raise UserIsBanned()
    return user
```

**장점**:
- 각 Dependency 독립적
- 자유롭게 조합 가능
- 테스트 쉬움

---

## 패턴 4: REST 경로 변수 재사용

### 문제: 경로 변수 이름 불일치

```python
# profiles/
@router.get("/profiles/{profile_id}")
async def get_profile(profile: dict = Depends(valid_profile_id)):
    return profile


# creators/ (같은 Profile인데 이름 다름!)
@router.get("/creators/{creator_id}")  # ❌ creator_id
async def get_creator(creator: dict = Depends(valid_creator_id)):
    return creator
```

**문제**:
- `valid_creator_id`가 `valid_profile_id` 재사용 불가
- 중복 코드 발생

---

### 해결: 경로 변수 통일

```python
# src/profiles/dependencies.py
async def valid_profile_id(profile_id: UUID4) -> dict:
    profile = await service.get_by_id(profile_id)
    if not profile:
        raise ProfileNotFound()

    return profile


# src/creators/dependencies.py
async def valid_creator_id(
    profile: dict = Depends(valid_profile_id)  # 재사용!
) -> dict:
    if not profile["is_creator"]:
       raise ProfileNotCreator()

    return profile


# src/profiles/router.py
@router.get("/profiles/{profile_id}", response_model=ProfileResponse)
async def get_profile(profile: dict = Depends(valid_profile_id)):
    return profile


# src/creators/router.py
@router.get("/creators/{profile_id}", response_model=ProfileResponse)  # ✅ profile_id 통일
async def get_creator(
    creator_profile: dict = Depends(valid_creator_id)
):
    return creator_profile
```

**핵심**:
- 경로 변수 이름: `profile_id` 통일
- Dependency 체이닝으로 재사용

---

## 패턴 5: 복잡한 Validation

### Pydantic으로 못 하는 검증

**Pydantic 범위**:
```python
class UserCreate(BaseModel):
    username: str = Field(min_length=3, max_length=20)
    email: EmailStr
    age: int = Field(ge=18)

    @field_validator("username")
    @classmethod
    def valid_username(cls, v: str) -> str:
        if not re.match(r"^[a-zA-Z0-9_]+$", v):
            raise ValueError("Only alphanumeric and underscore")
        return v
```

**못 하는 것**:
- ❌ DB 조회 ("이메일 중복 확인")
- ❌ 외부 API 호출 ("주소 유효성")
- ❌ 다른 리소스 참조 ("부모 Post 존재 확인")

---

### Dependency로 해결

```python
# dependencies.py
async def unique_email(email: EmailStr) -> EmailStr:
    existing = await users_service.get_by_email(email)
    if existing:
        raise EmailAlreadyExists()

    return email


async def valid_parent_post(parent_id: UUID4 | None) -> UUID4 | None:
    if parent_id is None:
        return None

    post = await posts_service.get_by_id(parent_id)
    if not post:
        raise ParentPostNotFound()

    return parent_id


# router.py
@router.post("/users", response_model=UserResponse)
async def create_user(
    data: UserCreate,
    email: EmailStr = Depends(unique_email),  # DB 검증!
):
    user = await service.create(data)
    return user


@router.post("/posts", response_model=PostResponse)
async def create_post(
    data: PostCreate,
    parent_id: UUID4 | None = Depends(valid_parent_post),  # 참조 검증!
):
    post = await service.create(data, parent_id=parent_id)
    return post
```

---

## async Dependency 권장

### ❌ Sync Dependency

```python
def get_current_user(token: str = Depends(oauth2_scheme)):
    # Thread Pool 사용 (리소스 낭비)
    payload = jwt.decode(token, "SECRET", algorithms=["HS256"])
    return payload
```

**문제**:
- 간단한 작업도 Thread Pool
- 비효율적

---

### ✅ Async Dependency

```python
async def get_current_user(token: str = Depends(oauth2_scheme)):
    # Event Loop에서 직접 실행 (빠름!)
    payload = jwt.decode(token, "SECRET", algorithms=["HS256"])
    return payload
```

**장점**:
- Thread 비용 없음
- 빠름
- 수천 요청 처리 가능

**예외**: DB 호출 등 I/O 작업이면 당연히 async

---

## 실전 템플릿

### 1. 리소스 존재 확인

```python
async def valid_{resource}_id({resource}_id: UUID4) -> dict:
    {resource} = await service.get_by_id({resource}_id)
    if not {resource}:
        raise {Resource}NotFound()

    return {resource}


# 예시
async def valid_post_id(post_id: UUID4) -> dict:
    post = await service.get_by_id(post_id)
    if not post:
        raise PostNotFound()
    return post
```

---

### 2. 인증 & 권한

```python
async def get_current_user(
    token: str = Depends(OAuth2PasswordBearer(tokenUrl="/auth/token"))
) -> dict:
    payload = jwt.decode(token, "SECRET", algorithms=["HS256"])
    user = await users_service.get_by_id(payload["id"])
    if not user:
        raise UserNotFound()
    return user


async def require_admin(
    user: dict = Depends(get_current_user)
) -> dict:
    if not user["is_admin"]:
        raise Forbidden()
    return user
```

---

### 3. 페이지네이션

```python
async def pagination_params(
    page: int = Query(1, ge=1),
    page_size: int = Query(20, ge=1, le=100)
) -> dict:
    return {
        "skip": (page - 1) * page_size,
        "limit": page_size
    }


@router.get("/posts")
async def get_posts(
    params: dict = Depends(pagination_params)
):
    posts = await service.get_posts(**params)
    return posts
```

---

## 테스트 용이성

### Dependency Overrides

```python
# tests/test_posts.py
from fastapi.testclient import TestClient

from src.main import app
from src.posts.dependencies import valid_post_id


# Dependency Mock
async def mock_valid_post_id(post_id: UUID4) -> dict:
    return {"id": post_id, "title": "Test Post"}


# Override
app.dependency_overrides[valid_post_id] = mock_valid_post_id


client = TestClient(app)


def test_get_post():
    response = client.get("/posts/123")
    assert response.status_code == 200
    assert response.json()["title"] == "Test Post"
```

---

## 핵심 요약

### Dependency 활용법

1. **Validation**
   - Pydantic 못 하는 것 (DB, 외부 API)

2. **Reuse**
   - 여러 엔드포인트에서 재사용

3. **Chain**
   - 복잡한 검증 체인

4. **Cache**
   - 요청 범위 내 자동 캐싱

---

### 설계 원칙

1. **작게 분해**
   - 1 Dependency = 1 책임

2. **명확한 이름**
   - `valid_post_id`, `get_current_user`

3. **async 우선**
   - Thread Pool 비용 절약

4. **테스트 가능**
   - Override로 Mock 쉬움

---

## 다음 문서

**05-pydantic-patterns.md**:
- Pydantic 과도 사용 권장
- Custom BaseModel
- BaseSettings 분리
- Validation 고급 패턴
