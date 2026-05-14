## Disko 2:
- unzip the downloaded file using gunzip 
- binwalk -e disko-2.dd
- ❯ ls  
	FAT32_partition.1  Linux_partition.0
-  strings Linux_partition.0  | grep "picoCTF"
## Disko 3:
- gunzip disko-3.dd.gz
- binwalk -e disko-3.dd 
- extractions/disko-3.dd.extracted/0/rootfs/log/
	- unzip flag.gz
	- the flag is in this plain text document
## Disko 4:
-  gunzip disko-4.dd.gz
-  binwalk -e disko-4.dd
-    fls -r -d disko-4.dd    
	r/r * 522629:   log/messages  
	r/r * 532021:   log/dont-delete.gz
- icat disko-4.dd 532021 > messages2
	 unzip the messages2 file and get the flag
## Bitlocker-1:
❯ sudo fdisk -l bitlocker-1.dd 


Disk bitlocker-1.dd: 100 MiB, 104857600 bytes, 204800 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x78787878

Device          Boot      Start        End    Sectors   Size Id Type
bitlocker-1.dd1      2021161080 4042322159 2021161080 963.8G 78 unknown
bitlocker-1.dd2      2021161080 4042322159 2021161080 963.8G 78 unknown
bitlocker-1.dd3      4294932600 8589899894 4294967295     2T 78 unknown
bitlocker-1.dd4      4294967295 5035196669  740229375   353G ff BBT



❯ bitlocker2john -i bitlocker-1.dd > bitlocker1.txt


$bitlocker$0$16$cb4809fe9628471a411f8380e0f668db$1048576$12$d04d9c58eed6da010a000000$60$68156e51e53f0a01c076a32ba2b2999afffce8530fbe5d84b4c19ac71f6c79375b87d40c2d871ed2b7b5559d71ba31b6779c6f41412fd6869442d66d
and so on

add all the hashes to hashes.txt and get rockyou.txt 
❯ wget https://github.com/brannondorsey/naive-hashcat/releases/download/data/rockyou.txt

❯ hashcat -m 22100 hashes.txt rockyou.txt

response from hashcat: 
$bitlocker$0$16$cb4809fe9628471a411f8380e0f668db$1048576$12$d04d9c58eed6da010a000000$60$68156e51e53f0a01c076a32ba2b2999afffce8530fbe5d84b4c19ac71f6c79375b87d40c2d871ed2b7b5559d71ba31b6779c6f41412fd6869442d66d:jacqueline

jacqueline

❯ mkdir mountedimg

❯ sudo dislocker -V bitlocker-1.dd  -u"jacqueline" -- mountedimg

❯ sudo mount -o loop path/to/dislocker-file path/to/bitlocker_data

a flag.txt exists in the mounted image 


## Bitlocker-2:
-  Download memory dump file 
- strings memdump.mem | grep "picoCTF"
	picoCTF{B1tl0ck3r_dr1v3_d3crypt3d_9029ae5b}  
	picoCTF{B1tl0ck3r_dr1v3_d3crypt3d_9029ae5b}  
	picoCTF{B1tl0ck3r_dr1v3_d3crypt3d_9029ae5b}
I'm pretty sure this is not how they intended us to solve it but oh well

## Blast from the past:
