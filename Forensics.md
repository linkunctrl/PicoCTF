## Riddle Registry:
- Used exiftool to find the encoded flag under author name
- Applied base64 -d to get flag

## Flag in Flame:
- Convert all binary from base64 in cyberchef and generate an image 
- The image has a hex code on the image
- Convert it from hex and you get the flag
-PS extracted hex code: `7069636F4354467B666F72656E736963735F616E616C797369735F69735F616D617A696E675F62396163346362397D`
-Used https://ocr.space/ to get the exact text on image

## Corrupted File:
- **Identity Check**
    
    - `strings file` ➜ `CDEFGHIJSTUVWXYZcdefghijstuvwxyz` (Huffman Table)
        
    - `head -n 1 file` ➜ `\x..JFIF` (Confirms JPEG/JFIF format)
    - `xxd or hexdump TELLS WHICH BYTES ARE CORRUPTED` 
        
    - **Magic Bytes:** `FF D8 FF E0`
        
- **Repair Command**
    
    - `(printf '\xff\xd8' && tail -c +3 file) > repair.jpg`
        
    - ↳ `printf '\xff\xd8'`: Injects correct **FF D8** start bytes.
        
    - ↳ `&&`: Sequence operator (runs next command if first succeeds).
        
    - ↳ `tail -c +3`: Strips the first **2** corrupted bytes.
        
    - ↳ `> repair.jpg`: Outputs the repaired binary to a new file.
        
- **Outcome**
    
    - Check `repair.jpg` for the flag.

## RED:
- Needed to use zsteg to get the encoded flag in text


❯ zsteg  red.png
meta Poem           .. text: "Crimson heart, vibrant and bold,\nHearts flutter at your sight.\nEvenings glow softly red,\nCherries burst with s
weet life.\nKisses linger with your warmth.\nLove deep as merlot.\nScarlet leaves falling softly,\nBold in every stroke."
chunk:0:IHDR        .. file: Adobe Photoshop Color swatch, version 0, 128 colors; 1st RGB space (0), w 0x80, x 0x806, y 0, z 0; 2nd HSB space (
1), w 0x100, x 0, y 0xff01, z 0xff
b1,rgba,lsb,xy      .. text: **"cGljb0NURntyM2RfMXNfdGgzX3VsdDFtNHQzX2N1cjNfZjByXzU0ZG4zNTVffQ==cGljb0NURntyM2RfMXNfdGgzX3VsdDFtNHQzX2N1cjNfZjByX  zU0ZG4zNTVffQ==cGljb0NURntyM2RfMXNfdGgzX3VsdDFtNHQzX2N1cjNfZjByXzU0ZG4zNTVffQ==cGljb0NURntyM2RfMXNfdGgzX3VsdDFtNHQzX2N1cjNfZjByXzU0ZG4zNTVffQ**==  
"  
b1,rgba,msb,xy      .. file: OpenPGP Public Key
b2,g,lsb,xy         .. text: "ET@UETPETUUT@TUUTD@PDUDDDPE"
b2,rgb,lsb,xy       .. file: OpenPGP Secret Key  
b2,bgr,msb,xy       .. file: OpenPGP Public Key  
b2,rgba,lsb,xy      .. file: OpenPGP Secret Key
b2,rgba,msb,xy      .. text: "CIkiiiII"
b2,abgr,lsb,xy      .. file: OpenPGP Secret Key 
b2,abgr,msb,xy      .. text: "iiiaakikk"  
b3,rgba,msb,xy      .. text: "#wb#wp#7p"  
b3,abgr,msb,xy      .. text: "7r'wb#7p" 
b4,b,lsb,xy         .. file: 0421 Alliant compact executable not stripped

## Verify:
It just needed a script to run each file in the files folder through the decrypt script however nano and mousepad could not be used so used cat with a heredoc:

cat > solve.sh << 'EOF'
#!/bin/bash
for file in /home/ctf-player/drop-in/files/*; do
    filename=$(basename "$file")
    echo "Trying: $filename"
    result=$(./decrypt.sh "files/$filename" 2>/dev/null)
    if echo "$result" | grep -q "picoCTF{"; then
        echo "FLAG FOUND in $filename:"
        echo "$result"
        break
    fi
done
echo "Done."
EOF

- Note: remember to make the .sh file executable with `chmod +x solve.sh`

## Timeline 0:

The challenge involves analyzing a Linux ext4 filesystem image (`partition4.img`) to find a hidden flag. A naive extraction misses it — the flag lives in file **metadata**, not file content.

---

#### Step 1 — Identify the Image

```bash
file partition4.img
# partition4.img: Linux rev 1.0 ext4 filesystem data, UUID=7a00e9da-...
```

---

#### Step 2 — Why `binwalk` Fails Here

```bash
binwalk -e partition4.img
```

Extracts filesystem contents (`bin/`, `etc/`, `usr/`, etc.) but **misses the flag** — it only extracts file _content_, not filesystem _metadata_ (timestamps, inodes). The flag is hidden in metadata.

> **Key insight:** Always consider filesystem metadata, not just file contents.

---

#### Step 3 — Build a Filesystem Body File

```bash
fls -m '/' -r partition4.img > body.txt
```

|Flag|Meaning|
|---|---|
|`-m '/'`|Prepend a mount point prefix (`/`) to all paths in output — required for `mactime`|
|`-r`|Recursively list all files and directories|

`fls` (File Listing) is a Sleuth Kit tool that lists file and directory names from a disk image, including **deleted** entries. Outputs a "body file" format used by `mactime`.

---

#### Step 4 — Generate a Timeline

```bash
mactime -b body.txt -d > timeline.csv
```

|Flag|Meaning|
|---|---|
|`-b body.txt`|Input body file (from `fls`)|
|`-d`|Output in CSV (comma-delimited) format|

`mactime` sorts filesystem events chronologically by **M**odified, **A**ccessed, **C**hanged, **B**irth (created) timestamps.

---

#### Step 5 — Spot the Anomaly

```
Tue Jan 01 1985 22:00:00,41,macb,...,4945,"/bin/bcab"   ← ANOMALY
Mon Oct 18 2021 22:54:17,...                              ← normal
Mon Oct 18 2021 22:54:17,...                              ← normal
```

Everything is dated 2021–2023 **except one file** timestamped **1985** — a classic CTF red flag (also hinted in the challenge hints).

**Suspicious file:** `/bin/bcab` — inode `4945`

---

#### Step 6 — Extract the File by Inode

```bash
icat partition4.img 4945
# NzFtMzExbjNfMHU3MTEzcl9oM3JfNDNhMmU3YWYK
```

`icat` (Inode Cat) opens a disk image and outputs the file with the specified inode number directly to stdout — works even on deleted files.

---

#### Step 7 — Decode Base64

```bash
echo "NzFtMzExbjNfMHU3MTEzcl9oM3JfNDNhMmU3YWYK" | base64 -d
# 71m311n3_0u7113r_h3r_43a2e7af
```

---

#### Bonus — `/etc/shadow` Findings

Only `root` and `ctf-player` had valid password hashes (SHA-512 / `$6$`). All other accounts were locked (`!`).

```
root:$6$D2p/Dx...
ctf-player:$6$gGponKq7...
```

Not needed for the flag, but useful for further enumeration if required.

---

#### Tool Summary

| Tool                   | Purpose                                      |
| ---------------------- | -------------------------------------------- |
| `file`                 | Identify image type                          |
| `binwalk -e`           | Extract file contents (insufficient here)    |
| `fls -m '/' -r`        | List files + metadata from image → body file |
| `mactime -b -d`        | Build CSV timeline from body file            |
| `icat <image> <inode>` | Extract file by inode number                 |
| `base64 -d`            | Decode base64 string                         |

## Timeline 1:
#### 1. Identify the Image

```
❯ file 2partition4.img
2partition4.img: Linux rev 1.0 ext4 filesystem data, UUID=7a00e9da-98f8-4f0f-b257-95edf422d902 (extents) (64bit) (large files) (huge files)
```

#### 2. Generate a Body File and Timeline

```
❯ fls -m '/' -r 2partition4.img > 2body.txt
❯ mactime -b 2body.txt -d > 2timeline.csv
```

Nothing abnormal in `2timeline.csv` on first look.

---

#### Analysis

Using hints from the challenge, the approach was to pay close attention to timestamps near an anti-forensic action and grep for `macb` in recent files, which leads to this line:

```
Tue Dec 02 2025 02:50:07,49,macb,r/rrw-r--r--,0,0,32716,"/etc/chat"
```

---

#### 3. Extract and Decode the File Contents

```
❯ icat 2partition4.img 32716
NTczNDE3aDEzcl83aDRuXzdoM18xNDU3XzU4NTI3YmIyMjIK
```

```
❯ echo "NTczNDE3aDEzcl83aDRuXzdoM18xNDU3XzU4NTI3YmIyMjIK" | base64 -d
573417h13r_7h4n_7h3_1457_58527bb222
```


## Rogue Tower:
#### 1. Packet Analysis (Hex to Text)

Observed unusual DNS query data identifying carrier information and network status.

- **Packet 1:** `43415252...` → `CARRIER: Verizon PLMN=310410 CELLID=25262`
    
- **Packet 2:** `43415252...` → `CARRIER: AT&T PLMN=310410 CELLID=25263`
    
- **Packet 14 (Suspicious):** `554e4155...` → `UNAUTHORIZED-TEST-NETWORK PLMN=00101 CELLID=90461`
    

---

#### 2. Network Discovery

- **Suspicious Source IP:** `192.168.99.1` (Broadcast traffic only).
    
- **Target Communication:**
    
    - **Source:** `10.100.142.84`
        
    - **Destination:** `198.51.100.247`
        
    - **Action:** POST requests and registration.
        
- **Registration Packet (16):**
    
    - `GET /api/register HTTP/1.1`
        
    - **User-Agent:** `IMSI:310410337059687`
        

---

#### 3. Encrypted Data Stream

Packets 17–22 contain fragmented data that forms a Base64 string.

- **Combined Hex:** `51313554576e7069666b784242316441436d6c62424639626230454a5151744662415142425131515841454153673d3d`
    
- **Base64 String:** `Q15TWnpifkxBB1dACmlbBF9bb0EJQQtFbAQBBQ1QXAEASg==`
    

---

#### 4. Decryption Logic

The encryption key is derived from the **IMSI** (`310410337059687`). Per investigation, the key consists of the **last 8 digits** of the IMSI.

- **Key:** `37059687`
    
- **Mechanism:** XOR operation on Base64 decoded bytes.
    

---

#### 5. Decryption Script

Python

```
import base64

def main():
    data_b64 = "Q15TWnpifkxBB1dACmlbBF9bb0EJQQtFbAQBBQ1QXAEASg=="
    
    # IMSI
    imsi = "310410337059687"
    key = imsi[-8:] # last 8 digits are the key
    
    # Base64
    encrypted_data = base64.b64decode(data_b64)
    key_bytes = key.encode('utf-8')
    
    # XOR
    decrypted = bytearray()
    for i in range(len(encrypted_data)):
        decrypted.append(encrypted_data[i] ^ key_bytes[i % len(key_bytes)])
        
    print("=== FLAG ===")
    print(decrypted.decode('utf-8'))

if __name__ == "__main__":
    main()
```

## Forensics Git 0:

#### 1. Initial Image Analysis

Identify the disk structure using `file` and `fdisk`.

Bash

```
file disk.img
sudo fdisk -l disk.img
```

- **Result:** Three partitions found (Linux, Swap, Linux).
    
- **Target:** Partition 3 (`disk.img3`), starting at sector **1140736**.
    

---

#### 2. Extraction & Mounting

Extract partitions or mount the specific sector offset.

Bash

```
# Calculate offset: 1140736 * 512 = 584056832
mkdir mnt_point
sudo mount -o loop,offset=584056832 disk.img mnt_point
```

---

#### 3. Filesystem Exploration

Navigate to the user directory to find interesting artifacts.

Bash

```
cd mnt_point/home/ctf-player/Code/Secrets
ls -la
```

- **Found:** `.git/` directory and `note.txt`.
    

---

#### 4. Git Forensic Discovery

Inspect the Git history or configuration to find sensitive data.

Bash

```
cd .git
cat COMMIT_EDITMSG
```

- **Phrase Found:** `g17_1n_7h3_d15k_041217d8`
    

## Forensics Git 1:
The disk image `2disk.img` was examined using `fdisk -l`, revealing a **1 GiB** file with a **DOS** disklabel and three partitions:

- **Partition 1:** 300M (Linux)
    
- **Partition 2:** 256M (Swap)
    
- **Partition 3:** 467M (Linux)
    

Bash

```
❯ sudo fdisk -l 2disk.img 
Disk 2disk.img: 1 GiB, 1073741824 bytes, 2097152 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x610b63c2

Device      Boot   Start     End Sectors  Size Id Type
2disk.img1 * 2048  616447  614400  300M 83 Linux
2disk.img2       616448 1140735  524288  256M 82 Linux swap / Solaris
2disk.img3      1140736 2097151  956416  467M 83 Linux
```

---

#### Extraction and Mounting

The image was processed with `binwalk` to extract the filesystem, and the Linux partition was mounted to a local directory.

Bash

```
❯ mkdir mountedimg
❯ binwalk -e 2disk.img 
❯ sudo mount -o loop path/to/Linux_partition.2 /path/to/mountedimg
```

---

#### Git Forensic Analysis

Inside the mounted filesystem, a `.git` repository was located at `/home/ctf-player/Code/secrets/`. The `COMMIT_EDITMSG` file indicated a recent removal of a flag.

Bash

```
mountedimg/home/ctf-player/Code/secrets/.git
❯ cat COMMIT_EDITMSG 
Remove flag
```

#### Commit History

Viewing the `git log` revealed two commits. The flag was added in the initial commit and removed in the most recent one.

Bash

```
❯ git log
commit 5fb8194539c770a830b8ba089a50778c07072b03 (HEAD -> master)
Author: ctf-player <ctf-player@example.com>
Date:   Wed Nov 19 09:20:05 2025 +0000

    Remove flag

commit 177789af0b300e043ea8f54ea57d6cee352291ae
Author: ctf-player <ctf-player@example.com>
Date:   Wed Nov 19 09:20:05 2025 +0000

    Add flag
```

---

#### Flag Recovery

By inspecting the changes in the "Remove flag" commit using `git show`, the deleted content of `flag.txt` was recovered.

Bash

```
❯ git show 5fb8194539c770a830b8ba089a50778c07072b03
commit 5fb8194539c770a830b8ba089a50778c07072b03 (HEAD -> master)
Author: ctf-player <ctf-player@example.com>
Date:   Wed Nov 19 09:20:05 2025 +0000

    Remove flag

diff --git a/flag.txt b/flag.txt
deleted file mode 100644
index f150f47..0000000
--- a/flag.txt
+++ /dev/null
@@ -1 +0,0 @@
-picoCTF{g17_r3m3mb3r5_d4ddf904}
```

## Forensics Git 2:
#### Disk Analysis

---

#### Disk Image Overview

- **Size:** 1 GiB
    
- **Identifier:** 0x610b63c2
    
- **Partitions:** 3 total
    

|**Device**|**Start**|**End**|**Size**|**Type**|
|---|---|---|---|---|
|`3disk.img1`|2048|616447|300M|Linux|
|`3disk.img2`|616448|1140735|256M|Swap|
|`3disk.img3`|1140736|2097151|467M|Linux|

---

#### Extraction & Mounting
 **Binwalk -e 3disk.img** && **mkdir mountimg**

- **Directory:** `.../3disk.img.extracted/0`
    
- **Mount Point:** `/mountimg/`
    
- **Action:** Mount `Linux_partition.2` via loop device.
     ❯ sudo mount -o loop /path/to/Linux_partition.2  /path/to/mountimg/

---

#### Git Log History

- **File Path:** `.../.git/logs/HEAD`
	 

e80b38b3322a5ba32ac07076ef5eeb4a59449875 2151ef0ccc15aed1ab88e1afdc7484aaeff211c4 ctf-player <ctf-player@example.com> 1763549240 +0000	commit: **Remove secret hideout log**


---

#### Flag Recovery

- **Target Commit:** `e80b38b3322a5ba32ac07076ef5eeb4a59449875`
     git show (above commit)
- **Recovered File:** `logs/3.txt`
		picoCTF{g17_r35cu3_16ac6bf3}

##
