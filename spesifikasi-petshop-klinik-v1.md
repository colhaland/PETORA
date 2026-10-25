# Spesifikasi Sistem Operasional Pet Shop dan Klinik Hewan

**Versi 1.0 · 25 Oktober 2026**
**Stack:** Laravel 12 + PHP 8.4 · Filament 4 + Livewire 3 · Blade + Tailwind CSS 4 + Vite · PostgreSQL 16
**Target deployment:** Laravel Forge di VPS Singapura dengan PostgreSQL terkelola

Dokumen ini adalah spesifikasi tunggal untuk Versi 1. Setiap keputusan sudah ditetapkan di Bagian 1 dan berlaku di seluruh dokumen. Dokumen ini tidak memuat roadmap, fase, atau jadwal. Dokumen ini menjelaskan apa yang harus dibangun, aturan bisnisnya, dan kriteria penerimaannya.

---

## 0. Tujuan dan Prinsip

1. Satu aplikasi Laravel untuk pet shop, klinik, rawat inap, grooming, penitipan, keuangan, dan portal pelanggan, dengan satu basis data untuk seluruh cabang.
2. Staf bekerja di panel admin Filament. Pelanggan menggunakan situs Blade yang responsif untuk komputer, tablet, dan ponsel.
3. Stok pet shop dan klinik berada dalam satu inventori per cabang. Stok berpindah melalui transfer.
4. Pembayaran dilakukan secara manual. Tidak ada payment gateway.
5. Setiap hak akses diperiksa di server melalui aturan terpusat. Menyembunyikan tombol bukan kontrol keamanan.
6. Perubahan stok, kas, pembayaran, dan data klinis dilakukan dalam transaksi basis data, dilindungi idempotensi, dan dicatat di jejak audit.
7. Aplikasi hanya memakai bahasa Indonesia untuk antarmuka pengguna.
8. Deployment harus dapat dilakukan oleh satu developer tanpa konfigurasi infrastruktur yang rumit.

---

## 1. Keputusan Terkunci

### 1.1 Keputusan teknologi

| Kode | Topik | Keputusan |
|------|-------|-----------|
| T1 | Versi inti | PHP 8.4, Laravel 12, Composer. Semua kode mengikuti `declare(strict_types=1)`. |
| T2 | Panel admin | Filament 4 di path `/admin`. Filament dipakai untuk seluruh antarmuka staf, termasuk POS sebagai halaman kustom. |
| T3 | Interaktivitas | Livewire 3 (dipasang oleh Filament 4) untuk komponen interaktif di portal pelanggan dan halaman POS. |
| T4 | Frontend pelanggan | Blade, Tailwind CSS 4 melalui plugin Vite, dan Vite sebagai bundler. |
| T5 | Basis data | PostgreSQL 16 dengan ekstensi `btree_gist` dan `pg_trgm`. |
| T6 | Cache, session, antrean | Driver `database`. Redis tidak digunakan pada Versi 1. |
| T7 | Autentikasi pelanggan | Laravel Fortify: pendaftaran, verifikasi email, login, dan reset kata sandi. |
| T8 | Autentikasi staf | Login Filament. MFA TOTP memakai paket Filament Breezy. |
| T9 | Otorisasi | Tabel `permissions`, `role_permissions`, dan `user_permission_grants`, dievaluasi oleh satu kelas `AccessControl`. Tidak memakai paket permission pihak ketiga. |
| T10 | Angka uang | Kelas `Money` berbasis `brick/math` (`BigDecimal`, mode pembulatan `HALF_UP`). Tidak memakai `float` atau `int` PHP untuk kuantitas dengan desimal. |
| T11 | Berkas | Disk `private` dan `public` di server (lokal). Unduhan berkas privat lewat controller yang memeriksa izin. Tidak memakai S3 untuk berkas aktif. |
| T12 | Email | Driver `resend` (paket resmi Resend untuk Laravel). |
| T13 | PDF | `barryvdh/laravel-dompdf`. |
| T14 | Pemantauan error | Sentry Laravel SDK, dengan `send_default_pii = false` dan pembersihan data klinis sebelum kirim. |
| T15 | Cadangan | `spatie/laravel-backup`, dengan tujuan penyimpanan S3-compatible di luar server. |
| T16 | Penjadwalan | Scheduler Laravel dengan satu entri cron. Job dicatat di tabel `job_runs`. |
| T17 | Pengujian | Pest (unit dan fitur), Laravel Dusk (E2E), dan tes basis data dan konkurensi dengan PostgreSQL nyata. |
| T18 | Analisis statis | Laravel Pint (format) dan Larastan level 8 untuk folder `app/Services`, `app/Support`, dan `app/Domain`. |
| T19 | Hosting | Laravel Forge mengelola VPS Ubuntu 24.04 di region Singapura, Nginx, PHP-FPM, SSL, worker antrean, dan cron. |
| T20 | Basis data hosting | PostgreSQL terkelola di region Singapura dengan point-in-time recovery (PITR) aktif. |
| T21 | API publik | Tidak ada REST API publik pada Versi 1. Semua operasi dijalankan melalui Filament, Livewire, dan form Blade. |
| T22 | Penyimpanan logika transaksi | Logika transaksi berada di kelas PHP pada `app/Services`. Fungsi PL/pgSQL hanya dipakai untuk trigger immutability dan constraint. |

### 1.2 Keputusan bisnis

| Kode | Topik | Keputusan |
|------|-------|-----------|
| B1 | Wali hewan | Satu hewan dapat memiliki lebih dari satu wali. Tepat satu wali berstatus utama. Wali utama menerima tagihan dan menyetujui biaya tambahan. Wali sekunder melihat riwayat, membuat janji temu, dan menerima update, tetapi tidak melihat tagihan. |
| B2 | Akun pelanggan | Satu pelanggan memiliki paling banyak satu akun login. Satu akun dapat menjadi wali bagi banyak hewan. Pelanggan walk-in dibuat staf tanpa akun dan dapat ditautkan ke akun melalui undangan verifikasi. |
| B3 | Dokter dan remunerasi | Sistem tidak menghitung gaji atau komisi. Sistem menyediakan laporan layanan per dokter. |
| B4 | Harga dan katalog per cabang | Master produk bersifat global. Harga, status jual POS, status jual online, dan stok minimum ditetapkan per cabang. Tidak ada harga fallback. Produk aktif di cabang wajib memiliki harga di cabang tersebut. |
| B5 | Harga bilangan bulat | Harga jual disimpan sebagai rupiah penuh (`NUMERIC(14,2)` dengan nilai pecahan nol). Total transaksi dibulatkan ke rupiah penuh dengan `HALF_UP`. |
| B6 | Batch dan kedaluwarsa | Produk `MEDICINE` dan produk dengan `track_lot = true` wajib memiliki nomor batch dan tanggal kedaluwarsa sejak penerimaan. Pengambilan stok memakai FEFO. |
| B7 | Pembayaran | POS dan pesanan online wajib lunas. Invoice layanan (klinik, rawat inap, grooming, penitipan) mendukung pembayaran parsial dan menghasilkan piutang. |
| B8 | Kasir dan metode per cabang | Setiap cabang memiliki kasir dan shift sendiri. Daftar metode pembayaran bersifat global, lalu diaktifkan per cabang. Setiap pembayaran tunai terkait shift yang terbuka. |
| B9 | Format nomor dokumen | `{KODE_CABANG}-{JENIS}-{TAHUN}-{NOMOR6}`. Jenis: `POS`, `ONL`, `SVC`, `RFD`, `RTN`, `PO`, `TRF`. Nomor berurutan per cabang, jenis, dan tahun. Contoh: `UTM-POS-2026-000145`. |
| B10 | Kapasitas | Target desain: 150 pengguna per hari, 40 sesi bersamaan, dan 15 transaksi tulis bersamaan. Sistem diuji pada 2× target. |
| B11 | Wilayah dan waktu | Basis data dan aplikasi berada di region Singapura. Waktu disimpan UTC. Tampilan memakai zona waktu cabang: `Asia/Jakarta` (default), `Asia/Makassar`, atau `Asia/Jayapura`. |
| B12 | Mata uang dan pajak | Hanya IDR. Sistem tidak menghitung PPN. Harga yang diinput dianggap final. Invoice mencantumkan "Harga sudah termasuk pajak sesuai ketentuan yang berlaku". |
| B13 | HPP | Biaya satuan disimpan dengan `NUMERIC(14,4)`. Metode: rata-rata tertimbang bergerak per produk per cabang. |
| B14 | Peran | Empat peran: `OWNER`, `DOCTOR`, `ADMIN`, `CUSTOMER`. Setiap pengguna memiliki tepat satu peran. Izin tambahan diberikan melalui `user_permission_grants`. |
| B15 | Dokter ganda | Pemilik yang memiliki profil dokter aktif dengan nomor STR memiliki kewenangan klinis. Pemilik tanpa profil dokter hanya dapat membaca rekam medis. |
| B16 | Rekam medis | Satu rekam medis per kunjungan. Rekam medis `SIGNED` tidak dapat diubah. Koreksi hanya melalui amendemen oleh dokter penanda asli. |
| B17 | Obat resep | Produk dengan `is_prescription = true` hanya dapat dikeluarkan melalui resep bertanda tangan. Produk tersebut tidak dapat masuk POS atau toko online. |
| B18 | Diskon POS | Diskon efektif sampai 10% dari harga kotor baris dapat diberikan oleh peran dengan `pos.discount.apply`. Diskon di atas 10% memerlukan verifikasi kata sandi pemilik yang tercatat. Diskon invoice layanan hanya oleh pemilik. |
| B19 | Persetujuan pengeluaran | Pengeluaran di atas Rp 1.000.000 wajib disetujui pengguna lain dengan `finance.expense.approve`. |
| B20 | Persetujuan refund | Setiap refund wajib disetujui pengguna lain dengan `refund.approve`. Default hanya pemilik. Pemohon tidak boleh menyetujui refund sendiri. Operasi live mensyaratkan minimal dua akun pemilik aktif (lihat B25). |
| B21 | Penyesuaian stok | Penyesuaian tidak memerlukan persetujuan tambahan, tetapi wajib memakai kode alasan. Laporan penyesuaian bulanan dikirim ke pemilik. |
| B22 | Kedaluwarsa | Batch yang sudah lewat tanggal kedaluwarsa tidak dapat dijual, dipakai, atau ditransfer. Batch yang kedaluwarsa dalam 90 hari ditandai di dashboard. |
| B23 | MFA | Wajib untuk pengguna yang memiliki salah satu izin: `payment.confirm`, `refund.approve`, `refund.complete`, `settings.manage`, `finance.*`, `medical_record.sign`, `medical_record.amend`, `user.manage`. |
| B24 | Sesi | Staf berakhir setelah 8 jam tanpa aktivitas. Pelanggan berakhir setelah 30 hari tanpa aktivitas. |
| B25 | Pemilik pertama | Dibuat melalui perintah artisan `app:create-owner` yang hanya dapat dijalankan sekali per lingkungan dan dicatat di audit. Sebelum operasi live, minimal dua akun pemilik aktif harus ada. |
| B26 | Undangan staf | Berlaku 72 jam dan hanya dapat dipakai sekali. |
| B27 | Pemulihan | RPO 15 menit. RTO 4 jam. |
| B28 | Retensi | Rekam medis dan transaksi disimpan minimal 10 tahun sejak kunjungan atau transaksi terakhir. Data pribadi pelanggan dianonimkan, tidak dihapus baris. Pelanggan tidak dihapus fisik. |
| B29 | Layanan tanpa stok | Konsultasi, grooming, dan penitipan tidak memiliki inventori. Layanan dikelola melalui `service_catalog`. |
| B30 | Fulfillment online | Setiap pesanan online ditetapkan ke satu cabang fulfillment. Stok diperiksa dan dialokasikan di cabang tersebut. |
| B31 | Kelebihan pembayaran layanan | Kelebihan pembayaran menjadi `customer_credits`, bukan tip atau saldo otomatis. Kredit dipakai untuk invoice berikutnya dari pelanggan yang sama atau dikembalikan melalui refund. |
| B32 | Notifikasi | Saluran: dalam aplikasi dan email. Tidak ada WhatsApp. Email dan notifikasi tidak memuat data medis. |
| B33 | Invoice per sumber | Setiap kunjungan, rawat inap, dan booking hanya memiliki satu invoice layanan aktif (belum `VOIDED`). |
| B34 | Kapasitas dokter | Janji temu dokter yang sama tidak boleh tumpang tindih untuk status aktif. |

---

## 2. Arsitektur Aplikasi

### 2.1 Komponen

| Komponen | Teknologi | Fungsi |
|----------|-----------|--------|
| Panel staf | Filament 4 di `/admin` | Dashboard, POS, inventori, klinik, rawat inap, grooming, keuangan, laporan, pengaturan |
| Situs publik dan portal | Blade + Tailwind CSS 4 di `/` dan `/akun` | Katalog, keranjang, pesanan, janji temu, hewan, invoice |
| Komponen interaktif | Livewire 3 | Keranjang, form booking, POS, unggah bukti |
| Logika bisnis | Kelas PHP di `app/Services` | Satu-satunya tempat yang mengubah stok, kas, status pembayaran, dan rekam medis |
| Otorisasi | `AccessControl` + Policies | Pemeriksaan izin, cabang, kepemilikan, dan status |
| Antrean | Tabel `jobs` (driver database) | Email, notifikasi, pembuatan PDF berat |
| Penjadwalan | Scheduler Laravel | Job harian dan per jam |
| Basis data | PostgreSQL 16 | Data transaksi dan audit |
| Berkas | Disk lokal `private` dan `public` | Bukti pembayaran, dokumen medis, foto hewan, foto produk |
| Email | Resend | Verifikasi, reset, notifikasi |
| Error | Sentry | Pemantauan error |

### 2.2 Peta rute

| Path | Aplikasi | Akses |
|------|----------|-------|
| `/` | Blade | Publik |
| `/katalog`, `/katalog/{sku}` | Blade | Publik (produk aktif online) |
| `/masuk`, `/daftar`, `/lupa-kata-sandi` | Fortify | Publik |
| `/akun/*` | Blade + Livewire | Pelanggan terautentikasi dengan email terverifikasi |
| `/admin/*` | Filament | Staf aktif (OWNER, DOCTOR, ADMIN) |
| `/undangan/{token}` | Blade | Publik dengan token belum terpakai |
| `/health` | Blade/JSON | Publik, hanya status DB dan antrean |

Pelanggan tidak dapat mengakses `/admin`. Staf tidak menggunakan portal `/akun` untuk operasi toko.

### 2.3 Pembagian tanggung jawab kode

- **Model Eloquent** hanya berisi relasi, casting, dan scope baca. Model tidak mengubah stok, kas, atau status melalui `save()` langsung dari controller atau komponen.
- **Services** (`app/Services/*`) berisi seluruh operasi tulis bisnis dalam `DB::transaction`. Setiap method service memanggil `AccessControl` di awal.
- **Filament Resources, Pages, dan Livewire components** tipis. Mereka memvalidasi bentuk input dan memanggil service.
- **Form Request dan validasi Livewire** memeriksa bentuk input. Aturan bisnis tetap di service.
- **Policies** memeriksa kepemilikan dan cakupan cabang untuk tampilan dan tindakan.
- Komponen Blade atau Livewire tidak menghitung total transaksi. Total selalu dihitung ulang oleh service.

---

## 3. Lingkungan dan Deployment

### 3.1 Lingkungan

| Lingkungan | Sumber | Basis data | Domain |
|------------|--------|-----------|--------|
| Lokal | Clone repository | PostgreSQL lokal (Docker atau instalasi native) | `petshop.test` |
| Staging | Branch `main` (deploy otomatis) | Database staging terpisah, data sintetis | `staging.{domain}` |
| Production | Branch `production` (deploy manual) | Database production | `app.{domain}` |

Data production tidak boleh disalin ke staging atau lokal. Data staging dibuat melalui seeder sintetis.

### 3.2 Alur deployment

1. Developer membuat branch `feat/...`, `fix/...`, atau `db/...` dari `main`.
2. Pull request dibuka ke `main`. GitHub Actions menjalankan tes, format, analisis statis, dan build.
3. Setelah review satu orang lain dan tes lulus, pull request di-merge ke `main`.
4. Forge men-deploy `main` ke staging secara otomatis. Migration berjalan otomatis di staging.
5. Smoke test di staging dilakukan oleh developer.
6. Untuk rilis, `main` di-fast-forward ke branch `production`. Deployment production dijalankan dengan tombol **Deploy** di Forge oleh pemilik atau developer lead setelah UAT disetujui.

### 3.3 Penyiapan Forge (satu kali)

1. Buat server di Forge: Ubuntu 24.04, PHP 8.4, Nginx, Node.js 22, dan Composer. Pilih region Singapura.
2. Buat PostgreSQL 16 terkelola di region yang sama. Aktifkan PITR dan catat host, port, database, user, dan password. Akses dibatasi ke IP server Forge.
3. Buat dua site Forge (staging dan production) dengan repository GitHub privat. Aktifkan SSL Let's Encrypt.
4. Isi environment variables di Forge (lihat 3.4).
5. Atur **Quick Deploy** hanya untuk site staging (branch `main`).
6. Aktifkan **Scheduler** untuk setiap site. Forge membuat cron `schedule:run` otomatis.
7. Buat **Queue Worker** (daemon) untuk koneksi `database`: `php artisan queue:work --sleep=3 --tries=3 --max-time=3600`.
8. Jalankan sekali `php artisan storage:link` dan `php artisan app:create-owner` di setiap lingkungan.

