# AI Instructions — ESP32-S3 Project

## Bahasa dan Cara Menjelaskan

Gunakan **bahasa Indonesia yang sederhana dan bahasa sehari-hari**.

Tujuannya supaya penjelasan mudah dipahami dan tidak terlalu banyak istilah teknis yang tidak perlu.

Jika istilah teknis memang diperlukan:

- jelaskan artinya dengan singkat
- gunakan contoh sederhana jika membantu
- langsung ke inti masalah
- hindari kalimat yang terlalu panjang
- jangan memakai bahasa yang terlalu formal atau rumit

Saat menjelaskan hasil debugging, gunakan pola sederhana:

**Masalah → Penyebab yang ditemukan → Perubahan → Hasil → Langkah berikutnya**

## Cara Bekerja dengan Repository

1. Baca `PROJECT_CONTEXT.md` sebelum mulai bekerja.
2. Baca `CURRENT_STATE.md` untuk mengetahui kondisi terbaru.
3. Baca `CHANGELOG_PROJECT.md` jika perlu melihat sejarah perubahan.
4. Audit repository firmware sebelum mengubah kode.
5. Gunakan log terbaru sebagai bukti utama.
6. Jangan menebak kondisi firmware jika bisa diperiksa langsung.
7. Jangan membuat branch baru kecuali diminta.
8. Kerjakan langsung pada branch yang sedang digunakan.
9. Lakukan perubahan sekecil dan sesederhana mungkin saat debugging.
10. Setelah perubahan penting, lakukan build/test dan periksa hasilnya.
11. Jangan menyatakan masalah sudah selesai tanpa bukti dari build, test, atau runtime log.

## Aturan Audio

- Audio baseline **v6.1.5 adalah baseline yang terkunci**.
- Jangan mengubahnya sembarangan.
- Jika tuning audio diperlukan, jelaskan bagian mana yang diubah dan kenapa.
- Pastikan perubahan tidak merusak fungsi microphone INMP441 dan speaker MAX98357A yang sudah terbukti bekerja.

## Dokumentasi Setelah Perubahan

Setelah perubahan yang sudah diuji:

- Perbarui `CURRENT_STATE.md` jika kondisi proyek berubah.
- Tambahkan ke `CHANGELOG_PROJECT.md` jika perubahan tersebut merupakan milestone penting.
- Perbarui `PROJECT_CONTEXT.md` hanya jika hasilnya menjadi pengetahuan atau keputusan proyek yang bersifat permanen.

Jangan mengubah semua file dokumentasi untuk setiap perubahan kecil.

## Prioritas Debugging

Utamakan:

1. Bukti dari log perangkat.
2. Kondisi kode yang benar-benar ada di repository.
3. Hasil build/test.
4. Perubahan minimal.
5. Menjaga baseline yang sudah terbukti.

Pisahkan masalah antar subsistem sampai ada bukti bahwa masalah tersebut saling berhubungan.

Contoh: jangan langsung menganggap I2C timeout sebagai masalah audio hanya karena muncul pada log yang sama.

## Tujuan Utama

AI harus membantu melanjutkan proyek tanpa kehilangan konteks, tetapi tetap menjaga agar perubahan pada firmware aman, terukur, dan mudah dipahami.