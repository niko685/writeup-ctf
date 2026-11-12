# 911 - NFX CTF

## Recon

Dari deskripsi challenge, diketahui ada sebuah video peristiwa 9/11 yang diunggah seseorang di forum.

Setelah melakukan pencarian, ditemukan video di Reddit pada subreddit r/PublicFreakout.

Username pengunggah video tersebut adalah:

```text
PigParkerPt2
```

## OSINT

Selanjutnya coba cari username tersebut di platform lain. Dari hasil pencarian ditemukan akun GitHub dengan username yang sama.

GitHub:
https://github.com/PigParkerPt2

Di akun tersebut terdapat beberapa repository. Salah satu yg menarik adalah:

```text
xor-operation
```

## Git Investigation

Clone repository tersebut:

```bash
git clone https://github.com/PigParkerPt2/xor-operation.git
cd xor-operation
```

Cek seluruh riwayat commit:

```bash
git log --oneline --all
```

Ditemukan commit yang mencurigakan, yaitu:

```text
df34505 add .env
ea765a9 chore : deleted .env
```

Ternyata file `.env` pernah ditambahkan lalu dihapus pada commit berikutnya.

Untuk melihat isi file sebelum dihapus, gunakan:

```bash
git show ea765a9^:.env
```

Dari hasil tersebut ditemukan konfigurasi server dan flag yang tersembunyi di dalam file `.env`.

## Flag

```text
NFX{OS1NT_DU4R_D34_D0R}
```
