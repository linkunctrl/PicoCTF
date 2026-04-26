# picoCTF Challenge Writeups

A collection of forensics and steganography challenge writeups.

---

## 1. Riddle Registry

**Category:** Forensics / Metadata

### Approach

The flag was hidden in the file's metadata under the author name field.

### Steps

1. Run `exiftool` on the file to inspect metadata:
   ```bash
   exiftool <file>
   ```
2. Locate the encoded value in the **Author** field.
3. Decode it with base64:
   ```bash
   echo "<encoded_value>" | base64 -d
   ```

### Flag
Found after base64 decoding the author metadata field.

---

## 2. Flag in Flame

**Category:** Forensics / Steganography

### Approach

The file contained base64-encoded data that, when decoded, produced an image with a hex string embedded in it.

### Steps

1. Decode the base64 content using [CyberChef](https://gchq.github.io/CyberChef/) → output is an image.
2. Use an OCR tool (e.g., [ocr.space](https://ocr.space/)) to extract the hex string from the image.
3. Extracted hex:
   ```
   7069636F4354467B666F72656E736963735F616E616C797369735F69735F616D617A696E675F62396163346362397D
   ```
4. Convert hex to ASCII (CyberChef: **From Hex**) to get the flag.

### Flag
`picoCTF{forensics_analysis_is_amazing_b9ac4cb9}`

---

## 3. Corrupted File

**Category:** Forensics / File Repair

### Approach

A JPEG file had corrupted magic bytes at the start. Restoring the correct header revealed the flag.

### Identifying the File

```bash
strings file          # Shows Huffman table → confirms JPEG internals
head -n 1 file        # Shows \x..JFIF → confirms JPEG/JFIF format
xxd file | head        # Inspect raw bytes to find corruption
```

**Correct JPEG magic bytes:** `FF D8 FF E0`

### Repair Command

```bash
(printf '\xff\xd8' && tail -c +3 file) > repair.jpg
```

| Part | Purpose |
|------|---------|
| `printf '\xff\xd8'` | Injects the correct `FF D8` start bytes |
| `&&` | Runs the next command only if the first succeeds |
| `tail -c +3` | Skips the first 2 corrupted bytes from the original file |
| `> repair.jpg` | Writes the repaired binary to a new file |

### Flag
Open `repair.jpg` to retrieve the flag.

---

## 4. RED

**Category:** Steganography / LSB

### Approach

The flag was hidden in the LSB (Least Significant Bit) layer of a PNG image, encoded in base64.

### Steps

1. Run `zsteg` on the image:
   ```bash
   zsteg red.png
   ```
2. In the output, locate the `b1,rgba,lsb,xy` channel — it contains a repeated base64 string:
   ```
   cGljb0NURntyM2RfMXNfdGgzX3VsdDFtNHQzX2N1cjNfZjByXzU0ZG4zNTVffQ==
   ```
3. Decode it:
   ```bash
   echo "cGljb0NURntyM2RfMXNfdGgzX3VsdDFtNHQzX2N1cjNfZjByXzU0ZG4zNTVffQ==" | base64 -d
   ```


---

## 5. Verify

**Category:** Forensics / Scripting

### Approach

Multiple files needed to be run through a provided `decrypt.sh` script. The flag would appear in the output of the correct file.

### Steps

Since `nano` and `mousepad` were unavailable, the script was created using a heredoc:

```bash
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
```

Then make it executable and run:
```bash
chmod +x solve.sh
./solve.sh
```

### Flag
Printed to stdout when the correct file is processed.

---

## 6. Timeline 0

**Category:** Forensics / Filesystem Analysis

### Approach

The flag was hidden in **filesystem metadata** (timestamps) inside a Linux ext4 image — not in any file's content. A file with an anomalous 1985 timestamp pointed to the target inode.

### Step 1 — Identify the Image

```bash
file partition4.img
# partition4.img: Linux rev 1.0 ext4 filesystem data, UUID=7a00e9da-...
```

### Step 2 — Why `binwalk` Falls Short

```bash
binwalk -e partition4.img
```

This extracts file *contents* (`bin/`, `etc/`, `usr/`, etc.) but **misses the flag**, which is stored in filesystem *metadata* (timestamps, inodes).

> **Key insight:** Always consider filesystem metadata, not just file contents.

### Step 3 — Build a Filesystem Body File

```bash
fls -m '/' -r partition4.img > body.txt
```

| Flag | Meaning |
|------|---------|
| `-m '/'` | Prepends mount point prefix to all paths (required for `mactime`) |
| `-r` | Recursively lists all files and directories |

### Step 4 — Generate a Timeline

```bash
mactime -b body.txt -d > timeline.csv
```

| Flag | Meaning |
|------|---------|
| `-b body.txt` | Input body file from `fls` |
| `-d` | Output as CSV (comma-delimited) |

### Step 5 — Spot the Anomaly

```
Tue Jan 01 1985 22:00:00  ...  4945  "/bin/bcab"   ← ANOMALY
Mon Oct 18 2021 22:54:17  ...                       ← normal
Mon Oct 18 2021 22:54:17  ...                       ← normal
```

Every file is dated 2021–2023 **except one** timestamped **1985**. Suspicious file: `/bin/bcab` at inode `4945`.

### Step 6 — Extract the File by Inode

```bash
icat partition4.img 4945
# NzFtMzExbjNfMHU3MTEzcl9oM3JfNDNhMmU3YWYK
```

### Step 7 — Decode Base64

```bash
echo "NzFtMzExbjNfMHU3MTEzcl9oM3JfNDNhMmU3YWYK" | base64 -d
# 71m311n3_0u7113r_h3r_43a2e7af
```

### Tool Reference

| Tool | Purpose |
|------|---------|
| `file` | Identify image type |
| `binwalk -e` | Extract file contents (insufficient here) |
| `fls -m '/' -r` | List files + metadata → body file |
| `mactime -b -d` | Build CSV timeline from body file |
| `icat <image> <inode>` | Extract file by inode number |
| `base64 -d` | Decode base64 string |

---

