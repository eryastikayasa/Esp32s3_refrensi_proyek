# ESP32-S3 Voice Commander — Project Changelog

Dokumen ini mencatat milestone, baseline, perubahan arsitektur penting, hasil hardware test, dan keputusan tuning agar konteks proyek tidak hilang saat pekerjaan dilanjutkan oleh AI.

> Jangan menghapus sejarah yang masih relevan. Tambahkan milestone baru.

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

Current audio buffering sebelum v7.0.31:

```text
ring=32768
prebuffer=12288
target=48000 B/s
```

**Status:** volume processing tetap OFF selama stability investigation.

---

## OLED OFF — Audio Isolation Test — August 20, 2026

OLED/I2C dimatikan untuk isolasi audio.

Hasil: timeout I2C sebelumnya tidak muncul. Wi-Fi, DNS, TCP, WSS Gemini, setupComplete, dan RX audio tetap berhasil.

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

### Perubahan

- Audio TX timeout: 150 ms -> 1000 ms.
- Retry audio write: 1 kali.
- Retry delay: 30 ms.
- Retry hanya jika connection generation masih valid.
- Network timeout: 10 s -> 15 s.
- WebSocket PING tetap OFF.

### Hardware result

Full Gemini turn berhasil:

```text
WS_EVENT: WebSocket TERHUBUNG ke Gemini!
WS_JSON: Gemini setupComplete: SESI SIAP
WS_JSON: Gemini: GENERATION COMPLETE
WS_JSON: Gemini: TURN COMPLETE - menunggu audio drain
WS_AUDIO: AUDIO PLAYBACK COMPLETE: received=375834 queued=373488 played=373488 pending=0 dropped=1990 balance=356
WS_JSON: AUDIO SUMMARY: chunks=370 received=375834 played=373488 pending=0 write_calls=183
```

Tidak terjadi `transport_poll_write(0)` atau disconnect sebelum turn selesai.

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

**Kesimpulan:** TX WRITE PATH VERIFIED IMPROVED. Fokus pindah ke RX fragment/slot pressure dan audio queue backpressure.

---

## v7.0.31 — RX Persistent Slot Headroom Tuning — August 20, 2026

### Dasar tuning

v7.0.30 menunjukkan:

```text
queue_hwm=4
buffer_drop=4
```

dan log berulang:

```text
RX slot persistent: slot=...
RX BUFFER DROP: no free slot
```

RX queue dan persistent slot pool sebelumnya sama-sama berjumlah **6**:

```text
WS_RX_SLOT_COUNT 6
WS_RX_QUEUE_LENGTH WS_RX_SLOT_COUNT
```

Dengan burst Gemini, empat `buffer_drop` menunjukkan pool dapat habis sebelum worker RX selesai membebaskan slot.

### Perubahan firmware

Pada repository firmware `eryastikayasa/esp32s3_voice_geminiproject`:

```text
WS_RX_SLOT_COUNT: 6 -> 8
WS_RX_QUEUE_LENGTH: tetap mengikuti WS_RX_SLOT_COUNT -> 8
```

Tidak ada perubahan pada:

- I2S v6.1.5.
- INMP441.
- MAX98357A.
- Mic 16 kHz.
- Speaker 24 kHz.
- Audio ring 32 KB.
- Prebuffer 12 KB.
- WebSocket TX timeout/retry v7.0.30.
- PING tetap OFF.
- OLED tetap OFF.

### Tujuan

Memberi headroom dua slot tambahan untuk burst RX tanpa mengubah jalur audio/I2S. Tuning ini secara khusus menargetkan `buffer_drop` dan `queue_hwm`, bukan mencoba menyembunyikan `seq_err`.

### Status

**FIRMWARE TUNED — MENUNGGU HARDWARE LOG v7.0.31**

### Kriteria keberhasilan

Target utama:

```text
buffer_drop -> 0
queue_hwm < 8
queue_drop -> 0
```

Kemudian lihat apakah `dropped_frag` dan `seq_err` ikut turun. Jika `seq_err` tetap tinggi walaupun `buffer_drop=0`, investigasi berikutnya harus fokus pada validitas `payload_offset`/fragment ordering, bukan menambah slot lagi.

---

# Current Development Direction

Per 20 Agustus 2026:

1. Gemini WebSocket dapat connect dan full turn sudah terbukti selesai.
2. WebSocket/TLS TX v7.0.30 terbukti jauh lebih stabil pada hardware test.
3. Audio/I2S v6.1.5 tetap **LOCKED**.
4. Volume processing tetap OFF.
5. OLED/I2C tetap OFF selama isolation.
6. PING tetap OFF.
7. v7.0.31 sekarang menguji peningkatan RX persistent slot pool dari 6 ke 8.
8. Jangan menaikkan ring buffer secara buta; v7.0.30 menunjukkan largest free block sekitar 31 KB, sehingga alokasi ring internal yang lebih besar berisiko gagal.
9. Fokus berikutnya ditentukan dari log v7.0.31:
   - pertama `buffer_drop` / `queue_hwm`;
   - kemudian `dropped_frag` / `seq_err`;
   - kemudian PCM drop dan underrun.
10. Jangan menyentuh I2S v6.1.5.

---

# Documentation Rule

Setiap tuning harus dicatat dengan:

1. nomor versi;
2. file yang diubah;
3. parameter sebelum/sesudah;
4. alasan berdasarkan log;
5. hal yang sengaja tidak diubah;
6. hasil build/hardware test;
7. metrik log sebelum/sesudah;
8. keputusan tuning berikutnya.

`CHANGELOG_PROJECT.md` = sejarah keputusan teknis.
`CURRENT_STATE.md` = kondisi aktif saat ini.
`PROJECT_CONTEXT.md` = pengetahuan proyek yang stabil.
