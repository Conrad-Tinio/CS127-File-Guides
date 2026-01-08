# LoanTrack Finance Manager - Technical Guide

## Overview

LoanTrack is a **full-stack web application** built with:

- **Backend**: Java 21 + Spring Boot 3.2.0 (REST API)
- **Frontend**: React 18 + TypeScript + Vite
- **Database**: PostgreSQL (via Supabase)
- **Architecture Pattern**: MVC (Model-View-Controller) / Layered Architecture

---

## IMPORTANT FEATURE 1: THREE TRANSACTION TYPES

### Technical Implementation

The application supports three distinct transaction types, each with different business logic:

```java
// Model: TransactionType.java
public enum TransactionType {
    STRAIGHT_EXPENSE,    // One-time payment
    INSTALLMENT_EXPENSE, // Scheduled payments
    GROUP_EXPENSE        // Shared among group
}
```

### How It Works:

#### 1. STRAIGHT EXPENSE

- **Model**: Simple `Entry` entity
- **Validation**: Only allows CASH payment method
- **Payment Flow**: Single payment updates `amountRemaining` directly

```java
// EntryService.java - Line 143-147
if (request.getTransactionType() == TransactionType.STRAIGHT_EXPENSE &&
    request.getPaymentMethod() != null &&
    request.getPaymentMethod() != PaymentMethod.CASH) {
    throw new IllegalArgumentException("Straight payment entries only allow CASH");
}
```

#### 2. INSTALLMENT EXPENSE

- **Model**: `Entry` + `InstallmentPlan` + `InstallmentTerm[]`
- **Database Relationships**:
  - Entry 1:1 InstallmentPlan
  - InstallmentPlan 1:N InstallmentTerm
- **Automatic Term Generation**: Creates terms based on frequency and day

```java
// EntryService.java - generateInstallmentTerms()
private void generateInstallmentTerms(InstallmentPlan plan, String paymentFrequencyDay) {
    LocalDate currentDate = calculateFirstDueDate(startDate, frequency, paymentFrequencyDay);

    for (int i = 1; i <= plan.getPaymentTerms(); i++) {
        InstallmentTerm term = new InstallmentTerm();
        term.setTermNumber(i);
        term.setDueDate(currentDate);
        term.setTermStatus(InstallmentStatus.NOT_STARTED);
        installmentTermRepository.save(term);

        // Calculate next due date
        currentDate = frequency == WEEKLY
            ? calculateNextWeeklyDate(currentDate, paymentFrequencyDay)
            : calculateNextMonthlyDate(currentDate, paymentFrequencyDay);
    }
}
```

**Key Algorithm**: Payment frequency with specific day selection

- **Monthly**: Stores day (1-28) in JSON format in `notes` field (no schema changes!)
- **Weekly**: Stores day of week (SUNDAY-SATURDAY)
- Calculates due dates using Java 8 `LocalDate` API with `TemporalAdjusters`

#### 3. GROUP EXPENSE

- **Model**: `Entry` + `PaymentAllocation[]`
- **Split Logic**: Each group member gets an allocated amount
- **Payment Tracking**: Tracks individual member payments separately

```java
// PaymentAllocationService.java
// Allocations must sum to entry amount
BigDecimal total = allocations.stream()
    .map(Allocation::getAmount)
    .reduce(BigDecimal.ZERO, BigDecimal::add);

if (total.compareTo(entry.getAmountBorrowed()) != 0) {
    throw new IllegalArgumentException("Total allocations must equal entry amount");
}
```

---

## IMPORTANT FEATURE 2: LAYERED ARCHITECTURE (Backend)

### Architecture Pattern: Repository-Service-Controller

```
┌─────────────────────────────────────────────────┐
│           FRONTEND (React/TypeScript)           │
│  - Pages (Components)                           │
│  - Services (API Calls)                         │
│  - Types (Interfaces)                           │
└──────────────────┬──────────────────────────────┘
                   │ HTTP/REST API
┌──────────────────▼──────────────────────────────┐
│         CONTROLLER LAYER (@RestController)      │
│  - EntryController, PaymentController, etc.     │
│  - Handles HTTP requests/responses              │
│  - Input validation                             │
└──────────────────┬──────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────┐
│          SERVICE LAYER (@Service)               │
│  - EntryService, PaymentService, etc.           │
│  - Business logic                               │
│  - Transaction management (@Transactional)      │
│  - DTO conversions                              │
└──────────────────┬──────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────┐
│       REPOSITORY LAYER (JPA Repository)         │
│  - EntryRepository, PersonRepository, etc.      │
│  - Database operations                          │
│  - Extends JpaRepository<Entity, UUID>          │
└──────────────────┬──────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────┐
│              DATABASE (PostgreSQL)              │
│  - Tables: entry, installment_plan, etc.        │
│  - Relationships via foreign keys               │
└─────────────────────────────────────────────────┘
```

