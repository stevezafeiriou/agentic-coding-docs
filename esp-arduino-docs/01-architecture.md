# Firmware Architecture, Boundaries & Project Layout

> This file defines the required architecture for production-grade Arduino IDE firmware targeting ESP32/ESP8266.

---

## ASSUMPTIONS

- The repository may be either:
  - a classic Arduino sketch folder, or
  - a structured sketch with `src/`, `lib/`, and generated UI folders.
- SquareLine Studio may generate LVGL UI code and assets into the repo.
- The same repo may support multiple board profiles.

---

## Architectural Rules

**MUST separate generated code from handwritten code.**  
Generated SquareLine/LVGL artifacts must live in a dedicated folder and be treated as read-only.

**MUST separate hardware from application logic.**  
Code that talks to GPIO, I2C, SPI, UART, ADC, display, touch, or network belongs behind driver or service interfaces.

**MUST centralize configuration.**  
Board pin maps, feature flags, credentials placeholders, and production tuning values must not be scattered.

**MUST design by module responsibility.**  
A module owns one thing. For example:
- display rendering
- sensor acquisition
- Wi-Fi management
- device settings
- actuator control
- telemetry publishing

**MUST support safe startup sequencing.**  
Initialization order must be explicit and logged.

**MUST support partial functionality.**  
The device must run in degraded mode if optional subsystems fail.

---

## Required Folder Layout

Use this structure unless the existing repo has a strong reason not to.

```text
MyDevice/
├── MyDevice.ino                     # Thin entrypoint only
├── README.md
├── arduino_secrets.h.example        # Placeholder secrets only
├── docs/
│   ├── wiring.md
│   ├── flashing.md
│   └── operations.md
├── include/
│   ├── app_config.h                 # Build-wide constants and feature flags
│   ├── board_profile.h              # Active board selection
│   ├── version.h                    # Firmware metadata
│   └── log.h                        # Logging macros
├── src/
│   ├── app/
│   │   ├── app_controller.h
│   │   ├── app_controller.cpp
│   │   ├── boot_sequence.h
│   │   └── boot_sequence.cpp
│   ├── board/
│   │   ├── board_config.h
│   │   ├── board_esp32_xxx.h
│   │   └── board_esp8266_xxx.h
│   ├── drivers/
│   │   ├── display_driver.h
│   │   ├── touch_driver.h
│   │   ├── sensor_xxx.h
│   │   ├── relay_driver.h
│   │   └── storage_driver.h
│   ├── services/
│   │   ├── wifi_service.h
│   │   ├── ota_service.h
│   │   ├── time_service.h
│   │   ├── telemetry_service.h
│   │   ├── settings_service.h
│   │   └── health_service.h
│   ├── ui/
│   │   ├── ui_controller.h
│   │   ├── ui_controller.cpp
│   │   ├── ui_bindings.h
│   │   └── ui_bindings.cpp
│   ├── state/
│   │   ├── app_state.h
│   │   └── app_state.cpp
│   └── util/
│       ├── scheduler.h
│       ├── scheduler.cpp
│       ├── safe_string.h
│       ├── retry_policy.h
│       └── reset_reason.h
├── generated/                       # Read-only generated artifacts
│   └── squareline_ui/
│       ├── ui.h
│       ├── ui.c
│       ├── ui_events.c
│       ├── ui_helpers.c
│       ├── screens/
│       ├── components/
│       └── assets/
├── data/                            # LittleFS or web assets if applicable
├── test/                            # Host-testable logic and test harness notes
└── tools/                           # Helper scripts, export checks, asset tools
```

---

## Entry Point Requirements

`MyDevice.ino` must be thin.

```cpp
#include "include/app_config.h"
#include "include/log.h"
#include "src/app/boot_sequence.h"
#include "src/app/app_controller.h"

static AppController app;

void setup() {
  BootSequence::run(app);
}

void loop() {
  app.tick();
}
```

### Rules

- `setup()` must not contain business logic.
- `loop()` must call a controller tick and return quickly.
- Direct peripheral handling in `.ino` is forbidden.

---

## Module Boundaries

### 1. Board Layer

Owns:
- pin maps
- bus assignments
- board capabilities
- active-low/active-high semantics
- hardware presence flags
- display geometry
- PSRAM capability flags

Must not own:
- business workflows
- network policy
- UI decisions

Template:

```cpp
#pragma once

struct BoardConfig {
  const char* boardName;
  int pinI2CSda;
  int pinI2CScl;
  int pinRelay1;
  bool relayActiveLow;
  bool hasDisplay;
  bool hasTouch;
  bool hasPSRAM;
};

const BoardConfig& getBoardConfig();
```

---

### 2. Driver Layer

Owns:
- low-level hardware interaction
- read/write primitives
- hardware initialization
- device presence detection
- raw error codes

Must not own:
- retry policy for business behavior
- application rules
- UI rules

Example: sensor driver API

```cpp
struct SensorReading {
  bool valid;
  float temperatureC;
  float humidityPct;
  uint32_t timestampMs;
};

class TempHumidityDriver {
public:
  bool begin();
  SensorReading read();
};
```

---

### 3. Service Layer

Owns:
- orchestration around a capability
- retries/backoff
- persistence access
- command validation
- higher-level domain semantics

Examples:
- Wi-Fi connection management
- settings persistence
- OTA lifecycle
- telemetry batching
- time sync status

Must not:
- directly render LVGL widgets
- hardcode board pins

---

### 4. UI Integration Layer

This layer exists only when a display/UI is present.