### 3.4 Environment variables

```dotenv
APP_NAME="Pet Shop dan Klinik"
APP_ENV=production
APP_DEBUG=false
APP_URL=https://app.{domain}
APP_KEY=
APP_TIMEZONE=UTC
APP_LOCALE=id
APP_FALLBACK_LOCALE=id

LOG_CHANNEL=stack
LOG_LEVEL=info

DB_CONNECTION=pgsql
DB_HOST=
DB_PORT=5432
DB_DATABASE=
DB_USERNAME=
DB_PASSWORD=
DB_SSLMODE=require

SESSION_DRIVER=database
SESSION_LIFETIME=43200
SESSION_SECURE_COOKIE=true
SESSION_SAME_SITE=lax
CACHE_STORE=database
QUEUE_CONNECTION=database
FILESYSTEM_DISK=private

MAIL_MAILER=resend
RESEND_KEY=
MAIL_FROM_ADDRESS=noreply@{domain}
MAIL_FROM_NAME="Pet Shop dan Klinik"

SENTRY_LARAVEL_DSN=
BACKUP_S3_KEY=
BACKUP_S3_SECRET=
BACKUP_S3_BUCKET=
BACKUP_S3_ENDPOINT=
BACKUP_S3_REGION=auto
```

Aturan:
- `APP_KEY`, password database, `RESEND_KEY`, dan kredensial backup hanya ada di environment Forge. Tidak ada di repository.
- Variabel `APP_DEBUG=true` tidak pernah digunakan di staging atau production.
- Kunci yang pernah terekspos wajib dirotasi dalam 24 jam.

### 3.5 Skrip deploy Forge

```bash
cd $FORGE_SITE_PATH
git pull origin $FORGE_SITE_BRANCH
composer install --no-dev --optimize-autoloader --no-interaction
npm ci
npm run build
php artisan migrate --force
php artisan filament:assets
php artisan optimize
php artisan filament:optimize
php artisan queue:restart
```

Aturan migration:
- Migration destruktif (drop kolom, tabel, constraint, atau pengubahan tipe data) tidak boleh masuk `main` tanpa persetujuan pemilik yang tercatat di pull request.
- Perubahan berisiko memakai pola expand and contract: tambah struktur baru, deploy kode yang kompatibel, backfill, verifikasi, lalu hapus struktur lama pada rilis terpisah.
- Migration tidak dijalankan manual di production kecuali melalui prosedur insiden (lihat 23.3).

### 3.6 Pemeriksaan kesehatan

- `GET /health` memeriksa koneksi basis data, antrean yang tertunda (lebih dari 500 job), dan keberadaan disk privat. Respons berupa JSON `{"status":"ok"}` atau `503`.
- Forge atau layanan uptime memanggil `/health` setiap 5 menit.

---

## 4. Cakupan Modul

| Kode | Modul | Isi |
|------|-------|-----|
| MOD-01 | Identitas dan akses | Login, MFA, undangan, peran, izin, sesi, audit |
| MOD-02 | Cabang | Profil, zona waktu, lokasi stok, kasir, metode dan rekening pembayaran |
| MOD-03 | Pelanggan dan hewan | Pelanggan, wali, hewan, dokumen hewan |
| MOD-04 | Produk | SKU, barcode, kategori, satuan, harga per cabang, status jual |
| MOD-05 | Inventori | Lokasi, batch, saldo, mutasi, penyesuaian, stok opname, transfer, HPP |
| MOD-06 | Pembelian | Supplier, pesanan pembelian, penerimaan, retur supplier, tagihan supplier |
| MOD-07 | POS | Shift kasir, penjualan, pembayaran terpisah, retur, void draft |
| MOD-08 | Toko online | Katalog, keranjang, pesanan, pembayaran manual, review stok, fulfillment |
| MOD-09 | Klinik | Jadwal dokter, janji temu, kunjungan, rekam medis, resep, pemakaian obat |
| MOD-10 | Rawat inap | Unit rawat, penempatan, observasi, tugas, update pelanggan, pulang |
| MOD-11 | Grooming dan penitipan | Katalog layanan, kapasitas, booking, check-in/out, layanan tambahan |
| MOD-12 | Tagihan layanan | Invoice layanan, pembayaran parsial, kredit pelanggan, piutang |
| MOD-13 | Keuangan | Buku kas, pengeluaran, refund, periode keuangan, HPP |
| MOD-14 | Portal pelanggan | Hewan, janji temu, booking, pesanan, invoice, rekam medis ringkas, update rawat inap |
| MOD-15 | Notifikasi | Dalam aplikasi dan email |
| MOD-16 | Laporan | Penjualan, online, stok, kedaluwarsa, klinik per dokter, layanan, piutang, kas, laba rugi sederhana |
| MOD-17 | Pengaturan dan operasional | Metode pembayaran, kode alasan, template invoice, pengaturan cabang, job, audit |

**Di luar cakupan Versi 1:** payment gateway, integrasi akuntansi penuh, perhitungan PPN, marketplace, aplikasi native, mode offline, payroll, AI diagnosis, integrasi alat laboratorium, WhatsApp, multi-mata uang, REST API publik, dan pemindaian virus berkas.

---

## 5. Identitas, Peran, dan Akses

### 5.1 Izin dan pemetaan peran

Izin adalah kode tetap yang disimpan di tabel `permissions` dan diisi oleh seeder dari enum PHP. Pemetaan peran ke izin disimpan di `role_permissions` dan hanya diubah lewat migration.

Legenda: ✔ = diberikan default; (C) = memerlukan profil dokter aktif; (M) = hanya untuk janji/tugas/kunjungan yang ditugaskan kepadanya; (B) = dibatasi cabang yang ditugaskan; (G) = hanya lewat `user_permission_grants`.

| Kelompok | Izin | OWNER | DOCTOR | ADMIN | CUSTOMER |
|----------|------|:-----:|:------:|:-----:|:--------:|
| Pengaturan | `branch.manage`, `user.manage`, `settings.manage` | ✔ | | | |
| Produk | `product.manage` | ✔ | | ✔ | |
| Harga | `price.manage` | ✔ | | | |
| Stok | `inventory.read` | ✔ | ✔ (B, klinik) | ✔ (B) | |
| | `inventory.read_cost` | ✔ | | | |
| | `inventory.adjust` | ✔ | | (G) | |
| | `inventory.count` | ✔ | | ✔ (B) | |
| | `inventory.transfer.create` | ✔ | | ✔ (B) | |
| | `inventory.transfer.approve` | ✔ | | | |
| | `inventory.transfer.receive` | ✔ | | ✔ (B) | |
| Pembelian | `purchase.manage` | ✔ | | ✔ (B) | |
| POS | `pos.sell` | ✔ | | ✔ (B) | |
| | `pos.discount.apply` | ✔ | | ✔ (B) | |
| | `pos.discount.override` | ✔ | | | |
| | `pos.return.request` | ✔ | | ✔ (B) | |
| | `cash_shift.manage` | ✔ | | ✔ (B) | |
| Pembayaran online | `payment.review` | ✔ | | ✔ (B) | |
| | `payment.confirm` | ✔ | | ✔ (B) | |
| | `stock_review.resolve` | ✔ | | ✔ (B) | |
| | `fulfillment.manage` | ✔ | | ✔ (B) | |
| Refund | `refund.request` | ✔ | | ✔ (B) | |
| | `refund.approve` | ✔ | | | |
| | `refund.complete` | ✔ | | | |
| Pelanggan | `customer.manage`, `pet.manage` | ✔ | | ✔ (B) | |
| Jadwal | `schedule.manage` | ✔ | | ✔ (B) | |
| | `appointment.manage` | ✔ | ✔ (M) | ✔ (B) | |
| Klinik | `visit.manage` | ✔ (C) | ✔ (M) | | |
| | `medical_record.read` | ✔ | ✔ (B) | | ✔ (miliknya, `SIGNED`) |
| | `medical_record.write` | | ✔ (C, M) | | |
| | `medical_record.sign` | ✔ (C, M) | ✔ (C, M) | | |
| | `medical_record.amend` | ✔ (C, penanda asli) | ✔ (C, penanda asli) | | |
| | `prescription.write` | ✔ (C) | ✔ (C, M) | | |
| | `medication.dispense` | ✔ (C) | ✔ (C, B) | | |
| Rawat inap | `hospitalization.manage` | ✔ | ✔ (B) | ✔ (B) | |
| | `hospitalization.observe` | ✔ (C) | ✔ (C, B) | | |
| | `hospitalization.publish_update` | ✔ (C) | ✔ (C, M) | | |
| Grooming | `grooming.manage` | ✔ | | ✔ (B) | |
| Tagihan layanan | `service_invoice.manage` | ✔ | | ✔ (B) | |
| | `service_invoice.discount` | ✔ | | | |
| | `service_invoice.void` | ✔ | | | |
| Keuangan | `finance.cash.read` | ✔ | | | |
| | `finance.expense.create` | ✔ | | ✔ (B) | |
| | `finance.expense.approve` | ✔ | | | |
| | `finance.report.read` | ✔ | | | |
| | `finance.period.close` | ✔ | | | |
| Laporan | `report.sales.read` | ✔ | | ✔ (B, tanpa HPP) | |
| Audit | `audit.read` | ✔ | | (G, B, tanpa data klinis) | |

Catatan:
- Istilah "(C)" berarti pengguna harus memiliki `doctor_profiles` dengan `is_active = true`. Ini berlaku untuk pemilik maupun dokter.
- Pemilik tanpa profil dokter hanya membaca rekam medis. Pemilik tidak dapat menandatangani atau menulis klinis.
- Admin tidak memiliki akses data klinis, observasi, atau laporan HPP.
- Izin `(G)` diberikan oleh pemilik melalui `user_permission_grants` dengan alasan dan tanggal kedaluwarsa opsional. Grant tidak dapat memberikan izin `(C)` kepada pengguna tanpa profil dokter.

### 5.2 Evaluasi izin

Urutan `AccessControl::can($user, $permission, $branchId = null, $object = null)`:

1. Pengguna aktif (`users.is_active = true`).
2. Izin ada dalam `role_permissions` untuk peran pengguna, atau ada grant aktif di `user_permission_grants`.
3. Jika izin bertanda (C): profil dokter aktif tersedia.
4. Jika izin bertanda (M): objek (janji, kunjungan, tugas) ditugaskan ke pengguna.
5. Jika `$branchId` diberikan: pengguna OWNER, atau ada baris `user_branch_access` untuk cabang tersebut.
6. Jika `$object` diberikan: objek berada dalam cakupan cabang dan kepemilikan.
7. Untuk izin di daftar MFA (B23): MFA aktif dan sesi sudah diverifikasi MFA dalam 12 jam terakhir.

Kegagalan pada langkah 1–6 untuk objek yang seharusnya tidak terlihat menghasilkan `NOT_FOUND` (404). Kegagalan tindakan pada objek yang terlihat menghasilkan `FORBIDDEN` (403). Kegagalan MFA menghasilkan halaman "Verifikasi MFA diperlukan".

### 5.3 Cakupan cabang

- `OWNER` memiliki cakupan semua cabang.
- `DOCTOR` dan `ADMIN` memiliki cakupan cabang sesuai `user_branch_access`. Setidaknya satu cabang wajib ada.
- Pelanggan tidak memiliki cakupan cabang. Pelanggan hanya memiliki cakupan atas data miliknya, yaitu hewan tempat ia berstatus wali, dan pesanan miliknya.
- Filament menerapkan cakupan melalui `Resource::getEloquentQuery()` yang memanggil scope `forBranches(...)`. Scope ini tidak boleh dilewati di resource apa pun.

### 5.4 Pemisahan tugas

- Pemohon refund tidak boleh menyetujui atau menyelesaikan refund yang sama. Validasi dilakukan di service dan dijaga oleh constraint `CHECK (approved_by <> requested_by)`.
- Pengaju pengeluaran di atas batas tidak boleh menyetujui pengeluarannya sendiri.
- Pengirim transfer antarcabang tidak boleh menerima transfer yang sama. Penerimaan dilakukan oleh pengguna cabang tujuan atau pemilik.
- Amendemen rekam medis hanya boleh dilakukan oleh dokter penanda asli.
- Konfirmasi pembayaran oleh staf yang sama dengan pemeriksa bukti diizinkan (keduanya memiliki `payment.review` dan `payment.confirm`), tetapi setiap langkah dicatat dengan identitas masing-masing.

### 5.5 Akun, sesi, dan MFA

- Login memakai email dan kata sandi minimal 10 karakter, dengan pemeriksaan kata sandi bocor (`Password::uncompromised()`).
- Lockout: lima kegagalan login dalam 15 menit untuk satu email menghasilkan penguncian 15 menit. Penghitung memakai `RateLimiter` dengan kunci email.
- Reset kata sandi dikirim ke email terdaftar dengan tautan berlaku 60 menit.
- Staf wajib mengaktifkan MFA TOTP saat login pertama jika izinnya termasuk B23. MFA dikelola lewat halaman profil Filament Breezy.
- Pengguna nonaktif tidak dapat login. Saat dinonaktifkan, sesinya dihapus dari tabel `sessions`.
- Sesi staf berakhir setelah 8 jam tanpa aktivitas. Middleware `EnforceSessionIdle` membandingkan `last_activity` dan peran pengguna. Sesi pelanggan memakai batas 30 hari dari `SESSION_LIFETIME`.
- Pemilik dapat melihat dan mencabut sesi aktif pengguna lain dari halaman Pengguna.

### 5.6 Undangan staf

1. Pemilik membuat undangan dengan email, peran (`DOCTOR`, `ADMIN`), dan cabang. Peran `DOCTOR` memerlukan data STR.
2. Sistem membuat token acak 64 karakter (disimpan sebagai hash SHA-256), kedaluwarsa 72 jam, dan mengirim tautan `/undangan/{token}` ke email.
3. Penerima menentukan nama dan kata sandi. Sistem membuat akun dengan `email_verified_at` terisi.
4. Token ditandai `used_at` dalam transaksi yang sama. Token yang sudah dipakai atau kedaluwarsa menghasilkan halaman "Undangan tidak berlaku".

### 5.7 Pelanggan

- Pendaftaran publik membuat akun dengan peran `CUSTOMER` dan wajib verifikasi email.
- Pelanggan walk-in dibuat staf tanpa akun. Jika email pelanggan walk-in sudah dipakai akun lain, staf tidak dapat menautkannya secara langsung. Penautan dilakukan dengan mengirim undangan tautan ke email pelanggan yang sudah terverifikasi.

---

## 6. Pelanggan, Hewan, Produk, dan Supplier

### 6.1 Pelanggan

Data minimal: nama lengkap, nomor telepon, email (opsional untuk walk-in), alamat. `customers.user_id` nullable dan unik jika tidak null.

Pelanggan tidak dihapus. Nonaktifkan melalui `is_active = false`. Anonimisasi (B28) memakai command `customers:anonymize {id}` yang hanya dapat dijalankan pemilik.

### 6.2 Hewan

Data: nama, spesies, ras, jenis kelamin, status steril, tanggal lahir atau estimasi usia (`birth_date_is_estimate`), berat terakhir, warna, nomor microchip (opsional, unik jika terisi), status meninggal (`is_deceased`, `deceased_at`).

- Spesies: `DOG`, `CAT`, `BIRD`, `RABBIT`, `REPTILE`, `OTHER`.
- Jenis kelamin: `MALE`, `FEMALE`, `UNKNOWN`.
- Hewan dengan `is_deceased = true` tidak dapat menerima janji, kunjungan, booking, atau rawat inap baru. Riwayatnya tetap dapat dibaca.

### 6.3 Wali

- Tabel `pet_guardians` menghubungkan hewan dan pelanggan dengan `is_primary`.
- Indeks unik parsial menegakkan tepat satu `is_primary = true` untuk setiap hewan aktif.
- Mengganti wali utama dilakukan oleh admin atau pemilik dalam transaksi: wali lama menjadi sekunder (atau relasinya diakhiri dengan `ended_at`), wali baru menjadi utama. Perubahan dicatat di audit.
- Hanya wali utama yang menerima tagihan dan menyetujui layanan tambahan.

### 6.4 Produk

- `products`: SKU unik global, nama, kategori, `unit_type` (`COUNT` atau `WEIGHT`), `base_unit` (misalnya `PCS`, `KG`, `PACK`), `track_lot`, `is_prescription`, `is_active`.
- Kategori: `RETAIL`, `MEDICINE`, `CLINIC_SUPPLY`, `FOOD`, `ACCESSORY`, `OTHER`.
- Produk kategori `MEDICINE` secara otomatis memiliki `track_lot = true`. Validasi dilakukan di model dan constraint `CHECK`.
- `product_barcodes` memungkinkan banyak barcode per produk. Barcode unik global.
- `product_branch_settings` (unik `(product_id, branch_id)`): `is_pos_sellable`, `is_online_sellable`, `min_stock`, `is_active`.
- `product_prices` bersifat append-only dengan `effective_from`. Harga aktif di cabang adalah baris terbaru dengan `effective_from <= now()`.
- Produk tanpa harga aktif di cabang tidak dapat masuk draft POS, keranjang, atau katalog cabang tersebut.
- Perubahan harga tidak mengubah transaksi lama.

### 6.5 Supplier

