---
layout: default
title: "Linux Memory Management, I/O, and eBPF: A Ground-Up Technical Walkthrough"
---

# Linux Memory Management, I/O, and eBPF: A Ground-Up Technical Walkthrough

*Written for someone who knows userspace memory and I/O well but wants to see what the kernel is actually doing underneath. Every code snippet is from a real kernel tree (v6.x). No handwaving.*

---

## Table of Contents

1. [Physical Memory: How the Hardware Sees It](#part-1-physical-memory-how-the-hardware-sees-it)
2. [The Data Structures That Describe Memory](#part-2-the-data-structures-that-describe-memory)
3. [The Page Fault: Where Everything Starts](#part-3-the-page-fault-where-everything-starts)
4. [Anonymous Memory and Copy-on-Write](#part-4-anonymous-memory-and-copy-on-write)
5. [Dirty Pages and Writeback: Getting Data to Disk](#part-5-dirty-pages-and-writeback-getting-data-to-disk)
6. [The Block I/O Layer: From Pages to Sectors](#part-6-the-block-io-layer-from-pages-to-sectors)
7. [Reclaim: How the Kernel Takes Memory Back](#part-7-reclaim-how-the-kernel-takes-memory-back)
8. [zRAM: Android's Swap Trick](#part-8-zram-androids-swap-trick)
9. [Page Migration: Moving Pages Without Anyone Noticing](#part-9-page-migration-moving-pages-without-anyone-noticing)
10. [Synchronization: How the mm Subsystem Stays Sane](#part-10-synchronization-how-the-mm-subsystem-stays-sane)
11. [eBPF: Observing the Kernel From the Inside](#part-11-ebpf-observing-the-kernel-from-the-inside)
12. [Building a Page Lifecycle Tracer with eBPF](#part-12-building-a-page-lifecycle-tracer-with-ebpf)
13. [Historical Context: How We Got Here](#part-13-historical-context-how-we-got-here)

---

## Part 1: Physical Memory — How the Hardware Sees It

Before any software runs, your hardware has some number of DRAM chips soldered to a board. The memory controller presents these as a flat array of bytes, addressed from 0 to however much RAM you have. The kernel slices this into 4KB chunks called **page frames**. On a phone with 8GB of RAM, that is roughly 2 million page frames.

Each page frame gets an index number — its **PFN** (page frame number). PFN 0 is the first 4KB, PFN 1 is the next, and so on. The kernel maintains a giant array of small descriptor structures, one per page frame, so it can track what each page frame is being used for.

The hardware also has an **MMU** (Memory Management Unit) that sits between the CPU and physical memory. When your program accesses address `0x7f000000`, the CPU does not send that address to the memory controller. Instead, the MMU translates it through a set of **page tables** — a tree structure stored in physical memory — to find the corresponding physical page frame. If no translation exists (the page table entry is empty), the MMU raises a **page fault exception** to the CPU, and the kernel's fault handler runs. This is the most important event in the mm subsystem.

The page tables are a 4-level tree on both x86-64 and ARM64:

```
Virtual address (48 bits on ARM64):
┌─────────┬─────────┬─────────┬─────────┬──────────────┐
│ PGD idx │ PUD idx │ PMD idx │ PTE idx │ page offset  │
│ 9 bits  │ 9 bits  │ 9 bits  │ 9 bits  │ 12 bits      │
└─────────┴─────────┴─────────┴─────────┴──────────────┘
```

Each level is a page-sized array of 512 entries (9 bits = 512). The bottom level, the PTE (Page Table Entry), contains the physical PFN plus permission bits (readable, writable, executable, dirty, accessed). The kernel walks this tree like so:

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

**Why 4 levels?** Because 4 levels of 9-bit indices (4 * 9 = 36 bits) plus a 12-bit page offset gives 48 bits of virtual address space — 256TB. That is enough for now. Some newer CPUs support 5 levels for 57-bit addressing (128PB), but most ARM64 devices use 4.

---

## Part 2: The Data Structures That Describe Memory

Five data structures form the backbone of the mm subsystem. Everything else builds on these, so it is worth spending time here.

### task_struct → mm_struct

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

The `mm_mt` field is a **maple tree** — a B-tree variant optimized for ranges. It replaced the old red-black tree (`mm_rb`) and linked list (`mmap`) in v6.1. The maple tree is more cache-friendly because it packs multiple entries into each node, reducing the number of pointer chases during VMA lookup. On a process with hundreds of VMAs (common on Android — shared libraries, mmap'd files, anonymous regions), this matters.

### vm_area_struct (VMA)

A VMA describes one contiguous region of virtual address space with uniform properties — same permissions, same backing. When you call `mmap()`, the kernel creates a VMA. When you call `munmap()`, it removes one (or splits one).

```c
// include/linux/mm_types.h:913
struct vm_area_struct {
    unsigned long vm_start;              // line 919 — first byte of this region
    unsigned long vm_end;                // line 920 — first byte AFTER this region
    struct mm_struct *vm_mm;             // line 929 — back pointer to the mm
    pgprot_t vm_page_prot;               // line 930 — hardware protection bits
    vm_flags_t vm_flags;                 // line 939 — VM_READ, VM_WRITE, VM_SHARED...
    struct anon_vma *anon_vma;           // line 968 — reverse mapping for anon pages
    const struct vm_operations_struct *vm_ops; // line 971 — fault handler, etc.
    unsigned long vm_pgoff;              // line 974 — file offset in pages
    struct file *vm_file;                // line 976 — the backing file, or NULL
};
```

The `vm_file` field is the key distinguisher:
- **Non-NULL**: This is a file-backed mapping. `vm_file` points to the open file, and `vm_pgoff` says where in the file this region starts.
- **NULL**: This is anonymous memory — heap, stack, or `mmap(MAP_ANONYMOUS)`. There is no file on disk; the data exists only in RAM (or in swap/zram if evicted).

The `vm_ops` field points to a table of function pointers. The most important one is `fault`, which the kernel calls when a page fault happens in this VMA. For file-backed mappings, this is typically `filemap_fault`. For anonymous mappings, the kernel takes a different code path entirely (it does not go through `vm_ops->fault`).

To find the VMA for a given virtual address, the kernel searches the maple tree:

```c
struct vm_area_struct *vma = find_vma(mm, address);
```

This is O(log n) in the number of VMAs. A typical Android app has 200-500 VMAs.

### struct page and struct folio

Every physical page frame has a `struct page` descriptor. These are allocated at boot time in a giant array (the **mem_map** array, or on NUMA systems, per-node arrays). Given a PFN, you can get the `struct page` instantly: `pfn_to_page(pfn)` is just array indexing.

```c
// include/linux/mm_types.h:79
struct page {
    memdesc_flags_t flags;                // PG_locked, PG_dirty, PG_writeback, etc.
    union {
        struct {                          // when used for page cache or anon memory
            struct list_head lru;         // position on LRU list (for reclaim)
            struct address_space *mapping; // who owns this page
            pgoff_t __folio_index;        // offset within that owner
            unsigned long private;        // filesystem-specific data
        };
        struct {                          // when used by the network stack
            unsigned long pp_magic;
            struct page_pool *pp;
            // ...
        };
        struct {                          // when a tail page of a compound page
            unsigned long compound_head;
        };
    };
    atomic_t _mapcount;                   // how many PTEs point to this page
    atomic_t _refcount;                   // reference count
};
```

Notice the `union`. The same 64-byte `struct page` is reused for completely different purposes depending on what the page frame is being used for. The `flags` field tells you which union member is active. This density is deliberate — with 2 million pages, every byte in this struct costs 2MB of RAM at boot.

The **mapping** field is particularly clever:

- For file-backed pages: `mapping` points to the file's `address_space` struct. You follow `mapping->host` to get the inode, which gives you the filename.
- For anonymous pages: `mapping` has its lowest bit set to 1. The rest of the pointer (with bit 0 masked off) points to an `anon_vma` struct, which is used for reverse mapping.
- For free pages: `mapping` is meaningless; the page is on a buddy freelist.

You can test which case applies:

```c
// include/linux/page-flags.h
if (folio->mapping && !((unsigned long)folio->mapping & 1))
    // file-backed
else if (folio->mapping && ((unsigned long)folio->mapping & 1))
    // anonymous
```

A **folio** is a newer abstraction (introduced around v5.16-v5.18) that wraps one or more contiguous `struct page` objects as a single unit. A folio of order 0 is a single 4KB page. A folio of order 4 is 16 contiguous pages (64KB). Large folios reduce overhead — instead of managing 16 separate pages, the kernel manages one folio. The transition from page to folio is still ongoing; much of the mm code now takes `struct folio *` parameters instead of `struct page *`.

In memory, a folio IS a page — they occupy the same bytes:

```c
// include/linux/mm_types.h:401
struct folio {
    union {
        struct {
            memdesc_flags_t flags;             // line 406
            struct address_space *mapping;      // line 421
            pgoff_t index;                      // line 423 — offset in file (in pages)
            atomic_t _mapcount;                 // line 430
            atomic_t _refcount;                 // line 431
        };
        struct page page;                       // line 445 — same memory!
    };
    // second cacheline for large folios:
    atomic_t _large_mapcount;                   // line 454
    atomic_t _nr_pages_mapped;                  // line 455
};
```

`page_folio(page)` is just a cast. No allocation, no indirection.

### The Page Cache (address_space)

Every file that has been read or mmap'd has an `address_space` struct that acts as a cache of the file's contents in RAM. This is the **page cache** — the single most important performance feature in the kernel.

```c
// include/linux/fs.h:470
struct address_space {
    struct inode        *host;            // the inode (file) this cache belongs to
    struct xarray       i_pages;          // the actual cache: file offset → folio
    gfp_t               gfp_mask;         // allocation flags for new pages
    atomic_t            i_mmap_writable;  // how many writable shared mmaps
    struct rb_root_cached i_mmap;         // all VMAs that map this file
    unsigned long       nrpages;          // pages currently in cache
    pgoff_t             writeback_index;  // where writeback left off
    const struct address_space_operations *a_ops; // readpage, writepage, etc.
    unsigned long       flags;            // AS_* flags
    errseq_t            wb_err;           // most recent writeback error
};
```

The `i_pages` xarray is the heart of the page cache. It is indexed by page offset within the file — given a file offset, you can find the corresponding cached page in O(log n) time. The xarray also supports **marks** (tags) on entries, which the kernel uses to efficiently find dirty or writeback pages without scanning the entire cache:

```c
// include/linux/fs.h:497
#define PAGECACHE_TAG_DIRTY     XA_MARK_0   // pages that need writing
#define PAGECACHE_TAG_WRITEBACK XA_MARK_1   // pages currently being written
#define PAGECACHE_TAG_TOWRITE   XA_MARK_2   // pages queued for writeback
```

When the kernel needs a page from a file, it first checks the page cache:

```c
folio = filemap_get_folio(mapping, index);
```

If the page is there (cache hit), no disk I/O is needed — the data is already in RAM. If it is not there (cache miss), the kernel allocates a fresh page, reads the data from disk into it, and inserts it into the page cache for next time. On a typical system, page cache hit rates are 95-99%.

The `i_mmap` rbtree tracks all VMAs that map this file. This is the **reverse mapping** for file pages — given a page, the kernel can find all processes that have it mapped (by looking up the page in `mapping->i_mmap`), which is essential during reclaim when the kernel needs to unmap pages from all processes before freeing them.

### struct bio — the I/O descriptor

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

---

## Part 3: The Page Fault — Where Everything Starts

When your program accesses a virtual address that has no page table entry, or has one with insufficient permissions, the CPU raises a page fault. This is not an error — it is the normal mechanism by which the kernel populates memory on demand. Most pages in a process are set up lazily: `mmap()` creates the VMA (the description of the mapping) but does not allocate any physical pages or page table entries. The first access triggers a fault, and the fault handler does the actual work.

### The entry point

The architecture-specific fault handler (e.g., `arch/arm64/mm/fault.c`) catches the CPU exception, determines the faulting address, looks up the VMA, and calls:

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

### Walking the page tables

`__handle_mm_fault` at `mm/memory.c:6355` walks the page table, allocating intermediate levels as needed:

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

At each level, if the entry is empty (`pud_none`, `pmd_none`), the kernel allocates a new page for the next level of the table. These page table pages are themselves allocated from the buddy allocator with `GFP_KERNEL` — they are non-movable kernel memory. This is why the kernel keeps page tables small: a process that maps enormous address spaces but touches little memory still pays for the intermediate page table levels.

Also notice the huge page checks — at the PUD level (1GB pages) and PMD level (2MB pages on x86, or variable on ARM64), the kernel can take a shortcut and map a single large page instead of walking down to individual PTEs. Android uses Transparent Huge Pages (THP) for some workloads, which can reduce TLB pressure significantly.

### The PTE-level dispatcher

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

### File-backed fault: do_fault and its three sub-paths

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

Let's trace `do_read_fault`:

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

## Part 4: Anonymous Memory and Copy-on-Write

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

Two things worth noting here:

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

## Part 5: Dirty Pages and Writeback — Getting Data to Disk

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

### The writeback thread

The kernel has a background writeback thread (`kworker/u:flush-*`) that periodically wakes up and writes dirty pages back to their filesystems. The wakeup is controlled by two sysctl parameters:

- `dirty_expire_centisecs` (default 3000 = 30 seconds): How long a page must be dirty before writeback picks it up.
- `dirty_writeback_centisecs` (default 500 = 5 seconds): How often the writeback thread wakes up to check.

Writeback also triggers eagerly when the total amount of dirty memory exceeds a threshold (`dirty_ratio` or `dirty_bytes`). This prevents a sudden burst of writes from consuming all available RAM with dirty pages.

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

## Part 6: The Block I/O Layer — From Pages to Sectors

The block layer sits between the filesystem and the disk driver. Its job is to take `bio` structs from the filesystem and deliver them to the hardware efficiently.

### From bio to request

When the filesystem calls `submit_bio(bio)`, the block layer does not immediately send it to the disk. Instead, it tries to **merge** the bio with existing pending I/O:

1. **Front merge**: If the new bio's sectors immediately precede an existing request's sectors, prepend it.
2. **Back merge**: If the new bio's sectors immediately follow an existing request, append it.
3. **No merge**: Create a new request.

Merging is hugely important for performance. Sequential writes to a file produce bios with consecutive sector numbers. Merging turns 100 small 4KB writes into one 400KB write, which is much faster for the disk — fewer commands, fewer interrupts, better utilization of the disk's internal queues.

### The I/O scheduler

Between the merge logic and the hardware dispatch, an I/O scheduler can reorder requests to minimize seek time (for spinning disks) or optimize flash write patterns (for SSDs/UFS). Modern block devices use `blk-mq` (multi-queue block layer), where each CPU has its own submission queue and the hardware may have multiple command queues.

The tracepoints fire at two critical moments:

```c
// block/blk-mq.c:1372
// When a request is sent to the disk driver:
trace_block_rq_issue(rq);

// block/blk-mq.c:891
// When the hardware signals completion:
trace_block_rq_complete(req, BLK_STS_OK, total_bytes);
```

The gap between `block_rq_issue` and `block_rq_complete` is the actual hardware latency — the time the disk takes to do the work. For UFS (the flash storage on modern phones), writes typically take 100-500 microseconds. Spikes to 5-10ms indicate the flash translation layer is doing garbage collection internally.

---

## Part 7: Reclaim — How the Kernel Takes Memory Back

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

## Part 8: zRAM — Android's Swap Trick

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

The compression ratio (total compressed size / total original size) is the key metric. On a typical Android phone, it is around 0.3-0.4, meaning you get 2.5-3x effective RAM capacity from zRAM.

When the page is needed again (a swap fault), `zram_read_page` decompresses it back into a fresh physical page. The decompression is typically faster than the compression — lz4 decompression runs at several GB/s on ARM64.

---

## Part 9: Page Migration — Moving Pages Without Anyone Noticing

Sometimes the kernel needs to move a page from one physical address to another without changing any process's view of memory. This happens during:

- **Compaction**: Defragmenting physical memory to create large contiguous blocks (needed for huge pages or DMA).
- **NUMA balancing**: Moving pages closer to the CPU that accesses them most.
- **Memory hotplug**: Rarely relevant on phones, but the mechanism exists.

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

**Why migration matters for tracing.** If you are tracking pages by PFN, migration breaks your tracking. The PFN changes, but it is logically the same page with the same data and the same owner. Any tracer that uses PFN as a key needs to handle migration or it will lose pages. This is one of the harder problems in building a page lifecycle tracer, and we address it in Part 12.

---

## Part 10: Synchronization — How the mm Subsystem Stays Sane

Multiple CPUs can fault on the same page simultaneously. Reclaim can try to evict a page while a process is mapping it. Writeback can be writing a page while the process is dirtying it again. The mm subsystem uses a layered locking strategy to handle all of this.

### mmap_lock (the big lock)

`mm_struct.mmap_lock` is a reader-writer semaphore that protects the VMA tree:

- **Read-locked** for page faults (you need to find the VMA, but you are not changing it).
- **Write-locked** for `mmap()`, `munmap()`, `mprotect()` (you are creating, destroying, or modifying VMAs).

Multiple page faults can proceed concurrently (read lock is shared). But `mmap()` blocks all faults (write lock is exclusive). This is why `mmap()` inside a hot loop can be a performance problem — it serializes against all page faults in the process.

Recent kernels (v6.4+) added **per-VMA locking**, which narrows the lock from per-mm to per-VMA. Page faults only need to lock the specific VMA they are faulting in, so faults in different regions proceed independently. This is a huge scalability win for processes with many threads faulting on different files.

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

---

## Part 11: eBPF — Observing the Kernel From the Inside

eBPF (extended Berkeley Packet Filter) lets you run small programs inside the kernel without modifying the kernel source or loading traditional kernel modules. Originally designed for network packet filtering, it has evolved into a general-purpose instrumentation framework.

### How eBPF works

An eBPF program is:
1. Written in restricted C (no loops that the verifier cannot prove terminate, no unbounded memory access).
2. Compiled to eBPF bytecode using clang.
3. Loaded into the kernel via the `bpf()` syscall.
4. **Verified** by the kernel's BPF verifier before execution.
5. JIT-compiled to native machine code (ARM64, x86).
6. Attached to a hook point (tracepoint, kprobe, etc.).

The verifier at `kernel/bpf/verifier.c` is the safety net. From its own comment (line 56):

> *"bpf_check() is a static code analyzer that walks eBPF program instruction by instruction and updates register/stack state. All paths of conditional branches are analyzed until 'bpf_exit' insn."*

It performs two passes:

1. **First pass**: Depth-first search to confirm the program is a DAG (Directed Acyclic Graph). Rejects programs with loops, unreachable instructions, or out-of-bounds jumps.
2. **Second pass**: Simulates all execution paths, tracking the **type** of every register. If R1 holds a `PTR_TO_MAP_VALUE`, the verifier knows you can dereference it within the map value's size. If R1 holds a `SCALAR_VALUE`, dereferencing it is rejected.

This means eBPF programs cannot crash the kernel, cannot access arbitrary memory, and always terminate. The tradeoff is that some legal programs are rejected because the verifier cannot prove they are safe — you sometimes need to restructure code to help the verifier along.

### BPF maps: sharing data between kernel and userspace

Maps are the primary data structure. They are kernel-managed key-value stores that both BPF programs and userspace can read and write.

The most common types:

| Map type | Description | Use case |
|---|---|---|
| `BPF_MAP_TYPE_HASH` | Hash table | General key-value lookup |
| `BPF_MAP_TYPE_ARRAY` | Fixed-size array (key is u32 index) | Configuration, counters |
| `BPF_MAP_TYPE_RINGBUF` | Lock-free ring buffer | High-throughput event streaming to userspace |
| `BPF_MAP_TYPE_PERCPU_HASH` | Per-CPU hash table | Counters without atomic contention |
| `BPF_MAP_TYPE_LRU_HASH` | Hash with automatic LRU eviction | Caches with bounded size |

Hash maps are implemented in `kernel/bpf/hashtab.c`. Lookups compute a hash using jhash, index into a bucket array, and walk a linked list. Updates acquire a per-bucket spinlock. The implementation supports both pre-allocated (no allocation at update time, important for use in NMI context) and dynamically-allocated modes.

Ring buffers (`kernel/bpf/ringbuf.c`) are the high-performance choice for streaming events to userspace. They use a multi-producer (multiple BPF programs can write), single-consumer (one userspace thread reads) design with lock-free atomic operations. The buffer pages are double-mapped in kernel virtual memory so that writes that wrap around the end of the buffer appear contiguous — no special wrapping logic needed.

### Attaching to tracepoints

Tracepoints are static instrumentation points compiled into the kernel. They have zero overhead when disabled (the compiler turns them into NOPs using static keys/branches). When a BPF program attaches, the tracepoint becomes active and calls the BPF program on every hit.

A tracepoint definition (from `include/trace/events/kmem.h:180`):

```c
TRACE_EVENT(mm_page_alloc,
    TP_PROTO(struct page *page, unsigned int order,
             gfp_t gfp_flags, int migratetype),

    TP_STRUCT__entry(
        __field(unsigned long, pfn)
        __field(unsigned int, order)
        __field(unsigned long, gfp_flags)
        __field(int, migratetype)
    ),

    TP_fast_assign(
        __entry->pfn       = page ? page_to_pfn(page) : -1UL;
        __entry->order     = order;
        __entry->gfp_flags = (__force unsigned long)gfp_flags;
        __entry->migratetype = migratetype;
    ),
    // ...
);
```

The `TP_STRUCT__entry` defines what data is available to BPF programs. The `TP_fast_assign` block runs at the tracepoint callsite and fills in the data. From BPF, you access these fields via the context pointer:

```c
SEC("tracepoint/kmem/mm_page_alloc")
int my_handler(struct trace_event_raw_mm_page_alloc *ctx) {
    unsigned long pfn = ctx->pfn;
    unsigned int order = ctx->order;
    // ...
}
```

### Kprobes: hooking any kernel function

Tracepoints are stable and efficient, but they only exist where kernel developers placed them. **Kprobes** let you hook any kernel function by dynamically patching the instruction at the function's entry point:

```c
SEC("kprobe/folio_mark_dirty")
int on_dirty(struct pt_regs *ctx) {
    struct folio *folio = (struct folio *)PT_REGS_PARM1(ctx);
    // read fields from folio using bpf_probe_read_kernel()
}
```

The tradeoff: kprobes are less stable (function signatures and internal names change between kernel versions), slightly slower (they use interrupts/breakpoints), and the arguments are raw register values (you need to know the calling convention). Use tracepoints when available; kprobes when no tracepoint exists.

### BTF and CO-RE: writing portable BPF programs

**BTF** (BPF Type Format) is type information embedded in the kernel image. It describes every struct, every field offset, every function signature. With BTF, a BPF program compiled on one kernel version can run on another — the BPF loader uses BTF to **relocate** struct field accesses at load time.

This is called **CO-RE** (Compile Once, Run Everywhere). Instead of hardcoding `offsetof(struct folio, mapping) = 24`, the BPF program records "I want the `mapping` field of `struct folio`" and libbpf patches the offset at load time based on the running kernel's BTF.

You enable this by compiling against a `vmlinux.h` header generated from the kernel's BTF:

```bash
bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h
```

And using `BPF_CORE_READ` macros:

```c
struct address_space *mapping = BPF_CORE_READ(folio, mapping);
unsigned long ino = BPF_CORE_READ(folio, mapping, host, i_ino);
```

---

## Part 12: Building a Page Lifecycle Tracer with eBPF

Now we put it all together. The goal: track every physical page from allocation through page cache insertion, fault, dirty, writeback, I/O, and free. Userspace reads the BPF maps to know who owns any page at any point.

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

### The data model

We store one entry per tracked page:

```c
struct page_info {
    u64 pfn;
    u32 order;
    u64 alloc_ts;           // when allocated
    u64 cache_ts;           // when entered page cache
    u64 fault_ts;           // when faulted into a process
    u64 dirty_ts;           // when first dirtied
    u64 writeback_ts;       // when writeback started
    u64 io_issue_ts;        // when I/O was sent to disk
    u64 io_complete_ts;     // when I/O finished
    u64 free_ts;            // when freed
    u64 dev;                // device major:minor
    u64 ino;                // inode number (identifies the file)
    u64 index;              // offset within the file (in pages)
    u32 tgid;               // process that faulted it in
    char comm[16];          // process name
    u8  page_type;          // file-backed, anonymous, or shmem
};
```

And two maps:

```c
// Central lifecycle map: pfn → page_info
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 65536);
    __type(key, u64);              // pfn
    __type(value, struct page_info);
} page_map SEC(".maps");

// Filter: which process or file to trace
struct {
    __uint(type, BPF_MAP_TYPE_ARRAY);
    __uint(max_entries, 1);
    __type(key, u32);
    __type(value, struct trace_filter);
} filter SEC(".maps");
```

### Handler: page allocation

```c
SEC("tracepoint/kmem/mm_page_alloc")
int on_page_alloc(struct trace_event_raw_mm_page_alloc *ctx) {
    u64 pfn = ctx->pfn;
    if (pfn == (u64)-1) return 0;  // allocation failed

    // Only track user pages (GFP_HIGHUSER_MOVABLE has __GFP_MOVABLE = 0x08)
    if (!(ctx->gfp_flags & 0x08)) return 0;

    u32 tgid = bpf_get_current_pid_tgid() >> 32;

    struct page_info info = {};
    info.pfn = pfn;
    info.order = ctx->order;
    info.alloc_ts = bpf_ktime_get_ns();
    info.tgid = tgid;
    bpf_get_current_comm(&info.comm, sizeof(info.comm));

    bpf_map_update_elem(&page_map, &pfn, &info, BPF_ANY);
    return 0;
}
```

### Handler: page enters page cache

```c
SEC("tracepoint/filemap/mm_filemap_add_to_page_cache")
int on_cache_add(struct trace_event_raw_mm_filemap_op_page_cache *ctx) {
    u64 pfn = ctx->pfn;

    struct page_info *info = bpf_map_lookup_elem(&page_map, &pfn);
    if (!info) return 0;

    info->cache_ts = bpf_ktime_get_ns();
    info->ino = ctx->i_ino;
    info->dev = ctx->s_dev;
    info->index = ctx->index;
    info->page_type = 1;  // file-backed

    return 0;
}
```

### Handler: page freed

```c
SEC("tracepoint/kmem/mm_page_free")
int on_page_free(struct trace_event_raw_mm_page_free *ctx) {
    u64 pfn = ctx->pfn;

    struct page_info *info = bpf_map_lookup_elem(&page_map, &pfn);
    if (!info) return 0;

    info->free_ts = bpf_ktime_get_ns();
    // Don't delete — let userspace read the complete lifecycle first
    return 0;
}
```

### The userspace reader

A simple poll loop that iterates the map, finds completed lifecycles (where `free_ts != 0`), prints them, and cleans up:

```c
int main(int argc, char **argv) {
    struct page_tracer_bpf *skel = page_tracer_bpf__open_and_load();
    page_tracer_bpf__attach(skel);

    int map_fd = bpf_map__fd(skel->maps.page_map);

    while (1) {
        sleep(1);
        u64 key = 0, next_key;
        struct page_info info;

        while (bpf_map_get_next_key(map_fd, &key, &next_key) == 0) {
            bpf_map_lookup_elem(map_fd, &next_key, &info);
            if (info.free_ts != 0) {
                u64 lifetime_us = (info.free_ts - info.alloc_ts) / 1000;
                printf("pfn=%lx ino=%lu off=%lu lifetime=%luus comm=%s\n",
                       info.pfn, info.ino, info.index, lifetime_us, info.comm);
                bpf_map_delete_elem(map_fd, &next_key);
            }
            key = next_key;
        }
    }
}
```

### The PFN problem and migration

The design above uses PFN as the map key. This works until a page migrates — the PFN changes, and the entry becomes stale. You have a few options:

**Option A: Hook migration with a kprobe.** Attach to `migrate_folio_move`. In the handler, look up the old PFN in `page_map`, save the entry, delete the old key, and re-insert under the new PFN.

**Option B: Use a stable page_id.** Patch the kernel to add a monotonically increasing `page_id` field to `struct folio`, assigned at allocation time, never changed. Use this as the map key instead of PFN, and maintain a separate `pfn_to_page_id` lookup map. Migration updates only the lookup map, not the lifecycle entry.

**Option C: Accept the loss.** On many workloads, migration is infrequent. If you are tracing a specific file or process for a few seconds, you may lose very few pages to migration. Start with option A and move to option B if the data shows gaps.

### Ring buffer vs. hash map polling

The hash map approach described above has a nice property: each page's lifecycle is accumulated in a single map entry, and you emit one row per page when it is freed. With a ring buffer, you would emit 5-10 events per page (one per lifecycle stage) and reconstruct the lifecycle in userspace.

Ring buffers are better when:
- You need real-time event streaming (sub-millisecond latency to userspace).
- Event volume is high and you want to avoid map contention.

Hash maps are better when:
- You want to correlate multiple events into one record.
- You can tolerate polling latency (100ms-1s).
- You want to avoid dropped events (ring buffers can drop under pressure; map updates overwrite in place).

For a page lifecycle tracer, the hash map approach is simpler and sufficient.

### Building the BPF program

You need four things:

1. **vmlinux.h** — generated from the running kernel's BTF:
   ```bash
   bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h
   ```

2. **libbpf** — either from `tools/lib/bpf/` in the kernel tree or the standalone repo.

3. **Compile the BPF program:**
   ```bash
   clang -O2 -g -target bpf -D__TARGET_ARCH_arm64 \
       -I./vmlinux/ -c page_tracer.bpf.c -o page_tracer.bpf.o
   ```

4. **Generate the skeleton and compile userspace:**
   ```bash
   bpftool gen skeleton page_tracer.bpf.o > page_tracer.bpf.skel.h
   cc -O2 page_tracer_user.c -lbpf -lelf -lz -o page_tracer
   ```

The skeleton (`page_tracer.bpf.skel.h`) is auto-generated C code that handles loading, map access, and program attachment. It gives you type-safe access to maps and programs without raw file descriptors.

---

## Part 13: Historical Context — How We Got Here

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

eBPF (v3.18, 2014) was a massive expansion: 10 64-bit registers, a 512-byte stack, maps for persistent storage, helper functions for accessing kernel state, and a sophisticated verifier. The key insight was that with a good enough verifier, you could safely run user-provided code inside the kernel — not just for packet filtering, but for tracing, security, scheduling, and more.

The verifier has grown from ~1,000 lines in v3.18 to over 20,000 lines today. Each version adds support for new patterns: bounded loops (v5.3), BPF-to-BPF function calls (v4.16), spinlocks (v5.1), ring buffers (v5.8), and BTF-based type checking that enables CO-RE portability.

### Why the mm subsystem looks the way it does

The mm subsystem's complexity comes from a fundamental tension: **memory is a shared resource with many concurrent users, and every decision involves tradeoffs between throughput, latency, fairness, and power consumption.**

Lazy allocation (demand paging) trades page fault latency for memory efficiency — you never allocate pages you do not use, but every first access costs a fault. COW trades fork latency for copy latency — fork is fast but the first write is slow. The page cache trades RAM capacity for I/O performance — you use RAM to avoid disk access, but now you need reclaim to manage that RAM. zRAM trades CPU cycles for memory capacity — compression costs time but saves space.

Every one of these tradeoffs is tunable, and Android tunes them differently from a server or a desktop. The kernel parameters (`dirty_ratio`, `swappiness`, `compact_proactiveness`, `watermark_scale_factor`) and the scheduling of background daemons (`kswapd`, `kcompactd`, `kworker/flush`) determine how these tradeoffs play out on a specific device.

Understanding the mm subsystem is fundamentally about understanding these tradeoffs — not just what the code does, but why it does it that way, and what would break if you changed it.

---

*All code references are from a v6.x kernel tree. Line numbers may shift between versions, but the structure and concepts are stable.*
