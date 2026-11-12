# insider - NFX CTF

## Deskripsi

Tgl 17 Maret 2025 laptop staf NFX Aerospace disita di Bandara Changi, Singapura, gara2 bocor dokumen proyek Helios. Cuma ada satu artefak: profil Chrome milik tersangka. Dari situ kita disuruh rekonstruksi aktivitasnya dan nyusun flag. Flag-nya baru dikasih web setelah semua pertanyaan bener.

## Bahan

File `evidence_nfx_phantom_insider.tar.gz`, isinya:

```
acquisition_note.txt
ChromeProfile/
  Local State, First Run
  Default/
    History, Bookmarks, Preferences, Secure Preferences
    Login Data, Web Data
    Network/Cookies
    Extensions/
      ghbmnnjooekpmoecnnnilnnbdlolhkhi/1.92.1_0/   (Google Docs Offline)
      mgndgikekgjfcpckkfioiaebcgkldpdb/2.7.1_0/    (ZenVPN)
```

Dari `acquisition_note.txt`: perangkat Dell Latitude 7440 milik Noah Fox, Chrome 138, dan yg penting **semua timestamp di database Chrome itu UTC** (epoch WebKit). Hash `History` di note cocok sama file yg dikasih (sha256 `327a1af0...98be5e`), jadi file ga berubah.

## Cara baca

`History`, `Login Data`, `Cookies`, `Web Data` itu SQLite. Timestamp Chrome = mikrodetik sejak 1601-01-01, jadi konversinya:

```sql
datetime(last_visit_time/1000000-11644473600,'unixepoch')
```

Dump tabel `urls` urut waktu:

```bash
sqlite3 History "select url,title,datetime(last_visit_time/1000000-11644473600,'unixepoch') from urls order by last_visit_time"
```

Bagian yg relevan (UTC, 14 Maret 2025):

| Jam | URL |
|---|---|
| 14:05:12 | Google search `windows uac bypass 2024` |
| 14:08:22 | github.com/hfiref0x/UACME |
| 14:10:05 | nvd.nist.gov/vuln/detail/CVE-2024-21412 |
| 14:12:40 | msrc.microsoft.com ... CVE-2024-21412 |
| 14:16:00 | virustotal.com (upload) |
| 14:17:00 | any.run |
| 14:19:30 | 0x00sec.org |
| 14:20:10 | 0x00sec.org/t/windows-uac-bypass-2024/28471 |

Sisanya (YouTube, Hacker News, Starlink, search wifi Changi T3) itu noise, atau cuma konteks.

## Jawaban

### 1. CVE yg dipelajari

`CVE-2024-21412`

Yg ditanya halaman advisory untuk kerentanan *security feature bypass* di Windows. Satu2nya halaman advisory di history adalah NVD dan MSRC, dua-duanya buat CVE-2024-21412. Bookmark folder `Research` juga nyimpen link NVD CVE ini.

### 2. Waktu lokal server VPN saat NVD pertama kali dibuka

`2025-03-14 22:10:05`

Di `urls`, halaman NVD dikunjungi jam `14:10:05` UTC. Exit node-nya Singapura (UTC+8), jadi tinggal tambah 8 jam:

```
14:10:05 UTC + 8 jam = 22:10:05 (Asia/Singapore)
```

Cek `visits` cuma ada satu kunjungan ke halaman itu, jadi yg ini sekaligus yg pertama.

Catatan: kalo dicek timestamp install ZenVPN di `Preferences` (`first_install_time`), hasilnya `14:14:20` UTC, 4 menit setelah NVD dibuka. Jadi dari artefak sebenernya ga bisa dibuktiin VPN udah nyala pas NVD diakses. Tapi soalnya nganggep lewat VPN, jadi jawaban tetep pake waktu itu.

### 3. Alias di forum 0x00sec.org

`wraithbyte`

Dari `Login Data`:

```sql
select origin_url, username_value, datetime(date_created/1000000-11644473600,'unixepoch'),
       datetime(date_last_used/1000000-11644473600,'unixepoch') from logins;
```

```
https://0x00sec.org/   wraithbyte   2025-03-13 09:12:44   2025-03-14 14:20:10
```

Akun dibuat tgl 13 Maret (daftar), dan terakhir dipake 14:20:10, pas banget sama jam buka thread `windows-uac-bypass-2024`. Itu yg dimaksud "berinteraksi". Akun lain di `Login Data` (`nfox-aero`, `n_fox`, email kantor) beda2 dan bukan buat forum ini. Di `Cookies` jg ada `xf_user` buat `.0x00sec.org`, dibuat 14:20:00, tapi isinya terenkripsi.

### 4. Tools VPN di Chrome

`ZenVPN 2.7.1`

Ada 2 extension di `Default/Extensions/`:

| ID | Nama | Versi | from_webstore |
|---|---|---|---|
| `ghbmnnjooekpmoecnnnilnnbdlolhkhi` | Google Docs Offline | 1.92.1 | true |
| `mgndgikekgjfcpckkfioiaebcgkldpdb` | ZenVPN | 2.7.1 | **false** |

`manifest.json` ZenVPN: punya permission `proxy`, `webRequest`, `<all_urls>`, dan deskripsinya "Lightweight VPN proxy". `background.js` nge-set proxy Chrome ke `sg1.zenvpn.example:8899` (server Singapura), cocok sama soal #2. Dipasang manual (bukan dari Web Store), jam `14:14:20` UTC, dan versinya `2.7.1`.

Jadi format `<Nama> <Versi>` -> `ZenVPN 2.7.1`.

## Nyusun flag

NFX{wraithbyte_cve202421412_20250314_zenvpn271}
```
