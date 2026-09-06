# Product Requirement Document (PRD)

**Nama Produk:** Sistem Manajemen & Tagihan Air RT (WaterBill RT)
**Tanggal:** 6 September 2026
**Status:** Revision 1.3 (Final)
**Target Pengguna:** Warga RT, Pengurus RT (Ketua, Sekretaris, Bendahara), Petugas Penagihan

Prinsip desain: aplikasi harus sederhana dan mudah dipakai warga yang gaptek/lansia. Tidak ada fitur yang menambah langkah atau menghalangi warga membayar — semua batas waktu bersifat pengingat, bukan penghalang. Warga tidak pernah perlu login atau mengetik data teknis — cukup foto dan satu kali tap konfirmasi.

---

## 1. Change Log

### Dari Revisi 1.1 ke 1.2
- Status Berlangganan "Dicabut" dan "Menunggak" dipisah — sebelumnya digabung dan salah logika (warga nunggak tapi masih pakai air jadi ditagih Rp0, seharusnya tidak).
- Status Pembayaran tagihan disederhanakan jadi 3 nilai (dihapus nilai "Menunggak" yang tadinya duplikat nama dengan status pelanggan).
- Kanal pembayaran (GoPay/OVO/DANA/QRIS/Bank) dijadikan data referensi bertabel (bukan enum tetap), supaya nama kanal di pesan WA selalu sinkron dengan yang dipilih warga.
- Ditambahkan 2 kolom pencatatan verifikasi (siapa & kapan) — cukup sebagai catatan sederhana, bukan sistem audit terpisah.
- Keterlambatan (upload bukti transfer / kirim tagihan) ditegaskan sebagai pengingat, bukan blocker: sistem tetap menerima, hanya menampilkan notifikasi keterlambatan di aplikasi.
- Fitur yang sengaja tidak dibuat: deteksi fraud bukti transfer otomatis, penanganan pembayaran sebagian/lebih otomatis, sistem audit log formal, integrasi API payment gateway (GoPay/OVO/DANA business API) — dinilai terlalu kompleks untuk skala RT dan butuh badan usaha + backend + maintenance rutin.

### Dari Revisi 1.2 ke 1.3
- Pencatatan meteran diubah dari "petugas baca manual" menjadi **warga foto sendiri meterannya**, dibaca otomatis oleh OCR, lalu dikonfirmasi 1 tap oleh warga (bukan ketik manual, bukan juga full-otomatis tanpa cek).
- Jadwal notifikasi dipecah jadi 2 tahap: reminder upload meteran (tgl 25–30) dan reminder bayar tagihan (H+1 setelah warga upload, deadline tgl 5) — menggantikan jadwal lama yang kirim tagihan serentak tgl 5 ke semua warga.
- Ditambahkan flag "⚠️ Perlu Dicek" untuk pemakaian yang tidak wajar (hasil OCR meleset atau memang ada anomali), sebagai jaring pengaman kedua.
- Ditambahkan fitur pengurus RT bisa ubah sendiri detail kanal pembayaran (Tab Metode_Pembayaran) kapan saja — pakai fitur bawaan AppSheet (Security filter), bukan sistem Admin/kredensial custom.
- Ditegaskan pemisahan akses: warga selamanya tanpa login (link personal via WA), pengurus (3 orang) login pakai akun Google yang sudah ada di HP mereka — bukan kredensial baru yang harus dihafal.
- Ditambahkan spesifikasi tampilan aplikasi (frontend) untuk warga dan pengurus.

---

## 2. Struktur Database & Skema Data (Google Sheets)

### Tab 1: Pelanggan (Master Data)

| Kolom | Keterangan |
|---|---|
| ID_Pelanggan (PK) | |
| Nama_Warga | |
| Alamat_Rumah | |
| No_WhatsApp | Kontak notifikasi & sumber link personal warga (bukan kredensial login) |
| Status_Berlangganan | Enum: `Aktif`, `Non-Aktif`, `Menunggak`, `Dicabut` |

