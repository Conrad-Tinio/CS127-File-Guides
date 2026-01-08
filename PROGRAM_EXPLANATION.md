# Loan Tracking System - Program Explanation

## Table of Contents
1. [Program Overview](#program-overview)
2. [Architecture & How It Runs](#architecture--how-it-runs)
3. [Individual Functions](#individual-functions)
4. [CRUD Operations](#crud-operations)
   - [Contacts (Persons)](#1-contacts-persons)
   - [Groups](#2-groups)
   - [General Entry (Loaner Perspective)](#3-general-entry-loaner-perspective)
   - [General Entry (Borrower Perspective)](#4-general-entry-borrower-perspective)
   - [Installment (Loaner Perspective)](#5-installment-loaner-perspective)
   - [Installment (Borrower Perspective)](#6-installment-borrower-perspective)
   - [Group Expense](#7-group-expense)

---

## Program Overview

This is a **Loan Tracking System** built with:
- **Backend**: Spring Boot (Java) with JPA/Hibernate
- **Frontend**: React + TypeScript with Vite
- **Database**: PostgreSQL (Supabase)
- **Purpose**: Track loans, expenses, installments, and group expenses between people

---

## Architecture & How It Runs

### Startup Flow

1. **Backend Startup** (`LoanTrackingApplication.java`):
   - Spring Boot application starts on port 8080
   - Connects to PostgreSQL database (Supabase)
   - Initializes JPA repositories
   - Sets up CORS for frontend communication
   - Auto-creates database tables via Hibernate (`ddl-auto=update`)

2. **Frontend Startup** (`main.tsx` → `App.tsx`):
   - React app starts on port 5173 (Vite dev server)
   - Initializes UserContext to manage current user
   - Sets up React Router with routes:
     - `/` - HomePage
     - `/entries` - AllPaymentsPage (list of entries)
     - `/entries/new` - CreateEntryPage
     - `/entries/:id` - EntryDetailPage
     - `/people-groups` - PeopleGroupsPage

3. **User Context System**:
   - Frontend stores selected user in localStorage
   - Sends user name in HTTP header `X-Selected-User-Name` with every request
   - Backend extracts user from header via `UserContext.getCurrentUserName()`
   - Default user: "tung tung tung sahur" (from `UserConfig`)

### Request Flow

```
Frontend (React) 
  → API Service (api.ts) 
    → HTTP Request with User Header 
      → Backend Controller (REST API) 
        → Service Layer (Business Logic) 
          → Repository (JPA) 
            → PostgreSQL Database
```

### Key Components

**Backend:**
- **Controllers**: Handle HTTP requests/responses (`@RestController`)
- **Services**: Business logic and validation (`@Service`)
- **Repositories**: Data access layer (JPA repositories)
- **Models**: Database entities (`@Entity`)
- **DTOs**: Data transfer objects for API communication

**Frontend:**
- **Pages**: Main UI screens
- **Components**: Reusable UI elements
- **Services**: API client (`api.ts`)
- **Contexts**: User context management
- **Types**: TypeScript type definitions

---

## Individual Functions

### Backend Services

#### 1. **PersonService** (`PersonService.java`)
- Manages contact/person CRUD operations
- Converts between Person entity and PersonDTO
- Provides search functionality

#### 2. **GroupService** (`GroupService.java`)
- Manages group CRUD operations
- Handles group member addition/removal
- Validates unique group names

#### 3. **EntryService** (`EntryService.java`)
- **Core service** for all entry types (straight, installment, group expense)
- Creates entries with validation:
  - Ensures borrower is either person OR group (not both)
  - Validates installment entries cannot have group borrowers
  - Validates payment methods (STRAIGHT_EXPENSE only allows CASH)
- Generates unique reference IDs
- Creates installment plans when needed
- Filters entries based on current user (must be lender or borrower)
- Updates entry status based on payments

#### 4. **PaymentService** (`PaymentService.java`)
- Manages payment records
- Links payments to entries via PaymentEntry join table
- Updates entry `amountRemaining` and status automatically
- Filters payments by current user context

#### 5. **PaymentAllocationService** (`PaymentAllocationService.java`)
- Manages payment allocations for group expenses
- Splits group expense amounts among group members
- Computes payment status (UNPAID, PARTIALLY_PAID, PAID) based on actual payments
- Calculates percentage of total expense per member

#### 6. **InstallmentService** (`InstallmentService.java`)
- Manages installment term status updates
- Allows skipping terms
- Updates delinquent terms (past due date and unpaid)
- Validates user access to installment terms

### User Access Control

**Key Function**: `isEntryRelatedToParentUser()` (in EntryService, PaymentService, etc.)
- Ensures users can only see/modify entries where they are:
  - The lender, OR
  - The borrower (person), OR
  - The lender (for group expenses)
- Uses XOR logic: user must be exactly one of lender/borrower, not both

---

## CRUD Operations

### 1. Contacts (Persons)

**Model**: `Person.java`
- Fields: `personId` (UUID), `fullName`, `createdAt`, `updatedAt`

#### CREATE
- **Endpoint**: `POST /api/persons`
- **Controller**: `PersonController.createPerson()`
- **Service**: `PersonService.createPerson()`
- **Flow**:
  1. Frontend sends PersonDTO with `fullName`
  2. Controller receives request
  3. Service creates new Person entity
  4. Repository saves to database
  5. Returns PersonDTO with generated UUID

**Code Path**:
```
Frontend: api.ts → personApi.create()
Backend: PersonController.createPerson()
  → PersonService.createPerson()
    → PersonRepository.save()
```

#### READ (All)
- **Endpoint**: `GET /api/persons`
- **Returns**: List of all persons as PersonDTOs

#### READ (By ID)
- **Endpoint**: `GET /api/persons/{id}`
- **Returns**: Single PersonDTO

#### READ (Search)
- **Endpoint**: `GET /api/persons/search?name={name}`
- **Returns**: List of persons matching name (case-insensitive)

#### UPDATE
- **Endpoint**: `PUT /api/persons/{id}`
- **Flow**: Updates `fullName` of existing person

#### DELETE
- **Endpoint**: `DELETE /api/persons/{id}`
- **Flow**: Deletes person (cascade deletes related entries if configured)

---

### 2. Groups

**Model**: `Group.java`
- Fields: `groupId` (UUID), `groupName` (unique), `createdAt`, `updatedAt`
- Related: `GroupMember` (join table linking groups to persons)

#### CREATE
- **Endpoint**: `POST /api/groups`
- **Controller**: `GroupController.createGroup()`
- **Service**: `GroupService.createGroup()`
- **Flow**:
  1. Validates group name is unique
  2. Creates Group entity
  3. Optionally adds members if provided in request
  4. Creates GroupMember records for each member
  5. Returns GroupDTO with members list

**Code Path**:
```
Frontend: api.ts → groupApi.create()
Backend: GroupController.createGroup()
  → GroupService.createGroup()
    → GroupRepository.save()
    → GroupMemberRepository.save() (for each member)
```

#### READ (All)
- **Endpoint**: `GET /api/groups`
- **Returns**: List of all groups with their members

#### READ (By ID)
- **Endpoint**: `GET /api/groups/{id}`
- **Returns**: Single GroupDTO with members

#### UPDATE
- **Endpoint**: `PUT /api/groups/{id}`
- **Flow**: Updates group name (validates uniqueness)

#### DELETE
- **Endpoint**: `DELETE /api/groups/{id}`
- **Flow**: Deletes group (cascade deletes group members)

#### ADD MEMBER
- **Endpoint**: `POST /api/groups/{groupId}/members/{personId}`
- **Flow**: Creates GroupMember record linking person to group

#### REMOVE MEMBER
- **Endpoint**: `DELETE /api/groups/{groupId}/members/{personId}`
- **Flow**: Deletes GroupMember record

---

### 3. General Entry (Loaner Perspective)

**Transaction Type**: `STRAIGHT_EXPENSE`
**Borrower**: Person
**Role**: User is the lender (loaner)

**Model**: `Entry.java`
- Key fields: `entryId`, `entryName`, `transactionType` (STRAIGHT_EXPENSE), `amountBorrowed`, `amountRemaining`, `status`, `lenderPerson`, `borrowerPerson`, `paymentMethod` (must be CASH)

#### CREATE
- **Endpoint**: `POST /api/entries`
- **Controller**: `EntryController.createEntry()`
- **Service**: `EntryService.createEntry()`
- **Flow**:
  1. Validates:
     - Borrower must be person (not group)
     - Payment method must be CASH (for STRAIGHT_EXPENSE)
     - Lender and borrower must exist
  2. Creates Entry entity
  3. Sets `amountRemaining = amountBorrowed`
  4. Sets `status = UNPAID`
  5. Generates unique `referenceId` (format: based on borrower and lender names)
  6. Saves to database

**Code Path**:
```
Frontend: CreateEntryPage.tsx → entryApi.create()
Backend: EntryController.createEntry()
  → EntryService.createEntry()
    → Validates borrower person exists
    → Validates payment method is CASH
    → ReferenceIdGenerator.generateReferenceId()
    → EntryRepository.save()
```

**Request Body**:
```json
{
  "entryName": "Office Supplies Loan",
  "transactionType": "STRAIGHT_EXPENSE",
  "borrowerPersonId": "<uuid>",
  "lenderPersonId": "<uuid>",
  "amountBorrowed": 5000.00,
  "paymentMethod": "CASH",
  "dateBorrowed": "2024-01-15",
  "notes": "Optional notes"
}
```

#### READ (All - Filtered by User)
- **Endpoint**: `GET /api/entries`
- **Flow**:
  1. Gets current user from UserContext
  2. Filters entries where user is lender OR borrower
  3. Returns list of EntryDTOs

**Code Path**:
```
EntryService.getAllEntries()
  → getOrCreateCurrentUser()
  → entryRepository.findAll()
  → filter by isEntryRelatedToParentUser()
  → convertToDTO()
```

#### READ (By ID)
- **Endpoint**: `GET /api/entries/{id}`
- **Flow**: Validates user has access, then returns EntryDTO

#### UPDATE
- **Endpoint**: `PUT /api/entries/{id}`
- **Flow**:
  1. Validates user access
  2. Updates: `entryName`, `description`, `dateBorrowed`, `notes`, `paymentNotes`
  3. **Note**: Cannot change borrower, lender, or transaction type after creation

#### DELETE
- **Endpoint**: `DELETE /api/entries/{id}`
- **Flow**: Validates user access, then deletes entry (cascade deletes related payments, allocations)

---

### 4. General Entry (Borrower Perspective)

**Transaction Type**: `STRAIGHT_EXPENSE`
**Borrower**: Person
**Role**: User is the borrower

**Same endpoints and flow as Loaner Perspective**, but:
- User appears as `borrowerPerson` in the entry
- Filtering shows entries where user is borrower
- User can view their own debt entries
- User can update entry details (name, notes)
- User can make payments (see Payment CRUD)

**Key Difference**: The entry is the same entity, but viewed from borrower's perspective. The `isEntryRelatedToParentUser()` function allows access if user is borrower.

---

### 5. Installment (Loaner Perspective)

**Transaction Type**: `INSTALLMENT_EXPENSE`
**Borrower**: Person
**Role**: User is the lender (loaner)

**Models**:
- `Entry.java`: Main entry record
- `InstallmentPlan.java`: Plan configuration (start date, frequency, terms, amount per term)
- `InstallmentTerm.java`: Individual payment terms/schedules

#### CREATE
- **Endpoint**: `POST /api/entries`
- **Flow**:
  1. Same validation as General Entry, plus:
     - Validates `installmentStartDate` is provided
     - Validates `paymentFrequency` (WEEKLY or MONTHLY)
     - Validates `paymentTerms` > 0
  2. Creates Entry entity (same as General Entry)
  3. Creates InstallmentPlan:
     - `startDate`: First payment date
     - `paymentFrequency`: WEEKLY or MONTHLY
     - `paymentTerms`: Number of installments
     - `amountPerTerm`: Auto-calculated = `amountBorrowed / paymentTerms`
  4. Generates InstallmentTerm records:
     - One term for each payment period
     - Each term has: `termNumber`, `dueDate`, `termStatus` (NOT_STARTED)
     - Due dates calculated based on frequency

**Code Path**:
```
EntryService.createEntry()
  → Validates installment parameters
  → EntryRepository.save()
  → createInstallmentPlan()
    → InstallmentPlanRepository.save()
    → generateInstallmentTerms()
      → InstallmentTermRepository.save() (for each term)
```

**Request Body**:
```json
{
  "entryName": "Car Loan",
  "transactionType": "INSTALLMENT_EXPENSE",
  "borrowerPersonId": "<uuid>",
  "lenderPersonId": "<uuid>",
  "amountBorrowed": 120000.00,
  "paymentMethod": "CASH",
  "installmentStartDate": "2024-01-15",
  "paymentFrequency": "MONTHLY",
  "paymentTerms": 12
}
```

#### READ (All)
- **Endpoint**: `GET /api/entries`
- **Returns**: EntryDTOs with `installmentPlan` populated (includes terms)

#### READ (By ID)
- **Endpoint**: `GET /api/entries/{id}`
- **Returns**: EntryDTO with full installment plan and all terms

#### UPDATE (Entry Details)
- **Endpoint**: `PUT /api/entries/{id}`
- **Flow**: Updates entry metadata (name, notes). **Cannot modify installment plan after creation**.

#### UPDATE (Term Status)
- **Endpoint**: `PUT /api/installments/terms/{termId}/status?status={status}`
- **Controller**: `InstallmentController.updateTermStatus()`
- **Service**: `InstallmentService.updateTermStatus()`
- **Flow**:
  1. Validates user access to term (via entry)
  2. Updates term status: NOT_STARTED, UNPAID, PAID, SKIPPED, DELINQUENT
  3. Returns updated InstallmentTermDTO

#### SKIP TERM
- **Endpoint**: `POST /api/installments/terms/{termId}/skip`
- **Flow**: Sets term status to SKIPPED

#### UPDATE DELINQUENT TERMS
- **Endpoint**: `POST /api/installments/update-delinquent`
- **Flow**: Automatically marks terms as DELINQUENT if:
  - Due date is before today
  - Status is UNPAID

#### DELETE
- **Endpoint**: `DELETE /api/entries/{id}`
- **Flow**: Deletes entry, cascades to InstallmentPlan and InstallmentTerms

---

### 6. Installment (Borrower Perspective)

**Transaction Type**: `INSTALLMENT_EXPENSE`
**Borrower**: Person
**Role**: User is the borrower

**Same endpoints and flow as Loaner Perspective**, but:
- User appears as `borrowerPerson` in the entry
- User can view their installment schedule
- User can see which terms are due, paid, or delinquent
- User can make payments to fulfill terms (see Payment CRUD)

**Key Feature**: The installment plan shows the borrower when payments are due and tracks progress.

---

### 7. Group Expense

**Transaction Type**: `GROUP_EXPENSE`
**Borrower**: Group
**Role**: User is the lender

**Models**:
- `Entry.java`: Main entry (borrower is Group, not Person)
- `PaymentAllocation.java`: Splits expense among group members

#### CREATE
- **Endpoint**: `POST /api/entries`
- **Flow**:
  1. Validates:
     - Borrower must be group (not person)
     - Group must exist
     - Lender must be person
  2. Creates Entry entity with `borrowerGroup` reference
  3. Generates reference ID based on group and lender
  4. Saves entry

**Request Body**:
```json
{
  "entryName": "Team Dinner Expense",
  "transactionType": "GROUP_EXPENSE",
  "borrowerGroupId": "<uuid>",
  "lenderPersonId": "<uuid>",
  "amountBorrowed": 5000.00,
  "paymentMethod": "CASH"
}
```

#### CREATE PAYMENT ALLOCATIONS
- **Endpoint**: `POST /api/payment-allocations`
- **Controller**: `PaymentAllocationController.createPaymentAllocations()`
- **Service**: `PaymentAllocationService.createPaymentAllocations()`
- **Flow**:
  1. Validates entry exists and is GROUP_EXPENSE
  2. Validates user access (must be lender)
  3. Creates PaymentAllocation records for each group member:
     - Each allocation has: `personId`, `amount`, `description`, `notes`
     - Amounts are split among members (manual allocation by user)
  4. Computes percentage of total for each allocation
  5. Sets initial status to UNPAID

**Code Path**:
```
PaymentAllocationController.createPaymentAllocations()
  → PaymentAllocationService.createPaymentAllocations()
    → For each allocation item:
      → Create PaymentAllocation
      → Set entry, person, amount
      → PaymentAllocationRepository.save()
```

**Request Body**:
```json
{
  "entryId": "<uuid>",
  "allocations": [
    {
      "personId": "<uuid>",
      "amount": 1250.00,
      "description": "Alice's share",
      "notes": "Extra appetizer"
    },
    {
      "personId": "<uuid>",
      "amount": 1250.00,
      "description": "Bob's share"
    }
  ]
}
```

#### READ (All Entries)
- **Endpoint**: `GET /api/entries`
- **Returns**: Group expense entries where user is lender

#### READ (Entry by ID)
- **Endpoint**: `GET /api/entries/{id}`
- **Returns**: EntryDTO with group borrower info

#### READ (Payment Allocations)
- **Endpoint**: `GET /api/payment-allocations/entry/{entryId}`
- **Returns**: List of PaymentAllocationDTOs showing:
  - Each member's allocated amount
  - Payment status (computed from actual payments)
  - Percentage of total expense

**Status Computation**:
- **UNPAID**: No payments made by this person for this entry
- **PARTIALLY_PAID**: Some payments, but less than allocated amount
- **PAID**: Payments equal or exceed allocated amount

#### UPDATE (Payment Allocation)
- **Endpoint**: `PUT /api/payment-allocations/{id}`
- **Flow**: Updates allocation amount, description, or notes

#### DELETE (Payment Allocation)
- **Endpoint**: `DELETE /api/payment-allocations/{id}`
- **Flow**: Removes allocation record

#### PAYMENTS (For Group Expense)
- Payments are created via Payment CRUD (same as other entries)
- When a payment is made by a group member:
  - Payment is linked to entry via `PaymentEntry`
  - PaymentAllocationService recomputes status based on total payments by that person
  - Entry `amountRemaining` decreases

**Code Path for Payment Creation**:
```
PaymentService.createPayment()
  → Creates Payment record
  → Links to Entry via PaymentEntry
  → Updates entry.amountRemaining
  → PaymentAllocationService.computeStatus() (called on next read)
```

#### DELETE (Entry)
- **Endpoint**: `DELETE /api/entries/{id}`
- **Flow**: Deletes entry, cascades to payment allocations

---

## Additional Features

### Payment Management
- **Create Payment**: `POST /api/payments`
  - Links payment to entry
  - Updates entry `amountRemaining` and status automatically
  - Status becomes PARTIALLY_PAID when payment < amountBorrowed
  - Status becomes PAID when amountRemaining = 0

### Reference ID Generation
- Each entry gets a unique `referenceId`
- Format: Based on borrower and lender names (e.g., "ALICE-BOB-1")
- Ensures uniqueness by appending counter if duplicate

### User Context
- All operations filter by current user
- User identified by name in HTTP header `X-Selected-User-Name`
- Default user: "tung tung tung sahur"
- User must be lender or borrower to access entry

### Status Tracking
- **Entry Status**: UNPAID, PARTIALLY_PAID, PAID
- **Installment Term Status**: NOT_STARTED, UNPAID, PAID, SKIPPED, DELINQUENT
- **Payment Allocation Status**: UNPAID, PARTIALLY_PAID, PAID

---

## Summary of CRUD Flow

**Create Pattern**:
1. Frontend validates form data
2. Sends POST request with DTO
3. Controller receives request
4. Service validates and creates entity
5. Repository saves to database
6. Returns created DTO

**Read Pattern**:
1. Frontend sends GET request
2. Service filters by user context
3. Repository queries database
4. Service converts entities to DTOs
5. Returns DTO list or single DTO

**Update Pattern**:
1. Frontend sends PUT request with updated data
2. Service validates user access
3. Service updates entity fields
4. Repository saves
5. Returns updated DTO

**Delete Pattern**:
1. Frontend sends DELETE request
2. Service validates user access
3. Repository deletes entity (cascade handled by JPA)
4. Returns 204 No Content
