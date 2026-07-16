# Business Workflows

## Purpose
Describe the end-to-end processes users and the organization follow, independent of UI or technical implementation.

## Responsibilities
- Owned by: **Business Analyst**.
- Consumed by: **Frontend Developer** (to sequence UI steps) and **QA/Testing Engineer** (to build realistic test scenarios).

## Inputs
- Stakeholder interviews and observed current processes (including manual/offline steps being digitized).

## Outputs
- Workflow diagrams/step lists referenced by `/frontend/notes.md` and `/testing/test-plan.md`.

## Rules
- Workflows are described as actor → action → outcome steps, including exception/error paths, not just the happy path.
- Any workflow step that maps to a business rule must reference its ID from `/business/business-rules.md`.
- Changes to a live workflow must note what the previous behavior was, so downstream roles understand what changed.

## Checklist
- [x] Workflow includes actors, trigger, steps, and outcome.
- [x] Exception/edge-case paths are documented, not just the happy path.
- [x] Referenced business rules are linked by ID.

---

## Siklus Keuangan Bulanan (Ringkasan)

Gambaran urutan alur kerja dalam satu periode bulan, sebagai acuan urutan WF di bawah:

```
AWAL BULAN
├── Unit input Jurnal Umum (WF-001) — ongoing
├── Unit input Jurnal Transaksi via MEMO (WF-002) — ongoing
└── Unit input Jurnal Pajak Terima tiap ada pemotongan (WF-003) — ongoing

AKHIR BULAN
├── Unit buat Jurnal Pajak Setor per jenis pajak (WF-004)
├── Operator Pajak bentuk Batch Pelaporan (WF-005)
├── Unit generate & upload LPJ (WF-008)
└── Admin tutup periode (WF-006)

SEPANJANG WAKTU (non-periodik)
├── Admin kelola user, COA, konfigurasi (WF-010, WF-011)
├── Admin impersonasi user untuk support (WF-009)
├── Admin sinkronisasi MEMO/PAGU (WF-012)
└── Admin/Operator Pusat generate laporan lain (WF-007)
```

---

## WF-001: Jurnal Umum

**Aktor:** Operator / BPP
**Trigger:** Kebutuhan pencatatan transaksi keuangan unit yang tidak menggunakan MEMO
**Business Rules:** RULE-001, RULE-002, RULE-003, RULE-004, RULE-005, RULE-006, RULE-008

### Happy Path

| # | Aktor | Aksi | Outcome |
|---|-------|------|---------|
| 1 | Operator/BPP | Pilih menu Jurnal → Jurnal Umum, pilih sub-jenis (Uang Muka / Pengembalian Uang Muka / Penerimaan UP / Lain-lain) | Form jurnal terbuka dengan akun yang dibatasi sesuai sub-jenis (RULE-003) |
| 2 | Operator/BPP | Isi tanggal, nomor bukti (format `YY.XXXX`), keterangan, dan baris akun Debet/Kredit | Nilai total Debet dan Kredit terhitung real-time |
| 3 | Sistem | Validasi balance Debet = Kredit (RULE-001, RULE-002) | Jika balance, tombol Simpan aktif |
| 4 | Operator/BPP | Upload file bukti PDF (opsional — lihat OI-006) maks 5 MB (RULE-005) | File terlampir pada jurnal |
| 5 | Operator/BPP | Klik Simpan / Rekam | Sistem generate Nomor Jurnal otomatis `[kode_unit].[YY].[XXXXX]` (RULE-004) |
| 6 | Sistem | Simpan jurnal, status: RECORDED | Jurnal tersimpan dan muncul di daftar jurnal unit |

### Exception Paths

| Kondisi | Penanganan |
|---------|------------|
| Debet ≠ Kredit saat Simpan | Sistem menolak, tampilkan selisih, jurnal tidak tersimpan (RULE-001) |
| File bukan PDF atau > 5 MB | Upload ditolak sebelum proses (RULE-005) |
| Periode bulan sudah ditutup | Form jurnal nonaktif / tombol Simpan diblokir untuk periode tersebut (RULE-008) |
| Akun yang dipilih di luar mapping jenis jurnal | Backend menolak meskipun data dikirim langsung (RULE-003) |

