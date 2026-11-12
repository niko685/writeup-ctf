# Fresh Arrival - NFX CTF

## Identifikasi Pesawat

Pertama, download file yang diberikan berupa `flight_recovery_manifest.xlsx`.

Dari file tersebut dilakukan pengecekan isi spreadsheet dan metadata untuk mencari petunjuk terkait identitas pesawat.

Ditemukan beberapa petunjuk berupa:

- `N4T0PH0N3T1C1`
- `C0mbO_H3X_`

Petunjuk tersebut mengarah ke NATO Phonetic dan kombinasi HEX.

## Investigasi Foto

Selanjutnya lakukan pencarian berdasarkan foto pesawat yang diberikan.

Dari hasil pencocokan foto ditemukan gambar yang sama di JetPhotos. Kredit foto menunjukkan nama fotografer **Raihan Al Faris**.

Setelah itu lakukan pengecekan riwayat pesawat berdasarkan serial number (MSN) dan registrasinya.

Diketahui pesawat tersebut memiliki data:

- Aircraft: Boeing 737-341(SF)
- MSN: 26854
- Original Registration: B-2908

## Riwayat Operator

Setelah menelusuri riwayat pesawat, ditemukan bahwa pesawat tersebut awalnya tercatat sebagai Varig NTU sebelum akhirnya diterima oleh China Xinhua Airlines pada 15 April 1993.

China Xinhua Airlines merupakan operator pertama pesawat tersebut.

Informasi yang dibutuhkan untuk flag:

- IATA: XW
- ICAO: CXH
- Headquarters: Beijing
- Photographer: Raihan

## Flag

Format flag: `NFX{IATA_ICAO_kota_namadepan}`

```text
NFX{XW_CXH_beijing_raihan}
```
