# 03. 고급 패턴 분석

**분석 대상**: Vet 도메인, Caching, i18n 설정
**핵심 질문**: 왜 이런 패턴을 선택했을까?

---

## 1. @ManyToMany 관계: Vet ↔ Specialty

### 1.1 왜 @ManyToMany인가?

**코드** (`vet/Vet.java:48-50`)
```java
@ManyToMany(fetch = FetchType.EAGER)
@JoinTable(name = "vet_specialties",
    joinColumns = @JoinColumn(name = "vet_id"),
    inverseJoinColumns = @JoinColumn(name = "specialty_id"))
private @Nullable Set<Specialty> specialties;
```

**비즈니스 요구사항**:
- 한 수의사는 여러 전문분야를 가질 수 있음 (예: 치과 + 외과)
- 한 전문분야는 여러 수의사가 공유 (여러 수의사가 "치과" 전문)

**왜 @OneToMany가 아닌가?**
- @OneToMany는 "소유" 관계 (Owner → Pet처럼)
- Specialty는 독립적으로 존재하며, 수의사가 삭제되어도 전문분야는 남아야 함
- 여러 수의사가 같은 Specialty 인스턴스를 공유해야 함

**왜 @JoinTable을 명시했는가?**
```java
@JoinTable(name = "vet_specialties",  // 중간 테이블 이름 명시
    joinColumns = @JoinColumn(name = "vet_id"),          // FK: Vet 쪽
    inverseJoinColumns = @JoinColumn(name = "specialty_id"))  // FK: Specialty 쪽
```
- JPA 기본 이름: `vet_specialty` (단수형)
- DB 스키마에서는 `vet_specialties` (복수형)
- **명시적 설정으로 DB 스키마와 정확히 일치**시킴

---

### 1.2 왜 Set을 사용했는가?

**코드** (`vet/Vet.java:51`)
```java
private @Nullable Set<Specialty> specialties;  // List가 아닌 Set
```

**이유**:
1. **중복 방지**: 같은 전문분야를 두 번 추가할 수 없음
2. **@ManyToMany 권장사항**: JPA에서 Set 사용 권장 (중복 insert 방지)
3. **순서 불필요**: 전문분야에는 우선순위가 없음

**비교**: Owner.pets는 List인 이유?
- Pet은 순서가 필요 (등록순, 이름순 정렬)
- Pet은 중복될 수 없음 (1:N 관계에서 FK로 보장)
- List로도 중복 문제 없음

---

### 1.3 왜 getSpecialtiesInternal() 패턴인가?

**코드** (`vet/Vet.java:53-58, 61-64`)
```java
protected Set<Specialty> getSpecialtiesInternal() {
    if (this.specialties == null) {
        this.specialties = new HashSet<>();  // Lazy initialization
    }
    return this.specialties;
}

@XmlElement
public List<Specialty> getSpecialties() {
    return getSpecialtiesInternal().stream()
        .sorted(Comparator.comparing(NamedEntity::getName))  // 정렬!
        .collect(Collectors.toList());
}
```

**왜 두 개의 getter가 필요한가?**

1. **getSpecialtiesInternal()** (protected)
   - **목적**: 내부 수정용 (addSpecialty 메소드에서 사용)
   - **null 체크**: JPA는 초기화를 보장하지 않음, NPE 방지
   - **반환 타입**: Set (원본 컬렉션)

2. **getSpecialties()** (public)
   - **목적**: 외부 노출용 (View, JSON 직렬화)
   - **정렬**: 이름 순으로 정렬된 List 반환
   - **불변성**: Stream으로 새 List 생성 (원본 수정 방지)

**왜 매번 새로운 List를 생성하는가?**
```java
.collect(Collectors.toList());  // 매번 새 리스트 생성
```
- **성능보다 안전성**: Vet 조회는 빈번하지 않음 (캐시 사용)
- **데이터 보호**: 외부에서 반환된 List를 수정해도 내부 Set에 영향 없음
- **항상 최신 정렬**: specialties가 변경되어도 호출 시마다 정렬 보장

---

## 2. Repository 수준별 추상화