---

## WF-002: Jurnal Transaksi (Berbasis MEMO)

**Aktor:** Operator / BPP
**Trigger:** Unit akan melakukan belanja yang sudah memiliki MEMO yang disetujui
**Business Rules:** RULE-001, RULE-002, RULE-003, RULE-004, RULE-005, RULE-006, RULE-008, RULE-016

### Happy Path

| # | Aktor | Aksi | Outcome |
|---|-------|------|---------|
| 1 | Operator/BPP | Pilih menu Jurnal → Jurnal Transaksi | Form terbuka, kolom MEMO tersedia |
| 2 | Operator/BPP | Cari dan pilih MEMO (data sudah tersinkronisasi dari sistem perencanaan) | Sistem auto-isi akun belanja (Debet) dan nominal dari MEMO (RULE-016) |
| 3 | Operator/BPP | Pilih akun lawan (Kredit): Kas Tangan / Kas Bank / Hutang Belanja | Sisi Kredit terisi |
| 4 | Operator/BPP | Lengkapi tanggal, nomor bukti, keterangan | Form lengkap |
| 5 | Sistem | Validasi balance (RULE-001) dan status MEMO (RULE-016) | Jika lolos, lanjut simpan |
| 6 | Operator/BPP | Upload bukti PDF (opsional, RULE-005), klik Simpan | Sistem generate Nomor Jurnal (RULE-004), MEMO ditandai SUDAH DIGUNAKAN |
| 7 | Sistem | Simpan jurnal, status: RECORDED | Jurnal tersimpan, MEMO tidak dapat dipilih ulang (RULE-016) |

### Exception Paths

| Kondisi | Penanganan |
|---------|------------|
| MEMO yang dipilih berstatus SUDAH DIGUNAKAN | Backend menolak; sistem menampilkan pesan bahwa MEMO sudah terpakai (RULE-016) |
| MEMO belum tersinkronisasi / tidak ditemukan | Tampilkan pesan sinkronisasi belum update; Admin dapat trigger sinkronisasi manual (WF-012) |
| Debet ≠ Kredit | Sistem menolak, tampilkan selisih (RULE-001) |
| Periode ditutup | Simpan diblokir (RULE-008) |

---

## WF-003: Jurnal Pajak Terima

**Aktor:** Operator / BPP
**Trigger:** Unit melakukan pemotongan pajak pada sebuah transaksi
**Business Rules:** RULE-001, RULE-002, RULE-003, RULE-004, RULE-006, RULE-008, RULE-012

### Happy Path

| # | Aktor | Aksi | Outcome |
|---|-------|------|---------|
| 1 | Operator/BPP | Pilih menu Jurnal → Pajak Terima, pilih jenis pajak (PPh 21/22/23/26/PPN/PPh4(2)) | Form terbuka; Debet otomatis = akun Hutang Pajak sesuai jenis, Kredit = akun Kas (RULE-003) |
| 2 | Operator/BPP | Isi nilai nominal | Debet dan Kredit terisi |
| 3 | Operator/BPP | Lengkapi data wajib pajak: Nama WP, NPWP, Nilai Bruto, dan field tambahan per jenis (RULE-012) | Data detail tersimpan bersama jurnal |
| 4 | Operator/BPP | Isi tanggal, nomor bukti, keterangan; upload bukti PDF (opsional) | Form lengkap |
| 5 | Sistem | Validasi balance (RULE-001) dan kelengkapan field wajib (RULE-012) | Jika lolos, simpan |
| 6 | Sistem | Simpan jurnal Pajak Terima, status: BELUM DISETOR | Jurnal siap untuk diikutkan ke Jurnal Setor di akhir bulan |

### Exception Paths

