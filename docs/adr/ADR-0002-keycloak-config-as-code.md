# ADR-0002: Keycloak sebagai IdP dengan Config-as-Code Terraform

- **Status:** Diterima
- **Tanggal:** 2026-10-18
- **Konteks tim:** Identity Platform Team

## Konteks

Seluruh aplikasi organisasi membutuhkan login tunggal (SSO), kontrol akses
berbasis peran, dan kebijakan token yang seragam. Awalnya tidak ada identity
provider: setiap aplikasi berpotensi membangun autentikasinya sendiri.

Fokus ADR ini bukan sekadar "pakai apa", melainkan **bagaimana mengelola
perubahan IAM dengan aman** — komponen yang salah konfigurasinya berarti
seluruh organisasi tidak bisa login.

## Keputusan

Kami memilih **Keycloak** sebagai identity provider dengan dua lapisan:

1. **Config-as-code (Terraform)** — realm, client, role, dan kebijakan token
   dideklarasikan di `identity-platform/terraform/modules/realm` dan diterapkan
   per environment `envs/{dev,staging,prod}`. Provider: `keycloak/keycloak ~> 5.10`.
2. **Registry data-driven** — definisi client hidup di `clients/*.json`
   (skema tervalidasi) dan kebijakan token di `policies/token-policy.json`;
   Terraform membaca keduanya, sehingga PR = perubahan yang tervalidasi.

**Gerbang mutu:** setiap perubahan diterapkan ke **realm ephemeral** di CI dan
diverifikasi end-to-end (lihat ADR-0005) sebelum menyentuh environment nyata;
ambiente nyata butuh approval environment berjenjang.

## Konsekuensi

**Positif**

- Tidak ada perubahan manual "klik-klik" di Admin Console; semua ter-audit via
  PR + apply tercatat.
- ReiLdev/staging/prod konsisten; drift dideteksi otomatis (plan mingguan).
- Kebijakan token (umur access token, rotasi refresh, PKCE wajib, secret
  wajib untuk confidential client) divalidasi oleh precondition di Terraform.
- Registrasi client menjadi self-service: tim aplikasi cukup membuka PR.

**Negatif / biaya**

- Tim Identity harus memelihara provider Terraform dan siklus upgrade Keycloak.
- State Terraform perlu backend terpusat + locking (didokumentasikan di
  `identity-platform/terraform/envs/*/backend.tf`).
- Secret client tidak bisa dibuat otomatis penuh; diinjeksi via variabel
  `TF_VAR_client_secrets` dari environment (tidak pernah masuk repo).

## Alternatif yang ditolak

| Alternatif | Alasan ditolak |
| --- | --- |
| Kelola Keycloak manual via Admin Console | Tidak dapat direview, rawan drift, tidak auditable |
| Keycloak Admin REST API langsung dari script | Erat ke versi, tidak memiliki plan/preview, tidak traceable seperti Terraform |
| SaaS IdP (Auth0/Okta) | Biaya per MAU tinggi untuk kebutuhan internal; kontrol data di PHP sendiri lebih penting |
