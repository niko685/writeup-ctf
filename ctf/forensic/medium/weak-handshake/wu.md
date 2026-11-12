# Weak Handshake - Forensics (180 pts)

## Deskripsi

> Beberapa kenangan mudah terlihat di awal. Namun ending-nya dikirim melalui koneksi terenkripsi, tersembunyi di balik handshake yang ternyata tidak cukup kuat untuk menjaganya. Awalnya mungkin terlihat jelas, tetapi ending milik Tomoyo masih menunggu di dalam capture.

Dari deskripsinya udah ketebak: ada 2 bagian, yg satu polos/plaintext, yg satu lagi dienkripsi TLS tapi "handshake-nya lemah". Tinggal cari di mana lemahnya.

## Recon pcap

Pas dicek pakai `file`, ternyata ini bukan pcap biasa:

```
capture.pcapng: pcap capture file, microsecond ts (little-endian) - version 2.4 (Linux cooked v2)
```

Linktype-nya SLL2 (Linux cooked capture v2), header-nya 20 byte sebelum masuk ke IP header. Sayangnya di environment gak ada tshark/scapy dan gak ada akses internet buat install, jadi parsing pcap-nya ditulis manual pakai Python (baca global header + record header pcap klasik, terus strip 20 byte SLL2 di tiap paket).

Setelah di-breakdown per flow, ketemu beberapa koneksi menarik, semua src/dst sama-sama `192.168.56.100` (loopback/local):

- Port **2121** <-> port acak → FTP
- Port **8443** <-> dua port beda (dua sesi terpisah) → TLS

## Bagian 1: FTP (yang "mudah terlihat di awal")

Reassemble stream TCP-nya (urutin byte pakai relative seq number biar aman dari out-of-order/retransmit), hasil plaintext-nya:

```
USER anonymous
PASS ftp@example.com
PWD
EPSV
TYPE I
SIZE clannad.webp
RETR clannad.webp
QUIT
```

Jadi cuma download `clannad.webp` biasa via FTP anonymous, gak ada yg disembunyiin, sesuai sama baris pertama deskripsi ("beberapa kenangan mudah terlihat di awal"). File ini bukan jawaban, cuma informasi konteks doang (dan buat kasih tau tema: Clannad / Tomoyo).

## Bagian 2: TLS (yang "ending"-nya)

Dua sesi TLS ke `tomoyo.local:8443`, cipher suite yg dipake: `0x002F` = `TLS_RSA_WITH_AES_128_CBC_SHA`.

Nah ini kuncinya: cipher suite ini pake **RSA key exchange**, bukan (EC)DHE. Artinya kalau private key RSA server bisa didapet, semua traffic bisa didekripsi dari capture, gak perlu MITM live.

Ambil sertifikat server dari handshake (Certificate message), terus di-dump pakai openssl:

```
Subject: CN = tomoyo.local
Public-Key: (1024 bit)
Exponent: 65537
```

RSA **1024-bit**. Itu dia "handshake yang tidak cukup kuat" nya — key segini sebenernya udah dianggap lemah, apalagi kalau primanya digenerate asal2an.

### Factoring si modulus

Ambil `n` dari pubkey, langsung coba Fermat's factorization (nebak p & q itu berdekatan nilainya):

```python
import math
a = math.isqrt(n)
if a*a < n: a += 1
while True:
    b2 = a*a - n
    b = math.isqrt(b2)
    if b*b == b2:
        p, q = a-b, a+b
        break
    a += 1
```

Dan... ketemu di percobaan **pertama** (`i=0`). Berarti selisih p sama q cuma beberapa ratus doang (deket banget), itu kelemahan generate key-nya. Dari situ langsung dapet p & q, terus susun ulang jadi RSA private key pake `cryptography` lib.

### Decrypt TLS

Dengan private key di tangan, proses decrypt-nya:

1. Ambil `encrypted premaster secret` dari pesan ClientKeyExchange tiap koneksi
2. Decrypt pake RSA (PKCS1v15) → dapet premaster secret (48 byte, 2 byte pertama = versi TLS, cocok `0x0303`, jadi key bener)
3. Awalnya coba derive master secret standar (`client_random + server_random`), tapi hasil decrypt record-nya gagal (ciphertext length gak kelipatan 16 block AES)
4. Cek lagi ekstensi di ClientHello, ternyata ada `extended_master_secret` (RFC 7627) sama `encrypt_then_mac` (RFC 7366) — dua ekstensi ini ngubah cara derive master secret & format record-nya
5. Fix:
   - Master secret dihitung pake `session_hash` (SHA256 dari transcript handshake: ClientHello+ServerHello+Certificate+ServerHelloDone+ClientKeyExchange), bukan cuma random doang
   - Format record jadi `IV(16) + ciphertext + MAC(20)` di luar ciphertext (karena Encrypt-then-MAC), bukan MAC di dalem plaintext kayak biasa

Setelah itu Finished message bisa didecode bener (`0x14 00 00 0c ...`), tandanya key udah pas. Lanjut decrypt application data di kedua sesi.

### Isi yang kebuka

Sesi 1 (client→server):
```
GET /begin.jpg HTTP/1.1
Host: tomoyo.local:8443
```

Sesi 2 (client→server):
```
GET /ending.jpg HTTP/1.1
Host: tomoyo.local:8443
```

Response-nya HTTP biasa (`SimpleHTTP/0.6 Python/3.10.13`) isinya JPEG. Strip header HTTP, sisanya langsung jadi file JPEG valid. `begin.jpg` cuma gambar awal (pasangan dari deskripsi "awalnya mungkin terlihat jelas"), sedangkan `ending.jpg` isinya gambar Tomoyo dengan watermark flag di bagian bawah.

## Flag

```
NFX{D36A6428F80788B60C677EF94FA7157}
```
