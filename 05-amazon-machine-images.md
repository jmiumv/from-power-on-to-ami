# Chapter 5: Amazon Machine Images (AMIs)

> **Environment note:** This chapter requires an AWS account and the AWS CLI. Unlike chapters 1 through 4, the concepts here are AWS specific. That said, if you understood disk images from Chapter 4, you already know 90% of what an AMI is. This chapter just shows how AWS wraps that concept.

## What is an AMI?

An AMI is Amazon's wrapper around the disk image concept we just explored. But here's something that surprises most people: **an AMI doesn't actually contain your disk data**. It's not a big file with 8GB of bytes inside it. It's a small record in the EC2 database that says:

* "The architecture is x86_64"
* "Boot using UEFI"
* "The root disk should be created from this EBS snapshot: snap-xxxxx"

That's it. The AMI is just metadata plus a pointer. The heavy lifting (the actual 8GB of disk bytes, the GPT table, the filesystem, all the files) lives in the **EBS snapshot** that the AMI points to. Think of it like a recipe card that says "for the cake, use the one in the freezer." The card itself is tiny. The cake is big and stored elsewhere.

This matters because:
* Multiple AMIs can point to the same snapshot (different configs, same base image)
* Copying an AMI across regions means copying the snapshot (the big thing), not just the AMI record
* When you "launch from an AMI," what's actually happening is AWS restoring the snapshot to a fresh EBS volume

**AMI = metadata + pointer to an EBS snapshot**

---

## Anatomy of an AMI

> **Note:** Everything in this section uses publicly available data. The AMI we're examining (`ami-00563078bca04e287`) is a public Amazon Linux AMI owned by AWS (account `137112412989`). All its metadata is visible to any AWS user via `describe-images`. You can run these same commands in your own account and get the same results.

Let's examine the AMI our instance was launched from. First, query the [instance metadata service (IMDS)](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html). This is a special HTTP endpoint at `169.254.169.254` available only from inside an EC2 instance. It lets the instance discover information about itself. The first `curl` gets a session token (required by IMDSv2 for security), and the second uses that token to fetch the AMI ID:

```bash
$ TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 60")
$ curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/ami-id
ami-00563078bca04e287
```

Now let's get the full details using the [AWS CLI](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-images.html). `describe-images` returns all metadata AWS stores about an AMI:

```bash
$ aws ec2 describe-images --image-ids ami-00563078bca04e287 --region us-west-2
{
    "Images": [{
        "Architecture": "x86_64",
        "ImageId": "ami-00563078bca04e287",
        "ImageLocation": "amazon/al2023-ami-2023.11.20260514.0-kernel-6.1-x86_64",
        "Public": true,
        "OwnerId": "137112412989",
        "State": "available",
        "BlockDeviceMappings": [{
            "DeviceName": "/dev/xvda",
            "Ebs": {
                "SnapshotId": "snap-001b43e1ff0063864",
                "VolumeSize": 8,
                "VolumeType": "gp3",
                "Iops": 3000,
                "Throughput": 125,
                "DeleteOnTermination": true
            }
        }],
        "RootDeviceName": "/dev/xvda",
        "RootDeviceType": "ebs",
        "BootMode": "uefi-preferred",
        "VirtualizationType": "hvm",
        "ImdsSupport": "v2.0"
    }]
}
```

### Breaking it down

**Identity:**
| Field | Value | Meaning |
|---|---|---|
| `ImageId` | `ami-00563078bca04e287` | Unique AMI identifier |
| `OwnerId` | `137112412989` | Amazon's account (public AMI) |
| `Architecture` | `x86_64` | Only runs on x86_64 instances |

**The disk (the core of the AMI):**
```json
"BlockDeviceMappings": [{
    "DeviceName": "/dev/xvda",
    "Ebs": {
        "SnapshotId": "snap-001b43e1ff0063864",
        "VolumeSize": 8,
        "VolumeType": "gp3"
    }
}]
```

This says: "When launching, create an 8GB gp3 EBS volume from snapshot `snap-001b43e1ff0063864` and attach it as the root device."

