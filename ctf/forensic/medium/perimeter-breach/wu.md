# Perimeter Breach - NFX CTF

## Deskripsi

Server e-commerce AromaCraft (web01) kena insiden tgl 10 Oktober 2026. Dikasih 3 log: `access.log` (Nginx), `auth.log` (syslog), `webapp.log` (log aplikasi). Zona waktu WIB (+0700). Tugasnya rekonstruksi aktivitas penyerang dari recon sampe persistence, terus jawab 5 pertanyaan.

## Bahan

- `access.log` - 6254 baris
- `auth.log` - 952 baris
- `webapp.log` - 63 baris

Log-nya banyak noise: traffic user biasa, Googlebot, sama brute force SSH dari IP lain. Jadi yg pertama dilakuin cari mana yg beneran relevan.

## Filter noise

Dari `auth.log`, hampir semua barisnya brute force SSH dari 2 IP:

```
91.240.118.172  -> 395 baris
194.26.135.88   -> 491 baris
```

Dua-duanya cuma `Failed password` / `Invalid user`, ga ada satupun `Accepted`. Jadi bukan yg masuk. Di `access.log` ada juga `45.155.205.233` yg nge-scan pake Nikto jam 11:47, tapi semua dpt 404.

Biar ketemu yg beneran sukses, cari `Accepted` di `auth.log`:

```bash
grep -n "Accepted" auth.log
```

```
08:25:40 Accepted publickey for ubuntu     from 36.80.14.22
14:20:03 Accepted password  for deploy     from 185.220.101.47
14:41:27 Accepted publickey for svc_backup from 185.220.101.47
```

`36.80.14.22` itu admin asli (jam 08:25 login SSH, 08:30 login ke webapp). Sisanya dari `185.220.101.47`, IP ini jg yg muncul di `access.log` pake tools gobuster, sqlmap, sama python-requests.

## Timeline

Semua jam WIB, tgl 10 Oktober 2026.

| Jam | Sumber | Kejadian |
|---|---|---|
| 13:02:10 - 13:02:15 | access | Gobuster dari `185.220.101.47`, ketemu `/admin/login.php` (200), `/robots.txt` (200). `/.env`, `/.git`, `/backup` kena 403 |
| 13:15:31 - 13:16:22 | access | sqlmap ke `products.php?id=`. Ada `'` -> 500, `ORDER BY`, `UNION SELECT`, terus blind extraction kolom `password` di tabel `users` |
| 13:41:00 - 13:42:00 | access + webapp | 60x POST ke `/admin/login.php` pake python-requests, semua gagal |
| 13:42:02 | access + webapp | Login `admin` berhasil |
| 13:42:06 - 13:42:17 | access | Buka `dashboard.php` lalu `upload.php` |
| 13:42:55 | webapp | Upload `cache_thumb.php` |
| 13:43:41 - 13:47:02 | access | Eksekusi command lewat webshell |
| 14:20:03 | auth | SSH masuk sbg `deploy` |
| 14:20:17 | auth | Bikin user `svc_backup` masuk grup sudo |
| 14:20:34 | auth | Set password `svc_backup` |
| 14:41:27 | auth | Login SSH sbg `svc_backup` pake publickey |

## Jawaban

### 1. IP penyerang yg berhasil mengeksploitasi server

`185.220.101.47`

IP ini yg nge-gobuster, sqlmap, brute force login admin, upload webshell, sampe login SSH. IP lain di log (`91.240.118.172`, `194.26.135.88`, `45.155.205.233`) cuma scan/brute force dan ga ada yg berhasil masuk.

### 2. Password admin yg berhasil dibobol

`Aroma#Craft2026`

`webapp.log` nyatet tiap percobaan login lengkap sama passwordnya (`pw_attempt`). Ada 60 percobaan gagal dari `185.220.101.47` jam 13:41:00 sampe 13:42:00, lalu percobaan ke-61 berhasil:

```
[2026-10-10 13:42:02] INFO  auth: login ok user=admin ip=185.220.101.47 pw_attempt="Aroma#Craft2026"
```

