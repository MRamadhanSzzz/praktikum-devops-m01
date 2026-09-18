# Blameless Postmortem — Insiden Kegagalan Deployment Manual

**Tanggal:** 19 September 2026
**Insiden:** Kegagalan deployment aplikasi `app-sentra` dari Developer ke Operations
**Durasi:** ~3 menit
**Status:** Resolved

## Ringkasan Insiden

Pada simulasi serah-terima aplikasi dari peran Developer ke Operations (JOB 2),
Operations gagal menjalankan aplikasi pada percobaan pertama. Aplikasi
menghasilkan `ModuleNotFoundError: No module named 'flask'` karena berkas
`requirements.txt` tidak disertakan dalam artefak serah-terima. Insiden
terselesaikan setelah Operations merekonstruksi daftar dependensi secara manual
dan membuat virtual environment terpisah.

## Kronologi (Timeline)

| Waktu (WIB) | Kejadian |
|---|---|
| 03:04 | Operations menerima folder `serah-terima/` berisi `HANDOVER.md` + `src/app.py` |
| 03:04 | Operations membaca `HANDOVER.md` — instruksi umum, tanpa daftar dependensi |
| 03:05 | `python3 src/app.py` → **GAGAL** `ModuleNotFoundError: No module named 'flask'` |
| 03:05 | Operations mengidentifikasi dependency dari `import` di `src/app.py` |
| 03:06 | Operations merekonstruksi `requirements.txt` (`Flask==3.0.3`) |
| 03:06 | Operations membuat `.venv-ops` dan install dependency |
| 03:07 | Aplikasi berjalan → `Running on http://127.0.0.1:5000` |
| 03:07 | `curl /health` → `{"status":"ok"}` — **RESOLVED** |

## Dampak (waktu terbuang, jumlah kegagalan)

| Dampak | Nilai |
|---|---|
| Waktu terbuang (wait time) | ~2 menit |
| Jumlah kegagalan (failed attempts) | 1 |
| Lead time manual total | ~3 menit |
| Komunikasi Dev↔Ops dibutuhkan | Tinggi |
| %C/A pada instalasi dependensi | 50% (bottleneck) |

## Akar Masalah pada SISTEM (bukan pada orang)

Insiden ini **bukan** disebabkan kelalaian individu Developer. Akar masalahnya
adalah **desain sistem kerja** yang memisahkan peran Dev dan Ops tanpa
mekanisme verifikasi otomatis:

1. **Tidak ada standar artefak serah-terima** — tidak ada definisi formal
   "artefak minimum" yang wajib disertakan (mis. `requirements.txt`,
   versi Python, port, endpoint verifikasi).

2. **Tidak ada pipeline otomasi** — prosedur deployment sepenuhnya manual
   dan bergantung pada dokumen teks yang bisa tidak lengkap.

3. **Tidak ada feedback loop cepat** — kegagalan baru terdeteksi saat Ops
   mencoba menjalankan aplikasi di lingkungan berbeda.

4. **Silo Dev ↔ Ops** — lingkungan Dev dan Ops berbeda (venv vs Python sistem).

## Tindakan Perbaikan (action items) + penanggung jawab peran

| # | Tindakan | Penanggung Jawab | Status |
|---|---|---|---|
| 1 | Standarkan artefak serah-terima minimum: `requirements.txt`, versi Python, port, endpoint `/health` | Peran Developer | ✅ Selesai (JOB 4) |
| 2 | Buat skrip otomasi `setup.sh` yang menggantikan prosedur manual | Peran Developer | Selesai (JOB 4) |
| 3 | Terapkan `set -euo pipefail` di semua skrip deployment agar fail-fast | Peran Developer | Selesai (JOB 4) |
| 4 | Tambahkan health check otomatis di akhir skrip deployment | Peran Developer | Selesai (JOB 4) |
| 5 | Gunakan `.gitignore` untuk mencegah commit `.venv` dan kredensial | Peran Developer | Selesai (JOB 4) |
| 6 | Dokumentasikan dependency dengan `pip freeze` di masa depan | Peran Developer | Rekomendasi |

## Pelajaran yang Diambil

1. **Dokumen manual rentan tidak lengkap** — instruksi seperti "install
   dependensi" tanpa daftar konkret memindahkan beban kerja ke Ops.

2. **Otomasi menghilangkan variabilitas** — `setup.sh` mengurangi langkah
   manual dari 5 menjadi 1.

3. **Fail-fast lebih aman** — `set -euo pipefail` menghentikan skrip di
   titik gagal, bukan melanjutkan dengan state tidak valid.

4. **Blameless culture** — insiden ini tidak menyalahkan individu, melainkan
   mengidentifikasi cacat sistem dan memperbaikinya secara struktural.

5. **Ukur, jangan asumsikan** — %C/A pada instalasi dependensi (50%) adalah
   bukti kuantitatif bahwa bottleneck nyata ada di sana.

---

*Dokumen ini disusun tanpa menyebut nama individu, sesuai prinsip
Blameless Postmortem.*
