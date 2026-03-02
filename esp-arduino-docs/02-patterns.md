# Coding Patterns, Templates & Implementation Rules

> This file defines mandatory coding patterns for production-grade Arduino firmware.

---

## Non-Negotiable Rules

- **MUST** use small, composable classes or modules.
- **MUST** prefer clear ownership over convenience globals.
- **MUST** keep public interfaces minimal.
- **MUST NOT** perform large blocking operations in event handlers.
- **MUST NOT** use exception-based flow.
- **MUST NOT** use recursion in firmware logic.
- **MUST** validate all external input before use.
- **MUST** use compile-time constants where possible.

---

## Naming Conventions

- Types: `PascalCase`
- Functions/methods: `camelCase`
- Constants/macros: `UPPER_SNAKE_CASE`
- Private members: `m_` prefix optional, but consistent
- Generated SquareLine identifiers: preserve original names
- File names: lowercase with underscores where possible

Examples:
- `WifiService`
- `readTemperature`
- `MAX_WIFI_RETRIES`
- `ui_controller.cpp`

---

## Logging Pattern

All logging goes through macros or a small logging wrapper.

```cpp
#pragma once
#include <Arduino.h>

namespace Log {
  inline void begin(unsigned long baud = 115200) {
    Serial.begin(baud);
  }

  inline void info(const char* tag, const char* msg) {
    Serial.printf("[INFO] [%s] %s\n", tag, msg);
  }

  inline void warn(const char* tag, const char* msg) {
    Serial.printf("[WARN] [%s] %s\n", tag, msg);
  }

  inline void error(const char* tag, const char* msg) {
    Serial.printf("[ERROR] [%s] %s\n", tag, msg);
  }
}
```

### Rules

- Include subsystem tag in every log.
- Log state transitions, not every loop iteration.
- Debug logs must be compile-time gated.
- Production logs must avoid leaking secrets.

---

## Cooperative Scheduling Pattern

Preferred over `delay()`.

```cpp
class PeriodicTimer {
public:
  explicit PeriodicTimer(uint32_t intervalMs) : m_interval(intervalMs), m_last(0) {}

  bool due(uint32_t now) {
    if (now - m_last >= m_interval) {
      m_last = now;
      return true;
    }
    return false;
  }

private:
  uint32_t m_interval;
  uint32_t m_last;
};
```

Usage:

```cpp
PeriodicTimer sensorTimer(1000);
PeriodicTimer wifiTimer(5000);

void AppController::tick() {
  const uint32_t now = millis();

  if (sensorTimer.due(now)) {
    sensorService.poll();
  }

  if (wifiTimer.due(now)) {
    wifiService.tick(now);
  }

  uiController.tick(now);
}
```

### Rules

- Use one timer per concern.
- Do not cascade delays.
- Heavy work must be chunked across ticks.

---

## Error Handling Pattern

Use explicit status return values.

```cpp
enum class StatusCode {
  Ok,
  NotReady,
  InvalidArgument,
  Timeout,
  IoFailure,
  CorruptData,
  Unsupported
};

template <typename T>
struct Result {
  StatusCode status;
  T value;
};
```

Example:

```cpp
Result<float> TemperatureService::readCelsius() {
  if (!m_driverReady) {
    return {StatusCode::NotReady, 0.0f};
  }

  const auto reading = m_driver.read();
  if (!reading.valid) {
    return {StatusCode::IoFailure, 0.0f};
  }

  return {StatusCode::Ok, reading.temperatureC};
}
```

### Rules

- Never silently swallow failures.
- Convert low-level errors into higher-level service outcomes.
- Log at the boundary where the error becomes operationally relevant.

---

## Configuration Pattern

Separate compile-time and persisted config.

### Compile-time config

```cpp
#pragma once

#define FEATURE_WIFI 1
#define FEATURE_DISPLAY 1
#define FEATURE_OTA 1

constexpr uint32_t WIFI_CONNECT_TIMEOUT_MS = 15000;
constexpr uint32_t SENSOR_POLL_INTERVAL_MS = 1000;
constexpr size_t JSON_DOC_CAPACITY = 512;
```

### Persisted settings

```cpp
struct DeviceSettings {
  char wifiSsid[33];
  char wifiPassword[65];
  char mqttHost[128];
  uint16_t mqttPort;
  bool telemetryEnabled;
};
```

### Rules

- Compile-time flags decide feature availability.
- Persisted settings decide runtime behavior.
- Persisted settings must be validated after load.
- Do not persist frequently changing telemetry counters.

---

## Safe String and Buffer Pattern

Prefer bounded formatting.

```cpp
char topic[96];
snprintf(topic, sizeof(topic), "devices/%s/status", deviceId);
```

### Rules

- Prefer fixed-size buffers over heap `String` in recurring paths.
- If `String` is used, confine it to startup, tooling, or non-hot code.
- All formatted output must respect buffer sizes.

---

