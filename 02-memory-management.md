# Chapter 2: Memory Management & Page Tables

> **Environment note:** Everything in this chapter is universal to x86_64 Linux. Page tables, virtual memory, and the MMU work identically whether you're on Ubuntu, Fedora, Amazon Linux, or a Raspberry Pi (ARM has a similar structure with different specifics). The commands and outputs here will look the same on any distro.

## Why RAM? Why not just read everything from disk?

Before we dive in, let's address something basic: why does everything need to be loaded into RAM? We have an 8GB disk with all our files. Why can't the CPU just read from there directly?

**Speed.** RAM is roughly 100,000x faster than disk for random access:

| Storage | Approximate access time | Analogy |
|---|---|---|
| CPU registers | ~1 nanosecond | Blinking |
| RAM | ~100 nanoseconds | Taking a breath |
| NVMe SSD | ~100 microseconds | Making a cup of tea |
| Spinning HDD | ~10 milliseconds | Walking to the shops |

The CPU executes billions of instructions per second. If it had to wait for disk every time it needed the next instruction or variable, it would spend 99.99% of its time just waiting. RAM exists to bridge this gap. The disk holds data permanently; RAM holds the data the CPU is actively working with.

That's why the boot process loads the kernel into RAM. That's why running programs live in RAM. The disk is for storage; RAM is the workspace.

## The Problem

Your instance is running a bunch of processes right now. Take a look with [`ps aux`](https://man7.org/linux/man-pages/man1/ps.1.html) (list all processes) piped to `wc -l` (count lines):

```bash
$ ps aux | wc -l
```

Your number will vary, but the point is: there are many processes and threads all sharing one physical RAM chip. Who decides which process gets what memory? And how do we stop one process from reading another's secrets?

---

## Why virtual addresses?

Imagine a world without virtual memory. Every process uses physical RAM addresses directly. This creates several problems:

1. **No isolation.** Process A could read or overwrite Process B's memory. A bug in sshd could corrupt systemd's data and crash the whole system.

2. **Fragmentation.** As processes start and stop, RAM becomes a patchwork of used and free chunks. A new process might need 10MB of contiguous memory but can only find scattered 1MB holes.

3. **No flexibility.** Every program would need to be compiled knowing exactly where in RAM it will be loaded. Two copies of the same program couldn't run at the same time (they'd both want the same addresses).

4. **No security.** A malicious process could read passwords, encryption keys, or anything else in RAM belonging to other processes.

Virtual memory solves all of these. Each process gets its own private address space. The kernel translates virtual addresses to physical addresses behind the scenes, and processes never see (or can access) each other's memory.

---

## How Much RAM?

