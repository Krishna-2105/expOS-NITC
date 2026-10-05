# 🖥️ eXpOS — Experimental Operating System for the XSM Machine

<p align="center">
  <img src="https://img.shields.io/badge/Language-SPL%20%7C%20ExpL%20%7C%20C-blue?style=for-the-badge&logo=gnu&logoColor=white"/>
  <img src="https://img.shields.io/badge/Platform-Linux-orange?style=for-the-badge&logo=linux&logoColor=white"/>
  <img src="https://img.shields.io/badge/Status-In%20Progress-yellow?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Stages-18%2F28-blueviolet?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Institution-NIT%20Calicut-red?style=for-the-badge"/>
</p>

<p align="center">
  A teaching operating system built stage by stage on the simulated XSM machine — covering bootstrapping, paging, interrupt handling, multiprogramming, a round-robin scheduler, modular kernel design, terminal I/O, a program loader (Exec) and a disk interrupt handler.
</p>

<p align="center">
  <a href="#-overview">Overview</a> •
  <a href="#-architecture">Architecture</a> •
  <a href="#-features">Features</a> •
  <a href="#-project-structure">Structure</a> •
  <a href="#-getting-started">Getting Started</a> •
  <a href="#-usage">Usage</a> •
  <a href="#-implementation-stages">Stages</a> •
  <a href="#-documentation">Docs</a> •
  <a href="#-contributing">Contributing</a>
</p>

---

## 📌 Overview

**eXpOS (Experimental Operating System)** is an educational OS developed at **NIT Calicut**. Students build a working kernel incrementally, using a roadmap of **28 stages**, for a simulated 16-bit-word machine called the **eXperimental String Machine (XSM)**.

This repository holds my own implementation of the OS, written in **SPL** (Systems Programming Language, used for kernel code) and **ExpL** (Experimental Language, used for user programs). It currently covers **Stages 1–18** (up to the Disk Interrupt Handler). Stages 19–25 are the planned next milestones, and the roadmap is tracked below.

Nothing here wraps an existing OS. The boot sequence, page tables, process table, scheduler, kernel modules, interrupt routines and system-call handlers are all written from scratch against the XSM hardware specification.

### ✨ At a Glance

| Component | Details |
|---|---|
| 🧮 Machine | XSM — 512-word pages, paged virtual memory, privileged / unprivileged modes |
| 💽 Disk | 512 blocks × 512 words, formatted as **eXpFS** (inode table, disk free list, root file) |
| 🧠 Memory | 128 pages; per-process page table of 20 words (10 logical pages) |
| 🚀 Boot | OS startup code (block 0) → **Boot Module** (MOD_7) → idle process first, then INIT |
| ⏱️ Scheduler | Round-robin (MOD_5) driven by the timer interrupt |
| 🧩 Kernel Modules | Resource Manager (0), Process Manager (1), Memory Manager (2), Device Manager (4), Scheduler (5), Boot (7) |
| 🔌 Interrupts | Timer, Disk (INT 2), Console, Read (INT 6), Write (INT 7), Exec (INT 9), Exit (INT 10) |
| 🛠️ Tooling | XFS-Interface (disk tool), SPL compiler, ExpL compiler, XSM simulator with debugger |

---

## 🏛 Architecture

