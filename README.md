# Panini Support App — PoC

**Context:** Support ticket management system for Panini FIFA World Cup 2026 album operations.

---

## Description

Mobile proof-of-concept for centralizing internal support requests related to supplier incidents, package distribution, inventory shortages, and logistics issues across Panini's point-of-sale network.

The app replaces the current email/spreadsheet workflow with a structured ticket lifecycle: creation → status tracking → resolution.

---

## Running the Project

### Requirements

- Android Studio Hedgehog (2023.1.1) or later
- JDK 17
- Android SDK 35

### Steps

1. Clone the repository:
   ```bash
   git clone https://github.com/Aaron-Arrieta-Arroyo/Panini-Support-App.git
   cd Panini-Support-App
   ```

2. Open the project root in Android Studio and let Gradle sync.

3. Run on an emulator or device (API 26+).

4. Login credentials: **admin / admin123** (simulated locally; API login is prepared but not wired yet).

---

## Tech Stack

| Technology | Purpose |
|---|---|
| Jetpack Compose | Declarative UI |
| Material 3 | Design system |
| Navigation Compose | Screen routing |
| ViewModel + StateFlow | MVVM state management |
| SharedFlow | Event-based communication |
| Retrofit + OkHttp | Networking layer (ready for backend) |
| Gson | JSON serialization |
| Kotlin Coroutines | Async operations |

---

## Project Structure

```
/app                Android project (Jetpack Compose, MVVM)
/contracts          API contracts in OpenAPI 3.0 YAML format
/docs               Technical documentation and architecture decisions
/video              Video demo link
README.md           This file
```

### Package structure inside `/app`

```
com.panini.supportapp
├── core/
│   ├── events/         TicketEventBus + TicketEvent (SharedFlow)
│   └── featureflags/   FeatureFlags (centralized registry)
├── data/
│   ├── dto/            API DTOs aligned with /contracts/tickets-api.yaml
│   ├── mock/           MockData + MockTicketRepository
│   └── remote/         TicketApiService + NetworkClient + RemoteTicketRepository
├── domain/
│   ├── model/          Ticket, Priority, TicketStatus, TicketCategory
│   └── repository/     TicketRepository interface
└── presentation/
    ├── common/         UiState<T>
    ├── navigation/     NavHost, Screen
    ├── theme/          Material 3 color scheme
    ├── login/          LoginScreen + LoginViewModel (simulated auth)
    ├── ticketlist/     TicketListScreen + TicketListViewModel
    ├── ticketdetail/   TicketDetailScreen + TicketDetailViewModel
    ├── createticket/   CreateTicketScreen + CreateTicketViewModel
    └── settings/       SettingsScreen (Feature Flag toggles)
```

---

## Key Technical Decisions

### Event-based communication (SharedFlow)

`TicketEventBus` broadcasts domain events. `TicketListViewModel` listens and updates the list without refetching:

```
CreateTicketViewModel  → TicketCreated   → list inserts + re-sorts by priority
TicketDetailViewModel  → StatusUpdated   → list updates status badge
TicketDetailViewModel  → PriorityUpdated → list re-sorts by priority
```

### Ticket list UI

Each list card shows **title**, **priority badge**, **status badge**, **category**, **supplier**, and **date**. Badges use vivid colors (priority: red / yellow / green; status: red / blue / green / gray). Full ticket data appears on the detail screen.

### Feature Flags

Two flags are controlled from **Settings** (gear icon on the ticket list):

| Flag | Effect |
|---|---|
| `ticketCreationEnabled` | Shows/hides the Create Ticket FAB |
| `priorityUpdateEnabled` | Shows/hides the **Change Priority** button in ticket detail |

Flags use Compose `mutableStateOf`, so toggling them in Settings updates the UI immediately.

### Switching to real backend

Change **one line** in `SupportApp.kt` (`AppContainer`):

```kotlin
// Before (mock):
val ticketRepository: TicketRepository = MockTicketRepository()

// After (production):
val ticketRepository: TicketRepository = RemoteTicketRepository(NetworkClient.ticketApiService)
```

No ViewModel or screen changes required. Wire JWT auth via `NetworkClient` when connecting `POST /auth/login`.

---

## API Contracts

See `/contracts/tickets-api.yaml` — OpenAPI 3.0 specification aligned with `TicketApiService` and DTOs:

| Endpoint | App usage today |
|---|---|
| `POST /auth/login` | Prepared in Retrofit; login screen uses local simulation |
| `GET /tickets` | `TicketRepository.getTickets()` |
| `POST /tickets` | `TicketRepository.createTicket()` |
| `GET /tickets/{id}` | `TicketRepository.getTicketById()` |
| `PATCH /tickets/{id}/status` | `TicketRepository.updateTicketStatus()` |
| `PATCH /tickets/{id}/priority` | `TicketRepository.updateTicketPriority()` |

Domain enums (`Priority`, `TicketStatus`, `TicketCategory`) match API enum values exactly. UI display labels (`High`, `Open`, etc.) are client-side only.

---

## Notes for the Team

- All mock data represents realistic Panini/FIFA 2026 operational scenarios (no generic placeholder text).
- Architecture is intentionally simple: MVVM + Repository without use cases.
- `docs/architecture.md` contains full rationale: architecture, **general system flow**, events, feature flags, API alignment, and package map.