[`free -h`](https://man7.org/linux/man-pages/man1/free.1.html) shows memory usage in human-readable sizes (`-h` = human):

```bash
$ free -h
               total        used        free      shared  buff/cache   available
Mem:           916Mi       157Mi       459Mi       0.0Ki       300Mi       622Mi
Swap:             0B          0B          0B
```

916MB total. But here's the trick: every process *thinks* it has the entire memory space to itself.

---

## Virtual Addresses: The Illusion

Every process lives in its own fake address space. Let's see what `cat` thinks its memory looks like. `/proc/self/` is a special directory that always refers to the currently running process, and `maps` shows its memory layout:

```bash
$ cat /proc/self/maps
55d3f7814000-55d3f7816000 r--p 00000000 103:01 662046  /usr/bin/cat
55d3f7816000-55d3f781a000 r-xp 00002000 103:01 662046  /usr/bin/cat
55d3f781d000-55d3f781e000 rw-p 00008000 103:01 662046  /usr/bin/cat
55d41f95e000-55d41f97f000 rw-p 00000000 00:00 0        [heap]
7fc688200000-7fc688229000 r--p 00000000 103:01 8546589 /usr/lib64/libc.so.6
7fc688229000-7fc68839f000 r-xp 00029000 103:01 8546589 /usr/lib64/libc.so.6
7ffdb455c000-7ffdb457d000 rw-p 00000000 00:00 0        [stack]
```

Address `55d3f7814000` in hex = roughly **94 terabytes** into the address space. But we only have 916MB of RAM! These are **virtual addresses**, fake addresses that get translated to real physical locations.

Key regions:
- `r-xp` = executable code (the program itself, libraries)
- `rw-p` = writable data (heap, stack)
- `[heap]` = dynamic allocations
- `[stack]` = function calls and local variables

Notice: no region has both `w` (write) and `x` (execute). This is the **NX bit** (No-eXecute) in action. It's a CPU feature where each memory page is marked as either writable OR executable, but never both. Why? If an attacker manages to inject malicious data into your process's memory (say, via a buffer overflow), they can't execute it because the page is marked as data only. And they can't modify existing code pages because those are read-only. See [NX bit on Wikipedia](https://en.wikipedia.org/wiki/NX_bit) for more detail.

---

## Virtual vs Physical: The Numbers

So if those addresses are fake, how much real memory is each process actually using? The kernel tracks this in `/proc/<pid>/status`. Since `/proc/self` always refers to the currently running process, we can check `cat`'s own stats. The `grep -i vm` filters to only show lines containing "vm" (case insensitive):

```bash
$ cat /proc/self/status | grep -i vm
VmPeak:      221524 kB
VmSize:      221524 kB
VmLck:           0 kB
VmPin:           0 kB
VmHWM:        1024 kB
VmRSS:        1024 kB
VmData:         360 kB
VmStk:         132 kB
VmExe:          16 kB
VmLib:        1664 kB
VmPTE:          56 kB
VmSwap:           0 kB
```

That's a lot of fields. The important ones:

| Metric | Value | What it means |
|---|---|---|
| **VmSize** | 221,524 kB (~216 MB) | Total virtual address space mapped. This is the "illusion" size, not real usage. |
| **VmRSS** | 1,024 kB (~1 MB) | "Resident Set Size." The actual physical RAM this process is using right now. |
| **VmPeak** | 221,524 kB | Largest VmSize has ever been during this process's lifetime. |
| **VmHWM** | 1,024 kB | "High Water Mark." Largest VmRSS has ever been. |
| **VmData** | 360 kB | Memory used by the process's data (heap, variables). |
| **VmStk** | 132 kB | Memory used by the stack (function calls, local variables). |
| **VmExe** | 16 kB | Size of the executable code (the `cat` binary itself). |
| **VmLib** | 1,664 kB | Memory mapped for shared libraries (libc, etc.). |
| **VmPTE** | 56 kB | Size of this process's **page table** (the translation map we'll discuss next). |
| **VmSwap** | 0 kB | How much was swapped to disk (none, since we have no swap). |

The key insight: `cat` thinks it has **216 MB** of address space (VmSize), but it's only physically using **1 MB** of real RAM (VmRSS). The other 215 MB is just mapped virtual space occupied by shared libraries, locale data, and the linker, most of which is shared with other processes or hasn't been touched yet.

Why is VmSize so large for a tiny program like `cat`? Because the virtual address space includes shared libraries (libc alone is several MB of mapped address space) and locale data. But the key word is "mapped," not "using." The kernel only allocates physical RAM for pages that get actually accessed.

---

## Page Tables: The Translation Map

So we have virtual addresses (what the process sees) and physical addresses (where data actually lives in RAM). Something has to translate between them. That something is the **page table**.

A page table is a data structure that the kernel builds for each process. It's essentially a lookup table that says "virtual address X maps to physical address Y." The CPU consults it on every single memory access, billions of times per second.

The name comes from the fact that memory is divided into fixed-size chunks called **pages**. The page table doesn't translate individual bytes, it translates whole pages at a time. The page size depends on the architecture:

| Architecture | Default page size | Notes |
|---|---|---|
| x86 (32-bit) | 4 KB | Same as x86_64 |
| x86_64 (64-bit) | 4 KB | What we're using (see below for "huge pages") |
| ARM (32-bit) | 4 KB | Same default |
| ARM64 / AArch64 | 4 KB, 16 KB, or 64 KB | Configurable at kernel build time. AWS Graviton uses 4KB |

4KB is the most common across all architectures. It's a tradeoff between granularity and overhead. For example, say a process needs 5KB of memory:

- With **4KB pages**: kernel allocates 2 pages (8KB total). 3KB is wasted. But only 2 page table entries to manage.
- With **2MB pages**: kernel allocates 1 huge page (2MB total). 2,043KB is wasted. But only 1 page table entry.

Smaller pages waste less memory per allocation, but you end up with more pages and a bigger page table. Bigger pages waste more memory, but the page table stays small and lookups are faster.

**What about huge pages?** On x86_64, the CPU also supports 2MB and 1GB pages. Why would you want them? Consider a database that uses 32GB of RAM. With 4KB pages, the page table needs 8 million entries. With 2MB pages, it only needs ~16,000 entries. Fewer entries means less memory used by the page table itself, and more importantly, fewer TLB cache misses (we'll cover TLB later). The tradeoff is that huge pages can waste memory if you don't need the full 2MB. They're mostly used by databases, VMs, and other large-memory applications that know they'll use big contiguous chunks. You can check if your system has them registered:

```bash
$ sudo dmesg | grep -i huge
[    0.067406] HugeTLB: registered 1.00 GiB page size, pre-allocated 0 pages
[    0.067413] HugeTLB: registered 2.00 MiB page size, pre-allocated 0 pages
```

"pre-allocated 0 pages" means they're available but none are reserved yet. Applications have to explicitly request them.

Every memory access goes through translation:

```
Process says: "read address 0x55d3f7814000"
         │
         ▼
    ┌──────────┐
    │  Page    │  (per-process, managed by kernel)
    │  Table   │
    │          │
    │ virtual  │──► physical
    │ 55d3f... │──► 0x3A012000 (actual RAM location)
    └──────────┘
         │
         ▼
CPU actually reads physical address 0x3A012000 from the RAM chip
```

The page table doesn't map every single byte individually. Instead, memory is divided into 4KB chunks called **pages**, and the translation happens one page at a time. You can think of it like a postal system: instead of tracking where every individual letter goes, it maps entire ZIP codes. One page table entry covers 4096 consecutive bytes.

You can check the page size on your system:

```bash
$ getconf PAGE_SIZE
4096
```

4096 bytes = 4KB. So both virtual memory and physical memory are split into 4KB pages. The page table maps virtual pages → physical pages. If a process accesses any byte within virtual page #100, the page table translates that to "physical page #5742" (or wherever it actually lives), and the offset within the page stays the same.

---

## Every Process Has Its Own Page Table

```bash
# cat's code lives at this virtual address:
$ cat /proc/self/maps | head -1
55e705a62000-55e705a64000 r--p 00000000 103:01 662046  /usr/bin/cat

# systemd's code lives at a different virtual address:
$ sudo cat /proc/1/maps | head -1
5589fd4ec000-5589fd4f2000 r--p 00000000 103:01 8804420 /usr/lib/systemd/systemd
```

Different addresses, even though they're in the same physical RAM. Each process has its own page table, creating complete isolation.

---

## ASLR: Addresses Change Every Launch

We just saw that different processes have different virtual addresses. But here's something even more interesting: the same program gets different addresses each time you run it. This is a security feature called **ASLR** (Address Space Layout Randomization). Try running the same command twice and compare the addresses:

```bash
$ cat /proc/self/maps | head -1
55e705a62000-55e705a64000 r--p 00000000 103:01 662046  /usr/bin/cat

$ cat /proc/self/maps | head -1
555db9c05000-555db9c07000 r--p 00000000 103:01 662046  /usr/bin/cat
```

Different every time! The kernel deliberately randomizes where it places code in virtual memory. Why? If an attacker finds a buffer overflow vulnerability in your program, they need to know where to jump in memory to execute their payload. ASLR makes that a guessing game, because the target address moves on every launch. See [ASLR on Wikipedia](https://en.wikipedia.org/wiki/Address_space_layout_randomization) for more detail.

---

## The 4-Level Page Table Tree

### Where does "128TB" come from?

x86_64 CPUs have 64-bit addresses, but they don't actually use all 64 bits. Currently, only **48 bits** are used for virtual addresses (the upper 16 bits must be copies of bit 47, a rule called "canonical addressing"). This is a hardware design decision to keep things simpler and cheaper. See the [Intel SDM Volume 1, Section 3.3.7.1](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html) and the [OSDev Wiki on canonical addresses](https://wiki.osdev.org/X86-64#Canonical_Addresses) for the full specification.

```
48 bits of address space = 2^48 bytes = 256 TB total
```

But that 256TB is split in half:
- Bottom half (addresses 0x0000000000000000 to 0x00007FFFFFFFFFFF): **user space** (128TB per process)
- Top half (addresses 0xFFFF800000000000 to 0xFFFFFFFFFFFFFFFF): **kernel space** (128TB, shared)

So each process gets 128TB of virtual addresses for its own use. The kernel lives in the upper half and is shared across all processes (but inaccessible from user mode).

For comparison, on a 32-bit system the math is:

```
32 bits = 2^32 = 4 GB total virtual space (split: 3GB user, 1GB kernel typically)
```

The jump is exponential, not linear. Each additional address bit doubles the space, so 48 bits gives 65,536 times more address space than 32 bits.

128TB is absurdly more than any process needs. Your `cat` command uses a few MB. A large database might use a few hundred GB. The point is that the virtual space is so large you never run out of addresses, even if physical RAM is tiny by comparison.

### The problem: why not a simple flat table?

On x86_64, each process gets 128TB of virtual address space. Let's do the math on what a simple flat lookup table would cost:

```
128 TB of virtual space ÷ 4 KB per page = 34,359,738,368 possible virtual pages
Each page table entry is 8 bytes (needs to hold a 52-bit physical address + flags)
34,359,738,368 × 8 bytes = 256 GB just for one process's page table
```

256 GB for the page table alone. We only have 916MB of RAM total. Obviously impossible.

But here's the thing: `cat` only actually uses a few MB of that 128TB space. The vast majority of the virtual address space is empty. We need a data structure that only stores entries for the pages that are actually mapped, and doesn't waste memory on the empty regions.

### The solution: a tree (only create branches where pages exist)

Think of it like a phone book. A flat array approach would need one slot for every possible phone number (10 billion entries). A tree approach organizes by area code, then prefix, then number. If nobody lives in area code 555, you just don't create that branch. No wasted space.

The x86_64 page table works the same way. It's a 4-level tree:

```
Virtual address (48 bits used):
┌─────────┬─────────┬─────────┬─────────┬──────────────┐
│ 9 bits  │ 9 bits  │ 9 bits  │ 9 bits  │   12 bits    │
│ L4 idx  │ L3 idx  │ L2 idx  │ L1 idx  │   offset     │
└────┬────┴────┬────┴────┬────┴────┬────┴───────┬──────┘
     │         │         │         │            │
     ▼         ▼         ▼         ▼            ▼
   L4 node → L3 node → L2 node → L1 node → physical page + offset
```

The 48-bit virtual address is split into 5 pieces: 9 + 9 + 9 + 9 + 12 = 48 bits. The first four pieces (9 bits each) index into each level of the tree. The last 12 bits are the offset within the final 4KB page (2^12 = 4096, which covers every byte in a page).

### Why 9 bits per level?

Each piece of the address (9 bits) is used as an index into a table. With 9 bits, how many different values can you represent? Each bit can be 0 or 1, so:

```
9 bits → 2 × 2 × 2 × 2 × 2 × 2 × 2 × 2 × 2 = 2^9 = 512 possible values
```

So 9 bits can represent any number from 0 to 511. That means each node in the tree has exactly **512 slots** (one for each possible 9-bit index value).

Now, each slot (entry) needs to store a physical address (where the next level is in RAM) plus some permission flags. On x86_64, that takes 8 bytes per entry. So one complete node takes:

```
512 entries × 8 bytes per entry = 4096 bytes = exactly one 4KB page
```

This is elegant hardware design: every node in the page table fits perfectly into one page of memory. The CPU designers chose 9 bits specifically because it makes each node exactly one page.

### How the tree saves space

Each entry in a node either says "EMPTY" (nothing below this branch) or points to the next level down. If a region of virtual memory is unused, the tree just has an "EMPTY" entry at a high level, and no lower nodes are created.

```
Level 4: 512 slots, but maybe only 3 have entries
  → Only 3 Level 3 nodes are created (not 512!)

Level 3: 3 nodes × 512 slots = 1536 slots, but maybe only 4 have entries
  → Only 4 Level 2 nodes are created

Level 2: 4 nodes × 512 slots = 2048 slots, but maybe only 5 have entries
  → Only 5 Level 1 nodes are created

Level 1: 5 nodes × 512 slots = 2560 slots, ~200 actually map real pages
```

Total: 1 + 3 + 4 + 5 = **13 nodes × 4KB = ~52KB**. Compare that to 256GB for a flat table. We saw earlier that `cat` has VmPTE of 56KB, which matches this nicely.

### Why a tree and not a hash table?

You might wonder: why not use a hash table which also only stores entries that exist? The answer is that the CPU hardware does this translation. The MMU (Memory Management Unit) is a circuit on the chip, not software. It needs a simple, deterministic, fixed-time algorithm it can implement in silicon. "Take 9 bits, index into a table, follow the pointer, repeat 4 times" is easy to build as a circuit. Hash table operations (compute hash, handle collisions, resize) are too complex for hardware.

For more detail, see the [OSDev Wiki on x86_64 Paging](https://wiki.osdev.org/Paging) and the [Linux kernel page table documentation](https://www.kernel.org/doc/html/latest/mm/page_tables.html).

### How does the CPU know where the tree starts?

The tree has to be stored somewhere in physical RAM, and the CPU needs to know where the root node (Level 4) is. This is stored in a special CPU register called **CR3** (Control Register 3). It holds the physical memory address of the current process's Level 4 page table.

When the kernel switches from running one process to another (a "context switch"), it updates CR3 to point to the new process's page table. This is how isolation works at the hardware level: the CPU literally can't see another process's memory because CR3 points to a different tree.

See the [Intel Software Developer Manual, Volume 3, Section 4.5](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html) for the full specification of CR3 and paging structures.

### Example: Translating address `0x55d3f7814abc`

```
Split into: [171] [335] [478] [20] [0xabc]

CPU reads CR3 register → gets physical address of Level 4 table
  L4[171] → points to a Level 3 table
    L3[335] → points to a Level 2 table
      L2[478] → points to a Level 1 table
        L1[20] → physical page address 0x3A012
          + offset 0xabc
          = physical address 0x3A012ABC

4 hops through RAM. The TLB cache makes this instant 99% of the time.
```

### What if you access an unmapped address?

We said that the tree only has branches for memory that's actually mapped. So what happens if a process tries to access a virtual address that has no page table entry? This is the mechanism that enforces memory isolation.

```
Process tries to read virtual address 0x9999deadbeef

CPU walks the tree:
  L4[307] = EMPTY / NOT PRESENT

  CPU has no idea where to go next. There's no Level 3, no Level 2,
  no Level 1, no physical page. The walk stops here.

  MMU raises a "page fault" exception
  → Kernel catches the exception
  → Kernel checks: was this a valid access? (No, nothing is mapped here)
  → Kernel kills the process with signal SIGSEGV
  → You see: "Segmentation fault"
```

This is how one process is prevented from accessing another's memory. The page table simply doesn't contain entries for addresses outside its own mapped regions. The hardware catches the violation before any unauthorized memory access can happen.

---

## Proving Segfaults

Let's trigger a real segfault and see what happens. We'll tell Python to read memory address 0x0 (NULL), which is intentionally never mapped in any process:

```bash
$ python3 -c "import ctypes; ctypes.string_at(0)"
Segmentation fault (core dumped)
```

The kernel also logs the event:

```bash
$ sudo dmesg | tail -2
python3[45097]: segfault at 0 ip 00007f072237077c sp 00007ffc83d48528 error 4 in libc.so.6
```

Reading the log: `segfault at 0` means it tried to access virtual address 0x0. `ip 00007f072237077c` is the instruction pointer (where in the code it crashed). `error 4` means a user-mode read of an unmapped page. The kernel killed the process because the page table had no entry for address 0.

Address 0 is deliberately never mapped on any Linux system. This ensures that NULL pointer dereferences (one of the most common programming bugs) always crash loudly instead of silently reading garbage data.

---

## System-Wide Page Table Stats

Every process has its own page table, and they all consume physical RAM. Let's see the total cost across the whole system:

```bash
$ sudo grep PageTables /proc/meminfo
PageTables:         2256 kB
```

2,256 KB (~2.2MB) of the 916MB total RAM is being used just to store page tables for all running processes. That's the cost of memory isolation.

We can also compare individual processes. We saw `cat` uses 56KB for its page table. Let's check systemd:

```bash
$ sudo grep VmPTE /proc/1/status
VmPTE:         112 kB
```

systemd uses 112KB because it has more code, more libraries loaded, and more data mapped than a simple tool like `cat`. More mapped memory = more tree branches = bigger page table.

---

## Key Hardware

Several hardware components work together to make virtual memory fast and secure:

| Component | Role |
|---|---|
| **MMU** | Memory Management Unit. A circuit inside the CPU that walks the page table on every memory access. Without it, every address translation would need software, which would be thousands of times slower. |
| **CR3 register** | Points to the current process's Level 4 page table. Updated by the kernel on every context switch. |
| **TLB** | Translation Lookaside Buffer. A small cache inside the CPU that stores recent virtual→physical translations. Since the same addresses are accessed repeatedly, the TLB avoids walking the 4-level tree 99%+ of the time. |
| **NX bit** | Marks pages as no-execute, prevents code injection (explained earlier). |
| **IOMMU** | Like a page table, but for PCIe devices. Prevents hardware devices from reading arbitrary RAM via DMA (Direct Memory Access). |

The TLB is worth emphasizing. Without it, every memory access would require 4 extra RAM reads (one per tree level) just for the translation. That would make programs run 5x slower. The TLB stores the most recent translations so subsequent accesses to the same page are instant.

We can see evidence of these components in the kernel's boot log. [`dmesg`](https://man7.org/linux/man-pages/man1/dmesg.1.html) shows kernel messages from boot, and we filter for memory-related entries:

```bash
$ sudo dmesg | grep -i "page size\|mmu\|paging"
[    0.067406] HugeTLB: registered 1.00 GiB page size, pre-allocated 0 pages
[    0.067413] HugeTLB: registered 2.00 MiB page size, pre-allocated 0 pages
[    0.096750] iommu: Default domain type: Translated
```

`iommu: Default domain type: Translated` confirms the IOMMU is active, protecting RAM from unauthorized device access.

---

## Summary

**Why RAM exists:** The CPU is ~100,000x faster than disk. RAM is the workspace where actively used code and data lives. Disk is for permanent storage.

**Why virtual memory:** Without it, processes could read each other's memory, programs would need to know their exact RAM location at compile time, and memory fragmentation would be unmanageable.

**The architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│ Each process gets its own VIRTUAL address space              │
│ (128TB on x86_64, derived from 48-bit addresses ÷ 2)       │
│                                                              │
│  bash          sshd           cat                            │
│  "I have       "I have        "I have                        │
│   128TB"        128TB"         128TB"                        │
│    │              │              │                            │
│    ▼              ▼              ▼                            │
│ ┌──────┐      ┌──────┐      ┌──────┐                       │
│ │Page  │      │Page  │      │Page  │  4-level tree,         │
│ │Table │      │Table │      │Table │  56-112KB each,        │
│ │      │      │      │      │      │  pointed to by CR3     │
│ └──┬───┘      └──┬───┘      └──┬───┘                       │
│    │              │              │                            │
│    ▼              ▼              ▼                            │
│ ┌──────────────────────────────────────────┐                │
│ │         PHYSICAL RAM (916MB)             │                │
│ │         Divided into 4KB pages           │                │
│ │         MMU translates on every access   │                │
│ │         TLB caches recent translations   │                │
│ └──────────────────────────────────────────┘                │
└─────────────────────────────────────────────────────────────┘
```

**Key concepts covered:**
- Page = 4KB chunk of memory (translation unit). Huge pages (2MB, 1GB) also available.
- Page table = 4-level tree. Only allocates nodes for mapped regions. 9 bits per level, 512 entries per node.
- CR3 register = points to the current process's page table root. Updated on context switch.
- TLB = cache of recent translations. Avoids the 4-hop tree walk 99%+ of the time.
- ASLR = addresses randomized on each launch (security).
- NX bit = memory pages are writable or executable, never both (prevents code injection).
- IOMMU = page table for hardware devices (prevents DMA attacks).
- Segfault = access an unmapped address → hardware exception → process killed.

**Why this matters for AMIs:** when you snapshot a disk, you capture the filesystem, not RAM. Each new instance starts fresh with new page tables, new virtual address spaces, new ASLR randomization. The AMI provides the files; the kernel builds fresh memory structures every boot.

---

## Further Reading

* [man proc(5)](https://man7.org/linux/man-pages/man5/proc.5.html) (`/proc/self/maps`, `/proc/meminfo`, `/proc/*/status` explained)
* [Linux Kernel: Page Table Documentation](https://www.kernel.org/doc/html/latest/mm/page_tables.html)
* [Intel x86_64 Paging (OSDev Wiki)](https://wiki.osdev.org/Paging) (deep dive into the 4 level tree)
* [ASLR on Wikipedia](https://en.wikipedia.org/wiki/Address_space_layout_randomization)
* [man getconf](https://man7.org/linux/man-pages/man1/getconf.1.html) (for `getconf PAGE_SIZE`)
* [Understanding the Linux Virtual Memory Manager (Mel Gorman)](https://www.kernel.org/doc/gorman/html/understand/) (free online book, very thorough)
* [CPU Speculative Execution Attacks (Spectre/Meltdown)](https://meltdownattack.com/) (why CPU microcode patches matter)

---

**Next:** [Chapter 3: Storage Fundamentals →](03-storage-fundamentals.md)
