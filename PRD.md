# Product Requirement Document (PRD)

**Nama Produk:** Sistem Manajemen & Tagihan Air RT (WaterBill RT)
**Tanggal:** 6 September 2026
**Status:** Revision 1.4 (Final)
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

### Dari Revisi 1.3 ke 1.4
- **Aksi warga (upload foto meteran & bukti transfer) dipindah dari AppSheet ke Google Form** — Google Form gratis tanpa batas, sementara app AppSheet publik tanpa-login ternyata tetap dihitung berbayar per device (lihat Bagian 5). Bot otomasi (OCR, hitung Pemakaian) tetap jalan karena menempel ke Google Sheets, bukan ke Form-nya.
- Layar "Riwayat Tagihan Saya" untuk warga (in-app) dihapus — warga cukup membaca ulang pesan WA yang sudah dikirim tiap periode untuk melihat riwayat, sesuai prinsip "warga tanpa app terpisah".
- **Akses login AppSheet disederhanakan jadi 1 user saja: Ketua RT (Sulastyo)** — mengubah status Lunas, edit Tab 2, dan semua aksi di sistem dilakukan oleh Ketua RT. Bendahara & Sekretaris tetap terima laporan via WA (F10) tapi tidak login ke sistem; Bendahara lapor hasil cek saldo ke Ketua RT via WA untuk diinput.
- Ditambahkan Bagian 5: Estimasi Biaya Operasional.

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

**Akses edit tab ini terbatas** (lihat F11) — hanya Ketua RT, pakai login akun Google (bukan sistem Admin terpisah).

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

Lihat definisi 4 status di Tab 1. Perubahan status dilakukan manual oleh Ketua RT lewat AppSheet — tidak ada perubahan status otomatis oleh sistem. Bendahara/Sekretaris yang mengusulkan perubahan (misal warga sudah menunggak lama) cukup menyampaikan ke Ketua RT via WA.

### F7. Pencatatan Mandiri Meteran Air (OCR + Flag Anomali)

Berlaku untuk warga dengan Status_Berlangganan `Aktif` atau `Menunggak`.

1. Warga menerima reminder WA untuk foto meteran (jadwal & pemicu di F9), berisi link Google Form.
2. Warga buka Google Form, foto meteran air pakai kamera HP — ini satu-satunya langkah dari warga, tidak ada langkah lanjutan yang perlu ditunggu/dikonfirmasi.
3. Sistem membaca angka meteran dari foto secara otomatis (OCR, dijalankan via bot otomasi yang menempel ke Google Sheets hasil Form), lalu langsung:
   - Mengisi Meter_Akhir.
   - Mengambil Meter_Awal dari Meter_Akhir bulan sebelumnya.
   - Menghitung Pemakaian dan Total_Tagihan.
   - Mengaktifkan Flag_Perlu_Dicek jika Pemakaian tidak wajar (negatif atau jauh di luar rata-rata bulan sebelumnya).
4. Ketua RT mendapat notifikasi kalau ada Flag_Perlu_Dicek, cek manual (lihat foto meteran) sebelum tagihan diteruskan ke warga. Kalau tidak ada flag, tagihan otomatis diteruskan (lihat F9).

Catatan teknis & risiko yang diterima: akurasi OCR bervariasi tergantung jenis meteran (meteran digital/LCD lebih akurat dibanding meteran analog roller-dial). Tidak ada langkah konfirmasi warga (dipertimbangkan tapi dihapus karena kalau dikirim manual per-warga akan membebani Ketua RT, lihat Change Log 1.4) — Flag_Perlu_Dicek jadi satu-satunya jaring pengaman. Konsekuensinya: OCR yang salah baca tapi hasilnya masih "kelihatan wajar" (misal salah 1 digit tapi angkanya tetap masuk akal) berpotensi lolos tanpa disadari. Risiko ini diterima demi kesederhanaan operasional.

### F8. Pengelolaan Status Pembayaran & Konfirmasi

