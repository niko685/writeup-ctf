# ECDSA Nonce Bias

## Deskripsi

Diberikan 200 signature ECDSA di kurva secp256k1. Nonce `k` ga acak seragam: 8 bit teratas selalu 0, jd `k < 2^248`. Flag dienkripsi AES-GCM dgn key turunan dari private key ECDSA.

## File

- `challenge.py` : ringkasan output + hint
- `output.txt` : `curve`, `n`, `signatures` (`r`, `s`, `h`, `top8_k`, `K_BITS`), `flag_ciphertext`, `pubkey_pem`
- `pubkey.pem` : public key

Hint dari `challenge.py`:

- nonce `k < 2^248`
- ini HNP (Hidden Number Problem), susun lattice, LLL/BKZ, recover `d`
- key AES-GCM = `SHA256(d)`, `d` 32 byte big-endian

## Ide

Persamaan signature ECDSA:

```
s = k^-1 (h + r*d) mod n
=> k = s^-1 * h + s^-1 * r * d mod n
```

Misal `t = r * s^-1` dan `a = h * s^-1`, maka:

```
k_i = a_i + t_i * d  (mod n)
```

`k_i` kecil (< 2^248) sedangkan `n` ~ 2^256, tiap signature bocorin 8 bit info ttg `d`. Dgn banyak signature, `d` bisa dicari lewat lattice (HNP).

## Lattice

Biar `k` simetris di sekitar 0, geser dulu: `a' = a - B/2` dgn `B = 2^248`, jd `k' = k - B/2` ada di `(-B/2, B/2)`.

Basis (dimensi `m+2`, `m` = jumlah signature), semua kolom dikali `n` supaya ukurannya seimbang:

```
baris i (0..m-1) : n^2 di kolom i
baris m          : t_i * n di kolom i, B/2 di kolom m
baris m+1        : a'_i * n di kolom i, n*B/2 di kolom m+1
```

Kombinasi `d * baris_m + baris_(m+1) - sum(c_i * baris_i)` ngasih vektor:

```
( n*k'_0, ..., n*k'_(m-1), d*B/2, n*B/2 )
```

Semua entri-nya ~ `n*B/2`, jd vektor ini pendek dan muncul di hasil LLL. Dari vektor itu `d = v[m] / (B/2)`, tandanya diambil dari `v[m+1]`.

## Catatan percobaan

- Percobaan pertama gagal, kolom `d` salah skala (bobot `B/2` tanpa kali `n`, hasilnya vektor target ga pendek).
- Setelah skala dibenerin, 200 signature + BKZ kelamaan (timeout).
- Pakai 60 signature + LLL aja sudah cukup. Tiap signature bocor 8 bit, jd minimal sekitar 32 signature utk key 256 bit.

## Solver

```python
import json, hashlib
from fpylll import IntegerMatrix, LLL
from cryptography.hazmat.primitives import serialization
from cryptography.hazmat.primitives.asymmetric import ec
from cryptography.hazmat.primitives.ciphers.aead import AESGCM

d = json.load(open("output.txt"))
n = int(d["n"], 16)
sigs = d["signatures"][:60]
m = len(sigs)
B = 1 << 248

t, a = [], []
for s in sigs:
    r, ss, h = int(s["r"], 16), int(s["s"], 16), int(s["h"], 16)
    si = pow(ss, -1, n)
    t.append(r * si % n)
    a.append((si * h - B // 2) % n)

M = IntegerMatrix(m + 2, m + 2)
for i in range(m):
    M[i, i] = n * n
    M[m, i] = t[i] * n
    M[m + 1, i] = a[i] * n
M[m, m] = B // 2
M[m + 1, m + 1] = n * (B // 2)

LLL.reduction(M)

fc = d["flag_ciphertext"]
ct = bytes.fromhex(fc["ct_hex"]) + bytes.fromhex(fc["tag_hex"])
nonce = bytes.fromhex(fc["nonce_hex"])

for row in M:
    v = list(row)
    if abs(v[m + 1]) != n * (B // 2):
        continue
    sgn = 1 if v[m + 1] > 0 else -1
    dd = sgn * v[m] // (B // 2)
    for cand in (dd % n, (-dd) % n):
        try:
            key = hashlib.sha256(cand.to_bytes(32, "big")).digest()
            print(hex(cand))
            print(AESGCM(key).decrypt(nonce, ct, None))
            raise SystemExit
        except SystemExit:
            raise
        except Exception:
            pass
```

Output:

```
0xe59b58bbc7a872a170ef684d62f92bbbe511f3bd5d9ec50814435c3666e1fffa
b'NXCTF{ecds4_hnp_n0nc3_b14s_8b1t_l4tt1c3_a3d2ae498097}'
```

Public key dari `d` ini dicek cocok dgn `pubkey.pem`.

## Flag

```
NXCTF{ecds4_hnp_n0nc3_b14s_8b1t_l4tt1c3_a3d2ae498097}
```