Cocok sama `access.log`, POST `/admin/login.php` terakhir dari IP itu balikannya 302 (redirect), sebelumnya 200 semua (balik ke form login):

```
13:42:00 POST /admin/login.php 200
13:42:02 POST /admin/login.php 302
13:42:06 GET  /admin/dashboard.php 200
```

Catatan: password yg sama juga muncul di login admin asli jam 08:30 dari `36.80.14.22`. Soalnya aplikasinya nyimpen password plaintext di log.

### 3. Nama file webshell

`cache_thumb.php`

Dari `webapp.log`:

```
[2026-10-10 13:42:55] INFO  upload: file saved name="cache_thumb.php" size=412 by=admin ip=185.220.101.47 mime="image/jpeg"
```

Namanya dibikin kayak file thumbnail, mime-nya `image/jpeg` padahal ekstensinya `.php`. Kemungkinan validasi upload cuma ngecek mime/Content-Type, ekstensi ga dicek. Ukurannya cuma 412 byte, cocok buat webshell kecil.

Di `access.log` file ini dipanggil lewat parameter `c`:

```
13:43:41 GET /uploads/cache_thumb.php?c=whoami
13:43:56 GET /uploads/cache_thumb.php?c=id
13:44:35 GET /uploads/cache_thumb.php?c=uname -a
13:44:49 GET /uploads/cache_thumb.php?c=pwd
13:45:13 GET /uploads/cache_thumb.php?c=ls -la /var/www/html
```

### 4. User Linux yg kredensialnya dicuri lewat webshell

`deploy`

Setelah recon dasar, command-nya mulai dikirim dalam bentuk base64 (`echo <b64>|base64 -d|sh`), kemungkinan buat ngehindarin filter. Kalo didecode:

```
13:45:37  Y2F0IC9ldGMvcGFzc3dk          -> cat /etc/passwd
13:45:49  bHMgLWxhIC9ob21lL2RlcGxveQ==  -> ls -la /home/deploy
13:46:06  Y2F0IC9ob21lL2RlcGxveS8uZW52  -> cat /home/deploy/.env
13:46:31  Y3JvbnRhYiAtbA==              -> crontab -l
```

Dia baca `/etc/passwd` buat liat daftar user, nemu `deploy`, ngecek home-nya, lalu baca `.env` (biasanya isi kredensial). Isi `.env` ga ada di log, tapi buktinya ada di `auth.log` 34 menit kemudian:

```
14:20:03 sshd: Accepted password for deploy from 185.220.101.47 port 45072 ssh2
```

Login SSH pake password (bukan key) dari IP penyerang, jadi password `deploy` itu hasil curian dari `.env`.

### 5. Akun backdoor baru dengan akses sudo

`svc_backup`

Dari `auth.log`, masih di sesi `deploy` yg tadi:

```
14:20:17 sudo: deploy : COMMAND=/usr/sbin/useradd -m -s /bin/bash -G sudo svc_backup
14:20:17 useradd: new user: name=svc_backup, UID=1002, GID=1002, shell=/bin/bash
14:20:17 usermod: add 'svc_backup' to group 'sudo'
14:20:34 sudo: deploy : COMMAND=/usr/bin/chpasswd
14:20:34 chpasswd: password changed for svc_backup
```

Terus penyerang logout (14:21:38) dan balik lagi 20 menit kemudian:

```
14:41:27 sshd: Accepted publickey for svc_backup from 185.220.101.47
```

Login pake publickey, artinya dia sempet masang public key sendiri buat akun itu (ga kecatet di log). Nama `svc_backup` dipilih biar keliatan kayak akun layanan biasa.

## Alur serangan

1. Recon: gobuster nemu `/admin/login.php`
2. SQLi di `products.php?id=` buat dump tabel `users`
3. Brute force login admin (60 gagal, ke-61 masuk)
4. Upload `cache_thumb.php` lewat `upload.php`, bypass pake mime `image/jpeg`
5. Eksekusi command lewat webshell, ambil `.env` milik `deploy`
6. SSH sbg `deploy`, bikin `svc_backup` (sudo) buat persistence, balik lagi pake publickey

## Flag

```
NFX{P4th_1nt0_Pr0duct10n}
```
