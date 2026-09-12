# 🚀 Panduan Hardening Pipeline: DevSecOps dari Laptop ke Produksi

> **Kenapa ada di repo ini**: server yang sudah dikeraskan jadi sia-sia kalau yang dikirim ke server itu sendiri berlubang — password nempel di repo, dependency bertahun-tahun belum di-update, image kontainer berisi base OS tua, atau deploy langsung dari laptop tanpa lewat pipeline. Panduan ini menutup **sisi hulu**: menjaga kualitas & keamanan apa pun yang masuk ke server.
> **Target**: developer/tim yang punya repo Git + CI/CD (GitHub Actions, GitLab CI, Jenkins, dsb.) dan deploy ke VPS/server.
> **Panduan terkait**: [Ubuntu/Debian](Panduan-Hardening-VPS-Ubuntu.md) · [Windows Server](Panduan-Hardening-VPS-Windows.md) · [README](README.md)
> **Diadaptasi dari**: artikel *"DevSecOps From Laptop to Production — A Practical Security Pipeline Guide"* (SecurityCipher, 27 Agu 2026), ditambah praktik standar industri & dokumentasi resmi.

---

## 📌 Prinsip Dasar (baca ini dulu)

1. **Security di pipeline = tes tambahan, bukan departemen terpisah.** Unit test bertanya *"apakah ini bekerja?"* — security test bertanya *"bisa disalahgunakan tidak?"*. Keduanya jalan di pipeline yang sama, gagal dengan cara yang sama (merah = stop).
2. **Fail-fast.** Kalau satu check merah, perjalanan berhenti dan kembali ke developer. Tidak ada yang lanjut sendiri.
3. **Shift left.** Semakin ke kiri (makin dekat ke orang yang mengetik kode), semakin murah biaya perbaikan. Bug di IDE = hitungan menit; bug di produksi = insiden + reputasi + biaya besar.
4. **Adopsi bertahap.** Pilih **satu tool per kategori**, pasang sebagai gate, baru tambah berikutnya setelah yang pertama dipercaya. Jangan pasang 20 scanner sekaligus — hasilnya alert fatigue dan tidak ada yang ditindaklanjuti.
5. **Semua contoh tool di panduan ini = contoh, bukan rekomendasi.** Setiap kategori punya opsi open source dan komersial; pilih yang cocok, bukan yang paling banyak disebut.

---

## 🚉 Enam Stasiun Perjalanan Kode

| Stasiun | Fokus check | Contoh tool | Paling sering salah |
|---|---|---|---|
| 01 IDE (laptop) | Rahasia (secret), linter | Gitleaks, pre-commit, SonarQube for IDE | Password di-hardcode "buat tes cepat" |
| 02 GitHub (pull request) | Review, branch protection, secret scanning, SAST/SCA | CodeQL, Dependabot, Secret Scanning | Merge tanpa review (ini checkpoint terpenting!) |
| 03 CI build | SAST, SCA, SBOM, lisensi, secrets | SonarQube, Semgrep, Snyk, Dependency-Check, Syft/Trivy | Build hijau → dianggap aman, padahal 400 dependency belum discan |
| 04 Package (artefak) | Image scan, IaC scan, signing, registry | Trivy, Grype, Checkov, KICS, Cosign, Harbor | Base image 3 tahun belum dipatch; Terraform buka bucket ke internet |
| 05 Staging | DAST, policy, uji logika | ZAP, Burp Suite Pro, Nuclei, OPA/Kyverno | Bagian "terlihat" baik sendiri, tapi bocor saat digabung (user enumeration) |
| 06 Produksi | WAF, runtime, CSPM, SIEM | Cloudflare/AWS WAF, Falco, Gatekeeper, Prowler/Wiz, Splunk/Elastic | Aman bulan lalu ≠ aman sekarang (CVE baru muncul tanpa ada yang sadar) |

