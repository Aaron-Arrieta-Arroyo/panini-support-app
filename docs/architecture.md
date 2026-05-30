# Technical Decisions — Panini Support App

Handoff document for engineers continuing the project.  
This file is the single source of truth for `/docs`. It explains **what** was built, **why** each decision was made, and **how** it maps to the running app and the API contract.

**Audience:** mobile/backend engineers joining the Panini Support PoC.  
**Related artifacts:** `/contracts/tickets-api.yaml`, `/video/demo-link.md`, root `README.md`.

---

## Table of contents

| Section | Topic |
|---|---|
| [1](#1-architectural-decisions) | Architectural decisions (MVVM + Repository) |
| [2](#2-general-system-flow) | General system flow (startup → screens → data → events) |
| [3](#3-application-screens) | Application screens and navigation |
| [4](#4-event-based-communication) | Event-based communication (SharedFlow) |
| [5](#5-feature-flags) | Feature Flags |
| [6](#6-data-layer-and-api-alignment) | Data layer and API contract alignment |
| [7](#7-mock-data) | Mock data |
| [8](#8-ui-and-state-handling) | UI and state handling |
| [9](#9-package-map) | Package map (what each folder/file does) |
| [10](#10-extending-the-project) | Extending the project |

---

## 1. Architectural decisions

### 1.1 Pattern: MVVM + Repository

MVVM separates three concerns:

| Layer | Responsibility | In this project |
|---|---|---|
| **View (Composable)** | Draw UI, react to state | `*Screen.kt` files |
| **ViewModel** | Hold state, run screen logic | `*ViewModel.kt` files |
| **Repository** | Provide data (mock or remote) | `TicketRepository` interface + implementations |

**Why MVVM here**

- ViewModels do not import Android UI types (`Context`, `Activity`), so they can be unit-tested without an emulator.
- Composables observe `StateFlow` and only render — they do not fetch data or emit domain events directly.
- The Repository is an **interface**. Swapping mock → remote backend does not touch ViewModels or Composables.

**Why not Clean Architecture with UseCases**

A short-term PoC with a small team does not justify UseCase overhead today. If the project grows, UseCases can be inserted between ViewModel and Repository without changing screens.

### 1.2 Layer structure

```
presentation/   Composables + ViewModels (observe StateFlow)
domain/         Business models + TicketRepository interface
data/           DTOs, Retrofit, MockTicketRepository, RemoteTicketRepository
core/           TicketEventBus, FeatureFlags (cross-cutting)
```

**Golden rule:** screens and ViewModels never call Retrofit or `MockData` directly. They only know `TicketRepository`.

### 1.3 Dependency injection

Manual DI lives in `SupportApp.kt` — no Hilt/Koin (intentional for PoC scope):

```kotlin
class AppContainer {
    val ticketRepository: TicketRepository = MockTicketRepository()
}
```

`MainActivity` reads `SupportApp.container.ticketRepository` and passes it into `AppNavigation`.

**To activate the real backend:** change one line to `RemoteTicketRepository(NetworkClient.ticketApiService)`.

### 1.4 Technology choices (technical justification)

| Choice | Why |
|---|---|
| **Jetpack Compose** | Declarative UI, less boilerplate than XML, fits MVVM with state observation |
| **StateFlow** | ViewModel state exposed to Composables; lifecycle-aware collection via `collectAsStateWithLifecycle` |
| **SharedFlow** | Event bus decouples screens; multiple ViewModels can listen without references to each other |
| **Navigation Compose** | Type-safe routes in `Screen.kt`, single `NavHost` in `AppNavigation.kt` |
| **Retrofit + Gson** | Matches OpenAPI contract in `/contracts`; ready when backend exists |
| **Coroutines + suspend** | Repository methods are async; mock uses `delay()` to simulate network latency |
| **Manual DI** | One swap point in `AppContainer`; no framework setup cost for a PoC |

---

## 2. General system flow

This section describes the **end-to-end flow** from app launch to ticket updates.

### 2.1 Application startup

```
Android OS
  → AndroidManifest.xml (LAUNCHER → MainActivity, Application → SupportApp)
  → MainActivity.onCreate()
      → enableEdgeToEdge()
      → SupportApp.container.ticketRepository  (MockTicketRepository today)
      → setContent { SupportAppTheme { AppNavigation(...) } }
  → AppNavigation startDestination = login
  → LoginScreen
```

**Files involved:** `AndroidManifest.xml`, `MainActivity.kt`, `SupportApp.kt`, `AppNavigation.kt`.

### 2.2 User navigation flow

```
Login (admin / admin123, simulated)
  → Ticket List  ←── main hub after login
        ├── tap ticket  → Ticket Detail (ticketId in route)
        ├── FAB "+"     → Create Ticket   (hidden if ticketCreationEnabled = false)
        └── gear icon   → Settings        (Feature Flag toggles)
```

After successful login, the login route is removed from the back stack (`popUpTo` inclusive), so the user cannot return to login with the back button.

**Files involved:** `Screen.kt`, `AppNavigation.kt`, each `*Screen.kt`.

### 2.3 Data flow (read/write tickets)

All ticket operations follow the same path:

```
Composable (*Screen)
  → observes ViewModel StateFlow
  → user action triggers ViewModel method
  → ViewModel calls TicketRepository (interface)
  → MockTicketRepository (today) OR RemoteTicketRepository (production)
        Mock:  MockData in memory + delay()
        Remote: TicketApiService → JSON DTO → toDomain() → Ticket
  → ViewModel updates StateFlow
  → Composable recomposes
```

**Repository methods (domain contract):**

| Method | Used by | Purpose |
|---|---|---|
| `getTickets()` | `TicketListViewModel` | Load list on screen open |
| `getTicketById(id)` | `TicketDetailViewModel` | Load one ticket |
| `createTicket(ticket)` | `CreateTicketViewModel` | Create new ticket |
| `updateTicketStatus(id, status)` | `TicketDetailViewModel` | Change lifecycle state |
| `updateTicketPriority(id, priority)` | `TicketDetailViewModel` | Change priority |

Login does **not** use `TicketRepository`. It is simulated in `LoginViewModel` (`admin` / `admin123`). `POST /auth/login` exists in Retrofit for a future phase.

### 2.4 Event flow (list updates without reload)

When a ticket is created or modified on another screen, the **list must update without calling `getTickets()` again**.

```
                    ┌─────────────────────┐
                    │   TicketEventBus    │
                    │   (SharedFlow)      │
                    └──────────▲──────────┘
                               │ emit
         ┌─────────────────────┼─────────────────────┐
         │                     │                     │
 CreateTicketVM          TicketDetailVM         (future VMs)
 TicketCreated           StatusUpdated
                         PriorityUpdated
                               │
                               │ collect
                               ▼
                    ┌─────────────────────┐
                    │ TicketListViewModel │
                    │ observeEvents()     │
                    │ → update local list │
                    │ → re-sort by priority│
                    └─────────────────────┘
                               │
                               ▼
                    TicketListScreen recomposes
```

**Emitters (who publishes events):**

| Event | Emitted after | By |
|---|---|---|
| `TicketCreated` | `repository.createTicket()` succeeds | `CreateTicketViewModel` |
| `StatusUpdated` | `repository.updateTicketStatus()` succeeds | `TicketDetailViewModel` |
| `PriorityUpdated` | `repository.updateTicketPriority()` succeeds | `TicketDetailViewModel` |

**Subscriber (who reacts):** `TicketListViewModel.observeEvents()` — runs for the lifetime of the list ViewModel in `viewModelScope`.

### 2.5 Feature Flag flow

```
SettingsScreen
  → user toggles Switch
  → FeatureFlags.ticketCreationEnabled or priorityUpdateEnabled changes (mutableStateOf)
  → any Composable reading the flag recomposes immediately
        TicketListScreen: FAB show/hide
        TicketDetailScreen: "Change Priority" button show/hide
```

No ViewModel is required for flags — intentional simplicity for this PoC.

### 2.6 Complete scenario: create a ticket and see it in the list

```
1. User opens Create Ticket (FAB on list)
2. CreateTicketScreen collects form input via CreateTicketViewModel
3. User taps "Create Ticket"
4. CreateTicketViewModel.createTicket()
     a. Validates title, description, supplier
     b. repository.createTicket(ticket)  → MockTicketRepository assigns TKT-XXX
     c. TicketEventBus.emit(TicketCreated(created))
     d. onSuccess → navigation popBackStack() → back to list
5. TicketListViewModel (already listening) receives TicketCreated
     a. insertAndSort() adds ticket to local Success state
     b. list re-sorted by Priority.sortOrder (HIGH first)
6. TicketListScreen shows new ticket — no loadTickets() call
```

### 2.7 Complete scenario: change priority and see reorder

```
1. User opens ticket detail from list
2. TicketDetailViewModel.loadTicket() → repository.getTicketById()
3. User taps "Change Priority" → selects HIGH (if priorityUpdateEnabled)
4. TicketDetailViewModel.updatePriority(HIGH)
     a. repository.updateTicketPriority()
     b. TicketEventBus.emit(PriorityUpdated)
5. User navigates back to list
6. TicketListViewModel already updated local list and re-sorted
7. Ticket appears higher in the list with red "High" badge
```

---

## 3. Application screens

| Screen | Route | Composable | ViewModel | Data / state source |
|---|---|---|---|---|
| Login | `login` | `LoginScreen` | `LoginViewModel` | Simulated auth (`admin` / `admin123`) |
| Ticket list | `ticket_list` | `TicketListScreen` | `TicketListViewModel` | `TicketRepository.getTickets()` + event bus |
| Ticket detail | `ticket_detail/{ticketId}` | `TicketDetailScreen` | `TicketDetailViewModel` | `getTicketById`, `updateStatus`, `updatePriority` |
| Create ticket | `create_ticket` | `CreateTicketScreen` | `CreateTicketViewModel` | `createTicket()` + emits `TicketCreated` |
| Settings | `settings` | `SettingsScreen` | None | `FeatureFlags` toggles |

Navigation is centralized in `AppNavigation.kt`. Routes are defined in `Screen.kt` to avoid magic strings.

---

## 4. Event-based communication

### 4.1 Problem

After creating or editing a ticket, the list screen must reflect changes **without** the user manually refreshing or re-fetching the full ticket list from the repository (which would be a full network round-trip in production).

### 4.2 Solution

**`TicketEvent.kt`** — sealed class defining the three domain events:

```kotlin
sealed class TicketEvent {
    data class TicketCreated(val ticket: Ticket) : TicketEvent()
    data class PriorityUpdated(val ticketId: String, val newPriority: Priority) : TicketEvent()
    data class StatusUpdated(val ticketId: String, val newStatus: TicketStatus) : TicketEvent()
}
```

**`TicketEventBus.kt`** — singleton bus:

```kotlin
object TicketEventBus {
    private val _events = MutableSharedFlow<TicketEvent>()
    val events: SharedFlow<TicketEvent> = _events.asSharedFlow()
    suspend fun emit(event: TicketEvent) = _events.emit(event)
}
```

### 4.3 Why SharedFlow

| Alternative | Reason not used |
|---|---|
| Call `loadTickets()` after every action | Extra repository/network call; not reactive |
| `LiveData` | Requires `LifecycleOwner`; not Kotlin-coroutine-native |
| Direct ViewModel references | Tight coupling between screens |
| **SharedFlow** ✓ | Lifecycle-independent, multiple collectors, fits coroutines |

`replay = 0` (default): late subscribers do not receive stale events.

### 4.4 Listener implementation

`TicketListViewModel` in `init`:

1. `loadTickets()` — initial fetch from repository.
2. `observeEvents()` — `TicketEventBus.events.collect { ... }` in `viewModelScope`.

Handlers: `insertAndSort`, `updateStatus`, `updatePriorityAndSort`.

---

## 5. Feature Flags

### 5.1 Purpose

Enable or disable features at runtime during internal testing **without** a new app release or code change.

### 5.2 Implementation

**File:** `core/featureflags/FeatureFlags.kt`

```kotlin
object FeatureFlags {
    var ticketCreationEnabled by mutableStateOf(true)
    var priorityUpdateEnabled by mutableStateOf(true)
}
```

Compose `mutableStateOf` triggers recomposition in any Composable that reads the property.

### 5.3 Active flags

| Flag | Default | UI when ON | UI when OFF |
|---|---|---|---|
| `ticketCreationEnabled` | `true` | FAB "+" on ticket list | FAB hidden; create screen unreachable from UI |
| `priorityUpdateEnabled` | `true` | "Change Priority" on detail | Button hidden; **Change Status** still works |

**Settings UI:** `SettingsScreen.kt` — toggles write directly to `FeatureFlags` (no ViewModel).

### 5.4 Future evolution

Load flag values from Firebase Remote Config or similar in `AppContainer` at startup. Composables and ViewModels stay unchanged.

---

## 6. Data layer and API alignment

The backend is not deployed. The mobile app is structured to consume `/contracts/tickets-api.yaml` via Retrofit when ready.

### 6.1 Endpoints

| Method | Path | Retrofit (`TicketApiService`) | Repository | PoC status |
|---|---|---|---|---|
| `POST` | `/auth/login` | `login()` | — | Retrofit ready; UI simulated |
| `GET` | `/tickets` | `getTickets()` | `getTickets()` | Active via mock |
| `POST` | `/tickets` | `createTicket()` | `createTicket()` | Active via mock |
| `GET` | `/tickets/{id}` | `getTicketById()` | `getTicketById()` | Active via mock |
| `PATCH` | `/tickets/{id}/status` | `updateTicketStatus()` | `updateTicketStatus()` | Active via mock |
| `PATCH` | `/tickets/{id}/priority` | `updateTicketPriority()` | `updateTicketPriority()` | Active via mock |

**Base URL:** `https://api.panini-support.com/v1` — defined in `NetworkClient.kt` and OpenAPI `servers`.

### 6.2 Domain ↔ API fields

| Domain model | JSON field | Wire values |
|---|---|---|
| `Ticket.id` | `id` | `"TKT-001"` |
| `Ticket.title` | `title` | string |
| `Ticket.description` | `description` | string |
| `Priority` | `priority` | `HIGH`, `MEDIUM`, `LOW` |
| `TicketStatus` | `status` | `OPEN`, `IN_PROGRESS`, `RESOLVED`, `CLOSED` |
| `TicketCategory` | `category` | `INVENTORY`, `DISTRIBUTION`, `LOGISTICS`, `SUPPLIER`, `QUALITY` |
| `Ticket.supplier` | `supplier` | string |
| `Ticket.createdAt` | `created_at` | ISO-8601 |
| `Ticket.assignedTo` | `assigned_to` | string, nullable |
| `Ticket.affectedQuantity` | `affected_quantity` | integer, nullable |

UI labels (`"High"`, `"Open"`, `"Inventory"`) are client-only on enum `label` properties. API and DTOs use enum **names**.

### 6.3 DTOs (Gson)

| OpenAPI schema | Kotlin DTO |
|---|---|
| `LoginRequest` | `LoginRequestDto` |
| `LoginResponse` / `User` | `LoginResponseDto` / `UserDto` |
| `Ticket` | `TicketDto` |
| `TicketListResponse` | `TicketListResponseDto` |
| `CreateTicketRequest` | `CreateTicketRequestDto` |
| `UpdateStatusRequest` | `UpdateStatusRequestDto` |
| `UpdatePriorityRequest` | `UpdatePriorityRequestDto` |

`RemoteTicketRepository` converts `TicketDto` → domain `Ticket` via `toDomain()`.

### 6.4 Switching mock → production

In `SupportApp.kt` → `AppContainer`:

```kotlin
val ticketRepository: TicketRepository = RemoteTicketRepository(NetworkClient.ticketApiService)
```

Next auth step: call `login()`, persist JWT, add Bearer interceptor in `NetworkClient`.

---

## 7. Mock data

**File:** `data/mock/MockData.kt` — **10 tickets** with realistic Panini/FIFA 2026 scenarios (inventory, logistics, quality, supplier issues). Example `TKT-001` aligns with the OpenAPI create-ticket example (Escazú shortage, Central Distributor S.A.).

**File:** `data/mock/MockTicketRepository.kt`

- Uses `delay()` (300–600 ms) so Loading states are visible in the UI.
- Mutations persist in memory for the app session (create, status/priority updates).
- `getTickets()` returns list sorted by `Priority.sortOrder` (HIGH = 0 first), matching contract description for `GET /tickets`.

---

## 8. UI and state handling

### 8.1 Ticket list card

Each card on the list shows:

1. **Title**
2. **Priority badge** — vivid colors: High red, Medium yellow, Low green
3. **Status badge** — Open red, In Progress blue, Resolved green, Closed gray
4. **Category**
5. **Supplier**
6. **Created date** (first 10 chars of ISO timestamp)

The detail screen shows all fields (id, description, assignee, affected units, actions).

### 8.2 State patterns

**`UiState<T>`** — used for async load on ticket list and ticket detail:

```kotlin
sealed class UiState<out T> {
    data object Loading : UiState<Nothing>()
    data class Success<T>(val data: T) : UiState<T>()
    data class Error(val message: String) : UiState<Nothing>()
}
```

| Screen | State type | Notes |
|---|---|---|
| Ticket list | `UiState<List<Ticket>>` | Loading / Success / Error + Retry |
| Ticket detail (load) | `UiState<Ticket>` | Loading / Success / Error |
| Ticket detail (actions) | `ActionState` | Idle / Loading / Error for status/priority updates |
| Create ticket | `CreateTicketUiState` | Form fields + isLoading + error |
| Login | `LoginUiState` | Form fields + isLoading + error |

Composables branch on state:

```kotlin
when (val state = uiState) {
    is UiState.Loading -> CircularProgressIndicator()
    is UiState.Error   -> Text(state.message)
    is UiState.Success -> /* render data */
}
```

---

## 9. Package map

Quick reference for `/app/src/main/java/com/panini/supportapp/`:

```
com.panini.supportapp
├── MainActivity.kt              Entry point; mounts Compose + passes repository
├── SupportApp.kt                Application + AppContainer (DI)
│
├── core/
│   ├── events/
│   │   ├── TicketEvent.kt       Sealed class: TicketCreated, StatusUpdated, PriorityUpdated
│   │   └── TicketEventBus.kt    SharedFlow emit/collect
│   └── featureflags/
│       └── FeatureFlags.kt      ticketCreationEnabled, priorityUpdateEnabled
│
├── domain/
│   ├── model/
│   │   ├── Ticket.kt            Core entity
│   │   ├── Priority.kt          HIGH, MEDIUM, LOW + sortOrder
│   │   ├── TicketStatus.kt      OPEN, IN_PROGRESS, RESOLVED, CLOSED
│   │   └── TicketCategory.kt    INVENTORY, DISTRIBUTION, etc.
│   └── repository/
│       └── TicketRepository.kt  Data contract (interface)
│
├── data/
│   ├── dto/                     Gson models matching OpenAPI
│   ├── mock/
│   │   ├── MockData.kt          10 seed tickets
│   │   └── MockTicketRepository.kt   Active repository (in-memory)
│   └── remote/
│       ├── TicketApiService.kt  Retrofit endpoints
│       ├── NetworkClient.kt     Retrofit + OkHttp singleton
│       └── RemoteTicketRepository.kt  Production repository (DTO → domain)
│
└── presentation/
    ├── common/UiState.kt        Loading / Success / Error
    ├── theme/Theme.kt           Material 3 colors
    ├── navigation/
    │   ├── Screen.kt            Route definitions
    │   └── AppNavigation.kt     NavHost wiring
    ├── login/                   LoginScreen + LoginViewModel
    ├── ticketlist/              TicketListScreen + TicketListViewModel (event listener)
    ├── ticketdetail/            TicketDetailScreen + TicketDetailViewModel (event emitter)
    ├── createticket/            CreateTicketScreen + CreateTicketViewModel (event emitter)
    └── settings/                SettingsScreen (flag toggles)
```

---

## 10. Extending the project

| Task | Steps |
|---|---|
| **New screen** | `Screen.kt` route → `AppNavigation.kt` composable → `*Screen.kt` + `*ViewModel.kt` |
| **New API endpoint** | `tickets-api.yaml` → `TicketApiService.kt` → DTO → `RemoteTicketRepository` + `MockTicketRepository` |
| **New feature flag** | Property in `FeatureFlags.kt` → read in Composable → toggle in `SettingsScreen.kt` |
| **Connect backend** | Swap `AppContainer` repository → JWT in `NetworkClient` |
| **Demo video** | Record 5–10 min handoff → upload → paste URL in `/video/demo-link.md` |

**Stack required:** Kotlin, Jetpack Compose, Coroutines, StateFlow/SharedFlow, MVVM + Repository, Retrofit + Gson.

**API single source of truth:** `/contracts/tickets-api.yaml`
