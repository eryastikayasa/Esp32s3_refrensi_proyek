# ESP32-S3 Voice Commander — Project Changelog

Dokumen ini mencatat milestone, baseline, perubahan arsitektur penting, hasil hardware test, dan keputusan tuning agar konteks proyek tidak hilang saat pekerjaan dilanjutkan oleh AI.

> Jangan menghapus sejarah yang masih relevan. Tambahkan milestone baru.
> Gunakan bahasa sehari-hari/sederhana saat menjelaskan perubahan.
> Setiap perubahan firmware wajib dicatat di dokumen ini agar AI tidak kembali ke awal.

---

## v7.1.2 — Audio Playback WDT Investigation — August 20, 2026

### Hasil hardware

Firmware berhasil:

- Boot ESP32-S3.
- Inisialisasi audio I2S v6.1.5.
- Mic 16 kHz dan speaker 24 kHz terinisialisasi.
- Wi-Fi GOT_IP.
- NTP berhasil.
- DNS `generativelanguage.googleapis.com` berhasil.
- TCP 443 berhasil.

Namun setelah `audio_playback` dibuat, Task WDT muncul sebelum WebSocket Gemini sempat terhubung:

```text
WS_AUDIO: Audio playback task ... core=0

task_wdt: - IDLE0
CPU 0: audio_playback
```

WDT berulang sekitar setiap 5 detik. Pada titik ini belum ada `WS_EVENT: WebSocket TERHUBUNG ke Gemini!`, sehingga masalah ini dipisahkan dari TLS/Gemini/RX audio.

### Temuan source

`audio_playback_task` saat ini dibuat pinned ke CPU0 dengan priority 7. Task melakukan polling `xStreamBufferReceive()` dan kemudian memanggil `audio_write_speaker()` ketika ada PCM.

Source juga masih menggunakan konfigurasi I2S v6.1.5 yang dikunci. Tidak ada perubahan hardware/I2S yang dilakukan berdasarkan hasil ini.

### Keputusan

Jangan menambah audio control, UART, servo, atau fitur JSON baru sebelum WDT playback selesai.

Tes berikutnya harus berupa **diagnostic patch kecil pada scheduler/task playback**, bukan perubahan format I2S, DMA, Gemini, atau WebSocket.

Target tes:

```text
audio_playback tidak boleh membuat IDLE0 starvation
WebSocket tetap bisa mulai normal
I2S v6.1.5 tetap LOCKED
```

**Status: INVESTIGATION / BELUM BASELINE**

---

## v6.1.3 — I2S Technical Test Baseline

- Baseline test tone 1 kHz untuk memisahkan masalah I2S dari Gemini/WebSocket.
- Digunakan untuk validasi jalur TX I2S dan MAX98357A.

**Status:** historical baseline.

---

## v6.1.5 — Audio Baseline LOCKED

- INMP441 microphone terbukti bekerja.
- MAX98357A speaker terbukti bekerja.
- Audio/I2S ditetapkan sebagai baseline teknis.
- Mic 16 kHz, speaker 24 kHz.

**Keputusan:** jangan mengubah konfigurasi I2S v6.1.5 selama investigasi WebSocket/audio buffering.

**Status: LOCKED / REFERENCE BASELINE**

---

## v7 — Language / Speech Development

Pengembangan voice assistant dan speech dimulai di atas baseline audio v6.1.5.

---

## v7.0.1 — Indonesian Language / Speech

Penyesuaian language/output speech ke Bahasa Indonesia tanpa mengubah baseline audio.

---

## v7.0.2 — WebSocket Crash Investigation

Fokus pada crash lifecycle WebSocket, termasuk jalur:

```text
ws_poll_read()
 -> esp_transport_poll_read()
 -> esp_websocket_client_task()
```

---

## v7.0.3 — WebSocket Lifecycle / Race Mitigation

- Setup Gemini dipindahkan ke task terpisah.
- Diperkenalkan connection generation.
- Setup task dari koneksi lama dapat diinvalidasi.
- Connection check dilakukan sebelum send.
- Send diberi timeout.

---

## v7.0.4 — WebSocket Crash Resolution Focus

Fokus pada penyelesaian crash setelah v7.0.3, termasuk lifecycle Wi-Fi/main/WebSocket dan analisis `addr2line` pada SSL/WebSocket initialization.

---

## TLS / Certificate Troubleshooting — Historical

Pernah terjadi kegagalan certificate bundle terhadap `generativelanguage.googleapis.com:443`:

```text
No matching trusted root certificate found
Failed to verify certificate
mbedtls_ssl_handshake returned -0x3000
```

Kemudian DNS dan TCP 443 terbukti OK dan WSS Gemini berhasil.

---

## Audio Tuning — Volume Processing Disabled — August 20, 2026

