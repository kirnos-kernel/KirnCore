# KirnCore: Complete Technical Specification & Implementation Plan
**The Next-Generation Micro-Hybrid Kernel for KirnOS**  
*Language: Kirn (`.kn`) | Targets: x86_64, AArch64, RISC-V (RV64GC) | Architecture: Capability-Based Micro-Hybrid*

---

## 1. Architectural Philosophy & Kernel Boundary

KirnCore adopts a **Micro-Hybrid** topology: it retains the raw execution speed of a monolithic kernel for virtual memory, scheduling, and IPC, while enforcing the strict fault isolation, crash resilience, and capability security of a microkernel (seL4/Fuchsia) for drivers, filesystems, and userland services.

```
+────────────────────────────────────────────────────────────────────────────────────────────────────────+
| RING 3 / USER SPACE                                                                                    |
|                                                                                                        |
|  +─────────────────────────+   +─────────────────────────+   +──────────────────────────────────────+  |
|  | Native Applications     |   | POSIX / WinNT Subsystem |   | User-Mode Drivers (KDF)              |  |
|  | (.kapp Bundles)         |   | Compatibility Runlines  |   | [NVMe] [GPU] [Net] [KirnFS Service]  |  |
|  +────────────┬────────────+   +────────────┬────────────+   +──────────────────┬───────────────────+  |
|               │                             │                                   │                      |
|               └─────────────────────────────┼───────────────────────────────────┘                      |
|                                             ▼                                                          |
|                    KirnRing Lockless Shared-Memory Syscall ABI                                         |
|                   (Submission Queue [SQ] & Completion Queue [CQ])                                      |
+────────────────────────────────────────────────────────────────────────────────────────────────────────+
| RING 0 / KERNEL SPACE (KirnCore)                                                                       |
|                                                                                                        |
|  +──────────────────────────────────────────────────────────────────────────────────────────────────+  |
|  | Executive Object Manager (Handles, ACLs, Security Descriptors, Lifetime Refcounts)               |  |
|  +──────────────────────────────────────────────────────────────────────────────────────────────────+  |
|  | Capability Security Guard (seL4-Style Immutable Rights Masks, Revocation Trees)                  |  |
|  +──────────────────────────────────────────────────────────────────────────────────────────────────+  |
|  | Grand Task Dispatcher (GTD) (Lock-Free Work-Stealing, Fibers, NUMA Core Affinity)                |  |
|  +──────────────────────────────────────────────────────────────────────────────────────────────────+  |
|  | Virtual Memory Manager (VMM) & Physical Frame Allocator (PMM) (Buddy System, Slab Caches)        |  |
|  +──────────────────────────────────────────────────────────────────────────────────────────────────+  |
|  | KirnProbe Bytecode Engine (eBPF-Inspired Sandboxed Tracing, Packet Filtering & Profiling)         |  |
|  +──────────────────────────────────────────────────────────────────────────────────────────────────+  |
|  | Hardware Abstraction Layer (HAL) (APIC/GIC/PLIC, Page Tables, Context Switching, IOMMU DMA)      |  |
+────────────────────────────────────────────────────────────────────────────────────────────────────────+
```

### 1.1. Invariants & Guarantees
1. **Zero Ambient Authority**: No thread possesses ambient system rights. A thread cannot even query the system time without presenting a valid `Handle` with `CAP_READ_TIME` rights.
2. **Zero-Copy Lockless Ring Syscalls**: Syscalls avoid `syscall`/`sysenter` context-switch overhead during high-throughput I/O. Userland submits requests to a shared ring buffer (`KirnRing SQ`), and kernel worker threads consume them continuously without pipeline flushes.
3. **Isolated Driver Space (KDF)**: Drivers run in isolated Ring 3 address spaces. If the GPU driver or NVMe driver crashes, the kernel resets the process, remaps its capability channels, and restarts it in sub-millisecond time without a kernel panic or Blue Screen.
4. **Predictable Latency**: The scheduler enforces hard priority bands for real-time audio and graphics compositing, preventing interactive stutter under heavy compile-time loads.

---

## 2. Memory Architecture & Address Space Layout

KirnCore implements a strictly partitioned higher-half 64-bit virtual address space.