### 2.1 왜 VetRepository는 Repository를 extends하나?

**비교**: OwnerRepository vs VetRepository

**OwnerRepository** (`owner/OwnerRepository.java:34`)
```java
public interface OwnerRepository extends JpaRepository<Owner, Integer> {
    // save, findById, delete 등 모든 CRUD 메소드 필요
}
```

**VetRepository** (`vet/VetRepository.java:38`)
```java
public interface VetRepository extends Repository<Vet, Integer> {
    // 필요한 메소드만 명시적으로 선언
    Collection<Vet> findAll();
    Page<Vet> findAll(Pageable pageable);
}
```

**왜 차이가 있는가?**

| 기준 | Owner | Vet |
|------|-------|-----|
| **생성** | ✅ 필요 (신규 고객 등록) | ❌ 불필요 (수의사는 관리자만 추가) |
| **수정** | ✅ 필요 (주소, 전화번호) | ❌ 불필요 (전문분야는 고정) |
| **삭제** | ✅ 필요 (고객 탈퇴) | ❌ 불필요 (수의사는 영구 보관) |
| **조회** | ✅ 필요 (ID, 이름 검색) | ✅ 필요 (전체 목록만) |

**설계 원칙: Interface Segregation Principle (ISP)**
- VetRepository에 save(), delete() 메소드가 있으면?
- → 실수로 호출할 위험 증가
- → **필요한 메소드만 노출**하여 안전성 확보

**Repository 인터페이스 계층**:
```
Repository<T, ID>              // 마커 인터페이스 (메소드 없음)
  └─ CrudRepository<T, ID>     // save, findById, delete 등
       └─ JpaRepository<T, ID> // flush, batch 등 JPA 전용 메소드
```

VetRepository는 가장 최소한의 `Repository`만 상속하여 findAll만 제공!

---

### 2.2 왜 @Cacheable을 Repository에 붙였는가?

**코드** (`vet/VetRepository.java:44-46, 54-56`)
```java
@Transactional(readOnly = true)
@Cacheable("vets")
Collection<Vet> findAll() throws DataAccessException;

@Transactional(readOnly = true)
@Cacheable("vets")
Page<Vet> findAll(Pageable pageable) throws DataAccessException;
```

**왜 캐시가 필요한가?**
1. **읽기 전용 데이터**: Vet 목록은 거의 변경되지 않음
2. **조인 비용**: Vet → Specialty (@ManyToMany, EAGER) 조인 필요
3. **빈번한 조회**: "/vets.html" 페이지는 자주 접근됨

**왜 Service가 아닌 Repository에 붙였는가?**
- Service Layer가 없음 (Controller → Repository 직접 호출)
- **가능한 가장 낮은 레벨에서 캐싱** (DB 접근 전에 차단)

**왜 두 메소드 모두 같은 캐시 이름인가?**
```java
@Cacheable("vets")  // 동일한 캐시 이름
```
- **데이터 일관성**: 하나의 캐시만 관리하면 됨
- **무효화 간단**: `@CacheEvict("vets")` 한 번으로 모든 캐시 삭제
- **실제 사용**: 이 프로젝트에서는 Vet가 변경되지 않으므로 무효화 불필요

**왜 @Transactional(readOnly = true)도 함께 사용했는가?**
- **성능 최적화**: Hibernate가 스냅샷 생성 스킵 (변경 감지 불필요)
- **명확한 의도**: 이 메소드는 읽기 전용임을 선언
- **DB 최적화**: 일부 DB는 읽기 전용 트랜잭션을 더 효율적으로 처리

---

## 3. Cache 설정: JCache 사용

### 3.1 왜 JCache API를 사용했는가?

**코드** (`system/CacheConfiguration.java:31-38`)
```java
@Configuration(proxyBeanMethods = false)
@EnableCaching
class CacheConfiguration {

    @Bean
    public JCacheManagerCustomizer petclinicCacheConfigurationCustomizer() {
        return cm -> cm.createCache("vets", cacheConfiguration());
    }
}
```

