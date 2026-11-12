# Broken Signature Scheme

## Deskripsi

Tantangan crypto 3 layer: signature RSA yg lemah, modulus RSA dgn faktor yg berdekatan + partial key exposure, lalu flag yg dienkripsi AES-GCM.

## File

- `challenge.py` : cuma ringkasan parameter, tanpa solusi
- `output.txt` : `N`, `e`, `leak`, `valid_sigs`, `target_msg_hex`, `flag_cipher`

Isi `challenge.py` menjelaskan skemanya:

- `sig(m) = m^d mod N` (tanpa hash)
- `q = nextprime(p + 2^300)`, leak 20 bit tengah `p`
- Flag: AES-GCM, key = `SHA256(p || q)`

## Layer 1 : Signature multiplikatif

Karena `H` identitas, signature-nya homomorfik:

```
sig(m1) * sig(m2) = (m1*m2)^d = sig(m1*m2)  (mod N)
```

Jadi dari `valid_sigs` kita bisa bikin signature utk pesan baru yg merupakan hasil kali pesan2 yg sudah ada. Semua signature di `output.txt` sudah dicek valid dgn `pow(sig, e, N) == msg`.

Utk soal ini signature forge-nya ga wajib, krn jalur lewat layer 2 langsung ngasih `p` dan `q`, jd `d` bisa dihitung sendiri.

## Layer 2 : Faktor berdekatan

`q = nextprime(p + 2^300)` artinya `q - p` cuma ~300 bit, padahal `p` ~1024 bit. Selisih sekecil ini bikin **Fermat factorization** langsung jalan:

```
a = ceil(sqrt(N))
b^2 = a^2 - N   -> cari sampai b^2 kotak sempurna
p = a - b, q = a + b
```

Dgn `(q-p)^2 / (8*sqrt(N))` jauh < 1, loop-nya selesai di iterasi pertama.

Cek hasil:

- `p * q == N`
- `(q - p).bit_length() = 301`
- 20 bit di posisi 500 dari `p` = `430674` = `0x69252`, sama dgn `leak`

Jadi leak 20 bit-nya sebenarnya ga kepake (bisa juga lewat lattice/Coppersmith, tapi Fermat lebih simpel).

## Layer 3 : AES-GCM

Key = `SHA256(p || q)` dgn `p`, `q` big-endian tanpa padding. Ciphertext = `ct_hex || tag_hex`, nonce dari `nonce_hex`, AAD kosong.

## Solver

```python
import json, math, hashlib
from cryptography.hazmat.primitives.ciphers.aead import AESGCM

d = json.load(open("output.txt"))
N = int(d["N"], 16)

a = math.isqrt(N) + 1
while True:
    b2 = a*a - N
    b = math.isqrt(b2)
    if b*b == b2:
        break
    a += 1
p, q = a - b, a + b
assert p * q == N

raw = p.to_bytes((p.bit_length()+7)//8, "big") + q.to_bytes((q.bit_length()+7)//8, "big")
key = hashlib.sha256(raw).digest()

fc = d["flag_cipher"]
ct = bytes.fromhex(fc["ct_hex"]) + bytes.fromhex(fc["tag_hex"])
print(AESGCM(key).decrypt(bytes.fromhex(fc["nonce_hex"]), ct, None))
```

Output:

```
b'NXCTF{mult1_l4y3r_l4tt1c3_r54_s1gn4tur3_hnp_8c211ad6b5a6}'
```

## Flag

```
NXCTF{mult1_l4y3r_l4tt1c3_r54_s1gn4tur3_hnp_8c211ad6b5a6}
```
