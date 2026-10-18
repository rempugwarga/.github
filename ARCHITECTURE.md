# Arsitektur Organisasi Platform (Polyrepo + Team Topologies)

> Status: **diterapkan** · Pemilik dokumen: Platform Engineering ·
> Review berikutnya: kuartalan (sensing review)

Dokumen ini mendefinisikan bagaimana organisasi rempugwarga disusun sebagai
**polyrepo dengan topologi tim eksplisit**, agar arsitektur sistem menuruti
struktur komunikasi tim (Conway's law dipakai secara sengaja / *inverse Conway
maneuver*) — bukan sebaliknya.

---

## 1. Konteks & masalah

Kondisi awal: satu repositori starter `design-system` tanpa commit, tanpa CI,
tanpa pembagian kepemilikan. Kebutuhan organisasi:

1. **Ownership** — setiap repo punya tim penanggung jawab jelas.
2. **CI/CD** — build, test, security scanning, deployment otomatis.
3. **Versioning** — design system berversi jelas; perubahan IAM diuji sebelum
   diterapkan.
4. **Kontrak integrasi** — dokumentasi OIDC, URL environment, mekanisme
   registrasi client, kebijakan token.
5. **Akses & keamanan** — tanpa secret di repo; akses Admin Console berbasis
   kebutuhan.
6. **Interaksi tim** — platform menyediakan layanan terdokumentasi; tim
   aplikasi berintegrasi self-service.

## 2. Prinsip desain

| Prinsip | Penerapan |
| --- | --- |
| Team adalah unit delivery | Repo = batas kepemilikan (satu tim pemilik per repo) |
| Inverse Conway | Pecahan repo mengikuti fracture plane: domain layanan, spesialisasi, dan cadence — bukan layer teknologi |
| Platform as product | Layanan platform diukur dari *beban kognitif yang dihapus*; adopsi sukarela, kompetitif |
| Thinnest viable platform | Dokumentasi + workflow reusable + template dulu; software baru hanya bila kurasi tidak cukup |
| X-as-a-Service default | Kontrak terdokumentasi > koordinasi manual; kolaborasi hanya untuk discovery, time-boxed |
| Secret adalah runtime concern | Tidak ada secret di repo; injeksi dari secret manager saat deploy |
| Machine-enforced governance | Kepemilikan & perlindungan branch sebagai kode (`repos.yaml` + Terraform) |

## 3. Topologi tim

```
                    ┌──────────────────────────────────────────┐
                    │        Stream-aligned teams (utama)      │
                    │  Team Billing     Team Web Dashboard ... │
                    │  (repo aplikasi, konsumen platform)      │
                    └───────┬───────────────┬──────────────────┘
        X-as-a-Service      │               │        X-as-a-Service
     (npm + MF remote)      │               │   (OIDC, registrasi client via PR)
                            ▼               ▼
┌─────────────────────┐   ┌───────────────────────────┐   ┌───────────────────────┐
│ Design System Team  │   │ Identity Platform Team    │   │ Platform Engineering  │
│ (platform product)  │   │ (complicated-subsystem +  │   │ (platform team)       │
│ design-system       │   │  platform) identity-      │   │ platform-ci           │
│                     │   │  platform                 │   │ platform-docs         │
│                     │   │                           │   │ app-template          │
└─────────────────────┘   └───────────────────────────┘   └───────────────────────┘
                                                                │
                          Facilitating (time-boxed, ≤ 2 minggu)  │
                          untuk tim baru / adopsi kapabilitas ◄──┘
```

| Tim | Tipe (Team Topologies) | Domain & beban kognitif | Repo |
| --- | --- | --- | --- |
| Design System Team (3–5 orang) | **Platform** (produk internal) | 1 domain kompleks: UI platform, rilis, kontrak komponen | `design-system` |
| Identity Platform Team (4–6 orang) | **Complicated-subsystem + Platform** | 1 domain kompleks: OIDC, token, keamanan identitas | `identity-platform` |
| Platform Engineering (4–6 orang) | **Platform** + **Enabling** | Standar CI/CD, governance, docs, golden path | `platform-ci`, `platform-docs`, `app-template` |
| Tim Billing / Web Dashboard / dst. | **Stream-aligned** | Domain bisnis end-to-end | repo masing-masing (mulai dari `app-template`) |

**Aturan cognitive load:** satu tim maksimal satu domain kompleks; tim tidak
memiliki dua repo platform berat. Tim platform dibentuk hanya untuk mengurangi
beban stream — bukan untuk memindahkan pekerjaan ke antrian tiket.

## 4. Mode interaksi (eksplisit)

