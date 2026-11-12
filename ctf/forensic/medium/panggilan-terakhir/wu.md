# Writeup - korban.pcap

**Kategori:** Forensics / Network

## Soal

Raka nelpon Dina lewat telpon meja kantor Cakrawala jam 01.18 WIB, abis itu Dina ditemukan tewas di ruang arsip. Kita dikasih rekaman paket panggilannya (`korban.pcap`) dan disuruh cari flag yg diucapkan Dina.

## Langkah

### 1. Cek isi pcap

Isi pcap cuma UDP, total 3430 paket. Dipisah per flow hasilnya:

| Flow | Jumlah paket | Keterangan |
|---|---|---|
| 10.24.8.24:4002 -> 10.24.8.31:4000 | 2374 | RTP |
| 10.24.8.31:4000 -> 10.24.8.24:4002 | 1051 | RTP |
| port 5060 (2 arah) | 5 | SIP |

Berarti ini panggilan VoIP: SIP buat signaling, RTP buat suaranya.

### 2. Baca SIP

Dari 5 paket SIP urutannya INVITE, 200 OK, ACK, BYE, 200 OK.

- Ext **204** (10.24.8.31) = Raka, yg nelpon (INVITE ke 231)
- Ext **231** (10.24.8.24) = Dina
- Call-ID: `c1a84d0e-204@pbx.cakrawala.internal`
- SDP-nya `m=audio ... RTP/AVP 0` dan `a=rtpmap:0 PCMU/8000`, jadi codec-nya G.711 u-law 8 kHz

BYE-nya dikirim Raka (204), bukan dari Dina.

### 3. Pisahin stream RTP

Ada 2 SSRC:

- `b2310001` -> Dina (231), 2374 paket
- `a2040001` -> Raka (204), 1051 paket

Sequence number di dua stream urut dan nggak ada paket dobel. Di stream Dina ada lompatan timestamp RTP yg gede (kira2 6 dan 8,5 detik) padahal seq-nya lanjut terus, itu kemungkinan jeda diam (silence suppression). Jadi pas rekonstruksi, gap-nya diisi silence biar durasi sama kayak aslinya.

Payload tiap paket 160 byte (20 ms @ 8 kHz), header RTP 12 byte dibuang. Payload semua paket digabung urut seq, trs di-decode:

```bash
ffmpeg -f mulaw -ar 8000 -ac 1 -i dina_231.ul dina_231.wav
```

Kalo pake Wireshark bisa lebih gampang: `Telephony > VoIP Calls > Play Streams`, atau `Telephony > RTP > RTP Streams > Export`.

### 4. Dengerin

Spektrogramnya cuma suara ngomong biasa, nggak ada DTMF atau morse. Jadi tinggal didengerin langsung. Di rekaman Dina (231), dia nyebut kode dokumen rahasianya, yaitu:

```
NFXCTF{ARSIP_MERAH_204}
```

## Flag

```
NFXCTF{ARSIP_MERAH_204}
```
