# Step-by-Step Workflow for AI Coding Agents

> Follow this exact workflow when adding or modifying any feature in an Arduino IDE ESP32/ESP8266 project.

---

## Phase 1: Repo Reconnaissance

Before writing code, inspect the repo.

### Required scan

- [ ] identify board targets
- [ ] identify current folder layout
- [ ] identify libraries in use
- [ ] identify storage approach
- [ ] identify UI stack presence
- [ ] identify generated code folders
- [ ] identify secrets placeholders
- [ ] identify board pin maps
- [ ] identify existing logging approach
- [ ] identify current boot/init flow

### SquareLine-specific scan

If SquareLine or LVGL files exist, inspect:

- [ ] generated folder name
- [ ] `ui.h`, `ui.c`, `ui_events.*`, `ui_helpers.*`
- [ ] screen/widget identifiers
- [ ] custom fonts and images
- [ ] asset loading approach
- [ ] LVGL version assumptions
- [ ] existing event callback boundaries
- [ ] files likely overwritten on export

### Output required from AI agent before coding

Produce a short implementation brief:
- target board(s)
- affected modules
- whether generated files are involved
- memory/performance risk
- persistence impact
- security impact
- test plan summary

---

## Phase 2: Design the Change

Answer these before coding:

- What module should own the feature?
- Does it touch hardware, services, UI, or all three?
- What state is added?
- What data must persist?
- What happens on boot with invalid config?
- What happens offline?
- What happens if the dependency is missing or failing?
- Does this increase heap, flash, or CPU load materially?

If the feature touches SquareLine UI:
- define which generated widgets will be read or updated
- define which callbacks remain generated vs handwritten
- define how re-export safety is preserved

---

## Phase 3: Implement Safely

### Coding rules

- change the smallest responsible surface area
- keep `.ino` thin
- do not place business logic in generated files
- add validation before adding convenience
- add logs at state transitions
- preserve backward compatibility where possible

### Required outputs

The AI agent must generate:
- implementation code
- any related config/header changes
- board profile changes if needed
- persistence/migration updates if needed
- test plan
- manual validation checklist
- documentation for new settings, pins, and behavior

---

## Phase 4: Verify

### Compile/build verification

- [ ] includes resolve
- [ ] no symbol conflicts
- [ ] no obvious API mismatch between ESP32 and ESP8266 targets
- [ ] no generated/manual callback mismatch

### Functional verification

- [ ] boot path tested
- [ ] degraded path tested
- [ ] invalid input tested
- [ ] safe-state behavior tested
- [ ] UI interactions tested if applicable

### Reliability verification

- [ ] no blocking loops introduced
- [ ] retry policy bounded
- [ ] heap implications reviewed
- [ ] logs are useful and not noisy
- [ ] settings writes not excessive

---

## Phase 5: Document

For each feature, the AI agent must update documentation with:

- what was added
- where it lives
- board or wiring assumptions
- config keys and defaults
- persistence schema changes
- security implications
- failure modes
- validation steps
- rollback/removal steps if relevant

---

## Required “AI Agent Output Contract”

When the AI agent finishes a feature, it must provide:

### 1. Summary

- feature goal
- changed files
- key risks
- key assumptions

### 2. Technical explanation

- module responsibilities
- lifecycle and boot impact
- scheduling impact
- memory/performance impact
- persistence impact
- security impact

### 3. Validation pack

- compile checks
- manual test checklist
- fault scenarios
- expected logs

### 4. Developer guidance

- wiring changes if any
- required libraries
- settings to edit
- how to regenerate UI if SquareLine is used
- how to flash and test

---

## Special Workflow: SquareLine Present

If SquareLine is present, use this stricter workflow.

1. Identify generated files
2. Mark them read-only by policy
3. Add bindings/wrappers outside generated files
4. If widget or screen changes are required, state whether they belong in SquareLine project source
5. After any re-export, re-check:
   - callback function names
   - widget identifiers
   - include paths
   - asset references
   - LVGL compatibility

### Required warning text in AI output

> This repository contains generated SquareLine/LVGL files. Custom business logic must remain outside generated files because future exports may overwrite generated content.

---

## Common Failure Modes During Feature Work

- AI edits generated UI files directly
- AI adds blocking reconnect loops
- AI stores rapidly changing state in flash
- AI introduces board-specific pin assumptions into generic modules
- AI adds a feature without a degraded-mode plan
- AI adds UI labels/images causing memory regressions without review

---

## Final Pre-Merge Checklist

- [ ] repo scan completed
- [ ] assumptions documented
- [ ] no generated file logic pollution
- [ ] security review done
- [ ] persistence review done
- [ ] performance review done
- [ ] manual validation checklist included
- [ ] docs updated
