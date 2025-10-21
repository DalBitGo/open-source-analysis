# Spring PetClinic 분석

**분석 기간**: 2025-01-18
**분석 깊이**: Level 2 (코드 상세 분석 + "왜" 설명)
**분석 문서**: 3개 (1,957줄)

---

## 원본 프로젝트

- **GitHub**: https://github.com/spring-projects/spring-petclinic
- **Stars**: 7.6k
- **목적**: Spring Boot 공식 샘플 애플리케이션
- **도메인**: 동물병원 관리 시스템 (Owner → Pet → Visit, Vet ↔ Specialty)

---

## 핵심 발견 사항 (15개 패턴)

### 1. 도메인별 패키지 구조 ⭐
```
src/main/java/org/springframework/samples/petclinic/
├── owner/          # Owner 도메인 (Entity/Repository/Controller 함께)
│   ├── Owner.java
│   ├── Pet.java
│   ├── Visit.java
│   ├── OwnerRepository.java
│   ├── OwnerController.java
│   ├── PetController.java
│   └── VisitController.java
├── vet/            # Vet 도메인
│   ├── Vet.java
│   ├── Specialty.java
│   ├── VetRepository.java
│   └── VetController.java
├── model/          # 공통 베이스 클래스 (BaseEntity, Person 등)
└── system/         # 시스템 설정 (Cache, i18n)
```

**왜 이렇게 나눴는가?**
- **도메인 중심**: 계층별(controller/, service/, repository/) 대신 도메인별
- **응집도**: 한 도메인의 모든 코드가 한 곳에 (변경 시 한 폴더만 수정)
- **FastAPI Best Practices와 동일한 구조!**

---

### 2. JPA Entity 상속 계층

```
BaseEntity (@MappedSuperclass)
    ├─ NamedEntity (name 필드 추가)
    │   ├─ PetType
    │   └─ Specialty
    └─ Person (firstName, lastName 필드)
        ├─ Owner
        └─ Vet
```

**왜 @MappedSuperclass인가?**
- `@Entity`가 아님 → 테이블 생성 안 함
- 자식 클래스에게 공통 필드만 상속 (id, isNew() 메소드)
- **설계 원칙**: 추상화는 테이블이 아니라 코드 재사용 목적

**왜 Integer id인가?** (`model/BaseEntity.java:35`)
```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private @Nullable Integer id;

public boolean isNew() {
    return this.id == null;  // 신규 엔티티 판별
}
```
- **주석**: "Long이 프로덕션에서는 권장되지만, Integer로도 충분" (샘플 앱)
- **isNew() 활용**: `owner.addPet()`, `pet.addVisit()`에서 신규 여부 판단

---

### 3. 관계 설정의 설계 원칙

#### 3.1 @OneToMany: Owner → Pet (소유 관계)

**코드** (`owner/Owner.java:66-69`)
```java
@OneToMany(cascade = CascadeType.ALL, fetch = FetchType.EAGER)
@JoinColumn(name = "owner_id")
@OrderBy("name")
private final List<Pet> pets = new ArrayList<>();
```

**왜 이렇게 설정했는가?**

| 설정 | 선택 | 이유 |
|------|------|------|
| `CascadeType.ALL` | 전체 전이 | Owner 삭제 시 Pet도 함께 삭제 (생명주기 결합) |
| `FetchType.EAGER` | 즉시 로딩 | Pet 없는 Owner는 의미 없음, N+1 허용 (데이터 적음) |
| `@OrderBy("name")` | DB 정렬 | Java sort()보다 DB 정렬이 효율적 |
| `List` | 순서 유지 | Pet 등록순, 이름순 표시 필요 |
| `final` | 불변 참조 | null 방지, 재할당 방지 |

**왜 @JoinColumn인가?**
- Pet 테이블에 `owner_id` FK 생성
- 양방향 아님 (Pet → Owner 참조 없음)
- **단방향 설계**: 필요한 방향만 구현 (복잡도 감소)

#### 3.2 @ManyToMany: Vet ↔ Specialty (공유 관계)

**코드** (`vet/Vet.java:48-51`)
```java
@ManyToMany(fetch = FetchType.EAGER)
@JoinTable(name = "vet_specialties",
    joinColumns = @JoinColumn(name = "vet_id"),
    inverseJoinColumns = @JoinColumn(name = "specialty_id"))
private @Nullable Set<Specialty> specialties;
```

