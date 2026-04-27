### Riddle Registry:
- Used exiftool to find the encoded flag under author name
- Applied base64 -d to get flag

### Flag in Flame:
- Convert all binary from base64 in cyberchef and generate an image 
- The image has a hex code on the image
- Convert it from hex and you get the flag
-PS extracted hex code: `7069636F4354467B666F72656E736963735F616E616C797369735F69735F616D617A696E675F62396163346362397D`
-Used https://ocr.space/ to get the exact text on image

### Corrupted File:
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

### RED:
- Needed to use zsteg to get the encoded flag in text
` 
`❯ zsteg  red.png`  
`meta Poem           .. text: "Crimson heart, vibrant and bold,\nHearts flutter at your sight.\nEvenings glow softly red,\nCherries burst with s`  
`weet life.\nKisses linger with your warmth.\nLove deep as merlot.\nScarlet leaves falling softly,\nBold in every stroke."`  
`chunk:0:IHDR        .. file: Adobe Photoshop Color swatch, version 0, 128 colors; 1st RGB space (0), w 0x80, x 0x806, y 0, z 0; 2nd HSB space (`  
`1), w 0x100, x 0, y 0xff01, z 0xff`  
`b1,rgba,lsb,xy      .. text: **"cGljb0NURntyM2RfMXNfdGgzX3VsdDFtNHQzX2N1cjNfZjByXzU0ZG4zNTVffQ==cGljb0NURntyM2RfMXNfdGgzX3VsdDFtNHQzX2N1cjNfZjByX`  `zU0ZG4zNTVffQ==cGljb0NURntyM2RfMXNfdGgzX3VsdDFtNHQzX2N1cjNfZjByXzU0ZG4zNTVffQ==cGljb0NURntyM2RfMXNfdGgzX3VsdDFtNHQzX2N1cjNfZjByXzU0ZG4zNTVffQ==`**  
`"`  
`b1,rgba,msb,xy      .. file: OpenPGP Public Key`  
`b2,g,lsb,xy         .. text: "ET@UETPETUUT@TUUTD@PDUDDDPE"`  
`b2,rgb,lsb,xy       .. file: OpenPGP Secret Key`  
`b2,bgr,msb,xy       .. file: OpenPGP Public Key`  
`b2,rgba,lsb,xy      .. file: OpenPGP Secret Key`  
`b2,rgba,msb,xy      .. text: "CIkiiiII"`  
`b2,abgr,lsb,xy      .. file: OpenPGP Secret Key`  
`b2,abgr,msb,xy      .. text: "iiiaakikk"`  
`b3,rgba,msb,xy      .. text: "#wb#wp#7p"`  
`b3,abgr,msb,xy      .. text: "7r'wb#7p"`  
`b4,b,lsb,xy         .. file: 0421 Alliant compact executable not stripped`

### Verify:
It just needed a script to run each file in the files folder through the decrypt script however nano and mousepad could not be used so used cat with a heredoc:
`cat > solve.sh << 'EOF'`
`#!/bin/bash`
`for file in /home/ctf-player/drop-in/files/*; do`
    `filename=$(basename "$file")`
    `echo "Trying: $filename"`
    `result=$(./decrypt.sh "files/$filename" 2>/dev/null)`
    `if echo "$result" | grep -q "picoCTF{"; then`
        `echo "FLAG FOUND in $filename:"`
        `echo "$result"`
        `break`
    `fi`
`done`
`echo "Done."`
`EOF`
- Note: remember to make the .sh file executable with `chmod +x solve.sh`

### Timeline 0:

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

### Timeline 1:
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
