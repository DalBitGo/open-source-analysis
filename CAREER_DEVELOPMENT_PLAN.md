# 커리어 개발 계획 & 논의 내용

> 기술 가이드 학습 → 실무 경험 전환 전략

**작성일**: 2025-01-16
**목적**: 기술 가이드 학습을 넘어 실제 이직/취업에 도움되는 커리어 구축

---

## 📋 핵심 문제 인식

### 1. 이력서/경력의 현실

**면접관이 보는 순서:**

1. **프로젝트 경력**
   - ❌ 중소기업 끼워팔기 솔루션 → 관심 없음
   - ❌ "넥사크로, 알티베이스, 큐브리드" → SI 수렁, 커리어 도움 안 됨
   - ✅ 필요한 것: "어떤 문제를 어떻게 해결했는가"

2. **기술 스택 & 깊이**
   - ❌ "Django, Spring 써봤어요" → 피상적
   - ✅ "Django ORM N+1 문제를 select_related로 해결했고, 쿼리 개수 100개→5개로 줄였습니다"

3. **포트폴리오 / 문제해결 능력**
   - ❌ "CRUD 구현했어요"
   - ✅ "동시접속 1만명 처리 위해 Redis 캐싱 도입, API 응답 3초→100ms"

### 2. 현재 가이드의 한계

**가이드가 제공하는 것:**
- ✅ 기술 지식: Django, Spring, Kafka, Kubernetes 등
- ✅ 도구 사용법: "어떻게 쓰는가"
- ✅ 개념 이해: 아키텍처, 패턴

**가이드가 제공하지 못하는 것:**
- ❌ **실무 문제해결 경험**
- ❌ "왜 이 기술을 선택했는가?" (의사결정)
- ❌ "어떤 문제가 있었고 어떻게 해결했는가?" (스토리)
- ❌ "트레이드오프는 무엇인가?" (장단점)
- ❌ "측정 가능한 결과" (개선 전후 수치)

### 3. 면접 시나리오 비교

```
면접관: "Redis를 왜 사용했나요?"

❌ 나쁜 답변:
"가이드에서 캐싱할 때 Redis 쓴다고 배웠습니다"
→ 암기형, 주도성 없음

✅ 좋은 답변:
"API 응답시간이 3초로 느려서 병목을 분석했더니
 같은 DB 쿼리가 전체의 80%를 차지했습니다.
 Cache-aside 패턴으로 Redis를 도입했고,
 Hit ratio 85% 달성해서 응답시간 100ms로 개선했습니다.
 단, Redis 장애 시 DB 부하 급증 문제가 있어
 Circuit Breaker 패턴을 추가했습니다"
→ 문제인식 → 분석 → 해결 → 측정 → 개선
```

---

## 🎯 해결 방안

### 방안 1: 사이드 프로젝트 (최우선) ⭐⭐⭐⭐⭐

**왜 가장 효과적인가?**
- 처음부터 끝까지 "내 결정"이 들어감
- 문제 정의 → 기술 선택 → 구현 → 배포 전체 경험
- 면접에서 모든 질문에 대답 가능
- 측정 가능한 결과 (응답시간, 동시접속, 성능 개선)

**예시 프로젝트:**
- 실시간 주식 알림 서비스
  - WebSocket, Kafka, Redis 사용 이유 설명 가능
  - "동시접속 1만명 처리, 메시지 지연 50ms 이하 달성"
- 블로그 플랫폼
  - Django ORM 최적화 경험
  - "N+1 쿼리 문제 → select_related → 쿼리 100개→5개"
- 추천 시스템
  - 협업 필터링 구현
  - "1000만 사용자 처리 위해 Spark 배치 처리 도입"

**단점:**
- 시간이 많이 걸림
- 혼자 하기 어려움 (멘토링 필요)

### 방안 2: 오픈소스 기여 ⭐⭐⭐⭐

**왜 효과적인가?**
- 대규모 프로덕션 코드 경험
- 시니어 개발자의 코드 리뷰 받음
- "Django 프로젝트에 기여" → 신뢰도 상승
- GitHub에 영구 기록

**단계별 접근:**
1. **Level 1: 코드 읽기** (오픈소스 실전 분석 가이드 활용)
   - Django 코드 읽으며 "왜 이렇게 설계했을까?" 이해
   - 면접: "Django ORM이 Lazy Evaluation 쓰는 이유는..."

2. **Level 2: 이슈 재현**
   - Good First Issue 찾기
   - 로컬에서 재현해보기

3. **Level 3: 작은 기여**
   - 오타 수정, 문서 개선
   - 기여 기록 남김

4. **Level 4: 코드 기여**
   - 버그 수정 PR 제출
   - 코드 리뷰 받으며 학습

**단점:**
- 진입장벽 높음
- Good First Issue 찾기 어려움
- 코드 리뷰가 까다로움

### 방안 3: 오픈소스 분석 (보조적) ⭐⭐⭐

**왜 도움되는가?**
- "왜 Django는 이렇게 설계했을까?" 이해
- 면접에서 "아키텍처 설명" 가능
- 트레이드오프 학습 (성능 vs 유지보수)

**단점:**
- 직접 구현 경험은 아님
- "분석만 했다" vs "직접 구현했다" → 후자가 강함

---

## 🚀 현실적인 실행 계획

### Phase 1: 가이드 학습 + 소형 프로젝트 (1~2개월)

**목표:** 기본 기술 스택 익히기 + 간단한 구현 경험

1. **가이드 학습**
   - Django/DRF, PostgreSQL, Redis 가이드 완독
   - 각 챕터 예제 코드 직접 실행
   - 개념 이해 위주

2. **소형 프로젝트 (Claude와 함께)**
   - 프로젝트: 간단한 블로그 플랫폼
   - 요구사항:
     - 사용자 인증 (JWT)
     - 게시글 CRUD
     - 댓글 기능
     - 페이지네이션

   **진행 방식:**
   ```
   Step 1: 요구사항 정의
   - Claude: "왜 JWT를 선택하나요? Session은?"
   - 나: 답변 시도
   - Claude: 보충 설명

   Step 2: 아키텍처 설계
   - Claude: "레이어를 어떻게 나눌까요?"
   - 나: 설계 시도
   - Claude: 피드백 & 개선

   Step 3: 구현
   - Claude: 기본 코드 작성
   - 나: 코드 이해 & 질문
   - 나: 기능 추가/수정 시도

   Step 4: 문제 해결
   - Claude: 일부러 성능 문제 넣기
   - 나: 문제 찾기 → 해결 시도
   - Claude: 힌트 & 리뷰
   ```

3. **학습 기록**
   - 블로그에 "이런 문제가 있었고 이렇게 해결" 작성
   - GitHub 커밋 메시지 상세하게

### Phase 2: 중형 프로젝트 (2~3개월)

**목표:** 실무 수준의 문제 해결 경험

**프로젝트 예시: 실시간 채팅 서비스**

**기술 스택:**
- Backend: Django + DRF
- WebSocket: Django Channels
- Message Queue: Redis Pub/Sub
- Database: PostgreSQL
- Cache: Redis
- Deployment: Docker + K8s

**해결할 문제들:**
1. **동시접속 확장성**
   - 문제: "1000명 접속 시 서버 멈춤"
   - 해결: Redis Pub/Sub + 수평 확장
   - 결과: "1만명 동시접속 처리 성공"

2. **메시지 유실 방지**
   - 문제: "서버 재시작 시 메시지 손실"
   - 해결: PostgreSQL 영구 저장 + Redis 캐싱
   - 결과: "메시지 유실률 0%"

3. **API 성능 최적화**
   - 문제: "채팅방 목록 조회 3초"
   - 해결: N+1 쿼리 발견 → select_related
   - 결과: "100ms로 개선"

**진행 방식:**
- Claude와 페어 프로그래밍
- 매 주 문제 하나씩 해결
- 해결 과정 블로그 작성

### Phase 3: 오픈소스 기여 (3개월~)

