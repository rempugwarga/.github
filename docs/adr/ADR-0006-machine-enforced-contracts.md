# ADR-0006: Penegakan Mesin untuk Kontrak Organisasi (Required Checks + Konsistensi Lintas Repo)

- **Status:** Diterima
- **Tanggal:** 2026-10-18
- **Konteks tim:** Platform Engineering (review bersama Identity Platform & Design System)

## Konteks

Setelah ADR-0001 (polyrepo) dan ADR-0004 (registry + reusable workflows)
diterapkan, ditemukan tiga celah penegakan:

1. **Branch protection tidak menegakkan CI.** `required_checks` di `repos.yaml`
   masih kosong di semua repo, sehingga PR dapat di-merge meski pipeline merah —
   governance baru sebatas niat, bukan kendali.
2. **Kontrak kepemilikan tersebar** di banyak artefak (`CODEOWNERS` tiap repo,
   `services.yaml` katalog, `clients/*.json`) dan hanya sebagian yang divalidasi.
   Drift antar-artefak tidak terdeteksi sampai menimbulkan insiden.
3. **Kontrak versioning belum ditegakkan di pintu masuk.** Changeset wajib
   secara kebijakan, tetapi PR tanpa changeset tetap bisa lolos CI.

## Keputusan

1. **`required_checks` wajib dan terisi** untuk setiap repo ber-CI di
   `repos.yaml`; skema + `tests/registry-policy.test.ts` menegakkan minimal satu
   check, dan repo `high`/`critical` wajib memuat minimal satu check keamanan.
   Nama check adalah kontrak: harus sama persis dengan `name:` job pemanggil
   (didokumentasikan di `platform-ci/docs/BRANCH-PROTECTION.md`).
2. **Validasi konsistensi lintas repo sebagai kode** —
   `platform-ci/scripts/check-consistency.ts` memverifikasi CODEOWNERS ↔
   registry, `services.yaml` ↔ registry, dan owner client OIDC ↔ registry.
   Dijalankan mingguan oleh workflow `org-consistency.yml` (secret
   `ORG_READ_TOKEN`, read-only) dan dapat dijalankan lokal dari workspace.
3. **Gerbang changeset di CI design system** — PR yang menyentuh kontrak publik
   (`src/`, `package.json`, konfigurasi build) ditolak bila tanpa
   `.changeset/*.md`; aturan versioning MF remote didokumentasikan di
   `docs/VERSIONING.md`.
4. **Kontrak kebijakan token berversi** — `token-policy.json` memuat `version`
   (semver) + `effectiveDate` yang diuji CI, sehingga perubahan perilaku token
   dapat dikaitkan dengan versi kontrak.
5. **Self-service ditegakkan di hulu** — generator `identity-platform/scripts/new-client.ts`
   menghasilkan konfigurasi client yang lolos schema + kebijakan sebelum PR dibuka.
6. **Review akses admin berjadwal** — workflow kuartalan membuka issue review
   akses Admin Console; otomasi menjamin tidak ada review yang terlewat.

## Konsekuensi

**Positif**

- Merge tanpa CI hijau menjadi tidak mungkin pada semua repo terdaftar.
- Drift kepemilikan terdeteksi mingguan dengan pesan yang menunjuk aksi
  perbaikan, bukan menunggu audit manual.
- Setiap perubahan perilaku token/komponen dapat ditelusuri ke versi kontrak.

**Negatif / biaya**

- Menambah/mengganti nama job di repo kini memerlukan PR ke `repos.yaml` (nama
  check adalah kontrak, bukan detail implementasi).
- `org-consistency.yml` membutuhkan token baca lintas repo (`ORG_READ_TOKEN`)
  yang harus dirotasi sesuai kebijakan secret.
- Pekerjaan tim bertambah kecil di depan (changeset, review PR) untuk menurunkan
  risiko di produksi.

## Alternatif yang ditolak

| Alternatif | Alasan ditolak |
| --- | --- |
| Mengandalkan review manusia untuk menegakkan kontrak | Tidak scalable; drift tetap lolos saat sibuk |
| Monorepo agar semua konsistensi trivial | Ditolak di ADR-0001 (batas tim = batas repo) |
| Bot/aksi pihak ketiga untuk changeset & konsistensi | Menggantungkan governance inti pada dependency tak terkurasi; script sendiri kecil dan teruji |
| Validasi konsistensi real-time via webhook | Kompleks (butuh server + token berumur panjang); jadwal mingguan cukup untuk drift |
