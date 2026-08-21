# ESP32-S3 Voice Commander — Project Changelog

Dokumen ini mencatat milestone, baseline, perubahan arsitektur penting, hasil hardware test, dan keputusan tuning agar konteks proyek tidak hilang saat pekerjaan dilanjutkan oleh AI.

> Jangan menghapus sejarah yang masih relevan. Tambahkan milestone baru.
> Gunakan bahasa sehari-hari/sederhana saat menjelaskan perubahan.
> Setiap perubahan firmware wajib dicatat di dokumen ini agar AI tidak kembali ke awal.

---

## v7.1.3 — Audio Playback WDT Stabilization — August 20, 2026

### Dasar perubahan

v7.1.2 menunjukkan `audio_playback` membuat CPU0/IDLE0 starvation sebelum WebSocket Gemini sempat berjalan. Task WDT muncul berulang pada CPU0 dengan:

```text
task_wdt: - IDLE0
CPU 0: audio_playback
```

Karena masalah muncul sebelum WebSocket terhubung, v7.1.3 difokuskan hanya pada scheduler/task playback.

### Perubahan firmware

File utama:

```text
components/websocket/websocket_audio.cpp
```

Perubahan:

- Priority task `audio_playback` diturunkan dari **7 menjadi 3**.
- Task tetap pinned ke **CPU0** untuk pengujian terkontrol.
- Ditambahkan `vTaskDelay(1)` sebagai yield scheduler setelah setiap siklus playback.
- Jalur ketika buffer kosong juga memberi kesempatan scheduler.
- `audio_clear_pending` tidak lagi menunggu mutex dengan `portMAX_DELAY`; digunakan timeout pendek **10 ms**.
- Jika mutex sedang sibuk, proses clear ditunda dan task melakukan yield.

### Yang TIDAK diubah

- I2S v6.1.5 tetap LOCKED.
- INMP441 tetap.
- MAX98357A tetap.
- Mic tetap 16 kHz.
- Speaker tetap 24 kHz.
- Format PCM tetap.
- DMA/I2S configuration tidak diubah.
- WebSocket/TLS tidak diubah.
- Ring buffer tidak diubah.
- Volume processing tetap OFF / forced 100%.
- OLED tetap OFF selama audio stability test.

### Tujuan hardware test

Menguji satu hipotesis secara terisolasi:

```text
audio_playback tidak boleh membuat IDLE0 starvation
WebSocket harus bisa mulai normal
I2S v6.1.5 tetap aman
```

### Hasil hardware v7.1.3

Pengujian hardware menunjukkan perubahan scheduler/task playback berhasil memperbaiki masalah WDT dan jalur audio Gemini berjalan normal.

#### WebSocket/RX stabil

```text
fragments=115 dropped_frag=0 messages=23 queue_hwm=3 seq_err=0 buffer_drop=0 queue_drop=0 invalid=0 oversize=0
```

Selama turn audio yang diuji:

- `dropped_frag=0`
- `seq_err=0`
- `buffer_drop=0`
- `queue_drop=0`
- `invalid=0`
- `oversize=0`

Ini merupakan perbaikan besar dibanding v7.1.1 yang sebelumnya mencapai `dropped_frag=154`, `seq_err=144`, dan `buffer_drop=10`.

#### Audio playback berhasil drain

```text
WS_JSON: AUDIO SUMMARY: chunks=95 received=97859 played=77824 pending=18918 write_calls=38
WS_AUDIO: AUDIO FLOW: pending=2534/49152 received=97859 queued=98790 played=96256 dropped=0
WS_AUDIO: AUDIO PLAYBACK COMPLETE: received=97859 queued=98790 played=98790 pending=0 dropped=0 balance=-931
```

Hasil penting:

- `dropped=0`
- `pending=0` setelah playback selesai
- `played=98790` sama dengan `queued=98790`
- `AUDIO PLAYBACK COMPLETE` berhasil
- Tidak muncul `AUDIO PLAYBACK UNDERRUN` pada turn ini

#### Task WDT

Tidak ditemukan lagi log:

```text
task_wdt: Task watchdog got triggered
```

pada `audio_playback` selama pengujian v7.1.3 yang dikirim.

### Analisis

v7.1.3 berhasil menghilangkan masalah starvation yang sebelumnya membuat `audio_playback` mengganggu scheduler. Jalur Gemini → RX → audio ring → speaker sekarang dapat menyelesaikan satu turn tanpa PCM drop dan tanpa WDT.

I2S v6.1.5 tetap tidak disentuh.

### Catatan yang masih perlu diuji

