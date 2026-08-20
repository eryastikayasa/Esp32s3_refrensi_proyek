# ESP32-S3 Voice Commander — Project Changelog

Dokumen ini mencatat milestone, baseline, perubahan arsitektur penting, dan hasil debugging yang relevan untuk melanjutkan proyek.

> Ini bukan daftar setiap commit. Hanya perubahan dan keputusan teknis yang penting untuk konteks AI.

---

## v6.1.3 — I2S Technical Test Baseline

- Dibuat baseline teknis untuk pengujian jalur speaker I2S secara deterministik.
- Digunakan test tone 1 kHz untuk memisahkan masalah I2S dari masalah Gemini/WebSocket.
- Tujuan utama: memastikan jalur TX I2S dan MAX98357A dapat diuji tanpa ketergantungan pada koneksi Gemini.

### Status
Baseline historis.

---

## v6.1.5 — Audio Baseline LOCKED

- Microphone INMP441 terbukti bekerja.
- Output speaker melalui MAX98357A terbukti bekerja.
- Konfigurasi audio/I2S pada tahap ini ditetapkan sebagai **technical audio baseline**.

### Keputusan penting
**v6.1.5 tidak boleh diubah sembarangan.**

Perubahan audio berikutnya harus mempertahankan fungsi yang telah terbukti atau disertai bukti pengujian yang cukup untuk menetapkan baseline baru.

### Status
**LOCKED / REFERENCE BASELINE**

---

## v7 — Language / Speech Development

- Pengembangan fitur bahasa dan output speech dimulai di atas baseline audio.
- Fokus bergeser dari validasi hardware/audio dasar menuju integrasi voice assistant.

### Status
Milestone historis.

---

## v7.0.1 — Indonesian Language / Speech

- Penyesuaian bahasa/output speech untuk penggunaan Bahasa Indonesia.
- Audio baseline tetap dipertahankan.

### Status
Milestone historis.

---

## v7.0.2 — WebSocket Crash Investigation

Fokus utama berpindah ke crash pada lifecycle WebSocket.

Crash path yang dianalisis mencakup:

```text
ws_poll_read()
    -> esp_transport_poll_read()
        -> esp_websocket_client_task()
```

### Status
Dilanjutkan ke v7.0.3.

---

## v7.0.3 — WebSocket Lifecycle / Race Mitigation

Dilakukan mitigasi terhadap kemungkinan race condition dan lifecycle WebSocket:

- Setup Gemini dipindahkan ke task terpisah.
- Diperkenalkan **connection generation** untuk membedakan setup dari koneksi lama dan koneksi baru.
- Setup task dari koneksi lama dapat dibatalkan/diinvalidasi ketika generasi koneksi berubah.
- Kondisi koneksi diperiksa sebelum melakukan send.
- WebSocket send diberikan timeout agar tidak menunggu tanpa batas.

### Status
Dilanjutkan ke v7.0.4.

---

## v7.0.4 — WebSocket Crash Resolution Focus

- Fokus pada penyelesaian crash setelah mitigasi v7.0.3.
- Area yang disentuh mencakup lifecycle Wi-Fi, main application flow, dan WebSocket manager.
- Investigasi crash juga menggunakan hasil `addr2line` yang mengarah ke jalur SSL/WebSocket initialization.

### Status
Milestone debugging historis; runtime terbaru menunjukkan koneksi Gemini sudah berhasil.

---

## TLS / Certificate Troubleshooting — Historical Milestone

Sebelum WebSocket berhasil terhubung, proyek mengalami kegagalan TLS terhadap `generativelanguage.googleapis.com:443`.

Log penting yang pernah muncul:

```text
esp-x509-crt-bundle: No matching trusted root certificate found
esp-x509-crt-bundle: Failed to verify certificate
esp-tls-mbedtls: mbedtls_ssl_handshake returned -0x3000
esp-tls: Failed to open new connection
transport_ws: Error connecting to host generativelanguage.googleapis.com:443
```

### Kesimpulan
DNS dan TCP kemudian terbukti berhasil, dan Gemini WebSocket akhirnya berhasil terhubung. TLS certificate verification bukan blocker utama pada fase sekarang.

---

## WebSocket Connection Success — August 20, 2026

Runtime log terbaru menunjukkan:

```text
WS_EVENT: WebSocket TERHUBUNG ke Gemini!
WS_EVENT: Connection generation=1
DISPLAY: [OLED STATUS]: AI Terhubung...
```

### Status
**VERIFIED IN RUNTIME LOG**

---

## Audio Tuning — Volume Processing Disabled — August 20, 2026

Pada commit firmware `9a5026d4` dilakukan perubahan khusus untuk uji kestabilan audio.

### Perubahan
- Pemrosesan volume Gemini dinonaktifkan sementara.
- Volume dipaksa 100%.
- Scaling PCM sample-per-sample pada jalur `queue_audio_pcm()` dilewati.
- PCM Gemini langsung dikirim ke ring buffer.

### Yang tidak diubah
- I2S.
- MAX98357A.
- Ring buffer 32 KB.
- Prebuffer 12 KB.
- Playback wait 5 ms.
- Playback task priority 7.
- Baseline audio v6.1.5.

