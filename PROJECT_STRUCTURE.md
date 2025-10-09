# Open Source Analysis Project

## 🎯 프로젝트 목적

오픈소스 프로젝트를 깊이 있게 분석하여 소프트웨어 아키텍처 원칙 관점에서 학습하고, 이를 포트폴리오로 만드는 프로젝트

### 분석 관점
- **Operability (운영성)**: 로깅, 모니터링, 에러 핸들링, 구성 관리 등
- **Simplicity (단순성)**: 복잡성 관리, 추상화, 모듈화 등
- **Evolvability (발전성)**: 확장성, 변경 용이성, 유연한 설계 등

## 📁 폴더 구조

```
open-source-analysis/
│
├─ README.md                         # 전체 프로젝트 소개 및 분석 완료 목록
├─ PROJECT_STRUCTURE.md              # 이 문서 (프로젝트 구조 설명)
│
├─ {프로젝트명}/                      # 각 분석 프로젝트 (레포지토리명 사용)
│  ├─ source/                        # 원본 코드 (gitignore에 추가)
│  │  └─ (git clone 결과)
│  │
│  └─ analysis/                      # 분석 결과 (Git 커밋 대상)
│     ├─ _README.md                  # 프로젝트 개요 및 분석 요약
│     ├─ _architecture.md            # 전체 아키텍처 분석
│     ├─ _insights.md                # 발견한 패턴, 개선점, 인사이트
│     │
│     └─ {원본과 동일한 폴더 구조}/   # 원본 소스와 1:1 매칭되는 구조
│        ├─ main.py.md               # 원본: source/main.py
│        ├─ utils.py.md              # 원본: source/utils.py
│        └─ core/
│           └─ engine.py.md          # 원본: source/core/engine.py
│
├─ DeepResearch/                     # 예시 프로젝트 1
│  ├─ source/
│  └─ analysis/
│
├─ MoneyPrinterTurbo/                # 예시 프로젝트 2
│  ├─ source/
│  └─ analysis/
│
└─ claude-agent-sdk-python/          # 예시 프로젝트 3
   ├─ source/
   └─ analysis/
```

## 📝 파일명 규칙

### 분석 문서 파일명
1. **소스 파일 분석**: `{원본파일명}.md`
   - `application.js` → `application.js.md`
   - `parser.py` → `parser.py.md`
   - **중요**: 원본 파일과 동일한 경로에 위치

2. **메타 문서**: `_{이름}.md` (언더스코어로 구분)
   - `_README.md` - 프로젝트 개요 및 분석 요약
   - `_architecture.md` - 전체 아키텍처 분석
   - `_insights.md` - 발견한 패턴 및 개선점

### 경로 매핑 예시
```
원본: DeepResearch/source/src/agent/retriever.py
분석: DeepResearch/analysis/src/agent/retriever.py.md
```

## 📋 분석 문서 템플릿

### _README.md 템플릿
```markdown
# {프로젝트명} Analysis

**원본**: {GitHub URL}
**언어**: {주 사용 언어}
**Stars**: {스타 수}
**분석일**: {분석 날짜}

## 프로젝트 개요
{프로젝트 설명}

## 분석 요약
{3가지 원칙 관점에서 간단한 요약}

## 주요 발견 사항
- {발견 1}
- {발견 2}
```

### 개별 파일 분석 템플릿
```markdown
# {파일 경로}

**원본 경로**: `source/{경로}/{파일명}`
**역할**: {파일의 주요 책임}

---

## 📊 구조 개요
- **클래스**: {개수} ({클래스명 나열})
- **함수**: {개수}
- **의존성**: {개수}개 모듈

## 🔍 상세 분석

### Class: {클래스명}
**책임**: {클래스의 주요 책임}

#### Methods:
- `method1()` - {역할}
  - 역할: {상세 설명}
  - 패턴: {사용된 디자인 패턴}
  - 복잡도: {평가}

### Function: {함수명}
- **입력**: {파라미터}
- **출력**: {반환값}
- **역할**: {설명}

## 🔗 의존성
- `{파일명}` - {관계 설명}
- `{모듈명}` - {관계 설명}

## 💡 설계 평가

### Operability (운영성)
- ✅/⚠️/❌ {평가 및 이유}

### Simplicity (단순성)
- ✅/⚠️/❌ {평가 및 이유}

### Evolvability (발전성)
- ✅/⚠️/❌ {평가 및 이유}

## 🔧 개선 제안
- {제안 1}
- {제안 2}
```

## 🚀 워크플로우