- Volume processing Gemini dinonaktifkan.
- Volume dipaksa 100%.
- PCM Gemini dikirim langsung ke ring buffer.
- I2S, INMP441, MAX98357A, dan baseline v6.1.5 tidak diubah.

Current audio buffering:

```text
ring=32768
prebuffer=12288
target=48000 B/s
```

**Status:** volume processing tetap OFF selama stability investigation.

---

## OLED OFF — Audio Isolation Test — August 20, 2026

OLED/I2C dimatikan untuk isolasi audio.

Hasil: timeout I2C sebelumnya tidak muncul. Wi-Fi, DNS, TCP, WSS Gemini, setupComplete, dan RX audio pernah berhasil pada pengujian sebelumnya.

**Status:** OLED tetap OFF selama fase audio/RX isolation.

---

## v7.0.27 — WebSocket Audio TX Lock Tuning

- Timeout audio TX diturunkan dari 5000 ms ke 250 ms.
- Frame audio yang gagal dibuang tanpa langsung mematikan koneksi.
- Logging TX diperjelas.

**Hasil:** masalah PING/ws-client lock masih ada.

---

## v7.0.28 — WebSocket PING Diagnostic

- WebSocket keep-alive PING dimatikan.
- Jalur I2S/audio tidak diubah.

**Status:** superseded by v7.0.29.

---

## v7.0.29 — WebSocket Audio TX Diagnostic

Gemini berhasil connect, setupComplete, dan RX audio berjalan. Kemudian muncul:

```text
E transport_ws: Error transport_poll_write(0)
E websocket_client: esp_transport_write() returned 0
E WS_EVENT: WebSocket Error!
```

Masalah dipersempit ke realtime WebSocket/TLS TX, bukan DNS/TCP/setup/RX dasar.

**Status: superseded by v7.0.30.**

---

## v7.0.30 — WebSocket Audio TX Write Retry Tuning — August 20, 2026

### Masalah sebelum v7.0.30

Hasil v7.0.29 menunjukkan WebSocket sudah berhasil tersambung ke Gemini dan setup berhasil, tetapi saat jalur audio TX aktif koneksi bisa gagal pada:

```text
transport_poll_write(0)
esp_transport_write() returned 0
WS_EVENT: WebSocket Error!
```

Sebelumnya timeout audio TX masih terlalu pendek untuk kondisi realtime WebSocket/TLS. Karena itu fokus v7.0.30 adalah **melonggarkan jalur write audio**, bukan mengubah I2S atau hardware audio.

### Perubahan yang dilakukan

Pada WebSocket TX/audio write:

```text
Audio TX timeout : 150 ms -> 1000 ms
Retry            : 1 kali
Retry delay      : 30 ms
```

Retry hanya boleh berjalan jika **connection generation masih valid**, supaya worker tidak meneruskan pengiriman ke koneksi lama yang sudah putus.

Network timeout juga dinaikkan:

```text
10 s -> 15 s
```

WebSocket keep-alive PING tetap **OFF**.

### Yang tidak diubah

- I2S v6.1.5 tetap LOCKED.
- INMP441 tetap.
- MAX98357A tetap.
- Mic tetap 16 kHz.
- Speaker tetap 24 kHz.
- Audio ring tetap 32768 byte.
- Prebuffer tetap 12288 byte.
- Volume processing tetap OFF.
- OLED tetap OFF.

### Hasil hardware v7.0.30

Ini menjadi hasil penting yang dipakai sebagai referensi tuning berikutnya. Full Gemini turn berhasil sampai audio selesai:

```text
WS_EVENT: WebSocket TERHUBUNG ke Gemini!
WS_JSON: Gemini setupComplete: SESI SIAP
WS_JSON: Gemini: GENERATION COMPLETE
WS_JSON: Gemini: TURN COMPLETE - menunggu audio drain
WS_AUDIO: AUDIO PLAYBACK COMPLETE: received=375834 queued=373488 played=373488 pending=0 dropped=1990 balance=356
WS_JSON: AUDIO SUMMARY: chunks=370 received=373488 played=373488 pending=0 write_calls=183
```

Pada hasil ini tidak terjadi `transport_poll_write(0)` atau disconnect sebelum turn selesai.

### Masalah yang masih tersisa

Walaupun TX write sudah jauh lebih stabil, RX masih menunjukkan tekanan fragment/queue:

```text
dropped_frag=12 seq_err=10 buffer_drop=2
dropped_frag=34 seq_err=31 buffer_drop=3
dropped_frag=49 seq_err=45 buffer_drop=4
```

Audio queue juga sempat drop:

```text
38 byte
998 byte
774 byte
180 byte
```

Ring buffer sempat hampir penuh:

```text
pending=31510/32768
```

