# Spring PetClinic 클래스별 상세 분석

> **목표**: 각 클래스가 왜 그렇게 설계되었는지 파악

---

## Model 패키지 - 상속 계층 구조

### 설계 철학: DRY (Don't Repeat Yourself)

```
BaseEntity (id, isNew())
    ├── NamedEntity (name)
    │       └── Pet
    │       └── PetType
    │       └── Specialty
    │
    └── Person (firstName, lastName)
            └── Owner
            └── Vet
```

---

## 1. BaseEntity - 모든 Entity의 뿌리

**파일**: `model/BaseEntity.java` (52줄)

### 왜 이 클래스를 만들었나?

**문제**:
```java
// BaseEntity 없다면?
class Owner {
    private Integer id;  // 중복!
    public boolean isNew() { return id == null; }  // 중복!
}

class Pet {
    private Integer id;  // 또 중복!
    public boolean isNew() { return id == null; }  // 또 중복!
}

class Vet {
    private Integer id;  // 또또 중복!
    public boolean isNew() { return id == null; }  // 또또 중복!
}
```

**해결**:
```java
@MappedSuperclass  // ⭐ JPA: 직접 테이블 안 만들고 상속용
public class BaseEntity implements Serializable {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private @Nullable Integer id;  // 모든 Entity가 공유

    public boolean isNew() {
        return this.id == null;  // 한 곳에만 정의!
    }
}
```

---

### 핵심 결정사항

#### 1. @MappedSuperclass 사용

**왜?**
- `@Entity`가 아님 → `base_entity` 테이블 생성 안 함
- 자식 클래스가 필드 상속
- DB 테이블: `owners`, `pets`, `vets` (base_entity 없음)

**대안과 비교**:
| 방식 | 테이블 | 장점 | 단점 |
|------|--------|------|------|
| `@MappedSuperclass` | 자식마다 | 단순 | 다형성 쿼리 불가 |
| `@Inheritance` (JOINED) | base + 자식 | 정규화 | Join 비용 |
| `@Inheritance` (SINGLE_TABLE) | 1개 | 빠름 | null 많음 |

**PetClinic 선택**: @MappedSuperclass
- 이유: 다형성 필요 없음 (Owner와 Pet을 함께 조회할 일 없음)

---

#### 2. isNew() 메소드

```java
public boolean isNew() {
    return this.id == null;
}
```

**왜 필요한가?**

**사용처 1**: Owner.addPet()
```java
public void addPet(Pet pet) {
    if (pet.isNew()) {  // 새 Pet만 추가
        getPets().add(pet);
    }
}
```

**사용처 2**: Owner.getPet()
```java
public Pet getPet(String name, boolean ignoreNew) {
    for (Pet pet : getPets()) {
        if (compName.equalsIgnoreCase(name)) {
            if (!ignoreNew || !pet.isNew()) {  // 저장된 것만
                return pet;
            }
        }
    }
    return null;
}
```

**핵심**: 영속화 전후 구분
- `id == null` → DB에 없음 (새 객체)
- `id != null` → DB에 있음 (저장됨)

---

#### 3. Integer vs Long

```java
private @Nullable Integer id;  // ⚠️ Long 아니고 Integer?
```

**왜 Integer?**
- 샘플 애플리케이션 → 데이터 적음
- Integer 범위: 21억 (충분)
- Long은 과함

**실무 추천**: Long 사용
```java
private @Nullable Long id;  // 실무는 Long!
```

---

## 2. NamedEntity - 이름 있는 Entity

**파일**: `model/NamedEntity.java` (53줄)

### 왜 이 클래스를 만들었나?

**문제**: Pet, PetType, Specialty 모두 `name` 필드 필요
```java
// NamedEntity 없다면?
class Pet {
    private String name;  // 중복!
}

class PetType {
    private String name;  // 또 중복!
}

class Specialty {
    private String name;  // 또또 중복!
}
```

**해결**:
```java
@MappedSuperclass
public class NamedEntity extends BaseEntity {

    @Column(name = "name")
    @NotBlank  // Validation
    private @Nullable String name;

    @Override
    public String toString() {
        return (name != null) ? name : "<null>";
    }
}
```

