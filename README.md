# platform-organization

Workspace **polyrepo** organisasi rempugwarga. Setiap repository dimiliki satu
tim yang jelas, memiliki CI/CD sendiri, dan berkomunikasi melalui kontrak
terdokumentasi.

> Dokumen arsitektur lengkap: **[ARCHITECTURE.md](ARCHITECTURE.md)**

## Peta repository

| Repository | Tim pemilik | Tipe tim | Fungsi | CI/CD |
| --- | --- | --- | --- | --- |
| [`design-system/`](design-system) | Design System Team | Platform (product) | Komponen React — npm semver + Module Federation remote | build/test/scan/publish + gate changeset + Storybook Pages |
| [`identity-platform/`](identity-platform) | Identity Platform Team | Complicated-subsystem + Platform | Keycloak as code (Terraform), registrasi client (generator), kebijakan token berversi | policy tests + ephemeral realm integration + apply berjenjang + review akses kuartalan |
| [`platform-ci/`](platform-ci) | Platform Engineering | Platform | Reusable workflows, `repos.yaml` (kepemilikan + required checks), branch protection as code | actionlint + registry tests + terraform validate + konsistensi lintas repo mingguan |
| [`platform-docs/`](platform-docs) | Platform Engineering | Platform | Portal kontrak integrasi (OIDC, environment, token, design system) + katalog layanan | build Rspress + validasi link kontrak + deploy Pages |
| [`app-template/`](app-template) | Platform Engineering | Platform (golden path) | Template aplikasi stream-aligned (OIDC + design system + CI) | reusable platform-ci + deploy berjenjang bergerbang security |

Kepemilikan berasal dari satu sumber kebenaran:
[`platform-ci/repos.yaml`](platform-ci/repos.yaml) — divalidasi otomatis
(`platform-ci/tests`), **ditegakkan sebagai required checks** (branch protection
tidak mengizinkan merge sebelum CI hijau), dan diterapkan ke GitHub melalui
[`platform-ci/terraform/org`](platform-ci/terraform/org). Drift antar-artefak
(CODEOWNERS, katalog layanan, owner client) ditangkap
`platform-ci/scripts/check-consistency.ts` secara mingguan — lihat
[ADR-0006](docs/adr/ADR-0006-machine-enforced-contracts.md).

## Cara menjelajah

1. **Arsitektur & keputusan** → [ARCHITECTURE.md](ARCHITECTURE.md),
   [docs/adr/](docs/adr)
2. **Kontrak integrasi untuk tim aplikasi** → `platform-docs/` (jalankan
   `bun install && bun run dev`)
3. **Onboarding aplikasi baru** → `app-template/README.md` +
   `platform-docs/docs/onboarding`
4. **Registrasi client OIDC** → `identity-platform/clients/README.md`

## Prasyarat

- Bun ≥ 1.4 (semua repo memakai `bun`), Node ≥ 20 (script integrasi)
- Terraform ≥ 1.9 + Docker (untuk kerja di `identity-platform`)

## Status verifikasi

| Repo | Verifikasi lokal yang sudah dijalankan |
| --- | --- |
| design-system | `typecheck`, `test`, `build` (publint bersih); gate changeset diuji dengan skenario git (src tanpa changeset → blokir; docs-only → lolos) |
| identity-platform | `typecheck`, `test` (36 uji: kebijakan client, token + versi kontrak, scaffold); generator `new-client` diuji end-to-end. Terraform & integration test berjalan di CI (butuh Docker/Terraform) |
| platform-ci | `typecheck`, `test` (21 uji registry + konsistensi); `check:consistency` bersih untuk kelima repo |
| platform-docs | `typecheck`, `test` (6 uji: schema katalog + link kontrak), `build` Rspress |
| app-template | `typecheck`, `test`, `build` (dengan design-system ter-pack lokal) |
