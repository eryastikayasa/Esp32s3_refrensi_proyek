# ESP32-S3 Voice Commander — Project Context

## 1. Project Identity

- Project: ESP32-S3 Voice Commander / Asisten Kamar
- Firmware repository: `eryastikayasa/esp32s3_voice_geminiproject`
- Reference/context repository: `eryastikayasa/Esp32s3_refrensi_proyek`
- Firmware stack: ESP32-S3, ESP-IDF 6.0.1, PlatformIO
- Main purpose: voice assistant / voice commander using Gemini Live API over secure WebSocket.

## 2. Hardware Architecture

### ESP32-S3 Voice Commander
- Microphone: INMP441
- Speaker amplifier: MAX98357A
- Display: OLED over I2C
- Communication to ESP32 DevKit V1: UART / Serial1

### ESP32-S3 microphone I2S
- BCLK / SCK: GPIO5
- WS / LRCK: GPIO4
- SD: GPIO6
- Input sample rate: 16 kHz
- Audio format: PCM16

### ESP32-S3 speaker I2S
- BCLK: GPIO15
- LRCK: GPIO16
- DOUT: GPIO7
- Gemini native output sample rate: 24 kHz
- Audio format: PCM16 mono from Gemini, converted as required by the I2S TX path.

## 3. Gemini Live / WebSocket

- Endpoint: Gemini Live API through `generativelanguage.googleapis.com:443`
- Transport: WebSocket Secure (WSS)
- TLS verification is enabled.
- Hostname verification is enabled.
- Previous TLS work included certificate-bundle configuration and certificate-chain troubleshooting.
- A later milestone successfully established the Gemini WebSocket connection.

## 4. Audio Baselines — IMPORTANT

### v6.1.3
Technical baseline used for deterministic I2S speaker test-tone validation.

### v6.1.5 — LOCKED AUDIO BASELINE
This is the proven technical audio baseline.

- Microphone input works.
- MAX98357A speaker output works.
- Audio I2S configuration at this baseline must NOT be changed casually.
- Any future audio tuning must preserve the proven v6.1.5 behavior unless an explicit decision is made to replace the baseline.

## 5. Firmware Version History

### v7
Language/output speech development began.

### v7.0.1
Updated language/output speech behavior for Indonesian.

### v7.0.2
Focused on WebSocket crash mitigation. A crash path involved `ws_poll_read()` → `esp_transport_poll_read()` → `esp_websocket_client_task()`.

### v7.0.3
Mitigated WebSocket race/lifecycle problems:
- Gemini setup moved into a separate task.
- Connection generation was introduced so setup belonging to an old connection can be cancelled/invalidated.
- Connection state is checked before sending.
- WebSocket send has a timeout.

### v7.0.4
Focused on completing the WebSocket crash-resolution work, including changes around Wi-Fi management, main application flow, and WebSocket manager lifecycle.

## 6. Known Diagnostic History

### I2C / OLED
Repeated runtime logs have shown:
`i2c.master: I2C software timeout`

This must be treated as a separate diagnostic path from the proven audio baseline unless evidence shows an interaction.

### TLS
Earlier logs included:
- `No matching trusted root certificate found`
- certificate verification failure
- mbedTLS handshake failure
- WebSocket connection failure to Gemini

Network diagnostics later established DNS and TCP connectivity to `generativelanguage.googleapis.com:443`, allowing the investigation to move to TLS/WebSocket.

### WebSocket
The project progressed from TLS/WebSocket connection failures and lifecycle crashes to successful Gemini WebSocket connection logs.

## 7. Current Working Direction

As of 20 August 2026, the project is in active debugging/tuning.

Recent runtime evidence includes:
- Gemini WebSocket connected successfully.
- Connection generation reported as `1`.
- OLED status reported `AI Terhubung...`.
- I2C software timeout messages were still observed.
- Audio tuning is being performed from actual runtime logs.

The latest runtime log is the primary evidence for deciding the next debugging/tuning action.

## 8. Working Rules for AI

1. Read this context before modifying the firmware repository.
2. Treat the firmware repository as the source of truth for current code.
3. Treat this repository as the source of truth for project decisions, baselines, and historical context.
4. Do not modify the locked v6.1.5 audio baseline without explicit justification and verification.
5. Prefer minimal, evidence-based changes over broad refactors during debugging.
6. Use current logs as evidence; do not assume an old failure is still present after it has been fixed.
7. Audit the current repository state before editing.
8. After a meaningful code change, build and inspect the result before declaring the change successful.
9. Record important verified changes and new discoveries in the project reference repository.
10. Do not create a new Git branch unless explicitly requested. Work directly on the intended branch.

## 9. Repository Relationship

The two repositories have different responsibilities:

- `esp32s3_voice_geminiproject` = firmware/source code.
- `Esp32s3_refrensi_proyek` = AI-readable project memory, decisions, baselines, and historical context.

Do not duplicate the firmware source into the reference repository.

## 10. Current Priority

Continue evidence-based audio/runtime tuning while protecting the proven v6.1.5 audio baseline and separately tracking OLED/I2C and WebSocket lifecycle issues.

Last context update: 20 August 2026.