| Kondisi | Penanganan |
|---------|------------|
| Field wajib pajak tidak lengkap | Sistem menolak; highlight field yang kosong (RULE-012) |
| Debet ≠ Kredit | Sistem menolak (RULE-001) |
| Periode ditutup | Simpan diblokir (RULE-008) |

---

## WF-004: Jurnal Pajak Setor (Unit)

**Aktor:** Operator / BPP
**Trigger:** Akhir bulan — unit akan menyetorkan pajak yang sudah dipotong ke DJP
**Business Rules:** RULE-001, RULE-002, RULE-004, RULE-006, RULE-008, RULE-013

### Happy Path

| # | Aktor | Aksi | Outcome |
|---|-------|------|---------|
| 1 | Operator/BPP | Pilih menu Jurnal → Pajak Setor, pilih jenis pajak dan periode bulan | Sistem menampilkan daftar Jurnal Pajak Terima berstatus BELUM DISETOR untuk jenis dan bulan tersebut (RULE-013) |
| 2 | Operator/BPP | Centang jurnal-jurnal terima yang akan dimasukkan ke setoran ini | Sistem auto-hitung total nominal dari yang dipilih |
| 3 | Operator/BPP | Isi tanggal setor, nomor bukti, keterangan | Form lengkap |
| 4 | Sistem | Validasi balance dan status setiap jurnal terima yang dipilih (RULE-013) | Jika lolos, simpan |
| 5 | Sistem | Simpan Jurnal Setor; ubah status jurnal terima yang terpilih menjadi SUDAH DISETOR | Jurnal Terima terpilih tidak muncul lagi di daftar pilihan (RULE-013) |

### Exception Paths

| Kondisi | Penanganan |
|---------|------------|
| Jurnal Terima yang dipilih sudah berstatus SUDAH DISETOR | Backend menolak entri tersebut; sistem menampilkan pesan (RULE-013) |
| Tidak ada jurnal terima yang memenuhi kriteria | Sistem menampilkan pesan kosong; tidak ada jurnal setor yang dapat dibuat |
| Periode ditutup | Simpan diblokir (RULE-008) |

---

## WF-005: Konsolidasi Batch Pajak (Operator Pajak)

**Aktor:** Operator Pajak
**Trigger:** Semua/sebagian unit telah membuat Jurnal Pajak Setor untuk bulan berjalan
**Business Rules:** RULE-014, RULE-015

### Happy Path

| # | Aktor | Aksi | Outcome |
|---|-------|------|---------|
| 1 | Operator Pajak | Pilih menu Pajak → Batch Pelaporan, pilih jenis pajak dan periode bulan | Sistem menampilkan semua Jurnal Setor yang belum masuk batch, per jenis pajak dan bulan (RULE-014) |
| 2 | Operator Pajak | Review daftar dari semua unit; pilih jurnal setor yang akan dibatch | Sistem hitung total batch |
| 3 | Operator Pajak | Simpan batch → status: DRAFT | Batch tersimpan, dapat diedit kembali |
| 4 | Operator Pajak | Review dan verifikasi isi batch | Jika ada koreksi, kembali ke langkah 2 |
| 5 | Operator Pajak | Finalisasi batch → status: FINAL | Batch terkunci, tidak dapat diubah (RULE-015) |
| 6 | Operator Pajak | Laporkan batch ke DJP (saat ini: manual di luar sistem) | Batch siap dilaporkan |

### Exception Paths

| Kondisi | Penanganan |
|---------|------------|
| Jurnal Setor dari unit lain belum ada | Batch dapat dibuat dengan jurnal yang sudah tersedia; tidak harus menunggu semua unit |
| Upaya gabung jurnal beda jenis pajak dalam satu batch | Backend menolak (RULE-014) |
| Upaya edit/tarik batch yang sudah FINAL | Sistem menolak (RULE-015) |

---

## WF-006: Penutupan & Pembukaan Periode

**Aktor:** Admin
**Trigger:** Akhir bulan (penutupan) atau permintaan koreksi dari unit (pembukaan)
**Business Rules:** RULE-006, RULE-008, RULE-009