**목표:** 실제 프로덕션 코드 경험 + 신뢰도

1. **Django 프로젝트 분석**
   - 오픈소스 실전 분석 가이드 활용
   - ORM, Request/Response 흐름 완전 이해

2. **Good First Issue 찾기**
   - Django, DRF, Celery 등
   - 버그 재현 → 원인 분석

3. **PR 제출**
   - 작은 버그 수정부터
   - 코드 리뷰 받으며 학습

---

## 💡 핵심 원칙

### 1. "왜?"를 계속 물어보기

```
나: "Redis 캐싱 코드 작성했어요"
Claude: "왜 Cache-aside 패턴을 썼나요? Write-through는?"
나: "음... 잘 모르겠어요"
Claude: "Cache-aside는 읽기 위주일 때 좋고..."
```

### 2. 직접 수정/확장하기

```
Claude: "기본 채팅 앱 완성"
나: "파일 전송 기능 추가해볼게요"
→ (시도 → 에러 → 디버깅 → 해결)
→ 이게 진짜 경험!
```

### 3. 문제를 만들고 해결하기

```
Claude: "일부러 성능 문제 넣을게요"
나: "프로파일링 해보니 이 쿼리가 문제네요"
→ (해결 방법 고민 → 구현)
```

### 4. 과정을 기록하기

```
블로그 제목:
"Django 채팅 앱에서 N+1 쿼리 문제 해결하기"

내용:
1. 문제: 채팅방 목록 조회가 3초 걸림
2. 분석: django-debug-toolbar로 확인 → 100개 쿼리 발견
3. 원인: Message 모델 조회 시 User 정보 매번 쿼리
4. 해결: select_related('user') 추가
5. 결과: 쿼리 100개→5개, 응답시간 100ms
6. 교훈: ORM은 편하지만 성능 주의 필요
```

---

## 📊 진행 상황 추적

### 완료된 가이드 학습
- [ ] Django & DRF
- [ ] PostgreSQL
- [ ] Redis & 캐싱
- [ ] Kubernetes
- [ ] Apache Kafka

### 프로젝트 진행
- [ ] Phase 1: 블로그 플랫폼 (소형)
- [ ] Phase 2: 실시간 채팅 (중형)
- [ ] Phase 3: 오픈소스 기여

### 블로그 작성
- [ ] 문제 해결 사례 1:
- [ ] 문제 해결 사례 2:
- [ ] 문제 해결 사례 3:

---

## 🤔 논의 사항 & 의문점

### 2025-01-16 논의

**Q: 사이드 프로젝트를 실무적으로 할 수 있나? 구현만 해도 도움될까?**

A:
- 단순 구현만으로는 부족 (CRUD는 누구나 함)
- **문제 해결 중심**이어야 함
  - "API가 느려서 → 분석 → Redis 도입 → 85% 개선"
- Claude와 "같이" 하면 가능
  - 코드 짜주는 게 아니라 "같이 고민"
  - "왜?"를 계속 물어보며 이해

**Q: Claude에게 부탁해서 구현하는 건 의미 없나?**

A:
- ❌ 의미 없는 경우: 100% 복붙, 이해 없이 "돌아가네?" 끝
- ✅ 의미 있는 경우:
  - "왜 이렇게 했는지" 계속 질문
  - 직접 수정, 기능 추가, 버그 수정
  - 블로그에 설계 이유 정리

**Q: 오픈소스는 어떻게?**

A: 단계적 접근
1. 코드 읽기 (분석)
2. 이슈 재현
3. 문서/오타 수정
4. 코드 기여

---

## 🎯 사이드 프로젝트 개발 수준

### 2025-01-16 논의: "어느 수준까지 개발해야 하나?"

**결론: 프로덕션 레벨까지 안 해도 됩니다!**

면접관이 보는 것:
- ❌ "완벽한 배포, 모니터링, 99.9% 가용성"
- ✅ **"문제를 찾고 해결한 능력"**

### 수준별 비교

**Level 1: 튜토리얼 따라하기 (의미 없음) ❌**
```
- Django 튜토리얼 그대로 복붙
- 면접: "왜 이렇게 했나요?" → "튜토리얼에 있어서요"
→ 별로
```

**Level 2: 문제 찾고 해결 (의미 있음!) ✅**
```
- N+1 쿼리 발견 → select_related로 해결
- 쿼리 100개 → 3개, 응답시간 3초 → 100ms
- 면접: "django-debug-toolbar로 분석했고..."
→ 좋음!
```

**Level 3: 프로덕션 레벨 (좋지만 필수 아님) ⭐**
```
- AWS 배포, CI/CD, 모니터링
- 테스트 커버리지 90%+
→ 있으면 좋지만 필수는 아님
```

### 현실적인 목표

**최소 목표: "문제 3개 해결"**

```
필수:
✅ GitHub 코드 업로드
✅ README에 문제 해결 사례 3개 작성
✅ 로컬에서 실행 가능 (docker-compose up)
✅ 핵심 기능 동작

선택 (있으면 좋음):
⭐ 간단한 테스트 코드
⭐ Heroku/AWS 무료 티어 배포
⭐ 블로그 포스팅

굳이 안 해도 됨:
❌ 완벽한 보안
❌ 100% 테스트 커버리지
❌ 모니터링 대시보드
❌ Auto Scaling
❌ 실제 사용자 유치
```

### 예시: 실시간 채팅 앱

**✅ 이 정도면 충분:**
```
필수 구현:
- WebSocket으로 실시간 메시지 전송
- Redis Pub/Sub으로 서버 간 통신
- PostgreSQL에 메시지 영구 저장
- Docker로 로컬 실행 가능

문제 해결 사례 (README에 작성):
1. 문제: 서버 재시작 시 메시지 유실
   해결: PostgreSQL 영구 저장 + Redis 캐싱
   결과: 메시지 유실률 0%

2. 문제: 동시 접속 100명에서 서버 멈춤
   해결: Redis Pub/Sub로 수평 확장
   결과: 1000명 동시 접속 처리

3. 문제: 채팅방 목록 조회 3초
   해결: N+1 쿼리 → select_related
   결과: 100ms로 개선
```

**❌ 굳이 안 해도 됨:**
- 사용자 10만명 처리
- 완벽한 보안 (2FA, Rate Limiting 등)
- 모니터링 대시보드 (Grafana)
- Auto Scaling, Load Balancer

### 개발 플랜 (5주)

```
Week 1-2: 기본 구현
- 핵심 기능만 동작하게
- "동작은 함" (완벽하지 않아도 OK)

Week 3-4: 문제 찾고 해결
- django-debug-toolbar로 성능 문제 찾기
- Locust로 부하 테스트
- 3가지 문제 해결

Week 5: 정리 & 문서화
- GitHub README 작성
- 문제 해결 사례 정리
- 블로그 포스팅 (선택)
```

**핵심 메시지:**
> "완벽한 프로젝트 1개"보다
> "문제를 찾고 해결한 프로젝트 1개"가 낫다!

---

## 🚀 가이드별 추천 사이드 프로젝트

### 전략: 직무별 분류 + 난이도별 프로젝트 경로

**장점:**
- ✅ 목표 직무에 맞는 프로젝트 조합 선택
- ✅ 입문→중급→고급 단계적 학습 가능
- ✅ 포트폴리오 완성도 향상 (직무 특화)
- ✅ 면접 시 명확한 직무 타겟팅

---

## 📂 직무별 사이드 프로젝트 분류

### 🎯 1. 백엔드 엔지니어 (Backend Engineer)

**핵심 역량:**
- REST API 설계 & 구현
- 데이터베이스 최적화
- 캐싱 전략
- 동시성 처리

#### 입문 (⭐⭐, 3-4주)
**Django 블로그 플랫폼 API**
- 기술: Django + DRF, PostgreSQL, Redis
- 해결 문제:
  1. N+1 쿼리 (100개 → 3개)
  2. API 응답시간 (2초 → 50ms, Redis 캐싱)
  3. Cursor-based 페이지네이션 (10만 건 처리)