Flow Pembayaran Non-Tunai:
1. Warga menerima notifikasi tagihan via WA beserta daftar kanal pembayaran (dari Tab 2) — dikirim setelah F7 selesai (lihat F9 untuk jadwal).
2. Warga transfer via kanal pilihannya.
3. Warga mengunggah foto bukti transfer via Google Form, kapan pun — termasuk setelah lewat tanggal pengingat. Sistem selalu menerima unggahan, tidak ada penolakan karena telat.
4. Status tagihan otomatis berubah menjadi **Menunggu Verifikasi**.
5. Bendahara mengecek saldo masuk secara manual, lapor ke Ketua RT via WA, lalu Ketua RT mengubah status menjadi **Lunas** di AppSheet (sistem otomatis mengisi Diverifikasi_Oleh dan Tgl_Verifikasi).

Jika ada selisih nominal pembayaran atau bukti transfer meragukan, Bendahara/Ketua RT menyelesaikannya langsung dengan warga via WA — tidak ada mekanisme otomatis di sistem untuk kasus ini.

### F9. Notifikasi WA Terintegrasi (2 Tahap)

Semua tenggat di bawah ini adalah **pengingat, bukan blocker** — sistem selalu menerima aksi warga kapan pun, terlambat atau tidak.

**Cara pengiriman (dipilih demi biaya Rp0 — lihat Bagian 5):** sistem menyiapkan daftar siapa yang perlu diingatkan dan teks pesannya secara otomatis, tapi pengiriman ke WA dilakukan **Ketua RT lewat 1 tap broadcast** (link `wa.me` dengan teks sudah terisi otomatis, tinggal kirim ke Grup WA RT atau broadcast list) — bukan pengiriman otomatis per-warga. Kalau nanti mau full-otomatis per-warga, ini bisa diganti WA gateway berbayar tanpa mengubah skema data.

**Tahap 1 — Reminder Upload Meteran**
- Pemicu: tanggal 25 setiap bulan, untuk warga `Aktif`/`Menunggak` yang belum upload meteran periode berjalan.
- Diulang sampai tanggal 30 kalau belum upload (misal reminder harian atau tiap 2 hari) — Ketua RT tinggal broadcast ulang link yang sama.
- Kalau warga upload setelah tanggal 30 (bahkan masuk bulan berikutnya), sistem tetap terima — proses lanjut ke Tahap 2 seperti biasa.

Format pesan:
```
Yth. Bapak/Ibu [Nama_Warga]
Mohon foto meteran air Bapak/Ibu untuk periode [Periode].
Batas waktu: tanggal 30 [Bulan] (boleh menyusul jika terlambat).
[Link upload foto meteran]
```

**Tahap 2 — Tagihan & Reminder Bayar**
- Pemicu: **tanggal 1 bulan berikutnya** (1 hari setelah jendela upload meteran tutup tgl 30) — satu broadcast gabungan ke semua warga yang tagihannya sudah selesai dihitung (F7), sama seperti mekanisme Tahap 1. Ini menggantikan rencana "H+1 per-individu" yang tidak praktis untuk dikirim manual satu-satu oleh Ketua RT.
- Deadline bayar: tanggal 5 bulan berikutnya.
- Warga yang telat upload meteran (baru terhitung setelah tanggal 1) otomatis tidak ikut broadcast utama — masuk daftar kecil terpisah yang Ketua RT kirimkan menyusul begitu sempat (jumlahnya diharapkan sedikit karena pengecualian, bukan pola rutin).

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

### F11. Pengelolaan Kanal Pembayaran (Self-Service, 1 User)

- Ketua RT bisa mengubah sendiri isi Tab 2 (Metode_Pembayaran) kapan saja langsung dari HP-nya via AppSheet — tanpa perlu minta bantuan developer.
- Akses AppSheet dibatasi **1 login: akun Google Ketua RT (Sulastyo)**. Bendahara & Sekretaris tidak login ke sistem — mereka menyampaikan perubahan/hasil cek via WA ke Ketua RT untuk diinput (lihat Change Log 1.4 untuk konsekuensi operasionalnya).
- Kalau Ketua RT berganti: pindahkan login akun Google ke Ketua RT baru, lalu update baris Atas_Nama/No_Rekening_HP di Tab 2.
- **Bukan** integrasi API e-wallet sungguhan — uang tidak lewat sistem ini, hanya nomor tujuan transfer yang ditampilkan ke warga, cuma bisa diubah sendiri oleh Ketua RT.

### F12. Tampilan Aplikasi (Frontend)

