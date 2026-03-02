# Logging, Diagnostics, Runbooks & Field Operations

> This file defines how production firmware must be observable and supportable.

---

## Required Diagnostics

Every production build must expose, through serial logs, UI, local API, or telemetry where applicable:

- firmware version
- board profile
- boot timestamp or uptime start
- reset reason
- Wi-Fi connected/disconnected state
- configuration validity state
- optional subsystem readiness
- last sensor update time
- last OTA result if OTA is enabled

---

## Logging Policy

### Log classes

- `BOOT`
- `NET`
- `UI`
- `SENSOR`
- `STORAGE`
- `OTA`
- `SECURITY`
- `HEALTH`

### Rules

- logs must be short and useful
- every failure log should include context
- repeated identical logs should be rate-limited
- production log level must be controllable at compile time

Example:

```cpp
Log::info("BOOT", "board=%s version=%s", FW_BOARD_PROFILE, FW_VERSION);
Log::warn("NET", "wifi disconnected, retry in %lu ms", retryDelay);
Log::error("STORAGE", "settings validation failed, using defaults");
```

---

## Health Signals

Recommended counters:

- boot count
- failed boot count
- wifi reconnect count
- sensor error count
- ota success count
- ota failure count
- factory reset count
- last settings migration result

Recommended gauges/state:
- free heap
- RSSI
- uptime
- current mode
- safe mode enabled
- current screen ID if UI-heavy diagnostics are needed

---

## Runbooks

Every repo should include basic operational runbooks.

### Runbook: Device boot loop

1. Capture serial logs immediately on boot
2. Check reset reason
3. Determine whether storage or settings validation failed
4. Enter factory reset path if documented and safe
5. Reflash known-good build if recovery fails

### Runbook: Device not connecting to Wi-Fi

1. Verify SSID and password presence
2. Verify provisioning mode rules
3. Check RSSI or AP availability
4. Confirm reconnect backoff is occurring, not a tight loop
5. Confirm device remains locally responsive while disconnected

### Runbook: UI frozen or laggy

1. Check watchdog/reset logs
2. Check whether network retries are monopolizing runtime
3. Inspect repeated LVGL redraw/update calls
4. Measure free heap
5. Confirm display/touch driver versions and SquareLine export compatibility

### Runbook: OTA failed

1. Confirm current firmware version and target version
2. Confirm partition capacity
3. Confirm auth and transport path
4. Inspect failure stage: download, validate, write, reboot
5. Verify whether previous known-good image still boots

---

## Incident Response Requirements

For products in the field:

- **MUST** document how to identify installed version
- **MUST** document how to recover an unreachable device
- **MUST** document factory reset behavior
- **MUST** document provisioning steps
- **MUST** document which logs are safe to request from end users

---

## Example Operations Documents

```text
docs/
├── operations.md
├── factory-reset.md
├── provisioning.md
├── ota-recovery.md
└── troubleshooting.md
```

---

## Telemetry Guidance

If telemetry exists:

- prefer compact payloads
- sample at bounded intervals
- do not publish unchanged values excessively unless required
- include version and device health context
- protect credentials and auth material

---

## SquareLine-Specific Ops Notes

If the UI is generated from SquareLine:
- record the SquareLine Studio version used
- record the LVGL version expected by generated files
- note which generated files are safe to overwrite
- document the re-export process so support engineers do not accidentally destroy custom bindings

SquareLine exports can be sensitive to project/export structure decisions, especially in Arduino/LVGL setups; teams should preserve a documented export workflow rather than relying on ad hoc manual copying. citeturn857924search16turn857924search2

---

## Failure Modes

- no way to know what firmware is installed
- no reset reason logging
- logs too noisy to diagnose field issues
- support team edits generated UI files and loses custom behavior on next export