- 학습 포인트: Django ORM, DRF Serializer, 캐싱 패턴

#### 중급 (⭐⭐⭐, 4-5주)
**Spring Boot REST API 서버**
- 기술: Spring Boot 3.x, PostgreSQL, Redis
- 해결 문제:
  1. N+1 쿼리 (EntityGraph, fetch join)
  2. 동시 주문 재고 처리 (Pessimistic Lock)
  3. API 응답시간 개선 (Spring Cache)
- 학습 포인트: Spring Data JPA, 트랜잭션, 동시성 제어

#### 고급 (⭐⭐⭐⭐, 5-6주)
**마이크로서비스 아키텍처**
- 기술: Spring Cloud, Kafka, Redis, PostgreSQL
- 해결 문제:
  1. 서비스 간 통신 (API Gateway, Service Discovery)
  2. 분산 트랜잭션 (SAGA 패턴)
  3. Circuit Breaker & Retry
- 학습 포인트: MSA 패턴, 이벤트 기반 아키텍처

**추천 학습 순서:**
```
Django 블로그 → Spring Boot API → 마이크로서비스
(총 12-15주, 백엔드 포트폴리오 완성)
```

**백엔드 엔지니어 특화 추가:**

#### 선택 1 (⭐⭐⭐, 2-3주)
**Redis 캐싱 전략**
- 기술: Redis, Django/Spring Boot
- 해결 문제:
  1. API 응답시간 개선 (Cache-aside 패턴)
  2. Rate Limiting (Token Bucket)
  3. 세션 관리 (분산 환경)
- 학습 포인트: 캐싱 패턴, TTL 전략, 분산 세션

#### 선택 2 (⭐⭐⭐, 3-4주)
**GraphQL API 서버**
- 기술: Django Graphene / Spring GraphQL
- 해결 문제:
  1. Over-fetching 해결 (N+1 방지)
  2. 동적 쿼리 최적화
  3. Subscription (실시간 업데이트)
- 학습 포인트: GraphQL 스키마, Resolver, DataLoader

#### 선택 3 (⭐⭐⭐⭐, 4-5주)
**API Gateway & 인증 서버**
- 기술: Kong/Spring Cloud Gateway, Keycloak
- 해결 문제:
  1. 중앙 인증/인가 (OAuth2, JWT)
  2. Rate Limiting & Circuit Breaker
  3. API 라우팅 & 로드밸런싱
- 학습 포인트: API Gateway 패턴, OAuth2, OIDC

---

### 🎯 2. 데이터 엔지니어 (Data Engineer)

**핵심 역량:**
- 대용량 데이터 처리
- 실시간 파이프라인 구축
- ETL 자동화
- 데이터 웨어하우스

#### 입문 (⭐⭐⭐, 4-5주)
**PostgreSQL 분석 대시보드**
- 기술: Django/FastAPI, PostgreSQL, Chart.js
- 해결 문제:
  1. 느린 집계 쿼리 (30초 → 200ms, Materialized View)
  2. Bulk Insert (100만 건, 10분 → 30초)
  3. 동시 읽기/쓰기 (Partitioning)
- 학습 포인트: PostgreSQL 고급 기능, 인덱스 전략

#### 중급 (⭐⭐⭐⭐, 5-6주)
**실시간 이벤트 처리 파이프라인**
- 기술: Kafka, Kafka Streams/Flink, PostgreSQL, Redis
- 해결 문제:
  1. 대량 이벤트 처리 (초당 10만 건)
  2. 실시간 집계 (Window 연산, 지연 1초 이하)
  3. Exactly-once 보장
- 학습 포인트: Kafka Producer/Consumer, 스트림 처리

#### 고급 (⭐⭐⭐⭐, 6-8주)
**대용량 데이터 분석 플랫폼**
- 기술: Apache Spark, Airflow, S3, Athena/BigQuery
- 해결 문제:
  1. Spark Join 최적화 (Broadcast, Skew 해결)
  2. ETL 파이프라인 자동화 (Airflow DAG)
  3. 데이터 품질 모니터링 (Great Expectations)
- 학습 포인트: Spark 튜닝, Airflow, Data Lake

**추천 학습 순서:**
```
PostgreSQL 대시보드 → Kafka 이벤트 처리 → Spark 분석 플랫폼
(총 15-19주, 데이터 엔지니어링 포트폴리오 완성)
```

**데이터 엔지니어 특화 추가:**

#### 선택 1 (⭐⭐⭐, 2-3주)
**Redis 실시간 순위 시스템**
- 기술: Redis (Sorted Set, Hash), Django/FastAPI
- 해결 문제:
  1. 순위 조회 성능 (5초 → 5ms, Sorted Set)
  2. 동시 카운터 업데이트 (INCR, Atomic)
  3. 캐시 무효화 전략 (Write-through + TTL)
- 학습 포인트: Redis 자료구조, 캐싱 패턴, 동시성 제어

#### 선택 2 (⭐⭐⭐, 3-4주)
**Elasticsearch 로그 분석 시스템**
- 기술: Elasticsearch, Logstash, Kibana, Filebeat
- 해결 문제:
  1. 대용량 로그 검색 (전문 검색, Nori Analyzer)
  2. 실시간 로그 수집 (Logstash → Elasticsearch)
  3. 대시보드 구축 (Kibana, Aggregation)
- 학습 포인트: ELK Stack, 전문 검색, 로그 파이프라인

#### 선택 3 (⭐⭐⭐⭐, 4-5주)
**Airflow + dbt 데이터 변환 파이프라인**
- 기술: Apache Airflow, dbt, PostgreSQL/BigQuery
- 해결 문제:
  1. 복잡한 ETL 자동화 (DAG 설계, 의존성 관리)
  2. 데이터 품질 검증 (dbt tests)
  3. 증분 업데이트 (Incremental models)
- 학습 포인트: Airflow DAG, dbt 모델링, 데이터 품질

---

### 🎯 3. DevOps/SRE 엔지니어

**핵심 역량:**
- 컨테이너 오케스트레이션
- CI/CD 파이프라인
- 모니터링 & 알림
- 무중단 배포

#### 입문 (⭐⭐, 3-4주)
**Django 블로그 + Docker 배포**
- 기술: Django, Docker, docker-compose
- 해결 문제:
  1. 멀티스테이지 빌드 (이미지 크기 50% 감소)
  2. Health Check & 재시작 정책
  3. 환경 변수 관리 (.env)
- 학습 포인트: Docker 기초, docker-compose

#### 중급 (⭐⭐⭐⭐, 6-8주)
**Kubernetes 마이크로서비스 배포**
- 기술: Kubernetes, Istio, Prometheus, Grafana
- 해결 문제:
  1. 무중단 배포 (Rolling Update, Readiness Probe)
  2. Auto Scaling (HPA, CPU 70% 이상 시)
  3. Circuit Breaker (Istio, 장애 격리)
- 학습 포인트: K8s 리소스, Service Mesh, Monitoring

#### 고급 (⭐⭐⭐⭐, 6-8주)
**CI/CD & GitOps 플랫폼**
- 기술: GitHub Actions, ArgoCD, Terraform, AWS/GCP
- 해결 문제:
  1. 자동화 파이프라인 (테스트 → 빌드 → 배포)
  2. GitOps 배포 (ArgoCD, Git 기반 배포)
  3. 인프라 코드화 (Terraform, 환경 재현성)
- 학습 포인트: CI/CD, GitOps, IaC

**추천 학습 순서:**
```
Docker 배포 → Kubernetes 배포 → CI/CD & GitOps
(총 15-20주, DevOps 포트폴리오 완성)
```

**DevOps/SRE 특화 추가:**

