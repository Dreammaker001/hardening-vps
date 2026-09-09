# 🔐 Panduan Hardening VPS Ubuntu (Versi Terkoreksi & Terpadu)

> **Target**: Ubuntu 22.04 / 24.04 LTS (Debian-based). Untuk RHEL/CentOS/Alma, perintah `apt` diganti `dnf` dan AppArmor diganti SELinux.
> **Asumsi**: kamu punya akses root awal via SSH + akses **console panel VPS** dari provider (jaring pengaman terakhir kalau terjadi self-lockout).
> **Urutan penting**: 🔑 pasang & verifikasi key SSH **dulu**, baru matikan autentikasi password. Jangan pernah membalik urutan ini.

---

## 📌 Variabel yang dipakai di seluruh panduan

Ganti nilai berikut **sekali** dan pakai konsisten di semua langkah:

| Variabel | Nilai contoh | Keterangan |
|---|---|---|
| `DEPLOY_USER` | `deploy` | User non-root untuk aktivitas harian |
| `SSH_PORT` | `2222` | Port SSH baru (opsional tapi sangat disarankan) |
| `EMAIL_ADMIN` | `admin@email.com` | Untuk laporan logwatch/unattended-upgrades |
| `IP_ANDALAN` | `203.0.113.10` | IP rumah/kantor kamu (untuk ignoreip & firewall) |

> ⚠️ Kalau ganti port SSH, wajib konsisten di **3 tempat**: `sshd_config`, aturan UFW, dan jail fail2ban. Panduan ini sudah menyinkronkannya — tinggal ganti nilainya.

---

## 1. Update Sistem + Auto Security Update (digabung)

```bash
sudo apt update && sudo apt upgrade -y
```

Aktifkan patch keamanan otomatis harian:

```bash
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure -plow unattended-upgrades   # pilih YES
```

Verifikasi:

```bash
cat /etc/apt/apt.conf.d/20auto-upgrades
# Harus berisi:
# APT::Periodic::Update-Package-Lists "1";
# APT::Periodic::Unattended-Upgrade "1";
```

Config tambahan (opsional) di `/etc/apt/apt.conf.d/50unattended-upgrades`:

```
# Auto-reboot JIKA diperlukan — hati-hati di produksi (lihat catatan di bawah)
# Unattended-Upgrade::Automatic-Reboot "true";
# Unattended-Upgrade::Automatic-Reboot-Time "02:00";
```

> ⚠️ **Catatan produksi**: auto-reboot pukul 02:00 bisa mematikan service produksi (misal aplikasi kamu) tanpa jaminan nyala lagi. Pastikan semua service penting punya `Restart=always` di unit systemd-nya, atau biarkan `Automatic-Reboot` false dan reboot manual terjadwal.
>
> 📧 Notifikasi email dari unattended-upgrades **butuh MTA lokal** — lihat Langkah 9 sebelum memutuskan menghapus postfix.

---

## 2. Jangan Pakai Root untuk Aktivitas Harian

```bash
sudo adduser deploy                  # buat password kuat
sudo usermod -aG sudo deploy
su - deploy                          # atau logout, login sebagai deploy
```

Satu typo sebagai root bisa menghancurkan sistem. `sudo` memberi bantalan aman + jejak audit (`journalctl` / auth log).

---

## 3. 🔑 Kunci SSH (Paling Kritis — ikuti urutannya!)

### 3a. Buat key & verifikasi dulu (di laptop lokal)

```bash
ssh-keygen -t ed25519 -C "email@kamu.com"          # di laptop, jangan di server!
ssh-copy-id deploy@IP-SERVER                       # kalau port sudah diganti: ssh-copy-id -p SSH_PORT deploy@IP-SERVER
```

**⚠️ WAJIB: tes login key dari terminal/device kedua** dan pastikan **berhasil tanpa password** sebelum melanjutkan. Belum matikan apa pun.

### 3b. Hardening sshd (pakai drop-in file, bukan edit file utama)

Ubuntu modern otomatis meng-include file di `/etc/ssh/sshd_config.d/`. Buat satu file baru:

```bash
sudo nano /etc/ssh/sshd_config.d/99-hardening.conf
```

Isi:

```
# ==== Hardening SSH ====
Port SSH_PORT                        # ganti: 2222 (opsional, sangat kurangi noise bot)
PermitRootLogin no
PasswordAuthentication no            # hanya setelah key TERVERIFIKASI di 3a!
PubkeyAuthentication yes
KbdInteractiveAuthentication no      # kalau pakai 2FA (Langkah 12), set jadi yes
PermitEmptyPasswords no
MaxAuthTries 3
LoginGraceTime 60
X11Forwarding no
AllowUsers DEPLOY_USER              # hanya user ini yang boleh SSH
ClientAliveInterval 300
ClientAliveCountMax 2
```

