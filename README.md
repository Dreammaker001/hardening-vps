# 🔐 Panduan Hardening VPS Ubuntu

Versi **terkoreksi & terpadu** dari panduan hardening VPS — disusun ulang dari sumber publik (1VPS.com, RunCloud, dokumentasi resmi Ubuntu), duplikasi dihilangkan, dan beberapa kesalahan umum yang sering ditemukan di artikel sejenis diperbaiki.

## 📚 Isi Panduan

1. Update sistem & auto security update (`unattended-upgrades`)
2. User non-root untuk aktivitas harian
3. 🔑 Penguncian SSH — key Ed25519, urutan aman anti self-lockout, drop-in config
4. Firewall UFW
5. Fail2ban (anti brute-force)
6. Audit & minimalisasi service
7. AIDE (deteksi perubahan file sistem)
8. Kernel hardening via sysctl (IPv4 + IPv6, satu file)
9. Logwatch — laporan harian versi cron yang benar
10. Audit user & permission
11. MAC: AppArmor / SELinux
12. 2FA untuk SSH (opsional)
13. Backup off-server
14. Monitoring & tanda-tanda kompromi
15. Langkah lanjutan + checklist implementasi akhir

## ✏️ Yang Dikoreksi dari Versi Asli

- **Bug cron `/etc/cron.d/`** — baris tanpa kolom user tidak akan pernah jalan oleh cron; diganti cron bawaan paket logwatch
- **Kontradiksi hapus postfix vs laporan email** logwatch/unattended-upgrades — kini ada keputusan eksplisit
- **Urutan restart SSH berisiko self-lockout** — key diverifikasi dulu, `sshd -t` sebelum restart, tes dari terminal kedua
- **`sysctl -p` tidak membaca `/etc/sysctl.d/`** — diganti `sudo sysctl --system`
- **Duplikasi langkah antar-sumber dihilangkan**, port SSH disatukan lewat variabel di awal dokumen

## 🚀 Cara Pakai

1. Baca seluruh panduan sebelum eksekusi.
2. Ganti nilai variabel di bagian atas (`SSH_PORT`, `IP_ANDALAN`, `EMAIL_ADMIN`, dll.) sesuai kebutuhan.
3. Kerjakan berurutan — terutama bagian SSH: **pasang & verifikasi key SEBELUM mematikan autentikasi password**.
4. Selalu siapkan akses **console VPS dari panel provider** sebagai jaring pengaman.

## ⚠️ Disclaimer

Panduan ini ditujukan untuk **Ubuntu/Debian (22.04/24.04 LTS)** — sesuaikan perintah untuk distribusi lain. Terapkan di staging dulu untuk lingkungan produksi. Penulis tidak bertanggung jawab atas lockout atau kerusakan akibat penerapan tanpa pemahaman.

## 📄 Lisensi

MIT — lihat [LICENSE](LICENSE).