**왜 @OneToMany가 아닌가?**
- 한 수의사 → 여러 전문분야 (치과 + 외과)
- 한 전문분야 → 여러 수의사가 공유
- Specialty는 독립적 존재 (수의사 삭제해도 전문분야는 유지)

**왜 Set을 사용했는가?**
- **중복 방지**: 같은 전문분야를 두 번 추가할 수 없음
- **@ManyToMany 권장**: JPA에서 Set 사용 권장
- **순서 불필요**: 전문분야에 우선순위 없음

**왜 getSpecialtiesInternal() 패턴인가?** (`vet/Vet.java:53-64`)
```java
protected Set<Specialty> getSpecialtiesInternal() {
    if (this.specialties == null) {
        this.specialties = new HashSet<>();  // Lazy init
    }
    return this.specialties;  // 내부 수정용
}

@XmlElement
public List<Specialty> getSpecialties() {
    return getSpecialtiesInternal().stream()
        .sorted(Comparator.comparing(NamedEntity::getName))  // 정렬
        .collect(Collectors.toList());  // 새 List 생성
}
```

**설계 의도**:
1. **getSpecialtiesInternal()**: 내부 수정용 (addSpecialty에서 사용)
2. **getSpecialties()**: 외부 노출용 (정렬된 List, 불변성)
3. **매번 새 List 생성**: 외부 수정이 내부 Set에 영향 없음

---

### 4. Spring Data JPA의 힘

#### 4.1 메소드 이름 → 쿼리 자동 생성

**코드** (`owner/OwnerRepository.java:34, 46`)
```java
public interface OwnerRepository extends JpaRepository<Owner, Integer> {

    Page<Owner> findByLastNameStartingWith(String lastName, Pageable pageable);
    //          ─────────────────────────── ↓
    // SELECT * FROM owners WHERE last_name LIKE ?% ORDER BY ... LIMIT ... OFFSET ...
}
```

**파싱 규칙**:
- `findBy` → SELECT
- `LastName` → last_name 컬럼
- `StartingWith` → LIKE ?%
- `Pageable` → ORDER BY, LIMIT, OFFSET 자동 생성

**코드 감소**:
- 전통적 방법: 400줄 (JDBC 보일러플레이트)
- Spring Data JPA: **3줄** (인터페이스 선언)

#### 4.2 Repository 추상화 수준 선택

**OwnerRepository** (모든 CRUD 필요):
```java
extends JpaRepository<Owner, Integer>  // save, delete, findById 등 20+ 메소드
```

**VetRepository** (조회만 필요) (`vet/VetRepository.java:38, 44-56`):
```java
extends Repository<Vet, Integer>  // 기본 메소드 0개

Collection<Vet> findAll();  // 필요한 메소드만 명시
Page<Vet> findAll(Pageable pageable);
```

**왜 차이가 있는가?**
- Owner: 생성/수정/삭제 필요 (고객 관리)
- Vet: 조회만 필요 (수의사는 관리자만 추가)
- **ISP (Interface Segregation Principle)**: 필요한 메소드만 노출

---

### 5. @ModelAttribute 패턴 (Controller 코드 감소)

**코드** (`owner/OwnerController.java:55-59`)
```java
@ModelAttribute("owner")
public Owner findOwner(@PathVariable(required = false) Integer ownerId) {
    return ownerId == null ? new Owner()
        : owners.findById(ownerId).orElseThrow(...);
}
```

**동작 방식**:
1. 모든 @RequestMapping 메소드 **실행 전** 호출
2. 반환값을 Model에 "owner" 이름으로 자동 추가
3. 모든 메소드에서 `Owner owner` 파라미터로 자동 주입

**예시** (`owner/PetController.java:67-70`):
```java
@PostMapping("/pets/new")
public String processCreationForm(
    Owner owner,  // @ModelAttribute로 자동 주입!
    @Valid Pet pet,
    BindingResult result
) { ... }
```

**왜 사용하는가?**
- **DRY 원칙**: Owner 로딩 코드를 한 곳에만 작성
- **코드 감소**: 각 메소드마다 `findById()` 호출 제거
- **일관성**: null 체크 로직 중앙화

---

### 6. Validation 3단계 전략

#### 6.1 Bean Validation (필드 레벨)

**코드** (`owner/Owner.java:49-55`)
```java
@NotBlank
private String firstName;

@NotBlank
private String lastName;

@NotBlank
@Pattern(regexp = "\\d{10}", message = "{telephone.invalid}")
private String telephone;
```

- **간단한 검증**: 필드 단위 (빈 값, 패턴)
- **선언적**: 어노테이션으로 표현

