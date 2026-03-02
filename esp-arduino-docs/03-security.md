# Security Model, Threats & Secure Defaults

> This file defines the minimum security posture for production-grade ESP32/ESP8266 firmware in Arduino IDE.

---

## ASSUMPTIONS

- Devices may be deployed on untrusted local networks.
- Attackers may have LAN access, physical access, or opportunistic access to exposed setup portals.
- Some devices may control relays, actuators, or sensors tied to safety, privacy, or cost.

**TODO VERIFY**
- Whether TLS is required end-to-end.
- Whether secure boot / flash encryption is in scope.
- Whether device-to-cloud credentials are per-device or fleet-wide.
- Whether captive portal provisioning is required.

---

## Threat Model

### Primary threat classes

1. **Unauthorized control**
   - attacker triggers outputs, relays, motors, or settings changes

2. **Credential disclosure**
   - Wi-Fi passwords
   - API tokens
   - MQTT credentials
   - OTA tokens

3. **Malicious or malformed input**
   - crashes firmware
   - exhausts memory
   - causes unsafe state transitions

4. **Unsafe update path**
   - unauthorized firmware
   - corrupted firmware
   - partial OTA failure

5. **Physical tampering**
   - reset pin abuse
   - serial console abuse
   - exposed debug menu access
   - local storage extraction

6. **Availability attacks**
   - reconnect storm
   - command spam
   - filesystem exhaustion
   - watchdog-triggering payloads

---

## Security Principles

**Least exposure.**  
If a service is not required, do not start it.

**Least privilege.**  
A feature may do only what it needs. Split read-only telemetry from control APIs when possible.

**Fail closed for control paths.**  
If auth state is invalid or config is corrupt, the device must reject control operations.

**No hardcoded secrets.**  
Never commit real credentials, device keys, or fleet secrets.

**Secure production builds differ from development builds.**  
Debug helpers, verbose logs, open AP setup portals, and insecure test credentials must be disabled or compile-time gated.

---

## Secrets & Credential Handling

### Rules

- **MUST NOT** hardcode production credentials in source.
- **MUST** provide `arduino_secrets.h.example` with placeholders only.
- **MUST** separate secrets from source control.
- **MUST** avoid printing secrets to serial logs.
- **MUST** support credential reset/reprovision flow if credentials are invalid.

Example placeholder file:

```cpp
#pragma once

#define SECRET_WIFI_SSID ""
#define SECRET_WIFI_PASSWORD ""
#define SECRET_MQTT_USERNAME ""
#define SECRET_MQTT_PASSWORD ""
#define SECRET_OTA_TOKEN ""
```

### Preferred practice

- Development: local uncommitted `arduino_secrets.h`
- Production: provisioning into persisted storage or per-device manufacturing step
- If credentials are persisted, validate length and sanitize before use

---

## Authentication & Authorization

### HTTP / Web UI

If a local web UI or API exists:

- **MUST** require authentication for configuration and control
- **MUST NOT** expose write endpoints unauthenticated
- **MUST** validate content type, size, and fields
- **MUST** rate-limit or debounce repeated control actions
- **MUST** protect factory reset and OTA triggers

### MQTT

- **MUST** use authenticated broker access
- **MUST** scope topics by device ID or tenant namespace
- **MUST** reject unexpected command topics
- **MUST** validate payload shape and bounds
- **SHOULD** separate state reporting topics from command topics

### BLE / SoftAP provisioning

- **MUST** time-limit provisioning modes
- **MUST** require explicit physical/user action to enter provisioning
- **MUST** exit provisioning automatically after inactivity
- **MUST NOT** leave open AP setup mode active indefinitely in production

---

## OTA Security

Espressif documents that ESP32 OTA requires a partition table with at least two OTA app slots and an OTA data partition. citeturn857924search3turn857924search17

### Production policy

- **MUST** authenticate OTA source or OTA trigger
- **MUST** verify image size fits partition strategy
- **MUST** log OTA attempt start, progress milestones, and result
- **MUST** keep prior known-good firmware when using dual-slot OTA
- **MUST** test power-loss and interrupted-update behavior on target hardware
- **MUST NOT** allow arbitrary unauthenticated upload endpoints in production

### ESP32 default

- Prefer dual-slot OTA layout
- Track current firmware version and update result
- On repeated boot failures after update, define rollback or safe-mode behavior

### ESP8266 note

OTA support exists in the ESP8266 ecosystem, but exact robustness depends heavily on board flash layout and implementation details. Treat OTA as feature-specific and verify on the target board before production rollout. citeturn857924search1

---

## Physical Security

- Disable or gate dangerous maintenance actions behind explicit triggers.
- Do not expose unauthenticated serial command shells in production.
- If factory reset uses a button, require press-hold timing and boot-state confirmation.
- Protect actuator outputs on boot with safe defaults before higher-level logic runs.
- If the device can control hazardous loads, define a hardware-safe fallback state.

---

## Input Hardening

All external inputs must be bounded.

### Mandatory checks

- maximum payload size
- valid UTF-8 or expected ASCII subset where relevant
- field presence
- numeric range checks
- enum whitelist checks
- timeout on reads
- duplicate command handling policy

### JSON guidance

- Avoid giant dynamic JSON documents.
- Use bounded document capacity.
- Reject oversized payloads before parse.
- Ignore or reject unknown keys intentionally; do not accidentally accept everything.

---

## Logging & Sensitive Data

### MUST NOT log

- Wi-Fi password
- cloud tokens
- OTA tokens
- full private certificate material
- personal data if product handles PII

### MAY log

- SSID name if allowed by product policy
- endpoint hostnames without credentials
- redacted config summary
- auth failures without revealing secret values

---

## Secure Defaults Checklist

- [ ] Setup portal disabled by default in production
- [ ] OTA disabled until credentials/config are valid
- [ ] All control actions require authenticated path
- [ ] Factory reset requires explicit physical intent
- [ ] Debug logs compile-time gated
- [ ] Device boots to safe output states
- [ ] Invalid config causes defaults or safe mode, not undefined behavior

---

## Recommended Build Profiles

### Development

- verbose logs on
- serial diagnostics on
- test credentials allowed locally
- optional insecure local endpoints only if isolated and clearly marked

### Staging

- production auth model enabled
- non-production credentials
- OTA path exercised
- logs moderate

### Production

- no debug shell
- no default credentials
- no open provisioning except explicit entry flow
- logs reduced and structured
- factory reset/documented maintenance path available

---

## Common Pitfalls

- Shipping with an always-open captive portal
- Logging full URLs containing credentials
- Treating local network as trusted
- Allowing OTA from any host on LAN
- Exposing relay control via unauthenticated HTTP GET
- Accepting malformed MQTT payloads and crashing parser