#### 선택 1 (⭐⭐⭐, 3-4주)
**로그 수집 & 모니터링 파이프라인**
- 기술: Fluentd/Filebeat, Elasticsearch, Kibana, Prometheus, Grafana
- 해결 문제:
  1. 대용량 로그 수집 (수집→파싱→저장)
  2. 실시간 알림 (Alertmanager)
  3. 커스텀 대시보드 (Grafana)
- 학습 포인트: ELK Stack, Prometheus, 메트릭 vs 로그

#### 선택 2 (⭐⭐⭐⭐, 4-5주)
**Infrastructure as Code (Terraform)**
- 기술: Terraform, AWS/GCP, Ansible
- 해결 문제:
  1. 인프라 재현성 (환경 복제 10분 이내)
  2. 상태 관리 (Remote State, Locking)
  3. 멀티 클라우드 배포
- 학습 포인트: Terraform 모듈, State 관리, Cloud 리소스

#### 선택 3 (⭐⭐⭐⭐, 4-5주)
**장애 대응 & SRE 실습**
- 기술: Chaos Engineering (Chaos Mesh), SLO/SLI 설정
- 해결 문제:
  1. 장애 시뮬레이션 (Pod 강제 종료, 네트워크 지연)
  2. 자동 복구 (Self-healing)
  3. SLO 달성 (99.9% 가용성)
- 학습 포인트: Chaos Engineering, SRE 원칙, Incident Response

---

### 🎯 4. AI/ML 엔지니어

**핵심 역량:**
- LLM 애플리케이션 개발
- 모델 서빙
- 실험 추적 & 재현성
- 모델 모니터링

#### 입문 (⭐⭐⭐, 4-5주)
**RAG 기반 AI 챗봇**
- 기술: OpenAI/Claude API, ChromaDB, FastAPI
- 해결 문제:
  1. RAG 검색 정확도 (60% → 90%, Reranking)
  2. LLM API 비용 절감 (70%, Prompt Caching)
  3. 응답 시간 단축 (Streaming, 첫 토큰 1초 이내)
- 학습 포인트: RAG 아키텍처, Vector DB, Prompt Engineering

#### 중급 (⭐⭐⭐⭐, 6-8주)
**MLOps 모델 서빙 시스템**
- 기술: MLflow, FastAPI/TorchServe, Airflow
- 해결 문제:
  1. 추론 레이턴시 (5초 → 100ms, 모델 캐싱)
  2. Model Drift Detection (Evidently, 자동 알림)
  3. 실험 재현성 (MLflow + DVC)
- 학습 포인트: MLflow, 모델 서빙, A/B Testing

#### 고급 (⭐⭐⭐⭐, 6-8주)
**End-to-End ML 파이프라인**
- 기술: Kubeflow, Airflow, Spark, MLflow
- 해결 문제:
  1. 분산 학습 (Horovod, 학습 시간 50% 단축)
  2. Feature Store (Feast, 재사용성 향상)
  3. 모델 A/B Testing (Canary Deployment)
- 학습 포인트: 분산 학습, Feature Store, MLOps 파이프라인

**추천 학습 순서:**
```
RAG 챗봇 → MLOps 서빙 → End-to-End ML 파이프라인
(총 16-21주, AI/ML 포트폴리오 완성)
```

**AI/ML 엔지니어 특화 추가:**

#### 선택 1 (⭐⭐⭐, 3-4주)
**LLM Fine-tuning & Evaluation**
- 기술: Hugging Face, LoRA, PEFT, OpenAI Fine-tuning
- 해결 문제:
  1. 도메인 특화 모델 (Fine-tuning 정확도 향상)
  2. 비용 절감 (LoRA로 GPU 메모리 50% 감소)
  3. 모델 평가 (BLEU, ROUGE, Human Eval)
- 학습 포인트: Fine-tuning 전략, LoRA, 모델 평가

#### 선택 2 (⭐⭐⭐⭐, 4-5주)
**추천 시스템**
- 기술: Spark MLlib, TensorFlow Recommenders, Redis
- 해결 문제:
  1. 협업 필터링 (Cold Start 문제 해결)
  2. 실시간 추천 (추론 레이턴시 50ms 이하)
  3. A/B Testing (CTR 20% 향상)
- 학습 포인트: 협업 필터링, 컨텐츠 기반, Hybrid 추천

#### 선택 3 (⭐⭐⭐⭐, 4-5주)
**컴퓨터 비전 파이프라인**
- 기술: PyTorch, YOLO/EfficientDet, TorchServe
- 해결 문제:
  1. 객체 탐지 (Real-time, 30 FPS)
  2. 모델 경량화 (Quantization, Pruning)
  3. Edge 배포 (ONNX, TensorRT)
- 학습 포인트: Object Detection, 모델 최적화, Edge AI

---

### 🎯 5. 풀스택 엔지니어 (Full-Stack Engineer)

**핵심 역량:**
- 백엔드 API 개발
- 프론트엔드 UI 개발
- 전체 시스템 설계
- 배포 & 운영

#### 입문 (⭐⭐, 4-5주)
**Django + React 블로그 플랫폼**
- 기술: Django + DRF (백엔드), React + Next.js (프론트)
- 해결 문제:
  1. N+1 쿼리 최적화 (백엔드)
  2. 렌더링 최적화 (React.memo, useMemo)
  3. SEO 최적화 (Next.js SSR)
- 학습 포인트: REST API 연동, SSR, 상태 관리

#### 중급 (⭐⭐⭐, 5-6주)
**실시간 채팅 애플리케이션**
- 기술: Django Channels (WebSocket), Redis, React
- 해결 문제:
  1. 동시접속 확장성 (Redis Pub/Sub, 1000명 처리)
  2. 메시지 유실 방지 (PostgreSQL 영구 저장)
  3. 실시간 업데이트 (WebSocket)
- 학습 포인트: WebSocket, Redis Pub/Sub, 실시간 통신

#### 고급 (⭐⭐⭐⭐, 6-8주)
**이커머스 풀스택 플랫폼**
- 기술: Spring Boot (백엔드), Next.js (프론트), K8s (배포)
- 해결 문제:
  1. 동시 주문 재고 처리 (Pessimistic Lock)
  2. 결제 트랜잭션 (SAGA 패턴)
  3. 무중단 배포 (K8s Rolling Update)
- 학습 포인트: 분산 트랜잭션, MSA, K8s

**추천 학습 순서:**
```
Django + React 블로그 → 실시간 채팅 → 이커머스 플랫폼
(총 15-19주, 풀스택 포트폴리오 완성)
```

**풀스택 엔지니어 특화 추가:**

#### 선택 1 (⭐⭐⭐, 3-4주)
**모바일 앱 (React Native / Flutter)**
- 기술: React Native, Django/Spring 백엔드
- 해결 문제:
  1. 크로스 플랫폼 (iOS + Android 동시 배포)
  2. 오프라인 동기화 (AsyncStorage)
  3. Push 알림 (FCM)
- 학습 포인트: React Native, 네이티브 모듈, 모바일 최적화

#### 선택 2 (⭐⭐⭐, 3-4주)
**관리자 대시보드**
- 기술: React Admin / Next.js, Django/Spring
- 해결 문제:
  1. CRUD 자동화 (RESTful Resource)
  2. 권한 관리 (RBAC)
  3. 대용량 데이터 테이블 (가상 스크롤링)
- 학습 포인트: Admin UI, 권한 관리, 데이터 테이블 최적화

#### 선택 3 (⭐⭐⭐⭐, 4-5주)
**실시간 협업 도구**
- 기술: Next.js, WebSocket, CRDT (Yjs), Redis
- 해결 문제:
  1. 실시간 동시 편집 (Conflict Resolution)
  2. 동시 사용자 100명 처리
  3. Offline-first (로컬 변경 동기화)
- 학습 포인트: CRDT, Operational Transform, 동시성 제어

---

## 🗺️ 프로젝트 로드맵 (직무별)

