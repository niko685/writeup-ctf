# Write Up - Flight Recovery Manifest

## Deskripsi

> Sebuah jet tempur Sukhoi Su-30 terjatuh, menyimpan pesan yg masih terselamatkan. Kejadian ini terjadi di perbatasan negara aliansi NATO.
>
> kunci: Placeholder

Dikasih 1 file: `flight_recovery_manifest.xlsx`

## Analisa File

Buka xlsx nya, isinya 1 sheet dgn 3 kolom:

| Kolom | Isi |
|---|---|
| Kode Pesawat | 15 baris kata bahasa Inggris (November, Four, Tango, ...) |
| kusi Pesawat | 10 baris angka 2 digit (43, 30, 6d, 62, ...) |
| Jumlah Barang | 3 angka gede, tampil dlm format scientific (2.78E+23 dst) |

Sheet nya ga ada hidden sheet, comment, ataupun drawing. Jadi semua datanya ada di 3 kolom itu.

## Solve

### 1. Kolom Kode Pesawat

Kata2 nya ternyata NATO phonetic alphabet (sesuai tema NATO di deskripsi). Ambil huruf/angka pertama tiap kata:

```
November Four Tango Zero Papa Hotel Zero November Three Tango One Charlie One November One
N        4    T     0    P    H     0    N        3     T     1   C       1   N        1
```

Hasil: `N4T0PH0N3T1C1N1`

Kalau dibaca leetspeak jadi `N4T0` `PH0N3T1C` `1N1`. Ada 3 kata, pas sama `World_World_World` di format flag.

### 2. Kolom kusi Pesawat

Isinya 10 angka 2 digit yg ada huruf a-f nya (`6d`, `4f`, dll), jadi kemungkinan besar hex. Convert ke ASCII:

```
43 30 6d 62 4f 5f 48 33 58 5f
C  0  m  b  O  _  H  3  X  _
```

Hasil: `C0mbO_H3X_`

Ada underscore di akhir, berarti flag nya masih lanjut.

### 3. Kolom Jumlah Barang

Bagian ini yg paling lama. Angka nya:

```
2.78842154354096E+23
7.07930905431442E+31
3.57380928075183E+30
```

Percobaan yg gagal:

- convert ke hex lalu ASCII
- convert ke basis 2 sampai 36 dan 62
- decimal ASCII per 2 atau 3 digit
- XOR sama kunci `Placeholder`

Semuanya ga ngasih teks yg masuk akal. Nilai nya kelihatan aneh karena digit nya kepotong cuma 15 digit (float), tapi justru itu petunjuk nya: angka ini mungkin bukan angka, tapi bytes yg dibaca sbg float.

Dicoba pack ke **float 32-bit big-endian** (`>f`), dan hasilnya langsung kebaca:

| Angka | Hex | ASCII |
|---|---|---|
| 2.78842154354096E+23 | `66 6C 30 41` | `fl0A` |
| 7.07930905431442E+31 | `74 5F 62 34` | `t_b4` |
| 3.57380928075183E+30 | `72 34 6E 67` | `r4ng` |

Disambung: `fl0At_b4r4ng` (float barang).

Script nya:

```python
import struct

vals = [2.78842154354096E23, 7.07930905431442E31, 3.57380928075183E30]
print("".join(struct.pack(">f", v).decode() for v in vals))
# fl0At_b4r4ng
```

Kunci `Placeholder` ternyata ga kepake.

### 4. Gabung semua

```
N4T0 _ PH0N3T1C _ 1N1 _ C0mbO _ H3X _ fl0At _ b4r4ng
```

Kalau dibaca: "NATO phonetic ini combo hex float barang". Sesuai sama isi file nya, ada phonetic, hex, dan float.

## Flag

```
NFX{N4T0_PH0N3T1C_1N1_C0mbO_H3X_fl0At_b4r4ng}
```