Satu playback underrun masih muncul, tetapi audio akhirnya berhasil drain sampai zero.

### Kesimpulan v7.0.30

**TX WRITE PATH terbukti membaik.** Ini adalah baseline hasil hardware yang penting dan jangan dibuang.

Setelah v7.0.30, fokus investigasi dipindahkan ke **RX fragment/slot pressure dan audio queue backpressure**, bukan kembali mengubah I2S.

**Status: HASIL HARDWARE BERHASIL / BASELINE TX TERBUKTI**

---

## v7.0.31 — RX Persistent Slot Headroom Tuning — August 20, 2026

### Dasar tuning

v7.0.30 menunjukkan:

```text
queue_hwm=4
buffer_drop=4
```

RX queue dan persistent slot pool kemudian dinaikkan:

```text
WS_RX_SLOT_COUNT 6 -> 8
WS_RX_QUEUE_LENGTH tetap mengikuti WS_RX_SLOT_COUNT -> 8
```

Tidak ada perubahan pada I2S, audio ring, prebuffer, volume, OLED, atau PING.

### Hasil hardware v7.0.31

Firmware berhasil boot, Wi-Fi/DNS/TCP/WSS dan `setupComplete` berhasil. Namun koneksi kembali putus sebelum burst audio RX dimulai:

```text
WS_EVENT: WebSocket TERHUBUNG ke Gemini!
WS_JSON: Gemini setupComplete: SESI SIAP
E transport_ws: Error transport_poll_write(0)
E websocket_client: esp_websocket_client_write() returned 0
W WS_EVENT: Connection generation invalidated
W WS_EVENT: WebSocket TERPUTUS dari Gemini
```

Karena putus terjadi sebelum audio RX dimulai, perubahan slot RX v7.0.31 belum bisa dinilai secara adil. Masalah TX muncul lebih dulu.

**Status:** RX slot tuning dipertahankan, tetapi validasi ditunda sampai TX stabil kembali.

---

## v7.0.32 — Restore Proven WebSocket TX Timing + RX Slot v7.0.31 — August 20, 2026

### Perubahan firmware

Pada WebSocket TX:

```text
AUDIO_SEND_TIMEOUT      = 1000 ms
AUDIO_SEND_RETRIES      = 1
AUDIO_SEND_RETRY_DELAY  = 30 ms
network_timeout_ms      = 15000
PING                    = OFF
PCM_SEND_CHUNK          = 1600 byte
```

RX slot v7.0.31 tetap dipertahankan:

```text
WS_RX_SLOT_COUNT = 8
WS_RX_QUEUE_LENGTH = 8
```

### Kenapa ini dilakukan

Log v7.0.31 kembali menunjukkan `transport_poll_write(0)` sebelum audio RX. Jadi timing TX v7.0.30 yang sudah terbukti berhasil dikembalikan.

### Hasil hardware v7.0.32

TX tidak langsung gagal seperti v7.0.31, tetapi muncul masalah baru yang lebih jelas:

```text
task_wdt: - IDLE1
CPU 1: audio_playback
```

Kemudian WebSocket mengalami lock timeout dan akhirnya:

```text
transport_poll_write(0)
esp_transport_write() returned 0
```

Log juga menunjukkan:

```text
TX audio write timeout/fail: attempt=1 sent=0 expected=2209 pcm_chunk=1600 offset=1600/3200 timeout=1000ms
TX audio command dihentikan: sent_pcm=1600/3200
```

**Kesimpulan:** v7.0.32 berhasil membawa kita melewati tahap TX awal, tetapi menemukan bottleneck yang lebih spesifik: `audio_playback` terlalu lama memblokir CPU1 dan menyebabkan watchdog/lock contention.

**Status: superseded by v7.0.33.**

---

## v7.0.33 — Audio Playback WDT Mitigation — August 20, 2026

Fokus v7.0.33 adalah masalah yang ditemukan langsung pada hardware v7.0.32: `audio_playback` memblokir CPU1 terlalu lama sehingga `IDLE1` gagal mendapat waktu dan Task WDT aktif.

Perubahan utama di `audio_hal.cpp`:

```text
I2S write sebelumnya : portMAX_DELAY
I2S write sekarang   : timeout 50 ms
Chunk write          : 512 PCM samples
```

Jika I2S timeout/gagal atau tidak menulis data, jalur speaker tidak boleh terus memblokir CPU. Fungsi memberi kesempatan scheduler dengan `vTaskDelay(1)` lalu keluar.

**Yang tetap dikunci:**

- I2S v6.1.5.
- Mic 16 kHz.
- Speaker 24 kHz.
- INMP441.
- MAX98357A.
- Audio ring 32 KB.
- Prebuffer 12 KB.
- Volume processing OFF.
- OLED OFF.
- PING OFF.

