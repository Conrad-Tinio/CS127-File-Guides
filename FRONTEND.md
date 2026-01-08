# Frontend Functions Overview

This document provides a detailed overview of the frontend functions that are primarily and most frequently used across the available features in the project.

## Table of Contents
1. [API Service Functions](#api-service-functions)
2. [Utility Functions](#utility-functions)
3. [Component-Level Functions](#component-level-functions)
4. [Feature-Specific Function Usage](#feature-specific-function-usage)

---

## API Service Functions

The project uses a centralized API service (`frontend/src/services/api.ts`) that provides functions organized by entity type. Here are the most frequently used functions across the application:

### Entry API (`entryApi`)

#### **Most Frequently Used:**

1. **`entryApi.getAll()`** - ⭐⭐⭐ **HIGHEST USAGE**
   - **Used in:** HomePage, AllPaymentsPage, PaymentHistoryPage
   - **Purpose:** Fetches all entries to display in lists, dashboards, and for filtering
   - **Return Type:** `Promise<AxiosResponse<Entry[]>>`
   - **Usage Pattern:** Called on component mount to populate entry lists

2. **`entryApi.getById(id: string)`** - ⭐⭐⭐ **HIGHEST USAGE**
   - **Used in:** EntryDetailPage, CreateEntryPage
   - **Purpose:** Fetches a specific entry's details by ID
   - **Return Type:** `Promise<AxiosResponse<Entry>>`
   - **Usage Pattern:** Called when navigating to entry detail view or verifying entry creation

3. **`entryApi.create(request: CreateEntryRequest)`** - ⭐⭐ **HIGH USAGE**
   - **Used in:** CreateEntryPage
   - **Purpose:** Creates a new entry (straight expense, installment, or group expense)
   - **Return Type:** `Promise<AxiosResponse<Entry>>`
   - **Features:** Supports file upload via `createWithProof()` variant

4. **`entryApi.createWithProof(request: CreateEntryRequest, file: File)`** - ⭐⭐ **HIGH USAGE**
   - **Used in:** CreateEntryPage
   - **Purpose:** Creates entry with proof document/image upload
   - **Return Type:** `Promise<AxiosResponse<Entry>>`
   - **Details:** Uses multipart/form-data for file upload

5. **`entryApi.update(id: string, request: CreateEntryRequest)`** - ⭐ **MEDIUM USAGE**
   - **Used in:** EntryDetailPage
   - **Purpose:** Updates existing entry information
   - **Return Type:** `Promise<AxiosResponse<Entry>>`

6. **`entryApi.delete(id: string)`** - ⭐ **MEDIUM USAGE**
   - **Used in:** EntryDetailPage
   - **Purpose:** Deletes an entry
   - **Return Type:** `Promise<void>`

7. **`entryApi.complete(id: string)`** - ⭐ **MEDIUM USAGE**
   - **Used in:** EntryDetailPage
   - **Purpose:** Marks an entry as fully paid/completed
   - **Return Type:** `Promise<AxiosResponse<Entry>>`

---

### Person API (`personApi`)

#### **Most Frequently Used:**

1. **`personApi.getAll()`** - ⭐⭐⭐ **HIGHEST USAGE**
   - **Used in:** PersonSelector, PeopleGroupsPage, UserSelectionModal
   - **Purpose:** Fetches all persons for dropdowns, lists, and selection
   - **Return Type:** `Promise<AxiosResponse<Person[]>>`
   - **Usage Pattern:** Loaded on component mount in selectors and management pages

2. **`personApi.search(name: string)`** - ⭐⭐ **HIGH USAGE**
   - **Used in:** PersonSelector
   - **Purpose:** Searches for persons by name (debounced search)
   - **Return Type:** `Promise<AxiosResponse<Person[]>>`
   - **Details:** Used with 300ms debounce for efficient searching

3. **`personApi.create(person: Omit<Person, 'personId'>)`** - ⭐⭐ **HIGH USAGE**
   - **Used in:** PersonSelector, PeopleGroupsPage
   - **Purpose:** Creates a new person on-the-fly during entry creation or management
   - **Return Type:** `Promise<AxiosResponse<Person>>`
   - **Feature:** Allows inline creation from selectors

4. **`personApi.getById(id: string)`** - ⭐ **MEDIUM USAGE**
   - **Used in:** PersonSelector
   - **Purpose:** Fetches a specific person by ID
   - **Return Type:** `Promise<AxiosResponse<Person>>`
   - **Usage:** Used when initializing selectors with a pre-selected person

5. **`personApi.update(id: string, person: Omit<Person, 'personId'>)`** - ⭐ **MEDIUM USAGE**
   - **Used in:** PeopleGroupsPage
   - **Purpose:** Updates person information (typically name)
   - **Return Type:** `Promise<AxiosResponse<Person>>`

6. **`personApi.delete(id: string)`** - ⭐ **MEDIUM USAGE**
   - **Used in:** PeopleGroupsPage
   - **Purpose:** Deletes a person
   - **Return Type:** `Promise<void>`

---

### Group API (`groupApi`)

#### **Most Frequently Used:**

1. **`groupApi.getAll()`** - ⭐⭐⭐ **HIGHEST USAGE**
   - **Used in:** GroupSelector, PeopleGroupsPage
   - **Purpose:** Fetches all groups for selection and management
   - **Return Type:** `Promise<AxiosResponse<Group[]>>`
   - **Usage Pattern:** Loaded on component mount

2. **`groupApi.getById(id: string)`** - ⭐⭐ **HIGH USAGE**
   - **Used in:** GroupSelector, EntryDetailPage, PeopleGroupsPage
   - **Purpose:** Fetches a specific group with its members
   - **Return Type:** `Promise<AxiosResponse<Group>>`
   - **Details:** Essential for group expense entries to load member information

3. **`groupApi.create(group: Omit<Group, 'groupId'>)`** - ⭐⭐ **HIGH USAGE**
   - **Used in:** GroupSelector, PeopleGroupsPage
   - **Purpose:** Creates a new group (with optional initial members)
   - **Return Type:** `Promise<AxiosResponse<Group>>`
   - **Feature:** Allows inline creation from selectors

4. **`groupApi.addMember(groupId: string, personId: string)`** - ⭐⭐ **HIGH USAGE**
   - **Used in:** PeopleGroupsPage
   - **Purpose:** Adds a person to a group
   - **Return Type:** `Promise<void>`

5. **`groupApi.removeMember(groupId: string, personId: string)`** - ⭐ **MEDIUM USAGE**
   - **Used in:** PeopleGroupsPage
   - **Purpose:** Removes a person from a group
   - **Return Type:** `Promise<void>`

6. **`groupApi.update(id: string, group: Omit<Group, 'groupId'>)`** - ⭐ **MEDIUM USAGE**
   - **Used in:** PeopleGroupsPage
   - **Purpose:** Updates group information (name, members)
   - **Return Type:** `Promise<AxiosResponse<Group>>`

7. **`groupApi.delete(id: string)`** - ⭐ **MEDIUM USAGE**
   - **Used in:** PeopleGroupsPage
   - **Purpose:** Deletes a group
   - **Return Type:** `Promise<void>`

---

### Payment API (`paymentApi`)

#### **Most Frequently Used:**

1. **`paymentApi.getAll()`** - ⭐⭐ **HIGH USAGE**
   - **Used in:** PaymentHistoryPage
   - **Purpose:** Fetches all payments across all entries for history view
   - **Return Type:** `Promise<AxiosResponse<Payment[]>>`
   - **Usage:** Combined with entry data for comprehensive payment tracking

2. **`paymentApi.getByEntry(entryId: string)`** - ⭐⭐⭐ **HIGHEST USAGE**
   - **Used in:** EntryDetailPage
   - **Purpose:** Fetches all payments for a specific entry
   - **Return Type:** `Promise<AxiosResponse<Payment[]>>`
   - **Usage Pattern:** Called when viewing entry details to display payment history

3. **`paymentApi.create(request: CreatePaymentRequest)`** - ⭐⭐ **HIGH USAGE**
   - **Used in:** EntryDetailPage
   - **Purpose:** Records a new payment for an entry
   - **Return Type:** `Promise<AxiosResponse<Payment>>`
   - **Features:** Supports linking to payment allocations for group expenses

4. **`paymentApi.createWithProof(request: CreatePaymentRequest, file: File)`** - ⭐⭐ **HIGH USAGE**
   - **Used in:** EntryDetailPage
   - **Purpose:** Records payment with proof document/image
   - **Return Type:** `Promise<AxiosResponse<Payment>>`
   - **Details:** Uses multipart/form-data for file upload

5. **`paymentApi.update(id: string, request: CreatePaymentRequest)`** - ⭐ **LOW USAGE**
   - **Purpose:** Updates payment information
   - **Return Type:** `Promise<AxiosResponse<Payment>>`

6. **`paymentApi.delete(id: string)`** - ⭐ **LOW USAGE**
   - **Purpose:** Deletes a payment
   - **Return Type:** `Promise<void>`

---

### Payment Allocation API (`paymentAllocationApi`)

#### **Most Frequently Used:**

1. **`paymentAllocationApi.getByEntry(entryId: string)`** - ⭐⭐ **HIGH USAGE**
   - **Used in:** EntryDetailPage
   - **Purpose:** Fetches payment allocations for a group expense entry
   - **Return Type:** `Promise<AxiosResponse<PaymentAllocation[]>>`
   - **Details:** Only used for GROUP_EXPENSE transaction types

2. **`paymentAllocationApi.create(entryId: string, allocations: any[])`** - ⭐⭐ **HIGH USAGE**
   - **Used in:** CreateEntryPage, EntryDetailPage
   - **Purpose:** Creates payment allocations when setting up group expense splits
   - **Return Type:** `Promise<AxiosResponse<PaymentAllocation[]>>`
   - **Features:** Supports equal, percentage, and amount-based allocation modes

3. **`paymentAllocationApi.update(id: string, allocation: Partial<PaymentAllocation>)`** - ⭐ **MEDIUM USAGE**
   - **Used in:** EntryDetailPage
   - **Purpose:** Updates an individual payment allocation
   - **Return Type:** `Promise<AxiosResponse<PaymentAllocation>>`

4. **`paymentAllocationApi.delete(id: string)`** - ⭐⭐ **HIGH USAGE**
   - **Used in:** EntryDetailPage
   - **Purpose:** Deletes a payment allocation (used when resetting allocations)
   - **Return Type:** `Promise<void>`
   - **Usage Pattern:** Often called in bulk using `Promise.all()` to clear existing allocations before creating new ones

---

### Installment API (`installmentApi`)

#### **Most Frequently Used:**

1. **`installmentApi.skipTerm(termId: string)`** - ⭐ **MEDIUM USAGE**
   - **Used in:** EntryDetailPage
   - **Purpose:** Marks an installment term as skipped
   - **Return Type:** `Promise<void>`

2. **`installmentApi.getSkipPenalty(termId: string)`** - ⭐ **MEDIUM USAGE**
   - **Used in:** EntryDetailPage
   - **Purpose:** Calculates penalty for skipping a term
   - **Return Type:** `Promise<AxiosResponse<{ penalty: number }>>`

3. **`installmentApi.updateTermStatus(termId: string, status: string)`** - ⭐ **MEDIUM USAGE**
   - **Used in:** EntryDetailPage
   - **Purpose:** Updates installment term status (e.g., to 'PAID')
   - **Return Type:** `Promise<void>`

4. **`installmentApi.updateDelinquent()`** - ⭐ **LOW USAGE**
   - **Purpose:** Marks overdue terms as delinquent
   - **Return Type:** `Promise<void>`

---

## Utility Functions

### Currency Utilities (`frontend/src/utils/currency.ts`)

#### **Most Frequently Used:**

1. **`parseCurrencyToNumber(value: string): number`** - ⭐⭐⭐ **HIGHEST USAGE**
   - **Used in:** CreateEntryPage, EntryDetailPage
   - **Purpose:** Converts currency string input (with commas, decimals) to a numeric value
   - **Features:**
     - Handles comma separators
     - Validates numeric format
     - Returns 0 for invalid input
   - **Example:** `"1,234.56"` → `1234.56`

2. **`formatCurrencyInput(value: string): string`** - ⭐⭐ **HIGH USAGE**
   - **Used in:** CreateEntryPage, EntryDetailPage
   - **Purpose:** Formats a currency string with proper thousands separators and decimal places
   - **Features:**
     - Ensures 2 decimal places
     - Adds thousand separators
     - Used on input blur for user feedback
   - **Example:** `"1234.5"` → `"1,234.50"`

---

## Component-Level Functions

### PersonSelector Component

**Key Internal Functions:**
- `loadAllPersons()` - Loads all persons on mount
- `searchPersons()` - Debounced search (300ms) for persons
- `handleSelectPerson()` - Handles person selection
- `handleCreateNewPerson()` - Creates new person inline
- `getFilteredPersons()` - Filters persons by allowed/excluded lists

**API Functions Used:**
- `personApi.getAll()` - Initial load
- `personApi.search()` - Search functionality
- `personApi.getById()` - Load selected person
- `personApi.create()` - Inline creation

### GroupSelector Component

**Key Internal Functions:**
- `loadAllGroups()` - Loads all groups on mount
- `handleSelectGroup()` - Handles group selection
- `handleCreateNewGroup()` - Creates new group inline
- Client-side filtering by group name

**API Functions Used:**
- `groupApi.getAll()` - Initial load
- `groupApi.getById()` - Load selected group
- `groupApi.create()` - Inline creation

---

## Feature-Specific Function Usage

### Dashboard/HomePage
- **Primary Function:** `entryApi.getAll()`
- **Purpose:** Display entry statistics, recent entries, financial overview
- **Data Processing:** Client-side filtering and aggregation (status counts, totals)

### Entry Creation (CreateEntryPage)
- **Primary Functions:**
  - `entryApi.create()` / `entryApi.createWithProof()`
  - `paymentAllocationApi.create()` (for group expenses)
  - `entryApi.getById()` (verification after creation)
- **Supporting Functions:**
  - `personApi.getAll()`, `personApi.search()`, `personApi.create()` (via PersonSelector)
  - `groupApi.getAll()`, `groupApi.create()` (via GroupSelector)
  - `parseCurrencyToNumber()`, `formatCurrencyInput()` (currency handling)

### Entry Detail (EntryDetailPage)
- **Primary Functions:**
  - `entryApi.getById()` - Load entry details
  - `paymentApi.getByEntry()` - Load payment history
  - `paymentAllocationApi.getByEntry()` - Load allocations (group expenses)
  - `groupApi.getById()` - Load group info (group expenses)
  - `paymentApi.create()` / `paymentApi.createWithProof()` - Record payments
  - `paymentAllocationApi.create()` - Create/update allocations
  - `paymentAllocationApi.update()` - Modify allocations
  - `paymentAllocationApi.delete()` - Remove allocations
  - `entryApi.update()` - Edit entry
  - `entryApi.delete()` - Delete entry
  - `entryApi.complete()` - Mark as complete
  - `installmentApi.skipTerm()` - Skip installment term
  - `installmentApi.getSkipPenalty()` - Calculate penalties
  - `installmentApi.updateTermStatus()` - Update term status

### People & Groups Management (PeopleGroupsPage)
- **Primary Functions:**
  - `personApi.getAll()` - Load all persons
  - `groupApi.getAll()` - Load all groups
  - `personApi.create()` - Create person
  - `personApi.update()` - Update person
  - `personApi.delete()` - Delete person
  - `groupApi.create()` - Create group
  - `groupApi.update()` - Update group
  - `groupApi.delete()` - Delete group
  - `groupApi.addMember()` - Add person to group
  - `groupApi.removeMember()` - Remove person from group
  - `groupApi.getById()` - Refresh group after member changes

### Payment History (PaymentHistoryPage)
- **Primary Functions:**
  - `paymentApi.getAll()` - Load all payments
  - `entryApi.getAll()` - Load entries for cross-referencing
- **Data Processing:** Combines payment and entry data, filters by date range and search term

### All Entries (AllPaymentsPage)
- **Primary Function:** `entryApi.getAll()`
- **Purpose:** Display all entries with filtering and search
- **Data Processing:** Client-side filtering by status, type, and search term

---

## API Request Interceptor

**Special Feature:** The API service includes a request interceptor that automatically adds user context to all requests:
- `X-Selected-User-Id` header
- `X-Selected-User-Name` header
- Defaults to "tung tung tung sahur" if no user is selected

This ensures all API calls include the current user context for proper data filtering and permissions.

---

## Summary Statistics

### Most Frequently Used API Functions (by call count):

1. **`entryApi.getAll()`** - 4 pages/components
2. **`personApi.getAll()`** - 3 pages/components
3. **`groupApi.getAll()`** - 2 pages/components
4. **`entryApi.getById()`** - 2 pages
5. **`paymentApi.getByEntry()`** - 1 page (but critical feature)
6. **`personApi.create()`** - 3 locations
7. **`groupApi.create()`** - 2 locations
8. **`paymentAllocationApi.create()`** - 2 pages

### Utility Functions Usage:

1. **`parseCurrencyToNumber()`** - Used in 2 major pages (entry creation & detail)
2. **`formatCurrencyInput()`** - Used in 2 major pages for user input formatting

---

## Patterns and Best Practices

1. **Data Loading Pattern:**
   - Most pages load data on mount using `useEffect`
   - Data is reloaded after mutations (create/update/delete)

2. **Error Handling:**
   - All API calls use try-catch blocks
   - Error messages extracted from `error.response.data.error` or `error.message`
   - Toast notifications used for user feedback

3. **Loading States:**
   - Loading indicators shown during async operations
   - Separate loading states for different operations

4. **File Uploads:**
   - Separate `createWithProof`/`createWithProof` functions for file uploads
   - Uses FormData with multipart/form-data encoding

5. **Inline Creation:**
   - PersonSelector and GroupSelector support inline creation
   - Users can create entities without leaving the current form

6. **Verification Pattern:**
   - After creating entries (especially GROUP_EXPENSE), verification retries are used
   - Helps avoid race conditions with database transactions

---

This overview demonstrates that the frontend relies heavily on CRUD operations for entries, persons, groups, payments, and payment allocations, with special handling for installment management and file uploads. The architecture follows a centralized API service pattern with utility functions for common tasks like currency formatting.