Nama, nama kontak, telepon, email, alamat, NPWP (opsional), syarat pembayaran dalam hari. Supplier dinonaktifkan, tidak dihapus.

---

## 7. Inventori

### 7.1 Lokasi dan batch

- Setiap cabang memiliki tepat satu lokasi `RETAIL` dan tepat satu lokasi `CLINIC_PHARMACY`. Lokasi tambahan bertipe `WAREHOUSE` boleh ada. Constraint parsial menegakkan dua lokasi wajib per cabang.
- Setiap pasangan (produk, lokasi) yang tidak memakai batch memiliki satu baris `inventory_lots` dengan `lot_number = 'DEFAULT'` dan `expiry_date = NULL`. Dengan begitu kunci unik tidak pernah memuat NULL.
- Produk `track_lot = true` membuat batch baru atau menambah batch yang sudah ada dengan `lot_number` dan `expiry_date` wajib pada penerimaan.
- Batch dengan `expiry_date < hari ini` (zona cabang) tidak dapat dipakai untuk penjualan, pengeluaran klinik, atau transfer.

### 7.2 Saldo

`stock_balances` memiliki kunci unik `(product_id, location_id, lot_id)` dan constraint:
- `on_hand >= 0`
- `reserved >= 0`
- `reserved <= on_hand`

```
available = on_hand − reserved
```

`on_hand` hanya berubah melalui `inventory_movements`. `reserved` hanya berubah melalui `stock_reservations`. Keduanya diubah dalam transaksi yang sama.

### 7.3 Ledger

`inventory_movements` bersifat append-only dan memuat setiap perubahan `on_hand`. Tipe:

| Tipe | Arah | Sumber |
|------|------|--------|
| `PURCHASE_RECEIPT` | + | Penerimaan barang |
| `SALE_POS` | − | Penyelesaian penjualan POS |
| `ORDER_FULFILLMENT` | − | Pesanan online diserahkan atau dikirim |
| `CLINIC_USE` | − | Pengeluaran obat atau bahan klinik |
| `CUSTOMER_RETURN_RESTOCK` | + | Retur POS yang dinyatakan layak jual |
| `SUPPLIER_RETURN` | − | Retur ke supplier |
| `ADJUSTMENT` | ± | Penyesuaian dan stok opname |
| `TRANSFER_OUT` | − | Pengiriman transfer dari lokasi asal |
| `TRANSFER_IN` | + | Penerimaan transfer di lokasi tujuan |

Setiap baris memuat `quantity_delta` (tidak boleh 0), `unit_cost_snapshot`, `reference_type`, `reference_id`, `reason_code` (untuk penyesuaian), `actor_id`, dan `occurred_at`.

Reservasi tidak menghasilkan baris ledger karena reservasi tidak mengubah `on_hand`. Reservasi dicatat di `stock_reservations`.

### 7.4 Reservasi

- `stock_reservations` unik per `(order_item_id, lot_id)`. Satu item dapat memiliki beberapa reservasi jika pengambilan FEFO melintasi beberapa batch.
- Status reservasi: `ACTIVE`, `RELEASED`, `FULFILLED`.
- Reservasi dibuat hanya oleh `OnlinePaymentService::confirm` ketika stok tersedia.
- Reservasi dilepas (`RELEASED`) saat pesanan dibatalkan sebelum diserahkan. Setiap pelepasan mengurangi `reserved` dalam transaksi yang sama.
- Saat pesanan diserahkan atau dikirim, reservasi berubah menjadi `FULFILLED`. `on_hand` dan `reserved` berkurang bersamaan dan ledger mencatat `ORDER_FULFILLMENT`.

### 7.5 Penyesuaian

- Kode alasan tetap: `COUNT_CORRECTION`, `DAMAGED`, `EXPIRY_WRITE_OFF`, `LOST`, `FOUND`, `CLINIC_USE_CORRECTION`, `OTHER`.
- Alasan `OTHER` wajib catatan.
- Penyesuaian positif memakai biaya rata-rata saat ini. Jika biaya rata-rata belum ada, `unit_cost` wajib diisi.
- Penyesuaian negatif tidak boleh membuat `on_hand` negatif. Pengurangan memakai FEFO.
- Penyesuaian dibuat melalui `InventoryAdjustmentService::apply`.

### 7.6 Stok opname

- Sesi opname (`count_sessions`) dibuat per lokasi dengan status `OPEN` atau `CLOSED`.
- Staf mengisi jumlah fisik per produk dan batch (`count_session_items`).
- Saat sesi ditutup, sistem membuat penyesuaian `COUNT_CORRECTION` untuk setiap selisih, dalam satu transaksi.
- Sesi `CLOSED` tidak dapat diubah. Satu lokasi hanya boleh memiliki satu sesi `OPEN`.

### 7.7 Transfer

Transfer dapat berpindah antarlokasi dalam satu cabang atau antarcabang.

Status: `DRAFT`, `IN_TRANSIT`, `RECEIVED`, `CANCELLED`.

1. Pengirim dengan `inventory.transfer.create` membuat transfer `DRAFT` berisi lokasi asal, lokasi tujuan, dan item.
2. Transfer antarcabang memerlukan persetujuan `inventory.transfer.approve`. Transfer dalam cabang tidak memerlukan persetujuan.
3. Saat dikirim (`dispatch`), sistem memeriksa ketersediaan dan batch (tidak kedaluwarsa), membuat `TRANSFER_OUT`, mengurangi `on_hand` lokasi asal, dan mengubah status menjadi `IN_TRANSIT`.
4. Penerimaan (`receive`) dilakukan di lokasi tujuan. Penerima mencatat jumlah diterima per item. Sistem membuat `TRANSFER_IN` sebesar jumlah diterima dan menambah `on_hand` lokasi tujuan.
5. Selisih (jumlah dikirim dikurangi jumlah diterima) disimpan sebagai `quantity_lost` di `inventory_transfer_items` dengan alasan wajib. Selisih tidak membuat baris ledger tambahan. Selisih dilaporkan di laporan stok sebagai kehilangan transfer.
6. Transfer `DRAFT` dapat dibatalkan. Transfer `IN_TRANSIT` tidak dapat dibatalkan, hanya diterima.

Barang dalam perjalanan tidak disimpan sebagai saldo. Jumlahnya dihitung dari `inventory_transfer_items` pada transfer berstatus `IN_TRANSIT`.

Transfer memakai biaya rata-rata cabang asal. Di cabang tujuan, biaya dihitung ulang sebagai rata-rata tertimbang dengan biaya tersebut.

### 7.8 Biaya rata-rata tertimbang (HPP)

- Biaya disimpan di `product_branch_costs` (unik `(product_id, branch_id)`) dengan `avg_unit_cost NUMERIC(14,4)`.
- Setiap penerimaan dan retur layak jual memperbarui biaya:

```
qty_lama = total on_hand produk di cabang sebelum perubahan
biaya_baru = (qty_lama × biaya_lama + qty_masuk × biaya_masuk) / (qty_lama + qty_masuk)
```

- Jika `qty_lama <= 0`, `biaya_baru = biaya_masuk`.
- Hasil dibulatkan ke 4 desimal dengan `HALF_UP`.
- Penjualan, pengeluaran klinik, dan pengeluaran rawat inap menyimpan `unit_cost_snapshot` pada baris transaksi. Snapshot tidak pernah berubah.
- Setiap perubahan biaya rata-rata dicatat di `cost_history` (`product_id`, `branch_id`, `old_cost`, `new_cost`, `reference_type`, `reference_id`).
- Retur layak jual memakai `unit_cost_snapshot` asal sebagai `biaya_masuk`.
- Retur ke supplier mengurangi `on_hand` dengan biaya rata-rata saat retur. Biaya rata-rata tidak berubah.

### 7.9 Pengambilan FEFO

Untuk produk `track_lot = true`, pengambilan dilakukan dengan urutan:
1. `expiry_date` terdekat (NULL paling akhir).
2. `received_at` paling awal.
3. `id` lot (tie-breaker).

Batch yang kedaluwarsa dilewati. Pengambilan boleh lintas batch untuk satu item. Untuk produk `track_lot = false`, pengambilan dari baris `DEFAULT`.

### 7.10 Rekonsiliasi

Job `stock_reconciliation` harian membandingkan `on_hand` setiap `(product, location, lot)` dengan jumlah `quantity_delta` di ledger.

- Selisih nol: tidak ada tindakan.
- Selisih tidak nol: membuat notifikasi ke pemilik dan entri di `job_runs` dengan detail. Sistem tidak memperbaiki saldo otomatis.

---

## 8. Pembelian dan Penerimaan

### 8.1 Pesanan pembelian

- Status: `DRAFT`, `SUBMITTED`, `PARTIALLY_RECEIVED`, `RECEIVED`, `CANCELLED`.
- `DRAFT` dapat diubah. Status lainnya tidak dapat diubah. Perubahan dilakukan dengan membatalkan pesanan dan membuat yang baru.
- Pesanan `CANCELLED` hanya dapat dibatalkan jika belum ada penerimaan.
- Nomor pesanan memakai jenis `PO`.

### 8.2 Penerimaan barang

- Penerimaan dicatat per item: jumlah diterima, lokasi tujuan, batch dan tanggal kedaluwarsa (wajib untuk `track_lot`), dan harga perolehan per unit.
- Jumlah penerimaan tidak boleh melebihi jumlah pesanan kecuali pemilik memasukkan alasan dan memberi izin eksplisit pada penerimaan.
- `PurchaseReceivingService::receive` dalam satu transaksi: membuat `goods_receipts`, menambah `on_hand`, membuat `PURCHASE_RECEIPT`, memperbarui biaya rata-rata, dan memperbarui status pesanan.
- Penerimaan tidak dapat dibatalkan. Kesalahan diperbaiki melalui penyesuaian atau retur supplier.

### 8.3 Retur supplier

- Retur mencantumkan referensi penerimaan asal, jumlah, dan alasan.
- Retur membuat `SUPPLIER_RETURN` yang mengurangi `on_hand` dan memakai biaya rata-rata saat retur.
- Jumlah retur tidak boleh melebihi stok batch yang tersedia.

### 8.4 Tagihan supplier

- `supplier_bills` dibuat dari penerimaan atau input manual dengan nomor faktur supplier, tanggal faktur, tanggal jatuh tempo, dan total.
- `supplier_payments` mencatat pembayaran terhadap tagihan. Setiap pembayaran membuat `cash_movements` bertipe `CASH_OUT_SUPPLIER`.
- Tagihan dan penerimaan barang dicatat terpisah. Penerimaan tidak otomatis berarti tagihan lunas.

---

## 9. Point of Sale

### 9.1 Shift kasir

- Shift dibuka dengan saldo kas awal (`opening_cash`).
- Satu kasir hanya boleh memiliki satu shift `OPEN`. Constraint unik parsial menegakkan ini.
- Setiap penjualan, pembayaran, dan retur tunai terkait shift aktif.
- Penutupan: kasir atau admin menghitung kas fisik (`closing_cash_counted`).
- Saldo yang diharapkan:

```
expected = opening_cash + Σ CASH_IN_* − Σ CASH_OUT_* (metode tunai, shift ini)
difference = closing_cash_counted − expected
```

- Selisih wajib memiliki `difference_reason` jika tidak nol.
- Shift `CLOSED` tidak dapat diubah.
- Penutupan menghasilkan rekap per metode pembayaran.

### 9.2 Draft penjualan

Status penjualan: `DRAFT`, `COMPLETED`, `PARTIALLY_RETURNED`, `RETURNED`, `VOIDED`.

- Draft tidak mereservasi stok dan tidak mengubah `on_hand`.
- Draft dapat diubah oleh kasir pembuatnya selama `DRAFT`.
- Draft yang tidak diselesaikan dalam 24 jam berubah menjadi `VOIDED` oleh job `void_stale_drafts` (setiap jam).
- Draft dapat dihapus (void) oleh kasir pembuatnya sebelum selesai.
- Setiap perubahan draft menghitung ulang harga, diskon, dan total di server.
- Produk dengan `is_prescription = true` tidak dapat ditambahkan ke draft.

### 9.3 Perhitungan

Semua nilai dalam rupiah penuh. Qty `COUNT` bilangan bulat. Qty `WEIGHT` tiga desimal.

```
unit_price      = harga aktif cabang (bilangan bulat)
line_gross      = ROUND_HALF_UP(qty × unit_price)
line_discount   = PERCENT: ROUND_HALF_UP(line_gross × pct / 100)
                  AMOUNT : nilai yang diinput
line_total      = line_gross − line_discount
effective_pct   = line_discount / line_gross × 100 (untuk pemeriksaan batas)
subtotal        = Σ line_gross
discount_total  = Σ line_discount
grand_total     = Σ line_total
```

Aturan:
- `effective_pct <= 10` dapat diberikan oleh pengguna dengan `pos.discount.apply`.
- `effective_pct > 10` memerlukan `pos.discount.override`. Pemilik memasukkan kata sandi sendiri pada dialog konfirmasi. Sistem memverifikasi kata sandi terhadap akun pemilik yang sedang memegang otorisasi dan mencatat `approved_by`, waktu, dan alasan.
- `line_discount` tidak boleh melebihi `line_gross`. `grand_total` tidak boleh negatif.
- Pembulatan dilakukan per baris. Tidak ada pembulatan ulang pada total.

### 9.4 Pembayaran

- Metode dipilih dari metode aktif di cabang: `CASH`, `TRANSFER`, `QRIS`, dan metode manual lain yang didefinisikan pemilik.
- `CREDIT` tidak tersedia di POS. Kredit hanya untuk invoice layanan.
- Satu penjualan dapat dibayar dengan beberapa metode. Jumlah seluruh `sale_payments.amount` harus sama dengan `grand_total`.
- Metode non-tunai: jumlahnya harus persis sama dengan porsi yang dialokasikan ke metode tersebut.
- Metode tunai: `tendered_amount` boleh lebih besar dari porsi. `change_amount = tendered_amount − amount` dihitung otomatis.
- Pada `cash_movements`, `CASH_IN_SALE` dicatat sebesar porsi tunai (`amount`). Kembalian dicatat sebagai `CASH_OUT_CHANGE` sebesar `change_amount`, sehingga rekonsiliasi shift akurat.

### 9.5 Penyelesaian penjualan

`SaleCompletionService::complete($sale, $payments, $idempotencyKey)` dalam satu transaksi dengan `statement_timeout` 5 detik:

1. Validasi: status `DRAFT`, shift `OPEN` milik kasir, cabang sesuai, izin `pos.sell`, tidak ada produk resep.
2. Hitung ulang harga, diskon, dan total dari `product_prices` dan `product_branch_settings`.
3. Validasi pembayaran: metode aktif di cabang, jumlah sesuai.
4. Kunci `sales` (induk), kemudian `stock_balances` terkait sesuai urutan (lihat 19.3), kemudian `product_branch_costs`.
5. Validasi batch (belum kedaluwarsa) dan ketersediaan (`available`). Jika tidak cukup, batalkan dengan `INSUFFICIENT_STOCK` dan detail item yang kurang.
6. Alokasikan batch dengan FEFO. Simpan alokasi di `sale_item_lots`.
7. Kurangi `on_hand`. Buat `SALE_POS` untuk setiap alokasi.
8. Simpan `unit_cost_snapshot` dari `product_branch_costs` di `sale_items`.
9. Buat `sale_payments` dan `cash_movements` untuk setiap metode.
10. Terbitkan nomor `POS` dari `invoice_sequences`.
11. Ubah status menjadi `COMPLETED`, isi `completed_at` dan `completed_by`.
12. Simpan hasil idempotensi (19.4).

Penjualan `COMPLETED` tidak dapat diedit. Koreksi dilakukan melalui retur atau transaksi pembalik yang disetujui.

### 9.6 Retur POS

- Retur hanya untuk item dari penjualan `COMPLETED` atau `PARTIALLY_RETURNED`.
- Jumlah retur per item tidak boleh melebihi jumlah yang belum diretur.
- Setiap item retur memiliki kondisi:
  - `RESTOCK`: barang layak jual. Dikembalikan ke batch asal (`sale_item_lots`) jika batch belum kedaluwarsa. Jika batch kedaluwarsa, kondisi wajib `DAMAGED`. Sistem membuat `CUSTOMER_RETURN_RESTOCK` dengan `unit_cost_snapshot` asal dan memperbarui biaya rata-rata.
  - `DAMAGED`: barang tidak layak jual. Tidak ada perubahan `on_hand`. Nilai kerugian dicatat di `sales_return_items` untuk laporan.
- Retur membuat `sales_returns` dengan nomor `RTN`. Jika ada nilai uang yang dikembalikan, sistem membuat `refunds` berstatus `REQUESTED`.
- Status penjualan menjadi `PARTIALLY_RETURNED` atau `RETURNED`.
- Retur tidak mengubah `sale_items` asal.

### 9.7 Void

- Hanya draft `DRAFT` yang dapat di-void.
- Penjualan `COMPLETED` tidak dapat di-void. Koreksi melalui retur.

---

## 10. Toko Online dan Pembayaran Manual

### 10.1 Katalog dan keranjang

- Katalog menampilkan produk dengan `is_online_sellable = true` dan harga aktif di cabang fulfillment yang dipilih.
- Pelanggan memilih cabang fulfillment saat checkout. Keranjang dibuat per pelanggan per cabang (`customer_carts` unik `(customer_id, branch_id)`).
- Keranjang tidak mereservasi stok.
- Produk `is_prescription` tidak dapat masuk keranjang.
- Keranjang disimpan di basis data. Keranjang tidak disimpan di session browser.

