# The Ghost Operator (OSINT - 360 pts)


Dikasih file `investigation.zip`, isinya katanya repo investigasi Dimas Wirawan di NetMetrics. Ceritanya dia operator inti Velox Phantom yg lagi dievakuasi ke safehouse, dan kita disuruh lacak identitas digital dari tanda tangan di repo itu buat nemu lokasi terakhirnya.

## Recon Awal

Extract zip-nya, isinya cuma repo git kecil sama satu file `IR_MEMO.md`. Gak ada branch/tag/stash lain, history juga cuma 1 commit. Jadi fokusnya langsung ke commit itu.

```
git log --format=fuller --show-signature
```

Commit-nya ternyata ditandatangani GPG. Pas dicek pake `git cat-file` ketemu signature packet-nya, dan dari situ keluar fingerprint:

```
B17171D40DE252F193E79707E6BE71CA2BD84A4E
```

Key ID-nya `E6BE71CA2BD84A4E`.

## Cari Pemilik Key

Fingerprint doang gak ngasih info apa2 kalo cuma di-google. Jadi langsung aja tarik public key-nya dari keyserver:

```
gpg --keyserver hkps://keyserver.ubuntu.com --recv-keys B17171D40DE252F193E79707E6BE71CA2BD84A4E
gpg --list-keys --with-fingerprint B17171D40DE252F193E79707E6BE71CA2BD84A4E
```

Hasilnya UID-nya kebaca:

```
uid  Dimas Wirawan (rentry.co/ghost-route-0914) <ghost.operator.rentry@veloxphantom.net>
```

Nah ini dia identitas digitalnya. Ada link rentry di comment UID-nya, `rentry.co/ghost-route-0914`.

## Buka Rentry

Buka link itu, isinya log evakuasi si operator. Intinya:

- Rute dari Jakarta via Tol Cipularang ke Bandung Barat
- Exit Pasteur, lewat Jl. Sersan Bajuri ke arah Parongpong-Lembang buat hindarin kamera ETLE
- Safehouse-nya disebut komplek villa di kaki Gunung Burangrang, Jl. Kolonel Masturi KM 9, Parongpong, Lembang
- Dikasih ciri lagi: villa bertema taman bunga, daerah pegunungan dingin berkabut, sinyal lemah, dan katanya salah satu komplek villa terbesar di Parongpong-Lembang

## Cocokin ke Lokasi Asli

Dari ciri2 di atas (alamat Jl. Kolonel Masturi KM 9 + tema taman bunga + komplek terbesar di Parongpong-Lembang), langsung ngarah ke **Villa Istana Bunga** di Lembang. Cek alamat resminya emang persis sama (Jl. Kolonel Masturi No. KM 9, Parongpong, Bandung Barat), dan beberapa listing nyebut ini salah satu komplek villa terbesar di kawasan itu (puluhan hektar, ratusan unit).

## Flag

```
NFX{villa_istana_bunga_lembang}
```

## Ringkasan Alur

1. Repo git -> commit ditandatangani GPG
2. Ambil fingerprint dari signature packet commit
3. Tarik public key dari keyserver -> UID bongkar link rentry
4. Buka rentry -> dapet ciri2 lokasi safehouse
5. Cocokin ciri2 itu ke tempat asli -> Villa Istana Bunga Lembang
