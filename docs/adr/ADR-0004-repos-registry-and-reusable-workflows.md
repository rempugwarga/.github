# ADR-0004: Registry `repos.yaml` + Reusable Workflows sebagai Governance Terpusat

- **Status:** Diterima
- **Tanggal:** 2026-10-18
- **Konteks tim:** Platform Engineering

## Konteks

Polyrepo (ADR-0001) menciptakan pertanyaan governance: dari mana kita tahu
repo apa saja yang ada, siapa pemiliknya, dan apakah semuanya terlindungi?
Duplikasi file `ci.yml` di setiap repo juga mengundang drift (perbedaan
versi aksi, aturan audit, dsb.).

## Keputusan

1. **Satu sumber kebenaran kepemilikan** — `platform-ci/repos.yaml`
   mendaftarkan setiap repo, tim pemilik, tipe (`node`/`app`/`iac`/`docs`),
   dan aturan khusus. Skema divalidasi otomatis (`repos.schema.json` +
   `tests/registry-policy.test.ts`); setiap repo wajib memiliki entri.
2. **Terraform org** — `platform-ci/terraform/org` membuat tim, repo, dan
   branch protection (ruleset: PR wajib, code owner review, status check)
   dari `repos.yaml` menggunakan provider `integrations/github ~> 6.0`.
3. **Reusable workflows** — `reusable-node-ci.yml`, `reusable-terraform-ci.yml`,
   `reusable-security-scan.yml` mendefinisikan pipeline standar sekali saja;
   repo memanggilnya dengan beberapa baris. Repo juga tetap dapat meng-override
   (escape hatch) tanpa memutus standar.

## Konsekuensi

**Positif**

- Menambah repo = satu PR ke `repos.yaml`; tim, repo, dan proteksi branch
  dibuat seragam.
- Upgrade baseline CI (versi aksi, aturan security scan) cukup di satu file.
- Inventaris repo dapat diaudit (repo tanpa CI = tidak terdaftar → gagal test).

**Negatif / biaya**

- Platform Engineering menjadi jalur kritis untuk perubahan workflow — perlu
  SLA review dan versioning perubahan yang jelas (dokumentasi di
  `platform-ci/docs/REUSABLE-WORKFLOWS.md`).
- Pin ke `@main` workflow reusable berarti perubahan langsung berlaku; untuk
  repo yang butuh stabilitas, disediakan cara pin ke SHA/tag (didokumentasikan).

## Alternatif yang ditolak

| Alternatif | Alasan ditolak |
| --- | --- |
| Salin `ci.yml` ke setiap repo | Drift pada baseline keamanan; perbaikan bug harus dikerjakan N kali |
| Composite action untuk seluruh pipeline | Kurang fleksibel (tidak bisa pakai matrix/strategy native per tahap) |
| GitHub App kustom untuk governance | Biaya pengembangan tinggi; provider Terraform sudah menutupi kebutuhan |
