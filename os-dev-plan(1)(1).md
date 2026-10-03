# Build Your Own OS — Day-by-Day Plan

**Target:** x86_64, using the Limine bootloader (Multiboot2/Limine protocol) + C, with small amounts of inline/standalone ASM where unavoidable.
**Pace:** ~1–2 hrs/session, 3–5 sessions/week. Each "Day" = one sitting. Adjust freely — the point is small, testable steps. Plan now runs to Day 75, ending with a real, working text editor (nano, with Vim as a stretch goal) running on your OS.
**Tools you'll set up once:** cross-compiler (`x86_64-elf-gcc`), `nasm`, `qemu-system-x86_64`, `make`, `git`, `xorriso` (for ISO creation), Limine.

## Languages used, by phase

Short answer: **it's almost all C.** Assembly shows up only in a handful of small, specific spots where C literally cannot do the job (loading special CPU registers, saving/restoring full register state on a context switch). Nowhere in this plan do you write more than a page of ASM in a single sitting — it's usually 2–20 lines per occurrence.

| Phase | Primary language | Assembly needed | Notes |
|---|---|---|---|
| 0 — Toolchain | Build config only (no app code) | None | You're compiling *a* compiler, not writing OS code yet. |
| 1 — Kernel Boots & Talks | **C** | A few lines: `lgdt`, `outb`/`inb` (Day 7, 9) | ASM is 1–2 line inline snippets, wrapped in C functions. |
| 2 — Interrupts & Input | **C** | ISR entry stubs (Day 12) — ~10–20 lines total, boilerplate you write once and reuse | The stubs just save registers and jump into C; all actual handling logic is C. |
| 3 — Memory Management | **C** | `lcr3`/reading `cr2` (1–2 lines, Day 22 & 26) | Paging logic, allocators, VMM — all C. |
| 4 — Multitasking | **C**, + ASM for one function | `switch_context()` (Day 30) is the single largest ASM routine in the whole plan — maybe 20–30 lines. `iretq` for ring 3 entry (Day 37) is a few lines. | This phase has the most ASM of any phase, but it's still small and written once. |
| 5 — Storage & Filesystem | **C** | None beyond what you already wrote | Disk drivers, FAT32, ELF loader — all C. |
| 6 — Shell & Usability | **C** | None | Shell, parsing, your mini-libc — all C. |
| 7 — POSIX Layer & Editor | **C** (yours) + **C** (nano/vim's own unmodified source, which you don't write, just compile) | None new — mlibc itself has some ASM internally (syscall entry trampolines), but that's code you vendor, not code you write | You're plumbing your existing C syscalls to match POSIX signatures. Nano and Vim are themselves written entirely in C, so nothing about porting them requires you to write ASM. |

**Bottom line:** across the whole 75-day plan, the ASM you personally write totals well under 100 lines, concentrated in Phases 1, 2, and 4. Everything else — kernel, drivers, memory manager, scheduler, filesystem, shell, and the POSIX plumbing for nano/vim — is C.

## Libraries / headers needed, by day