#### 6.2 Custom Validator (조건부 검증)

**코드** (`owner/PetValidator.java:33-38`)
```java
@Override
public void validate(Object obj, Errors errors) {
    Pet pet = (Pet) obj;

    // 신규 Pet만 type 필수
    if (pet.isNew() && pet.getType() == null) {
        errors.rejectValue("type", REQUIRED, REQUIRED);
    }

    // birthDate는 미래 날짜 불가
    if (pet.getBirthDate() != null && pet.getBirthDate().isAfter(LocalDate.now())) {
        errors.rejectValue("birthDate", "typeMismatch.birthDate");
    }
}
```

**주석** (31-32줄):
> Bean Validation보다 Java로 정의하는 게 더 쉽습니다

**왜 Custom Validator인가?**
- **조건부 검증**: `pet.isNew()`일 때만 type 필수
- **복잡한 로직**: 미래 날짜 체크 (어노테이션으로 표현 어려움)

#### 6.3 Controller 레벨 검증

**코드** (`owner/PetController.java:68-78`)
```java
@PostMapping("/pets/new")
public String processCreationForm(Owner owner, @Valid Pet pet, BindingResult result, Model model) {

    if (StringUtils.hasText(pet.getName()) && pet.isNew()
        && owner.getPet(pet.getName()) != null) {
        result.rejectValue("name", "duplicate", "already exists");
    }

    new PetValidator().validate(pet, result);  // Custom Validator 호출

    if (result.hasErrors()) {
        return "pets/createOrUpdatePetForm";  // 에러 시 폼 재표시
    }

    owner.addPet(pet);  // 성공 시 추가
    return "redirect:/owners/{ownerId}";
}
```

**3단계 검증 순서**:
1. **@Valid**: Bean Validation 실행 (필드 검증)
2. **Custom Validator**: 조건부 검증 (신규/수정 구분)
3. **Controller 로직**: 중복 이름 체크 (Owner 내에서 Pet 이름 중복 방지)

---

### 7. 보안: Mass Assignment Attack 방지

**코드** (`owner/OwnerController.java:62-66`)
```java
@InitBinder
public void setAllowedFields(WebDataBinder dataBinder) {
    dataBinder.setDisallowedFields("id");  // id 필드 바인딩 차단
}
```

**공격 시나리오**:
```html
<!-- 정상 폼 -->
<form action="/owners/1" method="POST">
  <input name="firstName" value="John">
  <input name="lastName" value="Doe">
</form>

<!-- 악의적 조작 -->
<form action="/owners/1" method="POST">
  <input name="firstName" value="John">
  <input name="id" value="999">  <!-- 다른 Owner의 id로 변경 시도 -->
</form>
```

**@InitBinder의 역할**:
- 모든 데이터 바인딩 전에 실행
- `id` 필드를 차단 → 폼에서 전송해도 무시됨
- **예방적 보안**: URL의 ownerId만 신뢰

---

### 8. Aggregate Root 패턴 (DDD)

**코드** (`owner/Owner.java:92-104`)
```java
// Pet 추가는 Owner를 통해서만
public void addPet(Pet pet) {
    if (pet.isNew()) {  // 신규 Pet만 추가
        getPetsInternal().add(pet);
    }
}

// Visit 추가도 Owner가 담당
public void addVisit(Integer petId, Visit visit) {
    Pet pet = getPet(petId);  // Pet 소유권 검증
    if (pet == null) {
        throw new IllegalArgumentException("Invalid Pet ID");
    }
    pet.addVisit(visit);  // 검증 후 추가
}
```

**왜 직접 Pet에 접근하지 않는가?**

**잘못된 방식**:
```java
Pet pet = petRepository.findById(petId);  // Owner 무시
pet.addVisit(visit);  // Owner가 Pet을 소유하는지 검증 안 함
```

**올바른 방식** (`owner/VisitController.java:67-71`):
```java
owner.addVisit(petId, visit);  // Owner가 Pet 소유권 검증
```

**설계 원칙**:
- **Owner = Aggregate Root**: 모든 변경은 Owner를 통해
- **데이터 일관성**: Owner가 Pet 소유 여부 검증
- **비즈니스 규칙**: "다른 사람의 Pet에 Visit 추가 불가"

---

### 9. Cache 전략

#### 9.1 왜 Vet만 캐싱하는가?

