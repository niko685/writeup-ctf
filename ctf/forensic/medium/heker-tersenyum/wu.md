# Write Up NFX CTF

Heker Tersenyum - Art Evidence Secret

## Gambar ASCII (Stego / LSB)

**Deskripsi**

> Dia tampak senyum, namun dia menyembunyikan sesuatu dibalik gambar itu. Dia anggota heker dari tim Lampung Siber Brotherhood (LSB). Ada secret yang tidak bisa dia beritahu ke siapapun. Pakaian dia memakai baju berwarna merah dan celana berwarna biru. Perhatikan nama tim dia dan pakaian dia.
>
> Format Flag: `NFX{your_flag_here}`

**File:** `image.png` (1495x2424, RGB)

### Solve

Clue-nya:
- LSB = Least Significant Bit -> stego LSB
- baju merah, celana biru -> kanal **R** dan **B**
- "secret" -> kemungkinan ada key

Cek metadata PNG, ada 2 field:

```
Comment: The smallest details matter.
PhotoLocation: SmFrYXJ0YSBTZWxhdGFuLCAxMzMwLCBSVyAyMCwgUlQgMTAsIFMzVnJkWEpwZFdzNU9RPT0=
```

`PhotoLocation` di-decode base64:

```
Jakarta Selatan, 1330, RW 20, RT 10, S3VrdXJpdWs5OQ==
```

Bagian terakhir di-decode lagi -> `Kukuriuk99`. Ini yg jadi password nanti.

Terus ambil LSB tiap kanal (urutan baris, kiri ke kanan, atas ke bawah). Kanal R dan B dua-duanya diawali magic `CTF1` + 4 byte panjang (big endian):

| kanal | magic | panjang |
|-------|-------|---------|
| R | `CTF1` | 108 |
| B | `CTF1` | 53 |

Isinya acak semua, jadi pasti terenkripsi. Dilihat dari panjangnya, dugaan awal AEAD:

- B = 53 = nonce 12 + ct 25 + tag 16
- R = 108 = salt 16 + nonce 12 + ct 64 + tag 16

Sempet coba XOR pake `Kukuriuk99` dan turunan hash-nya, ga ada yg jadi. Lanjut coba AES-GCM + PBKDF2-SHA256 dgn beberapa jumlah iterasi, dan ketemu di **200000** iterasi untuk payload R:

- salt = `R[0:16]`
- nonce = `R[16:28]`
- ciphertext = `R[28:]`
- key = PBKDF2-HMAC-SHA256(`Kukuriuk99`, salt, 200000, 32 byte)

Hasil dekripsi R:

```
3cb7804019e6c5247aef85e1ca5362575097b9955ae8d1fdda34128fdf3d32bd
```

64 karakter hex = 32 byte, jadi ini key AES-256 buat payload B. Dekrip B pake AES-GCM, nonce = `B[0:12]`, ciphertext = `B[12:]`, key = hex tadi (`bytes.fromhex`). Tag valid dan keluar flagnya.

### Script

```python
import numpy as np
from PIL import Image
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2HMAC
from cryptography.hazmat.primitives import hashes

a = np.array(Image.open('image.png').convert('RGB'))

def payload(ch):
    bits = (a[..., ch] & 1).reshape(-1)
    raw = np.packbits(bits[:8 * 400]).tobytes()
    assert raw[:4] == b'CTF1'
    n = int.from_bytes(raw[4:8], 'big')
    return raw[8:8 + n]

R = payload(0)
B = payload(2)

salt, nonce, ct = R[:16], R[16:28], R[28:]
k = PBKDF2HMAC(hashes.SHA256(), 32, salt, 200000).derive(b'Kukuriuk99')
key = bytes.fromhex(AESGCM(k).decrypt(nonce, ct, None).decode())

print(AESGCM(key).decrypt(B[:12], B[12:], None).decode())
```

**Flag:** `NFX{LSB_3ASY_OR_H4RD_IDK}`
