# 🔐 Panduan Hardening VPS Windows Server (Versi Terkoreksi & Terpadu)

> **Target**: Windows Server 2019 / 2022 / 2025 (Desktop Experience maupun Server Core).
> **Cara eksekusi**: semua perintah dijalankan di **PowerShell sebagai Administrator**. Di Server Core, PowerShell adalah satu-satunya antarmuka utama — panduan ini tetap berlaku penuh.
> **Jaring pengaman**: selalu kenali dulu cara akses **console dari panel VPS provider** (biasanya browser-based / Hyper-V console) — ini penyelamat utama kalau terjadi lockout.
> **Panduan OS lain**: [Ubuntu/Debian](Panduan-Hardening-VPS-Ubuntu.md) · kembali ke [README](README.md)

---

## 📌 Variabel yang dipakai di seluruh panduan

Ganti nilai berikut **sekali** dan pakai konsisten di semua langkah:

| Variabel | Nilai contoh | Keterangan |
|---|---|---|
| `ADMIN_USER` | `opsadmin` | Akun admin baru (pengganti Administrator bawaan) |
| `RDP_PORT` | `3390` | Port RDP baru (ganti dari 3389) |
| `IP_ANDALAN` | `203.0.113.10` | IP rumah/kantor kamu |
| `EMAIL_ADMIN` | `admin@email.com` | Penerima laporan keamanan |

> ⚠️ Kalau ganti port RDP, wajib konsisten di **2 tempat**: registry `RDP-Tcp` dan aturan Windows Firewall. Panduan ini sudah menyinkronkannya.

---

## 1. Update Sistem (Otomatis, Terjadwal)

```powershell
sconfig
# Pilih menu "Windows Update Settings" → atur ke otomatis
# Pilih menu "Download and install updates" → jalankan
# (nomor menu bisa berbeda antar versi Server — baca layarnya)
```

- Patch keamanan: **segera** setelah rilis (jadwalkan jendela maintenance).
- Patch non-kernel umumnya butuh **restart** — di VPS, restart = downtime singkat, jadwalkan di luar jam sibuk.
- Banyak panel provider VPS Windows menyediakan opsi "Install updates & reboot" — bisa dipakai sebagai cadangan.

> ⚠️ Di Windows, sebagian besar patch keamanan **baru aktif setelah restart**. Jangan menunda restart berbulan-bulan — itu alasan utama server Windows kena exploit publik.

---

## 2. Akun Admin Khusus (Jangan Pakai Administrator Bawaan)

Akun `Administrator` (SID-500) adalah target utama brute-force karena namanya sudah pasti diketahui semua orang.

```powershell
# 1. Buat akun admin baru dengan password sangat kuat
net user ADMIN_USER 'P@ssw0rd-Super-Kuat-Ganti-Ini!' /add
net localgroup Administrators ADMIN_USER /add

# 2. Nonaktifkan Guest (default-nya sudah off — pastikan tetap off)
Disable-LocalUser -Name Guest
```

**Urutan aman:**
1. Buat `ADMIN_USER` + masuk grup Administrators ✅
2. **Logout, tes login sebagai `ADMIN_USER`** (RDP atau console) — pastikan bisa ✅
3. Baru nonaktifkan Administrator bawaan:
   ```powershell
   Disable-LocalUser -Name Administrator
   ```

> 🚨 Jangan nonaktifkan `Administrator` **sebelum** berhasil login sebagai akun baru — kalau akun baru bermasalah, kamu terkunci dari RDP. Kalau terlanjur: pakai console provider untuk reset.
>
> Akun `Administrator` bawaan **tidak terkena account lockout** — semakin alasan untuk menonaktifkannya (lihat Langkah 5).

---

## 3. 🔑 Kunci RDP (Paling Kritis — ikuti urutannya!)

RDP di port default `3389` yang terbuka ke internet adalah **sasaran tembak nomor satu** botnet. Jangan biarkan dalam kondisi default.

### 3a. Pastikan NLA aktif (Network Level Authentication)

NLA mewajibkan autentikasi **sebelum** sesi desktop dibuat — memblokir banyak exploit lama dan mengurangi beban brute-force.

```powershell
Set-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' -Name UserAuthentication -Value 1
Set-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server' -Name fDenyTSConnections -Value 0
# Alternatif GUI (Desktop Experience): Server Manager → Local Server → Remote Desktop → "Allow remote connections ... "
```

