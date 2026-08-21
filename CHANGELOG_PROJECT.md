# ESP32-S3 Voice Commander — Project Changelog

Dokumen ini mencatat milestone, baseline, perubahan arsitektur penting, hasil hardware test, dan keputusan tuning agar konteks proyek tidak hilang saat pekerjaan dilanjutkan oleh AI.

> Jangan menghapus sejarah yang masih relevan. Tambahkan milestone baru.
> Gunakan bahasa sehari-hari/sederhana saat menjelaskan perubahan.
> Setiap perubahan firmware wajib dicatat di dokumen ini agar AI tidak kembali ke awal.

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
WS_JSON: AUDIO SUMMARY: chunks=370 received=375834 played=373488 pending=0 write_calls=183
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

### Yang tidak diubah

- I2S v6.1.5 tetap LOCKED.
- INMP441 tetap.
- MAX98357A tetap.
- Mic 16 kHz.
- Speaker 24 kHz.
- I2S/DMA tidak diubah.
- Audio ring 32768 byte.
- Prebuffer 12288 byte.
- Volume processing OFF.
- OLED OFF.
- PING OFF.
- TX timeout 1000 ms.
- Retry 1x / 30 ms.
- RX slot 8 / queue 8.

### Urutan hardware test

**Test 1 — idle:** jangan bicara setelah `setupComplete`.

Target:

```text
MIC TX gate: silent frames dropped=...
```

tetapi tidak ada disconnect/`transport_poll_write(0)`.

**Test 2 — bicara:** setelah idle test stabil, bicara normal dan lihat apakah frame aktif terkirim serta Gemini merespons.

### Keputusan berikutnya

- Idle stabil → fokus berikutnya pada TX audio aktif.
- Idle tetap disconnect → fokus pada write non-audio/internal WebSocket lifecycle.
- Jangan mengubah baseline I2S berdasarkan hasil ini saja.

**Status: superseded by v7.0.35 hardware result below.**

---

## v7.0.35 — Hardware Result: 30-Second Idle Then “Halo Gemini” — August 20, 2026

### Kondisi test

Perangkat didiamkan sekitar 30 detik setelah `Gemini setupComplete`, kemudian user berbicara: **“halo gemini”**.

### Hasil idle 30 detik

MIC TX gate bekerja. Selama idle muncul berulang kali:

```text
V7.0.35 MIC TX gate: silent frames dropped=...
```

Tidak terjadi disconnect WebSocket selama periode idle tersebut.

Ini adalah hasil penting: **masalah disconnect v7.0.34 saat perangkat diam tidak terulang pada v7.0.35.**

### Hasil saat Gemini merespons

Gemini mengirim audio dan jalur playback berjalan:

```text
WS_JSON: AUDIO GEMINI: 4800 byte -> AUDIO BUFFER
WS_JSON: AUDIO GEMINI: 1920 byte -> AUDIO BUFFER
WS_AUDIO: AUDIO FLOW: pending=10426/32768 received=12480 queued=12474 played=2048 dropped=0
WS_JSON: Gemini: GENERATION COMPLETE
```

Playback kemudian selesai:

```text
WS_AUDIO: AUDIO PLAYBACK COMPLETE: received=93105 queued=89984 played=89984 pending=0 dropped=3036 balance=85
WS_JSON: Gemini: TURN COMPLETE - menunggu audio drain
WS_JSON: AUDIO SUMMARY: chunks=90 received=93105 played=89984 pending=0 write_calls=44
```

### Masalah yang ditemukan

Masih ada tekanan pada RX/audio queue:

```text
Audio PCM drop terukur: 514 byte
Audio PCM drop terukur: 1018 byte
Audio PCM drop terukur: 494 byte
Audio PCM drop terukur: 1010 byte

dropped_frag=15
seq_err=12
buffer_drop=3
```

Ring buffer sempat hampir penuh:

```text
pending=31742/32768
```

Kemudian terjadi:

```text
AUDIO PLAYBACK UNDERRUN: PCM buffer kosong di tengah turn
```

Total audio yang terukur:

```text
received = 93105 byte
queued   = 89984 byte
played   = 89984 byte
pending  = 0 byte
write_calls = 44
dropped  = 3036 byte
```

Jadi audio akhirnya **drain sampai habis**, tetapi terdapat drop PCM dan satu underrun di tengah turn.

### Kesimpulan v7.0.35

**Berhasil:**

- Idle 30 detik stabil.
- Silent microphone tidak lagi memenuhi TX queue.
- WebSocket tetap hidup saat idle.
- User dapat memicu respons dengan bicara.
- Gemini menghasilkan audio.
- Speaker memutar audio sampai selesai.
- Tidak terlihat `task_wdt`/reset pada hasil log ini.
- Baseline I2S v6.1.5 tetap tidak disentuh.

**Belum stabil:**

- RX fragment pressure masih ada.
- `seq_err` masih ada.
- Audio queue masih drop PCM.
- Ring buffer hampir penuh.
- Satu playback underrun masih terjadi.

### Keputusan untuk perbaikan berikutnya

Fokus berikutnya dipindahkan dari **idle WebSocket disconnect** ke **RX burst → audio queue → playback backpressure**.

Jangan mengubah I2S v6.1.5, INMP441, MAX98357A, atau format 16 kHz mic / 24 kHz speaker hanya karena hasil ini.

**Status: HARDWARE TEST BERHASIL SEBAGIAN / IDLE GATE TERBUKTI / AUDIO BUFFERING MASIH PERLU DIPERBAIKI**

---

## v7.0.36 — Audio RX Buffer Headroom Tuning — August 20, 2026

### Dasar perubahan

Hasil hardware v7.0.35 menunjukkan idle sudah stabil dan respons Gemini berhasil, tetapi saat burst audio RX terjadi tekanan pada audio queue:

```text
Audio PCM drop terukur: 514 byte
Audio PCM drop terukur: 1018 byte
Audio PCM drop terukur: 494 byte
Audio PCM drop terukur: 1010 byte

dropped_frag=15
seq_err=12
buffer_drop=3
pending=31742/32768
AUDIO PLAYBACK UNDERRUN
```

Karena ring 32 KB hampir penuh dan masih terjadi drop/underrun, v7.0.36 difokuskan pada **memberi headroom lebih besar pada buffering audio RX**.

### Perubahan v7.0.36

Buffer audio dinaikkan:

```text
ring      : 32768 -> 49152 byte
prebuffer : 12288 -> 16384 byte
```

Waktu tunggu pengisian ring pada jalur playback juga dilonggarkan:

```text
5 ms -> 10 ms
```

Tujuannya memberi waktu agar burst PCM dari Gemini terkumpul lebih banyak sebelum playback mengejar data berikutnya.

### Yang tidak diubah

- I2S v6.1.5 tetap LOCKED.
- INMP441 tetap.
- MAX98357A tetap.
- Mic tetap 16 kHz.
- Speaker tetap 24 kHz.
- Format PCM tetap PCM16 mono.
- I2S/DMA tidak diubah.
- MIC TX gate v7.0.35 tetap.
- RX slot tetap 8.
- RX queue tetap 8.
- Volume processing tetap OFF.
- OLED tetap OFF.
- PING tetap OFF.
- TX timeout tetap 1000 ms.
- Retry tetap 1x dengan delay 30 ms.

### Hasil hardware v7.0.36

Firmware berhasil boot dan seluruh jalur dasar berhasil:

```text
Wi-Fi GOT_IP - network READY
DNS RESULT: OK
TCP RESULT: OK
NETWORK BASIC TEST = OK
WebSocket TERHUBUNG ke Gemini!
Gemini setupComplete: SESI SIAP
```

Audio Gemini berhasil diterima dan playback berhasil berjalan sampai drain selesai:

```text
WS_AUDIO: Audio playback task: 24kHz PCM16 mono, ring=49152, prebuffer=16384
WS_AUDIO: Audio ring buffer siap: 49152 byte, prebuffer=16384, target=48000 B/s
WS_JSON: AUDIO GEMINI: 2 byte -> AUDIO BUFFER
WS_AUDIO: AUDIO FLOW: pending=16302/49152 received=18366 queued=18350 played=2048 dropped=0
WS_AUDIO: AUDIO FLOW: pending=18810/49152 received=70080 queued=70010 played=51200 dropped=0
WS_JSON: AUDIO GEMINI: 1920 byte -> AUDIO BUFFER
WS_AUDIO: AUDIO FLOW: pending=24386/49152 received=124863 queued=124738 played=100352 dropped=0
WS_AUDIO: AUDIO FLOW: pending=49130/49152 received=198831 queued=198634 played=149504 dropped=0
```

Pada titik ini ring buffer hampir penuh. Terjadi satu drop PCM yang terukur:

```text
Audio PCM drop terukur: 988 byte
AUDIO STREAM: queue PCM gagal len=1023
```

Statistik RX saat burst audio menunjukkan tekanan fragment/sequence:

```text
fragments=254 dropped_frag=2 messages=56 queue_hwm=1 seq_err=1 buffer_drop=1
```

Setelah Gemini selesai mengirim audio:

```text
Gemini: GENERATION COMPLETE
Gemini: TURN COMPLETE - menunggu audio drain
```

Audio kemudian berhasil drain sampai nol:

```text
AUDIO SUMMARY: chunks=210 received=205992 played=178176 pending=24576 write_calls=87
AUDIO FLOW: pending=6144/49152 received=205992 queued=204800 played=198656 dropped=988
AUDIO PLAYBACK COMPLETE: received=205992 queued=204800 played=204800 pending=0 dropped=988 balance=204
```

### Hasil penting v7.0.36

**Berhasil:**

- Wi-Fi stabil.
- DNS OK.
- TCP 443 OK.
- TLS/WSS Gemini OK.
- `setupComplete` OK.
- Session resumption tetap berjalan.
- MIC TX gate tetap bekerja saat idle.
- Gemini berhasil mengirim audio.
- Speaker berhasil memainkan audio sampai selesai.
- Ring 49 KB berhasil menyerap burst jauh lebih besar dibanding ring 32 KB.
- Tidak ada `task_wdt`/reset pada log ini.
- Audio akhirnya drain sampai `pending=0`.

**Masih ada masalah:**

- Ring buffer tetap sempat hampir penuh: `49130/49152` byte.
- Terjadi `988 byte` PCM drop.
- `queue PCM gagal len=1023`.
- RX mencatat `dropped_frag=2`, `seq_err=1`, `buffer_drop=1`.
- Jadi buffering membaik dan audio selesai, tetapi burst RX masih sedikit lebih cepat daripada kemampuan jalur queue/playback.

### Kesimpulan v7.0.36

v7.0.36 **berhasil memperbesar headroom audio dan membuat satu turn Gemini selesai sampai drain**, tetapi belum menghilangkan audio drop sepenuhnya.

Masalah sekarang semakin sempit: **RX burst → ring buffer hampir penuh → queue PCM gagal/drop sekitar 20 ms audio**.

Jangan kembali mengubah baseline I2S v6.1.5. Fokus tuning berikutnya tetap pada **RX/audio buffering dan backpressure**, bukan hardware I2S.

**Status: HARDWARE TEST BERHASIL / BUFFER HEADROOM MEMBAIK / DROP PCM MASIH ADA**

---

## v7.1.1 — Audio Control Module — August 20, 2026

### Tujuan

v7.0.36 sudah dijadikan baseline. v7.1.1 mulai membuat **jalur kontrol audio terpisah** supaya fitur volume/mute tidak langsung mengubah jalur I2S yang sudah stabil.

### Perubahan firmware

Dibuat modul baru:

```text
components/audio_control/
```

Modul ini menangani kontrol pada PCM16 sebelum data masuk ke jalur speaker:

```text
PCM16 Gemini
    ↓
audio_control
    ↓
PCM16 speaker
```

Fungsi kontrol yang dibuat:

```text
audio_control_set_volume(percent)
audio_control_get_volume()
audio_control_set_mute(muted)
audio_control_is_muted()
audio_control_process_pcm16(pcm, sample_count)
```

Perilaku awal:

- Volume default = 100%.
- Mute default = OFF.
- Jika volume 100% dan tidak mute, PCM **tidak disentuh**.
- Jika mute atau volume 0%, sample PCM dibuat 0.
- Jika volume di antara 1–99%, PCM16 dikalikan sesuai persentase.
- Hasil scaling dijaga tetap pada rentang PCM16 `-32768..32767`.

### Catatan stabilitas

Aturan penting v7.1.1:

```text
100% + unmuted
        ↓
PCM asli langsung lewat
```

Artinya mode default tidak menambahkan proses scaling ke audio yang sudah terbukti stabil pada v7.0.36.

I2S v6.1.5, INMP441, MAX98357A, format PCM, dan jalur WebSocket tidak diubah untuk fitur kontrol ini.

### Masalah build awal

Build GitHub Actions pertama gagal pada `audio_control.cpp` karena pemanggilan:

```cpp
std::clamp(scaled, -32768, 32767)
```

memiliki konflik tipe antara `int32_t` dan `int` pada toolchain ESP32-S3.

Perbaikan dilakukan menjadi clamp bertipe eksplisit:

```cpp
std::clamp<int32_t>(scaled, -32768, 32767)
```