---

### 핵심 결정사항

#### 1. Person vs NamedEntity 분리

**왜 Person 따로 만들었나?**

```
BaseEntity
    ├── NamedEntity (name)           # Pet, PetType용
    └── Person (firstName, lastName) # Owner, Vet용
```

**이유**:
- Pet: `name` (단일 이름)
- Owner/Vet: `firstName` + `lastName` (성명)
- 용도가 다름!

**잘못된 설계**:
```java
// ❌ 나쁜 예
class Person extends NamedEntity {
    // name을 firstName으로 억지로 사용?
}
```

---

#### 2. toString() 오버라이드

```java
@Override
public String toString() {
    return (name != null) ? name : "<null>";
}
```

**왜?**
- Thymeleaf 템플릿에서 `${pet.type}` 출력 시
- `toString()` 자동 호출 → "dog", "cat" 출력
- 없으면 `Pet@1a2b3c4d` (객체 주소)

---

## 3. Person - 사람 Entity

**파일**: `model/Person.java` (56줄)

### 왜 Person을 만들었나?

**문제**: Owner와 Vet 모두 `firstName`, `lastName` 필요

**해결**:
```java
@MappedSuperclass
public class Person extends BaseEntity {

    @Column(name = "first_name")  // DB 컬럼명
    @NotBlank
    private @Nullable String firstName;

    @Column(name = "last_name")
    @NotBlank
    private @Nullable String lastName;
}
```

---

### 핵심 결정사항

#### 1. @Column(name = "first_name")

**왜 명시?**
```java
@Column(name = "first_name")  // DB: first_name
private String firstName;      // Java: firstName
```

**기본 동작**:
- 명시 안 하면: `firstName` → `first_name` (자동 변환)
- 하지만 **명시적으로 쓰는 게 좋은 습관**

**이유**:
1. DB 네이밍 컨벤션 명확 (snake_case)
2. Java 네이밍 컨벤션 명확 (camelCase)
3. 나중에 바꿀 때 안전

---

## 4. Owner - 반려동물 주인

**파일**: `owner/Owner.java` (178줄)

### Entity 설계

```java
@Entity
@Table(name = "owners")  // DB 테이블명
public class Owner extends Person {

    @Column(name = "address")
    @NotBlank
    private @Nullable String address;

    @Column(name = "telephone")
    @NotBlank
    @Pattern(regexp = "\\d{10}", message = "{telephone.invalid}")
    private @Nullable String telephone;

    @OneToMany(cascade = CascadeType.ALL, fetch = FetchType.EAGER)
    @JoinColumn(name = "owner_id")
    @OrderBy("name")
    private final List<Pet> pets = new ArrayList<>();
}
```

---

### 핵심 결정사항 분석

#### 1. @Pattern 정규식 Validation

```java
@Pattern(regexp = "\\d{10}", message = "{telephone.invalid}")
private @Nullable String telephone;
```

**왜?**
- Bean Validation API 사용
- 컨트롤러에서 `@Valid` 하면 자동 검증
- 10자리 숫자만 허용

**message = "{telephone.invalid}"**:
- `{}` = 메시지 파일 키
- `messages.properties`에 정의:
  ```properties
  telephone.invalid=Phone number must be 10 digits
  ```
- 다국어 지원 가능!

---

#### 2. @OneToMany 관계 설정

```java
@OneToMany(cascade = CascadeType.ALL, fetch = FetchType.EAGER)
@JoinColumn(name = "owner_id")
@OrderBy("name")
private final List<Pet> pets = new ArrayList<>();
```

**각 옵션 이유**:

**cascade = CascadeType.ALL**:
```java
owner.getPets().add(new Pet());  // Pet 추가
ownerRepository.save(owner);     // Owner 저장
// → Pet도 자동 저장! (cascade)
```

**왜 ALL?**
- Owner 삭제 → Pet도 삭제 (의존 관계)
- Owner 저장 → Pet도 저장
- 라이프사이클 함께

---