### Code Example: Request Flow

```java
// 1. CONTROLLER receives HTTP request
@RestController
@RequestMapping("/api/entries")
public class EntryController {
    @Autowired
    private EntryService entryService;

    @PostMapping
    public ResponseEntity<EntryDTO> createEntry(@RequestBody CreateEntryRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(entryService.createEntry(request, null));
    }
}

// 2. SERVICE processes business logic
@Service
@Transactional  // Entire method runs in one transaction
public class EntryService {
    @Autowired
    private EntryRepository entryRepository;

    public EntryDTO createEntry(CreateEntryRequest request, MultipartFile proof) {
        // Validation
        validateRequest(request);

        // Create entity
        Entry entry = new Entry();
        entry.setEntryName(request.getEntryName());
        // ... set other fields

        // Save to database
        Entry saved = entryRepository.save(entry);

        // Convert to DTO for response
        return convertToDTO(saved);
    }
}

// 3. REPOSITORY handles database operations
public interface EntryRepository extends JpaRepository<Entry, UUID> {
    Optional<Entry> findByReferenceId(String referenceId);
    // JPA automatically implements: save(), findById(), findAll(), delete()
}
```

---

## IMPORTANT FEATURE 3: DATA TRANSFER OBJECTS (DTOs)

### Why DTOs?

**Problem**: Entity models have database-specific details (JPA annotations, lazy loading) that shouldn't be exposed to frontend.

**Solution**: DTOs (Data Transfer Objects) provide clean, serializable data structures.

### Implementation:

```java
// MODEL (Database entity) - Entry.java
@Entity
@Table(name = "entry")
public class Entry {
    @Id
    private UUID entryId;

    @ManyToOne(fetch = FetchType.LAZY)  // Database relationship
    private Person borrowerPerson;

    // ... JPA annotations, lazy loading, etc.
}

// DTO (For API) - EntryDTO.java
@Data  // Lombok generates getters/setters
public class EntryDTO {
    private UUID entryId;
    private String borrowerPersonId;    // Just the ID, not full object
    private String borrowerPersonName;  // Flattened for frontend
    // ... no JPA annotations, simple POJO
}

// SERVICE converts Entity → DTO
private EntryDTO convertToDTO(Entry entry) {
    EntryDTO dto = new EntryDTO();
    dto.setEntryId(entry.getEntryId());
    dto.setBorrowerPersonId(entry.getBorrowerPerson().getPersonId());
    dto.setBorrowerPersonName(entry.getBorrowerPerson().getFullName());
    // ... converts all fields
    return dto;
}
```

**Benefits**:

- ✅ Frontend gets clean JSON
- ✅ No lazy loading issues
- ✅ Can customize data structure
- ✅ Separation of concerns

---

## IMPORTANT FEATURE 4: RELATIONAL DATABASE DESIGN

### Entity Relationships:

```java
// One-to-Many: Entry → PaymentEntries
@OneToMany(mappedBy = "entry", cascade = CascadeType.ALL)
private List<PaymentEntry> paymentEntries;

// One-to-One: Entry → InstallmentPlan
@OneToOne(mappedBy = "entry", cascade = CascadeType.ALL)
private InstallmentPlan installmentPlan;

// Many-to-One: Entry → Person (Borrower)
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "borrower_person_id")
private Person borrowerPerson;
```

### Cascade Operations:

```java
cascade = CascadeType.ALL
// Means: When you delete an Entry, automatically delete:
// - InstallmentPlan
// - InstallmentTerms
// - PaymentEntries
// - PaymentAllocations
// - Attachments
```

### Lazy Loading:

```java
@ManyToOne(fetch = FetchType.LAZY)  // Don't load until accessed
private Person borrowerPerson;

// Usage in service:
Person borrower = entry.getBorrowerPerson();  // Database query happens here
String name = borrower.getFullName();          // Uses loaded data
```

**Why LAZY?**

- Performance: Don't load related data unless needed
- Avoid N+1 query problem
- Only load what the API endpoint needs

---

## IMPORTANT FEATURE 5: INSTALLMENT PLAN DATE CALCULATION

### Algorithm Explanation:

This is one of the most complex features - calculating due dates with specific day requirements.

```java
private LocalDate calculateFirstDueDate(LocalDate startDate,
                                       PaymentFrequency frequency,
                                       String paymentFrequencyDay) {
    if (frequency == PaymentFrequency.MONTHLY) {
        int dayOfMonth = Integer.parseInt(paymentFrequencyDay);  // e.g., "15"

        // Ensure day is valid for the month (handle month-end)
        int actualDay = Math.min(dayOfMonth, startDate.lengthOfMonth());
        LocalDate firstDueDate = startDate.withDayOfMonth(actualDay);

        // If start date is after target day, move to next month
        if (startDate.getDayOfMonth() > dayOfMonth) {
            firstDueDate = startDate.plusMonths(1)
                .withDayOfMonth(Math.min(dayOfMonth,
                    startDate.plusMonths(1).lengthOfMonth()));
        }
        return firstDueDate;
    }
    // ... weekly logic
}
```

### Key Technical Details:

1. **Storage without Schema Change**:
   - Stores `paymentFrequencyDay` in JSON format in existing `notes` field
   - Backward compatible with old entries

```java
// Store as JSON
Map<String, String> notesData = new HashMap<>();
notesData.put("paymentFrequencyDay", "15");
notesData.put("userNotes", "Regular monthly payment");
plan.setNotes(objectMapper.writeValueAsString(notesData));
```

2. **Date Handling**:

   - Uses Java 8 `LocalDate` for type-safe date operations
   - `TemporalAdjusters.nextOrSame()` for day-of-week calculations
   - Handles edge cases (month-end, leap years)

3. **Term Generation Loop**:

```java
for (int i = 1; i <= plan.getPaymentTerms(); i++) {
    // Create term with calculated due date
    term.setDueDate(currentDate);

    // Calculate NEXT term's due date
    currentDate = calculateNextDate(currentDate, paymentFrequencyDay);
}
```

---

## IMPORTANT FEATURE 6: FRONTEND ARCHITECTURE

### React Component Structure:

```
src/
├── pages/              # Main page components
│   ├── CreateEntryPage.tsx
│   ├── EntryDetailPage.tsx
│   └── ...
├── components/         # Reusable components
│   ├── PersonSelector.tsx
│   ├── GroupSelector.tsx
│   └── Toast.tsx
├── services/           # API communication
│   └── api.ts
├── types/              # TypeScript interfaces
│   └── index.ts
└── contexts/           # React Context for state
    └── UserContext.tsx
```

### API Service Pattern:

```typescript
// services/api.ts
const api = axios.create({
  baseURL: "http://localhost:8080/api",
  headers: { "Content-Type": "application/json" },
});

// Request interceptor - adds user context to every request
api.interceptors.request.use((config) => {
  const selectedUserName = localStorage.getItem("selectedUserName");
  config.headers["X-Selected-User-Name"] = selectedUserName || "default";
  return config;
});

// API methods
export const entryApi = {
  getAll: () => api.get<Entry[]>("/entries"),
  create: (entry: CreateEntryRequest) => api.post<Entry>("/entries", entry),
  // ... other methods
};
```

### Component Pattern:

```typescript
// CreateEntryPage.tsx
export default function CreateEntryPage() {
  const [formData, setFormData] = useState({...})
  const [submitting, setSubmitting] = useState(false)

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()
    setSubmitting(true)

    try {
      const response = await entryApi.create(formData)
      navigate(`/entries/${response.data.entryId}`)
    } catch (error) {
      // Handle error
    } finally {
      setSubmitting(false)
    }
  }

  return <form onSubmit={handleSubmit}>...</form>
}
```

**Key Patterns**:

- ✅ **Hooks**: `useState`, `useEffect` for state management
- ✅ **Async/Await**: Handle API calls asynchronously
- ✅ **TypeScript**: Type safety for API responses
- ✅ **React Router**: Client-side routing

---

## IMPORTANT FEATURE 7: USER CONTEXT & MULTI-USER SUPPORT

### Backend: User Context

```java
// UserContext.java - Thread-local storage
public class UserContext {
    private static final ThreadLocal<String> currentUserName = new ThreadLocal<>();

    public static void setCurrentUserName(String userName) {
        currentUserName.set(userName);
    }

    public static String getCurrentUserName() {
        return currentUserName.get();
    }
}

// UserConfig.java - Intercepts HTTP requests
@Component
public class UserConfig implements Filter {
    public void doFilter(...) {
        String userName = request.getHeader("X-Selected-User-Name");
        UserContext.setCurrentUserName(userName);
        // ... continues filter chain
    }
}

// EntryService uses context
private Person getOrCreateCurrentUser() {
    String userName = UserContext.getCurrentUserName();
    return personRepository.findByFullName(userName)
        .orElseGet(() -> {
            Person user = new Person();
            user.setFullName(userName);
            return personRepository.save(user);
        });
}
```