### Happy Path — Penutupan

| # | Aktor | Aksi | Outcome |
|---|-------|------|---------|
| 1 | Admin | Pilih menu Periode → Tutup Periode | Sistem tampilkan daftar unit dan status periode bulan yang dipilih |
| 2 | Admin | Pilih unit (satu atau semua) dan bulan yang akan ditutup | Ringkasan jumlah jurnal bulan tersebut tampil |
| 3 | Admin | Konfirmasi penutupan | Periode unit terpilih berubah status: CLOSED (RULE-008) |
| 4 | Sistem | Blokir semua operasi tulis jurnal untuk unit + bulan yang ditutup | Unit tidak dapat membuat atau mengedit jurnal untuk periode tersebut |

### Happy Path — Pembukaan Kembali

| # | Aktor | Aksi | Outcome |
|---|-------|------|---------|
| 1 | — | Unit mengajukan permintaan koreksi (mekanisme: [NEEDS CLARIFICATION] — OI-005) | Admin menerima informasi kebutuhan koreksi |
| 2 | Admin | Pilih menu Periode → Buka Kembali, pilih unit spesifik dan bulan | Hanya unit tersebut yang periodenya dibuka (RULE-009) |
| 3 | Admin | Konfirmasi pembukaan | Status periode unit: OPEN; unit lain tidak terpengaruh (RULE-009) |
| 4 | Unit | Melakukan koreksi (WF-001 atau WF-007-Koreksi) | Koreksi selesai |
| 5 | Admin | Tutup kembali periode unit tersebut | Kembali ke status CLOSED |

### Exception Paths

| Kondisi | Penanganan |
|---------|------------|
| Role selain Admin mencoba tutup/buka periode | Akses ditolak 403 (RULE-008, RULE-009) |
| Buka periode yang belum pernah ditutup | Tidak relevan; sistem hanya menampilkan periode yang berstatus CLOSED sebagai pilihan buka |

---

## WF-007: Jurnal Koreksi

**Aktor:** Operator / BPP / Operator Pusat
**Trigger:** Ditemukan kesalahan pada jurnal yang sudah terkunci (tidak bisa diedit langsung)
**Business Rules:** RULE-006, RULE-007, RULE-008, RULE-009

### Happy Path

| # | Aktor | Aksi | Outcome |
|---|-------|------|---------|
| 1 | Operator/BPP | Identifikasi jurnal yang salah; periode sudah ditutup atau kondisi kunci lainnya | Jurnal tidak dapat diedit langsung (RULE-006) |
| 2 | Admin | Jika perlu, buka kembali periode unit tersebut (WF-006) | Periode terbuka untuk unit bersangkutan |
| 3 | Operator/BPP | Pilih menu Jurnal → Jurnal Koreksi; masukkan referensi Nomor Jurnal yang dikoreksi (RULE-007) | Sistem tampilkan jurnal asli sebagai referensi |
| 4 | Operator/BPP | Isi entry koreksi (reverse/contra dari jurnal asli, atau koreksi selisih) | Entry koreksi menghasilkan balance (RULE-001) |
| 5 | Operator/BPP | Simpan Jurnal Koreksi | Jurnal Koreksi tersimpan dengan referensi ke jurnal asli (RULE-007, RULE-009) |
| 6 | Operator/BPP | Jika diperlukan, buat jurnal baru yang benar (WF-001 atau WF-002) | Pencatatan kembali benar |

### Exception Paths

| Kondisi | Penanganan |
|---------|------------|
| Nomor jurnal referensi tidak ditemukan | Sistem menolak; tampilkan pesan nomor tidak valid (RULE-007) |
| Jurnal Koreksi tidak balance | Sistem menolak (RULE-001) |
| Periode belum dibuka kembali oleh Admin | Simpan diblokir (RULE-008) |

---

## WF-008: Generate & Upload Laporan