```
                       ┌─────────────────────────┐
                       │   User Programs (ExpL)  │   exposcall("Write" | "Read" | "Exec" ...)
                       └────────────┬────────────┘
                                    │  calls into
                       ┌────────────▼────────────┐
                       │  Shared Library         │   disk blocks 13–14 → memory pages 63–64
                       │  (logical pages 0–1)    │   (maps each system call to an INT n)
                       └────────────┬────────────┘
                                    │  INT n   (software interrupt, user → kernel mode)
    ┌───────────────────────────────▼───────────────────────────────────┐
    │                       Interrupt Layer (kernel mode)               │
    │                                                                   │
    │   Timer     Disk (INT 2)    Console    INT 6 Read    INT 7 Write  │
    │   INT 9 Exec    INT 10 Exit    Exception Handler                  │
    └───────────────────────────────┬───────────────────────────────────┘
                                    │  CALL MOD_n   (function number in R1, args in R2, R3 ...)
    ┌───────────────────────────────▼───────────────────────────────────┐
    │                          Kernel Modules                           │
    │                                                                   │
    │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐  │
    │  │ MOD_0       │ │ MOD_1       │ │ MOD_2       │ │ MOD_4       │  │
    │  │ Resource    │ │ Process     │ │ Memory      │ │ Device      │  │
    │  │ Manager     │ │ Manager     │ │ Manager     │ │ Manager     │  │
    │  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘  │
    │  ┌─────────────────────────┐   ┌─────────────────────────────┐    │
    │  │ MOD_5  Scheduler        │   │ MOD_7  Boot Module          │    │
    │  └─────────────────────────┘   └─────────────────────────────┘    │
    └───────────────────────────────┬───────────────────────────────────┘
                                    │
    ┌───────────────────────────────▼───────────────────────────────────┐
    │                  OS Data Structures (in memory)                   │
    │   Process Table • Page Tables • System Status Table               │
    │   Terminal Status Table • Disk Status Table • Memory Free List    │
    │   Inode Table (cached) • Per-Process Resource Table               │
    └───────────────────────────────┬───────────────────────────────────┘
                                    │
    ┌───────────────────────────────▼───────────────────────────────────┐
    │                    XSM Machine  (simulated)                       │
    │       CPU + MMU  •  Memory  •  Disk (disk.xfs)  •  Terminal       │
    └───────────────────────────────────────────────────────────────────┘
```

> User code never calls a kernel module directly. Modules are invoked only from interrupt routines, other modules, or the OS startup code, and they share the calling process's kernel stack.

