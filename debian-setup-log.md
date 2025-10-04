2025-09-25

** Setup log: **

# partitioning

```
user@debian:/media/user/Ventoy/files$ sudo fdisk -l /dev/sdd
Disk /dev/sdd: 931.51 GiB, 1000204886016 bytes, 1953525168 sectors
Disk model: PNY CS900 1TB SS
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 7FC4614F-2CE7-4FC9-B310-0E2D9644B995

Device        Start        End    Sectors   Size Type
/dev/sdd1  20973568 1953523711 1932550144 921.5G Linux filesystem
```

# encryption

set up encryption with the following command:

```
sudo cryptsetup --cipher aes-xts-plain64 --key-size 512 --hash sha256 --iter-time 5000 --use-random --verify-passphrase luksFormat /dev/sdd1
```

errata:
- --cipher is technically unnecessary because this is the current default, but I prefer to be explicit
- "The --use-random specifies to use /dev/random as an entropy (randomness) source as opposed to the default of /dev/urandom. The reason for this is that on a fresh, live environment, /dev/urandom may not have suitable entropy." <- from https://kneit.in/2015/09/17/brtfs-raid-on-dmcrypt.html#setting-up-dm-crypt

opened encryption:
```
sudo cryptsetup open /dev/sdd1 debian-luks 
```

in retrospect, I think it would have made more sense to name it `debian-btrfs-pool`. Ah well.

# creating the btrfs pool

```
user@debian:/media/user/Ventoy/files$ sudo mkfs.btrfs -L debian-btrfs-pool /dev/mapper/debian-luks
btrfs-progs v6.14
See https://btrfs.readthedocs.io for more information.

NOTE: several default settings have changed in version 5.15, please make sure
      this does not affect your deployments:
      - DUP for metadata (-m dup)
      - enabled no-holes (-O no-holes)
      - enabled free-space-tree (-R free-space-tree)

Label:              debian-btrfs-pool
UUID:               a9576c4d-8c49-42b0-954e-d52e81f4f0f0
Node size:          16384
Sector size:        4096        (CPU page size: 4096)
Filesystem size:    921.50GiB
Block group profiles:
  Data:             single            8.00MiB
  Metadata:         DUP               1.00GiB
  System:           DUP               8.00MiB
SSD detected:       yes
Zoned device:       no
Features:           extref, skinny-metadata, no-holes, free-space-tree
Checksum:           crc32c
Number of devices:  1
Devices:
   ID        SIZE  PATH
    1   921.50GiB  /dev/mapper/debian-luks
```


# creating btrfs subvolumes

first, mount the currently existing filesystem

```
sudo mount /dev/mapper/debian-luks /mnt
cd /mnt
```

when I created the root subvolume, it changed the name on me...

```
user@debian:/mnt$ sudo btrfs subvolume create @/
Create subvolume './@'
user@debian:/mnt$ sudo btrfs subvolume create @/boot
Create subvolume '@/boot'
user@debian:/mnt$ sudo btrfs subvolume list .
ID 256 gen 10 top level 5 path @
ID 257 gen 11 top level 256 path @/boot

```

maybe I should redo it and name it @root? idk...

Strange that @/boot was not changed....


trying to create the subvolumes within a directory did not work, because the parent directory didn't exist. Will have to do that later:

```
user@debian:/mnt$ sudo btrfs subvolume create @/boot/grub2/x86_64-efi
ERROR: Could not open: No such file or directory
```

I'm not confident about my naming.... should I be using slashes in the names?

anyways, for now I created the rest in one command

```
user@debian:/mnt$ sudo btrfs subvolume create @/home @/opt @/root @/srv @/tmp @/var
```

remaining to do:
- /boot/grub2/x86_64-efi
- /usr/local

# REDO! recreating btrfs subvolumes

on further review, I discovered I have nested my subvolumes.

```
user@debian:/mnt$ sudo btrfs subvolume list . -t
ID      gen     top level       path
--      ---     ---------       ----
256     13      5               @
257     11      256             @/boot
258     12      256             @/home
259     13      256             @/opt
260     13      256             @/root
261     13      256             @/srv
262     13      256             @/tmp
263     13      256             @/var
```

I have heard this makes snapshotting a pain, and it's easier if each subvolume is a sibling of the others. Let's try this again, without using any '/' in my subvolume names. I'm actually going to wipe the whole filesystem and start over from scratch.

```
user@debian:/mnt$ cd /
user@debian:/$ sudo umount /mnt
user@debian:/$ sudo wipefs -a /dev/mapper/debian-luks
/dev/mapper/debian-luks: 8 bytes were erased at offset 0x00010040 (btrfs): 5f 42 48 52 66 53 5f 4d
user@debian:/$ sudo mkfs.btrfs -L debian-btrfs-pool /dev/mapper/debian-luks
...
user@debian:/$ sudo mount /dev/mapper/debian-luks /mnt
user@debian:/$ cd /mnt
user@debian:/mnt$ sudo btrfs subvolume create root boot boot-grub2-x86_64-efi home opt root-home srv tmp usr-local var
Create subvolume './root'
Create subvolume './boot'
Create subvolume './boot-grub2-x86_64-efi'
Create subvolume './home'
Create subvolume './opt'
Create subvolume './root-home'
Create subvolume './srv'
Create subvolume './tmp'
Create subvolume './usr-local'
Create subvolume './var'
user@debian:/mnt$ sudo btrfs subvolume list . -t
ID      gen     top level       path
--      ---     ---------       ----
256     10      5               root
257     10      5               boot
258     10      5               boot-grub2-x86_64-efi
259     10      5               home
260     10      5               opt
261     10      5               root-home
262     10      5               srv
263     10      5               tmp
264     10      5               usr-local
265     10      5               var

```