### 10.2 Pembuatan pesanan

`OnlineOrderService::place` dalam satu transaksi:
1. Validasi item, status produk, harga aktif, dan `is_online_sellable`.
2. Hitung `subtotal`, `delivery_fee`, dan `grand_total = subtotal + delivery_fee`.
3. Simpan `orders` dengan `payment_status = UNPAID` dan `fulfillment_status = NOT_STARTED`.
4. Simpan `order_items` dengan snapshot nama, SKU, harga, dan qty.
5. Simpan snapshot instruksi pembayaran aktif cabang di `payment_instructions_snapshot`.
6. Tidak mengurangi atau mereservasi stok.

Pesanan dapat dibatalkan pelanggan selama `payment_status` berupa `UNPAID` atau `SUBMITTED`.

### 10.3 Instruksi pembayaran

- Setiap cabang memiliki satu atau lebih rekening atau metode manual aktif (`branch_payment_accounts`) dengan nama bank atau penyedia, nomor rekening, nama penerima, dan petunjuk.
- Instruksi yang ditampilkan saat checkout disalin ke pesanan sebagai snapshot. Mengubah rekening tidak mengubah pesanan yang sudah ada.

### 10.4 Pembayaran

- Pelanggan mengunggah bukti transfer dan memilih metode. Bukti disimpan di disk `private` dalam folder `payment-proofs/{order_id}/`.
- Setiap unggahan membuat `payments` dengan `kind = ORDER`, `status = SUBMITTED`, dan `amount = orders.grand_total`.
- Hanya satu pembayaran `ORDER` dengan status `SUBMITTED` atau `CONFIRMED` per pesanan (indeks unik parsial).
- Bukti transfer adalah klaim pembayaran, bukan bukti bahwa dana sudah diterima.
- Staf dengan `payment.review` memeriksa bukti dan dapat menolak dengan alasan. Penolakan mengubah pembayaran menjadi `REJECTED` dan `orders.payment_status` kembali ke `UNPAID`. Pelanggan dapat mengunggah ulang.

### 10.5 Konfirmasi pembayaran dan alokasi stok

`OnlinePaymentService::confirm($order, $payment, $idempotencyKey)` dalam satu transaksi:

1. Validasi izin `payment.confirm`, cabang fulfillment, `payment.status = SUBMITTED`, dan `payment.amount = orders.grand_total`.
2. Periksa idempotensi (19.4).
3. Kunci `orders` lalu `payments`.
4. Kunci `stock_balances` untuk setiap item, urut menurut `(product_id, location_id, lot_id)`.
5. Hitung `available` dan alokasi FEFO untuk setiap item.
6. Ubah `payments.status = CONFIRMED`, `confirmed_at`, `confirmed_by`. Ubah `orders.payment_status = CONFIRMED`.
7. Buat `cash_movements` `CASH_IN_ONLINE` sebesar `amount` dengan metode pembayaran.
8. Jika semua item tersedia:
   - Buat `stock_reservations` per alokasi batch dan naikkan `reserved`.
   - `orders.fulfillment_status = PROCESSING`.
9. Jika sebagian atau semua item kurang:
   - Jangan membuat reservasi.
   - `orders.fulfillment_status = STOCK_REVIEW`.
   - Simpan rincian kekurangan di `orders.stock_review_details` (JSON berisi produk, qty diminta, qty tersedia).
10. Catat audit dan hasil idempotensi.

Pembayaran yang sudah dikonfirmasi tidak dibatalkan karena stok kurang. Pembayaran tetap sah dan pesanan masuk `STOCK_REVIEW`.

### 10.6 Penanganan STOCK_REVIEW

`StockReviewService::resolve` hanya dapat dijalankan oleh pengguna dengan `stock_review.resolve`. Setiap tindakan dicatat di `order_review_actions` dan memerlukan `idempotency_key`.

| Tindakan | Syarat | Dampak |
|----------|--------|--------|
| `PARTIAL_FULFILL` | Pelanggan menyetujui secara tertulis (catatan portal atau catatan telepon oleh staf) | Item tersedia direservasi. Qty yang dibatalkan dihitung. Selisih nilai dibuat sebagai `refunds` `REQUESTED`. Pesanan masuk `PROCESSING`. |
| `SUBSTITUTE` | Pelanggan menyetujui pengganti tertulis. Pengganti harus aktif online di cabang yang sama dan tersedia. | Jika harga pengganti lebih rendah: selisih menjadi refund `REQUESTED`, reservasi dibuat, status `PROCESSING`. Jika lebih tinggi: dibuat `payments` `kind = ADDITIONAL` berstatus `SUBMITTED` dengan selisih. Pesanan tetap `STOCK_REVIEW` sampai pembayaran tambahan dikonfirmasi. Konfirmasi memakai alur 10.5 untuk item pengganti. |
| `CANCEL_AND_REFUND` | Selalu tersedia | Pesanan dibatalkan dan refund penuh `REQUESTED` dibuat. `payment_status` menjadi `REFUND_PENDING`. |

Pesanan `STOCK_REVIEW` tidak boleh masuk `PROCESSING`, `READY_FOR_PICKUP`, atau `DISPATCHED` sebelum tindakan tercatat dan reservasi (jika ada) dibuat.

### 10.7 Fulfillment

Status fulfillment: `NOT_STARTED`, `STOCK_REVIEW`, `PROCESSING`, `READY_FOR_PICKUP`, `DISPATCHED`, `COMPLETED`, `CANCELLED`.

Transisi valid:

```
NOT_STARTED      → PROCESSING         (pembayaran dikonfirmasi, stok tersedia)
NOT_STARTED      → STOCK_REVIEW       (pembayaran dikonfirmasi, stok kurang)
NOT_STARTED      → CANCELLED          (dibatalkan sebelum pembayaran dikonfirmasi)
STOCK_REVIEW     → PROCESSING         (setelah PARTIAL_FULFILL atau SUBSTITUTE selesai)
STOCK_REVIEW     → CANCELLED          (CANCEL_AND_REFUND)
PROCESSING       → READY_FOR_PICKUP   (pengambilan di toko)
PROCESSING       → DISPATCHED         (pengiriman)
PROCESSING       → CANCELLED          (dibatalkan dengan refund)
READY_FOR_PICKUP → COMPLETED          (diambil pelanggan)
DISPATCHED       → COMPLETED          (diterima pelanggan)
```

Aturan setiap transisi:
- Dicatat di `order_fulfillments` dengan pelaku, waktu, dan tipe (`PICKUP` atau `DELIVERY`).
- Saat `→ COMPLETED`, setiap reservasi `ACTIVE` menjadi `FULFILLED`. `on_hand` dan `reserved` berkurang bersamaan, dan ledger mencatat `ORDER_FULFILLMENT`.
- Saat `→ CANCELLED` dari `PROCESSING`, reservasi `ACTIVE` menjadi `RELEASED` (`reserved` berkurang), dan refund penuh `REQUESTED` dibuat.
- Pengiriman mencatat nama penerima, alamat (untuk delivery), waktu dispatch, dan waktu diterima. Kurir eksternal tidak dilacak pada Versi 1.

### 10.8 Refund pesanan

- Pembatalan pesanan yang sudah dikonfirmasi membuat refund `REQUESTED`.
- `payment_status` menjadi `REFUND_PENDING` saat refund dibuat, dan `REFUNDED` setelah refund `COMPLETED`.
- Refund mengikuti alur 14.4.

### 10.9 Biaya pengiriman

- Biaya pengiriman tetap per cabang di `branches.delivery_fee`.
- Biaya ditentukan saat checkout dan disimpan di `orders.delivery_fee`. Perubahan tarif tidak mengubah pesanan yang sudah ada.
- Pengambilan di toko tidak dikenai biaya.

---

## 11. Klinik dan Rekam Medis

### 11.1 Jadwal dokter

- `doctor_schedules`: dokter, cabang, hari, jam mulai, jam selesai, durasi slot.
- `appointments` memiliki constraint eksklusi `EXCLUDE USING gist (doctor_id WITH =, tstzrange(starts_at, ends_at) WITH &&) WHERE (status IN ('SCHEDULED','CHECKED_IN','IN_CONSULTATION'))`.
- Pelanggan (wali) membuat janji dari portal. Staf membuat janji untuk walk-in.
- Pembatalan oleh pelanggan wajib minimal 2 jam sebelum mulai. Pembatalan lebih dekat dilakukan oleh staf.

Status janji: `SCHEDULED`, `CHECKED_IN`, `IN_CONSULTATION`, `COMPLETED`, `CANCELLED`, `NO_SHOW`.

Janji `NO_SHOW` tercatat di profil pelanggan sebagai informasi (jumlah dan tanggal).

### 11.2 Kunjungan

- Kunjungan dibuat saat janji dimulai (`CHECKED_IN` menjadi `IN_CONSULTATION`) atau langsung untuk kunjungan tanpa janji.
- `visits`: hewan, cabang, dokter, janji terkait (opsional), status (`OPEN`, `COMPLETED`, `CANCELLED`), waktu mulai dan selesai, berat, suhu, detak jantung, laju napas.
- Persetujuan tindakan dicatat dengan nama wali yang menyetujui dan metode (`IN_PERSON` atau `PHONE`). Persetujuan wajib sebelum tindakan dicatat.
- Dokter kunjungan adalah dokter yang membuka kunjungan atau yang ditugaskan pada janji.

### 11.3 Rekam medis

Status: `DRAFT`, `SIGNED`, `AMENDED`, `VOIDED`.

Isi:
- Kolom terstruktur: `chief_complaint`, `history`, `physical_exam`, `diagnosis`, `treatment_plan`, `follow_up_date`.
- `clinical_data JSONB` untuk bagian formulir tambahan. Kolom yang difilter, dilaporkan, atau wajib diisi harus terstruktur.
- `internal_note TEXT` hanya terlihat staf klinis.

Aturan:
- Satu rekam medis per kunjungan (unik `visit_id`).
- `DRAFT` dapat diubah oleh dokter yang ditugaskan pada kunjungan.
- `DRAFT` dapat di-`VOIDED` oleh dokter pembuatnya dengan alasan. `VOIDED` tidak dapat diubah.
- Penandatanganan melalui `MedicalRecordService::sign`: memeriksa izin `medical_record.sign`, penugasan ke kunjungan, dan profil dokter aktif. Identitas penanda tangan diambil dari sesi, bukan dari input.
- Saat `SIGNED`, `doctor_license_snapshot` menyimpan nomor STR penanda tangan.
- Resep dalam kunjungan yang sama ditandatangani bersamaan dengan rekam medis.
- Setelah `SIGNED`, kolom klinis tidak dapat diubah. Trigger basis data menolak perubahan (18.4).

### 11.4 Amendemen

- Amendemen dibuat untuk rekam medis `SIGNED` atau `AMENDED`.
- Hanya dokter penanda asli dengan `medical_record.amend` yang dapat membuat amendemen.
- `medical_record_amendments` menyimpan: alasan (wajib), `previous_content` (JSON snapshot lengkap sebelum perubahan), `new_content` (JSON isi baru), `amended_by`, dan `amended_at`.
- Setelah amendemen pertama, status menjadi `AMENDED`.
- Snapshot sebelumnya tidak pernah diubah atau dihapus.
- Pelanggan melihat versi terbaru dan tanggal amendemen, tetapi tidak melihat `internal_note`.

### 11.5 Resep dan pengeluaran obat

- `prescriptions` (status `DRAFT`, `SIGNED`, `CANCELLED`) berisi item `prescription_items`: produk, dosis (teks bebas), frekuensi (teks), durasi hari, jumlah, dan instruksi.
- Dokter memasukkan dosis secara manual. Sistem tidak menghitung dosis otomatis.
- Resep `SIGNED` tidak dapat diubah. Pembatalan dilakukan dengan status `CANCELLED` dan alasan.
- Pembuatan resep tidak mengurangi stok.
- Pengeluaran (`medication_dispensings`) dilakukan melalui `MedicationDispensingService::dispense` oleh pengguna dengan `medication.dispense`, dari lokasi `CLINIC_PHARMACY`.
- Pengeluaran dapat sebagian dari resep dan dicatat per kejadian. Jika produk `is_prescription`, pengeluaran wajib terkait `prescription_item_id`.
- Setiap pengeluaran mengambil batch dengan FEFO, membuat `CLINIC_USE` per batch, dan menyimpan `unit_cost_snapshot` dan `unit_price_snapshot`. Satu pengeluaran dengan beberapa batch menghasilkan beberapa baris `medication_dispensings`.
- Biaya pengeluaran dimasukkan ke invoice layanan kunjungan atau rawat inap terkait sebagai item `MEDICATION`.

### 11.6 Tindakan dan bahan habis pakai

- Tindakan berasal dari `service_catalog` dengan kategori `CLINIC`.
- `treatment_records` mencatat tindakan, dokter, waktu, catatan, dan biaya layanan. Tindakan memerlukan persetujuan wali (11.2).
- Bahan habis pakai (kategori `CLINIC_SUPPLY`) dikeluarkan dengan `medication_dispensings` dan `reason = PROCEDURE`.

### 11.7 Laporan layanan dokter

Laporan per dokter dan rentang tanggal menampilkan:
- Jumlah kunjungan selesai.
- Jumlah tindakan.
- Jumlah rawat inap yang ditangani.
- Total nilai layanan dari invoice terkait.
- Jumlah rekam medis `DRAFT` yang belum ditandatangani.

Laporan tidak menghitung gaji atau komisi.

---

## 12. Rawat Inap

### 12.1 Unit rawat

- `care_units`: cabang, kode (unik per cabang), nama, tipe (`KENNEL`, `ICU`, `ISOLATION`, `BOARDING_ROOM`), kapasitas (minimal 1), status aktif.
- `BOARDING_ROOM` untuk penitipan. `KENNEL`, `ICU`, dan `ISOLATION` untuk rawat inap klinis.

### 12.2 Penempatan

- `hospitalization_assignments`: `hospitalization_id`, `unit_id`, `start_at`, `end_at` (`end_at > start_at`; `end_at` NULL berarti masih aktif).
- Penempatan tidak boleh melebihi kapasitas unit pada rentang waktu yang bertabrakan.
- Pemeriksaan kapasitas dilakukan di `CareUnitAllocator::assign` dalam transaksi dengan `lockForUpdate()` pada baris `care_units`.
- Pindah unit menutup penempatan lama dan membuat penempatan baru dalam satu transaksi.

### 12.3 Episode rawat inap

Status: `PLANNED`, `ADMITTED`, `DISCHARGE_PENDING`, `DISCHARGED`, `CANCELLED`.

- `PLANNED` dibuat saat penempatan dijadwalkan.
- `ADMITTED`: hewan sudah masuk unit. Pemantauan aktif.
- `DISCHARGE_PENDING`: dokter memulai proses pulang. Invoice layanan final dibuat.
- `DISCHARGED`: dokter menyelesaikan proses pulang. Semua penempatan ditutup dan unit dilepas.
- Pulang tidak mensyaratkan pelunasan. Sisa tagihan menjadi piutang (14.3).
- `CANCELLED` hanya dari `PLANNED`.

### 12.4 Observasi dan tugas

- `care_observations`: tipe (`VITALS`, `MEDICATION_GIVEN`, `FEEDING`, `URINATION`, `DEFECATION`, `PROCEDURE`, `NOTE`), `observed_at`, `values` (JSONB), `note`, `recorded_by`. Append-only.
- Observasi `MEDICATION_GIVEN` wajib terhubung dengan `medication_dispensings` agar stok tercatat.
- `care_tasks`: jenis, jatuh tempo, penerima, status (`OPEN`, `DONE`, `CANCELLED`), `done_at`, `done_by`.
- Observasi dan tugas internal dan tidak terlihat pelanggan.

### 12.5 Update pelanggan

- `shared_updates`: jenis (`DAILY_SUMMARY`, `PROGRESS`, `DISCHARGE_SUMMARY`), isi, `published_at`, `published_by`. Append-only.
- Dokter dengan `hospitalization.publish_update` membuat dan mempublikasikan update. Update dapat dibuat dari ringkasan observasi atau ditulis bebas.
- Update tidak menyalin observasi internal secara otomatis. Dokter memilih isi yang ditampilkan.
- Update yang sudah dipublikasikan tidak dapat diubah. Koreksi dibuat sebagai update baru.
- Wali hewan melihat update yang dipublikasikan.

---

## 13. Grooming dan Penitipan

### 13.1 Katalog layanan

- `service_catalog`: kode, nama, kategori (`GROOMING`, `BOARDING`, `CLINIC`), `duration_minutes`, `is_addon`, status aktif.
- Harga layanan disimpan di `service_prices` per cabang dengan struktur yang sama seperti `product_prices` (`service_id`, `branch_id`, `price`, `effective_from`, `created_by`). Tabel ini bersifat append-only.
- Layanan tambahan (`is_addon = true`) memerlukan persetujuan wali utama sebelum ditagih.

### 13.2 Kapasitas

- `service_capacity`: cabang, layanan, slot (`starts_at`, `ends_at`), dan kapasitas (minimal 1).
- Slot grooming dihitung dari durasi layanan dan jam operasional cabang.
- Penitipan memakai `care_units` bertipe `BOARDING_ROOM` dan pemeriksaan kapasitas yang sama dengan rawat inap.
- Booking yang melebihi kapasitas ditolak dengan `INSUFFICIENT_CAPACITY`.

