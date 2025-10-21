# fastapi-best-practices: 실행 흐름 분석

## 1. 핵심 실행 시나리오

이 프로젝트는 실제 실행 코드가 아닌 아키텍처 제안이므로, 가장 대표적인 시나리오인 **"사용자가 특정 게시물을 조회하는 HTTP GET 요청"** 이 제안된 아키텍처 내에서 어떻게 처리되는지 단계별로 분석합니다.

- **시나리오**: 클라이언트가 `GET /posts/{post_id}` 엔드포인트로 요청을 보낸다.
- **핵심 커맨드**: (가상의) Uvicorn/Gunicorn 웹 서버가 FastAPI 애플리케이션을 실행 중이다.

## 2. 요청 처리 실행 흐름 분석

`GET /posts/{post_id}` 요청은 다음과 같은 순서로 처리됩니다. 이는 제안된 아키텍처의 각 계층(Layer)이 어떻게 상호작용하는지를 명확히 보여줍니다.

- **입력(Input)**: 클라이언트로부터의 HTTP GET 요청. 경로에는 조회할 `post_id`가 포함되어 있습니다.

- **처리(Process)**:
    1.  **라우팅 (FastAPI Core)**: FastAPI는 들어온 요청의 경로(`/posts/{post_id}`)와 HTTP 메서드(`GET`)를 보고, `src/posts/router.py`에 `@router.get("/posts/{post_id}")`로 장식된(decorated) `get_post_by_id` 함수를 처리 대상으로 지정합니다.

    2.  **의존성 해결 (Dependency Resolution)**: FastAPI는 `get_post_by_id` 함수의 본문을 실행하기 **전에**, 함수의 파라미터 `post: dict = Depends(valid_post_id)`를 확인합니다.
        - `Depends(valid_post_id)`를 발견하고, `src/posts/dependencies.py`에 정의된 `valid_post_id` 함수를 실행 대상으로 지정합니다.
        - 요청 경로에서 `post_id` 값을 추출하여 `valid_post_id` 함수의 인자로 전달합니다.

    3.  **요청 검증 (Validation Layer - `dependencies.py`)**: `valid_post_id` 함수가 실행됩니다.
        - 이 함수는 `src/posts/service.py`의 `get_by_id(post_id)` 함수를 호출하여 데이터베이스에 해당 ID의 게시물이 있는지 조회합니다.
        - **(분기 1: 실패)** 만약 `service.py`가 `None`을 반환하면, `valid_post_id`는 `PostNotFound` 예외를 발생시킵니다. FastAPI는 이 예외를 잡아 미리 정의된 HTTP 404 Not Found 응답을 클라이언트에게 즉시 반환하고, **모든 처리를 중단합니다.**
        - **(분기 2: 성공)** 게시물이 존재하면, `valid_post_id` 함수는 조회된 게시물 데이터(`post` 객체)를 반환합니다.

    4.  **핵심 비즈니스 로직 (Business Logic Layer - `router.py`)**: FastAPI는 `valid_post_id`로부터 반환된 `post` 객체를 `get_post_by_id` 함수의 `post` 파라미터에 주입(inject)하고, 드디어 함수의 본문을 실행합니다.
        - 이 예시에서 `get_post_by_id` 함수의 본문은 매우 간단합니다: `return post`. 즉, 의존성 함수가 이미 모든 필요한 데이터 조회를 완료했으므로, 받은 데이터를 그대로 반환하기만 하면 됩니다.

    5.  **응답 모델 처리 (Serialization Layer - `schemas.py` & FastAPI Core)**: `get_post_by_id` 함수가 `post` 객체를 반환하면, FastAPI는 엔드포인트에 선언된 `response_model=PostResponse`를 확인합니다.
        - 반환된 `post` 객체의 필드들이 `src/posts/schemas.py`에 정의된 `PostResponse` Pydantic 모델의 필드들과 일치하는지, 타입은 올바른지 유효성을 검사합니다.
        - 유효성 검사를 통과하면, FastAPI는 `post` 객체를 JSON 형식의 문자열로 직렬화(serialization)합니다.

- **출력(Output)**: 직렬화된 JSON 데이터와 HTTP 200 OK 상태 코드를 담은 HTTP 응답이 클라이언트에게 최종적으로 전송됩니다.

## 3. 설계의 장점 요약

이러한 실행 흐름은 제안된 아키텍처의 장점을 명확히 보여줍니다.

- **선언적(Declarative)**: 라우터 함수는 "`valid_post_id`를 통과한 `post`가 필요하다"고 선언할 뿐, 어떻게 가져오고 검증하는지에 대한 절차적인 코드는 포함하지 않습니다.
- **관심사 분리(SoC)**: 요청 라우팅, 유효성 검증, 비즈니스 로직, 데이터 직렬화 등 각기 다른 관심사가 `FastAPI Core`, `dependencies.py`, `router.py`, `schemas.py`에 의해 명확하게 분리되어 처리됩니다.
- **안정성(Robustness)**: 비즈니스 로직이 실행되기 전에 모든 검증(게시물 존재 여부 등)이 먼저 완료되므로, `service.py`나 `router.py`의 함수들은 입력값이 항상 유효하다고 가정하고 코드를 작성할 수 있습니다.