Now it looks like everything is on the same level!

# mounting subvolumes in proper places

start by mounting the new root volume:

```
cd /
sudo umount /mnt
sudo mount -t btrfs -o defaults,noatime,ssd,compress=zstd,subvol=root /dev/mapper/debian-luks /mnt
```

then create directories for the other subvolumes:

```
user@debian:/mnt$ sudo mkdir /mnt/{boot,home,opt,root,srv,tmp,var}
user@debian:/mnt$ sudo mkdir -p /usr/local
```

`/boot/grub2/x86_64-efi` will be created after the `/boot` subvolume is mounted

for the simple volumes where the subvol name matches the path, we can mount them all at once with a bash for loop:

```
for vol in boot home opt srv tmp var ; do sudo mount -t btrfs -o defaults,noatime,ssd,compress=zstd,subvol=$vol /dev/mapper/debian-luks /mnt/$vol ; done
```

then we can handle the edge cases manually. First I'll compare what we have mounted with the full list of subvolumes to remind myself what I'm missing:

```
user@debian:/mnt$ findmnt -nt btrfs
/mnt        /dev/mapper/debian-luks[/root] btrfs rw,noatime,compress=zstd:3,ssd,space_cache=v2,subvolid=256,subvol=/root
├─/mnt/boot /dev/mapper/debian-luks[/boot] btrfs rw,noatime,compress=zstd:3,ssd,space_cache=v2,subvolid=257,subvol=/boot
├─/mnt/home /dev/mapper/debian-luks[/home] btrfs rw,noatime,compress=zstd:3,ssd,space_cache=v2,subvolid=259,subvol=/home
├─/mnt/opt  /dev/mapper/debian-luks[/opt]  btrfs rw,noatime,compress=zstd:3,ssd,space_cache=v2,subvolid=260,subvol=/opt
├─/mnt/srv  /dev/mapper/debian-luks[/srv]  btrfs rw,noatime,compress=zstd:3,ssd,space_cache=v2,subvolid=262,subvol=/srv
├─/mnt/tmp  /dev/mapper/debian-luks[/tmp]  btrfs rw,noatime,compress=zstd:3,ssd,space_cache=v2,subvolid=263,subvol=/tmp
└─/mnt/var  /dev/mapper/debian-luks[/var]  btrfs rw,noatime,compress=zstd:3,ssd,space_cache=v2,subvolid=265,subvol=/var
user@debian:/mnt$ sudo btrfs subvolume list /mnt -t
ID      gen     top level       path
--      ---     ---------       ----
256     12      5               root
257     10      5               boot
258     10      5               boot-grub2-x86_64-efi
259     10      5               home
260     10      5               opt
261     10      5               root-home
262     10      5               srv
263     10      5               tmp
264     10      5               usr-local
265     10      5               var
```

so it looks like the subvolumes we still need to mount are:
- boot-grub2-x86_64-efi
- root-home
- usr-local


```
user@debian:/mnt$ sudo mkdir -p /mnt/boot/grub2/x86_64-efi
user@debian:/mnt$ sudo mount -t btrfs -o defaults,noatime,ssd,compress=zstd,subvol=boot-grub2-x86_64-efi /dev/mapper/debian-luks /mnt/boot/grub2/x86_64-efi
user@debian:/mnt$ sudo mount -t btrfs -o defaults,noatime,ssd,compress=zstd,subvol=root-home /dev/mapper/debian-luks /mnt/root
user@debian:/mnt$ sudo mkdir -p /mnt/usr/local
user@debian:/mnt$ sudo mount -t btrfs -o defaults,noatime,ssd,compress=zstd,subvol=usr-local /dev/mapper/debian-luks /mnt/usr/local
```

that should be everything... let's double-check though.

```
user@debian:/mnt$ findmnt -nt btrfs
/mnt                    /dev/mapper/debian-luks[/root]                  btrfs rw,noatime,compress=zstd:3,ssd,space_cache=v2,subvolid=256,subvol=/root
├─/mnt/root             /dev/mapper/debian-luks[/root-home]             btrfs rw,noatime,compress=zstd:3,ssd,space_cache=v2,subvolid=261,subvol=/root-home
├─/mnt/boot             /dev/mapper/debian-luks[/boot]                  btrfs rw,noatime,compress=zstd:3,ssd,space_cache=v2,subvolid=257,subvol=/boot
│ └─/mnt/boot/grub2/x86_64-efi
│                       /dev/mapper/debian-luks[/boot-grub2-x86_64-efi] btrfs rw,noatime,compress=zstd:3,ssd,space_cache=v2,subvolid=258,subvol=/boot-grub2-
├─/mnt/home             /dev/mapper/debian-luks[/home]                  btrfs rw,noatime,compress=zstd:3,ssd,space_cache=v2,subvolid=259,subvol=/home
├─/mnt/opt              /dev/mapper/debian-luks[/opt]                   btrfs rw,noatime,compress=zstd:3,ssd,space_cache=v2,subvolid=260,subvol=/opt
├─/mnt/srv              /dev/mapper/debian-luks[/srv]                   btrfs rw,noatime,compress=zstd:3,ssd,space_cache=v2,subvolid=262,subvol=/srv
├─/mnt/tmp              /dev/mapper/debian-luks[/tmp]                   btrfs rw,noatime,compress=zstd:3,ssd,space_cache=v2,subvolid=263,subvol=/tmp
├─/mnt/var              /dev/mapper/debian-luks[/var]                   btrfs rw,noatime,compress=zstd:3,ssd,space_cache=v2,subvolid=265,subvol=/var
└─/mnt/usr/local        /dev/mapper/debian-luks[/usr-local]             btrfs rw,noatime,compress=zstd:3,ssd,space_cache=v2,subvolid=264,subvol=/usr-local
```

