# Writeup — Static IV Offline

## Soal

Dikasih 2 file: `challenge.py` sama `output.txt`.

`challenge.py`:

```python
MASTER = b"\x13\x37" * 16
PREFIX = b"user-input-goes-here:"

def get_key(counter):
    return hashlib.sha256(MASTER + counter.to_bytes(4, "little")).digest()[:32]

def encrypt_flag(flag: bytes, iv: bytes, counter: int) -> bytes:
    key = get_key(counter)
    cipher = AES.new(key, AES.MODE_CBC, iv)
    return cipher.encrypt(pad(PREFIX + b"|" + flag, 16))
```

`output.txt`:

```json
{
  "ct_hex": "7739b21e...",
  "iv_hex": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
  "counter": 11,
  "note": "IV statis, key = SHA256(MASTER || counter). MASTER ada di challenge.py."
}
```

## Bug

Semua bahan buat decrypt udah lengkap dikasih:

- `MASTER` bocor langsung di source
- `counter` ada di `output.txt`
- key diturunin deterministik dari `SHA256(MASTER || counter)`, jadi key bisa dihitung ulang persis sama
- IV statis juga dikasih di `output.txt`

Jadi nggak ada rahasia yang beneran disembunyiin — soal ini cuma minta reimplementasi proses enkripsinya dibalik aja.

## Langkah exploit

1. Hitung key: `SHA256(MASTER + counter.to_bytes(4, "little"))`, ambil 32 byte pertama.
2. Ambil IV dari `output.txt` (hex `aa` x16).
3. Decrypt `ct_hex` pake AES-256-CBC dengan key & IV di atas.
4. Unpad hasilnya (PKCS7), buang prefix `user-input-goes-here:|`.

```python
import hashlib
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

MASTER = b"\x13\x37" * 16
counter = 11
ct = bytes.fromhex(ct_hex)
iv = bytes.fromhex("aa" * 16)

key = hashlib.sha256(MASTER + counter.to_bytes(4, "little")).digest()
pt = AES.new(key, AES.MODE_CBC, iv).decrypt(ct)
print(unpad(pt, 16))
```

## Hasil

```
b'user-input-goes-here:|NXCTF{cbc_1v_r3u53_l34k5_k3y5tr34m_b0fbd4d35f80}'
```