**JCache (JSR-107)란?**
- Java 표준 캐시 API (javax.cache)
- 구현체: EhCache, Hazelcast, Caffeine 등

**왜 Spring의 @EnableCaching만으로는 부족한가?**
- `@EnableCaching`: Spring의 캐시 추상화 활성화
- **캐시 생성은 별도 설정 필요**: "vets" 캐시를 프로그래밍 방식으로 생성

**왜 JCacheManagerCustomizer를 사용했는가?**
```java
return cm -> cm.createCache("vets", cacheConfiguration());
```
- **함수형 인터페이스**: CacheManager를 커스터마이징
- **타입 안전**: "vets" 캐시를 코드에서 명시적으로 생성
- **설정 분리**: application.yml에 의존하지 않음

---

### 3.2 왜 proxyBeanMethods = false인가?

**코드** (`system/CacheConfiguration.java:31`)
```java
@Configuration(proxyBeanMethods = false)
```

**Spring 5.2+의 최적화**:

**기본값 (proxyBeanMethods = true)**:
```java
@Configuration  // proxyBeanMethods = true (기본값)
class Config {
    @Bean
    public MyService service() {
        return new MyService(repository());  // repository() 호출
    }

    @Bean
    public MyRepository repository() {
        return new MyRepository();
    }
}

// 실행 시:
service1.repository() == service2.repository()  // true (싱글톤 보장)
```
- `repository()`를 여러 번 호출해도 같은 인스턴스 반환
- **CGLIB 프록시 생성**: Config 클래스를 상속한 프록시 생성 (오버헤드)

**proxyBeanMethods = false**:
```java
@Configuration(proxyBeanMethods = false)  // Lite 모드
class CacheConfiguration {
    @Bean
    public JCacheManagerCustomizer petclinicCacheConfigurationCustomizer() {
        return cm -> cm.createCache("vets", cacheConfiguration());
    }

    private Configuration<Object, Object> cacheConfiguration() {
        // @Bean이 아님! 단순 private 메소드
        return new MutableConfiguration<>().setStatisticsEnabled(true);
    }
}
```

**왜 이 프로젝트에서 false를 선택했는가?**
1. **Bean 간 의존성 없음**: `cacheConfiguration()`은 @Bean이 아니므로 매번 새 객체 생성
2. **성능 향상**: CGLIB 프록시 생성 불필요
3. **간단한 설정**: Bean이 하나뿐이므로 싱글톤 보장 불필요

**언제 true를 사용해야 하는가?**
```java
@Configuration  // proxyBeanMethods = true 필요
class DatabaseConfig {
    @Bean
    public DataSource dataSource() { ... }

    @Bean
    public EntityManager entityManager() {
        return new EntityManager(dataSource());  // 같은 DataSource 인스턴스 필요
    }
}
```

---

### 3.3 왜 통계만 활성화했는가?

**코드** (`system/CacheConfiguration.java:49-51`)
```java
private javax.cache.configuration.Configuration<Object, Object> cacheConfiguration() {
    return new MutableConfiguration<>().setStatisticsEnabled(true);
}
```

**주석 해석** (41-48줄):
> JCache API에서 제공하는 설정은 매우 제한적입니다.
> 진짜 중요한 설정(캐시 크기 제한 등)은 JCache 구현체(EhCache 등)의 설정 방식을 사용해야 합니다.

**왜 크기 제한을 설정하지 않았는가?**
- **데이터가 적음**: Vet는 수십 명 수준 (KB 단위)
- **변경 없음**: 캐시 무효화가 불필요하므로 무한정 보관 가능
- **샘플 프로젝트**: 실전에서는 EhCache의 `ehcache.xml`에서 설정

**통계 활성화의 목적**:
```java
setStatisticsEnabled(true)
```
- **JMX 모니터링**: 캐시 히트율, 미스율 등을 JMX로 확인 가능
- **학습 목적**: "캐시가 실제로 작동하는지" 확인 가능
- **프로덕션**: APM 도구와 연동하여 성능 측정

---

## 4. 다국어 지원 (i18n) 설정

### 4.1 왜 SessionLocaleResolver를 사용했는가?