### Status
**VOLUME PROCESSING REMAINS DISABLED**

---

## OLED OFF — Audio Isolation Test — August 20, 2026

OLED dan akses I2C OLED dinonaktifkan sementara untuk mengisolasi masalah audio/WebSocket.

### Hasil runtime
Pada firmware `e860367`:

```text
DISPLAY: OLED DISABLED - audio stability test
```

Timeout I2C yang sebelumnya berulang tidak lagi muncul. Wi-Fi, DNS, TCP 443, WebSocket Gemini, setupComplete, dan RX audio tetap berhasil.

Kemudian ditemukan masalah WebSocket:

```text
websocket_client: Could not lock ws-client within 2000 timeout for PING
```

### Kesimpulan
OLED/I2C bukan kandidat utama untuk masalah runtime tersebut.

### Status
**OLED-OFF TEST VERIFIED — I2C TIMEOUT GONE**

---

## v7.0.27 — WebSocket Audio TX Lock Tuning — August 20, 2026

Runtime OLED-OFF menunjukkan:

```text
Could not lock ws-client within 2000 timeout for PING
```

Audio microphone dikirim setiap 3200 byte melalui TX worker. Timeout `esp_websocket_client_send_text()` audio sebelumnya dapat mencapai 5000 ms.

### Perubahan
- Timeout pengiriman audio diturunkan dari 5000 ms menjadi 250 ms.
- Frame audio yang timeout/fail dibuang.
- Kegagalan satu frame tidak langsung mematikan koneksi WebSocket.
- Logging TX audio diperjelas.
- TX queue dan kebijakan drop frame lama dipertahankan.

### Status
**TESTED — PING LOCK REMAINED AN ISSUE**

---

## v7.0.28 — WebSocket PING Lock Diagnostic — August 20, 2026

### Perubahan
- WebSocket keep-alive PING dinonaktifkan sementara.
- Jalur audio, RX worker, ring buffer, I2S, MAX98357A, dan volume processing tidak diubah.

### Tujuan
Menguji apakah PING/keep-alive menjadi pemicu WebSocket macet atau hanya korban dari lock yang sudah tertahan.

### Status
**SUPERSEDED BY v7.0.29**

---

## v7.0.29 — WebSocket Audio TX Diagnostic — August 20, 2026

Setelah PING dimatikan, sistem berhasil menerima audio Gemini tetapi koneksi kemudian gagal ketika jalur TX realtime aktif.

### Runtime evidence

```text
WS_EVENT: WebSocket TERHUBUNG ke Gemini!
WS_JSON: Gemini setupComplete: SESI SIAP
WS_JSON: AUDIO GEMINI: 1920 byte -> AUDIO BUFFER
WS_JSON: AUDIO GEMINI: 5760 byte -> AUDIO BUFFER
```

Statistik RX tetap sehat:

```text
seq_err=0
buffer_drop=0
queue_drop=0
invalid=0
oversize=0
```

Audio sempat berjalan:

```text
received=88320
queued=88238
played=88238
dropped=0
```

Kemudian:

```text
E transport_ws: Error transport_poll_write(0)
E websocket_client: esp_transport_write() returned 0
E WS_EVENT: WebSocket Error!
W WS_EVENT: WebSocket TERPUTUS dari Gemini
```

### Kesimpulan
Pada titik ini:

- Wi-Fi: **PASS**
- DNS: **PASS**
- TCP 443: **PASS**
- TLS/WebSocket handshake: **PASS**
- Gemini setup: **PASS**
- Gemini RX: **PASS**
- Audio playback: **PASS sebagian**
- WebSocket/TLS TX realtime: **FAIL**

Masalah telah dipersempit ke **jalur WebSocket/TLS write saat realtime audio TX**, bukan OLED, DNS, TCP, Gemini setup, atau RX parser.

### Status
**ACTIVE INVESTIGATION — WEBSOCKET/TLS AUDIO TX**

---

## v7.0.30 — WebSocket Audio TX Write Retry Tuning — August 20, 2026

### Latar belakang

Firmware v7.0.29 menunjukkan bahwa koneksi Gemini, `setupComplete`, RX audio, dan playback dapat berjalan. Kegagalan terjadi ketika jalur realtime TX melakukan:

```text
E transport_ws: Error transport_poll_write(0)
E websocket_client: esp_transport_write() returned 0
```

Pada pengujian berikutnya, kegagalan bahkan terjadi sangat awal setelah `setupComplete`, sehingga timeout TX yang terlalu pendek perlu diuji sebagai kemungkinan faktor.

### Perubahan

- Timeout TX audio dinaikkan dari **150 ms menjadi 1000 ms**.
- Ditambahkan **1 kali retry** untuk audio write yang gagal.
- Retry dilakukan setelah jeda singkat **30 ms**.
- Retry hanya dilakukan bila koneksi WebSocket dan `connection generation` masih valid.
- `network_timeout_ms` dinaikkan dari **10 s menjadi 15 s** untuk memberi margin pada operasi jaringan.
- WebSocket keep-alive **PING tetap OFF** selama investigasi.

