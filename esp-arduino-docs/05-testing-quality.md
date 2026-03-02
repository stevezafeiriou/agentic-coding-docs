# Testing Strategy, Hardware Validation & Quality Gates

> This file defines how AI agents must create testable firmware and verification artifacts.

---

## Core Rule

Every production feature must come with a test plan.  
Even when full host-side unit tests are limited in Arduino projects, the AI agent must still produce:

1. **host-testable logic separation**
2. **manual hardware validation checklist**
3. **fault-injection scenarios**
4. **performance and soak validation steps**
5. **example serial logs and expected behavior**

---

## Testing Layers

### 1. Compile-time and static checks

Minimum:
- build cleanly for target board profile
- no duplicate symbols
- no missing includes
- warnings reviewed
- no dead experimental code left enabled

### 2. Host-testable logic

Extract pure logic where possible:
- parsing
- validation
- retry policy
- debounce
- scheduling
- state transitions
- migration logic
- topic/path formatting

These can be tested outside hardware-specific code.

### 3. Module-level hardware tests

Examples:
- sensor returns valid sample under normal conditions
- relay output active-low logic correct
- display initializes without watchdog reset
- settings load/save round trip works

### 4. Integration tests on target hardware

Examples:
- cold boot with valid config
- cold boot with invalid config
- Wi-Fi down at startup
- OTA service disabled without credentials
- UI button changes persisted setting correctly

### 5. Soak and recovery tests

Examples:
- 24h runtime without heap drift beyond threshold
- repeated AP disconnect/reconnect
- repeated sensor unplug/replug if feasible
- repeated screen navigation without memory leak symptoms
- multiple reboots with persisted config retained

---

## Mandatory Deliverables for Each Feature

For every feature the AI agent adds, it must also produce:

- [ ] test plan section in code comments or docs
- [ ] serial-log expectations
- [ ] manual validation checklist
- [ ] edge-case list
- [ ] failure-mode list
- [ ] rollback/removal notes if feature is risky

---

## Testable Code Structure Rules

- Pure functions must be isolated from hardware.
- Hardware wrappers must be small and mockable conceptually.
- Parsing and validation must not live inside giant I/O functions.
- State machine transitions must be observable and logged.

Example pure function to test:

```cpp
bool shouldEnterSafeMode(uint8_t failedBoots, bool storageOk, bool settingsValid) {
  if (!storageOk) return true;
  if (failedBoots >= 3 && !settingsValid) return true;
  return false;
}
```

---

## Example Test Cases

### Settings validation

```cpp
// Pseudocode or host-side test harness
ASSERT_TRUE(validateSettings(validSettings));
ASSERT_FALSE(validateSettings(settingsWithBrightness101));
ASSERT_FALSE(validateSettings(settingsWithPortZero));
```

### Debounce logic

```cpp
DebouncedInput button(50);

// simulate bouncing transitions and verify stable change only after debounce window
```

### Retry logic

```cpp
RetryState state;
ASSERT_TRUE(shouldRetry(state, 1000, 500, 3));
ASSERT_FALSE(shouldRetry(state, 1100, 500, 3)); // not yet due
```

### Migration logic

- old schema V2 loads
- new fields defaulted
- migrated schema validates
- save persists V3

---

## Hardware Validation Checklists

### Boot

- [ ] cold boot with USB power
- [ ] cold boot with external power
- [ ] warm reset
- [ ] brownout or rapid power cycle behavior observed
- [ ] reset reason logged

### Networking

- [ ] device boots without Wi-Fi present
- [ ] wrong credentials handled safely
- [ ] reconnect after router returns
- [ ] NTP unavailable behavior defined
- [ ] command channel rejects malformed input

### Storage

- [ ] first boot with empty storage
- [ ] load after valid save
- [ ] invalid/corrupt settings fallback
- [ ] filesystem mount failure behavior observed

### UI / SquareLine

- [ ] all screens render
- [ ] all interactive widgets reachable
- [ ] expected callbacks fire
- [ ] no visible tearing or stalled touch events
- [ ] UI remains responsive during network reconnect
- [ ] repeated navigation does not crash or degrade

### Sensor/Actuator

- [ ] sensor disconnected at boot
- [ ] sensor disconnected during runtime
- [ ] actuator defaults safe on boot
- [ ] wrong polarity not possible due to explicit config

---

## Performance Validation

- [ ] `loop()` latency measured under normal load
- [ ] heap free monitored over extended runtime
- [ ] no watchdog resets during display/network stress
- [ ] LVGL update cadence acceptable if UI is present
- [ ] no repeated expensive re-renders on unchanged state

---

## Suggested Instrumentation

Record during tests:
- firmware version
- board model
- flash/PSRAM info if available
- free heap at boot
- free heap after 1 min / 1 hr / 24 hr
- Wi-Fi reconnect count
- sensor error count
- reset reason
- last OTA result if applicable

---

## Acceptance Gates

A feature fails quality gate if any of the following is true:

- boot blocked indefinitely
- watchdog resets appear under expected usage
- persisted config can brick device
- unauthenticated control path exists
- generated UI/manual code coupling is fragile and undocumented
- memory trend shows likely leak or fragmentation problem
- no documented recovery path exists

---

## Example Manual Test Matrix

| Scenario | Expected Result |
| --- | --- |
| Boot with no saved Wi-Fi | Device enters unprovisioned or offline-safe state |
| Boot with corrupt config | Defaults or safe mode, no boot loop |
| Press UI save button | Validated config persists, feedback shown |
| Router power off | Device remains responsive locally |
| Sensor unplugged | Error state shown, no crash |
| OTA requested without auth | Rejected |
| Rapid repeated commands | Device rate-limits or safely serializes |

---

## Common Pitfalls

- claiming code is “tested” after only one successful flash
- no test steps for generated UI flows
- no negative tests for bad config
- measuring only happy-path boot, not degraded-mode boot
