# Business Requirements

## Purpose
Capture *what* the business needs, independent of how it will be technically implemented.

## Responsibilities
- Owned by: **Business Analyst**.
- Consumed by: **Software Architect** (to design), **QA/Testing Engineer** (to derive test plans), **Project Manager** (to prioritize).

## Inputs
- Stakeholder conversations, domain knowledge, regulatory/institutional constraints.
- Existing workflows described in `/business/workflows.md`.

## Outputs
- Functional requirements consumed by `/architecture/architecture.md` and `/testing/test-plan.md`.

## Rules
- Requirements describe outcomes and constraints, never a specific implementation ("users must be able to export reports as PDF", not "add a PDF button using library X").
- Every requirement gets a stable ID (e.g., `REQ-001`) so it can be referenced from architecture, tests, and code review.
- Ambiguous requirements must be flagged as `[NEEDS CLARIFICATION]` rather than guessed at.

## Checklist
- [x] Requirement has a stable ID.
- [x] Requirement is testable (QA can write a test case against it).
- [x] Requirement does not prescribe implementation details.

## Notes

Aplikasi keuangan untuk **Universitas Sebelas Maret (UNS)** sebagai PTN-BH (Perguruan Tinggi Negeri Berbadan Hukum). Standar akuntansi mengacu pada **SAP (Standar Akuntansi Pemerintahan)** dan **BASK (Bagan Akun Standar Kemendikbud)**. Aplikasi dirancang agar parameterik sehingga berpotensi diadopsi oleh PTN-BH lain di bawah regulasi Kemendikbud yang sama.

**Konteks teknis:** Aplikasi berbasis web (desktop-first), deploy via CI/CD ke infrastruktur internal kampus (Git internal). Setiap tahun anggaran menggunakan database terpisah (e.g. `keuangan_2026`). Semua transaksi dalam IDR dengan 2 digit desimal.

---

## Requirements Log

### Modul: Jurnal

| ID | Requirement | Source | Status |
|----|-------------|--------|--------|
| REQ-001 | Sistem harus memungkinkan pengguna membuat jurnal dengan minimal lima jenis: Jurnal Umum (multi sub-jenis), Jurnal Transaksi, Jurnal Pajak Terima, Jurnal Pajak Setor, dan Jurnal Koreksi. | Stakeholder | Confirmed |
| REQ-002 | Setiap jurnal yang disimpan harus memiliki total sisi Debet sama dengan total sisi Kredit (double-entry balance). | Stakeholder / SAP | Confirmed |
| REQ-003 | Akun yang dapat dipilih pada setiap jurnal harus dibatasi sesuai jenis jurnal (bukan seluruh COA), dikonfigurasi oleh tim akuntansi melalui master data. | Stakeholder | Confirmed |
| REQ-004 | Untuk Jurnal Transaksi, sistem harus mengambil akun belanja dan nominal secara otomatis dari MEMO yang dipilih pengguna; pengguna hanya memilih akun lawan. | Stakeholder | Confirmed |
| REQ-005 | Setiap jurnal harus memiliki dua nomor identifikasi: Nomor Bukti (per unit, tidak unik global) dan Nomor Jurnal (unik sistem, di-generate otomatis). | Stakeholder | Confirmed |
| REQ-006 | Pengguna harus dapat melampirkan satu file bukti per jurnal, dengan format PDF dan ukuran maksimal 5 MB. | Stakeholder | Confirmed |
| REQ-007 | Jurnal yang sudah terkunci (periode ditutup atau kondisi bisnis tertentu) tidak boleh dapat diedit; koreksi harus dilakukan melalui Jurnal Koreksi baru. | Stakeholder | Confirmed |
| REQ-008 | Jurnal Koreksi harus menyimpan referensi ke nomor jurnal asli yang dikoreksi. | Stakeholder | Confirmed |
| REQ-009 | Sistem harus mencatat semua jurnal secara permanen (immutable); tidak ada mekanisme hapus jurnal yang sudah direkam. | Stakeholder / Audit | Confirmed |

### Modul: Pajak

