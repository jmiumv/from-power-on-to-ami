# Chapter 3: Storage Fundamentals

> **Environment note:** We're on EC2 where the disk appears as `/dev/nvme0n1`. On a physical machine or VM you might see `/dev/sda` (SATA) or `/dev/vda` (virtio). On Ubuntu, your root filesystem is likely `ext4` instead of `xfs`, and your partition layout may differ. The concepts (block devices, GPT, filesystems, mounting) are universal.

## The Storage Stack

From bottom to top, here's how Linux organizes storage:

```
Block Device → Partition Table → Partitions → Filesystem → Files
```

Let's explore each layer with real commands.

---

## Block Devices

Everything in Linux is represented as a file, including hardware. Disks are no exception. The kernel exposes each disk and partition as a special file in the `/dev/` directory. These aren't regular files with data inside them. They're portals that let you communicate directly with the hardware through the kernel's device drivers.

Let's look at what device files exist for our disk. We use `ls -la /dev/nvme*` because our disk uses the NVMe protocol. NVMe (Non-Volatile Memory Express) is a modern storage protocol designed for SSDs that communicates over the PCIe bus, which is the high-speed hardware connection between the CPU and peripherals. On EC2, the Nitro hardware presents your EBS volume to the instance as if it were a local NVMe drive. The `/dev/` directory is where Linux puts all device files, and `nvme*` matches everything starting with "nvme":

> On a physical machine with a SATA drive, you'd look at `/dev/sda*` instead. On a VM with virtio storage, it would be `/dev/vda*`. The concept is identical, only the naming differs.

```bash
$ ls -la /dev/nvme*
crw-------. 1 root root 247, 0 May 17 05:09 /dev/nvme0
brw-rw----. 1 root disk 259, 0 May 17 05:09 /dev/nvme0n1
brw-rw----. 1 root disk 259, 1 May 17 05:09 /dev/nvme0n1p1
brw-rw----. 1 root disk 259, 2 May 17 05:09 /dev/nvme0n1p127
brw-rw----. 1 root disk 259, 3 May 17 05:09 /dev/nvme0n1p128
```

Notice the first character:
- `c` (character device): command interface to the NVMe controller
- `b` (block device): data storage, read/write in fixed-size chunks

A **block device** stores data in fixed-size blocks and supports random access (jump to any block directly). Filesystems require block devices underneath them.

---

## Reading Raw Bytes

Why would you ever want to read raw disk bytes? Because it proves something important: underneath all the abstractions (files, directories, permissions), a disk is ultimately just a sequence of bytes. Understanding this is essential for later when we talk about disk images and AMIs, which are literally copies of these raw bytes.

Since block devices are files, you can read from them directly. When you do, you bypass the filesystem entirely. No file names, no directories. You get the raw binary content of the disk, sector by sector:

[`hexdump -C`](https://man7.org/linux/man-pages/man1/hexdump.1.html) displays raw bytes in hex format with an ASCII column on the right. `head -5` limits output to the first 5 lines:

```bash
$ sudo hexdump -C /dev/nvme0n1 | head -5
00000000  eb 63 90 00 00 00 00 00  00 00 00 00 00 00 00 00  |.c..............|
00000010  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00000050  00 00 00 00 00 00 00 00  00 00 00 80 00 58 00 00  |.............X..|
00000060  00 00 00 00 ff fa 90 90  f6 c2 80 74 05 f6 c2 70  |...........t...p|
```

Those first bytes (`eb 63 90`) are an x86 jump instruction, which is legacy MBR boot code. Even GPT disks keep this for backwards compatibility.

---

## Partition Table (GPT)

We just proved we can read raw bytes from the disk. But what's actually stored there? A disk is just a long sequence of sectors (each 512 bytes). On its own, it's just one big blob. So why divide it up?

Think back to our boot process. UEFI needs a tiny FAT16 partition to find the bootloader. The kernel needs a large XFS partition for the root filesystem. These are fundamentally different purposes requiring different filesystem formats. You can't format the entire disk as both FAT16 and XFS at the same time. So we split the disk into **partitions**, each with its own purpose and filesystem.

Other reasons to partition:
- **Isolation**: if `/var/log` fills up a separate partition, your root filesystem isn't affected
- **Different mount options**: you might want `/tmp` mounted with `noexec` (no running programs from there) but `/usr` needs to allow execution
- **Backup/recovery**: you can snapshot or replace one partition without touching others

The **partition table** is the small data structure at the beginning of the disk that acts as a map, defining where each partition starts and ends.

Our disk uses **GPT** (GUID Partition Table), the modern standard. Let's prove it exists by reading sector 1 (the GPT header). We use `dd` to read raw bytes from the disk:

```bash
$ sudo dd if=/dev/nvme0n1 bs=512 count=1 skip=1 | hexdump -C | head -5
```

Breaking down this command:
- `dd`: a tool for copying raw bytes
- `if=/dev/nvme0n1`: input file is our raw disk device
- `bs=512`: read in 512-byte blocks (one sector at a time)
- `count=1`: read exactly 1 block (one sector)
- `skip=1`: skip the first block (sector 0 is the MBR, we want sector 1)
- `| hexdump -C`: pipe the raw bytes through hexdump to display them in readable hex + ASCII format

Output:

```
00000000  45 46 49 20 50 41 52 54  00 00 01 00 5c 00 00 00  |EFI PART....\...|
00000010  25 12 79 b4 00 00 00 00  01 00 00 00 00 00 00 00  |%.y.............|
00000020  ff ff ff 00 00 00 00 00  22 00 00 00 00 00 00 00  |........".......|
```

**"EFI PART"** is the GPT magic signature. Every GPT disk starts with these 8 bytes at sector 1. The right column shows the ASCII representation of the bytes, and you can clearly read "EFI PART" there. This is how any tool knows "this disk uses GPT partitioning."

### Reading the partition table

While we can read the raw GPT bytes, there's a friendlier way. [`gdisk`](https://man7.org/linux/man-pages/man8/gdisk.8.html) (GPT fdisk) is a tool specifically for reading and editing GPT partition tables. The `-l` flag means "list" (read-only, just display what's there):