### 13.3 Booking

Status: `REQUESTED`, `CONFIRMED`, `CHECKED_IN`, `IN_PROGRESS`, `COMPLETED`, `CANCELLED`, `NO_SHOW`.

- Pelanggan membuat `REQUESTED`. Staf mengonfirmasi menjadi `CONFIRMED`.
- Staf dapat membuat booking langsung `CONFIRMED`.
- Pembatalan pelanggan minimal 4 jam sebelum slot. Pembatalan lebih dekat dilakukan staf.
- Perubahan jadwal membuat booking baru dan membatalkan yang lama dalam satu transaksi. Keduanya dicatat di audit.

### 13.4 Penitipan

- Booking tipe `BOARDING` memiliki `boarding_stays` (tanggal masuk, tanggal keluar, unit).
- Check-in mencatat kondisi awal (berat, catatan kondisi, foto opsional) dan membuat penempatan.
- Check-out mencatat kondisi akhir, menutup penempatan, dan membuat invoice layanan final.

### 13.5 Layanan tambahan

- Persetujuan: `PENDING`, `APPROVED`, `DECLINED`.
- Persetujuan oleh wali utama melalui portal, atau dicatat staf dengan metode `PHONE` dan nama pemberi.
- `PENDING` dan `DECLINED` tidak masuk invoice. `DECLINED` tercatat sebagai keputusan pelanggan.

### 13.6 Check-in dan check-out grooming

- Check-in mencatat kondisi awal dan waktu mulai, dan mengubah status menjadi `CHECKED_IN` lalu `IN_PROGRESS`.
- Check-out mencatat kondisi akhir, catatan petugas, layanan final, mengubah status menjadi `COMPLETED`, dan membuat invoice layanan.

---

## 14. Tagihan Layanan, Piutang, dan Refund

### 14.1 Invoice layanan

- `service_invoices` dibuat untuk kunjungan, episode rawat inap, booking grooming, dan penitipan. Satu invoice aktif per sumber (B33).
- Status: `OPEN`, `PARTIALLY_PAID`, `PAID`, `VOIDED`.
- `service_invoice_items`: jenis (`CONSULTATION`, `TREATMENT`, `MEDICATION`, `SUPPLY`, `BOARDING_DAY`, `GROOMING`, `ADDON`, `OTHER`), referensi sumber, deskripsi, qty, harga snapshot, diskon, total, dan `unit_cost_snapshot` (untuk HPP).
- Total dihitung server dengan pembulatan `HALF_UP` ke rupiah penuh. Diskon invoice hanya oleh pemilik (`service_invoice.discount`).
- Invoice dapat ditambah item selama `OPEN` atau `PARTIALLY_PAID`. Penambahan dicatat sebagai item baru.
- `VOIDED` hanya jika belum ada pembayaran, dengan alasan dan izin `service_invoice.void`.
- Setelah pembayaran pertama, koreksi dilakukan dengan item penyesuaian (qty negatif atau baris koreksi), bukan dengan menghapus item.

### 14.2 Pembayaran layanan

- `service_payments`: invoice, metode, jumlah, waktu, pencatat, referensi.
- Pembayaran parsial diizinkan. Jumlah tidak boleh melebihi `balance_due`.
- Setiap pembayaran membuat `cash_movements` `CASH_IN_SERVICE` untuk cabang invoice. Pembayaran tunai wajib shift `OPEN` milik pencatat.
- `amount_paid` dan `balance_due` diperbarui dalam transaksi yang sama. `balance_due = grand_total − amount_paid`.
- Pembayaran tidak dihapus. Koreksi dilakukan dengan pembayaran negatif bertanda alasan, hanya oleh pengguna dengan `finance.cash.read` dan dengan izin koreksi pemilik.
- Kelebihan pembayaran (melebihi `balance_due`) tidak diterima sebagai pembayaran biasa. Staf memilih: (a) mengembalikan selisih sebagai refund, atau (b) menyimpan sebagai `customer_credits`. Pilihan dicatat.

### 14.3 Piutang

- Piutang pelanggan adalah jumlah `balance_due` seluruh invoice berstatus `OPEN` atau `PARTIALLY_PAID`.
- Laporan piutang menampilkan umur piutang: 0–30, 31–60, 61–90, dan di atas 90 hari sejak `due_date`.
- Piutang ditampilkan kepada wali utama di portal.
- Job `receivable_reminders` harian mengirim email pengingat untuk invoice jatuh tempo.

### 14.4 Refund

Refund berasal dari: pembatalan pesanan online yang dikonfirmasi, retur POS yang menghasilkan nilai, tindakan `PARTIAL_FULFILL` atau `SUBSTITUTE` yang menghasilkan selisih, dan pengembalian kredit pelanggan.

Tabel `refunds` memiliki: `source_type`, `source_id`, `amount`, `reason`, `status`, `requested_by`, `approved_by`, `completed_by`, `payout_method`, `payout_reference`, `rejection_reason`, dan nomor `RFD`.

Alur:
1. Pemohon (`refund.request`) membuat refund `REQUESTED` dengan jumlah, alasan, dan sumber.
2. Pengguna dengan `refund.approve`, selain pemohon, menyetujui (`APPROVED`) atau menolak (`REJECTED`) dengan alasan.
3. Pengguna dengan `refund.complete` mencatat pengembalian dana secara manual: metode (`CASH` atau `TRANSFER`), referensi transfer, dan bukti opsional. Status `COMPLETED`.
4. Jika metode `CASH`, shift pencatat harus `OPEN`. `cash_movements` `CASH_OUT_REFUND` dibuat.
5. Jika metode `TRANSFER`, `cash_movements` `CASH_OUT_REFUND` dibuat dengan metode transfer dan referensi.

Aturan:
- Jumlah refund tidak boleh melebihi jumlah yang dibayar dikurangi refund yang sudah `APPROVED` atau `COMPLETED`.
- Refund yang sudah `COMPLETED` tidak dapat diubah.
- Penolakan tidak mengubah status pesanan atau penjualan, kecuali sebagai catatan.

---

## 15. Keuangan, Kas, dan Laporan

### 15.1 Buku kas

`cash_movements` mencatat seluruh uang masuk dan keluar per cabang dan metode. Bersifat append-only.

Tipe dan arah:

| Tipe | Arah | Sumber |
|------|------|--------|
| `CASH_IN_SALE` | Masuk | Penjualan POS |
| `CASH_IN_ONLINE` | Masuk | Pembayaran pesanan online yang dikonfirmasi |
| `CASH_IN_SERVICE` | Masuk | Pembayaran invoice layanan |
| `CASH_OUT_CHANGE` | Keluar | Kembalian POS |
| `CASH_OUT_REFUND` | Keluar | Refund `COMPLETED` |
| `CASH_OUT_EXPENSE` | Keluar | Pengeluaran operasional yang disetujui |
| `CASH_OUT_SUPPLIER` | Keluar | Pembayaran tagihan supplier |
| `SHIFT_OPENING` | Masuk | Saldo awal shift (metode tunai) |
| `SHIFT_CLOSING_DIFFERENCE` | Masuk atau keluar | Selisih penutupan shift (sesuai tanda) |

Kolom: `branch_id`, `shift_id` (wajib untuk metode tunai, NULL untuk non-tunai), `movement_type`, `method_code`, `amount` (selalu positif), `direction` (`IN` atau `OUT`, diturunkan dari tipe), `reference_type`, `reference_id`, `occurred_at`.

Saldo tunai shift dihitung dari baris dengan `method_code = CASH` dan `shift_id` yang sama. Saldo per rekening transfer dihitung dari `method_code` dan referensi rekening.

### 15.2 Pengeluaran operasional

- Pengeluaran memiliki kategori, jumlah, tanggal pengeluaran, cabang, pemohon, bukti (opsional), dan status (`PENDING_APPROVAL`, `APPROVED`, `REJECTED`).
- Jumlah ≤ Rp 1.000.000: status langsung `APPROVED` dan pemohon dapat langsung membayar.
- Jumlah di atas Rp 1.000.000: `PENDING_APPROVAL` sampai disetujui pengguna lain dengan `finance.expense.approve`.
- Saat `APPROVED` dan dibayar, sistem membuat `cash_movements` `CASH_OUT_EXPENSE`. Pembayaran tunai wajib shift `OPEN`.
- Pengeluaran `APPROVED` tidak dapat diubah. Koreksi dilakukan dengan pengeluaran penyesuaian yang juga memerlukan persetujuan jika melewati batas.

### 15.3 Periode keuangan

- Periode bulanan per cabang (`financial_periods`) ditutup oleh pengguna dengan `finance.period.close`.
- Tanggal transaksi dalam periode `CLOSED` tidak dapat dibuat, diubah, atau dibatalkan.
- Koreksi periode tertutup dicatat pada periode aktif berikutnya dengan referensi ke periode asal.
- Penutupan menghasilkan ringkasan tersimpan: penjualan, penerimaan kas per metode, pengeluaran, piutang, nilai persediaan, HPP, dan laba kotor.
- Penutupan periode tidak dapat dibuka kembali. Koreksi dilakukan sesuai aturan di atas.

### 15.4 Definisi laporan

| Laporan | Definisi | Sumber |
|---------|----------|--------|
| Penjualan POS | Total `grand_total` penjualan `COMPLETED`, `PARTIALLY_RETURNED`, dan `RETURNED`, dikurangi nilai retur, dalam rentang tanggal | `sales`, `sales_returns` |
| Penjualan online | Total `grand_total` pesanan dengan `payment_status` `CONFIRMED` atau setelahnya, dikurangi refund `COMPLETED` | `orders`, `refunds` |
| Penerimaan kas | Seluruh `cash_movements` masuk per metode | `cash_movements` |
| Pengeluaran | Pengeluaran `APPROVED` dalam rentang tanggal | `operating_expenses` |
| Piutang | `balance_due` invoice layanan belum lunas, per umur | `service_invoices` |
| Nilai persediaan | Σ `on_hand × avg_unit_cost` per cabang | `stock_balances`, `product_branch_costs` |
| HPP | Σ `quantity × unit_cost_snapshot` dari penjualan, pengeluaran klinik, dan item pesanan yang diserahkan, dikurangi HPP retur layak jual | `sale_items`, `medication_dispensings`, `order_items`, `sales_returns` |
| Laba kotor | Penjualan dikurangi HPP | Turunan |
| Laba rugi sederhana | Laba kotor dikurangi pengeluaran operasional, dikurangi kerugian (retur rusak dan kehilangan transfer) | Turunan |
| Arus kas sederhana | Penerimaan kas dikurangi pengeluaran kas dalam periode | `cash_movements` |
| Kehilangan stok | Nilai `quantity_lost` transfer dan penyesuaian negatif `LOST`, `EXPIRY_WRITE_OFF`, `DAMAGED` | `inventory_transfer_items`, `inventory_adjustments` |
| Klinik per dokter | Lihat 11.7 | `visits`, `treatment_records`, `service_invoices` |

Catatan:
- Pembelian stok bukan beban pada hari pembelian. Beban HPP diakui saat barang terjual atau dipakai.
- Laba rugi berbeda dari arus kas. Piutang dan pembayaran supplier tidak masuk laba rugi secara langsung.
- Setiap baris laporan memiliki tautan ke ID transaksi sumber.

### 15.5 Ekspor

- Setiap laporan dapat diekspor ke CSV (`fputcsv`, UTF-8 dengan BOM untuk Excel) dan PDF (dompdf).
- Ekspor PDF memakai template Blade dengan kop cabang dan informasi pencetak.
- Setiap ekspor dicatat di audit (`report.export`) dengan parameter filter dan pengguna.

---

## 16. Portal Pelanggan

Pelanggan dengan akun dan email terverifikasi dapat:

- Melihat hewan yang tertaut dengan akunnya beserta perannya (utama atau sekunder).
- Membuat janji temu klinik dan booking grooming atau penitipan.
- Melihat riwayat kunjungan dan rekam medis `SIGNED` atau `AMENDED` dalam bentuk terstruktur. `internal_note` dan observasi internal tidak ditampilkan.
- Melihat update rawat inap yang dipublikasikan.
- Melihat pesanan online sendiri, membatalkan pesanan yang belum dikonfirmasi, dan mengunggah bukti pembayaran.
- Melihat invoice layanan untuk hewan yang tertaut. Wali utama melihat angka tagihan. Wali sekunder melihat riwayat tanpa angka tagihan.
- Menyetujui atau menolak layanan tambahan sebagai wali utama.
- Melihat riwayat pembelian POS yang tercatat atas nomor telepon atau email yang sama dengan akunnya dan sudah terverifikasi.
- Mengubah profil kontak dan alamat sendiri.

Pelanggan tidak dapat melihat stok internal, HPP, harga pokok, data pelanggan lain, atau data pengguna staf.

---

## 17. Notifikasi

Saluran: notifikasi database (tabel `notifications`) dan email (Resend). Notifikasi dikirim melalui antrean.

| Peristiwa | Penerima | Saluran |
|-----------|----------|---------|
| Bukti pembayaran diunggah | Staf dengan `payment.review` di cabang | Database |
| Pembayaran dikonfirmasi | Pelanggan | Database, email |
| Pembayaran ditolak | Pelanggan | Database, email |
| Pesanan masuk `STOCK_REVIEW` | Staf cabang fulfillment dan pelanggan | Database, email |
| Status fulfillment berubah | Pelanggan | Database, email |
| Janji dibuat, diubah, atau dibatalkan | Pelanggan dan dokter | Database, email |
| Booking dikonfirmasi | Pelanggan | Database, email |
| Update rawat inap dipublikasikan | Wali hewan | Database, email |
| Invoice baru atau jatuh tempo | Wali utama | Database, email |
| Refund disetujui atau selesai | Pemohon dan pelanggan terkait | Database (pemohon), database dan email (pelanggan) |
| Stok di bawah minimum | Pemilik dan admin cabang | Database |
| Batch kedaluwarsa dalam 90 hari | Pemilik | Database |
| Selisih rekonsiliasi stok | Pemilik | Database, email |
| Job gagal dua kali berturut-turut | Pemilik dan developer on-call | Email |

Aturan isi:
- Email hanya memuat judul umum, nomor referensi, dan tautan ke portal.
- Email tidak memuat diagnosis, nama obat, hasil pemeriksaan, nominal invoice detail, atau isi update klinis.
- Notifikasi database untuk staf dapat memuat nama pelanggan dan nama hewan karena pengguna sudah terautentikasi.
- Notifikasi dapat ditandai dibaca. Notifikasi yang sudah dibaca dapat dihapus oleh pemilik notifikasi.

---

## 18. Basis Data

### 18.1 Konvensi

- Setiap tabel bisnis memiliki `id` bertipe `UUID` (`uuid` dengan default `gen_random_uuid()`), `created_at`, dan `created_by` (nullable untuk data sistem).
- Tabel yang dapat diubah memiliki `updated_at` dan `updated_by`.
- Setiap entitas terikat cabang memiliki `branch_id`.
- Nama tabel dan kolom `snake_case` bahasa Inggris. Nilai status dalam huruf besar.
- Uang: `NUMERIC(14,2)`. Biaya satuan: `NUMERIC(14,4)`. Qty `COUNT`: `NUMERIC(14,0)`. Qty `WEIGHT`: `NUMERIC(14,3)`.
- Setiap kolom uang dan qty memiliki `CHECK` nonnegatif, kecuali kolom yang secara eksplisit bertanda (pembayaran koreksi layanan).
- Setiap foreign key memakai `ON DELETE RESTRICT`. `CASCADE` tidak digunakan.
- Status memakai `CHECK (status IN (...))` pada kolom `varchar`, bukan tipe `ENUM` PostgreSQL, agar mudah diubah lewat migration. Nilai juga dipetakan ke PHP `enum` backed string.
- Penghapusan fisik data bisnis tidak tersedia di aplikasi. Model memakai `SoftDeletes` hanya untuk entitas master (produk, supplier, layanan) yang tidak pernah terhubung ke transaksi dan dapat diarsipkan.

### 18.2 Daftar tabel

**Identitas dan akses**
`users`, `branches`, `branch_settings`, `user_branch_access`, `doctor_profiles`, `permissions`, `role_permissions`, `user_permission_grants`, `invitations`, `audit_logs`, `idempotency_keys`, `notifications`, `job_runs`, `sessions`, `password_reset_tokens`, `jobs`, `failed_jobs`, `cache`

**Pelanggan dan hewan**
`customers`, `pets`, `pet_guardians`, `pet_documents`

**Produk dan pembayaran**
`product_categories`, `products`, `product_barcodes`, `product_branch_settings`, `product_prices`, `suppliers`, `payment_methods`, `branch_payment_methods`, `branch_payment_accounts`

**Inventori**
`inventory_locations`, `inventory_lots`, `stock_balances`, `inventory_movements`, `stock_reservations`, `inventory_adjustments`, `count_sessions`, `count_session_items`, `inventory_transfers`, `inventory_transfer_items`, `product_branch_costs`, `cost_history`

**Pembelian**
`purchase_orders`, `purchase_order_items`, `goods_receipts`, `goods_receipt_items`, `supplier_returns`, `supplier_return_items`, `supplier_bills`, `supplier_payments`

**POS**
`cash_shifts`, `sales`, `sale_items`, `sale_item_lots`, `sale_payments`, `sales_returns`, `sales_return_items`, `invoice_sequences`

