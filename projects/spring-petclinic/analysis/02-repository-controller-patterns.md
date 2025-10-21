# Spring PetClinic Repository & Controller 패턴 상세 분석

> **목표**: Spring Data JPA와 MVC 패턴의 "왜"와 "어떻게"

---

## Part 1: Spring Data JPA Repository

### OwnerRepository - 인터페이스만으로 완성

**파일**: `owner/OwnerRepository.java` (64줄)

```java
public interface OwnerRepository extends JpaRepository<Owner, Integer> {

    Page<Owner> findByLastNameStartingWith(String lastName, Pageable pageable);

    Optional<Owner> findById(Integer id);
}
```

**전체 코드 64줄, 실제 구현 0줄!**

---

### 핵심 질문: 구현체는 어디에?

**답**: Spring Data JPA가 **런타임에 자동 생성**

```java
// Spring이 하는 일 (내부 동작)
class OwnerRepositoryImpl implements OwnerRepository {

    @Override
    public Page<Owner> findByLastNameStartingWith(String lastName, Pageable pageable) {
        // SQL 자동 생성!
        // SELECT * FROM owners WHERE last_name LIKE :lastName% ...
    }

    @Override
    public Optional<Owner> findById(Integer id) {
        // JpaRepository에서 상속받아서 이미 구현됨
    }
}
```

---

### Method Naming Convention 마법

#### findByLastNameStartingWith 해부

```java
Page<Owner> findByLastNameStartingWith(String lastName, Pageable pageable);
```

**Spring이 이름을 파싱**:
```
findBy           → SELECT
LastName         → last_name 컬럼
StartingWith     → LIKE :param%
```

**생성된 SQL**:
```sql
SELECT *
FROM owners
WHERE last_name LIKE ?
LIMIT 5 OFFSET 0  -- Pageable
```

---

#### 네이밍 규칙 예시

| 메소드 이름 | 생성되는 SQL |
|------------|-------------|
| `findByLastName(String name)` | `WHERE last_name = ?` |
| `findByLastNameStartingWith(String name)` | `WHERE last_name LIKE ?%` |
| `findByLastNameContaining(String name)` | `WHERE last_name LIKE %?%` |
| `findByLastNameAndFirstName(String last, String first)` | `WHERE last_name = ? AND first_name = ?` |
| `findByAgeBetween(int min, int max)` | `WHERE age BETWEEN ? AND ?` |
| `findByOrderByLastNameAsc()` | `ORDER BY last_name ASC` |

**참고**: https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#jpa.query-methods

---

### 왜 인터페이스만?

**Before (전통적인 방식)**:
```java
@Repository
public class OwnerRepositoryImpl {

    @PersistenceContext
    private EntityManager em;

    public Owner findById(Integer id) {
        return em.find(Owner.class, id);
    }

    public List<Owner> findByLastName(String lastName) {
        String jpql = "SELECT o FROM Owner o WHERE o.lastName LIKE :lastName";
        TypedQuery<Owner> query = em.createQuery(jpql, Owner.class);
        query.setParameter("lastName", lastName + "%");
        return query.getResultList();
    }

    public void save(Owner owner) {
        if (owner.getId() == null) {
            em.persist(owner);
        } else {
            em.merge(owner);
        }
    }

    // 20개 메소드 × 20줄 = 400줄 코드!
}
```

**After (Spring Data JPA)**:
```java
public interface OwnerRepository extends JpaRepository<Owner, Integer> {
    Page<Owner> findByLastNameStartingWith(String lastName, Pageable pageable);
    // 끝! 3줄로 완성
}
```

**효과**:
- 코드: 400줄 → 3줄 (99% 감소!)
- 버그: N개 → 0개
- 테스트: 필요 → 불필요 (Spring이 검증)

---

### JpaRepository 상속으로 얻는 것

```java
extends JpaRepository<Owner, Integer>
```

**무료로 제공되는 메소드** (20개+):
```java
// CRUD
save(Owner)
saveAll(Iterable<Owner>)
findById(Integer)
findAll()
findAllById(Iterable<Integer>)
count()
delete(Owner)
deleteById(Integer)
deleteAll()
existsById(Integer)

// Paging
findAll(Pageable)
findAll(Sort)

// Batch
flush()
saveAndFlush(Owner)
deleteInBatch(Iterable<Owner>)
// ...
```

**Controller에서 바로 사용**:
```java
owners.findById(1);           // SELECT * FROM owners WHERE id = 1
owners.findAll();             // SELECT * FROM owners
owners.save(owner);           // INSERT or UPDATE
owners.count();               // SELECT COUNT(*) FROM owners
```