Owns:
- mapping state into widgets
- reacting to generated event callbacks
- screen transitions
- throttled redraw strategy

Must not:
- embed sensor/network logic directly inside generated callbacks
- manually edit generated widget construction unless absolutely necessary

Pattern:

```cpp
// ui_bindings.cpp
#include "generated/squareline_ui/ui.h"
#include "src/ui/ui_controller.h"

extern UiController g_ui;

void onButtonSavePressed(lv_event_t* e) {
  g_ui.handleSavePressed();
}
```

---

### 5. Application Layer

Owns:
- use cases
- feature coordination
- device mode transitions
- high-level state machine
- periodic scheduling registration

Recommended pattern:
- `AppController::begin()`
- `AppController::tick()`
- `AppController::handleEvent(...)`

---

## Boot Sequence

Boot order must be explicit and logged.

### Required order

1. Minimal serial/log init
2. Reset reason capture
3. Board config resolve
4. Safe GPIO default states
5. Persistence mount/init
6. Settings load and validate
7. Display/touch init if present
8. UI init if present
9. Network init
10. OTA service init
11. Sensors/actuators init
12. App state hydrate
13. Start scheduler and health timers
14. Enter normal operation

### Boot requirements

- Each step must return success/failure.
- Optional subsystem failures must be logged and downgraded, not crash the system.
- Fatal failures must lead to safe mode, not undefined behavior.

Template:

```cpp
bool BootSequence::run(AppController& app) {
  Log::begin();
  Log::info("boot", "firmware=%s version=%s", FW_NAME, FW_VERSION);

  if (!StorageDriver::begin()) {
    Log::error("boot", "storage init failed");
    app.enterSafeMode();
    return false;
  }

  if (!SettingsService::load()) {
    Log::warn("boot", "settings invalid, using defaults");
    SettingsService::resetToDefaults();
  }

  app.begin();
  return true;
}
```

---

## Scheduler Architecture

### Default rule

Use cooperative scheduling based on `millis()` first.  
Do **not** create FreeRTOS tasks on ESP32 unless there is a demonstrated need.

### Recommended scheduler responsibilities

- register recurring tasks
- register one-shot deferred actions
- enforce intervals
- avoid drift where needed
- expose execution time metrics

Example:

```cpp
struct ScheduledTask {
  uint32_t intervalMs;
  uint32_t lastRunMs;
  void (*callback)();
};
```

### Suitable for cooperative scheduling

- sensor polling
- status LED updates
- telemetry flush checks
- Wi-Fi health checks
- UI refresh throttles
- button polling/debounce updates

### Suitable for interrupt or special handling

- pulse counting
- precise edge capture
- time-critical drivers
- touch interrupt wake notification

### ESP32 task guidance

Create a dedicated task only if:
- display flush or networking causes measurable starvation, or
- vendor library already expects task separation, or
- the feature cannot be made cooperative safely

If using FreeRTOS tasks:
- pin ownership and shared state rules must be documented
- queues or guarded state access must be explicit
- task stack sizing must be measured, not guessed

---

## State Management

### State categories

1. **Static configuration**
   - compile-time flags
   - board definitions

2. **Persisted settings**
   - Wi-Fi credentials
   - calibration
   - UI preferences
   - API endpoints

3. **Runtime state**
   - network connected
   - current sensor reading
   - active screen
   - health alarms

4. **Derived state**
   - is device ready
   - is telemetry allowed
   - is control action permitted

### Rules

- Runtime state belongs in one central app state model.
- UI reads derived state; it does not invent state.
- Persisted settings must be validated before use.

---

## SquareLine Studio Integration Architecture

If SquareLine assets exist, the AI agent must preserve this separation:

### Generated folder

Contains:
- screens
- widgets
- theme
- fonts
- images
- generated event function declarations

### Handwritten folder

Contains:
- business callbacks
- adapter logic
- screen update helpers
- state-to-widget mapping
- custom widget utility wrappers if needed

### Required policy

- **MUST NOT** write business logic in generated `.c/.h` files.
- **MUST** re-check generated callback signatures after every SquareLine export.
- **MUST** document which files may be overwritten by re-export.
- **MUST** keep a regen-safe integration boundary.

### SquareLine study checklist

When generated files exist, inspect and record:
- LVGL major version
- display resolution
- color depth assumptions
- `lv_conf.h` expectations
- screen names
- widget IDs used for update binding
- custom fonts and assets
- navigation flow
- event callback names and where custom logic should connect

---

## Interface Contracts

Every module must document:

- what it owns
- how it is initialized
- what errors it can return
- whether it is optional
- whether it is safe to call before network/time/UI is ready
- whether it can allocate memory dynamically
- whether it touches hardware directly

Example contract block:

```cpp
// WifiService
// Owns station-mode connectivity and reconnect strategy.
// Requires SettingsService loaded.
// Safe to call begin() without valid credentials; enters idle-disconnected state.
// Does not block longer than configured connect timeout slice per tick.
```

---

## Failure Modes

- Generated UI overwrites custom callbacks
- Shared global state mutated from multiple contexts without discipline
- `setup()` becomes monolithic and impossible to recover safely
- board-specific hacks leak into generic modules
- display driver assumptions fail on alternate board
- network logic blocks boot
- file paths or asset references break after SquareLine re-export

---

## Common Pitfalls

- Putting all code in one `.ino`
- Directly touching LVGL widgets from random modules
- Hardcoding Wi-Fi retry loops in many places
- Duplicating sensor logic in UI and telemetry layers
- Building a multi-board repo without explicit board profiles
- Treating generated files as stable custom code