### 3b. Ganti port RDP (opsional tapi sangat disarankan)

> Seperti ganti port SSH: tidak menahan serangan terarah, tapi **menghilangkan ~99% noise botnet** yang memindai 3389.

**Urutan aman (aturan firewall DULU, baru ganti port):**

```powershell
# 1. Tambah aturan firewall untuk port BARU (sebelum port diganti!)
New-NetFirewallRule -DisplayName 'RDP-Kustom' -Direction Inbound -Action Allow -Protocol TCP -LocalPort 3390 -RemoteAddress IP_ANDALAN

# 2. Ganti port di registry
Set-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' -Name PortNumber -Value 3390

# 3. Restart service RDP — SESI RDP LAMA AKAN TERPUTUS, itu normal
Restart-Service TermService -Force
```

Setelah restart: buka **device/terminal kedua** dan tes koneksi ke port baru (`mstsc /v:IP_SERVER:3390`). Baru setelah sukses, tutup sesi lama.

> 🚨 Sesi RDP kamu akan putus saat `TermService` di-restart. Pastikan: (a) aturan firewall port baru sudah dibuat **sebelumnya**, (b) kamu tahu cara akses console provider sebagai cadangan.
>
> Server Core: aktifkan RDP via `sconfig` → menu Remote Desktop.

---

## 4. Firewall Windows Defender (Default-Deny)

```powershell
# Aktifkan semua profil + blok koneksi MASUK secara default
Set-NetFirewallProfile -Profile Domain,Private,Public -Enabled True -DefaultInboundAction Block -DefaultOutboundAction Allow

# Buka port yang benar-benar dipakai saja
New-NetFirewallRule -DisplayName 'RDP-Kustom' -Direction Inbound -Action Allow -Protocol TCP -LocalPort 3390 -RemoteAddress IP_ANDALAN   # RDP hanya dari IP kamu
New-NetFirewallRule -DisplayName 'HTTP'  -Direction Inbound -Action Allow -Protocol TCP -LocalPort 80
New-NetFirewallRule -DisplayName 'HTTPS' -Direction Inbound -Action Allow -Protocol TCP -LocalPort 443

# Verifikasi
Get-NetFirewallProfile | Select-Object Name,Enabled,DefaultInboundAction
```

> - Sesi RDP yang sedang berjalan **tidak terputus** saat default-inbound diubah ke Block — koneksi yang sudah ESTABLISHED tetap jalan.
> - Setiap port terbuka = satu permukaan serangan. Jangan buka port tanpa perlu.
> - Aturan firewall dibuat **sebelum** mengubah default inbound kalau kamu sedang remote — supaya port RDP baru sudah kebuka duluan.
> - Hapus aturan bawaan "Remote Desktop" (3389) setelah pindah port: `Remove-NetFirewallRule -DisplayName 'Remote Desktop*'` — cek dulu dengan `Get-NetFirewallRule -DisplayName 'Remote Desktop*'`.

---

## 5. Anti Brute-Force RDP (Account Lockout Policy)

Windows tidak punya fail2ban bawaan, tapi punya **Account Lockout Policy** yang efeknya serupa untuk akun lokal:

```powershell
# 5x salah password dalam 15 menit → kunci 15 menit
net accounts /lockoutthreshold:5 /lockoutduration:15 /lockoutwindow:15
net accounts        # verifikasi
```

**Lapisan anti brute-force RDP (kerjakan semua):**

| Lapisan | Status di panduan ini |
|---|---|
| Account lockout policy | ✅ Langkah 5 |
| Akun Administrator bawaan dinonaktifkan (kebal lockout) | ✅ Langkah 2 |
| NLA aktif | ✅ Langkah 3a |
| Port RDP diganti | ✅ Langkah 3b |
| Firewall: RDP hanya dari IP kamu | ✅ Langkah 4 |
| 2FA / VPN gateway | ⚠️ Langkah 12 (opsional, paling kuat) |

> ⚠️ Lockout bisa menjadi *denial of service* kecil untuk user sah yang lupa password — set nilai yang masuk akal (5–10 percobaan). Untuk proteksi lebih kuat, lihat Langkah 12 (RDP di belakang VPN).

---

## 6. Audit & Matikan Service / Fitur yang Tak Dipakai

Prinsip: **minimize attack surface**.

