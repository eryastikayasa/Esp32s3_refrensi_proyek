# CHANGELOG_PROJECT — Addendum v7.2.2

Tanggal: 21 Agustus 2026

## Keputusan arsitektur baru

Implementasi audio ESP32-S3 dilanjutkan dengan pola arsitektur AudioService XiaoZhi, tetapi codec/protokol tetap mengikuti Gemini Live.

Referensi yang dipakai:
- XiaoZhi AudioService: AudioInputTask → audio queue → codec/network queue; arah output dipisahkan menjadi decode queue → playback queue → AudioOutputTask.
- Gemini Live: WebSocket bidirectional, input PCM16 16 kHz, output PCM16 24 kHz.

Sumber referensi:
- https://github.com/78/xiaozhi-esp32/blob/main/main/audio/README.md
- https://github.com/78/xiaozhi-esp32/blob/main/main/audio/audio_service.cc
- https://ai.google.dev/gemini-api/docs/live-api/get-started-websocket

## Implementasi repo firmware

Repo: `eryastikayasa/esp32s3_voice_geminiproject`
Branch: `main`

Ditambahkan komponen:

```text
components/audio_input/
├── CMakeLists.txt
├── audio_input.cpp
└── include/audio_input.h
```

### Jalur AUDIO IN

```text
INMP441
   ↓
audio_input task
   ↓
input queue (3 × 3200 byte)
   ↓
audio_input_tx dispatch task
   ↓
websocket_send_audio_data()
   ↓
WebSocket TX queue
   ↓
WebSocket TX worker
   ↓
Gemini Live
```

Perubahan penting:
- `main.cpp` tidak lagi memiliki task microphone sendiri.
- Tidak ada lagi silent-frame gate di `main.cpp`; PCM input diperlakukan sebagai stream realtime.
- Audio input tidak melakukan TLS/WebSocket write langsung.
- Callback hanya meneruskan frame ke lapisan transport.
- Frame input 3200 byte = 100 ms PCM16 mono pada 16 kHz.
- Queue realtime tidak boleh membuat task microphone menunggu; jika penuh, frame tertua dibuang agar frame terbaru tetap dapat masuk.

### Jalur AUDIO OUT

Jalur output yang sudah dibuat pada v7.2.1 tetap dipertahankan:

```text
Gemini Live
   ↓
WebSocket RX worker
   ↓
Base64 / PCM decode
   ↓
audio_stream output ring 48 KiB
   ↓
playback task
   ↓
I2S v6.1.5
   ↓
MAX98357A
```

Output ring dan playback tetap menjadi milik `audio_stream`, bukan WebSocket.

## Yang TIDAK diubah

- I2S v6.1.5 tetap LOCKED.
- INMP441 tetap.
- MAX98357A tetap.
- PCM16 tetap.
- Mic 16 kHz tetap.
- Speaker 24 kHz tetap.
- Baseline test tone tetap.
- Tidak ada tuning amplifier/I2S berdasarkan dugaan.

## Tujuan pengujian berikutnya

Membuktikan bahwa pemisahan producer/queue/transport/playback menghilangkan drop/starvation yang terlihat pada Tahap 2.

Statistik yang harus diamati:

```text
input frames captured
input frames dropped
RX fragments
RX seq_err
RX buffer_drop
received
queued
played
pending
dropped
underrun
PLAYBACK COMPLETE
```

## Status

**CODE: SELESAI DI MAIN**

**BUILD/HARDWARE TEST: BELUM**

Jangan mengubah I2S v6.1.5 sebelum hasil hardware arsitektur baru tersedia.

## Catatan

Addendum ini dibuat agar keputusan arsitektur v7.2.2 tercatat tanpa menimpa sejarah `CHANGELOG_PROJECT.md`. Setelah validasi hardware berhasil, milestone ini perlu dimasukkan sebagai milestone baru ke `CHANGELOG_PROJECT.md` utama.