```
+──────────────────────────────────────────────────────────────────────────+ 0xFFFFFFFFFFFFFFFF
| Kernel Core Text & Static Data (.text, .rodata, .data, .bss)             | 2 GiB
+──────────────────────────────────────────────────────────────────────────+ 0xFFFFFFFF80000000
| Kernel Dynamic Slab & Object Heap                                        | 512 GiB
+──────────────────────────────────────────────────────────────────────────+ 0xFFFFF80000000000
| Higher-Half Direct Map (HHDM) - Complete Physical RAM 1:1 Identity View  | 64 TiB
+──────────────────────────────────────────────────────────────────────────+ 0xFFFF800000000000
| Guard Page Hole (Non-Canonical Canonical Space)                          |
+──────────────────────────────────────────────────────────────────────────+ 0x0000800000000000
| Userland Ring 3 Heap, Thread Stacks, and ASLR Mappings                   | 128 TiB
+──────────────────────────────────────────────────────────────────────────+ 0x0000000000400000
| Null Guard Page (Traps NULL pointer dereferences)                        | 4 MiB
+──────────────────────────────────────────────────────────────────────────+ 0x0000000000000000
```

### 2.1. Physical Memory Manager (PMM)
* **Design**: Hierarchical Buddy Allocator combined with per-NUMA-node allocation zones.
* **Allocation Classes**:
  * `Zone::Dma32`: Physical memory below 4 GiB for legacy 32-bit hardware.
  * `Zone::Normal`: General system RAM.
  * `Zone::HighSpeed`: Low-latency NUMA banks local to the executing CPU package.
* **Buddy Orders**: Orders $0$ to $11$ ($4\text{ KiB} \times 2^{11} = 8\text{ MiB}$ maximum contiguous chunk).

### 2.2. Virtual Memory Manager (VMM)
* **Multi-Architecture Paging**:
  * `x86_64`: 4-level paging (PML4 -> PDPT -> PD -> PT) with 5-level (PML5) dynamic detection.
  * `AArch64`: 4-stage translation tables (L0 -> L1 -> L2 -> L3) with 4KB/64KB granule support.
  * `RISC-V`: Sv39/Sv48 page table walkers.
* **KASLR**: Kernel text and data sections are randomized at boot with 9 bits of entropy provided by UEFI/RNG seed.
* **Page Fault Engine**:
  * Implements true **Demand Paging** (anonymous pages mapped on first write).
  * Implements **Copy-on-Write (CoW)** for instant process spawning without cloning physical frames.

### 2.3. Slab & Object Cache Allocator
Every kernel object (threads, processes, IPC ports, capability handles) is allocated from dedicated, CPU-local, cacheline-aligned `SlabCache` instances to eradicate heap fragmentation and lock contention.

---

## 3. Object Manager & Capability Security Subsystem

Borrowing the architectural elegance of **Windows NT’s Object Manager** and pairing it with **seL4's mathematical capability discipline**, every securable resource is an `ExecutiveObject`.

```
+──────────────────────────────────────────────────────────────────────────+
| Common Object Header (64 Bytes)                                          |
|  - Object Type Pointer (&TypeDescriptor)                                 |
|  - Pointer / Reference Count (atomic[u32])                               |
|  - Handle Reference Count (atomic[u32])                                  |
|  - Security Descriptor / ACL Pointer (&SecurityDescriptor)               |
|  - Monotonic Object ID (u64)                                             |
+──────────────────────────────────────────────────────────────────────────+
| Object Body (Type Specific Payload)                                      |
|  - ProcessObject | ThreadObject | ChannelObject | DeviceObject ...        |
+──────────────────────────────────────────────────────────────────────────+
```

### 3.1. The Capability Handle Table
A process never directly references a kernel pointer. Instead, processes hold an array of 32-bit `Handle` tokens:

```
Process Handle Index (e.g. 0x0000000C)
               │
               ▼
+──────────────────────────────────────────────────────────────────────────+
| Process Handle Table Entry (16 Bytes)                                    |
|  - Object Pointer:        *mut ExecutiveObject                           |
|  - Granted Rights Mask:   CapRights (READ | WRITE | TRANSFER | EXEC)     |
|  - Inheritance Flags:     INHERIT_ON_SPAWN                               |
+──────────────────────────────────────────────────────────────────────────+
```

