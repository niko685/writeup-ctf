# Bluebird Session 

## Soal

Workstation Linux diamankan, ada 2 user tersisa: Liz dan Blue Bird. Akses udah ga ada tapi shell masih nyimpen semua jejaknya. Disuruh telusuri kedua user, pulihin secret yang kesimpen, terus buka archive yang dilindungi.

File yang dikasih: `Bluebird_Session1.zip`, password `nxctf.id`.

## Ekstrak Archive

Zip luar dibuka pake password yang dikasih di soal (`nxctf.id`), isinya cuma satu file lagi: `Evidence.zip`. Archive ini encrypted pake AES (WinZip AES, bukan password protection legacy), jadi ga bisa langsung dibuka pake `unzip` biasa — perlu tool yang support AES decrypt (7z atau yang lain).

Setelah dibongkar, isinya adalah snapshot filesystem dari server bernama `srv`:

```
Evidence/
├── etc/
│   ├── passwd
│   ├── shadow
│   ├── group
│   └── ...
├── home/
│   ├── mizore/
│   └── nozomi/
├── root/
└── var/log/
```

## Mapping User

Dari petunjuk di soal: "Liz tetap tinggal" dan "blue bird tinggalin jejak". Cek `/etc/passwd`:

```
mizore:x:1001:1001::/home/mizore:/bin/bash
nozomi:x:1002:1002::/home/nozomi:/bin/bash
```

Liz = `mizore`, UID `1001`. Blue Bird = `nozomi`, UID `1002`.

**Q1-Q4:**
- Username Liz: `mizore`
- Username Blue Bird: `nozomi`
- UID Liz: `1001`
- UID Blue Bird: `1002`

## Shadow Hash Liz

Cek `/etc/shadow`, baris mizore:

```
mizore:$6$bluebird$DapjL6a0nzC4W7lGKYhAtHOEPMW06uuQOg4DiPxOWF3RKE6RRSKIumUGXj3QmER5G4ceko4QUsikFswkW330E1:20684:0:99999:7:::
```

Prefix `$6$` artinya algoritma hash-nya SHA-512 crypt, dan salt-nya `bluebird` — ini nyambung ke nama challenge-nya sendiri, "Bluebird Session".

**Q5:** hash Liz = `$6$bluebird$DapjL6a0nzC4W7lGKYhAtHOEPMW06uuQOg4DiPxOWF3RKE6RRSKIumUGXj3QmER5G4ceko4QUsikFswkW330E1`
**Q6:** jenis hash = `sha512crypt`

## Crack Password Liz

Salt `bluebird` dipake sebagai hint buat wordlist. Password yang cocok sama hash itu adalah:

```
Bluebirds1927!
```

Dicek pake `openssl passwd -6 -salt bluebird`, hasilnya sama persis sama hash di shadow.

**Q7:** password Liz = `Bluebirds1927!`

## Login SSH Terakhir

Cek `var/log/auth.log`:

```
Aug 19 21:07:27 bluebird sshd[2107]: Accepted password for mizore from 192.168.56.21 port 51422 ssh2
Aug 19 21:07:27 bluebird sshd[2107]: pam_unix(sshd:session): session opened for user mizore(uid=1001) by (uid=0)
```

Cuma ada satu login sukses buat mizore, dari IP `192.168.56.21`.

**Q8:** IP login SSH terakhir Liz = `192.168.56.21`

## Jejak Blue Bird

Cek `.bash_history` punya nozomi, isinya nunjukin dia bikin folder, naro file, terus zip-in jadi satu archive protected:

```
mv *.jpg /home/nozomi/Documents/for_nozomi_content/
cd /home/nozomi/Documents/for_nozomi_content
7z a -tzip ../for_nozomi.zip . -p'MizoreCanFly2077!@'
sudo chown nozomi:nozomi /home/nozomi/Documents/for_nozomi.zip
```

Password zip-nya langsung kebaca plaintext di command `7z a` itu, ga perlu di-crack lagi.

**Q9:** archive yang ditinggalin = `/home/nozomi/Documents/for_nozomi.zip`
**Q10:** password archive = `MizoreCanFly2077!@`

## Buka Protected Archive

`for_nozomi.zip` dibuka pake password `MizoreCanFly2077!@`. Isinya konten yang ditinggalin Blue Bird buat Liz, termasuk flag final.

**Q11:** final flag = `3220D64291CDFA4B26844C90690E8F1D`

## Final Flag

```
NFX{Mizore_Promise_Never_Found}
```