**코드** (`vet/VetRepository.java:44-56`)
```java
@Transactional(readOnly = true)
@Cacheable("vets")
Collection<Vet> findAll();

@Transactional(readOnly = true)
@Cacheable("vets")
Page<Vet> findAll(Pageable pageable);
```

**캐싱 대상 선정 기준**:

| Entity | 캐싱 여부 | 이유 |
|--------|-----------|------|
| **Vet** | ✅ | 읽기 전용 (거의 변경 없음), 조인 비용 높음 (@ManyToMany) |
| Owner | ❌ | 자주 변경됨 (주소, 전화번호), 캐시 무효화 부담 |
| Pet | ❌ | Visit 추가로 자주 변경 |

**왜 Repository에 @Cacheable을 붙였는가?**
- Service Layer 없음 → Repository가 가장 낮은 레벨
- **DB 접근 전 차단**: 쿼리 실행 자체를 막음

#### 9.2 JCache 설정

**코드** (`system/CacheConfiguration.java:31-38`)
```java
@Configuration(proxyBeanMethods = false)  // 성능 최적화
@EnableCaching
class CacheConfiguration {

    @Bean
    public JCacheManagerCustomizer petclinicCacheConfigurationCustomizer() {
        return cm -> cm.createCache("vets", cacheConfiguration());
    }
}
```

**왜 JCache (JSR-107)인가?**
- **표준 API**: 구현체 교체 가능 (EhCache ↔ Hazelcast)
- **프로그래밍 방식**: "vets" 캐시를 코드로 명시적 생성
- **타입 안전**: application.yml보다 안전

**왜 proxyBeanMethods = false인가?**
- **Lite 모드**: CGLIB 프록시 생성 안 함 (성능 향상)
- **Bean 간 의존성 없음**: 이 설정 클래스는 Bean이 하나뿐

---

### 10. 다국어 지원 (i18n)

#### 10.1 Locale 저장 전략

**코드** (`system/WebConfiguration.java:32-37`)
```java
@Bean
public LocaleResolver localeResolver() {
    SessionLocaleResolver resolver = new SessionLocaleResolver();
    resolver.setDefaultLocale(Locale.ENGLISH);
    return resolver;
}
```

**전략 비교**:

| 전략 | 저장 위치 | 장점 | 단점 |
|------|-----------|------|------|
| **SessionLocaleResolver** | 서버 세션 | 사용자별 유지, 안전 | 서버 메모리 사용 |
| CookieLocaleResolver | 클라이언트 쿠키 | 서버 부담 없음 | 쿠키 조작 가능 |
| AcceptHeaderLocaleResolver | HTTP Header | 브라우저 자동 | 변경 불가 |

**왜 Session을 선택했는가?**
- **UX**: 한 번 설정하면 세션 동안 유지
- **보안**: 서버에서 관리 (쿠키 조작 불가)

#### 10.2 언어 전환 Interceptor

**코드** (`system/WebConfiguration.java:44-58`)
```java
@Bean
public LocaleChangeInterceptor localeChangeInterceptor() {
    LocaleChangeInterceptor interceptor = new LocaleChangeInterceptor();
    interceptor.setParamName("lang");  // URL: ?lang=de
    return interceptor;
}

@Override
public void addInterceptors(InterceptorRegistry registry) {
    registry.addInterceptor(localeChangeInterceptor());  // 모든 요청에 적용
}
```

**동작 흐름**:
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
- **횡단 관심사**: 모든 페이지에서 `?lang=` 사용 가능

---

### 11. JSON/XML 응답: Wrapper 객체

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

**주석** (`VetController.java:46-47, 71-72`):
> Vet 컬렉션 대신 'Vets' 객체를 반환하는 이유:
> Object-XML/JSON 매핑을 더 간단하게 만들기 위해

**왜 List&lt;Vet&gt;를 직접 반환하지 않는가?**

**JSON 응답 비교**:
```json
// Wrapper 사용
{
  "vetList": [...]
}

// List 직접 반환
[...]  // 루트가 배열 (메타데이터 추가 불가)
```

**장점**:
1. **확장성**: 나중에 totalCount, lastUpdated 등 추가 가능
2. **XML 호환성**: XML은 루트 요소 필수 (@XmlRootElement)
3. **명확한 의도**: "이것은 Vet 목록입니다" 표현

---

### 12. Pageable 자동 페이징

