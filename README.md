# 🛡️ Panduan Hardening VPS

Kumpulan panduan hardening yang **terkoreksi & terpadu** — disusun ulang dari sumber publik dan dokumentasi resmi (1VPS.com, RunCloud, Microsoft Learn, CIS Benchmarks, SecurityCipher), dengan duplikasi dihilangkan dan kesalahan umum diperbaiki.

Cakupannya dua sisi: **server** (OS yang menjalankan aplikasi) dan **pipeline** (perjalanan kode dari laptop sampai ke server itu).

## 📚 Daftar Panduan per OS

| Panduan | Target | Status |
|---|---|---|
| [Panduan-Hardening-VPS-Ubuntu.md](Panduan-Hardening-VPS-Ubuntu.md) | Ubuntu/Debian (22.04/24.04 LTS) | ✅ Selesai |
| [Panduan-Hardening-VPS-Windows.md](Panduan-Hardening-VPS-Windows.md) | Windows Server 2019/2022/2025 | ✅ Selesai |
| _(panduan OS lain)_ | rencana: Rocky/Alma, FreeBSD, dll. | ⏳ Menyusul |

> Konvensi penamaan: `Panduan-Hardening-VPS-<OS>.md` — setiap panduan berdiri sendiri (self-contained), jadi bisa dibaca tanpa panduan lain.

## 🚀 Panduan Pendamping (Pipeline & Rantai Pasok)

| Panduan | Target | Status |
|---|---|---|
| [Panduan-Hardening-Pipeline-DevSecOps.md](Panduan-Hardening-Pipeline-DevSecOps.md) | Pipeline CI/CD: IDE → Git → CI → artefak → staging → produksi (GitHub Actions, GitLab CI, Jenkins, dll.) | ✅ Selesai |

> Server yang sudah dikeraskan tetap bisa bobol kalau yang dikirim ke server itu sendiri berlubang: password nempel di repo, dependency jadul, image kontainer berisi base OS tua, atau deploy langsung dari laptop. Panduan pipeline menutup sisi hulu-nya.

## 🧭 Prinsip yang Berlaku di Semua Panduan

1. **Patch rutin & otomatis** — update keamanan tidak boleh menunggu manusia ingat.
2. **Jangan pakai akun superuser untuk aktivitas harian** — root/Administrator bawaan dimatikan, pakai akun khusus + privilege yang dibatasi.
3. **Kunci akses jarak jauh** — autentikasi kuat (SSH key / NLA), port default diganti, proteksi brute-force (fail2ban / account lockout), firewall membatasi sumber IP.
4. **Minimize attack surface** — service, protokol, dan port yang tidak dipakai dimatikan (SMBv1, Print Spooler, cups, avahi, dll.).
5. **Firewall default-deny** — blok semua koneksi masuk, buka hanya yang dibutuhkan.
6. **Deteksi dini** — audit log aktif, laporan berkala, file integrity monitoring / event log.
7. **Backup off-server** — incremental + pernah di-restore-test.
8. **Rahasia tidak pernah di repo** — pakai secret manager, rotasi kalau bocor (hapus commit saja tidak cukup).
9. **Uji di staging dulu** — terutama langkah yang bisa memblokir aplikasi sah (MAC/AppLocker, enforce policy).
10. **Selalu siapkan jaring pengaman** — console VPS dari panel provider, kalau terjadi self-lockout.
11. **Isi bebas data internal** — semua panduan memakai placeholder (`SSH_PORT`, `IP_ANDALAN`, `ADMIN_USER`, dll.), aman untuk repositori publik.

## ✏️ Filosofi Koreksi di Panduan Ini

Panduan keamanan yang beredar di internet sering punya masalah yang sama — versi di repo ini sudah diperbaiki:

- **Urutan langkah anti self-lockout** — key/akun baru diverifikasi dulu, baru akses lama dimatikan.
- **Bug cron `/etc/cron.d/`** (baris tanpa kolom user tidak pernah jalan oleh cron).
- **Kontradiksi antar-langkah** (misal: menghapus mail server tapi tetap mengharapkan laporan email).
- **Perintah yang diam-diam tidak bekerja** (`sysctl -p` tidak membaca `/etc/sysctl.d/`).
- **Duplikasi antar-sumber** dihilangkan, variabel disatukan di awal dokumen.

## 🚀 Cara Pakai

1. Pilih panduan sesuai kebutuhan (lihat tabel di atas).
2. Baca seluruh panduan sebelum eksekusi.
3. Ganti nilai placeholder di bagian atas sesuai kebutuhan.
4. Kerjakan berurutan — jangan lompat-lompat, terutama bagian akses jarak jauh.
5. Terapkan di staging dulu untuk lingkungan produksi.
6. Untuk hasil terbaik: **kedua sisi sekaligus** — server dikeraskan *dan* pipeline-nya dijaga.

## ⚠️ Disclaimer

Panduan-panduan ini ditujukan untuk versi OS/stack yang tercantum di masing-masing judul — sesuaikan perintah untuk versi/distribusi lain. Penulis tidak bertanggung jawab atas lockout, kerusakan, atau gangguan layanan akibat penerapan tanpa pemahaman.

## 📄 Lisensi

MIT — lihat [LICENSE](LICENSE).
