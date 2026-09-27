---
title: "Forensics: Forensic git 1 - picoCTF"
description: "Disk image analysis, time-travelling through git logs, git object archaeology"
tags: [CTF, digital-forensics, picoCTF, disk-analysis, git-internals]
category: Forensics
difficulty: Medium
author: "@Kkhao"
date: 2026-09-27
---

# Forensic git 1 - Writeup

## Table of Contents
- [Challenge](#challenge)
- [Initial Triage](#initial-triage)
- [Locating the Git Repository](#locating-the-git-repository)
- [Understanding Git's Object Model](#understanding-gits-object-model)
- [Walking the Commit Chain](#walking-the-commit-chain)
- [Recovering the Flag](#recovering-the-flag)
- [Alternative Approach: Autopsy + Script](#alternative-approach-autopsy--script)
- [Key Takeaways](#key-takeaways)

## Challenge

> Can you find the flag in this disk image?
>
> **Hint:** How can you checkout the files of a previous commit?

We're given a raw disk image (`disk.img`) and need to recover a flag that has apparently been deleted from a tracked file at some point in its git history.

## Initial Triage

First, map out the partition table with `mmls`:

```bash
$ mmls disk.img
DOS Partition Table
Offset Sector: 0
Units are in 512-byte sectors

      Slot      Start        End          Length       Description
000:  Meta      0000000000   0000000000   0000000001   Primary Table (#0)
001:  -------   0000000000   0000002047   0000002048   Unallocated
002:  000:000   0000002048   0000616447   0000614400   Linux (0x83)
003:  000:001   0000616448   0001140735   0000524288   Linux Swap / Solaris x86 (0x82)
004:  000:002   0001140736   0002097151   0000956416   Linux (0x83)
```

Three partitions of interest:

| Offset (sectors) | Type | Verdict |
|---|---|---|
| 2048 | Linux (ext) | Alpine Linux boot partition — bootloader/kernel files only |
| 616448 | Linux Swap | No filesystem to analyze, skip |
| 1140736 | Linux (ext) | User/root filesystem — most likely to hold the flag |

### Partition @ 2048 — boot partition (ruled out)

```bash
$ fls -o 2048 disk.img
d/d 11:  lost+found
r/r 13:  ldlinux.sys
r/r 14:  ldlinux.c32
r/r 16:  config-6.6.116-0-lts
r/r 17:  vmlinuz-lts
r/r 18:  initramfs-lts
l/l 19:  boot
r/r 21:  libutil.c32
r/r 20:  extlinux.conf
r/r 22:  libcom32.c32
r/r 23:  mboot.c32
r/r 24:  menu.c32
r/r 15:  System.map-6.6.116-0-lts
r/r 25:  vesamenu.c32
V/V 76913: $OrphanFiles
```

Standard SYSLINUX/EXTLINUX bootloader files plus an Alpine `lts` kernel (6.6.116). Nothing suspicious — this is just the boot partition.

### Partition @ 616448 — swap (ruled out)

```bash
$ fls -o 616448 disk.img
Cannot determine file system type
```

No filesystem, can't hold user data in a way we can browse. Skipped.

### Partition @ 1140736 — root filesystem (target)

```bash
$ fls -o 1140736 disk.img
d/d 64770: home
d/d 11:    lost+found
d/d 64769: boot
d/d 32385: etc
d/d 13:    proc
d/d 64772: dev
d/d 14:    tmp
d/d 15:    lib
d/d 64773: var
d/d 22:    bin
d/d 24:    sbin
d/d 64778: usr
d/d 64948: media
d/d 170:   mnt
d/d 171:   opt
d/d 172:   root
d/d 173:   run
d/d 64952: srv
d/d 174:   sys
d/d 65275: swap
V/V 119417: $OrphanFiles
```

A typical Linux root layout. This is where a user's project (and its git history) would live.

## Locating the Git Repository

Drilling down through `home/`:

```bash
$ fls -o 1140736 disk.img 64770
d/d 64771: ctf-player

$ fls -o 1140736 disk.img 64771
d/d 65663: Code

$ fls -o 1140736 disk.img 65663
d/d 65664: secrets

$ fls -o 1140736 disk.img 65664
d/d 65662: .git
```

`~/Code/secrets/.git` — a bare working directory whose tracked files aren't visible anymore, but whose `.git` metadata is still on disk. That's exactly what the hint is pointing us at.

```bash
$ fls -o 1140736 disk.img 65662
d/d 65665: branches
r/r 65666: description
d/d 65667: hooks
d/d 65682: info
d/d 65684: refs
r/r 65710: HEAD
d/d 65689: objects
r/r 65702: config
r/r 65688: index
r/r 65704: COMMIT_EDITMSG
d/d 65705: logs
```

Reading the last commit message:

```bash
$ icat -o 1140736 disk.img 65704
Remove flag
```

Confirmed — the flag was present at some point and got removed in the latest commit. Since it's git, "removed" doesn't mean gone: the old blob still exists as an object until it's garbage-collected, and this image was captured before that happened.

## Understanding Git's Object Model

This is the core idea the challenge hint is testing. Git doesn't store history in one file you can just `cat`. Every commit, tree, and blob is a compressed **object**, addressed by its SHA-1 hash, and stored at:

```
.git/objects/<first 2 hex chars>/<remaining 38 hex chars>
```

A commit object simply contains a pointer to a `tree` (a snapshot of the file structure) and, optionally, a `parent` (the previous commit). Chaining `parent` pointers backwards *is* the commit history — there's no separate "log" data structure to inspect on disk.

So the recovery plan is:

1. Resolve `refs/heads/master` → current commit hash.
2. Find that hash's object → read it → get its `tree` and `parent`.
3. Follow `parent` back until there's no more `parent` (the initial commit).
4. Walk that commit's `tree` to find `flag.txt`'s blob hash.
5. Decompress the blob → read the flag.

## Walking the Commit Chain

### Step 1 — Resolve the branch ref

```bash
$ fls -o 1140736 disk.img 65684
d/d 65685: heads
d/d 65687: tags

$ fls -o 1140736 disk.img 65685
r/r 65686: master

$ icat -o 1140736 disk.img 65686
852b30c418e4161544c86b92cb92a2f8915f61ef
```

`refs/heads/master` gives us the tip commit: **`852b30c4...`**.

### Step 2 — Map hash → object path

Following the `objects/<xx>/<...>` layout, `85` is the subdirectory and `2b30c4...` is the filename:

```bash
$ fls -o 1140736 disk.img <inode of objects/85>
r/r 65701: 2b30c418e4161544c86b92cb92a2f8915f61ef
```

Every object we recover from here on is found the same way: take the hash, look up its 2-character subfolder, then find the file matching the remaining 38 characters.

### Step 3 — Read the tip commit ("Remove flag")

Git objects are zlib-compressed (`icat` gives us the raw bytes, recognizable by the `0x78` / `x` zlib header):

```bash
$ icat -o 1140736 disk.img 65701 > obj_85.raw
$ openssl zlib -d < obj_85.raw
commit 230
tree 4b825dc642cb6eb9a060e54bf8d69288fbee4904
parent 2a25cc47c382540ecf63a6e97216d7c211d58c0f
author ctf-player <ctf-player@example.com> 1763544005 +0000
committer ctf-player <ctf-player@example.com> 1763544005 +0000

Remove flag
```

Two useful things here:

- `parent 2a25cc47...` — the previous commit, our next stop.
- `tree 4b825dc642cb6eb9a060e54bf8d69288fbee4904` — this is **Git's well-known empty-tree hash**, identical in every git repository on earth. It confirms that after this commit, the tracked directory is empty — i.e. `flag.txt` was deleted here, exactly matching the "Remove flag" message.

### Step 4 — Read the parent commit ("Add flag")

```bash
$ fls -o 1140736 disk.img <inode of objects/2a>
r/r 65699: 25cc47c382540ecf63a6e97216d7c211d58c0f

$ icat -o 1140736 disk.img 65699 > obj_2a.raw
$ openssl zlib -d < obj_2a.raw
commit 179
tree 34670eb2b45b4e90de5bf107d8a3b1fef8a4f846
author ctf-player <ctf-player@example.com> 1763544005 +0000
committer ctf-player <ctf-player@example.com> 1763544005 +0000

Add flag
```

No `parent` field — this is the **initial commit**. The chain stops here, and its `tree` (`34670eb2...`) is exactly the snapshot we want: the one where the flag still existed.

## Recovering the Flag

### Step 5 — Read the tree object

```bash
$ fls -o 1140736 disk.img <inode of objects/34>
r/r 65697: 670eb2b45b4e90de5bf107d8a3b1fef8a4f846

$ icat -o 1140736 disk.img 65697 > obj_34.raw
$ openssl zlib -d < obj_34.raw > obj_34.dec
$ xxd obj_34.dec
00000000: 7472 6565 2033 3600 3130 3036 3434 2066  tree 36.100644 f
00000010: 6c61 672e 7478 7400 860f 7655 dc8a 9f8a  lag.txt...vU....
00000020: aa51 af43 a1d7 2adb c761 eb94             .Q.C..*..a..
```

A raw tree object's format is `tree <size>\0` followed by repeated entries of `<mode> <filename>\0<20-byte binary SHA-1>`. Parsed out:

- `tree 36\0` — 36-byte payload.
- `100644 flag.txt\0` — a regular file named `flag.txt`.
- `86 0f 76 55 dc 8a 9f 8a aa 51 af 43 a1 d7 2a db c7 61 eb 94` — the file's blob hash in raw binary, i.e. **`860f7655dc8a9f8aaa51af43a1d72adbc761eb94`**.

### Step 6 — Read the blob

```bash
$ fls -o 1140736 disk.img <inode of objects/86>
r/r 65693: 0f7655dc8a9f8aaa51af43a1d72adbc761eb94

$ icat -o 1140736 disk.img 65693 > obj_86.raw
$ openssl zlib -d < obj_86.raw
blob 31
academy{g17_r3m3mb3r5_d4ddf904}
```

**Flag:** `academy{g17_r3m3mb3r5_d4ddf904}`

### The full chain, visually

```
refs/heads/master
        │
        ▼
commit 85 2b30c4...  "Remove flag"
  tree  = 4b825dc6...  (empty tree — Git's universal empty-tree hash)
  parent = 2a 25cc4...
        │
        ▼
commit 2a 25cc4...   "Add flag"   (no parent → initial commit)
  tree  = 34 670eb2...
        │
        ▼
tree 34 670eb2...
  100644 flag.txt → blob 86 0f7655...
        │
        ▼
blob 86 0f7655...
  "academy{g17_r3m3mb3r5_d4ddf904}"
```

## Alternative Approach: Autopsy + Script

Manually resolving one hash at a time works but is slow and error-prone at scale. A faster, more reliable path: use **Autopsy** to browse the filesystem GUI-side, dump every object under `.git/objects/` in bulk, then batch-decompress with a small script instead of calling `openssl zlib -d` by hand each time.

**Step 1 — Dump every relevant inode to raw files**

```bash
for i in 65699 65697 65695 65701 65693; do
    icat -o 1140736 disk.img "$i" > object_$i.raw
done
```

**Step 2 — Zlib-decompress in Python**

```python
import zlib
import sys

fname = sys.argv[1]
data = open(fname, "rb").read()
try:
    out = zlib.decompress(data)
    print(out[:1000])
except Exception as e:
    print('error', e)
```

**Step 3 — Batch decode**

```bash
for f in object_*.raw; do
    echo "==== $f ===="
    python3 decode.py "$f"
    echo
done
```

Output:

```
==== object_65693.raw ====
b'blob 31\x00academy{g17_r3m3mb3r5_d4ddf904}'

==== object_65695.raw ====
b'tree 0\x00'

==== object_65697.raw ====
b'tree 36\x00100644 flag.txt\x00\x86\x0fvU\xdc\x8a\x9f\x8a\xaaQ\xafC\xa1\xd7*\xdb\xc7a\xeb\x94'

==== object_65699.raw ====
b'commit 179\x00tree 34670eb2b45b4e90de5bf107d8a3b1fef8a4f846\nauthor ctf-player <ctf-player@example.com> 1763544005 +0000\ncommitter ctf-player <ctf-player@example.com> 1763544005 +0000\n\nAdd flag\n'

==== object_65701.raw ====
b'commit 230\x00tree 4b825dc642cb6eb9a060e54bf8d69288fbee4904\nparent 2a25cc47c382540ecf63a6e97216d7c211d58c0f\nauthor ctf-player <ctf-player@example.com> 1763544005 +0000\ncommitter ctf-player <ctf-player@example.com> 1763544005 +0000\n\nRemove flag\n'
```

This confirms the exact same chain independently, and is the approach to prefer when a repo's history is deeper than 2-3 commits.

## Key Takeaways

- **Git history isn't a log file — it's a linked list of objects.** A `commit` object's `parent` field is the only thing connecting one commit to the previous one; there's no separate "history" structure on disk.
- **Deleted ≠ gone, until garbage collection runs.** A file removed by a later commit still exists as a `blob` object as long as some earlier commit or tree references it and `git gc` hasn't pruned it.
- **The empty-tree hash `4b825dc642cb6eb9a060e54bf8d69288fbee4904` is universal** — it's the SHA-1 of an empty tree, identical across every Git repository, and a handy signal that a commit deleted everything it was tracking.
- **Object addressing (`objects/<2 chars>/<38 chars>`) is what makes hash-by-hash forensic recovery possible** even when the working tree and normal `git log`/`git show` tooling aren't available — as here, where we're reading raw inodes off a disk image rather than a mounted, working git checkout.

---

**Flag:** `academy{g17_r3m3mb3r5_d4ddf904}`
**Author:** [@Kkhao](https://github.com/Kkhao)
**Solved:** 2026-09-27