### 백엔드 엔지니어 로드맵 (12-15주)
```
Week 1-4:   Django 블로그 (입문)
Week 5-9:   Spring Boot API (중급)
Week 10-15: 마이크로서비스 (고급)

결과: 3개 프로젝트, 9가지 문제 해결 사례
```

### 데이터 엔지니어 로드맵 (15-19주)
```
Week 1-5:   PostgreSQL 대시보드 (입문)
Week 6-11:  Kafka 이벤트 처리 (중급)
Week 12-19: Spark 분석 플랫폼 (고급)
선택: Redis 순위 시스템 (+2-3주)

결과: 3-4개 프로젝트, 9-12가지 문제 해결 사례
```

### DevOps/SRE 로드맵 (15-20주)
```
Week 1-4:   Docker 배포 (입문)
Week 5-12:  Kubernetes 배포 (중급)
Week 13-20: CI/CD & GitOps (고급)

결과: 3개 프로젝트, 9가지 문제 해결 사례
```

### AI/ML 엔지니어 로드맵 (16-21주)
```
Week 1-5:   RAG 챗봇 (입문)
Week 6-13:  MLOps 서빙 (중급)
Week 14-21: End-to-End 파이프라인 (고급)

결과: 3개 프로젝트, 9가지 문제 해결 사례
```

### 풀스택 엔지니어 로드맵 (15-19주)
```
Week 1-5:   Django + React 블로그 (입문)
Week 6-11:  실시간 채팅 (중급)
Week 12-19: 이커머스 플랫폼 (고급)

결과: 3개 프로젝트, 9가지 문제 해결 사례
```

---

## 📊 완성된 가이드 → 프로젝트 매핑

### 완성된 가이드 (29개) 활용

#### 백엔드 개발
- **Django & DRF** → Django 블로그 플랫폼
- **Spring Boot 실전** → Spring Boot REST API
- **Spring 심화 & MSA** → 마이크로서비스 아키텍처
- **PostgreSQL** → 분석 대시보드 (백엔드용)
- **Redis** → (백엔드에서 캐싱 활용)

#### 데이터 엔지니어링
- **PostgreSQL** → 분석 대시보드
- **Apache Airflow** → ETL 파이프라인 자동화
- **Apache Spark 실전** → 대용량 데이터 분석 플랫폼
- **BigQuery** → 데이터 웨어하우스 (Spark 대신 선택 가능)
- **Data Engineering** → 전체 아키텍처 설계

**🔥 TODO 가이드 (데이터 엔지니어링 강화):**
- **Kafka & 스트림 처리 심화 (15 챕터)** → 실시간 이벤트 처리 파이프라인
- **Redis & 캐싱 전략 (12 챕터)** → 실시간 순위 시스템
- **Elasticsearch 실전 (12 챕터)** → 로그 분석 & 검색 시스템

#### DevOps/SRE
- **Kubernetes** → 마이크로서비스 배포
- **DevOps & SRE 실무** → CI/CD & GitOps 플랫폼
- **AWS & GCP 실전** → 클라우드 인프라 구축
- **Observability** → 모니터링 & 알림

#### AI/ML
- **LLM & AI 애플리케이션** → RAG 기반 AI 챗봇
- **MLOps & AI 인프라** → MLOps 모델 서빙 시스템
- **Apache Spark 실전** → Feature Engineering (ML용)

#### 풀스택
- **Django & DRF** (백엔드) + **TODO: React & Next.js (15 챕터)** (프론트) → 블로그 플랫폼
- **Spring Boot** (백엔드) + **TODO: React & Next.js** (프론트) → 이커머스

#### 모바일
- **Kotlin 완벽 가이드** + **TODO: Android 기초 (12 챕터)** → Android 앱
- **Kotlin** + **TODO: Jetpack Compose 심화 (12 챕터)** → 현대적 Android UI

---

## 🎯 프로젝트 선택 가이드

### 어떤 직무로 갈지 모르겠다면?

**Step 1: 백엔드 입문부터 시작 (만능 기초)**
```
Django 블로그 플랫폼 (3-4주)
→ 모든 직무에서 활용 가능한 기초 역량
→ 백엔드, 데이터, 풀스택 모두 연결됨
```

**Step 2: 관심 분야 테스트**
```
데이터 관심 → PostgreSQL 대시보드
인프라 관심 → Docker 배포
AI 관심 → RAG 챗봇
```

**Step 3: 직무 확정 후 로드맵 선택**

### 시간이 부족하다면?

**최소 포트폴리오 (8-10주):**
```
백엔드: Django 블로그 (4주) + Spring Boot API (5주)
데이터: PostgreSQL 대시보드 (5주) + Kafka 이벤트 처리 (5주)
DevOps: Docker 배포 (4주) + Kubernetes 배포 (6주)
AI/ML: RAG 챗봇 (5주) + MLOps 서빙 (6주)
```

### 완벽주의 주의!

**목표:**
- ❌ 프로덕션 완벽 배포
- ✅ **문제 3개씩 해결 (총 9개 문제 해결 경험)**

**면접에서:**
```
면접관: "어떤 프로젝트 했나요?"
나: "3개 프로젝트에서 총 9가지 문제를 해결했습니다"
   1. Django 블로그: N+1 쿼리 100개→3개, 응답시간 80% 개선
   2. Django 블로그: Redis 캐싱으로 2초→50ms
   3. Django 블로그: Cursor 페이지네이션으로 10만 건 처리
   4. Spring Boot API: JPA N+1 해결 (EntityGraph)
   5. Spring Boot API: 재고 동시성 제어 (Pessimistic Lock)
   6. Spring Boot API: Spring Cache로 3초→50ms
   ... (총 9개)
```

---

## 📝 다음 논의 주제

- [x] 사이드 프로젝트 개발 수준 정리 (완료)
- [x] 가이드별 추천 사이드 프로젝트 목록 (완료)
- [x] **직무별 프로젝트 분류 & 로드맵** (완료) ⭐
- [ ] **목표 직무 선택** (백엔드? 데이터? DevOps? AI/ML? 풀스택?)
- [ ] 선택한 직무의 첫 번째 프로젝트 시작
- [ ] 프로젝트별 상세 요구사항 정의
- [ ] 블로그 작성 템플릿 만들기

---

## 🔥 직무별 프로젝트 충분성 분석

### 1️⃣ 백엔드 엔지니어

#### Q: 백엔드 3개 프로젝트로 충분한가?

**A: 기본 3개면 충분, 차별화는 5-6개 추천**

**최소 (3개 프로젝트, 12-15주):**
```
1. Django 블로그 API (3-4주)
2. Spring Boot REST API (4-5주)
3. 마이크로서비스 (5-6주)

총 9가지 문제 해결
→ 신입 백엔드 면접 충분 ✅
```

**차별화 (6개 프로젝트, 21-27주):**
```
기본 3개 + 선택 3개:
4. Redis 캐싱 (2-3주)
5. GraphQL API (3-4주)
6. API Gateway & 인증 (4-5주)

총 18가지 문제 해결
→ 대기업/외국계 지원 가능 🔥
```

**실제 채용공고 분석:**
```
필수 (90%):
✅ REST API 설계
✅ Database (PostgreSQL/MySQL)
✅ ORM (Django ORM / JPA)
✅ 캐싱 (Redis)
✅ 동시성 처리

우대 (60%):
⭐ MSA 경험
⭐ GraphQL
⭐ 인증/인가 (OAuth2, JWT)
⭐ 테스트 코드
⭐ 클라우드 배포

당신의 6개 프로젝트:
→ 필수 100% + 우대 80% 커버
```

---

### 2️⃣ DevOps/SRE 엔지니어

#### Q: DevOps 3개 프로젝트로 충분한가?

**A: 기본 3개로 시작, 실전 경험은 5-6개 추천**

**최소 (3개 프로젝트, 15-20주):**
```
1. Docker 배포 (3-4주)
2. Kubernetes 배포 (6-8주)
3. CI/CD & GitOps (6-8주)

총 9가지 문제 해결
→ 신입 DevOps 면접 가능 ✅
```