> Alternatif satu baris: `echo -e "Port 2222\nPermitRootLogin no\n..." | sudo tee /etc/ssh/sshd_config.d/99-hardening.conf`

### 3c. Validasi & restart dengan urutan AMAN

```bash
sudo sshd -t                          # 1) validasi syntax — harus tidak ada error
sudo systemctl restart sshd           # 2) restart (sesi lama BISA mati, itu normal)
```

**Urutan aman yang benar:**
1. Key terpasang & **terverifikasi bisa login** (3a) ✅
2. `sudo sshd -t` — pastikan tidak ada error syntax
3. Restart sshd — anggap sesi lama bisa terputus
4. Buka **terminal baru**, tes login ke `deploy@IP -p SSH_PORT`
5. Baru setelah sukses, tutup sesi lama

> 🚨 Kalau terkunci: jangan panik, pakai **console dari panel VPS provider**. Jangan pernah mengerjakan langkah ini tanpa tahu cara akses console.
>
> 🎯 Ganti port SSH memang *security-through-obscurity* (tidak menahan serangan terarah), tapi **mengurangi noise botnet hingga ~99%**: log auth bersih & CPU hemat.

---

## 4. Firewall UFW

```bash
sudo apt install ufw -y
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow SSH_PORT/tcp                     # port SSH custom
sudo ufw allow 80/tcp && sudo ufw allow 443/tcp
# OPSIONAL (paling efektif): SSH hanya dari IP kamu
sudo ufw allow from IP_ANDALAN to any port SSH_PORT/tcp
sudo ufw enable
sudo ufw status verbose
```

> ⚠️ **Urutan penting**: aturan `allow` ditulis **sebelum** `ufw enable`. Sesi SSH yang sudah berjalan tidak terputus saat enable (UFW mengizinkan koneksi ESTABLISHED), tapi koneksi **baru** akan diblokir kalau port-nya belum di-allow.
>
> Setiap port terbuka = satu permukaan serangan. Jangan buka port tanpa perlu. Untuk menutup port: `sudo ufw delete allow <port>`.

---

## 5. Fail2ban (Anti Brute-Force)

```bash
sudo apt install fail2ban -y
sudo nano /etc/fail2ban/jail.local
```

```ini
[DEFAULT]
bantime  = 1h
findtime = 10m
maxretry = 5
banaction = ufw
ignoreip = 127.0.0.1/8 ::1 IP_ANDALAN      # ⚠️ penting: IP kamu sendiri, biar tidak kena banned sendiri

[sshd]
enabled  = true
port     = SSH_PORT
maxretry = 3
bantime  = 24h
```

```bash
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd           # cek jumlah banned
```

> `ignoreip` mencegah kamu terkunci sendiri kalau IP dinamis / koneksi tidak stabil. Daftar ban sementara bisa dibersihkan dengan `sudo fail2ban-client set sshd unbanip <IP>`.

---

## 6. Audit & Matikan Service yang Tak Dipakai

Prinsip: **minimize attack surface** — setiap service berjalan = potensi pintu masuk.

```bash
# 1. Lihat semua service berjalan + port yang terbuka
sudo systemctl list-units --type=service --state=running
sudo ss -tulpn

# 2. Matikan yang tidak perlu (contoh: postfix — tapi baca catatan MTA di Langkah 9 dulu!)
sudo systemctl stop postfix
sudo systemctl disable postfix
sudo systemctl mask postfix              # kunci, supaya tidak nyala lewat dependency

# 3. Verifikasi
sudo ss -tulpn | grep postfix            # tidak ada output = berhasil

# 4. Bersihkan paket yang tidak terpakai
sudo apt autoremove --purge -y
```

**Panduan keputusan:**

- ✅ Aman dimatikan di server: `cups`, `bluetooth`, `avahi-daemon`, `postfix`/`sendmail` (hanya jika tidak butuh laporan email — lihat Langkah 9)
- ⚠️ Tergantung kebutuhan: `apache2`/`nginx`, `mysql`/`postgresql` — matikan hanya jika benar-benar tidak dipakai
- ❌ JANGAN dimatikan: `ssh`, `ufw`, `fail2ban`, `systemd-journald`, service sistem inti