**Arti tiap status:**
- **Aktif** — pakai air normal, tagihan bulanan normal, dapat reminder upload meteran tiap bulan.
- **Non-Aktif** — rumah kosong sementara. Tetap kena biaya abonemen jika disepakati RT, kalau tidak disepakati tagihan Rp0. Tidak perlu upload meteran (tidak pakai air).
- **Menunggak** — masih tersambung dan masih pakai air, tapi ada tunggakan yang belum lunas. **Tagihan tetap berjalan normal (bukan Rp0)**, tetap dapat reminder upload meteran. Status ini ditandai manual oleh Bendahara/Ketua RT ketika tunggakan sudah berlangsung lama (kebijakan RT, misal >2 bulan), untuk jadi dasar keputusan lanjutan (termasuk apakah perlu dicabut).
- **Dicabut** — sambungan air sudah diputus/disegel secara fisik oleh petugas RT. Tagihan otomatis Rp0, tidak perlu upload meteran.

### Tab 2: Metode_Pembayaran (Setting Kanal Pembayaran RT)

| Kolom | Keterangan |
|---|---|
| ID_Kanal (PK) | |
| Nama_Kanal | Contoh: GoPay, Shopee, DANA, OVO, LinkAja, Bank BCA, Tunai |
| No_Rekening_HP | Nomor tujuan transfer/e-wallet |
| Atas_Nama | Nama pemegang akun, misal "Sulastyo (Ketua RT)" |

Semua kanal yang tampil di pesan WA tagihan dan yang bisa dipilih warga saat konfirmasi pembayaran diambil langsung dari tab ini.

**Akses edit tab ini terbatas** (lihat F11) — hanya Ketua RT/Bendahara, pakai Security filter bawaan AppSheet berbasis akun Google, bukan sistem Admin terpisah.

### Tab 3: Tagihan_Bulanan (Data Transaksi)

| Kolom | Keterangan |
|---|---|
| ID_Tagihan (PK) | |
| Periode | Contoh: Agustus 2026 |
| ID_Pelanggan (FK) | |
| Foto_Meteran | Link foto meteran yang diupload warga |
| Meter_Awal | Otomatis = Meter_Akhir periode sebelumnya. Untuk bulan pertama warga terdaftar, diisi manual sekali oleh petugas sebagai baseline. |
| Meter_Akhir | Hasil bacaan OCR atas Foto_Meteran, dikonfirmasi 1 tap oleh warga (lihat F7). Bisa dikoreksi manual oleh warga kalau OCR salah baca. |
| Pemakaian | Otomatis = Meter_Akhir − Meter_Awal |
| Flag_Perlu_Dicek | Otomatis aktif kalau Pemakaian negatif atau jauh di luar wajar dibanding rata-rata bulan sebelumnya — muncul sebagai peringatan ke Bendahara sebelum tagihan dikirim |
| Total_Tagihan | Dihitung dari Pemakaian sesuai tarif berlaku |
| Status_Pembayaran | Enum: `Belum Bayar`, `Menunggu Verifikasi`, `Lunas` |
| ID_Kanal_Dipilih (FK) | Kanal yang dipilih warga saat bayar, merujuk ke Tab 2 |
| Tgl_Pembayaran | |
| Bukti_Transfer | Link foto bukti transfer |
| Diverifikasi_Oleh | Nama Bendahara yang mengubah status jadi Lunas |
| Tgl_Verifikasi | |

---

## 3. Spesifikasi Fitur (Functional Requirements)

### F6. Pengelolaan Status Berlangganan Warga

Lihat definisi 4 status di Tab 1. Perubahan status dilakukan manual oleh Bendahara/Ketua RT lewat AppSheet — tidak ada perubahan status otomatis oleh sistem.

### F7. Pencatatan Mandiri Meteran Air (OCR + Konfirmasi)

Berlaku untuk warga dengan Status_Berlangganan `Aktif` atau `Menunggak`.

1. Warga menerima reminder WA untuk foto meteran (jadwal & pemicu di F9).
2. Warga buka link personalnya, foto meteran air pakai kamera HP — ini satu-satunya langkah manual dari warga.
3. Sistem membaca angka meteran dari foto secara otomatis (OCR).
4. Warga melihat hasil bacaan ditampilkan ("Angka terbaca: 01234") dengan 2 tombol: **Ya, Benar** atau **Bukan, Ini Angkanya** (kalau OCR salah baca, warga ketik koreksi angka yang benar — kejadian ini diharapkan jarang, bukan alur utama).
5. Setelah dikonfirmasi, sistem otomatis:
   - Mengisi Meter_Akhir.
   - Mengambil Meter_Awal dari Meter_Akhir bulan sebelumnya.
   - Menghitung Pemakaian dan Total_Tagihan.
   - Mengaktifkan Flag_Perlu_Dicek jika Pemakaian tidak wajar.