**코드** (`vet/VetController.java:45-51, 63-66`)
```java
@GetMapping("/vets.html")
public String showVetList(@RequestParam(defaultValue = "1") int page, Model model) {
    Page<Vet> paginated = findPaginated(page);
    // ...
}

private Page<Vet> findPaginated(int page) {
    int pageSize = 5;
    Pageable pageable = PageRequest.of(page - 1, pageSize);  // 0-based
    return vetRepository.findAll(pageable);  // 자동 페이징
}
```

**Spring Data JPA의 마법**:
- `findAll(Pageable)` → `SELECT ... LIMIT 5 OFFSET 0`
- `Page<Vet>` 객체에 포함된 정보:
  - `getContent()`: 현재 페이지 데이터
  - `getTotalPages()`: 전체 페이지 수
  - `getTotalElements()`: 전체 항목 수

**왜 page - 1인가?**
- URL: `/vets.html?page=1` (사용자는 1부터 시작)
- Pageable: `PageRequest.of(0, 5)` (0-based)
- **사용자 친화적 URL** ↔ **내부 0-based 인덱스** 변환

---

### 13. @DateTimeFormat (날짜 바인딩)

**코드** (`owner/Pet.java:49-51`)
```java
@DateTimeFormat(pattern = "yyyy-MM-dd")
private LocalDate birthDate;
```

**왜 필요한가?**
- 폼에서 전송: `birthDate=2023-01-15` (문자열)
- Java 타입: `LocalDate` (객체)
- **자동 변환**: Spring이 pattern에 맞춰 파싱

**없으면 어떻게 되는가?**
```
Failed to convert value of type 'java.lang.String' to required type 'java.time.LocalDate'
```

---

### 14. Flash Attributes (리다이렉트 후 메시지)

**코드** (`owner/OwnerController.java:93-99`)
```java
@PostMapping("/owners/new")
public String processCreationForm(@Valid Owner owner, BindingResult result,
                                  RedirectAttributes redirectAttributes) {
    // ...
    this.owners.save(owner);
    redirectAttributes.addFlashAttribute("message", "New Owner Created");
    return "redirect:/owners/" + owner.getId();
}
```

**문제 상황**:
```
POST /owners/new → Owner 생성 성공 → redirect:/owners/1
↑
메시지를 어떻게 전달? (POST와 GET은 다른 요청)
```

**Flash Attributes의 역할**:
1. **리다이렉트 전**: Session에 일시 저장
2. **리다이렉트 후**: Session에서 꺼내서 Model에 추가
3. **한 번만 사용**: 다음 요청에서는 자동 삭제

**왜 URL 파라미터가 아닌가?**
```
redirect:/owners/1?message=New+Owner+Created  // URL 노출, 인코딩 필요
```
Flash는 URL에 노출되지 않음 (보안, 깔끔함)

---

### 15. package-level 접근 제어

**코드** (`vet/VetController.java:35-36`)
```java
@Controller
class VetController {  // public이 아님!
```

**왜 public이 아닌가?**
- **캡슐화**: 패키지 외부에서 직접 접근 불필요
- **Spring 관리**: @Controller 어노테이션으로 Bean 등록 (public 불필요)
- **설계 원칙**: 필요한 만큼만 노출 (Least Privilege)

**모든 Controller/Repository가 package-private**:
- `OwnerController` (package-private)
- `OwnerRepository` (public - 인터페이스는 관례상 public)
- `VetController` (package-private)

---

## 분석 문서

1. **[01-detailed-class-analysis.md](./analysis/01-detailed-class-analysis.md)** (708줄)
   - Entity 상속 계층 (BaseEntity → Person/NamedEntity)
   - 관계 설정 (@OneToMany, @ManyToOne)
   - 컬렉션 설계 (List vs Set, final)
   - 왜 이렇게 설계했는지 상세 설명

2. **[02-repository-controller-patterns.md](./analysis/02-repository-controller-patterns.md)** (685줄)
   - Spring Data JPA 메소드 이름 파싱
   - @ModelAttribute 패턴
   - Validation 3단계 전략
   - Mass Assignment 방지
   - Aggregate Root 패턴
   - Flash Attributes

3. **[03-advanced-patterns.md](./analysis/03-advanced-patterns.md)** (564줄)
   - @ManyToMany 관계 설계
   - getSpecialtiesInternal() 패턴
   - Repository 추상화 수준 (JpaRepository vs Repository)
   - JCache 설정 (proxyBeanMethods = false)
   - i18n 설정 (SessionLocaleResolver, Interceptor)
   - Wrapper 객체 (Vets)

---

## 즉시 적용 가능한 패턴