```powershell
# 1. Lihat service berjalan
Get-Service | Where-Object Status -eq 'Running' | Sort-Object Name | Format-Table Name,DisplayName

# 2. 🔴 Print Spooler — matikan kalau server tidak dipakai printer
#    (sumber banyak exploit RCE publik, mis. PrintNightmare)
Stop-Service Spooler -Force
Set-Service Spooler -StartupType Disabled

# 3. 🔴 SMBv1 — protokol jadul, sumber exploit (WannaCry dkk). MATIKAN.
Disable-WindowsOptionalFeature -Online -FeatureName SMB1Protocol -NoRestart

# 4. Wajibkan SMB signing (anti relay attack)
Set-SmbServerConfiguration -RequireSecuritySignature $true -Confirm:$false

# 5. Lihat role/feature terinstall, hapus yang tidak dipakai
Get-WindowsFeature | Where-Object Installed | Format-Table Name
Uninstall-WindowsFeature Web-Server    # contoh: hapus IIS kalau bukan web server — HATI-HATI, baca daftarnya dulu
```

**Panduan keputusan:**

- ✅ Matikan: **Print Spooler** (tanpa printer), **SMBv1**, **Windows Search** (opsional), role/feature yang tidak dipakai
- ⚠️ Tergantung kebutuhan: IIS/Web Server, DNS, DHCP, file server (SMB) — matikan hanya jika benar-benar tidak dipakai
- ❌ JANGAN dimatikan: Windows Defender, Windows Firewall, Windows Update, service sistem inti

---

## 7. Deteksi Intrusi: Defender + Audit Log

### 7a. Microsoft Defender Antivirus (bawaan)

```powershell
# Status real-time protection & signature
Get-MpComputerStatus | Select-Object RealTimeProtectionEnabled,AntivirusEnabled,AntivirusSignatureLastUpdated

# Update signature & jalankan scan cepat
Update-MpSignature
Start-MpScan -ScanType QuickScan
```

### 7b. Aktifkan audit log (untuk deteksi & forensik)

```powershell
# Audit Logon/Logoff (event 4624 sukses / 4625 gagal) — pakai GUID agar jalan di semua bahasa OS
auditpol /set /subcategory:"{0CCE9215-69AE-11D9-BED3-505054503030}" /success:enable /failure:enable
# Audit Process Creation (event 4688)
auditpol /set /subcategory:"{0CCE922B-69AE-11D9-BED3-505054503030}" /success:enable /failure:enable

# Verifikasi
auditpol /get /subcategory:"{0CCE9215-69AE-11D9-BED3-505054503030}"
```

**Event yang wajib kamu kenal (Event Viewer → Windows Logs → Security):**

| Event ID | Arti |
|---|---|
| `4624` | Logon berhasil — perhatikan jam aneh / user tak dikenal |
| `4625` | **Logon gagal** — banjir 4625 = brute-force sedang berlangsung |
| `4688` | Proses baru dibuat — tangkap process injection / malware |
| `4720` | User baru dibuat — tanda backdoor |

Cek cepat percobaan login gagal:

```powershell
Get-WinEvent -FilterHashtable @{LogName='Security';Id=4625} -MaxEvents 10 | Select-Object TimeCreated,Id | Format-Table
```

> Lanjutan (opsional): **Sysmon** dari Microsoft Sysinternals — mencatat proses, koneksi jaringan, dan registry jauh lebih detail. Standard untuk server produksi yang serius.

---

## 8. Hardening Sistem & Registry

Parameter keamanan inti (analog sysctl di Linux) — semuanya aman & tidak merusak fungsi normal:

```powershell
# ===== UAC (pastikan tidak pernah diturunkan) =====
Set-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System' -Name EnableLUA -Value 1
Set-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System' -Name ConsentPromptBehaviorAdmin -Value 5

# ===== Kirim NTLMv2 SAJA (tolak LM & NTLMv1 yang lemah) =====
# Catatan: cek kompatibilitas dengan client/device lama di lingkunganmu
Set-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Control\Lsa' -Name LmCompatibilityLevel -Value 5

# ===== Batasi akses anonim ke SAM/accounts =====
Set-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Control\Lsa' -Name RestrictAnonymous -Value 1
Set-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Control\Lsa' -Name RestrictAnonymousSAM -Value 1

# ===== Matikan LLMNR (anti LLMNR/NBT-NS poisoning attack) =====
New-Item -Path 'HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\DNSClient' -Force | Out-Null
Set-ItemProperty 'HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\DNSClient' -Name EnableMulticast -Value 0
```