Heap terbesar sempat turun dari sekitar 31 KB menjadi sekitar 15 KB ketika RX/audio sedang sibuk. Ini belum menunjukkan kegagalan, tetapi margin heap perlu dipantau pada pengujian multi-turn.

Karena hasil ini baru menunjukkan satu turn yang bersih, **v7.1.3 belum ditetapkan sebagai baseline final**. Pengujian berikutnya sebaiknya berupa beberapa turn/percakapan berturut-turut tanpa mengubah I2S.

### Keputusan

Jangan melakukan tuning I2S lagi berdasarkan log ini.

Jangan menambahkan audio control, UART, servo, atau fitur JSON baru sebelum stabilitas multi-turn v7.1.3 dibuktikan.

**Status: HASIL HARDWARE BERHASIL / BELUM BASELINE FINAL**

### Arah Investigasi Audio Berikutnya — WAJIB DIPERTAHANKAN

Hasil v7.1.3 menunjukkan jalur **Gemini → RX → decoder → audio ring → playback** sudah menerima dan memainkan data tanpa `dropped=0`, `pending=0`, dan playback dapat selesai. Namun suara dari speaker masih belum jelas. Karena itu investigasi berikutnya dibagi menjadi **dua hipotesis berurutan** dan tidak boleh dicampur.

#### Hipotesis A — PCM Gemini sudah bagus, tetapi output MAX98357A bermasalah

Ini adalah **hipotesis pertama yang harus diuji**.

Tujuan:

```text
PCM test tone lokal
    ↓
I2S v6.1.5
    ↓
MAX98357A
    ↓
speaker
```

Jika test tone lokal terdengar **jernih, normal, dan stabil**, maka jalur I2S + MAX98357A dianggap normal untuk sementara dan investigasi berpindah ke Hipotesis B.

Jika test tone lokal juga terdengar **pecah, serak, pelan, atau tidak jelas**, jangan menyalahkan PCM Gemini. Fokus tetap pada jalur:

```text
I2S → MAX98357A → speaker
```

**Aturan penting:** konfigurasi I2S v6.1.5 tetap LOCKED selama pengujian ini. Jangan melakukan tuning I2S secara acak.

#### Hipotesis B — PCM Gemini berubah/terkorupsi sebelum sampai I2S

Hipotesis ini **baru boleh diperiksa setelah Hipotesis A terbukti normal**.

Yang perlu diperiksa nanti:

```text
Gemini Base64
    ↓
Base64 decode
    ↓
PCM16
    ↓
queue_audio_pcm()
    ↓
audio ring
    ↓
audio_write_speaker()
    ↓
I2S
```

Pemeriksaan yang direncanakan:

- ukuran PCM hasil decode;
- sample min/max;
- peak/RMS sederhana;
- pola sample PCM;
- kemungkinan byte order/format yang salah;
- memastikan data PCM yang keluar dari decoder sama dengan data yang benar-benar diberikan ke jalur I2S.

### Urutan keputusan untuk versi berikutnya

```text
V7.1.3
  ↓
TES A: test tone lokal → MAX98357A
  ↓
Apakah speaker normal?
  ├─ TIDAK → investigasi I2S/MAX98357A/speaker
  │          tanpa menyentuh PCM Gemini
  │
  └─ YA → lanjut TES B
             ↓
       audit PCM Gemini
       Base64 → PCM16 → ring → I2S
```

**Keputusan:** dua hipotesis ini menjadi peta investigasi resmi untuk versi berikutnya agar troubleshooting tidak kembali ke awal dan tidak melakukan perubahan pada bagian yang sudah terbukti stabil.

### Hasil Pengujian Hipotesis A — PASS

Test:
PCM 1 kHz lokal
→ I2S v6.1.5
→ MAX98357A
→ Speaker

Hasil:
- 10/10 test tone berhasil.
- Suara speaker JERNIH dan BERSIH.
- Tidak ditemukan gangguan pada jalur output saat test tone.
- Tidak ada Gemini/WebSocket/Wi-Fi pada pengujian.
- I2S v6.1.5 tetap LOCKED dan tidak diubah.

Kesimpulan:
Hipotesis A tidak terbukti sebagai sumber masalah.
Jalur I2S v6.1.5 → MAX98357A → speaker dinyatakan normal.

Keputusan berikutnya:
Lanjut ke Hipotesis B.

Hipotesis B:
Audit jalur audio Gemini:
Gemini PCM/Base64
→ decode
→ PCM16
→ audio buffer/ring
→ audio_write_speaker()
→ I2S v6.1.5