**코드** (`system/WebConfiguration.java:32-37`)
```java
@Bean
public LocaleResolver localeResolver() {
    SessionLocaleResolver resolver = new SessionLocaleResolver();
    resolver.setDefaultLocale(Locale.ENGLISH);  // 기본값: 영어
    return resolver;
}
```

**Spring의 Locale 저장 전략**:

| 전략 | 저장 위치 | 장점 | 단점 |
|------|-----------|------|------|
| **SessionLocaleResolver** | 서버 세션 | 사용자별 유지 | 서버 메모리 사용 |
| CookieLocaleResolver | 클라이언트 쿠키 | 서버 부담 없음 | 쿠키 조작 가능 |
| AcceptHeaderLocaleResolver | HTTP Header | 브라우저 자동 감지 | 변경 불가 |

**왜 Session을 선택했는가?**
1. **사용자 경험**: 한 번 설정하면 브라우저 닫기 전까지 유지
2. **보안**: 쿠키보다 안전 (서버에서 관리)
3. **샘플 앱**: 다중 서버 환경 고려 불필요 (프로덕션에서는 Redis 세션 사용)

**왜 기본값을 ENGLISH로 설정했는가?**
- 국제 표준 언어
- 번역이 없을 때의 Fallback

---

### 4.2 왜 LocaleChangeInterceptor를 사용했는가?

**코드** (`system/WebConfiguration.java:44-49, 55-58`)
```java
@Bean
public LocaleChangeInterceptor localeChangeInterceptor() {
    LocaleChangeInterceptor interceptor = new LocaleChangeInterceptor();
    interceptor.setParamName("lang");  // URL 파라미터 이름
    return interceptor;
}

@Override
public void addInterceptors(InterceptorRegistry registry) {
    registry.addInterceptor(localeChangeInterceptor());  // 모든 요청에 적용
}
```

**동작 방식**:
```
사용자 요청: /owners?lang=de
↓
LocaleChangeInterceptor 감지
↓
SessionLocaleResolver에 Locale.GERMAN 저장
↓
이후 모든 요청에서 독일어 메시지 사용
```

**왜 Interceptor 패턴인가?**
- **AOP 스타일**: Controller 코드 수정 불필요
- **모든 요청에 적용**: 어떤 페이지에서든 `?lang=` 파라미터 사용 가능
- **선언적 설정**: `addInterceptors()`로 등록만 하면 자동 동작

**왜 파라미터 이름을 "lang"으로 했는가?**
```java
interceptor.setParamName("lang");  // 기본값은 "locale"
```
- **짧고 직관적**: `?lang=de`가 `?locale=de`보다 간결
- **국제 관례**: 많은 웹사이트가 `lang` 사용 (예: Google)

---

### 4.3 실제 번역 파일 구조

**파일 위치** (Spring Boot 기본 경로):
```
src/main/resources/messages/
├── messages.properties        # 기본 (영어)
├── messages_de.properties     # 독일어
├── messages_es.properties     # 스페인어
└── messages_ko.properties     # 한국어
```

**예시** (`messages.properties`):
```properties
owner.new=New Owner
owner.edit=Edit Owner
telephone.invalid=Phone number must be 10 digits
```

**독일어** (`messages_de.properties`):
```properties
owner.new=Neuer Besitzer
owner.edit=Besitzer bearbeiten
telephone.invalid=Telefonnummer muss 10 Ziffern haben
```

**Thymeleaf에서 사용**:
```html
<h2 th:text="#{owner.new}">New Owner</h2>
```
- `#{}`: 메시지 키로 번역
- 현재 Locale에 따라 자동으로 messages_de.properties에서 가져옴

---

## 5. Wrapper 객체: Vets

### 5.1 왜 Vets 클래스가 필요한가?

**코드** (`vet/Vets.java:31-43`)
```java
@XmlRootElement
public class Vets {

    private @Nullable List<Vet> vets;

    @XmlElement
    public List<Vet> getVetList() {
        if (vets == null) {
            vets = new ArrayList<>();
        }
        return vets;
    }
}
```

