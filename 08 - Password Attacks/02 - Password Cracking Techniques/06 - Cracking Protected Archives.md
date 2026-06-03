# Cracking Protected Archives

Crack ZIP:

```bash
zip2john <FILE>.zip > zip.hash
john --wordlist=<WORDLIST> zip.hash
john zip.hash --show
```

OpenSSL-encrypted GZIP — `file` first to confirm, then brute with openssl loop (errors are noise, ignore them):

```bash
file <FILE>.gzip
for i in $(cat <WORDLIST>);do openssl enc -aes-256-cbc -d -in <FILE>.gzip -k $i 2>/dev/null| tar xz;done
```

BitLocker — extract hashes, grep for the password hash only (`$bitlocker$0$`), mode 22100 is slow on AES:

```bash
bitlocker2john -i <FILE>.vhd > backup.hashes
grep "bitlocker\$0" backup.hashes > backup.hash
hashcat -a 0 -m 22100 backup.hash <WORDLIST>
```

Mount BitLocker on Linux after cracking:

```bash
sudo losetup -f -P <FILE>.vhd
sudo dislocker /dev/loop0p2 -u<PASS> -- /media/bitlocker
sudo mount -o loop /media/bitlocker/dislocker-file /media/bitlockermount
```

Unmount:

```bash
sudo umount /media/bitlockermount && sudo umount /media/bitlocker
```

> [!tip] `locate *2john*` covers zip, rar, 7z, keepass, bitlocker — same extract → crack pipeline for all of them. Unknown archive format → `file <ARCHIVE>` before picking the tool.