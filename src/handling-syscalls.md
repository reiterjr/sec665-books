# Handling syscalls

## Discovering the syscall handler

From the previous chapter, [The SYSCALL instruction](./syscall-instruction.md), we saw how to obtain the handler that services all syscalls coming from userland. Since we still have a kernel debugging session up and running, we can disassemble the function and also set a breakpoint on it.

## Breakpoints

The first instinct might have you set a BP right at the function itself like so: **`bp nt!KiSystemCall64`**. Not a bad idea at first and I did this as well. After BSODing my VM a few times after setting that BP, I realized that this isn't your normal function with a proper function prolog. Typically, a proper function prolog will set up the stack frame, make sure `RSP` is taken care of before execution of that function proceeds.

The first instruction here is a **`SWAPGS`** which is insanely critical for syscall handling and is also the reason the VM becomes unstable and ultimately BSODs when setting a BP on that instruction. After a bit more experimentation, I found that setting a BP at **`nt!KiSystemCall64+15h`** was more stable and reliable.

Let's look at the first several instructions for the syscall handler on this VM.

```text
nt!KiSystemCall64:
0f01f8                         swapgs      // BP here crashes the VM
654889242510000000             mov     qword ptr gs:[10h], rsp
65488b2425a8010000             mov     rsp, qword ptr gs:[1A8h]
6a2b                           push    2Bh  // BP here, all is good!
```

## Setting up `RSP`

If you look at the disassembly, you will see that `RSP` isn't properly set up until the second `MOV` instruction: **`MOV RSP, QWORD PTR GS:[1A8h]`**. This is why setting a BP any time before, or even on that instruction, creates instability. At the first `MOV` instruction, `RSP` will still be pointing to a userland address. This is not good now that we are in the kernel with CPL0. The offsets you see for the GS segment register won't really make sense yet. For that, I will dedicate a chapter for the **SWAPGS** instruction next.