### 01. IDE — tempat paling murah untuk salah
- Pasang **pre-commit hook** + **Gitleaks**: commit ditolak kalau ada pola password/key.
- **Linter** (SonarQube for IDE / dulu SonarLint) menandai kode berisiko sambil mengetik.
- Aturan sederhana: **jangan pernah** tempel kredensial asli ke file, walau cuma untuk tes. Pakai `.env` (yang di-gitignore) atau secret manager.

### 02. Pull Request — checkpoint terpenting
- **Wajib review** sebelum merge; wajib check hijau sebelum tombol merge aktif.
- **Branch protection**: larang push langsung ke `main`, larang force-push & rewrite history (biar jejak audit utuh: siapa mengubah apa, kapan).
- **Secret scanning + push protection**: mendeteksi format kredensial yang dikenal, dan memblokir sebelum benar-benar ter-push.
- **CodeQL** (SAST) & **Dependabot** (SCA) jalan otomatis di PR.
- 📌 Catatan lisensi: di **repo publik** semua fitur di atas gratis. Di **repo privat**, CodeQL butuh add-on *GitHub Code Security*, secret scanning/push protection butuh *GitHub Secret Protection* — bukan fiturnya hilang, cuma butuh lisensi.
- GitLab, Bitbucket, Azure DevOps punya padanan untuk hampir semua poin ini — namanya beda, idenya sama.

### 03. CI Build — robot yang tidak pernah lelah
- **SAST** (SonarQube, Semgrep) untuk scan menyeluruh; **SCA** (Snyk, OWASP Dependency-Check) untuk mencocokkan library dengan CVE.
- **SBOM** (Syft/Trivy, format CycloneDX/SPDX): "daftar bahan" — wajib supaya tahu persis apa yang ada di dalam aplikasi saat ada CVE baru.
- **Lisensi** (FOSSA & sejenis): peringatan kalau lisensi library melanggar kebijakan.
- **Secrets manager** (Vault, cloud secret manager): kredensial diberikan saat runtime, bukan disimpan di config.

### 04. Package & Registry — kotak yang kamu kirim
- **Image scan** (Trivy, Grype): cek paket OS *dan* library aplikasi di dalam image — bukan cuma kode kamu.
- **IaC scan** (Checkov, KICS): temukan setelan cloud tidak aman di file Terraform/Kubernetes.
- **Signing** (Cosign/Sigstore): segel artefak supaya perubahan tanpa izin ketahuan.
- **Registry** (Harbor, ECR): simpan artefak + **re-scan terjadwal** (artefak lama bisa jadi rentan karena CVE baru).
- Wajib: **artifact immutable** — satu kali dibangun, itu yang di-deploy. Jangan build ulang di server produksi.

### 05. Staging — boleh berperan jadi attacker
- **DAST** (ZAP, Burp Suite Pro) & **Nuclei**: menyerang aplikasi yang benar-benar jalan.
- **Policy as code** (OPA, Kyverno): aturan perusahaan dieksekusi otomatis, bukan diingat manusia.
- Uji hal yang tidak terlihat oleh scanner: **user enumeration** (pesan error beda untuk "password salah" vs "user tidak ada"), rate limiting login, IDOR, kebocoran data di respons API.

### 06. Produksi — pertanyaannya berubah
Bukan lagi *"apakah kode ini aman?"* tapi *"apakah ada yang buruk sedang terjadi sekarang?"*

- **WAF** (Cloudflare, AWS WAF): blokir trafik jahat sebelum sampai aplikasi.
- **Runtime security** (Falco): alert kalau container berperilaku aneh.
- **Admission control** (Gatekeeper): tolak container yang melanggar policy sejak awal.
- **CSPM** (Prowler, Wiz): scan setelan akun cloud (bucket terbuka, IAM terlalu longgar).
- **SIEM** (Splunk, Elastic): kumpulkan log, alert pola mencurigakan.
- **Dependency monitoring berkelanjutan**: CVE yang terbit hari ini untuk library yang kamu pakai 6 bulan lalu — harus ada yang memberi tahu.

---

## 🧰 Peta Tool per Kategori

