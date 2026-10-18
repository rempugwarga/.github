# ADR-0005: Uji IAM di Realm Ephemeral sebagai Gerbang Wajib

- **Status:** Diterima
- **Tanggal:** 2026-10-18
- **Konteks tim:** Identity Platform Team

## Konteks

Keycloak dikelola sebagai kode (ADR-0002). Namun `terraform plan` saja tidak
cukup: kesalahan konfigurasi keycloak (mis. redirect URI salah, PKCE tidak
wajib, audience token keliru) hanya muncul pada runtime autentikasi nyata.
Kehilangan login di produksi berdampak organisasi-wide.

## Keputusan

Setiap perubahan `identity-platform` melewati **uji integrasi realm ephemeral**
di CI sebelum boleh di-apply ke environment nyata:

1. Terraform `apply` ke Keycloak container sementara (image
   `quay.io/keycloak/keycloak:26.8`) yang dijalankan di runner
   (`scripts/wait-for-keycloak.sh` menunggu ready).
2. Script verifikasi kontrak (`tests/integration/verify-oidc.mjs`) memeriksa
   perilaku realm yang benar-benar berjalan:
   - OIDC discovery dapat dibaca dari issuer yang diharapkan.
   - Setiap client di `clients/*.json` benar-benar ada dengan setting kunci
     (PKCE, secret untuk confidential, redirect URI).
   - Alur token: authorization code + PKCE berhasil untuk client publik,
     endpoint token menolak permintaan tanpa verifier, kebijakan umur token
     (mis. access token 5 menit) berdampak nyata.
3. Apply ke dev/staging/prod hanya berjalan jika seluruh uji lolos dan
   environment terkait menyetujui (approval berjenjang).

## Konsekuensi

**Positif**

- Regresi konfigurasi IAM tertangkap sebelum menyentuh environment nyata;
  target "0 perubahan IAM gagal di prod" dapat dipantau.
- Kontrak OIDC (issuer, audience, umur token) diuji sebagai perilaku, bukan
  sekadar keberadaan resource di state.
- Kepercayaan meningkat: tim aplikasi dapat mereferensikan perilaku ini di
  dokumentasi mereka.

**Negatif / biaya**

- CI identity-platform lebih berat (Docker + container sehat + tunggu siap);
  durasi bertambah (kisaran menit, bukan detik).
- Container Keycloak di runner bisa break saat upgrade image; versi image
  dipin sekali dan di-Dependabot-kan (sesuai kebijakan repo).

## Alternatif yang ditolak

| Alternatif | Alasan ditolak |
| --- | --- |
| Hanya `terraform plan` | Tidak dapat mendeteksi semua kelas kesalahan perilaku (redirect, PKCE, audience) |
| Uji di environment dev bersama | Dampaknya luas ke tim lain; tidak hermetik; bisa bentrok |
| Mock Keycloak dengan API palsu | Tidak menguji perilaku realm sungguhan; memberi rasa aman palsu |
