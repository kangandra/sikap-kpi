# Kontrak Integrasi Backend ↔ Frontend (SIKAP-KPI)

Dokumen ini merupakan pelengkap dari **Dokumentasi Project SIKAP-KPI** agar anggota **Backend** dan **Frontend** dapat bekerja paralel tanpa konflik nama *route*, variabel Blade, maupun atribut `name` pada form.

## 1. Kesepakatan Umum & Klarifikasi Teknis

1. **Autentikasi Login:** Menggunakan **`email`** dan **`password`** (menyesuaikan struktur tabel `users` bawaan Laravel, bukan `username`).

2. **Flash Message Session:**

   * `session('success')` → Notifikasi berhasil (alert hijau Bootstrap).

   * `session('error')` → Notifikasi gagal / aturan bisnis dilanggar (misal: bobot ≠ 100% atau penilaian ganda; alert merah Bootstrap).

3. **Validasi Error:** Frontend wajib menampilkan `@error('nama_field')` di bawah tiap input dan/atau ringkasan `$errors->all()`.

4. **Konvensi Nama Kolom `users`:** Menggunakan `name` dan `email` (bawaan Laravel), ditambah kolom kustom `role`, `jabatan`, dan `departemen_id`.

5. **Konvensi Relasi Eloquent (wajib dipakai di Blade):**

   * `$user->departemen` (`belongsTo` ke `Departemen`, gunakan null-safe `$user->departemen?->nama_departemen ?? '-'`)

   * `$penilaian->periode` (`belongsTo` ke `Periode`)

   * `$penilaian->karyawan` (`belongsTo` ke `User` via `karyawan_id`)

   * `$penilaian->penilai` (`belongsTo` ke `User` via `penilai_id`)

   * `$penilaian->details` (`hasMany` ke `PenilaianDetail`)

   * `$detail->indikator` (`belongsTo` ke `IndikatorKpi` via `indikator_id`)

## 2. Daftar Route & Nama Route (`routes/web.php`)

Semua form `PUT` dan `DELETE` di Blade wajib menyertakan `@csrf` dan `@method('PUT')` atau `@method('DELETE')`.

| Modul | Method | URL | Nama Route | Controller@Method | Role | 
| ----- | ----- | ----- | ----- | ----- | ----- | 
| **Auth** | `GET` | `/login` | `login` | `AuthController@showLogin` | Guest | 
|  | `POST` | `/login` | `login.post` | `AuthController@login` | Guest | 
|  | `POST` | `/logout` | `logout` | `AuthController@logout` | Auth | 
| **Dashboard** | `GET` | `/dashboard` | `dashboard` | `DashboardController@index` | Admin, Penilai\* | 
| **Profil** | `GET` | `/profil` | `profil` | `ProfilController@edit` | Semua | 
|  | `PUT` | `/profil/password` | `profil.password` | `ProfilController@updatePassword` | Semua | 
| **Nilai Saya** | `GET` | `/nilai-saya` | `nilai-saya` | `NilaiSayaController@index` | Karyawan | 
| **Departemen** | `RESOURCE` | `/departemen` | `departemen.{index,create,store,edit,update,destroy}` | `DepartemenController` | Admin | 
| **Karyawan** | `RESOURCE` | `/karyawan` | `karyawan.{index,create,store,edit,update,destroy}` | `KaryawanController` | Admin | 
| **Indikator** | `RESOURCE` | `/indikator` | `indikator.{index,create,store,edit,update,destroy}` | `IndikatorController` | Admin | 
| **Periode** | `RESOURCE` | `/periode` | `periode.{index,create,store,edit,update,destroy}` | `PeriodeController` | Admin | 
| **Penilaian** | `RESOURCE` | `/penilaian` | `penilaian.{index,create,store,edit,update,destroy}` | `PenilaianController` | Admin, Penilai | 
| **Laporan** | `GET` | `/laporan` | `laporan` | `LaporanController@index` | Admin, Penilai | 
|  | `GET` | `/laporan/{penilaian}` | `laporan.show` | `LaporanController@show` | Admin, Penilai | 

> **Catatan:**
>
> * `*` Jika user ber-role `karyawan` mengakses `/dashboard`, `DashboardController@index` otomatis melakukan `redirect()->route('nilai-saya')`.
>
> * Route `penilaian.show` diarahkan ke `laporan.show` (atau tidak dibuat terpisah) agar tidak ada duplikasi tampilan detail nilai.

## 3. Kontrak Form Input (`name="..."`)

Frontend **wajib** memakai atribut `name` berikut, dan Backend **wajib** memvalidasi key yang sama pada `$request->validate(...)`.

### A. Login (`POST /login`)

* `email` (string, email, wajib)

* `password` (string, wajib)

### B. Ubah Password (`PUT /profil/password`)

* `current_password` (string, wajib)