### 3.2. Rights Mask Matrix
```kirn
pub const CAP_NONE:       u32 = 0x00000000;
pub const CAP_READ:       u32 = 0x00000001;
pub const CAP_WRITE:      u32 = 0x00000002;
pub const CAP_EXECUTE:    u32 = 0x00000004;
pub const CAP_MAP_PAGES:  u32 = 0x00000008;
pub const CAP_SEND_IPC:   u32 = 0x00000010;
pub const CAP_RECV_IPC:   u32 = 0x00000020;
pub const CAP_TRANSFER:   u32 = 0x00000040; // Allow passing handle to another process
pub const CAP_REVOKE:     u32 = 0x00000080; // Allow revoking derived capabilities
pub const CAP_MANAGE_IRQ: u32 = 0x00000100; // Reserved for user-mode drivers
```

---

## 4. The Grand Task Dispatcher (GTD) & Scheduler

KirnCore departs from legacy, complex CFS/O(1) schedulers by implementing a hybrid **M:N Fiber & Thread Work-Stealing Scheduler** inspired by Apple’s libdispatch and Windows thread pools.

```
Physical Core 0: [ RunQueue: Realtime Audio ] -> [ Local Lockless Ring ]
                                                        │ (Work Stealing)
                                                        ▼
Physical Core 1: [ RunQueue: Interactive UI ] -> [ Local Lockless Ring ]
                                                        │ (Work Stealing)
                                                        ▼
Physical Core 2: [ RunQueue: Batch Worker   ] -> [ Local Lockless Ring ]
```

### 4.1. Quality-of-Service (QoS) Bands
1. **`QoS::RealtimeAudio`**: Runs with pinned affinity, zero context jitter, fixed 1-millisecond quantum slices, preempting all other classes.
2. **`QoS::InteractiveUI`**: High responsiveness band for display server and user input events (guaranteed scheduling within 2ms of unblocking).
3. **`QoS::Normal`**: General computation and application background logic.
4. **`QoS::Idle`**: Power-saving maintenance, defragmentation, and background scrub passes.

### 4.2. Work-Stealing Invariant
Every CPU core maintains a local, double-ended lockless work queue (`WorkRing`). If Core $A$ runs out of runnable threads, it attempts to steal tasks from the tail of Core $B$'s queue using atomic Compare-And-Swap (`CAS`) operations, maximizing CPU utilization without global locks.

---

## 5. Syscall Architecture: `KirnRing`

The legacy approach of executing a CPU hardware interrupt for every file read or network packet wastes hundreds of clock cycles per call. `KirnRing` solves this by introducing dual-ring buffers shared between user space and kernel space.

```
USER SPACE                                        KERNEL SPACE
+──────────────────────────+                      +──────────────────────────+
| Submission Queue (SQ)    |                      | Worker Thread Pool       |
|  - Tail: User pushes op  | ─── Zero-Copy DMA ──>|  - Head: Kernel consumes |
|    (Read, Write, IPC)    |                      |    and dispatches task   |
+──────────────────────────+                      +──────────────────────────+
| Completion Queue (CQ)    | <── Status Posted ───| CQ Engine                |
|  - Head: User consumes   |                      |  - Tail: Kernel writes   |
|    completed operation   |                      |    result & error code   |
+──────────────────────────+                      +──────────────────────────+
```

### 5.1. Submission & Completion Queue Specifications
* **Ring Size**: Configurable per process (Default: 512 entries).
* **Memory Invariant**: The memory pages holding the SQ and CQ are pinned physical memory mapped into both user address space and kernel address space simultaneously, eliminating buffer copy operations (`memcpy`).
* **Kernel Doorbell**: If the kernel worker is sleeping, the userland process triggers a lightweight MMIO/IPI write. Under high loads, the kernel spins and batches requests, achieving **zero context switches per syscall**.

---

## 6. Kirn Driver Framework (KDF - User Mode)

In KirnCore, device drivers (NVMe, USB, Wi-Fi, Intel/AMD GPU) are ordinary Ring 3 processes with elevated capability privileges:

1. **MMIO Direct Mapping**: The kernel maps physical PCIe register pages directly into the driver’s userland page table via `CAP_MAP_PAGES`.
2. **IOMMU DMA Virtualization**: Physical device DMAs are strictly mapped through the IOMMU. A malfunctioning driver cannot write to arbitrary kernel memory.
3. **Interrupts as Channels**: When a hardware MSI-X interrupt triggers in Ring 0, the kernel writes an event into the driver’s `KirnRing CQ`. The driver awakes and services the device entirely within user space.
4. **Crash Recovery State Machine**: If a user-mode driver panics, its IPC channel fails with `Err(DriverCrashed)`. The kernel kills the driver process, resets the PCIe function via FLR (Function-Level Reset), launches a fresh driver binary, and reconnects open application handles transparently.

---

## 7. Dynamic Tracing & In-Kernel Sandbox (`KirnProbe`)

KirnCore embeds an internal, statically verified virtual machine (`KirnProbe`) similar to eBPF:
* **Safety Verification**: Before any tracing script is loaded, the `KirnProbe` verifier inspects the bytecode to mathematically prove:
  * No backward jumps (no infinite loops possible).
  * All memory access is bounded within local scratch registers.
  * No access to unverified kernel pointers.
* **Use Cases**:
  1. Low-overhead packet filtering inside the network stack before waking user space.
  2. Latency tracing for scheduling jitter and file system I/O latency.
  3. Dynamic firewall rule evaluation without kernel recompilation.

---

## 8. Complete Modular Code Architecture (`.kn`)

All kernel code is organized under `kernel/` and compiled directly by the Kirn compiler (`knc`):

```text
kernel/
├── main.kn                     # Kernel initialization pipeline
├── arch/
│   └── x86_64/
│       ├── hal.kn              # HAL entry, CPUID, CR0/CR3/CR4 management
│       ├── gdt.kn              # GDT and TSS initialization
│       ├── idt.kn              # IDT vectors & exception dispatches
│       └── context.kn          # Low-level fiber context switching
├── memory/
│   ├── pmm.kn                  # Buddy Physical Frame Allocator
│   ├── vmm.kn                  # 4-Level/5-Level Paging Engine
│   └── slab.kn                 # Object cache & typed slab allocator
├── core/
│   ├── object.kn               # Executive Object Manager
│   ├── process.kn              # Process control block & address spaces
│   └── thread.kn               # Thread states, stacks, fibers
├── sched/
│   └── gtd.kn                  # Grand Task Dispatcher & work-stealing
├── ipc/
│   └── ring.kn                 # KirnRing lockless submission/completion queues
├── security/
│   └── capability.kn           # Capability checks & rights enforcement
└── probe/
    └── vm.kn                   # KirnProbe safe bytecode verifier & executor
```

---

### 8.1. Object Manager Core (`kernel/core/object.kn`)

```kirn
module kernel.core.object;

import kernel.security.capability;
import sync.atomic;

pub enum ObjectType : u16 {
    Process  = 1,
    Thread   = 2,
    Channel  = 3,
    Device   = 4,
    Memory   = 5,
    Event    = 6,
}

@repr(packed)
pub struct ObjectHeader {
    pub object_type:  ObjectType,
    pub pointer_refs: atomic[u32],
    pub handle_refs:  atomic[u32],
    pub object_id:    u64,
    pub security_acl: *const capability::SecurityDescriptor,

    pub fn retain(&mut self) {
        self.pointer_refs.fetch_add(1, Ordering::Relaxed);
    }

    pub fn release(&mut self) -> bool {
        let prev = self.pointer_refs.fetch_sub(1, Ordering::Release);
        if prev == 1 {
            atomic::fence(Ordering::Acquire);
            return true; // Destroy object and free memory
        }
        return false;
    }
}

pub struct HandleEntry {
    pub object_ptr:   *mut ObjectHeader,
    pub rights_mask:  u32,
    pub is_inherited: bool,
}

pub struct ProcessHandleTable {
    entries: [Option[HandleEntry]; 1024],
    capacity: usize,

    pub fn insert(&mut self, obj: *mut ObjectHeader, rights: u32) -> Result[u32, KernelError] {
        for i in 0..1024 {
            if self.entries[i].is_none() {
                unsafe { (*obj).handle_refs.fetch_add(1, Ordering::Relaxed); }
                self.entries[i] = Option::Some(HandleEntry {
                    object_ptr: obj,
                    rights_mask: rights,
                    is_inherited: false,
                });
                return Result::Ok(i as u32);
            }
        }
        return Result::Err(KernelError::HandleTableFull);
    }

    pub fn lookup(&self, handle: u32, required_rights: u32) -> Result<*mut ObjectHeader, KernelError> {
        if handle as usize >= 1024 {
            return Result::Err(KernelError::InvalidHandle);
        }

        match &self.entries[handle as usize] {
            Option::Some(entry) => {
                if (entry.rights_mask & required_rights) != required_rights {
                    return Result::Err(KernelError::AccessDenied);
                }
                return Result::Ok(entry.object_ptr);
            },
            Option::None => Result::Err(KernelError::InvalidHandle),
        }
    }
}
```

