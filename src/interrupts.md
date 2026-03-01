# Interrupts

## What is INT 2Eh?

The **INT** instruction invokes something called an **interrupt** that kind of "halts" the system to "wake up" the kernel's interrupt handler routine. The value given to the instruction is an index into a table called the **Interrupt Descriptor Table**, or **IDT**. Every system is going to have one of these tables and you can see them with a kernel debugging session. Here is a dump of the IDT on a Windows 10 Dev-VM built for SEC670.

```text
0: kd> !idt
Dumping IDT: fffff804774f2000
00:    fffff80474a13800 nt!KiDivideErrorFault
[..SNIP..]
03:    fffff80474a14500 nt!KiBreakpointTrap
[..SNIP..]
1f:    fffff80474a0ce40 nt!KiApcInterrupt
[..SNIP..]
2f:    fffff80474a0efe0 nt!KiDpcInterrupt
```

There are some interesting ones there in this snippet like dividing by 0, a breakpoint getting hit, APC interrupts, DPC interrupts. Whenever one of those events happen, like a divide by 0, an interrupt will be triggered and INT 00h will be the underlying entry for that. You can also see the address of the routine that will handle an interrupt. Back in the day, people loved to hook entries in the IDT, but that's not really a thing anymore.

What's missing from the above snippet is the entry **2Eh**. This one is missing because even though Hyper-V is enabled, VBS is not. Thus, there will be no corresponding entry for it built into the table. It will never get hit and there's no need to have a handler routine for it. Simple as that.

## Building the IDT

The IDT is built when the kernel is being initialized in the routine **`nt!KiInitializeKernel`**. Also in that massive routine, the `_KUSER_SHARED_DATA` structure is filled out by the **`nt!KiInitializeBootStructures`** routine.
