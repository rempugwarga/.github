# ADR-0001: Polyrepo per Batas Tim (bukan Monorepo)

- **Status:** Diterima
- **Tanggal:** 2026-10-18
- **Konteks tim:** Platform Engineering

## Konteks

Organisasi memiliki beberapa domain dengan pemilik berbeda: UI platform (design
system), identitas (Keycloak), dan pipeline/governance (CI, docs, template).
Repositori awal hanya berisi `design-system` tanpa CI dan tanpa pemilik
organisasi. Kami perlu memutuskan: satu monorepo, atau polyrepo?

Tim masih kecil (5–6 tim potensial), sehingga *delivery* per repo lebih
menentukan daripada efisiensi *build* bersama.

## Keputusan

Kami memilih **polyrepo**: satu repository per batas kepemilikan tim
(`design-system`, `identity-platform`, `platform-ci`, `platform-docs`,
`app-template` + repo aplikasi per tim stream-aligned), dengan:

- **Fracture plane berdasarkan domain & cadence**, bukan lapisan teknologi.
- Kontrak lintas repo: semver npm (design system), OIDC discovery
  (identity), workflow reusable (CI), dokumen kontrak (platform-docs).
- Governance terpusat via `platform-ci/repos.yaml` + Terraform org (lihat
  ADR-0004), sehingga polyrepo tidak berarti chaos.

## Konsekuensi

**Positif**

- Kepemilikan tidak ambigu: CODEOWNERS = tim pemilik; tidak ada "semua orang
  boleh ubah".
- Cadence rilis independen: design system bisa naik minor tanpa mengganggu
  pipeline identity-platform.
- Hak akses, CI, dan on-call mengikuti batas tim (mendukung Conway's law).
- Setiap repo kecil → waktu CI singkat dan rebuild tidak saling memblokir.

**Negatif / biaya**

- Perubahan lintas repo butuh PR berurutan; membakar template `app-template`
  butuh publikasi npm lebih dulu.
- Tidak ada build graph terpadu; duplikasi tooling (tsconfig, rstest config).
  _Mitigasi:_ template dan workflow reusable menyediakan baseline bersama.
- Dependabot harus diaktifkan per repo (sudah disetel di setiap repo).

## Alternatif yang ditolak

| Alternatif | Alasan ditolak |
| --- | --- |
| Monorepo (Turborepo/Nx) | Satu antrian CI, kepemilikan tidak jelas per paket, dan blast radius akses terlalu besar untuk tim kecil |
| Satu repo starter + fork per tim | Tidak ada governance terpusat; fork menyimpang (drift) tanpa deteksi |
| Microrepo per komponen | Fragmentasi berlebihan; setiap komponen platform menghasilkan repo dengan perawatan penuh |
