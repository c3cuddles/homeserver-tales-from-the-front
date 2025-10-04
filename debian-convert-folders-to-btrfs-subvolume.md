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

Make separate subvolume:
- /var
- /tmp
- /home
- /root (home directory for root user... maybe unnecessary?)
- /srv should be its own subvolume if i ever start using it. I tend to run my
  servers in docker containers though

For now I will delay switching `/root` and `/srv` to separate subvols. For a
new user, I would recommend only converting the following to subvolumes:

- /home
- /tmp
- /var

This is simpler, and most users new to linux will not be using `/srv` as we
kinda push people to use docker these days.
