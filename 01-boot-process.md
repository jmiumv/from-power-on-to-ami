# Chapter 1: The Boot Process

> **Environment note:** We're running Amazon Linux 2023 on an EC2 t3.micro instance. If you're on Ubuntu or another distro, the concepts are identical but some details will differ: your kernel version will be different, paths to GRUB configs may vary slightly (`/boot/grub/` vs `/boot/grub2/`), and your filesystem might be `ext4` instead of `xfs`. Where it matters, we'll call out what to expect on other distros.

## What happens when a computer turns on?

When you launch an EC2 instance (or press the power button on any computer), a chain of software runs before you get a login prompt:

```
Power On
  → Firmware (UEFI)          "which disk do I boot from?"
    → Bootloader (GRUB)       "which kernel do I load?"
      → Linux Kernel          "I'm the OS now, let me set up hardware"
        → systemd (PID 1)    "let me start all the services"
          → You get a login prompt
```

Let's prove each link exists on a live system.

---

## Step 1: BIOS or UEFI?

**UEFI** (Unified Extensible Firmware Interface) is the first code that runs. It lives on a firmware chip, not on the disk. Its job: find a bootable disk and load the next piece of software from it. (See the [UEFI Spec, Boot Manager section](https://uefi.org/specs/UEFI/2.11/02_Overview.html#boot-manager) for the full definition of the boot flow.)

```bash
$ ls /sys/firmware/efi/
config_table  efivars  fw_platform_size  fw_vendor  runtime  runtime-map  systab
```

`ls` lists directory contents. `/sys/firmware/efi/` is a virtual directory where the kernel exposes UEFI firmware information. If this directory exists, the system booted via UEFI.

The directory exists → this instance booted with **UEFI** (if you got "No such file or directory", it would be legacy BIOS).

You can also check what UEFI version the firmware is running:

```bash
$ cat /sys/firmware/efi/fw_platform_size
64
```

This tells you it's a 64-bit UEFI implementation. For more detail on the firmware revision, check the `systab` file:

```bash
$ sudo cat /sys/firmware/efi/systab
ACPI20=0x3935d014
ACPI=0x3935d000
SMBIOS=0x3926a000
```

The kernel also logs the UEFI version during boot. You can find it with:

```bash
$ sudo dmesg | grep -i "efi:"
[    0.000000] efi: EFI v2.70 by EDK II
```

This tells us: UEFI specification version 2.70, implemented by [EDK II](https://github.com/tianocore/edk2) (the open source UEFI reference implementation).

A quick note on **hypervisors**: on EC2, you're not running on bare metal. A hypervisor is a thin software layer that sits between the physical hardware and your virtual machine. It creates the illusion of dedicated hardware (CPU, RAM, firmware chip, NVMe disk) for each instance, while actually sharing one physical server across many instances. EC2 uses the [Nitro hypervisor](https://aws.amazon.com/ec2/nitro/), which is based on [KVM](https://www.linux-kernel.org/doc/html/latest/virt/kvm/index.html) (Kernel-based Virtual Machine). KVM is a Linux kernel module that turns the Linux kernel itself into a hypervisor. It leverages hardware virtualization extensions built into modern CPUs (Intel VT-x, AMD-V) to run virtual machines at near-native speed. It's the most widely used hypervisor in the cloud industry. It's Nitro that provides the simulated UEFI firmware (built on EDK II) that your instance boots from. You can confirm this:

```bash
$ sudo dmesg | grep -i hypervisor
[    0.000000] Hypervisor detected: KVM
```

On a physical machine (your laptop, a server in a rack), there's no hypervisor. The UEFI firmware lives on an actual chip on the motherboard. On EC2, Nitro simulates that chip. Either way, from the OS's perspective, the boot process is identical.

### UEFI vs BIOS

| | BIOS (old) | UEFI (modern) |
|---|---|---|
| Partition table | MBR (max 2TB disks, 4 partitions) | GPT (huge disks, 128+ partitions) |
| Boot process | Loads first 512 bytes of disk | Reads files from a FAT32 partition |
| Architecture | 16-bit | 32/64-bit |

> Don't worry if MBR and GPT don't mean anything yet. We cover partition tables in detail in [Chapter 3](03-storage-fundamentals.md). For now, just know that UEFI uses the newer GPT format and BIOS uses the older MBR format.

**Key concept:** UEFI lives on a chip (on EC2, the Nitro hypervisor simulates this). Even with no disk attached, this code runs. It then looks for a special partition on the disk called the **EFI System Partition (ESP)**.

---

## Step 2: Find the Disk

So UEFI runs and needs to find a disk with an EFI System Partition. Let's see what disks and partitions exist on this machine. [`lsblk`](https://man7.org/linux/man-pages/man8/lsblk.8.html) (list block devices) shows all disks and their partitions in a tree format:

```bash
$ lsblk
NAME          MAJ:MIN RM SIZE RO TYPE MOUNTPOINTS
nvme0n1       259:0    0   8G  0 disk
├─nvme0n1p1   259:1    0   8G  0 part /
├─nvme0n1p127 259:2    0   1M  0 part
└─nvme0n1p128 259:3    0  10M  0 part /boot/efi
```

The columns: NAME (device name), SIZE, TYPE (disk or partition), and MOUNTPOINTS (where it's attached in the directory tree).

Our 8GB disk (`nvme0n1`) has 3 partitions:
- **p1** (8GB): root filesystem, mounted at `/`
- **p127** (1MB): BIOS boot fallback
- **p128** (10MB): **EFI System Partition**, mounted at `/boot/efi`

The naming `nvme0n1`:
- `nvme` = the protocol (Non-Volatile Memory Express, fast storage over PCIe)
- `0` = first controller
- `n1` = first namespace (disk)
- `p1`, `p127`, `p128` = partitions

On EC2, this is your EBS volume presented as a local NVMe drive by the Nitro hardware.

---

## Step 3: The EFI System Partition

UEFI looks for the partition marked as type "EFI System Partition" and reads its contents. It's formatted as FAT (a simple filesystem UEFI's firmware can read).

```bash
$ sudo ls /boot/efi/EFI
BOOT  amzn

$ sudo ls -la /boot/efi/EFI/BOOT/
-rwx------. 1 root root 1341184 Dec 17 10:24 BOOTX64.EFI

$ sudo ls -la /boot/efi/EFI/amzn/
-rwx------. 1 root root  134 May 15 00:28 grub.cfg
```

Reading `ls -la` columns: permissions, link count, owner, group, **size in bytes**, date, filename. So `1341184` = ~1.3MB and `134` = 134 bytes.

Two things here:
- **`BOOTX64.EFI`** (1.3MB): the bootloader binary. UEFI loads this into RAM and executes it.
- **`grub.cfg`** (134 bytes): a tiny pointer file.

```bash
$ sudo cat /boot/efi/EFI/amzn/grub.cfg
search.fs_uuid afcd3653-4916-4996-a80d-b02b83e8be92 root
set no_modules=y
set prefix=($root)'/boot/grub2'
configfile $prefix/grub.cfg
```

This says: "Find the partition with UUID `afcd3653...`, then read `/boot/grub2/grub.cfg` on it for the real configuration."

### Verifying the UUID → partition mapping

[`blkid`](https://man7.org/linux/man-pages/man8/blkid.8.html) (block ID) shows the UUID, filesystem type, and label of each partition:

```bash
$ sudo blkid
/dev/nvme0n1p1: LABEL="/" UUID="afcd3653-4916-4996-a80d-b02b83e8be92" TYPE="xfs" PARTLABEL="Linux"
/dev/nvme0n1p128: UUID="75B0-447F" TYPE="vfat" PARTLABEL="EFI System Partition"
/dev/nvme0n1p127: PARTLABEL="BIOS Boot Partition"
```

UUID `afcd3653...` = `/dev/nvme0n1p1` (the root partition). UUIDs are permanent IDs because device names like `nvme0n1p1` can change between boots, but UUIDs never do.

---

## Step 4: GRUB Bootloader

GRUB reads its real configuration and finds out which kernel to load. The `*.conf` files in `/boot/loader/entries/` follow the [Boot Loader Specification](https://systemd.io/BOOT_LOADER_SPECIFICATION/) format, with one file per available kernel:

```bash
$ sudo cat /boot/loader/entries/*.conf
title Amazon Linux (6.1.170-213.321.amzn2023.x86_64) 2023
version 6.1.170-213.321.amzn2023.x86_64
linux /boot/vmlinuz-6.1.170-213.321.amzn2023.x86_64
initrd /boot/initramfs-6.1.170-213.321.amzn2023.x86_64.img
options root=UUID=afcd3653-4916-4996-a80d-b02b83e8be92 ro console=tty0 console=ttyS0,115200n8 selinux=1 security=selinux quiet
```

This tells GRUB:
- **`linux`**: load this kernel file
- **`initrd`**: also load this initial RAM filesystem
- **`options`**: pass these parameters to the kernel (root UUID, console settings, SELinux)

> **What is SELinux?** You'll notice `selinux=1 security=selinux` in the kernel options. [SELinux](https://www.redhat.com/en/topics/linux/what-is-selinux) (Security-Enhanced Linux) is a mandatory access control system built into the kernel. Normal Linux permissions (rwx) only control who can access a file. SELinux goes further: it defines *what* each process is allowed to do, even if it's running as root. For example, SELinux can say "the web server process can read /var/www but nothing else, even though it runs as root." Amazon Linux has SELinux enabled by default. Ubuntu uses a similar system called [AppArmor](https://apparmor.net/) instead. You can check the current status with `getenforce` (should return `Enforcing` or `Permissive`).

---

## Step 5: Kernel and initramfs

GRUB loads two files into RAM:

```bash
$ ls -lh /boot/vmlinuz-6.1.170-213.321.amzn2023.x86_64
-rwxr-xr-x. 1 root root 11M May 14 12:20 /boot/vmlinuz-6.1.170-213.321.amzn2023.x86_64

$ ls -lh /boot/initramfs-6.1.170-213.321.amzn2023.x86_64.img
-rw-------. 1 root root 20M May 15 00:26 /boot/initramfs-6.1.170-213.321.amzn2023.x86_64.img
```

- **vmlinuz (11MB)**: the Linux kernel itself. Manages memory, processes, hardware, everything. The "z" means it's compressed.
- **initramfs (20MB)**: "initial RAM filesystem." A temporary mini-filesystem loaded into RAM with just enough drivers to access the real disk.

### Why initramfs exists

The kernel needs drivers to read the disk. But drivers are files on the disk. Chicken-and-egg problem! initramfs breaks the cycle by providing essential drivers in RAM alongside the kernel.

**However**, on Amazon Linux, the critical drivers are built directly into the kernel:

The file `/boot/config-*` is the kernel's build configuration. It records how every feature was compiled. We use [`grep`](https://man7.org/linux/man-pages/man1/grep.1.html) to search for specific settings:

```bash
$ cat /boot/config-6.1.170-213.321.amzn2023.x86_64 | grep CONFIG_NVME_CORE
CONFIG_NVME_CORE=y

$ cat /boot/config-6.1.170-213.321.amzn2023.x86_64 | grep CONFIG_XFS_FS
CONFIG_XFS_FS=y
```

`=y` means **built-in** (always available). Amazon builds NVMe and XFS into the kernel because EC2 always uses NVMe + XFS. The initramfs here mainly carries CPU microcode security patches. We can peek inside the initramfs image using [`lsinitrd`](https://man7.org/linux/man-pages/man1/lsinitrd.1.html), which lists the files packed inside it:

```bash
$ sudo lsinitrd /boot/initramfs-6.1.170-213.321.amzn2023.x86_64.img | head -12
Image: /boot/initramfs-6.1.170-213.321.amzn2023.x86_64.img: 20M
========================================================================
Early CPIO image
========================================================================
drwxr-xr-x   3 root     root            0 Dec 30 11:45 .
-rw-r--r--   1 root     root            2 Dec 30 11:45 early_cpio
drwxr-xr-x   3 root     root            0 Dec 30 11:45 kernel
drwxr-xr-x   3 root     root            0 Dec 30 11:45 kernel/x86
drwxr-xr-x   2 root     root            0 Dec 30 11:45 kernel/x86/microcode
-rw-r--r--   1 root     root       152098 Dec 30 11:45 kernel/x86/microcode/AuthenticAMD.bin
-rw-r--r--   1 root     root      5673984 Dec 30 11:45 kernel/x86/microcode/GenuineIntel.bin
```

### Built-in vs module drivers

| | Built-in (`=y`) | Module (`=m`) |
|---|---|---|
| Available at | Instant, always | Only after loading |
| Memory usage | Always in RAM | Only when loaded |
| Update | Must recompile kernel | Replace the `.ko` file |
| Use when | Essential for boot | Everything else |

---

## Step 6: systemd (PID 1)

Once the kernel mounts the root filesystem, it launches the first userspace process. We can see it with [`ps`](https://man7.org/linux/man-pages/man1/ps.1.html) (process status). The flags `-p 1` means "show process with PID 1" and `-o pid,comm,args` selects which columns to display:

```bash
$ ps -p 1 -o pid,comm,args
    PID COMMAND         COMMAND
      1 systemd         /usr/lib/systemd/systemd --switched-root --system --deserialize=32
```

- `--switched-root` confirms the kernel started with initramfs then switched to the real root
- Every process on the system is a descendant of PID 1

[`systemctl`](https://man7.org/linux/man-pages/man1/systemctl.1.html) is the tool for interacting with systemd. This command lists all services that are currently running:

```bash
$ systemctl list-units --type=service --state=running
  UNIT                       LOAD   ACTIVE SUB     DESCRIPTION
  acpid.service              loaded active running ACPI Event Daemon
  amazon-ssm-agent.service   loaded active running amazon-ssm-agent
  chronyd.service            loaded active running NTP client/server
  dbus-broker.service        loaded active running D-Bus System Message Bus
  sshd.service               loaded active running OpenSSH server daemon
  systemd-journald.service   loaded active running Journal Service
  systemd-networkd.service   loaded active running Network Configuration
  systemd-udevd.service      loaded active running Rule-based Manager for Device Events and Files
  ...
20 loaded units listed.
```

`sshd.service` is how you're connected right now.

---

## Important: Three Separate Driver Layers

A common misconception: UEFI, GRUB, and Linux share drivers. They don't. Each has its own:

| Software | Its disk driver | Its filesystem driver |
|---|---|---|
| UEFI firmware | Built into firmware ([UEFI Spec, Driver Model](https://uefi.org/specs/UEFI/2.11/02_Overview.html#uefi-driver-model)) | FAT only (simple) |
| GRUB bootloader | `disk.mod`, `ata.mod`, `biosdisk.mod` | `xfs.mod`, `fat.mod`, `part_gpt.mod` |
| Linux kernel | `CONFIG_NVME_CORE=y` | `CONFIG_XFS_FS=y` |

Each re-discovers the disk from scratch. No inheritance between layers.

You can see GRUB's modules yourself:

```bash
$ sudo ls /boot/grub2/i386-pc/ | grep -i "xfs\|fat\|part_gpt\|disk"
disk.mod
fat.mod
part_gpt.mod
xfs.mod
```

These are GRUB's own implementations of XFS, FAT, and GPT, completely separate from the kernel's. When GRUB reads `/boot/vmlinuz` from the XFS root partition, it uses its own `xfs.mod`. Once the kernel takes over, it uses its own built-in XFS code (`CONFIG_XFS_FS=y`). They never share code.

See the [GRUB documentation on modules](https://www.gnu.org/software/grub/manual/grub/html_node/Modules.html) for more detail.

---

## Summary

```
┌──────────────────────────────────────────────────────────────┐
│ Step 1: UEFI firmware (on chip, EFI v2.70 via EDK II)        │
│         Has own disk/filesystem drivers (FAT only)           │
│         Finds EFI System Partition (p128, 10MB, FAT)         │
│         Loads BOOTX64.EFI (GRUB) into RAM                    │
├──────────────────────────────────────────────────────────────┤
│ Step 2: GRUB bootloader (has own drivers: xfs.mod, fat.mod)  │
│         Reads /boot/loader/entries/*.conf                     │
│         Loads vmlinuz (11MB) + initramfs (20MB) into RAM     │
├──────────────────────────────────────────────────────────────┤
│ Step 3: Linux kernel (has own drivers: NVME_CORE, XFS_FS)    │
│         Applies CPU microcode from initramfs                  │
│         Mounts nvme0n1p1 (XFS) at /                          │
│         Switches root from initramfs → real filesystem       │
│         SELinux policy loaded                                 │
├──────────────────────────────────────────────────────────────┤
│ Step 4: systemd (PID 1)                                      │
│         Starts 20 services (sshd, networking, SSM agent...)  │
│         You SSH in                                            │
└──────────────────────────────────────────────────────────────┘

On EC2: Nitro hypervisor (KVM-based) simulates the UEFI firmware
and presents EBS volumes as NVMe devices.
Three driver layers (UEFI, GRUB, kernel) are fully independent.
```

---

## Why This Matters for AMIs

An AMI is a snapshot of the disk that contains ALL of this: GRUB config, kernel, initramfs, root filesystem, all services. When EC2 "launches" an instance, it restores this snapshot to an EBS volume, attaches it, and UEFI starts the chain above. Fresh RAM, fresh page tables, fresh process tree, but the same disk bytes.

---

## Further Reading

* [UEFI Specification, Boot Manager](https://uefi.org/specs/UEFI/2.11/02_Overview.html#boot-manager) (the official standard, specifically how the boot flow works)
* [GNU GRUB Manual](https://www.gnu.org/software/grub/manual/grub/)
* [Boot Loader Specification](https://systemd.io/BOOT_LOADER_SPECIFICATION/) (the BLS format we saw in `/boot/loader/entries/`)
* [man lsblk](https://man7.org/linux/man-pages/man8/lsblk.8.html)
* [man blkid](https://man7.org/linux/man-pages/man8/blkid.8.html)
* [man systemctl](https://man7.org/linux/man-pages/man1/systemctl.1.html)
* [The Linux Kernel Documentation: Boot Process](https://www.kernel.org/doc/html/latest/admin-guide/bootargs.html) (kernel command line parameters)
* [dracut (initramfs generator)](https://man7.org/linux/man-pages/man8/dracut.8.html)
* [lsinitrd man page](https://man7.org/linux/man-pages/man1/lsinitrd.1.html)

---

**Next:** [Chapter 2: Memory Management & Page Tables →](02-memory-management.md)