---

### Pageable 자동 처리

```java
Page<Owner> findByLastNameStartingWith(String lastName, Pageable pageable);
```

**Controller에서 사용**:
```java
Pageable pageable = PageRequest.of(page - 1, 5);  // page=1, size=5
Page<Owner> results = owners.findByLastNameStartingWith("Smith", pageable);

// Page 객체가 제공하는 것
results.getContent();         // List<Owner> (실제 데이터)
results.getTotalElements();   // 전체 개수 (SELECT COUNT(*))
results.getTotalPages();      // 전체 페이지 수
results.hasNext();            // 다음 페이지 있나?
results.hasPrevious();        // 이전 페이지 있나?
```

**SQL**:
```sql
-- 데이터 조회
SELECT * FROM owners WHERE last_name LIKE 'Smith%' LIMIT 5 OFFSET 0;

-- 개수 조회 (자동!)
SELECT COUNT(*) FROM owners WHERE last_name LIKE 'Smith%';
```

**2개 쿼리 자동 실행!**

---

## Part 2: Controller 패턴

### @ModelAttribute의 마법

#### OwnerController 분석

```java
@Controller
class OwnerController {

    @ModelAttribute("owner")
    public Owner findOwner(@PathVariable(required = false) Integer ownerId) {
        return ownerId == null ? new Owner()
            : owners.findById(ownerId).orElseThrow(...);
    }

    @GetMapping("/owners/{ownerId}")
    public ModelAndView showOwner(@PathVariable int ownerId) {
        // owner 파라미터 없어도 됨!
        // @ModelAttribute가 자동 주입!
    }
}
```

---

### 왜 @ModelAttribute?

**문제**: 모든 메소드에서 Owner 조회 반복

**Before**:
```java
@GetMapping("/owners/{ownerId}")
public String showOwner(@PathVariable int ownerId, Model model) {
    Owner owner = owners.findById(ownerId).orElseThrow(...);  // 중복!
    model.addAttribute("owner", owner);
    return "owners/ownerDetails";
}

@GetMapping("/owners/{ownerId}/edit")
public String editOwner(@PathVariable int ownerId, Model model) {
    Owner owner = owners.findById(ownerId).orElseThrow(...);  // 또 중복!
    model.addAttribute("owner", owner);
    return "owners/editForm";
}

@PostMapping("/owners/{ownerId}/edit")
public String updateOwner(@PathVariable int ownerId, ...) {
    Owner owner = owners.findById(ownerId).orElseThrow(...);  // 또또 중복!
    // ...
}
```

**After**:
```java
@ModelAttribute("owner")
public Owner findOwner(@PathVariable(required = false) Integer ownerId) {
    return ownerId == null ? new Owner() : owners.findById(ownerId).orElseThrow(...);
}

// 모든 메소드에서 자동 주입!
@GetMapping("/owners/{ownerId}")
public String showOwner() {  // owner 파라미터 필요 없음!
    return "owners/ownerDetails";  // 템플릿에서 ${owner} 사용 가능
}
```

---

### @ModelAttribute 실행 순서

```java
@Controller
class PetController {

    // 1️⃣ 모든 핸들러 메소드 실행 전 먼저 실행!
    @ModelAttribute("types")
    public Collection<PetType> populatePetTypes() {
        return types.findPetTypes();  // ["dog", "cat", "bird"]
    }

    @ModelAttribute("owner")
    public Owner findOwner(@PathVariable int ownerId) {
        return owners.findById(ownerId).orElseThrow(...);
    }

    @ModelAttribute("pet")
    public Pet findPet(@PathVariable(required = false) Integer petId) {
        return petId == null ? new Pet() : owner.getPet(petId);
    }

    // 2️⃣ 핸들러 메소드 실행
    @GetMapping("/owners/{ownerId}/pets/new")
    public String initCreationForm() {
        // 이미 Model에 types, owner, pet 있음!
        return "pets/createOrUpdatePetForm";
    }
}
```

**흐름**:
```
HTTP GET /owners/1/pets/new
    ↓
1. populatePetTypes() 실행 → Model에 "types" 추가
2. findOwner(1) 실행 → Model에 "owner" 추가
3. findPet(null) 실행 → Model에 "pet" 추가 (new Pet())
4. initCreationForm() 실행
    ↓
5. Thymeleaf 템플릿 렌더링
   - ${types} 사용 가능 (드롭다운)
   - ${owner} 사용 가능
   - ${pet} 사용 가능 (빈 객체)
```

