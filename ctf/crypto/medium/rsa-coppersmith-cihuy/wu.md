# Write Up - RSA Partial Key Leak

## Deskripsi

Server logistik ngenkripsi master flag, terus ada dump memori yg bocor. Bocornya cuma sebagian, dan kita disuruh nentuin cukup atau nggak buat buka flag-nya. Semua data ada di file yg dikasih.

## Data di file

```python
n = 0xE4ABA2D6...   # 1024 bit
e = 65537
c = 0xC5DD7E48...
p_msb = 0xF13228781AB150BC...
p_msb_bits = 300
unknown_bits = 212
```

Jadi ini RSA biasa, tapi `p_msb` ngasih tau 300 bit paling atas dari `p`. Krn `n` 1024 bit, `p` sekitar 512 bit, sisa yg ga diketahui 212 bit.

## Analisis

Kalau `p` ketemu, tinggal hitung `q = n / p`, `d`, terus dekripsi `c`. Masalahnya tinggal gimana nyari 212 bit sisanya.

Ini kasus klasik **partial key exposure**. Kalau lebih dari separuh bit `p` diketahui, `p` bisa dicari pakai metode Coppersmith. Di sini yg diketahui 300 dari 512 bit (sekitar 58%), jadi harusnya cukup.

Bikin polinomial:

```
f(x) = (p_msb << 212) + x
```

`x` itu 212 bit bawah dari `p` (nilainya kecil, `x < 2^212`). Krn `p` habis ngebagi `n`, maka `f(x) ≡ 0 (mod p)`. Batas Coppersmith buat kasus ini kira-kira `x < n^0.25 ≈ 2^256`, dan 212 masih di bawah itu.

## Solusi

Pakai Howgrave-Graham: bikin lattice dari polinomial `n^(m-i) * f^i` dan `x^j * f^m`, di-LLL, terus cari akar dari baris hasil reduksi yg akarnya integer kecil. Aku pakai `fpylll` dgn `m = 4`, `t = 4`, udah langsung dapet.

Setelah `p` ketemu:

1. `q = n // p`
2. `d = pow(e, -1, (p-1)*(q-1))`
3. `m = pow(c, d, n)`
4. ubah `m` ke bytes

### Script

```python
exec(open('chall(1).py').read())
from fpylll import IntegerMatrix, LLL
from sympy import symbols, Poly, ZZ

a = p_msb << unknown_bits
X = 1 << unknown_bits
x = symbols('x')
f = Poly(x + a, x, domain=ZZ)

def attempt(m, t):
    polys = [Poly(n**m, x, domain=ZZ)]
    for i in range(1, m + 1):
        polys.append(Poly(f**i, x, domain=ZZ) * n**(m - i))
    fm = Poly(f**m, x, domain=ZZ)
    for j in range(1, t + 1):
        polys.append(fm * Poly(x**j, x, domain=ZZ))

    dim = len(polys)
    M = IntegerMatrix(dim, dim)
    for r, P in enumerate(polys):
        coeffs = P.all_coeffs()[::-1]
        for k in range(dim):
            cf = int(coeffs[k]) if k < len(coeffs) else 0
            M[r, k] = cf * (X**k)
    LLL.reduction(M)

    for r in range(dim):
        coeffs = [M[r, k] // (X**k) for k in range(dim)]
        P = Poly(list(reversed(coeffs)), x, domain=ZZ)
        if P.is_zero:
            continue
        for root in P.ground_roots():
            if 0 <= root < X:
                p = a + int(root)
                if p > 1 and n % p == 0:
                    return p
    return None

p = attempt(4, 4)
q = n // p
d = pow(e, -1, (p - 1) * (q - 1))
m = pow(c, d, n)
print(m.to_bytes((m.bit_length() + 7) // 8, 'big'))
```

Dependency: `pip install fpylll cysignals sympy`

### Output

```
p = 0xf13228781ab150bc402df686354483d5fa1b226bb33dd3ad9e4d3011ac92ad2f2589b571ca30816efe24517161a24e6c35d3ca04b8c149ae440350586e70e051
b'NFXCTF{C0pp3rsm1th_p4rt14l_k3y_3xp0sur3_9617eda9}'
```

## Flag

```
NFXCTF{C0pp3rsm1th_p4rt14l_k3y_3xp0sur3_9617eda9}
```