> Banyak service "berjalan" sebenarnya socket-activated (mis. `cups.socket`) — matikan unit `.socket`-nya juga: `sudo systemctl disable --now cups.socket cups.service` (kalau ada).

---

## 7. AIDE — Deteksi Perubahan File Sistem

```bash
sudo apt install aide -y
sudo aideinit                              # butuh beberapa menit di server besar
sudo mv /var/lib/aide/aide.db.new /var/lib/aide/aide.db
sudo aide --check                          # jalankan berkala
```

Cek rutin mingguan via cron (format crontab user — tanpa kolom user):

```bash
sudo crontab -e
# tambahkan:
0 6 * * 1 /usr/bin/aide --check
```

> ⚠️ **Setiap kali selesai upgrade resmi** (apt upgrade), update database dulu supaya tidak banjir false positive:
> `sudo aide --update && sudo mv /var/lib/aide/aide.db.new /var/lib/aide/aide.db`
>
> File yang berubah misterius **tanpa** upgrade yang kamu lakukan = tanda awal kompromi.

---

## 8. Kernel Hardening via sysctl (Satu File, Satu Perintah)

Buat **satu** file konfigurasi (jangan pecah ke banyak file agar tidak dobel):

```bash
sudo nano /etc/sysctl.d/99-security.conf
```

```
# ===== ANTI NETWORK ATTACKS (IPv4 + IPv6) =====

# Anti IP spoofing (reverse path filter)
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# Tolak ICMP redirect (anti MITM) — IPv4 & IPv6
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0

# Matikan source routing — IPv4 & IPv6
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0
net.ipv6.conf.default.accept_source_route = 0

# Proteksi SYN flood (DoS)
net.ipv4.tcp_syncookies = 1

# Anti-Smurf (tidak merespons ping broadcast)
net.ipv4.icmp_echo_ignore_broadcasts = 1

# Catat paket spoofed / source route invalid (indikasi serangan)
net.ipv4.conf.all.log_martians = 1
```

Terapkan **semua** file sysctl (bukan hanya `/etc/sysctl.conf`):

```bash
sudo sysctl --system
```

Verifikasi:

```bash
sudo sysctl net.ipv4.tcp_syncookies        # output: net.ipv4.tcp_syncookies = 1
sudo sysctl net.ipv4.conf.all.rp_filter    # output: ... = 1
```

> ⚠️ Jebakan umum: `sudo sysctl -p` **hanya** membaca `/etc/sysctl.conf` — kalau parameter kamu ada di `/etc/sysctl.d/99-security.conf`, perintah itu diam-diam tidak meng-apply apa pun. Selalu pakai `sudo sysctl --system`.

---

## 9. Laporan Log Otomatis (Logwatch) — Versi yang Benar

> **Koreksi penting dari versi asli**: jangan membuat file manual di `/etc/cron.d/` tanpa kolom user (cron akan menolaknya), dan jangan hapus postfix kalau masih mau terima email.

**Instalasi logwatch di Ubuntu/Debian sudah otomatis membuat cron harian** di `/etc/cron.daily/00logwatch` — tidak perlu menulis cron manual:

```bash
sudo apt install logwatch -y
```

Set alamat email penerima:

```bash
sudo nano /etc/logwatch/conf/logwatch.conf
# set:
MailTo = EMAIL_ADMIN
Detail = High
```

**Syarat: harus ada MTA (mail server lokal).** Dua pilihan:

- **Opsi A — install postfix (paling sederhana)**:
  ```bash
  sudo apt install postfix -y      # pilih "Internet Site" atau "Local only"
  ```
  → Konsekuensi: **jangan** hapus postfix di Langkah 6 kalau memilih opsi ini.
- **Opsi B — tanpa postfix**: laporan ditulis ke file, dibaca manual / diambil via script:
  ```bash
  sudo /usr/sbin/logwatch --output file --filename /var/log/logwatch-report.txt --detail high
  ```

> Catatan: serangan biasanya menunjukkan tanda peringatan di log **sebelum** berhasil — laporan harian ini adalah detektor awal yang murah.

---

## 10. Audit User & Permission (rutin: mingguan)

### 10a. Cek UID 0 (harus HANYA root)

```bash
awk -F: '($3 == "0") {print}' /etc/passwd
# Output normal: root:x:0:0:root:/root:/bin/bash  (hanya 1 baris)
```

User lain dengan UID 0 = backdoor/misconfiguration:

```bash
sudo userdel -r <user-mencurigakan>        # hapus, atau:
sudo usermod -u 1001 <user-mencurigakan>   # ubah UID ke bukan 0
```