---

### 8.2. Physical Frame Allocator (Buddy Allocator) (`kernel/memory/pmm.kn`)

```kirn
module kernel.memory.pmm;

import sync.spinlock;

pub const PAGE_SIZE: usize   = 4096;
pub const MAX_ORDER: usize   = 11; // Chunks up to 8 MiB (4096 * 2048)

pub struct PhysFrame {
    pub paddr: u64,
}

struct FreeBlock {
    next: *mut FreeBlock,
}

pub struct BuddyAllocator {
    lock: spinlock::Spinlock,
    free_lists: [*mut FreeBlock; MAX_ORDER + 1],
    total_memory: u64,
    free_memory: u64,

    pub fn init(&mut self, memory_base: u64, memory_size: u64) {
        self.lock.init();
        self.total_memory = memory_size;
        self.free_memory = 0;

        for i in 0..=MAX_ORDER {
            self.free_lists[i] = null;
        }

        // Add physical memory ranges to highest possible order free-lists
        self.populate_zones(memory_base, memory_size);
    }

    pub fn allocate_pages(&mut self, order: usize) -> Result[PhysFrame, KernelError] {
        if order > MAX_ORDER {
            return Result::Err(KernelError::InvalidOrder);
        }

        self.lock.acquire();
        defer self.lock.release();

        // 1. Locate free block of requested or higher order
        let mut target_order = order;
        while target_order <= MAX_ORDER && self.free_lists[target_order] == null {
            target_order += 1;
        }

        if target_order > MAX_ORDER {
            return Result::Err(KernelError::OutOfPhysicalMemory);
        }

        // 2. Pop node from target list
        let raw_block = self.free_lists[target_order];
        unsafe {
            self.free_lists[target_order] = (*raw_block).next;
        }

        // 3. Split block down to requested order
        while target_order > order {
            target_order -= 1;
            let buddy_paddr = (raw_block as u64) + ((PAGE_SIZE as u64) << target_order);
            let buddy = buddy_paddr as *mut FreeBlock;
            unsafe {
                (*buddy).next = self.free_lists[target_order];
                self.free_lists[target_order] = buddy;
            }
        }

        let allocated_bytes = (PAGE_SIZE as u64) << order;
        self.free_memory -= allocated_bytes;

        return Result::Ok(PhysFrame { paddr: raw_block as u64 });
    }

    pub fn free_pages(&mut self, frame: PhysFrame, order: usize) {
        self.lock.acquire();
        defer self.lock.release();

        let mut curr_paddr = frame.paddr;
        let mut curr_order = order;

        // Coalesce with buddy if available
        while curr_order < MAX_ORDER {
            let buddy_paddr = curr_paddr ^ ((PAGE_SIZE as u64) << curr_order);
            if !self.remove_from_list(buddy_paddr, curr_order) {
                break; // Buddy is allocated; stop coalescing
            }
            curr_paddr = curr_paddr & buddy_paddr; // Keep lower base
            curr_order += 1;
        }

        let block = curr_paddr as *mut FreeBlock;
        unsafe {
            (*block).next = self.free_lists[curr_order];
            self.free_lists[curr_order] = block;
        }

        self.free_memory += (PAGE_SIZE as u64) << order;
    }

    fn remove_from_list(&mut self, paddr: u64, order: usize) -> bool {
        let mut curr = self.free_lists[order];
        let mut prev: *mut FreeBlock = null;

        while curr != null {
            if (curr as u64) == paddr {
                unsafe {
                    if prev == null {
                        self.free_lists[order] = (*curr).next;
                    } else {
                        (*prev).next = (*curr).next;
                    }
                }
                return true;
            }
            prev = curr;
            unsafe { curr = (*curr).next; }
        }
        return false;
    }

    fn populate_zones(&mut self, base: u64, size: u64) {
        // Implementation: Splits memory chunk into order blocks and populates free_lists
    }
}
```