**Target hardware test:**

```text
Tidak ada task_wdt IDLE1
Tidak ada CPU 1: audio_playback WDT
WebSocket tetap connected
Audio RX tetap berjalan
AUDIO PLAYBACK COMPLETE
```

**Status: superseded by v7.0.34.**

---

## v7.0.34 — Audio Playback Scheduler Yield — August 20, 2026

### Masalah yang ditargetkan

v7.0.33 sudah membatasi `i2s_channel_write()` menjadi timeout 50 ms dan hanya melakukan `vTaskDelay(1)` ketika write gagal.

Namun ada celah: jika I2S terus berhasil menulis, `audio_write_speaker()` bisa menjalankan banyak chunk 512 sample berturut-turut tanpa memberi kesempatan scheduler. Ini masih berpotensi membuat `audio_playback` terlalu lama aktif di CPU1 dan kembali menekan `IDLE1`/Task WDT.

### Perubahan v7.0.34

Tidak mengubah format I2S atau hardware audio.

Setelah setiap chunk I2S yang berhasil ditulis, sekarang dilakukan:

```text
I2S write sukses
      ↓
vTaskDelay(1)
      ↓
lanjut ke chunk berikutnya
```

Konfigurasi yang dipertahankan dari v7.0.33:

```text
I2S_WRITE_SAMPLES     = 512
I2S_WRITE_TIMEOUT_MS  = 50
```

Jadi v7.0.34 adalah **perbaikan scheduling**, bukan perubahan timing/data format I2S.

### Hasil hardware v7.0.34

Pada pengujian ini perangkat **belum mencoba berbicara**.

Yang berhasil:

- Wi-Fi/DNS/TCP 443 OK.
- WebSocket/TLS OK.
- Gemini `setupComplete` OK.
- Session resumption tetap OK.
- RX message berjalan sampai 21 message.
- Tidak muncul `task_wdt` pada `audio_playback`.

Yang masih gagal:

```text
transport_poll_write(0)
esp_websocket_client: esp_transport_write() returned 0
WS_EVENT: WebSocket Error!
WebSocket TERPUTUS dari Gemini
```

Setelah disconnect baru muncul timeout TX audio. Jadi timeout audio tersebut dianggap **efek setelah koneksi mati**, bukan penyebab pertama.

**Status: superseded by v7.0.35.**

---

## v7.0.35 — Idle Microphone TX Gate — August 20, 2026

### Alasan perubahan

Log v7.0.34 menunjukkan koneksi mati pada `transport_poll_write(0)` walaupun user belum berbicara.

Source code kemudian dicek. `audio_task` ternyata tetap membaca INMP441 dan mengirim frame PCM 3200 byte ke TX queue setelah Gemini `setupComplete`, sehingga perangkat dapat terus melakukan audio TX saat kondisi idle/senyap.

Ini menjadi jalur yang perlu diisolasi sebelum mengubah WebSocket transport lebih jauh.

### Perubahan

Di `main/main.cpp` ditambahkan gate aktivitas PCM16:

```text
Frame mic 3200 byte
       ↓
cek aktivitas sample
       ↓
senyap → tidak dikirim
aktif → kirim seperti biasa
```

Parameter diagnosis:

```text
SILENCE_THRESHOLD = 700
MIN_ACTIVE_SAMPLES = 8
```

Frame senyap hanya dibuang dari TX. I2S/audio hardware tidak disentuh.

Ditambahkan log periodik:

```text
V7.0.35 MIC TX gate: silent frames dropped=...
```

### Keputusan

Gate ini dipakai sebagai **diagnostic isolation**, bukan fitur final voice activity detection.

**Status: superseded by v7.0.36.**

---

## v7.0.36 — Idle Microphone TX Gate Validation — August 20, 2026

### Hasil hardware

Gate frame mic senyap tetap berhasil bekerja. Log menunjukkan frame senyap terus dibuang:

```text
V7.0.36 MIC TX gate: silent frames dropped=...
```

Gemini berhasil setup dan menghasilkan audio sampai `TURN COMPLETE`. Audio berhasil drain sampai `pending=0`.

Namun masih terlihat tekanan pada RX/audio buffer:

```text
dropped_frag=154
seq_err=144
buffer_drop=10
```

dan audio drop kumulatif:

```text
dropped=7012 byte
```

Satu `AUDIO PLAYBACK UNDERRUN` juga masih terjadi.

### Kesimpulan

v7.0.36 membuktikan gate mic idle membantu mengisolasi jalur TX, tetapi **belum menjadi baseline audio final** karena RX fragment pressure, queue drop, dan underrun masih ada.

**Status: HASIL HARDWARE / BELUM BASELINE FINAL**

---