Almost everything here is **freestanding C standard headers** (the handful that don't assume an OS underneath — you get these for free with your cross-compiler) plus **Limine's own header** for talking to the bootloader. Nothing else is a "library" to learn until Phase 7, where you bring in real POSIX headers and mlibc.

| Day(s) | Headers / Libraries | Notes |
|---|---|---|
| 1–3 | *(none — tooling only)* | binutils, gcc, nasm, qemu, xorriso, Limine are tools you install, not APIs you learn. |
| 4 | `stdint.h`, `stddef.h`, `limine.h` | Freestanding headers (fixed-width ints, `NULL`/`size_t`); `limine.h` is Limine's protocol header, drop it into your source tree. |
| 5 | `stdint.h`, `limine.h` | Framebuffer request struct comes from `limine.h`. |
| 6 | `stdint.h` | Bitmap font is a plain array you embed — no header for it. |
| 7 | `stdint.h` | Port I/O (`outb`/`inb`) is 2-line inline ASM wrapped in a C function — no header needed. |
| 8 | `stdarg.h` | Needed for `va_list`/`va_start`/`va_arg` to write a variadic `kprintf`. |
| 9 | `stdint.h` | |
| 10 | *(none)* | Refactor/cleanup day. |
| 11 | `stdint.h` | |
| 12 | `stdint.h` | ISR stubs are ASM; the common handler they call is plain C. |
| 13 | `stdint.h` | |
| 14 | `stdint.h` | |
| 15 | `stdint.h` | Scancode→ASCII table is a plain array. |
| 16 | *(none new)* | Reuses what you've already written. |
| 17 | `stdint.h`, `limine.h` | Memory map comes from a Limine request. |
| 18 | *(none)* | Review day. |
| 19 | `stdint.h`, `stddef.h` | `size_t` for tracking page counts. |
| 20 | *(none — reading/diagramming day)* | |
| 21 | `stdint.h` | |
| 22 | `stdint.h` | |
| 23 | `stdint.h` | |
| 24 | `stdint.h`, `stddef.h` | Writing your own `kmalloc`/`kfree` — no `stdlib.h` yet, you *are* the allocator. |
| 25 | *(none new)* | |
| 26 | `stdint.h` | |
| 27 | *(none new)* | |
| 28 | *(none)* | Review day. |
| 29 | `stdint.h`, `stddef.h` | |
| 30 | *(none — pure ASM)* | |
| 31–34 | *(none new)* | |
| 35 | `stdatomic.h` | C11 atomics are handy for a correct spinlock (`atomic_flag`); optional but recommended over hand-rolled ASM `lock` prefixes. |
| 36–39 | `stdint.h` | |
| 40 | *(none)* | Review day. |
| 41 | `stdint.h` | PCI/AHCI register layouts are structs you define yourself, informed by the PCI/AHCI spec docs (not a header you import). |
| 42 | `stdint.h` | |
| 43 | `stdint.h` | GPT/MBR structs use `__attribute__((packed))` — a compiler attribute, not a library. |
| 44 | `stddef.h` | Function-pointer-based VFS interface. |
| 45–46 | `stdint.h`, `string.h` | You'll want your own `string.h` implementation (`memcpy`, `memset`, `strcmp`, etc.) by this point — a small, common early task. |
| 47 | `string.h` | |
| 48–49 | *(none new)* | |
| 50 | `stdint.h`, `elf.h` | `elf.h` defines standard ELF structures/constants — you can write a minimal one yourself from the ELF spec, or vendor a public-domain one. |
| 51–52 | *(none new)* | |
| 53 | `string.h` | |
| 54 | `string.h`, `ctype.h` | `ctype.h` for `isspace`/`isalpha` etc. in your command tokenizer — easy to hand-write the few functions you need if you'd rather not pull in the whole header. |
| 55 | *(none new)* | |
| 56 | `string.h`, `stdarg.h` | Your mini freestanding libc — wraps your own syscalls, not real POSIX yet. |
| 57–60 | *(none new)* | |
| 61 | **mlibc** (external library) | This is the first real third-party library in the whole plan — a libc written specifically to be portable to custom kernels. |
| 62 | `sys/stat.h`, `unistd.h`, `fcntl.h` | Standard POSIX headers whose *declarations* mlibc provides — you're implementing the syscalls behind them. |
| 63 | `termios.h`, `sys/ioctl.h` | |
| 64 | `signal.h` | |
| 65 | `sys/mman.h` | |
| 66 | *(none new — you hand-write a minimal terminfo entry)* | Not pulling in full ncurses/terminfo database, just faking one entry. |
| 67–70 | nano's own source tree (unmodified) | You're not "learning" a library here so much as satisfying nano's existing `configure` script — it'll tell you what's missing as you go. |
| 71–73 | *(none new)* | |
| 74–75 | vim's own source tree (unmodified); relies more heavily on `regex.h` (POSIX regex) internally | Same iterative build-and-fix process as nano, just more of it. |

---

## Phase 0 — Environment & Toolchain (Days 1–3)
*Language: build/config only, no OS code yet*

**Day 1: Build a cross-compiler**
- Build `binutils` + `gcc` targeting `x86_64-elf` (do NOT use your host compiler — it assumes a hosted OS).
- Verify with `x86_64-elf-gcc --version`.
- Goal: toolchain builds cleanly, no OS knowledge needed yet.

**Day 2: Get QEMU running**
- Install `qemu-system-x86_64`.
- Boot a stock ISO (e.g. a Linux live CD) in QEMU just to confirm your virtualization setup works.
- Install `nasm` and `xorriso`/`mtools`.

**Day 3: Hello, bootloader**
- Download Limine, follow its "barebones" template.
- Write a linker script (`linker.ld`) placing your kernel at the higher-half address (`0xffffffff80000000`).
- Build a kernel that does nothing but halt (`while(1) { asm("hlt"); }`).
- Boot it in QEMU. Success = black screen, no crash, no reboot loop.

---

## Phase 1 — Minimal Kernel Boots & Talks (Days 4–10)
*Language: C, with a couple of 1–2 line inline ASM snippets*

**Day 4: Kernel entry point**
- Write `_start` in C, called from Limine.
- Confirm `main()` is reached (set a breakpoint in QEMU+GDB, or write to a known memory address and inspect it).

**Day 5: VGA/framebuffer text output**
- Use the Limine framebuffer request to get a pointer to video memory.
- Write a `putpixel()` function.
- Draw a solid color to the screen — your first visible "hello world."

**Day 6: Basic font rendering**
- Embed a simple 8x8 bitmap font (many are public domain).
- Write `putchar()` and `puts()` that draw text via the framebuffer.
- Print "Hello, kernel!" on boot.

**Day 7: Serial port output**
- Implement `outb`/`inb` (inline ASM, 2 lines each).
- Write to COM1 (0x3F8) so you can log debug output to your terminal via `qemu -serial stdio`. This becomes your best debugging tool from here on.

**Day 8: printf-style logging**
- Write a minimal `kprintf` supporting `%d`, `%x`, `%s`, `%c` — no need for full libc.
- Route it to both screen and serial.

**Day 9: GDT (Global Descriptor Table)**
- Set up a flat GDT (kernel code/data segments).
- Load it with `lgdt` (ASM), understand why it's needed even though paging does most of the work in long mode.

**Day 10: Review & cleanup**
- Refactor into folders: `kernel/`, `drivers/`, `arch/x86_64/`.
- Set up a `Makefile` that rebuilds + boots QEMU in one command.
- Commit to git. Tag this as `v0.1-boots`.

---

## Phase 2 — Interrupts & Input (Days 11–18)
*Language: C, with a small block of boilerplate ASM (ISR stubs, written once)*

**Day 11: IDT (Interrupt Descriptor Table)**
- Define the IDT structure, write `lidt` loader.
- Register a dummy handler for interrupt 0 (divide-by-zero) as a test.

**Day 12: ISR stubs**
- Write ASM stubs for the first 32 CPU exceptions, pushing state and calling a common C handler.
- Trigger divide-by-zero deliberately, confirm your handler catches it and prints a message instead of crashing QEMU silently.

**Day 13: PIC remapping**
- Remap the legacy 8259 PIC so hardware IRQs don't collide with CPU exception vectors.

**Day 14: Timer interrupt (IRQ0/PIT)**
- Handle the Programmable Interval Timer interrupt.
- Increment a global tick counter — this becomes your system clock.

**Day 15: Keyboard driver (IRQ1)**
- Read scancodes from port 0x60 on keyboard interrupt.
- Translate scancode → ASCII (US layout table).
- Print typed characters to screen.

**Day 16: Basic input line buffer**
- Buffer keystrokes into a line, handle backspace/enter.
- This is the seed of a future shell.

**Day 17: Physical memory detection**
- Parse the memory map Limine provides.
- Print available RAM regions via kprintf.

**Day 18: Review & cleanup**
- Tag `v0.2-interrupts`. Write a short README describing what works.

---

## Phase 3 — Memory Management (Days 19–28)
*Language: C, with 1–2 line inline ASM for special registers (`cr3`, `cr2`)*

**Day 19: Physical memory manager (PMM) — bitmap allocator**
- Track free/used physical pages (4KB each) with a bitmap.
- Implement `pmm_alloc_page()` / `pmm_free_page()`.

**Day 20: Understand paging**
- Read up on x86_64 4-level paging (PML4 → PDPT → PD → PT).
- No code yet — just diagram it out; this is the trickiest concept so far.

**Day 21: Set up your own page tables**
- Build a fresh set of page tables (don't rely solely on the bootloader's).
- Identity-map low memory + higher-half map the kernel.

**Day 22: Load CR3, enable your paging**
- Switch to your page tables.
- If it doesn't triple-fault, you've succeeded. (It probably will the first time — that's normal.)

**Day 23: Virtual memory manager (VMM)**
- Write `vmm_map(virt, phys, flags)` / `vmm_unmap(virt)`.
- Test by mapping a new page and writing/reading from it.

**Day 24: Kernel heap — simple allocator**
- Implement a basic `kmalloc`/`kfree` (bump allocator first, freelist later).
- Test with repeated alloc/free cycles.

**Day 25: Improve the heap allocator**
- Upgrade to a simple freelist or slab-style allocator to reduce fragmentation.

**Day 26: Page fault handler**
- Handle interrupt 14 (#PF), print the faulting address (from CR2) and reason.
- Deliberately fault to test it.

**Day 27: Guard pages & stack safety**
- Add unmapped guard pages below kernel stacks to catch overflows cleanly.

**Day 28: Review & cleanup**
- Tag `v0.3-memory`. This is a major milestone — most hobby OS projects stall before this point.

---

## Phase 4 — Multitasking (Days 29–40)
*Language: C, plus this plan's largest ASM routine (`switch_context`, ~20–30 lines) and a few lines for `iretq`*

**Day 29: Task/process data structure**
- Define a `task_t` struct: registers, page table pointer (CR3), stack, state.

**Day 30: Context switching (ASM)**
- Write `switch_context(old, new)` in ASM: save/restore general registers, stack pointer, instruction pointer.

**Day 31: Cooperative scheduling**
- Create 2 dummy tasks that just `kprintf` in a loop and voluntarily `yield()`.
- Confirm both run and interleave.

**Day 32: Preemptive scheduling**
- Trigger `schedule()` from the timer interrupt (IRQ0) instead of manual yield.
- Round-robin between tasks.

**Day 33: Kernel stacks per task**
- Give each task its own kernel stack, allocated via your PMM/VMM.

**Day 34: Task states & termination**
- Add states: READY, RUNNING, BLOCKED, TERMINATED.
- Implement `exit()` and reap terminated tasks.

**Day 35: Simple synchronization primitives**
- Implement a spinlock.
- Test with two tasks incrementing a shared counter.

**Day 36: Userspace vs kernel space concept**
- Set up separate GDT entries for user code/data segments (ring 3).
- Don't jump to userspace yet — just prepare the segments.

**Day 37: Enter ring 3**
- Use `iretq` to jump into a minimal userspace function.
- Confirm privilege level actually changed (try executing a privileged instruction and watch it fault).

**Day 38: System call interface**
- Implement `syscall`/`sysret` (or interrupt-based syscalls as a simpler first step).
- Add one syscall: `sys_write` (print a string from userspace).

**Day 39: Load a real user program**
- Compile a tiny freestanding user binary, load it into a process's address space, jump to it.

**Day 40: Review & cleanup**
- Tag `v0.4-multitasking`. Huge milestone — you now have a real, if primitive, OS.

---

## Phase 5 — Storage & Filesystem (Days 41–52)
*Language: 100% C*

**Day 41: ATA/AHCI disk driver basics**
- Detect disks via PCI enumeration.
- Read raw sectors via AHCI (or ATA PIO for simplicity first).

**Day 42: Disk read/write test**
- Read sector 0 (should be your boot sector/GPT), print bytes via kprintf.

**Day 43: GPT/MBR parsing**
- Parse partition tables to find your data partition.

**Day 44: VFS (virtual filesystem) abstraction layer**
- Define generic `open/read/write/close` function pointers so you can plug in real filesystems later.

**Day 45–46: Implement a simple filesystem (or FAT32 reader)**
- FAT32 is the standard hobby-OS choice — well documented, tooling everywhere.
- Start read-only: list root directory, read a file's contents.

**Day 47: File write support**
- Implement file creation/writing within FAT32 (harder — cluster chain management).

**Day 48: Wire VFS into syscalls**
- Add `sys_open`, `sys_read`, `sys_write`, `sys_close` for userspace programs.

**Day 49: Load programs from disk**
- Replace your embedded test binary with a real ELF loader that reads a program from the filesystem.

**Day 50: Minimal ELF loader**
- Parse ELF headers, map segments, jump to entry point.

**Day 51: Test full flow**
- Compile a userspace "hello world," put it on the disk image, load and run it from your OS.

**Day 52: Review & cleanup**
- Tag `v0.5-filesystem`.

---

## Phase 6 — Shell & Usability (Days 53–60)
*Language: 100% C*

**Day 53: Basic shell process**
- A userspace program that reads keyboard input (via syscall) and prints a prompt.

**Day 54: Command parsing**
- Tokenize input, implement built-ins: `echo`, `ls`, `clear`.

**Day 55: Process launching from shell**
- Implement `sys_exec` — shell forks/execs a program by name from disk.

**Day 56: Basic libc**
- Write a tiny freestanding libc (`string.h` functions, `malloc` wrapping your syscalls) so userspace programs aren't hand-rolling everything.

**Day 57: Pipes or simple IPC (optional)**
- Basic mechanism for one process to send data to another.

**Day 58: Terminal improvements**
- Scrolling text, better font/framebuffer console.

**Day 59: Bug hunt / stability pass**
- Deliberately stress test: spawn many processes, allocate/free heavily, fill the disk.

**Day 60: Review & cleanup**
- Tag `v1.0`. You have a bootable, interactive, multitasking OS with a filesystem and shell.

---

## Phase 7 — POSIX Layer & a Real Text Editor (Days 61–75)
*Language: 100% C — nano and vim are themselves C programs; you're only wiring up your existing C syscalls to match POSIX*

Real editors like **nano** and **vim** aren't written to be portable to bare-metal — they assume a POSIX environment (real libc, termios, proper file I/O). This phase builds that bridge. **Nano is the recommended target** — far fewer dependencies than Vim (no built-in regex engine dependency the way Vim has, simpler terminal handling, smaller codebase), so you'll hit success sooner. Vim is included as a stretch goal at the end for the same reason it's a well-known OSDev "flex" milestone.

**Day 61: Pick and vendor a portable libc**
- Use **mlibc** (the standard hobby-OS choice — designed to be ported to custom kernels) rather than writing your own POSIX layer from scratch.
- Set up mlibc's build system against your toolchain. Don't wire it to real syscalls yet — just get it to compile with stub syscalls.

**Day 62: Map your syscalls to POSIX numbers**
- Go from your ~5 custom syscalls to the POSIX set nano/vim actually need: `open`, `read`, `write`, `close`, `lseek`, `stat`/`fstat`, `mmap` (or a stub), `brk`/`sbrk`, `exit`, `fork` or `posix_spawn`-equivalent, `ioctl`.
- Wire each one into your existing VFS/PMM/scheduler — this is mostly plumbing, not new concepts.

**Day 63: Implement `ioctl` and raw terminal mode**
- Editors need to disable line-buffering and echo, and capture every keystroke (including Ctrl combos) directly.
- Implement `TCGETS`/`TCSETS`-equivalent `ioctl` calls so `termios` can flip your tty between "cooked" (shell) and "raw" (editor) mode.

**Day 64: Terminal resize & signals**
- Implement minimal signal delivery: `SIGINT` (Ctrl+C), `SIGWINCH` (resize) at minimum.
- Fake a fixed terminal size if you don't want to support live resize yet — hardcode 80x25 and report it via `ioctl(TIOCGWINSZ)`.

**Day 65: `mmap` support (even if limited)**
- Many programs (including nano/vim) call `mmap` for reading files or allocating memory. A minimal anonymous-mapping implementation backed by your VMM is enough — you don't need full file-backed mmap yet.

**Day 66: Minimal terminfo/termcap**
- Editors query terminal capabilities (cursor movement, clear screen, colors) via terminfo.
- Ship a single hardcoded entry mimicking a basic `xterm` or `vt100` so `TERM=vt100` "just works" — no need for the full ncurses database.

**Day 67–69: Cross-compile nano against mlibc**
- Fetch nano's source, configure it to build against your toolchain + mlibc.
- Expect this to be iterative: build, hit a missing syscall or header, implement/stub it, rebuild, repeat.
- Budget 3 sessions — this is where most of Phase 7's real debugging happens.

**Day 70: Get nano running and usable**
- Copy the compiled binary onto your disk image via your filesystem tooling.
- Launch it from your shell, open a file, edit it, save it (`Ctrl+O`), quit (`Ctrl+X`).
- This is the milestone: a real, unmodified text editor running on your OS.

**Day 71: Fix rough edges**
- Common issues at this stage: cursor position drift, backspace/delete key mapping, missing `Ctrl+`-key handling in your keyboard driver, screen redraw glitches. Iterate until it's solid enough to trust with real edits.

**Day 72: Wire nano into your shell workflow**
- Confirm `$EDITOR`-style invocation works if your shell supports it, and that exiting nano correctly returns you to the shell prompt with terminal mode restored to "cooked."

**Day 73: Review & cleanup**
- Tag `v1.1-editor`. You now have a usable text editor — a genuine milestone very few hobby OS projects reach.

**Day 74–75: Stretch goal — port real Vim**
- Same process as Days 67–70, but expect more missing-symbol iterations: Vim links against more of libc (`regex.h`, more elaborate `stdio` usage) and needs a fuller terminfo entry for its screen drawing.
- Skip Neovim specifically — it hard-depends on LuaJIT, which means porting a JIT compiler first; that's a separate, much larger project on its own.
- If you get real Vim's normal/insert mode and `:w`/`:q` working, that's parity with several established hobby OSes (SerenityOS, ToaruOS) that list this as a headline achievement.

---

## Beyond Day 75 — Where to go next
Pick based on interest:
- **Networking:** NIC driver (e.g. RTL8139/e1000) → basic TCP/IP stack
- **GUI:** window manager on top of your framebuffer driver
- **SMP:** multi-core support (APIC, per-CPU scheduling)
- **POSIX-ish compatibility:** port a real libc (e.g. mlibc) to run existing software
- **Better filesystem:** journaling, or your own custom FS design

---

## Practical tips throughout
- **Commit after every working day.** OS dev breaks in subtle ways; small commits let you bisect.
- **Use QEMU's `-d int,cpu_reset` and GDB stub (`-s -S`)** liberally — triple faults are your most common bug and they give almost no error message otherwise.
- **Join the OSDev community** (osdev.org wiki + forum) — nearly every bug you'll hit has been hit before.
- **Don't aim for correctness on first pass.** Bump allocators, round-robin scheduling, and read-only filesystems are all fine "good enough for now" stand-ins — you upgrade them once the rest of the system depends on them working.
