# 책/주제 정리 가이드

## 📋 개요
이 문서는 책이나 기술 주제를 체계적으로 정리하는 방법을 설명합니다.

## 🗂️ 디렉토리 구조

### 패턴 1: README + chapters (PostgreSQL 방식)
```
주제명/
├── README.md           # 전체 개요, 목차, 학습 가이드
├── OUTLINE.md          # (선택) 상세 목차
└── chapters/
    ├── chapter-01.md
    ├── chapter-02.md
    └── ...
```

**사용 케이스:**
- 책 정리 (Refactoring, Philosophy of Software Design 등)
- 체계적인 학습 경로가 필요한 주제
- 챕터가 순차적으로 연결되는 내용

**예시:**
```
PostgreSQL/
├── README.md           # PostgreSQL 전체 가이드
└── chapters/
    ├── chapter-01.md   # Architecture
    ├── chapter-02.md   # MVCC
    └── chapter-03.md   # Query Processing
```

### 패턴 2: guides 디렉토리 (BigQuery 방식)
```
주제명/
├── README.md (선택)
└── guides/
    ├── 01-introduction.md
    ├── 02-basics.md
    └── ...
```

**사용 케이스:**
- 가이드/튜토리얼 모음
- 독립적으로 읽을 수 있는 주제들
- 레퍼런스 문서

**예시:**
```
BigQuery/
└── guides/
    ├── 01-introduction-and-architecture.md
    ├── 02-datasets-and-tables.md
    └── 03-schema-and-data-types.md
```

### 패턴 3: 복합 구조 (Designing Data-Intensive Applications)
```
주제명/
├── README.md
├── chapters/          # 본문
├── appendices/        # 부록
└── notes/            # (선택) 추가 노트
```

## 📝 README.md 작성 가이드

### 필수 섹션

1. **📖 개요** - 한 문장으로 무엇을 다루는지
2. **🎯 대상 독자** - 누구를 위한 문서인지
3. **📚 목차** - 챕터/가이드 목록과 핵심 내용
4. **🚀 빠른 시작** - 학습 순서 추천
5. **💡 핵심 개념 요약** - 주요 내용 요약

### 선택 섹션

- **🔧 실무 체크리스트** - 바로 적용 가능한 팁
- **📊 학습 로드맵** - 단계별 학습 계획
- **🎓 참고 자료** - 추가 학습 리소스
- **⚡ 빠른 참조** - 자주 사용하는 명령어/쿼리

### PostgreSQL README 예시 구조:
```markdown
# PostgreSQL: Comprehensive Guide

## 📖 개요
PostgreSQL의 내부 동작 원리부터 실무 활용까지...

## 🎯 대상 독자
- 백엔드 개발자 (초급~중급)
- PostgreSQL 내부 동작을 이해하고 싶은 개발자

## 📚 목차
### Part I: Foundations
#### [Chapter 1: Architecture](./chapters/chapter-01.md)
- Multi-process architecture
- **핵심**: Postmaster, WAL Writer

## 🚀 빠른 시작
### 권장 학습 순서
1. Chapter 1 (Architecture) → PostgreSQL 전체 구조 이해
2. Chapter 4 (Indexing) → 즉시 적용 가능

## 💡 핵심 개념 요약
### MVCC
- **t_xmin/t_xmax**: Tuple visibility 판단
```

## 📄 개별 챕터/가이드 작성 가이드

### 파일명 규칙

**chapters 방식:**
```
chapter-01.md
chapter-02.md
chapter-10.md  (두 자리 숫자)
```

**guides 방식:**
```
01-introduction.md
02-basic-concepts.md
20-advanced-topics.md
```

### 내용 구조

```markdown
# Chapter N: 제목

## N.1 첫 번째 섹션

### N.1.1 서브섹션
내용...

### N.1.2 서브섹션
내용...

## N.2 두 번째 섹션

### 예제
```코드```

---

## 요약
- 핵심 내용 1
- 핵심 내용 2
```

### BigQuery 가이드 스타일 예시:
```markdown
# Chapter 2: Datasets & Tables

## 2.1 Dataset 개요
### Dataset이란?
설명...

### Dataset의 핵심 특징
```
구조도
```

## 2.2 Dataset 생성
### 2.2.1 명명 규칙
### 2.2.2 Console에서 Dataset 생성
### 2.2.3 SQL로 Dataset 생성

---

## 요약
### Dataset
- **정의**: 테이블, 뷰, 모델을 그룹화
- **위치**: 생성 후 변경 불가
```

## 🎨 작성 스타일 가이드