6. Bendahara mendapat notifikasi kalau ada Flag_Perlu_Dicek, cek manual sebelum tagihan diteruskan ke warga. Kalau tidak ada flag, tagihan otomatis diteruskan (lihat F9).

Catatan teknis: akurasi OCR bervariasi tergantung jenis meteran (meteran digital/LCD lebih akurat dibanding meteran analog roller-dial). Langkah konfirmasi 1 tap di atas adalah jaring pengaman utama terhadap kesalahan baca ini.

### F8. Pengelolaan Status Pembayaran & Konfirmasi

Flow Pembayaran Non-Tunai:
1. Warga menerima notifikasi tagihan via WA beserta daftar kanal pembayaran (dari Tab 2) — dikirim setelah F7 selesai (lihat F9 untuk jadwal).
2. Warga transfer via kanal pilihannya.
3. Warga mengunggah foto bukti transfer via aplikasi AppSheet, kapan pun — termasuk setelah lewat tanggal pengingat. Sistem selalu menerima unggahan, tidak ada penolakan karena telat.
4. Status tagihan otomatis berubah menjadi **Menunggu Verifikasi**.
5. Bendahara menerima notifikasi, mengecek saldo masuk secara manual, lalu mengubah status menjadi **Lunas** (sistem otomatis mengisi Diverifikasi_Oleh dan Tgl_Verifikasi).

Jika ada selisih nominal pembayaran atau bukti transfer meragukan, Bendahara menyelesaikannya langsung dengan warga via WA — tidak ada mekanisme otomatis di sistem untuk kasus ini.

### F9. Notifikasi WA Terintegrasi (2 Tahap)

Semua tenggat di bawah ini adalah **pengingat, bukan blocker** — sistem selalu menerima aksi warga kapan pun, terlambat atau tidak.

**Tahap 1 — Reminder Upload Meteran**
- Pemicu: tanggal 25 setiap bulan, untuk warga `Aktif`/`Menunggak` yang belum upload meteran periode berjalan.
- Diulang sampai tanggal 30 kalau belum upload (misal reminder harian atau tiap 2 hari).
- Kalau warga upload setelah tanggal 30 (bahkan masuk bulan berikutnya), sistem tetap terima — proses lanjut ke Tahap 2 seperti biasa.

Format pesan:
```
Yth. Bapak/Ibu [Nama_Warga]
Mohon foto meteran air Bapak/Ibu untuk periode [Periode].
Batas waktu: tanggal 30 [Bulan] (boleh menyusul jika terlambat).
[Link upload foto meteran]
```

**Tahap 2 — Tagihan & Reminder Bayar**
- Pemicu: **H+1 setelah warga menyelesaikan F7** (konfirmasi angka meteran) — bukan tanggal tetap untuk semua warga, mengikuti kapan masing-masing warga upload.
- Deadline bayar: tanggal 5 bulan berikutnya. Kalau warga upload meterannya sudah mepet/lewat tanggal 5, tagihan tetap dikirim H+1 seperti biasa, hanya saja sisa waktu sebelum deadline jadi lebih pendek — tidak masalah karena deadline ini pengingat, bukan penutupan kesempatan bayar.

Format pesan:
```
Yth. Bapak/Ibu [Nama_Warga]
Tagihan Air RT Bulan [Periode]:
• Pemakaian: [Pemakaian] m³
• TOTAL TAGIHAN: Rp [Total_Tagihan]
• Status Pembayaran: [Status_Pembayaran]

💳 METODE PEMBAYARAN NON-TUNAI:
[daftar Nama_Kanal + No_Rekening_HP + Atas_Nama dari Tab 2]

Batas waktu: tanggal 5 (boleh menyusul jika terlambat, tetap diproses).
Silakan upload bukti transfer via aplikasi AppSheet setelah membayar.
```

### F10. Laporan Keuangan Lengkap ke Pengurus RT

**Pemicu:** Otomatis dikirim ke Ketua (Sulastyo), Sekretaris (S. Nur Hanafi), dan Bendahara (H. Herman Effendi D.) setiap tanggal 10.

**Definisi "Daftar Warga Belum Bayar" di laporan ini** = warga dengan tagihan periode berjalan yang Status_Pembayaran-nya masih `Belum Bayar` (bukan akumulasi seluruh histori tunggakan). Status_Berlangganan `Menunggak` (keputusan manual jangka panjang) ditampilkan terpisah sebagai info tambahan.