| ID | Requirement | Source | Status |
|----|-------------|--------|--------|
| REQ-010 | Sistem harus mendukung pencatatan enam jenis pajak: PPh 21, PPh 22, PPh 23, PPh 26, PPN, dan PPh Pasal 4 Ayat 2. | Stakeholder | Confirmed |
| REQ-011 | Setiap Jurnal Pajak Terima harus dilengkapi data detail wajib: Nama Wajib Pajak, NPWP, Nilai Bruto, dan Jenis Pajak. Field tambahan per jenis pajak menyusul. | Stakeholder | Confirmed — field tambahan [NEEDS CLARIFICATION] per jenis pajak |
| REQ-012 | Pengguna harus dapat memilih satu atau lebih Jurnal Pajak Terima yang belum disetor untuk digabungkan ke dalam satu Jurnal Pajak Setor per jenis pajak per bulan. | Stakeholder | Confirmed |
| REQ-013 | Operator Pajak harus dapat mengkonsolidasikan Jurnal Pajak Setor dari semua unit dengan jenis pajak yang sama ke dalam satu Batch Pelaporan per periode. | Stakeholder | Confirmed |
| REQ-014 | Sistem harus menyediakan mekanisme verifikasi/review Batch Pajak sebelum batch difinalisasi untuk pelaporan ke DJP. | Stakeholder | Confirmed |
| REQ-015 | Sistem harus mendukung integrasi dengan Coretax/DJP di masa mendatang; desain modul pajak tidak boleh mengunci pada pelaporan manual. | Stakeholder | Planned — TBD |

### Modul: Laporan

| ID | Requirement | Source | Status |
|----|-------------|--------|--------|
| REQ-016 | Sistem harus menghasilkan minimal laporan: BKU, Neraca Saldo, Daya Serap, dan LPJ, dengan format output PDF dan/atau Excel sesuai jenis laporan. | Stakeholder | Confirmed |
| REQ-017 | BKU harus dapat digenerate per unit/bendahara, dan juga dalam versi konsolidasi seluruh universitas dalam satu laporan. | Stakeholder | Confirmed |
| REQ-018 | LPJ harus digenerate per unit per bulan dan dapat digenerate oleh unit bersangkutan, Admin, atau Operator Pusat. | Stakeholder | Confirmed |
| REQ-019 | Setiap laporan harus memiliki versi monitoring (tampilan di layar) yang dapat langsung dicetak menjadi format laporan formal. | Stakeholder | Confirmed |
| REQ-020 | Sistem harus mendukung upload kembali dokumen laporan yang telah ditandatangani, dan menyimpan status tracking (belum upload / sudah upload) beserta timestamp. | Stakeholder | Confirmed |
| REQ-021 | Dokumen laporan yang telah diupload harus dapat dilihat oleh Admin, Operator Pusat, dan Auditor/Reviewer. | Stakeholder | Confirmed |
| REQ-022 | Sistem harus mendukung laporan komparatif antara tahun aktif dan tahun sebelumnya; data tahun sebelumnya diakses via tabel bantu ringkasan di database tahun aktif. | Stakeholder | Confirmed |
| REQ-023 | Arsitektur laporan harus akomodatif terhadap penambahan jenis laporan baru; untuk saat ini penambahan dilakukan oleh pengembang. | Stakeholder | Confirmed — report builder mandiri direncanakan masa depan |

### Modul: Manajemen User & Akses

| ID | Requirement | Source | Status |
|----|-------------|--------|--------|
| REQ-024 | Sistem harus mendukung minimal delapan role pengguna: Admin, Operator Pusat, Operator, BPP, Verifikator, Operator Pajak, Operator Aset, dan Auditor/Reviewer. | Stakeholder | Confirmed |
| REQ-025 | Setiap pengguna (kecuali Admin, Operator Pusat, Auditor) harus dibatasi aksesnya hanya pada unit yang di-assign kepadanya. | Stakeholder | Confirmed |
| REQ-026 | Admin harus dapat melakukan impersonasi (pinjam user) terhadap user manapun; semua aktivitas selama impersonasi dicatat atas nama Admin di audit log. | Stakeholder | Confirmed |
| REQ-027 | Sistem harus menyediakan CRUD pengguna: tambah, ubah, nonaktifkan (bukan hapus permanen), dan reset password. | Stakeholder | Confirmed |

