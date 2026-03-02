# Build, Release, Flashing & Deployment Discipline

> This file defines how firmware should be built, versioned, flashed, and released.

---

## ASSUMPTIONS

- Arduino IDE is the primary developer workflow.
- Some teams may also use arduino-cli or PlatformIO for automation, but this document does not require that as the primary flow.
- Production devices may require OTA-capable builds on ESP32.

---

## Release Principles

- **MUST** version every firmware build.
- **MUST** tie each release to a board profile.
- **MUST** preserve release notes with upgrade risks.
- **MUST** distinguish development, staging, and production builds.
- **MUST** keep a reproducible list of required libraries and versions.

---

## Required Version Metadata

Expose at runtime:
- firmware name
- semantic version
- build date/time
- git commit or manual release ID
- board profile
- feature flags relevant to support

Example:

```cpp
#pragma once

#define FW_NAME "my-device"
#define FW_VERSION "1.4.0"
#define FW_BOARD_PROFILE "esp32_ili9341_touch"
#define FW_BUILD_DATE __DATE__ " " __TIME__
```

---

## Library & Toolchain Tracking

Maintain a checked-in document such as `docs/toolchain.md` or `libraries.lock.md` listing:

- Arduino IDE major version
- ESP32 board package version
- ESP8266 board package version
- LVGL version
- display driver library versions
- SquareLine Studio version used for export
- any networking or sensor library versions

### Why

Embedded builds drift quickly. Reproducibility requires recorded versions.

---

## Flashing Policy

### Development flashing

- USB/UART flashing allowed
- verbose serial logs allowed
- unsafe test credentials allowed only locally

### Production flashing

- board profile must be explicit
- factory image and release notes must match
- provisioning flow must be documented
- post-flash smoke test required

---

## OTA Release Policy

On ESP32, OTA deployment should use a partition layout that supports dual app slots plus OTA metadata partition, per Espressif guidance. citeturn857924search3turn857924search17

### OTA checklist

- [ ] firmware fits partition layout
- [ ] current and target versions logged
- [ ] update source authenticated
- [ ] rollback or safe failure behavior documented
- [ ] post-update health check defined

### Rollout strategy

- pilot devices first
- staged percentage rollout if fleet tooling exists
- monitor boot success and reconnect rates before broad rollout

---

## Recommended Repo Files

```text
docs/
├── flashing.md
├── release-process.md
├── libraries.lock.md
└── board-profiles.md

releases/
├── CHANGELOG.md
└── release-notes/
    ├── 1.4.0.md
    └── 1.4.1.md
```

---

## Example Release Notes Template

```md
# Release 1.4.0

## Boards
- esp32_ili9341_touch

## Changes
- Added Wi-Fi provisioning timeout
- Reduced UI redraw churn
- Added safe-mode fallback for invalid settings

## Migration
- Settings schema version increased from 2 to 3
- Brightness now stored as 0..100

## Risks
- Devices with old custom config files may reset to defaults

## Validation Performed
- cold boot
- no Wi-Fi boot
- save settings
- 2h soak
```

---

## Deployment Checklist

- [ ] correct board selected
- [ ] correct partition or memory settings selected
- [ ] required libraries installed in compatible versions
- [ ] secrets file present locally and not committed
- [ ] serial monitor smoke test passes
- [ ] version metadata matches intended release
- [ ] safe outputs confirmed on boot
- [ ] provisioning or offline mode confirmed

---

## Example Related Config Files

### `arduino_secrets.h.example`

```cpp
#pragma once

#define SECRET_WIFI_SSID ""
#define SECRET_WIFI_PASSWORD ""
#define SECRET_MQTT_HOST ""
#define SECRET_MQTT_USERNAME ""
#define SECRET_MQTT_PASSWORD ""
#define SECRET_OTA_TOKEN ""
```

### `docs/libraries.lock.md`

```md
# Toolchain Lock

- Arduino IDE: TODO VERIFY
- ESP32 Boards package: TODO VERIFY
- ESP8266 Boards package: TODO VERIFY
- LVGL: TODO VERIFY
- SquareLine Studio export version: TODO VERIFY
- TFT_eSPI or board display stack: TODO VERIFY
```

---

## Optional Automation Guidance

If the repo also supports CLI automation, mirror Arduino IDE builds with:
- arduino-cli board-specific compile command
- artifact naming by board + version
- release artifact checksum generation

This is optional, but recommended for teams.

---

## Failure Modes

- release built with wrong board package
- UI export generated against different LVGL version than runtime
- firmware binary exceeds OTA partition size
- undocumented library version drift breaks build
- production image shipped with debug setup portal enabled

---

## Common Pitfalls

- no record of which SquareLine version generated the UI
- no release notes for settings migrations
- flashing wrong board definition and misdiagnosing runtime failures
