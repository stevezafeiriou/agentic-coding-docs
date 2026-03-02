# Performance, Memory, Watchdogs & Reliability

> This file defines performance budgets and reliability rules for production firmware.

---

## Core Objectives

- responsive UI and controls
- bounded loop latency
- no watchdog resets in normal operation
- stable long-running memory behavior
- graceful recovery from external failures

---

## Performance Budgets

These are default targets unless the product requires stricter numbers.

### Main loop

- average loop turnaround: **under 5 ms**
- no routine blocking calls above **50 ms** in normal operation
- network operations must be chunked, timed out, or delegated safely

### UI projects

- visible input latency should remain low enough to feel immediate
- avoid unnecessary full-screen invalidations
- batch related widget updates when practical
- redraw only when state changes

### Memory

- no recurring allocation spikes on steady-state paths
- avoid large temporary buffers in hot code
- large assets must be budgeted explicitly

### Storage

- settings writes must be infrequent and intentional
- telemetry buffering must have explicit size limits

---

## Watchdog Strategy

Espressif documents the ESP32 Task Watchdog Timer as a mechanism to detect tasks that run too long without yielding. citeturn622475search0

### Required policy

- **MUST** design code so watchdogs are not needed as a crutch for bad blocking architecture
- **MUST** avoid long unyielding work in `loop()` or ESP32 tasks
- **MUST** identify operations that can starve idle tasks or UI refresh

### Common causes

- long synchronous Wi-Fi or HTTP operations
- oversized display flush operations
- tight loops without yield opportunities
- excessive filesystem or JSON work in one chunk

### Recovery guidance

- log reset reason
- identify long-running subsystem
- split work across ticks
- reduce buffer sizes or flush granularity
- move truly necessary heavy work into carefully designed tasks on ESP32 only

---

## Memory Discipline

### Rules

- Prefer fixed-size buffers.
- Keep assets out of RAM where possible.
- Measure free heap at meaningful checkpoints.
- Avoid repeated allocation/deallocation patterns in steady state.
- Use `F()` / flash-resident strings where it materially helps and remains maintainable.

### Red flags

- repeated `String` concatenation in telemetry loop
- parsing large JSON documents every second
- full-screen image assets embedded without flash budget review
- creating/destroying UI objects repeatedly when visibility toggling would suffice

---

## Reliability Patterns

### Network resilience

- backoff reconnect attempts
- do not block local control/UI while reconnecting
- separate “network disconnected” from “device failed”

### Peripheral resilience

- device continues if optional sensor missing
- fault counters tracked
- stale data marked stale, not treated as current

### Storage resilience

- invalid config triggers defaults or safe mode
- filesystem failure does not crash unrelated features
- write frequency bounded

### Boot resilience

- safe GPIO defaults set before services start
- invalid settings cannot cause infinite reboot loop
- reset reason visible for diagnostics

---

## SquareLine / LVGL Performance Guidance

- Study generated widget tree before changing behavior.
- Avoid updating labels, images, and styles every tick if values did not change.
- Use dirty flags or last-value caching.
- Verify display buffer size and flush strategy.
- Budget memory for fonts and images explicitly.
- If assets live on filesystem or SD, validate LVGL filesystem bridge and access latency.

SquareLine exports platform-independent LVGL code, so runtime performance depends on the board integration, buffer sizing, driver quality, and asset strategy rather than the design tool alone. citeturn857924search4

---

## Profiling Checklist

- [ ] measure boot time to ready state
- [ ] log free heap after boot
- [ ] log free heap after entering main UI
- [ ] log free heap after repeated navigation
- [ ] log reconnect attempts and intervals
- [ ] count sensor read failures
- [ ] verify no watchdog resets during stress run

---

## Soak Test Checklist

Run for extended duration on target hardware:

- [ ] stable power source test
- [ ] repeated network loss and recovery
- [ ] repeated UI navigation or command use
- [ ] periodic settings changes if supported
- [ ] asset-heavy UI screens opened repeatedly
- [ ] relay or actuator transitions repeated safely

Record:
- heap trend
- reset count
- reconnect count
- error counters
- user-visible stalls

---

## Failure Modes

- watchdog resets under display/network load
- heap fragmentation after hours of UI navigation
- reconnect storm starving UI
- full-screen redraw on every sensor update
- settings writes too frequent and causing latency spikes
- LVGL asset loading stalls from filesystem or SD access

---

## Common Pitfalls

- using `delay(1000)` in main loop for periodic work
- assuming more FreeRTOS tasks automatically improve performance
- over-allocating JSON docs and image buffers
- calling UI updates from too many code paths without change detection