### Frontend: User Context

```typescript
// UserContext.tsx
export function UserProvider({ children }) {
  const [selectedUser, setSelectedUser] = useState(null);

  useEffect(() => {
    const userId = localStorage.getItem("selectedUserId");
    const userName = localStorage.getItem("selectedUserName");
    // ... load user
  }, []);

  return (
    <UserContext.Provider value={{ selectedUser, setSelectedUser }}>
      {children}
    </UserContext.Provider>
  );
}
```

**How It Works**:

1. Frontend stores selected user in `localStorage`
2. Every API request includes `X-Selected-User-Name` header
3. Backend extracts header and stores in `ThreadLocal`
4. Service layer uses context to filter entries
5. Only shows entries where user is borrower OR lender

---

## IMPORTANT FEATURE 8: TRANSACTION MANAGEMENT

### Spring `@Transactional`:

```java
@Service
@Transactional  // All methods run in transactions
public class EntryService {

    public EntryDTO createEntry(CreateEntryRequest request, MultipartFile proof) {
        Entry entry = new Entry();
        entryRepository.save(entry);  // Part of transaction

        createInstallmentPlan(entry, request);  // Also part of transaction

        // If ANY step fails, ALL changes are rolled back
        return convertToDTO(entry);
    }
}
```

**Benefits**:

- ✅ **Atomicity**: All operations succeed or all fail
- ✅ **Consistency**: Database stays in valid state
- ✅ **Rollback**: Automatic on exceptions

**Example**:

```java
// If installment plan creation fails:
throw new IllegalArgumentException("Invalid payment frequency");
// → Entry is NOT saved (transaction rolled back)
// → Database stays consistent
```

---

## IMPORTANT FEATURE 9: FILE UPLOAD HANDLING

### Multi-part Form Data:

```java
// Controller
@PostMapping(consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
public ResponseEntity<EntryDTO> createEntry(
    @RequestPart("request") CreateEntryRequest request,
    @RequestPart(value = "proof", required = false) MultipartFile proof) {

    return ResponseEntity.status(HttpStatus.CREATED)
        .body(entryService.createEntry(request, proof));
}

// Service - Store in two places
if (proof != null && !proof.isEmpty()) {
    // 1. Store in Entry entity (for quick access)
    entry.setReceiptOrProof(proof.getBytes());

    // 2. Store in Attachment table (for better organization)
    Attachment attachment = new Attachment();
    attachment.setEntry(saved);
    attachment.setFileData(proof.getBytes());
    attachment.setOriginalFilename(proof.getOriginalFilename());
    attachment.setContentType(proof.getContentType());
    attachmentRepository.save(attachment);
}
```

### Frontend Upload:

```typescript
// CreateEntryPage.tsx
const handleSubmit = async (e: React.FormEvent) => {
  const formData = new FormData();
  formData.append(
    "request",
    new Blob([JSON.stringify(entry)], { type: "application/json" })
  );
  formData.append("proof", proofFile); // File object

  const response = await entryApi.createWithProof(formData, {
    headers: { "Content-Type": "multipart/form-data" },
  });
};
```

---

## IMPORTANT FEATURE 10: REFERENCE ID GENERATION

### Algorithm:

```java
// ReferenceIdGenerator.java
public static String generateReferenceId(Person borrower, Person lender) {
    String borrowerInitials = extractInitials(borrower.getFullName());
    String lenderInitials = extractInitials(lender.getFullName());
    return borrowerInitials + lenderInitials;  // e.g., "JD" + "AB" = "JDAB"
}

private static String extractInitials(String fullName) {
    // Handles multiple formats:
    // "John Doe" → "JD"
    // "Doe, John" → "DJ"
    // "Doe, John, M" → "DJM"

    String[] parts = fullName.split(",");
    if (parts.length >= 2) {
        // "Surname, First Name" format
        return parts[0].charAt(0) + parts[1].charAt(0);
    } else {
        // "First Name Last Name" format
        String[] nameParts = fullName.split("\\s+");
        return nameParts[0].charAt(0) + nameParts[1].charAt(0);
    }
}
```

**Uniqueness Handling**:

```java
String baseRefId = entry.getReferenceId();
String refId = baseRefId;
int counter = 1;
while (entryRepository.existsByReferenceId(refId)) {
    refId = baseRefId + counter;  // "JDAB1", "JDAB2", etc.
    counter++;
}
```

---

## IMPORTANT FEATURE 11: VALIDATION & ERROR HANDLING

### Backend Validation:

```java
// EntryService.java - Multiple validation layers
public EntryDTO createEntry(CreateEntryRequest request, MultipartFile proof) {
    // 1. Business Rule Validation
    if (request.getBorrowerPersonId() != null &&
        request.getBorrowerGroupId() != null) {
        throw new IllegalArgumentException("Cannot have both borrower types");
    }

    // 2. Cross-field Validation
    if (request.getBorrowerPersonId() != null &&
        request.getBorrowerPersonId().equals(request.getLenderPersonId())) {
        throw new IllegalArgumentException("Borrower and lender cannot be same");
    }

    // 3. Constraint Validation
    if (request.getTransactionType() == TransactionType.INSTALLMENT_EXPENSE &&
        request.getBorrowerGroupId() != null) {
        throw new IllegalArgumentException("Installments don't support groups");
    }

    // ... continue with creation
}
```

### Global Exception Handler:

```java
// GlobalExceptionHandler.java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<Map<String, String>> handleIllegalArgumentException(
            IllegalArgumentException ex) {
        Map<String, String> error = new HashMap<>();
        error.put("error", ex.getMessage());
        return ResponseEntity.badRequest().body(error);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<Map<String, String>> handleException(Exception ex) {
        Map<String, String> error = new HashMap<>();
        error.put("error", "An unexpected error occurred: " + ex.getMessage());
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
    }
}
```

### Frontend Error Handling:

```typescript
// CreateEntryPage.tsx
try {
  const response = await entryApi.create(formData);
  // Success
} catch (error: any) {
  const errorMessage =
    error?.response?.data?.error ||
    error?.response?.data?.message ||
    error?.message ||
    "Unknown error";

  // Show toast for specific errors
  if (errorMessage.includes("Borrower and lender")) {
    setToast({
      message: "Borrower and lender cannot be the same person",
      type: "error",
    });
  } else {
    setFormError(errorMessage); // General form error
  }
}
```

---

## IMPORTANT FEATURE 12: ENUM TYPES FOR TYPE SAFETY

### Backend Enums:

```java
// TransactionType.java
public enum TransactionType {
    STRAIGHT_EXPENSE,
    INSTALLMENT_EXPENSE,
    GROUP_EXPENSE
}

// Usage in Entity
@Enumerated(EnumType.STRING)  // Store as "STRAIGHT_EXPENSE" in DB
@Column(name = "transaction_type", nullable = false)
private TransactionType transactionType;

// Type-safe comparison
if (entry.getTransactionType() == TransactionType.INSTALLMENT_EXPENSE) {
    // Handle installment
}
```

### Frontend Enums:

```typescript
// types/index.ts
export enum TransactionType {
  STRAIGHT_EXPENSE = "STRAIGHT_EXPENSE",
  INSTALLMENT_EXPENSE = "INSTALLMENT_EXPENSE",
  GROUP_EXPENSE = "GROUP_EXPENSE",
}

// Usage - TypeScript ensures only valid values
const [transactionType, setTransactionType] = useState<TransactionType>(
  TransactionType.STRAIGHT_EXPENSE
);
```

**Benefits**:

- ✅ Compile-time type checking
- ✅ IDE autocomplete
- ✅ Prevents typos
- ✅ Self-documenting code

---

## KEY TECHNICAL PATTERNS

### 1. Dependency Injection (Spring)

```java
@Service
public class EntryService {
    @Autowired  // Spring automatically provides this
    private EntryRepository entryRepository;

    // Spring scans for @Repository and injects it
    // No manual instantiation needed!
}
```

### 2. Repository Pattern

```java
// Extends JpaRepository = Free CRUD operations
public interface EntryRepository extends JpaRepository<Entry, UUID> {
    // Spring generates implementation automatically
    Optional<Entry> findByReferenceId(String referenceId);

    // Can write custom queries:
    @Query("SELECT e FROM Entry e WHERE e.status = :status")
    List<Entry> findByStatus(@Param("status") PaymentStatus status);
}
```

### 3. Builder Pattern (via Lombok)

