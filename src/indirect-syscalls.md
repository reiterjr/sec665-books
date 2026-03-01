# Indirect syscalls

## Cons of direct syscalls

As a refresher, direct syscalls are being invoked from within our own EXE. Cool, we can maybe get past some EDRs but there is a downside to this. We can be caught because the return address is not going to be located where it should be; ntdll.dll. This check can happen when the kernel is done doing its thing. Inside the **`_KPROCESS.InstrumentationCallback`** field, is a pointer to the routine that will be invoked after the `SYSCALL` is done (`SYSRET`). The check can be quite simple and if you know how to parse a PE image, then I'm sure you can already think of the simple 3 or so lines of C++ code to get that done, like so:

```cpp
const auto ImageBase = NtCurrentPeb()->ImageBaseAddress;
const auto NtHeaders = RtlImageNtHeader(ImageBase);
if ((retaddr >= ImageBase) && (retaddr < ImageBase + NtHeaders->OptionalHeader.SizeOfImage))
{
    // retaddr is within the EXE
}
```

## What are indirect syscalls?

Instead of having the `SYSCALL` instruction coming from within our own EXE's image, we need to have it come from within NTDLL. Like any normal `SYSCALL` would look like. To help with this, **Bouncy Gate** and **Recycled Gate** were made. There are too many gates.

## Implementation

Here, instead of calling the `SYSCALL` ourselves, we **jump** to it. We find the address of the `SYSCALL` instruction inside of NTDLL and `JMP` there. `JMP`s and `CALL`s are indirect instructions. In addition to the gates mentioned above, **SysWhispers3** also uses indirect syscalls, just like **Cobalt Strike's BOFs** do too.

## Cons about indirect syscalls

One of the downsides is that EDRs are catching on here. In addition to checking the return address of them, they are now looking at where they came from, EXE or NTDLL. So now, we have to fake where they are coming from with a new thread and then spoofing the call stack.
