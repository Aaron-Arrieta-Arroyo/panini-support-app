# Installation Guide — Panini Support App

Direct steps to clone, build, and run the project on your machine.

---

## Requirements

| Tool | Version |
|---|---|
| Android Studio | Hedgehog (2023.1.1) or newer |
| JDK | 17 |
| Android SDK | 35 |
| Min device/emulator API | 26 |

---

## 1. Clone the repository

```bash
git clone https://github.com/Aaron-Arrieta-Arroyo/Panini-Support-App.git
cd Panini-Support-App
```

Use the **`main`** branch (default) for the full project.

---

## 2. Open in Android Studio

1. Launch **Android Studio**.
2. **File → Open** and select the **project root** folder (the folder that contains `app/`, `build.gradle.kts`, and `settings.gradle.kts`).
3. Wait for **Gradle Sync** to finish. First sync may take several minutes while dependencies download.

If Android Studio asks to install missing SDK components, accept and install **SDK 35** and **Build-Tools**.

---

## 3. Configure an emulator (if you don't have a physical device)

1. **Tools → Device Manager**.
2. **Create Device** → pick a phone (e.g. Pixel 6).
3. Select a system image with **API 26+** (API 34 or 35 recommended).
4. Finish and start the emulator.

---

## 4. Run the app

1. In the toolbar, select the **`app`** run configuration.
2. Select your emulator or connected device.
3. Click **Run** (green play button) or press `Shift+F10`.

The app installs and opens on the **Login** screen.

---

## 5. Login credentials

| Field | Value |
|---|---|
| Username | `admin` |
| Password | `admin123` |

Authentication is **simulated locally** (no backend required for this PoC).

---

## 6. Quick functional check

After login you should be able to:

- See a list of **10 mock tickets** sorted by priority.
- Open a ticket **detail** screen.
- **Change status** and **change priority** (if enabled in Settings).
- **Create** a new ticket with the **+** FAB (if enabled in Settings).
- Toggle **Feature Flags** from the **gear icon** on the ticket list.

---

## Build from terminal (optional)

```bash
./gradlew assembleDebug
```

APK output: `app/build/outputs/apk/debug/app-debug.apk`

---

## Troubleshooting

| Problem | Solution |
|---|---|
| Gradle sync fails | Check internet; **File → Invalidate Caches → Restart** |
| SDK not found | **Settings → Languages & Frameworks → Android SDK** → install SDK 35 |
| JDK error | **Settings → Build → Gradle → Gradle JDK** → select JDK 17 |
| Emulator too slow | Enable hardware acceleration (KVM/HAXM) or use a physical device |

---

## Project docs

| Resource | Location |
|---|---|
| Architecture & technical decisions | `/docs/architecture.md` |
| API contracts (OpenAPI YAML) | `/contracts/tickets-api.yaml` |
| Demo video | `/video/demo-link.md` |
| Full project overview | `/README.md` |