**실전 레벨 (6개 프로젝트, 26-33주):**
```
기본 3개 + 선택 3개:
4. 로그 & 모니터링 (3-4주)
5. IaC (Terraform) (4-5주)
6. Chaos Engineering (4-5주)

총 18가지 문제 해결
→ SRE/Platform Engineer 지원 가능 🔥
```

**실제 채용공고 분석:**
```
필수 (90%):
✅ Docker, Kubernetes
✅ CI/CD (Jenkins/GitHub Actions)
✅ 리눅스, Shell Script
✅ 클라우드 (AWS/GCP)

우대 (70%):
⭐ IaC (Terraform)
⭐ 모니터링 (Prometheus, Grafana)
⭐ GitOps (ArgoCD)
⭐ Service Mesh (Istio)
⭐ SRE 경험

당신의 6개 프로젝트:
→ 필수 100% + 우대 90% 커버
```

---

### 3️⃣ AI/ML 엔지니어

#### Q: AI/ML 3개 프로젝트로 충분한가?

**A: 최소 3개, LLM 특화면 추가 불필요 / CV/추천 하려면 5-6개**

**LLM 특화 (3개 프로젝트, 16-21주):**
```
1. RAG 챗봇 (4-5주)
2. MLOps 서빙 (6-8주)
3. End-to-End 파이프라인 (6-8주)

총 9가지 문제 해결
→ LLM 엔지니어 충분 ✅
```

**풀 스택 ML (6개 프로젝트, 28-37주):**
```
기본 3개 + 선택 3개:
4. LLM Fine-tuning (3-4주)
5. 추천 시스템 (4-5주)
6. 컴퓨터 비전 (4-5주)

총 18가지 문제 해결
→ ML Engineer / Research Engineer 🔥
```

**실제 채용공고 분석:**
```
LLM 엔지니어 필수 (90%):
✅ LLM API (OpenAI, Anthropic)
✅ RAG 아키텍처
✅ Vector DB
✅ Prompt Engineering
✅ MLOps (모델 서빙, 모니터링)

전통 ML 엔지니어 필수:
✅ Scikit-learn / PyTorch / TensorFlow
✅ Feature Engineering
✅ 모델 학습 & 평가
✅ 추천 시스템 / CV 중 하나

당신의 선택:
LLM 특화 → 3개 프로젝트 충분
풀스택 ML → 6개 프로젝트 추천
```

---

### 4️⃣ 풀스택 엔지니어

#### Q: 풀스택 3개 프로젝트로 충분한가?

**A: 최소 3개, 모바일까지 커버하려면 5개 추천**

**웹 풀스택 (3개 프로젝트, 15-19주):**
```
1. Django + React 블로그 (4-5주)
2. 실시간 채팅 (5-6주)
3. 이커머스 플랫폼 (6-8주)

총 9가지 문제 해결
→ 웹 풀스택 충분 ✅
```

**풀스택 + 모바일 (5개 프로젝트, 25-30주):**
```
기본 3개 + 선택 2개:
4. 모바일 앱 (React Native) (3-4주)
5. 관리자 대시보드 (3-4주)

총 15가지 문제 해결
→ 스타트업 창업 멤버급 🔥
```

**실제 채용공고 분석:**
```
필수 (90%):
✅ 백엔드 (Django/Spring)
✅ 프론트엔드 (React/Vue)
✅ Database (SQL)
✅ REST API
✅ 배포 경험

우대 (60%):
⭐ 모바일 (React Native)
⭐ 실시간 통신 (WebSocket)
⭐ 클라우드 배포
⭐ MSA 경험

당신의 프로젝트:
웹 3개 → 필수 100% 커버
+모바일 2개 → 우대 80% 커버
```

---

## 🔥 데이터 엔지니어 특화 추가 추천

### Q: 데이터 엔지니어는 기본 3개 프로젝트로 충분한가?

**A: 신입/주니어 → 충분, 경력직/시니어 → 5-6개 추천**

#### 신입/주니어 레벨 (최소 3개, 15-19주)
```
필수 3개:
1. PostgreSQL 대시보드 (4-5주)
2. Kafka 이벤트 처리 (5-6주)
3. Spark 분석 플랫폼 (6-8주)

총 9가지 문제 해결 사례
→ 신입 데이터 엔지니어 면접 충분
```

#### 경력직/차별화 원한다면 (5-6개, 28-36주)
```
필수 3개 + 선택 2-3개:
4. Redis 캐싱 (2-3주) - 성능 최적화 경험
5. Elasticsearch 로그 분석 (3-4주) - ELK Stack 경험
6. Airflow + dbt (4-5주) - 데이터 품질 & 변환 자동화

총 15-18가지 문제 해결 사례
→ 경력직/대기업/스타트업 데이터팀 리드 지원 가능
```

### 데이터 엔지니어 포트폴리오 완성도 비교

#### 최소 버전 (3개 프로젝트)
```
커버 범위:
✅ 배치 처리 (Spark)
✅ 실시간 스트림 (Kafka)
✅ 데이터베이스 최적화 (PostgreSQL)
✅ ETL 파이프라인 (Airflow - Spark 프로젝트에 포함)

부족한 부분:
❌ 캐싱 전략 (Redis)
❌ 로그 분석 (ELK)
❌ 데이터 품질 자동화 (dbt)
❌ 검색 엔진 (Elasticsearch)

평가:
→ 신입 면접: 충분 ✅
→ 3년차 이상: 부족할 수 있음 ⚠️
```

#### 풀 버전 (6개 프로젝트)
```
커버 범위:
✅ 배치 처리 (Spark)
✅ 실시간 스트림 (Kafka)
✅ 데이터베이스 최적화 (PostgreSQL)
✅ ETL 자동화 (Airflow + dbt)
✅ 캐싱 전략 (Redis)
✅ 로그 분석 & 검색 (Elasticsearch)

강점:
✅ 전체 데이터 스택 커버
✅ 성능 최적화 경험 풍부
✅ 실시간 + 배치 모두 가능
✅ 데이터 품질 관리 경험

평가:
→ 신입: 과한 감 있지만 차별화 가능 ⭐
→ 3년차 이상: 완벽 ✅✅✅
→ 데이터팀 리드 지원 가능 🔥
```

### 추천 전략

#### 전략 1: 최소 → 취업 → 실무 보강
```
Step 1: 기본 3개 완성 (15-19주)
Step 2: 취업 활동 시작
Step 3: 취업 안 되면 선택 프로젝트 추가
Step 4: 취업 후 실무에서 ELK, dbt 경험
```

#### 전략 2: 완벽 포트폴리오 먼저 (추천!)
```
Step 1: 기본 3개 + 선택 2-3개 완성 (28-36주, 7-9개월)
Step 2: 강력한 포트폴리오로 취업 활동
Step 3: 대기업/유니콘 스타트업 지원 가능
Step 4: 연봉 협상력 상승
```

### 실무 데이터 엔지니어가 보는 기술 스택

**실제 채용공고 분석:**
```
필수 (90% 이상 요구):
✅ SQL (PostgreSQL/MySQL)
✅ Python
✅ Airflow / 워크플로우 도구
✅ Spark / 대용량 배치
✅ Kafka / 실시간 스트림

우대사항 (60% 요구):
⭐ Redis (캐싱)
⭐ Elasticsearch (로그/검색)
⭐ dbt (데이터 변환)
⭐ Kubernetes (배포)
⭐ 클라우드 (AWS/GCP)

추가 가산점 (30% 요구):
🔥 Flink (고급 스트림 처리)
🔥 실시간 대시보드 (BI 툴)
🔥 데이터 품질 모니터링
```

**당신의 6개 프로젝트:**
```
✅ PostgreSQL 대시보드 → SQL + 최적화 + BI
✅ Kafka 이벤트 처리 → 실시간 스트림 + Flink
✅ Spark 분석 플랫폼 → 대용량 배치 + Airflow
✅ Redis 캐싱 → 성능 최적화
✅ Elasticsearch 로그 → ELK Stack
✅ Airflow + dbt → ETL 자동화 + 데이터 품질

→ 필수 100% 커버 + 우대사항 80% 커버 🔥
```

