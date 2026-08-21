# ESP32-S3 Voice Commander — Changelog Addendum v7.2.3

## v7.2.3 — Xiaozhi-Style Bidirectional Audio Pipeline

**Tanggal:** 21 Agustus 2026

### Dasar perubahan

v7.2.1 sudah memisahkan kepemilikan audio ke `audio_stream`, tetapi jalur RX masih terlalu banyak pekerjaan dilakukan di area WebSocket.

Keputusan arsitektur untuk v7.2.3:

- Protocol tetap mengikuti Gemini Live.
- Pola task/queue audio mengadopsi pola Xiaozhi ESP32.
- Transport WebSocket tidak menjadi tempat playback berat.
- Audio input dan audio output dipisahkan menjadi task/queue yang jelas.

### Arsitektur yang dituju

```text
AUDIO INPUT
INMP441
  ↓
audio_input task
  ↓
input queue
  ↓
WebSocket TX worker
  ↓
Gemini Live

AUDIO OUTPUT
Gemini Live
  ↓
WebSocket RX
  ↓
RX audio queue
  ↓
audio decode task
  ↓
PCM16
  ↓
audio_stream output ring
  ↓
playback task
  ↓
I2S v6.1.5
  ↓
MAX98357A
```

### Implementasi v7.2.3

1. `audio_input` menjadi task khusus untuk mengambil PCM microphone.
2. Data microphone dipisahkan dari pekerjaan WebSocket.
3. WebSocket TX bertugas mengirim audio input ke Gemini Live.
4. WebSocket RX fokus pada transport dan penerusan data audio.
5. Decode Base64/audio dipisahkan dari pekerjaan transport RX.
6. Audio Gemini diproses melalui jalur decode sebelum masuk ke output buffer.
7. Hasil decode berupa PCM16 diteruskan ke `audio_stream`.
8. `audio_stream` tetap menjadi pemilik output ring buffer.
9. Playback task menjadi pemilik konsumsi PCM menuju I2S.
10. Generation check dipertahankan agar data dari turn/koneksi lama tidak masuk ke playback baru.
11. `turnComplete` tidak langsung menghentikan playback; audio yang masih berada di jalur buffer harus di-drain terlebih dahulu.
12. Statistik RX → decode → buffer → playback dipertahankan untuk pembuktian hardware.

### Protocol vs pola implementasi

**Protocol:** Gemini Live.

**Pola task/queue:** mengadopsi pola Xiaozhi.

Perubahan ini tidak mengubah protocol Gemini. Yang diubah adalah cara ESP32-S3 mengelola pekerjaan audio secara internal agar transport, decode, buffering, dan playback tidak saling menghambat.

### Hardware yang tetap LOCKED

Jangan mengubah baseline hardware berikut selama pengujian arsitektur:

```text
INMP441
MAX98357A
I2S v6.1.5
PCM16
Mic 16 kHz
Speaker 24 kHz
```

Test tone v6.1.5 tetap menjadi baseline hardware.

### Statistik wajib saat hardware test

```text
RX:
fragments
dropped_frag
seq_err
buffer_drop
queue_drop

DECODE:
decode received
decode dropped
decode queue_hwm

AUDIO:
received
queued
played
dropped
pending
underrun

PLAYBACK:
write_calls
PLAYBACK COMPLETE
```

### Kriteria keberhasilan

Audio belum boleh dinyatakan berhasil hanya karena suara keluar dari speaker.

Minimal harus dibuktikan:

```text
RX stabil
  ↓
decode tidak kehilangan audio
  ↓
queue tidak penuh terus-menerus
  ↓
ring buffer tidak drop
  ↓
playback tidak underrun
  ↓
pending kembali 0
  ↓
played ≈ queued
  ↓
PLAYBACK COMPLETE
```

### Status

**CODE: IMPLEMENTASI v7.2.3 DICATAT**

**BUILD/HARDWARE: tetap harus diverifikasi**

Catatan ini tidak mengklaim bahwa suara Gemini sudah bebas sendat. Keberhasilan final harus dibuktikan dengan hardware test dan statistik jalur streaming.

### Keputusan untuk pengembangan berikutnya

```text
Protocol = Gemini Live
Pola task/queue = Xiaozhi
Hardware I2S = baseline v6.1.5 LOCKED
Pembuktian = RX → decode → buffer → I2S
```

Jangan kembali melakukan tuning I2S sebelum statistik jalur streaming membuktikan bahwa masalah memang berada di hardware.

---

**Catatan:** File ini sengaja dibuat sebagai addendum terpisah agar `CHANGELOG_PROJECT.md` utama tidak ditimpa atau diubah.