| Pasangan | Mode | Kontrak / batas |
| --- | --- | --- |
| Stream ↔ Identity Platform | **X-as-a-Service** | OIDC discovery + PR registrasi client; SLA review ≤ 2 hari kerja |
| Stream ↔ Design System | **X-as-a-Service** | npm semver + MF remote `designSystem`; breaking hanya di major |
| Stream ↔ Platform Engineering (CI/CD, docs) | **X-as-a-Service** | Workflow reusable + kontrak katalog di `services.yaml` |
| Tim baru ↔ Platform Engineering | **Facilitating** | Onboarding ≤ 2 minggu; platform hanya mendampingi (maks. 1 kolaborasi) — tujuannya kemandirian |
| Design System ↔ Identity Platform | **Collaboration** (per inisiatif) | Time-boxed: perubahan kontrak lintas platform (mis. redesign login) |

Mode dievaluasi ulang tiap kuartal melalui *sensing*: gesekan, waktu tunggu
PR review, dan sinyal on-call. Gesekan berulang = sinyal boundary perlu diubah.

## 5. Arsitektur sistem (runtime)

```
                                   ┌──────────────────────────────┐
        Tim aplikasi (stream)      │  Platform services (kontrak) │
  ┌──────────────────────────────┐ │                              │
  │  app-template  → my-app      │ │  identity-platform           │
  │  ├─ OIDC PKCE (oidc-client)  │◄┼──│  Keycloak realm-as-code     │
  │  ├─ design-system (npm/MF)   │◄┼──│  clients/ + token policy    │
  │  └─ CI reusable platform-ci  │◄┼──│  ephemeral integration test │
  └──────────────┬───────────────┘ │                              │
                 │ build/deploy    │  design-system               │
                 ▼                 │  npm semver + MF remote      │
  ┌──────────────────────────────┐ │                              │
  │ Runtime aplikasi             │ │  platform-docs               │
  │ ├─ login ke Issuer env       │◄┼──│  kontrak & katalog layanan  │
  │ └─ memakai token (aud per    │ │                              │
  │    API, exp 5 menit)         │ │  platform-ci                 │
  └──────────────────────────────┘ │  workflows + repos.yaml      │
                                   └──────────────────────────────┘
```

## 6. Pipeline CI/CD per jenis repo

| Tahap | Node/Bun repo (design-system, docs, app) | IaC repo (identity-platform) |
| --- | --- | --- |
| Build | `bun run build` (+ publint untuk paket) | `terraform fmt/validate/tflint` |
| Test | unit/policy tests (`rstest`) | policy tests + `terraform plan` |
| Security | gitleaks, `bun audit` (blokir critical), CodeQL/trivy | gitleaks, trivy config |
| Integration | Storybook build / Rspress build | **apply ke realm ephemeral + verifikasi kontrak OIDC** |
| Governance | gate changeset (kontrak publik design system) | — |
| Release/Deploy | changesets → npm (OIDC provenance) / Pages; deploy app bergerbang security scan | apply dev otomatis; staging & prod via approval environment |
| Drift | — | plan `-detailed-exitcode` mingguan → issue otomatis |
| Konsistensi org | `check-consistency` mingguan: CODEOWNERS/services.yaml/clients ↔ `repos.yaml` | sama |

**Kuncinya:** (1) perubahan IAM selalu melewati CI (policy tests) **dan** uji
integrasi realm ephemeral sebelum menyentuh environment nyata
(`identity-platform/scripts/integration-test.sh`); (2) **merge tanpa CI hijau
tidak mungkin** — setiap repo mengunci `required_checks`-nya melalui branch
protection as code, termasuk check keamanan untuk repo high/critical.

## 7. Versioning & kontrak

- **Design system**: Semantic Versioning via Changesets, **ditegakkan mesin** —
  CI menolak PR yang menyentuh kontrak publik tanpa changeset; nama expose MF
  mengikuti versi paket yang sama. Breaking hanya pada `major`; rilis
  dipublikasikan dengan provenance (npm Trusted Publishing — tanpa token).
- **Kebijakan token berversi**: `token-policy.json` memuat `version` (semver) +
  `effectiveDate`; perubahan nilai wajib menaikkan versi, sehingga perilaku
  token dapat dikaitkan ke versi kontrak oleh konsumen.
- **Kontrak integrasi publik** (OIDC flow, endpoint, kebijakan token, cara
  konsumsi design system) hidup di `platform-docs`, dirujuk dari katalog
  (`services.yaml`), direview code owner tim pemilik layanan, dan **validasinya
  otomatis**: link kontrak mati menggagalkan CI, drift kepemilikan tertangkap
  `check-consistency` mingguan.
