# Open Source Analysis Project
# 오픈소스 분석 프로젝트

> **코드는 문제를 해결하기 위한 의사결정의 결과물이다**

이 프로젝트는 오픈소스 프로젝트를 분석하여 **문제 해결력**과 **설계 능력**을 향상시키기 위한 학습 저장소입니다.

---

## 🎯 목표

단순히 "코드가 무엇을 하는가"가 아니라 **"왜 이런 선택을 했는가"**를 이해합니다.

- ❌ "이 클래스는 메시지를 전송한다"
- ✅ "왜 하나의 클래스가 아니라 여러 클래스로 나눴을까? → 배치 처리와 네트워크 전송의 책임 분리 → 각각 독립적으로 테스트 가능"

### 학습 목표

1. **문제 정의 능력**: 코드 작성 전에 "왜"를 명확히 하기
2. **선택지 평가 능력**: 여러 방법 중 최선을 고르는 기준 만들기
3. **트레이드오프 사고**: 완벽한 해결책은 없다는 것 이해하기
4. **패턴 적용 능력**: 비슷한 문제에 배운 패턴 적용하기
5. **설계 의사결정**: 아키텍처 결정의 근거 만들기

---

## 📋 분석 방법론

이 프로젝트는 **"문제 → 고민 → 해결"** 중심 분석 방법론을 사용합니다.

### 핵심 프레임워크

모든 코드를 다음 3단계로 분석합니다:

#### 1. 문제 (Problem)
- 어떤 문제를 해결하려고 하는가?
- 문제가 없었다면 어떤 불편함이 있었을까?
- 누구를 위한 문제인가? (개발자? 사용자? 시스템?)

#### 2. 고민 (Consideration)
- 어떤 선택지들이 있었는가?
- 각 선택지의 장단점은?
- 왜 다른 방법을 선택하지 않았는가?

#### 3. 해결 (Solution)
- 최종적으로 어떤 방법을 선택했는가?
- 이 해결책의 트레이드오프는?
- 어떤 부분을 희생하고 어떤 부분을 얻었는가?

### 자세한 방법론

전체 분석 방법론은 [`ANALYSIS_METHODOLOGY.md`](./ANALYSIS_METHODOLOGY.md)를 참고하세요.

---

## 📁 프로젝트 구조

```
open-source-analysis/
├── README.md                    # 이 파일
├── ANALYSIS_METHODOLOGY.md      # 상세 분석 방법론
├── .analysis-config             # 분석 설정 파일
│
└── projects/                    # 분석 대상 프로젝트들
    ├── aiokafka/
    │   ├── original/            # git clone한 원본 소스
    │   └── analysis/            # 분석 문서
    │       ├── 00_SUMMARY.md    # 전체 요약
    │       └── *.md             # 컴포넌트별 분석
    │
    ├── spring-petclinic/
    │   ├── original/
    │   └── analysis/
    │
    └── <other-projects>/
        ├── original/
        └── analysis/
```

---

## 🚀 사용 방법

### 1. 새 프로젝트 분석 시작

```bash
# 프로젝트 클론
cd projects
mkdir <project-name>
cd <project-name>
git clone <repo-url> original

# 분석 폴더 생성
mkdir analysis
```

### 2. 분석 진행

**6단계 프로세스:**

1. **프로젝트 탐색** (10분): `tree`, `wc -l`로 구조 파악
2. **입구 찾기** (15분): README 예제, 메인 API 확인
3. **핵심 흐름 추적** (30분): 하나의 기능 끝까지 따라가기
4. **"왜" 찾기** (45분): commit, 이슈, 테스트 읽기
5. **대안 생각하기** (20분): "나라면 어떻게?"
6. **문서화** (30분): "문제→고민→해결" 템플릿으로 작성

### 3. 분석 문서 작성

각 컴포넌트 분석 시 다음 질문에 답하세요:

- [ ] **문제**: 이 코드가 없었다면 어떤 불편함이 있었는가?
- [ ] **대안**: 다른 방법은 없었는가? 각 장단점은?
- [ ] **선택**: 왜 이 방법을 선택했는가?
- [ ] **트레이드오프**: 무엇을 얻고 무엇을 포기했는가?
- [ ] **적용**: 비슷한 상황에서 이 패턴을 사용할 수 있는가?

---

## 📚 분석 대상 프로젝트

### 완료된 분석

- **claude-agent-sdk-python**: Claude AI Agent SDK 구현 분석
- **aiokafka**: AsyncIO 기반 Kafka 클라이언트
- **fastkafka**: FastAPI 스타일 Kafka 프레임워크

### 진행 중

- **spring-petclinic**: Spring Boot 예제 애플리케이션

### 계획 중

- **dbt-jaffle-shop**: dbt 데이터 변환 예제
- **confluent-kafka-python**: Confluent Kafka Python 클라이언트
- **fastapi-best-practices**: FastAPI 베스트 프랙티스