## Input Validation Pattern

For any external input:
- serial commands
- HTTP query/body
- MQTT payload
- touch/UI event data
- file content
- persisted config read from flash

Template:

```cpp
bool parseBoolString(const char* input, bool& out) {
  if (!input) return false;
  if (strcmp(input, "true") == 0) { out = true; return true; }
  if (strcmp(input, "false") == 0) { out = false; return true; }
  return false;
}
```

Rules:
- Validate length first.
- Validate charset or format if applicable.
- Reject unknown fields unless intentionally extensible.
- Never trust NUL termination from external data without bounds checks.

---

## Sensor Read Pattern

Drivers must return structured validity.

```cpp
struct SensorSample {
  bool valid;
  float value;
  uint32_t atMs;
};

SensorSample LightSensorService::sample(uint32_t now) {
  const int raw = analogRead(m_pin);
  if (raw < 0) return {false, 0.0f, now};

  const float lux = convertRawToLux(raw);
  return {true, lux, now};
}
```

### Rules

- Timestamp readings at acquisition time.
- Keep conversion separate from transport.
- Debounce/noise-filter where appropriate.
- Distinguish “sensor returned invalid” from “sensor value is zero”.

---

## Debounce Pattern

```cpp
class DebouncedInput {
public:
  DebouncedInput(uint32_t debounceMs) : m_debounceMs(debounceMs) {}

  bool update(bool rawState, uint32_t now, bool& stableChanged, bool& stableState) {
    stableChanged = false;

    if (rawState != m_lastRaw) {
      m_lastRaw = rawState;
      m_lastChange = now;
    }

    if ((now - m_lastChange) >= m_debounceMs && m_stable != rawState) {
      m_stable = rawState;
      stableState = m_stable;
      stableChanged = true;
      return true;
    }

    stableState = m_stable;
    return false;
  }

private:
  uint32_t m_debounceMs;
  uint32_t m_lastChange = 0;
  bool m_lastRaw = false;
  bool m_stable = false;
};
```

---

## UI Binding Pattern for SquareLine/LVGL

Generated files own widget creation. Handwritten code owns app reactions.

Example:

```cpp
class UiController {
public:
  void begin();
  void tick(uint32_t now);
  void showTemperature(float value);
  void showWifiState(bool connected);
  void handleSavePressed();
};
```

Example binding callback:

```cpp
extern UiController g_ui;

void ui_event_save_button(lv_event_t* e) {
  if (lv_event_get_code(e) == LV_EVENT_CLICKED) {
    g_ui.handleSavePressed();
  }
}
```

### Rules

- Event callbacks must be thin.
- UI updates must be idempotent where possible.
- Avoid full-screen redraws when only one label changed.
- Use state change detection to avoid redundant LVGL writes.

---

## Retry/Backoff Pattern

```cpp
struct RetryState {
  uint8_t attempts = 0;
  uint32_t nextAllowedMs = 0;
};

bool shouldRetry(RetryState& state, uint32_t now, uint32_t baseDelayMs, uint8_t maxAttempts) {
  if (state.attempts >= maxAttempts) return false;
  if (now < state.nextAllowedMs) return false;

  state.attempts++;
  state.nextAllowedMs = now + (baseDelayMs * state.attempts);
  return true;
}
```

### Rules

- Network retries must be bounded.
- Reconnect storms are forbidden.
- Reset retry state only on successful recovery.

---

## Persistence Pattern

Use a service that validates schema version.

```cpp
struct PersistedConfig {
  uint16_t schemaVersion;
  DeviceSettings settings;
};

class SettingsService {
public:
  bool begin();
  bool load();
  bool save();
  bool resetToDefaults();
  const DeviceSettings& get() const;
};
```

### Rules

- Add schema version to persisted config.
- Define migration behavior on schema mismatch.
- Validate all fields after load.
- Never write on every loop tick.

---

## Reference Implementation: Thin Controller Pattern

```cpp
class AppController {
public:
  void begin();
  void tick();
  void enterSafeMode();

private:
  void tickNormal(uint32_t now);
  void tickSafeMode(uint32_t now);

  bool m_safeMode = false;
  PeriodicTimer m_sensorTimer{1000};
  PeriodicTimer m_healthTimer{5000};
};
```

---

## Anti-Patterns

- Giant `switch` with all application logic in `loop()`
- `while (WiFi.status() != WL_CONNECTED) { delay(500); }`
- Updating multiple LVGL widgets directly from sensor ISR
- Parsing unbounded JSON into a huge dynamic document
- Writing settings on every minor UI change
- Business logic in `ui_events.c`
- Repeated `new`/`delete` in recurring runtime paths

---

## Common Pitfalls

- Forgetting unsigned wrap-safe time comparisons with `millis()`
- Treating ESP32 and ESP8266 filesystem/storage APIs as identical
- Using synchronous HTTP requests in hot paths
- Allowing generated UI names to drift from application assumptions