You might notice that `DeviceName` says `/dev/xvda` but on our instance the disk showed up as `/dev/nvme0n1`. What gives? `/dev/xvda` is the legacy **Xen** naming convention (`xvd` = Xen Virtual Disk, `a` = first disk). AWS still uses these names in AMI metadata and the API for backwards compatibility. But modern Nitro-based instances present the volume as NVMe, so the actual device is `/dev/nvme0n1`. The kernel handles this translation. You can think of `/dev/xvda` as the "API name" and `/dev/nvme0n1` as the "actual device name inside the instance." See [AWS docs on device naming](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/device_naming.html) for the full mapping.

**Boot configuration:**
| Field | Meaning |
|---|---|
| `RootDeviceName: /dev/xvda` | Which device to boot from |
| `RootDeviceType: ebs` | Boot from EBS (vs ephemeral instance-store) |
| `BootMode: uefi-preferred` | Use UEFI if supported |
| `VirtualizationType: hvm` | Full hardware virtualization |

---

## The Snapshot: Where the Bytes Live

We just saw that the AMI points to a snapshot (`snap-001b43e1ff0063864`). The AMI itself is just metadata. The actual disk bytes, all 8GB of them (GPT, partitions, filesystem, every file), live in that snapshot. Let's look at the snapshot directly to understand what it is and how it was created:

```bash
$ aws ec2 describe-snapshots --snapshot-ids snap-001b43e1ff0063864 --region us-west-2
{
    "Snapshots": [{
        "Description": "/tmp/tmpmq71is09/al2023-ami-2023.11.20260514.0-kernel-6.1-x86_64.xfs.gpt",
        "SnapshotId": "snap-001b43e1ff0063864",
        "VolumeId": "vol-ffffffff",
        "VolumeSize": 8,
        "State": "completed",
        "OwnerId": "137112412989"
    }]
}
```

Two crucial revelations:

**1. The Description** is `/tmp/tmpmq71is09/al2023-ami-...xfs.gpt`

Amazon built this AMI from a **file in /tmp**, just like our exercise! The filename even tells us: XFS filesystem, GPT partition table.

**2. `VolumeId: vol-ffffffff`**

This is a placeholder meaning "no real EBS volume ever existed." Amazon uploaded the raw bytes directly to snapshot storage via the **EBS Direct API**. They built an image file and pushed it straight to S3-backed storage without creating an intermediate EBS volume.

---

## The AMI Mental Model

```
┌─────────────────────────────────────┐
│  AMI: ami-xxxxxxxxxxxxx             │
│                                      │
│  Metadata:                           │
│    - Architecture: x86_64            │
│    - Boot mode: UEFI                 │
│    - Root device: /dev/xvda          │
│                                      │
│  Block Device Mappings:              │
│    /dev/xvda → snap-xxxxx ──────────────┐
└─────────────────────────────────────┘   │
                                           ▼
                            ┌──────────────────────────┐
                            │  EBS Snapshot             │
                            │  8GB of raw disk bytes    │
                            │  (GPT + partitions +      │
                            │   XFS + all files)        │
                            └──────────────────────────┘
                                           │
                                  On launch:
                                           ▼
                            ┌──────────────────────────┐
                            │  New EBS Volume           │
                            │  8GB gp3, 3000 IOPS      │
                            │  Attached as NVMe         │
                            │  UEFI boots from it       │
                            └──────────────────────────┘
```

---

## Creating Your Own AMI

So far we've been examining an AMI that Amazon built. But the whole point of understanding AMIs is so you can create your own. Why would you want to? A few common scenarios:

* You've configured an instance exactly how you want it (installed packages, tuned settings, deployed your app) and you want to launch more copies of it.
* You want a "golden image" that your team standardizes on, so every new instance starts from a known good state.
* You're building a deployment pipeline where new code means a new AMI.

There are two fundamentally different ways to create an AMI, and understanding the difference connects everything we've learned.

### Method 1: Snapshot a running instance