> 📐 For the official design diagrams, see the [eXpOS Design documentation](https://exposnitc.github.io/expos-docs/os-design/).

### 🔄 Control Flow of a `Write` System Call

```
ExpL: exposcall("Write", -2, word)
        │
        ▼
Library  → pushes args and system-call number → INT 7
        │
        ▼
INT 7 handler   (switches to kernel stack, validates file descriptor, translates logical → physical address)
        │
        ▼
MOD_4  Device Manager  : Terminal Write
        │      ├── MOD_0 Acquire Terminal  (blocks and calls the scheduler if the terminal is busy)
        │      ├── print word
        │      └── MOD_0 Release Terminal  (wakes processes in WAIT_TERMINAL)
        ▼
INT 7   → writes return value to user stack → restores user SP → IRET
```

---

## 🔧 Features

### 🚀 Bootstrap & Boot Module
- The ROM loads **block 0** (the OS startup code) into page 1; it hand-creates the **idle** process and calls the **Boot Module (MOD_7)**.
- The Boot Module loads all interrupt routines, kernel modules, the library and the INIT program from disk into memory.
- It sets up the INIT page table and process table entry, marks the first 83 pages as reserved in the **Memory Free List**, and initialises the status tables.
- The idle process runs first so that its kernel context is initialised before any other process is scheduled.

### 🧠 Memory Model & Paging
- Each process has **10 logical pages**: library (0–1), heap (2–3), code (4–7), stack (8–9), described by a 20-word page table.
- Executables follow the **XEXE** format (8-word header, entry point at word 1).
- Page protection through auxiliary bits (code and library are read-only; heap and stack are read/write).
- **Memory Manager (MOD_2)** provides `Get Free Page` and `Release Page` using a reference-counted free list. When no page is free, the caller sleeps in `WAIT_MEM` until a page is released.

### ⏱️ Process Management & Scheduling
- A **16-entry Process Table**, with 16 words per entry. Fields include PID, state, inode index, input buffer, mode flag, user-area page, KPTR, UPTR, PTBR and PTLR.
- Process states in use: `CREATED`, `RUNNING`, `READY`, `TERMINATED`, `WAIT_TERMINAL`, `WAIT_DISK`, `WAIT_MEM`.
- **Round-robin Scheduler (MOD_5)** is invoked from the timer interrupt and from any module that blocks. It saves the kernel stack pointer, PTBR, PTLR and BP, then switches context.
- Separate **user stack and kernel stack** per process; the kernel stack lives in the process's *user area page*.
- **Process Manager (MOD_1)** frees the page table, user-area page and per-process resources when a process exits.

### 🔐 Resource Management
- **Resource Manager (MOD_0)** exposes `Acquire/Release Terminal` and `Acquire Disk`, built on busy-wait loops that call the scheduler (processes sleep until the resource is released).
- Terminal Status Table and Disk Status Table keep track of the process currently holding each device.

### 📟 Device I/O
- **Terminal Write (INT 7)**: serialised through the Device Manager and Resource Manager.
- **Terminal Read (INT 6)**: the reading process sleeps in `WAIT_TERMINAL` until the **console interrupt** delivers the input word into its process-table entry.
- **Disk Load**: interrupt-driven `load` (not polling `loadi`) with the process sleeping in `WAIT_DISK` until the **disk interrupt (INT 2)** wakes it.

### 📦 Program Loader (Exec)
- **INT 9** locates an executable's inode by file name (type `EXEC`) and returns `-1` to the caller if it isn't found.
- It tears down the old address space, reclaims the old user-area page, then rebuilds the library, heap, stack and code pages and loads the code blocks from disk through the Device Manager.
- Initialises the per-process resource table and starts execution at the entry point in the XEXE header.

### 🗂️ Disk & File System (eXpFS)
- 512-block disk image `disk.xfs`, formatted with **XFS-Interface**.
- Inode table (blocks 3–4), disk free list (block 2) and root file (block 5) are examined and manipulated from the host through XFS-Interface.
- The inode table is cached in memory (pages 59–60) at boot for use by Exec.

### 🛠️ Tool Chain
| Tool | Purpose |
|---|---|
| `spl` | Compiles SPL (kernel code) to XSM assembly |
| `expl` | Compiles ExpL (user programs) to XEXE executables |
| `xfs-interface` | Formats the disk and loads the OS, interrupts, modules, library and programs |
| `xsm` | XSM simulator with a built-in debugger (`--debug`) |

---

## 📁 Project Structure

```
myexpos/
│
├── README.md
├── Makefile                      # Builds expl, spl, xfs-interface and xsm in one go
├── download.sh                   # Downloads the eXpOS tool chain (expl, spl, xfs-interface, xsm)
│
├── Stage_02/                     # Filesystem exploration (inode table, root file, sample data)
├── Stage_03/                     # Bootstrap loader — hello world and 1–20 in assembly
├── Stage_04/                     # First SPL programs (odd numbers, sum of squares)
├── Stage_05/                     # Debugging with breakpoints
├── Stage_06/                     # First user process (INIT) and OS startup code
├── Stage_07/                     # ABI-compliant INIT, XEXE header, library, heap, stack
├── Stage_08/                     # Timer interrupt (+ Q1.md worked answers)
├── Stage_09/                     # Kernel stack and process table
├── Stage_10/                     # Console output (INT 7 — first version)
├── Stage_11/                     # Introduction to ExpL
├── Stage_12/                     # Multiprogramming (idle + INIT)
├── Stage_13/                     # Boot module (MOD_7)
├── Stage_14/                     # Round-robin scheduler (MOD_5), INT 10 (Exit)
├── Stage_15/                     # Resource Manager (MOD_0) and Device Manager (MOD_4)
├── Stage_16/                     # Console input (INT 6, console interrupt)
├── Stage_17/                     # Program loader — Exec (INT 9), Memory & Process Managers
├── Stage_18/                     # Disk interrupt handler (INT 2), disk load / acquire disk
│   ├── boot_module.spl
│   ├── device_manager.spl
│   ├── resource_manager.spl
│   ├── int2.spl
│   └── int9.spl
│
│   # Not tracked in git (fetched by download.sh):
├── expl/                         # ExpL compiler, library.lib, sample ExpL programs
├── spl/                          # SPL compiler and spl_progs/
├── xfs-interface/                # Disk formatter / loader (creates disk.xfs)
└── xsm/                          # XSM simulator and debugger
```

> The `Stage_NN/` folders contain **the files changed or added in that stage** (SPL kernel code, ExpL programs and assignments in `AssgN/` sub-folders). The `.gitignore` leaves out the tool-chain folders and compiled binaries, so run `download.sh` after cloning.

### 🗺️ Disk → Memory Layout Loaded at Boot (Stage 18)

| Component | Disk Blocks | Memory Pages |
|---|---|---|
| Exception handler | 15–16 | 2–3 |
| Timer interrupt | 17–18 | 4–5 |
| Disk interrupt (INT 2) | 19–20 | 6–7 |
| Console interrupt | 21–22 | 8–9 |
| INT 6 (Read) | 27–28 | 14–15 |
| INT 7 (Write) | 29–30 | 16–17 |
| INT 9 (Exec) | 33–34 | 20–21 |
| INT 10 (Exit) | 35–36 | 22–23 |
| MOD_0 Resource Manager | 53–54 | 40–41 |
| MOD_1 Process Manager | 55–56 | 42–43 |
| MOD_2 Memory Manager | 57–58 | 44–45 |
| MOD_4 Device Manager | 61–62 | 48–49 |
| MOD_5 Scheduler | 63–64 | 50–51 |
| Inode table (cached) | 3–4 | 59–60 |
| Library | 13–14 | 63–64 |
| INIT program | 7–8 | 65–66 |

---

## 🛠️ Prerequisites

Developed on Linux (Ubuntu / Debian-based distros recommended). Install the dependencies with:

```bash
sudo apt-get install build-essential flex bison libreadline-dev git wget unzip
```

| Tool | Purpose |
|---|---|
| `gcc` / `make` | Build the tool chain |
| `flex`, `bison` | Lexer / parser generators for SPL and ExpL compilers |
| `libreadline-dev` | Interactive prompt in XFS-Interface and the XSM debugger |
| `git`, `wget`, `unzip` | Fetching the repo and the eXpOS tool chain |

---

## 🚀 Getting Started

### 1. Clone the Repository

```bash
git clone https://github.com/Krishna-2105/expOS-NITC.git
cd expOS-NITC
```

If your clone's top-level folder is named `myexpos`, run the commands below from inside it.

### 2. Download the Tool Chain

```bash
./download.sh
```

This fetches `expl`, `spl`, `xfs-interface` and `xsm` from the official [eXpOSNitc](https://github.com/eXpOSNitc) organisation.

### 3. Build Everything

```bash
make          # builds expl, spl, xfs-interface and xsm
make clean    # removes build artefacts
```

### 4. Compile the Kernel Code (SPL → XSM)

```bash
cd spl
./spl spl_progs/os_startup.spl
./spl spl_progs/boot_module.spl
./spl spl_progs/int2.spl            # ...repeat for every .spl file you want to load
```

Copy the `.spl` files of the stage you want to run (e.g. `Stage_18/*.spl`) into `spl/spl_progs/` first.

### 5. Format the Disk and Load the OS

```bash
cd ../xfs-interface
./xfs-interface
```

Inside the XFS-Interface prompt (adjust paths to your files):

```text
fdisk
load --os        ../spl/spl_progs/os_startup.xsm
load --module 7  ../spl/spl_progs/boot_module.xsm
load --module 0  ../spl/spl_progs/resource_manager.xsm
load --module 1  ../spl/spl_progs/process_manager.xsm
load --module 2  ../spl/spl_progs/memory_manager.xsm
load --module 4  ../spl/spl_progs/device_manager.xsm
load --module 5  ../spl/spl_progs/scheduler.xsm
load --int=timer   ../spl/spl_progs/timer.xsm
load --int=disk    ../spl/spl_progs/int2.xsm
load --int=console ../spl/spl_progs/console.xsm
load --int=6       ../spl/spl_progs/int6.xsm
load --int=7       ../spl/spl_progs/int7.xsm
load --int=9       ../spl/spl_progs/int9.xsm
load --int=10      ../spl/spl_progs/int10.xsm
load --exhandler   ../spl/spl_progs/haltprog.xsm
load --library     ../expl/library.lib
load --idle        ../expl/samples/idle.xsm
load --init        ../expl/samples/exec.xsm
load --exec        ../expl/samples/gcd.xsm
exit
```

> 💡 Put these commands in a text file and run them in one go with the `run` command of XFS-Interface (see the [XFS-Interface docs](https://exposnitc.github.io/expos-docs/support-tools/xfs-interface/)).

### 6. Run the Machine

```bash
cd ../xsm
./xsm                 # normal run
./xsm --debug         # run under the XSM debugger
./xsm --timer 0       # disable the timer (useful for early stages)
```

---

## 💻 Usage

### Writing a User Program (ExpL)

```c
int main()
{
decl
    int temp, num;
enddecl
begin
    num = 1;
    while (num <= 50) do
        temp = exposcall("Write", -2, num);   // -2 = terminal
        num = num + 1;
    endwhile;
    return 0;
end
}
```

```bash
cd expl
./expl samples/numbers.expl         # produces samples/numbers.xsm
```

### Reading Input and Launching Other Programs (Exec)

```c
int main()
{
decl
    str a;
    int b;
    str c;
enddecl
begin
    c = "Type fname:";
    b = exposcall("Write", -2, c);
    b = exposcall("Read", -1, a);       // -1 = terminal input
    b = exposcall("Exec", a);           // replace this process with the named executable
    return 0;
end
}
```

### Useful Debugger Commands

```text
reg       # show registers
mem 1     # dump memory page 1 into the file "mem"
pt        # dump the page table of the current process
p         # dump the process table entry of the current process
s         # single step
c         # continue to the next breakpoint
```

Put `breakpoint;` in any SPL or ExpL program and run `./xsm --debug` to inspect the page table, process table, system status table and so on at that point.

> ⚠️ **Note:** User programs may only talk to the kernel through `exposcall()` (system calls). Kernel code written in SPL has no variables, so registers (`R0`–`R19`) are given readable names with `alias`.

---

## 📊 Implementation Stages

**18 of 28 stages completed.** The first twelve are *preparatory*, the next seven are *intermediate*, and the rest are the *final* stages.

### ✅ Completed

| Stage | Title | What Was Implemented |
|---|---|---|
| 1 | Setting up the System | Downloaded and built `expl`, `spl`, `xfs-interface`, `xsm` |
| 2 | Understanding the Filesystem | Formatted `disk.xfs`, loaded data files, inspected the inode table, root file and disk free list |
| 3 | Bootstrap Loader | OS startup code in assembly loaded into block 0 (hello world, print 1–20) |
| 4 | Learning the SPL Language | SPL programs compiled with `spl` (odd numbers, sum of squares) |
| 5 | XSM Debugging | Breakpoints and the XSM debugger (`reg`, `mem`, step / continue) |
| 6 | Running a User Program | First user process: INIT, page table setup, `ireturn` into user mode, halting INT 10 |
| 7 | ABI and XEXE Format | XEXE header, library pages 63–64, heap and stack pages, PTLR = 10 |
| 8 | Handling Timer Interrupt | Timer ISR loaded at page 4–5 and run with `./xsm --timer` |
| 9 | Handling Kernel Stack | Process table entry, user area page, user / kernel stack switch, `backup` / `restore` |
| 10 | Console Output | `INT 7` handler for terminal write with manual logical → physical address translation |
| 11 | Introduction to ExpL | ExpL programs using `exposcall("Write", ...)`, user-defined types |
| 12 | Introduction to Multiprogramming | Idle process, PID / state fields, timer-based switching between two processes |
| 13 | Boot Module | Module 7 initialises data structures; idle process runs first |
| 14 | Round Robin Scheduler | Scheduler module (MOD_5), `INT 10` terminates one process and halts only when all are done |
| 15 | Resource Manager Module | MOD_0 (Acquire/Release Terminal) and MOD_4 (Terminal Write); INT 7 now goes through the modules |
| 16 | Console Input | `INT 6` Terminal Read, console interrupt handler, `WAIT_TERMINAL` blocking (`gcd.expl`, `bubble.expl`) |
| 17 | Program Loader | `INT 9` (Exec), Memory Manager (get / release page), Process Manager (free page table, user area, exit process) |
| 18 | Disk Interrupt Handler | `INT 2`, interrupt-driven disk load via MOD_4, Acquire Disk in MOD_0, `WAIT_DISK` state; Exec now loads code through the Device Manager |

### 🔜 Planned (Intermediate & Final Stages)

| Stage | Title | Est. Time | Planned Scope |
|---|---|---|---|
| 19 | Exception Handler | 6 hrs | Handle illegal instructions, illegal memory access and page faults; terminate the faulting process safely |
| 20 | Process Creation and Termination | 12 hrs | **Fork**, final **Exec**, and **Exit** — process creation, address-space copy, termination with cleanup |
| 21 | Process Synchronization | 4 hrs | **Wait**, **Signal**, **Getpid**, **Getppid** — blocking and waking processes |
| 22 | Semaphores | 4 hrs | **Semget**, **Semrelease**, **Semlock**, **Semunlock** — kernel-managed semaphore table |
| 23 | File Creation and Deletion | 6 hrs | **Create** and **Delete** system calls on eXpFS (inode table, root file, disk free list) |
| 24 | File Read | 12 hrs | **Open**, **Close** and **Read** — open file table, per-process resource table, buffer cache |
| 25 | File Write | 12 hrs | **Write** to files (with block allocation) and **Seek** |

### 🗺️ Remaining Roadmap (not yet scheduled)

| Stage | Title | Est. Time |
|---|---|---|
| 26 | User Management | 12 hrs |
| 27 | Pager Module (virtual memory / swapping) | 18 hrs |
| 28 | Multi-Core Extension (NEXSM) | 12 hrs |

---

## 📖 Documentation

- 📚 [eXpOS Official Documentation](https://exposnitc.github.io/expos-docs/)
- 🗺️ [Roadmap](https://exposnitc.github.io/expos-docs/roadmap/)
- 🧱 [OS Design](https://exposnitc.github.io/expos-docs/os-design/)
- 🧩 [Kernel Modules](https://exposnitc.github.io/expos-docs/modules/)
- ⚙️ [XSM Architecture Specification](https://exposnitc.github.io/expos-docs/arch-spec/)
- 🔌 [Application Binary Interface (ABI)](https://exposnitc.github.io/expos-docs/abi/)
- 🧰 [Support Tools: XFS-Interface, SPL, ExpL, XSM](https://exposnitc.github.io/expos-docs/support-tools/xfs-interface/)

---

## 🤝 Contributing

1. Fork the repository
2. Create a new branch: `git checkout -b feature/your-feature`
3. Commit your changes: `git commit -m "Add: short description"`
4. Push to the branch: `git push origin feature/your-feature`
5. Open a Pull Request with a clear description

Please keep each stage in its own `Stage_NN/` folder, and keep kernel code in SPL modules (invoked only from interrupt handlers or other modules) rather than duplicating logic across handlers.

---

<p align="center">
  Built with 💙 at <strong>NIT Calicut</strong> &nbsp;|&nbsp;
  <a href="https://github.com/Krishna-2105/expOS-NITC/issues">Report an Issue</a> &nbsp;|&nbsp;
  <a href="https://exposnitc.github.io/expos-docs/">Official Docs</a>
</p>