**Aktor:** Operator/BPP (unit sendiri), Admin, Operator Pusat
**Trigger:** Kebutuhan laporan periodik atau laporan formal (BKU, LPJ, Neraca Saldo, Daya Serap, dll.)
**Business Rules:** RULE-011, RULE-019

### Happy Path — Generate Laporan

| # | Aktor | Aksi | Outcome |
|---|-------|------|---------|
| 1 | User | Pilih menu Laporan → pilih jenis laporan | Form parameter laporan terbuka |
| 2 | User | Set parameter: unit (sesuai akses role — RULE-019), periode, dan format output (PDF / Excel) | Parameter siap |
| 3 | User | Untuk laporan komparatif: pilih tahun pembanding (hanya tahun sebelumnya tersedia — RULE-011) | Parameter tahun terisi |
| 4 | Sistem | Generate laporan; tampilkan versi monitoring di layar | Preview laporan tersedia |
| 5 | User | Download / Cetak laporan ke format yang dipilih | File laporan tersedia |

### Happy Path — Upload Dokumen Bertanda Tangan

| # | Aktor | Aksi | Outcome |
|---|-------|------|---------|
| 6 | User | Cetak laporan, tandatangani secara fisik (di luar sistem) | Dokumen fisik siap |
| 7 | User | Buka history laporan di sistem; pilih laporan yang sudah digenerate | Entry laporan dengan status `BELUM UPLOAD` tampil |
| 8 | User | Upload file PDF dokumen yang sudah ditandatangani | File tersimpan; status berubah ke `SUDAH UPLOAD` beserta timestamp |
| 9 | Sistem | Dokumen tersedia untuk dilihat oleh Admin, Operator Pusat, Auditor/Reviewer | Akses terkontrol sesuai role |

### Exception Paths — Generate

| Kondisi | Penanganan |
|---------|------------|
| User memilih unit di luar assignment-nya | Backend tolak dengan 403 (RULE-019) |
| Tahun pembanding di luar tahun-sebelumnya dipilih | Pilihan dinonaktifkan di UI; ditolak di backend (RULE-011) |
| Data laporan Daya Serap tidak tersedia (MEMO belum sinkron) | Tampilkan peringatan bahwa data pagu belum tersedia |

### Catatan Khusus: BKU

BKU jika dipilih cakupan "Universitas Sebelas Maret" (semua unit), sistem menggabungkan BKU seluruh unit dalam satu laporan konsolidasi — data per unit tetap terperinci di dalamnya.

---

## WF-009: Impersonasi User (Pinjam User)

**Aktor:** Admin
**Trigger:** Kebutuhan troubleshooting, verifikasi tampilan, atau dukungan teknis ke user
**Business Rules:** RULE-017, RULE-020

### Happy Path

| # | Aktor | Aksi | Outcome |
|---|-------|------|---------|
| 1 | Admin | Pilih menu Pinjam User | Daftar semua user aktif tampil, dapat dicari/filter |
| 2 | Admin | Pilih user target | Konfirmasi impersonasi |
| 3 | Sistem | Catat log: `[Admin: X] memulai impersonasi sebagai [User: Y] pada [timestamp]` (RULE-017, RULE-020) | Log tersimpan |
| 4 | Sistem | Alihkan tampilan ke perspektif user target: menu, akses data, dan batasan sesuai role user target | Banner/indikator "Mode Pinjam User" aktif |
| 5 | Admin | Operasikan sistem sebagai user target | Semua aksi dicatat atas nama Admin (RULE-017) |
| 6 | Admin | Klik "Keluar dari Mode Pinjam User" | Tampilan kembali ke dashboard Admin |
| 7 | Sistem | Catat log: `[Admin: X] mengakhiri impersonasi pada [timestamp]` (RULE-020) | Log sesi impersonasi lengkap tersimpan |

### Exception Paths

