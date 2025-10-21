# FastAPI Async 패턴

> **핵심**: I/O 특성에 따라 async/sync 선택

---

## 문제: Async/Sync 혼용의 함정

### 초보자의 흔한 오해

```python
"FastAPI는 async 프레임워크니까
async def만 쓰면 되겠지?"
→ ❌ 틀림!

"async/sync 둘 다 되니까
아무거나 써도 되겠지?"
→ ❌ 완전히 틀림!
```

---

## 핵심 개념

### FastAPI의 두 가지 실행 모드

| 라우트 타입 | 실행 방식 | 적합한 작업 |
|------------|----------|------------|
| **async def** | Event Loop에서 직접 실행 | I/O 비동기 작업 |
| **def** | Thread Pool에서 실행 | I/O 동기 작업 |

---

## 패턴 1: I/O Intensive Tasks

### 3가지 방법 비교

#### ❌ Terrible: async + blocking I/O

```python
import time
from fastapi import APIRouter

router = APIRouter()


@router.get("/terrible-ping")
async def terrible_ping():
    time.sleep(10)  # 🔥 블로킹!
    return {"pong": True}
```

**무슨 일이?**
```
1. FastAPI 요청 받음
2. Event Loop가 time.sleep(10) 만남
3. Event Loop 전체가 10초 대기
   ❌ 다른 요청 처리 불가
   ❌ DB 쿼리 불가
   ❌ 서버 멈춤!
4. 10초 후 응답
```

**증상**:
- 요청 1개만 처리
- 동시 접속 불가
- 서버가 죽은 것처럼 보임

---

#### ⚠️ Good: sync + blocking I/O

```python
@router.get("/good-ping")
def good_ping():
    time.sleep(10)  # 블로킹이지만...
    return {"pong": True}
```

**무슨 일이?**
```
1. FastAPI 요청 받음
2. 전체 함수를 Thread Pool로 보냄
3. Worker Thread가 time.sleep(10) 실행
   ✅ Event Loop는 자유로움
   ✅ 다른 요청 처리 가능
   ✅ DB 쿼리 가능
4. 10초 후 응답
```

**장점**:
- Event Loop 안 막힘
- 동시 요청 처리 가능

**단점**:
- Thread 비용 (메모리, CPU)
- Thread Pool 제한 (기본 40개)

---

#### ✅ Perfect: async + non-blocking I/O

```python
import asyncio


@router.get("/perfect-ping")
async def perfect_ping():
    await asyncio.sleep(10)  # 비블로킹!
    return {"pong": True}
```

**무슨 일이?**
```
1. FastAPI 요청 받음
2. Event Loop가 asyncio.sleep(10) 만남
3. Event Loop가 다른 작업으로 전환
   ✅ 다른 요청 처리
   ✅ DB 쿼리 실행
   ✅ 리소스 효율적
4. 10초 후 돌아와서 응답
```

**장점**:
- Event Loop 효율적 활용
- Thread 비용 없음
- 수천 동시 요청 가능

---

### 실전 예시

#### ❌ 나쁜 예: requests (sync)

```python
import requests  # Sync 라이브러리


@router.get("/users/{user_id}")
async def get_user(user_id: int):
    # 🔥 async 안에서 sync I/O → Event Loop 블로킹!
    response = requests.get(f"https://api.example.com/users/{user_id}")
    return response.json()
```

**문제**:
- `requests.get()`은 blocking
- Event Loop 멈춤
- 동시 요청 처리 불가

---

#### ✅ 좋은 예 1: httpx (async)

```python
import httpx


@router.get("/users/{user_id}")
async def get_user(user_id: int):
    async with httpx.AsyncClient() as client:
        response = await client.get(f"https://api.example.com/users/{user_id}")
        return response.json()
```

**장점**:
- 완전 비동기
- Event Loop 안 막힘
- 수백 요청 동시 처리

---

#### ✅ 좋은 예 2: requests + run_in_threadpool

