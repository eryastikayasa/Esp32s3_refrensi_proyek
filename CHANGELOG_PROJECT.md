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

### Perubahan yang ditargetkan

- Audio TX timeout: 150 ms -> 1000 ms.
- Retry audio write: 1 kali.
- Retry delay: 30 ms.
- Retry hanya jika connection generation masih valid.
- Network timeout: 10 s -> 15 s.
- WebSocket PING tetap OFF.

### Hardware result yang menjadi referensi

Full Gemini turn berhasil:

```text
WS_EVENT: WebSocket TERHUBUNG ke Gemini!
WS_JSON: Gemini setupComplete: SESI SIAP
WS_JSON: Gemini: GENERATION COMPLETE
WS_JSON: Gemini: TURN COMPLETE - menunggu audio drain
WS_AUDIO: AUDIO PLAYBACK COMPLETE: received=375834 queued=373488 played=373488 pending=0 dropped=1990 balance=356
WS_JSON: AUDIO SUMMARY: chunks=370 received=375834 played=373488 pending=0 write_calls=183
```

Tidak terjadi `transport_poll_write(0)` atau disconnect sebelum turn selesai pada hasil referensi tersebut.

### Bottleneck yang ditemukan

```text
dropped_frag=12 seq_err=10 buffer_drop=2
dropped_frag=34 seq_err=31 buffer_drop=3
dropped_frag=49 seq_err=45 buffer_drop=4
```

Audio queue drop:

```text
38 byte
998 byte
774 byte
180 byte
```

Ring buffer mencapai:

```text
pending=31510/32768
```

Satu playback underrun masih muncul, tetapi audio akhirnya drain ke zero.

**Kesimpulan:** TX WRITE PATH terbukti membaik pada hasil referensi. Fokus kemudian pindah ke RX fragment/slot pressure dan audio queue backpressure.

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

Tidak ada perubahan pada:

- I2S v6.1.5.
- INMP441.
- MAX98357A.
- Mic 16 kHz.
- Speaker 24 kHz.
- Audio ring 32 KB.
- Prebuffer 12 KB.
- Volume processing tetap OFF.
- OLED tetap OFF.
- PING tetap OFF.

### Hasil hardware v7.0.31

Firmware berhasil boot, Wi-Fi/DNS/TCP/WSS dan `setupComplete` berhasil. Namun koneksi kembali putus **sebelum burst audio RX dimulai**:

```text
WS_EVENT: WebSocket TERHUBUNG ke Gemini!
WS_JSON: Gemini setupComplete: SESI SIAP
WS_JSON: Session resumption handle tersimpan
E transport_ws: Error transport_poll_write(0)
E websocket_client: esp_transport_write() returned 0
W WS_EVENT: Connection generation invalidated
W WS_EVENT: WebSocket TERPUTUS dari Gemini
```

Karena putus terjadi sekitar 16,7 detik dan sebelum log `AUDIO GEMINI`, perubahan slot RX v7.0.31 **belum bisa dinilai**. Ini penting: jangan menganggap v7.0.31 gagal karena slot RX; masalah TX muncul lebih dulu.

### Temuan penting

Source `main` setelah v7.0.31 masih memakai:

```text
AUDIO_SEND_TIMEOUT = 150 ms
network_timeout_ms = 10000
retry = tidak ada
```

Padahal hasil hardware v7.0.30 yang dijadikan referensi menggunakan tuning TX yang lebih longgar. Jadi v7.0.32 harus lebih dulu menyamakan source dengan konfigurasi TX yang sudah terbukti pada hasil referensi.

**Status:** RX slot tuning dipertahankan, tetapi validasi ditunda sampai TX stabil kembali.

---

## v7.0.32 — Restore Proven WebSocket TX Timing + RX Slot v7.0.31 — August 20, 2026

### Perubahan firmware

Pada `components/websocket/websocket_mgr.cpp`:

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

Log v7.0.31 menunjukkan error yang sama seperti masalah TX sebelumnya:

```text
transport_poll_write(0)
esp_transport_write() returned 0
```

dan terjadi sebelum audio RX masuk. Jadi jangan menambah tuning RX lagi dulu. Kita kembalikan timing TX yang sudah pernah menghasilkan **full Gemini turn berhasil**.

### Yang sengaja TIDAK diubah

- Audio/I2S v6.1.5.
- INMP441.
- MAX98357A.
- Mic 16 kHz.
- Speaker 24 kHz.
- Audio ring 32 KB.
- Prebuffer 12 KB.
- Volume processing.
- OLED.
- Session resumption.
- RX slot expansion v7.0.31.

### Target test v7.0.32

Urutan keberhasilan yang dicari:

```text
Wi-Fi READY
 -> DNS OK
 -> TCP 443 OK
 -> WebSocket CONNECTED
 -> setupComplete
 -> tidak ada transport_poll_write(0)
 -> AUDIO GEMINI masuk
 -> GENERATION COMPLETE
 -> TURN COMPLETE
 -> AUDIO PLAYBACK COMPLETE
```

Jika sudah sampai audio RX tetapi muncul `buffer_drop`, `seq_err`, atau `queue_hwm`, baru kita lanjut tuning RX. Jangan mengubah baseline audio v6.1.5.

**Status: FIRMWARE v7.0.32 SUDAH DITERAPKAN DI MAIN — MENUNGGU HARDWARE LOG.**

---

## Project Rule — Jangan Kembali ke Awal

Setiap tuning berikutnya wajib:

1. Baca `CHANGELOG_PROJECT.md` terlebih dahulu.
2. Pertahankan baseline v6.1.5.
3. Jangan mengulang eksperimen yang sudah terbukti atau sudah ditolak.
4. Catat versi baru, alasan perubahan, hasil log, dan keputusan berikutnya.
5. Gunakan bahasa sehari-hari/sederhana agar konteks mudah dipahami.
6. Setelah setiap perubahan firmware, update changelog ini.