**Toko online**
`customer_carts`, `customer_cart_items`, `orders`, `order_items`, `payments`, `order_review_actions`, `order_fulfillments`

**Refund dan kredit**
`refunds`, `customer_credits`

**Klinik**
`service_catalog`, `doctor_schedules`, `appointments`, `visits`, `medical_records`, `medical_record_amendments`, `prescriptions`, `prescription_items`, `treatment_records`, `medication_dispensings`

**Rawat inap**
`care_units`, `hospitalizations`, `hospitalization_assignments`, `care_observations`, `care_tasks`, `shared_updates`

**Grooming dan penitipan**
`service_bookings`, `service_booking_items`, `boarding_stays`, `service_capacity`, `service_prices`

**Tagihan dan keuangan**
`service_invoices`, `service_invoice_items`, `service_payments`, `cash_movements`, `operating_expenses`, `financial_periods`

### 18.3 Kolom dan constraint kritis

| Tabel | Kolom penting | Constraint dan indeks |
|-------|---------------|-----------------------|
| `users` | `name`, `email`, `password`, `role` (`OWNER`, `DOCTOR`, `ADMIN`, `CUSTOMER`), `is_active`, `mfa_enabled`, `email_verified_at` | `email` unik; `role` dalam daftar peran |
| `doctor_profiles` | `user_id`, `license_number` (STR), `is_active` | `user_id` unik; `license_number` unik; `role = DOCTOR` atau `OWNER` yang bertindak klinis wajib memiliki profil aktif (validasi aplikasi saat pengaktifan peran dan izin klinis) |
| `user_permission_grants` | `user_id`, `permission_id`, `reason`, `granted_by`, `expires_at` | Unik aktif `(user_id, permission_id)` |
| `invitations` | `email`, `role`, `token_hash`, `branch_ids`, `expires_at`, `used_at` | `token_hash` unik |
| `branches` | `code`, `name`, `timezone`, `delivery_fee`, `pos_enabled`, `is_active` | `code` unik; `timezone` dalam daftar tiga zona; `delivery_fee >= 0` |
| `pets` | `name`, `species`, `breed`, `sex`, `is_neutered`, `birth_date`, `birth_date_is_estimate`, `microchip`, `weight_kg`, `is_deceased`, `deceased_at` | `microchip` unik jika terisi; enum spesies dan jenis kelamin |
| `pet_guardians` | `pet_id`, `customer_id`, `is_primary`, `ended_at` | Indeks unik parsial: satu `is_primary` aktif per `pet_id` |
| `products` | `sku`, `name`, `category`, `unit_type`, `base_unit`, `track_lot`, `is_prescription`, `is_active` | `sku` unik; `category = MEDICINE` menyebabkan `track_lot = true` (CHECK) |
| `product_prices` | `product_id`, `branch_id`, `price`, `effective_from`, `created_by` | `price >= 0`; `price = trunc(price)`; append-only |
| `service_prices` | `service_id`, `branch_id`, `price`, `effective_from`, `created_by` | `price >= 0`; `price = trunc(price)`; append-only |
| `product_branch_settings` | `is_pos_sellable`, `is_online_sellable`, `min_stock`, `is_active` | Unik `(product_id, branch_id)`; `is_online_sellable` dan `is_pos_sellable` tidak aktif jika `products.is_prescription` |
| `inventory_locations` | `branch_id`, `code`, `type` (`RETAIL`, `CLINIC_PHARMACY`, `WAREHOUSE`) | Unik `(branch_id, code)`; indeks unik parsial satu `RETAIL` dan satu `CLINIC_PHARMACY` per cabang |
| `inventory_lots` | `product_id`, `location_id`, `lot_number`, `expiry_date`, `received_at` | Unik `(product_id, location_id, lot_number)`; `expiry_date` wajib untuk `track_lot` kecuali `DEFAULT` |
| `stock_balances` | `product_id`, `location_id`, `lot_id`, `on_hand`, `reserved` | Unik `(product_id, location_id, lot_id)`; `CHECK (on_hand >= 0 AND reserved >= 0 AND reserved <= on_hand)` |
| `inventory_movements` | `product_id`, `location_id`, `lot_id`, `quantity_delta`, `movement_type`, `reference_type`, `reference_id`, `unit_cost_snapshot`, `reason_code`, `actor_id`, `occurred_at` | Append-only; `quantity_delta <> 0`; tipe dalam daftar 7.3 |
| `stock_reservations` | `order_item_id`, `lot_id`, `product_id`, `location_id`, `quantity`, `status` | Unik `(order_item_id, lot_id)`; `quantity > 0` |
| `product_branch_costs` | `product_id`, `branch_id`, `avg_unit_cost NUMERIC(14,4)` | Unik `(product_id, branch_id)`; `avg_unit_cost >= 0` |
| `cash_shifts` | `branch_id`, `cashier_id`, `status` (`OPEN`, `CLOSED`), `opening_cash`, `closing_cash_counted`, `expected_cash`, `difference`, `difference_reason` | Indeks unik parsial satu `OPEN` per `cashier_id` |
| `sales` | `branch_id`, `shift_id`, `customer_id` (nullable), `invoice_number`, `status`, `subtotal`, `discount_total`, `grand_total`, `completed_at`, `completed_by` | `invoice_number` unik; `grand_total >= 0`; `shift_id` wajib untuk status `COMPLETED` |
| `sale_items` | `sale_id`, `product_id`, `quantity`, `unit_price`, `discount_type`, `discount_value`, `line_gross`, `line_discount`, `line_total`, `unit_cost_snapshot`, `returned_quantity` | `quantity > 0`; `returned_quantity <= quantity` |
| `sale_item_lots` | `sale_item_id`, `lot_id`, `quantity` | `quantity > 0` |
| `sale_payments` | `sale_id`, `method_code`, `amount`, `tendered_amount`, `change_amount` | Append-only; `tendered_amount >= amount` untuk tunai |
| `orders` | `customer_id`, `branch_id`, `payment_status`, `fulfillment_status`, `subtotal`, `delivery_fee`, `grand_total`, `payment_instructions_snapshot`, `stock_review_details` | `grand_total = subtotal + delivery_fee` (CHECK); status dalam daftar 10.5 dan 10.7 |
| `order_items` | `order_id`, `product_id`, `name_snapshot`, `sku_snapshot`, `unit_price`, `quantity`, `cancelled_quantity`, `fulfilled_quantity` | `quantity > 0`; `cancelled_quantity + fulfilled_quantity <= quantity` |
| `payments` | `order_id`, `kind` (`ORDER`, `ADDITIONAL`), `method_code`, `amount`, `status` (`SUBMITTED`, `CONFIRMED`, `REJECTED`), `proof_path`, `submitted_at`, `confirmed_at`, `confirmed_by`, `reviewed_by`, `rejection_reason` | Indeks unik parsial: satu `ORDER` `SUBMITTED` atau `CONFIRMED` per pesanan; `amount > 0` |
| `refunds` | `source_type`, `source_id`, `amount`, `reason`, `status` (`REQUESTED`, `APPROVED`, `COMPLETED`, `REJECTED`), `requested_by`, `approved_by`, `completed_by`, `payout_method`, `payout_reference`, `invoice_number` | `approved_by <> requested_by` (CHECK, nullable); `amount > 0`; trigger melarang perubahan setelah `COMPLETED` |
| `appointments` | `pet_id`, `doctor_id`, `branch_id`, `starts_at`, `ends_at`, `status`, `booked_by` | Constraint eksklusi dokter (18.4); `ends_at > starts_at` |
| `visits` | `pet_id`, `branch_id`, `doctor_id`, `appointment_id` (nullable), `status`, `started_at`, `completed_at`, `consent_given_by_name`, `consent_method`, `weight_kg`, `temperature_c`, `heart_rate`, `respiratory_rate` | Unik `appointment_id` jika terisi |
| `medical_records` | `visit_id`, `status`, `chief_complaint`, `history`, `physical_exam`, `diagnosis`, `treatment_plan`, `follow_up_date`, `clinical_data`, `internal_note`, `signed_by`, `signed_at`, `doctor_license_snapshot` | `visit_id` unik; trigger 18.4 |
| `medical_record_amendments` | `medical_record_id`, `reason`, `previous_content`, `new_content`, `amended_by`, `amended_at` | Append-only |
| `medication_dispensings` | `visit_id` atau `hospitalization_id`, `prescription_item_id` (nullable), `product_id`, `location_id`, `lot_id`, `quantity`, `unit_cost_snapshot`, `unit_price_snapshot`, `reason` (`PRESCRIPTION`, `PROCEDURE`, `WARD`), `dispensed_by` | Append-only; satu baris per batch; `quantity > 0`; `visit_id` atau `hospitalization_id` tepat salah satu |
| `care_units` | `branch_id`, `code`, `type`, `capacity`, `is_active` | Unik `(branch_id, code)`; `capacity >= 1` |
| `hospitalization_assignments` | `hospitalization_id`, `unit_id`, `start_at`, `end_at` | `end_at > start_at` jika terisi; kapasitas diperiksa di service |
| `care_observations` | `hospitalization_id`, `observed_at`, `observation_type`, `values`, `note`, `recorded_by` | Append-only |
| `shared_updates` | `hospitalization_id`, `update_type`, `body`, `published_at`, `published_by` | Append-only; `published_at` wajib |
| `service_bookings` | `branch_id`, `pet_id`, `booked_by`, `service_id`, `type` (`GROOMING`, `BOARDING`), `status`, `starts_at`, `ends_at` | `ends_at > starts_at`; kapasitas diperiksa di service |
| `service_invoices` | `branch_id`, `customer_id`, `source_type`, `source_id`, `invoice_number`, `status`, `subtotal`, `discount_total`, `grand_total`, `amount_paid`, `balance_due`, `due_date` | `balance_due = grand_total − amount_paid` (CHECK); `balance_due >= 0`; unik `(source_type, source_id)` jika `status <> 'VOIDED'` |
| `service_invoice_items` | `invoice_id`, `item_type`, `source_type`, `source_id`, `description`, `quantity`, `unit_price`, `discount_amount`, `line_total`, `unit_cost_snapshot` | `line_total >= 0` |
| `service_payments` | `invoice_id`, `method_code`, `amount`, `paid_at`, `recorded_by`, `shift_id` (nullable), `reference`, `is_correction` | `amount <> 0`; koreksi wajib `is_correction = true` dan alasan |
| `customer_credits` | `customer_id`, `source_type`, `source_id`, `amount`, `used_amount` | `used_amount <= amount`; `amount > 0` |
| `cash_movements` | `branch_id`, `shift_id` (wajib untuk tunai), `movement_type`, `method_code`, `direction`, `amount`, `reference_type`, `reference_id`, `occurred_at` | Append-only; `amount > 0`; `direction` sesuai tipe; `shift_id IS NOT NULL` jika `method_code = 'CASH'` |
| `operating_expenses` | `branch_id`, `category`, `amount`, `expense_date`, `requested_by`, `approved_by`, `status`, `period_id` | `approved_by <> requested_by` (CHECK, nullable) |
| `financial_periods` | `branch_id`, `period_start`, `period_end`, `status` (`OPEN`, `CLOSED`), `closed_by`, `closed_at`, `summary` (JSON) | Tidak ada overlap periode per cabang (`EXCLUDE` dengan `daterange`) |
| `invoice_sequences` | `branch_id`, `doc_type`, `year`, `last_number` | Unik `(branch_id, doc_type, year)` |
| `audit_logs` | `actor_id`, `action`, `entity_type`, `entity_id`, `branch_id`, `occurred_at`, `request_id`, `reason`, `metadata` (JSON) | Append-only (trigger 18.4) |
| `idempotency_keys` | `user_id`, `scope`, `key`, `request_hash`, `status` (`IN_PROGRESS`, `DONE`), `result_type`, `result_id`, `expires_at` | Unik `(user_id, scope, key)` |
| `job_runs` | `job_name`, `started_at`, `finished_at`, `status` (`SUCCESS`, `FAILED`), `detail` (JSON) | Append-only |

Catatan: harga layanan disimpan di `service_prices` (bukan `product_prices`) karena `service_id` merujuk ke `service_catalog`, bukan `products`.

### 18.4 Constraint eksklusi dan trigger

**Constraint eksklusi janji dokter**

```sql
ALTER TABLE appointments ADD CONSTRAINT appointments_no_overlap
EXCLUDE USING gist (
  doctor_id WITH =,
  tstzrange(starts_at, ends_at) WITH &&
) WHERE (status IN ('SCHEDULED','CHECKED_IN','IN_CONSULTATION'));
```

Ekstensi `btree_gist` wajib aktif.

**Trigger immutability rekam medis**
- `BEFORE UPDATE ON medical_records`: jika `OLD.status = 'SIGNED'` atau `'AMENDED'`, tolak perubahan kolom `chief_complaint`, `history`, `physical_exam`, `diagnosis`, `treatment_plan`, `follow_up_date`, `clinical_data`, `internal_note`. Perubahan `status` hanya diizinkan ke `AMENDED` melalui service amendemen.
- `BEFORE DELETE ON medical_records`: tolak jika `OLD.status <> 'DRAFT'`.

**Trigger append-only**
Tabel berikut menolak `UPDATE` dan `DELETE` melalui trigger `BEFORE UPDATE OR DELETE`: `inventory_movements`, `cash_movements`, `audit_logs`, `medical_record_amendments`, `shared_updates`, `sale_payments`, `product_prices`, `service_prices`, `cost_history`, `care_observations`, `medication_dispensings`, `service_payments`, `job_runs`. Koreksi selalu berupa baris baru.

**Trigger stok**
- `BEFORE UPDATE ON stock_balances`: tolak jika perubahan tidak dilakukan dengan `SET LOCAL app.allow_stock_write = 'on'` dari service. Pengaman ini mencegah update langsung dari kode selain service.

**Trigger refund**
- Refund berstatus `COMPLETED` dan `REJECTED` tidak dapat diubah.

**Trigger pembayaran**
- `payments.status` hanya dapat berubah dari `SUBMITTED` ke `CONFIRMED` atau `REJECTED`. Perubahan lain ditolak.

**Trigger shift**
- `cash_shifts` dengan `status = 'CLOSED'` tidak dapat diubah kecuali kolom `updated_at`.

Trigger di atas bertindak sebagai lapisan kedua. Aturan utama tetap di service.

### 18.5 Indeks

Indeks dibuat untuk pola kueri yang dipakai:

- Pencarian produk: `products(sku)`, `product_barcodes(barcode)`, GIN trigram `products(name gin_trgm_ops)`.
- Saldo: `stock_balances(location_id, product_id)`.
- Ledger: `inventory_movements(product_id, location_id, occurred_at)`, `cash_movements(branch_id, occurred_at)`.
- Pesanan: `orders(branch_id, fulfillment_status)`, `orders(customer_id, created_at)`.
- Penjualan: `sales(branch_id, completed_at)`.
- Klinik: `visits(pet_id, started_at)`, `appointments(branch_id, starts_at)`.
- Audit: `audit_logs(entity_type, entity_id)`, `audit_logs(branch_id, occurred_at)`.
- Batch kedaluwarsa: `inventory_lots(product_id, location_id, expiry_date)`.

Indeks lain ditambahkan hanya jika `EXPLAIN ANALYZE` pada data uji menunjukkan kebutuhan.

---

## 19. Layanan Transaksi

### 19.1 Aturan umum service

- Semua operasi tulis bisnis berada di kelas service dan dijalankan melalui `Transactions::run(callable $fn, string $scope)`.
- Setiap service memanggil `AccessControl` sebelum kunci apa pun diambil.
- Service menerima `actor` dari `Auth::user()`, bukan dari parameter input klien.
- Service mengembalikan model atau DTO hasil. Error domain dilempar sebagai `DomainException` dengan kode dari 20.3.
- Timeout transaksi: 5 detik untuk POS, pembayaran, dan penerimaan. 10 detik untuk laporan dan job. Diset dengan `SET LOCAL statement_timeout`.

### 19.2 Helper transaksi dan retry

`Transactions::run` menjalankan `DB::transaction` dengan logika retry:
- Retry hanya untuk `SQLSTATE 40001` (serialization failure) dan `40P01` (deadlock).
- Maksimal 3 percobaan, jeda acak 50–150 ms.
- Error domain tidak pernah di-retry.
- Setelah percobaan habis: `CONFLICT` (409).

Isolasi default transaksi: `READ COMMITTED` dengan `SELECT ... FOR UPDATE` pada baris yang diubah. Tidak memakai `SERIALIZABLE` pada seluruh aplikasi.

### 19.3 Urutan penguncian

Setiap transaksi yang mengubah stok atau kas mengunci baris dalam urutan berikut:

1. Dokumen induk: `sales`, `orders`, `visits`, `hospitalizations`, `service_invoices`, `purchase_orders`, `inventory_transfers`, atau `refunds`.
2. Pembayaran terkait: `payments`, `sale_payments`, `service_payments`.
3. `stock_balances` berdasarkan `(product_id, location_id, lot_id)` menaik.
4. `product_branch_costs` berdasarkan `(product_id, branch_id)` menaik.
5. `care_units` berdasarkan `id` menaik (rawat inap dan penitipan).
6. `invoice_sequences` berdasarkan `(branch_id, doc_type, year)`.
7. `cash_shifts` yang terkait.

Semua pengambilan kunci di service memakai helper yang mengurutkan entri sebelum mengunci.