**fetch = FetchType.EAGER**:
```java
Owner owner = ownerRepository.findById(1);
// → pets도 함께 로드! (EAGER)

// LAZY였다면?
owner.getPets().size();  // 여기서 추가 쿼리 (N+1 문제)
```

**왜 EAGER?**
- Owner 조회 시 항상 Pets 필요
- 화면에 Pet 목록 표시해야 함
- N+1 문제 피하기

**⚠️ 주의**: EAGER는 신중히 사용
- Owner 1000명 조회 → Pet 10000마리 함께 로드
- 메모리 부족 가능
- PetClinic은 데이터 적어서 OK

---

**@OrderBy("name")**:
```java
for (Pet pet : owner.getPets()) {
    // 이미 이름순 정렬됨!
}
```

**왜?**
- DB에서 정렬해서 가져옴
- Java에서 `Collections.sort()` 필요 없음
- SQL: `ORDER BY name`

---

#### 3. final List<Pet>

```java
private final List<Pet> pets = new ArrayList<>();
```

**왜 final?**
- List 객체 자체는 불변 (재할당 불가)
```java
// ❌ 컴파일 에러
owner.pets = new ArrayList<>();

// ✅ 가능 (내용 변경)
owner.getPets().add(new Pet());
```

**장점**:
- Null 안전
- 항상 빈 리스트 보장
- `owner.getPets().add()` 안전

---

### 비즈니스 로직

#### getPet() 메소드들

```java
// 1. 이름으로 찾기
public Pet getPet(String name) {
    return getPet(name, false);
}

// 2. ID로 찾기
public Pet getPet(Integer id) {
    for (Pet pet : getPets()) {
        if (!pet.isNew()) {  // 저장된 것만
            if (Objects.equals(compId, id)) {
                return pet;
            }
        }
    }
    return null;
}

// 3. 이름 + 옵션
public Pet getPet(String name, boolean ignoreNew) {
    for (Pet pet : getPets()) {
        if (compName.equalsIgnoreCase(name)) {
            if (!ignoreNew || !pet.isNew()) {
                return pet;
            }
        }
    }
    return null;
}
```

---

**왜 3개나?**

**사용처**:
```java
// 1. 화면에서 "Max" 검색
Pet pet = owner.getPet("Max");

// 2. URL에서 /owners/1/pets/5
Pet pet = owner.getPet(5);

// 3. 중복 이름 체크 (새 Pet 제외)
Pet existing = owner.getPet("Max", true);
if (existing != null) {
    errors.reject("duplicate");
}
```

---

#### addVisit() 메소드

```java
public void addVisit(Integer petId, Visit visit) {

    Assert.notNull(petId, "Pet identifier must not be null!");
    Assert.notNull(visit, "Visit must not be null!");

    Pet pet = getPet(petId);

    Assert.notNull(pet, "Invalid Pet identifier!");

    pet.addVisit(visit);
}
```

**왜 Owner에 이 메소드?**
- **Aggregate Root** 패턴 (DDD)
- Owner = 집합의 루트
- Pet, Visit는 Owner를 통해서만 접근

**흐름**:
```
VisitController
    → owner.addVisit(petId, visit)
        → pet = owner.getPet(petId)
        → pet.addVisit(visit)
```

**장점**:
- Owner가 일관성 보장
- Pet이 Owner 소속인지 검증
- 캡슐화

---

## 5. Pet - 반려동물

**파일**: `owner/Pet.java` (87줄)

```java
@Entity
@Table(name = "pets")
public class Pet extends NamedEntity {

    @Column(name = "birth_date")
    @DateTimeFormat(pattern = "yyyy-MM-dd")
    private @Nullable LocalDate birthDate;

    @ManyToOne
    @JoinColumn(name = "type_id")
    private @Nullable PetType type;

    @OneToMany(cascade = CascadeType.ALL, fetch = FetchType.EAGER)
    @JoinColumn(name = "pet_id")
    @OrderBy("date ASC")
    private final Set<Visit> visits = new LinkedHashSet<>();
}
```

---

### 핵심 결정사항

#### 1. @DateTimeFormat

