# CHANGELOG PROJECT ADDENDUM — v7.2.0

**Tanggal:** 21 Agustus 2026  
**Status:** ARSITEKTUR AUDIO DIUBAH — MENUNGGU BUILD/HARDWARE TEST

## Dasar keputusan

CHANGELOG_PROJECT.md menunjukkan Hipotesis A sudah PASS: I2S v6.1.5 → MAX98357A → speaker normal.

Log terbaru justru menunjukkan masalah pada jalur RX/audio:

```text
fragments=379
dropped_frag=138
seq_err=121
buffer_drop=17
```

dan audio queue mengalami drop:

```text
Audio PCM drop terukur: 1002 byte
Audio PCM drop terukur: 1018 byte
Audio PCM drop terukur: 994 byte
```

Karena itu jalur lama tidak lagi dianggap sebagai arsitektur streaming yang benar.

## Keputusan v7.2.0

Jalur audio diubah menjadi pipeline producer/consumer yang lebih mirip pola Gemini Live:

```text
Gemini Live WSS
      ↓
WebSocket callback
      ↓
RAW RX fragment queue
      ↓
RX worker
      ↓
JSON / Base64 streaming decode
      ↓
PCM16 audio ring buffer
      ↓
audio_playback task
      ↓
I2S v6.1.5
      ↓
MAX98357A
      ↓
speaker
```

### Aturan utama

WebSocket callback **tidak boleh lagi**:

- decode Base64;
- memproses JSON audio;
- menulis PCM ke audio ring;
- menunggu audio playback.

Callback hanya:

1. menerima fragment;
2. menyalin fragment ke slot;
3. memasukkan metadata fragment ke RX queue.

Semua pekerjaan berat dipindahkan ke `ws_rx` worker.

## File yang diubah

### `components/websocket/include/websocket_internal.h`

`ws_rx_command_t` sekarang membawa:

- connection generation;
- total payload length;
- payload offset;
- fragment length;
- slot ID;
- opcode;
- pointer data.

Artinya RX queue sekarang merupakan **transport fragment**, bukan queue message JSON lengkap.

### `components/websocket/websocket_rx.cpp`

Arsitektur lama yang melakukan streaming Base64/audio dari jalur WebSocket event callback diganti.

Sekarang:

```text
WEBSOCKET_EVENT_DATA
      ↓
websocket_rx_enqueue_data()
      ↓
raw_q
      ↓
rx_task()
      ↓
worker()
      ↓
stream_feed()
      ↓
queue_audio_pcm()
```

Fragment tetap mengikuti `payload_offset` sehingga urutan WebSocket dapat diverifikasi di worker.

Jika RX worker sedang sibuk, callback menggunakan backpressure terbatas daripada langsung membuang fragment.

## Mengapa perubahan ini dilakukan

Implementasi sebelumnya menjalankan Base64 decode dan `queue_audio_pcm()` dari jalur callback ketika payload audio besar.

Itu membuat jalur penerimaan WebSocket dan jalur audio saling mengganggu.

Dengan v7.2.0, producer WebSocket dan consumer audio dipisahkan:

```text
NETWORK PRODUCER
      ↓
   RX QUEUE
      ↓
  RX WORKER
      ↓
 AUDIO RING
      ↓
PLAYBACK/I2S
```

Ini adalah perubahan arsitektur, bukan tuning I2S.

## Yang TIDAK diubah

- I2S v6.1.5 tetap LOCKED.
- INMP441 tetap 16 kHz.
- MAX98357A tetap.
- Gemini output PCM tetap 24 kHz PCM16.
- Audio ring tetap 48 KiB.
- Playback priority v7.1.3 tetap priority 3.
- OLED tetap tidak digunakan selama audio stability test.
- Volume processing tetap OFF / 100%.

Google Live API sendiri menggunakan WebSocket stateful dan audio output PCM16 24 kHz; best practice Live API juga menekankan streaming real-time, chunk kecil, dan client-side audio buffering/interruption handling. citeturn1search0turn1search3

## Statistik yang WAJIB dibuktikan pada hardware test

Tahap berikutnya harus menghasilkan hubungan yang jelas:

```text
RX fragments
    ↓
RX queue high-water / drop
    ↓
PCM received
    ↓
PCM queued
    ↓
PCM played
    ↓
pending
    ↓
dropped
```

Target minimum satu turn:

```text
dropped_frag = 0
seq_err       = 0
buffer_drop   = 0
queue_drop    = 0
PCM dropped   = 0
pending       = 0
```

Jika target belum tercapai, jangan menyentuh I2S. Cari titik pertama tempat data hilang.

## Status

**v7.2.0 CODE:** perubahan arsitektur RX sudah ditulis langsung ke branch `main` project.

**BUILD:** belum dilakukan.

**FLASH/HARDWARE:** belum dilakukan.

**BASELINE FINAL:** belum ditetapkan.

## Langkah berikutnya

1. Build.
2. Jika build gagal, perbaiki compile error tanpa mengubah I2S.
3. Flash hardware.
4. Jalankan satu turn Gemini.
5. Ambil statistik RX → queue → PCM → ring → I2S.
6. Bandingkan dengan log v7.1.3 dan log terbaru.
7. Baru setelah hasil hardware stabil, tetapkan baseline baru.

**Penting:** sebelum perubahan berikutnya, wajib membaca CHANGELOG_PROJECT.md dan addendum ini terlebih dahulu agar troubleshooting tidak kembali ke awal.
