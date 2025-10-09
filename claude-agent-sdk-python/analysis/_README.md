# claude-agent-sdk-python Analysis

**원본**: https://github.com/anthropics/claude-agent-sdk-python
**언어**: Python
**Stars**: 2,334 ⭐
**분석일**: 2025-01-09
**버전**: 0.1.1

---

## 📖 프로젝트 개요

**Claude Agent SDK for Python**은 Anthropic의 공식 Python SDK로, Claude Code와 상호작용하기 위한 프로그래밍 인터페이스를 제공합니다.

### 핵심 기능
- **간단한 쿼리 인터페이스** (`query()`): 단방향 질의-응답
- **양방향 클라이언트** (`ClaudeSDKClient`): 지속적인 대화 세션
- **커스텀 도구 지원**: Python 함수를 MCP 서버로 제공
- **훅 시스템**: Claude 에이전트 루프에 대한 프로그래밍 가능한 제어
- **타입 안전성**: 완전한 타입 힌트 및 mypy strict 모드 지원

### 주요 사용 사례
1. **AI 에이전트 통합**: 애플리케이션에 Claude 에이전트 임베딩
2. **자동화 워크플로우**: 프로그래밍 방식으로 작업 자동화
3. **커스텀 도구 개발**: Python 함수를 Claude가 사용 가능한 도구로 제공
4. **에이전트 제어**: 훅을 통한 세밀한 동작 제어

---

## 🏗️ 아키텍처 개요

### 계층 구조
```
┌─────────────────────────────────────┐
│   Public API Layer                  │
│  (query, ClaudeSDKClient, types)    │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│   Internal Implementation           │
│  (_internal/client, _internal/query)│
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│   Transport Layer                   │
│  (subprocess_cli.py)                │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│   Claude Code CLI                   │
│  (External Node.js Process)         │
└─────────────────────────────────────┘
```

### 핵심 컴포넌트
- **query()**: 단순한 비동기 쿼리 함수
- **ClaudeSDKClient**: 상태 유지 클라이언트 클래스
- **Types**: 메시지, 옵션, 블록 타입 정의
- **Transport**: Claude Code CLI와의 통신 관리
- **Message Parser**: JSON 스트림 파싱

---

## 🔍 분석 요약

### Operability (운영성) - ⭐⭐⭐⭐☆

**강점:**
- ✅ **명확한 에러 타입 계층**: `ClaudeSDKError` 기반 6가지 구체적 에러
- ✅ **비동기 스트리밍**: `AsyncIterator`로 실시간 응답 처리
- ✅ **서브프로세스 관리**: 자동 재시작, 타임아웃 처리
- ✅ **타입 안전성**: mypy strict 모드, 완전한 타입 힌트

**개선점:**
- ⚠️ 로깅 시스템이 명시적으로 보이지 않음
- ⚠️ 메트릭/모니터링 기능 부재

### Simplicity (단순성) - ⭐⭐⭐⭐⭐

**강점:**
- ✅ **최소주의 API**: `query()` 하나로 시작 가능
- ✅ **명확한 책임 분리**: Public API ↔ Internal ↔ Transport
- ✅ **일관된 네이밍**: `ClaudeAgentOptions`, `ClaudeSDKClient` 등
- ✅ **점진적 복잡도**: query() → ClaudeSDKClient → 커스텀 도구 → 훅

**특징:**
- 초보자: 3줄 코드로 시작
- 고급 사용자: 세밀한 제어 가능

### Evolvability (발전성) - ⭐⭐⭐⭐⭐

**강점:**
- ✅ **인터페이스 추상화**: Transport 레이어 교체 가능
- ✅ **확장 가능한 도구 시스템**: MCP 서버 표준 활용
- ✅ **훅 아키텍처**: 기능 추가 없이 동작 커스터마이징
- ✅ **타입 확장성**: TypedDict, Protocol 활용

**설계 결정:**
- Public API와 Internal 명확히 분리 (`_internal/`)
- External MCP + SDK MCP 혼용 가능
- 서브프로세스 기반 → 다른 Claude 구현체로 교체 가능

---

## 📊 프로젝트 통계

### 코드 구조
- **핵심 모듈**: 9개 Python 파일
- **예제**: 11개
- **테스트**: E2E 7개 + 유닛 테스트

### 의존성
```python
dependencies = [
    "anyio>=4.0.0",           # 비동기 I/O 추상화
    "typing_extensions",       # 타입 힌트 지원
    "mcp>=0.1.0",             # MCP 프로토콜
]
```

### 지원 환경
- Python 3.10+
- anyio 기반 (asyncio + trio 지원)
- Node.js (Claude Code CLI 필요)

---

## 🎯 주요 발견 사항

### 1. **계층적 추상화**
- Public API는 극도로 단순
- 복잡성은 `_internal/`에 캡슐화
- 사용자는 내부 구현 몰라도 됨

### 2. **프로세스 간 통신 (IPC) 패턴**
- Python ↔ Node.js (Claude Code CLI)
- JSON Lines 프로토콜
- 비동기 스트리밍

### 3. **MCP 프로토콜 활용**
- 표준 프로토콜로 도구 확장
- In-process SDK MCP 서버 (혁신!)
- External MCP 서버와 혼용 가능

### 4. **타입 안전성 우선 설계**
- mypy strict 모드
- TypedDict로 런타임 타입 체크 불필요
- IDE 자동완성 지원

### 5. **점진적 공개 (Progressive Disclosure)**
```python
# 레벨 1: 기본 쿼리
query(prompt="Hello")

# 레벨 2: 옵션 추가
query(prompt="Hello", options=ClaudeAgentOptions(...))

# 레벨 3: 양방향 대화
ClaudeSDKClient(...)

# 레벨 4: 커스텀 도구
create_sdk_mcp_server(tools=[...])

# 레벨 5: 훅으로 제어
hooks={"PreToolUse": [...]}
```

---

## 🔧 설계 패턴

### 1. **Facade Pattern**
- `query()`, `ClaudeSDKClient`가 복잡한 내부 구현 숨김

### 2. **Adapter Pattern**
- `subprocess_cli.py`: Node.js CLI ↔ Python 어댑터

### 3. **Factory Pattern**
- `create_sdk_mcp_server()`: MCP 서버 생성

### 4. **Iterator Pattern**
- `AsyncIterator[Message]`: 스트리밍 응답

### 5. **Hook Pattern**
- 확장 포인트 제공 (PreToolUse, PostToolUse 등)

---

## 💡 배울 점

### 1. SDK 설계 원칙
- **간단한 시작, 복잡한 옵션**
- Public/Internal 명확한 분리
- 타입 안전성 우선

### 2. 프로세스 간 통신
- JSON Lines 프로토콜
- 비동기 스트리밍
- 에러 처리 및 재시작

### 3. 확장 가능한 아키텍처
- MCP 표준 활용
- 훅 시스템
- In-process 최적화 (SDK MCP)

### 4. 개발자 경험 (DX)
- 3줄로 시작 가능
- 풍부한 예제 (11개)
- 완전한 타입 힌트

---

## 🚀 다음 분석 단계

1. **_architecture.md**: 전체 아키텍처 심층 분석
2. **types.py.md**: 타입 시스템 상세 분석
3. **client.py.md**: ClaudeSDKClient 구조
4. **query.py.md**: query() 함수 구현
5. **_internal/transport/subprocess_cli.py.md**: IPC 메커니즘
6. **_insights.md**: 발견한 패턴 및 개선점

---

**분석 작성**: Claude Code
**분석 프레임워크**: Operability, Simplicity, Evolvability