### 1. 프로젝트 구조
```
✅ 도메인별 패키지 (계층별 X)
✅ Entity/Repository/Controller를 도메인 폴더에 함께
✅ 공통 클래스는 별도 패키지 (model/)
✅ 인프라 설정은 system/ 패키지
```

### 2. Entity 설계
```java
✅ @MappedSuperclass로 공통 필드 상속
✅ 관계 설정 시 Cascade, Fetch 전략 명확히
✅ 컬렉션은 final로 선언 (null 방지)
✅ getInternal() 패턴으로 내부/외부 getter 분리
```

### 3. Repository
```java
✅ Spring Data JPA 인터페이스만 선언 (구현 0줄)
✅ 메소드 이름으로 쿼리 생성 (findByLastNameStartingWith)
✅ 필요한 메소드만 노출 (Repository vs JpaRepository 선택)
✅ 읽기 전용 데이터는 @Cacheable
```

### 4. Controller
```java
✅ @ModelAttribute로 공통 객체 로딩
✅ @InitBinder로 Mass Assignment 방지
✅ Flash Attributes로 리다이렉트 후 메시지
✅ @RequestMapping 클래스 레벨 활용
```

### 5. Validation
```java
✅ 간단한 검증: Bean Validation (@NotBlank, @Pattern)
✅ 조건부 검증: Custom Validator
✅ 비즈니스 검증: Controller 로직
✅ 에러는 BindingResult로 수집 후 일괄 처리
```

### 6. 보안/설정
```java
✅ setDisallowedFields("id")로 필드 바인딩 차단
✅ Aggregate Root 패턴으로 일관성 보장
✅ Cache는 읽기 전용 데이터만
✅ i18n은 Interceptor + SessionLocaleResolver
```

---

## 예상 효과

| 지표 | 기존 방식 | Spring PetClinic 패턴 | 효과 |
|------|-----------|----------------------|------|
| **CRUD 코드량** | 400줄 (JDBC) | 3줄 (인터페이스) | **99% 감소** |
| **Repository 구현** | SQL 직접 작성 | 메소드 이름만 | **0줄** |
| **Validation** | if문 중첩 | 어노테이션 + Custom | **가독성 10배** |
| **페이징** | 수동 계산 | Pageable 자동 | **10줄 → 1줄** |
| **다국어** | 하드코딩 | messages.properties | **확장 무한대** |
| **캐시** | 수동 구현 | @Cacheable | **3줄** |
| **보안** | try-catch | @InitBinder | **선언적** |

---

## 핵심 학습 포인트

### 1. "왜"를 이해하는 것이 중요
- `CascadeType.ALL`: **왜?** → 생명주기 결합 (Owner 삭제 시 Pet도 삭제)
- `FetchType.EAGER`: **왜?** → Pet 없는 Owner는 의미 없음 (N+1 허용)
- `Set vs List`: **왜?** → 중복 방지 vs 순서 유지
- `Repository vs JpaRepository`: **왜?** → 필요한 메소드만 노출 (ISP)

### 2. Spring의 Convention over Configuration
- 메소드 이름 → 쿼리 자동 생성
- @ModelAttribute → 자동 주입
- Pageable → 자동 페이징
- **학습 곡선은 높지만, 익숙해지면 생산성 폭발**

### 3. 도메인 중심 설계
- 계층별이 아닌 도메인별 패키지
- Aggregate Root 패턴
- **비즈니스 로직과 코드 구조의 일치**

### 4. 선언적 프로그래밍
- Validation: 어노테이션
- Cache: @Cacheable
- Transaction: @Transactional
- i18n: Interceptor
- **"무엇을"만 선언, "어떻게"는 프레임워크가**

---

## 다른 프로젝트와의 비교

| 특징 | **Spring PetClinic** | **FastAPI Best Practices** | **dbt-jaffle-shop** |
|------|----------------------|----------------------------|---------------------|
| **구조** | 도메인별 패키지 | 도메인별 패키지 | Staging/Marts 레이어 |
| **핵심** | Spring Data JPA 마법 | Async/Sync 선택 | CTE + ref() |
| **Validation** | Bean + Custom | Pydantic | SQL 제약조건 |
| **공통점** | **도메인 중심 설계** | **도메인 중심 설계** | **계층 분리** |

**일관된 원칙**: 모든 프로젝트가 "도메인 중심" 설계를 강조!

---

**분석 완료**: 2025-01-18
**총 문서**: 3개 (1,957줄)
**총 패턴**: 15개 (즉시 적용 가능)
**분석 시간**: ~2시간