### 현실적인 추천

**시간이 충분하다면 (7-9개월 여유):**
- **6개 프로젝트 모두 완성** → 최상의 포트폴리오

**시간이 부족하다면 (4-5개월만 가능):**
- **기본 3개 + Redis 캐싱** → 최소한의 차별화
- 또는 **기본 3개 + Airflow/dbt** → 실무 파이프라인 강조

**초급자라면:**
- **기본 3개부터 시작** → 취업 활동 병행 → 필요 시 추가

### 데이터 엔지니어 최종 추천 로드맵

#### Phase 1: 기본 3개 (15-19주)
```
Week 1-5:   PostgreSQL 대시보드
Week 6-11:  Kafka 이벤트 처리
Week 12-19: Spark 분석 플랫폼

→ 취업 활동 가능 수준 ✅
```

#### Phase 2: 선택 추가 (+ 9-12주)
```
Week 20-22: Redis 캐싱 (성능 최적화 강조)
Week 23-26: Elasticsearch 로그 분석 (ELK 경험)
Week 27-31: Airflow + dbt (데이터 품질 자동화)

→ 대기업/유니콘 지원 가능 🔥
```

**총 28-31주 (약 7-8개월):**
```
6개 프로젝트
18가지 문제 해결 사례
전체 데이터 스택 커버
→ 최상의 데이터 엔지니어 포트폴리오 ⭐⭐⭐⭐⭐
```

---

### 🗄️ 기존 개별 프로젝트 상세 (참고용)

<details>
<summary>접기/펼치기 (개별 프로젝트 상세 설명)</summary>

---

### 1. Django & DRF 가이드 → 블로그 플랫폼 API

**난이도:** ⭐⭐ (입문)
**기간:** 3~4주

**기술 스택:**
- Backend: Django + DRF
- Database: PostgreSQL
- Cache: Redis (선택)
- Deploy: Docker

**핵심 기능:**
- 사용자 인증 (JWT)
- 게시글 CRUD
- 댓글 시스템
- 태그/카테고리
- 페이지네이션

**목표 문제 해결 3가지:**

1. **N+1 쿼리 문제**
   - 문제: 게시글 목록 조회 시 작성자 정보 조회로 100개 쿼리 발생
   - 분석: django-debug-toolbar 사용
   - 해결: `select_related('author')`, `prefetch_related('tags')`
   - 결과: 쿼리 100개 → 3개, 응답시간 80% 개선

2. **API 응답시간 개선**
   - 문제: 인기 게시글 조회 API가 2초 걸림
   - 분석: 매번 DB 쿼리 + 집계 계산
   - 해결: Redis 캐싱 (Cache-aside 패턴)
   - 결과: 응답시간 2초 → 50ms, Cache Hit ratio 85%

3. **대량 데이터 페이지네이션**
   - 문제: 게시글 10만개일 때 마지막 페이지 조회 느림
   - 분석: OFFSET 방식의 한계
   - 해결: Cursor-based 페이지네이션
   - 결과: 마지막 페이지 조회 시간 5초 → 100ms

**학습 포인트:**
- Django ORM 최적화
- DRF Serializer 성능
- JWT 인증 구현
- Redis 캐싱 전략

---

### 2. PostgreSQL 가이드 → 분석 대시보드

**난이도:** ⭐⭐⭐ (중급)
**기간:** 4~5주

**기술 스택:**
- Backend: Django/FastAPI
- Database: PostgreSQL (고급 기능 활용)
- Visualization: Chart.js (프론트)

**핵심 기능:**
- 사용자 행동 로그 저장
- 실시간 통계 대시보드
- 복잡한 집계 쿼리
- 시계열 데이터 분석

**목표 문제 해결 3가지:**

1. **느린 집계 쿼리 최적화**
   - 문제: 월별 매출 집계 쿼리 30초 소요
   - 분석: EXPLAIN ANALYZE로 실행 계획 확인
   - 해결: Materialized View + 인덱스 추가
   - 결과: 쿼리 시간 30초 → 200ms

2. **대용량 데이터 INSERT 성능**
   - 문제: 100만 건 로그 저장 시 10분 소요
   - 분석: 개별 INSERT의 비효율
   - 해결: COPY 명령어 + Bulk Insert
   - 결과: 10분 → 30초 (20배 개선)

3. **동시 읽기/쓰기 성능**
   - 문제: 통계 조회 중 새 로그 저장 시 락 발생
   - 분석: Table Lock 문제
   - 해결: Partitioning (날짜별) + MVCC 활용
   - 결과: 동시 처리 가능, 락 대기 시간 0

**학습 포인트:**
- PostgreSQL 고급 기능 (Window Functions, CTE)
- 인덱스 전략 (B-tree, GIN, BRIN)
- 파티셔닝
- 쿼리 최적화

---

### 3. Redis & 캐싱 가이드 → 실시간 순위 시스템

**난이도:** ⭐⭐ (입문~중급)
**기간:** 2~3주

**기술 스택:**
- Backend: Django/FastAPI
- Cache: Redis (Sorted Set, String, Hash)
- Database: PostgreSQL

**핵심 기능:**
- 실시간 게임 점수 순위
- 좋아요/조회수 카운터
- 세션 관리
- Rate Limiting

**목표 문제 해결 3가지:**

1. **순위 조회 성능**
   - 문제: DB에서 ORDER BY로 순위 조회 시 5초
   - 분석: 매번 전체 정렬 필요
   - 해결: Redis Sorted Set (ZADD, ZRANGE)
   - 결과: 5초 → 5ms (1000배 개선)

2. **동시 카운터 업데이트**
   - 문제: 조회수 증가 시 Race Condition 발생
   - 분석: Read-Modify-Write 문제
   - 해결: Redis INCR (Atomic Operation)
   - 결과: 정확한 카운팅, 동시성 문제 해결

3. **캐시 무효화 전략**
   - 문제: 데이터 변경 시 캐시와 DB 불일치
   - 분석: Cache-aside 패턴의 한계
   - 해결: Write-through + TTL 조합
   - 결과: 데이터 일관성 유지, Stale data 0%

**학습 포인트:**
- Redis 자료구조 활용
- 캐싱 패턴 (Cache-aside, Write-through)
- TTL 전략
- 동시성 제어

---

### 4. Kafka & 스트림 처리 가이드 → 실시간 이벤트 처리

**난이도:** ⭐⭐⭐⭐ (고급)
**기간:** 5~6주

**기술 스택:**
- Message Queue: Kafka
- Stream Processing: Kafka Streams / Flink
- Database: PostgreSQL
- Backend: Django/Spring

**핵심 기능:**
- 사용자 행동 이벤트 수집
- 실시간 이상 탐지
- 이벤트 기반 알림
- CQRS 패턴 구현

**목표 문제 해결 3가지:**

1. **대량 이벤트 처리**
   - 문제: 초당 1만 이벤트 처리 시 DB 병목
   - 분석: 동기 처리의 한계
   - 해결: Kafka Producer로 비동기 처리
   - 결과: 초당 10만 이벤트 처리 가능

2. **실시간 집계**
   - 문제: 1분 단위 집계를 배치로 처리 → 10분 지연
   - 분석: 배치 처리의 레이턴시
   - 해결: Kafka Streams Window 연산
   - 결과: 실시간 집계, 지연 1초 이하

3. **정확히 한 번 처리 (Exactly-once)**
   - 문제: Consumer 재시작 시 중복 처리
   - 분석: At-least-once 보장의 한계
   - 해결: Kafka Transactions + Idempotent Producer
   - 결과: 중복 처리 0%, 데이터 정합성 보장

**학습 포인트:**
- Kafka Producer/Consumer
- Kafka Streams
- Exactly-once semantics
- 이벤트 기반 아키텍처

