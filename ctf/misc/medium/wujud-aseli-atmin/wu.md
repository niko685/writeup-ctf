# Write Up - Foto Dokumentasi (Forensics / Stego)

## Deskripsi

> Foto dokumentasi dari event Oktober kemarin. Boleh dilihat, tapi jangan berhenti di situ doang!

File yg dikasih: `dokumentasi1.png` (1280x960, RGB)

## Solve

### 1. Cek file

Fotonya cuma foto rame2 di event National Cybersecurity Connect 2026, ga ada apa2 yg keliatan. Hint di deskripsi suruh jangan berhenti di "dilihat" aja, jadi lanjut cek struktur filenya.

Chunk PNG nya normal (IHDR, IDAT, IEND), tapi pas dicek ukuran file ternyata ada sisa data setelah `IEND` sebesar 4100 byte.

```python
d = open('dokumentasi1.png','rb').read()
i = d.rfind(b'IEND') + 8
print(len(d) - i)   # 4100
print(d[i:i+4])     # b'PK\x03\x04'
```

Awalan `PK\x03\x04` = signature ZIP, jadi ada zip yg ditempel di belakang PNG.

### 2. Extract zip

```python
open('hidden.zip','wb').write(d[i:])
```

```bash
unzip hidden.zip
```

Isinya 2 file:
- `catatan.txt` -> "Arsip internal. Tidak ada apa-apa di sini." (umpan)
- `scan_07.dat`

### 3. Benerin scan_07.dat

`file scan_07.dat` cuma bilang `data`. Pas dilihat hex nya, 8 byte pertama nol semua, tapi setelah itu ada `0000000d 49484452` (`\r IHDR`). Jadi ini PNG yg signature nya dihapus.

Tinggal dibalikin signature PNG nya:

```python
b = bytearray(open('scan_07.dat','rb').read())
b[:8] = b'\x89PNG\r\n\x1a\n'
open('scan_07.png','wb').write(b)
```

Gambarnya kebuka, isinya cuma gradien warna dgn tulisan "SCAN 07" (400x300).

### 4. LSB

Gambar gradien tapi ada tulisan doang, curiga ada data di LSB. Dicoba ekstrak LSB per channel:

```python
import numpy as np
from PIL import Image

a = np.array(Image.open('scan_07.png').convert('RGB'))
bits = (a[...,0] & 1).flatten()   # channel R
print(np.packbits(bits).tobytes()[:60])
```

Output nya:

```
b'TkZYe05GQ0NfR29lc190b19OQ1NDXzIwMjZ9...'
```

Channel G dan B ga ada hasil yg kebaca, cuma R yg isinya.

### 5. Decode

String nya base64:

```bash
echo 'TkZYe05GQ0NfR29lc190b19OQ1NDXzIwMjZ9' | base64 -d
```

```
NFX{NFCC_Goes_to_NCSC_2026}
```

## Flag

```
NFX{NFCC_Goes_to_NCSC_2026}
```

## Alur singkat

`dokumentasi1.png` -> data setelah IEND (zip) -> `scan_07.dat` (PNG signature rusak) -> LSB channel R -> base64 -> flag

## Tools

- Python 3 (PIL, numpy)
- strings, base64