```bash
$ sudo gdisk -l /dev/nvme0n1
GPT fdisk (gdisk) version 1.0.8

Partition table scan:
  MBR: protective
  GPT: present

Found valid GPT with protective MBR; using GPT.
Disk /dev/nvme0n1: 16777216 sectors, 8.0 GiB
Model: Amazon Elastic Block Store
Sector size (logical/physical): 512/512 bytes
Partition table holds up to 128 entries
Main partition table begins at sector 2 and ends at sector 33
First usable sector is 34, last usable sector is 16777182
Partitions will be aligned on 2048-sector boundaries

Number  Start (sector)    End (sector)  Size       Code  Name
   1           24576        16777182   8.0 GiB     8300  Linux
 127           22528           24575   1024.0 KiB  EF02  BIOS Boot Partition
 128            2048           22527   10.0 MiB    EF00  EFI System Partition
```

### What is a partition?

A partition is simply a defined region of the disk: a start sector and end sector. The partition table is the map listing all partitions:

```
Disk layout (sector order):
┌────────┬──────────┬─────┬──────────┬──────┬──────────────────────┐
│  MBR   │GPT Header│Free │  p128    │ p127 │  p1 (Linux root)     │
│sec 0   │& Table   │     │  ESP     │ BIOS │  8 GiB               │
│512 byte│sec 1-33  │     │  10MB    │ 1MB  │                      │
└────────┴──────────┴─────┴──────────┴──────┴──────────────────────┘
sector:  0         1    34  2048     22528 24576              16777182
```

### GPT vs MBR

| | MBR (old) | GPT (modern) |
|---|---|---|
| Max disk size | 2 TB | 9.4 ZB |
| Max partitions | 4 | 128 |
| Used with | BIOS | UEFI |
| Redundancy | None | Backup at end of disk |
| Location | First 512 bytes | Sectors 1-33 |

### Partition type codes

