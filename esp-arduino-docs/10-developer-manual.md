# Developer Manual for Non-Embedded Developers

> This file explains how to work with an Arduino IDE ESP32/ESP8266 project that follows this doc bundle.

---

## What This Project Structure Means

You are not expected to put all code in one `.ino` file.

The project is intentionally split so each concern lives in one place:

- `.ino` = startup entrypoint only
- `board/` = pin mappings and board differences
- `drivers/` = raw hardware access
- `services/` = Wi-Fi, OTA, storage, settings, telemetry
- `ui/` = handwritten display logic and event handling
- `generated/` = SquareLine-generated UI code that should usually not be edited manually
- `docs/` = operational notes like flashing, wiring, and troubleshooting

This makes firmware easier to maintain and safer to change.

---

## How to Think About the Device

A connected embedded device is not like a normal web app.

It must handle:

- limited RAM and flash
- intermittent Wi-Fi
- power loss
- corrupted settings
- missing hardware
- watchdog resets
- long-lived uptime

That is why these docs insist on:
- small modules
- no long blocking loops
- explicit error handling
- safe boot defaults
- strong separation between generated UI and custom logic

---

## ESP32 vs ESP8266

Both use Arduino-style code, but they are not interchangeable.

### ESP32

Common strengths:
- more RAM than ESP8266
- dual-core architecture
- Preferences/NVS support
- better fit for display-heavy and UI-heavy projects
- stronger option set for OTA, networking, and richer peripherals

### ESP8266

Common constraints:
- tighter memory budget
- more care needed with heap and feature count
- still useful for simpler Wi-Fi devices, sensors, switches, or compact control nodes

When supporting both in one repo, the AI agent must isolate differences behind board and service abstractions.

---

## Arduino IDE Workflow

### Typical development cycle

1. Open the sketch folder in Arduino IDE
2. Select the correct board
3. Install required libraries and board packages
4. Add local secrets in a non-committed secrets header if needed
5. Compile
6. Flash via USB
7. Open Serial Monitor
8. Validate boot logs and behavior
9. Iterate
10. Only then test OTA if used

---

## What to Do Before Changing Code

1. Read `00-principles.md`
2. Read `01-architecture.md`
3. Check whether the repo contains SquareLine or LVGL generated files
4. Check board profile and pin map
5. Understand whether the feature touches:
   - hardware
   - services
   - UI
   - persisted settings
   - networking

If you skip this, you risk breaking generated UI integration or board assumptions.

---

## Understanding SquareLine Studio in This Repo

SquareLine Studio can generate LVGL UI code for embedded displays. SquareLine itself is not the runtime UI engine; LVGL is. SquareLine exports generated code and assets that the firmware compiles and runs. citeturn857924search4

### Important rule

Treat generated SquareLine output as **generated code**, not handwritten application code.

That means:
- do not put business logic directly into generated files
- expect future exports to overwrite generated files
- keep custom event handling in wrapper or binding files

### What you should inspect if SquareLine exists

Look for files like:
- `ui.h`
- `ui.c`
- `ui_events.c`
- `ui_helpers.c`
- `screens/`
- `components/`
- `assets/`

These files tell you:
- screen names
- button/widget IDs
- callback hooks
- fonts/images/assets
- how the generated UI expects to be initialized

### Re-export warning

SquareLine export workflows for Arduino/LVGL projects can be sensitive to folder structure and export mode. Preserve the documented export flow used by the repo and avoid ad hoc copying. citeturn857924search16turn857924search2

---

## How Settings and Secrets Work

### Secrets

Never hardcode real credentials in the source repository.

You will usually see:
- `arduino_secrets.h.example` committed
- `arduino_secrets.h` ignored locally

### Persisted settings

Some settings may be stored on the device:
- Wi-Fi credentials
- API host
- calibration values
- UI preferences

Those are validated at boot. If invalid, the firmware should fall back to defaults or safe mode.

---

## How Boot Works

A good embedded project does not “just start doing things.”

A safe boot flow is usually:

1. serial logging starts
2. board config loads
3. storage initializes
4. settings load and validate
5. display/UI starts if present
6. Wi-Fi and services start
7. app enters normal operation

If something fails, the device should degrade gracefully rather than hang forever.

---

## How to Add a New Feature Safely

### Example: add a new temperature sensor

You should expect the AI agent to:

1. add or update a driver in `src/drivers/`
2. add a service in `src/services/`
3. update state in `src/state/`
4. update UI bindings in `src/ui/` if needed
5. avoid editing generated SquareLine files unless export changes are required
6. add validation and logs
7. add a manual hardware test checklist

### Example: add a new UI button in a SquareLine project

You should expect:

1. the visual/button itself may need to be created in SquareLine
2. SquareLine export updates generated files
3. custom button behavior is attached in handwritten binding/controller code
4. docs are updated to explain the new screen flow

---

## How to Review AI-Generated Firmware Changes

When the AI gives you code, review these questions:

- Did it keep `.ino` thin?
- Did it create or reuse proper modules?
- Did it avoid putting logic in generated SquareLine files?
- Did it define error behavior?
- Did it define invalid-config behavior?
- Did it add test and validation steps?
- Did it mention performance or memory impact?
- Did it keep secrets out of source?

If the answer is “no” to several of these, do not accept the change yet.

---

## How to Flash and Test

### USB flashing

Typical process:
1. connect the board by USB
2. select correct board and port in Arduino IDE
3. compile
4. upload
5. open Serial Monitor
6. verify:
   - version log
   - board profile log
   - no boot loops
   - expected subsystem init logs

### First boot checklist

- does it boot without crashing?
- does it connect to Wi-Fi if expected?
- does UI render?
- do sensors report?
- do outputs start in safe state?
- do logs show version and reset reason?

---

## How OTA Fits In

For ESP32, production OTA is best designed around a partition table with two OTA app slots plus OTA metadata, so one image can be updated while the other remains the bootable fallback. citeturn857924search3turn857924search17

### Practical meaning

If your product uses OTA:
- the firmware layout must support it
- the binary must fit
- update auth matters
- you need a recovery plan if update fails

Do not assume OTA is “just a library checkbox.”

---

## How to Troubleshoot Common Problems

### Problem: device boot loops

Check:
- serial logs
- reset reason
- settings validation
- recent persistence schema changes
- whether a new peripheral init hangs

### Problem: UI compiles but buttons do nothing

Check:
- whether callbacks are actually wired
- whether SquareLine export changed callback names
- whether custom logic was put in the wrong file
- whether the touch/display driver works independently

### Problem: UI is slow or freezes

Check:
- repeated full-screen redraws
- network reconnect storm
- heavy filesystem access during redraw
- low free heap
- watchdog reset clues

### Problem: settings do not persist

Check:
- whether writes are actually called
- whether schema changed
- whether validation rejects loaded values
- whether the selected storage backend is initialized properly

---

## What “Production Grade” Means Here

In this repo, “production grade” means the firmware is expected to be:

- maintainable
- testable
- observable
- secure by default
- resilient to bad input and bad connectivity
- safe on boot and after reset
- explicit about assumptions and limitations

It does **not** mean “largest feature set.”
It means the firmware can be understood, supported, and shipped responsibly.

---

## Next Steps for a Developer Using This Bundle

1. read `00-principles.md`
2. identify your actual board and peripherals
3. confirm whether SquareLine is used
4. create or verify board profile docs
5. lock library and toolchain versions
6. ask the AI agent to generate code only after it reports its repo scan and assumptions
7. review every generated change against the checklists in this bundle