```python
from fastapi import FastAPI
from fastapi.concurrency import run_in_threadpool
import requests  # Sync 라이브러리


@router.get("/users/{user_id}")
async def get_user(user_id: int):
    # Sync 라이브러리를 Thread Pool에서 실행
    response = await run_in_threadpool(
        requests.get,
        f"https://api.example.com/users/{user_id}"
    )
    return response.json()
```

**사용 시점**:
- Async 라이브러리 없을 때
- 레거시 SDK 사용 시 (boto3 등)

---

## 패턴 2: CPU Intensive Tasks

### 문제: CPU 작업은 다르다

```python
@router.post("/process-video")
async def process_video(video: UploadFile):
    # 🔥 CPU Intensive 작업 - async 소용없음!
    processed = await heavy_video_processing(video)  # 10초 소요
    return {"status": "done"}
```

**문제**:
- CPU가 10초 동안 100% 작업
- I/O 대기 없음
- `await`해도 Event Loop 블로킹

---

### 해결책: 별도 프로세스

#### GIL (Global Interpreter Lock) 이해

```
Python의 제약:
- 한 번에 1개 Thread만 실행 (CPU 작업 시)
- Thread Pool 무의미 (CPU 작업에는)
- I/O 작업에만 Thread Pool 효과적
```

#### ✅ Celery + Redis

```python
from celery import Celery

celery_app = Celery('tasks', broker='redis://localhost:6379')


@celery_app.task
def heavy_video_processing(video_path: str):
    # 별도 프로세스에서 실행
    # 10초 걸려도 FastAPI는 자유로움
    pass


@router.post("/process-video")
async def process_video(video: UploadFile):
    # 비동기 작업 큐에 넣고 즉시 반환
    task = heavy_video_processing.delay(video.filename)
    return {"task_id": task.id, "status": "processing"}
```

---

## 선택 기준 결정 트리

```
작업 특성 파악
    │
    ├─ I/O 작업? (DB, API, 파일)
    │   │
    │   ├─ Async 라이브러리 있음?
    │   │   YES → async def + await
    │   │   NO  → def (sync) 또는 run_in_threadpool
    │   │
    │   └─ 예시
    │       async def: httpx, asyncpg, aiofiles
    │       sync def:  requests, psycopg2, open()
    │
    └─ CPU 작업? (계산, 데이터 처리, 인코딩)
        │
        └─ 별도 프로세스로 (Celery, RQ)
```

---

## 실전 예시 모음

### 1. Database 쿼리

#### ✅ Async (asyncpg, databases)

```python
from databases import Database

database = Database("postgresql://...")


@router.get("/posts")
async def get_posts():
    query = "SELECT * FROM posts LIMIT 10"
    posts = await database.fetch_all(query)
    return posts
```

---

#### ✅ Sync (psycopg2)

```python
import psycopg2


@router.get("/posts")
def get_posts():
    # sync → Thread Pool에서 실행됨
    conn = psycopg2.connect("postgresql://...")
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM posts LIMIT 10")
    posts = cursor.fetchall()
    return posts
```

---

### 2. 파일 읽기

#### ✅ Async (aiofiles)

```python
import aiofiles


@router.get("/file/{filename}")
async def read_file(filename: str):
    async with aiofiles.open(f"files/{filename}", mode='r') as f:
        content = await f.read()
    return {"content": content}
```

---

#### ✅ Sync (open)

```python
@router.get("/file/{filename}")
def read_file(filename: str):
    with open(f"files/{filename}", 'r') as f:
        content = f.read()
    return {"content": content}
```

---

### 3. 외부 API 호출

#### ✅ Async (httpx)

```python
import httpx


@router.get("/weather/{city}")
async def get_weather(city: str):
    async with httpx.AsyncClient() as client:
        response = await client.get(f"https://api.weather.com/{city}")
        return response.json()
```

---

#### ❌ 나쁜 예 (requests in async)

```python
import requests


@router.get("/weather/{city}")
async def get_weather(city: str):
    # 🔥 Event Loop 블로킹!
    response = requests.get(f"https://api.weather.com/{city}")
    return response.json()
```

---