```java
@Data  // Lombok generates: getters, setters, toString, equals, hashCode
@AllArgsConstructor
@NoArgsConstructor
public class Entry {
    private UUID entryId;
    private String entryName;
    // ...
}
// Without Lombok: Would need 50+ lines of boilerplate code!
```

### 4. DTO Conversion Pattern

```java
// Separate conversion methods
private EntryDTO convertToDTO(Entry entry) { ... }
private Entry convertToEntity(EntryDTO dto) { ... }
private InstallmentPlanDTO convertInstallmentPlanToDTO(InstallmentPlan plan) { ... }

// Keeps entities and DTOs separate
// Allows different representations for same data
```

---

## DATABASE SCHEMA INSIGHTS

### Key Tables:

1. **entry**: Main loan/expense records
2. **installment_plan**: Installment configuration (1:1 with entry)
3. **installment_term**: Individual payment terms (1:N with plan)
4. **payment**: Payment records
5. **payment_entry**: Links payments to entries (many-to-many)
6. **payment_allocation**: Group expense splits
7. **person**: People/contacts
8. **group**: Groups of people
9. **group_member**: Many-to-many: groups ↔ people

### Foreign Key Relationships:

```sql
-- Entry references Person (Borrower & Lender)
entry.borrower_person_id → person.person_id
entry.lender_person_id → person.person_id

-- InstallmentPlan references Entry
installment_plan.entry_id → entry.entry_id

-- InstallmentTerm references InstallmentPlan
installment_term.installment_id → installment_plan.installment_id

-- PaymentEntry links Payment and Entry
payment_entry.entry_id → entry.entry_id
payment_entry.payment_id → payment.payment_id
```

---

## SECURITY CONSIDERATIONS

### Current Implementation:

1. **CORS Configuration**: Only allows `localhost:5173`
2. **User Context**: Thread-local storage (not shared between requests)
3. **Input Validation**: All inputs validated in service layer
4. **Transaction Safety**: Database operations are atomic

### Potential Improvements:

- Add authentication/authorization
- Add rate limiting
- Encrypt sensitive data
- Add audit logging

---

## PERFORMANCE OPTIMIZATIONS

### 1. Lazy Loading

```java
@ManyToOne(fetch = FetchType.LAZY)  // Only load when accessed
private Person borrowerPerson;
```

### 2. Eager Loading When Needed

```java
// EntryService - Explicitly load relationships
Entry entry = entryRepository.findById(id)
    .orElseThrow(...);

// Load installment plan if needed
if (entry.getTransactionType() == TransactionType.INSTALLMENT_EXPENSE) {
    InstallmentPlan plan = installmentPlanRepository
        .findByEntry_EntryId(entry.getEntryId())
        .orElse(null);
    // Use plan...
}
```

### 3. Batch Operations

```java
// Generate all terms in one transaction
for (int i = 1; i <= plan.getPaymentTerms(); i++) {
    InstallmentTerm term = new InstallmentTerm();
    // ... set fields
    installmentTermRepository.save(term);  // Batch insert
}
```

---

## TESTING STRATEGY (Not Yet Implemented)

### Unit Tests:

- Service layer business logic
- Date calculation algorithms
- Validation rules

### Integration Tests:

- Repository operations
- Controller endpoints
- Full request/response cycles

### Frontend Tests:

- Component rendering
- Form validation
- API integration

---

## CONCLUSION

This project demonstrates:

1. ✅ **Clean Architecture**: Separation of concerns (Controller → Service → Repository)
2. ✅ **Type Safety**: Enums and TypeScript prevent errors
3. ✅ **Data Integrity**: Transactions ensure consistency
4. ✅ **Flexibility**: Three transaction types with shared infrastructure
5. ✅ **Maintainability**: DTOs, clear naming, documentation
6. ✅ **Modern Stack**: Java 21, Spring Boot 3, React 18, TypeScript

**Key Learning Points**:

- Spring Boot simplifies enterprise Java development
- JPA/Hibernate handles ORM automatically
- React makes frontend state management easier
- TypeScript catches errors at compile-time
- RESTful API design enables frontend/backend separation

---

## NEXT STEPS TO EXPLORE

1. **Add Authentication**: JWT tokens, Spring Security
2. **Add Caching**: Redis for frequently accessed data
3. **Add Pagination**: For large entry lists
4. **Add Search**: Full-text search for entries
5. **Add Export**: CSV/PDF generation
6. **Add Notifications**: Email/SMS for due dates
7. **Add Analytics**: Charts and financial insights
8. **Add Mobile App**: React Native version

Happy coding! 🚀
