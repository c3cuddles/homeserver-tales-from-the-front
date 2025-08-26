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

That really doesn't seem right... I'm pretty sure my first partition started at sector 2048, because that's the default of most tools I've seen. Let's take a look at what's actually at sector 2048:

```bash
2025-07-18 06:46:58 riley-server ~ » sudo dd if=/dev/sdg bs=512 skip=2048 count=1 | hexdump -C | head

[sudo] password for riley:
1+0 records in
1+0 records out
512 bytes copied, 0.000735739 s, 696 kB/s
00000000  eb 52 90 4e 54 46 53 20  20 20 20 00 02 08 00 00  |.R.NTFS    .....|
00000010  00 00 00 00 00 f8 00 00  3f 00 ff 00 00 08 00 00  |........?.......|
00000020  00 00 00 00 80 00 80 00  f8 7f e0 e8 00 00 00 00  |................|
00000030  00 00 0c 00 00 00 00 00  02 00 00 00 00 00 00 00  |................|
00000040  f6 00 00 00 01 00 00 00  5f 78 85 0a ba 85 0a b6  |........_x......|
00000050  00 00 00 00 fa 33 c0 8e  d0 bc 00 7c fb 68 c0 07  |.....3.....|.h..|
00000060  1f 1e 68 66 00 cb 88 16  0e 00 66 81 3e 03 00 4e  |..hf......f.>..N|
00000070  54 46 53 75 15 b4 41 bb  aa 55 cd 13 72 0c 81 fb  |TFSu..A..U..r...|
00000080  55 aa 75 06 f7 c1 01 00  75 03 e9 dd 00 1e 83 ec  |U.u.....u.......|
00000090  18 68 1a 00 b4 48 8a 16  0e 00 8b f4 16 1f cd 13  |.h...H..........|
```

Aha! That looks like the *actual* beginning of my partition. `eb 52 90` followed by ascii `NTFS` indicates the beginning of the boot sector. Fun fact: `eb 52 90` is actually the jump instruction telling the computer that is booting to skip over the Bios Parameter Block and on to the rest of the code needed to boot. [Here's some more info](https://thestarman.pcministry.com/asm/mbr/NTFSBR.htm)

I think I know enough to recover my partition, but my curiosity got the better of me... So I dumped the first 100MB of the disk and opened it up in ImHex, my favorite hex editor.

```
2025-07-18 07:12:03 riley-server ~ » sudo dd if=/dev/sdg of=/media/riley/riley-storage/replacable/portable-drive-offload/disk-first-100mb.img bs=1024 count=100000
```

The first thing that caught my eye was the content of an old journal entry from circa 2015:

```bash
2025-07-18 04:29:21 riley-server replacable/portable-drive-offload » hexdump -C ./disk-first-100mb.img --skip 0x126000 --length $(( 16 * 6 )) # I wanted 6 lines and they are 16 bytes each!!!
00126000  42 45 47 49 4e 3a 56 4e  4f 54 45 0d 0a 56 45 52  |BEGIN:VNOTE..VER|
00126010  53 49 4f 4e 3a 31 2e 31  0d 0a 42 4f 44 59 3b 43  |SION:1.1..BODY;C|
00126020  48 41 52 53 45 54 3d 55  54 46 2d 38 3b 45 4e 43  |HARSET=UTF-8;ENC|
00126030  4f 44 49 4e 47 3d 51 55  4f 54 45 44 2d 50 52 49  |ODING=QUOTED-PRI|
00126040  4e 54 41 42 4c 45 3a 54  6f 64 61 79 20 49 20 6d  |NTABLE:Today I m|
00126050  65 74 20 77 69 74 68 20  54 6f 6e 79 20 61 67 61  |et with Tony aga|
```

I thought this was odd... Is NTFS really so simple that the content of a file is just exposed to the disk, raw? But I did some research and I found out that in NTFS, small files have their data stored directly in the MFT (Master File Table) record!!! So the data of a file is stored *as metadata*. Isn't that wild? This is called a "Resident File" and we can see here that this is a resident file because of the text `BEGIN:VNOTE` indicates that this is a resident file. Normal files, where the MFT just has a pointer to data, are called (conveniently) non-resident files.

Okay, I want to explore more, but first let's get a working filesystem back.

```bash
2025-07-18 07:25:23 riley-server ~ » sudo parted /dev/sdg
GNU Parted 3.5
Using /dev/sdg
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) mklabel msdos
Warning: The existing disk label on /dev/sdg will be destroyed and all data on this disk will be lost. Do you want to
continue?
Yes/No? Yes
(parted) mkpart primary ntfs 2048s 100%
(parted) quit
Information: You may need to update /etc/fstab.
2025-07-18 07:26:36 riley-server ~ » sudo mount /dev/sdg1 /mnt/portable-drive
2025-07-18 07:26:44 riley-server ~ » lsl /mnt/portable-drive
total 6.0M
drwxrwxrwx 1 root root  12K Jul 15 21:15  .
drwxr-xr-x 1 root root  102 Jul 21 11:51  ..
drwxrwxrwx 1 root root    0 May 23 21:19 '$RECYCLE.BIN'
drwxrwxrwx 1 root root 4.0K Aug 29  2024  archive-old-as-of-2024-08-29
drwxrwxrwx 1 root root 4.0K May 26 18:55 'AURORA I LOVE U'
drwxrwxrwx 1 root root    0 Jun  8 20:51  backups
-rwxrwxrwx 2 root root   56 Apr  9 16:12  .dropbox.device
drwxrwxrwx 1 root root    0 May 23 22:18  msdownld.tmp
drwxrwxrwx 1 root root 4.0K Jul  3 07:13  roms
drwxrwxrwx 1 root root 4.0K Jan 11  2025  server-backups
-rwxrwxrwx 1 root root 5.9M Jul  4 15:11  setup.exe
drwxrwxrwx 1 root root 4.0K Apr  9 22:09 'System Volume Information'
drwxrwxrwx 1 root root 4.0K Jul  2 17:15  Videos
```

There we are!!! All my files are back, right where I left them :) And everything is readable and intact:

```
2025-07-29 07:28:19 riley-server ~ » cat '/mnt/portable-drive/AURORA I LOVE U/README MY DARLING.txt'
hello :3

The folder "Avowed" contains your save games. This folder should be copied to: C:\Users\[Your Username]\Saved Games\Avowed.

Note that you will just copy the "Avowed" folder from the portable hard drive to the "Saved Games" folder on your computer, like don't make another "Avowed" folder to put the "Avowed" folder from the portable hard drive into :P
```

Okay, I'm so excited about forensics now... what else can we play around with? Oh wait! I should actually properly fix the drive with `ntfsfix` first... Let's make sure that we supply the *partition* this time. Oh, and let's give it then `-n` (`--no-action`, do nothing) flag for good measure.

```
2025-08-17 16:21:19 riley-server ~ » sudo ntfsfix -n /dev/sda1                                                   1 ↵
[sudo] password for riley:
Mounting volume... OK
Processing of $MFT and $MFTMirr completed successfully.
Checking the alternate boot sector... OK
NTFS volume version is 3.1.
NTFS partition /dev/sda1 was processed successfully.
```

The drive is now at `/dev/sda` because I did a reboot since working on this project last :) Okay, perhaps we don't need to do anything? Running `ntfsfix` without `-n` gave me the same output. 

Now we can play :) I wanna dump the MFT to disk and explore it a bit...

```
brew install sleuthkit # i am using linuxbrew
pip install analyzeMFT
```

I know that my partition starts at sector 2048, so it's easy to produce an icat command:

```
sudo /home/linuxbrew/.linuxbrew/bin/icat -o 2048 /dev/sda 0 > /media/riley/riley-storage/replacable/portable-drive-offload/mft.raw
```

And convert it to something readable:

```
cd /media/riley/riley-storage/replacable/portable-drive-offload
# analyzemft can produce a lot of output, let's store that for later review with 'tee'
analyzemft -f ./mft.raw -o ./mft.csv | tee ./mftanalyze.log
```

Python is slow and nearly always sequential, so now is a good time for a smoke break...

Exploring the csv shows we have quite a lot of attributes for each file:

```
2025-08-25 23:48:35 riley-server replacable/portable-drive-offload » head -1 mft.csv                                            1 ↵
Record Number,Record Status,Record Type,File Type,Sequence Number,Parent Record Number,Parent Record Sequence Number,Filename,Filepath,SI Creation Time,SI Modification Time,SI Access Time,SI Entry Time,FN Creation Time,FN Modification Time,FN Access Time,FN Entry Time,Object ID,Birth Volume ID,Birth Object ID,Birth Domain ID,Has Standard Information,Has Attribute List,Has File Name,Has Volume Name,Has Volume Information,Has Data,Has Index Root,Has Index Allocation,Has Bitmap,Has Reparse Point,Has EA Information,Has EA,Has Logged Utility Stream,Attribute List Details,Security Descriptor,Volume Name,Volume Information,Data Attribute,Index Root,Index Allocation,Bitmap,Reparse Point,EA Information,EA,Logged Utility Stream,MD5,SHA256,SHA512,CRC32
```

Even checksums? I didn't know ntfs stored file checksums.

It looks like the first file I ever created, besides the system-created things like the MFT and the folders I wanted, was an `.odt` file with the name `2015-07-07.odt`

```
63   Valid     In Use    File 1    62   0    2015-7-7.odt        2016-11-04T23:44:20.689Z 2015-07-09T00:03:02.000Z 2016-11-04T23:44:20.689Z 2016-11-04T23:44:20.689Z 2016-11-04T23:44:20.689Z 2016-11-04T23:44:20.689Z 2016-11-04T23:44:20.689Z 2016-11-04T23:44:20.689Z                     TRUE FALSE     TRUE FALSE     FALSE     TRUE FALSE     FALSE     FALSE     FALSE     FALSE     FALSE     FALSE     []   None      None {'name': '', 'non_resident': True, 'content_size': None, 'start_vcn': 0, 'last_vcn': 22}  None None None None None None None                
```

The cool thing about having the mft is that we can open files by record number (similar to inode number in ext filesystems used by linux). Let's recover that:

```
2025-08-26 00:00:38 riley-server replacable/portable-drive-offload » sudo /home/linuxbrew/.linuxbrew/bin/icat -o 2048 /dev/sda 63 > recovered-files/2015-7-7.odt
```

Notice the "63" as an argument to icat: that is where you supply the record number: `icat -o 2048 /dev/sda 63`

On opening that file I discovered it was a journal entry about something VERY traumatic that happened to me, so I will not be sharing the contents here. But that's a cool thing to recover!!!

After that was a bunch of `.vrt` files, and I don't know what that is. I have an inkling they were also automatically generated. But the second oldest file was a README!

```
119  Valid     In Use    File 1    64   0    00 README.txt       2016-11-04T23:44:23.934Z 2015-12-07T18:37:38.000Z 2016-11-04T23:44:23.934Z 2016-11-04T23:44:23.934Z 2016-11-04T23:44:23.934Z 2016-11-04T23:44:23.934Z 2016-11-04T23:44:23.934Z 2016-11-04T23:44:23.934Z                     TRUE FALSE     TRUE FALSE     FALSE     TRUE FALSE     FALSE     FALSE     FALSE     FALSE     FALSE     FALSE     []   None      None {'name': '', 'non_resident': False, 'content_size': 30, 'start_vcn': None, 'last_vcn': None}   None None None None None None None                
```

Let's recover that...

```
2025-08-26 00:01:08 riley-server replacable/portable-drive-offload » sudo /home/linuxbrew/.linuxbrew/bin/icat -o 2048 /dev/sda 119 > 'recovered-files/00 README.md'
```

And explore:

```
2025-08-26 00:08:25 riley-server replacable/portable-drive-offload » cat 'recovered-files/00 README.md'
read these files as plain text%
```

Well, that was disappointing lol. But you see how we can use the mft to recover files from an ntfs partition!!! This may even be possible on a borked filesystem, but I'm not going to try that today. I need to wipe this drive and repurpose it for timemachine backups. It's old, but it will not be used frequently and I hope it will last a few more years.

See ya!

Cuddles