#### ✅ 차선책 (run_in_threadpool)

```python
import requests
from fastapi.concurrency import run_in_threadpool


@router.get("/weather/{city}")
async def get_weather(city: str):
    response = await run_in_threadpool(
        requests.get,
        f"https://api.weather.com/{city}"
    )
    return response.json()
```

---

## Thread Pool 주의사항

### 문제: Thread Pool 고갈

```python
# Thread Pool 기본 크기: 40개

# 동시 요청 100개 → 문제 발생!
@router.get("/slow")
def slow_endpoint():
    time.sleep(5)  # 5초 블로킹
    return {"done": True}
```

**무슨 일이?**
```
요청 1-40:  Thread Pool에서 실행 중 (5초)
요청 41-100: 대기 중... (Thread 없음)
→ 요청 41은 5초 후에야 시작
→ 지연 누적
```

**해결책**:
1. Async 라이브러리 사용 (Thread Pool 안 씀)
2. Thread Pool 크기 증가 (임시방편)
3. 작업 큐 사용 (Celery)

---

## Dependencies도 Async 권장

### ❌ Sync Dependency

```python
def get_current_user(token: str = Depends(oauth2_scheme)):
    # 작은 작업이지만 Thread Pool 사용
    user_id = decode_token(token)
    return user_id
```

**문제**:
- 간단한 작업도 Thread Pool 사용
- 리소스 낭비

---

### ✅ Async Dependency

```python
async def get_current_user(token: str = Depends(oauth2_scheme)):
    # Event Loop에서 직접 실행
    user_id = decode_token(token)  # CPU 작업이지만 빠름 (0.001초)
    return user_id
```

**장점**:
- Thread 비용 없음
- 빠름

---

## 성능 비교

### 시나리오: 외부 API 호출 (100 요청)

**Sync (requests)**:
```
Thread Pool: 40개
요청 1-40:  동시 실행 (1초)
요청 41-80: 대기 → 실행 (1초)
요청 81-100: 대기 → 실행 (1초)
──────────────────────
총: 3초
```

**Async (httpx)**:
```
Event Loop: 무제한 (메모리 허용 한도)
요청 1-100: 동시 실행 (1초)
──────────────────────
총: 1초 (3배 빠름!)
```

---

## 핵심 요약

### 선택 가이드

| 작업 타입 | Async 라이브러리 | 권장 방법 |
|-----------|-----------------|----------|
| **I/O + Async SDK** | httpx, asyncpg | `async def` + `await` ✅ |
| **I/O + Sync SDK** | requests, psycopg2 | `def` (sync) ⚠️ |
| **I/O + Sync SDK (레거시)** | boto3 | `run_in_threadpool` ⚠️ |
| **CPU Intensive** | - | Celery (별도 프로세스) ✅ |
| **간단한 로직** | - | `async def` (Thread 비용 절약) ✅ |

---

### 디버깅 팁

**증상: 서버가 멈춤**
```python
# 의심 코드 찾기
@router.get("/...")
async def endpoint():
    time.sleep(10)  # 🔥 이런 거!
    requests.get()  # 🔥 이것도!
```

**해결**:
1. `time.sleep()` → `asyncio.sleep()`
2. `requests` → `httpx` 또는 `run_in_threadpool`

---

## 참고 자료

**혼란스러운 사례** (StackOverflow):
1. [Flask vs FastAPI 성능](https://stackoverflow.com/questions/62976648/architecture-flask-vs-fastapi/70309597#70309597)
2. [FastAPI UploadFile이 느림](https://stackoverflow.com/questions/65342833/fastapi-uploadfile-is-slow-compared-to-flask)
3. [FastAPI가 순차 실행?](https://stackoverflow.com/questions/71516140/fastapi-runs-api-calls-in-serial-instead-of-parallel-fashion)

**공통 원인**: async 안에서 sync I/O

---

## 다음 문서

**04-dependency-injection.md**:
- Dependency Injection 고급 활용
- Chain Dependencies
- 캐싱 메커니즘
- Validation 패턴
