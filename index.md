---
layout: default
title: "Linux Memory Management, I/O, and eBPF: A Ground-Up Technical Walkthrough"
---

# Linux Memory Management, I/O, and eBPF: A Ground-Up Technical Walkthrough

*For someone who knows userspace memory and I/O well but wants to see what the kernel is actually doing underneath. Every code snippet is from a real kernel tree (Linux 7.0-rc7).*

---

## Preface: What This Book Is, and How to Read It

This is a field guide, written to take you from "I know how `mmap` and `read` work in userspace" to "I can design, build, and operate a production-quality page lifecycle tracer that observes every physical page in the system." The tracer is the through-line. Each chapter builds up a specific piece of knowledge you will need to make the tracer work correctly.

**What you will be able to do after reading.** Given a running Android phone or Linux server, you will be able to instrument the kernel using eBPF to produce a stream of events covering every physical page's full lifecycle — allocation, page cache insertion, fault into a process, dirty, writeback, I/O issue, I/O completion, migration, zRAM compression, and free. You will understand every event precisely: what triggered it, which kernel function fired it, what guarantees it makes, what it cannot tell you. You will be able to read the event stream in userspace, reconstruct per-page lifecycles, measure latencies at each stage, and export the data for offline analysis.

**The approach.** Every claim in this book is grounded in actual kernel source code. When the text says "the kernel does X," the next paragraph shows you the function that does X, with the exact file path and line number.

**Kernel version.** All source references in this book are against the Linux kernel tree checked out at:

- **Version**: `7.0-rc7` (from `Makefile`: `VERSION=7, PATCHLEVEL=0, SUBLEVEL=0, EXTRAVERSION=-rc7`)
- **Commit**: `e774d5f1bc27a85f858bce7688509e866f8e8a4e` (merge of `riscv-for-linus-v7.0-rc8`, April 2026)
- **Upstream**: `git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git`

To check out the exact tree:

```bash
git clone git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
cd linux
git checkout e774d5f1bc27a85f858bce7688509e866f8e8a4e
```

Line numbers shift between releases — a function that is at `mm/memory.c:6589` in this tree may be at line 6601 in 7.1 or line 5932 in 6.10. Function names, parameters, and overall architecture are stable across the 6.x and 7.x series. If a line number does not match on your tree, `grep` for the function name; the structure of the code is what matters, not the exact line.

The eBPF handlers here will compile, pass the verifier, and run on a real kernel. The userspace program handles filter configuration, lifecycle reconstruction, ring buffer draining, and graceful shutdown. Where something is simplified, I say so.

**Prerequisites.** You should be comfortable in C and have mental models for processes, virtual memory, file descriptors, and system calls. You do not need prior kernel experience — Part 1 starts at the hardware (DRAM, caches, MMU). You do not need prior eBPF experience — Part 17 explains the entire eBPF architecture from instruction format up.

**How to read.** Parts 1–3 cover the hardware: DRAM, the cache hierarchy, and page tables. Parts 4–6 cover the kernel data structures and tools. Parts 7–9 trace faults, COW, and writeback. Part 10 covers the VFS and filesystem layer. Part 11 covers block I/O. Parts 12–14 cover reclaim, zRAM, and page migration. Part 15 covers the scheduler and how it interacts with mm. Part 16 covers synchronization. Part 17 introduces eBPF. Parts 18–19 build and operate the tracer. Part 20 covers Perfetto integration. Part 21 covers Android specifics. Part 22 gives historical context. Parts 23–24 cover DMA-buf and the network stack — how pages move between hardware and carry packets. Part 25 dives deep into f2fs, ext4, and erofs. Appendix A has the complete tracer source.

You can read linearly, or you can jump around. If you already know hardware and mm, skip to Part 17. If you already know eBPF, skip to Part 18.

**Reproducing the code.** All paths like `mm/memory.c:6589` refer to the kernel tree specified above. The BPF programs require a *running* kernel built with `CONFIG_DEBUG_INFO_BTF=y`, `CONFIG_BPF=y`, `CONFIG_BPF_SYSCALL=y`, and `CONFIG_BPF_EVENTS=y` — all enabled by default on modern distributions and on Android (GKI builds from Android 12 onward).

Everything is licensed under the kernel's GPL-2.0 terms for the code that is quoted from kernel source, and CC-BY for the original prose.

---

## Table of Contents

1. [The Hardware Foundation: DRAM, Banks, Channels](#part-1-the-hardware-foundation-dram-banks-channels)
2. [The Cache Hierarchy: How the CPU Hides Memory Latency](#part-2-the-cache-hierarchy-how-the-cpu-hides-memory-latency)
3. [Virtual Memory and Page Tables: The Translation Machinery](#part-3-virtual-memory-and-page-tables-the-translation-machinery)
4. [Kernel Data Structures: How Memory Is Described](#part-4-kernel-data-structures-how-memory-is-described)
5. [Process Memory Lifecycle: fork, COW, and the Zero Page](#part-5-process-memory-lifecycle-fork-cow-and-the-zero-page)
6. [Observing the Page Cache: Tools of the Trade](#part-6-observing-the-page-cache-tools-of-the-trade)
7. [The Page Fault: Where Everything Starts](#part-7-the-page-fault-where-everything-starts)
8. [Anonymous Memory and Copy-on-Write](#part-8-anonymous-memory-and-copy-on-write)
9. [Dirty Pages and Writeback: Getting Data to Disk](#part-9-dirty-pages-and-writeback-getting-data-to-disk)
10. [The VFS and Filesystem Layer](#part-10-the-vfs-and-filesystem-layer)
11. [The Block I/O Layer: From Pages to Sectors](#part-11-the-block-io-layer-from-pages-to-sectors)
12. [Reclaim: How the Kernel Takes Memory Back](#part-12-reclaim-how-the-kernel-takes-memory-back)
13. [zRAM: Android's Swap Trick](#part-13-zram-androids-swap-trick)
14. [Page Migration: Moving Pages Without Anyone Noticing](#part-14-page-migration-moving-pages-without-anyone-noticing)
15. [The Linux Scheduler](#part-15-the-linux-scheduler)
16. [Synchronization: How the mm Subsystem Stays Sane](#part-16-synchronization-how-the-mm-subsystem-stays-sane)
17. [eBPF: Observing the Kernel From the Inside](#part-17-ebpf-observing-the-kernel-from-the-inside)
18. [Building a Page Lifecycle Tracer with eBPF](#part-18-building-a-page-lifecycle-tracer-with-ebpf)
19. [Operating the Tracer: Build, Run, Debug, Tune](#part-19-operating-the-tracer-build-run-debug-tune)
20. [Perfetto Integration](#part-20-perfetto-integration)
21. [Android-Specific Considerations](#part-21-android-specific-considerations)
22. [Historical Context: How We Got Here](#part-22-historical-context-how-we-got-here)
23. [DMA-buf: How Pages Move Between Hardware](#part-23-dma-buf-how-pages-move-between-hardware)
24. [Network Stack: How Pages Carry Packets](#part-24-network-stack-how-pages-carry-packets)
25. [Filesystem Deep Dive: f2fs, ext4, erofs](#part-25-filesystem-deep-dive-f2fs-ext4-erofs)
26. [Networking From the Ground Up](#part-26-networking-from-the-ground-up)
27. [Appendix A: Complete Source Reference](#appendix-a-complete-source-reference)

---

## Part 1: The Hardware Foundation — DRAM, Banks, Channels {#part-1-the-hardware-foundation-dram-banks-channels}

Before any kernel code runs, you have silicon. Understanding what RAM actually *is* at the electrical level changes how you read the rest of this book. Every decision the kernel makes — how pages are sized, why NUMA matters, why DMA needs cache flushes, why rowhammer is a thing — comes from physical constraints of DRAM. Skip this part and later chapters read like arbitrary rules. Read it and they read like inevitable engineering.

### 1.1 DRAM: What Memory Actually Is

Every bit in your RAM is a tiny capacitor. Charged = 1, discharged = 0. These capacitors are organized into a grid of **rows** and **columns** inside each DRAM chip. The key physical facts:

**Reading is destructive.** When the memory controller activates a row, the charge flows out of every capacitor in that row into **sense amplifiers** (the "row buffer"). The sense amps detect 1 or 0, then write the charge back. This activate-sense-restore cycle is the fundamental operation of DRAM and takes ~30–50ns.

**Capacitors leak.** Even without reading, charge drains away in about 64ms. The memory controller must periodically **refresh** every row — activate it and restore the charges — thousands of times per second. This steals ~5% of bandwidth. This is why it's called *Dynamic* RAM: the storage is inherently unstable.

**SRAM (used in caches) is different.** Each bit uses a flip-flop circuit (6 transistors vs DRAM's 1 transistor + 1 capacitor). It holds state indefinitely while powered, reads non-destructively, and is much faster. But it costs ~6× the silicon area per bit, which is why caches are small (kilobytes to megabytes) and DRAM is large (gigabytes).

### 1.2 DRAM Organization: Banks, Channels, Rows, Columns

A single DRAM chip contains multiple **banks** (typically 16). Each bank is an independent array with its own row buffer:

```
One DRAM chip:
┌─────────────────────────────────────┐
│  Bank 0: [row buffer] [cell array]  │
│  Bank 1: [row buffer] [cell array]  │
│  ...                                │
│  Bank 15:[row buffer] [cell array]  │
│                                     │
│  Shared: output pins to the bus     │
└─────────────────────────────────────┘
```

**Banks share the chip's output pins** — only one bank can send data on the bus at a time. But their internal work overlaps: while bank 0 sends data, bank 3 can be activating a row, and bank 7 can be restoring charges. The memory controller pipelines these operations to keep the bus busy.

**Channels** are completely independent paths between the CPU's memory controller and separate sets of DRAM chips. Different wires, different chips, sharing nothing:

```
CPU
 └── Memory Controller
      ├── Channel 0 ── [bus] ── DRAM chips
      ├── Channel 1 ── [bus] ── DRAM chips
      └── Channel 2 ── [bus] ── DRAM chips
```

The parallelism stacks: channels are fully independent (different wires), banks overlap internal work (shared bus per channel). A system with 2 channels × 16 banks = 32 independent row buffers. This is the raw concurrency the kernel's page allocator and I/O scheduler try to exploit.

### 1.3 Reading Data: The Burst

When the memory controller needs data, it issues commands to the DRAM chip:

**Row already open in the row buffer (row hit):**

```
Column command → burst → done
~15ns
```

**Different row needed (row miss):**

```
Precharge old row  (~10ns)  — close current row
Activate new row   (~30–50ns) — destructive read into sense amps
Column command     (~15ns)  — select columns, start burst
Total: ~55–75ns
```

The burst is a series of consecutive transfers. On DDR4 with a 64-bit (8-byte) bus and burst length 8:

```
8 bytes × 8 transfers = 64 bytes per burst
```

This 64-byte burst size is not a coincidence — **it matches the CPU cache line size**. One cache miss = one burst = one cache line fill. The entire hardware/software stack is designed around this alignment, and the kernel's `struct page` granularity (4KB = 64 lines × 64 bytes) is a direct consequence.

Within a burst, the transfers are sequential (an auto-incrementing column counter inside the DRAM). Between separate column commands to the same open row, the row buffer is essentially random access — that is why sequential access within a 4KB page is so much faster than scattered access.

### 1.4 DRAM Timing Parameters

```
tRP  (Row Precharge):        ~10ns    Close the currently open row
tRCD (RAS to CAS Delay):     ~13ns    Activate a row, wait for sense amps
tCAS (CAS Latency / CL):     ~13ns    Column command to data on bus
tRAS (Row Active Time):      ~30ns    Minimum time row must stay open
tRRD (Row to Row Delay):     ~5ns     Min gap between activations in different banks
tFAW (Four Activate Window): ~25ns    No more than 4 activations in this window (power)
tREFI (Refresh Interval):    ~7.8µs   How often each row must be refreshed
```

A row miss costs `tRP + tRCD + tCAS ≈ 36ns` minimum. A row hit costs just `tCAS ≈ 13ns`. This 3× difference is why the memory controller reorders requests to maximize row hits — a reorder that changes "row A, row B, row A, row B" into "row A, row A, row B, row B" saves tens of nanoseconds per access.

Spec sheet notation: **DDR4-3200 CL16-18-18-36** means clock = 1600MHz (doubled to 3200 MT/s), tCAS=16 cycles, tRCD=18, tRP=18, tRAS=36. Multiply by the cycle time (0.625ns for DDR4-3200) to get nanoseconds.

### 1.5 Channel Interleaving

The mapping of physical addresses to channels is hardwired by the memory controller. Every address is permanently assigned to a channel based on its address bits:

```
With 2-channel interleaving on bit 5:

Physical address 0x000 (bit 5=0) → Channel 0, Chip A
Physical address 0x020 (bit 5=1) → Channel 1, Chip B
Physical address 0x040 (bit 5=0) → Channel 0, Chip A
```

A 64-byte cache line can be split across two channels — each channel delivers 32 bytes simultaneously, filling the line in one burst period. The memory controller assembles the pieces. This is invisible to software but doubles effective bandwidth for cache line fills.

Real controllers use **XOR hashing** rather than simple bit selection to distribute accesses evenly across channels regardless of stride pattern:

```
channel = bit6 XOR bit13 XOR bit20
```

This prevents pathological patterns where a common stride (e.g., 4KB page-aligned accesses) hits only one channel.

The mapping applies to **everyone** — CPU cores, DMA controllers, GPUs all send physical addresses to the same memory controller with the same interleaving rules. A DMA transfer gets the same channel parallelism as CPU accesses. This matters for the tracer later: I/O throughput peaks when bios are large enough to span multiple channels.

### 1.6 Rowhammer

Rapidly activating a DRAM row (millions of times within a 64ms refresh interval) causes electrical disturbance that can flip bits in physically adjacent rows. This is a hardware physics problem, not a software bug — the capacitors in an adjacent row see induced currents from the neighbor's wordline toggling.

Security implications: an attacker who knows the physical address layout can hammer rows adjacent to sensitive data (page table entries, credentials). A single bit flip in a PTE can change a permission bit from read-only to read-write, granting unauthorized access.

Mitigations include **TRR (Target Row Refresh)** in DRAM hardware, hiding physical addresses from userspace (`/proc/pid/pagemap` requires `CAP_SYS_ADMIN` since kernel 4.0), ECC memory (detects and often corrects single-bit flips), and increased refresh rates in DDR5. The Linux kernel exposes pagemap data carefully precisely because knowing the PFN of a target page is the first step in exploiting rowhammer.

### 1.7 From Hardware to the Kernel's View

Now for the kernel's perspective. The memory controller presents all this machinery as a flat array of bytes, addressed from 0 to however much RAM you have. The kernel slices this into fixed 4KB chunks called **page frames**. On a phone with 8GB of RAM, that is roughly 2 million page frames.

Each page frame gets an index number — its **PFN** (Page Frame Number). PFN 0 is the first 4KB, PFN 1 is the next, and so on. Conversion is pure bit shifting:

```c
pfn = physical_address >> 12;      // divide by 4096
physical_address = pfn << 12;      // multiply by 4096
```

8GB of RAM = 2,097,152 page frames. The kernel maintains a giant array of small descriptor structures, one per page frame, so it can track what each page frame is being used for. On modern kernels this array is `struct page mem_map[]`, allocated at boot with one 64-byte entry per frame:

```c
struct page mem_map[2097152];  // ~2M entries for 8GB
pfn_to_page(pfn) = &mem_map[pfn];  // just array indexing
page_to_pfn(page) = page - mem_map;
```

Total cost: 2M × 64 bytes = 128MB (~1.6% of 8GB). Every byte added to `struct page` costs 2MB of RAM at boot. This is why `struct page` uses unions aggressively (Part 4) — byte efficiency translates directly into boot-time RAM overhead.

The hardware also has an **MMU** (Memory Management Unit) that sits between the CPU and the memory controller. When your program accesses address `0x7f000000`, the CPU does not send that address to the memory controller. Instead, the MMU translates it through a set of **page tables** — a tree structure stored in physical memory — to find the corresponding physical page frame. If no translation exists (the page table entry is empty), the MMU raises a **page fault exception** to the CPU, and the kernel's fault handler runs. This is the most important event in the mm subsystem, and we return to it in Part 7.

Next, we look at the cache hierarchy that sits between the CPU and DRAM, because everything the kernel does about memory performance (from `____cacheline_aligned` to DMA sync calls) is shaped by it.

---

## Part 2: The Cache Hierarchy — How the CPU Hides Memory Latency {#part-2-the-cache-hierarchy-how-the-cpu-hides-memory-latency}

DRAM is slow. A CPU core running at 3 GHz completes a cycle in 0.33ns. A DRAM row miss costs ~55–75ns. If the CPU had to wait for every memory access, it would spend ~99% of its time stalled. Caches are how the hardware pretends memory is fast.

Understanding the cache hierarchy is not optional for kernel work. Every kernel decision about memory layout (`____cacheline_aligned`, per-CPU data, struct field ordering), every DMA operation, every atomic primitive is shaped by it.

### 2.1 Overview

```
Registers       ~1KB        ~0.3ns    same cycle
TLB             ~64–1024    ~1ns      translation cache
L1 cache        32–64KB     ~2ns      per-core, split I/D
L2 cache        256KB–1MB   ~7ns      per-core
L3 cache        4–32MB      ~20ns     shared across cores
RAM             4–64GB      ~100ns    main memory
SSD             128GB+      ~50µs     storage
Disk            1TB+        ~5ms      storage
```

Each level is ~10–100× slower and ~10–100× larger than the one above. The design exploits **locality**: programs tend to reuse the same data (temporal locality) and access nearby data (spatial locality). Without locality, caches would not help and the whole pyramid would be pointless.

### 2.2 Cache Line: The Unit of Transfer

The cache never moves individual bytes. The unit of transfer at every level is the **cache line** — 64 bytes on all modern x86 and ARM64 CPUs.

When you read 1 byte and it misses L1:

```
L1 asks L2 → L2 asks L3 → L3 asks DRAM
DRAM delivers 64 bytes (one burst — see Part 1)
Line installed in L3, L2, L1
CPU gets its 1 byte
The other 63 bytes are free cache hits if accessed soon
```

Why 64 bytes: smaller lines (16B) waste bandwidth on overhead and miss spatial locality. Larger lines (256B) fetch too much useless data and reduce total cache capacity (fewer lines fit). 64 bytes is the empirical sweet spot — one loop iteration over 4-byte ints gets 16 values per line.

This size is baked into the kernel. You see it in macros like `L1_CACHE_BYTES` (defined per-arch, equal to 64 on all supported modern arches) and in attributes like `____cacheline_aligned` that force a struct member to start on a 64-byte boundary.

### 2.3 Sets, Ways, and Tags

Cache organization for a 32KB, 8-way set-associative L1 cache:

```
Total lines:  32768 / 64 = 512
Sets:         512 / 8 = 64 sets
```

Think of it as a table with 64 rows (sets) and 8 columns (ways):

```
          Way 0     Way 1     Way 2    ...    Way 7
        ┌─────────┬─────────┬─────────┬───┬─────────┐
Set  0  │  line   │  line   │  line   │   │  line   │
Set  1  │  line   │  line   │  line   │   │  line   │
  ...   │   ...   │   ...   │   ...   │   │   ...   │
Set 63  │  line   │  line   │  line   │   │  line   │
        └─────────┴─────────┴─────────┴───┴─────────┘
```

A physical address is split into three fields:

```
 47                    12  11       6  5       0
┌────────────────────────┬──────────┬──────────┐
│          TAG           │   SET    │  OFFSET  │
│       (42 bits)        │ (6 bits) │ (6 bits) │
└────────────────────────┴──────────┴──────────┘
```

**Offset (bits 5–0):** which byte within the 64-byte line.
**Set index (bits 11–6):** which of the 64 sets — deterministic, not a choice.
**Tag (bits 47–12):** stored alongside the data to identify which address is cached.

On lookup: compute the set from the address, compare the tag against all 8 ways in parallel. If a tag matches and the valid bit is set, that's a hit. If no match, it's a miss.

Each way stores: valid bit + tag + 64 bytes of data + dirty bit. The tag comparison uses dedicated parallel hardware comparators — all 8 ways checked simultaneously in ~1 cycle.

The "32KB" is just the data. The actual silicon includes tag storage, valid/dirty bits, and LRU tracking — roughly 35–36KB total.

### 2.4 VIPT: The Parallel Lookup Trick

The cache set index (bits 11–6) falls entirely within the page offset (bits 11–0). Since the page offset is identical in virtual and physical addresses (the MMU doesn't translate it), the cache can start the set lookup using virtual address bits while the TLB simultaneously translates the upper bits.

```
CPU generates virtual address
  ├── bits 11–0 → cache: compute set, read all 8 tags  (immediate)
  └── upper bits → TLB: translate to physical PFN       (parallel)

  TLB finishes → compare physical tag against loaded tags
  Both done in ~same time → no added latency
```

This is **VIPT** — Virtually Indexed, Physically Tagged. It only works if the set index bits stay within the page offset. For 4KB pages (12-bit offset) with 64B lines: max set index is bits 11–6 = 6 bits = 64 sets. At 8-way, that's 64 × 8 × 64 = 32KB. This is why L1 caches are typically 32KB with 4KB pages.

Doubling to 64KB would need 128 sets = 7-bit index using bits 12–6. Bit 12 is above the page offset and differs between virtual and physical — the parallelism breaks. Solutions: keep L1 small, use 16KB pages (Apple M-series, giving 14 bits of offset and room for 128KB L1), speculate on both possible sets, or page coloring.

### 2.5 Page Coloring

When the cache set index extends above the page offset, two virtual addresses mapping the same physical page can disagree on the extended bit(s), placing the same data in two different cache sets — **aliasing**. This causes coherency bugs: a write to one virtual alias is not visible through the other.

Page coloring constrains the page allocator: a physical page's "color" (the problematic bit) must match the virtual address's color. This prevents aliasing but reduces allocator flexibility — you can only use pages of the right color, causing fragmentation. Linux on MIPS and some older ARM configurations implements page coloring; most modern designs avoid the problem by keeping L1 within the safe VIPT size.

### 2.6 Write Behavior

**Write-back (what all modern CPUs use):** writes go to L1 only. The line is marked dirty. L2, L3, and RAM have stale data — fine as long as L1 holds the line. Dirty data trickles down only on eviction.

**Write-allocate:** on a write miss, fetch the full 64-byte line first, then modify. Necessary because you're writing maybe 4 bytes but the cache needs all 64 to be valid.

```
Before write:  L1="hello"  L2="hello"  RAM="hello"  (all agree)
After write:   L1="world"  L2="hello"  RAM="hello"  (only L1 current)
After evict:   L1=[gone]   L2="world"  RAM="hello"  (L2 updated)
After L2 evict:             L2=[gone]  RAM="world"  (finally)
```

The dirty bit in each cache way tracks whether the line needs to be written back on eviction. An eviction of a clean line is free (just discard); an eviction of a dirty line costs a write back to the next level.

### 2.7 Multi-Core Coherency: MESI

Each core has its own L1/L2. Two cores can cache the same address. Writes by one core must be visible to the other. The hardware **MESI protocol** handles this automatically:

```
M (Modified):   only copy, it's been written, RAM is stale — we own it
E (Exclusive):  only copy, matches RAM — can write without telling anyone
S (Shared):     multiple copies exist, all match RAM — must notify before writing
I (Invalid):    not usable — treat as absent
```

When Core 1 wants to write to a line Core 0 has in S state:

1. Core 1 broadcasts "I want exclusive access" (Read-For-Ownership).
2. Core 0 sees the broadcast, marks its copy Invalid.
3. Core 1 transitions to Modified, does the write.

This costs ~20–50ns for the cross-core communication — far more than a local L1 hit (~2ns). This is why contended atomic operations are slow: every atomic read-modify-write transfers a line from M or S → E → M with coherency traffic.

### 2.8 False Sharing

Two independent variables in the same 64-byte cache line cause cores to ping-pong the line between them:

```c
struct bad {
    int core0_counter;  // offset 0, core 0 writes
    int core1_counter;  // offset 4, core 1 writes
};
// Both in same cache line — every write invalidates the other core's copy
```

Fix: pad to separate cache lines:

```c
struct good {
    int core0_counter;
    char pad[60];
    int core1_counter;  // now in a different cache line
};
```

The kernel uses `____cacheline_aligned` annotations extensively on per-CPU data, spinlocks, and frequently accessed structures. In `struct mm_struct` at `include/linux/mm_types.h:1123`, for example, `mm_count` sits alone in its own cache line via `____cacheline_aligned_in_smp`:

```c
struct {
    atomic_t mm_count;
} ____cacheline_aligned_in_smp;
```

This is not cosmetic. `mm_count` is incremented and decremented by reference tracking across all CPUs; sharing its line with anything else would cause false sharing.

### 2.9 DMA and Cache Coherency

DMA-capable devices (NIC, disk controller, GPU) access RAM through the memory controller, **bypassing CPU caches**. This creates coherency problems:

**CPU writes, device reads (e.g., sending a network packet):**

```
CPU writes data → sits in L1 cache, dirty
Device DMA reads from RAM → gets stale data
Result: corrupted packet
```

**Device writes, CPU reads (e.g., receiving a packet):**

```
Device DMA writes to RAM → new data in DRAM
CPU reads → cache hit with old data
Result: CPU processes garbage
```

The kernel must explicitly manage this with the DMA API:

```c
// Before device reads: flush dirty cache lines to RAM
dma_sync_single_for_device(dev, addr, size, DMA_TO_DEVICE);

// Before CPU reads DMA'd data: invalidate cache lines
dma_sync_single_for_cpu(dev, addr, size, DMA_FROM_DEVICE);
```

**Cache operations:**

- **Clean:** write dirty lines to RAM, keep in cache (cache and RAM agree).
- **Invalidate:** mark lines Invalid, discard (next CPU access fetches from RAM).
- **Flush:** clean + invalidate.

**Critical constraint:** cache invalidation operates on whole 64-byte lines. DMA buffers must be cache-line aligned, or invalidation can discard adjacent kernel data. The DMA allocator (`dma_alloc_coherent`) enforces this by rounding sizes up and alignments to the cache line.

**Coherent DMA:** for frequently shared memory (ring buffers between CPU and device), the kernel maps pages as **uncacheable** via PTE flags. Every access goes directly to RAM (~100ns vs ~2ns), but no sync calls are needed. This is the right tradeoff for small control regions; for bulk data (network packets, disk blocks), explicit sync on cacheable memory wins.

Architectures differ significantly: x86 has cache-coherent DMA by default (the fabric snoops caches), so many `dma_sync_*` calls are no-ops. ARM64 and most embedded architectures are not coherent — the sync calls translate to real cache maintenance instructions. The kernel's DMA API hides this difference so drivers work identically on both. Part 23 covers DMA-buf — how pages are shared between multiple hardware blocks (camera, GPU, display) with proper cache synchronization and fence-based ordering.

---

## Part 3: Virtual Memory and Page Tables — The Translation Machinery {#part-3-virtual-memory-and-page-tables-the-translation-machinery}

We've seen the hardware. Now we look at how the kernel uses it to give each process the illusion of its own isolated, contiguous address space.

### 3.1 Pages and Page Frames

The kernel slices physical RAM into fixed 4KB chunks called **page frames**. Each gets a sequential index — its **PFN** (Page Frame Number):

```
PFN 0  →  physical bytes 0 to 4095
PFN 1  →  physical bytes 4096 to 8191
...
```

We met this in Part 1. 8GB of RAM = 2,097,152 page frames (~2 million). Conversion is pure bit shifting:

```c
pfn = physical_address >> 12;
physical_address = pfn << 12;
```

### 3.2 Virtual Addresses and the MMU

Programs use **virtual addresses**. The MMU (Memory Management Unit) hardware translates them to **physical addresses** on every memory access. This provides three benefits the kernel depends on:

- **Isolation:** two processes using the same virtual address map to different physical pages.
- **Contiguity illusion:** scattered physical pages appear contiguous to the process.
- **Lazy allocation:** memory can be promised (`mmap` creates a VMA) without backing it with physical pages until first access. This is how a process can "allocate" 1GB in milliseconds — only the VMA structure is created; the pages are filled in lazily on fault.

### 3.3 The 4-Level Page Table

A virtual address (48 bits used on ARM64/x86-64) is split into five fields:

```
 47        39 38        30 29        21 20        12 11          0
┌──────────┬──────────┬──────────┬──────────┬──────────────────┐
│  PGD idx │  PUD idx │  PMD idx │  PTE idx │   page offset    │
│  9 bits  │  9 bits  │  9 bits  │  9 bits  │    12 bits       │
└──────────┴──────────┴──────────┴──────────┴──────────────────┘
```

Each 9-bit index selects one of 512 entries in a table. Each table is one 4KB page (512 × 8 bytes). The 12-bit page offset is the same in virtual and physical addresses — it's not translated. The kernel walks this tree like so:

```c
// This is the conceptual page table walk. The kernel does this in
// __handle_mm_fault() at mm/memory.c:6355
pgd = pgd_offset(mm, address);       // top level
p4d = p4d_alloc(mm, pgd, address);   // (folded into PGD on most ARM64)
pud = pud_alloc(mm, p4d, address);   // level 3
pmd = pmd_alloc(mm, pud, address);   // level 2
pte = pte_offset_map(pmd, address);  // level 1 — the actual mapping
```

If any level is missing, the kernel allocates a new page for it. This is demand-driven — page tables grow only as processes touch new memory regions. A process that maps 1TB of address space but only touches 4KB will have exactly the page table entries needed for that one page.

The 36 bits of page number (48 − 12) give 2^36 ≈ 68 billion possible virtual pages. A flat page table would need one 8-byte entry per possible page — 512GB per process. Impossible. The multi-level tree is sparse: if a process hasn't touched a region, the pointer at PGD or PUD level is NULL and the entire subtree below doesn't exist. One NULL at PGD eliminates 2^27 potential entries.

Each level covers a different granularity:

```
PGD entry covers: 512GB
PUD entry covers:   1GB
PMD entry covers:   2MB
PTE entry covers:   4KB
```

**Why 4 levels?** Because 4 levels of 9-bit indices (4 × 9 = 36 bits) plus a 12-bit page offset gives 48 bits of virtual address space — 256TB. That is enough for now. Some newer CPUs support 5 levels for 57-bit addressing (128PB), but most ARM64 devices use 4.

### 3.4 What's in a PTE

A PTE is 8 bytes (64 bits). On ARM64:

```
Bit 0:       Valid — is this entry present?
Bits 47–12:  Physical PFN (36 bits → 256TB addressable)
Bit 10:      Access Flag (AF) — hardware sets on any access
Bits 8–7:    Access Permission (AP) — read/write, kernel/user
Bit 51:      Dirty Bit Modifier (DBM) — hardware sets on write (ARMv8.1+)
Bit 54:      Execute Never (XN) — blocks code execution
Bits 58–55:  Software bits — kernel's private flags, hardware ignores
```

On older ARM64 without DBM (ARMv8.0), the kernel emulates dirty tracking: writable pages start with AP set to read-only, and the first write triggers a permission fault. The fault handler sets the dirty bit in software and makes the PTE writable. This costs one fault per first-write per page but is transparent to userspace.

**When bit 0 is 0 (invalid):** the MMU ignores the entry. The kernel repurposes the other 63 bits to encode swap locations, migration state, or NUMA hints. One set of bits serves completely different purposes depending on whether hardware or software is reading them. This bit-repurposing is how swap entries and migration entries fit in the same PTE slot — you saw a consequence of this in `handle_pte_fault` (Part 7 later): `if (!pte_present(vmf->orig_pte)) return do_swap_page(vmf);`. The PTE is "not present" but its bits encode where to find the page in swap.

**Hardware modifications:** the MMU sets the Access Flag on any access and the Dirty bit on writes. These are reported *by the hardware* and *read by the kernel* — the kernel uses them to drive LRU decisions (hot pages stay, cold pages get reclaimed) and to know which pages need writeback.

### 3.5 The TLB

The TLB (Translation Lookaside Buffer) caches recent page table translations. TLB hit = ~1 cycle. TLB miss = full 4-level page table walk through RAM = 4 memory accesses = ~200 cycles.

The TLB is small (typically 64–1024 entries) and highly associative. Modern CPUs have separate L1 instruction TLB, L1 data TLB, and a shared L2 TLB. Each entry maps a virtual page to a physical page plus permission bits — essentially a cached copy of a PTE.

When the kernel changes a PTE (unmap, permission change, migration), the old translation may be cached in the TLB. The kernel must **flush** it. On multi-core systems, this requires TLB shootdown IPIs (inter-processor interrupts) to all cores that might have the stale entry — expensive (~microseconds per IPI, since each CPU has to stop what it's doing to service the IPI).

The mm subsystem batches TLB flushes where possible. For example, during `try_to_unmap` (reclaim), the code sets `TTU_BATCH_FLUSH` to defer individual flushes and do one batch flush at the end. You'll see the performance consequences of TLB shootdowns in the COW path (Part 8), where `ptep_clear_flush` must happen before the new PTE is installed to prevent two CPUs from seeing different mappings.

### 3.6 Who Does the Walk

**Hardware (MMU):** on every memory access, billions of times per second. The process never knows. The MMU reads PTEs directly from the page table pages in physical memory, with the caveat that the tables must be laid out where the MMU expects them (pointed to by `CR3` on x86 or `TTBR0/TTBR1` on ARM64).

**Kernel (software):** when modifying page tables — setting up mappings on fault, clearing them on munmap, changing permissions on mprotect. The kernel writes to the tables; the MMU reads them. The CPU has memory barriers and TLB flush instructions to coordinate the two (kernel write → TLB flush → MMU sees new mapping).

### 3.7 ARM64 vs x86-64: Split Address Spaces

**ARM64:** two page table base registers:

```
TTBR0_EL1: per-process userspace page table (changes on context switch)
TTBR1_EL1: kernel page table (set once at boot, never changes, shared by all)
```

Addresses starting with `0x0000...` use TTBR0; addresses starting with `0xFFFF...` use TTBR1.

**x86-64:** one register (CR3), so kernel and user share one tree. The upper PGD entries in every process point to the same shared kernel PUD pages. Kernel mapping changes at PMD/PTE level are automatically visible everywhere. PGD-level changes propagate lazily via vmalloc fault fixup.

This is why interrupt handlers work without switching page tables — kernel addresses are mapped identically in every process's tree. When an interrupt fires, the CPU is already running some userspace process; its page table tree has the kernel mappings in the top half, so the interrupt handler can access kernel data, kernel code, and kernel stacks without changing any register.

### 3.8 Page Table Size in Practice

A typical process with 300 VMAs, 500MB virtual address space, 50,000 pages mapped:

```
PGD:       1 page        =    4KB
PUD:       1–2 pages     =    8KB
PMD:       1–4 pages     =   16KB
PTE pages: ~100–150      = ~600KB  (dominates)
Total: roughly 600–700KB
```

The bottom level (PTE pages) dominates because it's where actual mappings live. Each PTE page maps 2MB of virtual address space (512 × 4KB), so 500MB of mapped memory needs 250 PTE pages minimum, clustered by virtual address locality.

On Android, where a process has many shared libraries and mmap'd files scattered across the address space, page table memory can reach several megabytes. This is part of why `posix_spawn` (Part 5) is valuable — it avoids copying all this page table state.

---

## Part 4: Kernel Data Structures — How Memory Is Described {#part-4-kernel-data-structures-how-memory-is-described}

The kernel constantly needs to answer three questions. Each question leads to a data structure.

```
Question 1: "Is page 130 of libc.so already in RAM?"
  → page cache (xarray: address_space->i_pages)

Question 2: "Which file does PFN 5000 belong to?"
  → folio->mapping + folio->index

Question 3: "Which PTEs point to PFN 5000?"
  → reverse mapping (i_mmap for files, anon_vma for anonymous)
```

These three questions drive page faults, reclaim, writeback, migration, and COW. The structures that answer them form the backbone of the mm subsystem. The tracer we build in Part 18 reaches into almost every one of these structures, and getting them wrong produces subtly broken tracers.

### 4.1 task_struct → mm_struct

Every thread in the kernel is represented by a `task_struct`. Every process (which may have many threads) has exactly one `mm_struct` that describes its entire virtual address space. All threads in a process share the same `mm_struct`.

```c
// include/linux/sched.h:958
struct task_struct {
    // ... hundreds of other fields ...
    struct mm_struct        *mm;         // this process's address space
    struct mm_struct        *active_mm;  // used by kernel threads
    // ...
};
```

When the kernel needs to do anything with a process's memory, it starts here:

```c
struct mm_struct *mm = current->mm;  // "current" is always the running task
```

Kernel threads (like `kswapd`, the reclaim daemon) have `mm = NULL` because they do not have a userspace address space. They borrow the previous task's page tables via `active_mm` to avoid a costly TLB flush on context switch. This is a performance optimization — switching page tables means flushing the TLB (translation lookaside buffer, the hardware cache of recent page table lookups), which is expensive.

The `mm_struct` holds everything about the address space:

```c
// include/linux/mm_types.h:1123
struct mm_struct {
    atomic_t mm_count;              // line 1137 — reference count (kernel + user)
    struct maple_tree mm_mt;        // line 1140 — all VMAs, indexed by address
    pgd_t *pgd;                     // line 1150 — root of the page table tree
    atomic_t mm_users;              // line 1171 — how many tasks share this mm
    spinlock_t page_table_lock;     // line 1181 — protects page table modifications
    struct rw_semaphore mmap_lock;  // line 1196 — protects VMAs (the big lock)
    // ...
};
```

The `mm_mt` field is a **maple tree** — a B-tree variant optimized for ranges. It replaced the old red-black tree (`mm_rb`) and linked list (`mmap`) in v6.1. The maple tree is more cache-friendly because it packs multiple entries into each node (up to 16 VMA pointers per node), reducing the number of pointer chases during VMA lookup. On a process with hundreds of VMAs (common on Android — shared libraries, mmap'd files, anonymous regions), this matters.

Unlike the old rbtree, where each VMA had an embedded `rb_node` field used as the tree node, the maple tree manages its own internal nodes. VMAs do not contain any maple-tree-specific field — the tree stores pointers to VMAs, and the VMA struct is cleaner for it.

`mm_mt` and `pgd` are two parallel roots describing the same address space:

```
mm_mt → maple tree → VMAs     "what SHOULD exist" (intent, permissions, backing)
mm->pgd → page table → PTEs   "what DOES exist"   (actual physical mappings)
```

A VMA can exist with no PTEs (just `mmap`'d, never touched). A PTE cannot exist without a VMA — the fault handler checks `find_vma(mm, address)` first, and if no VMA covers the faulting address, the result is SIGSEGV. Every page fault starts at `mm_mt` to find the VMA, then walks `mm->pgd` to find or create the PTE.

### 4.2 vm_area_struct (VMA)

A VMA describes one contiguous region of virtual address space with uniform properties — same permissions, same backing. When you call `mmap()`, the kernel creates a VMA. When you call `munmap()`, it removes one (or splits one). When you call `mprotect()` to change permissions on part of a mapping, the kernel splits a VMA into pieces.

```c
// include/linux/mm_types.h:913
struct vm_area_struct {
    unsigned long vm_start;              // line 919 — first byte of this region
    unsigned long vm_end;                // line 920 — first byte AFTER this region
    struct mm_struct *vm_mm;             // line 929 — back pointer to the mm
    pgprot_t vm_page_prot;               // line 930 — hardware protection bits
    vm_flags_t vm_flags;                 // line 939 — VM_READ, VM_WRITE, VM_SHARED...
    struct {
        struct rb_node rb;               // node in address_space->i_mmap
        unsigned long rb_subtree_last;   // interval tree support
    } shared;                            // linkage into file's reverse mapping tree
    struct anon_vma *anon_vma;           // line 968 — reverse mapping for anon pages
    const struct vm_operations_struct *vm_ops; // line 971 — fault handler, etc.
    unsigned long vm_pgoff;              // line 974 — file offset in pages
    struct file *vm_file;                // line 976 — the backing file, or NULL
};
```

The `shared.rb` field links this VMA into the file's `address_space->i_mmap` interval tree (see section 4.11). When you `mmap` a file, the kernel inserts your VMA's `shared.rb` into the file's `i_mmap` tree. When you `munmap`, it removes it. This is how `try_to_unmap` finds all processes mapping a given file page.

**The vm_file discriminator:**

- `vm_file != NULL` → file-backed mapping (code, shared libraries, mmap'd files, tmpfs).
- `vm_file == NULL` → anonymous memory (heap, stack, `MAP_ANONYMOUS`).

This one field tells the entire fault path which way to go.

**vm_flags carry both current and possible permissions:**

```
VM_READ, VM_WRITE, VM_EXEC           — current permissions
VM_MAYREAD, VM_MAYWRITE, VM_MAYEXEC  — mprotect() can enable these
VM_SHARED                             — writes go back to file (MAP_SHARED)
VM_GROWSDOWN                          — stack, grows downward on fault below vm_start
VM_LOCKED                             — mlocked, cannot be swapped
VM_HUGETLB                            — backed by huge pages
VM_DONTCOPY                           — not duplicated on fork()
```

The distinction between VM_WRITE and VM_MAYWRITE is subtle and important. A private file mapping opened read-only starts with `VM_READ | VM_MAYWRITE` — writable by mprotect but not right now. `mprotect()` checks VM_MAYWRITE before allowing PROT_WRITE. A shared mapping of a read-only file has VM_MAYWRITE unset — mprotect to writable is denied because the write would need to go back to a file the process can't write.

VMAs are stored in `mm_mt` (the maple tree described in 4.1) for O(log n) lookup by address. `find_vma(mm, address)` searches this tree — called on every page fault. A typical Android app has 200–500 VMAs.

**mmap_lock discipline:**

- Page faults take `mmap_lock` for read (shared — concurrent faults on different VMAs proceed in parallel).
- `mmap`/`munmap`/`mprotect` take it for write (exclusive — blocks all faults during the modification).

Newer kernels add **per-VMA locks** for better scalability: a fault only needs to lock the specific VMA it's touching, so faults on different VMAs never serialize. We come back to this in Part 16.

**VMA manipulation primitives:**

- `mmap()` → creates a VMA.
- `munmap()` → removes a VMA (or the piece of one that's being unmapped), and clears the PTEs in that range.
- `mprotect()` → may split a VMA into pieces with different permissions. Adjacent VMAs with identical properties after the operation may be merged.
- `mremap()` → moves a VMA to a different virtual address range.

### 4.3 vm_operations_struct

Every file-backed VMA has a `vm_ops` pointer to a table of function callbacks. The filesystem provides these; mm code calls into them via the pointer:

```c
struct vm_operations_struct {
    void (*open)(struct vm_area_struct *);         // VMA duplicated (fork) or split
    void (*close)(struct vm_area_struct *);        // VMA removed
    vm_fault_t (*fault)(struct vm_fault *);        // page fault in this VMA
    vm_fault_t (*page_mkwrite)(struct vm_fault *); // shared page becoming writable
    vm_fault_t (*huge_fault)(struct vm_fault *, unsigned int order);
};
```

**`fault`**: called when the PTE is missing. For file-backed VMAs, this is typically `filemap_fault` (checks page cache, reads from disk on miss). For anonymous VMAs, `vm_ops` is `NULL` — the kernel uses a hardcoded path (`do_anonymous_page`). We trace both in Part 7.

**`page_mkwrite`**: called when a `MAP_SHARED` page transitions from read-only to writable. This is the filesystem's chance to reserve disk space, set up journaling, take internal locks — *before* the process can dirty the page freely. Without this, a process could fill up an ext4 filesystem simply by dirtying already-allocated pages, with no opportunity for the FS to signal ENOSPC.

**`open`/`close`**: called when VMAs are duplicated (fork), split (mprotect), or destroyed (munmap/exit). Used for reference counting on the backing object. A file-backed VMA's `open` increments the file's reference count; `close` decrements it.

### 4.4 struct page

One per physical page frame. Allocated at boot in a flat array. 64 bytes each (one cache line — not an accident; see Part 2). Never allocated or freed by mm code — repurposed as page frames change use.

```c
struct page mem_map[2097152];  // ~2M entries for 8GB of RAM
pfn_to_page(pfn) = &mem_map[pfn];   // just array indexing
page_to_pfn(page) = page - mem_map;
```

Total cost at boot: 2M × 64 bytes = 128MB (~1.6% of 8GB). Every byte added to `struct page` costs 2MB of RAM system-wide. This is why the union trick below matters so much.

**The union trick.** A page frame can be page cache, anonymous memory, slab cache, network buffer, ZONE_DEVICE, etc. Different uses need different metadata. A C union stores different interpretations in the same bytes:

```c
// include/linux/mm_types.h:79
struct page {
    memdesc_flags_t flags;                 // always present — tells you which union
    union {
        struct {                           // page cache or anonymous memory
            struct list_head lru;          // position on LRU list (for reclaim)
            struct address_space *mapping; // who owns this page
            pgoff_t __folio_index;         // offset within that owner
            unsigned long private;         // filesystem-specific data or swap entry
        };
        struct {                           // network stack page_pool
            unsigned long pp_magic;
            struct page_pool *pp;
            unsigned long _pp_mapping_pad;
            unsigned long dma_addr;
            atomic_long_t pp_ref_count;
        };
        struct {                           // tail page of a compound page
            unsigned long compound_head;   // bit 0 set; points to head page
        };
        struct {                           // ZONE_DEVICE pages (persistent memory)
            void *_unused_pgmap_compound_head;
            void *zone_device_data;
        };
        struct rcu_head rcu_head;          // for freeing via RCU
    };
    union {
        unsigned int page_type;            // for "typed" folios (slab, buddy, etc.)
        atomic_t _mapcount;                // how many PTEs point to this page
    };
    atomic_t _refcount;                    // total references from all sources
#ifdef CONFIG_MEMCG
    unsigned long memcg_data;              // memory cgroup accounting
#endif
};
```

The `flags` field tells you which union member is valid. Helper macros like `PageSlab(page)`, `PageBuddy(page)`, `PageLRU(page)` test specific flag bits to confirm which interpretation is safe. This density is deliberate — with 2 million pages, every byte costs 2MB of RAM. A 72-byte struct would cost 144MB instead of 128MB, non-trivial on a 4GB phone.

### 4.5 The mapping Field: Pointer Bit Stealing

`struct address_space` is always aligned to at least 8 bytes (usually more), so legitimate pointers always have bits 0–2 as zero. The kernel steals bit 0:

```
Bit 0 = 0: file-backed page
  mapping is a real address_space pointer. Dereference directly.
  folio->mapping→host gives the inode (file).

Bit 0 = 1: anonymous page
  Mask off bit 0 to get an anon_vma pointer.
  anon_vma is the reverse mapping structure for this page.

NULL: free page or special use.
```

You can test which case applies:

```c
// include/linux/page-flags.h and mm/rmap.c use these checks
if (folio->mapping && !((unsigned long)folio->mapping & 1))
    // file-backed
else if (folio->mapping && ((unsigned long)folio->mapping & 1))
    // anonymous
```

**Why this optimization matters.** File-backed is the common fast path — every page cache lookup, every writeback, every mmap fault checks `mapping`. Requiring a mask operation in the fast path would cost a cycle. Anonymous is the rare path (reclaim, migration), and the mask cost is acceptable there.

The scheme also lets the *same pointer field* serve two roles without adding a separate type tag — saving bytes in `struct page` at the cost of a tiny bit of complexity in the handful of places that care.

### 4.6 The index Field

`folio->index` (aliased as `page->__folio_index` in `struct page`) means different things for different page types:

```
File page:  page offset within the file (page 0, 1, 2, ...)
            Combined with mapping, uniquely identifies this page.

Anon page:  virtual page number where this page was first created
            (the address >> 12 from the fault that allocated it)
            Used during reverse mapping to compute virtual addresses.

Shmem:      page offset within the shmem inode (like file).
```

For file pages, `(mapping, index)` is the primary key in the page cache. `filemap_get_folio(mapping, index)` is an O(log n) xarray lookup.

### 4.7 _mapcount and _refcount

Two reference counters that mean very different things:

```
_mapcount: how many PTEs point to this page
  Value -1 = unmapped (zero PTEs)
  Value  0 = one PTE
  Value  N = N+1 PTEs
  Used by: COW (reuse page if mapcount is 1?), reclaim (need try_to_unmap?)

_refcount: total references from all sources
  Each PTE: +1
  Page cache xarray: +1
  Kernel code holding folio_get(): +1
  DMA pin: +1
  Active bio: +1
  When this hits 0: page is freed to the buddy allocator
```

They answer different questions. Concrete examples of how they diverge:

```
Scenario                              _mapcount    _refcount
──────────────────────────────────────────────────────────────
Just allocated, not yet used              -1           1
In page cache, not mapped by anyone       -1           1   (xarray holds ref)
In page cache, mapped by 1 process         0           2   (xarray + 1 PTE)
In page cache, mapped by 3 processes       2           4   (xarray + 3 PTEs)
Mapped by 1 process, active bio in flight  0           3   (xarray + PTE + bio)
Unmapped, DMA-pinned for GPU               -1           2   (xarray + DMA pin)
Being freed (refcount about to hit 0)     -1           0   → buddy allocator
```

Reclaim checks `_mapcount` first: if ≥ 0, it must call `try_to_unmap` to clear all PTEs before the page can be freed. If -1, no PTEs exist and reclaim skips straight to removing the page from the page cache (which drops the xarray's refcount, bringing `_refcount` to 0, which triggers the actual free).

The offset encoding of `_mapcount` (−1 for zero PTEs) is a trick so that `atomic_add_negative(-1, &page->_mapcount)` returns true exactly when the page becomes unmapped. This lets `folio_remove_rmap_pte` atomically decrement and check "was this the last PTE?" in one instruction.

### 4.8 Page Flags

```
PG_locked       page is locked for I/O (others sleep waiting on folio_wait_locked)
PG_uptodate     page contents are valid (set after successful read from disk)
PG_dirty        page modified, needs writeback to disk
PG_writeback    bio in flight writing this page to disk
PG_lru          page is on an LRU list (reclaim can find it)
PG_active       page is on the active LRU (hot, keep it)
PG_swapbacked   page goes to swap on reclaim (anon or tmpfs)
PG_referenced   page was recently accessed
PG_waiters      someone is sleeping on this folio — wake them on unlock
PG_reclaim      page is queued for reclaim
```

From `include/linux/page-flags.h` (see lines 94–111 for the enum):

```c
enum pageflags {
    PG_locked,         /* Page is locked. Don't touch. */
    PG_writeback,      /* Page is under writeback */
    PG_referenced,
    PG_uptodate,
    PG_dirty,
    PG_lru,
    PG_active,
    // ...
    PG_swapbacked,     /* Page is backed by RAM/swap */
    // ...
};
```

Flags are tested, set, and cleared via atomic helpers: `PageDirty(page)`, `SetPageDirty(page)`, `ClearPageDirty(page)`, `TestSetPageLocked(page)`. The atomic test-and-set variants are critical — many flag transitions happen under concurrent access and the atomic ops prevent lost updates.

The upper bits of `flags` encode the NUMA node and zone ID, so the kernel can determine a page's node/zone from flags alone without extra lookups. This is the "FIELDS" area at the high end of the flags word mentioned in the `page-flags.h` comment.

### 4.9 struct folio

A folio is a type-level guarantee that you have the head page of a (possibly compound) page. In memory, a folio IS a page — `page_folio(page)` is just a cast.

```c
// include/linux/mm_types.h:401
struct folio {
    union {
        struct {
            memdesc_flags_t flags;              // line 406
            struct address_space *mapping;       // line 421
            pgoff_t index;                       // line 423 — file offset in pages
            atomic_t _mapcount;                  // line 430
            atomic_t _refcount;                  // line 431
        };
        struct page page;                        // line 445 — same memory!
    };
    // Second cacheline: for large folios only
    atomic_t _large_mapcount;                    // line 454
    atomic_t _nr_pages_mapped;                   // line 455
};
```

**Why folios exist.** Before folios, compound pages (multiple contiguous pages treated as one unit — THP, hugetlb) required constant `compound_head()` checks: find the head page before accessing metadata like `mapping` or `index`, which are stored only on the head. Forgetting this check was a recurring source of bugs, because the code compiled and ran fine when the caller happened to pass a head page but silently corrupted state when given a tail page.

With folios, the type system enforces the invariant:

```c
// Function takes a folio — guaranteed to be the head page
void some_function(struct folio *folio) {
    folio->mapping;  // safe, no compound_head() needed
}
```

A folio of order 0 is a single 4KB page. A folio of order 4 is 16 contiguous pages (64KB). Large folios reduce per-page overhead: one order-4 folio has one LRU entry, one refcount, one dirty check instead of 16 of each. This matters for THP performance and for modern "large folio" work in the filesystem layer.

The transition from `struct page *` to `struct folio *` is still ongoing. Much of mm/ now uses folios; some older code still deals in raw pages. `page_folio(page)` bridges the two. When you see both types in the same function, the pattern is usually: take a page from a tracepoint or legacy callsite, immediately convert to folio for all real work.

`page_folio(page)` is a cast — no allocation, no indirection — because the first bytes of a folio are exactly the bytes of its head page.

### 4.10 The Page Cache (address_space)

Every file that has been read or mmap'd has an `address_space` struct that acts as a cache of the file's contents in RAM. This is the **page cache** — the single most important performance feature in the kernel.

```c
// include/linux/fs.h:470
struct address_space {
    struct inode        *host;            // back pointer: page → file (Question 2)
    struct xarray       i_pages;          // page cache: file offset → folio (Question 1)
    gfp_t               gfp_mask;         // allocation flags for new pages
    atomic_t            i_mmap_writable;  // how many writable shared mmaps
    struct rb_root_cached i_mmap;         // reverse map: all VMAs mapping this file (Question 3)
    unsigned long       nrpages;          // pages currently in cache
    pgoff_t             writeback_index;  // where writeback left off
    const struct address_space_operations *a_ops; // filesystem callbacks (readahead, writepages)
    unsigned long       flags;            // AS_* flags
    errseq_t            wb_err;           // most recent writeback error
};
```

Both `i_pages` and `i_mmap` live on the same struct. One file, one `address_space`, one page cache index, one reverse mapping tree. The `address_space` is embedded directly inside `struct inode` as the `i_data` field — not a separate allocation.

The `i_pages` xarray is the heart of the page cache. It is indexed by page offset within the file — given a file offset, you can find the corresponding cached page in O(log n) time. The xarray also supports **marks** (tags) on entries, which the kernel uses to efficiently find dirty or writeback pages without scanning the entire cache:

```c
// include/linux/fs.h:498
#define PAGECACHE_TAG_DIRTY     XA_MARK_0   // pages that need writing
#define PAGECACHE_TAG_WRITEBACK XA_MARK_1   // pages currently being written
#define PAGECACHE_TAG_TOWRITE   XA_MARK_2   // pages queued for writeback
```

When the kernel needs a page from a file, it first checks the page cache:

```c
folio = filemap_get_folio(mapping, index);
```

If the page is there (cache hit), no disk I/O is needed — the data is already in RAM. If it is not there (cache miss), the kernel allocates a fresh page, reads the data from disk into it, and inserts it into the page cache for next time. On a typical system, page cache hit rates are 95-99%.

**The page cache has no fixed size.** It grows to fill unused RAM. When processes need memory, reclaim shrinks it by evicting clean pages (dirty pages are written back first). This is why a freshly-booted Linux system with 16GB RAM shows 90% "used" memory — most of it is page cache, which is trivially reclaimable when an application wants RAM.

**mmap vs read():**

```
mmap:   PTE points directly at page cache page. Zero copying.
        200 processes mapping libc = 1 copy in RAM.

read(): kernel copies data from page cache to userspace buffer.
        200 processes reading libc = 1 cache copy + 200 buffer copies.
```

This is why dynamic linkers use mmap for shared libraries — one copy of libc shared by every process. Tools like `vmtouch` and `fincore` (Part 6) let you observe page cache residency directly.

### 4.11 The i_mmap Reverse Map

The `i_mmap` field in `struct address_space` is an **interval rbtree** of all VMAs mapping this file. Keyed by file offset range (`vm_pgoff` to `vm_pgoff + nr_pages`). It answers: "which VMAs overlap file page X?"

```
libc.so mapped by 3 processes:
  Process A: vm_start=0x7f000000, vm_pgoff=0   (pages 0–255)
  Process B: vm_start=0x40000000, vm_pgoff=0   (pages 0–255)
  Process C: vm_start=0x80000000, vm_pgoff=128 (pages 128–143)

Query: which VMAs contain page 130?
  A: 0–255 → yes
  B: 0–255 → yes
  C: 128–143 → yes
  All three.
```

For each matching VMA, the kernel computes the virtual address of page 130 in that process:

```
virtual_addr = vm_start + (page_index - vm_pgoff) × PAGE_SIZE
```

Process A: `0x7f000000 + (130 - 0) × 4096 = 0x7f082000`
Process B: `0x40000000 + (130 - 0) × 4096 = 0x40082000`
Process C: `0x80000000 + (130 - 128) × 4096 = 0x80002000`

Then for each process, walk that process's page table (`vma->vm_mm->pgd`) to find the PTE at the computed virtual address and clear it. Three VMAs checked, three PTEs cleared — not millions.

The question is a **range containment query**: "which VMAs contain page 130?" A VMA mapping pages 0–255 contains page 130, but you would never find it by hashing the number 130 or doing an exact-match lookup. A hash table finds exact keys; a linked list requires scanning every entry. An interval rbtree answers range containment in O(log n + k) where k is the number of matches — the tree can skip entire subtrees whose intervals do not contain the target.

**Why this matters for reclaim.** Before the kernel can free a file-backed page, it must clear every PTE that points to it. Without `i_mmap`, finding those PTEs would require scanning every process's page tables — impractical. With `i_mmap`, the kernel walks a small set of VMAs and for each one clears exactly one PTE. This is what `try_to_unmap` does in the reclaim path (Part 12).

### 4.12 Anonymous Reverse Mapping (anon_vma)

Anonymous pages have no file, no `address_space`, no `i_mmap`. Instead, the `mapping` field (with bit 0 set) points to an `anon_vma` — a structure linking all VMAs that might share this anonymous page.

**Why this structure is needed.** Without reverse mapping for anonymous pages, the kernel cannot reclaim them — it has no way to find and clear all PTEs pointing to an anonymous page, so it cannot safely free it. Before anon_vma was added (around kernel 2.6), anonymous pages were essentially mlocked. Adding anon_vma enabled swap for anonymous pages.

**Why not store PTE pointers directly on the page.** Two problems:

1. **Multiple mappings.** After fork, two processes have PTEs pointing to the same page. You would need a variable-length list of PTE pointers stored on each 64-byte `struct page`. There is no room — adding 16 bytes per potential mapping to `struct page` would cost hundreds of megabytes across 2 million pages.

2. **PTEs move.** When `mremap` moves a mapping, or `mprotect` splits a VMA into pieces, PTEs may end up in different page table pages at different addresses. Every page mapped in the affected VMA would need its stored PTE pointer updated — thousands of updates per VMA operation.

The `anon_vma` approach is indirect but stable. The `anon_vma` points to VMAs (few per region). Each VMA knows its `vm_start`. The virtual address is recomputed on demand from the folio's `index` field (which stores the virtual page number at birth). The PTE is found by walking the page table — never stored directly.

Sharing happens via fork: parent and child both have PTEs pointing to the same physical page.

```c
// include/linux/rmap.h:32
struct anon_vma {
    struct anon_vma *root;        /* Root of this anon_vma tree */
    struct rw_semaphore rwsem;    /* Serialize access to vma list */
    atomic_t refcount;
    // ...
    struct rb_root_cached rb_root; /* Interval tree of anon_vma_chain */
};

// include/linux/rmap.h:83
struct anon_vma_chain {
    struct vm_area_struct *vma;
    struct anon_vma *anon_vma;
    struct list_head same_vma;   /* linked list of anon_vma_chains */
    struct rb_node rb;           /* interval tree node */
    unsigned long rb_subtree_last;
};
```

Visualizing the relationship:

```
folio (anon page)
  └── mapping (bit 0 set) → anon_vma
                              ├── anon_vma_chain → process A's VMA
                              └── anon_vma_chain → process B's VMA (added at fork)
```

For anonymous pages, `folio->index` stores the virtual page number (not a file offset). During reverse mapping, this is used to compute the virtual address in each VMA that might map the page.

The `anon_vma_chain` indirection exists because of `fork()`: the child's VMA links back to the parent's `anon_vma` (so pages allocated before the fork can still be found in both processes) while also getting its own `anon_vma` for pages allocated after (so later COW copies don't pollute the parent's tree). This "tree of anon_vmas" is one of the more intricate parts of the mm subsystem — the brief `struct anon_vma { struct anon_vma *root; ...}` field is what stitches it together.

### 4.13 struct bio — the I/O descriptor

When the kernel needs to actually read or write disk sectors, it constructs a `bio`:

```c
// include/linux/blk_types.h:210
struct bio {
    struct bio          *bi_next;      // linked list within a request
    struct block_device *bi_bdev;      // which disk
    blk_opf_t           bi_opf;        // REQ_OP_READ, REQ_OP_WRITE, + flags
    unsigned short      bi_flags;      // internal flags
    blk_status_t        bi_status;     // completion status
    atomic_t            __bi_remaining; // outstanding sub-bios
    struct bio_vec      *bi_io_vec;    // scatter-gather list of pages
    struct bvec_iter    bi_iter;       // current position (sector, size)
    bio_end_io_t        *bi_end_io;    // completion callback
    void                *bi_private;   // caller's context
};
```

A `bio` is essentially a scatter-gather list: "write these pages to these disk sectors." The `bi_io_vec` array contains entries like:

```c
struct bio_vec {
    struct page *bv_page;    // the page containing the data
    unsigned int bv_len;     // how many bytes
    unsigned int bv_offset;  // offset within the page
};
```

Multiple `bio` structs can be merged into a single `request` by the block layer's I/O scheduler, which batches adjacent disk operations for efficiency.

### 4.14 How Everything Connects

Here is the graph of pointers that tie these structures together. The tracer in Part 18 traverses this graph constantly, so it is worth burning into memory:

```
Finding the folio from the file:
  inode → address_space → i_pages xarray → folio

Finding the file from the folio:
  folio → mapping → address_space → host → inode

Finding the folio from a virtual address:
  mm → page table → PTE → PFN → pfn_to_page() → struct page → folio

Finding all processes mapping a folio:
  File:  folio → mapping → i_mmap rbtree → VMAs → compute addr → PTEs
  Anon:  folio → mapping & ~1 → anon_vma → chain → VMAs → compute addr → PTEs
```

Every arrow is bidirectional. This connectivity is what makes reclaim, migration, COW, and sharing possible. Lose any one of these pointers and the whole system breaks: without `i_mmap`, reclaim cannot unmap file pages; without `anon_vma`, it cannot unmap anonymous pages; without `mapping` on the folio, it cannot find the owning file to remove the page cache entry; without `_mapcount`, COW cannot decide whether to reuse or copy.

Every one of these pointers is updated atomically with respect to the others, using a combination of spinlocks (for fast-path updates like PTE installation), RCU (for readers walking the trees), and RW semaphores (for long-duration operations like mmap). Part 16 dissects the locking discipline.

### 4.15 Key Identity Relationships Summary

```
Physical address ←→ PFN:       just bit shift by 12
PFN ←→ struct page:             array indexing (mem_map[pfn])
Virtual address → PFN:          4-level page table walk (MMU or kernel)
PFN → virtual addresses:        reverse mapping (i_mmap or anon_vma)
File + offset → folio:          page cache xarray lookup
Folio → file + offset:          folio->mapping + folio->index
VMA → page table:               VMA provides address range, walk mm->pgd
Process → VMAs:                 mm_struct → maple tree
```

### 4.16 The hybrid case: MAP_PRIVATE file after COW

A single VMA can have BOTH `vm_file` and `anon_vma`:

```
Process mmaps libc.so MAP_PRIVATE, then writes to page 130.

VMA:
  vm_file  = libc.so          (still a file mapping)
  anon_vma = anon_vma_X       (COW created anonymous pages)

Page 129 (unmodified):
  folio->mapping → libc's address_space (bit 0 = 0, file-backed)
  Still in the page cache, shared with other readers

Page 130 (COW'd):
  folio->mapping → anon_vma_X | 1 (bit 0 = 1, anonymous)
  Private copy, NOT in page cache
  Original file page 130 still in cache for other processes
```

The kernel checks `folio->mapping` bit 0 *per page* to decide which reverse mapping path to use. Same VMA, different pages, different mechanisms.

### 4.17 Side-by-side: file page vs anonymous page

```
                        FILE PAGE                  ANONYMOUS PAGE
────────────────────────────────────────────────────────────────────
Owner structure:        address_space              anon_vma

folio->mapping:         address_space ptr          anon_vma ptr | 1
                        (bit 0 = 0)                (bit 0 = 1)

folio->index:           file page offset           virtual page number
                        (e.g., 130)                (e.g., 0x7f500)

Page cache:             YES (i_pages xarray)       NO

Reverse map struct:     i_mmap (interval rbtree)   anon_vma->rb_root

Virtual addr formula:   vm_start +                 folio->index × 4096
                        (index - vm_pgoff) × 4096

Why shared:             processes mmap same file    fork() shares pages

Reclaim writes to:      disk (writeback)           swap/zram

VMA field:              vm_file != NULL            vm_file == NULL
                        vm_pgoff meaningful        anon_vma set
```

### 4.18 Why each structure must exist

```
Without i_pages (xarray):
  Every file read scans all 2M page frames to find cached pages

Without folio->mapping:
  Cannot find which file a page belongs to — writeback, reverse
  mapping, and page cache removal all break

Without i_mmap:
  Reclaim cannot find PTEs for file pages — must scan every PTE
  in every process (hundreds of millions of checks per page)

Without anon_vma:
  Reclaim cannot find PTEs for anonymous pages — same problem

Without folio->index:
  Cannot compute virtual addresses during reverse mapping —
  cannot find the specific PTE to clear

Without the bit 0 trick:
  Need a separate type field on folio — 8 bytes × 2M pages = 16MB

Without mm_mt (maple tree):
  Cannot find which VMA covers a faulting address —
  cannot distinguish legitimate faults from access violations
```

Every structure exists because a specific operation would be impossibly slow without it.

---

## Part 5: Process Memory Lifecycle — fork, COW, and the Zero Page {#part-5-process-memory-lifecycle-fork-cow-and-the-zero-page}

### 5.1 What fork() Actually Copies

```
Copied:
  mm_struct                 new allocation, independent
  All VMAs                  independent copies, same values
  Entire page table tree    all levels including leaf PTEs
  Leaf PTE values           same PFNs, but marked READ-ONLY (COW setup)

NOT copied:
  Physical data pages       shared, mapcount incremented
  struct page descriptors   global array, just update mapcount

Cost: proportional to VMAs + page table pages (~1–2MB for a typical process)
Not proportional to data (~hundreds of MB)
```

A process using 500MB of RAM forks in milliseconds because only the metadata — VMAs, page tables — is copied. The 500MB of data is shared until one of the processes writes.

### 5.2 Copy-on-Write (COW)

After fork, both parent and child PTEs point to the same physical pages, marked read-only. When either writes:

1. Write fault (PTE is read-only).
2. Kernel checks VMA — `VM_WRITE` is set, so this is a COW fault, not a violation.
3. Check `_mapcount` — if > 0 (shared), must copy.
4. Allocate new physical page from buddy allocator.
5. Copy 4096 bytes from old page to new page.
6. Update the writer's PTE to point to new page, set writable.
7. If old page's mapcount drops to 0 (exclusive to the other process), make the other's remaining PTE writable too — avoid a second, pointless fault.

If the VMA doesn't have `VM_WRITE`: SIGSEGV. The PTE alone doesn't carry enough information — the VMA determines whether a write fault is COW or a real permission violation.

We dig into the exact code in Part 8 (Anonymous Memory and Copy-on-Write), including the TLB-flush ordering that prevents two CPUs from seeing different mappings during the transition.

### 5.3 vfork() and posix_spawn()

`fork() + exec()` is wasteful. All that VMA and page table copying is thrown away the moment `exec()` destroys the address space.

**`vfork()`** skips all copying — child shares parent's `mm_struct` directly, parent is suspended until the child calls `exec()` or `_exit()`. This is brutally fast (microseconds regardless of process size) but dangerous: child must only call those two things. Any other operation (malloc, setenv, return) corrupts the parent's memory because they share it.

**`posix_spawn()`** wraps vfork safely with a declarative interface — you describe what to do between fork and exec (close fds, redirect I/O, set signals) and the kernel does it. No application code runs in the shared address space, so no corruption risk. Android's `Zygote` process uses a similar fork-based model with careful discipline about what runs between fork and the first dalvik class initialization.

### 5.4 Anonymous Page Allocation

When a process first writes to anonymous memory:

**Read fault on anonymous memory:** the kernel maps the shared **zero page** — one system-wide, read-only page of zeros, at a well-known PFN (`zero_pfn`, defined at `mm/memory.c:165`).

```c
// mm/memory.c:163
unsigned long zero_pfn __read_mostly;
EXPORT_SYMBOL(zero_pfn);
```

Every process's unwritten anonymous pages point to this same zero page. `malloc(1GB)` followed by reading the memory costs zero physical RAM — a read fault installs a read-only PTE pointing to `zero_pfn`, and the data (all zeros) is already there. This is why `top` can show a process with 1GB `VSZ` but only a few MB `RSS`.

**Write fault on anonymous memory:** allocate a real zeroed page from the buddy allocator. Born dirty (no on-disk copy). Set up anon_vma reverse mapping. Add to LRU. The PTE is replaced with one pointing to the new page, writable.

If the original PTE pointed to the zero page (a prior read fault), this is a COW-from-zero: the zero page's mapcount is not incremented (it's shared globally, tracking would be pointless), and the new page is allocated and zeroed anyway.

### 5.5 tmpfs / shmem — The In-Between

tmpfs has `vm_file != NULL` — structurally file-backed with an inode, address_space, and page cache. But its pages go to swap on reclaim (like anonymous), not to disk (like regular files), and the file disappears when the filesystem is unmounted.

```
                Regular file      tmpfs/shmem        Anonymous
Backing store:  disk              swap/zram          swap/zram
vm_file:        yes               yes                NULL
Page cache:     yes               yes                no
vm_ops->fault:  filemap_fault     shmem_fault        NULL (hardcoded path)
PG_swapbacked:  no                yes                yes
Survives reboot: yes              no                 no
```

The `PG_swapbacked` flag is what tells reclaim to write to swap/zram instead of trying writeback to disk. File pages are not swapbacked — they go to their filesystem. Shmem pages are swapbacked — they go to swap even though they have an `address_space` and sit in the page cache.

This hybrid matters on Android:

- **ashmem / memfd:** Android's inter-process shared memory (used by SurfaceFlinger, Binder, and app-to-app communication) is backed by tmpfs. These pages are in the page cache with an inode, but under memory pressure they get compressed into zRAM like anonymous pages.
- **GPU buffers:** Some GPU drivers allocate backing storage via shmem. The pages look file-backed to `mm_filemap_add_to_page_cache` tracepoints but behave like anonymous pages during reclaim.
- **ART runtime:** The Android Runtime's boot image can be backed by memfd (a tmpfs variant), meaning the shared runtime data across all apps is shmem-backed.

The tracer needs to handle this: shmem pages fire the `mm_filemap_add_to_page_cache` tracepoint (because they enter the page cache), and the `on_cache_add` handler will see a valid `i_ino` and `s_dev`. But during reclaim, these pages go through the swap path, not the writeback path. A page with `EVT_CACHE_INSERT` followed by `EVT_ZRAM_COMPRESS` (instead of `EVT_WRITEBACK`) is a shmem page.

---

## Part 6: Observing the Page Cache — Tools of the Trade {#part-6-observing-the-page-cache-tools-of-the-trade}

Before you build a complex eBPF tracer, learn what's already available. Many questions about page cache residency and memory use have answers in `/proc` and existing tools.

### 6.1 System-wide

```bash
cat /proc/meminfo | grep -E "Cached|Buffers"
# Cached: file-backed page cache
# Buffers: cache of raw block device reads (usually small)
```

`/proc/meminfo` is the quickest way to see how much memory is in the page cache vs other uses. `MemAvailable` is an estimate of how much can be allocated without swapping — includes reclaimable cache.

### 6.2 Per-file

```bash
vmtouch -v /usr/lib/*.so*           # cache residency per file
fincore /usr/lib/libc.so.6          # pages cached / total pages
```

`vmtouch` can also *pin* pages in cache (`vmtouch -t` = touch = cache, `vmtouch -l` = lock in RAM). Useful for pre-warming caches before a benchmark.

### 6.3 Per-process page info

```bash
cat /proc/pid/maps                    # VMAs (no root needed)
cat /proc/pid/pagemap                 # PTE-level info (binary, root for PFNs)
cat /proc/kpageflags                  # flags per PFN (root)
cat /proc/kpagecount                  # mapcount per PFN (root)
sudo tools/mm/page-types -p PID       # categorized page breakdown
```

`/proc/pid/pagemap` gives you one 64-bit entry per virtual page:
- Bits 0–54: the PFN (if the page is present in RAM).
- Bit 61: page is file-backed or shared anonymous.
- Bit 62: page is swapped.
- Bit 63: page is present.

Since kernel 4.0, reading PFNs requires `CAP_SYS_ADMIN` — rowhammer mitigation.

### 6.4 Programmatically

```c
#include <sys/mman.h>

void *addr = mmap(NULL, size, PROT_READ, MAP_PRIVATE, fd, 0);
unsigned char *vec = malloc(size / 4096);
mincore(addr, size, vec);
// vec[i] & 1 == 1  →  virtual page i is in the page cache
// vec[i] & 1 == 0  →  accessing it would fault
```

`mincore` doesn't require root — it tells you about your own mapping. Use it to decide whether a read will be cheap (cache hit) or expensive (disk read).

### 6.5 Drop the cache

```bash
echo 3 | sudo tee /proc/sys/vm/drop_caches
```

- `1` drops the page cache.
- `2` drops slab reclaimable objects (dentries, inodes).
- `3` drops both.

Only clean pages are dropped. Dirty pages skipped — they'd need writeback first. This is useful before benchmarks ("make sure the next read goes to disk") and for debugging cache behavior. Don't use it in production; you'll just force re-reads.

### 6.6 The "Free" Number Is Misleading

The `free` command shows a "free" column that seems too low and a "used" column that seems too high. This is because the page cache counts as "used" in the top row.

```
$ free -h
              total   used   free   shared  buff/cache   available
Mem:          16Gi    11Gi   1Gi    500Mi   4Gi          4Gi
```

Here, `used - buff/cache ≈ actual allocation` (7GB), and `available ≈ 4GB` is what an allocation can grow into by reclaiming cache. This is normal and healthy — empty RAM is wasted RAM. The kernel keeps the cache hot until something needs the memory more.

---

## Part 7: The Page Fault — Where Everything Starts {#part-7-the-page-fault-where-everything-starts}

### 7.1 Why faults are normal

The word "fault" sounds like an error. It is not. Page faults are the primary mechanism by which memory gets populated. Almost nothing is set up eagerly:

- `mmap()` creates a VMA but installs zero PTEs.
- `fork()` copies PTEs but marks them read-only (COW setup).
- `malloc()` returns an address backed by nothing (not even a PTE).

The first time you touch the memory, the MMU says "I can't translate this" and raises an exception. The kernel handles it — allocates a page, reads from disk, copies on write — installs a PTE, and returns. Your program retries the instruction and succeeds. It never knows a fault happened.

On a typical Android app launch, 10,000–50,000 page faults fire in the first few milliseconds. This is completely normal — the kernel being lazy in a good way, only doing work when actually needed. The benefit: memory is never allocated unless it is actually accessed, and the cost of that laziness (one fault per first access) is amortized over the lifetime of the page.

### 7.2 The architecture-level entry point

When the CPU raises a page fault, the architecture-specific handler (e.g., `arch/arm64/mm/fault.c`) runs first. Before calling into generic mm code, it does critical triage:

```c
// Simplified from arch/arm64/mm/fault.c:
// 1. Read the faulting address from hardware register (FAR_EL1 on ARM64)
unsigned long addr = read_faulting_address();

// 2. Find the VMA
struct vm_area_struct *vma = find_vma(mm, addr);

// 3. Is this address within any VMA?
if (!vma || addr < vma->vm_start)
    // No VMA covers this address → genuine SIGSEGV
    return handle_bad_area(addr);

// 4. Check permissions against VMA flags
if (is_write && !(vma->vm_flags & VM_WRITE))
    // Writing to non-writable region → SIGSEGV
    return handle_bad_area(addr);

// 5. VMA found, permissions OK → handle the fault
handle_mm_fault(vma, addr, flags, regs);
```

This is the first place the VMA matters. The page table alone can't distinguish "memory that hasn't been allocated yet" (legitimate fault, PTE is just empty) from "memory that was never mapped" (access violation, no VMA covers it). The VMA makes that distinction. Without the VMA lookup, the kernel would have no way to tell a `malloc`'d but untouched page from a wild pointer dereference.

### 7.3 The generic entry point

The architecture code calls `handle_mm_fault` at `mm/memory.c:6589`:

```c
// mm/memory.c:6589
vm_fault_t handle_mm_fault(struct vm_area_struct *vma, unsigned long address,
                           unsigned int flags, struct pt_regs *regs)
{
    struct mm_struct *mm = vma->vm_mm;
    // ...
    if (unlikely(is_vm_hugetlb_page(vma)))
        ret = hugetlb_fault(vma->vm_mm, vma, address, flags);
    else
        ret = __handle_mm_fault(vma, address, flags);  // the normal path
    // ...
}
```

### 7.4 Walking the page tables

`__handle_mm_fault` at `mm/memory.c:6355` walks down the page table, allocating intermediate levels as needed:

```c
static vm_fault_t __handle_mm_fault(struct vm_area_struct *vma,
        unsigned long address, unsigned int flags)
{
    struct vm_fault vmf = {
        .vma = vma,
        .address = address & PAGE_MASK,
        .pgoff = linear_page_index(vma, address),  // file offset for this address
        .gfp_mask = __get_fault_gfp_mask(vma),
    };
    struct mm_struct *mm = vma->vm_mm;

    pgd = pgd_offset(mm, address);                  // look up in top-level table
    p4d = p4d_alloc(mm, pgd, address);               // allocate level if missing
    vmf.pud = pud_alloc(mm, p4d, address);            // allocate level if missing
    // ... checks for huge pages at PUD and PMD level ...
    vmf.pmd = pmd_alloc(mm, vmf.pud, address);        // allocate level if missing
    // falls through to:
    return handle_pte_fault(&vmf);
}
```

**The `pgoff` field is important for file-backed VMAs.** It computes which page of the file corresponds to this virtual address:

```
pgoff = vma->vm_pgoff + (address - vma->vm_start) / PAGE_SIZE
```

If the VMA maps a file starting at offset 0 and the faulting address is 8192 bytes into the VMA, then `pgoff = 2` — we need page 2 of the file. This pgoff will be used to look up the page in the page cache via `filemap_get_folio(mapping, pgoff)`.

The `p4d_alloc`, `pud_alloc`, `pmd_alloc` functions allocate physical pages for intermediate page table levels if the current entry is NULL. These are non-movable kernel pages from the buddy allocator (`GFP_KERNEL`). This is demand-driven — page table levels are only created when a fault walks through them. A process that maps 1TB of address space but only touches 4KB will have exactly the page table entries needed for that one page.

Also notice the huge page checks — at the PUD level (1GB pages) and PMD level (2MB pages on x86, or variable on ARM64), the kernel can take a shortcut and map a single large page instead of walking down to individual PTEs. Android uses Transparent Huge Pages (THP) for some workloads, which can reduce TLB pressure significantly.

### 7.5 The five-way dispatch

`handle_pte_fault` at `mm/memory.c:6273` is the critical decision point. It reads the PTE and decides what to do:

```c
static vm_fault_t handle_pte_fault(struct vm_fault *vmf)
{
    // Try to get the PTE without locking (optimistic)
    if (unlikely(pmd_none(*vmf->pmd))) {
        vmf->pte = NULL;                    // no PMD → no PTE exists
    } else {
        vmf->pte = pte_offset_map_rw_nolock(vmf->vma->vm_mm, vmf->pmd,
                                            vmf->address, ...);
        vmf->orig_pte = ptep_get_lockless(vmf->pte);  // read PTE without lock

        if (pte_none(vmf->orig_pte)) {
            pte_unmap(vmf->pte);
            vmf->pte = NULL;                // PTE slot exists but is empty
        }
    }

    if (!vmf->pte)
        return do_pte_missing(vmf);         // ① no PTE → allocate a page

    if (!pte_present(vmf->orig_pte))
        return do_swap_page(vmf);           // ② PTE exists but page was swapped out

    if (pte_protnone(vmf->orig_pte) && vma_is_accessible(vmf->vma))
        return do_numa_page(vmf);           // ③ NUMA balancing hint fault

    // PTE is present and valid
    spin_lock(vmf->ptl);
    // re-check PTE under lock (it could have changed)
    if (vmf->flags & (FAULT_FLAG_WRITE|FAULT_FLAG_UNSHARE)) {
        if (!pte_write(entry))
            return do_wp_page(vmf);         // ④ write to read-only PTE → COW
        else
            entry = pte_mkdirty(entry);     // already writable → just mark dirty
    }
    entry = pte_mkyoung(entry);             // mark as recently accessed
    // update hardware PTE
    ptep_set_access_flags(vmf->vma, vmf->address, vmf->pte, entry, ...);
}
```

This is a five-way dispatch:

1. **PTE missing** (`do_pte_missing`): No physical page yet. If the VMA is file-backed, call `do_fault()`. If anonymous, call `do_anonymous_page()`.
2. **Swap entry** (`do_swap_page`): The page was evicted and its PTE was replaced with a swap entry. Read it back from swap/zram.
3. **NUMA hint** (`do_numa_page`): The PTE was deliberately made inaccessible so the kernel can detect which NUMA node the process is accessing from. Used to migrate pages closer to the CPU using them.
4. **Write-protect** (`do_wp_page`): A write to a read-only PTE. Could be COW (after fork) or a shared mapping becoming writable.
5. **Normal access**: PTE exists and has sufficient permissions. Just update the accessed/dirty bits and return.

### 7.6 File-backed fault: do_fault and its three sub-paths

`do_fault` at `mm/memory.c:5903` splits into three paths depending on the type of access:

```c
if (!(vmf->flags & FAULT_FLAG_WRITE))
    ret = do_read_fault(vmf);           // read fault
else if (!(vma->vm_flags & VM_SHARED))
    ret = do_cow_fault(vmf);            // write fault, private mapping → COW
else
    ret = do_shared_fault(vmf);         // write fault, shared mapping
```

**Why three paths?** Consider what `mmap` mode you used:

- `MAP_PRIVATE` + reading: The kernel can just map the page cache page directly. Read-only, shared with all other readers. This is `do_read_fault`.
- `MAP_PRIVATE` + writing: The kernel must make a private copy first (you should not modify the page cache version of a private mapping). This is `do_cow_fault`.
- `MAP_SHARED` + writing: The kernel maps the page cache page writable and marks it dirty. Writes propagate back to the file. This is `do_shared_fault`.

Tracing `do_read_fault`:

```c
// mm/memory.c:5779
static vm_fault_t do_read_fault(struct vm_fault *vmf)
{
    // Optimization: try to map several nearby pages at once ("fault-around")
    if (should_fault_around(vmf)) {
        ret = do_fault_around(vmf);      // maps up to 16 pages in one shot
        if (ret) return ret;
    }

    ret = __do_fault(vmf);               // calls vma->vm_ops->fault (filemap_fault)
    ret |= finish_fault(vmf);            // installs the PTE
    folio_unlock(folio);
    return ret;
}
```

The `__do_fault` call ends up in `filemap_fault` at `mm/filemap.c:3512`:

```c
vm_fault_t filemap_fault(struct vm_fault *vmf)
{
    struct file *file = vmf->vma->vm_file;
    struct address_space *mapping = file->f_mapping;  // the page cache
    pgoff_t index = vmf->pgoff;                       // which page of the file

    // Look in the page cache:
    folio = filemap_get_folio(mapping, index);
    if (likely(!IS_ERR(folio))) {
        // Cache HIT — the page is already in RAM.
        // Trigger async readahead for the pages after this one.
        fpin = do_async_mmap_readahead(vmf, folio);
    } else {
        // Cache MISS — page is not in RAM.
        count_vm_event(PGMAJFAULT);     // this is a major fault (needs disk I/O)
        fpin = do_sync_mmap_readahead(vmf);  // kick off read I/O

        folio = __filemap_get_folio(mapping, index,
                    FGP_CREAT|FGP_FOR_MMAP, vmf->gfp_mask);
        // FGP_CREAT means: allocate a new folio if one doesn't exist,
        // add it to the page cache, and start reading from disk.
    }
    // ... wait for I/O, check for errors ...
}
```

The distinction between **major** and **minor** faults is important for performance. A minor fault means the page was already in the page cache — the kernel just needs to set up the PTE. A major fault means the kernel had to go to disk, which is 10,000-100,000x slower. On Android, `ActivityManager` tracks major faults per-process and uses them as a signal for app health.

When `FGP_CREAT` triggers a new page allocation, the page eventually gets inserted into the page cache via `__filemap_add_folio` at `mm/filemap.c:848`:

```c
noinline int __filemap_add_folio(struct address_space *mapping,
        struct folio *folio, pgoff_t index, gfp_t gfp, void **shadowp)
{
    XA_STATE_ORDER(xas, &mapping->i_pages, index, folio_order(folio));
    // ...
    folio->mapping = mapping;    // line 868 — folio now knows its owner
    folio->index = xas.xa_index; // line 869 — and its offset in the file

    xas_store(&xas, folio);      // line 915 — insert into the xarray
    mapping->nrpages += nr;      // line 919 — update the page count

    trace_mm_filemap_add_to_page_cache(folio);  // line 939 — tracepoint fires
    return 0;
}
```

After the page is in the cache and the data has been read from disk, `finish_fault` at `mm/memory.c:5556` installs the PTE:

```c
vm_fault_t finish_fault(struct vm_fault *vmf)
{
    // ...
    // Lock the page table entry
    vmf->pte = pte_offset_map_lock(vma->vm_mm, vmf->pmd, addr, &vmf->ptl);

    // Re-check under lock — another thread might have raced us
    if (nr_pages == 1 && unlikely(vmf_pte_changed(vmf))) {
        // Someone else already handled this fault. Bail out.
        ret = VM_FAULT_NOPAGE;
        goto unlock;
    }

    // Install the PTE(s) pointing to the physical page(s)
    folio_ref_add(folio, nr_pages - 1);
    set_pte_range(vmf, folio, page, nr_pages, addr);
    add_mm_counter(vma->vm_mm, type, nr_pages);
}
```

Notice the **re-check under lock**. Between the time `filemap_fault` found the page and the time `finish_fault` locks the PTE, another CPU could have handled the same fault (two threads hitting the same unmapped page simultaneously). The re-check prevents double-mapping. This pattern — optimistic check, then recheck under lock — appears everywhere in the mm subsystem.

---

## Part 8: Anonymous Memory and Copy-on-Write {#part-8-anonymous-memory-and-copy-on-write}

### Anonymous page allocation

When your program calls `malloc()` and then writes to the memory, `malloc` (in libc) either uses `brk()` to extend the heap or `mmap(MAP_ANONYMOUS)` for larger allocations. Either way, the kernel creates a VMA with `vm_file = NULL` and no page table entries. The first write triggers a fault handled by `do_anonymous_page` at `mm/memory.c:5217`:

```c
static vm_fault_t do_anonymous_page(struct vm_fault *vmf)
{
    struct vm_area_struct *vma = vmf->vma;

    // For READS: map the shared zero page (no allocation!)
    if (!(vmf->flags & FAULT_FLAG_WRITE) &&
            !mm_forbids_zeropage(vma->vm_mm)) {
        entry = pte_mkspecial(pfn_pte(my_zero_pfn(vmf->address),
                                      vma->vm_page_prot));
        goto setpte;  // install PTE pointing to the zero page
    }

    // For WRITES: allocate a real page
    folio = alloc_anon_folio(vmf);        // allocate zeroed physical page
    __folio_mark_uptodate(folio);

    entry = folio_mk_pte(folio, vma->vm_page_prot);
    if (vma->vm_flags & VM_WRITE)
        entry = pte_mkwrite(pte_mkdirty(entry), vma);  // writable AND dirty

    // ... install PTE, add to LRU, set up reverse mapping ...
}
```

Two points here:

**The zero page optimization.** When you read from anonymous memory you have never written (e.g., the BSS segment, or freshly `mmap`'d memory), the kernel maps a single shared read-only page filled with zeros. There is exactly one zero page in the entire system, and every process's unwritten anonymous pages point to it. This means `malloc(1GB)` followed by reading it costs zero physical RAM. Only writes trigger real allocation. This is why `ps` and `/proc/pid/status` distinguish between VSS (virtual set size, how much is mapped) and RSS (resident set size, how much physical RAM is actually used).

**Born dirty.** File-backed pages start clean — they match the on-disk copy. Anonymous pages have no on-disk copy, so they are dirty from the moment they are created. Look at line 5285: `pte_mkdirty(entry)`. There is no separate "dirty" event for anonymous pages; the allocation IS the dirty event.

### Copy-on-Write (COW)

COW is one of the most important optimizations in the kernel. When a process calls `fork()`, the kernel does NOT copy all of the parent's memory. Instead:

1. It creates a new `mm_struct` for the child.
2. It copies all VMAs.
3. It copies all page table entries — but marks them **read-only in both parent and child**, even if the VMA allows writing.
4. It increments the `mapcount` of every shared page.

Now both processes share the same physical pages. No copying happens. A `fork()` of a process using 500MB of RAM takes microseconds instead of hundreds of milliseconds.

When either process writes to a shared page, the hardware triggers a write-protection fault because the PTE is read-only. The kernel handles this in `do_wp_page` at `mm/memory.c:4149`:

```c
static vm_fault_t do_wp_page(struct vm_fault *vmf)
{
    struct folio *folio = NULL;
    vmf->page = vm_normal_page(vma, vmf->address, vmf->orig_pte);
    if (vmf->page)
        folio = page_folio(vmf->page);

    // Case 1: Shared mapping → just make writable (no copy needed)
    if (vma->vm_flags & (VM_SHARED | VM_MAYSHARE)) {
        return wp_page_shared(vmf, folio);
    }

    // Case 2: Anonymous page, exclusively owned → reuse without copying
    if (folio && folio_test_anon(folio) &&
        (PageAnonExclusive(vmf->page) || wp_can_reuse_anon_folio(folio, vma))) {
        wp_page_reuse(vmf, folio);   // just make the PTE writable
        return 0;
    }

    // Case 3: Page is shared → must copy
    return wp_page_copy(vmf);
}
```

Case 2 is an important optimization. If the page is anonymous and has `mapcount == 1` (nobody else maps it — perhaps the other process already exited or already COW'd its own copy), the kernel can skip the copy and just make the existing PTE writable. This is common on Android after the zygote forks app processes — once the child writes, it gets its own copy; if the parent (zygote) never writes to the same page, no copy is needed on the parent side.

The actual copy path (`wp_page_copy` at `mm/memory.c:3758`) is meticulous about ordering:

```c
static vm_fault_t wp_page_copy(struct vm_fault *vmf)
{
    // 1. Allocate new page BEFORE acquiring any locks
    new_folio = folio_prealloc(mm, vma, vmf->address, pfn_is_zero);

    // 2. Copy the data
    __wp_page_copy_user(&new_folio->page, vmf->page, vmf);

    // 3. Notify MMU notifiers (for GPU/IOMMU awareness)
    mmu_notifier_invalidate_range_start(&range);

    // 4. Lock the PTE and re-check (might have raced with another fault)
    vmf->pte = pte_offset_map_lock(mm, vmf->pmd, vmf->address, &vmf->ptl);
    if (likely(pte_same(ptep_get(vmf->pte), vmf->orig_pte))) {
        // 5. Flush TLB for old page BEFORE installing new PTE
        ptep_clear_flush(vma, vmf->address, vmf->pte);

        // 6. Set up reverse mapping for new page
        folio_add_new_anon_rmap(new_folio, vma, vmf->address, RMAP_EXCLUSIVE);

        // 7. Install new PTE
        entry = maybe_mkwrite(pte_mkdirty(folio_mk_pte(new_folio, ...)), vma);
        set_pte_at(mm, vmf->address, vmf->pte, entry);

        // 8. Remove reverse mapping from old page
        folio_remove_rmap_pte(old_folio, vmf->page, vma);
    }
    pte_unmap_unlock(vmf->pte, vmf->ptl);
    mmu_notifier_invalidate_range_end(&range);
}
```

The ordering between steps 5, 7, and 8 is critical for correctness. The comment at line 3853 in the kernel explains why:

> *"Only after switching the pte to the new page may we remove the mapcount here. Otherwise another process may come and find the rmap count decremented before the pte is switched to the new page, and 'reuse' the old page writing into it while our pte here still points into it and can be read by other threads."*

The sequence is: clear old PTE and flush TLB → install new PTE → remove old rmap. This ensures no CPU can see a stale TLB entry pointing to the old page after its mapcount drops.

---

## Part 9: Dirty Pages and Writeback — Getting Data to Disk {#part-9-dirty-pages-and-writeback-getting-data-to-disk}

When a process writes to a `MAP_SHARED` file mapping, or when the kernel writes to a page cache page on behalf of a `write()` syscall, the page becomes **dirty** — its in-memory contents differ from what is on disk. The kernel tracks dirty pages and periodically writes them back to the filesystem.

### Marking a page dirty

`folio_mark_dirty` at `mm/page-writeback.c:2778`:

```c
bool folio_mark_dirty(struct folio *folio)
{
    struct address_space *mapping = folio_mapping(folio);

    if (likely(mapping)) {
        if (folio_test_reclaim(folio))
            folio_clear_reclaim(folio);     // don't evict, we're about to write
        return mapping->a_ops->dirty_folio(mapping, folio);  // filesystem callback
    }
    return noop_dirty_folio(mapping, folio);
}
```

For most filesystems, `dirty_folio` points to `filemap_dirty_folio` at `mm/page-writeback.c:2714`:

```c
bool filemap_dirty_folio(struct address_space *mapping, struct folio *folio)
{
    if (folio_test_set_dirty(folio))
        return false;                       // was already dirty, nothing to do

    __folio_mark_dirty(folio, mapping, ...);
    __mark_inode_dirty(mapping->host, I_DIRTY_PAGES);  // tell VFS the inode is dirty
    return true;
}
```

Inside `__folio_mark_dirty` (at line 2685), the key operation is tagging the page in the xarray:

```c
xa_lock_irqsave(&mapping->i_pages, flags);
folio_account_dirtied(folio, mapping);
__xa_set_mark(&mapping->i_pages, folio->index, PAGECACHE_TAG_DIRTY);
xa_unlock_irqrestore(&mapping->i_pages, flags);
```

The `PAGECACHE_TAG_DIRTY` mark is what makes writeback efficient. Instead of scanning every page in the cache to find dirty ones, the writeback code can use `xas_find_marked()` to jump directly to tagged entries. For a file with 10,000 cached pages and only 5 dirty ones, this saves examining 9,995 entries.

### The three triggers of writeback

Dirty pages get written back from three sources. All three converge on the same filesystem callback (`a_ops->writepages`), but they start in different places:

**1. Background timer.** The kernel has a background writeback thread (`kworker/u:flush-*`) per backing device. It periodically wakes up and writes dirty pages back to their filesystems. Two sysctl parameters control it:

- `dirty_expire_centisecs` (default 3000 = 30 seconds): how long a page must be dirty before writeback picks it up.
- `dirty_writeback_centisecs` (default 500 = 5 seconds): how often the writeback thread wakes up to check.

**2. Dirty memory pressure.** If the total amount of dirty memory grows too large, processes that are still dirtying pages get throttled inside `balance_dirty_pages` at `mm/page-writeback.c`. This is the "dirty budget" mechanism:

- `vm_dirty_ratio` (default 20 = 20% of available memory): hard cap — processes block in `balance_dirty_pages` when this is exceeded.
- `dirty_background_ratio` (default 10): soft cap — background writeback accelerates when this is exceeded.

From `mm/page-writeback.c:92`:

```c
static int vm_dirty_ratio = 20;
```

The ratios apply to *dirtyable* memory (free memory + reclaimable cache), not total RAM. The purpose is to prevent one greedy writer from filling all available RAM with dirty pages — which would then take minutes to drain to a slow disk, starving everyone else.

**3. Explicit sync.** `fsync()`, `fdatasync()`, `sync()`, and `msync()` all call into `filemap_write_and_wait_range` at `mm/filemap.c:675`, which immediately initiates writeback and then waits for it to complete. This is the only writeback path that blocks the caller until the data is on stable storage.

### Per-backing-device accounting

Writeback is per-backing-device (per `struct bdi_writeback`), not global. Each disk or file system has its own dirty accounting and its own writeback thread, so a slow USB drive cannot stall writeback to a fast NVMe. Each backing device tracks:

- Number of pages dirty on this BDI.
- Number of pages currently under writeback on this BDI.
- Write bandwidth estimate (updated in `wb_bandwidth_estimate_start`).

The bandwidth estimate feeds back into `balance_dirty_pages`: slower devices cause earlier throttling of their dirtying processes, so fast devices don't suffer because of a slow one.

The writeback path calls the filesystem's `writepages` callback, which builds `bio` structures containing the dirty pages and submits them to the block layer. After the I/O completes, `folio_end_writeback` at `mm/filemap.c:1681` clears the writeback flag:

```c
void folio_end_writeback(struct folio *folio)
{
    VM_BUG_ON_FOLIO(!folio_test_writeback(folio), folio);

    folio_get(folio);                     // prevent free during wakeup
    folio_end_writeback_no_dropbehind(folio);
    folio_end_dropbehind(folio);
    folio_put(folio);
}
```

And inside `folio_end_writeback_no_dropbehind` (line 1655):

```c
if (folio_test_reclaim(folio)) {
    folio_clear_reclaim(folio);
    folio_rotate_reclaimable(folio);  // move to tail of LRU for faster reclaim
}
__folio_end_writeback(folio);          // clear PG_writeback
folio_wake_bit(folio, PG_writeback);   // wake anyone waiting on this page
```

The `folio_wake_bit` call is important — other parts of the kernel (like `fsync()` or truncate) may be sleeping, waiting for writeback to finish. This wakeup lets them proceed.

---

## Part 10: The VFS and Filesystem Layer {#part-10-the-vfs-and-filesystem-layer}

Between "look up page in page cache" and "submit bio to block layer," there is a critical translation layer: the filesystem. This is where file offsets become disk sectors, where metadata is managed, and where the filesystem's on-disk layout affects I/O patterns.

```
Userspace:     read(fd, buf, 4096)
VFS:           vfs_read → generic_file_read_iter
Page cache:    filemap_read → filemap_get_pages
               "is page N of this file in RAM?"
                    │ (cache miss)
Filesystem:    a_ops->readahead → ext4_readahead / f2fs_readahead
               "page N of this file is at disk sector S"
                    │
Block layer:   submit_bio → blk_mq → driver → hardware
```

### The VFS abstraction

The Virtual File System dispatches to filesystem-specific implementations via function pointers. The kernel never calls ext4 or f2fs directly.

```c
struct inode {
    umode_t              i_mode;
    unsigned long        i_ino;
    loff_t               i_size;
    struct super_block  *i_sb;
    struct address_space i_data;     // page cache (embedded, not a pointer)
    struct inode_operations *i_op;   // lookup, create, link
    struct file_operations  *i_fop;  // read, write, mmap
};
```

The `address_space` (Part 4.10) is embedded directly inside `struct inode` as `i_data`. One inode, one page cache.

### File offset → disk sector translation

This is the filesystem's core job for I/O. Given a file and a page offset, where on disk are those bytes?

**ext4: extent tree.** An extent says "file bytes [start, start+len) are stored at disk block B." When the page cache needs page 130, `ext4_map_blocks` walks the extent tree in the inode to find the physical block number, then converts to a sector number for the bio.

**f2fs (default on Android).** Log-structured design optimized for flash. Data blocks and node blocks (metadata) are separate. f2fs writes new data sequentially to clean segments, never overwriting in place. This aligns with flash hardware, which cannot overwrite — it must erase first. The translation goes through a node block that maps file offsets to physical data block addresses.

f2fs has its own garbage collection (GC) thread that reads live blocks from old segments and rewrites them to new segments. This is f2fs-level GC on top of the FTL's own GC. Both compete for flash bandwidth — double GC is a real performance problem on Android.

### address_space_operations

The page cache calls the filesystem through these callbacks:

```c
struct address_space_operations {
    int (*writepages)(struct address_space *, struct writeback_control *);
    int (*readahead)(struct readahead_control *);
    bool (*dirty_folio)(struct address_space *, struct folio *);
    int (*write_begin)(struct file *, struct address_space *, loff_t,
                       unsigned, struct folio **, void **);
    int (*write_end)(struct file *, struct address_space *, loff_t,
                     unsigned, unsigned, struct folio *, void *);
};
```

**readahead** is called by the readahead engine (Part 11). The filesystem translates file offsets to disk sectors, builds bios, and submits them. **writepages** is called by writeback (Part 9) — same translation, opposite direction. **dirty_folio** tags the page in the xarray and marks the inode dirty. **write_begin/write_end** handle the `write()` syscall path — `write_begin` allocates disk blocks if the file is growing, `write_end` marks the page dirty.

### The fsync path

`fsync(fd)` guarantees data and metadata are on stable storage when it returns:

```
fsync(fd)
  → file->f_op->fsync → ext4_sync_file
    → filemap_write_and_wait_range   (write dirty pages, wait for I/O)
    → jbd2_complete_transaction      (wait for journal commit)
    → return 0 or -EIO
```

The `filemap_write_and_wait_range` call is what the tracer captures — EVT_WRITEBACK, EVT_IO_ISSUE, EVT_IO_COMPLETE, EVT_WB_DONE for each dirty page. The journal commit (step 2) is additional metadata I/O that the page tracer does not capture. To trace full fsync latency, hook `ext4_sync_file` or `f2fs_sync_file` directly.

### Direct I/O

`O_DIRECT` bypasses the page cache entirely. Reads and writes go from userspace buffers straight to bios and disk. The tracer will not see direct I/O through page lifecycle events — no EVT_ALLOC, no EVT_CACHE_INSERT. To trace direct I/O, hook `__blockdev_direct_IO` or the iomap direct I/O path.

### The dentry cache and inode cache

Two caches above the page cache:

**Dentry cache (dcache).** Maps (parent dentry, name) → child dentry. Hit rate is typically >99%. Avoids reading directory blocks from disk for every `open()`.

**Inode cache.** Every inode read from disk is cached in `inode_cache` slab. The cached inode contains the file's metadata and the `address_space` (page cache). Evicting an inode from the cache also evicts its page cache pages — the tracer sees a burst of EVT_CACHE_REMOVE events when the inode shrinker runs.

Both caches are reclaimable. `echo 2 > /proc/sys/vm/drop_caches` shrinks both. Shrinking the dcache forces future path lookups to go to disk; shrinking the inode cache forces re-reading metadata and re-populating the page cache.

### Where the tracer hooks into the filesystem

The page lifecycle tracer captures the filesystem's interaction implicitly:

```
Filesystem call          Tracer event           How
──────────────────────────────────────────────────────────────
a_ops->readahead         EVT_ALLOC              page allocated for readahead
  → filemap_add_folio    EVT_CACHE_INSERT       page enters cache
  → bio_add_folio        EVT_READ               bio built with page + sector
  → submit_bio           EVT_IO_ISSUE           dispatched to hardware
  → folio_end_read       EVT_READ_DONE          data arrived

a_ops->writepages        EVT_WRITEBACK          bio built for dirty page
  → submit_bio           EVT_IO_ISSUE
  → folio_end_writeback  EVT_WB_DONE

a_ops->dirty_folio       EVT_DIRTY              page marked dirty
```

To trace filesystem-specific operations (journal commits, f2fs GC, extent allocation), add kprobes on fs-specific functions. These would be additional events, separate from the page lifecycle but correlatable by timestamp and tgid. Part 25 covers f2fs, ext4, and erofs internals — the NAT, extent trees, log-structured writes, compression, GC, and crash consistency mechanisms.

---
## Part 11: The Block I/O Layer — From Pages to Sectors {#part-11-the-block-io-layer-from-pages-to-sectors}

The block layer sits between the filesystem and the disk driver. Its job is to take `bio` structs from the filesystem and deliver them to the hardware efficiently. This section traces the full read and write paths from userspace syscalls down to hardware and back, because you cannot build a useful tracer without understanding both directions.

### The read path: from `read()` to a physical page

When userspace calls `read(fd, buf, len)` on a regular file, the VFS routes it through the filesystem's `->read_iter` op, which for most filesystems is `generic_file_read_iter`. That calls `filemap_read`, the core page cache reader at `mm/filemap.c:2768`.

The work of finding pages (either in cache or by faulting them in from disk) is delegated to `filemap_get_pages` at `mm/filemap.c:2667`:

```c
static int filemap_get_pages(struct kiocb *iocb, size_t count,
        struct folio_batch *fbatch, bool need_uptodate)
{
    struct address_space *mapping = filp->f_mapping;
    pgoff_t index = iocb->ki_pos >> PAGE_SHIFT;
    pgoff_t last_index = round_up(iocb->ki_pos + count, ...) >> PAGE_SHIFT;
retry:
    // ── Step 1: try page cache lookup ──
    filemap_get_read_batch(mapping, index, last_index - 1, fbatch);

    if (!folio_batch_count(fbatch)) {
        // Cache miss — no folios found in the requested range.
        // Fire up synchronous readahead to read from disk.
        DEFINE_READAHEAD(ractl, filp, &filp->f_ra, mapping, index);
        page_cache_sync_ra(&ractl, last_index - index);

        // ── Step 2: retry cache lookup after readahead ──
        filemap_get_read_batch(mapping, index, last_index - 1, fbatch);
    }

    // ── Step 3: check if we hit the readahead marker ──
    folio = fbatch->folios[folio_batch_count(fbatch) - 1];
    if (folio_test_readahead(folio)) {
        // This folio was the lookahead marker — trigger async readahead
        // so the NEXT window is pre-fetched while we process this one.
        filemap_readahead(iocb, filp, mapping, folio, last_index);
    }

    // ── Step 4: if the folio is in cache but not up-to-date, wait for it ──
    if (!folio_test_uptodate(folio)) {
        err = filemap_update_page(iocb, mapping, count, folio, need_uptodate);
    }

    trace_mm_filemap_get_pages(mapping, index, last_index - 1);
    return 0;
}
```

This is a three-phase dance: cache lookup → readahead → wait. The interesting part is step 3 — the **readahead marker**. When readahead fetches a window of pages, it marks *one* of them (the last page of the "readahead-only" portion) with `PG_readahead`. When userspace reads that specific page, it triggers `filemap_readahead`, which kicks off the *next* async batch. This way, readahead stays one window ahead of the reader without blocking it.

### Readahead: the prefetch engine

`page_cache_sync_ra` at `mm/readahead.c:557` is where the heuristics live:

```c
void page_cache_sync_ra(struct readahead_control *ractl, unsigned long req_count)
{
    pgoff_t index = readahead_index(ractl);
    struct file_ra_state *ra = ractl->ra;

    // Detect sequential vs random access patterns:
    //   prev_pos = last read position (per-file, in f_ra)
    //   index    = current read position
    prev_index = (unsigned long long)ra->prev_pos >> PAGE_SHIFT;

    if (!index || req_count > max_pages || index - prev_index <= 1UL) {
        // Start of file, oversized read, or sequential access:
        // set up a fresh readahead window.
        ra->start = index;
        ra->size = get_init_ra_size(req_count, max_pages);
        ra->async_size = ra->size > req_count ? ra->size - req_count
                                              : ra->size >> 1;
        goto readit;
    }

    // Not clearly sequential — check how much contiguous history we have
    // in the page cache. If the walk behind the current index finds many
    // cached pages, we are probably in a long-running sequential stream.
    miss = page_cache_prev_miss(ractl->mapping, index - 1, max_pages);
    contig_count = index - miss - 1;

    if (contig_count <= req_count) {
        // Looks random. Read just what was asked, no speculation.
        do_page_cache_ra(ractl, req_count, 0);
        return;
    }

    // Sustained sequential access — ramp up the window
    ra->start = index;
    ra->size = min(contig_count + req_count, max_pages);
    ra->async_size = 1;
readit:
    page_cache_ra_order(ractl, ra);
}
```

The kernel tracks per-file readahead state in `struct file_ra_state` (embedded in `struct file` as `f_ra`). It contains:
- `ra_pages`: maximum window size (set from the backing device's advice, typically 128KB for UFS).
- `start`, `size`: the current readahead window.
- `async_size`: how many pages at the end of the window are "async" — when userspace hits them, kick off the next window.
- `prev_pos`: the last read position, used to detect sequential access.

The actual I/O submission happens in `page_cache_ra_unbounded` at `mm/readahead.c:211`:

```c
void page_cache_ra_unbounded(struct readahead_control *ractl,
        unsigned long nr_to_read, unsigned long lookahead_size)
{
    // Tell memory reclaim we're in a filesystem operation.
    // This prevents reclaim from recursing back into this filesystem
    // (which could deadlock if the FS is writing back dirty pages).
    unsigned int nofs = memalloc_nofs_save();

    trace_page_cache_ra_unbounded(mapping->host, index, nr_to_read,
                                  lookahead_size);

    // Preallocate folios, one per page in the window
    while (i < nr_to_read) {
        struct folio *folio = xa_load(&mapping->i_pages, index + i);

        if (folio && !xa_is_value(folio)) {
            // This slot is already cached — submit what we have so far
            // and continue with the next gap.
            read_pages(ractl);
            continue;
        }

        folio = ractl_alloc_folio(ractl, gfp_mask, ...);    // allocate
        ret = filemap_add_folio(mapping, folio, index + i, gfp_mask);  // insert

        if (i == mark)
            folio_set_readahead(folio);   // this is the lookahead marker

        ractl->_nr_pages += min_nrpages;
        i += min_nrpages;
    }

    // Now submit I/O for the batch of newly-allocated folios
    read_pages(ractl);
    memalloc_nofs_restore(nofs);
}
```

`read_pages` calls the filesystem's `a_ops->readahead` callback. For most filesystems, this is `mpage_readahead` at `fs/mpage.c`, which:

1. Iterates the pre-allocated folios from the readahead control.
2. For each folio, calls the filesystem's `get_block` to translate file offset → disk sector.
3. Accumulates contiguous sectors into a single `bio`, extending it across folios when possible.
4. Submits the bio via `mpage_bio_submit_read`:

```c
// fs/mpage.c:71
static struct bio *mpage_bio_submit_read(struct bio *bio)
{
    bio->bi_end_io = mpage_read_end_io;   // completion callback
    guard_bio_eod(bio);                    // clamp to device size
    submit_bio(bio);
    return NULL;
}
```

The key point: **reads are asynchronous from the kernel's perspective**. `submit_bio` hands the bio to the block layer and returns. The reader (in `filemap_update_page`) then calls `folio_wait_locked` to sleep on the folio lock, which the block completion handler releases when the data arrives.

### submit_bio: entering the block layer

`submit_bio` at `block/blk-core.c:916` is the universal entry point:

```c
void submit_bio(struct bio *bio)
{
    if (bio_op(bio) == REQ_OP_READ) {
        task_io_account_read(bio->bi_iter.bi_size);  // per-task I/O accounting
        count_vm_events(PGPGIN, bio_sectors(bio));   // global vm stats
    } else if (bio_op(bio) == REQ_OP_WRITE) {
        count_vm_events(PGPGOUT, bio_sectors(bio));
    }

    bio_set_ioprio(bio);
    submit_bio_noacct(bio);   // the real work
}
```

Inside `submit_bio_noacct`, the block layer performs three critical operations:

**1. Plug list insertion.** The kernel maintains a per-task plug list (`current->plug`) that batches I/O. When a task starts a bulk I/O operation, it calls `blk_start_plug` to begin batching; all `submit_bio` calls in between are added to the plug list. When the task calls `blk_finish_plug` (or sleeps), the whole batch is flushed to the block layer at once. This reduces the number of times we hit the queue lock.

**2. Bio merging.** Before creating a new request, the block layer tries to merge the bio with pending requests:

```c
// block/blk-merge.c:944
enum bio_merge_status bio_attempt_back_merge(struct request *req,
                                              struct bio *bio,
                                              unsigned int nr_segs)
{
    // Check: do the new bio's sectors immediately follow req's sectors?
    if (req->biotail->bi_iter.bi_sector + bio_sectors(req->biotail)
        != bio->bi_iter.bi_sector)
        return BIO_MERGE_NONE;

    // Chain the new bio onto the request
    req->biotail->bi_next = bio;
    req->biotail = bio;
    req->__data_len += bio->bi_iter.bi_size;

    // Notify QoS and I/O scheduler
    blk_account_io_merge_bio(req);
    return BIO_MERGE_OK;
}
```

Front-merging is the mirror operation for the *start* of a request. Merging turns 100 small 4KB sequential writes into one 400KB write — fewer commands, fewer interrupts, and the hardware processes the contiguous region much more efficiently.

**3. Request dispatch.** Once the bio is wrapped in a request (or merged into one), it goes to the I/O scheduler's per-hardware-queue list. When the hardware queue has capacity, `blk_mq_dispatch_rq_list` at `block/blk-mq.c:2116` hands requests to the driver via `q->mq_ops->queue_rq()`. This is where the `block_rq_issue` tracepoint fires. The driver programs the hardware (UFS, NVMe, etc.) and the hardware does the actual work.

### The completion path: from hardware back to userspace

When the hardware finishes, it raises an interrupt. The driver's interrupt handler calls `blk_mq_end_request`, which calls `bio_endio` at `block/bio.c:1749`:

```c
void bio_endio(struct bio *bio)
{
again:
    if (!bio_remaining_done(bio))    // split bio — wait for all parts
        return;
    if (!bio_integrity_endio(bio))
        return;

    blk_zone_bio_endio(bio);
    rq_qos_done_bio(bio);

    if (bio->bi_bdev && bio_flagged(bio, BIO_TRACE_COMPLETION)) {
        trace_block_bio_complete(bdev_get_queue(bio->bi_bdev), bio);
    }

    // Handle chained bios without blowing the stack
    if (bio->bi_end_io == bio_chain_endio) {
        bio = __bio_chain_endio(bio);
        goto again;
    }

    if (bio->bi_end_io)
        bio->bi_end_io(bio);    // filesystem's completion callback
}
```

For a read, `bi_end_io` is `mpage_read_end_io` at `fs/mpage.c:46`:

```c
static void mpage_read_end_io(struct bio *bio)
{
    struct folio_iter fi;
    int err = blk_status_to_errno(bio->bi_status);

    bio_for_each_folio_all(fi, bio)
        folio_end_read(fi.folio, err == 0);   // wake up waiters per folio

    bio_put(bio);    // return bio to the pool
}
```

And `folio_end_read` at `mm/filemap.c:1529` is what actually wakes the sleeping reader:

```c
void folio_end_read(struct folio *folio, bool success)
{
    unsigned long mask = 1 << PG_locked;

    if (likely(success))
        mask |= 1 << PG_uptodate;
    // Atomically clear PG_locked and (if success) set PG_uptodate
    // in a single operation. Check if anyone was waiting on PG_locked.
    if (folio_xor_flags_has_waiters(folio, mask))
        folio_wake_bit(folio, PG_locked);
}
```

This is the handoff: the I/O is done, the page is up-to-date, and any task sleeping on `folio_wait_locked` for this page gets woken up. Control returns to the filemap read path, which copies the data to userspace with `copy_folio_to_iter`.

### The write path

Writeback is the mirror of the read path, but with different triggers. Dirty pages get written back from three sources:

1. **Background writeback** (the `kworker/flush-*` thread) runs periodically based on `dirty_writeback_centisecs` (default 5 seconds) and picks up pages dirtier than `dirty_expire_centisecs` (default 30 seconds).
2. **Explicit sync** — `fsync()`, `fdatasync()`, or `sync()` calls `filemap_write_and_wait_range` which immediately initiates writeback.
3. **Memory pressure** — during reclaim, `shrink_folio_list` may call `pageout()` to write dirty anon pages to swap.

All three paths converge at `do_writepages` in `mm/page-writeback.c:2564`:

```c
int do_writepages(struct address_space *mapping, struct writeback_control *wbc)
{
    if (wbc->nr_to_write <= 0)
        return 0;
    wb = inode_to_wb_wbc(mapping->host, wbc);
    wb_bandwidth_estimate_start(wb);
    while (1) {
        if (mapping->a_ops->writepages)
            ret = mapping->a_ops->writepages(mapping, wbc);   // the FS handler
        else
            ret = 0;
        if (ret != -ENOMEM || wbc->sync_mode != WB_SYNC_ALL)
            break;
        // OOM during sync writeback — throttle and retry
        reclaim_throttle(NODE_DATA(numa_node_id()),
                         VMSCAN_THROTTLE_WRITEBACK);
    }
    return ret;
}
```

The filesystem's `writepages` callback is the dual of `readahead`. For most filesystems it's `mpage_writepages`, which:

1. Iterates dirty pages in the address_space using the `PAGECACHE_TAG_DIRTY` xarray mark (fast — no scanning clean pages).
2. For each dirty page, looks up the disk sector via `get_block`.
3. Accumulates contiguous sectors into a bio.
4. Sets `PG_writeback` on the folio (so reclaim knows not to evict it).
5. Submits the bio via `mpage_bio_submit_write`:

```c
// fs/mpage.c:79
static struct bio *mpage_bio_submit_write(struct bio *bio)
{
    bio->bi_end_io = mpage_write_end_io;
    guard_bio_eod(bio);
    submit_bio(bio);
    return NULL;
}
```

When the write I/O completes, `mpage_write_end_io` at `fs/mpage.c:57` runs:

```c
static void mpage_write_end_io(struct bio *bio)
{
    struct folio_iter fi;
    int err = blk_status_to_errno(bio->bi_status);

    bio_for_each_folio_all(fi, bio) {
        if (err)
            mapping_set_error(fi.folio->mapping, err);  // propagate to fsync
        folio_end_writeback(fi.folio);                  // clear PG_writeback
    }
    bio_put(bio);
}
```

The `folio_end_writeback` call (Part 9 traced this at `mm/filemap.c:1681`) clears `PG_writeback` and wakes anyone sleeping in `folio_wait_writeback` — typically an `fsync()` caller waiting for their data to hit disk.

### The full read/write cycle, visualized

```
READ PATH:
  read(fd) syscall
    → filemap_read              mm/filemap.c:2768
      → filemap_get_pages       mm/filemap.c:2667
        → filemap_get_read_batch    [cache lookup via xarray]
        → page_cache_sync_ra    mm/readahead.c:557
          → page_cache_ra_unbounded    [allocate folios, insert into cache]
            → read_pages → mpage_readahead [build bio]
              → mpage_bio_submit_read
                → submit_bio        block/blk-core.c:916
                  → submit_bio_noacct
                    → [plug/merge/scheduler/dispatch]
                      → driver queue_rq()  → [HARDWARE]

  [HARDWARE INTERRUPT]
    → driver IRQ handler
      → blk_mq_end_request
        → bio_endio               block/bio.c:1749
          → mpage_read_end_io     fs/mpage.c:46
            → folio_end_read      mm/filemap.c:1529
              → folio_wake_bit(PG_locked)
                [reader task wakes up]
  → copy_folio_to_iter → userspace buffer

WRITE PATH:
  fsync(fd) syscall  (or writeback timer, or reclaim)
    → filemap_write_and_wait_range  mm/filemap.c:675
      → filemap_fdatawrite_range
        → do_writepages           mm/page-writeback.c:2564
          → a_ops->writepages → mpage_writepages
            [iterate PAGECACHE_TAG_DIRTY, set PG_writeback]
            → mpage_bio_submit_write
              → submit_bio → [same block layer path as read]

  [HARDWARE INTERRUPT]
    → bio_endio → mpage_write_end_io  fs/mpage.c:57
      → folio_end_writeback           mm/filemap.c:1681
        → clear PG_writeback, wake fsync() waiters
```

### How this interacts with memory

Two points are worth calling out for the tracer:

**Reads allocate pages.** When the cache misses, readahead calls `filemap_add_folio`, which goes through the page allocator (`mm_page_alloc` tracepoint fires) and the page cache insertion path (`mm_filemap_add_to_page_cache` tracepoint fires). From the tracer's perspective, a cache-miss read produces the same allocation events as any other page. The difference is that the page starts life as the *destination* of a bio rather than being written to by userspace.

**Writes do not allocate.** Writeback operates on pages that already exist in the page cache. It sets `PG_writeback` on an existing folio, submits a bio pointing to that folio's page(s), and clears `PG_writeback` on completion. No allocation happens. The tracer sees the page's EVT_DIRTY event earlier (when userspace wrote to it) and the EVT_WB_DONE event later (when the data hits disk).

**The block_rq_issue/complete gap is hardware latency.** Between `block_rq_issue` and `block_rq_complete`, the page is sitting in a disk command queue, being processed by the flash translation layer, and finally being written to flash cells. For UFS on a phone, this is typically 100-500µs for writes. Spikes to 5-10ms mean the FTL is doing internal garbage collection or wear-leveling — often the biggest source of tail latency on Android.

### Block layer tracepoints for the tracer

```c
// block/blk-mq.c:1372 — when a request is sent to the driver
trace_block_rq_issue(rq);

// block/blk-mq.c:891/961 — when the driver reports completion
trace_block_rq_complete(req, BLK_STS_OK, total_bytes);
```

The challenge for our tracer: both tracepoints carry sector information (dev, sector, nr_sectors) but not PFN. To connect an I/O event back to a page, we need to record the (dev, sector) → page_id mapping *at bio construction time* (when we still have the folio). Part 18 shows exactly how.

---

## Part 12: Reclaim — How the Kernel Takes Memory Back {#part-12-reclaim-how-the-kernel-takes-memory-back}

When the system is running low on free memory, the kernel needs to reclaim pages. It does this through a background daemon called `kswapd` (one per NUMA node) and through **direct reclaim** in the allocation path when `kswapd` cannot keep up.

### The LRU lists

The kernel maintains separate LRU (Least Recently Used) lists for different page types:

```c
// include/linux/mmzone.h:316
enum lru_list {
    LRU_INACTIVE_ANON  = 0,  // cold anonymous pages
    LRU_ACTIVE_ANON    = 1,  // hot anonymous pages
    LRU_INACTIVE_FILE  = 2,  // cold file pages
    LRU_ACTIVE_FILE    = 3,  // hot file pages
    LRU_UNEVICTABLE,         // mlocked pages, cannot be reclaimed
    NR_LRU_LISTS
};
```

Pages start on the inactive list. When they are accessed again, they get promoted to the active list. The reclaim code scans the inactive list, looking for pages to evict. If it runs out of inactive pages, it demotes pages from the active list to the inactive list.

This two-list scheme approximates a true LRU without the overhead of reordering a list on every page access. It is based on the **2Q algorithm** — the idea is that a page needs to be accessed twice (once to get on the active list, once to stay there) before the kernel considers it "hot."

Recent kernels also support **MGLRU** (Multi-Generation LRU), which uses multiple generations instead of just active/inactive. MGLRU is the default on Android.

### The reclaim decision tree

`shrink_folio_list` at `mm/vmscan.c:1083` is where the kernel decides what to do with each candidate page. It is a long function (500+ lines), but the logic follows a clear decision tree:

```c
static unsigned int shrink_folio_list(struct list_head *folio_list, ...)
{
    while (!list_empty(folio_list)) {
        folio = lru_to_folio(folio_list);

        if (!folio_trylock(folio))
            goto keep;                    // can't lock → skip for now

        // Check dirty/writeback state
        folio_check_dirty_writeback(folio, &dirty, &writeback);

        if (writeback) {
            // Page is currently being written to disk.
            // Don't evict it — it will be clean soon.
            goto activate_locked;
        }
```

Then, for anonymous pages, the kernel needs to find somewhere to put the data before it can free the page:

```c
        // Anonymous page without swap space?
        if (folio_test_anon(folio) && folio_test_swapbacked(folio) &&
                !folio_test_swapcache(folio)) {
            if (folio_alloc_swap(folio))   // try to allocate a swap slot
                goto activate_locked;      // no swap space → cannot reclaim
        }
```

Next, the kernel unmaps the page from all processes:

```c
        if (folio_mapped(folio)) {
            try_to_unmap(folio, TTU_BATCH_FLUSH | TTU_SYNC);
            if (folio_mapped(folio))
                goto activate_locked;      // still mapped → someone re-faulted
        }
```

`try_to_unmap` uses the reverse mapping (`rmap`) to find all PTEs pointing to this page and clear them. This is why the kernel maintains `anon_vma` (for anonymous pages) and `address_space->i_mmap` (for file pages) — without these reverse mappings, the kernel would have to scan every page table in the system to find all references to a page.

For dirty pages, the kernel writes them back first:

```c
        if (folio_test_dirty(folio)) {
            if (folio_is_file_lru(folio))
                // File page: mark for writeback, let background thread handle it
                goto activate_locked;      // will be written back by kworker
            else
                // Anonymous page: write to swap/zram
                pageout(folio);            // triggers swap_writeout()
        }
```

Finally, for clean, unmapped pages, the kernel removes them from the page cache and frees the physical page:

```c
        // Remove from page cache / swap cache
        if (__remove_mapping(mapping, folio, ...)) {
            // Page is free! Add to batch for bulk freeing.
            free_folio_batch_add(&free_folios, folio);
            nr_reclaimed += nr_pages;
        }
```

### The buddy allocator

When a page is finally freed, it returns to the **buddy allocator** — the kernel's physical page allocator. The buddy system organizes free pages by powers of two: order-0 (4KB), order-1 (8KB), up to order-10 (4MB). When you free a 4KB page, the buddy allocator checks if its "buddy" (the adjacent 4KB page) is also free. If so, it merges them into an 8KB block, then checks if THAT block's buddy is free, and so on. This coalescing reduces fragmentation.

Allocation works in reverse: if you ask for 4KB and the order-0 list is empty, the allocator splits an 8KB block (or 16KB, or 32KB...) until it has a 4KB chunk to give you.

The actual free path is `__free_one_page` at `mm/page_alloc.c:978`:

```c
static inline void __free_one_page(struct page *page,
        unsigned long pfn, struct zone *zone, unsigned int order,
        int migratetype, fpi_t fpi_flags)
{
    unsigned long buddy_pfn = 0;
    unsigned long combined_pfn;
    struct page *buddy;

    VM_BUG_ON_PAGE(pfn & ((1 << order) - 1), page);  // pfn must be order-aligned

    account_freepages(zone, 1 << order, migratetype);

    // Try to merge with buddies, moving up orders
    while (order < MAX_PAGE_ORDER) {
        int buddy_mt = migratetype;

        // If compaction is capturing this page for a high-order allocation,
        // hand it over and stop merging.
        if (compaction_capture(capc, page, order, migratetype)) {
            account_freepages(zone, -(1 << order), migratetype);
            return;
        }

        // Find our buddy. The buddy is found by XOR-ing the PFN with
        // (1 << order) — this flips the bit that distinguishes the two
        // halves of a block at this order.
        buddy = find_buddy_page_pfn(page, pfn, order, &buddy_pfn);
        if (!buddy)
            goto done_merging;          // buddy is in use, cannot merge

        // Buddy is free — remove it from the freelist and merge
        __del_page_from_free_list(buddy, zone, order, buddy_mt);

        // The merged block starts at the lower of the two PFNs
        combined_pfn = buddy_pfn & pfn;
        page = page + (combined_pfn - pfn);
        pfn = combined_pfn;
        order++;
    }

done_merging:
    set_buddy_order(page, order);
    __add_to_free_list(page, zone, order, migratetype, to_tail);
}
```

The buddy-finding trick is pure arithmetic. Two buddies at order `n` have PFNs that differ only in bit `n`: `0...0 XXXX 0 YYYY` and `0...0 XXXX 1 YYYY`, where `YYYY` has `n` bits. `find_buddy_page_pfn` computes `buddy_pfn = pfn ^ (1 << order)` to get the buddy, and the merged block's PFN is `pfn & buddy_pfn` (the lower of the two). No pointer chasing — just arithmetic.

The tradeoff of the buddy system: you can get fragmentation. If the kernel has thousands of free order-0 pages scattered across memory but none of them have free buddies (because their buddies are in use), you cannot satisfy a request for an order-4 (64KB) allocation even though the total free memory is enough. This is why memory compaction exists — it physically moves pages around to create contiguous free regions.

The GFP flags passed to allocation functions control what the allocator is allowed to do:

```c
// include/linux/gfp_types.h:376
#define GFP_ATOMIC  (__GFP_HIGH|__GFP_KSWAPD_RECLAIM)    // can't sleep, emergency pool
#define GFP_KERNEL  (__GFP_RECLAIM | __GFP_IO | __GFP_FS) // normal kernel allocation
#define GFP_HIGHUSER_MOVABLE  (GFP_HIGHUSER | __GFP_MOVABLE | __GFP_SKIP_KASAN)
                                                           // user pages (can be migrated)
```

`GFP_HIGHUSER_MOVABLE` is the flag used for user-facing pages — page cache and anonymous pages. The `__GFP_MOVABLE` flag tells the allocator that this page can be physically moved later (via page migration), which is essential for memory compaction and defragmentation.

The allocation tracepoint fires here:

```c
// mm/page_alloc.c:5272
trace_mm_page_alloc(page, order, alloc_gfp, ac.migratetype);
```

And the free tracepoint here:

```c
// mm/page_alloc.c:1353
trace_mm_page_free(page, order);
```

---

## Part 13: zRAM — Android's Swap Trick {#part-13-zram-androids-swap-trick}

Traditional desktop Linux uses swap partitions — when anonymous pages are evicted, they are written to a dedicated area on disk. Android does not do this. Writing to flash storage is slow (relative to RAM), wears out the flash, and modern phones have plenty of RAM to compress into.

Instead, Android uses **zRAM**: a block device backed by compressed RAM. When the kernel evicts an anonymous page to "swap," zRAM compresses it and stores the compressed data in a pool of regular RAM. A 4KB page might compress to 1KB, effectively quadrupling your usable memory.

`zram_write_page` at `drivers/block/zram/zram_drv.c:2241`:

```c
static int zram_write_page(struct zram *zram, struct page *page, u32 index)
{
    // 1. Check if the entire page is filled with one repeated value
    mem = kmap_local_page(page);
    same_filled = page_same_filled(mem, &element);
    kunmap_local(mem);
    if (same_filled)
        return write_same_filled_page(zram, element, index);  // store just the value!

    // 2. Compress the page
    zstrm = zcomp_stream_get(zram->comps[ZRAM_PRIMARY_COMP]);
    mem = kmap_local_page(page);
    ret = zcomp_compress(zram->comps[ZRAM_PRIMARY_COMP], zstrm,
                         mem, &comp_len);
    kunmap_local(mem);

    // 3. If compressed size is too big, store uncompressed
    if (comp_len >= huge_class_size) {
        zcomp_stream_put(zstrm);
        return write_incompressible_page(zram, page, index);
    }

    // 4. Allocate compressed storage and copy compressed data
    handle = zs_malloc(zram->mem_pool, comp_len,
                       GFP_NOIO | __GFP_NOWARN | __GFP_HIGHMEM | __GFP_MOVABLE,
                       page_to_nid(page));
    zs_obj_write(zram->mem_pool, handle, zstrm->buffer, comp_len);
    zcomp_stream_put(zstrm);

    // 5. Update accounting
    atomic64_inc(&zram->stats.pages_stored);
    atomic64_add(comp_len, &zram->stats.compr_data_size);
    return 0;
}
```

There are three fast paths here:

1. **Same-filled page**: If the entire 4KB page is filled with the same byte pattern (common for zero-filled or newly allocated pages), zRAM stores just that one value — essentially free.
2. **Compressible page**: The page compresses to less than `huge_class_size` (typically 75% of page size). This is the normal case for text, DOM data, Java objects.
3. **Incompressible page**: Media buffers, encrypted data, already-compressed data. Stored uncompressed, which is wasteful. This is why apps with lots of media tend to be poor zRAM citizens.

The compression ratio (total compressed size / total original size) is the key metric. On a typical Android phone, it is around 0.3–0.4, meaning you get 2.5–3× effective RAM capacity from zRAM.

### The read path

When the page is needed again (a swap fault), the kernel asks zRAM to bring it back. `read_from_zspool` at `drivers/block/zram/zram_drv.c:2118` dispatches based on how the page was stored:

```c
static int read_from_zspool(struct zram *zram, struct page *page, u32 index)
{
    if (test_slot_flag(zram, index, ZRAM_SAME) ||
        !get_slot_handle(zram, index))
        return read_same_filled_page(zram, page, index);

    if (!test_slot_flag(zram, index, ZRAM_HUGE))
        return read_compressed_page(zram, page, index);
    else
        return read_incompressible_page(zram, page, index);
}
```

The three cases mirror the three write cases:

- **ZRAM_SAME**: the page was stored as a single repeated value. Just fill the new page with that value. Essentially free.
- **Compressed (default)**: decompress into the new page via `read_compressed_page` at `drivers/block/zram/zram_drv.c:2063`.
- **ZRAM_HUGE (incompressible)**: copy the stored raw bytes directly.

The compressed case does the real work:

```c
static int read_compressed_page(struct zram *zram, struct page *page, u32 index)
{
    handle = get_slot_handle(zram, index);
    size = get_slot_size(zram, index);
    prio = get_slot_comp_priority(zram, index);

    zstrm = zcomp_stream_get(zram->comps[prio]);
    src = zs_obj_read_begin(zram->mem_pool, handle, size, zstrm->local_copy);
    dst = kmap_local_page(page);
    ret = zcomp_decompress(zram->comps[prio], zstrm, src, size, dst);
    kunmap_local(dst);
    zs_obj_read_end(zram->mem_pool, handle, size, src);
    zcomp_stream_put(zstrm);

    return ret;
}
```

Notice `prio = get_slot_comp_priority(zram, index)`. zRAM can be configured with multiple compression backends simultaneously (e.g., lz4 as primary for speed, zstd as secondary for ratio). The slot records which backend compressed it, so decompression picks the right codec.

### Compression backends

zRAM supports multiple codecs via `drivers/block/zram/backend_*.c`:

- **lz4**: very fast, modest compression ratio. Default for speed-sensitive workloads.
- **lz4hc**: slower compression, same decompression speed, better ratio.
- **zstd**: significantly better ratio than lz4, slower compression but still fast decompression.
- **deflate**: legacy.
- **842**: IBM's hardware-accelerated codec (POWER).
- **lzo / lzo-rle**: older; still in use.

Each backend registers a `zcomp_backend` struct exposing `compress`, `decompress`, and stream-allocation callbacks. The backend selection is a sysfs knob (`/sys/block/zramN/comp_algorithm`).

### zsmalloc: the compressed page allocator

Compressed data doesn't fit into page-size chunks — compressed sizes vary from ~100 bytes to ~4KB. zRAM uses **zsmalloc**, a special slab-like allocator optimized for variable-size objects within a compressed RAM pool.

`zs_malloc(zram->mem_pool, comp_len, ...)` returns a 64-bit **handle** (not a pointer). Handles are stable even if zsmalloc internally moves objects around to defragment. Given a handle, `zs_obj_read_begin` and `zs_obj_write` copy data in and out.

This is why the zRAM tracer cannot naturally follow a page's PFN: once compressed, the data is at a zsmalloc handle, not a PFN. zsmalloc may also split the compressed data across two physical pages (because object sizes aren't aligned to page boundaries), which is why `read_from_zspool_raw` passes `zstrm->local_copy` as a scratch buffer.

### The LLM (Least Recently Mapped) choice

Traditional Linux swap writes evicted pages to a disk block; any subsequent read requires a disk read. zRAM's "swap" is all in RAM, so there's no wear-leveling or I/O wait — just a CPU decompression cost measured in microseconds.

On Android, zRAM is typically sized at 50–200% of physical RAM (the `disksize` parameter). A phone with 8GB RAM might configure 4GB of zRAM. Under pressure, the kernel evicts anonymous pages to zRAM, and the 4GB of compressed pool holds ~12GB of logical page data (at a 0.33 compression ratio). Effective memory: 8GB RAM + 12GB compressed = 20GB worth of working set.

### zRAM writeback

Since kernel 4.14, zRAM supports **writeback** — evicting *compressed* pages to a real disk-backed swap device when the compressed pool itself fills up. Configured via `/sys/block/zramN/backing_dev`. The idea: compressed pages in zRAM are "warm" (recently accessed); if not touched for a while, they get written out to flash. This lets zRAM act as a compression-layer cache in front of traditional swap.

For Android, this is generally disabled (flash wear) but available on servers.

### Why zRAM is hard to trace precisely

For the tracer in Part 18, zRAM is the hardest subsystem to instrument per-page. The flow is:

```
anonymous page (PFN=X) → reclaim → swap_writeout → zram_bvec_write
  → zcomp_compress → zs_malloc(pool) → handle=H
```

After compression, the original PFN is freed. The logical page exists as `(swap_entry, handle H, size S)`. When swap-in happens, a *new* PFN is allocated and `zcomp_decompress` fills it. The old PFN and new PFN have no direct relationship — they are connected only through the swap entry.

So the tracer can observe "page PFN=X was handed to zRAM" (via a `zram_write_page` kprobe) and separately "page PFN=Y came from zRAM" (via a decompression kprobe), but linking them requires tracking the swap entry through the `struct folio`'s `->swap` field during the zram trip. Part 18 discusses this limitation.

---

## Part 14: Page Migration — Moving Pages Without Anyone Noticing {#part-14-page-migration-moving-pages-without-anyone-noticing}

Sometimes the kernel must physically relocate a page — copy 4096 bytes from one PFN to another — without any process noticing. Three scenarios:

- **Compaction:** the buddy allocator needs contiguous physical pages for large allocations (huge pages, DMA buffers). Free pages are scattered. Migration moves in-use pages out of the way to create contiguous free regions.
- **NUMA balancing:** on multi-socket servers, a page allocated on node 0 but accessed by CPUs on node 1 should be migrated closer. Drops latency from ~100ns to ~70ns.
- **Memory hotplug:** before physically removing a RAM module on servers, all pages on it must be migrated elsewhere.

### The GFP_MOVABLE constraint

Not all pages can be migrated:

```
__GFP_MOVABLE:     user pages (page cache, anonymous)
                   Can be migrated — kernel can find and update all PTEs
                   via reverse mapping (i_mmap or anon_vma)

Not movable:       kernel pages (page tables, slab caches, kernel stacks)
                   Cannot be migrated — raw kernel pointers exist everywhere
                   that the kernel cannot find or update
```

The buddy allocator places movable pages in `MIGRATE_MOVABLE` freelists. Compaction only relocates these pages. This is why fragmentation is solvable for user allocations (the kernel can always move them) but fundamentally hard for kernel allocations (no reverse mapping for raw pointers).

### The migration sequence

Moving a page from PFN 3000 to PFN 8000, mapped by processes A and B:

**Step 1:** Allocate destination page (PFN 8000).

**Step 2:** Lock the source folio.

**Step 3:** Replace all PTEs with migration entries:

```
Before: Process A PTE: [PFN 3000 | present]
        Process B PTE: [PFN 3000 | present]

After:  Process A PTE: [PFN 3000 | migration | not present]
        Process B PTE: [PFN 3000 | migration | not present]
```

A migration entry is a special non-present PTE encoding. If a process accesses this address, the MMU faults, the fault handler sees the migration entry, calls `migration_entry_wait`, and the process sleeps until migration completes. The process experiences a ~10–50µs stall.

**Step 4:** Copy 4096 bytes from PFN 3000 to PFN 8000. Takes ~1µs (memcpy at memory bandwidth).

**Step 5:** Update destination folio's metadata — `mapping`, `index`, and flags are transferred from source.

**Step 6:** If file-backed: update page cache xarray so `i_pages[index]` now points to PFN 8000's folio.

**Step 7:** Replace migration entries with real PTEs pointing to PFN 8000. Wake any sleeping processes.

**Step 8:** Free PFN 3000 to buddy allocator.

No process noticed. No data changed. Only the physical location moved.

### Why migration entries instead of just locking the folio

The folio lock is a software construct — the MMU is hardware and doesn't check folio locks. If the PTE says "present and readable," the CPU reads directly through its TLB. While CPU 0 copies data to the new PFN, CPU 1 could still read through a valid PTE pointing to the old PFN. After the copy, when PFN 3000 is freed and reallocated for something else, CPU 1's cached TLB entry points to someone else's data.

Migration entries solve this by making the PTE non-present at the hardware level. The MMU faults, the kernel sees the migration flag, and the process sleeps. No access to the page during migration is possible through any TLB on any CPU.

### Compaction: migration in bulk

Two scanners work from opposite ends of a zone's memory:

```
[used][free][used][free][used][free][used][free]
  ↑ migrate scanner                    ↑ free scanner
  (finds movable pages)        (finds free slots)
```

The migrate scanner picks movable pages from the low end; the free scanner picks destinations from the high end. Pages are migrated until the scanners meet. The vacated low region becomes one contiguous free block, available for high-order allocations.

### The migrate_folio_move code

`migrate_folio_move` at `mm/migrate.c:1353`:

```c
static int migrate_folio_move(free_folio_t put_new_folio, unsigned long private,
                              struct folio *src, struct folio *dst,
                              enum migrate_mode mode, enum migrate_reason reason,
                              struct list_head *ret)
{
    // 1. Copy the page contents
    rc = move_to_new_folio(dst, src, mode);
    if (rc) goto out;

    // 2. Add new page to LRU
    folio_add_lru(dst);

    // 3. Update all PTEs: old PFN → new PFN
    if (old_page_state & PAGE_WAS_MAPPED)
        remove_migration_ptes(src, dst, 0);

    // 4. Unlock and release
    folio_unlock(dst);
    folio_put(dst);

    // The old page is now unreferenced and will be freed
    folio_unlock(src);
    migrate_folio_done(src, reason);
    return rc;
}
```

The `remove_migration_ptes` call is the tricky part. During migration:

1. The kernel first replaces all PTEs pointing to the old page with **migration entries** — special PTE values that tell the fault handler "this page is being moved, wait."
2. It copies the data from old to new physical page.
3. It replaces the migration entries with PTEs pointing to the new physical address.

If a process tries to access the page during step 2, it faults. The fault handler sees the migration entry, sleeps until migration completes, then retries. The process never knows its page moved — it just experiences a brief stall.

**Why migration matters for tracing.** If you are tracking pages by PFN, migration breaks your tracking. The PFN changes, but it is logically the same page with the same data and the same owner. Any tracer that uses PFN as a key needs to handle migration or it will lose pages. This is one of the harder problems in building a page lifecycle tracer, and we address it in Part 18.

### Migration entries: the bridge state

Between "old PTE cleared" and "new PTE installed," the PTE is neither absent nor pointing to a real page. It holds a **migration entry** — a special encoded value that tells the fault handler "this page is mid-move, go to sleep."

From the PTE's bit layout (Part 3.4), when bit 0 is 0 (invalid), the kernel reuses the remaining 63 bits for auxiliary data. Migration entries are one such encoding: they record the old PFN and the "this is a migration" type, packed into the invalid PTE slot.

When a fault occurs on a migration entry:

1. `handle_pte_fault` sees `!pte_present(pte)` and calls `do_swap_page` — counterintuitively, because the migration code path is implemented as part of the swap infrastructure.
2. `do_swap_page` recognizes the migration entry type, calls `migration_entry_wait`, which puts the task to sleep on the folio's migration waitqueue.
3. When `remove_migration_ptes` finishes installing the new PTEs, it wakes all waiters.
4. The woken task retries the fault and sees the new PTE.

This is how user processes experience migration as "brief stall" rather than "segfault."

### Compaction: the most common trigger

Memory compaction is the biggest user of migration. The kernel compacts memory to build contiguous free regions for high-order allocations (huge pages, CMA for device buffers, large bio allocations).

`mm/compaction.c` has the compaction loop. The algorithm:

1. Scan memory for movable pages, isolate them.
2. Migrate each isolated page to a free slot in a contiguous target region.
3. The vacated original region becomes one continuous free block.

Compaction runs in three modes:

- **Synchronous** (from `__alloc_pages_direct_compact` in the allocator): triggered when an allocation cannot be satisfied. The allocator itself blocks on compaction. Used for critical high-order allocations.
- **Asynchronous** (also from the allocator): lighter-weight, gives up more easily, runs on the allocating task's CPU time.
- **kcompactd**: a per-node background daemon that compacts proactively to keep a reserve of free high-order blocks available.

The tracepoints `compaction_isolate_migratepages`, `compaction_migratepages`, and `compaction_finished` let you observe compaction activity in real time.

### NUMA balancing

On multi-socket servers (rare on phones), the kernel can migrate pages to follow the CPUs that access them. This is driven by **protection faults** — periodically, the kernel temporarily marks a sample of PTEs as "no-access" (via `pte_protnone`). The next access faults into `do_numa_page`, which records which NUMA node touched the page. If the pattern shows the page is mostly accessed from a different node, the scheduler migrates it.

The `mm_numa_migrate_ratelimit` tracepoint counts NUMA migrations throttled by rate limits. On NUMA servers, excessive NUMA migrations can actually hurt performance — migration has a cost, and oscillating pages get worse performance than just accepting the original placement.

### Memory hotplug

On cloud servers and large systems, physical memory can be added or removed at runtime. Removing memory requires migrating all pages out of the affected region. This path calls `migrate_pages` with `reason=MR_MEMORY_HOTPLUG`. Rare on phones; present in the kernel for completeness.

### The migrate_pages top-level loop

`migrate_pages` at `mm/migrate.c:2072` is the outer loop that takes a list of folios and migrates each one. Internally it calls `migrate_pages_batch` at `mm/migrate.c:1783`, which:

1. Unlocks folios from their LRUs.
2. For each folio, allocates a target folio.
3. Copies data (via `migrate_folio_move`).
4. Retries failed migrations with exponential backoff.
5. Returns the number of pages that could not be migrated (so the caller can decide whether to fail or try again).

The batch design exists because the per-folio work has fixed overhead (lock acquisition, LRU isolation, rmap walk); doing many folios at once amortizes that.

---

## Part 15: The Linux Scheduler {#part-15-the-linux-scheduler}

The scheduler answers one question: which thread runs next on this CPU? The kernel has hundreds or thousands of runnable threads and maybe 4–8 CPU cores. The scheduler picks one thread per core, lets it run for a while, then picks another.

The scheduler runs in two situations. First, **voluntarily**: a thread calls `schedule()` because it is about to sleep — waiting for I/O, a lock, a futex, a timer. The scheduler picks the next thread from the runqueue. Second, **involuntarily (preemption)**: a thread's time slice expires, or a higher-priority thread wakes up. The scheduler sets `TIF_NEED_RESCHED` on the running thread. At the next preemption point (return from syscall, return from interrupt, explicit `cond_resched()`), the kernel calls `schedule()`.

Context switching means saving the current thread's register state (stack pointer, program counter, callee-saved registers) and restoring the next thread's. On ARM64 this is `__switch_to` — about 20 instructions. The expensive part is not the register swap but the cache and TLB pollution: the new thread's working set is cold.

### Scheduling classes

The kernel has a hierarchy of scheduling classes, checked in priority order:

```
stop_sched_class       → migration threads (highest, internal only)
dl_sched_class         → SCHED_DEADLINE (earliest deadline first)
rt_sched_class         → SCHED_FIFO, SCHED_RR (fixed priority, real-time)
fair_sched_class       → SCHED_NORMAL, SCHED_BATCH (CFS/EEVDF, the default)
idle_sched_class       → SCHED_IDLE (lowest, runs when nothing else can)
```

The scheduler checks each class in order. If the RT class has a runnable thread, it runs before any CFS thread, unconditionally. A SCHED_DEADLINE thread runs before any RT thread. This ordering is absolute — a nice -20 CFS thread will never preempt a nice +19 RT thread.

On Android, most app threads are SCHED_NORMAL (CFS). Audio threads are SCHED_FIFO (RT). The display compositor (SurfaceFlinger) runs at RT priority.

### CFS and vruntime

CFS (Completely Fair Scheduler, merged in 2.6.23) tracks how much CPU time each thread has received and always runs the thread that has received the least. Every CFS thread has a `vruntime` field in its `struct sched_entity`:

```c
struct sched_entity {
    struct load_weight  load;       // weight derived from nice
    struct rb_node      run_node;   // position in the rbtree
    u64                 vruntime;   // virtual runtime in nanoseconds
    u64                 sum_exec_runtime;
};
```

`vruntime` is wall-clock time scaled by the thread's weight. A thread with twice the weight accumulates vruntime at half the rate — it can run twice as long before its vruntime catches up. The scheduler always picks the thread with the smallest vruntime.

When a thread runs for `delta_exec` nanoseconds:

```c
vruntime += delta_exec * NICE_0_LOAD / weight;
```

`NICE_0_LOAD` is 1024. A nice -5 thread (weight 3121) advances vruntime at 1024/3121 ≈ 0.33× — it gets ~3× more CPU than a nice-0 thread. A nice +5 thread (weight 335) advances at 1024/335 ≈ 3×, getting ~1/3 the CPU.

The nice-to-weight table is at `kernel/sched/core.c`:

```c
static const int sched_prio_to_weight[40] = {
 /* -20 */ 88761, 71755, 56483, 46273, 36291,
 /* -15 */ 29154, 23254, 18705, 14949, 11916,
 /* -10 */  9548,  7620,  6100,  4904,  3906,
 /*  -5 */  3121,  2501,  1991,  1586,  1277,
 /*   0 */  1024,   820,   655,   526,   423,
 /*   5 */   335,   272,   215,   172,   137,
 /*  10 */   110,    87,    70,    56,    45,
 /*  15 */    36,    29,    23,    18,    15,
};
```

Each nice level changes weight by ~25%. Android sets foreground threads to nice -10 (weight 9548) and background to nice +10 (weight 110) — a ratio of ~87×.

Kernel 6.6 replaced classic CFS with **EEVDF** (Earliest Eligible Virtual Deadline First). The rbtree is now sorted by virtual deadline rather than raw vruntime. EEVDF assigns each thread a virtual deadline = vruntime + (time slice / weight). The data structure is still an rbtree; the comparison key changed.

### RT scheduling

RT threads have fixed priorities from 1 to 99. Every RT thread runs before any CFS thread. Within RT, **SCHED_FIFO** threads run until they yield or block — no time slicing. **SCHED_RR** adds time slicing among same-priority threads (default 100ms quantum).

The RT runqueue is a bitmap + per-priority linked list. `pick_next_task_rt` finds the highest set bit (one instruction) and takes the first entry. O(1).

**Priority inversion.** High-priority thread H blocks on a mutex held by low-priority thread L. Medium-priority thread M preempts L. H starves. **Priority inheritance (PI) mutexes** fix this: when H blocks on a PI mutex held by L, the kernel boosts L to H's priority. On Android, Binder transactions use PI-aware mutexes for the audio and display paths.

**RT throttling.** `/proc/sys/kernel/sched_rt_runtime_us` (default 950000) limits RT threads to 950ms per 1s per CPU. The remaining 50ms is guaranteed for CFS threads so you can ssh in and kill a runaway RT process.

### SCHED_DEADLINE

EDF (Earliest Deadline First) scheduling. Each thread declares runtime, deadline, and period. The scheduler guarantees completion before the deadline if CPU capacity permits. SCHED_DEADLINE threads preempt RT threads. A natural fit for display rendering (16ms deadline per frame) and audio (10ms per buffer).

### sched_ext: BPF-based scheduling

Merged in kernel 6.12. `sched_ext` lets you write a scheduler in BPF. The kernel provides callbacks (`select_cpu`, `enqueue`, `dequeue`, `dispatch`). You implement them in BPF, load them, and they replace CFS for `SCHED_EXT` threads. If your BPF scheduler crashes, the kernel falls back to CFS.

Why this matters for mm: a BPF scheduler can access `task_struct→mm→rss_stat` and deprioritize threads whose working set is being reclaimed (they will just fault and stall anyway) or boost threads whose I/O just completed (their pages are hot).

### How the scheduler connects to mm

**Page fault.** A major fault calls `folio_wait_locked` → `schedule()` → thread sleeps until I/O completes. The scheduler picks another thread. When the I/O completes, the faulting thread is re-enqueued. Its vruntime has not advanced during sleep, so it runs promptly.

**Direct reclaim.** When `alloc_pages` cannot find free memory, the allocating thread enters direct reclaim — it does kswapd's work itself. The thread is still on-CPU (not sleeping), doing reclaim work instead of application work. This can take milliseconds. Direct reclaim is the single biggest source of unpredictable latency in Linux.

**kswapd.** Background reclaim daemon at nice -5 (weight 3121). Competes with normal threads via CFS. At nice -5 it gets ~3× more CPU than a nice-0 thread, but a CPU-bound app can slow kswapd, causing more direct reclaim.

**Futex / mutex.** When a thread blocks on a futex, it calls `schedule()`. When the holder unlocks, `futex_wake` wakes the blocked thread. If the blocked thread has a smaller vruntime, it may preempt the unlocking thread immediately.

---
## Part 16: Synchronization — How the mm Subsystem Stays Sane {#part-16-synchronization-how-the-mm-subsystem-stays-sane}

Multiple CPUs can fault on the same page simultaneously. Reclaim can try to evict a page while a process is mapping it. Writeback can be writing a page while the process is dirtying it again. The mm subsystem uses a layered locking strategy to handle all of this.

### The lock hierarchy

From coarsest to finest, acquired in this order:

```
mmap_lock          (per-mm, rw semaphore)     — protects VMA tree
  per-VMA lock     (per-VMA, rw semaphore)    — protects individual VMA (v6.4+)
    folio lock     (per-folio, sleeping bit)   — serializes I/O on page contents
      page table lock (per-PTE-page, spinlock) — protects PTE modifications
        xarray lock   (per-address_space, spin) — protects page cache tree
          LRU lock    (per-lruvec, spinlock)   — protects LRU list operations
```

Acquiring locks out of order causes deadlocks. The kernel's lockdep infrastructure detects inversions at runtime.

### mmap_lock (the big lock)

`mm_struct.mmap_lock` is a reader-writer semaphore that protects the VMA tree:

- **Read-locked** for page faults (you need to find the VMA, but you are not changing it).
- **Write-locked** for `mmap()`, `munmap()`, `mprotect()` (you are creating, destroying, or modifying VMAs).

Multiple page faults can proceed concurrently (read lock is shared). But `mmap()` blocks all faults (write lock is exclusive). This is why `mmap()` inside a hot loop can be a performance problem — it serializes against all page faults in the process.

### Per-VMA locks (v6.4+)

Recent kernels narrow the lock from per-mm to per-VMA for page faults. A fault only needs to lock the specific VMA it is faulting in. Two threads faulting on different files (different VMAs) acquire different locks and proceed independently, even while a third thread is calling `mmap()` to create a new VMA.

The per-VMA lock is a separate `rw_semaphore` embedded in each `vm_area_struct`. The fault path tries the per-VMA lock first; if it succeeds, the fault proceeds without touching `mmap_lock` at all. If it fails (because `mmap()` is restructuring this specific VMA), it falls back to `mmap_lock`.

### Page table lock (ptl)

Each bottom-level page table page has its own spinlock. When `finish_fault` or `do_wp_page` modifies a PTE, it acquires this lock:

```c
vmf->pte = pte_offset_map_lock(vma->vm_mm, vmf->pmd, vmf->address, &vmf->ptl);
// ... modify PTE ...
pte_unmap_unlock(vmf->pte, vmf->ptl);
```

This is a fine-grained lock — two faults to different pages in the same process, if they fall in different page table pages, acquire different locks and proceed in parallel. The granularity is one lock per 512 PTEs (one page table page), which covers 2MB of virtual address space.

### Folio lock

Each folio has a lock bit in its flags (`PG_locked`). This is used to serialize operations on the folio's contents:

- `filemap_fault` locks the folio while reading data from disk into it.
- `filemap_add_folio` requires the folio to be locked.
- Truncation locks the folio before removing it from the page cache.
- Writeback locks the folio while writing it to disk.

The lock is a sleeping lock (`folio_lock()` can block). This is important — you cannot hold a spinlock and call `folio_lock()`, because `folio_lock()` might schedule.

### The xarray lock

The page cache xarray (`mapping->i_pages`) has its own spinlock, acquired with `xa_lock_irq`. Operations that modify the xarray (adding pages, removing pages, setting dirty marks) hold this lock. It is held for very short durations — just long enough to update the tree node.

### TLB flushing

When a PTE is changed (page unmapped, permissions changed, page migrated), the old translation may still be cached in the TLB. The kernel must flush it. On a single-CPU system, a local TLB flush suffices. On multi-CPU systems, the kernel must send **TLB shootdown IPIs** (inter-processor interrupts) to all CPUs that might have the old entry cached. This is expensive — each IPI costs several microseconds of latency on the target CPU.

The mm subsystem batches TLB flushes where possible. For example, during `try_to_unmap` (reclaim), it sets `TTU_BATCH_FLUSH` to defer individual flushes and do one batch flush at the end.

The COW path at `mm/memory.c:3846` shows the critical ordering:

```c
// Clear the PTE and flush TLB BEFORE installing the new PTE.
// This used to set the new PTE then flush TLBs, but that left a window
// where the new PTE could be loaded into some TLBs while the old PTE
// remains in others.
ptep_clear_flush(vma, vmf->address, vmf->pte);  // clear + flush (atomic)
set_pte_at(mm, vmf->address, vmf->pte, entry);  // install new PTE
```

If you get this ordering wrong, two CPUs can temporarily see different physical pages for the same virtual address. The results range from data corruption to security vulnerabilities.

### The optimistic pattern

The mm subsystem heavily uses an optimistic concurrency pattern:

1. Read some state without locks (or with a read lock).
2. Do expensive work (allocate pages, read from disk).
3. Acquire the real lock.
4. Re-check the state (did someone else handle this while we were working?).
5. If nothing changed, commit. If something changed, retry or bail.

You see this in `handle_pte_fault`:

```c
vmf->orig_pte = ptep_get_lockless(vmf->pte);  // step 1: read without lock
// ... extensive work including disk I/O ...
spin_lock(vmf->ptl);                           // step 3: acquire lock
if (!pte_same(ptep_get(vmf->pte), entry))      // step 4: re-check
    goto unlock;                                // someone else handled it
```

And in `finish_fault`:

```c
vmf->pte = pte_offset_map_lock(...);
if (unlikely(vmf_pte_changed(vmf))) {          // re-check under lock
    ret = VM_FAULT_NOPAGE;                      // bail out
    goto unlock;
}
set_pte_range(vmf, folio, page, nr_pages, addr); // commit
```

This pattern minimizes the time locks are held, which is critical for scalability — you do not want to hold a spinlock while waiting for disk I/O.

### LRU lock

The lock hierarchy listed above includes the LRU lock but it deserves its own explanation. The LRU lock (`lruvec->lru_lock`) is a per-lruvec spinlock that protects the LRU lists — the lists that reclaim scans to find pages to evict.

Every `folio_add_lru` (adding a page to the LRU after allocation) and every `folio_del_from_lru_list` (removing a page during reclaim) acquires this lock. Under heavy allocation or reclaim pressure, the LRU lock becomes contended.

The lock is per-lruvec, not global. On non-NUMA systems, there is one lruvec per memory cgroup (if memcg is enabled) or one global lruvec. On NUMA, there is one per node per memcg. This keeps contention proportional to the number of CPUs sharing a NUMA node, not the total CPU count.

Operations that hold the LRU lock are short — a `list_add` or `list_del`, plus a few counter updates. The lock is always acquired with IRQs disabled (`spin_lock_irq`) because LRU operations can happen from both process context and softirq context (reclaim runs from both).

### lockdep: how the kernel catches deadlocks before they happen

lockdep is a runtime lock validator compiled into debug kernels (`CONFIG_LOCKDEP=y`). It does not detect deadlocks after they happen — it detects lock ordering violations that *could* cause deadlocks, even if the actual deadlock never occurs during that boot.

lockdep assigns each lock a **class** based on where it was initialized in the source code. Every time any code path acquires a lock, lockdep records the class of every lock already held by that CPU. This builds a directed graph of "class A was held while acquiring class B."

If lockdep ever sees the reverse — "class B was held while acquiring class A" — it reports an inversion. Two code paths acquiring the same two locks in different orders will eventually deadlock if they run concurrently, even if they haven't yet.

Concrete example from mm:

```
CPU 0 (page fault path):
  1. Acquire mmap_lock (read)
  2. Acquire folio lock

  lockdep records: mmap_lock → folio_lock   (order A → B)

CPU 1 (truncate path, hypothetical bug):
  1. Acquire folio lock
  2. Acquire mmap_lock (write)

  lockdep records: folio_lock → mmap_lock   (order B → A)

  LOCKDEP WARNING: possible circular locking dependency detected
```

CPU 0 and CPU 1 have never actually deadlocked — they may have run hours apart. But lockdep sees that the graph now has a cycle (A → B and B → A) and reports it immediately. The warning includes full stack traces for both acquisition sites so you can find the bug.

lockdep also tracks **IRQ state**: whether a lock has been acquired with IRQs enabled or disabled. If a lock is ever acquired with IRQs enabled AND the same lock class is acquired inside an IRQ handler, that's a potential deadlock — the IRQ can fire while the non-IRQ path holds the lock, and the IRQ handler spins forever waiting for a lock held by code that can't run (because the IRQ preempted it).

The mm subsystem's xarray lock uses `xa_lock_irqsave` precisely because the xarray can be modified from both process context (page cache insertion during fault) and interrupt context (bio completion calling `folio_end_writeback` which may modify xarray tags). lockdep would catch it if someone used `xa_lock` without disabling IRQs.

### Lock ordering rule

Sleeping locks must be acquired before spinlocks. Never the reverse.

```c
// CORRECT
folio_lock(folio);           // might sleep — acquired first
spin_lock(&ptl);             // does not sleep
spin_unlock(&ptl);
folio_unlock(folio);

// WRONG — DEADLOCK
spin_lock(&ptl);             // preemption disabled
folio_lock(folio);           // might need to sleep — cannot!
```

Why: spinlocks disable preemption. If you sleep while holding a spinlock, the lock is held by a sleeping task. Other CPUs spin forever waiting. The sleeping task cannot wake up because the scheduler cannot run (preemption is disabled on that CPU). Deadlock.

The kernel's lockdep infrastructure automatically detects these inversions at runtime and prints a warning before the deadlock happens.

### Memory barriers

Write-back caches and store buffers mean writes from one CPU may not be visible to another immediately. The kernel uses explicit barriers to enforce ordering:

```c
// Producer writes data, then publishes position
buffer[0] = 'H';
smp_store_release(&position, new_pos);
// guarantees: buffer write visible before position update

// Consumer reads position, then reads data
pos = smp_load_acquire(&position);
char c = buffer[0];  // guaranteed to see 'H' if we saw new_pos
```

On ARM64 without barriers, the consumer might see the updated position but stale data — the store buffer on the producer's CPU has not drained yet. x86 has stricter default ordering (TSO — Total Store Order) but kernel code must use explicit barriers for portability.

The BPF ring buffer (`kernel/bpf/ringbuf.c:463`) uses exactly this pattern: `smp_store_release(&rb->producer_pos, new_prod_pos)` after writing the record header, `smp_load_acquire(&rb->consumer_pos)` before reading consumer state. These two barriers are what make the ring buffer safe without locks.

---

## Part 17: eBPF — Observing the Kernel From the Inside {#part-17-ebpf-observing-the-kernel-from-the-inside}

eBPF lets you run small programs inside the kernel, safely, without modifying the kernel source or loading traditional kernel modules. Originally designed as a packet filter in 1992, it has been rebuilt into a general-purpose instrumentation framework that powers tracing, networking, security, and scheduling on every modern Linux system.

This section explains what eBPF actually is at the implementation level — the instruction set, the interpreter, the JIT, the verifier, the helper call mechanism, and how a BPF program attaches to a tracepoint. Everything is grounded in actual kernel code.

### The brief history

**BPF (1992).** Steven McCanne and Van Jacobson at LBL designed the Berkeley Packet Filter for `tcpdump`. It was a tiny register machine — two 32-bit registers (A and X), a scratch memory area, ~20 instructions — interpreted by the kernel for each packet. The innovation was *safety through simplicity*: the instruction set was so restricted that the kernel could verify the program in linear time, and because it had no loops and no memory access outside a fixed scratch area, it could not crash or hang the kernel.

**Linux adopts it (1997).** Linux added BPF to socket filters. For the next 15 years, it stayed mostly as-is — a niche tool for packet filters.

**eBPF (2014, kernel 3.18).** Alexei Starovoitov rewrote BPF for the modern era. The new instruction set looked much more like a real CPU: ten 64-bit registers, a 512-byte stack, calls to kernel helper functions, and a more sophisticated verifier. Crucially, the architecture mirrored x86-64 register conventions so BPF code could be JIT-compiled to native instructions with minimal overhead.

**The tracing wave (2015–2018).** Brendan Gregg and others realized eBPF was the right tool for Linux tracing. The `bcc` toolkit and later `bpftrace` made it practical. Kernel developers added tracepoint attachment, kprobes, and uprobes. The BPF helper library grew to hundreds of functions.

**BTF and CO-RE (2019, kernel 5.4).** Andrii Nakryiko added BPF Type Format — debug information embedded in the kernel image. With BTF, a BPF program compiled against one kernel can run on a different kernel, with libbpf relocating struct field offsets at load time. This is what made BPF programs portable across distributions and versions.

**Today (kernel 6.x).** eBPF is everywhere: Cilium for networking, Tetragon for security, parca for profiling, Android's `netd`. The verifier has grown from ~1,000 lines to ~20,000 lines, the instruction set has gained bounded loops, function calls, spin locks, ring buffers, and dynamic pointers. You can write a scheduler in eBPF now (`sched_ext`).

### The instruction format

At the heart of eBPF is `struct bpf_insn` at `include/uapi/linux/bpf.h:80`:

```c
struct bpf_insn {
    __u8    code;       /* opcode */
    __u8    dst_reg:4;  /* dest register */
    __u8    src_reg:4;  /* source register */
    __s16   off;        /* signed offset */
    __s32   imm;        /* signed immediate constant */
};
```

Every eBPF instruction is exactly 8 bytes. An 8-bit opcode, two 4-bit register fields (you have 11 registers, so 4 bits is enough), a 16-bit signed offset, and a 32-bit signed immediate. This fixed width is what makes the verifier and JIT simple — there is no decoding ambiguity.

The opcode encodes three things:
- **Class** (low 3 bits): BPF_ALU64, BPF_ALU, BPF_JMP, BPF_JMP32, BPF_LD, BPF_LDX, BPF_ST, BPF_STX.
- **Operation** (high 4 bits): BPF_ADD, BPF_SUB, BPF_MOV, BPF_JEQ, BPF_JGT, BPF_CALL, etc.
- **Source mode** (bit 3): BPF_K (immediate) or BPF_X (register).

So `BPF_ALU64 | BPF_ADD | BPF_X` means "64-bit integer add, register-to-register." `BPF_ALU64 | BPF_MOV | BPF_K` means "64-bit integer move immediate."

There are 11 registers (`MAX_BPF_REG` = 11):
- **R0**: return value from helper calls and the program itself.
- **R1-R5**: argument registers for helper calls.
- **R6-R9**: callee-saved registers (preserved across helper calls).
- **R10**: the frame pointer — read-only, points to the 512-byte stack.

This calling convention deliberately matches x86-64. When a BPF program calls a kernel helper, the JIT translates the BPF call to a native `call` instruction, and R1-R5 map directly to the x86 argument registers. No shuffling required.

### The interpreter

Every eBPF program can either be interpreted or JIT-compiled. On most architectures (x86, arm64, arm, mips, riscv, s390x, power), JIT is the default — the interpreter exists as a fallback and for architectures without JIT support. The interpreter is `___bpf_prog_run` at `kernel/bpf/core.c:1775`. It uses a **computed-goto dispatch** that is faster than a switch statement:

```c
static u64 ___bpf_prog_run(u64 *regs, const struct bpf_insn *insn)
{
    static const void * const jumptable[256] __annotate_jump_table = {
        [0 ... 255] = &&default_label,
        /* Now overwrite non-defaults ... */
        BPF_INSN_MAP(BPF_INSN_2_LBL, BPF_INSN_3_LBL),
        [BPF_JMP | BPF_CALL_ARGS] = &&JMP_CALL_ARGS,
        [BPF_JMP | BPF_TAIL_CALL] = &&JMP_TAIL_CALL,
        [BPF_ST  | BPF_NOSPEC]    = &&ST_NOSPEC,
        [BPF_LDX | BPF_PROBE_MEM | BPF_B] = &&LDX_PROBE_MEM_B,
        // ...
    };

#define CONT     ({ insn++; goto select_insn; })
#define CONT_JMP ({ insn++; goto select_insn; })

select_insn:
    goto *jumptable[insn->code];

    /* ALU (rest) */
#define ALU(OPCODE, OP)                                 \
    ALU64_##OPCODE##_X:                                 \
        DST = DST OP SRC;                               \
        CONT;                                           \
    ALU_##OPCODE##_X:                                   \
        DST = (u32) DST OP (u32) SRC;                   \
        CONT;                                           \
    ALU64_##OPCODE##_K:                                 \
        DST = DST OP IMM;                               \
        CONT;                                           \
    ALU_##OPCODE##_K:                                   \
        DST = (u32) DST OP (u32) IMM;                   \
        CONT;
    ALU(ADD,  +)
    ALU(SUB,  -)
    ALU(AND,  &)
    ALU(OR,   |)
    ALU(XOR,  ^)
    ALU(MUL,  *)
    // ...
```

The GCC "labels as values" extension (`&&label`) lets the jumptable hold pointers to C labels directly. `goto *jumptable[insn->code]` jumps to the appropriate handler based on the opcode. Each handler ends with `CONT`, which advances the instruction pointer and dispatches the next instruction — no loop overhead, the CPU's branch predictor sees each site as an independent indirect jump and learns the common transitions.

The `ALU` macro is a neat trick: each arithmetic opcode is just `DST = DST OP SRC` or `DST = DST OP IMM`, with 32-bit vs 64-bit variants. The whole integer ALU is a few dozen lines.

### The JIT

On a tracing kernel, you almost never run the interpreter. The JIT at `arch/arm64/net/bpf_jit_comp.c` (on ARM64) translates each BPF instruction to one or a few native instructions. For example, BPF `BPF_ALU64 | BPF_ADD | BPF_X` (add reg-to-reg, 64-bit) becomes a single ARM64 `add x{dst}, x{dst}, x{src}` instruction. The JIT emits native code into a writable-then-readonly kernel memory region, and calling the program is a simple function call.

Because BPF uses the same register conventions as the host ABI, helper calls translate directly. When the JIT sees `BPF_JMP | BPF_CALL` with an immediate that identifies a helper, it emits a native `bl` (branch-and-link) to the helper function's address. R1-R5 are already in the correct ARM64 argument registers, R0 receives the return value — no marshalling code needed.

### The verifier

The verifier is the reason eBPF is safe. From the top-of-file comment at `kernel/bpf/verifier.c:56`:

> *"bpf_check() is a static code analyzer that walks eBPF program instruction by instruction and updates register/stack state. All paths of conditional branches are analyzed until 'bpf_exit' insn. The first pass is depth-first-search to check that the program is a DAG. The second pass is all possible path descent from the 1st insn."*

Two passes:

**Pass 1: DAG check.** Depth-first search over the control flow graph. Rejects:
- Programs larger than `BPF_MAXINSNS` (1M instructions).
- Loops (back-edges) — unless they are bounded loops that the verifier can unroll.
- Unreachable instructions.
- Out-of-bounds or malformed jumps.

**Pass 2: state simulation.** This is where most of the verifier's complexity lives. The verifier walks every path through the program, tracking the *type* of every register at every point. Types include:

- `SCALAR_VALUE`: a non-pointer integer.
- `PTR_TO_CTX`: pointer to the program's context (e.g., the tracepoint struct).
- `PTR_TO_MAP_VALUE`: pointer returned by `bpf_map_lookup_elem`, valid for the map's value size.
- `PTR_TO_MAP_VALUE_OR_NULL`: same, but might be NULL — must be null-checked before use.
- `PTR_TO_STACK`: pointer into the program's 512-byte stack.
- `PTR_TO_PACKET`: for XDP/socket filter programs, pointer into packet data.

The verifier rejects any memory access whose pointer type and offset cannot be proven safe. The core check is in `check_mem_access` at `kernel/bpf/verifier.c:7702`. Before a load or store, this function checks:
- The base register has a valid pointer type.
- The offset is within the known bounds for that type.
- For map values, the access is within `map->value_size`.
- For packet pointers, the program has verified `end - start >= access_offset + size`.

**The NULL check enforcement.** When you call `bpf_map_lookup_elem`, the verifier assigns the return value type `PTR_TO_MAP_VALUE_OR_NULL`. Any dereference is rejected until you do `if (ptr)`. After the comparison, the verifier knows one branch has `PTR_TO_MAP_VALUE` (dereferenceable) and the other has a scalar 0 (not dereferenceable). This is why you see `if (!ptr) return 0;` everywhere in BPF code — it's not defensive programming, it's a verifier requirement.

The downside: the verifier is conservative. Programs it cannot prove safe are rejected, even if a human can tell they are safe. You sometimes have to rewrite code to help the verifier along — bounded loops with obvious upper bounds, explicit range checks, helper functions that give the verifier more information.

### The bpf() syscall

Userspace loads programs, creates maps, and attaches programs via one syscall. `SYSCALL_DEFINE3(bpf, ...)` at `kernel/bpf/syscall.c:6360`:

```c
SYSCALL_DEFINE3(bpf, int, cmd, union bpf_attr __user *, uattr, unsigned int, size)
{
    return __sys_bpf(cmd, USER_BPFPTR(uattr), size);
}
```

The `cmd` argument determines what the syscall does:
- `BPF_MAP_CREATE`: create a map. `__sys_bpf` calls `map_create` at line 1369.
- `BPF_MAP_LOOKUP_ELEM`, `BPF_MAP_UPDATE_ELEM`, `BPF_MAP_DELETE_ELEM`: manipulate map entries from userspace.
- `BPF_PROG_LOAD`: load a program. Calls `bpf_prog_load` at line 2871, which runs the verifier and optionally JITs the program.
- `BPF_PROG_ATTACH`, `BPF_LINK_CREATE`: attach a loaded program to a hook.
- `BPF_RAW_TRACEPOINT_OPEN`: attach specifically to a raw tracepoint.

Each successful operation returns a file descriptor. The map fd and program fd are what tie everything together — when a BPF program references a map, it references it by fd (the verifier converts the fd to an internal pointer during load).

### Helpers: how BPF programs call into the kernel

BPF programs cannot directly call arbitrary kernel functions — that would be unsafe. Instead, they call a curated set of **helpers**. Each helper has a numeric ID, a C implementation, and a type signature that the verifier uses to check arguments.

Here is `bpf_map_lookup_elem` at `kernel/bpf/helpers.c:44`:

```c
BPF_CALL_2(bpf_map_lookup_elem, struct bpf_map *, map, void *, key)
{
    WARN_ON_ONCE(!bpf_rcu_lock_held());
    return (unsigned long) map->ops->map_lookup_elem(map, key);
}

const struct bpf_func_proto bpf_map_lookup_elem_proto = {
    .func       = bpf_map_lookup_elem,
    .gpl_only   = false,
    .pkt_access = true,
    .ret_type   = RET_PTR_TO_MAP_VALUE_OR_NULL,
    .arg1_type  = ARG_CONST_MAP_PTR,
    .arg2_type  = ARG_PTR_TO_MAP_KEY,
};
```

The `BPF_CALL_2` macro generates the actual function plus a wrapper that reads arguments from registers. The `bpf_func_proto` is the verifier's type signature:
- `ret_type = RET_PTR_TO_MAP_VALUE_OR_NULL`: the verifier will mark R0 as nullable after this call.
- `arg1_type = ARG_CONST_MAP_PTR`: the first argument must be a constant pointer to a map (the verifier tracks which map, so it knows the value size).
- `arg2_type = ARG_PTR_TO_MAP_KEY`: the second argument must be a pointer to stack memory sized at least `map->key_size`.

Another example — `bpf_ktime_get_ns` at `kernel/bpf/helpers.c:178`:

```c
BPF_CALL_0(bpf_ktime_get_ns)
{
    /* NMI safe access to clock monotonic */
    return ktime_get_mono_fast_ns();
}
```

Takes no arguments, returns a timestamp in nanoseconds. The `ktime_get_mono_fast_ns` variant is NMI-safe, which matters because BPF programs can run from any context, including NMI handlers.

And `bpf_get_smp_processor_id` at `kernel/bpf/helpers.c:155`:

```c
BPF_CALL_0(bpf_get_smp_processor_id)
{
    return smp_processor_id();
}
```

When a BPF program calls a helper, the JIT emits a native call to the wrapper function. R1-R5 are already in ARM64/x86 argument registers due to the matched calling convention, so the call costs roughly the same as a regular kernel function call.

### Hash maps

The hash map implementation is at `kernel/bpf/hashtab.c`. A map is an array of buckets (sized as the next power of 2 above `max_entries`), each protected by a raw spinlock. Lookup:

```c
// Simplified from __htab_map_lookup_elem at kernel/bpf/hashtab.c:732
static struct htab_elem *__htab_map_lookup_elem(struct bpf_map *map, void *key)
{
    struct bpf_htab *htab = container_of(map, struct bpf_htab, map);
    struct hlist_nulls_head *head;
    struct htab_elem *l;
    u32 hash, key_size;

    key_size = map->key_size;
    hash = htab_map_hash(key, key_size, htab->hashrnd);  // jhash
    head = select_bucket(htab, hash);

    l = lookup_nulls_elem_raw(head, hash, key, key_size, htab->n_buckets);
    return l;
}
```

jhash is the Jenkins hash — fast and has good distribution for small keys. `htab->hashrnd` is a per-map random seed, set at map creation, that prevents hash-collision attacks.

The `hlist_nulls_head` is a special linked list where the terminator encodes the bucket index. This lets RCU readers detect if they've walked off the end of a bucket (which can happen during concurrent updates) without taking a lock.

Updates (`htab_map_update_elem` at line 1167) acquire the bucket's spinlock, find an existing element with the matching key or allocate a new one, and insert it. For pre-allocated maps (the default), there's no allocation at update time — elements come from a pre-sized pool, which is what makes these maps usable from NMI context.

### Ring buffer internals

For high-throughput event streaming, the ring buffer is the right tool. Its reservation path at `kernel/bpf/ringbuf.c:463`:

```c
static void *__bpf_ringbuf_reserve(struct bpf_ringbuf *rb, u64 size)
{
    unsigned long cons_pos, prod_pos, new_prod_pos, pend_pos, over_pos, flags;
    struct bpf_ringbuf_hdr *hdr;
    u32 len, pg_off, hdr_len;

    if (unlikely(size > RINGBUF_MAX_RECORD_SZ))
        return NULL;

    len = round_up(size + BPF_RINGBUF_HDR_SZ, 8);   // 8-byte alignment
    if (len > ringbuf_total_data_sz(rb))
        return NULL;

    cons_pos = smp_load_acquire(&rb->consumer_pos);  // pairs with userspace store

    if (raw_res_spin_lock_irqsave(&rb->spinlock, flags))
        return NULL;

    // Advance pending_pos past any records that are no longer busy
    pend_pos = rb->pending_pos;
    prod_pos = rb->producer_pos;
    new_prod_pos = prod_pos + len;

    while (pend_pos < prod_pos) {
        hdr = (void *)rb->data + (pend_pos & rb->mask);
        hdr_len = READ_ONCE(hdr->len);
        if (hdr_len & BPF_RINGBUF_BUSY_BIT)
            break;                       // a producer is still writing here
        pend_pos += bpf_ringbuf_round_up_hdr_len(hdr_len);
    }
    rb->pending_pos = pend_pos;

    if (!bpf_ringbuf_has_space(rb, new_prod_pos, cons_pos, pend_pos)) {
        raw_res_spin_unlock_irqrestore(&rb->spinlock, flags);
        return NULL;                     // buffer full
    }

    // Carve out our slot
    hdr = (void *)rb->data + (prod_pos & rb->mask);
    hdr->len = size | BPF_RINGBUF_BUSY_BIT;     // mark busy so consumers skip
    hdr->pg_off = bpf_ringbuf_rec_pg_off(rb, hdr);

    // Advance producer_pos BEFORE releasing the spinlock
    smp_store_release(&rb->producer_pos, new_prod_pos);
    raw_res_spin_unlock_irqrestore(&rb->spinlock, flags);

    return (void *)hdr + BPF_RINGBUF_HDR_SZ;
}
```

A few things to notice:

**The busy bit.** Each record's header has a `BPF_RINGBUF_BUSY_BIT`. When a producer reserves a slot, the bit is set. The consumer on userspace sees the bit and knows the record is not yet committed — it stops reading at that point even though the producer_pos has advanced. This lets multiple producers write concurrently without corrupting the consumer's view.

**The memory barriers.** `smp_store_release` and `smp_load_acquire` are the critical synchronization primitives. The release-acquire pair ensures that when the consumer loads `producer_pos` and sees it has advanced, all the writes the producer did before `smp_store_release` are visible. Without this, the consumer could see an updated position but stale data.

**Submission.** When the producer finishes writing, it calls `bpf_ringbuf_submit` which clears the busy bit:

```c
// kernel/bpf/ringbuf.c:559
static void bpf_ringbuf_commit(void *sample, u64 flags, bool discard)
{
    hdr = sample - BPF_RINGBUF_HDR_SZ;
    rb = bpf_ringbuf_restore_from_rec(hdr);
    new_len = hdr->len ^ BPF_RINGBUF_BUSY_BIT;   // clear busy bit atomically

    xchg(&hdr->len, new_len);  // atomic xchg = full barrier + store

    // Wake userspace if the consumer has caught up to this slot
    rec_pos = (void *)hdr - (void *)rb->data;
    cons_pos = smp_load_acquire(&rb->consumer_pos) & rb->mask;

    if (flags & BPF_RB_FORCE_WAKEUP)
        irq_work_queue(&rb->work);
    else if (cons_pos == rec_pos && !(flags & BPF_RB_NO_WAKEUP))
        irq_work_queue(&rb->work);    // wake via softirq context
}
```

The wakeup uses `irq_work_queue` instead of a direct wakeup because BPF programs can run in hard IRQ context where you cannot safely touch scheduler locks. `irq_work_queue` defers the wakeup to softirq context.

**Userspace mmap.** The magic that lets userspace read the ring buffer efficiently: the buffer's data pages are **double-mapped** in kernel virtual memory. A write near the end of the buffer that logically wraps to the beginning is, in kernel virtual address space, just a linear write — the second mapping sits immediately after the first. This means the producer never has to handle wraparound specially. Userspace mmaps the consumer and producer position pages plus the data pages and reads entries in order.

### Attaching a BPF program to a tracepoint

Tracepoints are static instrumentation points compiled into the kernel. They are implemented with "static keys" — branches that the compiler turns into NOPs when disabled and into real calls when enabled. The disabled cost is zero; the enabled cost is one indirect call plus the program's runtime.

When you load a BPF program and attach it to a tracepoint (via `BPF_RAW_TRACEPOINT_OPEN` or `BPF_LINK_CREATE`), the kernel adds your program to a `prog_array` attached to the tracepoint. When the tracepoint fires, it calls `trace_call_bpf` at `kernel/trace/bpf_trace.c:109`:

```c
unsigned int trace_call_bpf(struct trace_event_call *call, void *ctx)
{
    unsigned int ret;

    cant_sleep();    // tracepoints run in contexts where you cannot sleep

    // Recursion guard: only allow one BPF program per CPU at a time.
    // Otherwise a BPF program that (indirectly) triggers another tracepoint
    // could infinitely recurse and blow the stack.
    if (unlikely(__this_cpu_inc_return(bpf_prog_active) != 1)) {
        rcu_read_lock();
        bpf_prog_inc_misses_counters(rcu_dereference(call->prog_array));
        rcu_read_unlock();
        ret = 0;
        goto out;
    }

    rcu_read_lock();
    ret = bpf_prog_run_array(rcu_dereference(call->prog_array),
                             ctx, bpf_prog_run);
    rcu_read_unlock();

 out:
    __this_cpu_dec(bpf_prog_active);
    return ret;
}
```

Three things here are worth understanding:

**The recursion guard (`bpf_prog_active`).** If a BPF program allocates memory (triggering `mm_page_alloc`) or takes a lock that fires a tracepoint, you could recurse into `trace_call_bpf` from inside a BPF program. The per-CPU counter prevents this — if we are already inside a BPF program on this CPU, the inner tracepoint fires but finds the counter nonzero and skips program execution. This is why you sometimes see "missed events" in BPF stats; usually this is the culprit.

**RCU for program lookup.** `call->prog_array` is the list of attached programs. It is modified only during attach/detach operations and read on every tracepoint hit. RCU is the natural fit: readers take a cheap `rcu_read_lock`; writers copy-and-replace the array and wait for a grace period before freeing the old one.

**`cant_sleep()`** is an assertion that catches bugs — tracepoints fire in atomic context (preemption disabled, IRQs possibly disabled), so no helper a BPF program calls can ever sleep.

### Kprobes: hooking any kernel function

Tracepoints exist only where kernel developers placed them. Kprobes let you hook any kernel function dynamically. When you attach a kprobe to `filemap_dirty_folio`, the kernel:

1. Makes a private copy of the first instruction(s) of `filemap_dirty_folio`.
2. Replaces the first instruction with a `brk` (ARM64) or `int3` (x86) breakpoint.
3. When the CPU hits that breakpoint, it traps into the kprobe handler, which calls your BPF program with `struct pt_regs *` pointing to the current register state.
4. After your BPF program returns, the kernel emulates the original instruction and resumes the target function.

The cost is higher than a tracepoint — a breakpoint trap is ~200-500ns versus ~50ns for a tracepoint — but you can hook anywhere. For a few functions per second, this is fine. For hot paths, prefer a tracepoint.

From BPF, kprobe arguments come via `struct pt_regs`:

```c
SEC("kprobe/filemap_dirty_folio")
int on_dirty(struct pt_regs *ctx) {
    // filemap_dirty_folio(mapping, folio) — args are in x0, x1 on ARM64
    struct address_space *mapping = (struct address_space *)PT_REGS_PARM1(ctx);
    struct folio *folio = (struct folio *)PT_REGS_PARM2(ctx);
    // ...
}
```

The `PT_REGS_PARM*` macros abstract over architecture-specific register layouts.

### BTF and CO-RE: compile once, run everywhere

A big problem with early eBPF tracing was portability. A program compiled against kernel 5.4 hardcoded offsets like `offsetof(struct folio, mapping) = 24`. On kernel 5.10, where the struct layout had changed, the program read the wrong memory.

**BTF** (BPF Type Format) is debug information embedded in the kernel image at `/sys/kernel/btf/vmlinux`. It describes every struct, every field offset, every function signature. With BTF, the BPF loader can **relocate** field accesses at load time — your program says "give me the `mapping` field of `struct folio`," and the loader patches in the correct offset for the running kernel.

This is **CO-RE** (Compile Once, Run Everywhere). In practice, you generate a header from the running kernel's BTF:

```bash
bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h
```

And you use `BPF_CORE_READ` macros that emit relocatable accesses:

```c
struct address_space *mapping = BPF_CORE_READ(folio, mapping);
unsigned long ino = BPF_CORE_READ(folio, mapping, host, i_ino);
```

Under the hood, `BPF_CORE_READ` expands to a load instruction with a special relocation record. At load time, libbpf reads the kernel's BTF, finds the struct and field, and patches the instruction's immediate operand with the correct offset. This is how the same BPF binary can run on kernel 5.4 and 6.12 unchanged.

### Putting it all together

A BPF program attached to a tracepoint runs this path on every event:

1. Kernel code reaches `trace_mm_page_alloc(page, order, ...)`.
2. The tracepoint's static key is enabled, so the call goes through.
3. `trace_call_bpf` is invoked with the tracepoint's `ctx` (the event struct).
4. Per-CPU recursion guard is checked and incremented.
5. Under `rcu_read_lock`, the list of attached BPF programs is walked.
6. For each program: jump to the JIT-compiled native code (or the interpreter).
7. The BPF program reads fields from `ctx`, calls helpers (`bpf_map_lookup_elem`, `bpf_ringbuf_reserve`, `bpf_ktime_get_ns`), writes output.
8. Return from the program. Guard is decremented. Tracepoint returns.

End-to-end, a well-written BPF handler on a hot tracepoint adds 100–500ns of overhead per event.

### NMI and eBPF

NMI (Non-Maskable Interrupt) on x86, or pseudo-NMI via GIC priority on ARM64, cannot be blocked. BPF programs attached to perf events can run in NMI context. Constraints: no spinlocks (the interrupted code might hold them), no `wake_up` (runqueue lock deadlock), no sleeping. This is why the ring buffer uses `irq_work_queue` to defer the consumer wakeup to softirq context — calling `wake_up` from NMI would deadlock if the interrupted code holds the scheduler's runqueue lock.

This also affects BPF map choice: `BPF_MAP_TYPE_HASH` with pre-allocated entries (`BPF_F_NO_PREALLOC` unset) is safe from NMI because updates never allocate memory. Dynamically-allocated hash maps are not NMI-safe.

### Tracepoints vs kprobes for the tracer

The full list of events the tracer needs and how each is captured:

```
Event                      Method         PFN source
────────────────────────────────────────────────────────────
mm_page_alloc              tracepoint     ctx->pfn (direct)
mm_page_free               tracepoint     ctx->pfn (direct)
mm_filemap_add_to_cache    tracepoint     ctx->pfn (direct)
mm_filemap_delete_from_cache tracepoint   ctx->pfn (direct)
block_rq_issue             tracepoint     sector lookup (sector_to_id map)
block_rq_complete          tracepoint     sector lookup (sector_to_id map)
filemap_dirty_folio        kprobe         folio → PFN (vmemmap math or kfunc)
folio_end_writeback        kprobe         folio → PFN
folio_end_read             kprobe         folio → PFN
bio_add_folio              kprobe         folio → PFN + sector (populates sector_to_id)
migrate_folio_move         kprobe         folio → PFN (both old and new)
do_anonymous_page          kprobe         folio → PFN
wp_page_copy               kprobe         folio → PFN (both old and new)
zram_write_page            kprobe         page → PFN (direct function argument)
```

No kernel patches needed. Stock tracepoints cover the high-volume events (alloc, free, cache); kprobes fill the gaps where no tracepoint carries the PFN.

---

## Part 18: Building a Page Lifecycle Tracer with eBPF {#part-18-building-a-page-lifecycle-tracer-with-ebpf}

Now we put it all together. The goal: track every physical page from allocation through page cache insertion, fault, dirty, writeback, I/O, and free. Userspace reads a stream of events from the BPF ring buffer and can reconstruct any page's history at any point.

### The page lifecycle

A physical page moves through a sequence of states. The tracer captures each transition:

```
FILE-BACKED PAGE:

  buddy allocator
       │
       ▼
  ALLOC → CACHE_INSERT → FAULT → DIRTY → WRITEBACK → IO_ISSUE
                                    │         │           │
                                    │         │           ▼
                                    │         │      IO_COMPLETE
                                    │         │           │
                                    │         ▼           ▼
                                    │      WB_DONE ──→ (re-dirty? → DIRTY again)
                                    │         │
                                    ▼         ▼
                               CACHE_REMOVE → FREE → buddy allocator

ANONYMOUS PAGE:

  ALLOC → ANON_FAULT (born dirty) → ZRAM_COMPRESS → FREE
               │
               └── COW (if forked) → new page_id

MIGRATION (can happen at any lifecycle stage):

  ... → MIGRATE(old_pfn → new_pfn, same page_id) → ...
```

Every box is an event the tracer emits. Every event carries the same `page_id`, so the full lifecycle reconstructs by `SELECT * FROM page_event WHERE page_id = N ORDER BY ts`.

### Why an event stream, not a per-page struct

The first instinct is to store one struct per page with timestamp fields for each lifecycle stage (`alloc_ts`, `cache_ts`, `dirty_ts`, ...). This has real problems:

1. **You lose history.** A page can go through dirty→writeback→clean→dirty→writeback multiple times. With one timestamp slot per stage, you only see the last cycle.
2. **You cannot see the current state mid-flight.** Until `free_ts` is set, a polling reader does not know which stage the page is in — it just sees a half-filled struct.
3. **The struct grows with every new event type.** Adding zram tracking means adding more fields, restructuring the map value, and recompiling everything.

The better model: emit individual **events** into a ring buffer, each tagged with a stable **page_id**. Userspace reads the stream and can reconstruct any page's lifecycle by filtering on its page_id. You can answer "what happened to page 42" and "what is happening right now" and "show me all dirty events in the last second" from the same data.

### The page_id: stable identity across migration

Every tracepoint gives you a PFN (physical frame number). But PFNs are not stable — page migration changes the PFN while the logical page stays the same. If you key everything on PFN, a migration silently breaks the connection between events before and after the move.

The solution is a **page_id** — a monotonically increasing 64-bit integer assigned once when a page is allocated, and never changed. The tracer maintains a small lookup map from PFN to page_id. Every handler looks up the page_id from the PFN, and every event carries the page_id. When migration happens, only the lookup map needs updating.

We generate page_ids inside BPF using an atomic counter in an array map — no kernel patches needed.

### The maps

Four maps, each with a clear purpose:

```c
// ── Map 1: page_id generator ──
// A single-element array holding the next page_id to assign.
// Atomically incremented on each page allocation.
struct {
    __uint(type, BPF_MAP_TYPE_ARRAY);
    __uint(max_entries, 1);
    __type(key, u32);
    __type(value, u64);
} page_id_counter SEC(".maps");

// ── Map 2: PFN → page_id lookup ──
// This is the bridge between what tracepoints give us (PFN) and
// our stable identifier (page_id). Updated on alloc, free, and migration.
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 131072);
    __type(key, u64);              // pfn
    __type(value, u64);            // page_id
} pfn_to_id SEC(".maps");

// ── Map 3: event ring buffer ──
// All events flow through here to userspace. Lock-free, multi-producer.
// Sized at 16MB — enough for ~200K events before wrapping.
struct {
    __uint(type, BPF_MAP_TYPE_RINGBUF);
    __uint(max_entries, 16 * 1024 * 1024);
} events SEC(".maps");

// ── Map 4: filter configuration ──
// Userspace writes this to control what we trace.
// Without a filter, tracking ALL user pages on a busy system will
// overwhelm the ring buffer within seconds.
struct trace_filter {
    u32 filter_type;  // 0=off, 1=by_tgid, 2=by_ino
    u32 tgid;
    u64 ino;
    u64 dev;
};
struct {
    __uint(type, BPF_MAP_TYPE_ARRAY);
    __uint(max_entries, 1);
    __type(key, u32);
    __type(value, struct trace_filter);
} filter SEC(".maps");
```

**Why a filter is required.** A typical Android phone allocates and frees tens of thousands of pages per second. Without filtering, the ring buffer fills up in under a second even at 64MB. The filter lets you target a specific process or a specific file, reducing volume by orders of magnitude.

**Why PID filtering alone breaks.** Many lifecycle events fire from kernel threads, not the app:

```
Event               Context          PID filtering works?
─────────────────────────────────────────────────────────
mm_page_alloc       app              yes
page_fault          app              yes
writeback           kworker          no
reclaim/free        kswapd           no
block_rq_complete   softirq          no
migration           kcompactd        no
```

The fix: filter by PID at allocation time, then track by PFN → page_id for all subsequent events regardless of which context triggers them. Once a page is in the `pfn_to_id` map, every subsequent event on that PFN is captured no matter who fires it.

**Filtering by process name.** `--comm chrome` uses `bpf_get_current_comm()` at allocation time. The handler caches `tgid → tracked` in a `tracked_tgids` hash map so the comm string comparison happens once per tgid, not once per allocation.

**Filtering by file.** `--file "/usr/lib/*.so"` is resolved in userspace: `glob()` → `stat()` each match → populate a `tracked_inodes` hash map with `(ino, dev)` pairs. The eBPF handler at `on_cache_add` checks `tracked_inodes`. If no match, the page is removed from `pfn_to_id` to avoid map bloat. The user never sees inode numbers — only file paths and globs.

**Filtering timing.** `on_page_alloc` fires before we know *what* the page is for — it could become file-backed, anonymous, slab, anything. So `on_page_alloc` can only filter by tgid. The ino/dev filter kicks in at `on_cache_add`, which is the earliest point where the kernel assigns the page to a file. Pages that fail the ino filter at `on_cache_add` are retroactively cleaned out of `pfn_to_id`.

### The event struct

Every event is the same struct. The `event_type` field tells you what happened; the rest is a union of event-specific data. This keeps the ring buffer entries uniform and easy to parse.

```c
enum event_type {
    EVT_ALLOC        = 0,
    EVT_FREE         = 1,
    EVT_CACHE_INSERT = 2,
    EVT_CACHE_REMOVE = 3,
    EVT_FAULT        = 4,
    EVT_DIRTY        = 5,
    EVT_WRITEBACK    = 6,
    EVT_WB_DONE      = 7,
    EVT_IO_ISSUE     = 8,
    EVT_IO_COMPLETE  = 9,
    EVT_ANON_FAULT   = 10,
    EVT_COW          = 11,
    EVT_MIGRATE      = 12,
    EVT_ZRAM_COMPRESS = 13,
    EVT_READ         = 14,   // bio carrying read for this page was built
    EVT_READ_DONE    = 15,   // folio_end_read fired — data is in the page
};

struct page_event {
    u64 page_id;          // stable identity — survives migration
    u64 pfn;              // current physical frame number
    u64 ts;               // bpf_ktime_get_ns()
    u32 event_type;       // enum event_type
    u32 cpu;              // which CPU this happened on
    u32 tgid;             // process context (0 for kernel threads)
    u32 order;            // folio order (0 = 4KB, 4 = 64KB, 9 = 2MB)

    // event-specific fields — only relevant fields are set per event type
    union {
        struct {                  // EVT_ALLOC
            u64 gfp_flags;
            char comm[16];
        } alloc;
        struct {                  // EVT_CACHE_INSERT, EVT_CACHE_REMOVE
            u64 ino;
            u64 dev;
            u64 index;            // file offset in pages
        } cache;
        struct {                  // EVT_FAULT, EVT_ANON_FAULT
            u64 address;          // virtual address
            u8  is_write;
        } fault;
        struct {                  // EVT_COW
            u64 old_page_id;      // the page we copied from
            u64 address;
        } cow;
        struct {                  // EVT_MIGRATE
            u64 old_pfn;
            u64 new_pfn;
            u32 reason;           // MR_COMPACTION, MR_NUMA_MISPLACED, etc.
        } migrate;
        struct {                  // EVT_IO_ISSUE, EVT_IO_COMPLETE, EVT_READ, EVT_READ_DONE, EVT_WRITEBACK
            u64 sector;
            u64 dev;
            u32 nr_sectors;
            u32 error;            // 0 = success, nonzero = I/O error
        } io;
        struct {                  // EVT_ZRAM_COMPRESS
            u32 compressed_size;
        } zram;
    };
};
```

### Available stock tracepoints

These exist in any kernel with tracing enabled — no patches needed:

| Tracepoint | Fires when | Data available |
|---|---|---|
| `kmem/mm_page_alloc` | Page allocated | pfn, order, gfp_flags |
| `kmem/mm_page_free` | Page freed | pfn, order |
| `filemap/mm_filemap_add_to_page_cache` | Page enters page cache | pfn, i_ino, index, s_dev, order |
| `filemap/mm_filemap_delete_from_page_cache` | Page leaves page cache | pfn, i_ino, index, s_dev, order |
| `filemap/mm_filemap_fault` | File fault begins | i_ino, s_dev, index (no pfn!) |
| `block/block_rq_issue` | I/O sent to disk | dev, sector, nr_sectors |
| `block/block_rq_complete` | I/O completed | dev, sector, nr_sectors, error |

### Events with no stock tracepoint (need kprobes)

| Event | Function to hook | What you get |
|---|---|---|
| Page dirty | `filemap_dirty_folio` | folio pointer → pfn, mapping, index |
| Writeback done | `folio_end_writeback` | folio pointer → pfn |
| Anonymous fault | `do_anonymous_page` | vmf → address, folio → pfn |
| COW | `wp_page_copy` | old and new folio pointers |
| zRAM compress | `zram_write_page` | page, compressed size |
| Page migration | `migrate_folio_move` | old and new folio, reason |

### Helper: emitting an event

Every handler follows the same pattern — look up page_id from PFN, reserve space in the ring buffer, fill in the event, submit. We factor this into a helper:

```c
// Look up page_id for a given PFN. Returns 0 if not tracked.
static __always_inline u64 get_page_id(u64 pfn) {
    u64 *id = bpf_map_lookup_elem(&pfn_to_id, &pfn);
    return id ? *id : 0;
}

// Allocate the next page_id (called only from on_page_alloc).
static __always_inline u64 next_page_id(void) {
    u32 zero = 0;
    u64 *counter = bpf_map_lookup_elem(&page_id_counter, &zero);
    if (!counter) return 0;
    // __sync_fetch_and_add returns the value BEFORE incrementing.
    // We add 1 so page_ids start at 1, not 0 — because 0 means
    // "not tracked" in get_page_id().
    return __sync_fetch_and_add(counter, 1) + 1;
}

// Reserve + submit a page_event to the ring buffer.
// Returns a pointer to fill in event-specific fields, or NULL on failure.
static __always_inline struct page_event *emit(u64 page_id, u64 pfn,
                                                u32 type, u32 order) {
    struct page_event *e = bpf_ringbuf_reserve(&events, sizeof(*e), 0);
    if (!e) return NULL;

    // bpf_ringbuf_reserve does NOT zero memory. We must initialize
    // everything — including the union — to avoid leaking stale data
    // from a prior event that occupied this slot.
    __builtin_memset(e, 0, sizeof(*e));

    e->page_id = page_id;
    e->pfn = pfn;
    e->ts = bpf_ktime_get_ns();
    e->event_type = type;
    e->cpu = bpf_get_smp_processor_id();
    e->tgid = bpf_get_current_pid_tgid() >> 32;
    e->order = order;

    return e;  // caller fills in the union, then calls bpf_ringbuf_submit(e, 0)
}
```

### Helper: getting a PFN from a folio pointer

Several kprobe handlers need the PFN of a folio given only a `struct folio *`. This is not as simple as it sounds, because BPF programs cannot directly do the pointer arithmetic that `page_to_pfn` does in kernel C. Here is the concrete implementation.

The kernel macro `page_to_pfn(page)` boils down to `((page) - vmemmap)` — the page's offset in the global `vmemmap` array. On ARM64 with `CONFIG_SPARSEMEM_VMEMMAP=y` (the default), `vmemmap` is a fixed virtual address: `VMEMMAP_START`.

From BPF, the approach is:

1. At program init, userspace reads `/proc/kallsyms` to get the value of `vmemmap_base` (or the architecture's equivalent — on ARM64 there's no runtime symbol, but the address is constant and can be computed from `/proc/iomem` or hardcoded for the target).
2. Userspace stores this value in a BPF array map.
3. BPF handlers read `vmemmap` from the map and compute the PFN by subtraction.

```c
// Configuration map written by userspace at init
struct {
    __uint(type, BPF_MAP_TYPE_ARRAY);
    __uint(max_entries, 1);
    __type(key, u32);
    __type(value, u64);
} vmemmap_base_map SEC(".maps");

static __always_inline u64 folio_to_pfn_helper(struct folio *folio) {
    u32 zero = 0;
    u64 *vmemmap = bpf_map_lookup_elem(&vmemmap_base_map, &zero);
    if (!vmemmap || !*vmemmap) return 0;

    // folio and page share the same address (folio IS a page), so
    // the PFN is simply (folio - vmemmap) / sizeof(struct page).
    // Since struct page is a known size from BTF, we can compute this.
    u64 folio_addr = (u64)folio;
    if (folio_addr < *vmemmap) return 0;

    return (folio_addr - *vmemmap) / sizeof(struct page);
}
```

On a running kernel, userspace initializes the map like this:

```c
// Read vmemmap_base from /proc/kallsyms (requires CAP_SYSLOG) or
// compute from a known per-arch constant.
u64 vmemmap_addr = read_kallsyms_symbol("vmemmap_base");
u32 zero = 0;
int fd = bpf_map__fd(skel->maps.vmemmap_base_map);
bpf_map_update_elem(fd, &zero, &vmemmap_addr, BPF_ANY);
```

Two caveats:

- **`sizeof(struct page)` in BPF.** The verifier's BTF support lets you use `bpf_core_type_size(struct page)`, which gets resolved at load time from the kernel's BTF. Hardcoding 64 (the common size on 64-bit) is risky — the struct is occasionally 56 or 72 depending on config.
- **KASLR.** If the kernel is built with KASLR, `vmemmap_base` shifts on each boot. Userspace must read the current value from kallsyms; compile-time constants will not work.

**`sizeof(struct page)` at runtime.** Hardcoding `sizeof(struct page) = 64` is common but wrong on some configs. Use BTF: `bpf_core_type_size(struct page)` is resolved at load time by libbpf reading the kernel's BTF. It produces a constant in the compiled BPF code — no runtime cost. If unavailable, fall back to 64 as the common case on 64-bit systems.

**Android GKI specifics.** As of Android 14 (kernel 5.15) and Android 15 (kernel 6.1), `bpf_page_to_pfn` is not exported as a kfunc. The vmemmap math approach is the only option. On ARM64 with 48-bit VA and 4KB pages (the standard GKI configuration), `VMEMMAP_START` is a compile-time constant that shifts with KASLR. Userspace must read the runtime value from `/proc/kallsyms` (requires root on Android, which the tracer already needs for `bpf()` syscalls).

The cleanest alternative, when your kernel supports it, is to use the `bpf_page_to_pfn` kfunc (added in newer kernels). A kfunc is a kernel function explicitly exported for BPF, with verifier type checking. If available, it is both safer and simpler:

```c
extern unsigned long bpf_page_to_pfn(struct page *page) __ksym;

static __always_inline u64 folio_to_pfn_helper(struct folio *folio) {
    return bpf_page_to_pfn(&folio->page);
}
```

### Handler: page allocation (assigns page_id)

This is the only handler that creates a new page_id. Every other handler looks it up.

```c
SEC("tracepoint/kmem/mm_page_alloc")
int on_page_alloc(struct trace_event_raw_mm_page_alloc *ctx) {
    u64 pfn = ctx->pfn;
    if (pfn == (u64)-1) return 0;              // allocation failed

    // Only track user pages (__GFP_MOVABLE = 0x08)
    if (!(ctx->gfp_flags & 0x08)) return 0;

    // Check filter (tgid-only at this point — we don't know the file yet)
    u32 zero = 0;
    struct trace_filter *f = bpf_map_lookup_elem(&filter, &zero);
    if (!f || f->filter_type == 0) return 0;   // filter off → don't track
    if (f->filter_type == 1) {
        u32 tgid = bpf_get_current_pid_tgid() >> 32;
        if (tgid != f->tgid) return 0;
    }
    // For filter_type == 2 (by ino), we must track optimistically here
    // and clean up in on_cache_add if the inode doesn't match.

    // Assign a new stable page_id
    u64 page_id = next_page_id();
    if (!page_id) return 0;

    // Register in the lookup map
    bpf_map_update_elem(&pfn_to_id, &pfn, &page_id, BPF_ANY);

    // Emit event
    struct page_event *e = emit(page_id, pfn, EVT_ALLOC, ctx->order);
    if (!e) return 0;

    e->alloc.gfp_flags = ctx->gfp_flags;
    bpf_get_current_comm(&e->alloc.comm, sizeof(e->alloc.comm));
    bpf_ringbuf_submit(e, 0);
    return 0;
}
```

**A note on large folios.** When `order > 0`, the tracepoint reports the PFN of the first page in the compound allocation. We register that PFN in `pfn_to_id`. Subsequent tracepoints (e.g., `mm_filemap_add_to_page_cache`) also report the first-page PFN for the folio, so the lookup works correctly. However, if something references an *interior* page of a large folio by its PFN, our lookup would miss it. In practice this is not a problem: the mm subsystem consistently uses the folio head page's PFN in tracepoints and function arguments. The `order` field in the event tells userspace how many physical pages this folio spans.

### Handler: page enters page cache

```c
SEC("tracepoint/filemap/mm_filemap_add_to_page_cache")
int on_cache_add(struct trace_event_raw_mm_filemap_op_page_cache *ctx) {
    u64 pfn = ctx->pfn;
    u64 page_id = get_page_id(pfn);
    if (!page_id) return 0;

    struct page_event *e = emit(page_id, pfn, EVT_CACHE_INSERT, ctx->order);
    if (!e) return 0;

    e->cache.ino = ctx->i_ino;
    e->cache.dev = ctx->s_dev;
    e->cache.index = ctx->index;
    bpf_ringbuf_submit(e, 0);
    return 0;
}
```

### Handler: page freed (cleans up lookup map)

```c
SEC("tracepoint/kmem/mm_page_free")
int on_page_free(struct trace_event_raw_mm_page_free *ctx) {
    u64 pfn = ctx->pfn;
    u64 page_id = get_page_id(pfn);
    if (!page_id) return 0;

    struct page_event *e = emit(page_id, pfn, EVT_FREE, ctx->order);
    if (!e) goto cleanup;

    bpf_ringbuf_submit(e, 0);

cleanup:
    // Remove from lookup map — this PFN is no longer tracked
    bpf_map_delete_elem(&pfn_to_id, &pfn);
    return 0;
}
```

### Handler: page migration (the critical one)

This is what keeps everything coherent. When a page migrates from one PFN to another, the page_id stays the same — we just update the lookup map to point the new PFN to the existing page_id.

```c
SEC("kprobe/migrate_folio_move")
int on_migrate(struct pt_regs *ctx) {
    // migrate_folio_move(put_new_folio, private, src, dst, mode, reason, ret)
    // src is arg3, dst is arg4 on ARM64
    struct folio *src = (struct folio *)PT_REGS_PARM3(ctx);
    struct folio *dst = (struct folio *)PT_REGS_PARM4(ctx);
    int reason = (int)PT_REGS_PARM6(ctx);

    // Getting the PFN from a struct folio* in BPF.
    //
    // In the kernel, page_to_pfn() uses the page's position in the
    // mem_map array: pfn = page - mem_map. But from BPF we cannot do
    // pointer arithmetic on kernel addresses that way.
    //
    // The practical approach: use a kfunc or read the PFN from a field
    // that the kernel exposes. On kernels with CONFIG_DEBUG_INFO_BTF,
    // you can use bpf_page_to_pfn() if available as a kfunc, or use
    // a tracepoint-based approach instead of a kprobe.
    //
    // The cleanest alternative: attach to a kretprobe on
    // folio_migrate_mapping() and read the PFNs from its arguments,
    // or add a custom tracepoint that emits both PFNs directly.
    //
    // For illustration, assume we have a helper that extracts the PFN:
    unsigned long old_pfn = folio_to_pfn_helper(src);
    unsigned long new_pfn = folio_to_pfn_helper(dst);

    // Look up the page_id using the OLD PFN
    u64 page_id = get_page_id(old_pfn);
    if (!page_id) return 0;

    // Update the lookup map: delete old PFN, insert new PFN
    bpf_map_delete_elem(&pfn_to_id, &old_pfn);
    bpf_map_update_elem(&pfn_to_id, &new_pfn, &page_id, BPF_ANY);

    // Emit the migration event
    struct page_event *e = emit(page_id, new_pfn, EVT_MIGRATE, 0);
    if (!e) return 0;

    e->migrate.old_pfn = old_pfn;
    e->migrate.new_pfn = new_pfn;
    e->migrate.reason = reason;
    bpf_ringbuf_submit(e, 0);
    return 0;
}
```

**Getting a PFN from a folio pointer in BPF — the hard part.**

In kernel C, `folio_pfn(folio)` is a macro that boils down to `page_to_pfn(&folio->page)`, which computes the page's position in the vmemmap array. From BPF, you cannot directly do this pointer arithmetic because `vmemmap` is a kernel symbol and BPF pointer arithmetic on kernel addresses is restricted by the verifier.

There are three practical approaches:

1. **Use a BPF kfunc.** If your kernel exports `bpf_page_to_pfn` as a kfunc (or you add one), you can call it directly from BPF. This is the cleanest path but requires kernel support.
2. **Read `vmemmap_base` from a BPF global and do the math.** On x86-64, `vmemmap_base` is a well-known address; on ARM64, it is `VMEMMAP_START`. You can store it in a BPF array map at init time and compute `pfn = (folio_addr - vmemmap_base) / sizeof(struct page)`. This works but is architecture-specific.
3. **Avoid kprobes for migration entirely.** Instead of hooking `migrate_folio_move`, hook the tracepoints that fire *around* migration — `mm_page_alloc` for the new page and `mm_page_free` for the old page. The new page gets a fresh PFN that you track normally. The cost is losing the explicit link between old and new page. For many workloads this is acceptable.

Without this handler, any event after migration would fail the `get_page_id(new_pfn)` lookup and be silently dropped. The page would appear to vanish mid-lifecycle. With it, the stream of events for a migrated page is seamless — the page_id ties pre-migration and post-migration events together, and the EVT_MIGRATE event itself records when and why the physical address changed.

### Handler: dirty (kprobe, since no stock tracepoint carries PFN)

The stock `writeback_dirty_folio` tracepoint exists but does not carry the PFN — it only has `(name, ino, index)`. We need the PFN to look up the page_id, so we hook `filemap_dirty_folio` directly.

```c
SEC("kprobe/filemap_dirty_folio")
int on_dirty(struct pt_regs *ctx) {
    // filemap_dirty_folio(mapping, folio) — folio is arg2
    struct folio *folio = (struct folio *)PT_REGS_PARM2(ctx);

    // We face the same PFN extraction problem as the migration handler.
    // The same three approaches apply (kfunc, vmemmap math, or avoidance).
    //
    // An alternative specific to dirty tracking: instead of hooking
    // filemap_dirty_folio, hook the PTE-level dirty bit setting in
    // handle_pte_fault (line 6335 in mm/memory.c), where pte_mkdirty()
    // is called. At that point we have the PTE, and pte_pfn() gives us
    // the PFN directly. The cost is that we would need to distinguish
    // first-dirty from re-dirty, which the PTE path does not naturally
    // expose (both just set the dirty bit).
    //
    // For now, assume we can extract the PFN (via kfunc or vmemmap):
    unsigned long pfn = folio_to_pfn_helper(folio);

    u64 page_id = get_page_id(pfn);
    if (!page_id) return 0;

    struct page_event *e = emit(page_id, pfn, EVT_DIRTY, 0);
    if (!e) return 0;
    bpf_ringbuf_submit(e, 0);
    return 0;
}
```

### Handler: page fault (covers both file-backed and anonymous)

Rather than hooking `do_anonymous_page` and `filemap_fault` separately, hook `finish_fault` — it is called for all successful fault paths (`do_read_fault`, `do_cow_fault`, `do_shared_fault`, and `do_anonymous_page` all converge on it). The `vm_file` check distinguishes file faults from anonymous faults:

```c
SEC("kprobe/finish_fault")
int on_fault(struct pt_regs *ctx) {
    struct vm_fault *vmf = (struct vm_fault *)PT_REGS_PARM1(ctx);

    struct page *page = BPF_CORE_READ(vmf, page);
    if (!page) return 0;

    unsigned long pfn = folio_to_pfn_helper(page_folio(page));
    u64 page_id = get_page_id(pfn);
    if (!page_id) return 0;

    unsigned long address = BPF_CORE_READ(vmf, address);
    unsigned int flags = BPF_CORE_READ(vmf, flags);
    struct vm_area_struct *vma = BPF_CORE_READ(vmf, vma);
    struct file *vm_file = BPF_CORE_READ(vma, vm_file);

    u32 evt_type = vm_file ? EVT_FAULT : EVT_ANON_FAULT;

    struct page_event *e = emit(page_id, pfn, evt_type, 0);
    if (!e) return 0;

    e->fault.address = address;
    e->fault.is_write = !!(flags & FAULT_FLAG_WRITE);
    bpf_ringbuf_submit(e, 0);
    return 0;
}
```

Caveat: `finish_fault` is called with the PTE lock held (it is about to install the PTE). The kprobe runs inside the lock. This is safe — kprobes and eBPF run in the same context as the hooked function — but it means the eBPF program adds latency to the PTE-locked critical section. Keep the handler fast.

### Handler: copy-on-write

When `wp_page_copy` fires, the kernel is copying a shared page for one process. The tracer needs to link the new page (being allocated for the writer) to the old page (being copied from). The approach: hook `wp_page_copy` entry to record the old page_id in a per-tgid scratch map, then check that map in `on_page_alloc` to emit `EVT_COW` instead of `EVT_ALLOC`:

```c
SEC("kprobe/wp_page_copy")
int on_cow_entry(struct pt_regs *ctx) {
    struct vm_fault *vmf = (struct vm_fault *)PT_REGS_PARM1(ctx);

    struct page *old_page = BPF_CORE_READ(vmf, page);
    if (!old_page) return 0;

    unsigned long old_pfn = folio_to_pfn_helper(page_folio(old_page));
    u64 old_page_id = get_page_id(old_pfn);

    u32 tgid = bpf_get_current_pid_tgid() >> 32;
    struct cow_pending val = {
        .old_page_id = old_page_id,
        .address = BPF_CORE_READ(vmf, address),
    };
    bpf_map_update_elem(&pending_cow, &tgid, &val, BPF_ANY);
    return 0;
}
```

Then in `on_page_alloc`, after assigning the new page_id:

```c
// Check if this allocation is a COW copy
u32 tgid = bpf_get_current_pid_tgid() >> 32;
struct cow_pending *cow = bpf_map_lookup_elem(&pending_cow, &tgid);
if (cow) {
    struct page_event *e = emit(page_id, pfn, EVT_COW, ctx->order);
    if (e) {
        e->cow.old_page_id = cow->old_page_id;
        e->cow.address = cow->address;
        bpf_ringbuf_submit(e, 0);
    }
    bpf_map_delete_elem(&pending_cow, &tgid);
    return 0;  // skip normal EVT_ALLOC
}
```

This gives a single `EVT_COW` event with both the new `page_id` (assigned by `on_page_alloc`) and the `old_page_id` (from `pending_cow`). Pure ID-based linking, no timestamp correlation.

### Handler: every other event

Every remaining handler follows the same pattern: get PFN from tracepoint/kprobe args → look up page_id → emit event. The pattern is always three lines of boilerplate plus the event-specific fields.

### Tracing the I/O path: from page to sector and back

The block I/O tracepoints (`block_rq_issue`, `block_rq_complete`) give you sector numbers, not PFNs. To connect an I/O event to a page_id, you need to record the `(dev, sector) → page_id` mapping *at bio construction time*, then look it up when the block tracepoints fire. Here is how to build this end-to-end.

**Step 1: the sector map.**

```c
// sector_to_id: (device, sector) → page_id
// Populated when bio is constructed; consulted when block layer tracepoints fire.
struct sector_key {
    u64 dev;
    u64 sector;
};

struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 16384);
    __type(key, struct sector_key);
    __type(value, u64);            // page_id
} sector_to_id SEC(".maps");
```

**Step 2: populate the map when the filesystem builds the bio.**

The point to hook is `bio_add_folio`, called by the filesystem (e.g., `mpage_readahead` or `mpage_writepages`) each time it adds a page to a bio. At that moment, we know both the folio (which gives us the PFN → page_id) and the bio's current sector:

```c
SEC("kprobe/bio_add_folio")
int on_bio_add_folio(struct pt_regs *ctx) {
    // bio_add_folio(bio, folio, len, off) — args in x0, x1, x2, x3 on ARM64
    struct bio *bio = (struct bio *)PT_REGS_PARM1(ctx);
    struct folio *folio = (struct folio *)PT_REGS_PARM2(ctx);

    // Extract PFN from folio (same challenge as elsewhere — see Part 16
    // notes on folio_to_pfn_helper)
    unsigned long pfn = folio_to_pfn_helper(folio);
    u64 page_id = get_page_id(pfn);
    if (!page_id) return 0;

    // Read the bio's current sector and device
    u64 sector = BPF_CORE_READ(bio, bi_iter.bi_sector);
    struct block_device *bdev = BPF_CORE_READ(bio, bi_bdev);
    dev_t dev = BPF_CORE_READ(bdev, bd_dev);
    u32 opf = BPF_CORE_READ(bio, bi_opf);

    // Record the sector → page_id mapping so on_rq_issue can find this page
    struct sector_key key = { .dev = dev, .sector = sector };
    bpf_map_update_elem(&sector_to_id, &key, &page_id, BPF_ANY);

    // Also emit a "bio submit" event so userspace knows when the page
    // was handed to the block layer (distinct from when I/O actually issues)
    u32 evt_type = (opf & REQ_OP_MASK) == REQ_OP_WRITE ? EVT_WRITEBACK : EVT_READ;
    struct page_event *e = emit(page_id, pfn, evt_type, 0);
    if (!e) return 0;

    e->io.sector = sector;
    e->io.dev = dev;
    bpf_ringbuf_submit(e, 0);
    return 0;
}
```

**Step 3: catch request issue.**

When the block layer actually dispatches a request to the driver, `block_rq_issue` fires. The tracepoint gives us `(dev, sector, nr_sectors)`. We look up page_ids for each 4KB sub-block in the request's sector range:

```c
SEC("tracepoint/block/block_rq_issue")
int on_rq_issue(struct trace_event_raw_block_rq_issue *ctx) {
    // One request can cover many pages. We emit an event for each page
    // we are tracking in this sector range.
    u64 dev = ctx->dev;
    u64 sector = ctx->sector;
    u32 nr_sectors = ctx->nr_sector;

    // Each page is 8 sectors (4KB / 512). Walk the range.
    #pragma unroll
    for (int i = 0; i < 64; i++) {   // verifier needs a bounded loop
        if (i * 8 >= nr_sectors) break;

        struct sector_key key = { .dev = dev, .sector = sector + i * 8 };
        u64 *page_id_ptr = bpf_map_lookup_elem(&sector_to_id, &key);
        if (!page_id_ptr) continue;

        struct page_event *e = emit(*page_id_ptr, 0, EVT_IO_ISSUE, 0);
        if (!e) continue;

        e->io.sector = sector + i * 8;
        e->io.dev = dev;
        e->io.nr_sectors = 8;
        bpf_ringbuf_submit(e, 0);
    }
    return 0;
}
```

The `#pragma unroll` hint and bounded loop satisfy the verifier's termination requirement. Cap it at the largest request size you expect (64 pages = 256KB, which covers most UFS/NVMe workloads).

**Step 4: catch request completion.**

`block_rq_complete` fires when the hardware reports the I/O is done. Same lookup, plus cleanup:

```c
SEC("tracepoint/block/block_rq_complete")
int on_rq_complete(struct trace_event_raw_block_rq_complete *ctx) {
    u64 dev = ctx->dev;
    u64 sector = ctx->sector;
    u32 nr_sectors = ctx->nr_sector;
    int error = ctx->error;

    #pragma unroll
    for (int i = 0; i < 64; i++) {
        if (i * 8 >= nr_sectors) break;

        struct sector_key key = { .dev = dev, .sector = sector + i * 8 };
        u64 *page_id_ptr = bpf_map_lookup_elem(&sector_to_id, &key);
        if (!page_id_ptr) continue;

        struct page_event *e = emit(*page_id_ptr, 0, EVT_IO_COMPLETE, 0);
        if (!e) {
            bpf_map_delete_elem(&sector_to_id, &key);
            continue;
        }

        e->io.sector = sector + i * 8;
        e->io.dev = dev;
        e->io.nr_sectors = 8;
        bpf_ringbuf_submit(e, 0);

        // Clean up — this sector is done, no more events expected for it
        bpf_map_delete_elem(&sector_to_id, &key);
    }
    return 0;
}
```

**Step 5: writeback completion.**

For write I/O, the `EVT_IO_COMPLETE` event tells us the hardware finished writing. But we also want to know when `folio_end_writeback` runs — because only then is the folio marked clean and reclaimable. This is the *logical* end of writeback, distinct from the hardware completion:

```c
SEC("kprobe/folio_end_writeback")
int on_wb_done(struct pt_regs *ctx) {
    struct folio *folio = (struct folio *)PT_REGS_PARM1(ctx);
    unsigned long pfn = folio_to_pfn_helper(folio);

    u64 page_id = get_page_id(pfn);
    if (!page_id) return 0;

    struct page_event *e = emit(page_id, pfn, EVT_WB_DONE, 0);
    if (!e) return 0;
    bpf_ringbuf_submit(e, 0);
    return 0;
}
```

### Read I/O: catching cache-miss faults

The read path is subtler. When userspace reads a file page that is already in the page cache, there is no I/O — just a memcpy. The events we want to capture are specifically the *cache miss* path, where readahead allocates new pages and submits a bio.

The good news: the read path converges on the same functions we are already hooking:

- `on_page_alloc` fires when readahead allocates a folio.
- `on_cache_add` fires when `filemap_add_folio` inserts it.
- `on_bio_add_folio` fires when `mpage_readahead` adds the folio to its bio.
- `on_rq_issue` and `on_rq_complete` fire as the bio goes through the block layer.
- When the read completes, `folio_end_read` (at `mm/filemap.c:1529`) is called. Hook it for an `EVT_READ_DONE` event.

```c
SEC("kprobe/folio_end_read")
int on_read_done(struct pt_regs *ctx) {
    struct folio *folio = (struct folio *)PT_REGS_PARM1(ctx);
    bool success = PT_REGS_PARM2(ctx);
    unsigned long pfn = folio_to_pfn_helper(folio);

    u64 page_id = get_page_id(pfn);
    if (!page_id) return 0;

    struct page_event *e = emit(page_id, pfn, EVT_READ_DONE, 0);
    if (!e) return 0;

    e->io.error = success ? 0 : 1;
    bpf_ringbuf_submit(e, 0);
    return 0;
}
```

With these handlers, the full read lifecycle shows up in the event stream:

```
[t=0]     EVT_ALLOC       page_id=42 pfn=0x1234 order=0 comm=app
[t=2µs]   EVT_CACHE_INSERT page_id=42 ino=8517 index=20
[t=5µs]   EVT_READ        page_id=42 dev=8:16 sector=1048
[t=8µs]   EVT_IO_ISSUE    page_id=42 sector=1048 nr_sectors=8
[t=412µs] EVT_IO_COMPLETE page_id=42 sector=1048
[t=415µs] EVT_READ_DONE   page_id=42
[t=418µs] EVT_FAULT       page_id=42 address=0x7f00... tgid=1234
```

The gaps tell the story: `ALLOC → CACHE_INSERT` is the page cache insertion (microseconds), `READ → IO_ISSUE` is block layer queuing, `IO_ISSUE → IO_COMPLETE` is the hardware latency, and `READ_DONE → FAULT` is the time to install the PTE after data arrived.

### Write I/O: from dirty to durable

For writes, the sequence is richer because writeback can happen long after the write, and pages can be re-dirtied during writeback:

```
[t=0]      EVT_FAULT       page_id=42 address=0x7f00... tgid=1234 is_write=1
[t=100ns]  EVT_DIRTY       page_id=42
   ...
[t=5s]     EVT_WRITEBACK   page_id=42 dev=8:16 sector=1048
[t=5s+1µs] EVT_IO_ISSUE    page_id=42 sector=1048 nr_sectors=8
[t=5s+500µs] EVT_IO_COMPLETE page_id=42 sector=1048
[t=5s+501µs] EVT_WB_DONE   page_id=42
```

Userspace can compute:
- **Dirty queue time** = WRITEBACK.ts - DIRTY.ts (governed by `dirty_expire_centisecs`).
- **Block queue time** = IO_ISSUE.ts - WRITEBACK.ts (how long the bio sat in the block layer).
- **Hardware latency** = IO_COMPLETE.ts - IO_ISSUE.ts (pure device time).
- **Writeback settle** = WB_DONE.ts - IO_COMPLETE.ts (time to clear PG_writeback and wake waiters).

If the same page_id produces *another* DIRTY event between WB_DONE and the next allocation/free event, it means userspace re-dirtied a page that was already written back. This is a common pattern for SQLite: the same page (e.g., the rollback journal header) is written over and over, each cycle showing up as a separate DIRTY → WB_DONE pair in the stream.

### Why we need the bio-add hook, not just the block tracepoints

You might ask: why bother with the `bio_add_folio` kprobe at all? Why not just attach to `block_rq_issue` and look up the page from the sector?

The problem is that `block_rq_issue` has the sector but has no connection back to the page. The mapping from sector to page_id *must* be recorded before the tracepoint fires, and the only point where both pieces are known together is at bio construction time (inside the filesystem's writepages/readahead callback). `bio_add_folio` is that point.

An alternative worth mentioning: some kernels expose `trace_block_bio_queue` which fires when `submit_bio` is called. It carries the bio pointer, so a kprobe-like approach could walk the bio's `bi_io_vec` and record the sector for each page. The `bio_add_folio` kprobe is simpler because it handles one page at a time.

### A note on plugging

The block layer's plug mechanism means bios submitted within a `blk_start_plug`/`blk_finish_plug` section are deferred. Your `on_bio_add_folio` fires early; your `on_rq_issue` fires later when the plug is flushed. This is fine — the sector map keeps the mapping alive across the gap. But if you are computing "time from dirty to dispatch," the gap between `EVT_WRITEBACK` and `EVT_IO_ISSUE` includes this plug time, which can be several milliseconds on a busy system.

### Large folio handling

When the kernel allocates a large folio (order > 0), the tracer assigns one `page_id` for the entire folio. A single order-4 folio (16 contiguous pages, 64KB) gets one page_id, one `pfn_to_id` entry for the head page's PFN, and one EVT_ALLOC event with `order=4`.

Tracepoints and kprobes consistently report the head page's PFN for large folios — `mm_page_alloc`, `mm_filemap_add_to_page_cache`, and all the kprobe targets receive the folio pointer (which IS the head page). The `get_page_id(pfn)` lookup works because the `pfn_to_id` map contains the head PFN.

The complication is `bio_add_folio` for large folios: the bio may cover the full 64KB, meaning the `sector_to_id` map needs an entry for every 4KB-aligned sector in the range. The `on_bio_add_folio` handler populates `sector_to_id` for every 4KB chunk:

```c
// Inside on_bio_add_folio, for large folios:
u32 nr_pages = len / 4096;
if (nr_pages == 0) nr_pages = 1;
if (nr_pages > 16) nr_pages = 16;  // bound for verifier

#pragma unroll
for (u32 i = 0; i < 16; i++) {
    if (i >= nr_pages) break;
    struct sector_key key = { .dev = dev, .sector = sector + i * 8 };
    bpf_map_update_elem(&sector_to_id, &key, &page_id, BPF_ANY);
}
```

Every event carries `order`. Userspace knows that a page_id with `order=4` represents 64KB. Latency calculations are per-folio, not per-4KB-page. The kernel treats the folio as a unit for I/O, reclaim, and dirty tracking.

### Swap-in / swap-out correlation

When an anonymous page is swapped out to zRAM and later swapped back in, the old page_id (pre-swap) and new page_id (post-swap) have no link. The PFN changed. The page_id was assigned fresh at allocation.

The link is the **swap entry** — a `(type, offset)` pair stored in the PTE after the page is swapped out. To correlate, the tracer maintains a `swap_to_id` map:

```c
struct {
    __uint(type, BPF_MAP_TYPE_LRU_HASH);  // LRU to avoid filling up
    __uint(max_entries, 65536);
    __type(key, u64);       // swap entry value
    __type(value, u64);     // old page_id
} swap_to_id SEC(".maps");
```

At swap-out time (inside a `swap_writepage` kprobe), before the page is freed, read `folio->swap` and store `swap_entry → old_page_id`. At swap-in time (`do_swap_page` kprobe), read the swap entry from the non-present PTE, look up `swap_to_id`, and emit an `EVT_SWAP_IN` linking old and new page_ids.

This is the single most fragile part of the tracer. The swap entry encoding is architecture-specific (different bit layouts on ARM64 vs x86-64). The `swap_to_id` map has finite size and can fill on long traces with heavy swap. If swap correlation is not needed, omit it — the rest of the tracer works without it. Swap-out shows as a normal lifecycle (`alloc → fault → zram_compress → free`). Swap-in shows as a fresh lifecycle (`alloc → fault`). They just aren't linked.

### File filter resolution

The user specifies `--file "/usr/lib/*.so"`. Userspace resolves this to `(ino, dev)` pairs using `glob()` + `stat()` and populates the `tracked_inodes` BPF map before attaching programs:

```c
static int resolve_file_filter(const char *pattern, int map_fd) {
    glob_t g = {0};
    int ret = glob(pattern, GLOB_TILDE | GLOB_BRACE, NULL, &g);
    if (ret == GLOB_NOMATCH) {
        fprintf(stderr, "error: no files match '%s'\n", pattern);
        return -1;
    }

    int count = 0;
    for (size_t i = 0; i < g.gl_pathc; i++) {
        struct stat st;
        if (stat(g.gl_pathv[i], &st) < 0) continue;
        if (!S_ISREG(st.st_mode)) continue;

        struct inode_key key = { .ino = st.st_ino, .dev = st.st_dev };
        u8 val = 1;
        bpf_map_update_elem(map_fd, &key, &val, BPF_ANY);
        count++;
    }
    globfree(&g);
    return count;
}
```

On Android, useful patterns:

```bash
page_tracer --file "/data/app/~~*/com.example.app-*/base.apk"
page_tracer --file "/data/dalvik-cache/arm64/*.odex"
page_tracer --file "/apex/com.android.runtime/lib64/bionic/libc.so"
```

The glob handles the `~~*` random directory suffix Android uses for APK installation paths.

### The userspace program: a production-quality implementation

The BPF side is only half the tracer. The userspace program is what loads the BPF program, attaches it to hooks, configures filters, drains the ring buffer, reconstructs lifecycles, and writes output. This section walks through a full implementation with every concern addressed: argument parsing, filter configuration, vmemmap discovery, event dispatch, lifecycle reconstruction, JSON output, signal handling, and statistics.

The code is split into four logical components. They can be separate files or one big file; real implementations tend to grow them into separate modules.

#### Component 1: loading and attaching

The skeleton generated by `bpftool gen skeleton` gives you a typed handle to every map and program. The skeleton's `open_and_load` allocates BPF objects (maps and programs), runs the verifier on each program, and JIT-compiles them. `attach` links each program to its tracepoint or kprobe.

```c
// page_tracer_user.c
#include <argp.h>
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <time.h>
#include <errno.h>
#include <sys/resource.h>
#include <bpf/libbpf.h>
#include <bpf/bpf.h>
#include "page_tracer.bpf.skel.h"

static volatile sig_atomic_t exiting;
static void sig_handler(int sig) { exiting = 1; }

// Bump RLIMIT_MEMLOCK — required for loading BPF programs on older kernels
// (kernels >= 5.11 use a different accounting scheme but the bump is harmless).
static void bump_memlock(void) {
    struct rlimit r = { RLIM_INFINITY, RLIM_INFINITY };
    if (setrlimit(RLIMIT_MEMLOCK, &r))
        fprintf(stderr, "warning: failed to set RLIMIT_MEMLOCK: %s\n",
                strerror(errno));
}

// libbpf logging callback — route libbpf's diagnostic messages through
// fprintf so we can gate them on a verbose flag.
static int libbpf_log(enum libbpf_print_level level, const char *fmt, va_list args) {
    if (level == LIBBPF_DEBUG && !getenv("PAGE_TRACER_DEBUG"))
        return 0;
    return vfprintf(stderr, fmt, args);
}
```

#### Component 2: filter and vmemmap setup

Before we attach programs, we need to write the configuration maps. The filter map tells the BPF handlers whose pages to track; the vmemmap base map is what `folio_to_pfn_helper` uses. Both must be populated before any event fires, otherwise the handlers see a zero filter and track nothing (or track everything, depending on the convention you picked in the BPF code).

```c
// Configuration struct matching what the BPF code expects
struct cli_opts {
    int filter_type;        // 0=off, 1=tgid, 2=ino
    int tgid;
    unsigned long ino;
    unsigned long dev;
    int verbose;
    const char *json_path;
};

// Read the kernel's vmemmap base from /proc/kallsyms.
// Requires CAP_SYSLOG (or running as root) because kallsyms is restricted
// under kptr_restrict=2.
static unsigned long read_vmemmap_base(void) {
    FILE *f = fopen("/proc/kallsyms", "r");
    if (!f) {
        perror("open /proc/kallsyms");
        return 0;
    }
    char line[256];
    unsigned long addr = 0;
    while (fgets(line, sizeof(line), f)) {
        unsigned long a;
        char type;
        char name[128];
        if (sscanf(line, "%lx %c %127s", &a, &type, name) == 3) {
            // On x86, the symbol is "vmemmap_base". On ARM64, vmemmap
            // is a fixed constant computed from PAGE_OFFSET and
            // VMEMMAP_SHIFT — we extract it from "vmemmap" if present,
            // otherwise compute from known constants.
            if (strcmp(name, "vmemmap_base") == 0 ||
                strcmp(name, "vmemmap") == 0) {
                addr = a;
                break;
            }
        }
    }
    fclose(f);
    return addr;
}

static int configure_maps(struct page_tracer_bpf *skel, struct cli_opts *opts) {
    uint32_t zero = 0;

    // Filter map
    struct trace_filter filter = {
        .filter_type = opts->filter_type,
        .tgid = opts->tgid,
        .ino = opts->ino,
        .dev = opts->dev,
    };
    if (bpf_map__update_elem(skel->maps.filter,
                             &zero, sizeof(zero),
                             &filter, sizeof(filter),
                             BPF_ANY) < 0) {
        fprintf(stderr, "failed to write filter: %s\n", strerror(errno));
        return -1;
    }

    // page_id counter starts at 0 (the BPF increment adds 1 before returning)
    uint64_t counter = 0;
    if (bpf_map__update_elem(skel->maps.page_id_counter,
                             &zero, sizeof(zero),
                             &counter, sizeof(counter),
                             BPF_ANY) < 0) {
        fprintf(stderr, "failed to init page_id counter\n");
        return -1;
    }

    // vmemmap base
    unsigned long vmemmap = read_vmemmap_base();
    if (!vmemmap) {
        fprintf(stderr, "could not determine vmemmap_base; "
                        "folio_to_pfn_helper will return 0 for everything\n");
        fprintf(stderr, "is /proc/kallsyms accessible? "
                        "(check /proc/sys/kernel/kptr_restrict)\n");
        return -1;
    }
    if (bpf_map__update_elem(skel->maps.vmemmap_base_map,
                             &zero, sizeof(zero),
                             &vmemmap, sizeof(vmemmap),
                             BPF_ANY) < 0) {
        fprintf(stderr, "failed to write vmemmap_base\n");
        return -1;
    }
    if (opts->verbose)
        fprintf(stderr, "vmemmap_base = 0x%lx\n", vmemmap);

    return 0;
}
```

Two points:

- **We write the filter before `attach`.** If we attached first, events would start flowing with a zero filter, the BPF code would early-return on every event, and we would miss the first few milliseconds of activity. Writing the filter first ensures there is no race.
- **kptr_restrict matters.** On most distributions, `/proc/kallsyms` returns zeroed addresses unless the caller has `CAP_SYSLOG`. If you run the tracer as a non-root user with insufficient capabilities, `read_vmemmap_base` returns 0 and the tracer silently fails. The verbose warning helps debug this.

#### Component 3: event handling and lifecycle reconstruction

The ring buffer callback is where you decide what the tracer does. The simplest use is to print each event as it arrives (live tail). A richer use is to accumulate events per page_id and emit a complete lifecycle record when the page is freed.

```c
// Per-page lifecycle record that we build up from the event stream.
// Indexed by page_id in a hashmap.
struct lifecycle {
    uint64_t page_id;
    uint64_t pfn;
    uint32_t order;

    // Key timestamps (0 = not yet observed)
    uint64_t alloc_ts;
    uint64_t cache_insert_ts;
    uint64_t first_fault_ts;
    uint64_t first_dirty_ts;
    uint64_t last_wb_start_ts;
    uint64_t last_wb_done_ts;
    uint64_t free_ts;

    // Identifying info filled in over the lifecycle
    uint64_t ino;
    uint64_t dev;
    uint64_t file_offset;
    uint32_t tgid;
    char comm[16];
    uint8_t  page_type;        // 0=unknown, 1=file, 2=anon, 3=shmem

    // Counters that can increment multiple times per page
    uint32_t dirty_count;       // how many dirty→wb_done cycles
    uint32_t migrate_count;
    uint32_t io_count;

    // COW parent chain
    uint64_t cow_parent_id;
};

// Hashmap keyed by page_id — we use a simple open-addressing hashmap
// or an stdc++ unordered_map in a real implementation. Sketched here.
struct lifecycle_map { /* ... */ };
static struct lifecycle_map *lm;

static FILE *json_out;   // optional JSON output stream
static uint64_t events_seen, events_dropped_at_handler;
```

The event handler dispatches on `event_type` and updates the corresponding `struct lifecycle`. When it sees `EVT_FREE`, the lifecycle is complete — it emits a JSON record and removes the entry from the map:

```c
static int handle_event(void *ctx, void *data, size_t len) {
    if (len < sizeof(struct page_event)) {
        events_dropped_at_handler++;
        return 0;                   // truncated — skip
    }
    const struct page_event *e = data;
    events_seen++;

    struct lifecycle *lc = lifecycle_map_get_or_create(lm, e->page_id);
    lc->page_id = e->page_id;
    lc->pfn = e->pfn;

    switch (e->event_type) {
    case EVT_ALLOC:
        lc->alloc_ts = e->ts;
        lc->order = e->order;
        lc->tgid = e->tgid;
        memcpy(lc->comm, e->alloc.comm, sizeof(lc->comm));
        break;

    case EVT_CACHE_INSERT:
        if (!lc->cache_insert_ts) lc->cache_insert_ts = e->ts;
        lc->ino = e->cache.ino;
        lc->dev = e->cache.dev;
        lc->file_offset = e->cache.index;
        lc->page_type = 1;   // file-backed
        break;

    case EVT_FAULT:
        if (!lc->first_fault_ts) lc->first_fault_ts = e->ts;
        if (!lc->tgid) lc->tgid = e->tgid;
        break;

    case EVT_ANON_FAULT:
        if (!lc->first_fault_ts) lc->first_fault_ts = e->ts;
        lc->page_type = 2;   // anonymous
        // For anon pages, fault IS the dirty event
        if (!lc->first_dirty_ts) lc->first_dirty_ts = e->ts;
        break;

    case EVT_DIRTY:
        if (!lc->first_dirty_ts) lc->first_dirty_ts = e->ts;
        lc->dirty_count++;
        break;

    case EVT_WRITEBACK:
    case EVT_IO_ISSUE:
        lc->last_wb_start_ts = e->ts;
        lc->io_count++;
        break;

    case EVT_WB_DONE:
    case EVT_IO_COMPLETE:
        lc->last_wb_done_ts = e->ts;
        break;

    case EVT_COW:
        lc->cow_parent_id = e->cow.old_page_id;
        lc->page_type = 2;
        break;

    case EVT_MIGRATE:
        lc->migrate_count++;
        // pfn already updated via the common e->pfn field
        break;

    case EVT_FREE:
        lc->free_ts = e->ts;
        emit_lifecycle_record(lc);
        lifecycle_map_delete(lm, e->page_id);
        break;

    default:
        break;
    }

    return 0;
}
```

Notice the handler follows a specific discipline:

- **Timestamps use `if (!lc->foo_ts) lc->foo_ts = e->ts;`** for "first observed" timestamps and bare assignment for "last observed" ones. This is explicit so you know which semantic you are computing later.
- **Counters** (`dirty_count`, `migrate_count`, `io_count`) are strict increments. They let you ask "how many writeback cycles did this page go through?" without ambiguity.
- **No `free_ts` overrides anything.** If a page_id sees `EVT_FREE` twice (it shouldn't), we still only emit one record and the second `FREE` looks up an empty entry and does nothing.

The emit function formats the lifecycle as one JSON line (NDJSON — newline-delimited JSON — is ideal for downstream tooling like `jq` or Perfetto):

```c
static void emit_lifecycle_record(struct lifecycle *lc) {
    if (!json_out) return;

    fprintf(json_out,
        "{"
            "\"page_id\":%lu,"
            "\"pfn\":%lu,"
            "\"order\":%u,"
            "\"type\":%u,"
            "\"comm\":\"%.*s\","
            "\"tgid\":%u,"
            "\"ino\":%lu,"
            "\"dev\":%lu,"
            "\"file_offset\":%lu,"
            "\"alloc_ts\":%lu,"
            "\"cache_insert_ts\":%lu,"
            "\"first_fault_ts\":%lu,"
            "\"first_dirty_ts\":%lu,"
            "\"last_wb_start_ts\":%lu,"
            "\"last_wb_done_ts\":%lu,"
            "\"free_ts\":%lu,"
            "\"lifetime_ns\":%lu,"
            "\"dirty_count\":%u,"
            "\"migrate_count\":%u,"
            "\"io_count\":%u,"
            "\"cow_parent_id\":%lu"
        "}\n",
        lc->page_id, lc->pfn, lc->order, lc->page_type,
        (int)sizeof(lc->comm), lc->comm,
        lc->tgid, lc->ino, lc->dev, lc->file_offset,
        lc->alloc_ts, lc->cache_insert_ts, lc->first_fault_ts,
        lc->first_dirty_ts, lc->last_wb_start_ts, lc->last_wb_done_ts,
        lc->free_ts,
        lc->free_ts - lc->alloc_ts,
        lc->dirty_count, lc->migrate_count, lc->io_count,
        lc->cow_parent_id);

    fflush(json_out);
}
```

#### Component 4: main loop, stats, and signal handling

The main loop is structured around three concerns: drain the ring buffer, periodically emit stats, and shut down cleanly on SIGINT/SIGTERM.

```c
static void print_stats(struct page_tracer_bpf *skel) {
    uint32_t zero = 0;
    uint64_t counter_val;
    int cfd = bpf_map__fd(skel->maps.page_id_counter);
    bpf_map_lookup_elem(cfd, &zero, &counter_val);

    fprintf(stderr,
        "[stats] events_seen=%lu dropped=%lu page_ids_issued=%lu "
        "pfn_map_entries=%d\n",
        events_seen, events_dropped_at_handler, counter_val,
        /* count entries in pfn_to_id by iterating */ 0);
}

int main(int argc, char **argv) {
    struct cli_opts opts = {0};
    // parse args with argp — filter type, tgid, ino, json path, verbose
    // ...

    libbpf_set_print(libbpf_log);
    bump_memlock();

    // Step 1: open + load (runs verifier)
    struct page_tracer_bpf *skel = page_tracer_bpf__open();
    if (!skel) { fprintf(stderr, "open failed\n"); return 1; }

    if (page_tracer_bpf__load(skel)) {
        fprintf(stderr, "verifier rejected the program. "
                        "run with PAGE_TRACER_DEBUG=1 for details.\n");
        goto cleanup;
    }

    // Step 2: configure maps BEFORE attach
    if (configure_maps(skel, &opts) < 0) goto cleanup;

    // Step 3: attach programs to hooks
    if (page_tracer_bpf__attach(skel)) {
        fprintf(stderr, "attach failed: %s\n", strerror(errno));
        goto cleanup;
    }
    fprintf(stderr, "tracer running. Ctrl-C to stop.\n");

    // Step 4: open output
    if (opts.json_path) {
        json_out = fopen(opts.json_path, "w");
        if (!json_out) { perror("open json"); goto cleanup; }
    }

    // Step 5: set up ring buffer
    lm = lifecycle_map_create();
    struct ring_buffer *rb = ring_buffer__new(
        bpf_map__fd(skel->maps.events),
        handle_event, NULL, NULL);
    if (!rb) { fprintf(stderr, "ring_buffer__new failed\n"); goto cleanup; }

    // Step 6: signal handling
    signal(SIGINT, sig_handler);
    signal(SIGTERM, sig_handler);

    // Step 7: event loop with periodic stats
    time_t last_stats = time(NULL);
    while (!exiting) {
        int n = ring_buffer__poll(rb, 100 /* ms */);
        if (n < 0 && errno != EINTR) {
            fprintf(stderr, "poll failed: %s\n", strerror(errno));
            break;
        }

        time_t now = time(NULL);
        if (now - last_stats >= 5) {
            print_stats(skel);
            last_stats = now;
        }
    }

    // Step 8: drain remaining events before exit
    ring_buffer__consume(rb);
    fprintf(stderr, "final: "); print_stats(skel);

    // Step 9: emit in-flight lifecycles (pages that didn't see FREE
    // before we shut down). These are "incomplete" lifecycles.
    lifecycle_map_for_each(lm, emit_lifecycle_record);

cleanup:
    if (json_out) fclose(json_out);
    ring_buffer__free(rb);
    lifecycle_map_destroy(lm);
    page_tracer_bpf__destroy(skel);
    return exiting ? 0 : 1;
}
```

The shutdown order matters:

1. **Consume remaining events** via `ring_buffer__consume` (non-blocking drain) after the main loop exits. Without this, events that arrived in the last 100ms window are lost.
2. **Flush in-flight lifecycles.** Pages that were alive when we stopped tracing never produced an `EVT_FREE`. Emit them as "incomplete" records so downstream analysis knows they exist.
3. **`page_tracer_bpf__destroy`** detaches programs from their hooks and frees the BPF objects. Once this returns, no more events can fire — the tracepoints are back to NOP.

### What userspace can do with the event stream

Because events are a stream (not pre-aggregated), userspace has full flexibility:

- **Live tail.** Print events as they arrive — the minimal case.
- **Per-page reconstruction.** The code above. Emit one JSON record per lifecycle.
- **Rolling window.** Keep the last N seconds of events in a circular buffer. At any moment, the full history for any page_id is available.
- **Live aggregation.** Count events by type, track per-tgid allocation rates, maintain histograms of dirty→writeback latencies. Useful for monitoring.
- **Perfetto / trace_processor export.** Serialize each event as a protobuf `TrackEvent` message with slice IDs keyed by page_id. Perfetto's UI shows each page as a horizontal track with colored slices for each lifecycle stage. `trace_processor` can pivot the stream into SQL-queryable tables.
- **Offline replay.** Write raw events to a file and replay them into any of the above modes later, from the same recording.

### Why this design handles edge cases correctly

**Repeated dirty→writeback cycles.** A page dirtied and written back three times produces six events, all with the same page_id. The `dirty_count` and `io_count` counters capture this explicitly. The `last_wb_done_ts` reflects the most recent cycle; earlier cycles are visible in the raw event stream if you need them.

**Page migration.** The page_id survives migration. Events before and after migration all carry the same page_id. The `EVT_MIGRATE` event itself records the PFN change and `migrate_count` counts how many times it happened. Userspace does not need to do any PFN stitching.

**PFN reuse.** After a page is freed and its PFN is reused for a new allocation, the new page gets a new page_id (the counter is monotonic). Events from the old and new page will have different page_ids even though they share a PFN. There is no ambiguity.

**Ring buffer drops.** Under extreme load, `bpf_ringbuf_reserve` can return NULL (buffer full). The handler returns without emitting. This is visible in stats: if the kernel's `bpf_prog_inc_misses_counters` fires or if you observe `events_seen` plateauing, you are hitting drops. The fix is a larger ring buffer or a tighter filter. For most tracing workloads, 16–64MB avoids drops entirely.

**Clock skew.** `bpf_ktime_get_ns` returns `CLOCK_MONOTONIC` nanoseconds. This is consistent across CPUs but does not match `CLOCK_REALTIME` wall clock. If you want wall-clock timestamps in the output, capture a baseline pair `(ktime_ns, clock_gettime(CLOCK_REALTIME))` at startup and compute the offset in userspace.

**Events out of order across CPUs.** The ring buffer preserves per-CPU order but two events produced on different CPUs can appear in the reader in either order. For tight sequencing (e.g., "did CACHE_INSERT happen before FAULT on this page?"), trust the `ts` field, not arrival order. If two events on different CPUs have timestamps within ~50ns of each other, they are effectively simultaneous from the tracer's perspective.

---

## Part 19: Operating the Tracer — Build, Run, Debug, Tune {#part-19-operating-the-tracer-build-run-debug-tune}

The code is written. Now you need to build it, run it, verify it works, and tune it when it doesn't. This part is the operator's guide.

### Prerequisites check

Before you compile anything, verify the running kernel supports what you need:

```bash
# 1. BTF must be present (for CO-RE and vmlinux.h)
ls -l /sys/kernel/btf/vmlinux
# Should show a file. On Android, also check /sys/kernel/btf/modules/.

# 2. BPF syscall, tracing, and kprobes must be enabled
zcat /proc/config.gz 2>/dev/null | grep -E 'CONFIG_BPF|CONFIG_KPROBES|CONFIG_DEBUG_INFO_BTF' \
  || grep -E 'CONFIG_BPF|CONFIG_KPROBES|CONFIG_DEBUG_INFO_BTF' /boot/config-$(uname -r)
# CONFIG_BPF=y, CONFIG_BPF_SYSCALL=y, CONFIG_BPF_EVENTS=y,
# CONFIG_KPROBES=y, CONFIG_DEBUG_INFO_BTF=y

# 3. Tracefs must be mounted
mount | grep -E 'tracefs|debugfs'
ls /sys/kernel/tracing/events/kmem/mm_page_alloc/
# Should list: enable, filter, format, id, trigger

# 4. kallsyms must be readable (for vmemmap_base lookup)
head -5 /proc/kallsyms
# If all addresses show 0000000000000000, either run as root
# or echo 0 > /proc/sys/kernel/kptr_restrict
```

On Android, all of these are typically enabled in GKI (Generic Kernel Image) builds from Android 12 forward. On older Android, you may need to build your own kernel with these options.

### Building

The build has three stages: generate `vmlinux.h`, compile the BPF object, compile the userspace program. A minimal Makefile:

```makefile
CLANG ?= clang
CC ?= cc
BPFTOOL ?= bpftool
ARCH := $(shell uname -m | sed 's/x86_64/x86/; s/aarch64/arm64/')

CFLAGS := -O2 -g -Wall -I.
BPF_CFLAGS := -O2 -g -Wall -target bpf -D__TARGET_ARCH_$(ARCH) -I.

all: page_tracer

vmlinux.h:
	$(BPFTOOL) btf dump file /sys/kernel/btf/vmlinux format c > $@

page_tracer.bpf.o: page_tracer.bpf.c vmlinux.h
	$(CLANG) $(BPF_CFLAGS) -c $< -o $@
	# Strip DWARF — BPF loader doesn't need it and it bloats the binary
	llvm-strip -g $@

page_tracer.bpf.skel.h: page_tracer.bpf.o
	$(BPFTOOL) gen skeleton $< > $@

page_tracer: page_tracer_user.c page_tracer.bpf.skel.h
	$(CC) $(CFLAGS) -o $@ $< -lbpf -lelf -lz

clean:
	rm -f page_tracer page_tracer.bpf.o page_tracer.bpf.skel.h vmlinux.h

.PHONY: all clean
```

Key gotchas:

- **Do not compile the BPF object with `-O0`.** The verifier rejects unoptimized code because it generates patterns (uninitialized stack reads, complex control flow from dead-branch elimination not happening) that the verifier flags as unsafe. `-O2` is the minimum.
- **Always include `-g` on the BPF compile.** BTF generation requires DWARF debug info at compile time. `llvm-strip -g` removes it from the final object after BTF is extracted.
- **`__TARGET_ARCH_*` matters.** This is how BPF headers choose which `PT_REGS_PARM*` macros to define. Get it wrong and kprobe argument reads return nonsense.

### Running

The tracer needs capabilities. As root:

```bash
./page_tracer --filter=tgid --tgid=12345 --json=/tmp/trace.ndjson --verbose
```

As non-root, with fine-grained capabilities:

```bash
sudo setcap "cap_bpf,cap_perfmon,cap_syslog+ep" ./page_tracer
./page_tracer --filter=tgid --tgid=12345 --json=/tmp/trace.ndjson
```

- `cap_bpf`: load BPF programs and access maps.
- `cap_perfmon`: attach to tracepoints and kprobes.
- `cap_syslog`: read unredacted `/proc/kallsyms` for `vmemmap_base`.

Without `cap_syslog`, `vmemmap_base` reads as 0, `folio_to_pfn_helper` returns 0 for everything, and the dirty/writeback/migrate handlers silently skip every event. The verbose log warns about this at startup.

### Verifying it works

First, check attachment:

```bash
bpftool prog list | grep -A1 -E 'tracepoint|kprobe'
# Should show your programs with their attach types

cat /sys/kernel/tracing/events/kmem/mm_page_alloc/enable
# 1 means at least one consumer is attached
```

Then, generate some activity and check output:

```bash
# In another terminal, make the filtered tgid do something I/O heavy.
# For example, force cache misses by dropping caches and reading a file:
sudo sh -c 'echo 3 > /proc/sys/vm/drop_caches'
cat /path/to/target/file > /dev/null

# Meanwhile the tracer should be writing lifecycles
tail -f /tmp/trace.ndjson | jq .
```

A working tracer on a mildly busy process emits hundreds to thousands of lifecycles per second. If you see zero, something is wrong.

### Debugging "zero events"

Run the tracer with `PAGE_TRACER_DEBUG=1 ./page_tracer ...` to enable libbpf's debug logs. Then check:

**1. Are the BPF programs attached?**

```bash
bpftool prog show
# Look for your programs: on_page_alloc, on_cache_add, etc.
# Each should have a prog_id and the attach type should match your SEC().
```

If a program is loaded but not attached, the tracepoint fires but nothing runs. This typically means `page_tracer_bpf__attach` returned an error you ignored.

**2. Is the filter too strict?**

```bash
bpftool map dump name filter
# Shows the filter_type, tgid, ino you configured.
# If filter_type=0, no events are tracked by design.
```

**3. Is the page_id counter incrementing?**

```bash
bpftool map dump name page_id_counter
# Should show a non-zero, growing number.
# If stuck at 0: on_page_alloc is not firing (attach problem)
# or the gfp_flags filter is rejecting everything.
```

**4. Are there entries in `pfn_to_id`?**

```bash
bpftool map dump name pfn_to_id | head
# Should show pfn → page_id mappings.
# If empty: on_page_alloc never inserts. Check filter_type and gfp mask.
```

**5. Is the ring buffer accumulating unread data?**

In the tracer's stats output, watch for growing `events_seen` with a plateauing output rate. If the kernel produces faster than userspace consumes, events eventually drop. `bpftool prog show` reports misses per program — rising misses means you are losing events.

### Debugging "wrong events"

Sometimes the tracer produces events but they look inconsistent — missing CACHE_INSERT for file-backed pages, fault events with tgid 0, or page_ids that repeat.

**Missing CACHE_INSERT for file-backed allocations.** Your filter probably rejects at `on_page_alloc`. The allocation happens in kernel context (a readahead worker, not the reading process), so `bpf_get_current_pid_tgid()` returns the kworker's tgid, not the app's. Two fixes: relax `on_page_alloc` to accept all `__GFP_MOVABLE` allocations and filter by ino at `on_cache_add` instead (cleaning up non-matching entries then), or attach an `on_page_alloc` kprobe that reads the owning mm's tgid instead of `current`.

**FAULT events with tgid 0.** You are catching faults that happen during boot or in kernel threads. Add a `tgid != 0` check at the BPF handler level.

**page_ids that repeat.** Your `page_id_counter` was re-initialized between runs (for example, if you call `configure_maps` on a hot-reload without clearing state). Harmless unless you correlate across runs; to prevent it, persist the counter to a file and restore on startup.

**Migration events with `new_pfn == old_pfn`.** Your `folio_to_pfn_helper` returned 0 (see the capability note above). Both sides compute to 0 and the migration is a no-op. Check that `cap_syslog` is set and that `vmemmap_base_map` is populated.

### Performance tuning

The overhead has three components:

**Per-event overhead.** Each tracepoint hit runs the BPF program: verifier-approved code, a map lookup, a ring buffer reserve/submit, and a return. On ARM64 this is ~200–500ns per event. At 100,000 events per second (a moderately busy process under load), that is 20–50ms of CPU time per second — 2–5% of a single core.

**Ring buffer contention.** Multiple CPUs producing into the same ring buffer contend on the spinlock inside `__bpf_ringbuf_reserve` at `kernel/bpf/ringbuf.c:478`. On a 16-core phone this is rarely a bottleneck; on a 128-core server it can be. The workaround is per-CPU ring buffers, at the cost of slightly more complex consumer logic (you poll one per CPU).

**Map update contention.** The `pfn_to_id` hash map is updated on every allocation, migration, and free. High update rates cause per-bucket spinlock contention (see `kernel/bpf/hashtab.c` for the locking model). Using a `BPF_MAP_TYPE_LRU_HASH` instead of `BPF_MAP_TYPE_HASH` helps when load is high — LRU maps never reject updates, they evict the least-recently-used entry instead.

Tuning knobs:

| Knob | Purpose | Typical range |
|---|---|---|
| Ring buffer size | Headroom before drops | 16–64MB |
| `pfn_to_id` max_entries | Memory for active page tracking | 64K–1M |
| `sector_to_id` max_entries | Memory for inflight I/O tracking | 8K–64K |
| Filter specificity | Reduce event volume | Always use one in production |
| Poll timeout | Userspace CPU vs latency | 10–100ms |

### Tracer lifecycle: start, stop, and map cleanup

When the tracer stops (trace session ends, SIGINT received), pages that were allocated and tracked but not yet freed have entries in `pfn_to_id`. On the next trace session, these stale entries cause problems: a PFN reused for a different page gets the old page_id, corrupting the event stream.

The fix: clean shutdown.

```c
static void cleanup(struct page_tracer_bpf *skel) {
    // 1. Detach all programs — stop capturing events
    page_tracer_bpf__detach(skel);

    // 2. Drain the ring buffer — process remaining events
    ring_buffer__poll(rb, 0);

    // 3. Clear all maps — stale entries corrupt next session
    clear_map(bpf_map__fd(skel->maps.pfn_to_id));
    clear_map(bpf_map__fd(skel->maps.sector_to_id));

    // 4. Reset page_id counter
    u32 zero = 0; u64 zero_val = 0;
    bpf_map_update_elem(bpf_map__fd(skel->maps.page_id_counter),
                        &zero, &zero_val, BPF_ANY);

    // 5. Destroy skeleton (frees programs and maps)
    page_tracer_bpf__destroy(skel);
}
```

Each trace session starts with a clean `open_and_load` → `configure_maps` → `attach` sequence. No cross-session state corruption.

When using the Perfetto C SDK, the cleanup is triggered by the data source `on_stop` callback, and `on_start` loads fresh state. Each Perfetto trace session gets its own BPF skeleton lifecycle.

### Known limitations

**Page tables themselves are not tracked.** The tracer only sees pages allocated with `__GFP_MOVABLE`. Page table pages are kernel-internal and use `GFP_KERNEL`, so they are filtered out. If you want to trace page table allocation, add a second `on_page_alloc` attachment with a different filter.

**Slab allocations are not tracked.** Same reason — slab uses `GFP_KERNEL`. If you care about slab, attach to `kmem_cache_alloc` tracepoints instead.

**The first ~100ms of an attach window can miss events.** Between `bpf_prog_load` and `bpf_prog_attach`, any events that would have fired are lost. For applications that are already running, this is usually invisible; for bringup tracing, start the tracer before the target process.

**Ring buffer can drop under extreme load.** A single kernel compile can produce a million `mm_page_alloc` events per second. Even with a 64MB buffer, if userspace falls behind by ~500ms, you get drops. The fix is a tighter filter, a larger buffer, or a consumer that keeps up (e.g., writes to a memory-mapped file rather than a socket).

**zRAM compression is hard to trace precisely.** `zram_write_page` fires on every write, but the events do not naturally correspond to a specific PFN from the tracer's perspective (the compressed page is stored by zsmalloc handle, not PFN). You get aggregate compression stats but not per-page correspondence unless you also hook the swap layer.

---

## Part 20: Perfetto Integration {#part-20-perfetto-integration}

### Why Perfetto instead of custom JSON

The tracer in Part 18 writes NDJSON from custom C. That works for prototyping but does not scale to production analysis. Perfetto replaces the custom output pipeline:

```
Ring buffer → Perfetto C SDK data source → TracePacket →
  trace_processor → SQL → Perfetto UI
```

Perfetto gives you timeline visualization in its UI, SQL queries for ad-hoc analysis via trace_processor, a standard `.perfetto-trace` format that tools across the Android ecosystem already consume, and the ability to correlate page lifecycle events with CPU scheduling, binder transactions, and GPU counters in a single trace file.

### The architecture

```
Kernel:
  Tracepoints + kprobes → eBPF handlers → BPF ring buffer

Userspace (Perfetto C SDK data source):
  On trace start: load eBPF, configure filters, attach
  Poll loop: read ring buffer, emit TracePackets
  On trace stop: detach, destroy eBPF

Perfetto:
  traced collects TracePackets
  Trace processor creates SQL tables
  UI visualizes timeline
```

The tracer registers as a Perfetto data source. When `traced` starts a trace with the `linux.page_lifecycle` data source enabled, the tracer loads and attaches the eBPF programs. When the trace stops, it detaches. This is the standard Perfetto data source lifecycle — the same pattern used by `traced_probes` for ftrace events.

### Why the C SDK, not the C++ SDK

Perfetto offers two APIs. The C++ `DataSource` API is designed for in-process tracing (instrumenting your own application). The C SDK (`perfetto/tracing/ctrace.h`) is for out-of-process system daemons that produce events from external sources — which is what the tracer is. The C SDK registers a data source with `traced`, receives start/stop callbacks, and writes TracePackets using the C protobuf serialization API.

### TrackEvent vs custom proto

Two approaches for encoding page lifecycle events in TracePackets:

**TrackEvent (no Perfetto changes needed).** Use the generic TrackEvent format with custom debug annotations. The existing trace processor already knows how to parse TrackEvents and expose them as the `slice` and `instant` tables. Page events appear as events on a custom track. The downside: no type-safe SQL columns — event-specific data is in debug annotation key-value pairs that require casting and string matching.

**Custom proto (requires Perfetto changes).** Define a `PageLifecycleEvent` proto message as a TracePacket extension. Write a trace processor plugin that parses these messages and creates a dedicated `page_event` SQL table with typed columns (`page_id INT`, `pfn INT`, `type TEXT`, etc.). The downside: you must modify the Perfetto source and rebuild the trace processor.

For prototyping, TrackEvent works immediately. For production use on Android, the custom proto path gives better SQL ergonomics and is worth the Perfetto-side work.

### SQL-based correlation

Every event carries `page_id`. Every join is on `page_id`:

```sql
-- Full lifecycle of one page
SELECT * FROM page_event WHERE page_id = 42 ORDER BY ts;

-- Hardware I/O latency per page
SELECT io_c.ts - io_i.ts AS hw_latency_ns
FROM page_event io_i
JOIN page_event io_c
    ON io_i.page_id = io_c.page_id
    AND io_c.type = 'IO_COMPLETE'
WHERE io_i.type = 'IO_ISSUE';

-- Cross-system: page faults during CPU scheduling slices
SELECT pe.page_id, pe.ts, s.dur
FROM page_event pe
JOIN sched_slice s
    ON pe.tgid = s.utid
    AND pe.ts BETWEEN s.ts AND s.ts + s.dur
WHERE pe.type = 'FAULT';
```

These joins work across data sources. A page fault event from the eBPF tracer can be correlated with a CPU scheduling event from ftrace and a binder transaction from atrace — all in one query, one trace file, one timeline.

### Triggering on Android

Perfetto is the standard tracing system on Android. `traced` and `traced_probes` run as system services. The custom data source registers with `traced` and appears in Perfetto trace configs alongside existing data sources:

```
adb shell perfetto \
  -c - --txt <<EOF
buffers: { size_kb: 65536 }
data_sources: {
  config {
    name: "linux.page_lifecycle"
    page_lifecycle_config {
      filter_comm: "com.example.app"
    }
  }
}
data_sources: {
  config { name: "linux.ftrace" }
}
duration_ms: 10000
EOF
```

The resulting trace contains both page lifecycle events and standard ftrace events. One timeline, one SQL query surface across all data sources.

---

## Part 21: Android-Specific Considerations {#part-21-android-specific-considerations}

### Kernel requirements

Android GKI (Generic Kernel Image) from Android 12+ has everything enabled:

```
CONFIG_BPF=y
CONFIG_BPF_SYSCALL=y
CONFIG_BPF_EVENTS=y
CONFIG_DEBUG_INFO_BTF=y
CONFIG_FTRACE=y
CONFIG_KPROBES=y
CONFIG_TRACEPOINTS=y
```

No kernel patches are needed. Stock GKI works with all the tracepoints and kprobes the tracer requires.

### zRAM instead of disk swap

Android uses zRAM exclusively for swap. Part 13 covers zRAM internals; here we note the tracer implications. The tracer hooks `zram_write_page` via kprobe to capture compression events. After compression, the page's PFN is freed — the data exists as a zsmalloc handle, not at a PFN. The `page_id` in the `pfn_to_id` map persists until the free event cleans it up.

On swap-in, a new PFN is allocated. The tracer sees `EVT_ALLOC` for the new PFN. Connecting the swap-out and swap-in requires tracking the swap entry through `folio->swap` — a limitation acknowledged in Part 16's swap correlation section.

### lmkd (Low Memory Killer Daemon)

When memory is tight, Android follows a priority order. First, reclaim clean file pages — these are free to evict because the data can be re-read from the APK or disk. Second, if that is not enough, kill background apps via lmkd. Killing drops the entire `mm_struct` at once — all VMAs, all page tables, all mapped pages freed in a single `exit_mmap` call. This is far cheaper than swapping each page individually. Third, as a last resort, swap foreground app pages to zRAM.

The tracer sees mass `EVT_FREE` events when lmkd kills an app — thousands of frees in a burst, all from the same tgid. This pattern is distinctive and easy to detect in the event stream.

### Zygote and fork

Android apps fork from the Zygote process. After fork, app-specific pages COW. The tracer sees:

- `EVT_ALLOC` for post-fork anonymous pages.
- `EVT_COW` when writing to shared Zygote pages (e.g., the ART runtime's boot image).
- The COW event carries `old_page_id`, linking to the Zygote's original page.

This lets you measure how much memory an app actually allocates after fork vs how much it shares with Zygote.

### File paths on Android

APK files contain DEX code, OAT compiled code, resources:

```
page_tracer --file "/data/app/com.example.app/base.apk"
page_tracer --file "/data/dalvik-cache/arm64/*.odex"
```

Shared libraries:

```
page_tracer --file "/apex/com.android.runtime/lib64/bionic/libc.so"
page_tracer --file "/system/lib64/*.so"
```

### UFS storage

Android uses UFS, not NVMe. Single command queue (UFS 3.x) or limited multi-queue (UFS 4.0), compared to NVMe's 65K queues. ~100–500µs typical hardware latency for 4KB random reads. FTL garbage collection causes 5–50ms spikes — the single biggest source of tail latency on Android. The tracer captures `EVT_IO_ISSUE → EVT_IO_COMPLETE` gaps that reveal these spikes directly.

Thermal throttling compounds this. When the SoC heats up, the UFS controller clock is reduced, increasing latency. A page that normally takes 200µs to read can take 2ms under thermal throttling. The tracer reveals these latency shifts because every I/O event has a precise timestamp — a histogram of `IO_COMPLETE - IO_ISSUE` across a trace shows the thermal throttling pattern as a bimodal distribution.

### MGLRU on Android

Android 14+ uses Multi-Generation LRU (MGLRU) as the default reclaim policy. Instead of two buckets (active/inactive), MGLRU uses 4 generations per memcg. Pages age through generations based on access patterns detected by periodic page table scans. This gives finer-grained reclaim ordering than 2Q and works better on memory-pressured phones where the difference between "accessed 1 second ago" and "accessed 30 seconds ago" matters for keeping the foreground app responsive.

The tracer does not need to handle MGLRU specially — the same `mm_page_alloc`, `mm_page_free`, and reclaim events fire regardless of whether 2Q or MGLRU drives the reclaim decisions. But knowing MGLRU is active helps interpret reclaim patterns: pages may be reclaimed in a different order than you would expect from classic active/inactive reasoning.

### Per-app memory limits (memcg)

Android uses cgroups v2 memory controllers to enforce per-app memory limits. Each app runs in its own memory cgroup with a `memory.max` limit. When an app exceeds its limit, the kernel's memcg reclaim kicks in — reclaiming pages from that specific cgroup rather than system-wide. If reclaim cannot free enough, the OOM killer targets a process in the overcommitted cgroup.

The tracer's `tgid` field on each event lets you correlate page lifecycle events with specific cgroups (since tgid → PID → cgroup mapping is available from `/proc/<pid>/cgroup`). This is how you answer "which app is causing the most reclaim activity" or "which app's pages are being compressed to zRAM most often."

---

## Part 22: Historical Context — How We Got Here {#part-22-historical-context-how-we-got-here}

### The evolution of struct page

In early Linux (2.0-2.4 era), `struct page` was a straightforward descriptor with explicit fields for everything: `mapping`, `index`, `count`, `flags`, `lru`, `buffers`. It was clean and readable, but large — every additional byte cost megabytes of boot-time RAM on machines with a lot of memory.

Over time, the kernel developers aggressively compressed `struct page` using unions. The same memory serves as LRU list links, buddy allocator metadata, slab allocator metadata, or network stack metadata — depending on the page's current role. This made `struct page` smaller but nearly incomprehensible to newcomers. The comment at `mm_types.h:83` warns: *"Five words (20/40 bytes) are available in this union. WARNING: bit 0 of the first word is used for PageTail()."*

The `struct folio` introduction (v5.16-v5.18, primarily by Matthew Wilcox) aimed to reduce confusion by creating a type that clearly represents "one or more contiguous pages treated as a unit." Functions take `struct folio *` when they operate on the whole unit and `struct page *` when they care about a specific sub-page. The transition is still underway — much of mm/ now uses folios, but some older code still deals in raw pages.

### From rb-tree to maple tree

VMAs were stored in a red-black tree (and simultaneously a linked list) from the early 2.6 days until v6.1, when Liam Howlett converted them to a **maple tree**. The maple tree is a B-tree variant that stores multiple entries per node (up to 16), which means fewer cache misses during VMA lookups. It also eliminated the linked list entirely — the tree alone handles both point lookups and range iteration.

The linked list was a particular problem because `find_vma` was O(log n) via the tree, but many code paths iterated the list, which was O(n). With the maple tree, both operations are O(log n).

### The move to multi-queue block I/O

Until v3.16, Linux had a single request queue per block device with a single I/O scheduler. This was fine for spinning disks (which have one head and can only serve one request at a time) but terrible for modern NVMe and UFS devices that support thousands of concurrent commands.

**blk-mq** (multi-queue block layer, v3.16) replaced the single queue with a per-CPU software queue structure that maps to hardware submission queues. This eliminated the single-queue spinlock that was the bottleneck at high IOPS and allowed the kernel to actually use modern storage hardware's parallelism.

### eBPF's evolution

The original BPF (Berkeley Packet Filter, 1992) was a simple register machine for filtering network packets. It had two 32-bit registers, a scratch memory area, and about 20 instructions.

eBPF (v3.18, 2014) expanded BPF into a general-purpose in-kernel VM: 10 64-bit registers, a 512-byte stack, maps for persistent storage, helper functions for accessing kernel state, and a verifier that proved programs safe before execution. With a good enough verifier, you can run user-provided code inside the kernel — not just for packet filtering, but for tracing, security, scheduling, and more.

The verifier has grown from ~1,000 lines in v3.18 to over 20,000 lines today. Each version adds support for new patterns: bounded loops (v5.3), BPF-to-BPF function calls (v4.16), spinlocks (v5.1), ring buffers (v5.8), and BTF-based type checking that enables CO-RE portability.

### The LRU: from 2Q to MGLRU

Early Linux used a single LRU list — simple but coarse. The kernel couldn't distinguish hot pages that got evicted by a scan from genuinely cold pages.

**Two-list (2Q) LRU** (kernel 2.4 era) introduced active and inactive lists. A page is put on the inactive list when first accessed; a second access promotes it to active. The reclaim scanner targets the inactive list, keeping hot pages safe on the active list. This is still the default on most desktop/server kernels and lives in `mm/vmscan.c`.

**MGLRU (Multi-Generation LRU)**, merged in kernel 6.1, generalizes this to multiple generations per memcg. Instead of two buckets (active/inactive), there are N (typically 4), and pages age through them based on access patterns tracked via periodic page table scanning. This gives finer-grained reclaim ordering and works better on memory-pressured workloads. MGLRU is the default on Android 14+ and on many server distributions now.

### The page allocator: watermarks and zones

The buddy allocator divides physical memory into **zones** (ZONE_DMA, ZONE_NORMAL, ZONE_MOVABLE, ZONE_DEVICE) based on physical constraints. Each zone has **watermarks**:

- `min`: emergency reserve. Only `PF_MEMALLOC` tasks (reclaim itself) can dip below.
- `low`: wake kswapd to begin background reclaim.
- `high`: kswapd stops when all zones are above this.

`watermark_scale_factor` (a sysctl) adjusts the spread between `low` and `high`. Higher values mean kswapd works harder and earlier, at the cost of more CPU time. Android tunes this aggressively — a low-RAM phone wants kswapd to start reclaiming before memory pressure becomes severe.

The transition from single-zone (2.4) to multi-zone with per-zone LRUs and per-zone reclaim watermarks happened gradually across the 2.6 series to support NUMA, larger memory, and memory hotplug.

### SLAB / SLUB / SLOB / SLUB (singular)

The kernel's small-object allocator has evolved dramatically:

- **SLAB** (original, 1996): object caches with hard-coded slab sizes. Complex but well-optimized.
- **SLUB** (2008): simpler, uses per-CPU freelists and per-node partial slab lists. Default since kernel 2.6.23.
- **SLOB** (2005): simple first-fit allocator for memory-constrained embedded systems.

As of kernel 6.4, SLAB was removed entirely. SLUB is now the universal small-object allocator. Every `kmalloc`, every `kmem_cache_alloc` goes through SLUB. It sits on top of the buddy allocator — SLUB requests whole pages from buddy, then carves them into small objects.

This matters for the tracer because slab allocations *also* go through `mm_page_alloc` — but with `GFP_KERNEL` or other non-user flags, which our tracer's filter rejects. Understanding the allocator hierarchy is what lets you write filters that select the right traffic.

### Writeback: from bdflush to per-BDI workers

Writeback in early Linux used a single kernel thread (`bdflush`, then `pdflush`). Every dirty page in the system was flushed by one thread, creating scalability bottlenecks on multi-disk systems.

Kernel 2.6.32 introduced per-BDI (per-backing-device) writeback threads. Each block device gets its own `kworker/u:flush-*` thread. A slow USB drive no longer stalls writeback to a fast SSD.

Kernel 4.2 added **per-cgroup writeback**: dirty-page accounting and throttling happen per memcg, so one container's heavy write workload doesn't throttle another container's normal-paced dirtying.

### The reverse-mapping saga

File-backed reverse mapping (i_mmap) existed from the earliest days — it's needed for `munmap` to clear PTEs. But anonymous reverse mapping (anon_vma) was added much later, around kernel 2.6.

Before anon_vma, anonymous pages couldn't be reclaimed — there was no way to find all PTEs pointing to an anonymous page, so the kernel couldn't safely unmap it. This meant all anonymous memory was essentially mlocked. Adding anon_vma enabled swap to work for anonymous pages.

The anon_vma data structures have since gone through several redesigns to handle fork scalability (the "anon_vma scalability" patches, ~2010) — early implementations had O(N) behavior on deep fork chains.

### Why the mm subsystem looks the way it does

The mm subsystem's complexity comes from a fundamental tension: **memory is a shared resource with many concurrent users, and every decision involves tradeoffs between throughput, latency, fairness, and power consumption.**

Lazy allocation (demand paging) trades page fault latency for memory efficiency — you never allocate pages you do not use, but every first access costs a fault. COW trades fork latency for copy latency — fork is fast but the first write is slow. The page cache trades RAM capacity for I/O performance — you use RAM to avoid disk access, but now you need reclaim to manage that RAM. zRAM trades CPU cycles for memory capacity — compression costs time but saves space.

Every one of these tradeoffs is tunable, and Android tunes them differently from a server or a desktop. The kernel parameters (`dirty_ratio`, `swappiness`, `compact_proactiveness`, `watermark_scale_factor`) and the scheduling of background daemons (`kswapd`, `kcompactd`, `kworker/flush`) determine how these tradeoffs play out on a specific device.

Understanding the mm subsystem is fundamentally about understanding these tradeoffs — not just what the code does, but why it does it that way, and what would break if you changed it.

### Further reading

If you want to dive deeper into any part of what we've covered, these are the canonical references:

**Books:**
- *Linux Kernel Development* by Robert Love — readable introduction, slightly dated but the fundamentals don't change.
- *Understanding the Linux Virtual Memory Manager* by Mel Gorman — the definitive reference on 2.6-era mm internals. Still accurate on the core concepts (buddy, zones, LRU).
- *Linux Kernel Networking* by Rami Rosen — for the BPF origin story and the ring buffer design.

**Kernel documentation:**
- `Documentation/admin-guide/mm/` — operator-facing docs on HugeTLB, THP, zswap, numa_balancing.
- `Documentation/core-api/xarray.rst` — the data structure at the heart of the page cache.
- `Documentation/filesystems/writeback.rst` — the dirty page and writeback accounting rules.
- `Documentation/bpf/` — the BPF subsystem's user and developer guides.

**LWN articles (search lwn.net):**
- "The folio" series (Matthew Wilcox, 2021–2023) — the folio transition explained by the person doing it.
- "The maple tree" (Jonathan Corbet, 2021) — the VMA data structure change.
- "Multi-generational LRU" (2022) — MGLRU design and merger into mainline.
- "Transparent huge pages: the race to fix them" — the long saga of THP.

**Direct source paths worth reading end-to-end:**
- `mm/memory.c` — the page fault handler, COW, swap-in.
- `mm/filemap.c` — the page cache and file-backed I/O.
- `mm/vmscan.c` — reclaim.
- `mm/page-writeback.c` — dirty page accounting and writeback.
- `kernel/bpf/verifier.c` (just the intro comment) — the verifier's philosophy.
- `kernel/bpf/core.c` (`___bpf_prog_run`) — the BPF interpreter.

The kernel source is the primary source of truth. Every book and article drifts; the code doesn't.

---

## Part 23: DMA-buf — How Pages Move Between Hardware {#part-23-dma-buf-how-pages-move-between-hardware}

When a camera sensor captures a frame, the ISP processes it, the GPU applies filters, the video encoder compresses it, and the display controller shows the preview. Each is a separate hardware block with its own DMA engine. The frame data — 18MB for a 12MP YUV frame — must be accessible to all of them without copying. At 30fps, copying between stages would consume 1.6GB/s of pure memcpy.

DMA-buf wraps a set of physical pages in a kernel object (`struct dma_buf` at `include/linux/dma-buf.h:294`) backed by a file descriptor. The fd provides lifetime management via reference counting. The export/import protocol handles address space translation across IOMMUs. The sync/fence APIs manage cache coherency and ordering between hardware stages.

### The IOMMU

The MMU translates CPU virtual addresses to physical addresses (Part 3). The **IOMMU** does the same for DMA devices. A device behind an IOMMU generates **I/O virtual addresses (IOVAs)**, and the IOMMU translates them to physical addresses. This means scattered physical pages can appear contiguous to the device:

```
Physical pages: 0x10000000, 0x20040000, 0x10100000 (scattered)

IOMMU mapping for GPU:
  IOVA 0x80000000 → physical 0x10000000
  IOVA 0x80001000 → physical 0x20040000
  IOVA 0x80002000 → physical 0x10100000

GPU sees: contiguous 12KB buffer at IOVA 0x80000000
```

Without an IOMMU, the device uses raw physical addresses. A buggy driver or malicious device can DMA to any physical address — overwriting kernel code, page tables, or other processes' data. The IOMMU restricts each device to only the physical pages it has been explicitly mapped to.

### Where DMA-buf pages come from

`dma_heap` (kernel 5.6+, replacing Android's ION) allocates pages. The **system heap** at `drivers/dma-buf/heaps/system_heap.c` uses a cascading order strategy:

```c
// drivers/dma-buf/heaps/system_heap.c:52
static const unsigned int orders[] = {8, 4, 0};
```

It tries order-8 (1MB) first for fewer scatter-gather entries and longer DMA bursts. Falls back to order-4 (64KB) on fragmented memory, then order-0 (4KB) as last resort. An 8MB buffer on a fresh system: 8 order-8 allocations. On a fragmented system: a mix producing a longer scatter-gather list.

### Cache coherency

On ARM64, DMA is not cache-coherent (Part 2.9). Every ownership transition between CPU and device requires explicit cache maintenance. `dma_sync_sg_for_device` cleans (flushes) dirty cache lines to RAM before a device reads. `dma_sync_sg_for_cpu` invalidates cache lines before the CPU reads device-written data. For an 8MB buffer, that is 131,072 cache line operations — ~200µs on a big core.

Userspace triggers sync via `DMA_BUF_IOCTL_SYNC` before and after CPU access. Forgetting this ioctl causes intermittent corruption — the CPU reads stale cache lines instead of fresh device data.

### Fences: hardware-to-hardware ordering

A `dma_fence` (at `include/linux/dma-fence.h:67`) represents a future hardware completion. When the camera finishes writing a frame, it signals a fence. The GPU's command stream contains a wait instruction for that fence — the GPU hardware stalls until the fence signals, then proceeds. The CPU is not involved in the actual wait.

Each dma_buf has a `struct dma_resv` holding associated fences. Writers add exclusive fences; readers add shared fences. A writer waits for all previous readers and writers. A reader waits only for the previous writer. Standard reader-writer synchronization, implemented in hardware.

Android uses explicit fences passed as `sync_file` file descriptors. SurfaceFlinger receives a fence fd from each app alongside the GraphicBuffer. The fence indicates when the app's GPU rendering will complete. SurfaceFlinger (or HWC) waits on this fence before scanning out the buffer to the display.

### The Android graphics pipeline

```
App renders → GPU draws into GraphicBuffer (wraps dma_buf fd)
  → GPU produces fence F_app
  → queueBuffer(buffer, F_app) via Binder to SurfaceFlinger
  → SurfaceFlinger composites (GPU waits on each app's fence)
  → produces fence F_sf → passes to HWC
  → display controller waits on F_sf in hardware
  → scans buffer on vsync → signals release fence
  → app reuses buffer for next frame
```

Zero CPU copies of pixel data. Every transfer is a dma_buf fd handoff. Every synchronization is a hardware fence wait.

### DMA-buf and the tracer

DMA-buf pages are outside the tracer's current scope. They are allocated via `alloc_pages` internally (so `mm_page_alloc` fires), but immediately pinned — not on any LRU, not in the page cache, not reclaimable. The tracer sees `EVT_ALLOC` with no subsequent lifecycle events. To trace DMA-buf lifecycle, hook the DMA-buf API directly: `dma_heap_buffer_alloc`, `dma_buf_attach`, `dma_buf_map_attachment`, `dma_fence_signal`, `dma_buf_release`.

---

## Part 24: Network Stack — How Pages Carry Packets {#part-24-network-stack-how-pages-carry-packets}

Filesystem I/O and network I/O both move data between userspace and hardware using pages. But the patterns differ: file pages live for seconds to hours in the page cache; network pages live for microseconds during packet transit. File pages are 4KB, matching disk blocks well; packets are 64–1500 bytes, wasting most of a page. The allocation rate for network can reach millions per second on busy servers.

### sk_buff: the packet descriptor

Every packet in the kernel is described by `struct sk_buff` (at `include/linux/skbuff.h:885`). It has two data regions:

**Linear data** (the `head` to `end` buffer): a contiguous slab-allocated buffer (256B–2KB) holding protocol headers — Ethernet (14B), IP (20B), TCP (20B). Protocol processing modifies headers extensively (checksum computation, routing lookup, netfilter inspection), so headers must be contiguous.

**Paged data** (the `frags[]` array): for payloads larger than the linear buffer. Each frag is a `(page, offset, length)` tuple. The page holds payload bytes that don't need header-level manipulation — just data being moved from one place to another.

A typical received TCP packet: 54 bytes of headers in the linear region, 1406 bytes of payload in `frags[0]` — same page, different offsets.

### page_pool: bypassing the buddy allocator

The NIC receive path needs a fresh page for every packet to refill the receive ring. At 1M packets/sec, calling `alloc_pages` for each — with its zone lock, watermark check, freelist walk — costs 100–300ms of CPU per second.

`page_pool` (at `include/net/page_pool/types.h:167`) eliminates this overhead. A per-CPU array of 128 pre-allocated pages serves as a fast cache. Allocation is one array index decrement (~5ns). In steady state, pages cycle: alloc from cache → NIC DMAs into page → stack processes → free back to cache. The buddy allocator is never touched.

The tracer sees the initial pool fill (128 `mm_page_alloc` events) and the final drain (128 `mm_page_free` events). The millions of per-packet reuses in between are invisible — the pages never leave the pool.

### The receive path

```
NIC hardware: packet arrives → DMA into pre-allocated page → interrupt
Driver (NAPI): build sk_buff pointing to page → napi_gro_receive
  → refill ring slot with page_pool_alloc
GRO: coalesce small TCP segments into one large sk_buff (10× fewer)
IP/TCP: route lookup, firewall, sequence check, ACK processing
Socket: sk_buff queued on sk->sk_receive_queue
recv(): skb_copy_datagram_msg → copies payload to userspace → frees sk_buff
  → page returned to page_pool
```

The copy in `skb_copy_datagram_msg` is the main overhead. Everything before it is pointer manipulation — the packet data stays in the original page the NIC DMA'd into.

### sendfile: zero-copy send

`sendfile(sock_fd, file_fd, &offset, count)` builds sk_buffs with frags pointing directly to page cache pages. The NIC DMAs from the page cache page — no intermediate buffer. The page cache page participates in three systems simultaneously: the page cache (xarray ref), the socket's send buffer (sk_buff frag ref), and the NIC's DMA (pinned by refcount). `_refcount` drops back to 1 (page cache only) after the NIC finishes.

### What the tracer sees

```
I/O type          Tracer coverage
─────────────────────────────────
File read/write   Full (page cache → block I/O)
Binder small txn  Full (alloc_pages + page_free fire)
Binder large txn  None (DMA-buf pages bypass mm events)
Network recv      Partial (page_pool: initial alloc visible, recycling invisible)
Network send      Partial (sendfile: page cache pages visible)
```

Extending to networking requires hooking the page_pool API (tracepoints available in kernel 5.17+) and the sk_buff lifecycle — a different data source from page lifecycle events.

---

## Part 25: Filesystem Deep Dive — f2fs, ext4, erofs {#part-25-filesystem-deep-dive-f2fs-ext4-erofs}

Part 10 covered the VFS abstraction. This part covers the three filesystems on Android in depth: how each translates file offsets to disk sectors, how each handles crash consistency, and how each interacts with the page cache and the tracer.

### f2fs: Flash-Friendly File System

f2fs (default `/data` on most Android devices since ~2015, merged kernel 3.8) is designed around NAND flash properties. Flash cannot overwrite in place — it must erase first (2–5ms per erase block). f2fs avoids in-place overwrites by appending to a log.

**On-disk layout.** f2fs divides the partition into a superblock (4KB, written once at format), a checkpoint area (CP, two packs for crash consistency), a Segment Information Table (SIT, tracks valid blocks per segment), a Node Address Table (NAT, translates node IDs to physical addresses), and the main area (data and node blocks, organized into 2MB segments).

**The SIT.** Each segment (512 blocks = 2MB) has a `struct f2fs_sit_entry` (at `include/linux/f2fs_fs.h:417`):

```c
struct f2fs_sit_entry {
    __le16 vblocks;                          // count of valid blocks (0-512)
    __u8   valid_map[SIT_VBLOCK_MAP_SIZE];   // bitmap: which blocks are valid
    __le64 mtime;                            // segment age
};
```

When f2fs overwrites a file block, the new version goes to a fresh location and the SIT marks the old location invalid. GC uses `vblocks` to find cheap victim segments — a segment with 5 valid blocks costs only 5 block copies to free 2MB.

**The NAT: breaking the wandering tree.** In a naive log-structured FS, updating data block D forces updating its parent node (new address), which forces updating the grandparent, cascading to the root. f2fs breaks this cascade with the NAT: parent nodes store **node IDs** (stable 32-bit identifiers) instead of physical addresses. The NAT translates node IDs to current physical addresses. When a node moves, only its NAT entry changes — the parent node is unaffected.

One data block update costs: 1 data write + 1 node write + 1 NAT entry update. Without NAT: 1 data + 1 node + 1 inode + 1 parent directory = 4 writes.

**Multi-head logging.** f2fs maintains 6 open log segments simultaneously — hot/warm/cold for both data and nodes. Temperature separation reduces GC cost: a hot segment (all blocks rewritten frequently) frees itself entirely when all blocks become stale. A cold segment (rarely touched) is never GC'd. Mixing hot and cold in one segment forces GC to copy cold blocks every time it wants to free the hot blocks' space.

**Checkpoint.** f2fs uses dual checkpoint packs instead of a journal. A checkpoint writes the complete filesystem state (SIT/NAT versions, segment summaries) to one of two CP packs, alternating. If power fails mid-write, the incomplete pack has mismatched header/footer versions and is discarded; the previous pack is still valid.

For `fsync`, f2fs has a shortcut: write the file's dirty data blocks + the inode node (marked with `FSYNC_FLAG`) + a device flush. On recovery, f2fs scans for flagged nodes after the last CP and replays them. Cost: 1 data write + 1 node write + 1 flush — lighter than ext4's full journal transaction.

**GC.** Victim selection uses cost-benefit analysis: `score = (1 - valid_ratio) × age / (2 × valid_ratio)`. Old, mostly-empty segments score highest. Background GC runs during idle periods. Foreground GC runs when free segments drop below a threshold and blocks the allocating thread.

The **double-GC cascade**: when f2fs GC writes valid blocks to a new segment, the UFS FTL maps these to NAND pages. When the old segment is TRIMmed, the FTL marks old NAND pages stale. The FTL's own GC then fires, reading valid NAND pages from the old erase block and rewriting them. Both GC processes compete for NAND bandwidth. The tracer sees this as `EVT_IO_ISSUE → EVT_IO_COMPLETE` gaps 25–250× longer than normal.

### ext4

ext4 uses in-place overwrites with a write-ahead journal (jbd2). Before modifying filesystem structures, ext4 writes the planned modifications to a journal area. If power fails during the apply phase, the journal is replayed on mount.

Three journaling modes: **journal** (data + metadata journaled, safest, slowest), **ordered** (default — metadata journaled, data forced to disk before metadata commit), **writeback** (metadata only, fastest, risk of stale data exposure after crash).

**Extents.** `struct ext4_extent` (at `fs/ext4/ext4_extents.h:56`) maps contiguous file blocks to contiguous disk blocks. A contiguous 1GB file needs one 12-byte extent. The extent tree root fits in the inode's `i_block` field — files with ≤4 extents need no extra disk blocks for metadata.

**Delayed allocation.** ext4 delays assigning physical disk blocks until writeback. 100 `write()` calls create 100 dirty pages with `DELAYED_ALLOCATION` markers. At writeback, ext4 allocates 100 contiguous blocks in one request — one extent instead of 100 scattered blocks. Reduces fragmentation and improves read performance.

### erofs: Enhanced Read-Only File System

Android's `/system`, `/vendor`, `/product` partitions never change at runtime. erofs (merged kernel 4.19, default on Android 12+) exploits this: no dirty pages, no writeback, no journal, no block allocation. The data is compressed at build time using LZ4 with large cluster sizes (64KB).

**Cluster-based compression.** erofs compresses groups of 16 contiguous pages (64KB) together instead of individual 4KB pages. This yields much better compression ratios (~0.31 vs ~0.70 per-page) because the larger window captures more redundancy. Reading one page decompresses the entire cluster — the other 15 pages go into the page cache as free prefetching.

A 5GB system image compresses to ~2.5GB on disk. Decompression at ~4GB/s LZ4 on a big core means effective read bandwidth from erofs exceeds raw UFS bandwidth.

**Reclaim.** erofs pages are never dirty. Reclaim discards them without writeback — free. If needed again, the kernel reads the compressed cluster from disk and decompresses. The disk read is smaller than the decompressed data, so effective re-read bandwidth is higher than from an uncompressed filesystem.

**Tracer behavior.** erofs pages are standard page cache pages. The tracer sees `EVT_ALLOC → EVT_CACHE_INSERT → EVT_READ → EVT_IO_ISSUE → EVT_IO_COMPLETE → EVT_READ_DONE → EVT_FAULT → EVT_FREE`. No `EVT_DIRTY`, no `EVT_WRITEBACK` — ever. The `nr_sectors` in I/O events reflects the compressed cluster size, not the decompressed size.

### Android partition summary

```
Partition    Filesystem    R/W          Compressed    Content
──────────────────────────────────────────────────────────────────
/system      erofs         read-only    LZ4           OS, frameworks
/vendor      erofs         read-only    LZ4           HAL libraries
/product     erofs         read-only    LZ4           pre-installed apps
/data        f2fs          read-write   no            app data, user files
/metadata    ext4          read-write   no            encryption keys
/cache       ext4          read-write   no            OTA staging
```

App launch touches all three: loading .so from `/system` (erofs, compressed read), loading DEX from `/data` (f2fs, cached), writing app storage on `/data` (f2fs, log-structured). The tracer's `s_dev` field in `EVT_CACHE_INSERT` identifies which partition backs each page.

---

## Part 26: Networking From the Ground Up {#part-26-networking-from-the-ground-up}

From electromagnetic waves to HTTPS. Every layer explained by the problem it solves.

### The physical layer: moving bits

A bit must travel from device A to device B. The medium determines how.

**Wires (Ethernet).** Twisted copper pairs carry differential voltage signals. The twisting causes external interference to affect both wires equally; the receiver subtracts, canceling the noise. Gigabit Ethernet uses 4 pairs at 250 Mbps each with PAM-5 encoding. Maximum cable length: 100 meters before signal attenuation makes the voltage levels indistinguishable.

**Fiber optic.** Glass fiber carries pulses of light — laser on = 1, off = 0. No electromagnetic interference. Wavelength-division multiplexing (WDM) sends multiple colors of light through the same fiber simultaneously: 96 wavelengths × 100 Gbps = 9.6 Tbps through a single hair-thin fiber. This is how undersea cables carry intercontinental traffic.

**WiFi.** Radio waves at 2.4 GHz (longer range, slower, crowded) or 5/6 GHz (shorter range, faster, less interference). The transmitter uses OFDM — divides the channel into hundreds of narrow sub-carriers, each carrying a few bits. WiFi 6 uses 1024-QAM, encoding 10 bits per symbol per sub-carrier.

**Cellular.** Licensed spectrum (no interference from other devices). LTE: 700 MHz–2.6 GHz. 5G sub-6: up to 6 GHz. 5G mmWave: 24–40 GHz (extremely fast, blocked by walls). OFDMA lets one tower serve 100+ phones by assigning each a subset of sub-carriers during its time slot.

The physical layer's contract: send a stream of bits from A to B, with some error rate. Bits may be corrupted, lost, or delayed. Higher layers fix these problems.

### The data link layer: talking to neighbors

Your phone and router are on the same WiFi network, along with your laptop, TV, and thermostat. The physical layer can send bits over the radio, but how does the router know which device a message is for?

**MAC addresses.** Every network interface has a unique 48-bit hardware address (e.g., `A4:83:E7:12:34:56`). Each frame carries a destination and source MAC. Every device on the network sees every WiFi frame; each checks the destination MAC and ignores non-matching frames.

**CRC.** A 32-bit checksum at the end of each frame detects corruption from noise. Mismatch = frame silently discarded. Higher layers handle retransmission.

**WiFi medium access (CSMA/CA).** On shared radio, two devices transmitting simultaneously corrupt each other's signals. CSMA/CA: listen before transmitting; if the channel is busy, wait a random backoff time; after transmitting, wait for an ACK. Random backoff prevents synchronized collisions. On a busy network with 20 devices, collisions add 1–50ms of variable latency.

**Switches.** A box with many Ethernet ports. It learns which MAC is on which port by watching incoming frames, then forwards frames only to the correct port — not to all ports.

### IP: crossing networks

MAC addresses are local — they only work within your home network. To reach a server thousands of miles away, you need a global addressing scheme.

**IP addresses.** IPv4: 32-bit addresses (e.g., `142.250.80.46`). ~4 billion addresses, not enough for every device — NAT solves this (below). IPv6: 128-bit addresses, enough for every atom on Earth.

**Routing.** Your phone sends packets to its default gateway (the router). The router forwards to the ISP. Each router has a routing table mapping destination IP ranges to output interfaces. Packets hop router-to-router until reaching the destination's network. `traceroute` shows this path.

**ARP.** Your phone knows the router's IP but needs its MAC to build the WiFi frame. ARP broadcasts "who has 192.168.1.1?" The router responds with its MAC. The mapping is cached. ARP is local-only — at each router hop, MAC addresses change but IP addresses stay the same end-to-end.

**TTL.** Each router decrements the packet's Time To Live by 1. When it hits 0, the packet is discarded. Prevents infinite loops.

### TCP: reliable delivery

IP delivers packets with no guarantees — out of order, lost, duplicated. TCP provides reliable, ordered delivery over this unreliable substrate.

**Three-way handshake.** SYN (my sequence number is X) → SYN-ACK (yours is Y, I got X) → ACK (I got Y). Both sides have synchronized state. One RTT before data flows.

**Sequence numbers.** Every byte has a sequence number. The receiver ACKs by sending the next expected sequence number. If a packet is lost, the ACK doesn't advance — the sender detects this and retransmits.

**Fast retransmit.** Three duplicate ACKs for the same sequence number trigger immediate retransmission, faster than waiting for the timeout (RTO, ~1 second initially).

**Flow control.** The receiver advertises a window size in every ACK — how much buffer space it has. The sender cannot exceed this. Window = 0 means "stop sending, I'm full." This is the receive-side backpressure tied to `sk->sk_rcvbuf` in Part 24.

**Congestion control.** The sender maintains a congestion window (`cwnd`). Slow start doubles `cwnd` each RTT (exponential ramp). After a threshold, growth slows to linear. Packet loss = congestion signal: `cwnd` is halved. The saw-tooth pattern — ramp up, detect loss, cut back — continuously adapts to available bandwidth. Linux uses CUBIC by default; Google uses BBR.

**Ports.** A connection is identified by `(src_ip, src_port, dst_ip, dst_port)`. Well-known ports: 80 (HTTP), 443 (HTTPS), 22 (SSH), 53 (DNS). Client ports are ephemeral (49152–65535).

### UDP: when reliability is wrong

TCP's handshake adds 1 RTT of latency. Retransmission adds delay. In-order delivery causes head-of-line blocking — one lost packet stalls everything behind it. For live video, DNS lookups, and games, UDP is better: just port multiplexing and a checksum. Send a datagram; it either arrives or it doesn't. QUIC (HTTP/3) is built on UDP with its own reliability layer in userspace.

### NAT: sharing one IP

Your ISP gives you one public IP. Your router maintains a NAT table: internal `(private_ip, port)` ↔ external `(public_ip, translated_port)`. Outgoing packets get source-rewritten. Incoming responses get destination-rewritten using the table. External servers only see your public IP. Incoming unsolicited connections are blocked — no table entry exists.

**DHCP.** When your phone joins WiFi, it broadcasts a DHCP Discover. The router responds with an IP, gateway, and DNS server. Four-packet exchange, all broadcast/unicast UDP.

### DNS: names to addresses

DNS is a distributed hierarchical database. Resolution: phone asks resolver (e.g., 8.8.8.8) → resolver asks root server ("where is .com?") → asks TLD server ("where is google.com?") → asks Google's nameserver ("what is www.google.com?") → returns the IP. Four round trips uncached (~80ms). Cached: ~1ms local, ~20ms resolver. Uses UDP port 53.

### TLS: encryption

TLS wraps TCP with confidentiality (encryption), integrity (tampering detection), and authentication (server proves identity via certificate). TLS 1.3 handshake completes in 1 RTT: client sends supported ciphers + ephemeral public key → server responds with chosen cipher + its public key + certificate → both compute a shared secret via Elliptic Curve Diffie-Hellman. All subsequent data is encrypted.

### HTTP: the application layer

With all layers in place, opening `https://www.google.com/search?q=hello`:

```
DNS resolve:  www.google.com → 142.250.80.46           (~20ms cached)
TCP handshake to 142.250.80.46:443                       (~20ms = 1 RTT)
TLS 1.3 handshake (overlaps)                             (~20ms = 1 RTT)
HTTP request: GET /search?q=hello HTTP/2                 
Server processes (query, build HTML)                     (~50ms)
Response: 150KB HTML → ~103 TCP segments                 (multiple RTTs)
Total: ~200-400ms for first meaningful paint
```

HTTP/2 multiplexes requests on one TCP connection — 50 resources requested simultaneously. HTTP/3 (QUIC) uses UDP, so a lost packet for one stream doesn't block others.

### Encapsulation

Each layer wraps the previous layer's data:

```
HTTP request → TLS record → TCP segment → IP packet → WiFi frame → radio waves
```

A 100-byte HTTP request becomes ~189 bytes on the air. The 89 bytes of headers are the cost of addressing, reliability, encryption, and physical delivery. At each router hop, the link-layer frame changes (new MACs) but IP and TCP headers stay the same end-to-end.

### Linux kernel implementation

Sockets are file descriptors. `write(sock_fd, data, len)` → `tcp_sendmsg` → builds sk_buffs → TCP adds headers → IP routes → driver queues for NIC → NIC DMAs and transmits. `read(sock_fd, buf, len)` → `tcp_recvmsg` → copies from `sk_receive_queue` to userspace → frees sk_buffs.

The receive path: NIC DMAs into page_pool page → driver builds sk_buff → NAPI poll → GRO coalesces → IP validates → TCP sequences and ACKs → sk_buff queued on socket → `recv()` copies to userspace.

The send path: userspace data → sk_buff pages → TCP headers → IP headers → link-layer headers → NIC DMA → transmit. sendfile uses page cache pages directly as sk_buff frags — zero-copy send.

Part 24 covers the page-level details: page_pool recycling, sk_buff frag references, and how `_refcount` prevents reclaim from evicting pages mid-DMA.

### WiFi specifics

**Association:** scan for beacons → authenticate → associate → WPA2/3 four-way key exchange. All subsequent frames encrypted with AES.

**Power saving:** phone sleeps between DTIM intervals (~100ms). AP buffers packets. Adds ~100ms latency to the first packet after sleep. WiFi 6/7 Target Wake Time (TWT) reduces this.

**Roaming:** phone monitors signal strength → finds stronger AP with same SSID → reassociates. Transition: 50–500ms. TCP retransmission handles lost packets during roaming.

### Cellular specifics

**Connection:** modem scans frequencies → finds strongest tower → RRC setup → SIM authentication → gets IP address from carrier. Packets: phone → tower (radio) → carrier core (fiber) → internet.

**Handover:** as you drive, the network migrates you between towers. Source tower contacts target → reserves resources → tells phone to switch → phone tunes to new frequency → target starts forwarding. Interruption: 20–50ms on LTE, ~0ms on 5G. IP address preserved via GTP tunnels between core network nodes.

---

## Appendix A: Complete Source Reference {#appendix-a-complete-source-reference}

All source files for the tracer, assembled from the document's fragments with gaps filled. Each file is self-contained and compilable against a GKI kernel with libbpf.

### File 1: `page_tracer.h` — Shared Definitions

Included by both the BPF program and the userspace daemon. Keep this C89-compatible (no flexible arrays, no VLAs).

```c
#ifndef PAGE_TRACER_H
#define PAGE_TRACER_H

enum event_type {
    EVT_ALLOC          = 0,
    EVT_FREE           = 1,
    EVT_CACHE_INSERT   = 2,
    EVT_CACHE_REMOVE   = 3,
    EVT_FAULT          = 4,
    EVT_DIRTY          = 5,
    EVT_WRITEBACK      = 6,
    EVT_WB_DONE        = 7,
    EVT_IO_ISSUE       = 8,
    EVT_IO_COMPLETE    = 9,
    EVT_ANON_FAULT     = 10,
    EVT_COW            = 11,
    EVT_MIGRATE        = 12,
    EVT_ZRAM_COMPRESS  = 13,
    EVT_READ           = 14,
    EVT_READ_DONE      = 15,
    EVT_SWAP_IN        = 16,
};

struct page_event {
    __u64 page_id;
    __u64 pfn;
    __u64 ts;
    __u32 event_type;
    __u32 cpu;
    __u32 tgid;
    __u32 order;
    union {
        struct { __u64 gfp_flags; char comm[16]; } alloc;
        struct { __u64 ino; __u64 dev; __u64 index; } cache;
        struct { __u64 address; __u8 is_write; } fault;
        struct { __u64 old_page_id; __u64 address; } cow;
        struct { __u64 old_pfn; __u64 new_pfn; __u32 reason; } migrate;
        struct { __u64 sector; __u64 dev; __u32 nr_sectors; __u32 error; } io;
        struct { __u32 compressed_size; } zram;
        struct { __u64 old_page_id; } swap;
    };
};

#define FILTER_OFF     0
#define FILTER_BY_COMM 1
#define FILTER_BY_INO  2
#define FILTER_BY_BOTH 3

struct trace_filter {
    __u32 filter_type;
    __u32 pad;
    __u64 ino;
    __u64 dev;
};

struct inode_key {
    __u64 ino;
    __u64 dev;
};

struct sector_key {
    __u64 dev;
    __u64 sector;
};

struct cow_pending {
    __u64 old_page_id;
    __u64 old_pfn;
    __u64 address;
};

#endif
```

### File 2: `page_tracer.bpf.c` — The BPF Program

All maps, helpers, and handlers. Compile with:

```bash
clang -g -O2 -target bpf -D__TARGET_ARCH_arm64 -I. -c page_tracer.bpf.c -o page_tracer.bpf.o
```

```c
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_tracing.h>
#include <bpf/bpf_core_read.h>
#include "page_tracer.h"

char LICENSE[] SEC("license") = "GPL";

// ── Maps ──

struct { __uint(type, BPF_MAP_TYPE_ARRAY); __uint(max_entries, 1);
         __type(key, u32); __type(value, u64); } page_id_counter SEC(".maps");

struct { __uint(type, BPF_MAP_TYPE_HASH); __uint(max_entries, 262144);
         __type(key, u64); __type(value, u64); } pfn_to_id SEC(".maps");

struct { __uint(type, BPF_MAP_TYPE_RINGBUF);
         __uint(max_entries, 16 * 1024 * 1024); } events SEC(".maps");

struct { __uint(type, BPF_MAP_TYPE_ARRAY); __uint(max_entries, 1);
         __type(key, u32); __type(value, struct trace_filter); } filter SEC(".maps");

struct { __uint(type, BPF_MAP_TYPE_ARRAY); __uint(max_entries, 1);
         __type(key, u32); __type(value, u64); } vmemmap_base_map SEC(".maps");

struct { __uint(type, BPF_MAP_TYPE_HASH); __uint(max_entries, 16384);
         __type(key, struct sector_key); __type(value, u64); } sector_to_id SEC(".maps");

struct { __uint(type, BPF_MAP_TYPE_HASH); __uint(max_entries, 8192);
         __type(key, struct inode_key); __type(value, u8); } tracked_inodes SEC(".maps");

struct { __uint(type, BPF_MAP_TYPE_HASH); __uint(max_entries, 1024);
         __type(key, u32); __type(value, u8); } tracked_tgids SEC(".maps");

struct { __uint(type, BPF_MAP_TYPE_HASH); __uint(max_entries, 256);
         __type(key, u32); __type(value, struct cow_pending); } pending_cow SEC(".maps");

struct { __uint(type, BPF_MAP_TYPE_LRU_HASH); __uint(max_entries, 65536);
         __type(key, u64); __type(value, u64); } swap_to_id SEC(".maps");

// ── Helpers ──

static __always_inline u64 get_page_id(u64 pfn) {
    u64 *id = bpf_map_lookup_elem(&pfn_to_id, &pfn);
    return id ? *id : 0;
}

static __always_inline u64 next_page_id(void) {
    u32 zero = 0;
    u64 *counter = bpf_map_lookup_elem(&page_id_counter, &zero);
    if (!counter) return 0;
    return __sync_fetch_and_add(counter, 1) + 1;
}

static __always_inline struct page_event *emit(u64 page_id, u64 pfn,
                                                u32 type, u32 order) {
    struct page_event *e = bpf_ringbuf_reserve(&events, sizeof(*e), 0);
    if (!e) return NULL;
    __builtin_memset(e, 0, sizeof(*e));
    e->page_id = page_id;
    e->pfn = pfn;
    e->ts = bpf_ktime_get_ns();
    e->event_type = type;
    e->cpu = bpf_get_smp_processor_id();
    e->tgid = bpf_get_current_pid_tgid() >> 32;
    e->order = order;
    return e;
}

static __always_inline u64 folio_to_pfn_helper(struct folio *folio) {
    u32 zero = 0;
    u64 *vmemmap = bpf_map_lookup_elem(&vmemmap_base_map, &zero);
    if (!vmemmap || !*vmemmap) return 0;
    u64 folio_addr = (u64)folio;
    if (folio_addr < *vmemmap) return 0;
    u32 page_sz = bpf_core_type_size(struct page);
    if (!page_sz) page_sz = 64;
    return (folio_addr - *vmemmap) / page_sz;
}

static __always_inline bool should_track_tgid(void) {
    u32 zero = 0;
    struct trace_filter *f = bpf_map_lookup_elem(&filter, &zero);
    if (!f || f->filter_type == FILTER_OFF) return false;
    if (f->filter_type == FILTER_BY_INO) return true;
    u32 tgid = bpf_get_current_pid_tgid() >> 32;
    u8 *tracked = bpf_map_lookup_elem(&tracked_tgids, &tgid);
    return tracked && *tracked == 1;
}

// ── Handlers ──

SEC("tracepoint/kmem/mm_page_alloc")
int on_page_alloc(struct trace_event_raw_mm_page_alloc *ctx) {
    u64 pfn = ctx->pfn;
    if (pfn == (u64)-1) return 0;
    if (!(ctx->gfp_flags & 0x08)) return 0;
    if (!should_track_tgid()) return 0;

    u32 tgid = bpf_get_current_pid_tgid() >> 32;
    u64 page_id = next_page_id();
    if (!page_id) return 0;
    bpf_map_update_elem(&pfn_to_id, &pfn, &page_id, BPF_ANY);

    struct cow_pending *cow = bpf_map_lookup_elem(&pending_cow, &tgid);
    if (cow) {
        struct page_event *e = emit(page_id, pfn, EVT_COW, ctx->order);
        if (e) {
            e->cow.old_page_id = cow->old_page_id;
            e->cow.address = cow->address;
            bpf_ringbuf_submit(e, 0);
        }
        bpf_map_delete_elem(&pending_cow, &tgid);
        return 0;
    }

    struct page_event *e = emit(page_id, pfn, EVT_ALLOC, ctx->order);
    if (!e) return 0;
    e->alloc.gfp_flags = ctx->gfp_flags;
    bpf_get_current_comm(&e->alloc.comm, sizeof(e->alloc.comm));
    bpf_ringbuf_submit(e, 0);
    return 0;
}

SEC("tracepoint/kmem/mm_page_free")
int on_page_free(struct trace_event_raw_mm_page_free *ctx) {
    u64 pfn = ctx->pfn;
    u64 page_id = get_page_id(pfn);
    if (!page_id) return 0;
    struct page_event *e = emit(page_id, pfn, EVT_FREE, ctx->order);
    if (e) bpf_ringbuf_submit(e, 0);
    bpf_map_delete_elem(&pfn_to_id, &pfn);
    return 0;
}

SEC("tracepoint/filemap/mm_filemap_add_to_page_cache")
int on_cache_add(struct trace_event_raw_mm_filemap_op_page_cache *ctx) {
    u64 pfn = ctx->pfn;
    u64 page_id = get_page_id(pfn);
    if (!page_id) return 0;

    u32 zero = 0;
    struct trace_filter *f = bpf_map_lookup_elem(&filter, &zero);
    if (f && (f->filter_type == FILTER_BY_INO || f->filter_type == FILTER_BY_BOTH)) {
        struct inode_key ik = { .ino = ctx->i_ino, .dev = ctx->s_dev };
        if (!bpf_map_lookup_elem(&tracked_inodes, &ik)) {
            bpf_map_delete_elem(&pfn_to_id, &pfn);
            return 0;
        }
    }

    struct page_event *e = emit(page_id, pfn, EVT_CACHE_INSERT, ctx->order);
    if (!e) return 0;
    e->cache.ino = ctx->i_ino;
    e->cache.dev = ctx->s_dev;
    e->cache.index = ctx->index;
    bpf_ringbuf_submit(e, 0);
    return 0;
}

SEC("tracepoint/filemap/mm_filemap_delete_from_page_cache")
int on_cache_remove(struct trace_event_raw_mm_filemap_op_page_cache *ctx) {
    u64 pfn = ctx->pfn;
    u64 page_id = get_page_id(pfn);
    if (!page_id) return 0;
    struct page_event *e = emit(page_id, pfn, EVT_CACHE_REMOVE, ctx->order);
    if (!e) return 0;
    e->cache.ino = ctx->i_ino;
    e->cache.dev = ctx->s_dev;
    e->cache.index = ctx->index;
    bpf_ringbuf_submit(e, 0);
    return 0;
}

SEC("kprobe/finish_fault")
int on_fault(struct pt_regs *ctx) {
    struct vm_fault *vmf = (struct vm_fault *)PT_REGS_PARM1(ctx);
    struct folio *folio = BPF_CORE_READ(vmf, folio);
    if (!folio) return 0;
    unsigned long pfn = folio_to_pfn_helper(folio);
    u64 page_id = get_page_id(pfn);
    if (!page_id) return 0;

    struct vm_area_struct *vma = BPF_CORE_READ(vmf, vma);
    struct file *vm_file = BPF_CORE_READ(vma, vm_file);
    u32 evt_type = vm_file ? EVT_FAULT : EVT_ANON_FAULT;

    struct page_event *e = emit(page_id, pfn, evt_type, 0);
    if (!e) return 0;
    e->fault.address = BPF_CORE_READ(vmf, address);
    e->fault.is_write = !!(BPF_CORE_READ(vmf, flags) & 0x02);
    bpf_ringbuf_submit(e, 0);
    return 0;
}

SEC("kprobe/filemap_dirty_folio")
int on_dirty(struct pt_regs *ctx) {
    struct folio *folio = (struct folio *)PT_REGS_PARM2(ctx);
    unsigned long pfn = folio_to_pfn_helper(folio);
    u64 page_id = get_page_id(pfn);
    if (!page_id) return 0;
    struct page_event *e = emit(page_id, pfn, EVT_DIRTY, 0);
    if (!e) return 0;
    bpf_ringbuf_submit(e, 0);
    return 0;
}

SEC("kprobe/wp_page_copy")
int on_cow_entry(struct pt_regs *ctx) {
    struct vm_fault *vmf = (struct vm_fault *)PT_REGS_PARM1(ctx);
    struct page *old_page = BPF_CORE_READ(vmf, page);
    if (!old_page) return 0;
    unsigned long old_pfn = folio_to_pfn_helper((struct folio *)old_page);
    u64 old_page_id = get_page_id(old_pfn);

    u32 tgid = bpf_get_current_pid_tgid() >> 32;
    struct cow_pending val = {
        .old_page_id = old_page_id,
        .old_pfn = old_pfn,
        .address = BPF_CORE_READ(vmf, address),
    };
    bpf_map_update_elem(&pending_cow, &tgid, &val, BPF_ANY);
    return 0;
}

SEC("kprobe/migrate_folio_move")
int on_migrate(struct pt_regs *ctx) {
    struct folio *src = (struct folio *)PT_REGS_PARM3(ctx);
    struct folio *dst = (struct folio *)PT_REGS_PARM4(ctx);
    int reason = (int)PT_REGS_PARM6(ctx);

    unsigned long old_pfn = folio_to_pfn_helper(src);
    unsigned long new_pfn = folio_to_pfn_helper(dst);
    u64 page_id = get_page_id(old_pfn);
    if (!page_id) return 0;

    bpf_map_delete_elem(&pfn_to_id, &old_pfn);
    bpf_map_update_elem(&pfn_to_id, &new_pfn, &page_id, BPF_ANY);

    struct page_event *e = emit(page_id, new_pfn, EVT_MIGRATE, 0);
    if (!e) return 0;
    e->migrate.old_pfn = old_pfn;
    e->migrate.new_pfn = new_pfn;
    e->migrate.reason = reason;
    bpf_ringbuf_submit(e, 0);
    return 0;
}

SEC("kprobe/bio_add_folio")
int on_bio_add_folio(struct pt_regs *ctx) {
    struct bio *bio = (struct bio *)PT_REGS_PARM1(ctx);
    struct folio *folio = (struct folio *)PT_REGS_PARM2(ctx);
    u32 len = (u32)PT_REGS_PARM3(ctx);

    unsigned long pfn = folio_to_pfn_helper(folio);
    u64 page_id = get_page_id(pfn);
    if (!page_id) return 0;

    u64 sector = BPF_CORE_READ(bio, bi_iter.bi_sector);
    struct block_device *bdev = BPF_CORE_READ(bio, bi_bdev);
    dev_t dev = BPF_CORE_READ(bdev, bd_dev);
    u32 opf = BPF_CORE_READ(bio, bi_opf);

    u32 nr_pages = len / 4096;
    if (nr_pages == 0) nr_pages = 1;
    if (nr_pages > 16) nr_pages = 16;

    #pragma unroll
    for (u32 i = 0; i < 16; i++) {
        if (i >= nr_pages) break;
        struct sector_key key = { .dev = dev, .sector = sector + i * 8 };
        bpf_map_update_elem(&sector_to_id, &key, &page_id, BPF_ANY);
    }

    u32 evt_type = (opf & 0xFF) == 1 ? EVT_WRITEBACK : EVT_READ;
    struct page_event *e = emit(page_id, pfn, evt_type, 0);
    if (!e) return 0;
    e->io.sector = sector;
    e->io.dev = dev;
    e->io.nr_sectors = nr_pages * 8;
    bpf_ringbuf_submit(e, 0);
    return 0;
}

SEC("tracepoint/block/block_rq_issue")
int on_rq_issue(struct trace_event_raw_block_rq_issue *ctx) {
    u64 dev = ctx->dev; u64 sector = ctx->sector; u32 nr = ctx->nr_sector;
    #pragma unroll
    for (int i = 0; i < 64; i++) {
        if (i * 8 >= nr) break;
        struct sector_key key = { .dev = dev, .sector = sector + i * 8 };
        u64 *pid = bpf_map_lookup_elem(&sector_to_id, &key);
        if (!pid) continue;
        struct page_event *e = emit(*pid, 0, EVT_IO_ISSUE, 0);
        if (!e) continue;
        e->io.sector = sector + i * 8; e->io.dev = dev; e->io.nr_sectors = 8;
        bpf_ringbuf_submit(e, 0);
    }
    return 0;
}

SEC("tracepoint/block/block_rq_complete")
int on_rq_complete(struct trace_event_raw_block_rq_complete *ctx) {
    u64 dev = ctx->dev; u64 sector = ctx->sector; u32 nr = ctx->nr_sector;
    #pragma unroll
    for (int i = 0; i < 64; i++) {
        if (i * 8 >= nr) break;
        struct sector_key key = { .dev = dev, .sector = sector + i * 8 };
        u64 *pid = bpf_map_lookup_elem(&sector_to_id, &key);
        if (!pid) continue;
        struct page_event *e = emit(*pid, 0, EVT_IO_COMPLETE, 0);
        if (e) {
            e->io.sector = sector + i * 8; e->io.dev = dev;
            e->io.nr_sectors = 8; e->io.error = ctx->error;
            bpf_ringbuf_submit(e, 0);
        }
        bpf_map_delete_elem(&sector_to_id, &key);
    }
    return 0;
}

SEC("kprobe/folio_end_writeback")
int on_wb_done(struct pt_regs *ctx) {
    struct folio *folio = (struct folio *)PT_REGS_PARM1(ctx);
    unsigned long pfn = folio_to_pfn_helper(folio);
    u64 page_id = get_page_id(pfn);
    if (!page_id) return 0;
    struct page_event *e = emit(page_id, pfn, EVT_WB_DONE, 0);
    if (!e) return 0;
    bpf_ringbuf_submit(e, 0);
    return 0;
}

SEC("kprobe/folio_end_read")
int on_read_done(struct pt_regs *ctx) {
    struct folio *folio = (struct folio *)PT_REGS_PARM1(ctx);
    bool success = (bool)PT_REGS_PARM2(ctx);
    unsigned long pfn = folio_to_pfn_helper(folio);
    u64 page_id = get_page_id(pfn);
    if (!page_id) return 0;
    struct page_event *e = emit(page_id, pfn, EVT_READ_DONE, 0);
    if (!e) return 0;
    e->io.error = success ? 0 : 1;
    bpf_ringbuf_submit(e, 0);
    return 0;
}

SEC("kprobe/zram_write_page")
int on_zram_compress(struct pt_regs *ctx) {
    struct page *page = (struct page *)PT_REGS_PARM2(ctx);
    unsigned long pfn = folio_to_pfn_helper((struct folio *)page);
    u64 page_id = get_page_id(pfn);
    if (!page_id) return 0;
    struct page_event *e = emit(page_id, pfn, EVT_ZRAM_COMPRESS, 0);
    if (!e) return 0;
    bpf_ringbuf_submit(e, 0);

    struct folio *folio = (struct folio *)page;
    u64 swap_entry = BPF_CORE_READ(folio, swap.val);
    if (swap_entry)
        bpf_map_update_elem(&swap_to_id, &swap_entry, &page_id, BPF_ANY);
    return 0;
}
```

### File 3: `Makefile`

```makefile
CLANG   ?= clang
BPFTOOL ?= bpftool
CC      ?= gcc
ARCH    ?= $(shell uname -m | sed 's/x86_64/x86/' | sed 's/aarch64/arm64/')

CFLAGS    = -g -O2 -Wall -I.
BPF_FLAGS = -g -O2 -target bpf -D__TARGET_ARCH_$(ARCH) -I.
LDFLAGS   = -lbpf -lelf -lz

all: page_tracer

vmlinux.h:
	$(BPFTOOL) btf dump file /sys/kernel/btf/vmlinux format c > $@

page_tracer.bpf.o: page_tracer.bpf.c page_tracer.h vmlinux.h
	$(CLANG) $(BPF_FLAGS) -c $< -o $@

page_tracer.bpf.skel.h: page_tracer.bpf.o
	$(BPFTOOL) gen skeleton $< > $@

page_tracer: page_tracer.c page_tracer.bpf.skel.h page_tracer.h
	$(CC) $(CFLAGS) -o $@ $< $(LDFLAGS)

clean:
	rm -f page_tracer page_tracer.bpf.o page_tracer.bpf.skel.h vmlinux.h

.PHONY: all clean
```

### Build and run

```bash
# On a machine with kernel headers and clang:
make

# For Android cross-compilation:
adb shell cat /sys/kernel/btf/vmlinux > /tmp/vmlinux.btf
bpftool btf dump file /tmp/vmlinux.btf format c > vmlinux.h
make ARCH=arm64 CC=aarch64-linux-gnu-gcc

# Push and run:
adb push page_tracer /data/local/tmp/
adb shell su -c /data/local/tmp/page_tracer --comm com.android.systemui --verbose
```

### Verification

```bash
# Programs attached:
bpftool prog show | grep page_tracer

# page_id counter advancing:
bpftool map dump name page_id_counter

# pfn_to_id has entries:
bpftool map dump name pfn_to_id | head -5

# Filter configured:
bpftool map dump name filter
bpftool map dump name tracked_tgids
```

---

*All code references are verified against the Linux 7.0-rc7 source tree at commit `e774d5f1bc27a85f858bce7688509e866f8e8a4e`. Every file:line reference has been cross-checked against the actual source. Line numbers may shift between versions, but the structure and concepts are stable.*
