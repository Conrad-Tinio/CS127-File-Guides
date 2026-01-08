# Backend Functions Overview

This document provides a detailed overview of the backend functions that are primarily and most frequently used across the available features in the project.

## Table of Contents
1. [REST API Controllers](#rest-api-controllers)
2. [Service Layer Functions](#service-layer-functions)
3. [Repository Layer Methods](#repository-layer-methods)
4. [Utility Functions](#utility-functions)
5. [Configuration and Context Management](#configuration-and-context-management)
6. [Data Access Patterns](#data-access-patterns)

---

## REST API Controllers

The backend follows a RESTful API design with controllers organized by entity type. All controllers are located in `com.loantracking.controller` package.

### Entry Controller (`EntryController`)

**Base Path:** `/api/entries`

#### **Most Frequently Used Endpoints:**

1. **`GET /api/entries`** - ⭐⭐⭐ **HIGHEST USAGE**
   - **Method:** `getAllEntries()`
   - **Service:** `entryService.getAllEntries()`
   - **Purpose:** Retrieves all entries related to the current user
   - **Return Type:** `ResponseEntity<List<EntryDTO>>`
   - **Authentication:** Uses `UserContext` to filter entries by current user
   - **Usage:** Called by frontend dashboard, entry lists, payment history

2. **`GET /api/entries/{id}`** - ⭐⭐⭐ **HIGHEST USAGE**
   - **Method:** `getEntryById(@PathVariable UUID id)`
   - **Service:** `entryService.getEntryById(id)`
   - **Purpose:** Retrieves a specific entry by ID
   - **Return Type:** `ResponseEntity<EntryDTO>`
   - **Features:** Includes installment plan, payments, and allocation data in DTO
   - **Usage:** Called when viewing entry details

3. **`POST /api/entries`** - ⭐⭐ **HIGH USAGE**
   - **Method:** `createEntry(@RequestPart CreateEntryRequest, @RequestPart MultipartFile)`
   - **Overload:** `createEntry(@RequestBody CreateEntryRequest)` (JSON-only)
   - **Service:** `entryService.createEntry(request, proof)`
   - **Purpose:** Creates a new entry (supports file upload for proof)
   - **Return Type:** `ResponseEntity<EntryDTO>` (HTTP 201 Created)
   - **Content Types:** 
     - `multipart/form-data` (with proof file)
     - `application/json` (without proof)
   - **Validation:** Validates borrower/lender constraints, transaction types
   - **Features:** 
     - Creates installment plan and terms if INSTALLMENT_EXPENSE
     - Generates unique reference ID
     - Stores proof as attachment

4. **`PUT /api/entries/{id}`** - ⭐ **MEDIUM USAGE**
   - **Method:** `updateEntry(@PathVariable UUID id, @RequestBody CreateEntryRequest)`
   - **Service:** `entryService.updateEntry(id, request)`
   - **Purpose:** Updates entry information
   - **Return Type:** `ResponseEntity<EntryDTO>`
   - **Note:** Does not update transaction type, amounts, or installment plans

5. **`DELETE /api/entries/{id}`** - ⭐ **MEDIUM USAGE**
   - **Method:** `deleteEntry(@PathVariable UUID id)`
   - **Service:** `entryService.deleteEntry(id)`
   - **Purpose:** Deletes an entry and related data
   - **Return Type:** `ResponseEntity<Void>` (HTTP 204 No Content)
   - **Cascade Handling:** Deletes payment allocations and linked payments first

6. **`POST /api/entries/{id}/complete`** - ⭐ **MEDIUM USAGE**
   - **Method:** `completeEntry(@PathVariable UUID id)`
   - **Service:** `entryService.completeEntry(id)`
   - **Purpose:** Marks an entry as fully paid/completed
   - **Return Type:** `ResponseEntity<EntryDTO>`
   - **Features:** Marks unpaid installment terms as PAID when completing installment entries

7. **`POST /api/entries/auto-complete`** - ⭐ **LOW USAGE**
   - **Method:** `autoCompleteEntries()`
   - **Service:** `entryService.autoCompleteEntries()`
   - **Purpose:** Automatically marks entries as PAID if fully paid based on payments
   - **Return Type:** `ResponseEntity<Map<String, Object>>` with count and message
   - **Features:** Recalculates `amountRemaining` from actual payments for accuracy

---

### Person Controller (`PersonController`)

**Base Path:** `/api/persons`

#### **Most Frequently Used Endpoints:**

1. **`GET /api/persons`** - ⭐⭐⭐ **HIGHEST USAGE**
   - **Method:** `getAllPersons()`
   - **Service:** `personService.getAllPersons()`
   - **Purpose:** Retrieves all persons in the system
   - **Return Type:** `ResponseEntity<List<PersonDTO>>`
   - **Usage:** Used by person selectors, people management page

2. **`GET /api/persons/{id}`** - ⭐⭐ **HIGH USAGE**
   - **Method:** `getPersonById(@PathVariable UUID id)`
   - **Service:** `personService.getPersonById(id)`
   - **Purpose:** Retrieves a specific person by ID
   - **Return Type:** `ResponseEntity<PersonDTO>`

3. **`GET /api/persons/search?name={name}`** - ⭐⭐ **HIGH USAGE**
   - **Method:** `searchPersons(@RequestParam String name)`
   - **Service:** `personService.searchPersons(name)`
   - **Purpose:** Searches persons by name (case-insensitive partial match)
   - **Return Type:** `ResponseEntity<List<PersonDTO>>`
   - **Usage:** Used by PersonSelector component for autocomplete

4. **`POST /api/persons`** - ⭐⭐ **HIGH USAGE**
   - **Method:** `createPerson(@RequestBody PersonDTO)`
   - **Service:** `personService.createPerson(personDTO)`
   - **Purpose:** Creates a new person
   - **Return Type:** `ResponseEntity<PersonDTO>` (HTTP 201 Created)
   - **Usage:** Inline creation from selectors

5. **`PUT /api/persons/{id}`** - ⭐ **MEDIUM USAGE**
   - **Method:** `updatePerson(@PathVariable UUID id, @RequestBody PersonDTO)`
   - **Service:** `personService.updatePerson(id, personDTO)`
   - **Purpose:** Updates person information (typically name)
   - **Return Type:** `ResponseEntity<PersonDTO>`

6. **`DELETE /api/persons/{id}`** - ⭐ **MEDIUM USAGE**
   - **Method:** `deletePerson(@PathVariable UUID id)`
   - **Service:** `personService.deletePerson(id)`
   - **Purpose:** Deletes a person
   - **Return Type:** `ResponseEntity<Void>` (HTTP 204 No Content)

---

### Group Controller (`GroupController`)

**Base Path:** `/api/groups`

#### **Most Frequently Used Endpoints:**

1. **`GET /api/groups`** - ⭐⭐⭐ **HIGHEST USAGE**
   - **Method:** `getAllGroups()`
   - **Service:** `groupService.getAllGroups()`
   - **Purpose:** Retrieves all groups with their members
   - **Return Type:** `ResponseEntity<List<GroupDTO>>`
   - **Features:** Includes member list in DTO

2. **`GET /api/groups/{id}`** - ⭐⭐ **HIGH USAGE**
   - **Method:** `getGroupById(@PathVariable UUID id)`
   - **Service:** `groupService.getGroupById(id)`
   - **Purpose:** Retrieves a specific group with members
   - **Return Type:** `ResponseEntity<GroupDTO>`
   - **Usage:** Used when loading group expense entries

3. **`POST /api/groups`** - ⭐⭐ **HIGH USAGE**
   - **Method:** `createGroup(@RequestBody GroupDTO)`
   - **Service:** `groupService.createGroup(groupDTO)`
   - **Purpose:** Creates a new group (with optional initial members)
   - **Return Type:** `ResponseEntity<GroupDTO>` (HTTP 201 Created)
   - **Features:** Validates group name uniqueness

4. **`PUT /api/groups/{id}`** - ⭐ **MEDIUM USAGE**
   - **Method:** `updateGroup(@PathVariable UUID id, @RequestBody GroupDTO)`
   - **Service:** `groupService.updateGroup(id, groupDTO)`
   - **Purpose:** Updates group information (name, preserves existing members)
   - **Return Type:** `ResponseEntity<GroupDTO>`

5. **`DELETE /api/groups/{id}`** - ⭐ **MEDIUM USAGE**
   - **Method:** `deleteGroup(@PathVariable UUID id)`
   - **Service:** `groupService.deleteGroup(id)`
   - **Purpose:** Deletes a group (cascade deletes members)
   - **Return Type:** `ResponseEntity<Void>` (HTTP 204 No Content)

6. **`POST /api/groups/{groupId}/members/{personId}`** - ⭐⭐ **HIGH USAGE**
   - **Method:** `addMember(@PathVariable UUID groupId, @PathVariable UUID personId)`
   - **Service:** `groupService.addMemberToGroup(groupId, personId)`
   - **Purpose:** Adds a person to a group
   - **Return Type:** `ResponseEntity<Void>` (HTTP 200 OK)
   - **Validation:** Prevents duplicate members

7. **`DELETE /api/groups/{groupId}/members/{personId}`** - ⭐ **MEDIUM USAGE**
   - **Method:** `removeMember(@PathVariable UUID groupId, @PathVariable UUID personId)`
   - **Service:** `groupService.removeMemberFromGroup(groupId, personId)`
   - **Purpose:** Removes a person from a group
   - **Return Type:** `ResponseEntity<Void>` (HTTP 204 No Content)

---

### Payment Controller (`PaymentController`)

**Base Path:** `/api/payments`

#### **Most Frequently Used Endpoints:**

1. **`GET /api/payments`** - ⭐⭐ **HIGH USAGE**
   - **Method:** `getAllPayments()`
   - **Service:** `paymentService.getAllPayments()`
   - **Purpose:** Retrieves all payments related to current user's entries
   - **Return Type:** `ResponseEntity<List<PaymentDTO>>`
   - **Features:** Includes entry reference information in DTO

2. **`GET /api/payments/{id}`** - ⭐ **LOW USAGE**
   - **Method:** `getPaymentById(@PathVariable UUID id)`
   - **Service:** `paymentService.getPaymentById(id)`
   - **Purpose:** Retrieves a specific payment by ID
   - **Return Type:** `ResponseEntity<PaymentDTO>`

3. **`GET /api/payments/entry/{entryId}`** - ⭐⭐⭐ **HIGHEST USAGE**
   - **Method:** `getPaymentsByEntry(@PathVariable UUID entryId)`
   - **Service:** `paymentService.getPaymentsByEntry(entryId)`
   - **Purpose:** Retrieves all payments for a specific entry
   - **Return Type:** `ResponseEntity<List<PaymentDTO>>`
   - **Usage:** Called when viewing entry details to show payment history

4. **`POST /api/payments`** - ⭐⭐ **HIGH USAGE**
   - **Method:** `createPayment(@RequestPart CreatePaymentRequest, @RequestPart MultipartFile)` (with proof)
   - **Overload:** `createPayment(@RequestBody CreatePaymentRequest)` (JSON-only)
   - **Service:** `paymentService.createPayment(request, proof)`
   - **Purpose:** Records a new payment for an entry
   - **Return Type:** `ResponseEntity<PaymentDTO>` (HTTP 201 Created)
   - **Content Types:**
     - `multipart/form-data` (with proof file)
     - `application/json` (without proof)
   - **Features:**
     - Calculates change amount if overpayment
     - Updates entry status and remaining balance
     - Links payment to payment allocation if provided (group expenses)
     - Stores proof as attachment

5. **`PUT /api/payments/{id}`** - ⭐ **LOW USAGE**
   - **Method:** `updatePayment(@PathVariable UUID id, @RequestBody CreatePaymentRequest)`
   - **Service:** `paymentService.updatePayment(id, request)`
   - **Purpose:** Updates payment information and recalculates entry balance
   - **Return Type:** `ResponseEntity<PaymentDTO>`

6. **`DELETE /api/payments/{id}`** - ⭐ **LOW USAGE**
   - **Method:** `deletePayment(@PathVariable UUID id)`
   - **Service:** `paymentService.deletePayment(id)`
   - **Purpose:** Deletes a payment
   - **Return Type:** `ResponseEntity<Void>` (HTTP 204 No Content)

---

### Payment Allocation Controller (`PaymentAllocationController`)

**Base Path:** `/api/payment-allocations`

#### **Most Frequently Used Endpoints:**

1. **`GET /api/payment-allocations`** - ⭐ **LOW USAGE**
   - **Method:** `getAllPaymentAllocations()`
   - **Service:** `paymentAllocationService.getAllPaymentAllocations()`
   - **Purpose:** Retrieves all payment allocations for current user's entries
   - **Return Type:** `ResponseEntity<List<PaymentAllocationDTO>>`

2. **`GET /api/payment-allocations/entry/{entryId}`** - ⭐⭐⭐ **HIGHEST USAGE**
   - **Method:** `getPaymentAllocationsByEntry(@PathVariable UUID entryId)`
   - **Service:** `paymentAllocationService.getPaymentAllocationsByEntry(entryId)`
   - **Purpose:** Retrieves all payment allocations for a specific group expense entry
   - **Return Type:** `ResponseEntity<List<PaymentAllocationDTO>>`
   - **Usage:** Called when viewing group expense entry details
   - **Features:** Computes payment status and percentage of total for each allocation

3. **`GET /api/payment-allocations/{id}`** - ⭐ **LOW USAGE**
   - **Method:** `getPaymentAllocationById(@PathVariable UUID id)`
   - **Service:** `paymentAllocationService.getPaymentAllocationById(id)`
   - **Purpose:** Retrieves a specific payment allocation
   - **Return Type:** `ResponseEntity<PaymentAllocationDTO>`

4. **`POST /api/payment-allocations`** - ⭐⭐ **HIGH USAGE**
   - **Method:** `createPaymentAllocations(@RequestBody CreatePaymentAllocationRequest)`
   - **Service:** `paymentAllocationService.createPaymentAllocations(request)`
   - **Purpose:** Creates payment allocations for a group expense entry
   - **Return Type:** `ResponseEntity<List<PaymentAllocationDTO>>` (HTTP 201 Created)
   - **Usage:** Called during entry creation or when setting up allocations
   - **Features:** Validates that allocations sum to entry amount

5. **`PUT /api/payment-allocations/{id}`** - ⭐ **MEDIUM USAGE**
   - **Method:** `updatePaymentAllocation(@PathVariable UUID id, @RequestBody PaymentAllocationDTO)`
   - **Service:** `paymentAllocationService.updatePaymentAllocation(id, dto)`
   - **Purpose:** Updates an individual payment allocation
   - **Return Type:** `ResponseEntity<PaymentAllocationDTO>`

6. **`DELETE /api/payment-allocations/{id}`** - ⭐⭐ **HIGH USAGE**
   - **Method:** `deletePaymentAllocation(@PathVariable UUID id)`
   - **Service:** `paymentAllocationService.deletePaymentAllocation(id)`
   - **Purpose:** Deletes a payment allocation
   - **Return Type:** `ResponseEntity<Void>` (HTTP 204 No Content)
   - **Cascade Handling:** Deletes linked payment_allocation_payment records first

---

### Installment Controller (`InstallmentController`)

**Base Path:** `/api/installments`

#### **Most Frequently Used Endpoints:**

1. **`POST /api/installments/terms/{termId}/skip`** - ⭐ **MEDIUM USAGE**
   - **Method:** `skipTerm(@PathVariable UUID termId)`
   - **Service:** `installmentService.skipTerm(termId)`
   - **Purpose:** Marks an installment term as skipped and applies penalty
   - **Return Type:** `ResponseEntity<InstallmentTermDTO>`
   - **Features:**
     - Calculates late fee penalty (5% of term amount, minimum ₱50)
     - Adds penalty to entry's remaining balance
     - Updates term status to SKIPPED

2. **`GET /api/installments/terms/{termId}/skip-penalty`** - ⭐ **MEDIUM USAGE**
   - **Method:** `getSkipPenalty(@PathVariable UUID termId)`
   - **Service:** `installmentService.calculateSkipPenalty(termId)`
   - **Purpose:** Calculates the penalty that would be applied if term is skipped (preview)
   - **Return Type:** `ResponseEntity<Map<String, BigDecimal>>` with "penalty" key
   - **Usage:** Called before skipping to show user the penalty amount

3. **`PUT /api/installments/terms/{termId}/status?status={status}`** - ⭐ **MEDIUM USAGE**
   - **Method:** `updateTermStatus(@PathVariable UUID termId, @RequestParam InstallmentStatus status)`
   - **Service:** `installmentService.updateTermStatus(termId, status)`
   - **Purpose:** Updates installment term status (e.g., to PAID)
   - **Return Type:** `ResponseEntity<InstallmentTermDTO>`
   - **Usage:** Called when recording payment for a specific term

4. **`POST /api/installments/update-delinquent`** - ⭐ **LOW USAGE**
   - **Method:** `updateDelinquentTerms()`
   - **Service:** `installmentService.updateDelinquentTerms()`
   - **Purpose:** Marks overdue unpaid terms as DELINQUENT
   - **Return Type:** `ResponseEntity<Void>`
   - **Features:** Scans all terms and updates status based on due date

---

## Service Layer Functions

Services contain the core business logic and are located in `com.loantracking.service` package. All services use `@Transactional` for database transaction management.

### Entry Service (`EntryService`)

#### **Most Frequently Used Methods:**

1. **`getAllEntries()`** - ⭐⭐⭐ **HIGHEST USAGE**
   - **Repository:** `entryRepository.findAll()`
   - **Purpose:** Retrieves all entries filtered by current user
   - **Logic:** Filters entries where user is lender, borrower, or group member
   - **Returns:** `List<EntryDTO>`

2. **`getEntryById(UUID id)`** - ⭐⭐⭐ **HIGHEST USAGE**
   - **Repository:** `entryRepository.findById(id)`
   - **Purpose:** Retrieves entry with all related data
   - **Logic:** Loads installment plan, terms, and payments
   - **Returns:** `EntryDTO`

3. **`createEntry(CreateEntryRequest request, MultipartFile proof)`** - ⭐⭐ **HIGH USAGE**
   - **Repositories Used:** Multiple
   - **Purpose:** Creates entry with validation and related entities
   - **Key Logic:**
     - Validates borrower/lender constraints
     - Generates unique reference ID
     - Creates installment plan and terms if INSTALLMENT_EXPENSE
     - Stores proof as attachment
     - Flushes and reloads to ensure data consistency
   - **Returns:** `EntryDTO`

4. **`updateEntry(UUID id, CreateEntryRequest request)`** - ⭐ **MEDIUM USAGE**
   - **Repository:** `entryRepository.findById(id)`, `entryRepository.save(entry)`
   - **Purpose:** Updates entry metadata (name, description, dates, notes)
   - **Returns:** `EntryDTO`

5. **`deleteEntry(UUID id)`** - ⭐ **MEDIUM USAGE**
   - **Repositories:** Multiple for cascade deletion
   - **Purpose:** Deletes entry and related data
   - **Cascade Logic:** Deletes payment allocations and linked payments first

6. **`completeEntry(UUID id)`** - ⭐ **MEDIUM USAGE**
   - **Repositories:** `entryRepository`, `installmentTermRepository`
   - **Purpose:** Marks entry as PAID and completes unpaid installment terms
   - **Returns:** `EntryDTO`

7. **`autoCompleteEntries()`** - ⭐ **LOW USAGE**
   - **Repositories:** `entryRepository.findAll()`, `paymentEntryRepository`
   - **Purpose:** Auto-completes entries based on payment totals
   - **Logic:** Recalculates `amountRemaining` from payments for accuracy
   - **Returns:** `int` (count of completed entries)

#### **Helper Methods (Internal):**

- `getOrCreateCurrentUser()` - Gets or creates user from UserContext
- `isEntryRelatedToParentUser(Entry, UUID)` - Checks if user has access to entry
- `convertToDTO(Entry)` - Converts Entry entity to DTO with related data
- `createInstallmentPlan(Entry, CreateEntryRequest)` - Creates installment plan
- `generateInstallmentTerms(InstallmentPlan, String)` - Generates installment terms
- `calculateFirstDueDate()`, `calculateNextWeeklyDate()`, `calculateNextMonthlyDate()` - Date calculations for installment terms

---

### Person Service (`PersonService`)

#### **Most Frequently Used Methods:**

1. **`getAllPersons()`** - ⭐⭐⭐ **HIGHEST USAGE**
   - **Repository:** `personRepository.findAll()`
   - **Purpose:** Retrieves all persons
   - **Returns:** `List<PersonDTO>`

2. **`getPersonById(UUID id)`** - ⭐⭐ **HIGH USAGE**
   - **Repository:** `personRepository.findById(id)`
   - **Purpose:** Retrieves specific person
   - **Returns:** `PersonDTO`

3. **`searchPersons(String name)`** - ⭐⭐ **HIGH USAGE**
   - **Repository:** `personRepository.findByFullNameContainingIgnoreCase(name)`
   - **Purpose:** Searches persons by name (case-insensitive)
   - **Returns:** `List<PersonDTO>`

4. **`createPerson(PersonDTO personDTO)`** - ⭐⭐ **HIGH USAGE**
   - **Repository:** `personRepository.save(person)`
   - **Purpose:** Creates new person
   - **Returns:** `PersonDTO`

5. **`updatePerson(UUID id, PersonDTO personDTO)`** - ⭐ **MEDIUM USAGE**
   - **Repository:** `personRepository.findById(id)`, `personRepository.save(person)`
   - **Purpose:** Updates person name
   - **Returns:** `PersonDTO`

6. **`deletePerson(UUID id)`** - ⭐ **MEDIUM USAGE**
   - **Repository:** `personRepository.existsById(id)`, `personRepository.deleteById(id)`
   - **Purpose:** Deletes person

---

### Group Service (`GroupService`)

#### **Most Frequently Used Methods:**

1. **`getAllGroups()`** - ⭐⭐⭐ **HIGHEST USAGE**
   - **Repositories:** `groupRepository.findAll()`, `groupMemberRepository`
   - **Purpose:** Retrieves all groups with their members
   - **Returns:** `List<GroupDTO>`

2. **`getGroupById(UUID id)`** - ⭐⭐ **HIGH USAGE**
   - **Repositories:** `groupRepository.findById(id)`, `groupMemberRepository`
   - **Purpose:** Retrieves group with members
   - **Returns:** `GroupDTO`

3. **`createGroup(GroupDTO groupDTO)`** - ⭐⭐ **HIGH USAGE**
   - **Repositories:** `groupRepository.save(group)`, `groupMemberRepository.save(member)`
   - **Purpose:** Creates group with optional initial members
   - **Validation:** Checks group name uniqueness
   - **Returns:** `GroupDTO`

4. **`updateGroup(UUID id, GroupDTO groupDTO)`** - ⭐ **MEDIUM USAGE**
   - **Repositories:** `groupRepository.findById(id)`, `groupRepository.save(group)`
   - **Purpose:** Updates group name
   - **Returns:** `GroupDTO`

5. **`addMemberToGroup(UUID groupId, UUID personId)`** - ⭐⭐ **HIGH USAGE**
   - **Repositories:** `groupRepository.findById(groupId)`, `personRepository.findById(personId)`, `groupMemberRepository.save(member)`
   - **Purpose:** Adds person to group
   - **Validation:** Prevents duplicate members

6. **`removeMemberFromGroup(UUID groupId, UUID personId)`** - ⭐ **MEDIUM USAGE**
   - **Repositories:** `groupMemberRepository.findByGroup_GroupId(groupId)`, `groupMemberRepository.delete(member)`
   - **Purpose:** Removes person from group

---

### Payment Service (`PaymentService`)

#### **Most Frequently Used Methods:**

1. **`getAllPayments()`** - ⭐⭐ **HIGH USAGE**
   - **Repositories:** `paymentRepository.findAll()`, `paymentEntryRepository`
   - **Purpose:** Retrieves all payments for current user's entries
   - **Logic:** Filters by entry relationship to current user
   - **Returns:** `List<PaymentDTO>`

2. **`getPaymentsByEntry(UUID entryId)`** - ⭐⭐⭐ **HIGHEST USAGE**
   - **Repositories:** `paymentEntryRepository.findByEntry_EntryId(entryId)`
   - **Purpose:** Retrieves all payments for a specific entry
   - **Returns:** `List<PaymentDTO>`

3. **`createPayment(CreatePaymentRequest request, MultipartFile proof)`** - ⭐⭐ **HIGH USAGE**
   - **Repositories Used:** Multiple
   - **Purpose:** Records payment and updates entry
   - **Key Logic:**
     - Calculates change amount if overpayment
     - Links payment to entry via PaymentEntry
     - Links to payment allocation if provided (group expenses)
     - Updates entry status and remaining balance
     - Stores proof as attachment
   - **Returns:** `PaymentDTO`

4. **`updateEntryAfterPayment(Entry entry, BigDecimal paymentAmount)`** - ⭐⭐ **HIGH USAGE** (Internal)
   - **Repository:** `entryRepository.save(entry)`
   - **Purpose:** Updates entry remaining balance and status after payment
   - **Logic:** Sets status to PAID if fully paid, PARTIALLY_PAID if partially paid

---

### Payment Allocation Service (`PaymentAllocationService`)

#### **Most Frequently Used Methods:**

1. **`getPaymentAllocationsByEntry(UUID entryId)`** - ⭐⭐⭐ **HIGHEST USAGE**
   - **Repositories:** `paymentAllocationRepository.findByEntry_EntryId(entryId)`, `paymentAllocationPaymentRepository`
   - **Purpose:** Retrieves allocations for group expense entry
   - **Logic:** Computes payment status and percentage for each allocation
   - **Returns:** `List<PaymentAllocationDTO>`

2. **`createPaymentAllocations(CreatePaymentAllocationRequest request)`** - ⭐⭐ **HIGH USAGE**
   - **Repositories:** `entryRepository.findById()`, `personRepository.findById()`, `paymentAllocationRepository.save()`
   - **Purpose:** Creates multiple payment allocations
   - **Returns:** `List<PaymentAllocationDTO>`

3. **`updatePaymentAllocation(UUID id, PaymentAllocationDTO dto)`** - ⭐ **MEDIUM USAGE**
   - **Repository:** `paymentAllocationRepository`
   - **Purpose:** Updates allocation amount, description, notes
   - **Returns:** `PaymentAllocationDTO`

4. **`deletePaymentAllocation(UUID id)`** - ⭐⭐ **HIGH USAGE**
   - **Repositories:** `paymentAllocationPaymentRepository`, `paymentAllocationRepository`
   - **Purpose:** Deletes allocation and linked payment records
   - **Cascade:** Deletes payment_allocation_payment records first

5. **`computeStatus(PaymentAllocation, Entry)`** - ⭐⭐ **HIGH USAGE** (Internal)
   - **Purpose:** Calculates allocation status (UNPAID, PARTIALLY_PAID, PAID)
   - **Logic:** Uses linked payments or falls back to person payments for entry

---

### Installment Service (`InstallmentService`)

#### **Most Frequently Used Methods:**

1. **`skipTerm(UUID termId)`** - ⭐ **MEDIUM USAGE**
   - **Repositories:** `installmentTermRepository.findById()`, `entryRepository.save()`
   - **Purpose:** Marks term as skipped and applies penalty
   - **Logic:**
     - Calculates penalty (5% of term amount, minimum ₱50)
     - Updates term status to SKIPPED
     - Adds penalty to entry's remaining balance
   - **Returns:** `InstallmentTermDTO`

2. **`calculateSkipPenalty(UUID termId)`** - ⭐ **MEDIUM USAGE**
   - **Repository:** `installmentTermRepository.findById(termId)`
   - **Purpose:** Calculates penalty without applying it (preview)
   - **Returns:** `BigDecimal` (penalty amount)

3. **`updateTermStatus(UUID termId, InstallmentStatus status)`** - ⭐ **MEDIUM USAGE**
   - **Repository:** `installmentTermRepository.findById()`, `installmentTermRepository.save()`
   - **Purpose:** Updates term status (e.g., to PAID)
   - **Returns:** `InstallmentTermDTO`

4. **`updateDelinquentTerms()`** - ⭐ **LOW USAGE**
   - **Repository:** `installmentTermRepository.findAll()`, `installmentTermRepository.save()`
   - **Purpose:** Marks overdue unpaid terms as DELINQUENT
   - **Logic:** Filters terms by due date and current status

---

## Repository Layer Methods

Repositories extend `JpaRepository` and are located in `com.loantracking.repository` package. They provide data access using Spring Data JPA.

### Entry Repository (`EntryRepository`)

#### **Most Frequently Used Methods:**

1. **`findAll()`** - ⭐⭐⭐ **HIGHEST USAGE**
   - **Returns:** `List<Entry>`
   - **Usage:** Loads all entries for filtering

2. **`findById(UUID id)`** - ⭐⭐⭐ **HIGHEST USAGE**
   - **Returns:** `Optional<Entry>`
   - **Usage:** Retrieves specific entry

3. **`save(Entry entry)`** - ⭐⭐ **HIGH USAGE**
   - **Returns:** `Entry`
   - **Usage:** Persists new or updated entries

4. **`existsByReferenceId(String referenceId)`** - ⭐ **MEDIUM USAGE**
   - **Returns:** `boolean`
   - **Usage:** Ensures reference ID uniqueness

5. **`deleteById(UUID id)`** - ⭐ **MEDIUM USAGE**
   - **Usage:** Deletes entry

6. **`flush()`** - ⭐ **MEDIUM USAGE**
   - **Usage:** Forces database flush for consistency

#### **Custom Query Methods:**
- `findByReferenceId(String referenceId)` - Finds entry by reference ID
- `findByBorrowerPerson_PersonId(UUID personId)` - Finds entries by borrower
- `findByBorrowerGroup_GroupId(UUID groupId)` - Finds entries by group borrower
- `findByLenderPerson_PersonId(UUID personId)` - Finds entries by lender
- `findByTransactionType(TransactionType type)` - Filters by transaction type
- `findByStatus(PaymentStatus status)` - Filters by payment status

---

### Person Repository (`PersonRepository`)

#### **Most Frequently Used Methods:**

1. **`findAll()`** - ⭐⭐⭐ **HIGHEST USAGE**
   - **Returns:** `List<Person>`
   - **Usage:** Loads all persons

2. **`findById(UUID id)`** - ⭐⭐ **HIGH USAGE**
   - **Returns:** `Optional<Person>`
   - **Usage:** Retrieves specific person

3. **`save(Person person)`** - ⭐⭐ **HIGH USAGE**
   - **Returns:** `Person`
   - **Usage:** Persists new or updated persons

4. **`findByFullName(String fullName)`** - ⭐⭐ **HIGH USAGE**
   - **Returns:** `Optional<Person>`
   - **Usage:** Finds person by exact name (for UserContext)

5. **`findByFullNameContainingIgnoreCase(String name)`** - ⭐⭐ **HIGH USAGE**
   - **Returns:** `List<Person>`
   - **Usage:** Person search functionality

6. **`existsById(UUID id)`** - ⭐ **MEDIUM USAGE**
   - **Returns:** `boolean`
   - **Usage:** Validates person exists before deletion

7. **`deleteById(UUID id)`** - ⭐ **MEDIUM USAGE**
   - **Usage:** Deletes person

---

### Group Repository (`GroupRepository`)

#### **Most Frequently Used Methods:**

1. **`findAll()`** - ⭐⭐⭐ **HIGHEST USAGE**
   - **Returns:** `List<Group>`
   - **Usage:** Loads all groups

2. **`findById(UUID id)`** - ⭐⭐ **HIGH USAGE**
   - **Returns:** `Optional<Group>`
   - **Usage:** Retrieves specific group

3. **`save(Group group)`** - ⭐⭐ **HIGH USAGE**
   - **Returns:** `Group`
   - **Usage:** Persists new or updated groups

4. **`existsByGroupName(String groupName)`** - ⭐ **MEDIUM USAGE**
   - **Returns:** `boolean`
   - **Usage:** Validates group name uniqueness

5. **`deleteById(UUID id)`** - ⭐ **MEDIUM USAGE**
   - **Usage:** Deletes group

#### **Custom Query Methods:**
- `findByGroupName(String groupName)` - Finds group by name

---

### Group Member Repository (`GroupMemberRepository`)

#### **Most Frequently Used Methods:**

1. **`findByGroup_GroupId(UUID groupId)`** - ⭐⭐⭐ **HIGHEST USAGE**
   - **Returns:** `List<GroupMember>`
   - **Usage:** Gets all members of a group

2. **`existsByGroup_GroupIdAndPerson_PersonId(UUID groupId, UUID personId)`** - ⭐⭐ **HIGH USAGE**
   - **Returns:** `boolean`
   - **Usage:** Validates member exists (for entry creation, member addition)

3. **`save(GroupMember member)`** - ⭐⭐ **HIGH USAGE**
   - **Returns:** `GroupMember`
   - **Usage:** Adds member to group

4. **`delete(GroupMember member)`** - ⭐ **MEDIUM USAGE**
   - **Usage:** Removes member from group

5. **`deleteAll(Iterable<GroupMember>)`** - ⭐ **LOW USAGE**
   - **Usage:** Bulk deletion (cascade operations)

---

### Payment Repository (`PaymentRepository`)

#### **Most Frequently Used Methods:**

1. **`findAll()`** - ⭐⭐ **HIGH USAGE**
   - **Returns:** `List<Payment>`
   - **Usage:** Loads all payments for filtering

2. **`findById(UUID id)`** - ⭐⭐ **HIGH USAGE**
   - **Returns:** `Optional<Payment>`
   - **Usage:** Retrieves specific payment

3. **`save(Payment payment)`** - ⭐⭐ **HIGH USAGE**
   - **Returns:** `Payment`
   - **Usage:** Persists new or updated payments

4. **`deleteById(UUID id)`** - ⭐ **LOW USAGE**
   - **Usage:** Deletes payment

#### **Custom Query Methods:**
- `findByPayeePerson_PersonId(UUID personId)` - Finds payments by payee
- `findByPaymentDateBetween(LocalDate start, LocalDate end)` - Finds payments by date range

---

### Payment Entry Repository (`PaymentEntryRepository`)

#### **Most Frequently Used Methods:**

1. **`findByEntry_EntryId(UUID entryId)`** - ⭐⭐⭐ **HIGHEST USAGE**
   - **Returns:** `List<PaymentEntry>`
   - **Usage:** Gets all payments for an entry

2. **`findByPayment_PaymentId(UUID paymentId)`** - ⭐⭐ **HIGH USAGE**
   - **Returns:** `List<PaymentEntry>`
   - **Usage:** Gets entry(s) linked to a payment

3. **`save(PaymentEntry paymentEntry)`** - ⭐⭐ **HIGH USAGE**
   - **Returns:** `PaymentEntry`
   - **Usage:** Links payment to entry

#### **Custom Query Methods:**
- `existsByPayment_PaymentIdAndEntry_EntryId(UUID paymentId, UUID entryId)` - Checks if link exists

---

### Payment Allocation Repository (`PaymentAllocationRepository`)

#### **Most Frequently Used Methods:**

1. **`findByEntry_EntryId(UUID entryId)`** - ⭐⭐⭐ **HIGHEST USAGE**
   - **Returns:** `List<PaymentAllocation>`
   - **Usage:** Gets all allocations for a group expense entry

2. **`findAll()`** - ⭐ **LOW USAGE**
   - **Returns:** `List<PaymentAllocation>`
   - **Usage:** Loads all allocations for filtering

3. **`findById(UUID id)`** - ⭐⭐ **HIGH USAGE**
   - **Returns:** `Optional<PaymentAllocation>`
   - **Usage:** Retrieves specific allocation

4. **`save(PaymentAllocation allocation)`** - ⭐⭐ **HIGH USAGE**
   - **Returns:** `PaymentAllocation>`
   - **Usage:** Persists new or updated allocations

5. **`deleteById(UUID id)`** - ⭐⭐ **HIGH USAGE**
   - **Usage:** Deletes allocation

#### **Custom Query Methods:**
- `findByPerson_PersonId(UUID personId)` - Finds allocations by person

---

### Payment Allocation Payment Repository (`PaymentAllocationPaymentRepository`)

#### **Most Frequently Used Methods:**

1. **`findByAllocation_AllocationId(UUID allocationId)`** - ⭐⭐ **HIGH USAGE**
   - **Returns:** `List<PaymentAllocationPayment>`
   - **Usage:** Gets all payments linked to an allocation (for status calculation)

2. **`save(PaymentAllocationPayment link)`** - ⭐⭐ **HIGH USAGE**
   - **Returns:** `PaymentAllocationPayment`
   - **Usage:** Links payment to allocation

3. **`deleteAll(Iterable<PaymentAllocationPayment>)`** - ⭐ **MEDIUM USAGE**
   - **Usage:** Removes payment-allocation links (cascade deletion)

---

### Installment Plan Repository (`InstallmentPlanRepository`)

#### **Most Frequently Used Methods:**

1. **`findByEntry_EntryId(UUID entryId)`** - ⭐⭐ **HIGH USAGE**
   - **Returns:** `Optional<InstallmentPlan>`
   - **Usage:** Gets installment plan for an entry

2. **`save(InstallmentPlan plan)`** - ⭐ **MEDIUM USAGE**
   - **Returns:** `InstallmentPlan`
   - **Usage:** Persists installment plan during entry creation

---

### Installment Term Repository (`InstallmentTermRepository`)

#### **Most Frequently Used Methods:**

1. **`findByInstallmentPlan_InstallmentId(UUID installmentId)`** - ⭐⭐ **HIGH USAGE**
   - **Returns:** `List<InstallmentTerm>`
   - **Usage:** Gets all terms for an installment plan

2. **`findById(UUID id)`** - ⭐ **MEDIUM USAGE**
   - **Returns:** `Optional<InstallmentTerm>`
   - **Usage:** Retrieves specific term

3. **`save(InstallmentTerm term)`** - ⭐⭐ **HIGH USAGE**
   - **Returns:** `InstallmentTerm`
   - **Usage:** Persists new or updated terms

4. **`findAll()`** - ⭐ **LOW USAGE**
   - **Returns:** `List<InstallmentTerm>`
   - **Usage:** Loads all terms for delinquent update

#### **Custom Query Methods:**
- `findByTermStatus(InstallmentStatus status)` - Filters by status
- `findByInstallmentPlan_InstallmentIdAndTermStatus(UUID installmentId, InstallmentStatus status)` - Filters by plan and status

---

## Utility Functions

### Reference ID Generator (`ReferenceIdGenerator`)

**Location:** `com.loantracking.util.ReferenceIdGenerator`

#### **Most Frequently Used Methods:**

1. **`generateReferenceId(Person borrower, Person lender)`** - ⭐⭐ **HIGH USAGE**
   - **Purpose:** Generates reference ID for person-to-person entries
   - **Logic:** Extracts initials from borrower and lender names
   - **Returns:** `String` (e.g., "JSMD" for John Smith / Mary Doe)
   - **Usage:** Called during entry creation

2. **`generateReferenceId(Group borrowerGroup, Person lender)`** - ⭐ **MEDIUM USAGE**
   - **Purpose:** Generates reference ID for group expense entries
   - **Logic:** Uses first 3-5 characters of group name + lender initials
   - **Returns:** `String`
   - **Usage:** Called during group expense entry creation

3. **`extractInitials(String fullName)`** - ⭐⭐ **HIGH USAGE** (Private)
   - **Purpose:** Extracts initials from full name
   - **Logic:** Handles formats like "Surname, First Name, Initial" or "First Name Last Name"
   - **Returns:** `String` (uppercase initials)

---

### User Context (`UserContext`)

**Location:** `com.loantracking.util.UserContext`

#### **Most Frequently Used Methods:**

1. **`getCurrentUserName()`** - ⭐⭐⭐ **HIGHEST USAGE**
   - **Purpose:** Gets current user name from request header
   - **Header:** `X-Selected-User-Name`
   - **Fallback:** Returns default parent user name from `UserConfig`
   - **Returns:** `String`
   - **Usage:** Called in all services for user-based filtering

2. **`getCurrentUserId()`** - ⭐ **MEDIUM USAGE**
   - **Purpose:** Gets current user ID from request header
   - **Header:** `X-Selected-User-Id`
   - **Returns:** `String` or `null`
   - **Usage:** Available but currently not heavily used

3. **`getCurrentRequest()`** - ⭐⭐⭐ **HIGHEST USAGE** (Private)
   - **Purpose:** Gets current HTTP request from Spring context
   - **Returns:** `HttpServletRequest`
   - **Usage:** Internal method used by other UserContext methods

---

## Configuration and Context Management

### User Configuration (`UserConfig`)

**Location:** `com.loantracking.config.UserConfig`

- **Constant:** `PARENT_USER_NAME` - Default user name (e.g., "tung tung tung sahur")
- **Usage:** Fallback when no user header is provided

### CORS Configuration (`CorsConfig`)

**Location:** `com.loantracking.config.CorsConfig`

- **Purpose:** Configures Cross-Origin Resource Sharing
- **Allowed Origin:** `http://localhost:5173` (Vite dev server)

### Global Exception Handler (`GlobalExceptionHandler`)

**Location:** `com.loantracking.config.GlobalExceptionHandler`

- **Purpose:** Centralized exception handling for REST endpoints
- **Features:** Converts exceptions to appropriate HTTP responses

---

## Data Access Patterns

### Common Patterns Observed:

1. **User-Based Filtering:**
   - All services use `UserContext.getCurrentUserName()` to get current user
   - Entries, payments, and allocations are filtered by user relationship
   - Users can be lenders, borrowers, or group members

2. **Repository Method Naming:**
   - Uses Spring Data JPA naming conventions
   - `findBy{Property}` - Single result
   - `findBy{Property}{Condition}` - With conditions (e.g., `ContainingIgnoreCase`)
   - `existsBy{Property}` - Boolean checks
   - Custom queries using `@Query` annotation

3. **DTO Conversion:**
   - All services have `convertToDTO()` methods
   - DTOs include related entity information (names, IDs)
   - Complex DTOs (EntryDTO) include nested data (installment plans, payments)

4. **Transaction Management:**
   - All services use `@Transactional` annotation
   - Ensures data consistency across multiple repository operations
   - Automatic rollback on exceptions

5. **Cascade Deletion:**
   - Manual cascade handling in services (deletes related records first)
   - Prevents foreign key constraint violations
   - Order matters: Delete child entities before parent

6. **Validation:**
   - Business logic validation in services (not just JPA constraints)
   - Validates relationships (borrower/lender constraints, group membership)
   - Throws `IllegalArgumentException` for validation failures

7. **File Handling:**
   - Multipart file support for proof documents
   - Files stored both in entity fields (for backward compatibility) and Attachment table
   - Attachment table provides better metadata (filename, content type, size)

8. **Reference ID Generation:**
   - Auto-generated unique reference IDs
   - Collision detection and increment logic
   - Used for human-readable entry identification

---

## Summary Statistics

### Most Frequently Used Service Methods (by call count):

1. **`EntryService.getAllEntries()`** - 4+ controller endpoints
2. **`EntryService.getEntryById()`** - 3+ controller endpoints
3. **`EntryService.createEntry()`** - Entry creation with complex logic
4. **`PersonService.getAllPersons()`** - 3+ controller endpoints
5. **`PersonService.searchPersons()`** - Search functionality
6. **`GroupService.getAllGroups()`** - 2+ controller endpoints
7. **`PaymentService.getPaymentsByEntry()`** - Critical for entry details
8. **`PaymentAllocationService.getPaymentAllocationsByEntry()`** - Critical for group expenses

### Most Frequently Used Repository Methods:

1. **`findAll()`** - Used across all repositories
2. **`findById(UUID id)`** - Standard retrieval
3. **`save(Entity entity)`** - Standard persistence
4. **`findByEntry_EntryId(UUID entryId)`** - Relationship queries
5. **`findByGroup_GroupId(UUID groupId)`** - Group member queries
6. **`existsBy{Property}`** - Validation queries

### Utility Function Usage:

1. **`UserContext.getCurrentUserName()`** - Called in all services
2. **`ReferenceIdGenerator.generateReferenceId()`** - Called during entry creation
3. **DTO conversion methods** - Called in all service methods that return data

---

## Architectural Patterns

1. **Layered Architecture:**
   - **Controller Layer** - REST endpoints, request/response handling
   - **Service Layer** - Business logic, validation, orchestration
   - **Repository Layer** - Data access, JPA queries
   - **Model Layer** - Entity classes, domain models
   - **DTO Layer** - Data transfer objects for API

2. **Dependency Injection:**
   - Uses Spring `@Autowired` for dependency injection
   - Services inject repositories
   - Controllers inject services

3. **Transaction Boundaries:**
   - Service methods marked with `@Transactional`
   - Automatic rollback on exceptions
   - Flush operations for consistency

4. **Error Handling:**
   - Service methods throw `IllegalArgumentException` for validation errors
   - Global exception handler converts to HTTP responses
   - Meaningful error messages for frontend consumption

5. **Data Consistency:**
   - Transaction management ensures atomic operations
   - Flush operations ensure database consistency before returning
   - Reload operations verify data persistence

---

This overview demonstrates that the backend follows a standard Spring Boot architecture with clear separation of concerns. The most frequently used functions are those related to CRUD operations, user-based filtering, and relationship management between entities. The system is designed to handle complex business logic around entries, payments, allocations, and installment plans while maintaining data integrity and user context.