Format Pesan WA Laporan:
```
LAPORAN REKAPITULASI TAGIHAN AIR RT
PERIODE: [Bulan - Tahun]

📊 RINGKASAN STATUS PELANGGAN:
• Aktif: [Jumlah_Aktif] | Non-Aktif: [Jumlah_NonAktif] | Menunggak: [Jumlah_Menunggak] | Dicabut: [Jumlah_Dicabut]

💰 RINGKASAN KEUANGAN & PEMBAYARAN (periode berjalan):
• Total Target Tagihan: Rp [Total_Target]
• Total Terkumpul (Lunas): Rp [Nominal_Lunas]
• Total Belum Bayar: Rp [Nominal_BelumBayar]

📱 RINCIAN KANAL PEMBAYARAN:
[breakdown nominal per Nama_Kanal dari Tab 2, dihitung dari ID_Kanal_Dipilih]

⚠️ DAFTAR WARGA BELUM BAYAR (periode ini):
 * [Nama_Warga_1] - Rp [Nominal_1]
 * [Nama_Warga_2] - Rp [Nominal_2]

Detail & Bukti Transfer dapat diverifikasi di Link Google Sheets.
```

### F11. Pengelolaan Kanal Pembayaran oleh Pengurus (Self-Service)

- Ketua RT/Bendahara bisa mengubah sendiri isi Tab 2 (Metode_Pembayaran) kapan saja langsung dari HP mereka via AppSheet — tanpa perlu minta bantuan developer.
- Akses edit dibatasi pakai fitur bawaan AppSheet (Security filter) berdasarkan akun Google mereka — bukan kredensial/Admin panel custom.
- Kalau pengurus RT berganti: (1) update daftar akun Google yang diizinkan di Security filter, (2) update baris Atas_Nama/No_Rekening_HP di Tab 2 ke pengurus baru.
- **Bukan** integrasi API e-wallet sungguhan — uang tidak lewat sistem ini, hanya nomor tujuan transfer yang ditampilkan ke warga (sama seperti sekarang, cuma bisa diubah sendiri oleh pengurus).

### F12. Tampilan Aplikasi (Frontend)

**Untuk Warga (tanpa login):**
Setiap pesan WA (F9) berisi link personal yang otomatis terfilter ke data mereka sendiri (berbasis ID_Pelanggan, fitur bawaan AppSheet). Hanya 2 layar:
1. **Riwayat Tagihan Saya** — daftar Periode, Pemakaian, Total, Status, dengan indikator warna: 🟢 Lunas, 🟡 Menunggu Verifikasi, 🔴 Belum Bayar.
2. **Foto Meteran / Upload Bukti Transfer** — form sederhana sesuai konteks (upload meteran saat F7, upload bukti transfer saat F8), tombol besar, minim langkah.

**Untuk Pengurus (login akun Google):**
Akses penuh: semua tab data, edit Tab 2 (F11), tombol "Tandai Lunas" per tagihan, notifikasi Flag_Perlu_Dicek, dan tampilan ringkasan laporan (versi visual dari F10).

**Catatan risiko kecil (bukan blocker):** link warga tanpa login berarti siapa pun yang memegang link bisa lihat/upload untuk ID tersebut — risikonya rendah karena data yang ditampilkan setara dengan yang sudah dikirim via WA biasa (tagihan air sendiri, bukan data sensitif).

---

## 4. Yang Sengaja Tidak Dibuat (Scope Cut)

Supaya aplikasi tetap sederhana untuk warga dan pengurus yang gaptek:
- Tidak ada deteksi otomatis foto bukti transfer palsu/duplikat — cukup pemeriksaan manual Bendahara.
- Tidak ada fitur pembayaran sebagian/kelebihan bayar otomatis — diselesaikan manual via WA.
- Tidak ada sistem audit log terpisah — cukup 2 kolom (Diverifikasi_Oleh, Tgl_Verifikasi) di Tab 3.
- Tidak ada penguncian/penalti otomatis karena telat — hanya notifikasi pengingat, di semua tahap (upload meteran maupun bayar tagihan).
- Tidak ada integrasi API payment gateway (Midtrans/Xendit/dll) atau API resmi e-wallet — pembayaran tetap manual transfer + verifikasi manual, kanal cukup diatur sendiri oleh pengurus (F11).
- Tidak ada sistem login/Admin panel custom — akses pengurus cukup pakai akun Google + Security filter bawaan AppSheet, warga selamanya tanpa login.