---

### 8.3. Virtual Memory Manager (4-Level Page Walker) (`kernel/memory/vmm.kn`)

```kirn
module kernel.memory.vmm;

import kernel.memory.pmm;

pub const PAGE_PRESENT:  u64 = 1 << 0;
pub const PAGE_WRITE:    u64 = 1 << 1;
pub const PAGE_USER:     u64 = 1 << 2;
pub const PAGE_NX:       u64 = 1 << 63; // No-Execute bit

pub struct PageTable {
    entries: [u64; 512],
}

pub struct VirtualAddress(pub u64);

impl VirtualAddress {
    @inline(always)
    pub fn pml4_index(self) -> usize { return ((self.0 >> 39) & 0x1FF) as usize; }
    @inline(always)
    pub fn pdpt_index(self) -> usize { return ((self.0 >> 30) & 0x1FF) as usize; }
    @inline(always)
    pub fn pd_index(self)   -> usize { return ((self.0 >> 21) & 0x1FF) as usize; }
    @inline(always)
    pub fn pt_index(self)   -> usize { return ((self.0 >> 12) & 0x1FF) as usize; }
}

pub struct AddressSpace {
    pub cr3_physical: u64,

    pub fn map_page(
        &mut self, 
        pmm: &mut pmm::BuddyAllocator, 
        vaddr: VirtualAddress, 
        paddr: u64, 
        flags: u64
    ) -> Result<(), KernelError> {
        let pml4 = self.cr3_physical as *mut PageTable;

        let pdpt = self.get_or_create_subtable(pmm, &unsafe { (*pml4).entries[vaddr.pml4_index()] })?;
        let pd   = self.get_or_create_subtable(pmm, &unsafe { (*pdpt).entries[vaddr.pdpt_index()] })?;
        let pt   = self.get_or_create_subtable(pmm, &unsafe { (*pd).entries[vaddr.pd_index()] })?;

        unsafe {
            (*pt).entries[vaddr.pt_index()] = paddr | flags | PAGE_PRESENT;
        }

        // Invalidate TLB page
        @asm volatile ("invlpg (%0)" : : "r"(vaddr.0) : "memory");

        return Result::Ok(());
    }

    fn get_or_create_subtable(
        &mut self, 
        pmm: &mut pmm::BuddyAllocator, 
        entry: *mut u64
    ) -> Result<*mut PageTable, KernelError> {
        let raw_entry = unsafe { *entry };

        if (raw_entry & PAGE_PRESENT) != 0 {
            let table_paddr = raw_entry & 0x000FFFFFFFFFF000;
            return Result::Ok(table_paddr as *mut PageTable);
        }

        // Allocate a new physical frame for the next-level page table
        let frame = pmm.allocate_pages(0)?;
        let table_ptr = frame.paddr as *mut PageTable;

        // Zero out the newly allocated table
        unsafe {
            @raw_set(table_ptr as *mut u8, 0, 4096);
            *entry = frame.paddr | PAGE_PRESENT | PAGE_WRITE | PAGE_USER;
        }

        return Result::Ok(table_ptr);
    }
}
```

---

### 8.4. `KirnRing` Lockless Syscall Implementation (`kernel/ipc/ring.kn`)