---

### @RequestMapping 클래스 레벨

```java
@Controller
@RequestMapping("/owners/{ownerId}")  // ⭐ 클래스 레벨!
class PetController {

    @GetMapping("/pets/new")  // 실제: /owners/{ownerId}/pets/new
    public String initCreationForm() { ... }

    @PostMapping("/pets/new")  // 실제: /owners/{ownerId}/pets/new
    public String processCreationForm() { ... }

    @GetMapping("/pets/{petId}/edit")  // 실제: /owners/{ownerId}/pets/{petId}/edit
    public String initUpdateForm() { ... }
}
```

**왜?**
- Pet은 Owner에 종속
- URL 구조: `/owners/{ownerId}/pets/...`
- 모든 메소드에서 `ownerId` 필요
- 한 번만 선언!

---

### @InitBinder - Validation 설정

```java
@InitBinder("pet")
public void initPetBinder(WebDataBinder dataBinder) {
    dataBinder.setValidator(new PetValidator());
}

@InitBinder("owner")
public void initOwnerBinder(WebDataBinder dataBinder) {
    dataBinder.setDisallowedFields("id");  // id 변조 방지!
}
```

---

#### setDisallowedFields("id") - 보안 패턴

**문제**: Mass Assignment 공격

```html
<!-- Form -->
<form method="POST" action="/owners/1/edit">
    <input name="firstName" value="John">
    <input name="lastName" value="Doe">
    <!-- 해커가 추가! -->
    <input name="id" value="999">  ⚠️
</form>
```

**Without setDisallowedFields**:
```java
@PostMapping("/owners/{ownerId}/edit")
public String update(@Valid Owner owner) {
    owners.save(owner);  // owner.id = 999 저장! (다른 Owner 덮어씀!)
}
```

**With setDisallowedFields**:
```java
dataBinder.setDisallowedFields("id");

// Form에서 id 전송되어도 무시!
owner.getId();  // null (바인딩 안 됨)
```

**추가 설정**:
```java
owner.setId(ownerId);  // Controller에서 명시적 설정
owners.save(owner);    // 안전!
```

---

### BindingResult - Validation 에러 처리

```java
@PostMapping("/owners/{ownerId}/pets/new")
public String processCreationForm(
    Owner owner,
    @Valid Pet pet,           // Validation 실행
    BindingResult result,     // 에러 저장
    RedirectAttributes redirectAttributes
) {
    // 1. Custom Validation (비즈니스 로직)
    if (owner.getPet(pet.getName(), true) != null) {
        result.rejectValue("name", "duplicate", "already exists");
    }

    if (pet.getBirthDate().isAfter(LocalDate.now())) {
        result.rejectValue("birthDate", "typeMismatch.birthDate");
    }

    // 2. 에러 체크
    if (result.hasErrors()) {
        return "pets/createOrUpdatePetForm";  // 다시 폼으로
    }

    // 3. 저장
    owner.addPet(pet);
    owners.save(owner);

    // 4. 성공 메시지
    redirectAttributes.addFlashAttribute("message", "New Pet has been Added");
    return "redirect:/owners/{ownerId}";
}
```

---

#### Validation 순서

```
1. @Valid Pet pet
   ↓
   PetValidator.validate() 실행
   - name 필수
   - type 필수
   - birthDate 필수
   ↓
2. BindingResult에 에러 저장
   ↓
3. Custom Validation
   - 중복 이름 체크
   - 미래 날짜 체크
   ↓
4. result.hasErrors()?
   YES → 폼으로 돌아감
   NO  → 저장
```

---

### RedirectAttributes - Flash Message

```java
redirectAttributes.addFlashAttribute("message", "New Pet has been Added");
return "redirect:/owners/{ownerId}";
```

**왜?**
- `redirect:` → 새 HTTP 요청
- Model은 소멸
- Flash: **1회용** 세션 저장

**흐름**:
```
POST /owners/1/pets/new
    ↓ (저장)
redirectAttributes.addFlashAttribute("message", "...")
    ↓
redirect:/owners/1
    ↓ (새 요청)
GET /owners/1
    ↓
Thymeleaf에서 ${message} 사용 가능
    ↓
(새로고침)
GET /owners/1
    ↓
${message} 사라짐!  ← Flash는 1회용
```

---

### RESTful URL 설계