### Modul: Master Data

| ID | Requirement | Source | Status |
|----|-------------|--------|--------|
| REQ-028 | Admin harus dapat menambah, mengubah, dan menonaktifkan akun COA tanpa melibatkan pengembang; COA mengacu pada BASK Kemendikbud. | Stakeholder | Confirmed |
| REQ-029 | Akun COA yang sudah digunakan dalam jurnal tidak boleh dapat dihapus permanen; hanya dapat dinonaktifkan. | Stakeholder / Audit | Confirmed |
| REQ-030 | Sistem harus menyediakan konfigurasi mapping akun yang diperbolehkan per jenis jurnal, dapat dikelola melalui master data. | Stakeholder | Confirmed |

### Modul: Periode & Tahun Anggaran

| ID | Requirement | Source | Status |
|----|-------------|--------|--------|
| REQ-031 | Sistem harus mendukung penutupan periode bulanan per unit; Admin dapat menutup satu unit atau serentak semua unit. | Stakeholder | Confirmed |
| REQ-032 | Admin harus dapat membuka kembali periode yang sudah ditutup untuk unit tertentu tanpa mempengaruhi unit lain. | Stakeholder | Confirmed |
| REQ-033 | Sistem harus mendukung multi-tahun anggaran dengan database terpisah per tahun; perpindahan tahun aktif dilakukan oleh Admin. | Stakeholder | Confirmed |
| REQ-034 | Data dari tahun anggaran lampau harus tetap dapat diakses dalam mode baca dari tahun aktif. | Stakeholder | Confirmed |

### Modul: Integrasi

| ID | Requirement | Source | Status |
|----|-------------|--------|--------|
| REQ-035 | Sistem harus dapat menyinkronisasi data MEMO/PAGU dari sistem perencanaan eksternal via API; data disimpan ke tabel lokal setelah sinkronisasi. | Stakeholder | Confirmed |
| REQ-036 | MEMO yang sudah digunakan dalam Jurnal Transaksi harus ditandai agar tidak dapat digunakan ulang pada jurnal lain. | Stakeholder | Confirmed — partial usage [NEEDS CLARIFICATION] |

### Non-Fungsional

| ID | Requirement | Source | Status |
|----|-------------|--------|--------|
| REQ-037 | Sistem harus mampu menangani volume 200–500 jurnal per hari dari seluruh unit dengan tetap responsif pada puncak volume. | Stakeholder | Confirmed |
| REQ-038 | Seluruh aktivitas pengguna (termasuk sesi impersonasi) harus dicatat dalam audit log yang tidak dapat diubah. | Stakeholder / Audit | Confirmed |
| REQ-039 | Konfigurasi kode unit, COA, dan alur bisnis harus parameterik (tidak hardcoded untuk UNS) untuk memungkinkan adopsi oleh PTN-BH lain. | Stakeholder | Confirmed |
| REQ-040 | Aplikasi berbasis web dan dioptimalkan untuk browser desktop. | Stakeholder | Confirmed |

---

## Open Items

| ID | Item | Keterangan |
|----|------|------------|
| OI-001 | Field detail tambahan per jenis pajak | Perlu didefinisikan bersama tim akuntansi untuk PPh 21, 22, 23, 26, PPN, PPh 4(2) |
| OI-002 | Partial MEMO usage | Apakah satu MEMO bisa digunakan sebagian di jurnal pertama, sisanya di jurnal lain? |
| OI-003 | Migrasi data historis | Menunggu keputusan pimpinan — greenfield atau migrasi dari sistem lama |
| OI-004 | Modul SPM | Direncanakan sebagai modul lanjutan; detail interaksi dengan jurnal masih TBD |
| OI-005 | Mekanisme permintaan buka periode | Apakah ada alur in-app untuk unit mengajukan permintaan buka periode ke Admin? |
| OI-006 | Wajib/opsional lampiran jurnal | Apakah file bukti PDF bersifat wajib untuk semua jenis jurnal, atau opsional? |
| OI-007 | Report builder mandiri | Direncanakan masa depan — penambahan laporan baru tanpa coding |
| OI-008 | Integrasi Coretax/DJP | Direncanakan masa depan — saat ini pelaporan pajak manual |