```java
@DateTimeFormat(pattern = "yyyy-MM-dd")
private @Nullable LocalDate birthDate;
```

**왜?**
- HTML form: `<input type="date" name="birthDate">`
- 브라우저 → "2023-01-15" (문자열)
- Spring이 LocalDate로 자동 변환
- 패턴 없으면 변환 실패!

---

#### 2. @ManyToOne (PetType)

```java
@ManyToOne  // Many Pets → One Type
@JoinColumn(name = "type_id")
private @Nullable PetType type;
```

**관계**:
```
Pet         PetType
────────    ────────
Max    ───→ dog
Bella  ───→ cat
Charlie ──→ dog
```

**왜 @ManyToOne?**
- 여러 Pet이 같은 Type 공유
- DB: `pets.type_id` (FK)

---

#### 3. Set vs List for Visits

```java
private final Set<Visit> visits = new LinkedHashSet<>();
// ⚠️ List가 아니고 Set?
```

**왜 Set?**
- Visit 중복 방지
- 같은 날짜 같은 Pet 중복 예약 방지

**왜 LinkedHashSet?**
- HashSet: 순서 없음
- TreeSet: Comparable 필요
- **LinkedHashSet**: 삽입 순서 유지

**@OrderBy("date ASC")**:
- DB에서 이미 정렬
- LinkedHashSet이 순서 유지

---

## 6. PetValidator - 커스텀 Validator

**파일**: `owner/PetValidator.java` (65줄)

```java
public class PetValidator implements Validator {

    @Override
    public void validate(Object obj, Errors errors) {
        Pet pet = (Pet) obj;

        // name 검증
        if (!StringUtils.hasText(name)) {
            errors.rejectValue("name", REQUIRED, REQUIRED);
        }

        // type 검증 (새 Pet만)
        if (pet.isNew() && pet.getType() == null) {
            errors.rejectValue("type", REQUIRED, REQUIRED);
        }

        // birthDate 검증
        if (pet.getBirthDate() == null) {
            errors.rejectValue("birthDate", REQUIRED, REQUIRED);
        }
    }
}
```

---

### 왜 @NotBlank 안 쓰고 Validator?

**주석 설명**:
```java
/**
 * We're not using Bean Validation annotations here because
 * it is easier to define such validation rule in Java.
 */
```

**이유**:

**복잡한 로직**:
```java
if (pet.isNew() && pet.getType() == null) {
    // 새 Pet만 type 필수!
}
```

**@NotNull로는 불가능**:
```java
@NotNull  // ❌ 항상 필수 (조건부 안 됨)
private PetType type;
```

**Validator로만 가능**:
- `pet.isNew()` 체크
- 조건부 검증

---

### 사용법

**PetController에서**:
```java
@InitBinder("pet")
public void initPetBinder(WebDataBinder dataBinder) {
    dataBinder.setValidator(new PetValidator());
}

@PostMapping("/owners/{ownerId}/pets/new")
public String processCreationForm(
    @Valid Pet pet,  // Validator 자동 실행!
    BindingResult result
) {
    if (result.hasErrors()) {
        return "pets/createOrUpdatePetForm";
    }
    // ...
}
```

---

## 핵심 요약

### 클래스 설계 원칙

1. **상속으로 중복 제거**
   - BaseEntity: id, isNew()
   - NamedEntity: name
   - Person: firstName, lastName

2. **@MappedSuperclass vs @Entity**
   - Base/Named/Person: @MappedSuperclass (테이블 없음)
   - Owner/Pet: @Entity (테이블 있음)

3. **관계 설정 신중**
   - EAGER: 항상 필요한 것만
   - CASCADE: 라이프사이클 함께할 때만
   - Set vs List: 중복 여부

4. **Validation 2가지**
   - 간단: @NotBlank, @Pattern
   - 복잡: Custom Validator

5. **비즈니스 로직 위치**
   - Entity에 넣기 (Aggregate Root 패턴)
   - getPet(), addVisit() 등

---

## 다음 문서

**02-repository-controller-analysis.md**:
- Spring Data JPA Repository 마법
- Controller 패턴 상세
- @ModelAttribute 활용