Looks good! Let's continue. I'm going to disable CoW on /tmp and /var because they contain a lot of rotating files that change quickly, which would cause bloat if CoW were to remain enabled.

```
user@debian:/mnt$ sudo chattr +C /mnt/tmp
user@debian:/mnt$ sudo chattr +C /mnt/var
```

I have heard it is easier to create subvolumes when the entire btrfs pool is mounted (i.e. mount the mapper without any 'subvol' option). If I was to do that, I think I would do it at /mnt/btrfs-pool. Maybe something like `mount -t btrfs -o defaults,noatime,ssd,compress=zstd /dev/mapper/debian-luks /mnt/mnt/btrfs-pool`. I'm not going to do that now, just noting it down in case I need to create subvolumes in the future.

at this point I got really worried about the potential friction associated with having '/boot' on a btrfs subvolume, and decided to make a separate ext2 partition for boot.

```
cd /
# unmount everything...
user@debian:/mnt$ sudo umount /mnt/usr/local /mnt/boot/grub2/x86_64-efi /mnt/{root,var,tmp,srv,opt,home,boot} /mnt
# close the luks container
user@debian:/$ sudo cryptsetup luksClose /dev/mapper/debian-luks
# time to repartition!
user@debian:/$ sudo fdisk /dev/sdd
```

I was extremely paranoid so that I would not destroy all my hard work with btrfs

Here is the original partition scheme:

```
user@debian:/media/user/Ventoy/files$ sudo fdisk -l /dev/sdd
Disk /dev/sdd: 931.51 GiB, 1000204886016 bytes, 1953525168 sectors
Disk model: PNY CS900 1TB SS
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 7FC4614F-2CE7-4FC9-B310-0E2D9644B995

Device        Start        End    Sectors   Size Type
/dev/sdd1  20973568 1953523711 1932550144 921.5G Linux filesystem
```

I deleted that partition, and then recreated everything with two more partitions at the start (one for the ESP, one for `/boot`)

Here is the new partition scheme:

```
user@debian:/$ sudo fdisk -l /dev/sdd
Disk /dev/sdd: 931.51 GiB, 1000204886016 bytes, 1953525168 sectors
Disk model: PNY CS900 1TB SS
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 7FC4614F-2CE7-4FC9-B310-0E2D9644B995

Device        Start        End    Sectors   Size Type
/dev/sdd1      2048   10485760   10483713     5G EFI System
/dev/sdd2  10487808   20971520   10483713     5G Linux filesystem
/dev/sdd3  20973568 1953523711 1932550144 921.5G Linux filesystem
user@debian:/$ 
```

The important lines are copied below for easy comparison:

```
/dev/sdd1  20973568 1953523711 1932550144 921.5G Linux filesystem # old partition scheme
/dev/sdd3  20973568 1953523711 1932550144 921.5G Linux filesystem # new partition scheme
```

The sectors are exactly the same for the beginning, end, and size. Things should work just fine.

Note that for now, I will be leaving the ESP in /dev/sdd1 completely empty (I am not even going to make a filesystem) because the other operating systems on my computer (windows, nixos, my old debian install) use the ESP on my primary drive, an nvme SSD. However, this partition will remain here in case I ever want to abandon those other installs and move my ESP to this disk.

```
user@debian:/$ sudo cryptsetup open /dev/sdd3 debian-luks
Enter passphrase for /dev/sdd3: 
user@debian:/$ sudo mount -t btrfs -o defaults,noatime,ssd,compress=zstd,subvol=root /dev/mapper/debian-luks /mnt
user@debian:/$ ls /mnt
boot  home  opt  root  srv  tmp  usr  var
user@debian:/$ for vol in home opt srv tmp var ; do sudo mount -t btrfs -o defaults,noatime,ssd,compress=zstd,subvol=$vol /dev/mapper/debian-luks /mnt/$vol ; done
user@debian:/$ sudo mount -t btrfs -o defaults,noatime,ssd,compress=zstd,subvol=root-home /dev/mapper/debian-luks /mnt/root
user@debian:/$ sudo mount -t btrfs -o defaults,noatime,ssd,compress=zstd,subvol=usr-local /dev/mapper/debian-luks /mnt/usr/local
user@debian:/$ sudo mkfs.ext4 -L debian-boot /dev/sdd2
mke2fs 1.47.2 (1-Jan-2025)
Discarding device blocks: done                            
Creating filesystem with 1310464 4k blocks and 327680 inodes
Filesystem UUID: d699eed6-1529-4ebe-9890-0d08ffdf1e44
Superblock backups stored on blocks: 
        32768, 98304, 163840, 229376, 294912, 819200, 884736

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done 

user@debian:/$ sudo mount /dev/sdd2 /mnt/boot
user@debian:/$ sudo mkdir /mnt/boot/efi
user@debian:/$ sudo mount /dev/nvme0n1p1 /mnt/boot/efi
```

That should be everything! Let's double check...