**Untuk Warga (tanpa login, tanpa app):**
Semua interaksi warga cukup lewat **WhatsApp + Google Form**, tidak ada aplikasi/tampilan terpisah yang harus dibuka-buka:
1. Terima pesan WA (broadcast dari Ketua RT, F9) berisi link Google Form.
2. Isi Form sesuai konteks: **Upload Foto Meteran** (F7) atau **Upload Bukti Transfer** (F8) — form pendek, cukup pilih kanal + foto/kamera + submit.
3. Untuk lihat riwayat tagihan, warga cukup scroll ulang pesan WA yang sudah pernah dikirim — tidak ada layar "riwayat" terpisah (dihapus di Revisi 1.4 karena versi AppSheet-nya berbayar per device, lihat Bagian 5).

**Untuk Ketua RT (1 login akun Google):**
Akses penuh di AppSheet: semua tab data, edit Tab 2 (F11), tombol "Tandai Lunas" per tagihan, notifikasi Flag_Perlu_Dicek, tombol broadcast (link wa.me siap kirim untuk F9), dan tampilan ringkasan laporan (versi visual dari F10).

---

## 4. Yang Sengaja Tidak Dibuat (Scope Cut)

Supaya aplikasi tetap sederhana untuk warga dan pengurus yang gaptek:
- Tidak ada deteksi otomatis foto bukti transfer palsu/duplikat — cukup pemeriksaan manual Bendahara.
- Tidak ada fitur pembayaran sebagian/kelebihan bayar otomatis — diselesaikan manual via WA.
- Tidak ada sistem audit log terpisah — cukup 2 kolom (Diverifikasi_Oleh, Tgl_Verifikasi) di Tab 3.
- Tidak ada penguncian/penalti otomatis karena telat — hanya notifikasi pengingat, di semua tahap (upload meteran maupun bayar tagihan).
- Tidak ada integrasi API payment gateway (Midtrans/Xendit/dll) atau API resmi e-wallet — pembayaran tetap manual transfer + verifikasi manual, kanal cukup diatur sendiri oleh Ketua RT (F11).
- Tidak ada sistem login/Admin panel custom — akses cukup 1 akun Google (Ketua RT), warga selamanya tanpa login/tanpa app (cukup WA + Google Form).
- Tidak ada layar "Riwayat Tagihan Saya" untuk warga — cukup baca ulang WA (dihapus di 1.4 untuk hindari biaya AppSheet publik).
- Tidak ada konfirmasi 1-tap warga atas hasil OCR — cukup Flag_Perlu_Dicek (dihapus di 1.4 karena kalau dikirim manual per-warga membebani Ketua RT; lihat F7).
- Tidak ada pengiriman WA otomatis per-individu — cukup broadcast manual 1-tap oleh Ketua RT (lihat F9 & Bagian 5), supaya biaya operasional tetap Rp0.

---

## 5. Estimasi Biaya Operasional

| Komponen | Kebutuhan | Biaya |
|---|---|---|
| Google Sheets | Data | Gratis |
| Google Form | Upload meteran & bukti transfer oleh warga | Gratis, unlimited respons |
| Google Cloud Vision API (OCR) | Baca angka meteran | Gratis dalam kuota (±1.000 pembacaan/bulan) — cukup untuk skala RT puluhan rumah |
| AppSheet | 1 login (Ketua RT), plan Starter | **$5/bulan (±Rp 80.000/bulan)** |
| Pengiriman WA | Broadcast manual 1-tap oleh Ketua RT (link wa.me) | Gratis |
| **Total** | | **±$5/bulan (±Rp 80.000/bulan)** |

Catatan: angka Rp memakai kurs perkiraan ±Rp16.000/USD dan harga AppSheet berdasarkan pricing publik yang ditemukan lewat pencarian web (bukan hasil akses langsung ke halaman resminya, karena sempat terblokir) — disarankan Bapak cek ulang di about.appsheet.com/pricing sebelum commit anggaran, dan kuota gratis Google Cloud Vision dicek langsung di cloud.google.com/vision/pricing karena bisa berubah.

**Opsi upgrade di masa depan (tidak dibangun sekarang, tinggal aktifkan kalau perlu):**
- Tambah Bendahara sebagai login ke-2 → +$5/bulan, supaya verifikasi pembayaran tidak bergantung 1 orang.
- Ganti broadcast manual dengan WA gateway berbayar (Fonnte/Wablas, ±Rp25.000-100.000/bulan) → reminder & konfirmasi jadi otomatis per-warga.
