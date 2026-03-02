# Data Model, State, Persistence & Consistency

> This file defines how production firmware stores, validates, and evolves data.

---

## Persistence Categories

### 1. Compile-time constants

Examples:
- pin maps
- feature flags
- board names
- static endpoint defaults

Storage:
- headers, `constexpr`, macros

### 2. Persisted settings

Examples:
- Wi-Fi credentials
- cloud endpoint
- telemetry enable flag
- timezone or locale
- calibration factors
- display brightness preference

Storage:
- ESP32: Preferences/NVS preferred for small structured config
- ESP32/ESP8266: LittleFS when file-like storage is needed
- SD only if already required for assets or logs

### 3. Runtime state

Examples:
- connection status
- current sensor sample
- active screen
- command queue occupancy
- uptime counters

Storage:
- RAM only

### 4. Derived state

Examples:
- `isReady`
- `shouldPublishTelemetry`
- `isControlAllowed`
- `isTimeSynchronized`

Storage:
- not persisted; recomputed

---

## Persistence Strategy

### Default rules

- **MUST** use Preferences/NVS for small key/value config on ESP32.
- **MUST** use LittleFS for structured files, assets, or larger persisted blobs.
- **MUST NOT** write high-frequency changing values to flash.
- **MUST** define schema versioning.
- **MUST** define invalid-data recovery behavior.

### Why

Frequent flash writes shorten lifetime and can increase corruption risk. Persist only what must survive reboot.

---

## Data Structures

Template:

```cpp
constexpr uint16_t SETTINGS_SCHEMA_VERSION = 3;

struct DeviceSettings {
  char wifiSsid[33];
  char wifiPassword[65];
  char mqttHost[128];
  uint16_t mqttPort;
  float tempOffsetC;
  bool telemetryEnabled;
  uint8_t brightnessPct;
};

struct PersistedEnvelope {
  uint16_t schemaVersion;
  DeviceSettings data;
  uint32_t crc32;
};
```

### Rules

- Use fixed-size fields where practical.
- Add schema version explicitly.
- Add checksum if data is stored as a file blob.
- Validate all fields after load.

---

## Validation Rules

Examples:
- SSID length: `1..32`
- password length: product-specific; allow empty only if open network is explicitly supported
- MQTT host length: bounded
- brightness: `0..100`
- calibration offsets: bounded and documented

Pattern:

```cpp
bool validateSettings(const DeviceSettings& s) {
  if (strlen(s.wifiSsid) > 32) return false;
  if (s.mqttPort == 0) return false;
  if (s.brightnessPct > 100) return false;
  if (s.tempOffsetC < -20.0f || s.tempOffsetC > 20.0f) return false;
  return true;
}
```

---

## Migration Strategy

When schema version changes:

1. Attempt to read old format
2. Migrate known fields
3. Apply defaults for new fields
4. Validate migrated result
5. Persist new schema atomically if possible

Rules:
- **MUST** support at least one prior version if deployed devices exist
- **MUST** log migration result
- **MUST** fall back safely if migration fails

---

## Atomicity & Consistency

### Preferences/NVS

- Good for smaller structured key/value state
- Batch writes carefully
- Do not rewrite unchanged values

### LittleFS

- Better for file-like config, cached assets, or larger JSON blobs
- Use temp file + rename pattern where possible
- On boot, detect incomplete write markers if implemented

### Rules

- Persisted state must remain recoverable after reset/power loss
- Never leave config half-updated without a known recovery path

---

## Runtime State Model

Recommended central state:

```cpp
struct NetworkState {
  bool wifiConnected;
  bool timeSynced;
  int32_t rssi;
};

struct SensorState {
  bool valid;
  float temperatureC;
  float humidityPct;
  uint32_t lastUpdateMs;
};

struct UiState {
  bool ready;
  uint8_t activeScreenId;
  bool dirty;
};

struct AppState {
  NetworkState network;
  SensorState sensor;
  UiState ui;
  bool safeMode;
  uint32_t uptimeSeconds;
};
```

### Rules

- One central state model is preferred.
- Services update state through defined methods.
- UI reads state snapshots or derived values.
- Avoid many ad hoc globals.

---

## Concurrency Rules

### Default

Use cooperative single-threaded access.

### If ESP32 tasks are used

- Shared mutable state must be:
  - queue-based, or
  - protected and documented
- UI objects must be updated from the correct context for the chosen LVGL integration
- File writes and network callbacks must not race settings updates

### Interrupt rules

ISRs:
- must be tiny
- must not allocate memory
- must not call heavy drivers or UI code
- should set flags or enqueue minimal data only

---

## Caching Strategy

Use caching only where it improves resilience.

Examples:
- last known sensor sample for UI display
- last known NTP sync time
- locally queued telemetry waiting for send
- preprocessed UI assets metadata

Do not cache:
- secrets in plaintext files unless explicitly approved
- giant historical telemetry logs without a retention plan

---

## SquareLine / LVGL Asset State

If SquareLine assets exist, document:

- asset source of truth
  - generated C arrays
  - LittleFS
  - SD card
- image/font memory impact
- lazy vs eager asset load
- LVGL file system bridge requirements

LVGL provides file system abstraction for mounted filesystems, which matters if assets are loaded from LittleFS or SD rather than compiled into firmware. citeturn622475search2turn622475search17

### Rules

- Do not assume all assets belong in flash arrays.
- Large image sets may need external storage strategy.
- Any asset path used by generated UI must be verified against actual filesystem registration.

---

## Failure Modes

- corrupted settings cause boot loop
- oversize asset storage exhausts flash
- repeated settings writes wear flash
- invalid migration leaves device unusable
- race between UI and service state updates
- stale cached network state shown as current

---

## Common Pitfalls

- storing every runtime metric in flash
- no schema version
- using `String`-serialized JSON for config without size bounds
- assuming LittleFS always mounts cleanly after bad power events