* `password` (string, min:6, confirmed, wajib)

* `password_confirmation` (string, wajib)

### C. Departemen (`POST /departemen` & `PUT /departemen/{id}`)

* `nama_departemen` (string, max:100, wajib)

### D. Karyawan / User (`POST /karyawan` & `PUT /karyawan/{id}`)

* `name` (string, max:100, wajib)

* `email` (string, email, unique, wajib)

* `password` (string, min:6 — wajib saat `store`, opsional/nullable saat `update`)

* `role` (enum: `'admin'`, `'penilai'`, `'karyawan'`, wajib)

* `jabatan` (string, max:100, nullable)

* `departemen_id` (integer, exists:departemen,id, nullable)

### E. Indikator KPI (`POST /indikator` & `PUT /indikator/{id}`)

* `nama_indikator` (string, max:100, wajib)

* `deskripsi` (text, nullable)

* `bobot` (numeric, min:1, max:100, wajib)

* `status` (enum: `'aktif'`, `'nonaktif'`, wajib)

### F. Periode Penilaian (`POST /periode` & `PUT /periode/{id}`)

* `nama_periode` (string, max:100, wajib)

* `tanggal_mulai` (date `YYYY-MM-DD`, wajib)

* `tanggal_selesai` (date `YYYY-MM-DD`, after_or_equal:tanggal_mulai, wajib)

* `status` (enum: `'buka'`, `'tutup'`, wajib)

### G. Penilaian (`POST /penilaian` & `PUT /penilaian/{id}`)

* `periode_id` (integer, exists:periode,id, wajib)

* `karyawan_id` (integer, exists:users,id, wajib)

* `catatan` (text, nullable)

* `rating[indikator_id]` (array asosiatif; key = `id` dari `indikator_kpi`, value = integer `1`–`5`, wajib untuk semua indikator berstatus `'aktif'`)

  * Contoh HTML: `<input type="radio" name="rating[{{ $indikator->id }}]" value="4" required>`

### H. Filter Pencarian & Laporan (`GET`)

* **Karyawan (`GET /karyawan`):** `?search=...`

* **Penilaian (`GET /penilaian`):** `?periode_id=...`

* **Laporan (`GET /laporan`):** `?periode_id=...&departemen_id=...&kategori=...`

  * Nilai kosong `""` pada filter dianggap "Semua".

  * Nilai valid `kategori`: `'Sangat Baik'`, `'Baik'`, `'Cukup'`, `'Kurang'`, `'Sangat Kurang'`.

## 4. Kontrak Variabel dari Controller ke View Blade

### A. `dashboard.blade.php` (`DashboardController@index`)

* `$periodeAktif` → Model `Periode` terbaru yang berstatus `'buka'` (atau `null` jika belum ada).

* `$ringkasan` → Array asosiatif:

  ```
  [
      'total_karyawan' => 6,      // int: jumlah user role 'karyawan'
      'sudah_dinilai'  => 4,      // int: jumlah penilaian pada periode aktif
      'belum_dinilai'  => 2,      // int: total_karyawan - sudah_dinilai
      'rata_rata'      => 81.50,  // float: rata-rata nilai_akhir periode aktif
  ]
  
  ```

* `$grafikDepartemen` → Array untuk Bar Chart (Chart.js):

  ```
  [
      'labels' => ['HRD', 'Keuangan', 'IT'], // array of string
      'data'   => [84.5, 79.0, 88.2],        // array of float
  ]
  
  ```

* `$grafikKategori` → Array untuk Pie/Doughnut Chart (Chart.js):

  ```
  [
      'labels' => ['Sangat Baik', 'Baik', 'Cukup', 'Kurang', 'Sangat Kurang'],
      'data'   => [1, 2, 1, 0, 0], // array of int
  ]
  
  ```

* `$ranking` → Collection/Array berisi maksimal 5–10 data `Penilaian` (diurutkan `nilai_akhir DESC`) dengan relasi `karyawan.departemen` sudah di-load (`with(['karyawan.departemen'])`).

### B. `departemen/` (`DepartemenController`)

* `departemen/index.blade.php`:

  * `$departemens` → Paginator/Collection model `Departemen` (dengan `withCount('users')` jika ingin menampilkan jumlah karyawan).

* `departemen/create.blade.php`: *(tanpa variabel khusus)*

* `departemen/edit.blade.php`:

  * `$departemen` → Single model `Departemen`.

### C. `karyawan/` (`KaryawanController`)

* `karyawan/index.blade.php`:

  * `$karyawans` → Paginator/Collection model `User` (`with('departemen')`).

* `karyawan/create.blade.php`:

  * `$departemens` → Collection semua `Departemen` (untuk dropdown).

* `karyawan/edit.blade.php`:

  * `$karyawan` → Single model `User`.

  * `$departemens` → Collection semua `Departemen`.

