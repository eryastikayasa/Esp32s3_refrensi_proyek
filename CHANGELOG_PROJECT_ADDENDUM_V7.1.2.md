# CHANGELOG PROJECT — ADDENDUM v7.1.2

> Catatan tambahan sementara untuk milestone v7.1.1 hardware dan v7.1.2. Sejarah `CHANGELOG_PROJECT.md` utama tidak dihapus atau ditimpa.

## v7.1.1 — Hardware Test Result — August 21, 2026

### Hasil utama

- Wi-Fi mendapatkan IP dan network READY.
- DNS `generativelanguage.googleapis.com` OK.
- TCP 443 OK.
- WebSocket Gemini TERHUBUNG.
- `Gemini setupComplete: SESI SIAP` OK.
- Session resumption tetap berjalan.
- Gemini berhasil mengirim audio PCM.
- Audio playback berhasil sampai `TURN COMPLETE` dan akhirnya `pending=0`.

### Masalah yang ditemukan

RX/audio burst masih terlalu berat:

```text
fragments=525
 dropped_frag=154
seq_err=144
buffer_drop=10
```

Audio juga mengalami drop:

```text
received=344076
queued=336752
played=336752
dropped=7012
pending=0
```

Terjadi satu:

```text
AUDIO PLAYBACK UNDERRUN
```

Dan Task WDT muncul dua kali:

```text
Task watchdog got triggered
- IDLE1 (CPU 1)
CPU 1: audio_playback
```

### Analisis

Masalah utama bukan DNS, TCP, TLS, WebSocket setup, atau format I2S.

Masalah sudah menyempit menjadi:

```text
Gemini RX burst
      ↓
RX processing
      ↓
audio ring/queue
      ↓
audio_playback
      ↓
CPU1 starvation / IDLE1 WDT
```

Referensi ESP-IDF menjelaskan bahwa TWDT terutama memantau Idle Task untuk mendeteksi starvation CPU, dan task harus memberi kesempatan scheduler melalui blocking/yielding. `i2s_channel_write()` sendiri adalah fungsi blocking dengan batas timeout. citeturn0search0turn0search1

### Keputusan

Jangan mengubah baseline I2S v6.1.5.

Jangan mengubah pin INMP441/MAX98357A.

Jangan mengubah format PCM16 24 kHz speaker.

Fokus berikutnya adalah scheduling task playback dan backpressure audio.

**Status: HARDWARE TEST BERHASIL SEBAGIAN / GEMINI AUDIO BERHASIL / WDT DAN RX PRESSURE MASIH ADA**

---

## v7.1.2 — Audio Playback CPU Affinity Fix — August 21, 2026

### Dasar perubahan

Pada v7.1.1, Task WDT secara jelas menunjuk:

```text
IDLE1 (CPU 1)
CPU 1: audio_playback
```

Source code dicek kembali setelah `CHANGELOG_PROJECT.md` dibaca terlebih dahulu.

`audio_playback` sebelumnya dibuat dengan:

```cpp
xTaskCreate(..., "audio_playback", ..., 7, ...)
```

Artinya task tidak mempunyai CPU affinity dan dapat dijadwalkan pada CPU1. Karena CPU1 Idle Task yang dipantau WDT mengalami starvation, perbaikan v7.1.2 dibuat sekecil mungkin.

### Perubahan

`audio_playback` sekarang dibuat pinned ke **CPU0**:

```cpp
xTaskCreatePinnedToCore(
    audio_playback_task,
    "audio_playback",
    4096,
    NULL,
    7,
    &audio_playback_task_handle,
    0
);
```

Ditambahkan logging core saat task berjalan:

```text
Audio playback task: ... core=0
```

### Kenapa hanya ini?

Karena I2S v6.1.5 sudah menjadi baseline hardware yang terbukti.

Perubahan v7.1.2 sengaja **tidak** menyentuh:

- konfigurasi I2S
- DMA I2S
- INMP441
- MAX98357A
- sample rate mic 16 kHz
- sample rate speaker 24 kHz
- PCM16
- ring buffer 49152 byte
- prebuffer 16384 byte
- WebSocket JSON
- Gemini setup
- MIC TX gate

Ini adalah **scheduling fix**, bukan audio-format fix.

### Referensi teknis

ESP-IDF menggunakan scheduler per-core pada ESP32-S3. Task yang tidak dipin dapat berjalan pada core yang kompatibel, sedangkan Idle Task dibuat terpisah untuk masing-masing core. Pinning task memungkinkan kita mengisolasi beban playback dari CPU1 yang sebelumnya mengalami IDLE1 starvation. citeturn1search0turn1search1

`i2s_channel_write()` tetap blocking sampai data terkirim atau timeout tercapai, sehingga task affinity dan scheduling tetap penting untuk mencegah starvation. citeturn0search1

### Target hardware test v7.1.2

```text
CPU1 IDLE WDT       = TIDAK MUNCUL
CPU1 audio_playback = TIDAK MUNCUL
WebSocket Gemini    = CONNECTED
setupComplete        = OK
Audio RX             = BERJALAN
Audio playback       = BERJALAN
AUDIO PLAYBACK COMPLETE = OK
pending              = 0
```

### Status

**v7.1.2 = CODE UPDATED / MENUNGGU BUILD + HARDWARE TEST.**

Baseline hardware tetap **v6.1.5**.

Baseline proyek sebelum v7.1.2 tetap **v7.0.36** sampai hasil hardware v7.1.2 terbukti stabil.

---

## v7.1.2 — Build Failure Correction — August 21, 2026

### Masalah build

Build v7.1.2 gagal pada tahap linker karena implementasi dua fungsi audio tidak sengaja hilang dari `websocket_audio.cpp`:

```text
begin_audio_turn()
queue_audio_pcm()
```

Fungsi tersebut masih digunakan oleh jalur WebSocket RX/JSON sehingga menghasilkan `undefined reference` saat linking.

### Perbaikan

Commit berikut memperbaiki `websocket_audio.cpp` dengan:

- mengembalikan `begin_audio_turn()`;
- mengembalikan `queue_audio_pcm()`;
- mempertahankan `audio_playback` pinned ke CPU0;
- mempertahankan logging `core=0`;
- tidak mengubah konfigurasi I2S, pin, sample rate, PCM16, ring buffer, Gemini setup, atau MIC TX gate.

### Commit Gemini Project

```text
c5071739e014f09995e2ba1e55ceaa44c1234b75
```

Message:

```text
fix(v7.1.2): restore audio queue functions
```

### Status

**v7.1.2 correction = CODE FIXED / MENUNGGU BUILD ULANG.**

Jangan flash sebelum GitHub Actions menghasilkan build hijau.

Baseline hardware tetap **v6.1.5**.
Baseline proyek terbukti terakhir tetap **v7.0.36** sampai v7.1.2 lolos build + hardware test.