---

## 💡 분석 팁

### 1. "왜"를 3번 물어보기

```
Q: 왜 이 클래스를 만들었는가?
A: 메시지를 모으기 위해

Q: 왜 메시지를 모아야 하는가?
A: 배치로 보내면 효율적이니까

Q: 왜 배치가 효율적인가?
A: 네트워크 왕복 비용이 크기 때문에 한 번에 여러 개 보내는 게 빠름

→ 진짜 문제: 네트워크 I/O 최적화
```

### 2. 테스트로 의도 파악하기

테스트 코드는 "이 코드가 해결하려는 문제"를 명확히 보여줍니다.

```python
def test_connection_pool_reuses_connection():
    """문제: 매번 새 연결 생성은 느림"""
    pool = ConnectionPool()
    conn1 = pool.get()
    pool.release(conn1)
    conn2 = pool.get()
    assert conn1 is conn2  # 해결: 연결 재사용
```

### 3. 설정(Config)으로 트레이드오프 파악하기

설정값은 사용자가 선택할 수 있는 트레이드오프를 보여줍니다.

```python
class Producer:
    def __init__(
        self,
        linger_ms=0,           # 고민: 지연 vs 처리량
        batch_size=16384,      # 고민: 메모리 vs 효율
        max_in_flight=5,       # 고민: 순서 보장 vs 동시성
    ):
        pass
```

### 4. 에러 처리로 고민 파악하기

에러 처리는 "어떤 케이스를 고민했는지" 보여줍니다.

### 5. 주석보다 코드 구조 보기

코드의 구조와 분리 방식이 설계 의도를 가장 잘 드러냅니다.

---

## 🎓 학습 리소스

### 분석 방법론
- [`ANALYSIS_METHODOLOGY.md`](./ANALYSIS_METHODOLOGY.md): 전체 분석 방법론 가이드
- [`.analysis-config`](./.analysis-config): 분석 설정 및 템플릿

### 프로젝트별 분석
- [`projects/aiokafka/analysis/`](./projects/aiokafka/analysis/): Kafka 클라이언트 분석
- [`projects/claude-agent-sdk-python/analysis/`](./projects/claude-agent-sdk-python/analysis/): AI Agent SDK 분석
- [`projects/spring-petclinic/analysis/`](./projects/spring-petclinic/analysis/): Spring Boot 애플리케이션 분석

---

## 📖 분석 예시

### 예시: Connection Pool 패턴

#### 문제
- 매번 TCP 연결 생성은 비용이 크다 (3-way handshake, 수 ms)
- 초당 1000개 메시지 전송 시 매우 느림

#### 고민했던 선택지

1. **매번 새 연결**: 간단하지만 느림 ❌
2. **하나의 연결 재사용**: 빠르지만 동시 요청 불가 ❌
3. **Connection Pool**: 복잡하지만 성능과 동시성 모두 확보 ✅

#### 최종 해결책

```python
pool = ConnectionPool(min_size=1, max_size=10)

async def send(msg):
    conn = await pool.acquire()
    try:
        await conn.send(msg)
    finally:
        await pool.release(conn)
```

**트레이드오프:**
- 얻은 것: 빠른 성능 + 동시 요청 가능
- 포기한 것: 간단함, 메모리 사용량 증가

#### 배운 점
- 네트워크 프로그래밍에서는 연결 재사용이 중요
- Pool 패턴은 리소스 재사용 + 동시성 제어에 유용
- 설정값(min/max)으로 트레이드오프 조절 가능

---

## ⚙️ 설정

### 기본 분석 방법론

이 프로젝트는 자동으로 "문제 해결 중심" 방법론을 사용합니다.
설정은 [`.analysis-config`](./.analysis-config) 파일에서 관리됩니다.

### 분석 레벨

5가지 레벨에서 질문합니다:

1. **폴더/모듈 구조**: 왜 이렇게 나눴는가?
2. **클래스/파일 분리**: 왜 여러 클래스로 나눴는가?
3. **함수/메서드 설계**: 왜 이런 시그니처인가?
4. **디자인 패턴**: 왜 이 패턴을 선택했는가?
5. **아키텍처 전체**: 왜 이런 계층 구조인가?

---

## 🤝 기여

이 프로젝트는 개인 학습용 저장소입니다.

---

## 📝 라이선스

학습 목적의 개인 프로젝트입니다.
분석 대상 프로젝트들은 각각의 라이선스를 따릅니다.

---

## 💬 철학

> "좋은 코드는 문제를 명확히 이해하고, 신중하게 선택하고, 트레이드오프를 수용한 결과입니다"

단순히 코드를 읽지 말고, **개발자가 했던 고민을 따라가 보세요**.
그 과정에서 진짜 문제 해결력이 자랍니다.

---

**Last Updated**: 2025-10-29
