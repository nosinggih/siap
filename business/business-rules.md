# Business Rules

## Purpose
Document the invariants and policies the business imposes on the system — the rules that must hold true regardless of implementation.

## Responsibilities
- Owned by: **Business Analyst**.
- Enforced in code by **Backend Developer**; verified by **QA/Testing Engineer**.

## Inputs
- Domain policy from stakeholders (institutional/regulatory constraints — SAP, BASK Kemendikbud, ketentuan PTN-BH).
- Edge cases discovered during implementation or QA.

## Outputs
- Authoritative rule list referenced by backend validation logic and test cases.

## Rules
- Each business rule gets a stable ID (e.g., `RULE-001`) and must state the *consequence* of violation, not just the constraint.
- Conflicting rules must be resolved and documented here before implementation proceeds — never resolved silently in code.
- Rules discovered mid-implementation (e.g., "actually, this field is required only for role X") must be added here retroactively.

## Checklist
- [x] Rule has a stable ID and clear trigger condition.
- [x] Rule has a defined consequence/enforcement point.
- [x] Rule is cross-referenced from the requirement(s) it supports.

---

## Rule Log

| ID | Rule | Trigger | Consequence of Violation | Req. Ref | Status |
|----|------|---------|--------------------------|----------|--------|
| RULE-001 | Setiap jurnal yang disimpan harus balance: total Debet = total Kredit, dihitung hingga 2 desimal (sen). | Penyimpanan jurnal (semua jenis). | Sistem menolak penyimpanan dan menampilkan selisih; jurnal tidak tersimpan. Berlaku di level backend (tidak hanya frontend). | REQ-002 | Confirmed |
| RULE-002 | Semua nilai moneter disimpan dan ditampilkan dalam IDR dengan tepat 2 digit desimal. Tidak ada pembulatan otomatis. | Input dan output nilai di seluruh sistem. | Nilai dengan lebih dari 2 desimal ditolak atau ditruncate — perlu keputusan teknis; data tidak boleh hilang akibat pembulatan diam-diam. | REQ-002 | Confirmed |
| RULE-003 | Akun yang tersedia untuk dipilih pada form jurnal dibatasi sesuai mapping jenis jurnal yang dikonfigurasi di master data — bukan seluruh COA. | Setiap kali form jurnal dibuka. | Akun di luar mapping tidak boleh tersimpan dalam jurnal jenis tersebut meskipun dikirim via API secara langsung. Backend memvalidasi. | REQ-003 | Confirmed |
| RULE-004 | Nomor Jurnal bersifat unik di seluruh sistem dalam satu database tahun anggaran. Format: `[kode_unit].[YY].[XXXXX]`. Nomor Bukti hanya unik per unit. | Generate nomor jurnal saat penyimpanan. | Duplikasi Nomor Jurnal ditolak; sistem harus memiliki mekanisme sequence yang aman dari race condition. | REQ-005 | Confirmed |
| RULE-005 | Lampiran jurnal hanya boleh berformat PDF dengan ukuran maksimal 5 MB. Hanya satu file per jurnal. | Upload lampiran. | File non-PDF atau melebihi 5 MB ditolak sebelum upload. File kedua menimpa file pertama ATAU ditolak — [NEEDS CLARIFICATION]. | REQ-006 | Confirmed — perilaku file kedua perlu klarifikasi |
| RULE-006 | Jurnal yang sudah direkam dan periodenya telah ditutup tidak dapat diedit atau dihapus. Koreksi hanya via Jurnal Koreksi baru. | Upaya edit/delete pada jurnal di periode tertutup. | Sistem menolak operasi edit/delete; mengembalikan error dengan panduan menggunakan Jurnal Koreksi. | REQ-007, REQ-009 | Confirmed |
| RULE-007 | Jurnal Koreksi wajib menyertakan referensi (nomor jurnal) ke entri asli yang dikoreksi. | Penyimpanan Jurnal Koreksi. | Jurnal Koreksi tanpa referensi jurnal asli ditolak. | REQ-008 | Confirmed |
| RULE-008 | Penutupan periode bulanan hanya dapat dilakukan oleh Admin. Penutupan dapat dilakukan per unit (parsial) atau serentak semua unit. Setelah ditutup, tidak ada jurnal baru maupun edit yang dapat dilakukan untuk unit tersebut pada bulan itu. | Eksekusi penutupan periode. | Operasi jurnal oleh unit yang periodenya sudah ditutup ditolak oleh sistem. | REQ-031 | Confirmed |
| RULE-009 | Pembukaan kembali periode yang sudah ditutup hanya dilakukan oleh Admin, dan hanya berlaku untuk unit yang dipilih secara eksplisit — tidak menyebar ke unit lain. | Eksekusi buka kembali periode. | Pembukaan periode oleh role selain Admin ditolak. Unit lain tidak terpengaruh status periodenya. | REQ-032 | Confirmed |
| RULE-010 | Setiap tahun anggaran menggunakan database terpisah. Perpindahan tahun aktif otomatis mengganti koneksi database. Data tahun lama hanya dapat dibaca (read-only) dari tahun aktif. | Perpindahan tahun oleh Admin. | Operasi tulis (insert/update) ke database tahun lama dari konteks tahun aktif harus diblokir. | REQ-033, REQ-034 | Confirmed |
| RULE-011 | Laporan komparatif hanya dapat membandingkan tahun aktif dengan tahun sebelumnya (bukan tahun depan). Data tahun lama diakses dari tabel bantu ringkasan, bukan dari database tahun lama secara langsung saat runtime laporan. | Generate laporan komparatif. | Pilihan tahun pembanding di luar tahun-sebelumnya dinonaktifkan di UI dan ditolak di backend. | REQ-022 | Confirmed |
| RULE-012 | Jurnal Pajak Terima wajib dilengkapi field: Nama Wajib Pajak, NPWP, Nilai Bruto, dan Jenis Pajak sebelum dapat disimpan. Field tambahan per jenis pajak menyusul setelah klarifikasi. | Penyimpanan Jurnal Pajak Terima. | Jurnal tanpa field wajib tersebut ditolak oleh backend. | REQ-011 | Confirmed — field tambahan [NEEDS CLARIFICATION] |
| RULE-013 | Jurnal Pajak Setor hanya boleh mereferensikan Jurnal Pajak Terima yang berstatus BELUM DISETOR. Setelah masuk ke Jurnal Setor, status Jurnal Terima berubah menjadi SUDAH DISETOR dan tidak dapat dipilih lagi. | Pemilihan jurnal terima saat buat jurnal setor. | Backend memvalidasi status setiap jurnal terima yang dipilih; entri yang sudah DISETOR ditolak masuk ke jurnal setor baru. | REQ-012 | Confirmed |
| RULE-014 | Batch Pajak hanya boleh berisi Jurnal Pajak Setor dengan jenis pajak yang sama dan periode bulan yang sama. | Pembentukan Batch oleh Operator Pajak. | Backend menolak penggabungan jurnal setor lintas jenis pajak atau lintas bulan dalam satu batch. | REQ-013 | Confirmed |
| RULE-015 | Jurnal Pajak Setor yang sudah masuk ke sebuah Batch yang berstatus FINAL tidak dapat ditarik kembali atau diubah. | Finalisasi Batch Pajak. | Upaya modifikasi batch final atau isinya ditolak oleh sistem. | REQ-014 | Confirmed |
| RULE-016 | MEMO yang sudah digunakan dalam Jurnal Transaksi ditandai SUDAH DIGUNAKAN dan tidak dapat dipilih ulang pada jurnal lain. | Pemilihan MEMO saat buat Jurnal Transaksi. | Backend memvalidasi status MEMO sebelum jurnal disimpan; MEMO berstatus SUDAH DIGUNAKAN ditolak. Perilaku partial usage [NEEDS CLARIFICATION]. | REQ-036 | Confirmed — partial TBD |
| RULE-017 | Impersonasi (pinjam user) hanya dapat dilakukan oleh Admin. Selama sesi impersonasi, semua aksi tercatat di audit log atas nama Admin — bukan atas nama user yang diimpersonasi. | Aktivasi fitur pinjam user oleh Admin. | Role selain Admin tidak boleh mengakses endpoint impersonasi. Log harus mencatat: [Admin: X] bertindak sebagai [User: Y] pada [timestamp]. | REQ-026 | Confirmed |
| RULE-018 | Akun COA yang sudah pernah digunakan dalam jurnal tidak dapat dihapus permanen dari sistem; hanya dapat dinonaktifkan. Akun nonaktif tidak muncul sebagai pilihan pada form jurnal baru, namun data historis tetap dapat dibaca. | Upaya hapus akun COA oleh Admin. | Delete permanen pada akun COA yang memiliki referensi jurnal ditolak oleh sistem (foreign key constraint / business validation). | REQ-029 | Confirmed |
| RULE-019 | Akses data dibatasi berdasarkan assignment unit user. Operator, BPP, dan Unit hanya dapat membaca dan menulis data pada unit yang di-assign ke akun mereka. | Setiap request data yang mengandung konteks unit. | Backend memvalidasi kepemilikan unit pada setiap operasi; request lintas unit oleh role terbatas menghasilkan error 403. | REQ-025 | Confirmed |
| RULE-020 | Audit log bersifat append-only dan tidak dapat dimodifikasi atau dihapus oleh siapapun, termasuk Admin. | Setiap aktivitas pengguna yang tercatat. | Tidak ada endpoint delete/update pada tabel audit log. Upaya modifikasi langsung ke database harus dimonitor oleh DevOps. | REQ-038 | Confirmed |