Every GPT partition has a type GUID that identifies its purpose. The `gdisk` tool shows these as short codes. The full list is maintained in the [UEFI Specification](https://uefi.org/specs/UEFI/2.11/05_GUID_Partition_Table_Format.html#defined-gpt-partition-entry-guid-partition-type-values) and on the [Wikipedia GPT page](https://en.wikipedia.org/wiki/GUID_Partition_Table#Partition_type_GUIDs):

| Code | Meaning | How it's used |
|---|---|---|
| `8300` | Linux filesystem | Kernel mounts this as root |
| `EF00` | EFI System Partition | UEFI reads bootloader from here |
| `EF02` | BIOS Boot Partition | Legacy BIOS fallback |

UEFI finds the ESP by scanning GPT entries for type `EF00`.

---

## Filesystems

We now have partitions, which are defined regions of the disk. But a partition is still just a range of raw sectors with no concept of "files" or "directories." A **filesystem** is the software layer that adds this structure. It takes those raw sectors and organizes them into something meaningful: files with names, directories with hierarchies, permissions, timestamps, and so on.

Different filesystems organize data differently. XFS, ext4, FAT, NTFS are all different approaches to the same problem. Let's look at the XFS filesystem on our root partition using [`xfs_info`](https://man7.org/linux/man-pages/man8/xfs_info.8.html), which displays the filesystem's structural parameters:

```bash
$ sudo xfs_info /
meta-data=/dev/nvme0n1p1         isize=512    agcount=2, agsize=1047040 blks
data     =                       bsize=4096   blocks=2094075, imaxpct=25
naming   =version 2              bsize=16384  ascii-ci=0
log      =internal log           bsize=4096   blocks=16384, version=2
```

Key: **block size = 4096** (4KB). The filesystem reads/writes in 4KB chunks. Total: 2,094,075 blocks ≈ 8GB.

### Inodes: metadata per file

Every file has an **inode**, which is its metadata card. [`stat`](https://man7.org/linux/man-pages/man2/stat.2.html) shows all the inode information for a file:

```bash
$ stat /usr/bin/cat
  File: /usr/bin/cat
  Size: 36616         Blocks: 72         IO Block: 4096   regular file
Device: 10301h/66305d    Inode: 662046      Links: 1
Access: (0755/-rwxr-xr-x)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2026-05-15 00:28:33.203861894 +0000
Modify: 2026-01-28 17:36:12.000000000 +0000
 Birth: 2026-05-15 00:25:16.428796012 +0000
```

The inode stores everything about a file **except its name and content**:
- Size, permissions, owner
- Which data blocks hold the actual content
- Timestamps

[`df -i`](https://man7.org/linux/man-pages/man1/df.1.html) shows inode usage (the `-i` flag switches from disk space to inode counts):

```bash
$ df -i /
Filesystem      Inodes IUsed   IFree IUse% Mounted on
/dev/nvme0n1p1 4188096 48992 4139104    2% /
```

4.1 million inodes available = max 4.1 million files this filesystem can hold.

---

## Reading the EFI System Partition (raw)

Earlier in Chapter 1, we learned that UEFI reads its bootloader from the EFI System Partition. Now that we understand partitions and sectors, we can read the ESP's raw bytes and see its filesystem header. This partition starts at sector 2048 (we know this from the gdisk output above):

```bash
$ sudo dd if=/dev/nvme0n1 bs=512 count=1 skip=2048 | hexdump -C | head -5
00000000  eb 3c 90 6d 6b 66 73 2e  66 61 74 00 02 04 01 00  |.<.mkfs.fat.....|
00000010  02 00 02 00 50 f8 14 00  3f 00 ff 00 00 00 00 00  |....P...?.......|
00000020  00 00 00 00 00 00 29 7f  44 b0 75 4e 4f 20 4e 41  |......).D.uNO NA|
00000030  4d 45 20 20 20 20 46 41  54 31 36 20 20 20 0e 1f  |ME    FAT16   ..|
```

You can see `mkfs.fat` (the tool that formatted it) and `FAT16` (the filesystem type). This is the tiny partition UEFI reads at boot.

---

## Mounting

So we have partitions with filesystems on them. But how does Linux present them to users and applications? You don't access `/dev/nvme0n1p1` directly when you type `ls /`. Linux uses **mounting** to connect a partition's filesystem to a specific directory in a single unified tree. Once mounted, you access files through the directory path and the kernel handles routing reads/writes to the correct partition behind the scenes.

[`mount`](https://man7.org/linux/man-pages/man8/mount.8.html) with no arguments shows all current mount points. We pipe through `grep nvme` to see just our disk partitions:

```bash
$ mount | grep nvme
/dev/nvme0n1p1 on / type xfs (rw,noatime,seclabel,attr2,inode64,noquota)
/dev/nvme0n1p128 on /boot/efi type vfat (rw,noatime,fmask=0077,dmask=0077)
```

Two partitions, mounted at two locations in one unified tree:

```
/ (root) ← nvme0n1p1 (XFS, 8GB)
├── usr/bin/cat
├── etc/fstab
├── home/
└── boot/
    └── efi/ ← nvme0n1p128 (FAT16, 10MB)
        └── EFI/
            ├── BOOT/
            └── amzn/
```

### [/etc/fstab](https://man7.org/linux/man-pages/man5/fstab.5.html): what to mount at boot

```bash
$ cat /etc/fstab
UUID=afcd3653-4916-4996-a80d-b02b83e8be92     /           xfs    defaults,noatime  1   1
UUID=75B0-447F        /boot/efi       vfat    defaults,noatime,uid=0,gid=0,umask=0077  0   2
```

Each line: `device  mountpoint  fstype  options  dump  fsck-order`

Uses UUIDs (permanent) instead of device names (can change between boots).

### Virtual filesystems

Not everything that's mounted is backed by a real disk. Linux also has virtual filesystems that exist only in memory or are generated by the kernel on the fly. [`df -hT`](https://man7.org/linux/man-pages/man1/df.1.html) shows all mounted filesystems with their types (`-T`) in human-readable sizes (`-h`):

```bash
$ df -hT
Filesystem       Type      Size  Used Avail Use% Mounted on
devtmpfs         devtmpfs  4.0M     0  4.0M   0% /dev
tmpfs            tmpfs     459M     0  459M   0% /dev/shm
tmpfs            tmpfs     184M  444K  183M   1% /run
/dev/nvme0n1p1   xfs       8.0G  1.6G  6.4G  20% /
tmpfs            tmpfs     459M     0  459M   0% /tmp
/dev/nvme0n1p128 vfat       10M  1.3M  8.7M  13% /boot/efi
```

| Type | What it is | Persists? |
|---|---|---|
| `xfs`, `vfat` | Real disk partitions | Yes, captured in AMI |
| `tmpfs` | RAM-only filesystem | No, gone on reboot |
| `proc`, `sysfs` | Kernel interfaces as files | No, generated by kernel |

**This matters for AMIs:** Only real disk partitions are captured in a snapshot. tmpfs, /proc, /sys are recreated fresh every boot.

---

## Summary

```
┌─────────────────────────────────────────────────────────┐
│ 1. BLOCK DEVICE (/dev/nvme0n1)                          │
│    Raw 8GB, 16M sectors × 512 bytes each                │
│    NVMe protocol over PCIe (on EC2: Nitro presents EBS) │
│    Everything is ultimately bytes at known offsets       │
├─────────────────────────────────────────────────────────┤
│ 2. PARTITION TABLE (GPT)                                │
│    Sector 0: Protective MBR                             │
│    Sector 1: GPT Header ("EFI PART" signature)          │
│    Sectors 2-33: Partition entries (128 × 128 bytes)    │
│    Why partition: different regions, different purposes  │
├─────────────────────────────────────────────────────────┤
│ 3. PARTITIONS                                           │
│    p128 (10MB, FAT16): EFI System Partition             │
│    p127 (1MB): BIOS Boot fallback                       │
│    p1  (8GB, XFS): Root filesystem                      │
│    Each identified by type GUID (EF00, EF02, 8300)      │
├─────────────────────────────────────────────────────────┤
│ 4. FILESYSTEM (XFS)                                     │
│    Block size: 4KB, Inodes: 4.1M available              │
│    Organizes raw blocks into files and directories      │
│    Inode = metadata per file (size, permissions, blocks)│
├─────────────────────────────────────────────────────────┤
│ 5. MOUNTING                                             │
│    /etc/fstab: UUID → mount point                       │
│    One unified tree from multiple partitions            │
│    tmpfs/proc/sys = not on disk, not in AMI             │
└─────────────────────────────────────────────────────────┘
```

**The key takeaway:** A disk is just bytes. Partition tables, filesystems, and files are all structures written into those bytes. When we copy an entire disk byte-for-byte (which is what a disk image or AMI snapshot does), we capture everything: partition table, filesystems, and all files within them.

---

## Further Reading

* [man lsblk](https://man7.org/linux/man-pages/man8/lsblk.8.html) (list block devices)
* [man gdisk](https://man7.org/linux/man-pages/man8/gdisk.8.html) (GPT partition editor)
* [man parted](https://man7.org/linux/man-pages/man8/parted.8.html) (partition manipulation)
* [GUID Partition Table on Wikipedia](https://en.wikipedia.org/wiki/GUID_Partition_Table) (excellent diagrams of the on disk layout)
* [man xfs_info](https://man7.org/linux/man-pages/man8/xfs_info.8.html)
* [man stat](https://man7.org/linux/man-pages/man2/stat.2.html) (inode details)
* [man fstab](https://man7.org/linux/man-pages/man5/fstab.5.html) (filesystem table format)
* [man mount](https://man7.org/linux/man-pages/man8/mount.8.html)
* [The Linux Filesystem Hierarchy Standard](https://refspecs.linuxfoundation.org/FHS_3.0/fhs/index.html) (why directories are named what they are)
* [NVMe Specification](https://nvmexpress.org/specifications/) (if you want to go really deep on the protocol)

---

**Next:** [Chapter 4: Disk Images →](04-disk-images.md)