Fokus:
Memastikan PCM dari Gemini tidak berubah, korup,
salah format, salah ukuran, atau rusak sebelum masuk I2S.

Catatan penting:
I2S v6.1.5 tetap LOCKED.
Jangan mengubah konfigurasi hardware/output selama pengujian Hipotesis B.

## Hipotesis B — Hasil Analisis V7.1.3

### Status
DIAGNOSIS BELUM FINAL — JANGAN PATCH DULU

### Tujuan Pengujian
Mencari penyebab audio Gemini Live yang berpotensi tersendat dengan memeriksa jalur:

Gemini → WebSocket RX → Base64 Decode → PCM16 → Ring Buffer → I2S → MAX98357A

### Hasil Pengamatan

1. WebSocket RX TERLIHAT SEHAT
- dropped_frag = 0
- seq_err = 0
- buffer_drop = 0
- queue_drop = 0
- invalid = 0
- oversize = 0
- queue_hwm hanya sekitar 1–2
- Tidak ditemukan indikasi kehilangan fragment WebSocket.

KESIMPULAN:
WebSocket packet/fragment loss bukan tersangka utama berdasarkan log ini.

2. PCM dari Gemini BERHASIL DITERIMA
- Terdapat AUDIO GEMINI dengan ukuran PCM seperti 5760 byte, 1920 byte, dll.
- PCM memiliki nilai min/max/RMS yang valid.
- Data bukan hanya PCM kosong/silent.

KESIMPULAN:
Base64 decode → PCM16 berhasil menghasilkan data audio.

3. Ring Buffer TIDAK MENUNJUKKAN DROP
Contoh hasil akhir:

received = 116160 byte
queued   = 116050 byte
played   = 116050 byte
dropped  = 0 byte
pending  = 0

Selisih received vs queued hanya 110 byte.

KESIMPULAN:
Tidak ada bukti ring buffer membuang audio secara signifikan.
Ring buffer bukan tersangka utama berdasarkan accounting ini.

4. Playback BERHASIL MENGHABISKAN SELURUH AUDIO
Log akhir:

AUDIO PLAYBACK COMPLETE
received=116160
queued=116050
played=116050
pending=0
dropped=0

KESIMPULAN:
Seluruh audio yang masuk ke jalur playback berhasil dikonsumsi.
Tidak ditemukan underflow/drop permanen dari accounting audio.

5. Producer AUDIO SEMPAT LEBIH CEPAT DARIPADA PLAYBACK
Contoh:

pending=17324/49152
pending=20866/49152
pending=24408/49152

Artinya audio dari Gemini datang secara burst dan sempat menumpuk di buffer.

Namun buffer akhirnya kembali:

pending=0

KESIMPULAN:
Ada perbedaan timing producer vs playback, tetapi belum terbukti sebagai penyebab audio tersendat.

6. TEMUAN PALING MENCURIGAKAN — PCM FULL-SCALE
Banyak audit PCM menunjukkan:

peak=32767
peak=32768

Bahkan berulang kali muncul:

FULL-SCALE PCM peak=32767
FULL-SCALE PCM peak=32768

Nilai tersebut muncul pada BEFORE_RING maupun BEFORE_I2S.

Selain itu amplitudo PCM berubah sangat ekstrem antar chunk:
- full-scale sekitar ±32768
- medium amplitude
- amplitude sangat kecil
- bahkan silent PCM

KESIMPULAN:
Perlu investigasi lebih lanjut apakah:
A. Audio Gemini memang mengalami clipping/peak ekstrem,
atau
B. Ada perubahan/distorsi PCM pada jalur decode/copy/ring.

Belum boleh menyimpulkan penyebab sebelum data PCM dibandingkan antar titik.

7. RX PROCESS MEMPUNYAI LATENCY YANG PERLU DIPERHATIKAN
Beberapa contoh:

RX PROCESS len=277  time=21 ms
RX PROCESS len=7888 time=36 ms

Latency ini perlu diperiksa karena sistem audio berjalan real-time.

Namun logging audit PCM sendiri sangat berat dan kemungkinan ikut menambah beban CPU/UART.

KESIMPULAN:
Latency belum dapat dinyatakan sebagai penyebab utama sebelum pengujian dilakukan dengan logging minimal.

8. PCM AUDIT LOGGING TERLALU AGRESIF
Log menghasilkan sangat banyak informasi:

min
max
peak
rms
mean_abs
zero
hash
first
last

dan dicetak sangat rapat.

Pada monitor 115200 baud, logging sebanyak ini berpotensi mengganggu timing sistem.

