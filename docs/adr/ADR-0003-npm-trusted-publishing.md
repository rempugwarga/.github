# ADR-0003: Rilis Design System via Changesets + npm Trusted Publishing

- **Status:** Diterima
- **Tanggal:** 2026-10-18
- **Konteks tim:** Design System Team

## Konteks

Design system dipakai seluruh tim aplikasi. Karena itu ia butuh **versioning
yang jelas** (breaking change tidak boleh mengejutkan konsumen), distribusi
yang andal, dan **tanpa token yang disimpan di repo** (kebutuhan keamanan:
tidak ada secret di repo). Modul juga perlu dapat dipakai sebagai remote
Module Federation untuk konsumsi *runtime*.

## Keputusan

1. **Semantic Versioning dengan Changesets** — setiap PR yang mengubah
   perilaku publik menambahkan changeset; rilis dijalankan otomatis oleh
   workflow `release.yml` (patch/minor/major).
2. **Publish ke npm dengan Trusted Publishing (OIDC)** — workflow GitHub
   mendapat identitas federasi dari npm; **tidak ada `NPM_TOKEN`** yang
   disimpan. Provenance disetel (`NPM_CONFIG_PROVENANCE=true`).
3. **Dua jalur konsumsi** — (a) paket npm `@rempugwarga/design-system`
   (build-time, direkomendasikan untuk aplikasi baru); (b) Module Federation
   remote bernama `designSystem` (runtime, untuk aplikasi besar yang ingin
   berbagi runtime design system). Keduanya keluar dari repo yang sama.
4. **Storybook dideploy ke GitHub Pages** per rilis, sehingga kontrak visual
   dan dokumentasi komponen selalu sinkron dengan versi yang dipublikasikan.

## Konsekuensi

**Positif**

- Riwayat rilis terlihat di CHANGELOG.md + tag git; konsumen dapat membaca
  breaking change sebelum upgrade.
- Tidak ada kredensial npm di repo/secret GitHub → risiko pencurian token
  hilang; publikasi hanya bisa dari workflow resmi.
- konsumen dapat memilih kompromi build-time vs runtime.

**Negatif / biaya**

- Rilis butuh disiplin changeset; PR tanpa changeset akan gagal di CI.
- Pemakai MF bertanggung jawab mengelola singleton React (dokumentasi ada di
  `docs/INTEGRATION.md`).
- Trusted Publishing menuntut konfigurasi npm sekali (organization/package
  allowlist) — didokumentasikan di `docs/VERSIONING.md`.

## Alternatif yang ditolak

| Alternatif | Alasan ditolak |
| --- | --- |
| `npm version` manual + `NPM_TOKEN` | Token tersimpan di secret repo, tidak ada provenance, rawan lupa changelog |
| Publish sebagai tarball/git submodule | Tidak ada resolusi semver bagi konsumen; upgrade rawan |
| Hanya Module Federation (tanpa npm) | Sulitnya type-safety dan version pinning; npm tetap cara utama yang paling mudah dipahami |