Sebagian perubahan baru aktif setelah **reboot** — lakukan di jendela maintenance, lalu tes login ulang.

---

## 9. Laporan Keamanan Harian (Jadwalkan via Task Scheduler)

Kirim ringkasan log gagal login setiap pagi ke email admin.

**1. Buat skrip** `C:\Scripts\SecurityReport.ps1` (sesuaikan SMTP & email):

```powershell
$report = "C:\Logs\security-report-$(Get-Date -Format yyyyMMdd).txt"
New-Item -ItemType Directory -Force -Path C:\Logs | Out-Null

"=== 20 percobaan login GAGAL terakhir (4625) ===" | Out-File $report
Get-WinEvent -FilterHashtable @{LogName='Security';Id=4625} -MaxEvents 20 -ErrorAction SilentlyContinue |
  ForEach-Object { $_.TimeCreated.ToString('yyyy-MM-dd HH:mm') + ' - IP: ' + $_.Properties[18].Value } |
  Out-File $report -Append

"=== 10 logon SUKSES terakhir (4624) ===" | Out-File $report -Append
Get-WinEvent -FilterHashtable @{LogName='Security';Id=4624} -MaxEvents 10 -ErrorAction SilentlyContinue |
  ForEach-Object { $_.TimeCreated.ToString('yyyy-MM-dd HH:mm') + ' - User: ' + $_.Properties[5].Value } |
  Out-File $report -Append

Send-MailMessage -To 'EMAIL_ADMIN' -From 'server@domainmu.com' -Subject 'Security Report Server' `
  -Body (Get-Content $report -Raw) `
  -SmtpServer 'smtp.domainmu.com' -Port 587 -UseSsl -Credential (Get-Credential)
```

**2. Daftarkan sebagai Scheduled Task harian:**

```powershell
$action  = New-ScheduledTaskAction -Execute 'powershell.exe' -Argument '-NoProfile -WindowStyle Hidden -ExecutionPolicy Bypass -File C:\Scripts\SecurityReport.ps1'
$trigger = New-ScheduledTaskTrigger -Daily -At 06:00
Register-ScheduledTask -TaskName 'Security Daily Report' -Action $action -Trigger $trigger -RunLevel Highest
```

> - Skrip memakai `Send-MailMessage` (ada di Windows PowerShell 5.1 bawaan) — butuh akun SMTP yang bisa kirim.
> - Tanpa SMTP? Ubah skrip untuk menulis laporan ke network share, atau kirim manual via `wevtutil qe Security "/q:*[System[(EventID=4625)]]" /c:50 /f:text`.
> - Serangan biasanya menunjukkan tanda di log **sebelum** berhasil — laporan harian ini detektor awal yang murah.

---

## 10. Audit User & Permission (rutin: bulanan)

```powershell
# 1. Semua akun lokal — cek yang Enabled tanpa password / jarang dipakai
Get-LocalUser | Select-Object Name,Enabled,PasswordRequired,PasswordLastSet,LastLogon | Format-Table

# 2. Siapa saja anggota Administrators? (analog "cek UID 0" di Linux)
Get-LocalGroupMember -Group 'Administrators' | Format-Table Name,ObjectClass,PrincipalSource
# Harusnya HANYA: ADMIN_USER + Administrator (yang sudah di-disable) + akun resmi lain

# 3. Share jaringan — matikan yang tidak dikenal
Get-SmbShare | Format-Table Name,Path
Remove-SmbShare -Name 'NamaShare' -Confirm:$false   # kalau ada yang mencurigakan
```

**Yang perlu diperiksa:**
- Anggota `Administrators` = hanya akun yang kamu kenal. Anggota baru yang tidak kamu buat = **backdoor**.
- Akun `Enabled` + `PasswordRequired False` = bisa login tanpa password → `net user NAMA /passwordreq:yes` atau `Disable-LocalUser -Name NAMA`.
- Share tersembunyi administratif (`C$`, `ADMIN$`) itu normal — share bernama aneh yang tidak kamu buat = tanda kompromi.

---

## 11. MAC Level: AppLocker / WDAC (Lanjutan — Hati-Hati)

AppLocker membatasi **aplikasi mana yang boleh jalan** — analog AppArmor/SELinux di Linux. Kalau ada malware, ia tidak bisa mengeksekusi binary sembarangan.