| Kondisi | Penanganan |
|---------|------------|
| Role selain Admin mengakses endpoint impersonasi | Akses ditolak 403 (RULE-017) |
| User target dinonaktifkan saat sesi impersonasi berjalan | Sesi berakhir otomatis; Admin dikembalikan ke dashboard (TBD — perlu konfirmasi) |

---

## WF-010: Manajemen User

**Aktor:** Admin
**Trigger:** Kebutuhan tambah, ubah, atau nonaktifkan pengguna

### Happy Path — Tambah User

| # | Aktor | Aksi | Outcome |
|---|-------|------|---------|
| 1 | Admin | Pilih menu User → Tambah User | Form input terbuka |
| 2 | Admin | Isi nama, username, email, password awal, role, dan assignment unit | Form lengkap |
| 3 | Admin | Simpan | User aktif tersimpan; dapat login |

### Happy Path — Nonaktifkan User

| # | Aktor | Aksi | Outcome |
|---|-------|------|---------|
| 1 | Admin | Pilih user dari daftar → Nonaktifkan | Konfirmasi |
| 2 | Admin | Konfirmasi | User tidak dapat login; data history tetap tersimpan; dapat diaktifkan kembali |

### Exception Paths

| Kondisi | Penanganan |
|---------|------------|
| Username / email sudah terdaftar | Sistem menolak duplikasi; tampilkan pesan |
| Hapus permanen user dengan data jurnal | Tidak diizinkan; hanya nonaktifkan |

---

## WF-011: Manajemen COA

**Aktor:** Admin
**Trigger:** Kebutuhan tambah atau ubah akun sesuai BASK atau kebutuhan internal
**Business Rules:** RULE-003, RULE-018

### Happy Path

| # | Aktor | Aksi | Outcome |
|---|-------|------|---------|
| 1 | Admin | Pilih menu Master Data → COA | Daftar akun aktif tampil |
| 2 | Admin | Tambah akun: isi kode (format BASK), nama, tipe, posisi normal (Debet/Kredit) | Akun baru aktif, tersedia di pilihan form jurnal (RULE-003) |
| 3 | Admin | Edit akun: ubah nama atau atribut yang diizinkan | Jika akun sudah digunakan di jurnal, kode tidak dapat diubah |
| 4 | Admin | Nonaktifkan akun | Akun tidak muncul di form jurnal baru; data historis tetap terbaca (RULE-018) |

### Exception Paths

| Kondisi | Penanganan |
|---------|------------|
| Hapus permanen akun yang sudah ada di jurnal | Sistem menolak; tawarkan opsi nonaktifkan saja (RULE-018) |
| Kode akun duplikat | Sistem menolak penyimpanan |

---

## WF-012: Sinkronisasi MEMO/PAGU

**Aktor:** Sistem (terjadwal) / Admin (manual)
**Trigger:** Jadwal otomatis berkala atau permintaan manual Admin

### Happy Path

| # | Aktor | Aksi | Outcome |
|---|-------|------|---------|
| 1 | Sistem/Admin | Trigger sinkronisasi (otomatis terjadwal atau Admin klik "Sinkronisasi Sekarang") | Proses dimulai |
| 2 | Sistem | Panggil API sistem perencanaan | Data MEMO/PAGU diterima |
| 3 | Sistem | Update tabel lokal: INSERT untuk MEMO baru, UPDATE untuk yang berubah, UPDATE status untuk yang tidak aktif | Tabel lokal terkini |
| 4 | Sistem | Catat log sinkronisasi: waktu, jumlah record, status (sukses/gagal) | Log tersedia untuk monitoring |
| 5 | Sistem | MEMO terbaru tersedia untuk dipilih di Jurnal Transaksi (WF-002) | Data siap digunakan |

### Exception Paths

| Kondisi | Penanganan |
|---------|------------|
| API sistem perencanaan tidak merespons | Log error sinkronisasi; data lama tetap digunakan; Admin dinotifikasi |
| Konflik data (MEMO yang sudah DIGUNAKAN berubah di sumber) | [NEEDS CLARIFICATION] — perlu kebijakan dari tim akuntansi |