### 19.4 Idempotensi

Setiap form yang mengubah data bisnis penting membuat `idempotency_key` (UUID v4) saat halaman dimuat dan menyimpannya di field tersembunyi. Livewire menyimpan kunci di state komponen.

Aturan `IdempotencyGuard::run($user, $scope, $key, $payloadHash, $callback)`:

1. Jika tidak ada baris `(user_id, scope, key)`: buat baris `IN_PROGRESS`, jalankan callback, simpan `DONE`, `result_type`, `result_id`.
2. Jika ada dan `DONE` dengan `request_hash` sama: kembalikan hasil tersimpan tanpa menjalankan ulang.
3. Jika ada dan `DONE` dengan `request_hash` berbeda: `IDEMPOTENCY_CONFLICT`.
4. Jika ada dan `IN_PROGRESS`: `CONFLICT` dengan pesan "Permintaan sedang diproses, coba lagi sebentar".

`request_hash` adalah SHA-256 dari payload kanonik (JSON dengan kunci terurut). Pembuatan baris `IN_PROGRESS` dan callback berada dalam transaksi yang sama, sehingga baris `IN_PROGRESS` tidak tertinggal jika callback gagal.

Kunci kedaluwarsa setelah 24 jam dan dibersihkan oleh job harian.

### 19.5 Penomoran dokumen

`DocumentNumberService::next($branchId, $docType, $year)`:
- Mengunci baris `invoice_sequences` dengan `FOR UPDATE` atau membuatnya dengan `INSERT ... ON CONFLICT DO NOTHING` lalu mengunci.
- Menaikkan `last_number` dan memformat `{KODE_CABANG}-{JENIS}-{TAHUN}-{NOMOR6}`.
- Dipanggil di dalam transaksi pembuat dokumen. Jika transaksi gagal, nomor ikut dibatalkan (tidak ada lubang nomor dari transaksi yang gagal).

### 19.6 Daftar operasi kritis

| Operasi | Service | Izin |
|---------|---------|------|
| Buka dan tutup shift | `CashShiftService` | `cash_shift.manage` |
| Selesaikan penjualan POS | `SaleCompletionService` | `pos.sell` |
| Retur POS | `SalesReturnService` | `pos.return.request` |
| Konfirmasi pembayaran online | `OnlinePaymentService` | `payment.confirm` |
| Tolak pembayaran online | `OnlinePaymentService` | `payment.review` |
| Resolusi STOCK_REVIEW | `StockReviewService` | `stock_review.resolve` |
| Maju fulfillment | `FulfillmentService` | `fulfillment.manage` |
| Penyesuaian stok | `InventoryAdjustmentService` | `inventory.adjust` |
| Stok opname | `CountSessionService` | `inventory.count` |
| Transfer (buat, kirim, terima) | `InventoryTransferService` | `inventory.transfer.*` |
| Penerimaan pembelian | `PurchaseReceivingService` | `purchase.manage` |
| Retur supplier | `SupplierReturnService` | `purchase.manage` |
| Tanda tangan rekam medis | `MedicalRecordService` | `medical_record.sign` |
| Amendemen rekam medis | `MedicalRecordService` | `medical_record.amend` |
| Pengeluaran obat | `MedicationDispensingService` | `medication.dispense` |
| Penempatan dan pindah unit | `CareUnitAllocator` | `hospitalization.manage` |
| Publikasi update | `SharedUpdateService` | `hospitalization.publish_update` |
| Check-in dan check-out booking | `ServiceBookingService` | `grooming.manage` |
| Pembayaran invoice layanan | `ServiceBillingService` | `service_invoice.manage` |
| Refund (ajukan, setujui, selesaikan) | `RefundService` | `refund.*` |
| Pengeluaran dan persetujuan | `ExpenseService` | `finance.expense.*` |
| Tutup periode | `FinancialPeriodService` | `finance.period.close` |

### 19.7 Pemeriksaan standar setiap operasi

1. Pengguna aktif dan MFA terpenuhi bila diwajibkan.
2. Izin tersedia (`AccessControl::can`).
3. Cabang objek dalam cakupan pengguna.
4. Objek dalam cakupan pelanggan atau penugasan.
5. Status objek mengizinkan operasi.
6. Periode keuangan untuk tanggal transaksi tidak `CLOSED`.
7. Persetujuan tambahan tersedia bila diperlukan (pemisahan tugas).

Kegagalan langkah 1–4 menghasilkan `NOT_FOUND` untuk objek yang tidak boleh terlihat atau `FORBIDDEN` untuk tindakan yang tidak diizinkan.

---

## 20. Keamanan

### 20.1 Otorisasi di lapisan aplikasi

- `AccessControl` adalah satu-satunya tempat evaluasi izin. Policies dan Filament memanggilnya.
- Filament `Resource` menerapkan `canViewAny`, `canCreate`, `canEdit`, `canDelete` (yang default `false` untuk entitas transaksi) melalui `AccessControl`.
- Panel Filament hanya dapat diakses peran staf (`OWNER`, `DOCTOR`, `ADMIN`) yang aktif. Middleware `EnsureStaffPanelAccess` memeriksa peran dan MFA.
- Rute `/akun` memakai middleware `auth`, `verified`, dan `customer`.
- Setiap rute yang menerima ID objek memanggil policy atau `AccessControl` dengan objek hasil route model binding yang sudah difilter cakupan pelanggan.

### 20.2 Keamanan basis data

- Pengguna basis data aplikasi (`app_user`) hanya memiliki `SELECT`, `INSERT`, `UPDATE`, dan `DELETE` pada tabel yang relevan sesuai daftar append-only. Tidak memiliki hak membuat atau menghapus tabel.
- Akses langsung ke PostgreSQL dari luar hanya dari IP server Forge dan jaringan admin yang terdaftar. Tidak ada port publik.
- Koneksi memakai `sslmode=require`.
- Migration dijalankan oleh pengguna `migrator` yang berbeda dari `app_user`.

Pada Versi 1 tidak memakai Row Level Security PostgreSQL. Isolasi dijamin oleh `AccessControl`, policies, dan scope query. Tes wajib (24.1) memverifikasi isolasi dari setiap jalur akses.

### 20.3 Kode error

| Kode | HTTP | Makna |
|------|------|-------|
| `VALIDATION_ERROR` | 422 | Input tidak valid |
| `UNAUTHENTICATED` | 401 | Sesi tidak ada atau tidak valid |
| `MFA_REQUIRED` | 403 | MFA belum aktif atau sesi belum verifikasi MFA |
| `FORBIDDEN` | 403 | Izin tidak tersedia |
| `NOT_FOUND` | 404 | Tidak ditemukan atau di luar cakupan akses |
| `CONFLICT` | 409 | Konflik status atau data |
| `IDEMPOTENCY_CONFLICT` | 409 | Kunci dipakai untuk payload berbeda |
| `INVALID_STATE_TRANSITION` | 422 | Transisi status tidak diizinkan |
| `INSUFFICIENT_STOCK` | 422 | Stok tidak mencukupi |
| `INSUFFICIENT_CAPACITY` | 422 | Kapasitas penuh |
| `PERIOD_CLOSED` | 422 | Tanggal berada di periode keuangan tertutup |
| `RATE_LIMITED` | 429 | Batas laju terlampaui |
| `INTERNAL_ERROR` | 500 | Kesalahan internal tanpa detail |

Pada antarmuka, kode error ditampilkan dengan pesan Indonesia. Pesan internal, SQL, dan stack trace tidak pernah ditampilkan. Setiap halaman error menampilkan `request_id` untuk pelaporan.

### 20.4 Request ID

- Middleware `AssignRequestId` membuat ID (`req_` + 26 karakter acak) pada setiap request dan menyimpannya di konteks log dan audit.
- Header `X-Request-Id` dikirim balik ke browser.

### 20.5 Rekam medis

- Penandatanganan hanya melalui `MedicalRecordService::sign`.
- Trigger basis data menolak perubahan kolom klinis setelah `SIGNED` (18.4).
- Amendemen menyimpan snapshot lengkap.
- Tes memverifikasi bahwa dokter yang tidak ditugaskan, admin, dan pelanggan tidak dapat membaca `internal_note` atau mengubah rekam medis.

### 20.6 Berkas

| Disk | Lokasi | Akses | Ukuran maks | Tipe | Retensi |
|------|--------|-------|-------------|------|---------|
| `private` | `storage/app/private/payment-proofs` | Controller dengan pemeriksaan izin | 5 MB | JPEG, PNG, PDF | 10 tahun |
| `private` | `storage/app/private/medical-documents` | Controller dengan pemeriksaan izin | 20 MB | PDF, JPEG, PNG | 10 tahun sejak kunjungan terakhir |
| `private` | `storage/app/private/pet-photos` | Controller dengan pemeriksaan izin | 5 MB | JPEG, PNG, WEBP | Selama hewan terdaftar |
| `public` | `storage/app/public/product-images` | Publik baca (symlink) | 2 MB | JPEG, PNG, WEBP | Selama produk aktif |

- Setiap berkas disimpan dengan nama UUID. Nama asli disimpan di kolom metadata.
- Validasi tipe memakai aturan `mimes:` dan `image` Laravel, yang memeriksa konten file (fileinfo), bukan hanya ekstensi.
- Unduhan berkas privat melalui route `GET /berkas/{type}/{id}` yang memeriksa izin dan mengirim berkas dengan `Content-Disposition: inline` dan `Cache-Control: private, no-store`.
- Berkas tidak dipindai antivirus pada Versi 1. Pembatasan tipe dan ukuran menjadi kontrol utama.

### 20.7 Pembatasan laju

Menggunakan `RateLimiter` Laravel:
- Login: 5 kegagalan per email per 15 menit (lockout 15 menit, 5.5).
- Registrasi dan reset kata sandi: 10 request per IP per menit.
- Form tulis pengguna: 60 request per pengguna per menit.
- Konfirmasi dan pembayaran: 20 request per pengguna per menit.
- Pelanggaran menghasilkan `RATE_LIMITED` dengan header `Retry-After`.

### 20.8 Keamanan aplikasi

