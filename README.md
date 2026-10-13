# Recovering an ext4 Filesystem After a Power Loss Mid-Partition-Move

![Linux](https://img.shields.io/badge/Linux-ext4-informational)
![Tools](https://img.shields.io/badge/tools-GParted%20%7C%20e2fsck%20%7C%20dmsetup%20%7C%20dd-blue)
![Result](https://img.shields.io/badge/result-0%20files%20lost-brightgreen)
![Method](https://img.shields.io/badge/method-read--only%20first-orange)

A write-up of an incident on my dual-boot laptop (Dell Latitude E6530, 500 GB HDD, GPT, Windows + Ubuntu): the power died while GParted was moving and growing my Ubuntu root partition. I recovered the filesystem without losing data, without a backup image, and without ever running a "repair" on the damaged state.

---

## TL;DR

| | |
|---|---|
| **Goal** | Shrink Windows `C:` by 80 GB, grow Ubuntu `/dev/sda6` from 230 GiB to 310 GiB |
| **Failure** | Power loss during GParted's "move left and grow" |
| **Symptom** | `e2fsck` reported HTree errors; the old location had no valid superblock |
| **Root cause** | A left-move copies data forward in place, so the filesystem was split across two disk regions |
| **Fix** | Located the copy cut-off, stitched both halves into a read-only virtual device, verified it, finished the copy by hand, then grew the filesystem |
| **Outcome** | Clean `e2fsck`, 3,266,959 files intact, root now 305 GB with 83 GB free |

---

## 1. Setup

- Disk: GPT, 512-byte logical sectors.
- Layout after shrinking `C:` (sda4): `[ sda4 NTFS ][ 80 GiB unallocated ][ sda6 ext4 230 GiB ][ sda5 recovery ]`
- Because the free space sat **before** `sda6`, GParted had to change the partition's *start*, which forces a full data move. Growing to the right would have been a metadata-only resize.
- Windows `C:` first failed to shrink with a "volume may be corrupted" error, so I ran `chkdsk C: /f` before retrying.

## 2. What went wrong

GParted runs `e2fsck`, then moves the partition 80 GiB left, then resizes. It was part-way through the move when power was lost. After rebooting:

- GParted showed `sda6` at 310 GiB, but the filesystem inside still claimed 230 GiB.
- A read-only `e2fsck -fn /dev/sda6` printed HTree errors (invalid root node, unsupported hash version, absurd tree depth). Those are what you see when a filesystem is read from the wrong offsets.

**Decision:** do not click GParted's Check/Apply and do not run `e2fsck -y`. Auto-repair on a spliced filesystem would delete directory indexes irreversibly.

## 3. Diagnosis

### 3.1 Where is the old filesystem?

From `fdisk -l`, the new `sda6` started at sector `292204544` and ended at `942321663`. The end never moved, so the original 230 GiB filesystem started at:

```
942321664 - (230 GiB / 512) = 459976704   (sector)
459976704 * 512             = 235508072448 (byte offset)
```

I mounted a read-only loop view of that offset. Both the superblock and the backup superblocks failed. I read that as "the copy already overwrote the start of the old filesystem."

> **Correction:** the offset I first used in that test had an arithmetic error (235508584448 instead of 235508072448), so that particular negative result was not trustworthy on its own. The conclusion was confirmed later by the duplicate-block scan and the clean stitched check. See *Lessons*.

### 3.2 How far did the copy get?

A left-move copies forward, so each block ends up at two places, 80 GiB apart, until the old one is overwritten. I sampled one 4 KiB block per GiB at both positions and recorded where they matched (read-only):

```python
f = open('/dev/sda', 'rb')
N = 292204544 * 512      # new partition start (bytes)
S = 80 << 30             # 80 GiB distance
G = 1 << 30
hits = []
for i in range(231):
    f.seek(N + i*G);     a = f.read(4096)
    f.seek(N + i*G + S); b = f.read(4096)
    if a == b and any(a):
        hits.append(i)
print(hits)
```

Result: a continuous run from GiB 43 to GiB 122. That is 80 consecutive samples, meaning the copy had reached about **123 GiB of 230 GiB** when power failed. The first ~123 GiB were valid at the new location, and everything past ~43 GiB was still valid at the old one.

## 4. Recovery

### 4.1 Stitch the two halves (read-only)

Using device-mapper, I built a virtual read-only device: the first 80 GiB from the new location and the remaining 150 GiB from the old one:

```bash
printf '0 167772160 linear /dev/sda 292204544\n167772160 314572800 linear /dev/sda 627748864\n' \
  | sudo dmsetup create --readonly rescue

sudo dumpe2fs -h /dev/mapper/rescue | head -30
sudo e2fsck -fn /dev/mapper/rescue
sudo mount -o ro,noload /dev/mapper/rescue /mnt/rescue
```

`e2fsck -fn` passed all five passes with no errors (3,266,959 files, 55,185,505 of 60,293,120 blocks), and the home directory was visible. Nothing had been written to the disk at this point.

### 4.2 Finish the interrupted copy

From GiB 100 onward the data was already identical at both locations, so I resumed the copy from there. The write head stays 80 GiB behind the read head, so unread data is never overwritten:

```bash
O=$((459976704*512)); N=$((292204544*512)); G=$((100*2**30))
sudo dd if=/dev/sda of=/dev/sda bs=64M \
  iflag=skip_bytes,count_bytes oflag=direct,seek_bytes \
  skip=$((O+G)) seek=$((N+G)) count=$((130*2**30)) status=progress
```

Output: 139,586,437,120 bytes (130 GiB) copied in 3,459 s at 40.4 MB/s, 2080 records in and 2080 out.

### 4.3 Verify, then grow

```bash
sync
sudo e2fsck -fn /dev/sda6     # clean, same file and block counts as the stitched check
sudo resize2fs /dev/sda6      # 81264640 (4k) blocks
```

Booted into Ubuntu: `df -h /` showed 305G total, 207G used, 83G available.

## 5. Lessons learned

1. **Stop writing the moment something goes wrong.** Every command until the final copy was read-only (`-n`, `-r`, `--readonly`, `ro,noload`).
2. **Never auto-repair an unfamiliar corruption state.** `e2fsck -y` is not a diagnostic tool.
3. **Reason from the on-disk layout.** Knowing that a left-move copies forward made recovery possible.
4. **Verify arithmetic and don't trust a single result.** One wrong offset gave a misleading test; independent evidence (duplicate scan, stitched fsck) caught it.
5. **Prevention:** back up before partition surgery, have the machine on charger with a healthy battery, and prefer layouts where the free space is adjacent on the growing side so the operation is a cheap resize rather than a data move.

## 6. Limitations

- Verification was filesystem-level (`e2fsck` clean plus a mount and directory listing). I did not checksum individual files against a prior backup, because none existed.
- This procedure fit this exact failure (a left-move interrupted partway). Different cut-offs or a failing disk would need different math, and imaging the disk first is the safer default.
- Do not run these commands on a disk you have not read the layout of: `dd` with the same device as input and output is destructive if the offsets are wrong.

## 7. Skills demonstrated

Filesystem internals (ext4, GPT, sector math) · incident response · read-only forensics · device-mapper · `dd` offsets · scripting a diagnostic in Python · verification discipline.