```kirn
module kernel.ipc.ring;

import sync.atomic;

pub const RING_CAPACITY: usize = 512;

pub enum SysOp : u16 {
    Nop         = 0x00,
    MemoryMap   = 0x01,
    MemoryUnmap = 0x02,
    ChannelSend = 0x03,
    ChannelRecv = 0x04,
    DeviceDMA   = 0x05,
    YieldThread = 0x06,
}

@repr(packed)
pub struct SubmissionEntry {
    pub opcode:     SysOp,
    pub flags:      u16,
    pub handle:     u32,
    pub target_buf: u64,
    pub length:     u64,
    pub user_data:  u64,
}

@repr(packed)
pub struct CompletionEntry {
    pub user_data:  u64,
    pub status:     i64,
    pub flags:      u32,
}

pub struct KirnRingBuffer {
    // Shared user/kernel pointers
    pub sq_head: atomic[u32],
    pub sq_tail: atomic[u32],
    pub cq_head: atomic[u32],
    pub cq_tail: atomic[u32],

    pub sq_entries: [SubmissionEntry; RING_CAPACITY],
    pub cq_entries: [CompletionEntry; RING_CAPACITY],

    pub fn poll_and_dispatch(&mut self, handler: fn(&SubmissionEntry) -> i64) -> u32 {
        let mut executed_count: u32 = 0;

        let head = self.sq_head.load(Ordering::Acquire);
        let tail = self.sq_tail.load(Ordering::Relaxed);

        while head != tail {
            let slot = (head as usize) % RING_CAPACITY;
            let entry = self.sq_entries[slot];

            // Execute system operation without context switch
            let result_code = handler(&entry);

            // Post result to completion ring
            self.post_completion(entry.user_data, result_code);

            self.sq_head.store(head + 1, Ordering::Release);
            executed_count += 1;
        }

        return executed_count;
    }

    fn post_completion(&mut self, user_data: u64, status: i64) {
        let cq_tail = self.cq_tail.load(Ordering::Relaxed);
        let slot = (cq_tail as usize) % RING_CAPACITY;

        self.cq_entries[slot] = CompletionEntry {
            user_data: user_data,
            status:    status,
            flags:     0,
        };

        self.cq_tail.store(cq_tail + 1, Ordering::Release);
    }
}
```

---

### 8.5. Grand Task Dispatcher Work-Stealing Scheduler (`kernel/sched/gtd.kn`)

```kirn
module kernel.sched.gtd;

import kernel.core.thread;
import sync.atomic;

pub const MAX_CPUS: usize = 64;

pub struct CoreRunQueue {
    pub core_id:    usize,
    pub queue_head: atomic[u32],
    pub queue_tail: atomic[u32],
    pub ring:       [*mut thread::ThreadControlBlock; 256],

    pub fn push_task(&mut self, task: *mut thread::ThreadControlBlock) -> bool {
        let tail = self.queue_tail.load(Ordering::Relaxed);
        let head = self.queue_head.load(Ordering::Acquire);

        if (tail - head) >= 256 {
            return false; // Queue full
        }

        let slot = (tail as usize) % 256;
        self.ring[slot] = task;
        self.queue_tail.store(tail + 1, Ordering::Release);
        return true;
    }

    pub fn steal_task(&mut self) -> Option[*mut thread::ThreadControlBlock] {
        let mut head = self.queue_head.load(Ordering::Acquire);
        loop {
            let tail = self.queue_tail.load(Ordering::Acquire);
            if head >= tail {
                return Option::None; // Nothing to steal
            }

            let slot = (head as usize) % 256;
            let stolen = self.ring[slot];

            // Attempt atomic CAS steal
            if self.queue_head.compare_exchange(head, head + 1, Ordering::SeqCst) {
                return Option::Some(stolen);
            }
            head = self.queue_head.load(Ordering::Acquire);
        }
    }
}

pub struct GlobalDispatcher {
    pub queues: [CoreRunQueue; MAX_CPUS],

    pub fn schedule_next(&mut self, current_core: usize) -> *mut thread::ThreadControlBlock {
        // 1. Try local core queue
        if let Option::Some(task) = self.queues[current_core].steal_task() {
            return task;
        }

        // 2. Work-Stealing: Inspect other cores
        for i in 0..MAX_CPUS {
            if i != current_core {
                if let Option::Some(stolen_task) = self.queues[i].steal_task() {
                    return stolen_task;
                }
            }
        }

        // 3. Fallback to CPU idle thread
        return thread::get_idle_thread(current_core);
    }
}
```

---