```
user@debian:/$ findmnt
...
├─/mnt                     /dev/mapper/debian-luks[/root]      btrfs       rw,noatime,compress=zstd:3,ssd,space_cache=v2,subvolid=256,subvol=/root
│ ├─/mnt/root              /dev/mapper/debian-luks[/root-home] btrfs       rw,noatime,compress=zstd:3,ssd,space_cache=v2,subvolid=261,subvol=/root-home
│ ├─/mnt/home              /dev/mapper/debian-luks[/home]      btrfs       rw,noatime,compress=zstd:3,ssd,space_cache=v2,subvolid=259,subvol=/home
│ ├─/mnt/opt               /dev/mapper/debian-luks[/opt]       btrfs       rw,noatime,compress=zstd:3,ssd,space_cache=v2,subvolid=260,subvol=/opt
│ ├─/mnt/srv               /dev/mapper/debian-luks[/srv]       btrfs       rw,noatime,compress=zstd:3,ssd,space_cache=v2,subvolid=262,subvol=/srv
│ ├─/mnt/tmp               /dev/mapper/debian-luks[/tmp]       btrfs       rw,noatime,compress=zstd:3,ssd,space_cache=v2,subvolid=263,subvol=/tmp
│ ├─/mnt/var               /dev/mapper/debian-luks[/var]       btrfs       rw,noatime,compress=zstd:3,ssd,space_cache=v2,subvolid=265,subvol=/var
│ ├─/mnt/usr/local         /dev/mapper/debian-luks[/usr-local] btrfs       rw,noatime,compress=zstd:3,ssd,space_cache=v2,subvolid=264,subvol=/usr-local
│ └─/mnt/boot              /dev/sdd2                           ext4        rw,relatime
│   └─/mnt/boot/efi        /dev/nvme0n1p1                      vfat        rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=ascii,shortname=mixed,ut
...
user@debian:/$ sudo lsattr -d /mnt/var
---------------C------ /mnt/var
user@debian:/$ sudo lsattr -d /mnt/tmp
---------------C------ /mnt/tmp
```

Looks good! The "C" flag shown by `lsattr` indicates that CoW is disabled for `/var` and `/tmp`. Let's remove the old btrfs subvolumes. In order to delete these, there must be a path to the subvolume, so I am mounting the btrfs root system to `/mnt/btrfs-root`. This means I will need to mount the pool without the "subvol" option.

```
user@debian:/$ sudo mkdir /mnt/btrfs-root
user@debian:/$ sudo mount -t btrfs -o defaults,noatime,ssd,compress=zstd /dev/mapper/debian-luks /mnt/btrfs-root
user@debian:/$ sudo btrfs subvolume list /mnt/btrfs-root
ID 256 gen 104 top level 5 path root
ID 257 gen 13 top level 5 path boot
ID 258 gen 10 top level 5 path boot-grub2-x86_64-efi
ID 259 gen 102 top level 5 path home
ID 260 gen 103 top level 5 path opt
ID 261 gen 105 top level 5 path root-home
ID 262 gen 106 top level 5 path srv
ID 263 gen 107 top level 5 path tmp
ID 264 gen 108 top level 5 path usr-local
ID 265 gen 109 top level 5 path var
user@debian:/$ sudo btrfs subvolume delete -i 257 /mnt/btrfs-root/boot 
Delete subvolume 257 (no-commit): '/mnt/btrfs-root/boot/boot'
user@debian:/$ sudo btrfs subvolume delete -i 258 /mnt/btrfs-root/boot-grub2-x86_64-efi/
Delete subvolume 258 (no-commit): '/mnt/btrfs-root/boot-grub2-x86_64-efi//boot-grub2-x86_64-efi'
user@debian:/$ sudo btrfs subvolume list /mnt/btrfs-root/
ID 256 gen 104 top level 5 path root
ID 259 gen 102 top level 5 path home
ID 260 gen 103 top level 5 path opt
ID 261 gen 105 top level 5 path root-home
ID 262 gen 106 top level 5 path srv
ID 263 gen 107 top level 5 path tmp
ID 264 gen 108 top level 5 path usr-local
ID 265 gen 109 top level 5 path var

```