| Kategori | Fungsi | Contoh | Stasiun |
|---|---|---|---|
| Secrets detection | Tangkap password/key sebelum ter-commit | Gitleaks, TruffleHog | 01 |
| Commit guard | Tolak commit kalau check lokal gagal | pre-commit | 01 |
| Linter | Tandai kode berisiko sambil mengetik | SonarQube for IDE | 01 |
| SAST | Query kode sumber untuk pola berbahaya | CodeQL, SonarQube, Semgrep | 02–03 |
| SCA | Cocokkan library dengan CVE | Dependabot, Snyk, Dependency-Check | 02–03 |
| Secret scanning | Deteksi format kredensial + push protection | GitHub Secret Scanning | 02 |
| Branch protection | Tidak ada tulis langsung ke main; merge hanya kalau hijau | GitHub / GitLab protected branch | 02 |
| CI engine | Jalankan semua tool otomatis tiap push | GitHub Actions, GitLab CI, Jenkins | 02–06 |
| SBOM | Tulis "daftar bahan" (CycloneDX/SPDX) | Syft, Trivy | 03 |
| Lisensi | Peringatkan lisensi library yang tidak boleh | FOSSA | 03 |
| Secrets manager | Simpan & bagikan kredensial on-demand | Vault, cloud secret manager | 03–06 |
| Image scan | Cek paket OS & library di dalam image | Trivy, Grype | 04 |
| IaC scan | Cari setelan cloud tidak aman | Checkov, KICS | 04 |
| Signing | Segel artefak, deteksi tampering | Cosign | 04 |
| Registry | Simpan artefak + re-scan harian | Harbor, ECR | 04 |
| DAST | Serang aplikasi yang berjalan dari luar | ZAP, Burp Suite Pro | 05 |
| Scanner cepat | Ribuan cek misconfiguration | Nuclei | 05 |
| Policy as code | Aturan perusahaan, otomatis | OPA, Kyverno | 05–06 |
| Admission | Tolak container yang langgar policy | Gatekeeper | 06 |
| WAF | Blokir trafik jahat sebelum ke aplikasi | Cloudflare, AWS WAF | 06 |
| Runtime | Alert perilaku container aneh | Falco | 06 |
| CSPM | Scan setelan akun cloud | Prowler, Wiz | 06 |
| SIEM | Kumpulkan log + alert pola aneh | Splunk, Elastic | 06 |

---

## 🧩 Yang Sering Terlewat (tambahan di luar artikel)

Bagian ini penting dan biasanya tidak dibahas di panduan pipeline pada umumnya:

### 1. Hardening CI/CD itu sendiri
Pipeline adalah mesin dengan akses ke produksi — dia juga target.
- **Pin Actions/plugin ke commit SHA**, bukan tag (`@v4` bisa diubah pemiliknya kapan saja).
- **Least privilege**: `GITHUB_TOKEN`/token CI hanya izin yang perlu; batasi siapa yang boleh menjalankan workflow dari fork.
- **OIDC / kredensial berumur pendek** untuk akses cloud — jangan simpan access key jangka panjang di CI.
- **Jangan pernah** `echo`/print secret ke log (log CI sering bisa dibaca banyak orang).
- **Environment protection**: deploy ke produksi butuh approval manual untuk tim besar.
- Runner self-hosted = server tersendiri → **ikut mengeraskan server-nya** (lihat panduan VPS di repo ini).

### 2. Rantai pasok (supply chain)
- **Pin & lock dependency** (`package-lock.json`, `go.sum`, dsb.) + review saat naik versi.
- Simpan **SBOM per rilis** — saat CVE besar muncul, kamu bisa menjawab "apakah kita terdampak?" dalam hitungan menit.
- **Verifikasi signature** artefak saat deploy (`cosign verify`), bukan sekadar menandatangani.
- Pertimbangkan **provenance/SLSA** untuk membuktikan artefak dibangun oleh pipeline resmi.

