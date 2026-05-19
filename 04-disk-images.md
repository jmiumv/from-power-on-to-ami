# Chapter 4: Disk Images

> **Environment note:** Everything in this chapter works on any Linux machine with root access. We use `dd`, `parted`, `losetup`, `mkfs.ext4`, and `mount`, which are available on all major distros. If you're on Ubuntu, you're already set. If any tool is missing, install it with your package manager.

## Why are we doing this?

In the previous chapters, we learned that a disk is just bytes: GPT header, partition entries, filesystem structures, file contents. All of it lives at specific offsets on the disk. So here's the natural question: if it's all just bytes, can we copy those bytes into a regular file and recreate the disk later?

The answer is yes, and that's exactly what a **disk image** is. It's a regular file that contains a byte-for-byte copy of a disk. This concept is the foundation of:

* **AMIs**: an EBS snapshot is a stored disk image. Launching an instance restores it to a new volume.
* **Virtual machines**: when you run a VM on your laptop (VirtualBox, QEMU), the "virtual hard drive" is a disk image file.
* **Backup and cloning**: copy a disk image file to another machine, attach it, and you have an exact clone.
* **AMI build tools**: they construct a disk image in a file, install an OS into it, then upload the bytes to AWS as a snapshot.

In this chapter, we're going to build a disk image from scratch. By hand. We'll create a blank file, partition it, format it, mount it, write files into it, and then prove that those files persist inside the image. This is the exact same process (conceptually) that AMI build tools automate.

## The Core Concept

A disk image is a **copy of a disk's bytes in a regular file**. The GPT header, partitions, filesystem structures, all files, everything is just bytes, and you can put them in a file.

Let's build one from scratch to prove it.

---

## Step 1: Create a Blank Disk

Think about what a brand new hard drive looks like before anyone uses it: just empty space. We're going to simulate that by creating a 64MB file full of zeros. This file will be our "disk." Everything that follows (partition table, filesystem, files) will be written into this file, just like it would be written onto a real disk.

