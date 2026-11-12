# Write Up - Makna yg Tidak Diungkapkan (Lirik)


## Deskripsi

Ada yg nulis lirik lagu, tugasnya nemuin makna yg ga diungkapin di lirik itu.
Format flag: `NFX{your_flag_here}`

## Solve

### 1. Cek lirik

Liriknya cuma 4 baris biasa dan secara visual ga ada yg aneh. Tapi pas di copy ke editor, kursor kadang "loncat" di antara kata, tanda ada karakter yg ga keliatan. Lirik nya disimpen ke file `lirik.txt` biar karakter aslinya ga berubah.

### 2. Ekstrak zero-width

Dicari semua karakter zero-width yg nempel di tiap kata:

| Char | Unicode |
|------|---------|
| ZWSP | U+200B |
| ZWNJ | U+200C |
| ZWJ  | U+200D |
| WJ   | U+2060 |

Hasilnya total 176 karakter di 26 grup (25 grup isinya 7 karakter, grup terakhir cuma 1). Ada 4 jenis simbol berarti 1 simbol = 2 bit.

### 3. Decode ke biner

Semua kemungkinan mapping simbol ke 2 bit dicoba (24 permutasi), dibaca per 8 bit. Satu mapping ngehasilin string yg ujungnya `==`, ciri base64:

```
ZWNJ = 00
WJ   = 01
ZWSP = 10
ZWJ  = 11
```

```
KDkvOC8VFBJVFTMeXxMOPgFfNl8JORUEXzYKUxQKEQ==
```

### 4. Base64

```bash
echo 'KDkvOC8VFBJVFTMeXxMOPgFfNl8JORUEXzYKUxQKEQ==' | base64 -d | xxd
```

Hasilnya 31 byte yg masih ga kebaca, kemungkinan masih ada 1 layer lagi.

### 5. XOR

Dicoba XOR pake kata2 yg ada di lirik sbg key. Pake key `falling` hasilnya kebaca:

```python
import base64
b = base64.b64decode('KDkvOC8VFBJVFTMeXxMOPgFfNl8JORUEXzYKUxQKEQ==')
key = b'falling'
print(bytes(x ^ key[i % len(key)] for i, x in enumerate(b)))
```

```
NXCTF{st4y_w1th_m3_1n_th3_d4rk}
```

Hasil dekripsi awalnya `NXCTF{`, bukan `NFX{`, jadi prefix nya diganti sesuai format soal. Isi di dalam kurung kurawal nya ga diubah.

## Flag

```
NFX{st4y_w1th_m3_1n_th3_d4rk}
```

## Alur singkat

lirik -> zero-width (4 simbol, 2 bit per simbol) -> base64 -> XOR key `falling` -> flag
