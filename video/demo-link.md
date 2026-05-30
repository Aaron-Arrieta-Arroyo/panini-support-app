# Video Demo — Panini Support App

## Link

[Video Demo](https://drive.google.com/file/d/1XlL8xWzGbXYoXkiYBlcY4_fqs0GieJkV/view?usp=sharing)

Direct URL: `https://drive.google.com/file/d/1XlL8xWzGbXYoXkiYBlcY4_fqs0GieJkV/view?usp=sharing`

## Access

Grant viewing access to project reviewers as required (course instructor and evaluators).

**Instructor email:** rachel.bolivar.morales@una.cr

---

## App flow screenshots

Screenshots complement the video demo and show the main user flow when live recording was not possible for every screen.

| Step | Screen | Screenshot |
|---|---|---|
| 1 | Login → Ticket list (hub) | [01-ticket-list.png](screenshots/01-ticket-list.png) |
| 2 | Create ticket form | [02-create-ticket.png](screenshots/02-create-ticket.png) |
| 3 | Settings / Feature Flags | [03-settings-feature-flags.png](screenshots/03-settings-feature-flags.png) |

### Flow covered

1. **Login** (`admin` / `admin123`) → ticket list loads with mock data sorted by priority.
2. **Ticket list** — title, priority badge, status badge, category, supplier, date. FAB (+) opens create screen when `ticketCreationEnabled` is ON.
3. **Create ticket** — form with priority/category chips; on submit, list updates via `TicketEventBus` (no manual refresh).
4. **Ticket detail** — full fields; Change Status / Change Priority (when flag enabled).
5. **Settings** — toggle ticket creation and priority updates at runtime.

---

## Contents (5–10 min)

1. Project overview and repository structure
2. Architecture walkthrough (MVVM + layers)
3. Live demo: Login → Ticket List → Detail → Create → Status/Priority update
4. Event-based communication explanation (SharedFlow flow)
5. Feature Flags explanation and live toggle
6. Networking layer readiness for backend integration
7. How other engineers can continue the project

## Demo credentials

| Field | Value |
|---|---|
| Username | `admin` |
| Password | `admin123` |
