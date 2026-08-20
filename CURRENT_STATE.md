# ESP32-S3 Voice Commander — Current State

> Kondisi aktif proyek. Sejarah lengkap ada di `CHANGELOG_PROJECT.md`.

**Last updated:** 20 August 2026 — v7.0.31 firmware tuning, waiting hardware log

## Current Phase

**RX persistent-slot headroom + audio backpressure tuning.**

v7.0.30 sudah membuktikan full Gemini turn dapat selesai tanpa `transport_poll_write(0)`. Bottleneck sekarang adalah RX burst/slot pressure dan audio buffering.

## v7.0.31 Change

Firmware repository: `eryastikayasa/esp32s3_voice_geminiproject`

Changed in `components/websocket/include/websocket_internal.h`:

```text
WS_RX_SLOT_COUNT: 6 -> 8
WS_RX_QUEUE_LENGTH: 6 -> 8 (through WS_RX_SLOT_COUNT)
```

Reason: v7.0.30 reached `queue_hwm=4` and `buffer_drop=4` during Gemini audio burst. The persistent RX pool could exhaust while RX worker still held queued slots.

Firmware commit:

```text
dce28378d717537361ded64f35cea5e1fedab9fd
```

## What Is Locked / Unchanged

- I2S baseline v6.1.5.
- INMP441.
- MAX98357A.
- Mic 16 kHz.
- Speaker 24 kHz.
- Audio ring 32 KB.
- Audio prebuffer 12 KB.
- WebSocket TX timeout/retry tuning from v7.0.30.
- WebSocket PING OFF.
- OLED/I2C OFF for isolation.
- No broad refactor.
- No new branch.

## v7.0.30 Baseline Evidence

```text
received=375834
queued=373488
played=373488
pending=0
dropped=1990
```

Peak ring occupancy:

```text
pending=31510/32768
```

RX pressure:

```text
dropped_frag=49
seq_err=45
buffer_drop=4
queue_hwm=4
```

PCM drops included 38, 998, 774, and 180 bytes. One playback underrun occurred, but final audio drain succeeded.

## v7.0.31 Success Criteria

Primary:

```text
buffer_drop -> 0
queue_drop -> 0
queue_hwm < 8
```

Secondary:

```text
dropped_frag decreases
seq_err decreases
PCM drop decreases
playback underrun disappears
```

If `buffer_drop` reaches zero but `seq_err` remains high, **do not increase slot count again**. Next investigation should inspect `payload_offset` / fragment ordering and the RX event pipeline.

## Current Priority Order

1. Flash/build v7.0.31.
2. Capture one complete Gemini turn.
3. Compare `buffer_drop`, `queue_hwm`, `queue_drop` against v7.0.30.
4. Then compare `seq_err` and `dropped_frag`.
5. Then evaluate PCM drops and playback underrun.
6. Do not touch I2S v6.1.5.

## State Table

| Subsystem | State |
|---|---|
| Wi-Fi / DNS / TCP 443 | VERIFIED |
| Gemini WebSocket | VERIFIED |
| Gemini setupComplete | VERIFIED |
| WebSocket/TLS TX | IMPROVED / MONITOR |
| Audio I2S | LOCKED v6.1.5 |
| INMP441 | VERIFIED BASELINE |
| MAX98357A | VERIFIED BASELINE |
| RX persistent slots | **TUNED v7.0.31 — TEST PENDING** |
| RX fragment handling | ISSUE UNDER TEST |
| Audio ring | PRESSURE OBSERVED |
| PCM queue | DROP OBSERVED |
| Playback | UNDERRUN OBSERVED |
| OLED/I2C | OFF |
| PING | OFF |

## Next Required Input

**Hardware log v7.0.31.**

Do not make another tuning change until that log is analyzed against the v7.0.30 baseline, unless a build failure requires a code correction.
