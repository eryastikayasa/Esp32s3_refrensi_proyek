# ESP32-S3 Voice Commander — Current State

> This file describes the **current working condition**, not the complete project history. Update it after each significant debugging, tuning, or verified code change.

**Last updated:** 20 August 2026

## Current Phase

**Active runtime debugging and audio tuning.**

The project has progressed beyond the earlier DNS/TCP/TLS connection problems. Gemini WebSocket connectivity is now working, and attention has moved to runtime behavior and audio tuning based on actual device logs.

## Firmware Repository

- Repository: `eryastikayasa/esp32s3_voice_geminiproject`
- Default branch: `main`
- Context repository: `eryastikayasa/Esp32s3_refrensi_proyek`
- Firmware must remain the source of truth for executable code.

## Current Verified Conditions

### Gemini WebSocket
- WebSocket connection to Gemini has been observed as successful.
- Recent log:
  - `WS_EVENT: WebSocket TERHUBUNG ke Gemini!`
  - `WS_EVENT: Connection generation=1`
- OLED status reached:
  - `[OLED STATUS]: AI Terhubung...`

### Network
- Earlier diagnostics established DNS resolution and TCP connectivity to `generativelanguage.googleapis.com:443`.
- The investigation therefore progressed from basic network connectivity into TLS/WebSocket and application lifecycle behavior.

### Audio
- Microphone path and MAX98357A speaker path were previously proven working at audio baseline **v6.1.5**.
- v6.1.5 remains the **locked technical audio baseline**.
- Current work is tuning audio/runtime behavior from new logs, not replacing the proven baseline without evidence.

### OLED / I2C
- Recent runtime logs still show:
  - `i2c.master: I2C software timeout`
- This is currently an active diagnostic issue.
- Do not automatically attribute the I2C problem to the audio path without evidence.

## Current Important Log Evidence

Recent boot/runtime sequence includes:

```text
WS_EVENT: WebSocket TERHUBUNG ke Gemini!
WS_EVENT: Connection generation=1
DISPLAY: [OLED STATUS]: AI Terhubung...
E i2c.master: I2C software timeout
```

The successful Gemini connection and the I2C timeout should be treated as separate observations until testing proves a relationship.

## Current Priority

1. Continue audio tuning using the latest runtime logs.
2. Preserve audio behavior proven by v6.1.5.
3. Investigate the repeated OLED/I2C software timeout separately.
4. Continue monitoring WebSocket lifecycle stability after the v7.0.x crash mitigations.
5. Use measured runtime evidence before making additional architectural changes.

## Locked Constraints

- **Do not casually change the v6.1.5 audio baseline.**
- Do not perform broad refactors during targeted debugging.
- Do not create a new Git branch unless explicitly requested.
- Make changes directly on the intended working branch.
- Build and inspect after meaningful firmware changes.
- Do not declare an issue fixed without supporting build/test/runtime evidence.

## Next Action

**Audit and tune the current audio/runtime path from the latest device log, while preserving the v6.1.5 audio baseline.**

If the next test changes the state, update this file with:

- date/time
- firmware commit/version
- exact change made
- build result
- device/runtime result
- relevant log evidence
- next action

## State Classification

| Subsystem | State | Notes |
|---|---|---|
| ESP32-S3 firmware | Active | Current development target |
| Gemini WebSocket | Working / monitor | Connection observed successfully; lifecycle stability still monitored |
| Microphone INMP441 | Proven at baseline | v6.1.5 baseline |
| MAX98357A speaker | Proven at baseline | v6.1.5 baseline |
| Audio tuning | **IN PROGRESS** | Current primary tuning work |
| OLED | Degraded / diagnostic | I2C software timeouts observed |
| I2C | **ISSUE PRESENT** | Requires separate investigation |
| TLS | Previously resolved enough for WSS | Gemini WebSocket now connects |
| Network DNS/TCP | Verified | Earlier diagnostics passed |

## Relationship to PROJECT_CONTEXT.md

- `PROJECT_CONTEXT.md` = stable project knowledge, architecture, baselines, and rules.
- `CURRENT_STATE.md` = what is happening **right now**.

When these appear to conflict, verify against the latest firmware repository and runtime log before changing either document.