- **Registry kepemilikan** (`repos.yaml`) adalah kontrak organisasi: perubahan =
  PR + review Platform Engineering; `required_checks` di dalamnya mengikat nama
  job CI setiap repo.

## 8. Keamanan & akses

| Kendali | Mekanisme |
| --- | --- |
| Secret tidak di repo | gitleaks di semua PR, push protection, injeksi via GitHub Environment / secret manager; publikasi npm tanpa token (OIDC); token baca lintas repo (`ORG_READ_TOKEN`) pun disimpan sebagai secret + read-only |
| Admin Console berbasis kebutuhan | Default tanpa akses; role auditor/admin di kode; break-glass four-eyes + MFA; audit events aktif; **review kuartalan otomatis** (issue berjadwal, tidak terlewat tanpa jejak) |
| Perubahan berisiko | Branch protection as code: PR + code owner review + **required checks yang benar-benar terisi** (build/test/security; IAM + integration test) |
| Deploy | Deployment app bergantung pada build + security scan hijau, artefak diverifikasi sebelum deploy |
| Audit | Terraform apply tercatat di Actions; admin events realm diekspor mingguan; drift mingguan → issue |

## 9. Interaksi tim (self-service)

Tim aplikasi dapat melakukan **tanpa menunggu tim platform**:

1. Scaffold aplikasi dari `app-template` (± 5 menit).
2. Mendaftarkan client OIDC dengan **generator** (`bun run new-client …`) yang
   langsung menghasilkan konfigurasi lolos kebijakan → PR ke `clients/`
   (SLA 2 hari, otomatis divalidasi CI + realm ephemeral).
3. Memakai design system (npm atau MF) dan meng-upgrade via PR Dependabot;
   versi kontrak jelas (changeset wajib, MF mengikuti versi paket).
4. Memakai pipeline CI/CD reusable tanpa menulis YAML dari nol.
5. Menambah repo baru: PR ke `repos.yaml` → repo + tim + branch protection +
   required checks dibuat otomatis.

Platform Engineering hanya terlibat **facilitating** (time-boxed) untuk:
onboarding tim baru, adopsi pola baru, atau perbaikan alur yang gagal.

## 10. Metrik keberhasilan

| Metrik | Target |
| --- | --- |
| Lead time onboarding app baru | ≤ 1 hari (vs hitungan minggu sebelumnya) |
| Waktu review registrasi client | ≤ 2 hari kerja (p95) |
| Perubahan IAM yang gagal di prod | 0 (gerbang ephemeral + approval) |
| Secret terdeteksi di repo | 0 (gitleaks blokir) |
| Repo tanpa CI | 0 (divalidasi registry) |
| Repo dengan `required_checks` kosong | 0 (divalidasi registry + branch protection) |
| Pelanggaran konsistensi lintas repo | 0 pada pemeriksaan mingguan |
| PR design system rilis dengan provenance | 100% |
| PR kontrak publik design system tanpa changeset | 0 (gate CI blokir) |
| Drift IAM tidak tercatat | 0 pada review mingguan |
| Review akses Admin Console terlewat | 0 (issue kuartalan otomatis) |

## 11. Roadmap evolusi (topology sebagai living architecture)

| Fase | Fokus | Pemicu untuk fase berikutnya |
| --- | --- | --- |
| 1 (sekarang) | Polyrepo + pipeline + kontrak dasar | Bila review registrasi client menjadi bottleneck |
| 2 | Otomasi onboarding client penuh (form/generator), katalog Backstage-style | Bila jumlah repo > 15 atau tim > 4 |
| 3 | Revisi boundary berbasis sinyal (waktu tunggu, gesekan PR, on-call) | Sesuai sensing kuartalan |

## 12. Keputusan arsitektur (ADR)

| ADR | Keputusan |
| --- | --- |
| [ADR-0001](docs/adr/ADR-0001-polyrepo-team-topologies.md) | Polyrepo per batas tim (bukan monorepo) |
| [ADR-0002](docs/adr/ADR-0002-keycloak-config-as-code.md) | Keycloak sebagai IdP dengan config-as-code Terraform |
| [ADR-0003](docs/adr/ADR-0003-npm-trusted-publishing.md) | Rilis design system via Changesets + npm Trusted Publishing (tanpa token) |
| [ADR-0004](docs/adr/ADR-0004-repos-registry-and-reusable-workflows.md) | `repos.yaml` + reusable workflows sebagai governance terpusat |
| [ADR-0005](docs/adr/ADR-0005-realm-ephemeral-integration-test.md) | Uji konfigurasi IAM di realm ephemeral sebagai gerbang wajib |
| [ADR-0006](docs/adr/ADR-0006-machine-enforced-contracts.md) | Penegakan mesin kontrak organisasi: required checks + konsistensi lintas repo |
