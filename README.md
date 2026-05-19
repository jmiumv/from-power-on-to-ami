# From Power On to AMI

A hands on tutorial that takes you from "what happens when a computer turns on?" all the way to understanding Amazon Machine Images at the byte level.

## Why does this exist?

I work on AMI infrastructure and I kept getting the same questions from colleagues: "what actually IS an AMI?", "what happens when we bake one?", "why does the boot process matter?". These are smart engineers, but the knowledge is scattered across man pages, random blog posts, and tribal knowledge that never got written down.

I had all of this in my head but never documented it properly. So I sat down with a live EC2 instance, traced every layer from firmware to filesystem, and wrote up what I found. Every concept here is backed by a real command you can run yourself. No hand waving.

If you've ever wondered what actually happens between "Launch Instance" and getting a shell prompt, or why AMI baking works the way it does, this is for you.

## Philosophy

If we claim something exists, we `cat`, `hexdump`, or `ls` it to prove it. Theory is useless without practice. Every chapter is designed so you can follow along on a real instance and see the same output.

## Prerequisites

* Any Linux machine (Ubuntu laptop, Raspberry Pi, a VM, or an EC2 instance)
* Basic terminal/SSH knowledge
* Curiosity
* **For Chapter 5 only:** an AWS account with permissions to launch EC2 instances

## Setup

Chapters 1 through 4 work on **any Linux machine**. If you already have an Ubuntu laptop, a VM, or any Linux box with root access, you can follow along there. Your output will differ slightly (different kernel version, `ext4` instead of `xfs`, `/dev/sda` instead of `/dev/nvme0n1`) but the concepts are identical.

If you want to follow along exactly as written, or if you need a clean environment to experiment with, launch a t3.micro Amazon Linux 2023 instance:

```bash
# Create a key pair
aws ec2 create-key-pair --key-name os-labs --query 'KeyMaterial' --output text > os-labs.pem
chmod 400 os-labs.pem

# Launch instance (uses latest AL2023 AMI)
aws ec2 run-instances \
  --image-id resolve:ssm:/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --instance-type t3.micro \
  --key-name os-labs \
  --associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=os-fundamentals-lab}]'

# SSH in once it's running
ssh -i os-labs.pem ec2-user@<instance-public-ip>
```

## Chapters

Jump to any chapter directly. They build on each other, but if you already know the basics you can skip ahead.

| # | Topic | What you'll understand after |
|---|-------|------------------------------|
| [Chapter 1: The Boot Process](01-boot-process.md) | UEFI, GRUB, kernel, systemd | The full chain from power on to login prompt |
| [Chapter 2: Memory & Page Tables](02-memory-management.md) | Virtual memory, MMU, TLB | How processes are isolated, virtual vs physical memory |
| [Chapter 3: Storage & Filesystems](03-storage-fundamentals.md) | Block devices, GPT, XFS, mounting | How data is organized on disk |
| [Chapter 4: Disk Images](04-disk-images.md) | dd, loop devices, raw images | What a machine image is at the byte level |
| [Chapter 5: Amazon Machine Images](05-amazon-machine-images.md) | AMIs, snapshots, EBS Direct API | AMI anatomy, creating and launching AMIs |

## The Big Picture

```
Chapter 1: Boot
  UEFI firmware → GRUB bootloader → Linux kernel → systemd → your shell

Chapter 2: Memory
  Virtual addresses → page tables (4 level tree) → physical RAM
  Each process isolated in its own 128TB virtual world

Chapter 3: Storage
  Block device → GPT partition table → partitions → filesystem → files
  Everything mounted into one unified directory tree

Chapter 4: Disk Images
  A disk image is a raw copy of disk bytes in a file
  Loop devices let you mount and modify image files

Chapter 5: AMIs
  AMI is metadata plus a pointer to an EBS snapshot
  Snapshot is stored disk bytes (like a .img file in AWS infrastructure)
  Launch restores the snapshot to a new volume and boots from it
```

## Who is this for?

* Engineers who want to deeply understand what an AMI actually is
* Anyone debugging boot issues, disk layouts, or image building
* People who learn by doing, not just reading

## A note on accuracy

I'm not a kernel developer or a systems architect. I'm an engineer who wanted to deeply understand how AMIs work, and this tutorial is the result of that learning process. I did my best to verify every claim with real commands and official documentation, but there may be places where I oversimplified, got a detail wrong, or where things have changed since I wrote this.

This is how I see it. Others may see it differently, and that's fine. If something is wrong, I genuinely want to know.

## Contributing

PRs and criticism are very welcome. In fact, please open a PR if you:

* Spot a technical inaccuracy (please include evidence: a command to run, a link to docs, or a reference to source code)
* Find an explanation confusing or incomplete
* Want to add a callout for a different distro (Ubuntu, Fedora, Arch, etc.)
* Have a better way to explain a concept
* Find a broken link or typo

I'd rather have someone point out I'm wrong than let readers learn incorrect information. Don't be shy.

Also, if there's a topic you'd like me to dive deeper into (swap, LVM, RAID, cloud-init, kernel modules, whatever), open an issue and I'll consider adding a chapter for it.