[`dd`](https://man7.org/linux/man-pages/man1/dd.1.html) is a low-level tool for copying raw bytes. Here we're reading from `/dev/zero` (a special device that produces infinite zeros) and writing to a file. `bs=1M` means read/write in 1 megabyte chunks, `count=64` means do it 64 times:

```bash
$ dd if=/dev/zero of=/tmp/my-disk.img bs=1M count=64
64+0 records in
64+0 records out
67108864 bytes (67 MB, 64 MiB) copied, 0.0239029 s, 2.8 GB/s
```

64MB of zeros, like a brand new blank hard drive.

```bash
$ hexdump -C /tmp/my-disk.img | head -3
00000000  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
04000000

$ ls -lh /tmp/my-disk.img
-rw-r--r--. 1 ec2-user ec2-user 64M May 18 07:16 /tmp/my-disk.img
```

The `*` means "everything in between is the same" (all zeros).

---

## Step 2: Add a Partition Table

We have a blank "disk" (file full of zeros). Just like a real disk, we need a partition table before we can do anything useful. Remember from Chapter 3: the partition table is what tells the OS where partitions start and end.

[`parted`](https://man7.org/linux/man-pages/man8/parted.8.html) is a partition manipulation tool. It works on real disks and image files alike. `mklabel gpt` writes a fresh GPT partition table onto the file:

```bash
$ sudo parted /tmp/my-disk.img mklabel gpt
```

Now let's verify the GPT signature appeared:

```bash
$ hexdump -C /tmp/my-disk.img | grep -A1 "EFI PART"
00000200  45 46 49 20 50 41 52 54  00 00 01 00 5c 00 00 00  |EFI PART....\...|
00000210  50 43 18 83 00 00 00 00  01 00 00 00 00 00 00 00  |PC..............|
--
03fffe00  45 46 49 20 50 41 52 54  00 00 01 00 5c 00 00 00  |EFI PART....\...|
03fffe10  95 8f 14 0d 00 00 00 00  ff ff 01 00 00 00 00 00  |................|
```

Two copies: primary GPT header at offset `0x200` (sector 1) and backup at the end of the disk. GPT's redundancy.

---

## Step 3: Create a Partition

Now we have a GPT table but no partitions defined in it. Let's create one that fills most of the disk. `mkpart primary ext4 1MiB 63MiB` means "create a partition starting at the 1MB mark and ending at 63MB" (we start at 1MB for alignment, as explained in Chapter 3):

```bash
$ sudo parted /tmp/my-disk.img mkpart primary ext4 1MiB 63MiB

$ sudo parted /tmp/my-disk.img print
Model:  (file)
Disk /tmp/my-disk.img: 67.1MB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name     Flags
 1      1049kB  66.1MB  65.0MB               primary
```

The partition starts at 1MB (sector 2048), which is the standard alignment for performance on modern storage hardware.

---

## Step 4: Loop Devices (Make the File Act Like a Disk)

We now have a file with a GPT partition table and a partition defined. But we can't just run `mkfs` or `mount` on a regular file. Those tools expect a block device (something in `/dev/`). We need a way to make the kernel treat our file as if it were a real disk.

This is what **loop devices** do. They're a kernel feature that takes a regular file and presents it as a block device. Once attached, the file becomes indistinguishable from a real disk as far as the rest of the system is concerned:

[`losetup`](https://man7.org/linux/man-pages/man8/losetup.8.html) manages loop devices. `--find` picks the next available loop device, `--show` prints which one it chose, and `--partscan` tells the kernel to read the partition table inside the file and create partition device files:

```bash
$ sudo losetup --find --show --partscan /tmp/my-disk.img
/dev/loop0
```

Now check what appeared:

```bash
$ ls /dev/loop0*
/dev/loop0  /dev/loop0p1
```

The kernel:
1. Read the file
2. Found the GPT partition table inside it
3. Created `/dev/loop0` (whole disk) and `/dev/loop0p1` (partition 1)

These behave identically to `/dev/nvme0n1` and `/dev/nvme0n1p1`. The filesystem layer doesn't know the difference.

### What is a loop device?

```
WITHOUT loop device:
  /tmp/my-disk.img → just a file, you can't mount it

WITH loop device:
  /tmp/my-disk.img → /dev/loop0 (block device!)
  
  The kernel translates:
    "read block 500 from /dev/loop0"
       → "read bytes 500×4096 from /tmp/my-disk.img"
```

---

## Step 5: Format with a Filesystem

We have a block device (`/dev/loop0p1`) that represents our partition. But it's still raw sectors with no structure. Remember from Chapter 3: a filesystem is what organizes raw blocks into files and directories. Without formatting, there's nothing to mount.

[`mkfs.ext4`](https://man7.org/linux/man-pages/man8/mkfs.ext4.8.html) (make filesystem) creates an ext4 filesystem on the partition. This writes the superblock, inode table, and block allocation structures:

```bash
$ sudo mkfs.ext4 /dev/loop0p1
mke2fs 1.46.5 (30-Dec-2021)
Creating filesystem with 63488 1k blocks and 15872 inodes
Filesystem UUID: 17236087-e66a-4a88-a39c-966e04e66557
Superblock backups stored on blocks:
    8193, 24577, 40961, 57345

Allocating group tables: done
Writing inode tables: done
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done
```

The partition now has an ext4 filesystem: inodes, data blocks, journal, just like the real root partition.

---

## Step 6: Mount and Write Files

Now for the payoff. We have a file that contains a GPT partition table, a partition, and an ext4 filesystem. Let's mount it and actually put files inside. This is the moment where our image file becomes something useful, not just structured bytes but actual content.

`mkdir -p` creates a directory (and any parent directories needed). [`mount`](https://man7.org/linux/man-pages/man8/mount.8.html) attaches the partition's filesystem to that directory:

```bash
$ sudo mkdir -p /mnt/myimage
$ sudo mount /dev/loop0p1 /mnt/myimage

$ df -hT /mnt/myimage
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/loop0p1   ext4   53M   14K   49M   1% /mnt/myimage
```

Now we can write files into our image:

```bash
$ sudo bash -c 'echo "Hello from my disk image!" > /mnt/myimage/hello.txt'
$ sudo mkdir -p /mnt/myimage/etc
$ sudo bash -c 'echo "myimage" > /mnt/myimage/etc/hostname'

$ ls -la /mnt/myimage/
total 15
drwxr-xr-x. 4 root root  1024 May 18 21:54 .
drwxr-xr-x. 3 root root    21 May 18 21:53 ..
drwxr-xr-x. 2 root root  1024 May 18 21:54 etc
-rw-r--r--. 1 root root    26 May 18 21:54 hello.txt
drwx------. 2 root root 12288 May 18 21:52 lost+found

$ cat /mnt/myimage/hello.txt
Hello from my disk image!
```

**Every write goes directly into the .img file** via the loop device. There's no separate "save" step because the file IS the disk.

---

## Step 7: Unmount and Detach

We've written files into our image. Now let's "disconnect" everything and verify that the data is permanently baked into the image file. This is important because it proves the image is self-contained. You could email this file to someone, they could attach it, and they'd see the same files.

Two steps to tear down (reverse of setup):

```bash
$ sudo umount /mnt/myimage    # detach filesystem from directory
$ sudo losetup -d /dev/loop0  # detach file from block device
```

Why both?
- `umount` disconnects the filesystem from the directory (flushes cached writes to .img)
- `losetup -d` disconnects the .img file from the loop device

```
┌────────────────────┐
│  /mnt/myimage/     │  ← umount disconnects this
├────────────────────┤
│  /dev/loop0p1      │  ← losetup -d disconnects this
├────────────────────┤
│  /tmp/my-disk.img  │  ← the file remains with all data baked in
└────────────────────┘
```

---

## Step 8: Proving Persistence

The image file is now just sitting there on disk, detached from everything. Let's re-attach it and prove the files are still inside. If this works, it confirms that all data is stored in the image file itself, not in some temporary kernel state:

```bash
$ sudo losetup --find --show --partscan /tmp/my-disk.img
/dev/loop0

$ sudo mount /dev/loop0p1 /mnt/myimage
$ cat /mnt/myimage/hello.txt
Hello from my disk image!
$ cat /mnt/myimage/etc/hostname
myimage
```

Files survived! They're baked into the .img bytes permanently.

---

## Step 9: Cloning = Copying the File

Here's the final and most important step. If a disk image is just a file, then copying the file should give us a perfect clone of the "disk." This is exactly what AWS does when multiple instances launch from the same AMI: each one gets its own copy of the same bytes.

```bash
$ sudo umount /mnt/myimage
$ sudo losetup -d /dev/loop0
$ cp /tmp/my-disk.img /tmp/my-disk-clone.img

$ sudo losetup --find --show --partscan /tmp/my-disk-clone.img
/dev/loop0
$ sudo mount /dev/loop0p1 /mnt/myimage
$ cat /mnt/myimage/hello.txt
Hello from my disk image!
```

`cp` = perfect clone of the entire "disk." This is the fundamental concept behind machine images:

```bash
$ lsblk
NAME          MAJ:MIN RM SIZE RO TYPE MOUNTPOINTS
loop0           7:0    0  64M  0 loop
└─loop0p1     259:5    0  62M  0 part /mnt/myimage
nvme0n1       259:0    0   8G  0 disk
├─nvme0n1p1   259:1    0   8G  0 part /
├─nvme0n1p127 259:2    0   1M  0 part
└─nvme0n1p128 259:3    0  10M  0 part /boot/efi
```

The kernel treats the loop device and the real NVMe disk identically. Both are block devices with partitions and filesystems. The filesystem doesn't know or care what's underneath.

---

## The Connection to AMIs

What we just did manually is exactly what happens when you create and launch an AMI:

| Our exercise | AWS equivalent |
|---|---|
| `dd if=/dev/zero` → create blank file | Not needed (starts from parent snapshot) |
| `parted mklabel gpt` | GPT already in the parent |
| `losetup` → attach as block device | Snapshot → EBS volume (attached as NVMe) |
| `mount` → access filesystem | Kernel mounts root at boot |
| Write files into mounted image | Install packages, configure services |
| `umount` + `losetup -d` | Flush all writes |
| `cp my-disk.img clone.img` | `create-image` → EBS snapshot → new volume on launch |

---

## Summary

**What we proved:** a disk image is just a regular file containing raw disk bytes. Nothing magic about it. You can create one, put a partition table and filesystem in it, write files into it, detach it, copy it, and the copy works identically.

**The process:**

```
dd (blank file)
  → parted (add GPT partition table)
    → parted (define a partition)
      → losetup (make the file appear as a block device)
        → mkfs (create a filesystem on the partition)
          → mount (access it as a directory)
            → write files (they go into the .img file)
              → umount + losetup -d (detach everything)
                → the .img file is a complete, self-contained disk image
```

**Key tools:**

| Tool | What it does | Man page |
|---|---|---|
| `dd` | Reads/writes raw bytes between files and devices | [man dd](https://man7.org/linux/man-pages/man1/dd.1.html) |
| `parted` | Creates and reads partition tables | [man parted](https://man7.org/linux/man-pages/man8/parted.8.html) |
| `losetup` | Makes a regular file appear as a block device | [man losetup](https://man7.org/linux/man-pages/man8/losetup.8.html) |
| `mkfs.ext4` | Creates an ext4 filesystem on a partition | [man mkfs.ext4](https://man7.org/linux/man-pages/man8/mkfs.ext4.8.html) |
| `mount` | Attaches a filesystem to a directory | [man mount](https://man7.org/linux/man-pages/man8/mount.8.html) |

**Key takeaways:**
* The .img file IS the disk. Writes to the mounted filesystem go directly into the file's bytes.
* `cp image.img clone.img` = perfect disk clone. This is the concept behind AMI snapshots.
* The kernel treats a loop device identically to a real disk. Filesystems can't tell the difference.
* AMI build tools do this same process at scale: construct an image file, install an OS into it, then upload the bytes to AWS.

---

## Further Reading

* [man dd](https://man7.org/linux/man-pages/man1/dd.1.html) (the raw byte copy tool)
* [man losetup](https://man7.org/linux/man-pages/man8/losetup.8.html) (loop device setup)
* [man mkfs.ext4](https://man7.org/linux/man-pages/man8/mkfs.ext4.8.html)
* [man mount](https://man7.org/linux/man-pages/man8/mount.8.html)
* [Loop device on Wikipedia](https://en.wikipedia.org/wiki/Loop_device) (good conceptual overview)
* [The Linux kernel loop device documentation](https://www.kernel.org/doc/html/latest/admin-guide/blockdev/loop.html)
* [man parted](https://man7.org/linux/man-pages/man8/parted.8.html)

---

**Next:** [Chapter 5: Amazon Machine Images →](05-amazon-machine-images.md)
