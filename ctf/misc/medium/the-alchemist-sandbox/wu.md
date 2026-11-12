# The Alchemist's Sandbox

## Deskripsi

Sang alkemis nyimpen formula rahasia di ruang penyimpanan ajaib. Ruangannya dikunci: angka, tanda kutip, dan "esensi tersembunyi" dilarang. Kalaupun rahasianya bocor, formulanya masih terkunci 32 putaran mantra.

## File

- `chall.py`

Isinya 3 bagian:

1. `_VAULT_STATE` : list 21 angka 16-bit, ini flag yg sudah dienkripsi
2. `_encrypt_vault` : cipher 32 putaran
3. `verify_ast` + `run_sandbox` : sandbox eval

## Sandbox

Batasan di `verify_ast`:

- ga boleh digit dan tanda kutip (`0-9`, `"`, `'`)
- panjang maks 1024
- node AST yg diblok: `Import`, `ImportFrom`, `Lambda`, comprehension, `FormattedValue`, `JoinedStr`
- atribut yg diawali `_` diblok
- call maks 2 argumen posisi, tanpa keyword

Env eval cuma punya `getattr`, `chr`, dan `als` (= `_encrypt_vault`). Jd jalur yg dimaksud kemungkinan lewat `getattr` + `chr` buat bangun nama atribut tanpa kutip/digit, lalu bocorin state di remote.

Di soal ini `_VAULT_STATE` sudah ada langsung di `chall.py`, jd bagian sandbox **ga saya pakai**. Langsung balik cipher-nya.

## Cipher

```python
def _encrypt_vault(flag_bytes):          # 42 byte
    w = list(struct.unpack(">21H", flag_bytes))
    for r in range(32):
        rc = ((r * 0x9E37) + 0x7F4A) & 0xFFFF
        for i in range(21):
            prev = w[(i - 1) % 21]
            mix = _rol16((prev * 257 + rc + i * 17) & 0xFFFF, (r + i) % 15 + 1)
            w[i] = (w[i] ^ mix) & 0xFFFF
            w[i] = _rol16((w[i] + rc) & 0xFFFF, 5)
        w = [w[(i * 4) % 21] for i in range(21)]
    return w
```

Semua operasinya bisa dibalik:

- **Permutasi** `w[(i*4) % 21]`: 4 dan 21 saling prima, jd bijektif. Baliknya: `old[(i*4) % 21] = new[i]`.
- **`rol16(x + rc, 5)`**: balik dgn `ror16` 5 lalu kurangi `rc` (mod 2^16).
- **XOR `mix`**: `mix` cuma bergantung ke `prev` (`w[i-1]`). Kalau `i` dibalik dari 20 ke 0, saat membalik `i`, nilai `w[i-1]` masih dlm keadaan yg sama seperti waktu enkripsi (utk `i = 0`, `w[20]` sudah dibalik, dan memang saat enkripsi `w[20]` belum diupdate di putaran itu). Jd `mix` bisa dihitung ulang persis.

Putaran dibalik dari `r = 31` sampai `0`.

## Solver

```python
import struct

STATE = [29406, 62193, 3550, 51604, 54474, 58556, 5807, 62536, 3004, 10962,
         33499, 52037, 51010, 45605, 6009, 18571, 21884, 35367, 29761, 57118, 51549]

rol = lambda v, s: ((v << s) & 0xFFFF) | (v >> (16 - s))
ror = lambda v, s: ((v >> s) | (v << (16 - s))) & 0xFFFF

w = list(STATE)
for r in range(31, -1, -1):
    rc = ((r * 0x9E37) + 0x7F4A) & 0xFFFF
    o = [0] * 21
    for i in range(21):
        o[(i * 4) % 21] = w[i]
    w = o
    for i in range(20, -1, -1):
        x = (ror(w[i], 5) - rc) & 0xFFFF
        prev = w[(i - 1) % 21]
        mix = rol((prev * 257 + rc + i * 17) & 0xFFFF, (r + i) % 15 + 1)
        w[i] = x ^ mix

print(struct.pack(">21H", *w))
```

Output:

```
b'NFXCTF{pyj41l_4st_4lch3my_vm_pwn_d84f216a}'
```

Dicek balik: `_encrypt_vault(flag) == _VAULT_STATE` -> `True`.

## Flag

```
NFXCTF{pyj41l_4st_4lch3my_vm_pwn_d84f216a}
```