```
GET    /owners/new              → 생성 폼
POST   /owners/new              → 생성 처리

GET    /owners?lastName=Smith   → 검색
GET    /owners                  → 목록

GET    /owners/1                → 상세보기
GET    /owners/1/edit           → 수정 폼
POST   /owners/1/edit           → 수정 처리

GET    /owners/1/pets/new       → Pet 생성 폼
POST   /owners/1/pets/new       → Pet 생성 처리

GET    /owners/1/pets/5/edit    → Pet 수정 폼
POST   /owners/1/pets/5/edit    → Pet 수정 처리

GET    /owners/1/pets/5/visits/new   → Visit 생성 폼
POST   /owners/1/pets/5/visits/new   → Visit 생성 처리
```

**원칙**:
1. 리소스 계층: `/owners/{ownerId}/pets/{petId}/visits/...`
2. GET/POST 구분: 폼/처리
3. `/new` vs `/{id}/edit`

---

## Part 3: VisitController - Aggregate 패턴

```java
@Controller
class VisitController {

    @ModelAttribute("visit")
    public Visit loadPetWithVisit(
        @PathVariable int ownerId,
        @PathVariable int petId,
        Map<String, Object> model
    ) {
        Owner owner = owners.findById(ownerId).orElseThrow(...);
        Pet pet = owner.getPet(petId);

        model.put("pet", pet);
        model.put("owner", owner);

        Visit visit = new Visit();
        pet.addVisit(visit);  // Pet에 미리 추가!
        return visit;
    }

    @PostMapping("/owners/{ownerId}/pets/{petId}/visits/new")
    public String processNewVisitForm(
        @ModelAttribute Owner owner,
        @PathVariable int petId,
        @Valid Visit visit,
        BindingResult result
    ) {
        if (result.hasErrors()) {
            return "pets/createOrUpdateVisitForm";
        }

        owner.addVisit(petId, visit);  // ⭐ Owner를 통해 추가!
        owners.save(owner);
        return "redirect:/owners/{ownerId}";
    }
}
```

---

### 왜 Owner를 통해 Visit 추가?

**DDD Aggregate 패턴**:
```
Owner (Aggregate Root)
  └─ Pet (Entity)
      └─ Visit (Entity)
```

**흐름**:
```java
owner.addVisit(petId, visit)
    ↓
pet = owner.getPet(petId)  // Owner가 Pet 소유 검증
    ↓
pet.addVisit(visit)
```

**장점**:
1. 일관성: Owner가 보장
2. 보안: 다른 Owner의 Pet에 Visit 추가 방지
3. 캡슐화: 내부 구조 감춤

---

## Part 4: VetController - @ResponseBody

```java
@Controller
class VetController {

    // HTML 응답
    @GetMapping("/vets.html")
    public String showVetList(@RequestParam(defaultValue = "1") int page, Model model) {
        Vets vets = new Vets();
        Page<Vet> paginated = findPaginated(page);
        vets.getVetList().addAll(paginated.toList());
        return addPaginationModel(page, paginated, model);
    }

    // JSON 응답
    @GetMapping("/vets")
    public @ResponseBody Vets showResourcesVetList() {
        Vets vets = new Vets();
        vets.getVetList().addAll(vetRepository.findAll());
        return vets;  // JSON으로 자동 변환!
    }
}
```

---

### @ResponseBody - JSON API

```java
@ResponseBody Vets showResourcesVetList()
```

**HTTP Response**:
```json
{
  "vetList": [
    {
      "id": 1,
      "firstName": "James",
      "lastName": "Carter",
      "specialties": [
        {"id": 1, "name": "radiology"}
      ]
    },
    {
      "id": 2,
      "firstName": "Helen",
      "lastName": "Leary",
      "specialties": [
        {"id": 2, "name": "surgery"}
      ]
    }
  ]
}
```

**자동 변환**:
- Jackson 라이브러리 사용
- Vets 객체 → JSON

---

## 핵심 요약

### Repository 패턴
1. **인터페이스만** 작성 → Spring이 구현
2. **Method Naming** → SQL 자동 생성
3. **Pageable** → 페이징 자동 처리

### Controller 패턴
1. **@ModelAttribute** → 공통 데이터 자동 주입
2. **@InitBinder** → Validation, 보안 설정
3. **BindingResult** → 에러 처리
4. **RedirectAttributes** → Flash 메시지
5. **RESTful URL** → 리소스 계층 구조

### 보안
1. **setDisallowedFields** → Mass Assignment 방지
2. **Aggregate Root** → Owner를 통한 접근만 허용

---

## 다음 문서

**03-advanced-patterns.md**:
- @ManyToMany (Vet-Specialty)
- Custom Validation
- Pagination 상세
- Thymeleaf 통합