- Header keamanan dikirim melalui middleware `SecurityHeaders`: `Content-Security-Policy` yang membatasi sumber ke `self` dan domain asset Vite, `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, `Permissions-Policy` yang membatasi kamera hanya pada halaman unggah bukti.
- Cookie sesi: `HttpOnly`, `Secure`, `SameSite=Lax`.
- CSRF melalui token bawaan Laravel untuk seluruh form dan Livewire.
- Blade `{{ }}` (escape otomatis). `{!! !!}` tidak dipakai untuk data pengguna.
- Mass assignment dilindungi dengan `$fillable` eksplisit. Model transaksi memakai `$guarded = ['*']`.
- Dependensi diperiksa di CI dengan `composer audit` dan `npm audit`. Kerentanan kritis atau tinggi memblokir merge.
- Data pribadi dan klinis dihapus dari log, URL, pesan error, dan Sentry melalui `beforeSend` dan `Log` processor.

### 20.9 Pemisahan rahasia

- Rahasia hanya di environment Forge, tidak di repository.
- File `.env` tidak di-commit. `.env.example` berisi nama variabel tanpa nilai.
- Tidak ada variabel yang berawalan `VITE_` berisi rahasia. Vite hanya membaca konfigurasi publik.

---

## 21. Antarmuka Pengguna

### 21.1 Navigasi panel staf (Filament)

| Peran | Menu |
|-------|------|
| OWNER | Dashboard, POS, Inventori, Pembelian, Pesanan online, Klinik, Rawat inap, Grooming & penitipan, Tagihan layanan, Keuangan, Laporan, Pelanggan & hewan, Pengaturan |
| DOCTOR | Dashboard dokter, Jadwal saya, Pasien, Kunjungan, Rekam medis, Resep, Rawat inap, Tugas klinis, Laporan saya |
| ADMIN | Dashboard admin, POS, Pesanan online, Pembayaran, Pelanggan & hewan, Janji temu, Grooming & penitipan, Stok (baca), Pembelian, Tagihan layanan, Pengeluaran |

Menu yang tidak diizinkan tidak ditampilkan. Akses langsung ke URL tanpa izin menghasilkan `FORBIDDEN` atau `NOT_FOUND` sesuai aturan 5.2.

### 21.2 Navigasi portal pelanggan (Blade)

Beranda, Katalog, Keranjang, Pesanan saya, Hewan saya, Janji temu, Rawat inap, Tagihan, Profil.

### 21.3 Halaman wajib

| Halaman | Elemen wajib |
|---------|--------------|
| Login staf | Email, kata sandi, lupa kata sandi, MFA bila diperlukan |
| Dashboard pemilik | Penjualan hari ini, kas per metode dan shift terbuka, stok di bawah minimum, batch kedaluwarsa 90 hari, janji hari ini, pesanan `STOCK_REVIEW`, piutang, selisih rekonsiliasi |
| Dashboard dokter | Jadwal hari ini, pasien, rekam medis `DRAFT`, rawat inap aktif, update yang belum dipublikasikan |
| Dashboard admin | Shift aktif, pembayaran `SUBMITTED`, pesanan siap diambil, booking hari ini, tagihan jatuh tempo |
| POS | Pencarian (nama, SKU, barcode), keranjang, pilihan pelanggan opsional, diskon, total, pembayaran terpisah per metode, cetak invoice |
| Inventori | Filter cabang dan lokasi, saldo, batch, mutasi, penyesuaian, transfer, stok opname |
| Detail hewan | Identitas, wali (utama dan sekunder), riwayat kunjungan, dokumen, rawat inap |
| Rekam medis | Form klinis, resep, status tanda tangan, amendemen, riwayat versi |
| Rawat inap | Denah unit, penempatan, observasi, tugas, update publik, proses pulang |
| Pesanan online | Item, pembayaran dan bukti, status, review stok, fulfillment, refund |
| Tagihan layanan | Item, pembayaran parsial, saldo, kredit |
| Keuangan | Kas per shift dan metode, pengeluaran, refund, periode |
| Laporan | Filter cabang, rentang tanggal, ekspor CSV dan PDF |
| Pengaturan | Pengguna, undangan, grant izin, cabang, rekening pembayaran, metode pembayaran, kode alasan, template, job, audit |
| Katalog publik | Daftar produk, pencarian, kategori, detail produk, pilih cabang |
| Keranjang dan checkout | Item, jumlah, cabang, pengambilan atau pengiriman, ringkasan, instruksi pembayaran |
| Portal hewan | Identitas hewan, wali, riwayat, dokumen |
| Portal pesanan | Status, item, instruksi pembayaran, unggah bukti, pembatalan |
| Portal tagihan | Invoice, pembayaran, saldo (wali utama) |

### 21.4 Perilaku UI

- Desain responsif. Portal Blade mobile-first. Panel Filament responsif dengan tabel yang berubah menjadi tampilan kartu di layar kecil.
- POS dirancang untuk tablet dengan lebar minimal 768 px. Pada ponsel, POS menampilkan mode terbatas: pencarian dan keranjang saja, dengan pembayaran tetap tersedia.
- Setiap halaman memiliki status loading, kosong, sukses, error, dan tidak berizin.
- Operasi berisiko (void, retur, refund, penyesuaian, pembatalan pesanan, penutupan shift, penutupan periode, transfer, stok opname) memakai dialog konfirmasi yang menyebutkan dampaknya.
- Tombol submit dinonaktifkan selama request berjalan (`wire:loading.attr="disabled"`). Idempotensi tetap diterapkan di server.
- Status ditampilkan dengan label teks dan warna, tidak hanya warna.
- Daftar besar memakai pagination dan pencarian server-side.
- Navigasi keyboard penuh, label form eksplisit, fokus terlihat. Target WCAG 2.2 AA.
- Format: Rupiah `Rp 1.250.000`; tanggal `DD/MM/YYYY`; waktu 24 jam dalam zona cabang.
- Input pemindai barcode berbasis keyboard wedge: kolom pencarian POS tetap memiliki fokus. Enter setelah barcode menambah item.
- Notifikasi browser dan pop-up tidak menampilkan data medis.
- Form dengan beberapa langkah menyimpan draft di basis data, bukan di browser.
- Pelanggan yang belum memverifikasi email melihat halaman pemberitahuan verifikasi, dan tidak dapat membuat pesanan atau janji.

### 21.5 Bahasa dan zona waktu

- Locale aplikasi `id`. Semua label, pesan validasi, dan pesan error dalam bahasa Indonesia (`lang/id`).
- Filament diset ke locale `id`.
- Waktu disimpan UTC dan ditampilkan dengan zona cabang. Pengguna dengan cakupan beberapa cabang melihat zona cabang yang sedang dipilih.

---

## 22. Dokumen Cetak dan Ekspor

- Invoice POS, pesanan online, dan invoice layanan dicetak melalui halaman Blade cetak (`@media print`) dan dapat diunduh sebagai PDF (dompdf).
- Invoice memuat: nomor (format B9), tanggal, cabang, alamat cabang, pelanggan (jika ada), item, diskon, total, metode pembayaran, saldo (untuk layanan), dan teks pajak (B12).
- Invoice tidak memuat diagnosis atau isi klinis.
- Template invoice dan kop laporan diatur di `branch_settings` (logo, alamat, nomor telepon). Logo diunggah ke disk `public`.
- Bukti refund dan kwitansi kas dicetak dengan template yang sama.
- Ekspor laporan CSV dan PDF mengikuti 15.5. Setiap ekspor mencantumkan tanggal cetak, nama pencetak, cabang, dan rentang filter.

---

## 23. Operasional

### 23.1 Cadangan dan pemulihan

- Basis data: PITR dari penyedia PostgreSQL terkelola dengan retensi minimal 7 hari. Ini memenuhi RPO 15 menit (B27).
- Dump logis harian (`spatie/laravel-backup`) basis data dan folder `storage/app/private` dikirim ke penyimpanan S3-compatible di luar server dengan enkripsi. Retensi 35 hari.
- Cadangan kode bersumber dari GitHub.
- Uji pemulihan dilakukan setiap bulan ke staging terpisah. Hasil dicatat di `docs/runbook.md`.
- Penanggung jawab pemulihan: developer lead. Penyetuju: pemilik.
- Target RTO 4 jam (B27).

### 23.2 Pemantauan

Dipantau melalui Sentry dan `job_runs`:
- Error aplikasi dan rasio request gagal per route.
- Kegagalan login yang tidak biasa (lonjakan per IP atau per email).
- Query lambat di atas 500 ms untuk POS dan pembayaran (Laravel `DB::listen` dengan ambang batas, dikirim ke Sentry).
- `IDEMPOTENCY_CONFLICT`, `CONFLICT`, dan deadlock.
- Kegagalan upload berkas.
- Kegagalan job terjadwal.
- Antrean yang tertunda dan job gagal di `failed_jobs`.
- Kapasitas basis data dan disk (dipantau melalui panel penyedia dan Forge).
- Endpoint `/health`.
- Selisih rekonsiliasi stok harian.

Alert dikirim ke email developer on-call dan pemilik untuk kategori kritis.

### 23.3 Insiden

1. Catat waktu deteksi, sumber, dan dampak di tiket. Tiket tidak berisi data pribadi.
2. Hentikan operasi yang memperburuk inkonsistensi: nonaktifkan POS cabang dengan `branches.pos_enabled = false`, dan nonaktifkan penjualan online secara massal dengan mengubah `product_branch_settings.is_online_sellable`. Tindakan ini dilakukan melalui Filament oleh pemilik.
3. Simpan `request_id` dan log terkait.
4. Rekonsiliasi stok, kas, dan pembayaran dengan query baca (`php artisan reconcile:stock`, `reconcile:cash`).
5. Perbaikan data dilakukan melalui service koreksi berotorisasi, bukan `UPDATE` langsung. Jika perbaikan darurat di basis data tidak terhindarkan, pemilik menyetujui secara tertulis dan langkah dicatat di audit manual dengan `SET LOCAL app.allow_stock_write`.
6. Pulihkan layanan dan aktifkan kembali POS.
7. Dokumentasikan penyebab, dampak, dan tindakan pencegahan dalam 5 hari kerja.

### 23.4 Retensi dan anonimisasi

- Data bisnis dan klinis mengikuti B28.
- Log aplikasi disimpan 30 hari (`LOG_DAILY_DAYS=30`).
- Permintaan anonimisasi pelanggan: nama, telepon, email, dan alamat diganti nilai tanda (misalnya `ANONIM-{id}`). Baris transaksi dan klinis tetap ada. Akun pengguna dinonaktifkan dan sesi dicabut. Permintaan dicatat di audit.

### 23.5 Job terjadwal

Didefinisikan di `routes/console.php`. Setiap job mencatat hasil ke `job_runs`.

| Job | Jadwal | Fungsi |
|-----|--------|--------|
| `void_stale_drafts` | Setiap jam | Draft POS lebih dari 24 jam menjadi `VOIDED` |
| `expire_invitations` | Harian | Menandai undangan kedaluwarsa |
| `expire_idempotency_keys` | Harian | Menghapus kunci lebih dari 24 jam |
| `stock_reconciliation` | Harian | Membandingkan `on_hand` dengan ledger, membuat notifikasi selisih |
| `expiry_alerts` | Harian | Notifikasi batch kedaluwarsa dalam 90 hari |
| `min_stock_alerts` | Harian | Notifikasi stok di bawah minimum |
| `receivable_reminders` | Harian | Email pengingat invoice jatuh tempo |
| `audit_export` | Harian | Ekspor audit ke penyimpanan S3-compatible terpisah |
| `backup_run` | Harian | Dump basis data dan berkas (`backup:run`) |
| `backup_clean` | Harian | Membersihkan cadangan lama (`backup:clean`) |
| `rate_limit_cleanup` | Harian | Membersihkan penghitung kedaluwarsa |
| `prune_sessions` | Harian | Menghapus sesi kedaluwarsa |

Job yang gagal dua kali berturut-turut (dibaca dari `job_runs`) mengirim email alert.

---

## 24. Pengujian dan Kriteria Penerimaan

### 24.1 Jenis pengujian

- **Unit (Pest):** perhitungan `Money`, diskon, pembulatan, biaya rata-rata tertimbang, FEFO, transisi status, validasi.
- **Fitur dan basis data (Pest dengan PostgreSQL nyata):** logika service, constraint, trigger immutability, `AccessControl` untuk setiap peran dan cabang, idempotensi, dan konkurensi dengan dua proses PHP (`Process` atau `pcntl`) yang menjalankan operasi bersamaan pada data yang sama.
- **Isolasi (Pest):** setiap route dan policy diuji dengan akun nyata dari setiap peran dan cabang, termasuk ID milik pelanggan lain. Hasil yang diharapkan `NOT_FOUND` atau `FORBIDDEN`.
- **E2E (Laravel Dusk):** alur POS, pesanan online, rekam medis, rawat inap, dan portal, pada viewport desktop (1280 px), tablet (768 px), dan ponsel (390 px).
- **Keamanan:** pemeriksaan header, uji CSRF, uji unggah berkas dengan tipe palsu, dan uji lockout login.

Cakupan minimum: 85% baris untuk `app/Services` dan `app/Support`, dan 100% untuk setiap method service yang mengubah stok, kas, atau status pembayaran.

Analisis statis: Larastan level 8 untuk `app/Services`, `app/Support`, dan `app/Domain` tanpa baseline baru.

### 24.2 Kriteria penerimaan

| Kode | Kriteria | Lulus bila |
|------|----------|-----------|
| AC-01 | RBAC | Setiap peran hanya dapat menjalankan operasi sesuai tabel 5.1 |
| AC-02 | Isolasi cabang | Pengguna cabang A mendapat `NOT_FOUND` untuk objek cabang B |
| AC-03 | Isolasi pelanggan | Pelanggan A tidak dapat membaca hewan, pesanan, invoice, atau rekam medis pelanggan B dengan mengubah ID |
| AC-04 | POS | Total dan stok benar; kunci idempotensi sama tidak membuat penjualan atau mutasi ganda |
| AC-05 | Konflik stok | Dua transaksi bersamaan atas stok 5 unit (permintaan 4 dan 3) tidak membuat stok negatif dan tidak keduanya berhasil |
| AC-06 | Bukti pembayaran | Unggah bukti tidak mengubah `payment_status` menjadi `CONFIRMED` |
| AC-07 | Alokasi online | Reservasi hanya dibuat saat `confirm` dengan pemeriksaan ulang stok |
| AC-08 | STOCK_REVIEW | Pesanan `STOCK_REVIEW` tidak dapat masuk `PROCESSING` tanpa `resolve` |
| AC-09 | Rekam medis | Rekam medis `SIGNED` tidak dapat diubah langsung, dihapus, atau ditandatangani ulang |
| AC-10 | Kapasitas rawat | Penempatan tidak melampaui kapasitas unit pada rentang waktu bertabrakan |
| AC-11 | Update pelanggan | Pelanggan hanya melihat `shared_updates` yang dipublikasikan untuk hewannya |
| AC-12 | Keuangan | Kas, pengeluaran, piutang, dan HPP dihitung dari sumber terpisah dan tidak tercampur |
| AC-13 | Audit | Perubahan kritis memiliki pelaku, waktu, objek, dan alasan bila diwajibkan |
| AC-14 | Berkas | Berkas privat tidak dapat diunduh tanpa pemeriksaan izin |
| AC-15 | Responsif | Alur POS, pesanan, dan janji dapat diselesaikan di ponsel, tablet, dan desktop |
| AC-16 | Pemulihan | Restore ke staging menghasilkan jumlah baris dan fungsi kritis yang sama |
| AC-17 | Deployment | Tes, format, analisis statis, build, dan migration berjalan melalui pipeline tercatat |
| AC-18 | Konsistensi stok | Rekonsiliasi `on_hand` dengan ledger menghasilkan selisih nol |
| AC-19 | Kas | Saldo tunai shift sama dengan `expected_cash` pada penutupan |
| AC-20 | Batch | Penjualan, pengeluaran klinik, dan transfer tidak mengambil batch kedaluwarsa |

### 24.3 Skenario wajib

1. **Konflik stok:** stok 5; POS menjual 4 dan pesanan online mengalokasikan 3 bersamaan. Total yang berhasil tidak melebihi 5.
2. **Konfirmasi ganda:** `confirm` dipanggil dua kali dengan kunci sama. Hasil: satu pembayaran `CONFIRMED`, satu set reservasi, satu `cash_movement`.
3. **Payload berbeda, kunci sama:** `IDEMPOTENCY_CONFLICT`.
4. **Admin menandatangani rekam medis:** `FORBIDDEN`.
5. **Dokter tidak ditugaskan mengubah rekam medis:** `NOT_FOUND` atau `FORBIDDEN` sesuai cakupan.
6. **Pelanggan mengganti ID hewan atau pesanan:** `NOT_FOUND`.
7. **Pembatalan pesanan `PROCESSING` yang sudah dibayar tanpa refund:** `INVALID_STATE_TRANSITION`.
8. **Pemohon refund menyetujui refund sendiri:** `FORBIDDEN` (database juga menolak).
9. **Pemindahan unit ke unit penuh:** `INSUFFICIENT_CAPACITY`.
10. **Pembayaran tunai POS dengan kembalian:** `cash_movements` `CASH_IN_SALE` sebesar porsi dan `CASH_OUT_CHANGE` sebesar kembalian; saldo shift sesuai.
11. **Diskon 10,01%:** memerlukan verifikasi pemilik; tanpa verifikasi, `FORBIDDEN`.
12. **Pengeluaran Rp 1.000.001 oleh pemohon:** status `PENDING_APPROVAL`, tidak membuat `cash_movements`.
13. **Transaksi di periode tertutup:** `PERIOD_CLOSED`.
14. **Backup dipulihkan ke staging:** jumlah baris, akses, berkas, dan fungsi kritis diverifikasi; durasi dicatat.

### 24.4 Gerbang rilis production

Rilis production hanya dilakukan jika:
- AC-01 sampai AC-20 lulus di staging.
- Tidak ada kerentanan kritis atau tinggi yang terbuka (`composer audit`, `npm audit`, dan tinjauan manual).
- Tes isolasi mencakup setiap peran dan cabang dengan akun nyata.
- Migration berjalan di staging tanpa error dan dinilai mundur kompatibel dengan versi sebelumnya.
- Pemulihan cadangan diuji dalam 30 hari terakhir.
- Dua akun pemilik aktif sudah ada (B25).
- Pemilik menyetujui UAT secara tertulis untuk POS, klinik, rawat inap, pesanan online, dan laporan.

---

## 25. Struktur Folder

Struktur mengikuti konvensi Laravel dengan folder tambahan untuk domain.

```text
petshop-klinik/
├── app/
│   ├── Domain/                 # enum status, konstanta izin, nilai kode alasan
│   ├── Services/               # seluruh operasi tulis bisnis (satu class per modul)
│   │   ├── Pos/
│   │   ├── Inventory/
│   │   ├── Purchasing/
│   │   ├── Online/
│   │   ├── Clinic/
│   │   ├── Hospitalization/
│   │   ├── Grooming/
│   │   ├── Billing/
│   │   ├── Finance/
│   │   └── Shared/             # Transactions, IdempotencyGuard, DocumentNumberService, AuditLogger
│   ├── Support/
│   │   ├── Money.php           # wrapper brick/math
│   │   ├── AccessControl.php
│   │   └── Errors/             # DomainException dan kode error
│   ├── Models/
│   ├── Policies/
│   ├── Filament/
│   │   ├── Pages/              # termasuk PosTerminal
│   │   ├── Resources/
│   │   └── Widgets/
│   ├── Livewire/               # komponen portal dan POS yang interaktif
│   ├── Http/
│   │   ├── Controllers/        # portal, unduhan berkas, undangan, health
│   │   └── Middleware/         # EnsureStaffPanelAccess, EnforceSessionIdle, AssignRequestId, SecurityHeaders
│   ├── Notifications/
│   ├── Console/Commands/       # app:create-owner, reconcile:stock, reconcile:cash, customers:anonymize, import:initial
│   └── Jobs/
├── database/
│   ├── migrations/             # expand and contract, tidak diedit setelah merge
│   ├── seeders/                # permissions, role_permissions, payment_methods, data demo sintetis
│   └── factories/
├── resources/
│   ├── views/
│   │   ├── portal/             # Blade portal pelanggan
│   │   ├── print/              # invoice, kwitansi, laporan PDF
│   │   └── auth/
│   ├── css/app.css             # Tailwind CSS 4
│   ├── js/app.js
│   └── lang/id/
├── routes/
│   ├── web.php
│   └── console.php
├── tests/
│   ├── Unit/
│   ├── Feature/
│   ├── Isolation/              # tes IDOR dan isolasi cabang
│   ├── Concurrency/            # tes dua proses
│   └── Browser/                # Laravel Dusk
├── docs/
│   ├── permissions.md
│   ├── deployment.md
│   └── runbook.md
├── .github/workflows/ci.yml
├── .env.example
├── composer.json
├── package.json
├── vite.config.js
└── README.md
```

Aturan struktur:
- Komponen Blade, Livewire, dan Filament tidak memanggil model untuk operasi tulis bisnis. Mereka memanggil service.
- Service tidak memanggil Filament, Livewire, atau Blade.
- `app/Services` tidak mengakses `request()` secara langsung. Input diterima sebagai parameter bertipe.
- `any` tidak digunakan. Setiap parameter dan return type dideklarasikan.

---

## 26. Data Awal

Sebelum operasi live, data berikut wajib tersedia dan diverifikasi pemilik:

- Profil bisnis, daftar cabang, kode cabang, zona waktu, alamat, dan logo.
- Dua akun pemilik aktif (B25). Akun pertama dibuat dengan `php artisan app:create-owner`. Akun kedua dibuat melalui undangan.
- Profil dokter dengan STR untuk setiap dokter dan untuk pemilik yang juga dokter.
- Pengguna staf dengan undangan, peran, dan cabang. Admin memiliki minimal satu cabang.
- Master produk: SKU, barcode, kategori, satuan, `track_lot`, `is_prescription`.
- Harga per cabang dan status jual POS dan online.
- Stok awal per lokasi dengan batch dan kedaluwarsa untuk produk `track_lot`, serta biaya rata-rata awal per cabang.
- Supplier.
- Pelanggan dan hewan yang datanya boleh dimigrasikan, lengkap dengan wali utama.
- Katalog layanan (klinik, grooming, penitipan) dengan harga per cabang.
- Unit rawat dan kapasitasnya.
- Metode pembayaran aktif per cabang dan rekening atau instruksi manual.
- Biaya pengiriman per cabang.
- Kode alasan untuk penyesuaian, refund, pengeluaran, dan selisih kas (daftar tetap di `app/Domain`).
- Saldo kas awal per cabang sebagai `cash_movements` `SHIFT_OPENING` pada shift pertama, piutang awal sebagai invoice layanan, dan tagihan supplier yang masih berlaku.
- Template invoice dan laporan.

Migrasi data awal dilakukan dengan perintah impor (`php artisan import:initial`) di staging, dengan rekonsiliasi jumlah dan nilai per cabang yang ditandatangani pemilik sebelum diterapkan di production.

Data demo sintetis tersedia melalui seeder untuk staging dan lokal. Seeder demo tidak dijalankan di production (dicek dengan `app()->environment('production')`).

---

## 27. Definition of Done

Pekerjaan selesai bila seluruh kondisi berikut terpenuhi:

1. Seluruh modul dalam Bagian 4 tersedia dan memenuhi aturan dokumen ini.
2. Migration, constraint, trigger, seeder izin, dan service tersimpan di repository.
3. Seluruh kriteria penerimaan di Bagian 24.2 lulus di staging.
4. Hak akses setiap peran diverifikasi di server dengan akun nyata.
5. Stok, pembayaran, dan kas tidak menunjukkan inkonsistensi pada tes konkurensi.
6. Rekam medis yang sudah ditandatangani tidak dapat diubah langsung.
7. Laporan dapat direkonsiliasi ke transaksi sumber.
8. Deployment, cadangan, pemulihan, pemantauan, dan prosedur insiden terdokumentasi di `docs/`.
9. Tidak ada rahasia di repository, log, atau browser.
10. Pemilik menyetujui UAT untuk POS, klinik, rawat inap, pesanan online, dan laporan, dan menandatangani berita acara serah terima.

---

## 28. Status Dokumen

Dokumen ini adalah spesifikasi Versi 1.0. Dokumen ini bukan bukti bahwa aplikasi, database, atau deployment sudah dibuat. Implementasi harus dibangun dan diverifikasi terhadap dokumen ini.

Setiap perubahan keputusan di Bagian 1 dilakukan dengan merevisi dokumen ini, menaikkan nomor versi, dan mencatat alasan perubahan di bagian riwayat revisi.