All clean! Let's proceed to the installation. I'll be following the manual installation steps specified [in the debian manual](https://www.debian.org/releases/stable/amd64/apds03.en.html).

```
user@debian:/mnt$ sudo debootstrap --arch amd64 trixie /mnt http://ftp.us.debian.org/debian
...
I: Base system installed successfully.
user@debian:/mnt$ sudo LANG=C.UTF-8 chroot /mnt /bin/bash
root@debian:/#
```

the /dev in our new chroot'ed system doesn't have much in it, and we need device files to continue!

```
root@debian:/# ls /dev
console  fd  full  null  ptmx  pts  random  shm  stderr  stdin  stdout  tty  urandom  zero
```

The debian manual cautious against bind-mounting the host /dev because the postinstall scripts may create files in /dev. However, this is targeted at users installing debian from another linux distribution (like RHEL). I am on a live USB and my system is transient, so it doesn't really matter if the installer mucks about with my device files. Following [the arch linux wiki](https://wiki.archlinux.org/title/Chroot#Using_chroot), I chose to do the following in a secondary terminal window:

```
user@debian:/media/user/Ventoy/files$ sudo mount --rbind /dev /mnt/dev
```

returning to the chrooted system, things look good!

```
root@debian:/# ls /dev
autofs           dri          hidraw6       loop2      nvme0n1p3  rtc0  sde       stderr  tty19  tty32  tty46  tty6     urandom      vcsa5        watchdog0
block            drm_dp_aux0  hidraw7       loop3      nvme0n1p4  sda   sde1      stdin   tty2   tty33  tty47  tty60    usb          vcsa6        zero
bsg              drm_dp_aux1  hidraw8       loop4      nvme0n1p5  sda1  sde2      stdout  tty20  tty34  tty48  tty61    userfaultfd  vcsu
btrfs-control    drm_dp_aux2  hidraw9       loop5      nvme0n1p6  sda2  sdf       tty     tty21  tty35  tty49  tty62    vcs          vcsu1
bus              fb0          hpet          loop6      nvme0n1p7  sdb   sdf1      tty0    tty22  tty36  tty5   tty63    vcs1         vcsu2
char             fd           hugepages     loop7      nvram      sdb1  sdf2      tty1    tty23  tty37  tty50  tty7     vcs2         vcsu3
console          full         hwrng         mapper     port       sdb2  sg0       tty10   tty24  tty38  tty51  tty8     vcs3         vcsu4
core             fuse         initctl       mem        ppp        sdb3  sg1       tty11   tty25  tty39  tty52  tty9     vcs4         vcsu5
cpu              gpiochip0    input         mqueue     psaux      sdc   sg2       tty12   tty26  tty4   tty53  ttyS0    vcs5         vcsu6
cpu_dma_latency  hidraw0      kmsg          net        ptmx       sdc1  sg3       tty13   tty27  tty40  tty54  ttyS1    vcs6         vfio
cuse             hidraw1      kvm           ng0n1      ptp0       sdc2  sg4       tty14   tty28  tty41  tty55  ttyS2    vcsa         vga_arbiter
disk             hidraw2      log           null       pts        sdd   sg5       tty15   tty29  tty42  tty56  ttyS3    vcsa1        vhci
dm-0             hidraw3      loop-control  nvme0      random     sdd1  shm       tty16   tty3   tty43  tty57  udmabuf  vcsa2        vhost-net
dm-1             hidraw4      loop0         nvme0n1    rfkill     sdd2  snapshot  tty17   tty30  tty44  tty58  uhid     vcsa3        vhost-vsock
dma_heap         hidraw5      loop1         nvme0n1p1  rtc        sdd3  snd       tty18   tty31  tty45  tty59  uinput   vcsa4        watchdog
```

Next step is to setup `fstab`, and in my case, the `crypttab` as well.

```
# /etc/crypttab
debian_root_crypt UUID=f03da021-0a3f-48ed-9730-6b9952ff8ab1 none luks,discard
```


```
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# systemd generates mount units based on this file, see systemd.mount(5).
# Please run 'systemctl daemon-reload' after making changes here.
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
# /boot/efi was on /dev/nvme0n1p1 during installation
UUID=31D9-3BA2  /boot/efi       vfat    umask=0077      0       1
# /boot
UUID=d699eed6-1529-4ebe-9890-0d08ffdf1e44       /boot   ext4    defaults,noatime        0       2
# btrfs volumes
UUID=5880a5d2-a9c4-4d8f-a946-8431b3e14608       /       btrfs   defaults,noatime,ssd,compress=zstd,subvol=root  0       0
UUID=5880a5d2-a9c4-4d8f-a946-8431b3e14608       /home   btrfs   defaults,noatime,ssd,compress=zstd,subvol=home  0       0
UUID=5880a5d2-a9c4-4d8f-a946-8431b3e14608       /opt    btrfs   defaults,noatime,ssd,compress=zstd,subvol=opt   0       0
UUID=5880a5d2-a9c4-4d8f-a946-8431b3e14608       /root   btrfs   defaults,noatime,ssd,compress=zstd,subvol=root-home     0       0
UUID=5880a5d2-a9c4-4d8f-a946-8431b3e14608       /srv    btrfs   defaults,noatime,ssd,compress=zstd,subvol=srv   0       0
UUID=5880a5d2-a9c4-4d8f-a946-8431b3e14608       /usr/local      btrfs   defaults,noatime,ssd,compress=zstd,subvol=usr-local     0       0
UUID=5880a5d2-a9c4-4d8f-a946-8431b3e14608       /var    btrfs   defaults,noatime,ssd,compress=zstd,subvol=var   0       0
# btrfs root
UUID=5880a5d2-a9c4-4d8f-a946-8431b3e14608       /mnt/btrfs-root btrfs   defaults,noatime,ssd,compress=zstd      0       0
```

Next, mount filesystems. I'm not sure why but I had to mount `/proc` manually.

```
root@debian:/# mount -a
root@debian:/# mount -t proc proc /proc
```

Next, configure /etc/adjtime. I will not be copying my file here, because it would reveal my timezone, which is personally identifiable.

A few more files to configure:

```
# /etc/network/interfaces
######################################################################
# /etc/network/interfaces -- configuration file for ifup(8), ifdown(8)
# See the interfaces(5) manpage for information on what options are
# available.
######################################################################

# I have an available ethernet connection... if you need wifi, this will
# be more complex

# dhcp ethernet:
auto eth0
iface eth0 inet dhcp
```

```
# /etc/resolv.conf

# my resolv.conf had already been generated by NetworkManager, I believe
# this came from my live system.

# Generated by NetworkManager
search example.com # see "search" in "man resolv.conf"
nameserver 192.168.1.1
```

```
# /etc/hostname
debian # change to whatever you want to name your computer. Have fun :) 
```

```
# /etc/hosts
127.0.0.1       localhost
::1             localhost ip6-localhost ip6-loopback
ff02::1         ip6-allnodes
ff02::2         ip6-allrouters
```

In my `sources.list`, I included non-free and non-free-firmware. This is optional, but I want nvidia drivers.

```
# /etc/apt/sources.list
# main
deb http://ftp.us.debian.org/debian trixie main contrib non-free non-free-firmware
deb-src http://ftp.us.debian.org/debian trixie main contrib non-free non-free-firmware

# fallback
deb http://deb.debian.org/debian/ trixie main contrib non-free non-free-firmware
deb-src http://deb.debian.org/debian/ trixie main contrib non-free non-free-firmware

# security, src
deb http://security.debian.org/ trixie-security main non-free-firmware
deb-src http://security.debian.org/ trixie-security main non-free-firmware
```

After changing `sources.list`, run `apt update`.

Just a few more things! Configure your locale....

```
root@debian:/# apt install locales
...
root@debian:/# dpkg-reconfigure locales
```

Also configure the keyboard. The keyboard can't be set while chroot, so this will take effect next boot.

```
root@debian:/# apt install console-setup
root@debian:/# dpkg-reconfigure keyboard-configuration 
```

And install a kernel! Things are getting exciting :3

```
root@debian:/# apt install linux-image-amd64
Installing:                      
  linux-image-amd64

Installing dependencies:
  apparmor  dracut-install   initramfs-tools-bin   klibc-utils  linux-base                       zstd
  busybox   initramfs-tools  initramfs-tools-core  libklibc     linux-image-6.12.48+deb13-amd64

Suggested packages:
  apparmor-profiles-extra  bash-completion      linux-doc-6.12          grub-pc           | extlinux
  apparmor-utils           firmware-linux-free  debian-kernel-handbook  | grub-efi-amd64

Summary:
  Upgrading: 0, Installing: 12, Removing: 0, Not Upgrading: 0
  Download size: 110 MB
  Space needed: 118 MB / 987 GB available

Continue? [Y/n] y
```

Most of the output looked fine, but I noticed one thing that makes me a little worried...

```
update-initramfs: deferring update (trigger activated)
Setting up linux-image-6.12.48+deb13-amd64 (6.12.48-1) ...
I: /vmlinuz.old is now a symlink to boot/vmlinuz-6.12.48+deb13-amd64
I: /initrd.img.old is now a symlink to boot/initrd.img-6.12.48+deb13-amd64
I: /vmlinuz is now a symlink to boot/vmlinuz-6.12.48+deb13-amd64
I: /initrd.img is now a symlink to boot/initrd.img-6.12.48+deb13-amd64
/etc/kernel/postinst.d/initramfs-tools:
update-initramfs: Generating /boot/initrd.img-6.12.48+deb13-amd64
W: /sbin/fsck.btrfs doesn't exist, can't install to initramfs
Setting up linux-image-amd64 (6.12.48-1) ...
Processing triggers for initramfs-tools (0.148.3) ...
update-initramfs: Generating /boot/initrd.img-6.12.48+deb13-amd64
W: /sbin/fsck.btrfs doesn't exist, can't install to initramfs
```

Specifically, the line `W: /sbin/fsck.btrfs doesn't exist, can't install to initramfs`... I'm guessing this is for checking filesystems on boot, which I disabled for `btrfs` filesystems anyways. (That's the final "0" in the fstab line) So I think things will be okay.