### D. `indikator/` (`IndikatorController`)

* `indikator/index.blade.php`:

  * `$indikators` → Collection semua `IndikatorKpi`.

  * `$totalBobot` → Float/Int total `bobot` dari indikator yang berstatus `'aktif'` (untuk peringatan jika `!= 100`).

* `indikator/create.blade.php`: *(tanpa variabel khusus)*

* `indikator/edit.blade.php`:

  * `$indikator` → Single model `IndikatorKpi`.

### E. `periode/` (`PeriodeController`)

* `periode/index.blade.php`:

  * `$periodes` → Collection semua `Periode` (diurutkan terbaru).

* `periode/create.blade.php`: *(tanpa variabel khusus)*

* `periode/edit.blade.php`:

  * `$periode` → Single model `Periode`.

### F. `penilaian/` (`PenilaianController`)

* `penilaian/index.blade.php`:

  * `$penilaians` → Paginator/Collection `Penilaian` (`with(['periode', 'karyawan.departemen', 'penilai'])`).

  * `$periodes` → Collection `Periode` (untuk filter).

* `penilaian/create.blade.php`:

  * `$periodes` → Collection `Periode` dengan `status = 'buka'`.

  * `$karyawans` → Collection `User` dengan `role = 'karyawan'` (`with('departemen')`).

  * `$indikators` → Collection `IndikatorKpi` dengan `status = 'aktif'` (wajib menyertakan `id`, `nama_indikator`, `deskripsi`, `bobot` agar JS dapat menghitung pratinjau skor).

  * `$totalBobot` → Float/Int total bobot indikator aktif (untuk validasi/peringatan UI jika `!= 100`).

* `penilaian/edit.blade.php`:

  * `$penilaian` → Single model `Penilaian` (`with(['periode', 'karyawan', 'details'])`).

  * `$indikators` → Collection `IndikatorKpi` aktif.

  * `$existingRatings` → Array asosiatif `[indikator_id => rating]` agar radio button otomatis terpilih (`checked`).

### G. `laporan/` (`LaporanController`)

* `laporan/index.blade.php`:

  * `$penilaians` → Collection `Penilaian` (`with(['periode', 'karyawan.departemen', 'penilai'])`).

  * `$periodes` → Collection semua `Periode` (dropdown filter).

  * `$departemens` → Collection semua `Departemen` (dropdown filter).

* `laporan/show.blade.php`:

  * `$penilaian` → Single model `Penilaian` (`with(['periode', 'karyawan.departemen', 'penilai', 'details.indikator'])`).

### H. `nilai-saya.blade.php` (`NilaiSayaController@index`)

* `$riwayat` → Collection `Penilaian` milik `Auth::id()` (`with(['periode', 'penilai', 'details.indikator'])`, diurutkan berdasarkan periode).

* `$penilaianTerbaru` → Model `Penilaian` terbaru milik karyawan tersebut (atau `null` jika belum pernah dinilai).

* `$grafikRiwayat` → Array untuk Line Chart perkembangan nilai antar periode:

  ```
  [
      'labels' => ['Semester 1 2026', 'Semester 2 2026'],
      'data'   => [78.5, 84.0],
  ]
  
  ```

### I. `profil.blade.php` (`ProfilController@edit`)

* `$user` → `Auth::user()` (`load('departemen')`).

## 5. Kontrak Logika Perhitungan & Aturan Bisnis

1. **Rumus Skor & Nilai Akhir:**

   * $\text{Skor Indikator} = \left(\frac{\text{rating}}{5}\right) \times \text{bobot}$

   * $\text{Nilai Akhir} = \sum \text{Skor Indikator}$ (dibulatkan 2 angka di belakang koma).

   * Frontend (JavaScript pratinjau) dan Backend (`PenilaianController`) **harus menggunakan rumus yang identik**.

2. **Kategori Nilai Akhir:**

   * `>= 90` → `'Sangat Baik'`

   * `>= 80` dan `< 90` → `'Baik'`

   * `>= 70` dan `< 80` → `'Cukup'`

   * `>= 60` dan `< 70` → `'Kurang'`

   * `< 60` → `'Sangat Kurang'`

3. **Validasi Sebelum Simpan Penilaian:**

   * Total bobot seluruh indikator berstatus `'aktif'` harus tepat `100`. Jika tidak, tolak dan kembalikan `redirect()->back()->with('error', 'Total bobot indikator aktif harus 100%.')`.

   * Kombinasi `(periode_id, karyawan_id)` tidak boleh duplikat. Jika sudah ada, tolak dan kembalikan `redirect()->back()->withInput()->with('error', 'Karyawan ini sudah dinilai pada periode tersebut.')`.

   * Hanya `periode` berstatus `'buka'` yang boleh menerima penilaian baru.