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

### Tujuan
Mencegah crash saat komunikasi WebSocket Gemini berlangsung.

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

### Tujuan
Mengurangi kemungkinan akses terhadap koneksi WebSocket yang sudah tidak valid serta memperbaiki lifecycle antar koneksi.

### Status
Dilanjutkan ke v7.0.4 untuk penyelesaian crash.

---

## v7.0.4 — WebSocket Crash Resolution Focus

- Fokus pada penyelesaian crash setelah mitigasi v7.0.3.
- Area yang disentuh mencakup lifecycle Wi-Fi, main application flow, dan WebSocket manager.
- Investigasi crash juga menggunakan hasil `addr2line` yang mengarah ke jalur SSL/WebSocket initialization.

### Status
Milestone debugging historis; runtime terbaru menunjukkan koneksi Gemini sudah berhasil.

---

## TLS / Certificate Troubleshooting — Historical Milestone

Sebelum WebSocket berhasil terhubung, proyek mengalami kegagalan TLS terhadap:

`generativelanguage.googleapis.com:443`

Log penting yang pernah muncul:

```text
esp-x509-crt-bundle: No matching trusted root certificate found
esp-x509-crt-bundle: Failed to verify certificate
esp-tls-mbedtls: mbedtls_ssl_handshake returned -0x3000
esp-tls: Failed to open new connection
transport_ws: Error connecting to host generativelanguage.googleapis.com:443
```

### Investigasi
- Certificate bundle ESP-IDF diperiksa.
- Konfigurasi mbedTLS certificate bundle diperiksa.
- DNS dan TCP connectivity diuji secara terpisah.
- DNS berhasil.
- TCP connect ke port 443 berhasil.
- Investigasi kemudian berfokus pada TLS certificate verification dan WebSocket lifecycle.

### Status
Tidak lagi menjadi blocker utama karena Gemini WebSocket kemudian berhasil terhubung.

---

## WebSocket Connection Success — August 20, 2026

Runtime log terbaru menunjukkan:

```text
WS_EVENT: WebSocket TERHUBUNG ke Gemini!
WS_EVENT: Connection generation=1
DISPLAY: [OLED STATUS]: AI Terhubung...
```

### Kesimpulan
Koneksi Gemini Live melalui WebSocket telah mencapai kondisi berhasil pada pengujian terbaru.

### Status
**VERIFIED IN RUNTIME LOG**

---

## Audio Tuning — Volume Processing Disabled — August 20, 2026

Pada commit firmware `9a5026d4` dilakukan perubahan khusus untuk **uji kestabilan audio**.

### Yang diubah
- Pemrosesan volume Gemini dinonaktifkan sementara.
- `set_gemini_volume()` sekarang memaksa volume tetap **100%**.
- `get_gemini_volume()` mengembalikan **100%**.
- Jalur `queue_audio_pcm()` tidak lagi melakukan scaling PCM sample-per-sample.
- PCM dari Gemini sekarang langsung dikirim ke ring buffer melalui `send_realtime_pcm()`.

### Yang tidak diubah
- I2S.
- MAX98357A.
- Ring buffer 32 KB.
- Prebuffer 12 KB.
- Playback wait 5 ms.
- Playback task priority 7.
- Baseline audio v6.1.5.

### Hasil runtime
Setelah volume processing dimatikan, log menunjukkan:

```text
buffer_drop=0
queue_drop=0
seq_err=0
dropped=0
pending=0
```

Selisih `balance` turun dari `-3768` menjadi `-1794` pada pengujian yang dibandingkan. Ini menunjukkan volume processing kemungkinan ikut berpengaruh, tetapi belum terbukti sebagai satu-satunya penyebab speaker sendat.

### Status
**VOLUME PROCESSING REMAINS DISABLED**

---

## OLED OFF — Audio Isolation Test — August 20, 2026

Untuk pengujian berikutnya, OLED dan akses I2C OLED **dinonaktifkan sementara**.

### Yang diubah
- `components/display/display.cpp` dibuat tidak melakukan transaksi I2C OLED.
- `oled_init()` tetap dipertahankan agar alur program tidak perlu dirombak.
- `display_status()` tetap mencatat status ke log, tetapi tidak mengirim data ke OLED.

### Hasil runtime
Pada firmware `e860367`, boot menunjukkan:

```text
DISPLAY: OLED DISABLED - audio stability test
```

dan seluruh `I2C software timeout` yang sebelumnya muncul berulang kali **hilang**.

Namun setelah `Gemini setupComplete`, sistem kemudian diam. Runtime berikutnya menunjukkan:

```text
E (41374) websocket_client: Could not lock ws-client within 2000 timeout for PING
```

### Kesimpulan
OLED/I2C **bukan lagi kandidat utama** untuk masalah diam tersebut. Pengujian ini berhasil mengisolasi I2C dari sistem dan menunjukkan masalah berikutnya berada di jalur WebSocket/runtime.

### Status
**OLED-OFF TEST VERIFIED — I2C TIMEOUT GONE**

---

## v7.0.27 — WebSocket Audio TX Lock Tuning — August 20, 2026

Runtime OLED-OFF menunjukkan masalah baru:

```text
Could not lock ws-client within 2000 timeout for PING
```

Setelah dibedah, audio microphone dikirim setiap **3200 byte** (sekitar 100 ms audio) melalui TX worker. Sebelumnya setiap `esp_websocket_client_send_text()` audio dapat menunggu sampai **5000 ms**.

### Perubahan
Pada `components/websocket/websocket_mgr.cpp`:

- Timeout pengiriman audio diturunkan dari **5000 ms menjadi 250 ms**.
- Jika satu frame audio gagal/timeout, frame tersebut **dibuang saja**.
- Kegagalan satu frame audio **tidak langsung mematikan koneksi WebSocket**.
- Ditambahkan log khusus saat TX audio timeout/fail.
- TX queue dan kebijakan drop frame lama tetap dipertahankan.

### Kenapa?
Tujuannya mencegah realtime audio memegang/menunggu lock WebSocket selama beberapa detik sehingga traffic kontrol seperti PING ikut tertahan.

### Yang tidak diubah
- I2S microphone.
- I2S speaker.
- MAX98357A.
- Ring buffer playback.
- Prebuffer.
- Volume processing tetap OFF.
- OLED tetap OFF untuk test ini.
- Connection generation tetap dipertahankan.

### Commit firmware
```text
3f6cd6d70085b2e1353742485e2945775656e272
```

### Status
**WAITING FOR RUNTIME TEST**

---

## Audio Tuning — Current Phase

Setelah koneksi Gemini berhasil, pekerjaan difokuskan pada tuning audio/runtime berdasarkan log aktual.

### Constraint
- Audio baseline v6.1.5 tetap dikunci.
- Jangan mengubah konfigurasi yang sudah terbukti tanpa alasan teknis.
- Pisahkan masalah input microphone, pemrosesan audio, output MAX98357A, WebSocket, dan I2C berdasarkan bukti log.
- Perubahan eksperimen harus diverifikasi melalui build dan runtime log sebelum dianggap sebagai baseline baru.

### Status
**IN PROGRESS — WEBSOCKET TX LOCK TEST**

---

# Current Development Direction

Per 20 Agustus 2026:

1. Gemini WebSocket sudah berhasil terhubung.
2. Audio v6.1.5 tetap menjadi baseline yang dilindungi.
3. Pemrosesan perintah volume masih dinonaktifkan untuk uji kestabilan.
4. OLED/I2C dinonaktifkan sementara dan timeout I2C sudah hilang dari runtime.
5. Masalah terbaru mengarah ke WebSocket client lock saat PING.
6. TX audio sekarang dibatasi timeout 250 ms agar tidak memonopoli lock terlalu lama.
7. Test berikutnya harus melihat apakah PING lock error dan kondisi diam hilang.
8. Jika masih bermasalah, lanjutkan bedah lifecycle/control WebSocket tanpa mengubah audio baseline.

Lihat `CURRENT_STATE.md` untuk status paling mutakhir.

---

# Documentation Rule

Jika sebuah perubahan menghasilkan keputusan teknis penting atau baseline baru:

1. Catat milestone di file ini.
2. Perbarui `CURRENT_STATE.md` jika kondisi aktif berubah.
3. Perbarui `PROJECT_CONTEXT.md` jika informasi tersebut menjadi pengetahuan proyek yang stabil.
4. Jangan menghapus sejarah yang masih relevan; tambahkan milestone baru.