### 1. 새 프로젝트 추가
```bash
# 프로젝트 폴더 생성
mkdir {프로젝트명}
cd {프로젝트명}

# 원본 코드 클론
git clone {GitHub URL} source

# 분석 폴더 생성 (Claude가 자동으로 생성)
mkdir analysis
```

### 2. 분석 요청
```
"{프로젝트명}/source 프로젝트를 Operability, Simplicity, Evolvability 관점에서 분석하고,
각 주요 파일별로 클래스/함수 단위까지 상세 분석해서
{프로젝트명}/analysis/에 문서 생성해줘.
파일 구조는 원본과 동일하게, 각 파일마다 .md로 분석"
```

### 3. 분석 단계
Claude가 수행하는 작업:
1. `source/` 폴더 구조 탐색 (Glob)
2. 주요 파일 읽기 (Read)
3. 전체 아키텍처 분석 → `_architecture.md` 생성 (Write)
4. 각 파일별 상세 분석 → `{파일명}.md` 생성 (Write)
5. 인사이트 정리 → `_insights.md` 생성 (Write)
6. 프로젝트 요약 → `_README.md` 생성 (Write)

### 4. Git 커밋
```bash
# 최상위 폴더로 이동
cd /home/junhyun/open-source-analysis

# 분석 결과만 커밋 (source/는 gitignore)
git add {프로젝트명}/analysis/
git commit -m "Add {프로젝트명} analysis"
git push
```

## 🎯 분석 깊이 레벨

```
레벨 1: 프로젝트 전체 아키텍처
  ↓
레벨 2: 폴더/모듈 구조
  ↓
레벨 3: 파일별 분석
  ↓
레벨 4: 클래스/함수 단위 상세 분석 ← 목표!
```

## 🔍 분석 항목

### 전체 아키텍처 (_architecture.md)
1. 전체 폴더 및 파일 구조
2. 계층별 책임과 역할
3. 모듈 간 의존성 관계
4. 주요 디자인 패턴
5. 아키텍처 스타일 (MVC, Layered, etc.)

### 파일별 분석 ({파일명}.md)
1. 파일의 주요 책임 (Single Responsibility 관점)
2. 포함된 클래스/함수 목록 및 기능
3. 각 클래스의 메서드별 역할
4. 내부/외부 의존 관계
5. 3가지 원칙 관점 평가
6. 리팩터링 제안

### 인사이트 (_insights.md)
1. 발견한 디자인 패턴
2. 잘 설계된 부분
3. 개선이 필요한 부분
4. 학습 포인트
5. 다른 프로젝트에 적용 가능한 아이디어

## 📊 .gitignore 설정

```gitignore
# 원본 프로젝트 소스 (용량 문제로 제외)
*/source/

# 일반적인 제외 항목
.DS_Store
*.pyc
__pycache__/
node_modules/
.vscode/
.idea/
```

## ✅ 주요 원칙

1. **원본과 분석 완전 분리**
   - 원본 코드는 읽기 전용
   - 분석 결과만 Git으로 관리

2. **1:1 경로 매핑**
   - 원본 파일 경로와 분석 문서 경로 동일하게 유지
   - 파일명에 `.md` 확장자만 추가

3. **체계적 문서화**
   - 템플릿 기반 일관된 형식
   - 3가지 원칙 관점 필수 포함

4. **확장 가능성**
   - 프로젝트 100개 추가해도 구조 유지
   - 각 프로젝트 완전 독립

## 🎨 포트폴리오 가치

이 프로젝트를 통해 보여줄 수 있는 것:
- ✅ 대규모 코드베이스 분석 능력
- ✅ 소프트웨어 아키텍처 원칙 이해
- ✅ 디자인 패턴 식별 능력
- ✅ 코드 품질 평가 능력
- ✅ 체계적 문서화 능력
- ✅ 발전 가능성 있는 구조 설계 능력

## 📚 분석 대상 후보 프로젝트

- Alibaba-NLP/DeepResearch (RAG Agent)
- harry0703/MoneyPrinterTurbo (AI Video)
- CorentinJ/Real-Time-Voice-Cloning (Voice Clone)
- anthropics/claude-agent-sdk-python (LLM SDK)
- HKUDS/AutoAgent (LLM Agent Framework)
- TheAlgorithms/Python (Algorithm Collection)
- microsoft/BitNet (1-bit LLM)
- NVIDIA/garak (LLM Security Scanner)

---

**작성일**: 2025-01-09
**최종 수정**: 2025-01-09