KESIMPULAN:
Pengujian berikutnya harus menggunakan logging minimal agar hasil timing audio tidak terkontaminasi oleh overhead debug.

### Yang SUDAH DIBEBASKAN DARI KECURIGAAN

- WebSocket fragment loss: tidak terbukti
- WebSocket queue drop: tidak terbukti
- Ring buffer drop: tidak terbukti
- Audio playback accounting: normal
- I2S baseline v6.1.5: JANGAN DIUBAH

### Fokus Investigasi Berikutnya

Perlu membandingkan PCM pada tiga titik:

1. PCM tepat setelah Base64 decode
2. PCM sebelum masuk ring buffer (BEFORE_RING)
3. PCM setelah keluar ring buffer tepat sebelum I2S (BEFORE_I2S)

Setiap titik harus dibandingkan menggunakan:

- length
- sample count
- min
- max
- peak
- RMS
- hash

Tujuan:
Memastikan apakah PCM berubah antara:

DECODE → RING → I2S

### Hipotesis Kerja Saat Ini

Hipotesis B BELUM TERBUKTI sebagai masalah WebSocket atau buffer overflow.

Tersangka utama sementara:

1. Integritas/transformasi PCM
2. Pola full-scale PCM yang berulang
3. Timing RX/decode
4. Overhead logging yang sangat berat

### Aturan Eksperimen

JANGAN mengubah I2S baseline v6.1.5.

JANGAN melakukan patch berdasarkan log ini saja.

Lakukan satu perubahan/eksperimen pada satu waktu.

Sebelum perubahan kode:
- wajib mencari/membaca CHANGELOG_PROJECT.md pada repositori referensi terlebih dahulu.
- Setelah eksperimen selesai, hasil harus dicatat kembali agar riwayat diagnosis tidak hilang.

### Status Akhir Hipotesis B

BELUM TERBUKTI.

Kesimpulan sementara:
Jalur WebSocket dan ring buffer terlihat sehat. Audio PCM berhasil diterima dan seluruh audio berhasil dimainkan. Anomali utama yang perlu dibedah adalah pola PCM FULL-SCALE yang berulang, kemungkinan transformasi PCM, latency RX/decode, dan efek logging debug yang terlalu berat.

NEXT STEP:
Audit integritas PCM dari DECODE → BEFORE_RING → BEFORE_I2S dengan logging minimal.

### CATATAN PROGRES — HIPOTESIS B
Tahap 1 — PCM 3 Checkpoint Audit
Status: ✅ SELESAI
Versi acuan: V7.1.3
Branch: Hipotesis B
File yang diubah: components/websocket/websocket_audio.cpp
Tujuan Tahap 1
Memastikan tersedia 3 titik pengamatan PCM tanpa mengubah logika audio.
Checkpoint
Lokasi
Fungsi
CP1 — AFTER_DECODE
Setelah PCM hasil decode diterima jalur audio
Melihat PCM paling awal
CP2 — BEFORE_RING
Tepat sebelum PCM masuk ring buffer
Memastikan PCM sebelum ring
CP3 — BEFORE_I2S
Tepat sebelum audio_write_speaker()
Memastikan PCM sebelum I2S
Prinsip perubahan
✅ Hanya untuk audit/diagnostik
✅ Tidak mengubah sample PCM
✅ Tidak mengubah ukuran/urutan PCM
✅ Tidak mengubah ring buffer
✅ Tidak mengubah playback
✅ I2S v6.1.5 tetap LOCKED
✅ Tidak mengubah logika audio utama
Commit Tahap 1
8d185e89e9eb16218a10a65723e2fd8acc0ddc84
Pesan commit:
test: add PCM checkpoint 1 for Hypothesis B
Commit tersebut mencatat penambahan CP1 AFTER_DECODE dan mempertahankan CP2/CP3 yang sudah ada.
Kesimpulan Tahap 1
Tiga checkpoint PCM sekarang sudah tersedia sebagai jalur observasi: CP1 → CP2 → CP3.
Belum ada kesimpulan bahwa PCM rusak di titik tertentu. Tahap 1 hanya menyiapkan alat pengamatan.
Berikutnya
Tahap 2 — Desain audit minimal:
length + hash + peak + RMS
Tujuannya nanti bukan sekadar melihat log, tetapi membandingkan karakteristik PCM CP1, CP2, dan CP3 untuk mengetahui apakah datanya berubah di antara checkpoint.
Catatan ini menjadi baseline. Jangan mengubah Tahap 1 lagi kecuali ditemukan masalah.

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
