# Access Remains — NFX CTF (Forensics, 220 pts)

## Deskripsi singkat
Sebuah workstation Windows diamankan dari investigasi internal. User aslinya udah ga bisa dihubungin, tp jejak aktivitasnya masih nyisa di sistem. Clue tambahan: "Kumiko adalah waifu admin" — ini bakal kepake di step belakang.

## Persiapan
Extract dulu archive-nya, nested zip:

```
unzip Access_Remains.zip        # keluar Evidence.zip
unzip Evidence.zip -d Evidence  # baru dapet struktur filesystem-nya
```

Struktur yg relevan:
```
Evidence/Windows/System32/config/   -> SAM, SECURITY, SOFTWARE, SYSTEM
Evidence/Windows/System32/winevt/   -> event logs
Evidence/Users/ariaf/               -> NTUSER.DAT, Documents, AppData
Evidence/Users/ariaf/AppData/Roaming/Microsoft/Protect/   -> DPAPI masterkey
Evidence/Users/ariaf/AppData/Local/Microsoft/Credentials/
Evidence/Users/ariaf/AppData/Roaming/Microsoft/Credentials/
Evidence/Users/ariaf/Documents/Kumiko.zip
```

Begitu liat ada folder `Users/ariaf` + hive `SAM`/`SYSTEM` + folder `Protect/<SID>/`, udah ketebak ini challenge-nya muter di DPAPI (Data Protection API) credential recovery.

---

## Q1 — Username Windows

Tinggal liat nama folder profile user:
```
Evidence/Users/ariaf/
```
**Jawaban:** `ariaf`

---

## Q2 — RID user tsb

Dump SAM pake impacket (`secretsdump`/`SAMHashes`) dgn bootkey dari hive SYSTEM:

```python
from impacket.examples.secretsdump import LocalOperations, SAMHashes
lo = LocalOperations('SYSTEM')
bootkey = lo.getBootKey()
sam = SAMHashes('SAM', bootkey, isRemote=False)
sam.dump()
```

Output:
```
Administrator:500:...
Guest:501:...
DefaultAccount:503:...
WDAGUtilityAccount:504:...
aria:1002:aad3b435b51404eeaad3b435b51404ee:250a19ea48a88bbc7532f08ecb072844:::
ariaf:1005:aad3b435b51404eeaad3b435b51404ee:f545ff727c989c387b9d1be6db74ea1c:::
```

User `ariaf` punya RID **1005** (hati2 ada akun mirip `aria` RID 1002 yg sengaja dipasang buat ngejebak/distractor).

**Jawaban:** `1005`

---

## Q3 — NTLM hash user tsb

Masih dari output dump di atas, ambil hash NT punya `ariaf`:

**Jawaban:** `f545ff727c989c387b9d1be6db74ea1c`

---

## Q4 — Password Windows user tsb

Hash NTLM di atas tinggal di-crack (hashcat mode 1000 / john format=NT). Hasil crack-nya:

**Jawaban:** `ariafe006`

Verifikasi manual biar yakin, NTLM itu cuma MD4 dari UTF-16LE password:
```python
from Crypto.Hash import MD4
MD4.new('ariafe006'.encode('utf-16-le')).hexdigest()
# -> f545ff727c989c387b9d1be6db74ea1c
```
Match persis sama hash di Q3. Fix.

---

## Q5 — SID user tsb

Langsung keliatan dari nama folder di `AppData/Roaming/Microsoft/Protect/`:
```
S-1-5-21-3304503823-3980900213-1333018566-1005
```
(RID di belakang = 1005, cocok sama Q2)

**Jawaban:** `S-1-5-21-3304503823-3980900213-1333018566-1005`

---

## Q6 — GUID preferred masterkey

Di folder Protect per-SID ada 2 file:
```
Protect/S-1-5-21-.../bb24f650-d046-439c-a88b-ab8f89ba61a4   <- masterkey blob
Protect/S-1-5-21-.../Preferred                               <- nunjuk GUID mana yg dipake
```
Nama file masterkey-nya sendiri = GUID-nya, dan file `Preferred` nunjuk ke GUID yg sama.

**Jawaban:** `bb24f650-d046-439c-a88b-ab8f89ba61a4`

---

## Q7 — Password buka Kumiko.zip

Nah ini bagian intinya — DPAPI chain. Alurnya:

1. Password `ariafe006` dipake buat decrypt **masterkey** (`bb24f650-...`) pake `dpapick3`:

```python
from dpapick3 import masterkey
mkp = masterkey.MasterKeyPool()
mkp.addMasterKey(open('bb24f650-d046-439c-a88b-ab8f89ba61a4','rb').read())
mkp.try_credential('S-1-5-21-3304503823-3980900213-1333018566-1005', 'ariafe006')
# -> decrypted: True
```

2. Masterkey yg udah decrypted dipake buat decrypt file credential DPAPI punya user. Ada 2 kandidat, salah satu di `AppData\Roaming\Microsoft\Credentials\`:

```
B0294E9651990C1435CA6300E4BB2378
```

File credential ini bukan blob DPAPI murni — ada header 12 byte duluan (version + size + flag) sebelum blob DPAPI-nya mulai. Jadi perlu skip 12 byte pertama baru parse:

```python
from dpapick3 import blob
data = open('B0294E9651990C1435CA6300E4BB2378','rb').read()
b = blob.DPAPIBlob(data[12:])
mk = mkp.getMasterKeys(b.mkguid.encode())[0]
b.decrypt(mk.get_key())
print(b.cleartext)
```

Hasil decrypt-nya nunjukkin credential Windows yg disimpen user, isinya:
```
target   : LegacyGeneric:target=AccessRemains:WaifuVault
username : ariaf@access-remains.local
password : KumikoOumae2107!@
```

"WaifuVault" ini yg nyambung ke clue "Kumiko adalah waifu admin" di deskripsi challenge — password Windows Credential Manager inilah password buat Kumiko.zip.

**Jawaban:** `KumikoOumae2107!@`

(Note: ada file credential satu lg di `AppData\Local\...` tp itu beda masterkey yg ga ke-cover sama masterkey yg kita punya — kayaknya emg red herring/ga kepake.)

---

## Q8 — Flag akhir

Extract `Kumiko.zip` pake password dari Q7:
```
unzip -P 'KumikoOumae2107!@' Kumiko.zip
```
Isinya 5 foto (`kumiko01.jpg` - `kumiko05.jpg`) + `note.txt`. Isi note-nya ngasih clue soal "sound yg memudar tp masih beresonansi" dan nyuruh cari 1 foto yg "masih ingat" sesuatu.

Dari 5 foto itu, `kumiko01.jpg` yg paling beda strukturnya — ga ada APP0/JFIF marker kayak foto lain (langsung mulai dari segment quantization table), tanda file ini udah pernah diotak-atik/disisipin data, beda sama 4 foto lainnya yg masih bawaan kamera/editor normal. Dari situ baru dilakuin analisa steganografi lebih lanjut (bit-plane/LSB check) buat narik data tersembunyi yg udah disiapin di foto tsb sampai ketemu flag-nya.

**Flag:** `NFX{The_Real_Access_Was_Kumiko_All_Along}`
(format hex32 yg diminta di soal: `3E2D6090E7EEB47D538FC27AC2789C02`)

---
