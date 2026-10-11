# Oracle Instance Memory — SGA & PGA From Step 0

A hands-on guide that starts with **why memory exists at all**, builds up a mental
model, names every memory structure, and ends with **how to inspect, resize and
tune each one** using `ALTER SYSTEM`, `ALTER SESSION`, and `V$` views.

> Target: Oracle Database 23ai Free (notes for 19c; **MGA** and **Unified Memory**
> are Oracle AI Database **26ai** features).
> **Oracle Database Free caps itself at 2 CPUs, 2 GB RAM (SGA + PGA combined), and
> 12 GB of user data** — see [8.0](#80-prerequisites-the-container-and-the-sample-hr-schema).
> Examples reference `HR.employees`, which is *not* shipped with 21c+; install it per
> [8.0](#80-prerequisites-the-container-and-the-sample-hr-schema). The running example
> is `UPDATE employees SET salary = 5000 WHERE employee_id = 100;` then `COMMIT;`.
> Companion file: `oracle_architecture.png`; every box is covered — see the
> [diagram coverage checklist](#step-10--diagram-coverage-checklist).
>
> **Numbering.** Part 0 uses **Foundation A…E** (A = NAND, B = von Neumann, C = OS
> theory, D = OS/HW contract, E = Oracle mapping) with subsections `A.1`, `B.2`, … so it
> never collides with **Step 0…11** and subsections `3.1`, `4.4`, … in Part I.
> *Foundation* and *Step* are deliberate, separate namespaces.

---

## Table of contents

**Part 0 — Foundations (read this first)**
- [Foundation A — From a NAND gate to computer architecture](#foundation-a--from-a-nand-gate-to-computer-architecture)
- [Foundation B — Computer architecture: the von Neumann machine](#foundation-b--computer-architecture-the-von-neumann-machine)
- [Foundation C — Operating-system theory](#foundation-c--operating-system-theory)
- [Foundation D — How OS theory is applied to computer architecture](#foundation-d--how-os-theory-is-applied-to-computer-architecture)
- [Foundation E — How it all applies to the Oracle database](#foundation-e--how-it-all-applies-to-the-oracle-database)

**Part I — Oracle instance memory**
0. [The problem memory solves (start here)](#step-0--the-problem-memory-solves-start-here)
1. [Instance vs. database vs. session](#step-1--instance-vs-database-vs-session)
2. [SGA vs. PGA at a glance](#step-2--sga-vs-pga-at-a-glance)
3. [How Oracle decides sizes: Manual, ASMM, AMM, Unified](#step-3--how-oracle-decides-sizes-manual-asmm-amm-unified)
4. [SGA components, one by one](#step-4--sga-components-one-by-one)
5. [PGA components, one by one](#step-5--pga-components-one-by-one)
6. [MGA (Managed Global Area) and namespaces](#step-6--mga-managed-global-area-and-namespaces)
7. [How to manipulate memory (the four-step loop)](#step-7--how-to-manipulate-memory-the-four-step-loop)
8. [Hands-on lab (copy-paste SQL)](#step-8--hands-on-lab-copy-paste-sql)
9. [Cheatsheet: parameters, views, symptoms](#step-9--cheatsheet-parameters-views-symptoms)
10. [Diagram coverage checklist](#step-10--diagram-coverage-checklist)
11. [Security considerations](#step-11--security-considerations)

**Reference**
- [Glossary](#glossary)
- [Practice questions (with answers)](#practice-questions-with-answers)
- [References](#references)

---

## Part 0 — Foundations: from a NAND gate to the SGA

> Read this part first if terms like *MMU*, *virtual address*, *context switch*,
> *page fault*, or *write-back cache* are not automatic yet. Everything Oracle does
> with the SGA and PGA is an application of the ideas below.

Three layers, each one virtualizing the layer beneath it:

```
+----------------------------------------------------------------+
| APPLICATION / DATABASE  (Oracle)                               |
|   builds its own caches, schedulers, locks, and write-ahead log |
+----------------------------------------------------------------+
| OPERATING SYSTEM                                               |
|   virtualizes CPU, memory, and disk for each process           |
+----------------------------------------------------------------+
| HARDWARE  (von Neumann machine)                                |
|   CPU + MMU + RAM + TLB + interrupts + buses + I/O devices      |
+----------------------------------------------------------------+
```

---

### Foundation A — From a NAND gate to computer architecture

> If you have never seen how a handful of logic gates become a CPU, this is the
> 10,000-foot version. The canonical hands-on path is *nand2tetris* (Nisan &
> Schocken, *The Elements of Computing Systems*); what follows is its spine, and it maps
> to its projects: **Project 1** = NAND/gates (§A.2), **Project 2** = the ALU (§A.3),
> **Project 3** = sequential chips & registers (§A.4), **Project 4** = machine language
> (the ISA), **Project 5** = the CPU and the full computer (§A.5), **Project 6** = the
> assembler (§A.6).

The whole machine is built by **layering abstractions** — each level hides the mess
below it and exposes a clean interface to the level above:

```
  application ──► instruction set (ISA) ──► CPU + memory ──►
  ... ──► logic gates (NAND) ──► transistors (switches) ──► atoms
```

Read it bottom-up: atoms make transistors, transistors make gates, gates make
circuits, circuits make the CPU/memory, and those realize an instruction set.

#### A.1 Transistor → switch → bit
- A **MOSFET** is a voltage-controlled switch: a gate voltage turns a channel on/off.
  Map "on/off" to **1/0** and you have a **binary** signal.
- Abstraction: stop thinking in voltages and think in **boolean values**. This is the
  birth of **digital logic**.
- Millions–billions of these switches on one die. **Moore's law** (transistor count
  roughly doubling every ~2 years) is why the ladder below could grow so tall.

#### A.2 The NAND gate: the universal building block
A **logic gate** computes a boolean function of its inputs. **NAND** outputs 0 only
when *all* inputs are 1:

| A | B | A NAND B |
|---|---|---|
| 0 | 0 | 1 |
| 0 | 1 | 1 |
| 1 | 0 | 1 |
| 1 | 1 | 0 |

**NAND is "universal":** *any* boolean function can be built from NAND gates alone,
which is why chips are designed in terms of it:

```
NOT(a)   = NAND(a, a)
AND(a,b) = NOT(NAND(a,b)) = NAND(NAND(a,b), NAND(a,b))
OR(a,b)  = NAND(NOT a, NOT b)                 -- De Morgan's law
XOR(a,b) = NAND( NAND(a, NAND(a,b)), NAND(b, NAND(a,b)) )
```

Build NOT, AND, OR, XOR and you can build everything else (MUX, decoder, comparator…).

#### A.3 Combinational circuits: from gates to the ALU
Wire gates together with **no memory** and you get **combinational logic** — the output
depends only on the current inputs:

- **Adders:** the half adder (XOR for sum, AND for carry) and the **full adder** (takes a
  carry-in). Chain them for a **ripple-carry adder**; use carry-lookahead to go faster.
  This is arithmetic, made of NANDs.
- **Multiplexer (MUX):** select one of N inputs — the fundamental "if".
- **Decoder:** turn an N-bit address into one active line — how memory and register files
  are selected.
- **ALU (Arithmetic Logic Unit):** a bank of functions (add, sub, and, or, not, compare)
  plus a **control input** that selects which one. **This is the ALU that Foundation B
  lists as a CPU component.**

#### A.4 Sequential circuits: memory from feedback
Feed a gate's output back into its input and the circuit **remembers** — this is where a
bit of memory is born:

- **SR latch:** two **cross-coupled NAND gates**; it holds one bit.
- **D flip-flop:** a clocked, edge-triggered latch (a master–slave pair) that safely
  stores a value at a clock edge. **This is the 1-bit memory cell of a register.**
- **Register:** N flip-flops sharing one **clock** — stores an N-bit word.
- **Register file:** an array of registers + a **decoder** (to select one) + a **MUX**
  (to route a value out). Small, fast, and inside the CPU core.
- **Counter / program counter (PC):** a register that increments — drives the fetch cycle.
- A **clock** (an oscillator) gives everything a heartbeat; sequential circuits change
  only on clock edges.

#### A.5 Assembling the components: CPU + memory
Combine the blocks and you have the von Neumann parts of Foundation B:

```
  +----------------------------- CPU ------------------------------+
  |   control unit  (decodes instructions, drives the datapath)     |
  |   ALU           (combinational: arithmetic + logic)             |
  |   registers     (flip-flops; PC, SP, general purpose)           |
  +----------------|-------------------|---------------------------+
                   |  address bus       |  data/control bus
  +----------------|-------------------|---------------------------+
  |   memory: RAM (registers addressed by a decoder) + ROM/BIOS     |
  +-----------------------------------------------------------------+
                   |
      I/O devices (controllers, DMA)  <── interrupts ── CPU
```

- **Control unit:** reads the instruction, sets the ALU op and register enables — it is
  combinational logic plus state.
- **Memory:** a large array of addressable words, built from SRAM/DRAM cells.
- **The stored-program idea** (Foundation B): because instructions live in the same
  memory as data, the same fetch/decode/execute hardware runs *any* program.

#### A.6 The instruction set (ISA) and assembly
The CPU and memory above are only useful once hardware and software agree on a
**vocabulary of instructions** — the **instruction set architecture (ISA)** (nand2tetris
**Project 4** gives you the machine language; **Project 6** writes an **assembler** that
turns mnemonics into it):

- An **instruction** encodes an **opcode** (what to do) plus **operands** (registers, an
  immediate value, or a memory address). Typical classes: arithmetic/logic, load/store,
  branch/jump, and system calls.
- **Addressing modes** (immediate, register, direct, indirect, indexed) decide how an
  operand names its data.
- The ISA is the **contract**: hardware can be built any way you like as long as it
  implements the ISA; compilers and assemblers target the ISA, not the transistors.
- **Assembly** is the human-readable form of that contract (`ADD R1, R2, R3`), and an
  **assembler** translates it to the binary the CPU fetches. This is the layer Oracle's
  compiled code ultimately runs on — the same fetch/decode/execute loop, billions of
  times a second.

#### A.7 The physical bit: SRAM, DRAM, flash
The "1-bit memory cell" above is realized in silicon in a few ways — and this is where
Oracle's memory story physically rests:

- **SRAM** — ~6 transistors per bit, fast, no refresh → **CPU caches** (L1/L2/L3) and TLBs.
- **DRAM** — 1 transistor + 1 capacitor per bit, dense/cheap, needs periodic **refresh**,
  slower → **main memory**, i.e. where the **SGA and PGA** live.
- **NAND flash** — floating-gate cells (named after the **NAND** layout, not the logic
  gate), non-volatile → **SSD**, where Oracle **datafiles and redo logs** live.

So the ladder closes: the NAND that builds logic is the ancestor of the bit, the bit is
the ancestor of DRAM, and DRAM is where the SGA's buffer cache physically sits.

**Hand-off:** with gates we built the CPU and memory. Foundation B now describes those
components from the architecture's point of view.

---

### Foundation B — Computer architecture: the von Neumann machine

#### B.1 The one idea: a stored program
Almost every computer you will touch follows the **von Neumann architecture**
(John von Neumann, *First Draft of a Report on the EDVAC*, 1945). Its defining
feature is the **stored-program concept**: **instructions and data live in the same
memory**, and the CPU fetches and executes them in a cycle.

- **Fetch** the instruction at the address in the program counter (PC).
- **Decode** it in the control unit.
- **Execute** it (ALU operation, memory access, or branch).
- **Store** the result and advance the PC. Repeat.

Contrast: a **Harvard architecture** has separate memories for instructions and data
(common in microcontrollers and in the instruction/data caches of modern CPUs).

#### B.2 The components
- **CPU (central processing unit)**
  - **Control unit** — fetches/decodes instructions and steers the datapath.
  - **ALU (arithmetic logic unit)** — integer/bitwise arithmetic; the FPU does floating point.
  - **Registers** — the fastest storage, inside the core:
    - `PC` (program counter), `IR` (instruction register)
    - `SP` (stack pointer), general-purpose registers, status/flags
    - `MAR`/`MDR` (memory address / data registers) in the classic model.
- **Main memory (RAM)** — a linear array of addressable bytes/words holding *both*
  code and data. Address **0 … 2^N − 1**.
- **Buses** — **address bus** (where), **data bus** (what), **control bus** (how).
- **I/O devices + controllers** — storage, NIC, GPU, timers; talk to the CPU via
  interrupts and to memory via DMA.
- **Persistent secondary storage** — disks/SSD. Not strictly part of the core model,
  but essential because RAM loses its contents on power loss.

#### B.3 The memory hierarchy and the von Neumann bottleneck
The CPU is far faster than DRAM, so a single master-memory design wastes most of the
CPU's time waiting. This throughput limit is the **von Neumann bottleneck**. It is
softened by a **hierarchy of caches** that exploit **locality of reference**
(*temporal*: recently used → reused; *spatial*: neighbors → used together):

| Level | Typical latency | Size |
|---|---|---|
| Registers | < 1 ns | hundreds of bytes |
| L1 cache | ~1 ns (≈ 4 cycles) | tens of KB |
| L2 cache | ~4 ns (≈ 12 cycles) | hundreds of KB–MB |
| L3 cache | ~10–20 ns (≈ 40 cycles) | tens of MB (shared) |
| DRAM (main memory) | ~60–100 ns | GB |
| NVMe SSD | ~20–100 µs | TB |
| HDD (random) | ~5–15 ms | TB |

Perspective: RAM is a few hundred times faster than a random SSD read (the table's
~60–100 ns vs. 20–100 µs is roughly **200–1,000×**) and ~100,000× faster than a random HDD
read. **That gap is the entire reason databases cache.** (This is the same point Step 0 will
make for Oracle specifically.)

#### B.4 Hardware primitives the operating system depends on
The OS cannot be built on the CPU alone. It leans on a handful of hardware features —
remember these, because they reappear as Oracle mechanisms:

- **Interrupts / traps / exceptions** — asynchronous signals (device I/O, timer) and
  synchronous faults (divide-by-zero, page fault) that transfer control to privileged code.
- **Privilege levels (rings)** — at least two: **user mode** and **kernel mode**.
  Privileged instructions (halt, set page tables, mask interrupts) run only in kernel mode.
- **MMU (Memory Management Unit)** — translates virtual addresses to physical addresses
  on every access and enforces page permissions.
- **TLB (Translation Lookaside Buffer)** — a small, fast cache of recent translations.
- **DMA (Direct Memory Access)** — a device moves data to/from RAM *without* the CPU
  copying it byte by byte; the CPU is interrupted only when the transfer finishes.
- **Atomic instructions** — compare-and-swap, test-and-set, load-linked/store-conditional:
  the building blocks of locks.

#### B.5 Hardware details that decide database performance
These low-level facts explain *why* latches, buffer caches, and `latch`/`mutex` waits
behave as they do, and they are the hardware behind Foundation C §C.2:

- **Cache lines (64 B on x86)** are the transfer unit. Touching one byte pulls the whole
  line; unrelated data packed in one line causes **false sharing** when two cores write
  it, bouncing the line between caches.
- **Cache coherence (MESI/MOESI)** keeps cores' copies consistent: a write **invalidates**
  other cores' copies. So *write-heavy shared data scales worse than read-mostly shared
  data* — which is why Oracle's read-mostly buffer cache and write-heavy redo/undo have
  very different contention profiles.
- **Memory ordering & barriers** — CPUs reorder loads/stores; `mfence`/`lock` and
  acquire/release semantics make locks correct. A latch's atomic test-and-set is only
  safe with the right fences.
- **NUMA** — access time depends on which socket owns the page; cross-socket traffic is
  far slower (see §C.3).
- **Granularity mismatch:** a 4 KB **page**, an 8 KB Oracle **block**, a 512 B/4 KB
  **redo** sector, a 64 B **cache line**. Changing one row dirties a whole 8 KB block and
  writes a redo record — the unit of I/O is the block, never the row.

---

### Foundation C — Operating-system theory

A classic way to organize the discipline (from the book *Operating Systems: Three Easy
Pieces*) is into **virtualization**, **concurrency**, and **persistence**. We follow it.

#### C.1 The OS as illusionist and resource manager
Physical reality: **one or a few CPUs**, a **finite pool of RAM**, a **few disks**, and
shared devices. The OS turns that into **virtual** resources so each program behaves as
if it owns the machine:

- each process gets the illusion of its **own private CPU**;
- each process gets its **own private, contiguous address space**;
- each process sees a **private, persistent file system**.

Two principles make this safe:

- **Mechanism vs. policy** — *mechanism* is how to do something (context switch,
  translate an address); *policy* is which choice to make (which process runs next,
  which page to evict). Keep them separate.
- **Controlled entry** — user programs cross into the kernel only through **system
  calls**, via a trap. The kernel is the only code that runs privileged.

#### C.2 CPU virtualization: processes, threads, scheduling
- **Process** = a running program: its **address space** + **registers** (incl. PC) +
  open files + kernel state. The *program* is static; the *process* is one instantiation in motion.
- **Process states:** `new → ready → running → blocked/waiting → terminated`.
  Only one process runs per core at a time; the rest are `ready` or `blocked`.
- **PCB (process control block)** — the kernel's data structure tracking a process
  (registers saved at switch time, page-table base, scheduling info, file table).
- **Context switch** — save the running process's registers/PC, restore another's, and
  jump. It costs more than the instructions suggest: it **pollutes caches and the TLB**.
- **Threads** — multiple execution contexts inside one process. Threads **share** the
  address space, heap, code, and open files, but each has its **own registers, PC, and
  stack**. Cheaper to create/switch than processes, but they can corrupt shared data.
- **CPU scheduling** — the *policy* that picks the next ready thread:
  - **Metrics:** turnaround time, response time, throughput, fairness, CPU utilization.
  - **Algorithms:** FCFS/FIFO → SJF → STCF (preemptive SJF) → **Round Robin** (time
    quantum) → priority → **MLFQ** (multi-level feedback queue) → proportional-share
    (lottery/stride) → **CFS/EEVDF** in modern Linux.
  - **Preemption** — a **timer interrupt** lets the kernel take the CPU back, which is
    what makes interactive systems feel responsive.
- **Concurrency & synchronization** — shared data + parallelism = **race conditions**.
  - **Critical sections** guarded by **mutexes, spinlocks, semaphores, condition variables**.
  - **Deadlock** needs four conditions at once (Coffman): mutual exclusion, hold-and-wait,
    no preemption, circular wait.
  - **Starvation** — a thread never gets its turn; fairness policies prevent it.

#### C.3 Memory virtualization: address spaces, paging, TLBs
- **Address space** — the abstraction each process sees: code, heap, stack, shared
  libraries, in a flat virtual range (e.g. `0x0 … 0x7fff_ffff_ffff`). Benefits:
  **isolation, protection, relocation, sparse use, sharing**.
- **Mechanisms, historically:**
  1. **Base & bounds** — add a base to every address, reject if beyond a limit.
  2. **Segmentation** — several variable-length segments (code/data/stack) with
     base, limit, and permissions.
  3. **Paging** — the winner. Split the virtual space into fixed **pages** and physical
     memory into equal **frames**; a **page table** maps *virtual page number (VPN)* →
     *physical frame number (PFN)*.
- **The MMU** does the translation in hardware, on **every** memory access.
- **Page tables** get huge, so they are **multi-level (radix)** trees, hashed, or
  inverted. A translation may require several memory reads (a **page-table walk**).
- **TLB (Translation Lookaside Buffer)** — a tiny cache of recent VPN→PFN translations.
  - **TLB hit:** translation is essentially free.
  - **TLB miss:** the MMU walks the page table (slow), then caches the result.
  - **TLB reach** = entries × page size; **huge pages** increase reach and cut misses.
  - A **context switch** or a page-table change can require a **TLB flush/shootdown**.
  - Effective time ≈ `hit% × (TLB + mem)` + `miss% × (TLB + walk + mem)` — so a high hit
    ratio is worth everything.
- **Page faults** — accessing a page not in RAM traps into the kernel, which finds a free
  frame (or evicts one), loads the page (from swap/file), updates the page table, and
  **restarts the instruction**. A **major** fault needs disk I/O; a **minor** fault does not.
- **Replacement policy** — when memory is full: OPT (Belady's optimum, unreachable),
  **LRU**, **clock/second-chance**, FIFO. Real systems approximate LRU with
  hardware **accessed/dirty bits**.
- **Demand paging & overcommit** — pages are loaded only when used; total virtual memory
  may exceed RAM. Too little physical RAM ⇒ **thrashing** (spending all time paging).
  The **working set** is the pages a process actually needs.
- **Copy-on-write (COW)** — after `fork`, parent and child share pages read-only until one
  writes, then the kernel copies just that page. Also used for snapshots/clones.
- **Memory-mapped files & shared memory** — map a file or a shared region into an address
  space so processes can share bytes and let the MMU do the work.
- **Huge pages** and **NUMA** — larger pages reduce TLB pressure; on multi-socket machines
  access to a *remote* socket's memory is slower (non-uniform memory access).

> **UGA / PGA / SGA hook:** an *address space* is private memory; a *shared memory
> segment* is memory mapped into many address spaces at once. Oracle's PGA is the first;
> Oracle's SGA is the second.

#### C.4 Persistence: disks, file systems, durability
- **HDD mechanics** — spinning platters, tracks, and sectors. Access time =
  **seek** (move the arm) + **rotational latency** (wait for the sector) + **transfer**.
  **Seek dominates**, so *sequential* beats *random* I/O by orders of magnitude.
- **SSD** — NAND flash, no moving parts, roughly **µs** latency. Random reads are fast,
  but writes must erase whole blocks (**write amplification**), handled by the **FTL**
  (flash translation layer) and **wear leveling**. `TRIM` reclaims freed blocks.
- **RAID** — combine disks for **capacity, performance, and reliability**:
  RAID 0 (stripe), 1 (mirror), 5/6 (parity), 10 (mirror+stripe).
- **File systems** — **inodes** (metadata), directories, and **extents** (contiguous
  block runs). **Journaling** keeps the metadata consistent across a crash.
- **I/O scheduling** — the block layer reorders/merges requests (elevator, deadline,
  `mq-deadline`, `none`/`kyber` for NVMe) and pipelines via **queue depth**.
- **Page cache & write-back** — the OS caches file blocks in RAM. **Write-back** defers
  disk writes for speed; **write-through** and **`fsync`** force durability. **`O_DIRECT`**
  bypasses the page cache for applications (like databases) that cache aggressively.
- **Asynchronous I/O (AIO) and `io_uring`** — issue many I/Os and continue computing;
  the completion arrives later as a notification. Essential for overlapping disk latency.
- **Crash consistency** — to guarantee durability you need ordering (barriers/`fsync`);
  the classic pattern is **write-ahead logging**: write the *intent* sequentially and
  flush it *before* the data, so you can replay or roll back after a crash.

#### C.5 The process image and the allocator
- **ELF layout** of a running program: `text` (code, read-only), `data`/`bss`
  (initialized/uninitialized globals), **heap** (grows up), **stack** (grows down), plus
  memory-mapped regions (shared libraries, files, shared memory).
- **The allocator:** `malloc` does **not** call the kernel per allocation. It carves from
  the heap, extends it with **`brk`/`sbrk`**, and for large blocks calls **`mmap`**.
  Freeing may not return memory to the OS (glibc often retains it).
- **`fork`** copies the parent via **copy-on-write** (§C.3); **`exec`** replaces the image.
  Oracle builds on this: dedicated servers are separate **processes**, while
  **`threaded_execution`** maps them to **threads** in one process.
- **ASLR** and **stack canaries** are why addresses aren't fixed and why memory bugs are
  security bugs.
- **Why this matters for the PGA:** "the PGA is a process heap" means each server process
  `malloc`s/`mmap`s its own work areas and UGA — which is exactly why it is *private*, and
  why its size tracks per-session work (sorts, hashes), not the instance.

---

### Foundation D — How OS theory is applied to computer architecture

OS virtualization is not magic: each "illusion" is bought from a specific hardware
feature. This is the contract between the OS and the machine.

| OS capability | Hardware that makes it possible |
|---|---|
| User vs. kernel mode | Privilege rings, privileged instructions, `syscall`/`trap` |
| Preemptive multitasking | **Timer interrupts** + the ability to save/restore registers |
| Fast context switch | Register file, PC, per-process page-table base register (e.g. `CR3`) |
| Private address spaces | **MMU** + page tables + page-fault exception |
| Fast translation | **TLB**; huge pages for reach |
| Page replacement & COW | PTE **accessed/dirty** bits, write-protection |
| Safe sharing | Page permissions; shared mappings of the same physical frames |
| Synchronization | **Atomic instructions** (CAS, test-and-set, LL/SC) |
| Efficient I/O | **DMA**, device controllers, interrupt coalescing |
| Durable writes | Block devices with flush/barrier semantics |

Put differently: **hardware provides primitives, the OS composes them into processes,
threads, address spaces, and files, and applications compose those into services.** Every
abstraction has a **performance cost** — a system call is a mode switch, a context switch
pollutes caches/TLB, a page fault is a disk trip — and good system design (and good
database design) is largely about **avoiding those costs**.

---

### Foundation E — How it all applies to the Oracle database

Now the payoff: an Oracle **instance** is, quite literally, a set of OS processes and OS
memory regions that implement all of the above *again*, one level up — a database inside
the operating system. The pieces you will learn in Step 0 onward are the mapping below.

#### E.1 Processes and scheduling
- Every **server process** is an **OS process** (or thread) serving one session; each
  **background process** (DBWn, LGWR, SMON, PMON, CKPT, MMON, ARCn…) is a **daemon**
  doing asynchronous housekeeping — exactly the OS process model from §C.2.
- **Dedicated server** = process-per-connection. **Shared server** = a **pool** of
  server processes behind dispatchers, a thread-pool pattern that cuts context-switch and
  memory overhead (§C.2) — at the cost of putting session state in the SGA.
- **Parallel query** runs many slave processes; **Oracle Resource Manager**
  (`DBMS_RESOURCE_MANAGER`, consumer groups, CPU shares) is a **second scheduler** layered
  on top of the OS scheduler — policy above policy (§C.1).
- **Latches and mutexes** are Oracle's spinlocks/mutexes (§C.2), built on the CPU's
  **atomic instructions** and tuned with adaptive spinning to avoid expensive kernel
  context switches. **Enqueues** are a different mechanism: the *lock manager* for
  row/table/distributed locks — with waiters, lock queues, convert operations, and
  **deadlock detection**. A latch is a tiny, brief SGA spinlock; an enqueue is a
  higher-level, blocking lock you can wait on.

#### E.2 Memory
- The **SGA is a shared memory segment**: Oracle obtains it with **`shmget()`** and maps it
  into each process with **`shmat()`** (System V), or with **`mmap(..., MAP_SHARED, ...)`**
  (POSIX / `memfd`); every Oracle process **attaches** the same physical pages — the
  shared-memory abstraction of §C.3, which is why the SGA is "shared" and the PGA is
  "private." Foundation C said "shared memory"; these are the actual system calls.
- The **PGA is each server process's private heap** — the process address space of §C.3.
- The **UGA** is session memory: in the **PGA** for dedicated servers, in the **large
  pool** for shared servers — a deliberate trade of isolation for consolidation.
- The **database buffer cache is the database's own page cache**: data blocks instead of
  file pages, with its own **LRU + touch-count** replacement (§C.3), and its own
  **KEEP / RECYCLE** pools. Oracle often uses **direct I/O** (`filesystemio_options=SETALL`)
  to **bypass the OS page cache** and avoid **double caching** (§C.4).
- **HugePages** for the SGA dramatically reduce **TLB misses** (§C.3) on large instances;
  this is why Oracle recommends them on Linux — and why large systems use **ASMM rather
  than AMM**, because AMM (`MEMORY_TARGET`) is **incompatible with HugePages** on Linux.
- **Swap is a last resort for the SGA/PGA**: paging a data block out to disk turns a
  ~100 ns access into a ~ms one. But Oracle's install requirements **mandate swap**. The
  Free installation guide states it exactly: *"For Oracle Database Free, Oracle recommends a
  minimum swap space of 2 GB or twice the size of RAM, whichever is less"*
  ([26ai Free Requirements](https://docs.oracle.com/en/database/oracle/oracle-database/26/xeinl/requirements.html)).
  So the right move is to keep a *little* swap and **lower `vm.swappiness`** (e.g. 1–10) so
  the kernel prefers evicting file cache over database memory. Oracle adds
  **`PGA_AGGREGATE_LIMIT`** as a database-level guard (analogous to a **cgroup** limit)
  against the OS **OOM killer** (§C.3, overcommit/thrashing).
- On multi-socket servers, **NUMA** means some SGA memory is "remote"; Oracle is
  **NUMA-aware** to keep a process near its memory (§C.3).
- Sizing maps directly: `SGA_TARGET` (shared) vs `PGA_AGGREGATE_TARGET` (private) vs
  newer **MGA** (semi-shared), and **Unified Memory** `MEMORY_SIZE` above both.

#### E.3 Persistence
- **LGWR + the redo log are a write-ahead log** (§C.4). Redo entries are written
  *sequentially* and flushed on **commit** — the same intent-before-data ordering that
  makes file systems crash-safe. `COMMIT` returning means the redo is durable.
- **DBWn is the write-back daemon** (§C.4): it lazily flushes *dirty* buffer-cache blocks;
  a **checkpoint** advances the point from which recovery must replay. **SMON** performs
  **instance recovery** by replaying redo — database-level journaling/recovery.
- **ASM (Automatic Storage Management)** is a **volume manager + file system for Oracle
  files**, providing striping, mirroring, and rebalancing — the RAID/LVM role of §C.4 at
  the database layer.
- Oracle uses **asynchronous I/O** and direct I/O so that DBWn/ARCn/RMAN overlap many
  disk operations while the CPU keeps working (§C.4). On recent Linux kernels Oracle *may*
  use **`io_uring`** where available and fall back to **`libaio`**; Oracle has not published
  a per-path breakdown of which I/O goes through which interface, so **verify against your
  release's platform guide** rather than assuming which paths use `io_uring`.

#### E.4 The whole mapping on one page

| OS / architecture concept | Oracle equivalent |
|---|---|
| Process / thread | Server process; background processes; parallel slaves |
| Thread pool | Shared server + dispatchers; connection pooling |
| Private address space | PGA (per server process) |
| Shared memory segment | SGA (attached by all processes) |
| Session state abstraction | UGA (in PGA or large pool) |
| Page cache + LRU | Database buffer cache (LRU + touch count, KEEP/RECYCLE) |
| Write-back daemon | DBWn; checkpoints by CKPT |
| Write-ahead log + `fsync` | Redo log buffer + LGWR; commit durability; group commit |
| Crash recovery / journal | Instance recovery / SMON replaying redo |
| Spinlock / mutex | Latch / mutex (short SGA spins); enqueues are a separate lock manager |
| CPU scheduler + cgroups | OS scheduler + Oracle Resource Manager (consumer groups, shares) |
| Timer preemption | OS preempts server processes; Oracle wait/timeout model |
| TLB / huge pages | HugePages for the SGA (`USE_LARGE_PAGES`) |
| Swap / OOM killer | Keep a little swap + low `vm.swappiness`; `PGA_AGGREGATE_LIMIT`; AMM `/dev/shm` |
| NUMA | NUMA-aware SGA placement and latches |
| RAID / LVM / FS | ASM (striping, mirroring, rebalance) |
| Async I/O / `O_DIRECT` | `filesystemio_options=ASYNC/DIRECTIO/SETALL`; `io_uring` where available, else `libaio` (verify per release) |
| Daemons | SMON, PMON, CKPT, MMON, MMNL, ARCn |
| mmap of a file | Datafiles / tempfiles mapped and read in blocks |

With this model in hand, Step 0 is simply the database version of one question:
**which bytes do we keep in RAM, and which do we fetch from disk?**

---

## Part I — Oracle instance memory

## Step 0 — The problem memory solves (start here)

Forget Oracle for a second. You have two kinds of storage:

| | Disk (datafiles) | RAM |
|---|---|---|
| Speed to reach a byte | ~20 µs (NVMe SSD) … ~10 ms (HDD) | ~100 ns |
| Cost per GB | cheap | expensive |
| Size | huge (TB/PB) | small (GB) |
| Survives power loss | yes | no |

RAM is roughly **200–1,000× faster** than a random **NVMe SSD** read and **~100,000×**
faster than a random **HDD** read. So every serious database is
fundamentally a machine that answers one question over and over:

> **Which bytes do we keep in RAM, and which bytes do we go to disk for?**

Oracle's entire memory architecture is the answer to that question. If you remember
only this, the rest is vocabulary:

- A **data block** that has been read is *cached in RAM*, so the next read is fast.
- A **parsed SQL statement** is *cached in RAM*, so the next execution skips parsing.
- **Sorted/hashed intermediate results** live in RAM while a query runs.

There are only two follow-up questions that decide **where** a byte lives:

1. **Shared or private?** If many sessions need it (a data block, a parsed plan) it
   goes in the **shared** area → SGA. If only one session needs it (your sort area,
   your bind values) it goes in the **private** area → PGA.
2. **Compute or data?** Reusable *derived* things (parsed SQL, execution plans,
   table metadata) go in the shared-pool family. Actual table/index *blocks* go in
   the buffer cache.

That's the whole game. Everything below is detail on those two questions.

---

## Step 1 — Instance vs. database vs. session

Three words that get mixed up constantly:

- **Database** = the files on disk: datafiles, control files, redo log files.
  *Persistent.* Survives shutdown. Also called the "physical database."
- **Instance** = the RAM (SGA) plus the background processes running right now.
  *Volatile.* Destroyed on shutdown. A single instance opens a single database
  (RAC is the exception: many instances, one database). Strictly, the instance is
  **SGA + background processes**; the **PGA** (and the newer **MGA**) belong to
  processes/sessions, not the instance. Oracle's own diagrams still draw PGA/MGA *inside*
  the instance box — read that as "all the server-side memory," not as a definition.
- **Session** = one logical user connection. The **user process** (SQL\*Plus,
  Python, JDBC) lives on the client; the **server process** on the Oracle host does
  the work on the session's behalf and owns a private PGA.

When someone says "Oracle is up," they mean the *instance* has the *database* in the
`OPEN` state. You can be `NOMOUNT` (only memory exists), `MOUNT` (control files
read), or `OPEN` (datafiles usable). Memory (SGA) exists from `NOMOUNT` onward.

**The big picture:**

```
  CLIENT (user process)     + INSTANCE (RAM + background processes) -------------+   DATABASE (disk)
  +------------------+      + SGA (shared by all processes) ---------------------+   +--------------+
  | sqlplus / Python |      | shared pool | large pool | java pool               |   |  Datafiles   |
  | JDBC / OCI       |      | streams pool | vector pool                         |   |  Redo logs   |
  +------------------+      | database buffer cache | redo log buffer            |   |  Control     |
           |                | fixed SGA                                          |   |   files      |
  connect  |                + PGA (private per server process) ------------------+   |  Parameter & |
  string   +--------------->| SQL work areas                                     |   |   password   |
           |                | session memory (UGA) | private SQL area            |   +--------------+
           |                + MGA (semi-shared, dynamic membership) -------------+
           |                | namespace 1 | namespace 2 | namespace 3 ...        |
           |                +----------------------------------------------------+
           |                | DBWn LGWR CKPT SMON PMON ARCn MMON MMNL            |
           |                +----------------------------------------------------+
```

**Running example.** `HR` runs `UPDATE employees SET salary = 5000 WHERE employee_id = 100;` then `COMMIT;`

1. The **server process** finds the `EMPLOYEES` block. If it is not in the
   **buffer cache**, it reads it from a **datafile** (a *physical read*) and caches it.
2. It changes the row *in RAM* in the buffer cache and records the change plus the
   undo in the **redo log buffer**.
3. On `COMMIT`, **LGWR** flushes the redo to the online **redo log files** — this is
   what makes the commit durable.
4. Later, **DBWn** writes the now-dirty block back to the datafile. The commit did
   **not** wait for this.

---

## Step 2 — SGA vs. PGA at a glance

| | SGA (System Global Area) | PGA (Program Global Area) |
|---|---|---|
| Scope | shared by **all** processes; one per instance | **private** to one server process |
| Lifetime | instance lifetime | session/process lifetime |
| Contains | buffer cache, shared pool, large pool, java pool, streams pool, vector pool, redo log buffer, fixed SGA | SQL work areas, private SQL area, session memory (UGA) |
| Handy analogy | the office filing cabinet everyone uses | your own desk drawer |
| Sizing parameter | `SGA_TARGET` / `SGA_MAX_SIZE` | `PGA_AGGREGATE_TARGET` / `PGA_AGGREGATE_LIMIT` |
| Example data | cached `EMPLOYEES` blocks, the parsed plan for the UPDATE | the in-progress sort of 1M rows, your `NLS_DATE_FORMAT` setting |
| Typical goal | cache the working set, parse each statement once | give each operation enough RAM to avoid spilling to temp |

`MEMORY_TARGET` (AMM, below) manages **both together**. A common one-liner:

```sql
ALTER SYSTEM SET sga_target = 1G SCOPE = BOTH;
ALTER SYSTEM SET pga_aggregate_target = 512M SCOPE = BOTH;
```

**The acronym you'll see in odd places: UGA.** The *User Global Area* is logically
part of session memory. With **dedicated** servers it physically lives in the PGA.
With **shared** servers it lives in the **large pool** (or shared pool). The
`oracle_architecture.png` diagram marks it in the legend, and shows it inside
*Session Memory*. This is why "shared server" changes where a session's memory is charged.

**A newer area you must not miss: MGA.** The *Managed Global Area* is **semi-shared**
memory. Unlike the SGA (shared by everyone) it is a *scoped* shared region: only a
chosen subset of trusted Oracle processes attach to it, and they attach/detach
**dynamically**. It exists to coordinate memory between cooperating processes —
parallel query, analytical operators, vector/ONNX inference and similar — without
imposing system-wide visibility. The `oracle_architecture.png` diagram shows the MGA
divided into **namespaces** (Namespace 1, 2, 3, …), each shareable between any number
of foreground/background processes. It is covered in [Step 6](#step-6--mga-managed-global-area-and-namespaces).

| | SGA | PGA | UGA | MGA |
|---|---|---|---|---|
| Sharing | all processes | one process | one session | a chosen subset of processes |
| Membership | static for the instance | fixed | fixed per session | **dynamic** attach/detach |
| Managed by | `SGA_TARGET` | `PGA_AGGREGATE_TARGET` | part of PGA/large pool | auto-managed; sized via Unified Memory |

**One more area, outside both: software code areas.** The Oracle executable's own
instructions live in a **software code area** — read-only, usually size-static, and
preferably installed **shared** so many processes (and instances) map the same pages
instead of each keeping a copy; the code area itself is **read-only and shareable**. It is
neither SGA nor PGA, but it is real memory the OS must find room for (see §C.5 on the
process image).

---

## Step 3 — How Oracle decides sizes: Manual, ASMM, AMM, Unified

This is the part you must understand *before* manipulating anything, because it
determines which `ALTER SYSTEM` commands do what.

### 3.1 Manual memory management
You size every structure by hand: `db_cache_size`, `shared_pool_size`,
`large_pool_size`, `java_pool_size`, … No automatic redistribution. Rare today,
used only for special cases.

### 3.2 Automatic Shared Memory Management (ASMM) — the usual default
You set **one SGA number** and Oracle figures out the split:

```sql
ALTER SYSTEM SET sga_max_size   = 4G  SCOPE = SPFILE;   -- static: needs restart
ALTER SYSTEM SET sga_target     = 2G  SCOPE = BOTH;      -- dynamic: live
ALTER SYSTEM SET pga_aggregate_target = 1G SCOPE = BOTH;
```

> These values assume a server with ample RAM. On **Free**, SGA + PGA must fit under
> the 2 GB cap — use the worked example in §3.8 instead.

- `SGA_MAX_SIZE` = the ceiling the SGA can ever grow to (static).
- `SGA_TARGET` = the current target Oracle will manage toward (dynamic). `0` disables ASMM.
- Oracle's **MMAN** background process moves memory between the *auto-tunable*
  components: buffer cache, shared pool, large pool, java pool, streams pool.
- **Not auto-tuned:** redo log buffer, fixed SGA, and the vector/memoptimize pools.
- If you explicitly set a component (e.g. `shared_pool_size=400M`) it stays, and
  only the rest are rebalanced.

### 3.3 Automatic Memory Management (AMM)
You set **one number for SGA + PGA combined**:

```sql
ALTER SYSTEM SET memory_max_target = 2G SCOPE = SPFILE;  -- static (fits the Free cap)
ALTER SYSTEM SET memory_target     = 1G SCOPE = BOTH;     -- dynamic
```

- Oracle auto-tunes both the SGA and the PGA up to `MEMORY_TARGET`.
- Requires OS support: on Linux it uses a large **`/dev/shm`** (tmpfs). Check with
  `df -h /dev/shm` — it must be at least as large as `MEMORY_MAX_TARGET`.
- When AMM is on, `SGA_TARGET` and `PGA_AGGREGATE_TARGET` become **minimums**, and
  `MEMORY_MAX_TARGET` is the ceiling.
- **Big caveat (Linux): AMM cannot use HugePages.** `MEMORY_TARGET`/`MEMORY_MAX_TARGET`
  are explicitly *incompatible* with huge pages, because AMM resizes the SGA at runtime
  and huge pages are pre-allocated. This is the single biggest reason large Linux systems
  run **ASMM (`SGA_TARGET`) + HugePages** instead of AMM. If you want huge pages, reset
  both `*_memory_target` parameters to `0`.

### 3.4 Unified Memory (Oracle AI Database 26ai)
The newest mode collapses **everything** — SGA, PGA, MGA, UGA and other segments —
under **one** parameter:

```sql
ALTER SYSTEM SET memory_size = 2G SCOPE = SPFILE;   -- 2 GB Free cap; raise on larger systems
-- when MEMORY_SIZE is set nonzero, MEMORY_TARGET / MEMORY_MAX_TARGET must be 0
```

- The database dynamically picks the ratio of SGA/PGA/MGA/UGA from the current
  workload and the sizing of the PDBs open in the CDB.
- If huge pages are configured, they can back both SGA and PGA.
- It avoids restarts to change the memory split, which is why it targets
  high-density consolidation. Verify availability in your release (`MEMORY_SIZE`
  starts at 26ai).

### 3.5 Which columns/parameters are static vs. dynamic

| Parameter | Static (needs restart) | Dynamic (`SCOPE=BOTH` works live) |
|---|---|---|
| `sga_max_size` | ✔ | |
| `memory_max_target` | ✔ | |
| `memory_size` | ✔ (SPFILE) | |
| `log_buffer` | ✔ | |
| `vector_memory_size` | ✔ | |
| `memoptimize_pool_size` | ✔ | |
| `inmemory_size` | ✔ | |
| `sga_target` | | ✔ |
| `memory_target` | | ✔ |
| `pga_aggregate_target` | | ✔ |
| `pga_aggregate_limit` | | ✔ |
| `db_cache_size`, `shared_pool_size`, `large_pool_size`, `java_pool_size`, `streams_pool_size` | | ✔ |
| `db_keep_cache_size`, `db_recycle_cache_size`, `db_nk_cache_size` | | ✔ |

### 3.6 `SCOPE` explained
- `SCOPE=MEMORY` → change now, lost at restart.
- `SCOPE=SPFILE` → write to the SPFILE, takes effect after restart (required for static params).
- `SCOPE=BOTH` → both (the usual choice for dynamic params).

### 3.7 Multitenant note (23ai Free is a CDB)
Free ships as a container database with a pluggable database (default `FREEPDB1`).
Per-PDB memory parameters — `SGA_TARGET`, `SGA_MIN_SIZE`, `DB_CACHE_SIZE`,
`SHARED_POOL_SIZE`, `PGA_AGGREGATE_TARGET`, `PGA_AGGREGATE_LIMIT` — can be set inside
the PDB, bounded by the CDB's values. Check which container you're in:

```sql
SELECT SYS_CONTEXT('USERENV','CON_NAME') FROM dual;
```

- **PDB memory isolation:** CDB-level parameters set the instance total; each PDB can set
  its own `SGA_TARGET`, **`SGA_MIN_SIZE`** (a guaranteed floor so a noisy neighbor cannot
  starve it), `DB_CACHE_SIZE`, `SHARED_POOL_SIZE`, `PGA_AGGREGATE_TARGET`, and
  `PGA_AGGREGATE_LIMIT`, all bounded by the CDB.
- **RAC (if applicable):** every instance has its own SGA/PGA. The shared pool holds
  **Global Resource Directory (GRD)** structures and **Cache Fusion** buffers, so per-node
  SGA is larger. Use `GV$` views (`GV$SGASTAT`, `GV$PGASTAT`) and the `INST_ID` column;
  interconnect health shows up as memory waits through Cache Fusion.

### 3.8 Sizing methodology (and a worked 2 GB Free example)

There is no magic number — size from the workload and the box.

1. **Budget from physical RAM.** Leave room for the OS (and page cache, unless using
   direct I/O). A common starting split on a dedicated DB host is **SGA ≈ 50–70%** of RAM;
   size **PGA** by concurrency, not by RAM.
2. **`PROCESSES` drives PGA.** `pga_aggregate_target ≈ (active sessions) × (avg work-area
   need)`. OLTP (many tiny sorts/hashes) needs *less per session* than DW (few huge ones).
3. **OLTP vs DW:** OLTP is buffer-cache-dominant with small PGA and high parse reuse; DW
   needs large work areas/PGA, `large_pool` for parallel query, and benefits from
   columnar **In-Memory** and direct-path reads.
4. **Pick a mode:** ASMM (`SGA_TARGET`) for most Linux systems (works with HugePages);
   AMM only for convenience when huge pages aren't needed; Unified Memory (`memory_size`)
   on 26ai for consolidation.
5. **Measure, then adjust** (§7.1): `V$DB_CACHE_ADVICE`, `V$SGA_TARGET_ADVICE`,
   `V$PGA_TARGET_ADVICE`, `V$SHARED_POOL_ADVICE`. Change one thing, watch the waits.

**Worked example — Oracle Database Free (2 GB cap):**

| Component | Target | Rationale |
|---|---|---|
| SGA total | ~1.1 GB | leave headroom for PGA + OS |
| `SGA_TARGET` | `1100M` | ASMM manages buffer cache/shared/… pools |
| `DB_CACHE_SIZE` (auto slice) | most of the SGA | OLTP read path |
| `PGA_AGGREGATE_TARGET` | `512M` | modest concurrency on 2 cores |
| `PGA_AGGREGATE_LIMIT` | default (≈2 GB) | do **not** set below the default |
| `log_buffer` | default (a few MB) | not a bottleneck at this scale |
| large pool | only if shared server/parallel | otherwise leave `0` |

```sql
-- SGA_TARGET / PGA_AGGREGATE_TARGET are INSTANCE-wide: set them in the root.
ALTER SESSION SET CONTAINER = CDB$ROOT;
ALTER SYSTEM SET sga_target            = 1100M SCOPE = BOTH;   -- instance SGA
ALTER SYSTEM SET pga_aggregate_target  = 512M  SCOPE = BOTH;   -- instance PGA
ALTER SESSION SET CONTAINER = FREEPDB1;
-- total ≈ 1.6 GB, safely under the 2 GB Free ceiling
```

> **Root vs. PDB (important).** `SGA_TARGET` and `PGA_AGGREGATE_TARGET` set in `CDB$ROOT`
> are the **instance totals**. The same-named parameters set while connected to a PDB are
> **per-PDB caps** — they bound how much of the instance SGA/PGA *that* PDB may consume and
> **do not resize the instance**. To guarantee a PDB a floor, use `SGA_MIN_SIZE`. The CDB's
> default PDB is `FREEPDB1`; verify where you are with
> `SELECT SYS_CONTEXT('USERENV','CON_NAME') FROM dual;`.

### 3.9 Where those parameters live: CDB, PDB, Resource Manager, RAC

- **One instance = one SGA/PGA.** In a multitenant database (23ai Free is a CDB) there is still
  a *single* instance SGA and PGA shared by every PDB. Set `SGA_TARGET` /
  `PGA_AGGREGATE_TARGET` while connected to **`CDB$ROOT`**; that is the instance total.
- **Per-PDB caps.** The same-named parameters set *inside* a PDB bound only that PDB's slice and
  never resize the instance: `SGA_TARGET` (cap), `SGA_MIN_SIZE` (floor),
  `PGA_AGGREGATE_TARGET`, `PGA_AGGREGATE_LIMIT`, `DB_CACHE_SIZE`. This is how you stop one PDB
  from starving another:
  ```sql
  ALTER SESSION SET CONTAINER = FREEPDB1;
  ALTER SYSTEM SET sga_target           = 1200M SCOPE = BOTH;  -- cap for this PDB only
  ALTER SYSTEM SET sga_min_size         = 256M  SCOPE = BOTH;  -- guaranteed floor
  ALTER SYSTEM SET pga_aggregate_limit  = 1G    SCOPE = BOTH;  -- hard cap for this PDB
  ```
- **Resource Manager.** `DBMS_RESOURCE_MANAGER` plans distribute CPU, I/O, parallel servers,
  sessions and undo across consumer groups and PDBs; memory itself is bounded by the per-PDB
  parameters above rather than by a "memory share". Monitor per-PDB memory with
  `V$RSRCPDBMETRIC` (26ai) and per-group activity with `V$RSRC_CONSUMER_GROUP`.
- **RAC.** In a Real Application Clusters database **each instance has its own SGA and PGA**.
  Most memory settings are database-wide, but many can be scoped to one instance (or all) with
  the `SID` clause:
  ```sql
  ALTER SYSTEM SET db_cache_size = 2G SCOPE = BOTH SID = 'RAC1';  -- one instance
  ALTER SYSTEM SET db_cache_size = 2G SCOPE = BOTH SID = '*';     -- all instances
  ```
  A block read on one instance can be served from another instance's buffer cache (global
  cache / GCS), so the *effective* cache is the sum across instances. Database Free is
  single-instance, so RAC does not apply to the labs here.

---

## Step 4 — SGA components, one by one

**A running example.** Keep one statement in mind throughout this Step —
`UPDATE hr.employees SET salary = 9000 WHERE employee_id = 100`, then `COMMIT`:

1. The block holding row 100 is sought in the **database buffer cache** (§4.2). A miss is
   a **physical read**, and the block lands on an LRU chain.
2. The change is applied to the cached block; a **redo** record goes to the **redo log
   buffer** (§4.3) and **undo** is written to an undo segment **also cached** in the
   buffer cache.
3. `COMMIT` makes **LGWR** flush the redo to disk (durable). **DBWn** writes the dirty
   block later. None of this needed the PGA yet.
4. A concurrent reader needing the *old* salary builds a **consistent-read (CR) block**
   from the undo — logged as `consistent gets`.
5. The change is now visible to all sessions through the **shared** buffer cache, while
   each session's private state lives in the **PGA** (§5) and cooperating
   producers/consumers share the **MGA** (§6).

One statement touches four components; the rest of this Step explains each.

### 4.1 Fixed SGA
- **What / why:** Oracle's internal bookkeeping. Holds pointers to the other SGA
  components, plus process/session state arrays and redo buffer pointers. Exists
  from instance start.
- **Example data:** the address of the buffer-cache hash table, the kernel state.
- **Inspect:**
  ```sql
  SELECT name, ROUND(bytes/1024/1024,1) AS mb FROM v$sgainfo WHERE name = 'Fixed SGA Size';
  SELECT name, value FROM v$sga WHERE name = 'Fixed Size';   -- V$SGA names it differently
  ```
- **Manipulate:** *Nothing directly.* Oracle sizes it at startup. It grows
  **indirectly** with parameters like `processes` and `sessions`.

### 4.2 Database buffer cache
- **What / why:** caches data blocks so reads hit RAM, not disk. Dirty blocks wait
  here for DBWn. Usually the **largest** SGA component.
- **Example data:** one 8 KB block holding `EMPLOYEES` rows 100–130. First read =
  physical read; later reads = cache hits.
- **Sub-pools:** `DEFAULT`, `KEEP`, `RECYCLE`, plus a cache per non-standard block
  size (`db_2k_cache_size` … `db_32k_cache_size`).
- **Inspect:**
  ```sql
  SELECT name, block_size, current_size AS mb, buffers FROM v$buffer_pool;
  SELECT name, value FROM v$sysstat
  WHERE name IN ('db block gets','consistent gets','physical reads','db block gets from cache');
  ```
- **Manipulate:**
  ```sql
  ALTER SYSTEM SET db_cache_size = 2G SCOPE = BOTH;
  ALTER SYSTEM SET db_keep_cache_size = 128M SCOPE = BOTH;   -- then pin hot objects:
  ALTER TABLE hr.employees STORAGE (BUFFER_POOL KEEP);
  ALTER INDEX hr.emp_name_ix STORAGE (BUFFER_POOL KEEP);
  ALTER SYSTEM FLUSH BUFFER_CACHE;   -- testing only; causes cold reads
  ```
- **Tune it with advice** (needs the advice view to have been on):
  ```sql
  -- Compute the ratio for context, NOT as a tuning target:
  SELECT 1 - (SELECT value FROM v$sysstat WHERE name='physical reads')
           / NULLIF((SELECT value FROM v$sysstat WHERE name='db block gets')
                  + (SELECT value FROM v$sysstat WHERE name='consistent gets'), 0)
    AS cache_hit_ratio FROM dual;
  SELECT size_for_estimate, estd_physical_read_factor FROM v$db_cache_advice;
  ```
- **Watch out:** `V$BUFFER_POOL.CURRENT_SIZE` is already in **MB** (not blocks).
- **Cache hit ratio is a diagnostic, not a goal.** "Aim for 99%" is a classic
  anti-pattern: the ratio *rises* when you do less work (a full scan touches each block
  once), so a bad plan can actually inflate it. Diagnose with **wait events**
  (`db file sequential read`, `db file scattered read`, `free buffer waits`,
  `cache buffers chains`) and `V$DB_CACHE_ADVICE`, and fix the *statements/objects* that
  miss — not the aggregate percentage.
- **How the cache actually works:**
  - **Hash chains:** a block address is hashed to a bucket; the `cache buffers chains`
    latch protects that bucket. Hot blocks → `latch: cache buffers chains` contention.
  - **LRU with a touch count:** the list has a **hot** end and a **cold** end. A block's
    touch count increments only if it was last incremented more than **3 seconds** ago (so
    a burst of pins counts once); a hot block stays, a cold one ages out. A **long full
    table scan** places its blocks at the **cold (tail) end**, so they age out quickly
    instead of flushing the hot set; a **short** scan of a table below the small-table
    threshold goes to the **hot (MRU) end**; and a **very large** scan may bypass the cache
    entirely with **direct-path reads** (11g+). See the
    [26ai Tuning Guide — Tuning the Database Buffer Cache](https://docs.oracle.com/en/database/oracle/oracle-database/26/tgdba/tuning-database-buffer-cache.html)
    ("Database blocks accessed during a long full table scan are placed on the tail end of
    the Least Recently Used (LRU) list").
  - **Buffer states:** each buffer is **Unused**, **Clean** (read-consistent, reusable), or
    **Dirty** (modified; must be written before reuse), and is either **pinned** (in use)
    or **free**.
  - **DBWn triggers:** no clean buffer available (**free buffer wait**), advancing the
    checkpoint, the dirty-list threshold, a `fast_start_mttr_target` checkpoint to bound
    recovery time, or a tablespace becoming read-only/offline.
  - **Checkpoints** advance the recovery **RBA** from which crash recovery replays redo;
    `fast_start_mttr_target` trades steady checkpoint I/O for shorter recovery.
  - **Undo & read consistency:** every DML writes **undo** (in undo segments, cached in
    this same buffer cache). Readers needing an older image build a **consistent-read
    (CR) block** by applying undo — why `consistent gets` is the normal read path and
    `db block gets` are current-mode reads. Flashback/MVCC all rest on this.
  - **Direct-path reads bypass the cache:** full scans, `PARALLEL`, and
    `/*+ APPEND */` loads read into the **PGA/temp** without polluting the cache —
    linking the SGA to the PGA/TEMP story (§5).

### 4.3 Redo log buffer
- **What / why:** a circular buffer of *change records* (redo entries) that LGWR
  writes to the online redo logs. This is what makes commits durable and enables
  instance recovery.
- **Example data:** "block 42, row 7, salary changed 4000 → 5000" plus the matching
  undo change.
- **LGWR flushes** when: a transaction **commits**; every **3 seconds**; when the buffer
  is **1/3 full**; when it accumulates roughly **1 MB** of redo; or before DBWn writes
  dirty blocks (so redo is safely on disk before the data it describes).
- **Inspect:**
  ```sql
  SELECT name, value FROM v$sysstat
  WHERE name IN ('redo size','redo entries','redo buffer allocation retries','redo writes');
  ```
- **Manipulate:**
  ```sql
  ALTER SYSTEM SET log_buffer = 64M SCOPE = SPFILE;   -- STATIC: restart required
  ```
  - You rarely size the buffer directly. Reduce **redo volume** with `NOLOGGING` /
    direct-path loads; reduce **`log file sync` waits** (not redo volume) by **batching
    commits** — e.g. one `COMMIT` per 1,000 rows instead of one per row.
  - `COMMIT WRITE BATCH NOWAIT` trades durability for speed.
  - Size the *redo log files* (not the buffer) for your switch frequency.
- **Watch out:** the wait event `log buffer space` means LGWR can't keep up; the
  wait `log file sync` means commits are waiting on LGWR. Repeated `redo buffer
  allocation retries` means the buffer is too small for your redo rate.

### 4.4 Shared pool
- **What / why:** caches expensive, *shared* computations so SQL is parsed once and
  reused by everyone. **Second-largest** SGA component.
- **Components** (these are exactly the boxes drawn inside *Shared Pool* in
  `oracle_architecture.png`):
  - **Library cache** — the parsed, shared representation of SQL and PL/SQL:
    - **Shared SQL area** — the statement text (e.g. `SELECT * FROM employees`) plus
      its execution plan, shared by every session that runs the same SQL.
    - **Private SQL area (shared server only)** — in shared-server mode the
      per-cursor runtime area is also kept here (in dedicated mode it lives in the PGA).
  - **Data dictionary (row) cache** — metadata while parsing: columns of `EMPLOYEES`,
    grants to `HR`, object definitions.
  - **Server result cache** — whole query-result sets and PL/SQL function results.
    E.g. `SELECT /*+ RESULT_CACHE */ COUNT(*) FROM employees` → the cached `107`.
  - **Other** — miscellaneous allocations that don't fit the named caches
    (control structures, locks, latches, fixed allocation callbacks, etc.).
  - **Reserved pool** — a reserved slice of the shared pool for large *contiguous*
    chunks, so big allocations don't fail on a fragmented pool
    (`shared_pool_reserved_size`, monitored in `V$SHARED_POOL_RESERVED`).
- **Inspect:**
  ```sql
  SELECT pool, name, ROUND(bytes/1024/1024,1) AS mb
  FROM v$sgastat WHERE pool = 'shared pool' ORDER BY bytes DESC FETCH FIRST 20 ROWS ONLY;
  SELECT name, value FROM v$sysstat
  WHERE name IN ('parse count (total)','parse count (hard)','parse time elapsed');
  ```
- **Manipulate:**
  ```sql
  ALTER SYSTEM SET shared_pool_size = 1G SCOPE = BOTH;   -- dynamic
  EXEC DBMS_SHARED_POOL.KEEP('SYS.STANDARD');            -- pin high-value objects
  ALTER SYSTEM FLUSH SHARED_POOL;                        -- testing only
  ```
  - **Use bind variables** (`:id`) instead of literals so plans are reused and
    hard parses stay low. `cursor_sharing=FORCE` is a blunt fallback.
  - Result cache: `result_cache_mode` (`MANUAL`/`FORCE`) and `result_cache_max_size`,
    plus the `/*+ RESULT_CACHE */` hint. Purge with `DBMS_RESULT_CACHE.FLUSH`.
- **Watch out:** a too-small shared pool → high hard-parse rate and `library cache`
  latch waits. A too-large one wastes RAM that the buffer cache could use.
- **How the library cache actually works:**
  - **Soft vs hard parse:** a **soft parse** finds a matching shared cursor and reuses its
    plan; a **hard parse** builds a new one (parse + optimize + row-source generation).
    Track `parse count (hard)` vs `parse count (total)`.
  - **Parent vs child cursors:** one parent (SQL text signature) may have many **child
    cursors** (different binds, optimizer environment, or privileges). Too many children →
    memory pressure and `_cursor_obsolete_threshold` behavior; diagnose with `V$SQL` and
    `V$SQL_SHARED_CURSOR`.
  - **Bind peeking & adaptive cursor sharing (ACS):** Oracle peeks at bind values on the
    first parse; with skewed data this can yield a plan good for one value, bad for others,
    so ACS builds **multiple child cursors** for different bind ranges.
  - **Cursor invalidation:** DDL, stats changes, or grants invalidate dependent cursors and
    force a re-parse — visible as `library cache lock/pin` waits and in
    `V$SQL_SHARED_CURSOR`.
  - **Latches vs mutexes:** since 10g, library-cache objects use fine-grained **mutexes**
    (`cursor: mutex S`, `library cache: mutex X`); contention appears as those waits, not a
    single "library cache latch".
  - **Pin high-value objects** with `DBMS_SHARED_POOL.KEEP`; keep SQL **textually
    identical** (case, whitespace, literals) or it will not match.

### 4.5 Large pool
- **What / why:** an **optional** pool for big, transient allocations so they don't
  fragment the shared pool.
- **Components** (the `oracle_architecture.png` diagram labels *Response Queue* and
  *Request Queue* inside the large pool; these are the shared-server queues):
  - Shared-server session memory (the UGA). Example: variables for 200 shared-server connections.
  - **Request Queue / Response Queue** — dispatchers put client requests on the
    request queue for a shared server to pick up; the response queue carries results
    back to the dispatcher. Backed by the large pool.
  - Parallel-execution message buffers (for `SELECT /*+ PARALLEL(4) */`).
  - **I/O buffer area** — buffers used by RMAN backup/restore, DBWn, recovery and
    I/O server processes (see the legend item below).
- **Inspect:**
  ```sql
  SELECT pool, name, ROUND(bytes/1024/1024,1) AS mb
  FROM v$sgastat WHERE pool = 'large pool' ORDER BY bytes DESC;
  ```
- **Manipulate:**
  ```sql
  ALTER SYSTEM SET large_pool_size = 512M SCOPE = BOTH;   -- dynamic
  ALTER SYSTEM SET shared_servers = 20 SCOPE = BOTH;      -- dispatchers too
  ```
  - Also influenced by RMAN channel count and `parallel_max_servers`.
- **Watch out:** if you use **shared server or parallel query**, size the large pool
  explicitly. Without it, that memory is taken from the shared pool and can cause
  `ORA-04031` (unable to allocate shared memory).

### 4.6 Java pool
- **What / why:** memory for the Oracle JVM: session-specific Java code and data.
- **Example data:** the loaded class `com.acme.Hello` and the objects created by a
  Java stored procedure during a call.
- **Inspect:** appears as pool `java pool` in `v$sgastat`.
- **Manipulate:**
  ```sql
  ALTER SYSTEM SET java_pool_size = 256M SCOPE = BOTH;    -- dynamic
  ```
  - Only matters if you use Java stored procedures. Related limits:
    `java_soft_sessionspace_limit`, `java_max_sessionspace_size`; load code with `loadjava`.

### 4.7 Streams pool
- **What / why:** buffered queues and replication. Used by Advanced Queuing
  buffered messages, GoldenGate integrated capture/apply, XStream, and Data Pump
  (which uses AQ). Oracle Streams itself is deprecated/desupported in modern
  releases, but the pool still serves these features.
- **Example data:** logical change records (LCRs) from `employees` waiting to be
  applied on a replica.
- **Inspect:** pool `streams pool` in `v$sgastat`.
- **Manipulate:**
  ```sql
  ALTER SYSTEM SET streams_pool_size = 256M SCOPE = BOTH;
  ```
  - If `0`, Oracle grows it on demand (borrowing from the buffer cache) when an AQ
    or Data Pump job needs it. Setting it explicitly guarantees a minimum.
- **Watch out:** Data Pump users are often surprised that the streams pool grows
  during exports/imports.

### 4.8 Vector pool (23ai)

- **What / why:** a dedicated **SGA** pool backing Oracle's vector indexes and vector
  operations. **HNSW** (Hierarchical Navigable Small World) indexes live *in the vector
  pool*; **IVF** (inverted-file) vector indexes do **not** use it, so don't expect this
  pool to matter for IVF workloads.
- **Example data:** an HNSW graph for a 1M-row `VECTOR(768, FLOAT32)` embedding column —
  the vectors and graph edges sit in RAM for fast nearest-neighbor search.
- **Key facts:**
  - It is **carved out of the SGA**, so `SGA_TARGET` must be **larger** than the pool or
    it cannot be allocated.
  - The default `vector_memory_size` is **0**: until you set it, HNSW index creation
    fails. Set it *before* building the index.
  - It is **static** (restart) and is **not** an auto-tuned ASMM/AMM component — Oracle
    will not grow it on demand.
- **Inspect:**
  ```sql
  SHOW PARAMETER vector_memory_size
  SELECT * FROM v$vector_memory_pool;
  ```
- **Manipulate** (the parameter is **static and instance-wide**, so set it in the root):
  ```sql
  -- vector_memory_size is static and CDB-wide: run in the root, then restart
  ALTER SESSION SET CONTAINER = CDB$ROOT;
  ALTER SYSTEM SET vector_memory_size = 512M SCOPE = SPFILE;   -- STATIC: restart required
  -- after restart, a self-contained runnable example in the PDB:
  ALTER SESSION SET CONTAINER = FREEPDB1;
  CREATE TABLE hr.docs (id NUMBER, embedding VECTOR(3, FLOAT32));
  INSERT INTO hr.docs VALUES (1, '[1, 2, 3]');
  COMMIT;
  CREATE VECTOR INDEX hr_docs_hnsw ON hr.docs (embedding)
    ORGANIZATION INMEMORY NEIGHBOR GRAPH
    DISTANCE COSINE WITH TARGET ACCURACY 90;
  ```
- **Watch out:** on Oracle Free (2 GB) a large vector pool competes directly with the
  buffer cache and shared pool — leave headroom.

### 4.9 Memoptimize pool (bonus, 18c+)
- **What / why:** a separate SGA area for high-speed, key-based lookups on heap-organized
  tables declared `MEMOPTIMIZE FOR READ` (e.g. `SELECT * FROM cust WHERE cid = 10`).
  Clients can pull buffers **directly from the SGA over the network**, avoiding CPU/OS
  overhead. (Fast **ingest** — `MEMOPTIMIZE FOR WRITE` — is a separate feature; don't
  assume this pool backs it.)
- **Two parts:** the **memoptimize buffer area** (≈75% of the pool) permanently locks
  buffers for those tables until the table is `NO MEMOPTIMIZE FOR READ`, and a **hash
  index** (~25%) maps primary keys to those buffers. These buffers are **separate from the
  database buffer cache** and do **not** count toward its size.
- **Manipulate:** `memoptimize_pool_size` is **static** (restart) and disabled by default.
  Its value **does count toward `SGA_TARGET`**, but Oracle does **not** grow/shrink it
  automatically. Only relevant if you enable memoptimized tables.

### 4.10 Other optional SGA subareas
The docs group these as *Optional Performance-Related SGA Subareas*; they are SGA
components all the same:

- **In-Memory Area** — the **In-Memory Column Store (IM column store)** for `INMEMORY`
  tables/partitions/materialized views, stored in columnar format. Requires the In-Memory
  option. Sized by the **static** `inmemory_size` parameter; monitor with `V$INMEMORY_AREA`.
- **Memoptimize Pool** — see §4.9.
- **Shared IO Pool** — a pool of I/O buffers shared by the instance (appears in
  `V$SGAINFO` as *Shared IO Pool Size*, and in the large pool's I/O buffer area).

### 4.11 Legend: free memory and I/O buffer areas
The `oracle_architecture.png` legend has two swatches that describe *state*, not
separate structures:

- **Free memory** — unused space inside a pool (e.g. `v$sgastat` rows named
  `free memory`). Oracle can hand it out on demand; zero free memory with high waits
  (`ORA-04031`) means the pool is too small.
- **I/O buffer area** — buffers a process borrows to stage disk I/O (DBWn writes,
  RMAN, recovery). Look for `io buffer` / I/O entries in `V$SGASTAT` and the large pool.

### 4.12 Granules (why you can't set arbitrary sizes)
SGA memory is allocated in **granules**, so sizes round to the nearest granule:

| `SGA_MAX_SIZE` | Granule |
|---|---|
| < 1 GB | 4 MB |
| 1–8 GB | 16 MB |
| 8–16 GB | 32 MB |
| 16–32 GB | 64 MB |
| 32–64 GB | 128 MB |
| 64–128 GB | 256 MB |
| > 128 GB | 512 MB |

> The **redo log buffer** and the **fixed SGA** are the exceptions: neither divides memory
> into granules (the redo buffer is one contiguous `LOG_BUFFER`, the fixed SGA is internal
> bookkeeping). Everything else in the SGA rounds to the granule size. Oracle's own wording:
> *"All SGA components except the redo log buffer allocate and deallocate space in units of
> contiguous memory called granules"* and *"the redo log buffer and fixed SGA buffer do not
> divide memory into granules"* — see
> [26ai Concepts, Memory Architecture](https://docs.oracle.com/en/database/oracle/oracle-database/26/cncpt/memory-architecture.html).

See the exact value per component in `V$SGA_DYNAMIC_COMPONENTS.GRANULE_SIZE` (the non-granule
items show as `RESIZEABLE = No` in `V$SGAINFO`).

---

## Step 5 — PGA components, one by one

The PGA is **private** memory owned by a server process. It has three logical parts.

### 5.1 SQL work areas
- **What / why:** dedicated memory for memory-hungry operators — **sorts**
  (`ORDER BY`, `GROUP BY`, `DISTINCT`), **hash joins**, and **bitmap merges**. If the
  data fits → fast in-memory. If not, it spills to **temp** and runs one-pass or
  multi-pass (much slower).
- **Example data:** the in-progress sort of 1M `EMPLOYEES` rows by `salary`.
- **Inspect (live):**
  ```sql
  SELECT sid, operation_type, ROUND(actual_mem_used/1024/1024,1) AS mb_used,
         number_passes, tempseg_size
  FROM v$sql_workarea_active;
  -- histograms show how many operations are optimal/one-pass/multi-pass
  SELECT optimal_executions, onepass_executions, multipasses_executions
  FROM v$sql_workarea_histogram;
  ```
- **Manipulate:**
  ```sql
  SHOW PARAMETER workarea_size_policy;               -- AUTO is default
  ALTER SYSTEM SET pga_aggregate_target = 1G SCOPE = BOTH;
  -- MANUAL mode (rarely needed), per session:
  ALTER SESSION SET workarea_size_policy = MANUAL;
  ALTER SESSION SET sort_area_size = 50M;
  ALTER SESSION SET hash_area_size = 100M;
  ```
  - **Best fix is often not memory:** add an index, filter earlier, or avoid
    unnecessary sorts so there is less to sort.
  - **`cache hit percentage` is a diagnostic, not a target.** A sustained value near 100%
    means work areas are running `OPTIMAL`; a drop below it tells you some operations went
    one-pass or multi-pass and spilled to temp. Read it with `V$SQL_WORKAREA_HISTOGRAM` and
    `V$TEMPSEG_USAGE`, and fix the *operation* (give it memory **or** give it less work) —
    do not tune the percentage itself.
    ```sql
    SELECT name, value FROM v$pgastat
    WHERE name IN ('cache hit percentage','over allocation count');
    ```

### 5.2 Private SQL area
- **What / why:** the per-session, per-cursor part of a statement. The *shared* part
  (SQL text + plan) is in the shared pool; the *private* part is here:
  - **Persistent area** — bind values. Freed when the cursor closes.
  - **Runtime area** — execution state, e.g. how far a full scan has progressed.
- **Example data:** `SELECT * FROM employees WHERE dept_id = :d` with `:d = 10`,
  25 of 100 rows fetched so far.
- **Inspect:**
  ```sql
  SELECT sid, cursor_type, sql_text FROM v$open_cursor
  WHERE sid = SYS_CONTEXT('USERENV','SID');
  ```
- **Manipulate:**
  ```sql
  ALTER SYSTEM SET open_cursors = 500 SCOPE = BOTH;
  ALTER SYSTEM SET session_cached_cursors = 50 SCOPE = BOTH;
  ALTER SESSION SET cursor_sharing = EXACT;   -- or FORCE (use bind vars!)
  ```
  - **Always close cursors in your app.** Leaked cursors cause `ORA-01000`
    (maximum open cursors exceeded). Use bind variables and a sensible fetch array size.

### 5.3 Session memory (UGA)
- **What / why:** state of one session — logon info, `ALTER SESSION` settings,
  package variables, application contexts.
- **Location:** in the **PGA** for dedicated servers; in the **large pool** (or
  shared pool) for shared servers.
- **Example data:** SID 123, user `HR`, `NLS_DATE_FORMAT='YYYY-MM-DD'`,
  package variable `pkg.g_counter = 5`.
- **Inspect:**
  ```sql
  SELECT s.sid, s.username, n.name, ROUND(st.value/1024) AS kb
  FROM v$sesstat st
  JOIN v$statname n ON n.statistic# = st.statistic#
  JOIN v$session s   ON s.sid = st.sid
  WHERE n.name IN ('session uga memory','session pga memory')
    AND s.username IS NOT NULL
  ORDER BY kb DESC;
  ```
- **Manipulate:** `ALTER SESSION SET ...`; `DBMS_SESSION.SET_CONTEXT`; keep package
  collections small; the profile limit `PRIVATE_SGA` (for shared-server sessions).

### 5.4 PGA as a whole — inspect and manipulate

```sql
-- How much PGA is being used vs. targeted?
SELECT name, ROUND(value/1024/1024) AS mb
FROM v$pgastat
WHERE name IN ('aggregate PGA target parameter','aggregate PGA auto target',
               'total PGA allocated','maximum PGA allocated',
               'total PGA used for auto workareas','over allocation count',
               'cache hit percentage');

-- Per-process breakdown (PGA categories). V$PROCESS_MEMORY has NO name column:
-- use V$PROCESS.program, and join on PID (not PADDR).
SELECT p.spid, p.program, pm.category,
       ROUND(pm.allocated/1024/1024,1) AS alloc_mb,
       ROUND(pm.used/1024/1024,1)      AS used_mb
FROM   v$process_memory pm JOIN v$process p ON p.pid = pm.pid
ORDER BY pm.allocated DESC;
```

The `CATEGORY` values are Oracle's own buckets (they exist in every recent release):
`SQL`, `PL/SQL`, `Freeable`, `Other`:

- **`SQL`** — SQL work areas (§5.1) plus the private SQL area of executing statements.
  Grows with sorts/hashes and large hash joins; watch it during a bad plan.
- **`PL/SQL`** — PL/SQL runtime state: package variables, collections, `REF CURSOR`s. This
  is where a `BULK COLLECT ... LIMIT` that is set too high (or a runaway nested-table
  collection) blows up a single session's PGA. Cap it with `LIMIT` and bulk binds.
- **`Freeable`** — memory Oracle has allocated but can release back to the OS (typically
  freed work areas not yet returned). Temporary spikes here are normal.
- **`Other`** — session/UGA overhead, logon state and **untunable** allocations. When
  `PGA_AGGREGATE_LIMIT` is hit, Oracle aborts/terminates the sessions with the most
  **untunable** (`Other`) PGA first — so a growing `Other` is the dangerous one.

```sql
ALTER SYSTEM SET pga_aggregate_target = 1G SCOPE = BOTH;   -- soft target
ALTER SYSTEM SET pga_aggregate_limit  = 4G SCOPE = BOTH;   -- hard ceiling
```

- `PGA_AGGREGATE_TARGET` is a **soft** goal; Oracle may exceed it.
- `PGA_AGGREGATE_LIMIT` (12c+) is a **hard** limit. Its default is the greater of
  **2 GB**, **200% of PGA_AGGREGATE_TARGET**, and **3 MB × PROCESSES**. It may be set
  to `0` to disable, but must not be set **below** the default or startup fails. When
  exceeded, Oracle aborts, then terminates, the sessions using the most *untunable* PGA.
- **`over allocation count` rising** = `PGA_AGGREGATE_TARGET` is too small to even
  run your workload's minimum — raise it (or reduce concurrency).

### 5.5 PGA vs. DGA vs. MGA — and what the hard limit does

Three areas are easy to conflate. They overlap in accounting but not in nature:

| Area | Scope | Backs | Where you see it |
|---|---|---|---|
| **PGA** | private, per server process | sort/hash/bitmap work areas, session UGA, cursor state | `V$PGASTAT`, `V$PROCESS_MEMORY` |
| **DGA** (Direct Global Area) | private, *part of the PGA* | direct (non-buffered) I/O buffers: **parallel query** message buffers, PX hash/bitmap build areas | inside `total PGA used for auto workareas`; `V$PROCESS_MEMORY_DETAIL` |
| **MGA** | semi-shared, on `/dev/shm` (26ai) | AI/vector/analytical structures chosen for dynamic sharing | files under `/dev/shm`; see §6 |

- DGA is **not** separately tunable; it grows with parallel execution. Kill the runaway PX query
  and lower `PARALLEL_*` for that object.
- MGA is **counted under PGA allocation** (§6.3) even though its pages live on tmpfs — so it
  appears in `V$PGASTAT` PGA totals even though it is not anonymous process memory.

**The hard limit, in order.** `PGA_AGGREGATE_TARGET` is soft; the real ceiling is
`PGA_AGGREGATE_LIMIT` (default = greatest(**2 GB**, **200% of target**, **3 MB × `PROCESSES`**)).
As PGA approaches it, Oracle first pushes work areas to one-pass/multi-pass; if it is still
exceeded it **aborts, then terminates the sessions with the most untunable (`Other`) PGA** — a
`V$PROCESS_MEMORY` category worth watching (§5.4), not the sort users. Per-process detail:

```sql
SELECT p.spid, p.program, pmd.category, pmd.name, pmd.heap_name,
       ROUND(pmd.bytes/1024/1024,1) mb, pmd.allocation_count
FROM   v$process_memory_detail pmd JOIN v$process p ON p.pid = pmd.pid
ORDER BY pmd.bytes DESC FETCH FIRST 20 ROWS ONLY;
```

Set the limit only **at or above** the default (a lower value is rejected); to relieve pressure,
raise it, cap the offending PDB (`ALTER SYSTEM SET pga_aggregate_limit` inside the PDB), or fix
the SQL/session (bounded `BULK COLLECT`, lower DOP, avoid pathological hashes).

---

## Step 6 — MGA (Managed Global Area) and namespaces

The `oracle_architecture.png` diagram shows an area next to the PGA labeled **MGA**,
subdivided into **Namespace 1, Namespace 2, Namespace 3**, with the note:

> *"Namespaces can be shared between any arbitrary number of Oracle Process (both FG and BG)."*

### 6.1 What the MGA is
- **Managed Global Area** — a **semi-shared** memory framework, new in Oracle AI
  Database. It is neither fully shared like the SGA nor fully private like the PGA.
- **Scoped sharing:** modules choose exactly *which* trusted processes attach. Unlike
  the SGA, membership is **dynamic** — processes attach and detach at runtime.
- **Why it exists:** some workloads (parallel query, analytical hash joins, vector
  index builds, ONNX/AI inference) need to share large structures among a small
  cooperating set of processes, not the whole instance. Putting that in the SGA wastes
  address space and exposes it to everyone; putting it in the PGA forces copying.
- **Properties:** configurable, elastic, modular, on-demand, and reusable. Freed
  memory is returned, which reduces churn and fragmentation.

### 6.2 Namespaces
- A **namespace** is a named scoped region inside the MGA with its own stable set of
  offsets ("fixed variables"), analogous to how the fixed SGA gives stable addresses —
  but scoped per namespace instead of global.
- Any number of **foreground (FG)** and **background (BG)** processes may share a
  namespace. Membership is tracked per bucket so attach/detach can proceed in parallel.
- Each namespace keeps both a **shared segment** view and a **private segment** view
  for the attached processes.

### 6.3 Inspect and manipulate
- **Sizing:** there is no dedicated `mga_size` parameter. The MGA is sized by the
  memory-management mode — under **Unified Memory** (`MEMORY_SIZE`, 26ai) it is one
  of the segments Oracle balances automatically; otherwise it is derived internally.
  MGA is reported under the PGA umbrella (`V$PGASTAT` has `MGA allocated (under PGA)` and
  `maximum MGA allocated`), so treat it as part of what `PGA_AGGREGATE_LIMIT` guards.
- **Inspect:** MGA/DGA figures show up in `V$PGASTAT` on newer releases:
  ```sql
  SELECT name, ROUND(value/1024/1024) AS mb
  FROM v$pgastat
  WHERE name IN ('maximum MGA allocated','maximum DGA allocated','maximum PGA allocated');

  -- all dynamically tuned components
  SELECT component, ROUND(current_size/1024/1024) AS mb, ROUND(max_size/1024/1024) AS max_mb
  FROM v$memory_dynamic_components WHERE current_size > 0;
  ```
- **Manipulate:** you do not tune MGA directly. You influence it by (a) choosing the
  memory mode (`MEMORY_SIZE` for Unified Memory, else `SGA_TARGET`/`PGA_AGGREGATE_TARGET`),
  and (b) the features that use it (parallelism, vector indexes, AI inference).
- **Watch out:** MGA support and its exact monitoring views differ by release — always
  confirm against your version before relying on a statistic.
- **MGA and `/dev/shm`:** MGA's named namespaces are backed by files on **tmpfs
  (`/dev/shm`)** — a live 26ai Free container shows `KSTRCB_MGA_NMSPC_0_0.dat` and
  `KSTRCB_MGA_NMSPC_1_0.dat`, 16 MB each (32 MB total). Two separate sizing facts:
  - **AMM / Unified Memory** need `/dev/shm` to be at least the greater of
    `MEMORY_MAX_TARGET` / `MEMORY_TARGET` (or `MEMORY_SIZE`); that is the classic
    `ORA-00845` constraint. It is **not** sized to `PGA_AGGREGATE_LIMIT`.
  - The **MGA itself** needs enough tmpfs for its namespaces. In the Docker Free image
    `/dev/shm` was only 64 MB while MGA used 32 MB and the instance was healthy.
  Check with `df -h /dev/shm` (§8.6).

---

## Step 7 — How to manipulate memory (the four-step loop)

Use the same loop for every structure:

1. **See it** — query the relevant `V$` view (current size, usage, statistics).
2. **Change it** — `ALTER SYSTEM SET ... SCOPE=BOTH` (dynamic) or `SCOPE=SPFILE`
   + restart (static); use `ALTER SESSION` for session-private settings.
3. **Verify** — re-run step 1 and compare; check the alert log for errors.
4. **Persist / document** — confirm the SPFILE was updated (`SHOW PARAMETER`), and
   record *why* you changed it.

**Golden rules**

- Prefer **one knob** (ASMM `SGA_TARGET`) over hand-sizing five pools.
- Keep `SGA_MAX_SIZE` (or `MEMORY_MAX_TARGET`) as the ceiling; don't set targets above it.
- Never run `FLUSH SHARED_POOL` / `FLUSH BUFFER_CACHE` on production — they cause
  hard-parse storms and cold reads.
- Static parameters need `SCOPE=SPFILE` plus a restart: `log_buffer`,
  `sga_max_size`, `memory_max_target`, `vector_memory_size`, `memoptimize_pool_size`.
- Verify against your exact release; the vector pool needs 23ai.

### 7.1 Measurement discipline (read this before trusting a number)

- **`V$SYSSTAT`/`V$SESSTAT` are cumulative since startup.** A single number is meaningless;
  take **two snapshots** and diff them:
  ```sql
  SELECT name, value FROM v$sysstat
  WHERE name IN ('physical reads','consistent gets','redo size');
  -- ... wait ~10s, run again, and subtract
  ```
- **Use rate-based tools:** **AWR** (`DBA_HIST_*`, Diagnostics Pack licensed), **ASH**
  (`V$ACTIVE_SESSION_HISTORY`) for short spikes, **Statspack** as a free snapshot
  alternative, and `V$SYSMETRIC`/`V$METRIC` for near-real-time rates.
- **Use the advisors, not guesses:** `V$SGA_TARGET_ADVICE`, `V$DB_CACHE_ADVICE`,
  `V$PGA_TARGET_ADVICE`, `V$SHARED_POOL_ADVICE`, and (for AMM) `V$MEMORY_TARGET_ADVICE` /
  `V$MEMORY_SIZE_ADVICE` where available.
- **Correlate memory with waits, not ratios.** The real signals are wait events:
  `free buffer waits`, `db file sequential read`, `db file scattered read`,
  `log file sync`, `latch: cache buffers chains`, `cursor: mutex X`.
- **Change one thing, then measure again.** Memory knobs interact; A/B a single change at
  a time using the 4-step loop above.

---

## Step 8 — Hands-on lab (copy-paste SQL)

### 8.0 Prerequisites, the container, and the sample HR schema

**Know the Free limits first:** up to **2 CPUs**, **2 GB RAM (SGA + PGA combined)**,
and **12 GB of user data**. So keep `SGA_TARGET + PGA_AGGREGATE_TARGET ≤ ~2 GB`; larger
values are silently capped. Give Docker a little headroom:

```bash
# `read -rs` keeps the password out of your *shell history* — nothing more. Passing it
# to the container with `-e ORACLE_PWD` still stores the value in the container's Env,
# which `docker inspect` displays. For a stronger option use an env-file or a Docker
# secret (see §11.1).
read -rs -p "Choose SYS/SYSTEM password: " ORACLE_PWD; echo; export ORACLE_PWD

# Bind the listener to localhost only, and persist data across container recreation.
# `--memory=3g` leaves the host room for the ~2 GB instance + OS; it is a Docker cgroup
# cap, NOT the Free 2 GB limit (see §8.8).
docker run -d --name oradb \
  --memory=3g \
  -p 127.0.0.1:1521:1521 \
  -e ORACLE_PWD \
  -v oradata:/opt/oracle/oradata \
  container-registry.oracle.com/database/free:latest

docker logs -f oradb          # wait for "DATABASE IS READY TO USE!" then Ctrl-C
docker exec -it oradb sqlplus / as sysdba
```

**First lab step — know your release.** `free:latest` is a moving tag; at the time of
writing it reports **Oracle AI Database 26ai Free** (`23.x` version string), but it can
change. Run this first, because later steps use features older releases lack:

```sql
SELECT banner_full FROM v$version;   -- e.g. "Oracle AI Database 26ai Free Release 23.26.3.0.0"
```

| Feature used in this guide | 19c | 23ai | 26ai |
|---|---|---|---|
| ASMM `SGA_TARGET` / AMM `MEMORY_TARGET` | ✔ | ✔ | ✔ |
| Multitenant / `FREEPDB1` | ✔ | ✔ | ✔ |
| Vector pool / `vector_memory_size` (§4.8) | ✘ | ✔ | ✔ |
| Work-area views (`V$SQL_WORKAREA*`) | ✔ | ✔ | ✔ |
| MGA / `maximum MGA allocated` (§6) | ✘ | partial | ✔ |
| Unified Memory `MEMORY_SIZE` (§3.4) | ✘ | ✘ | ✔ |
| `io_uring` I/O path (§E.3) | ✘ | ✘ | verify |

Install **HR** — it does **not** ship with 21c+ (Docker Free included). The GitHub release
page has **no zip asset**, so fetch the source archive (`curl` and `unzip` are present in
the Oracle Free image):

```bash
docker exec -it oradb bash
cd /home/oracle
# Source archive for the current release (the releases page ships no binary asset):
curl -L -o dss.zip \
  https://github.com/oracle-samples/db-sample-schemas/archive/refs/tags/v23.3.zip
unzip -q dss.zip && cd db-sample-schemas-*/human_resources

# OS authentication needs no password and keeps one off the command line.
sqlplus / as sysdba
SQL> ALTER SESSION SET CONTAINER = FREEPDB1;
SQL> @hr_install.sql
#   prompts: HR password, tablespace (accept USERS), overwrite if it already exists
```

> Inside the container `$ORACLE_PWD` is **not** exported for you (the Official image
> consumes it at startup; the community `gvenzl` image uses `ORACLE_PASSWORD`). Prefer
> `sqlplus / as sysdba` (OS auth) or `sqlplus /nolog` + `CONNECT`; avoid
> `sqlplus sys/$ORACLE_PWD@...`, which puts the password on the command line (§11.1).

```sql
SET LINESIZE 200 PAGESIZE 100
ALTER SESSION SET CONTAINER = FREEPDB1;      -- do user work in the PDB
SELECT COUNT(*) FROM hr.employees;            -- expect 107
```

> **Static / system parameters are CDB-wide.** `log_buffer`, `sga_max_size`,
> `memory_max_target`, `memory_size`, `vector_memory_size`, and `memoptimize_pool_size` live
> in the **root**. Run those `ALTER SYSTEM` statements *before* switching to the PDB:
> ```sql
> ALTER SESSION SET CONTAINER = CDB$ROOT;
> -- ALTER SYSTEM SET log_buffer = 32M SCOPE = SPFILE;
> ALTER SESSION SET CONTAINER = FREEPDB1;
> ```
> `SGA_TARGET` and `PGA_AGGREGATE_TARGET` are **instance-wide** too — set them in the root
> (§3.8), not the PDB.

### 8.1 Orientation

```sql
-- Instance identity
SELECT instance_name, host_name, status, startup_time FROM v$instance;

-- Which memory mode am I in?
SHOW PARAMETER memory_size       -- Unified Memory (26ai)
SHOW PARAMETER memory_target     -- AMM
SHOW PARAMETER sga_target
SHOW PARAMETER sga_max_size
SHOW PARAMETER pga_aggregate_target
SHOW PARAMETER pga_aggregate_limit

-- SGA breakdown (Fixed/Variable/Database Buffers/Redo)
SHOW SGA
SELECT name, ROUND(bytes/1024/1024) AS mb, resizeable FROM v$sgainfo;
SELECT component, ROUND(current_size/1024/1024) AS mb, granule_size
FROM v$sga_dynamic_components WHERE current_size > 0;
```

### 8.2 Pools and components

```sql
SELECT pool, name, ROUND(bytes/1024/1024,1) AS mb
FROM v$sgastat
WHERE pool IN ('shared pool','large pool','java pool','streams pool')
ORDER BY bytes DESC FETCH FIRST 15 ROWS ONLY;

SELECT name, block_size, current_size AS mb FROM v$buffer_pool;

SELECT name, value FROM v$sysstat
WHERE name IN ('redo size','redo entries','redo buffer allocation retries');

SELECT * FROM v$vector_memory_pool;   -- 23ai

-- Reserved pool, free memory, and all tunable components
SELECT name, ROUND(bytes/1024/1024,1) AS mb FROM v$sgastat WHERE name = 'free memory';
SELECT * FROM v$shared_pool_reserved;
SELECT component, ROUND(current_size/1024/1024) AS mb FROM v$memory_dynamic_components WHERE current_size > 0;
```

### 8.3 PGA (and MGA)

```sql
SELECT name, ROUND(value/1024/1024) AS mb FROM v$pgastat
WHERE name IN ('aggregate PGA target parameter','total PGA allocated',
               'maximum PGA allocated','maximum MGA allocated','over allocation count',
               'cache hit percentage');

SELECT sid, operation_type, ROUND(actual_mem_used/1024/1024,1) AS mb_used, tempseg_size
FROM v$sql_workarea_active;

SELECT s.sid, s.username, n.name, ROUND(st.value/1024) AS kb
FROM v$sesstat st
JOIN v$statname n ON n.statistic# = st.statistic#
JOIN v$session s   ON s.sid = st.sid
WHERE n.name IN ('session uga memory','session pga memory') AND s.username IS NOT NULL;
```

### 8.4 Sessions, processes, cursors

```sql
SELECT name, description FROM v$bgprocess WHERE paddr <> '00' ORDER BY name;

SELECT s.sid, s.username, s.program, p.spid AS os_pid, s.server
FROM v$session s JOIN v$process p ON s.paddr = p.addr
WHERE s.type = 'USER';

SELECT sid, cursor_type, sql_text FROM v$open_cursor
WHERE sid = SYS_CONTEXT('USERENV','SID');
```

### 8.5 Practice changing things (DBA)

```sql
-- SGA_TARGET / PGA_AGGREGATE_TARGET are INSTANCE-wide: run them in the root. On Free
-- keep the SUM within ~2 GB.
ALTER SESSION SET CONTAINER = CDB$ROOT;
ALTER SYSTEM SET sga_target           = 1100M SCOPE = BOTH;   -- instance SGA (matches §3.8)
ALTER SYSTEM SET pga_aggregate_target = 512M  SCOPE = BOTH;   -- instance PGA
SHOW PARAMETER sga_target;

-- Per-PDB knobs (bounded by the CDB): a PDB-level SGA_TARGET is a per-PDB cap, and
-- SGA_MIN_SIZE is that PDB's guaranteed floor. Set these while in the PDB.
ALTER SESSION SET CONTAINER = FREEPDB1;
ALTER SYSTEM SET sga_min_size  = 256M SCOPE = BOTH;
ALTER SYSTEM SET open_cursors  = 500  SCOPE = BOTH;

-- PGA_AGGREGATE_LIMIT: do NOT set it below the default or it is rejected.
-- Default = greatest(2 GB, 200% of PGA_AGGREGATE_TARGET, 3 MB x PROCESSES).
-- With a 512M target the default is 2 GB, so a smaller value like 1G fails.
SHOW PARAMETER pga_aggregate_limit;      -- observe the default, usually leave it
-- ALTER SYSTEM SET pga_aggregate_limit = 2G SCOPE = BOTH;   -- only if >= default

-- CDB-wide / static change: run in the root, then restart to apply
ALTER SESSION SET CONTAINER = CDB$ROOT;
ALTER SYSTEM SET log_buffer = 32M SCOPE = SPFILE;
SELECT name, value FROM v$spparameter WHERE name = 'log_buffer';
ALTER SESSION SET CONTAINER = FREEPDB1;

-- Unified Memory (26ai): one knob for SGA+PGA+MGA+UGA (static, SPFILE)
-- ALTER SYSTEM SET memory_size = 2G SCOPE = SPFILE;   -- Free cap

-- Resize advice (enable advice first: db_cache_advice=ON)
SELECT size_for_estimate, estd_physical_read_factor FROM v$db_cache_advice;
SELECT size_for_estimate, estd_pga_cache_hit_percentage FROM v$pga_target_advice;
SELECT shared_pool_size_for_estimate, estd_lc_load_time
FROM v$shared_pool_advice WHERE shared_pool_size_factor = 1;
```

### 8.6 The OS-level bridge: prove SGA is shared and PGA is private

Part 0 asserts that the SGA is a **shared segment** and the PGA is a **process's private
heap**. This experiment shows it. Many images omit `ipcs`, `pgrep`, `ps` and even `free`;
the fallbacks below use SQL and `/proc` so they work anywhere.

**1. The shared segment (the SGA).** With `ipcs` often missing, read the kernel's
System V shared-memory table `/proc/sysvipc/shm` directly:

```bash
docker exec oradb bash -lc '
  echo "== SYSV shared segments (SGA) =="
  cat /proc/sysvipc/shm 2>/dev/null || ipcs -m
  echo "== tmpfs (AMM/MGA) =="
  ls -l /dev/shm; df -h /dev/shm'
```

- One large segment (~1.5 GB in the Free image) with `nattch` in the dozens is the **SGA**;
  `nattch` counts how many Oracle processes have attached it.
- Under **AMM**, `ipcs -m` / `/proc/sysvipc/shm` can show little or nothing because AMM uses
  **files under `/dev/shm`** instead — so an *empty* SYSV list means AMM, not "no SGA".

**2. The private side (the PGA).** Don't probe **PMON**: it is a background process, so
comparing its `/proc` counters against *your session's* `V$PROCESS` row compares two
different processes. Create a stable foreground session, find its OS pid, and read *that*
process (the following is tested on Oracle AI Database 26ai Free):

```bash
docker exec -i oradb bash -s <<'OUTER'
# a throwaway session that just sleeps
sqlplus -s / as sysdba <<'EOS' >/dev/null 2>&1 &
exec dbms_session.sleep(60);
EOS
sleep 4

SPID=$(sqlplus -s / as sysdba <<'EOS' | tr -d '[:space:]'
SET HEAD OFF FEED OFF PAGES 0
SELECT p.spid FROM v$session s JOIN v$process p ON p.addr = s.paddr
WHERE s.type = 'USER' AND s.event LIKE '%timer%' FETCH FIRST 1 ROW ONLY;
EOS
)
echo "server pid: $SPID"

echo "== OS view of that process =="
grep -E 'VmRSS|RssAnon|RssShmem|VmSwap' /proc/$SPID/status

echo "== Oracle view of the same process =="
sqlplus -s / as sysdba <<EOS
SET LINESIZE 200
COL spid FOR A8
SELECT spid, ROUND(pga_used_mem/1024/1024,1) pga_used_mb,
       ROUND(pga_alloc_mem/1024/1024,1) pga_alloc_mb
FROM v\$process WHERE spid = '$SPID';
EOS
OUTER
```

- **`RssAnon` ≈ `pga_alloc_mb`**: both describe the same process's private, anonymous
  memory. They won't match exactly — `RssAnon` also counts the binary's stack and data — but
  they should be the same order of magnitude.
- **`RssShmem` is *not* "the process's share of the SGA."** It is only the **touched** part
  of every shared mapping the process holds at that instant, so it is usually far smaller
  than `SGA / nattch` and varies with what the process has accessed.
- **`VmSwap`** should stay ≈ 0.

**3. Kernel knobs.**

```bash
# Host-side view of RAM, swap, and huge pages
grep -E "MemTotal|MemFree|MemAvailable|SwapTotal|SwapFree|Huge" /proc/meminfo
sysctl vm.swappiness vm.nr_hugepages vm.overcommit_memory 2>/dev/null
docker exec oradb bash -c 'ulimit -l'      # memlock limit (needed for huge pages, §8.9)
```

What you just proved:
- **SGA = one shared segment** attached by N processes (`/proc/sysvipc/shm`).
- **PGA = private anonymous memory** per process (`RssAnon` ≈ `pga_alloc_mem`).
- **AMM** shows up as files under `/dev/shm`; **ASMM + huge pages** shows huge-page
  mappings with a small `/dev/shm` — the incompatibility from §3.3 made real.
- **`vm.swappiness`** should be low; **`memlock`** high enough for huge pages, with
  `nr_hugepages × Hugepagesize` covering the SGA.

### 8.7 Four experiments that make the model concrete

All four were run on Oracle AI Database 26ai Free in the §8.0 container. Magnitudes vary by
box; the *direction* is what matters. Snapshot the counter before and after each step.

**A — Soft vs. hard parse (why bind variables matter).** Create scratch data, then run the
same lookup twice: once with the literal concatenated into the SQL text, once with a bind
variable. `parse count (total)` / `(hard)` come from `V$MYSTAT`.

```sql
ALTER SESSION SET CONTAINER = FREEPDB1;
DROP TABLE t_parse PURGE;
CREATE TABLE t_parse (id NUMBER, pad VARCHAR2(200));   -- 100k rows ≈ 20 MB, used again in B and C
INSERT INTO t_parse SELECT level, RPAD('x',200,'y') FROM dual CONNECT BY level <= 100000;
COMMIT;
```

```sql
COL name FOR A22
-- 1) literals: every iteration is a *different* SQL string -> hard parse
DECLARE v VARCHAR2(200);
BEGIN
  FOR i IN 1..500 LOOP
    EXECUTE IMMEDIATE 'SELECT pad FROM t_parse WHERE id = '||i INTO v;
  END LOOP;
END;
/
SELECT n.name, m.value FROM v$mystat m JOIN v$statname n ON n.statistic#=m.statistic#
WHERE n.name IN ('parse count (total)','parse count (hard)') ORDER BY n.name;

-- 2) binds: one SQL string reused -> soft parse after the first
DECLARE v VARCHAR2(200);
BEGIN
  FOR i IN 1..500 LOOP
    EXECUTE IMMEDIATE 'SELECT pad FROM t_parse WHERE id = :1' INTO v USING i;
  END LOOP;
END;
/
SELECT n.name, m.value FROM v$mystat m JOIN v$statname n ON n.statistic#=m.statistic#
WHERE n.name IN ('parse count (total)','parse count (hard)') ORDER BY n.name;
```

Observed: the literal loop added ~1,000 to `parse count (hard)`; the bind loop added **5**.
Hard parsing is CPU spent in the shared pool (§4.4); a parser-heavy workload also shows up as
`library cache`/`row cache` waits and shared-pool churn.

**B — A work area spilling to temp.** Force *manual* work-area mode with a tiny
`sort_area_size`, then run a real sort. The AUTO histogram (§5.1) only reports
one-pass/multi-pass once a work area exceeds the auto target, so manual mode makes the spill
unmistakable via `sorts (disk)` and `V$SQL_WORKAREA`.

```sql
ALTER SESSION SET workarea_size_policy = MANUAL;
ALTER SESSION SET sort_area_size = 65536;   -- 64 KB (deliberately tiny)

SELECT n.name, m.value FROM v$mystat m JOIN v$statname n ON n.statistic#=m.statistic#
WHERE n.name IN ('sorts (memory)','sorts (disk)') ORDER BY n.name;   -- baseline

DROP TABLE t_sorted PURGE;
CREATE TABLE t_sorted AS SELECT * FROM t_parse ORDER BY pad;   -- must sort ~20 MB

SELECT n.name, m.value FROM v$mystat m JOIN v$statname n ON n.statistic#=m.statistic#
WHERE n.name IN ('sorts (memory)','sorts (disk)') ORDER BY n.name;

SELECT operation_type, policy, ROUND(last_tempseg_size/1024/1024,1) last_temp_mb
FROM v$sql_workarea WHERE last_tempseg_size > 0
ORDER BY last_tempseg_size DESC FETCH FIRST 3 ROWS ONLY;

ALTER SESSION SET workarea_size_policy = AUTO;   -- reset
```

Observed: `sorts (disk)` went 0 → 1 and `V$SQL_WORKAREA` reported a ~21 MB
`last_tempseg_size` — the sort **spilled to a TEMP segment**. Note the exact histogram column
names (a common mistake is `optimal_count`/`onepass_count`/`multipass_count`, which do not
exist):

```sql
SELECT optimal_executions, onepass_executions, multipasses_executions
FROM v$sql_workarea_histogram;
-- the currently-running operation (note: V$SQL_WORKAREA_ACTIVE, not V$SQL_WORKAREA):
SELECT sid, operation_type, policy, number_passes,
       ROUND(actual_mem_used/1024/1024,1) mem_mb, ROUND(tempseg_size/1024/1024,1) temp_mb
FROM v$sql_workarea_active;
```

On a small Free instance a single sort usually stays `OPTIMAL` (the auto target lets one work
area grow large); one-pass/multi-pass appear only when a work area is forced past the auto
bound — give it more PGA or give it less to sort (§5.1).

**C — Cold vs. warm reads (the buffer cache).** `ALTER SYSTEM FLUSH BUFFER_CACHE` (test
instances only — §11) evicts the read blocks. Read the same table twice and watch
`physical reads` vs `session logical reads`:

```sql
SELECT n.name, m.value FROM v$mystat m JOIN v$statname n ON n.statistic#=m.statistic#
WHERE n.name IN ('physical reads','session logical reads') ORDER BY n.name;

ALTER SYSTEM FLUSH BUFFER_CACHE;
SELECT COUNT(*) FROM t_parse;   -- cold: blocks come off disk
SELECT n.name, m.value FROM v$mystat m JOIN v$statname n ON n.statistic#=m.statistic#
WHERE n.name IN ('physical reads','session logical reads') ORDER BY n.name;

SELECT COUNT(*) FROM t_parse;   -- warm: no physical reads
SELECT n.name, m.value FROM v$mystat m JOIN v$statname n ON n.statistic#=m.statistic#
WHERE n.name IN ('physical reads','session logical reads') ORDER BY n.name;
```

Observed: after the flush, `physical reads` jumped by ~2,900 (the 100k-row table); the second
pass added **0** physical reads and only logical reads. That gap *is* the buffer cache (§4.2).

**D — Per-row vs. batched commits (`log file sync`).** Every `COMMIT` must make LGWR flush
redo, which shows as the `log file sync` wait. Subtlety on 23ai/26ai: a **plain** `COMMIT`
inside a PL/SQL loop is often **batched** (one sync for many commits), so make the per-row
cost explicit with `COMMIT WRITE IMMEDIATE WAIT`:

```sql
DROP TABLE t_commit PURGE;
CREATE TABLE t_commit (id NUMBER, v VARCHAR2(100));

SELECT total_waits, ROUND(time_waited_micro/1000,1) ms
FROM v$system_event WHERE event = 'log file sync';   -- baseline

-- 200 separate, synchronous commits -> one log file sync each
BEGIN
  FOR i IN 1..200 LOOP
    INSERT INTO t_commit VALUES (i, RPAD('x',100,'y'));
    COMMIT WRITE IMMEDIATE WAIT;
  END LOOP;
END;
/
SELECT total_waits, ROUND(time_waited_micro/1000,1) ms
FROM v$system_event WHERE event = 'log file sync';   -- +200 waits

-- one batched commit -> a single log file sync for 200 rows
BEGIN
  FOR i IN 201..400 LOOP
    INSERT INTO t_commit VALUES (i, RPAD('x',100,'y'));
  END LOOP;
  COMMIT;
END;
/
SELECT total_waits, ROUND(time_waited_micro/1000,1) ms
FROM v$system_event WHERE event = 'log file sync';   -- +1 wait
```

Observed: the 200 forced per-row commits added **200** `log file sync` waits (~0.2 s of wait
in that run); the batched 200 rows added **1**. Same work, ~200× fewer syncs — the fix named
in the §9 symptom table.

### 8.8 Docker failure modes (and how to read them)

The Free images are bounded, so most "memory" failures are really container or kernel limits.

| Error / symptom | What it means | Fix |
|---|---|---|
| `ORA-00845: MEMORY_TARGET not supported` | AMM needs `/dev/shm` ≥ `MEMORY_TARGET`, but Docker defaults `/dev/shm` to **64 MB** | `--shm-size=1g` (≥ target), or use ASMM (`sga_target`) |
| `ORA-27102: out of memory` | SGA request exceeds a kernel/cgroup limit | raise `--memory`/`--shm-size`; lower `sga_target` |
| container exits, session killed | the **cgroup** OOM-killed a process | raise `--memory=`; check `docker stats` |
| huge-page SGA won't start | too few huge pages / `memlock` too low | see §8.9; cannot create huge pages *inside* a container |

Triage commands (host side):

```bash
docker stats --no-stream oradb                   # live cgroup memory/CPU
docker inspect -f '{{.State.OOMKilled}}' oradb   # did the host OOM-kill it?
df -h /dev/shm                                   # AMM/MGA backing store
dmesg | tail -30                                 # 'Killed process ...' = host OOM, not Oracle
```

Key point: Oracle Database Free caps the **SGA at 2 GB**. Giving the container *more* RAM does
not raise the in-DB cap, but giving it *less* — or leaving `/dev/shm` at 64 MB — is a very
common way to break startup.

### 8.9 HugePages (why the SGA wants them)

Linux tracks RAM in 4 KB pages by default; a large SGA then needs a big page table, and the
SGA can be swapped out. **Huge pages** (typically 2 MB) shrink that overhead and, allocated up
front, are **not swappable** — Oracle recommends them for any serious SGA (§3.3).

```bash
grep -E 'Hugepagesize|HugePages_Total|HugePages_Free|HugePages_Rsvd' /proc/meminfo
# this container: Hugepagesize 2048 kB, HugePages_Total 0  -> none configured
```

Sizing and enabling (host, root):

```bash
# 1) how many 2 MB pages cover the SGA (plus a small reserve)
echo $(( 1100/2 + 8 ))          # ~558 pages for a 1.1 GB SGA
# 2) reserve them now and persistently
sysctl -w vm.nr_hugepages=558
echo 'vm.nr_hugepages = 558' >> /etc/sysctl.d/99-oracle.conf
# 3) let Oracle lock them, then restart the instance
ulimit -l unlimited             # in Docker: --ulimit memlock=-1:-1
```

Then point the database at them and confirm from the alert log:

```sql
SHOW PARAMETER use_large_pages;   -- TRUE (use if available) / ONLY / FALSE / AUTO
ALTER SYSTEM SET use_large_pages = ONLY SCOPE = SPFILE;   -- restart to apply
```

```
# alert log at startup:
Large Pages Information
  Total Large Page size = 1152 MB
  SGA: 1152 MB, LargePages used = 558 pages
```

Caveats: huge pages back only the **SGA**, not general `malloc`; and inside a container they
come from the **host** kernel, so you cannot create them "in" the container. If
`use_large_pages=ONLY` and there are too few, startup fails (`ORA-27102`), so leave headroom.
**AMM (`memory_target`) is incompatible with huge pages** (§3.3) — use ASMM.

### 8.10 Minimal shared-memory demo (SGA vs. PGA in ~30 lines of C)

Part 0 claims the SGA is one shared segment and the PGA is private. Reproduce that with the
same OS primitives Oracle uses: `shmget()/shmat()` (System V — how the SGA appears in
`/proc/sysvipc/shm`) and the `MAP_SHARED` vs `MAP_PRIVATE` contrast (shared vs. private). Run
on the **host** (the Oracle image ships no compiler):

```c
/* memdemo.c :  cc -O2 -o memdemo memdemo.c && ./memdemo        */
/*              ./memdemo sysv     (also maps a System V segment) */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/mman.h>
#include <sys/ipc.h>
#include <sys/shm.h>
#include <sys/wait.h>

int main(int argc, char **argv) {
  size_t n = 4096;
  if (argc > 1 && !strcmp(argv[1], "sysv")) {        /* shmget = how the SGA is made */
    int id = shmget(IPC_PRIVATE, n, IPC_CREAT | 0600);
    char *p = shmat(id, NULL, 0);
    strcpy(p, "SGA-like shared segment");
    if (fork() == 0) { printf("  child sees: '%s'\n", p); _exit(0); }
    wait(NULL); shmdt(p); shmctl(id, IPC_RMID, NULL);
    return 0;
  }
  char *sh = mmap(NULL, n, PROT_READ|PROT_WRITE,
                  MAP_ANONYMOUS|MAP_SHARED,  -1, 0);   /* one copy, shared */
  char *pr = mmap(NULL, n, PROT_READ|PROT_WRITE,
                  MAP_ANONYMOUS|MAP_PRIVATE, -1, 0);   /* private copies   */
  strcpy(sh, "shared"); strcpy(pr, "private");
  if (fork() == 0) { sh[0]='S'; sh[1]='H'; pr[0]='P'; pr[1]='R'; _exit(0); }
  wait(NULL);
  printf("MAP_SHARED  parent sees '%s'  (child's write survived)\n", sh);
  printf("MAP_PRIVATE parent sees '%s'  (parent's copy unchanged)\n", pr);
  return 0;
}
```

```bash
cc -O2 -o memdemo memdemo.c && ./memdemo
# MAP_SHARED  parent sees 'SHared'   (child's write survived)
# MAP_PRIVATE parent sees 'private'  (parent's copy unchanged)
./memdemo sysv
# child sees: 'SGA-like shared segment'
```

`MAP_SHARED` is the SGA (one physical copy, all processes see writes); `MAP_PRIVATE` is the
PGA (copy-on-write, private). The `sysv` mode is literally how the SGA shows up in
`/proc/sysvipc/shm` (§8.6). Under AMM/MGA Oracle uses `mmap(MAP_SHARED)` **files** under
`/dev/shm` instead — same idea, different interface.

---

## Step 9 — Cheatsheet: parameters, views, symptoms

**Sizing parameters**

| Structure | Parameter | Dynamic? |
|---|---|---|
| Whole SGA (ASMM) | `sga_target`, `sga_max_size` | target ✔ / max static |
| Whole SGA+PGA (AMM) | `memory_target`, `memory_max_target` | target ✔ / max static |
| SGA+PGA+MGA+UGA (Unified, 26ai) | `memory_size` | ✘ static (SPFILE) |
| PGA | `pga_aggregate_target`, `pga_aggregate_limit` | ✔ |
| Buffer cache | `db_cache_size`, `db_keep_cache_size`, `db_recycle_cache_size`, `db_nk_cache_size` | ✔ |
| Shared pool | `shared_pool_size`, `shared_pool_reserved_size` | pool ✔ / **reserved ✘ static** |
| Large pool | `large_pool_size` | ✔ |
| Java pool | `java_pool_size` | ✔ |
| Streams pool | `streams_pool_size` | ✔ |
| Redo log buffer | `log_buffer` | ✘ static |
| In-Memory area | `inmemory_size` | ✘ mostly static |
| Vector pool | `vector_memory_size` | ✘ static |
| Memoptimize pool | `memoptimize_pool_size` | ✘ static |
| MGA (Managed Global Area) | no direct parameter — set via `memory_size` / memory mode | derived |
| Work areas | `workarea_size_policy`, `sort_area_size`, `hash_area_size` | session/system |
| Cursors | `open_cursors`, `session_cached_cursors` | ✔ |

**Monitoring views**

| View | Shows |
|---|---|
| `V$SGA`, `V$SGAINFO`, `V$SGASTAT` | SGA totals and per-pool breakdown |
| `V$SGA_DYNAMIC_COMPONENTS` | Each auto-tunable component + granule size |
| `V$SGA_TARGET_ADVICE` / `V$MEMORY_TARGET_ADVICE` | Projected benefit of a bigger target |
| `V$BUFFER_POOL`, `V$DB_CACHE_ADVICE`, `V$BH` | Buffer cache sizing |
| `V$SHARED_POOL_ADVICE`, `V$LIBRARYCACHE`, `V$SQLAREA` | Shared pool / parsing |
| `V$RESULT_CACHE_*` | Result cache |
| `V$SHARED_POOL_RESERVED` | Reserved pool hits/misses |
| `V$PGASTAT`, `V$PGA_TARGET_ADVICE` | PGA (and MGA/DGA) usage and advice |
| `V$SQL_WORKAREA*`, `V$SESSTAT`, `V$PROCESS_MEMORY` | Work areas and per-session memory |
| `V$MEMORY_DYNAMIC_COMPONENTS` | All dynamically tuned components (SGA/PGA/MGA) |
| `V$INMEMORY_AREA` | In-Memory Column Store |
| `V$VECTOR_MEMORY_POOL` | Vector pool (23ai) |
| `V$PARAMETER`, `V$SPPARAMETER` | Current vs. stored parameter values |

**Symptom → likely cause → fix**

| Symptom | Likely cause | Fix |
|---|---|---|
| `ORA-04031` unable to allocate shared memory | shared/large pool too small or fragmented | raise `shared_pool_size` / `large_pool_size`; pin objects |
| `ORA-01000` max open cursors | cursor leak | close cursors; raise `open_cursors` |
| High `parse count (hard)` | literals instead of binds | use bind variables; consider `cursor_sharing` |
| Wait `log buffer space` | LGWR can't keep up with redo | bigger `log_buffer`, bigger/faster log files |
| Wait `log file sync` | commit too often | batch commits |
| `over allocation count` rising | `pga_aggregate_target` too small | raise it or reduce concurrency |
| Multi-pass work areas | sort/hash spills to temp | more PGA or better SQL/indexes |
| `PGA_AGGREGATE_LIMIT` termination | hard limit hit | raise limit or fix top PGA consumers |

---

## Step 10 — Diagram coverage checklist

![Oracle instance architecture](oracle_architecture.png)

Every label in `oracle_architecture.png` and where it is explained in this document:

| Component in the diagram | Covered in |
|---|---|
| Instance | [Step 1](#step-1--instance-vs-database-vs-session) |
| System Global Area (SGA) | [Step 2](#step-2--sga-vs-pga-at-a-glance) / [Step 4](#step-4--sga-components-one-by-one) |
| Shared Pool | [4.4](#44-shared-pool) |
| └ Library Cache | [4.4](#44-shared-pool) |
| └ Shared SQL Area | [4.4](#44-shared-pool) |
| └ Private SQL Area (Shared Server Only) | [4.4](#44-shared-pool) / [5.2](#52-private-sql-area) |
| └ Data Dictionary Cache | [4.4](#44-shared-pool) |
| └ Server Result Cache | [4.4](#44-shared-pool) |
| └ Other | [4.4](#44-shared-pool) |
| └ Reserved Pool | [4.4](#44-shared-pool) |
| Large Pool | [4.5](#45-large-pool) |
| └ Response Queue / Request Queue | [4.5](#45-large-pool) |
| Database Buffer Cache | [4.2](#42-database-buffer-cache) |
| Redo Log Buffer | [4.3](#43-redo-log-buffer) |
| Fixed SGA | [4.1](#41-fixed-sga) |
| Java Pool | [4.6](#46-java-pool) |
| Streams Pool | [4.7](#47-streams-pool) |
| Vector Pool (23ai) | [4.8](#48-vector-pool-23ai) |
| PGA | [Step 5](#step-5--pga-components-one-by-one) |
| └ SQL Work Areas | [5.1](#51-sql-work-areas) |
| └ Session Memory (UGA) | [5.3](#53-session-memory-uga) |
| └ Private SQL Area | [5.2](#52-private-sql-area) |
| MGA (Managed Global Area) | [Step 6](#step-6--mga-managed-global-area-and-namespaces) |
| └ Namespace 1 / 2 / 3 (shared across FG and BG) | [6.2](#62-namespaces) |
| Server Process / Client Process | [Step 1](#step-1--instance-vs-database-vs-session) |
| Legend: Free Memory | [4.11](#411-legend-free-memory-and-io-buffer-areas) |
| Legend: I/O Buffer Area | [4.5](#45-large-pool) / [4.11](#411-legend-free-memory-and-io-buffer-areas) |
| Legend: UGA | [5.3](#53-session-memory-uga) |
| Legend: Client Process | client-side swatch — [Step 1](#step-1--instance-vs-database-vs-session) |

**Diagram notation.** In `oracle_architecture.png`: **shaded boxes** are the structures
above; **solid arrows** show what a process shares at that moment; **dashed arrows**
(e.g. toward *Namespace 2/3*) mean *possible* sharing by processes that may **attach
dynamically** later — MGA membership is dynamic, unlike the instance-wide SGA. See
[6.1](#61-what-the-mga-is).

Bonus components present in Oracle but not drawn in this diagram (still covered):
In-Memory Area, Memoptimize Pool, and granules — see [4.9](#49-memoptimize-pool-bonus-18c)
and [4.10](#410-other-optional-sga-subareas).

---

## Step 11 — Security considerations

- Change the example `ORACLE_PWD` and never expose port 1521 beyond localhost (see the
  hardened `docker run` in §8.0).
- `ALTER SYSTEM` requires the `ALTER SYSTEM` privilege; use a least-privilege DBA account,
  not `SYS`, for day-to-day work.
- `V$` views expose **other users'** SQL text and bind values; grant `SELECT_CATALOG_ROLE`
  only to those who need it.
- `FLUSH SHARED_POOL` and `FLUSH BUFFER_CACHE` cause hard-parse storms and cold reads —
  never run them in production.
- Use `SCOPE=SPFILE` for static parameters (`log_buffer`, `vector_memory_size`,
  `sga_max_size`); they apply only after a restart.
- Parameter and view availability varies by release (the vector pool needs 23ai); always
  verify against your version.

### 11.1 Memory, secrets, and least privilege

- **Literals and PII live in the shared pool.** Raw SQL text (and, with literal SQL, the
  values) is stored in the library cache and readable via **`V$SQL`/`V$SQLTEXT`** —
  including data from *other* schemas. Prefer **bind variables** (less leakage, fewer hard
  parses).
- **Binds can be captured too:** `V$SQL_BIND_CAPTURE`, `DBMS_XPLAN`, and trace files can
  expose bind values — audit who may read them.
- **TDE keys are in memory.** Transparent Data Encryption master/rekey material is held by
  the instance; protect the keystore and wallet credentials, and treat memory/core dumps as
  sensitive.
- **Core & trace dumps** (`DIAGNOSTIC_DEST`, background dump destinations) can contain SQL,
  bind values, and row data — restrict permissions and never ship them unencrypted.
- **Container secrets:** `-e ORACLE_PWD=...` leaks into shell history and `docker inspect`;
  use `read -rs`/Docker secrets (§8.0), bind to `127.0.0.1`, and rotate `SYS`, `SYSTEM`,
  and `PDBADMIN` passwords.
- **Least privilege for monitoring:** give a dedicated user `SELECT` on the needed `V$`
  views instead of `SYSDBA` for routine memory inspection.

---

## Glossary

| Term | Meaning |
|---|---|
| **Instance** | The running Oracle software: background processes + SGA (+PGA). Wears out on shutdown. |
| **Database** | The on-disk files (data, control, redo, undo). Persists across instance lifetimes. |
| **Session** | A logical connection on top of a process. Owns a cursor/state area (UGA). |
| **CDB / PDB** | Multitenant container database / pluggable database. 23ai Free is a CDB with `FREEPDB1`. |
| **SGA** | System Global Area — shared by every server/background process of the instance. |
| **PGA** | Program Global Area — private, non-shared memory of one server process. |
| **UGA** | User Global Area — session state; lives in the SGA under shared server, else in the PGA. |
| **MGA** | Managed Global Area (26ai) — semi-shared, dynamically-attached region (namespaces). |
| **DGA** | Direct Global Area — the part of the PGA used for direct I/O / parallel-query buffers. |
| **Granule** | The allocation/deallocation unit for dynamic SGA components (`V$SGA_DYNAMIC_COMPONENTS.GRANULE_SIZE`). |
| **ASMM** | Automatic Shared Memory Management — `SGA_TARGET` sizes the SGA; PGA via `PGA_AGGREGATE_TARGET`. |
| **AMM** | Automatic Memory Management — `MEMORY_TARGET` moves memory between SGA and PGA. Needs `/dev/shm`. |
| **Unified Memory** | 26ai `MEMORY_SIZE` — one knob for SGA + PGA + MGA + UGA. |
| **Buffer cache** | SGA cache of data blocks (`db_cache_size`); the read path. |
| **Shared pool** | SGA area holding the library cache, dictionary cache, result cache, reserved pool. |
| **Library cache** | Parsed SQL/PLSQL and their execution plans (shared SQL areas). |
| **Data dictionary cache** | Cached rows from the data dictionary to avoid recursive queries. |
| **Large pool** | Backs shared server, parallel query, RMAN I/O buffers, `DBMS_LOB` sessions. |
| **Redo log buffer** | SGA buffer for redo records; flushed by LGWR on commit/3 s/full. |
| **Fixed SGA** | Internal bookkeeping created at startup; grows with `processes`/`sessions`. |
| **In-Memory area** | Column store (`inmemory_size`) for scan-heavy analytics. |
| **Vector pool** | 23ai+ area (`vector_memory_size`) for vector indexes / AI workloads. |
| **Work area** | PGA memory for a sort, hash join, or bitmap merge. |
| **Optimal / one-pass / multi-pass** | Work area fits in memory / spills once to TEMP / spills repeatedly. |
| **Hard / soft parse** | New SQL text compiled from scratch / an existing plan reused. Bind variables make parses soft. |
| **LGWR / DBWn / CKPT** | Background processes that write redo, write dirty blocks, and signal checkpoints. |
| **Huge pages** | Larger (usually 2 MB) OS pages that back the SGA without a huge page table or swapping. |
| **`shmget` / `shmat` / `mmap`** | OS calls to create/attach a shared segment; how the SGA (and AMM files) are mapped. |
| **Namespace** | A named region inside the MGA shared between a chosen set of processes. |

---

## Practice questions (with answers)

<details><summary><b>Part 0 &amp; Step 0</b></summary>

1. **Why does a cache exist, and what does a miss cost?** RAM is ~10^4× faster than disk and
   the working set is usually small, so keeping hot data in RAM avoids most disk trips. A miss
   pays disk latency (plus, for a dirty block, a later write-back).
2. **Virtual vs. physical address?** Programs use virtual addresses; the **MMU** translates them
   to physical addresses via the page table (TLB-cached). The OS gives each process its own
   virtual space.
3. **Two problems memory solves for a database.** Avoid re-reading/re-computing hot data (buffer
   cache, caches) and decouple fast producers from slow writers (redo log buffer, dirty blocks).

</details>

<details><summary><b>Step 1–2</b></summary>

4. **Instance vs. database vs. session?** The *database* is the files on disk; the *instance* is
   the processes + memory that run against them; a *session* is one connection with private state.
5. **Shared vs. private?** SGA = shared by all processes; PGA = private per server process; UGA
   = private to a session (in the PGA unless shared server is used).

</details>

<details><summary><b>Step 3–4</b></summary>

6. **You set `sga_target=1100M` inside `FREEPDB1` — does the instance SGA shrink?** No. In a PDB
   that parameter is a *per-PDB cap*; the instance total is set in `CDB$ROOT` (§3.8–3.9).
7. **AMM fails with ORA-00845?** `MEMORY_TARGET` needs `/dev/shm` ≥ the target; Docker defaults
   `/dev/shm` to 64 MB. Raise `--shm-size` or switch to ASMM (`sga_target`).
8. **What does one `UPDATE … COMMIT` touch?** Buffer cache (find/modify the block), redo log
   buffer (the change vector), undo (in the buffer cache), then LGWR on commit; DBWn writes the
   dirty block later.
9. **Why is cache-hit ratio a bad target?** It *rises* when you do less work (e.g. a full scan
   touches each block once), so chasing it can hide bad plans. Tune the waits/statements instead.

</details>

<details><summary><b>Step 5–6</b></summary>

10. **Optimal vs. one-pass vs. multi-pass?** The work area holds the whole input (optimal),
    spills to TEMP once (one-pass), or re-reads TEMP repeatedly (multi-pass) — each step is
    slower and pollutes TEMP.
11. **Where does the MGA live on Free, and why is it in PGA figures?** On tmpfs files under
    `/dev/shm` (26ai), yet its allocation is **accounted under PGA** — so `maximum PGA allocated`
    can exceed what `/dev/shm` stores (§6.3).
12. **How does MGA membership differ from the SGA?** The SGA is attached by every process for the
    instance's life; MGA namespaces are attached **dynamically** by only the processes that opt in.

</details>

<details><summary><b>Step 7–8</b></summary>

13. **The four-step manipulation loop?** (1) Inspect the current value/usage, (2) decide
    static vs dynamic, (3) change it (`ALTER SYSTEM`/`SESSION` with the right `SCOPE`), (4) verify
    the effect and watch the waits.
14. **After `FLUSH BUFFER_CACHE`, why does the second read add 0 physical reads?** The first read
    repopulated the buffer cache, so the second pass serves every block from RAM (only logical
    reads grow) — §8.7C.
15. **Why did a plain per-row `COMMIT` loop show only one `log file sync` on 26ai?** 23ai/26ai
    **batches** plain commits issued in a tight PL/SQL loop into one redo sync; force each one
    with `COMMIT WRITE IMMEDIATE WAIT` to see 200 waits — §8.7D.

</details>

---

## References

Primary Oracle documentation — always verify details against your exact release:

- **Oracle AI Database Concepts — Memory Architecture (26c)**
  <https://docs.oracle.com/en/database/oracle/oracle-database/26/cncpt/memory-architecture.html>
- Oracle AI Database Administrator's Guide — *Managing Memory*
- Oracle AI Database Reference — initialization parameters (`SGA_TARGET`,
  `PGA_AGGREGATE_TARGET`, `MEMORY_SIZE`, `MEMORY_TARGET`, `VECTOR_MEMORY_SIZE`, …)
- Oracle AI Database Performance Tuning Guide — memory advisors and wait events
- *The Elements of Computing Systems* (nand2tetris) — the hardware foundations in Part 0

This guide targets **Oracle Database 23ai Free** (with 19c notes and 26ai features called
out explicitly).