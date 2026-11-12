# Write-up: Cipher Buatan Vendor Logistik (Crypto)

## Soal

Vendor logistik bikin cipher sendiri utk ngelindungin token gudang dan ngeklaim cipher-nya aman. Dikasih 3 file:

- `cipher.py` : implementasi cipher
- `pairs.txt` : pasangan plaintext/ciphertext
- `flag.enc` : flag yg udah dienkripsi (hex, 52 byte)

Tugasnya buktiin klaim aman itu bener atau nggak.

## Baca source

Cipher-nya Feistel, blok 32 bit, 7 putaran, tiap putaran pakai subkey 16 bit sendiri (total kunci 112 bit).

```python
def round_f(x, k):
    y = x ^ k
    m = SBOX[y & 0xF] | (SBOX[(y>>4)&0xF] << 4) | ...   # S-box per nibble
    return permute(m)                                    # tuker pasangan nibble

# tiap putaran
L, R = R, L ^ round_f(R, k)
```

`permute` cuma nuker nibble: `[n0,n1,n2,n3] -> [n1,n0,n3,n2]`.

## Data di pairs.txt

File isinya 14 blok, masing-masing 20000 baris dgn satu input difference tetap, misal `# input difference: 0000000a`. Tiap baris formatnya:

```
P   E(P)   E(P ^ delta)
```

Difference-nya cuma `a`/`5` di satu nibble (plus `77`/`55` di nibble 4-5). Ini hint kuat kalau soalnya differential cryptanalysis.

## Kelemahan

**1. Cipher-nya sebenernya kepecah jadi dua.**
`round_f` kerja per nibble, dan nibble `i` cuma pindah ke `i^1`. Jadi nibble 0-1 gak pernah ketemu nibble 2-3. Cipher 32 bit ini isinya 2 cipher 16 bit yg independen (sub-cipher A = nibble 0,1 dan B = nibble 2,3), tiap putaran kuncinya cuma 8 bit per sub-cipher.

Utk sub-cipher dgn state `(a,b,c,d)` (nibble L0,L1,R0,R1), satu putaran jadi:

```
(a,b,c,d) -> (c, d, a ^ S[d^k1], b ^ S[c^k0])
```

**2. S-box-nya bias differensial.**
Dari DDT, input diff `a` atau `5` punya transisi dgn peluang tinggi. Pas aku itung propagasi diff-nya lewat DP, hasilnya karakteristik yg masih kuat sampai 6 putaran:

| input diff | setelah 5 putaran | peluang |
|---|---|---|
| R0 = a | (a,0,0,0) | 0.14 |
| R0 = 5 | (5,0,0,0) | 0.14 |
| R1 = a | (0,a,0,0) | 0.14 |
| R1 = 5 | (0,5,0,0) | 0.14 |

Peluang segitu jauh di atas random (1/65536), jadi bisa dipakai buat ngetes tebakan kunci.

## Serangan

Kunci dicari dari putaran terakhir ke awal, per sub-cipher:

1. Tebak kunci 8 bit putaran terakhir (256 kemungkinan), lalu "buka" satu putaran dari kedua ciphertext di tiap pasangan:
   ```python
   def peel(st, k0, k1):
       x0, x1, y0, y1 = st
       return [y0 ^ S[x1^k1], y1 ^ S[x0^k0], x0, x1]
   ```
2. Hitung diff state hasil buka, lalu kasih skor = probabilitas diff itu menurut tabel DP. Kunci yg bener bakal dapet skor paling gede.
3. Ambil kunci terbaik, buka putaran itu di semua pasangan, ulang utk putaran sebelumnya sampai putaran 2.
4. Awalnya aku pakai cara "diff harus sama persis dgn target", tapi pas putaran 6 dan 3 skornya seri (diff di input S-box nol, jadi kunci gak keliatan). Makanya pindah ke skor berbobot probabilitas, itu yg dipakai di hasil akhir.
5. Kunci putaran 1 gak perlu differential, cukup brute force 256 kemungkinan per sub-cipher pakai known plaintext (`P`, `E(P)`), cuma 1 yg cocok.

Pakai blok diff `a`, `5` di nibble 0-1 utk sub-cipher A, dan nibble 2-3 utk B. Blok lainnya gak kepakai.

## Hasil

Subkey putaran 1 sampai 7:

```
0xa383  0x3ab5  0xd728  0x7222  0x9737  0x7212  0xfdec
```

Master key (`subkeys[i] = master >> (16*i)`):

```
0xfdec721297377222d7283ab5a383
```

Dicek ke 200 pasangan `P -> E(P)`, semuanya cocok. Lalu `flag.enc` didekripsi per blok 4 byte (ECB), hasilnya:

```
NFXCTF{d1ff3r3nt14l_crypt0_pwn5_f31st3l_45322835}\x03\x03\x03
```

Tiga byte `\x03` di akhir itu padding PKCS#7.

