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
   git clone <repo-url>
   cd PaniniSupportApp
   ```

2. Open the `app/` folder in Android Studio as an Android project (or open the root and let Gradle sync).

3. If the Gradle wrapper is missing, generate it:
   ```bash
   gradle wrapper
   ```

4. Run on an emulator or device (API 26+).

5. Login credentials: **admin / admin123**

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
│   ├── events/         TicketEventBus (SharedFlow)
│   └── featureflags/   FeatureFlags (centralized registry)
├── data/
│   ├── dto/            API DTOs (Gson-annotated)
│   ├── mock/           MockData + MockTicketRepository
│   └── remote/         Retrofit service + NetworkClient + RemoteTicketRepository
├── domain/
│   ├── model/          Ticket, Priority, TicketStatus, TicketCategory
│   └── repository/     TicketRepository interface
└── presentation/
    ├── common/         UiState<T>
    ├── navigation/     NavHost, Screen
    ├── theme/          Material 3 color scheme
    ├── login/          LoginScreen + LoginViewModel
    ├── ticketlist/     TicketListScreen + TicketListViewModel
    ├── ticketdetail/   TicketDetailScreen + TicketDetailViewModel
    └── createticket/   CreateTicketScreen + CreateTicketViewModel
```

---

## Key Technical Decisions

### Event-based communication (SharedFlow)

`TicketEventBus` is a singleton `SharedFlow` that broadcasts domain events across the app.  
When a ticket is created or its priority changes, `TicketListViewModel` reacts automatically — no manual screen refresh.

```
CreateTicketViewModel → TicketEventBus.emit(TicketCreated) → TicketListViewModel reacts
TicketDetailViewModel → TicketEventBus.emit(PriorityUpdated) → list re-sorts automatically
```

### Feature Flags

`FeatureFlags` object controls four behaviors:

| Flag | Effect |
|---|---|
| `ticketCreationEnabled` | Shows/hides the Create Ticket FAB |
| `priorityUpdateEnabled` | Shows/hides the "Cambiar Prioridad" button |
| `adminFeaturesEnabled` | Reserved for admin panel (Phase 2) |
| `logisticsCategoryVisible` | Includes/excludes LOGISTICS in creation form |

### Switching to real backend

Change **one line** in `AppContainer.kt`:

```kotlin
// Before (mock):
val ticketRepository: TicketRepository = MockTicketRepository()

// After (production):
val ticketRepository: TicketRepository = RemoteTicketRepository(NetworkClient.ticketApiService)
```

No ViewModel or screen changes required.

---

## API Contracts

See `/contracts/tickets-api.yaml` — OpenAPI 3.0 specification covering all endpoints:  
`POST /auth/login`, `GET/POST /tickets`, `GET /tickets/{id}`, `PATCH /tickets/{id}/status`, `PATCH /tickets/{id}/priority`

---

## Notes for the Team

- All mock data represents realistic Panini/FIFA 2026 operational scenarios (no generic Lorem ipsum data).
- Architecture is intentionally simple: MVVM + Repository without use cases, since scope does not justify the overhead. Use cases can be added as an intermediate layer without touching ViewModels or screens.
- `docs/architecture.md` contains full rationale for every technical decision.