---

### 5. LLM & AI 애플리케이션 가이드 → AI 챗봇 서비스

**난이도:** ⭐⭐⭐ (중급)
**기간:** 4~5주

**기술 스택:**
- LLM API: OpenAI / Anthropic Claude
- Vector DB: ChromaDB / Pinecone
- Backend: Django/FastAPI
- Database: PostgreSQL

**핵심 기능:**
- RAG 기반 문서 QA
- 대화 히스토리 관리
- 프롬프트 엔지니어링
- 비용 최적화

**목표 문제 해결 3가지:**

1. **RAG 검색 정확도 개선**
   - 문제: 관련 없는 문서 검색 → 잘못된 답변
   - 분석: Chunk 크기, Embedding 모델 문제
   - 해결: Chunk 전략 개선 + Reranking
   - 결과: 검색 정확도 60% → 90%

2. **LLM API 비용 절감**
   - 문제: 월 API 비용 $500
   - 분석: 불필요한 긴 프롬프트, 중복 요청
   - 해결: Prompt Caching + 응답 캐싱
   - 결과: 비용 $500 → $150 (70% 절감)

3. **응답 시간 단축**
   - 문제: 답변 생성 10초 소요
   - 분석: 동기 처리 + 긴 컨텍스트
   - 해결: Streaming 응답 + 컨텍스트 압축
   - 결과: 첫 토큰 응답 1초 이내

**학습 포인트:**
- RAG 아키텍처
- Vector DB 활용
- Prompt Engineering
- LLM API 비용 최적화

---

### 6. Kubernetes 가이드 → 마이크로서비스 배포

**난이도:** ⭐⭐⭐⭐ (고급)
**기간:** 6~8주

**기술 스택:**
- Container: Docker
- Orchestration: Kubernetes
- Service Mesh: Istio (선택)
- Monitoring: Prometheus + Grafana

**핵심 기능:**
- 마이크로서비스 배포 (User, Order, Payment)
- Service Discovery
- Load Balancing
- Auto Scaling

**목표 문제 해결 3가지:**

1. **무중단 배포**
   - 문제: 배포 시 서비스 다운타임 발생
   - 분석: 기존 Pod 종료 → 새 Pod 시작 순서 문제
   - 해결: Rolling Update + Readiness Probe
   - 결과: 다운타임 0, 배포 중 요청 성공률 100%

2. **트래픽 급증 대응**
   - 문제: 이벤트 시 트래픽 10배 증가 → 서버 다운
   - 분석: 고정된 Pod 수
   - 해결: HPA (Horizontal Pod Autoscaler)
   - 결과: CPU 70% 이상 시 자동 스케일 아웃

3. **서비스 간 통신 장애**
   - 문제: Order 서비스 장애 시 User 서비스도 영향
   - 분석: Circuit Breaker 부재
   - 해결: Istio Circuit Breaker + Retry
   - 결과: 장애 격리, 전체 시스템 안정성 향상

**학습 포인트:**
- Kubernetes 리소스 (Pod, Service, Deployment)
- Auto Scaling
- Service Mesh
- Monitoring & Logging

---

### 7. Spring Boot 가이드 → REST API 서버

**난이도:** ⭐⭐⭐ (중급)
**기간:** 4~5주

**기술 스택:**
- Framework: Spring Boot 3.x
- Database: PostgreSQL
- ORM: Spring Data JPA
- Cache: Redis

**핵심 기능:**
- 사용자 인증 (JWT)
- 상품 관리 CRUD
- 주문 처리
- 재고 관리

**목표 문제 해결 3가지:**

1. **N+1 쿼리 문제**
   - 문제: 주문 목록 조회 시 100개 쿼리 발생
   - 분석: Lazy Loading으로 Order → User, Product 조회
   - 해결: `@EntityGraph`, `fetch join`
   - 결과: 쿼리 100개 → 1개, 응답시간 90% 개선

2. **동시 주문 재고 처리**
   - 문제: 동시 주문 시 재고 음수 발생
   - 분석: Race Condition (Read-Modify-Write)
   - 해결: `@Lock(PESSIMISTIC_WRITE)` + 트랜잭션
   - 결과: 재고 정합성 보장, 오버셀링 0건

3. **API 응답시간 개선**
   - 문제: 상품 상세 조회 3초
   - 분석: 복잡한 집계 쿼리 매번 실행
   - 해결: Spring Cache + Redis
   - 결과: 3초 → 50ms

**학습 포인트:**
- Spring Data JPA
- 트랜잭션 관리
- Spring Cache
- Exception Handling

---

### 8. MLOps 가이드 → ML 모델 서빙 시스템

**난이도:** ⭐⭐⭐⭐ (고급)
**기간:** 6~8주

**기술 스택:**
- ML Framework: Scikit-learn / PyTorch
- Model Serving: FastAPI / TorchServe
- Experiment Tracking: MLflow
- Orchestration: Airflow

**핵심 기능:**
- 모델 학습 파이프라인
- 모델 서빙 API
- A/B Testing
- 모델 모니터링

**목표 문제 해결 3가지:**

1. **모델 추론 레이턴시**
   - 문제: 추론 API 응답 5초
   - 분석: 모델 로딩, 전처리 병목
   - 해결: 모델 캐싱 + 배치 추론
   - 결과: 5초 → 100ms

2. **모델 성능 저하 감지**
   - 문제: 모델 정확도 85% → 70%로 하락 (모르고 있음)
   - 분석: Data Drift 발생
   - 해결: Evidently로 Drift Detection
   - 결과: 자동 알림, 재학습 트리거

3. **재현 가능한 실험**
   - 문제: 같은 코드인데 결과 다름
   - 분석: Random Seed, 데이터 버전 관리 부재
   - 해결: MLflow + DVC
   - 결과: 완벽한 재현성, 실험 추적

**학습 포인트:**
- MLflow 실험 추적
- 모델 서빙
- A/B Testing
- Drift Detection

---

## 🎯 프로젝트 선택 가이드

### 초급자 추천 순서

```
1단계: Django 블로그 플랫폼 (3~4주)
   → ORM, DRF, 캐싱 기본

2단계: Redis 실시간 순위 (2~3주)
   → 캐싱 심화, 동시성

3단계: PostgreSQL 대시보드 (4~5주)
   → DB 최적화, 복잡한 쿼리
```

### 중급자 추천 순서

```
1단계: Spring Boot REST API (4~5주)
   → Java 생태계, JPA

2단계: Kafka 이벤트 처리 (5~6주)
   → 스트림 처리, MSA

3단계: Kubernetes 배포 (6~8주)
   → 컨테이너, 오케스트레이션
```

### 고급자 추천 순서

```
1단계: MLOps 모델 서빙 (6~8주)
   → ML 파이프라인

2단계: LLM AI 챗봇 (4~5주)
   → RAG, Vector DB

3단계: 종합 프로젝트 (MSA + K8s + Kafka)
```

---

## 📝 다음 논의 주제

- [x] 사이드 프로젝트 개발 수준 정리 (완료)
- [x] 가이드별 추천 사이드 프로젝트 목록 (완료)
- [ ] Phase 1 프로젝트 선택 (Django 블로그 vs 다른 것)
- [ ] 선택한 프로젝트 상세 요구사항 정의
- [ ] 블로그 작성 템플릿 만들기
- [ ] 오픈소스 기여 시작 시점

---

**마지막 업데이트**: 2025-01-16

**핵심 메시지**:
> 가이드는 "총(도구)"이고, 사이드 프로젝트는 "실전(경험)"이다.
> 둘 다 필요하지만, **문제 해결 경험이 더 중요하다**.
>
> 프로덕션 레벨 완성도보다 **"문제 3개 해결"**이 면접에서 더 강력하다.
>
> **직무별 로드맵**: 목표 직무에 맞는 프로젝트 3개 → 9가지 문제 해결 → 포트폴리오 완성