### 1. 제목 번호 체계
- README: 번호 없음 (## 섹션명)
- 챕터: 챕터 번호 포함 (## 2.1, ### 2.1.1)

### 2. 코드 블록
```markdown
```sql
-- SQL 예제
SELECT * FROM table;
```

```python
# Python 예제
import library
```

```bash
# Shell 명령어
bq ls my-project:dataset
```
```

### 3. 강조 표현
- **볼드**: 중요 용어, 핵심 개념
- `코드`: 명령어, 파일명, 변수명
- ⚠️ **주의**: 경고사항
- ✅ **권장** / ❌ **비권장**: 좋은/나쁜 예시

### 4. 구조화 요소
- 테이블: 비교, 옵션 정리
- 리스트: 항목 나열
- 구조도: ASCII art로 계층 표현
- 예제: 실전 사용 코드

### 5. 섹션 구분
```markdown
---
```
주요 섹션 사이에 구분선 사용

## 📊 OUTLINE.md (선택사항)

상세 목차가 필요한 경우:

```markdown
# 책 제목 - 전체 목차

## Part I: 파트명

### Chapter 1: 챕터명
1.1 섹션
1.2 섹션
  1.2.1 서브섹션
  1.2.2 서브섹션

### Chapter 2: 챕터명
...
```

## 🏷️ 네이밍 컨벤션

### 디렉토리명
```
✅ 좋은 예시:
- PostgreSQL
- BigQuery
- Designing-Data-Intensive-Applications
- Philosophy-of-Software-Design
- Django-DRF

❌ 나쁜 예시:
- postgres (축약)
- big_query (언더스코어보다 하이픈)
- ddia (약어)
```

### 파일명
```
✅ 좋은 예시:
- README.md
- OUTLINE.md
- chapter-01.md
- 01-introduction.md
- 02-datasets-and-tables.md

❌ 나쁜 예시:
- readme.md (소문자)
- ch1.md (축약)
- introduction.md (번호 누락 - guides에서)
```

## 🔄 작업 플로우

### 1. 새 주제 시작
```bash
# 1. 디렉토리 생성
mkdir 주제명
cd 주제명

# 2. 구조 선택
# 패턴 1: 챕터 방식
mkdir chapters
touch README.md

# 패턴 2: 가이드 방식
mkdir guides

# 3. README.md 골격 작성
```

### 2. 챕터/가이드 작성
```bash
# chapters/chapter-01.md 생성
# 또는
# guides/01-introduction.md 생성
```

### 3. README.md 업데이트
- 목차에 새 챕터 링크 추가
- 핵심 개념 업데이트

### 4. Git 커밋
```bash
git add .
git commit -m "Add comprehensive [주제] guide with [N] chapters"
# 또는
git commit -m "Add [주제] chapter N: [제목]"
```

## ✅ 품질 체크리스트

### README.md
- [ ] 개요가 명확한가?
- [ ] 대상 독자가 명시되어 있는가?
- [ ] 모든 챕터가 목차에 링크되어 있는가?
- [ ] 핵심 개념이 요약되어 있는가?
- [ ] 학습 가이드/순서가 제시되어 있는가?

### 챕터/가이드
- [ ] 번호 체계가 일관적인가? (2.1, 2.1.1 등)
- [ ] 코드 예제가 포함되어 있는가?
- [ ] 중요 개념에 **볼드** 표시가 되어 있는가?
- [ ] 요약 섹션이 있는가?
- [ ] 실전 예제가 포함되어 있는가?

### 전체 구조
- [ ] 디렉토리/파일명이 컨벤션을 따르는가?
- [ ] README에서 모든 파일에 접근 가능한가?
- [ ] 학습 순서가 논리적인가?

## 📚 실제 예시

### PostgreSQL (Book 스타일)
```
PostgreSQL/
├── README.md                    # 전체 가이드, 목차, 학습 로드맵
└── chapters/
    ├── chapter-01.md           # Architecture
    ├── chapter-02.md           # MVCC
    ├── ...
    └── chapter-12.md           # HA and Backup
```

### BigQuery (Guide 스타일)
```
BigQuery/
└── guides/
    ├── 01-introduction-and-architecture.md
    ├── 02-datasets-and-tables.md
    ├── ...
    └── 22-advanced-topics.md
```

### Refactoring (Book + OUTLINE)
```
Refactoring/
├── README.md                    # 개요
├── OUTLINE.md                   # 상세 목차
└── chapters/
    ├── chapter-01.md
    └── ...
```

## 🚀 빠른 템플릿

### README.md 템플릿
```markdown
# 주제명: Comprehensive Guide

## 📖 개요
[한 문장 설명]

## 🎯 대상 독자
- [독자 1]
- [독자 2]

## 📚 목차

### [Chapter 1: 제목](./chapters/chapter-01.md)
- 내용
- **핵심**: 핵심 개념

## 🚀 빠른 시작

권장 학습 순서:
1. Chapter 1
2. Chapter 2

## 💡 핵심 개념 요약

### 개념 1
- 설명

## 🔧 실무 체크리스트

```코드/설정```

## 🎓 참고 자료
- [링크]
```

### 챕터 템플릿
```markdown
# Chapter N: 제목

## N.1 첫 번째 섹션

### N.1.1 서브섹션
내용...

```코드 예제```

### N.1.2 서브섹션
내용...

## N.2 두 번째 섹션

### 실전 예제

```코드```

---

## 요약

- 핵심 내용 1
- 핵심 내용 2
```

## 🎯 핵심 원칙

1. **일관성**: 네이밍, 번호 체계, 구조 일관성 유지
2. **접근성**: README에서 모든 내용 접근 가능
3. **실용성**: 코드 예제, 실전 팁 포함
4. **명확성**: 핵심 개념 강조, 요약 제공
5. **구조화**: 논리적 순서, 명확한 계층

---

**이 가이드 자체도 계속 개선됩니다!**