## 9. Delivery Schedule & Implementation Roadmap

```
Sprint 1 (Weeks 1-2):   [ Bare-Metal Boot, GDT, IDT, Serial UART, Physical Memory Manager (pmm.kn) ]
Sprint 2 (Weeks 3-4):   [ 4-Level Paging (vmm.kn), Higher-Half Mapping, Kernel Slab Allocator ]
Sprint 3 (Weeks 5-6):   [ Executive Object Manager (object.kn), Handle Tables, Capability Security ]
Sprint 4 (Weeks 7-8):   [ Grand Task Dispatcher (gtd.kn), Fibers, Low-Level Context Switch Trampoline ]
Sprint 5 (Weeks 9-10):  [ KirnRing Asynchronous Syscall Engine (ring.kn), Lockless IPC Channels ]
Sprint 6 (Weeks 11-12): [ Kirn Driver Framework (KDF), PCIe Enumeration, User-Mode Driver Isolation ]
Sprint 7 (Weeks 13-14): [ KirnProbe Bytecode Verifier, Tracing Subsystem, Profiling Engine ]
Sprint 8 (Weeks 15-16): [ Multicore SMP Bringup (APIC/IPI), Ring 3 Userland Transition, Init Process ]
```

### 9.1. Sprint 1 & 2 Verification Targets
* Kernel boots from Limine bootloader in 64-bit long mode.
* Serial console registers output from all CPU cores.
* `BuddyAllocator` stress tests: Allocate and free 1,000,000 pages of randomized orders ($0$ to $11$) without leaking a single byte.
* Higher-half address translation verified: Accessing `0xFFFFFFFF80000000` translates to physical address `0x0`.

### 9.2. Sprint 3 & 4 Verification Targets
* Handles strictly enforced: An unauthorized thread attempting an operation without the required bitmask returns `KernelError::AccessDenied`.
* Threads run across multiple CPU cores; the Grand Task Dispatcher successfully demonstrates work-stealing when synthetic load is applied to a single core.

### 9.3. Sprint 5 & 6 Verification Targets
* `KirnRing` benchmarks: Userspace issues 2,000,000 asynchronous operations per second without issuing a single hardware interrupt context switch.
* KDF isolation test: An intentional segmentation fault injected into the user-mode storage driver process is caught by KirnCore, resetting the driver without bringing down the operating system.

---

## 10. Bootstrap Harness (`kernel/main.kn`)

```kirn
module kernel.main;

import kernel.arch.x86_64.hal;
import kernel.memory.pmm;
import kernel.memory.vmm;
import kernel.core.object;
import kernel.sched.gtd;
import kernel.ipc.ring;

@export
pub fn kmain(boot_info: *const hal::BootInfo) -> noreturn {
    // 1. Initialize Hardware Abstraction Layer
    hal::early_init();
    hal::serial_print("[KirnCore] HAL initialized. Initializing memory.\n");

    // 2. Initialize Physical Frame Allocator
    let mut pmm_allocator = pmm::BuddyAllocator {};
    pmm_allocator.init((*boot_info).mem_base, (*boot_info).mem_size);
    hal::serial_print("[KirnCore] Physical Buddy Allocator online.\n");

    // 3. Initialize Virtual Memory & Kernel Page Tables
    let mut root_vmm = vmm::AddressSpace { cr3_physical: hal::get_cr3() };
    hal::serial_print("[KirnCore] 4-Level Paging operational.\n");

    // 4. Initialize Grand Task Dispatcher
    let mut dispatcher = gtd::GlobalDispatcher {};
    hal::serial_print("[KirnCore] Grand Task Dispatcher online. Enabling SMP.\n");

    // 5. Initialize System Ring Syscall Engine
    ring::init_subsystem();
    hal::serial_print("[KirnCore] KirnRing IPC active. Spawning Userland Init.\n");

    // Transition CPU into interrupts enabled mode & start scheduler
    hal::enable_interrupts();
    dispatcher.schedule_next(0);

    // Idle loop
    while true {
        hal::cpu_halt();
    }
}
```

This completes the full engineering specification, algorithmic architecture, and source code implementation plan for **KirnCore**, standardized for the **Kirn** (`.kn`) programming language.
