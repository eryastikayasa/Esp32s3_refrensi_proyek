# ESP32-S3 Project Reference

## AI Project Memory / Reference Repository

Repository ini adalah **pusat dokumentasi dan konteks proyek** untuk proyek ESP32-S3 Voice Commander.

Repository ini **bukan repository firmware** dan tidak digunakan untuk menyimpan salinan source code.

### Repository Firmware

`eryastikayasa/esp32s3_voice_geminiproject`

urlOpen firmware repositoryhttps://github.com/eryastikayasa/esp32s3_voice_geminiproject

## Tujuan Repository Ini

Repository ini dibuat agar AI maupun pengembang dapat memahami proyek secara konsisten ketika melanjutkan pekerjaan, meskipun sesi percakapan atau konteks sebelumnya sudah tidak tersedia.

Dokumentasi di sini menyimpan:

- arsitektur dan konfigurasi penting
- hardware dan pinout
- baseline yang sudah terbukti
- keputusan teknis
- sejarah perkembangan versi
- kondisi proyek terbaru
- masalah yang masih aktif
- aturan kerja saat melakukan debugging dan tuning

## File Utama

### `PROJECT_CONTEXT.md`

Pengetahuan proyek yang relatif stabil.

Berisi arsitektur, hardware, audio, Gemini Live/WebSocket, baseline, riwayat versi penting, serta aturan yang harus dipatuhi AI.

➡️ **Baca ini untuk memahami proyek secara keseluruhan.**

### `CURRENT_STATE.md`

Kondisi proyek **saat ini**.

Berisi status setiap subsistem, masalah yang sedang aktif, hasil pengujian terbaru, prioritas, dan tindakan berikutnya.

➡️ **Baca ini setelah `PROJECT_CONTEXT.md` untuk mengetahui posisi pekerjaan terakhir.**

### `CHANGELOG_PROJECT.md`

Riwayat perkembangan proyek dari versi ke versi dan perubahan penting yang telah dilakukan.

### `AI_INSTRUCTIONS.md`

Aturan operasional untuk AI ketika melakukan audit, debugging, tuning, atau perubahan terhadap repository firmware.

## Cara AI Menggunakan Repository Ini

Urutan yang disarankan:

1. Baca `PROJECT_CONTEXT.md`.
2. Baca `CURRENT_STATE.md`.
3. Jika perlu memahami sejarah perubahan, baca `CHANGELOG_PROJECT.md`.
4. Baca `AI_INSTRUCTIONS.md` sebelum melakukan perubahan pada firmware.
5. Audit repository firmware untuk memastikan kondisi kode aktual.
6. Gunakan log/runtime terbaru sebagai bukti utama.
7. Setelah perubahan yang terverifikasi, perbarui dokumentasi kondisi proyek.

## Prinsip Source of Truth

Ada dua repository dengan fungsi berbeda:

| Repository | Fungsi |
|---|---|
| `esp32s3_voice_geminiproject` | Source code firmware dan kondisi kode aktual |
| `Esp32s3_refrensi_proyek` | Konteks, keputusan, baseline, status, dan sejarah proyek |

Jika dokumentasi dan kode berbeda, **kode aktual + hasil build/test/runtime harus diverifikasi terlebih dahulu** sebelum dokumentasi diperbarui.

## Baseline Penting

### Audio v6.1.5 — LOCKED

Baseline audio v6.1.5 telah terbukti bekerja untuk microphone dan MAX98357A speaker.

Baseline ini tidak boleh diubah secara sembarangan. Tuning berikutnya harus menjaga fungsi yang telah terbukti atau menyediakan bukti pengujian yang jelas jika baseline memang perlu diganti.

## Kondisi Terakhir

Per 20 Agustus 2026:

- Gemini WebSocket sudah berhasil terhubung.
- Audio baseline v6.1.5 tetap menjadi baseline teknis.
- Audio/runtime tuning sedang berlangsung.
- OLED/I2C masih menunjukkan `I2C software timeout` dan sedang dipantau sebagai masalah terpisah.
- Pekerjaan berikutnya harus berdasarkan log perangkat terbaru.

Lihat [`CURRENT_STATE.md`](CURRENT_STATE.md) untuk kondisi terbaru.

## Aturan Singkat

- Jangan membuat branch baru kecuali diminta.
- Jangan mengubah audio baseline v6.1.5 tanpa alasan dan verifikasi.
- Hindari refactor besar ketika sedang melakukan debugging terarah.
- Audit kondisi repository sebelum mengedit.
- Build/test setelah perubahan penting.
- Jangan menyatakan masalah selesai tanpa bukti.
- Perbarui `CURRENT_STATE.md` setelah perubahan penting yang sudah diverifikasi.

---

**Purpose:** menjaga konteks proyek tetap tersedia, terstruktur, dan dapat dibaca AI saat pekerjaan dilanjutkan.