**Alur aman (audit dulu, baru enforce):**
1. `gpedit.msc` → Computer Configuration → Windows Settings → Security Settings → Application Control Policies → AppLocker
2. Buat rule default: izinkan `Program Files`, `Windows`, `Program Files (x86)` + folder aplikasi server kamu
3. Set **Configure rule enforcement = Audit only** → pantau log beberapa minggu
4. Setelah yakin tidak ada aplikasi sah yang terblokir → pindah ke **Enforce rules**

> ⚠️ Ini langkah paling kompleks & paling berisiko memblokir aplikasi sah (aplikasi yang jalan dari folder non-standar, installer, Java/Python, dll). **Jangan enforce langsung di produksi.** Server Core tidak punya `gpedit.msc` — konfigurasi via PowerShell/XML policy (tingkat lanjutan). Alternatif modern: **Windows Defender Application Control (WDAC)**.

---

## 12. 2FA / VPN untuk RDP (Opsional — Paling Kuat)

Jujur dan penting: **di server standalone tanpa Active Directory, 2FA untuk RDP tidak ada di bawaan Windows** — butuh solusi tambahan:

| Opsi | Keterangan |
|---|---|
| **VPN gateway** ⭐ rekomendasi | Pasang WireGuard/OpenVPN di VPS → RDP **hanya** bisa diakses dari dalam VPN. RDP tidak lagi terekspos ke internet sama sekali. |
| Duo / Okta MFA | Proxy autentikasi pihak ketiga (Duo punya free tier) — dipasang di depan RDP |
| RDPGuard & sejenis | Tool pihak ketiga anti brute-force + lockout otomatis (berbayar) |

**Cara paling bersih: RDP tidak pernah terbuka ke internet.** VPN dulu, baru RDP dari dalam.

---

## 13. Backup Off-Server (Non-Negotiable)

Hardening tidak menahan hardware failure, kebakaran DC, atau penghapusan tidak sengaja.

**Opsi A — Windows Server Backup (bawaan):**

```powershell
Install-WindowsFeature Windows-Server-Backup

# Backup harian 02:00 ke network share (ganti target & kredensial)
# -allCritical = volume sistem + boot + system state
wbadmin enable backup -addtarget:\\SERVER-BACKUP\share\nama-server -schedule:02:00 -allCritical -user:BACKUP_USER -password:'PASSWORD' -quiet

# Verifikasi
wbadmin get versions
```

> Catatan: `-allCritical` dan `-include` tidak bisa dipakai bersamaan — untuk volume data tambahan, gunakan GUI Windows Server Backup (Backup Schedule wizard) atau `wbadmin start backup`.

**Opsi B — Veeam Agent for Windows Free** — backup image level, lebih mudah di-restore, populer untuk VPS.

- Backup harus **di luar server** (NAS/network share/object storage) — backup di disk yang sama dengan sistem = bukan backup.
- **Test restore berkala** — backup yang tidak pernah di-restore-test bukan backup, hanya harapan.

---

## 14. Monitoring Kesehatan Server

```powershell
# Cek cepat: status Defender, 10 proses teratas pemakan CPU, disk C:
Get-MpComputerStatus | Select RealTimeProtectionEnabled,AntivirusSignatureLastUpdated
Get-Process | Sort-Object CPU -Descending | Select-Object -First 10 Name,CPU,Id | Format-Table
Get-PSDrive C | Select-Object @{n='UsedGB';e={[math]::Round($_.Used/1GB,1)}},@{n='FreeGB';e={[math]::Round($_.Free/1GB,1)}}

# Koneksi keluar aktif — cari yang mencurigakan (malware "phone home")
netstat -ano | findstr ESTABLISHED
```

**Tanda server kena hack:** CPU naik tak wajar, proses tak dikenal (cek `Get-Process`), banjir event 4625/4624 di jam aneh, akun admin baru (4720), file berubah misterius, trafik keluar aneh. **Kalau terkunci dari RDP** → console dari panel provider VPS (browser-based / Hyper-V console), jangan panik.

---

## 15. Langkah Lanjutan (Opsional)

1. **Matikan TLS 1.0/1.1 & SSL** (wajibkan TLS 1.2+) — cek kompatibilitas aplikasi/client dulu:
   ```powershell
   # Contoh: nonaktifkan TLS 1.0 (ulangi pola yang sama untuk TLS 1.1, SSL 2.0/3.0)
   $base = 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols'
   foreach ($p in 'TLS 1.0','SSL 2.0','SSL 3.0') {
     New-Item -Path "$base\$p\Client" -Force | Out-Null
     New-Item -Path "$base\$p\Server" -Force | Out-Null
     Set-ItemProperty "$base\$p\Client" -Name Enabled -Value 0
     Set-ItemProperty "$base\$p\Server" -Name Enabled -Value 0
     Set-ItemProperty "$base\$p\Client" -Name DisabledByDefault -Value 1
     Set-ItemProperty "$base\$p\Server" -Name DisabledByDefault -Value 1
   }
   ```
