# Open Source Analysis Project Structure

> **분석 철학**: 프로덕션 검증된 오픈소스에서 실전 문제 해결 방법을 배운다

**최종 수정**: 2025-01-18

---

## 📋 개요

이 레포지토리는 **잘 만들어진 오픈소스 프로젝트를 분석**하여 실전 패턴과 문제 해결 방법을 학습하는 공간입니다.

**핵심 목표**:
- ❌ 프레임워크 내부 구조 암기
- ✅ **"어떤 문제를 어떻게 해결했는가"** 학습
- ✅ 재사용 가능한 패턴 추출
- ✅ 실무에 바로 적용

---

## 🗂️ 디렉토리 구조

```
open-source-analysis/
├── README.md                  # 레포 전체 개요
├── PROJECT_STRUCTURE.md       # 이 문서
├── ANALYSIS_APPROACH.md       # 분석 방법론 (필독!)
├── .gitignore
│
├── projects/                  # 분석한 프로젝트들
│   ├── claude-agent-sdk-python/      # 완료 (기존 방식)
│   │   ├── _README.md
│   │   ├── _architecture.md
│   │   └── ...
│   │
│   ├── kafka-streams-example/        # 예시 (새 방식)
│   │   ├── README.md                # 분석 요약
│   │   ├── original/                # (선택) Fork/Clone
│   │   └── analysis/
│   │       ├── 01-overview.md
│   │       ├── 02-architecture.md
│   │       ├── 03-problems-solved.md   # 핵심!
│   │       ├── 04-key-patterns.md
│   │       └── 05-apply-to-my-work.md
│   │
│   └── [next-project]/
│
└── learning-notes/            # (선택) 공통 패턴 추출
    ├── kafka-patterns.md
    ├── error-handling.md
    └── performance-optimization.md
```

---

## 📝 분석 템플릿

### 새 프로젝트 시작

```bash
cd projects
mkdir project-name
cd project-name
mkdir analysis
```

### 필수 파일

#### 1. `README.md` (프로젝트 루트)
```markdown
# [프로젝트명] 분석

## 원본 프로젝트
- GitHub: [링크]
- 별: [숫자]
- 목적: [한 문장]

## 분석 개요
- 분석 기간: 2025-01-XX ~ 2025-01-XX
- 분석 깊이: 레벨 2 (핵심 로직)
- 배운 핵심 패턴: 3가지

## 핵심 발견 사항
1. [패턴/문제해결 1]
2. [패턴/문제해결 2]
3. [패턴/문제해결 3]

## 실전 적용
- [내 프로젝트에 적용 가능한 부분]

## 분석 문서
1. [개요](./analysis/01-overview.md)
2. [아키텍처](./analysis/02-architecture.md)
3. [문제 해결](./analysis/03-problems-solved.md) ⭐ 핵심
4. [핵심 패턴](./analysis/04-key-patterns.md)
5. [실전 적용](./analysis/05-apply-to-my-work.md)
```

#### 2. `analysis/01-overview.md`
```markdown
# 프로젝트 개요

## 기본 정보
- **원본**: [GitHub 링크]
- **목적**: [구체적으로]
- **기술 스택**: [나열]
- **규모**: [사용자 수, 데이터 처리량 등]

## 해결하는 문제
[구체적으로 어떤 문제를 해결하려고 만들어졌나]

## 왜 이 프로젝트를 선택했나
[학습 목표와 연결]
```

#### 3. `analysis/02-architecture.md`
```markdown
# 아키텍처 분석

## 전체 구조
[Mermaid 다이어그램 또는 ASCII Art]

## 주요 컴포넌트
### 컴포넌트 1
- **역할**:
- **위치**: `src/...`
- **핵심 로직**: [설명]

## 데이터 흐름
1. Input → Processing → Output
```

#### 4. `analysis/03-problems-solved.md` ⭐ **가장 중요!**
```markdown
# 해결한 문제들

## 문제 1: [제목]

### 배경
- [왜 이 문제가 발생했나]

### 시도한 방법들
1. ❌ [실패한 방법 1] → [왜 안 됐나]
2. ✅ [성공한 방법] → [왜 됐나]

### 최종 해결책
**코드 위치**: `src/...`
```code
[핵심 코드]
```

**결과**:
- Before: [수치]
- After: [수치]
- 개선율: [%]

### 트레이드오프
- 장점: [...]
- 단점: [...]

### 배운 점
- [핵심 인사이트]
```

