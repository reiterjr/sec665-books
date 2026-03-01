# What is a direct syscall?

Over in [The flow to a syscall](./flow-to-syscall.md), you see what are viewed as wrappers that wrap around direct syscalls. Some of the wrapping could be seen as bloat or unwanted overhead in a program. Also, those wrapper routines can be subject to user mode hooks, but let's leave hooks out of this for now.

An option that can be done is to avoid those wrapper functions that are typically found in `kernel32.dll` and `kernelbase.dll`, to name a few, and just go directly to the `syscall` itself in `ntdll.dll`. This action is how the technique named **direct syscalls** was born. Skip all of the higher level stuff and go directly to the lowest level possible.

## Syscall stub

We have already seen the format of a `syscall` and many folks simply call it a **stub** that prepares for the transition from ring 3 to ring 0, since a Windows installation only configures Intel CPUs to use those two rings (there are rings 1 and 2). The transition of rings is not done until the `syscall` instruction is executed. At that point in time, user mode code is left behind and the system will make the transition from ring 3 to ring 0. This will also change something called the **Current Privilege Level** (CPL) to 0.

## The syscall stub pattern

When implant developers want to perform a direct syscall, many times a search is implemented looking for the following sequence of bytes:

```text
// the start of the syscall stub
4c 8b d1 XX XX 00 00 

// the end of the stub
0f 05 c3 cd 2e c3
```

At the root of it, that is not really a complete stub, but on some older versions of Windows and ntdll.dll, it is. On more recent versions of Windows, there are some missing instructions from what is shown above. Those missing instructions are the following:

```asm
test    byte ptr [7FFE0308h], 1
jne     ntdll!<Some Nt Function>+0x15
```

Let's dive into those instructions in the next chapter.
