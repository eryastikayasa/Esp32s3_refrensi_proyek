# ESP32-S3 Voice Commander — Project Changelog

Dokumen ini mencatat milestone, baseline, perubahan arsitektur penting, dan hasil debugging yang relevan untuk melanjutkan proyek.

> Ini bukan daftar setiap commit. Hanya perubahan dan keputusan teknis yang penting untuk konteks AI.

---

## v7.0.30 — WebSocket Audio TX Write Retry Tuning — August 20, 2026

### Hasil hardware test

Firmware v7.0.30 berhasil boot, Wi-Fi GOT_IP, DNS OK, TCP 443 OK, WebSocket Gemini CONNECTED, `setupComplete`, menerima burst audio Gemini, mencapai `GENERATION COMPLETE` dan `TURN COMPLETE`, serta tidak mengalami `transport_poll_write(0)` / `esp_transport_write() returned 0` atau disconnect sampai turn selesai.

Bukti runtime:

```text
WS_EVENT: WebSocket TERHUBUNG ke Gemini!
WS_JSON: Gemini setupComplete: SESI SIAP
WS_JSON: Gemini: GENERATION COMPLETE
WS_JSON: Gemini: TURN COMPLETE - menunggu audio drain
WS_AUDIO: AUDIO PLAYBACK COMPLETE: received=375834 queued=373488 played=373488 pending=0 dropped=1990 balance=356
WS_JSON: AUDIO SUMMARY: chunks=370 received=375834 played=373488 pending=0 write_calls=183
```

### Perubahan yang diuji

- Timeout TX audio: 150 ms -> 1000 ms.
- Retry audio write: 1 kali, jeda 30 ms.
- Retry hanya jika connection generation masih valid.
- `network_timeout_ms`: 10 s -> 15 s.
- WebSocket PING tetap OFF.

### Yang tetap dikunci

- Audio/I2S baseline v6.1.5.
- INMP441 dan MAX98357A.
- Mic 16 kHz / speaker 24 kHz.
- Ring buffer 32 KB / prebuffer 12 KB.
- PCM chunk 3200 byte / WebSocket audio chunk 1600 byte.
- OLED tetap OFF untuk isolasi.

### Temuan baru

TX WebSocket/TLS sekarang **bertahan sampai full Gemini turn** pada pengujian ini. Namun burst RX/audio menimbulkan tekanan buffer:

```text
 dropped_frag=12 seq_err=10 buffer_drop=2
 dropped_frag=34 seq_err=31 buffer_drop=3
 dropped_frag=49 seq_err=45 buffer_drop=4
```

Audio queue juga mengalami drop:

```text
Audio PCM drop terukur: 38 byte
Audio PCM drop terukur: 998 byte
Audio PCM drop terukur: 774 byte
Audio PCM drop terukur: 180 byte
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

Masih terdapat satu `AUDIO PLAYBACK UNDERRUN` setelah generation complete, tetapi audio akhirnya berhasil drain sampai `pending=0`.

### Kesimpulan

v7.0.30 **memperbaiki/menstabilkan jalur TX WebSocket pada pengujian ini**, tetapi membuka bottleneck berikutnya: **RX fragment pressure + sequence errors + buffer drops + audio queue backpressure**.

### Status
**TX WRITE PATH: VERIFIED IMPROVED**  
**RX/AUDIO BUFFER PRESSURE: ACTIVE INVESTIGATION**

---

# Current Development Direction

Per 20 Agustus 2026:

1. Gemini WebSocket sudah berhasil terhubung dan satu turn penuh dapat selesai.
2. Audio v6.1.5 tetap menjadi baseline yang dilindungi.
3. Pemrosesan volume tetap dinonaktifkan untuk uji kestabilan.
4. OLED/I2C tetap OFF selama isolasi audio.
5. WebSocket keep-alive PING tetap OFF selama investigasi.
6. Fokus utama sekarang adalah RX burst handling, fragment/sequence integrity, buffer drops, dan audio queue backpressure.
7. Jangan mengubah konfigurasi I2S v6.1.5.
8. Perubahan berikutnya harus berupa tuning terukur dan diverifikasi dengan log hardware.

---

# Documentation Rule

Jika sebuah perubahan menghasilkan keputusan teknis penting atau baseline baru:

1. Catat milestone di file ini.
2. Perbarui `CURRENT_STATE.md` jika kondisi aktif berubah.
3. Perbarui `PROJECT_CONTEXT.md` jika informasi tersebut menjadi pengetahuan proyek yang stabil.
4. Jangan menghapus sejarah yang masih relevan; tambahkan milestone baru.