2. **RDP di belakang VPN** — rekomendasi terkuat (lihat Langkah 12).
3. **PowerShell ExecutionPolicy**: `Set-ExecutionPolicy RemoteSigned -Scope LocalMachine` — batasi skrip yang bisa jalan.
4. **Credential Guard** — proteksi hash password di memori; butuh dukungan TPM/virtualisasi (banyak VPS cloud tidak mendukung — cek dulu).
5. **BitLocker** untuk volume data — hanya kalau provider VPS mendukung TPM/vTPM.
6. **Sinkronisasi waktu**: `w32tm /config /manualpeerlist:"0.pool.ntp.org,1.pool.ntp.org" /syncfromflags:manual /update` + `Restart-Service w32time` — log yang akurat penting untuk forensik.

---

## ✅ Checklist Implementasi Akhir

- [ ] Windows Update mode otomatis + patch terbaru terinstall + sudah restart
- [ ] Akun `ADMIN_USER` dibuat & **terverifikasi bisa login**; `Administrator` bawaan di-disable; `Guest` off
- [ ] NLA aktif (UserAuthentication = 1)
- [ ] Port RDP diganti + aturan firewall port baru dibuat + sesi baru terverifikasi
- [ ] Firewall: semua profil aktif, default inbound = Block, port 80/443 & RDP-kustom terbuka
- [ ] Account lockout policy aktif (threshold 5 / window 15 / duration 15)
- [ ] Print Spooler mati, SMBv1 mati, SMB signing wajib
- [ ] Defender: real-time ON, signature terbaru, quick scan jalan
- [ ] Audit log aktif (4624/4625/4688) + tahu cara bacanya di Event Viewer
- [ ] Registry hardening diterapkan (UAC, LmCompatibilityLevel, RestrictAnonymous, LLMNR off)
- [ ] Security report harian via Task Scheduler terkirim / tersimpan
- [ ] Audit user: anggota Administrators hanya akun resmi, tidak ada akun tanpa password
- [ ] Backup off-server berjalan (`wbadmin get versions` OK) + pernah di-restore-test
- [ ] (Opsional) RDP di belakang VPN / 2FA
- [ ] Console panel VPS diketahui cara aksesnya (jaring pengaman)

---

## ⚠️ Kesalahan Umum yang Wajib Dihindari

- ❌ RDP port default 3389 terbuka ke internet tanpa proteksi
- ❌ Akun `Administrator` bawaan tetap aktif dengan password lemah
- ❌ Menonaktifkan Administrator **sebelum** akun pengganti terverifikasi → self-lockout
- ❌ Account lockout policy tidak diaktifkan → brute-force bebas jalan
- ❌ NLA dimatikan (beberapa tool lama "butuh" — itu alasan untuk upgrade tool-nya)
- ❌ SMBv1 & Print Spooler tetap jalan (sumber exploit RCE publik)
- ❌ Windows Defender dimatikan / signature tidak pernah update
- ❌ Update ditunda berbulan-bulan tanpa restart
- ❌ Firewall di-disable "biar gampang" — semua port kebuka
- ❌ Banyak akun admin / password sama untuk semua server
- ❌ Backup cuma di disk yang sama dengan sistem
- ❌ Melupakan arah **outbound** — malware "phone home" lewat koneksi keluar

---

## 📚 Sumber

- Microsoft Learn — Security Baselines & dokumentasi Windows Server (RDP, Firewall, Defender, AppLocker)
- CIS Benchmarks — Windows Server (kumpulan parameter hardening standar industri)
- Dokumentasi resmi: `auditpol`, `wbadmin`, `Set-NetFirewallRule`, `Disable-WindowsOptionalFeature`

---

*Panduan ini disusun ulang & dipadukan dari dokumentasi resmi Microsoft dan benchmark keamanan publik. Menggunakan placeholder — bebas dari data internal, aman untuk repositori publik. Versi terkoreksi & terpadu: urutan langkah dibuat aman (anti self-lockout), perintah diverifikasi untuk Server 2019/2022/2025.*
