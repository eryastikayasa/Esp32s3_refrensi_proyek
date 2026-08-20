# ESP32-S3 Voice Commander — Current State

> This file describes the **current working condition**, not the complete project history. Update it after each significant debugging, tuning, or verified code change.

**Last updated:** 20 August 2026 — v7.0.30 hardware test

## Current Phase

**Active RX/audio buffering and backpressure tuning.**

The project has progressed beyond the earlier DNS/TCP/TLS connection problems and the v7.0.29 WebSocket/TLS TX failure. v7.0.30 completed a full Gemini turn without the previous `transport_poll_write(0)` failure. Attention is now on RX fragment pressure, sequence errors, buffer drops, and audio queue backpressure.

## Firmware Repository

- Repository: `eryastikayasa/esp32s3_voice_geminiproject`
- Default branch: `main`
- Context repository: `eryastikayasa/Esp32s3_refrensi_proyek`
- Firmware must remain the source of truth for executable code.

## Current Verified Conditions

### Gemini WebSocket
- WebSocket connection succeeds.
- `setupComplete` succeeds.
- v7.0.30 completed a full Gemini generation and turn.
- No `transport_poll_write(0)` / `esp_transport_write() returned 0` was observed during the tested turn.
- No WebSocket disconnect occurred before turn completion.

Evidence:

```text
WS_EVENT: WebSocket TERHUBUNG ke Gemini!
WS_JSON: Gemini setupComplete: SESI SIAP
WS_JSON: Gemini: GENERATION COMPLETE
WS_JSON: Gemini: TURN COMPLETE - menunggu audio drain
```

### Network
- DNS resolution: verified.
- TCP 443 connectivity: verified.
- Gemini WebSocket/TLS handshake: verified in runtime.

### Audio
- Microphone path and MAX98357A speaker path remain based on the proven audio baseline **v6.1.5**.
- v6.1.5 remains the **locked technical audio baseline**.
- v7.0.30 completed playback drain:

```text
received=375834
queued=373488
played=373488
pending=0
dropped=1990
```

- The audio ring buffer reached high occupancy:

```text
pending=31510/32768
```

- PCM drops were measured during RX/audio bursts.
- One playback underrun was still observed.

### RX / Audio Buffering — ACTIVE ISSUE

v7.0.30 shows increasing RX fragment and sequence errors during large audio bursts:

```text
dropped_frag=12 seq_err=10 buffer_drop=2
dropped_frag=34 seq_err=31 buffer_drop=3
dropped_frag=49 seq_err=45 buffer_drop=4
```

Measured PCM queue drops included:

```text
38 byte
998 byte
774 byte
180 byte
```

This is the current primary investigation target.

### OLED / I2C
- OLED remains disabled for audio isolation.
- The OLED-off runtime did not reproduce the earlier repeated I2C software timeout messages.
- Do not reintroduce OLED/I2C into the audio stability test until the current RX/audio issue is characterized.

## Current Important Log Evidence

v7.0.30:

```text
WS_EVENT: WebSocket TERHUBUNG ke Gemini!
WS_JSON: Gemini setupComplete: SESI SIAP
WS_JSON: Gemini: GENERATION COMPLETE
WS_JSON: Gemini: TURN COMPLETE - menunggu audio drain
WS_AUDIO: AUDIO PLAYBACK COMPLETE: received=375834 queued=373488 played=373488 pending=0 dropped=1990 balance=356
```

The previous v7.0.29 failure:

```text
E transport_ws: Error transport_poll_write(0)
E websocket_client: esp_transport_write() returned 0
```

was not reproduced in the v7.0.30 turn test. Treat TX write stability as **improved/verified for this test**, not as a permanent guarantee yet.

## Current Priority

1. Investigate RX fragment drops and sequence errors during Gemini audio bursts.
2. Investigate RX buffer drops and persistent-slot pressure.
3. Investigate audio queue backpressure when the 32 KB ring approaches full.
4. Reduce PCM drop and eliminate playback underrun.
5. Preserve the audio behavior proven by v6.1.5.
6. Keep WebSocket PING OFF during this controlled investigation.
7. Keep OLED OFF during this audio isolation phase.

## Locked Constraints

- **Do not casually change the v6.1.5 audio baseline.**
- Do not perform broad refactors during targeted debugging.
- Do not create a new Git branch unless explicitly requested.
- Make changes directly on the intended working branch.
- Build and inspect after meaningful firmware changes.
- Do not declare an issue fixed without supporting build/test/runtime evidence.
- Do not treat one successful v7.0.30 turn as proof that the TX issue can never recur.

## Next Action

**Audit the RX persistent-slot/fragment pipeline and audio queue backpressure from the v7.0.30 log. Tune only the bottleneck responsible for `dropped_frag`, `seq_err`, `buffer_drop`, PCM drops, and underrun. Do not change I2S v6.1.5.**

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
| Gemini WebSocket | **Working / monitor** | Full v7.0.30 turn completed |
| WebSocket/TLS TX write | **Improved / verify further** | Previous write failure not reproduced in v7.0.30 test |
| Microphone INMP441 | Proven at baseline | v6.1.5 baseline |
| MAX98357A speaker | Proven at baseline | v6.1.5 baseline |
| Audio tuning | **IN PROGRESS** | RX/audio buffering now primary |
| RX fragment handling | **ISSUE PRESENT** | `dropped_frag` and `seq_err` increase during burst |
| RX buffer | **ISSUE PRESENT** | `buffer_drop` observed |
| Audio queue | **PRESSURE PRESENT** | Ring reached 31510/32768; PCM drops observed |
| Playback | **MOSTLY WORKING / UNDERRUN OBSERVED** | Final drain succeeds but one underrun remains |
| OLED | Disabled / diagnostic | Kept OFF during isolation |
| I2C | Not active in current test | OLED disabled |
| TLS | Verified enough for WSS | Full turn completed |
| Network DNS/TCP | Verified | Diagnostics pass |

## Relationship to PROJECT_CONTEXT.md

- `PROJECT_CONTEXT.md` = stable project knowledge, architecture, baselines, and rules.
- `CURRENT_STATE.md` = what is happening **right now**.

When these appear to conflict, verify against the latest firmware repository and runtime log before changing either document.