Next (and hopefully last!) we install grub for the bootloader. I deviate a bit from the debian manual here, because they seem to be using both an old MBR partition table and BIOS booting. I am using UEFI. If you set up your `/etc/fstab` similar to me, your efi directory will also be `/boot/efi`. The commands below should work for a modern, 64-bit system booting from UEFI. My bootloader id is `debian2` because I have an existing debian install that I am hoping to supercede with this new install, but I will not overwrite it until I am sure everything is working on the new install. You can set your bootloader id to whatever you want :) Have fun with it! Thanks to [the arch linux wiki](https://wiki.archlinux.org/title/GRUB#Installation) for helping me with this one.
f
```
root@debian:/# apt install grub-efi-amd64
root@debian:/# grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=debian2
Installing for x86_64-efi platform.
grub-install: warning: EFI variables cannot be set on this system.
grub-install: warning: You will have to complete the GRUB setup manually.
Installation finished. No error reported.
```

EFI variables cannot be set... hmmm... 

Oh! `/sys` is empty! Let's fix that, by running the following command in the host system, *outside* the chroot environment.

```
user@debian:/media/user/Ventoy/files$ sudo mount --rbind /sys /mnt/sys
```

Back inside the chroot environment, that seems to have done the trick:

```
root@debian:/# ls /sys
block  bus  class  dev  devices  firmware  fs  hypervisor  kernel  module  power
```

Let's try `grub-install` again:

```
root@debian:/# grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=debian2
Installing for x86_64-efi platform.
Installation finished. No error reported.
```

Wonderful :) Don't forget to run `update-grub` to make the `grub.cfg` file and add a boot menu entry.

```
root@debian:/# update-grub
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-6.12.48+deb13-amd64
Found initrd image: /boot/initrd.img-6.12.48+deb13-amd64
Warning: os-prober will be executed to detect other bootable partitions.
Its output will be used to detect bootable binaries on them and create new boot entries.
grub-probe: error: cannot find a GRUB drive for /dev/sdf1.  Check your device.map.
Adding boot menu entry for UEFI Firmware Settings ...
done
```

`/dev/sdf1` is just the live system, so no issues there... However, I spent a good hour trying to get my other linux distributions (of which I have many) and windows to show up in grub.... Couldn't do it. I'll fiddle more after the install is done. Honestly, `systemd-boot` is so much nicer and I wish debian had more support for it. That's the only part of setting up nixos that was easy: configuring `systemd-boot` and adding my other operating systems to the menu.

Create a new user for your system:

```
root@debian:/# adduser riley
...
root@debian:/# adduser riley sudo # give her sudo priveleges
```