#### 5. `analysis/04-key-patterns.md`
```markdown
# 핵심 패턴 추출

## 패턴 1: [패턴명]

### 문제
[어떤 문제를 해결하나]

### 패턴
```code
[재사용 가능한 코드 패턴]
```

### 언제 사용?
- [사용 시나리오]

### 장단점
- ✅ 장점:
- ❌ 단점:

### 대안
- [다른 방법과 비교]
```

#### 6. `analysis/05-apply-to-my-work.md`
```markdown
# 실전 적용 계획

## 즉시 적용 가능
### 패턴 1: [패턴명]
- **적용 대상**: [내 프로젝트/업무]
- **방법**: [구체적으로]
- **예상 효과**: [측정 가능하게]

## 향후 학습 필요
- [아직 이해 안 된 부분]
- [추가 학습 계획]

## Mini Project 아이디어
- [패턴을 직접 구현해볼 작은 프로젝트]
```

---

## 🏷️ 네이밍 컨벤션

### 디렉토리명
```
✅ 좋은 예시:
- kafka-streams-fraud-detection
- fastapi-production-template
- spark-optimization-case

❌ 나쁜 예시:
- project1 (의미 없음)
- kafka_example (언더스코어)
- ksfrd (약어)
```

### 파일명

**새 방식 (권장)**:
```
analysis/
├── 01-overview.md
├── 02-architecture.md
├── 03-problems-solved.md
├── 04-key-patterns.md
└── 05-apply-to-my-work.md
```

**기존 방식 (claude-agent-sdk-python)**:
```
_README.md
_architecture.md
_design_patterns.md
```
→ 이미 완료된 프로젝트는 유지, 새 프로젝트는 새 방식 사용

---

## 📊 분석 깊이 레벨

### 레벨 1: 빠른 훑기 (1-2일)
- README + 블로그만 읽기
- 주요 문제 & 해결책만 파악
- 산출물: `README.md` + `01-overview.md`

### 레벨 2: 핵심 분석 (1주) ⭐ **권장**
- 주요 로직 코드 읽기
- 문제 해결 방법 상세 분석
- 패턴 2-3개 추출
- 산출물: 모든 analysis 문서

### 레벨 3: 전체 분석 (2-3주)
- 전체 코드베이스 읽기
- 모든 패턴 추출
- 기여 가능한 수준
- 산출물: 상세 문서 + PR

---

## ✅ 체크리스트

### 분석 시작 전
- [ ] ANALYSIS_APPROACH.md 읽기
- [ ] 분석 깊이 레벨 결정
- [ ] 예상 소요 시간 설정 (1주? 2주?)
- [ ] 배경 지식 확인 (books/ 가이드 참고)

### 분석 중
- [ ] Phase별로 문서 작성
- [ ] "왜?"를 계속 질문하며 읽기
- [ ] 코드 위치 메모 (`src/...`)
- [ ] 수치로 결과 정리 (Before/After)

### 분석 완료 후
- [ ] README.md 작성
- [ ] 핵심 패턴 3개 이상 추출
- [ ] 실전 적용 계획 수립
- [ ] Git commit

---

## 🎯 현재 상태

### 완료된 분석
1. **claude-agent-sdk-python**
   - 깊이: 레벨 3 (전체 분석)
   - 기간: 2주
   - 패턴: 9개 추출
   - 방식: 기존 (아키텍처 중심)

### 진행 중
- 없음

### 계획 중
- [ ] 첫 데이터 엔지니어링 프로젝트 선정
- [ ] Kafka 또는 Spark 관련
- [ ] 새 방식 (문제 해결 중심) 적용

---

## 📚 관련 문서

- [ANALYSIS_APPROACH.md](./ANALYSIS_APPROACH.md) - 분석 방법론 (필독!)
- [books/CAREER_DEVELOPMENT_PLAN.md](../books/CAREER_DEVELOPMENT_PLAN.md) - 커리어 계획
- [books/TODO_TOPICS.md](../books/TODO_TOPICS.md) - 완성된 가이드 목록

---

## 💡 핵심 원칙

1. **문제 해결 중심**: 코드보다 "왜"에 집중
2. **실용성**: 내 프로젝트에 적용 가능한가?
3. **측정 가능성**: Before/After 수치로 정리
4. **재사용성**: 패턴으로 추출
5. **완벽주의 경계**: 레벨 2로 충분

---

## 📊 .gitignore 설정

```gitignore
# 원본 프로젝트 소스 (선택적으로 제외)
*/original/

# 일반적인 제외 항목
.DS_Store
*.pyc
__pycache__/
node_modules/
.vscode/
.idea/
.claude/
```

---

**마지막 업데이트**: 2025-01-18
