---
title: Fixing (and recovering!!!) ntfs partition
updated: 2025-07-18 08:58:57Z
created: 2025-07-18 08:48:07Z
latitude: 30.26715300
longitude: -97.74306080
altitude: 0.0000
---

I was offloading data from an aging portable hard drive, so I could retire it to a secondary timeMachine backup device that would see fewer reads and writes. I discovered while `rsync` ing all the data to my zfs RAIDZ1 array that I got input/output errors while accessing a specific folder. Here I unsuccessfully try to delete it:

```bash
2025-07-18 03:16:18 riley-server replacable/portable-drive-offload » sudo sudo ls -alh /mnt/portable-drive/archive-old-as-of-2024-08-29/Software/games/World\ of\ Warcraft/Data
total 172K
drwxrwxrwx 1 root root    0 May  6 10:01 .
drwxrwxrwx 1 root root 4.0K May  6 10:01 ..
drwxrwxrwx 1 root root 168K May  6 10:01 data
2025-07-18 03:16:20 riley-server replacable/portable-drive-offload » sudo sudo ls -alh /mnt/portable-drive/archive-old-as-of-2024-08-29/Software/games/World\ of\ Warcraft/Data/data
ls: reading directory '/mnt/portable-drive/archive-old-as-of-2024-08-29/Software/games/World of Warcraft/Data/data': Input/output error
total 0
2025-07-18 03:16:21 riley-server replacable/portable-drive-offload » sudo rm -rf /mnt/portable-drive/archive-old-as-of-2024-08-29/Software/games/World\ of\ Warcraft/Data/data
rm: cannot remove '/mnt/portable-drive/archive-old-as-of-2024-08-29/Software/games/World of Warcraft/Data/data': Directory not empty
```

I went to investigate it for problems. It was old, so I thought it might be failing. But dmesg didn't mention it at all! And `smartctl` was unfortunately useless:

```bash
2025-07-18 03:18:01 riley-server replacable/portable-drive-offload » sudo smartctl -a /dev/sdg
zsh: command not found: 2025-07-18
grep: 1: No such file or directory
grep: ↵: No such file or directory
zsh: no matches found: [452687.335016]
2025-07-18 03:19:33 riley-server replacable/portable-drive-offload » sudo smartctl -a /dev/sdg                       2 ↵
smartctl 7.3 2022-02-28 r5338 [x86_64-linux-6.1.0-37-amd64] (local build)
Copyright (C) 2002-22, Bruce Allen, Christian Franke, www.smartmontools.org

Read Device Identity failed: scsi error unsupported field in scsi command

If this is a USB connected device, look at the various --device=TYPE variants
A mandatory SMART command failed: exiting. To continue, add one or more '-T permissive' options.
```

Next, I unmounted it and naively tried `fsck -p`, but this is an `ntfs` partition so it did not work. I did, however, discover a tool `ntfsfix`!!! Ah, surely my problems are nearly over, let's just run...

```bash
2025-07-18 03:41:47 riley-server replacable/portable-drive-offload » sudo sudo ntfsfix /dev/sdg
Mounting volume... NTFS signature is missing.
FAILED
Attempting to correct errors... NTFS signature is missing.
FAILED
Failed to startup volume: Invalid argument
NTFS signature is missing.
Trying the alternate boot sector
The alternate bootsector is usable
Set sector count to 3907029166 instead of 3907022847
Rewriting the bootsector
The boot sector has been rewritten
ntfs_mst_post_read_fixup_warn: magic: 0x78cb7ff2  size: 1024   usa_ofs: 15596  usa_count: 1361: Invalid argument
Record 0 has no FILE magic (0x78cb7ff2)
Failed to load $MFT: Input/output error
Volume is corrupt. You should run chkdsk.
2025-07-18 03:43:57 riley-server replacable/portable-drive-offload » sudo fdisk -l /dev/sdg
Disk /dev/sdg: 1.82 TiB, 2000398933504 bytes, 3907029167 sectors
Disk model: BUP Slim RD
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: dos
Disk identifier: 0x6e697373

Device     Boot      Start        End    Sectors   Size Id Type
/dev/sdg1       1936269394 3772285809 1836016416 875.5G 4f QNX4.x 3rd part
/dev/sdg2       1917848077 2462285169  544437093 259.6G 73 unknown
/dev/sdg3       1818575915 2362751050  544175136 259.5G 2b unknown
/dev/sdg4       2844524554 2844579527      54974  26.8M 61 SpeedStor

Partition 1 does not start on physical sector boundary.
Partition 2 does not start on physical sector boundary.
Partition 3 does not start on physical sector boundary.
Partition 4 does not start on physical sector boundary.
Partition table entries are not in disk order.
```

Oh no! I've fucked something up *BAD*. `ntfsfix` absolutely murdered my partition table! What a horrible thing for a piece of software to do with no warning! I did not even supply a flag like `-y` or `--quiet`. Turns out `ntfsfix` *assumes* without warning that it was supplied an ntfs *partition*, and not a disk. Well. I should have read the documentation instead of charging in! I learned that lesson the hard way. Luckily, I think I can recover it, it was just one giant ntfs partition. And, besides, I just rsync'd the entire contents to my raid array. So it's fine. But I want to try to recover it. 

I whipped out `testdisk`, told it there was an `Intel` style partition table, and then asked it to `Analyse` my disk. I think it may have picked up the bad partitions `ntfsfix` dumped into the table...

```
TestDisk 7.1, Data Recovery Utility, July 2019
Christophe GRENIER <grenier@cgsecurity.org>
https://www.cgsecurity.org

Disk /dev/sdg - 2000 GB / 1863 GiB - CHS 243201 255 63
Current partition structure:
     Partition                  Start        End    Size in sectors

 1 * Sys=4F               120527  49 53 234813 237 34 1836016416

Bad relative sector.
 2 * Sys=73               119380 132 62 153270  41 37  544437093

Bad relative sector.
 3 * Sys=2B               113201  29 24 147074 114 59  544175136

Bad relative sector.
 4 * SpeedStor            177063 118 26 177066 225 63      54974

Bad relative sector.
Only one partition must be bootable
Space conflict between the following two partitions
 3 * Sys=2B               113201  29 24 147074 114 59  544175136
 2 * Sys=73               119380 132 62 153270  41 37  544437093
Space conflict between the following two partitions
 2 * Sys=73               119380 132 62 153270  41 37  544437093
 1 * Sys=4F               120527  49 53 234813 237 34 1836016416
Space conflict between the following two partitions
 1 * Sys=4F               120527  49 53 234813 237 34 1836016416
 4 * SpeedStor            177063 118 26 177066 225 63      54974

*=Primary bootable  P=Primary  L=Logical  E=Extended  D=Deleted
```