Tidak ada perubahan pada jalur I2S untuk memperbaiki error ini. Ini murni perbaikan compile C++.

### Hasil CI

Setelah perbaikan `std::clamp`, GitHub Actions build **PASS / centang hijau**.

### Yang tidak dilakukan pada v7.1.1

- Belum flash ke ESP32-S3 pada saat pencatatan ini.
- Belum menyatakan v7.1.1 sebagai baseline baru.
- Tidak mengubah baseline v7.0.36.
- Tidak mengubah konfigurasi I2S v6.1.5.
- Tidak mengubah INMP441 atau MAX98357A.
- Tidak mengubah WebSocket JSON protocol.

### Status

# v7.1.1 — Hardware Test Result

Tanggal: 21 Agustus 2026

## Status

v7.1.1 **belum ditetapkan sebagai baseline**. Baseline aman tetap v7.0.36.

## Yang berhasil

- Wi-Fi berhasil mendapatkan IP.
- NTP berhasil sinkron dan tahun terbaca 2026.
- DNS `generativelanguage.googleapis.com` berhasil.
- TCP `generativelanguage.googleapis.com:443` berhasil.
- WebSocket/TLS Gemini berhasil terhubung.
- Gemini `setupComplete` berhasil.
- Session resumption berhasil dan handle tersimpan.
- Audio Gemini berhasil diterima dan masuk ke audio buffer.
- Audio playback berhasil sampai `AUDIO PLAYBACK COMPLETE`.
- Pada akhir turn: `received=344076`, `queued=336752`, `played=336752`, `pending=0`, `dropped=7012`.

## Masalah yang ditemukan

### 1. RX WebSocket mengalami fragment/sequence drop

Pada pengujian terlihat:

```text
fragments=525
 dropped_frag=154
seq_err=144
buffer_drop=10
```

Nilai tersebut menunjukkan jalur RX masih mengalami tekanan dan kehilangan fragment.

### 2. Audio PCM mengalami drop/backpressure

Beberapa kali muncul:

```text
Audio PCM drop terukur
AUDIO STREAM: queue PCM gagal
Gagal memasukkan audio ke ring buffer
```

Total audio drop pada akhir playback tercatat:

```text
dropped=7012
```

### 3. Playback underrun

Muncul:

```text
AUDIO PLAYBACK UNDERRUN: PCM buffer kosong di tengah turn
```

Namun audio kemudian berhasil menyelesaikan drain sampai:

```text
pending=0
played=336752
```

### 4. Task Watchdog masih muncul pada `audio_playback`

Dua kejadian WDT tercatat:

```text
task_wdt: Task watchdog got triggered
- IDLE1 (CPU 1)
CPU 1: audio_playback
```

Terjadi sekitar 37,2 detik dan 42,2 detik setelah boot.

## Kesimpulan

v7.1.1 membuktikan jalur utama Gemini Live dan playback audio **sudah berjalan end-to-end**, tetapi belum cukup stabil untuk dijadikan baseline baru.

Masalah berikutnya harus difokuskan pada:

1. RX WebSocket fragment/sequence handling.
2. Audio queue/ring-buffer backpressure.
3. `audio_playback` yang masih berpotensi menahan CPU1 dan memicu Task WDT.

## Yang tidak boleh diubah sembarangan

- Baseline I2S v6.1.5 tetap LOCKED.
- INMP441 tetap 16 kHz.
- MAX98357A tetap 24 kHz.
- Jangan kembali ke perubahan hardware/I2S untuk menangani masalah RX/WDT ini.
- OLED tetap OFF selama fase isolasi audio/RX.

## Keputusan versi

**v7.1.1 = TEST RESULT / NOT BASELINE**

**v7.0.36 = baseline yang tetap dipertahankan.**

Setiap perubahan berikutnya wajib melihat catatan referensi ini terlebih dahulu sebelum menyentuh repository firmware utama.

Baseline tetap **v7.0.36** sampai v7.1.1 selesai diuji di hardware.

---

## Project Rule — Jangan Kembali ke Awal

Setiap tuning berikutnya wajib:

1. Baca `CHANGELOG_PROJECT.md` terlebih dahulu.
2. Pertahankan baseline v6.1.5.
3. Jangan mengulang eksperimen yang sudah terbukti atau sudah ditolak.
4. Catat versi baru, alasan perubahan, hasil log, dan keputusan berikutnya.
5. Gunakan bahasa sehari-hari/sederhana agar konteks mudah dipahami.
6. Setelah setiap perubahan firmware, update changelog ini.
