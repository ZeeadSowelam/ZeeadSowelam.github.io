---
layout: page
title: RISC-V Operating System Kernel
description: A preemptive multitasking kernel for 64-bit RISC-V, built from bare metal in C and assembly. Interrupt controllers and device drivers up through virtual memory, a block cache, a filesystem, and a user-space shell.
img: assets/img/oskernel_read_path.svg
importance: 2
category: systems
related_publications: false
---

This was the systems-engineering course (ECE 391), and the point of it was to build a Unix-like kernel for RISC-V with nothing underneath you. It boots on emulated hardware under QEMU, schedules threads and processes preemptively, and exposes a system-call interface backed by virtual memory, a block cache, a filesystem, and a full device-driver stack. The way I think about the whole thing is one user program calling `read()`: that call crosses the syscall boundary, the I/O layer, the filesystem, the block cache, and a device driver before any bytes come back, and on the version I worked on every one of those layers was code I wrote.

The work came in two phases with very different flavors. The first was solo and lived close to the silicon. The second was a three-person team building the abstractions that sit on top.

## Phase one: bring up the machine yourself

The first phase had no libc, no `printf`, no console, nothing, until you make it exist. You start with a processor and some memory-mapped device registers and that is the entire world. I implemented the PLIC interrupt controller and the interrupt manager, an NS16550 UART driver for an interrupt-driven console, a Goldfish RTC driver, and the VirtIO entropy and block drivers down at the virtqueue level. On top of that came a cooperative threading system (spawn, exit, join, with the context switch written register by register in assembly), condition variables, reader-writer locks, and a hardware timer that let threads sleep, which I later extended into real preemption.

Bringing up the UART is the first thing that makes the machine feel real, because until it works your bugs are invisible. You cannot poke at a running program when there is no way for it to tell you anything. You configure the divisor latch for the baud rate, set the line for 8 data bits with no parity and one stop bit, turn on the FIFO, and only then can a byte you print actually leave the chip.

```c
// NS16550 UART bring-up: 115200 8N1, FIFO on, RX interrupts enabled.
#define UART0      0x10000000UL
#define REG(off)  (*(volatile uint8_t *)(UART0 + (off)))
// register offsets: DLL=0, IER/DLM=1, FCR=2, LCR=3
void uart_init(void) {
    REG(1) = 0x00;   // mask UART interrupts while we configure
    REG(3) = 0x80;   // LCR: set DLAB to reach the divisor latch
    REG(0) = 0x01;   // divisor low  (input clock / (16 * baud))
    REG(1) = 0x00;   // divisor high
    REG(3) = 0x03;   // LCR: clear DLAB, 8 data bits, no parity, 1 stop
    REG(2) = 0x07;   // FCR: enable FIFO, clear RX/TX queues
    REG(1) = 0x01;   // IER: raise an interrupt when a byte arrives
}
```

When something broke in this phase the only tool was to put prints everywhere and watch for which one came out last. Most of the time you cannot poke around at all, so you reason instead about what each register should hold and check whether it actually does.

## Phase two: stack the abstractions

In the second phase I owned virtual memory, the whole block cache, the system-call interface, and the NGFS file operations (NGFS was shared with a teammate, Henry, rather than solo). Virtual memory meant Sv39, which is the 64-bit RISC-V scheme with three levels of page table and 4 KiB pages. A virtual address splits into three 9-bit indices plus a page offset, and translating it means walking three tables, allocating the intermediate ones the first time a process touches an address.

```c
// Sv39: walk the 3-level page table for vaddr, allocating tables on demand.
static pte_t *walk(pte_t *root, uintptr_t vaddr, int alloc) {
    pte_t *table = root;
    for (int level = 2; level > 0; level--) {
        uint64_t vpn = (vaddr >> (12 + 9 * level)) & 0x1ff;
        if (table[vpn] & PTE_V) {
            table = (pte_t *)((table[vpn] >> 10) << 12);  // next table PPN
        } else {
            if (!alloc) return NULL;                       // not mapped
            pte_t *next = alloc_page();                    // zeroed 4 KiB page
            table[vpn] = (((uintptr_t)next >> 12) << 10) | PTE_V;
            table = next;
        }
    }
    return &table[(vaddr >> 12) & 0x1ff];                  // leaf PTE
}
```

Below the filesystem sits a block cache over two kinds of block device: the VirtIO disk and an in-memory (memio) device. The filesystem is a FAT-style layout (NGFS) reached through a unified I/O abstraction, so `open`, `read`, `write`, `seek`, and `ioctl` look the same to a program whether the bytes live on disk or in memory. The diagram below is the path a single `read()` takes through all of it.

<div class="row justify-content-sm-center">
    <div class="col-sm-9 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/oskernel_read_path.svg" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    The route one user-space read() takes from the syscall trap down to the disk and back. Drawn by hand on purpose, because the point is the layering, not the picture.
</div>

## The bug that taught me the most

The hardest bug of the project was a one-line null-pointer dereference, and it was hard precisely because everything about the environment lied to me about where it was. The symptom from the autograder was a flat 120-second timeout on `test_fork_simple`. No panic message, no stack trace, no log output, just silence for two minutes and then a fail.

It did not reproduce locally. My own fork tests passed cleanly every run. The bug only fired because the grader calls `fork` from the kernel's `main_proc` context, where `proc->exeio` is NULL, so `ioaddref(proc->exeio)` walked off a null pointer and the kernel stopped in its panic handler. The reason it looked like a hang instead of a crash is that the panic printed to UART0 while the grader was only watching UART1 and UART2, so the crash output went somewhere nobody was reading. The fix was a single NULL guard. Finding it meant reasoning about a machine I could not directly observe, with a limited number of grader runs and no log capture, and accepting that my local success was not evidence of correctness but a thing actively pointing me the wrong way.

That pattern showed up across the project. Because every test ran the whole OS end to end, a lot of the bugs did not belong to any one person's file; they only appeared when all the layers ran together. I did a fair amount of that cross-cutting debugging, and the habit it forced was to confirm a root cause with instrumented logging before touching anything, since guessing across that many layers is how you spend a day fixing the wrong thing.

## Components, briefly

- Interrupts and drivers: RISC-V trap handling, PLIC routing, interrupt-driven UART, RTC, and VirtIO block and entropy over the virtqueue protocol.
- Concurrency: cooperative threads with a ready-list scheduler, assembly context switching, condition variables, reader-writer locks, timer-driven sleep, and preemption.
- Virtual memory: Sv39 page tables, per-process address spaces, and page-fault handling.
- Storage: a block cache over VirtIO and memio devices beneath the NGFS filesystem and its unified I/O layer.
- Processes: fork, the syscall interface, pipes, and preemptive scheduling.

<hr>

Built in C and RISC-V assembly on QEMU with VirtIO devices, GCC, and GDB. The source is a graded course solution, so it stays private, but I am happy to walk through the full code on request.