### 10b. Cek akun tanpa password / akun terbuka

```bash
sudo awk -F: '($2 == "") {print $1 " has no password!"}' /etc/shadow
sudo awk -F: '($2 == "!" || $2 == "*") {print $1 " LOCKED/disabled"}' /etc/shadow
```

> 📖 Cara baca: field ke-2 `/etc/shadow` = hash password.
> - Kosong (`::`) = **bahaya** (bisa login tanpa password) → `sudo passwd <user>` atau `sudo usermod -L <user>`
> - `!` atau `*` = terkunci/tidak bisa login — **normal** untuk akun sistem seperti `daemon`, `bin`, dll. Di Ubuntu akun sistem tidak pernah punya field kosong, jadi contoh "daemon has no password" di banyak artikel adalah menyesatkan.

### 10c. Direktori world-writable tanpa sticky bit

```bash
sudo find / -xdev -type d -perm -0002 -a ! -perm -1000 -print 2>/dev/null
```

- `drwxrwxrwt` (1777) = aman (sticky bit — hanya owner file bisa hapus), contoh: `/tmp`
- `drwxrwxrwx` (777) tanpa sticky = **bahaya** di direktori bersama

Perbaikan:

```bash
sudo chmod 1777 /var/www/uploads     # opsi 1: tambah sticky bit
sudo chmod 750 /opt/app/data         # opsi 2: permission ketat
```

---

## 11. MAC: AppArmor (Ubuntu) / SELinux (RHEL)

**Bedanya dengan chmod (DAC):** chmod = "siapa pun yang punya key bisa masuk rumah"; MAC = "bahkan yang punya key hanya bisa ke kamar tertentu". Kalau nginx di-hack, AppArmor membatasi blast radius: hacker tidak bisa baca `/etc/shadow` atau `/root/.ssh` dari proses nginx.

```bash
sudo aa-status                 # cek status (butuh sudo di Ubuntu modern)
```

Strategi aman: **complain mode dulu → monitor log → baru enforce**:

```bash
sudo aa-complain /etc/apparmor.d/usr.sbin.nginx    # log saja, tidak memblokir
# pantau log beberapa hari, perbaiki policy kalau ada aplikasi yang "menjerit"
sudo aa-enforce /etc/apparmor.d/usr.sbin.nginx     # baru aktifkan strict
sudo systemctl reload apparmor
```

> ⚠️ MAC adalah lapisan terakhir, tapi **paling kompleks** — policy yang salah bisa memblokir aplikasi yang sah (misal nginx tidak bisa baca sertifikat). Jangan di-enforce di produksi tanpa uji di staging dulu. Mulai dari profile bawaan distribusi, jangan menulis policy sendiri di awal.

---

## 12. (Opsional) 2FA untuk SSH

```bash
sudo apt install libpam-google-authenticator -y
google-authenticator                        # jalankan sebagai DEPLOY_USER, simpan kode cadangan!
```

Edit `/etc/pam.d/sshd`, tambahkan **di baris paling atas (sebelum `@include common-auth`)**:

```
auth required pam_google_authenticator.so
```

Edit `/etc/ssh/sshd_config.d/99-hardening.conf`:

```
KbdInteractiveAuthentication yes
AuthenticationMethods publickey,keyboard-interactive
```

```bash
sudo sshd -t && sudo systemctl restart sshd
```

> ⚠️ **Simpan recovery codes** di password manager. HP hilang tanpa cadangan = terkunci (kecuali console panel). Tes dari terminal kedua sebelum menutup sesi lama.

---

## 13. Backup Off-Server (Non-Negotiable)

Hardening tidak menahan hardware failure, kebakaran DC, atau `rm -rf` yang tidak disengaja.

- Backup harus **di luar server** (object storage / VPS lain / NAS)
- Idealnya **incremental + terverifikasi** (restore test berkala!)
- Target: VPS hilang total → bisnis pulih dalam hitungan menit
- Tool yang umum: `restic`, `borgbackup`, `duplicati` — atau solusi panel provider

---

## 14. Monitoring Kesehatan Server

Hardening bukan satu kali jadi — pantau terus:

```bash
# Cek rutin: load, memori, disk
uptime && free -h && df -h
journalctl -u ssh -n 50 --no-pager        # aktivitas SSH
sudo fail2ban-client status sshd           # jumlah IP terban
```

Pasang alert sederhana (CPU/memori/disk) — mis. `netdata`, `uptime-kuma`, atau cron + script cek disk:

```bash
# contoh: alert disk > 85%
df -h / | awk 'NR==2 && $5+0 > 85 {system("echo \"Disk hampir penuh: \"$5 | mail -s \"ALERT DISK\" EMAIL_ADMIN")}'
```

**Tanda server kena hack:** CPU naik tak wajar, proses tak dikenal, file berubah (AIDE menangkap ini), cron janggal, trafik jaringan aneh.

---

## 15. Langkah Lanjutan (Opsional, Level Advanced)

1. **Batasi SSH hanya dari IP kamu** (jauh lebih efektif daripada ganti port):
   ```bash
   sudo ufw allow from IP_ANDALAN to any port SSH_PORT/tcp
   ```
2. **`AllowUsers`** di sshd_config — sudah ada di Langkah 3b, pastikan terisi user yang benar.
3. **Nonaktifkan IPv6** hanya jika infrastruktur kamu tidak memakainya (cek dulu: `ip -6 addr`).
4. **Password manager + password unik** untuk setiap service — jangan pernah pakai password sama.
5. **Jangan lupa aturan outbound** — malware "phone home" lewat koneksi keluar. UFW default `allow outgoing` itu wajar, tapi untuk server sensitif pertimbangkan whitelist outbound (level advanced).
6. **Sinkronisasi waktu (NTP)** — log yang akurat penting untuk forensik: `timedatectl` (systemd-timesyncd biasanya sudah aktif di Ubuntu).

---

## ✅ Checklist Implementasi Akhir

- [ ] `sudo apt update && sudo apt upgrade -y` — sistem up-to-date
- [ ] `unattended-upgrades` aktif (`20auto-upgrades` berisi `"1"`)
- [ ] User `deploy` dibuat, masuk grup `sudo`, bisa login via key
- [ ] Key Ed25519 terverifikasi login **tanpa password**
- [ ] `PermitRootLogin no`, `PasswordAuthentication no`, `AllowUsers deploy` aktif
- [ ] `sudo sshd -t` lolos tanpa error
- [ ] UFW: default deny + port SSH/80/443 terbuka, `status verbose` hijau
- [ ] Fail2ban aktif, `fail2ban-client status sshd` jalan, `ignoreip` terisi IP sendiri
- [ ] Service tidak perlu sudah di-disable (cek `ss -tulpn`)
- [ ] AIDE: db dibuat, `aide --check` pertama bersih, cron mingguan terpasang
- [ ] `/etc/sysctl.d/99-security.conf` ada, `sudo sysctl --system` sukses, parameter terverifikasi
- [ ] Logwatch terpasang + `MailTo` terisi + MTA diputuskan (postfix ATAU output file)
- [ ] Audit user: hanya 1 UID 0, tidak ada shadow kosong, tidak ada world-writable aneh
- [ ] Backup off-server berjalan + pernah di-restore-test
- [ ] (Opsional) 2FA SSH aktif & recovery codes aman
- [ ] Console panel VPS diketahui cara aksesnya (jaring pengaman)

---

## ⚠️ Kesalahan Umum yang Wajib Dihindari

- ❌ Mematikan password auth **sebelum** key SSH terpasang & terverifikasi → self-lockout
- ❌ Restart sshd tanpa `sshd -t` / tanpa tes dari terminal kedua
- ❌ Terlalu banyak port terbuka di firewall
- ❌ Semua hal dikerjakan sebagai root
- ❌ Key SSH lemah (wajib Ed25519 atau RSA 4096+)
- ❌ Password sama untuk semua service
- ❌ Menghapus postfix tapi tetap mengharapkan email logwatch/unattended-upgrades
- ❌ `sysctl -p` untuk file yang ada di `/etc/sysctl.d/` (diam-diam tidak apply)
- ❌ Cron manual di `/etc/cron.d/` tanpa kolom user → tidak pernah jalan
- ❌ Melupakan aturan **outbound** — malware "phone home" lewat koneksi keluar

**Kalau terkunci dari SSH** → jangan panik, gunakan console dari panel VPS provider.

---

## 📚 Sumber

- Complete VPS Security Hardening Guide — 1VPS.com
- Linux Server Hardening: 11 Steps to Secure a Production VPS — RunCloud
- Dokumentasi resmi: Ubuntu Server Guide (AppArmor, ufw), man pages `sshd_config`, `sysctl`, `fail2ban`, `aide`, `logwatch`

---

*Dokumen ini adalah versi terkoreksi & terpadu: duplikasi dihilangkan, bug cron diperbaiki, urutan SSH dibuat aman, dan sysctl disatukan ke satu file.*
