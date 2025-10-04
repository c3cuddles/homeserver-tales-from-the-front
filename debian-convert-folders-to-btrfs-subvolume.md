2025-10-04

# Plan

Convert debian 13 server install with simple partitioning: one volume mounted at `/` into
multiple subvolumes.

## Motivation

This will make it so we can snapshot `/` while excluding `/home`, `/var`, and
`/tmp` from snapshots. This enables more reliable rollbacks, as we don't want
to touch user files and `/var` and `/tmp` can cause issues if they are rolled
back with `/` (trust me, I have learned this from experience trying to rollback
my desktop running my first attempt at btrfs...)

# What folders will become subvolumes?

As a reminder, on debian:

	The root partition / must always physically contain /etc, /bin, /sbin, /lib, /dev and /usr, otherwise you won't be able to boot. 

[source](https://www.debian.org/releases/stable/amd64/apcs02.en.html)

I want this to be fairly minimal...

considering: 
- /var
- /tmp
- /home
- /root
- /srv

/home will definitely get its own subvolume. I think this is the most important
one to get its own subvolume.

/var is tough... Docker stores its images in /var/lib/Docker and I really want
that on a separate subvolume so it doesn't bloat snapshots. (I would disable
CoW there as well with chattr +C) But also, apt stores information in
`/var/lib/apt` and `/var/cache/apt` (see the ["Files" header in the apt man
page](https://linux.die.net/man/8/apt-get)), so if we do not rollback those
volumes with `/` we will be in trouble. Our first option is to create separate
subvolumes for docker and other large software as needed, but this is annoying
because we will need to keep watch for any software that starts dumping lots of
data to `/var`. Our second option is to create a separate subvolume for `/var`
and also for `/var/lib/apt` and `/var/cache/apt` and make sure the subvolumes
associated with `apt` get rolled back with `/`. Hmm... I don't like either of
these solutions. I could write a script to make sure rollbacks happen for both
`/` and for `/var`, but can I configure snapper to create snapshots for both
`/` and the apt subvolumes at the same time? I will need to think about this...

/tmp will get its own subvolume as well, since the files are meant to be
temporary anyways. I will also be disabling CoW with chattr +C to reduce bloat
from the churn that happens in /tmp

/root and /srv might benefit from being on separate subvolumes. If you use /srv
to host something with apache or nginx, you should *definitely* keep it on a
separate subvolume so it doesn't get rolled back with the rest of the system.

/root should probably get its own subvolume because it is the home directory
for the /root user. But I'm going to delay that for now, since I don't use the
root user and therefore don't keep anything in /root


## What I've decided
For now I will delay switching `/root` and `/srv` to separate subvols. I'm
going to delay deciding how I will handle `/var` for now, I have solved the
docker problem by configuring it to store images on a separate ext4 filesystem
and I want to move on. 

The setup I'm choosing is actually perfect for a new user, because it doesn't
touch things like `/var`
For a
new user, I would recommend only converting the following to subvolumes:

- /home
- /tmp

This is simpler, and most users new to linux will not be using `/srv` as we
kinda push people to use docker these days.

# Initial State:

My only subvolume is `/`.

```
2025-10-04 03:30:27 riley-server ~ » sudo btrfs subvolume list /                                                         1 ↵
[sudo] password for riley:
ID 256 gen 3955885 top level 5 path @rootfs
ID 258 gen 3955827 top level 256 path .snapshots
ID 5914 gen 3947620 top level 258 path .snapshots/5654/snapshot
ID 5919 gen 3947781 top level 258 path .snapshots/5659/snapshot
ID 5920 gen 3947783 top level 258 path .snapshots/5660/snapshot
...
```

# Converting `/tmp`

I decided to start with `/tmp` because it seemed the least risky of the
folders. The command I ran is a nice one-liner:

```
sudo mv /tmp /tmp-old && sudo btrfs subvolume create /tmp && sudo rsync -auHAX /tmp-old/ /tmp/
```

I use `rsync` almost exclusively over `cp`. I just like that I know exactly
what will be copied over, and what will be copied over is everything.

I inspected `/tmp` and everything is there! Listing the subvolumes again,
omitting snapshots:

```
2025-10-04 04:13:47 riley-server / » sudo btrfs subvolume list /                                                         1 ↵
[sudo] password for riley:
ID 256 gen 3955972 top level 5 path @rootfs
ID 5995 gen 3955972 top level 256 path tmp
```

Wonderful! /tmp is now on its own subvol. 

# Converting `/home`

The first thing to do is to log out of my main user and log in as root so
nothing will be writing to my home directory while we do the indiana jones
switcheroo. 
