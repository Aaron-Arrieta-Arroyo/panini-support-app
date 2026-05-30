# Technical Decisions — Panini Support App

Handoff document for engineers continuing the project.  
Explains **what** was built, **why** each tool was chosen, and **how** it maps to the API contract and the running app.

---

## 1. Architectural pattern: MVVM + Repository

### Why MVVM

MVVM separates UI (Composables) from state (ViewModel) and data (Repository).  
The concrete benefit for this project:

- The ViewModel does not import any Android UI types (`Context`, `Activity`, etc.), so it can be tested without an emulator.
- The Composable only observes a `StateFlow` — it does not decide or process, it only renders.
- The Repository is an interface: switching from mock to a real backend does not touch the ViewModel or the screen.

### Why not Clean Architecture with UseCases

A short-term PoC with a small team **does not justify** the overhead of UseCases. Adding them here would be complexity without real benefit today.

If the project grows: inserting UseCases between ViewModel and Repository is a local change that does not require touching screens or repositories.

### Layer structure

```
presentation/   Composables + ViewModels (observe StateFlow)
domain/         Business models + TicketRepository interface
data/           DTOs, Retrofit, MockTicketRepository
core/           Cross-cutting tools (events, feature flags)
```

### Dependency injection

Manual DI lives in `SupportApp.kt`:

```kotlin
class AppContainer {
    val ticketRepository: TicketRepository = MockTicketRepository()
}
```

`MainActivity` reads `SupportApp.container.ticketRepository` and passes it into navigation. No Hilt/Koin — intentional for PoC scope.

---

## 2. Application screens

| Screen | Route | ViewModel | Data source |
|---|---|---|---|
| Login | `login` | `LoginViewModel` | Simulated locally (`admin` / `admin123`) — not wired to `POST /auth/login` yet |
| Ticket list | `ticket_list` | `TicketListViewModel` | `TicketRepository.getTickets()` |
| Ticket detail | `ticket_detail/{ticketId}` | `TicketDetailViewModel` | `TicketRepository.getTicketById()` |
| Create ticket | `create_ticket` | `CreateTicketViewModel` | `TicketRepository.createTicket()` |
| Settings | `settings` | None | Reads/writes `FeatureFlags` directly |

All ticket screens depend on `TicketRepository`, never on Retrofit or mock classes directly.

---

## 3. Event-based communication: SharedFlow

### Problem it solves

When the user creates a ticket or changes its status or priority, the list screen must update **without the user having to go back and reload**.

The obvious solution (calling `loadTickets()` after every action) would work, but it would re-run the full repository call — in production that is an unnecessary network request.

### Solution: `TicketEventBus` with `SharedFlow`

```kotlin
sealed class TicketEvent {
    data class TicketCreated(val ticket: Ticket) : TicketEvent()
    data class PriorityUpdated(val ticketId: String, val newPriority: Priority) : TicketEvent()
    data class StatusUpdated(val ticketId: String, val newStatus: TicketStatus) : TicketEvent()
}
```

```kotlin
object TicketEventBus {
    private val _events = MutableSharedFlow<TicketEvent>()
    val events: SharedFlow<TicketEvent> = _events.asSharedFlow()
    suspend fun emit(event: TicketEvent) = _events.emit(event)
}
```

It is an application-level event bus. Any ViewModel can emit or listen.

**Flow for ticket creation:**

```
CreateTicketViewModel
  → repository.createTicket(ticket)
  → TicketEventBus.emit(TicketCreated)

TicketListViewModel (collector in viewModelScope)
  → inserts ticket into local list
  → re-sorts by priority (HIGH sortOrder=0 first)
  → Composable recomposes
```

**Flow for status update:**

```
TicketDetailViewModel
  → repository.updateTicketStatus(id, status)
  → TicketEventBus.emit(StatusUpdated)

TicketListViewModel
  → updates ticket status in local list
  → Composable recomposes
```

**Flow for priority update:**

```
TicketDetailViewModel
  → repository.updateTicketPriority(id, priority)
  → TicketEventBus.emit(PriorityUpdated)

TicketListViewModel
  → updates priority and re-sorts list
  → Composable recomposes — ticket moves up/down without reload
```

### Why SharedFlow and not other options

| Alternative | Reason not to use it |
|---|---|
| Call `loadTickets()` again | Unnecessary re-fetch, not truly reactive |
| `LiveData` | Requires `LifecycleOwner`, not Kotlin-native |
| Callback between ViewModels | Tight coupling, hard to maintain |
| `SharedFlow` ✓ | Lifecycle-independent, supports multiple collectors, native Kotlin coroutines |

`replay = 0` (default): a new collector does not receive past events.

---

## 4. Feature Flags

### Implementation

```kotlin
object FeatureFlags {
    var ticketCreationEnabled by mutableStateOf(true)
    var priorityUpdateEnabled by mutableStateOf(true)
}
```

They use Compose `mutableStateOf`: when the value changes from `SettingsScreen`, every Composable that reads the flag recomposes automatically.

| Flag | UI effect |
|---|---|
| `ticketCreationEnabled` | Shows/hides the Create Ticket FAB on the list |
| `priorityUpdateEnabled` | Shows/hides the **Change Priority** button on ticket detail |