The system is technically bootable now, but if you were to boot here, you would only have a bare tty to use. I recommend adding some more packages, and `tasksel` makes it easy to do so. Start by listing your available options:

```
root@debian:/# apt install tasksel
root@debian:/# tasksel --list-tasks
u desktop       Debian desktop environment
u gnome-desktop GNOME
u xfce-desktop  Xfce
u gnome-flashback-desktop       GNOME Flashback
u kde-desktop   KDE Plasma
u cinnamon-desktop      Cinnamon
u mate-desktop  MATE
u lxde-desktop  LXDE
u lxqt-desktop  LXQt
u web-server    web server
u ssh-server    SSH server
u laptop        laptop
u blendsel      Choose a Debian Blend for installation
```

These are pretty self-explanatory. I recommend installing your favorite desktop with `tasksel install <your-fav>-desktop`. Also, look into [the debian blends](https://www.debian.org/blends/)! There are some really interesting ones, like one for astronomy, one for GIS, and even one for gaming :3

For my case, I'm going to install KDE. But before proceeding further, I'm going to create a snapshot of all volumes in the freshly-installed system. Typically, `tmp` and `var` will not be snapshotted, but I think it's useful to have a snapshot of the fresh system so I can see differences between that snapshot and a later state of the system. Snapshotting a subvolume with CoW disabled has [some side effects](https://wiki.archlinux.org/title/Btrfs#Effect_on_snapshots), so it's not recommended to do it often, but I think this will be okay.

```
root@debian:/# cd /mnt/btrfs-root/
root@debian:/mnt/btrfs-root# btrfs subvolume create snapshots
Create subvolume './snapshots'
root@debian:/mnt/btrfs-root# ls
root@debian:/mnt/btrfs-root# btrfs subvolume snapshot home /mnt/btrfs-root/snapshots/home-fresh-install
Create snapshot of 'home' in '/mnt/btrfs-root/snapshots/home-fresh-install'
root@debian:/mnt/btrfs-root# NAME=opt btrfs subvolume snapshot $NAME /mnt/btrfs-root/snapshots/$NAME-fresh-install
btrfs subvolume snapshot: exactly 2 arguments expected, 1 given
root@debian:/mnt/btrfs-root# NAME=opt; btrfs subvolume snapshot $NAME /mnt/btrfs-root/snapshots/$NAME-fresh-install
Create snapshot of 'opt' in '/mnt/btrfs-root/snapshots/opt-fresh-install'
root@debian:/mnt/btrfs-root# for vol in home opt root root-home srv tmp usr-local var ; do btrfs subvolume snapshot $vol /mnt/btrfs-root/snapshots/$vol-fresh-install; done
Create snapshot of 'root' in '/mnt/btrfs-root/snapshots/root-fresh-install'
Create snapshot of 'root-home' in '/mnt/btrfs-root/snapshots/root-home-fresh-install'
Create snapshot of 'srv' in '/mnt/btrfs-root/snapshots/srv-fresh-install'
Create snapshot of 'tmp' in '/mnt/btrfs-root/snapshots/tmp-fresh-install'
Create snapshot of 'usr-local' in '/mnt/btrfs-root/snapshots/usr-local-fresh-install'
Create snapshot of 'var' in '/mnt/btrfs-root/snapshots/var-fresh-install'
```

DOCTORED: 

```
root@debian:/# cd /mnt/btrfs-root/
root@debian:/mnt/btrfs-root# btrfs subvolume create snapshots
Create subvolume './snapshots'
root@debian:/mnt/btrfs-root# for vol in home opt root root-home srv tmp usr-local var ; do btrfs subvolume snapshot $vol /mnt/btrfs-root/snapshots/$vol-fresh-install; done
Create snapshot of 'home' in '/mnt/btrfs-root/snapshots/home-fresh-install'
Create snapshot of 'opt' in '/mnt/btrfs-root/snapshots/opt-fresh-install'
Create snapshot of 'root' in '/mnt/btrfs-root/snapshots/root-fresh-install'
Create snapshot of 'root-home' in '/mnt/btrfs-root/snapshots/root-home-fresh-install'
Create snapshot of 'srv' in '/mnt/btrfs-root/snapshots/srv-fresh-install'
Create snapshot of 'tmp' in '/mnt/btrfs-root/snapshots/tmp-fresh-install'
Create snapshot of 'usr-local' in '/mnt/btrfs-root/snapshots/usr-local-fresh-install'
Create snapshot of 'var' in '/mnt/btrfs-root/snapshots/var-fresh-install'
```

Let's install a DE!

```
root@debian:/mnt/btrfs-root# tasksel install kde-desktop
root@debian:/mnt/btrfs-root# apt clean
```

and after that, a snapshot with the new Desktop Environment.

```
root@debian:/mnt/btrfs-root# for vol in home opt root root-home srv tmp usr-local var ; do btrfs subvolume snapshot $vol /mnt/btrfs-root/snapshots/$vol-1-post-kde-install; done
```

Finally time to reboot and test things out :) Make sure to select your new install if you have multiple OSes installed like me.

My first boot did not work because the `dm-crypt` container failed to unlock. I fixed this by:
1. booting into the live system
2. installing `cryptsetup-initramfs`
3. making sure the name of the entry in `/etc/crypttab` matched exactly the name given to `cryptsetup luksOpen`. In my case I chose `debian-btrfs-pool`. For clarity, the exact commands I ran were the following:

```
# cryptsetup luksOpen /dev/sdd3 debian-btrfs-pool
# blkid /dev/sdd3 # to find the UUID for the next line
# echo "debian-btrfs-pool UUID=f03da021-0a3f-48ed-9730-6b9952ff8ab1 none luks,discard" > /etc/crypttab
```

4.

# Notes after getting system working

## default subvolume

I also set my "root" subvolume to be the default subvolume:

```
riley@riley-desktop-debian:~$ sudo btrfs subvolume list -t /mnt/btrfs-root/
ID      gen     top level       path
--      ---     ---------       ----
256     270     5               root
...
riley@riley-desktop-debian:~$ sudo btrfs subvolume set-default 256 /
```

Doing this means we will need to edit our fstab to properly mount the btrfs root filesystem. This is easy because [the id of the on-device filesystem is always 5](https://btrfs.readthedocs.io/en/latest/btrfs-subvolume.html#subvolume-and-snapshot).

```
# /etc/fstab
# some lines skipped...
UUID=5880a5d2-a9c4-4d8f-a946-8431b3e14608       /mnt/btrfs-root btrfs   defaults,noatime,ssd,compress=zstd,subvolid=5   0       0d
```

## networking

I noticed on boot that "networking.service" had an error. After boot I investigated this and found that the networking file we created during setup used the wrong interface name:

```
riley@riley-desktop-debian:~$ sudo systemctl status networking.service
[sudo] password for riley: 
× networking.service - Raise network interfaces
     Loaded: loaded (/usr/lib/systemd/system/networking.service; enabled; preset: enabled)
     Active: failed (Result: exit-code) since Thu 2025-09-25 17:41:10 UTC; 45s ago
 Invocation: 84bcb81a873d4a938ae9661a796ef2e1
       Docs: man:interfaces(5)
    Process: 1369 ExecStart=/usr/sbin/ifup -a --read-environment (code=exited, status=1/FAILURE)
    Process: 1408 ExecStopPost=/usr/bin/touch /run/network/restart-hotplug (code=exited, status=0/SUCCESS)
   Main PID: 1369 (code=exited, status=1/FAILURE)
   Mem peak: 5.8M
        CPU: 42ms

Sep 25 17:41:10 riley-desktop-debian ifup[1394]: dhcpcd-10.1.0 starting
Sep 25 17:41:10 riley-desktop-debian dhcpcd[1394]: dhcpcd-10.1.0 starting
Sep 25 17:41:10 riley-desktop-debian ifup[1397]: eth0: interface not found
Sep 25 17:41:10 riley-desktop-debian dhcpcd[1397]: eth0: interface not found
Sep 25 17:41:10 riley-desktop-debian ifup[1397]: dhcpcd exited
Sep 25 17:41:10 riley-desktop-debian dhcpcd[1397]: dhcpcd exited
Sep 25 17:41:10 riley-desktop-debian ifup[1369]: ifup: failed to bring up eth0
Sep 25 17:41:10 riley-desktop-debian systemd[1]: networking.service: Main process exited, code=exited, status=1/FAILURE
Sep 25 17:41:10 riley-desktop-debian systemd[1]: networking.service: Failed with result 'exit-code'.
Sep 25 17:41:10 riley-desktop-debian systemd[1]: Failed to start networking.service - Raise network interfaces.
riley@riley-desktop-debian:~$ vim /etc/net
netconfig  network/   networks   
riley@riley-desktop-debian:~$ vim /etc/net
netconfig  network/   networks   
riley@riley-desktop-debian:~$ vim /etc/network/i
if-down.d/      if-post-down.d/ if-pre-up.d/    if-up.d/        interfaces      interfaces.d/   
riley@riley-desktop-debian:~$ ls /etc/network/interfaces.d/
basic-eth-for-install
riley@riley-desktop-debian:~$ cat /etc/network/interfaces.d/basic-eth-for-install 
######################################################################
# /etc/network/interfaces -- configuration file for ifup(8), ifdown(8)
# See the interfaces(5) manpage for information on what options are
# available.
######################################################################

# dhcp ethernet:
auto eth0
iface eth0 inet dhcp
riley@riley-desktop-debian:~$ ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: enp39s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:d8:61:7c:86:1d brd ff:ff:ff:ff:ff:ff
    altname enx00d8617c861d
    inet 192.168.1.142/24 brd 192.168.1.255 scope global dynamic noprefixroute enp39s0
       valid_lft 86314sec preferred_lft 86314sec
    inet6 fe80::22e3:9510:2b3c:bba1/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```

So I just deleted that interface, KDE seems to take care of networking automatically.

       
```
riley@riley-desktop-debian:~$ rm /etc/network/interfaces.d/basic-eth-for-install 
```

## non-working guis

Almost none of my applications with GUIs were working. VSCodium, Joplin, tuta mail, signal messenger were all not working, regardless of if I insrtalled them natively, used flatpak, or ran an AppImage. I'm still not sure the cause, but all of it was fixed when I installed nvidia drivers.

## NTP 

ntp wasn't working so my time would get reset and I would run into the signature verification error again. I set up ntp by using `ntpsec` and [the ntp pool project](https://www.ntppool.org/en/use.html). For me, configuration was automatic. All I did was run `sudo apt install ntpsec` and it fixed my time! Verify it worked by running `ntpq -pn`. You should see a list of ntp servers. Alternatively, run `timedatectl` (included with debian 13) to verify the time is correct. You can also just check your visual clock if your DE has one. But keep in mind that it can take sometime (sometimes as much as half an hour!) for the time to update. I recommend manually setting it with `timedatectl` in the meantime to get signature verification working.

TODO: add timezone setting to the setup instructions to avoid the signature verification error until ntp can be setup.

