# My Lovely Freya

## Deskripsi

Ada satu foto yg selalu disimpan, dan bersamanya dititipkan sebuah surat. Setiap malam nama disebut "seratus ribu kali", ada "dua ratus lima puluh enam alasan" utk menunggu. Katanya ada rahasia di balik foto itu, mungkin benar mungkin tidak.

Format flag di soal: `NFX{your_flag_here}`

## File

```
my_lovely_freya.zip
├── freya_sayangkuh.png     (1024x1200, 1.7 MB)
└── surat_untuk_freya.enc
```

```
$ file surat_untuk_freya.enc
openssl enc'd data with salted password

$ xxd surat_untuk_freya.enc | head -1
00000000: 5361 6c74 6564 5f5f ...   Salted__
```

Header `Salted__` = dienkripsi pakai `openssl enc`, jd butuh password yg disembunyikan di PNG.

## Petunjuk dari deskripsi

- "seratus ribu kali" -> PBKDF2 dgn 100000 iterasi
- "dua ratus lima puluh enam" -> AES-256

Jd perintah decrypt-nya kira2: `openssl enc -d -aes-256-cbc -pbkdf2 -iter 100000`.

## Cari password

### 1. Chunk PNG

Parse chunk PNG-nya, cuma ada `IHDR`, `IDAT`, `tEXt`, `IEND`. Chunk `tEXt`:

```
Secret = QXlhbWdHb3Jlbmc0NQ==   ->  AyamgGoreng45
```

Dicoba sbg password, **gagal**. Ini umpan.

### 2. LSB

Bit plane LSB tiap channel dicek, hasilnya cuma noise kompresi gambar, bukan pesan.

### 3. Ukuran IDAT ga cocok

Decompress `IDAT` lalu bandingin dgn ukuran dari header:

```
raw hasil decompress : 4720128 byte
yg diharapkan IHDR   : 1200 * (1024*3 + 1) = 3687600 byte
```

Ada sisa 1032528 byte. Kalau dibagi per baris:

```
4720128 / (1024*3 + 1) = 1536 baris
```

Jd gambar aslinya **1024x1536**, tapi tinggi di `IHDR` sengaja ditulis 1200 supaya bagian bawah kepotong.

### 4. Benerin IHDR

Ubah tinggi jadi 1536 (`0x600`) dan hitung ulang CRC chunk `IHDR`:

```python
import struct, zlib

d = bytearray(open("freya_sayangkuh.png", "rb").read())
ihdr = bytearray(d[16:29])
ihdr[4:8] = struct.pack(">I", 1536)
d[16:29] = ihdr
d[29:33] = struct.pack(">I", zlib.crc32(b"IHDR" + bytes(ihdr)) & 0xffffffff)
open("fixed.png", "wb").write(d)
```

Di bagian bawah gambar yg tadinya kepotong muncul teks:

```
<!HayamGoreng45@/>
```

Itu passwordnya (perhatikan huruf `H`, bukan `A` seperti di tEXt).

## Decrypt

```
openssl enc -d -aes-256-cbc -pbkdf2 -iter 100000 -md sha256 \
  -pass pass:'<!HayamGoreng45@/>' \
  -in surat_untuk_freya.enc
```

Output:

```
NXCTF{FR3Y4_MY_L0V3LY!}
```

## Flag

```
NXCTF{FR3Y4_MY_L0V3LY!}
```