Status updates remain available even when priority updates are disabled.

---

## 5. Ticket list UI

The list shows a **compact card** with three elements only (full data stays on the detail screen):

1. **Title**
2. **Priority badge** — vivid colors: High red, Medium yellow, Low green
3. **Status badge** — Open red, In Progress blue, Resolved green, Closed gray

Detail screen shows all fields: id, title, status, priority, supplier, category, created date, assignee, affected units, description, and action buttons.

---

## 6. Networking and API contract alignment

The backend is not deployed yet. The mobile app is ready to consume `/contracts/tickets-api.yaml` via Retrofit.

### Endpoints (contract ↔ `TicketApiService`)

| Method | Path | Retrofit method | Repository method |
|---|---|---|---|
| `POST` | `/auth/login` | `login()` | Not wired in PoC (login is simulated) |
| `GET` | `/tickets` | `getTickets()` | `getTickets()` |
| `POST` | `/tickets` | `createTicket()` | `createTicket()` |
| `GET` | `/tickets/{id}` | `getTicketById()` | `getTicketById()` |
| `PATCH` | `/tickets/{id}/status` | `updateTicketStatus()` | `updateTicketStatus()` |
| `PATCH` | `/tickets/{id}/priority` | `updateTicketPriority()` | `updateTicketPriority()` |

**Base URL** (contract server + `NetworkClient`): `https://api.panini-support.com/v1`

### Domain model ↔ OpenAPI schemas

| Domain (`domain/model/`) | API schema / field | Wire format |
|---|---|---|
| `Ticket.id` | `Ticket.id` | string, e.g. `TKT-001` |
| `Ticket.title` | `Ticket.title` | string |
| `Ticket.description` | `Ticket.description` | string |
| `Priority` enum | `priority` | `HIGH`, `MEDIUM`, `LOW` |
| `TicketStatus` enum | `status` | `OPEN`, `IN_PROGRESS`, `RESOLVED`, `CLOSED` |
| `TicketCategory` enum | `category` | `INVENTORY`, `DISTRIBUTION`, `LOGISTICS`, `SUPPLIER`, `QUALITY` |
| `Ticket.supplier` | `supplier` | string |
| `Ticket.createdAt` | `created_at` | ISO-8601 date-time |
| `Ticket.assignedTo` | `assigned_to` | string, nullable |
| `Ticket.affectedQuantity` | `affected_quantity` | integer, nullable |

Display labels (`High`, `Open`, `Inventory`, etc.) exist only in domain enums for UI. The API and DTOs use enum **names** as strings.

### DTO mapping (`data/dto/`)

| OpenAPI schema | Kotlin DTO |
|---|---|
| `LoginRequest` | `LoginRequestDto` |
| `LoginResponse` / `User` | `LoginResponseDto` / `UserDto` |
| `Ticket` | `TicketDto` |
| `TicketListResponse` | `TicketListResponseDto` |
| `CreateTicketRequest` | `CreateTicketRequestDto` |
| `UpdateStatusRequest` | `UpdateStatusRequestDto` |
| `UpdatePriorityRequest` | `UpdatePriorityRequestDto` |

`RemoteTicketRepository` maps `TicketDto` → domain `Ticket` via `toDomain()`.

### Activate the real backend

Change **one line** in `SupportApp.kt` (`AppContainer`):

```kotlin
// Current (mock):
val ticketRepository: TicketRepository = MockTicketRepository()

// Production:
val ticketRepository: TicketRepository = RemoteTicketRepository(NetworkClient.ticketApiService)
```

Next step for auth: call `TicketApiService.login()`, store the JWT, and add a Bearer interceptor in `NetworkClient`.

---

## 7. Mock data

`MockData.kt` contains **10 tickets** aligned with the contract examples (e.g. `TKT-001`, Central Distributor S.A., Escazú scenario). Scenarios cover:

- Inventory shortages at points of sale
- Missed supplier deliveries
- Batch count errors
- Packaging damage in transit
- Invalid QR codes on packs
- Duplicate ERP orders
- Tracking system failures

`MockTicketRepository` simulates latency (`delay`) and keeps mutations in memory for the session. Lists are sorted by `Priority.sortOrder` (HIGH=0 first), matching `GET /tickets` contract description.

---

## 8. Screen state handling

All async screens use `UiState<T>`:

```kotlin
sealed class UiState<out T> {
    data object Loading : UiState<Nothing>()
    data class Success<T>(val data: T) : UiState<T>()
    data class Error(val message: String) : UiState<Nothing>()
}
```

Used by: ticket list, ticket detail (load + actions), create ticket (submit).

---

## 9. Continuing the project

**New screen:** add route in `Screen.kt` → composable in `AppNavigation.kt` → Screen + ViewModel under `presentation/`.

**New endpoint:** add to `contracts/tickets-api.yaml` → `TicketApiService.kt` → DTO → `RemoteTicketRepository` + `MockTicketRepository`.

**New feature flag:** add property in `FeatureFlags.kt` → read in Composable → toggle in `SettingsScreen.kt`.

**Minimum stack:** Kotlin, Jetpack Compose, Coroutines, StateFlow/SharedFlow, MVVM + Repository, Retrofit + Gson.

**Single source of truth for the API:** `/contracts/tickets-api.yaml`