### Yang tidak diubah

- I2S.
- Konfigurasi INMP441.
- Konfigurasi MAX98357A.
- Sample rate microphone 16 kHz.
- Sample rate speaker 24 kHz.
- Ring buffer audio 32 KB.
- Prebuffer 12 KB.
- PCM chunk 3200 byte.
- WebSocket audio chunk 1600 byte.
- Audio baseline **v6.1.5**.
- OLED tetap OFF untuk isolasi masalah.

### Tujuan pengujian

Menentukan apakah `transport_poll_write(0)` disebabkan oleh timeout TX yang terlalu agresif atau merupakan kegagalan yang lebih fundamental pada jalur WebSocket/TLS write realtime.

### Hasil hardware test

v7.0.30 berhasil boot, mendapatkan IP, lolos DNS/TCP, terhubung ke Gemini, menerima `setupComplete`, menerima burst audio, mencapai `GENERATION COMPLETE` dan `TURN COMPLETE`, serta tidak mengalami `transport_poll_write(0)` / `esp_transport_write() returned 0` atau disconnect sampai turn selesai.

Bukti runtime:

```text
WS_EVENT: WebSocket TERHUBUNG ke Gemini!
WS_JSON: Gemini setupComplete: SESI SIAP
WS_JSON: Gemini: GENERATION COMPLETE
WS_JSON: Gemini: TURN COMPLETE - menunggu audio drain
WS_AUDIO: AUDIO PLAYBACK COMPLETE: received=375834 queued=373488 played=373488 pending=0 dropped=1990 balance=356
WS_JSON: AUDIO SUMMARY: chunks=370 received=375834 played=373488 pending=0 write_calls=183
```

### Masalah baru yang terukur

Burst RX/audio menimbulkan tekanan buffer dan fragment handling:

```text
WS_RX: RX STATS ... dropped_frag=12 seq_err=10 buffer_drop=2
WS_RX: RX STATS ... dropped_frag=34 seq_err=31 buffer_drop=3
WS_RX: RX STATS ... dropped_frag=49 seq_err=45 buffer_drop=4
```

Audio queue mengalami drop terukur:

```text
WS_AUDIO: Audio PCM drop terukur: 38 byte
WS_AUDIO: Audio PCM drop terukur: 998 byte
WS_AUDIO: Audio PCM drop terukur: 774 byte
WS_AUDIO: Audio PCM drop terukur: 180 byte
```

Ring buffer sempat hampir penuh:

```text
pending=31510/32768
```

Total akhir:

```text
received=375834
queued=373488
played=373488
dropped=1990
```

Masih terdapat satu `AUDIO PLAYBACK UNDERRUN` setelah `GENERATION COMPLETE`, tetapi audio akhirnya berhasil drain sampai `pending=0`.

### Analisis

v7.0.30 memberikan bukti kuat bahwa menaikkan timeout TX dan menambahkan retry **menghilangkan kegagalan WebSocket/TLS write pada pengujian ini**. Namun sistem sekarang menunjukkan bottleneck berbeda pada jalur **RX burst handling + audio queue/backpressure**.

Ini menggeser fokus investigasi dari **WebSocket/TLS TX failure** menjadi:

1. RX fragment pressure.
2. Sequence errors.
3. RX buffer drops.
4. Audio queue backpressure.
5. Playback underrun saat burst/akhir turn.

Semua ini harus ditangani tanpa mengubah baseline I2S v6.1.5.

### Status
**TX WRITE PATH: VERIFIED IMPROVED**  
**RX/AUDIO BUFFER PRESSURE: ACTIVE INVESTIGATION**

---

# Current Development Direction

Per 20 Agustus 2026:

1. Gemini WebSocket sudah berhasil terhubung dan satu turn penuh dapat selesai.
2. Audio v6.1.5 tetap menjadi baseline yang dilindungi.
3. Pemrosesan volume masih dinonaktifkan untuk uji kestabilan.
4. OLED/I2C dinonaktifkan sementara dan timeout I2C tidak muncul pada pengujian OLED-OFF.
5. WebSocket keep-alive PING dinonaktifkan sementara.
6. v7.0.30 menunjukkan jalur WebSocket/TLS TX dapat bertahan sampai satu Gemini turn selesai.
7. Fokus utama sekarang bergeser ke **RX fragment pressure, sequence errors, buffer drops, dan audio queue backpressure**.
8. Jangan mengubah konfigurasi I2S v6.1.5 selama investigasi ini.
9. Perubahan berikutnya harus berupa tuning terukur dan diverifikasi dengan log hardware.

Lihat `CURRENT_STATE.md` untuk status paling mutakhir.

---

# Documentation Rule

Jika sebuah perubahan menghasilkan keputusan teknis penting atau baseline baru:

1. Catat milestone di file ini.
2. Perbarui `CURRENT_STATE.md` jika kondisi aktif berubah.
3. Perbarui `PROJECT_CONTEXT.md` jika informasi tersebut menjadi pengetahuan proyek yang stabil.
4. Jangan menghapus sejarah yang masih relevan; tambahkan milestone baru.