### 3. Container
- Base image **minimal/distroless**, **pin digest** (jangan `:latest`).
- Jalankan sebagai **non-root**, root filesystem **read-only**, drop capabilities.
- Scan **saat build** dan **rescan terjadwal** — base image lama = lubang baru.

### 4. Rahasia (secrets)
- Kalau sudah ter-commit: **hapus commit saja tidak cukup** — anggap bocor, **rotasi kredensialnya**.
- Satu secret per environment (dev/staging/prod), rotasi berkala, akses dicatat.
- Cara paling sehat: developer **tidak pernah melihat** secret produksi.

### 5. Kebijakan rilis
- Hanya **artefak hasil pipeline resmi** yang boleh sampai produksi.
- **Dilarang deploy langsung dari laptop** ke produksi — bypass pipeline = bypass semua check di panduan ini.

---

## ✅ Checklist Adopsi Bertahap

**Fase 1 — gratis & cepat (mulai dari sini)**
- [ ] Branch protection: no direct push ke `main`, wajib review, wajib check hijau
- [ ] Secret scanning + push protection aktif
- [ ] Dependabot alerts + update PR aktif
- [ ] pre-commit + Gitleaks di lokal developer

**Fase 2 — dasar yang solid**
- [ ] SAST jalan otomatis di setiap PR/CI (CodeQL atau Semgrep atau SonarQube)
- [ ] SCA sebagai gate (temuan kritis = build merah)
- [ ] SBOM dihasilkan & diarsipkan tiap rilis
- [ ] Image scan di setiap build image

**Fase 3 — artefak & staging**
- [ ] IaC scan (Checkov/KICS) untuk Terraform/K8s
- [ ] Signing artefak (Cosign) + verifikasi saat deploy
- [ ] Registry re-scan harian
- [ ] DAST (ZAP) jalan di staging

**Fase 4 — produksi & kedewasaan**
- [ ] OIDC / kredensial berumur pendek untuk cloud
- [ ] Actions/plugin di-pin ke SHA, token CI least-privilege
- [ ] Runtime detection (Falco) + admission control (Gatekeeper)
- [ ] WAF di depan aplikasi
- [ ] CSPM + SIEM + alert yang benar-benar dibaca

---

## ⚠️ Kesalahan Umum

- ❌ Hanya mengeraskan server, tapi pipeline bebas: secret di repo, dependency tanpa lock
- ❌ Menganggap "build hijau = aman"
- ❌ Gate dinyalakan tapi temuannya tidak pernah ditindaklanjuti (alert fatigue) → mulai dari yang kritis saja, beri tenggat
- ❌ Access key cloud jangka panjang tersimpan di CI (harusnya OIDC berumur pendek)
- ❌ Pin Actions pakai tag (`@v4`) — tag bisa dipindahkan; pakai commit SHA
- ❌ Tidak pernah re-scan artefak lama — CVE baru muncul belakangan
- ❌ Deploy manual dari laptop langsung ke produksi
- ❌ Menyimpan secret SMTP/token di skrip yang ter-commit (termasuk skrip laporan otomatis)
- ❌ Menganggap staging = opsional ("nanti saja kalau sudah besar")
- ❌ Scoping tool terlalu banyak sekaligus → tidak ada yang benar-benar dipakai

---

## 📚 Sumber

- **SecurityCipher** — *DevSecOps From Laptop to Production: A Practical Security Pipeline Guide* (27 Agustus 2026): <https://securitycipher.com/2026/08/27/devsecops-laptop-to-production/>
- GitHub Docs — secret scanning, code scanning (CodeQL), Dependabot, hardening GitHub Actions (OIDC, pinning)
- OWASP — Dependency-Check, ZAP, SAMM; SLSA (provenance); Sigstore/Cosign; dokumentasi resmi tiap tool

---

*Panduan ini diadaptasi & dikembangkan dari artikel sumber di atas, disusun ulang dalam Bahasa Indonesia mengikuti struktur panduan lain di repo ini (placeholder, bebas data internal, siap dibaca publik).*