**주석 해석** (`VetController.java:46-47, 71-72`):
> Vet 컬렉션 대신 'Vets' 객체를 반환하는 이유:
> Object-XML/JSON 매핑을 더 간단하게 만들기 위해

**왜 List&lt;Vet&gt;를 직접 반환하지 않는가?**

**JSON 직렬화 시** (`GET /vets`):
```java
// Vets 객체 사용
{
  "vetList": [
    {"id": 1, "firstName": "James", ...},
    {"id": 2, "firstName": "Helen", ...}
  ]
}

// List<Vet> 직접 반환 시
[
  {"id": 1, "firstName": "James", ...},
  {"id": 2, "firstName": "Helen", ...}
]
```

**왜 Wrapper가 더 나은가?**
1. **확장성**: 나중에 메타데이터 추가 가능
```json
{
  "vetList": [...],
  "totalCount": 10,
  "lastUpdated": "2025-01-18"
}
```

2. **XML 호환성**: XML은 루트 요소 필요
```xml
<vets>  <!-- @XmlRootElement -->
  <vetList>
    <vet>...</vet>
  </vetList>
</vets>
```

3. **Spring MVC 통합**: @ResponseBody가 Wrapper를 감지하여 자동 변환

---

### 5.2 왜 @XmlRootElement와 @XmlElement를 사용했는가?

**코드** (`vet/Vets.java:31, 36`)
```java
@XmlRootElement  // XML 루트 요소
public class Vets {

    @XmlElement  // XML 자식 요소
    public List<Vet> getVetList() { ... }
}
```

**JAXB (Java Architecture for XML Binding)**:
- Java 객체 ↔ XML 자동 변환
- Spring MVC가 자동으로 JAXB 사용 (Content-Type: application/xml 요청 시)

**왜 이 프로젝트에서 XML 지원이 필요한가?**
- **레거시 호환**: 오래된 클라이언트는 XML 선호
- **샘플 목적**: JSON과 XML 모두 지원하는 방법 시연
- **실전**: 현대 웹에서는 JSON만 사용 (XML 지원 제거 가능)

**VetController에서의 사용**:
```java
@GetMapping("/vets")
public @ResponseBody Vets showResourcesVetList() {
    return vets;  // Content-Type에 따라 JSON 또는 XML로 변환
}
```

---

## 6. 패턴 요약

### 6.1 주요 설계 결정

| 패턴 | 선택 | 이유 |
|------|------|------|
| **Vet-Specialty 관계** | @ManyToMany | 양방향 공유 관계 |
| **컬렉션 타입** | Set | 중복 방지 |
| **Getter 분리** | Internal + Public | 내부 수정 vs 외부 노출 |
| **Repository 추상화** | Repository (최소) | 불필요한 메소드 숨김 (ISP) |
| **캐시 레벨** | Repository | 가장 낮은 레벨에서 DB 접근 차단 |
| **Cache API** | JCache (JSR-107) | 표준 API, 구현체 교체 가능 |
| **Locale 저장** | Session | 사용자별 유지 + 보안 |
| **언어 전환** | Interceptor | Controller 수정 불필요 |
| **JSON/XML 응답** | Wrapper 객체 | 확장성 + XML 호환성 |

### 6.2 핵심 학습 포인트

1. **관계 선택의 기준**
   - 소유 관계 (@OneToMany) vs 공유 관계 (@ManyToMany)
   - 생명주기 결합도 (Cascade)

2. **인터페이스 최소화 (ISP)**
   - JpaRepository vs Repository
   - 필요한 메소드만 노출하여 안전성 확보

3. **캐시 전략**
   - @Cacheable 위치 선택 (Service vs Repository)
   - JCache 표준 사용으로 벤더 종속성 회피

4. **설정 분리**
   - 도메인 로직 (owner/, vet/)
   - 인프라 설정 (system/)
   - 각 설정의 책임 명확히 분리

5. **확장 가능한 설계**
   - Wrapper 객체로 미래 변경 대비
   - Interceptor 패턴으로 횡단 관심사 처리

---

**다음 문서**: 전체 패턴 종합 및 실전 적용 가이드