This is the most common approach and what tools like [EC2 Image Builder](https://docs.aws.amazon.com/imagebuilder/latest/userguide/what-is-image-builder.html) do under the hood: launch an instance, configure it (install packages, apply patches, harden security), then snapshot its disk. The resulting snapshot becomes your new AMI.

```bash
$ aws ec2 create-image \
  --instance-id <your-instance-id> \
  --name "my-first-ami-$(date +%Y%m%d)" \
  --description "My hand-built AMI" \
  --no-reboot
{
    "ImageId": "ami-0123456789example"
}
```

What this does:
1. Snapshots the live EBS volume (copies all bytes at that moment)
2. Registers a new AMI pointing to that snapshot
3. `--no-reboot` means don't stop the instance (faster but snapshot may be slightly inconsistent)

Now let's look at the snapshot that was created. The key thing to notice is the `VolumeId` field, which tells us how this snapshot was made:

```bash
$ aws ec2 describe-snapshots --snapshot-ids <new-snap-id>
{
    "Description": "Created by CreateImage(i-xxxxx) for ami-xxxxx",
    "VolumeId": "vol-0abc123def456789",  ← real volume (not vol-ffffffff!)
    "VolumeSize": 8,
    "State": "completed"
}
```

Notice `VolumeId` is a real volume ID this time, because we snapshotted a running instance's actual volume.

### Method 2: Upload raw bytes via EBS Direct API

Method 1 requires a running EC2 instance for the entire duration of the build. You're paying for that instance, and you need network access to install packages. But think back to Chapter 4: we built a disk image as a file on our local machine. We didn't need a separate instance to do it.

The [EBS Direct API](https://docs.aws.amazon.com/ebs/latest/APIReference/Welcome.html) lets you upload raw disk image bytes directly into an EBS snapshot, without ever creating an EC2 instance or EBS volume. You can build the image on your laptop, in a CI/CD pipeline, or on a build server, then push the bytes to AWS. No EC2 Image Builder costs, no running instance during the build.

The open source tool [Coldsnap](https://github.com/awslabs/coldsnap) wraps the EBS Direct API and makes this straightforward. The process is:

1. Build an .img file locally (like our Chapter 4 exercise, but with a real OS installed)
2. Upload the raw bytes directly to EBS snapshot storage via Coldsnap
3. Register an AMI pointing to that snapshot

Result: `VolumeId: vol-ffffffff` (no real volume ever existed).

```
Method 1 (snapshot live instance):
  Running Instance → EBS Volume → Snapshot → AMI
  VolumeId = real

Method 2 (upload raw bytes):
  .img file → EBS Direct API → Snapshot → AMI
  VolumeId = vol-ffffffff (no volume ever existed)
```

---

## Launching from Your AMI

We've created an AMI. Now let's close the loop and launch a new instance from it. This is the whole reason AMIs exist: you build once, launch many. Every launch produces an identical clone (same OS, same packages, same configs) but with a fresh identity.

The [`run-instances`](https://docs.aws.amazon.com/cli/latest/reference/ec2/run-instances.html) command creates a new instance from an AMI. We specify which AMI to use, what size instance, and which SSH key to attach for login:

```bash
$ aws ec2 run-instances \
  --image-id ami-0123456789example \
  --instance-type t3.micro \
  --key-name os-labs
```

Behind the scenes, here's what AWS does:
1. Reads the AMI's block device mappings
2. Creates a new 8GB gp3 EBS volume from the snapshot
3. Attaches it to the new instance as NVMe
4. UEFI firmware boots the chain: ESP → GRUB → kernel → systemd

### Verifying the clone

How do we know the new instance is actually a clone of our AMI? Let's SSH in and check two things: that the disk layout is identical (same partitions, same structure), and that the instance knows which AMI it came from:

```bash
# SSH into the new instance
$ lsblk
NAME          MAJ:MIN RM SIZE RO TYPE MOUNTPOINTS
nvme0n1       259:0    0   8G  0 disk
├─nvme0n1p1   259:1    0   8G  0 part /
├─nvme0n1p127 259:2    0   1M  0 part
└─nvme0n1p128 259:3    0  10M  0 part /boot/efi

$ curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/ami-id
ami-0123456789example
```

Same disk layout, same OS, same configs, but fresh identity (new IP, new instance-id).

---

## What's Captured vs What's Not

This is one of the most important things to understand when working with AMIs. People often get bitten by assuming something is in their AMI when it isn't (e.g., "why is my temp file gone?") or worry that something sensitive is captured when it shouldn't be. Here's the definitive list:

```
CAPTURED (in the snapshot / AMI):
  ✓ GPT partition table
  ✓ EFI System Partition (bootloader)
  ✓ Root filesystem with ALL files
  ✓ Kernel, initramfs, GRUB config
  ✓ Installed packages, service configs
  ✓ Anything on the EBS root volume

NOT CAPTURED:
  ✗ RAM contents (processes, page tables)
  ✗ Anything on tmpfs mounts (lives in RAM only, gone on reboot)
    Note: on our instance /tmp and /run are tmpfs, but this is
    configurable. On some systems /tmp is a real on-disk partition
    and WOULD be captured. Check with `df -hT` to see what's what.
  ✗ /proc, /sys, generated by the kernel on boot
  ✗ Instance identity (IP, hostname, instance-id)
  ✗ Additional EBS volumes (only root by default)
```

---

## The Full Lifecycle

Let's zoom out and see the entire journey of an AMI from creation to running instances. This ties together everything from all five chapters:

```
        BUILD                    STORE                  LAUNCH
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────────┐
│ Install OS      │    │                  │    │ Restore snapshot    │
│ Configure       │───►│  EBS Snapshot    │───►│ → new EBS volume    │
│ Write files     │    │  (stored bytes)  │    │ Attach as NVMe      │
│ Create snapshot │    │                  │    │ UEFI → GRUB →       │
└─────────────────┘    │  AMI             │    │ Kernel → systemd    │
                       │  (metadata +     │    │ → running instance  │
                       │   snap pointer)  │    └─────────────────────┘
                       └──────────────────┘
                              │
                              │ Can launch unlimited instances
                              │ from the same AMI
                              ▼
                       ┌─────────────────────┐
                       │ Instance A (clone)  │
                       │ Instance B (clone)  │
                       │ Instance C (clone)  │
                       └─────────────────────┘
```

---

## Summary

**What an AMI is:**
* Metadata (architecture, boot mode, device names) plus a pointer to one or more EBS snapshots
* The snapshot contains the raw disk bytes (GPT, partitions, filesystem, all files)
* The AMI itself is tiny. The data lives in the snapshot.

**Two ways to create an AMI:**
* Snapshot a running instance's volume (`create-image`). Results in a real VolumeId.
* Upload raw bytes via EBS Direct API (how Amazon and advanced build tools do it). Results in `VolumeId: vol-ffffffff`.

**What happens on launch:**
1. AWS creates a new EBS volume from the snapshot bytes
2. Attaches it to the instance as NVMe
3. UEFI boots the chain: ESP → GRUB → kernel → systemd
4. Fresh RAM, fresh page tables, fresh identity. Same disk content.

**What's captured vs not:**
* Captured: everything on the EBS root volume (partition table, filesystem, all files, kernel, bootloader)
* Not captured: RAM, tmpfs, /proc, /sys, instance identity (IP, hostname)

**Connecting everything we learned:**
* The boot process (Chapter 1) runs because the AMI contains GRUB, kernel, initramfs, systemd
* Memory (Chapter 2) starts fresh every launch. New page tables, new ASLR.
* Storage (Chapter 3) is what the AMI actually captures: the block device with its partitions and filesystems
* Disk images (Chapter 4) are the underlying concept. An AMI snapshot is just a disk image stored in AWS infrastructure.

---

## What's Next?

You now have the complete picture from power-on to AMI lifecycle. The foundations covered here apply to any AMI building tool, whether you're using `aws ec2 create-image`, Packer, EC2 Image Builder, or more advanced tools that build images at the byte level using the EBS Direct API.

The key insight: **an AMI is just a stored copy of disk bytes, and launching an instance is just restoring those bytes to a fresh volume and booting from them.**

---

## Further Reading

* [AWS: Amazon Machine Images (AMI) Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html)
* [AWS: EBS Snapshots](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSSnapshots.html)
* [AWS: EBS Direct APIs](https://docs.aws.amazon.com/ebs/latest/APIReference/Welcome.html) (how tools upload raw bytes as snapshots)
* [AWS: Instance Metadata Service (IMDS)](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html)
* [AWS: Block Device Mappings](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/block-device-mapping-concepts.html)
* [AWS: create-image CLI reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/create-image.html)
* [AWS: register-image CLI reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/register-image.html)
* [AWS: Nitro System](https://aws.amazon.com/ec2/nitro/) (the hardware that presents EBS as NVMe)
* [Coldsnap (open source EBS Direct API tool)](https://github.com/awslabs/coldsnap)
